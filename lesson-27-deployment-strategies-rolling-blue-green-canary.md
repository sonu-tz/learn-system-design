# Lesson 27 — Deployment Strategies: Rolling, Blue-Green, Canary, and Zero-Downtime Deployments

A production system must do more than handle normal traffic. It must continue operating when servers crash, databases become unavailable, network connections fail, or an entire data center experiences an outage.

In this lesson, we'll explore high availability, redundancy, failover, multi-AZ and multi-region architectures, disaster recovery, RTO, RPO, and backup strategies.

## 1. Why Deployment Strategy Matters

A deployment changes the software running in production. Even when the code is correct, deployment can introduce problems:

- Application instances may restart simultaneously.
- New code may be incompatible with the existing database schema.
- Frontend and backend versions may temporarily differ.
- Background workers may process old and new message formats.
- A new release may introduce unexpected latency or errors.
- Existing user sessions or long-running requests may be interrupted.

A deployment strategy determines how a new version reaches production and how the system behaves during the transition.

The goal is not simply to deploy successfully. It is to deploy with controlled risk and a reliable recovery path.

## 2. Deployment vs Release

These terms are related but different.

**Deployment:** Making a software version available in the production environment.

**Release:** Making a feature available to users.

For example:

1. Deploy application code containing a new driver-assignment feature.
2. Keep the feature disabled using a feature flag.
3. Test it internally.
4. Enable it for 5% of users.
5. Gradually enable it for everyone.

The code was deployed before the feature was released to all users.

This separation is important because it allows teams to control when users receive a feature independently of when code is deployed.

## 3. Recreate Deployment

A recreate deployment stops the existing application instances and starts new instances with the updated version.

### How it works

```text
Before deployment:

[App v1] [App v1] [App v1]
          |
        Users

During deployment:

[Stopped] [Stopped] [Stopped]
          |
     Service unavailable

After deployment:

[App v2] [App v2] [App v2]
          |
        Users
```

### Advantages

- Simple to implement.
- Easy to understand.
- No need to maintain two application versions simultaneously.
- Useful for applications where downtime is acceptable.

### Disadvantages

- Causes downtime during replacement.
- All users may experience interruptions.
- Deployment failure can leave the application unavailable.

**When to use:** Development environments, internal tools, scheduled maintenance windows, or systems where brief downtime is acceptable.

## 4. Rolling Deployment

A rolling deployment gradually replaces old application instances with new ones.

Instead of replacing every instance at once, the deployment system updates a small number at a time.

### How it works

Suppose we have four application instances.

```text
Initial state:

[ v1 ] [ v1 ] [ v1 ] [ v1 ]

Step 1:

[ v2 ] [ v1 ] [ v1 ] [ v1 ]

Step 2:

[ v2 ] [ v2 ] [ v1 ] [ v1 ]

Step 3:

[ v2 ] [ v2 ] [ v2 ] [ v1 ]

Step 4:

[ v2 ] [ v2 ] [ v2 ] [ v2 ]
```

Traffic continues to reach healthy instances while the remaining instances are updated.

### Important configuration

Rolling deployments commonly use two settings:

- **maxSurge:** Maximum number of additional instances allowed above the desired replica count during deployment.
- **maxUnavailable:** Maximum number of instances that may be unavailable during deployment.

Example:

Desired replicas = 4

```text
maxSurge = 1
maxUnavailable = 1
```

The deployment system can temporarily run up to five instances, while allowing at most one desired instance to be unavailable.

Exact behavior depends on the deployment platform and its rollout configuration.

### Readiness checks

A new instance should receive production traffic only after it is ready.

For example:

```text
Start new instance
       |
       v
Application initializes
       |
       v
Database connection established
       |
       v
Readiness check passes
       |
       v
Instance receives traffic
```

A process being alive does not necessarily mean it is ready to serve requests.

### Advantages

- Usually avoids full application downtime.
- Gradual replacement reduces deployment risk.
- Uses fewer resources than maintaining two complete environments.
- Supported by platforms such as Kubernetes.

### Disadvantages

