# Lesson 12: Database Transactions, Isolation Levels, and Concurrency

## 1. Why Do Transactions Matter?

Imagine a ride-booking platform where two drivers try to accept the same ride at almost exactly the same time.

The system currently has:

```text
Ride ID: R-101
Status: requested
Driver: none
```

Driver A sends an acceptance request. At nearly the same moment, Driver B sends another acceptance request.

If the application simply reads the ride, checks that it is available, and then updates the record, both requests might observe the same initial state.

The result could be:

- Two drivers assigned to one ride.
- Conflicting ride states.
- Incorrect notifications.
- A broken business workflow.

This is a concurrency problem.

**Concurrency** means multiple operations are executing or progressing during overlapping periods.

Databases need mechanisms to ensure that concurrent operations do not violate important business rules.

Transactions, isolation levels, locks, and constraints are fundamental tools for solving these problems.

---

## 2. What Is a Database Transaction?

A transaction is a group of database operations treated as one logical unit.

The goal is to ensure that the transaction's required changes are committed together or rolled back together.

Consider transferring $100 from Account A to Account B.

Initial balances:

- Account A: $500
- Account B: $200

The transfer requires two changes:

1. Subtract $100 from Account A.
2. Add $100 to Account B.

If the first operation succeeds but the second fails, the system must not leave the accounts in an inconsistent state.

Conceptually:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE account_id = 'A';

UPDATE accounts
SET balance = balance + 100
WHERE account_id = 'B';

COMMIT;
```

If a required operation fails, the application can roll back the transaction rather than committing only one side of the transfer.

The example is conceptual; production financial operations also need safeguards against insufficient balances, concurrent withdrawals, authorization errors, and other business constraints.

### What does a transaction guarantee?

Transactions provide a controlled boundary around database operations.

In PostgreSQL, transactions are designed around the ACID properties.

---

## 3. ACID Properties

ACID stands for Atomicity, Consistency, Isolation, and Durability.

### A — Atomicity

All required operations in a transaction commit together, or the transaction is rolled back.

Example: A transfer should not debit one account while failing to credit the other.

### C — Consistency

A transaction must preserve the database's defined integrity rules when moving from one valid state to another.

Examples:
- A ride must reference valid records.
- A required field must not be null.
- A unique identifier must not be duplicated.
- A business invariant must remain true.

Consistency depends on the constraints and application rules that are actually enforced. A database cannot automatically protect a business rule that has never been encoded or coordinated.

### I — Isolation

Concurrent transactions should interact according to the database's isolation rules.

Isolation determines which effects one transaction can observe from another and which concurrent outcomes are allowed.

### D — Durability

After a transaction is committed successfully, its changes should survive failures covered by the database's durability guarantees.

Actual durability depends on configuration, storage behavior, and the failure model. For example, local commit durability and survival of a complete site failure are different requirements.

**Remember:** ACID describes transaction guarantees. It does not automatically solve every distributed-system problem involving external APIs, message brokers, or separate databases.

---

## 4. The Race Condition Problem

A race condition occurs when the result depends on the timing or interleaving of concurrent operations.

Suppose one ride is available.

Two workers execute this workflow:

```text
Worker A                     Worker B
--------                     --------
Read ride: available         Read ride: available
Check availability           Check availability
Assign Driver A              Assign Driver B
Save assignment              Save assignment
```

Both workers may pass the availability check because neither has yet committed the other worker's change.

This is a classic time-of-check to time-of-use problem: the state changes, or can change, between checking a condition and acting on it.

### Why application-level checks are insufficient

Consider this logic:

```text
if ride.status == "requested":
    ride.driver_id = current_driver
    ride.status = "accepted"
    save(ride)
```

The logic appears correct when one request executes at a time.

Under concurrency, two application instances may execute the check simultaneously.

Using an in-memory lock on one application instance does not protect the record from another instance unless coordination is shared and reliable.

The business rule must be enforced at an appropriate shared boundary, commonly through a database transaction, conditional update, constraint, or lock.

---

## 5. Atomic Conditional Updates

One way to prevent duplicate assignment is to update the ride only if it is still available.

Conceptually:

```sql
UPDATE rides
SET driver_id = :driver_id,
    status = 'accepted'
WHERE ride_id = :ride_id
  AND status = 'requested';
