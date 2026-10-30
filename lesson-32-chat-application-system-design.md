# Lesson 32 — System Design of a Chat Application: Messaging, Presence, Delivery Receipts, and Real-Time Communication

This lesson brings together several concepts we've already studied: WebSockets, message brokers, event-driven architecture, database design, distributed systems, and real-time communication.

We'll design a chat platform similar to WhatsApp or Slack, covering message delivery, conversation management, online presence, delivery receipts, group messaging, message ordering, offline users, and scaling.

I've also included the introduction, suggested filename, key takeaway, and practice focus inside the notes so everything stays together in one document.

## Introduction

A chat application looks simple from the user's perspective:

1. Open a conversation.
2. Write a message.
3. Press Send.
4. The other person receives it.

However, building this system at scale is challenging.

Consider what happens when:

- The recipient is offline.
- The sender loses network connectivity.
- The same message is submitted multiple times.
- A user sends messages from multiple devices.
- Thousands of users send messages simultaneously.
- A group contains thousands of members.
- A WebSocket server crashes.
- Messages arrive out of order.
- A user opens an old conversation and expects complete message history.

A production chat application must solve all these problems.

This lesson focuses on designing a reliable, scalable, real-time messaging platform.

---

## 1. Problem Statement

Design a chat application that allows users to communicate privately and in groups.

Examples of familiar products include:

- WhatsApp
- Telegram
- Slack
- Microsoft Teams

### Functional requirements

The application should support:

- User authentication.
- One-to-one conversations.
- Group conversations.
- Sending and receiving messages.
- Message history.
- Online and offline presence.
- Delivery receipts.
- Read receipts.
- Typing indicators.
- Multiple devices per user.
- Message notifications.
- Offline message delivery.
- Message editing and deletion.

### Non-functional requirements

| Requirement | Explanation |
|---|---|
| Low latency | Messages should arrive quickly |
| High availability | Users should be able to communicate during failures |
| Durability | Accepted messages should not be lost |
| Scalability | Support increasing users and message volume |
| Ordering | Preserve message order within a conversation |
| Consistency | Maintain reliable message and receipt state |
| Security | Protect conversations and user information |

### Scope

We'll initially design a text-based chat platform. Voice calls, video calls, and end-to-end encryption can be added as separate architectural concerns.

---

## 2. Capacity Estimation

Before designing the architecture, estimate the expected workload.

Suppose the platform has:

- 10 million daily active users.
- Each user sends 40 messages per day.
- Average message payload size is 1 KB.
- Peak traffic is 5 times the average.

### Daily message volume

\[
10,000,000 \times 40 = 400,000,000
\]

The platform receives 400 million messages per day.

### Average messages per second

\[
\frac{400,000,000}{86,400}
\approx 4,630
\]

Approximately 4,630 messages per second on average.

### Peak messages per second

\[
4,630 \times 5 = 23,150
\]

Approximately 23,150 messages per second at peak.

### Daily raw message storage

\[
400,000,000 \times 1KB = 400GB
\]

Approximately 400 GB per day, before database indexes, replication, metadata, and storage overhead.

### Annual raw storage

\[
400GB \times 365 = 146TB
\]

This is a substantial storage requirement.

In practice, compression, retention policies, attachment storage, replication, and message metadata must also be considered.

**Important:** Message traffic and connection traffic are different workloads. A chat platform may have millions of simultaneous WebSocket connections but only a fraction of those users actively sending messages at any moment.

---

## 3. High-Level Architecture

A chat system usually separates connection management, message processing, storage, and delivery.

```text
                  Web / Mobile Clients
                           |
                           v
                     Load Balancer
                           |
                           v
                   WebSocket Gateway
                           |
              +------------+------------+
              |            |            |
              v            v            v
          Gateway 1     Gateway 2    Gateway 3
              |            |            |
              +------------+------------+
                           |
                           v
                     Message Service
                           |
                           v
                      Message DB
                           |
                           v
                      Message Broker
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
         Delivery       Presence     Notification
         Service        Service       Service
             |             |             |
             v             v             v
       WebSocket       Presence       Push Provider
       Gateways        Store
```

