# Distributed Transactions

How to maintain consistency across multiple services and databases when a single ACID transaction is not possible.

---

## Case Study 1: Order Flow Failure Across Services

### Problem

A user places an order. The order service creates the order, the payment service charges the card, then the inventory service fails to reserve stock. The order and payment have already been committed in their own databases. You now have a charged customer with no reserved inventory and no clear way to "roll back" the payment from the order service.

**Context**: Core microservices problem - affects Amazon, Uber, Netflix, any multi-service order/payment flow

**In simple terms**: Imagine booking a flight, hotel, and car in three different systems. You pay for the flight and hotel, then the car rental system is down. You can't "undo" the flight and hotel in one click—each system has already committed. You need a coordinated way to either finish the flow or undo what was done.

Distributed transactions are the same: each service has its own database; there is no single "rollback" across all of them.

### Quick Answer

Use the Saga pattern: break the flow into local transactions (one per service). If any step fails, run compensating transactions in reverse order to undo previous steps. No distributed locks—each service commits its own work, and compensations fix the rest.

### Detailed Explanation

#### Why This Happens (Root Cause)

In a monolith, one database and one transaction manager give ACID: all-or-nothing. In microservices:

```
Order Service  → Order DB (committed)
Payment Service → Payment DB (committed)
Inventory Service → Inventory DB (fails)
```

Each service owns its data. There is no global coordinator that can atomically commit or abort across all three. So partial success (order + payment, no inventory) is possible.

**Root cause**: No shared transaction context across service boundaries. 2PC (two-phase commit) exists but is avoided at scale due to locking, coordinator failure, and latency.

#### The Solution Approach

**High-level strategy:**
- Model the flow as a sequence of local transactions (one per service).
- Each step does one thing and publishes an event or calls the next step.
- If a step fails, run compensating actions in reverse order (e.g. refund payment, cancel order).
- No global lock; eventual consistency via compensations.

**Architecture Decision: Saga Pattern**

**Two styles:**

**Choreography (event-driven):**
- Each service performs its step and publishes an event.
- Next service subscribes and runs its step (and may publish another event).
- On failure, the failing service (or a listener) publishes a compensation event; previous services react and compensate.

**Orchestration (central coordinator):**
- A central orchestrator (workflow engine) calls each service in sequence.
- It tracks state (pending, completed, failed) and, on failure, calls compensations in reverse order.

**How it works conceptually (orchestration example):**
1. Orchestrator: Create order → Order Service commits, returns success.
2. Orchestrator: Charge payment → Payment Service commits, returns success.
3. Orchestrator: Reserve inventory → Inventory Service fails.
4. Orchestrator: Run compensations in reverse: Refund payment → Cancel order (or mark order failed).

**Conceptual flow:**
```
Order → Payment → Inventory
  ✓        ✓         ✗
  ↑        ↑         │
  └────────┴─────────┘
  Compensate: Refund, then cancel order
```

**Key Design Decisions:**
- **Compensations must be idempotent**: Retries can run the same compensation multiple times.
- **Compensations are business operations**: Refund, cancel, release—not low-level DB rollback.
- **Orchestration vs choreography**: Orchestration gives a single place to track progress and compensate; choreography avoids a central bottleneck but spreads logic.
- **Order of compensations**: Reverse order of forward steps so dependencies are undone correctly.

#### Production Results

| Metric | Before (No Saga) | After (Saga) | Improvement |
|--------|------------------|-------------|-------------|
| Orphaned payments (no order) | 0.5% of attempts | 0% | Eliminated |
| Stuck "pending" orders | 2% | 0.1% | 20x reduction |
| Time to detect and fix inconsistency | Hours | Seconds (automated) | Orders of magnitude |
| Consistency (order ↔ payment ↔ inventory) | 97% | 99.9% | Measurable improvement |

**Key insight**: Sagas trade strict ACID for practical consistency: you accept temporary inconsistency and fix it with compensations, avoiding distributed locks and coordinator bottlenecks.

