---
layout: post
title: "Go Worker Pools, Job Queues, and Background Processing"
date: 2026-05-03 12:30:00 +0700
categories: golang backend concurrency
tags: [golang, worker-pool, concurrency, background-jobs, redis]
description: "Implementing safe background processing in Go: bounded worker pools, durable job queues with Asynq and Redis, cron scheduling, and graceful shutdown to avoid losing in-flight jobs."
---

Background processing is where Go shines. Goroutines are cheap (~2KB stack), but unbounded concurrency is still dangerous. Here's how to build robust background job systems.

## The Naive Approach (and Why It Breaks)

```go
// DO NOT DO THIS in production
func handleUpload(w http.ResponseWriter, r *http.Request) {
    file := parseUpload(r)
    go processFile(file) // no limit, no monitoring, fire-and-forget
    w.WriteHeader(http.StatusAccepted)
}
```

With 10,000 concurrent uploads, you spawn 10,000 goroutines doing heavy I/O. Memory explodes, the process OOMs.

## Bounded Worker Pool

```go
type WorkerPool struct {
    jobs    chan Job
    results chan Result
    wg      sync.WaitGroup
}

type Job struct {
    ID      string
    Payload any
    Handler func(ctx context.Context, payload any) error
}

func NewWorkerPool(numWorkers int, queueSize int) *WorkerPool {
    pool := &WorkerPool{
        jobs:    make(chan Job, queueSize),
        results: make(chan Result, queueSize),
    }
    pool.start(numWorkers)
    return pool
}

func (p *WorkerPool) start(numWorkers int) {
    for i := 0; i < numWorkers; i++ {
        p.wg.Add(1)
        go func() {
            defer p.wg.Done()
            for job := range p.jobs {
                ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
                err := job.Handler(ctx, job.Payload)
                cancel()
                p.results <- Result{JobID: job.ID, Err: err}
            }
        }()
    }
}

func (p *WorkerPool) Submit(job Job) error {
    select {
    case p.jobs <- job:
        return nil
    default:
        return ErrQueueFull // back-pressure: reject when queue is full
    }
}

func (p *WorkerPool) Shutdown() {
    close(p.jobs)
    p.wg.Wait()
    close(p.results)
}
```

## Durable Job Queue with Redis (Asynq)

In-memory queues lose jobs on restart. Use a durable queue backed by Redis:

```go
import "github.com/hibiken/asynq"

// Enqueue a job
func EnqueueEmailJob(client *asynq.Client, userID string) error {
    payload, _ := json.Marshal(map[string]string{"user_id": userID})
    task := asynq.NewTask("email:welcome", payload,
        asynq.MaxRetry(3),
        asynq.Timeout(30*time.Second),
        asynq.Queue("critical"),
    )
    _, err := client.Enqueue(task)
    return err
}

// Process the job
type EmailHandler struct{ mailer Mailer }

func (h *EmailHandler) ProcessTask(ctx context.Context, t *asynq.Task) error {
    var payload struct{ UserID string `json:"user_id"` }
    if err := json.Unmarshal(t.Payload(), &payload); err != nil {
        return fmt.Errorf("unmarshal: %w", err) // non-retryable
    }
    return h.mailer.SendWelcome(ctx, payload.UserID)
}

// Start the worker server
func StartWorkers() error {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: "localhost:6379"},
        asynq.Config{
            Concurrency: 20,
            Queues: map[string]int{
                "critical": 6,
                "default":  3,
                "low":      1,
            },
            ErrorHandler: asynq.ErrorHandlerFunc(func(ctx context.Context, task *asynq.Task, err error) {
                log.Printf("task %s failed: %v", task.Type(), err)
            }),
        },
    )

    mux := asynq.NewServeMux()
    mux.Handle("email:welcome", &EmailHandler{})

    return srv.Run(mux)
}
```

## Scheduled Jobs (Cron)

```go
// Asynq scheduler for periodic tasks
func StartScheduler() error {
    scheduler := asynq.NewScheduler(
        asynq.RedisClientOpt{Addr: "localhost:6379"},
        nil,
    )

    // Run every day at midnight UTC
    scheduler.Register("0 0 * * *",
        asynq.NewTask("reports:daily", nil, asynq.Queue("low")))

    // Run every 5 minutes
    scheduler.Register("*/5 * * * *",
        asynq.NewTask("metrics:flush", nil))

    return scheduler.Run()
}
```

## Graceful Shutdown

The most important part — don't lose in-flight jobs:

```go
func main() {
    pool := NewWorkerPool(20, 1000)
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    
    go func() {
        <-quit
        log.Println("shutting down: waiting for in-flight jobs...")
        
        // Stop accepting new work
        pool.Shutdown() // waits for all running goroutines to finish
        
        log.Println("all jobs complete, exiting")
        os.Exit(0)
    }()
    
    // Start the HTTP server, etc.
}
```

## Monitoring Worker Health

```go
var (
    jobsProcessed = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "worker_jobs_total",
        Help: "Total jobs processed",
    }, []string{"queue", "status"})
    
    jobDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "worker_job_duration_seconds",
        Buckets: []float64{.1, .5, 1, 5, 30, 60},
    }, []string{"task_type"})
    
    queueDepth = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Name: "worker_queue_depth",
        Help: "Current queue depth",
    }, []string{"queue"})
)
```

## Key Principles

1. **Always bound concurrency** — never `go func()` without a semaphore or pool
2. **Use durable queues** for anything you cannot afford to lose on restart
3. **Implement back-pressure** — reject work when at capacity, return 503
4. **Handle graceful shutdown** — in-flight jobs must complete before process exits
5. **Make jobs idempotent** — retries are inevitable; design for them
