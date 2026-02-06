# Sharding Strategies

How to partition databases across multiple servers to handle scale beyond a single machine's capacity.

---

## Case Study 1: Database Hitting Capacity Limits

### Problem

Your database is growing rapidly. It's approaching the maximum capacity of a single server (storage, memory, CPU). Vertical scaling (bigger server) is expensive and has limits. You need to scale horizontally by splitting data across multiple servers.

**Context**: Common scaling challenge - affects any high-growth application

**In simple terms**: Imagine a library that's running out of space. You can't just make the building bigger (vertical scaling is expensive). Instead, you split the books across multiple libraries - A-M in Library 1, N-Z in Library 2. That's sharding: splitting data across multiple databases.

### Quick Answer

Use sharding: partition data by a key (user_id, order_id, etc.) and distribute partitions across multiple database servers. Each shard handles a subset of data. Application routes queries to the correct shard based on the shard key.

### Detailed Explanation

#### Why This Happens (Root Cause)

**Single database limitations:**
- Storage: Can't store more than server capacity
- Memory: Can't fit all indexes in memory
- CPU: Can't process queries fast enough
- Network: Bandwidth limits
- Cost: Larger servers exponentially more expensive

**When to shard:**
- Database size approaching server limits (80%+ capacity)
- Query performance degrading despite optimization
- Vertical scaling becoming cost-prohibitive
- Need geographic distribution

#### The Solution Approach

**High-level strategy:**
- Choose shard key (determines how data is partitioned)
- Partition data by shard key (hash or range-based)
- Distribute partitions across multiple database servers
- Route queries to correct shard based on shard key

**Sharding Strategy 1: Hash-Based Sharding**

**How it works conceptually:**
1. Choose shard key (e.g., `user_id`)
2. Hash the shard key: `hash(user_id) % num_shards`
3. Route to shard based on hash result
4. Example: `hash(user_123) % 4 = 2` → Route to Shard 2

**Benefits:**
- Even data distribution (if hash function is good)
- Simple routing logic
- Works well for uniform access patterns

**Drawbacks:**
- Can't do range queries across shards
- Resharding is complex (need to redistribute data)

**Sharding Strategy 2: Range-Based Sharding**

**How it works conceptually:**
1. Choose shard key (e.g., `user_id`)
2. Define ranges: Shard 1: 1-1000000, Shard 2: 1000001-2000000, etc.
3. Route based on which range the key falls into
4. Example: `user_id = 500000` → Shard 1

**Benefits:**
- Supports range queries within shard
- Easy to understand and debug
- Can add new shards for new ranges

**Drawbacks:**
- Uneven distribution (some ranges more popular)
- Hot shards (one shard gets most traffic)
- Resharding requires splitting ranges

**Architecture Flow:**
```
Query Request
      │
      ▼
┌─────────────────┐
│  Application    │
│  (Shard Router) │
└────────┬────────┘
         │
    Extract shard_key
         │
    Calculate shard_id
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│ Shard 1│ │ Shard 2│
│ (DB 1) │ │ (DB 2) │
└────────┘ └────────┘
```

**Key Design Decisions:**
- **Shard key selection**: Critical - affects distribution and query patterns
- **Number of shards**: Start with few, add more as needed
- **Shard routing**: Application or middleware handles routing
- **Cross-shard queries**: Avoid if possible, use application-level joins

#### Production Results

| Metric | Before (Single DB) | After (4 Shards) | Improvement |
|--------|-------------------|------------------|-------------|
| Storage capacity | 2TB (max) | 8TB (4 × 2TB) | 4x increase |
| Query throughput | 10K QPS | 40K QPS | 4x increase |
| P99 latency | 200ms | 50ms | 4x faster |
| Cost | $X/month | $1.2X/month | 20% increase (acceptable) |

**Key insight**: Sharding enables horizontal scaling beyond single-server limits. The complexity is worth the ability to scale indefinitely.

**Lessons Learned**:
- Choose shard key carefully (affects distribution and queries)
- Start with few shards, add more as needed
- Monitor shard distribution (ensure even load)
- Avoid cross-shard queries when possible
- Plan for resharding (data redistribution is complex)

