# Lesson 15 — Distributed Transactions, Sagas, and Eventual Consistency Patterns

So far, we've covered database transactions, message queues, the transactional outbox, and CAP theorem.

Now let's connect those concepts.

Imagine a passenger books a ride. The system needs to:
1. Create a ride.
2. Find an available driver.
3. Assign the driver.
4. Process the payment.
5. Notify the passenger and driver.
What happens if ride creation succeeds, driver assignment succeeds, but payment processing fails?

A normal database transaction cannot easily roll back changes across multiple independent services and databases.

This is where distributed transactions, Sagas, and compensating actions become important.

## 1. Why Do We Need Distributed Transactions?

Let's start with a simple architecture.

A ride-booking application has multiple services:

- Ride Service
- Driver Service
- Payment Service
- Notification Service

Each service owns its own database.

<pre>
             API Gateway
                  |
          ┌───────┴────────┐
          |                |
      Ride Service    Driver Service
          |                |
      Ride DB          Driver DB
          |
     Payment Service
          |
      Payment DB
          |
  Notification Service
          |
    Notification DB
</pre>

Consider a passenger booking a ride.

The application needs to:

1. Create a ride.
2. Reserve or assign a driver.
3. Authorize payment.
4. Confirm the booking.
5. Send notifications.

Each operation may succeed or fail independently.

### The problem

Suppose:

- Ride creation succeeds.
- Driver assignment succeeds.
- Payment authorization fails.

Now the system has inconsistent business state:

- The ride exists.
- A driver is assigned.
- Payment was not authorized.

If everything lived in one database, a single transaction might roll back all changes.

But with independent services and databases, a local transaction cannot automatically undo work performed by another service.

This is the distributed transaction problem.

---

## 2. Why Not Use One Big Database Transaction?

A traditional database transaction provides ACID guarantees within its transaction boundary.

For example:

```sql
BEGIN;

INSERT INTO rides (...);

UPDATE drivers
SET availability = 'assigned'
WHERE id = 101;

INSERT INTO payments (...);

COMMIT;
```

If an operation fails before commit, the database can roll back the transaction.

But imagine the payment service uses a separate database or an external payment provider.

The ride database cannot directly roll back an already successful payment authorization.

Similarly, the payment provider cannot automatically undo a driver assignment.

This creates multiple independent transaction boundaries.

### The distributed transaction challenge

- One service can commit while another fails.
- Network requests can time out after an operation succeeds.
- Services may be temporarily unavailable.
- Retrying can accidentally repeat operations.
- There may be no single transaction manager controlling every resource.

We need an approach that handles partial success.

---

## 3. Two Common Approaches

There are two broad approaches to coordinating work across multiple systems.

1. Two-Phase Commit (2PC).
2. Saga Pattern.

They solve different problems and have different trade-offs.

---

## 4. Two-Phase Commit (2PC)

Two-Phase Commit is a distributed transaction protocol.

It coordinates multiple participants so they can commit or abort a transaction together, assuming the participants and transaction coordinator support the protocol.

There are two phases.

### Phase 1: Prepare

A coordinator asks every participant:

"Can you commit this transaction?"

Each participant prepares its changes and responds:

- YES — ready to commit.
- NO — cannot commit.

### Phase 2: Commit or Abort

If every participant votes YES, the coordinator tells all participants to commit.

If any participant votes NO, the coordinator tells them to abort.

### Architecture

<pre>
                Coordinator
                     |
           ┌─────────┼─────────┐
           |         |         |
           ▼         ▼         ▼
        Service A Service B Service C
           |         |         |
           ▼         ▼         ▼
         DB A      DB B      DB C
</pre>

### Flow

<pre>
Coordinator
    |
    |--- PREPARE ---> DB A
    |--- PREPARE ---> DB B
    |--- PREPARE ---> DB C
    |
    |<--- YES -------- DB A
    |<--- YES -------- DB B
    |<--- YES -------- DB C
    |
    |--- COMMIT ----> DB A
    |--- COMMIT ----> DB B
    |--- COMMIT ----> DB C
</pre>

### Advantages

- Provides coordinated commit or abort.
- Helps maintain atomicity across supported participants.
- Useful where all participants support a compatible transaction protocol.

### Disadvantages

- Participants may hold locks or prepared transaction state while waiting.
- The coordinator can become a bottleneck or failure dependency.
- A coordinator failure can leave participants waiting for a decision.
- Long network delays can increase transaction latency.
- Not all external services support 2PC.

