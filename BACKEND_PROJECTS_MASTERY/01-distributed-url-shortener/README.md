# Project 01: Distributed URL Shortener

## Difficulty Level: Intermediate
**Estimated Time**: 2-3 weeks per language
**Prerequisites**: HTTP, REST APIs, Databases, Caching basics

---

## What You're Building

A high-performance URL shortening service similar to bit.ly or tinyurl.com that can handle millions of requests per day. Users submit long URLs and receive short, unique codes that redirect to the original URL.

---

## Layman Explanation

Think of this like a **coat check service at a restaurant**:

1. You arrive with a big bulky coat (long URL: `https://www.example.com/products/category/electronics/smartphones/model-xyz-123`)
2. The attendant gives you a small ticket with a number (short URL: `http://short.ly/abc123`)
3. When you're ready to leave, you show the ticket and get your coat back (redirecting from short URL to long URL)
4. The service needs to handle hundreds of people simultaneously without losing anyone's coat

The challenge: Making this system fast, reliable, and able to serve millions of people across the world.

---

## Real-World Use Cases

1. **Social Media**: Twitter's character limit makes short URLs essential
2. **Marketing**: Track link clicks and analytics
3. **SMS**: Short URLs save space in text messages
4. **Print Media**: QR codes and printed materials need short, memorable URLs
5. **Email**: Clean, professional-looking links in email campaigns

---

## Technical Requirements

### Functional Requirements
1. **Shorten URL**: Accept a long URL and return a unique short code
2. **Redirect**: When someone visits the short URL, redirect to original URL
3. **Custom Aliases**: Optional custom short codes (e.g., `short.ly/myproduct`)
4. **Expiration**: URLs can have time-to-live (TTL)
5. **Analytics**: Track click count, referrers, geographic location
6. **API Rate Limiting**: Prevent abuse

### Non-Functional Requirements
1. **High Availability**: 99.9% uptime
2. **Low Latency**: Redirects should happen in < 200ms
3. **Scalability**: Handle 1000+ requests per second
4. **Data Consistency**: No duplicate short codes
5. **Fault Tolerance**: System continues working if some servers fail

---

## Key Concepts You'll Learn

### 1. System Design Fundamentals
- **Scalability**: Horizontal scaling with load balancer
- **High Availability**: Multiple servers, no single point of failure
- **CAP Theorem**: Choosing between consistency and availability

### 2. Data Structures & Algorithms
- **Hashing**: Generating unique short codes
- **Base62 Encoding**: Converting numbers to alphanumeric strings
- **Bloom Filters**: Quick duplicate detection
- **Consistent Hashing**: Distributing data across cache servers

### 3. Database Design
- **Indexing**: Fast lookups on short_code column
- **Sharding**: Distributing data across multiple databases
- **Replication**: Master-slave for read scaling
- **NoSQL vs SQL**: When to use each

### 4. Caching Strategies
- **Cache-Aside Pattern**: Check cache first, then database
- **TTL (Time-To-Live)**: Automatic cache expiration
- **Cache Invalidation**: When to remove cached entries
- **Multi-Level Caching**: Application + Redis + CDN

### 5. Distributed Systems
- **Distributed ID Generation**: Snowflake algorithm, UUID
- **Race Conditions**: Preventing duplicate short codes
- **Idempotency**: Handling duplicate requests
- **Clock Synchronization**: Dealing with server time differences

### 6. Performance Optimization
- **Connection Pooling**: Reuse database connections
- **Batch Processing**: Handle multiple operations together
- **Query Optimization**: Using EXPLAIN to optimize SQL
- **Profiling**: Identifying bottlenecks

### 7. API Design
- **RESTful Endpoints**: POST /shorten, GET /:shortCode
- **Status Codes**: 201 Created, 301 Moved Permanently, 404 Not Found
- **Rate Limiting**: Token bucket algorithm
- **Versioning**: /v1/shorten for API versioning

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT REQUESTS                          │
│                     (Browsers, Mobile Apps)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       LOAD BALANCER                              │
│                    (Round Robin, Least Conn)                     │
└────────────┬──────────────────────────────────┬─────────────────┘
             │                                   │
    ┌────────▼────────┐                 ┌───────▼────────┐
    │  APP SERVER 1   │                 │  APP SERVER 2   │
    │  (API Service)  │                 │  (API Service)  │
    └────────┬────────┘                 └───────┬─────────┘
             │                                   │
             └───────────────┬───────────────────┘
                             │
                    ┌────────▼─────────┐
                    │   REDIS CACHE    │
                    │  (Hot URLs)      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    POSTGRESQL   │
                    │   (Master DB)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  READ REPLICAS  │
                    │  (Slave DBs)    │
                    └─────────────────┘
