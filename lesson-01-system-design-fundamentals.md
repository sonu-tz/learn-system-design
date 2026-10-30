# Lesson 1: System Design Fundamentals

## 1. What Is System Design?

System design is the process of deciding how different components of a software
system work together to meet functional and non-functional requirements.

It focuses on architecture, component responsibilities, data flow, scalability,
performance, reliability, and trade-offs.

The goal is not to choose the most complicated architecture. It is to choose an
architecture that meets the requirements with an acceptable level of complexity
and cost.

## 2. Functional vs Non-Functional Requirements

### Functional Requirements

Describe WHAT the system must do.

Examples:
- Users can create, update, and delete tasks.
- Users can view their tasks.
- Users can filter tasks by status.
- Users can authenticate and access their own data.

### Non-Functional Requirements

Describe HOW WELL the system must operate.

Examples:
- API response time should remain below a defined target.
- The system must handle expected peak traffic.
- The system must remain available when an application server fails.
- User data must be protected.
- The system must recover from failures.

Important non-functional requirements include:
- Performance
- Scalability
- Availability
- Reliability
- Security
- Maintainability
- Consistency
- Cost efficiency

## 3. High-Level System Architecture

A typical web application may follow this structure:

Client
  |
  v
DNS / CDN / Load Balancer
  |
  v
Application Server
  |
  v
Database

Responsibilities:

### Client / Frontend
- Displays the user interface.
- Collects user input.
- Calls backend APIs.
- Manages appropriate client-side state.

### Load Balancer
- Distributes requests across application instances.
- Helps route traffic away from unhealthy instances when configured.
- Does not create additional processing capacity by itself.

### Application Server
- Handles API requests.
- Performs authentication, validation, and business logic.
- Reads and writes data through the data layer.

### Database
- Stores persistent application data.
- Enforces data integrity and supports queries and transactions.

Additional components such as caches, queues, CDNs, and search engines
should be introduced when the requirements justify them.

## 4. Core System Design Concepts

### Latency
The time taken to complete an operation or return a response.

Example:
An API responds in 120 milliseconds.

Lower latency generally improves user experience.

### Throughput
The amount of work a system completes per unit of time.

Example:
An API successfully processes 500 requests per second.

Incoming request rate and successful throughput are not always equal.

### Scalability
The ability of a system to handle increased workload.

Vertical scaling:
- Add more CPU, memory, or other resources to an existing machine.
- Simple to operate initially.
- Limited by machine capacity and cost.

Horizontal scaling:
- Add more application instances.
- Can distribute work and improve fault tolerance.
- Requires appropriate load distribution and handling of shared state.

### Availability
The proportion of time a system is operational and accessible.

### Reliability
The ability of a system to perform correctly and consistently over time.

### Utilization
The proportion of available capacity being used.

Utilization = Actual Load / Available Capacity * 100%

High utilization can reduce headroom and increase queueing and latency.

## 5. Capacity Estimation

Capacity estimation helps determine approximately how much traffic a system
needs to support.

Example:

Daily active users (DAU) = 100,000
Average API requests per user per day = 100

Daily requests:
100,000 * 100 = 10,000,000 requests/day

Average requests per second:
10,000,000 / 86,400 = approximately 116 RPS

If peak traffic is 5 times the average:

Peak RPS = 116 * 5 = approximately 580 RPS

Important:
- Average traffic is not enough for capacity planning.
- Estimate peak traffic and account for traffic spikes.
- Real traffic patterns may differ from these assumptions.

## 6. Estimating Application Server Capacity

Assume:
- Peak demand = 580 RPS
- Each server can sustain 200 RPS while meeting performance targets.

Minimum servers without failures:

ceil(580 / 200) = 3 servers

However, three servers cannot sustain 580 RPS if one fails.

To support 580 RPS after losing one server:

- Four servers provide 800 RPS under normal conditions.
- After one fails, three remain.
- Remaining capacity = 3 * 200 = 600 RPS.

Therefore, four servers satisfy the stated requirement, assuming the
capacity estimate is valid and load is distributed effectively.

Always account for headroom, failures, and the limits of downstream dependencies.

## 7. Stateless Application Servers

A stateless application server does not depend on instance-local client state
being available for every subsequent request.

For example:
- A request reaches Server A.
- The next request reaches Server B.
- Server B can still process it because required session or application state
  is available through a shared store, token, or another appropriate mechanism.

Statelessness does NOT mean the system stores no state.

It means the application instance does not need to retain unique local state
between requests for the system to work correctly.

Benefits:
- Easier horizontal scaling.
- Easier replacement of failed instances.
- More flexible load balancing.

## 8. Common Architecture Trade-Offs

### Modular Monolith vs Microservices

Modular monolith:
- One deployable application with clearly separated modules.
- Simpler deployment and debugging.
- Often a good starting point for a new system.

Microservices:
- Independently deployable services with explicit boundaries.
- Can support independent scaling and team ownership.
- Introduce network communication, distributed failure, and operational complexity.

Choose microservices when independent deployment, scaling, ownership, or
other concrete requirements justify their costs.

### PostgreSQL vs NoSQL

PostgreSQL:
- Relational data model.
- Transactions and strong integrity guarantees.
- Powerful querying and joins.

NoSQL:
- Several different database models with different strengths.
- May fit particular access patterns or scaling requirements.
- Does not automatically mean faster or more scalable.

Choose a database based on data relationships, access patterns, consistency
requirements, scale, and operational capabilities.

### Cache vs No Cache

Without caching:
- Simpler data flow.
- Database serves each applicable read.

With caching:
- Can reduce repeated database work and improve latency.
- Introduces freshness, invalidation, and failure-handling concerns.

Do not add a cache without a reason.

## 9. A System Design Thinking Process

For any system design problem:

1. Clarify functional requirements.
2. Define non-functional requirements and measurable targets.
3. Estimate users, traffic, storage, and peak load.
4. Draw a high-level architecture.
5. Identify components and their responsibilities.
6. Identify likely bottlenecks and failure points.
7. Compare possible solutions and their trade-offs.
8. Consider security, reliability, and data consistency.
9. Start with the simplest architecture that meets the requirements.
10. Measure actual behavior and evolve the design as needed.

## 10. Key Principles to Remember

- Start with requirements, not technology choices.
- Design for peak demand, not just average traffic.
- Horizontal scaling requires more than adding servers.
- A load balancer distributes work; it does not create capacity.
- Stateless application instances are easier to scale.
- High utilization can increase latency and reduce resilience.
- Every architectural choice has trade-offs.
- More components create more operational complexity.
- Measure before optimizing.
- Avoid premature optimization and unnecessary microservices.
- Consider failure scenarios as part of the initial design.

## Lesson 1 Summary

System design is about making informed architectural decisions based on
requirements, capacity, performance, reliability, and cost.

A good design is not necessarily the most sophisticated design.
It is the simplest design that meets the requirements and can evolve
as the system grows.