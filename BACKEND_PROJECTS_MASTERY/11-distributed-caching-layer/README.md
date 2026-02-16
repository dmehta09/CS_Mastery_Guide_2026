# Project 11: Distributed Caching Layer

## Difficulty Level: Advanced
**Estimated Time**: 2-3 weeks per language

## What You're Building

A production-grade distributed caching system like Redis Cluster or Memcached that can scale horizontally, with features like cache invalidation, consistent hashing, and multi-level caching.

## Key Concepts

- Cache strategies (Aside, Through, Behind)
- Cache invalidation patterns
- Consistent hashing
- Cache stampede prevention
- Hot key problem
- Multi-level caching (L1, L2)
- TTL strategies
- Write-behind buffering

## Architecture

```
Application → L1 Cache (In-Memory) → L2 Cache (Redis) → Database
                     ↓                      ↓
                LRU Eviction         Consistent Hashing
```

## Cache Patterns

### 1. Cache-Aside (Lazy Loading)

```typescript
async function getData(key: string) {
  // Try L1 cache
  let data = l1Cache.get(key);
  if (data) return data;

  // Try L2 cache (Redis)
  data = await redis.get(key);
  if (data) {
    l1Cache.set(key, data);
    return data;
  }

  // Fallback to database
  data = await db.query(key);
  await redis.set(key, data, "EX", 3600);
  l1Cache.set(key, data);
  return data;
}
```

### 2. Write-Through Cache

```typescript
async function setData(key: string, value: any) {
  // Write to cache and DB simultaneously
  await Promise.all([
    redis.set(key, value, "EX", 3600),
    db.update(key, value)
  ]);
  l1Cache.set(key, value);
}
```

### 3. Write-Behind (Write-Back)

```typescript
const writeBuffer = new Map();

async function setData(key: string, value: any) {
  // Write to cache immediately
  await redis.set(key, value, "EX", 3600);
  l1Cache.set(key, value);

  // Buffer DB write
  writeBuffer.set(key, value);

  // Flush buffer every 5 seconds
  scheduleFlush();
}

async function flushBuffer() {
  const batch = Array.from(writeBuffer.entries());
  writeBuffer.clear();

  await db.batchUpdate(batch);
}
```

## Consistent Hashing

```typescript
class ConsistentHash {
  private ring: Map<number, string> = new Map();
  private replicas = 150; // Virtual nodes per server

  addNode(node: string) {
    for (let i = 0; i < this.replicas; i++) {
      const hash = this.hash(`${node}:${i}`);
      this.ring.set(hash, node);
    }
    this.sortRing();
  }

  getNode(key: string): string {
    const hash = this.hash(key);
    const sortedHashes = Array.from(this.ring.keys()).sort();

    // Find first hash >= key hash
    const idx = sortedHashes.findIndex((h) => h >= hash);
    const targetHash = sortedHashes[
      idx === -1 ? 0 : idx
    ];

    return this.ring.get(targetHash)!;
  }

  private hash(str: string): number {
    // Use MD5 or similar
    return crc32(str);
  }
}
```

## Hot Key Detection

```typescript
class HotKeyDetector {
  private counters = new Map<string, number>();
  private window = 60000; // 1 minute

  async track(key: string) {
    const count = (this.counters.get(key) || 0) + 1;
    this.counters.set(key, count);

    if (count > 1000) {
      // Hot key detected!
      await this.handleHotKey(key);
    }
  }

  private async handleHotKey(key: string) {
    // 1. Replicate to local caches
    const value = await redis.get(key);
    this.broadcastToLocalCaches(key, value);

    // 2. Notify monitoring
    metrics.increment("hot_keys", { key });

    // 3. Consider sharding this key
    console.log(`Hot key detected: ${key}`);
  }
}
```

## Success Criteria

✅ Sub-millisecond cache lookups
✅ 99.9% cache hit rate
✅ Consistent hashing working
✅ Hot key mitigation
✅ Automatic invalidation

---

**Next**: Project 12 (Observability System)
