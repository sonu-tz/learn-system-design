# Lesson 6: Resilience and Failure Handling

## 1. What Is Resilience?

Resilience is a system's ability to continue providing useful service when some of its components fail, become slow, or behave unexpectedly.

Consider a booking platform:

- A user submits a booking.
- The application server calls PostgreSQL.
- PostgreSQL becomes slow.
- The application keeps waiting for the database.
- Requests accumulate.
- Connections and memory are consumed.
- Other users can no longer create bookings.

The original problem was a slow database, but the failure spread throughout the application.

A resilient system prevents a local failure from becoming a system-wide outage.

### Reliability vs. Resilience vs. Availability

| Concept | Meaning |
|---|---|
| Reliability | The system performs correctly over time. |
| Availability | The system is accessible and usable when needed. |
| Resilience | The system can withstand failures and recover. |
| Fault tolerance | The system can continue operating despite specific component failures. |
| Recovery | The system returns to normal operation after a failure. |

These concepts are related but not interchangeable. For example, a service might become temporarily unavailable but recover quickly because its resilience mechanisms work correctly.

---

## 2. The Failure Scenario: Cascading Failures

Imagine this architecture:

```text
Client
  |
  v
Load Balancer
  |
  v
Application Servers
  |
  v
PostgreSQL
```

Initially, PostgreSQL responds in 50 ms. Suddenly, query latency increases to 5 seconds.

What happens next?

1. Application requests wait longer for database responses.
2. Each request holds resources, such as a database connection and application memory.
3. More requests arrive while earlier requests are still waiting.
4. The connection pool becomes exhausted.
5. Requests begin timing out.
6. Clients retry failed requests.
7. Retries add even more load.
8. The application becomes unhealthy, even if its own CPU usage is relatively low.

This is called a **cascading failure**: one component's problem causes other components to fail in succession.

### Why waiting is dangerous

Suppose an application can handle 200 concurrent requests. If requests complete in 100 ms, the application can process requests quickly. If they instead remain blocked for 5 seconds, those 200 request slots remain occupied much longer.

Using Little's Law:

\[
L = \lambda W
\]

Where:

- \(L\) = average number of requests in the system
- \(\lambda\) = arrival rate, in requests per second
- \(W\) = average time spent in the system, in seconds

At 100 requests per second:

- With 0.1-second response time: \(L = 100 \times 0.1 = 10\)
- With 5-second response time: \(L = 100 \times 5 = 500\)

The same arrival rate now produces approximately 500 concurrent requests instead of 10.

**Lesson:** A slow dependency can exhaust resources even when traffic has not increased.

---

## 3. Timeouts: Do Not Wait Forever

A timeout defines how long a component is allowed to wait for an operation before treating it as failed.

Without a timeout, a request may wait indefinitely or until another resource limit is reached.

### Example

A booking API calls a payment provider:

```text
Booking API
    |
    | Payment request
    v
Payment Provider
    |
    | No response
    v
Booking API keeps waiting
```

If the payment provider stops responding, the booking API must not wait forever.

A timeout allows the application to stop waiting and decide what to do next.

### Common types of timeouts

| Timeout | Purpose |
|---|---|
| Connection timeout | Limits how long establishing a connection can take. |
| Request/read timeout | Limits how long the client waits for a response or data. |
| Database query timeout | Limits how long a query is allowed to execute. |
| Pool-acquisition timeout | Limits how long a request waits for a database connection. |
| Overall deadline | Limits the total time allowed for an operation. |

The exact timeout behavior depends on the client library and the underlying service. Timing out on the client does not necessarily cancel the work already running in the database or remote service.

### Timeout vs. deadline

A timeout is commonly applied to an individual operation. A deadline is the total time budget for the complete operation.

Consider this request:

```text
Incoming request
      |
      v
Read booking       200 ms
      |
      v
Call payment       500 ms
      |
      v
Save booking       300 ms
      |
      v
Return response
```

The total is 1,000 ms, excluding other overhead.

