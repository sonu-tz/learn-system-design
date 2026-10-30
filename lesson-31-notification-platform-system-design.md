# Lesson 31 — System Design of a Notification Platform

In previous lessons, we covered notification systems as individual components, event-driven architecture, message brokers, and microservices. Now we'll bring these concepts together and design a complete notification platform.

We'll explore how systems deliver millions of notifications reliably across push notifications, SMS, email, and in-app channels while handling retries, user preferences, provider failures, duplicate messages, and delivery tracking.

This is an important real-world system design problem because notifications are asynchronous, externally dependent, and often time-sensitive.

## 1. Problem Statement

Imagine a ride-booking platform.

Whenever an important business event happens, users may need to receive notifications.

Examples:

| Event | Recipient | Notification |
|---|---|---|
| Booking created | Passenger | Booking confirmation |
| Driver assigned | Passenger | Driver details |
| Booking cancelled | Passenger and driver | Cancellation update |
| Trip completed | Passenger | Trip completion |
| Payment failed | Passenger | Payment failure |
| New trip assigned | Driver | Assignment alert |
| Password reset requested | User | Password reset email |

The platform may need to deliver notifications through:

- Push notifications.
- SMS.
- Email.
- In-app notifications.
- WebSocket or Server-Sent Events for real-time updates.

### Main challenge

How do we design a notification platform that:

1. Handles high notification volume.
2. Supports multiple delivery channels.
3. Avoids duplicate notifications.
4. Handles provider failures.
5. Supports retries.
6. Respects user preferences.
7. Tracks delivery status.
8. Scales independently of business services.

---

## 2. Functional Requirements

The system should support:

- Accept notification requests from multiple services.
- Process business events asynchronously.
- Support push, SMS, email, and in-app delivery.
- Support templates and localization.
- Manage user notification preferences.
- Support scheduled notifications.
- Support notification priorities.
- Retry temporary failures.
- Track notification lifecycle.
- Support multiple providers per channel.
- Prevent duplicate notification creation where possible.
- Support cancellation of scheduled notifications.

## 3. Non-Functional Requirements

Important system qualities include:

| Requirement | Explanation |
|---|---|
| Scalability | Handle increasing notification volume |
| Availability | Accept and process notifications reliably |
| Durability | Avoid losing accepted notification requests |
| Low latency | Deliver urgent notifications quickly |
| Reliability | Handle temporary provider failures |
| Security | Protect personal information |
| Observability | Track failures, delays, and delivery rates |
| Extensibility | Add channels and providers without major redesign |

Not every notification needs the same delivery guarantee.

For example:

- A password reset notification is time-sensitive.
- A promotional email may be delayed.
- A booking cancellation notification is important.
- An analytics notification may be low priority.

The architecture should support different delivery policies.

---

## 4. High-Level Architecture

A notification platform should be independent of the business services that generate notification requests.

```text
                  Business Services
                         |
              +----------+----------+
              |          |          |
              v          v          v
           Booking    Payment     Driver
           Service    Service     Service
              |          |          |
              +----------+----------+
                         |
                         v
                  Message Broker
                         |
                         v
                 Notification Service
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      Preference      Template       Scheduler
       Service         Engine         Service
          |              |              |
          +--------------+--------------+
                         |
                         v
                  Channel Router
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       Push Queue     SMS Queue      Email Queue
          |              |              |
          v              v              v
      Push Worker    SMS Worker     Email Worker
          |              |              |
          v              v              v
      Push Provider  SMS Provider   Email Provider
```

This architecture separates:

- Notification creation.
- Business-event consumption.
- Preference evaluation.
- Template rendering.
- Channel routing.
- Provider delivery.
- Delivery tracking.

Each component can evolve independently.

---

## 5. Notification Request vs Business Event

A business event and a notification request are not the same thing.

### Business event

Represents something that happened.

```json
{
  "eventId": "evt-1001",
  "eventType": "DriverAssigned",
  "bookingId": "B123",
  "driverId": "D456",
  "passengerId": "P789"
}
```

### Notification request

Represents a request to communicate something to a recipient.

