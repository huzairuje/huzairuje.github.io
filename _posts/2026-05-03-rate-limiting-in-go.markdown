---
layout: post
title: "Rate Limiting in Go: Token Bucket, Sliding Window, and Redis"
date: 2026-05-03 09:00:00 +0700
categories: golang backend performance
tags: [golang, rate-limiting, redis, token-bucket, backend]
description: "Production-grade rate limiting in Go: token bucket with golang.org/x/time/rate, distributed sliding window with Redis Lua scripts, and adaptive throttling based on system health."
---

Uncontrolled traffic kills backends. Rate limiting is not optional in production — it's the difference between a degraded service and a complete outage. Here's how to implement it correctly in Go.

## The Wrong Way: In-Memory Single Instance

```go
// Looks fine, breaks in production
var (
    mu       sync.Mutex
    requests = make(map[string]int)
)

func isAllowed(clientID string) bool {
    mu.Lock()
    defer mu.Unlock()
    requests[clientID]++
    return requests[clientID] <= 100 // 100 per... when? This has no window.
}
```

**Problems**: no time window, doesn't work with multiple instances, memory leak.

## Token Bucket with `golang.org/x/time/rate`

The standard library approach for single-instance rate limiting:

```go
import "golang.org/x/time/rate"

type RateLimiter struct {
    limiters sync.Map
    r        rate.Limit
    b        int
}

func NewRateLimiter(rps float64, burst int) *RateLimiter {
    return &RateLimiter{r: rate.Limit(rps), b: burst}
}

func (rl *RateLimiter) getLimiter(key string) *rate.Limiter {
    if v, ok := rl.limiters.Load(key); ok {
        return v.(*rate.Limiter)
    }
    l := rate.NewLimiter(rl.r, rl.b)
    rl.limiters.Store(key, l)
    return l
}

func (rl *RateLimiter) Allow(key string) bool {
    return rl.getLimiter(key).Allow()
}

// Middleware
func RateLimitMiddleware(rl *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            clientIP := r.RemoteAddr
            if !rl.Allow(clientIP) {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

## Distributed Rate Limiting with Redis

For multi-instance deployments, Redis is the source of truth. The sliding window log algorithm:

```go
const rateLimitScript = `
local key = KEYS[1]
local window = tonumber(ARGV[1])  -- window in seconds
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])     -- current timestamp in ms

-- Remove entries outside the window
redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window * 1000)

-- Count current entries
local count = redis.call('ZCARD', key)

if count < limit then
    -- Add this request
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window)
    return 1  -- allowed
else
    return 0  -- denied
end
`

type RedisRateLimiter struct {
    rdb    *redis.Client
    script *redis.Script
    window time.Duration
    limit  int
}

func (r *RedisRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now().UnixMilli()
    result, err := r.script.Run(ctx, r.rdb,
        []string{"ratelimit:" + key},
        int(r.window.Seconds()),
        r.limit,
        now,
    ).Int()
    if err != nil {
        // Fail open: if Redis is down, allow the request
        return true, err
    }
    return result == 1, nil
}
```

## Fixed Window vs. Sliding Window

```
Fixed Window: 100 req/min
              [  0s ........ 59s ] [ 60s ........ 119s ]
              100 requests → OK     100 more → OK
              But: 100 at t=59, 100 at t=61 = 200 in 2 seconds!

Sliding Window: always looks back exactly 60 seconds
                Much fairer, slightly more expensive.
```

## Adaptive Rate Limiting

The smartest approach: back off when the system is stressed.

```go
type AdaptiveRateLimiter struct {
    base     *RedisRateLimiter
    metrics  MetricsCollector
}

func (a *AdaptiveRateLimiter) effectiveLimit() int {
    p99Latency := a.metrics.P99Latency()
    errorRate  := a.metrics.ErrorRate()
    
    switch {
    case errorRate > 0.05 || p99Latency > 2*time.Second:
        return a.base.limit / 4  // severe throttle
    case errorRate > 0.01 || p99Latency > 500*time.Millisecond:
        return a.base.limit / 2  // moderate throttle
    default:
        return a.base.limit
    }
}
```

## Response Headers (Do This)

Always tell clients when to retry:

```go
func writeRateLimitHeaders(w http.ResponseWriter, remaining int, reset time.Time) {
    w.Header().Set("X-RateLimit-Limit", "100")
    w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(remaining))
    w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(reset.Unix(), 10))
    w.Header().Set("Retry-After", strconv.Itoa(int(time.Until(reset).Seconds())))
}
```

## Summary

| Algorithm | Pros | Cons | Use When |
|-----------|------|------|----------|
| Token Bucket | Smooth, allows bursts | In-memory only | Single instance |
| Fixed Window | Simple, fast | Boundary burst problem | Coarse limits |
| Sliding Window Log | Accurate | O(N) memory per client | Strict fairness |
| Sliding Window Counter | Approximate, cheap | Slight inaccuracy | High-volume APIs |
| Adaptive | Self-healing | Complex | Production SLO protection |
