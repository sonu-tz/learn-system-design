# Lesson 16 — API Design, API Gateway, and Communication Patterns

In the previous lesson, we designed a ride-booking workflow using multiple services, message queues, Sagas, and compensating transactions.

Now let's focus on how these services communicate.

Imagine a ride-booking platform with Passenger, Driver, Ride, Payment, and Notification services. How should they communicate? Should every request use REST? When should we use gRPC? When should we introduce an API Gateway? How do we handle API versioning, rate limiting, pagination, and backward compatibility?

These are important decisions in real-world system design and interviews.

## 1. What Is API Design?

An API (Application Programming Interface) defines how two software components communicate.

In a distributed system, APIs establish contracts between:

- Frontend applications and backend services.
- Backend services and other backend services.
- External applications and your platform.
- Services and third-party providers.

For example, a passenger application might call:

```http
POST /api/v1/rides
```

The Ride Service receives the request, validates it, creates a ride, and returns a response.

A good API design should provide:

- Clear resource names.
- Predictable request and response structures.
- Consistent error handling.
- Authentication and authorization.
- Versioning and backward compatibility.
- Pagination and filtering.
- Rate limiting.
- Idempotency where required.

**Important:** An API is a contract, not just an HTTP endpoint.

---

## 2. Communication Patterns in Distributed Systems

There are two major communication patterns.

1. Synchronous communication.
2. Asynchronous communication.

### 2.1 Synchronous communication

The caller sends a request and waits for a response.

Example:

```text
Passenger App
      |
      | POST /rides
      ▼
   Ride Service
      |
      | Response
      ▼
Passenger App
```

Common technologies:

- REST over HTTP.
- gRPC.
- GraphQL over HTTP.

Example:

A passenger requests their ride history.

```http
GET /api/v1/passengers/101/rides
```

The application waits for the server to return the ride list.

**Advantages**

- Simple request-response model.
- Easy to understand.
- Immediate result is available.
- Suitable for queries and operations requiring immediate confirmation.

**Disadvantages**

- Caller depends on the availability of the downstream service.
- Network latency increases response time.
- Slow downstream services can cause timeouts.
- Failure can propagate across service dependencies.

### 2.2 Asynchronous communication

The caller sends a message or event and does not wait for the complete downstream workflow.

Example:

```text
Ride Service
     |
     | RideCreated
     ▼
 Message Broker
     |
     ├──────────────┐
     ▼              ▼
Notification     Analytics
Service          Service
```

Common technologies:

- RabbitMQ.
- Apache Kafka.
- AWS SQS.
- Google Pub/Sub.

Example:

After creating a ride, the Ride Service publishes `RideCreated`.

Notification Service sends notifications, while Analytics Service updates reporting data.

The Ride Service does not need to wait for either service to finish.

**Advantages**

- Reduces direct service dependency.
- Supports background processing.
- Helps absorb traffic spikes.
- Allows consumers to scale independently.

**Disadvantages**

- Results may not be immediately available.
- Eventual consistency must be handled.
- Duplicate messages and retries require attention.
- Debugging workflows can be more complicated.

### Synchronous vs. asynchronous

| Consideration | Synchronous | Asynchronous |
|---|---|---|
| Response | Immediate response expected | Result may arrive later |
| Coupling | Caller depends on response | Reduced runtime dependency |
| Latency | Includes downstream response time | Caller can return earlier |
| Failure handling | Timeout and retry | Retry, acknowledgement, DLQ |
| Best fit | Queries and immediate decisions | Notifications, analytics, background jobs |

**Rule:** Use synchronous communication when the caller needs an immediate result. Use asynchronous communication when work can happen independently or later.

---

## 3. REST API Design

REST (Representational State Transfer) is an architectural style commonly used to design HTTP APIs.

It models application data as resources.

For example:

- `/rides`
- `/drivers`
- `/passengers`
- `/payments`

### HTTP methods

| Method | Purpose | Example |
|---|---|---|
| GET | Retrieve a resource | `GET /rides/501` |
| POST | Create a resource or initiate an operation | `POST /rides` |
| PUT | Replace a resource representation | `PUT /passengers/101` |
| PATCH | Partially update a resource | `PATCH /rides/501` |
| DELETE | Delete a resource | `DELETE /passengers/101` |

