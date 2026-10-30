# Lesson 13 — SQL vs. NoSQL and Data Modeling

## 1. The Most Important Principle

**Choose the data model based on how the application uses the data. Then choose the database that fits the model and workload.**

Before selecting a database, answer:

- What information must we store?
- How is the information related?
- What queries will the application execute?
- Which operations must be atomic?
- How fresh must the data be?
- How much data and traffic do we expect?
- How will the data evolve over time?

For example, a ride-booking platform has several kinds of data:

- Passengers, drivers, rides, payments, and assignments.
- The latest location of each driver.
- Historical driver locations.
- Searchable addresses and places.
- Events used for analytics.

These workloads have different requirements. A single database may handle several of them, but different storage systems can be justified when their access patterns differ significantly.

---

## 2. SQL Databases

SQL databases are relational databases. They organize data into tables with defined columns and relationships.

Examples include PostgreSQL, MySQL, and SQL Server.

### Example: A ride-booking database

**Passengers**

| id | name | email |
|---|---|---|
| 101 | Rahul | rahul@example.com |
| 102 | Priya | priya@example.com |

**Rides**

| id | passenger_id | status | fare |
|---|---|---|---:|
| 501 | 101 | completed | 250 |
| 502 | 102 | requested | 180 |

Here, `passenger_id` connects a ride to a passenger.

A foreign key can enforce this relationship, preventing a ride from referencing a passenger that does not exist.

### Why use SQL?

**1. Relationships**

Relational databases are well suited to data with meaningful relationships.

For example:

- A passenger can have many rides.
- A ride can have multiple status-history records.
- A ride can have one or more payment attempts.
- A driver can be assigned to many rides over time.

**2. Transactions**

Suppose two dispatchers attempt to assign different drivers to the same ride. Database transactions, constraints, and concurrency controls can help prevent conflicting assignments.

**3. Constraints**

SQL databases can enforce rules such as:

- A passenger email must be unique.
- A fare cannot be negative.
- A ride must reference a valid passenger.
- A ride must have a valid status.

These rules protect data even when multiple services or concurrent requests modify it.

**4. Flexible querying**

SQL supports joins, aggregations, filtering, grouping, and sorting.

For example, we can retrieve a passenger's ride history and include the driver's information in a query.

### SQL trade-offs

- Complex joins can become expensive at scale.
- Poor indexing can cause slow queries.
- Large write workloads may require partitioning or other scaling strategies.
- A single relational database can become a bottleneck if every workload is forced into it.

These are engineering challenges, not reasons to avoid SQL by default.

---

## 3. NoSQL Databases

NoSQL is an umbrella term for several database models that do not primarily rely on the traditional relational-table model.

**NoSQL does not mean “no structure,” and it does not automatically mean “faster or more scalable.”**

The major categories are key-value, document, wide-column, and graph databases.

### 3.1 Key-value databases

Data is stored as a key associated with a value.

<pre>
Key                         Value
------------------------------------------------
driver:101:location         {"lat":28.61,"lng":77.23}
driver:102:location         {"lat":28.62,"lng":77.24}
</pre>

Typical use cases:

- Caching.
- Sessions.
- Latest driver locations.
- Temporary state.
- Fast lookup by a known key.

Examples: Redis and Amazon DynamoDB's key-value access patterns.

**Strength:** Efficient lookups when the key is known.

**Trade-off:** Queries across arbitrary fields or complex relationships may be less natural than in a relational database.

### 3.2 Document databases

Data is stored as documents, commonly in JSON-like structures.

Example:

```json
{
  "rideId": "ride-501",
  "passengerId": "passenger-101",
  "status": "completed",
  "pickup": {
    "latitude": 28.61,
    "longitude": 77.23
  },
  "dropoff": {
    "latitude": 28.62,
    "longitude": 77.24
  }
}
```

Typical use cases:

- Product catalogs with variable attributes.
- Content management.
- Data whose structure varies between records.
- Applications that commonly retrieve an entire document at once.

Example: MongoDB.

**Strength:** Natural representation of nested and evolving data.

**Trade-off:** Duplicated data, embedded-document growth, and cross-document consistency can become difficult to manage.

