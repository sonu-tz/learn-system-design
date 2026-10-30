# Lesson 25 — Rate Limiting, Throttling, Load Shedding, and Traffic Management

As applications grow, handling more traffic is not just about adding servers. We also need to control how much traffic enters the system, protect expensive resources, and ensure that important operations remain available during traffic spikes.

In this lesson, we'll explore rate limiting algorithms, distributed rate limiting, throttling, load shedding, quotas, and traffic management strategies.

## 1. Why Do We Need Traffic Management?

Imagine our ride-booking platform normally handles:

- 1,000 API requests per second.
- 500 booking requests per second during peak hours.
- 20,000 driver location updates per second.
- 5,000 active WebSocket connections.

Suddenly, a large event ends and 50,000 passengers open the application simultaneously.

Incoming traffic increases dramatically.

Without traffic controls, the system may experience:

- API server overload.
- Database connection exhaustion.
- Excessive CPU and memory usage.
- Message broker backlog.
- Increased latency.
- Cascading failures.

Adding more servers is one solution, but it may not be fast enough or economically practical.

We need mechanisms to control traffic.

### Four Important Concepts

| Concept | Purpose |
|---|---|
| Rate Limiting | Restricts how many requests a client can make |
| Throttling | Controls or slows the rate of incoming work |
| Load Shedding | Rejects lower-priority work when the system is overloaded |
| Traffic Shaping | Controls when and how traffic is delivered |

These concepts are related, but they solve different problems.

---

## 2. Rate Limiting

Rate limiting restricts the number of requests a client can make within a defined period.

Example:

```text
Maximum: 100 requests
Window: 1 minute
```

If a client makes 100 requests within one minute, additional requests are rejected until capacity becomes available.

### Example

```text
Client
  |
  | 1st request
  v
Rate Limiter
  |
  | Allowed
  v
API Server

Client
  |
  | 101st request
  v
Rate Limiter
  |
  | Rejected
  v
HTTP 429 Too Many Requests
```

### Why Rate Limiting?

1. Prevent abuse.
2. Protect backend resources.
3. Prevent brute-force attacks.
4. Control third-party API usage.
5. Enforce customer subscription quotas.
6. Maintain fair resource allocation.

### Where Can We Apply It?

- API Gateway.
- Load Balancer.
- Application Middleware.
- Authentication Service.
- Individual Backend Services.
- Message Consumers.
- External Provider Integrations.

The best location depends on what resource we are protecting.

---

## 3. Rate Limiting Algorithms

There are several algorithms, each with different characteristics.

### 3.1 Fixed Window Counter

The simplest rate-limiting algorithm.

Example:

```text
Limit: 100 requests
Window: 60 seconds
```

The system counts requests within fixed time intervals.

```text
10:00:00 - 10:00:59 -> 100 requests
10:01:00 - 10:01:59 -> 100 requests
```

If a client reaches 100 requests during the first window, additional requests are rejected until the next window.

**Problem: Boundary Burst**

Imagine:

```text
10:00:59 -> 100 requests
10:01:00 -> 100 requests
```

The client can make 200 requests in approximately two seconds.

This happens because the counter resets at the window boundary.

Advantages:
- Simple implementation.
- Low memory usage.
- Easy to distribute using shared counters.

Disadvantages:
- Allows boundary bursts.
- Does not provide smooth traffic control.

Use it when simplicity is more important than precise burst control.

### 3.2 Sliding Window Log

Instead of maintaining one counter, store timestamps of requests.

For every incoming request:

1. Remove timestamps older than the rolling window.
2. Count remaining timestamps.
3. Reject if the limit is reached.
4. Otherwise, record the new timestamp.

Example:

```text
Limit: 5 requests
Window: 60 seconds

Current timestamps:
10:00:10
10:00:20
10:00:30
10:00:40
10:00:50
```

A new request at 10:00:55 is rejected.

A request at 10:01:15 can be allowed because the earliest timestamp has expired from the rolling window.