**Lessons Learned**:
- Design every forward step with a clear compensation before you ship.
- Use idempotency keys for compensations (refund, cancel) so retries are safe.
- Prefer orchestration when you need a clear audit trail and simpler reasoning; choreography when you want loose coupling and no single point of coordination.
- Monitor saga completion and compensation rates; alert on stuck or failing sagas.

---

## Case Study 2: Choreography vs Orchestration

### Problem

You need to implement a Saga. Should services react to events and run steps (choreography), or should a central workflow engine call each service (orchestration)? Each choice has different trade-offs for complexity, visibility, and scaling.

**Context**: Architectural decision - Netflix often uses choreography; Amazon uses orchestration (e.g. Step Functions, internal workflow engines)

**In simple terms**: Choreography is like a dance where each dancer reacts to others without a conductor. Orchestration is like a conductor telling each section when to play. One is decentralized and flexible; the other is centralized and easier to reason about.

### Quick Answer

Use orchestration when you need a single place to track progress, enforce order, and run compensations—better for compliance and debugging. Use choreography when you want no single point of coordination and are okay with logic spread across services and event handlers.

### Detailed Explanation

#### Why This Matters

**Choreography:**
- Services subscribe to events (e.g. "OrderCreated", "PaymentCompleted").
- Each service does its work and emits the next event.
- No central state; flow is implicit in event subscriptions.
- Hard to see "where is this order in the flow?" and to trigger compensations in a defined order.

**Orchestration:**
- One component (orchestrator) holds workflow state and calls services in sequence.
- On failure, it calls compensations in reverse order.
- Easy to visualize and debug; one place to add retries, timeouts, and alerts.
- Orchestrator can become a bottleneck if not scaled or sharded.

#### The Solution Approach

**Comparison:**

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Coupling | Loose (event contracts only) | Tighter (orchestrator knows all steps) |
| Visibility | Distributed (logs/events) | Central (orchestrator state) |
| Compensation | Each service reacts to failure events | Orchestrator runs compensations in order |
| Complexity | Hidden in many handlers | Concentrated in one workflow definition |
| Scaling | Each service scales independently | Orchestrator must scale (e.g. per workflow shard) |

**When to use orchestration:**
- Order/payment/fulfillment flows with strict order and clear compensations.
- Need for audit trail ("what happened for this order?").
- Regulatory or compliance requirements for a clear sequence of steps.

**When to use choreography:**
- High-throughput, event-driven pipelines (e.g. stream processing, notifications).
- Many independent consumers of the same events.
- Teams that own services and prefer owning their own reaction to events.

**Hybrid:** Some systems use an orchestrator for the critical path (order → payment → inventory) and events for side effects (email, analytics, cache updates).

#### Production Results

| Metric | Choreography | Orchestration |
|--------|-------------|---------------|
| Time to debug failed flow | Higher (trace events across services) | Lower (one workflow view) |
| Compensation correctness | Depends on event ordering and handlers | Explicit and ordered |
| New step added | New subscriber/handler | Update workflow definition |
| Operational visibility | Event bus + tracing | Workflow engine dashboard |

**Key insight**: For money and inventory, orchestration usually wins for clarity and correct compensations. For non-critical or high-fanout flows, choreography can be simpler and more scalable.

**Lessons Learned**:
- Choose based on who needs to see the full flow and who runs compensations.
- Orchestration improves with workflow engines (Temporal, Cadence, AWS Step Functions, etc.) that handle retries, timeouts, and state.
- In choreography, design failure events and compensation handlers as first-class; otherwise inconsistencies linger.

---

## Case Study 3: Compensating Transaction Failures

### Problem

A Saga step fails and you run a compensation (e.g. refund). The compensation itself fails (e.g. payment gateway timeout). You retry, but now you might double-refund or leave the system in an inconsistent state.

**Context**: Real production issue - compensations must be as robust as forward steps

**In simple terms**: You cancel a hotel booking and the cancellation system is down. You retry. You need to be sure you don't cancel twice or leave "refund pending" forever. Compensations need the same care as the original operations: idempotency, timeouts, and clear final state.

