---
layout: post
title: "Keeping the Domain Up: Layered HA with HAProxy, Keepalived, and Postgres Master-Slave"
date: 2026-05-23 08:00:00 +0700
categories: infrastructure backend
tags: [high-availability, haproxy, keepalived, postgresql, replication, load-balancer, failover, resilience]
description: "How a service stays online when something dies. Floating VIP at the edge, master-slave Postgres at the data tier, and the failure modes that survived the first outage."
---

The phrase "the site is down" hides a lot of complexity. Down for whom? Down because of what? In my experience, "down" almost always traces back to a single box doing a single thing, and that box picking the worst possible moment to stop doing it. Yesterday I wrote about [HAProxy master-slave with Keepalived and a floating VIP]({% post_url 2026-05-22-haproxy-master-slave-keepalived-vip %}) — that handles the edge. But an HA load balancer in front of a single Postgres primary just moves the single point of failure one layer deeper. This post is about layering it all the way down: edge, app, database. What stays up, what fails over, and what the client actually sees during the bad minute.

<!--more-->

## The picture

Here's the shape of the stack I keep coming back to. Stateless app tier in the middle, redundant pairs on either side.

![High-availability stack from DNS to disk](/assets/ha-architecture.svg)

A few things worth calling out before any config:

- **Clients only know one address.** DNS resolves `api.example.com` to a single VIP. They never learn the IPs of `lb-01` or `lb-02`. That's the whole point — when the master LB dies, clients don't have to refresh DNS, retry stale caches, or know anything happened.
- **The app tier is stateless.** Sessions live in Redis or in a JWT, never in process memory. Any app node can serve any request. Killing one is a non-event.
- **The database tier is the hard part.** It's stateful, it has a single writer, and replication is asynchronous by default. Most outages I've debugged in HA setups eventually traced back to something the DB layer did under stress.

## The edge tier in one paragraph

Two HAProxy nodes, identical config, run side by side. Keepalived runs on both and uses VRRP to negotiate ownership of a floating VIP. The master holds the VIP on its NIC; the backup is hot but idle. When the master misses three VRRP advertisements (~3 seconds at default timing), the backup promotes itself, claims the VIP, and sends a gratuitous ARP so the local switch updates its MAC table. Existing TCP sessions on the dead node are gone — you can't preserve those without session sync — but new connections to the same VIP land on the new master immediately. Full setup, the actual `haproxy.cfg`, and the `net.ipv4.ip_nonlocal_bind` gotcha are in [yesterday's post]({% post_url 2026-05-22-haproxy-master-slave-keepalived-vip %}).

What that post didn't cover: this whole dance is useless if the database behind it is single-homed.

## Postgres master-slave: same idea, harder problem

The Postgres equivalent of HAProxy's active-passive pair is **streaming replication with a hot standby**. One node is the primary, accepting reads and writes. One or more replicas connect to it, receive a continuous stream of WAL (write-ahead log) records, and replay them in order. The replica is queryable while it's replaying — that's the "hot" in hot standby — but it rejects writes.

![Postgres streaming replication and replica promotion](/assets/postgres-replication.svg)

The key configs on the primary:

```ini
# postgresql.conf — primary
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
synchronous_commit = on
synchronous_standby_names = 'ANY 1 (replica_01, replica_02)'
```

A note on `synchronous_commit`. Setting it to `on` with a `synchronous_standby_names` quorum means the primary won't acknowledge a commit until at least one replica has the WAL durably. That costs latency on every write — usually 1–3 ms in the same DC, more across regions — but it's the difference between "we lost 30 seconds of orders" and "we lost zero orders" when the primary goes down without warning. For a payment system, that latency was non-negotiable. For an analytics ingest, I left it `off` and ate the risk.

The replica config:

```ini
# postgresql.conf — replica
hot_standby = on
max_standby_streaming_delay = 30s
hot_standby_feedback = on
```

And the standby connection, written into the data directory at restore time:

```ini
# standby.signal exists (empty file)
# postgresql.auto.conf
primary_conninfo = 'host=pg-primary port=5432 user=replicator application_name=replica_01 sslmode=require'
primary_slot_name = 'replica_01_slot'
```

Replication slots matter. Without one, if the replica falls behind far enough that the primary recycles a WAL segment the replica still needs, replication breaks and you have to re-clone the replica from a base backup. With a slot, the primary keeps WAL around as long as the slot exists. The trade-off: a forgotten or disconnected slot will fill the primary's disk. I've seen this happen. Monitor `pg_replication_slots.active` and alert on any slot that's been inactive for more than a few minutes.

