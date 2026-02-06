# Advanced Software Engineering Case Studies & Scenarios - 2026
## For Senior Engineers & CTOs

---

## Original Case Studies (Foundational Questions)

### 1. Distributed Cache Consistency
You implement a distributed cache (e.g., Redis) in your microservices architecture to improve performance. However, during scaling, different services update the cache at different points, resulting in stale data being served to clients. How would you design a solution to handle cache consistency across multiple services to avoid stale data being served?

### 2. 503 Service Unavailable Investigation
Your service suddenly starts returning 503 Service Unavailable, but CPU and memory look fine. What do you investigate next?

### 3. Deep Pagination Request
As a backend dev, you see this request hitting your API from front-end:
```
GET /api/v1/trades?page=8000&size=50
```
What are your next steps?

### 4. Gzip Compression CPU Impact
Clients upload compressed JSON (gzip) to save bandwidth. CPU usage increased instead of decreasing. What would have happened here?

### 5. WhatsApp "Typing..." Indicator
As a Developer, have you asked yourself how "Typing…" shows up instantly in WhatsApp? Is the app refreshing every second? Or is there something else happening behind the scenes?

### 6. Netflix Resume Playback
As a developer, have you asked yourself how Netflix resumes exactly where you stopped? Is it just local storage, or synced state across devices?

### 7. SQL LIKE Performance
Which is faster in DB: `LIKE 'abc%'` or `LIKE '%abc'`?

### 8. Slow Query with Indexes
You see this query running slow:
```sql
SELECT *
FROM payments
WHERE LOWER(email) = 'john.doe@gmail.com'
  AND status != 'FAILED';
```
Index exists on email. Index exists on status. Table has millions of rows. What exactly is stopping the database from using your indexes efficiently here?

### 9. Netflix "Are You Still Watching?"
As a developer, have you asked yourself how Netflix knows when to show "Are you still watching?" Is it just time-based? If it were time-based, you'd see it every time.

### 10. Deduplication Across Apps
How do Instagram, Medium, and Tinder prevent showing you the same content twice? (Bloom filters explanation)

### 11. WhatsApp Read Receipts at Scale
You're building the "seen" / blue-tick feature (WhatsApp-style read receipts) for a messenger with 1B+ users. Naively adding an `is_read` boolean column kills write throughput. How do you actually make it scale while keeping semantics correct?

### 12. Duplicate Payment Prevention
User double clicks the pay button, how do you stop duplicate API calls?

### 13. Redis Queue Exactly-Once Processing
You're using Redis as a message queue, and messages are being processed twice by different workers. How will you ensure exactly-once processing?

### 14. ClickHouse Backup Optimization
Your ClickHouse backup takes 12 hours for a 50TB database, and you need faster recovery time. How will you optimize backup/restore strategy?

### 15. Zero-Downtime Deployments
A service runs 50 instances behind a load balancer, but during deploy, 10% of requests fail with 502 errors for 2 minutes. How will you achieve zero-downtime deployments?

### 16. Large File Download API
You are implementing a file download API, but some files are 5GB in size. How will you return the file to the client efficiently?

### 17. ClickHouse Streaming Latency
You're streaming real-time events to ClickHouse using Kafka, but data appears in queries 5-10 minutes late. How will you reduce this latency?

### 18. Cascading Service Failures
Your Order Service calls Inventory Service, which calls Pricing Service, which calls Tax Service. When Tax Service goes down, all orders fail. How will you make this more resilient?

### 19. Distributed Cache Consistency (Detailed)
How do you stop services from serving old cached data after an update? (Event-driven invalidation, single owner pattern)

### 20. Infinite Scroll Performance
You have a social-media infinite scroll feed with fully personalized content for 10M+ daily active users. Classic `OFFSET + LIMIT` queries get brutally slow as users scroll deeper (10k+ items in). How do you build a fast, reliable infinite scroll experience?

---

## Advanced Case Studies (Extended Questions)

### 21. Multi-Region Database Failover
Your globally distributed application uses PostgreSQL with read replicas across 5 regions. During a regional AWS outage, automated failover to the secondary region succeeded, but you lost 45 seconds of committed transactions. Your SLA promises RPO < 10 seconds. How would you redesign your data replication strategy to meet this requirement?

