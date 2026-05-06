---
layout: post
title: "Go Cache Stampede: Fixing Redis Meltdowns with singleflight"
date: 2026-05-08 01:00:00 +0700
categories: golang backend caching
tags: [golang, redis, cache, singleflight, performance]
description: "How cache expiration spikes can flood databases, and the Go singleflight pattern that collapses duplicate work."
---

At midnight, one hot key expired. Then thousands of requests hit the same miss path.

Redis was healthy. PostgreSQL was not.

## Stampede Pattern

When a hot key expires, many workers do the same expensive DB query simultaneously. Cache saves nothing, DB takes all the pain.

## Mitigation with `singleflight`

```go
var group singleflight.Group

func GetUser(ctx context.Context, userID string) (User, error) {
    key := "user:" + userID
    if u, ok := readCache(key); ok {
        return u, nil
    }

    v, err, _ := group.Do(key, func() (any, error) {
        // double check cache in case another goroutine filled it
        if u, ok := readCache(key); ok {
            return u, nil
        }
        u, err := loadUserFromDB(ctx, userID)
        if err != nil {
            return User{}, err
        }
        writeCache(key, u, 5*time.Minute)
        return u, nil
    })
    if err != nil {
        return User{}, err
    }
    return v.(User), nil
}
```

## Additional Hardening

- Add TTL jitter to avoid synchronized expirations.
- Serve stale data briefly when DB is overloaded.
- Pre-warm critical keys before known traffic spikes.

## What Went Wrong in My Incident

- **What alerted first:** Database CPU and lock waits spiked exactly when a hot cache key expired.
- **What misled us:** We treated it as a PostgreSQL regression because Redis was still healthy.
- **What confirmed root cause:** Request traces showed thousands of concurrent cache misses executing the same query.

Caching is easy on diagrams. In production, expiration timing is where real complexity lives.
