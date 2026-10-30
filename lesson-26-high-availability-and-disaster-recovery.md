# Lesson 26 — High Availability and Disaster Recovery: Multi-AZ, Multi-Region, Failover, RTO, and RPO

A production system must do more than handle normal traffic. It must continue operating when servers crash, databases become unavailable, network connections fail, or an entire data center experiences an outage.

In this lesson, we'll explore high availability, redundancy, failover, multi-AZ and multi-region architectures, disaster recovery, RTO, RPO, and backup strategies.

## 1. What Is High Availability?

High Availability (HA) means designing a system to remain accessible and operational despite individual component failures.

Imagine our ride-booking platform runs on a single application server.

```text
Passenger
    |
    v
Application Server
    |
    v
PostgreSQL
```

If the application server crashes, the entire application becomes unavailable.

This is a Single Point of Failure (SPOF).

A SPOF is a component whose failure can cause the entire system, or a critical part of it, to stop working.

### High Availability Architecture

We can introduce redundancy:

```text
                  Passenger
                      |
                      v
                 Load Balancer
                      |
              +-------+-------+
              |               |
              v               v
           App 1             App 2
              |               |
              +-------+-------+
                      |
                      v
                  PostgreSQL
```

Now, if App 1 fails, the load balancer can route requests to App 2.

But PostgreSQL is still a single point of failure.

We need to consider redundancy at every critical layer.

### Key Principle

**High availability is not achieved by adding multiple application servers alone. Every critical dependency must be considered.**

---

## 2. Availability and Downtime

Availability is commonly expressed as a percentage.

Formula:

\[
Availability = \frac{Uptime}{Total\ Time}\times100
\]

Alternatively:

\[
Availability = \frac{MTBF}{MTBF+MTTR}\times100
\]

Where:

- MTBF = Mean Time Between Failures.
- MTTR = Mean Time To Repair or Recover.

### Availability Targets

| Availability | Approximate Downtime per 30-Day Month |
|---|---:|
| 99% | 7 hours 12 minutes |
| 99.9% | 43 minutes 12 seconds |
| 99.95% | 21 minutes 36 seconds |
| 99.99% | 4 minutes 19 seconds |
| 99.999% | 26 seconds |

These figures assume a 30-day month.

### Why Do Additional Nines Matter?

Moving from 99% to 99.9% reduces allowed downtime significantly.

But moving from 99.99% to 99.999% requires much more engineering effort.

Higher availability can require:
- Additional infrastructure.
- Redundant network paths.
- Automated failover.
- Multi-region deployments.
- Continuous monitoring.
- More complex data replication.
- Regular disaster recovery testing.

Availability targets should reflect business requirements and cost.

---

## 3. Redundancy

Redundancy means having additional components available to take over when the primary component fails.

### Active-Passive Architecture

One component handles production traffic while another waits as a standby.

```text
             Incoming Traffic
                    |
                    v
                Primary
                    |
                    v
               Active Server

                Standby
             (Waiting for Failover)
```

If the primary fails, the standby becomes active.

Advantages:
- Simpler traffic management.
- Useful for stateful components.
- Can reduce operational complexity.

Disadvantages:
- Standby capacity may be underutilized.
- Failover takes time.
- Standby may not be fully ready or synchronized.

### Active-Active Architecture

Multiple components handle production traffic simultaneously.

```text
              Incoming Traffic
                     |
                     v
                Load Balancer
                     |
              +------+------+
              |             |
              v             v
           Server 1      Server 2
           (Active)      (Active)
```

Advantages:
- Better resource utilization.
- Traffic is already distributed.
- Can provide better availability.

Disadvantages:
- More complex state management.
- Requires careful data consistency design.
- Failover and traffic routing can be more complicated.

### Comparison

| Active-Passive | Active-Active |
|---|---|
| One primary component | Multiple active components |
| Standby waits | All active components serve traffic |
| Failover required | Traffic can shift between active components |
| Potentially simpler state handling | More complex coordination |
| Some capacity may be unused | Better capacity utilization |

---

## 4. High Availability Across Infrastructure Layers

