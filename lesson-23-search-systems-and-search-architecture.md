# Lesson 23 — Search Systems: Full-Text Search, Elasticsearch, Indexing, Ranking, and Geospatial Search

Most applications eventually need search functionality. Users may want to search bookings, drivers, passengers, addresses, products, or documents.

A basic SQL query works well initially, but as search requirements grow, we need dedicated search architectures.

In this lesson, we'll explore full-text search, inverted indexes, Elasticsearch, relevance ranking, autocomplete, geospatial search, and synchronization between databases and search engines.

## 1. Why Do We Need a Search System?

Imagine our ride-booking platform has millions of bookings, thousands of drivers, and hundreds of thousands of passengers.

The operations team needs to search for:

- A passenger by name or phone number.
- A booking using its reference number.
- A driver by name or vehicle registration.
- Trips using pickup and drop-off addresses.
- Bookings created within a date range.
- Drivers available near a particular location.

Initially, we might use SQL:

```sql
SELECT *
FROM passengers
WHERE name LIKE '%john%';
```

This can work for small datasets, but it becomes less suitable as search requirements become more sophisticated.

Common challenges include:

1. Searching across multiple fields.
2. Handling typos and partial words.
3. Ranking results by relevance.
4. Supporting autocomplete.
5. Searching millions of records quickly.
6. Searching by geographic distance.
7. Filtering and sorting search results efficiently.

A dedicated search engine is designed to solve many of these problems.

---

## 2. Database Search vs Dedicated Search Engine

Let's compare PostgreSQL with a dedicated search engine such as Elasticsearch.

| Feature | PostgreSQL | Elasticsearch |
|---|---|---|
| Primary purpose | Transactional data management | Search and analytics |
| Exact matching | Excellent | Excellent |
| Full-text search | Supported | Core capability |
| Fuzzy matching | Available with extensions and techniques | Built-in query capabilities |
| Relevance ranking | Supported | Advanced relevance features |
| Geospatial search | PostGIS | Geo queries |
| Transactions | ACID transactions | Not a replacement for relational transactions |
| Data consistency | Transactional guarantees | Near-real-time search visibility |

### When Should We Use PostgreSQL?

PostgreSQL is often sufficient when:
- Dataset size is manageable.
- Search requirements are simple.
- Exact matching is enough.
- Full-text search requirements are moderate.
- We want to avoid additional infrastructure.

PostgreSQL provides full-text search through features such as:
- `tsvector`
- `tsquery`
- GIN indexes
- Ranking functions

### When Should We Consider Elasticsearch?

Consider a dedicated search engine when:
- Search is a major product feature.
- We need advanced relevance ranking.
- We need sophisticated filtering and aggregations.
- We need fuzzy matching and autocomplete at scale.
- Search workloads should be isolated from transactional database workloads.

**Important:** A search engine should not automatically replace the primary database.

PostgreSQL can remain the source of truth while Elasticsearch serves search queries.

---

## 3. How Search Engines Work

To understand search engines, we need to understand indexing.

Imagine we have these documents:

```text
Document 1:
John Smith is a passenger.

Document 2:
John Williams is a driver.

Document 3:
Sarah Smith is a passenger.
```

A search engine analyzes documents and builds an index.

### Inverted Index

An inverted index maps terms to the documents containing them.

Simplified example:

| Term | Document IDs |
|---|---|
| John | 1, 2 |
| Smith | 1, 3 |
| Williams | 2 |
| Sarah | 3 |
| passenger | 1, 3 |
| driver | 2 |

When someone searches for `John`, the engine can quickly find documents 1 and 2.

Without an index, the system may need to examine every document.

### Why Is It Called an Inverted Index?

A normal document structure looks like:

```text
Document 1 -> John, Smith, Passenger
Document 2 -> John, Williams, Driver
Document 3 -> Sarah, Smith, Passenger
```

An inverted index reverses the relationship:

```text
John      -> Document 1, Document 2
Smith     -> Document 1, Document 3
Passenger -> Document 1, Document 3
Driver    -> Document 2
```

