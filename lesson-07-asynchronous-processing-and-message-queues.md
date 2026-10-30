# Lesson 7: Asynchronous Processing and Message Queues

## 1. What Is Asynchronous Processing?

Asynchronous processing means accepting work now and completing some or all of it later, without requiring the caller to wait for the entire operation to finish.

Consider a booking platform. When a user creates a booking, the system might need to:

1. Validate the booking.
2. Save the booking in PostgreSQL.
3. Send an email confirmation.
4. Notify a driver.
5. Update analytics.
6. Generate a reporting event.

Not every operation needs to finish before the user receives a response.

Saving the booking is part of the critical business operation. Sending an email and updating analytics may be completed later.

### Synchronous processing

```text
Client
  |
  v
Booking API
  |
  +----> Save booking
  |
  +----> Send email
  |
  +----> Update analytics
  |
  v
Return response
```

The client waits for all three operations to finish.

If the email provider is slow, the booking response becomes slow even if the booking itself was saved quickly.

### Asynchronous processing

```text
Client
  |
  v
Booking API
  |
  v
Save booking
  |
  v
Publish background work
  |
  v
Return response

Background worker
  |
  +----> Send email
  |
  +----> Update analytics
```

The user does not need to wait for the email and analytics tasks to finish.

This can improve response latency and isolate the booking API from noncritical dependencies.

**Important:** The diagram assumes that background work is accepted reliably. Simply sending a message after saving the booking can introduce a failure gap, which we will address with the transactional outbox pattern.

---

## 2. When Should You Use Asynchronous Processing?

Use asynchronous processing when the work:

- Does not need to finish before the user receives a response.
- Can be completed later without violating business requirements.
- Is expensive or slow relative to the request's latency budget.
- Can be retried safely.
- Can be processed independently by workers.
- Benefits from buffering traffic spikes.

Common examples include:

| Use case | Why asynchronous processing helps |
|---|---|
| Email and SMS notifications | External providers can be slow or unavailable. |
| Image or video processing | Processing may take seconds or minutes. |
| Analytics events | Analytics should not slow down the main transaction. |
| Report generation | Reports can take a long time to calculate. |
| Search indexing | Search data can be updated shortly after the source record changes. |
| Webhook delivery | Remote systems may temporarily reject requests. |
| Bulk imports | Large workloads can be processed in controlled batches. |

### When should you stay synchronous?

Keep an operation synchronous when the user needs its result to proceed safely.

For example, if a booking must be validated and persisted before the API can confirm success, that work normally belongs in the synchronous request path.

Similarly, if payment authorization is a prerequisite for confirming a booking, you cannot simply move it to a background queue and immediately report that the payment succeeded.

You may still design an asynchronous workflow, but its API must represent the operation as pending until the required business steps complete.

---

## 3. What Is a Message Queue?

A message queue is a mechanism for storing work until a consumer is ready to process it.

A producer creates messages. A queue holds them. A consumer retrieves and processes them.

```text
                 Producer
                    |
                    v
              +-----------+
              |   Queue   |
              |-----------|
              | Message A |
              | Message B |
              | Message C |
              +-----------+
                    |
                    v
                 Consumer
                    |
                    v
              Process result
```

For a booking platform:

- Producer: Booking service.
- Message: `BookingCreated`.
- Queue: Stores the event until it is processed.
- Consumer: Notification worker.
- Result: Email or SMS confirmation is sent.

### What does the queue provide?

A queue can provide:

1. **Buffering:** Work can wait while consumers catch up.
2. **Decoupling:** The producer does not need to call the consumer directly.
3. **Independent scaling:** Consumers can be scaled separately from the API.
4. **Retry support:** Failed processing can be attempted again.
5. **Durability:** Properly configured queues can preserve accepted messages across certain failures.
6. **Traffic smoothing:** A burst of incoming work can be processed at a sustainable rate.

These benefits depend on the queue technology, configuration, message persistence, acknowledgements, and failure-handling design. A queue does not automatically guarantee that no messages will be lost or duplicated.

---

## 4. Queue vs. Direct API Call

Suppose the booking API needs to notify an external notification service.

### Direct API call

```text
Booking API
    |
    v
Notification API
    |
    v
Email Provider
```

Potential problems:

- The booking API depends on the notification service's availability.
- Notification latency adds to the booking response time.
- Notification traffic consumes booking API resources.
- A notification outage can affect booking requests.

