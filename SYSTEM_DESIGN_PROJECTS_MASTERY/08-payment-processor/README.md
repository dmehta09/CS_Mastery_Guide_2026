# Project 08: Resilient Payment Processing Engine

## Difficulty: Expert
**Estimated Time**: 4-5 weeks

## Problem Statement

Design a payment processing system like Stripe, PayPal, or Square that handles distributed transactions, ensures exactly-once processing, implements the Saga pattern for rollbacks, and guarantees financial consistency even during failures.

**Critical Requirement**: Money must NEVER be lost or duplicated. This is the hardest requirement in system design.

## Functional Requirements

### Core Features:
1. **Payment Processing**: Charge credit cards, process refunds
2. **Multi-Step Transactions**: Reserve funds → Capture → Settlement
3. **Idempotency**: Same request twice = same result (no double-charging)
4. **Distributed Transactions**: Coordinate across multiple services (inventory, payment, shipping)
5. **Saga Pattern**: Rollback if any step fails
6. **Reconciliation**: Match internal records with bank statements
7. **Fraud Detection**: Basic checks before processing

### Out of Scope:
- Advanced fraud detection (ML-based)
- PCI compliance infrastructure
- Currency exchange

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Consistency** | 100% | Exactly-once processing |
| **Availability** | 99.99% | < 1 hour downtime/year |
| **Latency** | < 500ms | P95 for payment authorization |
| **Throughput** | 10K transactions/sec | Peak load |
| **Durability** | 100% | No data loss |
| **Audit Trail** | Forever | Complete transaction history |

## Capacity Estimation

### Assumptions:
- **10 million transactions per day**
- **Average transaction value**: $50
- **Total daily volume**: $500 million
- **Peak:Average ratio**: 10:1 (holiday shopping)

### Traffic:
```
Average: 10M / 86400 = 116 TPS
Peak: 116 × 10 = 1,160 TPS

With safety margin (10x): 11,600 TPS capacity needed
```

### Storage:
```
Transaction record: ~2 KB
Daily storage: 10M × 2 KB = 20 GB/day
Yearly: 20 GB × 365 = 7.3 TB/year

With 7-year retention: 51 TB

Event store (audit): 3x transaction size = 153 TB
```

## Critical Concepts

### 1. Idempotency

**Problem**: Network failures cause retries. How to prevent double-charging?

```
User clicks "Pay $100"
Request 1: ───────────────→ [Payment Service]
Network failure, timeout      Process: $100
User clicks again
Request 2: ───────────────→ [Payment Service]
                             Process: $100 again?!

Result: Charged $200 instead of $100 ❌
```

**Solution**: Idempotency Keys

```typescript
interface PaymentRequest {
  idempotencyKey: string;  // Client-generated UUID
  amount: number;
  currency: string;
  customerId: string;
}

async function processPayment(request: PaymentRequest): Promise<Payment> {
  // Check if we've seen this idempotency key before
  const existing = await db.query(
    'SELECT * FROM payments WHERE idempotency_key = $1',
    [request.idempotencyKey]
  );

  if (existing) {
    // Already processed, return same result
    return existing;
  }

  // First time seeing this request, process it
  const payment = await chargeCard(request);

  // Store with idempotency key
  await db.query(
    'INSERT INTO payments (id, idempotency_key, amount, status) VALUES ($1, $2, $3, $4)',
    [payment.id, request.idempotencyKey, request.amount, 'completed']
  );

  return payment;
}
```

### 2. Two-Phase Commit (2PC)

**Problem**: How to ensure atomicity across multiple databases?

```
Transaction: Debit $100 from Account A, Credit $100 to Account B

Phase 1: Prepare
  ├─ DB A: Can you commit? → Yes (lock acquired)
  └─ DB B: Can you commit? → Yes (lock acquired)

Phase 2: Commit
  ├─ DB A: Commit! → Done
  └─ DB B: Commit! → Done

If ANY says "No" in Phase 1 → Abort all
```

