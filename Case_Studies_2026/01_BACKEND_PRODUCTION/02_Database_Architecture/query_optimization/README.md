# Query Optimization

How to identify and fix slow database queries, improving performance by orders of magnitude.

---

## Case Study 1: Slow Query Due to Missing Index

### Problem

A query that should be fast is taking 5+ seconds. The query looks simple: `SELECT * FROM users WHERE email = 'user@example.com'`. But the database is doing a full table scan, checking every row.

**Context**: Most common database performance issue

**In simple terms**: Imagine finding a book in a library by checking every single book one by one, instead of using the card catalog. That's a full table scan: the database checks every row instead of using an index to jump directly to the right row.

### Quick Answer

Add an index on the `email` column. The database can use the index to find the row instantly instead of scanning the entire table.

### Detailed Explanation

#### Why This Happens (Root Cause)

**Query without index:**
```sql
SELECT * FROM users WHERE email = 'user@example.com';
```

**What database does (full table scan):**
1. Start at first row
2. Check if email matches
3. Move to next row
4. Repeat for all rows
5. Return matching row

**Performance:**
- 1 million rows = 1 million comparisons
- Time: O(n) - linear with table size
- Slow for large tables

**With index:**
```sql
CREATE INDEX idx_users_email ON users(email);
```

**What database does (index lookup):**
1. Look up email in index (B-tree)
2. Index points directly to row
3. Fetch row
4. Return result

**Performance:**
- Index lookup: O(log n) - logarithmic
- Time: Milliseconds instead of seconds
- Fast regardless of table size

#### The Solution Approach

**High-level strategy:**
- Identify slow queries (EXPLAIN, query logs)
- Add indexes on WHERE clause columns
- Verify index is used (EXPLAIN again)
- Monitor query performance

**Index Selection:**

**Single column index:**
```sql
CREATE INDEX idx_users_email ON users(email);
```
- Good for: Single column WHERE clauses
- Example: `WHERE email = '...'`

**Composite index:**
```sql
CREATE INDEX idx_users_name_email ON users(name, email);
```
- Good for: Multiple columns in WHERE
- Example: `WHERE name = '...' AND email = '...'`
- Order matters: Put most selective column first

**Key Design Decisions:**
- **Index on WHERE columns**: Enables fast filtering
- **Index on JOIN columns**: Enables fast joins
- **Index on ORDER BY columns**: Enables fast sorting
- **Don't over-index**: Each index slows writes, uses storage

#### Production Results

| Metric | Before (No Index) | After (With Index) | Improvement |
|--------|------------------|-------------------|-------------|
| Query latency | 5.2s | 12ms | 433x faster |
| Database CPU | 85% | 15% | 5.7x reduction |
| Rows examined | 1,000,000 | 1 | 1Mx reduction |

**Key insight**: Proper indexing is the most impactful query optimization. A single index can improve performance by orders of magnitude.

**Lessons Learned**:
- Always index columns in WHERE clauses
- Use EXPLAIN to verify index usage
- Monitor slow query logs regularly
- Don't over-index (balance read vs write performance)

---

## Case Study 2: The LOWER() Function Trap

### Problem

Query is slow: `SELECT * FROM users WHERE LOWER(email) = 'user@example.com'`. You added an index on `email`, but the query is still slow. The index isn't being used.

**Context**: Common mistake - function on indexed column prevents index usage

**In simple terms**: Imagine the library card catalog is organized by exact book titles. But you're searching for books by converting titles to lowercase first. The catalog is useless because it's organized differently than how you're searching.

Using functions on indexed columns prevents the database from using the index.

### Quick Answer

Don't use functions on indexed columns. Store data in the format you'll query (lowercase), or use a functional index. Alternatively, restructure the query to avoid the function.

### Detailed Explanation

#### Why This Happens

**Query with function:**
```sql
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
```

**Problem:**
- Index is on `email` (original case)
- Query uses `LOWER(email)` (transformed)
- Database can't use index (different values)
- Must do full table scan

**Solution 1: Store Lowercase**

