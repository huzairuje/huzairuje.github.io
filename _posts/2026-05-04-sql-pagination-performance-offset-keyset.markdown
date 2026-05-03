---
layout: post
title: "The Dangers of Pagination with OFFSET and LIMIT"
date: 2026-05-04 01:00:00 +0700
categories: database sql performance
tags: [sql, postgresql, database, performance, backend]
description: "Why OFFSET/LIMIT pagination kills your database performance at scale, and how to implement Keyset Pagination (Cursor Pagination) instead."
---

If you've ever built an API with a query string like `?page=500&size=20`, you've used `OFFSET` and `LIMIT`. It is the default way developers build pagination. 

And it is a performance disaster waiting to happen.

## Why OFFSET is Slow

Look at this query:

```sql
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 100000;
```

To the untrained eye, this looks like it only fetches 20 rows. However, the database engine **cannot jump directly to row 100,000**. 

To fulfill this query, the database must:
1. Scan the index to find the first 100,020 rows.
2. Sort them.
3. Discard the first 100,000 rows.
4. Return the remaining 20.

As your offset grows, your query time grows linearly. Page 1 is instant. Page 5,000 will time out your API.

## The UX Problem: Missing and Duplicate Data

Beyond performance, `OFFSET` causes data anomalies when the underlying table changes.

1. **User requests Page 1** (Items 1-20).
2. While reading, **a new item is added** at the top.
3. All existing items are pushed down by 1. Item 20 is now Item 21.
4. **User requests Page 2** (Items 21-40).
5. The user sees the original Item 20 again as the first item on Page 2!

## The Solution: Keyset Pagination (Cursor Pagination)

Keyset pagination remembers the *value* of the last row fetched, rather than its absolute position.

```sql
-- Instead of OFFSET, we filter based on the last seen cursor
SELECT * FROM orders 
WHERE created_at < '2026-05-01 12:00:00'
ORDER BY created_at DESC 
LIMIT 20;
```

### Advantages
1. **O(1) Performance:** The database uses an index seek to jump straight to the correct timestamp, taking milliseconds regardless of page depth.
2. **Stable Data:** Inserts and deletions do not shift the pagination window.

### Handling Ties

If multiple rows have the exact same `created_at`, the query above might skip rows. You need a unique tie-breaker (like the primary key `id`).

```sql
SELECT * FROM orders 
WHERE (created_at, id) < ('2026-05-01 12:00:00', 10523)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

*(Note: Tuple comparison `(a,b) < (x,y)` is supported natively in PostgreSQL and is highly optimized).*

### Building the API

Your API response should encode this cursor opaquely, so clients don't need to parse it:

```json
{
  "data": [...],
  "next_cursor": "MjAyNi0wNS0wMVQxMjowMDowMFp8MTA1MjM="
}
```

The cursor is simply a base64 encoded string of `timestamp|id`.

## When is OFFSET Okay?
- Admin dashboards where you absolutely need a "Go to Page 45" button.
- Small tables (< 10,000 rows).

For everything else, specifically infinite-scroll feeds or public APIs, cursor pagination is the only way to scale.
