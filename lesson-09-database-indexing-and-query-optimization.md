# Lesson 9: Database Indexing and Query Optimization

## 1. Why Database Performance Matters

Imagine a ride-booking application with 10 million historical rides.

A user opens their ride history and requests the latest 20 rides.

A poorly designed query might scan millions of rows before finding those 20 records.

A well-designed query with an appropriate index might locate the relevant records quickly and retrieve only the required rows.

The difference affects:

- API response latency.
- Database CPU utilization.
- Disk I/O.
- Memory consumption.
- Concurrent request capacity.
- Infrastructure costs.
- Overall system scalability.

**Key principle:** Before scaling a database, understand whether the query is doing unnecessary work.

Adding more database resources can help, but it does not automatically fix inefficient queries.

---

## 2. What Is a Database Index?

A database index is a data structure that helps the database locate rows without examining every row in a table.

Think of a book with an index at the back.

Without the index, finding a topic might require reading every page. With the index, you can jump to the relevant page.

A database index serves a similar purpose.

### Without an index

Suppose we have this table:

`rides`

| ride_id | passenger_id | status | created_at |
|---|---|---|---|
| 101 | P10 | completed | 2026-10-20 |
| 102 | P20 | requested | 2026-10-21 |
| 103 | P10 | completed | 2026-10-22 |
| 104 | P30 | cancelled | 2026-10-23 |
| 105 | P10 | requested | 2026-10-24 |

Query:

```sql
SELECT *
FROM rides
WHERE passenger_id = 'P10';
```

Without a useful index, PostgreSQL may perform a sequential scan: examine rows in the table and check whether each row matches.

For a table with millions of rows, this can require substantial work.

### With an index

```sql
CREATE INDEX idx_rides_passenger_id
ON rides (passenger_id);
```

The database can use the index to find matching entries and then retrieve the corresponding table rows.

An index typically stores searchable key values and references to table rows. It is separate from the table's full row data, although some index types and designs can store additional information.

### Important qualification

An index does not guarantee that the database will use it. The query planner estimates the cost of different execution plans and chooses one based on statistics, query structure, table size, and other factors.

For a small table, a sequential scan may be faster than using an index.

---

## 3. How Indexes Work Internally

PostgreSQL commonly uses a B-tree for ordinary indexes.

A simplified representation:

```text
                  [ 40 | 70 ]
                  /    |    \
                 /     |     \
                v      v      v
           [10,20]  [40,50]  [70,80,90]
```

This is an illustrative diagram, not an exact representation of PostgreSQL's internal page layout.

The database traverses the tree to find the relevant key range instead of examining every indexed value.

### Why B-trees are useful

B-tree indexes support:

- Equality searches: `WHERE passenger_id = 'P10'`
- Range searches: `WHERE created_at >= ...`
- Sorting: `ORDER BY created_at`
- Prefix conditions on composite indexes, subject to index structure and query conditions.

B-trees are balanced, so searching remains efficient as the number of entries grows.

For many ordinary indexed lookups, the number of tree levels grows approximately logarithmically with the number of entries. Actual query cost also includes page access, row retrieval, filtering, and other work.

### Common index types in PostgreSQL

| Index type | Typical use |
|---|---|
| B-tree | Equality, range queries, and ordering |
| GIN | Inverted indexes for arrays, JSONB containment, and full-text search |
| GiST | Spatial and other extensible search operations |
| BRIN | Very large tables whose values correlate with physical row order |
| Hash | Equality comparisons, with a narrower range of use cases than B-tree |

Choose an index based on the operators and access patterns the query requires, not simply because a particular index type sounds faster.

---

## 4. Full Table Scan vs. Index Scan

Consider this query:

```sql
SELECT *
FROM rides
WHERE passenger_id = 'P10';
```

### Sequential scan

```text
Read table pages
      |
      v
Check each row
      |
      v
Return matching rows
```

A sequential scan is often reasonable when the database must read a large portion of the table.

### Index scan

```text
Search index
      |
      v
Find matching row references
      |
      v
Read relevant table rows
      |
      v
Return results
```

An index scan is often beneficial when a query retrieves a relatively small subset of rows.

### Why an index scan can still be slow

Suppose the query matches 40% of the table. An index scan may require many separate table-page accesses. A sequential scan can be cheaper because it reads pages more efficiently.

Therefore, an index is not automatically the fastest option for every query.

The correct question is not:

"Does the table have an index?"

It is:

"Does the execution plan access the necessary data efficiently for this query and data distribution?"

---

## 5. Selectivity and Cardinality

Two concepts help explain whether an index is useful.

**Selectivity** describes how narrowly a condition filters the data.

**Cardinality** describes the number of distinct values in a column, or the number of rows in a result set, depending on the context.

Consider a table with one million rides.

