# Lesson 5: Caching and Consistency

## 1. Learning Objectives

By the end of this lesson, you should be able to:

- Explain why caching improves system performance.
- Understand cache hits, cache misses, and cache hit rate.
- Identify suitable data for caching.
- Compare cache-aside, read-through, write-through, and write-behind strategies.
- Understand TTL and cache invalidation.
- Recognize cache stampedes, cache penetration, and cache avalanche.
- Explain eventual consistency, bounded staleness, and strong consistency.
- Design caching strategies for frequently changing data.
- Understand what happens when a cache becomes unavailable.
- Decide when a cache such as Redis is justified.

---

## 2. What Is Caching?

Caching stores a copy of data that can be accessed more quickly than retrieving it from the original source.

The original source is commonly a database or another service. The cache is an additional layer that can serve repeated requests without always contacting that source.

For example, a booking platform may repeatedly retrieve the same booking details from PostgreSQL.

Without a cache:

```text
Client
  |
  v
Application API
  |
  v
PostgreSQL
  |
  v
Response
```

With a cache:

```text
Client
  |
  v
Application API
  |
  v
Cache
  |
  +---- Cache hit ----> Return cached data
  |
  +---- Cache miss ---> Read PostgreSQL
                            |
                            v
                       Populate cache
                            |
                            v
                         Response
```

The cache can reduce repeated database work and improve response time.

However, caching introduces another copy of the data that must be managed correctly.

**Key principle:** Caching trades some additional complexity and freshness risk for reduced work and potentially lower latency.

---

## 3. Why Do We Need Caching?

Consider an API that receives 800 requests per second.

Suppose many requests repeatedly read the same booking details.

If every request queries PostgreSQL, the database must handle the repeated work.

If a shared cache can serve some of these requests, PostgreSQL receives fewer reads.

### Benefits

- Lower latency for cache hits.
- Reduced database query volume.
- Reduced load on downstream services.
- Potentially higher effective read throughput.
- Better handling of repeated read-heavy workloads.

### Trade-offs

- Cached data may become stale.
- Cache invalidation can be difficult.
- Additional infrastructure must be monitored.
- Cache misses still require the original data source.
- Cache failure can shift substantial traffic to the database.
- Cache keys, expiration, and memory usage require management.

Caching is not automatically beneficial. Its value depends on the workload, data access patterns, freshness requirements, and the cost of the original operation.

---

## 4. Important Caching Terminology

### Cache hit

The requested value exists in the cache and can be used.

### Cache miss

The requested value is not available in the cache, so the application must retrieve it elsewhere.

### Cache hit rate

The proportion of cache lookups that are hits.

```text
Cache Hit Rate =
    Cache Hits / (Cache Hits + Cache Misses) * 100%
```

Example:

- Cache hits = 800.
- Cache misses = 200.

```text
Hit Rate = 800 / (800 + 200) * 100
         = 80%
```

### Cache miss penalty

The extra cost incurred when the application must retrieve data from the original source.

For example, a cache miss may require a database query that takes 100 ms.

### Cache invalidation

Removing or marking cached data as unusable when it should no longer be served.

### TTL (Time to Live)

The duration after which a cache entry expires.

### Cache eviction

Removing entries to free memory, often according to a policy such as least recently used (LRU).

Expiration and eviction are different: an entry may be evicted before its TTL expires if the cache needs memory.

---

## 5. Where Can Caching Be Used?

Caching can exist at several layers.

### A. Browser cache

Stores suitable assets or responses on the user's device.

Examples:
- Images.
- JavaScript bundles.
- CSS files.
- Certain HTTP responses.

Benefits:
- Reduces repeated network transfers.
- Improves repeat visits.

Trade-offs:
- Cache-control headers and versioning must be designed correctly.
- Private or sensitive data requires careful handling.

### B. CDN cache

A Content Delivery Network can cache eligible content closer to users.

Examples:
- Static assets.
- Public images.
- Public content.
- Some cacheable API responses.

Benefits:
- Reduces origin traffic.
- Can reduce latency for geographically distributed users.