A document database can support transactions and validation; its capabilities depend on the product and configuration.

### 3.3 Wide-column databases

These databases organize data around partition keys and clustered rows or columns, depending on the system.

Typical use cases:

- Very large write-heavy workloads.
- Time-series or event data.
- Queries designed around known partition keys.
- Distributed workloads requiring horizontal scaling.

Examples: Apache Cassandra and ScyllaDB.

**Strength:** Can handle substantial distributed workloads when data is partitioned and queried appropriately.

**Trade-off:** Data modeling is often tightly coupled to access patterns. Arbitrary queries and relational joins are generally not its strength.

### 3.4 Graph databases

Graph databases model entities as nodes and relationships as edges.

Example:

<pre>
Passenger ──booked──> Ride
                         │
                      assigned
                         ▼
                       Driver
                         │
                     belongs to
                         ▼
                       Fleet
</pre>

Typical use cases:

- Recommendation systems.
- Fraud-ring detection.
- Social networks.
- Traversing complex relationships.

Example: Neo4j.

**Strength:** Relationship traversal is central to the data model.

**Trade-off:** A graph database may be unnecessary overhead for ordinary CRUD applications with straightforward relationships.

---

## 4. SQL vs. NoSQL: Comparison

| Consideration | SQL | NoSQL |
|---|---|---|
| Data model | Relational tables | Key-value, document, wide-column, graph, and others |
| Relationships | Foreign keys and joins | Often modeled through keys, embedding, duplication, or graph edges |
| Schema | Usually explicitly defined | May be flexible, but still needs a data contract |
| Transactions | Strong relational transaction support is common | Capabilities vary; some support multi-record transactions |
| Querying | Rich general-purpose querying | Often optimized around particular access patterns |
| Scaling | Vertical and horizontal strategies, depending on database | Many systems are designed for distributed horizontal scaling |
| Consistency | Depends on isolation, configuration, and topology | Depends on product, configuration, and operation |
| Best fit | Relational data and transactional integrity | Workloads that benefit from a particular non-relational model |

### Important correction: SQL is not simply strong consistency, and NoSQL is not simply eventual consistency

These are separate design decisions.

A relational database can use asynchronous replicas, which may return stale reads. A NoSQL database can provide strong consistency for supported operations.

Always evaluate the specific database's guarantees, topology, and configuration.

**Interview takeaway:** Do not answer “NoSQL is better because it scales.” Explain the workload, required guarantees, access patterns, and operational trade-offs.

---

## 5. Data Modeling: Start With the Domain

Data modeling is the process of deciding what data exists, how it relates, what rules it must satisfy, and how it will be accessed.

A useful process:

1. Identify entities.
2. Identify their attributes.
3. Define relationships and cardinality.
4. Identify business constraints.
5. List the application's important queries.
6. Design tables, documents, keys, and indexes.
7. Validate the model against expected traffic and data growth.

### Example: Ride-booking entities

- **Passenger** — passenger profile and account details.
- **Driver** — driver profile and availability.
- **Ride** — pickup, destination, status, and fare.
- **Assignment** — which driver was assigned to a ride and when.
- **Payment** — payment attempts and outcomes.
- **RideStatusHistory** — status transitions over time.

Relationships might look like this:

<pre>
Passenger 1 ─────── * Ride
                         │
                         ├────── * RideStatusHistory
                         │
                         ├────── * Payment
                         │
                         └────── * Assignment * ─────── 1 Driver
</pre>

Here, `1` means one and `*` means many.

This model allows a ride to have multiple assignment attempts over time, rather than overwriting the entire assignment history whenever a driver changes.

---

## 6. Normalization

Normalization organizes relational data to reduce unnecessary duplication and prevent inconsistent updates.

### Poorly normalized example

Imagine storing this in every ride row:

| ride_id | passenger_name | passenger_email | driver_name | status |
|---|---|---|---|---|
| 501 | Rahul | rahul@example.com | Amit | completed |
| 502 | Rahul | rahul@example.com | Neha | completed |

Passenger details are duplicated across rows.

If Rahul changes his email address, multiple records might need updating. Missing one creates inconsistent data.