### 22. WebSocket Connection Storm
Your real-time collaboration platform supports 500K concurrent WebSocket connections. During a marketing campaign, 200K users joined simultaneously within 90 seconds. Load balancers started dropping connections, and existing users experienced disconnections. What's your systematic approach to handle connection storms while maintaining service for existing users?

### 23. GraphQL N+1 at Scale
Your GraphQL API serves mobile apps with complex nested queries. Monitoring shows a single query like:
```graphql
query {
  users(limit: 100) {
    posts {
      comments {
        author { profile { avatar } }
      }
    }
  }
}
```
triggers 15,000+ database queries and takes 45 seconds. DataLoader is already implemented. What's actually happening, and how do you solve it?

### 24. Event Sourcing Replay Performance
Your event-sourced system stores 5 years of events (2 billion events). Rebuilding aggregate state from events now takes 4 minutes per entity, making customer support investigations impossible. How do you optimize event replay without compromising audit requirements?

### 25. Kubernetes Pod Eviction Crisis
Your Kubernetes cluster starts evicting pods every few hours due to memory pressure, but actual memory usage is only at 60%. Pods have memory limits set correctly. What hidden Kubernetes behaviors could cause this, and how do you diagnose it?

### 26. Rate Limiter Race Condition
You implemented a sliding window rate limiter using Redis. During load testing at 50K req/s, some users exceed their limits by 15-20%, while others get falsely limited. The Redis instance shows no errors. What race conditions exist in typical Redis-based rate limiters, and how do you fix them?

### 27. Microservices Transaction Consistency
You're migrating a monolith to microservices. The checkout flow requires updating inventory (Inventory Service), charging payment (Payment Service), and creating an order (Order Service). A payment succeeds but inventory update fails due to network timeout. How do you ensure consistency without distributed transactions?

### 28. CDN Cache Poisoning
Your Next.js app behind Cloudflare CDN suddenly serves admin panel content to regular users for 2 hours. No deployment happened. Cache headers seem correct. What CDN-specific edge cases could cause this, and how do you prevent it?

### 29. gRPC Deadline Propagation
Your gRPC microservices chain: API Gateway → Service A → Service B → Service C. A client sets a 5-second deadline, but Service C still processes requests that should have timed out 10 seconds ago, wasting resources. What's the likely root cause?

### 30. S3 Eventually Consistent Reads
Your image upload service writes to S3 and immediately redirects users to view their upload. 2% of users see a 404 for 1-2 seconds before the image appears. Your code looks correct. What S3 consistency model behavior is causing this in 2026, and what's your solution?

### 31. Database Connection Pool Exhaustion
Your Node.js service has 20 instances, each with a Postgres connection pool of 10. Database supports 500 connections. Yet you see "connection pool exhausted" errors during peak traffic at 150 RPS. What's likely misconfigured?

### 32. JWT Token Revocation at Scale
Your API uses JWTs with 24-hour expiration. When a user logs out or changes password, you need immediate token invalidation across 100+ service instances. Checking a blacklist on every request kills performance. What's a scalable revocation strategy?

### 33. Elasticsearch Shard Rebalancing Storm
After adding 3 nodes to your 20-node Elasticsearch cluster, query latency spiked from 50ms to 8 seconds for 6 hours. CPU on all nodes was high. No other changes were made. What happened, and how do you prevent it during scaling?

### 34. B2B SaaS Tenant Isolation
Your B2B SaaS platform uses shared tables with a `tenant_id` column. A new Fortune 500 customer demands data isolation for compliance. Row-level security exists, but they want stronger guarantees. What are your options without migrating to per-tenant databases?

### 35. Container Image Vulnerability Crisis
A critical CVE is announced in a base image your 200 microservices depend on. You need to rebuild and redeploy all services within 4 hours. Your current CI/CD takes 45 minutes per service. How do you handle this organizationally and technically?

### 36. API Gateway Timeout Paradox
Your API Gateway has a 30-second timeout, and your backend service has a 25-second timeout. Yet you see requests hitting your backend for 2+ minutes during incidents. How is this possible, and what fixes it?

