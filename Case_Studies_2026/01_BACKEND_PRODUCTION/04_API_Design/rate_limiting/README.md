# Rate Limiting

How to protect your APIs from abuse and ensure fair usage across clients.

---

## Case Study 1: API Getting Hammered by Runaway Client

### Problem

Your API serves 1000 clients. One client has a bug that sends 10,000 requests per second instead of 10. Your entire API becomes slow for everyone.

**Context**: Common production issue - happened at Stripe, GitHub, Cloudflare

**In simple terms**: Imagine a water pipe serving 1000 houses. One house leaves all faucets running full blast. Everyone else gets a trickle. Rate limiting is like putting a flow limiter on each house's connection.

### Quick Answer

Use token bucket rate limiting: each client gets a "bucket" of tokens that refills over time. Each request costs one token. No tokens = request rejected.

### Detailed Explanation

#### Why This Happens (Root Cause)

Without rate limiting, resources are "first come, first served":

```
Normal client: 10 req/sec × 1000 clients = 10,000 req/sec ✓
Bug client: 10,000 req/sec × 1 client = 10,000 req/sec 😱

Total: 20,000 req/sec → Your server can only handle 15,000 → Everyone suffers
```

The bug client consumes 50% of your capacity, affecting 999 other clients.

**Root cause**: No per-client quota enforcement means one misbehaving client can consume all resources.

#### The Solution Approach

**High-level strategy:**
- Give each client a quota (e.g., 100 requests/minute)
- Track usage per client in shared storage (Redis)
- Reject requests that exceed quota
- Return clear error (429) with retry information

**Architecture Decision: Token Bucket Algorithm**

**Why token bucket?**
- Allows burst traffic (bucket can hold multiple tokens)
- Smooths out traffic over time (tokens refill gradually)
- Better than fixed window (handles bursts gracefully)

**How it works conceptually:**
1. Each client has a "bucket" that holds tokens (e.g., 100 tokens max)
2. Tokens refill at steady rate (e.g., 10 tokens/second)
3. Each request consumes 1 token
4. If bucket is empty → Request rejected
5. Bucket allows bursts (can handle 100 requests instantly) but limits sustained rate

**Storage Decision: Redis**

**Why Redis?**
- Sub-millisecond lookups (critical for every request)
- Shared across all servers (consistent limits)
- Built-in expiration (auto-cleanup of old counters)
- Atomic operations (INCR + EXPIRE in single operation)

**Architecture Flow:**
```
Request → Check Redis Counter → Within Limit? → Process Request
                              → Over Limit? → Return 429 with Retry-After
```

**Key Design Decisions:**
- **Per-client tracking**: Use API keys, not just IPs (many clients share IPs)
- **Distributed storage**: Redis ensures consistent limits across multiple servers
- **Token bucket over fixed window**: Handles traffic bursts better
- **429 status code**: Tells client to back off, not retry immediately

#### Production Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| P99 latency (normal clients) | 850ms | 45ms | 19x faster |
| Failed requests (good clients) | 12% | 0.1% | 120x better |
| Bad client impact radius | All users | Just that client | Isolated |
| API availability | 88% | 99.9% | 13.5% improvement |

**Key insight**: Rate limiting isn't just about blocking bad actors - it's about protecting good clients from each other's bugs.

**Lessons Learned**:
- Always return rate limit headers (X-RateLimit-*) so clients can self-adjust
- Use 429 status code, not 500 (tells client to back off, not retry)
- Consider API keys over IP addresses (many clients share IPs)
- Monitor rate limit hit rates to identify problematic clients early

---

## Case Study 2: Distributed Rate Limiting Across Multiple Servers

### Problem

You have 10 API servers behind a load balancer. Each server tracks rate limits independently. A client can send 100 requests/second by hitting different servers, bypassing your 50 req/sec limit.

**Context**: Common issue in distributed systems - affects any multi-server API

**In simple terms**: Imagine a bank with 10 tellers, each with their own customer limit. A customer can visit all 10 tellers to bypass the limit. You need a shared ledger that all tellers check.

### Quick Answer

Use a shared rate limit store (Redis) that all servers check. Each server atomically increments the counter and checks if it exceeds the limit.

### Detailed Explanation

