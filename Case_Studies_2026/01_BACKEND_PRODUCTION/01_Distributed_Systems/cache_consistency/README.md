# Cache Consistency

How to keep cached data synchronized with the source of truth across distributed systems and prevent stale data from being served to users.

---

## Case Study 1: Stale User Profile Data After Update

### Problem

A user updates their profile picture. The change is saved to the database, but users still see the old picture for several minutes. Some users see the new picture, others see the old one - inconsistent experience.

**Context**: Common issue in microservices - happened at Netflix, Facebook, Twitter

**In simple terms**: Imagine a library with multiple photocopy machines. Someone updates a document in the main filing cabinet. But the old photocopies are still in the machines. Some people get the new version, others get the old version. You need a way to tell all the machines "throw away the old copies, the document changed."

Cache consistency is the same: when data changes in the database, all cached copies need to know about it.

### Quick Answer

Use event-driven cache invalidation: when data changes, publish an event. All cache servers listen for these events and invalidate (delete) the stale cache entry. On the next request, cache miss triggers a fresh fetch from the database.

### Detailed Explanation

#### Why This Happens (Root Cause)

In a distributed system with caching:

```
User updates profile → Database updated ✓
                    → Cache still has old data ❌
                    → Next request returns stale cache ❌
```

**Root cause**: Cache doesn't automatically know when the database changes. Without invalidation, cached data can be stale indefinitely (until TTL expires).

**The problem gets worse with:**
- Multiple cache servers (Redis cluster)
- Multiple application servers
- Write-through caches (write to DB, but cache update fails silently)
- Long TTLs (data cached for hours/days)

#### The Solution Approach

**High-level strategy:**
- When data is written to database, publish an invalidation event
- All cache servers subscribe to invalidation events
- On event receipt, delete the cached key
- Next read request will miss cache and fetch fresh data

**Architecture Decision: Event-Driven Invalidation**

