# Lesson 28 — Microservices Architecture: Service Boundaries, Communication, and Trade-offs

This lesson moves from individual system components to the architecture of an entire application.

We'll explore how to divide a large application into services, how those services communicate, how to manage data ownership, and when microservices are actually worth the additional complexity.

A key principle: Microservices are not automatically better than a well-designed monolith. The goal is to choose boundaries that allow teams and systems to evolve independently.

## 1. What Are Microservices?

Microservices architecture is an architectural style in which an application is divided into multiple small, independently deployable services.

Each service owns a specific business capability and is responsible for its own logic and data.

For example, consider a ride-booking platform.

Instead of building one large application that handles everything, we could divide it into:

- User Service
- Booking Service
- Driver Service
- Dispatch Service
- Payment Service
- Notification Service
- Tracking Service

Each service has a focused responsibility and communicates with other services through well-defined interfaces.

### Monolithic architecture

```text
             Client Applications
                     |
                     v
             Monolithic Backend
          +-----------------------+
          | User Management       |
          | Booking Management    |
          | Driver Management     |
          | Dispatch              |
          | Payments              |
          | Notifications         |
          | Tracking              |
          +-----------------------+
                     |
                     v
                Database
```

All business modules are deployed as one application.

### Microservices architecture

```text
                Client Applications
                        |
                        v
                   API Gateway
                        |
       +----------------+----------------+
       |        |       |       |        |
       v        v       v       v        v
     User    Booking  Driver  Dispatch  Payment
    Service  Service  Service Service  Service
       |        |       |       |        |
       v        v       v       v        v
      DB       DB      DB      DB       DB
                        |
                  Message Broker
                        |
                  Notification
                    Service
                        |
                        v
                       DB
```

Each service can be deployed and scaled independently.

## 2. Microservices vs Monolith

| Area | Monolith | Microservices |
|---|---|---|
| Deployment | Usually one application | Independent services |
| Development | Shared codebase | Multiple service codebases or modules |
| Scaling | Scale the application | Scale individual services |
| Database | Often shared | Prefer service-owned data |
| Communication | In-process calls | Network communication |
| Failure isolation | Often limited | Can be improved through isolation |
| Operational complexity | Lower initially | Higher |
| Testing | Easier integration setup | More distributed testing |
| Team ownership | Often shared | Can align with business capabilities |

### Example

Suppose the tracking functionality receives 20,000 location updates per second, while user management receives only 100 requests per second.

In a monolith, you may need to scale the entire application even though only tracking needs additional capacity.

With microservices, the Tracking Service can scale independently.

However, microservices introduce network calls, service discovery, distributed tracing, deployment coordination, and data consistency challenges.

**Important:** Independent scaling is useful only when the system has workloads or organizational needs that justify the separation.

## 3. When Should You Choose Microservices?

Microservices are useful when:

- Different business capabilities have very different scaling requirements.
- Teams need independent development and deployment.
- Different services have distinct availability requirements.
- A large codebase has become difficult to maintain.
- Different capabilities require different technology or storage choices.
- Independent ownership and release cycles provide measurable value.

Microservices may be unnecessary when:

- The application is small.
- The team is small.
- Business requirements change frequently.
- There are few independent scaling requirements.
- The organization lacks the operational maturity to manage distributed systems.

### A practical rule

Start with a modular monolith unless you have a strong reason to distribute the application.

A modular monolith can preserve clear service-like boundaries without introducing network communication and distributed transactions.

## 4. Service Boundaries and Domain-Driven Design

One of the hardest parts of microservices architecture is deciding where one service ends and another begins.

A common mistake is splitting an application based only on technical layers.

For example:

- Controller Service
- Database Service
- Validation Service
- Utility Service

These are not necessarily business capabilities.

Instead, services should be organized around business domains.

### Example: Ride-booking platform

```text
Ride Booking Domain
    |
    +-- Booking
    |
    +-- Driver Management
    |
    +-- Dispatch
    |
    +-- Payment
    |
    +-- Tracking
    |
    +-- Notifications
```

Each domain has different responsibilities.

| Service | Responsibility |
|---|---|
| Booking | Create, update, cancel bookings |
| Driver | Driver profiles, availability and status |
| Dispatch | Find and assign suitable drivers |
| Payment | Payment authorization, capture and refunds |
| Tracking | Process and distribute location updates |
| Notification | Send push, SMS, email and in-app notifications |

### Bounded Context