If the user's request has a 1.2-second deadline, the application must manage the individual operations within that remaining budget. It should not give every operation an independent 1.2-second timeout, because the total request could then take much longer.

### How to choose a timeout

Choose timeouts based on:

- Expected operation latency.
- The user-facing latency objective.
- Network conditions.
- The dependency's normal behavior.
- The remaining request deadline.
- Whether the operation can safely be abandoned.

Avoid choosing an extremely large timeout just to reduce visible errors. It can turn fast failures into long periods of resource exhaustion.

Also avoid choosing an excessively small timeout that rejects normal requests during ordinary latency variation.

**Important:** A timeout does not prove that the operation failed. The server may have completed the operation while the response was delayed or lost.

This distinction becomes critical for booking, payment, and order creation.

---

## 4. Retries: Try Again Carefully

A retry means attempting an operation again after a failure.

Retries are useful for temporary failures, such as:

- A brief network interruption.
- A temporary service-unavailable response.
- A connection reset.
- A transient rate limit, when the service's guidance allows another attempt.

However, retries are dangerous when used without limits.

### The retry amplification problem

Suppose 1,000 requests arrive at an application. Each request fails and is retried three times.

If every request makes three additional attempts, the dependency may receive:

\[
1000 \times (1+3) = 4000
\]

attempts instead of 1,000.

The retry policy has multiplied the traffic by four.

If the dependency is already overloaded, retries can make the outage worse.

### Exponential backoff

Instead of retrying immediately, wait longer after each failed attempt.

A common pattern is:

\[
d_n = \min(d_{\max}, d_0 \times 2^n)
\]

Where:

- \(d_n\) = delay before a retry.
- \(d_0\) = initial delay.
- \(d_{\max}\) = maximum delay.
- \(n\) = retry number, starting at zero for the first retry delay.

For example, with an initial delay of 200 ms:

```text
Attempt 1 fails
      |
      v
Wait 200 ms
      |
      v
Attempt 2 fails
      |
      v
Wait 400 ms
      |
      v
Attempt 3 fails
      |
      v
Wait 800 ms
      |
      v
Stop when the retry limit or deadline is reached
```

This is an example, not a universal timing configuration.

### Jitter

If every application server retries at exactly the same time, all of them may hit the dependency simultaneously again.

Jitter adds randomness to retry delays so that retries are spread over time.

```text
Without jitter:
Server A ------ retry at 1.0 s
Server B ------ retry at 1.0 s
Server C ------ retry at 1.0 s

With jitter:
Server A ------ retry at 0.8 s
Server B ------ retry at 1.2 s
Server C ------ retry at 0.9 s
```

The exact jitter algorithm depends on the chosen policy.

### When should you retry?

Retry when all of the following are reasonably true:

1. The failure may be temporary.
2. The operation is safe to repeat, or idempotency is implemented.
3. The failure type is retryable.
4. The retry budget has not been exhausted.
5. Enough time remains before the overall deadline.
6. The retry will not overload an already struggling dependency.

Do not blindly retry validation errors, authorization failures, or other permanent failures.

For HTTP APIs, the appropriate decision depends on the status code, the endpoint's semantics, and the API's documented retry guidance. For example, a rate-limited request may need to respect `Retry-After`.

### Retry ownership

If the frontend retries three times, the backend retries three times, and a downstream service retries three times, one user action can produce many downstream attempts.

Therefore:

- Decide which layer owns retries.
- Keep retry counts bounded.
- Understand the total attempts across the call chain.
- Avoid retries at multiple layers unless their combined behavior is intentional.

---

## 5. Idempotency: Make Repeated Requests Safe

Idempotency means that repeating an operation produces the same intended effect as performing it once.

Mathematically:

\[
f(f(x)) = f(x)
\]

This expression illustrates the idea of idempotency, although real API operations usually involve stored state, not just mathematical functions.

### The booking problem

Imagine a user creates a booking.