This is the foundation of many full-text search engines.

---

## 4. Text Analysis: Tokenization and Normalization

Search engines do not necessarily index raw text exactly as it appears.

They process text through an analyzer.

An analyzer commonly includes:

1. Character filters.
2. Tokenizer.
3. Token filters.

Example:

Original text:

```text
The Drivers are driving in New Delhi!
```

After basic normalization and tokenization:

```text
drivers
are
driving
in
new
delhi
```

Depending on the analyzer, stop words such as `are` and `in` may be removed, and stemming or lemmatization may normalize word forms.

### Important Concepts

**Tokenization:** Splitting text into searchable terms.

**Lowercasing:** Converting terms to lowercase.

**Stop-word removal:** Removing common words that add little search value.

**Stemming:** Reducing words to a common root form.

**Lemmatization:** Reducing words to a linguistically meaningful base form.

**Synonyms:** Treating related terms as equivalent for search purposes.

Example:

```text
car -> automobile
```

A search for `car` may also find documents containing `automobile`, depending on the synonym configuration.

### Why Does Analysis Matter?

Consider a user searching for:

```text
DRIVER
```

But the stored text is:

```text
Driver
```

Normalization allows the search engine to treat them consistently.

The analyzer must be designed according to the language and search experience we want to support.

---

## 5. Full-Text Search

Full-text search allows users to search for meaningful words or phrases within text fields.

Example:

A dispatcher searches:

```text
airport pickup john
```

The system searches relevant fields:
- Passenger name.
- Pickup address.
- Drop-off address.
- Booking reference.
- Notes, where appropriate.

A search engine can tokenize the query and match documents based on terms and relevance.

### Exact Search vs Full-Text Search

| Exact Search | Full-Text Search |
|---|---|
| Matches exact values or terms | Matches analyzed text |
| Useful for IDs and reference numbers | Useful for names, descriptions, and addresses |
| Usually predictable | May involve ranking |
| Example: Booking ID `B123` | Example: `airport pickup` |

Do not treat all fields the same.

For example:
- Booking reference: exact match.
- Phone number: normalized exact or prefix matching.
- Passenger name: text search.
- Pickup address: text and possibly geospatial search.

Different fields need different indexing and query strategies.

---

## 6. Relevance Ranking

Search engines often return many matching documents.

How do we decide which results appear first?

This is the problem of relevance ranking.

For example, a dispatcher searches:

```text
John Smith
```

Possible results:

1. John Smith
2. John Smithson
3. John Williams
4. Jonathan Smith

The exact full-name match should usually appear first.

Search engines use scoring mechanisms to rank matching documents.

### TF-IDF

TF-IDF stands for Term Frequency–Inverse Document Frequency.

It considers:
- How frequently a term appears in a document.
- How common the term is across all documents.

A term that appears frequently in one document but rarely across the collection can be more informative.

### BM25

BM25 is a widely used relevance-ranking algorithm.

It builds on term frequency and inverse document frequency while accounting for document length.

A simplified intuition:

- Matching important query terms increases relevance.
- Rare terms often contribute more than common terms.
- Repeating a term helps, but with diminishing returns.
- Very long documents are normalized so they do not automatically rank higher.

Elasticsearch uses BM25 by default for many text fields.

### Business-Level Ranking

We can also combine text relevance with business rules.

For example:

```text
Final Score =
Text Relevance
+ Exact Name Match Boost
+ Booking Reference Match Boost
+ Recency Boost
```

The weights should be tested against actual search behavior.

Do not blindly add arbitrary boosts. Poor ranking can make search results confusing even when the underlying matches are correct.

---

## 7. Fuzzy Search and Typo Tolerance

Users frequently make spelling mistakes.

For example:

```text
User searches: Jonh
Actual name:   John
```

An exact search may return no results.

Fuzzy search can match terms that are similar.

### Levenshtein Distance

