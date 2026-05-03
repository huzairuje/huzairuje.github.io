---
layout: post
title: "Streaming AI Responses via gRPC + SSE: Python FastAPI → Go Backend → Browser"
date: 2026-05-05 01:00:00 +0700
categories: [Backend, AI]
tags: [grpc, sse, python, go, fastapi]
---

When building an AI-powered feature, you often end up with a Python service running the model (because the ML ecosystem lives in Python) and a Go backend serving your API. The challenge: how do you stream AI-generated tokens all the way from Python to the browser in real time?

This post walks through the full stack: gRPC server-side streaming from Python FastAPI to Go, then Server-Sent Events (SSE) from Go to the browser — including heartbeats so long-running AI jobs don't drop the connection.

## Architecture Overview

`Browser ──(SSE/HTTP)──► Go Backend ──(gRPC stream)──► Python FastAPI (AI)`

Three layers, two protocols:
- **gRPC between Go and Python** — HTTP/2, strongly typed, efficient binary framing
- **SSE between Go and the browser** — simple HTTP/1.1 text stream, natively supported in browsers with `EventSource`

The Python service runs the AI model and yields tokens one by one as they're generated. Go receives them over a gRPC stream and forwards each token to the browser as an SSE event.

## Step 1: Define the Proto

The key detail here is `returns (stream GenerateResponse)` — this is a server-side streaming RPC, not the unary RPC you see in most gRPC tutorials. Without streaming, you'd have to wait for the entire AI response before sending anything back.

```protobuf
// proto/ai.proto
syntax = "proto3";
package ai;
option go_package = "yourmodule/proto/ai";

message GenerateRequest {
  string prompt     = 1;
  string session_id = 2;
}

message GenerateResponse {
  string token = 1;
  bool   done  = 2;
}

service AIService {
  // Server-side streaming RPC — Python yields, Go reads
  rpc Generate(GenerateRequest) returns (stream GenerateResponse);
}
```

Generate stubs for both sides:

```bash
# Go
protoc \
  --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  proto/ai.proto

# Python
python -m grpc_tools.protoc \
  -I ./proto \
  --python_out=. \
  --grpc_python_out=. \
  proto/ai.proto
```

## Step 2: Python FastAPI — gRPC Streaming Server

The Python service acts as the gRPC server. It runs the AI model and yields tokens as they're produced. The `async for` loop is important here — you want to yield each token immediately rather than buffering the entire response.

```python
# server.py
import asyncio
import grpc
from grpc import aio

import ai_pb2
import ai_pb2_grpc

# Example with Anthropic SDK — swap for OpenAI, Ollama, etc.
# from anthropic import AsyncAnthropic
# client = AsyncAnthropic()

class AIServicer(ai_pb2_grpc.AIServiceServicer):
    async def Generate(self, request, context):
        prompt = request.prompt
        print(f"[gRPC] Received prompt: {prompt}")

        # --- Replace with your actual LLM streaming call ---
        # async with client.messages.stream(
        #     model="claude-opus-4-5",
        #     max_tokens=1024,
        #     messages=[{"role": "user", "content": prompt}]
        # ) as stream:
        #     async for text in stream.text_stream:
        #         yield ai_pb2.GenerateResponse(token=text, done=False)

        # Simulated token stream for demo
        words = f"Streaming response to: {prompt}".split()
        for word in words:
            await asyncio.sleep(0.1)
            yield ai_pb2.GenerateResponse(token=word + " ", done=False)

        # Signal completion
        yield ai_pb2.GenerateResponse(token="", done=True)

async def serve():
    options = [
        # Match these with Go client keepalive settings (see Step 4)
        ("grpc.keepalive_time_ms", 20000),
        ("grpc.keepalive_timeout_ms", 10000),
        ("grpc.keepalive_permit_without_calls", 0),
        ("grpc.http2.min_recv_ping_interval_without_data_ms", 15000),
    ]

    server = aio.server(options=options)
    ai_pb2_grpc.add_AIServiceServicer_to_server(AIServicer(), server)
    listen_addr = "[::]:50051"
    server.add_insecure_port(listen_addr)
    print(f"[gRPC] Python server started on {listen_addr}")
    await server.start()
    await server.wait_for_termination()

if __name__ == "__main__":
    asyncio.run(serve())
```

