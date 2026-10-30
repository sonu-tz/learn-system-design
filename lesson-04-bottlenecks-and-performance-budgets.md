# Lesson 4: Bottlenecks and Performance Budgets

## 1. Learning Objectives

By the end of this lesson, you should be able to:

- Explain what a system bottleneck is.
- Identify bottlenecks across frontend, backend, database, network, and external services.
- Understand how a request's latency is distributed across different components.
- Use logs, metrics, traces, and profiling to investigate performance problems.
- Understand performance budgets and service-level objectives.
- Decide when to scale, optimize queries, introduce caching, or reduce unnecessary work.
- Recognize the trade-offs of common performance optimizations.
- Understand why optimizing one component does not always improve the complete system.
- Apply a systematic performance investigation process.

---

## 2. What Is a Bottleneck?

A bottleneck is a component or resource that limits the performance, throughput, or capacity of a system.

Imagine a booking platform with this architecture:

```text
Users
  |
  v
Frontend
  |
  v
Load Balancer
  |
  v
Application Servers
  |
  v
PostgreSQL
  |
  v
External Services
```

Any component along this path could become the bottleneck.

Examples:

- The frontend performs too much JavaScript work.
- The API server spends too much time processing requests.
- The database executes expensive queries.
- The connection pool is exhausted.
- An external service responds slowly.
- The network transfers large payloads.
- A message queue accumulates work faster than workers can process it.

A system can have several bottlenecks, and the limiting component can change as traffic increases.

### Example

Suppose an API has:

- Application CPU utilization: 35%.
- Database CPU utilization: 95%.
- High database query latency.
- API P99 latency: 2 seconds.

The database is a strong bottleneck hypothesis.

Adding application servers might increase concurrent database requests and make the problem worse.

**Key principle:** Identify the component limiting the system before choosing an optimization.

---

## 3. Understand the Complete Request Lifecycle

Consider a user opening a booking details page.

```text
User opens booking
       |
       v
Frontend sends API request
       |
       v
Network / gateway
       |
       v
Authentication and validation
       |
       v
Application business logic
       |
       v
Database connection acquisition
       |
       v
Database query execution
       |
       v
Response serialization
       |
       v
Network response
       |
       v
Frontend processing and rendering
```

The complete user experience includes more than backend processing.

A request can be slow because of:

- Network latency.
- Application processing.
- Connection-pool waiting.
- Database execution.
- External API calls.
- Serialization and payload size.
- Frontend JavaScript execution.
- UI rendering.

The first task is to determine where the time is being spent.

---

## 4. Decompose Request Latency

Suppose a booking API has an end-to-end latency of 900 ms.

An illustrative timing breakdown might be:

| Stage | Time |
|---|---:|
| Network and gateway overhead | 50 ms |
| Application processing | 50 ms |
| Waiting for a database connection | 200 ms |
| Database query execution | 550 ms |
| Response serialization | 50 ms |
| Total | 900 ms |

Database-related time accounts for 750 ms, including connection acquisition.

Reducing application processing from 50 ms to 10 ms saves 40 ms.

Reducing database-related time from 750 ms to 250 ms saves 500 ms.

The database optimization has a much larger potential impact on this request.

### Important distinction

The numbers above describe one illustrative timing breakdown. In a real distributed system, stages can overlap, and network timings may be measured differently.

For parallel operations, total request latency is not necessarily the sum of every operation's duration.

For example, if two independent services are called in parallel, the total waiting time may be closer to the duration of the slower call rather than the sum of both calls.

### How to investigate

Measure or trace:

- Time waiting for a connection.
- Database query duration.
- Application processing time.
- External dependency duration.
- Network and serialization time.

Avoid treating all unaccounted time as application processing. Instrumentation gaps can hide the real bottleneck.

---

## 5. The Difference Between Logs, Metrics, and Traces

These observability tools answer different questions.

### Logs

Logs record individual events.

Examples:

- Booking creation failed.
- Database query timed out.
- External provider returned an error.
- Cache invalidation failed.

Logs help explain what happened and provide contextual details.

### Metrics

Metrics measure behavior over time.

Examples:

- API request rate.
- P95 and P99 latency.
- Error rate.
- CPU and memory utilization.
- Database query duration.
- Connection-pool usage.
- Queue depth.

