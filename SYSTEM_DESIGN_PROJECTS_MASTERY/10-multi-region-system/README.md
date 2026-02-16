# Project 10: Multi-Region Active-Active System

## Difficulty: Expert
**Estimated Time**: 5-6 weeks

## Problem Statement

Design a globally distributed system like DynamoDB, Cassandra, or Spanner that operates across multiple regions with active-active replication, handles conflict resolution, and provides low latency worldwide.

**Challenge**: How do you keep data consistent across continents when network latency is 150ms+ and partitions can occur?

## Functional Requirements

1. **Multi-Region Deployment**: Operate in 3+ geographic regions
2. **Active-Active Writes**: Accept writes in any region
3. **Conflict Resolution**: Resolve conflicting writes
4. **Low Latency**: < 50ms local reads, < 200ms cross-region writes
5. **High Availability**: Survive region failures
6. **Eventual Consistency**: Guarantee convergence

## CAP Theorem Trade-offs

```
CAP Theorem: Choose 2 of 3 during network partition
- Consistency (C)
- Availability (A)
- Partition Tolerance (P)

Multi-region systems choose: AP (Availability + Partition Tolerance)
→ Sacrifice strong consistency for availability
→ Use eventual consistency
```

## Replication Strategies

### 1. Master-Slave (Active-Passive)

```
Region 1 (Master)    Region 2 (Slave)    Region 3 (Slave)
    │                     ▲                   ▲
    │                     │                   │
    └─────────────────────┴───────────────────┘
              Replication (async)

Write: Always to master
Read: From any region (may be stale)

Pros: Simple, no conflicts
Cons: Master is bottleneck, high write latency for distant users
```

### 2. Multi-Master (Active-Active)

```
Region 1 (Master) ←──→ Region 2 (Master) ←──→ Region 3 (Master)
         ▲                      ▲                      ▲
         │                      │                      │
         └──────────────────────┴──────────────────────┘
                    Bi-directional replication

Write: To any region (local writes)
Read: From local region

Pros: Low latency everywhere, high availability
Cons: Conflicts! Need resolution strategy
```

## Conflict Resolution

### Problem: Concurrent Writes

```
Scenario: User updates profile in US and EU simultaneously

t=0: US writes: name="Alice"
t=0: EU writes: name="Bob"

t=1: Replication propagates

US receives: name="Bob"
EU receives: name="Alice"

Question: Which value wins?
```

### Strategy 1: Last-Write-Wins (LWW)

```typescript
interface VersionedValue {
  value: string;
  timestamp: number;
  region: string;
}

function resolveConflict(v1: VersionedValue, v2: VersionedValue): VersionedValue {
  if (v1.timestamp > v2.timestamp) {
    return v1;
  } else if (v2.timestamp > v1.timestamp) {
    return v2;
  } else {
    // Tie-breaker: lexicographic order of region
    return v1.region > v2.region ? v1 : v2;
  }
}
```

**Problem with LWW**: Can lose data

```
User clicks "Save" twice (network issue)
Request 1: {value: "Important data", timestamp: 100}
Request 2: {value: "", timestamp: 101} (accidental empty)

Result: Empty value wins! Data lost!
```

### Strategy 2: Version Vectors

```typescript
interface VersionVector {
  [regionId: string]: number;
}

// Example:
// Region US: {US: 3, EU: 2, ASIA: 1}
// Region EU: {US: 2, EU: 3, ASIA: 1}

function compareClock(v1: VersionVector, v2: VersionVector): 'before' | 'after' | 'concurrent' {
  let v1Greater = false;
  let v2Greater = false;

  const allRegions = new Set([...Object.keys(v1), ...Object.keys(v2)]);

  for (const region of allRegions) {
    const count1 = v1[region] || 0;
    const count2 = v2[region] || 0;

    if (count1 > count2) v1Greater = true;
    if (count2 > count1) v2Greater = true;
  }

  if (v1Greater && !v2Greater) return 'after';
  if (v2Greater && !v1Greater) return 'before';
  return 'concurrent';
}

// Concurrent writes require application-level resolution
```

### Strategy 3: CRDT (Conflict-Free Replicated Data Type)

**Concept**: Data structures that automatically merge without conflicts

**Example: G-Counter (Grow-only Counter)**

```typescript
class GCounter {
  private counts: Map<string, number> = new Map();

  increment(regionId: string, amount: number = 1) {
    const current = this.counts.get(regionId) || 0;
    this.counts.set(regionId, current + amount);
  }

  value(): number {
    let sum = 0;
    for (const count of this.counts.values()) {
      sum += count;
    }
    return sum;
  }

  merge(other: GCounter): GCounter {
    const merged = new GCounter();

    // Take max of each region
    const allRegions = new Set([
      ...this.counts.keys(),
      ...other.counts.keys()
    ]);

    for (const region of allRegions) {
      const max = Math.max(
        this.counts.get(region) || 0,
        other.counts.get(region) || 0
      );
      merged.counts.set(region, max);
    }

    return merged;
  }
}

// Example: Like counter on post
Region US:   {US: 10, EU: 5}  → Total: 15 likes
Region EU:   {US: 8, EU: 7}   → Total: 15 likes

After merge: {US: 10, EU: 7}  → Total: 17 likes (consistent!)
```

**CRDT Types**:
- **G-Counter**: Increment-only counter
- **PN-Counter**: Positive-negative counter (can decrement)
- **LWW-Register**: Last-write-wins register
- **OR-Set**: Add/remove set (observed-remove)