### Queue-based approach

```text
Booking API
    |
    v
Message Queue
    |
    v
Notification Worker
    |
    v
Email Provider
```

Benefits:

- The notification worker can process messages later.
- Temporary email-provider failures do not necessarily block booking creation.
- Workers can retry notification delivery.
- Notification processing capacity can be scaled independently.

However, the queue introduces additional infrastructure and operational concerns, such as backlog management, duplicate processing, ordering, and message retention.

### Important distinction

A queue improves decoupling only if the producer can safely hand off work without depending on the consumer finishing immediately.

It also does not eliminate the need to define what success means. An API that returns success after accepting a message is not necessarily claiming that the final email has already been delivered.

---

## 5. Understanding Queue Throughput and Backlog

A queue can absorb bursts, but it cannot compensate indefinitely for insufficient processing capacity.

Let:

- \(\lambda\) = incoming message rate, in messages per second.
- \(\mu\) = processing rate, in messages per second.

The approximate backlog growth rate is:

\[
\text{Backlog growth} = \lambda - \mu
\]

when incoming traffic exceeds processing capacity.

### Example 1: Processing keeps up

- Incoming rate: 100 messages/second.
- Processing rate: 120 messages/second.

The consumers can keep up with the average arrival rate, assuming sufficient capacity and no unusual bursts.

### Example 2: Backlog grows

- Incoming rate: 200 messages/second.
- Processing rate: 150 messages/second.

The backlog grows by:

\[
200 - 150 = 50
\]

messages per second.

Over one minute:

\[
50 \times 60 = 3000
\]

The queue accumulates approximately 3,000 additional messages per minute while these rates persist.

<text color="secondary" size="sm">These calculations assume steady arrival and processing rates and ignore retries, message size, and other overhead.</text>

### What if the backlog is already large?

Suppose the queue has 60,000 pending messages.

- Incoming rate: 100 messages/second.
- Processing rate: 150 messages/second.

The net drain rate is:

\[
150 - 100 = 50
\]

messages per second.

The approximate time to clear the backlog is:

\[
\frac{60000}{50} = 1200 \text{ seconds}
\]

That is approximately 20 minutes, assuming the rates remain steady.

This calculation is important because a queue may be processing messages successfully while users still experience significant delays.

### Queue stability

For a queue to remain stable over time, its long-term processing capacity must exceed its long-term incoming workload, with enough headroom for bursts and failures.

Simply adding a queue does not increase the processing capacity of the underlying system.

---

## 6. Queue Depth vs. Message Age

Queue depth tells you how many messages are waiting.

Message age tells you how long the oldest pending message has been waiting.

Both metrics are useful.

Consider two queues:

| Queue | Pending messages | Oldest message age |
|---|---:|---:|
| A | 10,000 | 2 seconds |
| B | 500 | 15 minutes |

Queue A may be processing a large burst efficiently. Queue B may have a stuck consumer or a small number of very slow tasks.

For time-sensitive workflows, the age of the oldest message may be more important than the total number of pending messages.

Monitor:

- Queue depth.
- Age of the oldest message.
- Incoming message rate.
- Successful processing rate.
- Processing latency.
- Retry rate.
- Dead-letter queue size.
- Consumer errors.
- Time from event creation to successful completion.

Define alert thresholds according to business requirements. For example, an email notification might tolerate a delay of a minute, while a driver assignment event might need much faster processing.

---

## 7. Message Delivery Semantics

Queues differ in how they deliver messages and handle failures.

Three important delivery models are at-most-once, at-least-once, and exactly-once.

### At-most-once delivery

A message is delivered zero or one times.

It may be lost if a failure occurs at the wrong point.

This can be acceptable for some noncritical events, depending on the business requirements, but it is usually unsuitable when every event must be processed.

### At-least-once delivery

A message is delivered one or more times until the system considers it successfully processed, according to the broker's delivery and acknowledgement rules.

Duplicates are possible.

This is common in reliable background-processing systems because unacknowledged messages can be redelivered.

### Exactly-once delivery or processing

The phrase "exactly once" can mean different things depending on the technology and scope.

A queue or stream platform may provide exactly-once guarantees within a defined boundary, such as a particular transaction or stream-processing operation. That does not automatically guarantee exactly-once effects across an external payment provider, email service, or separate database.