Advantages:
- Accurate rolling-window enforcement.
- Avoids fixed-window boundary bursts.

Disadvantages:
- Stores individual request timestamps.
- Higher memory and processing overhead.

### 3.3 Sliding Window Counter

This algorithm combines the previous and current fixed-window counters.

It estimates the request count in a rolling window by weighting the previous window.

Example:

```text
Limit: 100 requests/minute

Previous window count: 80
Current window count: 40
Current window progress: 25%
```

Estimated count:

```text
Estimated Count =
Previous Count × (1 - Window Progress)
+ Current Count
```

Therefore:

```text
= 80 × (1 - 0.25) + 40

= 60 + 40

= 100 requests
```

The request would be rejected if the configured limit is 100.

Advantages:
- Lower memory usage than a full sliding-window log.
- Smoother enforcement than a fixed window.

Disadvantages:
- It is an approximation.
- It assumes traffic was distributed reasonably across the previous window.

### 3.4 Token Bucket

Token Bucket is one of the most useful algorithms for API traffic management.

Imagine a bucket that holds tokens.

- Tokens are added at a fixed rate.
- Each request consumes one token.
- If no token is available, the request is rejected or delayed.

Example:

```text
Bucket Capacity: 100 tokens
Refill Rate: 10 tokens/second
```

A client can make 100 requests immediately if the bucket is full.

After that, it can continue at an average rate of 10 requests per second.

```text
         Token Refill
         10 tokens/sec
               |
               v
         +-----------+
         |   Bucket  |
         | 100 max   |
         +-----------+
               |
          Request
               |
          Consume 1
               |
         Token Available?
            /     \
          Yes      No
           |        |
           v        v
         Allow    Reject
```

Advantages:
- Allows controlled bursts.
- Controls average request rate.
- Flexible configuration.

Disadvantages:
- Requires state management.
- Distributed implementations need coordination.

### 3.5 Leaky Bucket

Leaky Bucket processes incoming requests at a controlled rate.

Imagine a bucket with a small hole.

Requests enter the bucket, but leave at a steady rate.

```text
Incoming Requests
       |
       v
  +----------+
  |  Queue   |
  |          |
  |          |
  +----------+
       |
       | Fixed processing rate
       v
    Backend
```

If requests arrive faster than they can be processed, the queue grows.

When the queue is full, additional requests are rejected.

Advantages:
- Smooths traffic.
- Helps protect downstream systems.
- Useful for controlled processing pipelines.

Disadvantages:
- Can introduce waiting time.
- Requires queue capacity management.
- Does not necessarily provide immediate responses.

### Algorithm Comparison

| Algorithm | Burst Support | Accuracy | Memory |
|---|---|---|---|
| Fixed Window | Boundary bursts | Low | Low |
| Sliding Window Log | Controlled | High | High |
| Sliding Window Counter | Controlled | Approximate | Low |
| Token Bucket | Yes | Good | Low |
| Leaky Bucket | Smooth output | Good | Queue-dependent |

**Important:** Token Bucket is often a good choice for APIs that should allow short bursts while controlling sustained traffic.

---

## 4. Distributed Rate Limiting

A rate limiter on one application server is not sufficient when we have multiple servers.

Imagine:

```text
              Load Balancer
                    |
          +---------+---------+
          |         |         |
          v         v         v
        App 1     App 2     App 3
```

Suppose each server independently allows:

```text
100 requests/minute
```

A client can potentially make:

```text
100 × 3 = 300 requests/minute
```

The actual limit is no longer 100.

### Solution: Shared Rate-Limit State

Use a shared state store such as Redis.

```text
              Load Balancer
                    |
          +---------+---------+
          |         |         |
          v         v         v
        App 1     App 2     App 3
          |         |         |
          +---------+---------+
                    |
                    v
                   Redis
              Shared Counters
```

All servers use the same rate-limit state.

### Atomic Operations

A rate-limit decision often involves multiple operations:

1. Read current state.
2. Check the limit.
3. Update the state.
4. Decide whether to allow the request.