```json
{
  "notificationId": "ntf-1001",
  "recipientId": "P789",
  "templateId": "driver-assigned",
  "channels": ["PUSH", "SMS"],
  "priority": "HIGH",
  "data": {
    "bookingId": "B123",
    "driverName": "Driver A"
  }
}
```

One business event may generate multiple notification requests.

For example, DriverAssigned may produce:

- A passenger push notification.
- A passenger SMS.
- An in-app notification.
- A driver confirmation.

A notification platform should decide which communication is appropriate rather than making every business service implement provider integrations independently.

---

## 6. Notification Data Model

A production notification system typically needs several related entities.

### A. Notification

Represents the logical notification request.

```text
notifications
-------------
id
recipient_id
event_id
template_id
priority
status
created_at
scheduled_at
```

### B. Notification Delivery

Represents an attempt to deliver through a particular channel.

```text
notification_deliveries
-----------------------
id
notification_id
channel
provider
status
provider_message_id
attempt_count
next_retry_at
sent_at
delivered_at
failure_reason
```

### C. Notification Preferences

Stores user preferences.

```text
notification_preferences
-------------------------
user_id
channel
category
enabled
updated_at
```

### D. Templates

Stores message templates.

```text
notification_templates
----------------------
id
event_type
channel
locale
version
subject
body
```

### Why separate notifications from deliveries?

A single notification may have multiple delivery attempts or channels.

For example:

```text
Notification: N123

    |
    +---- Push Delivery
    |       |
    |       +---- Attempt 1: Failed
    |       |
    |       +---- Attempt 2: Delivered
    |
    +---- SMS Delivery
            |
            +---- Attempt 1: Delivered
```

This separation gives us a clear notification lifecycle and channel-specific delivery history.

---

## 7. Notification Lifecycle

A notification can move through several states.

```text
CREATED
   |
   v
PENDING
   |
   v
PROCESSING
   |
   +------> SENT
   |          |
   |          v
   |       DELIVERED
   |
   +------> RETRYING
   |          |
   |          v
   |       PROCESSING
   |
   +------> FAILED
   |
   +------> CANCELLED
```

The exact state machine depends on the channel and provider.

Important distinction:

- **Created:** The logical notification exists.
- **Sent:** The provider accepted the message or request.
- **Delivered:** Delivery was confirmed through reliable provider evidence.
- **Failed:** The system determined that delivery cannot proceed successfully.
- **Cancelled:** The notification was intentionally stopped.

A successful provider API response does not necessarily mean that the recipient received the message.

For example, an SMS provider may accept a message and later report that delivery failed.

---

## 8. Channel Selection and Routing

Not every notification should use every channel.

Example policies:

| Notification | Primary channel | Fallback |
|---|---|---|
| Password reset | Email or SMS, depending on account configuration | Another verified recovery channel |
| Driver assigned | Push | SMS |
| Payment failed | Push or email | SMS, if appropriate |
| Promotional offer | Push or email | None |
| Booking cancelled | Push | SMS |

The notification platform should support configurable routing policies.

### Example routing

```text
DriverAssigned Event
        |
        v
Notification Policy
        |
        v
Check User Preferences
        |
        v
Select Channels
        |
    +---+---+
    |       |
    v       v
   Push    SMS
    |       |
    v       v
  Worker  Worker
```

Channel selection may depend on:

- Notification category.
- User preferences.
- User's verified contact information.
- Business urgency.
- Channel availability.
- Cost.
- Locale.
- Tenant configuration.

### Important distinction

A fallback channel should be triggered by a defined failure condition.

For example, if push delivery is not confirmed within a certain period, SMS may be used for a high-priority notification.

Do not automatically send SMS every time a push provider accepts a request but does not immediately confirm delivery. Push delivery can be delayed for reasons outside the platform's control.

---

## 9. Notification Preferences

Users should have control over which notifications they receive.

Possible preference categories:

- Booking updates.
- Payment updates.
- Marketing messages.
- Product announcements.
- Driver updates.

Example:

```json
{
  "userId": "P123",
  "preferences": {
    "booking_updates": {
      "push": true,
      "sms": true,
      "email": true
    },
    "marketing": {
      "push": false,
      "sms": false,
      "email": false
    }
  }
}
```

### Important design consideration

Not every notification is optional.

Some transactional or security notifications may be necessary to provide the service or protect an account.

Preference rules should distinguish:

- Mandatory service notifications.
- Optional notifications.
- Marketing communications.
- Security and account notifications.

Applicable legal requirements and consent rules must also be respected.

### Where should preferences be checked?

Usually before creating channel-specific delivery jobs.

However, check them again before delivery when a notification may have been queued for a long time and the user could have changed preferences in the meantime.

This avoids sending an outdated scheduled notification after the user disables that category.

---

## 10. Template Management

Templates separate notification content from application logic.

Instead of hardcoding messages in every service, store reusable templates.

Example:

```text
Template: driver-assigned

Subject:
Your driver has been assigned

Body:
Hi {{passengerName}},

{{driverName}} has been assigned to your booking.

Booking ID: {{bookingId}}
Vehicle: {{vehicleNumber}}
```

The template engine replaces variables with actual values.

### Why templates matter

- Consistent messaging.
- Easier localization.
- Content changes without changing business services.
- Channel-specific formatting.
- Better content governance.

### Localization

A user may prefer Hindi or English.

```text
driver-assigned.en
driver-assigned.hi
driver-assigned.fr
```

The template engine selects the correct locale.

### Template versioning

If a template changes, already scheduled notifications may need to retain their original content or use the latest version, depending on the business requirement.

Define this behavior explicitly.

### Security considerations

- Escape untrusted content appropriately.
- Prevent template injection.
- Avoid including unnecessary personal information.
- Validate required variables.
- Keep secrets out of templates.

---

## 11. Queue-Based Processing

Notification delivery should generally be asynchronous.

Suppose a major event generates 100,000 notifications.

Sending all of them synchronously would create high latency and could overwhelm external providers.

Instead, use queues.

```text
Business Event
      |
      v
Notification Service
      |
      v
Notification Queue
      |
      +------> Push Workers
      |
      +------> SMS Workers
      |
      +------> Email Workers
```

Workers process messages independently.

### Benefits

- Absorbs traffic spikes.
- Allows independent channel scaling.
- Supports retries.
- Prevents provider slowness from blocking business requests.
- Provides a buffer during provider outages.

### Separate queues by channel

A useful design is to maintain separate queues for push, SMS, and email.

Why?

Suppose the SMS provider is unavailable.

If every channel shares one queue, SMS failures and retries could delay push and email processing.

Separate queues provide resource isolation.

You can also separate queues by priority.

---

## 12. Priority-Based Notification Processing

Not all notifications should wait equally long.

Example:

| Priority | Notification | Expected handling |
|---|---|---|
| Critical | Security alert | Immediate |
| High | Booking cancellation | Fast |
| Normal | Booking confirmation | Standard |
| Low | Marketing campaign | Best effort |

A single queue can create head-of-line blocking.

For example, a large promotional campaign may delay an urgent booking cancellation notification.

### Solution: Priority queues

```text
Notification Router
        |
   +----+----+
   |    |    |
   v    v    v
Critical High Normal
Queue   Queue Queue
   |    |    |
   v    v    v
Workers Workers Workers
```

Workers should prioritize higher-priority queues while ensuring lower-priority work does not starve indefinitely.

### Important consideration

Do not create too many priority levels.

A small number of meaningful priority classes is easier to operate.

---

## 13. Retry Strategy

External providers may fail temporarily.

Examples:

- Network timeout.
- HTTP 429 rate limit.
- HTTP 500 server error.
- Temporary provider outage.
- Connection failure.

Retries can improve delivery reliability.

### Exponential backoff

A common retry delay strategy is:

\[
delay = \min(maxDelay, baseDelay \times 2^{attempt})
\]

Example with a 5-second base delay:

| Attempt | Delay |
|---|---:|
| 1 | 5 seconds |
| 2 | 10 seconds |
| 3 | 20 seconds |
| 4 | 40 seconds |
| 5 | 80 seconds |

