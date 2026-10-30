# Lesson 11: Database Sharding and Partitioning

In this lesson, we'll learn how to handle datasets that become too large or workloads that exceed what a single database server can efficiently support.

The key distinction to remember is:
- Partitioning divides a table into smaller parts.
- Sharding distributes data across multiple database nodes.
They can be used together, but they solve different architectural problems.

## 1. Why Do We Need Partitioning and Sharding?

Imagine a ride-booking platform with the following data:

- 100 million ride records.
- Billions of driver location updates.
- Millions of passengers and drivers.
- A continuously growing ride history.
- High volumes of concurrent reads and writes.

Initially, a single PostgreSQL database may handle this workload comfortably. As data and traffic grow, several problems may emerge:

- Large tables become expensive to scan and maintain.
- Indexes consume substantial storage.
- Queries and maintenance operations take longer.
- A single database server becomes a resource bottleneck.
- Storage capacity becomes a concern.
- The database may struggle to meet latency targets under peak traffic.

There are several ways to address these problems:

1. Optimize queries and indexes.
2. Archive or delete data according to retention requirements.
3. Partition large tables.
4. Add read replicas.
5. Shard data across multiple database nodes.

These techniques are not interchangeable. The right choice depends on whether the primary problem is query efficiency, storage organization, read throughput, or write and storage capacity.

**Important principle:** Do not introduce sharding just because a table has millions of rows. First determine whether a single well-configured database can still meet the requirements.

---

## 2. What Is Database Partitioning?

Partitioning divides a large logical table into smaller physical or logical pieces, called partitions.

From the application's perspective, the data can still appear as one table.

For example, a `rides` table could contain 100 million records.

Instead of storing all records in one undivided table, the database divides them into partitions based on a chosen rule.

```text
                    rides
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
      Partition 1  Partition 2  Partition 3
      Jan–Mar      Apr–Jun      Jul–Sep
```

This example uses time ranges. The actual partition boundaries depend on the schema and workload.

### Why partition a table?

Partitioning can help with:

- Managing very large tables.
- Pruning irrelevant data from eligible queries.
- Improving retention and archival operations.
- Managing indexes for individual partitions.
- Organizing data by time, tenant, or another suitable key.
- Reducing the amount of data scanned for queries that constrain the partition key.

However, partitioning does not automatically make every query faster.

If a query cannot exclude irrelevant partitions, it may still need to access many or all partitions.

---

## 3. Common Partitioning Strategies

### A. Range partitioning

Rows are assigned to partitions according to a range of values.

For example, partition rides by creation date:

```text
rides_2026_q1
rides_2026_q2
rides_2026_q3
rides_2026_q4
```

A partition might contain rides created during a particular quarter.

**Good use cases:**
- Time-series data.
- Ride histories.
- Audit records.
- Logs and event records.
- Data with time-based retention policies.

**Advantages:**
- Time-based queries can skip unrelated partitions.
- Old partitions can be archived or detached more easily.
- Retention operations can be more efficient than deleting huge numbers of individual rows.

**Trade-offs:**
- Queries spanning many time periods may touch many partitions.
- Future partitions must be created or handled through an appropriate default strategy.
- Poor partition boundaries can produce uneven data distribution.
- Partition management introduces operational work.

### B. List partitioning

Rows are assigned according to a predefined list of values.

For example:

```text
rides_region_north
rides_region_south
rides_region_east
rides_region_west
```

This may be useful when regions are a meaningful and stable classification.

**Advantages:**
- Easy to isolate known categories.
- Can help queries that consistently filter by the partition key.

**Trade-offs:**
- New categories require a management strategy.
- One region may contain far more data than others.
- Region-based partitioning does not automatically distribute writes across separate servers.

### C. Hash partitioning

Rows are assigned using a hash of a partition key.

For example:

```text
hash(passenger_id)
        |
        v
  +-----+-----+-----+
  |     |     |     |
  v     v     v     v
 P0    P1    P2    P3
```

The actual hash function and partition mapping depend on the database implementation.

**Advantages:**
- Can distribute rows more evenly when the key has a suitable distribution.
- Useful when a workload frequently looks up records by the chosen key.
- Avoids some forms of uneven range distribution.