Let's examine each major layer.

### 4.1 Application Layer

Run multiple stateless application instances.

```text
             Load Balancer
                   |
          +--------+--------+
          |        |        |
          v        v        v
        App 1    App 2    App 3
```

Important requirements:
- Stateless application servers.
- Health checks.
- Automatic replacement of failed instances.
- Graceful shutdown.
- No critical local-only state.

### 4.2 Database Layer

A common PostgreSQL architecture uses a primary database and one or more replicas.

```text
               Application
                    |
                    v
               Primary DB
                    |
              Replication
                    |
             +------+------+
             |             |
             v             v
          Replica 1     Replica 2
```

If the primary fails, an eligible replica may be promoted.

However, failover must consider:
- Replication lag.
- Data loss risk.
- Split-brain prevention.
- Client reconnection.
- Transaction recovery.
- Connection routing.

A replica is not automatically a complete HA solution.

### 4.3 Cache Layer

A Redis deployment can use replicas and automated failover.

Important considerations:
- Replication.
- Persistence configuration.
- Failover detection.
- Client reconnection.
- Cache warming.
- Behavior when cache is unavailable.

If Redis is only a cache, the database may be the fallback.

But if Redis stores sessions, locks, or other important state, its failure can have greater consequences.

### 4.4 Message Broker Layer

For Kafka, resilience involves:
- Multiple brokers.
- Replication factor.
- In-sync replicas.
- Appropriate producer acknowledgements.
- Leader election.
- Consumer recovery.

For RabbitMQ, resilience may involve:
- Multiple nodes.
- Quorum queues.
- Replicated queue state.
- Publisher confirms.
- Consumer recovery.

Broker replication should be designed around the required durability and availability guarantees.

### 4.5 Network Layer

Consider:
- Redundant load balancers.
- Multiple network paths.
- DNS failover.
- Firewall availability.
- Private connectivity between services.
- External provider dependencies.

A highly available application is still unavailable if its only network entry point fails.

---

## 5. Availability Zones (AZs)

A cloud region commonly contains multiple Availability Zones.

An AZ is an isolated infrastructure location within a region, designed to reduce the impact of certain failures.

Example:

```text
              Cloud Region
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
     AZ-1        AZ-2        AZ-3
       |           |           |
       v           v           v
     App 1       App 2       App 3
       |           |           |
       +-----------+-----------+
                   |
             Shared Services
```

The goal is to survive failures affecting one zone.

### Multi-AZ Architecture

Suppose App 1 runs in AZ-1 and App 2 runs in AZ-2.

If AZ-1 becomes unavailable, traffic can continue through AZ-2.

Database services may also use multi-AZ replication or managed failover capabilities.

### Benefits

- Protection from certain data-center-level failures.
- Better fault isolation.
- Improved infrastructure availability.
- Lower latency between components than geographically distant regions in many cases.

### Limitations

Multi-AZ does not necessarily protect against:
- Entire regional outages.
- Application bugs deployed everywhere.
- Shared configuration errors.
- Credential compromise.
- Logical data corruption.

It is important to understand which failure scenarios the architecture actually protects against.

---

## 6. Multi-Region Architecture

Multi-region architecture deploys infrastructure in geographically separate cloud regions.

Example:

```text
                 Global DNS / Routing
                         |
                 +-------+-------+
                 |               |
                 v               v
             Region A         Region B
                 |               |
          +------+-----+   +-----+------+
          |            |   |            |
          v            v   v            v
       App Servers  DB    App Servers   DB
```

If Region A becomes unavailable, traffic can be redirected to Region B.

### Why Multi-Region?

- Protection against regional outages.
- Lower latency for geographically distributed users.
- Geographic disaster recovery.
- Regional data residency requirements.

### Challenges

Multi-region architecture introduces complexity:

1. Data replication.
2. Replication lag.
3. Conflict resolution.
4. Global routing.
5. Failover automation.
6. Regional capacity planning.
7. Cross-region network costs.
8. Data residency and compliance.

Multi-region is not automatically better for every application.

