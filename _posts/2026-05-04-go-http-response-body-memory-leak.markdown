---
layout: post
title: "The Silent Go Memory Leak: Unclosed HTTP Response Bodies"
date: 2026-05-04 13:00:00 +0700
categories: golang backend performance
tags: [golang, http, memory-leak, performance, backend]
description: "A common Go bug where failing to close the HTTP response body drains connection pools, leaks memory, and crashes production services."
---

One of the most insidious bugs in Go backend engineering isn't a race condition or a logic flaw—it's five missing characters: `defer`.

Failing to close an HTTP response body will slowly choke your application to death.

## The Bug

```go
func FetchData(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    // Bug: We forgot to close resp.Body!
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("bad status: %d", resp.StatusCode)
    }
    
    return io.ReadAll(resp.Body)
}
```

## Why This is Disastrous

In Go, the HTTP client uses a connection pool. When you make a request, Go opens a TCP connection. To reuse that TCP connection for the next request, Go needs to know you are completely finished reading the response.

You signal this by calling `resp.Body.Close()`.

If you don't close the body:
1. The TCP connection remains open and bound to that specific request.
2. The connection cannot return to the pool.
3. The underlying file descriptors and memory buffers leak.
4. The next `http.Get()` must open a brand new TCP connection.

Under high load, your server will rapidly exhaust all available file descriptors (the dreaded `too many open files` error) or run out of memory, causing the process to crash.

## The Fix

Always defer closing the body immediately after checking the error:

```go
func FetchDataSafe(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    // DO THIS EVERY TIME
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("bad status: %d", resp.StatusCode)
    }
    
    return io.ReadAll(resp.Body)
}
```

## Advanced Gotcha: Draining the Body

There is an even more subtle bug. Even if you call `Close()`, if you haven't read the entire body, Go **will not reuse the connection**. It will forcefully close the TCP connection to prevent data corruption.

If you are making an API call where you only care about the HTTP Status Code, you must drain the body before closing it to keep Keep-Alive and connection pooling working:

```go
func PingService(url string) error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    
    // Drain the body so the connection can be reused
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("service down")
    }
    return nil
}
```

## How to Catch It

1. **Unit Tests:** You won't catch this in normal unit tests.
2. **Linter:** Use `golangci-lint` with the `bodyclose` analyzer enabled. It will fail your CI build if you forget to close a body.
3. **Metrics:** Monitor your active TCP connections. If the number of `ESTABLISHED` or `CLOSE_WAIT` connections climbs linearly and never drops, you have a leak.

The `defer resp.Body.Close()` is the seatbelt of Go HTTP clients. Put it on before you drive.
