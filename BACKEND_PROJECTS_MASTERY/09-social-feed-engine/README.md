# Project 09: Social Media Feed Engine

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language

## What You're Building

A scalable feed system like Twitter/X, Instagram, or LinkedIn that generates personalized feeds for millions of users in real-time.

## Key Concepts

- Fan-out on write vs fan-out on read
- Feed generation strategies
- Pagination (cursor-based)
- Graph database (followers/following)
- Caching strategies
- Hot user problem
- Real-time updates

## The Fan-Out Problem

### Fan-Out on Write (Twitter approach for most users)

```
User A posts:
1. Write post to database
2. Fan out to all followers' timelines
   - User B's timeline: add post
   - User C's timeline: add post
   - ... (all N followers)

Read is fast: Just read from pre-computed timeline
Write is slow: N writes per post
```

### Fan-Out on Read (Twitter approach for celebrities)

```
User A (celebrity with 1M followers) posts:
1. Write post to database only
2. Don't fan out (too expensive!)

When follower reads feed:
1. Fetch their timeline (regular users)
2. Fetch recent posts from celebrities they follow
3. Merge and sort
4. Cache result

Write is fast: 1 write
Read is slower: Must merge multiple sources
```

### Hybrid Approach (Best of Both)

```
- Regular users: Fan-out on write
- Celebrities: Fan-out on read
- Threshold: Fan out if followers < 10,000
```

## Architecture

```
[Post Creation] → [Fan-Out Service] → [Timeline Service] → [Cache]
                          ↓
                  [Graph Database]
                   (Relationships)
```

## Database Schema

### Posts (PostgreSQL)

```sql
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    media_urls TEXT[],
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_user_created (user_id, created_at DESC)
);
```

### Timelines (Redis)

```
Key: timeline:{user_id}
Type: Sorted Set
Score: timestamp
Members: post_ids

ZADD timeline:123 1708084800 post_456
ZADD timeline:123 1708084900 post_789

# Get latest 20 posts
ZREVRANGE timeline:123 0 19
```

### Relationships (Neo4j)

```cypher
// Create follow relationship
CREATE (u1:User {id: 123})-[:FOLLOWS]->(u2:User {id: 456})

// Get followers
MATCH (follower:User)-[:FOLLOWS]->(user:User {id: 123})
RETURN follower

// Get feed sources (users you follow)
MATCH (user:User {id: 123})-[:FOLLOWS]->(following:User)
RETURN following
```

## Implementation

### Fan-Out Service

```typescript
async function handleNewPost(post: Post) {
  const user = await getUser(post.user_id);

  if (user.followers_count < 10000) {
    // Fan-out on write
    const followers = await getFollowerIds(user.id);
    await Promise.all(
      followers.map((followerId) =>
        addToTimeline(followerId, post.id)
      )
    );
  } else {
    // Fan-out on read (do nothing here)
    // Followers will fetch on read
  }
}

async function addToTimeline(
  userId: number,
  postId: number
) {
  const score = Date.now();
  await redis.zadd(
    `timeline:${userId}`,
    score,
    postId.toString()
  );

  // Keep timeline size manageable
  await redis.zremrangebyrank(
    `timeline:${userId}`,
    0,
    -1001 // Keep last 1000 posts
  );
}
```

### Feed Generation

```typescript
async function getFeed(userId: number, cursor?: string) {
  // Get pre-computed timeline
  const timeline = await redis.zrevrangebyscore(
    `timeline:${userId}`,
    cursor || "+inf",
    "-inf",
    "LIMIT",
    0,
    20
  );

  // Get celebrity posts
  const celebrities = await getCelebritiesFollowed(userId);
  const celebrityPosts = await getRecentPosts(celebrities, 20);

  // Merge and sort
  const allPosts = [...timeline, ...celebrityPosts].sort(
    (a, b) => b.created_at - a.created_at
  );

  // Hydrate post details
  const posts = await getPostsByIds(allPosts.map((p) => p.id));

  return {
    posts: posts.slice(0, 20),
    next_cursor: posts[19]?.created_at
  };
}
```

## Cursor-Based Pagination

```typescript
// Good: Cursor-based (consistent, fast)
GET /feed?cursor=2026-02-16T10:30:00Z&limit=20

async function getFeed(cursor: string, limit: number) {
  return await db.query(`
    SELECT * FROM posts
    WHERE created_at < $1
    ORDER BY created_at DESC
    LIMIT $2
  `, [cursor, limit]);
}

// Bad: Offset-based (slow for large offsets)
GET /feed?page=1000&limit=20 // Skip 20,000 rows!
```

## Success Criteria

✅ Generate feed in < 500ms
✅ Handle 100K+ posts/sec
✅ Efficient pagination
✅ Real-time updates
✅ Celebrity scaling

---

**Next**: Project 10 (AI Search & RAG)
