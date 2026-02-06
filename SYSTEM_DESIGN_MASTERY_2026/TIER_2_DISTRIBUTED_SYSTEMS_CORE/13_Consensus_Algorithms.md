# Consensus Algorithms

## Introduction

**Consensus** is one of the most fundamental problems in distributed systems: How do multiple independent computers agree on a single value or sequence of events when they communicate over an unreliable network and some of them may fail?

**Layman explanation:** Imagine a group of friends trying to decide where to eat dinner, but they can only communicate by passing notes through unreliable mail. Some friends might not receive all the notes, some notes might arrive out of order, and some friends might fall asleep mid-conversation. Consensus algorithms are the rules that help the group still reach an agreement despite these challenges.

### Why Consensus Matters

Consensus is critical for building reliable distributed systems. It enables:

- **Consistency**: All nodes agree on the same data
- **Coordination**: Nodes can coordinate actions (like leader election)
- **Fault tolerance**: System continues working despite failures
- **State machine replication**: Multiple servers maintain identical state

### Common Use Cases

```
┌─────────────────────────────────────────────────────┐
│           CONSENSUS IN PRACTICE                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Leader Election:                                   │
│  [Node A] [Node B] [Node C] → Agree: Node B leads  │
│                                                     │
│  Distributed Lock:                                  │
│  Which node gets exclusive access to resource?     │
│                                                     │
│  Configuration Management:                          │
│  All nodes agree on: { replicas: 3, timeout: 5s }  │
│                                                     │
│  Transaction Commit:                                │
│  Should transaction #12345 commit or abort?        │
│                                                     │
│  Log Replication:                                   │
│  All nodes agree on order: [op1, op2, op3, ...]   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### The Fundamental Challenge

Achieving consensus is difficult because distributed systems face:

1. **Network unreliability**: Messages can be lost, delayed, or duplicated
2. **Node failures**: Servers can crash at any time
3. **Asynchronous communication**: No global clock, unbounded message delays
4. **Partial failures**: Some nodes may fail while others continue

**Key impossibility result**: The FLP (Fischer, Lynch, Paterson) impossibility result proves that in a fully asynchronous system with even one potential crash failure, no deterministic consensus algorithm can guarantee termination. In practice, algorithms work around this by making timing assumptions.

### Consensus Algorithm Properties

Good consensus algorithms must satisfy:

**Safety properties** (nothing bad happens):
- **Agreement**: All non-faulty nodes decide on the same value
- **Validity**: The decided value must have been proposed by some node
- **Integrity**: Nodes decide at most once

**Liveness properties** (something good eventually happens):
- **Termination**: All non-faulty nodes eventually decide

---

## Raft Algorithm

**Raft** is a consensus algorithm designed to be understandable. Created by Diego Ongaro and John Ousterhout in 2013, Raft was explicitly designed to be easier to understand than Paxos while providing equivalent functionality.

**Design philosophy**: "If a system is to serve a practical purpose, it must be easy to understand. We believe that Raft is easier to understand than other consensus algorithms."

**Key idea**: Raft decomposes consensus into three relatively independent subproblems:
1. Leader election
2. Log replication
3. Safety

### Raft Basics

Raft organizes servers into three states:

```
┌─────────────────────────────────────────────────┐
│              RAFT SERVER STATES                 │
│                                                 │
│                  ┌──────────┐                   │
│                  │  LEADER  │                   │
│                  └──────────┘                   │
│                       ↑                         │
│                       │ Election                │
│                       │ timeout                 │
│    ┌──────────┐       │       ┌──────────┐     │
│    │ FOLLOWER │───────┴──────→│CANDIDATE │     │
│    └──────────┘                └──────────┘     │
│         ↑                           │           │
│         │                           │           │
│         └───────────────────────────┘           │
│           Discovers current leader              │
│           or new election term                  │
│                                                 │
└─────────────────────────────────────────────────┘
```

**State descriptions:**

- **Follower**: Passive state. Responds to requests from leaders and candidates. All nodes start as followers.

- **Candidate**: Tries to become a leader by requesting votes from other nodes. Entered when follower doesn't hear from leader.

- **Leader**: Handles all client requests and replicates log entries to followers. Only one leader per term.

### Raft Terms

Raft divides time into **terms** of arbitrary length, numbered with consecutive integers.

```
Timeline divided into terms:

Term 1    Term 2    Term 3         Term 4    Term 5
├─────────┼─────────┼─────────┬────┼─────────┼──────→
│         │         │ Election│    │         │
│ Leader A│ Leader B│ Failed  │    │ Leader C│
│         │         │ (split) │    │         │
└─────────┴─────────┴─────────┴────┴─────────┴──────→

Each term begins with an election.
Terms may have no leader if election fails (split vote).
```

**Term properties:**
- Terms act as a logical clock in the system
- Each server maintains current term number
- Terms are monotonically increasing
- Servers reject requests with older term numbers
- If a server's term is stale, it updates to the newer term

### Leader Election

Leader election ensures there is at most one leader per term.

**When does an election occur?**
- System starts up (no leader exists)
- Follower's election timeout expires (leader appears dead)
- Current leader fails

**Election process:**

```
NORMAL OPERATION:
[Leader] ─heartbeat→ [Follower A] ─heartbeat→ [Follower B]
   ↓                      ↓                        ↓
 Sends empty            Resets                  Resets
 AppendEntries         timeout                 timeout


LEADER FAILURE:
[Leader] X             [Follower A]            [Follower B]
                            ↓                        ↓
                       No heartbeat             No heartbeat
                       Timeout!                 Timeout!
                            ↓                        ↓
                       CANDIDATE                CANDIDATE
                       Term = 2                 Term = 2
                            ↓                        ↓
                       Votes for self           Votes for self
                       Requests votes           Requests votes
                            ↓                        ↓
                       ┌────┴─────────────────────┐
                       ↓                          ↓
                   Gets majority             Becomes LEADER
```

**Step-by-step election process:**

1. **Timeout triggers candidacy**:
   - Follower increments term
   - Transitions to candidate state
   - Votes for itself
   - Sends RequestVote RPCs to all other servers

2. **Voting rules**:
   - Each server votes for at most one candidate per term
   - First-come, first-served basis
   - Candidate must have log at least as up-to-date as voter's log

3. **Election outcomes**:

   **Scenario A: Candidate wins (most common)**
   - Receives votes from majority of servers
   - Becomes leader
   - Sends heartbeats to establish authority and prevent new elections

   **Scenario B: Another server becomes leader**
   - Receives AppendEntries from new leader
   - Reverts to follower state

   **Scenario C: Split vote (election fails)**
   - No candidate receives majority
   - Timeout occurs
   - New election starts with incremented term

**Layman explanation:** Think of it like a classroom where the teacher (leader) regularly calls out "I'm here!" (heartbeat). If students don't hear the teacher for a while, they assume the teacher left. Students then nominate themselves to be the new teacher and ask classmates to vote for them. Whoever gets votes from more than half the class becomes the new teacher.

**Election timeout randomization:**

To prevent repeated split votes, Raft uses randomized election timeouts (typically 150-300ms).

```
Node A timeout: 200ms ───────────→ Starts election
Node B timeout: 250ms ────────────────→ (Usually too late)
Node C timeout: 180ms ─────────→ Starts election first!

With randomization, one node usually starts election
before others timeout, winning election quickly.
```

**Why randomization works:**
- Reduces probability of split votes
- First candidate to start election usually wins
- If split vote occurs, new random timeouts prevent repeated splits

### Log Replication

Once a leader is elected, it services client requests. Each client request contains a command to be executed by the replicated state machines.

**The goal**: Ensure all servers execute the same commands in the same order, keeping their state machines in sync.

**Log structure:**

```
Each log entry stores:
- Command from client
- Term number when entry was received
- Index position in log

Server A (Leader):
Index:    1       2       3       4       5
Term:    [1]     [1]     [2]     [2]     [3]
Cmd:     [x=5]   [y=2]   [x=7]   [z=1]   [y=9]
         ↓       ↓       ↓       ↓       ↓
       Committed──────────────→ Not yet committed

Server B (Follower):
Index:    1       2       3       4
Term:    [1]     [1]     [2]     [2]
Cmd:     [x=5]   [y=2]   [x=7]   [z=1]
         ↓       ↓       ↓       ↓
       Committed──────────→
```

**Key terminology:**

- **Committed entry**: Entry is safe to apply to state machine. Guaranteed to never be lost.
- **Applied entry**: Entry whose command has been executed by the state machine.
- **Log matching property**: If two logs contain an entry with the same index and term, they are identical up to that index.

**Log replication process:**

```
CLIENT REQUEST FLOW:

Step 1: Client sends command to leader
        Client → [Leader]: "SET x = 5"

Step 2: Leader appends to local log
        Leader's log: [..., (index=5, term=3, cmd="SET x=5")]

Step 3: Leader sends AppendEntries to followers
        Leader → [Follower A]: AppendEntries(entries=[...])
        Leader → [Follower B]: AppendEntries(entries=[...])

Step 4: Followers append to logs and acknowledge
        Follower A → Leader: Success
        Follower B → Leader: Success

Step 5: Leader waits for majority acknowledgment
        Majority reached? Yes (Leader + 2 followers = 3/5)

Step 6: Leader commits entry
        - Marks entry as committed in its log
        - Applies command to state machine
        - Responds to client: Success

Step 7: Leader notifies followers in next AppendEntries
        - Followers learn about commit
        - Followers apply command to their state machines
```

**AppendEntries RPC:**

The leader uses AppendEntries for two purposes:
1. **Heartbeats**: Empty AppendEntries (no entries) to maintain leadership
2. **Log replication**: AppendEntries with log entries to replicate

**AppendEntries contains:**
- Leader's term
- Leader's ID
- Index of log entry immediately preceding new ones
- Term of that preceding entry
- Entries to store (empty for heartbeat)
- Leader's commit index

**Follower's AppendEntries handling:**

```
FOLLOWER RECEIVES AppendEntries:

1. Is leader's term >= my term?
   NO → Reject (stale leader)
   YES → Continue

2. Does my log contain entry at prevLogIndex with term = prevLogTerm?
   NO → Reject (log inconsistency detected)
   YES → Continue

3. If conflict found (existing entry with same index but different term):
   → Delete conflicting entry and all that follow

4. Append any new entries not in log

5. If leaderCommit > commitIndex:
   → commitIndex = min(leaderCommit, index of last new entry)

6. Return success
```

**Layman explanation:** Imagine the leader is a teacher taking notes on a whiteboard, and students (followers) are copying those notes into their notebooks. The teacher regularly checks that students have copied everything correctly by asking "Do you have notes up to line 5?" If a student is behind or has wrong notes, the teacher helps them correct it before moving forward.

**Handling log inconsistencies:**

Follower logs may differ from leader's due to previous leader crashes. Raft handles this by forcing followers' logs to match the leader's.

```
EXAMPLE: Log inconsistency after crashes

Leader (Term 8):
Index: 1    2    3    4    5    6    7    8    9    10   11
Term: [1]  [1]  [1]  [4]  [4]  [5]  [5]  [6]  [6]  [6]  [8]

Follower A (missing entries):
Index: 1    2    3    4    5    6    7    8
Term: [1]  [1]  [1]  [4]  [4]  [5]  [5]  [6]

Follower B (extra uncommitted entries from old term):
Index: 1    2    3    4    5    6    7    8    9    10   11
Term: [1]  [1]  [1]  [4]  [4]  [5]  [5]  [6]  [6]  [6]  [7]
                                                            ↑
                                                    Wrong term!

Follower C (missing entries and has extra):
Index: 1    2    3    4    5    6    7
Term: [1]  [1]  [1]  [4]  [4]  [4]  [4]
```

**Leader's repair process:**

1. Leader maintains `nextIndex` for each follower (next log entry to send)
2. If AppendEntries fails due to log inconsistency:
   - Decrement `nextIndex` for that follower
   - Retry AppendEntries with earlier entries
3. Eventually find the point where logs match
4. Follower deletes conflicting entries
5. Follower appends entries from leader
6. Logs are now consistent

**Optimization**: Instead of decrementing by 1 each time, follower can return information about conflicting term to allow leader to skip all entries in that term.

### Safety Guarantees

Raft provides strong safety guarantees to ensure correctness:

**1. Election Safety**
- **Guarantee**: At most one leader can be elected in a given term
- **How**: Each server votes for at most one candidate per term (first-come first-served)
- **Why it matters**: Prevents split-brain scenarios

**2. Leader Append-Only**
- **Guarantee**: Leader never overwrites or deletes entries in its log; only appends
- **How**: By design, leader's log is immutable except for appends
- **Why it matters**: Simplifies reasoning about log consistency

**3. Log Matching Property**
- **Guarantee**: If two logs contain an entry with same index and term, then:
  - The entries contain the same command
  - The logs are identical in all preceding entries
- **How**: AppendEntries consistency check (prevLogIndex and prevLogTerm)
- **Why it matters**: Ensures all servers have consistent view of committed entries

**4. Leader Completeness**
- **Guarantee**: If a log entry is committed in a given term, that entry will be present in the leaders' logs for all higher-numbered terms
- **How**: Voting restriction—candidate must have log at least as up-to-date as voter
- **Why it matters**: Ensures committed entries are never lost

**5. State Machine Safety**
- **Guarantee**: If a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index
- **How**: Follows from Leader Completeness + Log Matching
- **Why it matters**: This is the ultimate goal—all servers execute same commands in same order

**The "up-to-date" check:**

Critical for safety: a candidate can only win election if its log is at least as up-to-date as any other log in the majority.

```
Comparing logs for "up-to-date":

Log A: [1,1,1,4,4,5,5,6]
Log B: [1,1,1,4,4,5,5]

Question: Which is more up-to-date?
Answer: Log A (has later term 6)

Log A: [1,1,1,4,4,5,5,5]
Log B: [1,1,1,4,4,5,5]

Question: Which is more up-to-date?
Answer: Log A (same last term, but longer)

Comparison rules:
1. If last terms differ → log with later term is more up-to-date
2. If last terms same → longer log is more up-to-date
```

**Commitment rules:**

A leader cannot immediately mark an entry as committed when it's replicated to majority. Special rules apply:

```
COMMITMENT PROBLEM:

Term 2 Leader:
[1,1,1,2,2] → Replicated to majority, but not yet committed
              Leader crashes before committing!

Term 3 Leader (new):
[1,1,1,3] → Could overwrite entry at index 4 from term 2!

SOLUTION:
- Leader only commits entries from its current term when
  replicated to majority
- Old-term entries get committed indirectly when a current-term
  entry is committed
```

**Commitment rule**: A log entry is committed once:
1. It's replicated on a majority of servers, AND
2. At least one entry from the leader's current term is also replicated on a majority

**Layman explanation:** It's like a teacher (leader) who won't consider homework "officially graded" (committed) until they've verified that most students have that assignment AND at least one newer assignment. This prevents old, possibly incorrect homework from being counted if there's been a teacher change.

### Membership Changes

Real systems need to add or remove servers (for maintenance, scaling, or replacing failed machines) without downtime. This is challenging because naive approaches can lead to split-brain.

**The problem with naive approaches:**

```
DANGEROUS: Switching directly from old to new configuration

Old config: {A, B, C} (majority = 2)
New config: {A, B, C, D, E} (majority = 3)

Problem during transition:
Old majority: A, B could elect leader X
New majority: C, D, E could elect leader Y
→ Two leaders in same term! (split-brain)

     Old Config          New Config
    {A, B, C}           {A, B, C, D, E}
        ↓                      ↓
    Leader X              Leader Y
```

**Raft's solution: Joint Consensus**

Raft uses a two-phase approach with a transitional "joint consensus" configuration.

```
SAFE: Two-phase membership change

Phase 1: Enter joint consensus (C-old,new)
- System uses BOTH old and new configurations
- Requires majority in BOTH configurations for decisions
- Log entries must be replicated to majorities of BOTH

Phase 2: Transition to new configuration (C-new)
- Once C-old,new is committed, transition to C-new
- Now only new configuration is used
```

**Timeline of membership change:**

```
Time ──────────────────────────────────────────────────→

[C-old]──────┬──────[C-old,new]──────┬──────[C-new]────→
             │                       │
             │                       │
        C-old,new             C-new committed
        committed             Start using C-new only
        Both configs
        required for
        majority