### 37. Distributed Tracing Sampling Bias
Your observability platform samples 1% of traces to reduce costs. During an incident, you can't reproduce the issue because affected traces weren't sampled. How do you balance cost with debuggability?

### 38. Postgres Autovacuum Blocking
Your high-write Postgres table hasn't been autovacuumed in 3 days because of long-running transactions. Table bloat is causing query slowdowns. You can't kill the long transactions. What's your immediate and long-term strategy?

### 39. DynamoDB Hot Partition
Your DynamoDB table uses `user_id` as the partition key. One celebrity user with 50M followers causes read throttling affecting all users. Increasing provisioned capacity doesn't help. What's the architectural fix?

### 40. Message Queue Poison Pills
Your Kafka consumer keeps crashing and restarting because one malformed message (out of millions) causes deserialization failure. The consumer is part of a consumer group. How do you handle poison pills without data loss or manual intervention?

### 41. gRPC Load Balancing Failure
You deployed gRPC services behind an L7 load balancer. All traffic goes to only 2 out of 20 backend instances. HTTP/REST endpoints load balance fine. What's different about gRPC that breaks standard load balancing?

### 42. Database Migration Zero-Downtime
You need to add a `NOT NULL` column with a default value to a 500M-row table in production Postgres. A naive `ALTER TABLE` will lock the table for hours. What's your zero-downtime migration strategy?

### 43. Redis Cluster Split-Brain
Your Redis cluster experienced a network partition. After recovery, you discover data inconsistencies—some keys have different values on different nodes. How did this happen in Redis Cluster, and how do you prevent it?

### 44. Serverless Cold Start Chain
Your AWS Lambda function (1GB memory, Java) calls another Lambda which calls another. End-to-end P99 latency is 8 seconds despite each function running in 200ms. What's causing the latency, and how do you optimize?

### 45. Log Aggregation Cost Explosion
Your log aggregation bill jumped from $5K/month to $80K/month after a new service launch. The service only handles 1000 RPS. What logging anti-patterns cause cost explosions, and how do you fix them?

### 46. Feature Flag Consistency
Your feature flag service allows gradual rollouts. During a 50% rollout, the same user sees different behavior on each request (new feature, old feature, new, old). Sticky sessions aren't an option. What's wrong with the flag evaluation?

### 47. Circuit Breaker Cascading Failure
Your circuit breaker trips when Error Service fails, protecting your app. But now all requests immediately fail with "Circuit Open" errors, creating the same user impact as if the circuit breaker didn't exist. How do you implement graceful degradation?

### 48. Multi-Tenancy Query Performance
Your SaaS platform has 10,000 tenants in shared tables. Queries for small tenants (100 rows) take milliseconds, but queries for your largest tenant (50M rows) time out despite proper indexes on `tenant_id`. What query patterns are you missing?

### 49. Webhook Delivery Guarantees
You send webhooks to customer endpoints. Some customers' servers are down for hours. How do you implement reliable webhook delivery with retry logic, dead letter queues, and delivery guarantees without overwhelming your infrastructure?

### 50. Time-Series Data Compaction
Your time-series database stores IoT sensor data (1M writes/sec). Raw data must be kept for 7 days, then aggregated to hourly averages for 1 year. How do you architect automatic data compaction without impacting write throughput?

### 51. OAuth Token Theft Prevention
Your mobile app stores OAuth refresh tokens. Security audit reveals that if a device is compromised, the attacker has permanent API access. You can't force all users to re-login. What's your token rotation and binding strategy?

### 52. Database Read Replica Lag
Your read replicas lag 30 seconds behind the primary during peak hours despite 10Gbps network. Replication is synchronous. The primary isn't CPU/memory bound. What causes replication lag, and how do you diagnose it?

### 53. API Versioning Migration
You need to deprecate API v1 (used by 500 enterprise clients) and force migration to v2 within 6 months. v1 and v2 have incompatible breaking changes. What's your technical and communication strategy?

### 54. Real-Time Leaderboard at Scale
You're building a real-time gaming leaderboard for 10M concurrent players where ranks update every second. Redis sorted sets work for 100K users but don't scale. What architecture handles 10M+ players with sub-second rank updates?