```

The application must inspect the number of rows affected.

- One row updated: the operation successfully claimed the ride, assuming all other required business checks also passed.
- Zero rows updated: the ride was no longer in the expected state, or the record did not match the condition.

PostgreSQL coordinates concurrent updates to the same row. If two workers attempt the conditional update, the second operation will re-evaluate the relevant condition against the updated row after waiting where required. It should not successfully claim the same requested state once the first update has changed it.

### Why this is useful

The availability condition and the state change happen in one database operation.

There is no separate gap between checking the ride's state and changing it.

This is often simpler than implementing a separate read, application-level check, and write.

### Important limitation

The example solves a particular state-transition race. A real assignment workflow may also need to validate driver availability, prevent duplicate assignment records, handle cancellation, and coordinate related state changes.

Those requirements must be included in the transaction or enforced through appropriate constraints and workflow logic.

---

## 6. Unique Constraints as a Safety Net

Suppose the system stores ride assignments in a separate table.

A business rule might require each ride to have at most one active assignment.

A database uniqueness constraint can protect this invariant from concurrent application requests.

For example, a simplified schema could use:

```sql
CREATE TABLE ride_assignments (
    assignment_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ride_id BIGINT NOT NULL,
    driver_id BIGINT NOT NULL,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (ride_id)
);
```

This example assumes that the table contains at most one assignment record per ride. If the system needs assignment history, a different model may be required.

A uniqueness constraint causes conflicting inserts to be rejected rather than allowing duplicate values.

### Why constraints matter

Application validation can contain bugs or run concurrently in multiple processes.

A database constraint protects the invariant at the database boundary.

Use constraints for rules that can be represented accurately in the schema, such as:
- Uniqueness.
- Foreign-key relationships.
- Non-null requirements.
- Valid value ranges or checks.

Do not rely only on frontend validation or a prior `SELECT` query to guarantee uniqueness.

---

## 7. What Is Transaction Isolation?

Isolation defines how transactions behave when they execute concurrently.

For example, one transaction might read a ride's status while another transaction changes it.

Depending on the isolation level, the first transaction may see:
- A snapshot of the data.
- A newer committed state on a later statement.
- A serialization error requiring retry.

Different databases implement isolation levels with different details. PostgreSQL uses multiversion concurrency control (MVCC), with its own documented behavior.

The major SQL isolation levels are:

1. Read Uncommitted.
2. Read Committed.
3. Repeatable Read.
4. Serializable.

PostgreSQL accepts all four names, but its behavior for Read Uncommitted is effectively Read Committed.

---

## 8. Read Uncommitted

Read Uncommitted is intended to allow the weakest isolation behavior.

In database systems that implement it as a distinct level, a transaction may read changes made by another transaction before those changes are committed.

This is called a **dirty read**.

Example:

```text
Transaction A
--------------
Change ride status to accepted
(Not committed yet)

Transaction B
--------------
Read ride status
See uncommitted value
```

If Transaction A rolls back, Transaction B has observed a value that never became committed database state.

PostgreSQL does not permit dirty reads. A transaction using Read Uncommitted behaves like Read Committed.

This distinction matters when comparing database engines: the isolation-level name alone does not describe every implementation detail.

---

## 9. Read Committed

Read Committed is PostgreSQL's default isolation level.

A query sees rows committed before that query's snapshot was taken, subject to the database's concurrency and visibility rules.

At this level, separate statements in the same transaction can see different committed states.

### Example: Two reads, different results

Transaction A:

```sql
BEGIN;

SELECT status
FROM rides
WHERE ride_id = 101;

-- Another statement later

SELECT status
FROM rides
WHERE ride_id = 101;

