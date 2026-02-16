# Project 01: Design a Global Content Delivery Network (CDN)

## Difficulty: Intermediate
**Estimated Time**: 3-4 weeks

## Problem Statement

Design a Content Delivery Network (CDN) like Cloudflare, Akamai, or AWS CloudFront that serves static content (images, videos, CSS, JS) to users worldwide with minimal latency.

**Real-World Example**: When you visit Netflix, the video streams from a nearby CDN server, not from Netflix's origin servers in California. This reduces latency from 500ms to 20ms.

## Functional Requirements

### Core Features:
1. **Content Caching**: Cache static assets at edge locations
2. **Content Delivery**: Serve cached content to users from nearest edge
3. **Origin Fetch**: Fetch content from origin if not cached
4. **Cache Invalidation**: Remove/update cached content
5. **Content Upload**: Allow customers to upload content to CDN
6. **Analytics**: Track bandwidth usage, cache hit ratio, requests per region

### Out of Scope (for this design):
- DDoS protection
- WAF (Web Application Firewall)
- Rate limiting per customer
- Video transcoding

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Availability** | 99.99% | < 53 minutes downtime/year |
| **Latency** | < 50ms | From user to edge server |
| **Cache Hit Ratio** | > 90% | Percentage of requests served from cache |
| **Throughput** | 1M requests/sec | Per region |
| **Storage** | Petabyte-scale | Across all edge locations |
| **Bandwidth** | 100 Tbps | Global capacity |

## Capacity Estimation

### Assumptions:
- **10 million customers** using the CDN
- **100 billion requests per day** (1.16M requests/second)
- **Average object size**: 500 KB
- **Cache hit ratio**: 90%
- **Data transfer per day**: 100B requests × 500 KB = 50 PB/day

### Storage Calculation:
```
Popular content (top 10% accessed frequently):
- 10M customers × 1000 objects/customer = 10B objects
- Top 10% = 1B objects to cache heavily
- 1B objects × 500 KB = 500 PB of hot storage

Total storage needed (with replication):
- Hot: 500 PB × 3 replicas = 1.5 EB
- Cold: Remaining in origin
```

### Bandwidth Calculation:
```
Requests per second: 100B / 86400 = 1.16M RPS

With 90% cache hit ratio:
- Cache hits: 1.16M × 0.9 = 1.04M RPS (served from edge)
- Cache misses: 1.16M × 0.1 = 116K RPS (fetch from origin)

Bandwidth needed:
- Edge to users: 1.16M RPS × 500 KB = 580 GB/s = 4.6 Tbps
- Origin to edge: 116K RPS × 500 KB = 58 GB/s = 464 Gbps
```

