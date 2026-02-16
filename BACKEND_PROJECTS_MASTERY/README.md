# Backend Engineering Mastery Projects 2026
## Learning Path: NestJS, Golang & Python

This repository contains 15 carefully designed backend projects to master backend engineering concepts. Each project is designed to be implemented in **three languages** (NestJS/TypeScript, Golang, and Python) to understand how different tech stacks solve the same problems.

---

## Why Three Languages?

- **NestJS (TypeScript)**: Enterprise-grade framework, decorator-based, built for microservices
- **Golang**: High-performance, built-in concurrency, ideal for distributed systems
- **Python**: Rapid development, rich ecosystem, excellent for data processing and ML integration

---

## Project Overview & Progression

### Phase 1: Foundations (Projects 1-3)
Building blocks for backend systems with essential patterns.

### Phase 2: Distributed Systems (Projects 4-7)
Understanding distributed computing, consistency, and coordination.

### Phase 3: Advanced Patterns (Projects 8-11)
Complex architectural patterns and real-world scenarios.

### Phase 4: Modern Infrastructure (Projects 12-15)
2026-relevant technologies including AI integration and observability.

---

## Projects List

| # | Project Name | Difficulty | Key Concepts | Time Estimate |
|---|-------------|------------|--------------|---------------|
| 01 | Distributed URL Shortener | Intermediate | Caching, Consistent Hashing, High Availability | 2-3 weeks |
| 02 | Real-Time Collaborative Task Manager | Intermediate | WebSockets, Event-Driven, CQRS | 2-3 weeks |
| 03 | Multi-Tenant SaaS API Gateway | Intermediate | Rate Limiting, Auth, Multi-tenancy | 2-3 weeks |
| 04 | Payment Processing Engine | Advanced | Saga Pattern, Idempotency, Event Sourcing | 3-4 weeks |
| 05 | Stream Processing Analytics Platform | Advanced | Kafka, Stream Processing, Time-Series | 3-4 weeks |
| 06 | Distributed File Storage System | Advanced | Object Storage, Sharding, CDN | 3-4 weeks |
| 07 | GraphQL Federation Backend | Advanced | GraphQL, Service Mesh, Federation | 3-4 weeks |
| 08 | E-Commerce Inventory System | Advanced | Distributed Locks, Eventual Consistency | 3-4 weeks |
| 09 | Social Media Feed Engine | Advanced | Fan-out, Graph DB, Caching Strategies | 3-4 weeks |
| 10 | AI-Powered Search & RAG System | Advanced | Vector DB, Embeddings, RAG Pattern | 4-5 weeks |
| 11 | Distributed Caching Layer | Advanced | Cache Strategies, Invalidation, Clusters | 2-3 weeks |
| 12 | Observability & APM System | Advanced | Metrics, Logs, Traces, Alerting | 3-4 weeks |
| 13 | Multi-Region Auth Service | Advanced | JWT, OAuth2, Geo-Replication | 3-4 weeks |
| 14 | Background Job Processor | Intermediate | Job Queues, Retry, Priority Queues | 2-3 weeks |
| 15 | API Rate Limiter Service | Intermediate | Token Bucket, Sliding Window, Redis | 2-3 weeks |

---

## Concept Coverage Matrix

Each project is designed to teach specific concepts from the Backend Engineering Mastery framework:

### Data Structures & Algorithms
- Hash tables, Trees, Graphs, Tries
- Consistent hashing, Bloom filters
- Time-series data structures

### System Design
- Scalability, Availability, Reliability
- CAP theorem, Consistency models
- Load balancing, Caching strategies

### Databases
- SQL (PostgreSQL) - ACID, Transactions, Indexing
- NoSQL (MongoDB, Cassandra) - Eventual consistency
- Redis - Caching, Pub/Sub
- Vector DBs (Pinecone, Weaviate) - AI/ML integration
- Time-series (InfluxDB, TimescaleDB)
- Graph DBs (Neo4j) - Relationships

### APIs
- REST API design
- GraphQL & Federation
- gRPC & Protocol Buffers
- WebSockets for real-time

### Messaging & Events
- Message queues (RabbitMQ, Kafka)
- Event sourcing
- CQRS
- Stream processing

### Microservices
- Service communication patterns
- Resilience patterns (Circuit breaker, Bulkhead, Retry)
- Saga pattern
- Service mesh

### Concurrency
- Thread pools, Async/Await
- Synchronization primitives
- Race conditions, Deadlocks
- Event loops

### Security
- Authentication (JWT, OAuth2, Session)
- Authorization (RBAC, ABAC)
- OWASP Top 10
- Encryption, TLS/SSL

### Performance
- Profiling, Bottleneck identification
- Query optimization
- Connection pooling
- Caching strategies

### Observability
- Structured logging
- Metrics (Prometheus)
- Distributed tracing (Jaeger, OpenTelemetry)
- Alerting, SLOs

### DevOps
- Docker, Docker Compose
- CI/CD basics
- Infrastructure as Code
- Health checks, Graceful shutdown

### 2026 Trends
- AI/ML integration (Vector DBs, RAG)
- Real-time analytics
- Edge computing patterns
- Zero trust security

---

## How to Use This Repository

