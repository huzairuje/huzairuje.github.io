---
layout: post
title: "Distributed Transactions in Go: Sagas, Outbox, and 2PC"
date: 2026-05-03 08:30:00 +0700
categories: golang backend distributed-systems
tags: [golang, distributed-transactions, saga, outbox, 2pc]
description: "How to handle distributed transactions in Go without losing data. Deep dive into the Saga pattern, Transactional Outbox, Two-Phase Commit, and compensating transactions."
---

The hardest problem in distributed systems isn't consensus — it's money changing hands across service boundaries. Here's how real Go backends handle it.

## Why You Can't Use Database Transactions Across Services

```go
// This looks reasonable. It is not.
func PlaceOrder(ctx context.Context, order Order) error {
    // Service A: deduct inventory
    if err := inventoryClient.Reserve(ctx, order.Items); err != nil {
        return err
    }
    // Service B: charge payment — what if this fails?
    if err := paymentClient.Charge(ctx, order.Total); err != nil {
        // NOW WHAT? Inventory is reserved, payment failed.
        // You just created an inconsistency.
        inventoryClient.Release(ctx, order.Items) // this can also fail!
        return err
    }
    return nil
}
```

This is the **dual-write problem**. No distributed transaction can make this atomic without significant tradeoffs.

## Pattern 1: Saga (Choreography)

Each service publishes events; downstream services react and publish their own compensating events on failure.

```go
// Order service publishes an event
type OrderPlacedEvent struct {
    OrderID   string    `json:"order_id"`
    Items     []Item    `json:"items"`
    Total     float64   `json:"total"`
    CreatedAt time.Time `json:"created_at"`
}

func (s *OrderService) PlaceOrder(ctx context.Context, req PlaceOrderRequest) error {
    // 1. Save order in PENDING state (local DB transaction)
    tx, _ := s.db.BeginTx(ctx, nil)
    order, err := s.repo.CreateOrder(ctx, tx, req)
    if err != nil { tx.Rollback(); return err }

    // 2. Publish event in same transaction (Outbox pattern — see below)
    err = s.outbox.Publish(ctx, tx, "order.placed", OrderPlacedEvent{
        OrderID: order.ID,
        Items:   order.Items,
        Total:   order.Total,
    })
    if err != nil { tx.Rollback(); return err }

    return tx.Commit()
}
```

Inventory service listens and either reserves stock (publishing `inventory.reserved`) or publishes `inventory.failed` which triggers compensation.

## Pattern 2: The Transactional Outbox

The critical piece that makes sagas reliable: **write the event to the DB in the same transaction as your state change**, then a relay picks it up and publishes to the broker.

```go
type OutboxEvent struct {
    ID          string    `db:"id"`
    AggregateID string    `db:"aggregate_id"`
    Topic       string    `db:"topic"`
    Payload     []byte    `db:"payload"`
    PublishedAt *time.Time `db:"published_at"` // nil = not yet published
    CreatedAt   time.Time `db:"created_at"`
}

// Outbox relay — runs as a background goroutine
func (r *OutboxRelay) Run(ctx context.Context) {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            r.publishPending(ctx)
        }
    }
}

func (r *OutboxRelay) publishPending(ctx context.Context) {
    events, err := r.repo.FindUnpublished(ctx, 100)
    if err != nil { return }

    for _, evt := range events {
        if err := r.broker.Publish(ctx, evt.Topic, evt.Payload); err != nil {
            continue // retry next tick
        }
        r.repo.MarkPublished(ctx, evt.ID)
    }
}
```

Schema:
```sql
CREATE TABLE outbox_events (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id TEXT NOT NULL,
    topic        TEXT NOT NULL,
    payload      JSONB NOT NULL,
    published_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_outbox_unpublished ON outbox_events (created_at)
    WHERE published_at IS NULL;
```

## Pattern 3: Two-Phase Commit (and why you probably shouldn't)

2PC requires all participants to agree before committing. In Go, this means coordinating a distributed lock across services:

```go
// Phase 1: Prepare (all services lock their resources)
// Phase 2: Commit (coordinator tells everyone to finalize)
```

**Why to avoid it**: blocking protocol, coordinator SPOF, terrible latency. Use it only for XA transactions with your DB and message broker (Kafka supports it).

## Choosing the Right Pattern

| Scenario | Recommended Pattern |
|----------|-------------------|
| E-commerce checkout | Saga + Outbox |
| Bank transfer within one DB | Local ACID transaction |
| Cross-bank transfer | Saga with idempotent steps |
| Batch job across services | Orchestration saga (e.g. Temporal) |
| Strong consistency required | Reconsider service boundary |

## Compensating Transactions

Every forward step must have a compensating step:

```go
type SagaStep struct {
    Name       string
    Execute    func(ctx context.Context) error
    Compensate func(ctx context.Context) error
}

func RunSaga(ctx context.Context, steps []SagaStep) error {
    executed := make([]SagaStep, 0, len(steps))
    
    for _, step := range steps {
        if err := step.Execute(ctx); err != nil {
            // Roll back in reverse order
            for i := len(executed) - 1; i >= 0; i-- {
                executed[i].Compensate(ctx) // best-effort, log failures
            }
            return fmt.Errorf("saga failed at step %s: %w", step.Name, err)
        }
        executed = append(executed, step)
    }
    return nil
}
```

The saga pattern doesn't give you ACID — it gives you **eventual consistency with compensations**. Design your UX accordingly: show "pending" states, support manual review queues for stuck sagas.