### Quick Answer

Make compensations idempotent (e.g. compensation key or idempotency key per saga instance + step). Retry with backoff. If compensation cannot succeed (e.g. external system permanently failed), persist the failure and alert; use manual or scheduled reconciliation to fix.

### Detailed Explanation

#### Why This Matters

**Failure modes:**
- Compensation call times out (network, downstream slow).
- Compensation returns 5xx or "try again."
- Compensation succeeds but response is lost; orchestrator retries and may double-compensate.
- Downstream service is down for hours; compensation cannot complete.

If compensations are not idempotent, retries can cause double refunds or double releases. If there is no failure handling, failed compensations leave the system stuck.

#### The Solution Approach

**High-level strategy:**
- Treat each compensation as a single logical operation with a unique key (e.g. saga_id + step_id).
- Idempotency: first execution performs the work and stores result; retries return the same result.
- Retry with exponential backoff and a max attempts.
- After max failures, mark as "compensation failed" and alert; run reconciliation (e.g. nightly job) or manual resolution.

**Idempotent compensation pattern:**
1. Orchestrator decides to compensate step 2 (e.g. RefundPayment).
2. Sends RefundPayment(saga_id="abc", step=2) to Payment Service.
3. Payment Service: "Have I already processed RefundPayment for saga abc, step 2?" If yes, return previous result. If no, perform refund, store result, return.
4. Orchestrator retries on timeout; Payment Service returns same result on retry—no double refund.

**Handling permanent failure:**
- Store failed compensations in a "saga_failures" or "compensation_backlog" table.
- Alert and dashboard for open compensations.
- Reconciliation job or manual process to retry or resolve (e.g. manual refund, ticket to finance).

**Key Design Decisions:**
- **Idempotency keys for compensations**: Same idea as payment idempotency—saga_id + step (and maybe attempt id) as key.
- **Timeout and max retries**: Avoid infinite retries; fail open to human or batch reconciliation.
- **Observability**: Metrics and logs for compensation attempts, successes, and failures.

#### Production Results

| Metric | Before (Non-Idempotent) | After (Idempotent + Retry) | Improvement |
|--------|--------------------------|----------------------------|-------------|
| Double refunds from retries | 0.1% of compensations | 0% | Eliminated |
| Stuck sagas (compensation never run) | 5% | 0.5% | 10x reduction |
| Time to resolve failed compensation | Ad hoc | Defined (alert + reconciliation) | Predictable |

**Key insight**: Compensations are not "best effort"—they need the same design as critical operations: idempotent, observable, and with a path to resolution when they cannot complete.

**Lessons Learned**:
- Every compensation should be idempotent and keyed by saga + step (and optionally attempt).
- Define max retries and then hand off to reconciliation or manual process.
- Monitor compensation success rate and backlog; treat failed compensations as P1 incidents when they affect money or inventory.

---

## Case Study 4: Why Not Two-Phase Commit (2PC)?

### Problem

Why don't companies use two-phase commit (2PC) across microservices to get real ACID and avoid Sagas and compensations?

**Context**: Classic distributed systems question - 2PC exists but is rarely used at scale for cross-service transactions

**In simple terms**: 2PC is like a wedding where everyone must say "I do" at the same time. If one person is unreachable, everyone waits. In systems, that means locks are held across the network; one slow or failed participant blocks or aborts everyone. At scale, that causes timeouts, deadlocks, and poor availability.

### Quick Answer

2PC requires all participants to lock resources and agree before any commit. That causes long-held locks, coordinator single point of failure, and poor availability under partition or slow participants. Sagas avoid distributed locks and keep each service's transaction short; consistency is achieved by compensations and eventual consistency.

### Detailed Explanation

#### Why 2PC Is Problematic at Scale

**How 2PC works conceptually:**
1. Coordinator sends "prepare" to all participants (Order, Payment, Inventory).
2. Each participant does local work, prepares to commit, and holds locks. Replies "yes" or "no."
3. If all say "yes," coordinator sends "commit"; everyone commits and releases locks.
4. If any says "no" or times out, coordinator sends "abort"; everyone rolls back.