```

**Key properties:**
- Never a point where old and new majorities can make independent decisions
- Cluster continues serving requests throughout change
- Can be aborted if problems occur during C-old,new phase
- Multiple changes cannot be in progress simultaneously

**Simplified approach (single-server changes):**

If only adding or removing one server at a time, can use simpler approach:
- Add/remove one server
- Wait for it to catch up or be removed
- Then add/remove next server

This works because majorities always overlap by at least one server when changing by one.

**Layman explanation:** Imagine a committee of 3 people deciding to expand to 5 people. If they immediately let all 5 vote, the original 3 might make one decision while the new 5 make a different decision. Instead, there's a transition period where both "the original 3" AND "all 5 people" must agree—ensuring continuity and preventing conflicts.

### Raft Optimizations

Several optimizations can improve Raft's performance and practicality:

#### 1. Log Compaction (Snapshots)

**Problem**: Logs grow unbounded, consuming memory and requiring long replays

**Solution**: Periodically compact logs by taking snapshots

```
Before snapshot:
Index: 1    2    3    4    5    6    7    8    9    10
       [x=1][y=2][x=3][y=4][x=5][y=6][x=7][y=8][x=9][y=10]

After snapshot (at index 5):
Snapshot: {x=5, y=6}  |  Index: 6    7    8    9    10
Last included: 5, 2   |         [y=6][x=7][y=8][x=9][y=10]
```

**Snapshot includes:**
- Complete state machine state
- Last included index and term
- Latest cluster configuration

**Benefits:**
- Bounded memory usage
- Faster recovery (apply snapshot instead of replaying all entries)
- Can discard old log entries

**Challenge**: Follower might be so far behind it needs snapshot instead of log entries

**Solution**: Leader sends `InstallSnapshot` RPC with snapshot data

#### 2. Batching and Pipelining

**Batching**: Group multiple client requests into single AppendEntries RPC
- Reduces number of RPCs
- Amortizes network overhead
- Improves throughput

**Pipelining**: Send next AppendEntries before receiving acknowledgment for previous
- Reduces latency
- Keeps network saturated
- Leader maintains multiple outstanding requests

#### 3. Asynchronous Application

Apply committed entries to state machine asynchronously:
- Leader can commit and respond to client before applying
- Followers apply in background
- Improves latency for clients

#### 4. Local Read Optimization

**Problem**: Read requests go through consensus (slow)

**Optimization**: Leader serves reads from local state without consensus

**Requirement**: Must verify it's still leader
- Send heartbeat to majority before serving read
- Use lease mechanism (leader knows it's leader for next N milliseconds)

**Benefit**: Much faster reads (single roundtrip vs. full consensus)

#### 5. Leadership Transfer

Smoothly transfer leadership to another server:
- Current leader stops accepting new requests
- Ensures target server's log is up-to-date
- Sends TimeoutNow message to trigger immediate election
- Target server wins election (has most up-to-date log)

**Use cases:**
- Planned maintenance
- Load balancing across regions
- Smooth shutdowns

#### 6. Pre-Vote Phase

**Problem**: A server with very out-of-date log can disrupt cluster by starting elections

**Solution**: Add pre-vote phase before actual election
- Candidate asks "Would you vote for me?"
- Doesn't increment term
- Only starts real election if would win
- Prevents disruption from partitioned servers

**Layman explanation:** Before officially running for class president (which disrupts the class), a student first asks classmates "Would you vote for me if I ran?" Only if they'd likely win do they actually start a formal campaign.

---

## Paxos Algorithm

**Paxos** is one of the first and most important consensus algorithms, invented by Leslie Lamport in 1989. It's notoriously difficult to understand, leading Lamport to later write "Paxos Made Simple" (though it's still considered complex).

**Key difference from Raft**: Paxos is more theoretical and foundational, while Raft was designed to be practical and understandable.

### The Paxos Challenge

Paxos solves consensus in the face of:
- Asynchronous network (no timing guarantees)
- Crash failures (nodes can fail and recover)
- Message loss, delay, and duplication

**What Paxos does NOT handle**: Byzantine failures (nodes behaving maliciously)

### Paxos Roles

Paxos nodes can play multiple roles:

```
┌─────────────────────────────────────────────────┐
│              PAXOS ROLES                        │
├─────────────────────────────────────────────────┤
│                                                 │
│  PROPOSERS:                                     │
│  - Propose values                               │
│  - Handle client requests                       │
│  - Drive consensus rounds                       │
│                                                 │
│  ACCEPTORS:                                     │
│  - Vote on proposed values                      │
│  - Remember promises and accepted values        │
│  - Form quorum for safety                       │
│                                                 │
│  LEARNERS:                                      │
│  - Learn which value was chosen                 │
│  - Don't participate in voting                  │
│  - Can be clients or replicated state machines  │
│                                                 │
│  In practice: One node often plays all roles    │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Layman explanation:** Think of Paxos roles like a government:
- **Proposers** are like legislators who propose laws
- **Acceptors** are like voting members of Congress who approve laws
- **Learners** are like citizens who learn about approved laws but don't vote

### Basic Paxos (Single-Decree Paxos)

Basic Paxos reaches consensus on a single value. The protocol operates in two phases.

**Proposal numbers:**
- Each proposal has a unique number (globally ordered)
- Higher numbers take precedence
- Typically: `round_number.server_id` (e.g., 1.5, 2.3, 3.1)

**Phase 1: Prepare Phase**

```
PREPARE PHASE:

Proposer                  Acceptors (A, B, C)
   │                           │  │  │
   │─── Prepare(n=5) ─────────→  │  │
   │                           │  │  │
   │                        Promise(5)
   │                        "I won't accept
   │                         proposals < 5"
   │                           │  │  │
   │←──── Promise(5, null) ────  │  │
   │←──── Promise(5, null) ───────  │
   │←──── Promise(5, null) ──────────
   │
   Majority received → Continue to Phase 2
```

**Prepare(n) request**: Proposer asks acceptors:
- "Promise not to accept any proposals numbered less than n"
- "Tell me the highest-numbered proposal you've accepted (if any)"

**Promise response**: Acceptor replies:
- "I promise not to accept proposals < n"
- "The highest proposal I accepted was: (n', v')" OR "I haven't accepted anything"
- If acceptor already promised a higher number: rejects

**Phase 2: Accept Phase**

```
ACCEPT PHASE:

Proposer                  Acceptors (A, B, C)
   │                           │  │  │
   │──── Accept(n=5, v) ──────→  │  │
   │    "Please accept           │  │  │
   │     value v"                │  │  │
   │                          Accept
   │                          & Store
   │                           │  │  │
   │←──── Accepted(5, v) ───────  │  │
   │←──── Accepted(5, v) ──────────  │
   │←──── Accepted(5, v) ─────────────
   │
   Majority accepted → Value v is chosen!
```

**Accept(n, v) request**: Proposer asks acceptors:
- "Please accept proposal numbered n with value v"

**Value selection rule**:
- If any acceptor in Phase 1 reported an accepted value: proposer MUST use highest-numbered accepted value
- Otherwise: proposer can choose any value (typically from client request)

**Accepted response**: Acceptor replies:
- Accepts if hasn't promised to ignore n
- Rejects if promised a higher number

**Layman explanation:** Imagine trying to decide on a restaurant:
- Phase 1 (Prepare): You call friends saying "I'm planning option #5, is that okay? Also, have you agreed to any other plans?" Friends either say "Yes, I'm free" or "No, I already agreed to plan #7"
- Phase 2 (Accept): Once enough friends are available, you say "Let's go to Pizza Place for plan #5." If enough agree, the decision is made!

**Complete execution example:**

```
Scenario: Two proposers trying to reach consensus

Proposer P1                 Acceptors               Proposer P2
                         (A1, A2, A3)

1. P1 sends Prepare(1)
   ─────────────────→  Promise(1)
   ←─────────────────  from A1, A2, A3
                       (no prior accepts)

2. P1 sends Accept(1, "X")
   ─────────────────→
                       A1, A2 accept...
                       But A3 is slow

3.                                      P2 sends Prepare(2)
                       ←─────────────────
                       Promise(2, accepted=(1,"X"))
                       from A2, A3
                       (A2 reports it accepted 1,"X")

4.                                      P2 must use "X"!
                                        (highest accepted value)
                                        P2 sends Accept(2, "X")
                       ←─────────────────
                       A2, A3 accept

5. A1 receives Accept(1, "X")
   ─────────────────→
                       But A1 already promised
                       to ignore proposals < 2
                       Rejects!

Result: Value "X" is chosen (accepted by A2, A3 with n=2)
```

**Key safety property**: Once a value is chosen, future proposals must propose the same value. This is ensured by the "highest-numbered accepted value" rule.

### Multi-Paxos

Basic Paxos is inefficient for reaching consensus on a sequence of values (like a replicated log). **Multi-Paxos** optimizes this.

**Problem with repeated Basic Paxos:**
- Need full two-phase protocol for each log entry
- High latency (4 message delays per decision)
- High message count

**Multi-Paxos optimizations:**

#### 1. Stable Leader

