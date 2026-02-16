# Project 07: GraphQL Federation Gateway

## Difficulty: Advanced
**Estimated Time**: 3-4 weeks

## Problem Statement

Design a GraphQL Federation Gateway that unifies multiple microservices into a single GraphQL API, solves the N+1 query problem, implements DataLoader for batching, and handles schema stitching across services.

**Real-World Example**: Netflix has 100+ microservices. Instead of clients calling each service separately, a GraphQL gateway provides a unified API where one query fetches data from multiple services efficiently.

## Functional Requirements

### Core Features:
1. **Schema Federation**: Combine schemas from multiple services
2. **Query Routing**: Route parts of query to appropriate services
3. **DataLoader**: Batch and cache requests to prevent N+1 problem
4. **Authentication**: Handle auth once at gateway
5. **Caching**: Cache frequent queries
6. **Error Handling**: Aggregate errors from multiple services

## The N+1 Problem

### Problem Statement:

```graphql
# Query: Get posts with author details
query {
  posts {          # 1 query
    id
    title
    author {       # N queries (one per post)
      name
    }
  }
}

# Without batching:
1. SELECT * FROM posts;  (returns 100 posts)
2. SELECT * FROM users WHERE id=1;  (post 1's author)
3. SELECT * FROM users WHERE id=2;  (post 2's author)
...
101. SELECT * FROM users WHERE id=100;

Total: 101 queries! (1 + 100)
```

### Solution: DataLoader

```graphql
# With DataLoader:
1. SELECT * FROM posts;  (returns 100 posts)
2. SELECT * FROM users WHERE id IN (1,2,3,...,100);  (batch!)

Total: 2 queries!
```

## Architecture

```
┌─────────────────────────────────────────────────┐
│              GraphQL Gateway                    │
│  ┌──────────────────────────────────────────┐  │
│  │        Unified Schema                    │  │
│  │  type Query {                            │  │
│  │    user(id: ID): User                    │  │
│  │    posts: [Post]                         │  │
│  │  }                                       │  │
│  └──────────────────────────────────────────┘  │
│              ▼Query Planner▼                    │
│  ┌─────────┬──────────┬─────────────┐         │
└──┤         │          │             │──────────┘
   │         │          │             │
   ▼         ▼          ▼             ▼
┌────────┐ ┌──────┐ ┌────────┐ ┌──────────┐
│ Users  │ │Posts │ │Comments│ │ Products │
│Service │ │Service│ │Service│ │ Service  │
└────────┘ └──────┘ └────────┘ └──────────┘
```

## Schema Federation

### Subgraph Schemas:

**Users Service:**
```graphql
# users-service/schema.graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  createdAt: DateTime!
}

extend type Query {
  user(id: ID!): User
  users: [User!]!
}
```

**Posts Service:**
```graphql
# posts-service/schema.graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  content: String!
  authorId: ID!
  author: User @external
  comments: [Comment!]!
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post!]!
}

extend type Query {
  post(id: ID!): Post
  posts: [Post!]!
}
```

**Comments Service:**
```graphql
# comments-service/schema.graphql
type Comment @key(fields: "id") {
  id: ID!
  text: String!
  postId: ID!
  authorId: ID!
  author: User @external
  post: Post @external
}

extend type Post @key(fields: "id") {
  id: ID! @external
  comments: [Comment!]!
}

extend type Query {
  comment(id: ID!): Comment
}
```

### Federated Schema (Combined):

```graphql
# Unified schema at gateway
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!      # From Posts Service
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!        # From Users Service
  comments: [Comment!]! # From Comments Service
}

type Comment {
  id: ID!
  text: String!
  author: User!        # From Users Service
  post: Post!          # From Posts Service
}

type Query {
  user(id: ID!): User
  post(id: ID!): Post
  posts: [Post!]!
}
```

## Query Execution

### Example Query:

```graphql
query {
  posts {
    id
    title
    author {
      name
      email
    }
    comments {
      text
      author {
        name
      }
    }
  }
}
```

### Execution Plan:

```
1. Query Planner analyzes query and creates execution plan:

Step 1: Fetch posts from Posts Service
  → [{ id: 1, title: "Post 1", authorId: 101 }, ...]

Step 2: Batch fetch authors (DataLoader)
  → Collect authorIds: [101, 102, 103]
  → Single request to Users Service: getUsers([101, 102, 103])

Step 3: Fetch comments from Comments Service
  → Pass postIds: [1, 2, 3]
  → Returns comments with authorIds

Step 4: Batch fetch comment authors (DataLoader)
  → Collect unique authorIds from comments
  → Single request to Users Service

Step 5: Assemble response
  → Combine all data according to query structure
```

## DataLoader Implementation

```typescript
import DataLoader from 'dataloader';

class UserDataLoader {
  private loader: DataLoader<number, User>;

  constructor(private userService: UserService) {
    this.loader = new DataLoader<number, User>(
      async (ids: readonly number[]) => {
        console.log(`Batching request for users: ${ids}`);

        // Single database query for all IDs
        const users = await this.userService.getUsersByIds([...ids]);

        // Return in same order as requested IDs
        return ids.map(id => users.find(u => u.id === id) || null);
      },
      {
        // Optional: caching within request
        cache: true,
        batchScheduleFn: callback => setTimeout(callback, 10) // Wait 10ms to batch
      }
    );
  }

  async load(id: number): Promise<User | null> {
    return this.loader.load(id);
  }

  async loadMany(ids: number[]): Promise<Array<User | null>> {
    return this.loader.loadMany(ids);
  }
}

// Usage in resolver:
const resolvers = {
  Post: {
    author: async (post, args, context) => {
      // Will be batched with other author requests
      return context.dataloaders.users.load(post.authorId);
    }
  },
  Query: {
    posts: async (_, args, context) => {
      return postsService.getAllPosts();
    }
  }
};
```

### Before/After DataLoader:

```
WITHOUT DataLoader (N+1 problem):
  GET /posts → 100 posts
  GET /users/1
  GET /users/2
  ...
  GET /users/100
  Total: 101 requests

WITH DataLoader:
  GET /posts → 100 posts
  GET /users?ids=1,2,3,...,100  (batched!)
  Total: 2 requests

Improvement: 50x fewer requests!
```

## Resolver Implementation

```typescript
// Gateway resolvers that delegate to services

const resolvers = {
  Query: {
    user: async (_, { id }, context) => {
      return context.dataloaders.users.load(id);
    },

    posts: async (_, args, context) => {
      return context.services.posts.getPosts();
    }
  },

  User: {
    // Resolve User.posts by calling Posts Service
    posts: async (user, args, context) => {
      return context.services.posts.getPostsByAuthorId(user.id);
    }
  },

  Post: {
    // Resolve Post.author using DataLoader
    author: async (post, args, context) => {
      return context.dataloaders.users.load(post.authorId);
    },

    // Resolve Post.comments by calling Comments Service
    comments: async (post, args, context) => {
      return context.services.comments.getCommentsByPostId(post.id);
    }
  },

  Comment: {
    author: async (comment, args, context) => {
      return context.dataloaders.users.load(comment.authorId);
    },

    post: async (comment, args, context) => {
      return context.dataloaders.posts.load(comment.postId);
    }
  }
};

// Context creation (per request)
const createContext = ({ req }) => {
  return {
    user: req.user,  // From authentication
    dataloaders: {
      users: new UserDataLoader(userService),
      posts: new PostDataLoader(postService)
    },
    services: {
      users: userService,
      posts: postService,
      comments: commentService
    }
  };
};
```

## Caching Strategies

### 1. Response Caching

```typescript
import { KeyvAdapter } from '@apollo/utils.keyvadapter';
import Keyv from 'keyv';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  cache: new KeyvAdapter(new Keyv('redis://localhost:6379')),
  plugins: [
    ApolloServerPluginCacheControl({
      defaultMaxAge: 300,  // 5 minutes
      calculateHttpHeaders: true
    })
  ]
});

// In schema:
type User {
  id: ID!
  name: String! @cacheControl(maxAge: 3600)  # Cache 1 hour
  email: String! @cacheControl(maxAge: 0)    # Don't cache (sensitive)
}
```

### 2. Persisted Queries

