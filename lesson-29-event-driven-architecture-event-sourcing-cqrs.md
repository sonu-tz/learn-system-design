# Lesson 29 — Event-Driven Architecture: Events, Event Sourcing, CQRS, and Eventual Consistency

In the previous lesson, we explored microservices and how they communicate. Now we'll go deeper into one of the most important architectural patterns used in distributed systems: Event-Driven Architecture (EDA).

We'll understand how services communicate through events, how event sourcing stores business history, how CQRS separates reads and writes, and how these patterns work together.

These concepts are particularly useful for booking systems, payment platforms, real-time applications, and large-scale distributed systems.

## 1. What Is Event-Driven Architecture?

Event-Driven Architecture (EDA) is an architectural style in which components communicate by producing and consuming events.

An event represents something that has already happened in the system.

Examples:

- BookingCreated
- BookingCancelled
- DriverAssigned
- PaymentAuthorized
- PaymentFailed
- DriverLocationUpdated
- NotificationSent

Instead of one service directly calling every other service, it publishes an event. Other interested services consume that event and react independently.

### Traditional request-response architecture

Suppose a passenger creates a booking.

```text
Client
  |
  v
Booking Service
  |
  +------> Dispatch Service
  |
  +------> Payment Service
  |
  +------> Notification Service
  |
  v
Response
```

The Booking Service knows about several downstream services.

If Notification Service is unavailable, the request flow may become affected depending on how the integration is implemented.

### Event-driven architecture

```text
Client
  |
  v
Booking Service
  |
  v
Message Broker
  |
  +------> Dispatch Service
  |
  +------> Payment Service
  |
  +------> Notification Service
```

The Booking Service publishes an event such as `BookingCreated`.

Each consumer processes it independently.

The Booking Service does not need to know every consumer.

### Key benefit

Event-driven architecture reduces direct runtime coupling between services.

However, it introduces eventual consistency, duplicate delivery handling, ordering concerns, and more complex debugging.

---

## 2. What Is an Event?

An event is an immutable record describing something that happened.

For example:

```json
{
  "eventId": "evt-1001",
  "eventType": "BookingCreated",
  "eventVersion": 1,
  "occurredAt": "2026-10-30T10:30:00Z",
  "bookingId": "booking-123",
  "passengerId": "passenger-456",
  "pickupLocation": {
    "latitude": 28.6139,
    "longitude": 77.2090
  }
}
```

An event should describe a fact, not an instruction.

### Event vs Command

These two concepts are different.

**Command:** Requests that something should happen.

Examples:
- CreateBooking
- AssignDriver
- AuthorizePayment
- CancelBooking

**Event:** States that something has happened.

Examples:
- BookingCreated
- DriverAssigned
- PaymentAuthorized
- BookingCancelled

| Command | Event |
|---|---|
| Expresses intent | Expresses a fact |
| May be rejected | Represents an accepted occurrence |
| Usually imperative | Usually past tense |
| Sent to a specific handler | Can have multiple consumers |

Example:

```text
AssignDriver
     |
     v
Dispatch Service
     |
     v
Driver successfully assigned
     |
     v
DriverAssigned event
```

The command may fail. The event is published only after the assignment has successfully occurred.

### Important principle

Do not publish `DriverAssigned` simply because someone requested an assignment.

Publish it when the assignment has actually happened.

---

## 3. Event Producer, Broker, and Consumer

Three important components form the foundation of event-driven systems.

### Producer

A producer creates and publishes events.

Example:
- Booking Service publishes BookingCreated.
- Payment Service publishes PaymentAuthorized.

### Message Broker

A broker receives events and makes them available to consumers.

Examples:
- Apache Kafka
- RabbitMQ
- Amazon SNS/SQS
- Google Pub/Sub

### Consumer

A consumer receives and processes events.

Example:
- Dispatch Service consumes BookingCreated.
- Notification Service consumes DriverAssigned.

