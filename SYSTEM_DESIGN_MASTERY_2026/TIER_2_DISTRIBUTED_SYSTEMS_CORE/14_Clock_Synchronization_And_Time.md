# Clock Synchronization & Time in Distributed Systems

## Introduction

In distributed systems, understanding "when" something happened is surprisingly difficult. Unlike a single computer where you can check the system clock, distributed systems have multiple machines, each with its own clock, and these clocks are never perfectly synchronized.

**Why time matters in distributed systems**:
- **Ordering events**: Did Event A happen before Event B?
- **Detecting conflicts**: Did two users edit the same document simultaneously?
- **Data consistency**: Which version of data is newer?
- **Debugging**: What sequence of events led to a failure?
- **Compliance**: Proving when a transaction occurred

**The fundamental challenge**: There is no global "now" in a distributed system. Each machine has its own notion of time, and these can drift apart.

**Layman explanation**: Imagine coordinating a group project where everyone works from home in different time zones with watches that run at slightly different speeds. How do you agree on which task was completed first?

---

## Logical Clocks

**Logical clocks** abandon the idea of measuring "real time" and instead focus on the **ordering of events**. They answer: "Did Event A happen before Event B?" without caring about the exact time.

**Key insight**: In many distributed systems, we don't care about the actual time something happened—we only care about the order of events.

### Lamport Timestamps

**Lamport timestamps** are the simplest form of logical clock, invented by Leslie Lamport in 1978. Each process maintains a counter that increments with each event.

**How it works**:

```
Each process maintains a counter (initially 0)

On local event (computation, I/O):
  1. Increment counter
  2. Assign counter value to event

On sending a message:
  1. Increment counter
  2. Include counter value in message

On receiving a message:
  1. Set counter = max(local_counter, message_counter) + 1
  2. Assign counter value to receive event
```

**Example Execution**:

```
Process A                    Process B                    Process C
counter = 0                  counter = 0                  counter = 0

[Event: Start]               [Event: Start]               [Event: Start]
counter = 1                  counter = 1                  counter = 1

[Send msg to B]
counter = 2
msg: "Hello", ts=2 ────────> [Receive msg from A]
                             counter = max(1, 2) + 1 = 3

                             [Local event]
                             counter = 4

                             [Send msg to C]
                             counter = 5
                             msg: "Data", ts=5 ─────────> [Receive msg from B]
                                                          counter = max(1, 5) + 1 = 6

[Local event]                                             [Send msg to A]
counter = 3                                               counter = 7
                                                          msg: "Ack", ts=7 ─────┐
[Receive msg from C] <────────────────────────────────────────────────────────┘
counter = max(3, 7) + 1 = 8
```

**Timeline with Lamport Timestamps**:

```
Process A:  [1: Start]  [2: Send→B]  ..................  [8: Recv←C]
Process B:  [1: Start]  .......  [3: Recv←A]  [4: Local]  [5: Send→C]
Process C:  [1: Start]  ..................  [6: Recv←B]  [7: Send→A]
```

**Properties**:

1. **Causality**: If event A happened before event B (causally), then timestamp(A) < timestamp(B)
2. **But NOT vice versa**: If timestamp(A) < timestamp(B), we CANNOT conclude that A happened before B
   - They might be concurrent (happened on different processes with no causal link)

**Example of limitation**:

```
Process A: [Event X, ts=5]
Process B: [Event Y, ts=7]

Question: Did X happen before Y?
Answer: We don't know! They might be concurrent.

ts(X) < ts(Y) doesn't prove causality.
```

**Total Ordering with Tie-Breaking**:

To create a total order (every event can be compared), use process ID as tie-breaker:

```
Comparison:
  IF ts_a < ts_b: Event A before Event B
  IF ts_a > ts_b: Event B before Event A
  IF ts_a == ts_b: Use process_id
    IF pid_a < pid_b: Event A before Event B
    ELSE: Event B before Event A

Example:
  Event (ts=5, pid=A) < Event (ts=5, pid=B)
  Event (ts=5, pid=B) < Event (ts=7, pid=A)
```

### Happens-Before Relation

The **happens-before relation** (denoted as →) is a partial ordering of events that captures causality.

**Definition**: Event A → Event B (A happens before B) if:

1. **Same process**: A and B occur on the same process, and A occurs before B in that process's local timeline
2. **Message send/receive**: A is the sending of a message, and B is the receipt of that message
3. **Transitivity**: If A → B and B → C, then A → C

**Visual representation**:

```
Process P1:    A ────────────> B ────────────> C
                               │
                               │ message
                               ├───────────┐
Process P2:    D               ↓           │
                               E ──────> F │
                                           │
                                           │ message
                                           ↓
Process P3:    G ────────────> H ────────> I


Happens-before relationships:
  A → B (same process)
  B → C (same process)
  B → E (message send/receive)
  E → F (same process)
  F → I (message send/receive)
  H → I (same process)

  Through transitivity:
  A → C (A → B → C)
  A → E (A → B → E)
  A → F (A → B → E → F)
  A → I (A → B → E → F → I)
  B → F (B → E → F)
  B → I (B → E → F → I)
  etc.

Concurrent events (no happens-before):
  A and D are concurrent (A ∦ D)
  D and E are concurrent (D ∦ E)
  A and G are concurrent
  G and D are concurrent
```

**Concurrent Events**:

Events that are NOT related by happens-before are **concurrent**:

```
Event A and Event B are concurrent if:
  NOT (A → B) AND NOT (B → A)

Notation: A ∦ B (A is concurrent with B)
```

**Why this matters**:

```
Concurrent events might conflict!

Example: Collaborative document editing

Process A (User Alice):  [Delete paragraph 2, ts=5]
Process B (User Bob):    [Edit paragraph 2, ts=7]

If these are concurrent (no causal relationship):
  → Conflict! Bob editing paragraph that Alice deleted
  → System must detect and resolve this conflict
```

### Vector Clocks

**Vector clocks** improve upon Lamport timestamps by capturing the happens-before relation precisely. If V(A) < V(B), then A → B (A happened before B).

**Structure**: Each process maintains a vector (array) of counters, one for each process in the system.

```
For N processes in the system:
  Vector clock = [counter₁, counter₂, ..., counterₙ]

Process i maintains:
  V[i] = its own event counter
  V[j] = its knowledge of process j's event counter
```

**Operations**:

```
Initialize:
  V = [0, 0, 0, ..., 0]  (all zeros)

On local event at process i:
  1. Increment V[i]
  2. Assign V to event

On sending message from process i:
  1. Increment V[i]
  2. Include entire vector V in message

On receiving message at process j with vector M:
  1. For each k: V[k] = max(V[k], M[k])
  2. Increment V[j]
  3. Assign V to receive event
```

**Example with 3 Processes**:

```
Process A (index 0)      Process B (index 1)      Process C (index 2)

V_A = [0,0,0]            V_B = [0,0,0]            V_C = [0,0,0]

[Event: e1]
V_A = [1,0,0]
                         [Event: e2]
                         V_B = [0,1,0]

[Send to B]                                       [Event: e3]
V_A = [2,0,0]                                     V_C = [0,0,1]
msg: [2,0,0] ──────────> [Receive from A]
                         V_B = [max(0,2), max(1,0), max(0,0)]
                         V_B = [2,1,0]
                         V_B = [2,2,0] (increment own)

                         [Send to C]
                         V_B = [2,3,0]
                         msg: [2,3,0] ──────────> [Receive from B]
                                                   V_C = [max(0,2), max(0,3), max(1,0)]
                                                   V_C = [2,3,1]
                                                   V_C = [2,3,2] (increment own)

[Event: e4]
V_A = [3,0,0]

[Send to C]
V_A = [4,0,0]
msg: [4,0,0] ─────────────────────────────────> [Receive from A]
                                                 V_C = [max(2,4), max(3,0), max(2,0)]
                                                 V_C = [4,3,2]
                                                 V_C = [4,3,3] (increment own)
```

**Comparing Vector Clocks**:

```
Given two vector clocks V and W:

V < W (V happened before W) if:
  • For all i: V[i] ≤ W[i]  (V is less than or equal in all positions)
  • AND there exists j: V[j] < W[j]  (V is strictly less in at least one position)

V ∦ W (V and W are concurrent) if:
  • NOT (V < W) AND NOT (W < V)
  • Meaning: Some positions V > W, some positions W > V

Examples:

[2,1,0] < [2,2,1]  → TRUE (happened before)
  Position 0: 2 ≤ 2 ✓
  Position 1: 1 ≤ 2 ✓
  Position 2: 0 ≤ 1 ✓
  Position 1: 1 < 2 ✓ (strictly less in at least one)

[3,1,0] ∦ [2,2,0]  → TRUE (concurrent)
  Position 0: 3 > 2 (first is greater here)
  Position 1: 1 < 2 (second is greater here)
  → Neither is less than the other → Concurrent

[4,3,2] < [4,3,3]  → TRUE
  Positions 0,1 equal, position 2 strictly less
```

**Detecting Conflicts**:

```
Example: Distributed database with replicas

Replica A writes: key="user123", value="Alice", V_A=[5,2,1]
Replica B writes: key="user123", value="Bob",   V_B=[4,3,1]

Compare vectors:
  Position 0: 5 > 4 (A greater)
  Position 1: 2 < 3 (B greater)

  Result: Concurrent writes! Conflict detected.

  System must resolve:
    • Last-write-wins (using tie-breaker)
    • Keep both versions (show conflict to user)
    • Application-specific merge
```

