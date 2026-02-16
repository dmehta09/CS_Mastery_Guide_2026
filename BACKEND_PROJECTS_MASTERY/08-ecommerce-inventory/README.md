# Project 08: E-Commerce Inventory Management

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language

## What You're Building

A high-concurrency inventory system that prevents overselling during flash sales (think Black Friday on Amazon). Must handle race conditions when thousands of users buy the last item simultaneously.

## Key Concepts

- Optimistic locking
- Pessimistic locking
- Distributed locks (Redis)
- Inventory reservation
- Stock reconciliation
- Eventually consistent inventory
- Conflict resolution

## The Problem

```
Problem: 1 item in stock, 1000 users try to buy simultaneously

Without proper locking:
Thread 1: Read stock = 1 ✓
Thread 2: Read stock = 1 ✓
Thread 1: Decrement to 0, Commit
Thread 2: Decrement to 0, Commit
Result: 2 orders for 1 item! 😱
```

## Solutions

### 1. Optimistic Locking

```sql
UPDATE inventory
SET quantity = quantity - 1,
    version = version + 1
WHERE product_id = $1
  AND version = $2  -- Must match current version
  AND quantity >= 1;

-- Returns 0 rows if version mismatch
```

### 2. Pessimistic Locking

```sql
BEGIN TRANSACTION;

SELECT quantity
FROM inventory
WHERE product_id = $1
FOR UPDATE; -- Locks this row

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = $1
  AND quantity >= 1;

COMMIT;
```

### 3. Distributed Lock (Redis)

```typescript
async function reserveInventory(productId, quantity) {
  const lockKey = `lock:inventory:${productId}`;
  const lock = await redis.set(
    lockKey,
    "1",
    "NX",
    "EX",
    10
  );

  if (!lock) {
    throw new Error("Could not acquire lock");
  }

  try {
    // Check and update inventory
    const current = await getInventory(productId);
    if (current >= quantity) {
      await updateInventory(productId, -quantity);
      return true;
    }
    return false;
  } finally {
    await redis.del(lockKey);
  }
}
```

### 4. Reservation Pattern

```typescript
// Two-phase commit for orders
async function placeOrder(orderId, items) {
  // Phase 1: Reserve inventory
  const reservations = await Promise.all(
    items.map((item) =>
      reserveInventory(item.productId, item.quantity)
    )
  );

  if (reservations.some((r) => !r.success)) {
    // Rollback all reservations
    await rollbackReservations(reservations);
    throw new Error("Insufficient inventory");
  }

  // Phase 2: Process payment
  try {
    await processPayment(orderId);
    // Commit reservations
    await commitReservations(reservations);
  } catch (error) {
    await rollbackReservations(reservations);
    throw error;
  }
}
```

## Database Schema

```sql
CREATE TABLE inventory (
    product_id BIGINT PRIMARY KEY,
    quantity INT NOT NULL CHECK (quantity >= 0),
    reserved INT NOT NULL DEFAULT 0,
    version INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP NOT NULL
);

CREATE TABLE inventory_reservations (
    id UUID PRIMARY KEY,
    product_id BIGINT REFERENCES inventory(product_id),
    order_id UUID NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(20) NOT NULL, -- 'pending', 'committed', 'released'
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

## Success Criteria

✅ No overselling
✅ Handle 1000+ concurrent requests
✅ Automatic reservation expiry
✅ Stock reconciliation
✅ Audit trail

---

**Next**: Project 09 (Social Feed Engine)
