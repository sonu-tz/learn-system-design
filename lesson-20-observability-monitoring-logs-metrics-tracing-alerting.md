# Lesson 20 — Observability and Monitoring: Logs, Metrics, Distributed Tracing, and Alerting

In this lesson, we'll learn how to understand what's happening inside a distributed system when something goes wrong. We'll cover logs, metrics, distributed tracing, alerting, SLOs, and incident response.

This builds directly on our previous lessons about microservices, message brokers, real-time systems, and failure handling.

## 1. Why Observability Matters

As a system grows, a single user request may travel through multiple components:

- API Gateway
- Authentication Service
- Booking Service
- PostgreSQL
- Message Broker
- Notification Service
- Real-time Gateway
- WebSocket clients

If a booking request takes 8 seconds or fails unexpectedly, how do we identify where the problem occurred?

Was it:
- API Gateway latency?
- A slow database query?
- A message broker backlog?
- An overloaded consumer?
- An external service timeout?
- A network issue?

Without observability, we are guessing.

**Observability is the ability to understand a system's internal state by examining the data it produces.**

It helps answer three important questions:

1. What is happening?
2. Where is it happening?
3. Why is it happening?

### Monitoring vs Observability

| Monitoring | Observability |
|---|---|
| Tracks predefined metrics | Helps investigate unknown problems |
| Tells us something is wrong | Helps explain why it is wrong |
| Uses dashboards and alerts | Uses logs, metrics, and traces |
| Example: API error rate increased | Example: Which dependency caused the errors? |

Monitoring is part of observability. Both are necessary.

---

## 2. The Three Pillars of Observability

The three commonly recognized pillars are:

1. Logs
2. Metrics
3. Distributed Traces

Each provides a different perspective.

### 2.1 Logs

A log is a timestamped record of an event that occurred inside an application.

Example:

```json
{
  "timestamp": "2026-10-30T10:15:30Z",
  "level": "ERROR",
  "service": "booking-service",
  "message": "Failed to create booking",
  "bookingId": "B123",
  "errorCode": "DB_TIMEOUT",
  "traceId": "abc123"
}
```

This tells us:
- When the event occurred.
- Which service generated it.
- What happened.
- Which booking was affected.
- Which distributed trace to investigate.

### Log Levels

| Level | Purpose |
|---|---|
| DEBUG | Detailed information for debugging |
| INFO | Normal application events |
| WARN | Unexpected situations that may need attention |
| ERROR | An operation failed |
| FATAL | A critical failure that prevents continued operation |

### Structured Logging

Instead of writing:

```text
Booking B123 failed because database timed out
```

Prefer structured JSON:

```json
{
  "event": "booking_creation_failed",
  "bookingId": "B123",
  "reason": "database_timeout",
  "durationMs": 5000
}
```

Why?

- Easier to search and filter.
- Easier to aggregate.
- Consistent fields across services.
- Easier integration with log platforms.

**Important:** Never log passwords, access tokens, payment details, or other sensitive information.

---

## 3. Metrics

A metric is a numerical measurement collected over time.

Examples:

- Requests per second.
- API error rate.
- CPU utilization.
- Memory usage.
- Database connections.
- Queue depth.
- WebSocket connections.

Unlike logs, metrics are designed for aggregation and trend analysis.

### 3.1 Types of Metrics

#### Counter

A counter only increases over time, except when the application restarts.

Examples:
- Total API requests.
- Total failed requests.
- Total bookings created.

Example:

```text
booking_requests_total = 125000
```

We can calculate the request rate by examining how quickly the counter increases.

#### Gauge

A gauge represents a value that can increase or decrease.

Examples:
- Active WebSocket connections.
- Current CPU usage.
- Number of active database connections.
- Queue depth.

```text
active_connections = 8500
```

#### Histogram

A histogram measures the distribution of observations by grouping them into buckets.

