# Lesson 14 — CAP Theorem, Consistency, and Availability in Distributed Systems

This lesson is one of the most important topics in system design interviews.

We'll understand what happens when distributed systems experience network failures, why consistency and availability involve trade-offs, and how systems such as ride-booking platforms handle those trade-offs in practice.

The key idea: A distributed system cannot guarantee every desirable property under every failure condition. Good system design means choosing the right guarantees for each operation.

## 1. Why Do We Need the CAP Theorem?

Imagine a ride-booking application running across two data centers.

- Data Center A stores a copy of the database.
- Data Center B stores another copy.
- Both locations serve application requests.
- Database updates are replicated between them.

Initially, everything works correctly.

Now imagine the network connection between A and B fails.

Both data centers are still running, but they cannot communicate with each other.

A passenger sends a request to Data Center A to cancel a ride.

At almost the same time, a dispatcher sends a request to Data Center B to assign a driver to that ride.

The two data centers cannot coordinate their decisions.

This creates a fundamental question:

**Should both data centers continue processing requests, or should one refuse requests until coordination is restored?**

This is the kind of trade-off explained by the CAP theorem.

---

## 2. The Three Properties of CAP

CAP stands for:

- Consistency
- Availability
- Partition Tolerance

### 2.1 Consistency (C)

In CAP, consistency means that every successful read receives the most recent successful write, or an error.

This is often called linearizable consistency.

Example:

1. A ride has status `requested`.
2. A dispatcher changes its status to `assigned`.
3. The update succeeds.
4. A subsequent read must return `assigned`, not the older `requested` value.

In a distributed system, this guarantee requires coordination when multiple replicas are involved.

**Important:** CAP consistency is not the same as database ACID consistency. They describe different properties.

### 2.2 Availability (A)

Availability means that every request received by a non-failed node receives a non-error response.

The response does not necessarily contain the newest data.

Example:

1. Data Center A updates a ride to `assigned`.
2. Data Center B cannot communicate with A.
3. B receives a request for that ride.
4. B responds with its old value: `requested`.

The request succeeded, but the result is stale.

That is an availability-oriented response that does not provide strong consistency.

### 2.3 Partition Tolerance (P)

A network partition happens when communication between parts of a distributed system is interrupted.

For example:

<pre>
       Data Center A                 Data Center B
      ┌─────────────┐               ┌─────────────┐
      │ Database A  │       X       │ Database B  │
      │             │   Network     │             │
      │ Application │   Failure     │ Application │
      └─────────────┘               └─────────────┘
</pre>

Both data centers may still be operational, but they cannot exchange information.

Partition tolerance means the system is designed to continue operating despite this communication failure, subject to its chosen consistency and availability guarantees.

---

## 3. The CAP Trade-off

When a network partition occurs, a distributed data system cannot guarantee both:

- Strong consistency for every operation.
- Availability for every request.

It must make a trade-off.

### CP — Consistency and Partition Tolerance

A CP-oriented system prioritizes consistency during a partition.

If a node cannot safely coordinate with the other nodes, it may reject or delay requests.

Example:

- A ride assignment requires a coordinated decision.
- A replica cannot confirm that the ride is still assignable.
- The system rejects or delays the assignment rather than risk conflicting assignments.

<pre>
Request
   │
   ▼
Can safely coordinate?
   │
   ├── Yes ──> Process request
   │
   └── No  ──> Reject or delay request
</pre>

**Advantage:** Protects correctness.

**Trade-off:** Some requests may fail or wait during a partition.

### AP — Availability and Partition Tolerance

An AP-oriented system prioritizes responding to requests during a partition.

A node may accept an update even if it cannot coordinate with other replicas.

Example:

- A driver's location is updated in Data Center A.
- Data Center B cannot communicate with A.
- B continues serving location requests using its local data.
- The returned location might be stale.

After communication is restored, replicas can synchronize according to the database's conflict-resolution and replication rules.

**Advantage:** More requests can receive responses during a partition.

**Trade-off:** Reads may be stale, and concurrent writes may need conflict resolution.

### CA — Consistency and Availability

