# Lesson 19 — Message Brokers Deep Dive: Kafka, RabbitMQ, Partitions, Consumer Groups, and Delivery Guarantees

In Lesson 7, we introduced message queues and asynchronous processing. In Lesson 15, we used queues in distributed transactions and the Saga pattern. In Lesson 18, we used Pub/Sub to distribute real-time events.

Now we'll go deeper into how message brokers work internally and how to choose between technologies such as Kafka and RabbitMQ.

The central question is:

How do we reliably process millions of events across multiple consumers without losing messages, processing everything twice, or creating bottlenecks?

## 1. What Is a Message Broker?

A message broker is infrastructure that enables different application components to communicate asynchronously.

Instead of one service directly calling another service, the producer sends a message to a broker. The broker delivers it to the appropriate consumer or consumers.

### Without a message broker

```text
Ride Service
     |
     ├────────> Notification Service
     |
     ├────────> Analytics Service
     |
     └────────> Billing Service
```

The Ride Service directly depends on multiple downstream services.

If Analytics Service is unavailable, the Ride Service may need additional failure-handling logic.

### With a message broker

```text
                  Ride Service
                       |
                       | Publish event
                       ▼
                  Message Broker
                       |
             ┌─────────┼─────────┐
             |         |         |
             ▼         ▼         ▼
        Notification Analytics  Billing
          Service    Service    Service
```

Now the Ride Service only needs to publish an event.

The broker manages delivery to consumers.

### Important terminology

| Term | Meaning |
|---|---|
| Producer | Application that publishes messages |
| Consumer | Application that processes messages |
| Broker | Infrastructure that receives and delivers messages |
| Queue | A collection of messages waiting for processing |
| Topic | A named stream or category of messages |
| Partition | A subdivision of a topic |
| Offset | Position of a message within a partition |
| Acknowledgement | Confirmation that a message was processed or received |
| Consumer group | Consumers cooperating to process a topic's partitions |

---

## 2. Queue vs. Pub/Sub

Two common messaging models are queue-based messaging and publish-subscribe messaging.

### 2.1 Queue model

A producer sends messages to a queue.

Consumers retrieve messages from that queue.

```text
Producer
   |
   ▼
 Queue
   |
   ├──────> Consumer A
   |
   └──────> Consumer B
```

In a competing-consumer model, each message is normally delivered to one consumer for processing.

Example:

A booking service publishes 1,000 email jobs.

Three email workers share the workload.

```text
              Email Queue
                   |
          ┌────────┼────────┐
          |        |        |
          ▼        ▼        ▼
        Worker 1 Worker 2 Worker 3
```

The broker distributes work among consumers according to its delivery and acknowledgement rules.

### 2.2 Publish-subscribe model

In Pub/Sub, publishers publish events to a topic.

Multiple independent subscribers can receive their own copy of the event.

```text
                Producer
                    |
                    ▼
                  Topic
                    |
          ┌─────────┼─────────┐
          |         |         |
          ▼         ▼         ▼
      Consumer A Consumer B Consumer C
```

Example:

When a ride is created:

- Notification Service sends a notification.
- Analytics Service records the event.
- Billing Service processes billing-related work.

All three need to receive the event independently.

### Queue vs. Pub/Sub

| Consideration | Queue | Pub/Sub |
|---|---|---|
| Primary purpose | Distribute work | Distribute events |
| Message handling | Usually one competing consumer handles a message | Multiple subscriptions can receive the same event |
| Typical use | Email jobs, image processing | Ride-created events, analytics, notifications |
| Consumer independence | Consumers share work | Subscribers independently process events |

The exact behavior depends on the messaging technology. Kafka, for example, supports both competing consumers within a group and independent consumer groups.

---

## 3. Apache Kafka

<Entity value="Apache Kafka" category="software" disambig="Distributed event streaming platform"/> is a distributed event-streaming platform.

It is designed to handle high-volume event streams, retain events for configured periods, and allow multiple independent consumers to read them.

Kafka is particularly useful when:

- Event throughput is high.
- Events need to be retained.
- Multiple consumer applications need the same events.
- Consumers need to replay historical events.
- Processing needs to scale horizontally.

### Kafka architecture

