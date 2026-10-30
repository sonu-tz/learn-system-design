# Lesson 8: End-to-End System Design — Ride-Booking Platform

## 1. What Is End-to-End System Design?

End-to-end system design is the process of designing a complete system by combining requirements, architecture, data storage, networking, performance, reliability, security, and operational concerns.

In earlier lessons, we studied individual concepts:

- Scalability and capacity planning.
- Latency, throughput, and utilization.
- Bottlenecks and performance budgets.
- Caching and consistency.
- Resilience and failure handling.
- Asynchronous processing and message queues.

Now we will apply those concepts to one complete system.

The objective is not to create the most complicated architecture. It is to build the simplest architecture that satisfies the requirements and can evolve as demand grows.

## 2. Understand the Problem Before Designing

Imagine we need to build a ride-booking platform.

Passengers should be able to:

1. Register and log in.
2. View nearby available drivers.
3. Request a ride.
4. Receive a driver's assignment.
5. Track the driver's location.
6. See ride status changes.
7. Complete the ride and view the final fare.

Drivers should be able to:

1. Go online or offline.
2. Share their current location.
3. Receive ride requests.
4. Accept or reject a request.
5. Update the ride status.

These are the functional requirements.

### Functional requirements

| Requirement | Description |
|---|---|
| Authentication | Identify passengers and drivers. |
| Driver availability | Determine which drivers are available. |
| Ride creation | Create a ride request. |
| Driver assignment | Assign an appropriate available driver. |
| Live tracking | Deliver driver location updates to passengers. |
| Ride lifecycle | Manage requested, accepted, arrived, started, and completed states. |
| Notifications | Inform users about important status changes. |
| Fare calculation | Calculate the final fare according to business rules. |

### Non-functional requirements

These describe how the system should behave.

| Requirement | Example objective |
|---|---|
| Latency | Ride requests should normally receive a response within a defined target. |
| Scalability | Support growth in passengers, drivers, and concurrent rides. |
| Availability | Keep critical ride operations available during component failures. |
| Consistency | Prevent two drivers from being assigned the same ride. |
| Freshness | Deliver driver locations with an acceptable delay. |
| Durability | Do not lose confirmed ride records after a successful commit. |
| Security | Protect user information and authorize ride operations. |
| Observability | Diagnose latency, failures, and assignment problems. |

The example targets must eventually be quantified. We should not claim that the system meets a latency or availability target until it has been designed, measured, and tested against it.

### Why requirements come first

Different requirements lead to different architectural decisions.

For example:

- If driver locations can be a few seconds old, caching is easier.
- If ride assignments must be correct, the assignment operation needs strong concurrency controls.
- If notifications can be delayed, a queue is appropriate.
- If the system needs live tracking, a request-response API alone may not be sufficient.

A common system-design mistake is choosing technologies before understanding these requirements.

## 3. Estimate the Scale

Before selecting infrastructure, estimate how much work the system needs to handle.

For this example, assume the following hypothetical traffic:

- 1,000,000 daily active passengers.
- 100,000 daily active drivers.
- 10 ride-related API requests per passenger per day.
- 100 location updates per active driver per hour.
- 12 hours of daily driver activity.

These numbers are assumptions for learning, not actual production measurements.

### Estimate API request volume

Passenger API requests:

\[
1{,}000{,}000 \times 10
= 10{,}000{,}000
\]

That is 10 million passenger API requests per day.

Average passenger request rate:

\[
\frac{10{,}000{,}000}{86{,}400}
\approx 116 \text{ RPS}
\]

RPS means requests per second.

This is only the passenger API estimate. Driver operations, tracking, notifications, and other APIs add more load.

If peak traffic is five times the average:

\[
116 \times 5 \approx 580 \text{ RPS}
\]

The system should be evaluated against peak demand rather than average demand alone.

### Estimate location updates

Assume 100,000 drivers each send 100 updates per hour for 12 hours.

\[
100{,}000 \times 100 \times 12
= 120{,}000{,}000
\]

That is 120 million driver location updates per day.

Average update rate during the assumed 12-hour activity period:

\[
\frac{120{,}000{,}000}{12 \times 3{,}600}
\approx 2{,}778 \text{ updates/second}
\]

This estimate assumes updates are spread evenly across those 12 hours. Real traffic may be concentrated in particular locations and times.

Notice the important result: location updates can produce much more traffic than passenger-facing ride requests.

This is why live tracking often needs a different processing strategy from transactional ride operations.

### What capacity planning tells us

We now have at least two distinct workload types:

1. Ride-management requests: relatively low volume but high business importance.
2. Location updates: high volume, frequent, and freshness-sensitive.

They should not automatically share identical scaling and data-processing strategies.

## 4. Define the High-Level Architecture

Start with a simple architecture.

```text
                       Passengers / Drivers
                                |
                                v
                         DNS / Load Balancer
                                |
                                v
                         Application Layer
                          /      |       \
                         v       v        v
                    Ride API  Tracking   Auth
                         |       API
                         v
                      PostgreSQL
                         |
                         v
                    Message Queue
                         |
                         v
                    Background Workers
```

This is a logical overview, not a complete deployment diagram.

We still need to decide how live updates reach clients, how drivers are assigned, and how database and queue failures are handled.

### Component responsibilities

**Client applications**

Passenger and driver apps provide the user interface, send requests, and receive status and location updates.

**Load balancer**

Distributes incoming traffic across healthy application instances.

**Application layer**

Validates requests, applies business rules, authorizes operations, and coordinates services.

**PostgreSQL**

Stores durable business data such as users, rides, assignment records, and ride lifecycle state.

**Tracking subsystem**

Processes frequent driver location updates and makes the latest location available to authorized clients.

**Message queue**

Buffers work that can happen asynchronously, such as notifications, analytics, and downstream event processing.

**Background workers**

Consume messages, send notifications, update secondary systems, and perform other deferred tasks.

The key is that each component has a defined responsibility. We should avoid adding separate microservices unless independent scaling, ownership, or isolation justifies the operational complexity.

## 5. Choose the Database and Data Model

A ride-booking system has both transactional data and high-frequency location data.

These workloads have different characteristics.

### Transactional data

Examples:

- Passenger accounts.
- Driver accounts.
- Ride records.
- Driver assignments.
- Fare records.
- Payment references.
- Ride status transitions.

PostgreSQL is a strong starting choice because it supports relational modeling, transactions, constraints, and consistency guarantees.

A simplified ride record might contain:

```text
Ride
----
ride_id
passenger_id
driver_id
status
pickup_location
destination
estimated_fare
final_fare
created_at
updated_at
```

A separate assignment record may capture the history of which driver was assigned and when.

### Why transactions matter

Suppose two drivers try to accept the same ride at nearly the same time.

A naive design could allow both requests to read the ride as available and both proceed to assign themselves.

The system needs an atomic operation or concurrency-control strategy that permits only one successful assignment.

Possible techniques include:

- Conditional updates.
- Row-level locking.
- Uniqueness constraints.
- Transaction isolation appropriate to the operation.

For example, a conditional update can change the ride from `requested` to `accepted` only if it is still in the expected state. The application must check whether the update actually succeeded.

**The important principle:** The database, not merely the UI or an in-memory check, must enforce the business invariant.

### Location data

Driver locations change frequently and are often consumed as the latest known position rather than as a complete historical record.

The system might store:

- Latest location in a fast-access store.
- Recent location history for operational needs.
- Long-term location history only where required by business, audit, or legal requirements.

The latest location could be held in an in-memory or distributed key-value store, while durable records are stored separately according to retention and recovery requirements.

Do not automatically store every location update forever in the main transactional table. That can create unnecessary write load, storage growth, and query contention.

## 6. Design the Ride-Creation Flow

Let's trace the most important user journey.

A passenger requests a ride.

```text
Passenger
    |
    v
Ride API
    |
    v
Validate request
    |
    v
PostgreSQL transaction
    |
    +---- Create ride
    |
    +---- Create outbox event
    |
    v
Commit transaction
    |
    v
Return ride request accepted
    |
    v
Outbox publisher
    |
    v
Message queue
    |
    v
Driver assignment workflow
```