Add random jitter to prevent many workers from retrying simultaneously.

### Retry classification

Not every error should be retried.

| Error | Typical handling |
|---|---|
| HTTP 429 | Retry after provider-specified delay when available |
| HTTP 500 | Retry with backoff |
| Network timeout | Retry carefully |
| Invalid phone number | Permanent failure |
| Invalid email address | Permanent failure |
| Authentication failure | Pause or alert; fix credentials |
| User unsubscribed | Do not retry |
| Invalid template | Permanent failure until corrected |

### Dead-letter queue

After the retry policy is exhausted, move the message to a dead-letter queue.

```text
Notification Queue
        |
        v
    Worker
        |
   +----+----+
   |         |
Success    Failure
   |         |
   v         v
Complete   Retry
              |
         Max retries?
              |
         +----+----+
         |         |
        No        Yes
         |         |
         v         v
       Queue      DLQ
```

Dead-letter messages should be investigated and replayed only after the underlying problem is understood.

---

## 14. Duplicate Notifications and Idempotency

Duplicate notifications can happen when:

1. A worker sends a message successfully.
2. The provider accepts the request.
3. The worker crashes before recording success.
4. The queue redelivers the message.
5. The worker sends the same message again.

This is a classic distributed-systems problem.

### Idempotency key

Use a stable delivery identifier.

Example:

```text
notificationId = N123
channel = PUSH

idempotencyKey = N123:PUSH
```

The worker checks whether this logical delivery has already been completed.

Where supported, pass the idempotency key to the external provider as well.

### Important limitation

If a provider does not support idempotency, a timeout after sending creates uncertainty:

- The provider may have accepted the message.
- The provider may not have accepted the message.
- The response may simply have been lost.

Blindly retrying can create duplicates.

Exactly-once delivery to an external recipient is generally not something the notification platform can guarantee by itself.

The system should define acceptable duplicate behavior and use provider delivery records or status APIs when available.

---

## 15. Provider Abstraction and Failover

A notification platform should avoid tightly coupling business services to a specific provider.

For example:

```text
Notification Service
        |
        v
   Channel Adapter
        |
   +----+----+
   |         |
   v         v
Provider A Provider B
```

A provider adapter hides provider-specific APIs behind a common internal interface.

Example:

```text
sendPushNotification()
sendSms()
sendEmail()
```

The implementation translates the internal request into the provider's API format.

### Provider failover

Suppose SMS Provider A becomes unavailable.

The platform may route new deliveries to Provider B.

However, failover needs care:

- Check whether Provider A already accepted the message.
- Avoid sending through both providers unnecessarily.
- Maintain provider-specific rate limits.
- Verify that Provider B supports the required destination.
- Preserve delivery tracking across providers.

Provider failover should be based on explicit health and routing policies, not simply on every request failure.

---

## 16. Rate Limiting and Provider Quotas

External providers impose limits.

For example:

```text
SMS Provider:
100 messages/second

Email Provider:
5,000 messages/minute

Push Provider:
Provider-specific quotas
```

If the notification platform sends messages faster than the provider allows, it may receive rate-limit errors.

### Solution

Apply rate limiting at the provider adapter or worker layer.

```text
SMS Queue
    |
    v
SMS Workers
    |
    v
Rate Limiter
    |
    v
SMS Provider
```

Useful techniques:

- Token bucket.
- Leaky bucket.
- Concurrency limits.
- Per-tenant quotas.
- Provider-specific throttling.

### Dynamic throttling

If the provider starts returning HTTP 429 responses, reduce the send rate and respect its retry guidance.

Do not respond to rate limiting by immediately increasing worker concurrency.

---

## 17. Fanout-on-Write vs Fanout-on-Read

Fanout describes how notifications are distributed to recipients.

### Fanout-on-Write

When an event occurs, the system immediately creates delivery work for all recipients.

Example: A driver is assigned to a booking.

The notification platform creates the necessary recipient notifications immediately.

```text
Event
  |
  v
Notification Service
  |
  +------> Recipient 1
  |
  +------> Recipient 2
  |
  +------> Recipient 3
```

