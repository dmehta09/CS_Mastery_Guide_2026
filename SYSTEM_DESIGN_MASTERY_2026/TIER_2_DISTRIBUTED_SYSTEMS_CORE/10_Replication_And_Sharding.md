# Replication & Sharding: A Comprehensive Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Replication Topologies](#replication-topologies)
3. [Synchronization Mechanisms](#synchronization-mechanisms)
4. [Read Replicas](#read-replicas)
5. [Database Sharding Strategies](#database-sharding-strategies)
6. [Consistent Hashing](#consistent-hashing)
7. [Rebalancing Strategies](#rebalancing-strategies)
8. [Cross-Partition Operations](#cross-partition-operations)

---

## Introduction

**Replication** and **sharding** are two fundamental techniques for scaling databases and distributed systems. They solve different but complementary problems:

- **Replication** means keeping copies of the same data on multiple machines. This improves **availability** (if one machine fails, others can serve requests), **fault tolerance** (data isn't lost when hardware fails), and **read performance** (queries can be distributed across multiple copies).

- **Sharding** (also called **partitioning**) means dividing data across multiple machines, where each machine holds a different subset. This improves **write performance** and enables handling datasets too large for a single machine.

Think of replication like having backup copies of a book in different libraries (same content, multiple locations), while sharding is like splitting an encyclopedia across volumes (different content in each location).

Modern large-scale systems typically use both: data is sharded for horizontal scaling, and each shard is replicated for reliability.

---

## Replication Topologies

Replication topology refers to the architecture that defines how data flows between different copies (replicas) of your database. The choice of topology profoundly impacts consistency, availability, performance, and operational complexity.

### Single-Leader Replication (Primary-Backup / Master-Slave)

In single-leader replication, one replica is designated as the **leader** (also called primary or master), while all others are **followers** (also called replicas, secondaries, or slaves).

**How it works:**
1. All write operations go to the leader
2. The leader writes to its local storage
3. The leader sends changes to all followers (via a replication log)
4. Followers apply changes in the same order
5. Reads can come from leader or followers

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
                    Writes │  Reads
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  Leader  │───>│ Follower │    │ Follower │
    │ (Primary)│───>│    1     │    │    2     │
    └──────────┘    └──────────┘    └──────────┘
         │
         └─> Replication Log (changes flow one direction)
```

**Advantages:**
- Simple to understand and implement
- Strong consistency guarantees (all writes serialized through one point)
- Easy to reason about data flow
- No write conflicts (single source of truth)

**Disadvantages:**
- Leader is a single point of failure for writes
- Leader can become a bottleneck under high write load
- Geographic distribution challenges (far-away clients have high latency to leader)

**Common use cases:** Traditional relational databases (MySQL, PostgreSQL), MongoDB, message queues, file systems.

---

### Multi-Leader Replication (Active-Active / Master-Master)

In multi-leader replication, multiple replicas can accept writes simultaneously. Each leader acts independently and exchanges updates with other leaders.

**How it works:**
1. Multiple nodes can accept writes
2. Each leader applies writes locally
3. Leaders asynchronously replicate changes to each other
4. Conflict resolution handles simultaneous edits

```
         Client A                Client B
             │                       │
             │ Writes                │ Writes
             ▼                       ▼
      ┌──────────┐            ┌──────────┐
      │ Leader 1 │<──────────>│ Leader 2 │
      │ (US East)│  Bi-directional  │ (EU West)│
      └────┬─────┘   Replication    └────┬─────┘
           │                              │
           ▼                              ▼
      ┌──────────┐                   ┌──────────┐
      │ Follower │                   │ Follower │
      └──────────┘                   └──────────┘
```

**Advantages:**
- Better write performance (writes distributed across leaders)
- Lower latency for geographically distributed users (write to nearby leader)
- Continued operation even if one leader fails
- No single point of failure for writes

**Disadvantages:**
- **Write conflicts** are inevitable (two leaders may modify same data simultaneously)
- Complex conflict resolution logic required
- Harder to maintain consistency
- More complex operational model

**Conflict resolution strategies:**
- Last-write-wins (using timestamps, but clock synchronization is hard)
- Application-level merge logic
- Conflict-free replicated data types (CRDTs)
- Custom resolution handlers

**Common use cases:**
- Multi-datacenter deployments (each datacenter has a leader)
- Offline-capable applications (each device is a leader)
- Collaborative editing systems
- CouchDB, Cassandra (when configured this way)

---

### Leaderless Replication (Dynamo-Style)

In leaderless replication, there is no distinguished leader node. Clients send writes to multiple replicas simultaneously, and reads query multiple replicas to ensure freshness.

**How it works:**
1. Client sends write to N replicas in parallel
2. Write succeeds when W replicas acknowledge (W = write quorum)
3. For reads, client queries R replicas (R = read quorum)
4. Client resolves any version conflicts
5. As long as W + R > N, you'll read the latest write

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
              Parallel writes/reads to multiple nodes
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ Node A  │<──────>│ Node B  │<──────>│ Node C  │
   │(Replica)│  Gossip │(Replica)│  Gossip │(Replica)│
   └─────────┘  Protocol └─────────┘        └─────────┘
        ▲                                        │
        └────────────────────────────────────────┘
                   Background sync
```

**Key concepts:**

- **Quorum:** A minimum number of nodes that must participate in a read/write operation
  - N = total replicas
  - W = write quorum (nodes that must acknowledge a write)
  - R = read quorum (nodes queried for a read)
  - Common setting: N=3, W=2, R=2

- **Read repair:** When reading reveals inconsistencies, client writes back the latest value to stale replicas

- **Anti-entropy:** Background process where replicas compare data and sync differences

**Advantages:**
- High availability (no leader to fail)
- Excellent for geographically distributed systems
- Tunable consistency vs. availability tradeoff
- Continues operating despite node failures

**Disadvantages:**
- Complex conflict resolution on the client side
- Eventual consistency (not always strongly consistent)
- Higher overhead (multiple round trips per operation)
- Version conflicts require application logic to resolve

**Common use cases:** Amazon DynamoDB, Apache Cassandra, Riak, Voldemort

---

### Chain Replication

Chain replication is a specialized topology where replicas are arranged in a linear chain. Writes flow through the chain from head to tail, while reads come from the tail.

**How it works:**
1. Write request goes to the **head** of the chain
2. Head processes write and forwards to next replica
3. Each replica in turn processes and forwards
4. **Tail** acknowledges back to client once write completes
5. Read requests go only to the tail

```
  Client Write ──────> ┌──────┐      ┌──────┐      ┌──────┐
                       │ Head │─────>│Middle│─────>│ Tail │
  Client Read <─────── │Node 1│      │Node 2│      │Node 3│
                       └──────┘      └──────┘      └──────┘
                           │            │            │
                           └────────────┴────────────┘
                              Write flows forward
                           Acknowledgment flows back
```

**Key properties:**

- **Writes:** Sequential through chain, acknowledged by tail
- **Reads:** Only from tail (which has all committed writes)
- **Consistency:** Strong consistency for reads from tail
- **Fault tolerance:** If middle node fails, chain is reconnected; if head/tail fails, new head/tail is promoted

**Advantages:**
- Simple strong consistency model
- High read throughput (tail can handle all reads)
- Clear ordering of operations
- Good for workloads with many reads, fewer writes

**Disadvantages:**
- Write latency increases with chain length (must traverse entire chain)
- Write throughput limited by slowest node in chain
- Head node can be a bottleneck for writes
- More complex failure recovery than single-leader

**When to use:** Distributed databases prioritizing availability (Cassandra, DynamoDB, Riak), systems spanning multiple datacenters, highly available systems.

---

## Read Replicas

**Read replicas** are copies of your database that serve read queries, offloading the primary database and improving read scalability. They're a cornerstone of scaling read-heavy applications like social media, e-commerce sites, and content platforms.

### Lag and Staleness

**Replication lag** is the delay between when data is written to the leader and when it appears on a follower. This creates **staleness** - the state where a follower serves outdated data.

**What causes lag:**
- Network latency between leader and follower
- Follower processing speed (disk I/O, CPU)
- High write volume on leader
- Long-running transactions or large bulk writes
- Geographic distance between datacenters

```
    Time →

    Leader:    [Write X] ────────────────────────> [Current State]
                   │
                   │ Replication Lag (2 seconds)
                   │
    Follower:      └──────────> [Write X Applied] ──> [Current State]

    During lag window, follower returns old data
```

**Measuring lag:**
- Time difference: Compare follower's last-applied transaction timestamp to leader's current time
- Log position: Track how far behind follower is in the replication log (e.g., "100 commits behind")

**Impact on applications:**
- User sees their own write missing (just updated profile, still see old data)
- Inconsistent views (refresh page, see old data again)
- Violates causality (reply appears before the message it's replying to)

**Mitigations:**
- Monitoring and alerting on lag metrics
- Faster hardware for followers
- Read from leader for critical queries
- Application-level consistency mechanisms (covered below)

---

### Read-After-Write Consistency

**Read-after-write consistency** (also called **read-your-writes consistency**) guarantees that users always see their own updates immediately, even if other users' updates may lag.

**The problem it solves:**

```
    Timeline:

    t1: User posts comment on leader
    t2: User refreshes page, reads from lagging follower
    t3: User doesn't see their own comment! (frustrating experience)
```

**Implementation strategies:**

**Strategy 1: Read from leader for user's own data**
- After a user writes, route their subsequent reads to the leader (not followers)
- Time-based: Read from leader for 1 minute after user's last write
- Data-based: Read from leader when accessing data the user recently modified

**Strategy 2: Track client's last write timestamp**
- Client remembers timestamp of their last write
- When reading from follower, client checks follower's replication position
- If follower is behind client's last write, wait or query different follower

```
    User A writes at t=100

    ┌─────────────────────────────────────────┐
    │ Client tracks: last_write_time = 100    │
    └─────────────────────────────────────────┘
              │
              ├──> Follower 1 (position: t=95)  ← Too stale, skip
              │
              └──> Follower 2 (position: t=105) ← Fresh enough, read here
```

**Strategy 3: Session-based routing**
- Assign user session to a specific follower
- All reads in that session go to same follower
- Ensures monotonic reads (see below)

**Challenges:**
- Increases complexity in load balancing
- May reduce read scalability if too many reads forced to leader
- Requires client-side or proxy logic to implement

---

### Monotonic Reads

**Monotonic reads** guarantee that if a user sees data at a certain state, they will never see an older state in subsequent reads. In other words, time doesn't go backward for that user.

**The problem it solves:**

```
    Timeline:

    t1: User reads from Follower 1 (up-to-date) → sees comment #100
    t2: User reads from Follower 2 (lagging) → sees only comment #95
    t3: Comment #100 "disappeared"! (confusing experience)
```

This violates causality and creates a jarring user experience.

**Implementation:**

**Sticky sessions:** Always route the same user to the same replica
- Use consistent hashing on user ID to pick replica
- Load balancer maintains session affinity
- User always sees monotonically increasing state from their chosen replica

```
    User Session ID: alice@example.com
                 │
                 │ Hash(alice) = 0x7F
                 │
                 ├──> Always routes to Follower 2
                 │
    ┌────────────┴────────────┐
    │                         │
    Read 1 → Follower 2 (t=100)
    Read 2 → Follower 2 (t=103)
    Read 3 → Follower 2 (t=107)

    Monotonic: 100 < 103 < 107 ✓
```

**Advantages:**
- Simple to implement (session affinity)
- Predictable user experience
- No time travel paradoxes

**Disadvantages:**
- Load balancing less flexible
- If a user's designated follower fails, must switch (and may see staleness)
- Doesn't solve cross-user consistency

---

### Replica Selection Strategies

Choosing which replica to read from is a critical decision that affects latency, load distribution, and consistency.

**Strategy 1: Round-robin**
- Distribute requests evenly across all replicas
- Simple, fair load distribution
- Doesn't account for lag or geographic proximity

**Strategy 2: Geographic proximity**
- Route to closest replica (lowest network latency)
- Best user experience for latency
- May create hot spots if users concentrated in one region

```
    User in Tokyo          User in New York
         │                      │
         └───> Replica (Tokyo)  └───> Replica (NYC)

         Low latency            Low latency
```

**Strategy 3: Lag-aware routing**
- Monitor replication lag on each replica
- Route to replicas with acceptable lag threshold
- If all replicas too stale, route to leader

```
    ┌──────────────────────────────────┐
    │ Health Monitor                   │
    │                                  │
    │ Follower 1: lag = 0.5s  ✓       │
    │ Follower 2: lag = 10s   ✗       │
    │ Follower 3: lag = 1.2s  ✓       │
    └──────────────────────────────────┘
                 │
                 └──> Route to Follower 1 or 3
```

**Strategy 4: Workload-aware routing**
- Heavy analytical queries → dedicated analytics replica
- Transactional queries → primary or low-lag follower
- Reporting/dashboards → can tolerate more staleness

**Strategy 5: Causal consistency token**
- Leader issues token with last committed transaction ID
- Client includes token in read request
- Replica only serves read if it has applied that transaction
- Guarantees causality (effects seen after causes)

**Hybrid approaches:**

Most production systems combine multiple strategies:
1. Prefer geographically close replica
2. Filter out replicas with excessive lag
3. Apply read-after-write for user's own data
4. Use round-robin among remaining candidates

---

## Database Sharding Strategies

**Sharding** (also called **partitioning**) is the practice of splitting your data across multiple databases or servers, where each **shard** holds a subset of the total data. This is essential for horizontal scaling when data outgrows a single machine.

**Why shard?**
- **Scalability:** Distribute data and query load across multiple machines
- **Performance:** Smaller indexes fit in memory, queries scan less data
- **Geographic distribution:** Place data close to users who need it
- **Cost:** Use many commodity machines instead of one expensive server

**Key terminology:**
- **Shard key:** The attribute used to determine which shard holds a piece of data
- **Shard/partition:** A single division of the dataset
- **Routing layer:** Component that directs queries to the correct shard(s)

### Hash-Based Sharding

Hash-based sharding applies a hash function to the shard key and uses the result to determine which shard stores the data.

**How it works:**
1. Choose a shard key (e.g., user_id, order_id)
2. Apply hash function: `shard = hash(shard_key) % num_shards`
3. Data with that key goes to the computed shard

```
    Data: user_id = 42

    hash(42) = 0x7A3F → 31295
    31295 % 4 shards = 3

    ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ Shard 0  │  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │ ← Store here
    └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

**Advantages:**
- **Uniform distribution:** Good hash functions distribute data evenly
- **Simple to implement:** Just modulo arithmetic
- **No hotspots:** Randomization prevents sequential keys clustering

**Disadvantages:**
- **Range queries are inefficient:** Users 1-100 scattered across all shards
- **Rebalancing is expensive:** Adding/removing shards changes hash(key) % N for most keys, requiring massive data movement
- **No semantic grouping:** Related data may end up on different shards

**When to use:**
- Simple key-value lookups (get user by ID)
- Workloads without range queries
- Relatively stable cluster size (infrequent shard additions)

**Common implementations:** Memcached, Redis Cluster (with hash slots), early Cassandra

---

### Range-Based Sharding

Range-based sharding divides data into contiguous ranges based on the shard key's natural ordering.

**How it works:**
1. Define ranges for shard key values
2. Each shard handles a specific range
3. Route data based on which range it falls into

```
    Shard Key: user_id (integer)

    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │  Shard 0    │  │  Shard 1    │  │  Shard 2    │
    │ user_id     │  │ user_id     │  │ user_id     │
    │ 1 - 100000  │  │100001-200000│  │200001-300000│
    └─────────────┘  └─────────────┘  └─────────────┘

    Query: user_id = 50000 → Shard 0
    Query: user_id BETWEEN 150000 AND 250000 → Shards 1 & 2
```

**Advantages:**
- **Efficient range queries:** Queries for consecutive keys hit few/single shard
- **Predictable data location:** Easy to reason about where data lives
- **Semantic grouping:** Related data naturally clusters together

**Disadvantages:**
- **Hotspots:** If keys aren't uniformly accessed (e.g., recent users more active), some shards overloaded
- **Skewed data distribution:** If key distribution isn't uniform, shards have uneven sizes
- **Manual rebalancing:** Requires monitoring and splitting/merging ranges

**Preventing hotspots:**
- Add randomization prefix to keys
- Split hot ranges into smaller shards
- Use composite shard key (e.g., `date + user_id`)

**When to use:**
- Time-series data (shard by timestamp)
- Sequential IDs with range query patterns
- Data with natural ordering (alphabetical names, dates, geographic coordinates)

**Common implementations:** Google Bigtable, HBase, older MongoDB versions

---

### Directory-Based Sharding

Directory-based sharding uses a lookup table (directory) that explicitly maps each shard key to a specific shard.

**How it works:**
1. Maintain a directory/index: `{key → shard_id}`
2. For each query, consult directory to find correct shard
3. Route query to that shard

```
    Directory Service:
    ┌───────────────────────────┐
    │ user_id  │  shard_id      │
    ├──────────┼────────────────┤
    │ 42       │  shard_2       │
    │ 100      │  shard_1       │
    │ 555      │  shard_3       │
    │ ...      │  ...           │
    └───────────────────────────┘
              │
              ├──> Lookup user_id=42 → shard_2
              │
    ┌─────────┴──────┐  ┌──────────┐  ┌──────────┐
    │   Shard 1      │  │ Shard 2  │  │ Shard 3  │
    └────────────────┘  └─────┬────┘  └──────────┘
                              │
                         Route here
```

**Advantages:**
- **Flexible assignment:** Can place data on shards based on any criteria
- **Easy rebalancing:** Just update directory, move data, no rehashing
- **Controlled placement:** Manually optimize for workload patterns
- **Supports complex sharding logic:** Different rules for different data types

**Disadvantages:**
- **Directory is a bottleneck:** Every query requires directory lookup
- **Directory is a single point of failure:** Must be highly available
- **Extra latency:** Added hop to consult directory
- **Operational complexity:** Directory must be maintained and kept consistent

**Optimizations:**
- Cache directory entries at application layer
- Replicate directory for availability
- Use hierarchical directories (reduce lookup size)

**When to use:**
- Irregular data distribution that hash/range can't handle
- Need fine-grained control over data placement
- Frequent rebalancing requirements
- Multi-tenant systems (assign each tenant to specific shard)

**Common implementations:** Some NoSQL systems, custom sharded MySQL deployments, multi-tenant SaaS architectures

---

### Composite Sharding

Composite sharding combines multiple sharding strategies or uses multiple attributes as the shard key.

**Approach 1: Hierarchical sharding**

Shard first by one attribute, then sub-shard by another:

```
    Step 1: Shard by region        Step 2: Shard by user_id hash

    ┌────────────┐                  ┌──────────┐
    │   Region   │                  │ US-West  │
    │            │                  │          │
    │  ┌──────┐  │                  │ Shard 0  │ hash(user) % 4 = 0
    │  │ US   │──┼──────────────────>│ Shard 1  │ hash(user) % 4 = 1
    │  └──────┘  │                  │ Shard 2  │ hash(user) % 4 = 2
    │  ┌──────┐  │                  │ Shard 3  │ hash(user) % 4 = 3
    │  │ EU   │──┼──────────────────>└──────────┘
    │  └──────┘  │
    └────────────┘
```

**Approach 2: Compound shard key**

Use multiple fields together: `shard_key = (tenant_id, user_id)`

**Example:** Multi-tenant SaaS application
- First level: Shard by `tenant_id` (keeps all tenant data together)
- Second level: Within tenant, sub-shard by `user_id` (balances large tenants)

**Advantages:**
- **Addresses multiple access patterns:** Can support both single-tenant and cross-tenant queries efficiently
- **Hierarchical organization:** Reflects natural data relationships
- **Better locality:** Related data stays together while still distributing load
- **Prevents single-tenant hotspots:** Large tenants don't overwhelm one shard

**Disadvantages:**
- Increased complexity in routing logic
- Harder to rebalance (must consider multiple dimensions)
- Queries not matching shard key require scatter-gather

**When to use:**
- Multi-tenant applications
- Data with multiple natural access patterns
- Systems with hierarchical data relationships

---

### Geographic Sharding

Geographic sharding (also called **geo-partitioning** or **location-based sharding**) places data based on geographic location, keeping data close to the users who access it most.

**How it works:**

Data is partitioned by geographic region (country, continent, city):

```
    ┌──────────────────────────────────────────────────────┐
    │                  User Request                        │
    │          "I'm in Germany, fetch my data"             │
    └───────────────────────┬──────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
         ┌────────┐    ┌────────┐    ┌────────┐
         │   US   │    │   EU   │    │  Asia  │
         │ Shard  │    │ Shard  │    │ Shard  │
         │(Oregon)│    │(Dublin)│    │(Tokyo) │
         └────────┘    └────────┘    └────────┘
                           │
                      Route here (low latency)
```

**Shard key:** Geographic identifier (country code, region, lat/long range)

**Advantages:**
- **Low latency:** Data physically close to users
- **Compliance:** Satisfies data residency laws (GDPR, data localization)
- **Disaster recovery:** Geographic distribution provides resilience
- **Reduced cross-region traffic:** Most queries stay local

**Disadvantages:**
- **Uneven distribution:** Some regions have more data/users than others
- **Cross-region queries expensive:** Global queries must hit multiple shards
- **Difficult to rebalance:** Moving data across continents is slow/expensive
- **User mobility:** Users traveling may experience higher latency

**Handling edge cases:**
- Users who travel: Use session location or allow selecting preferred region
- Global entities: Replicate across regions or designate primary region
- Multi-region queries: Accept higher latency or maintain separate aggregated views

**When to use:**
- Applications with strong geographic user patterns (social networks, ride-sharing)
- Regulatory requirements for data locality
- Global applications prioritizing local latency

**Common implementations:** Google Spanner (geo-replication), CockroachDB (geo-partitioning), Vitess (geographic sharding)

---

## Consistent Hashing

**Consistent hashing** is an elegant technique for distributing data across nodes in a way that minimizes reorganization when nodes are added or removed. It's fundamental to building scalable distributed systems.

**The problem with naive hashing:**

Using simple modulo hashing (`shard = hash(key) % N`), adding or removing even one node requires remapping almost all keys:

```
    Original: 4 nodes             After adding 1 node:
    hash(key) % 4                 hash(key) % 5

    Key X: hash = 7               Key X: hash = 7
    7 % 4 = 3 → Node 3            7 % 5 = 2 → Node 2  ← MOVED!

    Result: ~80% of keys need to move when cluster size changes
```

This is catastrophic for caches (cache stampede) and databases (massive data migration).

### Basic Consistent Hashing

Consistent hashing maps both nodes and keys onto a circular hash ring, then assigns each key to the first node encountered going clockwise.

**How it works:**

1. Hash node IDs onto a ring (0 to 2^32 - 1)
2. Hash data keys onto the same ring
3. Walk clockwise from key position to find responsible node

```
                        0 (2^32)
                          │
                    ┌─────┴─────┐
                    │           │
              Node C │           │ Node A
           (hash=3B) │           │ (hash=0F)
                    │           │
         ┌──────────┘           └──────────┐
         │        HASH RING                │
         │   (circular number space)       │
         │                                 │
    Key X ●────┐                      ┌────● Key Y
  (hash=45)    │                      │  (hash=12)
         │     │                      │    │
         │     └──> Clockwise ──> Node A  │
         │                                 │
         └──> Clockwise ──> Node D         │
                         (hash=5C)          │
                                           └──> Node A

    Assignments:
    - Key X (45) → Node D (first node clockwise)
    - Key Y (12) → Node A (first node clockwise)
```

**When adding a node:**

Only keys between the new node and its predecessor move. Most keys stay put!

```
    Before:                      After adding Node E (hash=25):

         Node A (0F)                   Node A (0F)
              │                              │
         [Keys 0F-3B]                   [Keys 0F-25]
              │                              │
         Node C (3B)                   Node E (25) ← NEW
              │                              │
         [Keys 3B-5C]                   [Keys 25-3B]
              │                              │
         Node D (5C)                   Node C (3B)
                                             │
                                        [Keys 3B-5C]
                                             │
                                        Node D (5C)

    Only keys in range [25-3B] move from Node C to Node E
    (~25% of keys move instead of 80%)
```

**Advantages:**
- **Minimal reorganization:** On average, only K/N keys move when adding/removing nodes (K = total keys, N = nodes)
- **Scalability:** Easy to add/remove capacity
- **Load balancing:** With good hash function, load distributes evenly

**Disadvantages:**
- **Uneven distribution:** Random hash positions can create imbalanced partitions
- **Hotspots:** Popular keys still create hotspots
- **Cascading failures:** If one node fails, its load goes entirely to successor

---

### Virtual Nodes (VNodes)

Virtual nodes solve the uneven distribution problem by having each physical node appear at multiple positions on the hash ring.

**How it works:**

Instead of hashing the node ID once, hash it many times with different tokens:

```
    Physical Node A appears as:
    - vnode_A_1 (hash = 0x0012)
    - vnode_A_2 (hash = 0x4F89)
    - vnode_A_3 (hash = 0x9A43)
    - vnode_A_4 (hash = 0xE2B7)

                        Ring:

           vnode_A_1 ●     ● vnode_B_2

     vnode_C_3 ●                   ● vnode_A_2

                 vnode_B_1 ●   ● vnode_C_1

           vnode_A_4 ●     ● vnode_B_3

                    ● vnode_C_2
```

Each physical node owns multiple virtual positions (typically 100-500 vnodes per node).

**Advantages:**
- **Even distribution:** More positions = better averaging, fewer random imbalances
- **Granular balancing:** Can move individual vnodes between nodes
- **Flexible capacity:** Give powerful nodes more vnodes
- **Smooth scaling:** When adding node, it receives vnodes from many existing nodes (not just one)

**Disadvantages:**
- **Metadata overhead:** Must track many more partitions
- **Increased bookkeeping:** More complex routing tables
- **Slightly slower lookups:** Must check more positions

**Typical configuration:**
- 256 virtual nodes per physical node (Cassandra default)
- Distribute vnodes evenly around ring

**When to use:** Virtually always preferred over basic consistent hashing in production systems.

**Common implementations:** Amazon Dynamo, Apache Cassandra, Riak

---

### Jump Consistent Hash

Jump consistent hash is a fast, memory-efficient alternative to ring-based consistent hashing that computes shard assignments directly using only the key and number of buckets.

**How it works:**

Uses a mathematical function that "jumps" through buckets, ensuring minimal movement when buckets change:

```
    Pseudocode concept:

    function jump_hash(key, num_buckets):
        b = -1
        j = 0
        while j < num_buckets:
            b = j
            key = next_random(key)  # deterministic random
            j = floor((b + 1) / random_fraction(key))
        return b
```

**Properties:**
- **No ring structure:** No need to store node positions
- **Deterministic:** Same key always maps to same bucket
- **Minimal movement:** When increasing from N to N+1 buckets, only 1/N keys move on average
- **Fast:** O(log N) time complexity, very efficient

**Advantages:**
- **Zero memory overhead:** No hash ring to store
- **Blazing fast:** Much faster than ring lookups
- **Optimal redistribution:** Mathematically proven minimal key movement
- **Simple implementation:** Just a small function

**Disadvantages:**
- **Cannot remove buckets easily:** Only supports adding buckets (N increases)
- **All buckets must be numbered 0 to N-1:** Can't have arbitrary bucket IDs
- **Less flexible:** Can't weight nodes differently or use complex topologies

**When to use:**
- Caching layers (Google's jump hash used in many Google services)
- Stateless services where you can drain before removing nodes
- Systems that primarily grow, rarely shrink
- Performance-critical routing with many lookups

**Common implementations:** Google Maglev load balancer, various CDN systems, memcached proxies

---

### Rendezvous Hashing (Highest Random Weight - HRW)

Rendezvous hashing assigns each key to the node with the highest hash score for that key. Each node is scored independently for each key.

**How it works:**

1. For a given key, compute hash(key, node) for each node
2. Assign key to node with highest hash value

```
    Key = "user_42"

    Scoring:
    hash("user_42", "Node A") = 0x4B2F
    hash("user_42", "Node B") = 0x9A81 ← Highest
    hash("user_42", "Node C") = 0x2E17
    hash("user_42", "Node D") = 0x7C44

    → Assign "user_42" to Node B

    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ Node A  │  │ Node B  │  │ Node C  │  │ Node D  │
    │ 0x4B2F  │  │ 0x9A81  │  │ 0x2E17  │  │ 0x7C44  │
    └─────────┘  └────┬────┘  └─────────┘  └─────────┘
                      │
                   Winner! Stores key
```

**When a node is removed:**

Simply recompute without that node. Only keys assigned to the removed node move (to the new highest scorer).

**Advantages:**
- **Minimal redistribution:** Only 1/N keys move when adding/removing a node
- **No ring structure needed:** Simpler than consistent hashing
- **Easy to weight nodes:** Multiply hash score by node capacity
- **Deterministic:** Same result from any client, no coordination needed

**Disadvantages:**
- **Computational overhead:** Must compute hash for all nodes per key (O(N) time)
- **Not suitable for large clusters:** Becomes slow with thousands of nodes
- **No locality:** Can't optimize for data center or rack placement easily

**Optimizations:**
- Pre-compute hashes for common keys
- Use subset of candidate nodes for large clusters
- Cache assignments

**When to use:**
- Small to medium clusters (dozens to hundreds of nodes)
- CDN edge server selection
- Load balancers distributing connections
- Distributed caches with heterogeneous node sizes

**Common implementations:** Content delivery networks, Microsoft's Azure Storage, some research systems

---

### Bounded-Load Consistent Hashing

Bounded-load consistent hashing extends consistent hashing to prevent any node from becoming overloaded, even with skewed workloads or uneven key distributions.

**The problem:**

Standard consistent hashing can create overloaded nodes:
- Popular keys create hotspots
- Uneven vnode distribution
- Some nodes more powerful than others

**Solution:**

Set a maximum load threshold per node. If a node would exceed threshold, assign key to next node on ring.

**How it works:**

1. Define: `max_load = avg_load × (1 + ε)` where ε is load factor (e.g., 0.25)
2. When assigning key, walk clockwise on ring
3. Skip nodes already at max_load
4. Assign to first node below threshold

```
    Ring with load limits:

         Node A                    avg_load = 1000 keys/node
     (current: 1200)              max_load = 1250 keys/node
     [AT MAX] ✗
           │
           │ Skip (overloaded)
           ▼
         Node B                    Key X arrives
     (current: 900)                    │
     [BELOW MAX] ✓ ←──────────────────┘
           │                        Assigned here
           │
         Node C
     (current: 1100)
     [BELOW MAX] ✓
```

**Parameters:**
- **ε (epsilon):** Load imbalance factor
  - ε = 0: Perfectly balanced (but may reject keys if impossible)
  - ε = 0.25: Nodes can be up to 25% above average
  - Higher ε = more flexible but less balanced

**Advantages:**
- **Prevents cascading failures:** No node overwhelmed by hotspots
- **Graceful degradation:** System remains balanced under skewed workloads
- **Capacity-aware:** Can set different limits for different node sizes
- **Maintains consistent hashing benefits:** Still minimal movement on node changes

**Disadvantages:**
- **More complex routing:** Must track node loads
- **Potential reassignments:** Keys may occasionally move for balancing
- **Coordination required:** Nodes must share load information

**When to use:**
- Production clusters with unpredictable workloads
- Systems with heterogeneous node capacities
- Preventing cache hotspots in CDNs
- Any consistent hashing use case where load balancing is critical

**Common implementations:** Google Maglev (with enhancements), research systems, custom distributed caches

---

## Rebalancing Strategies

**Rebalancing** is the process of moving data between shards to maintain even distribution as the cluster grows, shrinks, or as workload patterns change. Good rebalancing minimizes disruption while ensuring balanced load.

### Why Rebalancing is Needed

Systems require rebalancing when:
- Adding nodes to increase capacity
- Removing failed nodes
- Data growth creates skewed shard sizes
- Query patterns create hotspots
- Hardware heterogeneity (different node capacities)

**Goals of rebalancing:**
- Distribute data and query load evenly
- Minimize data movement (reduces network usage and service disruption)
- Maintain availability during rebalancing
- Complete quickly without overwhelming the system

---

### Fixed Number of Partitions

Create many more partitions than nodes at the outset, then assign multiple partitions to each node. When rebalancing, move entire partitions between nodes.

**Setup:**

```
    Cluster: 4 nodes, 16 partitions

    Initial distribution:
    ┌──────────────────────┐  ┌──────────────────────┐
    │ Node A               │  │ Node B               │
    │ Partitions: 0,1,2,3  │  │ Partitions: 4,5,6,7  │
    └──────────────────────┘  └──────────────────────┘
    ┌──────────────────────┐  ┌──────────────────────┐
    │ Node C               │  │ Node D               │
    │ Partitions: 8,9,10,11│  │ Partitions:12,13,14,15│
    └──────────────────────┘  └──────────────────────┘

    Each node holds 4 partitions
```

**When adding Node E:**

Move some partitions from existing nodes to new node:

```
    After rebalancing:
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ Node A           │ │ Node B           │ │ Node C           │
    │ Partitions: 0,1,2│ │ Partitions: 5,6,7│ │ Partitions: 8,9  │
    └──────────────────┘ └──────────────────┘ └──────────────────┘
    ┌──────────────────┐ ┌──────────────────────────┐
    │ Node D           │ │ Node E (NEW)             │
    │ Partitions:12,13 │ │ Partitions: 3,4,10,11,14,15│
    └──────────────────┘ └──────────────────────────┘

    Moved: Partitions 3,4,10,11,14,15 to Node E
    Result: More even distribution (2-4 partitions per node)
```

**Choosing the right number of partitions:**

Too few: Limited rebalancing granularity, uneven load
Too many: Overhead in managing metadata, smaller partition size reduces efficiency

**Rule of thumb:** 10-100 partitions per node initially, plan for growth

**Advantages:**
- **Simple rebalancing:** Just move whole partitions
- **Predictable overhead:** Know exactly how much data moves
- **No rehashing:** Partition boundaries fixed forever
- **Granular control:** Can move partitions one at a time

**Disadvantages:**
- **Partitions fixed at creation:** Wrong initial choice hard to correct later
- **Difficult to optimize:** Partition size increases as data grows
- **May not adapt well:** If data growth dramatically exceeds predictions

**When to use:**
- Systems with predictable growth
- Databases where partition management overhead is acceptable
- Riak, Elasticsearch, Cassandra (with fixed token ranges)

---

### Dynamic Partitioning

Partitions are created and destroyed dynamically as data size changes. When a partition grows too large, it splits; when too small, it merges.

**How it works:**

**Splitting:** When partition exceeds size threshold (e.g., 10 GB)

```
    Before:                           After split:
    ┌────────────────────┐           ┌──────────┐  ┌──────────┐
    │   Partition A      │           │ Part A1  │  │ Part A2  │
    │   Size: 12 GB      │  ──────>  │ 0-5000   │  │5001-10000│
    │   Range: 0-10000   │   Split   │ Size: 6GB│  │ Size: 6GB│
    └────────────────────┘           └──────────┘  └──────────┘
                                           │             │
                                      (May live on different nodes)
```

**Merging:** When partition shrinks below threshold (e.g., 2 GB), merge with neighbor

```
    Before:                          After merge:
    ┌──────────┐  ┌──────────┐      ┌─────────────────┐
    │ Part B1  │  │ Part B2  │      │  Partition B    │
    │ 0-5000   │  │5001-10000│ ───> │  0-10000        │
    │Size: 1GB │  │Size: 1.5GB│Merge │  Size: 2.5 GB   │
    └──────────┘  └──────────┘      └─────────────────┘
```

**Advantages:**
- **Adapts to data size:** Partitions naturally match data volume
- **No upfront capacity planning:** System self-organizes
- **Efficient storage:** Each partition stays within optimal size range
- **Responds to hotspots:** Hot partitions split, spreading load

**Disadvantages:**
- **Unpredictable partition count:** Number of partitions changes over time
- **Split/merge overhead:** Rebalancing operations consume resources
- **Complexity:** Must implement split/merge logic correctly
- **Potential for oscillation:** Partition near threshold may split and merge repeatedly

**Configuration parameters:**
- Split threshold: 10-50 GB per partition (varies by system)
- Merge threshold: Usually half of split threshold
- Hysteresis: Add delay before merging to prevent oscillation

**When to use:**
- Unpredictable data growth patterns
- Systems where data distribution changes significantly over time
- HBase, MongoDB (with auto-splitting)

---

### Proportional to Nodes

Make the number of partitions proportional to the number of nodes. Each node gets a fixed number of partitions.

**How it works:**

Maintain ratio like 10 partitions per node:
- 4 nodes → 40 partitions
- Add 1 node → Create 10 new partitions, move data from existing partitions

```
    4 nodes, 40 partitions (10 each):

    ┌──────────────┐  ┌──────────────┐
    │ Node A       │  │ Node B       │
    │ Part 0-9     │  │ Part 10-19   │
    └──────────────┘  └──────────────┘
    ┌──────────────┐  ┌──────────────┐
    │ Node C       │  │ Node D       │
    │ Part 20-29   │  │ Part 30-39   │
    └──────────────┘  └──────────────┘

    Add Node E → Create partitions 40-49:

    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ Node A       │  │ Node B       │  │ Node C       │
    │ Part 0-9     │  │ Part 10-19   │  │ Part 20-29   │
    └──────────────┘  └──────────────┘  └──────────────┘
    ┌──────────────┐  ┌──────────────┐
    │ Node D       │  │ Node E (NEW) │
    │ Part 30-39   │  │ Part 40-49   │
    └──────────────┘  └──────────────┘

    Then split existing partitions to populate new ones
```

**Advantages:**
- **Scales with cluster:** Partition count grows/shrinks with nodes
- **Consistent partition size:** Each partition manages similar data volume
- **Flexible:** Combines benefits of fixed and dynamic approaches

**Disadvantages:**
- **Rebalancing complexity:** Must redistribute existing data when adding nodes
- **Overhead:** Creating new partitions requires data movement
- **Challenging with hash-based sharding:** May require rehashing keys

**When to use:**
- Systems with frequent scaling operations
- Clusters that grow/shrink regularly with demand
- Cassandra vnodes (automatically adjusts), Kafka (partitions per broker)

---

### Minimizing Data Movement

Regardless of strategy, minimizing data movement during rebalancing is crucial to maintain performance and availability.

**Techniques to minimize movement:**

**1. Pre-splitting:**
Create more partitions than immediately needed, so adding nodes just requires partition assignment (not data splitting)

**2. Gradual rebalancing:**
Move partitions one at a time with throttling to avoid overwhelming network/disks

```
    Rebalancing schedule:

    Hour 1:  Move partition 5  ────> Node E
    Hour 2:  Move partition 12 ────> Node E
    Hour 3:  Move partition 23 ────> Node E

    Instead of moving all at once
```

**3. Incremental copy with dual writes:**
- Start copying partition to new node
- During copy, write to both old and new location
- Switch reads to new location once copy complete
- Remove old copy

**4. Consistent hashing / vnodes:**
Use techniques designed for minimal movement (as discussed earlier)

**5. Avoid rehashing entire dataset:**
Never use `hash(key) % N` where N changes frequently

**Monitoring rebalancing:**
- Data transfer rate (GB/hour)
- Impact on query latency
- CPU/disk utilization on involved nodes
- Estimated time to completion

**When to prioritize minimal movement:**
- Large datasets (terabytes to petabytes)
- Systems with tight SLAs
- Network-constrained environments
- 24/7 availability requirements

---

## Cross-Partition Operations

When data is sharded across multiple nodes, operations that need to access data across multiple shards become significantly more complex. This is one of the primary challenges of distributed databases.

### Scatter-Gather Queries

**Scatter-gather** is a pattern where a query must be sent to multiple (or all) shards, with results gathered and combined at the coordinator.

**How it works:**

```
    Client Query: "SELECT * FROM users WHERE age > 30"
    (age is not the shard key)

                    ┌──────────────┐
                    │  Coordinator │
                    │   (Router)   │
                    └───────┬──────┘
                            │
              Scatter to all shards
           ┌────────┬───────┴────────┬────────┐
           │        │                │        │
           ▼        ▼                ▼        ▼
      ┌────────┐ ┌────────┐    ┌────────┐ ┌────────┐
      │Shard 0 │ │Shard 1 │    │Shard 2 │ │Shard 3 │
      │users   │ │users   │    │users   │ │users   │
      │1-25K   │ │25-50K  │    │50-75K  │ │75-100K │
      └───┬────┘ └───┬────┘    └───┬────┘ └───┬────┘
          │          │              │          │
       [20 rows] [15 rows]      [18 rows] [12 rows]
          │          │              │          │
          └──────────┴──────────────┴──────────┘
                            │
                         Gather
                            │
                            ▼
                    Merge, sort, limit
                            │
                            ▼
                   Return to client
```

**Challenges:**

**1. Performance:**
- Latency: Slowest shard determines total query time
- Network overhead: Must transfer all intermediate results
- CPU cost: Coordinator must merge results

**2. Correctness:**
- Aggregations (SUM, AVG, COUNT): Must combine correctly
- Sorting/limiting: Can't just take top N from each shard
- DISTINCT: Must deduplicate across all results

**Optimization techniques:**

**Push computation to shards:**
Instead of transferring all rows, compute partial results:

```
    Query: SELECT COUNT(*), AVG(age) WHERE age > 30

    Each shard returns: {count: X, sum_age: Y, count_age: Z}
    Coordinator computes:
        total_count = sum(count)
        global_avg = sum(sum_age) / sum(count_age)
```

**Early filtering:**
Apply WHERE clauses at shard level to reduce data transfer

**Limit pushdown:**
For LIMIT queries, fetch LIMIT × N rows from each shard, then take top LIMIT

**Parallel execution:**
Query all shards simultaneously, don't serialize

**When scatter-gather is acceptable:**
- Analytics queries (batch processing, can tolerate high latency)
- Admin operations (occasional, user not waiting)
- Small result sets (few shards need to be queried)

**When to avoid:**
- High-frequency transactional queries
- Real-time user-facing queries
- Large result sets

---

### Secondary Indexes

Secondary indexes allow queries on non-shard-key attributes, but sharding makes them challenging.

**The problem:**

```
    Shard by user_id:

    Shard 0: users 1-1000
    Shard 1: users 1001-2000
    Shard 2: users 2001-3000

    Query: "Find users where email = 'alice@example.com'"

    Problem: email could be on ANY shard!
```

**Two approaches:** Local secondary indexes vs. global secondary indexes

---

#### Local Secondary Indexes (Document-Partitioned)

Each shard maintains its own independent index covering only the data on that shard.

**Structure:**

```
    Shard 0 (users 1-1000):
    ┌─────────────────────────────────┐
    │ Primary Data                    │
    │ user_id=5, email=alice@ex.com   │
    │ user_id=42, email=bob@ex.com    │
    └─────────────────────────────────┘
    ┌─────────────────────────────────┐
    │ Local Index on email            │
    │ alice@ex.com → user_id 5        │
    │ bob@ex.com → user_id 42         │
    └─────────────────────────────────┘

    Shard 1 (users 1001-2000):
    ┌─────────────────────────────────┐
    │ Primary Data                    │
    │ user_id=1234, email=carol@ex.com│
    └─────────────────────────────────┘
    ┌─────────────────────────────────┐
    │ Local Index on email            │
    │ carol@ex.com → user_id 1234     │
    └─────────────────────────────────┘
```

**Querying with local indexes:**

Query must scatter to all shards and gather results:

```
    Query: "SELECT * FROM users WHERE email = 'alice@example.com'"

    1. Send query to all shards
    2. Each shard checks its local index
    3. Most return empty, one returns matching user
    4. Coordinator returns result
```

**Advantages:**
- **Fast writes:** Only update local shard's index
- **Simple to implement:** Each shard self-contained
- **No cross-shard coordination:** Writes are local operations

**Disadvantages:**
- **Slow reads:** Every query hits all shards (scatter-gather)
- **No benefit for single-item lookups:** Can't narrow down shard
- **Scales poorly:** More shards = more scatter overhead

**When to use:**
- Write-heavy workloads
- Queries are rare or can tolerate high latency
- Analytical queries that scan large portions anyway
- MongoDB, Cassandra (secondary indexes use this approach)

---

#### Global Secondary Indexes (Term-Partitioned)

A single index spans all shards, partitioned independently by indexed attribute.

**Structure:**

```
    Primary Data sharded by user_id:

    Shard 0: users 1-1000
    Shard 1: users 1001-2000
    Shard 2: users 2001-3000

    Global index partitioned by email:

    Index Shard A (emails a-m):
    ┌──────────────────────────────────────┐
    │ alice@example.com → user_id 5        │
    │ bob@example.com → user_id 42         │
    │ carol@example.com → user_id 1234     │
    └──────────────────────────────────────┘

    Index Shard B (emails n-z):
    ┌──────────────────────────────────────┐
    │ norman@example.com → user_id 2500    │
    │ zoe@example.com → user_id 750        │
    └──────────────────────────────────────┘
```

**Querying with global indexes:**

Query goes to relevant index shard, then fetches from data shards:

```
    Query: "SELECT * FROM users WHERE email = 'alice@example.com'"

    1. Route to Index Shard A (emails a-m)
    2. Index returns: user_id = 5
    3. Route to Shard 0 (users 1-1000)
    4. Fetch user record
    5. Return to client

    Only 2 shards involved (not all shards)
```

**Advantages:**
- **Fast reads:** Single lookup to find exact shard
- **Efficient queries:** Only hit necessary shards
- **Scales better:** Adding shards doesn't slow queries

**Disadvantages:**
- **Slow writes:** Must update both data shard and index shard
- **Cross-shard transaction:** Writing user requires updating two shards
- **Eventual consistency:** Index updates may lag behind data
- **Complex failure handling:** Index and data can get out of sync

**When to use:**
- Read-heavy workloads
- Queries by secondary attributes common
- Can tolerate potential inconsistency
- DynamoDB Global Secondary Indexes, some custom systems

---

### Cross-Shard Transactions

Transactions that modify data across multiple shards are notoriously difficult in distributed systems.

**The problem:**

```
    Transfer $100 from Account A to Account B:

    Account A (on Shard 1):
        balance = $500
    Account B (on Shard 3):
        balance = $300

    Transaction:
        1. Deduct $100 from Account A
        2. Add $100 to Account B

    Challenge: How to ensure both happen or neither?
```

**Without coordination:** Risk of inconsistency
- Shard 1 succeeds, Shard 3 fails → $100 disappears
- Shard 3 succeeds, Shard 1 fails → $100 appears from nowhere

**Most common solution: Two-Phase Commit (2PC)**

---

### Two-Phase Commit Limitations

Two-phase commit is the traditional protocol for distributed transactions, but it has significant limitations.

**How 2PC works:**

**Phase 1 - Prepare:**
1. Coordinator asks all shards: "Can you commit this transaction?"
2. Each shard locks resources and replies YES (prepared) or NO (abort)
3. If all say YES, proceed to Phase 2; if any say NO, abort

**Phase 2 - Commit:**
1. Coordinator sends COMMIT (or ABORT) to all shards
2. Each shard commits (or aborts) and releases locks
3. Shards acknowledge completion

```
    Coordinator                  Shard 1         Shard 3
         │                          │               │
         │────── PREPARE ──────────>│               │
         │────── PREPARE ─────────────────────────> │
         │                          │               │
         │<────── YES ──────────────│               │
         │<────── YES ──────────────────────────────│
         │                          │               │
         │  (All prepared, decide COMMIT)           │
         │                          │               │
         │────── COMMIT ────────────>│               │
         │────── COMMIT ─────────────────────────────>│
         │                          │               │
         │<────── ACK ──────────────│               │
         │<────── ACK ──────────────────────────────│
         │                          │               │
    Transaction Complete
```

**Critical limitations:**

**1. Blocking protocol:**
During Phase 1, shards hold locks. If coordinator crashes, shards are **blocked** indefinitely waiting for commit/abort decision.

**2. Coordinator is single point of failure:**
If coordinator fails between phases, entire system stalls

**3. Performance overhead:**
- Multiple network round trips
- Locks held for extended time
- Reduced throughput and increased latency

**4. Availability impact:**
If any participant is unavailable, transaction cannot complete

**Why it's problematic at scale:**

```
    Probability of failure increases with participants:

    P(failure) = 1 - (1 - P_node_failure)^N

    Example:
    Single node: 99.9% available (0.1% failure)
    2 nodes: 99.8% transaction success
    10 nodes: 99.0% transaction success
    100 nodes: 90.5% transaction success

    More shards = more failures!
```

**Alternatives to 2PC:**

**1. Avoid cross-shard transactions entirely:**
- Design schema so related data lives on same shard
- Use shard key that keeps transactional data together

**2. Application-level compensation:**
- Use saga pattern (sequence of local transactions with compensating actions)
- Accept eventual consistency

**3. Modern consensus protocols:**
- Raft, Paxos for better availability
- Spanner's TrueTime for globally consistent timestamps

**4. Single-partition transactions only:**
- Many systems (Cassandra, MongoDB) only support single-shard ACID
- Application handles coordination

**When 2PC is acceptable:**
- Low-scale systems (few shards)
- Transactions are rare
- Availability requirements are relaxed
- Inside a single datacenter (lower network failure rates)

**Best practice:** Design to avoid cross-shard transactions whenever possible. When unavoidable, use modern distributed transaction protocols or embrace eventual consistency with compensating actions.

---

## Conclusion

Replication and sharding are foundational techniques for building scalable, reliable distributed systems. Key takeaways:

**Replication:**
- Choose topology based on consistency needs: single-leader for simplicity, multi-leader for geo-distribution, leaderless for availability
- Balance synchronous (strong consistency) vs. asynchronous (performance) replication
- Use read replicas strategically with awareness of lag and consistency models

**Sharding:**
- Select sharding strategy based on access patterns: hash for uniform distribution, range for sequential scans, geographic for latency
- Use consistent hashing or vnodes to enable graceful scaling
- Plan for rebalancing from day one

**Cross-partition operations:**
- Avoid when possible through careful schema design
- Use global indexes for read-heavy workloads, local indexes for write-heavy
- Minimize cross-shard transactions; prefer single-partition operations

The art of building distributed systems lies in understanding these tradeoffs and choosing the right combination of techniques for your specific requirements. There is no one-size-fits-all solution—context matters.

---

**Document Version:** 1.0
**Last Updated:** February 2026
**Topics Covered:** Replication topologies, synchronization mechanisms, read replicas, sharding strategies, consistent hashing, rebalancing, cross-partition operations

## Synchronization Mechanisms

Synchronization mechanisms determine **when** the leader considers a write successful and **how quickly** changes propagate to replicas. This is one of the most critical tradeoffs in distributed systems: consistency versus performance.

### Synchronous Replication

In synchronous replication, the leader waits for confirmation from followers before acknowledging the write to the client.

**Process:**
1. Client sends write to leader
2. Leader writes to its own storage
3. Leader sends change to followers
4. Leader **waits** for follower acknowledgments
5. Only after followers confirm does leader acknowledge to client

```
  Client                Leader              Follower
    │                     │                     │
    │──── Write ────────> │                     │
    │                     │─── Replicate ─────> │
    │                     │                     │
    │                     │ <──── ACK ────────┘ │
    │ <── Success ────────│                     │
    │                     │                     │

    Time: Client waits for follower confirmation
```

**Advantages:**
- **Strong durability:** Data guaranteed on multiple nodes before success
- **No replication lag:** Followers always up-to-date (within transaction)
- **Easy failover:** Can promote follower immediately if leader fails

**Disadvantages:**
- **High latency:** Each write waits for network round-trip to followers
- **Reduced availability:** If a follower is slow/down, writes are delayed or fail
- **Performance bottleneck:** Write speed limited by slowest follower

**When to use:** Critical data where durability is paramount (financial transactions, medical records), systems where consistency is more important than speed.

---

### Asynchronous Replication

In asynchronous replication, the leader acknowledges writes to the client immediately without waiting for followers.

**Process:**
1. Client sends write to leader
2. Leader writes to its own storage
3. Leader **immediately** acknowledges to client
4. Leader sends changes to followers in background
5. Followers apply changes whenever they can

```
  Client                Leader              Follower
    │                     │                     │
    │──── Write ────────> │                     │
    │                     │                     │
    │ <── Success ────────│                     │
    │                     │                     │
    │                     │─── Replicate ─────> │
    │                     │                     │
    │                     │ <──── ACK ──────────┘

    Time: Client doesn't wait; replication happens later
```

**Advantages:**
- **Low latency:** Writes complete as fast as leader can process
- **High availability:** Slow/failed followers don't block writes
- **Better performance:** Leader not bottlenecked by follower speed

**Disadvantages:**
- **Data loss risk:** If leader fails before replicating, recent writes are lost
- **Replication lag:** Followers may be seconds/minutes/hours behind
- **Stale reads:** Reading from follower may return old data
- **Complex failover:** Must choose between consistency (lose recent writes) or accepting divergence

**When to use:** High-throughput systems, social media feeds, analytics systems, caching layers, systems where some data loss is acceptable for better performance.

---

### Semi-Synchronous Replication

Semi-synchronous replication is a middle ground: the leader waits for acknowledgment from **at least one** follower, but not all.

**Process:**
1. Client sends write to leader
2. Leader writes to its own storage
3. Leader sends changes to all followers
4. Leader waits for acknowledgment from **one** follower (at minimum)
5. Leader acknowledges to client
6. Other followers replicate asynchronously

```
  Client           Leader         Follower 1      Follower 2
    │                │                 │               │
    │─── Write ─────>│                 │               │
    │                │── Replicate ───>│               │
    │                │── Replicate ────────────────────>│
    │                │<──── ACK ───────┘               │
    │<── Success ────│                                 │
    │                │                           Later  │
    │                │                 <──── ACK ──────┘

    Waits for 1 follower, others replicate asynchronously
```

**Advantages:**
- **Balanced durability:** At least one confirmed copy beyond leader
- **Better performance than fully synchronous:** Not blocked by all followers
- **Reduced data loss:** If leader fails, at least one follower has latest data
- **More available than synchronous:** One slow follower doesn't halt all writes

**Disadvantages:**
- **Still some latency overhead:** Must wait for one follower
- **Partial replication lag:** Some followers may lag behind
- **Complexity in failover:** Must identify which follower is most up-to-date

**When to use:** Production databases needing good balance (MySQL with semi-sync, PostgreSQL streaming replication), systems where some durability is required but full synchronous is too slow.

---

### Quorum-Based Replication

Quorum-based replication formalizes the idea of requiring acknowledgment from a **minimum number** of replicas. This is the foundation of leaderless systems.

**Core principle:**

Given:
- **N** = total number of replicas
- **W** = write quorum (minimum replicas that must acknowledge a write)
- **R** = read quorum (minimum replicas queried for a read)

If **W + R > N**, then reads are guaranteed to see the most recent write (because read and write sets overlap).

**Example configuration:** N=3, W=2, R=2

```
    Write to 3 replicas, need 2 ACKs:

    Client ────> Node A ✓ (ACK)
            ───> Node B ✓ (ACK)  ← Write succeeds (2/3)
            ───> Node C ✗ (slow/failed)

    Read from 3 replicas, need 2 responses:

    Client ────> Node A (returns v1)
            ───> Node B (returns v2)  ← Takes latest (v2)
            ───> Node C (timeout)
```

**Quorum configurations trade off consistency vs. availability:**

- **W=N, R=1:** Strong durability, fast reads, slow writes, low write availability
- **W=1, R=N:** Fast writes, strong read consistency, low read availability
- **W=N/2+1, R=N/2+1:** Balanced (common choice)
- **W=1, R=1:** Maximum availability, eventual consistency only

**Sloppy quorum:** If W nodes are unavailable, accept writes to other nodes temporarily (**hinted handoff**) and transfer later. This prioritizes availability over consistency.

**Advantages:**
- Tunable consistency/availability tradeoff
- Mathematically provable consistency guarantees (when W+R>N)
- Fault tolerant (can lose N-W nodes and still write)
- No single leader bottleneck

**Disadvantages:**
- More complex to implement correctly
- Network partitions can lead to split-brain scenarios
- Version conflicts require resolution
- Higher read/write latency (must contact multiple nodes)

**When to use:** Distributed databases prioritizing availability (Cassandra, DynamoDB, Riak), systems spanning multiple datacenters, highly available systems.

---