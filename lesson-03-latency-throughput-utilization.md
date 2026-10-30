# Lesson 3: Latency, Throughput, and Utilization

## 1. Learning Objectives

By the end of this lesson, you should be able to:

- Explain latency, throughput, and utilization.
- Distinguish between average latency and percentile latency.
- Understand the difference between incoming request rate and successful throughput.
- Calculate resource utilization and remaining capacity.
- Explain how high utilization affects latency.
- Understand queueing, concurrency, and bottlenecks.
- Reason about system performance under increasing traffic.
- Use performance targets to determine sustainable capacity.
- Identify which metrics to investigate when a system becomes slow.

---

## 2. The Three Fundamental Concepts

Three metrics help describe how a system behaves under load.

### Latency

How long does one operation take?

Example:
A booking API receives a request and returns the response in 150 milliseconds.

### Throughput

How much work does the system complete per unit of time?

Example:
The booking API successfully processes 500 requests per second.

### Utilization

How much of a resource's available capacity is being used?

Example:
An application server is using 65% of its available CPU capacity.

These metrics are related, but they are not interchangeable.

A system might have:
- Low CPU utilization but high latency because it is waiting on a database.
- High throughput but unacceptable latency.
- Low latency at low traffic but poor performance at peak traffic.
- High CPU utilization while throughput remains flat because the system has reached a bottleneck.

**Key principle:** Evaluate latency, throughput, utilization, and errors together rather than relying on one metric.

---

## 3. Latency

Latency measures the time taken for an operation to complete from a defined starting point to a defined endpoint.

For an API, it might mean the time between the server receiving a request and finishing its response. For an end-to-end user experience, it may include network travel and frontend processing too.

Latency is commonly measured in milliseconds (ms) or seconds.

### Example

A client requests booking details.

```text
Request sent
    |
    v
API receives request
    |
    v
Authentication and validation
    |
    v
Database query
    |
    v
Response serialization
    |
    v
Response received by client
```

Suppose the measured time from sending the request to receiving the response is 250 ms.

The end-to-end request latency is 250 ms.

Be precise about what the measurement includes. Server-side latency and client-observed latency are not necessarily the same.

### Why latency matters

High latency can cause:

- Slow user experiences.
- Requests timing out.
- Longer-running concurrent requests.
- More resources being occupied while requests wait.
- Cascading failures when dependencies become overloaded.

A request that takes two seconds may not sound expensive by itself. But if many requests remain in progress simultaneously, the system can consume substantial memory, connections, and worker capacity.

---

## 4. Average Latency Is Not Enough

Consider a service that handles 1,000 requests.

Suppose 990 requests complete in 100 ms, while 10 requests take 2 seconds.

Most users may experience fast responses, but some users experience significant delays.

The average alone can hide the distribution of request times.

### Percentile latency

Percentiles show how response times are distributed.

#### P50 — Median latency

50% of requests complete at or below this latency.

If P50 is 100 ms, half the measured requests took 100 ms or less.

#### P95 — 95th-percentile latency

95% of requests complete at or below this latency. The slowest 5% take longer.

#### P99 — 99th-percentile latency

99% of requests complete at or below this latency. The slowest 1% take longer.

### Example

| Metric | Latency |
|---|---:|
| P50 | 100 ms |
| P95 | 250 ms |
| P99 | 1,500 ms |

Interpretation:

- Typical requests are relatively fast.
- A small percentage of requests take much longer.
- Investigating only the median would miss the slow-request problem.

### Why tail latency matters

A slow request might be caused by:

- A database query with an inefficient execution plan.
- Lock contention.
- Waiting for a connection.
- A slow external API.
- Garbage collection or resource contention.
- Network delays.
- A cache miss.
- A large or unusually expensive request.

Tail latency is particularly important in systems that make several downstream calls for one user request.

### Important percentile detail

Percentiles are calculated from a population of measurements over a defined period. They are not guarantees that every request will finish within the percentile value.

For example, a P99 target of 300 ms allows approximately 1% of measured requests to exceed 300 ms, assuming the metric and measurement window are defined consistently.

---

## 5. Latency Budgets

A latency budget allocates a total response-time target across different parts of the system.

Suppose the booking API has a target of 300 ms.

An initial budget might be:

| Component | Target allocation |
|---|---:|
| Network and gateway overhead | 40 ms |
| Application processing | 40 ms |
| Database operations | 150 ms |
| Response serialization | 20 ms |
| Remaining margin | 50 ms |
| Total | 300 ms |

The margin accounts for variability and work that is difficult to allocate precisely.

These are illustrative engineering allocations, not a mathematical guarantee that the end-to-end P95 equals the sum of component P95 values.

### Why budgets help

Without budgets, each component can become slower without anyone recognizing the impact on the complete request.