---

## Case Study 2: Resharding Without Downtime

### Problem

You sharded your database into 4 shards. Now you need to add 4 more shards (total 8) to handle growth. But you can't take the system offline. How do you redistribute data across shards without downtime?

**Context**: Common challenge as systems grow - need to add shards

**In simple terms**: Imagine splitting a library into 4 buildings, then needing to split into 8 buildings. You can't close the library while moving books. You need to move books gradually while the library stays open.

Resharding is the same: redistribute data across new shards while the system continues serving traffic.

### Quick Answer

Use dual-write pattern: write to both old and new shard assignments during transition. Background jobs migrate historical data. Gradually shift reads to new shards. Once complete, stop writing to old shards.

### Detailed Explanation

#### Why This Happens

**Scenario:**
- Started with 4 shards
- Data growing, need 8 shards
- Can't take system offline
- Need to redistribute data

**Challenge:**
- Data currently in 4 shards
- Need to move to 8 shards (new shard mapping)
- System must stay online
- No data loss or inconsistency

#### The Solution Approach

**High-level strategy:**
- Calculate new shard assignments (old and new mapping)
- Dual-write: Write to both old and new shard during transition
- Background migration: Move historical data to new shards
- Gradual read migration: Shift reads to new shards
- Cutover: Stop writing to old shards

**Phase 1: Dual-Write Setup**

**How it works conceptually:**
1. Calculate shard mapping:
   - Old: `hash(user_id) % 4` → Shard 0-3
   - New: `hash(user_id) % 8` → Shard 0-7
2. For each write:
   - Calculate old shard (write here)
   - Calculate new shard (also write here)
   - Both shards have same data
3. Reads still from old shards (stable)

**Phase 2: Historical Data Migration**

**How it works conceptually:**
1. For each record in old shards:
   - Calculate new shard assignment
   - If different from old shard:
     - Copy data to new shard
     - Verify copy succeeded
     - Mark as migrated
2. Continue until all data migrated

**Phase 3: Read Migration**

**How it works conceptually:**
1. Start reading from new shards for low-traffic features
2. Gradually increase percentage of reads from new shards
3. Monitor for issues
4. Once confident, all reads from new shards

**Phase 4: Cutover**

**How it works conceptually:**
1. All reads from new shards
2. Stop writing to old shards
3. Old shards can be decommissioned

#### Production Results

| Metric | Before Resharding | After Resharding | Improvement |
|--------|------------------|------------------|-------------|
| Shard count | 4 | 8 | 2x capacity |
| Storage per shard | 80% full | 40% full | Room to grow |
| Migration downtime | 0 hours | 0 hours | Zero downtime |
| Data consistency | 100% | 100% | Maintained |

**Key insight**: Dual-write pattern enables zero-downtime resharding. The complexity is worth the ability to scale without service interruption.

**Lessons Learned**:
- Dual-write is essential for zero-downtime resharding
- Background migration can take days/weeks - plan accordingly
- Monitor both old and new shards during transition
- Have rollback plan (can revert to old shards if needed)
- Verify data consistency after migration

---

## Case Study 3: Hot Shard Problem

### Problem

You sharded your database into 10 shards using hash-based sharding. Most shards have even load, but one shard (Shard 5) is handling 40% of all traffic. The shard is overloaded, causing slow queries and timeouts.

**Context**: Common issue with hash-based sharding - uneven distribution

**In simple terms**: Imagine splitting a library into 10 buildings by hashing book titles. Most buildings have similar numbers of books, but one building has all the popular bestsellers. That building is overcrowded while others are empty.

Hot shards are the same: one shard gets disproportionate traffic due to uneven data distribution or access patterns.

### Quick Answer

Identify the cause (uneven hash distribution or hot keys). Solutions include: rebalancing data, using consistent hashing, splitting hot shard, or using range-based sharding for known hot ranges.

### Detailed Explanation

#### Why This Happens (Root Cause)