### 55. Dependency Version Lock-in
Your monorepo has 50 services. Service A depends on library X v1.0, Service B requires X v2.0 with breaking changes. Both run in the same cluster. How do you handle conflicting dependency versions?

### 56. Blockchain State Synchronization
You're building a cryptocurrency wallet that needs to validate transactions. Full node sync takes 2 weeks to download 800GB blockchain. Users need wallet functionality immediately. What's your state sync strategy?

### 57. ML Model Serving Latency
Your ML model inference API must respond in <100ms. Model loading takes 8 seconds. Cold starts kill user experience. You're on Kubernetes, not serverless. How do you architect this?

### 58. Multi-Cloud DNS Failover
Your application runs on AWS and GCP for redundancy. When AWS region fails, DNS should route to GCP. But DNS TTL means users can't access your app for 5 minutes. How do you achieve faster failover?

### 59. Object Storage Listing Performance
Your application lists S3 bucket contents with 10M objects. The `ListObjects` API call takes 40 seconds and times out. You need to paginate through all objects efficiently. What's your strategy?

### 60. Distributed Lock Correctness
You use Redis `SETNX` for distributed locking across services. During network partitions, two services acquire the same lock. What's wrong with your lock implementation, and how do you make it safe?

### 61. Background Job Priority Inversion
Your background job queue processes low-priority email jobs before high-priority payment processing jobs. Both have separate queues with priorities set. What's causing priority inversion in your job processor?

### 62. Schema Registry Version Conflicts
Your Kafka consumers use Avro with a schema registry. A producer deploys an incompatible schema change. Old consumers start failing. How do you enforce schema compatibility and handle breaking changes?

### 63. Database Query Plan Regression
A query that ran in 50ms for 2 years suddenly takes 30 seconds after a routine stats update. `EXPLAIN ANALYZE` shows a different query plan. Indexes exist. How do you fix it immediately and prevent recurrence?

### 64. Session Store Scalability
Your session store (Redis) contains 100M sessions. Session validation on every request adds 5ms latency. You need to reduce this without changing the authentication flow. What's your optimization approach?

### 65. File Upload Progress Tracking
Users upload 5GB video files. You need to show real-time progress and support resumable uploads. Direct S3 uploads work, but you lose upload progress visibility. How do you architect this?

### 66. Idempotency Key Design
Your payment API uses idempotency keys to prevent duplicate charges. A client retries the same request with the same key after 7 days. Your keys expire after 24 hours. What's your idempotency strategy for long retry windows?

### 67. Message Ordering in Distributed Systems
Your distributed order processing system must process orders in sequence per customer but can process different customers in parallel. Kafka partitioning by `customer_id` works until a customer places 1000 orders/second. What's your approach?

### 68. Database Backup Verification
Your automated database backups run daily, but you discover backups have been corrupted for 3 months when you try to restore. How do you implement backup verification without impacting production?

### 69. API Rate Limit Coordination
Your API has global rate limits (10K req/sec) enforced across 50 instances. Each instance tracks its own counter. Users exceed limits by 40% because instances don't coordinate. How do you implement accurate distributed rate limiting?

### 70. Canary Deployment Rollback
You deployed a canary (10% traffic) of a new service version. Canary error rate is 0.5% vs 0.1% baseline. Is this statistically significant enough to rollback, or just noise? How do you make this decision programmatically?

---

## Production System Design Questions (Senior Backend Engineer Focus)

### 71. Design a URL Shortener at Scale
Design a URL shortener service (like bit.ly) that handles 10M new URLs per day and 100M redirects per day. Requirements: custom aliases, expiration, analytics (click counts, geography, referrers), and 99.99% uptime. What's your complete system architecture including database schema, caching strategy, rate limiting, and how do you generate short codes to avoid collisions?

### 72. Design a Distributed Task Scheduler
Design a distributed cron-like task scheduler that executes millions of tasks daily across a cluster. Requirements: tasks can be scheduled for specific times or recurring intervals, guaranteed execution (at-least-once), observability (success/failure tracking), retry logic with exponential backoff, and the ability to add/remove worker nodes dynamically. How do you prevent duplicate execution and ensure fair distribution?