In Domain-Driven Design (DDD), a bounded context is a boundary within which a particular domain model and its terminology have a consistent meaning.

For example, the term `Driver Status` may mean different things in different contexts.

**Driver Management:**
- Active
- Inactive
- Suspended

**Dispatch:**
- Available
- Busy
- Offline

**Tracking:**
- Sending location
- Stale location
- Connected

These concepts are related but not identical.

Trying to force every service to share one universal Driver model can create unnecessary coupling.

Each bounded context should define the model appropriate to its responsibilities.

## 5. Single Responsibility and Service Ownership

A microservice should own a cohesive business capability.

For example:

**Booking Service owns:**
- Booking creation.
- Booking lifecycle.
- Booking cancellation.
- Booking status.

**Payment Service owns:**
- Payment authorization.
- Payment capture.
- Refund processing.
- Payment transaction history.

Booking Service may request a payment operation, but it should not directly modify Payment Service's database.

This establishes clear ownership.

### Why ownership matters

Without ownership boundaries, services become tightly coupled.

For example:

```text
Booking Service
      |
      v
Updates booking table
      |
      v
Directly updates payment table
      |
      v
Directly updates driver table
```

This is effectively a distributed monolith.

Services are deployed separately but remain dependent on each other's internal database structures.

A better approach:

```text
Booking Service
      |
      v
Booking API / Event
      |
      v
Payment Service

Booking Service
      |
      v
Booking Event
      |
      v
Dispatch Service
```

Each service changes its own data and exposes supported operations or events.

## 6. Database per Service

A core microservices principle is that each service owns its data.

```text
Booking Service  ---> Booking DB

Driver Service   ---> Driver DB

Payment Service  ---> Payment DB

Tracking Service ---> Tracking Store
```

This does not necessarily mean every service must have a separate database server.

Multiple services can use the same database infrastructure while maintaining separate databases, schemas, credentials, and ownership boundaries.

The important principle is that one service should not directly depend on another service's internal tables.

### Advantages

- Independent schema evolution.
- Reduced database coupling.
- Better control over access.
- Services can choose storage technologies suited to their workloads.
- Independent scaling and performance tuning.

### Disadvantages

- Cross-service queries become more complicated.
- Distributed transactions are difficult.
- Data duplication may be necessary.
- Eventual consistency becomes common.
- Reporting may require a separate read model or data warehouse.

### Example

Suppose the frontend needs:

```text
Booking ID
Passenger Name
Driver Name
Payment Status
```

These fields belong to different services.

The frontend should not query four service databases directly.

Possible solutions:

1. API composition.
2. Backend for Frontend (BFF).
3. Materialized read model.
4. Event-driven denormalization.

We will examine these approaches next.

## 7. How Do Microservices Communicate?

Microservices generally communicate through synchronous or asynchronous communication.

### A. Synchronous communication

One service sends a request and waits for the response.

Common technologies:

- REST over HTTP.
- gRPC.

Example:

```text
Booking Service
      |
      | POST /payments/authorize
      |
      v
Payment Service
      |
      | Authorization result
      |
      v
Booking Service
```

**Advantages:**

- Simple request-response model.
- Immediate result.
- Easy to understand for direct operations.

**Disadvantages:**

- Services become runtime-dependent.
- Network latency increases request latency.
- Downstream failures can affect upstream services.
- Excessive synchronous calls can create cascading failures.

### B. Asynchronous communication

A service publishes a message or event without waiting for every consumer to finish processing it.

Common technologies:

- Kafka.
- RabbitMQ.
- Cloud message queues.
- Other event-broker platforms.

Example:

```text
Booking Service
      |
      | BookingCreated
      v
 Message Broker
      |
      +-----------> Notification Service
      |
      +-----------> Dispatch Service
      |
      +-----------> Analytics Service
```

**Advantages:**

- Reduced runtime coupling.
- Independent processing.
- Better buffering during traffic spikes.
- Multiple consumers can react to the same event.

**Disadvantages:**

- Eventual consistency.
- Duplicate delivery handling.
- More complex debugging.
- Ordering and retry challenges.

### Choosing communication style

Use synchronous communication when the caller genuinely needs an immediate response.

Use asynchronous communication when work can continue independently or when multiple consumers need to react to an event.

Do not turn every service interaction into a message. The communication pattern should match the business requirement.

## 8. Avoid Chatty Microservices

A common anti-pattern is making too many small network calls to complete one user operation.

Example:

```text
Client
  |
  v
Booking Service
  |
  +--> User Service
  |
  +--> Driver Service
  |
  +--> Vehicle Service
  |
  +--> Pricing Service
  |
  +--> Payment Service
  |
  +--> Notification Service
```

If each call takes 100 ms and the calls happen sequentially, the total service latency can become significant.

If calls happen in parallel, latency may improve, but failure handling and resource consumption remain important.

### Better approaches

- Combine related business responsibilities where appropriate.
- Avoid unnecessary network hops.
- Use asynchronous events for independent work.
- Introduce read models for frequently required combined data.
- Cache appropriate reference data.
- Design APIs around meaningful business operations.

The objective is not to maximize the number of services. It is to minimize unnecessary coupling while preserving clear ownership.

## 9. API Gateway

An API Gateway is the entry point for client requests.

Instead of exposing every internal service to the internet, clients communicate through a gateway.

```text
                 Web / Mobile
                      |
                      v
                 API Gateway
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       Booking      Driver      Payment
       Service      Service     Service
```

### Responsibilities

- Request routing.
- Authentication integration.
- Rate limiting.
- Request validation.
- TLS termination.
- Logging and tracing.
- API version routing.
- Response transformation, where appropriate.

### What should not happen?

Avoid putting all business logic inside the API Gateway.

For example, the gateway should not become responsible for deciding whether a driver can be assigned to a booking.

That belongs in the appropriate business service.

An API Gateway is an infrastructure and API-management component, not a replacement for domain services.

## 10. Service Discovery

In a distributed system, service instances may change frequently.

Instances can be created, removed, restarted, or moved.

How does Booking Service find the available Payment Service instance?

This is the problem solved by service discovery.

### Client-side discovery

The calling service discovers available instances and selects one.

```text
Booking Service
      |
      v
Service Registry
      |
      v
Payment instances
  /      |      \
 P1      P2      P3
```

### Server-side discovery

The caller sends requests to a stable endpoint, and a load balancer or proxy selects the actual service instance.

```text
Booking Service
      |
      v
Payment Service Endpoint
      |
      v
Internal Load Balancer
      |
   +--+--+
   |  |  |
   P1 P2 P3
```

### Common approaches

- Kubernetes Services and DNS.
- Service registries.
- Internal load balancers.
- Service meshes.

The right choice depends on the deployment platform and operational requirements.

## 11. Distributed Failures and Resilience

In a monolith, a function call typically executes within the same process.

In microservices, every network call introduces additional failure possibilities.

For example:

```text
Booking Service
      |
      v
Payment Service
      |
      v
External Payment Provider
```

Possible failures:

- Payment Service is unavailable.
- Network timeout.
- External provider is slow.
- Duplicate request.
- Payment response is lost.
- Booking Service restarts before receiving the result.

### Important resilience patterns

**Timeouts:** Avoid waiting indefinitely for another service.

**Retries:** Retry only appropriate failures, with bounded attempts and backoff.

**Circuit breakers:** Temporarily stop calling an unhealthy dependency.

**Bulkheads:** Prevent one dependency from consuming all available resources.

**Idempotency:** Ensure repeated requests do not create duplicate business operations.

**Graceful degradation:** Allow unaffected functionality to continue.

These patterns are especially important when synchronous service calls are chained.

## 12. Distributed Transactions and Sagas

A local database transaction cannot automatically guarantee atomicity across multiple independent service databases.

Consider a booking flow:

1. Create booking.
2. Reserve driver.
3. Authorize payment.
4. Confirm booking.

Each operation belongs to a different service or subsystem.

If payment authorization fails after the driver has been reserved, the system must handle that partial completion.

A Saga coordinates these operations using local transactions and compensating actions.

```text
Create Booking
      |
      v
Reserve Driver
      |
      v
Authorize Payment
      |
   +--+--+
   |     |
Success Failure
   |     |
   v     v
Confirm  Release Driver
Booking
         |
         v
      Cancel Booking
```

A compensation is a business action that counteracts an earlier operation. It is not necessarily a literal database rollback.

Saga orchestration or choreography can be used depending on the workflow.

We covered these patterns in Lesson 15.

## 13. Data Aggregation Across Services

A frontend screen may need data from multiple services.

Example: A dispatcher opens a booking details page.

Required information:

- Booking details.
- Passenger information.
- Driver information.
- Vehicle details.
- Payment status.

There are several approaches.

### A. API Composition

A gateway or BFF calls multiple services and combines their responses.

```text
             Client
                |
                v
             BFF API
          /    |    \
         v     v     v
      Booking Driver Payment
         \     |     /
          \    |    /
           Combined Response
```