- Old and new versions run simultaneously.
- Both versions must be compatible with the same database and shared dependencies.
- Rollback can be complicated if new instances have already changed shared data.
- Deployment takes longer than replacing everything at once.

**Important:** A rolling deployment is safe only when old and new application versions can coexist.

## 5. Blue-Green Deployment

Blue-green deployment maintains two production-like environments.

- **Blue:** Current production version.
- **Green:** New application version.

Only one environment receives normal production traffic at a time.

### How it works

```text
                  Load Balancer
                        |
                  Traffic Switch
                   /         \
                  /           \
             Blue v1         Green v2
             Active          Inactive
```

Initially, Blue handles production traffic.

The new version is deployed and tested in Green. Once it is ready, traffic is switched from Blue to Green.

```text
Before:

Users ---> Blue v1
          Green v2 (testing)

After:

Users ---> Green v2
          Blue v1 (standby)
```

### Rollback

If Green introduces a critical problem:

```text
Users ---> Blue v1
          Green v2
```

Traffic can be switched back to Blue, assuming Blue remains compatible with the current database and shared state.

### Advantages

- Fast traffic switching.
- Easy application-level rollback.
- New version can be tested before receiving production traffic.
- Clear separation between old and new environments.

### Disadvantages

- Requires additional infrastructure capacity.
- Shared database changes can prevent a simple rollback.
- Background workers and scheduled jobs require careful coordination.
- Switching traffic does not automatically migrate existing connections.

### Important consideration

Blue-green deployment does not mean the database should also be duplicated and switched blindly.

Consider this situation:

1. Blue uses database schema v1.
2. Green uses schema v2.
3. Green writes data using the new schema.
4. You switch traffic back to Blue.
5. Blue cannot understand the data written by Green.

The application rollback may succeed while the overall system remains broken.

Database compatibility must be planned separately.

## 6. Canary Deployment

A canary deployment exposes the new version to a small portion of production traffic before expanding the rollout.

The objective is to detect problems using real production traffic while limiting the number of affected users.

### How it works

```text
Initial:

100% traffic ---> v1

Canary:

95% traffic  ---> v1
 5% traffic  ---> v2

Next stage:

75% traffic  ---> v1
25% traffic  ---> v2

Next stage:

50% traffic  ---> v1
50% traffic  ---> v2

Final:

100% traffic ---> v2
```

The rollout progresses only when the new version meets the defined health criteria.

### Example

Suppose you release a new driver-assignment algorithm.

You begin with 5% of eligible traffic.

Compare the new version against the existing version:

| Metric | Existing v1 | Canary v2 |
|---|---:|---:|
| Error rate | 0.2% | 0.3% |
| P95 latency | 180 ms | 190 ms |
| Assignment success | 98.5% | 98.4% |

These numbers may be acceptable depending on the predefined release thresholds.

However, suppose the canary shows:

| Metric | Existing v1 | Canary v2 |
|---|---:|---:|
| Error rate | 0.2% | 3.5% |
| P95 latency | 180 ms | 850 ms |
| Assignment success | 98.5% | 90% |

The rollout should stop, and the team should investigate or roll back the canary.

### Progressive rollout

A possible rollout schedule:

1. 5% traffic for 10 minutes.
2. 10% traffic for 15 minutes.
3. 25% traffic for 20 minutes.
4. 50% traffic for 30 minutes.
5. 100% traffic after validation.

These durations are examples, not universal rules. They should reflect traffic volume, risk, and the time needed to observe meaningful behavior.

### What should you monitor?

- Error rate.
- P95 and P99 latency.
- CPU and memory utilization.
- Database load.
- Queue backlog.
- Business metrics, such as successful bookings or driver assignments.

### Challenges

**Low traffic:** A 5% canary may receive too few requests to reveal problems.

**Traffic imbalance:** The canary may receive a different type of workload from the stable version.

**Delayed failures:** Some problems appear only after hours or days.

**Statistical confidence:** A small number of successful requests does not prove that a release is safe.

**When to use:** High-risk changes, performance-sensitive features, algorithm changes, and large production releases.

## 7. Feature Flags

