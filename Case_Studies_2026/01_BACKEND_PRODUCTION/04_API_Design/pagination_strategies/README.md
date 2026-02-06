# Pagination Strategies

How to efficiently paginate through large datasets in APIs, handling millions of records with consistent performance.

---

## Case Study 1: Offset Pagination Performance Degradation

### Problem

Your API uses offset-based pagination (`?page=1&limit=100`). For the first few pages, it's fast. But as users paginate deeper (page 100, page 1000), queries become slower. At page 10,000, queries take 30+ seconds and sometimes timeout.

**Context**: Common issue with offset pagination - affects any API with large datasets

**In simple terms**: Imagine a library with millions of books. To get book #10,000, you have to count through the first 9,999 books. The deeper you go, the longer it takes. Offset pagination works the same way: to get page 100, the database must skip the first 9,900 records.

### Quick Answer

Use cursor-based pagination: instead of skipping records (offset), use a cursor (pointer) to the last seen record. Database can use index to jump directly to the cursor position, making all pages equally fast.

### Detailed Explanation

#### Why This Happens (Root Cause)

**Offset pagination query:**
```sql
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 100 OFFSET 9900;
```

**What database does:**
1. Sort all records by `created_at DESC`
2. Skip first 9,900 records (count through them)
3. Return next 100 records

**Problem:**
- Database must process all skipped records
- Deeper pages = more records to skip = slower queries
- Can't use index efficiently for skipping

**Performance degradation:**
```
Page 1:   OFFSET 0    → 10ms   (fast)
Page 10:  OFFSET 900  → 50ms   (acceptable)
Page 100: OFFSET 9900 → 500ms  (slow)
Page 1000: OFFSET 99900 → 5s    (very slow)
```

**Root cause**: OFFSET requires database to count through skipped records. This is O(n) complexity - linear with offset size.

#### The Solution Approach

**High-level strategy:**
- Use cursor-based pagination instead of offset
- Cursor = pointer to last seen record (usually ID or timestamp)
- Database uses index to jump directly to cursor position
- All pages equally fast (O(log n) with index)

**Architecture Decision: Cursor-Based Pagination**

**Why cursor-based?**
- Constant performance (all pages equally fast)
- Works with indexes efficiently
- Handles new data gracefully (no duplicate/missing records)
- No performance degradation at depth

**How it works conceptually:**
1. Client requests first page: `GET /posts?limit=100`
2. Server returns 100 posts + cursor: `{"cursor": "2024-01-15T10:30:00Z:post_123"}`
3. Client requests next page: `GET /posts?cursor=2024-01-15T10:30:00Z:post_123&limit=100`
4. Server queries: `WHERE created_at < '2024-01-15T10:30:00Z' AND id > 'post_123'`
5. Database uses index to jump directly to cursor position
6. Returns next 100 posts + new cursor

**Cursor format:**
- **Timestamp + ID**: `"2024-01-15T10:30:00Z:post_123"`
  - Timestamp for ordering
  - ID for tie-breaking (multiple posts at same timestamp)
- **Base64 encoded**: Opaque to client, can contain multiple fields

**Query pattern:**
```sql
-- First page
SELECT * FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 100;

-- Next page (with cursor)
SELECT * FROM posts
WHERE (created_at < '2024-01-15T10:30:00Z'
   OR (created_at = '2024-01-15T10:30:00Z' AND id > 'post_123'))
ORDER BY created_at DESC, id DESC
LIMIT 100;
```

**Key Design Decisions:**
- **Composite cursor (timestamp + ID)**: Handles ties (multiple records with same timestamp)
- **Opaque cursor**: Client doesn't need to understand structure
- **Index on (created_at, id)**: Enables fast cursor lookups
- **Consistent ordering**: Always use same ORDER BY clause

#### Production Results

| Metric | Before (Offset) | After (Cursor) | Improvement |
|--------|----------------|----------------|-------------|
| Page 1 latency | 10ms | 10ms | Same |
| Page 100 latency | 500ms | 12ms | 42x faster |
| Page 1000 latency | 5s | 12ms | 417x faster |
| Page 10000 latency | Timeout | 12ms | Infinite improvement |
| Database CPU (deep pages) | 80% | 5% | 16x reduction |

