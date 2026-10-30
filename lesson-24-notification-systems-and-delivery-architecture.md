# Lesson 24 — Notification Systems: Push Notifications, Email, SMS, Delivery Guarantees, Preferences, and Fanout

Notifications are an important part of almost every distributed application. A ride-booking platform needs to notify passengers when a booking is confirmed, drivers when a trip is assigned, and dispatchers when an important event occurs.

The challenge is not simply sending a notification. We need to deliver the right message, through the right channel, at the right time—even when external providers fail or users are offline.

## 1. Why Do We Need a Notification System?

Imagine a passenger creates a booking.

The system needs to:
- Confirm the booking to the passenger.
- Notify the assigned driver.
- Notify the dispatcher.
- Send an email confirmation.
- Send an SMS if required.
- Update the passenger's application in real time.

A naive implementation might send all notifications directly from the Booking Service.

```text
Booking Service
   |
   +----> Email Provider
   |
   +----> SMS Provider
   |
   +----> Push Provider
   |
   +----> WebSocket Gateway
```

This creates several problems:

1. Booking creation depends on external providers.
2. A slow email provider can delay the booking API.
3. Provider failures may cause booking failures.
4. Every service needs notification-related logic.
5. Retry and delivery tracking become difficult.
6. Adding another notification channel requires changes across services.

A better architecture separates notification delivery from business operations.

---

## 2. Notification System Architecture

A dedicated Notification Service handles delivery through different channels.

```text
                 Booking Service
                       |
                       v
                 Message Broker
                       |
                       v
                Notification Service
                       |
             +---------+---------+
             |         |         |
             v         v         v
          Email      SMS       Push
          Worker    Worker     Worker
             |         |         |
             v         v         v
          Provider  Provider  Push Provider
```

The Booking Service publishes a business event.

The Notification Service consumes it and determines:
- Who should receive the notification?
- Which channel should be used?
- Which template should be rendered?
- Whether user preferences allow delivery?
- Whether delivery should happen immediately or later?

This creates loose coupling between business services and notification providers.

### Key Principle

**Business services publish events; notification infrastructure handles delivery.**

---

## 3. Notification vs Event

These concepts are related but different.

### Event

An event represents something that happened in the system.

Examples:

```text
BOOKING_CREATED
BOOKING_CONFIRMED
DRIVER_ASSIGNED
TRIP_STARTED
TRIP_COMPLETED
PAYMENT_FAILED
```

### Notification

A notification communicates information to a recipient.

For example:

Event:

```json
{
  "eventType": "DRIVER_ASSIGNED",
  "bookingId": "B123",
  "driverId": "D456"
}
```

Passenger notification:

```text
Your driver has been assigned.
```

Driver notification:

```text
You have received a new trip assignment.
```

The same event can generate different notifications for different recipients.

An event does not automatically mean that every recipient should receive a message.

---

## 4. Types of Notification Channels

Applications commonly support multiple notification channels.

### 4.1 Push Notifications

Push notifications are messages delivered through platform push services.

Examples:
- Firebase Cloud Messaging (FCM).
- Apple Push Notification service (APNs).

Typical use cases:
- Booking confirmation.
- Driver assignment.
- Trip status updates.
- Important account alerts.

Simplified architecture:

```text
Backend
   |
   v
Push Notification Service
   |
   +------> FCM ------> Android Device
   |
   +------> APNs -----> Apple Device
```

The backend generally sends messages to the platform's push infrastructure, which handles delivery to registered devices.

Important concepts:
- Device registration token.
- Token refresh.
- Notification payload.
- Data payload.
- Platform delivery behavior.
- Token invalidation.

Push delivery is not guaranteed to mean that the user saw or interacted with the notification.

Devices can be offline, notifications can be disabled, and operating systems can delay or suppress delivery.

### 4.2 Email Notifications

Email is useful for:
- Booking confirmations.
- Invoices.
- Account verification.
- Password resets.
- Important transactional updates.

Typical providers include:
- Amazon SES.
- SendGrid.
- Mailgun.
- Postmark.

Architecture:

```text
Notification Service
       |
       v
Email Worker
       |
       v
Email Provider
       |
       v
Recipient Mail Server
       |
       v
User Inbox
```

Important email delivery concepts:

| Concept | Meaning |
|---|---|
| Accepted | Provider accepted the request |
| Delivered | Provider reports delivery to the recipient's mail server |
| Bounced | Email could not be delivered |
| Deferred | Delivery was temporarily delayed |
| Opened | Tracking system detected an open event, where supported |
| Clicked | Tracking system detected a link click |

An accepted email is not necessarily an email that reached the user's inbox.

### 4.3 SMS Notifications

SMS is useful for:
- OTP verification.
- Critical booking updates.
- Driver arrival alerts.
- Time-sensitive communication.

Providers include:
- Twilio.
- AWS End User Messaging.
- Regional telecom messaging providers.

SMS delivery depends on:
- Telecom networks.
- Recipient phone availability.
- Country-specific regulations.
- Sender registration requirements.
- Provider delivery status.

SMS is often more expensive than other channels, so use it selectively.

### 4.4 In-App Notifications

In-app notifications are displayed inside the application.

Examples:
- Notification center.
- Unread notification badge.
- Booking status update.
- System announcements.

Unlike push notifications, in-app notifications can be stored and retrieved when a user opens the application.

A typical architecture:

```text
Notification Service
       |
       +------> PostgreSQL
       |          |
       |          v
       |      Notification History
       |
       +------> WebSocket Gateway
                  |
                  v
               Connected Client
```

Persist important in-app notifications so users can access them later.

---

## 5. Notification Delivery Architecture

Let's design a more complete system.

```text
                  Business Services
                         |
                         v
                 Transactional Outbox
                         |
                         v
                    Kafka
                         |
                         v
                Notification Service
                         |
                  Recipient Resolver
                         |
                  Preference Checker
                         |
                   Template Engine
                         |
                  Channel Dispatcher
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      Email Queue     SMS Queue      Push Queue
          |              |              |
          v              v              v
      Email Worker    SMS Worker     Push Worker
          |              |              |
          v              v              v
      Email API       SMS API        Push API
```

### Responsibilities

**Business Services**
- Produce domain events.
- Do not implement provider-specific notification logic.

**Message Broker**
- Decouples event production from notification processing.
- Buffers traffic during temporary failures.

**Recipient Resolver**
- Determines who should receive a notification.

**Preference Checker**
- Checks notification preferences and applicable mandatory-message policies.

**Template Engine**
- Selects and renders the correct message.

**Channel Dispatcher**
- Routes notifications to the appropriate delivery queues.

**Workers**
- Handle provider communication, retries, and delivery updates.

**Notification Database**
- Stores notification records, delivery attempts, and statuses.

---

## 6. Notification Data Modeling

A notification system needs to track both the logical notification and its delivery attempts.

A simplified model:

### Notifications

| Field | Purpose |
|---|---|
| id | Notification ID |
| recipient_id | User receiving the notification |
| event_id | Source business event |
| notification_type | Type of notification |
| title | Notification title |
| body | Notification content |
| status | Logical notification status |
| created_at | Creation timestamp |

### Notification Deliveries

| Field | Purpose |
|---|---|
| id | Delivery ID |
| notification_id | Parent notification |
| channel | EMAIL, SMS, PUSH, IN_APP |
| provider | External provider |
| status | Delivery status |
| attempt_count | Number of attempts |
| provider_message_id | Provider reference |
| last_error | Latest error |
| updated_at | Last update |

### Why Separate Notifications and Deliveries?

Suppose a passenger receives a booking notification through:
- Push notification.
- Email.
- In-app notification.

These are three delivery operations for one logical notification.

Separating them allows us to track each channel independently.

Example:

```text
Notification N123
       |
       +------> Push Delivery: DELIVERED
       |
       +------> Email Delivery: FAILED
       |
       +------> In-App Delivery: READ
```

A failure in one channel does not necessarily mean the whole notification failed.

---

## 7. Delivery Guarantees

Notification delivery is a distributed-systems problem.

A message can be:
- Lost.
- Delivered more than once.
- Delayed.
- Accepted by a provider but not delivered to the user.

### At-Most-Once Delivery

The system attempts delivery once and does not retry.

Advantages:
- No duplicate attempts caused by retries.

Disadvantages:
- A transient failure can result in message loss.

### At-Least-Once Delivery

The system retries until it receives an acceptable acknowledgement or reaches its retry policy.