**Important:** 2PC is not the same as making an HTTP request to multiple services and then rolling back manually. It requires protocol support from the participants.

For many microservice architectures, 2PC is not the preferred approach.

---

## 5. Saga Pattern

A Saga breaks a distributed business transaction into a sequence of smaller local transactions.

Each service performs its own transaction.

If a later step fails, the system executes compensating actions for earlier successful steps where possible.

Example:

Ride booking:

1. Create ride.
2. Reserve driver.
3. Authorize payment.
4. Confirm ride.

If payment authorization fails, the system may:

1. Release the reserved driver.
2. Cancel the ride.

These are compensating actions.

### Architecture

<pre>
             Saga Orchestrator
                    |
          ┌─────────┼─────────┐
          |         |         |
          ▼         ▼         ▼
       Ride       Driver    Payment
       Service    Service   Service
          |         |         |
          ▼         ▼         ▼
        Ride DB  Driver DB Payment DB
</pre>

Each service commits its own local transaction.

The Saga coordinates the overall workflow.

### Example sequence

<pre>
Create Ride
    |
    ▼
Reserve Driver
    |
    ▼
Authorize Payment
    |
    ▼
Confirm Ride
    |
    ▼
Saga Completed
</pre>

If payment authorization fails:

<pre>
Create Ride
    |
    ▼
Reserve Driver
    |
    ▼
Authorize Payment
    |
    X  Payment Failed
    |
    ▼
Release Driver
    |
    ▼
Cancel Ride
    |
    ▼
Saga Compensated
</pre>

### Important distinction

A compensating action is not necessarily a database rollback.

It is a new business operation that attempts to reverse the effect of a previously completed operation.

For example:

- A driver reservation is released.
- A ride is marked cancelled.
- An authorized payment is voided.

The original action may already have been visible to other services or users.

---

## 6. Saga: Choreography vs. Orchestration

There are two common ways to coordinate a Saga.

### 6.1 Choreography

In choreography, services communicate through events.

There is no central coordinator controlling every step.

Each service listens for events and decides what to do next.

Example:

<pre>
Ride Service
     |
     | RideCreated
     ▼
Message Broker
     |
     ▼
Driver Service
     |
     | DriverReserved
     ▼
Message Broker
     |
     ▼
Payment Service
     |
     | PaymentAuthorized
     ▼
Message Broker
     |
     ▼
Ride Service
     |
     ▼
RideConfirmed
</pre>

If payment fails:

<pre>
Payment Service
     |
     | PaymentFailed
     ▼
Message Broker
     |
     ├───────────────┐
     ▼               ▼
Driver Service   Ride Service
     |               |
Release Driver   Cancel Ride
</pre>

**Advantages**

- Services are relatively independent.
- No central workflow controller is required.
- Event-driven architecture can support high scalability.
- Each service owns its local business logic.

**Disadvantages**

- Workflow logic becomes distributed across services.
- Debugging is harder.
- Event dependencies can become difficult to understand.
- Adding or changing steps may require changes across multiple services.

Choreography works well for relatively simple workflows with clear event ownership.

### 6.2 Orchestration

In orchestration, a central component coordinates the workflow.

This component is called the Saga orchestrator.

It knows:

- Which step runs next.
- Which service must be called.
- What happens when a step succeeds.
- What compensation is required after failure.

Example:

<pre>
               Saga Orchestrator
                 /     |     \
                /      |      \
               ▼       ▼       ▼
          Ride Svc  Driver Svc Payment Svc
               \       |       /
                \      |      /
                 Results
                    |
                    ▼
              Next Step
</pre>

Example workflow:

| Step | Action | Compensation |
|---|---|---|
| 1 | Create ride | Cancel ride |
| 2 | Reserve driver | Release driver |
| 3 | Authorize payment | Void authorization |
| 4 | Confirm ride | Initiate cancellation workflow if required |

**Advantages**

- Workflow is visible in one place.
- Easier to understand complex business processes.
- Centralized retries and compensation coordination.
- Easier to monitor workflow progress.

**Disadvantages**

- Orchestrator requires high availability.
- It can become overly complex.
- Services may become too dependent on the orchestrator if boundaries are poorly designed.

### Choreography vs. Orchestration