Levenshtein distance measures the minimum number of single-character edits needed to transform one string into another.

Operations:
- Insertion.
- Deletion.
- Substitution.

Example:

```text
John
Jonh
```

Two character positions need to be transformed using substitutions, so the distance is 2.

Fuzzy matching can use edit distance to identify approximate matches.

### When to Use Fuzzy Search

Useful for:
- Passenger names.
- Driver names.
- Addresses.
- Product names.
- General text search.

Usually inappropriate for:
- Payment IDs.
- Booking references.
- Phone numbers.
- Security-sensitive identifiers.

Fuzzy matching can increase query cost and return unexpected results. Apply it selectively.

---

## 8. Autocomplete and Search Suggestions

Autocomplete provides suggestions while a user types.

Example:

```text
User types: New D

Suggestions:
- New Delhi
- New Delhi Airport
- New Delhi Railway Station
```

### Common Approaches

**Prefix matching**

Matches terms beginning with the entered text.

```text
New D -> New Delhi
```

**Edge n-grams**

An analyzer generates prefixes for tokens.

For example:

```text
Delhi -> D, De, Del, Delh, Delhi
```

This makes prefix matching efficient, although it increases index size.

**Completion suggester**

Elasticsearch provides a completion suggester designed for fast autocomplete.

**Search-as-you-type**

A search strategy that supports incremental matching as users type.

### Autocomplete Architecture

```text
Client
  |
  | User types
  v
Search API
  |
  v
Search Engine
  |
  | Top suggestions
  v
Client
```

### Important Optimizations

- Debounce requests.
- Set a minimum query length.
- Limit the number of suggestions.
- Cancel or ignore stale requests.
- Cache common suggestions where appropriate.
- Avoid triggering a full search on every keystroke.

For example, a 250ms debounce waits briefly after typing stops before sending a request.

---

## 9. Filtering, Sorting, and Faceted Search

Search often combines text matching with filters.

Example:

A dispatcher searches for:

```text
John
```

And applies filters:

```text
Status = ASSIGNED
Service Area = Delhi
Date = Today
```

The search engine returns matching records.

### Filtering

Filters determine whether a document qualifies.

Examples:
- Booking status.
- Tenant ID.
- Service area.
- Date range.
- Driver availability.

### Sorting

Sorting determines the order of matching results.

Examples:
- Newest booking first.
- Highest relevance first.
- Nearest driver first.

### Faceted Search

Faceted search provides aggregated filter options.

Example:

```text
Search Results: 1,200

Status:
  Assigned: 500
  Completed: 400
  Pending: 200
  Cancelled: 100
```

Users can click a status to narrow the results.

Search engines can calculate these aggregations efficiently, depending on the index and query.

**Important:** Security and tenant filters must always be enforced server-side. Never rely on the client to provide the correct tenant or authorization scope.

---

## 10. Elasticsearch Architecture

Elasticsearch is a distributed search and analytics engine built around indexed documents.

Important concepts:

### Cluster

A collection of Elasticsearch nodes working together.

### Node

A server participating in the cluster.

### Index

A logical collection of documents.

For example:

```text
bookings
passengers
drivers
```

### Document

A JSON record stored in an index.

Example:

```json
{
  "bookingId": "B123",
  "passengerName": "John Smith",
  "status": "ASSIGNED",
  "pickupCity": "Delhi"
}
```

### Shard

An index is divided into shards.

Shards allow data and search work to be distributed across nodes.

### Replica

A replica shard is a copy of a primary shard.

Replicas can improve availability and provide additional search capacity.

### Simplified Architecture

```text
             Search Request
                   |
                   v
              Coordinator
                   |
          +--------+--------+
          |        |        |
          v        v        v
       Shard 1  Shard 2  Shard 3
          |        |        |
          v        v        v
        Results  Results  Results
          |        |        |
          +--------+--------+
                   |
                   v
             Merge Results
                   |
                   v
              Search Response
```

The coordinator distributes search work and merges results.

### Primary Shards and Replicas