### 73. Design a Real-Time Notification System
Design a multi-channel notification system (email, SMS, push, in-app) for 100M users. Requirements: priority levels, user preferences, delivery guarantees, deduplication, rate limiting per channel, template management, and delivery status tracking. How do you handle email provider rate limits and ensure notifications aren't lost during provider outages?

### 74. Design a Rate Limiting System
Design a distributed rate limiting system that enforces limits like "100 requests per user per minute" and "10,000 requests per API key per hour" across 500 service instances. Requirements: multiple limit tiers (user, IP, API key, endpoint), burst allowances, accurate counting despite network partitions, and <1ms latency overhead. What algorithms and data structures do you use?

### 75. Design a Distributed File Storage System
Design a file storage system like Dropbox that handles 1PB of data across millions of users. Requirements: file versioning, deduplication, sharing with permissions, sync across devices, and 99.9% durability. How do you handle large file uploads (10GB+), implement delta sync for efficient bandwidth usage, and ensure conflict resolution when the same file is edited offline on multiple devices?

### 76. Design a Search Autocomplete System
Design a search autocomplete/typeahead system for an e-commerce site with 50M products. Requirements: sub-100ms latency, personalized suggestions based on user history, trending searches, typo tolerance, and support for 20+ languages. How do you handle the data pipeline from product updates to search index, and how do you rank suggestions?

### 77. Design an Analytics Pipeline
Design a real-time analytics system that processes 500K events/second from web and mobile apps (page views, clicks, purchases). Requirements: sub-second dashboard updates, historical queries for 2 years, custom event properties, funnel analysis, and user cohorts. What's your data ingestion, storage, and query architecture?

### 78. Design a Distributed Counter System
Design a distributed counter service for social media "likes" that handles 1M writes/second and 10M reads/second globally. Requirements: eventual consistency is acceptable with <1 second propagation, read-after-write consistency for the user who liked, and accurate total counts. How do you prevent count inflation from duplicate clicks?

### 79. Design a Content Delivery Network (CDN)
Design a CDN that serves static assets (images, CSS, JS) globally with <50ms P95 latency. Requirements: 100TB of content, 1M requests/second, cache invalidation, SSL/TLS, gzip compression, and image resizing on-the-fly. How do you handle cache consistency, origin shield, and DDoS protection?

### 80. Design a Payment Processing System
Design a payment gateway that processes credit card transactions with PCI DSS compliance. Requirements: support multiple payment processors (Stripe, PayPal), idempotency, webhook notifications, refunds, fraud detection, and reconciliation with bank statements. How do you ensure exactly-once payment processing and handle processor failures?

### 81. Design a Live Comment System
Design a live comment system for streaming videos (like YouTube Live) where 100K concurrent viewers can post comments appearing in real-time for all viewers. Requirements: rate limiting, moderation (automatic profanity filtering, manual review queue), comment reactions, and threaded replies. How do you handle scaling and prevent spam?

### 82. Design a Ride-Sharing Dispatch System
Design the core matching engine for a ride-sharing app (like Uber) that matches drivers to riders. Requirements: match within 30 seconds, consider distance, driver rating, surge pricing, multiple ride types, and driver acceptance/rejection. How do you handle high-density areas with 1000+ drivers and riders, and what happens when a driver rejects?

### 83. Design a Newsfeed/Timeline System
Design a social media newsfeed (like Facebook/Twitter) for 500M users. Requirements: personalized feeds using ML ranking, real-time updates when friends post, pagination, and support for multiple content types (text, images, videos). How do you generate feeds on-demand vs pre-compute, and handle users who follow 10K+ accounts?

### 84. Design a Distributed Session Store
Design a session management system for a web application with 1M concurrent users across 200 server instances. Requirements: <5ms session lookup latency, automatic expiration, session sharing across subdomains, and ability to invalidate all sessions for a user. How do you handle session migration during deployments and prevent session fixation attacks?

### 85. Design a Metrics Collection System
Design a metrics collection system (like Prometheus/Datadog) that ingests time-series metrics from 10K services. Requirements: 1M metrics/second write throughput, retention for 90 days, alerting when thresholds are breached, and complex queries (aggregations, rate calculations). How do you handle cardinality explosion and storage efficiency?