If these operations are not atomic, concurrent requests can exceed the configured limit.

Use atomic operations, such as Redis Lua scripts or suitable transactional primitives, to ensure the decision and state update happen together.

### Important Trade-offs

Distributed rate limiting introduces:
- Network latency to the state store.
- Redis availability dependency.
- State expiration management.
- Hot keys for high-volume clients.
- Cross-region coordination challenges.

For very high-throughput systems, consider hierarchical or approximate rate limiting instead of a centralized lookup for every request.

---

## 5. Rate-Limiting Strategies

Rate limits can be applied using different identifiers.

### 5.1 IP-Based Rate Limiting

Example:

```text
100 requests/minute/IP
```

Useful for:
- Public APIs.
- Anonymous endpoints.
- Basic abuse protection.

Limitations:
- Multiple users may share one IP.
- Attackers may rotate IP addresses.
- Proxy headers must be handled carefully.

Only trust forwarded IP headers from known proxies.

### 5.2 User-Based Rate Limiting

Example:

```text
1000 requests/minute/user
```

Useful for authenticated applications.

The user identity must come from trusted authentication context.

### 5.3 API-Key-Based Rate Limiting

Example:

```text
Customer A: 10,000 requests/day
Customer B: 100,000 requests/day
```

Useful for public APIs and enterprise customers.

### 5.4 Endpoint-Based Rate Limiting

Different APIs consume different resources.

Example:

| Endpoint | Limit |
|---|---|
| GET /profile | 100/minute |
| POST /bookings | 20/minute |
| POST /otp | 5/15 minutes |
| GET /search | 60/minute |
| POST /login | Risk-based limits |

These are illustrative limits, not universal recommendations.

### 5.5 Resource-Based Rate Limiting

Some operations consume more resources than others.

For example:
- A simple profile lookup may be inexpensive.
- A complex report generation request may be expensive.
- A video processing request may consume substantial CPU.

We can assign different costs to operations.

Example:

```text
Profile API: 1 token
Search API: 5 tokens
Report API: 20 tokens
```

This is useful when request count alone does not represent actual resource consumption.

---

## 6. Rate-Limit Responses

When a request exceeds the configured limit, the API commonly returns:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

Response:

```json
{
  "error": "RATE_LIMIT_EXCEEDED",
  "message": "Too many requests. Please try again later."
}
```

Useful response headers may include:

```text
RateLimit-Limit
RateLimit-Remaining
RateLimit-Reset
Retry-After
```

The exact header conventions depend on the API and standard being followed.

Clients should:
- Respect retry information.
- Avoid immediate retry loops.
- Use exponential backoff with jitter for retryable failures.
- Avoid retrying requests that are not safe to repeat unless idempotency is supported.

---

## 7. Throttling

Rate limiting and throttling are often used interchangeably, but they describe different control behaviors.

**Rate limiting** defines how much traffic is permitted.

**Throttling** controls or slows the rate at which work is accepted or processed.

Example:

A message-processing system receives 1,000 messages per second, but the downstream service can safely process only 300 per second.

Instead of forwarding everything immediately, we throttle delivery to approximately 300 messages per second.

```text
Incoming: 1,000 messages/sec
                |
                v
           Throttling
                |
                v
Outgoing: 300 messages/sec
                |
                v
          Downstream API
```

### Where Is Throttling Useful?

- Third-party API integrations.
- Message consumers.
- Database-intensive background jobs.
- Email and SMS providers.
- File processing pipelines.
- Real-time event delivery.

### Consumer Throttling Example

Suppose Kafka receives:

```text
10,000 messages/sec
```

Our consumer can process:

```text
3,000 messages/sec
```

We can limit consumer processing to protect the database.

The backlog will grow by approximately:

```text
10,000 - 3,000 = 7,000 messages/sec
```

Throttling protects the database, but it does not solve insufficient processing capacity.

We must also monitor queue depth and oldest-message age.

---

## 8. Load Shedding

Load shedding means deliberately rejecting or dropping work when a system is overloaded.

