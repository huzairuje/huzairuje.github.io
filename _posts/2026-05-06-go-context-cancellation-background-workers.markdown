---
layout: post
title: "Go Context Cancellation: Why Your Workers Never Stop"
date: 2026-05-06 01:00:00 +0700
categories: golang backend concurrency
tags: [golang, context, workers, goroutines, concurrency]
description: "A production bug where background worker goroutines ignored cancellation and leaked across deploys."
---

Our service memory kept growing after every deploy.

Heap profiles showed old worker goroutines still alive long after shutdown.

## Leak Pattern

Workers were spawned with `context.Background()` instead of request/service lifecycle context:

```go
go runWorker(context.Background(), jobs) // detached forever
```

These workers ignored shutdown signals and kept polling, logging, and allocating.

## Correct Lifecycle Wiring

Tie worker context to the process context and respect `ctx.Done()`.

```go
func runWorker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return
        case job := <-jobs:
            handle(job)
        }
    }
}
```

At startup:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
go runWorker(ctx, jobs)
```

## Production Guardrails

- Expose `runtime.NumGoroutine()` metric.
- Add shutdown test asserting goroutine count returns near baseline.
- Never create long-lived goroutines from request handlers without explicit ownership.

## What Went Wrong in My Incident

- **What alerted first:** Memory and goroutine counts climbed after each deploy instead of resetting.
- **What misled us:** We initially blamed traffic growth and GC tuning.
- **What confirmed root cause:** Goroutine dumps showed long-lived workers rooted in `context.Background()` with no cancellation path.

In Go, every goroutine needs an owner and an exit path.