```text
                  Producers
                      |
                      ▼
                 Kafka Cluster
                      |
          ┌───────────┼───────────┐
          |           |           |
          ▼           ▼           ▼
       Broker 1    Broker 2    Broker 3
          |           |           |
          └───────────┼───────────┘
                      |
                 Consumers
                      |
          ┌───────────┼───────────┐
          |           |           |
          ▼           ▼           ▼
       Group A     Group B     Group C
```

A Kafka cluster consists of multiple brokers.

Each broker stores some of the cluster's partitions.

---

## 4. Kafka Topics and Partitions

A topic is a logical category of events.

Examples:

```text
ride-created
ride-updated
driver-location
payment-completed
notification-requested
```

A topic can be divided into partitions.

Why?

Because a single partition limits the parallelism available for consuming that partition.

Partitions allow Kafka to distribute storage and processing across brokers.

### Example

Suppose we create a topic:

```text
driver-location
```

With three partitions:

```text
              driver-location
                      |
          ┌───────────┼───────────┐
          |           |           |
          ▼           ▼           ▼
      Partition 0 Partition 1 Partition 2
```

Each partition is an ordered sequence of records.

Example:

```text
Partition 0: [M1, M4, M7, M10]
Partition 1: [M2, M5, M8, M11]
Partition 2: [M3, M6, M9, M12]
```

The broker stores records in an append-oriented log.

### Important: Ordering

Kafka guarantees ordering within a partition, not across all partitions of a topic.

If two events must be processed in order for the same driver, publish them using the same key.

Example:

```text
Key: driver-101
Event: LocationUpdated
```

Kafka uses the key to determine the partition according to the configured partitioning strategy.

Therefore, events with the same key normally go to the same partition and retain their partition order.

This is extremely useful for:

- Driver location updates.
- Ride status changes.
- Account events.
- Order lifecycle events.

However, adding partitions or changing partitioning behavior can affect key-to-partition mapping. Plan partition changes carefully when ordering is important.

---

## 5. Kafka Offsets

An offset identifies a record's position within a partition.

Example:

```text
Partition 0

Offset 0 → RideCreated
Offset 1 → RideAssigned
Offset 2 → RideUpdated
Offset 3 → RideCompleted
```

A consumer tracks its progress using offsets.

Suppose a consumer has processed through offset 2.

Its next record is offset 3.

### Why are offsets important?

Offsets allow consumers to:

- Resume after a restart.
- Track processing progress.
- Replay historical events.
- Maintain independent progress across consumer groups.

### Committing offsets

Consumers periodically commit their processed offsets.

There are two important approaches.

**Automatic offset commit**

The consumer library commits offsets automatically according to its configuration.

This is convenient, but it can create problems if offsets are committed before processing actually succeeds.

**Manual offset commit**

The application commits offsets after processing.

This gives more control over processing and failure handling.

### Example failure

1. Consumer reads offset 10.
2. Consumer processes the event.
3. Consumer crashes before committing offset 10.
4. Consumer restarts.
5. Offset 10 is read again.

The event is processed twice.

Therefore, Kafka consumers should be designed to handle duplicate processing safely.

---

## 6. Kafka Consumer Groups

Consumer groups allow multiple consumer instances to share the processing of a topic.

Example:

A topic has six partitions.

We create a consumer group with three consumers.

```text
              Topic: ride-events
                    |
        ┌───────────┼───────────┐
        |           |           |
        ▼           ▼           ▼
       P0          P1          P2
        |           |           |
        └──────┐    |    ┌──────┘
               ▼    ▼    ▼
             Consumer Group
             ┌─────┬─────┬─────┐
             │ C1  │ C2  │ C3  │
             └─────┴─────┴─────┘
```

A simplified assignment might be:

- Consumer 1: P0 and P1.
- Consumer 2: P2 and P3.
- Consumer 3: P4 and P5.

### Important rule

Within one consumer group, a partition is assigned to at most one consumer at a time.

This preserves ordered processing within that partition.

### What if we add more consumers?

Suppose there are six partitions and six consumers.

Each consumer can receive one partition.

If we add a seventh consumer, it cannot receive another partition from this topic because all six are already assigned.

This leads to an important rule:

**Maximum useful consumer parallelism within a consumer group is limited by the number of partitions.**

Example:

| Partitions | Consumers | Maximum active consumers |
|---:|---:|---:|
| 3 | 2 | 2 |
| 3 | 3 | 3 |
| 3 | 5 | 3 |
| 6 | 4 | 4 |
| 6 | 8 | 6 |