**Advantages:**

- Fast delivery after event creation.
- Recipient-specific delivery work is already prepared.
- Suitable for time-sensitive notifications.

**Disadvantages:**

- Large fanout can generate enormous write volume.
- Popular events can create sudden spikes.
- More storage and queue operations.

### Fanout-on-Read

The system stores the notification or event once and determines the relevant recipient data when the user retrieves notifications.

This is useful for some in-app feeds and large audience systems.

**Advantages:**

- Less upfront fanout work.
- Useful for large audiences.
- Can reduce storage duplication.

**Disadvantages:**

- Read operations become more expensive.
- More work happens when users retrieve their feeds.
- Delivery is not immediate unless combined with another mechanism.

### Choosing a strategy

Use fanout-on-write for small or moderate recipient groups and time-sensitive delivery.

Consider fanout-on-read for large audience feeds where immediate per-recipient delivery is not required.

A hybrid approach is often appropriate.

---

## 18. Scheduled Notifications

Some notifications must be sent in the future.

Examples:

- Trip reminder 30 minutes before pickup.
- Payment reminder tomorrow.
- Appointment reminder.
- Scheduled marketing campaign.

A scheduler stores the intended execution time.

```text
Notification Request
        |
        v
Scheduled Notification Store
        |
        v
Scheduler
        |
   Is due?
        |
        v
Notification Queue
        |
        v
Delivery Worker
```

### Important design considerations

- Time zones.
- Daylight saving time.
- Cancellation.
- Rescheduling.
- Duplicate scheduler execution.
- Late execution.
- Large scheduled batches.

Store timestamps consistently, typically in UTC, and retain the intended timezone where needed for user-facing schedules.

### Efficient scheduling

Avoid repeatedly scanning an enormous notification table without suitable indexes.

Possible approaches include:

- Indexed due-time queries.
- Time buckets.
- Delayed message queues.
- Dedicated scheduling systems.

The scheduler should safely claim work so multiple instances do not process the same scheduled notification unnecessarily.

---

## 19. Delivery Tracking and Webhooks

External providers often send callbacks indicating delivery status.

Example:

```text
Notification Platform
        |
        v
SMS Provider
        |
        v
Recipient Device
        |
        v
Provider Delivery Report
        |
        v
Webhook Endpoint
        |
        v
Delivery Status Update
```

Possible delivery statuses:

- Accepted.
- Sent.
- Delivered.
- Failed.
- Rejected.
- Expired.

### Webhook handling

Provider webhooks must be treated as external, potentially repeated messages.

Important practices:

1. Verify webhook signatures.
2. Validate the provider message ID.
3. Deduplicate webhook events.
4. Handle out-of-order status updates.
5. Persist the webhook before acknowledging it.
6. Update the delivery state safely.
7. Monitor webhook failures.

Do not assume every provider reports the same status lifecycle.

A push provider may confirm acceptance without knowing whether the user actually saw the notification.

---

## 20. In-App Notifications

In-app notifications are different from push, SMS, and email.

They usually require persistent notification records that users can retrieve later.

Example:

```text
in_app_notifications
--------------------
id
user_id
title
body
category
read_at
created_at
```

### Common operations

```http
GET /api/notifications
GET /api/notifications/unread-count
PATCH /api/notifications/N123/read
PATCH /api/notifications/read-all
```

### Real-time updates

A WebSocket or SSE connection can notify the frontend that a new notification has arrived.

```text
Notification Service
        |
        v
Real-time Gateway
        |
        v
WebSocket / SSE
        |
        v
Browser / Mobile App
```

The persistent database remains the source for notification history.

The real-time connection provides immediate updates but should not be the only way to retrieve notifications.

If the client disconnects, it should fetch unread notifications when it reconnects.

---

## 21. Multi-Tenant Notification Architecture

In a multi-tenant platform, notifications may have tenant-specific configuration.

Examples:

- Different branding.
- Different SMS providers.
- Different email domains.
- Different templates.
- Different sender identities.
- Different delivery quotas.

A notification record should carry the tenant context.