Advantages:
- Better resilience to temporary failures.

Disadvantages:
- Duplicate delivery is possible.

### Exactly-Once Delivery

True end-to-end exactly-once delivery across our database, broker, external provider, telecom network, and recipient device is generally not something we can guarantee.

Instead, design for:
- Durable event processing.
- Idempotent consumers.
- Provider idempotency where supported.
- Duplicate detection.
- Clear delivery-state tracking.

**Important:** An acknowledged message from a broker does not necessarily mean the end user received the notification.

---

## 8. Idempotency and Duplicate Notifications

Imagine Kafka delivers the same `BOOKING_CONFIRMED` event twice.

Without deduplication:

```text
Passenger receives:
1. Your booking is confirmed.
2. Your booking is confirmed.
```

We need an idempotency strategy.

### Use Event IDs

Every business event should have a stable unique ID.

Example:

```json
{
  "eventId": "EVT-123",
  "eventType": "BOOKING_CONFIRMED",
  "bookingId": "B123",
  "recipientId": "P456"
}
```

Create a unique constraint or deduplication record based on the logical notification identity.

For example:

```text
UNIQUE(event_id, recipient_id, notification_type)
```

The exact key depends on whether multiple legitimate notifications of the same type can originate from one event.

### Idempotent Processing Flow

```text
Receive Event
     |
     v
Check Existing Notification
     |
     +----> Already Processed
     |          |
     |          v
     |       Skip Duplicate
     |
     v
Create Notification
     |
     v
Create Delivery Records
     |
     v
Commit Transaction
```

Provider calls introduce another failure boundary.

For example:
1. Worker sends an SMS.
2. Provider accepts it.
3. Worker crashes before recording success.
4. Worker retries.

This may send a duplicate SMS.

Mitigations include:
- Provider-supported idempotency keys.
- Stable delivery identifiers.
- Reconciliation using provider message IDs.
- Explicitly accepting residual duplicate risk when provider guarantees are limited.

A database deduplication record alone cannot guarantee exactly-once external side effects.

---

## 9. Retry Strategy and Dead-Letter Queues

External providers can fail for temporary reasons.

Examples:
- Network timeout.
- Provider outage.
- Rate limiting.
- Temporary server error.

A retry strategy should distinguish temporary failures from permanent failures.

### Retryable Errors

Examples:
- HTTP 429.
- HTTP 500.
- HTTP 502.
- HTTP 503.
- Temporary network errors.

### Usually Non-Retryable Errors

Examples:
- Invalid phone number.
- Invalid email address.
- Invalid authentication credentials.
- Invalid request payload.

The exact classification depends on provider behavior and error codes.

### Exponential Backoff

Instead of retrying immediately:

```text
Attempt 1: Immediately
Attempt 2: 5 seconds
Attempt 3: 15 seconds
Attempt 4: 45 seconds
Attempt 5: 2 minutes
```

These are illustrative intervals.

Add jitter to avoid many workers retrying simultaneously.

### Dead-Letter Queue

If a notification repeatedly fails, move it to a dead-letter queue for investigation or controlled reprocessing.

```text
Notification Queue
       |
       v
Notification Worker
       |
       +----> Success
       |
       +----> Retryable Failure
       |          |
       |          v
       |        Retry
       |
       +----> Max Attempts Exceeded
                  |
                  v
             Dead-Letter Queue
```

Do not retry invalid messages indefinitely.

---

## 10. Notification Preferences

Users should be able to control which notifications they receive.

Example:

| Notification Type | Push | Email | SMS |
|---|---|---|---|
| Booking Confirmation | Yes | Yes | No |
| Driver Assigned | Yes | No | Yes |
| Promotional Offers | Yes | Yes | No |
| Payment Failure | Yes | Yes | Optional |

Preferences should be modeled separately from notification delivery.

Example:

```json
{
  "userId": "P123",
  "preferences": {
    "bookingUpdates": {
      "push": true,
      "email": true,
      "sms": false
    },
    "promotionalOffers": {
      "push": false,
      "email": false,
      "sms": false
    }
  }
}
```

### Important Distinction

Not all notifications should be controlled in the same way.

For example:
- Marketing messages generally require appropriate consent and opt-out handling.
- Security alerts may have different rules.
- Essential transactional notifications may be treated differently depending on legal requirements and product policy.