Metrics help detect trends and identify when a problem started.

### Distributed traces

Traces show the path of a request across multiple components.

For example:

```text
Booking API:          900 ms
  |
  +-- Authentication:  10 ms
  |
  +-- Database wait:  200 ms
  |
  +-- SQL query:      550 ms
  |
  +-- Serialization:  50 ms
  |
  +-- Other overhead:  90 ms
```

The example is illustrative. Real traces should show measured spans and their relationships.

Traces are especially useful when an API depends on multiple services or databases.

### Combined approach

A practical investigation often follows this sequence:

1. Use metrics to detect a latency or error-rate increase.
2. Use traces to identify the slow component.
3. Use logs to understand the specific failure or event.
4. Use profiling or database execution plans to investigate the root cause.
5. Apply a targeted change and compare results.

---

## 6. Performance Budgets

A performance budget defines how much time or resource consumption a component should use while meeting an overall requirement.

Suppose the booking API has an end-to-end P95 latency objective of 300 ms.

An initial engineering budget could be:

| Component | Target allocation |
|---|---:|
| Network and gateway | 40 ms |
| Application processing | 40 ms |
| Database work | 150 ms |
| Serialization | 20 ms |
| Remaining margin | 50 ms |
| Total | 300 ms |

These are illustrative allocations, not a guarantee that component P95 values add up to the end-to-end P95.

Percentiles do not generally add together directly. Correlation between stages, parallel operations, and measurement boundaries affect the total.

### Why performance budgets help

Without budgets, individual components can become slower without clear ownership of the end-to-end impact.

Budgets help teams ask:

- Is database work consuming too much time?
- Are external calls exceeding their allocated time?
- Is the frontend spending too long processing a response?
- Is the overall latency objective still achievable?

### Frontend performance budgets

Examples:

- Maximum JavaScript bundle size.
- Maximum number of network requests for a user action.
- Maximum API latency.
- Maximum rendering time.
- Target time until the page becomes usable.

### Backend performance budgets

Examples:

- Maximum API response time.
- Maximum database execution time.
- Maximum external-service timeout.
- Maximum queueing delay.
- Maximum memory or CPU consumption under expected load.

A performance budget should be measurable and tied to a user or business requirement.

---

## 7. Service-Level Indicators, Objectives, and Agreements

These terms help make performance expectations explicit.

### Service-Level Indicator (SLI)

An SLI is a measurement of actual service behavior.

Examples:

- Percentage of successful API requests.
- P95 latency.
- Percentage of requests completing within 300 ms.
- Availability over a defined period.

### Service-Level Objective (SLO)

An SLO is a target for an SLI.

Example:

- 99% of eligible API requests complete within 300 ms over a defined rolling window.

This is a latency objective, not a guarantee that every request finishes within 300 ms.

### Service-Level Agreement (SLA)

An SLA is a formal agreement with users or customers that may define service commitments and remedies if they are not met.

Not every internal SLO is an SLA.

### Why these matter for system design

Without a defined target, statements such as "the API should be fast" are difficult to evaluate.

An explicit SLO helps determine:

- Required capacity.
- Appropriate timeouts.
- Acceptable error rates.
- Scaling thresholds.
- Whether an optimization is worthwhile.

---

## 8. Amdahl's Law: Why Some Optimizations Have Limited Impact

Amdahl's Law describes the maximum improvement possible when only part of a workload is optimized.

Suppose an API spends:

- 80% of its time on database work.
- 20% on everything else.

If database processing becomes twice as fast, the total speedup is:

```text
Speedup = 1 / ((1 - P) + (P / S))
```

Where:

- P = Fraction of work being optimized.
- S = Speedup of that portion.

In this example:

```text
P = 0.8
S = 2

Speedup = 1 / ((1 - 0.8) + (0.8 / 2))
        = 1 / (0.2 + 0.4)
        = 1.67
```

The total workload becomes approximately 1.67 times as fast, rather than twice as fast.

If the original response time were 1,000 ms and the same proportions applied, the new response time would be approximately 600 ms.

### What this teaches us

Optimizing the largest component often produces the biggest improvement, but the total benefit depends on how much of the complete request it represents.

After optimizing a bottleneck, another component may become the next bottleneck.

---

## 9. Optimization Strategy 1: Scale Application Servers