### Main components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication and HTTP request routing |
| WebSocket Gateway | Maintain real-time client connections |
| Message Service | Validate, persist, and process messages |
| Message Database | Store message history |
| Message Broker | Distribute message events |
| Delivery Service | Route messages to recipients |
| Presence Service | Track online and offline state |
| Notification Service | Send push notifications |
| Object Storage | Store images, videos, and files |

These components can be deployed independently as the system grows.

---

## 4. WebSocket-Based Communication

Chat applications need two-way communication.

Traditional HTTP request-response requires clients to repeatedly request new messages.

WebSockets provide a persistent, bidirectional communication channel.

### HTTP polling

```text
Client ---> Server: Any new messages?
Client <--- Server: No messages

Client ---> Server: Any new messages?
Client <--- Server: No messages

Client ---> Server: Any new messages?
Client <--- Server: New message
```

This creates repeated requests.

### WebSocket communication

```text
Client <================> Server
         Persistent
         Connection
```

Both client and server can send messages over the established connection.

### Connection lifecycle

1. Client authenticates.
2. Client establishes a WebSocket connection.
3. Gateway validates the connection.
4. Gateway associates the connection with a user and device.
5. Client sends and receives messages.
6. Heartbeats help detect stale connections.
7. Client reconnects if the connection is lost.

### Why use WebSockets?

- Low-latency communication.
- Bidirectional messaging.
- Reduced repeated HTTP request overhead.
- Efficient real-time updates.

### Important limitation

A WebSocket connection is not a durable message store.

If the recipient disconnects, the application needs persistent message storage and a delivery mechanism to catch up after reconnection.

We covered WebSockets, SSE, and long polling in Lesson 18.

---

## 5. Connection Management at Scale

Suppose the application has one million concurrent users.

Maintaining one million connections on a single server is not practical.

Instead, use multiple WebSocket gateway instances.

```text
                 Load Balancer
                       |
          +------------+------------+
          |            |            |
          v            v            v
       Gateway 1    Gateway 2    Gateway 3
          |            |            |
       300K          350K          350K
     Connections  Connections  Connections
```

These are illustrative connection distributions, not universal server capacities.

### Connection registry

The platform needs to know which gateway currently owns a user's connection.

Example:

```text
user_connections

user_id | device_id | gateway_id | connected_at
------------------------------------------------
U101    | D1        | G1         | ...
U101    | D2        | G2         | ...
U102    | D1        | G3         | ...
```

A distributed store or registry can maintain this information.

When a message arrives for U101, the delivery system can locate both connected devices.

### Connection state

Useful connection information includes:

- User ID.
- Device ID.
- Gateway ID.
- Connection ID.
- Last heartbeat.
- Authentication/session metadata.

Avoid treating a single user as having only one connection. Users commonly have multiple active devices.

### Sticky sessions

A load balancer may use sticky sessions to keep a connection associated with the same gateway.

However, sticky sessions do not solve message routing across gateway instances.

The system still needs a way to route messages to the gateway that owns the recipient's connection.

---

## 6. Message Data Model

A basic message record could look like this:

```json
{
  "messageId": "M1001",
  "conversationId": "C101",
  "senderId": "U101",
  "messageType": "TEXT",
  "content": "Hello!",
  "createdAt": "2026-10-30T12:00:00Z",
  "sequenceNumber": 1001
}
```

### Suggested message table

```sql
CREATE TABLE messages (
    message_id UUID PRIMARY KEY,
    conversation_id UUID NOT NULL,
    sender_id UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    message_type VARCHAR(30) NOT NULL,
    content TEXT,
    created_at TIMESTAMP NOT NULL
);
```

A production schema may also include:

- Edited timestamp.
- Deleted timestamp.
- Attachment references.
- Reply-to message ID.
- Client-generated idempotency ID.
- Encryption metadata.
- Message version.

### Important index

A common query is:

```sql
SELECT *
FROM messages
WHERE conversation_id = 'C101'
ORDER BY sequence_number DESC
LIMIT 50;
```

A suitable composite index is:

```sql
CREATE INDEX idx_messages_conversation_sequence
ON messages (conversation_id, sequence_number DESC);
```

This helps retrieve recent messages efficiently.

The exact index strategy depends on the database and query patterns.

---

## 7. Conversation Data Model

A chat application needs to represent conversations and participants.

### Conversations table

```text
conversations
-------------
conversation_id
conversation_type
created_by
created_at
```

Possible conversation types:

- DIRECT
- GROUP

### Conversation members table

```text
conversation_members
--------------------
conversation_id
user_id
role
joined_at
left_at
```

This supports:

- Group membership.
- Admin permissions.
- Member removal.
- Membership history.
- Access validation.

### Why separate conversations and members?

A conversation may contain multiple participants.

Separating membership allows the system to manage participants without duplicating conversation metadata.

A unique constraint on `(conversation_id, user_id)` can prevent duplicate membership records for a given membership period, depending on how membership history is modeled.

---

## 8. End-to-End Message Sending Flow

Let's understand what happens when User A sends a message to User B.

### Step 1: Client sends a message

```json
{
  "clientMessageId": "client-M123",
  "conversationId": "C101",
  "content": "Hello!"
}
```

The client-generated ID helps identify retries of the same logical message.

### Step 2: WebSocket Gateway receives it

The gateway:

- Identifies the authenticated user.
- Validates the connection.
- Forwards the message to Message Service.

### Step 3: Message Service validates it

It verifies:

- Sender belongs to the conversation.
- Sender has permission to send.
- Message format is valid.
- Message size is within limits.

### Step 4: Message is persisted

The Message Service writes the message to durable storage.

### Step 5: Message event is published

After successful persistence, a `MessageCreated` event is published.

A transactional outbox can ensure the database write and event record are committed atomically.

### Step 6: Delivery Service processes the event

It checks the recipient's active device connections.

### Step 7: Message is delivered

If User B is online, the system sends the message through the appropriate WebSocket gateway.

If User B is offline, the message remains available in persistent storage and can be retrieved when the user reconnects.

### Step 8: Sender receives acknowledgement

The sender receives confirmation that the message has been accepted and persisted.

```text
User A
  |
  v
WebSocket Gateway
  |
  v
Message Service
  |
  v
Message Database
  |
  v
MessageCreated Event
  |
  v
Delivery Service
  |
  v
Recipient Gateway
  |
  v
User B
```

**Important:** Message persistence, delivery to a gateway, and delivery to a recipient device are separate milestones.

---

## 9. Message Delivery Guarantees

A chat application must distinguish between several states.

### Sent

The server has accepted and durably stored the message.

### Delivered

The recipient device has acknowledged receiving the message.

### Read

The recipient has opened or otherwise marked the message as read according to the application's rules.

Example:

```text
Sender:
       Message Sent       ✓
       Message Delivered  ✓✓
       Message Read       ✓✓ (read indicator)
```

These states are not interchangeable.

A successful WebSocket write does not prove that the recipient received or displayed the message.

### Delivery receipt flow

```text
Sender
  |
  v
Message Service
  |
  v
Database
  |
  v
Delivery Service
  |
  v
Recipient Device
  |
  v
Delivery ACK
  |
  v
Message Service
  |
  v
Receipt Event
  |
  v
Sender Device
```

Delivery receipts should be persisted when the product requires reliable status across devices.

---

## 10. Message Ordering

Message ordering is one of the most important challenges in chat systems.

Suppose User A sends:

```text
Message 1: Hello
Message 2: How are you?
Message 3: Are you there?
```

The recipient should see the messages in the intended conversation order.

However, distributed processing can introduce reordering.

### Why can ordering break?

- Multiple gateway instances.
- Concurrent message requests.
- Multiple producers.
- Broker partitioning.
- Retries.
- Network delays.
- Concurrent database writes.

### Solution: Conversation sequence numbers