In business applications, a practical goal is often:

**Allow messages to be delivered more than once, but ensure that processing them repeatedly does not create duplicate business effects.**

That requires idempotent consumers and appropriate coordination with the database or external service.

---

## 8. Idempotent Consumers

An idempotent consumer can process the same logical message more than once without applying the same business effect repeatedly.

Suppose a worker receives:

```text
Message ID: msg-101
Event: BookingCreated
Booking ID: booking-123
```

The worker sends a notification, then crashes before acknowledging the message.

The queue may deliver the same message again.

Without deduplication, the customer could receive two emails.

### A common design

1. Assign each message a stable identifier.
2. Store processed-message identifiers durably.
3. Check whether the message has already been processed.
4. Apply the business change and record processing atomically where possible.
5. Acknowledge the message only after the required processing has committed.

```text
Receive message
      |
      v
Check message ID
      |
      +---- Already processed ----> Acknowledge/skip
      |
      v
Process business operation
      |
      v
Commit effect and deduplication record
      |
      v
Acknowledge message
```

The exact order and transaction boundary depend on the broker and the operation being performed.

### Important limitation: external side effects

Suppose a worker calls an email provider successfully but crashes before recording that the email was sent.

On redelivery, it may send the email again.

A database transaction cannot ordinarily make an independent external email API call atomic with the database commit.

Possible mitigations include:

- Use an idempotency key supported by the external provider.
- Maintain a durable notification state machine.
- Use a provider with suitable deduplication capabilities.
- Accept and document that occasional duplicate notifications are possible.
- Reconcile uncertain outcomes when duplicate effects would be harmful.

Idempotency must cover the actual business effect, not just the act of reading a message.

---

## 9. Message Acknowledgements

An acknowledgement tells the queue or broker that a consumer has handled a message according to the delivery protocol.

Consider this sequence:

```text
1. Consumer receives message
2. Consumer processes message
3. Consumer commits its database transaction
4. Consumer acknowledges message
```

Why acknowledge after the required work is committed?

If the consumer acknowledges first and crashes before processing the message, the broker may consider the message complete even though the business operation never happened.

If the consumer commits the business operation but crashes before acknowledging, the broker may redeliver the message. This is why idempotent processing is important.

The right acknowledgement behavior depends on the queue's delivery model, transaction capabilities, and the business operation.

---

## 10. Retries and Dead-Letter Queues

Not every failed message should be retried forever.

Some failures are temporary:

- Network timeouts.
- A temporary database outage.
- Rate limiting.
- A remote service being unavailable.

Other failures may persist:

- Invalid message structure.
- Missing required business data.
- An unsupported event version.
- A permanent authorization problem.
- A bug that causes the same message to fail every time.

### A retry strategy

```text
Message received
      |
      v
Processing fails
      |
      v
Is failure retryable?
      |
   +--+--+
   |     |
  Yes    No
   |     |
   v     v
Retry   Dead-letter
with    queue or
backoff rejection
   |
   v
Retry limit reached?
   |
   +---- Yes ----> Dead-letter queue
```

A dead-letter queue (DLQ) stores messages that could not be processed successfully after the defined handling policy.

### What should happen to dead-letter messages?

A production system should support:

- Inspecting the original message and error details.
- Identifying the consumer and event type.
- Finding out how many attempts were made.
- Correcting the underlying problem.
- Replaying the message safely.
- Preventing a replay from duplicating business effects.
- Alerting on abnormal dead-letter growth.

A DLQ is not a place to forget messages. It is an operational mechanism for investigating and recovering failed work.

### Avoid retry storms

If every consumer retries immediately after a database outage, the database may receive a sudden wave of requests as it recovers.

Use bounded retries, backoff, jitter, concurrency controls, and appropriate circuit-breaker or pause behavior to prevent this.

---

## 11. Message Ordering

Queues do not universally guarantee that all messages will be processed in the order they were created.

Consider a booking:

```text
Event 1: BookingCreated
Event 2: BookingCancelled
```

If `BookingCancelled` is processed first and `BookingCreated` is processed later, a naive consumer might incorrectly recreate or reactivate the booking.

### Why ordering becomes difficult

Ordering may be affected by:

- Multiple producers.
- Multiple consumers.
- Parallel processing.
- Retries and redelivery.
- Partitioning.
- Different processing times.

### Possible strategies

**1. Preserve order per entity**

