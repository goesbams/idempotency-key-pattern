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

- **Unique:** Each distinct operation should have a different key
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

### System Context (L1)
![System Context](/images/system-context.png)

### System Container (L2)
![System Container](/images/system-container.png)

### System Component (L3)
![System Component](/images/system-component.png)

### Idempotency Key Flow Sequence Diagram
![Sequence Diagram](/images/sequence-diagram.png)

# 🌟 Understanding the Architecture
The C4 diagrams provided earlier follow the C4 model for visualizing software architecture at different levels of detail:

1. **Context (L1):** Shows how our system fits into the world around it
2. **Container (L2):** Shows the high-level technology components that make up the system
3. **Component (L3):** Shows how a container is made up of components and their relationships
4. **Sequence:** Shows the runtime behavior in different scenarios

> These diagrams help stakeholders understand the system from multiple perspectives. The Context diagram helps business stakeholders understand the system boundaries, while the Container and Component diagrams help developers understand the technical architecture.

# 📝  Idempotency Key Management

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
	"bytes"
	"context"
	"encoding/json"
	"io"
	"net/http"
	"strconv"
	"time"

	"github.com/go-redis/redis/v8"
	"github.com/labstack/echo/v4"
)

// IdempotencyMiddleware checks and enforces idempotency
func IdempotencyMiddleware(redisClient *redis.Client) echo.MiddlewareFunc {
	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			// Only apply to mutating methods (POST, PUT, PATCH, DELETE)
			if c.Request().Method == "GET" {
				return next(c)
			}

			idempotencyKey := c.Request().Header.Get("X-Idempotency-Key")
			if idempotencyKey == "" {
				return c.JSON(http.StatusBadRequest, map[string]string{"error": "X-Idempotency-Key header is required"})
			}

			ctx := context.Background()

			// Check if idempotency key exists before
			keyExists, err := redisClient.Exists(ctx, "idempotent:"+idempotencyKey).Result()
			if err != nil {
				return c.JSON(http.StatusInternalServerError, map[string]string{"error": "Failed to check idempotency key"})
			}

			if keyExists == 1 {
				// Key exists, check if processing or completed
				status, err := redisClient.HGet(ctx, "idempotent:"+idempotencyKey, "status").Result()
				if err != nil {
					return c.JSON(http.StatusInternalServerError, map[string]string{"error": "Failed to check idempotency key status"})
				}

				if status == "processing" {
					// Request is still being processed
					return c.JSON(http.StatusConflict, map[string]string{"error": "A similar request is currently being processed"})
				} else if status == "completed" {
					// Return the stored response
					responseBody, _ := redisClient.HGet(ctx, "idempotent:"+idempotencyKey, "response").Result()
					statusCode, _ := redisClient.HGet(ctx, "idempotent:"+idempotencyKey, "statusCode").Result()

					var response map[string]interface{}
					json.Unmarshal([]byte(responseBody), &response)

					code := http.StatusOK
					if statusCode != "" {
						code, _ = strconv.Atoi(statusCode)
					}

					return c.JSON(code, response)
				}
			}

			// Set key as processing
			redisClient.HSet(ctx, "idempotent:"+idempotencyKey, "status", "processing")
			redisClient.HSet(ctx, "idempotent:"+idempotencyKey, "created_at", time.Now().String())
			redisClient.Expire(ctx, "idempotent:"+idempotencyKey, 24*time.Hour) // TTL for keys

			// Create a custom response writer to capture the response
			resBody := new(bytes.Buffer)
			mw := io.MultiWriter(c.Response().Writer, resBody)
			writer := &bodyLogWriter{
				ResponseWriter: c.Response().Writer,
				body:           resBody,
			}
			c.Response().Writer = writer

			// Process the request
			err = next(c)
			if err != nil {
				return err
			}

			// After processing, store the response
			redisClient.HSet(ctx, "idempotent:"+idempotencyKey, "status", "completed")
			redisClient.HSet(ctx, "idempotent:"+idempotencyKey, "response", resBody.String())
			redisClient.HSet(ctx, "idempotent:"+idempotencyKey, "statusCode", strconv.Itoa(c.Response().Status))

			return nil
		}
	}
}

// Custom response writer to capture response body
type bodyLogWriter struct {
	http.ResponseWriter
	body *bytes.Buffer
}

func (w *bodyLogWriter) Write(b []byte) (int, error) {
	w.body.Write(b)
	return w.ResponseWriter.Write(b)
}

// Example usage in Echo application
func SetupEchoServer() *echo.Echo {
	e := echo.New()
	
	// Setup Redis client
	redisClient := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	
	// Apply idempotency middleware to specific route groups
	apiGroup := e.Group("/api")
	apiGroup.Use(IdempotencyMiddleware(redisClient))
	
	// Define your routes
	apiGroup.POST("/payments", handlePayment)
	apiGroup.PUT("/users/:id", updateUser)
	
	return e
}

func handlePayment(c echo.Context) error {
	// Payment processing logic here
	return c.JSON(http.StatusOK, map[string]string{"status": "payment_success"})
}

func updateUser(c echo.Context) error {
	// User update logic here
	return c.JSON(http.StatusOK, map[string]string{"status": "user_updated"})
}
```

# ⚠️ Common Challenges and Solutions
## 1. Race Conditions
**Problem:** Multiple identical requests arriving simultaneously before the first one completes.

**Solution:** Implement a distributed lock system using Redis with SETNX.

```go
// setting an exclusive lock with Redis
func acquireLock(rc *redis.Client, key string, ttl time.Duration) (bool, error) {
  return rc.SetNX(context.Background(), "lock:"+key, "1", ttl).Result()
}

func releaseLock(rc *redis.Client, key string) {
  rc.Del(context.Background(), "lock:"+key)
}
```

## 2. Key Collisions
**Problem:** Different operations generating the same idempotency key.

**Solution:** Include operation-specific data in key generation.

```go
// Client-side: Generate key based on operation data
func generateIdempotencyKey(operationType string, resourceID string, payload []byte) string {
  h := sha256.New()
  h.Write([]byte(operationType + resourceID))
  h.Write(payload)
  return fmt.Sprintf("%x", h.Sum(nil))
}
```
# 📊 Monitoring and Observability
```go
// Prometheus metrics for idempotency tracking
var (
  idempotencyHits = prometheus.NewCounter(prometheus.CounterOpts{
    Name: "idempotency_hits_total",
    Help: "Total number of repeated requests caught by idempotency checks",
  })

  idempotencyProccesingtime = prometheus.NewHistogram(prometheus.HistogramOpts{
    Name: "idempotency_processing_seconds",
    Help: "Time spent processing idempotency logic",
  })
)

func init() {
  prometheus.MustRegister(idempotencyHits)
  prometheus.MustRegister(idempotencyProcessingTime)
}
```

# 📚 Additional Resources
- [Understanding the Idempotent Property in REST APIs](https://www.restapitutorial.com/lessons/idempotency.html)
- [Stripe's Idempotent Requests Documentation](https://stripe.com/docs/api/idempotent_requests)
- [Implementing Idempotency in Distributed Systems](https://blog.cloudflare.com/introducing-workers-durable-objects/)