> **Note**: No heartbeat needed on the Python side. The Go client will simply block on `stream.Recv()` while Python is processing. The gRPC layer (HTTP/2) handles the connection at the transport level.


## Step 3: Go — gRPC Client Wrapper

Create the gRPC client once at startup and reuse it. `grpc.ClientConn` is safe for concurrent use, so a single connection handles all in-flight requests.

```go
// internal/aiclient/client.go
package aiclient

import (
    "context"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/keepalive"

    pb "yourmodule/proto/ai"
)

type Client struct {
    conn   *grpc.ClientConn
    client pb.AIServiceClient
}

func New(addr string) (*Client, error) {
    // Keepalive: important when Go and Python sit behind a proxy or
    // Kubernetes ingress — idle TCP connections can be silently dropped
    // while the AI model is thinking.
    kaParams := keepalive.ClientParameters{
        Time:                20 * time.Second, // send ping after 20s of inactivity
        Timeout:             10 * time.Second, // consider dead if no pong in 10s
        PermitWithoutStream: false,            // only ping during active streams
    }

    conn, err := grpc.Dial(addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithKeepaliveParams(kaParams),
    )
    if err != nil {
        return nil, err
    }

    return &Client{
        conn:   conn,
        client: pb.NewAIServiceClient(conn),
    }, nil
}

func (c *Client) Close() {
    c.conn.Close()
}

func (c *Client) StreamGenerate(ctx context.Context, prompt, sessionID string) (pb.AIService_GenerateClient, error) {
    return c.client.Generate(ctx, &pb.GenerateRequest{
        Prompt:    prompt,
        SessionId: sessionID,
    })
}
```

## Step 4: Go — SSE Handler with Heartbeat

This is the core of the whole setup. The handler:

- Opens the SSE connection to the browser
- Starts a goroutine to read from the gRPC stream
- Forwards tokens as SSE `data:` events
- Sends a heartbeat comment (`: ping`) every 15 seconds when no tokens arrive — keeping the browser connection alive during long AI processing

```go
// internal/handler/sse.go
package handler

import (
    "context"
    "fmt"
    "io"
    "log"
    "net/http"
    "time"

    "yourmodule/internal/aiclient"
)

const heartbeatInterval = 15 * time.Second

type SSEHandler struct {
    ai *aiclient.Client
}

func NewSSEHandler(ai *aiclient.Client) *SSEHandler {
    return &SSEHandler{ai: ai}
}

type streamEvent struct {
    token string
    done  bool
    err   error
}

func (h *SSEHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Required SSE headers
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", http.StatusInternalServerError)
        return
    }

    prompt := r.URL.Query().Get("prompt")
    sessionID := r.URL.Query().Get("session_id")
    if prompt == "" {
        http.Error(w, "prompt is required", http.StatusBadRequest)
        return
    }

    // Hard timeout — adjust to your worst-case AI processing time
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Minute)
    defer cancel()

    stream, err := h.ai.StreamGenerate(ctx, prompt, sessionID)
    if err != nil {
        log.Printf("failed to start gRPC stream: %v", err)
        fmt.Fprintf(w, "event: error\ndata: failed to connect to AI service\n\n")
        flusher.Flush()
        return
    }

    // Buffered channel so the gRPC reader doesn't block immediately
    // if the SSE writer is briefly busy flushing
    eventCh := make(chan streamEvent, 8)

    // Goroutine: read from gRPC stream → push to channel
    go func() {
        defer close(eventCh)
        for {
            resp, err := stream.Recv()
            if err == io.EOF {
                eventCh <- streamEvent{done: true}
                return
            }
            if err != nil {
                eventCh <- streamEvent{err: err}
                return
            }
            if resp.Done {
                eventCh <- streamEvent{done: true}
                return
            }
            eventCh <- streamEvent{token: resp.Token}
        }
    }()

    // Ticker resets on real activity — heartbeats only fire during idle gaps
    heartbeat := time.NewTicker(heartbeatInterval)
    defer heartbeat.Stop()

    for {
        select {
        case evt, ok := <-eventCh:
            if !ok {
                return
            }
            if evt.err != nil {
                log.Printf("stream error: %v", evt.err)
                fmt.Fprintf(w, "event: error\ndata: stream interrupted\n\n")
                flusher.Flush()
                return
            }
            if evt.done {
                fmt.Fprintf(w, "event: done\ndata: [DONE]\n\n")
                flusher.Flush()
                return
            }
            // Forward token to browser
            fmt.Fprintf(w, "data: %s\n\n", evt.token)
            flusher.Flush()
            // Reset heartbeat — no ping needed while tokens are flowing
            heartbeat.Reset(heartbeatInterval)

        case <-heartbeat.C:
            // SSE comment line — browsers ignore it, proxies/nginx see activity
            fmt.Fprintf(w, ": ping\n\n")
            flusher.Flush()
            log.Printf("[SSE] heartbeat sent for session=%s", sessionID)

        case <-ctx.Done():
            fmt.Fprintf(w, "event: error\ndata: request timeout or cancelled\n\n")
            flusher.Flush()
            return
        }
    }
}
```