1. The client sends `Create Booking`.
2. The server successfully saves the booking.
3. The response is lost because of a network problem.
4. The client sees a timeout.
5. The client retries the request.
6. The server creates another booking.

The user intended to create one booking but received two.

A timeout did not tell the client whether the original operation had committed.

### Idempotency keys

The client can send a unique key for a logical operation:

```text
POST /bookings

Idempotency-Key: request-abc123
```

The server records the key alongside the result.

On the first request:

```text
Key: request-abc123
      |
      v
No existing result
      |
      v
Create booking and store result
```

On a retry with the same key:

```text
Key: request-abc123
      |
      v
Existing result found
      |
      v
Return the stored result
Do not create another booking
```

The key must identify the same logical action across retries. Generating a fresh key for every retry defeats the purpose.

### What a robust idempotency implementation needs

- A well-defined scope for the key, such as a customer or tenant.
- A durable record of the key and operation result.
- Protection against concurrent requests using the same key.
- A defined retention period.
- A policy for requests that reuse a key with a different payload.
- Atomic coordination between recording the key and performing the business operation.

For example, storing the idempotency record separately from the booking transaction can leave a dangerous gap: the booking might commit while the idempotency record does not.

Where possible, use database transactions and uniqueness constraints to coordinate the business effect and its idempotency record.

### Idempotency is not the same as retrying

Retries determine **whether to attempt an operation again**.

Idempotency determines **whether repeating that operation causes duplicate effects**.

Both are often needed for reliable APIs.

---

## 6. Circuit Breakers: Stop Calling a Failing Dependency

A circuit breaker prevents an application from repeatedly calling a dependency that is currently failing or too slow.

It is inspired by an electrical circuit breaker: when a problem is detected, the circuit opens and interrupts further attempts.

### The three states

**1. Closed**

Requests flow normally. The circuit monitors failures and latency.

**2. Open**

Requests to the dependency are rejected quickly, or a defined fallback is used, without making the usual downstream call.

**3. Half-open**

After a recovery interval, a limited number of test requests are allowed. If the dependency appears healthy, the circuit closes. If failures continue, it opens again.

```text
          Failures exceed threshold
  +----------------------------------+
  |                                  v
CLOSED --------------------------> OPEN
  ^                                  |
  |                                  | Recovery interval
  |                                  v
  +----------------------------- HALF-OPEN
             Success                  |
                                      |
                         Failure -----+
                         back to OPEN
```

This is a simplified state diagram. Actual circuit breakers may use different thresholds and transition rules.

### Why circuit breakers help

Without a circuit breaker:

```text
Application
    |
    | Request 1: fails
    | Request 2: fails
    | Request 3: fails
    | Request 4: fails
    | ...
    v
Failing Dependency
```

The application continues spending resources on requests that are unlikely to succeed.

With a circuit breaker, repeated failures can cause the application to stop making normal downstream calls temporarily.

### Circuit breakers do not replace timeouts

A timeout limits how long an individual operation can wait.

A circuit breaker limits repeated calls to a dependency when its failure behavior crosses a configured threshold.

Use both where appropriate.

### Choosing a fallback

A fallback is an alternative response or behavior when the primary dependency is unavailable.

Examples:

- Display cached, clearly identified data for a noncritical dashboard.
- Return a temporary-unavailable response for a required payment operation.
- Queue a noncritical notification for later delivery.
- Show a booking's last known status with its timestamp.

A fallback must not misrepresent the system's state. For example, an old booking status must not be presented as guaranteed current.

For a critical booking or payment operation, a fallback should not pretend the transaction succeeded if the result is unknown.

### Trade-offs

- A circuit can open during a temporary burst of failures.
- Thresholds that are too sensitive can reject requests unnecessarily.
- Thresholds that are too relaxed can allow cascading failures.
- Recovery probes must be limited so they do not overwhelm the recovering service.
- Circuit-breaker state and metrics need to be observable.

---

## 7. Bulkheads: Isolate Resources