```json
{
  "notificationId": "N123",
  "tenantId": "T001",
  "recipientId": "P123",
  "eventType": "BookingCreated"
}
```

### Important considerations

- Tenant isolation.
- Tenant-specific credentials.
- Per-tenant rate limits.
- Tenant-specific preferences.
- Template ownership.
- Audit logs.
- Data retention.
- Provider usage and cost reporting.

Do not allow a tenant to access another tenant's notification records or provider configuration.

---

## 22. Capacity Planning Example

Suppose the platform has:

- 1 million daily active users.
- Average 5 notifications per user per day.
- Peak traffic is 5 times the average.

### Daily notification volume

\[
1,000,000 \times 5 = 5,000,000
\]

The platform handles 5 million notifications per day.

### Average notifications per second

\[
\frac{5,000,000}{86,400}
\approx 57.9
\]

Approximately 58 notifications per second on average.

### Peak notifications per second

\[
58 \times 5 = 290
\]

Approximately 290 notifications per second during peak periods.

### Multi-channel expansion

Suppose 40% of notifications use two channels and the remaining 60% use one channel.

Average deliveries per notification:

\[
(0.6 \times 1) + (0.4 \times 2) = 1.4
\]

Total daily delivery operations:

\[
5,000,000 \times 1.4 = 7,000,000
\]

Average delivery operations per second:

\[
\frac{7,000,000}{86,400}
\approx 81
\]

At a 5x peak, approximately 405 delivery operations per second are required.

This distinction matters: notification requests and channel-specific delivery operations are different workloads.

---

## 23. Monitoring and Observability

A notification platform needs both technical and business metrics.

### Technical metrics

- Queue depth.
- Oldest queued message age.
- Processing throughput.
- Worker concurrency.
- Retry count.
- Dead-letter queue size.
- Provider API latency.
- Provider error rate.
- Webhook processing latency.

### Business metrics

- Notifications created.
- Delivery success rate.
- Delivery failure rate.
- Time from creation to provider acceptance.
- Time from creation to confirmed delivery.
- Channel usage.
- Cost per notification.
- User opt-out rate.

### Important distinction

A queue can have a low message count but still contain old messages.

For example, 100 messages waiting for 30 minutes may be more concerning than 10,000 messages waiting for one second.

Always monitor both queue depth and oldest message age.

### Trace example

```text
BookingCreated
      |
      v
Notification Created
      |
      v
Preference Check
      |
      v
Template Rendered
      |
      v
SMS Queue
      |
      v
SMS Worker
      |
      v
Provider API
      |
      v
Delivery Webhook
```

A correlation ID should connect the steps so that an individual notification can be investigated across the entire workflow.

---

## 24. Security and Privacy

Notifications often contain personal or sensitive information.

Important protections include:

- Encrypt data in transit.
- Protect stored notification content.
- Restrict access to delivery logs.
- Avoid logging passwords, tokens, or full payment details.
- Use verified recipient addresses and phone numbers.
- Verify webhook signatures.
- Protect provider credentials.
- Enforce tenant-level authorization.
- Apply retention and deletion policies.
- Avoid including sensitive information in push notification previews.

For example, a lock-screen notification should not expose full financial or personal details.

Prefer:

"Your payment requires attention."

Avoid putting unnecessary sensitive information into the notification body.

---

## 25. Common Failure Scenarios

| Failure | Recommended response |
|---|---|
| SMS provider unavailable | Retry with backoff or use approved fallback |
| Email provider rate-limited | Reduce sending rate |
| Push token invalid | Mark token invalid and stop using it |
| Worker crashes | Allow safe message redelivery |
| Duplicate event | Deduplicate using stable event ID |
| Provider accepted but response lost | Reconcile before retrying if possible |
| Notification queue grows | Scale workers within provider limits |
| Template missing | Record failure and alert |
| User disables preferences | Suppress eligible pending deliveries |
| Webhook arrives twice | Deduplicate webhook processing |

A robust notification platform must be designed for partial failures rather than assuming every provider call succeeds.

---

## 26. Common Architectural Mistakes

