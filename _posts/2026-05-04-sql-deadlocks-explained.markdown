---
layout: post
title: "SQL Deadlocks: Why They Happen and How to Fix Them"
date: 2026-05-04 01:10:00 +0700
categories: database sql backend
tags: [sql, postgresql, database, backend, deadlocks]
description: "A deep dive into why SQL deadlocks occur in concurrent backend systems and how to fix them with consistent ordering and retry mechanisms."
---

Deadlocks are a rite of passage for backend engineers. Everything works perfectly in your local environment, but as soon as the system faces high concurrency in production, transactions start failing with `deadlock detected`.

Here is why they happen and how to eliminate them.

## The Classic Deadlock Scenario

A deadlock happens when two transactions wait for locks held by each other.

**Transaction A:**
1. Updates User X (Locks User X)
2. Needs to update User Y

**Transaction B:**
1. Updates User Y (Locks User Y)
2. Needs to update User X

If both transactions execute step 1 at the same time, they will wait forever at step 2. The database engine detects this circular dependency and abruptly aborts one of them to let the other proceed.

## Real-World Example: Moving Money

```go
// A naive transfer function that causes deadlocks
func TransferMoney(ctx context.Context, db *sql.DB, fromID, toID int, amount float64) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer tx.Rollback()

    // Deduct from sender
    _, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, fromID)
    if err != nil { return err }

    // Add to receiver
    _, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, toID)
    if err != nil { return err }

    return tx.Commit()
}
```

If User 1 transfers money to User 2 at the same exact time User 2 transfers money to User 1, you get a deadlock.

## The Fix: Consistent Lock Ordering

The easiest and most reliable way to prevent deadlocks is to **always acquire locks in a deterministic order**.

```go
func TransferMoneySafe(ctx context.Context, db *sql.DB, fromID, toID int, amount float64) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer tx.Rollback()

    // Lock in a consistent order (e.g., lowest ID first)
    firstID, secondID := fromID, toID
    if fromID > toID {
        firstID, secondID = toID, fromID
    }

    // You can lock both explicitly, or just ensure the UPDATEs happen in this order
    _, err = tx.ExecContext(ctx, "SELECT id FROM accounts WHERE id = $1 FOR UPDATE", firstID)
    if err != nil { return err }

    _, err = tx.ExecContext(ctx, "SELECT id FROM accounts WHERE id = $1 FOR UPDATE", secondID)
    if err != nil { return err }

    // Now proceed with updates
    _, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, fromID)
    // ...
```

By always locking the smaller ID first, both transactions will attempt to lock the same row first. One will succeed, and the other will queue behind it peacefully. Deadlock eliminated.

## The Fallback: Transaction Retries

Sometimes, consistent ordering is too complex (e.g. distributed transactions or complex ORM queries). In these cases, you must implement application-level retries.

```go
func WithDeadlockRetry(ctx context.Context, fn func() error) error {
    const maxRetries = 3
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        
        // Check if the error is a PostgreSQL deadlock (SQLSTATE 40P01)
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "40P01" {
            // Sleep briefly and retry
            time.Sleep(time.Millisecond * time.Duration(10*(i+1)))
            continue
        }
        return err // Not a deadlock, return immediately
    }
    return errors.New("max retries exceeded for deadlock")
}
```

Deadlocks aren't necessarily a sign of bad architecture; they are an expected part of concurrent databases. You just have to handle them gracefully.