Assign a monotonically increasing sequence number within each conversation.

Example:

| Message | Sequence |
|---|---:|
| Hello | 1001 |
| How are you? | 1002 |
| Are you there? | 1003 |

The recipient can sort messages by sequence number.

### How to allocate sequence numbers?

One possible approach is to maintain a conversation record with a sequence counter.

A transaction:

1. Locks or atomically updates the conversation counter.
2. Allocates the next sequence.
3. Inserts the message.
4. Commits.

This establishes a serialization point for message ordering.

### Trade-off

A per-conversation counter can become a contention point for extremely active group conversations.

The system must balance ordering guarantees against write throughput.

### Important principle

Timestamps alone are not a reliable global ordering mechanism.

Two servers may generate timestamps at slightly different times due to clock skew or request concurrency.

---

## 11. Idempotency and Duplicate Messages

A user may tap Send twice or retry after losing network connectivity.

The client may submit the same logical message more than once.

Without idempotency, the recipient could see duplicate messages.

### Client-generated message ID

Each logical send operation gets a unique ID.

```json
{
  "clientMessageId": "device-123-message-456",
  "conversationId": "C101",
  "content": "Hello!"
}
```

The server stores a uniqueness constraint based on the authenticated sender and client message ID.

Conceptually:

```text
(sender_id, client_message_id) UNIQUE
```

If the same request arrives again, the server returns the existing message result rather than creating another message.

### Important consideration

The server must scope the idempotency key correctly.

A client-generated ID alone should not allow another user to retrieve or modify someone else's message.

---

## 12. Online Presence

Presence indicates whether a user is currently available or connected.

Common states:

- Online.
- Offline.
- Away.
- Busy.
- Do Not Disturb.

Presence is different from message delivery status.

A user may be offline but still receive messages through persistent storage and push notifications.

### Presence architecture

```text
User Devices
     |
     v
WebSocket Gateways
     |
     v
Presence Service
     |
     v
Distributed Presence Store
     |
     v
Presence Updates
```

A presence record may contain:

```text
user_id
device_id
status
last_seen_at
expires_at
```

### Heartbeats

A connected client periodically sends a heartbeat.

For example:

```text
Client ---> Heartbeat ---> Gateway
Client ---> Heartbeat ---> Gateway
Client ---> Heartbeat ---> Gateway
```

If heartbeats stop for a configured period, the connection can be considered stale.

### TTL

Presence records can use a time-to-live mechanism.

If the client disappears without a clean disconnect, the record eventually expires.

### Multiple devices

A user may be online on:

- Laptop.
- Phone.
- Tablet.

The user should generally be considered online while at least one valid connection remains active, according to the product's presence rules.

Do not mark a user offline simply because one device disconnects.

### Important limitation

Presence is usually eventually consistent.

A user may briefly appear online after disconnecting or offline while reconnecting.

This is acceptable for many chat applications.

---

## 13. Typing Indicators

Typing indicators show that another participant is currently composing a message.

Unlike actual messages, typing events are temporary and do not normally need durable storage.

Example:

```text
User A starts typing
       |
       v
TypingStarted Event
       |
       v
Conversation Gateway
       |
       v
User B sees "Typing..."
```

When User A stops typing, a `TypingStopped` event may be sent.

### Design considerations

- Throttle typing events.
- Use short-lived state.
- Automatically expire typing status.
- Avoid publishing every keyboard event.
- Scope typing indicators to the conversation.
- Ensure only conversation members receive them.

For example, instead of sending an event for every character, the client can send a typing update at a limited interval.

Typing indicators are best treated as ephemeral real-time signals rather than durable business records.

---

## 14. Offline Message Delivery

What happens when User B is offline?

The message should still be stored.

When User B reconnects:

1. The client authenticates.
2. The gateway establishes a connection.
3. The client requests messages after its last synchronized sequence number.
4. The server retrieves missing messages.
5. The client stores and displays them.
6. The client acknowledges delivery where appropriate.

Example:

```http
GET /api/conversations/C101/messages?afterSequence=1000
```

Response:

```json
{
  "messages": [
    {
      "messageId": "M1001",
      "sequenceNumber": 1001,
      "content": "Hello!"
    },
    {
      "messageId": "M1002",
      "sequenceNumber": 1002,
      "content": "How are you?"
    }
  ]
}
```

### Why use sequence-based synchronization?

It allows clients to retrieve messages they missed while disconnected.

It also helps prevent duplicate messages when combined with client-side message ID deduplication.

### Important consideration

A reconnecting client may have thousands of missed messages.

Use pagination, bounded batches, and incremental synchronization rather than returning an unlimited history.

---

## 15. Group Messaging and Fanout

One-to-one messaging is relatively straightforward.

Group messaging introduces a fanout problem.

Suppose a group has 1,000 members.

When one user sends a message, the system may need to deliver it to 999 other members.

### Fanout-on-Write

The system creates recipient delivery work when the message is sent.

```text
             New Group Message
                    |
                    v
               Group Service
                    |
                    v
                Message DB
                    |
                    v
                Fanout Worker
                    |
       +------------+------------+
       |            |            |
       v            v            v
    User 1       User 2       User 3
       |            |            |
       v            v            v
    Delivery     Delivery     Delivery
```

**Advantages:**

- Fast recipient delivery.
- Easy recipient-specific delivery tracking.
- Efficient reads for many common access patterns.

**Disadvantages:**

- Large write amplification.
- A large group creates many delivery operations.
- Membership changes require careful handling.

### Fanout-on-Read

The message is stored once in the group conversation.

When a member opens the conversation, the system retrieves the relevant messages.

```text
             Group Message
                   |
                   v
              Message DB
                   |
                   v
             Shared History
                   |
          +--------+--------+
          |        |        |
          v        v        v
        User 1   User 2   User 3
```

**Advantages:**

- Fewer writes per message.
- Suitable for very large groups.
- Less recipient-specific write amplification.

**Disadvantages:**

- More work during reads.
- Delivery status requires additional design.
- Offline synchronization can be more complex.

### Hybrid approach

A practical chat platform may use:

- Shared conversation storage for message history.
- Fanout for active connected members.
- Offline synchronization from durable storage.
- Push notifications for selected offline recipients.

This avoids creating unnecessary per-recipient copies of every message.

---

## 16. Message Storage and Database Selection

Chat systems have large write volumes and conversation-oriented read patterns.

The storage system must support:

- High write throughput.
- Efficient conversation history queries.
- Pagination.
- Message ordering.
- Retention.
- Horizontal scaling.

### Relational database

PostgreSQL can work well when:

- Traffic is moderate.
- Strong transactional behavior is useful.
- Relational queries are important.
- Operational simplicity is a priority.

A suitable index can support conversation history retrieval.

### Wide-column database

A database such as Cassandra may be appropriate for very high-volume, partitioned message workloads.

A possible partition key is:

```text
partition_key = conversation_id
```

Messages within a conversation can be clustered by sequence number.

### Partitioning concerns

A very large group can create a hot partition.

Possible solutions include:

- Time-bucketed partitions.
- Conversation sharding.
- Controlled partition sizes.
- Separate storage strategies for unusually large conversations.

### Important principle

Do not select a database solely because a large chat platform uses it.

Choose based on message volume, query patterns, consistency requirements, operational expertise, and growth expectations.

---

## 17. Message Attachments

Chat applications often support:

- Images.
- Videos.
- Documents.
- Audio files.

These should generally not be stored directly inside the message database.

Instead, use object storage.

```text
Client
  |
  v
Upload API
  |
  v
Object Storage
  |
  v
Attachment Metadata
  |
  v
Message Service
  |
  v
Message Database
```

### Recommended flow

1. Client requests an upload URL.
2. Server validates the user and conversation permissions.
3. Server generates a pre-signed upload URL.
4. Client uploads the file directly to object storage.
5. File validation and malware scanning take place.
6. Message references the stored attachment.
7. Recipient retrieves the file through authorized access.