A CA system can provide consistency and availability when there is no network partition.

However, it cannot guarantee both when a partition occurs.

A single-node relational database is sometimes used as a simplified CA example because it has no replica-to-replica partition to tolerate. But a real distributed system must account for network partitions.

**Important:** CA is not a practical third option for a distributed database that must tolerate network partitions.

### Comparison

| Model | During a partition | Main trade-off |
|---|---|---|
| CP | May reject or delay some requests | Consistency over availability |
| AP | May return stale or conflicting data | Availability over immediate consistency |
| CA | Cannot guarantee both through a partition | Does not tolerate distributed partitions |

---

## 4. CAP Is About Network Partitions

A common interview mistake is saying:

"PostgreSQL is CP and MongoDB is AP."

This is an oversimplification.

CAP describes behavior during a network partition, not a permanent label that fully describes a database.

Real databases have different configurations and operations. They may provide different guarantees depending on:

- Replication mode.
- Read consistency settings.
- Write acknowledgement requirements.
- Quorum configuration.
- Failover behavior.
- Whether a node can coordinate with a majority.

A database may also provide stronger guarantees for one operation and weaker guarantees for another.

**Interview answer:** During a partition, a distributed system must choose whether to reject or delay operations that cannot be completed consistently, or to continue responding with potentially stale or conflicting data.

---

## 5. Understanding Quorum

Quorum is a mechanism used by some distributed systems to coordinate reads and writes across replicas.

Suppose we have five replicas:

<pre>
Replica 1
Replica 2
Replica 3
Replica 4
Replica 5
</pre>

A majority quorum is:

\[
Q = \lfloor N/2 \rfloor + 1
\]

Where:

- N = total number of replicas.
- Q = majority quorum.

For five replicas:

\[
Q = \lfloor 5/2 \rfloor + 1 = 3
\]

So three replicas form a majority.

### Read and write quorum

Some quorum-based systems use read quorum R and write quorum W.

A common simplified condition for overlapping read and write sets is:

\[
R + W > N
\]

Example:

- N = 5
- W = 3
- R = 3

Then:

\[
R + W = 6 > 5
\]

This means the read and write replica sets must overlap, under the assumed quorum model.

However, quorum overlap alone does not guarantee full linearizability. The system's versioning, coordination, conflict resolution, and read/write protocols also matter.

### What happens if a replica fails?

With five replicas and a quorum of three:

- Five available: quorum is available.
- Four available: quorum is available.
- Three available: quorum is still available.
- Two available: majority quorum is unavailable.

This is why quorum-based systems can tolerate some node failures while maintaining their configured guarantees.

---

## 6. Consistency Models

CAP is not the only consistency concept.

Different systems offer different consistency models depending on their architecture and configuration.

### 6.1 Strong consistency

A successful write is immediately visible to subsequent reads according to the consistency guarantee.

Example:

A dispatcher assigns ride 501 to driver 101. A subsequent read must not return the old unassigned state.

Useful for:

- Ride assignments.
- Financial transactions.
- Inventory reservations.
- Uniqueness-sensitive operations.

### 6.2 Eventual consistency

Replicas may temporarily return different values, but if no new updates occur and replication succeeds, they eventually converge.

Example:

A driver changes location.

- Replica A has the new location.
- Replica B still has the previous location.
- After replication catches up, both hold the updated location.

Useful for:

- Driver location.
- Social activity feeds.
- Analytics updates.
- Non-critical cached data.

Eventual consistency does not specify exactly how long convergence takes. If the application needs a freshness bound, that must be designed and monitored separately.

### 6.3 Read-your-writes consistency

After a client successfully writes a value, that client should see its own write in subsequent reads.

Example:

1. A passenger updates their profile name.
2. The update succeeds.
3. The passenger opens the profile again.
4. The application should show the new name, even if a replica is lagging.

Possible implementation approaches include:

- Read from the primary after a write.
- Route subsequent reads to the same session-consistent replica.
- Use a version or session token to enforce the required read behavior.

### 6.4 Monotonic reads

Once a client has seen a particular version of data, it should not later see an older version.