| Condition | Approximate matching rows | Selectivity |
|---|---:|---|
| `ride_id = 123` | 1 | Very selective |
| `passenger_id = 'P10'` | 100 | Selective |
| `status = 'completed'` | 700,000 | Not very selective |

An index on `ride_id` is likely useful because the condition identifies very few rows.

An index on `status` may still be useful for some queries, especially when a value is rare or the index supports other parts of the query. But if most rows match, PostgreSQL may prefer a sequential scan.

### Important distinction

A column with low cardinality is not automatically a bad index candidate.

For example, a status column might have only five distinct values, but one particular status could be rare. A partial index on that status might be very useful.

Always consider the actual data distribution and query workload.

---

## 6. Composite Indexes

A composite index contains multiple columns.

Suppose the application frequently runs:

```sql
SELECT *
FROM rides
WHERE passenger_id = 'P10'
ORDER BY created_at DESC
LIMIT 20;
```

A possible index is:

```sql
CREATE INDEX idx_rides_passenger_created
ON rides (passenger_id, created_at DESC);
```

This index can support finding a passenger's rides and retrieving them in creation-time order.

Depending on the query plan and selected columns, PostgreSQL may be able to avoid a separate sorting step and reduce how many rows it needs to examine.

### Why column order matters

Consider this index:

```sql
CREATE INDEX idx_rides_passenger_status
ON rides (passenger_id, status);
```

It is generally most useful when a query constrains the leading column, `passenger_id`, or uses suitable conditions on the leading columns.

It can help with:

```sql
WHERE passenger_id = 'P10'
```

and:

```sql
WHERE passenger_id = 'P10'
  AND status = 'completed'
```

It generally is not the ideal index for a query filtering only on `status`:

```sql
WHERE status = 'completed'
```

PostgreSQL can sometimes use a multicolumn index in other ways, but you should not assume that a composite index is equally effective for every column independently.

### A practical column-order guideline

For common B-tree composite indexes:

1. Identify the query's equality conditions.
2. Identify range conditions.
3. Consider the required ordering.
4. Consider selectivity and the actual workload.
5. Verify the plan with representative data.

Equality columns are often placed before range or ordering columns, but this is a guideline, not an absolute rule.

The right order depends on the complete set of queries the index must support.

---

## 7. Indexes for Sorting and Pagination

Consider:

```sql
SELECT *
FROM rides
WHERE passenger_id = 'P10'
ORDER BY created_at DESC
LIMIT 20;
```

Without a suitable index, the database might need to find matching rows, sort them, and then return 20.

An index on `(passenger_id, created_at DESC)` may allow PostgreSQL to retrieve the newest matching rides in index order and stop once it has enough rows.

### Offset pagination

A common approach is:

```sql
SELECT *
FROM rides
WHERE passenger_id = 'P10'
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000;
```

Even with an appropriate index, the database may need to walk past a large number of entries to reach the requested offset.

Large offsets can become increasingly expensive.

### Keyset pagination

An alternative is to use the last-seen ordering value as a cursor.

For example, if `created_at` is not unique, use both `created_at` and a unique tie-breaker such as `ride_id`.

```sql
SELECT *
FROM rides
WHERE passenger_id = 'P10'
  AND (created_at, ride_id) < (:last_created_at, :last_ride_id)
ORDER BY created_at DESC, ride_id DESC
LIMIT 20;
```

A matching index might be:

```sql
CREATE INDEX idx_rides_passenger_created_id
ON rides (passenger_id, created_at DESC, ride_id DESC);
```

The comparison direction and cursor values must match the chosen sort order.

Keyset pagination can avoid walking past thousands of rows for later pages. It also gives more predictable performance for large datasets.

However, it is not identical to offset pagination: jumping directly to an arbitrary page is less convenient, and the behavior under concurrent inserts depends on the cursor and consistency requirements.

---

## 8. Covering Indexes and Index-Only Scans

A normal index may identify the matching rows, but PostgreSQL may still need to read the table to obtain the requested columns.

Suppose the application runs:

```sql
SELECT ride_id, created_at
FROM rides
WHERE passenger_id = 'P10'
ORDER BY created_at DESC
LIMIT 20;
```

A suitable index may support an index-only scan, where the required values can be obtained from the index without fetching every matching table row.

In PostgreSQL, an index-only scan also depends on visibility information. The database may still need heap visits for pages whose visibility cannot be established from the visibility map.

You can use `INCLUDE` to store additional non-key columns in a B-tree index:

```sql
CREATE INDEX idx_rides_passenger_created_cover
ON rides (passenger_id, created_at DESC)
INCLUDE (ride_id);
```

The included column can be returned from the index without participating in index ordering.

### Trade-offs

Covering indexes can reduce table reads, but they:

- Consume additional storage.
- Increase index maintenance work on writes.
- Can increase cache pressure.
- May provide little benefit if the query is already inexpensive.

Create them for demonstrated workload needs rather than adding every selected column to every index.

---

## 9. Partial Indexes

A partial index contains only rows that satisfy a specified predicate.

Imagine that most rides are completed, but the application frequently searches for active rides.

A partial index could be:

```sql
CREATE INDEX idx_active_rides_created
ON rides (created_at DESC)
WHERE status IN ('requested', 'accepted', 'in_progress');
```

This index is smaller than an equivalent full-table index if active rides make up a small fraction of the table.

It can be useful for queries whose predicates match the partial-index condition.

### Example query

```sql
SELECT *
FROM rides
WHERE status = 'accepted'
ORDER BY created_at DESC;
```

The planner may use the partial index if it can establish that the query's conditions are compatible with the index predicate.

### Trade-offs

- Smaller index and potentially fewer pages to search.
- Less maintenance for rows outside the predicate.
- More complex index management.
- The query must satisfy the predicate in a way the planner can prove.
- Updates that move rows into or out of the indexed subset require index maintenance.

Partial indexes are especially useful when a workload repeatedly accesses a small, well-defined subset of a large table.

---

## 10. Why Too Many Indexes Are a Problem

Indexes speed up certain reads, but they are not free.

Every additional index creates maintenance work when indexed data changes.

For example:

```sql
INSERT INTO rides (...)
VALUES (...);
```

PostgreSQL must maintain the table and its applicable indexes.

Similarly, updates to indexed columns may require index changes. Deletes also create maintenance work, and cleanup processes may later reclaim space.

### Consequences of excessive indexing

- Slower inserts and updates.
- More disk space consumption.
- More memory and cache pressure.
- Additional vacuum and maintenance work.
- More complicated schema changes.
- More index pages to back up and restore.

### Example

Imagine a table with ten indexes, many of which serve rarely used queries.

Adding an eleventh index to improve one read query might be a poor trade-off if the application performs thousands of writes per second.

A better approach is to evaluate the workload:

- Which queries are frequent?
- Which queries are slow?
- Which indexes are actually used?
- Which indexes overlap?
- What is the write overhead?
- Does the index materially improve the target query?

An index should justify its cost.

---

## 11. Query Optimization: Start With the Execution Plan

When a query is slow, do not guess which index it needs.

Start by examining the execution plan.

PostgreSQL provides:

```sql
EXPLAIN
SELECT *
FROM rides
WHERE passenger_id = 'P10';
```

This shows the planner's estimated execution plan.

To execute the query and inspect actual timing and row counts:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM rides
WHERE passenger_id = 'P10';
```

`ANALYZE` actually executes the statement. Use caution with statements that modify data, because `EXPLAIN ANALYZE` executes them too.

### What should you inspect?

**1. Scan type**

Is PostgreSQL using a sequential scan, index scan, or index-only scan?

A sequential scan is not automatically a problem.

**2. Estimated rows vs. actual rows**

If the planner estimates 100 rows but the query actually produces 100,000, its cost estimates may be inaccurate.

Outdated statistics or complex data distributions may contribute to the mismatch.

**3. Execution time**

Find which operations consume significant time. Total query time alone may not explain the cause.

**4. Buffer activity**

`BUFFERS` reports information about buffer access, including shared-buffer hits and reads. Interpret it alongside the execution plan and system-level metrics.

**5. Sorts and joins**

Large sorts, repeated nested-loop lookups, or unexpectedly expensive joins may be responsible for the slowdown.

**6. Loops**

An operation executed thousands of times can be expensive even if each individual execution appears quick.

### Important caution

`EXPLAIN ANALYZE` measures execution under the conditions in which you run it. A query may behave differently under production concurrency, cache conditions, and data volume.

---

## 12. Common Query Optimization Mistakes

### Mistake 1: Using `SELECT *` unnecessarily

Instead of:

```sql
SELECT *
FROM rides
WHERE passenger_id = 'P10';
```

Prefer selecting only the required columns when appropriate:

```sql
SELECT ride_id, status, created_at
FROM rides
WHERE passenger_id = 'P10';
```

This can reduce data transfer and memory work and may allow a suitable index-only scan. It does not guarantee one.

### Mistake 2: Applying functions to indexed columns

Consider:

```sql
SELECT *
FROM rides
WHERE DATE(created_at) = DATE '2026-10-20';
```

Applying a function to the indexed column can prevent a normal index on `created_at` from being used as a straightforward range lookup.

A range condition is often preferable:

```sql
SELECT *
FROM rides
WHERE created_at >= TIMESTAMP '2026-10-20 00:00:00'
  AND created_at <  TIMESTAMP '2026-10-21 00:00:00';