It sounds undesirable, but it can protect the system from total failure.

Imagine:

```text
Incoming Traffic: 10,000 RPS
Safe Capacity:     4,000 RPS
```

If we accept all 10,000 requests:
- Queues grow.
- Latency increases.
- Connections remain occupied.
- Timeouts increase.
- Retries amplify traffic.
- The system may collapse.

Instead, we shed excess traffic.

```text
Incoming Traffic
      |
      v
Load Management
      |
      +------> Critical Requests
      |             |
      |             v
      |          Accepted
      |
      +------> Low Priority Requests
      |             |
      |             v
      |           Rejected
      |
      +------> Excess Traffic
                    |
                    v
                 Rejected
```

### Priority-Based Load Shedding

Consider these operations:

| Operation | Priority |
|---|---|
| Emergency trip updates | Critical |
| Booking confirmation | High |
| Driver assignment | High |
| Profile picture loading | Low |
| Promotional recommendations | Low |
| Analytics events | Low |

When overloaded, preserve critical functionality and reduce nonessential work.

### Graceful Degradation

Instead of rejecting everything, reduce functionality.

Examples:
- Disable recommendation features.
- Serve cached data.
- Reduce map refresh frequency.
- Delay analytics processing.
- Temporarily disable expensive reports.

This is graceful degradation.

The core user journey should remain available as much as possible.

---

## 9. Backpressure

Backpressure is a mechanism by which downstream systems communicate that they cannot accept more work at the current rate.

Example:

```text
Producer
   |
   v
Message Broker
   |
   v
Consumer
   |
   v
Database
```

Suppose the database is overloaded.

The consumer should reduce processing rather than continuously sending more queries.

Possible strategies:
- Pause consumption.
- Reduce concurrency.
- Reduce batch size.
- Limit in-flight messages.
- Apply bounded queues.
- Slow producers where supported.

### Backpressure vs Load Shedding

| Backpressure | Load Shedding |
|---|---|
| Slows incoming work | Rejects or drops selected work |
| Tries to regulate flow | Protects capacity during overload |
| Can preserve work in queues | May intentionally discard work |
| Useful for producer-consumer systems | Useful during severe overload |

### Important Principle

A queue is not an unlimited solution.

If incoming traffic consistently exceeds processing capacity, backlog grows indefinitely until storage or operational limits are reached.

---

## 10. Concurrency Limiting

Rate limiting controls requests over time.

Concurrency limiting controls how many operations can run simultaneously.

Example:

```text
Maximum concurrent report jobs: 20
```

Even if 100 users submit reports simultaneously, only 20 jobs execute at a time.

The remaining jobs wait in a bounded queue or are rejected.

### Why Is Concurrency Limiting Useful?

Some requests are much more expensive than others.

For example:
- A simple API request may complete in 50ms.
- A complex report may take 30 seconds.
- A video processing job may take several minutes.

Limiting concurrent expensive work protects CPU, memory, database connections, and downstream services.

### Example

Suppose each report consumes 500 MB of memory.

Running 100 reports simultaneously would require approximately:

```text
100 × 500 MB = 50 GB
```

If the available worker memory is 16 GB, this is unsafe.

A concurrency limit of 20 would require approximately:

```text
20 × 500 MB = 10 GB
```

This leaves additional memory for the runtime and other processes, although actual sizing must include overhead.

---

## 11. Quotas vs Rate Limits

These concepts are related but different.

### Rate Limit

Controls request frequency.

Example:

```text
100 requests/minute
```

### Quota

Controls total consumption over a longer period.

Example:

```text
1 million API requests/month
```

A customer might have:

| Dimension | Limit |
|---|---:|
| Requests per second | 50 |
| Requests per day | 100,000 |
| Requests per month | 1 million |
| Concurrent jobs | 10 |

Rate limits protect short-term capacity.

Quotas control longer-term usage and commercial entitlements.

A system may enforce both.

---

## 12. Traffic Prioritization

Not all traffic has equal business importance.