## Cross-Region Communication

### Replication Lag

```
Write in US → Replicate to EU → Replicate to ASIA

Latency:
- US → EU: 80ms
- US → ASIA: 150ms
- EU → ASIA: 200ms

User writes in US, reads in ASIA:
→ May see stale data for 150-200ms
```

### Read-Your-Writes Consistency

```typescript
class SessionStickiness {
  private lastWriteRegion: Map<string, string> = new Map();

  async write(userId: string, region: string, data: any) {
    // Write to local region
    await db.write(data);

    // Remember which region user wrote to
    this.lastWriteRegion.set(userId, region);
  }

  async read(userId: string, currentRegion: string) {
    const writeRegion = this.lastWriteRegion.get(userId);

    if (writeRegion && writeRegion !== currentRegion) {
      // User wrote in different region, read from there
      return await db.readFromRegion(writeRegion);
    }

    // Read from current region (fast)
    return await db.readLocal();
  }
}
```

## Database Schema

```sql
-- Multi-region table with version vectors
CREATE TABLE users (
  id UUID PRIMARY KEY,
  name VARCHAR(255),
  email VARCHAR(255),
  version_vector JSONB,  -- {us: 3, eu: 2, asia: 1}
  last_modified_region VARCHAR(10),
  last_modified_at TIMESTAMP,

  INDEX idx_email (email)
);

-- Replication log
CREATE TABLE replication_log (
  id BIGSERIAL PRIMARY KEY,
  table_name VARCHAR(100),
  record_id UUID,
  operation VARCHAR(10),  -- INSERT, UPDATE, DELETE
  data JSONB,
  source_region VARCHAR(10),
  version_vector JSONB,
  created_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_created (created_at)
);
```

## Topology Designs

### Topology 1: Ring

```
Region A ←→ Region B
    ▲           │
    │           ▼
    └─ Region C

Each region replicates to next
Pro: Simple
Con: Higher latency (3 hops for full replication)
```

### Topology 2: Full Mesh

```
Region A ←→ Region B
    ▲  ╲    ╱  ▲
    │   ╳  ╱   │
    │  ╱  ╲    │
    ▼ ╱    ╲   ▼
Region C ←→ (all connected)

Each region replicates to all others
Pro: Lowest latency
Con: O(N²) connections
```

### Topology 3: Hub-and-Spoke

```
      Region B
         ▲ │
         │ ▼
Region A ◄─► Region C
    (Hub)

All regions replicate through hub
Pro: O(N) connections
Con: Hub is bottleneck
```

## Failure Scenarios

### Scenario 1: Region Failure

```
Region US: DOWN
Region EU: UP
Region ASIA: UP

Solution: Redirect US traffic to EU (nearest)
- DNS failover (TTL: 60s)
- Load balancer health checks
- Client retries with exponential backoff
```

### Scenario 2: Network Partition

```
Split-brain: US-EU can talk, but both cut off from ASIA

US-EU: Continue operations (quorum: 2/3)
ASIA: Read-only mode OR continue with risk of conflicts

When partition heals:
→ Merge conflicting writes using CRDT/version vectors
```

### Scenario 3: Clock Skew

```
Problem: Server clocks out of sync

US server time: 10:00:00
EU server time: 09:59:55 (5 seconds behind)

Write in US: timestamp 10:00:00
Write in EU: timestamp 09:59:55

LWW says: US wins, but EU write might have been later!

Solution: Use logical clocks (Lamport, vector clocks)
```

## API Design

```http
# Write with region routing
POST /api/v1/users
X-Region-Hint: us-east-1
Body: {
  "name": "Alice",
  "email": "alice@example.com"
}
Response: {
  "id": "user-123",
  "region": "us-east-1",
  "version": {"us-east-1": 1}
}

# Read with consistency level
GET /api/v1/users/user-123?consistency=strong
→ Reads from majority of regions (slow, consistent)

GET /api/v1/users/user-123?consistency=eventual
→ Reads from local region (fast, may be stale)

# Global query (expensive!)
GET /api/v1/users?limit=10
→ Scatter-gather across all regions
→ Merge results
```

## Monitoring

```yaml
Metrics per region:
  - replication_lag_ms: Time to replicate to other regions
  - conflict_rate: Conflicts per second
  - cross_region_latency_p95: Network latency
  - region_availability: % uptime

Global metrics:
  - global_read_latency_p95: < 50ms
  - global_write_latency_p95: < 200ms
  - consistency_violations: Should be 0

Alerts:
  - name: HighReplicationLag
    condition: replication_lag > 5000ms
    severity: warning

  - name: RegionDown
    condition: region_availability < 99%
    severity: critical
```

## Summary

### What We Built:
- Multi-region active-active system
- Conflict resolution strategies
- CRDT for automatic merging
- Cross-region replication

### Key Concepts:
- ✅ CAP theorem (AP system)
- ✅ Eventual consistency
- ✅ Conflict resolution (LWW, version vectors, CRDT)
- ✅ Replication topologies
- ✅ Read-your-writes consistency

### Real-World Examples:
- DynamoDB Global Tables: Active-active across regions
- Cassandra: Multi-datacenter replication
- CockroachDB: Geo-partitioning

---

**Next Project**: [11. Zero-Trust Authentication System](../11-zero-trust-auth/README.md)