**Causes of hot shards:**
1. **Uneven hash distribution**: Hash function doesn't distribute evenly
2. **Hot keys**: Few keys get most traffic (e.g., popular users, trending content)
3. **Time-based patterns**: Recent data more popular (all goes to one shard if time-based sharding)
4. **Geographic patterns**: Users from one region (all hash to same shard)

**Example:**
```
Shard 1: 5% of traffic
Shard 2: 8% of traffic
Shard 3: 12% of traffic
Shard 4: 10% of traffic
Shard 5: 40% of traffic  ← HOT SHARD
Shard 6: 7% of traffic
...
```

**Root cause**: Hash function or shard key doesn't account for access patterns. Even distribution of data doesn't mean even distribution of traffic.

#### The Solution Approach

**High-level strategy:**
- Identify hot shard and root cause
- Rebalance data (move some data to other shards)
- Split hot shard (create more shards for hot range)
- Use consistent hashing (better distribution)
- Consider different sharding strategy

**Solution 1: Rebalance Data**

**How it works conceptually:**
1. Identify hot shard and which keys are hot
2. Move some hot keys to less-loaded shards
3. Update shard routing logic
4. Monitor load distribution

**Challenges:**
- Requires data migration
- Must update routing logic
- Temporary inconsistency during move

**Solution 2: Split Hot Shard**

**How it works conceptually:**
1. Identify hot shard
2. Split shard into 2-3 smaller shards
3. Redistribute data within split shards
4. Update routing to use new shards

**Example:**
```
Before: Shard 5 (40% load)
After:  Shard 5a (15% load)
        Shard 5b (15% load)
        Shard 5c (10% load)
```

**Solution 3: Consistent Hashing**

**How it works conceptually:**
1. Use consistent hashing instead of modulo
2. Better distribution (fewer collisions)
3. Easier to add/remove shards
4. More even load distribution

**Benefits:**
- Better distribution than simple modulo
- Easier resharding (only affects adjacent shards)
- Handles shard failures better

**Solution 4: Separate Hot Data**

**How it works conceptually:**
1. Identify hot keys (popular users, trending content)
2. Put hot keys in separate shard(s)
3. Regular keys in other shards
4. Scale hot shards independently

**Benefits:**
- Isolates hot traffic
- Can scale hot shards separately
- Doesn't affect regular shards

#### Production Results

| Metric | Before (Hot Shard) | After (Rebalanced) | Improvement |
|--------|-------------------|-------------------|-------------|
| Hot shard load | 40% of traffic | 12% of traffic | 3.3x reduction |
| P99 latency (hot shard) | 2.5s | 150ms | 16x faster |
| Overall system throughput | Limited by hot shard | 3x increase | 3x improvement |
| Shard load variance | 35% | 5% | 7x more even |

**Key insight**: Hot shards are a common problem with hash-based sharding. Monitoring and rebalancing are essential for maintaining performance.

**Lessons Learned**:
- Monitor shard load distribution continuously
- Identify and address hot shards early
- Consider consistent hashing for better distribution
- Separate hot data if access patterns are known
- Plan for rebalancing as part of sharding strategy

---

## Case Study 4: Cross-Shard Queries

### Problem

You need to query data that spans multiple shards. For example, "Get all orders for users in California" requires querying all shards, then joining results. This is very slow and complex.

**Context**: Common challenge with sharding - some queries need data from multiple shards

**In simple terms**: Imagine your library is split across 10 buildings. To find all books by a specific author, you'd need to search all 10 buildings, then combine the results. That's a cross-shard query - slow and complex.

### Quick Answer

Avoid cross-shard queries when possible. Design queries to hit single shard. If cross-shard queries are necessary, use scatter-gather pattern: query all shards in parallel, aggregate results in application.

### Detailed Explanation

#### Why This Happens

**Queries that span shards:**
- Geographic queries: "Users in California" (users sharded by user_id, not location)
- Time-range queries: "Orders in last month" (orders sharded by order_id, not date)
- Join queries: "Orders with product details" (orders and products in different shards)
- Aggregation queries: "Total revenue" (requires all shards)