### Normalized version

**Passengers**

| passenger_id | name | email |
|---|---|---|
| 101 | Rahul | rahul@example.com |

**Rides**

| ride_id | passenger_id | status |
|---|---|---|
| 501 | 101 | completed |
| 502 | 101 | completed |

Passenger information is stored once, and rides reference it using `passenger_id`.

### The normal forms, conceptually

- **First Normal Form (1NF):** Each field holds an appropriate single value; avoid repeating groups inside a relational row.
- **Second Normal Form (2NF):** Meet 1NF and ensure non-key attributes depend on the whole candidate key, not just part of a composite key.
- **Third Normal Form (3NF):** Meet 2NF and avoid non-key attributes depending transitively on a candidate key.

For most application designs, the practical starting point is to keep each entity's facts in the appropriate table and use constraints to express relationships.

### Benefits

- Fewer inconsistent updates.
- Less unnecessary duplication.
- Better data integrity.
- Clearer ownership of information.

### Trade-offs

Normalization can require joins. If a read-heavy workload repeatedly needs the same combined result, a deliberately denormalized representation may be useful.

---

## 7. Denormalization

Denormalization deliberately duplicates or precomputes data to improve a particular read path.

Suppose the operations dashboard frequently displays:

- Ride ID.
- Passenger name.
- Driver name.
- Ride status.
- Pickup location.

A normalized design may require several joins. A read-optimized representation could store the frequently displayed fields together.

The trade-off is that duplicated values must be updated when their authoritative values change.

### Common approaches

- Store a small snapshot of passenger or driver details on a ride.
- Maintain a precomputed dashboard read model.
- Cache frequently accessed results.
- Create materialized views where appropriate.

### Normalization vs. denormalization

| Normalization | Denormalization |
|---|---|
| Minimizes redundant facts | Intentionally duplicates selected facts |
| Helps maintain consistency | Can improve common read paths |
| Often needs joins | May reduce joins |
| Usually simpler for authoritative writes | Requires a synchronization strategy |

**Rule of thumb:** Normalize authoritative transactional data first. Denormalize when a measured workload justifies the additional complexity.

---

## 8. Model Around Access Patterns

The same business domain can be modeled differently depending on the queries the system must support.

Consider these access patterns:

1. Retrieve a ride by ID.
2. List a passenger's most recent rides.
3. Find available drivers near a pickup location.
4. Retrieve the latest location of a specific driver.
5. Produce daily completed-ride totals.
6. Trace every status transition for a ride.

Each query places different demands on storage.

| Access pattern | Likely design consideration |
|---|---|
| Retrieve a ride by ID | Primary key lookup |
| Passenger ride history | Index by passenger and creation time |
| Find nearby drivers | Geospatial index or specialized location search |
| Latest location by driver ID | Key-value lookup or indexed latest-location table |
| Daily ride totals | Aggregation, materialized view, or analytics store |
| Full status history | Append-oriented history table with time and ride indexes |

The important distinction is that **a data model should support the actual queries, not every hypothetical query equally well**.

For example, if the most common query is “get the latest 20 rides for passenger 101,” a composite index on `(passenger_id, created_at)` may be more useful than an elaborate collection of unrelated indexes.

---

## 9. Polyglot Persistence

Polyglot persistence means using more than one type of data store within the same system, choosing each for a particular responsibility.

A ride-booking architecture could look like this:

<pre>
                    Application Services
                             │
              ┌──────────────┼───────────────┐
              ▼              ▼               ▼
        PostgreSQL          Redis       Search Engine
        Transactions      Latest state  Geospatial/
        and relationships  and cache     text search
              │              │               │
              └──────────────┴───────────────┘
                             │
                      Event / Queue
                             │
                             ▼
                     Analytics Store
</pre>

The diagram is illustrative: not every system needs all these components.

### Example responsibilities

**PostgreSQL**

- Passenger accounts.
- Ride records.
- Driver assignments.
- Payments and transactional history.

**Redis or another key-value store**

- Latest driver location.
- Temporary state.
- Frequently accessed cached values.

Whether Redis is suitable as the authoritative location store depends on durability, persistence, recovery, and data-loss requirements.