Instead of any node proposing, elect a distinguished proposer (leader):
- Only leader proposes values
- Eliminates contention between proposers
- Leader can skip Phase 1 for subsequent proposals

```
Leader Election (One-time Phase 1):

Leader candidate sends Prepare(n) to all
Receives promises from majority
Becomes established leader

Subsequent Proposals (Phase 2 only):

For log entry 1: Leader sends Accept(n, v1)
For log entry 2: Leader sends Accept(n, v2)
For log entry 3: Leader sends Accept(n, v3)
...

Only 2 message delays instead of 4!
```

**Why it works**: Once leader successfully completes Phase 1 with proposal number n, it can reuse n for multiple values, only sending Phase 2 messages.

#### 2. Batching

Leader batches multiple client requests:
- Send Accept messages for multiple values together
- Reduces message count
- Improves throughput

#### 3. Pipelining

Leader sends Accept messages for entry i+1 before receiving acceptances for entry i:
- Keeps network busy
- Reduces latency
- Maintains multiple outstanding proposals

**Multi-Paxos flow:**

```
Normal operation with stable leader:

Client → Leader: Request 1
Client → Leader: Request 2
Client → Leader: Request 3

Leader → Acceptors: Accept(n, [v1, v2, v3])  (batched)

Acceptors → Leader: Accepted

Leader → Learners: Values v1, v2, v3 chosen

Leader → Clients: Responses
```

**Handling leader failure:**

When leader fails, system must detect failure and elect new leader:
1. Followers timeout waiting for heartbeats
2. New leader candidate runs Phase 1
3. Learns about any uncommitted values
4. Becomes new leader

**Comparison to Raft:**

Multi-Paxos with optimizations is very similar to Raft:
- Both have stable leader
- Both replicate log entries
- Both handle leader failure similarly

**Key differences:**
- Raft specifies complete system (leader election, log replication, safety)
- Multi-Paxos is more of a framework (many details left to implementer)
- Raft is generally considered easier to understand and implement

### Paxos Made Simple - Key Insights

Leslie Lamport's "Paxos Made Simple" paper distills Paxos to its essence:

**The fundamental insight**: The protocol works by ensuring:
1. An acceptor can accept multiple proposals
2. But once a value is chosen (accepted by majority), all subsequent proposals must propose that same value
3. This is guaranteed by requiring proposers to check what was previously accepted

**Why the two-phase protocol?**
- **Phase 1**: Establishes proposer's right to propose (gets promise from majority)
- **Phase 2**: Actually proposes the value

**Why check for previously accepted values?**
- Ensures safety: if value already chosen, new proposal must use same value
- Maintains consistency across all consensus instances

**The beauty of Paxos**: Minimal protocol that provably solves consensus despite failures and asynchrony.

**The challenge of Paxos**: Many practical details left unspecified:
- Leader election mechanism
- Failure detection
- Log compaction
- Membership changes
- Performance optimizations

This is why modern systems often prefer Raft—it specifies these details.

---

## EPaxos (Egalitarian Paxos)

**EPaxos** is a leaderless consensus algorithm that provides high performance for geo-distributed systems. Created by Iulian Moraru and others in 2013.

**Key innovation**: No stable leader—any replica can initiate and commit commands with low latency.

### Why EPaxos?

**Limitations of leader-based consensus** (Raft, Multi-Paxos):
- Commands must go through leader
- If leader is geographically distant from client, high latency
- Leader can become bottleneck
- Leader failure causes unavailability

```
Leader-based (Multi-Paxos/Raft):

Client in Europe → Leader in US West → Replicas worldwide
                   (High latency)

All commands funnel through US West leader
```

**EPaxos advantages**:
- Clients can send commands to nearest replica
- Fast path: commit in one round trip to quorum
- No leader bottleneck
- Better for geo-distributed deployments

```
EPaxos (Leaderless):

Client in Europe → Replica in Europe → Quorum
                   (Low latency)

Client in Asia → Replica in Asia → Quorum
                 (Low latency)

Each replica can independently commit commands
```

### Core Concepts

#### Command Dependencies

EPaxos tracks **dependencies** between commands rather than imposing a total order.

**Dependency**: Command A depends on command B if:
- They access the same data (e.g., both modify variable X)
- B was proposed before A

```
Example dependencies:

Command 1: SET x = 5
Command 2: SET y = 10    (no dependency on Cmd 1)
Command 3: SET x = 7     (depends on Cmd 1 - same key)
Command 4: GET x         (depends on Cmd 1, 3)

Dependency graph:
    Cmd 1 ────→ Cmd 3 ────→ Cmd 4
    Cmd 2 (independent)
```

**Key insight**: Commands that don't conflict can be executed in any order. Only conflicting commands need ordering.

#### Execution Order

EPaxos determines execution order by analyzing the dependency graph:
1. Build transitive closure of all dependencies
2. Execute commands respecting dependency order
3. Commands without dependencies can execute concurrently

**Layman explanation:** Imagine a kitchen where chefs prepare different dishes. Some dishes are independent (salad and soup), while others depend on each other (you must bake the cake before frosting it). EPaxos lets chefs work on independent dishes simultaneously while ensuring dependent tasks happen in the right order.

### EPaxos Protocol

#### Fast Path (Normal Case)

When there are no conflicts with concurrent commands:

```
FAST PATH (1 round trip):

1. Command replica (leader for this instance):
   - Assigns sequence number
   - Computes dependencies
   - Sends PreAccept to fast quorum (⌈N/2⌉ + ⌈N/4⌉ replicas)

2. Fast quorum replicas:
   - Check for conflicts
   - Return dependencies

3. If all replicas agree on dependencies:
   - Command is committed!
   - Execute when dependencies satisfied

Total: 1 RTT (round-trip time)
```

#### Slow Path (Conflicting Commands)

When replicas disagree on dependencies:

```
SLOW PATH (2 round trips):

1. PreAccept phase (as above)

2. Replicas report different dependencies

3. Command replica runs Paxos Accept phase:
   - Sends Accept with union of reported dependencies
   - Waits for majority acknowledgment

4. Commit phase:
   - Sends Commit message
   - Command is committed

Total: 2 RTT
```

### Commit Ordering

EPaxos doesn't impose a global total order on all commands. Instead:
- Each command has a sequence number within its replica
- Commands have explicit dependencies
- Execution order determined by dependency graph

**Execution process:**

```
1. Wait for all dependencies to be committed
2. Execute dependencies in dependency order
3. Execute this command
4. Return result

Example execution:

Committed commands:
- Cmd A (seq=1): SET x=5, deps=[]
- Cmd B (seq=2): SET y=10, deps=[]
- Cmd C (seq=3): SET x=7, deps=[A]
- Cmd D (seq=4): GET x, deps=[A,C]

Execution order: A → C → D (B can run anytime)
```

### Geo-Distributed Optimization

EPaxos shines in geo-distributed scenarios:

**Traditional leader-based consensus**:
- WAN latency to leader: ~100ms
- Total latency: 200ms+ (request + response)

**EPaxos fast path**:
- LAN latency to local replica: ~1ms
- WAN latency to fast quorum: ~100ms
- Total latency: ~100ms (single round trip)

**Performance gains**:
- 2x latency reduction for commits
- No single point of failure
- Better load distribution
- Scales with number of replicas

### EPaxos Trade-offs

**Advantages**:
- Low latency for geo-distributed clients
- No leader bottleneck
- Symmetric load across replicas
- Handles conflicts automatically

**Disadvantages**:
- More complex than leader-based protocols
- Dependency tracking overhead
- Slow path when conflicts occur
- More complex recovery procedures
- Graph execution more complex than sequential log

**When to use EPaxos**:
- Geo-distributed deployment across multiple datacenters
- Workload with low conflict rate
- Need for low latency from multiple regions
- Symmetric load desired

**When not to use EPaxos**:
- Single datacenter (leader-based is simpler)
- High conflict rate (slow path becomes common)
- Simplicity valued over optimal latency

---

## Byzantine Fault Tolerance

**Byzantine Fault Tolerance (BFT)** refers to consensus algorithms that can handle Byzantine failures—nodes that behave arbitrarily, including malicious behavior.

### The Byzantine Generals Problem

The classical formulation by Lamport, Shostak, and Pease (1982):

**Scenario**: Byzantine generals surround a city. They must coordinate to attack or retreat. Some generals may be traitors who send conflicting messages.

**Goal**: All loyal generals must agree on the same plan (attack or retreat).

**Challenge**: Cannot distinguish between:
- A traitor general sending different messages to different generals
- A loyal general whose messages were corrupted
- Network delays vs. generals not responding

