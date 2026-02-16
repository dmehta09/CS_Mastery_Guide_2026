# Project 04: Polyglot E-Commerce Platform

## Difficulty: Advanced
**Estimated Time**: 4-5 weeks

## Problem Statement

Design an e-commerce platform like Amazon or Shopify that uses multiple databases (polyglot persistence) based on data access patterns, implements CQRS for read/write separation, and uses event sourcing for audit trails.

**Key Insight**: Different data has different access patterns. Use the right database for each use case rather than forcing everything into one database.

## Functional Requirements

### Core Features:
1. **Product Catalog**: Browse, search products
2. **Shopping Cart**: Add/remove items, calculate totals
3. **Orders**: Place orders, track status
4. **Inventory**: Track stock levels, prevent overselling
5. **User Accounts**: Profile, order history
6. **Search**: Full-text search across products
7. **Recommendations**: "You might also like"

### Out of Scope:
- Payment processing (assume external)
- Shipping/logistics
- Multi-vendor marketplace

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Read Latency** | < 100ms | For product browsing |
| **Write Latency** | < 500ms | For order placement |
| **Availability** | 99.95% | ~22 hours downtime/year |
| **Consistency** | Eventual (1-2s) | For catalog updates |
| **Consistency** | Strong | For inventory/orders |
| **Scalability** | 100K concurrent users | |

## Capacity Estimation

### Assumptions:
- **10 million users**
- **1 million products**
- **10K orders per day**
- **100K concurrent shoppers**
- **Read:Write ratio = 100:1** (mostly browsing)

### Traffic:
```
Daily requests:
- Product views: 100M requests/day
- Searches: 10M/day
- Cart operations: 5M/day
- Orders: 10K/day

Peak (10x average):
- Product views: 11.5K RPS
- Searches: 115 RPS
- Cart: 57 RPS
- Orders: 1 RPS
```

### Storage:
```
Products:
- 1M products × 10 KB/product = 10 GB
- Images (S3): 1M × 500 KB × 5 images = 2.5 TB

Orders:
- 10K orders/day × 365 days × 5 years = 18.25M orders
- 18.25M × 5 KB = 91 GB

Users:
- 10M users × 2 KB = 20 GB

Total: ~150 GB (structured data) + 2.5 TB (images)
```

## Database Selection Matrix

| Data Type | Access Pattern | Database | Why? |
|-----------|---------------|----------|------|
| **Product Catalog** | Read-heavy, occasional updates | PostgreSQL (Primary)<br>+ Read Replicas | ACID for inventory, fast reads |
| **Product Search** | Full-text search, filters | Elasticsearch | Inverted index, faceted search |
| **Shopping Cart** | Temporary, fast R/W | Redis | In-memory, TTL for abandoned carts |
| **User Sessions** | Fast R/W, TTL | Redis | Sub-millisecond latency |
| **Order History** | Immutable, audit trail | PostgreSQL + Event Store | ACID, append-only log |
| **Product Views** | Time-series analytics | InfluxDB / TimescaleDB | Time-based queries, aggregations |
| **Recommendations** | Graph relationships | Neo4j / Neptune | "Users who bought X also bought Y" |
| **Product Images** | Large objects, CDN | S3 + CloudFront | Object storage, global delivery |

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                     API Gateway                     │
│              (GraphQL / REST / gRPC)                │
└──────────────┬──────────────────────────────────────┘
               │
    ┌──────────┴──────────┬─────────────────┬────────┐
    │                     │                 │        │
    ▼                     ▼                 ▼        ▼
┌────────┐          ┌──────────┐      ┌────────┐  ┌────────┐
│Catalog │          │  Cart    │      │ Order  │  │ Search │
│Service │          │ Service  │      │Service │  │Service │
└───┬────┘          └────┬─────┘      └───┬────┘  └───┬────┘
    │                    │                 │           │
    ▼                    ▼                 ▼           ▼
┌──────────┐       ┌─────────┐     ┌───────────┐ ┌──────────┐
│PostgreSQL│       │  Redis  │     │PostgreSQL │ │Elastic-  │
│(Products)│       │ (Carts) │     │ (Orders)  │ │ search   │
└──────────┘       └─────────┘     └─────┬─────┘ └──────────┘
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │ Event Store  │
                                   │ (Audit Log)  │
                                   └──────────────┘
```

## CQRS (Command Query Responsibility Segregation)

### Concept

```
Traditional:
  Client → Service → Single DB (both reads and writes)

CQRS:
  Client → Write Service → Write DB (Commands)
       └→ Read Service  → Read DB (Queries)

  Write DB ──[sync]──> Read DB (eventual consistency)
