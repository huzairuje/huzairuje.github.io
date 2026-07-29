---
layout: post
title:  "Building a Simple RAG Stack with Redis 8 Vector Sets, Go, and Python"
date:   2026-05-10 21:35:00 +0700
categories: architecture ai redis golang python
tags: [rag, redis, golang, python, vector-search, sse]
description: "Build a compact RAG stack with Redis 8 Vector Sets, a Go API gateway, Python FastAPI embeddings, SSE streaming, and local Ollama support."
---

Retrieval-Augmented Generation (RAG) has become the standard architectural pattern for building intelligent applications that can chat over private data. While many RAG implementations rely on complex arrays of managed services, I recently put together a "Simple RAG Stack" that leverages the latest features in **Redis 8** to handle multiple infrastructure roles simultaneously.

You can check out the source code here: [RAG-stack-simple](https://github.com/huzairuje/RAG-stack-simple).

## Why Redis 8?

Redis has always been a Swiss Army knife for caching, session management, and message brokering. With **Redis 8**, native Vector Sets make it an incredibly capable Vector Database without the need for external plugins like RediSearch. In this stack, Redis acts as:

1. **Vector Database**: Storing document embeddings using `VSET` (`VADD` and `VSIM`).
2. **Semantic Cache**: Caching frequent query vectors to bypass expensive LLM calls.
3. **Session Store**: Maintaining chat history using standard Redis Lists with a 24-hour TTL.
4. **Key-Value Store**: Holding raw document text chunks and cached LLM responses.

By consolidating these responsibilities into a single dependency, we drastically simplify the deployment topology.

## The Architecture

The architecture is divided into clear, responsibility-focused microservices:

```text
[Svelte Frontend] 
     │  (HTTP + SSE)
     ▼
[Go API Gateway :8080]  ← Pure SSE proxy & document chunker
     │
     ├── POST /api/chat/stream      → Proxies SSE to Python
     └── POST /api/documents/upload → Chunks text → calls Python /index
     │
     ▼
[Python FastAPI :8001]  ← Embedding & RAG Pipeline
     ├── POST /embed        → SentenceTransformer → base64 FP32
     ├── POST /index        → VADD FP32 blobs into Redis Vector Set
     └── POST /chat/stream  → VSIM → LLM stream → SSE
     │
     ▼
[Redis 8 :6379] ← The Single Source of Truth
```

### The Go API Gateway (Echo)
The Go service acts as the front door. It handles document uploads, parsing them, chunking the text, and forwarding those chunks to the Python service for embedding. For chat interactions, it acts as a transparent Server-Sent Events (SSE) proxy, ensuring that the frontend gets the streaming tokens as fast as they are generated.

### The Python Embedding Service (FastAPI)
Python remains the undisputed king of AI integrations. This FastAPI service handles the heavy lifting of generating embeddings using `SentenceTransformer` (outputting base64 FP32 vectors), managing the RAG pipeline, and performing vector similarity search (`VSIM`) against Redis 8.

It's also designed to be flexible with LLM providers. While it can connect to OpenAI (`gpt-4o-mini`), the stack is natively configured to support **Ollama** (`host.docker.internal:11434`), allowing you to run the entire stack locally with open-weights models.

### Svelte Frontend
A lightweight Svelte 4 frontend consumes the SSE streams, providing a snappy and responsive chat interface.

## Seamless Local Development

Thanks to Docker Compose, bringing the entire stack up is trivial:

```bash
docker compose up --build
```

This provisions the Redis 8 instance, builds the Go and Python services, and serves the Svelte frontend on port `5173`. The Docker environment is specifically configured to map the host gateway, making local Ollama integration seamless right out of the box.

## Conclusion

Building RAG applications doesn't require a dozen different SaaS products. By utilizing the native vector capabilities of Redis 8 combined with the concurrency of Go for the gateway and the rich AI ecosystem of Python for the pipeline, we can build a robust, scalable, and entirely local RAG stack. 
