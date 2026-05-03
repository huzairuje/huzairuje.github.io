---
layout: post
title: "Handling Context Cancellations Correctly in Go Database Queries"
date: 2026-05-04 15:00:00 +0700
categories: golang backend database
tags: [golang, database, context, backend]
description: "Why passing context.Background() to database queries is a bad idea, and how to properly handle timeouts and client disconnects using context.Context."
---

The `context.Context` package in Go is the standard way to propagate deadlines, cancellation signals, and request-scoped values across API boundaries. 

However, many developers mistakenly use `context.Background()` or `context.TODO()` when querying the database, defeating the entire purpose of context cancellation.

## The Danger of Ignoring Context

When a user makes an HTTP request to your Go backend, the server attaches a context to that request (`r.Context()`). 

If the user closes their browser or their mobile app loses connectivity, the HTTP server cancels that context.

```go
// BAD: Ignoring the request context
func HandleUserRequest(w http.ResponseWriter, r *http.Request) {
    // We fetch data using Background context instead of r.Context()
    data, err := db.QueryContext(context.Background(), "SELECT * FROM huge_table")
    // ...
}
```

In the bad example above, if the client disconnects, the database query **keeps running**. It will consume database CPU, hold connection pool slots, and transfer MBs of data to your Go server, only for your server to realize the client is gone and throw the data away.

Under a minor DDoS attack or sudden traffic spike, this will cause your database to buckle under the weight of "ghost queries."

## The Right Way: Propagate Context

Always pass the request context down to your repository layer:

```go
// GOOD: Propagating the context
func HandleUserRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Pass ctx down!
    data, err := fetchUserData(ctx, db, userID)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // The client gave up and disconnected. 
            // We can stop processing and return early.
            return
        }
        http.Error(w, err.Error(), 500)
        return
    }
    // ...
}

func fetchUserData(ctx context.Context, db *sql.DB, id int) (Data, error) {
    // The query will instantly abort if ctx is canceled
    row := db.QueryRowContext(ctx, "SELECT ... WHERE id = $1", id)
    // ...
}
```

If the client disconnects, `db.QueryRowContext` will intercept the cancellation signal, send an interrupt to PostgreSQL/MySQL, and abort the query at the database level, saving precious resources.

## Adding Database Timeouts

Even if the client doesn't disconnect, you shouldn't let a bad query run forever. Use `context.WithTimeout` to enforce an upper bound on execution time.

```go
func fetchCriticalData(ctx context.Context, db *sql.DB) (Data, error) {
    // Ensure this query never takes longer than 2 seconds, 
    // even if the parent request context allows 30 seconds.
    queryCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel() // Always defer cancel to prevent context leaks!

    var data Data
    err := db.QueryRowContext(queryCtx, "SELECT ...").Scan(&data.ID)
    
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return data, fmt.Errorf("database query timed out")
        }
        return data, err
    }
    return data, nil
}
```

### The Golden Rule of Context
Contexts should flow down the call stack, never up. Functions should take `ctx context.Context` as their first argument, and you should always `defer cancel()` when creating derived contexts.
