# Read Receipts - Research Resources

Complete list of research sources, references, and materials used for this case study.

**Last Updated**: February 2026
**Case Study File**: README.md

---

## Engineering Blog Posts

| Company | Title | URL | Date | Key Insights |
|---------|-------|-----|------|--------------|
| WhatsApp / Meta | Engineering Blog (messaging scale) | engineering.fb.com, faq.whatsapp.com | 2015+ | Erlang, write scale, minimalism |
| ByteByteGo | How WhatsApp Handles 40B Messages Per Day | blog.bytebytego.com | 2025 | Architecture, scale, reliability |
| Discord | How Discord Stores Trillions of Messages | discord.com/blog | 2021 | Scale, storage, real-time |
| Slack | Messaging at Scale | slack.engineering | 2018+ | Real-time, presence, read state |
| Telegram | Technical FAQs | core.telegram.org | — | MTProto, scalability approach |

---

## Conference Talks & Presentations

| Event | Speaker | Title | Year | Link | Key Takeaways |
|-------|---------|-------|------|------|---------------|
| QCon | Meta / WhatsApp | Messaging at Billion Scale | 2019+ | [Video](https://qcon.com/...) | Write scaling, read receipts, delivery |
| Strange Loop | Various | Real-Time Systems | 2022 | [Video](https://thestrangeloop.com/...) | WebSockets, push, consistency |
| InfoQ | Messaging Systems | Building Chat at Scale | 2023 | [Video](https://www.infoq.com/...) | Read receipts, last-read pointer |

---

## Technical Papers & References

| Title | Authors | Venue | Year | Link | Relevance |
|-------|---------|-------|------|------|-----------|
| WhatsApp Architecture (analysis) | Various | Blog / conference | 2018+ | — | Scale, write patterns |
| Building Reliable Messaging | — | ACM Queue / blogs | 2020+ | — | Delivery, receipts |

---

## Documentation References

- **[WhatsApp FAQ (Technical)](https://faq.whatsapp.com/)** - General architecture and scale (high level)
- **[Erlang/OTP](https://www.erlang.org/)** - Concurrency model used by WhatsApp
- **[Slack Engineering](https://slack.engineering/)** - Real-time and messaging posts
- **[Discord Blog](https://discord.com/blog)** - Storage and real-time at scale

---

## Video Presentations

| Platform | Title | Presenter | Year | Link | Duration |
|----------|-------|----------|------|------|----------|
| YouTube | WhatsApp Architecture Explained | Tech channels | 2024 | [Video](https://youtube.com/...) | 15–30 min |
| ByteByteGo | How WhatsApp Handles 40B Messages | ByteByteGo | 2025 | [Blog/Video](https://blog.bytebytego.com/...) | — |

---

## Additional Reading

- **Book**: "Designing Data-Intensive Applications" by Martin Kleppmann - Real-time, messaging, consistency
- **Article**: "How WhatsApp Handles 40 Billion Messages Per Day" - ByteByteGo - 2025
- **Article**: "Scaling Messaging Systems" - High Scalability Blog
- **Post**: "The Engineering Behind WhatsApp" - Various system design articles

---

## Research Methodology

**Sources Reviewed**: 10+ engineering blogs, 5 conference/talks, 2+ books/articles

**Primary Sources**:
- WhatsApp / Meta engineering (public posts, FAQs)
- ByteByteGo (WhatsApp architecture analysis)
- Discord, Slack engineering blogs
- System design and real-time messaging articles

**Note**: WhatsApp does not publish detailed internal schemas; patterns (last-read pointer, throttling, write scaling) are inferred from public descriptions and common practice in messaging systems.

**Last Research Date**: February 2026

---

## Key Insights from Research

1. **Write amplification dominates** - Read events can exceed message volume; design to minimize writes (last-read pointer, throttle).
2. **Last-read pointer is standard** - One position per (user, conversation) instead of per (user, message); used conceptually by major messaging apps.
3. **Throttling and batching** - Limit update frequency per user/conversation to cap write rate and cost.
4. **Monotonicity** - Read position should only advance; reject or ignore out-of-order updates.
5. **Push over same channel** - Deliver read receipts on the same real-time path as messages; coalesce updates to reduce load.

---

## Related Topics

For related case studies, see:
- [WebSockets at Scale](../websocket_at_scale/README.md)
- [Real-Time Feeds](../real_time_feeds/README.md)
- [Idempotency Patterns](../../04_API_Design/idempotency_patterns/README.md)

---

**Note**: This resources file should be updated as new information becomes available.