**Search engine**

- Complex text search.
- Specialized geospatial search.
- Search ranking and filtering.

Use a separate search engine only when the database's own indexing and geospatial capabilities are insufficient for the requirements.

**Analytics store**

- Historical aggregations.
- Large reporting queries.
- Business intelligence workloads.

### The hidden cost of multiple databases

Every additional store introduces complexity:

- More infrastructure to deploy and monitor.
- More backups and recovery procedures.
- More security policies and credentials.
- More data synchronization.
- More opportunities for stale or missing data.
- More operational knowledge required by the team.

For example, when a driver's location is written to Redis and also published to a search index, those updates may not happen atomically. The system must tolerate temporary divergence or implement a reliable synchronization process.

**Start with the simplest architecture that meets the requirements. Add specialized stores when their benefits justify their operational cost.**

---

## 10. Schema Evolution and Data Lifecycle

Data models change as applications evolve.

A passenger record might initially contain a name and email. Later, the application may add phone verification, preferences, or a new status field.

A safe database migration process usually includes:

1. Make the schema change backward-compatible where possible.
2. Deploy code that can work with the old and new representations.
3. Backfill existing records if necessary.
4. Switch reads and writes to the new representation.
5. Remove obsolete fields only after dependent code no longer needs them.

This is often called an **expand-and-contract migration**.

For example, when renaming `passenger_name` to `display_name`, avoid removing the old column immediately if older application instances still depend on it.

### Other lifecycle questions

- How long must historical records be retained?
- Can old location events be deleted or archived?
- Which data must be recoverable after accidental deletion?
- Are personal data deletion requirements applicable?
- What happens when an event schema changes?
- Can a migration run without blocking important production traffic?

A database design is not complete until it accounts for the evolution and eventual retirement of its data.

---

## 11. Data Modeling and Distributed Systems

Your choice of data model also influences scaling and transaction boundaries.

### Partition keys matter

In a distributed database, the partition key helps determine where records are stored.

A poor key may create a hot partition where one node receives a disproportionate amount of traffic.

For example, partitioning all active rides by a constant status such as `requested` can concentrate writes and queries on a small subset of partitions.

A well-chosen key distributes workload while supporting important queries. There is no universally optimal key: it depends on traffic distribution and access patterns.

### Transaction boundaries matter

Suppose a ride is created in PostgreSQL and an event must be published to a queue.

If the database commits but event publication fails, downstream services may never learn about the new ride.

A transactional outbox can help by committing the ride and an event record in the same database transaction. A separate publisher then delivers the event.

This is not merely a database selection issue; it is a consistency problem across systems.

### CAP theorem preview

When a distributed data store experiences a network partition, certain consistency and availability guarantees cannot all be maintained simultaneously under the classical CAP model.

The details matter, so we will examine CAP and its practical implications in the next lesson.

For now, remember that distributing data across nodes introduces trade-offs around consistency, availability, latency, and recovery.

---

## 12. A Practical Decision Framework

Before selecting SQL or NoSQL, answer these questions in order:

1. **Data relationships:** Are there many relationships and cross-entity queries?
2. **Correctness:** Do operations require atomic updates and strict business constraints?
3. **Access patterns:** Are queries mostly key lookups, document retrievals, joins, or relationship traversals?
4. **Scale:** What are the expected read/write rates, record sizes, and growth?
5. **Consistency:** Can a read return slightly stale data, or must it reflect the latest committed write?
6. **Query flexibility:** Will requirements change frequently, introducing new filters and reports?
7. **Operations:** Can the team operate another database reliably?
8. **Migration and recovery:** How will data be migrated, backed up, restored, and retained?

### Example: Choosing storage for a ride-booking platform

| Requirement | Starting choice | Why |
|---|---|---|
| Create rides and enforce assignments | PostgreSQL | Transactions, constraints, relational queries |
| List passenger ride history | PostgreSQL | Relational model and suitable indexes |
| Read latest driver location | Redis or an indexed database table | Efficient lookup by driver ID |
| Store long-term location history | Partitioned PostgreSQL or a specialized store | Depends on retention, volume, and query patterns |
| Search places and addresses | PostgreSQL geospatial/text search or a dedicated search engine | Choose based on query complexity and performance |
| Generate business reports | PostgreSQL initially; analytics store if needed | Avoid premature infrastructure complexity |