**Key insight**: Cursor-based pagination provides constant performance regardless of page depth. The complexity is worth the massive performance improvement for deep pagination.

**Lessons Learned**:
- Always use cursor-based pagination for large datasets
- Composite cursor (timestamp + ID) handles ties correctly
- Index on cursor fields is critical for performance
- Make cursor opaque to clients (base64 encode)
- Document cursor format and usage clearly

---

## Case Study 2: Handling New Data During Pagination

### Problem

User starts paginating through posts. They're on page 5 when a new post is created. With offset pagination, they might see the new post on page 5 (duplicate) or miss it entirely (inconsistent). How do you handle new data appearing during pagination?

**Context**: Common issue with real-time data - affects social feeds, activity streams

**In simple terms**: Imagine reading a newspaper page by page. While you're reading, a new article is added. With offset pagination, you might see the new article twice (once on current page, once when you turn the page) or miss it entirely. Cursor-based pagination solves this.

### Quick Answer

Cursor-based pagination naturally handles new data: cursor points to a specific record, not a position. New records before the cursor don't affect pagination. New records after the cursor appear in later pages (correct behavior).

### Detailed Explanation

#### Why This Matters

**Scenario with offset pagination:**
```
Time T0: User requests page 1 (posts 1-100)
Time T1: New post created (inserted at position 50)
Time T2: User requests page 2 (OFFSET 100, LIMIT 100)
Result: Post #50 appears on both page 1 and page 2 (duplicate)
```

**Or:**
```
Time T0: User requests page 1 (posts 1-100)
Time T1: New post created (inserted at position 1)
Time T2: User requests page 2 (OFFSET 100, LIMIT 100)
Result: Post #100 is now #101, user misses it (inconsistent)
```

**Root cause**: Offset is position-based, not record-based. New data shifts positions, causing duplicates or missing records.

#### The Solution Approach

**How cursor-based pagination handles it:**

**Scenario:**
```
Time T0: User requests page 1
         Returns posts with cursor: "2024-01-15T10:00:00Z:post_100"
Time T1: New post created (timestamp: 2024-01-15T09:00:00Z)
         This post is BEFORE the cursor (older timestamp)
Time T2: User requests page 2 with cursor
         Query: WHERE created_at < '2024-01-15T10:00:00Z'
         New post appears (correct - it's older than cursor)
         No duplicates, no missing records
```

**Key insight**: Cursor points to a specific record (timestamp + ID), not a position. New records are inserted based on their timestamp, not affecting existing cursors.

**Benefits:**
- **No duplicates**: Cursor ensures each record appears once
- **No missing records**: New records appear in correct position
- **Consistent view**: User sees consistent snapshot (records before cursor)
- **Real-time updates**: New records appear in later pages naturally

**Trade-off:**
- User might not see very new records until they refresh (by design)
- This is acceptable for most use cases (eventual consistency)

#### Production Results

| Metric | Before (Offset) | After (Cursor) | Improvement |
|--------|----------------|----------------|-------------|
| Duplicate records | 2% of pagination sessions | 0% | 100% elimination |
| Missing records | 1.5% of pagination sessions | 0% | 100% elimination |
| User complaints (inconsistent data) | 25/day | 0/day | Complete elimination |
| Data consistency | 96.5% | 100% | Perfect consistency |

**Key insight**: Cursor-based pagination provides consistent, duplicate-free pagination even with real-time data updates. The slight delay in seeing new records is acceptable for most use cases.

**Lessons Learned**:
- Cursor-based pagination naturally handles new data
- No special logic needed for real-time updates
- Trade-off: New records might not appear immediately (acceptable)
- Document this behavior to users if needed
- Consider "refresh" option for users who want latest data

---

## Case Study 3: Cursor vs Offset Trade-offs

### Problem

You need to choose between cursor-based and offset-based pagination. Each has different characteristics. When should you use which?

**Context**: Architectural decision - affects API design and performance

**In simple terms**:
- **Offset**: Like page numbers in a book - easy to understand, but slow for deep pages
- **Cursor**: Like a bookmark - fast for any page, but can't jump to arbitrary pages

### Quick Answer