Examples:
- API request duration.
- Database query duration.
- Message processing duration.

Suppose request durations are:

```text
100ms, 150ms, 200ms, 500ms, 1200ms
```

A histogram can group them into buckets such as:

| Duration | Requests |
|---|---:|
| ≤ 100ms | 1 |
| ≤ 250ms | 3 |
| ≤ 500ms | 4 |
| ≤ 1s | 4 |
| ≤ 2s | 5 |

Histograms are especially useful for calculating percentiles across aggregated observations.

#### Summary

A summary calculates statistical values such as quantiles over a configured observation window.

Histograms are generally easier to aggregate across multiple instances. Summary quantiles generally cannot be accurately combined across instances.

### 3.2 Metrics Cardinality

Cardinality means the number of unique combinations of metric labels.

Consider:

```text
http_requests_total{
  method="GET",
  endpoint="/bookings",
  status="200"
}
```

This is reasonable.

But imagine adding:

```text
bookingId="B123"
userId="U456"
```

Every booking or user can create a new time series.

This causes high cardinality, which increases monitoring-system memory, storage, and processing costs.

**Rule:** Use metrics for bounded dimensions such as service, endpoint, method, and status code. Use logs and traces for unique identifiers such as booking IDs and user IDs.

---

## 4. RED Method, USE Method, and Golden Signals

These are practical frameworks for deciding what to monitor.

### 4.1 RED Method

Useful for request-driven services.

RED stands for:

- **Rate:** How many requests are being processed?
- **Errors:** How many requests are failing?
- **Duration:** How long do requests take?

Example for Booking Service:

| Metric | Example |
|---|---:|
| Request rate | 1,200 requests/sec |
| Error rate | 2.5% |
| P95 duration | 850ms |

### 4.2 USE Method

Useful for infrastructure and resource monitoring.

USE stands for:

- **Utilization:** How busy is the resource?
- **Saturation:** How much work is waiting?
- **Errors:** Are resource operations failing?

Example:

| Resource | Utilization | Saturation | Errors |
|---|---:|---:|---:|
| CPU | 85% | High run queue | 2 |
| Database pool | 95% | 40 waiting | 12 |
| Disk | 70% | 5 pending I/O | 0 |

### 4.3 Four Golden Signals

The four golden signals are:

1. Latency
2. Traffic
3. Errors
4. Saturation

These provide a useful starting point for monitoring almost any service.

| Signal | Example |
|---|---|
| Latency | P95 API response time |
| Traffic | Requests per second |
| Errors | HTTP 5xx percentage |
| Saturation | CPU, memory, connection pool usage |

---

## 5. Why Percentiles Matter

Suppose an API receives 10,000 requests.

Most requests complete quickly, but some are very slow.

| Percentile | Latency |
|---|---:|
| P50 | 100ms |
| P90 | 250ms |
| P95 | 500ms |
| P99 | 2,000ms |

Interpretation:

- P50: 50% of requests completed within 100ms.
- P95: 95% completed within 500ms.
- P99: 99% completed within 2 seconds.

The slowest 1% may still affect many users.

For example, at 10,000 requests per minute, 1% represents 100 slow requests every minute.

**Why not just monitor average latency?**

An average can hide tail latency.

Imagine 99 requests take 100ms and one request takes 10 seconds. The average is approximately 199ms, which does not make the slow request obvious.

Monitor P50, P95, and P99 where appropriate.

---

## 6. Distributed Tracing

Logs tell us what happened. Metrics tell us how frequently or how slowly something happens. Traces show how a request travels through multiple services.

A distributed trace represents the journey of one operation across a distributed system.

### Example: Booking Creation

```text
Passenger Client
      |
      v
API Gateway
      |
      v
Booking Service
      |
      +------> PostgreSQL
      |
      +------> Message Broker
                    |
                    v
             Notification Service
```

A trace records the operations involved in this journey.

### 6.1 Trace and Span