The remaining consumers may be idle.

---

## 7. Kafka Replication and Fault Tolerance

Kafka replicates partitions across brokers to tolerate failures.

Suppose a topic has three partitions and replication factor 3.

Each partition has three copies distributed across brokers.

### Leader and followers

For each partition:

- One replica is the leader.
- Other replicas are followers.

Producers and consumers normally interact with the partition leader.

Followers replicate data from the leader.

```text
              Partition 0
                   |
              Leader: B1
                /     \
               ▼       ▼
          Follower B2  Follower B3
```

If the leader broker fails, Kafka can elect another eligible replica as leader.

### Replication factor

Replication factor determines the number of copies of a partition.

For example:

- Replication factor 1: one copy.
- Replication factor 2: two copies.
- Replication factor 3: three copies.

A higher replication factor improves fault tolerance but increases storage and replication traffic.

### Important configurations

**min.insync.replicas**

Defines the minimum number of replicas that must be in sync for certain acknowledged writes to succeed when using the appropriate producer acknowledgement setting.

**acks=all**

The producer waits for acknowledgement from the current in-sync replica set, subject to the broker configuration.

For example:

- Replication factor: 3.
- `min.insync.replicas`: 2.
- Producer `acks`: `all`.

A write generally requires at least two in-sync replicas to acknowledge it under these settings.

This improves durability but can reduce write availability when too few replicas are healthy.

---

## 8. Kafka Retention and Replay

A traditional queue often removes a message after successful processing.

Kafka works differently.

Kafka retains records according to configured retention policies.

Example:

```text
Topic: ride-events

Retention: 7 days
```

Consumers maintain their own offsets.

This means:

- Consumer A can process the event today.
- Consumer B can process it independently.
- Consumer C can replay it later, provided the record is still retained.

### Why replay is useful

Suppose Analytics Service has a bug and calculates ride revenue incorrectly.

After fixing the bug, the analytics team may reset its consumer offset and reprocess historical events.

This is a powerful capability.

### Retention vs. compaction

Kafka supports different log-cleanup strategies.

**Delete retention**

Old records are removed based on time or size policies.

**Log compaction**

For a keyed topic, Kafka retains the latest value for each key over time, subject to compaction and tombstone behavior.

Example:

```text
driver-101 → Location A
driver-101 → Location B
driver-101 → Location C
```

After compaction, the latest value may remain:

```text
driver-101 → Location C
```

Compaction is useful for maintaining the latest state of keyed entities, but it is not the same as an ordinary cache.

---

## 9. RabbitMQ

<Entity value="RabbitMQ" category="software" disambig="Message broker"/> is a message broker commonly used for queue-based messaging and routing.

It is particularly useful for:

- Background jobs.
- Task distribution.
- Request-driven asynchronous workflows.
- Flexible message routing.
- Work queues with acknowledgements and retries.

### RabbitMQ architecture

RabbitMQ commonly uses:

- Producers.
- Exchanges.
- Queues.
- Bindings.
- Consumers.

```text
Producer
   |
   ▼
Exchange
   |
   ▼
Binding / Routing
   |
   ▼
 Queue
   |
   ▼
Consumer
```

### What is an exchange?

An exchange receives messages from producers and routes them to queues according to its type and binding rules.

Unlike a simple queue, an exchange allows flexible routing.

---

## 10. RabbitMQ Exchange Types

There are four commonly used exchange types.

### 10.1 Direct exchange

Routes messages based on an exact routing-key match.

Example:

```text
Exchange: ride-events

Routing key: ride.created
    |
    ▼
Ride Created Queue
```

A queue bound with `ride.created` receives messages with that exact routing key.

Useful for routing messages by event type or task category.

### 10.2 Topic exchange

Routes messages based on patterns in routing keys.

Example routing keys:

```text
ride.created
ride.cancelled
driver.location.updated
payment.completed
```

Bindings can use patterns:

```text
ride.*
```

This matches:

- `ride.created`
- `ride.cancelled`

Another pattern:

```text
driver.#
```

This matches routing keys beginning with `driver.` and additional segments.

Useful for flexible event subscriptions.

### 10.3 Fanout exchange

Broadcasts messages to all queues bound to the exchange.

```text
                Fanout Exchange
                       |
          ┌────────────┼────────────┐
          |            |            |
          ▼            ▼            ▼
       Queue A       Queue B      Queue C
```

