# Complete Software Engineering Case Studies & Questions - 2026

## Original Case Studies from Document

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