Suppose an index has:

```text
3 Primary Shards
1 Replica per Primary Shard
```

Total shard copies:

```text
3 × (1 + 1) = 6 shard copies
```

Primary and replica shards should be allocated appropriately across nodes to avoid losing availability when a node fails.

Adding more shards is not always better. Every shard has memory, file-handle, and coordination overhead.

Choose shard sizes and counts based on expected data volume, query workload, and cluster capacity.

---

## 11. Database and Search Engine Synchronization

This is one of the most important design challenges.

Suppose PostgreSQL stores booking data and Elasticsearch indexes it for search.

What happens when a booking changes?

We need to update both systems.

A naive approach:

```text
Booking Service
   |
   +------> PostgreSQL
   |
   +------> Elasticsearch
```

This creates a consistency problem.

Imagine:
1. PostgreSQL update succeeds.
2. Elasticsearch update fails.

Now the database and search index contain different information.

### Recommended Architecture: Transactional Outbox

Use the transactional outbox pattern.

```text
Booking Service
      |
      v
PostgreSQL Transaction
      |
      +----> Update Booking
      |
      +----> Insert Outbox Event
                    |
                    v
              Outbox Publisher
                    |
                    v
                Kafka
                    |
                    v
             Search Consumer
                    |
                    v
              Elasticsearch
```

The booking update and outbox event are written in the same database transaction.

A publisher sends the event to Kafka. A consumer updates Elasticsearch.

### Why Is This Better?

- Avoids the database-to-broker dual-write gap.
- Supports retries.
- Keeps search indexing asynchronous.
- Allows consumers to recover after failure.

### Eventual Consistency

PostgreSQL is the source of truth.

Elasticsearch is a derived search index.

A user may update a booking and not see the new value in search immediately.

This is eventual consistency.

The system should define an acceptable indexing delay.

### Idempotent Search Consumers

Events can be delivered more than once.

Consumers should safely handle duplicates.

Common approaches:
- Upsert documents.
- Use stable document IDs.
- Track entity versions.
- Reject stale updates.

For example, if booking version 5 has already been indexed, a delayed version 4 event should not overwrite it.

---

## 12. Geospatial Search

Many systems need location-based search.

Our ride-booking platform needs to find drivers near a passenger's pickup location.

Example:

```text
Find available drivers within 5 km
of latitude 28.6139, longitude 77.2090
```

### Geospatial Concepts

**Latitude:** North-south position.

**Longitude:** East-west position.

**Geographic coordinates:** Represent a location on Earth.

**Radius search:** Find points within a specified distance.

**Bounding box:** Find points within a rectangular geographic area.

### Haversine Formula

The Haversine formula calculates the great-circle distance between two geographic coordinates.

Let:

- \(\phi_1\), \(\phi_2\) be latitudes in radians.
- \(\Delta\phi\) be the latitude difference.
- \(\Delta\lambda\) be the longitude difference.
- \(R\) be Earth's approximate radius.

\[
a =
\sin^2\left(\frac{\Delta\phi}{2}\right)
+
\cos(\phi_1)\cos(\phi_2)
\sin^2\left(\frac{\Delta\lambda}{2}\right)
\]

\[
d = 2R\arcsin(\sqrt{a})
\]

This gives the approximate surface distance between two points.

For production geospatial queries, use a geospatial index rather than calculating the distance against every record.

### Geospatial Indexes

Common options include:
- PostgreSQL with PostGIS.
- Elasticsearch `geo_point`.
- Elasticsearch `geo_shape`.

### Example Driver Search

```text
Input:
  Passenger location
  Radius = 5 km

Filters:
  Driver status = AVAILABLE
  Vehicle type = SEDAN
  Service area = DELHI

Sort:
  Distance ascending
```

The system returns eligible drivers ordered by proximity.

**Important:** The nearest driver is not necessarily the best driver to assign. Availability, route distance, traffic, vehicle type, and dispatch policies may also matter.

---

## 13. Search Pagination