Scaling is appropriate when application processing is the limiting resource and the work can be distributed across additional instances.

### Example

Suppose:

- One server sustains 200 RPS within the required latency target.
- Peak traffic is 500 RPS.
- The database has sufficient capacity for the additional load.

Three servers provide a nominal combined capacity of 600 RPS.

However, if the system must sustain 500 RPS after one server fails, three servers are insufficient:

```text
Three servers normally:
3 * 200 = 600 RPS

After one server fails:
2 * 200 = 400 RPS
```

Four servers provide:

```text
Four servers normally:
4 * 200 = 800 RPS

After one server fails:
3 * 200 = 600 RPS
```

These calculations assume independent capacity and no shared bottleneck.

### Advantages

- More application processing capacity.
- Potentially better resilience to instance failures.
- Ability to scale based on demand.

### Trade-offs

- Higher infrastructure cost.
- More instances to manage.
- More concurrent requests to shared dependencies.
- Potentially greater database load.

**Do not scale the application tier without checking the downstream capacity it depends on.**

---

## 10. Optimization Strategy 2: Optimize Database Queries

Database optimization is appropriate when database execution, connection waits, or contention dominate request latency.

### Common causes

- Missing or ineffective indexes.
- Queries scanning excessive rows.
- Inefficient joins.
- N+1 query patterns.
- Unnecessary repeated queries.
- Large result sets.
- Lock contention.
- Excessive database round trips.
- Poor connection-pool configuration.

### Example: N+1 query problem

Suppose an endpoint retrieves 100 bookings and then makes one additional query for each booking's passenger details.

The query count might be:

```text
1 query to fetch bookings
100 queries to fetch passenger details

Total = 101 queries
```

An appropriate join or batched query may reduce the number of database round trips substantially.

However, fewer queries are not automatically faster. Query plans, returned data volume, indexing, and the database workload still matter.

### Advantages

- Reduces database processing.
- May reduce connection occupancy.
- Can improve throughput and latency together.

### Trade-offs

- Query changes require validation.
- Additional indexes consume storage and add write overhead.
- Complex query rewrites may increase maintenance cost.
- An optimization for one workload may hurt another.

Measure before and after using representative data and load.

---

## 11. Optimization Strategy 3: Introduce Caching

Caching is appropriate when repeated reads can reuse results without violating freshness requirements.

Example architecture:

```text
Client
  |
  v
Application API
  |
  v
Check cache
  |
  +---- Cache hit ----> Return cached data
  |
  +---- Cache miss ---> Query database
                           |
                           v
                       Store result
                           |
                           v
                       Return result
```

A shared cache such as Redis may reduce repeated database reads.

### Advantages

- Reduces repeated database work.
- Can lower latency for cache hits.
- Can increase effective read capacity.

### Trade-offs

- Stale data.
- Cache invalidation complexity.
- Cache misses.
- Cache stampedes.
- Memory cost.
- Additional infrastructure and operational work.

### Cache hit rate

```text
Cache Hit Rate =
Cache Hits / (Cache Hits + Cache Misses) * 100%
```

Suppose a cache receives 1,000 lookups:

- 800 hits.
- 200 misses.

Then:

```text
Hit rate = 800 / 1,000 * 100
         = 80%
```

This measures lookup frequency, not necessarily database workload reduction. Some misses may be much more expensive than others, and a single cache lookup may not correspond to a single database query.

### When caching may not help

Caching may be ineffective when:

- Requests mostly access unique data.
- Data changes frequently.
- Cache hit rate is low.
- Cache lookup overhead is significant.
- The actual bottleneck is unrelated to database reads.

---

## 12. Optimization Strategy 4: Reduce Unnecessary Work

Sometimes the best optimization is to avoid doing work at all.

Examples:

- Eliminate duplicate API calls.
- Batch repeated database queries.
- Fetch only the required fields.
- Avoid unnecessary data transformations.
- Deduplicate repeated event handling.
- Reduce oversized API payloads.
- Debounce or coalesce updates when the product requirements permit it.

### Example: Duplicate requests

Suppose a frontend triggers five identical booking requests for one user action.

If the backend receives 500 user actions per second, this pattern can generate:

```text
500 actions/second * 5 requests/action
= 2,500 API requests/second
```