A booking confirmation is more important than loading a promotional banner.

A good architecture assigns priorities to different workloads.

Example:

```text
                  Incoming Traffic
                         |
                         v
                  Priority Router
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
          Critical      Normal       Low
             |           |           |
             v           v           v
          Dedicated   Shared Pool  Limited Pool
          Capacity
```

### Priority Scheduling

Possible approaches:
- Strict priority.
- Weighted fair scheduling.
- Round-robin across priority classes.
- Reserved capacity for critical operations.

### Strict Priority Problem

If critical traffic is continuously high, low-priority work may never execute.

This is called starvation.

Weighted scheduling helps ensure that lower-priority workloads still receive some capacity.

---

## 13. Adaptive Traffic Management

Static rate limits are easy to understand, but real systems experience changing traffic and resource availability.

Adaptive systems adjust traffic based on current conditions.

Example:

```text
If DB connections < 60%:
    Allow normal traffic

If DB connections > 80%:
    Reduce expensive operations

If DB connections > 95%:
    Shed noncritical traffic
```

Other signals:
- CPU utilization.
- Memory pressure.
- Queue depth.
- Request latency.
- Error rate.
- Database connection saturation.
- Downstream dependency health.

### Important Caution

Adaptive control must be stable.

If limits change too frequently or aggressively, the system can oscillate between allowing and rejecting traffic.

Use:
- Hysteresis.
- Minimum and maximum limits.
- Cooldown periods.
- Smoothed measurements.
- Clear recovery conditions.

---

## 14. Traffic Management at Different Layers

A production system should protect resources at multiple layers.

```text
                 Internet
                    |
                    v
               CDN / WAF
                    |
                    v
             Load Balancer
                    |
                    v
                API Gateway
                    |
                    v
              Application
                    |
           +--------+--------+
           |                 |
           v                 v
        Redis             PostgreSQL
           |
           v
       Message Broker
           |
           v
         Workers
```

### CDN / WAF

Protects public entry points.

Controls:
- DDoS mitigation.
- Bot protection.
- Request filtering.
- Geographic restrictions where appropriate.

### API Gateway

Controls:
- Client and API-key rate limits.
- Authentication.
- Request-size limits.
- Per-endpoint quotas.

### Application Layer

Controls:
- Business-specific limits.
- User-level permissions.
- Expensive-operation concurrency.
- Tenant quotas.

### Database Layer

Controls:
- Connection pools.
- Query timeouts.
- Maximum concurrent operations.
- Workload isolation.

### Message Consumers

Controls:
- Consumer concurrency.
- Batch sizes.
- Prefetch.
- Processing rate.
- Backpressure.

No single layer should be expected to protect the entire system.

---

## 15. Worked Example — Ride-Booking Traffic Spike

Suppose our platform normally handles:

```text
Booking API:       500 RPS
Driver Updates:  5,000 RPS
Search API:        200 RPS
```

After a major event, traffic increases:

```text
Booking API:     3,000 RPS
Driver Updates: 20,000 RPS
Search API:      2,000 RPS
```

Our safe capacities are:

| Component | Safe Capacity |
|---|---:|
| Booking Service | 1,000 RPS |
| Driver Update Service | 10,000 RPS |
| Search Service | 500 RPS |

### Step 1 — Apply Rate Limits

Protect each API using appropriate per-user, per-tenant, and global limits.

### Step 2 — Prioritize Work

Preserve booking creation and critical trip updates.

Reduce low-priority search features and nonessential analytics.

### Step 3 — Apply Concurrency Limits

Prevent expensive database operations from exhausting connection pools.

### Step 4 — Buffer Asynchronous Work

Queue suitable background tasks such as analytics and notifications.

### Step 5 — Apply Backpressure

Slow consumers if downstream databases or providers become saturated.

### Step 6 — Shed Excess Traffic

Reject requests that exceed safe capacity rather than allowing the entire application to become unhealthy.

### Step 7 — Recover Gradually