Feature flags allow you to control whether a feature is enabled without deploying another application version.

Example:

```text
Application deployed
        |
        v
Feature flag checked
        |
    +---+---+
    |       |
  Enabled Disabled
    |       |
 New flow Old flow
```

### Common types

**Release flags:** Hide unfinished features until they are ready.

**Operational flags:** Disable expensive or problematic functionality during an incident.

**Experiment flags:** Divide users into groups to evaluate different experiences.

**Permission or entitlement controls:** Determine which customers are entitled to a product capability, although actual authorization must still be enforced securely on the server.

### Example

Suppose you deploy a new trip-list implementation.

Initially:

```text
new_trip_list = false
```

After QA:

```text
new_trip_list = true
```

You can also enable it for a specific tenant or internal user group.

### Kill switches

A kill switch is an operational feature flag that disables a feature quickly when it causes problems.

For example, if a new real-time tracking implementation increases server load, you can disable it without deploying another release.

### Feature flag lifecycle

Feature flags introduce complexity if they remain in the code forever.

A healthy lifecycle is:

1. Create the flag.
2. Deploy the code.
3. Test with the flag disabled.
4. Enable gradually.
5. Monitor results.
6. Enable for everyone.
7. Remove the flag and obsolete code.

### Security warning

A frontend feature flag is not an authorization mechanism.

Hiding an admin button does not prevent a user from calling the admin API directly.

The backend must independently validate permissions.

## 8. Shadow Traffic and Dark Launches

Shadow traffic sends copies of production requests to a new version without using the new version's response for the actual user experience.

```text
                Production Request
                       |
                 Traffic Router
                  /          \
                 /            \
            Existing v1     Shadow v2
                 |              |
           Real response    Discard response
                 |
               User
```

The existing version continues to serve users.

The shadow version processes copied traffic so the team can compare:

- Response latency.
- Error rates.
- Query performance.
- Calculated results.
- Resource consumption.

### Important safety consideration

Shadow requests must not accidentally perform real side effects.

For example, a shadow booking request must not create another booking or send a duplicate notification.

Use read-only execution, isolated dependencies, side-effect suppression, or carefully designed test data.

**When to use:** Replacing a critical algorithm, migrating search infrastructure, or validating a new read path against real production workloads.

## 9. Canary Deployment vs A/B Testing

These concepts may look similar, but their goals differ.

| Area | Canary deployment | A/B testing |
|---|---|---|
| Goal | Validate release health | Compare user outcomes |
| Focus | Reliability and performance | Product behavior |
| Success metrics | Errors, latency, saturation | Conversion, engagement |
| Decision | Continue or roll back | Choose better variant |
| Duration | Usually release window | May run longer |

Example:

- Canary: Does the new checkout service increase error rates?
- A/B test: Does a new checkout layout improve conversion?

Both can use traffic splitting, but they answer different questions.

## 10. Zero-Downtime Deployments

A zero-downtime deployment aims to keep the service available while new application versions are introduced.

It requires more than choosing rolling or blue-green deployment.

Several operational details must work together.

### A. Graceful shutdown

When an old application instance is replaced, it should stop accepting new traffic while allowing existing requests to finish.

```text
Deployment system sends SIGTERM
              |
              v
Application becomes unready
              |
              v
Load balancer stops new requests
              |
              v
Existing requests finish
              |
              v
Application closes connections
              |
              v
Process exits
```

If the process is killed immediately, users may receive failed requests.

### B. Connection draining

Long-lived connections require special handling.

Examples:

- WebSocket connections.
- Server-Sent Events.
- Long polling.
- Streaming HTTP responses.

The old instance should stop accepting new connections and allow existing connections to close gracefully or reconnect to a healthy instance.

For real-time systems, reconnect logic and subscription restoration are especially important.

### C. In-flight requests

A deployment must account for requests that are still processing.

For example:

- A booking creation request.
- A payment operation.
- A large report generation request.

If an instance shuts down before completion, the client may retry.

This is why idempotency and request deadlines matter.

### D. Background workers

Workers also need graceful shutdown.

A worker should:

1. Stop accepting new messages.
2. Finish processing its current message, or safely release it.
3. Acknowledge successfully processed messages.
4. Close broker connections.
5. Exit.

Otherwise, deployment can cause duplicate processing or lost work.

### E. Readiness vs liveness

These checks have different purposes.

| Check | Purpose |
|---|---|
| Liveness | Is the application process healthy enough to continue running? |
| Readiness | Should this instance receive traffic? |

An instance can be alive but not ready.

For example, it may still be loading configuration or establishing required connections.

## 11. Database Migrations During Deployments

Database migrations are one of the most important challenges in zero-downtime deployments.

Consider an application changing a column:

```text
Old schema:
users.name

New schema:
users.first_name
users.last_name
```

If the database column is renamed immediately, old application instances may fail while the rolling deployment is still in progress.

The solution is often the **expand-and-contract migration strategy**.

### Phase 1: Expand

Add the new schema without removing the old schema.

```text
users
-----
name
first_name
last_name
```

The old application continues using `name`.

### Phase 2: Migrate data

Backfill existing data:

```text
name = "Sonu Kumar"

first_name = "Sonu"
last_name  = "Kumar"
```

Backfills should be designed to run safely in batches and be repeatable where possible.

### Phase 3: Transition application

Deploy a version that understands both representations.

It may read the new fields while temporarily maintaining compatibility with the old application.

If old and new versions both write data, define exactly how consistency is maintained. Dual writes can introduce race conditions and should not be added casually.

### Phase 4: Validate

Confirm:

- New fields contain valid data.
- Old application versions are no longer running.
- No important consumer still depends on the old schema.
- Monitoring shows no unexpected errors.

### Phase 5: Contract

Remove the old column only after it is no longer needed.

```text
users
-----
first_name
last_name
```

### Important principle

Database migrations should be backward-compatible with application versions that may still be running.

A deployment rollback does not necessarily mean a database rollback.

For destructive changes, a forward fix may be safer than trying to reverse the migration.

## 12. API and Event Schema Compatibility

Compatibility is not limited to the database.

A system may have multiple application versions communicating during a deployment.

For example:

```text
Service A v1 ---> Service B v1
Service A v2 ---> Service B v1
Service A v2 ---> Service B v2
```

The transition should not break communication.

### API compatibility

Prefer additive changes:

- Add optional response fields.
- Add optional request fields.
- Preserve existing field meanings.
- Avoid changing response types unexpectedly.
- Avoid removing fields immediately.

### Event compatibility

Suppose a Kafka event originally contains:

```json
{
  "bookingId": "B123",
  "status": "ASSIGNED"
}
```

A new version adds a field:

```json
{
  "bookingId": "B123",
  "status": "ASSIGNED",
  "driverId": "D456"
}
```

Old consumers should continue to work if they ignore unknown fields.

Be careful with:

- Renaming fields.
- Changing data types.
- Removing fields.
- Changing enum values.
- Changing event semantics.

A schema registry and compatibility rules can help enforce safe evolution.

## 13. Frontend and Backend Deployment Compatibility

Frontend applications create another deployment challenge.

Suppose:

1. Backend v2 is deployed.
2. Frontend v2 is deployed.
3. A user still has frontend v1 loaded in their browser.

The old frontend may continue making API requests for some time.

Therefore, backend APIs should remain compatible with older frontend clients during the transition.

### Static asset caching

Use content-hashed filenames:

```text
main.a83f91.js
styles.92bd31.css
```

When the frontend changes, the filename changes too.

This allows long-lived CDN caching without accidentally serving stale content under a new version identifier.

Keep previous assets available long enough for users with older application sessions to load any remaining lazy-loaded chunks.

## 14. CI/CD Deployment Pipeline

A typical production pipeline looks like this:

```text
Developer pushes code
         |
         v
       Build
         |
         v
    Automated tests
         |
         v
   Security checks
         |
         v
  Create immutable artifact
         |
         v
   Deploy to staging
         |
         v
   Integration tests
         |
         v
  Production deployment
         |
         v
   Health verification
         |
         v
   Gradual rollout
         |
         v
      Complete
```

### Important practices