Search results may contain millions of records.

We need pagination to avoid returning everything at once.

### Offset Pagination

Example:

```http
GET /search?page=3&size=20
```

This means:
- Skip 40 records.
- Return the next 20.

Advantages:
- Simple.
- Easy to implement.

Disadvantages:
- Deep offsets can be expensive.
- Results can shift when documents are inserted or updated.
- Search engines may have limits on deep pagination.

### Search-After Pagination

Search-after uses the sort values from the last returned document to retrieve the next page.

Example:

```text
Page 1:
  Records 1–20

Last sort values:
  [2026-10-30T10:00:00Z, B120]

Page 2:
  Search after [2026-10-30T10:00:00Z, B120]
```

Advantages:
- Better for deep pagination.
- Avoids repeatedly skipping large numbers of results.

Use a stable sort with a unique tie-breaker. For consistent multi-page views, consider a point-in-time search context where supported.

---

## 14. Search Performance Optimization

Search performance depends on:
- Index design.
- Query complexity.
- Shard count and size.
- Data distribution.
- Hardware.
- Cache behavior.
- Concurrency.

### Practical Optimizations

1. Index only fields needed for search.
2. Use keyword fields for exact filtering.
3. Use text fields for analyzed full-text search.
4. Avoid unnecessarily expensive wildcard queries.
5. Use pagination.
6. Limit returned fields.
7. Use filters for structured constraints.
8. Avoid excessive shard counts.
9. Monitor slow queries.
10. Test realistic query patterns and data volumes.

### Text vs Keyword Fields

A common Elasticsearch mapping distinction:

| Field Type | Purpose |
|---|---|
| `text` | Analyzed full-text search |
| `keyword` | Exact matching, sorting, and aggregations |
| `date` | Date and time queries |
| `integer` / `long` | Numeric values |
| `geo_point` | Geographic coordinates |

For a passenger name, we might need both analyzed search and exact matching.

A multi-field mapping can support both use cases.

---

## 15. Search Security and Multi-Tenancy

Suppose our platform supports multiple transportation companies.

Tenant A must never see Tenant B's bookings.

Search queries must enforce tenant isolation.

Example:

```text
Search:
  passengerName = "John"

Mandatory filters:
  tenantId = authenticatedTenantId
```

The tenant ID must come from trusted authentication context, not an arbitrary client-supplied value.

Other important security considerations:

- Restrict access to the search cluster.
- Use authenticated service connections.
- Apply least-privilege index permissions.
- Avoid exposing internal cluster endpoints.
- Avoid indexing unnecessary sensitive data.
- Protect private fields from search responses.
- Monitor unusual search activity.

A search engine is another data store from a security perspective. It needs the same seriousness as the primary database.

---

## 16. Monitoring a Search System

Useful metrics include:

### Search API
- Requests per second.
- P50, P95, P99 latency.
- Error rate.
- Timeout rate.

### Search Cluster
- CPU utilization.
- Heap and memory usage.
- Disk usage.
- Node availability.
- Shard allocation.
- Search thread-pool rejections.

### Indexing
- Documents indexed per second.
- Indexing failures.
- Indexing latency.
- Refresh latency.
- Queue backlog.

### Data Consistency
- Outbox backlog.
- Kafka consumer lag.
- Time between database update and search visibility.
- Failed indexing events.

### Search Quality
- Zero-result search percentage.
- Click-through rate.
- Search abandonment.
- Percentage of queries returning relevant results.

A technically fast search engine can still provide a poor user experience if ranking and matching quality are bad.

---

## 17. Worked Example — Dispatcher Search

Suppose a dispatcher searches for:

```text
John Delhi
```

The platform needs to find bookings matching:

- Passenger name.
- Driver name.
- Pickup city.
- Drop-off city.
- Booking reference.

Additional filters:
- Tenant ID.
- Service area.
- Booking status.
- Date range.

### Architecture

