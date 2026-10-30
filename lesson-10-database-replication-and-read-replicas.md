# Lesson 10: Database Replication and Read Replicas

## 1. Why Do We Need Database Replication?

Imagine a ride-booking platform with millions of passengers and drivers.

PostgreSQL stores:
- Passenger and driver information.
- Ride requests and assignments.
- Ride status transitions.
- Payment records.
- Driver and ride history.

As the platform grows, the database may become a bottleneck.

For example:
- Thousands of users query their ride history.
- Drivers and passengers frequently request ride information.
- Reporting queries consume database resources.
- The database must also process writes, such as new bookings and status changes.

Adding more application servers does not necessarily solve this problem. Every application server may still send queries to the same database.

**Database replication** creates and maintains copies of data on other database servers.

Replication can improve read capacity, provide additional copies for recovery, and support certain availability strategies.

However, replication adds complexity and does not automatically make every query faster or every write safer.

---

## 2. What Is Database Replication?

Database replication is the process of copying database changes from one database server to another.

A common PostgreSQL architecture is:

- **Primary:** Accepts writes and usually handles authoritative reads.
- **Standby or replica:** Receives changes from the primary and maintains a copy of its data.

```text
             Application
                  |
          +-------+-------+
          |               |
          v               v
     Write request    Read request
          |               |
          v               v
       Primary          Replica
          |               ^
          |               |
          +-- Changes ----+
             replicated
```

This is a simplified diagram. The application must deliberately route reads and writes, and the replication mechanism determines how changes are transferred.

### Example

A passenger creates a ride.

1. The application writes the ride to the primary database.
2. The primary commits the transaction.
3. The database sends the corresponding changes to its standby.
4. The standby replays those changes.
5. A read query on the standby can return the ride after the relevant changes have been applied.

The important detail is that step 4 might happen after step 2.

That delay creates a central trade-off: **a replica can be available for reads while still being behind the primary.**

---

## 3. Why Not Use Replication for Everything?

Replication helps with particular problems, but it is not a universal scaling solution.

| Problem | Can replication help? |
|---|---|
| Large volume of read queries | Yes, if reads can be distributed to replicas |
| Need an additional copy of data | Yes |
| Need another server for recovery | Often, depending on the setup |
| Slow SQL query | Not necessarily; the query may need optimization |
| Heavy write workload | Usually not by itself; replicas still need to apply changes |
| Excessive database connections | Not automatically; connection routing and pool limits still matter |
| Conflicting ride assignments | Replication alone does not solve concurrency control |
| Cross-region latency | Possibly, but topology and routing determine the result |

If a query is inefficient, copying the database does not make that query inherently efficient. A replica may simply perform the same expensive work.

---

## 4. Primary-Replica Architecture

A primary-replica architecture separates write ownership from read capacity.

```text
                 Application
                 /          \
                v            v
          Write queries   Read queries
                |            |
                v            v
             Primary      Read Replica
                |
                +-----------> Replica
                   Replication
```

The primary handles authoritative writes. Read replicas handle selected reads.

### Which queries should go to replicas?

Good candidates include:
- Historical ride lists.
- Read-heavy dashboards.
- Analytics queries that tolerate some staleness.
- Large reports that would otherwise compete with transactional reads.
- Search or lookup operations whose consistency requirements permit replica lag.

Queries that often need the primary include:
- Checking the latest committed state immediately after a write.
- Validating whether a driver can be assigned to a ride.
- Reading state that must be current for a business decision.
- Operations that depend on a transaction's newly written data.

The exact routing depends on the required consistency guarantees, not merely whether a query is a `SELECT`.

**Important:** A read query can be business-critical. Reads are not automatically safe to send to a replica.

---

## 5. Synchronous vs. Asynchronous Replication

Replication can use different acknowledgement and consistency models.

In PostgreSQL, physical streaming replication commonly sends write-ahead log (WAL) records from the primary to a standby.

Two important concepts are synchronous and asynchronous replication.

### Asynchronous replication

The primary does not need to wait for the standby to confirm receipt or application of each transaction before reporting a commit as successful.

Simplified flow:

```text
Application
    |
    v
Primary commits transaction
    |
    +------> Success returned
    |
    v
WAL changes sent to standby
    |
    v
Standby replays changes
```

Advantages:
- Lower write latency in many configurations.
- Primary writes are less dependent on replica responsiveness.
- Suitable for many read-scaling and reporting workloads.