### Step-by-step

1. Authenticate the passenger.
2. Validate pickup and destination.
3. Validate business rules, including any applicable request limits.
4. Create the ride record.
5. Record the corresponding event in the transactional outbox.
6. Commit the transaction.
7. Return the appropriate response.
8. Publish the event asynchronously.
9. Start or notify the driver-assignment workflow.
10. Update the ride state as the workflow progresses.

### What does the API response mean?

A successful response at step 7 means the ride request has been accepted and durably recorded.

It does not necessarily mean that a driver has already been assigned.

The API should communicate the correct state, such as `requested` or `searching_for_driver`, rather than claiming the ride is confirmed prematurely.

### Why use the outbox?

If the database commit succeeds but event publication fails, a plain database-then-queue implementation can leave the ride without a corresponding assignment event.

The transactional outbox records the ride and event together in one database transaction. A publisher can retry publishing later.

The consumer must still tolerate duplicate events.

## 7. Design Driver Assignment

Driver assignment is one of the most business-critical operations.

A simplified workflow:

```text
Ride requested
      |
      v
Find eligible nearby drivers
      |
      v
Send offer to a candidate
      |
      v
Driver accepts?
   /       \
 No         Yes
 |           |
 v           v
Try next   Atomically assign
candidate  driver to ride
               |
               v
         Notify passenger
```

Real assignment systems may involve ranking, timeouts, multiple candidates, pricing rules, and cancellation handling. This diagram shows the core idea.

### Nearby-driver search

The system needs to find drivers who are:

- Online.
- Available for assignment.
- Close enough to be eligible.
- Compatible with the ride requirements.

A spatial index or geospatial query can help identify nearby candidates. The system can then rank eligible drivers according to business rules.

However, location alone does not establish availability. A driver may have gone offline or accepted another ride since the location was last updated.

### Avoid double assignment

Suppose two workers attempt to assign the same ride concurrently.

Both may identify an eligible driver, but only one should win the assignment operation.

Use an atomic database operation, transaction, or equivalent coordination mechanism that enforces the invariant.

The losing worker must detect the conflict and take the appropriate next step.

### Idempotency and event handling

Assignment messages may be delivered more than once.

Consumers should recognize repeated events and avoid creating duplicate assignments. State transitions should validate the current ride state, and the database should enforce the relevant invariants.

This is a direct application of the resilience and asynchronous-processing lessons.

## 8. Design Real-Time Driver Tracking

Live tracking has different performance requirements from ride creation.

The system may need to process thousands of location updates per second, but most consumers care about the driver's latest position rather than every intermediate point.

### Proposed architecture

```text
Driver App
    |
    v
Location Ingestion API
    |
    v
Validate / Authenticate / Rate Limit
    |
    v
Latest-Location Store
    |
    v
Real-Time Distribution Layer
    |
    v
Passenger App
```

The distribution layer could use a suitable publish/subscribe or persistent connection technology.

The exact technology depends on concurrency, delivery guarantees, latency, infrastructure constraints, and client support.

### Latest location vs. historical location

These are different requirements.

**Latest location:** Where was the driver most recently observed?

**Historical location:** Where was the driver over a previous time interval?

For latest location, a key-value representation can be effective:

```text
driver_id -> {
  latitude,
  longitude,
  observed_at,
  sequence
}
```

The timestamp indicates when the location was observed. A sequence number or comparable ordering mechanism can help prevent older updates from overwriting newer ones.

### Why stale updates matter

Imagine location update 101 arrives before update 100, even though update 100 was generated earlier.

If the system blindly stores whichever message arrives last, it could replace a newer position with an older one.

Use event timestamps or sequence numbers with clearly defined ordering rules. If client clocks are unreliable, server-assigned ordering or sequence mechanisms may be needed.

### Batching and coalescing

If the driver sends many updates in a short period, the system may not need to distribute every update to every passenger.

For some consumers, processing only the latest location at a bounded interval reduces redundant work.

However, coalescing should be used only where intermediate updates are not required. Do not apply it indiscriminately to events that represent important business transitions.

