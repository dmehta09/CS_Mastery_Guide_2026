# Getting Started with System Design Mastery

Welcome! This guide will help you navigate the 15 system design projects and master distributed systems, scalability, and architecture patterns.

## Quick Start

### Step 1: Assess Your Level

**Beginner (0-2 years experience)**:
- Start with: Projects 01-03 (Foundation tier)
- Focus on: Understanding basic concepts, trade-offs
- Timeline: 8-12 weeks

**Intermediate (2-4 years experience)**:
- Start with: Projects 04-06 (Data & Architecture tier)
- Focus on: Database selection, architectural patterns
- Timeline: 10-14 weeks

**Advanced (4+ years experience)**:
- Start with: Any project, focus on depth
- Focus on: Production considerations, edge cases
- Timeline: 12-16 weeks

### Step 2: Study Methodology

For each project:

```
Day 1-2: Read & Understand
  ├─ Read problem statement
  ├─ Understand requirements
  ├─ Review capacity estimation
  └─ Study architecture diagrams

Day 3-5: Design Exercise
  ├─ Close the document
  ├─ Try to design system yourself
  ├─ Draw your own architecture
  └─ Make your own trade-off decisions

Day 6-7: Compare & Learn
  ├─ Compare your design with provided solution
  ├─ Understand differences
  ├─ Learn new patterns you missed
  └─ Document learnings

Day 8-10: Deep Dive (Optional)
  ├─ Implement a simplified version
  ├─ Test at scale
  └─ Measure actual vs estimated capacity
```

## Learning Paths

### Path 1: Interview Preparation (4 weeks intensive)

**Week 1: Foundation**
- Project 01: CDN (caching, replication)
- Project 02: Load Balancer (traffic distribution)
- Practice: Design questions on scaling

**Week 2: Data**
- Project 04: E-Commerce (database selection)
- Project 05: Monitoring (time-series)
- Practice: Design questions on data storage

**Week 3: Reliability**
- Project 08: Payment Processing (distributed transactions)
- Project 09: API Gateway (resilience patterns)
- Practice: Design questions on fault tolerance

**Week 4: Modern**
- Project 13: APM (observability)
- Project 15: Vector DB (AI/ML)
- Practice: Mock interviews

### Path 2: Production Engineering (12 weeks comprehensive)

**Weeks 1-3: Foundations**
- All concepts from Tier 1 (Projects 1-3)
- Build simplified versions
- Load test and measure

**Weeks 4-6: Data & Architecture**
- Projects 4-6 in depth
- Implement CQRS pattern
- Build event sourcing system

**Weeks 7-9: Advanced Patterns**
- Projects 7-10
- Focus on distributed systems
- Handle failure scenarios

**Weeks 10-12: Modern Infrastructure**
- Projects 11-15
- Observability and monitoring
- AI/ML infrastructure

## Study Techniques

### 1. Active Recall

Don't just read - test yourself:

```
Question: How does consistent hashing work?
Your answer: [Write it out before looking]
Verify: Check against Project 02 (Load Balancer)
```

### 2. Spaced Repetition

Review concepts at increasing intervals:

```
Day 1: Learn concept
Day 2: Review
Day 5: Review
Day 14: Review
Day 30: Review
```

### 3. Teach to Learn

Explain concepts to others:

```
- Write blog posts
- Create presentations
- Explain to rubber duck
- Answer questions on forums
```

### 4. Build to Understand

Implement simplified versions:

```
Example: After studying CDN (Project 01)
→ Build: Simple cache server with Redis
→ Measure: Cache hit ratio, latency
→ Compare: Your results vs estimates
```

## Interview Preparation

### Common Questions by Project

**Scaling Questions:**
- "Design Twitter" → Projects 01, 02, 09
- "Design Instagram" → Projects 01, 06
- "Design Uber" → Projects 02, 10

**Data Questions:**
- "Design Amazon" → Project 04
- "Design Analytics System" → Project 05
- "Design Elasticsearch" → Project 06

