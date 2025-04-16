# Idempotency Key Pattern

# 🔄 What is Idempotency?
Idempotency is a property of certain operations where applying the same operation multiple times produces the same result as applying it once. In distributed systems and API design, idempotency ensures that if a client makes the same request multiple times (due to network issues, timeouts, or retries), the state of the system changes exactly as if the request was made only once.

>_"Idempotence is the property of certain operations in mathematics and computer science whereby they can be applied multiple times without changing the result beyond the initial application."_

Source from [Wikipedia](https://https://en.wikipedia.org/wiki/Idempotence)

## Why is Idempotency Important?
  - **Network Reliability:** Networks are unreliable. Requests can time out, connections can drop.
  - **Retries:** Clients often implement retry mechanisms that can lead to duplicate requests.
  - **User Experience:** Prevents duplicate transactions (e.g., double payments).
  - **System Integrity:** Maintains data consistency even when operations are repeated.

# 🔑 Idempotency Keys
An idempotency key is a unique identifier associated with a specific request. When a client sends a request to your API, it includes this key, allowing backend service to:

1. Detect duplicate requests
2. Return the same response for the same operation
3. Prevent duplicate side effects

## Properties of Idempotency Keys:

- ** Unique:** Each distinct operation should have a different key
- **Client-generated:** Usually created by the client (UUID v4 recommended)
- **Consistent:** The same operation should always use the same key
- **Time-bound:** Keys can expire after a retention period

# 🛠️ Implementation in Golang with Redis and PostgreSQL

This repository demonstrates a production-grade implementation of idempotent APIs using Go, Redis, and PostgreSQL.

## System Architecture
```
┌─────────┐     ┌─────────────────┐     ┌─────────┐     ┌───────────┐
│  Client │────▶│  API Gateway    │────▶│  Go API │────▶│ Business  │
└─────────┘     └─────────────────┘     └─────────┘     │   Logic   │
                                             │          └───────────┘
                                             │                │
                                        ┌────▼────┐      ┌────▼─────┐
                                        │  Redis  │      │ Postgres │
                                        │ (Cache) │      │   (DB)   │
                                        └─────────┘      └──────────┘
```
## Core Components

- **Idempotency Middleware:** Intercepts all requests to check for idempotency keys
- **Redis Cache:** Stores in-flight request states and completed response payloads
- **PostgreSQL:** Persists idempotency records for long-term reference
- **Cleanup Job:** Manages key expiration and cleanup

### System Context
![System Context](/images/system-context.png)

### System Container
![System Container](/images/system-container.png)

### System Component
![System Component](/images/system-component.png)

### Idempotency Key Flow Sequence Diagram
![Sequence Diagram](/images/sequence-diagram.png)

# 📝 How to Implement Idempotency

## 1. Idempotency Key Management

### Client-Side Implementation:
```go
  import (
    "github.com/google/uuid"
    "net/http"
  )

  func IdempotentRequest() {
    // Generate a unique idempotency key for this operation
    idempotencyKey := uuid.New().String()

    // Store this key if you need to make exact same request later
    req, _ := http.NewRequest("POST", "https://api.goesbams.com/payments", payloadBody)
    req.Header.Add("X-Idempotency-Key", idempotencyKey)

    // make the request
    // if network fails, this request will be retried with the same idempotency key
  }
```

### Server-Side Implementation

```go
  package middleware

  import (
    "github.com/gin-gonic/gin"
    "github.com/go-redis/redis/v8"
    "context"
    "time"
    "encoding/json"
  )

  // Idempotency Middleware checks and enforces idempotency
  func IdempotencyMiddleware(redisClient *redis.Client) gin.HandleFunc {
    return func(c *gin.Context) {
      // Only apply to mutating methods (POST, PUT, PATCH, DELETE)
      if c.Request.Method == "GET" {
        c.Next()
        return
      }

      idempotencyKey :c.GetHeader("X-Idempotency-Key")
      if idempotencyKey == "" {
        c.JSON(400, gin.H{"error": "X-Idempotency-Key header is required"})
        c.Abort()
        return
      }

      ctx := context.Background()

      // check if idempotency key exists before
      keyExists, err := redisClient.Exists(ctx, "idempotent:"+idempotencyKey).Result()
      if err != nil {
        c.JSON(500, gin.H{"error": "Failed to check idempotency key"})
        c.Abort()
        return
      }
    }
  }
```