**Limitations**:
- Blocking: Locks held during both phases
- Coordinator is single point of failure
- Poor performance across WANs

### 3. Saga Pattern (Better Alternative)

**Concept**: Break transaction into multiple local transactions with compensating actions

```
Order Processing Saga:

Step 1: Reserve Inventory
Step 2: Authorize Payment
Step 3: Create Shipment

If Step 3 fails:
  Compensate Step 2: Release Payment
  Compensate Step 1: Release Inventory
```

**Implementation**:

```typescript
// Orchestrator-based Saga
class OrderSaga {
  async execute(order: Order): Promise<void> {
    const sagaLog: SagaStep[] = [];

    try {
      // Step 1: Reserve inventory
      const inventoryReserved = await inventoryService.reserve(order.items);
      sagaLog.push({
        step: 'reserve_inventory',
        compensate: () => inventoryService.release(inventoryReserved)
      });

      // Step 2: Authorize payment
      const paymentAuth = await paymentService.authorize(order.total);
      sagaLog.push({
        step: 'authorize_payment',
        compensate: () => paymentService.void(paymentAuth.id)
      });

      // Step 3: Create shipment
      const shipment = await shippingService.create(order);
      sagaLog.push({
        step: 'create_shipment',
        compensate: () => shippingService.cancel(shipment.id)
      });

      // Success!
      await this.markComplete(order.id);

    } catch (error) {
      // Failure! Compensate in reverse order
      console.error(`Saga failed at step: ${sagaLog.length}`, error);

      for (let i = sagaLog.length - 1; i >= 0; i--) {
        try {
          await sagaLog[i].compensate();
        } catch (compensateError) {
          // Compensate failed! Log for manual intervention
          await this.logCompensationFailure(sagaLog[i], compensateError);
        }
      }

      throw error;
    }
  }
}
```

**Choreography-based Saga** (Event-driven):

```typescript
// Each service reacts to events

// Inventory Service
eventBus.on('OrderCreated', async (order) => {
  try {
    await reserveInventory(order.items);
    eventBus.emit('InventoryReserved', { orderId: order.id });
  } catch (error) {
    eventBus.emit('InventoryReserveFailed', { orderId: order.id, error });
  }
});

// Payment Service
eventBus.on('InventoryReserved', async ({ orderId }) => {
  const order = await getOrder(orderId);
  try {
    await authorizePayment(order.total);
    eventBus.emit('PaymentAuthorized', { orderId });
  } catch (error) {
    eventBus.emit('PaymentAuthorizationFailed', { orderId, error });
  }
});

// Compensations
eventBus.on('PaymentAuthorizationFailed', async ({ orderId }) => {
  await releaseInventory(orderId);
});
```

### 4. Event Sourcing for Audit Trail

```sql
-- Event Store
CREATE TABLE payment_events (
  id BIGSERIAL PRIMARY KEY,
  payment_id UUID NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  event_data JSONB NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  user_id UUID,

  INDEX idx_payment (payment_id),
  INDEX idx_created (created_at)
);

-- Events:
PaymentInitiated { paymentId, amount, customerId }
PaymentAuthorized { paymentId, authCode, timestamp }
PaymentCaptured { paymentId, amount, timestamp }
PaymentFailed { paymentId, reason, timestamp }
PaymentRefunded { paymentId, amount, refundId }
```

**Reconstruct payment state**:

```typescript
class Payment {
  id: string;
  amount: number;
  status: string;
  authCode?: string;
  refundId?: string;

  static async load(paymentId: string): Promise<Payment> {
    const events = await db.query(
      'SELECT * FROM payment_events WHERE payment_id = $1 ORDER BY id',
      [paymentId]
    );

    const payment = new Payment();

    for (const event of events.rows) {
      payment.applyEvent(event);
    }

    return payment;
  }

  private applyEvent(event: PaymentEvent) {
    switch (event.event_type) {
      case 'PaymentInitiated':
        this.id = event.event_data.paymentId;
        this.amount = event.event_data.amount;
        this.status = 'initiated';
        break;

      case 'PaymentAuthorized':
        this.status = 'authorized';
        this.authCode = event.event_data.authCode;
        break;

      case 'PaymentCaptured':
        this.status = 'captured';
        break;

      case 'PaymentFailed':
        this.status = 'failed';
        break;

      case 'PaymentRefunded':
        this.status = 'refunded';
        this.refundId = event.event_data.refundId;
        break;
    }
  }
}
```

## Payment Flow

### Authorization Flow

```
1. User submits payment
         │
         ▼
2. Generate idempotency key
         │
         ▼
3. Fraud check (pre-auth)
         │
         ├─ Fraud detected → Decline
         │
         ▼
4. Authorize with payment gateway
   (Hold funds, don't transfer yet)
         │
         ├─ Declined → Return error
         │
         ▼
5. Create payment record (status: authorized)
         │
         ▼
6. Return success to user
         │
         ▼
7. Capture payment (async, within 7 days)
   (Actually transfer funds)
         │
         ▼
8. Settlement (batch process, nightly)
   (Transfer to merchant account)
```

**Why Authorize then Capture?**

```
Authorize: "Hold $100 on card" (instant)
Capture: "Actually charge $100" (can be delayed)

Benefits:
- Can cancel order before capture (no refund needed)
- Can adjust amount (e.g., final shipping cost)
- Reduce risk of chargebacks

Example: Hotel booking
  Check-in: Authorize $500
  Check-out: Capture actual cost $437 (3 nights + damages)
```

### Implementation:

```typescript
class PaymentProcessor {
  async authorize(request: PaymentRequest): Promise<Payment> {
    // 1. Idempotency check
    const existing = await this.checkIdempotency(request.idempotencyKey);
    if (existing) return existing;

    // 2. Fraud check
    const fraudScore = await fraudService.check(request);
    if (fraudScore > 0.8) {
      throw new PaymentError('FRAUD_DETECTED');
    }

    // 3. Call payment gateway
    const authResult = await paymentGateway.authorize({
      cardNumber: request.cardNumber,
      amount: request.amount,
      currency: request.currency
    });

    if (!authResult.approved) {
      throw new PaymentError('DECLINED', authResult.reason);
    }

    // 4. Store payment
    const payment = await db.transaction(async (tx) => {
      const payment = await tx.query(
        `INSERT INTO payments (
          id, idempotency_key, amount, currency, status, auth_code
        ) VALUES ($1, $2, $3, $4, $5, $6) RETURNING *`,
        [
          uuid(),
          request.idempotencyKey,
          request.amount,
          request.currency,
          'authorized',
          authResult.authCode
        ]
      );

      // 5. Log event
      await tx.query(
        `INSERT INTO payment_events (payment_id, event_type, event_data)
         VALUES ($1, $2, $3)`,
        [
          payment.id,
          'PaymentAuthorized',
          { authCode: authResult.authCode, timestamp: new Date() }
        ]
      );

      return payment;
    });

    return payment;
  }

  async capture(paymentId: string): Promise<Payment> {
    const payment = await Payment.load(paymentId);

    if (payment.status !== 'authorized') {
      throw new Error('Payment not in authorized state');
    }

    // Call payment gateway to capture
    const captureResult = await paymentGateway.capture({
      authCode: payment.authCode,
      amount: payment.amount
    });

    // Update payment
    await db.transaction(async (tx) => {
      await tx.query(
        'UPDATE payments SET status = $1 WHERE id = $2',
        ['captured', paymentId]
      );

      await tx.query(
        `INSERT INTO payment_events (payment_id, event_type, event_data)
         VALUES ($1, $2, $3)`,
        [paymentId, 'PaymentCaptured', { timestamp: new Date() }]
      );
    });

    return payment;
  }
}
```

