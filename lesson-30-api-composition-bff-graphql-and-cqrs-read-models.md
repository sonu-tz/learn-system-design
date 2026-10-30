# Lesson 30 — API Composition, Backend for Frontend (BFF), GraphQL, and CQRS Read Models

In the previous lessons, we explored microservices, event-driven architecture, and CQRS. Now let's look at a practical challenge: How do we efficiently serve data to frontend applications when that data belongs to multiple backend services?

Imagine a dispatcher dashboard that needs booking, passenger, driver, vehicle, payment, and live tracking information. Should the frontend call six different services? Should one backend aggregate everything? Should we introduce GraphQL or a dedicated read model?

This lesson explores those architectural choices and their trade-offs.

## 1. The Problem: Frontend Needs Data from Multiple Services

Consider a ride-booking platform with the following microservices:

- Booking Service
- Passenger Service
- Driver Service
- Vehicle Service
- Payment Service
- Tracking Service

Now imagine a dispatcher opens a booking details screen.

The screen needs:

```text
Booking Details
├── Booking ID
├── Booking Status
├── Pickup Location
├── Drop-off Location
├── Passenger Details
├── Driver Details
├── Vehicle Details
├── Payment Status
└── Latest Driver Location
```

These details belong to different services.

The frontend needs a way to retrieve and combine them.

### Option 1: Frontend calls every service

```text
                 Dispatcher UI
                       |
        +--------------+--------------+
        |       |      |      |       |
        v       v      v      v       v
     Booking Passenger Driver Payment Tracking
     Service  Service  Service Service Service
```

This approach is sometimes called client-side composition.

It may work for simple applications, but it introduces problems:

- More network requests from the browser.
- More complex frontend logic.
- Frontend must understand backend service boundaries.
- Authentication and authorization become harder to coordinate.
- One slow service can delay the entire screen.
- Changes to backend architecture can affect frontend code.

For a large application with several frontend clients, this becomes difficult to maintain.

### Option 2: Backend aggregates the data

Instead of making the frontend call every service, it calls one backend endpoint.

```text
              Dispatcher UI
                    |
                    v
             Booking Details API
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Booking   Passenger  Driver
       Service   Service    Service
          |         |         |
          +---------+---------+
                    |
                    v
              Combined Response
```

The backend retrieves the required data and returns a single response.

This is the foundation of API composition.

---

## 2. What Is API Composition?

API composition is a pattern where a component calls multiple services, combines their responses, and returns a unified result.

For example:

```http
GET /api/bookings/B123/details
```

The aggregation layer performs:

1. Get booking from Booking Service.
2. Get passenger from Passenger Service.
3. Get driver from Driver Service.
4. Get vehicle from Vehicle Service.
5. Get payment information from Payment Service.
6. Get the latest location from Tracking Service.
7. Combine the results.

Example response:

```json
{
  "bookingId": "B123",
  "status": "ASSIGNED",
  "pickup": "Delhi",
  "dropoff": "Noida",
  "passenger": {
    "id": "P123",
    "name": "Passenger A"
  },
  "driver": {
    "id": "D456",
    "name": "Driver A"
  },
  "vehicle": {
    "registrationNumber": "UP16AB1234",
    "type": "SEDAN"
  },
  "payment": {
    "status": "AUTHORIZED"
  },
  "tracking": {
    "latitude": 28.6139,
    "longitude": 77.2090
  }
}
```

The frontend receives one response instead of managing six separate requests.

### Advantages

- Simplifies frontend development.
- Hides internal service boundaries.
- Reduces browser network requests.
- Centralizes response transformation.
- Can improve user experience.

### Disadvantages

- Adds another backend component.
- Can become a performance bottleneck.
- May introduce runtime dependencies on many services.
- Requires timeout and partial-failure handling.
- Can become a centralized monolith if it accumulates business logic.

**Important:** API composition combines data. It should not become the owner of every business rule involved in the underlying services.

---

## 3. Sequential vs Parallel API Composition

How the aggregation layer calls downstream services matters.

Suppose five services each take approximately 100 ms to respond.

### Sequential execution

```text
Start
  |
  v
Booking       100 ms
  |
  v
Passenger     100 ms
  |
  v
Driver        100 ms
  |
  v
Payment       100 ms
  |
  v
Tracking      100 ms
  |
  v
Response
```

