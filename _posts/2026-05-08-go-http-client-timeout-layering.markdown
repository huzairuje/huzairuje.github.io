---
layout: post
title: "Go HTTP Timeouts: One Timeout Is Not Enough"
date: 2026-05-08 14:00:00 +0700
categories: golang backend networking
tags: [golang, http, timeouts, networking, reliability]
description: "A production outage caused by missing layered HTTP timeouts in Go clients and how to configure them correctly."
---

Our API had a timeout. Still, goroutines piled up and latency kept rising.

Why? Because we configured only one timeout and assumed it covered everything.

## Timeout Layers You Need

Different failure modes happen at different phases:

- TCP dial hangs
- TLS handshake stalls
- Server delays response headers
- Body read drags forever

## Better Client Configuration

```go
transport := &http.Transport{
    DialContext: (&net.Dialer{
        Timeout:   2 * time.Second,
        KeepAlive: 30 * time.Second,
    }).DialContext,
    TLSHandshakeTimeout:   2 * time.Second,
    ResponseHeaderTimeout: 3 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}

client := &http.Client{
    Timeout:   5 * time.Second, // total request budget
    Transport: transport,
}
```

Then wrap each request with a context deadline that matches business SLA.

## Lesson Learned

## What Went Wrong in My Incident

- **What alerted first:** Tail latency and goroutine counts rose during partial network degradation.
- **What misled us:** We had a global timeout set, so we assumed timeout coverage was complete.
- **What confirmed root cause:** Phase-level tracing showed stalls in dial/handshake/header phases without proper per-phase limits.

Timeouts are not one number. They are a contract for each network phase.

If you do not define that contract, the kernel and defaults will define it for you.