### Resource-oriented API design

Prefer:

```http
GET /api/v1/rides/501
```

Instead of:

```http
GET /api/v1/getRideById?id=501
```

Prefer:

```http
GET /api/v1/passengers/101/rides
```

Instead of:

```http
GET /api/v1/getPassengerRides?passengerId=101
```

Resource-oriented naming makes APIs easier to understand and maintain.

### Nested resources

For relationships:

```http
GET /api/v1/rides/501/payments
```

This retrieves payments associated with ride 501.

However, avoid excessively deep nesting.

For example:

```http
GET /passengers/101/rides/501/drivers/201/locations
```

This may be unnecessarily complicated if the driver location is independently accessible.

### When to use action endpoints

Not every business operation maps naturally to CRUD.

For example, assigning a driver is a business command:

```http
POST /api/v1/rides/501/assignments
```

Cancelling a ride might be:

```http
POST /api/v1/rides/501/cancellation
```

These are reasonable designs when they represent explicit domain operations.

Do not force every operation into a generic PATCH request just to follow a simplistic REST convention.

---

## 4. HTTP Status Codes

Status codes communicate the result of an HTTP request.

### Success responses

| Status | Meaning | Example |
|---|---|---|
| 200 | OK | Ride retrieved successfully |
| 201 | Created | Ride created |
| 202 | Accepted | Background workflow accepted |
| 204 | No Content | Successful deletion without response body |

### Client errors

| Status | Meaning | Example |
|---|---|---|
| 400 | Bad Request | Invalid request format |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | User lacks permission |
| 404 | Not Found | Ride does not exist |
| 409 | Conflict | Ride already assigned |
| 422 | Unprocessable Content | Valid request format but business validation fails |
| 429 | Too Many Requests | Rate limit exceeded |

### Server errors

| Status | Meaning | Example |
|---|---|---|
| 500 | Internal Server Error | Unexpected application failure |
| 502 | Bad Gateway | Invalid response from upstream |
| 503 | Service Unavailable | Service temporarily unavailable |
| 504 | Gateway Timeout | Upstream response timed out |

The exact choice between 400 and 422, for example, depends on the API's documented conventions. Consistency matters.

---

## 5. API Request and Response Design

Use predictable response structures.

Example request:

```http
POST /api/v1/rides
Content-Type: application/json
Authorization: Bearer <token>
```

```json
{
  "pickup": {
    "latitude": 28.61,
    "longitude": 77.23
  },
  "dropoff": {
    "latitude": 28.62,
    "longitude": 77.24
  }
}
```

Example successful response:

```http
HTTP/1.1 201 Created
```

```json
{
  "id": "ride-501",
  "status": "requested",
  "createdAt": "2026-10-30T10:30:00Z"
}
```

For errors, provide a consistent structure:

```json
{
  "error": {
    "code": "RIDE_ALREADY_ASSIGNED",
    "message": "This ride has already been assigned.",
    "requestId": "req-123"
  }
}
```

### Why stable error codes matter

Frontend applications should not need to parse human-readable error messages to determine what happened.

A stable error code allows the client to handle specific business conditions reliably.

For example:

```text
RIDE_ALREADY_ASSIGNED
DRIVER_UNAVAILABLE
PAYMENT_DECLINED
INVALID_PICKUP_LOCATION
```

Messages can change or be localized without changing the underlying error code.

---

## 6. Idempotency in APIs

An API is idempotent if repeating the same operation has the same intended effect as performing it once.

This is particularly important when clients retry after network timeouts.

### Example: Ride creation

Suppose a passenger clicks Book Ride.

1. Client sends a POST request.
2. Server creates the ride.
3. Response is lost.
4. Client retries.
5. Server receives the request again.

Without protection, two rides might be created.

### Idempotency key

The client sends a unique key:

```http
POST /api/v1/rides
Idempotency-Key: booking-abc-123
```

The server stores the key and its associated result.

On a retry with the same key:

- If the operation completed, return the original result.
- If it is still processing, return an appropriate in-progress response.
- If the same key is reused with a different request payload, reject it according to the API contract.

### Important details