**Immutable artifacts:** Build once and promote the same artifact through environments rather than rebuilding different binaries for production.

**Automated tests:** Include unit, integration, contract, and smoke tests.

**Approvals:** Use appropriate approval gates for sensitive production changes.

**Deployment health checks:** Verify application readiness and key system metrics.

**Auditability:** Track which version was deployed, by whom, when, and with which configuration.

**Rollback readiness:** Know how to stop a rollout and restore the previous application version.

## 15. Rollback Strategy and Release Health

A rollback should be based on predefined signals, not just subjective judgment.

Example release policy:

| Signal | Example rollback condition |
|---|---|
| Error rate | Significant increase above baseline |
| P95 latency | Sustained increase beyond release threshold |
| CPU/memory | Unexpected saturation |
| Queue lag | Growing backlog |
| Business metric | Booking success rate drops |
| Database | Query latency or connection usage spikes |

The exact thresholds should be based on the system's SLOs, traffic volume, and normal operating behavior.

### Rollback vs roll-forward

**Rollback:** Return to the previous application version.

**Roll-forward:** Deploy a new version that fixes the problem.

Rollback is useful when the previous version remains compatible with the current state.

Roll-forward may be necessary when:

- Database changes cannot safely be reversed.
- New data has already been written.
- External systems have processed irreversible operations.
- The previous version cannot understand the new state.

A good release process plans for both.

## 16. GitOps and Progressive Delivery

### GitOps

GitOps uses version-controlled configuration as the desired state of infrastructure and deployments.

For example:

```text
Git repository
     |
     v
Deployment configuration
     |
     v
GitOps controller
     |
     v
Kubernetes cluster
```

A controller continuously compares the desired state in Git with the actual cluster state and reconciles differences.

Benefits include:

- Auditable deployment changes.
- Reproducible environments.
- Easier configuration review.
- Reduced manual deployment steps.

### Progressive delivery

Progressive delivery extends deployment automation with controlled rollout decisions.

For example:

```text
Deploy canary
      |
      v
Evaluate metrics
      |
  +---+---+
  |       |
Healthy  Unhealthy
  |       |
Increase  Rollback
traffic
  |
  v
Full release
```

This is where deployment systems, feature flags, and observability work together.

## 17. Choosing the Right Deployment Strategy

| Strategy | Downtime | Infrastructure cost | Rollback | Typical use |
|---|---|---|---|---|
| Recreate | Usually yes | Low | Restart old version | Development or maintenance |
| Rolling | Usually none if designed correctly | Low to moderate | More complex | Regular application releases |
| Blue-green | Usually none if designed correctly | High | Fast traffic switch | Critical applications |
| Canary | Usually none if designed correctly | Moderate | Stop or reverse traffic | High-risk releases |
| Shadow | Existing service stays active | Additional processing cost | Stop shadow traffic | Validation of new implementations |

No strategy eliminates every deployment risk.

For example, blue-green can provide fast application rollback, but it cannot automatically undo database writes or external side effects.

The right choice depends on the application's availability requirements, infrastructure budget, compatibility requirements, and operational maturity.

## 18. Worked Example: Releasing a Ride-Booking Feature

Suppose we are releasing a new driver-assignment algorithm.

### Requirements

- Existing booking flow must remain available.
- New algorithm should be tested with real traffic.
- Assignment failures must not increase.
- Rollback should be possible without a complete redeployment.

### Proposed approach

**Step 1 — Deploy**

Deploy the new algorithm behind a feature flag.

**Step 2 — Internal validation**

Enable it for internal test accounts and verify the assignment flow.

**Step 3 — Canary**

Enable it for 5% of eligible production bookings.

**Step 4 — Compare**

Monitor:

- Assignment success rate.
- Assignment latency.
- Driver availability query latency.
- Database load.
- Booking failure rate.

**Step 5 — Progressive rollout**

Increase exposure to 10%, 25%, 50%, and then 100% if release criteria remain satisfied.

**Step 6 — Recovery**

If assignment failures increase, disable the flag or route traffic back to the old implementation.

**Step 7 — Cleanup**

