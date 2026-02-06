# STYLE GUIDE: Writing Case Studies

## Critical Rules for Every Case Study

**Focus: Crisp, Concise, Well-Researched Understanding**
- **No code implementation** - Focus on concepts, architecture, and decision-making
- **Well-researched** - Based on actual engineering blogs, papers, and conference talks
- **Crisp & Concise** - Get to the point quickly, no fluff
- **Sharp insights** - Deep understanding of why decisions were made
- **Complete resources** - All research sources documented in separate resources file
- **Minimum 3-5 case studies per file** - Each case study file (README.md) must contain a minimum of 3 case studies and a maximum of 5 case studies covering different aspects of the topic

---

## RULE 0: CASE STUDY COUNT REQUIREMENT (MANDATORY)

**Each case study file (README.md) must contain a minimum of 3 case studies and a maximum of 5 case studies.**

### Why This Requirement Exists

- **Comprehensive coverage**: Multiple case studies ensure the topic is covered from different angles
- **Better learning**: Readers see various scenarios and solutions
- **Real-world relevance**: Different companies/contexts provide diverse perspectives
- **Maintainable size**: 3-5 case studies keeps files focused and readable

### What Counts as a Case Study

Each case study must be:
- A distinct problem/scenario (not just a variation)
- Complete with Problem, Quick Answer, Detailed Explanation, and Results
- Well-researched with real-world context
- Substantial enough to stand alone (not just a paragraph)

### Examples

**✅ GOOD**: A file with 4 case studies covering:
- Case Study 1: Basic problem (most common scenario)
- Case Study 2: Edge case or variation
- Case Study 3: Scale-related challenge
- Case Study 4: Different approach or trade-off

**❌ BAD**: A file with only 2 case studies (below minimum)

**❌ BAD**: A file with 7 case studies (above maximum, should be split into multiple topics)

---

## RULE 1: LAYMAN EXPLANATION FIRST (MANDATORY)

**Before ANY technical explanation, provide a simple, relatable analogy.**

### Why This Matters

- Not everyone has the same background
- Complex concepts stick better with analogies
- Interviewers love candidates who can explain simply
- It shows you truly understand the topic

### How To Do It

```markdown
## Case Study: Cache Stampede

### Problem

❌ BAD (jumping straight to technical):
"When cache TTL expires, multiple concurrent requests cause thundering herd
to the database, overwhelming it with duplicate queries."

✅ GOOD (layman first, then technical):
"Imagine a popular coffee shop. At 9 AM, their pre-made coffee runs out.
Suddenly, 50 customers all ask for fresh coffee at the same time.
The barista can only make one pot at a time, but now they're trying
to make 50 pots simultaneously. Chaos.

That's cache stampede: When cached data expires, hundreds of requests
all try to rebuild the cache at once, overwhelming your database.

**Technical term**: Thundering herd problem"
```

### Templates for Common Concepts

| Concept | Layman Analogy |
|---------|---------------|
| Load balancing | Traffic cop directing cars to different lanes |
| Caching | Keeping a photocopy on your desk instead of walking to the filing room |
| Sharding | Splitting a phone book into A-M and N-Z volumes |
| Circuit breaker | Electrical fuse that trips to prevent fire |
| Rate limiting | Amusement park limiting how many people enter per hour |
| Eventual consistency | Bank ATM showing slightly outdated balance |
| Idempotency | Elevator button that does the same thing no matter how many times pressed |
| Message queue | Post office holding packages until recipient is ready |

---

## RULE 2: FOCUS ON CONCEPTUAL UNDERSTANDING (NO CODE IMPLEMENTATION)

**Explain concepts, architecture, and decisions - NOT code implementation.**

### Conceptual Explanation Philosophy

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
5. Key insight: Same idempotency_key = same logical operation

**Why this design**:
- Network can fail AFTER payment succeeds but BEFORE response reaches client
- Client retries with same key → Gets cached result, no double charge
- 24-hour TTL: Long enough for retries, short enough to prevent cache bloat