Trade-offs:
- Replicas may lag.
- A replica may not contain the latest committed changes.
- During a primary failure, recently acknowledged transactions may be absent from an asynchronously promoted standby, depending on what was received and durably preserved.

### Synchronous replication

In a synchronous configuration, commit acknowledgement waits for confirmation from one or more designated synchronous standbys.

The exact guarantee depends on the configured acknowledgement level.

For example, confirming that a standby has received and durably flushed WAL is not the same as confirming that the standby has replayed the changes and can immediately serve a query with those changes.

Advantages:
- Can reduce the risk of losing acknowledged transactions during certain failover scenarios.
- Provides stronger guarantees about replication progress at commit time.

Trade-offs:
- Higher write latency.
- Commit latency depends on standby acknowledgement and network conditions.
- If the required synchronous standby is unavailable, commits may block or fail, depending on the configuration and operational policy.

Synchronous replication does not eliminate every data-loss scenario. Failover policy, durable storage, acknowledgement settings, and which standby is promoted all matter.

### Choosing a model

| Requirement | Typical consideration |
|---|---|
| Low-latency writes and scalable reads | Asynchronous replication may be appropriate |
| Stronger protection against losing acknowledged commits | Synchronous replication may be appropriate |
| Geographically distant replicas | Network latency may make synchronous acknowledgement expensive |
| Read-your-writes behavior | Routing and consistency strategy are still required |
| High availability | Replication must be combined with monitoring and failover procedures |

---

## 6. Replication Lag

Replication lag is the delay between changes becoming available on the primary and those changes being received or applied on a replica.

Different monitoring measurements can describe different stages of this delay.

Consider this timeline:

```text
10:00:00.000  Ride committed on primary
10:00:00.100  WAL received by replica
10:00:00.450  Change replayed on replica
```

A query against the replica at 10:00:00.200 might not yet see the ride.

The exact lag varies with network conditions, write volume, replica resources, and workload.

### Why does lag happen?

Common causes include:
- High write throughput.
- Slow network communication.
- Replica CPU or disk limitations.
- Expensive queries competing with replay.
- Large transactions.
- Delayed replay by configuration.
- Recovery or maintenance activity.

### How can lag affect users?

Suppose a passenger creates a ride and immediately opens ride history.

1. The ride commits on the primary.
2. The application routes the next history query to a replica.
3. The replica has not replayed the new ride.
4. The passenger sees an incomplete list.

The write succeeded, but the user does not immediately see it.

This is a read-after-write consistency problem.

---

## 7. Read-After-Write Consistency

Read-after-write consistency means a client can observe its own successful write when it subsequently reads the relevant data.

A simple primary-replica architecture does not automatically provide this guarantee when subsequent reads can go to a lagging replica.

### Strategy 1: Route reads to the primary temporarily

After a write, direct relevant reads to the primary for a bounded period or until the application can establish the required consistency condition.

Advantages:
- Straightforward conceptually.
- Avoids replica lag for those reads.

Trade-offs:
- Adds primary read load.
- Requires routing logic.
- A fixed time window does not mathematically guarantee that a replica has caught up.

### Strategy 2: Use a consistency token or replication position

The application can track a relevant commit or replication position and ensure a selected replica has replayed at least that position before using it.

This requires a carefully implemented mechanism that matches the database's replication guarantees.

Advantages:
- Can allow reads from replicas after the relevant change is available.
- Can reduce unnecessary reads against the primary.

Trade-offs:
- More complicated routing and waiting behavior.
- Not every replica-routing framework exposes the required controls.
- The system must define what happens if a replica cannot catch up in time.

### Strategy 3: Return the committed result

After creating a ride, the API can return the committed ride representation rather than immediately re-reading it from a replica.

Advantages:
- Avoids an unnecessary read.
- Provides the client with the result of the write operation.

Trade-offs:
- Does not solve every later read.
- The client still needs a strategy for subsequent operations that require fresh state.

### Strategy 4: Read critical state from the primary

For business decisions that require current authoritative state, query the primary or use an equivalent consistency mechanism.

This is often the safest starting point for operations such as validating ride assignment.

---

## 8. Scaling Reads With Multiple Replicas

One replica may not provide enough read capacity.

The architecture can be extended:

```text
                     Application
                          |
                     Read Router
                    /     |     \
                   v      v      v
               Replica 1 Replica 2 Replica 3
                    ^      ^      ^
                    |      |      |
                    +------+------+
                           |
                         Primary
                           |
                        Writes
```

The read router can distribute eligible reads among replicas.

However, replica selection should consider more than round-robin distribution.

Potential factors include:
- Replica health.
- Replication lag.
- Available connection capacity.
- Query workload.
- Geographic latency.
- Consistency requirements.

### A critical limitation: write amplification

Every replica must process the changes it receives.

Adding more replicas increases read-serving capacity only if the system can sustain the additional replication work and infrastructure overhead.

If the primary's write workload or the replication network becomes the bottleneck, adding replicas may not solve the underlying problem.

---

## 9. Replication Is Not Sharding

These concepts are frequently confused.

### Replication

Replication creates copies of the same logical dataset.

```text
Primary:   Rides 1–10,000
Replica A: Rides 1–10,000
Replica B: Rides 1–10,000
```

The purpose may be read scaling, recovery, or availability.

### Sharding

Sharding divides a dataset across multiple database nodes.

```text
Shard A: Rides 1–3,000
Shard B: Rides 3,001–6,000
Shard C: Rides 6,001–10,000
```

The actual distribution is determined by the shard key or partitioning strategy.

Sharding can distribute storage and write load, but it adds complexity involving routing, rebalancing, cross-shard queries, and transactions.

### Comparison

| Property | Replication | Sharding |
|---|---|---|
| Data distribution | Copies of the same dataset | Different portions of the dataset |
| Primary goal | Read capacity, recovery, availability | Distribute data and workload |
| Read scaling | Often helps | Can help by distributing queries |
| Write scaling | Usually limited by the primary architecture | Can help if writes distribute across shards |
| Operational complexity | Moderate, depending on setup | Often higher |
| Cross-node queries | May need replica routing | May require coordination across shards |

A system may use both replication and sharding: each shard can have its own primary and replicas.

---

## 10. Replication and Failover

Replication is often used as part of a high-availability architecture.

Suppose the primary database fails.

A failover process may:
1. Detect that the primary is unavailable.
2. Determine which standby is eligible for promotion.
3. Promote the selected standby.
4. Redirect database connections or update routing.
5. Allow application traffic to resume.
6. Re-establish replication from the new primary.

The exact steps depend on the database platform and orchestration tooling.

### Failover is not instantaneous

Potential problems include:
- Detection delays.
- Connection failures during the transition.
- Stale connections in application pools.
- A standby that is behind the old primary.
- Split-brain risk if two nodes accept conflicting writes.
- A failback process that is not safe or well tested.

### Split brain

Split brain occurs when two database nodes both behave as authoritative writers for the same dataset without proper coordination.

This can lead to divergent histories and conflicting updates.

A production design needs a safe leader-election or promotion mechanism, fencing or equivalent protection, and clear rules for which node is allowed to accept writes.

### Recovery objectives

Two useful measures are:

**RTO — Recovery Time Objective:** How long the system can take to recover after a disruption.

**RPO — Recovery Point Objective:** How much data loss, measured in time or data exposure, the system can tolerate.

For example, an RTO of two minutes means the recovery process should restore service within the agreed two-minute target.

An RPO of zero means the design aims to avoid losing committed data within the defined failure model. Achieving this requires careful guarantees; simply having a replica does not establish it.

---

## 11. PostgreSQL Replication: What to Monitor

For PostgreSQL, useful signals include:
- Primary and standby availability.
- WAL generation and transfer rates.
- Standby receive and replay progress.
- Replication lag.
- Database CPU and disk I/O.
- Active connections and pool saturation.
- Long-running transactions.
- Lock waits.
- Query latency and error rates.
- Failover events and recovery duration.

PostgreSQL exposes replication-related information through system views such as `pg_stat_replication` on the primary and recovery-related statistics on standbys. Exact columns and availability depend on PostgreSQL version and role.

Do not rely on a single lag metric. For example, time since a recent transaction may be misleading during periods when no changes are being generated.

Monitor the underlying replication positions and the user-facing freshness objective where possible.

---

## 12. Common Mistakes