## What "the domain is up" actually requires

When someone says "the site stayed up during the failover," they mean a specific thing:

1. DNS still resolved.
2. TCP connections to the resolved IP completed.
3. TLS handshakes succeeded.
4. HTTP requests returned 2xx (or at worst, retryable 5xx that succeeded on retry).
5. Writes that the client believed succeeded actually persisted.

The first four are the load balancer's job. The fifth is the database's job, and it's the one most often quietly broken in HA setups.

## What the client sees during failover

This is the moment that decides whether a postmortem says "brief blip" or "incident."

![Failover sequence diagram](/assets/ha-failover-sequence.svg)

Walking through it:

- **t+0**: traffic flows normally. Client → VIP → lb-01 → app → primary, with WAL streaming to the replica.
- **t+12.4s**: lb-01 dies. Could be a kernel panic, could be the network card, could be someone tripping over the rack.
- **t+12.8s**: lb-02's Keepalived has missed three VRRP advertisements (`advert_int 1`, `dead = 3 * advert_int + skew`). It promotes itself.
- **t+13.0s**: lb-02 binds the VIP to its NIC and broadcasts a gratuitous ARP. The L2 switch updates its MAC table within milliseconds.
- **t+13.3s**: the client's in-flight TCP connection to lb-01 was reset when lb-01 died. The client retries (any well-behaved HTTP client does this for idempotent requests, and connection pools transparently reconnect on `ECONNRESET`). The retry hits the VIP, lands on lb-02, and completes.

From the client's perspective, the user-visible event is one request that was ~900 ms slower than usual. No 5xx, no error toast, no support ticket. The total observable downtime is bounded by VRRP detection time plus one TCP retry. Sub-second is achievable; under three seconds is normal.

## The Postgres failover is messier

LB failover is fast because it's stateless from the protocol's perspective. Postgres is the opposite. The primary holds open transactions, replication lag is real, and there's no equivalent of "gratuitous ARP" for "I'm the new primary now."

You need three things working together:

1. **A failover orchestrator** — Patroni, repmgr, or pg_auto_failover. Don't write your own. I tried. It's a graveyard of edge cases.
2. **A way to redirect application traffic** — usually pgbouncer with a config that gets rewritten, or a service-discovery layer like Consul, or a second VIP just for the database.
3. **Fencing** — making absolutely sure the old primary cannot accept writes after the new one is promoted. This is the part people skip and regret.

The classic failure mode is **split-brain**: the old primary wasn't really dead, it was network-partitioned. The orchestrator promoted the replica. Now writes are flowing to the new primary, but some clients with stale connections are still writing to the old one. When the partition heals, you have two databases that disagree about what happened, and no clean way to merge them. Patroni handles this by integrating with a consensus store (etcd or Consul) and refusing to be primary unless it holds the leader lock. The lock has a TTL, so a partitioned old primary loses its lock and demotes itself before it can do harm.

A minimal Patroni config sketch (the real one is longer):