Routing keys are ignored.

Useful when multiple independent consumers need the same event.

### 10.4 Headers exchange

Routes messages based on message headers rather than routing keys.

Useful when routing depends on multiple message attributes.

### Exchange comparison

| Exchange | Routing behavior |
|---|---|
| Direct | Exact routing-key match |
| Topic | Pattern-based routing |
| Fanout | Broadcast to bound queues |
| Headers | Header-based matching |

---

## 11. RabbitMQ Acknowledgements

RabbitMQ consumers can acknowledge messages after processing.

Example:

```text
RabbitMQ
    |
    ▼
Consumer receives message
    |
    ▼
Process message
    |
    ├── Success ──> ACK
    |
    └── Failure ──> NACK / Reject
```

### ACK

The consumer confirms successful processing.

The broker can then remove the message from the queue according to its queue semantics.

### NACK / Reject

The consumer indicates that it cannot process the message.

Depending on configuration, the message may be:

- Requeued.
- Rejected without requeue.
- Routed to a dead-letter exchange if configured.

### What if the consumer crashes?

Suppose:

1. RabbitMQ delivers a message.
2. Consumer receives it.
3. Consumer crashes before acknowledgement.

The broker can requeue and redeliver the unacknowledged message.

This is why consumers must be idempotent.

---

## 12. RabbitMQ Prefetch and Fair Distribution

Imagine three consumers processing jobs.

Consumer A is slow.

Consumer B is fast.

If the broker sends too many messages to Consumer A, work can become unevenly distributed.

RabbitMQ supports a prefetch limit through consumer acknowledgements.

Example:

```text
Prefetch = 1
```

A consumer receives one unacknowledged message at a time.

After acknowledging it, the consumer can receive another.

This can help distribute work more fairly.

However, very low prefetch can reduce throughput for some workloads.

### Choosing prefetch

- Small prefetch: lower consumer memory pressure and potentially fairer distribution.
- Large prefetch: higher throughput in some workloads but more unacknowledged work per consumer.
- Excessively large prefetch: can create uneven workload and memory pressure.

Tune prefetch based on message size, processing time, and consumer capacity.

---

## 13. Kafka vs. RabbitMQ

Both are messaging technologies, but their architectural models differ.

| Consideration | Kafka | RabbitMQ |
|---|---|---|
| Core model | Distributed event log | Message broker with exchanges and queues |
| Storage | Retained partition logs | Queue-based message storage |
| Consumer progress | Offsets | Message acknowledgements |
| Replay | Native offset-based replay within retention | Not the typical queue-consumption model |
| Ordering | Within a partition | Queue-level ordering, subject to concurrency and requeue behavior |
| Routing | Topics and partitions, often with application-side routing | Flexible exchanges and bindings |
| Parallelism | Consumer groups and partitions | Multiple competing consumers |
| Typical workload | Event streaming and data pipelines | Background tasks and routed messages |
| Retention | Configurable retention and compaction | Queue lifecycle and message policies |
| Best fit | High-volume retained event streams | Flexible task and message distribution |

### When to choose Kafka

- Large event streams.
- Multiple independent consumers.
- Event replay.
- Long-term event retention.
- High-throughput data pipelines.
- Stream processing.

### When to choose RabbitMQ

- Background task queues.
- Complex message routing.
- Work distribution.
- Request-driven asynchronous processing.
- Flexible acknowledgement and retry workflows.

Neither is universally better.

Also, both can be deployed in highly available distributed configurations. Compare the actual durability, routing, ordering, and recovery requirements rather than relying only on product labels.

---

## 14. Delivery Guarantees

One of the most important messaging concepts is delivery semantics.

There are three common categories.

### 14.1 At-most-once

A message is delivered zero or one times.

It may be lost, but duplicate delivery is avoided under the defined delivery behavior.

Example:

A consumer commits its offset before completing processing.

If it crashes after the commit but before the business update, the message may not be processed again.

### 14.2 At-least-once

A message is delivered one or more times.

It should not be lost under the system's configured durability and retry assumptions, but duplicates are possible.

Example:

1. Consumer processes an event.
2. Consumer crashes before acknowledgement.
3. Broker redelivers the event.
4. Consumer processes it again.

This is a common delivery model in distributed systems.

### 14.3 Exactly-once

Exactly-once behavior is more complicated.