COMMIT;
```

Transaction B changes the ride status and commits between the two reads.

The first query might return `requested`, while the second returns `accepted`.

This is called a **non-repeatable read**.

### Advantages

- Good general-purpose performance.
- Appropriate for many application operations.
- Short transactions often work well at this level.

### Limitations

Read Committed does not automatically guarantee that a multi-statement workflow sees an unchanged snapshot throughout the transaction.

It also does not prevent every race condition. For example, separate reads and writes can still implement an unsafe check-then-act workflow.

### When is it appropriate?

It is a common default for CRUD operations and transactional workflows that use appropriate atomic updates, constraints, and locking where needed.

The key is to design each critical operation correctly rather than assuming the isolation level alone will protect every invariant.

---

## 10. Repeatable Read

Repeatable Read provides a stable transaction snapshot in PostgreSQL.

Repeated ordinary reads of the same row generally observe the same committed version, even if another transaction later updates that row.

### Example

Transaction A:

1. Starts a Repeatable Read transaction.
2. Reads a ride with status `requested`.
3. Transaction B changes the ride to `accepted` and commits.
4. Transaction A reads the ride again.

Transaction A continues to see the version visible to its snapshot for ordinary reads.

This avoids the non-repeatable-read behavior described earlier.

### What does it protect against?

PostgreSQL Repeatable Read also prevents phantom reads in the SQL-standard sense through its snapshot-based implementation. However, it can still permit certain serialization anomalies, including write-skew patterns.

### Example: Write skew

Imagine two doctors are on call, and the rule is that at least one doctor must remain on call.

Initial state:

| Doctor | On call |
|---|---|
| A | Yes |
| B | Yes |

Two transactions start with snapshots that show both doctors on call.

- Transaction A checks that B is on call and turns A off.
- Transaction B checks that A is on call and turns B off.

If each transaction updates a different row, both might commit under a snapshot-isolation implementation such as PostgreSQL Repeatable Read.

The final state violates the business rule: neither doctor is on call.

This is a write-skew anomaly.

The lesson is important: Repeatable Read does not guarantee that every multi-row business invariant is protected from every concurrent interleaving.

Possible solutions include:
- Serializable isolation.
- Explicit locking of a shared coordination row.
- A schema constraint where the rule can be encoded.
- Another carefully designed coordination strategy.

---

## 11. Serializable Isolation

Serializable isolation aims to make concurrent transactions behave as though they executed in some serial order.

In PostgreSQL, Serializable uses Serializable Snapshot Isolation and can abort transactions when a potentially unsafe dependency pattern is detected.

### Example

Suppose a business rule requires that at least one of two resources remain available.

Two transactions simultaneously attempt to consume the last available capacity.

Serializable isolation can reject one transaction if allowing both to commit would produce a result inconsistent with any valid serial execution.

The application must handle the resulting serialization failure and, when appropriate, retry the entire transaction.

### Advantages

- Strong protection against many concurrency anomalies.
- Helps preserve complex business invariants.

### Trade-offs

- Transactions may be aborted and need retrying.
- Poorly designed long-running transactions can increase contention.
- The application must implement safe retry behavior.
- Serializable does not eliminate the need for database constraints or good transaction design.

### When should you use it?

Consider Serializable when correctness depends on complex relationships across multiple rows or predicates and lower isolation levels are insufficient.

Do not choose it solely because "stronger is always better." Evaluate the transaction pattern, expected contention, and retry requirements.

---

## 12. Isolation Levels Compared

| Behavior | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|---|---|---|---|---|
| Dirty reads in PostgreSQL | Not permitted | Not permitted | Not permitted | Not permitted |
| Non-repeatable reads | Possible in some engines; PostgreSQL behaves as Read Committed | Possible | Prevented for ordinary reads | Prevented |
| Phantom reads | Depends on engine | Possible | Prevented in PostgreSQL's implementation | Prevented |
| Write skew | Possible | Possible | Possible in PostgreSQL | Prevented when it would violate serializability |
| Transaction aborts due to serialization conflicts | Implementation-dependent | Possible for some concurrent operations | Possible | Possible and important to handle |
| General overhead | Depends on implementation | Often a good default | Snapshot and conflict considerations | May require more retries |

The table summarizes common phenomena, but exact behavior depends on the database engine. For PostgreSQL, its documented MVCC and isolation behavior is the authoritative reference when making production decisions.

---

## 13. Pessimistic Concurrency Control

Pessimistic concurrency control assumes that conflicts are possible and uses locks to coordinate access.

In PostgreSQL, a row can be locked with `SELECT ... FOR UPDATE`.

Example:

```sql
BEGIN;

SELECT ride_id, status
FROM rides
WHERE ride_id = 101
FOR UPDATE;

-- Validate the current state.
-- Perform required updates.

COMMIT;
```

The row lock prevents conflicting transactions from independently modifying the same locked row until the lock is released, subject to PostgreSQL's locking rules.

### When is it useful?

- Several operations may update the same record.
- A workflow must read and then modify a row.
- Contention is expected and the operation needs explicit coordination.
- The transaction must protect a particular record while making a decision.

### Trade-offs

- Other transactions may wait.
- Long-held locks reduce concurrency.
- Deadlocks can occur when transactions acquire multiple locks in incompatible orders.
- Lock contention can increase latency.

Keep transactions short, avoid unnecessary work while holding locks, and do not call slow external APIs inside a database transaction unless there is a carefully justified design.

---

## 14. Optimistic Concurrency Control

Optimistic concurrency control assumes conflicts are relatively uncommon.

Instead of locking a record before every read, the application checks whether the record has changed before applying its update.

A common technique is a version column.

Example record:

```text
ride_id: 101
status: requested
version: 7
```

The application reads version 7 and prepares an update.

It then executes a conditional update:

```sql
UPDATE rides
SET status = 'accepted',
    version = version + 1