```

### Benefits:
1. **Optimize separately**: Different databases for reads vs writes
2. **Scale independently**: 100 read replicas, 1 write primary
3. **Flexibility**: Denormalize read models for performance

### Implementation

```typescript
// Command Side (Writes)
class PlaceOrderCommand {
  userId: string;
  items: CartItem[];
  totalAmount: number;
}

class OrderCommandHandler {
  async handle(command: PlaceOrderCommand) {
    // 1. Validate
    await this.validateInventory(command.items);

    // 2. Create order (write to primary DB)
    const order = await this.orderRepository.create({
      userId: command.userId,
      items: command.items,
      status: 'pending',
      total: command.totalAmount
    });

    // 3. Publish event
    await this.eventBus.publish(new OrderPlacedEvent(order));

    // 4. Update inventory (transactional)
    await this.inventoryService.decrementStock(command.items);

    return { orderId: order.id };
  }
}

// Query Side (Reads)
class OrderQueryHandler {
  async getOrderHistory(userId: string): Promise<Order[]> {
    // Read from optimized read model
    return await this.orderReadRepository.findByUser(userId);
  }

  async getOrderDetails(orderId: string): Promise<OrderDetails> {
    // Denormalized view with all data pre-joined
    return await this.orderReadRepository.findById(orderId);
  }
}

// Event Handler (Sync write → read)
class OrderPlacedEventHandler {
  async handle(event: OrderPlacedEvent) {
    // Update read model (eventual consistency)
    await this.orderReadRepository.upsert({
      orderId: event.orderId,
      userId: event.userId,
      items: event.items,
      status: 'pending',
      createdAt: event.timestamp,
      // Denormalized: include product names, images, etc.
      itemsWithDetails: await this.enrichWithProductDetails(event.items)
    });

    // Update analytics
    await this.analyticsService.recordOrder(event);
  }
}
```

## Event Sourcing

### Concept

Instead of storing current state, store sequence of events:

```
Traditional (State):
  Order { id: 123, status: 'shipped', total: $100 }

Event Sourcing (Events):
  OrderCreated { orderId: 123, items: [...], total: $100 }
  OrderPaid { orderId: 123, amount: $100, paymentId: 'abc' }
  OrderShipped { orderId: 123, trackingNumber: 'XYZ123' }

Current state = replay all events
```

### Benefits:
1. **Complete audit trail**: Know exactly what happened and when
2. **Time travel**: Reconstruct state at any point in time
3. **Event replay**: Rebuild read models from scratch

### Implementation

```typescript
// Event Store Schema
CREATE TABLE order_events (
  id BIGSERIAL PRIMARY KEY,
  aggregate_id UUID NOT NULL,           -- Order ID
  aggregate_type VARCHAR(50) NOT NULL,  -- 'Order'
  event_type VARCHAR(100) NOT NULL,     -- 'OrderCreated', 'OrderPaid', etc.
  event_data JSONB NOT NULL,
  event_version INT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  user_id UUID,

  INDEX idx_aggregate (aggregate_id, event_version),
  INDEX idx_created (created_at)
);

// Event Types
interface OrderEvent {
  aggregateId: string;  // Order ID
  eventType: string;
  eventData: any;
  version: number;
  timestamp: Date;
}

class OrderCreatedEvent implements OrderEvent {
  aggregateId: string;
  eventType = 'OrderCreated';
  eventData: {
    userId: string;
    items: CartItem[];
    total: number;
  };
  version: number;
  timestamp: Date;
}

class OrderPaidEvent implements OrderEvent {
  aggregateId: string;
  eventType = 'OrderPaid';
  eventData: {
    paymentId: string;
    amount: number;
    method: string;
  };
  version: number;
  timestamp: Date;
}

// Order Aggregate (reconstructed from events)
class Order {
  id: string;
  userId: string;
  items: CartItem[];
  total: number;
  status: string;
  paymentId?: string;
  trackingNumber?: string;
  version: number;

  // Reconstruct from events
  static fromEvents(events: OrderEvent[]): Order {
    const order = new Order();

    events.forEach(event => {
      order.applyEvent(event);
    });

    return order;
  }

  private applyEvent(event: OrderEvent) {
    switch (event.eventType) {
      case 'OrderCreated':
        this.id = event.aggregateId;
        this.userId = event.eventData.userId;
        this.items = event.eventData.items;
        this.total = event.eventData.total;
        this.status = 'pending';
        break;

      case 'OrderPaid':
        this.status = 'paid';
        this.paymentId = event.eventData.paymentId;
        break;

      case 'OrderShipped':
        this.status = 'shipped';
        this.trackingNumber = event.eventData.trackingNumber;
        break;

      // ... more event types
    }

    this.version = event.version;
  }