**Trade-offs:**
- Queries that do not constrain the partition key may need to visit many partitions.
- Changing the number of partitions or rebalancing data can require planning.
- Hash partitioning does not, by itself, distribute data across different database servers.

---

## 4. PostgreSQL Table Partitioning Example

Imagine a ride-history table with a large number of records.

We decide to partition it by `created_at`.

A simplified PostgreSQL schema could be:

```sql
CREATE TABLE rides (
    ride_id BIGINT NOT NULL,
    passenger_id BIGINT NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (ride_id, created_at)
) PARTITION BY RANGE (created_at);
```

Then create a partition:

```sql
CREATE TABLE rides_2026_q4
PARTITION OF rides
FOR VALUES FROM ('2026-10-01 00:00:00+00')
             TO ('2027-01-01 00:00:00+00');
```

The range's lower bound is inclusive, and its upper bound is exclusive.

In a real system, all required partitions must be created or managed through a suitable automated process. The schema and indexes also need to be designed around the application's query patterns.

### Why include `created_at` in the primary key?

In PostgreSQL declarative partitioning, a unique constraint or primary key on the partitioned table generally must include all partition-key columns.

This example therefore uses `(ride_id, created_at)`.

It also introduces a modeling consideration: if the application requires `ride_id` to be globally unique by itself, that uniqueness may need a separate design, such as an appropriate globally unique identifier strategy or another authoritative uniqueness mechanism.

Do not assume partitioning preserves every uniqueness guarantee in exactly the same way as an unpartitioned table.

---

## 5. Partition Pruning

Partition pruning allows PostgreSQL to skip partitions that cannot contain rows matching a query.

Suppose the table is partitioned by `created_at`.

Query:

```sql
SELECT ride_id, status
FROM rides
WHERE created_at >= TIMESTAMPTZ '2026-10-01 00:00:00+00'
  AND created_at <  TIMESTAMPTZ '2027-01-01 00:00:00+00';
```

If the partition boundaries match this range, PostgreSQL may access only the relevant partition or partitions.

Without partition pruning, a query might need to consider a much larger portion of the dataset.

### When pruning helps

It is most useful when queries include conditions that allow the planner to determine which partitions can contain matching rows.

### When pruning does not help much

Suppose the query is:

```sql
SELECT *
FROM rides
WHERE passenger_id = 12345;
```

If the table is partitioned by `created_at` and the query does not restrict the time range, PostgreSQL may need to inspect many partitions to find all rides for that passenger.

This is why partition keys should be chosen based on actual query patterns and operational needs.

**A partitioning strategy that helps one query can make another query more complicated or expensive.**

---

## 6. Partitioning Is Not Sharding

This is one of the most important distinctions in database architecture.

### Partitioning

Partitioning divides a table into smaller pieces within a database architecture.

```text
               PostgreSQL Database
                       |
                       v
                  rides table
                       |
             +---------+---------+
             |         |         |
             v         v         v
          Part A     Part B     Part C
```

In PostgreSQL declarative partitioning, partitions are managed within the same database cluster, although storage and tablespaces can be configured in different ways.

Partitioning can improve data management and eligible query performance, but it does not automatically spread write workload across independent database servers.

### Sharding

Sharding distributes data across multiple database nodes.

```text
                 Application
                      |
                  Shard Router
                 /     |     \
                v      v      v
           Database A Database B Database C
           Passengers Passengers Passengers
           1–100K     100K–200K  200K–300K
```

The diagram is illustrative. A real shard distribution may use hashes, ranges, virtual shards, or another mapping strategy.

Each shard stores only a portion of the logical dataset.

### Comparison

| Property | Partitioning | Sharding |
|---|---|---|
| Main purpose | Organize large tables and improve eligible operations | Distribute data and workload across database nodes |
| Data placement | Usually within a database cluster | Across separate database nodes or clusters |
| Read scaling | Can reduce work through pruning | Can distribute queries across shards |
| Write scaling | Usually does not distribute writes across independent servers by itself | Can distribute writes when data is spread appropriately |
| Cross-partition or cross-shard queries | Often handled within the database | May require distributed routing or coordination |
| Operational complexity | Usually lower | Usually higher |
| Rebalancing | Partition management | Shard migration and redistribution |