| Consideration | Choreography | Orchestration |
|---|---|---|
| Coordination | Events | Central coordinator |
| Workflow visibility | Distributed | Centralized |
| Complexity | Can grow across services | Concentrated in orchestrator |
| Service coupling | Event-contract dependencies | Orchestrator knows workflow steps |
| Debugging | More difficult for complex flows | Generally easier to trace |
| Best fit | Simpler event-driven workflows | Complex multi-step business workflows |

---

## 7. Compensation Is Not Always a Perfect Undo

This is a critical concept.

Suppose:

1. Ride created.
2. Driver reserved.
3. Payment authorized.
4. Passenger receives confirmation.

Then a later step fails.

Can we simply undo everything?

Not necessarily.

Some actions are reversible, others are not.

Examples:

| Original action | Possible compensation |
|---|---|
| Reserve driver | Release driver |
| Create ride | Cancel ride |
| Authorize card payment | Void authorization |
| Capture payment | Refund payment |
| Send SMS | Cannot unsend SMS |
| Send email | Cannot unsend email |

A compensation may introduce additional business consequences.

For example, a captured payment followed by a refund is not identical to never capturing the payment. It may involve fees, delays, and financial records.

**Design principle:** Define compensations based on business meaning, not simply technical rollback.

---

## 8. Eventual Consistency in a Saga

A Saga does not usually provide instantaneous global consistency.

Each local transaction commits independently.

This means different services may temporarily have different views of the workflow.

Example:

- Ride Service says `pending`.
- Driver Service says `reserved`.
- Payment Service says `authorizing`.

These states may be valid while the Saga is running.

Eventually, the workflow reaches a terminal state:

- `confirmed`
- `cancelled`
- `failed`
- `manual_review`

The application should represent intermediate states explicitly.

### Example ride state machine

<pre>
        PENDING
           |
           ▼
     DRIVER_RESERVED
           |
           ▼
      PAYMENT_PENDING
         /       \
        ▼         ▼
   CONFIRMED   PAYMENT_FAILED
                   |
                   ▼
               CANCELLING
                   |
                   ▼
               CANCELLED
</pre>

The exact state machine depends on the business process.

Avoid representing every intermediate failure as an immediate generic `failed` state. The system may still be executing compensation.

---

## 9. Idempotency in Distributed Workflows

Distributed workflows frequently retry operations.

A request may succeed, but the response may be lost.

Example:

1. Payment service receives an authorization request.
2. Payment provider successfully authorizes the payment.
3. Network response is lost.
4. Saga orchestrator times out.
5. Orchestrator retries.

Without idempotency, the passenger could be charged or authorized twice.

### Idempotency key

Generate a stable identifier for the business operation.

Example:

```text
payment:ride-501:authorization
```

When a request is retried with the same idempotency key, the payment service returns the existing result rather than creating a duplicate operation.

### Important requirements

- Use a stable key across retries.
- Store the operation's outcome.
- Prevent concurrent duplicate processing.
- Handle a retry that arrives while the original operation is still running.
- Define how long idempotency records must be retained.

Idempotency should be designed at the business-operation level, not just the HTTP request level.

---

## 10. Transactional Outbox Pattern

There is another important problem in distributed systems.

Suppose a service updates its database and publishes an event.

Example:

```text
1. Save ride to PostgreSQL.
2. Publish RideCreated event to queue.
```

What if the database commit succeeds but event publication fails?

The ride exists, but downstream services never receive `RideCreated`.

What if the event publishes successfully but the database transaction fails?

Consumers receive an event for a ride that does not exist.

### The transactional outbox solution

Store the business change and event record in the same database transaction.

Example:

```sql
BEGIN;

INSERT INTO rides (...);

INSERT INTO outbox_events (
  event_type,
  aggregate_id,
  payload
);

COMMIT;
```

A separate publisher reads the outbox table and publishes events to the message broker.

<pre>
             Ride Service
                  |
                  ▼
             PostgreSQL
             ┌───────────┐
             │ Rides     │
             │ Outbox    │
             └───────────┘
                  |
                  ▼
             Outbox Worker
                  |
                  ▼
             Message Broker
                  |
          ┌───────┴────────┐
          ▼                ▼
      Driver Service   Analytics
</pre>

### Why is this reliable?

The ride record and outbox event are committed atomically.

If the transaction fails, neither is committed.

If it succeeds, the event remains available for publishing even if the broker is temporarily unavailable.

### Important limitation

