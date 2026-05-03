---
layout: post
title: "Zero-Downtime Database Migrations on Large Tables"
date: 2026-05-04 01:10:00 +0700
categories: database backend architecture
tags: [sql, postgresql, database, migrations, backend]
description: "How to safely add columns, build indexes, and modify schema on massive PostgreSQL tables without causing production downtime or lock contention."
---

Running `ALTER TABLE` in a local development environment is easy. Running it on a 50GB production table with 10,000 transactions per second is terrifying. 

If you do it wrong, your migration will acquire an `AccessExclusiveLock`, blocking all reads and writes to the table until it finishes, taking your entire service offline.

Here are the rules for zero-downtime PostgreSQL migrations.

## 1. Adding an Index: Use CONCURRENTLY

**The Bad Way:**
```sql
CREATE INDEX idx_users_email ON users(email);
```
This locks the table against writes for the entire duration of the index build.

**The Safe Way:**
```sql
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```
This builds the index in the background without blocking writes. It takes slightly longer, but your app stays online. *(Note: You cannot run this inside a `BEGIN/COMMIT` transaction block).*

## 2. Adding a Column with a Default Value

**The Bad Way (Pre-Postgres 11):**
```sql
ALTER TABLE users ADD COLUMN is_active BOOLEAN DEFAULT TRUE;
```
Before Postgres 11, this forced the database to rewrite every single row in the table to add the default value. A 50 million row table would be locked for minutes.

**The Safe Way:**
Since Postgres 11, adding a column with a constant default is metadata-only and practically instant. However, if you are on an older version or using a volatile default (like `RANDOM()`), use this pattern:

1. Add the column without a default.
2. Update the application to write the default value for new rows.
3. Backfill old rows in small batches using a background script.

## 3. Changing a Column Type

**The Bad Way:**
```sql
ALTER TABLE users ALTER COLUMN phone TYPE VARCHAR(255);
```
If changing the type requires rewriting the data (e.g., `INT` to `TEXT`), Postgres will lock and rewrite the entire table.

**The Safe Way (The Expand and Contract Pattern):**
1. **Expand**: Add a new column (`phone_new VARCHAR(255)`).
2. **Dual-Write**: Update your application to write to both columns, and set up a database trigger to sync them.
3. **Backfill**: Run a script to copy data from `phone` to `phone_new` in small batches for existing rows.
4. **Switch**: Update your application to read from `phone_new`.
5. **Contract**: Drop the old `phone` column and rename `phone_new` to `phone`.

## 4. Lock Timeouts

Never run a migration without a `lock_timeout`. 

If your `ALTER TABLE` statement is waiting in the queue to acquire a lock, it blocks all subsequent queries behind it. Even a fast migration can cause an outage if it gets stuck behind a long-running `SELECT`.

```sql
-- Abort the migration if we can't acquire the lock in 2 seconds
SET lock_timeout = '2s';
ALTER TABLE users ADD COLUMN last_login TIMESTAMP;
```

If it fails, retry during an off-peak time or use a migration runner that automatically retries with backoff.

Zero-downtime migrations require more steps, but they are the only way to evolve schema in high-availability backend systems.
