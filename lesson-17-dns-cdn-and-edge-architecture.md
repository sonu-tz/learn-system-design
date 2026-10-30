# Lesson 17 — DNS, CDN, and Edge Architecture

So far, we've focused primarily on backend architecture, databases, distributed systems, and APIs.

Now let's look at what happens before a request even reaches your application server.

Imagine a passenger opens your ride-booking application from India, while your application servers are hosted in Europe. How can the application load quickly? How does the browser find your server? How can images, JavaScript bundles, and other static assets be delivered without repeatedly hitting your origin servers?

This is where DNS, CDN, caching at the edge, and edge architecture become important.

## 1. Understanding the Request Journey

When a user opens an application, several things happen before the application server processes an API request.

Consider:

```text
https://app.example.com
```

A simplified request journey:

```text
User Browser
     |
     ▼
    DNS
     |
     ▼
   CDN / Edge
     |
     ├── Cache Hit ──> Return Cached Content
     |
     └── Cache Miss
             |
             ▼
       Load Balancer
             |
             ▼
       Application Server
             |
             ▼
          Database
```

Each component has a different responsibility.

| Component | Responsibility |
|---|---|
| DNS | Resolves domain names to network destinations |
| CDN | Delivers cached content from geographically distributed locations |
| Edge | Processes requests closer to users |
| Load Balancer | Distributes traffic across backend instances |
| Application Server | Executes business logic |
| Database | Stores application data |

Understanding this journey helps identify where latency originates and which optimizations are appropriate.

---

## 2. DNS — Domain Name System

DNS translates human-readable domain names into IP addresses.

For example:

```text
app.example.com
       |
       ▼
   DNS Lookup
       |
       ▼
  203.0.113.10
```

Browsers and other clients use the resolved network destination to connect to a server or intermediary.

Without DNS, users would need to remember IP addresses.

### How DNS resolution works

Suppose a user visits:

```text
https://app.example.com
```

A simplified resolution process:

1. Browser checks its DNS cache.
2. Operating system checks its local DNS cache.
3. A configured recursive resolver is queried.
4. If necessary, the resolver queries the DNS hierarchy.
5. The resolver obtains the relevant DNS record.
6. The answer is cached according to its TTL.
7. The browser uses the resolved address to connect.

### DNS hierarchy

```text
              Root DNS
                  |
                  ▼
               .com TLD
                  |
                  ▼
           example.com DNS
                  |
                  ▼
          app.example.com
                  |
                  ▼
             IP Address
```

The recursive resolver normally performs the work of querying the hierarchy on behalf of the client.

---

## 3. Important DNS Record Types

DNS supports different record types for different purposes.

| Record | Purpose | Example |
|---|---|---|
| A | Maps a hostname to an IPv4 address | `app.example.com → 203.0.113.10` |
| AAAA | Maps a hostname to an IPv6 address | `app.example.com → 2001:db8::10` |
| CNAME | Maps one hostname to another hostname | `www.example.com → example.com` |
| MX | Specifies mail servers | Mail routing |
| TXT | Stores text-based information | Domain verification, SPF |
| NS | Identifies authoritative name servers | DNS delegation |
| PTR | Reverse lookup from IP to hostname | Reverse DNS |

### A record vs. CNAME

An A record points to an IPv4 address.

A CNAME points to another hostname.

Example:

```text
api.example.com    A       203.0.113.10
www.example.com    CNAME   example.com
```

The target of a CNAME is resolved separately.

A traditional CNAME cannot coexist with other record types at the same DNS name. Some DNS providers offer ALIAS or ANAME-like functionality to work around this limitation at the zone apex.

---

## 4. DNS TTL and Caching

TTL means Time To Live.

It specifies how long a DNS answer can generally be cached before it should be refreshed.

Example:

```text
app.example.com → 203.0.113.10
TTL = 300 seconds
```

The resolver can cache the answer for approximately five minutes.

### Why does TTL matter?

Suppose your application server moves from:

```text
203.0.113.10
```