```text
+----------------+
| Booking Service|
+----------------+
        |
        | BookingCreated
        v
+----------------+
| Message Broker |
+----------------+
        |
        +-------------------+
        |                   |
        v                   v
+----------------+  +--------------------+
| Dispatch       |  | Notification       |
| Consumer       |  | Consumer           |
+----------------+  +--------------------+
```

A broker helps decouple producers and consumers, but it does not automatically guarantee that every consumer processes every event correctly.

Consumers still need durable processing, acknowledgements or offset management, retries, and idempotency.

---

## 4. Event-Driven Architecture Patterns

There are several ways to structure event-driven communication.

### A. Event Notification

An event contains enough information to notify consumers that something happened.

Example:

```json
{
  "eventType": "BookingCreated",
  "bookingId": "B123"
}
```

The consumer may call Booking Service to retrieve additional information.

**Advantage:** Small messages and less duplicated data.

**Disadvantage:** Consumers may need additional network calls.

### B. Event-Carried State Transfer

The event contains more of the data needed by consumers.

Example:

```json
{
  "eventType": "BookingCreated",
  "bookingId": "B123",
  "passengerId": "P456",
  "pickup": {
    "latitude": 28.6139,
    "longitude": 77.2090
  },
  "dropoff": {
    "latitude": 28.5355,
    "longitude": 77.3910
  },
  "vehicleType": "SEDAN"
}
```

Consumers can process the event without immediately calling the producer.

**Advantage:** Fewer synchronous dependencies.

**Disadvantage:** More data duplication, larger events, and additional schema-management responsibilities.

### Choosing between them

Use event notification when consumers need only an identifier or a small notification.

Use event-carried state transfer when consumers need the event's data to perform their work independently.

Avoid putting unnecessary sensitive or rapidly changing data into events.

---

## 5. Event Topics and Multiple Consumers

In a broker such as Kafka, events are commonly published to topics.

Example:

```text
Topic: booking-events

BookingCreated
BookingUpdated
BookingCancelled
BookingCompleted
```

Multiple consumer groups can independently subscribe to the topic.

```text
                 booking-events
                       |
          +------------+------------+
          |            |            |
          v            v            v
      Dispatch     Analytics    Notification
      Consumer      Consumer      Consumer
```

Each consumer group maintains its own processing position.

This means Dispatch can process an event independently of Analytics.

### Important distinction

Within a consumer group, multiple consumers generally share the work across partitions.

Different consumer groups independently consume the same topic.

We covered Kafka topics, partitions, consumer groups, and offsets in Lesson 19.

---

## 6. Eventual Consistency

In event-driven systems, different services may update their data at different times.

Suppose a booking is created.

```text
Booking Service
      |
      v
BookingCreated event
      |
      v
Message Broker
      |
      +------> Dispatch Service
      |
      +------> Notification Service
      |
      +------> Analytics Service
```

The Booking Service may commit the booking immediately.

Dispatch may process the event a few milliseconds later.

Analytics may process it later still.

During this period, different services may have different views of the same business process.

This is called eventual consistency.

### Example

At 10:30:00:

```text
Booking DB:
Booking B123 = CREATED

Dispatch DB:
Booking B123 = NOT FOUND

Analytics DB:
Booking B123 = NOT FOUND
```

At 10:30:02:

```text
Booking DB:
Booking B123 = CREATED

Dispatch DB:
Booking B123 = ASSIGNING

Analytics DB:
Booking B123 = CREATED
```

Eventually, all relevant systems process the event and converge to the appropriate state.

### Important distinction

Eventual consistency does not mean data will always become correct automatically.

It assumes that events are delivered and successfully processed, failures are recovered, and conflicting updates are handled correctly.

---

## 7. The Dual-Write Problem

One of the most important problems in event-driven architecture is the dual-write problem.

Suppose Booking Service needs to:

1. Insert a booking into PostgreSQL.
2. Publish BookingCreated to Kafka.

A naive implementation:

```text
BEGIN TRANSACTION

INSERT INTO bookings ...

COMMIT

PUBLISH BookingCreated
```

What happens if the database commit succeeds but Kafka is unavailable?

