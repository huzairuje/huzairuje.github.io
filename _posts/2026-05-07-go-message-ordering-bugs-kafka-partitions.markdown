---
layout: post
title: "Message Ordering Bugs in Go: Kafka Partitions Surprise"
date: 2026-05-07 01:00:00 +0700
categories: golang backend messaging
tags: [golang, kafka, ordering, events, distributed-systems]
description: "How incorrect partition keys in event-driven systems break ordering guarantees and corrupt downstream state."
---

We assumed event order was guaranteed. It was not.

Kafka preserves order only **within a partition**, not across all partitions.

## The Real Bug

Our producer used a random key for throughput:

```go
key := uuid.NewString() // bad for ordered entity updates
producer.Publish("invoice.events", key, payload)
```

Events for the same invoice landed in different partitions. Consumers received:

- `InvoicePaid`
- then `InvoiceCreated`

Downstream projections drifted and failed validation checks.

## The Fix

Partition by entity identity so all related events share a partition.

```go
key := invoiceID // stable key per aggregate/entity
producer.Publish("invoice.events", key, payload)
```

## Consumer Hardening

Even with proper partitioning, we added:

1. Version checks (`eventVersion` monotonic per entity)
2. Idempotency on `eventID`
3. Dead-letter queue for impossible transitions

## What Went Wrong in My Incident

- **What alerted first:** Projection mismatches appeared for a small subset of high-traffic entities.
- **What misled us:** Individual events were valid, so we suspected consumer logic before producer keys.
- **What confirmed root cause:** Partition/offset analysis showed same entity events split across partitions and arriving out of sequence.

Ordering bugs are nasty because each event is valid in isolation. The sequence is where corruption begins.