### Important considerations

- File size limits.
- File type validation.
- Malware scanning.
- Access control.
- Expiring download URLs.
- CDN delivery where appropriate.
- Retention and deletion.

We covered object storage and media processing in Lesson 22.

---

## 18. Push Notifications

If a recipient is offline, the platform may send a push notification.

```text
Message Created
      |
      v
Delivery Service
      |
      v
Check User Connections
      |
   +--+--+
   |     |
Online Offline
   |     |
   v     v
WebSocket Push Provider
             |
             v
         Mobile Device
```

The push notification is a signal that a new message is available.

The actual message should remain in the durable message store.

When the user opens the application, the client synchronizes message history.

### Important considerations

- Do not assume push delivery is guaranteed.
- Avoid exposing sensitive message content in notification previews.
- Respect notification preferences.
- Use device-specific push tokens.
- Remove invalid tokens.
- Deduplicate notifications where possible.

Push delivery and chat message delivery are separate concerns.

---

## 19. Read Receipts at Scale

Read receipts can create substantial write volume in large conversations.

Suppose a group has 10,000 members.

If every member's read status is stored separately for every message, the number of records can grow rapidly.

### Possible data models

**Per-message receipts:**

```text
message_receipts
----------------
message_id
user_id
delivered_at
read_at
```

This supports detailed message-level receipts but can create high write volume.

**Per-conversation read cursor:**

```text
conversation_read_state
-----------------------
conversation_id
user_id
last_read_sequence
updated_at
```

This stores the highest sequence number read by a user.

For many chat products, a read cursor is more efficient.

### Trade-off

A read cursor is compact and works well when messages are read in order.

It may not represent every possible case where a user reads messages selectively or out of sequence.

Choose the model based on the receipt behavior the product actually needs.

---

## 20. Security and Privacy

Chat systems contain private user communications.

Important security controls include:

- Authentication.
- Conversation membership authorization.
- Object-level message access checks.
- Encryption in transit.
- Encryption at rest.
- Secure attachment access.
- Abuse prevention.
- Rate limiting.
- Audit logging where appropriate.

### Conversation authorization

Every message retrieval must verify that the authenticated user is a member of the conversation and is allowed to access the requested messages.

Knowing a conversation ID must not grant access.

### End-to-end encryption

In end-to-end encryption, messages are encrypted on the sender's device and decrypted on authorized recipient devices.

The server stores and forwards ciphertext rather than having access to plaintext message content.

This changes the architecture:

- Key management becomes essential.
- Multi-device synchronization becomes more complicated.
- Server-side search becomes difficult.
- Content moderation and recovery capabilities are constrained.
- Group membership changes require key-management operations.

End-to-end encryption is a separate design topic and should be considered from the beginning if it is a core product requirement.

---

## 21. Failure Handling

A production chat platform must handle failures at multiple layers.

| Failure | Handling |
|---|---|
| WebSocket gateway crashes | Client reconnects to another gateway |
| Recipient offline | Store message and synchronize later |
| Message broker unavailable | Persist event in outbox and retry publishing |
| Message DB unavailable | Reject or defer acceptance safely; do not falsely acknowledge persistence |
| Duplicate send request | Use idempotency key |
| Out-of-order events | Use conversation sequence numbers |
| Push provider unavailable | Retry according to policy |
| Presence store unavailable | Degrade presence rather than blocking message persistence |
| Delivery worker crashes | Redeliver safely |
| Network interruption | Reconnect and synchronize missing messages |

### Graceful degradation

If the presence service is unavailable, users may still be able to send and receive messages.

The system can temporarily display unknown or stale presence status.

A failure in an optional presence feature should not necessarily make core messaging unavailable.

---

## 22. Monitoring and Observability

A chat platform needs operational and business metrics.

### Connection metrics

- Active WebSocket connections.
- Connections per gateway.
- Connection establishment failures.
- Reconnection rate.
- Heartbeat timeout rate.
- Average connection lifetime.

### Message metrics