**Trace:** The complete journey of a request.

**Span:** One individual operation within that journey.

Example:

```text
Trace: Create Booking

|--- API Gateway -----------  50ms
    |--- Booking Service --- 300ms
        |--- DB Query ----- 180ms
        |--- Publish Event - 40ms
            |--- Notification Service -- 120ms
```

Each span can contain:
- Operation name.
- Start time.
- Duration.
- Service name.
- Parent span ID.
- Status.
- Attributes and events.

The trace ID connects all spans belonging to the same trace.

### 6.2 Trace Context Propagation

How does the Booking Service know that an incoming request belongs to a particular trace?

The trace context is propagated between services.

The W3C Trace Context standard defines headers such as:

```text
traceparent
tracestate
```

A simplified traceparent header looks like:

```text
00-<trace-id>-<parent-span-id>-01
```

The receiving service extracts the context and creates a child span.

Without context propagation, a trace can become disconnected at service boundaries.

### 6.3 Correlation ID vs Trace ID

These concepts are related but not identical.

| Correlation ID | Trace ID |
|---|---|
| Connects related application events | Identifies a distributed trace |
| Often created by an application or gateway | Usually managed by tracing instrumentation |
| Can be used in logs and messages | Connects spans across services |
| May span multiple related operations | Represents one traced operation |

For example, a booking ID can correlate multiple operations over the booking lifecycle, while a trace ID identifies one particular request journey.

Do not use a booking ID as a trace ID.

---

## 7. Observability in Asynchronous Systems

Tracing becomes more challenging when messages and background workers are involved.

Consider:

```text
Booking Service
      |
      v
Message Broker
      |
      v
Booking Consumer
      |
      v
Notification Service
```

The original HTTP request may finish before the consumer processes the message.

How do we connect these operations?

The producer should inject trace context into message metadata or headers. The consumer extracts that context and continues the trace, or creates a linked trace where appropriate.

Useful metadata includes:

```json
{
  "eventId": "EVT123",
  "bookingId": "B123",
  "traceparent": "00-...",
  "eventType": "BOOKING_CREATED"
}
```

### What Should We Monitor?

| Component | Important Metrics |
|---|---|
| Message broker | Queue depth, publish rate, delivery rate |
| Consumer | Processing rate, processing duration |
| Consumer group | Consumer lag |
| Dead-letter queue | Message count |
| Retry system | Retry count and failure rate |

**Important:** Queue depth alone is not enough.

A queue with 10,000 messages may be healthy if consumers are processing them quickly. A queue with 100 messages may be unhealthy if the oldest message has been waiting for 30 minutes.

Monitor both queue depth and oldest message age.

---

## 8. SLI, SLO, and SLA

These terms define how we measure and communicate service reliability.

### SLI — Service Level Indicator

An SLI is a measurement of service behavior.

Examples:
- Successful request percentage.
- P95 latency.
- Availability.
- Message processing delay.

### SLO — Service Level Objective

An SLO is the target for an SLI over a defined period.

Example:

```text
99.9% of eligible booking API requests
should complete successfully within 30 days.
```

### SLA — Service Level Agreement

An SLA is a formal agreement with users or customers. It may define consequences if service commitments are not met.

| Term | Meaning |
|---|---|
| SLI | What we measure |
| SLO | What target we aim for |
| SLA | What we formally promise |

### 8.1 Availability Calculation

Formula:

```text
Availability =
Successful eligible requests / Total eligible requests × 100
```

Suppose a service receives 1,000,000 eligible requests in a month and 999,000 succeed.

```text
Availability = 999,000 / 1,000,000 × 100

Availability = 99.9%
```

### 8.2 Error Budget

An error budget is the amount of unreliability allowed by an SLO.

For a 99.9% availability target:

```text
Error Budget = 100% - 99.9%

             = 0.1%
```

For 1,000,000 eligible requests:

```text
Allowed failures = 1,000,000 × 0.001

                 = 1,000 requests
```

If 1,000 requests fail, the error budget is exhausted.

Error budgets help teams balance feature development and reliability work.

A rapidly consumed error budget may justify slowing risky releases and prioritizing reliability improvements.

---

## 9. Alerting Principles

Monitoring without useful alerts can leave teams discovering failures too late.

However, too many alerts create alert fatigue.

An alert should generally be:
- Actionable.
- Relevant to user impact or a meaningful risk.
- Supported by a clear investigation path.
- Assigned to an appropriate owner.

### Bad Alert

```text
CPU usage is 70%
```

This may not indicate a problem.

### Better Alert

```text
Booking Service P95 latency > 2 seconds
for 10 consecutive minutes
and error rate > 5%.
```

This is closer to a user-impacting symptom.

### 9.1 Threshold Alerts

Example:

```text
Database connections > 90%
```

Easy to understand, but static thresholds can generate noise.

### 9.2 Rate-of-Change Alerts

Detect rapid changes.

Example:

```text
API error rate increased from 0.5% to 8%
within 5 minutes.
```

### 9.3 SLO Burn-Rate Alerts

Burn rate measures how quickly an error budget is being consumed relative to its allowed rate.

For example:
- A burn rate of 1 means consuming the budget at the expected rate.
- A burn rate of 10 means consuming it ten times faster than expected.

Multi-window burn-rate alerts can detect both fast, severe incidents and slower reliability degradation.

### Avoid Alert Fatigue

Do not alert on every warning or minor metric fluctuation.

Prefer alerts for:
- High user-impacting error rates.
- Sustained high latency.
- Queue age exceeding an important processing deadline.
- Database saturation.
- Failed critical background workflows.
- SLO budget burning too quickly.

---

## 10. Designing Useful Dashboards

A dashboard should answer a question, not simply display every metric available.

### Service Dashboard

For each service, consider:

| Category | Dashboard Panels |
|---|---|
| Traffic | Requests per second |
| Reliability | Error percentage |
| Performance | P50, P95, P99 latency |
| Resources | CPU, memory |
| Dependencies | Database and external API latency |
| Saturation | Connection pool usage, worker queue |

### Database Dashboard

Monitor:
- Active connections.
- Connection pool saturation.
- Query latency.
- Slow queries.
- Lock waits.
- Deadlocks.
- Replication lag.
- CPU, memory, disk I/O.

### Cache Dashboard

Monitor:
- Cache hit ratio.
- Cache miss rate.
- Evictions.
- Memory usage.
- Command latency.
- Connection errors.

### Real-Time Dashboard

Monitor:
- Active WebSocket connections.
- Connection establishment rate.
- Disconnect rate.
- Reconnection rate.
- Messages published per second.
- Messages delivered per second.
- Message delivery latency.
- Slow-client count.

### Message Broker Dashboard

Monitor:
- Queue or topic throughput.
- Queue depth.
- Oldest message age.
- Consumer lag.
- Retry rate.
- Dead-letter queue size.
- Consumer processing duration.

---

## 11. Incident Response

Observability is not only about collecting data. It must help teams respond effectively.

A typical incident response process:

```text
Detect
  |
  v
Triage
  |
  v
Mitigate
  |
  v
Investigate Root Cause
  |
  v
Resolve
  |
  v
Postmortem
```

### Step 1 — Detect

An alert or customer report identifies an issue.

Example: Booking API error rate reaches 10%.

### Step 2 — Triage

Determine:
- Which service is affected?
- When did the problem start?
- How many users are affected?
- Is it limited to one region or tenant?
- Did a recent deployment happen?

### Step 3 — Mitigate

Restore service first.

Possible actions:
- Roll back a deployment.
- Disable a problematic feature flag.
- Scale consumers.
- Reduce incoming load.
- Fail over to a healthy dependency.

### Step 4 — Root Cause Analysis