```text
Database:
Booking created successfully

Kafka:
Event not published
```

The booking exists, but Dispatch never receives the event.

The opposite order is also unsafe:

```text
PUBLISH BookingCreated

INSERT INTO bookings ...
```

If publishing succeeds but the database transaction fails, consumers receive an event for a booking that does not exist.

These operations are not automatically atomic because PostgreSQL and Kafka are separate systems.

### Solution: Transactional Outbox Pattern

The transactional outbox pattern stores the business change and the event record in the same database transaction.

```text
BEGIN TRANSACTION

INSERT INTO bookings ...

INSERT INTO outbox_events ...

COMMIT
```

Now both records are committed or rolled back together.

A separate publisher reads the outbox and publishes events to the broker.

```text
             Booking Service
                    |
                    v
              PostgreSQL
              /         \
             v           v
       bookings      outbox_events
                          |
                          v
                    Outbox Publisher
                          |
                          v
                     Message Broker
```

### Example outbox record

```json
{
  "id": "evt-1001",
  "aggregateType": "Booking",
  "aggregateId": "B123",
  "eventType": "BookingCreated",
  "payload": {
    "bookingId": "B123",
    "passengerId": "P456"
  },
  "createdAt": "2026-10-30T10:30:00Z",
  "publishedAt": null
}
```

### How does the publisher work?

1. Read unpublished outbox records.
2. Publish the event to Kafka.
3. Wait for broker acknowledgement.
4. Mark the outbox record as published.

If the publisher crashes after Kafka accepts the event but before the database record is marked published, it may publish the event again.

Therefore, consumers must be idempotent.

### Important principle

The transactional outbox prevents the database-write/event-publish gap.

It does not guarantee exactly-once delivery to every consumer.

---

## 8. Idempotent Event Consumers

An event may be delivered more than once.

For example:

1. Consumer receives BookingCreated.
2. Consumer updates its database.
3. Consumer crashes before committing its Kafka offset.
4. The event is delivered again.

Without protection, the consumer may repeat the business operation.

An idempotent consumer ensures that processing the same event more than once does not produce duplicate business effects.

### Deduplication table

```sql
CREATE TABLE processed_events (
    event_id VARCHAR(100) PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL
);
```

A consumer can process an event and record its ID in the same local database transaction.

Conceptually:

```text
BEGIN TRANSACTION

Check whether event ID was processed.

If already processed:
    Do nothing.

Otherwise:
    Apply business change.
    Record event ID.

COMMIT
```

The unique constraint protects against concurrent duplicate processing.

### Important consideration

The deduplication record and business update must be atomic in the consumer's database.

If they are separate operations, another dual-write problem can occur.

If processing also calls an external provider, such as a payment gateway, database deduplication alone cannot guarantee that the external side effect happens exactly once. Use the provider's idempotency mechanism where available.

---

## 9. Event Ordering

Some business processes depend on event order.

Consider:

```text
BookingCreated
DriverAssigned
BookingStarted
BookingCompleted
```

If BookingCompleted arrives before DriverAssigned, a consumer may process an invalid transition.

### Why can ordering break?

- Multiple producers.
- Multiple partitions.
- Concurrent consumers.
- Retries.
- Delayed messages.
- Network failures.

### Common solutions

**Partition by entity ID**

For Kafka, events with the same booking ID can be sent using the same partition key.

```text
Partition key = bookingId
```

This preserves order within that partition for records produced in the intended order.

**Sequence numbers**

Include a version or sequence number in each event.

```json
{
  "bookingId": "B123",
  "eventType": "BookingUpdated",
  "sequence": 12
}
```

A consumer can detect duplicate, stale, or out-of-order updates.

**State transition validation**

Validate whether an event is valid for the current business state.

**Idempotent updates**

Ensure repeated or stale events do not corrupt the state.

### Important limitation

Kafka ordering is guaranteed within a partition, not across all partitions.

Also, multiple producers require care: partitioning by key alone does not guarantee the correct business order if producers publish concurrently without coordination.