A broker may provide exactly-once guarantees within a specific processing boundary, but that does not automatically make external side effects exactly once.

For example:

- A Kafka transaction can atomically publish output records and commit consumed offsets within Kafka.
- It cannot automatically guarantee that an external payment provider charges a customer exactly once.

For business operations, combine idempotency, durable state, and appropriate transaction boundaries.

### Comparison

| Delivery model | Possible loss | Possible duplicates |
|---|---|---|
| At-most-once | Yes | No, under the defined semantics |
| At-least-once | Not under the configured delivery assumptions | Yes |
| Exactly-once | Depends on the defined transaction boundary | Avoided within that boundary |

**Interview takeaway:** Exactly-once processing is not a universal property of a distributed application. Always identify exactly which operations and systems are covered by the guarantee.

---

## 15. Idempotent Consumers

Because at-least-once delivery can produce duplicates, consumers should be idempotent.

An operation is idempotent if repeating it does not create additional unintended effects.

Example event:

```json
{
  "eventId": "evt-1001",
  "type": "RideCompleted",
  "rideId": "ride-501"
}
```

The consumer maintains a processed-events table.

```text
processed_events
----------------
event_id
processed_at
```

Processing flow:

1. Receive event.
2. Begin a database transaction.
3. Check whether the event ID has already been processed.
4. If yes, skip the business operation.
5. If no, apply the business update.
6. Insert the event ID.
7. Commit.
8. Acknowledge the message.

The business update and deduplication record must be committed atomically.

Otherwise, a crash between those operations can still cause duplicate effects.

### Example

Suppose an analytics consumer increments completed rides.

Without deduplication:

```text
RideCompleted received twice
          |
          ▼
Completed ride count += 2
```

With deduplication:

```text
RideCompleted received twice
          |
          ▼
Check event ID
          |
          ▼
Already processed?
          |
          ▼
Skip duplicate
```

---

## 16. Dead-Letter Queues

Some messages repeatedly fail.

Example:

A consumer receives a message with an invalid payload.

It retries several times, but the error never goes away.

Continuously retrying that message wastes resources and can block useful work.

A Dead-Letter Queue (DLQ) stores messages that cannot be successfully processed under the configured failure policy.

```text
             Main Queue
                  |
                  ▼
               Consumer
                  |
          ┌───────┴────────┐
          |                |
          ▼                ▼
        Success          Failure
          |                |
          ▼                ▼
       Complete       Retry Policy
                           |
                     Retry Exhausted
                           |
                           ▼
                          DLQ
```

### What should happen to DLQ messages?

- Store the original payload.
- Record failure details.
- Preserve correlation IDs.
- Alert the operations team.
- Allow controlled investigation.
- Support safe replay after fixing the issue.

A DLQ should not become a permanent dumping ground.

---

## 17. Ordering Guarantees

Sometimes events must be processed in a specific order.

Example:

```text
RideCreated
RideAssigned
RideStarted
RideCompleted
```

If `RideCompleted` is processed before `RideStarted`, downstream state may become incorrect.

### Kafka ordering

Kafka preserves record order within a partition.

Use a stable key such as:

```text
ride-501
```

This helps ensure events for the same ride go to the same partition.

### RabbitMQ ordering

RabbitMQ queues generally deliver messages in queue order, but multiple consumers, redelivery, priorities, and requeue behavior can affect observed processing order.

If strict ordering matters:

- Use a single active consumer for the relevant queue where appropriate.
- Partition work by entity.
- Avoid concurrent processing of dependent events.
- Include sequence numbers or versions.
- Validate transitions in the consumer.

**Important:** Message order and business-state correctness are separate concerns. Consumers should validate that incoming events are valid for the current entity state.

---

## 18. Handling Consumer Backlogs

A message backlog occurs when messages arrive faster than consumers can process them.

Suppose:

- Incoming rate: 5,000 messages/second.
- Processing rate: 4,000 messages/second.

Backlog growth:

\[
5,000 - 4,000 = 1,000 \ messages/second
\]

After one minute:

\[
1,000 \times 60 = 60,000 \ messages
\]

The backlog grows by 60,000 messages per minute.

### How to handle it

1. Add consumers where parallelism permits.
2. Increase partitions if Kafka parallelism is insufficient and the workload allows it.
3. Optimize consumer processing.
4. Batch database writes where appropriate.
5. Reduce unnecessary event volume.
6. Apply backpressure to producers when possible.
7. Prioritize critical events if the business supports it.