Approximate total latency:

\[
100 \times 5 = 500\text{ ms}
\]

### Parallel execution

If the requests are independent:

```text
              Start
                |
       +--------+--------+
       |        |        |
       v        v        v
    Booking  Passenger  Driver
       |        |        |
       +--------+--------+
                |
       +--------+--------+
       |                 |
       v                 v
     Payment          Tracking
       |                 |
       +--------+--------+
                |
             Response
```

Approximate latency:

\[
\max(100,100,100,100,100)=100\text{ ms}
\]

In practice, total latency also includes orchestration overhead, network time, serialization, and response processing.

### Important consideration

Parallel calls improve latency only when the operations are independent.

Some operations have dependencies.

For example:

```text
Get Booking
     |
     v
Get Assigned Driver ID
     |
     v
Get Driver Details
```

The second call cannot start until the first returns the driver ID.

### Concurrency limits

Do not blindly start hundreds of downstream requests in parallel.

Use bounded concurrency to avoid overwhelming downstream services.

---

## 4. Handling Partial Failures

Suppose the Booking Service and Driver Service respond successfully, but Tracking Service is unavailable.

Should the entire booking details screen fail?

It depends on the importance of the missing information.

Example:

| Service | Required data | Failure impact |
|---|---|---|
| Booking | Booking details | Critical |
| Passenger | Passenger details | Usually important |
| Driver | Driver details | Important |
| Payment | Payment status | Depends on screen |
| Tracking | Latest location | May be optional |

The aggregation layer can return partial data when the product requirements allow it.

Example:

```json
{
  "booking": {
    "id": "B123",
    "status": "ASSIGNED"
  },
  "driver": {
    "id": "D456",
    "name": "Driver A"
  },
  "tracking": null,
  "warnings": [
    "LIVE_LOCATION_TEMPORARILY_UNAVAILABLE"
  ]
}
```

This is graceful degradation.

However, partial responses should be explicit. Do not silently return incomplete data that the frontend interprets as valid.

### Useful techniques

- Timeouts per downstream dependency.
- Overall request deadlines.
- Bounded concurrency.
- Circuit breakers.
- Optional versus mandatory dependency classification.
- Cached fallback data where appropriate.
- Structured partial-response contracts.

A critical business operation should not proceed using incomplete data unless its business rules explicitly permit it.

---

## 5. What Is Backend for Frontend (BFF)?

Backend for Frontend (BFF) is an architectural pattern in which a backend layer is designed specifically for a particular frontend or client experience.

Different clients often have different requirements.

For example:

- Passenger Web App
- Passenger Mobile App
- Driver App
- Dispatcher Web App
- Admin Portal

A single generic API may not be ideal for all these clients.

### Without BFF

```text
Passenger Web -----+
                   |
Driver App --------+----> Generic API
                   |
Dispatcher Web ----+
```

The generic API may return large responses containing information that some clients do not need.

### With BFF

```text
Passenger Web
      |
      v
Passenger BFF
      |
      +------> Backend Services

Driver App
      |
      v
Driver BFF
      |
      +------> Backend Services

Dispatcher Web
      |
      v
Dispatcher BFF
      |
      +------> Backend Services
```

Each BFF optimizes the API contract for its frontend.

### Example

**Passenger BFF response:**

```json
{
  "bookingId": "B123",
  "bookingStatus": "ASSIGNED",
  "driverName": "Driver A",
  "driverEta": "5 minutes"
}
```

**Dispatcher BFF response:**

```json
{
  "bookingId": "B123",
  "passenger": {
    "name": "Passenger A",
    "phone": "REDACTED"
  },
  "driver": {
    "name": "Driver A",
    "status": "BUSY"
  },
  "vehicle": {
    "registrationNumber": "UP16AB1234"
  },
  "assignment": {
    "status": "ASSIGNED"
  },
  "payment": {
    "status": "AUTHORIZED"
  }
}
```

The dispatcher needs more operational information than the passenger.

### Advantages

- Client-specific response models.
- Less unnecessary data transfer.
- Frontend teams can evolve APIs independently.
- Client-specific aggregation and transformation.
- Better control over mobile and web performance.

### Disadvantages

- More backend components to maintain.
- Business logic can accidentally be duplicated.
- Multiple BFFs may create repeated downstream calls.
- Additional deployment and monitoring complexity.