---

## 10. What Is Event Sourcing?

Event sourcing is a data-storage pattern in which the system stores the sequence of business events as the primary record of state changes.

Instead of storing only the latest state, it stores the history of changes.

### Traditional state-based storage

A booking table might contain:

| Booking ID | Status |
|---|---|
| B123 | COMPLETED |

The current state is stored, but the sequence of transitions may not be available from this table alone.

### Event-sourced storage

The system stores events:

| Sequence | Booking ID | Event |
|---:|---|---|
| 1 | B123 | BookingCreated |
| 2 | B123 | DriverAssigned |
| 3 | B123 | BookingStarted |
| 4 | B123 | BookingCompleted |

The current state is derived by replaying the events.

```text
BookingCreated
      |
      v
DriverAssigned
      |
      v
BookingStarted
      |
      v
BookingCompleted
      |
      v
Current Booking State = COMPLETED
```

### Event store

An event store stores the ordered event history for business entities or streams.

Example:

```json
{
  "eventId": "evt-1004",
  "aggregateId": "B123",
  "eventType": "BookingCompleted",
  "version": 4,
  "occurredAt": "2026-10-30T11:15:00Z",
  "payload": {
    "completedAt": "2026-10-30T11:15:00Z"
  }
}
```

### Advantages

- Complete business history.
- Ability to reconstruct past states.
- Auditability.
- Ability to build new read models from historical events.
- Useful for debugging complex state transitions.

### Disadvantages

- More complex data modeling.
- Event schema evolution is difficult.
- Rebuilding state can be expensive.
- Event storage grows continuously.
- Deletion and privacy requirements need careful design.
- Queries often require separate projections.

### Important distinction

Event sourcing is not the same as event-driven architecture.

You can use event-driven architecture without event sourcing.

You can also use event sourcing inside one application without building a large microservices architecture.

---

## 11. Aggregates and Event Streams

In Domain-Driven Design, an aggregate is a consistency boundary around related business state.

An aggregate typically has:

- An aggregate ID.
- Business rules.
- State transitions.
- Events describing accepted changes.

Example: Booking aggregate.

```text
Booking Aggregate: B123

Events:
1. BookingCreated
2. DriverAssigned
3. BookingStarted
4. BookingCompleted
```

The aggregate version can increase with every accepted state change.

```text
Version 1: BookingCreated
Version 2: DriverAssigned
Version 3: BookingStarted
Version 4: BookingCompleted
```

A command handler can use the expected version to prevent conflicting concurrent updates.

For example, two commands attempting to modify version 3 should not both silently create version 4.

Optimistic concurrency control can reject one conflicting write.

---

## 12. Replaying Events

One of the powerful capabilities of event sourcing is event replay.

Suppose we have:

```text
BookingCreated
DriverAssigned
BookingStarted
BookingCompleted
```

If we replay these events in order, we can reconstruct the booking's current state.

### Example

Initial state:

```json
{
  "bookingId": "B123",
  "status": "NOT_CREATED"
}
```

After BookingCreated:

```json
{
  "bookingId": "B123",
  "status": "CREATED"
}
```

After DriverAssigned:

```json
{
  "bookingId": "B123",
  "status": "ASSIGNED"
}
```

After BookingStarted:

```json
{
  "bookingId": "B123",
  "status": "IN_PROGRESS"
}
```

After BookingCompleted:

```json
{
  "bookingId": "B123",
  "status": "COMPLETED"
}
```

### Replay safety

Replaying historical events must not repeat real-world side effects.

For example, rebuilding booking state must not:

- Charge a passenger again.
- Send another SMS.
- Assign another driver.
- Trigger another external payment request.

Projection rebuilding should be separated from side-effecting workflows.

---

## 13. Snapshots

Replaying millions of events every time an entity is loaded can become expensive.

Event sourcing commonly uses snapshots.

A snapshot stores the derived state at a particular event version.

Example:

```text
Events 1-1000
      |
      v
  Snapshot at 1000
      |
      v
Events 1001-1050
      |
      v
Current State
```

