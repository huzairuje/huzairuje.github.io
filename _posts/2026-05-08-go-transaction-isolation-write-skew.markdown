---
layout: post
title: "Transaction Isolation in Go: The Write-Skew Incident"
date: 2026-05-08 09:00:00 +0700
categories: golang backend database
tags: [golang, sql, transactions, isolation, postgres]
description: "How read-committed transactions caused write skew in production and the locking strategy that fixed it."
---

We had a quota rule: at least one on-call engineer must remain active per team.

Two admins updated different rows at the same time. Both transactions passed validation. End result: zero active on-call engineers.

## Why Validation Failed

Each transaction did:

1. `SELECT count(*) WHERE active = true`
2. If count > 1, deactivate one engineer
3. Commit

Under `READ COMMITTED`, both saw the same pre-update snapshot and both committed.

## Safer Pattern

Lock the rows participating in the invariant.

```go
tx, _ := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
defer tx.Rollback()

rows, err := tx.QueryContext(ctx, `
    SELECT id FROM team_oncall
    WHERE team_id = $1 AND active = true
    FOR UPDATE
`, teamID)
if err != nil {
    return err
}

// validate then update inside same locked scope
```

## Practical Notes

- `SERIALIZABLE` is safest but can raise retryable serialization errors.
- `FOR UPDATE` can be enough for targeted invariants.
- Always encode critical business invariants close to the write path, not only in async jobs.

## What Went Wrong in My Incident

- **What alerted first:** An invariant alert fired showing impossible team state in production.
- **What misled us:** Manual replay of each transaction separately always passed validation.
- **What confirmed root cause:** Running concurrent transactions under `READ COMMITTED` reproduced write skew consistently.

Race conditions are not only in Go code. They are also in SQL behavior.