### 86. Design a Distributed Cache
Design a distributed caching layer (like Memcached/Redis) that serves 500K requests/second with <1ms P99 latency. Requirements: consistent hashing for data distribution, replication for availability, cache warming strategies, and LRU eviction. How do you handle cache stampede, hot keys, and node failures without data loss?

### 87. Design a Job Queue System
Design a distributed job queue (like Sidekiq/Celery) that processes 100K background jobs per minute. Requirements: job priorities, delayed jobs, scheduled jobs, retry with exponential backoff, dead letter queue, and job result storage. How do you ensure at-least-once processing, prevent duplicate execution, and handle worker crashes mid-job?

### 88. Design a Subscription/Billing System
Design a subscription billing system for a SaaS platform with tiered pricing (free, pro, enterprise). Requirements: monthly/annual billing cycles, proration for upgrades/downgrades, usage-based billing, invoice generation, payment retries for failed charges, and dunning management. How do you handle timezone differences and ensure revenue recognition compliance?

### 89. Design a Logging Infrastructure
Design a centralized logging system that collects logs from 1000 microservices (10GB logs/day per service). Requirements: full-text search, structured log parsing, log retention policies (7 days hot, 90 days warm, 1 year cold), alerting on error patterns, and log sampling to reduce costs. What's your architecture for ingestion, storage, and querying?

### 90. Design a Geospatial Search System
Design a location-based search service (like "find restaurants within 5km") for 1M points of interest. Requirements: sub-100ms query latency, real-time updates when businesses open/close, filter by category, and ranked results by distance and rating. What indexing strategy and database do you use for efficient radius queries?

### 91. Design an API Gateway
Design an API Gateway that sits in front of 100 microservices. Requirements: authentication/authorization, rate limiting, request/response transformation, circuit breaking, retry logic, request routing based on headers/paths, and observability (logging, metrics, tracing). How do you prevent the gateway from becoming a single point of failure?

### 92. Design a Feature Flag System
Design a feature flag service that controls feature rollouts across 500 services. Requirements: percentage-based rollouts, user targeting (by ID, attributes, segments), A/B test support, flag dependency management, and real-time flag updates without service restarts. How do you ensure flag evaluation consistency for the same user across multiple requests?

### 93. Design a Data Export System
Design a system that allows users to export their entire account data (GDPR compliance) as a downloadable archive. Requirements: support for accounts with 100GB+ data, background processing, email notification when ready, temporary download links, and audit logging. How do you handle exports for deleted/deactivated accounts and rate limit export requests to prevent abuse?

### 94. Design a Recommendation Engine
Design a recommendation system for an e-commerce platform with 10M products and 50M users. Requirements: personalized recommendations based on browsing history, purchases, and collaborative filtering, real-time updates when user behavior changes, A/B testing different algorithms, and cold start problem handling for new users/products. What's your model training pipeline and serving architecture?

### 95. Design a Webhook Infrastructure
Design a webhook delivery system that sends HTTP callbacks to customer endpoints for 10K customers. Requirements: guaranteed delivery with retries, exponential backoff, configurable retry policies, signature verification, rate limiting per customer, and webhook event replay. How do you handle customers with slow/failing endpoints and prevent one bad customer from affecting others?

### 96. Design a Multi-Tenant Database Architecture
Design a database architecture for a B2B SaaS with 5K customers ranging from 10 users to 100K users. Requirements: data isolation, per-tenant backups, per-tenant schema customization, and cost allocation by tenant. When do you use shared database/shared schema vs shared database/separate schema vs separate database per tenant?

### 97. Design a Document Collaboration System
Design a Google Docs-like collaborative document editing system. Requirements: 100 concurrent editors per document, real-time sync with <100ms latency, conflict resolution, version history, commenting, and offline editing with sync when online. What's your operational transformation or CRDT approach, and how do you handle network partitions?

### 98. Design an Email Service
Design a transactional email service (like SendGrid) that sends 1B emails/month. Requirements: template management, personalization, bounce/complaint handling, unsubscribe management, open/click tracking, and SPF/DKIM/DMARC compliance. How do you warm up IP addresses, handle reputation management, and ensure deliverability?