#### Why This Happens (Root Cause)

With independent rate limiters per server:

```
Client sends 50 requests → Load balancer distributes:
  - Server 1: 5 requests (under 50 limit) ✓
  - Server 2: 5 requests (under 50 limit) ✓
  - Server 3: 5 requests (under 50 limit) ✓
  ... (across all 10 servers)

Total: 50 requests across 10 servers = 500 requests processed
Limit bypassed!
```

**Root cause**: Each server only sees its own slice of traffic, not the global picture.

#### The Solution Approach

**High-level strategy:**
- Store rate limit counters in shared storage (Redis)
- All servers check the same counter
- Use atomic operations to prevent race conditions
- Consider Redis Cluster for high availability

**Architecture Decision: Centralized Redis Store**

**Why centralized?**
- Single source of truth for all servers
- Consistent limits regardless of which server handles the request
- Atomic operations prevent race conditions

**How it works conceptually:**
1. Client sends request to any server
2. Server extracts client identifier (API key, IP, etc.)
3. Server checks Redis: `INCR client:api_key:123`
4. Redis returns new count atomically
5. If count > limit → Return 429
6. If count <= limit → Process request

**Race Condition Prevention:**

Without atomic operations, two servers might both check and increment simultaneously:
```
Server 1: Check count (50) → OK → Increment → 51
Server 2: Check count (50) → OK → Increment → 51
Result: Both requests pass, but count should be 52
```

With Redis INCR (atomic):
```
Server 1: INCR → Returns 51 (atomic)
Server 2: INCR → Returns 52 (atomic)
Result: Correct count, second request correctly rejected
```

**Performance Considerations:**
- Redis latency adds ~1-2ms per request
- Mitigation: Use Redis pipelining for batch operations
- Consider local cache with TTL for frequently accessed limits
- Use Redis Cluster for horizontal scaling

#### Production Results

| Metric | Before (Per-Server) | After (Centralized) | Improvement |
|--------|---------------------|---------------------|-------------|
| Limit bypass incidents | 15/day | 0/day | 100% reduction |
| False positives (legitimate traffic blocked) | 2% | 0.1% | 20x better |
| Redis latency overhead | N/A | +1.5ms | Acceptable |
| Rate limit accuracy | 60% | 99.9% | 66% improvement |

**Key insight**: Distributed rate limiting requires shared state. The trade-off is slightly higher latency for accurate limits.

**Lessons Learned**:
- Always use atomic operations (Redis INCR) to prevent race conditions
- Monitor Redis latency - it's on the critical path
- Consider hybrid approach: local cache + Redis for very high-traffic clients
- Use Redis Cluster for high availability (single Redis is a SPOF)

---

## Case Study 3: Sliding Window vs Token Bucket Trade-offs

### Problem

You need to choose between sliding window and token bucket rate limiting. Each has different characteristics for handling bursts and smooth traffic.

**Context**: Architectural decision - affects how clients experience rate limits

**In simple terms**:
- **Fixed window**: Like a restaurant with 100 seats per hour. If you arrive at 2:59 PM, you might get in, but at 3:00 PM, 100 new people can also enter.
- **Sliding window**: Like a restaurant that tracks the last hour continuously. If 100 people came in the last hour, you wait.
- **Token bucket**: Like a water tank that fills gradually. You can use all stored water at once (burst), but it refills slowly.

### Quick Answer

Token bucket is better for bursty traffic (allows bursts). Sliding window is better for smooth, predictable limits (no bursts). Choose based on your traffic pattern.

### Detailed Explanation

#### Why This Matters

Different algorithms handle traffic patterns differently:

**Fixed Window Problem:**
```
Time: 2:59 PM - Client uses 99/100 requests
Time: 3:00 PM - Window resets, client can use 100 more requests
Result: 199 requests in 2 minutes (bypasses 100/min limit)
```

**Sliding Window Solution:**
```
Time: 2:59 PM - Client uses 99/100 requests (last hour)
Time: 3:00 PM - Old requests expire, but new ones count
Result: True 100 requests per hour limit
```

**Token Bucket Characteristics:**
```
Bucket size: 100 tokens
Refill rate: 10 tokens/second

Client can:
- Use all 100 tokens instantly (burst)
- Then wait 10 seconds for refill
- Sustained rate: 10 req/sec max
```

