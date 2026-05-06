---
layout: post
title: "Schema Drift Between Go Services: The Silent Contract Break"
date: 2026-05-08 10:30:00 +0700
categories: golang backend microservices
tags: [golang, schema, contracts, protobuf, compatibility]
description: "A real-world microservice failure caused by schema drift and how to enforce backward-compatible contracts."
---

Service A deployed fine. Service B did not crash. Yet checkout failed for 12% of traffic.

Root cause: schema drift.

## What Drift Looked Like

One service started sending a new enum value and made an old field effectively required. Older consumers treated it as unknown and dropped business-critical paths.

No compile error because each service had its own generated artifacts pinned to different commits.

## The Fixes That Worked

1. **Single source of truth** for contracts (dedicated schema repo/module).
2. **Compatibility checks in CI** (backward + forward compatibility).
3. **Expand/contract migrations**: introduce new fields as optional first.
4. **Runtime metrics** for unknown enums and decode failures.

## Defensive Go Handling

```go
switch event.Status {
case pb.Status_STATUS_CREATED, pb.Status_STATUS_CONFIRMED:
    // normal flow
default:
    // unknown enum should be observable, not silently ignored
    metrics.UnknownStatus.Inc()
    return fmt.Errorf("unsupported status: %v", event.Status)
}
```

## What Went Wrong in My Incident

- **What alerted first:** Checkout success rate dropped without any crash or obvious error spike.
- **What misled us:** Both services were healthy, so we investigated infra before contracts.
- **What confirmed root cause:** Payload inspection revealed new enum/field semantics consumed by older generated clients.

Distributed systems fail at boundaries. Schemas are one of the sharpest boundaries you own.