```

### Component Responsibilities

1. **Load Balancer**
   - Distributes traffic across multiple app servers
   - Health checks to remove unhealthy servers
   - SSL termination

2. **App Servers**
   - Handle API requests (shorten URL, redirect)
   - Business logic validation
   - Generate short codes
   - Interact with cache and database

3. **Redis Cache**
   - Cache popular URLs (80/20 rule - 20% URLs get 80% traffic)
   - Reduce database load
   - Sub-millisecond response times
   - Distributed cache with replication

4. **PostgreSQL Master**
   - Write operations (new short URLs)
   - ACID guarantees for data consistency
   - Unique constraints on short_code

5. **Read Replicas**
   - Handle read-heavy traffic (redirects)
   - Asynchronous replication from master
   - Scale horizontally for more reads

---

## Database Schema

### URLs Table (PostgreSQL)

```sql
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    custom_alias BOOLEAN DEFAULT FALSE,
    user_id BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,

    -- Indexes for performance
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);

CREATE TABLE analytics (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL,
    clicked_at TIMESTAMP NOT NULL DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT,
    referer TEXT,
    country VARCHAR(2),
    city VARCHAR(100),

    -- Indexes
    INDEX idx_short_code (short_code),
    INDEX idx_clicked_at (clicked_at)
);

CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    api_key VARCHAR(64) UNIQUE NOT NULL,
    tier VARCHAR(20) DEFAULT 'free', -- free, premium
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### Redis Cache Structure

```
Key Pattern: url:{short_code}
Value: JSON {
    "long_url": "https://...",
    "expires_at": "2026-12-31T23:59:59Z"
}
TTL: 3600 seconds (1 hour)

Key Pattern: stats:{short_code}:count
Value: Integer (click count)
TTL: 86400 seconds (24 hours)
```

---

## API Endpoints

### 1. Shorten URL
```http
POST /api/v1/shorten
Content-Type: application/json
Authorization: Bearer <api_key>

{
  "long_url": "https://www.example.com/very/long/url",
  "custom_alias": "mylink", // optional
  "expires_in_days": 30 // optional
}

Response 201:
{
  "short_code": "abc123",
  "short_url": "https://short.ly/abc123",
  "long_url": "https://www.example.com/very/long/url",
  "expires_at": "2026-03-18T10:30:00Z"
}
```

### 2. Redirect to Original URL
```http
GET /:short_code

Response 301:
Location: https://www.example.com/very/long/url
```

### 3. Get URL Analytics
```http
GET /api/v1/analytics/:short_code
Authorization: Bearer <api_key>

Response 200:
{
  "short_code": "abc123",
  "total_clicks": 1247,
  "unique_clicks": 892,
  "created_at": "2026-02-15T10:30:00Z",
  "last_clicked": "2026-02-16T09:15:00Z",
  "top_countries": [
    {"country": "US", "clicks": 450},
    {"country": "UK", "clicks": 230}
  ],
  "top_referers": [
    {"referer": "twitter.com", "clicks": 320},
    {"referer": "facebook.com", "clicks": 180}
  ]
}
```

### 4. Delete URL
```http
DELETE /api/v1/urls/:short_code
Authorization: Bearer <api_key>

Response 204: No Content
```

---

## Algorithm: Generating Short Codes

### Approach 1: Base62 Encoding (Counter-based)
```
1. Maintain a global counter (stored in DB)
2. When new URL comes: counter++
3. Convert counter to Base62 (0-9, a-z, A-Z)
4. Result: 1 -> "1", 62 -> "10", 3844 -> "100"

Pros: Short codes, predictable
Cons: Sequential (security issue), single point of failure (counter)
```

### Approach 2: Hash + Collision Resolution (Recommended)
```
1. Hash the long URL using MD5/SHA256
2. Take first 7 characters of hash
3. Convert to Base62
4. Check if exists in DB
5. If collision, append counter or rehash with salt

Pros: Distributed, no central counter
Cons: Need collision handling, slightly longer codes
```