**Remember:** A system can use both techniques. Each shard may contain its own partitioned tables.

---

## 7. What Is a Shard Key?

A shard key determines which shard stores a particular record.

Consider a ride-booking platform with three database shards.

One possible strategy is to shard by `passenger_id`.

```text
passenger_id
      |
      v
Hash function
      |
      +------> Shard A
      +------> Shard B
      +------> Shard C
```

For a given passenger, the router computes the shard location and sends the query to the appropriate database.

### What makes a good shard key?

A good shard key should be evaluated against:

1. **Distribution:** Does it spread data and traffic reasonably evenly?
2. **Query locality:** Do common queries contain the key?
3. **Write distribution:** Does it avoid concentrating writes on one shard?
4. **Growth:** Will the key remain useful as data volume increases?
5. **Operational needs:** Can data be moved or rebalanced safely?
6. **Business requirements:** Do common transactions need records stored together?

There is no universally best shard key. It depends on the workload.

---

## 8. Choosing a Shard Key: Passenger ID vs. Ride ID

Suppose the platform frequently queries ride history for an individual passenger.

### Option A: Shard by passenger ID

Advantages:
- A passenger's ride history can be located on one shard.
- Common passenger-history queries may need only one shard.
- Passenger-scoped operations can have good data locality.

Trade-offs:
- A highly active passenger can create a hot key.
- Queries organized around drivers may need to access multiple shards.
- Cross-passenger reporting can become more complex.

### Option B: Shard by ride ID

Advantages:
- Ride records may be distributed more evenly if IDs are generated appropriately.
- A particular ride can be routed directly when its ID determines the shard.

Trade-offs:
- Passenger history may require queries across many shards unless there is another mapping or indexing strategy.
- Multi-record passenger operations may need coordination.
- A unique ride ID alone does not guarantee that related data is stored together.

### The decision

Choose a shard key based on the dominant workload, not merely the entity that appears most important in the business domain.

If passenger history is the primary access pattern, passenger-based sharding may be attractive. If rides are independently accessed and distributed reporting is more important, another key may be preferable.

The choice also depends on write rates, hot-key risk, consistency requirements, and operational constraints.

---

## 9. Shard Distribution and Hotspots

A system can have multiple shards and still suffer from an uneven workload.

Imagine three shards:

| Shard | Data volume | Incoming writes per second |
|---|---:|---:|
| A | 30% | 25% |
| B | 35% | 30% |
| C | 35% | 45% |

Shard C receives a disproportionate amount of write traffic.

This may happen because:
- One key is unusually active.
- A particular tenant dominates usage.
- A time-based key directs new writes to one location.
- The shard distribution is uneven.
- A routing or partitioning decision creates a hotspot.

### Hot-key problem

Suppose all ride records for one very active passenger go to the same shard.

That passenger may generate enough traffic to overload the shard even if the overall dataset is well distributed.

Potential approaches include:
- Choosing a different shard key.
- Using composite or derived keys where business requirements allow.
- Separating exceptional high-volume tenants.
- Distributing read traffic through replicas.
- Redesigning access patterns to reduce concentrated work.

These approaches have trade-offs. Splitting data that must participate in one transaction can make consistency and coordination more complicated.

---

## 10. Cross-Shard Queries

Sharding makes some queries easy and others difficult.

Consider a query:

```sql
SELECT *
FROM rides
WHERE passenger_id = 12345;
```

If `passenger_id` determines the shard, the router can send the query directly to one shard.

Now consider:

```sql
SELECT status, COUNT(*)
FROM rides
GROUP BY status;
```

If ride records are distributed across all shards, this query may need to execute on every shard and combine the results.

This is sometimes called a scatter-gather query.

### Why cross-shard queries are expensive

- More network requests.
- More database work across nodes.
- Result aggregation.
- Uneven response times across shards.
- More complicated failure handling.
- Potentially more complex consistency semantics.

A query that is fast on a single database may become much more expensive when it must access every shard.

### Common strategies

