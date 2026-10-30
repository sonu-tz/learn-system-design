# Lesson 2: Scalability, Load Balancing, Statelessness, and Capacity Planning

## 1. Learning Objectives

By the end of this lesson, you should be able to:

- Explain vertical and horizontal scaling.
- Understand the role of a load balancer.
- Explain why stateless application servers are easier to scale.
- Estimate average and peak requests per second (RPS).
- Estimate application server capacity.
- Plan for server failures without exceeding capacity.
- Understand utilization, headroom, and redundancy.
- Identify bottlenecks that prevent linear scaling.
- Make architecture decisions based on requirements rather than assumptions.

---

## 2. What Is Scalability?

Scalability is a system's ability to handle increasing workloads while maintaining acceptable performance, reliability, and cost.

Workload can increase in several ways:

- More users.
- More requests per second.
- More data stored.
- More concurrent connections.
- More computationally expensive operations.
- More background jobs or real-time events.

For example, a booking platform might initially handle 100 requests per second and eventually need to handle 2,000 requests per second.

The design question is:

**How can we increase the system's capacity without unacceptable latency, failures, or cost?**

Scalability is not the same as performance.

- **Performance:** How efficiently does the system handle its current workload?
- **Scalability:** How does the system behave as its workload increases?

A system can be fast at 100 RPS but perform poorly at 1,000 RPS.

---

## 3. Vertical Scaling

Vertical scaling means increasing the resources available to an existing machine.

Examples:

- Increasing CPU cores.
- Adding memory.
- Using faster storage.
- Upgrading to a more powerful database instance.

### Architecture

```text
             Incoming Traffic
                    |
                    v
          +-------------------+
          | Application Server|
          |                   |
          |  More CPU / RAM   |
          +-------------------+
                    |
                    v
                Database
```

### Advantages

- Simple architecture.
- Usually fewer application instances to manage.
- Less distributed coordination between application instances.
- Can be an effective first step when a machine is under-provisioned.

### Disadvantages

- A machine has a practical maximum capacity.
- Larger machines can become disproportionately expensive.
- The machine may remain a single point of failure.
- Upgrades may require downtime or a restart, depending on the platform.
- A more powerful application server does not necessarily fix a database or network bottleneck.

### When to use vertical scaling

Consider it when:

- The current machine is resource-constrained.
- Traffic is moderate.
- Operational simplicity is important.
- A larger instance offers sufficient capacity for the foreseeable workload.

Vertical scaling is not inherently a bad design. The right choice depends on the workload, cost, reliability target, and available infrastructure.

---

## 4. Horizontal Scaling

Horizontal scaling means adding more machines or application instances and distributing work across them.

### Architecture

```text
                    Clients
                       |
                       v
               +---------------+
               | Load Balancer |
               +---------------+
                  /     |     \
                 /      |      \
                v       v       v
             +------+ +------+ +------+
             | App  | | App  | | App  |
             |  A   | |  B   | |  C   |
             +------+ +------+ +------+
                 \      |      /
                  \     |     /
                       v
                +--------------+
                |  PostgreSQL  |
                +--------------+
```

### Advantages

- Capacity can be increased by adding instances.
- Individual application instances can be replaced when they fail.
- Work can be distributed across multiple machines.
- It can support high availability when combined with appropriate redundancy.

### Disadvantages

- Requires traffic distribution and instance health management.
- Shared state must be handled correctly.
- More instances may increase pressure on shared dependencies.
- Networking, monitoring, deployment, and troubleshooting become more complex.
- Capacity does not always increase linearly.

### When to use horizontal scaling

Consider it when:

- Traffic is growing beyond the capacity of individual instances.
- The workload can be distributed across independent instances.
- High availability requires surviving an instance failure.
- The infrastructure supports adding and removing instances as demand changes.

---

## 5. Horizontal Scaling Is Not Always Linear

Suppose one application server sustains 200 RPS while meeting the required latency and error-rate targets.

A naive estimate suggests:

| Servers | Theoretical Capacity |
|---:|---:|
| 1 | 200 RPS |
| 2 | 400 RPS |
| 3 | 600 RPS |
| 4 | 800 RPS |
| 5 | 1,000 RPS |