```yaml
scope: payments-cluster
namespace: /db/
name: pg-01

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.11:8008

etcd3:
  hosts: 10.0.1.5:2379,10.0.1.6:2379,10.0.1.7:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1 MB max lag for promotion
    synchronous_mode: true
    synchronous_mode_strict: false
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        synchronous_commit: "on"

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.11:5432
  data_dir: /var/lib/postgresql/15/main
  authentication:
    replication:
      username: replicator
      password: ...
    superuser:
      username: postgres
      password: ...

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

The line that earns its keep: `maximum_lag_on_failover: 1048576`. A replica more than 1 MB behind the primary at failover time is *not* eligible for promotion. Better to fail loudly than to promote a stale replica and silently lose committed transactions.

## Gotchas the diagrams don't show

These are the things I learned by getting paged.

**`net.ipv4.ip_nonlocal_bind = 1` on both LBs.** Without it, HAProxy on the backup can't even start, because the VIP isn't bound to its NIC at boot. I covered this in the HAProxy post but it's worth repeating because it's the #1 reason a "tested" failover suddenly doesn't work in production — someone reboots the backup, HAProxy fails to start, and now your "HA pair" is a single LB with a corpse next to it.

**Replication lag during traffic spikes.** Async replication lag is usually measured in tens of milliseconds. Under load, especially when the replica is doing heavy reads, it can blow out to seconds or minutes. If you fail over during a lag spike, you lose every transaction that was in flight. Monitor `pg_stat_replication.replay_lag` and alert at thresholds that match your tolerance for data loss.

**Read-after-write on the replica.** This one bites everyone. App writes to primary, immediately reads from replica, gets stale data because replication hasn't caught up. The fixes, in order of preference: don't read from the replica right after a write (use the primary for reads inside a request that just wrote), use logical session-level read consistency, or accept the staleness and design the UI for it. There's no magic.

**Connection pool poisoning after failover.** PgBouncer holds open connections to the old primary. After failover, those connections error out, but the pool may keep handing them to clients before discarding. The fix is `server_check_query` and a short `server_check_delay`, plus `server_lifetime` low enough that connections cycle regularly. And the app side has to handle "I got a closed connection from the pool" as a retryable error.

**Backup is not HA.** I'll say this loudly because I keep seeing it: a daily `pg_dump` is not high availability. It's disaster recovery. If your primary dies and your only fallback is restoring last night's dump, your RPO is "up to 24 hours of data loss" and your RTO is "however long the restore takes plus however long it takes to rebuild indexes." Streaming replication with an orchestrator is what gets you to seconds-level RPO and minute-level RTO. You still need backups, for the case where someone runs `DROP TABLE orders` at 3 AM. They're complementary, not substitutes.

## Testing the failover before production tests it for you

Untested failover is a coin flip. The two drills I run on a schedule:

**LB failover drill.** SSH into the master LB and `systemctl stop keepalived`. The VIP should move to the backup within 3–4 seconds. Watch `ip addr` on both nodes. Watch `tcpdump -i eth0 vrrp` to see the protocol chatter. From a client outside the network, run a tight `curl` loop and count failures. A clean drill shows zero or one failed request, all subsequent ones succeeding. After verifying, restart keepalived on the original master — it should reclaim the VIP because of its higher priority.

**DB failover drill.** With Patroni: `patronictl -c /etc/patroni/patroni.yml failover`. Pick the target replica. The orchestrator stops the primary, promotes the chosen replica, and the old primary either rejoins as a replica via `pg_rewind` or gets reinitialized. Time the whole sequence. From the app side, run a write loop and count how many writes failed and how long the gap was. For my payment system the target was "fewer than 5 seconds of write unavailability, zero committed transactions lost." Measured, not guessed.

Run these drills monthly at minimum. Run them in business hours, with people watching, in environments that mirror production. The first time you do it on a Friday afternoon, it's terrifying. By the fifth time, it's boring, which is exactly what you want.

## Monitoring that actually catches things

The metrics I page on:

- `keepalived_vrrp_state` — alert if neither node reports MASTER, or both do (split brain).
- `haproxy_backend_servers_up` — per-backend, alert below quorum.
- `pg_stat_replication.replay_lag` — alert above your tolerance (I use 5 seconds for warning, 30 for page).
- `pg_replication_slots.active` — alert on any inactive slot, since that's a disk-fill bomb.
- Synthetic check from outside the DC hitting `api.example.com/healthz` — if this fails, nothing else matters.

The metrics I do *not* page on but watch in dashboards:

- VRRP transitions per day. A healthy pair has zero or one a week (deploys). A pair flapping every hour means a network or timing issue.
- Connection pool wait time on the app side. Climbing wait time during normal traffic is an early warning that the DB is struggling, which often precedes a failover.

## Wrapping up

The phrase "high availability" sounds like a property of the system. It's actually a property of *every layer* of the system. A redundant LB in front of a single DB is just an expensive way to have the same outage. Two DBs without a tested orchestrator and fencing strategy is a split-brain incident waiting for a network partition. A perfect HA stack with no monitoring is a system that will fail silently until a customer notices first.

The honest version of "the site is up" is: every box that could die has a partner ready to take over, every failover path has been tested in the last month, and somebody is paged within seconds when any of those partners stops talking. Everything else is theater.

If you're starting from a single LB and a single DB and want to get to this picture, the order I'd do it in: app statelessness first (it's free and easy), LB pair second (one weekend, scoped), DB replica with no automated failover third (it's already useful as a read replica and a backup target), and DB orchestration last (it's the hardest, do it when you actually understand your write patterns and have your monitoring honest). Skipping straight to "Patroni in prod" without the earlier steps is how you end up debugging consensus protocols at 3 AM with no idea what your application's actual behavior is under partial database availability.

Down is inevitable. Visibly down is optional.