**Advantage:** Data is retrieved from its owning services.

**Disadvantage:** Runtime latency and availability depend on the participating services.

### B. Materialized Read Model

A separate read model stores data needed by a particular screen or query.

```text
Booking Service ----+
                    |
Driver Service -----+----> Events
                    |         |
Payment Service ----+         v
                             Read Model
                                |
                                v
                              Client
```

The read model is updated as events arrive.

**Advantage:** Fast queries and fewer runtime service calls.

**Disadvantage:** Data may be temporarily stale and must be maintained.

This is closely related to CQRS and event-driven denormalization.

## 14. The Distributed Monolith Anti-Pattern

A distributed monolith is an application split into multiple services that remain tightly coupled.

Typical symptoms:

- One small change requires deploying many services.
- Services frequently access each other's databases.
- A request depends on many synchronous calls.
- Services cannot operate independently.
- A single service failure brings down the entire user journey.
- Teams cannot release independently.

Example:

```text
Service A <----> Service B
    ^                |
    |                v
Service D <----> Service C
```

Every service depends on several others.

This creates distributed-system complexity without receiving the full benefits of microservices.

### How to avoid it

- Establish clear business boundaries.
- Give each service ownership of its data.
- Prefer stable contracts.
- Avoid unnecessary synchronous dependencies.
- Use events for independent business reactions.
- Keep service count proportional to actual organizational and technical needs.

## 15. Microservices Observability

A user request may travel through multiple services.

For example:

```text
Client
  |
  v
API Gateway
  |
  v
Booking Service
  |
  v
Dispatch Service
  |
  v
Driver Service
  |
  v
Message Broker
  |
  v
Notification Service
```

If a booking takes five seconds, which service caused the delay?

Without distributed observability, finding the answer is difficult.

### Important practices

**Distributed tracing:** Propagate trace context across service boundaries.

**Correlation IDs:** Associate related operations and logs.

**Structured logging:** Record service name, operation, request ID, and relevant business identifiers without exposing sensitive data.

**Metrics:** Monitor service latency, error rates, throughput, saturation, and dependency health.

**Message tracing:** Preserve trace context when publishing and consuming asynchronous events.

These are extensions of the observability concepts covered in Lesson 20.

## 16. Security Between Microservices

Internal services should not automatically trust each other just because they are inside a private network.

Important practices include:

- Service-to-service authentication.
- Authorization for individual operations.
- TLS or mutual TLS where appropriate.
- Short-lived credentials.
- Least-privilege service identities.
- Secret management.
- Network segmentation.
- Audit logging.

For example, Notification Service may be allowed to read a booking event but should not automatically have permission to cancel bookings.

A private network is not a substitute for authorization.

## 17. Service Ownership and Team Organization

Microservices work best when service boundaries align with team ownership.

For example:

| Team | Services owned |
|---|---|
| Booking Team | Booking Service |
| Dispatch Team | Dispatch Service |
| Driver Team | Driver Service |
| Payments Team | Payment Service |
| Platform Team | Gateway, infrastructure and observability |

Benefits:

- Clear accountability.
- Independent planning.
- Faster release cycles.
- Clearer operational ownership.

However, creating one team per tiny service can create excessive organizational overhead.

A service boundary should reflect a meaningful business capability and a sustainable ownership model.

## 18. Worked Example: Designing a Ride-Booking Microservices Architecture

### Requirements

- Passengers can create and cancel bookings.
- Drivers can update their availability and location.
- Dispatch assigns suitable drivers.
- Payments are authorized and captured.
- Users receive notifications.
- The system supports real-time tracking.

### Proposed services

| Service | Primary responsibility | Data ownership |
|---|---|---|
| Identity | Authentication and user identity | Identity data |
| Booking | Booking lifecycle | Booking database |
| Driver | Driver profiles and availability | Driver database |
| Dispatch | Assignment decisions | Assignment data |
| Payment | Authorization and refunds | Payment database |
| Tracking | Latest locations and tracking data | Location store |
| Notification | Delivery of messages | Notification database |

### High-level architecture

```text
                  Web / Mobile
                       |
                       v
                  API Gateway
                       |
       +---------------+----------------+
       |               |                |
       v               v                v
    Booking          Driver           Tracking
    Service          Service          Service
       |               |                |
       v               v                v
    Booking DB      Driver DB        Location Store
       |
       v
   Message Broker
       |
       +--------------------+
       |                    |
       v                    v
   Dispatch              Notification
   Service               Service
       |                    |
       v                    v
   Assignment DB       Notification DB
       |
       v
   Payment Service
       |
       v
   Payment DB
```

