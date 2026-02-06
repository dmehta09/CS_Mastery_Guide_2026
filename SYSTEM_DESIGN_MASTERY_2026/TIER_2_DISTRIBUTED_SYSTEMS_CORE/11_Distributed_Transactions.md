# Distributed Transactions: A Comprehensive Guide

## Introduction

In modern software systems, data and operations are often spread across multiple servers, databases, or services. A **distributed transaction** is an operation that involves multiple independent systems working together to complete a single logical unit of work—either all parts succeed together, or all fail together.

Think of it like coordinating a group purchase: if you and three friends agree to split the cost of a gift, either everyone pays their share (success), or the purchase is cancelled (failure). You can't have some people paying and others not—it's all or nothing.

**Key Challenge**: Unlike a single database where transactions are straightforward, distributed systems face network delays, partial failures, and the impossibility of perfect coordination. The concepts in this guide address these challenges.

---

## Two-Phase Commit (2PC)

### Overview

The **Two-Phase Commit protocol** is one of the oldest and most fundamental approaches to ensuring that a distributed transaction either completes successfully across all participants or fails completely with no partial updates.

**Layman explanation**: Imagine a wedding ceremony. The officiant (coordinator) first asks "Do you take this person..." to both parties (prepare phase), waits for both to say "I do," and only then declares them married (commit phase). If either says "no," the ceremony is called off entirely.

### Architecture

```
                    ┌─────────────┐
                    │ Coordinator │
                    │  (Manager)  │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
    ┌────────┐        ┌────────┐        ┌────────┐
    │Participant│      │Participant│      │Participant│
    │    A     │        │    B     │        │    C     │
    │(Database)│        │(Database)│        │(Database)│
    └────────┘        ┌────────┘        └────────┘
```

**Roles**:
- **Coordinator**: The single entity that orchestrates the entire transaction
- **Participants**: The distributed systems (databases, services) that execute parts of the transaction

### Phase 1: Prepare Phase

In the prepare phase, the coordinator asks all participants: "Can you commit this transaction?"

**Flow**:
```
Coordinator                 Participant A       Participant B       Participant C
    │                            │                   │                   │
    │──── PREPARE ──────────────>│                   │                   │
    │──── PREPARE ───────────────────────────────────>│                   │
    │──── PREPARE ────────────────────────────────────────────────────────>│
    │                            │                   │                   │
    │                      [Lock resources]     [Lock resources]    [Lock resources]
    │                      [Write to log]       [Write to log]      [Write to log]
    │                            │                   │                   │
    │<─── YES (ready) ───────────│                   │                   │
    │<─── YES (ready) ───────────────────────────────│                   │
    │<─── YES (ready) ────────────────────────────────────────────────────│
```

