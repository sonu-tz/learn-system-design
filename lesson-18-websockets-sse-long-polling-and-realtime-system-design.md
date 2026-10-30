# Lesson 18 — WebSockets, Server-Sent Events, Long Polling, and Real-Time System Design

This lesson is particularly important for applications that need live updates, such as ride-booking platforms, chat applications, stock dashboards, and collaborative tools.

Imagine a dispatcher monitoring 500 drivers on a map. Driver locations are changing continuously. How should the backend deliver those updates to the browser without repeatedly making hundreds of API requests?

We'll compare the major real-time communication approaches, understand their architecture and trade-offs, and design a scalable live-tracking system.

## 1. What Is Real-Time Communication?

Traditional HTTP communication follows a request-response model.

The client sends a request, and the server returns a response.

Example:

```text
Browser
   |
   | GET /api/drivers
   ▼
Server
   |
   | Driver data
   ▼
Browser
```

But what if driver locations change every second?

The browser needs updated information without requiring the user to refresh the page.

There are several approaches:

- Polling.
- Long polling.
- Server-Sent Events (SSE).
- WebSockets.

Each approach solves the problem differently.

---

## 2. Polling

Polling means the client repeatedly requests data from the server at fixed intervals.

Example:

```text
Browser                Server
   |                      |
   |---- GET /drivers --->|
   |<--- Driver Data -----|
   |                      |
   |      Wait 5 sec      |
   |                      |
   |---- GET /drivers --->|
   |<--- Driver Data -----|
   |                      |
   |      Wait 5 sec      |
   |                      |
   |---- GET /drivers --->|
```

For example, the browser requests driver locations every five seconds.

### Advantages

- Very simple to implement.
- Uses ordinary HTTP requests.
- Easy to understand and debug.
- Works with most infrastructure.

### Disadvantages

- Many unnecessary requests.
- Updates are delayed until the next poll.
- Repeated requests increase server and network load.
- Less efficient when updates happen frequently.

### Example calculation

Suppose:

- 500 dispatcher sessions.
- Each session polls every five seconds.

Requests per second:

\[
RPS = \frac{500}{5} = 100
\]

The backend receives 100 requests per second even if no driver location has changed.

If each request retrieves information about 500 drivers, the application may perform unnecessary processing and data transfer.

**Polling is suitable when updates are infrequent and a small delay is acceptable.**

---

## 3. Long Polling

Long polling improves on traditional polling.

Instead of immediately returning a response, the server holds the HTTP request open until:

- New data becomes available.
- A timeout occurs.
- The connection fails.

After receiving a response, the client immediately sends another request.

### Flow

```text
Browser                 Server
   |                       |
   |---- Request --------->|
   |                       |
   |    Wait for event     |
   |                       |
   |<---- New Data --------|
   |                       |
   |---- New Request ----->|
   |                       |
   |    Wait for event     |
   |                       |
   |<---- New Data --------|
```

### Advantages

- Updates can arrive without frequent polling intervals.
- Works over ordinary HTTP.
- Useful when infrastructure does not support persistent bidirectional connections.

### Disadvantages

- Requests must be repeatedly recreated.
- Many concurrent waiting requests consume resources.
- Connection management becomes important.
- Timeouts and reconnects require careful handling.

### Example

Suppose 500 dispatcher sessions each maintain one outstanding long-poll request.

Instead of generating 100 new requests every second, the system may have approximately 500 requests waiting for events.

However, long polling still involves repeated HTTP request cycles and associated connection-management overhead.

**Long polling is useful when you need near-real-time server updates but cannot use WebSockets or SSE.**

---

## 4. Server-Sent Events (SSE)

Server-Sent Events allow the server to continuously send events to the browser over a long-lived HTTP connection.

Unlike polling, the browser does not repeatedly request updates.

### Flow

```text
Browser                 Server
   |                       |
   |---- Open SSE -------->|
   |                       |
   |<---- Event 1 ----------|
   |                       |
   |<---- Event 2 ----------|
   |                       |
   |<---- Event 3 ----------|
   |                       |
   |<---- Event 4 ----------|
   |                       |
```

SSE is unidirectional:

**Server → Client**

The browser uses an `EventSource` connection to receive events.

Example event stream:

```text
event: driver-location
id: 101
data: {"driverId":101,"lat":28.61,"lng":77.23}

```

### Advantages

- Simple server-to-client communication.
- Uses standard HTTP.
- Supports automatic reconnection in browser implementations.
- Can use event IDs for resuming from a previous event where supported by the server.
- Good for notifications, dashboards, and live feeds.

### Disadvantages

- Communication is unidirectional.
- Client-to-server updates require separate HTTP requests or another channel.
- Browser connection limits can matter, particularly with HTTP/1.1.
- Authentication and proxy behavior need consideration.

### Common use cases

- Live dashboards.
- Notifications.
- Progress updates.
- Stock price feeds.
- Server-generated status updates.

**Use SSE when the server primarily sends updates to the client and the client does not need a persistent bidirectional channel.**

---

## 5. WebSockets

WebSockets provide a persistent, full-duplex communication channel between a client and server.

Full-duplex means both sides can send messages independently.

### Traditional HTTP vs. WebSocket

Traditional HTTP:

```text
Client ───── Request ─────> Server
Client <──── Response ───── Server
```

WebSocket:

```text
Client <==================> Server
          Persistent
          Connection
```

After an HTTP-based WebSocket handshake, the connection is upgraded to the WebSocket protocol.

The client and server can then exchange messages without creating a new HTTP request for every message.

### WebSocket connection lifecycle

1. Client sends an HTTP upgrade request.
2. Server accepts the upgrade.
3. A WebSocket connection is established.
4. Both sides exchange messages.
5. Connection remains open while healthy.
6. Either side can close the connection.

### Advantages

- Full-duplex communication.
- Low overhead for frequent messages compared with repeated HTTP request cycles.
- Suitable for interactive real-time applications.
- Supports client-to-server and server-to-client messages.

### Disadvantages

- Persistent connections require connection management.
- Reconnection and heartbeat logic are important.
- Scaling across multiple servers requires coordination.
- Load balancers and proxies must support WebSocket connections.
- Authentication and authorization must be handled throughout the connection lifecycle.

### Common use cases

- Chat applications.
- Multiplayer games.
- Collaborative editing.
- Live location tracking.
- Real-time operational dashboards.
- Interactive trading applications.

---

## 6. Comparing Polling, Long Polling, SSE, and WebSockets

| Feature | Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| Communication | Request-response | Request-response held open | Server → Client | Bidirectional |
| Persistent connection | No | Temporarily | Yes | Yes |
| Real-time capability | Limited by interval | Near real-time | Yes | Yes |
| Client-to-server messages | HTTP requests | HTTP requests | Separate channel | Same connection |
| Reconnection | Client-managed | Client-managed | Browser support | Application-managed |
| Complexity | Low | Medium | Medium | Higher |
| Typical use | Periodic refresh | Legacy real-time updates | Notifications and feeds | Interactive real-time apps |

### Which should you choose?

- Polling: simple updates every several seconds or minutes.
- Long polling: near-real-time updates when a persistent streaming connection is unavailable.
- SSE: continuous server-to-client updates.
- WebSockets: frequent, interactive communication in both directions.

Don't choose WebSockets just because an application is described as real-time. Choose based on the communication requirements.

---

## 7. WebSocket Architecture at Scale

A WebSocket connection remains open.

Suppose your application has:

- 100,000 connected users.
- 10,000 location updates per second.
- Multiple WebSocket servers.

A single application server may not be able to manage all connections.

We need a distributed architecture.

### Basic architecture

```text
              Clients
                 |
                 ▼
           Load Balancer
                 |
       ┌─────────┼─────────┐
       |         |         |
       ▼         ▼         ▼
    WS Server WS Server WS Server
       |         |         |
       └─────────┼─────────┘
                 |
                 ▼
             Pub/Sub
                 |
                 ▼
           Event Producers
```

Each WebSocket server manages a subset of connected clients.

The Pub/Sub layer distributes events between servers.

### Why do we need Pub/Sub?

Imagine:

- Passenger A is connected to WebSocket Server 1.
- Driver B is connected to WebSocket Server 2.
- Driver B sends a location update.

Server 2 receives the update, but Passenger A is connected to Server 1.

Without a cross-server messaging mechanism, Server 1 may never receive the event.

Pub/Sub solves this by distributing relevant events across the WebSocket infrastructure.