```text
Dispatcher Portal
       |
       v
Search API
       |
       v
Authorization
       |
       v
Elasticsearch
       |
       +----> Search passenger name
       |
       +----> Search driver name
       |
       +----> Search pickup/drop-off
       |
       +----> Apply filters
       |
       v
Rank and Merge Results
       |
       v
Return Results
```

### Indexing Flow

```text
PostgreSQL
    |
    v
Transactional Outbox
    |
    v
Kafka
    |
    v
Search Index Consumer
    |
    v
Elasticsearch
```

### Ranking Strategy

A possible ranking order:

1. Exact booking reference.
2. Exact passenger name.
3. Exact driver name.
4. Partial passenger or driver name.
5. Pickup or drop-off text match.

This is a product decision and should be validated against real dispatcher behavior.

### Important Design Decision

If the dispatcher opens a booking, the Booking Service should still retrieve the authoritative record from PostgreSQL and enforce authorization.

Elasticsearch provides search results; it should not become the source of truth for transactional booking operations.

---

## 18. Common Search Architecture Mistakes

1. Using a dedicated search engine for every simple SQL search.
2. Treating Elasticsearch as the primary transactional database.
3. Updating PostgreSQL and Elasticsearch without handling dual-write failures.
4. Forgetting that search indexes are eventually consistent.
5. Indexing unnecessary sensitive data.
6. Using fuzzy search on identifiers.
7. Creating too many shards.
8. Using deep offset pagination.
9. Ignoring relevance quality.
10. Forgetting tenant-level search filters.
11. Assuming the nearest geographic point is the best business result.
12. Monitoring latency without monitoring indexing lag.

---

## 19. Revision Questions

1. What is the difference between database search and dedicated search engines?
2. What is an inverted index?
3. Explain tokenization and normalization.
4. What is full-text search?
5. What is the difference between exact matching and fuzzy matching?
6. What is TF-IDF?
7. What is BM25?
8. How does autocomplete work?
9. What are edge n-grams?
10. What is faceted search?
11. Explain cluster, node, index, document, shard, and replica.
12. Why do we use replicas in Elasticsearch?
13. Why should PostgreSQL remain the source of truth?
14. How does the transactional outbox pattern synchronize a search index?
15. What is eventual consistency in search?
16. Why should search consumers be idempotent?
17. What is geospatial search?
18. Explain the Haversine formula.
19. What is the difference between offset pagination and search-after?
20. How would you enforce tenant isolation in search?

---

## 20. Practical Exercise — Design a Search System for Ride Booking

Design a search architecture for an operations dashboard.

Requirements:

- Search passengers.
- Search drivers.
- Search bookings.
- Search by booking reference.
- Search pickup and drop-off addresses.
- Support typo tolerance for names.
- Support autocomplete.
- Filter by tenant, service area, status, and date.
- Find available drivers within a geographic radius.

Answer these questions:

1. Would you use PostgreSQL, Elasticsearch, or both?
2. Which fields should be indexed as text?
3. Which fields should be indexed as keyword?
4. How would you implement autocomplete?
5. How would you rank exact matches above partial matches?
6. How would you synchronize PostgreSQL and Elasticsearch?
7. What happens if Kafka is unavailable?
8. How would you prevent stale events from overwriting newer search documents?
9. How would you enforce tenant isolation?
10. How would you find the nearest available drivers?
11. Which pagination strategy would you choose?
12. What metrics would you monitor?

### Final Takeaway

A good search architecture separates responsibilities:

- **PostgreSQL:** Source of truth and transactional data.
- **Search Engine:** Fast, flexible querying and relevance ranking.
- **Kafka / Message Broker:** Asynchronous index synchronization.
- **Search API:** Authorization, filtering, ranking, and pagination.
- **Geospatial Index:** Efficient location-based searches.

The goal is not simply to return results quickly. It is to return the correct, relevant, authorized results quickly.

### Practice focus

Design the dispatcher search experience. Focus on the PostgreSQL-to-Elasticsearch synchronization flow, relevance ranking, and tenant isolation.