## 9. Caching and Consistency

Caching can reduce repeated reads and improve response latency.

Possible candidates include:

- Latest driver location.
- Frequently accessed, relatively stable metadata.
- Public configuration.
- Selected read-heavy lookup data.

However, ride status and driver assignment require more careful consistency decisions.

### Example: Driver location

A location cache with a short freshness window may be appropriate because location updates arrive frequently.

If a passenger sees a location that is two seconds old, that may be acceptable for a defined tracking objective.

### Example: Ride assignment

A stale cache should not determine whether a driver can accept a ride.

The authoritative assignment operation must use a consistency and concurrency strategy that prevents conflicting assignments.

### A useful classification

| Data | Typical strategy |
|---|---|
| Latest driver location | Fast latest-value store, with freshness checks |
| Ride status | PostgreSQL as the authoritative state |
| Driver availability | Carefully coordinated state; stale reads must not authorize conflicting assignments |
| Static configuration | Cache with a suitable invalidation or TTL policy |
| Notification history | Durable database record where audit or user history is required |

The exact approach depends on the business requirements, but the principle remains: **not all data needs the same consistency guarantees.**

## 10. Synchronous vs. Asynchronous Operations

We should explicitly classify the system's operations.

| Operation | Processing approach | Reason |
|---|---|---|
| Authenticate a ride request | Synchronous | The API needs the authorization result |
| Persist a ride request | Synchronous | The API must know the transaction committed before confirming acceptance |
| Enforce assignment uniqueness | Synchronous atomic operation or equivalent | Conflicting assignments must be prevented |
| Send a noncritical email | Asynchronous | Delivery can be delayed |
| Update analytics | Asynchronous | Analytics should not block the core transaction |
| Publish committed ride events | Asynchronous through an outbox publisher | Decouples publication from the request |
| Deliver live locations | Real-time streaming or push distribution | Frequent updates should reach interested clients efficiently |
| Generate a historical report | Asynchronous for expensive reports | Avoid long-running user requests |

A critical business process may still use asynchronous orchestration. In that case, the API must expose a pending state until the required work completes.

## 11. Scaling the System

Now consider how the architecture evolves as traffic grows.

### Application layer

Start with multiple stateless application instances behind a load balancer.

Scale horizontally based on measured demand, latency, CPU, memory, and concurrency.

Avoid storing essential session or assignment state only in a particular instance's local memory.

### Database layer

Optimize queries and indexes before assuming a larger database is necessary.

Then evaluate:

- Connection-pool sizing.
- Read replicas for suitable read-heavy workloads.
- Partitioning for very large datasets.
- Archival and retention policies.
- Separate storage strategies for location and transactional workloads.

Read replicas may provide stale data, so they should not automatically be used for operations that require current assignment state.

### Tracking layer

Scale location ingestion and distribution independently from ride management.

Use bounded queues, rate limits, batching where appropriate, and capacity limits to prevent tracking traffic from exhausting resources needed for booking operations.

### Workers

Scale assignment and notification workers based on queue backlog, message age, processing rate, and downstream capacity.

More workers are not always better if they overload the database or an external provider.

### Cache layer

Use caching to reduce repeated reads where the freshness and consistency requirements permit it.

Measure hit rate, memory utilization, eviction rate, and latency before assuming caching is solving the bottleneck.

## 12. Resilience and Failure Handling

A production system must handle partial failure.

### Scenario A: PostgreSQL becomes slow

Potential protections:

- Query and connection-acquisition timeouts.
- Overall request deadlines.
- Connection-pool limits.
- Concurrency controls.
- Load shedding.
- Query optimization and database monitoring.

Avoid unlimited retries, which can amplify database load.

### Scenario B: Notification provider is unavailable

Keep notifications in a durable queue and retry with bounded backoff and jitter.

Do not make the booking transaction depend on email delivery unless the business explicitly requires it.

### Scenario C: Message consumer crashes

The queue may redeliver unacknowledged messages.

Use idempotent consumers and acknowledge work after the required business effect has committed.

### Scenario D: Tracking infrastructure is overloaded

