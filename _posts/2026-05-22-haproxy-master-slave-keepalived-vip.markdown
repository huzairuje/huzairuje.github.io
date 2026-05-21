---
layout: post
title: "HAProxy Master-Slave: Active-Passive Load Balancing with Keepalived and a Floating VIP"
date: 2026-05-22 08:00:00 +0700
categories: infrastructure backend
tags: [haproxy, keepalived, load-balancer, high-availability, vrrp, linux, networking]
description: "Building an active-passive HAProxy pair with Keepalived and a floating virtual IP. Real failover, real split-brain risks, and the gotchas that bit me in production."
---

A single load balancer is a single point of failure dressed up in a fancy hat. The first time the HAProxy box in front of our payment service rebooted for a kernel patch at 2 AM, every retry from every upstream client piled up against a dead TCP socket for the four minutes it took to come back. That outage is what pushed us from "one HAProxy, hope for the best" to a proper master-slave (active-passive) pair with a floating IP.

This is the setup, the config that actually shipped, and the failure modes I learned the hard way.

<!--more-->

## The shape of the thing

Two HAProxy nodes. One **virtual IP** (VIP) that sits on whichever node is currently the master. **Keepalived** runs on both nodes and uses VRRP (Virtual Router Redundancy Protocol) to negotiate who holds the VIP. DNS points at the VIP, never at either node directly.

```
                       clients
                          │
                   ┌──────▼──────┐
                   │  VIP        │   10.0.0.100  (floats)
                   └──────┬──────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                                   │
   ┌────▼────┐                         ┌────▼────┐
   │ lb-01   │  ◀── VRRP heartbeat ──▶ │ lb-02   │
   │ MASTER  │                         │ BACKUP  │
   │ HAProxy │                         │ HAProxy │
   │ Keepalvd│                         │ Keepalvd│
   └────┬────┘                         └────┬────┘
        │                                   │
        └────────────┬──────────────────────┘
                     │
              ┌──────▼──────┐
              │  app-01..N  │  (real backends)
              └─────────────┘
```

The key idea: only **one** node holds the VIP at a time. HAProxy on the backup is running, configured identically, with health checks live, but no traffic reaches it because the VIP isn't bound to its interface. When the master dies, Keepalived on the backup notices the missed VRRP advertisements, promotes itself, and brings up the VIP on its own NIC. Convergence is sub-second on a healthy network.

## Why floating VIP and not DNS round-robin

DNS round-robin sounds simpler. Two A records, both LBs in DNS, clients pick one. The problem is **DNS caching**. Resolvers, OS stub resolvers, JVM `InetAddress` cache, language runtimes — they all cache, often ignoring TTL. When a node dies, a non-trivial fraction of clients keep hammering the dead address for minutes to hours.

A floating VIP is at the network layer. The IP itself moves. Existing TCP sessions on the dead node are gone (you can't preserve those without session sync, which HAProxy doesn't do for TCP), but **new connections to the same IP** land on the new master immediately because ARP gets updated by a gratuitous ARP from Keepalived.

The trade-off: VIP requires both nodes to be on the same L2 segment. If your LBs are in different VPCs or different AZs without an L2 stretch, you need a different approach (BGP/anycast, cloud LB in front, or DNS with low TTL and aggressive client-side retries).

## HAProxy config

Same config on both nodes. Identical. The only thing that differs is Keepalived. Here's a trimmed version of what we run:

```haproxy
# /etc/haproxy/haproxy.cfg
global
    log         /dev/log local0
    log         /dev/log local1 notice
    maxconn     50000
    user        haproxy
    group       haproxy
    daemon
    stats socket /run/haproxy/admin.sock mode 660 level admin
    # Bind to a non-local address so HAProxy can listen on the VIP
    # even when Keepalived hasn't assigned it yet.
    # Without this, starting HAProxy on the backup fails with EADDRNOTAVAIL.
    # Set via sysctl: net.ipv4.ip_nonlocal_bind = 1

defaults
    log         global
    mode        http
    option      httplog
    option      dontlognull
    option      forwardfor
    option      http-server-close
    timeout connect 5s
    timeout client  60s
    timeout server  60s
    timeout http-request 10s
    retries 3

frontend fe_https
    bind 10.0.0.100:443 ssl crt /etc/haproxy/certs/site.pem alpn h2,http/1.1
    bind 10.0.0.100:80
    http-request redirect scheme https code 301 unless { ssl_fc }
    default_backend be_app

backend be_app
    balance roundrobin
    option httpchk GET /healthz
    http-check expect status 200
    default-server inter 2s fall 3 rise 2 maxconn 1000
    server app-01 10.0.1.11:8080 check
    server app-02 10.0.1.12:8080 check
    server app-03 10.0.1.13:8080 check
```

Two things worth pinning to the wall:

**`net.ipv4.ip_nonlocal_bind = 1`** on both nodes. HAProxy on the backup needs to bind to `10.0.0.100:443` even though the VIP isn't on its interface yet. Without this sysctl, HAProxy fails to start on the backup with `Cannot assign requested address`. Set it permanently:

```bash
echo 'net.ipv4.ip_nonlocal_bind = 1' | sudo tee /etc/sysctl.d/99-haproxy.conf
sudo sysctl --system
```

