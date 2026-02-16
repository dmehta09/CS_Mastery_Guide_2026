# Project 07: GraphQL Federation Backend

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language

## What You're Building

A federated GraphQL API that combines multiple microservices into a single unified graph (similar to Netflix's architecture). Clients query one endpoint, but data comes from multiple services.

## Key Concepts

- GraphQL schema design
- Federation & subgraphs
- Schema stitching
- N+1 problem & DataLoader
- Query complexity analysis
- Persisted queries
- Real-time subscriptions
- Caching strategies

## Architecture

```
Client → Apollo Gateway → [User Service, Product Service, Order Service]
                ↓
           Redis Cache
```

## Schema Example

```graphql
# User Service
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]! @requires(fields: "id")
}

# Product Service
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
  inventory: Int!
}

# Order Service
type Order @key(fields: "id") {
  id: ID!
  user: User!
  products: [Product!]!
  total: Float!
  status: OrderStatus!
}

# Unified Query
type Query {
  user(id: ID!): User
  product(id: ID!): Product
  orders(userId: ID!): [Order!]!
}
```

## DataLoader Pattern

```typescript
// Batch and cache database queries
const userLoader = new DataLoader(async (ids) => {
  const users = await db.findUsersByIds(ids);
  return ids.map((id) =>
    users.find((u) => u.id === id)
  );
});

// Resolvers
const resolvers = {
  Order: {
    user: (order, args, { dataloaders }) => {
      return dataloaders.user.load(order.userId);
    }
  }
};
```

## Success Criteria

✅ Sub-100ms query response
✅ Solve N+1 problem
✅ Query complexity limiting
✅ Federated services working
✅ Subscription support

---

**Next**: Project 08 (E-Commerce Inventory)