Partition or route messages using a key such as `bookingId`, so events for the same booking can be processed in order.

This can preserve ordering within the relevant partition or processing lane, depending on the technology and consumer configuration.

**2. Process sequentially**

Handle messages one at a time for a particular entity or workflow.

This simplifies ordering but can reduce throughput.

**3. Make consumers state-aware**

Include event identifiers, entity versions, or sequence numbers so consumers can detect stale or out-of-order updates.

For example, an update with version 8 should not overwrite a record that already reflects version 9.

**4. Design for commutative operations where possible**

If the business operation can be safely applied regardless of order, strict ordering may not be necessary.

### Trade-off

Strict global ordering can limit parallelism. Often, the real requirement is ordering per booking, customer, or other business entity rather than across the entire system.

---

## 12. The Database-to-Queue Consistency Problem

One of the most important problems in asynchronous architecture is ensuring that a committed business change produces the corresponding message.

Consider this design:

```text
1. Save booking in PostgreSQL
2. Publish BookingCreated to the queue
```

What if the database commit succeeds but publishing the message fails?

The booking exists, but the notification worker never learns about it.

Now consider the reverse order:

```text
1. Publish BookingCreated to the queue
2. Save booking in PostgreSQL
```

What if the message is published but the database transaction fails?

A consumer may process an event for a booking that does not exist.

These are consistency gaps between two separate systems.

### Why a normal database transaction may not solve this

A PostgreSQL transaction can atomically commit database changes. It cannot ordinarily atomically commit a separate message broker operation unless a supported distributed transaction protocol is used across both systems.

Distributed transactions introduce additional complexity and are not always supported or desirable.

A widely used alternative is the transactional outbox pattern.

---

## 13. Transactional Outbox Pattern

The transactional outbox pattern stores the business change and the event to be published in the same database transaction.

### Architecture

```text
             Booking API
                  |
                  v
        PostgreSQL transaction
          /                 \
         v                   v
  Bookings table       Outbox table
         \                   /
          \---- COMMIT ------/
                   |
                   v
           Outbox publisher
                   |
                   v
              Message queue
                   |
                   v
             Event consumer
```

### Step-by-step example

A booking is created.

Within one PostgreSQL transaction:

1. Insert the booking.
2. Insert an outbox record describing `BookingCreated`.
3. Commit both records together.

The transaction ensures that either both database records are committed or neither is.

A separate publisher then:

1. Finds unpublished outbox records.
2. Publishes an event to the message broker.
3. Marks the outbox record as published after the publish operation is confirmed.

If the publisher is temporarily unavailable, the outbox record remains in the database and can be published later.

### What if publishing succeeds but marking the record fails?

This is a critical edge case.

1. The publisher sends the event.
2. The broker accepts the message.
3. The publisher crashes before marking the outbox record as published.
4. The publisher restarts and sends the same event again.

The event can be published twice.

Therefore, the transactional outbox pattern typically provides reliable publication with possible duplicates, not automatic exactly-once business effects.

Use stable event IDs and idempotent consumers.

### Operational considerations

A production outbox design needs:

- A durable outbox table.
- A publisher that can resume after failures.
- Stable event IDs.
- Safe handling of concurrent publishers.
- Retry and backoff policies.
- Monitoring for old unpublished records.
- Cleanup or archival of published records.
- Schema versioning and event compatibility.
- Consumer-side idempotency.

The outbox publisher can use polling or a change-data-capture approach, depending on the architecture.

### When is an outbox useful?

Use it when a database change must reliably lead to an event, such as:

- Booking created → notify downstream systems.
- Order placed → update fulfillment.
- Payment recorded → initiate follow-up processing.
- Account created → send a welcome notification.
- Record updated → refresh a search index.

---

## 14. Queue-Based Architecture vs. Event-Driven Architecture

These concepts are related but different.

### Queue-based processing

A producer places work into a queue. A consumer processes it.

Example:

```text
Booking API
     |
     v
Email Queue
     |
     v
Email Worker
```

The emphasis is on distributing work to a worker.

### Event-driven architecture

A component emits an event describing something that happened. Other components react to that event.

Example:

```text
                BookingCreated
                       |
                Event infrastructure
                 /      |       \
                v       v        v
           Notification Analytics Search index
```

The emphasis is on reacting to events and reducing direct dependencies between components.