**How it works:**
1. Store email in lowercase: `user@example.com`
2. Query without function: `WHERE email = 'user@example.com'`
3. Index can be used

**Solution 2: Functional Index**

**How it works:**
```sql
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```
1. Create index on function result
2. Query can use functional index
3. Database matches `LOWER(email)` with index

**Solution 3: Restructure Query**

**How it works:**
```sql
-- Instead of:
WHERE LOWER(email) = 'user@example.com'

-- Use:
WHERE email = 'user@example.com' OR email = 'User@Example.com'
-- Or better: Store lowercase, query lowercase
```

#### Production Results

| Metric | Before (LOWER()) | After (No Function) | Improvement |
|--------|------------------|-------------------|-------------|
| Query latency | 3.5s | 15ms | 233x faster |
| Index usage | No | Yes | Index now used |
| Rows examined | 500,000 | 1 | 500Kx reduction |

**Key insight**: Functions on indexed columns prevent index usage. Store data in queryable format or use functional indexes.

**Lessons Learned**:
- Avoid functions on indexed columns
- Store data in format you'll query
- Use functional indexes if functions are necessary
- Monitor EXPLAIN output to verify index usage

---

## Case Study 3: N+1 Query Problem

### Problem

Your API endpoint returns a list of users with their orders. The query is fast for 10 users, but slow for 1000 users. You're making 1 query for users, then 1000 queries for orders (one per user). That's 1001 queries total.

**Context**: Most common performance issue in ORMs and APIs

**In simple terms**: Imagine asking a librarian for 1000 books, but instead of getting them all at once, the librarian goes to the shelf 1000 times (once per book). That's the N+1 problem: 1 query for the list, then N queries for related data.

### Quick Answer

Use eager loading or JOIN queries to fetch all related data in a single query (or few queries). Instead of 1 + N queries, use 1-2 queries total.

### Detailed Explanation

#### Why This Happens (Root Cause)

**N+1 query pattern:**
```sql
-- Query 1: Get users
SELECT * FROM users LIMIT 1000;

-- Query 2-1001: Get orders for each user (N queries)
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
...
SELECT * FROM orders WHERE user_id = 1000;
```

**Total: 1 + 1000 = 1001 queries**

**Root cause**: ORM or application code loads related data lazily (on-demand). Each access triggers a new query.

**Performance:**
- 1001 queries × 5ms = 5 seconds
- Network overhead: 1001 round trips
- Database connection overhead

#### The Solution Approach

**High-level strategy:**
- Eager load related data (fetch in initial query)
- Use JOIN to get all data in one query
- Batch load related data (fetch all at once)
- Use data loader pattern (batch requests)

**Solution 1: JOIN Query**

**How it works conceptually:**
```sql
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.id IN (1, 2, ..., 1000);
```

**Benefits:**
- Single query (fast)
- All data fetched at once
- Database optimizes join

**Trade-offs:**
- Returns duplicate user data (one row per order)
- Application must group results
- Can be large result set

**Solution 2: Eager Loading (Two Queries)**

**How it works conceptually:**
```sql
-- Query 1: Get users
SELECT * FROM users LIMIT 1000;

-- Query 2: Get all orders for those users
SELECT * FROM orders
WHERE user_id IN (1, 2, ..., 1000);
```

**Benefits:**
- Only 2 queries (much better than 1001)
- No duplicate data
- Application groups results easily

**Solution 3: Data Loader Pattern**

**How it works conceptually:**
1. Collect all user IDs that need orders
2. Batch load all orders in single query
3. Group orders by user_id in application
4. Return grouped data

**Benefits:**
- Works with GraphQL and similar patterns
- Automatic batching
- Prevents N+1 in nested queries

#### Production Results

| Metric | Before (N+1) | After (Eager Load) | Improvement |
|--------|--------------|-------------------|-------------|
| Query count | 1001 | 2 | 500x reduction |
| Query latency | 5.2s | 120ms | 43x faster |
| Database load | 85% | 20% | 4.25x reduction |
| Network round trips | 1001 | 2 | 500x reduction |

