# Project 03: Multi-Tenant SaaS API Gateway

## Difficulty Level: Intermediate
**Estimated Time**: 2-3 weeks per language
**Prerequisites**: REST APIs, Authentication, Rate Limiting

---

## What You're Building

A production-grade API Gateway that sits in front of multiple microservices, handling authentication, rate limiting, request routing, and multi-tenancy (like Kong, AWS API Gateway, or Apigee).

## Layman Explanation

Think of this as a **security guard and traffic cop** for your building (backend services):

1. **Security Guard**: Checks if visitors (requests) have valid ID (authentication)
2. **Traffic Cop**: Directs visitors to the right office (routing)
3. **Bouncer**: Limits how many times someone can enter (rate limiting)
4. **Accountant**: Tracks usage for billing (analytics)
5. **Multi-Building Manager**: Each tenant gets their own space with own rules

---

## Key Concepts

### 1. Authentication & Authorization
- JWT validation
- API key management
- OAuth 2.0 flows
- Role-Based Access Control (RBAC)
- Multi-tenant isolation

### 2. Rate Limiting
- Token Bucket algorithm
- Sliding Window algorithm
- Distributed rate limiting (Redis)
- Per-tenant limits
- Burst handling

### 3. Request Routing
- Path-based routing
- Header-based routing
- Load balancing strategies
- Circuit breaker pattern
- Retry logic

### 4. Multi-Tenancy
- Tenant isolation
- Per-tenant configuration
- Resource quotas
- Tenant-specific rate limits
- Data segregation

---

## Architecture

```
Client → API Gateway → [Auth Check] → [Rate Limit] → [Route] → Microservice
           ↓
      [Analytics]
      [Logging]
      [Metrics]
```

---

## Database Schema

```sql
-- Tenants
CREATE TABLE tenants (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    api_key VARCHAR(64) UNIQUE NOT NULL,
    tier VARCHAR(20) NOT NULL, -- free, basic, premium, enterprise
    rate_limit_per_minute INT NOT NULL,
    rate_limit_per_hour INT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- API Routes
CREATE TABLE routes (
    id BIGSERIAL PRIMARY KEY,
    path_pattern VARCHAR(255) NOT NULL,
    upstream_url VARCHAR(500) NOT NULL,
    methods VARCHAR(50)[], -- ['GET', 'POST']
    auth_required BOOLEAN DEFAULT TRUE,
    rate_limit_override INT, -- Optional route-specific limit
    timeout_ms INT DEFAULT 30000,
    is_active BOOLEAN DEFAULT TRUE
);

-- Rate Limit Tracking (Redis)
Key: rate_limit:{tenant_id}:{window}
Value: Counter
TTL: window duration
```

---

## API Endpoints

### Gateway Admin API

```http
# Create Route
POST /admin/routes
{
  "path": "/api/users/*",
  "upstream": "http://user-service:3000",
  "methods": ["GET", "POST"],
  "auth_required": true,
  "rate_limit": 100
}

# Create Tenant
POST /admin/tenants
{
  "name": "Acme Corp",
  "tier": "premium",
  "rate_limit_per_minute": 1000
}
```

### Gateway Proxy

```http
# All client requests go through gateway
GET /api/users/123
X-API-Key: tenant_api_key_xyz

# Gateway:
# 1. Validates API key
# 2. Checks rate limit
# 3. Routes to http://user-service:3000/api/users/123
# 4. Returns response
```

---

## Rate Limiting Algorithms

### Token Bucket

```typescript
class TokenBucket {
  private tokens: number;
  private capacity: number;
  private refillRate: number; // tokens per second

  async consume(count: number = 1): Promise<boolean> {
    await this.refill();

    if (this.tokens >= count) {
      this.tokens -= count;
      return true; // Allow request
    }

    return false; // Deny request (429)
  }

  private async refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRate;

    this.tokens = Math.min(
      this.capacity,
      this.tokens + tokensToAdd
    );
    this.lastRefill = now;
  }
}
```

### Sliding Window (Redis)

```typescript
async function checkRateLimit(
  tenantId: string,
  limit: number,
  windowSec: number
): Promise<boolean> {
  const now = Date.now();
  const windowStart = now - windowSec * 1000;
  const key = `rate_limit:${tenantId}`;

  // Remove old entries
  await redis.zremrangebyscore(key, 0, windowStart);

  // Count requests in window
  const count = await redis.zcard(key);

  if (count >= limit) {
    return false; // Rate limit exceeded
  }

  // Add current request
  await redis.zadd(key, now, `${now}-${Math.random()}`);
  await redis.expire(key, windowSec);

  return true; // Allow request
}
```

---

## Implementation Steps

### Week 1: Basic Gateway
1. HTTP proxy functionality
2. Route management
3. Basic authentication (API keys)
4. Request/response logging

### Week 2: Rate Limiting & Multi-Tenancy
1. Implement token bucket algorithm
2. Redis integration for distributed rate limiting
3. Tenant management
4. Per-tenant rate limits

### Week 3: Advanced Features
1. Circuit breaker for upstream services
2. Request retry logic
3. Health checks
4. Analytics and metrics
5. WebSocket support

---

## Technology Stack

### NestJS
- @nestjs/throttler
- http-proxy-middleware
- ioredis

### Golang
- net/http/httputil (ReverseProxy)
- go-redis
- rate limiter library

### Python
- FastAPI
- httpx (async HTTP client)
- redis-py

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Latency Overhead | < 10ms |
| Throughput | 10,000 req/s |
| Rate Limit Check | < 5ms |
| Auth Check | < 10ms |

---

## Success Criteria

✅ Handle 10,000+ req/s
✅ Sub-10ms overhead
✅ Accurate rate limiting
✅ Zero auth bypasses
✅ Comprehensive logging
✅ Circuit breaker working
✅ Multi-tenant isolation

---

**Next**: Move to Project 04 (Payment Engine) for advanced distributed transactions!