To rebuild the state:

1. Load the latest valid snapshot.
2. Read events after the snapshot version.
3. Apply the remaining events.

Snapshots improve performance but are not the source of truth. They are derived data that can be rebuilt from the event history.

---

## 14. What Is CQRS?

CQRS stands for Command Query Responsibility Segregation.

It separates the model used to change data from the model used to read data.

### Traditional architecture

The same model often handles both reads and writes.

```text
Client
  |
  v
Application
  |
  v
Database
```

### CQRS architecture

```text
                 Client
                    |
              +-----+-----+
              |           |
              v           v
         Command API   Query API
              |           |
              v           v
        Write Model    Read Model
              |           ^
              v           |
          Write DB        |
              |           |
              v           |
             Events ------+
```

### Command side

Handles operations that change business state.

Examples:

- CreateBooking.
- AssignDriver.
- CancelBooking.
- AuthorizePayment.

### Query side

Handles operations that retrieve data.

Examples:

- GetBookingDetails.
- SearchAvailableDrivers.
- GetPassengerTripHistory.
- GetDispatcherDashboard.

The two sides can use different models and storage technologies.

### Why separate reads and writes?

A booking write may require strong consistency and transactional validation.

A dispatcher dashboard may require a denormalized view combining booking, driver, and payment data.

These workloads have different requirements.

CQRS allows each side to be optimized independently.

---

## 15. CQRS Does Not Require Event Sourcing

These patterns are related but independent.

| Architecture | Write model | Read model |
|---|---|---|
| Traditional CRUD | Shared model | Shared model |
| CQRS | Separate command model | Separate query model |
| Event sourcing | Events are primary record | Usually projections |
| CQRS + Event sourcing | Events drive write-side state | Projections serve queries |

You can introduce CQRS without an event store.

For example, a command service can write to PostgreSQL while a separate query model is updated through an outbox and message broker.

Similarly, event sourcing can be implemented without using microservices.

Do not combine all these patterns unless the requirements justify the additional complexity.

---

## 16. Building Read Models with CQRS

Suppose the dispatcher screen needs:

- Booking ID.
- Passenger name.
- Pickup address.
- Driver name.
- Driver location.
- Assignment status.
- Payment status.

This information belongs to different business domains.

Instead of querying every service whenever the screen loads, build a read model.

### Data flow

```text
Booking Service -----> BookingCreated
                            |
                            v
Driver Service ------> DriverUpdated
                            |
                            v
Payment Service -----> PaymentUpdated
                            |
                            v
                      Message Broker
                            |
                            v
                    Projection Consumer
                            |
                            v
                    Dispatcher Read DB
                            |
                            v
                       Dispatcher UI
```

The projection consumer combines events and updates a query-friendly representation.

Example read model:

```json
{
  "bookingId": "B123",
  "passengerName": "Sonu",
  "driverName": "Driver A",
  "assignmentStatus": "ASSIGNED",
  "paymentStatus": "AUTHORIZED"
}
```

This model is optimized for the dispatcher screen rather than for ownership of the underlying business data.

### Trade-off

The read model can be temporarily stale.

If Payment Service updates a payment status, the dispatcher read model may reflect that update slightly later.

The application must decide whether that delay is acceptable.

For high-risk operations, validate the authoritative state with the owning service before performing a critical action.

---

## 17. Event Schema Evolution

Events are contracts between producers and consumers.

Changing an event schema can break consumers that were built against an older version.

Example version 1:

```json
{
  "bookingId": "B123",
  "status": "ASSIGNED"
}
```

Version 2:

```json
{
  "bookingId": "B123",
  "status": "ASSIGNED",
  "driverId": "D456"
}
```

Adding an optional field is generally easier to make backward-compatible than removing or changing an existing field.

### Common practices

- Add optional fields instead of removing existing fields.
- Preserve existing field meanings.
- Avoid changing field types.
- Define how consumers handle unknown fields.
- Use explicit event schema versions when appropriate.
- Validate compatibility in CI/CD.
- Support old and new consumers during migration.