- Messages accepted per second.
- Message persistence latency.
- Message delivery latency.
- Duplicate message rate.
- Message processing failures.
- Out-of-order event count.

### Broker metrics

- Queue depth.
- Consumer lag.
- Oldest event age.
- Retry count.
- Dead-letter queue size.

### Presence metrics

- Online user count.
- Stale presence records.
- Presence update latency.

### User experience metrics

- Time from Send to Sent.
- Time from Sent to Delivered.
- Time from Delivered to Read.
- Failed message percentage.

### Important distinction

Measure the complete message lifecycle.

A low message persistence latency does not prove that users are receiving messages quickly.

Track each stage separately.

---

## 23. Scaling Strategy

Different components have different scaling requirements.

| Component | Scaling approach |
|---|---|
| WebSocket Gateway | Horizontal scaling by connection capacity |
| Message Service | Horizontal scaling by request throughput |
| Delivery Service | Horizontal scaling by delivery workload |
| Message Broker | Partitions and broker capacity |
| Presence Service | Distributed in-memory storage |
| Message Database | Partitioning and sharding |
| Notification Service | Independent worker scaling |
| Attachment Storage | Object storage and CDN |

### Sharding by conversation ID

Messages from the same conversation can be routed to the same logical shard.

This helps with:

- Conversation-local ordering.
- Message history retrieval.
- Data locality.

However, large conversations can create hot shards.

A production system needs monitoring and a strategy for repartitioning or isolating hot conversations.

### Backpressure

If message production exceeds delivery capacity, the system should not allow unbounded memory growth.

Use:

- Durable queues.
- Bounded worker concurrency.
- Rate limits.
- Admission control.
- Priority handling.
- Load shedding for nonessential features.

Core message acceptance and optional features such as typing indicators should have different resource priorities.

---

## 24. Worked Example: Sending a Message in a Large Chat System

Suppose:

- User A sends a message to a group of 500 members.
- 300 members are online.
- 200 members are offline.
- Members may have multiple devices.

### Step 1: Validate

Message Service verifies that User A belongs to the group and can send messages.

### Step 2: Persist

The message is written to durable storage with a conversation sequence number.

### Step 3: Publish event

A `MessageCreated` event is published through the transactional outbox.

### Step 4: Fanout

Delivery Service identifies the active recipient connections.

It routes the message to the appropriate WebSocket gateways.

### Step 5: Offline handling

Offline users do not need to receive the message immediately through WebSockets.

They can retrieve it from conversation history when they reconnect.

Push notifications may be generated according to user preferences and notification policy.

### Step 6: Acknowledgements

The system records delivery receipts from recipient devices where required.

Read receipts update the relevant conversation read state.

### Step 7: Reconnection

An offline user reconnects and requests messages after their last synchronized sequence number.

The system returns the missing messages in order.

### Architecture summary

```text
User A
  |
  v
WebSocket Gateway
  |
  v
Message Service
  |
  +------> Message Database
  |
  +------> Transactional Outbox
                   |
                   v
              Message Broker
                   |
                   v
             Delivery Service
                   |
           +-------+-------+
           |               |
           v               v
      Online Members   Offline Members
           |               |
           v               v
       WebSockets      Push / Sync Later
           |
           v
       Receipts
```

---

## 25. Common Mistakes

1. Using WebSockets as the only message storage mechanism.
2. Assuming a successful socket write means the recipient received the message.
3. Using timestamps as the only ordering mechanism.
4. Ignoring duplicate client submissions.
5. Treating a user as having only one device connection.
6. Storing large attachments inside message records.
7. Creating excessive per-message receipt records for large groups.
8. Treating presence as strongly consistent.
9. Sending push notifications instead of storing messages.
10. Using one delivery queue for every type of work.
11. Ignoring hot partitions caused by large groups.
12. Failing to synchronize missed messages after reconnection.
13. Acknowledging messages before durable persistence.
14. Ignoring conversation membership checks.
15. Failing to distinguish message acceptance, delivery, and read status.

---