These numbers assume each server can sustain 200 RPS independently and that no other component becomes a bottleneck.

Real systems may not behave this way.

For example, adding application servers might increase database connections and query concurrency. If the database can sustain only 600 RPS worth of the relevant workload, adding ten more application servers will not make the database process 2,000 RPS.

Other shared bottlenecks include:

- Database CPU, locks, and connection limits.
- Shared caches.
- External APIs.
- Network bandwidth.
- Message queues.
- Storage throughput.
- Centralized session stores.

**Key principle:** Scaling one component helps only while that component is the limiting factor or while its additional capacity can be used by the rest of the system.

---

## 6. What Is a Load Balancer?

A load balancer distributes incoming requests across a group of application instances.

It can help with:

- Traffic distribution.
- Routing requests to healthy instances.
- Removing unhealthy instances from normal traffic.
- Supporting rolling deployments.
- Providing a stable entry point for clients.

A load balancer does not create processing capacity. It distributes the work among the capacity available.

### Common load-balancing strategies

#### A. Round Robin

Requests are assigned to instances in rotation.

Example:

```text
Request 1 -> Server A
Request 2 -> Server B
Request 3 -> Server C
Request 4 -> Server A
```

Useful when instances have similar capacities and requests have roughly comparable costs.

Limitation: requests may have very different processing costs, so an even request count does not guarantee even resource usage.

#### B. Weighted Round Robin

Instances receive traffic in proportion to configured weights.

Example:

```text
Server A: weight 2
Server B: weight 1

Approximate traffic split:
Server A: 2/3
Server B: 1/3
```

Useful when instances have different capacities.

Limitation: static weights may not reflect changing load or actual request cost.

#### C. Least Connections

The load balancer prefers an instance with fewer active connections.

Useful when requests or connections have varying durations.

Limitation: the number of active connections is not always a reliable measure of CPU load, query cost, or remaining capacity.

#### D. Health-Based Routing

Instances that fail configured health checks can be removed from normal routing.

Health checks should distinguish between an instance that cannot serve requests and a temporary problem with a shared dependency. If every application instance reports unhealthy because PostgreSQL is unavailable, removing all instances may not solve the underlying issue.

### Important design question

A load balancer can route traffic around a failed application instance, but the system must still have enough remaining capacity to handle the workload.

---

## 7. Stateless Application Servers

A stateless application instance does not depend on local state surviving between requests.

Consider this example:

```text
Request 1 -> Server A
Request 2 -> Server B
Request 3 -> Server C
```

The requests can be processed by different servers without losing necessary session or application state.

### What does statelessness mean?

It does not mean the application has no state.

A system may still store state in:

- PostgreSQL.
- Redis.
- A shared session store.
- Signed authentication tokens.
- Other durable or shared services.

The distinction is that a particular application instance does not need to retain unique local state for subsequent requests to succeed.

### Problem with in-memory sessions

Imagine Server A stores a user's session only in its own memory.

```text
User logs in
     |
     v
Server A stores session locally
     |
     v
Next request reaches Server B
     |
     v
Server B cannot find the session
```

Possible solutions include:

1. Store sessions in a shared session store.
2. Use an appropriate token-based authentication design.
3. Use sticky sessions when there is a justified reason and their limitations are acceptable.

### Sticky sessions

Sticky sessions attempt to route a client to the same application instance.

They can reduce the need to move some instance-local session state, but they introduce trade-offs:

- Traffic may become unevenly distributed.
- A failed instance may cause session disruption.
- Scaling and deployments can become more complicated.

Sticky sessions do not eliminate the need for appropriate failure handling.

### Why statelessness helps

Stateless application instances make it easier to:

- Add instances.
- Remove instances.
- Replace failed instances.
- Perform rolling deployments.
- Distribute requests across the fleet.

---

## 8. Capacity Planning: Estimate the Workload

Before deciding how many servers to run, estimate the traffic they must handle.

### Example assumptions

Suppose a booking platform has:

- 100,000 daily active users.
- 100 API requests per user per day.

Daily request volume:

```text
100,000 users * 100 requests/user/day
= 10,000,000 requests/day
```