As traffic decreases and dependencies recover, restore normal limits gradually.

**The goal is not to accept every request. The goal is to keep the most important user journeys working.**

---

## 16. Traffic Management Metrics

Monitor the following:

### Rate Limiting
- Requests allowed.
- Requests rejected.
- Limit utilization.
- Rejection percentage.

### Throttling
- Incoming rate.
- Processing rate.
- Throttled request count.
- Waiting duration.

### Load Shedding
- Rejected requests by priority.
- Shed workload percentage.
- Critical request success rate.

### Backpressure
- Queue depth.
- Oldest message age.
- Producer wait time.
- Consumer processing rate.

### Resource Saturation
- CPU.
- Memory.
- Database connections.
- Worker concurrency.
- Network bandwidth.

### User Experience
- P95/P99 latency.
- Error rate.
- Successful booking rate.
- Critical workflow completion rate.

---

## 17. Common Traffic Management Mistakes

1. Applying the same rate limit to every endpoint.
2. Using independent in-memory counters across multiple servers.
3. Ignoring rate-limit state-store failures.
4. Retrying immediately after HTTP 429.
5. Allowing unbounded queues.
6. Treating rate limiting as a replacement for capacity planning.
7. Rejecting critical and noncritical traffic equally.
8. Limiting requests but ignoring concurrent expensive operations.
9. Allowing low-priority work to starve critical workflows.
10. Increasing consumer concurrency without considering database capacity.
11. Using static thresholds without understanding actual resource saturation.
12. Forgetting to recover traffic gradually after an incident.

---

## 18. Revision Questions

1. What is rate limiting, and why is it needed?
2. Explain fixed-window rate limiting.
3. What is the boundary burst problem?
4. How does sliding-window log differ from sliding-window counter?
5. Explain Token Bucket with an example.
6. How does Leaky Bucket control traffic?
7. What is distributed rate limiting?
8. Why is Redis commonly used for shared rate-limit state?
9. What is the difference between IP-based and user-based rate limiting?
10. What is the difference between rate limiting and throttling?
11. What is load shedding?
12. Explain graceful degradation.
13. What is backpressure?
14. How does backpressure differ from load shedding?
15. What is concurrency limiting?
16. Explain rate limits versus quotas.
17. Why can strict priority scheduling cause starvation?
18. How would you protect PostgreSQL during a traffic spike?
19. How would you protect critical booking operations?
20. What metrics would you monitor during an overload incident?

---

## 19. Practical Exercise — Protect the Ride-Booking Platform

Assume the following traffic:

| API | Normal RPS | Peak RPS | Safe Capacity |
|---|---:|---:|---:|
| Booking API | 500 | 3,000 | 1,000 |
| Driver Location API | 5,000 | 20,000 | 10,000 |
| Search API | 200 | 2,000 | 500 |
| Notification API | 300 | 5,000 | 1,000 |

Design a traffic management strategy.

Answer:

1. Which rate-limiting algorithm would you use for each API?
2. Would you use IP-based, user-based, tenant-based, or global limits?
3. Which operations should receive reserved capacity?
4. Which operations can be queued?
5. Which operations should be shed first?
6. How would you protect PostgreSQL?
7. How would you handle Redis rate-limiter failure?
8. How would you prevent unbounded queue growth?
9. How would you adjust traffic limits dynamically?
10. How would you measure whether the system recovered?

### Final Takeaway

A scalable system does not simply accept more traffic.

It controls traffic at multiple layers:

- **Rate Limiting:** Restricts request frequency.
- **Throttling:** Controls processing speed.
- **Concurrency Limiting:** Restricts simultaneous work.
- **Backpressure:** Signals upstream systems to slow down.
- **Load Shedding:** Rejects less important work during overload.
- **Traffic Prioritization:** Protects critical business operations.

The ultimate goal is to protect system stability and preserve the most important user experiences under pressure.

### Practice focus

Work through the traffic-spike exercise. The most important design decision is deciding what to protect, what to queue, and what to reject when demand exceeds capacity.