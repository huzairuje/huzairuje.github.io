---
layout: post
title: "Event-Driven Architecture in Go with Kafka"
date: 2026-05-03 10:30:00 +0700
categories: golang backend event-driven
tags: [golang, kafka, event-driven, messaging, backend]
description: "Building reliable event-driven systems in Go with Kafka. Covers franz-go producer/consumer setup, dead letter queues, idempotency, and schema evolution."
---

Event-driven architecture (EDA) decouples producers from consumers, enabling independent scaling and resilience. Go's concurrency model makes it a natural fit for building Kafka-based event pipelines.

## Core Concepts

In EDA, services communicate by **publishing facts** ("an order was placed") rather than commands ("place this order"). This inversion of control is powerful but requires discipline.

```
Producer: Order Service
    ↓ publishes "order.placed"
    
Consumer A: Inventory Service  → reserves stock
Consumer B: Notification Service → sends confirmation email
Consumer C: Analytics Service → updates dashboards

All independently, all at their own pace.
```

## Producing Events Reliably with franz-go

```go
import "github.com/twmb/franz-go/pkg/kgo"

type EventProducer struct {
    client *kgo.Client
}

func NewEventProducer(brokers []string) (*EventProducer, error) {
    client, err := kgo.NewClient(
        kgo.SeedBrokers(brokers...),
        kgo.RequiredAcks(kgo.AllISRAcks()),  // wait for all in-sync replicas
        kgo.RecordPartitioner(kgo.StickyKeyPartitioner(nil)),
        kgo.ProducerBatchMaxBytes(1_000_000),
        kgo.WithLogger(kgo.BasicLogger(os.Stderr, kgo.LogLevelInfo, nil)),
    )
    if err != nil { return nil, err }
    return &EventProducer{client: client}, nil
}

func (p *EventProducer) Publish(ctx context.Context, topic string, key string, payload any) error {
    data, err := json.Marshal(payload)
    if err != nil { return fmt.Errorf("marshal: %w", err) }
    
    record := &kgo.Record{
        Topic: topic,
        Key:   []byte(key),
        Value: data,
        Headers: []kgo.RecordHeader{
            {Key: "content-type", Value: []byte("application/json")},
            {Key: "event-version", Value: []byte("v1")},
            {Key: "produced-at", Value: []byte(time.Now().UTC().Format(time.RFC3339))},
        },
    }
    
    // Synchronous produce — use ProduceSync for guaranteed delivery
    results := p.client.ProduceSync(ctx, record)
    return results.FirstErr()
}
```

## Consuming Events with Proper Error Handling

```go
type EventConsumer struct {
    client   *kgo.Client
    handlers map[string]EventHandler
}

type EventHandler func(ctx context.Context, record *kgo.Record) error

func (c *EventConsumer) Run(ctx context.Context) error {
    for {
        fetches := c.client.PollFetches(ctx)
        if fetches.IsClientClosed() {
            return nil
        }
        
        fetches.EachError(func(t string, p int32, err error) {
            log.Printf("topic=%s partition=%d error=%v", t, p, err)
        })
        
        fetches.EachRecord(func(record *kgo.Record) {
            if err := c.processRecord(ctx, record); err != nil {
                // Decision: retry, dead-letter, or skip?
                c.handleProcessingError(ctx, record, err)
            }
        })
        
        if err := c.client.CommitUncommittedOffsets(ctx); err != nil {
            log.Printf("commit failed: %v", err)
        }
    }
}

func (c *EventConsumer) processRecord(ctx context.Context, record *kgo.Record) error {
    handler, ok := c.handlers[record.Topic]
    if !ok {
        return nil // unknown topic, skip
    }
    
    // Add deadline per message
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    return handler(ctx, record)
}
```

## Dead Letter Queue

Never lose a message. Route failed messages to a DLQ for later inspection:

```go
func (c *EventConsumer) handleProcessingError(ctx context.Context, record *kgo.Record, err error) {
    dlqRecord := &kgo.Record{
        Topic: record.Topic + ".dlq",
        Key:   record.Key,
        Value: record.Value,
        Headers: append(record.Headers,
            kgo.RecordHeader{Key: "dlq-error", Value: []byte(err.Error())},
            kgo.RecordHeader{Key: "dlq-original-topic", Value: []byte(record.Topic)},
            kgo.RecordHeader{Key: "dlq-original-offset", Value: []byte(strconv.FormatInt(record.Offset, 10))},
            kgo.RecordHeader{Key: "dlq-time", Value: []byte(time.Now().UTC().Format(time.RFC3339))},
        ),
    }
    
    results := c.producer.ProduceSync(ctx, dlqRecord)
    if err := results.FirstErr(); err != nil {
        log.Printf("CRITICAL: failed to write to DLQ: %v", err)
        // At this point, manual intervention may be needed
    }
}
```

## Idempotency: The Critical Consumer Property

```go
type OrderEventHandler struct {
    db          *pgxpool.Pool
    idempotency IdempotencyStore
}

func (h *OrderEventHandler) Handle(ctx context.Context, record *kgo.Record) error {
    eventID := getHeader(record, "event-id")
    
    // Check idempotency key before processing
    processed, err := h.idempotency.IsProcessed(ctx, eventID)
    if err != nil { return err }
    if processed {
        return nil // already handled, skip
    }
    
    var event OrderPlacedEvent
    if err := json.Unmarshal(record.Value, &event); err != nil {
        return fmt.Errorf("unmarshal: %w", err) // DLQ this
    }
    
    tx, err := h.db.Begin(ctx)
    if err != nil { return err }
    
    // Process the event
    if err := h.processOrderPlaced(ctx, tx, event); err != nil {
        tx.Rollback(ctx)
        return err
    }
    
    // Mark as processed in same transaction
    if err := h.idempotency.MarkProcessed(ctx, tx, eventID); err != nil {
        tx.Rollback(ctx)
        return err
    }
    
    return tx.Commit(ctx)
}
```

## Consumer Group Lag Monitoring

```go
// Alert when consumer lag exceeds threshold
func MonitorConsumerLag(ctx context.Context, client *kgo.Client, maxLag int64) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        lag, err := client.Lag(ctx)
        if err != nil { continue }
        
        for _, groupLag := range lag {
            for _, topicLag := range groupLag.Topics {
                for _, partitionLag := range topicLag.Partitions {
                    if partitionLag.Lag > maxLag {
                        metrics.ConsumerLag.Set(float64(partitionLag.Lag),
                            topicLag.Topic, strconv.Itoa(int(partitionLag.Partition)))
                        // Trigger alert
                    }
                }
            }
        }
    }
}
```

## Schema Evolution

Use a schema registry or at minimum version your events:

```go
type EventEnvelope struct {
    ID        string          `json:"id"`
    Version   int             `json:"version"`
    EventType string          `json:"event_type"`
    OccurredAt time.Time      `json:"occurred_at"`
    Payload   json.RawMessage `json:"payload"`
}

// Consumer handles multiple versions
func (h *Handler) Dispatch(env EventEnvelope) error {
    switch env.Version {
    case 1:
        var p PayloadV1
        json.Unmarshal(env.Payload, &p)
        return h.handleV1(p)
    case 2:
        var p PayloadV2
        json.Unmarshal(env.Payload, &p)
        return h.handleV2(p)
    default:
        return fmt.Errorf("unsupported event version: %d", env.Version)
    }
}
```

Event-driven systems trade synchronous simplicity for async resilience. The overhead is worth it once your system needs to scale beyond a single deployment unit.