to:

```text
203.0.113.20
```

If DNS records have a long TTL, some clients may continue using the previous answer until their cached entry expires.

A shorter TTL can help clients discover changes sooner, but increases DNS query traffic.

### Important limitation

Changing DNS does not instantly redirect every client.

- Resolvers may have cached the old answer.
- Some clients may cache results independently.
- Existing TCP connections may remain active.
- DNS TTL is not a guaranteed global propagation timer.

**Practical rule:** Lower the TTL before a planned migration, make the change, verify traffic movement, and keep the old destination available during the transition where possible.

---

## 5. DNS-Based Traffic Routing

DNS can direct users toward different destinations.

For example:

```text
                    DNS
                     |
             ┌───────┴────────┐
             |                |
             ▼                ▼
         India Region     Europe Region
             |                |
             ▼                ▼
        App Servers       App Servers
```

Common routing strategies:

### 5.1 Round-robin DNS

DNS returns multiple IP addresses.

Example:

```text
app.example.com → IP-A
app.example.com → IP-B
app.example.com → IP-C
```

Resolvers and clients may distribute connections among returned addresses.

**Limitation:** DNS does not guarantee equal request distribution or immediate health-based failover.

### 5.2 Weighted routing

Traffic is directed toward destinations according to configured weights.

Example:

- Region A: 70%.
- Region B: 30%.

Useful for gradual migrations or traffic distribution.

Actual distribution depends on resolver caching and client behavior, so DNS weights are not exact request-level controls.

### 5.3 Geolocation routing

DNS directs users based on approximate geographic location.

Example:

- Users in India → India region.
- Users in Europe → Europe region.
- Users in North America → North America region.

### 5.4 Latency-based routing

DNS attempts to direct clients toward a destination with lower network latency.

This can improve user experience, but the routing decision depends on the DNS provider's measurement and resolver context.

### 5.5 Failover routing

If a primary destination becomes unhealthy, DNS can return a backup destination.

However, DNS caching means failover may not be instantaneous.

For faster traffic switching, health-aware load balancers and CDN routing may be used in combination with DNS.

---

## 6. What Is a CDN?

CDN stands for Content Delivery Network.

A CDN is a geographically distributed network of servers that delivers content closer to users.

Instead of every user downloading static assets from your origin server, the CDN can serve cached copies from nearby edge locations.

Examples include:

- Cloudflare.
- Amazon CloudFront.
- Akamai.
- Fastly.

### Without a CDN

Imagine your application is hosted in Europe.

A user in India requests:

```text
GET /assets/app.js
```

The request travels to the origin server in Europe.

```text
India User
    |
    | Long-distance network request
    ▼
Europe Origin Server
    |
    ▼
   app.js
    |
    ▼
India User
```

### With a CDN

The user requests the same file.

If an edge location in or near India already has a cached copy, the CDN can serve it directly.

```text
India User
    |
    ▼
India CDN Edge
    |
    ├── Cache Hit ──> Return app.js
    |
    └── Cache Miss ──> Fetch from Origin
```

The origin server handles fewer requests, and users may experience lower latency.

---

## 7. CDN Architecture

A simplified CDN architecture:

```text
                    Users
                      |
                      ▼
                   DNS
                      |
                      ▼
                 CDN Network
                      |
            ┌─────────┴─────────┐
            |                   |
            ▼                   ▼
       Edge Location A     Edge Location B
            |                   |
            └─────────┬─────────┘
                      |
                      ▼
                 Origin Server
                      |
                      ▼
                   Storage
```

### Important CDN components

**Origin server**

The original source of the content.

Examples:

- Application server.
- Object storage.
- Static asset storage.

**Edge location**

A geographically distributed point of presence that receives and serves requests.

**Cache**

Stores copies of content to reduce repeated origin requests.

**Origin shield**

An additional caching layer between edge locations and the origin. It can reduce duplicate origin requests when many edge locations request the same uncached object.

---

## 8. CDN Cache Hit and Cache Miss

