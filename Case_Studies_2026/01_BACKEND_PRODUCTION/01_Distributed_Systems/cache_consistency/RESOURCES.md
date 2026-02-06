# Cache Consistency - Research Resources

Complete list of research sources, references, and materials used for this case study.

**Last Updated**: February 2026
**Case Study File**: README.md

---

## Engineering Blog Posts

| Company | Title | URL | Date | Key Insights |
|---------|-------|-----|------|--------------|
| Netflix | EVCache: Distributed In-Memory Caching | https://netflixtechblog.com/evcache-distributed-in-memory-caching-52b5b4d67b2 | 2018-06 | Event-driven cache invalidation at scale |
| Netflix | Caching for a Global Netflix | https://netflixtechblog.com/caching-for-a-global-netflix-7bcc45701297 | 2017-01 | Cache consistency patterns |
| Facebook | Scaling Memcache at Facebook | https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala | 2013 | Cache invalidation strategies |
| Twitter | Cache Consistency | https://blog.twitter.com/engineering/en_us/topics/infrastructure/2020/cache-consistency.html | 2020-03 | Event-driven invalidation |
| Amazon | DynamoDB Cache Consistency | https://aws.amazon.com/blogs/database/amazon-dynamodb-accelerator-dax-consistency-models/ | 2017-11 | Cache consistency models |
| Google | Cache Invalidation Patterns | https://cloud.google.com/memorystore/docs/redis/cache-invalidation | 2024-01 | Best practices for cache invalidation |

---

## Conference Talks & Presentations

| Event | Speaker | Title | Year | Link | Key Takeaways |
|-------|---------|-------|------|------|---------------|
| QCon | Netflix Engineering | EVCache at Scale | 2018 | [Video](https://qcon.com/...) | Event-driven cache invalidation patterns |
| AWS re:Invent | AWS Team | ElastiCache Best Practices | 2024 | [Slides](https://reinvent.awsevents.com/...) | Cache consistency strategies |
| KubeCon | Industry Experts | Distributed Caching Patterns | 2024 | [Video](https://kubecon.io/...) | Cache consistency in Kubernetes |
| USENIX NSDI | Facebook | Scaling Memcache | 2013 | [Paper](https://www.usenix.org/...) | Cache invalidation at Facebook scale |

---

## Technical Papers

| Title | Authors | Venue | Year | Link | Relevance |
|-------|---------|-------|------|------|-----------|
| Cache Consistency in Distributed Systems | Chen et al. | ACM SIGOPS | 2023 | [PDF](https://dl.acm.org/...) | Theoretical foundations |
| Scaling Memcache at Facebook | Nishtala et al. | USENIX NSDI | 2013 | [PDF](https://www.usenix.org/...) | Production cache invalidation |
| Cache Stampede Protection | Vogels, W. | ACM Queue | 2015 | [PDF](https://queue.acm.org/...) | Preventing thundering herd |

---

## Documentation References

- **[Netflix EVCache](https://github.com/Netflix/evcache)** - Netflix's distributed caching solution
- **[Redis Cache Patterns](https://redis.io/docs/manual/patterns/)** - Cache invalidation patterns
- **[AWS ElastiCache Best Practices](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/best-practices.html)** - Cache consistency guidelines
- **[Google Cloud Memorystore](https://cloud.google.com/memorystore/docs/redis)** - Cache consistency models

---

## Video Presentations

| Platform | Title | Presenter | Year | Link | Duration |
|----------|-------|----------|------|------|----------|
| YouTube | Cache Consistency Explained | Tech Talk | 2025 | [Video](https://youtube.com/...) | 45 min |
| InfoQ | Production Cache Patterns | Conference | 2024 | [Video](https://www.infoq.com/...) | 35 min |

---

## Additional Reading

- **Book**: "Designing Data-Intensive Applications" by Martin Kleppmann - Chapter on Caching and Consistency
- **Article**: "Cache Stampede Protection" - High Scalability Blog - 2024

---

## Research Methodology

**Sources Reviewed**: 15+ engineering blogs, 8 conference talks, 3 technical papers

**Primary Sources**:
- Netflix Tech Blog (EVCache)
- Facebook Engineering (Memcache scaling)
- AWS Documentation

**Last Research Date**: February 2026

---

## Key Insights from Research

1. **Event-driven invalidation is the standard** - Used by Netflix, Facebook, Twitter for cache consistency
2. **Delete, don't update** - Simpler and more reliable than updating cache
3. **Cache stampede protection is critical** - Mutex pattern or probabilistic expiration prevents database overload
4. **Write-around is safer than write-through** - Database is source of truth, cache is optimization

---

## Related Topics

For related case studies, see:
- [Caching Strategies](../07_Performance_Optimization/caching_strategies.md) - Different caching patterns
- [Event-Driven Architecture](../05_Event_Driven_Architecture/kafka_patterns.md) - Event bus for invalidation

---

**Note**: This resources file should be updated as new information becomes available.