### Step 1: Choose a Project
Start with Project 01 if you're beginning your journey, or pick a project that interests you.

### Step 2: Read the Project Documentation
Each project has:
- **Overview**: What you're building and why
- **Layman Explanation**: Non-technical explanation
- **Technical Requirements**: What you need to know
- **Architecture**: System design diagrams
- **Learning Objectives**: What you'll learn
- **Implementation Guide**: Step-by-step approach
- **Testing Strategy**: How to test your system
- **Deployment**: How to run in production

### Step 3: Implement in All Three Languages
1. Start with the language you're most comfortable with
2. Implement the same project in the second language
3. Complete with the third language
4. Compare implementations and note differences

### Step 4: Build and Test
- Write comprehensive tests (unit, integration, e2e)
- Set up observability (logs, metrics, traces)
- Load test your system
- Document your learnings

### Step 5: Review and Iterate
- Review the code with the concepts from mastery documents
- Identify areas for improvement
- Optimize based on profiling data
- Document architectural decisions

---

## Recommended Learning Path

### For Beginners (0-1 years experience)
1. Project 01: URL Shortener
2. Project 02: Task Manager
3. Project 14: Job Processor
4. Project 15: Rate Limiter
5. Project 03: API Gateway

### For Intermediate (1-3 years experience)
1. Project 04: Payment Engine
2. Project 05: Stream Analytics
3. Project 08: Inventory System
4. Project 09: Social Feed
5. Project 13: Auth Service

### For Advanced (3+ years experience)
1. Project 06: File Storage
2. Project 07: GraphQL Federation
3. Project 10: AI Search & RAG
4. Project 11: Caching Layer
5. Project 12: Observability System

---

## Tech Stack Requirements

### Common Tools (All Projects)
- **Git**: Version control
- **Docker & Docker Compose**: Containerization
- **PostgreSQL**: Primary relational database
- **Redis**: Caching and pub/sub
- **Postman/Insomnia**: API testing

### Language-Specific

#### NestJS Stack
- Node.js 20+
- NestJS 10+
- TypeORM or Prisma
- Jest for testing
- Class-validator, Class-transformer

#### Golang Stack
- Go 1.22+
- Gin or Fiber (web framework)
- GORM (ORM)
- Testify for testing
- go-redis, pq (PostgreSQL driver)

#### Python Stack
- Python 3.12+
- FastAPI
- SQLAlchemy or Django ORM
- Pytest for testing
- Pydantic for validation

### Infrastructure & Tools
- **Message Queues**: RabbitMQ, Kafka
- **Monitoring**: Prometheus, Grafana
- **Tracing**: Jaeger, OpenTelemetry
- **Load Testing**: k6, Locust
- **NoSQL**: MongoDB, Cassandra
- **Search**: Elasticsearch, OpenSearch
- **Vector DB**: Pinecone, Weaviate (for AI projects)

---

## Project Structure

Each project folder contains:

```
project-name/
├── README.md                    # Comprehensive project documentation
├── ARCHITECTURE.md              # System design and architecture
├── LEARNING_OBJECTIVES.md       # What you'll learn
├── IMPLEMENTATION_GUIDE.md      # Step-by-step implementation
├── TESTING_STRATEGY.md          # How to test
├── nestjs/                      # NestJS implementation
│   ├── src/
│   ├── test/
│   ├── docker-compose.yml
│   └── README.md
├── golang/                      # Golang implementation
│   ├── cmd/
│   ├── internal/
│   ├── test/
│   ├── docker-compose.yml
│   └── README.md
└── python/                      # Python implementation
    ├── src/
    ├── tests/
    ├── docker-compose.yml
    └── README.md
```

---

## Success Criteria

You've successfully mastered backend engineering when you can:

1. **Design** scalable systems with proper trade-off analysis
2. **Implement** systems in multiple languages/frameworks
3. **Explain** architectural decisions and their implications
4. **Debug** distributed systems effectively
5. **Optimize** for performance based on profiling data
6. **Secure** applications against OWASP Top 10
7. **Observe** systems with metrics, logs, and traces
8. **Deploy** applications with proper CI/CD
9. **Lead** technical discussions and code reviews
10. **Mentor** junior engineers on best practices

---

## Resources

### Books
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "System Design Interview" by Alex Xu
- "Release It!" by Michael Nygard
- "Database Internals" by Alex Petrov

### Online Resources
- Backend Engineering Mastery Documents (in this repo)
- System Design Primer (GitHub)
- High Scalability Blog
- AWS/GCP Architecture Blogs

### Practice
- LeetCode (Data Structures & Algorithms)
- System Design Mock Interviews
- Open Source Contributions
- Tech Blog Writing

---

## Contributing

This is your personal learning repository. As you build these projects:

1. Document your learnings in each project's README
2. Add notes on challenges faced and solutions
3. Include performance benchmarks
4. Share architectural decision records (ADRs)
5. Create diagrams for complex flows

---

## Next Steps

1. Read through all project descriptions
2. Choose your first project based on your experience level
3. Set up your development environment
4. Start building and learning!

Remember: The goal is not just to complete projects, but to **understand the concepts deeply** and be able to **explain trade-offs** in technical discussions.

Good luck on your journey to backend engineering mastery!