Trade-offs:
- Cache invalidation and freshness rules.
- Cache-key design.
- Risk of incorrectly caching personalized responses.

### C. Application-local cache

Stores data in an application's process memory.

Benefits:
- Very low lookup overhead.
- No network hop to a cache service.

Trade-offs:
- Each application instance may have a different copy.
- Memory is limited.
- Entries disappear when an instance restarts.
- Keeping multiple instances synchronized is difficult.

### D. Distributed cache

A separate service, such as Redis, stores reusable data for multiple application instances.

Benefits:
- Shared cache across application instances.
- Can reduce repeated database reads.
- Cache lifecycle can be managed independently of application instances.

Trade-offs:
- Network calls to the cache.
- Additional operational cost.
- Cache outages and network failures.
- Consistency and invalidation complexity.

### E. Database caching

Databases may cache query results, index pages, or data pages internally.

This is different from an application-managed cache. Database internal caches do not automatically eliminate every repeated query or application-level processing step.

---

## 6. Choosing Data to Cache

Not all data is equally suitable for caching.

Consider two types of booking data.

### Relatively stable data

Examples:
- Service-area descriptions.
- Vehicle type names.
- Configuration values.
- Frequently used reference data.

These are often good candidates because many requests reuse the same values, and updates may be infrequent.

### Frequently changing data

Examples:
- Live driver location.
- Booking status.
- Available vehicle count.
- Inventory availability.

These require more careful freshness and invalidation rules.

### Questions to ask before caching

1. Is the data read frequently?
2. Is the same data requested repeatedly?
3. How expensive is retrieving the data?
4. How frequently does it change?
5. How stale may it be?
6. What happens if the cache returns an outdated value?
7. How large are the cached objects?
8. How will the cache be invalidated?
9. What happens when the cache is unavailable?

**A good cache candidate is data that can be reused enough to justify the cost and consistency trade-offs.**

---

## 7. Cache-Aside Pattern

Cache-aside is a common starting point for application-managed caching.

The application explicitly checks the cache and retrieves data from the original source when necessary.

### Read flow

```text
1. Client requests booking B123
              |
              v
2. Application checks cache
              |
        +-----+-----+
        |           |
        v           v
    Cache hit    Cache miss
        |           |
        v           v
   Return data   Read database
                    |
                    v
                Populate cache
                    |
                    v
                Return data
```

### Example

Suppose the cache contains:

```text
booking:B123 -> {
    id: "B123",
    status: "scheduled"
}
```

When the application requests `booking:B123`, it returns the cached value if the entry is available and acceptable under the freshness policy.

If the entry is missing, the application queries PostgreSQL and may populate the cache.

### Advantages

- Simple to understand.
- Data is loaded only when requested.
- Unpopular data does not need to be cached.
- Cache failures can sometimes be handled by reading the original source.

### Trade-offs

- The first read after a miss is slower.
- Cache entries may become stale.
- Many simultaneous misses can overload the database.
- Application code must manage cache behavior.

Cache-aside does not inherently guarantee consistency between the cache and the database.

---

## 8. Write-Through Caching

With write-through, the write path updates the database and the cache as part of its defined write workflow.

Conceptually:

```text
Write request
     |
     v
Application
     |
     +------> Database update
     |
     +------> Cache update
```

The exact ordering and failure behavior depend on the implementation.

### Advantages

- Can keep frequently read entries fresher after writes.
- May reduce the need for a later cache miss.
- Can make recently updated data available quickly.

### Trade-offs

- Writes involve extra work.
- The database and cache are separate systems.
- A successful database commit followed by a failed cache update creates inconsistency.
- Concurrent writes may update the cache out of order.

An ordinary PostgreSQL transaction cannot automatically make a separate Redis update part of the same atomic transaction.

Therefore, write-through is a strategy, not a guarantee of atomic consistency.

---

## 9. Read-Through Caching

In a read-through design, the application asks the caching layer for the data, and the caching layer or its integrated loader retrieves the original value on a miss.

Conceptually:

```text
Application
     |
     v
Caching layer
     |
     +---- Hit ----> Return cached value
     |
     +---- Miss ---> Load from data source
                          |
                          v
                     Store value
                          |
                          v
                     Return value
```

