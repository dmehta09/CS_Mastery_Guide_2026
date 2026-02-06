# Production Case Studies 2026

Comprehensive Q&A case studies for **Backend** and **Cloud** systems based on real engineering challenges from Netflix, Uber, Discord, Stripe, Spotify, Airbnb, LinkedIn, Pinterest, and other big tech companies.

---

## Planning Documents

**Read these first before creating case studies:**

| Document | Purpose |
|----------|---------|
| [MASTER_PLAN.md](./MASTER_PLAN.md) | Project overview, strategy, and complete file structure |
| [STYLE_GUIDE.md](./STYLE_GUIDE.md) | **Critical**: Writing guidelines - focus on understanding, not code |
| [FILE_STRUCTURE.md](./FILE_STRUCTURE.md) | Complete list of all files to create with priorities |
| [PROGRESS.md](./PROGRESS.md) | Track completion status |
| [RESOURCES_FILE_TEMPLATE.md](./RESOURCES_FILE_TEMPLATE.md) | Template for creating resources files for each case study |

---

## Key Writing Rules (Summary)

**Focus: Crisp, Concise, Well-Researched Understanding**
- **No code implementation** - Focus on concepts, architecture, and decision-making
- **Well-researched** - Based on actual engineering blogs, papers, and conference talks (2024-2026)
- **Crisp & Concise** - Get to the point quickly, no fluff
- **Complete resources** - All research sources documented in separate `[topic]_RESOURCES.md` file
- **Minimum 3-5 case studies per file** - Each case study file must contain 3-5 case studies covering different aspects of the topic

Every case study **MUST** include:

### 1. Layman Explanation First

```markdown
❌ "Implement token bucket rate limiting with Redis INCR"

✅ "Imagine a water pipe serving 1000 houses. One house leaves all
   faucets running full blast. Everyone else gets a trickle.
   Rate limiting is like putting a flow limiter on each connection.

   Technical solution: Token bucket algorithm with Redis..."
```

### 2. Conceptual Understanding (No Code Implementation)

**Focus on explaining concepts, architecture, and decisions - NOT code.**

```markdown
❌ BAD: Showing full code implementation
[Long code block with implementation details]

✅ GOOD: Explaining the concept and approach
**Approach**: Stripe uses idempotency keys stored in Redis to prevent duplicate charges.

**How it works conceptually**:
1. Client sends request with unique idempotency_key (UUID)
2. Server checks Redis: "Have we seen this key before?"
3. If yes → Return cached result (prevents duplicate charge)
4. If no → Process payment, cache result for 24 hours

**Why this design**:
- Network can fail AFTER payment succeeds but BEFORE response reaches client
- Client retries with same key → Gets cached result, no double charge
- 24-hour TTL: Long enough for retries, short enough to prevent cache bloat

**Architecture decision**: Redis chosen for sub-millisecond lookups and atomic operations
```

### 3. Complete Research Resources

Each case study must have a corresponding `[topic]_RESOURCES.md` file with:
- Engineering blog posts (with URLs and dates)
- Conference talks (with links)
- Technical papers (with citations)
- Documentation references
- Video presentations

**See [STYLE_GUIDE.md](./STYLE_GUIDE.md) for complete guidelines.**

---

## Quick Stats