### Monitor more than queue depth

Useful metrics include:

- Queue depth.
- Oldest message age.
- Consumer processing rate.
- Incoming message rate.
- Failed processing rate.
- Retry rate.
- Consumer lag.
- DLQ volume.

For Kafka, consumer lag measures how far a consumer group is behind the latest available records.

A growing lag indicates that consumers are not keeping up.

---

## 19. Partitioning Strategy

Partitioning is essential for Kafka scalability.

The partition key determines how records are distributed.

Examples:

| Key | Benefit | Potential risk |
|---|---|---|
| `rideId` | Preserves ride-specific ordering | A very active ride could become a hotspot |
| `driverId` | Preserves driver-specific ordering | Uneven driver traffic can create imbalance |
| `tenantId` | Supports tenant-level grouping | A large tenant may create a hot partition |
| Random key | Distributes load | Loses entity-level ordering |

### Example: Driver location

Suppose driver 101 sends:

```text
LocationUpdate 1
LocationUpdate 2
LocationUpdate 3
```

Using `driverId` as the partition key ensures these events are routed to the same partition under a stable partitioning configuration.

This makes per-driver ordering possible.

However, if the number of partitions is changed, the mapping may change. Consider the impact on ordering during partition expansion and consumer transitions.

---

## 20. Message Schema Evolution

Message contracts evolve over time.

Version 1:

```json
{
  "rideId": "ride-501",
  "status": "assigned"
}
```

Version 2:

```json
{
  "rideId": "ride-501",
  "status": "assigned",
  "driverId": "driver-101"
}
```

If an old consumer cannot handle the new schema, processing may fail.

### Good practices

- Add optional fields where possible.
- Avoid changing existing field meanings.
- Avoid removing fields immediately.
- Use explicit schema versions when appropriate.
- Test old consumers against new producer messages.
- Consider schema registries for large event ecosystems.

A schema registry can help manage event contracts and enforce compatibility rules.

Backward-compatible schema evolution is especially important when different services deploy independently.

---

## 21. End-to-End Architecture: Ride Event Processing

Let's design a ride event pipeline.

Requirements:

- Ride Service publishes ride lifecycle events.
- Notification Service sends notifications.
- Analytics Service processes reporting events.
- Search Service updates its read model.
- Consumers can recover after failures.

### Architecture

```text
                  Ride Service
                       |
                       ▼
                  PostgreSQL
                 ┌──────────┐
                 │ Rides    │
                 │ Outbox   │
                 └──────────┘
                       |
                       ▼
                 Outbox Worker
                       |
                       ▼
                   Kafka
                       |
             ┌─────────┼─────────┐
             |         |         |
             ▼         ▼         ▼
        Notification Analytics Search
          Group       Group     Group
             |         |         |
             ▼         ▼         ▼
          Notify     Reports   Read Model
```

### Why Kafka?

- Multiple independent consumer groups.
- Event retention.
- Replay capability.
- Partition-based parallelism.
- High-volume event handling.

### Example failure

Analytics Service crashes.

Notification Service continues processing its events.

Search Service continues updating its read model.

Analytics Service restarts and resumes from its committed offset.

This independence is one of Kafka's major architectural advantages.

---

## 22. Choosing Between Kafka and RabbitMQ: Practical Examples

| Scenario | Suggested starting point | Reason |
|---|---|---|
| Sending 100,000 emails asynchronously | RabbitMQ or managed queue | Work distribution |
| Processing ride lifecycle events for multiple services | Kafka | Retention and independent consumers |
| Image-processing background jobs | RabbitMQ or managed queue | Task distribution and acknowledgements |
| Driver location event stream | Kafka or specialized streaming service | High-volume ordered event stream |
| Payment workflow commands | RabbitMQ or managed queue | Command delivery and retries |
| Historical event replay | Kafka | Retained event log |
| Complex routing based on event type | RabbitMQ | Exchange and binding model |
| Large analytics pipelines | Kafka | Event streaming ecosystem |

These are starting recommendations, not strict rules. Both products support more capabilities than shown in this comparison.

---

## 23. Common Mistakes