1. Sending notifications synchronously from business APIs.
2. Making every business service integrate directly with every provider.
3. Using one queue for all channels and priorities.
4. Treating provider acceptance as confirmed delivery.
5. Retrying permanent errors.
6. Ignoring duplicate delivery risks.
7. Not respecting provider quotas.
8. Failing to verify provider webhooks.
9. Ignoring user preferences.
10. Using a single template for all locales and channels.
11. Storing sensitive information unnecessarily.
12. Failing to track delivery attempts separately from notification requests.
13. Scaling workers without considering provider rate limits.
14. Replaying events and accidentally sending duplicate notifications.

---

## 27. Practical Architecture Decision Checklist

### Ingestion
- [ ] Are notification requests asynchronous?
- [ ] Can business events generate multiple notifications?
- [ ] Is event deduplication implemented?
- [ ] Is the notification request durable?

### Routing
- [ ] Are channels selected using explicit policies?
- [ ] Are user preferences respected?
- [ ] Are priority levels defined?
- [ ] Is provider selection configurable?

### Delivery
- [ ] Are channel-specific queues used?
- [ ] Are workers independently scalable?
- [ ] Are provider rate limits enforced?
- [ ] Are retries bounded?
- [ ] Is a dead-letter queue available?

### Reliability
- [ ] Are duplicate delivery risks addressed?
- [ ] Are provider failures handled?
- [ ] Are webhooks authenticated?
- [ ] Can scheduled notifications be cancelled?
- [ ] Can delivery status be reconciled?

### Operations
- [ ] Are queue age and depth monitored?
- [ ] Is provider latency tracked?
- [ ] Is delivery success measured?
- [ ] Is cost tracked per channel and tenant?
- [ ] Is there a process for investigating failed deliveries?

---

## 28. Revision Questions

1. Why should notification delivery be asynchronous?
2. What is the difference between a business event and a notification request?
3. Why separate notification records from delivery records?
4. What are the main notification lifecycle states?
5. What is the difference between provider acceptance and confirmed delivery?
6. How should channel selection work?
7. Why should user preferences be checked before delivery?
8. What is the purpose of a template engine?
9. Why use separate queues for push, SMS, and email?
10. What is priority-based notification processing?
11. How does exponential backoff work?
12. What is a dead-letter queue?
13. How can duplicate notifications occur?
14. Why can't exactly-once delivery generally be guaranteed end to end?
15. What is provider failover?
16. How should provider rate limits be handled?
17. What is the difference between fanout-on-write and fanout-on-read?
18. How would you design scheduled notifications?
19. Why should webhook processing be idempotent?
20. Why should in-app notification history be persisted separately from WebSocket delivery?
21. How would you isolate notification configurations across tenants?
22. Which metrics indicate that a notification platform is falling behind?

---

## 29. Practical Exercise

Design a notification platform for a multi-tenant ride-booking application.

### System requirements

- 500,000 daily active users.
- Average 6 notifications per user per day.
- Push, SMS, email, and in-app channels.
- Multiple tenants with different provider configurations.
- Urgent booking updates must be prioritized.
- Promotional messages can be delayed.
- Providers may become temporarily unavailable.
- Users can configure notification preferences.

### Design questions

1. What services and components would you create?
2. What would be the notification request data model?
3. Would you use one queue or separate queues for each channel?
4. How would you prioritize urgent notifications?
5. How would you prevent duplicate delivery?
6. How would you handle provider failover?
7. How would you manage user preferences?
8. How would you schedule notifications?
9. How would you handle a provider outage lasting 30 minutes?
10. How would you calculate the required worker capacity?
11. Which metrics and alerts would you configure?
12. How would you ensure one tenant cannot access another tenant's notification data?

**Key takeaway:** A notification platform is a distributed delivery system, not simply an SMS or email API integration. Reliable ingestion, channel isolation, provider abstraction, idempotency, rate limiting, delivery tracking, and user preferences are all essential parts of its architecture.

### Practice focus

Design the queue and worker architecture first. Think about what happens when the SMS provider is unavailable for 30 minutes while urgent booking notifications continue arriving.