```
BYZANTINE BEHAVIOR EXAMPLE:

General Traitor T sends:
  To General A: "ATTACK"
  To General B: "RETREAT"

How do A and B reach consensus?
```

**Proven impossibility**: With 3 generals and 1 traitor, no algorithm can guarantee consensus.

**General result**: Need at least **3f + 1** nodes to tolerate f Byzantine failures.

**Why 3f + 1?**
- f nodes can be Byzantine
- f nodes might be slow/partitioned (appear faulty)
- Need f + 1 nodes remaining to form majority
- Total: f (Byzantine) + f (slow) + f + 1 (majority) = 3f + 1

**Layman explanation:** Imagine a committee where up to 1/3 of members might lie or act maliciously. You need at least 4 people total (3×1 + 1) so that even if 1 person lies and 1 person is unavailable, you still have 2 honest people who can agree.

### PBFT (Practical Byzantine Fault Tolerance)

**PBFT**, created by Miguel Castro and Barbara Liskov in 1999, was the first practical BFT algorithm for asynchronous networks.

**Key innovation**: Made BFT practical by reducing message complexity from exponential to polynomial.

**PBFT assumptions**:
- Asynchronous network (messages eventually delivered)
- Independent node failures (no coordinated attacks)
- Cryptographic signatures prevent forgery
- 3f + 1 replicas to tolerate f Byzantine faults

#### PBFT Protocol Phases

PBFT operates in views (similar to Raft terms). Each view has a primary (leader).

```
PBFT THREE-PHASE PROTOCOL:

Client                    Primary              Replicas
  │                         │                  (R1, R2, R3)
  │                         │                    │  │  │
  │─── Request ────────────→                     │  │  │
  │                         │                    │  │  │
  │                    PRE-PREPARE PHASE:        │  │  │
  │                         │                    │  │  │
  │                    Assign seq #              │  │  │
  │                    Broadcast                 │  │  │
  │                         │────────────────────→  │  │
  │                         │                       │  │
  │                    PREPARE PHASE:              │  │
  │                         │                       │  │
  │                         │←──────────────────────  │
  │                         │←─────────────────────────
  │                         │                       │  │
  │                    Collect 2f                  │  │
  │                    prepares                    │  │
  │                         │                       │  │
  │                    COMMIT PHASE:               │  │
  │                         │                       │  │
  │                         │────────────────────→  │  │
  │                         │────────────────────────→ │
  │                         │                       │  │
  │                         │←──────────────────────  │
  │                         │←─────────────────────────
  │                         │                       │  │
  │                    Collect 2f+1                │  │
  │                    commits                     │  │
  │                         │                       │  │
  │←── Reply ───────────────                       │  │
  │←─────────────────────────────────────────────────  (Replicas also reply)
```

**Phase descriptions:**

**1. Pre-Prepare**: Primary assigns sequence number and broadcasts
- Primary receives client request
- Assigns sequence number n in current view v
- Broadcasts `<PRE-PREPARE, v, n, m>` to all replicas
- Includes cryptographic signature

**2. Prepare**: Replicas broadcast prepare messages
- Replica verifies pre-prepare is valid
- Broadcasts `<PREPARE, v, n, digest(m), i>` to all replicas
- Collects 2f prepare messages (including own)
- Ensures 2f + 1 replicas agree on order

**3. Commit**: Replicas commit and execute
- Replica broadcasts `<COMMIT, v, n, digest(m), i>` to all
- Waits for 2f + 1 commit messages
- Executes request
- Sends reply to client

**Client waits for f + 1 matching replies** (ensures at least one honest replica).

#### Why Three Phases?

**Pre-prepare + Prepare**: Ensures all honest replicas agree on order within a view

**Commit**: Ensures order persists across view changes

**Layman explanation:** The three phases are like a verification chain:
1. Teacher announces homework (pre-prepare): "Problem 5 is due tomorrow"
2. Students confirm to each other (prepare): "Yes, I heard problem 5 tomorrow"
3. Students commit publicly (commit): "I've written it in my planner"

Only after hearing confirmation from enough classmates do you trust the assignment is real.

#### View Changes (Leader Replacement)

When primary fails or is suspected to be Byzantine:

```
VIEW CHANGE PROTOCOL:

1. Replica times out waiting for primary
   Broadcasts VIEW-CHANGE message

2. New primary collects 2f + 1 VIEW-CHANGE messages

3. New primary broadcasts NEW-VIEW message
   - Includes VIEW-CHANGE messages as proof
   - Includes pre-prepare messages to continue

4. Replicas verify NEW-VIEW and resume
```

**Why view changes are complex**: Must ensure safety across views—committed operations must not be lost.

### BFT in Blockchain Systems

Modern blockchain systems use BFT consensus:

#### Proof-of-Work (Bitcoin)

Not traditional BFT, but achieves Byzantine fault tolerance through computational work:
- Miners solve cryptographic puzzles
- Longest chain wins
- Tolerates up to ~25% malicious mining power (in practice)
- Very slow (10 minute blocks) but works at internet scale

#### Proof-of-Stake BFT (Ethereum, Cosmos)

Modern blockchains use BFT variants with Proof-of-Stake:
- **Tendermint**: BFT consensus for Cosmos
- **Casper FFG**: Ethereum's finality gadget
- Validators stake cryptocurrency
- Byzantine validators lose stake (slashing)
- Faster than Proof-of-Work (~5-10 second finality)

#### BFT for Permissioned Blockchains

Enterprise blockchains (Hyperledger Fabric) use PBFT variants:
- Known set of participants
- No cryptocurrency needed
- Much faster than public blockchains
- Traditional 3f + 1 requirement

### When BFT is Needed

**Use BFT when:**
- Cannot trust all participants
- Nodes might be compromised or malicious
- Cryptoeconomic incentives exist (blockchain)
- Operating in adversarial environment
- Regulatory requirements for tamper-resistance

**Don't use BFT when:**
- All nodes trusted (enterprise systems)
- Crash failures sufficient model
- Performance critical (BFT is 3-10x slower)
- Complexity unwarranted

**Cost of BFT**:
- Need 3f + 1 replicas (vs. 2f + 1 for crash tolerance)
- More message rounds
- Cryptographic overhead (signatures)
- Complex protocols
- Harder to implement correctly

**Layman explanation:** BFT is like having security cameras and multiple witnesses for every transaction—necessary in a bank vault where guards might be corrupt, but overkill for a private family safe where everyone trusts each other.

---

## Leader Election Patterns

Leader election is a fundamental building block for distributed systems. A **leader** (also called coordinator or primary) is a single node that coordinates actions for the group.

### Why Leaders?

**Benefits of having a leader:**
- **Simplifies coordination**: One node makes decisions
- **Avoids conflicts**: Single writer prevents concurrent update conflicts
- **Improves performance**: No need for expensive multi-node coordination
- **Enables optimizations**: Leader can pipeline operations, batch requests

**Challenges**:
- Single point of failure
- Must detect leader failure
- Must elect new leader when current fails
- Must prevent split-brain (multiple leaders)

```
┌─────────────────────────────────────────────┐
│        LEADER-BASED ARCHITECTURE            │
│                                             │
│            Client                           │
│              ↓                              │
│         ┌─────────┐                         │
│         │ LEADER  │ (coordinates)           │
│         └─────────┘                         │
│            ↓   ↓   ↓                        │
│        ┌───┴───┴───┴───┐                    │
│        ↓   ↓   ↓   ↓   ↓                    │
│      [F] [F] [F] [F] [F]                    │
│    Followers replicate                      │
│                                             │
└─────────────────────────────────────────────┘
```

### Bully Algorithm

One of the simplest leader election algorithms. Each node has a unique ID (e.g., 1, 2, 3, 4, 5). The node with **highest ID** becomes leader.

**Algorithm flow:**

```
SCENARIO: Node 3 detects leader (Node 5) has failed

Step 1: Node 3 sends ELECTION message to all higher-ID nodes
        Node 3 → Node 4: "ELECTION"
        Node 3 → Node 5: "ELECTION" (no response, it's dead)

Step 2: Node 4 receives ELECTION
        Node 4 → Node 3: "OK, I'm taking over"
        Node 4 starts its own election:
        Node 4 → Node 5: "ELECTION" (no response)

Step 3: Node 4 waits for responses from higher nodes
        (None exist or none respond)

Step 4: Node 4 declares itself leader
        Node 4 → All: "COORDINATOR (I'm the new leader)"

Step 5: Other nodes accept Node 4 as leader
```