Use cursor-based pagination for large datasets, real-time data, or when users paginate deeply. Use offset-based pagination only for small datasets (< 10,000 records) or when you need random page access.

### Detailed Explanation

#### Comparison

| Feature | Offset Pagination | Cursor Pagination |
|---------|------------------|-------------------|
| Performance (shallow) | Fast | Fast |
| Performance (deep) | Slow (O(n)) | Fast (O(log n)) |
| Random page access | Yes (page 5, page 100) | No (sequential only) |
| New data handling | Duplicates/missing | Consistent |
| Implementation complexity | Simple | Medium |
| Client complexity | Simple | Medium |
| Index requirements | None | Required |

#### When to Use Offset

**Good for:**
- Small datasets (< 10,000 records)
- Admin interfaces (need random page access)
- Static data (rarely changes)
- Simple use cases (no deep pagination)

**Example:**
```
Admin panel showing users
- Total: 5,000 users
- Need to jump to page 50 directly
- Data rarely changes
→ Use offset pagination
```

#### When to Use Cursor

**Good for:**
- Large datasets (> 100,000 records)
- Real-time data (feeds, activity streams)
- Deep pagination (users go many pages deep)
- High-traffic APIs (performance critical)

**Example:**
```
Social media feed
- Millions of posts
- Users scroll continuously
- New posts added constantly
→ Use cursor pagination
```

#### Hybrid Approach

Some APIs support both:

**Strategy:**
- Default to cursor-based (better performance)
- Support offset for admin/backward compatibility
- Document performance implications
- Rate limit offset-based requests (prevent abuse)

**API Design:**
```
GET /posts?cursor=abc123&limit=100  (preferred)
GET /posts?page=5&limit=100         (supported, but slower)
```

#### Production Results

| Use Case | Offset | Cursor | Recommendation |
|----------|--------|--------|----------------|
| Small dataset (< 10K) | ✅ Good | ✅ Good | Either works |
| Large dataset (> 100K) | ❌ Slow | ✅ Fast | Use cursor |
| Real-time data | ❌ Inconsistent | ✅ Consistent | Use cursor |
| Random page access | ✅ Yes | ❌ No | Use offset |
| Deep pagination | ❌ Very slow | ✅ Fast | Use cursor |

**Key insight**: Choose pagination strategy based on dataset size, access patterns, and data update frequency. For most production APIs, cursor-based is the better choice.

**Lessons Learned**:
- Use cursor for large datasets and real-time data
- Use offset only for small, static datasets
- Consider hybrid approach for flexibility
- Document performance implications clearly
- Monitor pagination performance and adjust strategy

---

## Common Mistakes

1. **Mistake**: Using offset for large datasets
   **Why it's wrong**: Performance degrades linearly with page depth, becomes unusable
   **Instead**: Use cursor-based pagination for datasets > 10,000 records

2. **Mistake**: Cursor without proper index
   **Why it's wrong**: Cursor lookups are slow without index, negates benefits
   **Instead**: Create composite index on cursor fields (timestamp, id)

3. **Mistake**: Exposing cursor structure to clients
   **Why it's wrong**: Clients might manipulate cursor, breaking pagination
   **Instead**: Make cursor opaque (base64 encode, don't document structure)

4. **Mistake**: Not handling ties in cursor
   **Why it's wrong**: Multiple records with same timestamp cause inconsistent ordering
   **Instead**: Use composite cursor (timestamp + ID) for tie-breaking

5. **Mistake**: Changing sort order between pages
   **Why it's wrong**: Cursor becomes invalid, pagination breaks
   **Instead**: Always use same ORDER BY clause, document it clearly

6. **Mistake**: No limit on page size
   **Why it's wrong**: Clients can request huge pages, overwhelming database
   **Instead**: Enforce maximum limit (e.g., max 100 records per page)

---

## Related Patterns

- **[Query Optimization](./query_optimization.md)** - Database query performance
- **[Rate Limiting](../rate_limiting/README.md)** - Prevent pagination abuse
- **[API Design](./../README.md)** - General API design patterns

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for complete list of:
- Engineering blog posts (Twitter, Facebook, Stripe)
- Conference talks (QCon, AWS re:Invent)
- Technical papers on pagination
- Documentation references
