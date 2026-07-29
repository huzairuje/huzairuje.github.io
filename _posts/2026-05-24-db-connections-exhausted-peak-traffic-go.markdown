---
layout: post
title: "Database Connections Exhausted at Peak Traffic — Even With SetMaxOpenConns Set"
date: 2026-05-24 13:58:00 +0700
categories: golang backend database
tags: [golang, database, postgresql, connection-pool, debugging, production]
description: "SetMaxOpenConns is set, but connections still run out under load. Here are the five real causes — with diagrams and fixes for each."
---

You configured the pool. You set the limit. And yet, at 9am on Monday, your Go service starts throwing:

```
driver: bad connection
sql: database is closed
context deadline exceeded
```

Here's what's actually happening — and how to fix each cause.

---

## What `SetMaxOpenConns` Actually Does

```go
db.SetMaxOpenConns(25)
```

![Connection pool — how SetMaxOpenConns works](/assets/db-pool-overview.svg)

It caps how many connections can be open at once. But it does **not** prevent exhaustion on its own. Here's why.

---

## Culprit #1 — Connection Leak (The Most Common One)

You query the DB, forget to close `rows`, and the connection never returns to the pool.

```go
// ❌ LEAKING — rows never closed
func getUsers(db *sql.DB) {
    rows, err := db.Query("SELECT id FROM users")
    if err != nil {
        return
    }
    // forgot rows.Close()
    // connection is now held forever
}
```

What this looks like over time:

![Connection leak — rows.Close() missing vs present](/assets/db-connection-leak.svg)

The fix:

```go
// ✅ Always defer rows.Close()
func getUsers(db *sql.DB) ([]User, error) {
    rows, err := db.Query("SELECT id, name FROM users")
    if err != nil {
        return nil, err
    }
    defer rows.Close() // ← this line saves you

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()
}
```

Same rule applies to transactions. An uncommitted or unrolled-back `tx` holds a connection open:

```go
// ❌ tx never committed or rolled back — connection leaked
tx, _ := db.Begin()
_, err := tx.Exec("UPDATE accounts SET balance = ?", newBalance)
if err != nil {
    return // connection is still held
}

// ✅ Always defer Rollback, then commit explicitly
func transfer(ctx context.Context, db *sql.DB, amount float64) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback() // no-op if already committed

    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount, fromID,
    )
    if err != nil {
        return err // Rollback fires here
    }

    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, toID,
    )
    if err != nil {
        return err // Rollback fires here
    }

    return tx.Commit()
}
```

---

## Culprit #2 — `SetMaxIdleConns` Is Too Low (or Zero)

`SetMaxOpenConns` controls the ceiling. `SetMaxIdleConns` controls how many connections stay warm in the pool when idle.

![SetMaxIdleConns too low — pool thrashing vs stable](/assets/db-pool-thrashing.svg)

If `MaxIdleConns < MaxOpenConns`, connections get destroyed and recreated on every burst. This adds latency and can cause exhaustion during the reconnect window.

```go
// ✅ Balanced configuration
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)         // keep all of them warm
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(1 * time.Minute)
```

---

## Culprit #3 — Long-Running Queries Holding Connections

Each connection is occupied for the full duration of a query. If one query takes 30 seconds, that connection is gone for 30 seconds.

```
Normal query:   ──[query]──  (5ms)   → connection free quickly
Slow query:     ──────────────────────────────[query]── (30s)
                ↑ connection blocked for entire duration

Under load:
┌────────────────────────────────────────────────────┐
│ C1  [============================slow query=======] │
│ C2  [============================slow query=======] │
│ C3  [============================slow query=======] │
│ ...                                                 │
│ C25 [============================slow query=======] │
│                                                     │
│ New requests → queue → timeout ❌                   │
└────────────────────────────────────────────────────┘
```

Always use context with a deadline:

```go
// ❌ No timeout — query can hang indefinitely
rows, err := db.Query("SELECT * FROM orders WHERE user_id = $1", userID)

// ✅ Query with timeout — connection released when context expires
func getOrders(ctx context.Context, db *sql.DB, userID string) ([]Order, error) {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    rows, err := db.QueryContext(ctx,
        "SELECT id, total, status FROM orders WHERE user_id = $1", userID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var orders []Order
    for rows.Next() {
        var o Order
        if err := rows.Scan(&o.ID, &o.Total, &o.Status); err != nil {
            return nil, err
        }
        orders = append(orders, o)
    }
    return orders, rows.Err()
}
```

If the query exceeds 3 seconds, the context cancels, the connection is freed, and the caller gets a clear error instead of a silent hang.

---

## Culprit #4 — No `SetConnMaxLifetime`, Connections Go Stale

Without a lifetime limit, connections live forever. The DB server — or a firewall/load balancer sitting between your app and DB — may silently close idle connections. Your pool thinks they're alive. They're not.

```
App Pool                         Firewall / DB Server
   │                                     │
   │  C1 open (idle for 10 min)          │
   │                                     │── closes C1 silently
   │                                     │
   │  New request tries to use C1 ───────┼──► "bad connection" ❌
   │                                     │
   │  Go retries with a new connection   │
   │  (adds latency + burns a new slot)  │
```