There are 86,400 seconds in a day.

Average requests per second:

```text
10,000,000 / 86,400
= approximately 116 RPS
```

This means the average request rate is approximately 116 RPS.

But designing for only 116 RPS would be risky if traffic peaks are much higher.

### Estimate peak traffic

Assume peak traffic is five times the daily average.

```text
Peak RPS = Average RPS * Peak Multiplier

Peak RPS = 116 * 5
         = approximately 580 RPS
```

The five-times multiplier is an assumption, not a universal rule. Ideally, use production traffic patterns or realistic load tests to estimate peaks.

### What this estimate does not tell us

DAU and requests per user provide only a starting point. A more complete estimate may require:

- Requests per minute by time of day.
- Concurrent users.
- Endpoint-specific request rates.
- Read/write ratios.
- Request payload sizes.
- Database queries per API request.
- Long-running requests.
- Background job volume.
- Traffic growth expectations.

Different endpoints may have very different costs. One RPS value does not capture the full workload.

---

## 9. Estimate the Number of Application Servers

Suppose:

- Expected peak traffic = 580 RPS.
- Each application server sustains 200 RPS while meeting latency and error-rate targets.

The theoretical minimum number of servers is:

```text
Required Servers = ceil(Peak RPS / Capacity per Server)

Required Servers = ceil(580 / 200)
                 = 3
```

Three servers have a combined nominal capacity of 600 RPS.

However, this leaves little headroom, and one failed server would leave only 400 RPS.

The next step is to account for failures.

---

## 10. Capacity Planning with Failure Tolerance

Suppose the system must support 580 RPS even when one application server is unavailable.

Each server can sustain 200 RPS.

With three servers:

```text
Normal capacity = 3 * 200 = 600 RPS
After one failure = 2 * 200 = 400 RPS
```

The system cannot meet the 580 RPS requirement after a failure.

With four servers:

```text
Normal capacity = 4 * 200 = 800 RPS
After one failure = 3 * 200 = 600 RPS
```

Four servers meet the stated peak requirement after one server fails.

### General formula

Let:

- D = Required peak demand in RPS.
- C = Sustainable capacity per server.
- F = Number of simultaneous server failures to tolerate.

Then, under simplified assumptions:

```text
Required server count = ceil(D / C) + F
```

For this example:

```text
ceil(580 / 200) + 1
= 3 + 1
= 4 servers
```

This formula assumes the remaining servers can use their full sustainable capacity and that there are no additional bottlenecks. In a real architecture, validate the assumptions through load testing and failure testing.

### Important distinction

Having enough servers to survive one failure is not the same as having enough capacity for multiple failures.

If the system must survive two simultaneous failures, the simplified estimate becomes:

```text
ceil(580 / 200) + 2
= 5 servers
```

Whether this level of redundancy is necessary depends on the availability target, risk, cost, and failure model.

---

## 11. Utilization and Headroom

Utilization measures the fraction of available capacity currently being used.

```text
Utilization = Actual Load / Available Capacity * 100%
```

Suppose three healthy servers remain after one failure:

```text
Total capacity = 3 * 200 = 600 RPS
Actual load = 500 RPS

Utilization = 500 / 600 * 100%
            = 83.3%
```

The remaining capacity is:

```text
Headroom = Total Capacity - Actual Load
         = 600 - 500
         = 100 RPS
```

### Why headroom matters

Headroom allows the system to absorb:

- Sudden traffic spikes.
- Uneven request distribution.
- Temporary increases in processing cost.
- Some capacity loss during failures.
- Workload variability.

A system running close to saturation may experience rapidly increasing queueing and latency.

However, there is no universal ideal utilization percentage. The right target depends on workload variability, scaling speed, failure tolerance, and service-level objectives.

### Important caution

Server CPU utilization alone is not enough to determine whether the system has sufficient headroom.

A system with 40% application CPU utilization may still be constrained by:

- Database connections.
- Database CPU.
- Network I/O.
- Locks.
- External service latency.
- Memory or thread limits.

Measure the resources that actually constrain the workload.

---

## 12. Sustainable Capacity Is More Important Than Maximum Capacity