## Handling Money (Critical!)

### Never Use Floats!

```typescript
// ❌ WRONG - Floating point errors!
const price = 0.1 + 0.2;  // 0.30000000000004
const total = 10.1 * 3;   // 30.299999999999997

// ✅ CORRECT - Use integers (cents)
class Money {
  private amountInCents: number;
  private currency: string;

  constructor(amountInCents: number, currency: string) {
    this.amountInCents = Math.round(amountInCents);
    this.currency = currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('Currency mismatch');
    }
    return new Money(this.amountInCents + other.amountInCents, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.amountInCents * factor), this.currency);
  }

  toDollars(): number {
    return this.amountInCents / 100;
  }

  toString(): string {
    return `${this.currency} ${this.toDollars().toFixed(2)}`;
  }
}

// Usage:
const price = new Money(1000, 'USD');  // $10.00
const tax = price.multiply(0.08);      // $0.80
const total = price.add(tax);          // $10.80
```

### Database Schema for Money:

```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY,
  amount_cents BIGINT NOT NULL,  -- Store as cents!
  currency CHAR(3) NOT NULL,     -- ISO 4217 (USD, EUR, etc.)
  status VARCHAR(50),

  -- Never do: amount DECIMAL(10,2) for financial calculations!
  -- Reason: DECIMAL is better but still has rounding issues

  CHECK (amount_cents >= 0),
  CHECK (currency IN ('USD', 'EUR', 'GBP', 'JPY'))
);
```

## Distributed Transaction Patterns

### Pattern 1: Outbox Pattern

**Problem**: How to update database AND send event atomically?

```typescript
// ❌ Problem: What if DB succeeds but Kafka fails?
async function processPayment(payment: Payment) {
  await db.insert(payment);      // Success
  await kafka.publish(payment);  // Fails! Event lost!
}
```

**Solution**: Outbox Pattern

```sql
-- Add outbox table
CREATE TABLE outbox (
  id BIGSERIAL PRIMARY KEY,
  event_type VARCHAR(100),
  event_data JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  processed BOOLEAN DEFAULT FALSE
);

-- Transaction: Insert payment + outbox entry
BEGIN;
  INSERT INTO payments (...) VALUES (...);
  INSERT INTO outbox (event_type, event_data)
  VALUES ('PaymentCompleted', '{"paymentId": "123", ...}');
COMMIT;

-- Background worker: Publish events from outbox
while (true) {
  const events = await db.query(
    'SELECT * FROM outbox WHERE processed = FALSE LIMIT 100'
  );

  for (const event of events) {
    await kafka.publish(event.event_type, event.event_data);
    await db.query('UPDATE outbox SET processed = TRUE WHERE id = $1', [event.id]);
  }

  await sleep(1000);
}
```

### Pattern 2: Retry with Exponential Backoff

```typescript
async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  maxAttempts: number = 5
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;

      if (attempt === maxAttempts) {
        throw error;
      }

      // Exponential backoff: 1s, 2s, 4s, 8s, 16s
      const delayMs = Math.pow(2, attempt) * 1000;
      console.log(`Attempt ${attempt} failed, retrying in ${delayMs}ms`);
      await sleep(delayMs);
    }
  }

  throw lastError!;
}

// Usage:
const payment = await retryWithBackoff(() =>
  paymentGateway.authorize(request)
);
```

## Reconciliation

**Problem**: Internal records must match bank statements