**Reliability Questions:**
- "Design Payment System" → Project 08
- "Design Booking System" → Project 08, 09
- "Design Stock Trading" → Project 12

### Interview Framework

Use this framework for ANY system design question:

```
1. Requirements (5 minutes)
   ├─ Functional: What features?
   ├─ Non-functional: Scale? Latency? Availability?
   └─ Out of scope: What to exclude?

2. Capacity Estimation (5 minutes)
   ├─ Users, requests, storage
   ├─ Calculate QPS, bandwidth
   └─ Estimate servers needed

3. High-Level Design (10 minutes)
   ├─ Draw boxes and arrows
   ├─ Identify major components
   └─ Data flow

4. Deep Dive (20 minutes)
   ├─ Pick 2-3 components
   ├─ Explain algorithms/data structures
   ├─ Discuss trade-offs
   └─ Handle edge cases

5. Wrap-up (5 minutes)
   ├─ Bottlenecks?
   ├─ Improvements?
   └─ Monitoring?
```

## Capacity Estimation Cheat Sheet

### Back-of-Envelope Calculations

```
1 million = 10^6
1 billion = 10^9

1 KB = 10^3 bytes
1 MB = 10^6 bytes
1 GB = 10^9 bytes
1 TB = 10^12 bytes

1 day = 86,400 seconds ≈ 10^5 seconds
1 year ≈ 365 days ≈ 10^7 seconds

Examples:
- 100M requests/day = 100M / 10^5 = 1K RPS
- 1K RPS × 10 KB = 10 MB/s = 80 Mbps
```

### Latency Numbers (2026)

```
L1 cache reference:              0.5 ns
L2 cache reference:              7   ns
Main memory reference:           100 ns
SSD random read:                 16  μs
Network within datacenter:       0.5 ms
SSD sequential read (1 MB):      1   ms
HDD seek:                        10  ms
Network cross-continent:         150 ms
```

### Availability Math

```
99.9%   = 43 minutes downtime/month
99.95%  = 22 minutes downtime/month
99.99%  = 4.3 minutes downtime/month
99.999% = 26 seconds downtime/month
```

## Trade-off Checklists

### When to Use...

**Caching:**
- ✅ Read-heavy workload (read:write > 10:1)
- ✅ Frequently accessed data
- ✅ Can tolerate stale data
- ❌ Highly dynamic data
- ❌ Cache stampede risk

**Sharding:**
- ✅ Data doesn't fit on one machine
- ✅ High write throughput
- ✅ Can partition data logically
- ❌ Need cross-shard transactions
- ❌ Hotspots (uneven distribution)

**Async Processing:**
- ✅ Non-critical path operations
- ✅ Time-consuming tasks
- ✅ Need to decouple services
- ❌ Need immediate consistency
- ❌ User needs instant feedback

**Microservices:**
- ✅ Large team (>20 engineers)
- ✅ Different scaling needs per service
- ✅ Independent deployment
- ❌ Small team (<5 engineers)
- ❌ Simple domain

## Mental Models

### 1. CAP Theorem Triangle

```
        Consistency
             /\
            /  \
           /    \
          /      \
    Availability - Partition Tolerance

Choose 2 out of 3 during network partition
```

### 2. Consistency Spectrum

```
Linearizable → Sequential → Causal → Eventual
(Strongest)                          (Weakest)

   Slower                              Faster
   Lower availability                  Higher availability
```

### 3. Database Selection Matrix

```
                Structured    Semi-structured   Unstructured
Transactional   PostgreSQL    MongoDB          N/A
Analytical      Redshift      Elasticsearch    HDFS
Time-series     TimescaleDB   InfluxDB        N/A
Graph           N/A           Neo4j           N/A
```

## Common Pitfalls to Avoid

### 1. Over-engineering

```
❌ Bad: "We'll use Kafka, Kubernetes, microservices for 1000 users"
✅ Good: "Start with monolith, scale when needed"
```

### 2. Ignoring Trade-offs

```
❌ Bad: "We'll use caching" (without discussing invalidation)
✅ Good: "We'll use caching with 1-hour TTL. Trade-off: stale data vs performance"
```