- Route queries using the shard key whenever possible.
- Maintain suitable pre-aggregated data for common reporting needs.
- Use a dedicated analytical system for broad reporting workloads.
- Avoid unnecessary cross-shard joins.
- Denormalize carefully when the read-performance benefit justifies consistency costs.

The best strategy depends on whether the query is transactional, operational, or analytical.

---

## 11. Transactions Across Shards

Transactions within a single database are relatively straightforward compared with transactions that span independent database nodes.

Suppose a ride-booking operation needs to update records on two different shards.

A normal local transaction on one shard cannot automatically make changes on the other shard atomic.

Potential approaches include:

### Strategy A: Keep related records on the same shard

If the transaction's related records share the shard key, the operation can often remain local.

This is one reason data locality is important when choosing a shard key.

### Strategy B: Distributed transactions

A distributed transaction protocol can coordinate commits across multiple participants when supported.

Trade-offs include additional complexity, coordination overhead, and availability implications during failures.

### Strategy C: Saga or workflow-based coordination

For workflows that span multiple services or databases, a saga coordinates multiple local transactions and may execute compensating actions when a later step fails.

For example:

1. Reserve a resource.
2. Charge a payment.
3. Confirm the booking.
4. If confirmation fails, initiate an appropriate compensation.

A compensation is a business operation that attempts to counteract an earlier effect. It is not necessarily a literal rollback of history.

Sagas require explicit state management, idempotency, failure handling, and recovery procedures.

### Important design principle

Do not distribute a transaction across shards casually. Prefer data placement that keeps common critical transactions local, where feasible.

---

## 12. Resharding and Rebalancing

As traffic grows, the original shard layout may no longer be sufficient.

Suppose a system starts with four shards and later needs eight.

The system must redistribute some data and update routing.

### Challenges

- Copying large amounts of data.
- Maintaining consistency during migration.
- Handling writes that occur while data is moving.
- Updating shard mappings.
- Avoiding downtime.
- Managing increased load during migration.
- Recovering safely if migration fails.

### A high-level migration strategy

1. Define the target shard layout.
2. Establish how each record maps to the new layout.
3. Copy data while tracking changes that occur during the copy.
4. Validate the copied data.
5. Synchronize remaining changes.
6. Switch routing in a controlled manner.
7. Monitor errors, latency, and data correctness.
8. Retain a recovery path until the migration is verified.

The precise method depends on the database and sharding technology.

### Virtual shards

Some architectures divide data into many logical or virtual shards and then map those virtual shards to physical database nodes.

This can make rebalancing easier because moving a subset of virtual shards does not necessarily require changing the mapping rule for every individual record.

However, virtual shards add routing and operational complexity and must be supported by the architecture.

---

## 13. Partitioning and Retention

Partitioning is particularly useful when old data needs to be removed or archived according to a retention policy.

Imagine driver location history is partitioned by month.

Instead of deleting millions of expired rows individually, the system may be able to detach or drop an entire expired partition, depending on the schema and retention policy.

This can be much more efficient than a massive row-by-row deletion.

However, dropping a partition is a destructive operation. Before doing so, the system must verify:
- The partition is entirely outside the retention period.
- Required records have been archived if necessary.
- Dependencies and constraints permit the operation.
- The operation will not affect required audit or business records.
- The backup and recovery strategy is appropriate.

Partitioning supports data lifecycle management; it does not replace a retention policy.

---

## 14. Partitioning vs. Indexing vs. Replication vs. Sharding

These techniques are often used together.

| Technique | Main purpose | Typical example |
|---|---|---|
| Indexing | Find relevant rows efficiently | Find rides for one passenger |
| Partitioning | Divide a large table into manageable parts | Separate rides by month |
| Replication | Maintain copies of data | Serve eligible reads from replicas |
| Sharding | Distribute data across database nodes | Store different passenger groups on different shards |

### Example: A growing ride-booking platform

A reasonable evolution might be:

**Stage 1:** One PostgreSQL database with well-designed indexes.

**Stage 2:** Partition large time-oriented tables to improve eligible queries and retention management.

**Stage 3:** Add read replicas when eligible read traffic overwhelms the primary.