Possible protections include rate limiting, coalescing eligible updates, separating tracking resources from booking resources, and temporarily reducing nonessential tracking work.

The system should communicate stale or unavailable tracking data honestly.

### Scenario E: A driver accepts a ride while the ride is cancelled

Use authoritative state transitions and concurrency controls. The assignment operation must validate the current state rather than trusting a stale message or client view.

These examples illustrate a general rule: reliability comes from handling failure at the correct boundary, not from adding a retry to every operation.

## 13. Observability: Know What the System Is Doing

A system cannot be operated reliably if its behavior cannot be measured.

### Metrics

Track:

- API request rate.
- API latency percentiles, including P95 and P99.
- Error and timeout rates.
- Database query latency.
- Database connection-pool utilization.
- Ride assignment success and failure rates.
- Time from ride request to assignment.
- Location update rate and freshness.
- Queue depth and oldest-message age.
- Worker processing latency and retry rate.
- Cache hit rate and eviction rate.

### Logs

Use structured logs for significant events, including:

- Ride creation.
- Assignment attempts and outcomes.
- State transitions.
- Message-processing failures.
- Retry attempts.
- Timeout and circuit-breaker events.

Avoid logging secrets or unnecessary sensitive location information.

### Distributed tracing

A trace can follow a ride request through:

```text
Client request
   -> Ride API
      -> PostgreSQL
      -> Outbox publisher
         -> Message broker
            -> Assignment worker
               -> Assignment database operation
```

Asynchronous boundaries often require explicit trace-context propagation to connect the producer and consumer work.

### Why these three tools differ

- Metrics show trends and aggregate system health.
- Logs explain individual events.
- Traces show how work moves through the system and where time is spent.

Together, they help identify bottlenecks and explain failures.

## 14. Security and Data Protection

System design also needs to address security from the beginning.

For this platform:

- Authenticate passengers and drivers.
- Authorize every ride operation against the relevant user and ride.
- Prevent one passenger from reading another passenger's ride details.
- Protect driver location updates against unauthorized publication.
- Validate input and enforce rate limits.
- Use encrypted connections.
- Protect credentials and signing keys.
- Avoid logging tokens, payment secrets, or unnecessary personal data.
- Apply retention and access controls to location history.
- Audit sensitive administrative operations.

Real-time subscriptions also need authorization. Knowing a channel name should not be sufficient to access a passenger's ride or a driver's private location.

Security is part of the architecture, not a final checklist added after development.

## 15. High-Level Architecture: Final View

Combining the previous decisions, the logical architecture looks like this:

```text
                  Passenger App / Driver App
                             |
                             v
                      Load Balancer
                             |
                  +----------+----------+
                  |                     |
                  v                     v
             Ride Service         Tracking Service
                  |                     |
                  v                     v
             PostgreSQL          Latest-Location Store
                  |
                  v
              Outbox Table
                  |
                  v
             Outbox Publisher
                  |
                  v
              Message Broker
             /       |       \
            v        v        v
       Assignment  Notification Analytics
          Worker      Worker     Worker
            |
            v
      PostgreSQL transaction

Tracking Service
        |
        v
Real-Time Distribution
        |
        v
Passenger App
```

This is a logical architecture, not a requirement to deploy every box as a separate microservice.

A modular monolith can implement multiple logical responsibilities initially. Separate deployment and independent scaling become worthwhile when traffic, team ownership, failure isolation, or operational needs justify them.

## 16. Important Trade-Offs

| Decision | Benefit | Cost or risk |
|---|---|---|
| PostgreSQL for transactional state | Strong transactional capabilities and constraints | Requires careful scaling and query management |
| Separate tracking subsystem | Isolates high-volume location traffic | Adds infrastructure and operational complexity |
| Asynchronous notifications | Reduces coupling and request latency | Introduces delivery delay and duplicate-handling requirements |
| Transactional outbox | Closes the database-to-event publication gap | Adds publisher logic, storage, and monitoring |
| Cache latest locations | Fast reads and reduced repeated work | Staleness and invalidation concerns |
| Horizontal scaling | Adds application capacity | More instances increase downstream demand |
| Read replicas | Can increase read capacity | Replication lag can expose stale state |
| Microservices | Independent deployment and scaling | Network failures, distributed debugging, and more operational overhead |