## Step 5: Wire It Up in main.go

```go
// main.go
package main

import (
    "log"
    "net/http"

    "yourmodule/internal/aiclient"
    "yourmodule/internal/handler"
)

func main() {
    // Single connection, created once, reused across all requests
    ai, err := aiclient.New("localhost:50051")
    if err != nil {
        log.Fatalf("failed to connect to AI gRPC service: %v", err)
    }
    defer ai.Close()

    sseHandler := handler.NewSSEHandler(ai)

    mux := http.NewServeMux()
    mux.Handle("/api/ai/stream", sseHandler)

    log.Println("Go HTTP server listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

## Step 6: Frontend — Consuming SSE

```javascript
const prompt = "Explain gRPC streaming in simple terms";

const evtSource = new EventSource(
  `/api/ai/stream?prompt=${encodeURIComponent(prompt)}&session_id=abc123`
);

const output = document.getElementById("output");

// Default message event — receives tokens
evtSource.onmessage = (e) => {
  output.innerText += e.data;
};

// Named "done" event — AI finished
evtSource.addEventListener("done", () => {
  evtSource.close();
  console.log("Stream complete");
});

// Named "error" event — something went wrong
evtSource.addEventListener("error", (e) => {
  console.error("SSE error", e);
  evtSource.close();
});
```

> **Tip**: The heartbeat (`: ping\n\n`) is a comment line in the SSE spec. Browsers silently ignore it — you don't need any client-side handler for it. It exists purely to prevent proxies and load balancers from closing what they perceive as an idle connection.


## Heartbeat: Where and Why

A common question: do you need heartbeats on the Python gRPC server side too?
Short answer: no — but it depends on your infrastructure.

| Layer | Protocol | Needs heartbeat? | Why |
|-------|----------|------------------|-----|
| Browser ↔ Go | SSE / HTTP/1.1 | ✅ Yes | Proxies drop idle HTTP connections |
| Go ↔ Python | gRPC / HTTP/2 | ⚠️ Only behind proxies | HTTP/2 handles keepalive at transport level |
| Python (internal) | — | ❌ No | Just keep yielding tokens |

If Go and Python run in the same cluster with no intermediate proxy, the gRPC connection is fine with no extra config. If there's a load balancer (AWS ALB, nginx, Kubernetes ingress) between them, add the `keepalive.ClientParameters` on the Go side and matching `grpc.keepalive_*` options on the Python server (as shown in Step 2) — otherwise the LB may silently reset the TCP connection mid-stream.

## Key Takeaways

- **Use server-side streaming in your proto**. `returns (stream GenerateResponse)` is what makes this work. Unary RPCs wait for the full response before returning — useless for LLM token streaming.
- **One gRPC connection, many streams**. Create `aiclient.Client` once at startup. HTTP/2 multiplexes many concurrent streams over a single TCP connection — don't open a new `grpc.Dial` per request.
- **Context cancellation is automatic**. When the browser closes the SSE tab, `r.Context()` is cancelled, which propagates to the gRPC stream — `stream.Recv()` returns an error and the goroutine exits cleanly. No manual cleanup needed.
- **Heartbeat resets on activity**. The `heartbeat.Reset(heartbeatInterval)` call means the ping only fires during true idle gaps (model thinking), not while tokens are actively streaming.
- **Hard timeout as a safety net**. `context.WithTimeout(3 * time.Minute)` ensures a stuck Python worker can't hold a goroutine and browser connection indefinitely.
