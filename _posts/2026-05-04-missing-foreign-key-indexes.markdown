---
layout: post
title: "The Hidden Cost of Missing Foreign Key Indexes"
date: 2026-05-04 01:10:00 +0700
categories: database sql performance
tags: [sql, postgresql, database, indexing, performance]
description: "Why you should almost always index your foreign keys, and the catastrophic performance hits that occur on DELETE and UPDATE when you don't."
---

It is a common misconception that database engines automatically create an index for every Foreign Key constraint you define. **In PostgreSQL and many other databases, they do not.**

Failing to manually index your foreign keys can lead to catastrophic performance issues that only show up months after deployment.

## The Problem: Table Scans on DELETE

Consider a simple e-commerce schema:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id) ON DELETE CASCADE,
    amount DECIMAL
);
```

When you query `SELECT * FROM orders WHERE user_id = 42`, without an index on `user_id`, the database must do a **Sequential Scan** of the entire `orders` table. Most developers notice this quickly and add an index to speed up the read.

However, even if you *never* query `orders` by `user_id` directly, the missing index is a time bomb. 

Look what happens when you delete a user:

```sql
DELETE FROM users WHERE id = 42;
```

Because of the `ON DELETE CASCADE` (or even standard referential integrity checks), the database must verify if there are any child records in `orders` that reference `user_id = 42`. 

Without an index on `orders.user_id`, **every single user deletion triggers a full sequential scan of the orders table.**

If `orders` has 10 million rows, deleting a user will take seconds instead of milliseconds, locking the `users` row the entire time and crushing your database CPU.

## The Lock Escalation Problem

Missing foreign key indexes also cause severe locking issues during concurrent updates.

If Transaction A updates a parent row in `users`, and Transaction B inserts a child row in `orders` referencing that parent, the database must acquire locks to ensure referential integrity. In some database engines, without a foreign key index, the database might lock the entire parent table or block concurrent inserts heavily, crippling concurrency.

## The Solution

It is a simple rule of backend engineering: **If a column is a foreign key, index it.**

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### Finding Missing Foreign Key Indexes

Run this query in PostgreSQL to find all foreign keys that lack a corresponding index:

```sql
WITH fk_actions AS (
    SELECT
        c.conname AS constraint_name,
        c.conrelid::regclass AS table_name,
        c.confrelid::regclass AS foreign_table_name,
        unnest(c.conkey) AS column_index
    FROM pg_constraint c
    WHERE c.contype = 'f'
),
fk_columns AS (
    SELECT
        fk.constraint_name,
        fk.table_name,
        fk.foreign_table_name,
        a.attname AS column_name
    FROM fk_actions fk
    JOIN pg_attribute a ON a.attnum = fk.column_index AND a.attrelid = fk.table_name
)
-- This is a simplified version; tools like pg_stat_statements 
-- and pgAdmin provide comprehensive missing index views.
```

Don't wait for your DELETEs to time out. Check your schemas today.