Possible technologies:

- Redis Pub/Sub.
- Redis Streams.
- Apache Kafka.
- NATS.
- PubNub.
- Cloud-managed messaging services.

The correct choice depends on durability, ordering, throughput, delivery guarantees, and operational requirements.

---

## 8. Connection Management

Persistent connections introduce operational challenges.

### 8.1 Heartbeats

A connection may appear open even though the client is no longer reachable.

Heartbeat mechanisms help detect dead connections.

WebSocket protocol-level Ping/Pong frames can be used for this purpose.

Example:

```text
Server ─── Ping ───> Client
Server <── Pong ──── Client
```

If the client does not respond within the configured timeout, the server can close the connection and clean up associated state.

### 8.2 Reconnection

Networks can fail temporarily.

Clients should reconnect using:

- Exponential backoff.
- Random jitter.
- Maximum retry intervals.
- Connection-state handling.

Avoid having every client reconnect simultaneously after an outage.

That can create a reconnection storm.

### 8.3 Connection authentication

A WebSocket connection should be authenticated during connection establishment or through a secure application protocol.

Also consider:

- Token expiration.
- Permission changes.
- Tenant isolation.
- Connection revocation.
- Message-level authorization.

A successful handshake does not mean every subsequent message is authorized.

### 8.4 Sticky sessions

Suppose a client connects to WebSocket Server A.

A later reconnect may reach Server B.

This is normally acceptable if the application can restore the client's subscriptions and session state.

Sticky sessions can help with certain architectures, but they should not be used as a substitute for distributed state management.

---

## 9. Broadcasting vs. Targeted Messaging

Not every event should be sent to every connected client.

Consider a ride-booking application.

A driver location update should generally reach:

- Dispatchers monitoring the relevant service area.
- The passenger assigned to that driver.
- Other authorized operational users.

It should not reach every user in the application.

### Broadcasting

Send an event to every client in a group.

Example:

```text
service-area-101
      |
      ├── Dispatcher A
      ├── Dispatcher B
      └── Dispatcher C
```

### Targeted messaging

Send an event to a particular user, device, ride, or subscription.

Example:

```text
ride-501
   |
   └── Passenger 101
```

### Topic-based messaging

Clients subscribe to topics or channels.

Examples:

```text
tenant-10:service-area-101
ride-501
driver-101
```

Topics should be designed with authorization and data-isolation requirements in mind.

Never assume that a client is authorized simply because it knows a channel name.

---

## 10. Designing Live Driver Tracking

Let's design a live tracking system for a ride-booking application.

### Requirements

- 100,000 active drivers.
- Each driver sends one location update every five seconds.
- Dispatchers monitor drivers on a live map.
- Passengers monitor their assigned driver.
- Location updates should appear within a few seconds.
- Historical location data may be stored separately.

### Step 1: Estimate incoming traffic

Drivers send one update every five seconds.

\[
Updates/sec = \frac{100,000}{5}
\]

\[
= 20,000 \ updates/sec
\]

The system must handle approximately 20,000 incoming location updates per second while all 100,000 drivers are active.

This is an average under the stated assumptions. Real traffic may be uneven and bursty.

### Step 2: High-level architecture

```text
              Driver Devices
                    |
                    ▼
             Location Gateway
                    |
                    ▼
             Event Processing
                    |
             ┌──────┴────────┐
             |               |
             ▼               ▼
        Latest Location   Location Stream
          Store             / Queue
             |               |
             ▼               ▼
       Dispatch Query    Historical Storage
             |
             ▼
        Realtime Pub/Sub
             |
       ┌─────┴──────┐
       |            |
       ▼            ▼
  WebSocket     WebSocket
   Server A      Server B
       |            |
       ▼            ▼
 Dispatchers    Passengers
```

### Step 3: Separate latest location from history

The latest driver location and historical location records have different access patterns.

**Latest location**

We need fast access to the most recent position of a driver.

Possible storage:

- Redis.
- PostgreSQL with suitable indexes.
- A dedicated geospatial or key-value store.

Example:

```text
driver:101:latest
{
  latitude: 28.61,
  longitude: 77.23,
  timestamp: 1793360000
}
```

**Historical locations**

We may need to store every accepted location update for:

- Trip reconstruction.
- Analytics.
- Dispute investigation.
- Operational reporting.

This is a write-heavy workload and may require partitioning, retention policies, or specialized time-series storage.

Do not automatically store every location update in the same table used for live map queries.

### Step 4: Publish updates

When a driver location arrives:

1. Authenticate the driver.
2. Validate the payload.
3. Check its timestamp and sequence/version.
4. Update the latest location.
5. Publish an event to the appropriate topic.
6. Send the update to authorized subscribers.
7. Persist historical data asynchronously if required.

### Step 5: Coalesce updates

Suppose a driver sends 10 location updates in one second.

A dispatcher map may not need to render all 10 updates.

The system can coalesce updates and send only the latest location at a controlled frequency.

For example:

```text
Incoming updates:
28.61, 77.23
28.62, 77.24
28.63, 77.25
28.64, 77.26

Published update:
28.64, 77.26
```

This reduces downstream message volume and frontend rendering work.

However, don't apply the same coalescing policy to historical data if every accepted location is required for audit or reconstruction.

---

## 11. Ordering and Stale Location Updates

Distributed systems do not always process messages in the order they were generated.

Example:

```text
Update A: timestamp 10:00:01
Update B: timestamp 10:00:02
```

Network delays may cause B to arrive before A.

If the application blindly stores every incoming update, an older location can overwrite a newer one.

### Solution: sequence numbers or versions

Each driver update can include:

```json
{
  "driverId": 101,
  "sequence": 502,
  "latitude": 28.61,
  "longitude": 77.23,
  "timestamp": "2026-10-30T10:00:02Z"
}
```

The system compares the incoming sequence with the stored sequence.

Only a newer update is allowed to replace the latest state.

This helps prevent out-of-order updates from moving a driver backwards in time.

Timestamps alone require care because device clocks may be inaccurate.

### Partitioning by driver ID

A message broker can partition location events by driver ID.

This helps preserve the order of events for a given driver within a partition, depending on the broker's guarantees.

It does not automatically establish global ordering across all drivers.

---

## 12. Backpressure and Slow Clients

Imagine one dispatcher browser cannot process incoming events quickly enough.

The WebSocket server continues receiving messages, but the client processes them slowly.

This can create:

- Growing buffers.
- Increased memory usage.
- High CPU usage.
- Increasing latency.
- Eventually dropped connections.

### Possible solutions

1. Coalesce frequent state updates.
2. Limit per-client outbound queues.
3. Drop obsolete updates where business rules allow.
4. Reduce update frequency for slow clients.
5. Disconnect clients that exceed resource limits.
6. Allow clients to request a fresh snapshot after reconnecting.

For live driver positions, dropping an older queued location in favor of a newer one may be reasonable.

For chat messages or financial events, dropping messages is generally not acceptable.

**Backpressure policy must match the meaning of the data.**

---

## 13. Security in Real-Time Systems

Real-time connections can expose sensitive information if authorization is weak.

Consider these questions:

- Can a passenger subscribe to another passenger's ride?
- Can one tenant receive another tenant's driver updates?
- Can an unauthenticated client publish driver locations?
- What happens when a driver's permissions change?
- How are disconnected subscriptions cleaned up?

### Recommended protections

- Authenticate connections.
- Authorize every subscription.
- Validate every incoming message.
- Scope topics by tenant and service area where appropriate.
- Apply per-user and per-connection rate limits.
- Enforce message size limits.
- Revalidate authorization when necessary.
- Avoid exposing internal identifiers unnecessarily.

Authorization must be enforced on the server, not just in frontend subscription logic.

---

## 14. Choosing the Right Technology

| Requirement | Suitable approach |
|---|---|
| Refresh dashboard every 30 seconds | Polling |
| Receive updates over HTTP without WebSockets | Long polling |
| Server sends notifications to browser | SSE |
| Chat application | WebSockets |
| Live driver tracking | WebSockets or managed real-time messaging |
| Financial event processing | Durable message broker |
| High-volume event history | Kafka or another suitable streaming platform |
| Managed real-time channels | PubNub or similar services |

These technologies are not interchangeable in every respect.

For example, WebSockets provide a connection protocol, while Kafka provides durable event streaming. They can be used together in one architecture.