**Health checks (`option httpchk`)** are non-negotiable. Without them HAProxy will happily route requests to a backend that's CPU-pegged, deadlocked, or returning 500s, because as far as TCP is concerned the socket is open. `httpchk` plus an `expect status 200` makes HAProxy actually probe the app's `/healthz` endpoint.

## Keepalived config — master

```conf
# /etc/keepalived/keepalived.conf  on lb-01 (master)
global_defs {
    router_id LB_01
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"   # exit 0 if process exists
    interval 2
    weight -20      # if HAProxy dies, drop priority by 20
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 110
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <change-me>
    }
    virtual_ipaddress {
        10.0.0.100/24 dev eth0
    }
    track_script {
        chk_haproxy
    }
}
```

## Keepalived config — backup

Identical except for `state`, `priority`, and `router_id`:

```conf
# /etc/keepalived/keepalived.conf  on lb-02 (backup)
global_defs {
    router_id LB_02
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight -20
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51        # MUST match master
    priority 100                # lower than master
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <change-me>    # MUST match master
    }
    virtual_ipaddress {
        10.0.0.100/24 dev eth0
    }
    track_script {
        chk_haproxy
    }
}
```

The `vrrp_script` is what makes this an *active-passive load balancer pair* and not just an *active-passive IP failover*. Without it, Keepalived only watches the network — if HAProxy crashes but the kernel is fine, the VIP stays on the dead master and traffic blackholes. With `track_script`, a dead HAProxy drops the master's effective priority below the backup's, the backup wins the election, and the VIP migrates.

## Testing failover, properly

Three failure modes to exercise. If you only test one, you don't have HA, you have a placebo.

**1. Process death.** SSH to the master, `sudo systemctl stop haproxy`. Within ~4 seconds (`interval 2` × `fall 2`) the VIP should move to the backup. Confirm with `ip addr show eth0` on both nodes and `arping 10.0.0.100` from a third host.

**2. Node death.** Hard reboot the master (`sudo reboot` is too graceful — try `echo b | sudo tee /proc/sysrq-trigger`). Backup should take over within ~3 seconds (3 missed VRRP adverts at `advert_int 1`).

**3. Network partition.** This is the nasty one. Block VRRP between the nodes with `iptables`:

```bash
# on master
sudo iptables -A INPUT  -p vrrp -j DROP
sudo iptables -A OUTPUT -p vrrp -j DROP
```

Both nodes now think they're master. Both bring up the VIP. **This is split-brain.** Upstream switches see two MACs claiming the same IP and either flap or pin to whichever ARP they saw last. Traffic distribution becomes random.

Mitigations:
- Use `unicast_peer` instead of multicast VRRP. It still doesn't fix true partitions, but it removes one class of switch-level VRRP issues.
- Add a second tracking script that checks reachability of a third host (a gateway, a DB) so a node that's network-isolated demotes itself instead of claiming master.
- Monitor for "two nodes, state MASTER" in your alerting. If both LBs report MASTER for more than a few seconds, page someone.

Here's the gateway-reachability variant that saved us once:

```conf
vrrp_script chk_gateway {
    script "/bin/ping -c 1 -W 1 10.0.0.1"
    interval 3
    weight -30
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    # ... same as before ...
    track_script {
        chk_haproxy
        chk_gateway
    }
}
```

A node that can't reach the default gateway drops priority by 30 and stops fighting for master.

## Gotchas that cost me sleep

**Gratuitous ARP gets dropped by some switches.** When the VIP migrates, Keepalived sends GARP packets so upstream switches update their CAM tables. Some enterprise switches with aggressive ARP inspection drop these. Symptom: VIP is on the new master, but traffic still goes to the old one for 5+ minutes until ARP entries naturally expire. Fix: disable dynamic ARP inspection for the LB ports, or shorten ARP cache timeout on upstream devices.

**`virtual_router_id` collision.** VRID is a single byte that identifies the VRRP group on the L2 segment. If another team runs Keepalived on the same VLAN and picked the same VRID, you'll get bizarre, intermittent failover. Pick a VRID, document it somewhere shared, and grep your network for collisions before deploying.

**Asymmetric configs drift.** Six months in, someone tweaks `maxconn` on lb-01 during a fire and forgets to update lb-02. When failover happens, capacity changes silently. Run a config-diff cron between the two nodes, or — better — manage both with the same Ansible/Salt role and disallow manual edits.

**HAProxy reloads vs restarts.** `systemctl reload haproxy` is graceful and keeps connections alive via socket handover. `restart` drops every in-flight TCP connection. Roll out config changes with `reload`, do them on the backup first, fail over, then do the master.

**SSL cert sync.** Both nodes need the same cert at the same path. If only the master has the new cert and failover happens mid-renewal, clients see cert errors. Automate cert distribution (we use a small `rsync` job triggered by certbot's deploy hook).

## When this isn't the right answer

Active-passive HAProxy + Keepalived is great for: on-prem, single-DC, single-VLAN, predictable traffic, control over the network gear. It is **not** great for: multi-AZ cloud (use the cloud LB), global traffic (use anycast or GeoDNS), or extreme throughput where one active node bottlenecks (you want active-active with ECMP or a real LB cluster).

For a backend service handling a few thousand RPS in one datacenter, though, this setup has been boring and reliable for years — which is exactly what you want from a load balancer.