The outbox publisher can publish the same event more than once.

For example:

1. Publisher sends event.
2. Broker accepts it.
3. Publisher crashes before marking the event as published.
4. Publisher restarts and sends it again.

Therefore, consumers must be idempotent.

The outbox pattern provides reliable event publication through durable storage and retries, not magical exactly-once delivery.

---

## 11. Inbox Pattern and Duplicate Event Handling

The inbox pattern helps consumers process duplicate events safely.

A consumer stores the event ID in a table with a uniqueness constraint.

Example:

```text
processed_events
----------------
event_id
processed_at
```

Processing flow:

1. Receive event.
2. Start a database transaction.
3. Check whether event ID has already been processed.
4. If yes, skip duplicate business processing.
5. If no, perform the business update.
6. Insert event ID into processed-events table.
7. Commit.
8. Acknowledge the message.

The business update and event-ID insertion must happen in the same local transaction.

This prevents a crash between processing and recording the event from causing duplicate business effects.

### Outbox vs. Inbox

| Pattern | Purpose |
|---|---|
| Outbox | Reliably publish events after a database transaction |
| Inbox | Safely process duplicate incoming events |

They are frequently used together.

---

## 12. Retry, Timeout, and Dead-Letter Handling

A Saga may encounter temporary service failures.

For example:

- Payment service is temporarily unavailable.
- Driver service times out.
- Message broker is recovering.

Use bounded retries with:

- Exponential backoff.
- Jitter.
- Maximum attempts or deadline.
- Idempotency.
- Clear distinction between transient and permanent failures.

Do not retry permanent business failures indefinitely.

Examples:

| Failure | Typical response |
|---|---|
| Temporary network failure | Retry with backoff |
| Payment declined | Business failure; do not blindly retry |
| Invalid ride ID | Permanent failure |
| Driver unavailable | Find another driver or compensate |
| Service timeout | Check operation status before repeating non-idempotent work |

If a message repeatedly fails, route it to a dead-letter queue for investigation or controlled replay.

A dead-letter queue is not a substitute for defining the business outcome of a failed Saga.

---

## 13. What Happens When the Orchestrator Crashes?

A Saga orchestrator must not rely only on in-memory workflow state.

Imagine:

1. Ride created.
2. Driver reserved.
3. Orchestrator crashes before payment authorization.

After restart, how does it know where to continue?

### Durable workflow state

Store the Saga state in persistent storage.

Example:

| saga_id | ride_id | current_step | status |
|---|---|---|---|
| S1001 | R501 | payment | running |
| S1002 | R502 | compensation | running |
| S1003 | R503 | completed | completed |

The orchestrator should:

- Persist completed steps.
- Persist pending actions.
- Persist compensation progress.
- Resume unfinished workflows after restart.
- Avoid executing duplicate operations.
- Track workflow deadlines.

This makes the Saga recoverable.

---

## 14. When to Use 2PC vs. Saga

| Requirement | 2PC | Saga |
|---|---|---|
| Coordinated atomic commit | Yes, with protocol support | No global atomic commit |
| Independent service databases | Requires compatible participants | Natural fit |
| Long-running workflows | Generally poor fit | Better fit |
| Compensation logic | Not the core mechanism | Core mechanism |
| Availability during failures | Can block or wait | Can make progress through retries and compensation, depending on design |
| External APIs | Usually unsuitable | Can integrate through local actions and compensations |
| Operational complexity | Transaction coordination | Workflow and recovery management |

### Practical guidance

Use a local database transaction whenever the work belongs to one database boundary.

Use 2PC when coordinated atomic commit is genuinely required and all participants support it.

Use a Saga when a business workflow spans independent services and can be modeled using local transactions and compensating actions.

Do not split a simple application into many services just to use Sagas.

---

## 15. End-to-End Ride Booking Example

Let's put everything together.

### Successful workflow

1. Passenger requests a ride.
2. Ride Service creates a pending ride and an outbox event in one transaction.
3. Outbox worker publishes `RideRequested`.
4. Driver Service selects and reserves a driver.
5. Driver Service publishes `DriverReserved`.
6. Saga orchestrator requests payment authorization.
7. Payment Service authorizes payment using an idempotency key.
8. Payment Service publishes `PaymentAuthorized`.
9. Ride Service confirms the ride.
10. Notification Service sends confirmation.

### Failure workflow

Suppose payment authorization fails.

