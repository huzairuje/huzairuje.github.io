---
layout: post
title: "Observability in Go: Structured Logging, Metrics, and Distributed Tracing"
date: 2026-05-03 11:30:00 +0700
categories: golang backend observability
tags: [golang, observability, opentelemetry, prometheus, logging]
description: "The three pillars of observability for Go backends: structured logging with slog, Prometheus metrics with golden signals, and distributed tracing with OpenTelemetry."
---

You cannot debug what you cannot observe. In distributed Go systems, the three pillars of observability — logs, metrics, and traces — are non-negotiable. Here's how to implement them correctly from day one.

## Structured Logging with slog

Go 1.21 added `log/slog` to the standard library. Use it:

```go
import "log/slog"

func NewLogger(env string) *slog.Logger {
    var handler slog.Handler
    
    if env == "production" {
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelInfo,
            ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
                if a.Key == slog.TimeKey {
                    a.Value = slog.StringValue(a.Value.Time().UTC().Format(time.RFC3339Nano))
                }
                return a
            },
        })
    } else {
        handler = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelDebug,
        })
    }
    
    return slog.New(handler)
}

// Usage — structured, not string formatting
logger.InfoContext(ctx, "order placed",
    slog.String("order_id", order.ID),
    slog.String("user_id", order.UserID),
    slog.Float64("total", order.Total),
    slog.Int("item_count", len(order.Items)),
)
```

## Injecting Trace IDs into Logs

The killer feature: correlate logs with traces automatically.

```go
// Middleware: extract trace ID from OTEL span and add to logger in context
func WithTraceLogger(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            span := trace.SpanFromContext(r.Context())
            spanCtx := span.SpanContext()
            
            enrichedLogger := logger.With(
                slog.String("trace_id", spanCtx.TraceID().String()),
                slog.String("span_id", spanCtx.SpanID().String()),
                slog.String("request_id", r.Header.Get("X-Request-ID")),
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
            )
            
            ctx := context.WithValue(r.Context(), loggerKey{}, enrichedLogger)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func LoggerFromCtx(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(loggerKey{}).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}
```

## Metrics with Prometheus

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests by method, path, and status",
    }, []string{"method", "path", "status"})
    
    httpRequestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration in seconds",
        Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
    }, []string{"method", "path"})
    
    activeConnections = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "active_connections",
        Help: "Current number of active connections",
    })
)

func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        activeConnections.Inc()
        defer activeConnections.Dec()
        
        rw := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(rw, r)
        
        duration := time.Since(start).Seconds()
        path := sanitizePath(r.URL.Path) // normalize /users/123 → /users/{id}
        
        httpRequestsTotal.WithLabelValues(r.Method, path, strconv.Itoa(rw.statusCode)).Inc()
        httpRequestDuration.WithLabelValues(r.Method, path).Observe(duration)
    })
}
```

## Distributed Tracing with OpenTelemetry

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/trace"
)

func InitTracer(ctx context.Context, serviceName string) (func(), error) {
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint("otel-collector:4318"),
        otlptracehttp.WithInsecure(),
    )
    if err != nil { return nil, err }
    
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithSampler(trace.TraceIDRatioBased(0.1)), // sample 10% in prod
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(serviceName),
            semconv.DeploymentEnvironmentKey.String(os.Getenv("ENV")),
        )),
    )
    
    otel.SetTracerProvider(tp)
    
    return func() { tp.Shutdown(ctx) }, nil
}

// Usage in handlers
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx, span := otel.Tracer("order-service").Start(r.Context(), "CreateOrder")
    defer span.End()
    
    span.SetAttributes(
        attribute.String("user.id", getUserID(ctx)),
        attribute.Int("items.count", len(req.Items)),
    )
    
    order, err := h.svc.PlaceOrder(ctx, req)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    span.SetAttributes(attribute.String("order.id", order.ID))
    json.NewEncoder(w).Encode(order)
}
```

## SLO Alerting

Define SLOs before writing the first line of code:

```yaml
# Prometheus alerting rules
groups:
  - name: slo_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 1% SLO"
          
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency above 1s SLO for {{ $labels.path }}"
```

## The Golden Signals

Monitor these four metrics for every service:

| Signal | What to Track |
|--------|--------------|
| **Latency** | P50, P95, P99 — not average |
| **Traffic** | Requests per second per endpoint |
| **Errors** | 5xx rate, panic rate, timeout rate |
| **Saturation** | CPU, memory, goroutine count, DB pool wait time |

Observability is not a feature — it's the foundation that makes all other features debuggable.
