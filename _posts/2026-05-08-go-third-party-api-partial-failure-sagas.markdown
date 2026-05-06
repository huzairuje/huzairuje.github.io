---
layout: post
title: "Go and Third-Party APIs: Surviving Partial Failures"
date: 2026-05-08 16:00:00 +0700
categories: golang backend distributed-systems
tags: [golang, saga, compensation, api, failures]
description: "How to design Go backend workflows that remain consistent when third-party API calls partially fail."
---

Payment succeeded. Inventory reserved. Shipping label creation failed.

Welcome to partial failure: the normal state of distributed workflows.

## Why This Hurts

If you model a multi-step workflow as one happy-path transaction, you end up with stranded state when any external step fails.

## Practical Saga Pattern

Model each step with a forward action and a compensation action:

1. Reserve inventory
2. Charge payment
3. Create shipment

If step 3 fails, compensate:

- Refund payment
- Release inventory

## Go Structure

```go
type Step struct {
    Do   func(context.Context) error
    Undo func(context.Context) error
}

func RunSaga(ctx context.Context, steps []Step) error {
    completed := make([]Step, 0, len(steps))
    for _, s := range steps {
        if err := s.Do(ctx); err != nil {
            for i := len(completed) - 1; i >= 0; i-- {
                _ = completed[i].Undo(ctx)
            }
            return err
        }
        completed = append(completed, s)
    }
    return nil
}
```

## Production Tips

- Persist saga state so recovery survives process restarts.
- Make compensations idempotent.
- Alert on stuck sagas, not just failed requests.

## What Went Wrong in My Incident

- **What alerted first:** Support tickets reported failed orders with successful payment captures.
- **What misled us:** Request status showed failure, so we assumed no side effects were committed.
- **What confirmed root cause:** Step-by-step audit logs exposed completed upstream actions without matching compensations.

Strong systems are not the ones that avoid partial failure. They are the ones designed to recover from it.