Bulkheads divide a system into isolated resource pools so that a failure in one workload does not consume all resources needed by another.

The name comes from compartments in ships that limit the spread of flooding.

### Example: Booking and reporting

Imagine the application supports:

- Booking creation.
- Booking search.
- Large reporting queries.

If reporting consumes every database connection, booking creation may stop working even though booking queries are normally fast.

A bulkhead strategy might separate the workloads:

```text
                  Application
                  /         \
                 /           \
        Booking workload   Reporting workload
               |                  |
        Reserved capacity   Limited capacity
               |                  |
               v                  v
          Database pool      Reporting pool
```

The precise implementation might use separate connection pools, concurrency limits, worker pools, queues, or other resource boundaries.

### Why bulkheads matter

Without isolation, one workload can monopolize:

- Database connections.
- Worker threads.
- CPU capacity.
- Memory.
- Outbound HTTP connections.
- Queue consumers.

With bulkheads, the application reserves or limits capacity so that critical workloads can continue when noncritical workloads become expensive.

### Bulkheads versus horizontal scaling

Horizontal scaling adds capacity.

Bulkheads isolate capacity.

Adding more application servers may not solve a database connection bottleneck if every server shares the same database connection limit. Resource isolation and downstream capacity must be considered together.

### Important limitation

Separate application pools do not automatically isolate a shared database. If both pools execute expensive queries against the same overloaded database, the database can still become the common failure point.

Bulkheads are most effective when designed across the relevant resource boundaries.

---

## 8. Graceful Degradation

Graceful degradation means reducing nonessential functionality while preserving the most important user journeys.

A system does not always need to be completely healthy to remain useful.

### Example: A booking platform

Suppose the system has these capabilities:

| Capability | Priority during an incident |
|---|---|
| Create a booking | Critical |
| Confirm a booking's persisted state | Critical |
| Show live driver location | Important, but may tolerate a temporary stale value |
| Send an email notification | Often deferrable |
| Generate an analytics report | Usually deferrable |

If the analytics service fails, booking creation should not necessarily fail with it.

If live tracking is unavailable, the UI might show the last known position and its timestamp.

If the email provider is down, the system might queue the notification while still completing the booking, provided the notification is not a prerequisite for the business transaction.

### Graceful degradation strategies

- Disable nonessential features temporarily.
- Serve cached data where staleness is acceptable.
- Queue background tasks for later processing.
- Reduce expensive optional computations.
- Limit optional recommendations or analytics.
- Return a clear partial-service response.

### The key design question

When a dependency fails, ask:

**Which user journeys must continue, and which features can be delayed, simplified, or temporarily disabled?**

This should be decided before an incident, not improvised during one.

---

## 9. Load Shedding and Backpressure

Sometimes the incoming workload exceeds what the system can safely process.

In this situation, accepting every request can make the entire system slower and less reliable.

### Load shedding

Load shedding means rejecting or dropping lower-priority work when the system is overloaded.

Examples:

- Reject excess requests with a temporary-unavailable response.
- Apply per-tenant rate limits.
- Prioritize booking creation over expensive reporting.
- Reject requests before they occupy scarce database connections.

Load shedding is not ideal for users, but controlled rejection can be better than allowing every request to time out.

### Backpressure

Backpressure communicates that a downstream component cannot accept more work at the current rate.

For example:

```text
Producer: 500 messages/second
                 |
                 v
              Queue
                 |
                 v
Consumer capacity: 300 messages/second
```

The queue backlog grows by 200 messages per second if these rates remain constant.

A queue can absorb a temporary burst, but it cannot solve a permanent mismatch between arrival rate and processing capacity.

Backpressure may involve slowing producers, limiting concurrency, pausing consumption, or rejecting excess work.

### How to decide what to shed

Consider:

- Business criticality.
- Tenant or user fairness.
- Cost of processing.
- Whether work can be retried safely.
- Whether work can be queued.
- The remaining system capacity.

The goal is to protect critical operations, not merely to reject requests randomly.

---

