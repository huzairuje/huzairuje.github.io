---
layout: post
title: "Timezones and time.Time Bugs in Go"
date: 2026-05-04 16:00:00 +0700
categories: golang backend architecture
tags: [golang, time, timezone, backend, bugs]
description: "How time.Time equality works in Go, why DeepEqual fails on times, and how to safely store and transmit timestamps in backend systems."
---

Handling time in backend systems is notoriously difficult. Go's `time.Time` struct is powerful, but it has a few sharp edges that routinely catch developers off guard, especially when dealing with databases and JSON APIs.

## The `==` vs `.Equal()` Trap

A `time.Time` in Go contains both a wall clock reading and a monotonic clock reading (used for measuring elapsed time locally). 

Because of this, two `time.Time` values that represent the *exact same instant* might not be evaluated as equal using the `==` operator.

```go
func main() {
    t1 := time.Now()
    t2 := t1.Round(0) // Strips the monotonic clock reading

    fmt.Println(t1 == t2)      // FALSE!
    fmt.Println(t1.Equal(t2))  // TRUE
}
```

**Rule 1: Always use `t1.Equal(t2)` to compare times, never `==`.**

This also means you should be extremely careful when using `time.Time` as a map key, or when using `reflect.DeepEqual` in unit tests (it will often fail on structs containing times).

## The JSON Timezone Ambiguity

When you encode a `time.Time` to JSON, Go uses RFC3339 format.

```go
t := time.Date(2026, 5, 4, 12, 0, 0, 0, time.FixedZone("WIB", 7*3600))
b, _ := json.Marshal(t)
fmt.Println(string(b)) 
// Output: "2026-05-04T12:00:00+07:00"
```

When a frontend client parses this and sends it back to your Go server, Go decodes it flawlessly. But what happens if the frontend sends the time in UTC?

`"2026-05-04T05:00:00Z"`

Go will parse this, and the resulting `time.Time` will be mathematically identical, but its internal `.Location()` will be set to UTC, not WIB. If you are grouping records by day or running timezone-specific logic (`t.Hour()`), you will introduce subtle bugs.

**Rule 2: Normalize times to UTC immediately at the system boundary.**

```go
type Request struct {
    ScheduledAt time.Time `json:"scheduled_at"`
}

func handle(req Request) {
    // Immediately convert to UTC before doing any logic or DB writes
    utcTime := req.ScheduledAt.UTC()
    // ...
}
```

## Database Storage: PostgreSQL `TIMESTAMP` vs `TIMESTAMPTZ`

If you use PostgreSQL, you have two column types for time:
- `TIMESTAMP WITHOUT TIME ZONE`
- `TIMESTAMP WITH TIME ZONE` (TIMESTAMPTZ)

You should **almost always use `TIMESTAMPTZ`**.

If you use `TIMESTAMP WITHOUT TIME ZONE`, the database strips the timezone information entirely. If your Go server writes a time in EST, and later reads it, the database will return it assuming it was UTC, effectively shifting the time by 5 hours.

With `TIMESTAMPTZ`, Postgres converts the incoming time to UTC for storage. When Go reads it back, the PostgreSQL driver (like `pgx` or `pq`) correctly instantiates a UTC `time.Time`.

## Summary
1. Never use `==` for time comparisons. Use `t1.Equal(t2)`.
2. Convert all incoming times to `.UTC()` as soon as they enter your application.
3. Always use timezone-aware column types in your database (e.g. `TIMESTAMPTZ`).
4. If you need to render time in a specific timezone for a user, do it in the presentation layer (frontend), or right before generating the email/PDF on the backend. Keep internal state in UTC.