A queue may be used to deliver events to a particular consumer. A pub/sub system may distribute events to multiple independent subscribers.

The precise behavior depends on the messaging technology and configuration.

### Command vs. event

A command requests that something happen.

Example: `CreateBooking`.

An event records that something happened.

Example: `BookingCreated`.

A command may be rejected. An event should describe an established fact.

Keeping this distinction clear helps prevent consumers from treating an uncompleted request as a successful business outcome.

---

## 15. Choosing the Right Queue Technology

The choice depends on workload characteristics rather than popularity alone.

| Requirement | What to evaluate |
|---|---|
| Background jobs | Delayed retries, job state, scheduling, worker management |
| Durable business events | Persistence, acknowledgements, replay, delivery semantics |
| High-volume event streams | Throughput, partitioning, retention, consumer offsets |
| Multiple independent subscribers | Fan-out, subscription isolation, replay requirements |
| Per-entity ordering | Partition or key-based ordering guarantees |
| Long-running workflows | Durable workflow state, timers, compensation, recovery |
| Small application with simple tasks | Operational simplicity and integration with the existing stack |

### Do you always need a message broker?

No.

For a small application, a durable jobs table in PostgreSQL may be sufficient. A worker can claim pending jobs, process them, and update their status.

A dedicated broker becomes more attractive when throughput, independent consumers, retention, fan-out, or operational requirements justify the extra infrastructure.

Do not introduce a distributed messaging platform simply because asynchronous processing is useful.

---

## 16. Scaling Consumers

Consumers can often be scaled independently from API servers.

Suppose a queue receives 300 messages per second, while each worker processes 50 messages per second.

Ignoring overhead and uneven workload distribution:

\[
\text{Workers needed} =
\left\lceil \frac{300}{50} \right\rceil = 6
\]

Six workers can theoretically match the arrival rate.

However, operating exactly at theoretical capacity leaves little headroom for bursts, retries, slow messages, and worker failures.

If the system needs to process 300 messages per second while tolerating one worker failure, seven workers at 50 messages per second provide 350 messages per second of nominal capacity. If one worker fails, six remain and provide 300 messages per second.

This assumes the workload is evenly distributed and the database or external service can handle the resulting load.

### A critical limitation

Increasing the number of workers can make the system slower if all workers compete for the same constrained database or external API.

Consumer scaling must account for downstream capacity, not just CPU capacity.

---

## 17. End-to-End Example: Booking Platform

Consider a booking platform with the following requirements:

- Booking creation must be persisted before success is returned.
- Email notifications may be delayed.
- Analytics may be delayed.
- Downstream systems must eventually receive booking events.
- Duplicate message delivery must not create duplicate business effects.

### Proposed architecture

```text
                         Client
                           |
                           v
                       Booking API
                           |
                           v
                PostgreSQL transaction
                   /              \
                  v                v
             Bookings          Outbox events
                                    |
                                    v
                              Outbox publisher
                                    |
                                    v
                              Message broker
                             /              \
                            v                v
                    Notification worker   Analytics worker
                            |                |
                            v                v
                       Email provider    Analytics store
```

### Request flow

1. The client sends a booking request with an idempotency key.
2. The API validates the request.
3. PostgreSQL stores the booking and an outbox event in one transaction.
4. The API returns success after the transaction commits.
5. The outbox publisher sends the event to the broker.
6. Notification and analytics consumers process the event independently.
7. Consumers record their processing state or use another idempotency mechanism.
8. Failed deliveries are retried according to bounded policies.
9. Poison messages are moved to a dead-letter queue for investigation.
10. Monitoring detects growing backlog or increasing message age.

### What if the email provider fails?

The notification worker retries according to its policy. The queue holds pending work, while booking creation can continue as long as the booking service and its required dependencies remain healthy.

### What if the analytics worker fails?

Analytics processing can fall behind without necessarily blocking booking creation. The backlog must still be monitored and eventually processed.

### What if the outbox publisher crashes?

Committed outbox records remain available for publication after recovery.

### What if the same event is delivered twice?

Consumers use stable event identifiers and idempotent processing to avoid duplicate business effects.

This design separates the critical booking transaction from work that can happen later while addressing the consistency gap between PostgreSQL and the message broker.

---

## 18. Common Mistakes

1. **Putting every operation in a queue:** Some operations require a synchronous result before the system can safely confirm success.