## 10. Health Checks and Recovery

Health checks help infrastructure determine whether an application instance can receive traffic.

Two common concepts are:

**Liveness check:** Is the process alive, or is it stuck in a state where it should be restarted?

**Readiness check:** Is the instance ready to serve the requests it is expected to handle?

These checks answer different questions.

### Example

An application process may be alive but unable to acquire a database connection. It may not be ready to accept booking requests.

Conversely, if every application instance marks itself unhealthy whenever a shared database briefly slows down, the load balancer might remove every instance, making the incident worse.

### Health-check design principles

- Keep liveness checks focused on whether the process itself can continue.
- Use readiness checks to determine whether the instance can serve its intended traffic.
- Avoid expensive health checks that add load to an already overloaded dependency.
- Define how long a failure must persist before removing an instance.
- Define recovery behavior and re-entry into traffic.
- Avoid synchronized restarts that cause all instances to reconnect at once.

Health checks help manage traffic, but they do not replace timeouts, resource isolation, or dependency-level monitoring.

---

## 11. Putting the Patterns Together

Consider a booking API that depends on PostgreSQL and an external payment provider.

A resilient design might use the following protections:

| Failure risk | Protection | Purpose |
|---|---|---|
| Database query takes too long | Query timeout and request deadline | Bound waiting time |
| Temporary network failure | Bounded retry with backoff and jitter | Recover from transient errors |
| Repeated payment-provider failures | Circuit breaker | Stop repeated calls to a failing dependency |
| Reporting consumes all resources | Bulkheads and concurrency limits | Protect booking capacity |
| Client retries after a lost response | Idempotency key | Prevent duplicate business effects |
| Notifications provider is unavailable | Durable queue | Delay noncritical work |
| Traffic exceeds safe capacity | Rate limiting and load shedding | Protect the system |
| Live tracking dependency is unavailable | Explicit stale-data fallback | Preserve useful functionality without hiding staleness |

These mechanisms work together. None is a complete solution on its own.

For example, a circuit breaker without timeouts might still leave individual requests waiting too long. Retries without idempotency might create duplicate bookings. Bulkheads without database capacity planning might only move the bottleneck.

---

## 12. Worked Example: Booking API Under Failure

### Initial conditions

Assume:

- Incoming traffic: 150 requests per second.
- Normal database latency: 40 ms.
- Database latency during an incident: 4 seconds.
- The application has a limited database connection pool.
- The client retries requests when it receives a timeout.

### What can go wrong?

1. Database queries become slow.
2. Connections remain occupied for longer.
3. The connection pool becomes exhausted.
4. Requests time out.
5. Clients retry.
6. Retry traffic increases the database load.
7. More requests fail, causing a cascading failure.

### A better design

**Step 1: Bound the wait.** Use connection-acquisition timeouts, database query timeouts, and an overall request deadline.

**Step 2: Protect the database.** Limit concurrency and prevent noncritical reporting from consuming booking capacity.

**Step 3: Retry only where justified.** Use bounded retries with backoff and jitter for transient failures. Do not retry blindly when the deadline has expired.

**Step 4: Prevent duplicate bookings.** Require a stable idempotency key for retries of the same booking-creation operation.

**Step 5: Stop repeated calls to a failing external dependency.** Use a circuit breaker where appropriate.

**Step 6: Degrade noncritical functionality.** Delay notifications and analytics rather than coupling them unnecessarily to the booking response.

**Step 7: Reject excess load deliberately.** If the system reaches safe capacity, reject excess work before it exhausts all resources.

**Step 8: Monitor recovery.** Track database latency, connection-pool saturation, request timeouts, retry rates, error rates, and recovery behavior.

The objective is not to guarantee that every request succeeds during every incident. It is to prevent a local failure from becoming a total outage and to preserve the most important operations.

---

## 13. Common Mistakes

1. **Retrying every failure:** Permanent failures are repeated unnecessarily, and overloaded dependencies receive even more traffic.