### Important principle

A BFF should contain presentation-oriented orchestration and transformation, not duplicate core business rules.

For example, calculating whether a driver is eligible for assignment belongs in Dispatch Service, not Dispatcher BFF.

---

## 6. API Gateway vs BFF vs API Composition

These concepts are related but solve different problems.

| Concept | Primary responsibility |
|---|---|
| API Gateway | Entry point, routing, security, rate limiting |
| API Composition | Combining data from multiple services |
| BFF | Providing APIs tailored to a specific frontend |
| CQRS Read Model | Storing query-optimized data |
| GraphQL | Client-driven data selection through a query API |

They can work together.

Example:

```text
                Dispatcher UI
                      |
                      v
                 API Gateway
                      |
                      v
                Dispatcher BFF
                      |
            +---------+---------+
            |         |         |
            v         v         v
         Booking   Driver    Payment
         Service   Service   Service
```

The API Gateway handles common entry-point concerns.

The BFF handles dispatcher-specific aggregation.

The underlying services remain responsible for their business capabilities.

---

## 7. What Is GraphQL?

GraphQL is a query language and runtime for APIs that allows clients to request the specific fields they need.

It was designed to address some limitations of fixed REST response structures.

### REST example

Suppose the frontend needs:

- Booking ID.
- Booking status.
- Driver name.

A REST endpoint might return:

```http
GET /api/bookings/B123
```

Response:

```json
{
  "id": "B123",
  "status": "ASSIGNED",
  "passenger": {
    "id": "P456",
    "name": "Passenger A",
    "email": "passenger@example.com"
  },
  "driver": {
    "id": "D789",
    "name": "Driver A",
    "phone": "0000000000",
    "vehicle": {
      "registrationNumber": "UP16AB1234"
    }
  },
  "createdAt": "2026-10-30T10:30:00Z"
}
```

The frontend receives fields it may not need.

This can be an example of over-fetching.

### GraphQL example

The client requests only the fields it needs:

```graphql
query BookingDetails {
  booking(id: "B123") {
    id
    status
    driver {
      name
    }
  }
}
```

Response:

```json
{
  "data": {
    "booking": {
      "id": "B123",
      "status": "ASSIGNED",
      "driver": {
        "name": "Driver A"
      }
    }
  }
}
```

The client controls the shape of the requested data within the schema's available fields.

---

## 8. GraphQL Architecture

A GraphQL API typically contains:

- Schema.
- Types.
- Queries.
- Mutations.
- Resolvers.

### Schema

The schema defines which types and operations are available.

Example:

```graphql
type Booking {
  id: ID!
  status: String!
  driver: Driver
  passenger: Passenger
}

type Driver {
  id: ID!
  name: String!
}

type Passenger {
  id: ID!
  name: String!
}

type Query {
  booking(id: ID!): Booking
}
```

### Query

A query retrieves data.

```graphql
query {
  booking(id: "B123") {
    id
    status
  }
}
```

### Mutation

A mutation changes data.

```graphql
mutation {
  cancelBooking(id: "B123") {
    id
    status
  }
}
```

### Resolver

A resolver implements how a field is retrieved or calculated.

Conceptually:

```text
booking(id)
    |
    v
Booking Resolver
    |
    v
Booking Service
    |
    v
Booking Data
```

The `driver` field might use another resolver to retrieve driver information.

---

## 9. GraphQL and Microservices

GraphQL can act as a unified API layer over multiple backend services.

```text
                  Frontend
                     |
                     v
                GraphQL API
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Booking    Driver     Payment
       Service    Service    Service
```

The frontend sends one GraphQL query.

The GraphQL server coordinates the necessary backend calls.

### Benefits

- Flexible response shapes.
- Reduced over-fetching.
- Potentially fewer client round trips.
- Strongly defined API schema.
- Useful for applications with many different frontend requirements.

### Important limitation

One GraphQL request does not automatically mean one backend request.

A query that requests booking, driver, payment, and vehicle data may still result in several internal service calls.

GraphQL solves the client-facing query interface problem. It does not automatically solve backend aggregation or performance problems.

---

## 10. GraphQL N+1 Query Problem

One common GraphQL performance issue is the N+1 query problem.

Suppose the frontend requests ten bookings, including each booking's driver.