```typescript
class ReconciliationService {
  async reconcile(date: Date): Promise<ReconciliationReport> {
    // 1. Get our internal records for the day
    const ourPayments = await db.query(
      `SELECT id, amount_cents, external_ref
       FROM payments
       WHERE DATE(created_at) = $1 AND status = 'captured'`,
      [date]
    );

    // 2. Get bank statement for the day
    const bankTransactions = await bankAPI.getTransactions(date);

    // 3. Match transactions
    const matched: Match[] = [];
    const unmatched: Unmatch[] = [];

    for (const payment of ourPayments) {
      const bankTx = bankTransactions.find(
        tx => tx.referenceId === payment.external_ref
      );

      if (!bankTx) {
        unmatched.push({
          type: 'missing_in_bank',
          paymentId: payment.id,
          amount: payment.amount_cents
        });
      } else if (bankTx.amount !== payment.amount_cents) {
        unmatched.push({
          type: 'amount_mismatch',
          paymentId: payment.id,
          ourAmount: payment.amount_cents,
          bankAmount: bankTx.amount
        });
      } else {
        matched.push({ paymentId: payment.id, bankTxId: bankTx.id });
      }
    }

    // 4. Generate report
    return {
      date,
      totalPayments: ourPayments.length,
      matched: matched.length,
      unmatched: unmatched.length,
      discrepancies: unmatched
    };
  }
}
```

## API Design

```http
# Authorize payment
POST /api/v1/payments
Idempotency-Key: unique-request-id-123
Body: {
  "amount": 10000,  # $100.00 (in cents)
  "currency": "USD",
  "card": {
    "number": "4242424242424242",
    "exp_month": 12,
    "exp_year": 2026,
    "cvc": "123"
  }
}
Response: {
  "id": "pay_123",
  "status": "authorized",
  "amount": 10000,
  "currency": "USD",
  "auth_code": "AUTH123"
}

# Capture payment
POST /api/v1/payments/pay_123/capture
Response: {
  "id": "pay_123",
  "status": "captured",
  "captured_amount": 10000
}

# Refund payment
POST /api/v1/payments/pay_123/refund
Body: {
  "amount": 5000  # Partial refund: $50.00
}

# Get payment
GET /api/v1/payments/pay_123
Response: {
  "id": "pay_123",
  "status": "captured",
  "amount": 10000,
  "refunded_amount": 5000,
  "events": [
    {"type": "authorized", "timestamp": "..."},
    {"type": "captured", "timestamp": "..."},
    {"type": "refunded", "timestamp": "..."}
  ]
}
```

## Failure Scenarios

### 1. Partial Failure in Saga

```
Scenario: Reserve inventory succeeded, payment failed

Solution: Compensate inventory reservation
  → Release reserved items back to stock
```

### 2. Payment Gateway Timeout

```
Problem: Called gateway, no response. Was payment processed?

Solution:
1. Check idempotency: Query gateway with idempotency key
2. If exists: Return existing result
3. If not exists: Retry with same idempotency key
```

### 3. Double Refund

```
Problem: User clicks refund twice

Solution: Idempotency keys on refunds
```

## Monitoring

```yaml
Metrics:
  - payment_success_rate: 99%+
  - payment_latency_p95: < 500ms
  - authorization_decline_rate: 5-10%
  - refund_rate: < 2%
  - reconciliation_discrepancies: 0

Alerts:
  - name: HighDeclineRate
    condition: decline_rate > 20%
    severity: warning

  - name: ReconciliationMismatch
    condition: discrepancies > 0
    severity: critical
    action: Manual review required

  - name: PaymentGatewayDown
    condition: gateway_error_rate > 50%
    severity: critical
```

## Summary

### What We Built:
- Payment processing with exactly-once semantics
- Saga pattern for distributed transactions
- Idempotency for duplicate prevention
- Event sourcing for audit trail
- Reconciliation system

### Key Concepts:
- ✅ Idempotency keys
- ✅ Saga pattern (orchestration & choreography)
- ✅ Event sourcing
- ✅ Outbox pattern
- ✅ Money handling (use integers!)
- ✅ Reconciliation

### Real-World Examples:
- Stripe: 10K+ TPS, 99.999% uptime
- PayPal: $1T+ annual volume
- Square: Real-time authorization

---

**Next Project**: [09. API Gateway with Resilience](../09-api-gateway/README.md)
