# Project 15: API Rate Limiter Service

## Difficulty Level: Intermediate
**Estimated Time**: 2-3 weeks per language

## What You're Building

A distributed rate limiting service that protects APIs from abuse and ensures fair usage (like Cloudflare Rate Limiting, AWS WAF, or Kong rate limiting plugin).

## Key Concepts

- Token bucket algorithm
- Sliding window algorithm
- Fixed window algorithm
- Distributed rate limiting (Redis)
- Rate limit headers
- Tiered limits
- Burst handling

## Algorithms

### 1. Token Bucket

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Request comes:
- If tokens available: Allow & consume token
- If no tokens: Deny (429)

Allows bursts up to bucket capacity
```

```typescript
class TokenBucket {
  private tokens: number;
  private capacity: number;
  private refillRate: number; // per second
  private lastRefill: number;

  constructor(capacity: number, refillRate: number) {
    this.capacity = capacity;
    this.refillRate = refillRate;
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }

  tryConsume(count = 1): boolean {
    this.refill();

    if (this.tokens >= count) {
      this.tokens -= count;
      return true; // Allow
    }

    return false; // Deny
  }

  private refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRate;

    this.tokens = Math.min(
      this.capacity,
      this.tokens + tokensToAdd
    );
    this.lastRefill = now;
  }

  getRemainingTokens(): number {
    this.refill();
    return Math.floor(this.tokens);
  }
}
```

### 2. Sliding Window (Redis)

```typescript
async function checkRateLimit(
  userId: string,
  limit: number,
  windowSeconds: number
): Promise<{
  allowed: boolean;
  remaining: number;
  resetAt: number;
}> {
  const now = Date.now();
  const windowStart = now - windowSeconds * 1000;
  const key = `rate_limit:${userId}`;

  // Redis transaction
  const multi = redis.multi();

  // Remove old entries
  multi.zremrangebyscore(key, 0, windowStart);

  // Count requests in window
  multi.zcard(key);

  // Add current request
  multi.zadd(key, now, `${now}-${Math.random()}`);

  // Set expiry
  multi.expire(key, windowSeconds);

  const [, , count] = await multi.exec();

  const allowed = count < limit;
  const remaining = Math.max(0, limit - count - 1);
  const resetAt = now + windowSeconds * 1000;

  return { allowed, remaining, resetAt };
}
```

### 3. Fixed Window

```typescript
async function checkFixedWindow(
  userId: string,
  limit: number,
  windowSeconds: number
): Promise<boolean> {
  const now = Date.now();
  const windowKey = Math.floor(now / (windowSeconds * 1000));
  const key = `rate_limit:${userId}:${windowKey}`;

  const count = await redis.incr(key);

  if (count === 1) {
    await redis.expire(key, windowSeconds);
  }

  return count <= limit;
}
```

### 4. Leaky Bucket

```typescript
class LeakyBucket {
  private queue: number[] = [];
  private capacity: number;
  private leakRate: number; // requests per second

  constructor(capacity: number, leakRate: number) {
    this.capacity = capacity;
    this.leakRate = leakRate;
    this.startLeaking();
  }

  tryAdd(): boolean {
    if (this.queue.length < this.capacity) {
      this.queue.push(Date.now());
      return true; // Allow
    }
    return false; // Deny
  }

  private startLeaking() {
    setInterval(() => {
      if (this.queue.length > 0) {
        // Leak tokens
        const leakCount = Math.min(
          this.queue.length,
          this.leakRate
        );
        this.queue.splice(0, leakCount);
      }
    }, 1000);
  }
}
```

## Middleware Implementation

### Express/NestJS

```typescript
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  standardHeaders: true, // Return rate limit info in headers
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({
      error: "Too many requests",
      retryAfter: req.rateLimit.resetTime
    });
  }
});