A CDN request can result in either a cache hit or cache miss.

### Cache hit

The requested content is already cached at the edge.

```text
User
  |
  ▼
CDN Edge
  |
  ▼
Cache Found
  |
  ▼
Return Content
```

The origin server is not required for that request.

### Cache miss

The requested content is not cached or is no longer fresh.

```text
User
  |
  ▼
CDN Edge
  |
  ▼
Cache Not Found
  |
  ▼
Origin Server
  |
  ▼
Fetch Content
  |
  ▼
Cache Content
  |
  ▼
Return Content
```

The exact caching behavior depends on CDN configuration and origin cache headers.

### Cache hit ratio

The cache hit ratio measures the proportion of requests served from cache.

\[
Hit\ Ratio = \frac{Cache\ Hits}{Total\ Cache\ Requests}
\]

Example:

- Total requests: 1,000,000.
- Cache hits: 850,000.
- Cache misses: 150,000.

\[
Hit\ Ratio = \frac{850,000}{1,000,000} = 85\%
\]

An 85% hit ratio means 85% of eligible requests were served from cache.

Do not assume every application request is cacheable. Dynamic authenticated API requests may require different treatment.

---

## 9. What Should Be Cached at the CDN?

CDNs are particularly useful for content that is shared across users and changes relatively infrequently.

### Good candidates

- JavaScript bundles.
- CSS files.
- Images.
- Fonts.
- Videos.
- Public documents.
- Public API responses with suitable cache policies.

### Usually not safe to cache publicly without careful controls

- Passenger private profile.
- Payment details.
- Authenticated ride information.
- Personal location data.
- Tenant-specific operational information.

A major risk is accidentally sharing one user's response with another user.

For private data, use appropriate private caching policies or disable caching.

### Example asset caching policy

For versioned static assets:

```http
Cache-Control: public, max-age=31536000, immutable
```

This tells caches that the asset can be reused for a long period because its URL is versioned.

For an application HTML entry point:

```http
Cache-Control: no-cache
```

This permits storage but requires revalidation before reuse.

The exact policy depends on deployment and update requirements.

---

## 10. Cache Invalidation and Versioned Assets

One common problem:

You deploy a new frontend version, but users continue receiving an older JavaScript bundle from the CDN.

How do we prevent this?

### Strategy 1: Cache invalidation

Explicitly purge cached files from the CDN.

Example:

```text
/assets/app.js
```

The CDN is instructed to remove or refresh the cached object.

**Trade-off:** Purging large numbers of objects can be slow or operationally expensive.

### Strategy 2: Content-hashed filenames

Generate filenames based on file content.

Example:

```text
main.a8f31c.js
main.b9e42d.js
```

When the file changes, its hash changes, creating a new URL.

The browser requests the new URL instead of relying on the old cached asset.

This is one of the most reliable approaches for frontend deployment.

### Recommended frontend strategy

| Resource | Typical caching strategy |
|---|---|
| HTML entry point | Revalidate frequently |
| Content-hashed JS | Long-lived immutable caching |
| Content-hashed CSS | Long-lived immutable caching |
| Fonts | Long-lived caching |
| User-specific API responses | Private or no caching unless explicitly designed |

---

## 11. CDN vs. Reverse Proxy vs. Load Balancer

These components can overlap in functionality, but they solve different problems.

| Component | Main purpose |
|---|---|
| CDN | Deliver content from distributed edge locations |
| Reverse proxy | Receive requests on behalf of backend servers |
| Load balancer | Distribute traffic across backend targets |
| API Gateway | Apply API-aware routing, security, and policies |

Example architecture:

```text
             User
               |
               ▼
              DNS
               |
               ▼
              CDN
               |
               ▼
          Load Balancer
               |
               ▼
          API Gateway
               |
        ┌──────┴──────┐
        ▼             ▼
    App Server A  App Server B
```

A real platform may combine several of these responsibilities in one product.

---

## 12. Edge Computing

A traditional architecture executes most business logic in a centralized region.