```graphql
query {
  bookings {
    id
    driver {
      name
    }
  }
}
```

A naive resolver implementation may execute:

1. One query to retrieve all ten bookings.
2. One driver query for each booking.

Total:

\[
1 + 10 = 11 \text{ queries}
\]

If there are 1,000 bookings, this can become:

\[
1 + 1000 = 1001 \text{ queries}
\]

### Solution: DataLoader and batching

Instead of retrieving each driver independently, collect the driver IDs and fetch them together.

```text
Bookings Query
      |
      v
Get 1000 bookings
      |
      v
Collect Driver IDs
      |
      v
Batch Driver Lookup
      |
      v
Return Combined Result
```

This reduces the number of backend calls.

Other useful techniques:

- Request-scoped caching.
- Query depth limits.
- Query complexity limits.
- Pagination.
- Bounded resolver concurrency.
- Monitoring expensive queries.

DataLoader-style batching is particularly useful when multiple resolvers request overlapping data.

---

## 11. GraphQL Security and Performance

GraphQL provides flexibility, but that flexibility creates additional risks.

### Query complexity

A client might request deeply nested data.

```graphql
query {
  bookings {
    driver {
      bookings {
        driver {
          bookings {
            id
          }
        }
      }
    }
  }
}
```

Without controls, expensive queries can consume substantial backend resources.

### Important protections

**Depth limiting:** Restrict nested query depth.

**Complexity limiting:** Estimate the cost of a query and reject overly expensive operations.

**Pagination:** Avoid returning unlimited collections.

**Rate limiting:** Apply limits based on user, tenant, API key, or other relevant dimensions.

**Authorization:** Validate access to every relevant object and field.

**Timeouts:** Prevent expensive resolvers from running indefinitely.

**Persisted queries:** Where suitable, allow approved queries to be registered and reused.

### Important principle

GraphQL does not automatically provide object-level authorization.

For example, knowing a booking ID must not allow one passenger to access another passenger's booking.

Authorization must be enforced by the API and underlying services.

---

## 12. GraphQL vs REST

| Area | REST | GraphQL |
|---|---|---|
| Data selection | Defined by endpoint | Selected by client |
| Response shape | Usually fixed | Flexible |
| Endpoints | Multiple resource endpoints | Often one query endpoint |
| Caching | HTTP caching is straightforward for GET | Requires additional strategy |
| Versioning | Often URL/header versioning | Often schema evolution |
| File uploads | Established HTTP patterns | Usually handled separately |
| Complexity | Relatively simple | Query execution can be complex |
| N+1 problem | Possible | Common resolver concern |
| Best fit | Resource-oriented APIs | Flexible, connected data requirements |

### When REST is a good choice

- Simple resource-based APIs.
- Standard CRUD operations.
- Straightforward HTTP caching.
- External integrations.
- Services with stable response requirements.

### When GraphQL is a good choice

- Multiple frontend clients need different data shapes.
- Screens require connected data from many domains.
- Frontend teams need more control over field selection.
- A well-managed schema and query execution layer can be maintained.

Neither is universally better.

GraphQL should be introduced to solve a real API problem, not simply because it is newer.

---

## 13. CQRS Read Models vs API Composition

Both approaches combine information from multiple sources, but their architectures differ.

### API Composition

Data is fetched at request time.

```text
Client
  |
  v
Aggregation API
  |
  +----> Booking Service
  |
  +----> Driver Service
  |
  +----> Payment Service
  |
  v
Combined Response
```

The response reflects what the services return at that moment.

### CQRS Read Model

Data is combined before the client requests it.

```text
Booking Events ----+
                   |
Driver Events -----+----> Projection Consumer
                   |              |
Payment Events ----+              v
                             Read Database
                                  |
                                  v
                                Client
```

The read database is updated asynchronously as events arrive.

### Comparison

| Area | API Composition | CQRS Read Model |
|---|---|---|
| Data retrieval | At request time | Precomputed |
| Latency | Depends on downstream services | Usually lower |
| Freshness | Depends on source responses | May be stale |
| Runtime dependencies | Multiple services | Primarily read database |
| Complexity | Moderate | Higher |
| Good fit | Low-to-moderate query complexity | High-volume or complex queries |

### Example

For a dispatcher dashboard with thousands of bookings, repeatedly calling five services for every screen refresh may become expensive.