**What happens in each participant**:
1. Participant receives the PREPARE message
2. It attempts to perform the transaction (writes to a temporary location)
3. It locks all necessary resources to prevent other transactions from interfering
4. It writes a "prepared" record to a durable log (survives crashes)
5. It responds with either:
   - **YES** (I'm ready to commit)
   - **NO** (I cannot complete this transaction)

**Critical point**: After voting YES, a participant MUST be able to commit later, even if it crashes. The "prepared" log entry ensures this promise.

### Phase 2: Commit Phase

Once the coordinator receives responses from all participants, it makes a decision:

**If ALL participants voted YES**:
```
Coordinator                 Participant A       Participant B       Participant C
    │                            │                   │                   │
    │[Decision: COMMIT]          │                   │                   │
    │[Write decision to log]     │                   │                   │
    │                            │                   │                   │
    │──── COMMIT ────────────────>│                   │                   │
    │──── COMMIT ─────────────────────────────────────>│                   │
    │──── COMMIT ──────────────────────────────────────────────────────────>│
    │                            │                   │                   │
    │                      [Make permanent]     [Make permanent]   [Make permanent]
    │                      [Release locks]      [Release locks]    [Release locks]
    │                            │                   │                   │
    │<─── ACK ────────────────────│                   │                   │
    │<─── ACK ─────────────────────────────────────────│                   │
    │<─── ACK ──────────────────────────────────────────────────────────────│
```

**If ANY participant voted NO** (or didn't respond):
```
Coordinator                 Participant A       Participant B       Participant C
    │                            │                   │                   │
    │[Decision: ABORT]           │                   │                   │
    │[Write decision to log]     │                   │                   │
    │                            │                   │                   │
    │──── ABORT ─────────────────>│                   │                   │
    │──── ABORT ──────────────────────────────────────>│                   │
    │──── ABORT ───────────────────────────────────────────────────────────>│
    │                            │                   │                   │
    │                      [Undo changes]       [Undo changes]     [Undo changes]
    │                      [Release locks]      [Release locks]    [Release locks]
    │                            │                   │                   │
    │<─── ACK ────────────────────│                   │                   │
    │<─── ACK ─────────────────────────────────────────│                   │
    │<─── ACK ──────────────────────────────────────────────────────────────│
```

### Coordinator Failure Handling

**Scenario**: What if the coordinator crashes?

**Before logging the decision**:
```
Timeline of events:

1. Coordinator sends PREPARE to all participants
2. All participants respond YES
3. 💥 Coordinator crashes BEFORE writing COMMIT/ABORT to its log
4. Participants are stuck in "prepared" state, holding locks
```

**Problem**: Participants don't know whether to commit or abort. They're **blocked** (this is 2PC's biggest weakness).

**Recovery approach**:
- Participants remain blocked until the coordinator recovers
- When coordinator restarts, it reads its log
- If no decision was logged, coordinator sends ABORT to all participants (safety-first approach)
- If decision was logged, coordinator resends the decision

**After logging the decision**:
- Coordinator recovers and reads its log
- Finds the COMMIT or ABORT decision
- Resends the decision to all participants (some may have already received it)
- Participants acknowledge (idempotently—receiving COMMIT twice is safe)

### Participant Failure Handling

**Scenario 1: Participant crashes BEFORE responding to PREPARE**

```
Timeline:

1. Coordinator sends PREPARE to Participants A, B, C
2. Participant B crashes 💥
3. Coordinator waits for timeout period
4. Coordinator receives no response from B
5. Coordinator decides to ABORT (conservative approach)
6. Transaction rolls back everywhere
```

**Result**: Transaction is aborted. Safe, but possibly unnecessary if B could have participated.

**Scenario 2: Participant crashes AFTER voting YES (in prepared state)**

```
Timeline:

1. Participant B receives PREPARE
2. B writes "prepared" to durable log
3. B responds YES
4. 💥 B crashes before receiving COMMIT/ABORT
5. Coordinator sends COMMIT to all
6. B misses the COMMIT message
```

**Recovery**:
- When B restarts, it reads its log and sees "prepared" state
- B must contact the coordinator to ask: "What was the decision?"
- Coordinator responds with COMMIT or ABORT
- B executes the decision and updates its log to "committed" or "aborted"

**Alternative**: B can also ask other participants for the outcome (if it can't reach the coordinator).

**Scenario 3: Participant crashes AFTER committing**

```
Timeline:

1. Participant C receives COMMIT
2. C makes changes permanent
3. C writes "committed" to log
4. 💥 C crashes before acknowledging
5. Coordinator doesn't receive ACK
```

**Recovery**:
- When C restarts, it reads its log and sees "committed"
- The work is already done; C simply sends ACK to coordinator
- If coordinator resends COMMIT, C responds with ACK (idempotent)

### Blocking Nature and Limitations

**The Fundamental Blocking Problem**

2PC is called a **blocking protocol** because participants can get stuck waiting indefinitely:

```
Scenario: Coordinator crashes after participants vote YES

Participant A          Participant B          Participant C
     │                      │                      │
     │ [PREPARED]           │ [PREPARED]           │ [PREPARED]
     │ Locks held           │ Locks held           │ Locks held
     │ Waiting...           │ Waiting...           │ Waiting...
     │ Waiting...           │ Waiting...           │ Waiting...
     │ Waiting...           │ Waiting...           │ Waiting...
     │                      │                      │
     └──── Cannot safely commit or abort ─────────┘
```

**Why they can't decide themselves**:
- If they independently COMMIT, but coordinator had decided ABORT → **inconsistency**
- If they independently ABORT, but coordinator had decided COMMIT → **inconsistency**
- They must wait for the coordinator's decision to maintain correctness

**Real-world impact**: Resources remain locked, blocking other transactions from proceeding. This can cascade into system-wide slowdowns.

**Other Key Limitations**

1. **Single Point of Failure**
   - Coordinator is critical; its failure blocks all transactions
   - Even with coordinator backups, brief unavailability causes blocking

2. **Performance Overhead**
   - Requires two rounds of network communication
   - Synchronous blocking at each phase reduces throughput
   - Locks held for extended periods during both phases

3. **Network Partition Vulnerability**
   - If network splits, participants in different partitions cannot communicate
   - Partitioned participants remain blocked even if healthy

4. **Timeout Challenges**
   - Too short: false failures, unnecessary aborts
   - Too long: prolonged blocking, poor user experience

5. **Scalability Issues**
   - As participants increase, probability of failure increases
   - Coordinator becomes bottleneck for all distributed transactions

**When to use 2PC despite limitations**:
- Within a single data center with reliable networking
- When strong consistency is absolutely required
- For infrequent, critical transactions
- In controlled environments with automated coordinator failover

---

## Three-Phase Commit (3PC)

### Overview

**Three-Phase Commit** was designed to address the blocking problem of 2PC by adding an additional phase that provides better failure recovery. The key insight: create a state where participants know the coordinator has made a decision, but haven't yet committed.

**Layman explanation**: Think of a rocket launch countdown. Instead of going from "ready" directly to "launch," there's an intermediate "all systems confirmed, launching in 3... 2... 1..." phase. If something fails during this countdown, everyone knows the launch was approved and can proceed independently if the mission control (coordinator) fails.

### The Three Phases

**Phase 1: Can-Commit (Prepare)**
```
Coordinator asks: "Can you commit?"
Participant responds: YES or NO
(Same as 2PC Phase 1)
```

**Phase 2: Pre-Commit (NEW - the key addition)**
```
Coordinator tells: "Prepare to commit"
Participant acknowledges: "Ready to commit"
```

**Phase 3: Do-Commit (Execute)**
```
Coordinator tells: "Now commit"
Participant executes: "Committed"
```

### Complete Flow Diagram

```
Coordinator              Participant A      Participant B      Participant C
    │                         │                  │                  │
    │──PHASE 1: CAN-COMMIT────│──────────────────│──────────────────│
    │                         │                  │                  │
    │                    [Check if can commit]  [Check]         [Check]
    │                         │                  │                  │
    │<─── YES ────────────────│                  │                  │
    │<─── YES ────────────────│──────────────────│                  │
    │<─── YES ────────────────│──────────────────│──────────────────│
    │                         │                  │                  │
    │[All YES received]       │                  │                  │
    │[Decision: PREPARE]      │                  │                  │
    │                         │                  │                  │
    │──PHASE 2: PRE-COMMIT────│──────────────────│──────────────────│
    │                         │                  │                  │
    │                    [Lock resources]    [Lock resources] [Lock resources]
    │                    [Write prepared log][Write log]      [Write log]
    │                    [Enter PRE-COMMIT] [PRE-COMMIT]     [PRE-COMMIT]
    │                         │                  │                  │
    │<─── ACK ────────────────│                  │                  │
    │<─── ACK ────────────────│──────────────────│                  │
    │<─── ACK ────────────────│──────────────────│──────────────────│
    │                         │                  │                  │
    │[All ACKs received]      │                  │                  │
    │[Decision: COMMIT]       │                  │                  │
    │                         │                  │                  │
    │──PHASE 3: DO-COMMIT─────│──────────────────│──────────────────│
    │                         │                  │                  │
    │                    [Make permanent]   [Make permanent] [Make permanent]
    │                    [Release locks]    [Release locks]  [Release locks]
    │                         │                  │                  │
    │<─── COMMITTED ──────────│                  │                  │
    │<─── COMMITTED ──────────│──────────────────│                  │
    │<─── COMMITTED ──────────│──────────────────│──────────────────│
    │                         │                  │                  │
    │[Transaction complete]   │                  │                  │
```

### Pre-Commit Phase Addition: The Game Changer

**What makes Pre-Commit special**:

The Pre-Commit phase creates a **point of no return** that all participants know about:

```
Traditional 2PC states:
  [Working] → [Prepared] → ??? coordinator fails → [BLOCKED]

3PC states:
  [Working] → [Prepared] → [Pre-Commit] → [Committed]
                               ↑
                          Point of no return!
                    Everyone knows commit will happen
```

**Key properties**:

1. **Before Pre-Commit**: If coordinator fails, participants can safely ABORT
2. **After Pre-Commit**: If coordinator fails, participants can safely COMMIT
3. **The boundary is known**: Participants can distinguish between these states

**How this prevents blocking**:

```
Scenario: Coordinator fails after sending Pre-Commit

Participant A          Participant B          Participant C
     │                      │                      │
     │ [PRE-COMMIT]         │ [PRE-COMMIT]         │ [PREPARED]
     │ Received Pre-Commit  │ Received Pre-Commit  │ Didn't receive it
     │                      │                      │
     │────── Contact each other ───────────────────│
     │                      │                      │
     │ "I'm in Pre-Commit"  │                      │
     │─────────────────────>│                      │
     │                      │ "Me too!"            │
     │<─────────────────────│                      │
     │                      │                      │
     │───────────────── "Majority in Pre-Commit" ──│
     │                                              │
     │──── All independently decide: COMMIT ───────│
     │                      │                      │
```

**Decision rules when coordinator fails**:

```
IF any participant is in PRE-COMMIT state:
    → All participants COMMIT
    → (Coordinator must have decided to commit)

ELSE IF all reachable participants are in PREPARED state:
    → All participants ABORT
    → (Coordinator failed before deciding)

ELSE:
    → Wait for timeout, then apply above rules
```

### Non-Blocking Properties

**How 3PC avoids 2PC's blocking problem**:

1. **Definitive decision points**
   - Pre-Commit received = Coordinator decided to commit
   - Pre-Commit not received = Safe to abort

2. **Participants can make progress independently**
   - They can query each other's states
   - They can apply decision rules without coordinator
   - No indefinite waiting required

3. **Timeout-based recovery**
   ```
   Participant timeout logic:

   IF waiting for Pre-Commit AND timeout expires:
       Query other participants
       IF any in Pre-Commit → COMMIT
       ELSE → ABORT

   IF waiting for Do-Commit AND timeout expires:
       Query other participants
       IF majority in Pre-Commit → COMMIT
       ELSE → ABORT
   ```

**The freedom from blocking**:

```
2PC Blocking Scenario:
    Coordinator crashes → Participants wait indefinitely → System stuck

3PC Non-Blocking Scenario:
    Coordinator crashes → Participants timeout → Query peers → Decide independently → System progresses
```

### Limitations in Practice

Despite its theoretical non-blocking property, 3PC has **critical limitations** that prevent widespread adoption:

**1. Network Partition Vulnerability (The Fatal Flaw)**

```
Scenario: Network partition splits participants

Coordinator + Part. A          │ Network Partition │        Part. B + Part. C
                               │                    │
Pre-Commit sent to A           │                    │    No Pre-Commit received
                               │                    │
A is in PRE-COMMIT             │                    │    B and C are in PREPARED
                               │                    │
A decides: COMMIT              │                    │    B and C decide: ABORT
                               │                    │
    💥 INCONSISTENCY! 💥       │                    │
```

**The problem**: 3PC assumes participants can reliably communicate to query each other's states. Network partitions violate this assumption.

**Result**: Different partitions can make opposite decisions, leading to **split-brain** scenarios and data inconsistency.

**2. Perfect Failure Detection Required**

3PC assumes you can distinguish between:
- A slow coordinator (still working, just delayed)
- A failed coordinator (actually crashed)

```
Reality:
    "Is it slow or dead?" is impossible to answer perfectly in distributed systems

    Timeout too short → False positives → Unnecessary independent decisions
    Timeout too long → Back to blocking behavior
```

This is known as the **impossibility of perfect failure detection** in asynchronous networks.

**3. Increased Latency**

```
2PC: 2 network round-trips
3PC: 3 network round-trips

For every transaction:
    Additional 50-200ms latency (depending on network)
    Reduced throughput
    More resource locking time
```

**4. Higher Coordinator Complexity**

- More state transitions to manage
- More failure scenarios to handle
- More complex recovery logic
- Harder to implement correctly

**5. The Fundamental Truth**

The **CAP theorem** (Consistency, Availability, Partition tolerance) proves you cannot have all three. 3PC tries to maintain consistency and availability, but:

```
During network partition:
    3PC must choose: Consistency OR Availability

    Reality: It often chooses availability → Risks inconsistency
    OR: It blocks → Back to 2PC's problem
```

**Why 3PC is rarely used in practice**:

- **Google Spanner**, **Apache CockroachDB**, **AWS Aurora**: Use 2PC with sophisticated failure handling
- **Modern microservices**: Use Saga pattern instead
- **Event-driven systems**: Use eventual consistency
- **Financial systems**: Prefer 2PC with manual intervention procedures

**The paradox**: 3PC solved 2PC's blocking problem theoretically, but introduced worse practical problems. In real-world distributed systems with unreliable networks, the cure is worse than the disease.

**When 3PC might be considered** (rare):
- Controlled network environments with no partition risk
- Academic or research contexts
- Systems with very reliable failure detection mechanisms

---

## Saga Pattern

### Overview

The **Saga pattern** takes a completely different approach to distributed transactions: instead of trying to coordinate atomic commits across services, it breaks a long-running transaction into a series of smaller, independent local transactions. Each local transaction updates data within a single service and publishes an event or message to trigger the next step.

**Layman explanation**: Think of booking a vacation package (flight + hotel + car rental). Instead of requiring all three to confirm simultaneously (which might fail if one is slow), you book them one by one. If the car rental fails after you've booked the flight and hotel, you have predefined "cancellation steps" that automatically undo the flight and hotel bookings.

**Key philosophy**: Embrace eventual consistency rather than fight for immediate consistency.

### Core Concept: Compensating Transactions

Unlike 2PC which prevents changes from being made permanent until all agree, Sagas:

1. **Make changes immediately** (local commits)
2. **Define compensating actions** to undo changes if something later fails
3. **Execute compensations in reverse order** when failure occurs

```
Happy path (all succeed):
    Order Service → Payment Service → Inventory Service → Shipping Service
    [Order Created]  [Payment Taken]   [Items Reserved]    [Shipment Started]
         ✓                ✓                 ✓                    ✓

Failure path (Shipping fails):
    Order Service → Payment Service → Inventory Service → Shipping Service
    [Order Created]  [Payment Taken]   [Items Reserved]    [FAILED ✗]
         ✓                ✓                 ✓
                         ↓                  ↓
              [Refund Payment]    [Release Inventory]
                     ↓
                [Cancel Order]
```

### Choreography-Based Sagas

In **choreography**, there's no central coordinator. Each service knows what to do when it receives an event, and what event to publish when it's done.

**Architecture**:
```
                        Event Bus / Message Broker
                    ┌──────────────────────────────┐
                    │   (Kafka, RabbitMQ, etc.)    │
                    └──────────────┬───────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
  ┌──────────┐              ┌──────────┐              ┌──────────┐
  │  Order   │              │ Payment  │              │Inventory │
  │ Service  │              │ Service  │              │ Service  │
  └──────────┘              └──────────┘              └──────────┘
```

**Flow Example: E-commerce Order**

```
Step 1: User initiates order
  Order Service:
    • Create order (status: PENDING)
    • Publish: OrderCreated event

        │
        └─────> Event: { type: "OrderCreated", orderId: 123, items: [...], total: 99.99 }
                    │
                    ▼

Step 2: Payment Service listens for OrderCreated
  Payment Service:
    • Charge customer
    • IF SUCCESS:
        Publish: PaymentCompleted event
      IF FAILURE:
        Publish: PaymentFailed event

        │ (Success)
        └─────> Event: { type: "PaymentCompleted", orderId: 123, transactionId: "tx789" }
                    │
                    ▼

Step 3: Inventory Service listens for PaymentCompleted
  Inventory Service:
    • Reserve inventory
    • IF SUCCESS:
        Publish: InventoryReserved event
      IF FAILURE:
        Publish: InventoryReservationFailed event

        │ (Success)
        └─────> Event: { type: "InventoryReserved", orderId: 123, warehouseId: "W1" }
                    │
                    ▼

Step 4: Shipping Service listens for InventoryReserved
  Shipping Service:
    • Create shipment
    • Publish: ShipmentCreated event
```

**Compensation Flow (when something fails)**:

```
Failure at Inventory step:

InventoryReservationFailed event
    │
    ├─────> Payment Service listens
    │         • Refund payment
    │         • Publish: PaymentRefunded event
    │
    └─────> Order Service listens
              • Update order status: FAILED
              • Publish: OrderCancelled event
```

**Advantages of Choreography**:
- **Loose coupling**: Services don't know about each other, only events
- **Scalability**: No central bottleneck
- **Resilience**: Failure of one service doesn't block others
- **Flexibility**: Easy to add new services that react to events

**Disadvantages of Choreography**:
- **Hard to understand**: Flow logic is scattered across services
- **Difficult to debug**: No single place showing the entire saga flow
- **Cyclic dependencies**: Services can accidentally create event loops
- **No single source of truth**: Saga state is distributed

### Orchestration-Based Sagas

In **orchestration**, a central **Saga Orchestrator** (or Saga Execution Coordinator) explicitly controls the transaction flow. It tells each service what to do and coordinates compensations.

**Architecture**:
```
                    ┌────────────────────┐
                    │ Saga Orchestrator  │
                    │   (Coordinator)    │
                    └─────────┬──────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
  ┌──────────┐          ┌──────────┐          ┌──────────┐
  │  Order   │          │ Payment  │          │Inventory │
  │ Service  │          │ Service  │          │ Service  │
  └──────────┘          └──────────┘          └──────────┘
```

**Flow Example: Same E-commerce Order**

```
Orchestrator's State Machine:

START
  │
  ▼
[1] CALL Order.createOrder(items, total)
  │
  ├─ SUCCESS → Store orderId
  │              │
  │              ▼
  │            [2] CALL Payment.charge(orderId, amount)
  │              │
  │              ├─ SUCCESS → Store transactionId
  │              │              │
  │              │              ▼
  │              │            [3] CALL Inventory.reserve(orderId, items)
  │              │              │
  │              │              ├─ SUCCESS → Store reservationId
  │              │              │              │
  │              │              │              ▼
  │              │              │            [4] CALL Shipping.ship(orderId)
  │              │              │              │
  │              │              │              ├─ SUCCESS → END (COMMITTED)
  │              │              │              │
  │              │              │              └─ FAILURE → COMPENSATE
  │              │              │                            │
  │              │              └─ FAILURE → COMPENSATE      │
  │              │                            │              │
  │              └─ FAILURE → COMPENSATE      │              │
  │                            │              │              │
  └─ FAILURE → END (ABORTED)  │              │              │
                               │              │              │
                               ▼              ▼              ▼
                        Compensation Flow (reverse order):

                        [4] SKIP (Shipping never happened)
                        [3] CALL Inventory.releaseReservation(reservationId)
                        [2] CALL Payment.refund(transactionId)
                        [1] CALL Order.cancelOrder(orderId)

                        END (COMPENSATED)
```

**Orchestrator's Persistent State**:

```
Saga Instance ID: saga-123
Status: IN_PROGRESS
Current Step: 3 (Inventory.reserve)

Execution Log:
  Step 1: Order.createOrder → SUCCESS, orderId: "ord-456"
  Step 2: Payment.charge → SUCCESS, transactionId: "tx-789"
  Step 3: Inventory.reserve → IN_PROGRESS

Compensation Plan (if needed):
  Step 3: Inventory.releaseReservation(reservationId)
  Step 2: Payment.refund("tx-789")
  Step 1: Order.cancelOrder("ord-456")
```

**Advantages of Orchestration**:
- **Clear flow**: Saga logic is centralized and easy to understand
- **Easy debugging**: Single place to see saga progress and state
- **Explicit control**: Orchestrator explicitly manages compensations
- **Easier testing**: Can test entire saga flow in isolation
- **Better observability**: Can query orchestrator for saga status

**Disadvantages of Orchestration**:
- **Single point of coordination**: Orchestrator becomes a dependency
- **Tighter coupling**: Services are aware of orchestrator
- **Additional complexity**: Need to build/maintain orchestrator service
- **Potential bottleneck**: All sagas flow through orchestrator

### Compensating Transactions

**Compensating transactions** are the heart of the Saga pattern. They're operations that semantically undo previous operations.

**Important distinction**:
```
NOT a database rollback (which erases changes as if they never happened)
BUT a business-level reversal (which creates new records showing the reversal)
```

**Examples**:

| Forward Transaction | Compensating Transaction | Notes |
|---------------------|-------------------------|-------|
| Create order | Cancel order | Order record remains, marked CANCELLED |
| Charge credit card | Refund credit card | Both charge and refund are recorded |
| Reserve inventory | Release inventory | Inventory count updated, reservation removed |
| Send email | Send apology email | Can't unsend, but can notify |
| Update account balance +$100 | Update account balance -$100 | Ledger shows both transactions |

**Designing compensating transactions**:

```
Forward Action: Reserve inventory
  Database changes:
    • inventory.available -= quantity
    • Create reservation record

Compensating Action: Release reservation
  Database changes:
    • inventory.available += quantity
    • Delete reservation record
    • Log compensation action
```

**Critical properties**:

1. **Idempotent**: Running compensation multiple times has the same effect as running it once
2. **Retriable**: Compensations should always eventually succeed
3. **Semantically correct**: Must truly reverse the business impact

**Challenges with compensations**:

```
Problem 1: Some actions are hard to compensate
  • Email already sent → Can send apology, but can't unsend
  • Bitcoin transferred → Irreversible on blockchain
  • Third-party API called → May not support reversal

  Solution: Design with compensability in mind, use eventual consistency

Problem 2: Compensations can fail
  • Network timeout during refund
  • Service unavailable during inventory release

  Solution: Retry mechanism with exponential backoff, dead letter queue

Problem 3: Partial visibility
  • User saw "order confirmed" before it was compensated
  • External systems were notified of initial state

  Solution: Clear user communication, status tracking, eventual consistency
```

### Saga Execution Coordinator

The **Saga Execution Coordinator** (SEC) is the component that manages orchestration-based sagas. It's responsible for:

**Core Responsibilities**:

```
┌─────────────────────────────────────┐
│   Saga Execution Coordinator        │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  Saga Instance Manager        │ │ ← Tracks active sagas
│  └───────────────────────────────┘ │
│  ┌───────────────────────────────┐ │
│  │  State Machine Engine         │ │ ← Executes saga steps
│  └───────────────────────────────┘ │
│  ┌───────────────────────────────┐ │
│  │  Command Dispatcher           │ │ ← Calls services
│  └───────────────────────────────┘ │
│  ┌───────────────────────────────┐ │
│  │  Compensation Executor        │ │ ← Runs compensations
│  └───────────────────────────────┘ │
│  ┌───────────────────────────────┐ │
│  │  Persistence Layer            │ │ ← Stores saga state
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
```

**Saga Definition Example** (what SEC executes):

```
Saga Name: OrderFulfillmentSaga

Steps:
  1. CreateOrder
     Service: Order Service
     Command: createOrder(customerId, items)
     Compensation: cancelOrder(orderId)

  2. ProcessPayment
     Service: Payment Service
     Command: chargeCard(orderId, amount)
     Compensation: refundPayment(transactionId)

  3. ReserveInventory
     Service: Inventory Service
     Command: reserveItems(orderId, items)
     Compensation: releaseReservation(reservationId)

  4. CreateShipment
     Service: Shipping Service
     Command: createShipment(orderId, address)
     Compensation: cancelShipment(shipmentId)

Compensation Order: Reverse (4 → 3 → 2 → 1)
Retry Policy: Exponential backoff, max 5 attempts
Timeout: 30 seconds per step
```

**Execution Flow**:

```
1. SEC receives saga start request
2. SEC creates saga instance with unique ID
3. SEC persists initial state to database
4. SEC executes Step 1:
   a. Dispatch command to Order Service
   b. Wait for response
   c. IF SUCCESS:
        • Update saga state
        • Persist state
        • Move to Step 2
      IF FAILURE:
        • No compensation needed (first step)
        • Mark saga as FAILED
        • END
5. Repeat for Steps 2, 3, 4...
6. IF any step fails:
   a. Mark saga as COMPENSATING
   b. Execute compensations in reverse order
   c. Mark saga as COMPENSATED
```

**Persistence is critical**:

```
Why persist state after every step?

If SEC crashes:
  • On restart, read saga state from database
  • Resume from last completed step
  • OR start compensation from last successful step

Without persistence:
  • Saga progress lost
  • Partial changes remain without compensation
  • System inconsistency
```

**SEC Implementation Patterns**:

1. **Embedded SEC**: Part of a single service
   - Simple for single-application sagas
   - Couples saga logic to one service

2. **Dedicated SEC Service**: Separate microservice
   - Reusable across many saga types
   - Better for polyglot environments

3. **Framework-based**: Use existing saga frameworks
   - **Temporal**: Workflow engine with saga support
   - **Camunda**: Process automation platform
   - **Axon Framework**: Event sourcing + sagas
   - **Eventuate Tram Saga**: Saga framework for Java

### Isolation Challenges

Unlike traditional database transactions with ACID properties, sagas provide **eventual consistency** but sacrifice **isolation**. This creates several challenges.

**The Core Problem**:

```
Traditional Transaction (ACID):
  Transaction A: Read inventory → Modify → Write inventory
  [All changes invisible to others until COMMIT]
  [Isolated from Transaction B]

Saga Transaction:
  Step 1: Order Service creates order ← IMMEDIATELY VISIBLE
  Step 2: Payment Service charges card ← IMMEDIATELY VISIBLE
  Step 3: Inventory Service reserves items ← IMMEDIATELY VISIBLE
  [Each step commits locally, changes visible immediately]
  [NOT isolated from other operations]
```

**Isolation Problem Examples**:

**1. Lost Updates**

```
Timeline:

Saga A (Ordering Product X):          Saga B (Ordering Product X):
  │                                      │
  ├─ Read inventory: 5 units             │
  │                                      ├─ Read inventory: 5 units
  ├─ Reserve 3 units                     │
  │  (inventory now: 2 units)            │
  │                                      ├─ Reserve 3 units
  │                                         (tries to reserve from 5, not 2!)
  │
  Result: Over-committed inventory (tried to reserve 6 from 5 available)
```

**2. Dirty Reads**

```
Timeline:

Saga A (Create order, then fails):     External Query:
  │                                      │
  ├─ Create order (id: 123)              │
  │  Status: PENDING                     │
  │                                      ├─ Read order 123
  │                                      │  Shows: PENDING ← Dirty read!
  │                                      │  (might be compensated soon)
  ├─ Payment FAILS                       │
  ├─ COMPENSATE:                         │
  │  Cancel order 123                    │
  │  Status: CANCELLED                   │
                                         ├─ Read order 123 again
                                            Shows: CANCELLED
```

The external query saw an order that eventually got cancelled—it read "dirty" data.

**3. Non-Repeatable Reads**

```
Saga A:                                Report Generation:
  │                                      │
  ├─ Create order $100                   │
  │                                      ├─ Read total sales: $100
  ├─ Process payment                     │
  │                                      │
  ├─ Reserve inventory                   │
  │                                      │
  │  Payment fails!                      │
  ├─ COMPENSATE: cancel order            │
  │                                      ├─ Read total sales: $0
                                            (Same query, different result!)
```

**Isolation Strategies**:

**Strategy 1: Semantic Locks**

Instead of database-level locks, use business-level locking:

```
Example: Inventory Reservation

Instead of:
  inventory.quantity -= 5  (immediate update)

Use:
  CREATE reservation WHERE item_id = X AND quantity = 5 AND status = 'PENDING'
  [Inventory physically still available]
  [But logically reserved via separate record]

  Other sagas check: available_quantity - SUM(pending_reservations)
```

**Strategy 2: Commutative Updates**

Design operations that can be reordered:

```
Bad (non-commutative):
  account.balance = account.balance + 100
  [Read-modify-write: order matters]

Good (commutative):
  CREATE transaction { account_id: X, amount: +100, timestamp: T }
  [Append-only: order doesn't matter]
  [Balance calculated by SUM(transactions)]
```

**Strategy 3: Pessimistic View**

Re-read data before each step:

```
Step 1: Reserve inventory
  • Read current available quantity
  • Create reservation

Step 2: Process payment
  • Re-read inventory to confirm still available
  • If not available, fail and compensate Step 1

Step 3: Ship
  • Re-read reservation to confirm still valid
  • Proceed only if valid
```

**Strategy 4: Version Files / Optimistic Locking**

```
Each update increments a version:

Step 1: Read inventory (version: 5)
Step 2: Update inventory WHERE version = 5
  IF affected_rows = 0:
    [Someone else updated it, version is now 6]
    FAIL and compensate
  ELSE:
    [Update succeeded, increment to version 6]
    Continue
```

**Strategy 5: Reordering / Saga Step Design**

Place risky or hard-to-compensate steps early:

```
Bad Order:
  1. Charge credit card (hard to compensate: refunds are visible)
  2. Check inventory (might fail)
  3. Reserve inventory

Good Order:
  1. Check inventory (fails fast if unavailable)
  2. Reserve inventory (easy to compensate)
  3. Charge credit card (only after ensuring inventory)
```

**Strategy 6: Read-Only Queries on Stable Views**

```
Problem: Reports reading saga-in-progress data

Solution: Separate read models
  • Saga writes to operational database
  • Completed transactions update read-optimized view
  • Reports query read-only view (eventual consistency)

  Operational DB:        Read Model:
    Order 123: PENDING     [Order 123 not yet visible]
    [Saga running...]
    Order 123: COMPLETE    [Order 123 appears now]
```

**The Fundamental Trade-off**:

```
Sagas sacrifice isolation FOR:
  ✓ Scalability (no distributed locks)
  ✓ Availability (services fail independently)
  ✓ Long-running operations (no blocking)

You must accept:
  ✗ Dirty reads possible
  ✗ Lost updates possible
  ✗ Non-repeatable reads
  ✗ Application-level handling required
```

**When to use Sagas**:
- Microservices architectures where services are independent
- Long-running business processes (hours, days)
- Operations spanning multiple bounded contexts
- When availability is more important than immediate consistency
- E-commerce, booking systems, order processing

**When NOT to use Sagas**:
- Financial accounting requiring strong consistency
- Operations requiring strict isolation (use 2PC instead)
- Short-lived transactions within a single database
- When compensations are impossible or extremely complex

---

## Distributed Locks

### Overview

A **distributed lock** is a synchronization mechanism that allows multiple processes or services running on different machines to coordinate access to shared resources. Unlike locks in a single process (which use memory), distributed locks require coordination across a network.

**Layman explanation**: Imagine a single bathroom at a large office. The lock on the door ensures only one person uses it at a time. A distributed lock is like a bathroom shared across multiple buildings—you need a coordination system to ensure someone in Building A knows if someone in Building B is using it.

**Why distributed locks are needed**:

```
Scenario: Two services trying to process the same job

Service Instance A (Server 1)         Service Instance B (Server 2)
         │                                     │
         ├─ Read job queue                     ├─ Read job queue
         │  Job 123: "pending"                 │  Job 123: "pending"
         │                                     │
         ├─ Process Job 123                    ├─ Process Job 123
         │  (send email)                       │  (send email)
         │                                     │
         └─ Mark complete                      └─ Mark complete

         Problem: Customer receives two emails! (duplicate work)

         Solution: Acquire distributed lock before processing
```

### Lock Acquisition and Release

**Basic Lock Flow**:

```
Service A                   Lock Service               Service B
    │                       (Redis/ZK/etc)                 │
    │                             │                        │
    ├──TRY_ACQUIRE("job:123")────>│                        │
    │                             │                        │
    │                        [Check if locked]             │
    │                        [Not locked]                  │
    │                        [Set lock: job:123 → A]       │
    │                             │                        │
    │<──SUCCESS────────────────────│                        │
    │                             │                        │
    │                             │<──TRY_ACQUIRE("job:123")│
    │                             │                        │
    │                        [Check if locked]             │
    │                        [Already locked by A]         │
    │                             │                        │
    │                             │──FAILURE──────────────>│
    │                             │                        │
    ├──[Process job 123]          │                        │
    │                             │                        │
    ├──RELEASE("job:123")────────>│                        │
    │                             │                        │
    │                        [Remove lock]                 │
    │                             │                        │
    │<──SUCCESS────────────────────│                        │
    │                             │                        │
    │                             │<──TRY_ACQUIRE("job:123")│
    │                             │                        │
    │                             │  [Not locked]          │
    │                             │  [Set lock: job:123 → B]│
    │                             │                        │
    │                             │──SUCCESS──────────────>│
    │                             │                        │
    │                             │  ├──[Process job 123]  │
```

**Lock Structure** (typically stored in lock service):

```
Lock Key: "job:123"
Owner: "service-A-instance-42"
Acquired At: 2026-02-06T10:30:00Z
Expires At: 2026-02-06T10:30:30Z (30 seconds from acquisition)
Lock Token: "random-uuid-abc123def456"
```

**Acquisition Pseudocode**:

```
Function AcquireLock(lockKey, ownerId, ttl):
    lockToken = GenerateUniqueToken()

    success = LockService.SetIfNotExists(
        key: lockKey,
        value: {
            owner: ownerId,
            token: lockToken,
            acquired_at: CurrentTime()
        },
        ttl: ttl  // Time-to-live (expiration)
    )

    IF success:
        RETURN lockToken
    ELSE:
        RETURN null  // Lock already held by someone else
```

**Release Pseudocode**:

```
Function ReleaseLock(lockKey, lockToken):
    currentLock = LockService.Get(lockKey)

    IF currentLock.token == lockToken:
        // Only release if we still own it
        LockService.Delete(lockKey)
        RETURN true
    ELSE:
        // Someone else owns it now (or expired and was reacquired)
        RETURN false
```

**Critical requirement**: Release must verify ownership via token to prevent releasing someone else's lock.

### Lock Expiration and Renewal

**The Expiration Problem**:

```
Scenario: Service crashes while holding lock

Service A                   Lock Service
    │                             │
    ├──ACQUIRE("job:123")────────>│
    │<──SUCCESS────────────────────│
    │                             │
    ├──[Start processing...]      │
    │                             │
    💥 Service A crashes          │
    │                             │
    [Lock held forever]           │
    [No one can process job 123]  │ ← Deadlock!
```

**Solution: Time-To-Live (TTL) / Expiration**

Every lock has an expiration time:

```
Lock with 30-second TTL:

T=0s:  Service A acquires lock
T=15s: Service A still processing (lock still valid)
T=30s: Lock expires automatically
T=31s: Service B can now acquire the lock
```

**But this creates a new problem: What if processing takes longer than TTL?**

```
Service A's perspective:

T=0s:  Acquire lock (expires at T=30s)
T=5s:  Processing...
T=10s: Processing...
T=15s: Processing...
T=20s: Processing...
T=25s: Processing...
T=30s: [Lock expires!] ← Service A doesn't know!
T=31s: Service B acquires lock
T=32s: Processing... ← Both A and B processing! Conflict!
T=35s: Service A completes, tries to release
       [Lock is owned by B now, release fails]
```

**Solution: Lock Renewal (Heartbeat)**

```
Service A                   Lock Service
    │                             │
    ├──ACQUIRE("job:123", TTL=30s)│
    │<──SUCCESS (token: abc123)───│
    │                             │
    ├──[Start processing]         │
    │                             │
T=10s│                            │
    ├──RENEW("job:123", token:abc123, TTL=30s)
    │<──SUCCESS (expires at T=40s)│
    │                             │
    ├──[Continue processing]      │
    │                             │
T=20s│                            │
    ├──RENEW("job:123", token:abc123, TTL=30s)
    │<──SUCCESS (expires at T=50s)│
    │                             │
    ├──[Continue processing]      │
    ├──[Complete!]                │
    ├──RELEASE("job:123", token:abc123)
    │<──SUCCESS───────────────────│
```

**Renewal Pseudocode**:

```
Function ProcessWithLock(lockKey, work):
    lockToken = AcquireLock(lockKey, myId, ttl=30)

    IF lockToken == null:
        RETURN "Could not acquire lock"

    // Start background renewal thread
    renewalThread = StartRenewalThread(lockKey, lockToken, interval=10s)

    TRY:
        result = ExecuteWork(work)
    FINALLY:
        StopRenewalThread(renewalThread)
        ReleaseLock(lockKey, lockToken)

    RETURN result

Function RenewalThread(lockKey, lockToken, interval):
    WHILE true:
        Sleep(interval)
        success = RenewLock(lockKey, lockToken, ttl=30)

        IF NOT success:
            // Lost the lock somehow (network partition, etc.)
            LogError("Lost lock during processing!")
            TerminateWork()
            BREAK
```

**Choosing TTL and Renewal Interval**:

```
Guidelines:

TTL: Long enough to handle temporary network issues
     Short enough to recover quickly from crashes
     Typical: 10-60 seconds

Renewal Interval: Much shorter than TTL
                  Allows multiple retry attempts before expiration
                  Typical: TTL / 3

Example:
  TTL = 30 seconds
  Renewal Interval = 10 seconds

  This allows:
    • 2 failed renewal attempts
    • 1 successful renewal
    Before lock expires
```

**Edge Case: Renewal Fails**

```
T=0s:  Acquire lock (expires T=30s)
T=10s: Renewal attempt... network delay...
T=20s: Renewal attempt... network delay...
T=30s: Lock expires (renewals didn't reach lock service in time)
T=31s: Service B acquires lock
T=32s: Service A's renewal finally arrives
       [Lock owned by B now, renewal fails]
       [Service A must stop processing immediately]
```

**Defensive programming**: Service should monitor renewal success and stop work if renewal fails.

### Fencing Tokens

**The Problem: Delayed Messages**

Even with proper lock handling, network delays can cause correctness issues:

```
Service A                   Lock Service            Shared Resource
    │                             │                       │
    ├──ACQUIRE("lock")───────────>│                       │
    │<──SUCCESS (expires T=30s)───│                       │
    │                             │                       │
    ├──[Processing...]            │                       │
T=15s│                            │                       │
    │  Network delay...           │                       │
T=30s│  Lock expires!             │                       │
    │                             │                       │
    │                             │<──ACQUIRE("lock")─────Service B
    │                             │                       │
    │                             │──SUCCESS (T=60s)─────>│
    │                             │                       │
    │                             │  ├──UPDATE resource──>│
    │  [Network recovers]         │                       │
    ├──UPDATE resource────────────────────────────────────>│
    │  (Stale update from A)      │                  [Conflict!]
```

Service A's update arrived late, after it had already lost the lock. The resource accepted it because it had no way to know A's lock had expired.

**Solution: Fencing Tokens**

A **fencing token** is a monotonically increasing number that the lock service assigns to each lock acquisition:

```
Lock Acquisition with Fencing:

Acquisition 1: Service A gets lock, token = 33
Acquisition 2: Service B gets lock, token = 34
Acquisition 3: Service C gets lock, token = 35
[Each new lock has a higher token]
```

**How Fencing Works**:

```
Service A                   Lock Service            Shared Resource
    │                             │                       │
    ├──ACQUIRE("lock")───────────>│                       │
    │<──SUCCESS (token=33)─────────│                       │
    │                             │                       │
    ├──[Processing...]            │                       │
T=30s│  Lock expires               │                       │
    │                             │                       │
    │                             │<──ACQUIRE("lock")─────Service B
    │                             │                       │
    │                             │──SUCCESS (token=34)──>│
    │                             │                       │
    │                             │  ├──UPDATE (token=34)>│
    │                             │  │  [Accept: 34 is newest]
    │                             │                       │
    │  [Delayed message arrives]  │                       │
    ├──UPDATE (token=33)──────────────────────────────────>│
    │                             │         [Reject: 33 < 34]
    │<──ERROR: "Stale token"──────────────────────────────┤
```

**Resource-Side Implementation**:

```
Shared Resource Pseudocode:

Function UpdateResource(data, fencingToken):
    currentToken = LoadCurrentFencingToken()

    IF fencingToken <= currentToken:
        // Reject stale updates
        RETURN Error("Stale fencing token")

    // Accept update and store new token
    ApplyUpdate(data)
    StoreFencingToken(fencingToken)
    RETURN Success

Current State:
  lastAcceptedToken: 34

Request from Service A (token=33):
  33 <= 34 → REJECT

Request from Service C (token=35):
  35 > 34 → ACCEPT, update lastAcceptedToken to 35
```

**Benefits**:
- Prevents delayed messages from causing corruption
- Resource can independently verify lock freshness
- Works even if lock service is partitioned

**Requirements**:
- Shared resource must support checking fencing tokens
- Lock service must maintain monotonic counter
- All clients must include token in every request

### Redlock Algorithm (and Controversies)

**Redlock** is an algorithm for distributed locks using Redis, proposed by Redis creator Salvatore Sanfilippo. It attempts to provide stronger guarantees than a single Redis instance.

**The Problem Redlock Solves**:

```
Single Redis Instance:

    Services → [Single Redis] → Lock
                     ↑
                  Single point of failure
                  If Redis crashes, all locks lost
```

**Redlock Architecture**:

```
Multiple Independent Redis Instances (N = 5 recommended):

                    Services
                       │
         ┌─────────────┼─────────────┐
         │             │             │
         ▼             ▼             ▼
    [Redis 1]     [Redis 2]     [Redis 3]     [Redis 4]     [Redis 5]

    [No replication between instances]
    [Each is completely independent]
```

**Redlock Algorithm Flow**:

```
Step 1: Get current time in milliseconds (T1)

Step 2: Try to acquire lock in all N instances sequentially:
  FOR each Redis instance:
    SET lock_key "my-unique-id" NX PX 30000
    [NX = only set if not exists]
    [PX 30000 = expires in 30 seconds]
    [Use short timeout, e.g., 5-50ms per instance]

Step 3: Calculate acquisition time:
  T2 = CurrentTime()
  elapsed = T2 - T1

Step 4: Check if lock acquired:
  instances_locked = count of successful SET operations

  IF instances_locked >= (N/2 + 1)  AND  elapsed < lock_validity_time:
    // Lock acquired!
    // Validity = 30000ms - elapsed - clock_drift_margin
    Use the lock
  ELSE:
    // Failed to acquire lock
    Release all instances where lock was acquired

Step 5: When done, release lock:
  FOR each Redis instance:
    DEL lock_key (using Lua script to verify ownership)
```

**Example Execution**:

```
N = 5 Redis instances
Lock TTL = 30 seconds = 30,000 ms
Timeout per instance = 10 ms

T1 = 0 ms: Start acquisition

T1+5ms:   Redis 1: SET success ✓
T1+8ms:   Redis 2: SET success ✓
T1+timeout: Redis 3: Timeout (network issue) ✗
T1+15ms:  Redis 4: SET success ✓
T1+18ms:  Redis 5: SET failed (already locked) ✗

T2 = 20 ms: Finish acquisition
Elapsed = 20 ms
instances_locked = 3
Quorum needed = 3 (5/2 + 1 = 3)

3 >= 3 ✓  AND  20ms < 30,000ms ✓
→ Lock acquired successfully!

Remaining validity = 30,000 - 20 - 100 (drift) = 29,880 ms
```

**Why Majority (N/2 + 1)?**

```
Scenario: Network partition

Partition A: Redis 1, 2, 3
Partition B: Redis 4, 5

Service A (in Partition A):
  Tries to lock → succeeds in Redis 1, 2, 3
  3 >= 3 (majority) → Lock acquired

Service B (in Partition B):
  Tries to lock → can only reach Redis 4, 5
  2 < 3 (not majority) → Lock NOT acquired

Result: Only Service A gets the lock (safety preserved)
```

**The Martin Kleppmann Controversy**

Computer scientist Martin Kleppmann published a famous critique of Redlock titled "How to do distributed locking" arguing:

**Critique Point 1: Timing Assumptions**

Redlock assumes bounded clock drift and network delay:

```
Redlock assumes:
  - Clocks don't jump forward significantly
  - Network delays are bounded
  - Processing doesn't pause (no long GC pauses)

Reality:
  Service A acquires Redlock (token=33)
  Service A experiences long GC pause (10 seconds)
  All locks expire during GC pause
  Service B acquires Redlock (token=34)
  Service A resumes, thinks it still has lock

  Without fencing tokens → Data corruption
```

**Critique Point 2: Doesn't Solve Distributed Lock Problems**

Kleppmann argues: If you're just using locks for efficiency (preventing duplicate work), use a simple single-instance lock. If you need correctness (preventing data corruption), use fencing tokens—Redlock without fencing doesn't guarantee correctness.

**Critique Point 3: Complexity vs. Benefit**

```
Complexity added:
  - Managing 5 Redis instances
  - Handling clock synchronization
  - Dealing with partial failures

Benefit gained:
  - Better availability (can survive 2 instance failures)

BUT:
  - Still not safe against timing issues without fencing
  - More complex than ZooKeeper locks which provide better guarantees
```

**Salvatore Sanfilippo's Response**:

Sanfilippo (Redlock creator) responded that:
- Redlock is for efficiency, not perfect correctness
- For correctness, use consensus systems like Raft/ZooKeeper
- Redlock provides better availability than single instance
- Many systems don't need absolute correctness guarantees

**Industry Consensus (as of 2026)**:

```
Use Redlock when:
  ✓ Duplicate work prevention (not data safety)
  ✓ Better availability needed than single Redis
  ✓ Fencing tokens used if data safety required

Avoid Redlock when:
  ✗ Need absolute correctness guarantees
  ✗ Have access to consensus systems (use those instead)
  ✗ Simple single-instance lock is sufficient
```

**Practical Reality**: Many companies use Redlock successfully for non-critical distributed locking, but use stronger systems (ZooKeeper, etcd) for critical safety.

### ZooKeeper Locks

**ZooKeeper** is a distributed coordination service that provides stronger guarantees than Redis-based locks. It uses the Zab consensus protocol (similar to Raft/Paxos) to ensure consistency.

**Architecture**:

```
                  ZooKeeper Ensemble

        ┌────────────────────────────────┐
        │   Leader      Follower Follower│
        │     │             │       │    │
        │     └─────────────┴───────┘    │
        │         [Replication]          │
        └────────────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
         Service A         Service B
```

**Key Differences from Redlock**:

| Aspect | Redlock | ZooKeeper |
|--------|---------|-----------|
| Consensus | No (independent instances) | Yes (Zab protocol) |
| Consistency | Best-effort | Linearizable |
| Fencing | Must add separately | Built-in (zxid) |
| Failure handling | Majority voting | Leader election |

**ZooKeeper Lock Implementation**:

ZooKeeper doesn't have a built-in "lock" primitive, but provides **ephemeral sequential znodes** that can be used to implement locks.

**How It Works**:

```
Lock represented as directory: /locks/my-resource/

Service A wants lock:
  1. CREATE /locks/my-resource/lock-0000000001 [ephemeral sequential]
  2. GET all children of /locks/my-resource/
  3. IF my node has smallest sequence number:
       I have the lock!
     ELSE:
       WATCH the node just before mine
       WAIT for notification

Service B wants lock:
  1. CREATE /locks/my-resource/lock-0000000002 [ephemeral sequential]
  2. GET all children of /locks/my-resource/
     [lock-0000000001, lock-0000000002]
  3. My node (002) is NOT smallest
  4. WATCH lock-0000000001
  5. WAIT...

Service A releases lock:
  1. DELETE /locks/my-resource/lock-0000000001

Service B receives notification:
  1. "lock-0000000001 was deleted"
  2. Check again: GET children
  3. [lock-0000000002]
  4. My node is now smallest → I have the lock!
```

**Visual Flow**:

```
State 1: Service A acquires lock
    /locks/my-resource/
        ├── lock-0000000001 (Service A) ← Smallest, has lock

State 2: Service B tries to acquire
    /locks/my-resource/
        ├── lock-0000000001 (Service A) ← Smallest, has lock
        └── lock-0000000002 (Service B) ← Waiting, watching 0001

State 3: Service C tries to acquire
    /locks/my-resource/
        ├── lock-0000000001 (Service A) ← Has lock
        ├── lock-0000000002 (Service B) ← Waiting, watching 0001
        └── lock-0000000003 (Service C) ← Waiting, watching 0002

State 4: Service A releases (deletes its node)
    /locks/my-resource/
        ├── lock-0000000002 (Service B) ← Now has lock (smallest)
        └── lock-0000000003 (Service C) ← Waiting, watching 0002
```

**Ephemeral Nodes: Automatic Cleanup**

**Ephemeral** means the node automatically disappears when the client's session ends:

```
Service A creates lock-0000000001 [ephemeral]

IF Service A:
  - Explicitly deletes → Node removed
  - Crashes → Session ends → Node automatically removed
  - Network partition → Session timeout → Node removed

This prevents deadlocks from crashed lock holders!
```

**Fencing with ZooKeeper: zxid**

ZooKeeper provides built-in fencing via **zxid** (ZooKeeper transaction ID):

```
Every state change in ZooKeeper gets a monotonically increasing zxid:

Service A creates lock node → zxid = 1001
Service B creates lock node → zxid = 1002
Service C creates lock node → zxid = 1003

Use zxid as fencing token when accessing shared resources!
```

**Example with Fencing**:

```
Service A                   ZooKeeper              Shared Resource
    │                             │                       │
    ├──CREATE lock node──────────>│                       │
    │<──SUCCESS (zxid=1001)────────│                       │
    │  [Has lock]                 │                       │
    │                             │                       │
    ├──UPDATE (zxid=1001)─────────────────────────────────>│
    │                             │         [Accept: new highest]
    │                             │                       │
T=30s│  Session timeout           │                       │
    │  [Node auto-deleted]        │                       │
    │                             │                       │
    │                             │<──CREATE lock node────Service B
    │                             │──SUCCESS (zxid=1002)─>│
    │                             │                       │
    │                             │  ├──UPDATE (zxid=1002)>│
    │                             │  │  [Accept: 1002 > 1001]
    │                             │                       │
    │  [Delayed message]          │                       │
    ├──UPDATE (zxid=1001)─────────────────────────────────>│
    │                             │      [Reject: 1001 < 1002]
```

**Benefits of ZooKeeper Locks**:

1. **Strong Consistency**: Linearizable reads/writes via consensus
2. **Automatic Cleanup**: Ephemeral nodes prevent deadlocks
3. **Built-in Fencing**: zxid provides ordering guarantee
4. **Fair Ordering**: FIFO queue of lock waiters
5. **Watch Notifications**: Efficient waiting (no polling)

**Drawbacks**:

1. **Operational Complexity**: Must run ZooKeeper ensemble (3-5 servers)
2. **Higher Latency**: Consensus adds overhead (typically 10-50ms)
3. **Not as Fast**: Slower than Redis for simple cases
4. **Learning Curve**: More complex API than simple Redis SET/DEL

**When to Use ZooKeeper Locks**:

```
✓ Critical operations requiring strong consistency
✓ When you need ordering guarantees (fair locks)
✓ When fencing is required
✓ Systems already using ZooKeeper for coordination
✓ Long-lived locks (hours/days)

✗ High-throughput, low-latency locking
✗ Simple duplicate prevention (Redis sufficient)
✗ Microservice environments without ZooKeeper infrastructure
```

**Modern Alternatives** (2026):

- **etcd**: Similar to ZooKeeper, uses Raft consensus, more modern API
- **Consul**: Service mesh with distributed lock support
- **Google Chubby**: Internal Google system (not public)
- **Database-based locks**: PostgreSQL advisory locks, for simplicity

---

## Idempotency

### Overview

**Idempotency** means that performing an operation multiple times has the same effect as performing it once. This property is crucial in distributed systems where messages can be duplicated, retried, or delivered out of order.

**Layman explanation**: Pressing an elevator button multiple times doesn't call multiple elevators—it has the same effect as pressing it once. The elevator button is idempotent.

**Mathematical Definition**:

```
An operation f is idempotent if:
  f(f(x)) = f(x)

Examples:

Idempotent:
  • Set value: SET(x, 5) → SET(SET(x, 5), 5) = SET(x, 5)
  • Delete: DELETE(x) → DELETE(DELETE(x)) = DELETE(x)
  • Absolute value: ||-5|| = |-5| = 5

NOT Idempotent:
  • Increment: INCREMENT(x) → INCREMENT(INCREMENT(x)) ≠ INCREMENT(x)
  • Append: APPEND(list, item) → different list each time
  • Send email: Sending twice → User gets two emails
```

### Idempotent Operations Definition

**Types of Idempotent Operations**:

**1. Naturally Idempotent (Inherent)**

Operations that are idempotent by their nature:

```
PUT /users/123
{
  "name": "Alice",
  "email": "alice@example.com"
}

First call:  Creates or replaces user 123
Second call: Replaces user 123 with same data (same result)
Result: User 123 exists with specified data (idempotent ✓)

versus

POST /users
{
  "name": "Alice",
  "email": "alice@example.com"
}

First call:  Creates user (id: 123)
Second call: Creates another user (id: 456)
Result: Two users created (NOT idempotent ✗)
```

**Examples of naturally idempotent operations**:

```
• SET operations:        SET balance = 1000
• DELETE operations:     DELETE user WHERE id = 123
• Read operations:       GET /users/123
• Absolute updates:      UPDATE status = 'CANCELLED'
```

**2. Made Idempotent (Designed)**

Operations that aren't naturally idempotent, but we design them to be:

```
POST /payments  (naturally not idempotent)
↓
POST /payments with idempotency-key: "abc123"  (made idempotent)
```

### Why Idempotency Matters in Distributed Systems

**Problem 1: Network Retries**

```
Client                      Payment Service              Database
  │                                │                         │
  ├──POST /charge $100────────────>│                         │
  │                                ├──INSERT payment────────>│
  │                                │<──Success───────────────┤
  │                                │                         │
  │ ← Network timeout (no response)│ [Response lost]         │
  │                                │                         │
  │  Client doesn't know if it succeeded!                    │
  │  Safe choice: Retry                                      │
  │                                │                         │
  ├──POST /charge $100 (retry)────>│                         │
  │                                ├──INSERT payment────────>│
  │                                │<──Success───────────────┤
  │                                │                         │
  │<──Success────────────────────────                         │

  Result: Customer charged $200 instead of $100! 💥
```

**Problem 2: Message Queue Processing**

```
Message Queue:    [Process Order 123]
                         │
                         ├────> Worker A starts processing
                         │       [Worker A crashes]
                         │       [Message not acknowledged]
                         │       [Message redelivered]
                         │
                         └────> Worker B processes same message

If not idempotent:
  Order 123 processed twice
  Customer billed twice
  Inventory decremented twice
```

**Problem 3: Load Balancer Retries**

```
User → Load Balancer → Service Instance A
                  ↓
            [Instance A slow to respond]
            [Load balancer timeout]
            [Retry to Instance B]

Both Instance A and B process the request!
```

### Idempotency Keys

An **idempotency key** is a unique identifier provided by the client to make non-idempotent operations idempotent. The server uses this key to detect and handle duplicate requests.

**Basic Concept**:

```
Client generates unique key: "payment_abc123xyz"

Request 1:
  POST /payments
  Idempotency-Key: payment_abc123xyz
  Body: { amount: 100, card: "..." }
  → Server processes payment

Request 2 (retry/duplicate):
  POST /payments
  Idempotency-Key: payment_abc123xyz  [SAME KEY]
  Body: { amount: 100, card: "..." }
  → Server detects duplicate, returns cached response
```

**How It Works**:

```
Server Processing Flow:

1. Receive request with idempotency key
2. Check if key exists in idempotency store

   IF key exists:
     IF request still processing:
       WAIT and then return result
     IF request completed:
       RETURN cached response (same status, body, headers)
     IF request failed:
       Either return cached error OR allow retry (policy decision)

   IF key does NOT exist:
     Store key with status: "PROCESSING"
     Execute operation
     Store result with key
     Return response
```

**Database Schema Example**:

```
Table: idempotency_records

Columns:
  idempotency_key    VARCHAR(255)  PRIMARY KEY
  created_at         TIMESTAMP     NOT NULL
  completed_at       TIMESTAMP     NULL
  status             ENUM          ('PROCESSING', 'COMPLETED', 'FAILED')
  request_hash       VARCHAR(64)   -- Hash of request body
  response_status    INTEGER       -- HTTP status code
  response_body      TEXT          -- Cached response
  response_headers   JSON          -- Cached headers

Indexes:
  PRIMARY KEY (idempotency_key)
  INDEX (created_at)  -- For cleanup of old records
```

**Detailed Flow Diagram**:

```
Request 1 (new idempotency key):

Client                         Server                    Database
  │                              │                           │
  ├─POST payment                 │                           │
  │ Idempotency-Key: abc123      │                           │
  │                              │                           │
  │                              ├─SELECT * WHERE key='abc123'
  │                              │<─Not found────────────────┤
  │                              │                           │
  │                              ├─INSERT (key='abc123',    │
  │                              │  status='PROCESSING')     │
  │                              │<─Success──────────────────┤
  │                              │                           │
  │                              ├─[Charge payment]          │
  │                              │                           │
  │                              ├─UPDATE (key='abc123',     │
  │                              │  status='COMPLETED',      │
  │                              │  response='...')          │
  │                              │<─Success──────────────────┤
  │                              │                           │
  │<─200 OK {paymentId: 789}─────│                           │


Request 2 (duplicate key):

Client                         Server                    Database
  │                              │                           │
  ├─POST payment (retry)         │                           │
  │ Idempotency-Key: abc123      │                           │
  │                              │                           │
  │                              ├─SELECT * WHERE key='abc123'
  │                              │<─Found: COMPLETED─────────┤
  │                              │  response={paymentId:789} │
  │                              │                           │
  │<─200 OK {paymentId: 789}─────│ [Cached response]         │
  │ [Same response, no duplicate charge]
```

**Key Generation Strategies**:

**1. Client-Generated UUID**:
```
Pros: Simple, globally unique
Cons: Client must store/retry with same key

key = UUID.v4()  // "550e8400-e29b-41d4-a716-446655440000"
```

**2. Hash of Request Content**:
```
Pros: Automatic deduplication of identical requests
Cons: Different requests might hash to same value (unlikely but possible)

key = SHA256(userId + amount + currency + timestamp.truncate(minute))
```

**3. Domain-Based Keys**:
```
Pros: Meaningful, easier to debug
Cons: Must ensure uniqueness

key = "order_" + orderId + "_payment_" + paymentAttempt
```

**4. Combination**:
```
key = userId + "_" + resourceType + "_" + UUID.v4()
// "user123_payment_550e8400-e29b-41d4-a716-446655440000"
```

**Idempotency Key Best Practices**:

```
1. Expiration:
   Delete old idempotency records after TTL (e.g., 24 hours)
   Prevents unbounded growth

2. Request Validation:
   Hash request body and store with key
   If retry has different body, reject as invalid

3. Status Handling:
   PROCESSING: Return 409 Conflict or wait
   COMPLETED: Return cached success response
   FAILED: Policy decision (allow retry or return cached error)

4. Scope:
   Key should be scoped to user/tenant
   "user123_abc" different from "user456_abc"

5. Headers:
   Standard header: Idempotency-Key
   Also support: X-Idempotency-Key, Request-Id
```

### Implementing Idempotency

**Pattern 1: Database-Level Idempotency**

Use unique constraints to enforce idempotency:

```
Table: payments

CREATE TABLE payments (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL,
  idempotency_key VARCHAR(255) NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, idempotency_key)
);

Processing:
  Try to INSERT payment with idempotency_key

  IF insert succeeds:
    [New payment, process it]
    Charge card
    Update status to 'COMPLETED'
    Return success

  IF insert fails (unique constraint violation):
    [Duplicate detected]
    SELECT existing payment
    Return existing payment details
```

**Pattern 2: Two-Phase Insert**

Separate idempotency tracking from business logic:

```
BEGIN TRANSACTION

  -- Phase 1: Record idempotency
  INSERT INTO idempotency_records (key, status)
  VALUES ('abc123', 'PROCESSING')
  [If fails due to duplicate, return cached result]

  -- Phase 2: Execute business logic
  INSERT INTO payments (user_id, amount)
  VALUES (123, 100.00)

  Charge external payment processor

  -- Phase 3: Mark complete
  UPDATE idempotency_records
  SET status = 'COMPLETED', response = {...}
  WHERE key = 'abc123'

COMMIT
```

**Pattern 3: Distributed Cache**

Use Redis/Memcached for high-performance idempotency:

```
Processing Flow:

1. Check cache: GET idempotency:abc123

   IF exists:
     RETURN cached_response

2. Set lock: SET idempotency:abc123 "PROCESSING" NX EX 300
   [NX = only if not exists, EX 300 = expire in 5 min]

   IF lock failed:
     [Another instance processing]
     WAIT and retry OR return 409 Conflict

3. Execute operation

4. Cache result: SET idempotency:abc123 "COMPLETED:{response}" EX 86400
   [Cache for 24 hours]

5. Return response
```

**Pattern 4: Event Sourcing with Idempotency**

Append-only event log with idempotent event IDs:

```
Events:
  event_id: "payment_abc123"   ← Idempotency key
  event_type: "PaymentProcessed"
  data: { amount: 100, userId: 123 }

Processing:
  Try to append event with event_id

  IF event_id already exists:
    [Duplicate event, skip processing]
    Return existing result
  ELSE:
    Append event
    Process event
    Update state
```

**Handling Edge Cases**:

**1. Request Body Mismatch**

```
Request 1:
  Idempotency-Key: abc123
  Body: { amount: 100 }

Request 2:
  Idempotency-Key: abc123
  Body: { amount: 200 }  ← Different body!

Solution:
  Store hash of request body with key
  On duplicate key, compare hashes
  IF hash mismatch: Return 422 Unprocessable Entity
```

**2. Concurrent Requests with Same Key**

```
Request A and B arrive simultaneously with key "abc123"

Option 1: Pessimistic Lock
  First request sets "PROCESSING" status
  Second request waits or returns 409 Conflict

Option 2: Optimistic Lock
  Both try to insert
  Database unique constraint ensures only one succeeds
  Loser retries and gets cached result
```

**3. Long-Running Operations**

```
Problem: Request takes 10 minutes, retry comes after 30 seconds

Solution: Return status endpoint

  POST /payments → Returns 202 Accepted
  {
    "status": "processing",
    "statusUrl": "/payments/abc123/status"
  }

  Client polls: GET /payments/abc123/status
  Returns: "processing" or "completed" with result
```

**4. Failed Operations**

```
Should failed requests be retryable?

Policy A: Cache failure, don't retry
  First attempt fails → Cache failure
  Retry → Return cached failure
  [Prevents retry of transient failures]

Policy B: Allow retry on failure
  First attempt fails → Don't cache
  Retry → Attempt again
  [Allows recovery from transient issues]

Recommendation: Distinguish error types
  • Permanent errors (invalid card): Cache failure
  • Transient errors (timeout): Allow retry
```

### At-Least-Once with Idempotency = Exactly-Once

**The Fundamental Trade-Off in Distributed Systems**:

```
Message Delivery Guarantees:

At-Most-Once:
  • Send message, don't retry
  • Fast, no duplicates
  • BUT: Messages can be lost ✗

At-Least-Once:
  • Retry until acknowledged
  • No messages lost ✓
  • BUT: Duplicates possible ✗

Exactly-Once:
  • Deliver each message exactly once
  • Ideal, but impossible in pure distributed systems ✗
  • Requires idempotent processing
```

**The Semantic Equivalence**:

```
At-Least-Once Delivery + Idempotent Processing = Exactly-Once Semantics

Explanation:
  • Messages may be delivered multiple times
  • BUT idempotent processing ensures same effect as processing once
  • Result: Appears as exactly-once to the application
```

**Example: Order Processing**

```
Message Queue (at-least-once):
  Message: "Process Order 123"
  Delivered to Worker A
  Worker A crashes before ACK
  Message redelivered to Worker B
  [Duplicate delivery]

Worker Processing (idempotent):

  Function ProcessOrder(orderId):
    key = "order_processed_" + orderId

    IF AlreadyProcessed(key):
      RETURN "Already processed"  // Duplicate detected

    MarkAsProcessing(key)
    ChargeCustomer(orderId)
    ShipOrder(orderId)
    MarkAsCompleted(key)

  Result: Order charged and shipped exactly once ✓
```

**Real-World Example: Kafka with Idempotent Producers**:

```
Kafka Producer (2026):

producer.send(record)
  ↓
[Network error, uncertain if sent]
  ↓
producer.send(record)  ← Retry
  ↓
Kafka detects duplicate via sequence number
  ↓
Duplicate ignored, appears exactly-once
```

**Implementation Pattern**:

```
Message Processing Loop:

WHILE true:
  message = queue.receive()  // At-least-once queue

  idempotencyKey = message.id

  IF ProcessingCompleted(idempotencyKey):
    // Already processed, skip
    queue.acknowledge(message)
    CONTINUE

  TRY:
    RecordStartProcessing(idempotencyKey)
    ExecuteBusinessLogic(message.data)
    RecordCompleted(idempotencyKey)
    queue.acknowledge(message)
  CATCH error:
    LogError(error)
    queue.nack(message)  // Requeue for retry
```

**Benefits**:

```
✓ Reliability: No lost messages (at-least-once)
✓ Correctness: No duplicate effects (idempotency)
✓ Simplicity: Easier than building exactly-once infrastructure
✓ Performance: Can use efficient at-least-once systems
```

**When This Pattern Works Best**:

```
• Message queues: RabbitMQ, AWS SQS, Azure Service Bus
• Stream processing: Kafka (with idempotent consumer logic)
• API retries: HTTP clients with retry logic
• Distributed workflows: Temporal, Airflow
```

---

## Event Sourcing

### Overview

**Event Sourcing** is a pattern where you store the full history of state changes as a sequence of events, rather than just storing the current state. The current state is derived by replaying all events from the beginning.

**Layman explanation**: Instead of keeping just your current bank balance ($1,000), you keep a complete transaction log: "Deposit $500, Withdraw $100, Deposit $600, Withdraw $0..." Your balance is calculated by replaying all transactions.

**Traditional CRUD vs. Event Sourcing**:

```
Traditional (CRUD - Create, Read, Update, Delete):

Users Table:
  id  | name   | email              | status
  123 | Alice  | alice@example.com  | ACTIVE

Update: UPDATE users SET status='INACTIVE' WHERE id=123
Result: Previous status lost forever

---

Event Sourcing:

Events Table:
  event_id | aggregate_id | event_type      | data                        | timestamp
  1        | user-123     | UserCreated     | {name: "Alice", ...}        | T1
  2        | user-123     | EmailChanged    | {email: "alice@ex.com"}     | T2
  3        | user-123     | StatusChanged   | {status: "INACTIVE"}        | T3

Current state: Replay events 1, 2, 3 → User 123 is INACTIVE
Full history: Can see user was ACTIVE before T3
```

**Core Principles**:

1. **Events are immutable**: Never update or delete events
2. **Events are the source of truth**: State is derived from events
3. **Complete audit trail**: Every change is recorded
4. **Time travel**: Can reconstruct state at any point in time

### Event Store Design

The **event store** is a specialized database optimized for appending and reading sequences of events.

**Conceptual Structure**:

```
Event Store
├── Streams (one per aggregate/entity)
│   ├── user-123
│   │   ├── Event 1: UserCreated
│   │   ├── Event 2: EmailChanged
│   │   └── Event 3: StatusChanged
│   │
│   ├── order-456
│   │   ├── Event 1: OrderCreated
│   │   ├── Event 2: ItemAdded
│   │   ├── Event 3: OrderPlaced
│   │   └── Event 4: PaymentReceived
│   │
│   └── account-789
│       ├── Event 1: AccountOpened
│       └── Event 2: DepositMade
│
└── Global Event Log (all events in order)
    ├── Event 1: UserCreated (user-123)
    ├── Event 2: AccountOpened (account-789)
    ├── Event 3: OrderCreated (order-456)
    ├── Event 4: EmailChanged (user-123)
    └── ...
```

**Database Schema** (Relational implementation):

```
Table: events

Columns:
  event_id          BIGSERIAL     PRIMARY KEY
  stream_id         VARCHAR(255)  NOT NULL     -- Aggregate identifier (e.g., "user-123")
  event_type        VARCHAR(100)  NOT NULL     -- Event class name
  event_version     INTEGER       NOT NULL     -- Version in this stream
  data              JSONB         NOT NULL     -- Event payload
  metadata          JSONB                      -- Correlation ID, user ID, etc.
  timestamp         TIMESTAMP     NOT NULL     DEFAULT NOW()

Constraints:
  UNIQUE(stream_id, event_version)  -- Ensures ordered sequence per stream

Indexes:
  PRIMARY KEY (event_id)                    -- Global ordering
  INDEX (stream_id, event_version)          -- Stream lookup
  INDEX (event_type)                        -- Query by event type
  INDEX (timestamp)                         -- Time-based queries
  INDEX ((metadata->>'correlation_id'))     -- Correlation tracking
```

**Example Events**:

```
Event 1:
{
  "event_id": 1001,
  "stream_id": "user-123",
  "event_type": "UserRegistered",
  "event_version": 1,
  "data": {
    "user_id": "123",
    "email": "alice@example.com",
    "name": "Alice",
    "registered_at": "2026-02-06T10:00:00Z"
  },
  "metadata": {
    "correlation_id": "req-abc123",
    "user_agent": "Mozilla/5.0...",
    "ip_address": "192.168.1.1"
  },
  "timestamp": "2026-02-06T10:00:00Z"
}

Event 2:
{
  "event_id": 1002,
  "stream_id": "user-123",
  "event_type": "EmailVerified",
  "event_version": 2,
  "data": {
    "user_id": "123",
    "verified_at": "2026-02-06T10:05:00Z"
  },
  "metadata": {...},
  "timestamp": "2026-02-06T10:05:00Z"
}
```

**Key Design Decisions**:

**1. Stream Granularity**

```
Option A: One stream per aggregate instance
  Streams: user-123, user-456, order-789
  ✓ Natural aggregate boundary
  ✓ Easy to load single aggregate
  ✗ Many small streams

Option B: One stream per aggregate type
  Streams: all-users, all-orders
  ✓ Fewer streams
  ✗ Harder to query single aggregate
  ✗ Concurrency issues

Recommendation: One stream per aggregate instance (Option A)
```

**2. Event Ordering**

```
Global ordering (event_id):
  All events have sequential ID across entire system
  Enables building global projections
  Required for event processors that need total ordering

Per-stream ordering (event_version):
  Events within one stream are versioned: 1, 2, 3...
  Enables optimistic concurrency control
  Detects concurrent modifications
```

**3. Optimistic Concurrency Control**

```
Append Operation:

Function AppendEvent(streamId, event, expectedVersion):

  currentVersion = GetLatestVersion(streamId)

  IF currentVersion != expectedVersion:
    THROW ConcurrencyException("Stream modified concurrently")

  newVersion = currentVersion + 1

  INSERT INTO events (
    stream_id,
    event_type,
    event_version,  ← UNIQUE constraint enforces
    data
  ) VALUES (
    streamId,
    event.type,
    newVersion,
    event.data
  )

  RETURN newVersion
```

**Handling Concurrent Writes**:

```
Timeline:

T1: Process A reads stream user-123, sees version 5
T1: Process B reads stream user-123, sees version 5

T2: Process A appends event (expected version: 5)
    → Success, version becomes 6

T3: Process B appends event (expected version: 5)
    → FAILS! (current version is 6, not 5)
    → Process B must:
       1. Reload events 1-6
       2. Re-apply business logic
       3. Retry append with expected version: 6
```

**4. Partition Strategy** (for scalability):

```
Partition by stream_id hash:

Partition 0: user-001, user-127, order-253, ...
Partition 1: user-002, user-128, order-254, ...
Partition 2: user-003, user-129, order-255, ...

Benefits:
  • All events for one aggregate in same partition
  • Enables parallel processing
  • Maintains ordering within stream
```

### Event Schema Evolution

Over time, event structures need to change. **Event schema evolution** handles this while maintaining backward compatibility with old events.

**The Challenge**:

```
Year 2025 events:
{
  "event_type": "UserRegistered",
  "data": {
    "user_id": "123",
    "email": "alice@example.com"
  }
}

Year 2026 events (added name field):
{
  "event_type": "UserRegistered",
  "data": {
    "user_id": "123",
    "email": "alice@example.com",
    "name": "Alice"              ← New field
  }
}

Problem: How to replay 2025 events with 2026 code?
```

**Evolution Strategies**:

**Strategy 1: Versioned Events**

```
Event Class Versions:

class UserRegisteredV1:
  user_id: String
  email: String

class UserRegisteredV2:
  user_id: String
  email: String
  name: String

Storage:
{
  "event_type": "UserRegistered",
  "event_schema_version": 1,    ← Explicit version
  "data": {...}
}

Handling:
  IF event_schema_version == 1:
    event = DeserializeAs(UserRegisteredV1)
    event = UpgradeToV2(event)  // Set name = ""
  ELSE IF event_schema_version == 2:
    event = DeserializeAs(UserRegisteredV2)
```

**Strategy 2: Upcasting**

Transform old events to new format when reading:

```
Function LoadEvents(streamId):
  rawEvents = Database.LoadEvents(streamId)

  upcastedEvents = []
  FOR each rawEvent in rawEvents:
    upcastedEvent = Upcaster.Apply(rawEvent)
    upcastedEvents.Add(upcastedEvent)

  RETURN upcastedEvents

Upcaster:
  UserRegisteredV1 → UserRegisteredV2:
    Add field: name = "(unknown)"

  OrderPlacedV1 → OrderPlacedV2:
    Rename field: total_price → total_amount
```

**Strategy 3: Weak Schema (JSON flexibility)**

```
Events stored as flexible JSON:

Old event:
{
  "user_id": "123",
  "email": "alice@example.com"
}

New event:
{
  "user_id": "123",
  "email": "alice@example.com",
  "name": "Alice"
}

Code handles missing fields:
  name = event.data.get("name") or "(unknown)"
```

**Strategy 4: Copy-and-Replace**

For breaking changes, create new event type:

```
Old: UserRegistered
New: UserRegisteredV2

Both event types coexist
Event handlers support both:

  ON UserRegistered:
    HandleLegacyRegistration()

  ON UserRegisteredV2:
    HandleNewRegistration()
```

**Best Practices**:

```
1. Never delete fields (breaks old events)
   ✗ Remove email field
   ✓ Add email_deprecated field, keep email

2. Make new fields optional
   ✓ name: String? = null
   ✗ name: String (required)

3. Use default values for missing fields
   ✓ name = data.get("name", default="Unknown")

4. Version your event schemas
   Store schema version in metadata

5. Test replay with old events
   Maintain old event samples for testing

6. Document schema changes
   Track evolution in changelog
```

**Schema Registry** (advanced):

```
Centralized schema management:

Schema Registry
  ├── UserRegistered (version 1)
  ├── UserRegistered (version 2)
  ├── OrderPlaced (version 1)
  └── ...

Each event references schema by ID:
{
  "event_type": "UserRegistered",
  "schema_id": "user-registered-v2-schema-123",
  "data": {...}
}

Benefits:
  • Enforced schema validation
  • Centralized evolution tracking
  • Compatibility checks before deployment
```

### Snapshotting Strategies

**The Performance Problem**:

```
Stream with 1,000,000 events:

To get current state:
  Load 1,000,000 events from database
  Replay 1,000,000 events

Time: Could take minutes! ✗
```

**Solution: Snapshots**

A **snapshot** is a saved state at a specific point in time, allowing replay from that point instead of from the beginning.

```
Without snapshot:
  [Event 1] → [Event 2] → ... → [Event 1,000,000] → Current State

With snapshot at Event 500,000:
  [Snapshot] → [Event 500,001] → ... → [Event 1,000,000] → Current State

Performance: 50% fewer events to replay!
```

**Snapshot Storage**:

```
Table: snapshots

Columns:
  stream_id          VARCHAR(255)  NOT NULL
  snapshot_version   INTEGER       NOT NULL  -- Last event version included
  state              JSONB         NOT NULL  -- Serialized aggregate state
  created_at         TIMESTAMP     NOT NULL

PRIMARY KEY (stream_id, snapshot_version)
INDEX (stream_id)  -- Latest snapshot lookup
```

**Example**:

```
Snapshot for user-123 at version 5000:

{
  "stream_id": "user-123",
  "snapshot_version": 5000,
  "state": {
    "user_id": "123",
    "email": "alice@example.com",
    "name": "Alice",
    "status": "ACTIVE",
    "total_orders": 42,
    "loyalty_points": 850
  },
  "created_at": "2026-02-01T00:00:00Z"
}
```

**Loading with Snapshot**:

```
Function LoadAggregate(streamId):

  // 1. Try to load latest snapshot
  snapshot = Database.GetLatestSnapshot(streamId)

  IF snapshot exists:
    state = Deserialize(snapshot.state)
    startVersion = snapshot.snapshot_version + 1
  ELSE:
    state = NewEmptyState()
    startVersion = 1

  // 2. Load events after snapshot
  events = Database.LoadEvents(
    streamId,
    fromVersion: startVersion
  )

  // 3. Replay events onto snapshot
  FOR each event in events:
    state = state.Apply(event)

  RETURN state
```

**Snapshot Creation Strategies**:

**Strategy 1: Periodic Snapshots** (time-based)

```
Every N hours/days:
  Create snapshot for all active streams

Example: Daily snapshot job
  FOR each stream in active_streams:
    IF no snapshot in last 24 hours:
      CreateSnapshot(stream)
```

**Strategy 2: Event Count Threshold**

```
After every N events:
  Create new snapshot

Example: Snapshot every 1000 events
  ON event appended:
    IF (currentVersion % 1000) == 0:
      CreateSnapshot(streamId)
```

**Strategy 3: On-Demand Snapshots**

```
When loading aggregate:
  IF (events_since_snapshot > threshold):
    Create new snapshot after loading

Example:
  Load aggregate (replay 2000 events)
  Current version: 7000
  Last snapshot: version 5000
  After loading, save snapshot at version 7000
```

**Strategy 4: Hybrid**

```
Combine multiple strategies:
  • Snapshot every 5000 events (event threshold)
  • OR snapshot daily (time threshold)
  • Whichever comes first
```

**Snapshot Deletion Policy**:

```
Keep only N latest snapshots per stream:

Snapshots for user-123:
  Version 1000  (created 30 days ago)  ← Delete
  Version 5000  (created 10 days ago)  ← Keep
  Version 10000 (created 2 days ago)   ← Keep
  Version 15000 (created today)        ← Keep (latest)

Reason: Old snapshots rarely needed, consume storage
```

**Trade-offs**:

```
Frequent snapshots:
  ✓ Faster aggregate loading
  ✗ More storage space
  ✗ Snapshot creation overhead

Infrequent snapshots:
  ✓ Less storage
  ✗ Slower aggregate loading
  ✗ More events to replay
```

### Rebuilding State from Events

**The Core Operation**: Taking a sequence of events and computing the current state.

**Aggregate State Machine**:

```
Initial State → [Event 1] → State 1 → [Event 2] → State 2 → ... → Current State
```

**Example: User Aggregate**

```
Initial State:
  { }  (empty)

Event 1: UserRegistered
  {
    user_id: "123",
    email: "alice@example.com",
    name: "Alice"
  }

State after Event 1:
  {
    user_id: "123",
    email: "alice@example.com",
    name: "Alice",
    status: "UNVERIFIED",
    created_at: <Event 1 timestamp>
  }

Event 2: EmailVerified
  {
    user_id: "123",
    verified_at: "2026-02-06T10:05:00Z"
  }

State after Event 2:
  {
    user_id: "123",
    email: "alice@example.com",
    name: "Alice",
    status: "ACTIVE",           ← Changed
    verified_at: "2026-02-06T10:05:00Z",  ← Added
    created_at: <Event 1 timestamp>
  }

Event 3: EmailChanged
  {
    user_id: "123",
    new_email: "alice.smith@example.com"
  }

State after Event 3:
  {
    user_id: "123",
    email: "alice.smith@example.com",  ← Changed
    name: "Alice",
    status: "ACTIVE",
    verified_at: "2026-02-06T10:05:00Z",
    created_at: <Event 1 timestamp>
  }
```

**Replay Implementation**:

```
Aggregate Class:

Class UserAggregate:
  state = {}

  Function Apply(event):
    MATCH event.type:
      CASE "UserRegistered":
        ApplyUserRegistered(event)
      CASE "EmailVerified":
        ApplyEmailVerified(event)
      CASE "EmailChanged":
        ApplyEmailChanged(event)

  Function ApplyUserRegistered(event):
    state.user_id = event.data.user_id
    state.email = event.data.email
    state.name = event.data.name
    state.status = "UNVERIFIED"
    state.created_at = event.timestamp

  Function ApplyEmailVerified(event):
    state.status = "ACTIVE"
    state.verified_at = event.data.verified_at

  Function ApplyEmailChanged(event):
    state.email = event.data.new_email

Loading:
  user = new UserAggregate()
  events = LoadEvents("user-123")
  FOR each event in events:
    user.Apply(event)
  RETURN user.state
```

**Projections: Read Models**

Different views of the data optimized for queries:

```
Event Stream (write model):
  Events: UserRegistered, EmailVerified, OrderPlaced, OrderShipped...

Projection 1: User Detail View (read model)
  Table: user_details
  {
    user_id: "123",
    name: "Alice",
    email: "alice@example.com",
    status: "ACTIVE"
  }

Projection 2: User Orders View (read model)
  Table: user_orders
  {
    user_id: "123",
    total_orders: 5,
    total_spent: 450.00,
    last_order_date: "2026-02-05"
  }

Projection 3: User Activity Timeline (read model)
  Table: user_activity
  [
    {user_id: "123", action: "Registered", timestamp: "2026-01-01"},
    {user_id: "123", action: "Verified Email", timestamp: "2026-01-01"},
    {user_id: "123", action: "Placed Order", timestamp: "2026-01-15"}
  ]
```

**Projection Updates**:

```
Event Processor (background service):

WHILE true:
  event = EventStore.ReadNext()

  MATCH event.type:
    CASE "UserRegistered":
      UserDetailsProjection.Handle(event)

    CASE "OrderPlaced":
      UserOrdersProjection.Handle(event)
      UserActivityProjection.Handle(event)

  MarkEventProcessed(event.event_id)

Example UserOrdersProjection:

Function Handle(OrderPlaced event):
  UPDATE user_orders
  SET
    total_orders = total_orders + 1,
    total_spent = total_spent + event.data.amount,
    last_order_date = event.timestamp
  WHERE user_id = event.data.user_id
```

**Rebuilding Projections**:

```
If projection becomes corrupted or schema changes:

1. Stop projection processor
2. Drop projection tables
3. Recreate tables with new schema
4. Replay ALL events from beginning:

   FOR each event in EventStore (from event_id = 1):
     Projection.Handle(event)

5. Restart projection processor

Time: Might take hours for large event stores
Solution: Use snapshots or optimize replay
```

### Event Sourcing vs CRUD

**Side-by-Side Comparison**:

| Aspect | CRUD | Event Sourcing |
|--------|------|----------------|
| **Data Storage** | Current state only | Full history of changes |
| **Updates** | Overwrite previous value | Append new event |
| **History** | Lost (unless audited separately) | Preserved automatically |
| **Queries** | Direct from tables | Rebuild from events or use projections |
| **Complexity** | Simple | More complex |
| **Debug** | Current state only | Can replay to any point |
| **Performance (writes)** | Fast updates | Fast appends |
| **Performance (reads)** | Fast (indexed) | Slower (rebuild) unless projections used |

**When to Use Event Sourcing**:

```
✓ Domain with complex business logic
✓ Audit trail is critical (finance, healthcare, legal)
✓ Need to analyze historical data
✓ Regulatory compliance requires complete history
✓ Temporal queries ("what was the state on Jan 1?")
✓ Event-driven architecture
✓ Domain-driven design with aggregates

Examples:
  • Banking: Full transaction history required
  • E-commerce: Order lifecycle tracking
  • Healthcare: Patient treatment history
  • Git: Version control (every commit is an event!)
```

**When NOT to Use Event Sourcing**:

```
✗ Simple CRUD applications
✗ Current state is all that matters
✗ Team lacks event sourcing expertise
✗ High read volume, low write volume (CRUD is faster)
✗ Rapid prototyping (CRUD is simpler)
✗ External systems expect current-state queries

Examples:
  • Simple blogs, content management
  • Configuration settings
  • User preferences
  • Reference data
```

**Hybrid Approach**:

```
Use Event Sourcing for critical domains:
  • Orders (event sourced)
  • Payments (event sourced)
  • Inventory (event sourced)

Use CRUD for supporting domains:
  • User profiles (CRUD)
  • Product catalog (CRUD)
  • Email templates (CRUD)
```

**Migration Path**:

```
From CRUD to Event Sourcing:

Phase 1: Start logging events alongside CRUD
  • Write to both database and event store
  • Verify events match CRUD operations

Phase 2: Build projections from events
  • Create read models from event stream
  • Compare with CRUD tables for correctness

Phase 3: Switch reads to projections
  • Route queries to projections
  • Keep CRUD as backup

Phase 4: Make events primary
  • Write only to event store
  • Generate CRUD tables as projections

Phase 5: Remove legacy CRUD
  • Retire old tables
  • Event store is source of truth
```

**Common Pitfalls**:

```
1. Storing too much in events
   ✗ Include entire aggregate in every event
   ✓ Store only what changed

2. Not using projections
   ✗ Replay events for every query
   ✓ Maintain optimized read models

3. Ignoring event versioning
   ✗ Breaking changes to event schema
   ✓ Plan for evolution from day one

4. Over-engineering
   ✗ Use event sourcing everywhere
   ✓ Use where it adds value

5. Neglecting performance
   ✗ No snapshots, slow aggregate loading
   ✓ Strategic snapshotting

6. Weak event names
   ✗ "DataUpdated", "StateChanged"
   ✓ "EmailVerified", "OrderShipped" (domain language)
```

---

## Conclusion

Distributed transactions are challenging because we're working across multiple independent systems that can fail, get delayed, or become partitioned. The patterns covered in this guide each make different trade-offs:

**Coordination Protocols (2PC, 3PC)**:
- Strong consistency, but blocking and fragile
- Best within trusted, low-latency environments

**Sagas**:
- Eventual consistency with explicit compensations
- Excellent for microservices and long-running processes

**Distributed Locks**:
- Coordination primitive for shared resources
- Choose implementation based on consistency needs

**Idempotency**:
- Essential technique for reliability
- Enables at-least-once delivery with exactly-once semantics

**Event Sourcing**:
- Complete audit trail and time-travel capabilities
- Adds complexity but powerful for complex domains

The key is choosing the right pattern for your specific requirements, understanding the trade-offs, and implementing carefully with proper testing and monitoring.