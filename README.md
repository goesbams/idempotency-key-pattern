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

### System Context
![System Context](/images/system-context.png)

### System Container
![System Container](/images/system-container.png)

### System Component
![System Component](/images/system-component.png)

### Idempotency Key Flow Sequence Diagram
![Sequence Diagram](/images/sequence-diagram.png)