This is a conceptual architecture. The exact communication paths and ownership boundaries should be determined by detailed requirements.

### Booking flow

1. Passenger submits a booking request.
2. Booking Service validates and creates the booking.
3. Booking Service publishes a `BookingCreated` event through a transactional outbox.
4. Dispatch Service consumes the event.
5. Dispatch Service finds and reserves a suitable driver.
6. Booking status is updated through a defined service contract or event.
7. Payment Service handles the required payment operation.
8. Notification Service sends relevant updates.
9. Tracking Service distributes driver location updates.

### Important design decisions

- Booking Service owns booking lifecycle.
- Driver Service owns driver profile and availability data.
- Dispatch Service owns assignment decisions.
- Payment Service owns payment transactions.
- Tracking Service handles high-volume location updates independently.
- Notifications are asynchronous.
- Cross-service workflows use Sagas where necessary.
- Each service exposes supported APIs or events rather than its internal database.

### Scaling example

Suppose:

- Booking Service receives 500 requests per second.
- Tracking Service receives 20,000 location updates per second.
- Notification Service processes 2,000 messages per second.

These workloads have very different resource and scaling requirements.

Microservices allow independent capacity planning for each workload.

## 19. Common Microservices Mistakes

1. Splitting services based only on technical layers.
2. Creating too many tiny services.
3. Sharing database tables between services.
4. Making excessive synchronous service calls.
5. Treating every service interaction as an event.
6. Ignoring network latency and partial failures.
7. Assuming message delivery is exactly once.
8. Failing to implement idempotency.
9. Using a frontend feature flag as backend authorization.
10. Ignoring distributed tracing.
11. Creating service boundaries without clear team ownership.
12. Migrating to microservices before understanding the existing domain.

## 20. Practical Decision Checklist

Before extracting a service, ask:

### Business boundary
- [ ] Does this capability have a clear business responsibility?
- [ ] Does it have a well-defined domain model?
- [ ] Can its responsibilities be explained independently?

### Data ownership
- [ ] Does it own its data?
- [ ] Can other services access its capabilities through contracts?
- [ ] Are cross-service queries handled intentionally?

### Deployment
- [ ] Does independent deployment provide value?
- [ ] Can old and new versions coexist?
- [ ] Are API and event contracts backward-compatible?

### Reliability
- [ ] Are timeouts and bounded retries defined?
- [ ] Are critical operations idempotent?
- [ ] Can the service fail without bringing down unrelated capabilities?

### Operations
- [ ] Is monitoring available?
- [ ] Is distributed tracing configured?
- [ ] Is there clear team ownership?
- [ ] Can the organization operate another production service reliably?

## 21. Revision Questions

1. What is microservices architecture?
2. What are the main differences between a monolith and microservices?
3. When should you avoid microservices?
4. What is a bounded context?
5. Why should services own their data?
6. What is the difference between synchronous and asynchronous communication?
7. What problems can chatty microservices introduce?
8. What is the role of an API Gateway?
9. What is service discovery?
10. What is a distributed monolith?
11. How does API composition differ from a materialized read model?
12. Why are distributed transactions difficult?
13. How does a Saga help coordinate cross-service workflows?
14. Why is distributed tracing important?
15. Why should internal services authenticate and authorize one another?
16. How should service boundaries relate to team ownership?

## 22. Practical Exercise

Imagine your current application is a large monolithic backend that supports:

- Passenger management.
- Driver management.
- Booking management.
- Dispatch and assignment.
- Payment processing.
- Real-time tracking.
- Notifications.

You are asked to gradually migrate it toward microservices.

Design a migration plan.

Your answer should explain:

1. Which business capability would you extract first, and why?
2. What responsibilities should remain in the monolith initially?
3. How will the new service own its data?
4. How will the monolith communicate with the extracted service?
5. How will you avoid duplicate processing during migration?
6. How will you migrate existing data?
7. How will you monitor the new service?
8. What conditions would tell you that the extraction was successful?

**Key takeaway:** Microservices are about independent business capabilities, ownership, and evolution—not simply splitting an application into multiple deployments. Clear boundaries and disciplined communication matter more than the number of services.

### Practice focus

Work through the migration exercise. Think carefully about which service you would extract first and how you would migrate its data without disrupting the existing monolith.