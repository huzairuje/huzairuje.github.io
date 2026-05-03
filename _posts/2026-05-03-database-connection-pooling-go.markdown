---
layout: post
title: "Database Connection Pooling and Query Optimization in Go"
date: 2026-05-03 10:00:00 +0700
categories: golang backend database
tags: [golang, database, postgresql, connection-pool, performance]
description: "Fixing the most common Go database performance problems: N+1 queries, connection pool sizing with pgx, bulk inserts with COPY, and index strategy for PostgreSQL."
---

Most Go backend performance problems trace back to the database. Connection pool misconfiguration and N+1 queries are responsible for more production incidents than any other cause.

## The N+1 Query Problem

```go
// Classic N+1: 1 query for orders + N queries for each order's items
func GetOrders(ctx context.Context, db *sql.DB) ([]OrderWithItems, error) {
    orders, err := db.QueryContext(ctx, "SELECT id, user_id, total FROM orders")
    // ...
    
    for _, order := range orders {
        // This fires a query PER ORDER — brutal at scale
        items, err := db.QueryContext(ctx,
            "SELECT * FROM order_items WHERE order_id = $1", order.ID)
        // ...
    }
}
```

At 1000 orders, that's 1001 queries. Fix it with a JOIN or batch fetch:

```go
// Option A: JOIN (best for small result sets)
const query = `
    SELECT 
        o.id, o.user_id, o.total,
        oi.id AS item_id, oi.sku, oi.quantity, oi.price
    FROM orders o
    LEFT JOIN order_items oi ON oi.order_id = o.id
    WHERE o.user_id = $1
    ORDER BY o.created_at DESC
`

// Option B: Batch fetch (best when result sets are large)
func GetOrdersWithItems(ctx context.Context, db *sql.DB, userID string) ([]OrderWithItems, error) {
    orders, err := fetchOrders(ctx, db, userID)
    if err != nil { return nil, err }
    
    // Collect all IDs
    ids := make([]string, len(orders))
    for i, o := range orders { ids[i] = o.ID }
    
    // Single batch query using ANY()
    items, err := db.QueryContext(ctx,
        `SELECT order_id, sku, quantity, price 
         FROM order_items 
         WHERE order_id = ANY($1)`, pq.Array(ids))
    
    // Group items by order ID in Go
    itemMap := groupItemsByOrder(items)
    for i := range orders {
        orders[i].Items = itemMap[orders[i].ID]
    }
    return orders, nil
}
```

## Connection Pool Configuration

```go
func NewDB(cfg DBConfig) (*sql.DB, error) {
    db, err := sql.Open("pgx", cfg.DSN)
    if err != nil {
        return nil, err
    }
    
    // These defaults are too conservative for production
    db.SetMaxOpenConns(25)          // max concurrent connections
    db.SetMaxIdleConns(10)          // keep these warm
    db.SetConnMaxLifetime(5 * time.Minute)  // recycle to catch stale connections
    db.SetConnMaxIdleTime(1 * time.Minute)  // don't hold idle conns forever
    
    return db, nil
}
```

**How to size `MaxOpenConns`**: Start with `(num_cores * 2) + effective_spindle_count`. For PostgreSQL, a common rule: `100 / num_app_instances`. Increase based on observed wait times in `pg_stat_activity`.

```sql
-- Check how many connections are actually waiting
SELECT count(*), wait_event_type, wait_event 
FROM pg_stat_activity 
WHERE state = 'active'
GROUP BY wait_event_type, wait_event;
```

## Using pgx for Better Performance

```go
import "github.com/jackc/pgx/v5/pgxpool"

func NewPgxPool(ctx context.Context, cfg DBConfig) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(cfg.DSN)
    if err != nil { return nil, err }
    
    config.MaxConns = 25
    config.MinConns = 5
    config.MaxConnLifetime = 5 * time.Minute
    config.HealthCheckPeriod = 30 * time.Second
    
    // Prepared statement cache per connection
    config.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeCacheDescribe
    
    return pgxpool.NewWithConfig(ctx, config)
}
```

pgx is significantly faster than `database/sql` with the `lib/pq` driver for PostgreSQL, especially for bulk operations.

## Bulk Insert with COPY

```go
// Inserting 10,000 rows with individual INSERTs: ~5-10 seconds
// With COPY: ~200ms

func BulkInsertOrders(ctx context.Context, pool *pgxpool.Pool, orders []Order) error {
    conn, err := pool.Acquire(ctx)
    if err != nil { return err }
    defer conn.Release()
    
    _, err = conn.CopyFrom(ctx,
        pgx.Identifier{"orders"},
        []string{"id", "user_id", "total", "status", "created_at"},
        pgx.CopyFromSlice(len(orders), func(i int) ([]any, error) {
            o := orders[i]
            return []any{o.ID, o.UserID, o.Total, o.Status, o.CreatedAt}, nil
        }),
    )
    return err
}
```

## Query Tracing with OpenTelemetry

```go
import "github.com/jackc/pgx/v5/tracelog"

// Trace every query with spans
config.ConnConfig.Tracer = &tracelog.TraceLog{
    Logger:   pgxLogger,
    LogLevel: tracelog.LogLevelInfo,
}

// Or with OTEL directly
config.ConnConfig.Tracer = otelpgx.NewTracer()
```

## Index Strategy

```sql
-- Partial index: only index what you query
CREATE INDEX idx_orders_pending 
ON orders (created_at DESC) 
WHERE status = 'PENDING';

-- Covering index: include columns to avoid table lookup
CREATE INDEX idx_orders_user 
ON orders (user_id, created_at DESC) 
INCLUDE (id, total, status);

-- Check if your indexes are being used
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

## Key Takeaways

1. Always measure first: use `EXPLAIN (ANALYZE, BUFFERS)` before optimizing
2. N+1 queries are the #1 performance killer — use sqlc or an ORM with eager loading
3. Pool size matters more than query micro-optimization at scale
4. `pgx` + `pgxpool` outperforms `database/sql` + `lib/pq` for PostgreSQL
5. Bulk operations with `COPY` are orders of magnitude faster than individual inserts