A server may technically process 300 RPS but produce unacceptable P99 latency and errors at that load.

Suppose testing produces:

| Load per server | P99 latency | Error rate |
|---:|---:|---:|
| 150 RPS | 120 ms | 0.1% |
| 200 RPS | 180 ms | 0.2% |
| 250 RPS | 450 ms | 1.5% |
| 300 RPS | 1,200 ms | 5% |

If the system requires P99 latency below 300 ms and an error rate below 1%, 200 RPS is the highest tested load that meets both targets.

The server's sustainable capacity for this design is therefore approximately 200 RPS, not 300 RPS.

These figures are illustrative. Real capacity must be measured under representative workloads, including database activity and realistic request distributions.

**Capacity planning should use the highest throughput that meets the performance and reliability requirements, with appropriate headroom.**

---

## 13. Why Adding Servers Can Make Things Worse

Consider this architecture:

```text
               Load Balancer
                /         \
               v           v
           App Server A  App Server B
                \         /
                 v       v
                PostgreSQL
```

Suppose the application servers are underutilized, but PostgreSQL is already near saturation.

Adding six more application servers might increase the number of concurrent database requests. This can make query queues longer and cause latency to rise.

The actual bottleneck could be:

- Expensive SQL queries.
- Excessive query volume.
- Poor indexing.
- Database connection contention.
- Lock contention.
- A limited database CPU or I/O budget.

The correct response is to identify the bottleneck first.

Possible solutions include query optimization, connection-pool tuning, batching, caching suitable reads, or increasing database capacity.

Not every problem requires additional application servers.

---

## 14. Load Balancing Does Not Replace Fault Tolerance

Suppose four servers each sustain 200 RPS and peak demand is 500 RPS.

Normal capacity:

```text
4 * 200 = 800 RPS
```

After one failure:

```text
3 * 200 = 600 RPS
```

The remaining servers can still handle 500 RPS, assuming capacity and distribution remain valid.

Now suppose only three servers were deployed.

After one failure:

```text
2 * 200 = 400 RPS
```

The load balancer can redirect traffic to the two healthy servers, but it cannot make them process 500 RPS within the required latency and error-rate limits.

**Routing around a failure and having sufficient capacity after a failure are two different requirements.**

Also consider whether the load balancer itself, the database, or another shared component introduces a single point of failure.

---

## 15. Autoscaling

Autoscaling adjusts the number of application instances according to configured signals or schedules.

Common signals include:

- CPU utilization.
- Request rate.
- Request latency.
- Queue depth.
- Concurrent requests.
- Custom application metrics.

### Benefits

- Can increase capacity as demand rises.
- Can reduce infrastructure cost when demand falls.
- Reduces the need to predict every traffic change in advance.

### Trade-offs

- Scaling takes time.
- New instances need initialization and warm-up.
- Poor thresholds can cause scaling oscillations.
- A sudden traffic spike may arrive before new instances are ready.
- Scaling the application tier may overload a shared database.

Autoscaling should have suitable minimum and maximum instance counts, sensible thresholds, cooldown or stabilization behavior, and monitoring.

It does not eliminate the need for baseline capacity or failure planning.

---

## 16. Common Mistakes

### Mistake 1: Planning only for average traffic

Average traffic hides peaks. Estimate the peak workload and understand its variability.

### Mistake 2: Adding application servers without checking dependencies

The database, cache, queue, or external API may be the actual bottleneck.

### Mistake 3: Assuming every request has the same cost

A health-check request and a complex report request may consume very different resources.

### Mistake 4: Treating the load balancer as a capacity solution

It distributes work among available instances; it does not create additional compute capacity.

### Mistake 5: Using local in-memory sessions without considering scaling

A subsequent request may reach a different server. Session state must be handled deliberately.

### Mistake 6: Assuming maximum throughput is sustainable capacity

The system must meet its latency and error-rate requirements at the chosen capacity.

### Mistake 7: Ignoring failure scenarios

A system that meets peak demand only when every server is healthy may not meet its availability requirements.

### Mistake 8: Overprovisioning without a defined requirement

Additional servers cost money. Redundancy should follow explicit availability and capacity objectives rather than arbitrary instance counts.

---

