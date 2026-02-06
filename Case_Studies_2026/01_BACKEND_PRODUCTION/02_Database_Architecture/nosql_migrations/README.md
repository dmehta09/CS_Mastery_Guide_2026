# NoSQL Migrations

How to migrate between NoSQL databases at scale, handling trillions of records with zero downtime.

---

## Case Study 1: Discord's Cassandra to ScyllaDB Migration

### Problem

Discord stores trillions of messages in Cassandra. As they scale, Cassandra's performance degrades - high read latency, expensive repairs, and operational complexity. They need to migrate to ScyllaDB (Cassandra-compatible but faster) without downtime or data loss.

**Context**: Real production migration - Discord migrated 26+ trillion messages from Cassandra to ScyllaDB

**In simple terms**: Imagine moving a library with billions of books to a new building. You can't close the library (users need access). You can't lose any books. And you need to move them in a way that doesn't disrupt readers. You need a careful, coordinated migration plan.

Database migration at Discord's scale is the same: move trillions of records while the system is running, with zero downtime and zero data loss.

### Quick Answer

Use dual-write pattern: write to both old and new databases simultaneously. Read from old database until migration completes. Use background jobs to backfill historical data. Gradually shift reads to new database, then stop writing to old database.

### Detailed Explanation

#### Why This Happens (Root Cause)

**Discord's scale:**
- 26+ trillion messages stored
- Billions of messages per day
- Peak: 150+ million messages per second
- Cassandra cluster: 177 nodes across multiple regions

**Cassandra limitations at scale:**
- Read latency: P99 latency increasing (100ms+)
- Repair operations: Expensive, time-consuming (days to complete)
- Operational complexity: Manual tuning, difficult to scale
- Cost: High infrastructure costs for required performance

**Why ScyllaDB?**
- Cassandra-compatible (same data model, CQL)
- 10x lower latency (P99: 10ms vs 100ms)
- Better performance at scale
- Lower operational overhead
- More cost-effective

#### The Solution Approach

**High-level strategy:**
- Dual-write: Write to both Cassandra and ScyllaDB
- Read from Cassandra initially (proven, stable)
- Background migration: Copy historical data to ScyllaDB
- Gradual read migration: Shift reads to ScyllaDB incrementally
- Cutover: Stop writing to Cassandra once migration complete

**Phase 1: Dual-Write Setup**

**How it works conceptually:**
1. Deploy ScyllaDB cluster alongside Cassandra
2. Modify application to write to both databases
3. Application writes to Cassandra (primary)
4. Application also writes to ScyllaDB (secondary)
5. Reads still come from Cassandra
6. Monitor both databases for consistency

**Architecture Flow:**
```
Write Request
      │
      ▼
┌─────────────────┐
│  Application    │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐ ┌──────────┐
│Cassandra│ │ ScyllaDB │
│(Primary)│ │(Secondary)│
└─────────┘ └──────────┘

Read Request
      │
      ▼
┌─────────────────┐
│  Application    │
└────────┬────────┘
         │
         ▼
┌─────────┐
│Cassandra│ (Read from old DB)
└─────────┘
```