Example:

1. A dispatcher sees ride status `assigned`.
2. The next request should not show the earlier `requested` status.

This can matter when reads are routed across replicas with different replication lag.

### 6.5 Bounded staleness

A system permits stale reads but limits how stale the data can be.

Example:

A tracking dashboard can tolerate a driver location that is at most three seconds old.

The system should monitor location timestamps and define what to display when the freshness target is exceeded.

---

## 7. Strong Consistency vs. Eventual Consistency in a Ride Platform

Not all data requires the same consistency guarantee.

| Data | Suggested consistency | Reason |
|---|---|---|
| Ride assignment | Strong transactional coordination | Avoid conflicting assignments |
| Payment state | Strong transactional correctness | Financial correctness |
| Driver latest location | Bounded staleness may be acceptable | Small delays are often tolerable |
| Driver availability display | Depends on operation; often freshness-aware | Avoid assigning an unavailable driver |
| Analytics dashboard | Eventual consistency | Reports need not update instantly |
| Passenger profile | Read-your-writes or stronger where required | Users expect to see their changes |

A key lesson:

**Choose consistency requirements per operation, not merely per database.**

For example, a dispatch map may display slightly stale driver locations, but the final assignment must still be validated atomically against authoritative state.

---

## 8. How Quorum Helps During Network Partitions

Imagine five replicas distributed across a cluster.

A network partition separates them into two groups:

<pre>
       Partition A              Partition B
    ┌───────────────┐        ┌───────────────┐
    │ R1  R2  R3    │        │ R4  R5        │
    └───────────────┘        └───────────────┘
          3 replicas              2 replicas
</pre>

If the system requires a majority of three:

- Partition A can continue operations that require quorum.
- Partition B cannot obtain quorum and may reject or delay those operations.

This avoids allowing both sides to independently commit conflicting majority-approved decisions.

The exact behavior depends on the database's protocol and configuration.

### Why not allow both sides to write?

If both sides accept conflicting updates independently, the system needs a way to reconcile them later.

Some data can be merged. Other data cannot be safely merged.

For example:

- Two updates to a driver's display preference may be reconcilable.
- Two different drivers assigned to the same ride cannot both be accepted as valid final assignments.

The business meaning of the data matters.

---

## 9. Conflict Resolution

When a system allows concurrent updates during a partition, it may need conflict resolution.

Common approaches include:

### Last-write-wins

The system chooses the update with the latest timestamp or version according to its conflict-resolution rule.

Simple, but potentially dangerous if clock differences or business semantics matter.

### Version-based conflict detection

Each record has a version.

If two clients attempt to update version 5, the system can detect that both updates were based on the same previous state.

One update succeeds; the other must be retried or reconciled.

### Application-level merge

The application defines how to combine updates.

For example, adding two independent entries to a set may be safely merged.

### CRDTs

Conflict-free Replicated Data Types are data structures designed so that certain concurrent updates can be merged deterministically while preserving defined properties.

They are useful for selected collaborative and distributed data types, but they are not a general solution for every business transaction.

**Important:** Never assume that a technical conflict-resolution strategy automatically preserves business correctness.

---

## 10. CAP vs. PACELC

CAP focuses on what happens during a network partition.

PACELC extends the discussion:

- **P:** If there is a partition, choose between Availability (A) and Consistency (C).
- **E:** Else, when there is no partition, choose between Latency (L) and Consistency (C).

The idea is that even when a network is healthy, stronger coordination may introduce additional latency.

Example:

A write that requires confirmation from multiple replicas may take longer than a local write.

So a distributed database design involves more than the CAP partition scenario.

PACELC is a conceptual framework, not a fixed classification of every database.

---

## 11. Applying These Concepts to Ride Assignment

Consider the requirement:

"A ride must never be assigned to two drivers simultaneously."

A possible design:

1. A dispatcher selects a candidate driver.
2. The assignment service sends a request to the authoritative database.
3. The database performs an atomic conditional update or transaction.
4. A uniqueness constraint or equivalent invariant protects the assignment.
5. If the ride is already assigned or cancelled, the operation fails.
6. On success, the service commits the transaction.
7. An event is published through a transactional outbox.
8. Other services update their read models and dashboards asynchronously.

