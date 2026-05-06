---
layout: post
title: "Graceful Shutdown in Go: The In-Flight Request Trap"
date: 2026-05-07 09:00:00 +0700
categories: golang backend reliability
tags: [golang, shutdown, http, reliability, backend]
description: "How a Go service can lose user requests during deploys, and the shutdown sequence that prevents dropped in-flight work."
---

We once shipped a harmless-looking deploy and got a wave of user complaints: successful button clicks, missing results.

No panic. No crash. Just silently dropped requests.

## What Actually Happened

Our Kubernetes pod received `SIGTERM`, but the Go process exited too fast:

1. Load balancer still routed traffic for a short window.
2. Existing handlers were still running.
3. Process terminated before handlers finished.

From the client side, this looked random and impossible to reproduce.

## The Wrong Shutdown Pattern

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: routes()}
    go srv.ListenAndServe()

    // Wait for signal...
    <-sigCh
    os.Exit(0) // Abrupt exit: active requests are cut off
}
```

## The Production-Safe Pattern

Use `http.Server.Shutdown` with a timeout, and stop accepting new traffic first.

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: routes()}

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server failed: %v", err)
        }
    }()

    <-sigCh

    ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("graceful shutdown failed: %v", err)
        _ = srv.Close() // hard close fallback
    }
}
```

## Two Extra Details That Matter

- **Readiness first:** return non-ready before shutdown so new requests stop.
- **Background jobs:** stop consumers/workers too, or they may keep mutating state while HTTP is draining.

## What Went Wrong in My Incident

- **What alerted first:** A spike in client retries right after deploy start.
- **What misled us:** App logs looked clean, so we blamed transient network issues.
- **What confirmed root cause:** Correlating pod termination timestamps with failed requests showed traffic was still routed during shutdown.

Graceful shutdown is not a polish feature. It is data integrity during deploys.
