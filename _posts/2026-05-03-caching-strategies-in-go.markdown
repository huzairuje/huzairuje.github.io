---
layout: post
title: "Caching Strategies in Go: From In-Process to Distributed"
date: 2026-05-03 11:00:00 +0700
categories: golang backend caching
tags: [golang, redis, caching, performance, backend]
description: "Multi-layer caching in Go from in-process ristretto to Redis. Covers stampede protection with singleflight, probabilistic early refresh, and tag-based cache invalidation."
---

Cache invalidation is one of the two hard problems in computer science. The other is naming. Here's how to implement caching in Go without shooting yourself in the foot.

## The Cache Hierarchy

```
Request → L1: In-Process (sync.Map / ristretto) → ns latency
        → L2: Redis (distributed)               → ~1ms
        → L3: Database / External API           → 10-100ms+
```

## L1: In-Process Cache with Ristretto

```go
import "github.com/dgraph-io/ristretto"

func NewLocalCache() (*ristretto.Cache, error) {
    return ristretto.NewCache(&ristretto.Config{
        NumCounters: 1e7,      // 10M counters for frequency tracking
        MaxCost:     100 << 20, // 100MB max memory
        BufferItems: 64,
        Metrics:     true,     // enable hit rate tracking
    })
}

type UserCache struct {
    local *ristretto.Cache
    redis *redis.Client
    db    UserRepository
}

func (c *UserCache) GetUser(ctx context.Context, id string) (*User, error) {
    // L1 check
    if v, ok := c.local.Get("user:" + id); ok {
        return v.(*User), nil
    }
    
    // L2 check
    var user User
    data, err := c.redis.Get(ctx, "user:"+id).Bytes()
    if err == nil {
        if err := json.Unmarshal(data, &user); err == nil {
            // Backfill L1
            c.local.SetWithTTL("user:"+id, &user, 1, 30*time.Second)
            return &user, nil
        }
    }
    
    // L3: database
    u, err := c.db.FindByID(ctx, id)
    if err != nil { return nil, err }
    
    // Populate both caches
    c.setRedis(ctx, "user:"+id, u, 5*time.Minute)
    c.local.SetWithTTL("user:"+id, u, 1, 30*time.Second)
    return u, nil
}
```

## Cache-Aside vs Write-Through

```go
// Cache-Aside (Lazy Loading): read → miss → load → cache
// Best for: read-heavy, tolerance for stale data

// Write-Through: write → update cache AND DB simultaneously
// Best for: write-heavy, strong consistency needed

func (c *UserCache) UpdateUser(ctx context.Context, user *User) error {
    // Write-through pattern
    if err := c.db.Update(ctx, user); err != nil {
        return err
    }
    
    // Update cache immediately
    data, _ := json.Marshal(user)
    c.redis.Set(ctx, "user:"+user.ID, data, 5*time.Minute)
    c.local.Del("user:" + user.ID) // invalidate L1, let it reload
    return nil
}
```

## Stampede Protection (Thundering Herd)

When cache expires, thousands of requests hit the DB simultaneously:

```go
import "golang.org/x/sync/singleflight"

type CacheWithSingleFlight struct {
    cache  *redis.Client
    db     Repository
    group  singleflight.Group
}

func (c *CacheWithSingleFlight) Get(ctx context.Context, key string) (*Data, error) {
    // Check cache first
    if data := c.getFromCache(ctx, key); data != nil {
        return data, nil
    }
    
    // Only ONE goroutine fetches from DB; all others wait and share the result
    result, err, _ := c.group.Do(key, func() (any, error) {
        data, err := c.db.Find(ctx, key)
        if err != nil { return nil, err }
        c.setCache(ctx, key, data, 5*time.Minute)
        return data, nil
    })
    
    if err != nil { return nil, err }
    return result.(*Data), nil
}
```

## Probabilistic Early Expiration (XFetch)

A smarter alternative: refresh cache slightly before it expires, probabilistically:

```go
// Based on the XFetch algorithm (Optimal Probabilistic Cache Stampede Prevention)
func (c *Cache) GetWithEarlyRefresh(ctx context.Context, key string, ttl time.Duration, fetch func() (any, error)) (any, error) {
    type cachedValue struct {
        Data      any       `json:"data"`
        ExpiresAt time.Time `json:"expires_at"`
        Delta     float64   `json:"delta"` // time to compute the value
    }
    
    var cv cachedValue
    raw, err := c.redis.Get(ctx, key).Bytes()
    if err == nil && json.Unmarshal(raw, &cv) == nil {
        // XFetch: should we proactively refresh?
        beta := 1.0
        earlyRefreshTime := cv.ExpiresAt.Add(-time.Duration(cv.Delta * beta * math.Log(rand.Float64()) * float64(time.Second)))
        
        if time.Now().Before(earlyRefreshTime) {
            return cv.Data, nil // still fresh, or not yet time to refresh
        }
    }
    
    // Fetch fresh data
    start := time.Now()
    data, err := fetch()
    if err != nil { return nil, err }
    delta := time.Since(start).Seconds()
    
    cv = cachedValue{Data: data, ExpiresAt: time.Now().Add(ttl), Delta: delta}
    raw, _ = json.Marshal(cv)
    c.redis.Set(ctx, key, raw, ttl)
    return data, nil
}
```

## Cache Invalidation Strategies

| Strategy | When to Use | Consistency |
|----------|-------------|-------------|
| TTL expiry | Non-critical data | Eventual |
| Event-based invalidation | Domain events (Kafka/pub-sub) | Near-real-time |
| Write-through | Strong consistency | Immediate |
| Tag-based invalidation | Related keys (e.g., all user posts) | Manual but precise |

```go
// Tag-based: cache a set of keys under a tag, then delete all at once
func (c *Cache) SetWithTag(ctx context.Context, key string, tag string, value any, ttl time.Duration) error {
    pipe := c.redis.Pipeline()
    data, _ := json.Marshal(value)
    pipe.Set(ctx, key, data, ttl)
    pipe.SAdd(ctx, "tag:"+tag, key)
    pipe.Expire(ctx, "tag:"+tag, ttl+time.Minute)
    _, err := pipe.Exec(ctx)
    return err
}

func (c *Cache) InvalidateByTag(ctx context.Context, tag string) error {
    keys, err := c.redis.SMembers(ctx, "tag:"+tag).Result()
    if err != nil { return err }
    if len(keys) == 0 { return nil }
    
    pipe := c.redis.Pipeline()
    pipe.Del(ctx, keys...)
    pipe.Del(ctx, "tag:"+tag)
    _, err = pipe.Exec(ctx)
    return err
}
```

## Monitoring Cache Effectiveness

Always track hit rate. Below 80% hit rate usually means your TTL or key strategy needs tuning:

```go
type CacheMetrics struct {
    hits   prometheus.Counter
    misses prometheus.Counter
}

func (m *CacheMetrics) HitRate() float64 {
    h := m.hits.(prometheus.Gauge).Get()
    miss := m.misses.(prometheus.Gauge).Get()
    if h+miss == 0 { return 0 }
    return h / (h + miss)
}
```

A well-tuned cache at 95% hit rate reduces database load by 20x. It's worth the complexity.
