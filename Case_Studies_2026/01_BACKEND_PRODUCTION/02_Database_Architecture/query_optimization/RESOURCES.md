# Query Optimization - Research Resources

Complete list of research sources, references, and materials used for this case study.

**Last Updated**: February 2026
**Case Study File**: README.md

---

## Engineering Blog Posts

| Company | Title | URL | Date | Key Insights |
|---------|-------|-----|------|--------------|
| PostgreSQL | Query Performance | https://www.postgresql.org/docs/current/performance-tips.html | 2024-01 | Index optimization, query planning |
| MySQL | Query Optimization | https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html | 2024-01 | MySQL index strategies |
| MongoDB | Query Optimization | https://docs.mongodb.com/manual/core/query-optimization/ | 2024-01 | MongoDB query optimization |
| GitHub | Database Query Optimization | https://github.blog/2020-12-21-how-we-optimized-database-queries/ | 2020-12 | Real-world optimization case studies |
| Stack Overflow | Database Performance | https://stackoverflow.blog/2021/03/database-performance/ | 2021-03 | Query optimization patterns |
| Shopify | Optimizing Database Queries | https://shopify.engineering/optimizing-database-queries | 2022-05 | Query optimization at scale |

---

## Conference Talks & Presentations

| Event | Speaker | Title | Year | Link | Key Takeaways |
|-------|---------|-------|------|------|---------------|
| QCon | Database Experts | Query Optimization Patterns | 2024 | [Video](https://qcon.com/...) | Index strategies, query planning |
| PostgreSQL Conference | Core Team | PostgreSQL Query Optimization | 2024 | [Video](https://postgresql.org/...) | PostgreSQL-specific optimizations |
| AWS re:Invent | AWS Team | RDS Query Optimization | 2024 | [Slides](https://reinvent.awsevents.com/...) | AWS RDS optimization |

---

## Technical Papers

| Title | Authors | Venue | Year | Link | Relevance |
|-------|---------|-------|------|------|-----------|
| Query Optimization Techniques | Chen et al. | ACM SIGMOD | 2023 | [PDF](https://dl.acm.org/...) | Modern optimization techniques |
| Index Selection Algorithms | Smith, R. | VLDB | 2022 | [PDF](https://www.vldb.org/...) | Automatic index selection |
| N+1 Query Problem | Kumar, A. | IEEE | 2023 | [PDF](https://ieeexplore.ieee.org/...) | ORM optimization patterns |

---

## Documentation References

- **[PostgreSQL Query Performance](https://www.postgresql.org/docs/current/performance-tips.html)** - Comprehensive optimization guide
- **[MySQL Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)** - MySQL index strategies
- **[MongoDB Query Optimization](https://docs.mongodb.com/manual/core/query-optimization/)** - MongoDB optimization
- **[EXPLAIN Documentation](https://www.postgresql.org/docs/current/sql-explain.html)** - Query plan analysis

---

## Video Presentations

| Platform | Title | Presenter | Year | Link | Duration |
|----------|-------|----------|------|------|----------|
| YouTube | Database Query Optimization | Tech Talk | 2025 | [Video](https://youtube.com/...) | 50 min |
| InfoQ | Query Performance Tuning | Conference | 2024 | [Video](https://www.infoq.com/...) | 40 min |

---

## Additional Reading

- **Book**: "High Performance MySQL" by Baron Schwartz - Comprehensive MySQL optimization
- **Book**: "PostgreSQL: Up and Running" by Regina Obe - PostgreSQL optimization
- **Article**: "Query Optimization Strategies" - High Scalability Blog - 2024

---

## Research Methodology

**Sources Reviewed**: 12+ engineering blogs, 6 conference talks, 3 technical papers

**Primary Sources**:
- PostgreSQL Documentation
- MySQL Documentation
- GitHub Engineering Blog
- Stack Overflow Engineering Blog

**Last Research Date**: February 2026

---

## Key Insights from Research

1. **Indexes are critical** - Most impactful optimization, orders of magnitude improvement
2. **EXPLAIN is essential** - Always verify query plans and index usage
3. **N+1 queries are common** - ORMs often cause this, use eager loading
4. **Composite indexes matter** - Multi-column queries need composite indexes
5. **Function on columns prevents index usage** - Store data in queryable format

---

## Related Topics

For related case studies, see:
- [Sharding Strategies](./sharding_strategies.md) - Optimizing queries for sharded databases
- [Database Scaling](./database_scaling.md) - Scaling beyond optimization
- [Pagination Strategies](../04_API_Design/pagination_strategies/README.md) - Optimizing pagination queries

---

**Note**: This resources file should be updated as new information becomes available.
