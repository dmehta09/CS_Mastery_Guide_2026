# Project 03: Distributed Rate Limiting Service

## Difficulty: Intermediate
**Estimated Time**: 2-3 weeks

## Problem Statement

Design a distributed rate limiting service like Cloudflare Rate Limiting, AWS WAF, or Kong that protects APIs from abuse, ensures fair usage, and prevents DDoS attacks.

**Real-World Analogy**: Think of a nightclub bouncer with a clicker counter who only allows 100 people per hour. Once the limit is reached, new customers must wait until someone leaves.

## Functional Requirements

### Core Features:
1. **Rate Limiting**: Limit requests per user/IP/API key
2. **Multiple Windows**: Per second, minute, hour, day
3. **Distributed**: Work across multiple servers
4. **Tiered Limits**: Different limits for different user tiers
5. **Burst Handling**: Allow short bursts within limits
6. **Custom Rules**: Rate limit based on path, method, headers

### Out of Scope:
- DDoS mitigation (at network layer)
- Bot detection
- Billing/payment processing

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Latency** | < 5ms | Overhead per request |
| **Accuracy** | 99%+ | Correctly enforced limits |
| **Throughput** | 100K checks/sec | Per rate limiter instance |
| **Availability** | 99.99% | Fail open if down |
| **Storage** | O(users) | Track state per user |
| **Consistency** | Eventual (1-2s) | Across distributed nodes |

## Capacity Estimation

### Assumptions:
- **10 million active users**
- **100K requests/second** (globally)
- **Average 10 rate limits per user** (e.g., per API endpoint)
- **Memory needed per rate limit entry**: ~200 bytes

### Storage Calculation:
```
Total rate limit entries:
  10M users × 10 endpoints = 100M entries

Memory per entry:
  - User ID: 8 bytes
  - Endpoint: 32 bytes
  - Counter: 8 bytes
  - Timestamp: 8 bytes
  - Metadata: 144 bytes
  Total: ~200 bytes

Total memory:
  100M entries × 200 bytes = 20 GB

With overhead and growth (3x):
  Required: ~60 GB Redis
```

### Redis Operations:
```
Requests per second: 100K

Redis operations per check:
  - GET counter (1 op)
  - INCR counter (1 op)
  - SET TTL (1 op)
  Total: ~3 ops per request

Redis load:
  100K requests/sec × 3 ops = 300K ops/sec

Redis capacity:
  Single Redis: ~100K ops/sec
  Required: 3-4 Redis instances (with replication)
```

## Algorithms Comparison

### Algorithm 1: Fixed Window Counter

**How it works**: Count requests in fixed time windows (e.g., 00:00-00:59, 01:00-01:59)

```
Timeline: ────|────────────|────────────|────
Windows:      0:00        1:00        2:00
Limit: 100 requests per hour

Example:
- 0:00-0:59: 90 requests ✅
- 1:00-1:59: 100 requests ✅
- 1:58: 50 requests
- 2:01: 50 requests
→ Total in 3 minutes: 100 requests (allowed), but burst of 100 in 3 min!
```

**Implementation**:
```typescript
class FixedWindowRateLimiter {
  async checkRateLimit(
    userId: string,
    limit: number,
    windowSeconds: number
  ): Promise<{ allowed: boolean; remaining: number }> {
    const now = Date.now();
    const windowKey = Math.floor(now / (windowSeconds * 1000));
    const key = `rate_limit:${userId}:${windowKey}`;

    const count = await redis.incr(key);

    if (count === 1) {
      // First request in this window, set expiry
      await redis.expire(key, windowSeconds);
    }

    const allowed = count <= limit;
    const remaining = Math.max(0, limit - count);

    return { allowed, remaining };
  }
}
```

**Pros**:
- Simple to implement
- Memory efficient
- Fast (1-2 Redis ops)

**Cons**:
- Edge case: Burst at window boundaries
- Example: 100 requests at 0:59, 100 at 1:00 = 200 in 1 minute

**Use case**: When bursts are acceptable, simplicity preferred

### Algorithm 2: Sliding Window Log

**How it works**: Store timestamp of each request, count requests in sliding window

```
Window: Last 60 seconds from now

Current time: 1:30:00
Check requests from 1:29:00 to 1:30:00

Requests: [1:29:05, 1:29:10, 1:29:45, 1:30:00]
Count: 4 requests in last 60 seconds
```