**Advantages of Vector Clocks**:

```
✓ Precisely captures causality
✓ Can detect concurrent events
✓ Enables conflict detection
✓ Foundation for causal consistency
```

**Disadvantages**:

```
✗ Size grows with number of processes (N integers)
✗ Not suitable for systems with thousands of nodes
✗ Requires knowing all processes upfront
```

### Version Vectors

**Version vectors** are a practical variant of vector clocks used in distributed databases like Riak, Voldemort, and Dynamo.

**Key difference from vector clocks**:

```
Vector Clocks:
  • Track events in a distributed computation
  • One entry per process

Version Vectors:
  • Track versions of data objects
  • One entry per replica that has modified the object
  • Sparse representation (only modified replicas appear)
```

**Structure**:

```
Version vector for an object:
  [
    (replica_A, counter),
    (replica_B, counter),
    (replica_C, counter)
  ]

Only replicas that have written to this object appear.
```

**Example: Distributed Key-Value Store**:

```
Object: user_profile (key="user:123")

Initial state:
  Version: []  (empty, never written)

Write at Replica A:
  user_profile.name = "Alice"
  Version: [(A, 1)]

Replicate to Replica B:
  Replica B receives version [(A, 1)]

Write at Replica B:
  user_profile.email = "alice@example.com"
  Version: [(A, 1), (B, 1)]

Replicate to Replica C:
  Replica C receives version [(A, 1), (B, 1)]

Write at Replica C:
  user_profile.status = "active"
  Version: [(A, 1), (B, 1), (C, 1)]

Concurrent write at Replica A (before seeing C's update):
  user_profile.name = "Alice Smith"
  Version: [(A, 2), (B, 1)]  ← Doesn't include C's update

Conflict detection:
  Version from C: [(A, 1), (B, 1), (C, 1)]
  Version from A: [(A, 2), (B, 1)]

  Compare:
    A: 2 > 1 (A's version is newer)
    B: 1 = 1 (same)
    C: missing vs 1 (A doesn't know about C's update)

  Result: Concurrent versions! → Conflict
```

**Comparison Algorithm**:

```
Compare version vectors V1 and V2:

Function Compare(V1, V2):
  all_replicas = union of replicas in V1 and V2

  v1_greater = false
  v2_greater = false

  FOR each replica in all_replicas:
    counter1 = V1[replica] OR 0
    counter2 = V2[replica] OR 0

    IF counter1 > counter2:
      v1_greater = true
    IF counter2 > counter1:
      v2_greater = true

  IF v1_greater AND NOT v2_greater:
    RETURN "V1 > V2" (V1 descends from V2)

  IF v2_greater AND NOT v1_greater:
    RETURN "V2 > V1" (V2 descends from V1)

  IF NOT v1_greater AND NOT v2_greater:
    RETURN "V1 = V2" (identical)

  IF v1_greater AND v2_greater:
    RETURN "V1 ∦ V2" (concurrent, conflict)
```

**Conflict Resolution**:

```
Strategy 1: Last-Write-Wins (LWW)
  Use timestamp or replica ID as tie-breaker
  Simple but loses data

  [(A, 2), (B, 1)] vs [(A, 1), (B, 1), (C, 1)]
  → Pick replica A's version (higher counter for A)

Strategy 2: Multi-Value
  Keep all concurrent versions
  Return all to client for resolution

  Client sees:
    Version 1: {name: "Alice Smith"}  [(A, 2), (B, 1)]
    Version 2: {status: "active"}     [(A, 1), (B, 1), (C, 1)]

  Client merges:
    Final: {name: "Alice Smith", status: "active"}
    Version: [(A, 2), (B, 1), (C, 1)]

Strategy 3: Semantic Merge
  Application-specific merge logic

  Shopping cart example:
    Version 1: cart = [item_A, item_B]
    Version 2: cart = [item_A, item_C]
    Merge: cart = [item_A, item_B, item_C]  (union)
```

**Version Vector Pruning**:

To prevent unbounded growth, use **dotted version vectors** or **pruning strategies**:

```
Problem:
  After 1000 replicas write, version vector has 1000 entries

Solution 1: Dotted Version Vectors
  Separate "context" (what we've seen) from "dots" (who wrote)

  Context: [(A, 5), (B, 3), (C, 2)]  (max seen from each)
  Dot: (D, 1)  (who wrote this version)

  More compact representation

Solution 2: Pruning
  Remove old/inactive replicas from vector
  Trade-off: Might not detect all conflicts
```

### Interval Tree Clocks

**Interval Tree Clocks (ITC)** are a generalization of vector clocks that work well with dynamic systems where processes can join, leave, fork, and merge.

**The Problem with Vector Clocks in Dynamic Systems**:

```
Vector clocks assume:
  • Fixed number of processes
  • Each process has a fixed position in the vector

But in reality:
  • Processes crash and new ones start
  • Processes fork (one becomes two)
  • Processes merge (two become one)
  • Don't know total number of processes upfront
```

**ITC Solution**:

ITC represents time using three components:
1. **ID**: A process's identity (can be split and merged)
2. **Event counter**: Logical time
3. **Interval tree structure**: Enables dynamic splitting/merging

**Structure**:

```
ITC Stamp = (id, event)

Where:
  id: Represents a fraction of the total "space" [0, 1]
  event: Event counter (similar to Lamport timestamp)

Example ITC stamps:
  (1, 5)     → Process owns entire space, at event 5
  (1/2, 3)   → Process owns half the space, at event 3
  (1/4, 7)   → Process owns quarter of space, at event 7
```

**Operations**:

```
Fork (create new process):
  Original: (id, e)
  Split id into two parts
  Child 1: (id/2, e)
  Child 2: (id/2, e)

  Example:
    Parent (1, 5) forks into:
      Child A: (1/2, 5)
      Child B: (1/2, 5)

Event (local operation):
  Increment event counter
  (id, e) → (id, e+1)

Join (merge two processes):
  Process A: (id_a, e_a)
  Process B: (id_b, e_b)
  Merged: (id_a + id_b, max(e_a, e_b))

  Example:
    A: (1/4, 10)
    B: (1/4, 8)
    Merged: (1/2, 10)
```

**Visual Example**:

```
Initial state:
  Process P: (1, 0)  [owns entire space, event 0]

P forks:
  Process A: (1/2, 0)  [owns left half]
  Process B: (1/2, 0)  [owns right half]

A does event:
  Process A: (1/2, 1)

B does event:
  Process B: (1/2, 1)

B forks:
  Process B1: (1/4, 1)
  Process B2: (1/4, 1)

B1 does event:
  Process B1: (1/4, 2)

A and B1 communicate:
  A: (1/2, 1)
  B1: (1/4, 2)
  A after receive: (1/2, 2)  [event = max(1, 2) = 2]

Space allocation:
  ┌─────────────────────────────────┐
  │         A (1/2)                 │  B1 (1/4) │  B2 (1/4) │
  └─────────────────────────────────┘
  0                                0.5         0.75        1
```

**Advantages**:

```
✓ Processes can fork and join dynamically
✓ No need to know total number of processes
✓ Bounded space (doesn't grow with number of forks)
✓ Maintains causality tracking like vector clocks
✓ Works in peer-to-peer systems
```

**Disadvantages**:

```
✗ More complex than Lamport or vector clocks
✗ Requires careful implementation
✗ Less intuitive
✗ Not as widely used or understood
```

**Use Cases**:

```
• Peer-to-peer systems (processes come and go)
• Mobile/edge computing (devices go offline)
• Version control systems
• Optimistic replication
• Systems with ephemeral processes
```

---

## Hybrid Logical Clocks (HLC)

### Overview

**Hybrid Logical Clocks** combine the best of both worlds: physical time (wall-clock time) and logical clocks. They provide timestamps that are both human-readable AND capture causal relationships.

**The problem HLC solves**:

```
Logical Clocks (Lamport, Vector):
  ✓ Capture causality perfectly
  ✗ Timestamps are meaningless to humans (what does "event 5284" mean?)
  ✗ Can't correlate with external events (logs, monitoring)

Physical Clocks (NTP):
  ✓ Human-readable (2026-02-06 14:30:00)
  ✓ Can correlate with external systems
  ✗ Don't capture causality
  ✗ Clock skew causes incorrect ordering

Hybrid Logical Clocks:
  ✓ Capture causality (like logical clocks)
  ✓ Close to physical time (like wall clocks)
  ✓ Bounded drift from physical time
```

**Layman explanation**: HLC is like having a watch that mostly shows real time, but occasionally "adjusts itself forward" to ensure that if you saw Event A before Event B on your screen, the timestamp of A will always be less than B, even if your watch was slow.

### Combining Physical and Logical Time

**HLC Structure**:

```
HLC Timestamp = (physical_time, logical_counter)

Components:
  physical_time (pt):
    • Actual wall-clock time from the system
    • Synchronized via NTP or similar
    • Main component of the timestamp

  logical_counter (lc):
    • Incremented when physical time doesn't advance enough
    • Ensures causality when events happen "too fast"
    • Small integer (usually stays close to 0)

Example timestamps:
  (1707223800000, 0)   → Physical time: Feb 6 2026 14:30:00, no logical increment
  (1707223800000, 1)   → Same millisecond, logical counter = 1
  (1707223800001, 0)   → Physical time advanced, counter resets
```

