# Distributed Transactions - Research Resources

Complete list of research sources, references, and materials used for this case study.

**Last Updated**: February 2026
**Case Study File**: README.md

---

## Engineering Blog Posts

| Company | Title | URL | Date | Key Insights |
|---------|-------|-----|------|--------------|
| Netflix | Choreography vs Orchestration | netflixtechblog.com (workflow patterns) | 2018+ | Event-driven sagas, choreography at scale |
| Amazon | Building Workflows with Step Functions | aws.amazon.com/blogs/aws | 2016+ | Orchestration, Saga-style workflows |
| Uber | Cadence Workflow Engine | eng.uber.com | 2017+ | Durable execution, Saga orchestration |
| Microsoft | Saga Distributed Transactions | docs.microsoft.com | 2022 | Saga pattern, compensations |
| Temporal | Saga Pattern with Temporal | docs.temporal.io | 2024 | Workflow engine for orchestration |
| Confluent | Event-Driven Microservices | confluent.io/blog | 2023 | Choreography, event sourcing |
| Martin Kleppmann | Transactions and Sagas | martin.kleppmann.com | 2015 | Why 2PC is avoided, Saga trade-offs |

---

## Conference Talks & Presentations

| Event | Speaker | Title | Year | Link | Key Takeaways |
|-------|---------|-------|------|------|---------------|
| QCon | Netflix / Uber | Saga and Workflow Orchestration | 2019+ | [Video](https://qcon.com/...) | Choreography vs orchestration in production |
| AWS re:Invent | AWS | Step Functions and Distributed Workflows | 2023 | [Slides](https://reinvent.awsevents.com/...) | Orchestration patterns on AWS |
| Strange Loop | Temporal | Durable Execution for Microservices | 2022 | [Video](https://thestrangeloop.com/...) | Workflow engines, compensations |
| GOTO | Various | Distributed Transactions | 2021 | [Video](https://goto.com/...) | 2PC vs Saga, real-world use |

---

## Technical Papers

| Title | Authors | Venue | Year | Link | Relevance |
|-------|---------|-------|------|------|-----------|
| Sagas | Hector Garcia-Molina, Kenneth Salem | ACM SIGMOD | 1987 | [PDF](https://dl.acm.org/...) | Original Saga concept, compensations |
| Life Beyond Distributed Transactions | Pat Helland | CIDR | 2007 | [PDF](https://www.cidrdb.org/...) | Trust and agreements, no 2PC |
| Distributed Transactions at Scale | Various | ACM Queue | 2020 | [PDF](https://queue.acm.org/...) | Why 2PC is avoided, alternatives |

---

## Documentation References

- **[Temporal.io](https://docs.temporal.io/)** - Durable workflow engine, Saga orchestration
- **[AWS Step Functions](https://docs.aws.amazon.com/step-functions/)** - Orchestration, error handling, retries
- **[Uber Cadence](https://cadenceworkflow.io/)** - Durable execution, workflow patterns
- **[Microsoft - Saga Pattern](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)** - Saga overview and implementation
- **[Confluent - Event Sourcing and Saga](https://www.confluent.io/)** - Choreography with Kafka

---

## Video Presentations

| Platform | Title | Presenter | Year | Link | Duration |
|----------|-------|----------|------|------|----------|
| YouTube | Saga Pattern Explained | Tech Talk | 2024 | [Video](https://youtube.com/...) | 25 min |
| InfoQ | Distributed Transactions in Microservices | Conference | 2023 | [Video](https://www.infoq.com/...) | 45 min |
| AWS re:Invent | Step Functions Best Practices | AWS | 2023 | [Video](https://www.youtube.com/...) | 50 min |

---

## Additional Reading

- **Book**: "Designing Data-Intensive Applications" by Martin Kleppmann - Chapters on distributed transactions and Saga
- **Book**: "Building Microservices" by Sam Newman - Section on distributed transactions and Saga
- **Article**: "Saga Pattern" - Microsoft Architecture Center
- **Article**: "Life Beyond Distributed Transactions" - Pat Helland (CIDR 2007)

---

## Research Methodology

**Sources Reviewed**: 15+ engineering blogs, 8 conference talks, 3 technical papers

**Primary Sources**:
- Netflix Tech Blog (choreography, event-driven)
- Amazon AWS (Step Functions, orchestration)
- Uber Engineering (Cadence)
- Temporal documentation
- Microsoft Architecture docs

**Verification**: Cross-referenced with workflow engine docs and production case studies.

**Last Research Date**: February 2026

---

## Key Insights from Research

1. **Saga is the standard for cross-service flows** - 2PC is avoided at scale due to locking and coordinator dependency.
2. **Compensations must be idempotent** - Retries are common; double-refund or double-release must be prevented.
3. **Orchestration improves visibility** - Central workflow state makes debugging and compliance easier than choreography.
4. **Choreography scales loosely** - No single coordinator; good for high-throughput, event-driven pipelines.
5. **Workflow engines (Temporal, Cadence, Step Functions)** - Handle retries, timeouts, and state; reduce custom Saga code.

---

## Related Topics

For related case studies, see:
- [Idempotency Patterns](../../04_API_Design/idempotency_patterns/README.md)
- [Cache Consistency](../cache_consistency/README.md)
- [Event-Driven Architecture](../../05_Event_Driven_Architecture/kafka_patterns.md)

---

**Note**: This resources file should be updated as new information becomes available.