app.use("/api/", limiter);
```

### Custom Middleware with Redis

```typescript
function rateLimitMiddleware(options: {
  limit: number;
  window: number; // seconds
}) {
  return async (req, res, next) => {
    const userId = req.user?.id || req.ip;

    const result = await checkRateLimit(
      userId,
      options.limit,
      options.window
    );

    // Add headers
    res.set({
      "X-RateLimit-Limit": options.limit,
      "X-RateLimit-Remaining": result.remaining,
      "X-RateLimit-Reset": result.resetAt
    });

    if (!result.allowed) {
      res.status(429).json({
        error: "Rate limit exceeded",
        retryAfter: Math.ceil(
          (result.resetAt - Date.now()) / 1000
        )
      });
      return;
    }

    next();
  };
}

// Usage
app.use(
  "/api/",
  rateLimitMiddleware({ limit: 100, window: 60 })
);
```

## Tiered Rate Limits

```typescript
enum UserTier {
  FREE = "free",
  BASIC = "basic",
  PREMIUM = "premium",
  ENTERPRISE = "enterprise"
}

const rateLimits = {
  [UserTier.FREE]: { limit: 100, window: 3600 },
  [UserTier.BASIC]: { limit: 1000, window: 3600 },
  [UserTier.PREMIUM]: { limit: 10000, window: 3600 },
  [UserTier.ENTERPRISE]: { limit: 100000, window: 3600 }
};

async function getRateLimit(user: User) {
  const tier = user.subscription_tier || UserTier.FREE;
  return rateLimits[tier];
}

app.use(async (req, res, next) => {
  if (!req.user) {
    return next();
  }

  const limits = await getRateLimit(req.user);
  const result = await checkRateLimit(
    req.user.id,
    limits.limit,
    limits.window
  );

  if (!result.allowed) {
    return res.status(429).json({
      error: "Rate limit exceeded",
      message: `Upgrade to ${nextTier(req.user.tier)} for higher limits`
    });
  }

  next();
});
```

## Distributed Rate Limiting

### Using Redis Lua Script

```lua
-- Atomic rate limit check
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Remove old entries
redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

-- Get count
local count = redis.call('ZCARD', key)

if count < limit then
  -- Add request
  redis.call('ZADD', key, now, now .. '-' .. math.random())
  redis.call('EXPIRE', key, window)
  return {1, limit - count - 1} -- allowed, remaining
else
  return {0, 0} -- denied, 0 remaining
end
```

```typescript
const script = `...`; // Lua script above

async function checkRateLimit(userId: string, limit: number, window: number) {
  const result = await redis.eval(
    script,
    1,
    `rate_limit:${userId}`,
    limit,
    window,
    Date.now()
  );

  return {
    allowed: result[0] === 1,
    remaining: result[1]
  };
}
```

## API Endpoints

```http
# Check rate limit status
GET /api/rate-limit/status
Authorization: Bearer <token>

Response:
{
  "limit": 1000,
  "remaining": 842,
  "reset_at": "2026-02-16T11:00:00Z",
  "window_seconds": 3600
}

# Admin: Set custom limit
POST /api/admin/rate-limits
{
  "user_id": 123,
  "limit": 5000,
  "window_seconds": 3600
}
```

## Response Headers

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1708088400
Retry-After: 3600
```

## Success Criteria

✅ Accurate rate limiting
✅ Distributed (Redis-based)
✅ Sub-5ms overhead
✅ Tiered limits working
✅ Burst handling
✅ Proper headers
✅ Admin controls

---

## Congratulations!

You've reached the end of all 15 backend mastery projects! 🎉

By implementing these projects in NestJS, Golang, and Python, you now have:
- Deep understanding of distributed systems
- Hands-on experience with production patterns
- Portfolio of senior-level projects
- Mastery of backend engineering concepts

**Next Steps**:
1. Pick your first project and start building
2. Compare implementations across languages
3. Write blog posts about your learnings
4. Open source your implementations
5. Prepare for senior engineer interviews

Good luck on your backend mastery journey! 🚀
