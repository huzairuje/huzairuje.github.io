---
layout: post
title: "Monolith vs Microservices in Go: Choosing the Right Architecture"
date: 2026-05-03 08:00:00 +0700
categories: golang backend architecture
tags: [golang, monolith, microservices, architecture, backend]
description: "A practical framework for choosing between monolith and microservices in Go. Covers the strangler fig pattern, go work for multi-module repos, and a real-world decision checklist."
---

Every Go backend project starts with the same question: **monolith or microservices?** The wrong answer at the start costs months of refactoring. Here's a practical framework based on real production experience.

## The Monolith First Principle

Despite the hype around microservices, most successful distributed systems started as monoliths. The Go standard library actually makes a well-structured monolith incredibly pleasant to work with.

```go
// A clean monolith: one binary, multiple internal packages
package main

import (
    "myapp/internal/orders"
    "myapp/internal/inventory"
    "myapp/internal/billing"
    "myapp/internal/notifications"
)

func main() {
    db := connectDB()
    
    orderSvc    := orders.NewService(db)
    inventorySvc := inventory.NewService(db)
    billingSvc  := billing.NewService(db)
    notifySvc   := notifications.NewService(db)
    
    // All in one process. Simple. Debuggable.
    router := setupRouter(orderSvc, inventorySvc, billingSvc, notifySvc)
    router.ListenAndServe(":8080")
}
```

This is not "legacy" — it's pragmatic. Shopify ran as a Rails monolith until ~$1B GMV.

## When Monoliths Break Down

You'll feel the pain at these inflection points:

| Signal | What It Means |
|--------|--------------|
| Different team ownership per domain | Conway's Law pressure |
| Independent scaling needs | Orders vs. analytics have very different load profiles |
| Different deployment cadences | Payment team ships 10x/day; core ships weekly |
| Language/runtime mismatch | ML inference needs Python; core is Go |

## Splitting Thoughtfully: The Strangler Fig Pattern

Don't rewrite — strangle.

```go
// Step 1: Extract the domain boundary with an interface
type InventoryService interface {
    Reserve(ctx context.Context, skuID string, qty int) error
    Release(ctx context.Context, reservationID string) error
}

// Step 2: Local implementation (still in-process)
type localInventory struct { db *sql.DB }

// Step 3: Remote implementation (HTTP/gRPC) – swap without changing callers
type remoteInventory struct { client inventorypb.InventoryClient }

func (r *remoteInventory) Reserve(ctx context.Context, skuID string, qty int) error {
    _, err := r.client.Reserve(ctx, &inventorypb.ReserveRequest{
        SkuId:    skuID,
        Quantity: int32(qty),
    })
    return err
}
```

## The Go-Specific Advantage

Go's compile-time dependency graph makes module boundaries explicit. Use `go work` for multi-module monorepos:

```bash
# go.work — keep all services in sync without publishing to a registry
go 1.22

use (
    ./services/orders
    ./services/inventory
    ./services/billing
    ./shared/proto
    ./shared/telemetry
)
```

## Decision Checklist

Before extracting a microservice, confirm:
- [ ] The domain boundary is stable (no cross-cutting changes in last 3 months)
- [ ] You have automated integration tests covering the boundary
- [ ] You have distributed tracing in place (you'll need it)
- [ ] The team owning it is ≥2 engineers
- [ ] The latency budget for a network hop is acceptable

**Rule of thumb**: Start monolith, extract when you *feel* the friction, not when you *anticipate* it.