**Implementation**:
```typescript
class SlidingWindowLogRateLimiter {
  async checkRateLimit(
    userId: string,
    limit: number,
    windowSeconds: number
  ): Promise<{ allowed: boolean; remaining: number }> {
    const now = Date.now();
    const windowStart = now - windowSeconds * 1000;
    const key = `rate_limit:${userId}`;

    // Redis sorted set: score = timestamp, member = unique request ID
    const multi = redis.multi();

    // Remove old entries outside window
    multi.zremrangebyscore(key, 0, windowStart);

    // Count requests in window
    multi.zcard(key);

    // Add current request
    multi.zadd(key, now, `${now}-${Math.random()}`);

    // Set expiry
    multi.expire(key, windowSeconds);

    const [, , count] = await multi.exec();

    const allowed = count <= limit;
    const remaining = Math.max(0, limit - count);

    return { allowed, remaining };
  }
}
```

**Pros**:
- Accurate: No bursts at boundaries
- Precise sliding window

**Cons**:
- Memory intensive: Store timestamp for each request
- Slower: Multiple Redis ops
- Memory: 100 req/min × 10M users = 1B entries

**Use case**: Need high accuracy, memory not a concern

### Algorithm 3: Sliding Window Counter (Hybrid)

**How it works**: Combine fixed window with interpolation

```
Current window: 1:00-2:00, count = 80
Previous window: 0:00-1:00, count = 100

Current time: 1:30 (50% into window)

Estimated count = (Previous × 50%) + (Current × 100%)
                = (100 × 0.5) + (80 × 1.0)
                = 50 + 80
                = 130 requests in last 60 min
```

**Implementation**:
```typescript
class SlidingWindowCounterRateLimiter {
  async checkRateLimit(
    userId: string,
    limit: number,
    windowSeconds: number
  ): Promise<{ allowed: boolean }> {
    const now = Date.now();
    const currentWindow = Math.floor(now / (windowSeconds * 1000));
    const previousWindow = currentWindow - 1;

    const currentKey = `rate_limit:${userId}:${currentWindow}`;
    const previousKey = `rate_limit:${userId}:${previousWindow}`;

    // Get counts for both windows
    const [currentCount, previousCount] = await Promise.all([
      redis.get(currentKey).then(v => parseInt(v || '0')),
      redis.get(previousKey).then(v => parseInt(v || '0'))
    ]);

    // Calculate position in current window (0.0 to 1.0)
    const currentWindowTime = currentWindow * windowSeconds * 1000;
    const elapsed = now - currentWindowTime;
    const progress = elapsed / (windowSeconds * 1000);

    // Interpolate: weight previous window by remaining portion
    const estimatedCount = previousCount * (1 - progress) + currentCount;

    const allowed = estimatedCount < limit;

    if (allowed) {
      await redis.incr(currentKey);
      await redis.expire(currentKey, windowSeconds * 2); // Keep for 2 windows
    }

    return { allowed };
  }
}
```

**Pros**:
- Good balance: Memory efficient + accurate
- Smooth rate limiting

**Cons**:
- Approximate (not 100% accurate)
- Slightly complex

**Use case**: Production systems (best trade-off)

### Algorithm 4: Token Bucket

**How it works**: Bucket has tokens, each request consumes 1 token, bucket refills at constant rate

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

t=0s:  100 tokens → Request (99 tokens)
t=1s:  99 + 10 = 109 → capped at 100
t=10s: 100 tokens (fully refilled)

Allows bursts up to bucket capacity
```

**Implementation**:
```typescript
class TokenBucketRateLimiter {
  async checkRateLimit(
    userId: string,
    capacity: number,
    refillRate: number // tokens per second
  ): Promise<{ allowed: boolean; tokens: number }> {
    const key = `token_bucket:${userId}`;
    const now = Date.now();

    // Get current state
    const state = await redis.get(key);
    let tokens: number;
    let lastRefill: number;

    if (!state) {
      // Initialize
      tokens = capacity;
      lastRefill = now;
    } else {
      const parsed = JSON.parse(state);
      tokens = parsed.tokens;
      lastRefill = parsed.lastRefill;

      // Calculate tokens to add
      const elapsed = (now - lastRefill) / 1000; // seconds
      const tokensToAdd = elapsed * refillRate;
      tokens = Math.min(capacity, tokens + tokensToAdd);
    }

    // Try to consume token
    const allowed = tokens >= 1;
    if (allowed) {
      tokens -= 1;
    }

    // Save state
    await redis.set(
      key,
      JSON.stringify({ tokens, lastRefill: now }),
      'EX',
      3600 // 1 hour TTL
    );

    return { allowed, tokens: Math.floor(tokens) };
  }
}
```

**Pros**:
- Allows controlled bursts
- Smooth rate limiting

**Cons**:
- State management (tokens + lastRefill)
- Not intuitive for users

**Use case**: APIs that need burst tolerance (e.g., traffic spikes)

### Algorithm 5: Leaky Bucket

**How it works**: Requests enter bucket, leak out at constant rate

```
Bucket capacity: 100 requests
Leak rate: 10 requests/second