**Complete algorithm steps:**

1. **Detect failure**: Node notices leader isn't responding
2. **Start election**: Send ELECTION to all nodes with higher IDs
3. **Two outcomes**:
   - **If any higher node responds**: Give up, wait for new leader announcement
   - **If no response from higher nodes**: Declare self as leader
4. **Announce**: Winner sends COORDINATOR message to all nodes

**Why "Bully"?**: Higher ID nodes "bully" lower ID nodes out of leadership.

**Handling simultaneous elections:**

```
Node 2 and Node 3 both detect failure:

Node 2: Sends ELECTION to 3, 4, 5
Node 3: Sends ELECTION to 4, 5

Node 3 responds OK to Node 2 (higher ID)
Node 4 responds OK to both Node 2 and Node 3

Node 4: Sends ELECTION to 5 (no response)
Node 4: Declares itself leader
```

**Advantages**:
- Simple to understand and implement
- Eventually converges to highest-ID node as leader
- Deterministic outcome

**Disadvantages**:
- High message complexity: O(n²) in worst case
- Always elects highest ID (may not be optimal—might be slow or overloaded)
- Multiple simultaneous elections can cause message storms
- Requires reliable failure detection

**Layman explanation:** It's like a group of kids where the oldest always gets to be "it" in a game. If the oldest kid leaves, the next-oldest takes over. When a younger kid notices the leader is gone, they ask all older kids "Will you be leader?" If an older kid responds, the younger backs off.

### Ring-Based Election

Nodes are logically arranged in a ring. Election messages travel around the ring.

**Algorithm (Chang and Roberts):**

```
RING TOPOLOGY:

    [1] → [2] → [3] → [4] → [5]
     ↑                         ↓
     └─────────────────────────┘

Election process:

1. Node 3 detects leader failure
   Sends ELECTION(3) to next node

2. Node 4 receives ELECTION(3)
   Adds own ID: ELECTION(3,4)
   Forwards to Node 5

3. Node 5 receives ELECTION(3,4)
   Adds own ID: ELECTION(3,4,5)
   Forwards to Node 1

4. Node 1 receives ELECTION(3,4,5)
   Adds own ID: ELECTION(3,4,5,1)
   Forwards to Node 2

5. Node 2 receives ELECTION(3,4,5,1)
   Adds own ID: ELECTION(3,4,5,1,2)
   Forwards to Node 3

6. Node 3 receives message with all IDs
   Finds highest ID: 5
   Sends COORDINATOR(5) around ring

7. All nodes accept Node 5 as leader
```

**Optimized version**: Include only highest ID seen so far

```
1. Node 3 starts: ELECTION(3)
2. Node 4 receives: max(3,4)=4 → ELECTION(4)
3. Node 5 receives: max(4,5)=5 → ELECTION(5)
4. Node 1 receives: max(5,1)=5 → ELECTION(5)
5. Node 2 receives: max(5,2)=5 → ELECTION(5)
6. Node 3 receives ELECTION(5)
   Message made full circle!
   Node 5 is winner
   Send COORDINATOR(5)
```

**Handling concurrent elections**: Multiple nodes can start elections simultaneously—algorithm still converges to highest ID.