**Algorithm**:

```
Initialize:
  pt = 0  (physical time)
  lc = 0  (logical counter)

On local event:
  current_physical_time = wall_clock_time()

  IF current_physical_time > pt:
    pt = current_physical_time
    lc = 0
  ELSE:
    lc = lc + 1

  timestamp = (pt, lc)

On send message:
  Perform local event (increments HLC)
  Include (pt, lc) in message

On receive message with (pt_msg, lc_msg):
  current_physical_time = wall_clock_time()

  pt' = max(pt, pt_msg, current_physical_time)

  IF pt' == pt AND pt' == pt_msg:
    lc' = max(lc, lc_msg) + 1
  ELSE IF pt' == pt:
    lc' = lc + 1
  ELSE IF pt' == pt_msg:
    lc' = lc_msg + 1
  ELSE:
    lc' = 0

  pt = pt'
  lc = lc'
```

**Example Execution**:

```
Process A                         Process B
Physical clock: 1000              Physical clock: 1005 (5ms ahead)

[Event e1]
wall_clock = 1000
pt_A = 1000, lc_A = 0
Timestamp: (1000, 0)

[Event e2] (1ms later)
wall_clock = 1001
pt_A = 1001, lc_A = 0
Timestamp: (1001, 0)

[Send msg to B]
pt_A = 1002, lc_A = 0
msg: (1002, 0) ─────────────────> [Receive msg]
                                  wall_clock = 1007
                                  pt_B = max(1005, 1002, 1007) = 1007
                                  lc_B = 0 (pt' > all others)
                                  Timestamp: (1007, 0)

                                  [Event e3] (immediately after)
                                  wall_clock = 1007 (same ms!)
                                  pt_B = 1007 (no change)
                                  lc_B = 1 (increment logical)
                                  Timestamp: (1007, 1)

                                  [Event e4] (immediately after)
                                  wall_clock = 1007 (still same ms!)
                                  pt_B = 1007
                                  lc_B = 2
                                  Timestamp: (1007, 2)

                                  [Send msg to A]
                                  pt_B = 1007, lc_B = 3
                                  msg: (1007, 3) ───────────┐
                                                             │
[Receive msg] <──────────────────────────────────────────────┘
wall_clock = 1005
pt_A = max(1002, 1007, 1005) = 1007
lc_A = 3 + 1 = 4 (pt' == pt_msg)
Timestamp: (1007, 4)
```

**Key Properties**:

```
1. Monotonicity:
   Timestamps always increase (never go backward)

2. Causality:
   If e → f (e happens before f), then hlc(e) < hlc(f)

3. Close to Physical Time:
   HLC timestamps stay close to actual wall-clock time
   Bounded drift (more on this below)
```

### Bounded Clock Skew

One of HLC's most important properties is that it stays **bounded** with respect to physical time, unlike pure logical clocks which can drift arbitrarily far from real time.

**The Bound**:

```
At any time t, HLC timestamp satisfies:
  physical_time(t) ≤ hlc.pt ≤ physical_time(t) + ε

Where ε is the maximum clock skew between any two processes.

In practice:
  • If NTP keeps clocks within ±100ms
  • Then HLC timestamps are within 100ms of real time
```

**Why this matters**:

```
Scenario: Debugging a system outage

With Logical Clocks:
  Event 1: (counter: 52847)
  Event 2: (counter: 52848)
  Event 3: (counter: 52849)

  Question: When did the outage start?
  Answer: No idea! Just that event 52847 came before 52848.

With HLC:
  Event 1: (1707223800000, 0) = Feb 6 2026 14:30:00.000
  Event 2: (1707223800015, 0) = Feb 6 2026 14:30:00.015
  Event 3: (1707223800023, 1) = Feb 6 2026 14:30:00.023

  Question: When did the outage start?
  Answer: Around 14:30:00 on Feb 6, 2026!
  Can correlate with external logs, monitoring, user reports.
```

**Bounding the Logical Counter**:

The logical counter typically stays very small:

```
Conditions for logical counter to increment:
  • Multiple events in same millisecond
  • Receiving message from future (clock skew)

Typical values:
  • lc = 0: ~99% of timestamps
  • lc = 1-10: ~0.9% of timestamps
  • lc > 10: Very rare (indicates many events in same ms or clock issues)

If lc grows large (e.g., > 1000):
  → Indicator of problems:
     • Clock not advancing (frozen clock)
     • Extremely high event rate
     • Major clock skew between processes
```

**Comparison with Physical Time**:

```
Compare two HLC timestamps:

(pt1, lc1) < (pt2, lc2) if:
  pt1 < pt2  OR
  (pt1 == pt2 AND lc1 < lc2)

Example:
  (1000, 5) < (1001, 0)  → TRUE (physical time differs)
  (1000, 5) < (1000, 10) → TRUE (same physical, compare logical)
  (1000, 5) < (999, 100) → FALSE (1000 > 999)

Result:
  Timestamps are mostly ordered by physical time
  Logical counter is rare tie-breaker
```

### NTP Dependency Reduction

While HLC works best with synchronized clocks, it's more **resilient to clock synchronization failures** than pure physical timestamps.

**Traditional Timestamp-Based Systems**:

```
Database with Last-Write-Wins (LWW):

Replica A (clock: 1000ms):
  WRITE key="user", value="Alice", timestamp=1000

Replica B (clock: 900ms, 100ms behind!):
  WRITE key="user", value="Bob", timestamp=900

Merge:
  Compare timestamps: 1000 > 900
  Winner: Alice

But reality:
  Bob's write happened AFTER Alice's!
  Wrong result due to clock skew 💥
```

**With HLC**:

```
Replica A (clock: 1000ms):
  HLC_A = (pt: 1000, lc: 0)
  WRITE key="user", value="Alice", hlc=(1000, 0)

Replica B (clock: 900ms, behind):
  Receives replication of Alice's write: (1000, 0)
  Updates own HLC:
    pt_B = max(900, 1000) = 1000
    lc_B = 1
  HLC_B = (1000, 1)

  WRITE key="user", value="Bob", hlc=(1000, 1)

Merge:
  Alice: (1000, 0)
  Bob:   (1000, 1)
  Winner: Bob (1000, 1) > (1000, 0)
  Correct! Bob's write was causally after ✓
```

**Key Insight**:

```
HLC doesn't require perfect clock sync:
  • Tolerates clock skew up to ε
  • Logical component ensures causality even with skew
  • Physical component stays close to real time

Requirements:
  • Clocks don't go backward (monotonic)
  • Clocks eventually advance (not frozen)
  • Bounded skew (NTP running, even if imperfect)
```

**Failure Modes**:

```
1. Clock runs backward:
   Problem: Violates HLC assumptions
   Mitigation: Use monotonic clocks (guaranteed non-decreasing)

2. Clock frozen:
   Symptom: Logical counter grows large
   Impact: Timestamps still correct, but debugging harder
   Mitigation: Monitor lc growth, alert if > threshold

3. Excessive skew (> several seconds):
   Problem: HLC timestamps drift far from real time
   Impact: Human correlation becomes harder
   Mitigation: Improve NTP sync, use PTP

4. No NTP at all:
   Impact: HLC degrades to Lamport timestamps
   Physical time component becomes meaningless
   Causality still captured, but not tied to real time
```

### Use in CockroachDB, MongoDB

Several major distributed databases use HLC for transaction ordering and consistency.

**CockroachDB**:

```
Uses HLC for:
  • Transaction timestamps
  • MVCC (Multi-Version Concurrency Control)
  • Causality tracking across distributed transactions
  • Serializable isolation

Key design:
  • Every transaction gets HLC timestamp
  • Read timestamp: HLC at transaction start
  • Write timestamp: HLC at transaction commit
  • Later read sees write if: read_ts > write_ts

Example:
  T1 starts:  hlc = (1000, 0)
  T1 writes:  key="x", value=5, ts=(1000, 0)
  T1 commits: hlc = (1001, 0)

  T2 starts:  hlc = (1001, 2)  (after seeing T1's commit)
  T2 reads:   key="x" at ts=(1001, 2)
  Result:     Sees value=5 (because (1001, 2) > (1001, 0))
```

**Uncertainty Windows in CockroachDB**:

```
Problem: Clock skew creates "uncertainty"

Node A (clock: 1000ms):
  Transaction T1 commits at hlc=(1000, 0)

Node B (clock: 999ms, 1ms behind):
  Transaction T2 starts at hlc=(999, 0)

Question: Should T2 see T1's write?

In real time: T2 might have started before T1 committed
By timestamps: (999, 0) < (1000, 0), so T2 shouldn't see T1

Solution: Uncertainty window
  T2's read timestamp: (999, 0)
  Uncertainty window: [(999, 0), (999 + ε, 0)]

  If T1's commit timestamp falls in uncertainty window:
    → T2 must retry or wait
    → Ensures external consistency
```

**MongoDB (Sharded Clusters)**:

```
Uses HLC (called "cluster time") for:
  • Causal consistency in sharded clusters
  • Read concern "majority" with causality
  • Change streams ordering

Client-driven causality:
  Client sends read to Shard A
  Shard A returns: data + clusterTime=(1000, 5)

  Client sends next read to Shard B
  Client includes: afterClusterTime=(1000, 5)

  Shard B waits until its clusterTime > (1000, 5)
  This ensures Shard B has seen all changes before (1000, 5)

Example:
  1. Client writes to Shard A: user.name = "Alice"
     Returns: clusterTime=(1000, 0)

  2. Client reads from Shard B: find user
     Sends: afterClusterTime=(1000, 0)

  3. Shard B ensures it has replicated up to (1000, 0)
     Then returns user data

  4. Client is guaranteed to see its own write ✓
```

**Advantages in Production Systems**:

```
CockroachDB benefits:
  ✓ Serializable transactions without centralized coordinator
  ✓ Causality across geo-distributed clusters
  ✓ Debugging: Timestamps correlate with logs
  ✓ Reduced NTP dependency vs pure physical timestamps

MongoDB benefits:
  ✓ Causal consistency without excessive coordination
  ✓ Clients can request "read your own writes"
  ✓ Change streams have causally ordered events
  ✓ Works across WAN with clock skew
```

**Implementation Details**:

```
Typical HLC implementation:

1. Use 64-bit integer for physical time
   • Milliseconds since Unix epoch
   • Fits in 64 bits until year 2262

2. Use 32-bit integer for logical counter
   • Enough for billions of events per millisecond
   • In practice, rarely exceeds single digits

3. Total: 96 bits (12 bytes) per timestamp
   • Compact, efficient
   • Comparable to UUID (16 bytes)

4. Monotonic clock source
   • Use CLOCK_MONOTONIC on Linux
   • Prevents backward jumps

5. Periodic NTP sync
   • Sync every 60 seconds typical
   • Keep skew < 100ms
```

---

## Physical Clock Synchronization

Physical clock synchronization aims to keep the clocks on different machines aligned with real time (UTC - Coordinated Universal Time). Unlike logical clocks which only care about ordering, physical clocks provide actual wall-clock time.

**Why synchronize physical clocks**:
- **Distributed debugging**: Correlate logs across machines
- **Timeout enforcement**: Know when leases expire
- **Transaction timestamps**: Determine which write is "newer"
- **Compliance**: Prove when events occurred
- **Monitoring**: Time-series metrics need aligned timestamps

### NTP (Network Time Protocol)

**NTP** is the most widely used protocol for synchronizing computer clocks over a network. It's been around since 1985 and is used by billions of devices.

**How NTP Works**:

```
Client                                  NTP Server
  │                                          │
  │─── Request (timestamp: T1) ─────────────>│
  │                                          │ [Receive at T2]
  │                                          │ [Process]
  │                                          │ [Send at T3]
  │<─── Response (T2, T3) ───────────────────│
  │ [Receive at T4]                          │
  │                                          │

Timestamps collected:
  T1 = Client send time (client's clock)
  T2 = Server receive time (server's clock)
  T3 = Server send time (server's clock)
  T4 = Client receive time (client's clock)

Calculate:
  Round-trip delay: δ = (T4 - T1) - (T3 - T2)
  Clock offset:     θ = ((T2 - T1) + (T3 - T4)) / 2

Interpretation:
  δ = total network delay
  θ = how much client clock is ahead/behind server
```

**Example Calculation**:

```
T1 = 1000 (client sends)
T2 = 1150 (server receives, server's clock is ahead)
T3 = 1151 (server sends)
T4 = 1002 (client receives)

Round-trip delay:
  δ = (T4 - T1) - (T3 - T2)
  δ = (1002 - 1000) - (1151 - 1150)
  δ = 2 - 1 = 1ms

Clock offset:
  θ = ((T2 - T1) + (T3 - T4)) / 2
  θ = ((1150 - 1000) + (1151 - 1002)) / 2
  θ = (150 + 149) / 2
  θ = 149.5ms

Result: Client is ~150ms behind server
Action: Gradually adjust client clock forward
```

**NTP Hierarchy (Stratum)**:

```
Stratum 0: Reference Clocks (atomic clocks, GPS)
             │
             ├─> [Atomic Clock]
             ├─> [GPS Receiver]
             └─> [Radio Clock]

Stratum 1: Primary Time Servers (directly connected to Stratum 0)
             │
             ├─> [time.nist.gov]
             ├─> [time.google.com]
             └─> [time.cloudflare.com]

Stratum 2: Secondary Servers (sync from Stratum 1)
             │
             ├─> [Company NTP Server]
             └─> [ISP NTP Server]

Stratum 3: Sync from Stratum 2
             │
             └─> [Your Servers]

Lower stratum = more accurate
Each hop adds uncertainty
```

**Clock Adjustment Strategies**:

```
Small offset (< 128ms):
  Slew the clock: Gradually speed up/slow down

  Example: Clock 100ms behind
    Instead of jumping forward 100ms (jarring!)
    Run clock 0.01% faster for ~16 minutes
    Eventually catches up smoothly

  Benefit: No time discontinuities
  Drawback: Takes time to converge

Large offset (> 128ms):
  Step the clock: Jump immediately to correct time

  Example: Clock 5 seconds behind
    Immediately set clock forward 5 seconds

  Benefit: Quick correction
  Drawback: Time "goes backward" for some events
             Can break applications!
```

**NTP Accuracy**:

```
Typical accuracy (Internet):
  • LAN: ±1ms
  • WAN (Internet): ±10-100ms
  • Public NTP servers: ±50ms typical
  • Good conditions: ±10ms
  • Poor network: ±500ms

Factors affecting accuracy:
  • Network latency variation (jitter)
  • Asymmetric network paths
  • Server load
  • Distance to time source
  • Number of hops
```

**NTP Best Practices**:

```
1. Use multiple time sources
   • Poll 3-5 NTP servers
   • NTP uses algorithm to select best
   • Protects against single server failure

2. Use nearby/reliable servers
   • Company's own Stratum 2 server
   • Public: time.google.com, time.cloudflare.com
   • Avoid distant or overloaded servers

3. Monitor clock offset
   • Alert if offset > 100ms
   • Investigate if offset growing

4. Use NTP discipline mode
   • Keeps clock continuously adjusted
   • Better than one-time sync

5. Protect against time attacks
   • Use authenticated NTP (NTP with MAC)
   • Firewall NTP traffic
```

### PTP (Precision Time Protocol)

**PTP** (IEEE 1588) provides much higher accuracy than NTP, targeting sub-microsecond synchronization. Used in telecommunications, financial trading, and industrial automation.

**How PTP Differs from NTP**:

```
NTP:
  • Software-based
  • Accuracy: ~1-100ms
  • Works over Internet
  • Simple to deploy

PTP:
  • Hardware-assisted (network switches)
  • Accuracy: <1 microsecond
  • Requires PTP-aware network infrastructure
  • More complex to deploy
```

**PTP Architecture**:

```
         ┌─────────────────┐
         │  Grandmaster    │ ← Most accurate clock (Stratum 0)
         │  (GPS-synced)   │
         └────────┬────────┘
                  │
     ┌────────────┼────────────┐
     │                         │
┌────▼─────┐              ┌────▼─────┐
│ Boundary │              │ Boundary │ ← PTP-aware switches
│  Clock   │              │  Clock   │
└────┬─────┘              └────┬─────┘
     │                         │
┌────▼─────┐              ┌────▼─────┐
│  Slave   │              │  Slave   │ ← End devices
│  Clock   │              │  Clock   │
└──────────┘              └──────────┘
```

**PTP Message Exchange**:

```
Master (Grandmaster)              Slave
  │                                 │
  │─── Sync message (T1) ──────────>│
  │                                 │ [Receive at T2]
  │                                 │
  │─── Follow_Up (T1) ──────────────>│
  │                                 │ [Now knows precise T1]
  │                                 │
  │<─── Delay_Req ──────────────────│ [Send at T3]
  │ [Receive at T4]                 │
  │                                 │
  │─── Delay_Resp (T4) ─────────────>│
  │                                 │

Slave calculates offset and delay (similar to NTP)
But with hardware timestamping for precision!
```

**Hardware Timestamping**:

```
Software timestamping (NTP):
  Packet arrives → OS processes → Timestamp recorded
  Delay: ~1-100ms (unpredictable)
  Uncertainty: High (OS scheduler, interrupts)

Hardware timestamping (PTP):
  Packet arrives → Network card timestamps immediately
  Delay: ~nanoseconds
  Uncertainty: Very low (<1μs)
```

**PTP Accuracy Levels**:

```
Without hardware support:
  • Similar to NTP: ~1-100ms

With hardware-timestamping NICs:
  • ~1-10 microseconds

With PTP boundary clocks (switches):
  • <1 microsecond

With PTP transparent clocks:
  • <100 nanoseconds
```

**Use Cases**:

```
Financial Trading:
  • Regulations require timestamp accuracy
  • Microsecond precision for trade ordering
  • Prove trade sequence in disputes

5G Telecommunications:
  • Base stations need synchronized time
  • Critical for handoffs
  • <1μs required

Industrial Automation:
  • Synchronized robot movements
  • Distributed control systems
  • Real-time coordination

Scientific Instruments:
  • Particle accelerators
  • Radio telescopes
  • Coordinated measurements
```