  // Get events from database
  static async load(orderId: string): Promise<Order> {
    const events = await db.query(
      'SELECT * FROM order_events WHERE aggregate_id = $1 ORDER BY event_version',
      [orderId]
    );

    return Order.fromEvents(events.rows);
  }
}

// Saving events
class OrderRepository {
  async save(order: Order, newEvent: OrderEvent): Promise<void> {
    await db.query(
      `INSERT INTO order_events (aggregate_id, aggregate_type, event_type, event_data, event_version)
       VALUES ($1, $2, $3, $4, $5)`,
      [
        order.id,
        'Order',
        newEvent.eventType,
        newEvent.eventData,
        order.version + 1
      ]
    );

    // Publish to event bus for read model updates
    await eventBus.publish(newEvent);
  }
}
```

## Database Schemas

### PostgreSQL - Products

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL,
  category VARCHAR(100),
  brand VARCHAR(100),
  stock_quantity INT NOT NULL DEFAULT 0,
  image_urls TEXT[],
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_category (category),
  INDEX idx_stock (stock_quantity),
  FULLTEXT INDEX idx_search (name, description)
);

CREATE TABLE product_variants (
  id UUID PRIMARY KEY,
  product_id UUID REFERENCES products(id),
  size VARCHAR(20),
  color VARCHAR(50),
  sku VARCHAR(100) UNIQUE,
  price_adjustment DECIMAL(10, 2) DEFAULT 0,
  stock_quantity INT NOT NULL
);
```

### PostgreSQL - Orders

```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  status VARCHAR(50) NOT NULL,  -- pending, paid, shipped, delivered
  total_amount DECIMAL(10, 2) NOT NULL,
  shipping_address JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_user (user_id),
  INDEX idx_status (status),
  INDEX idx_created (created_at)
);

CREATE TABLE order_items (
  id UUID PRIMARY KEY,
  order_id UUID REFERENCES orders(id),
  product_id UUID REFERENCES products(id),
  quantity INT NOT NULL,
  price DECIMAL(10, 2) NOT NULL,
  snapshot JSONB  -- Product details at time of order
);
```

### Redis - Shopping Carts

```redis
Key: cart:{user_id}
Type: Hash
Fields:
  product:{product_id} → { quantity: 2, price: 29.99, addedAt: timestamp }

Example:
HSET cart:user123 product:abc '{"quantity":2,"price":29.99,"addedAt":1708084800}'
HGETALL cart:user123
EXPIRE cart:user123 604800  # 7 days TTL for abandoned carts

# Cart totals (cached)
Key: cart_total:{user_id}
Value: "59.98"
TTL: 300  # 5 minutes
```

### Elasticsearch - Search Index

```json
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": { "type": "text" },
      "price": { "type": "float" },
      "category": { "type": "keyword" },
      "brand": { "type": "keyword" },
      "in_stock": { "type": "boolean" },
      "rating": { "type": "float" },
      "num_reviews": { "type": "integer" },
      "tags": { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}
```

## API Design

### REST Endpoints

```http
# Browse products
GET /api/v1/products?category=electronics&page=1&limit=20
Response: {
  "products": [...],
  "total": 500,
  "page": 1,
  "pages": 25
}

# Search products
GET /api/v1/products/search?q=laptop&min_price=500&max_price=2000
Response: {
  "results": [...],
  "facets": {
    "brands": {"Dell": 50, "HP": 30, "Apple": 20},
    "categories": {"Laptops": 100}
  }
}

# Add to cart
POST /api/v1/cart/items
Body: {
  "product_id": "abc-123",
  "quantity": 2
}

# Get cart
GET /api/v1/cart
Response: {
  "items": [
    {
      "product_id": "abc-123",
      "name": "Laptop",
      "quantity": 2,
      "price": 999.99,
      "subtotal": 1999.98
    }
  ],
  "total": 1999.98
}

# Place order
POST /api/v1/orders
Body: {
  "shipping_address": {...},
  "payment_method_id": "pm_123"
}
Response: {
  "order_id": "ord-456",
  "status": "pending",
  "total": 1999.98
}

# Get order history
GET /api/v1/orders?user_id=user123
```

### GraphQL (Alternative)

```graphql
query GetProduct($id: ID!) {
  product(id: $id) {
    id
    name
    price
    description
    images {
      url
      alt
    }
    variants {
      size
      color
      inStock
    }
    recommendations {
      id
      name
      price
    }
  }
}

mutation AddToCart($productId: ID!, $quantity: Int!) {
  addToCart(productId: $productId, quantity: $quantity) {
    cart {
      items {
        product {
          name
          price
        }
        quantity
        subtotal
      }
      total
    }
  }
}
```

