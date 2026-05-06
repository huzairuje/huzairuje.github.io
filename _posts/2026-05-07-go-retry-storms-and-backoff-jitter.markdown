---
layout: post
title: "Retry Storms in Go: When Resilience Becomes an Outage"
date: 2026-05-07 07:00:00 +0700
categories: golang backend resilience
tags: [golang, retries, backoff, jitter, resilience]
description: "A practical guide to preventing retry storms in Go services with bounded retries, jitter, and retry budgets."
---

Retries are supposed to save your system.

In one incident, ours took it down faster.

## The Failure Pattern

An upstream dependency slowed down. Every service layer retried three times:

- API gateway retried
- Service retried
- Repository client retried

One failed request turned into a burst of duplicated traffic. CPU spiked, queue depth exploded, and the upstream never recovered.

## The Buggy Retry Logic

```go
for i := 0; i < 3; i++ {
    err := callUpstream(ctx, req)
    if err == nil {
        return nil
    }
    time.Sleep(100 * time.Millisecond) // fixed interval, synchronized herd
}
```

## Safer Retry Strategy

Use exponential backoff with jitter, strict caps, and only retry transient errors.

```go
base := 100 * time.Millisecond
max := 2 * time.Second

for i := 0; i < 4; i++ {
    err := callUpstream(ctx, req)
    if err == nil {
        return nil
    }
    if !isTransient(err) {
        return err
    }

    backoff := minDuration(max, base*time.Duration(1<<i))
    jitter := time.Duration(rand.Int63n(int64(backoff / 2)))
    sleep := backoff/2 + jitter

    select {
    case <-time.After(sleep):
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

## Rules We Now Use in Production

1. Retry only idempotent operations unless protected by idempotency keys.
2. Never stack retries at every layer.
3. Enforce retry budgets so failure paths cannot exceed normal traffic by large factors.
4. Expose retry metrics (`attempt_count`, `retry_reason`) so storms are visible early.

## What Went Wrong in My Incident

- **What alerted first:** Upstream latency increased, followed by a sudden request-rate surge.
- **What misled us:** We assumed traffic growth, not retry amplification, caused the spike.
- **What confirmed root cause:** Attempt-level metrics showed multiple retry layers multiplying a single failure path.

Retries are a multiplier. Make sure they multiply stability, not pressure.