**Architecture decision**: Redis chosen for sub-millisecond lookups and atomic operations
```

### When to Use Diagrams vs Code

**Use diagrams/flows for:**
- Architecture overview
- Data flow
- Request/response flow
- System interactions
- Decision trees

**Avoid code blocks for:**
- Implementation details
- Full function definitions
- Configuration examples
- Setup scripts

**Use conceptual explanations for:**
- How something works
- Why decisions were made
- Trade-offs considered
- Patterns applied

---

## RULE 3: EXPLAIN CONCEPTS CLEARLY

**Always explain WHAT and WHY before diving into HOW (conceptually).**

```markdown
❌ BAD: Jump straight to technical solution without context

"Use exponential backoff with 3 retries for API calls."

✅ GOOD: Explain the problem, then the conceptual solution

When calling external APIs, things will fail. Networks are unreliable.
We need to handle three scenarios:
1. **Timeout**: API is slow or down
2. **Transient error**: 503, 429, network blip (retry these)
3. **Permanent error**: 404, 400 (don't retry these)

**Retry Strategy: Exponential Backoff**

**Approach**:
- Retry transient errors (503, 429, timeouts)
- Don't retry permanent errors (404, 400)
- Use exponential backoff: Wait 1s, then 2s, then 4s between retries
- Maximum 3 attempts (initial + 2 retries)

**Why exponential backoff?**
- Gives the failing service time to recover
- Prevents overwhelming a struggling service
- Reduces unnecessary load during outages

**Architecture decision**: Fail fast (5s timeout) and let retry logic handle recovery
```

---

## RULE 4: USE DIAGRAMS FOR COMPLEX FLOWS

**Any flow with more than 3 steps should have a visual diagram.**

### ASCII Diagrams (Preferred for Markdown)

```markdown
### Request Flow with Rate Limiting

```
Client Request
      │
      ▼
┌─────────────────┐
│ Load Balancer   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────┐
│  Rate Limiter   │────▶│   Redis     │
│  (Check quota)  │◀────│   (counts)  │
└────────┬────────┘     └─────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
  Allow     Reject
    │      (429 Too Many Requests)
    ▼
┌─────────────────┐
│  Application    │
│  Server         │
└─────────────────┘
```
```

### Sequence Diagrams

```markdown
### Payment Flow

```
Customer        Frontend        Backend         Stripe          Database
    │               │               │               │               │
    │──── Click ───▶│               │               │               │
    │   "Pay Now"   │               │               │               │
    │               │               │               │               │
    │               │── POST /pay ─▶│               │               │
    │               │ (idempotency  │               │               │
    │               │   key: abc)   │               │               │
    │               │               │               │               │
    │               │               │── Check ─────▶│               │
    │               │               │   idempotency │               │
    │               │               │◀── Not found ─│               │
    │               │               │               │               │
    │               │               │───────────── Create charge ──▶│
    │               │               │◀──────────── Success ─────────│
    │               │               │               │               │
    │               │               │── Save ──────────────────────▶│
    │               │               │   result      │               │
    │               │               │               │               │
    │               │◀── Success ───│               │               │
    │◀── Show ──────│               │               │               │
    │   receipt     │               │               │               │
```
```

---

## RULE 5: ALWAYS SHOW BEFORE/AFTER

**Concrete metrics make case studies believable and memorable.**

```markdown
❌ BAD: Vague claims
"This significantly improved performance."

✅ GOOD: Specific metrics
| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| P50 latency | 120ms | 15ms | 8x faster |
| P99 latency | 2.5s | 180ms | 14x faster |
| Error rate | 3.2% | 0.05% | 64x fewer errors |
| Database CPU | 85% | 20% | 4x reduction |
| Monthly cost | $45,000 | $12,000 | 73% savings |
```

---

## RULE 6: ADDRESS COMMON MISTAKES

**Every case study should include a "Common Mistakes" section.**

```markdown
### Common Mistakes

1. **Mistake**: Using `SELECT *` in production queries
   **Why it's wrong**: Fetches columns you don't need, wastes bandwidth and memory
   **Instead**: List only needed columns: `SELECT id, name, email FROM users`

2. **Mistake**: No timeout on HTTP calls
   **Why it's wrong**: Hanging connection blocks thread forever
   **Instead**: Always set timeout: `requests.get(url, timeout=5)`

3. **Mistake**: Logging sensitive data
   **Why it's wrong**: Passwords/tokens in logs = security breach
   **Instead**: Redact sensitive fields: `{"email": user.email, "password": "[REDACTED]"}`
```

---

## RULE 7: LINK RELATED CONCEPTS

**Every case study should reference related topics.**

```markdown
### Related Patterns

- **[Circuit Breaker](../01_Distributed_Systems/circuit_breaker_patterns.md)**:
  Stop calling failing services
- **[Retry with Backoff](./api_versioning.md#retry-strategies)**:
  How to retry safely
- **[Idempotency](./idempotency_patterns.md)**:
  Make operations safe to retry

### See Also

- Netflix Tech Blog: [How we handle retries at scale](https://netflixtechblog.com/...)
- Stripe Docs: [Idempotent requests](https://stripe.com/docs/api/idempotent_requests)
```

---

## CHECKLIST FOR EVERY CASE STUDY FILE

Before submitting a case study file, verify:

### File Structure Requirements

- [ ] **Minimum 3 case studies per file** (MANDATORY)
- [ ] **Maximum 5 case studies per file** (to maintain focus and readability)
- [ ] Each case study covers a distinct aspect or scenario of the topic
- [ ] Case studies are well-balanced (not all covering the same problem)
- [ ] Case studies progress logically (basic to advanced, or different scenarios)

### Content Quality

- [ ] Layman explanation provided first (for each case study)
- [ ] Technical details follow the simple explanation
- [ ] Real company reference included (when available)
- [ ] Root cause explained (WHY it happens)
- [ ] Solution approach explained conceptually (not code)
- [ ] Architecture and design decisions clearly explained
- [ ] Before/after metrics included
- [ ] Common mistakes section added (at file level, not per case study)
- [ ] Related patterns linked
- [ ] Research resources file created

### Understanding Focus

- [ ] Concepts explained clearly and concisely
- [ ] No unnecessary code examples (focus on understanding)
- [ ] Architecture diagrams/flows included where helpful
- [ ] Design decisions and trade-offs explained
- [ ] Key insights and lessons learned highlighted
- [ ] Crisp and concise (no fluff)

### Research Quality

- [ ] All sources properly cited
- [ ] Resources file includes:
  - Engineering blog posts (with URLs and dates)
  - Conference talks (with links)
  - Technical papers (with citations)
  - Documentation references
  - Video presentations
- [ ] Sources are current (2024-2026 where applicable)
- [ ] Multiple perspectives included when available

### Formatting

- [ ] Diagrams for flows with 3+ steps
- [ ] Tables for comparisons
- [ ] Consistent header hierarchy
- [ ] No walls of text (break into sections)
- [ ] Crisp and concise writing

---

## QUICK REFERENCE: Conceptual Explanation Templates

### Architecture Explanation

```markdown
**Architecture Decision**: [What was chosen]

**Why this approach**:
- Reason 1: [Explanation]
- Reason 2: [Explanation]

**How it works conceptually**:
1. Step 1: [What happens]
2. Step 2: [What happens]
3. Step 3: [What happens]

**Key insight**: [One sentence takeaway]
```

### Design Decision Template

```markdown
**Decision**: [What was chosen]

**Alternatives considered**:
- Option A: [Why not chosen]
- Option B: [Why not chosen]

**Why this decision**:
- [Primary reason]
- [Secondary reason]

**Trade-offs**:
- Pros: [Benefits]
- Cons: [Drawbacks]
```

### Conceptual Flow Template

```markdown
**Flow**:
```
Step 1 → Step 2 → Step 3
  ↓        ↓        ↓
Result 1  Result 2 Result 3
```

**Why this flow**:
[Explanation of the sequence and reasoning]
```

---

**Remember**: The goal is that someone reading the case study should understand:
1. **WHAT** the problem is
2. **WHY** it happens
3. **WHAT** approach was taken to solve it
4. **WHY** that approach was chosen
5. **WHAT** the results were

Focus on crisp, concise understanding - not code implementation.