**Key insight**: N+1 queries are a common performance killer. Eager loading or JOINs reduce queries by orders of magnitude.

**Lessons Learned**:
- Always eager load related data (don't lazy load)
- Use JOIN or IN queries to batch fetch
- Monitor query counts (should be constant, not linear with results)
- Use data loader pattern for nested queries
- Profile queries to identify N+1 patterns

---

## Case Study 4: Missing Composite Index

### Problem

Query is slow: `SELECT * FROM orders WHERE user_id = 123 AND status = 'pending' ORDER BY created_at DESC`. You have indexes on `user_id` and `status` separately, but the query is still slow.

**Context**: Common mistake - single column indexes don't help multi-column queries

**In simple terms**: Imagine a library with two separate card catalogs - one by author, one by title. To find a book by a specific author with a specific title, you'd need to check both catalogs and find the intersection. That's inefficient.

Composite indexes are like a combined catalog (author + title) - you can find the book directly.

### Quick Answer

Create a composite index on all columns used in WHERE and ORDER BY: `CREATE INDEX idx_orders_user_status_created ON orders(user_id, status, created_at DESC)`.

### Detailed Explanation

#### Why This Happens

**Query:**
```sql
SELECT * FROM orders
WHERE user_id = 123 AND status = 'pending'
ORDER BY created_at DESC;
```

**With single column indexes:**
- Index on `user_id`: Returns all orders for user 123
- Index on `status`: Returns all pending orders
- Database must intersect results (slow)
- Then sort by `created_at` (slow)

**Root cause**: Single column indexes can't efficiently handle multi-column queries. Database must use one index, then filter by other columns.

#### The Solution Approach

**High-level strategy:**
- Create composite index on all query columns
- Order matters: Most selective column first
- Include ORDER BY columns in index

**Composite Index Design:**

**Index:**
```sql
CREATE INDEX idx_orders_user_status_created
ON orders(user_id, status, created_at DESC);
```

**Why this works:**
- Index organized by (user_id, status, created_at)
- Query can use index for filtering AND sorting
- Database jumps directly to matching rows
- Already sorted by created_at (no sort needed)

**Column Order Matters:**

**Good order:**
```sql
(user_id, status, created_at)
```
- `user_id` most selective (narrows down quickly)
- `status` next (further narrows)
- `created_at` last (for sorting)

**Bad order:**
```sql
(status, user_id, created_at)
```
- `status` less selective (many pending orders)
- Must scan more rows before filtering by user_id

**Key Design Decisions:**
- **Most selective first**: Reduces rows quickly
- **Include ORDER BY columns**: Enables index-only sort
- **Covering index**: Include SELECT columns if possible (avoids table lookup)

#### Production Results

| Metric | Before (Single Indexes) | After (Composite) | Improvement |
|--------|------------------------|-------------------|-------------|
| Query latency | 850ms | 15ms | 57x faster |
| Rows examined | 50,000 | 10 | 5000x reduction |
| Sort operation | Yes (slow) | No (index sorted) | Eliminated |
| Index usage | Partial | Full | Complete usage |

**Key insight**: Composite indexes are essential for multi-column queries. The column order matters - most selective column first.

**Lessons Learned**:
- Create composite indexes for multi-column queries
- Order columns by selectivity (most selective first)
- Include ORDER BY columns in index
- Consider covering indexes (include SELECT columns)
- Monitor index usage with EXPLAIN

---

## Case Study 5: Slow Query Due to Large Result Set

### Problem

Query returns 100,000 rows: `SELECT * FROM logs WHERE date = '2024-01-15'`. The query itself is fast (uses index), but transferring 100,000 rows over the network takes 30+ seconds. Application is slow and memory usage is high.

**Context**: Common issue - query is optimized but result set is too large

**In simple terms**: Imagine asking a librarian for every book published in 2024. The librarian finds them quickly (good index), but carrying 10,000 books takes forever. The problem isn't finding the books, it's transferring them.

Large result sets are the same: query is fast, but network transfer and memory usage are the bottlenecks.

### Quick Answer

Limit result sets (pagination), select only needed columns, use streaming for large datasets, or aggregate data instead of returning raw rows.

### Detailed Explanation

#### Why This Happens

**Query:**
```sql
SELECT * FROM logs WHERE date = '2024-01-15';
-- Returns 100,000 rows
```

**Bottlenecks:**
1. **Network transfer**: 100,000 rows × 1KB = 100MB transfer
2. **Memory usage**: Application loads all rows into memory
3. **Processing time**: Application processes all rows
4. **Database connection**: Kept open during transfer

**Root cause**: Query returns too much data. Even with perfect index, large result sets are slow to transfer and process.

#### The Solution Approach

**High-level strategy:**
- Limit result sets (pagination)
- Select only needed columns (not SELECT *)
- Use streaming for large datasets
- Aggregate data when possible
- Use cursors for large exports

**Solution 1: Pagination**

**How it works conceptually:**
```sql
-- Instead of:
SELECT * FROM logs WHERE date = '2024-01-15';

-- Use:
SELECT * FROM logs
WHERE date = '2024-01-15'
ORDER BY id
LIMIT 100 OFFSET 0;
```

**Benefits:**
- Small result sets (fast transfer)
- Lower memory usage
- Better user experience (progressive loading)

**Solution 2: Select Only Needed Columns**

**How it works conceptually:**
```sql
-- Instead of:
SELECT * FROM logs WHERE date = '2024-01-15';

-- Use:
SELECT id, message, timestamp
FROM logs
WHERE date = '2024-01-15';
```

**Benefits:**
- Less data transferred
- Faster queries (smaller result set)
- Lower memory usage

**Solution 3: Aggregation**

**How it works conceptually:**
```sql
-- Instead of returning all rows:
SELECT * FROM logs WHERE date = '2024-01-15';

-- Return aggregated data:
SELECT
  COUNT(*) as total_logs,
  COUNT(DISTINCT user_id) as unique_users,
  AVG(response_time) as avg_response_time
FROM logs
WHERE date = '2024-01-15';
```

**Benefits:**
- Single row result (very fast)
- No network transfer overhead
- Database does aggregation (efficient)

**Solution 4: Streaming**

**How it works conceptually:**
1. Use database cursor (streaming result set)
2. Process rows one at a time
3. Don't load all rows into memory
4. Write to file or process incrementally

**Benefits:**
- Constant memory usage
- Can handle very large datasets
- Good for exports and reports

#### Production Results

| Metric | Before (100K rows) | After (Pagination) | Improvement |
|--------|-------------------|-------------------|-------------|
| Transfer time | 30s | 200ms | 150x faster |
| Memory usage | 500MB | 5MB | 100x reduction |
| Query latency | 30s | 200ms | 150x faster |
| User experience | Poor (slow) | Good (fast) | Much better |

**Key insight**: Large result sets are a bottleneck even with perfect queries. Limit, paginate, or aggregate to reduce data transfer.

**Lessons Learned**:
- Always limit result sets (pagination)
- Select only needed columns (not SELECT *)
- Use aggregation when possible (single row result)
- Stream large datasets (don't load into memory)
- Monitor result set sizes

---

## Common Mistakes

1. **Mistake**: No indexes on WHERE columns
   **Why it's wrong**: Full table scans, very slow queries
   **Instead**: Always index columns used in WHERE clauses

2. **Mistake**: Functions on indexed columns
   **Why it's wrong**: Prevents index usage, forces full table scan
   **Instead**: Store data in queryable format, avoid functions

3. **Mistake**: Too many indexes
   **Why it's wrong**: Slows writes, uses storage, maintenance overhead
   **Instead**: Index only columns used in queries, monitor index usage

4. **Mistake**: Not using EXPLAIN
   **Why it's wrong**: Can't verify index usage, guessing at optimization
   **Instead**: Always use EXPLAIN to verify query plans

---

## Related Patterns

- **[Sharding Strategies](./sharding_strategies.md)** - Scaling beyond single database
- **[Database Scaling](./database_scaling.md)** - Database infrastructure scaling

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for complete list of research sources.