This is especially common with:
- AWS RDS Proxy (idle timeout: 10 min by default)
- PgBouncer in transaction mode
- Any NAT gateway with connection tracking

```go
// ✅ Rotate connections before they go stale
db.SetConnMaxLifetime(5 * time.Minute)  // max age of any connection
db.SetConnMaxIdleTime(1 * time.Minute)  // max idle time before closing
```

Set `ConnMaxLifetime` lower than your infrastructure's idle timeout. If RDS Proxy kills at 10 minutes, rotate at 5.

---

## Culprit #5 — Context Not Propagated, Goroutines Pile Up

When a request is cancelled (user closes browser, upstream timeout fires), if you're not using `context`, the DB query keeps running and holds the connection.

```
User cancels request at t=0ms
  │
  ├── HTTP handler returns
  │
  └── goroutine still running db.Query("SELECT...") ← no context
        │
        └── holds connection for 10 more seconds
              │
              └── 500 concurrent cancellations = 500 stuck connections
                    │
                    └── pool exhausted ❌
```

```go
// ❌ Context ignored — query outlives the request
func handler(w http.ResponseWriter, r *http.Request) {
    rows, err := db.Query("SELECT * FROM big_table")
    // if client disconnects, query keeps running
    // ...
}

// ✅ Context flows through — query cancelled with the request
func handler(w http.ResponseWriter, r *http.Request) {
    rows, err := db.QueryContext(r.Context(), "SELECT * FROM big_table")
    // client disconnects → r.Context() cancelled → query stops → connection freed
    if err != nil {
        http.Error(w, "query failed", http.StatusInternalServerError)
        return
    }
    defer rows.Close()
    // ...
}
```

The rule: every DB call should accept and use a `context.Context`. No exceptions.

---

## The Complete Fix — Sane Pool Configuration

```go
func newDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("pgx", dsn)
    if err != nil {
        return nil, err
    }

    // How many connections can be open at once
    db.SetMaxOpenConns(25)

    // How many to keep warm (match MaxOpenConns for bursty traffic)
    db.SetMaxIdleConns(25)

    // Rotate connections to avoid stale/firewall issues
    // Set lower than your infra's idle timeout (RDS Proxy default: 10min)
    db.SetConnMaxLifetime(5 * time.Minute)

    // Close connections that have been idle too long
    db.SetConnMaxIdleTime(2 * time.Minute)

    // Verify the pool works at startup — fail fast
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("db ping failed: %w", err)
    }

    return db, nil
}
```

---

## How to Catch This Before It Hits Production

`sql.DB` exposes pool stats. Export them to your metrics system on every scrape interval:

```go
func recordDBStats(db *sql.DB, metrics *prometheus.GaugeVec) {
    stats := db.Stats()

    metrics.WithLabelValues("open").Set(float64(stats.OpenConnections))
    metrics.WithLabelValues("in_use").Set(float64(stats.InUse))
    metrics.WithLabelValues("idle").Set(float64(stats.Idle))
    metrics.WithLabelValues("wait_count").Set(float64(stats.WaitCount))
    metrics.WithLabelValues("wait_duration_ms").Set(
        float64(stats.WaitDuration.Milliseconds()),
    )
    metrics.WithLabelValues("max_idle_closed").Set(float64(stats.MaxIdleClosed))
    metrics.WithLabelValues("max_lifetime_closed").Set(float64(stats.MaxLifetimeClosed))
}
```

The signals to watch:

```
stat                  what it tells you
────────────────────  ──────────────────────────────────────────────
WaitCount ↑           requests queuing for a connection → pool too small
                      or connections are leaking
InUse == MaxOpen      pool is saturated right now
MaxIdleClosed ↑       MaxIdleConns too low, pool is thrashing
MaxLifetimeClosed ↑   normal rotation — expected if lifetime is short
WaitDuration ↑        users are feeling this — act immediately
```

Set an alert on `WaitCount` growing over time. That's your earliest warning before users see errors.

---

## The Mental Model

```
SetMaxOpenConns    → the size of your parking lot
SetMaxIdleConns    → reserved spots kept warm and ready
SetConnMaxLifetime → how long a car can stay before it must leave
SetConnMaxIdleTime → how long an empty spot waits before being released
Context            → the valet that cancels the reservation if you leave
rows.Close()       → actually returning the car to the lot
tx.Rollback()      → returning the car even when something went wrong
```

Miss any one of these and the lot fills up — even if the sign says "25 spaces."

---

## Key Takeaways

1. `rows.Close()` and `tx.Rollback()` are not optional — leak one and you leak a connection
2. `SetMaxIdleConns` should match `SetMaxOpenConns` for bursty workloads
3. Always pass `context.Context` to every DB call — it's your safety valve
4. Set `ConnMaxLifetime` lower than your infra's idle timeout
5. Export `db.Stats()` to metrics — `WaitCount` is your canary

The pool config is one line. The discipline around it is the whole job.

---

Next up: **Goroutines pile up into the hundreds of thousands even though CPU usage is flat** — same format, same depth.