```typescript
// Client sends hash instead of full query
POST /graphql
{
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "abc123..."
    }
  }
}

// Server:
const persistedQueries = new Map([
  ['abc123...', 'query { user(id: 1) { name } }']
]);

// Benefits:
// - Reduced bandwidth (hash vs full query)
// - Faster parsing (pre-parsed queries)
// - Security (whitelist approved queries)
```

## Error Handling

### Partial Errors:

```graphql
query {
  user(id: 1) {
    name     # ✅ Success
    posts {  # ❌ Posts Service down
      title
    }
  }
}

# Response:
{
  "data": {
    "user": {
      "name": "John",
      "posts": null
    }
  },
  "errors": [
    {
      "message": "Posts service unavailable",
      "path": ["user", "posts"],
      "extensions": {
        "code": "SERVICE_UNAVAILABLE",
        "serviceName": "posts"
      }
    }
  ]
}
```

### Error Masking:

```typescript
const formatError = (error: GraphQLError) => {
  // Don't expose internal errors to client
  if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
    return new GraphQLError('An error occurred', {
      extensions: {
        code: 'INTERNAL_SERVER_ERROR'
      }
    });
  }

  // Log internal error
  logger.error(error);

  return error;
};
```

## Authentication & Authorization

```typescript
// Authentication at gateway
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: async ({ req }) => {
    const token = req.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      throw new GraphQLError('Not authenticated', {
        extensions: { code: 'UNAUTHENTICATED' }
      });
    }

    const user = await verifyToken(token);
    return { user };
  }
});

// Authorization in resolvers
const resolvers = {
  Query: {
    adminUsers: async (_, args, context) => {
      if (!context.user.isAdmin) {
        throw new GraphQLError('Not authorized', {
          extensions: { code: 'FORBIDDEN' }
        });
      }

      return userService.getAllUsers();
    }
  },

  User: {
    email: (user, args, context) => {
      // Only return email if requesting own profile
      if (context.user.id !== user.id) {
        return null;
      }
      return user.email;
    }
  }
};
```

## Performance Monitoring

```typescript
import { ApolloServerPluginLandingPageGraphQLPlayground } from 'apollo-server-core';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    // Apollo Studio integration
    ApolloServerPluginUsageReporting({
      sendVariableValues: { all: true }
    }),

    // Custom performance tracking
    {
      async requestDidStart(requestContext) {
        const start = Date.now();

        return {
          async willSendResponse(requestContext) {
            const duration = Date.now() - start;

            metrics.histogram('graphql.query.duration', duration, {
              operationName: requestContext.operationName || 'anonymous'
            });

            console.log({
              operation: requestContext.operationName,
              duration,
              query: requestContext.request.query,
              variables: requestContext.request.variables
            });
          }
        };
      }
    }
  ]
});
```

## Trade-offs

### GraphQL vs REST

| Aspect | GraphQL | REST |
|--------|---------|------|
| **Over-fetching** | No (client specifies fields) | Yes (fixed endpoints) |
| **Under-fetching** | No (get all data in one request) | Yes (multiple requests) |
| **Versioning** | No versions (schema evolution) | API versions (v1, v2) |
| **Caching** | Complex (query-based) | Simple (URL-based) |
| **File Upload** | Complex | Simple |
| **Learning Curve** | Higher | Lower |

### When to Use GraphQL:
- ✅ Multiple clients (web, mobile) with different needs
- ✅ Need to aggregate data from multiple services
- ✅ Want to minimize round trips
- ✅ Complex, nested data requirements

### When to Use REST:
- ✅ Simple CRUD operations
- ✅ Need HTTP caching
- ✅ File uploads/downloads
- ✅ Team unfamiliar with GraphQL

## Summary

### What We Built:
- GraphQL Federation Gateway
- Schema stitching across microservices
- DataLoader for N+1 problem solution
- Request batching and caching
- Unified authentication

### Key Concepts:
- ✅ N+1 query problem and DataLoader solution
- ✅ Schema federation across services
- ✅ Query planning and execution
- ✅ Response caching strategies
- ✅ Error handling in distributed systems

### Real-World Examples:
- Netflix: 100+ services unified via GraphQL
- GitHub: Public GraphQL API
- Shopify: Storefront API (GraphQL)

---

**Next Project**: [08. Resilient Payment Processor](../08-payment-processor/README.md)