**Advantages**:
- O(n) messages (much better than Bully's O(n²))
- Every node learns about election
- Works with only unidirectional links

**Disadvantages**:
- Slow: must traverse entire ring
- Vulnerable to single link failure (breaks ring)
- May elect node that's far away or overloaded

**Layman explanation:** Like passing a note around a classroom in a circle. Each student adds their name if their grade is higher than any names already on the note. When the note comes back to where it started, whoever has the highest grade becomes class president.

### Lease-Based Leadership

Instead of permanent leadership, the leader holds a **lease** (time-limited leadership) that must be periodically renewed.

**Key concepts:**

**Lease**: A time-bound grant of leadership
- Leader holds lease for duration T
- Must renew before expiration
- Followers won't accept commands from expired leader

```
LEASE TIMELINE:

Leader's perspective:
├─────────────────┬─────────────────┬──────────
│  Lease 1        │  Lease 2        │  Lease 3
│  (60 seconds)   │  (60 seconds)   │
├─────────────────┴─────────────────┴──────────
0s                60s               120s

Renewal at:       50s               110s
(before expiration)
```

**Lease algorithm:**

1. **Acquisition**: Node acquires lease through consensus (Raft election, Paxos, etc.)

2. **Duration**: Lease granted for duration T (e.g., 60 seconds)

3. **Renewal**: Leader renews lease before expiration
   - Sends heartbeats to followers
   - Followers acknowledge
   - Majority acknowledgment → lease extended

4. **Expiration**: If leader fails to renew:
   - Lease expires
   - Followers start new election
   - New leader acquires new lease

**Clock synchronization challenges:**

```
CLOCK SKEW PROBLEM:

Leader's clock:     |────────────────────────| 60s lease
Follower's clock:   |──────────────────────────| (clock is fast)

Follower might think lease expired when leader thinks it's still valid!

SOLUTION: Use shorter lease duration on leader side:
- Leader grants 60s lease to followers
- Leader considers lease valid for only 50s (safety margin)
- Leader renews at 40s
```

**Advantages**:
- **Automatic failover**: No explicit failure detection needed—lease expiration triggers election
- **Bounded split-brain**: Old leader can't act after lease expires
- **Simplifies reasoning**: Clear time bounds on leadership
- **Safe with clock skew**: Using appropriate safety margins

**Disadvantages**:
- Requires bounded clock drift (loose time synchronization)
- Lease renewals add overhead
- Conservative lease duration increases failover time
- Too short lease → frequent renewals; too long → slow failover

**Typical lease durations**:
- Datacenter: 10-60 seconds
- WAN: 30-120 seconds
- Depends on failure detection requirements

**Layman explanation:** Like renting an apartment month-to-month. You're the "leader" (tenant) for a specific period. If you don't renew before the lease expires, someone else can take over. This prevents situations where you disappear but still claim you live there.

### Leadership Transfer

**Leadership transfer** is the process of gracefully moving leadership from one node to another without downtime or election overhead.

**Use cases:**
- Planned maintenance (shutting down leader node)
- Load balancing (current leader overloaded)
- Geo-optimization (move leader closer to clients)
- Software upgrades (rolling upgrade without elections)

**Transfer protocol (Raft-style):**

```
GRACEFUL LEADERSHIP TRANSFER:

Current Leader (L)           Target (T)           Other Followers
      │                          │                      │
      │                          │                      │
1.    │────── Sync logs ─────────→                      │
      │                          │                      │
      │     (Ensure T is up-to-date)                    │
      │                          │                      │
2.    │                          │                      │
      │  Stop accepting          │                      │
      │  new requests            │                      │
      │                          │                      │
3.    │── TimeoutNow ───────────→                       │
      │                          │                      │
      │               T starts election immediately     │
      │                     with current term + 1       │
      │                          │                      │
4.    │                          │─── RequestVote ─────→│
      │←─── RequestVote ─────────                       │
      │                          │                      │
      │  Votes for T             │←──── Vote ──────────│
      │  (knows transfer         │                      │
      │   in progress)           │                      │
      │                          │                      │
5.    │                      T becomes leader           │
      │                          │                      │
      │                          │──── Heartbeat ──────→│
      │←──── Heartbeat ──────────                       │
      │                          │                      │
      │  Becomes follower        │                      │
```

**Steps in detail:**

1. **Ensure target is up-to-date**: Current leader sends all log entries to target

2. **Stop accepting new requests**: Current leader stops accepting client requests (optional, depends on implementation)

3. **Trigger immediate election**: Send `TimeoutNow` message to target, causing it to immediately start election

4. **Vote for target**: Current leader votes for target (ensures target wins)

5. **Target becomes leader**: Target wins election, starts sending heartbeats

**Advantages**:
- Zero downtime (clients might see brief latency increase)
- Deterministic outcome (specific node becomes leader)
- No election randomness or potential split votes
- Faster than waiting for timeout

**Disadvantages**:
- Requires current leader to be functional
- Target must be reachable and up-to-date
- If transfer fails, must fall back to regular election

**Layman explanation:** Like a manager passing the baton to their replacement in a relay race. The current manager makes sure the new manager is fully briefed (caught up on all work), then explicitly hands over control instead of waiting for everyone to notice the first manager left.

### Split-Brain Prevention

**Split-brain** occurs when multiple nodes believe they are the leader simultaneously, leading to data corruption or inconsistency.

```
SPLIT-BRAIN SCENARIO:

Original topology:
[Leader A] ←→ [Follower B] ←→ [Follower C]

Network partition:
[Leader A]  │  [Follower B] ←→ [Follower C]
            │
     Partition!

After partition:
[Leader A]  │  [Leader B]  ←→  [Follower C]
(thinks it's│   (elected new leader)
 still      │
 leader)    │

Both leaders accept writes → Data divergence!
```

**Prevention mechanisms:**

#### 1. Majority Quorum

Require majority (> n/2) of nodes to acknowledge leadership:
- In partition, only one side can have majority
- Side without majority cannot elect leader or commit operations
- Guarantees at most one leader

```
5-node cluster partitioned:

Partition 1: [A, B]           (2 nodes - no majority)
Partition 2: [C, D, E]        (3 nodes - has majority)

Only Partition 2 can elect leader and accept writes
Partition 1 becomes read-only or unavailable
```

#### 2. Lease-Based Fencing

Use leases with fencing tokens:
- Each leadership term has monotonically increasing token
- Storage systems reject operations with lower token
- Old leader cannot commit after lease expires

```
Leader with token 5 holds lease until T=100

At T=90, network partition occurs

At T=101, lease expires
New leader elected with token 6

Old leader tries to write at T=110:
  Storage: "Token 5 < 6, rejected!"
```

#### 3. Witness Nodes

Place tiebreaker node in third location:
- Three availability zones: A, B, C
- Cluster nodes in A and B
- Witness node in C (doesn't store data)
- Witness participates in elections but not data replication

```
Normal:
  Zone A: [Leader]         ←→
  Zone B: [Follower-1]     ←→
  Zone C: [Witness]        ←→

Zone A partition:
  Zone A: [Leader] (isolated - no quorum)
  Zone B: [New-Leader] ←→ [Witness] (has quorum!)
```

#### 4. Generation Numbers

Each leadership term has generation number:
- Monotonically increasing
- Persisted to stable storage
- Higher generation always wins

**Example**:
```
Leader A (gen=5) gets partitioned
Cluster elects Leader B (gen=6)
Network heals
Leader A tries to send commands with gen=5
Followers reject: "Current generation is 6"
Leader A steps down
```

#### 5. External Coordination

Use external service for coordination:
- ZooKeeper, etcd, or Consul for leader election
- Distributed lock with fencing token
- Only holder of lock can act as leader

**Advantages**:
- Proven, tested implementations
- Handles split-brain automatically
- Can support multiple client applications

**Disadvantages**:
- Additional dependency
- Single point of failure (though coordination service itself is distributed)

**Layman explanation:** Split-brain prevention is like preventing two people from both thinking they're the CEO. Solutions include: (a) requiring majority board approval (quorum), (b) having term-limited authority that expires (leases), (c) using ID badges with increasing numbers (generation tokens), or (d) having an external registry of who's currently CEO (coordination service).

**Best practices**:
- Always use majority quorum for safety
- Combine with leases for bounded split-brain duration
- Use generation numbers for recovery
- Test partition scenarios regularly (chaos engineering)
- Monitor for signs of split-brain (conflicting leaders, divergent data)

---

## Quorum-Based Systems

A **quorum** is the minimum number of nodes that must participate in an operation for it to be valid. Quorum-based systems achieve consistency without requiring all nodes to agree.

### Core Concepts

**Basic idea**: If a read quorum and write quorum overlap, readers will see at least one up-to-date value.

```
N = Total replicas
W = Write quorum (replicas that must acknowledge write)
R = Read quorum (replicas that must respond to read)

CONSISTENCY REQUIREMENT:
R + W > N

This ensures read and write quorums overlap
```

**Layman explanation:** Imagine a secret message copied into 5 notebooks. To write a new message, you must update at least 3 notebooks (W=3). To read the message, you must check at least 3 notebooks (R=3). Since 3+3 > 5, when you read, you're guaranteed to see at least one notebook with the latest message.

### Read and Write Quorums

**Quorum configurations for N=5 replicas:**

```
Configuration A: R=3, W=3  (R+W=6 > N=5)
- Read from 3 replicas
- Write to 3 replicas
- Balanced for mixed read/write workload

Configuration B: R=2, W=4  (R+W=6 > N=5)
- Read from 2 replicas (fast reads)
- Write to 4 replicas (slower writes)
- Optimized for read-heavy workload

Configuration C: R=4, W=2  (R+W=6 > N=5)
- Read from 4 replicas (slower reads)
- Write to 2 replicas (fast writes)
- Optimized for write-heavy workload

Configuration D: R=1, W=5  (R+W=6 > N=5)
- Read from 1 replica (fastest reads)
- Write to all 5 replicas (slowest writes)
- Extreme read optimization
```

**Write operation flow:**

```
CLIENT WRITE REQUEST:

Client → Coordinator: WRITE(key=x, value=5)
    ↓
Coordinator sends to 5 replicas
    ↓
┌────────┬────────┬────────┬────────┬────────┐
│ Rep 1  │ Rep 2  │ Rep 3  │ Rep 4  │ Rep 5  │
│  ACK   │  ACK   │  ACK   │  slow  │ failed │
└────────┴────────┴────────┴────────┴────────┘
    ↓        ↓        ↓
    └────────┴────────┘
          ↓
    W=3 achieved!
    Success returned to client

Replicas 4 and 5 will eventually catch up
```

**Read operation flow:**

```
CLIENT READ REQUEST:

Client → Coordinator: READ(key=x)
    ↓
Coordinator sends to 5 replicas
    ↓
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ Rep 1   │ Rep 2   │ Rep 3   │ Rep 4   │ Rep 5   │
│ v=5,t=10│ v=5,t=10│ v=3,t=8 │  slow   │ failed  │
└─────────┴─────────┴─────────┴─────────┴─────────┘
    ↓         ↓         ↓
    └─────────┴─────────┘
          ↓
    R=3 achieved!
    Return latest: v=5 (timestamp 10)
```

**Versioning and conflict resolution:**

Each value has a version (timestamp, vector clock, or version number):
- Read returns multiple versions
- Client or coordinator selects latest version
- May need to repair stale replicas (read-repair)

### Quorum Intersection

The mathematical guarantee of consistency comes from quorum intersection.

```
QUORUM INTERSECTION PROOF:

N = 5 replicas
W = 3 (write quorum)
R = 3 (read quorum)

Write to replicas: {1, 2, 3}
Read from replicas: {3, 4, 5}

Intersection: {3}
↓
Replica 3 has the latest value
Reader will see latest value!

Visual:
Write set:   {1, 2, [3]}
Read set:         [3], 4, 5}
Overlap:          [3]  ← Guaranteed by R+W > N
```

**Why R + W > N ensures consistency:**

Proof by contradiction:
- Assume R + W ≤ N
- Write succeeds with W replicas
- Read could access R replicas, none in write set
- R + W ≤ N means read set and write set might not overlap
- Reader could miss latest write
- Therefore: R + W > N is necessary

**Trade-offs in quorum selection:**

```
╔═══════════════════════════════════════════════╗
║  QUORUM TRADE-OFFS                            ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  High W (e.g., W=4, R=2):                     ║
║  ✓ Fast reads                                 ║
║  ✗ Slow writes                                ║
║  ✗ Less write availability                    ║
║                                               ║
║  High R (e.g., W=2, R=4):                     ║
║  ✓ Fast writes                                ║
║  ✗ Slow reads                                 ║
║  ✗ Less read availability                     ║
║                                               ║
║  Balanced (e.g., W=3, R=3):                   ║
║  ≈ Moderate read/write performance            ║
║  ≈ Moderate availability                      ║
║                                               ║
║  Majority (W = R = ⌈N/2⌉ + 1):               ║
║  ✓ Symmetric                                  ║
║  ✓ Best for leader election                   ║
║  ≈ Moderate everything                        ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

### Sloppy Quorums

**Sloppy quorum** relaxes the strict quorum requirement to improve availability during failures.

**Standard (strict) quorum**:
- Must write to W of the N designated replicas
- If insufficient designated replicas available, write fails
- Prioritizes consistency over availability

**Sloppy quorum**:
- If designated replicas unavailable, write to other replicas temporarily
- Later, transfer data back to designated replicas (hinted handoff)
- Prioritizes availability over strict consistency

```
STRICT QUORUM FAILURE:

Designated replicas for key "X": {A, B, C, D, E}
W = 3

Replicas available: {A, B}  (C, D, E are down)
Result: WRITE FAILS (can't reach W=3)

────────────────────────────────────────────────

SLOPPY QUORUM:

Designated replicas for key "X": {A, B, C, D, E}
W = 3

Replicas available: {A, B, F, G, H}
(C, D, E are down; F, G, H are replicas for other keys)

Result: WRITE SUCCEEDS
  → Write to A, B (designated)
  → Write to F (temporary substitute for C)
  → Mark F's copy as "hint" for C
```

**Why "sloppy"?**
- Read quorum might not overlap with write quorum if temporary replicas were used
- Weaker consistency guarantees
- Eventually consistent after hinted handoff completes

**When to use sloppy quorums:**
- Availability is paramount (e.g., shopping cart, session data)
- Can tolerate temporary inconsistency
- System experiences frequent partial failures
- Acceptable to trade consistency for availability

**When NOT to use:**
- Need strong consistency (banking, inventory)
- Cannot tolerate any divergence
- Critical operations requiring linearizability

### Hinted Handoff

**Hinted handoff** is the mechanism for returning data to designated replicas after temporary replicas were used.

**Process:**

```
HINTED HANDOFF LIFECYCLE:

Step 1: Temporary Write (during failure)
  Key X normally stored on: {A, B, C}
  C is down
  Write goes to {A, B, F} (F is temporary)
  F stores: (key=X, value=..., hint="belongs on C")

Step 2: Detection (C comes back online)
  F detects C is available again
  OR periodic check finds C is healthy

Step 3: Handoff (transfer data)
  F → C: "Here's data that belongs to you"
  C stores the data

Step 4: Cleanup (remove hint)
  F deletes the hinted data
  F: "Successfully handed off to C"

Step 5: Normal Operation
  Key X now on designated replicas: {A, B, C}
```

**Implementation details:**

**Hint metadata:**
- Original key
- Value and version
- Designated replica ID
- Timestamp of temporary write

**Transfer mechanism:**
- Background process periodically checks hints
- Transfers data when designated replica available
- May batch multiple hints for efficiency
- Uses anti-entropy to verify completion

**Failure handling:**
- If designated replica never comes back: hints may be discarded after timeout
- If transfer fails: retry with backoff
- If temporary replica fails: hints lost (acceptable for availability-first systems)

**Layman explanation:** Hinted handoff is like accepting a package for your neighbor when they're not home. You hold onto it temporarily (with a note: "This is for next door"), and when you see them return, you bring it over. This way, deliveries don't fail just because someone isn't home.

### Practical Examples

#### Amazon Dynamo

Amazon Dynamo uses sloppy quorums with hinted handoff:
- Typical config: N=3, R=2, W=2
- Sloppy quorums enabled for availability
- Eventual consistency model
- Vector clocks for conflict resolution

**Use case**: Shopping cart
- Availability critical (customer must add to cart)
- Temporary inconsistency acceptable
- Conflicts resolved by merging carts

#### Apache Cassandra

Cassandra provides tunable consistency:
```
Consistency Levels:

ONE:   R=1 or W=1 (fastest, least consistent)
TWO:   R=2 or W=2
THREE: R=3 or W=3
QUORUM: R or W = ⌈N/2⌉ + 1 (most common)
ALL:   R=N or W=N (slowest, most consistent)

Special:
LOCAL_QUORUM: Quorum within local datacenter
EACH_QUORUM: Quorum in each datacenter
```

**Per-operation tuning**:
```
// Fast write, relaxed consistency
INSERT INTO users VALUES (...)
  USING CONSISTENCY ONE;

// Important read, strong consistency
SELECT * FROM transactions
  WHERE id = 123
  USING CONSISTENCY QUORUM;
```

#### Riak

Riak uses sloppy quorums by default:
- N=3, R=2, W=2 typical
- Active anti-entropy for repair
- Allows tuning: PR (primary reads), PW (primary writes) for stricter consistency

### Quorum Best Practices

**1. Choose appropriate N, R, W:**
- Start with N=3, R=2, W=2 (good default)
- Increase N for better durability and read performance
- Adjust R and W based on workload (read-heavy vs write-heavy)

**2. Consider failure scenarios:**
- With N=3, W=2: Can tolerate 1 write failure
- With N=5, W=3: Can tolerate 2 write failures
- Formula: tolerate N - W failures for writes

**3. Monitor quorum health:**
- Track successful vs failed operations
- Alert when quorums frequently not achieved
- Monitor hinted handoff queue size

**4. Use appropriate consistency levels:**
- Critical operations: Use stricter quorums
- Background/analytics: Use relaxed quorums
- Tune per operation type

**5. Plan for network partitions:**
- Sloppy quorums for availability
- Strict quorums for consistency
- Consider multi-datacenter quorums (LOCAL_QUORUM)

**6. Implement read-repair:**
- When read finds inconsistent replicas, repair them
- Reduces divergence over time
- Can be async to not impact read latency

---

## Conclusion

Consensus algorithms are the foundation of reliable distributed systems. Each algorithm makes different trade-offs:

### Algorithm Comparison

```
┌─────────────────────────────────────────────────────────────┐
│              CONSENSUS ALGORITHM COMPARISON                 │
├──────────┬──────────┬─────────────┬──────────┬─────────────┤
│Algorithm │Complexity│Performance  │Fault     │Best For     │
│          │          │             │Tolerance │             │
├──────────┼──────────┼─────────────┼──────────┼─────────────┤
│Raft      │Moderate  │Good         │Crash     │General      │
│          │          │             │failures  │purpose      │
├──────────┼──────────┼─────────────┼──────────┼─────────────┤
│Paxos     │High      │Good         │Crash     │Theoretical  │
│          │          │             │failures  │foundation   │
├──────────┼──────────┼─────────────┼──────────┼─────────────┤
│EPaxos    │Very High │Excellent    │Crash     │Geo-         │
│          │          │(geo-dist)   │failures  │distributed  │
├──────────┼──────────┼─────────────┼──────────┼─────────────┤
│PBFT      │Very High │Moderate     │Byzantine │Untrusted    │
│          │          │             │failures  │nodes        │
├──────────┼──────────┼─────────────┼──────────┼─────────────┤
│Quorum    │Low       │Tunable      │Crash     │Tunable      │
│          │          │             │failures  │consistency  │
└──────────┴──────────┴─────────────┴──────────┴─────────────┘
```

### Choosing the Right Algorithm

**Use Raft when:**
- Building a new distributed system from scratch
- Want well-documented, understandable algorithm
- Strong consistency required
- Single datacenter or low-latency network
- Examples: etcd, Consul, CockroachDB

**Use Paxos when:**
- Building on theoretical foundations
- Implementing custom consensus protocol
- Need to understand consensus deeply
- Examples: Google Chubby, Spanner

**Use EPaxos when:**
- Geo-distributed deployment across continents
- Low-conflict workload
- Willing to manage additional complexity
- Need optimal latency from multiple regions

**Use BFT (PBFT) when:**
- Cannot trust all participants
- Blockchain or cryptocurrency
- Regulatory requirements for tamper-resistance
- Can afford 3f+1 replica overhead
- Examples: Hyperledger Fabric, some blockchain platforms

**Use Quorum-based when:**
- Need tunable consistency/availability trade-offs
- Different operations have different consistency needs
- Large-scale distributed databases
- Can tolerate eventual consistency
- Examples: Cassandra, Riak, DynamoDB

**Use Leader Election patterns when:**
- Simple coordination needed
- Not building full consensus system
- Need to designate single coordinator
- Can tolerate brief unavailability during failover

### Key Takeaways

1. **There's no perfect consensus algorithm**: Each has trade-offs between performance, complexity, fault tolerance, and consistency.

2. **Majority quorums are fundamental**: Most algorithms require > N/2 nodes for safety (preventing split-brain).

3. **Network partitions are inevitable**: Algorithms must handle partitions gracefully.

4. **Understand your failure model**: Crash failures vs Byzantine failures require very different approaches.

5. **CAP theorem applies**: During partitions, choose between consistency and availability.

6. **Complexity has a cost**: Byzantine fault tolerance and leaderless consensus are powerful but complex—use only when needed.

7. **Testing is critical**: Consensus bugs are subtle. Use chaos engineering, formal verification, and extensive testing.

8. **Build on proven systems**: Use battle-tested implementations (etcd, ZooKeeper, Consul) rather than building from scratch.

The field of consensus algorithms continues to evolve, with new algorithms addressing specific challenges like geo-distribution (EPaxos), Byzantine faults in permissioned settings, and performance optimizations. Understanding these fundamentals enables you to choose and deploy the right consensus mechanism for your distributed system.

---

*Document created: 2026*
*Topic: Consensus Algorithms*