**Deployment Challenges**:

```
✗ Requires specialized hardware
✗ Network infrastructure must support PTP
✗ More expensive than NTP
✗ Complex configuration
✗ Not needed for most applications

Typical approach:
  • NTP for general servers
  • PTP only where microsecond accuracy needed
```

### Clock Skew and Drift

**Clock drift**: Natural tendency of hardware clocks to run at slightly different speeds.

```
Perfect clock:    Advances 1 second per real second
Real clock:       Advances 1.00001 seconds per real second (100ppm drift)

After 1 hour:
  Perfect clock: 3600 seconds
  Real clock:    3600.36 seconds
  Drift: 360ms

After 1 day:
  Drift: ~8.6 seconds

After 1 week (no sync):
  Drift: ~60 seconds (1 minute!)
```

**Drift Rate (ppm - parts per million)**:

```
Typical hardware:
  • Desktop/laptop:  ±50-200 ppm (±0.00005-0.0002 = ±0.005%-0.02%)
  • Server:          ±20-50 ppm
  • Datacenter:      ±10-20 ppm
  • OCXO (oven):     ±0.01-1 ppm (very expensive)
  • Atomic clock:    ±0.000001 ppm

Example: 100 ppm drift
  1 hour:   0.36 seconds
  1 day:    8.64 seconds
  1 week:   60.48 seconds
  1 month:  259.2 seconds (~4.3 minutes)
```

**Clock skew**: Difference between two clocks at a given instant.

```
At time T:
  Server A clock:  1000.500
  Server B clock:  1000.650
  Clock skew:      150ms (B is ahead of A)

Skew changes over time due to drift!

After 1 hour (with 100ppm drift difference):
  Server A drifted: +36ms
  Server B drifted: +72ms (2x drift rate)
  New skew:         150 + 36 = 186ms
```

**Effects of Clock Skew/Drift**:

```
1. Incorrect Ordering
   Event A on Server 1: timestamp = 1000
   Event B on Server 2: timestamp =  995

   Real-time order: A → B
   Timestamp order: B → A (wrong!)

2. Lease Expiration Disagreement
   Server A (clock: 1000): "Lease expires at 1030"
   Server B (clock: 1100): "Lease already expired!"
   Server A: "I still hold the lease"
   Conflict! Both servers act as primary 💥

3. Duplicate Detection Failures
   Client sends request with timestamp T=1000
   Server A: "New request at 1000"
   Server B (clock behind): "Request at 1000 is in the future! Reject"

4. Performance Monitoring Issues
   Request at Server A: starts at 1000
   Request at Server B: ends at   995
   Calculated duration: -5ms (impossible!)
```

**Mitigations**:

```
1. Regular NTP Synchronization
   • Sync every 60 seconds
   • Keeps skew bounded
   • Typical: ±100ms with NTP

2. Use Logical/Hybrid Clocks
   • HLC for causality + approximate time
   • Vector clocks when exact time not needed

3. Timestamp Tolerance Windows
   • Accept events within ±200ms of expected time
   • Reject obvious future/past timestamps

4. Monotonic Clocks for Duration
   • Use CLOCK_MONOTONIC for measuring intervals
   • Immune to NTP adjustments

5. Centralized Timestamp Authority
   • Single server issues timestamps
   • No skew between timestamps (but bottleneck)
```

### GPS-Based Time Synchronization

**GPS** satellites broadcast precise time signals, providing Stratum 0 time source accessible anywhere on Earth.

**How GPS Time Works**:

```
GPS Satellite Constellation:
  • ~30 satellites orbiting Earth
  • Each carries atomic clocks
  • Broadcast time signals continuously
  • Accuracy: ±14 nanoseconds (incredible!)

GPS Receiver:
  • Receives signals from 4+ satellites
  • Calculates position AND time
  • Outputs: PPS (Pulse Per Second) signal
  • Connects to server via serial/USB/PCIe
```

**GPS Timing Architecture**:

```
          ┌─────────────────┐
          │  GPS Satellites │
          │  (Atomic Clocks)│
          └────────┬────────┘
                   │ Radio signals
                   │
          ┌────────▼────────┐
          │  GPS Receiver   │
          │  (Antenna +     │
          │   Processing)   │
          └────────┬────────┘
                   │ PPS signal (1 pulse/second)
                   │
          ┌────────▼────────┐
          │  Time Server    │
          │  (Stratum 1)    │
          │  Runs NTP/PTP   │
          └────────┬────────┘
                   │ NTP/PTP
     ┌─────────────┼─────────────┐
     │             │             │
┌────▼────┐   ┌────▼────┐   ┌────▼────┐
│Server A │   │Server B │   │Server C │
│(Stratum │   │(Stratum │   │(Stratum │
│   2)    │   │   2)    │   │   2)    │
└─────────┘   └─────────┘   └─────────┘
```

**PPS (Pulse Per Second)**:

```
GPS receiver outputs electrical pulse exactly once per second:

Time: ────┬────────┬────────┬────────┬────────
          │        │        │        │
          │ 1sec   │ 1sec   │ 1sec   │
          ↓        ↓        ↓        ↓
       T=0      T=1      T=2      T=3

Each pulse edge marks a precise second boundary
Server synchronizes its clock to these pulses
Accuracy: <100 nanoseconds to GPS time
```

**GPS vs NTP Comparison**:

```
GPS-based:
  ✓ Extremely accurate (nanoseconds)
  ✓ Not dependent on network
  ✓ Stratum 1 (direct reference)
  ✓ Works anywhere with GPS signal
  ✗ Requires GPS antenna/receiver
  ✗ Hardware cost ($100-$10,000)
  ✗ Doesn't work indoors (needs sky view)
  ✗ Vulnerable to GPS jamming

NTP over Internet:
  ✓ No special hardware
  ✓ Works anywhere with network
  ✓ Free public servers
  ✓ Works indoors
  ✗ Less accurate (milliseconds)
  ✗ Depends on network quality
  ✗ Vulnerable to network issues
```

**Use Cases for GPS Time**:

```
Financial Industry:
  • Trading platforms require precise timestamps
  • Regulations (MiFID II) mandate GPS-synchronized time
  • Prove trade ordering

Telecommunications:
  • Cell towers need synchronized time
  • Critical for handoffs and coordination
  • 4G/5G networks

Power Grid:
  • Synchronized phasor measurements
  • Detect faults and anomalies
  • Grid stability monitoring

Scientific Research:
  • Radio telescope arrays
  • Particle physics experiments
  • Precise event correlation

Critical Infrastructure:
  • Emergency services (911)
  • Air traffic control
  • Military systems
```

**Redundancy and Backup**:

```
Best practice: Multiple time sources

Primary: GPS receiver
  ├─> If GPS lost (jamming, antenna failure):
  │
  ├─> Fallback 1: Disciplined oscillator
  │   • High-quality local clock
  │   • Trained by GPS over weeks
  │   • Holdover: maintains accuracy for hours/days
  │
  └─> Fallback 2: NTP servers
      • Network-based time
      • Allows operation if GPS offline

Example:
  Normal: GPS provides ±100ns accuracy
  GPS lost: Oscillator provides ±10μs for 24 hours
  Oscillator fails: NTP provides ±10ms
```

**Modern Alternatives**:

```
GPS is US-controlled, so other systems exist:

• GLONASS: Russian satellite navigation
• Galileo: European Union system
• BeiDou: Chinese system

Multi-GNSS receivers:
  • Receive signals from multiple systems
  • Better reliability
  • More satellites visible
  • Redundancy against regional issues
```

---

## Google TrueTime

**TrueTime** is Google's globally-distributed clock system that provides an API with explicit **uncertainty bounds**. Instead of giving a single timestamp, TrueTime tells you: "The current time is somewhere in this interval."

**The Innovation**:

```
Traditional timestamp:
  clock.now() → returns 1707223800000 (claims precision it doesn't have)

TrueTime:
  TT.now() → returns [1707223800000, 1707223800007]
              "The true time is somewhere in this 7ms interval"
```

**Layman explanation**: Instead of your watch saying "It's exactly 3:00:00 PM" (which is probably wrong), TrueTime is like a watch that honestly says "It's between 2:59:59.995 PM and 3:00:00.005 PM." By admitting the uncertainty, the system can make better decisions.

### Confidence Intervals

TrueTime provides two API calls that return time intervals:

```
TT.now() → Returns an interval [earliest, latest]

  earliest: Earliest possible current time
  latest:   Latest possible current time

The true time is GUARANTEED to be somewhere in this interval.

TT.after(t) → Returns true if t has definitely passed
  Returns: t < earliest (t is before the earliest possible current time)

TT.before(t) → Returns true if t has definitely not occurred yet
  Returns: t > latest (t is after the latest possible current time)
```

**Example**:

```
TT.now() = [1000, 1007]

TT.after(990)  = true   (990 < 1000, definitely passed)
TT.after(995)  = true   (995 < 1000, definitely passed)
TT.after(1003) = false  (1003 might be current time, uncertain)
TT.after(1010) = false  (1010 > 1007, in the future)

TT.before(990)  = false (990 < 1000, already passed)
TT.before(1003) = false (1003 might be current time, uncertain)
TT.before(1010) = true  (1010 > 1007, definitely future)
```