- Scope keys appropriately, such as by customer or account.
- Store a request fingerprint or payload hash.
- Handle concurrent requests using the same key.
- Define key retention and expiration.
- Make the database operation and idempotency record safe against partial failure.

**Remember:** POST is not automatically idempotent. An API can implement idempotency explicitly.

---

## 7. API Pagination

Imagine a passenger has 10,000 rides.

Returning every ride in one API response is inefficient.

It increases:

- Response size.
- Network transfer time.
- Database work.
- Memory usage.
- Frontend rendering cost.

Pagination limits the amount of data returned.

### 7.1 Offset pagination

Example:

```http
GET /api/v1/rides?limit=20&offset=40
```

This skips 40 rows and returns the next 20.

Conceptually:

```sql
SELECT *
FROM rides
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

**Advantages**

- Simple to implement.
- Easy to navigate to a particular page.
- Suitable for smaller datasets.

**Disadvantages**

- Large offsets can become expensive.
- New records inserted between requests can shift page boundaries.
- Records may be duplicated or skipped during changing datasets.

### 7.2 Cursor pagination

Instead of specifying an offset, the client provides a cursor representing the last item from the previous page.

Example:

```http
GET /api/v1/rides?limit=20&cursor=ride-480
```

The server retrieves records after the cursor.

A robust cursor is generally based on a stable ordering key.

For example:

```sql
SELECT *
FROM rides
WHERE passenger_id = 101
  AND (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

The combination of `created_at` and `id` provides a deterministic ordering when timestamps are equal.

**Advantages**

- Better suited to large datasets.
- Avoids scanning and skipping large offsets.
- More stable when new records are inserted.

**Disadvantages**

- More complicated to implement.
- Arbitrary page-number navigation is difficult.
- Requires a stable ordering and cursor design.

### Offset vs. cursor

| Consideration | Offset | Cursor |
|---|---|---|
| Simplicity | Easier | More complex |
| Large datasets | Can become expensive | Generally more efficient |
| New records during pagination | Can shift results | More stable with suitable ordering |
| Jump to page | Easy | Difficult |
| Best fit | Small or moderate lists | Large, frequently changing datasets |

---

## 8. API Versioning

APIs evolve.

Imagine version 1 returns:

```json
{
  "id": "ride-501",
  "status": "completed"
}
```

Later, the application needs to return a more detailed status representation.

You may introduce a new version:

```http
GET /api/v2/rides/501
```

### Common versioning strategies

**URL versioning**

```http
/api/v1/rides
/api/v2/rides
```

Simple and easy to understand.

**Header versioning**

```http
Accept: application/vnd.example.v2+json
```

Keeps the URL stable but requires more client and server handling.

**Query parameter versioning**

```http
/api/rides?version=2
```

Easy to implement but can make caching and documentation more complicated.

### Backward compatibility

A change is backward-compatible when existing clients continue working without modification.

Generally safer changes:

- Adding an optional response field.
- Adding a new endpoint.
- Adding an optional query parameter.
- Adding a new resource.

Potentially breaking changes:

- Removing a field.
- Renaming a field.
- Changing a field's type.
- Changing the meaning of an existing field.
- Making an optional request field mandatory.
- Changing response structure.

### Expand-and-contract API migration

1. Add the new field or endpoint.
2. Keep supporting the old contract.
3. Migrate clients.
4. Monitor usage of the old contract.
5. Deprecate and eventually remove it according to a published timeline.

Do not remove an API version simply because the new version has been deployed.

---

## 9. API Gateway

In a microservices architecture, clients should not necessarily need to know every internal service address.

An API Gateway provides a common entry point.

### Without an API Gateway

```text
Passenger App
    |
    ├────> Ride Service
    ├────> Driver Service
    ├────> Payment Service
    └────> Profile Service
```

The client needs to understand multiple service endpoints.

### With an API Gateway

```text
                  Passenger App
                        |
                        ▼
                  API Gateway
                        |
          ┌─────────────┼─────────────┐
          |             |             |
          ▼             ▼             ▼
      Ride Service  Driver Service Payment Service
```

The gateway routes requests to the appropriate backend service.

### Common responsibilities

1. Request routing.
2. Authentication.
3. Rate limiting.
4. TLS termination.
5. Request and response transformation.
6. Logging and tracing.
7. API version routing.
8. CORS handling.
9. Sometimes response aggregation.

### What should not live in the gateway?

Avoid putting complex business logic in the gateway.

For example, the gateway should not independently decide whether a driver can be assigned to a ride.

That business rule belongs to the service that owns the assignment operation.

### API Gateway vs. Load Balancer

| API Gateway | Load Balancer |
|---|---|
| Understands API routing and policies | Distributes traffic across targets |
| Can perform authentication and rate limiting | Typically focuses on traffic distribution and health |
| Can transform requests and responses | Usually does not implement API business policies |
| Routes to different services | Routes to healthy instances or targets |

They can work together.

A common architecture is:

```text
Client
  |
  ▼
Load Balancer
  |
  ▼
API Gateway
  |
  ├── Ride Service
  ├── Driver Service
  └── Payment Service
```

In practice, some gateways include load-balancing capabilities, and some load balancers provide API-aware features.

---

## 10. Backend for Frontend (BFF)

Different clients may need different API representations.

For example:

- Passenger mobile app.
- Driver mobile app.
- Dispatcher web dashboard.

A single generic API might return too much data or require many separate calls.

The Backend for Frontend pattern introduces a dedicated backend layer for each client type.

```text
              API Gateway
                   |
          ┌────────┼─────────┐
          |        |         |
          ▼        ▼         ▼
     Passenger   Driver   Dispatcher
        BFF        BFF       BFF
          |        |         |
          └────────┼─────────┘
                   |
             Backend Services
```

### Example

The dispatcher dashboard may need:

- Ride status.
- Driver details.
- Live location.
- Assignment history.
- Operational alerts.

The passenger app may need only:

- Current ride.
- Driver name.
- Driver location.
- ETA.

A BFF can aggregate and shape these responses for the relevant client.

### Advantages

- Client-specific response formats.
- Fewer frontend network requests.
- Independent evolution of client APIs.
- Keeps frontend-specific aggregation out of core services.

### Disadvantages

- More services to maintain.
- Potential duplicated aggregation logic.
- Additional network hops.
- More operational complexity.

Use BFF when client requirements differ significantly, not automatically for every application.

---

## 11. REST vs. gRPC vs. GraphQL

These technologies solve different API communication needs.

### REST

Uses HTTP methods and resource-oriented endpoints.

Best for:

- Public APIs.
- Browser and mobile applications.
- CRUD operations.
- Broad interoperability.

### gRPC

Uses Protocol Buffers and HTTP/2 in its common implementation.

It supports:

- Unary request-response calls.
- Server streaming.
- Client streaming.
- Bidirectional streaming.

Example service definition:

```protobuf
service DriverService {
  rpc GetDriver (DriverRequest) returns (DriverResponse);
}
```

Best for:

- Internal service-to-service communication.
- Strongly defined contracts.
- Efficient binary serialization.
- Streaming communication.

Trade-offs:

- Browser integration can require gRPC-Web or another adaptation.
- Debugging raw payloads is less convenient than JSON.
- It requires tooling and schema management.

### GraphQL

Allows clients to request specific fields through a query.

Example:

```graphql
query {
  ride(id: "ride-501") {
    id
    status
    driver {
      name
      location {
        latitude
        longitude
      }
    }
  }
}
```

Best for:

- Applications with complex and varying data requirements.
- Frontends needing flexible response shapes.
- Aggregating data from multiple sources.

Trade-offs:

- Query complexity and resource limits need careful management.
- Caching can be more complicated than conventional REST.
- Poor resolver design can cause N+1 database queries.

### Comparison

| Consideration | REST | gRPC | GraphQL |
|---|---|---|---|
| Data format | Usually JSON | Protocol Buffers | Usually JSON |
| Contract | HTTP resources and schemas | `.proto` definitions | GraphQL schema |
| Main use | Public and general-purpose APIs | Internal service communication | Flexible client queries |
| Streaming | Not native to ordinary REST request-response | Built-in streaming patterns | Subscriptions supported by implementations |
| Browser support | Excellent | Often needs adaptation | Excellent |
| Caching | Standard HTTP caching works well | Requires other strategies | More involved |

No technology is universally best. Select based on clients, traffic, latency, contracts, and operational requirements.

---

## 12. Rate Limiting

Rate limiting controls how frequently a client can make requests.

Suppose a passenger sends 500 requests per minute to an endpoint.

This could be accidental or malicious.

Without rate limiting, the service may become overloaded.

### Common algorithms

**Fixed Window Counter**

Allow 100 requests per minute.

Simple, but a client could send 100 requests at the end of one window and another 100 at the beginning of the next.

**Sliding Window Log**

Track timestamps of requests and count those within the active time window.

More accurate but may require substantial storage.

**Sliding Window Counter**

Approximate the sliding-window request count using adjacent fixed windows.

Provides a balance between accuracy and efficiency.

**Token Bucket**

Tokens are added at a fixed rate up to a maximum capacity.

Each request consumes a token.

Example:

- Bucket capacity: 100 tokens.
- Refill rate: 10 tokens/second.
- A client can make a burst of up to 100 requests.
- Once depleted, it can make requests as tokens refill.

**Leaky Bucket**

Requests enter a queue and are processed at a controlled rate.

This smooths traffic but may introduce queueing latency.

### Where to apply rate limits

- Per IP address.
- Per user.
- Per API key.
- Per tenant.
- Per endpoint.
- Per service.

For multi-tenant systems, per-tenant limits are often important so one tenant cannot consume the entire system's capacity.

### Response

When a limit is exceeded:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

The client should respect retry guidance rather than immediately sending another request.

---

## 13. API Security

API design must account for security from the beginning.

### Authentication

Authentication answers:

"Who is making this request?"

Common approaches:

- Session-based authentication.
- OAuth 2.0 / OpenID Connect.
- Access tokens.
- API keys for appropriate machine-to-machine use cases.

### Authorization

Authorization answers:

"What is this user allowed to do?"

Example:

A passenger should not be able to retrieve another passenger's private ride details merely by changing the ride ID in the URL.

This is an object-level authorization problem.

### Additional protections

- TLS for network communication.
- Input validation.
- Output filtering.
- Rate limiting.
- Request size limits.
- Protection against injection attacks.
- Audit logging for sensitive operations.
- Secure token storage and expiration handling.

Never assume that authentication alone means the caller is authorized to access every requested resource.

---

## 14. API Observability

A distributed API request can pass through multiple services.

Without tracing, it can be difficult to identify where latency or failure occurred.

Example:

```text
Passenger App
     |
     ▼
API Gateway       15 ms
     |
     ▼
Ride Service      30 ms
     |
     ▼
Driver Service    80 ms
     |
     ▼
Database          25 ms
```

Total approximate latency: 150 ms, excluding any unlisted overhead.

### Important observability fields

- Request ID.
- Trace ID.
- Span ID.
- User or tenant context where safe and appropriate.
- Endpoint.
- HTTP status code.
- Request duration.
- Downstream service duration.
- Error category.

Use distributed tracing to connect calls across service boundaries.

Do not log passwords, access tokens, payment credentials, or unnecessary personal data.

### Metrics to monitor

- Request rate.
- Error rate.
- P50, P95, and P99 latency.
- Rate-limit rejections.
- Timeout counts.
- Downstream dependency failures.
- Request sizes.
- Queue or asynchronous workflow delays.

---

## 15. End-to-End Architecture Example

Let's design a simple ride-booking API architecture.

```text
                 Passenger App
                       |
                       ▼
                 Load Balancer
                       |
                       ▼
                  API Gateway
                       |
                Authentication
                       |
                       ▼
                  Ride Service
                       |
              ┌────────┴─────────┐
              |                  |
              ▼                  ▼
          PostgreSQL       Message Broker
              |                  |
          Ride + Outbox          |
                                 ▼
                          Driver Service
                                 |
                              Driver DB
                                 |
                                 ▼
                          Payment Service
                                 |
                              Payment DB
                                 |
                                 ▼
                         Notification Service
```

### Request flow

1. Passenger sends `POST /api/v1/rides`.
2. Gateway validates authentication and applies rate limits.
3. Ride Service validates the request.
4. Ride Service creates the ride and outbox event in one transaction.
5. API returns `201 Created` with the ride ID and initial status.
6. Outbox worker publishes the event.
7. Driver Service processes the request.
8. Payment and notification workflows continue asynchronously.
9. Passenger checks ride status through `GET /api/v1/rides/{id}`.

The passenger receives an immediate acknowledgement without waiting for every downstream operation.

The API contract should make it clear that the ride is initially pending, not necessarily confirmed.

---

## 16. Common Mistakes

1. Using synchronous communication for every service interaction.
2. Returning success before the system has actually accepted the operation.
3. Treating API Gateway as the owner of business logic.
4. Returning huge unpaginated datasets.
5. Changing API response contracts without considering older clients.
6. Retrying POST requests without idempotency protection.
7. Applying only IP-based rate limiting in a multi-tenant platform.
8. Assuming authentication guarantees authorization.
9. Creating too many nested REST resources.
10. Using GraphQL without query-depth and complexity limits.
11. Ignoring distributed tracing across gateway and backend services.
12. Exposing internal service addresses directly to public clients without a clear reason.

---

## 17. Practical API Design Checklist

### API contract
- [ ] Resource names and HTTP methods are consistent.
- [ ] Request and response schemas are documented.
- [ ] HTTP status codes are meaningful.
- [ ] Error responses have stable machine-readable codes.
- [ ] Idempotency is implemented for important retryable operations.

### Performance
- [ ] Large lists are paginated.
- [ ] Filtering and sorting are controlled.
- [ ] Request and response sizes are limited.
- [ ] Timeouts and deadlines are defined.
- [ ] Rate limits protect critical endpoints.

### Compatibility
- [ ] Breaking changes are versioned or migrated safely.
- [ ] Old clients remain supported during migration.
- [ ] Deprecation timelines are communicated.

### Security
- [ ] Authentication is enforced.
- [ ] Object-level authorization is checked.
- [ ] Input validation is performed.
- [ ] Sensitive information is not exposed in logs.
- [ ] TLS and appropriate request limits are configured.

### Operations
- [ ] Request and trace IDs are propagated.
- [ ] P95 and P99 latency are monitored.
- [ ] Error rates and downstream failures are monitored.
- [ ] API dependencies and failure behavior are documented.

---

## 18. Revision Questions

1. What is the difference between synchronous and asynchronous communication?
2. When would you choose REST over gRPC?
3. What is the difference between PUT and PATCH?
4. What is an idempotency key, and why is it important for POST?
5. What is the difference between offset and cursor pagination?
6. What makes an API change backward-compatible?
7. What is the role of an API Gateway?
8. How is an API Gateway different from a Load Balancer?
9. What is the Backend for Frontend pattern?
10. What are the differences between REST, gRPC, and GraphQL?
11. Explain the Token Bucket algorithm.
12. What is the difference between authentication and authorization?
13. Why are request IDs and distributed tracing important?
14. Why should a ride-creation API return a pending status if assignment happens asynchronously?

---

## 19. Practice Exercise — Design a Ride-Booking API

Design APIs for the following operations:

- Passenger creates a ride.
- Passenger retrieves a ride.
- Passenger retrieves ride history.
- Passenger cancels a ride.
- Dispatcher assigns a driver.
- Driver updates their location.
- Passenger checks live ride status.

For each API, define:

1. HTTP method.
2. Endpoint.
3. Required request fields.
4. Expected response.
5. Appropriate HTTP status codes.
6. Authentication and authorization requirements.
7. Whether idempotency is needed.
8. Whether the operation should be synchronous or asynchronous.

### Bonus interview question

"Your ride-creation endpoint is receiving 10,000 requests per second during a peak period. How would you protect the API and its downstream services without unnecessarily rejecting legitimate traffic?"

Consider rate limiting, load balancing, queueing, idempotency, backpressure, and capacity planning.

**Key takeaway:** Good API design is about clear contracts, predictable behavior, security, backward compatibility, and choosing the right communication pattern for each operation.

### Practice focus

Before moving forward, try designing the seven APIs in the exercise. Pay particular attention to:
- Which operations return immediately and which are asynchronous.
- Which operations need idempotency.
- How pagination works for ride history.
- How the API protects one tenant from another.