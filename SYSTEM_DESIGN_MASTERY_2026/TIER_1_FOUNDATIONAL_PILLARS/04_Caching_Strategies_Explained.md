# Caching Strategies

## Table of Contents
1. [Introduction to Caching](#introduction-to-caching)
2. [Cache Patterns](#cache-patterns)
3. [Cache Invalidation Strategies](#cache-invalidation-strategies)
4. [Distributed Caching](#distributed-caching)
5. [CDN and Edge Caching](#cdn-and-edge-caching)
6. [Cache Consistency Models](#cache-consistency-models)
7. [Cache Performance](#cache-performance)

---

## Introduction to Caching

**Caching** is a technique where frequently accessed data is stored in a fast-access storage layer (the cache) so that future requests for that data can be served more quickly. Think of it like keeping your most-used kitchen utensils on the countertop rather than in a drawer—you save time by not having to search for them every time.

### Why Caching Matters

- **Speed**: Reading from memory (cache) is thousands of times faster than reading from disk or network
- **Cost**: Reduces load on expensive resources like databases or external APIs
- **Scalability**: Allows systems to handle more users without proportionally increasing backend resources
- **Reliability**: Can serve cached data even if the original data source is temporarily unavailable

### The Fundamental Trade-off

Caching involves balancing:
- **Freshness** (how up-to-date the cached data is)
- **Performance** (how fast you can serve requests)
- **Complexity** (how difficult the caching system is to maintain)

---

## Cache Patterns

Cache patterns define **how** and **when** data flows between your application, the cache, and the database. Each pattern has different characteristics regarding who controls the cache and when data gets loaded or written.

```
Application Layer
      |
      v
  Cache Layer  <--> Pattern determines data flow
      |
      v
Database Layer
```

### Cache-Aside (Lazy Loading)

**What it is**: The application code directly manages the cache. When data is needed, the app checks the cache first; if not found (a "cache miss"), it fetches from the database and then stores it in the cache.

**How it works**:

```
READ FLOW:
---------
App needs data
    |
    v
Check Cache
    |
    +---> [MISS] ---> Query Database ---> Store in Cache ---> Return to App
    |
    +---> [HIT] ----> Return from Cache ---> Return to App


WRITE FLOW:
----------
App writes data
    |
    v
Write to Database
    |
    v
Invalidate Cache (remove stale data)
```

**Characteristics**:
- **Application manages everything**: The app code explicitly checks cache, loads data, and updates cache
- **Lazy loading**: Data only gets cached when it's actually requested (not preloaded)
- **Cache miss penalty**: First request is slow (database query), subsequent requests are fast

**Best for**:
- Read-heavy workloads where the same data is accessed repeatedly
- When you want fine-grained control over what gets cached
- Systems where stale data for a short period is acceptable

**Layman explanation**: Imagine you're cooking and need a spice. You check your spice rack (cache) first. If it's not there (miss), you go to the pantry (database), get it, and put it on the rack for next time. If it's already on the rack (hit), you just grab it immediately.

---

### Write-Through

**What it is**: Every write operation updates both the cache and the database simultaneously. The cache always stays in sync with the database.

**How it works**:

```
WRITE FLOW:
----------
App writes data
    |
    v
    +----------+----------+
    |                     |
    v                     v
Write to Cache    Write to Database
    |                     |
    +----------+----------+
               |
               v
    Both complete --> Confirm to App


READ FLOW:
---------
App needs data
    |
    v
Check Cache
    |
    +---> [MISS] ---> Query Database ---> Store in Cache ---> Return
    |
    +---> [HIT] ----> Return from Cache
```

**Characteristics**:
- **Synchronous dual writes**: Both cache and database are updated before confirming success
- **Always consistent**: Cache never has data that's not in the database
- **Write latency**: Writes are slower because they must complete in both places
- **No data loss risk**: If cache fails, data is still in database

**Best for**:
- Applications that cannot tolerate stale reads after writes
- When data consistency is more important than write speed
- Often combined with read-through for complete automation

**Layman explanation**: It's like writing something in both your physical notebook and your digital notes app at the same time—you spend a bit more time writing, but you're guaranteed both copies match.

---

### Write-Behind (Write-Back)

**What it is**: Writes go to the cache immediately, and the cache asynchronously writes to the database in the background. This decouples write speed from database performance.

**How it works**:

```
WRITE FLOW:
----------
App writes data
    |
    v
Write to Cache (FAST)
    |
    v
Confirm to App immediately
    |
    v
[Background Process]
    |
    v
Batch write to Database (async)


TIMELINE VIEW:
-------------
Time 0ms:  App writes -> Cache updated -> App gets confirmation
Time 100ms: Cache accumulates more writes
Time 500ms: Background job flushes batch to Database
```

**Characteristics**:
- **Asynchronous writes**: Database updates happen later, not immediately
- **Very fast writes**: App doesn't wait for database operations
- **Data loss risk**: If cache crashes before flushing, recent writes could be lost
- **Write batching**: Multiple writes can be grouped for efficiency

**Best for**:
- Write-heavy applications where write speed is critical
- Logging and metrics systems where occasional data loss is acceptable
- Systems with bursty write patterns that benefit from batching

**Layman explanation**: Like jotting quick notes on a whiteboard during a meeting (fast), then later typing them into a document properly (slow but permanent). If the whiteboard gets erased before you type them up, you lose those notes.

---

### Read-Through

**What it is**: The cache itself manages fetching data from the database. When the application requests data, it only talks to the cache—the cache automatically loads missing data from the database.

**How it works**:

```
READ FLOW:
---------
App requests data from Cache
    |
    v
Cache checks itself
    |
    +---> [HIT] ---> Return to App
    |
    +---> [MISS] ---> Cache queries Database
                           |
                           v
                      Cache stores data
                           |
                           v
                      Return to App


App's perspective:
-----------------
App --> Cache (single point of contact)

Cache's internal logic:
----------------------
if (data in cache):
    return data
else:
    data = fetch_from_database()
    store_in_cache(data)
    return data
```

**Characteristics**:
- **Cache manages loading**: Application code is simpler—just ask the cache
- **Transparent caching**: App doesn't know whether data came from cache or database
- **First-access penalty**: Initial miss still requires database query
- **Simplified code**: No manual cache population logic in application

**Best for**:
- Applications where you want to abstract away caching complexity
- Read-heavy workloads with predictable access patterns
- Often paired with write-through for complete cache automation

**Layman explanation**: It's like having a smart assistant. You ask them for a document, and they either hand it to you if they have it, or go get it from the filing cabinet without you having to know where it's stored.

---

### Refresh-Ahead (Proactive Refresh)

**What it is**: The cache proactively refreshes data **before** it expires, based on access patterns or predictions. Instead of waiting for a cache miss, the system anticipates which data will be needed and refreshes it in the background.

**How it works**:

```
NORMAL FLOW (without refresh-ahead):
-----------------------------------
Time 0:   Data cached, TTL = 10 minutes
Time 10m: Data expires
Time 10m+1s: User requests data --> MISS --> Slow response


WITH REFRESH-AHEAD:
------------------
Time 0:   Data cached, TTL = 10 minutes
Time 8m:  System detects data is frequently accessed
Time 8m:  Background job refreshes data from database
Time 10m: Old TTL expires, but fresh data already loaded
Time 10m+1s: User requests data --> HIT --> Fast response


DETECTION FLOW:
--------------
Monitor access patterns
    |
    v
Identify "hot" items (frequently accessed)
    |
    v
Before TTL expires:
    |
    v
Asynchronously fetch fresh data
    |
    v
Replace cache entry
```

**Characteristics**:
- **Predictive loading**: Anticipates future requests based on past behavior
- **No cache miss penalty**: Users don't experience slow responses due to expired data
- **Resource overhead**: Background processes consume CPU/network even when data might not be needed
- **Complexity**: Requires logic to determine what to refresh and when

**Best for**:
- Popular content that's accessed very frequently (news sites, product pages)
- Applications where consistent fast response times are critical
- Data that changes predictably (daily reports, scheduled updates)

**Layman explanation**: Like a coffee shop that starts brewing a fresh pot of coffee 10 minutes before the current pot runs out, based on knowing customers always come around that time. Customers never wait for coffee to brew.

---

## Cache Invalidation Strategies

**Cache invalidation** is the process of removing or updating stale (outdated) data from the cache. As the famous quote goes: *"There are only two hard things in Computer Science: cache invalidation and naming things."*

The challenge: How do we ensure the cache doesn't serve old data when the underlying data has changed?

```
THE CACHE INVALIDATION PROBLEM:
------------------------------
Database: User's email = "new@email.com" (updated 1 minute ago)
Cache:    User's email = "old@email.com" (stale data)
App:      Reads from cache → gets wrong email!
```

---

### TTL-Based Expiration (Time-To-Live)

**What it is**: Each cached item is assigned a lifespan. After that time expires, the data is considered stale and is either automatically deleted or must be refreshed.

**How it works**:

```
CACHE ENTRY STRUCTURE:
---------------------
Key: "user:123"
Value: {name: "Alice", email: "alice@example.com"}
TTL: 300 seconds (5 minutes)
Created: 2026-02-06 10:00:00


TIMELINE:
--------
10:00:00 - Data cached with 5-minute TTL
10:03:00 - Data still valid (2 minutes remaining)
10:05:00 - TTL expires, data marked as stale
10:05:01 - Next request triggers cache miss
          -> Fetch fresh data from database
          -> Cache again with new TTL
```

**Characteristics**:
- **Simple to implement**: Just set an expiration time
- **Predictable behavior**: You know exactly when data becomes stale
- **Stale data window**: Data might be outdated before TTL expires
- **No instant updates**: Changes in database won't reflect immediately in cache

**Common TTL patterns**:
- **Short TTL** (seconds to minutes): Frequently changing data (stock prices, social media feeds)
- **Medium TTL** (hours): Semi-static data (user profiles, product details)
- **Long TTL** (days): Rarely changing data (configuration settings, reference data)

**Best for**:
- Data where eventual consistency is acceptable
- When you can predict how "fresh" data needs to be
- Simplicity is more important than perfect accuracy

**Layman explanation**: Like putting an expiration date on milk. After 5 days, you throw it out and buy fresh milk, even if the old milk might still be fine. You accept a trade-off between freshness and convenience.

---

### Event-Driven Invalidation

**What it is**: The cache is invalidated immediately when the underlying data changes, triggered by events from the database or application.

**How it works**:

```
WRITE OPERATION TRIGGERS INVALIDATION:
-------------------------------------
User updates profile in database
    |
    v
Database write succeeds
    |
    v
Publish "user.updated" event
    |
    v
Cache listener receives event
    |
    v
Invalidate cached user data
    |
    v
Next read will fetch fresh data


EVENT FLOW DIAGRAM:
------------------
App --> Write to DB --> Event Bus --> Cache Service
                           |
                           v
                     "user:123 changed"
                           |
                           v
                    Delete cache key "user:123"
```

**Characteristics**:
- **Immediate invalidation**: Cache updates as soon as data changes
- **Strong consistency**: No stale data (assuming events are reliable)
- **Complexity**: Requires event infrastructure (message queues, listeners)
- **Coupling**: Cache system must know about all data change events

**Implementation approaches**:
- **Direct invalidation**: Write code explicitly invalidates cache
- **Database triggers**: Database sends events on UPDATE/DELETE
- **Change Data Capture (CDC)**: Monitor database transaction logs
- **Message queues**: Use systems like Kafka, RabbitMQ for event distribution

**Best for**:
- Applications requiring strong consistency
- When you have reliable event infrastructure
- Critical data that must always be accurate (financial data, inventory)

**Layman explanation**: Like having a smart home where turning off a light immediately updates the status on your phone app. The change is communicated instantly, not on a timer.

---

### Write-Invalidate vs Write-Update

These are two strategies for handling cache when data changes.

**Write-Invalidate** (Delete on Write):

```
WRITE-INVALIDATE FLOW:
---------------------
Data changes in database
    |
    v
DELETE the cache entry
    |
    v
Cache is now empty for that key
    |
    v
Next read will be a MISS
    |
    v
Fetch fresh data from database and cache it
```

**Characteristics**:
- **Simpler**: Just delete the cached item
- **Lazy refresh**: Fresh data is loaded only when needed
- **Cache miss penalty**: Next read is slower
- **Lower write overhead**: Don't need to generate fresh cache data immediately

**Write-Update** (Update on Write):

```
WRITE-UPDATE FLOW:
-----------------
Data changes in database
    |
    v
COMPUTE new cached representation
    |
    v
UPDATE the cache entry with new value
    |
    v
Cache now has fresh data
    |
    v
Next read will be a HIT with correct data
```

**Characteristics**:
- **No cache miss**: Reads remain fast after writes
- **Immediate freshness**: Cache always has latest data
- **Higher write overhead**: Must compute and write cache entry
- **More complex**: Need logic to generate cached value during write

**Comparison**:

```
SCENARIO: Update user profile 10 times, read it 1000 times

Write-Invalidate:
- 10 deletes (fast)
- First read after each write is slow (10 slow reads)
- Remaining 990 reads are fast
- Total: 10 fast deletes + 10 slow reads + 990 fast reads

Write-Update:
- 10 updates (slower than deletes)
- All 1000 reads are fast
- Total: 10 slow updates + 1000 fast reads
```

**When to use**:
- **Write-Invalidate**: Read-to-write ratio is high, cache computation is expensive
- **Write-Update**: Reads must be consistently fast, cache computation is cheap

**Layman explanation**:
- **Write-Invalidate**: When you update a document, you throw away the printed copy. Next time someone needs it, you print a fresh one.
- **Write-Update**: When you update a document, you immediately print a new copy to replace the old one. It's ready for anyone who needs it.

---

### Versioned Keys

**What it is**: Instead of updating or deleting cache entries, you create new cache entries with version numbers in the key. Old versions naturally expire while new versions coexist temporarily.

**How it works**:

```
WITHOUT VERSIONING:
------------------
Key: "product:123"
Value: {name: "Widget", price: 10.00}

Update price to 15.00:
Delete "product:123"
Write "product:123" with new price


WITH VERSIONING:
---------------
Key: "product:123:v1"
Value: {name: "Widget", price: 10.00}

Update price to 15.00:
Write "product:123:v2" with new price
"product:123:v1" expires naturally via TTL


VERSION SCHEME EXAMPLES:
-----------------------
Timestamp-based:   "user:456:20260206100530"
Sequential:        "config:v15"
Hash-based:        "page:/home:abc123def"
Database version:  "post:789:rev42"
```

**Characteristics**:
- **No invalidation needed**: Just write new version, ignore old one
- **Atomic transitions**: All clients see either old or new version, never partial updates
- **Version coordination**: Application must track which version is current
- **Storage overhead**: Multiple versions exist simultaneously until TTL

**Benefits**:
- **No race conditions**: Old and new versions don't interfere
- **Rollback capability**: Can switch back to old version if needed
- **Simplicity**: No complex invalidation logic

**Challenges**:
- **Key management**: Application must know current version number
- **Memory usage**: Old versions consume cache memory until expiration
- **Coordination**: All app instances must agree on current version

**Best for**:
- Configuration data that changes infrequently
- Content that needs atomic updates (entire page HTML)
- Systems where rollback is important
- When race conditions during invalidation are problematic

**Layman explanation**: Instead of erasing and rewriting a whiteboard, you use a new whiteboard for the updated information. The old whiteboard stays up for a while (in case someone still needs it), then gets cleaned automatically later.

---

### Cache Stampede Prevention

**What it is**: A **cache stampede** (also called **thundering herd**) occurs when many requests simultaneously try to regenerate the same expired or missing cache entry, overwhelming the database.

**The Problem**:

```
CACHE STAMPEDE SCENARIO:
-----------------------
Time 0: Popular cache entry expires (1000 requests/sec hitting it)
Time 1: Request #1 arrives, finds cache empty, queries database
Time 1: Request #2 arrives, finds cache empty, queries database
Time 1: Request #3 arrives, finds cache empty, queries database
...
Time 1: Request #1000 arrives, finds cache empty, queries database

Result: 1000 simultaneous database queries for same data!
        Database overloaded, system crashes


NORMAL OPERATION:
----------------
Cache hit ratio: 99.9%
1000 requests/sec × 0.1% miss = 1 database query/sec (manageable)

CACHE STAMPEDE:
--------------
All requests miss simultaneously
1000 requests/sec × 100% miss = 1000 database queries/sec (disaster!)
```

**Solution 1: Locking (Mutex)**

**How it works**: When a cache miss occurs, acquire a lock. Only the first request regenerates the cache; others wait for the lock holder to finish.

```
LOCKING FLOW:
------------
Request 1 --> Cache Miss --> Try Lock --> SUCCESS --> Query DB --> Update Cache --> Release Lock
Request 2 --> Cache Miss --> Try Lock --> WAIT (blocked by Request 1)
Request 3 --> Cache Miss --> Try Lock --> WAIT (blocked by Request 1)
...
Request 1000 --> Cache Miss --> Try Lock --> WAIT

Request 1 completes:
    |
    v
Lock released, cache updated
    |
    v
Waiting requests get lock --> Find data in cache --> Return (no DB query)


IMPLEMENTATION PATTERN:
----------------------
cache_key = "product:123"
lock_key = "lock:product:123"

data = cache.get(cache_key)
if data is None:
    if cache.lock(lock_key, timeout=5):  # Acquire lock
        try:
            data = database.query()      # Only one thread does this
            cache.set(cache_key, data)
        finally:
            cache.unlock(lock_key)       # Release lock
    else:
        # Lock held by another request
        wait_for_lock_release()
        data = cache.get(cache_key)      # Try cache again
```

**Characteristics**:
- **Single database query**: Only lock holder queries database
- **Request waiting**: Other requests blocked until cache is populated
- **Lock timeouts**: Must handle cases where lock holder crashes
- **Complexity**: Distributed locking can be tricky

**Solution 2: Probabilistic Early Expiration**

**How it works**: Randomly refresh cache entries before they expire, with probability increasing as expiration approaches. This spreads out cache misses over time.

```
FORMULA:
-------
current_time = now()
expire_time = cache_entry.expire_time
ttl_remaining = expire_time - current_time
original_ttl = cache_entry.original_ttl

# Calculate probability of early refresh
beta = 1.0  # tuning parameter
delta = ttl_remaining / original_ttl
probability = -beta * log(random(0,1))

if delta * probability >= 1:
    # Refresh cache now (before expiration)
    refresh_cache()


EXAMPLE TIMELINE:
----------------
TTL = 60 seconds, accessed every 1 second

Second 50: 10 seconds remaining, 17% TTL left, ~5% chance of refresh
Second 55: 5 seconds remaining, 8% TTL left, ~15% chance of refresh
Second 59: 1 second remaining, 2% TTL left, ~60% chance of refresh

Result: Cache likely refreshes somewhere between seconds 50-60,
        spreading the load instead of all at second 60
```

**Characteristics**:
- **No locks needed**: Each request independently decides whether to refresh
- **Gradual load**: Refreshes spread over time
- **Some redundant work**: Multiple requests might refresh simultaneously (but fewer)
- **No waiting**: Requests don't block on each other

**Layman explanation**:
- **Locking**: When the milk expires, the first person who notices locks the fridge, goes to the store, and buys more. Everyone else waits for them to return.
- **Probabilistic**: As the milk gets closer to expiring, there's an increasing chance someone will proactively buy more before it actually expires, avoiding everyone rushing to the store at once.

**Best for**:
- **Locking**: Critical data where database overload is unacceptable
- **Probabilistic**: High-traffic scenarios where some redundant work is acceptable
- Often used together: Probabilistic early expiration + lock as fallback

---

### Lease-Based Invalidation

**What it is**: A **lease** is a time-limited guarantee that cached data won't be invalidated. During the lease period, the cache can serve data confidently; after expiration, it must check for updates.

**How it works**:

```
LEASE LIFECYCLE:
---------------
Client requests data
    |
    v
Server grants: Data + Lease (expires in 60 seconds)
    |
    v
Client caches data with lease
    |
    v
For next 60 seconds:
    Client serves from cache (guaranteed valid)
    Server CANNOT invalidate this client's cache
    |
    v
After 60 seconds:
    Client must revalidate with server
    Server may grant new lease or indicate data changed


TIMELINE VIEW:
-------------
T=0s:   Client gets data + 60-second lease
T=30s:  Database updates data
        Server CANNOT invalidate client (lease still valid)
T=60s:  Lease expires
        Client requests: "Do I have latest version?"
        Server: "No, here's new data + new 60-second lease"


LEASE PROTOCOL:
--------------
Request:
  Client -> Server: "Give me user:123"

Response:
  Server -> Client: {
    data: {name: "Alice"},
    version: 42,
    lease_expires: timestamp(+60 seconds)
  }

Revalidation after 60 seconds:
  Client -> Server: "I have user:123 version 42, still valid?"

  If unchanged:
    Server -> Client: "Yes, valid. New lease: +60 seconds"

  If changed:
    Server -> Client: "No, here's version 43 + new lease"
```

**Characteristics**:
- **Guaranteed validity window**: Cache knows data is correct for lease duration
- **Delayed invalidation**: Changes don't propagate until leases expire
- **Reduced revalidation**: Only check server when lease expires
- **Server tracking**: Server must track outstanding leases (can be complex)

**Lease duration trade-offs**:

```
SHORT LEASES (seconds):
+ Fresher data
+ Changes propagate quickly
- More server revalidation requests
- Higher network overhead

LONG LEASES (minutes/hours):
+ Fewer server requests
+ Better caching efficiency
- Staler data
- Longer delay for updates to propagate
```

**Advanced: Revocable leases**

Some systems allow the server to revoke leases early when critical updates occur:

```
REVOCATION FLOW:
---------------
T=0s:  Client gets lease valid until T=60s
T=20s: Critical update happens
       Server sends revocation message to client
       Client invalidates cache immediately
```

**Best for**:
- Systems with predictable data change patterns
- When freshness requirements vary by time of day
- Mobile apps with intermittent connectivity (long leases for offline use)
- Content Distribution Networks (CDNs)

**Layman explanation**: Like renting a car for 3 days. For those 3 days, you're guaranteed to have the car—the rental company can't take it back. After 3 days, you return it and they can give you a different car if needed. This is simpler than constantly calling to check if your car is still available.

---

## Distributed Caching

**Distributed caching** means spreading cached data across multiple servers (cache nodes) rather than keeping everything on a single machine. This provides higher capacity, better performance, and fault tolerance.

```
SINGLE CACHE:                 DISTRIBUTED CACHE:
------------                  ------------------
   [App] --> [Cache]            [App] ---------> [Cache Node 1]
              |                     |-------> [Cache Node 2]
              v                     |-------> [Cache Node 3]
          [Database]                |-------> [Cache Node 4]
                                    |
                                    v
                                [Database]
```

**Why distribute cache?**
- **Capacity**: More total memory than one machine can provide
- **Throughput**: Distribute request load across multiple servers
- **Fault tolerance**: If one cache node fails, others continue working
- **Geographic distribution**: Cache nodes closer to users in different regions

---

### Redis: Data Structures, Clustering, Persistence

**Redis** is an advanced in-memory data store that supports rich data structures beyond simple key-value pairs. It's widely used for caching, session storage, real-time analytics, and message queues.

**Data Structures** (layman: different ways to organize your data):

```
STRING (simplest):
-----------------
SET user:123:name "Alice"
GET user:123:name  --> "Alice"


HASH (like a mini-database record):
-----------------------------------
HSET user:123 name "Alice" email "alice@example.com" age "30"
HGET user:123 email  --> "alice@example.com"
HGETALL user:123  --> {name: "Alice", email: "alice@example.com", age: "30"}


LIST (ordered collection):
--------------------------
LPUSH notifications:user:123 "New message from Bob"
LPUSH notifications:user:123 "Your order shipped"
LRANGE notifications:user:123 0 -1  --> ["Your order shipped", "New message from Bob"]


SET (unique items):
------------------
SADD tags:article:456 "redis" "caching" "performance"
SMEMBERS tags:article:456  --> ["redis", "caching", "performance"]


SORTED SET (ordered by score):
------------------------------
ZADD leaderboard 1500 "Alice" 1200 "Bob" 1800 "Charlie"
ZRANGE leaderboard 0 -1 WITHSCORES  --> [Bob(1200), Alice(1500), Charlie(1800)]
```

**Why data structures matter for caching**:
- **Partial updates**: Update single field in hash without rewriting entire object
- **Efficient operations**: Increment counters, add to lists, check set membership
- **Memory efficiency**: Optimized internal representations
- **Complex queries**: Range queries, set intersections, sorted retrieval

**Redis Clustering** (distributing data across multiple Redis instances):

```
CLUSTER ARCHITECTURE:
--------------------
       Client
          |
          v
    Cluster Router
          |
    +-----+-----+-----+
    |     |     |     |
    v     v     v     v
  Node1 Node2 Node3 Node4
  (0-4k) (4k-8k) (8k-12k) (12k-16k)

Redis divides keyspace into 16,384 "hash slots"
Each key is hashed to determine its slot
Each node owns a range of slots
```

**How clustering works**:
1. **Hash slot assignment**: Each key is hashed using CRC16, result modulo 16,384 determines slot
2. **Node responsibility**: Slots distributed among nodes (e.g., 4 nodes = ~4,096 slots each)
3. **Client routing**: Client calculates slot, sends request to correct node
4. **Resharding**: Can move slots between nodes to rebalance

**Redis Persistence** (saving data to disk):

Redis is in-memory, but offers persistence options to survive restarts:

```
RDB (Redis Database Backup):
---------------------------
Periodic snapshots of entire dataset
"Save every 5 minutes if 100+ keys changed"

+ Fast restarts (load snapshot)
+ Compact files
- Data loss possible (between snapshots)
- Can be slow for large datasets


AOF (Append-Only File):
----------------------
Log of every write operation
Replay log on restart to reconstruct data

+ Minimal data loss (logs every write)
+ Can be configured for different durability levels
- Larger files
- Slower restarts


HYBRID (RDB + AOF):
------------------
RDB snapshots + AOF for recent changes
Best of both: fast restarts + minimal data loss
```

**When to use Redis**:
- Complex caching needs (partial updates, counters, lists)
- Session storage with rich data
- Real-time analytics (leaderboards, statistics)
- When persistence is needed
- When you need Pub/Sub messaging

---

### Memcached: Simple Key-Value, Multi-Threaded

**Memcached** is a simple, high-performance, distributed memory caching system. It focuses on doing one thing well: fast key-value storage.

**Characteristics**:

```
MEMCACHED MODEL:
---------------
Key: String (max 250 bytes)
Value: Blob (max 1 MB)
Operations: SET, GET, DELETE, INCREMENT, DECREMENT

That's it. Simple.


ARCHITECTURE:
------------
Multi-threaded server
Each thread handles client connections
Lock-free algorithms for speed
Slab allocator for memory management
```

**Multi-threading advantage**:
- **Utilizes all CPU cores**: Can handle many simultaneous requests
- **No GIL (Global Interpreter Lock)**: True parallelism
- **High throughput**: Designed for millions of operations per second

**Memory management** (slab allocation):

```
SLAB CLASSES:
------------
Class 1: 96-byte chunks   (small items)
Class 2: 120-byte chunks
Class 3: 152-byte chunks
...
Class 42: 1 MB chunks     (large items)

When you store 100 bytes:
- Memcached finds smallest fitting slab (120-byte class)
- Allocates a chunk from that class
- Wastes 20 bytes (trade-off for speed)
```

**Why slab allocation?**
- **No fragmentation**: Memory organized in fixed-size chunks
- **Fast allocation**: No searching for "best fit" space
- **Predictable performance**: No garbage collection pauses

**Redis vs Memcached comparison**:

```
REDIS:
+ Rich data structures
+ Persistence options
+ Transactions and Lua scripting
+ Pub/Sub messaging
+ Single-threaded (simpler, but less CPU utilization)
- More complex
- Higher memory overhead

MEMCACHED:
+ Simpler, easier to understand
+ Multi-threaded (better CPU utilization)
+ Lower memory overhead
+ Mature and stable
- Only key-value storage
- No persistence
- No complex data types
```

**When to use Memcached**:
- Simple caching needs (just key-value)
- Very high throughput requirements
- Want to utilize multi-core CPUs fully
- Minimal operational complexity
- Don't need persistence

**Layman explanation**: Redis is like a Swiss Army knife—lots of tools for different jobs. Memcached is like a hammer—does one thing extremely well. Choose based on whether you need versatility or raw speed for simple tasks.

---

### Cache Sharding Strategies

**Sharding** means dividing your cache data across multiple cache servers. The question is: how do you decide which server stores which data?

**Strategy 1: Modulo Hashing (Simple but Problematic)**

```
HOW IT WORKS:
------------
hash(key) % number_of_servers = server_index

Example with 3 servers:
hash("user:123") % 3 = 2  --> Server 2
hash("user:456") % 3 = 0  --> Server 0
hash("user:789") % 3 = 1  --> Server 1


DISTRIBUTION:
------------
[Server 0] <-- keys with hash % 3 = 0
[Server 1] <-- keys with hash % 3 = 1
[Server 2] <-- keys with hash % 3 = 2
```

**The Problem**: Adding or removing servers destroys the distribution:

```
WITH 3 SERVERS:
hash("user:123") % 3 = 2  --> Server 2

ADDING 4TH SERVER (now 4 total):
hash("user:123") % 4 = 1  --> Server 1 (DIFFERENT!)

Result: Most keys map to different servers
        Cache completely invalidated
        Massive database load spike
```

**When servers change, ~(N/(N+1)) of all cached data becomes invalid**
- 3 to 4 servers: 75% cache invalidation
- 10 to 11 servers: 91% cache invalidation

**Strategy 2: Range-Based Sharding**

```
PARTITION BY KEY RANGE:
----------------------
Server 0: keys starting A-H
Server 1: keys starting I-P
Server 2: keys starting Q-Z


OR BY USER ID:
-------------
Server 0: user IDs 1-1,000,000
Server 1: user IDs 1,000,001-2,000,000
Server 2: user IDs 2,000,001-3,000,000
```

**Advantages**:
- Predictable: Easy to know which server has what
- Range queries possible: "Get all users 1000-2000" hits one server

**Disadvantages**:
- **Hot spots**: Some ranges might be accessed more (users 1-1M might be older, less active)
- **Imbalanced**: Data might not distribute evenly
- **Resharding complex**: Must carefully plan new ranges

**Strategy 3: Consistent Hashing (Best Practice)**

Consistent hashing solves the problem of server changes causing massive cache invalidation. See next section for details.

**When to use each**:
- **Modulo**: Small, static clusters that never change size (rare in practice)
- **Range-based**: When range queries are important, data distribution is predictable
- **Consistent hashing**: Default choice for distributed caches (most common)

---

### Consistent Hashing for Cache Distribution

**Consistent hashing** is a technique that minimizes cache invalidation when servers are added or removed. Instead of rehashing all keys, only a small fraction needs to be remapped.

**How it works**:

```
HASH RING CONCEPT:
-----------------
Imagine a circular ring of hash values from 0 to 2^32 - 1

        0
        |
  2^32-1+----> 1
    |             |
    |    RING     |
    |             |
2^31 ----------- 2^30
```

**Step 1**: Hash server names onto the ring

```
hash("Server A") = 100  --> Position 100
hash("Server B") = 500  --> Position 500
hash("Server C") = 900  --> Position 900

RING:
    0
    |
    100 (Server A)
    |
    500 (Server B)
    |
    900 (Server C)
    |
    2^32-1
```

**Step 2**: Hash each key and find the next server clockwise

```
hash("user:123") = 250  --> Next server clockwise: Server B (500)
hash("user:456") = 600  --> Next server clockwise: Server C (900)
hash("user:789") = 950  --> Next server clockwise: Server A (100, wraps around)


VISUAL:
------
    0
    |
    100 [Server A] <-- user:789 (hash=950, wraps)
    |
    250 (user:123) |
    |              v
    500 [Server B]
    |
    600 (user:456) |
    |              v
    900 [Server C]
    |
    950 (user:789) |
    |              |
    2^32-1---------+
```

**Adding a server**:

```
ADD Server D at position 700:

BEFORE:
user:456 (hash=600) --> Server C (900)

AFTER:
user:456 (hash=600) --> Server D (700, new closest)

IMPACT:
Only keys between 600-700 are affected
All other keys stay on same servers!
```

**Mathematical advantage**:

```
With N servers, adding 1 new server:
- Modulo hashing: ~N/(N+1) keys remapped (~75-95%)
- Consistent hashing: ~1/(N+1) keys remapped (~10-25%)

Example with 10 servers, add 1:
- Modulo: ~91% of keys need new server
- Consistent: ~9% of keys need new server
```

**Virtual Nodes** (solving uneven distribution):

Real-world problem: With only 3 servers, distribution might be uneven:

```
BAD DISTRIBUTION:
----------------
Server A at position 100
Server B at position 200  --> Only 100 units apart
Server C at position 900  --> 700 units apart

Server B gets very few keys (small arc)
Server C gets many keys (large arc)
```

Solution: Create multiple "virtual nodes" per physical server:

```
VIRTUAL NODES:
-------------
Server A: hash("A-1")=100, hash("A-2")=400, hash("A-3")=800
Server B: hash("B-1")=200, hash("B-2")=500, hash("B-3")=950
Server C: hash("C-1")=300, hash("C-2")=600, hash("C-3")=850

Now each server has 3 positions on ring
Better distribution: each server gets ~1/3 of keys
```

Typical practice: 100-200 virtual nodes per physical server

**Layman explanation**: Imagine a circular racetrack. Servers are positioned at certain mile markers. When a runner (data key) starts at their position, they run clockwise until they reach the next server—that's where they're stored. If you add a new server at mile 7, only runners between miles 5-7 need to go to this new server instead of the old one at mile 10. Everyone else is unaffected.

---

### Cache Replication vs Partitioning

Two fundamental approaches to distributing cache: **replication** (copy all data everywhere) vs **partitioning** (divide data among servers).

**Replication** (Master-Slave / Leader-Follower):

```
ARCHITECTURE:
------------
         Write
           |
           v
    [Master Cache]
           |
    +------+------+
    |      |      |
    v      v      v
[Slave] [Slave] [Slave]
    |      |      |
    v      v      v
  Read   Read   Read


DATA DISTRIBUTION:
-----------------
Master: {user:123, user:456, user:789, product:1, product:2}
Slave1: {user:123, user:456, user:789, product:1, product:2} (COPY)
Slave2: {user:123, user:456, user:789, product:1, product:2} (COPY)
Slave3: {user:123, user:456, user:789, product:1, product:2} (COPY)
```

**Characteristics**:
- **All nodes have all data**: Complete copy on each server
- **Read scaling**: Distribute read load across all replicas
- **Write bottleneck**: All writes go through master
- **High availability**: If master fails, promote a slave
- **Memory cost**: Total capacity = capacity of one node (wasteful)
- **Consistency challenge**: Slaves might lag behind master

**Replication delay** (time for slaves to sync with master):

```
TIMELINE:
--------
T=0ms:  Write to master: user:123 email = "new@email.com"
T=5ms:  Slave1 syncs (now has new email)
T=8ms:  Slave2 syncs (now has new email)
T=12ms: Slave3 syncs (now has new email)

Problem: Between T=0 and T=12ms, reading from Slave3 gives old data!
```

**Partitioning** (Sharding):

```
ARCHITECTURE:
------------
       Client
          |
    +-----+-----+
    |     |     |
    v     v     v
  Node1 Node2 Node3

DATA DISTRIBUTION:
-----------------
Node1: {user:123, product:1}      (Shard 1)
Node2: {user:456, product:2}      (Shard 2)
Node3: {user:789}                 (Shard 3)

Each node has DIFFERENT data
Total capacity = sum of all nodes
```

**Characteristics**:
- **Distributed data**: Each node stores portion of dataset
- **Linear scaling**: Add more nodes = more total capacity
- **Fault tolerance issue**: If one node fails, that data is unavailable
- **Complex queries**: Queries touching multiple partitions need aggregation
- **Write scaling**: Writes distributed across nodes

**Comparison**:

```
SCENARIO: 1 TB of cache data, 10,000 requests/second

REPLICATION:
- 4 nodes × 1 TB each = 4 TB total memory needed
- Read capacity: 10,000 req/s ÷ 4 nodes = 2,500 req/s per node
- Write capacity: All writes to master = bottleneck
- Fault tolerance: 3 replicas can fail, 1 must survive
- Use case: Read-heavy (90%+ reads)

PARTITIONING:
- 4 nodes × 250 GB each = 1 TB total memory needed (4x more efficient!)
- Read capacity: 10,000 req/s ÷ 4 nodes = 2,500 req/s per node
- Write capacity: Distributed across nodes = 2,500 writes/s per node
- Fault tolerance: Any single node failure loses 25% of data
- Use case: Balanced read/write, need large total capacity
```

**Hybrid Approach** (Partitioning + Replication):

Most production systems use both:

```
PARTITIONED WITH REPLICATION:
----------------------------
Partition 1: [Master1] + [Slave1a] + [Slave1b]
Partition 2: [Master2] + [Slave2a] + [Slave2b]
Partition 3: [Master3] + [Slave3a] + [Slave3b]

Each partition is replicated 3x
- Capacity: Sum of partition sizes (like pure partitioning)
- Read scaling: Distribute reads among replicas
- Write scaling: Distribute writes among partitions
- Fault tolerance: Can lose 2 nodes per partition
```

**When to use**:
- **Replication only**: Small dataset, extremely read-heavy, need high availability
- **Partitioning only**: Large dataset, balanced read/write, okay with node failure impact
- **Hybrid**: Production systems with large datasets and high availability requirements

**Layman explanation**:
- **Replication**: Like having 4 identical filing cabinets. Great if many people need to look things up, but you need 4x the space.
- **Partitioning**: Like splitting your files across 4 cabinets by first letter (A-F, G-M, N-S, T-Z). Uses space efficiently, but losing one cabinet loses that section.
- **Hybrid**: Each letter range has multiple backup cabinets. Best of both, but most complex.

---

## CDN and Edge Caching

**CDN** (Content Delivery Network) and **edge caching** bring content geographically closer to users by caching it at locations distributed around the world. This dramatically reduces latency (response time) for global users.

```
WITHOUT CDN:                      WITH CDN:
-----------                       --------
User in Tokyo                     User in Tokyo
    |                                 |
    | 150ms                           | 10ms
    v                                 v
Server in New York               Edge Server in Tokyo
                                      |
                                      | 150ms (cache miss only)
                                      v
                                 Origin Server in New York
```

**Latency impact**:
- Same continent: ~20-50ms
- Cross-continent: ~100-300ms
- Around the world: ~300-500ms

A page load requiring 20 resources from New York to Tokyo: 20 × 200ms = 4 seconds just in network delay!

---

### CDN Architecture and PoPs

**PoP** (Point of Presence) is a physical location where a CDN has servers. Major CDNs have 100-300+ PoPs worldwide.

**CDN Architecture**:

```
GLOBAL DISTRIBUTION:
-------------------
        [Origin Server]
         (New York)
              |
    +---------+---------+
    |         |         |
    v         v         v
[PoP Tokyo] [PoP London] [PoP Sydney]
    |         |         |
    v         v         v
 Users in   Users in   Users in
  Asia      Europe     Australia


EDGE SERVER LOCATIONS (example):
-------------------------------
North America: 80 PoPs
Europe: 60 PoPs
Asia: 50 PoPs
South America: 15 PoPs
Africa: 10 PoPs
Oceania: 10 PoPs
```

**Request flow**:

```
STEP 1: DNS Resolution
----------------------
User requests: www.example.com
    |
    v
DNS returns IP of NEAREST edge server
    |
    v
Tokyo user gets: Tokyo PoP IP
London user gets: London PoP IP


STEP 2: Edge Server Handling
----------------------------
User --> Edge Server
         |
         v
    Check local cache
         |
    +----+----+
    |         |
  [HIT]     [MISS]
    |         |
    v         v
 Return    Fetch from Origin
 cached    |
 content   v
          Cache locally
           |
           v
          Return to user
```

**Cache hierarchy** (multi-tier CDN):

```
           [Origin Server]
                 |
                 v
         [Regional Cache]
         (Mid-tier PoP)
                 |
        +--------+--------+
        |        |        |
        v        v        v
   [Edge 1]  [Edge 2]  [Edge 3]
   (Tokyo)   (Osaka)   (Seoul)

Typical flow:
1. Edge cache miss --> Check regional cache
2. Regional cache miss --> Fetch from origin
3. Both edge and regional cache the response
```

**Benefits of multi-tier**:
- **Reduced origin load**: Regional cache shields origin from most requests
- **Faster cache fill**: Edge servers fetch from nearby regional cache
- **Bandwidth optimization**: Origin sends data once to region, not to every edge

---

### Cache-Control Headers

**Cache-Control** is an HTTP header that tells caches (CDN, browser, proxies) how to handle a response. It's the primary mechanism for controlling cache behavior.

**Basic syntax**:

```
Cache-Control: directive1, directive2, directive3


EXAMPLE HEADERS:
---------------
Cache-Control: public, max-age=3600
Cache-Control: private, no-cache
Cache-Control: public, max-age=86400, s-maxage=3600
Cache-Control: no-store
```

**Common directives**:

**public vs private**:

```
public:
- Anyone can cache (CDN, browser, proxy)
- Use for: Images, CSS, JS, public web pages

private:
- Only browser can cache, not CDN/proxy
- Use for: User-specific data (dashboard, settings)
- CDN will NOT cache responses marked private


EXAMPLE:
-------
GET /user/profile
Response: Cache-Control: private, max-age=300

Browser caches for 5 minutes
CDN does NOT cache (private)
```

**max-age** (time in seconds):

```
Cache-Control: max-age=3600

Meaning: Response is fresh for 3600 seconds (1 hour)
After 1 hour: Must revalidate with server


COMMON VALUES:
-------------
max-age=0        No caching (always revalidate)
max-age=60       Cache for 1 minute
max-age=3600     Cache for 1 hour
max-age=86400    Cache for 1 day
max-age=31536000 Cache for 1 year (immutable assets)
```

**s-maxage** (CDN-specific max-age):

```
Cache-Control: max-age=60, s-maxage=3600

Browser caches for 60 seconds
CDN caches for 3600 seconds (1 hour)

Use case: Serve slightly stale content from CDN for performance,
          but keep browser cache shorter for better freshness
```

**no-cache vs no-store**:

```
no-cache:
- CAN cache, but MUST revalidate before using
- Sends conditional request to origin
- If unchanged: 304 Not Modified, use cached version
- If changed: 200 OK, update cache

no-store:
- DO NOT cache at all
- Every request must go to origin
- Use for: Sensitive data, always-changing content


EXAMPLE WITH no-cache:
---------------------
GET /news/latest
Response: Cache-Control: no-cache, ETag: "abc123"

Next request:
GET /news/latest
Header: If-None-Match: "abc123"

If content unchanged:
Response: 304 Not Modified (fast, no body sent)

If content changed:
Response: 200 OK, ETag: "def456" (full content sent)
```

**must-revalidate**:

```
Cache-Control: max-age=3600, must-revalidate

After 1 hour:
- MUST contact origin before serving stale content
- If origin unreachable: Return error, NOT stale content

Without must-revalidate:
- Some caches might serve stale content if origin is down
```

**immutable**:

```
Cache-Control: public, max-age=31536000, immutable

Meaning: This file will NEVER change
Browser won't revalidate even on refresh (F5)

Use for: Versioned assets (style.abc123.css)
Never use for: Unversioned URLs (/style.css)
```

**Practical examples**:

```
STATIC ASSETS (versioned):
-------------------------
style.v123.css
Cache-Control: public, max-age=31536000, immutable
Reason: File name changes when content changes, cache forever


HTML PAGES:
----------
index.html
Cache-Control: public, max-age=300, s-maxage=3600
Reason: Browser checks every 5 min, CDN every hour


USER-SPECIFIC DATA:
------------------
/api/user/profile
Cache-Control: private, max-age=60
Reason: Only browser caches, not CDN


SENSITIVE DATA:
--------------
/api/payment/details
Cache-Control: no-store
Reason: Never cache payment information


FREQUENTLY UPDATED:
------------------
/api/stock-price
Cache-Control: public, max-age=10, s-maxage=10
Reason: Cache for 10 seconds only, updates quickly
```

---

### Edge Caching Strategies

**Edge caching strategies** define what content is cached at edge servers and how cache freshness is maintained.

**Strategy 1: Static Asset Caching**

```
CONTENT TYPES:
-------------
Images: .jpg, .png, .gif, .webp
Stylesheets: .css
Scripts: .js
Fonts: .woff, .woff2
Videos: .mp4, .webm

STRATEGY:
--------
Cache-Control: public, max-age=31536000, immutable
Versioned URLs: /assets/style.v123.css

When file changes:
- Deploy with new version: style.v124.css
- HTML references new version
- Old version stays cached (no harm)
- New version cached on first request
```

**Strategy 2: Dynamic Content with Short TTL**

```
CONTENT:
-------
API responses
News articles
Product listings

STRATEGY:
--------
Cache-Control: public, max-age=60, s-maxage=300

Edge caches for 5 minutes
Balances freshness vs performance

Alternative: Stale-while-revalidate
Cache-Control: max-age=60, stale-while-revalidate=300

- Serve from cache if < 60 seconds old
- If 60-360 seconds old: Serve stale, async refresh cache
- If > 360 seconds old: Wait for refresh, then serve
```

**Strategy 3: Personalized Content Caching** (Vary header):

```
PROBLEM:
-------
Same URL, different content per user:
/homepage --> Shows personalized recommendations

Can't cache naively: Would show User A's homepage to User B!


SOLUTION - Vary header:
----------------------
Response headers:
Cache-Control: public, max-age=300
Vary: Cookie

Meaning: Cache separate versions based on Cookie header

Cache storage:
Key: URL + Cookie value
- /homepage + Cookie=user_a_token --> Version for User A
- /homepage + Cookie=user_b_token --> Version for User B


COMMON VARY VALUES:
------------------
Vary: Accept-Encoding (cache gzip and brotli separately)
Vary: User-Agent (cache mobile and desktop versions)
Vary: Accept-Language (cache per language)
Vary: Cookie (cache per user session)
```

**WARNING**: Too many Vary values fragment cache:

```
Vary: User-Agent, Accept-Language, Accept-Encoding

Possible combinations:
10 browsers × 20 languages × 3 encodings = 600 cache versions!

Each URL might have 600 separate cached copies
Cache efficiency drops dramatically
```

**Strategy 4: Cache Key Customization**

Some CDNs allow custom cache keys to cache more aggressively:

```
EXAMPLE - Ignore certain query parameters:
-----------------------------------------
Request: /product?id=123&utm_source=email&session=xyz

Default cache key:
/product?id=123&utm_source=email&session=xyz
(Every marketing email creates new cache entry!)

Custom cache key (ignore utm_*, session):
/product?id=123
(All requests for same product hit same cache)


EXAMPLE - Normalize request:
----------------------------
/api/users?sort=name&limit=10
/api/users?limit=10&sort=name  (same query, different order)

Default: Two separate cache entries
Custom: Normalize to canonical order --> one cache entry
```

---

### Origin Shield Pattern

**Origin shield** is an additional caching layer between edge servers and the origin that consolidates requests, protecting the origin from request spikes.

```
WITHOUT ORIGIN SHIELD:
---------------------
[Edge Tokyo] ---->  |
[Edge London] ---->  |---> [Origin Server]
[Edge Sydney] ---->  |
[Edge NYC] -------->  |

If all edges have cache miss simultaneously:
4 requests hit origin at once


WITH ORIGIN SHIELD:
------------------
[Edge Tokyo] ---->  |
[Edge London] ---->  |---> [Origin Shield] ---> [Origin Server]
[Edge Sydney] ---->  |
[Edge NYC] -------->  |

If all edges have cache miss:
- All 4 edges request from Origin Shield
- Origin Shield makes 1 request to Origin
- Result served to all 4 edges
```

**How it works**:

```
CACHE MISS FLOW:
---------------
1. User in Tokyo requests /video.mp4
2. Edge Tokyo cache miss
3. Edge Tokyo requests from Origin Shield
4. Origin Shield checks its cache:

   HIT:
   - Return cached copy to Edge Tokyo
   - Origin never contacted

   MISS:
   - Request from Origin
   - Cache response
   - Return to Edge Tokyo

5. Edge Tokyo caches and serves to user
```

**Request coalescing** (deduplication):

```
SCENARIO:
--------
Time 0ms:  Edge Tokyo requests /popular-video.mp4 from shield
Time 5ms:  Edge London requests /popular-video.mp4 from shield
Time 8ms:  Edge Sydney requests /popular-video.mp4 from shield

WITHOUT COALESCING:
- 3 simultaneous requests to origin
- 3x bandwidth usage
- 3x origin load

WITH COALESCING:
- Shield fetches once from origin
- While fetching, queues subsequent requests
- When fetch completes, serves all 3 edges
- Result: 1 origin request instead of 3
```

**Benefits**:
- **Reduced origin load**: Especially during cache invalidation/purge
- **Lower bandwidth costs**: Origin sends data once, not N times
- **Better cache hit ratio**: Larger aggregated cache
- **Protection from thundering herd**: Natural request coalescing

**Trade-offs**:
- **Additional latency**: Extra network hop (edge -> shield -> origin)
- **Single point of failure**: Shield outage affects all edges
- **More complex**: Additional infrastructure to manage

**When to use**:
- Origin has limited capacity or expensive to scale
- Serving very large files (videos, downloads)
- Expecting traffic spikes (viral content, marketing campaigns)
- Origin charges by bandwidth

---

### Cache Purging and Invalidation

**Cache purging** (also called **cache invalidation** or **cache clearing**) forcibly removes content from CDN edge caches before natural expiration.

**Why purge?**
- Content updated on origin (news article edited)
- Critical bug fix in JavaScript
- Removed sensitive data accidentally published
- Security incident requiring immediate content removal

**Purge Methods**:

**1. Purge by URL** (Exact match):

```
Purge: https://example.com/images/logo.png

Result:
- Removes /images/logo.png from all edge servers
- Other URLs unaffected
- Next request fetches fresh copy from origin


API EXAMPLE:
-----------
POST /purge
{
  "files": [
    "https://example.com/images/logo.png",
    "https://example.com/styles/main.css"
  ]
}
```

**2. Purge by Wildcard/Pattern**:

```
Purge: /images/*

Result:
- Removes all files in /images/ directory
- /images/logo.png ✓
- /images/icons/home.png ✓
- /styles/main.css ✗ (not affected)


WARNING: Can purge massive amounts of content
Purging /* = empty entire cache (use carefully!)
```

**3. Purge by Tag/Key**:

```
TAG ASSIGNMENT (in origin response):
-----------------------------------
Response for /product/123:
Cache-Tag: product-123, category-shoes, brand-nike

Response for /product/124:
Cache-Tag: product-124, category-shoes, brand-adidas


PURGE BY TAG:
------------
Purge tag: "category-shoes"
Result: Both product-123 and product-124 purged

Use case:
- Edit shoe category description
- Purge all pages tagged "category-shoes"
- No need to track every product URL
```

**4. Soft Purge (Revalidation)**:

```
HARD PURGE:
----------
- Immediately deletes cached content
- Next request: Full origin fetch (slow)
- Origin sees spike in requests

SOFT PURGE:
----------
- Marks content as stale (not deleted)
- Next request: Conditional request to origin
  - If unchanged: 304 Not Modified, serve cached version
  - If changed: 200 OK, update cache
- Less origin load
- Faster for unchanged content


EXAMPLE:
-------
Content rarely changes but purge frequently for freshness

Hard purge: 1000 requests/sec × full content = heavy origin load
Soft purge: 1000 requests/sec × 304 responses = light origin load
```

**Propagation time** (how long purge takes):

```
TYPICAL TIMELINE:
----------------
T=0s:   Purge request submitted
T=1s:   Purge starts distributing to edge servers
T=5s:   50% of edges purged
T=10s:  95% of edges purged
T=30s:  99.9% of edges purged

Users might see old content for 5-30 seconds after purge
For truly instant updates: Use short TTL instead of purging
```

**Best Practices**:

```
DON'T PURGE FOR EVERY UPDATE:
----------------------------
Bad: max-age=86400, purge on every change
Good: max-age=300 (5 minutes), no purging needed


USE VERSIONED URLS INSTEAD:
--------------------------
Bad:
  /style.css (must purge on changes)

Good:
  /style.v123.css (no purge needed, change URL)


PURGE GRANULARLY:
----------------
Bad:  Purge /* (clears everything)
Good: Purge /api/users/* (specific section)


COMBINE WITH TAGS:
-----------------
Tag all product pages with product-{id}
Single purge when product updated
```

---

### Dynamic Content Caching

**Dynamic content** is personalized or generated per request (user dashboards, search results, API responses). Traditionally considered "uncacheable," but modern techniques enable caching dynamic content.

**Challenge**: Same URL, different content for different users/contexts

```
PROBLEM:
-------
URL: /search?q=hotels
User A sees: Hotels in New York (based on their location)
User B sees: Hotels in London (based on their location)

Naive caching: User B might see User A's results!
```

**Technique 1: Cache Key Customization**

```
INCLUDE RELEVANT CONTEXT IN CACHE KEY:
-------------------------------------
Default cache key: URL only
/search?q=hotels

Custom cache key: URL + User location
/search?q=hotels + user_country=US
/search?q=hotels + user_country=UK

Result: Separate cache entries per country
Personalized content cached safely
```

**Technique 2: Edge Side Includes (ESI)**

```
CONCEPT:
-------
Split page into cacheable and uncacheable parts
Cache static parts, fetch dynamic parts per request


EXAMPLE - E-commerce homepage:
-----------------------------
<html>
  <body>
    <header>
      <!-- Static, cache for 1 hour -->
    </header>

    <esi:include src="/api/recommendations?user=$user_id" />
    <!-- Dynamic, fetch per request -->

    <footer>
      <!-- Static, cache for 1 day -->
    </footer>
  </body>
</html>


EXECUTION:
---------
1. Edge server receives request
2. Returns cached HTML template
3. Sees <esi:include> tag
4. Fetches /api/recommendations for current user
5. Inserts dynamic content into template
6. Returns complete page

Result: 80% cached (header/footer), 20% fresh (recommendations)
```

**Technique 3: Stale-While-Revalidate**

```
Cache-Control: max-age=10, stale-while-revalidate=86400

Timeline:
--------
T=0s:   Cache entry created
T=9s:   Request arrives, content fresh, serve immediately
T=15s:  Request arrives, content stale but < 24h old:
        - Serve stale content immediately (fast)
        - Asynchronously fetch fresh content from origin
        - Update cache for next request

T=1day: Request arrives, content too old:
        - Wait for fresh fetch
        - Then serve and cache


BENEFIT:
-------
Always fast responses (serve stale immediately)
Content stays reasonably fresh (async updates)
Origin sees smooth, predictable load (not spiky)
```

**Technique 4: Segmented Caching**

```
SEGMENT USERS INTO GROUPS:
-------------------------
Instead of caching per user (millions of cache entries),
cache per user segment (dozens of cache entries)

Example - News site:
- Free users: Generic content
- Premium users: Full articles
- Ad-blockers: Different ad layout

Cache keys:
/article/123 + segment=free
/article/123 + segment=premium
/article/123 + segment=adblocker

Result: 3 cache entries instead of millions
```

**Technique 5: Client-Side Personalization**

```
APPROACH:
--------
1. Server caches generic content
2. Client (browser/app) fetches personalization data
3. Client merges generic + personalized


EXAMPLE - Product listing:
-------------------------
Server response (cached at CDN):
{
  products: [
    {id: 1, name: "Shoes", price: 50},
    {id: 2, name: "Shirt", price: 30}
  ]
}

Client fetches (not cached):
{
  user_favorites: [1],
  user_cart: [2]
}

Client renders:
- Shoes ⭐ (favorite)  $50
- Shirt 🛒 (in cart)   $30

Result: Product list cached, personalization fresh
```

**When to use each**:

```
CACHE KEY CUSTOMIZATION:
- Limited personalization dimensions (language, country)
- Content varies by request headers/cookies

ESI:
- Mix of static and dynamic sections
- Compatible CDN (not all support ESI)

STALE-WHILE-REVALIDATE:
- Acceptable to show slightly old content
- Content changes gradually, not critically

SEGMENTED CACHING:
- Large user base fits into few segments
- Privacy not a concern (users in same segment see same content)

CLIENT-SIDE:
- Personalization is additive (badges, stars)
- Base content same for everyone
```

**Layman explanation**: Dynamic content caching is like printing a newspaper with some sections left blank, then handwriting the personalized parts (name, custom ads) for each subscriber. You cache the expensive printing, customize the cheap handwriting.

---

## Cache Consistency Models

**Consistency** in caching refers to how closely cached data matches the source data (database/origin server). Different consistency models make different trade-offs between performance and accuracy.

```
THE CONSISTENCY SPECTRUM:
------------------------
Strong              Eventual           No
Consistency         Consistency        Consistency
|                   |                  |
Always              Eventually         May be
correct             correct            wrong
|                   |                  |
Slow                Fast               Fastest
```

---

### Strong Consistency (Single Cache)

**What it is**: Cached data is always identical to the source data. Any read always returns the most recent write. This is typically achievable only with a single cache instance.

**How it works**:

```
WRITE FLOW:
----------
1. Write to database
2. Immediately invalidate/update cache
3. Confirm write only after both complete

READ FLOW:
---------
1. Read from cache
2. Cache guaranteed to have latest data

GUARANTEE:
---------
If write completes at T=5s,
any read at T=5.001s sees the new value
No windows of inconsistency
```

**Implementation patterns**:

**Pattern 1: Synchronous cache update**

```
ATOMIC OPERATION:
----------------
BEGIN TRANSACTION
  UPDATE database SET email = 'new@email.com' WHERE user_id = 123
  DELETE cache_key('user:123')
COMMIT TRANSACTION

If either fails, both rollback
Cache always matches database
```

**Pattern 2: Write-through cache**

```
APPLICATION WRITE:
-----------------
user.email = "new@email.com"

write_to_cache('user:123', user)  # Synchronous
write_to_database('user:123', user)  # Synchronous

Both must succeed before returning
Cache and database always in sync
```

**Characteristics**:
- **Perfect accuracy**: Zero stale reads
- **Slower writes**: Must wait for both database and cache
- **Single point of access**: Works with single cache instance only
- **Complexity**: Requires careful transaction management

**Limitations**:
- **Doesn't scale to multiple caches**: Can't maintain strong consistency across distributed caches easily
- **Write latency**: Synchronous updates slow down writes
- **Single cache limitation**: If cache crashes, all requests go to database

**When to use**:
- Financial transactions (account balances, payments)
- Inventory management (prevent overselling)
- Single-server applications
- When data correctness is absolutely critical

**Layman explanation**: Like a classroom with one whiteboard. When the teacher updates it, every student immediately sees the new information because they're all looking at the same board.

---

### Eventual Consistency (Distributed Cache)

**What it is**: Cached data will eventually match the source data, but there may be temporary windows where different caches have different values. This is the typical model for distributed caches.

**The inconsistency window**:

```
SCENARIO - Distributed cache with 3 nodes:
-----------------------------------------
T=0s:   Database updated: user:123 email = "new@email.com"
T=0s:   Invalidation sent to all cache nodes

T=0.05s: Node 1 receives invalidation, purges cache
T=0.10s: Node 2 receives invalidation, purges cache
T=0.15s: Node 3 receives invalidation, purges cache


INCONSISTENCY WINDOW:
--------------------
T=0.00s - T=0.05s:
  Node 1: old email (not yet invalidated)
  Node 2: old email
  Node 3: old email

T=0.05s - T=0.10s:
  Node 1: cache miss → fetches new email
  Node 2: old email (still has stale cache)
  Node 3: old email (still has stale cache)

T=0.10s - T=0.15s:
  Node 1: new email ✓
  Node 2: cache miss → fetches new email
  Node 3: old email (still has stale cache)

T=0.15s+:
  Node 1: new email ✓
  Node 2: new email ✓
  Node 3: cache miss → fetches new email ✓

Eventually consistent: All nodes have correct data by T=0.15s
```

**Why eventual consistency?**

```
DISTRIBUTED CACHE REALITIES:
---------------------------
Network delays: Messages take time to propagate
Async invalidation: Don't wait for all nodes before confirming write
Independent nodes: Each cache node operates independently

Trade-off:
+ Much faster writes (don't wait for cache)
+ Highly scalable (add more cache nodes)
+ More available (node failures don't block writes)
- Temporary inconsistency (stale reads possible)
```

**Inconsistency window size**:

```
FACTORS:
-------
Network latency: 1-100ms typically
Invalidation propagation: 10-500ms
Cache TTL: 1 second to hours

TYPICAL WINDOWS:
---------------
Low-latency network: 10-50ms
Cross-region CDN: 1-5 seconds
Long TTL cache: Minutes to hours
```

**Example - Real-world scenario**:

```
E-COMMERCE PRODUCT UPDATE:
-------------------------
Product price changed: $100 → $80

Distributed cache with 100 edge nodes worldwide:
- 70% of nodes update within 1 second
- 95% within 5 seconds
- 99.9% within 30 seconds
- 100% within 60 seconds

User experience:
- Most users see new price quickly
- Few users might see old price for ~30 seconds
- Eventually all users see correct price
```

**Managing eventual consistency**:

**Technique 1: Short TTL**

```
Instead of relying on invalidation:
Cache-Control: max-age=10

After 10 seconds, cache automatically expires
Max inconsistency window: 10 seconds
Simpler than distributed invalidation
```

**Technique 2: Version numbers**

```
CACHE KEY INCLUDES VERSION:
--------------------------
Write to database:
  product:123 price = $80, version = 15

Cache invalidation:
  Delete cache keys matching product:123:*

Next read:
  Cache key: product:123:v15 (miss)
  Fetch from database (version 15)
  Cache as product:123:v15

Old cache entries (v14, v13) harmlessly coexist
Naturally expire via TTL
```

**Technique 3: Read-after-write from master**

```
SCENARIO:
--------
User updates profile → Write to database
Immediate redirect → Read profile page

Problem: User might see old data from stale cache!

Solution: Read from database (not cache) for this user temporarily
---------
Set flag: user:123:recently_updated = true (TTL 5 seconds)

On profile read:
  if flag exists:
    Read from database (guaranteed fresh)
  else:
    Read from cache (eventual consistency okay)

After 5 seconds, flag expires, back to normal caching
```

**When to use**:
- Large-scale distributed systems
- Read-heavy workloads
- Content where perfect accuracy is not critical (social media, news)
- When write performance and scalability matter more than immediate consistency

**Layman explanation**: Like a company memo. When management sends an update, it takes time for everyone in all offices worldwide to receive and read it. For a while, some people have the old information, others have the new. Eventually everyone is on the same page.

---

### Cache Coherency Protocols

**Cache coherency** protocols ensure that multiple caches sharing the same data stay reasonably synchronized. These are techniques borrowed from CPU cache design, adapted for distributed systems.

**MESI Protocol** (Modified, Exclusive, Shared, Invalid):

Each cache entry is in one of four states:

```
STATES:
------
M (Modified): This cache has the only copy, and it's been modified
E (Exclusive): This cache has the only copy, unmodified
S (Shared): Multiple caches have copies, all unmodified
I (Invalid): This cache entry is stale, must fetch


STATE TRANSITIONS:
-----------------
Read miss:
  Fetch from database
  If other caches have it → S (Shared)
  If no other caches → E (Exclusive)

Write:
  If state = E or M → Update locally, state = M
  If state = S → Invalidate all other caches, state = M
  If state = I → Fetch, then update, state = M

Other cache writes:
  State = I (Invalid)
```

**Example scenario**:

```
TIMELINE:
--------
T=0: Cache A reads user:123
     State A: E (Exclusive, only cache with data)

T=1: Cache B reads user:123
     State A: S (Shared)
     State B: S (Shared)

T=2: Cache C reads user:123
     State A: S (Shared)
     State B: S (Shared)
     State C: S (Shared)

T=3: Cache A writes user:123 (email changed)
     State A: M (Modified, has exclusive write)
     State B: I (Invalid, must re-fetch)
     State C: I (Invalid, must re-fetch)

     Invalidation messages sent to B and C

T=4: Cache B reads user:123
     State B: Miss (was Invalid)
     Fetches from Cache A (has Modified version)
     State A: S (Shared)
     State B: S (Shared)
```

**Write-invalidate vs Write-update** (coherency strategies):

```
WRITE-INVALIDATE:
----------------
On write: Invalidate all other cache copies
Other caches: Must re-fetch on next read

Timeline:
  Cache A writes → Send "Invalidate" to Caches B, C
  Cache B next read → Miss → Fetch from database/Cache A

+ Lower bandwidth (send small invalidation message)
+ Fewer updates if write-heavy but read-light
- Cache miss penalty on next read


WRITE-UPDATE:
------------
On write: Send new value to all other caches
Other caches: Update their copies immediately

Timeline:
  Cache A writes → Send "Update user:123 = new_value" to Caches B, C
  Cache B immediately has new value

+ No cache miss after write
+ Reads always fast
- Higher bandwidth (send full value)
- Wasted if data written but not read
```

**Practical implementation** (simplified for distributed caches):

Most real-world distributed caches use simpler approaches:

```
COMMON PATTERN - Pub/Sub Invalidation:
-------------------------------------
Components:
- Cache nodes (subscribers)
- Message broker (Redis Pub/Sub, Kafka)

Flow:
1. Node A writes to database
2. Node A publishes: "INVALIDATE user:123"
3. All cache nodes (A, B, C) receive message
4. All nodes delete their cached user:123
5. Next read on any node: cache miss → fetch fresh


EXAMPLE WITH REDIS PUB/SUB:
--------------------------
# Node A writes
database.update(user:123, email='new@email.com')
redis.publish('cache-invalidation', 'user:123')

# All nodes (including A) subscribe
redis.subscribe('cache-invalidation', (key) => {
  cache.delete(key)
})

Result: All cache nodes invalidate within milliseconds
```

**Layman explanation**: Cache coherency protocols are like rules for maintaining multiple copies of a document across offices. When one office edits their copy, the rules determine whether to: (1) tell other offices "your copy is outdated, shred it" (write-invalidate), or (2) photocopy the changes and send them to all offices (write-update).

---

### Read-Your-Writes in Caching

**Read-your-writes consistency** guarantees that if you write data, you'll immediately see that write when you read it. This is important for user experience, even in eventually consistent systems.

**The problem**:

```
SCENARIO - User updates profile:
-------------------------------
T=0s: User submits form: "Change email to new@email.com"
      |
      v
T=1s: Server writes to database successfully
      Server sends cache invalidation (async)
      |
      v
T=1.1s: Server redirects user to profile page
        User's browser requests profile from CDN edge cache
        |
        v
T=1.1s: CDN edge cache still has old email (invalidation not arrived yet)
        User sees: old@email.com

USER: "I just changed it! Why does it show the old email?!"


TIMELINE:
--------
T=0:   Write submitted
T=1:   Database updated
T=1.1: User reads from cache → Stale data ❌
T=2:   Cache invalidation arrives
T=3:   User refreshes → New data ✓

Problem: 1-2 second window where user's own write is invisible to them
```

**Solution 1: Session-based bypass**

```
IMPLEMENTATION:
--------------
On write:
  1. Update database
  2. Set session flag: recently_updated[user_id] = true (TTL 5 seconds)
  3. Return success to user

On read:
  if session.recently_updated[user_id]:
    bypass_cache()  # Read directly from database
  else:
    read_from_cache()  # Normal caching

After 5 seconds:
  Flag expires, user back to normal caching
  By then, cache likely updated


EXAMPLE:
-------
User 123 updates email
Session: recently_updated[123] = true, expires in 5s

Next 5 seconds:
  User 123 reads → Database (fresh) ✓
  User 456 reads → Cache (might be stale, but they didn't write)

After 5 seconds:
  User 123 reads → Cache (now fresh due to eventual consistency)
```

**Solution 2: Client-side version tracking**

```
IMPLEMENTATION:
--------------
On write:
  1. Database updated, gets new version: v=42
  2. Return to client: {success: true, version: 42}
  3. Client stores: lastKnownVersion = 42

On read:
  Client sends: GET /user/123, Header: X-Min-Version: 42

  Server checks cache:
    if cache.version >= 42:
      return cache (safe)
    else:
      bypass_cache()  # Client needs at least v42


EXAMPLE:
-------
T=1: User writes, database v=42, client stores v=42
T=2: User reads, sends "I need at least v=42"
     Cache has v=41 → Server fetches from database (v=42)
T=5: User reads again, sends "I need at least v=42"
     Cache now has v=42 → Cache hit (safe to use)
```

**Solution 3: Master-slave read routing**

```
ARCHITECTURE:
------------
Database: Master (writes) + Slave replicas (reads)
Cache: Backed by slave replicas

On write:
  Write to master
  Set flag: user:123:read_from_master = true (TTL 3s)

On read:
  if flag exists:
    Read from master (guaranteed to have write)
  else:
    Read from slave/cache (eventual consistency)


TIMELINE:
--------
T=0:  User writes to master
      Set flag: read_from_master = true

T=0-3s:
  User reads → Master (sees own write immediately)
  Slave replicates in background

T=3s+:
  Flag expires
  User reads → Slave/cache (now consistent)
```

**Solution 4: Critical path cache control**

```
STRATEGY:
--------
For user-initiated writes where immediate readback is expected:
Cache-Control: no-cache (force revalidation)

For background/async writes:
Cache-Control: max-age=300 (normal caching)


EXAMPLE - Social media post:
---------------------------
User creates post:
  Write to database
  Return: Cache-Control: no-cache
  User's browser immediately sees new post

Someone else views profile:
  Cache-Control: max-age=60
  May take up to 60s to see new post
  (Acceptable, they didn't create it)
```

**Comparison**:

```
SESSION-BASED BYPASS:
+ Simple implementation
+ Works with any cache
- Requires session management
- Bypasses cache (database load)

CLIENT VERSION TRACKING:
+ Precise (only bypass when necessary)
+ Stateless server
- Client complexity
- Requires version support

MASTER-SLAVE ROUTING:
+ Clean separation
+ Database-level solution
- Requires DB replication
- More infrastructure

CACHE-CONTROL:
+ No special logic needed
+ Browser-handled
- Bypasses cache unnecessarily
- Only works for browser caching
```

**When each matters**:

```
CRITICAL:
- User updates own profile → Must see immediately
- User posts content → Must see own post
- User makes payment → Must see updated balance

ACCEPTABLE TO DELAY:
- User follows someone → Okay if follower list updates in 30s
- User likes a post → Okay if like count delayed
- Background data sync → User doesn't expect immediate visibility
```

**Layman explanation**: Read-your-writes is like saving a document on your computer and expecting to see the changes when you reopen it immediately. It would be frustrating if it showed the old version for 5 seconds while your changes "sync" to the disk. The system should ensure you see your own changes right away, even if others see them a bit later.

---

## Cache Performance

Cache performance optimization focuses on maximizing **cache hit ratio** (percentage of requests served from cache) while minimizing latency and resource consumption.

```
CACHE PERFORMANCE METRICS:
-------------------------
Hit Ratio = Cache Hits / Total Requests

Examples:
90% hit ratio: 9 out of 10 requests from cache
99% hit ratio: 99 out of 100 requests from cache
99.9% hit ratio: 999 out of 1000 requests from cache

IMPACT:
------
At 10,000 requests/second:
- 90% hit ratio → 1,000 database queries/second
- 99% hit ratio → 100 database queries/second
- 99.9% hit ratio → 10 database queries/second

10x improvement from 90% to 99%!
```

---

### Hot Key Handling

**Hot keys** (also called **hot spots**) are cache keys that receive disproportionately high traffic. This can overwhelm a single cache node or cause performance bottlenecks.

**The problem**:

```
SCENARIO - Celebrity tweets:
---------------------------
Elon Musk tweets something controversial
10,000,000 users request: /tweet/12345 within 1 minute

Distributed cache with consistent hashing:
hash(tweet:12345) → Node 2

All 10 million requests hit Node 2:
[Node 1]  ← 100 req/s
[Node 2]  ← 166,000 req/s (HOT!)  ⚠️
[Node 3]  ← 100 req/s

Node 2: CPU maxed, memory pressure, slow responses
Nodes 1 & 3: Idle, wasting resources
```

**Impact of hot keys**:
- **Node overload**: Single node can't handle traffic
- **Cascading failures**: If hot node fails, requests flood next node
- **Wasted capacity**: Other nodes underutilized
- **Poor user experience**: Timeouts, errors, slow responses

**Solution 1: Local Caching** (L1 cache):

```
ARCHITECTURE:
------------
Application Server 1:
  Local Cache (L1) → Distributed Cache (L2) → Database

Application Server 2:
  Local Cache (L1) → Distributed Cache (L2) → Database


HOT KEY FLOW:
------------
Request for tweet:12345 arrives at App Server 1:
  1. Check local cache (L1)
     - HIT: Return immediately (microseconds)
     - MISS: Continue to step 2

  2. Check distributed cache (L2)
     - HIT: Store in L1, return (milliseconds)
     - MISS: Continue to step 3

  3. Query database
     - Store in both L1 and L2
     - Return (10+ milliseconds)


BENEFIT:
-------
10M requests for hot key distributed across 100 app servers:
- Each app server: 100,000 requests
- L1 cache handles 99,900 (99.9% hit ratio)
- L2 cache sees only 100 requests per server = 10,000 total
- Database sees 1-10 requests total

Result: Hot key load distributed across all app servers
        L2 cache node not overwhelmed
```

**L1 cache characteristics**:

```
SIZE: Small (10-100 MB)
  Only most popular items fit

TTL: Very short (1-10 seconds)
  Prevents stale data in distributed environment

EVICTION: LRU or LFU
  Keep only hottest items


EXAMPLE CONFIG:
--------------
L1_CACHE_SIZE = 50 MB
L1_CACHE_TTL = 5 seconds
L1_MAX_ITEMS = 10000

Only cache in L1 if:
  - Request frequency > 100 req/s
  - Item size < 100 KB
```

**Solution 2: Key Splitting** (Replication):

```
CONCEPT:
-------
Instead of one cache entry, create multiple copies with different keys

Original:
tweet:12345 → Stored on Node 2 only

Split:
tweet:12345:replica-1 → Node 1
tweet:12345:replica-2 → Node 2
tweet:12345:replica-3 → Node 3


REQUEST DISTRIBUTION:
--------------------
Application randomly chooses replica:
Request 1 → tweet:12345:replica-2 (Node 2)
Request 2 → tweet:12345:replica-1 (Node 1)
Request 3 → tweet:12345:replica-3 (Node 3)

10M requests distributed across 3 nodes:
Node 1: ~3.3M requests
Node 2: ~3.3M requests
Node 3: ~3.3M requests

Load balanced!
```

**Automatic hot key detection**:

```
MONITORING:
----------
Track request frequency per key
If frequency > threshold (e.g., 1000 req/s):
  → Mark as hot key
  → Enable mitigation (local cache or replication)


IMPLEMENTATION:
--------------
Cache middleware:

request_counts = {}  # In-memory counter

on_request(key):
  request_counts[key] += 1

  if request_counts[key] > 1000 per second:
    enable_local_cache(key, ttl=10s)
    # or
    replicate_key(key, copies=3)


EXAMPLE OUTPUT:
--------------
Detected hot keys:
- tweet:12345: 100,000 req/s → Local cached for 5s
- user:celebrity: 50,000 req/s → Replicated 5x
- product:viral-item: 25,000 req/s → Local cached for 10s
```

**Layman explanation**: Hot key handling is like a popular ride at an amusement park. Instead of everyone queuing at one entrance (overloaded), you create multiple identical entrances across the park (replication), or give frequent riders a fast-pass they can use anywhere (local cache).

---

### Cache Warming Strategies

**Cache warming** is preloading cache with data before traffic arrives, avoiding the **cold start problem** where the first requests are slow due to empty cache.

**Cold start problem**:

```
SCENARIO - Deploy new cache:
---------------------------
T=0:  Cache deployed, completely empty

T=1:  Traffic arrives (1000 req/s)
      All requests are cache MISS
      1000 database queries/second
      Database overloaded!

T=60: Cache gradually filled (70% hit ratio)
      300 database queries/second
      Still struggling

T=300: Cache fully warmed (95% hit ratio)
       50 database queries/second
       Normal operation

Problem: 5 minutes of degraded performance and database stress
```

**Strategy 1: Scheduled Pre-warming**

```
APPROACH:
--------
Before deploying new cache, run warming script


EXAMPLE - E-commerce:
--------------------
Warm script:
  1. Fetch top 1000 products (by sales)
  2. Load into cache
  3. Fetch top 100 categories
  4. Load into cache
  5. Fetch homepage content
  6. Load into cache


TIMELINE:
--------
T=-10m: Start cache warming script
        Load 10,000 items (5 minutes)

T=-5m:  Warming complete
        Cache 80% populated with likely traffic

T=0:    Route traffic to new cache
        Immediate high hit ratio
        Database handles only 20% misses


WARMING SCRIPT EXAMPLE (pseudocode):
-----------------------------------
# Identify popular items
popular_products = database.query(
  "SELECT id FROM products ORDER BY views DESC LIMIT 1000"
)

# Load into cache
for product_id in popular_products:
  product = database.get(product_id)
  cache.set(f"product:{product_id}", product, ttl=3600)

# Verify warming
hit_ratio = cache.stats.hit_ratio
print(f"Cache warmed: {hit_ratio}% hit ratio")
```

**Strategy 2: Lazy Warming with Read-Through**

```
APPROACH:
--------
Use read-through cache pattern
Cache auto-populates on first request
Accept slower initial requests


COMBINED WITH PRIORITIZATION:
-----------------------------
1. Warm critical paths manually (homepage, top products)
2. Let other items warm lazily on demand


IMPLEMENTATION:
--------------
# Critical paths - pre-warm
warm_critical_paths([
  '/homepage',
  '/top-10-products',
  '/navigation-menu'
])

# Everything else - lazy load
cache.enable_read_through()
```

**Strategy 3: Mirror Traffic Warming**

```
APPROACH:
--------
Duplicate production traffic to new cache before cutover


ARCHITECTURE:
------------
Production Traffic
    |
    +---> Old Cache (serving users)
    |
    +---> New Cache (warming only)


After warming complete:
Production Traffic
    |
    +---> New Cache (serving users)


TIMELINE:
--------
T=-60m: Deploy new cache
        Start mirroring 100% traffic

T=-30m: New cache 50% warmed
        Continue mirroring

T=-10m: New cache 90% warmed
        Ready for cutover

T=0:    Cutover to new cache
        Immediate high hit ratio
```

**Strategy 4: Gradual Ramp-Up**

```
APPROACH:
--------
Slowly increase traffic to new cache
Allows warming without overload


PERCENTAGE ROLLOUT:
------------------
T=0:   1% traffic → New cache, 99% → Old cache
T=10m: 5% traffic → New cache, 95% → Old cache
T=20m: 25% traffic → New cache, 75% → Old cache
T=40m: 50% traffic → New cache, 50% → Old cache
T=60m: 100% traffic → New cache


MONITORING:
----------
At each step, verify:
- Hit ratio acceptable (>90%)
- Latency within bounds (<50ms p99)
- Database load stable (<500 qps)

If any metric fails: Pause rollout, investigate
```

**Strategy 5: Continuous Background Warming**

```
APPROACH:
--------
Continuously refresh popular items in background
Ensures hot data always cached


IMPLEMENTATION:
--------------
Background job (runs every 1 minute):

popular_keys = get_popular_keys_last_hour()

for key in popular_keys:
  if cache.ttl(key) < 60 seconds:  # Expiring soon
    data = database.get(key)
    cache.set(key, data, ttl=3600)  # Refresh


EXAMPLE:
-------
Product page views (last hour):
- product:123 → 10,000 views
- product:456 → 8,000 views
- product:789 → 5,000 views

Background job:
  Every minute, check if these are expiring soon
  If yes, refresh from database proactively

Result: Hot products never expire, always cache hit
```

**When to use each**:

```
SCHEDULED PRE-WARMING:
- Known popular items (homepage, top products)
- Planned deployments
- Small, predictable dataset

LAZY WARMING:
- Unpredictable access patterns
- Large dataset (can't pre-warm everything)
- Acceptable initial slowness

MIRROR TRAFFIC:
- Zero-downtime migrations
- Production-like warming needed
- Have capacity to mirror traffic

GRADUAL RAMP-UP:
- Risky deployments
- Want to validate performance
- Can afford slow rollout

CONTINUOUS BACKGROUND:
- Long-running systems
- Prevent popular item expiration
- Have background job capacity
```

---

### Eviction Policies

**Eviction policies** determine which cache entries to remove when the cache is full and new items need to be stored. Different policies optimize for different access patterns.

**The eviction problem**:

```
SCENARIO:
--------
Cache capacity: 1000 items
Currently storing: 1000 items (FULL)
New item arrives: Must evict something!

Question: Which item to evict?
Answer: Depends on eviction policy
```

**Policy 1: LRU (Least Recently Used)**

```
CONCEPT:
-------
Evict the item that hasn't been accessed for the longest time


DATA STRUCTURE:
--------------
Doubly-linked list + Hash map

[Head] <-> [Item A] <-> [Item B] <-> [Item C] <-> [Tail]
           (newest)                               (oldest)

Hash map: {A → Node A, B → Node B, C → Node C}


OPERATIONS:
----------
GET(key):
  1. Hash map lookup: O(1)
  2. Move to head (most recent): O(1)

SET(key, value):
  If cache full:
    1. Remove tail (least recent): O(1)
  2. Insert at head: O(1)


EXAMPLE TIMELINE:
----------------
Initial: [A] <-> [B] <-> [C]
Access C: [C] <-> [A] <-> [B]  (C moved to head)
Access A: [A] <-> [C] <-> [B]  (A moved to head)
Add D (full, evict B): [D] <-> [A] <-> [C]  (B evicted)
```

**LRU characteristics**:
- **Good for**: Temporal locality (recently used = likely to be used again)
- **Example**: User sessions, recent blog posts, active user profiles
- **Weakness**: One-time bulk reads can evict hot items

**LRU Cache Pollution**:

```
PROBLEM - Bulk scan:
-------------------
Cache has: [Hot items used 1000x/day]

Batch job runs: Reads 10,000 items once each
Result: Hot items evicted by one-time-use items!

Cache now has: [10,000 items never accessed again]
Next real user requests: Cache miss on hot items
Performance degraded
```

**Policy 2: LFU (Least Frequently Used)**

```
CONCEPT:
-------
Evict the item accessed least often


DATA STRUCTURE:
--------------
Hash map + Frequency counters

{
  Item A: {value: ..., frequency: 100},
  Item B: {value: ..., frequency: 50},
  Item C: {value: ..., frequency: 10}
}

On eviction: Remove item with lowest frequency


OPERATIONS:
----------
GET(key):
  frequency[key] += 1

SET(key, value):
  If cache full:
    Find min_frequency item
    Evict that item
  Insert new item with frequency = 1


EXAMPLE TIMELINE:
----------------
Cache: {A: freq=100, B: freq=50, C: freq=10}
Access A: {A: freq=101, B: freq=50, C: freq=10}
Access C: {A: freq=101, B: freq=50, C: freq=11}
Add D (full, evict C): {A: freq=101, B: freq=50, D: freq=1}
(C evicted because lowest frequency)
```

**LFU characteristics**:
- **Good for**: Frequency-based patterns (popular items stay cached)
- **Example**: Popular products, trending content, common API responses
- **Weakness**: Old popular items stay cached even if no longer accessed

**LFU Staleness Problem**:

```
PROBLEM - Historical popularity:
-------------------------------
Item A: Accessed 10,000 times last month (frequency=10,000)
Item A: Not accessed at all this month
Item B: Accessed 100 times this month (frequency=100)

LFU: Keeps item A (higher frequency)
But: Item B is currently popular, A is stale

Result: Cache filled with historically popular but now-unused items
```

**Policy 3: ARC (Adaptive Replacement Cache)**

```
CONCEPT:
-------
Combines LRU and LFU
Adapts to workload automatically


ARCHITECTURE:
------------
Two lists:
- T1: Recent items (LRU logic)
- T2: Frequent items (LFU logic)

Dynamic sizing:
If workload favors recency → Grow T1
If workload favors frequency → Grow T2


GHOST LISTS (tracking evicted items):
------------------------------------
- B1: Items evicted from T1 (recently used)
- B2: Items evicted from T2 (frequently used)


ADAPTATION:
----------
If hit on B1 (evicted recent item):
  → Workload is recency-biased
  → Increase T1 size, decrease T2

If hit on B2 (evicted frequent item):
  → Workload is frequency-biased
  → Increase T2 size, decrease T1


EXAMPLE:
-------
Sequential scan workload (recency matters):
ARC: Grows T1, shrinks T2 → Behaves like LRU

Popular item workload (frequency matters):
ARC: Grows T2, shrinks T1 → Behaves like LFU

Mixed workload:
ARC: Balances T1 and T2 → Optimal for mix
```

**ARC characteristics**:
- **Good for**: Unknown or changing workloads
- **Example**: General-purpose caching, unpredictable traffic patterns
- **Weakness**: More complex implementation, higher memory overhead

**Policy 4: FIFO (First In, First Out)**

```
CONCEPT:
-------
Evict oldest item by insertion time (not access time)


DATA STRUCTURE:
--------------
Queue

[Front] → [Item A] → [Item B] → [Item C] → [Back]
          (oldest)                          (newest)


OPERATIONS:
----------
SET(key, value):
  If cache full:
    Evict front (oldest insertion)
  Add to back


EXAMPLE:
-------
Cache: [A, B, C]
Access A (no change): [A, B, C]
Add D (full): [B, C, D]  (A evicted, even though just accessed!)
```

**FIFO characteristics**:
- **Good for**: Simple, predictable behavior
- **Example**: Log buffers, temporary data
- **Weakness**: Ignores access patterns completely, poor performance

**Policy Comparison**:

```
WORKLOAD: 80% requests to 20% of items (power law distribution)

LRU:
- Hit ratio: 85%
- Good at keeping recent hot items
- Vulnerable to scans

LFU:
- Hit ratio: 88%
- Excellent for power law
- Vulnerable to stale popular items

ARC:
- Hit ratio: 87%
- Adapts well
- More complex

FIFO:
- Hit ratio: 60%
- Simple but inefficient
- Ignores access patterns
```

**Modern approach - W-TinyLFU**:

```
CONCEPT:
-------
Combines admission policy with LFU
Protects against cache pollution


ADMISSION:
---------
New item must "prove" it's popular before entering cache

Track: Count-Min Sketch (frequency estimator)
Question: Is new item more popular than victim item?
  If yes → Admit
  If no → Reject


EXAMPLE:
-------
Cache full, new item arrives (freq=5)
Victim candidate (freq=100)

Question: Is 5 > 100? No
Action: Reject new item, keep victim
Result: One-time items don't evict popular items
```

**Choosing an eviction policy**:

```
USE LRU IF:
- Temporal locality (recent = important)
- Simple implementation needed
- Workload is recency-biased

USE LFU IF:
- Frequency matters (popular items)
- Power law distribution
- Okay with stale popular items

USE ARC IF:
- Unknown workload
- Workload changes over time
- Want best general performance

USE FIFO IF:
- Truly sequential data (logs)
- Simplicity critical
- Cache hit ratio not important
```

**Layman explanation**:
- **LRU**: Like your email inbox—recently read emails stay accessible, old unread ones get archived.
- **LFU**: Like Netflix recommendations—shows you've watched many times stay in your "Continue Watching," even if you haven't watched lately.
- **ARC**: Like a smart organizer that notices whether you're rewatching old favorites (LFU) or discovering new shows (LRU) and adjusts accordingly.

---

### Memory Fragmentation Management

**Memory fragmentation** occurs when cache memory becomes divided into small, non-contiguous free blocks, wasting space and degrading performance.

**The fragmentation problem**:

```
SCENARIO - Variable-sized cache entries:
---------------------------------------
Cache stores:
- User profiles: 1 KB each
- Images: 500 KB each
- API responses: 10 KB each


INITIAL STATE (empty cache):
----------------------------
[========================= 10 GB free =========================]


AFTER LOADING:
-------------
[User|Image|User|API|User|Image|User|API|User|Image]
 1KB  500KB 1KB  10KB 1KB  500KB 1KB  10KB 1KB  500KB


AFTER EVICTING 3 IMAGES:
-----------------------
[User|FREE|User|API|User|FREE|User|API|User|FREE]
 1KB  500KB 1KB  10KB 1KB  500KB 1KB  10KB 1KB  500KB

Total free: 1.5 GB
But: Scattered in 500 KB chunks

NEW ITEM: 1 GB image
Problem: No single contiguous 1 GB block!
Result: Allocation fails despite 1.5 GB free (FRAGMENTATION)
```

**External fragmentation** (free space scattered):

```
VISUAL:
------
[Used][Free 100KB][Used][Free 200KB][Used][Free 150KB][Used]

Total free: 450 KB
Largest contiguous: 200 KB
Can't allocate: 300 KB item (even though total > 300 KB)
```

**Internal fragmentation** (wasted space within allocations):

```
SLAB ALLOCATOR:
--------------
Slab sizes: 64B, 128B, 256B, 512B, 1KB, 2KB...

Store 100-byte item:
  Allocate from 128B slab
  Waste: 28 bytes (22% wasted)

Store 1000-byte item:
  Allocate from 1KB slab
  Waste: 24 bytes (2.4% wasted)
```

**Solution 1: Slab Allocation** (Memcached approach):

```
PRE-ALLOCATE FIXED-SIZE SLABS:
-----------------------------
Slab Class 1: 100 × 128-byte chunks
Slab Class 2: 100 × 256-byte chunks
Slab Class 3: 100 × 512-byte chunks
...


ALLOCATION:
----------
Store 200-byte item:
  Find smallest fitting slab: 256-byte
  Allocate one chunk
  Waste: 56 bytes (acceptable trade-off)


BENEFITS:
--------
+ No external fragmentation (all chunks same size within slab)
+ Fast allocation (pop from free list)
+ Predictable memory layout

DRAWBACKS:
---------
- Internal fragmentation (wasted space)
- Need to tune slab sizes for workload
```

**Solution 2: Defragmentation** (Active compaction):

```
CONCEPT:
-------
Periodically move items to consolidate free space


BEFORE DEFRAG:
-------------
[A][free][B][free][C][free][D]


AFTER DEFRAG:
------------
[A][B][C][D][========free========]


PROCESS:
-------
1. Identify fragmented regions
2. Move live items to front
3. Consolidate free space at end
4. Update pointers/indices


CHALLENGES:
----------
- Must happen while serving traffic
- Moving items is expensive (CPU, memory bandwidth)
- Risk of corruption if process interrupted
```

**Redis defragmentation**:

```
CONFIGURATION:
-------------
activedefrag yes                # Enable defragmentation
active-defrag-threshold-lower 10  # Start if >10% fragmentation
active-defrag-cycle-min 5        # Min 5% CPU for defrag
active-defrag-cycle-max 25       # Max 25% CPU for defrag


PROCESS:
-------
Background thread:
  Scan memory
  Identify fragmented objects
  Reallocate to contiguous space
  Update pointers
  Free old fragmented space


EXAMPLE:
-------
Before: 10 GB allocated, 2 GB fragmented waste, 8 GB used
After: 8 GB allocated, 0.2 GB waste, 8 GB used
Reclaimed: 2 GB
```

**Solution 3: Size Classes** (Reduce fragmentation):

```
GROUP ITEMS BY SIZE:
-------------------
Small items (< 1 KB):   Allocate from small pool
Medium items (1-100 KB): Allocate from medium pool
Large items (> 100 KB):  Allocate from large pool


BENEFITS:
--------
Each pool has similar-sized items
Less fragmentation within pools
Eviction doesn't create huge holes


EXAMPLE:
-------
BEFORE (mixed):
[1KB][500KB][1KB][500KB][1KB]
Evict 500KB → Hole too big for 1KB item

AFTER (separated):
Small pool: [1KB][1KB][1KB]
Large pool: [500KB][500KB]
Evict 500KB → Hole perfect for next 500KB item
```

**Solution 4: Monitor and Tune**:

```
METRICS TO TRACK:
----------------
fragmentation_ratio = allocated / used
  Good: < 1.2 (< 20% waste)
  Warning: 1.2-1.5 (20-50% waste)
  Critical: > 1.5 (> 50% waste)

eviction_rate
  High eviction + high fragmentation = bad sizing


ACTIONS:
-------
If fragmentation > 30%:
  - Enable defragmentation
  - Adjust slab sizes
  - Consider increasing cache size


EXAMPLE MONITORING:
------------------
redis-cli INFO memory

used_memory: 8.2 GB
used_memory_rss: 10.5 GB  (actual OS allocation)
fragmentation_ratio: 1.28  (28% fragmentation)

Action: Acceptable, but monitor. Consider defrag if increases.
```

**Preventing fragmentation**:

```
BEST PRACTICES:
--------------
1. Use slab allocation (Memcached)
2. Enable active defragmentation (Redis)
3. Size cache appropriately (don't run at 99% capacity)
4. Use TTL to naturally clear old items
5. Monitor fragmentation metrics
6. Separate storage for very large items


DON'T:
-----
- Store highly variable-sized items in same cache
- Run cache at >90% capacity constantly
- Ignore fragmentation warnings
- Mix tiny and huge objects
```

---

### Cache Hit Ratio Optimization

**Cache hit ratio** is the most important cache performance metric. Higher hit ratio = fewer database queries = better performance and lower costs.

```
FORMULA:
-------
Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)


EXAMPLE:
-------
10,000 requests:
- 9,500 cache hits
- 500 cache misses
Hit ratio = 9,500 / 10,000 = 95%


IMPACT OF IMPROVEMENT:
---------------------
Current: 90% hit ratio → 1,000 DB queries/sec
Improved: 95% hit ratio → 500 DB queries/sec (50% reduction!)
Improved: 99% hit ratio → 100 DB queries/sec (90% reduction!)
```

**Technique 1: Increase Cache Size**

```
RELATIONSHIP:
------------
Larger cache = More items fit = Higher hit ratio

DIMINISHING RETURNS:
-------------------
Cache Size    Hit Ratio    Marginal Gain
1 GB          70%          -
2 GB          85%          +15%
4 GB          92%          +7%
8 GB          95%          +3%
16 GB         96.5%        +1.5%


COST-BENEFIT:
------------
Doubling from 1GB→2GB: +15% hit ratio (great ROI)
Doubling from 8GB→16GB: +1.5% hit ratio (poor ROI)

Sweet spot: Where marginal gain drops below threshold
```

**Technique 2: Optimize TTL**

```
PROBLEM - TTL too short:
-----------------------
TTL = 10 seconds
Item accessed every 5 seconds
Every 10 seconds: Expires, must refetch
Unnecessary database load


SOLUTION:
--------
Analyze access patterns:
- Item accessed every 5 seconds
- Item changes every 1 hour

Optimal TTL: 1 hour (or less if freshness critical)


TTL TUNING:
----------
Too short:
  + Fresher data
  - Lower hit ratio (more expirations)
  - Higher database load

Too long:
  + Higher hit ratio (fewer expirations)
  - Staler data
  - More memory usage


EXAMPLE OPTIMIZATION:
--------------------
Before: TTL = 60 seconds, hit ratio = 85%
Analysis: Items change on average every 30 minutes
After: TTL = 1800 seconds (30 min), hit ratio = 94%
Result: 9% hit ratio improvement, data still fresh
```

**Technique 3: Cache More Layers**

```
MULTI-LAYER CACHING:
-------------------
Layer 1: Browser cache (individual user)
Layer 2: CDN edge cache (geographic region)
Layer 3: Application cache (shared across users)
Layer 4: Database query cache


FLOW:
----
User request
  |
  v
Browser cache hit? Yes → Return (0ms)
  | No
  v
CDN cache hit? Yes → Return (10ms)
  | No
  v
App cache hit? Yes → Return (50ms)
  | No
  v
DB query cache hit? Yes → Return (100ms)
  | No
  v
Database query → Return (200ms)


AGGREGATE HIT RATIO:
-------------------
Browser: 60% hit → 40% pass through
CDN: 80% hit → 40% × 20% = 8% pass through
App: 90% hit → 8% × 10% = 0.8% pass through
DB cache: 50% hit → 0.8% × 50% = 0.4% reach DB

Total hit ratio: 99.6%!
Total database load: 0.4% of requests
```

**Technique 4: Normalize Cache Keys**

```
PROBLEM - Key variations:
------------------------
/api/search?query=hotels&city=NYC&sort=price
/api/search?city=NYC&query=hotels&sort=price
/api/search?sort=price&query=hotels&city=NYC

Same query, different parameter order
Three separate cache entries!
Hit ratio: 33% (should be 100%)


SOLUTION - Normalize keys:
-------------------------
Canonical form: Sort parameters alphabetically

All become: /api/search?city=NYC&query=hotels&sort=price
Single cache entry
Hit ratio: 100%


EXAMPLE:
-------
Before normalization:
  1000 requests, 10 unique queries
  Each query 100 requests
  But: 5 different parameter orders each
  Cache entries: 10 × 5 = 50
  Hit ratio: 94% (some variations don't hit)

After normalization:
  Cache entries: 10 (one per unique query)
  Hit ratio: 99%
```

**Technique 5: Pre-compute and Cache Complex Results**

```
PROBLEM - Expensive computation:
-------------------------------
User requests dashboard
Server must:
  - Query 5 tables
  - Join results
  - Aggregate metrics
  - Format response
Total: 500ms

Each cache miss costs 500ms


SOLUTION - Pre-compute:
----------------------
Background job (runs every 5 minutes):
  - Compute dashboard for all active users
  - Cache results with TTL = 5 minutes

User request:
  - Cache hit: Return pre-computed dashboard (5ms)
  - Cache miss: Rare (only new users)

Result: 99% hit ratio, 100x faster responses
```

**Technique 6: Partial Caching**

```
CONCEPT:
-------
Cache parts of computation, not necessarily final result


EXAMPLE - Product page:
----------------------
Final response needs:
  - Product details (changes rarely) ✓ Cache
  - User-specific recommendations (changes often) ✗ Don't cache
  - Inventory status (changes constantly) ✗ Don't cache


IMPLEMENTATION:
--------------
Request handler:
  # Cache hit
  product = cache.get(f"product:{id}")  # 95% hit ratio

  # Always fresh
  inventory = db.query_inventory(id)
  recommendations = recommendation_service.get(user_id)

  # Combine
  return {product, inventory, recommendations}


BENEFIT:
-------
Product details: 95% cached (expensive query)
Inventory: Always fresh (cheap query)
Recommendations: User-specific (medium query)

Overall: 80% of page load time cached
Personalization + freshness maintained
```

**Technique 7: Refresh-Ahead for Popular Items**

```
CONCEPT:
-------
Proactively refresh popular items before expiration


IMPLEMENTATION:
--------------
Monitor: Track access frequency per cache key

If access_rate > 100 req/s AND ttl_remaining < 10%:
  Background refresh:
    data = fetch_from_database()
    cache.set(key, data, ttl=original_ttl)


EXAMPLE:
-------
Homepage content:
  - Accessed 10,000 req/s
  - TTL: 300 seconds
  - At T=270s (90% of TTL), background refresh starts
  - At T=300s, fresh data already in cache
  - No cache miss, no user waits


RESULT:
------
Popular items: 100% hit ratio (never expire)
Unpopular items: Normal cache misses (acceptable)
```

**Monitoring hit ratio**:

```
METRICS:
-------
Overall hit ratio: Total hits / total requests
Per-endpoint hit ratio: Identify low-performing endpoints
Per-TTL hit ratio: Validate TTL tuning
Miss reasons: Expired vs never-cached vs evicted


ACTIONABLE INSIGHTS:
-------------------
Endpoint /api/products: 60% hit ratio
  → Low! Investigate: Is TTL too short? Items not cacheable?

High eviction rate on user profiles:
  → Cache too small or bad eviction policy

Expired misses > 50%:
  → TTL too short, increase it


EXAMPLE DASHBOARD:
-----------------
Overall hit ratio: 94%  ✓ Good

By endpoint:
  /api/products: 98%  ✓
  /api/search: 75%   ⚠️ Investigate
  /api/user/feed: 45% ⚠️⚠️ Fix!

By TTL:
  300s: 90%
  600s: 95%
  1800s: 97%
  → Consider increasing TTL
```

**Layman explanation**: Optimizing cache hit ratio is like organizing a kitchen. Put frequently used items (salt, spatula) within easy reach (high hit ratio). Less-used items (specialty spices) can be in the back (occasional cache miss, fetch from storage). The better you organize based on usage patterns, the less time you waste searching.

---

## Conclusion

Caching is a fundamental technique in modern software systems, enabling applications to scale, perform better, and reduce infrastructure costs. The key principles to remember:

1. **Match patterns to use cases**: Cache-aside for flexibility, write-through for consistency, write-behind for performance, read-through for simplicity

2. **Invalidate wisely**: Use TTL for simplicity, event-driven for accuracy, or versioned keys for atomic updates

3. **Distribute strategically**: Choose between replication (read scaling) and partitioning (capacity scaling), or combine both

4. **Leverage edge caching**: Put content close to users with CDNs and tune Cache-Control headers appropriately

5. **Accept consistency trade-offs**: Strong consistency for critical data, eventual consistency for scale, read-your-writes for user experience

6. **Optimize performance**: Handle hot keys, warm caches proactively, choose appropriate eviction policies, and continuously monitor hit ratios

The "best" caching strategy depends entirely on your specific requirements—latency, consistency, scale, and complexity constraints. Start simple (cache-aside + TTL), measure performance, and add complexity only as needed.

---

**End of Document**