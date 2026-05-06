---
layout: post
title: "Go Outbox Pattern: Stop Losing Events After Commit"
date: 2026-05-07 12:00:00 +0700
categories: golang backend distributed-systems
tags: [golang, outbox, events, transactions, kafka]
description: "Why publishing events directly after DB commit causes inconsistencies, and how the outbox pattern fixes it in Go services."
---

We had orders in the database that never produced downstream events.

No broker outage. No database corruption. Just a tiny crash window between commit and publish.

## The Inconsistency Window

Classic flow:

1. Write order row in SQL transaction.
2. Commit transaction.
3. Publish `OrderCreated` to Kafka.

If process dies between steps 2 and 3, the order exists forever without an event.

## Direct Publish Anti-Pattern

```go
tx.Commit()
if err := kafkaProducer.Publish("order.created", payload); err != nil {
    return err // too late: DB already committed
}
```

## Outbox Approach

Inside the same DB transaction, write both:

- Business row (`orders`)
- Outbox row (`outbox_events`, status `pending`)

A separate relay worker reads pending outbox rows and publishes safely.

```go
err := withTx(ctx, db, func(tx *sql.Tx) error {
    if err := insertOrder(tx, order); err != nil {
        return err
    }
    return insertOutbox(tx, OutboxEvent{
        Topic:   "order.created",
        Key:     order.ID,
        Payload: payloadJSON,
    })
})
```

## Operational Lessons

- Make consumer handlers idempotent (duplicates happen).
- Mark outbox rows as sent only after broker ack.
- Add dead-letter handling for poison payloads.
- Monitor outbox lag; lag is your hidden consistency debt.

## What Went Wrong in My Incident

- **What alerted first:** Support reported records present in DB but missing in downstream services.
- **What misled us:** Broker health and consumer lag looked normal, so messaging infra seemed innocent.
- **What confirmed root cause:** Tracing a single request showed process exit between DB commit and event publish.

The outbox pattern is boring infrastructure. That is exactly why it works.
