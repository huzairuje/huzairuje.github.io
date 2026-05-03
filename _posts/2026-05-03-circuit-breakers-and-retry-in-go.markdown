---
layout: post
title: "Building Resilient Go Services: Circuit Breakers and Retry Strategies"
date: 2026-05-03 09:30:00 +0700
categories: golang backend resilience
tags: [golang, circuit-breaker, retry, resilience, backend]
description: "Building resilient Go services with exponential backoff, jitter, circuit breakers, and bulkhead isolation. Includes a full resilient HTTP client implementation."
---

In distributed systems, **partial failure is the norm**. A downstream service being slow is more dangerous than it being completely down. Circuit breakers and retry strategies are your first line of defense.

## The Retry Anti-Pattern

```go
// Don't do this — it's a DDoS on your own services
func callExternalService(ctx context.Context, req Request) (Response, error) {
    for i := 0; i < 10; i++ {
        resp, err := client.Do(req)
        if err == nil {
            return resp, nil
        }
        time.Sleep(time.Second) // constant delay — thundering herd!
    }
    return Response{}, errors.New("all retries failed")
}
```

If 1000 requests fail at the same time and all retry with a 1-second sleep, you get a thundering herd at t=1s.

## Exponential Backoff with Jitter

```go
type RetryConfig struct {
    MaxAttempts     int
    InitialDelay    time.Duration
    MaxDelay        time.Duration
    Multiplier      float64
    JitterFraction  float64
}

func WithRetry(ctx context.Context, cfg RetryConfig, fn func() error) error {
    delay := cfg.InitialDelay
    
    for attempt := 1; attempt <= cfg.MaxAttempts; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }
        
        // Don't retry on context cancellation or non-retryable errors
        if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
            return err
        }
        if isNonRetryable(err) {
            return err
        }
        
        if attempt == cfg.MaxAttempts {
            return fmt.Errorf("after %d attempts: %w", attempt, err)
        }
        
        // Jitter: add ±jitterFraction of delay
        jitter := time.Duration(float64(delay) * cfg.JitterFraction * (rand.Float64()*2 - 1))
        sleepFor := delay + jitter
        if sleepFor > cfg.MaxDelay {
            sleepFor = cfg.MaxDelay
        }
        
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(sleepFor):
        }
        
        delay = time.Duration(float64(delay) * cfg.Multiplier)
    }
    return nil
}

func isNonRetryable(err error) bool {
    var apiErr *APIError
    if errors.As(err, &apiErr) {
        // 4xx errors (except 429) are not worth retrying
        return apiErr.StatusCode >= 400 && apiErr.StatusCode < 500 && apiErr.StatusCode != 429
    }
    return false
}
```

## Circuit Breaker Pattern

Three states: **Closed** (normal) → **Open** (failing, reject all) → **Half-Open** (probe if recovered).

```go
type State int

const (
    StateClosed   State = iota // normal operation
    StateOpen                  // failing, reject fast
    StateHalfOpen              // testing recovery
)

type CircuitBreaker struct {
    mu              sync.Mutex
    state           State
    failureCount    int
    successCount    int
    lastFailureTime time.Time
    
    maxFailures     int
    timeout         time.Duration  // time in Open before trying Half-Open
    successThreshold int           // successes needed to close from Half-Open
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    
    switch cb.state {
    case StateOpen:
        if time.Since(cb.lastFailureTime) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successCount = 0
        } else {
            cb.mu.Unlock()
            return ErrCircuitOpen
        }
    }
    
    cb.mu.Unlock()
    
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failureCount++
        cb.lastFailureTime = time.Now()
        if cb.failureCount >= cb.maxFailures {
            cb.state = StateOpen
        }
        return err
    }
    
    // Success
    cb.failureCount = 0
    if cb.state == StateHalfOpen {
        cb.successCount++
        if cb.successCount >= cb.successThreshold {
            cb.state = StateClosed
        }
    }
    return nil
}
```

## Combining Both: The Resilient Client

```go
type ResilientClient struct {
    httpClient *http.Client
    breaker    *CircuitBreaker
    retryCfg   RetryConfig
}

func (c *ResilientClient) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
    var resp *http.Response
    
    err := WithRetry(ctx, c.retryCfg, func() error {
        return c.breaker.Execute(func() error {
            var err error
            resp, err = c.httpClient.Do(req.WithContext(ctx))
            if err != nil {
                return err
            }
            if resp.StatusCode >= 500 {
                resp.Body.Close()
                return fmt.Errorf("server error: %d", resp.StatusCode)
            }
            return nil
        })
    })
    
    return resp, err
}
```

## Bulkhead Pattern

Isolate critical paths from being starved by non-critical ones:

```go
type Bulkhead struct {
    semaphore chan struct{}
}

func NewBulkhead(maxConcurrent int) *Bulkhead {
    return &Bulkhead{semaphore: make(chan struct{}, maxConcurrent)}
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.semaphore <- struct{}{}:
        defer func() { <-b.semaphore }()
        return fn()
    case <-ctx.Done():
        return ctx.Err()
    default:
        return ErrBulkheadFull // shed load immediately
    }
}
```

## Real-World Config

```go
var paymentBreakerConfig = CircuitBreakerConfig{
    MaxFailures:      5,              // open after 5 consecutive failures
    Timeout:          30 * time.Second, // stay open for 30s
    SuccessThreshold: 2,              // need 2 successes to close
}

var paymentRetryConfig = RetryConfig{
    MaxAttempts:    3,
    InitialDelay:   100 * time.Millisecond,
    MaxDelay:       2 * time.Second,
    Multiplier:     2.0,
    JitterFraction: 0.25,
}
```

Combine with **timeouts at every layer**: connection timeout, read timeout, and a per-request context deadline.