## 26. Practical Architecture Checklist

### Messaging
- [ ] Messages are durably stored.
- [ ] Client-generated idempotency IDs are supported.
- [ ] Conversation ordering is defined.
- [ ] Message history supports pagination.
- [ ] Conversation membership is validated.

### Real-time communication
- [ ] WebSocket connections are horizontally scalable.
- [ ] Heartbeats detect stale connections.
- [ ] Multiple devices are supported.
- [ ] Gateway-to-gateway delivery routing exists.
- [ ] Reconnection and message synchronization are implemented.

### Delivery
- [ ] Sent, delivered, and read are distinct states.
- [ ] Delivery events are durable where required.
- [ ] Offline messages can be retrieved.
- [ ] Push notifications are treated separately from message storage.

### Groups
- [ ] Group membership is modeled separately.
- [ ] Fanout strategy is defined.
- [ ] Large-group workloads are considered.
- [ ] Read receipt storage is scalable.

### Reliability and security
- [ ] Outbox pattern prevents lost event publication.
- [ ] Consumers are idempotent.
- [ ] Retries and dead-letter handling are defined.
- [ ] Conversation authorization is enforced.
- [ ] Sensitive data is protected.

### Observability
- [ ] Connection health is monitored.
- [ ] Message delivery latency is tracked.
- [ ] Broker lag is monitored.
- [ ] Failed deliveries are observable.
- [ ] Reconnection behavior is measured.

---

## 27. Revision Questions

1. Why are WebSockets useful for chat applications?
2. Why can't WebSockets replace persistent message storage?
3. How does a WebSocket gateway scale horizontally?
4. What is a connection registry?
5. Why can a user have multiple active connections?
6. What is the difference between sent, delivered, and read?
7. How can a chat system preserve message ordering?
8. Why are timestamps insufficient for strict conversation ordering?
9. How does client-generated idempotency prevent duplicate messages?
10. How should offline users receive missed messages?
11. What is the difference between fanout-on-write and fanout-on-read?
12. Why can large group conversations create hot partitions?
13. How does presence tracking work?
14. Why is presence usually eventually consistent?
15. Why should typing indicators be ephemeral?
16. How can a system efficiently store read receipts?
17. Why are message attachments stored separately?
18. How does push notification delivery differ from message delivery?
19. What is the role of a transactional outbox in chat?
20. How would you handle a WebSocket gateway failure without losing messages?

---

## 28. Practical Exercise

Design a chat application with the following requirements:

### Expected scale

- 5 million daily active users.
- 500,000 concurrent WebSocket connections at peak.
- Average 30 messages per user per day.
- Support one-to-one and group conversations.
- Groups can contain up to 5,000 members.
- Users can connect from multiple devices.
- Messages must remain available when recipients are offline.

### Your design should explain:

1. What services would you create?
2. How would you manage 500,000 concurrent WebSocket connections?
3. How would you route messages between gateway instances?
4. Which database would you choose for message storage, and why?
5. How would you preserve message ordering?
6. How would you prevent duplicate messages?
7. How would you handle offline message synchronization?
8. Would you use fanout-on-write or fanout-on-read for groups of 5,000 members?
9. How would you model delivery and read receipts?
10. How would you track online presence across multiple devices?
11. How would you handle gateway crashes?
12. How would you store images and videos?
13. Which events would you publish through the message broker?
14. How would you protect private conversations?
15. Which metrics would you monitor to identify delivery delays?

### Practice focus

Start by designing the message-sending flow from the sender's device to the recipient's device.

Then handle three scenarios separately:

1. Recipient is online on one device.
2. Recipient is offline.
3. Recipient is online on three devices.

Pay particular attention to the difference between **message persistence, message delivery, and read acknowledgement**. These are three separate stages and should not be treated as the same operation.

### Key takeaway

A scalable chat system is not just a WebSocket server. It combines persistent message storage, connection management, message ordering, distributed delivery, offline synchronization, presence tracking, and reliable acknowledgements. Each part has different consistency and availability requirements.