1. Payment Service publishes `PaymentFailed`.
2. Orchestrator records the failure.
3. Orchestrator requests driver release.
4. Driver Service releases the reservation.
5. Ride Service cancels the ride.
6. Passenger is notified.
7. Saga reaches `cancelled`.

### Architecture

<pre>
Passenger
    |
    ▼
Ride Service
    |
    ├── PostgreSQL (Ride + Outbox)
    |
    ▼
Message Broker
    |
    ▼
Saga Orchestrator
    |
    ├───────────────┬────────────────┐
    ▼               ▼                ▼
Driver Service  Payment Service  Notification
    |               |                |
 Driver DB       Payment DB      Notification DB
</pre>

Every service owns its own data and local transaction boundaries.

The Saga coordinates the business process across those boundaries.

---

## 16. Common Mistakes

1. Treating a Saga as one large ACID transaction.
2. Assuming compensating actions perfectly undo previous operations.
3. Forgetting that notifications and external side effects may not be reversible.
4. Retrying non-idempotent operations without checking their status.
5. Publishing events after a database commit without an outbox or equivalent reliability mechanism.
6. Assuming a message broker guarantees exactly-once business processing.
7. Keeping Saga state only in memory.
8. Ignoring compensation failures.
9. Building a complex orchestrator for a workflow that needs only one local transaction.
10. Failing to expose intermediate workflow states to users and operators.

---

## 17. Practical Checklist

### Workflow design
- [ ] Identify each local transaction.
- [ ] Define the Saga's successful terminal state.
- [ ] Define failure and compensation paths.
- [ ] Identify actions that cannot be reversed.
- [ ] Define timeouts and workflow deadlines.

### Reliability
- [ ] Use idempotency keys for retryable business operations.
- [ ] Use an outbox or equivalent reliable event-publication mechanism.
- [ ] Make consumers idempotent.
- [ ] Persist workflow state.
- [ ] Define retry and dead-letter handling.
- [ ] Recover unfinished Sagas after restart.

### Observability
- [ ] Include a correlation ID and Saga ID.
- [ ] Track each workflow step.
- [ ] Monitor workflows stuck in intermediate states.
- [ ] Track retry and compensation failures.
- [ ] Provide a safe manual recovery process.

---

## 18. Revision Questions

1. Why can't a normal database transaction coordinate multiple independent services?
2. What are the two phases of 2PC?
3. What is the Saga pattern?
4. What is a compensating transaction?
5. What is the difference between Saga choreography and orchestration?
6. Why is compensation not the same as rollback?
7. Why is idempotency important in distributed workflows?
8. What problem does the transactional outbox solve?
9. What problem does the inbox pattern solve?
10. Why should Saga state be persisted?
11. What happens if an outbox publisher crashes after publishing but before marking an event as published?
12. When would you choose 2PC instead of Saga?

---

## 19. Practice Exercise — Design a Reliable Booking Workflow

Design a distributed booking workflow with these services:

- Ride Service
- Driver Service
- Payment Service
- Notification Service

Requirements:

- A passenger requests a ride.
- A driver must be reserved.
- Payment must be authorized.
- The ride is confirmed only after successful authorization.
- If payment fails, the driver must be released.
- The passenger must receive a final status notification.
- The workflow must recover after an orchestrator restart.

Answer these questions:

1. Would you choose choreography or orchestration? Why?
2. What are the local transactions?
3. What compensation is needed if payment fails?
4. How would you prevent duplicate payment authorization?
5. How would you ensure `RideRequested` is not lost after the ride database commits?
6. How would you handle duplicate `DriverReserved` events?
7. Where would you persist Saga state?
8. What happens if releasing the driver fails?
9. How would you identify workflows that remain stuck for too long?

### Bonus interview question

"Your payment provider successfully charges the customer, but the response times out. The Saga orchestrator thinks payment failed. How would you prevent the system from charging the customer again or incorrectly cancelling a paid ride?"

Think about idempotency keys, payment-status lookup, durable state, and compensation.

### What to focus on

The most important concepts in this lesson are:
- 2PC vs. Saga: coordinated atomic commit versus independently committed local transactions.
- Compensation: reversing business effects is not the same as rolling back a database transaction.
- Outbox + Inbox: reliable event publication and duplicate-safe consumption.
- Idempotency: essential when operations can be retried.
- Durable Saga state: workflows must survive service restarts.