After the new implementation is stable, remove the old code and feature flag.

### What else must be checked?

- Does the new implementation require database migrations?
- Are old and new event consumers compatible?
- Can existing booking requests finish during deployment?
- Can workers safely shut down?
- Can the old application understand data written by the new version?

This is why deployment strategy is an end-to-end system design concern rather than simply a CI/CD configuration choice.

## 19. Common Deployment Mistakes

1. Deploying every instance simultaneously when downtime is unacceptable.
2. Sending traffic to instances before they pass readiness checks.
3. Assuming application rollback also reverses database changes.
4. Removing database columns while old application versions are still running.
5. Forgetting that frontend clients can remain open for hours or days.
6. Deploying incompatible event schemas.
7. Ignoring WebSocket and long-polling connections during shutdown.
8. Retrying interrupted operations without idempotency.
9. Using canary traffic percentages without collecting enough meaningful data.
10. Keeping feature flags permanently.
11. Allowing shadow requests to trigger real side effects.
12. Declaring a deployment successful merely because all instances started.

## 20. Practical Deployment Checklist

### Before deployment

- [ ] Define deployment and rollback strategy.
- [ ] Verify database migration compatibility.
- [ ] Check API and event schema compatibility.
- [ ] Build and test the immutable artifact.
- [ ] Define release health thresholds.
- [ ] Confirm monitoring and alerts are working.
- [ ] Verify configuration and feature flags.

### During deployment

- [ ] Verify readiness before sending traffic.
- [ ] Monitor error rates and latency.
- [ ] Monitor database and queue health.
- [ ] Check business success metrics.
- [ ] Stop rollout if predefined thresholds are breached.
- [ ] Ensure old instances shut down gracefully.

### After deployment

- [ ] Confirm all instances run the expected version.
- [ ] Verify critical business workflows.
- [ ] Check frontend and backend compatibility.
- [ ] Confirm background workers are healthy.
- [ ] Document deployment results.
- [ ] Remove obsolete feature flags when safe.

## 21. Revision Questions

1. What is the difference between deployment and release?
2. How does a rolling deployment work?
3. What are `maxSurge` and `maxUnavailable`?
4. Why must old and new versions be compatible during a rolling deployment?
5. How does blue-green deployment work?
6. Why can a database migration make blue-green rollback unsafe?
7. What is a canary deployment?
8. Why is a 5% canary not always statistically meaningful?
9. What is the difference between canary deployment and A/B testing?
10. What is the purpose of a feature flag?
11. Why should frontend feature flags never replace backend authorization?
12. What is shadow traffic, and why must side effects be controlled?
13. What is graceful shutdown?
14. What is the difference between readiness and liveness?
15. Explain the expand-and-contract migration strategy.
16. Why should API and event schemas remain backward-compatible?
17. What is the difference between rollback and roll-forward?
18. How can a deployment affect WebSocket connections and background workers?

## 22. Practical Exercise

Imagine you are deploying a new version of a production ride-booking application.

Current environment:

- 6 application instances.
- 3 background workers.
- PostgreSQL primary with one read replica.
- Kafka for asynchronous events.
- Angular frontend served through a CDN.
- Existing users may keep the application open for several hours.

The new release includes:

- A new booking-assignment algorithm.
- A new optional field in the booking API response.
- A new Kafka event field.
- A database column required by the new algorithm.
- A new Angular UI for driver assignment.

Design a safe deployment plan.

Your plan should explain:

1. Which deployment strategy would you choose and why?
2. In what order would you deploy the database, backend, workers, and frontend?
3. How would you handle old and new application versions running together?
4. How would you gradually enable the new algorithm?
5. Which metrics would trigger a rollback?
6. How would you handle a rollback after the new database column has been populated?
7. How would you ensure existing users and real-time connections are not disrupted?

**Key takeaway:** A safe deployment is not just about starting a new version. It is about maintaining compatibility, controlling traffic, observing production behavior, and having a recovery strategy for every layer of the system.

### Practice focus

Work through the ride-booking deployment exercise. Pay particular attention to deployment order and database backward compatibility—these are common sources of production deployment failures.