### Difference from cache-aside

- With cache-aside, the application explicitly implements the cache lookup and fallback logic.
- With read-through, that loading behavior is encapsulated by the cache layer or caching abstraction.

### Advantages

- Centralizes cache-loading behavior.
- Can reduce repeated loading logic across callers.

### Trade-offs

- More dependence on the caching abstraction.
- Loading and error-handling behavior may be less visible to application code.
- Not every cache technology supports read-through behavior natively.

---

## 10. Write-Behind (Write-Back) Caching

With write-behind, the system accepts a change into the cache or an intermediate layer and persists it to the database later.

Conceptually:

```text
Write request
     |
     v
Cache or write buffer
     |
     v
Background persistence
     |
     v
Database
```

### Advantages

- Can reduce synchronous write latency.
- Can batch writes.
- Can improve write throughput for suitable workloads.

### Trade-offs

- Data may be lost if the buffered write is not durable and the system fails.
- Database persistence is delayed.
- Ordering and conflict handling become more complicated.
- The application must track failed or pending writes.

Write-behind is riskier for critical business data unless durability, replay, and recovery are carefully designed.

For a booking platform, acknowledging a booking as successfully persisted before its durable storage is guaranteed requires a deliberate reliability design.

---

## 11. Comparing Caching Strategies

| Strategy | Read behavior | Write behavior | Main consideration |
|---|---|---|---|
| Cache-aside | Application checks cache and loads on a miss | Application updates source and manages invalidation | Simple, but freshness must be handled |
| Read-through | Cache layer loads on a miss | Depends on the write strategy | Centralized loading behavior |
| Write-through | Reads can use cached entries | Write path updates source and cache | Write latency and cross-system failures |
| Write-behind | Reads can use cached entries | Persistence happens asynchronously | Durability and delayed persistence |

These patterns can be combined. For example, an application may use cache-aside reads and invalidate entries after successful database writes.

No strategy automatically solves every consistency problem.

---

## 12. TTL: Time to Live

TTL defines how long an entry remains eligible for use before expiration.

Suppose:

```text
booking:B123
TTL = 30 seconds
```

After the entry expires, a subsequent cache lookup should treat it as unavailable and load fresh data according to the cache strategy.

### Advantages

- Limits how long an entry can remain cached under normal expiration behavior.
- Helps remove abandoned entries.
- Reduces the need to explicitly invalidate every entry.

### Trade-offs

- A long TTL may permit stale data.
- A short TTL may increase database load.
- Expiration can create bursts of cache misses.
- TTL alone does not guarantee a strict end-to-end freshness bound during failures.

### Example

Suppose booking status changes from `scheduled` to `arrived`.

If the cache contains the old status with a remaining TTL of 25 seconds, TTL alone may allow that old status to be served during that interval.

If the requirement is that status changes become visible within two seconds, a 30-second TTL is not sufficient as the sole freshness mechanism.

TTL should be selected based on data volatility, workload, and business requirements.

---

## 13. Cache Invalidation

Cache invalidation means making a cached value unavailable or replacing it when the underlying data changes.

Consider:

```text
PostgreSQL:
booking:B123 -> scheduled

Redis:
booking:B123 -> scheduled
```

A dispatcher updates the booking to `arrived`.

PostgreSQL now contains the new status, but Redis may still contain the old status.

### Strategy A: Invalidate on write

After a successful database update, delete the corresponding cache entry.

```text
1. Update PostgreSQL
2. Commit database transaction
3. Invalidate cache entry
4. Future reads load the latest database value
```

Advantages:
- Simple invalidation behavior.
- Avoids explicitly calculating the replacement cache value.

Trade-offs:
- If invalidation fails, the old entry may remain.
- A cache miss can create a database-read burst.
- Concurrent requests may repopulate stale values.

### Strategy B: Update cache on write

After updating the database, update the cached value.

Advantages:
- The cache may immediately contain the new value.
- A subsequent read may avoid a cache miss.

Trade-offs:
- The cache update can fail after the database commits.
- Concurrent updates can overwrite newer values with older ones.
- Cache and database writes are not automatically atomic.