### Approach 3: Snowflake ID (Twitter)
```
Structure: 64 bits
- 1 bit: unused
- 41 bits: timestamp (milliseconds)
- 10 bits: machine ID
- 12 bits: sequence number

Pros: Distributed, time-sorted, unique
Cons: Longer codes (need Base62 encoding)
```

**Our Choice**: Hybrid approach
- Use Snowflake for ID generation
- Convert to Base62 for short code
- Results in ~7 character codes

---

## Caching Strategy

### Multi-Level Cache

```
Request Flow:

1. Client Request → App Server
2. Check Application Memory Cache (local LRU)
   ├─ Hit? Return immediately
   └─ Miss? Continue to Redis
3. Check Redis Cluster
   ├─ Hit? Return + update local cache
   └─ Miss? Continue to Database
4. Query PostgreSQL (Read Replica)
5. Store in Redis + Local Cache
6. Return to client
```

### Cache Invalidation Rules

1. **TTL-based**: Popular URLs cached for 1 hour
2. **Event-based**: When URL is deleted, invalidate cache
3. **Write-through**: When URL is updated, update cache immediately
4. **Probabilistic early expiration**: Prevent cache stampede

### Cache Stampede Prevention

```python
# Problem: Cache expires, 1000 requests hit DB simultaneously

# Solution: Locking
def get_url(short_code):
    # Try cache
    data = redis.get(f"url:{short_code}")
    if data:
        return data

    # Acquire lock
    lock_key = f"lock:url:{short_code}"
    if redis.set(lock_key, "1", nx=True, ex=10):
        # This thread acquired lock, fetch from DB
        data = db.query(short_code)
        redis.set(f"url:{short_code}", data, ex=3600)
        redis.delete(lock_key)
        return data
    else:
        # Another thread is fetching, wait and retry
        time.sleep(0.1)
        return get_url(short_code)
```

---

## Implementation Steps

### Phase 1: Basic Implementation (Week 1)
1. Set up project structure (NestJS/Golang/Python)
2. Create database schema
3. Implement POST /shorten endpoint
   - Validate URL
   - Generate short code
   - Save to database
   - Return short URL
4. Implement GET /:shortCode redirect
   - Query database
   - Return 301 redirect
5. Basic error handling
6. Unit tests

### Phase 2: Caching (Week 2)
1. Set up Redis
2. Implement cache-aside pattern
3. Add cache for redirects
4. Implement cache invalidation
5. Add metrics for cache hit/miss rate
6. Load testing with caching

### Phase 3: Advanced Features (Week 3)
1. Implement custom aliases
2. Add URL expiration
3. Implement analytics tracking
4. Add rate limiting
5. User authentication (API keys)
6. Distributed ID generation (Snowflake)

### Phase 4: Production Ready (Week 4)
1. Set up read replicas
2. Implement connection pooling
3. Add distributed tracing
4. Set up monitoring (Prometheus + Grafana)
5. Load balancing configuration
6. Comprehensive integration tests
7. Performance benchmarking

---

## Testing Strategy

### Unit Tests
- Short code generation logic
- URL validation
- Base62 encoding/decoding
- Cache hit/miss scenarios

### Integration Tests
- API endpoint testing
- Database operations
- Cache operations
- Rate limiting

### Load Tests
```bash
# Using k6
k6 run --vus 100 --duration 30s load-test.js

# Expected Results:
- Shorten URL: 500 req/s, p95 < 100ms
- Redirect: 2000 req/s, p95 < 50ms
- Cache hit rate: > 80%
```

### Failure Tests
- Database connection failure
- Cache server failure
- Network partition
- High load (10x normal traffic)

---

## Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Redirect Latency (p95) | < 50ms | Cache hit |
| Redirect Latency (p95) | < 200ms | Cache miss |
| Shorten URL (p95) | < 100ms | With DB write |
| Throughput | 2000 req/s | Single server |
| Cache Hit Rate | > 80% | Redis metrics |
| Database Connection Pool | 20-50 | Per server |
| Error Rate | < 0.1% | All requests |

---

## Technology Stack

### NestJS Implementation
- NestJS 10+ (Framework)
- TypeORM (ORM)
- ioredis (Redis client)
- class-validator (Validation)
- @nestjs/throttler (Rate limiting)
- jest (Testing)

### Golang Implementation
- Gin or Fiber (Web framework)
- GORM (ORM)
- go-redis (Redis client)
- validator (Validation)
- rate limiter library
- testify (Testing)

