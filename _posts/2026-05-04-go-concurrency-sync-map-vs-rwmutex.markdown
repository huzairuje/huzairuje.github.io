---
layout: post
title: "Concurrency in Go: sync.Map vs map + sync.RWMutex"
date: 2026-05-04 02:30:00 +0700
categories: golang backend concurrency
tags: [golang, concurrency, data-structures, performance]
description: "When to use Go's built-in map with an RWMutex vs the specialized sync.Map for concurrent access in backend applications."
---

Maps in Go are not safe for concurrent use. If one goroutine writes to a map while another reads from it, the program will crash with a fatal error: `concurrent map read and map write`.

To fix this, Go developers usually reach for one of two solutions: a standard `map` wrapped in a `sync.RWMutex`, or the specialized `sync.Map`. 

Choosing the wrong one will silently choke your application's performance.

## The Standard Approach: `map` + `sync.RWMutex`

This is the most common pattern and what you should reach for 90% of the time.

```go
type SafeCache struct {
    mu sync.RWMutex
    m  map[string]interface{}
}

func (c *SafeCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()         // Multiple readers can hold RLock concurrently
    defer c.mu.RUnlock()
    val, ok := c.m[key]
    return val, ok
}

func (c *SafeCache) Set(key string, val interface{}) {
    c.mu.Lock()          // Exclusive lock. Blocks all readers and writers.
    defer c.mu.Unlock()
    c.m[key] = val
}
```

**Pros:** Strong type safety (you define the exact map type), easy to understand, and very fast for workloads that have a balanced mix of reads and writes.

**Cons:** Lock contention. On multi-core machines, if hundreds of goroutines are constantly hammering the `RLock()`, the CPU cache invalidation surrounding the mutex state can become a massive bottleneck.

## The Specialized Tool: `sync.Map`

Introduced in Go 1.9, `sync.Map` is heavily optimized to avoid lock contention, but it uses `any` (interface{}) for keys and values, sacrificing compile-time type safety.

```go
var cache sync.Map

// Write
cache.Store("user:123", User{Name: "Alice"})

// Read
if val, ok := cache.Load("user:123"); ok {
    user := val.(User) // Type assertion required
    fmt.Println(user.Name)
}
```

Behind the scenes, `sync.Map` maintains two maps: a "read" map (accessed using atomic operations, lock-free) and a "dirty" map (protected by a mutex).

## When to use `sync.Map`

The official Go documentation states `sync.Map` is optimized for exactly two use cases:

### 1. Append-Only Caches (Write-Once, Read-Many)
If keys are written once but read thousands of times (like an in-memory cache of static configuration or a memoized function result), `sync.Map` excels. After the key migrates to the internal "read" map, subsequent reads are completely lock-free.

### 2. Disjoint Key Access
If multiple goroutines are reading, writing, and overwriting entries, but **each goroutine operates on its own distinct set of keys**, `sync.Map` minimizes cache contention.

## When NOT to use `sync.Map`

**Do not use `sync.Map` for heavy overwrite workloads.** 

If your backend is constantly updating the same keys (e.g., tracking a live counter `cache.Store("views", views+1)`), `sync.Map` performs terribly. Every update misses the fast path, falls back to the dirty map, and forces heavy mutex locking and memory allocation overhead.

## Summary Rule of Thumb

- **Default:** Use `map` + `sync.RWMutex`.
- **High Concurrency, 99% Reads, 1% Writes:** Use `sync.Map`.
- **Need Type Safety:** Use generic map + Mutex (Go 1.18+ makes this even easier to wrap into a reusable struct).