A WebSocket server might consume events from Kafka and forward selected messages to connected browsers.

---

## 15. Common Mistakes

1. Using polling at very short intervals without measuring the load.
2. Choosing WebSockets for every real-time requirement.
3. Assuming a persistent connection guarantees message delivery.
4. Ignoring reconnect storms.
5. Sending every event to every connected client.
6. Keeping all connection state only in one application instance.
7. Ignoring out-of-order events.
8. Not applying backpressure to slow clients.
9. Treating latest location and historical location as the same workload.
10. Forgetting subscription authorization.
11. Assuming sticky sessions solve distributed Pub/Sub.
12. Ignoring connection limits and memory per connection.

---

## 16. Practical Checklist

### Communication
- [ ] Choose polling, long polling, SSE, or WebSockets based on requirements.
- [ ] Define update frequency and acceptable latency.
- [ ] Define message format and event contracts.

### Scaling
- [ ] Estimate concurrent connections.
- [ ] Estimate incoming and outgoing messages per second.
- [ ] Use distributed event delivery where necessary.
- [ ] Monitor connection count and memory per connection.
- [ ] Define reconnection and heartbeat behavior.

### Reliability
- [ ] Handle duplicate and out-of-order messages.
- [ ] Define backpressure policies.
- [ ] Recover client subscriptions after reconnect.
- [ ] Provide snapshot synchronization where necessary.
- [ ] Distinguish durable events from replaceable state updates.

### Security
- [ ] Authenticate connections.
- [ ] Authorize subscriptions.
- [ ] Validate incoming messages.
- [ ] Enforce tenant isolation.
- [ ] Apply connection and message limits.

### Observability
- [ ] Monitor active connections.
- [ ] Monitor messages per second.
- [ ] Monitor delivery latency.
- [ ] Track reconnect and disconnect rates.
- [ ] Track dropped messages and queue sizes.

---

## 17. Revision Questions

1. What is polling?
2. How is long polling different from normal polling?
3. What is the difference between SSE and WebSockets?
4. What does full-duplex communication mean?
5. Why do large WebSocket systems need Pub/Sub?
6. What are WebSocket heartbeats?
7. Why can reconnect storms occur?
8. What are sticky sessions, and why are they not a complete solution?
9. Why should location updates be partitioned by driver ID?
10. How do sequence numbers prevent stale updates?
11. What is backpressure?
12. Why should historical location and latest location be stored separately?
13. When is it safe to coalesce real-time events?
14. How do you prevent unauthorized clients from subscribing to private channels?

---

## 18. Practice Exercise — Design a Real-Time Tracking System

Design a real-time tracking architecture with:

- 50,000 active drivers.
- One location update every five seconds.
- 1,000 dispatcher sessions.
- Each dispatcher monitors up to 500 drivers.
- Passengers can monitor their assigned driver.
- Location updates should be delivered within two seconds under normal conditions.
- The system must support reconnecting clients.

Answer these questions:

1. Calculate incoming location updates per second.
2. Would you choose WebSockets, SSE, or polling for dispatchers? Why?
3. How would you distribute updates across multiple real-time servers?
4. How would you ensure a dispatcher receives only authorized driver updates?
5. How would you prevent older location events from overwriting newer ones?
6. Would you store every location update or only the latest location?
7. How would you handle a dispatcher browser that cannot process messages fast enough?
8. What happens when a WebSocket server crashes?
9. How would clients recover missed updates after reconnecting?
10. Which metrics would you monitor?

### Bonus interview question

"Your system supports 100,000 WebSocket connections. After a deployment, all clients reconnect within a short period, and the backend becomes overloaded. How would you prevent this from happening?"

Think about exponential backoff, jitter, connection admission control, capacity headroom, and gradual deployments.

**Key takeaway:** Real-time architecture is not simply about choosing WebSockets. It is about managing connections, distributing events, maintaining ordering, controlling traffic, handling failures, and delivering only the information each client needs.

### Practice focus

For this lesson, pay special attention to:
- Why WebSocket servers need distributed Pub/Sub.
- Why latest location and historical location should be treated differently.
- How to handle stale and out-of-order events.
- How to recover after a WebSocket server failure.
- How to prevent reconnect storms.
These concepts connect directly to the real-time architecture used in large dispatch and tracking applications.