WHERE ride_id = 101
  AND version = 7;
```

If one row is updated, the operation successfully applied against the expected version.

If zero rows are updated, another operation may have changed the record, or the record may no longer match the expected state. The application must reload, reject, or retry according to the business rules.

### Advantages

- Avoids holding locks throughout application-side processing.
- Works well when conflicts are uncommon.
- Makes conflicting changes detectable.

### Trade-offs

- Conflicts require handling.
- High-contention records can cause repeated retries.
- The version check must cover the data and invariant that matter.
- A version column alone does not protect an invariant involving unrelated records.

### Optimistic vs. pessimistic

| Factor | Optimistic | Pessimistic |
|---|---|---|
| Main approach | Detect conflicts when writing | Coordinate access using locks |
| Best fit | Low or moderate conflict rates | Frequent contention or explicit lock requirements |
| Conflict behavior | Reject, reload, or retry | Wait for locks or encounter lock-related errors |
| Main cost | Failed attempts and retry logic | Waiting and lock contention |
| Example | Conditional update using a version | `SELECT ... FOR UPDATE` |

Neither approach is universally better. Choose according to the workload and correctness requirements.

---

## 15. Deadlocks

A deadlock occurs when transactions wait for locks held by each other in a way that prevents progress.

Imagine two transactions updating two accounts.

Transaction A:

1. Locks Account A.
2. Attempts to lock Account B.

Transaction B:

1. Locks Account B.
2. Attempts to lock Account A.

```text
Transaction A              Transaction B
-------------              -------------
Locks Account A            Locks Account B
Waits for Account B        Waits for Account A
       |                          |
       +-------- DEADLOCK --------+
```

Neither transaction can proceed until the database resolves the situation.

PostgreSQL detects deadlocks and aborts one of the transactions so the other can continue.

### How can you reduce deadlocks?

- Acquire locks in a consistent order.
- Keep transactions short.
- Avoid unnecessary updates.
- Avoid external network calls while holding database locks.
- Use appropriate indexes so locking operations do not scan and interact with more rows than necessary.
- Monitor deadlock errors.

### Should the application retry?

A deadlock victim's transaction can often be retried if the operation is safe to repeat.

However, the application should:
- Retry the complete transaction, not just an arbitrary final statement.
- Use a bounded retry policy.
- Apply backoff where appropriate.
- Ensure repeated attempts cannot create duplicate external side effects.

---

## 16. Long-Running Transactions

Long-running transactions can create problems even when they do not deadlock.

They may:
- Hold locks longer than necessary.
- Delay other transactions.
- Prevent PostgreSQL from reclaiming some obsolete row versions.
- Increase table bloat and maintenance pressure.
- Increase the risk of timeouts and operational incidents.

Consider this workflow:

1. Start a database transaction.
2. Read a ride.
3. Call a remote payment service.
4. Wait five seconds.
5. Update the ride.
6. Commit.

Holding database locks while waiting on an external service is usually undesirable.

A better design often separates the database transaction from the external operation and uses explicit workflow states, idempotency, and recovery mechanisms.

Not every workflow can be reduced to one short transaction, but its transaction boundaries should be intentional.

---

## 17. Transactions and External Services

A PostgreSQL transaction cannot ordinarily roll back an external email, payment, or message-broker operation.

Consider:

1. Charge a payment provider.
2. Update the ride record.
3. Database transaction fails.

Rolling back the database does not automatically reverse the charge.

Similarly, publishing an event to a queue inside an ordinary database transaction does not make the queue publish atomic with the database commit.

### Common solutions

**Idempotency:** Prevent repeated requests from creating duplicate business effects.

**Transactional outbox:** Store a business change and its event record in the same database transaction, then publish the event asynchronously.

**Saga:** Coordinate multiple local transactions and compensating actions across a longer workflow.

**Explicit state machine:** Represent pending, succeeded, failed, and recovery states rather than pretending the entire workflow is one atomic operation.

The right approach depends on the business process and the guarantees provided by each external service.

---

## 18. Practical Design: Preventing Double Ride Assignment

Let's combine the concepts.

Requirement: A ride can have at most one active driver assignment.

A robust design may include:

1. A durable ride record with an explicit lifecycle state.
2. A database constraint that prevents duplicate assignments where the schema can express that invariant.
3. An atomic conditional update or a row lock for the ride-state transition.
4. A short transaction that records the assignment.
5. Idempotency for repeated requests and events.
6. Safe handling of lock timeouts, deadlocks, and serialization failures.
7. An outbox event if assignment changes must reliably trigger downstream notifications.
8. Monitoring of assignment conflicts and retry rates.

### Example flow

```text
Driver accepts ride
       |
       v