If identical requests can safely be deduplicated or combined, the system may process much less unnecessary work.

However, deduplication must account for request identity, authorization, response semantics, and data freshness.

### Advantages

- Reduces demand across multiple layers.
- Can improve latency without additional infrastructure.
- Often reduces cost as well as resource consumption.

### Trade-offs

- More complex request coordination.
- Potentially more complicated state management.
- Batching may add waiting time before a request can be sent.
- Deduplication can be incorrect if requests that look similar have different semantics.

---

## 13. Optimization Strategy 5: Control Concurrency

Concurrency limits restrict how many operations can be active simultaneously.

Suppose an application allows 2,000 concurrent database operations, but the database performs best with a much smaller number of active queries.

Unbounded concurrency can increase:
- Database contention.
- Memory usage.
- Connection waits.
- Queueing.
- Tail latency.

A concurrency limit can protect the database by bounding the number of active operations.

### Trade-off

A lower concurrency limit can reduce contention but may also reduce throughput if the limit is too restrictive.

The goal is not to minimize concurrency. It is to find a level that supports the required throughput while meeting latency and reliability objectives.

---

## 14. How to Choose the Right Optimization

| Observed symptom | Initial investigation | Potential intervention |
|---|---|---|
| High application CPU | CPU profiling and request cost | Optimize expensive processing or scale application instances |
| High database CPU | Query plans and query volume | Optimize queries, indexes, or data access |
| Low CPU but high latency | Traces and dependency waits | Investigate network, database connections, locks, or external services |
| Repeated identical reads | Cache hit potential and freshness needs | Caching or request deduplication |
| Queue depth increasing | Incoming versus processing rate | Increase worker capacity or reduce job cost |
| High memory usage | Heap and allocation profiling | Reduce retained objects, payloads, or concurrency |
| High external-service latency | Dependency timing and error rates | Appropriate timeouts, bounded retries, caching, or fallback |
| Too many database round trips | Query traces and request patterns | Batching or reducing repeated queries |

These are starting hypotheses, not automatic prescriptions.

The same symptom can have multiple causes. Measure the actual workload before choosing a solution.

---

## 15. The Optimization Feedback Loop

A disciplined optimization process follows a loop:

```text
Define performance objective
          |
          v
Measure current behavior
          |
          v
Identify the limiting component
          |
          v
Form a hypothesis
          |
          v
Apply a targeted change
          |
          v
Repeat the same workload test
          |
          v
Compare latency, throughput,
errors, and resource usage
          |
          v
Keep, refine, or revert the change
```

### Example

Objective:
P99 booking API latency must remain below 300 ms at 500 RPS.

Baseline:
- P99 latency = 700 ms.
- Database CPU = 95%.
- Application CPU = 40%.

Hypothesis:
Expensive database queries are the main bottleneck.

Intervention:
Optimize the identified query and validate the execution plan.

Retest:
Measure P99 latency, database CPU, successful throughput, and error rate at the same load.

The intervention is successful only if the results improve against the defined objectives without creating unacceptable regressions elsewhere.

---

## 16. Common Mistakes

### Mistake 1: Optimizing the easiest component instead of the bottleneck

Reducing 20 ms of frontend work may not solve an API that spends 700 ms waiting for PostgreSQL.

### Mistake 2: Scaling without measurement

More servers may add cost and increase pressure on shared dependencies.

### Mistake 3: Using average latency alone

A system may have acceptable median latency but unacceptable P99 latency.

### Mistake 4: Treating performance budgets as guaranteed sums

Component percentiles do not generally add directly to end-to-end percentiles.

### Mistake 5: Introducing caching before understanding data access

Caching unique or frequently changing data may offer little benefit and add complexity.

### Mistake 6: Ignoring concurrency

Many requests waiting for a slow dependency can exhaust resources even when CPU is low.

### Mistake 7: Changing multiple things simultaneously

If several architectural changes are applied at once, it becomes difficult to determine which change produced the result.

### Mistake 8: Ignoring regressions

A change may improve average latency while increasing errors or harming tail latency.

### Mistake 9: Testing only with ideal traffic

Realistic tests should include representative payloads, request distributions, database activity, and concurrency.

---

## 17. Practical Performance Investigation Checklist

When a system is slow:

1. Define the required latency, throughput, and error-rate targets.
2. Identify which users, endpoints, or operations are affected.
3. Measure P50, P95, and P99 latency.
4. Compare incoming traffic with successful throughput.
5. Check application CPU, memory, and concurrency.
6. Inspect database query time, connection waits, and lock contention.
7. Examine external dependency latency.
8. Review network and payload sizes.
9. Inspect traces to determine where time is spent.
10. Identify the most likely bottleneck.
11. Choose one targeted intervention.
12. Retest under comparable conditions.
13. Verify that the change meets the objective.
14. Check for new bottlenecks and regressions.
15. Document the results and assumptions.

---

## 18. Worked Example: Booking API Under Load

### Requirements

- Peak traffic: 500 RPS.
- P99 latency target: below 300 ms.
- Error rate target: below 1%.

### Initial measurements

| Metric | Observed value |
|---|---:|
| Incoming traffic | 500 RPS |
| Successful throughput | 490 RPS |
| P99 latency | 800 ms |
| Application CPU | 50% |
| Database CPU | 95% |

### Analysis

The database is a strong bottleneck hypothesis because CPU is high and query latency is elevated.

The application server CPU is not particularly high, so immediately adding application instances may increase database pressure.

### Investigation

- Inspect slow queries and query plans.
- Measure query volume per API request.
- Examine connection-pool waits and lock contention.
- Identify repeated or unnecessary reads.
- Determine whether caching is appropriate for frequently read data.

### Intervention

Suppose query optimization reduces database work substantially.

Retest the same workload and evaluate:
- P99 latency.
- Successful throughput.
- Error rate.
- Database CPU.
- Connection waits.

Do not assume the optimization succeeded simply because one metric improved. Verify the full set of service objectives.

### Next bottleneck

After database optimization, the database might no longer be the limiting component. Application CPU, network bandwidth, or another dependency could become the next bottleneck.

System optimization is iterative.

---

## 19. Revision Questions

### Q1. What is a bottleneck?

Answer:
A component or resource that limits system performance, throughput, or capacity.

### Q2. Why is measuring the complete request lifecycle important?

Answer:
Because latency can be caused by frontend processing, networking, application logic, database waits, external services, or other stages.

### Q3. What is the difference between logs, metrics, and traces?

Answer:
Logs record events, metrics quantify behavior over time, and traces show the path and timing of operations across components.

### Q4. Why should performance budgets be measurable?

Answer:
Measurable budgets allow teams to identify when components consume too much time or capacity relative to the overall objective.

### Q5. What does Amdahl's Law explain?

Answer:
The total speedup from an optimization is limited by the fraction of work that was not optimized.

### Q6. When should application servers be scaled?

Answer:
When application processing is the limiting resource and additional instances can add useful capacity without overwhelming dependencies.

### Q7. When is caching useful?

Answer:
When frequently accessed data can be reused safely and the reduction in downstream work justifies the complexity and freshness trade-offs.

### Q8. Why might a database query optimization be better than adding servers?

Answer:
It can reduce the actual work required, lowering database pressure and potentially improving latency and throughput across the entire application.

### Q9. What is a performance budget?

Answer:
A measurable allocation of time or resource consumption intended to help a system meet its overall performance objective.

### Q10. Why should one change be tested at a time where practical?

Answer:
It helps attribute observed improvements or regressions to a specific intervention.

---

## 20. Final Summary

Remember these principles:

- A bottleneck limits the performance or capacity of the system.
- Measure the full request lifecycle before optimizing.
- Logs, metrics, and traces serve different but complementary purposes.
- Performance budgets connect component behavior to user-facing requirements.
- SLOs make performance and reliability targets explicit.
- Amdahl's Law explains why partial optimizations have limited total impact.
- Scaling is useful when the limiting workload can be distributed.
- Database optimization is often more effective than scaling the application tier when database work dominates.
- Caching can reduce repeated work but introduces consistency and operational trade-offs.
- Reducing unnecessary work can improve performance across multiple layers.
- Concurrency limits can protect constrained resources.
- Always validate improvements using representative workloads and the same performance objectives.

**The central lesson:** Do not optimize based on assumptions or isolated metrics. Identify the bottleneck, select the smallest effective intervention, and measure whether it improves the system as a whole.