Good system design means understanding these trade-offs and selecting the option that fits the requirements.

## 17. How to Approach Any System Design Interview

Use this sequence when asked to design a system.

### Step 1: Clarify requirements

Identify users, core workflows, non-functional requirements, and what is explicitly out of scope.

### Step 2: Estimate scale

Estimate active users, requests per second, data volume, peak traffic, and read/write ratios.

### Step 3: Define APIs and data

Identify core operations, important entities, state transitions, and data consistency requirements.

### Step 4: Draw the high-level architecture

Start with clients, application services, storage, caching, and asynchronous components where needed.

### Step 5: Walk through critical flows

Explain how the most important operation works from request to durable state and response.

### Step 6: Identify bottlenecks

Estimate which resources become constrained first and explain how you would measure them.

### Step 7: Handle failures

Discuss timeouts, retries, idempotency, concurrency, dependency failures, and recovery.

### Step 8: Explain scaling decisions

Show what you would scale, when you would scale it, and what downstream limits might remain.

### Step 9: Discuss trade-offs

Explain why you selected each major component and what you would change at a larger scale.

### Step 10: Define observability and security

Explain how you would detect problems, protect data, and verify that the system meets its objectives.

Do not begin by naming every technology you know. Begin with the problem and justify each design decision.

## 18. Common Mistakes

1. Starting with microservices before understanding the requirements.
2. Designing around average traffic without considering peak demand.
3. Treating all data as if it requires the same consistency.
4. Using a cache to make decisions that require authoritative state.
5. Assuming a queue guarantees exactly-once business effects.
6. Ignoring concurrency problems in driver assignment.
7. Allowing location updates to overwhelm transactional operations.
8. Scaling application servers without checking database capacity.
9. Returning success before the required business operation is durable.
10. Ignoring observability, security, retention, and operational recovery.
11. Overengineering the first version instead of allowing the architecture to evolve.
12. Describing components without explaining why they are needed.

## 19. Revision Questions

Try answering these without looking at the notes.

1. What are the functional and non-functional requirements of the ride-booking platform?
2. Why should ride-management traffic and location-update traffic be treated differently?
3. How would you estimate the system's peak request rate?
4. Why is PostgreSQL a reasonable starting point for transactional ride data?
5. How would you prevent two drivers from being assigned the same ride?
6. Why might the system use a separate latest-location store?
7. What problem does the transactional outbox solve?
8. Which operations should be synchronous, and which can be asynchronous?
9. How would you protect ride creation when the notification provider is down?
10. Why should a cache not be the authority for driver assignment?
11. What happens if the queue receives messages faster than workers process them?
12. Which metrics would you monitor to diagnose slow ride assignment?
13. What would you do if tracking traffic started exhausting database connections?
14. When would you introduce separate microservices?
15. Explain the complete flow from ride creation to driver assignment.

## Final Takeaway

End-to-end system design is about connecting requirements to architecture and architecture to measurable behavior.

For the ride-booking platform:

- PostgreSQL protects transactional business state.
- Atomic operations and constraints protect ride-assignment correctness.
- A dedicated tracking strategy handles frequent location updates.
- Caching improves eligible reads while accepting explicitly defined freshness limits.
- Queues and an outbox support reliable asynchronous processing.
- Timeouts, bounded retries, idempotency, and resource isolation limit failures.
- Metrics, logs, traces, and security controls make the system operable.

The best design is not the one with the most components. It is the one that satisfies the requirements with understandable trade-offs and a clear path to scale.

## Your practice task

Now apply the design yourself. Imagine your ride-booking platform is experiencing these three problems:

1. Driver location updates are arriving at 3,000 per second and slowing down ride-related database queries.
2. Two drivers occasionally get assigned to the same ride.
3. The notification service goes down for five minutes, causing booking requests to fail.

Try explaining one architectural or design change for each problem, and why it would solve the issue. You don't need to write code.