## 17. A Practical Capacity-Planning Checklist

When planning application server capacity, ask:

1. What is the expected average traffic?
2. What is the expected peak traffic?
3. How quickly can traffic spike?
4. What is the latency target, including P95 or P99?
5. What error rate is acceptable?
6. What sustainable RPS can one instance handle?
7. Does that capacity include realistic database and dependency work?
8. How many instances must remain after a failure?
9. What headroom is needed?
10. What other components may become bottlenecks?
11. How quickly can autoscaling add capacity?
12. What metrics will confirm that the capacity estimate is correct?

Capacity estimates are hypotheses until validated by load testing and production observations.

---

## 18. Worked Example: Booking Platform

### Requirements

- Peak demand: 500 RPS.
- Each application server sustains 200 RPS.
- One application server may fail.
- The system must continue meeting peak demand after that failure.

### Design

Use four application servers behind a load balancer.

### Validation

Normal operation:

```text
4 * 200 = 800 RPS
```

After one failure:

```text
3 * 200 = 600 RPS
```

Remaining headroom:

```text
600 - 500 = 100 RPS
```

Utilization after failure:

```text
500 / 600 * 100% = 83.3%
```

### Decision

Four servers satisfy the simplified capacity requirement. Before production, validate that:

- The 200 RPS capacity estimate is realistic.
- The load balancer can route around an unhealthy instance.
- The database can support the peak workload.
- Application instances do not depend on local session state.
- The system behaves correctly during a server failure.
- The service continues to meet latency and error-rate targets.

The design may require additional capacity if traffic variability, uneven distribution, or downstream limits reduce the effective throughput.

---

## 19. Revision Questions

### Q1. What is the difference between vertical and horizontal scaling?

Answer:
Vertical scaling increases the resources of an existing machine. Horizontal scaling adds machines or application instances and distributes the workload.

### Q2. Why is horizontal scaling not always linear?

Answer:
Shared dependencies, coordination overhead, uneven traffic, and other bottlenecks can prevent each new instance from adding its full nominal capacity.

### Q3. Why are stateless application servers easier to scale?

Answer:
Requests can be handled by different instances without depending on state held only in a previous instance's memory.

### Q4. What does a load balancer do?

Answer:
It distributes requests among instances and can route around unhealthy instances when configured appropriately. It does not create processing capacity.

### Q5. What is the difference between average and peak RPS?

Answer:
Average RPS represents the average request rate over a period. Peak RPS represents a high-demand interval that the system may need to support.

### Q6. How many servers are required to handle 580 RPS if each sustains 200 RPS and one failure must be tolerated?

Answer:
Four servers under the simplified assumptions:

ceil(580 / 200) + 1 = 4.

### Q7. Why is CPU utilization alone insufficient for capacity planning?

Answer:
A system may be constrained by database connections, network I/O, locks, external dependencies, or other resources while CPU remains relatively low.

### Q8. What is headroom?

Answer:
The capacity remaining above the current workload. It helps absorb traffic spikes, failures, and variability.

### Q9. Why can adding application servers overload the database?

Answer:
More instances may generate more concurrent database queries and connections, increasing contention and latency.

### Q10. What should define sustainable server capacity?

Answer:
The throughput the server can sustain while meeting the system's latency, error-rate, and reliability requirements.

---

## 20. Final Summary

Remember these core principles:

- Scalability is about handling increased workload while maintaining acceptable behavior.
- Vertical scaling increases the resources of an existing machine.
- Horizontal scaling adds instances and distributes work.
- Load balancing helps distribute traffic but does not create capacity.
- Stateless application instances make scaling and replacement easier.
- Estimate average and peak traffic before selecting capacity.
- Plan for the required number of failures.
- Use sustainable throughput, not theoretical maximum throughput.
- Maintain headroom based on workload variability and reliability goals.
- Identify the actual bottleneck before scaling a component.
- Autoscaling helps respond to changing demand but is not a replacement for capacity planning.
- Validate assumptions through load testing and failure testing.

**The central lesson:** Good capacity planning is not about maximizing the number of servers. It is about providing enough sustainable capacity to meet performance and reliability requirements at an acceptable cost.