Begin database transaction
       |
       v
Claim ride atomically
       |
       +---- Claim failed ---> Return conflict / try another ride
       |
       v
Record assignment
       |
       v
Record outbox event
       |
       v
Commit transaction
       |
       v
Return assignment result
       |
       v
Publish notification asynchronously
```

The exact transaction statements depend on the schema and assignment rules. The important point is that the assignment state and its required database invariants are protected at the database boundary.

---

## 19. How to Choose an Isolation Level

Use the following decision process.

### Step 1: Identify the invariant

What must never happen?

Examples:
- Two drivers assigned to one ride.
- A balance becoming negative.
- Both resources becoming unavailable when one must remain available.

### Step 2: Identify the concurrency pattern

Is the operation modifying one row, multiple rows, or a condition involving many rows?

### Step 3: Check whether a constraint solves it

Unique constraints, foreign keys, and check constraints can protect many invariants directly.

### Step 4: Prefer a simple atomic operation where possible

Conditional updates and appropriate row locks often solve specific state-transition problems without requiring Serializable isolation.

### Step 5: Consider isolation for broader invariants

If correctness depends on multiple reads and writes across a transaction, determine whether Read Committed, Repeatable Read, or Serializable provides the required behavior.

### Step 6: Plan for contention

Estimate how often operations conflict. Consider locking, transaction duration, retries, and failure behavior.

### Step 7: Test concurrent execution

Test two or more operations attempting the same action at the same time. Verify not only the response codes but also the final persisted state.

A design is not correct merely because each request works when tested individually.

---

## 20. Common Mistakes

1. Assuming ACID automatically enforces every business rule.
2. Performing a separate availability check and update without concurrency protection.
3. Relying only on frontend validation for uniqueness.
4. Assuming Read Committed provides one stable snapshot for an entire transaction.
5. Assuming Repeatable Read prevents every concurrency anomaly.
6. Choosing Serializable without implementing safe retry behavior.
7. Holding locks while waiting for external APIs.
8. Retrying only the final statement after a transaction-level failure.
9. Ignoring deadlocks and lock timeouts.
10. Using application-instance memory to coordinate operations across multiple servers.
11. Assuming a database transaction can roll back an external payment or message publish.
12. Failing to test concurrent requests against the same record.

---

## 21. Revision Questions

Answer these without looking at the notes.

1. What are the four ACID properties?
2. What is a race condition?
3. Why is an application-level availability check insufficient for assigning a ride?
4. How does an atomic conditional update prevent double assignment?
5. What are dirty reads, non-repeatable reads, phantom reads, and write skew?
6. What does PostgreSQL Read Committed guarantee about snapshots?
7. How does Repeatable Read differ from Serializable in PostgreSQL?
8. What is pessimistic concurrency control?
9. What is optimistic concurrency control?
10. How does a version column help detect conflicting updates?
11. What is a deadlock, and how does PostgreSQL handle it?
12. Why are long-running transactions harmful?
13. Why can a database transaction not automatically roll back an external payment?
14. When would you choose Serializable isolation?
15. Design a transaction-safe ride-assignment workflow that handles concurrent requests and repeated messages.

## Final Takeaway

Transactions provide atomic database operations. Isolation controls how concurrent transactions interact. Locks, constraints, conditional updates, and version checks help preserve business invariants.

Remember:

- ACID is the foundation of reliable database transactions.
- Read Committed is PostgreSQL's default isolation level.
- Repeatable Read provides a stable snapshot but can still allow write skew.
- Serializable provides stronger protection but may require transaction retries.
- Pessimistic concurrency control coordinates access using locks.
- Optimistic concurrency control detects conflicts when applying changes.
- Deadlocks and long-running transactions must be handled deliberately.
- External side effects require additional coordination beyond ordinary database transactions.

**The central design rule:** Define the business invariant first, then select the simplest database mechanism that reliably preserves it under concurrent execution.


## Practice exercise

Imagine two drivers attempt to accept the same ride simultaneously.

1. Why can a separate SELECT followed by an UPDATE produce a race condition?
2. Would you choose an atomic conditional update, pessimistic locking, or Serializable isolation for this operation? Explain why.
3. Suppose the database commit succeeds but the API response is lost. How would you prevent a retry from creating a duplicate assignment?