**Challenge:**
- Must query all shards
- Combine results in application
- Very slow (network latency × number of shards)
- Complex to implement

#### The Solution Approach

**High-level strategy:**
- Design queries to hit single shard (preferred)
- If cross-shard needed, use scatter-gather
- Cache aggregated results
- Consider denormalization (duplicate data to avoid joins)

**Solution 1: Single-Shard Queries (Preferred)**

**How it works conceptually:**
1. Design queries to use shard key
2. Route query to correct shard
3. Query executes on single shard
4. Fast and simple

**Example:**
```
Good: SELECT * FROM orders WHERE user_id = 123
      → Shard determined by user_id
      → Single shard query

Bad:  SELECT * FROM orders WHERE state = 'CA'
      → No shard key
      → Must query all shards
```

**Solution 2: Scatter-Gather Pattern**

**How it works conceptually:**
1. Query arrives (needs multiple shards)
2. Application queries all relevant shards in parallel
3. Each shard returns results
4. Application aggregates results (merge, sort, limit)
5. Return combined results

**Architecture Flow:**
```
Query Request
      │
      ▼
┌─────────────────┐
│  Application    │
│  (Query Router) │
└────────┬────────┘
         │
    ┌────┼────┐
    │    │    │
    ▼    ▼    ▼
┌────┐ ┌────┐ ┌────┐
│S 1 │ │S 2 │ │S N │
└─┬──┘ └─┬──┘ └─┬──┘
  │      │      │
  └──────┼──────┘
         │
    Aggregate Results
         │
    Return Combined
```

**Performance considerations:**
- Query all shards in parallel (not sequential)
- Set timeout per shard (don't wait for slow shard)
- Limit results per shard (then aggregate)
- Cache aggregated results if possible

**Solution 3: Denormalization**

**How it works conceptually:**
1. Duplicate data across shards to avoid joins
2. Example: Store product details in orders table
3. Query orders shard only (no join needed)
4. Trade storage for query performance

**Benefits:**
- Avoids cross-shard joins
- Faster queries (single shard)
- Simpler implementation

**Trade-offs:**
- More storage (duplicated data)
- Update complexity (update multiple shards)
- Potential inconsistency

#### Production Results

| Metric | Before (Cross-Shard) | After (Single-Shard) | Improvement |
|--------|---------------------|---------------------|-------------|
| Query latency | 2.5s | 50ms | 50x faster |
| Database load | 80% | 15% | 5.3x reduction |
| Query complexity | High | Low | Simplified |
| Cache hit rate | 0% | 85% | Can cache single-shard |

**Key insight**: Cross-shard queries are expensive. Design queries to hit single shards whenever possible. Use scatter-gather only when necessary.

**Lessons Learned**:
- Design queries to use shard key (single-shard queries)
- Avoid cross-shard queries when possible
- Use scatter-gather for necessary cross-shard queries
- Cache aggregated results
- Consider denormalization to avoid joins

---

## Common Mistakes

1. **Mistake**: Poor shard key selection
   **Why it's wrong**: Uneven distribution, hot shards, poor query performance
   **Instead**: Choose shard key that distributes evenly and matches query patterns

2. **Mistake**: Too many shards initially
   **Why it's wrong**: Unnecessary complexity, harder to manage
   **Instead**: Start with few shards (2-4), add more as needed

3. **Mistake**: Cross-shard queries
   **Why it's wrong**: Very slow, complex to implement
   **Instead**: Design queries to hit single shard, use application-level joins if needed

4. **Mistake**: No plan for resharding
   **Why it's wrong**: Will need to reshard eventually, unprepared when time comes
   **Instead**: Design sharding strategy with resharding in mind from start

---

## Related Patterns

- **[Database Scaling](./database_scaling.md)** - Scaling database infrastructure
- **[NoSQL Migrations](./nosql_migrations.md)** - Migrating between databases
- **[Query Optimization](./query_optimization.md)** - Optimizing queries for sharded databases

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for complete list of research sources.