Do not silently override user preferences without a defined policy.

---

## 11. Notification Priority and Scheduling

Not every notification needs immediate delivery.

Consider these examples:

| Notification | Priority |
|---|---|
| OTP | Critical |
| Driver assigned | High |
| Booking confirmation | High |
| Invoice generated | Normal |
| Promotional campaign | Low |

### Priority Queues

Separate high-priority and low-priority workloads.

```text
                 Notification Service
                         |
             +-----------+-----------+
             |                       |
             v                       v
       High Priority             Low Priority
             |                       |
             v                       v
       Fast Workers              Batch Workers
             |                       |
             v                       v
         Immediate               Scheduled
```

This prevents large marketing campaigns from delaying OTP or booking notifications.

### Scheduled Notifications

Some notifications should be sent at a particular time.

Examples:
- Trip reminders.
- Appointment reminders.
- Scheduled reports.

A scheduling system needs:
- Scheduled execution time.
- Time zone handling.
- Cancellation support.
- Retry handling.
- Duplicate prevention.

For large-scale scheduling, use a durable scheduling mechanism rather than relying only on in-memory timers.

---

## 12. Fanout: One Event to Many Recipients

Fanout means distributing one event to multiple recipients.

Suppose an administrator sends an announcement to 1 million users.

A naive approach:

```text
Admin
  |
  v
Notification Service
  |
  +----> User 1
  +----> User 2
  +----> User 3
  +----> ...
  +----> User 1,000,000
```

Sending all notifications synchronously is not scalable.

### Asynchronous Fanout

```text
Admin
  |
  v
Campaign Service
  |
  v
Campaign Event
  |
  v
Message Broker
  |
  +----> Worker 1
  |
  +----> Worker 2
  |
  +----> Worker 3
  |
  v
Recipient Delivery
```

Workers process recipient batches independently.

### Fanout-on-Write

When an event occurs, the system creates individual recipient notifications.

Advantages:
- Easy to retrieve recipient-specific notifications.
- Delivery work happens in the background.

Disadvantages:
- Large fanout can create substantial write load.
- Storage usage grows with recipient count.

### Fanout-on-Read

The system stores an event once and determines relevant notifications when a user opens the application.

Advantages:
- Less write amplification.
- Efficient for some large audiences.

Disadvantages:
- Reads become more complex.
- Notification retrieval may be more expensive.
- Historical recipient state can be harder to preserve.

### Choosing a Strategy

| Situation | Possible Approach |
|---|---|
| One booking to a few recipients | Fanout-on-write |
| One campaign to millions of users | Batched asynchronous fanout |
| Large shared activity feed | Consider fanout-on-read or hybrid |
| Time-critical individual alert | Dedicated high-priority delivery |

---

## 13. Provider Rate Limits and Backpressure

External providers often enforce rate limits.

Suppose an SMS provider permits 100 requests per second.

Our system receives 500 SMS requests per second.

If we send all requests immediately:
- The provider may reject excess requests.
- Retry volume may increase.
- Queue depth may grow.
- Delivery latency may increase.

### Solution

Control the sending rate.

```text
SMS Queue
    |
    v
Rate Limiter
    |
    v
SMS Workers
    |
    v
SMS Provider
```

The worker system should respect provider quotas.

Monitor:
- Provider request rate.
- HTTP 429 responses.
- Queue depth.
- Oldest message age.
- Delivery latency.
- Retry rate.

If incoming volume exceeds provider capacity for an extended period, the backlog grows.

A queue absorbs temporary spikes, but it does not create unlimited processing capacity.

---

## 14. Delivery Status and Tracking

A notification system needs to track its lifecycle.

Example states:

```text
CREATED
   |
   v
QUEUED
   |
   v
PROCESSING
   |
   +------> FAILED
   |
   v
PROVIDER_ACCEPTED
   |
   v
DELIVERED
   |
   v
READ
```

Not every channel supports every state.

For example:
- Email providers may report delivery to a mail server.
- SMS providers may report carrier delivery status.
- Push providers may report acceptance without proving that the user saw the notification.
- In-app notifications can track read status when the application reports it.

Keep provider acceptance, delivery, and user interaction as separate concepts.