#### The Solution Approach

**Algorithm Comparison:**

| Algorithm | Burst Handling | Smooth Traffic | Implementation Complexity | Memory Usage |
|-----------|----------------|----------------|--------------------------|--------------|
| Fixed Window | Poor (resets allow bursts) | Good | Simple | Low |
| Sliding Window | Good (no bursts) | Excellent | Complex | High |
| Token Bucket | Excellent (allows controlled bursts) | Good | Medium | Medium |

**When to use Token Bucket:**
- Clients need to send bursts (e.g., batch uploads)
- You want to allow short spikes but limit sustained rate
- Example: Image upload API - allow 10 images at once, but max 100/hour

**When to use Sliding Window:**
- You need strict "never exceed X per hour" limits
- Bursts are not acceptable
- Example: Payment API - strict 1000 requests/hour, no exceptions

**Architecture Decision: Hybrid Approach**

Many production systems use token bucket with small bucket size:
- Allows small bursts (better UX)
- Prevents large bursts (better protection)
- Simpler than sliding window
- Good balance of flexibility and control

**How it works conceptually:**
1. Token bucket with bucket size = 2x sustained rate
   - Example: 200 tokens, refill 100 tokens/minute
   - Allows 2x burst, but sustained rate = 100/min
2. Client can burst up to 200 requests
3. After burst, must wait for refill
4. Sustained rate naturally limited by refill rate

#### Production Results

| Metric | Fixed Window | Sliding Window | Token Bucket |
|--------|--------------|----------------|--------------|
| Burst bypass incidents | 12/day | 0/day | 0/day (controlled) |
| False positives (legitimate bursts blocked) | 0% | 8% | 2% |
| Implementation complexity | Low | High | Medium |
| Client satisfaction | Medium | Low | High |

**Key insight**: Token bucket provides the best balance - allows legitimate bursts while preventing abuse.

**Lessons Learned**:
- Fixed window is simple but has reset loopholes
- Sliding window is accurate but complex and can block legitimate bursts
- Token bucket is the sweet spot for most APIs
- Consider your traffic pattern: bursty = token bucket, smooth = sliding window

---

## Common Mistakes

1. **Mistake**: Rate limit by IP only
   **Why it's wrong**: Many clients share IPs (corporate NAT, mobile carriers, VPNs)
   **Instead**: Use API keys + IP as fallback for unauthenticated endpoints

2. **Mistake**: No rate limit headers in response
   **Why it's wrong**: Good clients can't adjust their behavior or implement backoff
   **Instead**: Always return X-RateLimit-* headers (Remaining, Limit, Reset)

3. **Mistake**: Returning 500 instead of 429
   **Why it's wrong**: Client thinks your server is broken, keeps retrying aggressively
   **Instead**: Return 429 Too Many Requests with Retry-After header

4. **Mistake**: Per-server rate limiting in distributed systems
   **Why it's wrong**: Clients can bypass limits by hitting different servers
   **Instead**: Use shared storage (Redis) for centralized rate limiting

5. **Mistake**: No differentiation between endpoint types
   **Why it's wrong**: Expensive operations (search) and cheap operations (health check) treated the same
   **Instead**: Different limits per endpoint based on cost

6. **Mistake**: Too aggressive limits for legitimate users
   **Why it's wrong**: Blocks legitimate use cases, poor developer experience
   **Instead**: Start conservative, monitor, and adjust based on actual usage patterns

---

## Related Patterns

- **[Circuit Breaker](../01_Distributed_Systems/circuit_breaker_patterns.md)** - For downstream protection when services are overloaded
- **[Load Balancing](../07_Performance_Optimization/load_balancing.md)** - Distribute traffic across servers
- **[API Versioning](../api_versioning/README.md)** - Different rate limits per API version
- **[Idempotency Patterns](../idempotency_patterns/README.md)** - Make retries safe when rate limited
- **[Caching Strategies](../07_Performance_Optimization/caching_strategies.md)** - Cache rate limit checks for performance

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for complete list of:
- Engineering blog posts (Stripe, GitHub, Cloudflare)
- Conference talks (QCon, AWS re:Invent)
- Technical papers and algorithms
- Documentation references