### Number of Edge Locations:
```
Target: < 50ms latency to users

Coverage needed:
- Major cities worldwide: ~200 locations
- Tier 2 cities: ~500 locations
- Total: ~700 edge locations (PoPs)

Each location capacity:
- 1.16M RPS / 700 = ~1660 RPS per location
- Storage: 500 PB / 700 = ~714 TB per location
```

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Customer                             │
│                    (Content Owner)                          │
└──────────────┬──────────────────────────────────────────────┘
               │
               │ 1. Upload content
               ▼
     ┌─────────────────────┐
     │   Origin Server     │
     │  (Customer's CDN)   │
     └─────────┬───────────┘
               │
               │ 2. Content propagates to edges
               │
┌──────────────┴───────────────────────────────────────┐
│                                                       │
│            Global Distribution Network                │
│                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  Edge    │    │  Edge    │    │  Edge    │      │
│  │  NA-East │    │  EU-West │    │  Asia-SE │      │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘      │
│       │               │               │             │
└───────┼───────────────┼───────────────┼─────────────┘
        │               │               │
        │ 3. User requests content      │
        ▼               ▼               ▼
   ┌────────┐      ┌────────┐      ┌────────┐
   │  User  │      │  User  │      │  User  │
   │  (US)  │      │  (EU)  │      │  (Asia)│
   └────────┘      └────────┘      └────────┘
```

## Detailed Design

### 1. Request Flow

```
User → DNS Resolution → Edge Server → [Cache Hit?]
                                          │
                      ┌───────────────────┴─────────────────┐
                      │                                     │
                   Cache HIT                            Cache MISS
                      │                                     │
                      ▼                                     ▼
              Return from cache              Fetch from origin server
              (< 20ms)                               │
                                                     ▼
                                          Cache at edge + Return to user
                                                 (100-200ms first time)
```

### 2. Components Breakdown

#### 2.1 Edge Server (PoP - Point of Presence)

**Responsibilities**:
- Serve cached content
- Handle cache misses by fetching from origin
- Implement caching policies (TTL, LRU)
- Monitor health and report metrics

**Cache Storage**:
```
Tiered Storage:
├── L1: Memory (SSD) - Hot content (1-10 TB)
│   - Most frequently accessed
│   - TTL: 1 hour to 1 day
│
├── L2: Disk (HDD) - Warm content (100 TB)
│   - Less frequently accessed
│   - TTL: 1 day to 1 week
│
└── L3: Origin - Cold content (Infinite)
    - Rarely accessed
    - Fetch on demand
```

**Caching Strategy**:
```typescript
// LRU Cache with TTL
interface CacheEntry {
  key: string;
  value: Buffer;
  size: number;
  ttl: number;
  createdAt: number;
  lastAccessedAt: number;
}

class EdgeCache {
  private cache = new Map<string, CacheEntry>();
  private maxSize: number; // e.g., 1 TB
  private currentSize: number = 0;

  async get(key: string): Promise<Buffer | null> {
    const entry = this.cache.get(key);

    if (!entry) return null;

    // Check TTL
    if (Date.now() - entry.createdAt > entry.ttl) {
      this.evict(key);
      return null; // Expired
    }

    // Update last accessed time (for LRU)
    entry.lastAccessedAt = Date.now();
    return entry.value;
  }

  async set(key: string, value: Buffer, ttl: number) {
    const size = value.length;

    // Evict if needed
    while (this.currentSize + size > this.maxSize) {
      this.evictLRU();
    }

    this.cache.set(key, {
      key,
      value,
      size,
      ttl,
      createdAt: Date.now(),
      lastAccessedAt: Date.now()
    });

    this.currentSize += size;
  }

  private evictLRU() {
    let oldestKey: string | null = null;
    let oldestTime = Infinity;

    for (const [key, entry] of this.cache) {
      if (entry.lastAccessedAt < oldestTime) {
        oldestTime = entry.lastAccessedAt;
        oldestKey = key;
      }
    }

    if (oldestKey) this.evict(oldestKey);
  }

  private evict(key: string) {
    const entry = this.cache.get(key);
    if (entry) {
      this.currentSize -= entry.size;
      this.cache.delete(key);
    }
  }
}
```

#### 2.2 Global Load Balancer (DNS-based)

**DNS Resolution**:
```
User requests: cdn.example.com
                    ↓
              DNS Lookup
                    ↓
    ┌───────────────┴───────────────┐
    │                               │
    │  GeoDNS (Route 53, etc.)     │
    │  - Check user IP geolocation  │
    │  - Find nearest edge servers  │
    │  - Return IP of closest edge  │
    │                               │
    └───────────────┬───────────────┘
                    ↓
    Returns: edge-us-east.cdn.example.com (1.2.3.4)
```

**GeoDNS Logic**:
```python
def get_nearest_edge(user_ip: str) -> str:
    # 1. Get user location from IP
    user_lat, user_lon = geolocate_ip(user_ip)

    # 2. Get all healthy edge servers
    healthy_edges = get_healthy_edges()

    # 3. Calculate distance to each edge
    distances = []
    for edge in healthy_edges:
        distance = haversine_distance(
            user_lat, user_lon,
            edge.lat, edge.lon
        )
        distances.append((edge, distance))

    # 4. Sort by distance and return closest
    distances.sort(key=lambda x: x[1])
    return distances[0][0].ip_address
```

#### 2.3 Origin Server

**Responsibilities**:
- Store original content
- Handle cache misses from edge servers
- Process invalidation requests
- Serve as source of truth

**Origin Shield** (Optional Layer):
```
Edge Servers → Origin Shield → Origin
   (100s)          (5-10)         (1)

Purpose: Reduce load on origin by adding intermediate cache layer
Benefit: If 100 edge servers miss cache, only 1 request goes to origin
```

### 3. Cache Invalidation Strategies

**Problem**: How to update cached content when origin changes?

#### Strategy 1: TTL-based Expiration
```
Pros: Simple, automatic
Cons: Stale data until TTL expires

Example:
Cache-Control: max-age=3600  # 1 hour TTL
```

#### Strategy 2: Purge API (Immediate)
```
Customer calls: POST /purge
Body: { "urls": ["/images/logo.png"] }

CDN:
1. Marks content as invalid in database
2. Sends purge signal to all edge servers
3. Edge servers delete cached content
4. Next request fetches fresh content

Latency: 5-10 seconds global propagation
```

#### Strategy 3: Versioned URLs
```
Pros: No purge needed, instant updates
Cons: Requires client-side coordination

Example:
Old: cdn.example.com/logo.png
New: cdn.example.com/logo.png?v=2

Each version is a new cache key
```

#### Strategy 4: Cache Tags
```
Tag related content:
- /products/123/image.jpg → tags: [product-123, images]
- /products/123/desc.html → tags: [product-123, html]

Purge by tag:
POST /purge { "tags": ["product-123"] }
→ Invalidates all content with that tag
```

## Database Design

### Metadata Store (PostgreSQL)

```sql
-- Customers (Content Owners)
CREATE TABLE customers (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  api_key VARCHAR(255) UNIQUE,
  bandwidth_limit_gbps DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Origins (Customer's origin servers)
CREATE TABLE origins (
  id SERIAL PRIMARY KEY,
  customer_id INT REFERENCES customers(id),
  domain VARCHAR(255),
  origin_url VARCHAR(255),
  health_check_url VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Cached Objects Metadata
CREATE TABLE cached_objects (
  id BIGSERIAL PRIMARY KEY,
  customer_id INT REFERENCES customers(id),
  url_path VARCHAR(500),
  content_hash VARCHAR(64),  -- SHA-256 of content
  size_bytes BIGINT,
  content_type VARCHAR(100),
  ttl_seconds INT,
  cache_tags TEXT[],  -- For invalidation
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,

  INDEX idx_customer_url (customer_id, url_path),
  INDEX idx_content_hash (content_hash),
  INDEX idx_cache_tags USING GIN(cache_tags)
);

-- Edge Locations
CREATE TABLE edge_locations (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  region VARCHAR(50),
  latitude DECIMAL(9,6),
  longitude DECIMAL(9,6),
  capacity_gbps DECIMAL(10,2),
  storage_tb DECIMAL(10,2),
  is_healthy BOOLEAN DEFAULT true,
  last_health_check TIMESTAMP
);

-- Cache Statistics (Time-series data)
CREATE TABLE cache_stats (
  id BIGSERIAL PRIMARY KEY,
  edge_location_id INT REFERENCES edge_locations(id),
  timestamp TIMESTAMP,
  requests_per_second INT,
  cache_hit_ratio DECIMAL(5,2),  -- 0.00 to 100.00
  bandwidth_gbps DECIMAL(10,2),
  avg_latency_ms INT,

  INDEX idx_edge_time (edge_location_id, timestamp)
);
```

### Cache Storage (Redis)

```redis
# Cache Entry
Key: cache:{customer_id}:{url_path}
Value: {
  "content_hash": "abc123...",
  "location": "s3://bucket/abc123",  # Pointer to actual content
  "ttl": 3600,
  "size": 102400,
  "cached_at": 1708084800
}
TTL: 3600 seconds

# Popular Content Tracking
Key: popular:{customer_id}
Type: ZSET (Sorted Set)
Members: url_path
Score: access_count
Commands:
  ZINCRBY popular:123 1 "/logo.png"  # Increment access count
  ZREVRANGE popular:123 0 99  # Get top 100 popular
```

## API Design

### 1. Content Delivery API (Public)

```http
# Serve content (main API)
GET https://cdn.example.com/{customer}/{path}
Response: 200 OK
Headers:
  Content-Type: image/jpeg
  Cache-Control: max-age=3600
  X-Cache: HIT  # or MISS
  X-Cache-Server: us-east-1a
  Age: 1234  # Seconds since cached
Body: <binary content>
```

### 2. Customer API (Private)

```http
# Upload content to origin
POST /api/v1/content
Authorization: Bearer <api_key>
Body: multipart/form-data
{
  "file": <binary>,
  "path": "/images/logo.png",
  "content_type": "image/png",
  "cache_control": "max-age=3600"
}
Response: 201 Created
{
  "url": "https://cdn.example.com/customer123/images/logo.png",
  "size": 102400,
  "content_hash": "sha256:abc123..."
}

# Purge cache
POST /api/v1/purge
Authorization: Bearer <api_key>
Body:
{
  "urls": ["/images/logo.png"],
  "purge_type": "immediate"  # or "after_ttl"
}
Response: 200 OK
{
  "purged_count": 1,
  "propagation_time_seconds": 8
}

# Purge by tags
POST /api/v1/purge/tags
Body: { "tags": ["product-123"] }
Response: { "purged_count": 45 }

# Get analytics
GET /api/v1/analytics?start=2026-02-01&end=2026-02-16
Response:
{
  "total_requests": 1000000000,
  "cache_hit_ratio": 92.5,
  "bandwidth_gb": 5000000,
  "requests_by_region": {
    "us-east": 400000000,
    "eu-west": 300000000,
    "asia-se": 300000000
  },
  "top_content": [
    { "path": "/images/logo.png", "requests": 50000000 },
    { "path": "/css/main.css", "requests": 45000000 }
  ]
}
```

## Trade-off Analysis

### 1. TTL vs Manual Invalidation

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **TTL-based** | Simple, automatic, no API calls | Stale data possible, wastes bandwidth | Content doesn't change often (logos, libraries) |
| **Manual Purge** | Immediate updates, control | Requires API integration, propagation delay | Frequently updated content |
| **Versioned URLs** | Best of both worlds, instant | Requires client coordination | Build artifacts, immutable content |

**Recommendation**: Use versioned URLs for static assets (CSS, JS, images) with long TTL (1 year). Use purge API for dynamic content that needs immediate updates.

### 2. Pull vs Push CDN

| Approach | How it Works | Pros | Cons |
|----------|-------------|------|------|
| **Pull CDN** | Edge fetches from origin on cache miss | Simple, automatic, lazy loading | First request slow, origin must be accessible |
| **Push CDN** | Customer uploads directly to all edges | Fast first request, origin can be offline | Complex, bandwidth waste for unpopular content |

**Recommendation**: Pull CDN for most use cases. Use push for critical content that must be pre-warmed (e.g., major product launches).

### 3. Consistent Hashing for Cache Placement

```
Without Consistent Hashing:
- Edge server determined by: hash(url) % num_servers
- Problem: Adding/removing server invalidates most caches

With Consistent Hashing:
- Edge server determined by: closest server on hash ring
- Benefit: Only affects ~1/N of cache keys
```

## Failure Handling

### 1. Edge Server Failure

```
Detection:
- Health check fails 3 times in a row (30 seconds)
- Automatically remove from DNS rotation

Recovery:
- GeoDNS redirects traffic to next nearest edge
- Latency increases by ~20-50ms temporarily
- Ops team notified, server repaired/replaced

SLA Impact:
- No impact if >= 2 edges in region
- Users automatically failover
```

### 2. Origin Server Failure

```
Problem: Cache miss cannot be fulfilled

Solutions:
1. Serve stale content (if available)
   Cache-Control: stale-while-revalidate=86400

2. Return 503 with Retry-After header
   HTTP 503 Service Unavailable
   Retry-After: 60

3. Failover to backup origin
   Primary: origin-primary.example.com
   Secondary: origin-backup.example.com
```

### 3. Cache Stampede

**Problem**: Popular content expires, 1000s of requests hit origin simultaneously

**Solution**: Cache locking
```typescript
async function getWithLock(key: string): Promise<Buffer> {
  // Try to get from cache
  const cached = await cache.get(key);
  if (cached) return cached;

  // Acquire lock to fetch from origin
  const lock = await redis.set(
    `lock:${key}`,
    "1",
    "NX",  // Only set if not exists
    "EX", 10  // Expire after 10 seconds
  );

  if (lock) {
    // This request won the lock, fetch from origin
    const content = await fetchFromOrigin(key);
    await cache.set(key, content);
    await redis.del(`lock:${key}`);
    return content;
  } else {
    // Another request is fetching, wait and retry
    await sleep(100);
    return getWithLock(key);  // Retry
  }
}
```

### 4. Network Partition

**Scenario**: Edge location loses connectivity to origin

**Handling**:
```
1. Serve only cached content (Cache-only mode)
2. Return 503 for cache misses
3. Queue purge requests to apply when connection restored
4. Monitor partition duration, escalate if > 5 minutes
```

## Monitoring & Alerting

### Key Metrics

```yaml
Availability Metrics:
  - Edge server uptime: > 99.99%
  - Origin reachability: > 99.95%
  - DNS resolution success: > 99.99%

Performance Metrics:
  - P50 latency: < 20ms
  - P95 latency: < 50ms
  - P99 latency: < 100ms
  - Cache hit ratio: > 90%

Business Metrics:
  - Bandwidth usage per customer
  - Requests per customer
  - Cost per GB served
```

### Alerting Rules

```yaml
alerts:
  - name: LowCacheHitRatio
    condition: cache_hit_ratio < 85%
    duration: 5 minutes
    severity: warning
    action: Investigate cache TTL settings

  - name: HighOriginLoad
    condition: origin_requests > 50000 RPS
    duration: 2 minutes
    severity: critical
    action: Check for cache misconfiguration

  - name: EdgeServerDown
    condition: edge_health_check_failed
    duration: 30 seconds
    severity: critical
    action: Failover to backup edge

  - name: HighLatency
    condition: p95_latency > 100ms
    duration: 5 minutes
    severity: warning
    action: Check network congestion
```

## Scalability Considerations

### 1. Scaling Edge Locations

```
Growth Strategy:
- Start: 50 major cities (cover 80% of users)
- Year 1: 200 locations (cover 95% of users)
- Year 2: 500 locations (cover 99% of users)

Selection Criteria:
- User density (requests from region)
- Network connectivity
- Data center availability
- Cost per Gbps
```

### 2. Scaling Storage

```
Hot/Cold Separation:
- Hot tier (SSD): Top 10% content, 90% of requests
  - Cost: $0.20/GB/month
  - Latency: < 10ms

- Cold tier (HDD): Remaining 90% content, 10% of requests
  - Cost: $0.05/GB/month
  - Latency: 50-100ms

- Archive tier (S3 Glacier): Rarely accessed
  - Cost: $0.004/GB/month
  - Latency: 3-5 hours (not for CDN, only origin backup)
```

### 3. Scaling Control Plane

```
Metadata Database:
- PostgreSQL primary (writes)
- Read replicas in each region (reads)
- Sharded by customer_id if > 100M customers

Cache Invalidation:
- Kafka for purge messages (partition by customer_id)
- Each edge subscribes to Kafka
- Eventual consistency (5-10 seconds propagation)
```

## Further Optimizations

### 1. Smart Prefetching

```
Predict next requests based on patterns:
- User loads index.html
  → Prefetch main.css, logo.png, app.js

- User views product page
  → Prefetch product images, related products
```

### 2. Image Optimization

```
Automatic transformations:
- Original: logo.png (2 MB)
- Serve:
  - WebP format (500 KB) for modern browsers
  - JPEG (800 KB) for older browsers
  - Resize on-the-fly: ?width=300&height=200
```

### 3. Brotli/Gzip Compression

```
Compress text assets:
- HTML, CSS, JS: Brotli compression (~20% smaller than gzip)
- Store both compressed and uncompressed
- Serve based on Accept-Encoding header
```

## Interview Discussion Points

### Questions to Ask Interviewer:

1. **Scale**: How many users? How much traffic?
2. **Content Types**: Only static? Or also dynamic content?
3. **Regions**: Global or specific regions?
4. **SLA**: What latency is acceptable?
5. **Cost**: Optimize for cost or performance?
6. **Features**: Just caching or also DDoS, WAF, etc.?

### Deep Dive Topics:

1. **Cache Eviction**: LRU vs LFU vs TTL-based
2. **Consistency**: How to ensure all edges have same version?
3. **Security**: Signed URLs, token authentication
4. **Billing**: Track bandwidth per customer
5. **Anycast**: Single IP address for all edge locations

## Summary

### What We Built:
- Global CDN with 700 edge locations
- < 50ms latency worldwide
- 90%+ cache hit ratio
- Handles 1M+ RPS globally
- Petabyte-scale storage

### Key Concepts Covered:
- ✅ Caching strategies (TTL, LRU, purge)
- ✅ Geographic distribution (GeoDNS)
- ✅ Cache invalidation patterns
- ✅ Failure handling (origin down, cache stampede)
- ✅ Monitoring and SLAs

### Real-World Examples:
- Cloudflare: 275+ cities, 100+ Tbps
- Akamai: 300K+ servers, 135+ countries
- AWS CloudFront: 400+ PoPs, 90+ cities

---

**Next Project**: [02. Load Balancer & Health Check System](../02-load-balancer/README.md)