### What if a replica is disconnected?

The system must not let a disconnected replica independently approve an assignment based on stale information.

It should route the write to the authoritative coordination path or reject/delay the operation.

### What if Redis is unavailable?

The assignment service should still rely on authoritative transactional data.

Redis may help identify candidate drivers, but it should not be the final authority for whether a ride can be assigned.

### What if the dashboard is stale?

The dashboard can display a freshness indicator or refresh its data. The final assignment still goes through the authoritative transaction.

This is an example of using different consistency requirements for different parts of the same workflow.

---

## 12. Common Interview Mistakes

1. **Saying CAP means you can permanently choose any two properties.** The trade-off becomes important during a partition.
2. **Confusing CAP consistency with ACID consistency.** They are different concepts.
3. **Assuming AP means data is always available.** Node failures, overload, and other problems can still make requests fail.
4. **Assuming CP means the whole system must shut down.** Some nodes and operations may continue.
5. **Treating every operation as equally consistency-sensitive.** Business requirements differ.
6. **Assuming quorum automatically guarantees linearizability.** The complete replication protocol matters.
7. **Using last-write-wins for business-critical records without analysis.** A later timestamp does not necessarily represent the correct business decision.
8. **Ignoring stale reads from replicas.** This can cause surprising user experiences and incorrect workflows.

---

## 13. Practical Design Checklist

Before choosing a consistency model:

- [ ] Identify which operations require strong consistency.
- [ ] Identify which data can tolerate stale reads.
- [ ] Define acceptable staleness where needed.
- [ ] Understand replication and acknowledgement behavior.
- [ ] Define what happens when a quorum is unavailable.
- [ ] Plan for network partitions and recovery.
- [ ] Identify possible concurrent-write conflicts.
- [ ] Define business-level conflict-resolution rules.
- [ ] Monitor replication lag and stale-read behavior.
- [ ] Test failure scenarios, not just normal operation.

---

## 14. Revision Questions

1. What does CAP stand for?
2. What is a network partition?
3. What does consistency mean in the CAP theorem?
4. What is the difference between CAP consistency and ACID consistency?
5. Why can't a distributed system guarantee both consistency and availability during a partition?
6. What is the difference between CP and AP behavior?
7. What is quorum? Calculate the majority quorum for seven replicas.
8. If N = 5, R = 2, and W = 4, does the common quorum-overlap condition hold?
9. What is eventual consistency?
10. What is read-your-writes consistency?
11. Why might driver locations tolerate stale reads while ride assignments cannot?
12. What does PACELC add to CAP?
13. Why is last-write-wins not always safe?
14. Why should a disconnected replica not independently approve a ride assignment?

---

## 15. Practice Exercise — Design for Network Failure

Consider a ride-booking platform with:

- Three database replicas.
- A ride-assignment service.
- A driver-location service.
- A dispatcher dashboard.
- A Redis cache.

A network partition separates one replica from the other two.

Answer these questions:

1. Should the two-replica majority continue accepting ride assignments? Why?
2. Should the isolated replica accept ride assignments independently?
3. Can the isolated replica continue serving driver-location reads?
4. What should the dispatcher dashboard display if its driver-location data is stale?
5. What should happen if Redis is unavailable?
6. How would you prevent two conflicting assignments after the network recovers?
7. Which operations should require strong consistency, and which can tolerate eventual consistency?

### Bonus interview question

"You are designing a globally distributed ride-booking system. The business wants low latency in every region, but ride assignment must never be duplicated. How would you approach the architecture?"

Think about regional reads, authoritative writes, coordination, and the latency trade-off.

**Key takeaway:** Distributed systems are not about making every operation perfectly consistent and always available. They are about identifying what must never go wrong, what can temporarily be stale, and how the system behaves when communication fails.

### Practice before moving on

Pay particular attention to questions 5, 7, 8, and the bonus interview question. These test whether you understand the actual trade-offs rather than just memorizing CAP terminology.