A well-designed multi-AZ system may be sufficient for many products.

---

## 7. Database Failover

Database failover means promoting a standby database when the primary database becomes unavailable.

### Normal Operation

```text
Application
    |
    v
Primary PostgreSQL
    |
    | Replication
    v
Standby PostgreSQL
```

### Failure Scenario

```text
Application
    |
    v
Primary PostgreSQL
    |
    X
  Failure

Standby PostgreSQL
    |
    v
Promoted to Primary
    |
    v
Application Reconnects
```

### Failover Steps

1. Detect primary database failure.
2. Confirm that the primary is genuinely unavailable.
3. Select an eligible standby.
4. Promote the standby.
5. Update database routing or service discovery.
6. Reconnect application clients.
7. Verify write availability.
8. Rebuild redundancy by creating a new standby.

### Important Risks

**Split Brain**

Two database nodes both believe they are the primary.

This can lead to conflicting writes and data divergence.

Use proper leader election, fencing, and cluster management mechanisms to prevent this.

**Replication Lag**

With asynchronous replication, recent writes may not have reached the standby before failure.

Promoting that standby may lose some acknowledged transactions.

**Client Reconnection**

Applications may retain connections to the old primary.

Connection pools and routing layers must recover correctly.

Failover is not complete merely because a standby has been promoted.

---

## 8. Synchronous vs Asynchronous Replication in HA

Replication mode affects both performance and data-loss risk.

### Asynchronous Replication

```text
Primary
   |
   | Write committed
   |
   +----> Acknowledgement to Application
   |
   +----> Replication to Standby
```

The primary does not wait for the standby to confirm the write.

Advantages:
- Lower write latency.
- Better tolerance of replica unavailability.

Disadvantages:
- Recent acknowledged writes may be lost during failover.

### Synchronous Replication

```text
Application
    |
    v
Primary
    |
    | Send write
    v
Standby
    |
    | Acknowledge according to sync mode
    v
Primary
    |
    v
Application receives acknowledgement
```

The primary waits for the configured synchronous replication acknowledgement before reporting success.

Advantages:
- Stronger protection against losing acknowledged writes under the configured failure assumptions.

Disadvantages:
- Higher write latency.
- Availability can depend on standby health and network connectivity.
- The exact guarantee depends on the database's synchronous commit configuration.

**Important:** Synchronous replication does not automatically mean every standby has durably flushed the transaction. Understand the precise acknowledgement mode.

---

## 9. Disaster Recovery (DR)

High availability protects against component failures.

Disaster Recovery focuses on restoring operations after a major disruptive event.

Examples:
- Entire region unavailable.
- Accidental database deletion.
- Data corruption.
- Ransomware.
- Major infrastructure misconfiguration.
- Loss of critical credentials.

A disaster recovery strategy defines:
- What needs to be recovered.
- How quickly it must be recovered.
- How much data loss is acceptable.
- Where recovery infrastructure exists.
- How recovery is tested.

### HA vs DR

| High Availability | Disaster Recovery |
|---|---|
| Handles component failures | Handles major disruptions |
| Usually automatic | May require coordinated recovery |
| Often uses redundancy | Uses backups and recovery infrastructure |
| Focuses on continuity | Focuses on restoration |
| Example: App instance failure | Example: Entire region lost |

HA and DR complement each other. Neither replaces the other.

---

## 10. RTO and RPO

These are two of the most important disaster recovery metrics.

### RTO — Recovery Time Objective

RTO defines the maximum acceptable time to restore a service after a disruption.

Example:

```text
RTO = 30 minutes
```

If a regional outage occurs, the system should be restored within 30 minutes.

### RPO — Recovery Point Objective

RPO defines the maximum acceptable amount of data loss, expressed as a period of time.

Example:

```text
RPO = 5 minutes
```

This means the recovery strategy should limit data loss to approximately five minutes of changes under the defined failure scenario.

### Example

Suppose:
- Outage starts at 10:00 AM.
- Service recovers at 10:25 AM.
- Last recoverable data point is 9:57 AM.

Then:

```text
Actual Recovery Time = 25 minutes

Actual Data Loss Window = 3 minutes
```

If our targets are:

```text
RTO = 30 minutes
RPO = 5 minutes
```

Both targets were met in this example.

### RTO vs RPO

| RTO | RPO |
|---|---|
| How quickly must we recover? | How much data can we lose? |
| Time to restore service | Time-based recovery point |
| Drives recovery infrastructure | Drives backup and replication strategy |

---

## 11. Disaster Recovery Strategies

There are several common recovery strategies.

### 11.1 Backup and Restore

Backups are stored, but no production-ready recovery environment is running.

During a disaster:
1. Provision infrastructure.
2. Restore database backups.
3. Restore files.
4. Deploy application.
5. Validate and reopen traffic.

Advantages:
- Lowest infrastructure cost.
- Suitable for less critical systems.

Disadvantages:
- Slow recovery.
- RTO can be large.
- Recovery depends on backup quality and restore speed.

### 11.2 Pilot Light

A minimal recovery environment is maintained.

Some core services and data replication are already configured, but most compute capacity is started during a disaster.

Advantages:
- Faster recovery than backup and restore.
- Lower ongoing cost than a warm standby.

Disadvantages:
- Scaling and configuration are required during recovery.
- Recovery time depends on automation.

### 11.3 Warm Standby

A smaller but functional environment runs continuously in the recovery region.

During a disaster, capacity is increased and traffic is redirected.

Advantages:
- Faster recovery.
- Infrastructure is already operational.

Disadvantages:
- Higher ongoing cost.
- Requires continuous replication and validation.

### 11.4 Hot Standby / Active-Active

A fully provisioned recovery environment is ready, or multiple regions actively serve production traffic.

Advantages:
- Very fast failover potential.
- High availability.
- Better regional distribution.

Disadvantages:
- Highest complexity.
- Higher infrastructure cost.
- More difficult data consistency and conflict handling.

### Comparison

| Strategy | Recovery Speed | Cost | Complexity |
|---|---|---|---|
| Backup and Restore | Slow | Low | Low to medium |
| Pilot Light | Medium | Low to medium | Medium |
| Warm Standby | Fast | Medium to high | High |
| Hot Standby | Very fast | High | Very high |

These are relative comparisons. Actual recovery times depend on architecture, automation, data volume, and tested procedures.

---

## 12. Backup Strategies

Backups are essential for disaster recovery.

Replication is not a substitute for backups.

Why?

If someone accidentally deletes a database record, that deletion may replicate to every replica.

If malicious software corrupts data, replication may copy the corrupted data too.

Backups provide recovery points from before the problem occurred.

### Common Backup Types

**Full Backup**

Copies the entire database or dataset.

Advantages:
- Simple restoration model.

Disadvantages:
- More storage.
- More time to create and transfer.

**Incremental Backup**

Stores changes since the previous backup of any type.

Advantages:
- Smaller backup size.
- Efficient frequent backups.

Disadvantages:
- Restore may require a chain of backups.

**Differential Backup**

Stores changes since the last full backup.

Advantages:
- Restore usually requires the last full backup and the latest differential backup.

Disadvantages:
- Differential backup size grows until the next full backup.

### PostgreSQL Backup Considerations

PostgreSQL recovery strategies can include:
- Physical base backups.
- Write-Ahead Log (WAL) archiving.
- Point-in-Time Recovery (PITR).
- Logical backups for selected schemas or objects.

PITR allows restoring a database to a specific time using a base backup and a continuous sequence of WAL records.

Example:

```text
Full Backup:  Sunday 12:00 AM
WAL Archive:  Continuous
Accidental Delete: Tuesday 10:15 AM

Recovery:
Restore base backup
    |
    v
Replay WAL records
    |
    v
Stop before 10:15 AM
    |
    v
Recovered Database
```

The exact recovery point depends on the backup and WAL archive completeness.

### Backup Best Practices

1. Automate backups.
2. Encrypt backups.
3. Keep backups in a separate failure domain.
4. Use immutable or protected backup copies where appropriate.
5. Define retention policies.
6. Monitor backup failures.
7. Test restoration regularly.
8. Document recovery procedures.