2. **Using timeouts without an overall deadline:** Each downstream operation may consume its entire timeout, making the total request excessively slow.

3. **Assuming a timeout means the transaction failed:** The operation may have committed even though the response was lost.

4. **Retrying non-idempotent operations without safeguards:** Duplicate bookings, payments, or orders may be created.

5. **Using circuit breakers as a replacement for timeouts:** Individual requests can still hang while the circuit is closed or half-open.

6. **Adding application servers without checking the database:** More instances can increase pressure on a shared database.

7. **Failing every feature when one dependency is unavailable:** Noncritical functionality can unnecessarily take down critical user journeys.

8. **Returning stale data without identifying it:** Users may believe outdated information is current.

9. **Accepting unlimited work into a queue:** The queue can accumulate an unbounded backlog when processing capacity is insufficient.

10. **Ignoring recovery behavior:** A recovering dependency can be overwhelmed by synchronized retries or a sudden return of traffic.

11. **Relying only on health checks:** Health checks do not replace dependency timeouts, concurrency controls, and meaningful observability.

12. **Failing to test incidents:** A resilience mechanism that has never been exercised may not work as expected when needed.

---

## 14. Practical Resilience Checklist

Before considering a service production-ready, ask:

### Request protection
- [ ] Are connection, query, and request timeouts configured?
- [ ] Is there an overall request deadline?
- [ ] Are concurrency and connection-pool limits defined?
- [ ] Can excess traffic be rejected safely?

### Retry safety
- [ ] Are retries bounded?
- [ ] Is exponential backoff with jitter used where appropriate?
- [ ] Are permanent failures excluded from retries?
- [ ] Is retry behavior coordinated across application layers?
- [ ] Are non-idempotent operations protected against duplicate effects?

### Dependency protection
- [ ] Are circuit breakers appropriate for critical remote dependencies?
- [ ] Are bulkheads used to isolate critical and noncritical workloads?
- [ ] Are fallbacks safe and honest about data freshness?
- [ ] Can nonessential work be queued or delayed?

### Monitoring and recovery
- [ ] Are timeouts, retries, dependency latency, and error rates monitored?
- [ ] Are pool exhaustion and queue backlog visible?
- [ ] Are liveness and readiness checks designed appropriately?
- [ ] Has recovery behavior been tested?
- [ ] Are incident thresholds and operational procedures documented?

---

## 15. Revision Questions

Try answering these without looking at the notes.

1. What is the difference between reliability, availability, and resilience?
2. How can a slow database cause a cascading failure?
3. What is the difference between an operation timeout and an overall deadline?
4. Why can retrying a failed request make an outage worse?
5. What do exponential backoff and jitter accomplish?
6. Why is a timeout not proof that a booking failed?
7. How does an idempotency key prevent duplicate bookings?
8. What are the three circuit-breaker states?
9. How do bulkheads protect critical operations?
10. What is graceful degradation? Give two examples.
11. What is the difference between load shedding and backpressure?
12. Why should liveness and readiness checks be treated differently?
13. Why might adding more application servers make a database bottleneck worse?
14. Which protections would you use if a payment provider becomes slow?
15. How would you design a booking API so that a temporary database slowdown does not bring down the entire platform?

## Final Takeaway

A resilient system assumes that dependencies will fail, networks will be unreliable, traffic will spike, and responses may be lost.

Design for those conditions using **timeouts, deadlines, bounded retries, idempotency, circuit breakers, bulkheads, graceful degradation, load shedding, and observability**.

The goal is to control the impact of failures, protect critical operations, and recover safely.


## Quick practice exercise
Imagine your booking platform has these conditions:
- PostgreSQL becomes slow.
- The client times out while creating a booking.
- The client retries the request.
- Meanwhile, the notification service is unavailable.
Answer these three questions in your own words:
1. How would you prevent the retry from creating a duplicate booking?
2. How would you protect the application and database while PostgreSQL is slow?
3. Should the booking fail just because the notification service is unavailable? Why or why not?