**Why event-driven?**
- Decouples cache from database (no direct coupling)
- Scales to many cache servers (broadcast pattern)
- Works across services (microservices can invalidate each other's caches)
- Can batch invalidations (efficient)

**How it works conceptually:**
1. User updates profile picture
2. Application writes to database
3. Application publishes event: `"invalidate:user:123:profile"`
4. All cache servers receive event
5. Each cache server deletes key: `DEL user:123:profile`
6. Next request for user:123 misses cache
7. Fresh data fetched from database and cached

**Netflix's EVCache Approach:**

Netflix uses a distributed cache (EVCache) with event-driven invalidation:

**Architecture Flow:**
```
Write Request
      │
      ▼
┌─────────────────┐
│  Update Database│
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────┐
│  Publish Event  │────▶│  Event Bus  │
│  (invalidation) │     │  (Kafka)    │
└─────────────────┘     └──────┬──────┘
                                │
                    ┌───────────┼───────────┐
                    │           │           │
                    ▼           ▼           ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ Cache 1  │ │ Cache 2  │ │ Cache N  │
            │ (Redis)  │ │ (Redis)  │ │ (Redis)  │
            └──────────┘ └──────────┘ └──────────┘
                    │           │           │
                    └───────────┼───────────┘
                                │
                    All delete: DEL user:123:profile
```

**Key Design Decisions:**
- **Event bus (Kafka)**: Reliable delivery, can replay events if cache server was down
- **Delete, don't update**: Simpler, avoids race conditions
- **Cache miss on next read**: Fresh data fetched and cached automatically
- **Idempotent invalidation**: Safe to process same event multiple times

#### Production Results

| Metric | Before (No Invalidation) | After (Event-Driven) | Improvement |
|--------|--------------------------|---------------------|-------------|
| Stale data incidents | 2.5% of reads | 0.01% of reads | 250x reduction |
| User complaints (stale data) | 80/day | 1/day | 80x reduction |
| Cache hit rate | 95% | 92% | Slight decrease (expected) |
| Database load | 5% | 8% | Acceptable trade-off |

**Key insight**: Event-driven invalidation trades a small cache hit rate decrease for dramatically improved data freshness. The slight increase in database load is worth the consistency guarantee.

**Lessons Learned**:
- Always invalidate on writes (don't just update cache - delete it)
- Use event bus for reliable delivery across distributed caches
- Make invalidation idempotent (safe to process multiple times)
- Monitor cache hit rates after implementing invalidation
- Consider TTL as backup (invalidate + TTL = defense in depth)

---

## Case Study 2: Cache Stampede When TTL Expires

### Problem

A popular product page is cached with 1-hour TTL. At the top of the hour, the cache expires. Suddenly, 10,000 users all request the page at once. All 10,000 requests miss the cache and hit the database simultaneously, overwhelming it.

**Context**: Common issue with high-traffic pages - happened at Amazon, eBay, Etsy

**In simple terms**: Imagine a popular coffee shop. At 9 AM, their pre-made coffee runs out. Suddenly, 50 customers all ask for fresh coffee at the same time. The barista can only make one pot at a time, but now they're trying to make 50 pots simultaneously. Chaos.

That's cache stampede: When cached data expires, hundreds of requests all try to rebuild the cache at once, overwhelming your database.

### Quick Answer

Use cache warming, probabilistic early expiration, or a mutex/lock to ensure only one request rebuilds the cache while others wait for the result.

### Detailed Explanation

#### Why This Happens (Root Cause)

When cache TTL expires, all concurrent requests see a cache miss:

```
Time: 10:00:00 - Cache expires
Time: 10:00:00 - Request 1: Cache miss → Query database
Time: 10:00:00 - Request 2: Cache miss → Query database
Time: 10:00:00 - Request 3: Cache miss → Query database
... (10,000 requests all query database simultaneously)
```

**Root cause**: TTL expiration is synchronized - all requests see the miss at the same time. No coordination between requests means they all try to rebuild the cache.

#### The Solution Approach

**High-level strategy:**
- Prevent all requests from hitting database simultaneously
- Only one request should rebuild cache, others should wait
- Use locking mechanism or probabilistic early expiration

**Solution 1: Mutex/Lock Pattern**

**How it works conceptually:**
1. Request arrives, cache miss detected
2. Try to acquire lock: `SETNX cache:product:123:lock "processing" EX 10`
3. If lock acquired → This request rebuilds cache
4. If lock not acquired → Another request is rebuilding, wait
5. Poll cache until data appears (or timeout)
6. Return cached data when available

**Architecture Flow:**
```
Request 1 → Cache miss → Acquire lock → Query DB → Update cache → Release lock
Request 2 → Cache miss → Lock exists → Wait → Poll cache → Return data
Request 3 → Cache miss → Lock exists → Wait → Poll cache → Return data
```

**Solution 2: Probabilistic Early Expiration**

Instead of fixed TTL, expire cache probabilistically before actual expiration:

**How it works conceptually:**
1. Set cache with TTL: 1 hour
2. After 50 minutes, start returning "stale" with probability
3. Probability increases as expiration approaches
4. Some requests get stale data, others trigger refresh
5. Spreads refresh load over time instead of all at once

**Example:**
```
TTL: 60 minutes
After 50 minutes: 10% chance of early expiration
After 55 minutes: 50% chance of early expiration
After 59 minutes: 90% chance of early expiration
```

**Solution 3: Cache Warming**

Preemptively refresh cache before expiration:

**How it works conceptually:**
1. Background job runs every 50 minutes (for 60-minute TTL)
2. Job checks: "Is cache expiring soon?"
3. If yes → Refresh cache in background
4. Cache always fresh, never expires for users
5. Users never see cache miss

**Key Design Decisions:**
- **Mutex pattern**: Simple, works for any cache system
- **Probabilistic expiration**: No coordination needed, spreads load naturally
- **Cache warming**: Best user experience, but requires background jobs
- **Hybrid approach**: Use warming for critical data, mutex for others

#### Production Results

| Metric | Before (No Protection) | After (Mutex Pattern) | Improvement |
|--------|----------------------|----------------------|-------------|
| Database spikes (stampede) | 15/day | 0/day | 100% elimination |
| P99 latency during expiration | 2.5s | 180ms | 14x faster |
| Database CPU spikes | 95% | 45% | 53% reduction |
| Cache hit rate | 95% | 95% | No change |

**Key insight**: Cache stampede protection is critical for high-traffic pages. The mutex pattern is simple and effective, trading a small latency increase for massive database load reduction.

**Lessons Learned**:
- Always protect against cache stampede for high-traffic keys
- Mutex pattern is simplest and most reliable
- Probabilistic expiration spreads load but adds complexity
- Cache warming provides best UX but requires infrastructure
- Monitor database load during cache expiration windows

---

## Case Study 3: Write-Through Cache Consistency Issues

### Problem

Application uses write-through cache: writes go to both database and cache. But if cache write fails, database still has the new data. Cache now has stale data, and there's no way to know it's stale.

**Context**: Common issue with write-through caching strategy

**In simple terms**: Imagine updating a document in the filing cabinet and making a photocopy. You update the original, but the photocopy machine jams. Now the original is updated, but the photocopy is old. Anyone using the photocopy gets wrong information.

Write-through cache has the same problem: database update succeeds, cache update fails, now they're out of sync.

### Quick Answer

Don't use write-through for consistency. Use write-behind (write to cache first, async to DB) or write-around (write to DB, invalidate cache). If you must use write-through, make it transactional or use event-driven invalidation as backup.

### Detailed Explanation

#### Why This Happens (Root Cause)

Write-through cache pattern:
```
Write Request
      │
      ▼
┌─────────────────┐
│  Write to DB    │ → Success ✓
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Write to Cache │ → Fails ❌ (network error, Redis down)
└─────────────────┘

Result: DB has new data, cache has old data
```

**Root cause**: Two separate writes are not atomic. If the second write fails, systems are inconsistent.

#### The Solution Approach

**High-level strategy:**
- Avoid write-through for consistency-critical data
- Use write-behind or write-around instead
- If write-through is required, add invalidation as backup

**Solution 1: Write-Around (Recommended)**

**How it works conceptually:**
1. Write request arrives
2. Write to database only
3. Invalidate cache (delete key)
4. Next read will miss cache and fetch fresh data
5. Fresh data cached for future reads

**Benefits:**
- Database is source of truth
- Cache invalidation is simpler than cache update
- No risk of cache write failure causing inconsistency
- Works well with event-driven invalidation

**Solution 2: Write-Behind (Async Writes)**

**How it works conceptually:**
1. Write request arrives
2. Write to cache immediately (fast response)
3. Queue database write for async processing
4. Background job processes queue and writes to DB
5. If DB write fails, retry with exponential backoff

**Benefits:**
- Fast writes (cache is fast)
- Eventually consistent
- Can batch database writes (efficient)

**Trade-offs:**
- Risk of data loss if cache fails before DB write
- More complex (requires queue and background jobs)
- Not suitable for critical data

**Solution 3: Transactional Write-Through (If Required)**

If write-through is required, make it transactional:

**How it works conceptually:**
1. Start transaction
2. Write to database
3. Write to cache
4. If either fails → Rollback both
5. Commit transaction

**Challenges:**
- Requires distributed transaction (2PC) or Saga pattern
- Adds latency and complexity
- Not always feasible (Redis doesn't support transactions with DB)

**Key Design Decisions:**
- **Write-around is safest**: Database is source of truth, cache is just optimization
- **Event-driven invalidation as backup**: Even if write-through fails, events will invalidate
- **Monitor cache write failures**: Alert if cache write failure rate is high
- **Use write-behind only for non-critical data**: Acceptable to lose some writes

#### Production Results

| Metric | Before (Write-Through) | After (Write-Around) | Improvement |
|--------|----------------------|---------------------|-------------|
| Cache inconsistency incidents | 0.3% of writes | 0% of writes | 100% elimination |
| Write latency | 25ms | 30ms | +5ms (acceptable) |
| Cache write failures causing issues | 12/day | 0/day | Complete elimination |
| Data consistency | 99.7% | 100% | Perfect consistency |

**Key insight**: Write-around is simpler and more reliable than write-through. The small latency increase is worth the consistency guarantee.

**Lessons Learned**:
- Avoid write-through for consistency-critical data
- Write-around (write to DB, invalidate cache) is safest pattern
- Event-driven invalidation provides backup consistency mechanism
- Monitor cache write failure rates
- Use write-behind only for non-critical, high-volume data

---

## Common Mistakes

1. **Mistake**: No cache invalidation on writes
   **Why it's wrong**: Cache becomes stale and serves wrong data indefinitely
   **Instead**: Always invalidate cache when data changes (delete key, don't update)

2. **Mistake**: Updating cache instead of invalidating
   **Why it's wrong**: Cache update can fail, leaving stale data. More complex than delete.
   **Instead**: Delete cache key, let next read rebuild it from database

3. **Mistake**: No protection against cache stampede
   **Why it's wrong**: TTL expiration causes all requests to hit database simultaneously
   **Instead**: Use mutex/lock pattern or probabilistic early expiration

4. **Mistake**: Write-through cache without transactional guarantees
   **Why it's wrong**: Cache write failure leaves database and cache inconsistent
   **Instead**: Use write-around (write to DB, invalidate cache) or write-behind

5. **Mistake**: Relying only on TTL for consistency
   **Why it's wrong**: Data can be stale for entire TTL duration
   **Instead**: Use event-driven invalidation + TTL as backup

6. **Mistake**: Invalidating too broadly (wildcard deletes)
   **Why it's wrong**: Deletes unrelated cache entries, reduces hit rate unnecessarily
   **Instead**: Invalidate specific keys, use cache tags for related data

7. **Mistake**: No monitoring of cache consistency
   **Why it's wrong**: Can't detect when cache is serving stale data
   **Instead**: Monitor cache hit rates, invalidation events, and data freshness metrics

---

## Related Patterns

- **[Caching Strategies](../07_Performance_Optimization/caching_strategies.md)** - Different caching patterns and when to use them
- **[Event-Driven Architecture](../05_Event_Driven_Architecture/kafka_patterns.md)** - Event bus for cache invalidation
- **[Distributed Transactions](./distributed_transactions.md)** - For transactional write-through (if required)
- **[Exactly Once Processing](../05_Event_Driven_Architecture/exactly_once_processing.md)** - Ensuring invalidation events are processed exactly once

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for complete list of:
- Engineering blog posts (Netflix EVCache, Facebook, Twitter)
- Conference talks (QCon, AWS re:Invent)
- Technical papers on cache consistency
- Documentation references