A dedicated read model may be more efficient.

For an occasional booking details request, API composition may be simpler.

Choose based on access patterns and performance requirements.

---

## 14. API Composition vs Data Federation

API composition retrieves data through service APIs and combines it in an application layer.

Data federation is a broader approach that presents data from multiple sources through a unified query interface.

For example, a federated GraphQL architecture can expose fields backed by different services while presenting one unified schema.

However, federation does not remove the need for:

- Clear service ownership.
- Authorization.
- Timeouts.
- Query planning.
- Performance controls.
- Failure handling.

A unified API surface does not mean the underlying services have become one database.

---

## 15. Caching Aggregated API Responses

Aggregated endpoints can be expensive because they depend on several downstream services.

Caching can reduce repeated work.

Example:

```text
Client
  |
  v
Booking Details API
  |
  v
Cache
  |
  +---- Cache Hit ---> Return Response
  |
  +---- Cache Miss --> Call Services
                         |
                         v
                    Build Response
                         |
                         v
                     Update Cache
```

### Cache considerations

- Cache key must include tenant and authorization scope where relevant.
- Different users may have different access permissions.
- Sensitive information should not leak across users.
- TTL should reflect data freshness requirements.
- Invalidation may be required after important events.
- Partial responses should not accidentally be cached as complete responses.

For example, a public vehicle-type catalogue may be safely cached longer than live driver location.

Do not apply the same cache policy to every field in an aggregated response.

---

## 16. Multi-Tenant Considerations

In a multi-tenant application, every aggregation and query layer must preserve tenant boundaries.

Suppose the dispatcher requests:

```http
GET /api/bookings/B123/details
```

The aggregation layer must verify:

1. Which tenant owns the booking?
2. Does the authenticated user belong to that tenant?
3. Does the user have permission to access this booking?
4. Are downstream service calls scoped correctly?

A dangerous design is to validate tenant access only in the frontend or only at the gateway.

Authorization must be enforced at the appropriate backend trust boundaries.

### Cache key example

Unsafe:

```text
booking:B123
```

Safer, where tenant-scoped caching is required:

```text
tenant:T001:booking:B123
```

The cache key is not a substitute for authorization. It is an additional isolation measure.

---

## 17. Worked Example: Dispatcher Dashboard Architecture

Let's design a dispatcher dashboard that needs to display 500 active bookings.

### Requirements

- Show booking status.
- Show passenger and driver information.
- Show vehicle details.
- Display payment status.
- Show latest driver location.
- Support filters and pagination.
- Update important information in near real time.

### Option A: API Composition

```text
                 Dispatcher UI
                       |
                       v
                Dispatcher BFF
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
    Booking          Driver          Payment
    Service          Service         Service
       |               |               |
       +---------------+---------------+
                       |
                       v
                 Combined Response
```

This is relatively straightforward.

However, for 500 bookings, the BFF may need to retrieve large datasets and coordinate many downstream operations.

### Option B: CQRS Read Model

```text
Booking Service ------+
                      |
Driver Service -------+
                      |
Payment Service ------+----> Events
                      |         |
Tracking Service -----+         v
                              Broker
                                |
                                v
                        Projection Consumer
                                |
                                v
                         Dispatcher Read DB
                                |
                                v
                          Dispatcher BFF
                                |
                                v
                           Dispatcher UI
```

The projection consumer maintains a dispatcher-optimized view.

Example:

```json
{
  "bookingId": "B123",
  "bookingStatus": "ASSIGNED",
  "passengerName": "Passenger A",
  "driverName": "Driver A",
  "vehicleType": "SEDAN",
  "paymentStatus": "AUTHORIZED",
  "driverLocation": {
    "latitude": 28.6139,
    "longitude": 77.2090
  },
  "lastUpdatedAt": "2026-10-30T10:30:00Z"
}
```

### Recommended design considerations

- Use a paginated read model for the main booking list.
- Keep high-frequency location updates in a suitable tracking store or real-time pipeline.
- Do not rebuild the entire dispatcher projection for every location update.
- Update only the relevant booking or driver record.
- Use a real-time delivery mechanism for changes that need immediate UI updates.
- Define acceptable staleness separately for booking status, payment status, and location.

A single screen does not necessarily need one storage model for every data type.

---

## 18. Common Mistakes

