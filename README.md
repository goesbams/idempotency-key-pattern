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