**Epsilon (ε): The Uncertainty Bound**:

```
If TT.now() = [earliest, latest], then:
  ε = (latest - earliest) / 2

This is the maximum clock uncertainty.

Example:
  TT.now() = [1000, 1014]
  ε = 7ms

Google's TrueTime:
  ε typically ~7ms (average)
  ε can spike to ~10ms under load
  ε kept below 10ms 99.9% of the time
```

**How TrueTime Achieves Low Uncertainty**:

```
Hardware Infrastructure:

Each datacenter has:
  1. GPS receivers
     • Atomic clock reference
     • ±100 nanoseconds accuracy
     • But can fail (jamming, weather)

  2. Atomic clocks
     • Rubidium or Cesium
     • Drift: ~1 microsecond/day
     • Backup when GPS unavailable

Distribution:
  ├─ GPS master (primary)
  ├─ Atomic clock master (backup)
  ├─ Time servers in each datacenter (hundreds)
  └─ Machines sync from local time servers

Redundancy:
  • Multiple GPS antennas
  • Multiple atomic clocks
  • Multiple time servers
  • Cross-datacenter validation
```

**Time Synchronization Process**:

```
Every machine:
  1. Polls multiple time servers (7-10) every 30 seconds
  2. Uses Marzullo's algorithm to determine consensus time
  3. Rejects outliers
  4. Calculates uncertainty based on:
     • Network round-trip time
     • Time since last sync
     • Known clock drift rate
  5. Adjusts local clock + uncertainty bound

Uncertainty grows between syncs:

Sync at T=0:     ε = 1ms
After 10 sec:    ε = 3ms (drift accumulated)
After 20 sec:    ε = 5ms
After 30 sec:    ε = 7ms
[Sync again]     ε = 1ms (reset)
```

**Failure Handling**:

```
GPS failure:
  • Switch to atomic clocks
  • ε increases slightly (~2-3ms)
  • Still usable

Time server failure:
  • Use redundant servers
  • Increase polling frequency
  • ε might increase

Network partition:
  • Can't sync with time servers
  • ε grows unbounded
  • Eventually: Refuse to serve TT.now()
  • System halts rather than risk inconsistency ✓
```

### Spanner's Use of TrueTime

**Google Spanner** is a globally-distributed SQL database that uses TrueTime to provide **external consistency** (linearizability) without requiring a global lock or coordinator.

**The Challenge**:

```
Problem: Distributed transactions across datacenters

Datacenter US-East:  Transaction T1 commits
Datacenter Europe:   Transaction T2 commits

Question: Which transaction came first?

Without TrueTime:
  Rely on clock timestamps → Clock skew causes wrong ordering
  Use distributed locking → Slow, requires coordination

With TrueTime:
  Use confidence intervals → Guarantee correct ordering
```

**How Spanner Uses TrueTime**:

```
Transaction Execution:

1. Transaction starts
   • Gets start timestamp: TT.now().latest
   • Reads data as of this timestamp

2. Transaction executes (reads, writes)

3. Transaction ready to commit
   • Gets commit timestamp: s_commit
   • s_commit > all previous operation timestamps

4. Wait for commit timestamp to be definitely in past
   • Wait until: TT.after(s_commit) == true
   • Ensures all observers agree commit is in past

5. Commit and replicate
```

**Visual Example**:

```
Transaction T1 in US-East:

T=1000  [Start transaction]
        start_ts = TT.now().latest = 1005

T=1020  [Read data]
        Read as of timestamp 1005

T=1050  [Write data]
        Generate commit timestamp: s_commit = 1055

T=1055  [Ready to commit]
        TT.now() = [1053, 1060]
        TT.after(1055) = false (1055 > 1053, might be current)

T=1060  [Wait...]
        TT.now() = [1058, 1065]
        TT.after(1055) = true (1055 < 1058, definitely past!)

T=1061  [Commit!]
        Commit timestamp = 1055
        Replicate to other datacenters
```

**Why the Wait Matters**:

```
Without waiting:

T1 commits at s_commit=1055
TT.now() = [1053, 1060]  (s_commit is uncertain)

Client sees commit
Client starts T2 in different datacenter
That datacenter's TT.now() = [1050, 1057]  (clock skew)

T2 gets timestamp 1057
But T2 started AFTER T1 committed (causally)
Yet 1057 > 1055 by only 2ms, might be reordered!

With waiting:

T1 commits at s_commit=1055
WAIT until TT.after(1055) = true
Now all datacenters agree 1055 is in the past

T2 in any datacenter gets timestamp > 1058
Guaranteed: T2 timestamp > T1 timestamp ✓
Causality preserved!
```

### Wait-Out Uncertainty

The **commit wait** is the time Spanner waits after choosing a commit timestamp before actually committing. This ensures external consistency.

**Commit Wait Duration**:

```
Wait time = ε (the uncertainty bound)

If ε = 7ms (typical):
  • Commit wait = ~7ms
  • Every write operation delays 7ms

This is the price of correctness!
```

**Why Exactly ε**:

```
When transaction ready to commit at real time T_real:

Local TT.now() = [T_real - ε, T_real + ε]
Choose s_commit = T_real + ε (latest)

Wait for ε time:
After ε time, real time = T_real + ε

Now: TT.now() = [T_real, T_real + 2ε]
TT.after(s_commit) = TT.after(T_real + ε)
                   = (T_real + ε < T_real)
                   = false... wait more

After exactly ε:
Real time = T_real + ε
TT.now().earliest = T_real + ε - ε = T_real
But we chose s_commit at T_real
So TT.after(T_real) = true!

Mathematical bound:
  Wait time ≥ ε ensures commit timestamp in past
```

**Performance Impact**:

```
Read-only transactions:
  • No commit wait needed!
  • Can execute at any timestamp in the past
  • Very fast

Read-write transactions:
  • Must wait ~7ms per commit
  • Becomes bottleneck for high-throughput writes

Google's optimizations:
  1. Batch multiple commits together (amortize wait)
  2. Use read-only transactions when possible
  3. Pipeline operations (overlap compute and wait)
  4. Keep ε as small as possible (better hardware)
```

**Comparison with Other Systems**:

```
Traditional 2PC:
  • Prepare phase: ~10-50ms (network RTT)
  • Commit phase: ~10-50ms
  • Total: 20-100ms

Spanner:
  • Commit wait: ~7ms
  • Replication: ~10-50ms
  • Total: ~17-57ms
  • BUT: Provides external consistency globally!

Cassandra (eventual consistency):
  • Write: ~1-5ms
  • No coordination needed
  • BUT: No consistency guarantees
```

### External Consistency

**External consistency** (also called linearizability) means: If transaction T1 commits before transaction T2 starts (in real time), then T1's commit timestamp < T2's commit timestamp.

**Layman explanation**: If you submit a bank transfer and THEN your friend checks the balance, they MUST see the transferred amount. The system's timestamps must respect the actual order in which things happened in the real world.

**Formal Definition**:

```
Let:
  T1 commit at real time t1
  T2 start at real time t2

If t1 < t2 (T1 finishes before T2 starts):
  Then: commit_timestamp(T1) < commit_timestamp(T2)
```

**Example**:

```
Real Timeline:

T=1000  User A: Transfer $100 (T1 starts)
T=1010  T1 commits
T=1015  User B: Check balance (T2 starts)
T=1020  T2 reads balance

External consistency requires:
  T2 must see T1's transfer
  Because T2 started after T1 committed (real-time)

Implementation:
  T1 commit timestamp: s1 = 1010
  Wait until TT.after(s1) = true
  At T=1017, TT.after(1010) = true, commit

  T2 start timestamp: s2 = TT.now().latest = 1020
  Read at timestamp 1020
  Since 1020 > 1010, sees T1's changes ✓
```

**Why External Consistency is Hard**:

```
Challenge: Different datacenters, different clocks

Datacenter A (clock slightly ahead):
  Real time: 1000
  Clock: 1010
  T1 commits with timestamp 1010

Datacenter B (clock slightly behind):
  Real time: 1005 (5ms after A)
  Clock: 1000
  T2 starts with timestamp 1000

Result without TrueTime:
  T2 timestamp (1000) < T1 timestamp (1010)
  But T2 started AFTER T1 in real time!
  Violates external consistency 💥

With TrueTime:
  Datacenter A:
    TT.now() = [1005, 1015]
    T1 commits at 1015 (latest)
    Wait until TT.after(1015) = true

  Datacenter B:
    Real time has advanced during wait
    TT.now() = [1012, 1022]
    T2 starts at 1022 (latest)
    1022 > 1015 ✓
```

**Applications Beyond Spanner**:

```
External consistency enables:

1. Distributed Snapshots
   • Read entire database as of timestamp T
   • Consistent view across all data
   • No locks needed

2. Time-Travel Queries
   • "Show me the database as of yesterday 3pm"
   • Useful for debugging, auditing

3. Change Data Capture
   • Capture all changes in timestamp order
   • Guaranteed causal ordering

4. Multi-Region Active-Active
   • Write to any region
   • Reads see causally consistent data
   • No master-slave lag issues
```

**The Cost**:

```
Benefits:
  ✓ External consistency (linearizability)
  ✓ No distributed locking for transactions
  ✓ Read-only transactions are fast
  ✓ Scalable across globe

Costs:
  ✗ Requires TrueTime infrastructure ($$$)
  ✗ Every write waits ~7ms (commit wait)
  ✗ Complex implementation
  ✗ Only Google has full implementation

Alternatives for others:
  • CockroachDB: Uses HLC + read/write timestamps (similar ideas)
  • YugabyteDB: Hybrid logical clocks
  • FaunaDB: Similar timestamp-based consistency
  • Cloud Spanner: Google offers as a service!
```

---

## Timestamp Ordering

**Timestamp ordering** uses timestamps to determine the order in which operations should be executed or how conflicts should be resolved.

### Total Ordering with Timestamps

A **total order** means every pair of events can be compared: for any events A and B, either A < B, A > B, or A = B.

**Why Total Ordering Matters**:

```
Distributed system with replicas:

Replica 1 receives: Write(x, 5)
Replica 2 receives: Write(x, 10)
Replica 3 receives: Write(x, 7)

Question: What is the final value of x?
  Without total ordering: Each replica might choose differently!
  With total ordering: All replicas agree on the same final value
```

**Creating Total Order from Timestamps**:

```
Method 1: Physical Timestamps + Tie-Breaker

Order events by:
  1. Timestamp (lower is earlier)
  2. If timestamps equal, use process ID
  3. If still equal, use sequence number

Example:
  Event A: (timestamp=1000, pid=node-1, seq=5)
  Event B: (timestamp=1000, pid=node-1, seq=6)
  Event C: (timestamp=1000, pid=node-2, seq=1)
  Event D: (timestamp=1001, pid=node-1, seq=1)

  Total order: A < B < C < D

  A vs B: Same timestamp and pid, compare seq (5 < 6)
  B vs C: Same timestamp, compare pid (node-1 < node-2)
  C vs D: Different timestamp (1000 < 1001)
```

**Method 2: Lamport Timestamps**:

```
Inherently provides total order:
  • Events have monotonically increasing counters
  • Process ID breaks ties

Example:
  (counter=5, pid=A) < (counter=5, pid=B) < (counter=6, pid=A)
```

**Method 3: Hybrid Logical Clocks**:

```
Order by:
  1. Physical time component
  2. Logical counter (for events in same millisecond)
  3. Process ID (if both above are equal)

Example:
  (pt=1000, lc=0, pid=A) < (pt=1000, lc=1, pid=A) < (pt=1001, lc=0, pid=B)
```

**Application: Last-Write-Wins (LWW)**:

```
Conflict resolution using total ordering:

Two concurrent writes to key "user":
  Write 1: (ts=1000, pid=A) → value="Alice"
  Write 2: (ts=1005, pid=B) → value="Bob"

Compare timestamps:
  1005 > 1000
  Winner: Write 2
  Final value: "Bob"

Edge case - simultaneous:
  Write 1: (ts=1000, pid=A) → value="Alice"
  Write 2: (ts=1000, pid=B) → value="Bob"

Compare process IDs:
  A < B (lexicographic)
  Winner: Write 2 (higher pid)
  Final value: "Bob"
```

### Tie-Breaking Strategies

When timestamps are equal, we need deterministic tie-breaking to ensure all replicas make the same decision.

**Strategy 1: Process/Node ID**:

```
Every node has unique identifier:
  • node-1, node-2, node-3, ...
  • UUID
  • IP address

Comparison:
  IF timestamps equal:
    Compare node IDs (lexicographically or numerically)

Advantages:
  ✓ Simple
  ✓ Deterministic
  ✓ No additional data needed

Disadvantages:
  ✗ Arbitrary (no semantic meaning)
  ✗ Might not match business logic
```

**Strategy 2: Value Hash**:

```
Hash the value being written:

  Write 1: ts=1000, value="Alice", hash=0x1234
  Write 2: ts=1000, value="Bob",   hash=0x5678

  Compare: 0x1234 < 0x5678
  Winner: Write 2

Advantages:
  ✓ Deterministic based on content
  ✓ Same value always produces same hash

Disadvantages:
  ✗ Still arbitrary
  ✗ Different values for same semantic meaning
```

**Strategy 3: Sequence Number**:

```
Each node maintains per-key sequence number:

Node A writes to key "user":
  ts=1000, seq=1 → "Alice"
  ts=1002, seq=2 → "Alice2"

Node B writes to key "user":
  ts=1000, seq=1 → "Bob"

Conflict at ts=1000:
  A: (ts=1000, node=A, seq=1)
  B: (ts=1000, node=B, seq=1)

Tie-break by node ID: B wins
```

**Strategy 4: Causal Information**:

```
Include vector clock in timestamp:

Write 1: ts=1000, VC=[2,1,0] → "Alice"
Write 2: ts=1000, VC=[1,2,0] → "Bob"

These are concurrent (neither VC < other)
Fall back to node ID or keep both versions
```

**Strategy 5: Application-Specific Logic**:

```
Example: Collaborative editing

Write 1: ts=1000, type="delete", text="hello"
Write 2: ts=1000, type="insert", text="world"

Tie-break rule:
  "Deletes always win over inserts at same timestamp"

Rationale: User intention to delete is stronger
Result: Delete wins
```

**Multi-Level Tie-Breaking**:

```
Complete ordering function:

Function CompareWrites(w1, w2):
  // Level 1: Timestamp
  IF w1.timestamp < w2.timestamp:
    RETURN w1 < w2
  IF w1.timestamp > w2.timestamp:
    RETURN w1 > w2

  // Level 2: Vector clock (if available)
  IF w1.vectorClock < w2.vectorClock:
    RETURN w1 < w2
  IF w1.vectorClock > w2.vectorClock:
    RETURN w1 > w2

  // Level 3: Node ID
  IF w1.nodeId < w2.nodeId:
    RETURN w1 < w2
  IF w1.nodeId > w2.nodeId:
    RETURN w1 > w2

  // Level 4: Sequence number
  IF w1.seqNum < w2.seqNum:
    RETURN w1 < w2
  IF w1.seqNum > w2.seqNum:
    RETURN w1 > w2

  // Level 5: Value hash (last resort)
  RETURN w1.valueHash < w2.valueHash
```

### Timestamp-Based Concurrency Control

**Timestamp-based concurrency control** uses timestamps to ensure serializability without locking.

**Basic Principle**:

```
Each transaction T gets timestamp TS(T) when it starts

Each data item X has:
  • Read timestamp: TS of last transaction that read X
  • Write timestamp: TS of last transaction that wrote X

Rules ensure:
  If T1's timestamp < T2's timestamp
  Then T1's effects appear before T2's
```

**Read Operation**:

```
Transaction T wants to read item X:

IF TS(T) < write_TS(X):
  // T is reading "too late"
  // X was already written by a later transaction
  ABORT T (or restart with new timestamp)
ELSE:
  Read X
  read_TS(X) = max(read_TS(X), TS(T))
```

**Example**:

```
Initial state:
  X: value=5, read_TS=0, write_TS=0

T1 (TS=10) writes X=7:
  TS(T1)=10 ≥ write_TS(X)=0 ✓
  X: value=7, write_TS=10

T2 (TS=15) reads X:
  TS(T2)=15 ≥ write_TS(X)=10 ✓
  Read value=7
  X: value=7, read_TS=15, write_TS=10

T3 (TS=12) wants to write X=9:
  TS(T3)=12 < read_TS(X)=15 ✗
  T2 already read X with later timestamp
  ABORT T3 (violates serializability)
```

**Write Operation**:

```
Transaction T wants to write item X:

IF TS(T) < read_TS(X):
  // T is writing "too late"
  // X was already read by a later transaction
  ABORT T
ELSE IF TS(T) < write_TS(X):
  // This write is obsolete (later write already happened)
  IGNORE this write (Thomas Write Rule)
ELSE:
  Write X
  write_TS(X) = TS(T)
```

**Advantages**:

```
✓ No locks needed (no deadlocks)
✓ Reads never block writes
✓ Writes never block reads
✓ Good for read-heavy workloads
```

**Disadvantages**:

```
✗ High abort rate with many conflicts
✗ Cascading aborts possible
✗ Requires storage for timestamps
✗ Starvation possible (long transaction keeps getting aborted)
```

**Optimizations**:

```
1. Optimistic Timestamp Ordering
   • Don't check timestamps until commit
   • Validate all reads/writes at once
   • Lower overhead for non-conflicting transactions

2. Multi-Version Timestamp Ordering (MVCC)
   • Keep multiple versions of each item
   • Each version has timestamp
   • Reads can always proceed (read older version)

3. Timestamp Allocation
   • Centralized: Single counter (bottleneck)
   • Hybrid Logical Clocks: Distributed but ordered
   • Block allocation: Each node gets range of timestamps
```

---

## Causality Tracking

**Causality** captures the fundamental "happened-before" relationship between events in a distributed system. Causality tracking is essential for ensuring consistency without sacrificing availability.

### Detecting Concurrent Operations

**Concurrent operations** are operations that have no causal relationship—neither happened before the other.

**Detection Methods**:

**Method 1: Vector Clocks**:

```
Two operations A and B with vector clocks V_A and V_B:

A → B (A happened before B):
  V_A < V_B (all components ≤, at least one <)

A ∦ B (A and B are concurrent):
  NOT (V_A < V_B) AND NOT (V_B < V_A)

Example:

Operation A: Write(x=5), V_A=[2,1,0]
Operation B: Write(x=7), V_B=[1,2,0]

Compare:
  V_A[0]=2 > V_B[0]=1 (A is ahead on process 0)
  V_A[1]=1 < V_B[1]=2 (B is ahead on process 1)

Neither V_A < V_B nor V_B < V_A
Result: A ∦ B (concurrent) → Conflict!
```

**Method 2: Version Vectors**:

```
Data item has version vector tracking updates:

Initial: []
Update at replica A: [(A,1)]
Update at replica B: [(B,1)]

Version [(A,1)] and [(B,1)] are concurrent
→ Both versions must be kept
→ Application must resolve conflict
```

**Method 3: Causal Histories**:

```
Each operation includes its causal history:

Op A: {dependencies: []}          (no dependencies)
Op B: {dependencies: [A]}          (depends on A)
Op C: {dependencies: [A]}          (depends on A)
Op D: {dependencies: [B, C]}       (depends on B and C)

B and C are concurrent:
  Neither is in the other's dependency set
  B ∦ C

D depends on both:
  B → D and C → D
```

**Visual Detection**:

```
Event graph:

    A ────> B ────> D
    │               ↑
    └─────> C ──────┘

Concurrent pairs:
  B and C (no path between them)

Sequential pairs:
  A → B (path exists)
  A → C (path exists)
  B → D (path exists)
  C → D (path exists)
  A → D (path through B or C)
```

### Causal Consistency Implementation

**Causal consistency** ensures that operations that are causally related are seen in the same order by all processes, but concurrent operations can be seen in any order.

**Definition**:

```
If operation A → operation B (A happened before B):
  Then all processes see A before B

If A ∦ B (concurrent):
  Different processes may see them in different orders
```

**Example**:

```
Process 1 writes: x=5  (Op A)
Process 1 writes: y=x  (Op B, depends on A)

Causally: A → B

All replicas MUST see:
  x=5 before y=5

But for concurrent operations:
  Process 2 writes: z=10 (Op C)
  Process 3 writes: w=20 (Op D)

  C ∦ D (concurrent)

  Replica 1 might see: C then D
  Replica 2 might see: D then C
  Both are valid!
```

**Implementation Approach 1: Dependency Tracking**:

```
Every operation carries causal dependencies:

Operation structure:
{
  id: unique_id,
  data: operation_data,
  dependencies: [list of operation IDs this depends on]
}

Example:

Op1: {id: "op1", data: "write(x=5)", deps: []}
Op2: {id: "op2", data: "write(y=10)", deps: ["op1"]}
Op3: {id: "op3", data: "write(z=15)", deps: ["op1"]}

Replica receives Op2:
  Check: Has it seen op1?
  IF not: Buffer Op2, wait for op1
  IF yes: Apply Op2

Replica receives Op3:
  Check: Has it seen op1?
  IF yes: Apply Op3 immediately

Op2 and Op3 can be applied in any order (both depend on op1 only)
```

**Implementation Approach 2: Vector Clocks**:

```
Each replica maintains vector clock:

Replica A: V_A = [0,0,0]
Replica B: V_B = [0,0,0]
Replica C: V_C = [0,0,0]

Operation at Replica A:
  Increment V_A[A]
  V_A = [1,0,0]
  Broadcast: (op_data, V_A=[1,0,0])

Replica B receives operation with V_op=[1,0,0]:
  Check: Can apply?
    V_op[A]=1 ≤ V_B[A]+1 ✓ (at most one ahead)
    V_op[B]=0 ≤ V_B[B] ✓
    V_op[C]=0 ≤ V_B[C] ✓

  Yes, apply immediately:
    Update V_B = [1,0,0]

Replica B receives operation with V_op=[5,2,0] when V_B=[3,0,0]:
  Check: Can apply?
    V_op[A]=5 > V_B[A]+1=4 ✗ (missing operations!)

  No, buffer it, wait for missing operations
```

**Implementation Approach 3: Causal Broadcast**:

```
Ensures messages are delivered in causal order:

Process A sends message M1
Process B receives M1, then sends M2

Causal broadcast guarantees:
  All processes receive M1 before M2
  (Because M1 → M2 causally)

Implementation using vector clocks:

Send message:
  Increment own vector clock entry
  Attach vector clock to message

Receive message M with V_M:
  Wait until:
    For sender S: V_M[S] = V_local[S] + 1
    For all others i: V_M[i] ≤ V_local[i]

  Then:
    Deliver message
    Update V_local[S] = V_M[S]
```

### Conflict Detection

**Conflict detection** identifies when concurrent operations modify the same data item, requiring resolution.

**Types of Conflicts**:

```
1. Write-Write Conflict:
   Two concurrent writes to the same key

   Replica A: Write(user:123, name="Alice")  V=[2,1,0]
   Replica B: Write(user:123, name="Bob")    V=[1,2,0]

   Both concurrent → Conflict!

2. Read-Write Conflict:
   Read and write to same key concurrently

   Replica A: Read(user:123)  V=[2,1,0]
   Replica B: Write(user:123) V=[1,2,0]

   Concurrent → Read might see old or new value

3. Structural Conflict:
   Concurrent operations on related data

   Replica A: Delete folder "/docs"
   Replica B: Create file "/docs/file.txt"

   Concurrent → Conflict!
```

**Detection Algorithm**:

```
Function DetectConflict(op1, op2):
  // Check if operations affect same data
  IF NOT SameDataItem(op1, op2):
    RETURN NO_CONFLICT

  // Check causal relationship using vector clocks
  V1 = op1.vectorClock
  V2 = op2.vectorClock

  IF V1 < V2:
    // op1 happened before op2
    RETURN NO_CONFLICT (op2 overrides op1)

  IF V2 < V1:
    // op2 happened before op1
    RETURN NO_CONFLICT (op1 overrides op2)

  // Neither V1 < V2 nor V2 < V1
  // Operations are concurrent
  RETURN CONFLICT
```

**Conflict Resolution Strategies**:

```
1. Last-Write-Wins (LWW):
   Use timestamp or replica ID to pick winner
   Simple but loses data

   IF conflict(op1, op2):
     winner = max(op1.timestamp, op2.timestamp)

2. Multi-Value (Keep Both):
   Store all conflicting versions
   Return all to client for resolution

   user:123 → [
     {name: "Alice", version: [2,1,0]},
     {name: "Bob", version: [1,2,0]}
   ]

3. Merge Function:
   Application-specific merge logic

   Shopping cart:
     Cart A: [item1, item2]  V=[2,1,0]
     Cart B: [item1, item3]  V=[1,2,0]
     Merged: [item1, item2, item3]  (union)

   Counter:
     Counter A: +5  V=[2,1,0]
     Counter B: +3  V=[1,2,0]
     Merged: +8  (sum)

4. Operational Transform:
   Transform concurrent operations to commute
   Used in collaborative editing

   Op A: Insert("hello", pos=0)
   Op B: Insert("world", pos=0)

   Transform B relative to A:
     B' = Insert("world", pos=5)  (after "hello")
```

**Practical Example: Riak**:

```
Riak's conflict handling:

1. Client writes:
   PUT user:123 = {name: "Alice"}
   Vector clock: [(replica_A, 1)]

2. Concurrent write on different replica:
   PUT user:123 = {name: "Bob"}
   Vector clock: [(replica_B, 1)]

3. Conflict detected:
   [(replica_A, 1)] ∦ [(replica_B, 1)]

4. Riak response:
   Returns BOTH values to client:
   [
     {name: "Alice", vclock: [(replica_A, 1)]},
     {name: "Bob", vclock: [(replica_B, 1)]}
   ]

5. Client resolves:
   Choose "Bob" (business logic)
   PUT with merged vclock:
     {name: "Bob", vclock: [(replica_A, 1), (replica_B, 1)]}

6. Future reads:
   See single value: {name: "Bob"}
   vclock indicates it descends from both conflicting versions
```

---

## Conclusion

Clock synchronization and time in distributed systems involve fundamental trade-offs:

**Logical Clocks** (Lamport, Vector):
- Precise causality tracking
- No dependency on physical time
- Can detect concurrent operations
- Best for: Ordering events, conflict detection, causal consistency

**Hybrid Logical Clocks**:
- Combines causality tracking with approximate physical time
- Bounded drift from real time
- Human-readable timestamps
- Best for: Modern distributed databases (CockroachDB, MongoDB)

**Physical Clocks** (NTP, PTP, GPS):
- Real wall-clock time
- External observability
- Clock skew and drift challenges
- Best for: Logging, monitoring, coordination with external systems

**TrueTime** (Google):
- Explicit uncertainty bounds
- Enables external consistency globally
- Requires specialized infrastructure
- Best for: Global-scale strongly consistent databases (Spanner)

**Choosing the Right Approach**:

```
Need strong consistency globally?
  → TrueTime (if you're Google) or HLC with transaction protocols

Need causality without coordination?
  → Vector clocks or version vectors

Need human-readable timestamps?
  → HLC or physical clocks with bounded skew

Need microsecond accuracy?
  → PTP with hardware support

General distributed system?
  → HLC (best balance of features)
```

The key insight: Perfect global time is impossible in distributed systems, but we can build correct systems by carefully choosing the right time abstraction for our consistency and coordination needs.