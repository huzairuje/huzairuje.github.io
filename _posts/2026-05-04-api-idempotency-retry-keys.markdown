---
layout: post
title: "API Idempotency: Why You Need Retry Keys"
date: 2026-05-04 17:00:00 +0700
categories: backend architecture
tags: [backend, architecture, idempotency, api]
description: "How to design idempotent APIs using Idempotency Keys to prevent double-charging users during network timeouts and retries."
---

Imagine a user clicks "Checkout" on your e-commerce app. 

1. Your mobile app sends a `POST /checkout` request to your backend.
2. Your backend charges the user's credit card.
3. Your backend attempts to send a `200 OK` response back to the app.
4. The user's phone goes through a tunnel, losing cellular connection. The response is dropped.

From the backend's perspective, the payment succeeded. From the mobile app's perspective, the request timed out. 

What does the mobile app do? It automatically retries the request. 
Your backend receives `POST /checkout` again, and **charges the user's credit card a second time**.

This is why critical APIs must be **idempotent**. An idempotent API guarantees that making the same request multiple times has the same effect as making it exactly once.

## The Solution: Idempotency Keys

To fix this, the client must generate a unique identifier for the *intent* of the action, usually a UUID, and send it as an HTTP header (e.g., `Idempotency-Key: <UUID>`).

When the backend receives the request, it checks if it has seen this key before.

### Step 1: The Database Schema

Create a table to store idempotency records:

```sql
CREATE TABLE idempotency_keys (
    key VARCHAR(255) PRIMARY KEY,
    user_id INT NOT NULL,
    request_path VARCHAR(255) NOT NULL,
    response_status INT,
    response_body JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Step 2: The Logic Flow

When a request arrives:

1. Attempt to insert the `Idempotency-Key` into the database.
   - If it inserts successfully, this is a **new request**. Proceed with the payment.
   - If it fails with a `Unique Constraint Violation`, we have **seen this request before**.

2. If we've seen it before, check the `response_status`:
   - If `response_status` is NULL, the original request is *still processing*. Return a `409 Conflict` (or ask the client to poll).
   - If `response_status` is populated, return the exact same `response_body` and `response_status` we sent the first time, without charging the card again.

3. Once the payment finishes successfully, update the `idempotency_keys` table with the result in the same database transaction.

### Implementation in Go

```go
func HandleCheckout(w http.ResponseWriter, r *http.Request) {
    idempKey := r.Header.Get("Idempotency-Key")
    if idempKey == "" {
        http.Error(w, "Idempotency-Key required", 400)
        return
    }

    // Attempt to lock the key
    inserted, existingRecord, err := db.TryLockIdempotencyKey(ctx, idempKey, userID)
    if err != nil {
        http.Error(w, "Internal Error", 500)
        return
    }

    if !inserted {
        if existingRecord.Status == nil {
            http.Error(w, "Concurrent request processing", 409)
            return
        }
        // Return the cached response
        w.WriteHeader(*existingRecord.Status)
        w.Write(existingRecord.Body)
        return
    }

    // Process the actual payment (wrapped in a transaction)
    responseBody, err := processPayment(ctx, req)
    
    // Save the result for future retries
    db.SaveIdempotencyResult(ctx, idempKey, 200, responseBody)
    
    w.WriteHeader(200)
    w.Write(responseBody)
}
```

## Key Considerations

1. **Scoped Keys:** Always bind the idempotency key to the specific user making the request. If User A maliciously guesses User B's idempotency key, you don't want to leak User B's receipt.
2. **Expiry:** Idempotency keys should not live forever. A 24-hour TTL is usually sufficient to handle network blips.
3. **Payload Hashing:** For high security, hash the request body and store it with the key. If a client sends the same key but a different request body, reject it with a `400 Bad Request`.

Idempotency isn't just for payments; it's essential for any distributed system (like webhooks) where retries are an inherent part of the network architecture.