### Strategy C: TTL as a fallback

Use a TTL to limit the lifetime of cached entries, alongside explicit invalidation or updates.

This provides a cleanup mechanism but should not be treated as a complete solution for strict freshness requirements.

---

## 14. The Cache Invalidation Race Condition

A subtle race can cause stale data even when invalidation is implemented.

Suppose two requests overlap.

```text
Request A:
1. Reads old booking status from database
2. Pauses before writing to cache

Request B:
1. Updates booking status in database
2. Invalidates the cache

Request A:
3. Writes the old status into cache
```

The cache now contains stale data again.

This demonstrates that deleting a cache key after a database write does not automatically eliminate every race.

Possible mitigations include:

- Versioned cache entries.
- Conditional cache writes.
- Coordinated update workflows.
- Reliable event-driven invalidation.
- Avoiding caching for particularly volatile fields.
- Reading from the authoritative database for operations requiring stronger freshness.

The right solution depends on the consistency requirements and concurrency patterns.

---

## 15. Cache Stampede

A cache stampede occurs when many requests simultaneously experience a cache miss for the same key and all attempt to load the same data from the original source.

Example:

```text
Cache entry expires
        |
        v
1,000 requests arrive
        |
        v
1,000 cache misses
        |
        v
1,000 database queries
```

This can overload the database and make the problem worse.

### Common mitigations

#### Request coalescing

Allow one request to load the missing value while other requests wait for or reuse its result.

#### TTL jitter

Add a small random variation to expiration times so many entries do not expire simultaneously.

#### Refresh ahead

Refresh selected entries before expiration when the workload and cache architecture justify it.

#### Stale-while-revalidate

Serve a previously cached value while refreshing it in the background, but only where the freshness policy allows stale data.

### Trade-offs

These techniques add coordination or freshness complexity. They should be used where cache-miss bursts are a meaningful risk.

---

## 16. Cache Penetration and Cache Avalanche

### Cache penetration

A cache penetration problem occurs when repeated requests for nonexistent or uncacheable data bypass the cache and repeatedly hit the database.

Example:

```text
Request for nonexistent booking
           |
           v
Cache miss
           |
           v
Database confirms not found
           |
           v
No result cached
           |
           v
Repeated requests hit database again
```

Possible mitigations:
- Briefly cache negative results where appropriate.
- Validate request identifiers.
- Apply rate limits.
- Use a Bloom filter for suitable large key spaces.

Negative caching must be designed carefully so newly created data does not remain incorrectly invisible.

### Cache avalanche

A cache avalanche occurs when many entries become unavailable around the same time, producing a sudden surge of requests to the original source.

Possible causes:
- Many keys share similar expiration times.
- A cache service becomes unavailable.
- A large cache is flushed.

Possible mitigations:
- TTL jitter.
- Controlled cache warming.
- Rate limits and concurrency limits.
- Graceful degradation.
- Protection for the database during cache failure.

A cache can reduce normal database load but create a significant risk if the database cannot handle the miss traffic during an outage.

---

## 17. Cache Consistency Models

Consistency describes how fresh and coordinated the data returned by a system must be.

### Eventual consistency

After updates stop, replicas or caches are expected to converge eventually.

During propagation, readers may see old data.

Useful when temporary staleness is acceptable.

### Bounded staleness

The system targets an explicit maximum age or delay for stale data under defined operating conditions.

Example:
Booking status should reflect an update within two seconds.

A TTL alone does not necessarily guarantee this bound. Invalidation delays, failures, retries, and cache behavior must be considered.

### Strong consistency

Reads and writes follow a defined consistency model that prevents certain stale observations.

Strong consistency is more expensive in some distributed architectures because coordination may be required.

It is important to define exactly what is guaranteed rather than assuming that a cache or database automatically provides the desired semantics across all components.

### Choosing a model

| Data | Possible approach |
|---|---|
| Public reference data | Longer TTL may be acceptable |
| Product descriptions | Cached reads with controlled invalidation |
| Booking status | Short freshness target and reliable update handling |
| Financial balance | Stronger correctness requirements; caching requires careful design |
| Live driver location | Time-sensitive data with an explicit acceptable age |