These are starting points, not universal answers. PostgreSQL with appropriate indexing and partitioning may be sufficient for more workloads than expected.

---

## 13. Common Mistakes

- **Choosing NoSQL just because the system is expected to scale.** First identify the actual scaling bottleneck.
- **Assuming every NoSQL database behaves the same way.** Their data models and guarantees vary significantly.
- **Treating flexible schemas as a substitute for data validation.** Application contracts and database validation still matter.
- **Over-normalizing every read path.** Some measured workloads benefit from denormalization.
- **Denormalizing without a synchronization strategy.** Duplicate values can become inconsistent.
- **Adding multiple databases too early.** Operational complexity can exceed the performance benefit.
- **Designing tables without listing important queries.** The resulting model may be difficult to query efficiently.
- **Ignoring retention and schema migrations.** Historical data and evolving schemas are production concerns.
- **Using a cache as the authoritative source without evaluating durability.** A cache may lose data or be unavailable during recovery.
- **Confusing storage technology with architecture.** Choosing PostgreSQL or MongoDB does not automatically solve distributed consistency, scaling, or reliability.

---

## 14. Practical Design Checklist

Before finalizing a data model, verify the following.

### Requirements
- [ ] Entities and relationships are identified.
- [ ] Important read and write operations are listed.
- [ ] Transaction and consistency requirements are explicit.
- [ ] Expected data volume and traffic are estimated.

### Schema and integrity
- [ ] Primary keys are defined.
- [ ] Foreign keys and unique constraints are considered.
- [ ] Business invariants are protected against concurrent writes.
- [ ] Normalization and intentional denormalization are documented.

### Performance and scaling
- [ ] Indexes support important queries.
- [ ] Partitioning is justified by volume, retention, or performance.
- [ ] Partition and shard keys avoid obvious hotspots.
- [ ] Expensive reporting queries are isolated when necessary.

### Operations
- [ ] Backups and restore procedures are defined.
- [ ] Schema migrations are safe for rolling deployments.
- [ ] Retention and deletion policies are established.
- [ ] Additional databases have clear owners and operational plans.
- [ ] Data synchronization and recovery behavior are understood.

---

## 15. Revision Questions

Try answering these without looking back at the notes.

1. What is the fundamental difference between a relational database and a key-value database?
2. Why is NoSQL not automatically more scalable than SQL?
3. When would a document database be a better fit than a relational database?
4. What problems does normalization help prevent?
5. Why might a read-heavy dashboard benefit from denormalization?
6. What is polyglot persistence, and what operational costs does it introduce?
7. Why should data modeling begin with access patterns?
8. When might a search engine be justified alongside PostgreSQL?
9. What is an expand-and-contract database migration?
10. How can a poorly chosen partition key create a hotspot?

---

## 16. Practice Exercise — Design the Data Layer

Design the data layer for a ride-booking application with these requirements:

- Passengers can create rides and view their ride history.
- A ride must not be assigned to two drivers simultaneously.
- Drivers send location updates every few seconds.
- Dispatchers need to find nearby available drivers.
- Operations staff need daily and monthly ride reports.
- The application must tolerate temporary failure of a supporting cache or search service.

Answer these questions:

1. Which data would you store in PostgreSQL, and why?
2. Would you store the latest driver location in PostgreSQL, Redis, or both? Explain the trade-offs.
3. How would you represent rides, drivers, assignments, and ride history?
4. Which indexes would support the passenger ride-history query?
5. Would you add a search engine or analytics database immediately? Explain your decision.
6. How would you prevent stale or duplicated data when synchronizing between stores?
7. What should happen if Redis becomes unavailable?

**Design principle to remember:** Start with the business rules and access patterns, select the simplest data model that meets them, and introduce specialized storage only when there is a clear requirement.


## Next step

Attempt the practice exercise, even briefly. Focus on explaining why you chose each store, what consistency guarantees you need, and what happens when a supporting service fails.