2. **Assuming queues eliminate failures:** Messages can be lost, duplicated, delayed, or stuck depending on configuration and implementation.

3. **Ignoring backlog growth:** A queue can accumulate work indefinitely if the arrival rate consistently exceeds processing capacity.

4. **Claiming exactly-once effects without evidence:** Broker guarantees do not automatically cover external APIs or independent databases.

5. **Acknowledging before processing commits:** A crash can lose the business effect while the broker considers the message complete.

6. **Retrying indefinitely:** Poison messages consume resources and can block progress or cause repeated load.

7. **Ignoring ordering requirements:** Events can be processed in an order that produces an invalid business state.

8. **Publishing after a database commit without a recovery mechanism:** A crash can leave a committed record without its corresponding event.

9. **Assuming the outbox prevents duplicates:** The outbox closes the database-to-publication consistency gap but can still publish duplicate events.

10. **Scaling workers without considering downstream capacity:** More consumers can overload the database or external API.

11. **Treating a queued task as completed:** Message acceptance means the work has been accepted for processing, not necessarily finished.

12. **Ignoring event schema evolution:** Changes to message fields or meanings can break older consumers.

---

## 19. Practical Design Checklist

### Architecture
- [ ] Identify which operations must complete synchronously.
- [ ] Move only suitable work into asynchronous processing.
- [ ] Decide whether a queue, job table, or event stream fits the workload.
- [ ] Define what the API's success response actually guarantees.

### Reliability
- [ ] Configure appropriate durability and acknowledgement behavior.
- [ ] Use stable message IDs.
- [ ] Make consumers idempotent.
- [ ] Implement bounded retries and dead-letter handling.
- [ ] Define replay and recovery procedures.

### Consistency
- [ ] Identify gaps between database commits and event publication.
- [ ] Use an outbox when reliable publication of committed changes is required.
- [ ] Decide which events require ordering.
- [ ] Version event schemas and maintain consumer compatibility.

### Capacity and operations
- [ ] Estimate arrival and processing rates.
- [ ] Monitor queue depth and oldest-message age.
- [ ] Track processing latency, retries, failures, and dead-letter messages.
- [ ] Scale workers within downstream capacity limits.
- [ ] Test crashes at different points in the processing lifecycle.

---

## 20. Revision Questions

Answer these without looking at the notes.

1. What is the difference between synchronous and asynchronous processing?
2. When should booking creation remain synchronous?
3. What benefits does a message queue provide?
4. If 200 messages arrive per second and workers process 150 per second, how quickly does the backlog grow?
5. Why is queue depth not enough to measure processing health?
6. What is the difference between at-most-once and at-least-once delivery?
7. Why is exactly-once business processing difficult across multiple systems?
8. How does an idempotent consumer handle duplicate messages?
9. Why should a consumer usually acknowledge a message only after the required business operation commits?
10. What is a dead-letter queue, and how should failed messages be recovered?
11. Why can parallel consumers cause ordering problems?
12. What problem does the transactional outbox pattern solve?
13. Why can the outbox pattern still produce duplicate events?
14. What is the difference between a command and an event?
15. When would a PostgreSQL-backed job table be sufficient instead of a dedicated message broker?
16. Why can increasing consumer count overload a database?
17. Design an asynchronous notification system that can recover after a worker crash.

## Final Takeaway

Asynchronous processing separates the completion of critical work from the completion of work that can happen later.

Message queues help buffer traffic, isolate failures, and scale background processing. But reliable asynchronous systems also need delivery semantics, acknowledgements, idempotent consumers, retry policies, ordering decisions, monitoring, and recovery procedures.

When a database change must reliably produce an event, the transactional outbox pattern is a key design option.

**Remember:** A queue does not make a system automatically reliable. Reliability comes from how the entire message lifecycle is designed, from the original database transaction through delivery, processing, acknowledgement, retry, and recovery.


## Practice exercise
Imagine your booking platform saves a booking successfully, but the notification worker is temporarily down.
1. How should the system ensure the booking event is not lost if the application crashes after the database commit but before publishing the message?


2. If the worker sends the email but crashes before acknowledging the message, what could happen? How would you reduce the risk?


3. The queue receives 200 messages/second, but workers process only 150 messages/second. How much does the backlog grow in 2 minutes?


4. Would you increase the number of workers immediately if the queue backlog grows? What else would you check first?