**A backup that has never been restored successfully is not a proven recovery strategy.**

---

## 13. Failover vs Failback

These are different operations.

### Failover

Switch from the failed primary environment to a standby or recovery environment.

```text
Primary Region A
       |
       X
     Failure
       |
       v
Standby Region B
       |
       v
Becomes Active
```

### Failback

Return operations to the original or preferred environment after recovery.

```text
Region A
   |
   X
 Failure

Region B
   |
   v
Active

Region A Recovered
   |
   v
Data Reconciliation
   |
   v
Controlled Failback
```

Failback should not be automatic without checking data consistency and replication direction.

A safe procedure may require:
1. Recovering the original infrastructure.
2. Synchronizing data from the active environment.
3. Validating consistency.
4. Planning a controlled traffic switch.
5. Monitoring after the switch.

In some architectures, it may be safer to keep the recovery region active rather than immediately switching back.

---

## 14. Disaster Recovery Testing

A disaster recovery plan must be tested.

Possible tests include:

### Backup Restore Test

Restore a backup into an isolated environment and verify application functionality.

### Database Failover Test

Simulate primary database failure and measure:
- Detection time.
- Promotion time.
- Client reconnection time.
- Data loss.

### Region Failover Exercise

Redirect traffic to the recovery region.

Validate:
- DNS or global routing.
- Authentication.
- Database access.
- Message processing.
- Object storage access.
- External integrations.

### Chaos Engineering

Introduce controlled failures to understand how the system behaves.

Examples:
- Terminate an application instance.
- Introduce network latency.
- Make a dependency unavailable.
- Restart a consumer.
- Simulate a database failover.

Chaos engineering should use controlled experiments, safety boundaries, and clear rollback procedures.

### DR Test Checklist

- [ ] Recovery infrastructure available.
- [ ] Backups restorable.
- [ ] Replication healthy.
- [ ] Secrets and certificates available.
- [ ] DNS and routing procedures documented.
- [ ] Application deployment automated.
- [ ] External dependencies identified.
- [ ] RTO measured.
- [ ] RPO measured.
- [ ] Recovery ownership assigned.
- [ ] Failback procedure documented.

---

## 15. Capacity Planning During Failover

A common mistake is designing a system where each region handles only half the normal traffic.

Imagine:

```text
Normal Traffic = 10,000 RPS

Region A = 5,000 RPS
Region B = 5,000 RPS
```

If Region A fails, Region B must suddenly handle 10,000 RPS.

If its safe capacity is only 5,000 RPS, failover will cause another outage.

### N+1 Capacity Planning

Ensure enough remaining capacity to handle expected demand after a component or zone failure.

For example:

```text
Required Peak Traffic = 10,000 RPS

Region B Failover Capacity = 12,000 RPS
```

This gives some headroom after failover.

Consider:
- Database capacity.
- Worker capacity.
- Network bandwidth.
- Cache capacity.
- Third-party provider limits.
- Connection pools.

A region that can accept traffic but cannot process it is not a successful recovery region.

---

## 16. Worked Example — Ride-Booking Platform DR Design

Suppose our platform has:

- Passenger applications.
- Driver applications.
- API Gateway.
- Booking Service.
- Driver Service.
- PostgreSQL.
- Redis.
- Kafka.
- Object Storage.
- Notification Service.

Business requirements:

```text
RTO = 30 minutes
RPO = 5 minutes
```

### Proposed Architecture

```text
                   Global Routing
                         |
                +--------+--------+
                |                 |
                v                 v
             Region A          Region B
              Primary           Recovery
                |                 |
          +-----+-----+     +-----+-----+
          |           |     |           |
          v           v     v           v
       App Tier    Primary  App Tier   Standby
                    DB                  DB
          |           |     |           |
          +-----------+     +-----------+
                      Replication

                Object Storage
                Replication / Backup

                   Kafka
                Replication or
                Recovery Strategy
```

### Design Decisions

**Application Layer**