The examples are starting points. Actual requirements depend on the product and consequences of stale data.

---

## 18. Handling Cache Failure

Suppose Redis becomes unavailable.

The application may need to read PostgreSQL directly, but doing so can create a large increase in database load.

A resilient design should consider:

- Cache connection timeouts.
- A bounded fallback policy.
- Database concurrency limits.
- Rate limiting and load shedding.
- Monitoring for sudden query-volume increases.
- Recovery behavior when Redis returns.
- Whether stale data is acceptable for specific reads.

### Important trade-off

Fallback to PostgreSQL is not automatically safe.

If the database is already near capacity, redirecting all cache traffic to it can cause a cascading failure.

Some operations may need to fail fast rather than overload the database. Other operations may tolerate a controlled fallback.

The correct choice depends on business importance, data freshness, and available capacity.

---

## 19. Cache Key Design

A cache key uniquely identifies the cached value.

Example:

```text
booking:B123
```

For user-specific data, the key may need to incorporate tenant, user, locale, or other relevant dimensions.

Conceptually:

```text
tenant:{tenantId}:booking:{bookingId}
```

The exact format is a design choice.

### Why key design matters

If two logically different requests map to the same cache key, one user may receive another user's data or a response intended for a different tenant.

Keys should account for all dimensions that affect the returned data.

Consider:
- Tenant identity.
- Authorization scope.
- Locale.
- Query filters.
- Data version.
- Representation format.

Do not include secrets or sensitive personal information in cache keys unnecessarily. Apply access controls and suitable expiration policies to cached data.

---

## 20. Cache Invalidation Across Multiple Application Instances

Suppose three application servers maintain independent in-memory caches.

```text
             Load Balancer
              /    |    \
             v     v     v
          App A  App B  App C
          Cache  Cache  Cache
```

A write processed by App A may update its local cache while App B and App C continue serving older values.

Possible approaches include:

- Use a shared distributed cache.
- Publish invalidation events to all instances.
- Use versioned keys or cache namespaces.
- Keep local caches only for data whose staleness is acceptable.

A shared cache reduces the problem of independently stored values, but it does not eliminate consistency issues between the shared cache and the authoritative database.

---

## 21. Calculating Potential Database Read Reduction

Suppose:

- Total incoming API requests = 800 RPS.
- 70% of requests are eligible for caching.
- Cache hit rate for eligible requests = 80%.
- Each eligible cache miss requires one database read.

Eligible requests:

```text
800 * 0.70 = 560 RPS
```

Cache hits:

```text
560 * 0.80 = 448 RPS
```

Eligible cache misses:

```text
560 * 0.20 = 112 RPS
```

Under these assumptions, the cache serves 448 requests per second that would otherwise have required a database read.

This is a simplified estimate. Actual database work may differ because:
- Some requests perform multiple queries.
- Some cache misses may not reach the database.
- Some requests may require additional uncached reads.
- Cache misses may be more expensive than cache hits.

A cache hit rate alone does not fully describe the performance benefit.

---

## 22. When Should You Introduce Redis?

Redis is a common choice for distributed caching, but it should solve a specific problem.

Consider Redis when:
- Multiple application instances need a shared cache.
- Repeated reads create significant database load.
- Low-latency key-based lookups are useful.
- The data can be cached under a defined freshness policy.
- The team can operate and monitor the additional service.

Consider alternatives when:
- The database is already fast enough.
- Requests rarely reuse the same data.
- The workload is small.
- A browser cache or CDN can handle the relevant content.
- The application only needs a small, local cache.

Before adding Redis, identify the expected hit rate, memory requirement, failure behavior, and invalidation strategy.

**Do not introduce a distributed cache solely because a system is expected to scale.**

---

## 23. Worked Example: Booking Platform

### Requirements

- Peak traffic: 800 RPS.
- Booking details are frequently read.
- Booking status must become visible within two seconds.
- PostgreSQL is experiencing high read load.

### Initial design

Use PostgreSQL as the authoritative data source and evaluate Redis as a shared cache for suitable booking details.