**Stage 4:** Introduce sharding when a single database's write, storage, or resource capacity is insufficient despite appropriate optimization.

These stages are illustrative, not mandatory. A particular workload may need a different sequence.

The key is to identify the actual bottleneck before introducing a more complex architecture.

---

## 15. Common Mistakes

1. Sharding before exhausting simpler optimization options.
2. Assuming partitioning automatically distributes writes across multiple servers.
3. Choosing a shard key without examining query patterns.
4. Ignoring hot keys and uneven traffic.
5. Assuming every query will target a single shard.
6. Performing cross-shard joins without understanding the cost.
7. Ignoring distributed transaction requirements.
8. Using a time-based shard key that concentrates current writes on one shard.
9. Assuming that adding more shards automatically fixes hotspots.
10. Ignoring the complexity of rebalancing and data migration.
11. Treating partition removal as a substitute for a carefully defined retention policy.
12. Forgetting that replication, indexing, partitioning, and sharding solve different problems.

---

## 16. Worked Example: Ride History at Large Scale

Assume the platform has 500 million ride records and growing traffic.

The most common queries are:
- Retrieve a passenger's latest rides.
- Retrieve rides from a specified time period.
- Find the current state of a particular ride.
- Produce broad operational reports.

### Step 1: Optimize indexes

Use suitable indexes for passenger history, ride identifiers, and common filters.

Measure query plans and actual performance.

### Step 2: Evaluate partitioning

Partition time-oriented data if time-based queries, retention, or maintenance operations benefit from it.

Ensure the common queries can take advantage of partition pruning.

### Step 3: Evaluate replicas

If historical read traffic dominates database capacity, add read replicas for queries that tolerate the relevant consistency guarantees.

Keep authoritative operations on a suitable consistency path.

### Step 4: Evaluate sharding

If a single primary cannot meet write or storage requirements after appropriate optimization, consider sharding.

Choose the shard key based on dominant query patterns and write distribution.

### Step 5: Plan cross-shard operations

Determine how broad reports, ride lookups, and multi-record transactions will work across shards.

Consider pre-aggregation, analytical systems, and workflows where appropriate.

### Step 6: Plan recovery and rebalancing

Define how data will move as capacity changes, how routing will update, and how correctness will be verified.

The architecture should evolve because measured requirements demand it, not because a large row count alone seems intimidating.

---

## 17. Revision Questions

Answer these without looking at the notes.

1. What is database partitioning?
2. What is database sharding?
3. How do range, list, and hash partitioning differ?
4. What is partition pruning?
5. Why might partitioning fail to improve a query that does not filter on the partition key?
6. What is a shard key, and how do you choose one?
7. Why might sharding by passenger ID help ride-history queries?
8. What is a hot-key problem?
9. Why can cross-shard aggregation be expensive?
10. Why are cross-shard transactions more complicated than local transactions?
11. What is the difference between a distributed transaction and a saga?
12. What is resharding, and why is it risky?
13. How can partitioning help with retention management?
14. When should you consider sharding rather than adding read replicas?
15. Explain how indexing, partitioning, replication, and sharding could work together in a large ride-booking platform.

## Final Takeaway

Partitioning organizes large tables into smaller parts. Sharding distributes data across multiple database nodes.

Remember:

- Partitioning can improve eligible query performance and data lifecycle management.
- Sharding can distribute storage and workload when data is appropriately spread.
- Shard-key selection determines data locality and affects query complexity.
- Hotspots and cross-shard transactions require deliberate design.
- Rebalancing must preserve data correctness.
- Indexing, partitioning, replication, and sharding address different bottlenecks.

**The central rule:** Use partitioning to organize data, and use sharding when the architecture needs to distribute data and workload across database nodes. Measure first, because distributed data introduces significant operational and consistency costs.

## Practice exercise

Your ride-booking platform has 500 million rides. The most common query retrieves the latest 20 rides for a passenger, while reporting queries aggregate rides across all passengers.

1. Would you start with indexing, partitioning, sharding, or replication? Explain your first choice.
2. If you shard by passenger_id, what becomes easier, and what becomes harder?
3. If you partition by created_at, which queries benefit most, and which might still be expensive?