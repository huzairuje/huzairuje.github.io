---
layout: post
title: "Pessimistic vs Optimistic Locking in Concurrent Backends"
date: 2026-05-04 10:00:00 +0700
categories: database backend concurrency
tags: [sql, database, concurrency, backend]
description: "How to handle high-concurrency race conditions in your database using pessimistic locking (FOR UPDATE) and optimistic locking (version numbers)."
---

Race conditions in databases happen when two concurrent requests try to update the same row based on stale data. The classic example is two users trying to book the last seat on a flight. Both see the seat is available, both try to book it, and if not handled correctly, both succeed.

To solve this, we have two primary tools: **Pessimistic Locking** and **Optimistic Locking**.

## Pessimistic Locking (`SELECT ... FOR UPDATE`)

Pessimistic locking assumes the worst: conflicts *will* happen, so we must lock the row at the database level before we read it.

```sql
BEGIN;

-- Lock the row. Any concurrent transaction trying to read this 
-- with FOR UPDATE will block and wait here.
SELECT status FROM seats WHERE id = 42 FOR UPDATE;

-- Application logic decides if booking is allowed
UPDATE seats SET status = 'BOOKED' WHERE id = 42;

COMMIT;
```

**In Go:**
```go
tx, _ := db.Begin()
var status string
err := tx.QueryRow("SELECT status FROM seats WHERE id = 42 FOR UPDATE").Scan(&status)
if status != "AVAILABLE" {
    tx.Rollback()
    return ErrAlreadyBooked
}
tx.Exec("UPDATE seats SET status = 'BOOKED' WHERE id = 42")
tx.Commit()
```

### When to use Pessimistic Locking:
- High contention: Many requests trying to update the *exact same row* at the same time.
- Strict consistency requirements (e.g. financial ledgers).
- Short transaction times (don't hold the lock for long!).

## Optimistic Locking (Version Columns)

Optimistic locking assumes conflicts are rare. We don't block concurrent reads. Instead, we add a `version` column to the table. When updating, we verify the version hasn't changed since we read it.

```sql
-- Read the current state
SELECT status, version FROM seats WHERE id = 42;
-- Result: status='AVAILABLE', version=1

-- Attempt to update, expecting version 1
UPDATE seats 
SET status = 'BOOKED', version = version + 1 
WHERE id = 42 AND version = 1;
```

If a concurrent transaction beat us to the punch, the `version` in the DB is now 2. Our `UPDATE` statement will affect **0 rows**. 

**In Go:**
```go
var status string
var version int
db.QueryRow("SELECT status, version FROM seats WHERE id = 42").Scan(&status, &version)

if status != "AVAILABLE" { return ErrAlreadyBooked }

result, _ := db.Exec("UPDATE seats SET status = 'BOOKED', version = version + 1 WHERE id = 42 AND version = $1", version)

rowsAffected, _ := result.RowsAffected()
if rowsAffected == 0 {
    // The version changed! Conflict!
    return ErrConcurrentUpdateConflict
}
```

### When to use Optimistic Locking:
- Low contention: Conflicts are possible but rare.
- High read throughput: You don't want readers blocking writers or vice versa.
- Long-running processes: A user opens a form, goes to lunch, and hits "Save". You can't hold a DB lock for 45 minutes, but you *can* pass a version token to the frontend and check it on submit.

## The Verdict

Pessimistic locking provides absolute safety but hurts throughput because it serializes access. Optimistic locking scales infinitely better for reads, but requires application-level retry logic to handle the `ErrConcurrentUpdateConflict`.

Choose Optimistic by default, unless you are building a system where conflicts are the normal operating mode.