With budgets, teams can ask:

- Is the database consuming too much of its allocated time?
- Are network calls taking longer than expected?
- Is application processing expensive?
- Is the total latency target still achievable?

For frontend systems, a similar budget can account for network requests, JavaScript execution, rendering, and time until the user can interact with the page.

---

## 6. Throughput

Throughput measures the amount of work a system successfully completes over a period of time.

For web APIs, a common unit is requests per second (RPS).

Other examples include:
- Database transactions per second.
- Messages processed per second.
- Files uploaded per minute.
- Orders completed per hour.

### Incoming request rate versus successful throughput

These are different measurements.

Suppose an API receives 800 requests per second.

Of those requests:
- 600 complete successfully.
- 100 fail.
- 100 remain in progress or time out during the measurement interval.

The incoming rate is 800 RPS, but successful throughput is only 600 successful requests per second.

The exact accounting depends on the measurement window and how requests are classified.

### Why this distinction matters

A service may appear to handle a high request rate while many requests are failing.

Capacity planning should therefore examine:
- Incoming request rate.
- Successful completion rate.
- Error rate.
- Timeouts.
- Requests in progress.
- Queue depth and queueing delay.

### Example

Suppose a worker processes 100 messages per second successfully.

If 20 messages per second fail and are retried, the worker's total processing attempts may be higher than 100 per second.

Distinguish useful completed work from total processing attempts.

---

## 7. Maximum Throughput and Sustainable Capacity

A system's maximum observed throughput is not necessarily its sustainable capacity.

Suppose load testing produces these results:

| Incoming load | P99 latency | Error rate |
|---:|---:|---:|
| 300 RPS | 100 ms | 0.1% |
| 400 RPS | 150 ms | 0.1% |
| 500 RPS | 250 ms | 0.3% |
| 600 RPS | 600 ms | 1.5% |
| 700 RPS | 1,500 ms | 5% |

Assume the service requirements are:
- P99 latency below 300 ms.
- Error rate below 1%.

Under these test results, 500 RPS is the highest tested load that meets both requirements.

It is not necessarily the absolute maximum. Additional tests at intermediate loads might reveal a higher valid threshold.

### Sustainable capacity

Sustainable capacity is the workload the system can maintain while meeting its performance and reliability requirements.

The capacity target should also account for:
- Traffic variability.
- Failures.
- Workload mix.
- Resource headroom.
- Changes in dependencies.
- Expected traffic growth.

**Design principle:** Do not optimize for the largest RPS number if users experience unacceptable latency or failures at that load.

---

## 8. Utilization

Utilization measures how much of a resource's available capacity is being used.

### Formula

```text
Utilization (%) = Actual Load / Available Capacity * 100
```

For a CPU, utilization may represent the fraction of available CPU time being used.

For an application fleet, capacity utilization can represent the current workload relative to its sustainable throughput.

These measurements describe different resources and should not be confused.

### Example: Application server utilization

Suppose:
- Sustainable capacity per server = 200 RPS.
- Actual load = 120 RPS.

Then:

```text
Utilization = 120 / 200 * 100
            = 60%
```

The server is operating at 60% of its assumed sustainable request capacity.

This does not necessarily mean its CPU utilization is 60%. The server might be constrained by database waits, network operations, or another resource.

### Example: Fleet utilization

Suppose three healthy servers can each sustain 200 RPS.

Total capacity:

```text
3 * 200 = 600 RPS
```

If the fleet receives 500 RPS:

```text
Fleet utilization = 500 / 600 * 100
                   = 83.3%
```

Remaining nominal capacity:

```text
600 - 500 = 100 RPS
```

The fleet has 100 RPS of nominal headroom, assuming traffic is distributed effectively and the capacity assumptions hold.

---

## 9. Headroom

Headroom is the capacity available above the current workload.

```text
Headroom = Available Capacity - Current Load
```

Suppose:
- Available capacity = 1,000 RPS.
- Current load = 700 RPS.

Then:

```text
Headroom = 1,000 - 700
         = 300 RPS
```

The fleet has 300 RPS of nominal headroom.

### Why headroom matters

Headroom helps absorb:
- Sudden traffic increases.
- Unexpected expensive requests.
- Uneven load distribution.
- Temporary performance degradation.
- Some component failures.

There is no universally correct headroom percentage. A system with predictable traffic and fast scaling may need a different margin from a system with sudden spikes or strict availability requirements.

Headroom should be based on workload variability, recovery time, failure tolerance, and service objectives.

---

## 10. Why High Utilization Can Increase Latency

A common mistake is to assume that a system performs equally well at 30% and 95% utilization.

As utilization approaches saturation, requests may need to wait for resources.

For example:

1. Requests arrive at the application server.
2. The server starts database queries.
3. The database becomes busy.
4. Queries wait for CPU, I/O, locks, or connections.
5. More application requests remain in progress.
6. Connection pools and other resources become occupied.
7. Latency increases, and timeouts may begin.

This is a queueing problem.

### Queueing intuition

A service has a finite rate at which it can complete work. If requests arrive faster than they can be completed, the queue grows.

If arrival rate remains above processing capacity, the backlog will continue increasing.

Even when the long-term arrival rate is below capacity, bursts and variable processing times can create queues.

### Why latency can rise sharply

Near saturation, there is less spare capacity to absorb variability. A small increase in incoming traffic can cause a disproportionately large increase in queueing delay.

The exact relationship depends on the workload and queueing behavior. It is not a universal rule that latency suddenly increases at a fixed utilization percentage.

---

## 11. Little's Law: Connecting Concurrency, Throughput, and Latency

Little's Law is a fundamental relationship in stable systems.

```text
L = λ * W
```

Where:

- L = Average number of items in the system.
- λ = Average throughput or arrival rate, when the system is stable and the rates are appropriately defined.
- W = Average time spent in the system.

For an API:

```text
Average requests in progress
= Throughput * Average response time
```

### Example

Suppose an API successfully processes 200 requests per second, with average latency of 0.25 seconds.

```text
Average requests in progress = 200 * 0.25
                             = 50 requests
```

The system has approximately 50 requests in progress on average.

Now suppose latency increases to 1 second while throughput remains 200 RPS.

```text
Average requests in progress = 200 * 1
                             = 200 requests
```

The average number of requests in progress increases to 200.

Those requests can consume memory, connections, and other resources.

This explains why slower dependencies can exhaust application resources even when the incoming traffic rate has not changed.

Little's Law is an average relationship, not a guarantee about every individual request. It applies under stable conditions with consistent definitions of throughput and time in the system.

---

## 12. Concurrency Is Not the Same as Throughput

Concurrency is the number of operations in progress at the same time.

Throughput is the rate at which operations complete.

Latency is how long an operation takes.

These concepts are related but different.

### Example

System A:
- 50 requests in progress on average.
- Average latency = 250 ms.
- Throughput = 200 RPS.

System B:
- 200 requests in progress on average.
- Average latency = 1 second.
- Throughput = 200 RPS.

Both systems have the same throughput, but System B has four times as many requests in progress and four times the average latency.

Increasing concurrency does not necessarily increase throughput. If a database is saturated, allowing more concurrent queries may simply create more waiting.

Concurrency limits can help protect a service by preventing too many requests from consuming resources simultaneously.

---

## 13. Identify the Actual Bottleneck

Suppose a booking API has:
- Incoming traffic: 400 RPS.
- Application CPU utilization: 45%.
- Database CPU utilization: 95%.
- P99 latency: 2 seconds.

The database is a strong initial bottleneck hypothesis.

Possible causes include:
- Expensive queries.
- Missing or ineffective indexes.
- Excessive query volume.
- Connection-pool waits.
- Lock contention.
- High database I/O.

The next step is to investigate query latency, execution plans, connection waits, locks, and request traces.

Adding application servers immediately may make the problem worse by increasing concurrent database work.

### Another example: Low CPU, high latency

Suppose application CPU is 30%, but API latency is 3 seconds.

Possible explanations:
- Slow external service.
- Database connections are exhausted.
- The application is waiting on network I/O.
- Requests are blocked by locks.
- A queue or worker pool is saturated.

Low CPU does not prove that the system has spare end-to-end capacity.

### A useful investigation sequence

1. Confirm the latency and error-rate pattern.
2. Determine whether the issue affects all requests or specific endpoints.
3. Inspect traces and timing breakdowns.
4. Check application, database, network, and dependency metrics.
5. Identify the resource or dependency causing the delay.
6. Test a targeted change.
7. Compare performance under the same workload.

---

## 14. Latency, Throughput, and Utilization Across the Stack

### Frontend

Relevant measurements include:
- Time until the page becomes usable.
- API request latency.
- Rendering time.
- JavaScript execution time.
- Number of requests per user action.
- Payload sizes.

A backend API can be fast while the frontend remains slow because it performs excessive requests or expensive rendering.

### Backend

Relevant measurements include:
- Request latency and percentiles.
- Successful throughput.
- Error and timeout rates.
- Requests in progress.
- CPU and memory utilization.
- Queueing and connection waits.

### Database

Relevant measurements include:
- Query latency.
- Query volume.
- CPU and I/O utilization.
- Connection-pool usage.
- Lock waits.
- Slow queries.

### Message queues

Relevant measurements include:
- Incoming message rate.
- Successful processing rate.
- Queue depth.
- Age of the oldest message.
- Retry rate.
- Dead-letter queue size.

