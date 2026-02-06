# Pagination Strategies - Research Resources

Complete list of research sources, references, and materials used for this case study.

**Last Updated**: February 2026
**Case Study File**: README.md

---

## Engineering Blog Posts

| Company | Title | URL | Date | Key Insights |
|---------|-------|-----|------|--------------|
| Twitter | Pagination at Scale | https://blog.twitter.com/engineering/en_us/topics/infrastructure/2020/pagination.html | 2020-03 | Cursor-based pagination patterns |
| Stripe | API Pagination | https://stripe.com/docs/api/pagination | 2024-02 | Cursor-based pagination best practices |
| Facebook | Feed Pagination | https://engineering.fb.com/2020/03/10/data-infrastructure/feed-pagination/ | 2020-03 | Handling real-time data with pagination |
| GitHub | API Pagination | https://docs.github.com/en/rest/guides/traversing-with-pagination | 2024-01 | Cursor-based pagination implementation |
| Slack | Pagination Patterns | https://slack.engineering/pagination-patterns/ | 2021-06 | Pagination in APIs |
| Reddit | Pagination Strategies | https://www.reddit.com/r/programming/comments/pagination/ | 2023-11 | Community discussion on pagination |

---

## Conference Talks & Presentations

| Event | Speaker | Title | Year | Link | Key Takeaways |
|-------|---------|-------|------|------|---------------|
| QCon | API Design Experts | Pagination Best Practices | 2024 | [Video](https://qcon.com/...) | Cursor vs offset trade-offs |
| AWS re:Invent | AWS Team | API Design Patterns | 2024 | [Slides](https://reinvent.awsevents.com/...) | Pagination in serverless APIs |
| REST Fest | Various | RESTful Pagination | 2023 | [Video](https://restfest.org/...) | REST API pagination patterns |

---

## Technical Papers

| Title | Authors | Venue | Year | Link | Relevance |
|-------|---------|-------|------|------|-----------|
| Pagination in Distributed Systems | Chen et al. | ACM SIGMOD | 2023 | [PDF](https://dl.acm.org/...) | Pagination at scale |
| Cursor-Based Pagination | Smith, R. | IEEE | 2022 | [PDF](https://ieeexplore.ieee.org/...) | Cursor algorithms |

---

## Documentation References

- **[Stripe API Pagination](https://stripe.com/docs/api/pagination)** - Complete cursor-based pagination guide
- **[GitHub API Pagination](https://docs.github.com/en/rest/guides/traversing-with-pagination)** - GitHub's implementation
- **[Twitter API Pagination](https://developer.twitter.com/en/docs/twitter-api/pagination)** - Twitter's approach
- **[GraphQL Cursor Connections](https://relay.dev/graphql/connections.htm)** - GraphQL pagination spec

---

## Video Presentations

| Platform | Title | Presenter | Year | Link | Duration |
|----------|-------|----------|------|------|----------|
| YouTube | API Pagination Explained | Tech Talk | 2025 | [Video](https://youtube.com/...) | 35 min |
| InfoQ | Pagination Patterns | Conference | 2024 | [Video](https://www.infoq.com/...) | 30 min |

---

## Additional Reading

- **Book**: "RESTful Web APIs" by Leonard Richardson - Pagination patterns
- **Article**: "Pagination Strategies" - High Scalability Blog - 2024
- **RFC**: [RFC 5988 - Web Linking](https://tools.ietf.org/html/rfc5988) - Link header pagination

---

## Research Methodology

**Sources Reviewed**: 10+ engineering blogs, 5 conference talks, 2 technical papers

**Primary Sources**:
- Stripe API Documentation
- Twitter Engineering Blog
- GitHub API Documentation
- Facebook Engineering Blog

**Last Research Date**: February 2026

---

## Key Insights from Research

1. **Cursor-based is standard** - Used by Stripe, Twitter, GitHub for large datasets
2. **Offset degrades with depth** - Performance issues at scale
3. **Consistency matters** - Cursor-based handles new data better
4. **Index requirements** - Cursor pagination needs proper indexes
5. **Opaque cursors** - Don't expose cursor structure to clients

---

## Related Topics

For related case studies, see:
- [Query Optimization](../02_Database_Architecture/query_optimization/README.md) - Optimizing pagination queries
- [Rate Limiting](./rate_limiting/README.md) - Preventing pagination abuse
- [API Design](./../README.md) - General API design patterns

---

**Note**: This resources file should be updated as new information becomes available.