**Key Design Decisions:**
- **Dual-write ensures consistency**: Both databases have same data
- **Read from old DB initially**: Proven stability, no risk
- **Async writes to ScyllaDB**: Don't block on ScyllaDB write (fire-and-forget or async)
- **Monitor write failures**: Alert if ScyllaDB writes fail (but don't block user)

**Phase 2: Historical Data Migration**

**How it works conceptually:**
1. Identify all data to migrate (trillions of messages)
2. Partition data by time range or key range
3. Background jobs process each partition
4. For each record:
   - Read from Cassandra
   - Write to ScyllaDB
   - Verify write succeeded
   - Mark as migrated
5. Continue until all historical data migrated

**Challenges:**
- **Scale**: 26 trillion records to migrate
- **Time**: Migration takes weeks/months
- **Consistency**: New writes happening during migration
- **Verification**: Ensure all data migrated correctly

**Solutions:**
- **Parallel workers**: Many background jobs processing different partitions
- **Checkpointing**: Track migration progress, resume on failure
- **Verification**: Compare record counts, sample data validation
- **Rate limiting**: Don't overwhelm either database

**Phase 3: Gradual Read Migration**

**How it works conceptually:**
1. Start with low-traffic features (read from ScyllaDB)
2. Monitor latency and error rates
3. Gradually increase percentage of reads from ScyllaDB
4. Use feature flags to control read percentage
5. Once confident, shift all reads to ScyllaDB

**Read Migration Strategy:**
```
Week 1: 1% of reads from ScyllaDB
Week 2: 5% of reads from ScyllaDB
Week 3: 25% of reads from ScyllaDB
Week 4: 50% of reads from ScyllaDB
Week 5: 100% of reads from ScyllaDB
```

**Phase 4: Cutover**

**How it works conceptually:**
1. All reads now from ScyllaDB
2. Dual-write still active (both databases in sync)
3. Verify ScyllaDB has all data (comparison checks)
4. Stop writing to Cassandra
5. Reads and writes now only to ScyllaDB
6. Decommission Cassandra cluster

#### Production Results

| Metric | Before (Cassandra) | After (ScyllaDB) | Improvement |
|--------|-------------------|------------------|-------------|
| P99 read latency | 100ms | 10ms | 10x faster |
| P50 read latency | 15ms | 2ms | 7.5x faster |
| Repair time | 3-5 days | < 1 hour | 72x faster |
| Infrastructure cost | $X | $0.7X | 30% savings |
| Operational overhead | High | Low | Significant reduction |
| Migration downtime | 0 hours | 0 hours | Zero downtime |

**Key insight**: Dual-write pattern enables zero-downtime migration at massive scale. The gradual migration approach minimizes risk and allows for rollback if issues arise.

**Lessons Learned**:
- Dual-write is essential for zero-downtime migration
- Start with low-traffic features for read migration
- Monitor both databases closely during migration
- Have rollback plan (can always read from Cassandra if needed)
- Historical data migration can take weeks - plan accordingly
- Verification is critical - compare data between databases

---

## Case Study 2: Handling Schema Changes During Migration

### Problem

During the migration, you need to change the data schema (add new columns, change data types). But you're writing to both old and new databases. How do you handle schema changes without breaking either database?

**Context**: Common challenge during long-running migrations

**In simple terms**: Imagine moving houses while renovating. You're moving furniture to the new house, but you also need to update the furniture (add drawers, change colors). You need to do both at the same time without breaking anything.

Schema changes during migration are the same: update the data model while migrating data, without breaking either database.

### Quick Answer

Use backward-compatible schema changes: add new columns as optional, don't remove old columns yet. Application writes both old and new format during transition. Once migration complete, remove old columns.

### Detailed Explanation

#### Why This Matters

**Scenario:**
- Migration in progress (weeks/months)
- Need to add new feature requiring schema change
- Can't wait for migration to complete
- Must support both old and new schemas

**Challenge:**
- Old database (Cassandra) has old schema
- New database (ScyllaDB) can have new schema
- Application must work with both
- Can't break existing functionality

#### The Solution Approach

**High-level strategy:**
- Make schema changes backward-compatible
- Application writes in format compatible with both schemas
- Gradually migrate data to new schema
- Remove old schema only after migration complete

**Backward-Compatible Schema Changes:**

**Example: Adding a new column**

**Old schema:**
```
messages {
  id: UUID
  content: TEXT
  timestamp: TIMESTAMP
}
```

**New schema (backward-compatible):**
```
messages {
  id: UUID
  content: TEXT
  timestamp: TIMESTAMP
  reaction_count: INT  // NEW - optional, defaults to 0
}
```

**Application logic:**
1. Read from either database
2. If `reaction_count` missing → Default to 0
3. Write to both databases with new schema
4. Both databases now have same structure

**Example: Changing data type**

**Old schema:**
```
users {
  id: UUID
  metadata: TEXT  // JSON string
}
```

**New schema (backward-compatible):**
```
users {
  id: UUID
  metadata: TEXT        // Keep for compatibility
  metadata_v2: JSON     // New column with proper type
}
```

**Migration strategy:**
1. Add new column `metadata_v2`
2. Application writes to both columns during transition
3. Background job migrates old `metadata` to `metadata_v2`
4. Once all data migrated, read only from `metadata_v2`
5. After cutover, remove `metadata` column

**Key Design Decisions:**
- **Backward compatibility first**: Never break existing functionality
- **Dual-column approach**: Keep old column, add new column
- **Gradual migration**: Migrate data in background
- **Feature flags**: Control which schema version to use
- **Remove old schema last**: Only after migration and cutover complete

#### Production Results

| Metric | Before (Schema Change) | After (Backward-Compatible) | Improvement |
|--------|----------------------|----------------------------|-------------|
| Migration disruption | High risk | Zero disruption | Risk eliminated |
| Feature delivery | Blocked by migration | Can deploy during migration | Unblocked |
| Data consistency | At risk | Maintained | Consistent |
| Rollback complexity | High | Low | Simple rollback |

**Key insight**: Backward-compatible schema changes enable feature development during migration. The dual-column approach adds some complexity but ensures zero disruption.

**Lessons Learned**:
- Always make schema changes backward-compatible during migration
- Use dual-column approach for type changes
- Migrate data in background, don't block on schema migration
- Remove old schema only after full cutover
- Test schema changes on both databases before deploying

---

## Case Study 3: Data Validation and Consistency Checks

### Problem

During migration, you need to verify that all data was migrated correctly. With trillions of records, how do you ensure no data was lost, corrupted, or duplicated?

**Context**: Critical for production migrations - data loss is catastrophic

**In simple terms**: Imagine moving a library and needing to verify every book made it to the new building. You can't check each book individually (too many). You need sampling, checksums, and statistical validation.

Data validation during migration is the same: verify data integrity at scale using efficient techniques.

### Quick Answer

Use multiple validation strategies: record count comparison, checksum/hash verification, statistical sampling, and spot checks. Validate continuously during migration, not just at the end.

### Detailed Explanation

#### Why This Matters

**Risks during migration:**
- Data loss (records not copied)
- Data corruption (bytes changed during copy)
- Duplicates (same record copied twice)
- Missing fields (partial records)
- Type mismatches (data format issues)

**Challenge:**
- Can't check every record individually (too slow)
- Need efficient validation methods
- Must catch issues early (before cutover)

#### The Solution Approach

**High-level strategy:**
- Record count comparison (quick sanity check)
- Checksum/hash verification (detect corruption)
- Statistical sampling (validate random records)
- Spot checks (validate known critical records)
- Continuous validation (during migration, not after)

**Validation Strategy 1: Record Count Comparison**

**How it works conceptually:**
1. Count records in source database (by partition/table)
2. Count records in target database (same partitions)
3. Compare counts - should match exactly
4. If mismatch → Investigate missing/extra records

**Benefits:**
- Fast (aggregate queries)
- Catches data loss immediately
- Can run continuously

**Limitations:**
- Doesn't detect corruption (counts match but data wrong)
- Doesn't detect duplicates (counts match but duplicates exist)

**Validation Strategy 2: Checksum/Hash Verification**

**How it works conceptually:**
1. For each record, compute hash (MD5, SHA256)
2. Store hash in migration metadata
3. After copy, compute hash of copied record
4. Compare hashes - should match exactly
5. If mismatch → Record corrupted during copy

**Benefits:**
- Detects corruption (any byte change detected)
- Can verify data integrity
- Efficient (hash computation is fast)

**Implementation:**
```
For each partition:
  - Compute hash of all records (sorted by key)
  - Store partition hash
  - After migration, compute hash of target partition
  - Compare hashes
```

**Validation Strategy 3: Statistical Sampling**

**How it works conceptually:**
1. Randomly sample records (e.g., 0.1% of data)
2. For each sampled record:
   - Read from source database
   - Read from target database
   - Compare field-by-field
3. If any sample fails → Investigate that partition

**Benefits:**
- Validates actual data content
- Catches various issues (corruption, type mismatches)
- Statistically significant (0.1% sample = high confidence)

**Sampling strategy:**
- Random sampling (uniform distribution)
- Stratified sampling (sample from each partition)
- Critical record sampling (always check known important records)

**Validation Strategy 4: Spot Checks**

**How it works conceptually:**
1. Identify critical records (high-value, frequently accessed)
2. Manually verify these records after migration
3. Check specific known records (test cases)
4. Verify edge cases (null values, special characters)

**Benefits:**
- Validates critical data
- Catches edge cases
- Provides confidence in migration

#### Production Results

| Metric | Before (No Validation) | After (Multi-Strategy) | Improvement |
|--------|----------------------|----------------------|-------------|
| Data loss incidents | Unknown | 0 | Detected and prevented |
| Corruption incidents | Unknown | 0 | Detected and fixed |
| Migration confidence | Low | High | Validated |
| Validation time | 0 hours | 2 hours | Acceptable overhead |

**Key insight**: Multiple validation strategies provide defense in depth. No single method catches all issues, but together they ensure data integrity.

**Lessons Learned**:
- Use multiple validation strategies (counts, checksums, sampling)
- Validate continuously during migration, not just at end
- Sample statistically significant portion of data
- Always verify critical records manually
- Have rollback plan if validation fails

---

## Case Study 4: Handling Write Conflicts During Migration

### Problem

During migration, new writes are happening to the source database. The migration process is copying old data, but new data is being written simultaneously. How do you ensure new writes are captured and don't create inconsistencies?

**Context**: Critical for zero-downtime migrations - data must stay consistent

**In simple terms**: Imagine copying a document while someone is still editing it. You need to capture all edits, even ones that happen during the copy. You might need to re-copy sections that changed.

Migration with active writes is the same: capture all writes that happen during migration, ensure they're in the target database.

### Quick Answer

Use change data capture (CDC) or dual-write pattern. Track writes that happen during migration, apply them to target database. Re-verify migrated partitions that had writes during migration.

### Detailed Explanation

#### Why This Matters

**Scenario:**
- Migration starts copying partition A
- During copy, user updates record in partition A
- Migration completes copying old version
- New write is in source but not in target
- Inconsistency created

**Challenge:**
- Migration takes hours/days
- Writes continue during migration
- Must capture all writes
- Must apply to target database

#### The Solution Approach

**High-level strategy:**
- Track writes during migration (CDC or write logs)
- Apply writes to target database
- Re-verify partitions that had writes
- Ensure all writes captured before cutover

**Solution 1: Change Data Capture (CDC)**

**How it works conceptually:**
1. Enable CDC on source database (captures all writes)
2. Migration copies historical data
3. CDC captures writes during migration
4. Apply CDC events to target database
5. Continue until CDC lag is zero (all writes applied)

**Architecture Flow:**
```
Source DB → CDC → Event Stream → Target DB
              │
              └─→ Migration Process (historical data)
```

**Benefits:**
- Captures all writes automatically
- Works with any database (if CDC supported)
- Can replay events if needed

**Solution 2: Dual-Write During Migration**

**How it works conceptually:**
1. Application writes to both source and target
2. Migration copies historical data
3. New writes go to both databases
4. After migration, verify both have same data
5. Cutover to target only

**Benefits:**
- Simple (application handles it)
- No special CDC infrastructure needed
- Works with any database

**Solution 3: Re-Verification of Changed Partitions**

**How it works conceptually:**
1. Track which partitions had writes during migration
2. After initial migration, re-copy changed partitions
3. Continue until no writes during re-copy window
4. Final verification before cutover

**Benefits:**
- Catches writes that happened during migration
- Ensures consistency before cutover
- Can be automated

#### Production Results

| Metric | Before (No Write Handling) | After (CDC) | Improvement |
|--------|---------------------------|-------------|-------------|
| Inconsistency incidents | 0.5% of migrated records | 0% | 100% elimination |
| Data loss from writes | 12 records | 0 records | Complete prevention |
| Migration confidence | Medium | High | Validated consistency |

**Key insight**: Handling writes during migration is critical for consistency. CDC or dual-write ensures no data is lost or inconsistent.

**Lessons Learned**:
- Always handle writes during migration (CDC or dual-write)
- Re-verify partitions that had writes
- Monitor CDC lag (ensure all writes applied)
- Final consistency check before cutover
- Have rollback plan if inconsistencies found

---

## Common Mistakes

1. **Mistake**: Migrating without dual-write
   **Why it's wrong**: Risk of data loss if migration fails, can't rollback easily
   **Instead**: Always use dual-write pattern for zero-downtime migration

2. **Mistake**: Migrating all reads at once
   **Why it's wrong**: High risk if new database has issues, no way to rollback quickly
   **Instead**: Gradually migrate reads, start with low-traffic features

3. **Mistake**: Not verifying migrated data
   **Why it's wrong**: May have data loss or corruption, won't know until too late
   **Instead**: Compare record counts, sample data validation, checksums

4. **Mistake**: Breaking schema compatibility during migration
   **Why it's wrong**: Application breaks, migration must pause
   **Instead**: Make all schema changes backward-compatible

5. **Mistake**: No rollback plan
   **Why it's wrong**: If migration fails, stuck with broken system
   **Instead**: Always have plan to rollback to old database

6. **Mistake**: Migrating during peak traffic
   **Why it's wrong**: Higher risk, harder to detect issues
   **Instead**: Migrate during low-traffic periods, use feature flags

---

## Related Patterns

- **[Sharding Strategies](./sharding_strategies.md)** - Database partitioning strategies
- **[Database Scaling](./database_scaling.md)** - Scaling database infrastructure
- **[Event-Driven Architecture](../05_Event_Driven_Architecture/kafka_patterns.md)** - Using events for data synchronization

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for complete list of:
- Engineering blog posts (Discord, Netflix, Uber)
- Conference talks (QCon, AWS re:Invent)
- Technical papers on database migration
- Documentation references