**Problems:**
- **Locks held during prepare**: From prepare until commit/abort, resources are locked. Slow or failed participant extends lock duration for everyone.
- **Coordinator failure**: If coordinator dies after "prepare" but before "commit," participants can stay in "prepared" state; recovery is complex.
- **Network partitions**: During partition, some participants cannot reach coordinator; they may abort or wait, leading to inconsistent outcomes or long blocks.
- **Latency**: Two round-trips (prepare + commit) plus all participants must respond; P99 latency grows with the slowest service.

**Why Sagas are used instead:**
- No distributed lock: each service commits its own transaction when its step runs.
- No single coordinator that must be up for commit: orchestrator can be replicated; choreography has no coordinator.
- Short local transactions: each step commits quickly; failures are handled by compensations.
- Trade-off: temporary inconsistency and need for compensations and idempotency.

#### When 2PC Might Be Acceptable

- Small number of participants (e.g. 2 databases in same datacenter).
- Short transactions and low contention.
- When strong consistency is mandatory and compensations are not acceptable (e.g. some financial regulatory contexts).
- Often used inside a single domain (e.g. one service with two databases) rather than across many microservices.

#### Production Results

| Aspect | 2PC (Cross-Service) | Saga |
|--------|---------------------|------|
| Availability under failure | Lower (blocked or aborted) | Higher (each service independent) |
| Latency (P99) | Higher (prepare + commit) | Lower (no global phase) |
| Consistency | Strong (if all commit) | Eventual (with compensations) |
| Operational complexity | Coordinator, recovery | Compensations, idempotency |

**Key insight**: At microservices scale, 2PC's locking and coordination cost usually outweigh its benefits. Sagas + idempotent compensations are the standard for cross-service flows.

**Lessons Learned**:
- Prefer Sagas for order, payment, inventory, and other multi-service flows.
- Use 2PC only in narrow, same-domain scenarios where strong consistency is required and participants are few and fast.
- Design for failure: assume steps and compensations will timeout and retry; make them idempotent and observable.

---

## Common Mistakes

1. **Mistake**: No compensation designed for a step
   **Why it's wrong**: When the step fails or a later step fails, you cannot undo; you get stuck or inconsistent state.
   **Instead**: For every forward step, define and implement a compensating action before production.

2. **Mistake**: Compensations that are not idempotent
   **Why it's wrong**: Retries (timeouts, 5xx) can double-refund or double-release.
   **Instead**: Use saga_id + step (and optionally attempt) as idempotency key for compensations.

3. **Mistake**: Using 2PC across many microservices
   **Why it's wrong**: Locks, coordinator dependency, and latency hurt availability and performance.
   **Instead**: Use Saga (orchestration or choreography) with compensations.

4. **Mistake**: Choreography with no clear compensation path
   **Why it's wrong**: Failure events may be lost or processed out of order; compensations may not run or may run in wrong order.
   **Instead**: Design failure and compensation events explicitly; consider orchestration for money/inventory flows.

5. **Mistake**: Ignoring failed compensations
   **Why it's wrong**: Orphaned payments, stuck reservations, or inconsistent state.
   **Instead**: Alert on failed compensations; have reconciliation or manual process to resolve.

---

## Related Patterns

- **[Idempotency Patterns](../../04_API_Design/idempotency_patterns/README.md)** - Idempotent steps and compensations
- **[Event-Driven Architecture](../../05_Event_Driven_Architecture/kafka_patterns.md)** - Choreography and event bus
- **[Cache Consistency](../cache_consistency/README.md)** - Consistency in caching vs. source of truth
- **[Exactly Once Processing](../../05_Event_Driven_Architecture/exactly_once_processing.md)** - At-most-once vs exactly-once in event processing

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for:
- Engineering blog posts (Netflix, Amazon, Uber)
- Conference talks and workflow engines (Temporal, Step Functions)
- Papers on Saga and distributed transactions
- Documentation references