### 3. Skipping Capacity Estimation

```
❌ Bad: "We'll use 3 servers"
✅ Good: "1K RPS × 10KB = 10MB/s. At 50% capacity, need 2 servers + 1 for failover = 3"
```

### 4. No Failure Handling

```
❌ Bad: "Service A calls Service B"
✅ Good: "Service A calls B with retry + timeout. If B is down, use cached data"
```

## Resources

### Books (Must-Read)

1. **"Designing Data-Intensive Applications"** by Martin Kleppmann
   - The Bible of system design
   - Read: Chapters 1-3, 5, 7-9

2. **"System Design Interview Vol 1 & 2"** by Alex Xu
   - Great for interview prep
   - 15 detailed designs

3. **"Database Internals"** by Alex Petrov
   - Deep dive into database architecture
   - Read if interested in storage engines

### Online Resources

- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [High Scalability Blog](http://highscalability.com/)
- [Papers We Love](https://paperswelove.org/)

### Tech Blogs

- [Netflix Tech Blog](https://netflixtechblog.com/)
- [Uber Engineering](https://eng.uber.com/)
- [Airbnb Engineering](https://medium.com/airbnb-engineering)
- [Discord Engineering](https://discord.com/category/engineering)

## Practice Schedule

### Daily (30-60 minutes)

```
Monday: Read new project
Tuesday: Design exercise
Wednesday: Compare with solution
Thursday: Deep dive on one component
Friday: Review and document learnings
Weekend: Build simplified version (optional)
```

### Weekly Goals

```
Week 1-4:   Complete 2 foundation projects
Week 5-8:   Complete 2 intermediate projects
Week 9-12:  Complete 2 advanced projects
Week 13-16: Review all + mock interviews
```

## Mock Interview Tips

### Preparation

```
1. Choose random project
2. Set 45-minute timer
3. Design from scratch (don't look at notes)
4. Draw diagrams on whiteboard/paper
5. Explain out loud (record yourself)
6. Compare with solution after
```

### During Interview

```
DO:
✅ Ask clarifying questions
✅ Start with high-level design
✅ Explain your thought process
✅ Discuss trade-offs
✅ Admit when you don't know

DON'T:
❌ Jump into implementation details
❌ Assume requirements
❌ Ignore constraints
❌ Design in silence
❌ Refuse to change approach
```

## Progress Tracking

Create a checklist:

```markdown
## Foundation (Weeks 1-3)
- [ ] Project 01: CDN
- [ ] Project 02: Load Balancer
- [ ] Project 03: Rate Limiter

## Data & Architecture (Weeks 4-6)
- [ ] Project 04: E-Commerce
- [ ] Project 05: Monitoring
- [ ] Project 06: Search Engine

## APIs & Reliability (Weeks 7-9)
- [ ] Project 07: GraphQL Gateway
- [ ] Project 08: Payment Processor
- [ ] Project 09: API Gateway

## Advanced (Weeks 10-12)
- [ ] Project 10: Multi-Region System
- [ ] Project 11: Zero-Trust Auth
- [ ] Project 12: Trading Platform

## Modern (Weeks 13-16)
- [ ] Project 13: APM Platform
- [ ] Project 14: Streaming Analytics
- [ ] Project 15: Vector DB & RAG
```

## Next Steps

1. **Choose your starting project** based on your experience level
2. **Set a schedule** (e.g., "Complete Project 01 by Friday")
3. **Find a study partner** or join online communities
4. **Document your learnings** in a blog or notebook
5. **Practice explaining** designs to others

---

**Ready to start?**

Navigate to your first project:
- Beginner: [Project 01: CDN](./01-global-cdn/README.md)
- Intermediate: [Project 04: E-Commerce](./04-polyglot-ecommerce/README.md)
- Advanced: [Project 10: Multi-Region](./10-multi-region-system/README.md)

**Good luck on your system design mastery journey!** 🚀

Remember: The goal is not just to memorize designs, but to understand principles and trade-offs so you can design ANY system from first principles.