Incoming: Burst of 50 requests
Bucket: [50 requests queued]
Outgoing: 10 requests/sec constant rate

If bucket full (>100), reject new requests
```

**Implementation**:
```typescript
class LeakyBucketRateLimiter {
  async checkRateLimit(
    userId: string,
    capacity: number,
    leakRate: number // requests per second
  ): Promise<{ allowed: boolean; queueSize: number }> {
    const key = `leaky_bucket:${userId}`;
    const now = Date.now();

    const state = await redis.get(key);
    let queueSize: number;
    let lastLeak: number;

    if (!state) {
      queueSize = 0;
      lastLeak = now;
    } else {
      const parsed = JSON.parse(state);
      queueSize = parsed.queueSize;
      lastLeak = parsed.lastLeak;

      // Calculate leaked requests
      const elapsed = (now - lastLeak) / 1000;
      const leaked = Math.floor(elapsed * leakRate);
      queueSize = Math.max(0, queueSize - leaked);
    }

    // Try to add request to queue
    const allowed = queueSize < capacity;
    if (allowed) {
      queueSize += 1;
    }

    // Save state
    await redis.set(
      key,
      JSON.stringify({ queueSize, lastLeak: now }),
      'EX',
      3600
    );

    return { allowed, queueSize };
  }
}
```

**Pros**:
- Smooth, constant output rate
- Good for protecting downstream services

**Cons**:
- Added latency (queuing)
- Complex to implement

**Use case**: Message queue rate limiting, API gateway

## Distributed Rate Limiting

### Challenge: Synchronization Across Servers

```
Problem:
User sends 100 requests/sec
2 servers handle 50 requests each
Each server thinks user is within limit (50 < 100)
→ User actually sends 100 requests (over limit!)

Solution: Centralized Redis
```

### Centralized Redis Architecture

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  API     │    │  API     │    │  API     │
│  Server 1│    │  Server 2│    │  Server 3│
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
            ┌────────▼─────────┐
            │                  │
            │   Redis Cluster  │
            │  (Rate Limit     │
            │   Counters)      │
            │                  │
            └──────────────────┘
```

### Redis Lua Script (Atomic Operations)

```lua
-- Sliding window counter with Lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local current_window = math.floor(now / window)
local previous_window = current_window - 1

local current_key = key .. ":" .. current_window
local previous_key = key .. ":" .. previous_window

-- Get counts
local current_count = tonumber(redis.call('GET', current_key) or '0')
local previous_count = tonumber(redis.call('GET', previous_key) or '0')

-- Calculate sliding window count
local current_window_time = current_window * window
local elapsed = now - current_window_time
local progress = elapsed / window

local estimated_count = previous_count * (1 - progress) + current_count

if estimated_count < limit then
  -- Increment current window
  redis.call('INCR', current_key)
  redis.call('EXPIRE', current_key, window * 2)
  return {1, limit - estimated_count - 1}  -- allowed, remaining
else
  return {0, 0}  -- denied, 0 remaining
end
```

**Benefits of Lua Script**:
- Atomic: No race conditions
- Fast: Single round trip to Redis
- Accurate: All logic on server side

## Tiered Rate Limiting

```typescript
enum UserTier {
  FREE = 'free',
  BASIC = 'basic',
  PREMIUM = 'premium',
  ENTERPRISE = 'enterprise'
}

const rateLimits = {
  [UserTier.FREE]: {
    requestsPerHour: 100,
    requestsPerDay: 1000,
    burst: 10
  },
  [UserTier.BASIC]: {
    requestsPerHour: 1000,
    requestsPerDay: 10000,
    burst: 50
  },
  [UserTier.PREMIUM]: {
    requestsPerHour: 10000,
    requestsPerDay: 100000,
    burst: 100
  },
  [UserTier.ENTERPRISE]: {
    requestsPerHour: 100000,
    requestsPerDay: 1000000,
    burst: 1000
  }
};

async function checkRateLimit(user: User, endpoint: string) {
  const limits = rateLimits[user.tier];

  // Check multiple windows
  const [hourlyCheck, dailyCheck] = await Promise.all([
    limiter.check(user.id, limits.requestsPerHour, 3600),
    limiter.check(user.id, limits.requestsPerDay, 86400)
  ]);

  if (!hourlyCheck.allowed) {
    throw new RateLimitError('Hourly limit exceeded', {
      limit: limits.requestsPerHour,
      resetAt: hourlyCheck.resetAt
    });
  }

  if (!dailyCheck.allowed) {
    throw new RateLimitError('Daily limit exceeded', {
      limit: limits.requestsPerDay,
      resetAt: dailyCheck.resetAt
    });
  }

  return {
    allowed: true,
    limits: {
      hourly: hourlyCheck,
      daily: dailyCheck
    }
  };
}
```