Edge computing moves selected processing closer to the user.

Example:

```text
                    Users
                      |
                      ▼
                 Edge Network
                      |
          ┌───────────┼───────────┐
          |           |           |
          ▼           ▼           ▼
       India       Europe       US Edge
       Edge        Edge         Edge
          |           |           |
          └───────────┼───────────┘
                      |
                      ▼
               Central Backend
                      |
                      ▼
                  Database
```

### What can run at the edge?

- Request routing.
- Authentication checks, where supported and safely designed.
- Rate limiting.
- Redirects.
- Header transformations.
- Lightweight request processing.
- Content personalization using suitable non-sensitive context.
- Static content delivery.

### Example

Suppose your application receives requests from different countries.

An edge function could inspect the request region and route it to an appropriate backend region.

This can reduce unnecessary network travel.

### Advantages

- Lower latency for suitable operations.
- Reduced origin traffic.
- Distributed request processing.
- Better handling of geographically distributed users.

### Disadvantages

- More complex deployment and debugging.
- Limited runtime and execution constraints.
- Distributed state is harder to manage.
- Some operations still need centralized coordination.
- Data residency and privacy requirements may restrict processing locations.

**Important:** Edge computing is not a replacement for your database or core transactional services.

---

## 13. CDN and Dynamic APIs

A common misconception is that a CDN only works for static files.

CDNs can also accelerate dynamic requests through:

- Connection reuse.
- TLS termination.
- Network optimization.
- Request routing.
- Compression.
- Edge-side processing.
- Caching selected API responses.

However, caching dynamic APIs requires careful consideration.

Example:

```http
GET /api/public/vehicle-types
```

This response might be identical for every user and suitable for caching.

But:

```http
GET /api/rides/501
Authorization: Bearer <token>
```

This response may contain private passenger information.

Caching it in a shared public cache could expose sensitive data if cache keys and policies are incorrect.

### Vary and cache keys

A CDN may use request properties to determine whether two requests refer to the same cached object.

Possible cache-key dimensions include:

- URL path.
- Query parameters.
- Selected request headers.
- Language.
- Content encoding.

Incorrect cache-key design can cause users to receive the wrong representation.

For example, if tenant identity affects the response but is not represented in the cache key or cache policy, one tenant's data could be served to another.

For sensitive endpoints, the safest design is often to bypass shared caching unless the isolation requirements are explicitly implemented and tested.

---

## 14. DNS, CDN, and Deployment Strategy

Consider deploying a new version of a frontend application.

A safe deployment might follow these steps:

1. Build versioned frontend assets.
2. Upload assets to object storage.
3. Verify that the new assets are available.
4. Update the HTML entry point to reference the new asset hashes.
5. Configure suitable cache headers.
6. Deploy the backend changes using backward-compatible API contracts.
7. Verify the application from multiple regions.
8. Monitor error rates and latency.
9. Roll back the HTML or application deployment if needed.

### Why backward compatibility matters

Some users may still have an older frontend bundle cached in their browser.

If the new backend immediately removes old API fields or endpoints, those users may encounter errors.

Frontend and backend deployments should therefore tolerate overlapping versions when possible.

This is especially important for applications with long-lived browser sessions.

---

## 15. Performance Metrics

To understand whether your CDN and edge architecture are helping, monitor:

### CDN metrics

- Cache hit ratio.
- Cache miss ratio.
- Origin request count.
- Origin bandwidth.
- Edge response latency.
- Origin response latency.
- Cache eviction rate.
- Error rate.

### DNS metrics

- DNS lookup latency.
- Resolution failures.
- DNS query volume.
- Health-check and failover behavior.

### Application metrics

- P50, P95, and P99 latency.
- API error rate.
- Backend CPU and memory.
- Database load.
- Regional traffic distribution.

### Example

Before CDN:

- 1,000,000 static asset requests/day.
- Almost all requests reach the origin.

After CDN:

- 850,000 cache hits/day.
- 150,000 cache misses/day.

Potential reduction in origin requests:

\[
\frac{850,000}{1,000,000} \times 100 = 85\%
\]

This assumes all requests are eligible for caching and the cache hit measurement reflects those requests.

The actual benefit should be verified through origin traffic and performance metrics.

---

## 16. Common Mistakes

1. Assuming DNS changes take effect instantly everywhere.
2. Treating DNS round-robin as a reliable load balancer.
3. Caching authenticated API responses in shared caches without proper isolation.
4. Using long-lived caching for HTML entry points without a versioning strategy.
5. Assuming a CDN eliminates the need for backend scaling.
6. Forgetting that cache misses still reach the origin.
7. Ignoring CDN cache-key design.
8. Assuming edge functions can replace centralized transactional logic.
9. Ignoring geographic data residency requirements.
10. Failing to test deployments with older cached frontend versions.

---

## 17. Practical Checklist

### DNS
- [ ] DNS records are correctly configured.
- [ ] TTL values are appropriate.
- [ ] DNS changes are tested before major migrations.
- [ ] Failover behavior accounts for DNS caching.

### CDN
- [ ] Static assets use content-hashed filenames.
- [ ] Cache-Control headers are configured.
- [ ] Private data is not accidentally shared.
- [ ] Cache keys include the necessary dimensions.
- [ ] CDN hit ratio and origin traffic are monitored.

### Edge
- [ ] Only suitable lightweight processing runs at the edge.
- [ ] Regional routing is understood.
- [ ] Data residency requirements are considered.
- [ ] Edge failures have a defined fallback.

### Deployment
- [ ] Frontend and backend changes are backward-compatible.
- [ ] New assets are available before the HTML references them.
- [ ] Rollback procedures are defined.
- [ ] Performance is tested from multiple regions.

---

## 18. Revision Questions

1. What problem does DNS solve?
2. Explain the DNS resolution process.
3. What is the difference between A, AAAA, and CNAME records?
4. What is DNS TTL?
5. Why doesn't changing DNS instantly redirect every user?
6. What is a CDN?
7. What is the difference between a cache hit and cache miss?
8. Calculate the cache hit ratio if 720,000 out of 900,000 requests are cache hits.
9. Which frontend assets should use long-lived immutable caching?
10. Why should HTML entry points generally be revalidated frequently?
11. What is the difference between a CDN and a load balancer?
12. What is edge computing?
13. Why can an incorrect CDN cache key cause a security problem?
14. Can CDNs accelerate dynamic APIs? Explain how.
15. Why should frontend and backend deployments support overlapping versions?

---

## 19. Practice Exercise — Design Global Content Delivery

You are designing a ride-booking application with:

- Frontend hosted in one central region.
- Users across India, Europe, and North America.
- Large JavaScript and CSS bundles.
- Driver profile images.
- Public vehicle-type API.
- Private passenger ride APIs.
- Frequent frontend deployments.

Answer these questions:

1. How would you configure DNS?
2. Where would you introduce a CDN?
3. Which resources would you cache?
4. What cache headers would you use for content-hashed JS and CSS files?
5. How would you prevent private passenger data from being shared between users?
6. How would you handle CDN cache invalidation after a deployment?
7. Would you use edge computing? Which operations would run there?
8. How would you ensure users with an older frontend version can still use the new backend?
9. Which metrics would you monitor to verify that the CDN is improving performance?

### Bonus interview question

"Your application is hosted in one region, but users in multiple continents report high latency. How would you investigate and improve performance without immediately moving your entire backend to multiple regions?"

Think about DNS, CDN, edge routing, static assets, connection latency, API placement, and database dependencies.

**Key takeaway:** DNS helps users discover where to connect, CDNs bring content closer to users, and edge computing moves suitable processing closer to the request. Together, they improve global performance while reducing unnecessary load on centralized infrastructure.

### Practice focus

Try the bonus interview question. The important part is to distinguish between:
- Static content delivery problems.
- Network latency problems.
- Backend processing problems.
- Database access latency.
They require different solutions.