### Provider Webhooks

Providers may send delivery callbacks to our backend.

Example:

```http
POST /webhooks/sms/delivery
```

The webhook might contain:

```json
{
  "messageId": "MSG123",
  "status": "DELIVERED",
  "timestamp": "2026-10-30T10:15:00Z"
}
```

Webhook handling should include:
- Signature verification.
- Idempotent event processing.
- Timestamp and replay checks where applicable.
- Safe status transitions.
- Audit logging.

Do not trust an arbitrary request claiming that a notification was delivered.

---

## 15. Notification Templates and Localization

Notification content should not be hardcoded inside every service.

Use templates.

Example:

```text
Template: DRIVER_ASSIGNED

Hello {{passengerName}},

Your driver {{driverName}} has been assigned.
Your trip is scheduled for {{pickupTime}}.
```

### Why Templates?

- Centralized message management.
- Consistent branding.
- Easier content updates.
- Localization.
- Different channel formats.

### Localization

A passenger may prefer Hindi while another prefers English.

The notification service can select the appropriate template using user preferences.

Example:

```text
Language: en
Template: DRIVER_ASSIGNED_EN
```

```text
Language: hi
Template: DRIVER_ASSIGNED_HI
```

Store structured event data rather than a fully rendered message wherever possible.

This allows the notification service to render the correct language and channel-specific content.

Be careful not to expose sensitive event payloads to template authors or logs.

---

## 16. Notification System Reliability

A production notification system should handle failures at every stage.

### Failure Scenarios

**Scenario 1: Email provider is unavailable**

- Keep email jobs in a durable queue.
- Retry with backoff.
- Avoid blocking booking creation.
- Alert if delivery delay exceeds the expected limit.

**Scenario 2: Notification worker crashes**

- Broker redelivers unacknowledged messages according to its delivery semantics.
- Consumer handles duplicate delivery.
- Another worker continues processing.

**Scenario 3: Provider accepts a request but response times out**

- Do not assume the request failed.
- Check provider capabilities for idempotency or status lookup.
- Avoid blindly resending if it may create a duplicate.

**Scenario 4: Large campaign creates a traffic spike**

- Process recipients in batches.
- Apply provider rate limits.
- Use queue backpressure.
- Protect high-priority transactional notifications.

**Scenario 5: Invalid recipient**

- Mark the delivery appropriately.
- Do not retry permanently invalid destinations.
- Update recipient validity where justified.

---

## 17. Notification Observability

A notification system requires its own operational dashboards.

### Throughput Metrics

- Notifications created per second.
- Notifications processed per second.
- Deliveries per channel.
- Provider requests per second.

### Reliability Metrics

- Delivery success rate.
- Failure rate.
- Retry rate.
- Dead-letter queue count.
- Duplicate suppression count.

### Latency Metrics

- Event-to-queue latency.
- Queue waiting time.
- Processing duration.
- Provider response time.
- End-to-end delivery delay where measurable.

### Provider Metrics

- HTTP 429 rate.
- HTTP 5xx rate.
- Timeout rate.
- Provider availability.
- Provider quota consumption.

### Business Metrics

- Booking confirmation notification success.
- OTP delivery latency.
- Driver assignment notification delay.
- User notification engagement.
- Marketing opt-out rate.

Useful alerts:

1. OTP delivery failures exceed the defined threshold.
2. Booking notification queue age exceeds the service objective.
3. SMS provider returns sustained 429 errors.
4. Dead-letter queue starts growing unexpectedly.
5. Push delivery failures increase significantly.

Alert thresholds should be based on actual service requirements.

---

## 18. Worked Example — Driver Assignment Notification

A passenger creates a booking and a driver is assigned.

We need to:
- Notify the passenger through push and in-app notification.
- Notify the driver through push.
- Send an email confirmation.

### Step 1 — Business Event

Booking Service writes:

```json
{
  "eventId": "EVT-123",
  "eventType": "DRIVER_ASSIGNED",
  "bookingId": "B123",
  "passengerId": "P456",
  "driverId": "D789"
}
```

The booking update and outbox event are committed in one database transaction.

### Step 2 — Event Publication

Outbox publisher sends the event to Kafka.

### Step 3 — Notification Processing

Notification Service consumes the event.