## Rate Limit Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1708084800
Retry-After: 3600

If rate limited:
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1708084800
Retry-After: 3600

{
  "error": "Rate limit exceeded",
  "message": "You have exceeded your hourly limit of 100 requests",
  "retry_after_seconds": 3600,
  "upgrade_url": "/upgrade-plan"
}
```

## API Design

```http
# Check rate limit (without consuming)
GET /api/v1/rate-limit/status
Authorization: Bearer <token>
Response:
{
  "limits": {
    "hourly": {
      "limit": 1000,
      "remaining": 842,
      "reset_at": "2026-02-16T11:00:00Z"
    },
    "daily": {
      "limit": 10000,
      "remaining": 8234,
      "reset_at": "2026-02-17T00:00:00Z"
    }
  },
  "tier": "basic"
}

# Admin: Set custom limit for user
POST /api/v1/admin/rate-limits
Body:
{
  "user_id": 123,
  "limits": {
    "requests_per_hour": 5000,
    "requests_per_day": 50000
  }
}

# Admin: Reset rate limit
DELETE /api/v1/admin/rate-limits/{user_id}
```

## Trade-off Analysis

| Algorithm | Accuracy | Memory | Performance | Bursts | Complexity |
|-----------|----------|--------|-------------|--------|-----------|
| Fixed Window | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ❌ Boundary issue | Low |
| Sliding Log | ⭐⭐⭐ | ⭐ | ⭐⭐ | ✅ No issues | Medium |
| Sliding Counter | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ✅ Smooth | Medium |
| Token Bucket | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ✅ Controlled | High |
| Leaky Bucket | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ✅ Queued | High |

**Recommendation**: **Sliding Window Counter** for most production systems (best trade-off)

## Failure Handling

### Redis Failure

```typescript
class RateLimiterWithFailover {
  async checkRateLimit(userId: string, limit: number, window: number) {
    try {
      return await this.redis.checkRateLimit(userId, limit, window);
    } catch (error) {
      logger.error('Redis rate limiter failed', error);

      // Fail open: Allow request
      // (Better to allow some abuse than block legitimate users)
      return { allowed: true, remaining: limit };

      // Alternative: Fail closed (more secure)
      // return { allowed: false, remaining: 0 };

      // Alternative: Use local cache (approximate)
      // return await this.localCache.checkRateLimit(userId, limit, window);
    }
  }
}
```

**Options**:
1. **Fail Open**: Allow request (better UX, less secure)
2. **Fail Closed**: Deny request (secure, worse UX)
3. **Local Cache**: Use in-memory cache (approximate, eventual consistency)

## Monitoring

```yaml
Metrics:
  - rate_limit_checks_per_second: 100K
  - rate_limit_blocks_per_second: 1K (1% block rate)
  - rate_limit_latency_p95: 3ms
  - redis_connection_errors: 0
  - users_hitting_limits: 500 (by tier)

Alerts:
  - name: HighRateLimitLatency
    condition: p95_latency > 10ms
    severity: warning

  - name: RedisDown
    condition: redis_connection_errors > 10
    severity: critical

  - name: HighBlockRate
    condition: block_rate > 10%
    severity: warning
    action: Possible attack or limits too strict
```

## Summary

### What We Built:
- Distributed rate limiter with < 5ms latency
- Multiple algorithms (Fixed, Sliding, Token Bucket)
- Tiered limits for different user types
- Redis-based for consistency
- Fail-open for reliability

### Key Concepts:
- ✅ Rate limiting algorithms comparison
- ✅ Distributed coordination with Redis
- ✅ Lua scripts for atomicity
- ✅ Tiered limits
- ✅ Failure handling strategies

### Real-World Examples:
- Cloudflare: 100M+ RPS, sub-millisecond latency
- AWS WAF: Integrated rate limiting
- Kong: API gateway with rate limiting plugin

---

**Next Project**: [04. Polyglot E-Commerce Platform](../04-polyglot-ecommerce/README.md)