## Trade-off Analysis

### CQRS Pros/Cons

| Pros | Cons |
|------|------|
| ✅ Optimize reads and writes separately | ❌ Complexity: Two models to maintain |
| ✅ Scale independently | ❌ Eventual consistency (delay) |
| ✅ Flexible read models (denormalization) | ❌ Need event synchronization |
| ✅ Better performance | ❌ Debugging harder |

**When to use**:
- High read:write ratio (100:1 or more)
- Need different optimization strategies
- Can tolerate eventual consistency

**When NOT to use**:
- Simple CRUD apps
- Need strong consistency everywhere
- Small scale (<10K users)

### Event Sourcing Pros/Cons

| Pros | Cons |
|------|------|
| ✅ Complete audit trail | ❌ Storage grows indefinitely |
| ✅ Time travel debugging | ❌ Query complexity |
| ✅ Event replay for analytics | ❌ Learning curve |
| ✅ Natural fit for undo/redo | ❌ Schema evolution challenges |

**When to use**:
- Need audit trail (financial, healthcare)
- Complex business logic
- Event-driven architecture

**When NOT to use**:
- Simple applications
- Storage constraints
- Team unfamiliar with pattern

## Preventing Overselling

### Problem:
```
Stock: 1 laptop
User A: Add to cart (sees 1 available)
User B: Add to cart (sees 1 available)
Both checkout → 2 orders for 1 item!
```

### Solution 1: Pessimistic Locking

```sql
BEGIN TRANSACTION;

-- Lock row for update
SELECT stock_quantity FROM products
WHERE id = 'laptop-123'
FOR UPDATE;

-- Check stock
IF stock_quantity >= requested_quantity THEN
  -- Decrement
  UPDATE products
  SET stock_quantity = stock_quantity - requested_quantity
  WHERE id = 'laptop-123';

  COMMIT;
ELSE
  ROLLBACK;
END IF;
```

### Solution 2: Optimistic Locking

```typescript
async function checkoutCart(userId: string) {
  while (true) {
    // Read current version
    const product = await db.query(
      'SELECT id, stock_quantity, version FROM products WHERE id = $1',
      [productId]
    );

    if (product.stock_quantity < requestedQuantity) {
      throw new OutOfStockError();
    }

    // Try to update with version check
    const result = await db.query(
      `UPDATE products
       SET stock_quantity = stock_quantity - $1,
           version = version + 1
       WHERE id = $2 AND version = $3
       RETURNING *`,
      [requestedQuantity, productId, product.version]
    );

    if (result.rowCount > 0) {
      // Success!
      break;
    } else {
      // Version mismatch, someone else updated, retry
      await sleep(10);
    }
  }
}
```

### Solution 3: Reserve Stock in Cart

```typescript
// When adding to cart
async function addToCart(userId: string, productId: string, quantity: number) {
  // Reserve stock
  await db.query(
    `UPDATE products
     SET stock_quantity = stock_quantity - $1,
         reserved_quantity = reserved_quantity + $1
     WHERE id = $2`,
    [quantity, productId]
  );

  // Set TTL on cart
  await redis.setex(`cart:${userId}`, 900, JSON.stringify({...}));  // 15 min

  // Background job: Release reserved stock for abandoned carts
}

// On checkout
async function checkout(userId: string) {
  // Convert reserved → sold
  await db.query(
    `UPDATE products
     SET reserved_quantity = reserved_quantity - $1
     WHERE id = $2`,
    [quantity, productId]
  );
}

// On cart expiry
async function releaseReservation(userId: string) {
  await db.query(
    `UPDATE products
     SET stock_quantity = stock_quantity + $1,
         reserved_quantity = reserved_quantity - $1
     WHERE id = $2`,
    [quantity, productId]
  );
}
```

## Summary

### What We Built:
- E-commerce platform with polyglot persistence
- CQRS for read/write separation
- Event sourcing for audit trails
- Multiple databases for different use cases
- Inventory management with concurrency control

### Key Concepts:
- ✅ Database selection matrix
- ✅ CQRS pattern
- ✅ Event sourcing
- ✅ Eventual consistency
- ✅ Optimistic vs pessimistic locking
- ✅ Polyglot persistence

### Databases Used:
- PostgreSQL: Products, orders (ACID transactions)
- Redis: Carts, sessions (fast, temporary)
- Elasticsearch: Search (full-text, facets)
- Event Store: Audit trail
- S3: Images, static assets

---

**Next Project**: [05. Time-Series Monitoring System](../05-timeseries-monitoring/README.md)