A schema registry can enforce compatibility rules for supported serialization formats.

### Event versioning vs event type versioning

These approaches should be designed deliberately.

For example:

```text
BookingCreated.v1
BookingCreated.v2
```

or a stable event type with a version field.

The important point is that producers and consumers must agree on the contract and its evolution strategy.

---

## 18. Eventual Consistency and User Experience

Event-driven systems often update read models asynchronously.

This creates a user experience challenge.

Example:

1. Passenger creates a booking.
2. Booking Service commits it.
3. The frontend requests the booking details.
4. Read model has not processed BookingCreated yet.
5. The booking is temporarily missing from the read model.

Possible approaches:

**Read from the authoritative write model:** Useful when immediate consistency is necessary.

**Return the created resource from the command:** The frontend can display the result immediately.

**Optimistic UI:** Show the expected result while waiting for confirmation.

**Read-your-writes routing:** Route a user's immediate follow-up reads to a source that has processed their write.

**Version-based waiting:** Include a version or sequence number and wait until the read model has reached the required version.

Choose based on the business requirement.

---

## 19. When Should You Use These Patterns?

| Pattern | Best suited for |
|---|---|
| Event-driven architecture | Independent reactions to business events |
| Transactional outbox | Reliable database-to-broker publishing |
| Event sourcing | Complete business history and state reconstruction |
| CQRS | Different read and write requirements |
| Materialized read model | Fast combined queries |
| Saga | Multi-service business workflows |
| Snapshots | Faster event-sourced state reconstruction |

### A practical decision rule

Use event-driven architecture when independent components need to react to business changes.

Use the outbox pattern when a database update must reliably produce an event.

Use CQRS when read and write models have meaningfully different needs.

Use event sourcing when the event history itself is an important business requirement.

Do not introduce event sourcing just because your system already uses Kafka.

---

## 20. Worked Example: End-to-End Booking Architecture

Let's combine the concepts.

### Requirement

A passenger creates a booking. The system must:

1. Save the booking.
2. Find and assign a driver.
3. Authorize payment.
4. Notify the passenger.
5. Update the dispatcher dashboard.
6. Preserve the booking history.

### Architecture

```text
                 Passenger App
                       |
                       v
                  Booking API
                       |
                       v
                 Booking Service
                       |
                 PostgreSQL
                  /        \
                 v          v
            Bookings    Outbox Events
                            |
                            v
                       Outbox Publisher
                            |
                            v
                       Message Broker
                            |
            +---------------+---------------+
            |               |               |
            v               v               v
         Dispatch        Payment       Notification
         Service         Service        Service
            |               |               |
            v               v               v
         Driver DB       Payment DB     Notification DB

                            |
                            v
                     Projection Consumer
                            |
                            v
                    Dispatcher Read Model
```

### Step-by-step flow

**Step 1 — Command**

Passenger sends:

```text
CreateBooking
```

**Step 2 — Write transaction**

Booking Service:

- Validates the command.
- Creates the booking.
- Writes BookingCreated into the outbox.
- Commits the transaction.

**Step 3 — Event publishing**

Outbox publisher sends BookingCreated to the broker.

**Step 4 — Independent processing**

- Dispatch Service processes the booking.
- Payment Service starts the required payment operation.
- Notification Service sends a booking acknowledgement.
- Projection consumer updates the dispatcher read model.

**Step 5 — Subsequent events**

Dispatch publishes DriverAssigned after successful assignment.

Payment publishes PaymentAuthorized after successful authorization.

**Step 6 — Read model update**

The dispatcher projection consumes these events and updates the dashboard representation.

**Step 7 — Event history**

If the system uses event sourcing for the booking aggregate, the accepted business events are persisted in the event store and can be replayed to reconstruct booking state.

If it does not use event sourcing, the booking table remains the primary state store and events are integration messages.

### Important design decisions