1. Assuming Kafka and RabbitMQ are interchangeable.
2. Assuming a consumer receives each message exactly once.
3. Committing Kafka offsets before successful processing without understanding the loss risk.
4. Forgetting idempotency when processing at-least-once messages.
5. Adding consumers without considering partition count.
6. Assuming Kafka guarantees ordering across an entire topic.
7. Using random partition keys when entity-level ordering is required.
8. Ignoring message retention and replay requirements.
9. Retrying permanently invalid messages indefinitely.
10. Allowing DLQs to accumulate without monitoring.
11. Publishing events without considering the database-to-broker consistency gap.
12. Changing event schemas without backward compatibility.
13. Monitoring queue depth but ignoring oldest message age.
14. Assuming broker-level exactly-once guarantees cover external database and payment side effects.

---

## 24. Practical Checklist

### Architecture
- [ ] Decide whether the workload is task distribution or event streaming.
- [ ] Select the broker based on durability, routing, ordering, and replay requirements.
- [ ] Define topic and queue ownership.
- [ ] Define consumer groups or subscriptions.

### Reliability
- [ ] Use durable messages and appropriate replication settings.
- [ ] Define acknowledgement behavior.
- [ ] Implement idempotent consumers.
- [ ] Define retry limits and DLQ handling.
- [ ] Use transactional outbox where required.

### Performance
- [ ] Estimate incoming and outgoing message rates.
- [ ] Choose suitable partition counts.
- [ ] Select partition keys carefully.
- [ ] Monitor consumer lag and oldest message age.
- [ ] Tune batching and prefetch.

### Operations
- [ ] Monitor broker health and replication.
- [ ] Monitor queue depth and consumer failures.
- [ ] Define replay procedures.
- [ ] Test broker and consumer failures.
- [ ] Maintain backward-compatible event schemas.

---

## 25. Revision Questions

1. What is the difference between a message queue and Pub/Sub?
2. What is a Kafka topic?
3. Why does Kafka use partitions?
4. What is an offset?
5. What is a consumer group?
6. Why is the number of Kafka consumers limited by the number of partitions?
7. What is the role of a Kafka partition leader?
8. What does replication factor mean?
9. What is the purpose of `min.insync.replicas`?
10. How does Kafka replay work?
11. What are the four RabbitMQ exchange types?
12. What is RabbitMQ prefetch?
13. What is the difference between Kafka offsets and RabbitMQ acknowledgements?
14. Explain at-most-once, at-least-once, and exactly-once delivery.
15. Why are idempotent consumers important?
16. What is a dead-letter queue?
17. Why should related Kafka events use the same partition key?
18. What is consumer lag?
19. How would you handle a consumer that cannot keep up with incoming traffic?
20. Why does broker-level exactly-once processing not guarantee exactly-once payment execution?

---

## 26. Practice Exercise — Design a Messaging Architecture

Design a messaging system for a ride-booking platform.

Requirements:

- 100,000 active drivers.
- Each driver sends a location update every five seconds.
- Ride lifecycle events must be processed by multiple services.
- Notifications must be delivered asynchronously.
- Analytics must be able to replay historical events.
- A consumer failure must not stop other independent consumers.
- Duplicate events must not create duplicate business effects.

Answer these questions:

1. Would you use Kafka, RabbitMQ, or both? Explain.
2. How many incoming driver-location events are generated per second?
3. How would you partition the driver-location topic?
4. Which partition key would you choose?
5. How would you distribute events to Notification, Analytics, and Search services?
6. How would you handle duplicate messages?
7. What happens if an Analytics consumer falls behind?
8. How would you recover from a consumer crash?
9. How would you handle permanently invalid messages?
10. How would you evolve event schemas without breaking existing consumers?

### Bonus interview question

"Your Kafka topic has 12 partitions and 20 consumer instances in one consumer group. All consumers are healthy, but eight of them remain idle. Why? How would you improve processing throughput?"

Think about partition parallelism, consumer groups, batching, processing time, and partition-key distribution.

**Key takeaway:** A message broker is not just a place to put messages. It is part of your system's reliability, scaling, ordering, and recovery strategy. Understand the delivery guarantees and failure behavior before choosing the technology.

### Practice focus

Pay particular attention to these five concepts:
1. Kafka partitions and consumer groups.
2. Kafka offsets versus RabbitMQ acknowledgements.
3. At-least-once delivery and idempotent consumers.
4. Partition keys and ordering guarantees.
5. Consumer lag and backlog management.