Use logs, metrics, and traces to identify the actual cause.

### Step 5 — Resolve and Verify

Confirm:
- Error rates returned to normal.
- Latency recovered.
- Backlogs are decreasing.
- No new failures are appearing.

### Step 6 — Postmortem

Document:
- What happened?
- User impact.
- Timeline.
- Root cause.
- Why existing safeguards did not prevent or detect it.
- Corrective and preventive actions.

A postmortem should focus on improving the system, not blaming individuals.

---

## 12. OpenTelemetry and Common Observability Tools

OpenTelemetry (OTel) is an open-source observability framework for generating, collecting, and exporting telemetry.

It supports:
- Traces.
- Metrics.
- Logs.

It provides APIs, SDKs, instrumentation libraries, and a Collector.

### Simplified Architecture

```text
Application
    |
    v
OpenTelemetry SDK
    |
    v
OpenTelemetry Collector
    |
    +------> Metrics Backend
    |
    +------> Trace Backend
    |
    +------> Log Backend
```

The Collector can receive, process, and export telemetry.

### Common Tools

| Tool | Common Purpose |
|---|---|
| OpenTelemetry | Instrumentation and telemetry collection |
| Prometheus | Metrics collection and querying |
| Grafana | Dashboards and visualization |
| Jaeger | Distributed tracing |
| Grafana Tempo | Distributed trace storage |
| Elasticsearch / OpenSearch | Log search and analytics |
| Kibana / OpenSearch Dashboards | Log visualization |
| Sentry | Application errors and performance monitoring |

These tools can be combined. They are not all direct replacements for one another.

For example, OpenTelemetry can instrument an application, Prometheus can store metrics, Grafana can visualize them, and Jaeger or Tempo can store and display traces.

---

## 13. Sampling, Storage, and Privacy

Observability data has a cost.

A high-volume application can generate millions of log entries and spans.

### Sampling

Sampling means collecting only a portion of telemetry.

Example:

```text
Incoming traces: 100,000
Sampled traces:   10,000

Sampling rate: 10%
```

Advantages:
- Reduced storage costs.
- Lower telemetry processing overhead.
- Less network traffic.

Disadvantages:
- Some events are not recorded.
- Rare failures may be missed with naive sampling.

Common approaches:

- Head-based sampling: Decide near the start of a trace.
- Tail-based sampling: Decide after observing the trace, often using its outcome or characteristics.

Tail-based sampling can retain traces with errors or unusually high latency, but requires more infrastructure and buffering.

### Data Retention

Not all telemetry needs the same retention period.

For example:
- High-volume debug logs: short retention.
- Application errors: longer retention.
- Aggregated service metrics: longer-term trend analysis.
- Detailed traces: sampling and retention based on investigation needs.

Also consider:
- PII and secret redaction.
- Access controls.
- Encryption.
- Tenant isolation.
- Retention policies.
- Telemetry storage costs.

---

## 14. Worked Example: Investigating a Slow Booking Request

Imagine users report that creating a booking takes 5 seconds.

The service dashboard shows:

| Metric | Value |
|---|---:|
| Request rate | 1,500 RPS |
| Error rate | 1.2% |
| P95 latency | 5 seconds |
| CPU utilization | 45% |
| DB connection pool utilization | 95% |

CPU is not saturated, but database connections are nearly exhausted.

Now inspect a distributed trace:

```text
Create Booking Trace — 5,000ms

API Gateway
  └── 50ms

Booking Service
  ├── Validate Request       20ms
  ├── Check Availability   300ms
  ├── Acquire DB Connection 2,800ms
  ├── Insert Booking        180ms
  └── Publish Event          50ms
```

The trace reveals the problem: the application spends 2.8 seconds waiting for a database connection.

The SQL insert itself takes only 180ms.

Possible causes:
- Connection pool is too small.
- Connections are held too long.
- Slow queries consume connections.
- A traffic spike exhausted the pool.
- A connection leak exists.