1. Sending every read to a replica without considering consistency.
2. Assuming a successful primary commit means all replicas already contain the change.
3. Adding replicas to solve a slow query without examining its execution plan.
4. Assuming synchronous replication means every standby has replayed every transaction before the client receives success.
5. Ignoring the write and storage overhead of maintaining replicas.
6. Treating replication as equivalent to sharding.
7. Assuming failover is automatic simply because a standby exists.
8. Failing to plan for stale connections during promotion.
9. Ignoring split-brain risks.
10. Assuming a replica replaces backups and point-in-time recovery.
11. Using replicas for authoritative business decisions without a consistency strategy.
12. Measuring only infrastructure health while ignoring user-visible stale reads.

---

## 13. Practical Decision Framework

When deciding whether to add read replicas, ask:

### Step 1: Identify the bottleneck
Is the primary overloaded by read queries, writes, locks, or inefficient SQL?

### Step 2: Optimize first
Check indexes, query plans, unnecessary reads, and connection-pool behavior.

### Step 3: Classify read consistency
Which queries tolerate stale data? Which require read-after-write consistency or authoritative state?

### Step 4: Estimate replica capacity
How much read traffic can each replica serve while maintaining acceptable lag?

### Step 5: Design routing
Decide which queries go to the primary and which can use replicas.

### Step 6: Define failure behavior
What happens when a replica becomes unhealthy or falls behind?

### Step 7: Define recovery objectives
Set acceptable recovery time and data-loss exposure for the failure scenarios that matter.

### Step 8: Test the architecture
Simulate replica lag, primary failure, connection disruption, and recovery before depending on the design in production.

---

## 14. Worked Example: Ride History

Suppose a ride-booking platform has:
- 20,000 read queries per second at peak.
- 2,000 write transactions per second.
- Many users reading historical rides.
- A requirement that ride history usually reflects newly created rides immediately after creation.

These are illustrative assumptions.

### Initial investigation

Before adding replicas, inspect query plans, indexing, connection saturation, and the ratio of repeated reads to writes.

### Architecture decision

If historical reads dominate primary load, introduce one or more read replicas for queries that tolerate bounded staleness.

Keep critical operations, such as validating the current state of a ride assignment, on the primary or another suitable authoritative consistency path.

For a passenger who just created a ride, return the committed ride representation or route the relevant read to the primary until read-after-write requirements are satisfied.

### Operational requirements

Monitor replica lag and stop routing freshness-sensitive reads to a replica that cannot meet the required consistency condition.

Test failover and ensure the application can reconnect safely after a primary promotion.

### Result

The architecture can serve more reads while preserving the correctness of critical operations.

The improvement is not guaranteed merely by adding a replica; it must be validated under representative traffic and failure conditions.

---

## 15. Revision Questions

1. What is database replication?
2. What is the difference between a primary and a read replica?
3. How do synchronous and asynchronous replication differ?
4. What is replication lag, and how can it affect users?
5. What is read-after-write consistency?
6. How would you prevent a user from missing a newly created ride when reading from a replica?
7. Why does adding replicas not necessarily increase write capacity?
8. What is the difference between replication and sharding?
9. What is split brain, and why is it dangerous?
10. What do RTO and RPO mean?
11. Why are replicas not a complete substitute for backups?
12. Which queries from a ride-booking application are suitable for replicas, and which require stronger consistency?
13. What would you monitor when introducing PostgreSQL read replicas?
14. How would you design failover for a primary database?
15. What would you investigate before deciding that a database needs more replicas?

## Final Takeaway

Database replication creates additional copies of data to support read scaling, recovery, and availability.

The key concepts to remember are:

- A primary typically owns authoritative writes.
- Replicas can lag behind the primary.
- Asynchronous replication often favors write latency over immediate replica freshness.
- Synchronous replication provides stronger acknowledgement guarantees but introduces latency and availability trade-offs.
- Read routing must respect business consistency requirements.
- Replication is different from sharding.
- Failover requires explicit planning, safe promotion, and recovery testing.

**The central design rule:** Use replicas to distribute eligible reads and improve recovery options, but never assume a replica contains the latest committed state unless your architecture provides that guarantee.

## Practice exercise

Imagine your ride-booking platform uses PostgreSQL with one primary and two asynchronous read replicas.

1. A passenger creates a ride, then immediately checks ride history. The new ride is missing. What caused this, and how would you fix it?
2. The primary is overloaded by historical ride queries. What would you change, and which queries would you keep on the primary?
3. If the primary fails, what must happen before a standby can safely take over?