Run multiple instances across zones in both regions.

**PostgreSQL**

Use primary and standby replication with a clearly defined promotion procedure.

**Redis**

Treat cache data as rebuildable where possible. Protect any critical Redis state separately.

**Kafka**

Define how important events are replicated or recovered. Consider replay from a durable source or replicated topics, depending on the deployment architecture.

**Object Storage**

Use versioning, backup, or cross-region replication according to file recovery requirements.

**DNS and Routing**

Prepare health-based routing and documented failover procedures.

### Important Consideration

Not every component needs identical RTO and RPO targets.

For example:
- Booking transactions may need strict recovery objectives.
- Analytics events may tolerate longer delays.
- Profile images may be recoverable from object storage.
- Some cache entries can simply be rebuilt.

Design recovery based on business impact.

---

## 17. Common High Availability and DR Mistakes

1. Assuming multiple application servers guarantee high availability.
2. Ignoring database and network single points of failure.
3. Treating database replicas as backups.
4. Assuming asynchronous replication cannot lose acknowledged writes.
5. Deploying multi-region without defining data ownership.
6. Forgetting recovery-region capacity.
7. Failing to test backup restoration.
8. Having no documented failover procedure.
9. Ignoring secrets and certificates during disaster recovery.
10. Forgetting message broker recovery.
11. Automatically failing back without data reconciliation.
12. Defining RTO and RPO without validating them through tests.
13. Assuming DNS changes happen instantly.
14. Ignoring third-party provider dependencies.

---

## 18. Revision Questions

1. What is high availability?
2. What is a Single Point of Failure?
3. Explain active-active and active-passive architectures.
4. What is the difference between availability and durability?
5. What is the difference between multi-AZ and multi-region?
6. How does database failover work?
7. What is split brain?
8. Explain synchronous and asynchronous replication.
9. What is disaster recovery?
10. How is HA different from DR?
11. What is RTO?
12. What is RPO?
13. Calculate monthly downtime for 99.9% availability.
14. Explain backup and restore, pilot light, warm standby, and hot standby.
15. Why are backups necessary even when replication exists?
16. What is PostgreSQL Point-in-Time Recovery?
17. What is the difference between failover and failback?
18. Why should disaster recovery be tested regularly?
19. What is N+1 capacity planning?
20. How would you design a recovery architecture for a ride-booking platform?

---

## 19. Practical Exercise — Design HA and DR for Ride Booking

Design an HA and DR architecture for the ride-booking platform.

Assume:

- 1 million daily passengers.
- 100,000 active drivers.
- Peak booking traffic of 5,000 RPS.
- PostgreSQL is the primary transactional database.
- Kafka handles asynchronous events.
- Redis handles caching and selected shared state.
- Object storage holds invoices and driver documents.

Business requirements:

```text
Booking API Availability: 99.99%
RTO: 30 minutes
RPO: 5 minutes
```

Answer:

1. How many application instances would you deploy across zones?
2. How would you eliminate database single points of failure?
3. Would you use synchronous or asynchronous replication?
4. How would you prevent split brain?
5. How would you recover Kafka and its messages?
6. How would you protect uploaded files?
7. Which DR strategy would you choose and why?
8. How would you ensure the recovery region can handle 5,000 RPS?
9. How would you implement failover?
10. How would you test the RTO and RPO requirements?
11. Which components need backups, and which can be rebuilt?
12. How would you perform failback?

### Final Takeaway

High availability and disaster recovery solve different problems:

- **Redundancy:** Provides replacement capacity.
- **Multi-AZ:** Protects against certain zone failures.
- **Multi-Region:** Protects against regional failures.
- **Failover:** Moves operations to a healthy environment.
- **Backups:** Protect against data loss and corruption.
- **RTO:** Defines how quickly service must recover.
- **RPO:** Defines how much data loss is acceptable.

A system is not truly production-ready until its recovery strategy has been tested.

### Practice focus

Design a multi-AZ PostgreSQL architecture and a regional disaster recovery plan. Pay particular attention to the difference between replication, backups, RTO, and RPO.