The correct metrics depend on the role of each component.

---

## 15. Common Mistakes

### Mistake 1: Looking only at average latency

The average can hide slow requests. Inspect P50, P95, and P99 where appropriate.

### Mistake 2: Treating incoming RPS as successful throughput

A system can receive a high request rate while returning errors or timing out.

### Mistake 3: Assuming low CPU means the system is healthy

The bottleneck may be a database, network, connection pool, lock, or external service.

### Mistake 4: Assuming higher concurrency always improves throughput

More concurrent work can overload a saturated dependency.

### Mistake 5: Scaling based on CPU alone

CPU is only one resource. Use workload-specific measurements and service-level objectives.

### Mistake 6: Treating theoretical capacity as sustainable capacity

Use capacity measurements that meet latency and error-rate requirements.

### Mistake 7: Ignoring traffic variability

A service that performs well under steady traffic may behave differently during bursts.

### Mistake 8: Optimizing without a baseline

Measure before and after changes under comparable conditions.

---

## 16. Practical Performance Analysis Checklist

When investigating a slow system, ask:

1. What is the incoming request rate?
2. What is the successful throughput?
3. What are P50, P95, and P99 latency?
4. What is the error and timeout rate?
5. How many requests are in progress?
6. What is CPU and memory utilization?
7. Are requests waiting for database connections?
8. Are database queries slow or expensive?
9. Are external services contributing to latency?
10. Is queue depth increasing?
11. Is the workload distributed evenly?
12. What throughput still meets the latency and error-rate targets?
13. Is there sufficient headroom for spikes and failures?
14. Have the proposed improvements been validated under realistic load?

---

## 17. Worked Example

### Scenario

A booking API has these measurements:

- Incoming traffic = 500 RPS.
- Successful throughput = 490 RPS.
- P99 latency = 800 ms.
- Application CPU = 50%.
- Database CPU = 95%.
- Database query latency is elevated.

### Interpretation

The system receives 500 RPS but successfully completes only 490 RPS during the measurement interval.

P99 latency is high, indicating that at least a portion of requests are slow.

Application CPU is only 50%, while database CPU is 95% and query latency is elevated. This suggests that the database may be the limiting component.

### Reasonable next steps

1. Investigate expensive database queries and execution plans.
2. Measure query volume per API request.
3. Inspect connection waits and lock contention.
4. Identify unnecessary repeated queries.
5. Optimize the database access pattern.
6. Evaluate caching only where data freshness permits.
7. Retest under the same traffic and workload mix.

### What not to assume

Do not conclude that adding application servers will automatically fix the problem. More application instances could increase pressure on PostgreSQL.

---

## 18. Revision Questions

### Q1. What is latency?

Answer:
The time taken for an operation to complete over a defined measurement interval.

### Q2. What is throughput?

Answer:
The amount of work successfully completed per unit of time.

### Q3. What is utilization?

Answer:
The proportion of a resource's available capacity being used.

### Q4. Why are P95 and P99 useful?

Answer:
They expose slow-request behavior that an average or median can hide.

### Q5. Can a system have low CPU utilization and high latency?

Answer:
Yes. It may be waiting for a database, network, connection, lock, or external dependency.

### Q6. What is headroom?

Answer:
The capacity available above the current workload.

### Q7. What happens when requests arrive faster than they can be processed?

Answer:
The queue grows. If the imbalance continues, latency and resource consumption increase, and failures may follow.

### Q8. What does Little's Law describe?

Answer:
The relationship between average items in a system, throughput, and average time spent in the system: L = λ * W.

### Q9. Does increasing concurrency always improve throughput?

Answer:
No. It can increase contention and overload a bottleneck.

### Q10. How should sustainable capacity be selected?

Answer:
Choose the throughput that meets latency and error-rate requirements, with appropriate headroom for variability and failures.

---

## 19. Final Summary

Remember these principles:

- Latency measures time; throughput measures completed work; utilization measures resource usage.
- Percentiles reveal slow requests that averages may hide.
- Incoming request rate and successful throughput are different.
- High utilization can increase queueing and latency.
- Concurrency, latency, and throughput are related but distinct.
- Low application CPU does not guarantee that the system has spare capacity.
- A slow dependency can keep many requests in progress and exhaust resources.
- Little's Law helps connect throughput, average latency, and average concurrency.
- Sustainable capacity must satisfy performance and reliability requirements.
- Measure the actual bottleneck before optimizing.
- Maintain headroom for traffic variation and failures.
- Validate performance improvements under representative workloads.

**The central lesson:** A healthy system is not simply one that processes many requests. It is one that completes useful work at the required throughput, within acceptable latency and error-rate targets, without exhausting its resources.