### 99. Design a Backup and Disaster Recovery System
Design a backup and DR strategy for a microservices architecture with 50 services and 20 databases (Postgres, MongoDB, Redis). Requirements: RPO of 15 minutes, RTO of 1 hour, cross-region replication, automated failover, backup verification, and point-in-time recovery. How do you orchestrate failover across all services and test DR regularly?

### 100. Design a Tenant Provisioning System
Design an automated tenant onboarding system for a B2B SaaS. Requirements: provision infrastructure (database, cache, storage), configure SSO/SAML, set up custom domains with SSL certificates, initialize demo data, and send welcome emails—all within 5 minutes. How do you handle provisioning failures, rollback partial setups, and scale to onboard 100+ tenants daily?

---

## Question Categories

### Foundational Questions (1-20)
Original case studies covering cache consistency, pagination, compression, real-time indicators, query optimization, deployment strategies, and scaling patterns.

### Advanced Scenario Questions (21-70)

**Infrastructure & Scalability (21-25, 31, 33, 38, 44, 52, 58)**: Multi-region failover, WebSocket scaling, Kubernetes pod eviction, connection pooling, serverless optimization, DNS failover.

**Data Consistency & Transactions (24, 27, 43, 48, 63, 68)**: Event sourcing, microservices consistency, Redis cluster split-brain, query plan regression, backup verification.

**Performance Optimization (23, 28, 29, 30, 36, 37, 48, 57, 64)**: GraphQL N+1, CDN cache poisoning, gRPC deadlines, S3 consistency, API gateway timeouts, distributed tracing, session store optimization, ML model serving.

**Security & Compliance (34, 51)**: B2B tenant isolation, OAuth token theft prevention.

**API Design & Integration (12, 32, 46, 49, 53, 66, 69)**: JWT revocation, feature flags, webhook delivery, idempotency keys, API versioning, rate limiting.

**Real-Time & Streaming Systems (17, 22, 54, 67)**: ClickHouse latency, WebSocket storms, real-time leaderboards, message ordering.

**Cloud & Distributed Systems (35, 39, 40, 41, 55, 59, 60)**: Container vulnerabilities, DynamoDB hot partitions, Kafka poison pills, gRPC load balancing, dependency conflicts, distributed locks.

**Database Engineering (8, 31, 38, 42, 45, 52, 59, 63, 65)**: Index optimization, connection pools, autovacuum blocking, zero-downtime migrations, time-series compaction, replica lag, upload tracking.

**Observability & Operations (25, 26, 37, 47, 50, 61, 62, 70)**: Log cost explosion, feature flag consistency, tracing sampling, circuit breakers, canary deployments, job priority, schema compatibility.

### Production System Design Questions (71-100)

**Core Infrastructure (86, 91, 99)**: Distributed cache design, API gateway architecture, backup and disaster recovery.

**Data & Storage Systems (75, 90, 96)**: Distributed file storage, geospatial search, multi-tenant database architecture.

**Real-Time & Collaboration (81, 83, 97)**: Live comments, social media newsfeed, document collaboration.

**Background Processing (72, 87)**: Distributed task scheduler, job queue system.

**Communication & Notifications (73, 98, 95)**: Multi-channel notifications, email service, webhook infrastructure.

**Analytics & Observability (77, 85, 89)**: Analytics pipeline, metrics collection, logging infrastructure.

**E-commerce & Marketplace (71, 76, 82, 94)**: URL shortener, search autocomplete, ride-sharing dispatch, recommendation engine.

**Financial Systems (80, 88)**: Payment processing, subscription billing.

**Platform & Developer Tools (74, 84, 92, 93)**: Rate limiting, distributed session store, feature flags, data export.

**Distributed Coordination (78, 79, 100)**: Distributed counters, CDN design, tenant provisioning.

---

**Total Questions**: 100 (20 foundational + 50 advanced scenarios + 30 system design)

**Difficulty Levels:**
- **Questions 1-20**: Foundational - Good for mid to senior engineers
- **Questions 21-70**: Advanced Scenarios - Senior engineers and architects
- **Questions 71-100**: System Design - Senior backend engineers, staff engineers, and CTOs

*These questions reflect real-world challenges faced in modern distributed systems, cloud-native architectures, high-scale applications, and production system design as of 2026.*