### Read strategy

1. Check Redis.
2. If a valid entry exists, return it.
3. If the entry is missing, read PostgreSQL.
4. Populate the cache under the defined freshness policy.
5. Return the result.

### Write strategy

1. Update booking status in PostgreSQL.
2. Commit the database transaction.
3. Invalidate or update the relevant cache entry.
4. Retry or recover from invalidation failures through a reliable mechanism where required.
5. Ensure concurrent requests cannot indefinitely repopulate stale values.

### Failure considerations

If Redis is unavailable:
- Avoid unbounded fallback traffic to PostgreSQL.
- Apply appropriate timeouts and concurrency controls.
- Monitor database load.
- Use a safe fallback or fail fast according to the operation's requirements.

### Validation

Measure:
- Cache hit rate.
- Database query volume.
- Database CPU and latency.
- End-to-end P95 and P99 latency.
- Cache invalidation delay.
- Stale-read incidents.
- Behavior during cache failure.

The architecture is successful only if it improves performance while meeting the two-second freshness requirement.

---

## 24. Practical Caching Decision Checklist

Before introducing or changing a cache, ask:

1. What performance problem are we solving?
2. Which data should be cached?
3. How often is the data read?
4. How often does the data change?
5. What staleness is acceptable?
6. Which caching strategy fits the read/write pattern?
7. How are entries invalidated or refreshed?
8. Can concurrent requests cause stale values to be repopulated?
9. What happens when the cache is unavailable?
10. Can the database handle cache-miss traffic?
11. How are cache stampedes prevented?
12. How are cache keys scoped to tenants and users?
13. What is the expected hit rate?
14. How much memory is required?
15. How will performance and correctness be measured?

---

## 25. Revision Questions

### Q1. What is caching?

Answer:
Storing a reusable copy of data to reduce the cost of retrieving or computing it again.

### Q2. What is the difference between a cache hit and a cache miss?

Answer:
A hit finds a usable cached value. A miss requires the application to retrieve the value from another source or compute it.

### Q3. What is cache-aside?

Answer:
The application checks the cache, loads data from the source on a miss, and may populate the cache.

### Q4. What is TTL?

Answer:
The duration an entry remains eligible for use before expiration.

### Q5. Why is TTL alone not always sufficient?

Answer:
TTL may allow stale data to remain available for too long, and it does not guarantee an end-to-end freshness bound during failures or concurrent updates.

### Q6. What is a cache stampede?

Answer:
Many requests simultaneously miss the same cache entry and overwhelm the original data source with duplicate work.

### Q7. What is eventual consistency?

Answer:
Updates may not become visible immediately, but replicas or cached values are expected to converge eventually.

### Q8. Why can cache failure cause a database outage?

Answer:
Requests that normally hit the cache may suddenly be redirected to the database, exceeding its available capacity.

### Q9. Why must cache keys include relevant tenant or user dimensions?

Answer:
To prevent logically different requests from mapping to the same cached value and potentially exposing incorrect or unauthorized data.

### Q10. When should caching be avoided?

Answer:
When reuse is low, freshness requirements cannot be met economically, the original source is already sufficiently fast, or cache complexity outweighs the expected benefit.

---

## 26. Final Summary

Remember these principles:

- Caching reduces repeated work but introduces freshness and operational trade-offs.
- Cache-aside is a common starting point for application-managed caching.
- Read-through, write-through, and write-behind describe different loading and persistence strategies.
- TTL limits entry lifetime but does not solve every invalidation problem.
- Cache invalidation can race with concurrent requests.
- Cache stampedes can overload the original data source.
- Eventual consistency, bounded staleness, and strong consistency represent different requirements.
- Cache failure needs a deliberate fallback strategy.
- Cache keys must preserve tenant, user, and query boundaries where relevant.
- Hit rate is useful but does not fully measure database workload reduction.
- Caching is most effective when repeated work is expensive and data can be reused safely.
- Measure performance and correctness before and after introducing a cache.

**The central lesson:** A cache is not simply a faster database. It is an additional data-management layer whose value depends on reuse, freshness, failure behavior, and operational complexity.