1. Making the frontend call every microservice directly.
2. Building a BFF that contains core business logic.
3. Calling downstream services sequentially when calls are independent.
4. Starting unlimited parallel requests.
5. Failing the entire response when optional data is unavailable.
6. Assuming GraphQL automatically reduces backend calls.
7. Ignoring the GraphQL N+1 problem.
8. Allowing unlimited query depth or complexity.
9. Introducing CQRS without a genuine read/write workload difference.
10. Using a read model without defining acceptable staleness.
11. Caching aggregated data without tenant isolation.
12. Duplicating business rules across multiple BFFs.
13. Treating API Gateway, BFF, and API composition as interchangeable concepts.

---

## 19. Practical Architecture Decision Checklist

Before choosing an approach, consider:

### Frontend requirements

- [ ] Do multiple frontend clients require different response shapes?
- [ ] Does the screen need data from multiple services?
- [ ] Is client-side composition becoming complicated?

### Performance

- [ ] Are downstream calls independent?
- [ ] Can calls run in parallel?
- [ ] Is bounded concurrency required?
- [ ] Would a materialized read model reduce repeated work?
- [ ] Are pagination and filtering supported?

### Reliability

- [ ] Are downstream timeouts defined?
- [ ] Can optional fields be degraded gracefully?
- [ ] Is there a clear overall request deadline?
- [ ] Are circuit breakers and fallbacks appropriate?

### Security

- [ ] Is object-level authorization enforced?
- [ ] Are tenant boundaries preserved?
- [ ] Are cache keys properly scoped?
- [ ] Does every resolver or aggregation path enforce access rules?

### Architecture

- [ ] Is API composition sufficient?
- [ ] Would a BFF provide meaningful client-specific value?
- [ ] Is GraphQL justified?
- [ ] Is a CQRS read model worth the extra complexity?

---

## 20. Revision Questions

1. What is API composition?
2. Why is frontend-side composition sometimes problematic?
3. What is the difference between sequential and parallel aggregation?
4. What is partial failure handling?
5. What is Backend for Frontend?
6. How is a BFF different from an API Gateway?
7. What is GraphQL?
8. What are GraphQL queries, mutations, and resolvers?
9. Does one GraphQL request always mean one backend request?
10. What is the GraphQL N+1 problem?
11. How does DataLoader-style batching help?
12. What are GraphQL query depth and complexity limits?
13. What are the differences between REST and GraphQL?
14. How is API composition different from a CQRS read model?
15. When should you use a materialized read model?
16. What is data federation?
17. What are the risks of caching aggregated API responses?
18. How should multi-tenant authorization be handled?
19. When should you choose a BFF?
20. Can API composition, GraphQL, and CQRS work together?

---

## 21. Practical Exercise

Design the API architecture for a large dispatcher application.

The application has three frontend clients:

- Dispatcher Web.
- Passenger Mobile.
- Driver Mobile.

The backend has these services:

- Booking.
- Driver.
- Passenger.
- Payment.
- Tracking.
- Notification.

### Requirements

**Dispatcher Web:**
- Displays hundreds of active bookings.
- Supports advanced filtering.
- Shows driver and payment information.
- Requires frequent updates.

**Passenger Mobile:**
- Displays only the passenger's own bookings.
- Shows booking status, driver details, and ETA.

**Driver Mobile:**
- Displays assigned trips.
- Updates driver location.
- Receives assignment notifications.

Design a solution that answers:

1. Would you create separate BFFs for each client?
2. Which operations should use REST?
3. Would GraphQL be useful for any client?
4. Would you build a CQRS read model for Dispatcher Web?
5. How would you handle high-frequency location updates?
6. How would you handle partial failures in the dispatcher screen?
7. How would you prevent data leakage between tenants?
8. How would you optimize requests that need data from multiple services?
9. Which data would you cache, and for how long?
10. How would you monitor aggregation latency and downstream service failures?

**Key takeaway:** API composition, BFF, GraphQL, and CQRS read models are different ways to solve data-access problems. Choose the simplest approach that satisfies the frontend's performance, consistency, security, and maintainability requirements.

### Practice focus

Design the three frontend API experiences independently. Pay particular attention to why a dispatcher dashboard may benefit from a CQRS read model while a passenger mobile screen may be perfectly served by a simpler BFF.