The correct response is to investigate connection usage and query behavior, not blindly add more application servers.

This illustrates why metrics and traces complement each other:
- Metrics reveal that the DB connection pool is saturated.
- Traces reveal where requests are waiting.
- Logs provide detailed failure events and contextual information.

---

## 15. Common Observability Mistakes

1. Monitoring average latency only.
2. Logging unstructured text without consistent fields.
3. Using unique IDs as metric labels and creating high cardinality.
4. Collecting logs but not correlating them with traces.
5. Losing trace context when publishing messages.
6. Alerting on CPU or memory without considering user impact.
7. Monitoring queue depth but ignoring oldest message age.
8. Sampling away all useful error traces.
9. Logging secrets or sensitive user data.
10. Creating dashboards without clear operational questions.
11. Treating successful deployment as proof of system health.
12. Having alerts without owners or runbooks.

---

## 16. Practical Observability Checklist

### Application Services
- [ ] Request rate
- [ ] Error rate
- [ ] P50, P95, P99 latency
- [ ] CPU and memory
- [ ] Dependency latency
- [ ] Structured logs
- [ ] Distributed tracing

### Database
- [ ] Active connections
- [ ] Connection pool utilization
- [ ] Slow query count
- [ ] Query latency
- [ ] Lock waits and deadlocks
- [ ] Replication lag

### Message Broker
- [ ] Publish and consume rate
- [ ] Queue depth
- [ ] Oldest message age
- [ ] Consumer lag
- [ ] Retry rate
- [ ] Dead-letter queue

### Real-Time Infrastructure
- [ ] Active connections
- [ ] Reconnection rate
- [ ] Message delivery latency
- [ ] Publish and delivery throughput
- [ ] Slow clients
- [ ] Dropped messages

### Reliability
- [ ] SLIs and SLOs defined
- [ ] Error budget tracked
- [ ] Actionable alerts configured
- [ ] Incident runbooks available
- [ ] Postmortems completed

---

## 17. Revision Questions

Try answering these without looking at the notes.

1. What is the difference between monitoring and observability?
2. What are the three pillars of observability?
3. What is the difference between a counter, gauge, histogram, and summary?
4. Why is high metric cardinality a problem?
5. Explain the RED and USE methods.
6. Why should we monitor P95 and P99 instead of average latency alone?
7. What is a distributed trace?
8. What is the difference between a trace and a span?
9. How does trace context propagate between services?
10. What is the difference between a correlation ID and a trace ID?
11. How do we continue tracing across asynchronous message processing?
12. Explain SLI, SLO, and SLA.
13. What is an error budget?
14. What is a burn rate?
15. What is the difference between head-based and tail-based sampling?
16. Why should we monitor oldest message age in addition to queue depth?
17. How would you investigate a slow API using metrics, logs, and traces?
18. What is OpenTelemetry, and where does it fit into an observability architecture?

---

## 18. Practical Exercise

Design an observability strategy for our ride-booking platform.

Assume the system has:

- API Gateway
- Booking Service
- Driver Service
- PostgreSQL
- Redis
- Kafka
- Notification Service
- WebSocket Gateway

Define:

1. Three important metrics for each service.
2. Five alerts that should wake up an on-call engineer.
3. One SLO for booking creation.
4. How you would trace a booking request from the API Gateway through Kafka to the Notification Service.
5. How you would investigate a situation where booking creation is slow but the API error rate remains low.
6. How you would monitor delayed driver location updates.
7. Which telemetry would you sample, and which would you retain for every request?

**The key takeaway:** A distributed system is not production-ready merely because it works. We need to know when it is unhealthy, where the problem is, how users are affected, and whether recovery is successful.

### Practice focus

Work through the ride-booking exercise, especially the distributed trace and the five alerts. The goal is to design observability around user impact—not just collect as many metrics as possible.