It resolves:
- Passenger device tokens.
- Driver device tokens.
- Passenger email address.
- Notification preferences.

### Step 4 — Delivery Creation

It creates separate delivery records:

```text
Notification N123
   |
   +----> Passenger Push
   |
   +----> Passenger In-App
   |
   +----> Driver Push
   |
   +----> Passenger Email
```

### Step 5 — Channel Delivery

Workers send each delivery independently.

### Step 6 — Status Tracking

Provider responses and webhooks update delivery records.

### Step 7 — Failure Handling

If email fails:
- Retry if the error is temporary.
- Do not block push or in-app delivery.
- Move to the DLQ after the configured retry policy.

### Architecture

```text
Booking Service
      |
      v
PostgreSQL + Outbox
      |
      v
Kafka
      |
      v
Notification Service
      |
      v
Recipient + Preference Resolution
      |
      v
Notification Database
      |
      +------> Push Queue
      |            |
      |            v
      |        Push Provider
      |
      +------> Email Queue
      |            |
      |            v
      |        Email Provider
      |
      +------> In-App Storage
      |
      +------> WebSocket Gateway
```

The booking transaction completes independently of external notification delivery.

---

## 19. Common Notification Architecture Mistakes

1. Sending notifications synchronously inside business API requests.
2. Coupling every business service to email and SMS providers.
3. Assuming provider acceptance means user receipt.
4. Retrying every failure without classification.
5. Ignoring duplicate notifications.
6. Not separating notification records from channel delivery records.
7. Sending marketing traffic through the same unrestricted priority path as OTPs.
8. Ignoring provider rate limits.
9. Not tracking delivery status.
10. Treating email, SMS, push, and in-app delivery as identical.
11. Trusting provider webhooks without verification.
12. Not maintaining notification preferences.
13. Treating queues as unlimited capacity.
14. Assuming exactly-once external delivery is guaranteed.

---

## 20. Revision Questions

1. Why should notification delivery be separated from business services?
2. What is the difference between a business event and a notification?
3. Explain push, email, SMS, and in-app notifications.
4. What is the difference between provider acceptance and actual delivery?
5. Explain at-most-once and at-least-once delivery.
6. Why is exactly-once delivery difficult across external providers?
7. How do idempotency keys help prevent duplicate notifications?
8. Why should notifications and delivery attempts be stored separately?
9. How does a dead-letter queue help?
10. Why should retryable and permanent failures be treated differently?
11. Explain fanout-on-write and fanout-on-read.
12. How would you handle a campaign targeting one million users?
13. Why do notification systems need rate limiting?
14. How would you handle provider downtime?
15. How do user preferences affect notification delivery?
16. Why should provider webhooks be verified?
17. How would you prioritize OTP messages over marketing messages?
18. Which metrics would you monitor for a notification platform?

---

## 21. Practical Exercise — Design a Notification Platform

Design a centralized notification system for our ride-booking platform.

Requirements:

- Booking confirmation.
- Driver assignment.
- Driver arrival.
- Trip completion.
- Payment failure.
- OTP verification.
- Email invoices.
- Promotional campaigns.
- In-app notification history.

Your design should answer:

1. Which events should business services publish?
2. How will events reach the Notification Service?
3. How will recipients be resolved?
4. How will notification preferences be applied?
5. How will you support push, email, SMS, and in-app channels?
6. How will you prevent duplicate notifications?
7. How will you handle provider timeouts?
8. How will you implement retries and DLQs?
9. How will you handle one million recipients?
10. How will you prevent marketing traffic from delaying OTPs?
11. How will you track delivery status?
12. What metrics and alerts will you configure?

### Final Takeaway

A scalable notification architecture separates business events from delivery infrastructure.

- **Business Services:** Publish domain events.
- **Message Broker:** Buffers and distributes events.
- **Notification Service:** Resolves recipients and preferences.
- **Workers:** Deliver messages through individual channels.
- **Providers:** Handle external delivery infrastructure.
- **Notification Database:** Tracks notification and delivery history.
- **Observability:** Tracks delivery success, latency, retries, and failures.

The goal is to make notifications reliable without making core business operations dependent on external messaging providers.

### Practice focus

Design the driver-assignment notification flow. Pay particular attention to duplicate prevention, provider timeouts, independent channel delivery, and priority queues.