### Python Implementation
- FastAPI (Framework)
- SQLAlchemy (ORM)
- redis-py (Redis client)
- pydantic (Validation)
- slowapi (Rate limiting)
- pytest (Testing)

### Infrastructure
- PostgreSQL 15
- Redis 7 (Cluster mode)
- Docker & Docker Compose
- Nginx (Load balancer)
- Prometheus + Grafana (Monitoring)

---

## Deployment Architecture

```
                    [Internet]
                        │
                   [CDN / WAF]
                        │
                  [Load Balancer]
                   /     |     \
            [Server 1] [Server 2] [Server 3]
                   \     |     /
                  [Redis Cluster]
                   /           \
            [PostgreSQL]  [Read Replicas]
```

---

## Scaling Strategy

### Vertical Scaling (Initial)
- Increase server resources (CPU, RAM)
- Optimize queries
- Add indexes

### Horizontal Scaling (Growth)
- Add more app servers
- Redis cluster (sharding)
- Database read replicas
- CDN for popular URLs

### Database Sharding (High Scale)
- Shard by hash(short_code) % num_shards
- Each shard handles subset of URLs
- Consistent hashing for even distribution

---

## Monitoring & Observability

### Key Metrics
1. **Latency**: p50, p95, p99 for redirects
2. **Throughput**: Requests per second
3. **Error Rate**: 4xx, 5xx errors
4. **Cache Hit Rate**: Redis hits vs misses
5. **Database**: Query time, connection pool utilization
6. **System**: CPU, Memory, Disk, Network

### Alerts
- Latency p95 > 200ms
- Error rate > 1%
- Cache hit rate < 70%
- Database connection pool > 80%
- Disk space > 80%

### Logs
```json
{
  "timestamp": "2026-02-16T10:30:00Z",
  "level": "info",
  "service": "url-shortener",
  "trace_id": "abc123def456",
  "event": "url_shortened",
  "short_code": "abc123",
  "long_url": "https://...",
  "user_id": 12345,
  "latency_ms": 45
}
```

---

## Common Pitfalls & Solutions

### 1. Duplicate Short Codes
**Problem**: Two requests generate same short code
**Solution**: Database unique constraint + retry logic

### 2. Hot Spot URLs
**Problem**: One URL gets 90% of traffic
**Solution**: Multi-level caching, CDN

### 3. Cache Stampede
**Problem**: Cache expires, all requests hit DB
**Solution**: Locking, probabilistic early expiration

### 4. Database Bottleneck
**Problem**: Too many writes to master
**Solution**: Read replicas, write buffering, batch inserts

### 5. Malicious URLs
**Problem**: Users submit phishing/malware URLs
**Solution**: URL validation, Google Safe Browsing API

---

## Advanced Challenges

Once you've built the basic system, try these:

1. **Geo-Distributed**: Deploy across multiple regions (US, EU, Asia)
2. **Custom Domains**: Allow users to use their own domains
3. **QR Code Generation**: Generate QR codes for short URLs
4. **A/B Testing**: Support multiple destinations for same short code
5. **Bulk API**: Shorten 1000 URLs in one request
6. **Analytics Dashboard**: Real-time charts and graphs
7. **Webhook Integration**: Notify on URL clicks
8. **Link Preview**: Show preview of destination before redirect

---

## Learning Resources

### Articles
- "Design URL Shortener" - System Design Interview
- "Distributed ID Generation" - Twitter Snowflake
- "Consistent Hashing Explained"

### Videos
- "How to Design TinyURL" - System Design
- "Caching Strategies" - High Scalability

### Code Examples
- bitly/bitly-api (GitHub)
- short (Go implementation)

---

## Success Criteria

You've successfully completed this project when you can:

1. ✅ Explain the system architecture and trade-offs
2. ✅ Handle 1000+ requests per second
3. ✅ Achieve < 200ms p95 latency for redirects
4. ✅ Implement proper caching with > 80% hit rate
5. ✅ Set up monitoring and alerting
6. ✅ Write comprehensive tests (unit, integration, load)
7. ✅ Deploy the system with Docker Compose
8. ✅ Document architectural decisions
9. ✅ Optimize based on profiling data
10. ✅ Compare implementations across three languages

---

## Next Steps

After completing this project:

1. Move to **Project 02: Real-Time Collaborative Task Manager** (WebSockets, CQRS)
2. Or dive deeper: Implement geo-distributed deployment
3. Write a blog post explaining your architecture decisions
4. Present your system design to peers for feedback

---

Happy Building! 🚀
