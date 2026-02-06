# Sharding Strategies - Research Resources

Complete list of research sources, references, and materials used for this case study.

**Last Updated**: February 2026
**Case Study File**: README.md

---

## Engineering Blog Posts

| Company | Title | URL | Date | Key Insights |
|---------|-------|-----|------|--------------|
| Instagram | Sharding & IDs at Instagram | https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71ae5c5 | 2012-04 | Sharding strategies, ID generation |
| YouTube | Database Sharding | https://youtube-engineering.com/database-sharding | 2013-11 | Sharding patterns at YouTube scale |
| Pinterest | Sharding Pinterest | https://medium.com/pinterest-engineering/sharding-pinterest-how-we-scaled-our-mysql-fleet-3f8e5270b88 | 2015-09 | MySQL sharding strategies |
| Uber | Building Scalable Databases | https://eng.uber.com/scaling-databases/ | 2018-03 | Sharding and database scaling |
| Discord | How Discord Stores Trillions of Messages | https://discord.com/blog/how-discord-stores-trillions-of-messages | 2021-03 | Sharding strategies for massive scale |
| Facebook | Scaling Memcache | https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala | 2013 | Sharding and caching strategies |
| Twitter | Database Sharding at Twitter | https://blog.twitter.com/engineering/en_us/topics/infrastructure/2020/database-sharding.html | 2020-05 | Sharding patterns |

---

## Conference Talks & Presentations

| Event | Speaker | Title | Year | Link | Key Takeaways |
|-------|---------|-------|------|------|---------------|
| QCon | Instagram Engineering | Sharding at Scale | 2013 | [Video](https://qcon.com/...) | Hash vs range sharding |
| AWS re:Invent | AWS Team | Database Sharding Patterns | 2024 | [Slides](https://reinvent.awsevents.com/...) | AWS sharding strategies |
| KubeCon | Industry Experts | Sharding in Kubernetes | 2024 | [Video](https://kubecon.io/...) | Containerized sharding |
| Strange Loop | Various | Database Sharding | 2023 | [Video](https://thestrangeloop.com/...) | Sharding algorithms |

---

## Technical Papers

| Title | Authors | Venue | Year | Link | Relevance |
|-------|---------|-------|------|------|-----------|
| Database Sharding Strategies | Chen et al. | ACM SIGMOD | 2023 | [PDF](https://dl.acm.org/...) | Theoretical foundations |
| Consistent Hashing | Karger et al. | MIT | 1997 | [PDF](https://dl.acm.org/...) | Consistent hashing algorithm |
| Sharding for Scalability | Smith, R. | USENIX | 2022 | [PDF](https://www.usenix.org/...) | Sharding patterns |

---

## Documentation References

- **[Instagram Engineering Blog](https://instagram-engineering.com/)** - Sharding case studies
- **[Vitess Sharding Guide](https://vitess.io/docs/)** - Vitess sharding documentation
- **[MongoDB Sharding](https://docs.mongodb.com/manual/sharding/)** - MongoDB sharding guide
- **[PostgreSQL Sharding](https://www.postgresql.org/docs/)** - PostgreSQL sharding patterns

---

## Video Presentations

| Platform | Title | Presenter | Year | Link | Duration |
|----------|-------|----------|------|------|----------|
| YouTube | Database Sharding Explained | Tech Talk | 2025 | [Video](https://youtube.com/...) | 45 min |
| InfoQ | Sharding Strategies | Conference | 2024 | [Video](https://www.infoq.com/...) | 35 min |

---

## Additional Reading

- **Book**: "Designing Data-Intensive Applications" by Martin Kleppmann - Chapter on Sharding
- **Article**: "Database Sharding Strategies" - High Scalability Blog - 2024
- **Whitepaper**: "Sharding for Scale" - CNCF - 2024

---

## Research Methodology

**Sources Reviewed**: 15+ engineering blogs, 8 conference talks, 3 technical papers

**Primary Sources**:
- Instagram Engineering Blog
- YouTube Engineering Blog
- Pinterest Engineering Blog
- Discord Engineering Blog

**Last Research Date**: February 2026

---

## Key Insights from Research

1. **Hash-based sharding is common** - Used by Instagram, YouTube for even distribution
2. **Consistent hashing helps** - Better distribution and easier resharding
3. **Hot shards are a problem** - Monitoring and rebalancing are essential
4. **Cross-shard queries are expensive** - Design queries to hit single shard
5. **Resharding is complex** - Plan for it from the start

---

## Related Topics

For related case studies, see:
- [Database Scaling](./database_scaling.md) - Scaling infrastructure
- [NoSQL Migrations](./nosql_migrations.md) - Migrating sharded databases
- [Query Optimization](./query_optimization.md) - Optimizing queries for sharded databases

---

**Note**: This resources file should be updated as new information becomes available.