- Booking creation is transactional.
- Outbox prevents lost event publication.
- Consumers are idempotent.
- Dispatch and payment process independently.
- Saga coordination handles multi-service workflow failures.
- Read model is eventually consistent.
- Event schemas evolve compatibly.
- Event sourcing is used only if historical event reconstruction is required.

---

## 21. Common Mistakes

1. Treating commands and events as the same thing.
2. Publishing events before the underlying business transaction commits.
3. Assuming a broker guarantees exactly-once business processing.
4. Not implementing idempotent consumers.
5. Depending on global ordering across Kafka partitions.
6. Using event sourcing for every database table.
7. Assuming CQRS requires Kafka or event sourcing.
8. Allowing read models to become stale without considering user experience.
9. Changing event schemas without considering older consumers.
10. Replaying events that trigger real external side effects.
11. Ignoring event retention and privacy requirements.
12. Introducing too many asynchronous workflows without operational observability.

---

## 22. Practical Checklist

### Event design
- [ ] Does each event describe a fact that has happened?
- [ ] Is the event name meaningful?
- [ ] Is there a stable event ID?
- [ ] Is the event schema versioned or governed?
- [ ] Does the event contain only necessary data?

### Publishing
- [ ] Is the business update committed before the event is published?
- [ ] Is the transactional outbox used where required?
- [ ] Can publishers safely retry?

### Consumption
- [ ] Are consumers idempotent?
- [ ] Are business updates and deduplication atomic?
- [ ] Are retries and dead-letter handling defined?
- [ ] Is event ordering important?
- [ ] Can consumers recover after downtime?

### CQRS and projections
- [ ] Is a separate read model justified?
- [ ] Is acceptable staleness defined?
- [ ] Can projections be rebuilt?
- [ ] Are read models monitored for lag?

### Event sourcing
- [ ] Is historical reconstruction a real requirement?
- [ ] Are event schemas evolvable?
- [ ] Is concurrency control defined?
- [ ] Are snapshots needed?
- [ ] Are replay side effects prevented?

---

## 23. Revision Questions

1. What is Event-Driven Architecture?
2. What is the difference between a command and an event?
3. What are producers, brokers, and consumers?
4. What is the difference between event notification and event-carried state transfer?
5. What is eventual consistency?
6. Explain the dual-write problem.
7. How does the transactional outbox pattern solve it?
8. Why do consumers need idempotency?
9. Why is event ordering difficult in distributed systems?
10. What is event sourcing?
11. How is event sourcing different from traditional CRUD?
12. What is an aggregate?
13. Why do event-sourced systems use snapshots?
14. What is CQRS?
15. Does CQRS require event sourcing?
16. What is a materialized read model?
17. How do you handle read-your-writes consistency?
18. How should event schemas evolve?
19. When should you avoid event sourcing?
20. How do outbox, CQRS, and Saga patterns complement each other?

---

## 24. Practical Exercise

Design an event-driven architecture for a ride-booking platform.

The platform supports:

- Booking creation and cancellation.
- Driver assignment.
- Payment authorization and refunds.
- Driver location updates.
- Passenger notifications.
- Dispatcher dashboards.

Your design must answer:

1. Which services publish events?
2. What events should each service publish?
3. Which services consume those events?
4. Where would you use synchronous communication?
5. Where would you use asynchronous communication?
6. How would you prevent losing events after a database transaction?
7. How would you prevent duplicate event processing?
8. How would you maintain event ordering for a booking?
9. Would you use CQRS for the dispatcher dashboard? Explain why.
10. Would you use event sourcing for the booking lifecycle? Explain the trade-offs.
11. How would you rebuild a failed read model?
12. How would you ensure replaying events does not trigger duplicate payments or notifications?

**Key takeaway:** Event-driven architecture helps services react independently to business events. The transactional outbox protects event publication, idempotency protects consumers, CQRS optimizes read and write workloads, and event sourcing preserves the history of business changes. These patterns solve different problems and should be combined only when their benefits justify their complexity.

### Practice focus

Focus on understanding the differences between Event-Driven Architecture, Event Sourcing, and CQRS. They are related, but they solve three different architectural problems.