| Domain | Categories | Case Study Files | Resources Files | Total Files | Case Studies |
|--------|-----------|------------------|-----------------|-------------|--------------|
| [Backend](#backend-production) | 8 | 46 | 46 | 92 | 140-230 |
| [Cloud](#cloud-infrastructure) | 8 | 47 | 47 | 94 | 140-235 |
| **Total** | **16** | **93** | **93** | **186** | **280-465** |

**Note**: Each case study has a corresponding `[topic]_RESOURCES.md` file with complete research sources, references, and links.

---

## Backend Production

Real-world backend engineering challenges at scale.

| Category | Key Topics | Companies |
|----------|-----------|-----------|
| [01_Distributed_Systems](./01_BACKEND_PRODUCTION/01_Distributed_Systems/) | Cache, Sagas, Circuit breakers | Netflix, Uber |
| [02_Database_Architecture](./01_BACKEND_PRODUCTION/02_Database_Architecture/) | Sharding, Replication, Migrations | Discord, Pinterest |
| [03_Real_Time_Systems](./01_BACKEND_PRODUCTION/03_Real_Time_Systems/) | WebSockets, Typing, Read receipts | Discord, WhatsApp |
| [04_API_Design](./01_BACKEND_PRODUCTION/04_API_Design/) | Idempotency, Rate limiting, GraphQL | Stripe, Cloudflare |
| [05_Event_Driven_Architecture](./01_BACKEND_PRODUCTION/05_Event_Driven_Architecture/) | Kafka, Event sourcing, CQRS | Uber, LinkedIn |
| [06_Search_And_Discovery](./01_BACKEND_PRODUCTION/06_Search_And_Discovery/) | Elasticsearch, Embeddings, Bloom filters | Airbnb, Pinterest |
| [07_Performance_Optimization](./01_BACKEND_PRODUCTION/07_Performance_Optimization/) | Caching, Load balancing, Async | Netflix, Uber |
| [08_Data_Processing](./01_BACKEND_PRODUCTION/08_Data_Processing/) | Batch/Stream, ClickHouse, Data lakes | Cloudflare, Uber |

---

## Cloud Infrastructure

Production-grade cloud architecture and DevOps.

| Category | Key Topics | Companies |
|----------|-----------|-----------|
| [01_Container_Orchestration](./03_CLOUD_INFRASTRUCTURE/01_Container_Orchestration/) | Kubernetes, Service mesh, Scaling | Spotify, Uber |
| [02_CI_CD_Deployment](./03_CLOUD_INFRASTRUCTURE/02_CI_CD_Deployment/) | Zero-downtime, GitOps, Rollbacks | Netflix, Uber |
| [03_Observability](./03_CLOUD_INFRASTRUCTURE/03_Observability/) | Prometheus, Tracing, SLIs/SLOs | Uber (M3), Google |
| [04_Multi_Cloud_And_Resilience](./03_CLOUD_INFRASTRUCTURE/04_Multi_Cloud_And_Resilience/) | Multi-cloud, Disaster recovery | Uber, Netflix |
| [05_Security](./03_CLOUD_INFRASTRUCTURE/05_Security/) | Secrets, Zero trust, Container security | Google, Stripe |
| [06_AWS_Patterns](./03_CLOUD_INFRASTRUCTURE/06_AWS_Patterns/) | Lambda, DynamoDB, ElastiCache | Various |
| [07_Cost_Optimization](./03_CLOUD_INFRASTRUCTURE/07_Cost_Optimization/) | Spot instances, FinOps, Right-sizing | Netflix, Airbnb |
| [08_Platform_Engineering](./03_CLOUD_INFRASTRUCTURE/08_Platform_Engineering/) | Backstage, IaC, Golden paths | Spotify, Netflix |

---

## Featured Case Studies

### Must-Read Production Challenges

1. **[Discord: Cassandra to ScyllaDB](./01_BACKEND_PRODUCTION/02_Database_Architecture/nosql_migrations.md)** - Handling trillions of messages
2. **[Stripe: Idempotency Keys](./01_BACKEND_PRODUCTION/04_API_Design/idempotency_patterns/README.md)** - Preventing double charges
3. **[Netflix: Cache Consistency](./01_BACKEND_PRODUCTION/01_Distributed_Systems/cache_consistency.md)** - Event-driven invalidation
4. **[Zero-Downtime Deploys](./03_CLOUD_INFRASTRUCTURE/02_CI_CD_Deployment/zero_downtime_deploy.md)** - PreStop hooks and SIGTERM
5. **WhatsApp: Read Receipts** - Billion-scale write throughput (coming soon)
6. **LinkedIn: LLM Feed Ranking** - Real-time personalization (coming soon)
7. **Pinterest: PinSage** - Visual search with GNN (coming soon)
8. **Airbnb: Embedding Search** - ANN algorithms (coming soon)
9. **AWS/Azure Oct 2025 Outages** - Multi-cloud lessons (coming soon)
10. **Uber: GCP + Oracle Strategy** - Multi-cloud (coming soon)

---

## Companies Referenced

| Company | Key Contributions |
|---------|-------------------|
| **Netflix** | Microservices, Chaos Engineering, EVCache, GraphQL (DGS) |
| **Uber** | Multi-cloud, M3 metrics, Kafka at scale, Arm adoption |
| **Discord** | Elixir/BEAM, ScyllaDB migration, Real-time messaging |
| **Stripe** | Idempotency, Observability, Payment reliability |
| **Spotify** | Kubernetes, Backstage |
| **Airbnb** | Bazel, React optimization, Search ranking, Server-driven UI |
| **LinkedIn** | Feed ranking, Real-time personalization, Kafka |
| **Pinterest** | Visual search, PinSage, Recommendations |
| **WhatsApp** | WebSockets, Read receipts, Billion-scale systems |
| **Cloudflare** | Edge computing, ClickHouse, Workers |

---

## Research Sources

### Engineering Blogs (2024-2026)

- [Netflix Tech Blog](https://netflixtechblog.com/)
- [Uber Engineering](https://eng.uber.com/)
- [Discord Engineering](https://discord.com/blog)
- [Stripe Engineering](https://stripe.com/blog/engineering)
- [Spotify Engineering](https://engineering.atspotify.com/)
- [Airbnb Engineering](https://medium.com/airbnb-engineering)
- [LinkedIn Engineering](https://engineering.linkedin.com/)
- [Pinterest Engineering](https://medium.com/pinterest-engineering)

### Conferences

- QCon (quarterly)
- AWS re:Invent (November)
- Google Cloud Next (April)
- KubeCon (March, November)

---

## Contributing

Before adding a case study:

1. Read [STYLE_GUIDE.md](./STYLE_GUIDE.md) completely
2. Check [PROGRESS.md](./PROGRESS.md) for what's needed
3. Follow the format in [MASTER_PLAN.md](./MASTER_PLAN.md)
4. Include layman explanations + inline comments
5. Add before/after metrics when available

---

**Last Updated**: February 2026
**Version**: 1.0