```

This example assumes `created_at` is a timestamp without time zone and the intended date boundaries follow that column's semantics. For `timestamptz`, define the intended time zone explicitly.

### Mistake 3: Assuming every filter needs an index

An index may be wasteful if a query frequently reads a large proportion of the table.

Measure the actual workload and compare plans.

### Mistake 4: Ignoring query patterns

Indexes should be designed around real filters, joins, sorting, and pagination.

### Mistake 5: Ignoring data distribution

An index useful for one tenant or status value may behave differently for another if the data distribution is uneven.

### Mistake 6: Optimizing before measuring

A query that looks complicated is not necessarily the bottleneck. Measure query latency, execution plans, rows examined, and system-level behavior first.

---

## 13. A Practical Investigation Workflow

Suppose users report that the ride-history page has become slow.

Use this workflow:

### Step 1: Identify the affected request

Determine which API endpoint is slow and whether the latency comes from the database or another layer.

### Step 2: Find the actual SQL query

Identify the SQL executed by the endpoint and the parameters used in representative requests.

### Step 3: Check the data volume

How many rides exist? How many rows match the passenger? How many rows are returned?

### Step 4: Inspect the execution plan

Use `EXPLAIN` and, in a safe environment, `EXPLAIN ANALYZE`.

### Step 5: Check index suitability

Does an existing index match the filtering and ordering requirements? Would a composite or partial index help?

### Step 6: Review query structure

Check for unnecessary columns, expensive sorts, inefficient joins, repeated queries, and pagination issues.

### Step 7: Check database statistics and contention

Outdated statistics, lock waits, connection-pool saturation, and storage latency can make an otherwise reasonable query slow.

### Step 8: Test the proposed change

Use representative data and compare latency, buffer activity, and write overhead.

### Step 9: Monitor after deployment

Verify that the query improved in production and that other workloads did not regress.

The goal is to reduce unnecessary work, not simply to increase the number of indexes.

---

## 14. Worked Example

Assume the rides table contains 20 million records.

The application frequently executes:

```sql
SELECT ride_id, status, created_at
FROM rides
WHERE passenger_id = 'P10'
ORDER BY created_at DESC
LIMIT 20;
```

### Initial problem

The database uses a sequential scan and examines a large number of rows.

The query returns only 20 rows, but locating them requires much more work.

### Potential improvement

Create a composite index:

```sql
CREATE INDEX idx_rides_passenger_created
ON rides (passenger_id, created_at DESC);
```

### Why this can help

1. PostgreSQL can search for the passenger's index entries.
2. The index can provide rows in the required creation-time order.
3. The query may stop after finding the first 20 matching rows.
4. The database may avoid sorting all matching rows.

The actual result depends on the query plan, the data distribution, and the selected columns.

### What if the query remains slow?

Investigate:

- Whether many rows match the passenger.
- Whether table-page reads dominate execution time.
- Whether the planner estimates are accurate.
- Whether a covering index would help.
- Whether the query is waiting on locks or I/O.
- Whether the database is overloaded by other queries.

Do not assume that adding more indexes is the next step.

---

## 15. Revision Questions

Answer these without looking at the notes.

1. What is a database index, and why can it improve query performance?
2. Why might PostgreSQL choose a sequential scan even when an index exists?
3. What is the difference between selectivity and cardinality?
4. Why does the order of columns matter in a composite index?
5. How can an index support both filtering and sorting?
6. What is the difference between offset pagination and keyset pagination?
7. What is a covering index, and what is PostgreSQL's index-only scan visibility limitation?
8. When is a partial index useful?
9. Why can too many indexes reduce overall system performance?
10. What information does `EXPLAIN ANALYZE` provide?
11. Why can applying a function to an indexed column affect index usage?
12. How would you investigate a slow ride-history query?
13. Why should index decisions be based on actual query patterns?
14. What would you measure before and after adding a composite index?

## Final Takeaway

Indexes are tools for reducing unnecessary data access. Their value depends on the query, the data distribution, and the workload.

Remember these principles:

- Design indexes around real queries.
- Understand selectivity and composite-index ordering.
- Use execution plans instead of guessing.
- Consider pagination and sorting as part of query design.
- Balance read improvements against write and storage costs.
- Measure changes using representative data.

The goal is not to index every column. It is to give the database an efficient way to execute the queries that matter most.


## Practice exercise
Imagine your rides table contains 20 million rows. Your application frequently runs this query:

`SELECT ride_id, status, created_at
FROM rides
WHERE passenger_id = 'P10'
ORDER BY created_at DESC
LIMIT 20;`


Answer these three questions:

1. Which index would you consider adding, and why?
2. Why might a single-column index on passenger_id be less effective than a composite index for this query?
3. After adding the index, which measurements would you compare to confirm the improvement?