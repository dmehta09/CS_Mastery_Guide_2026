# Project 04: Payment Processing Engine

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language
**Prerequisites**: Distributed Transactions, Event Sourcing, Idempotency

---

## What You're Building

A reliable, distributed payment processing system similar to Stripe's backend or PayPal's transaction engine. This system must handle money transfers with absolute correctness - no double charges, no lost payments, and perfect audit trails.

---

## Layman Explanation

Think of this like a **bank's money transfer system**:

1. Customer says "I want to pay $100 to Merchant X"
2. System checks: Does customer have $100? Is merchant valid?
3. **Critical**: Money must either be fully transferred or not at all - never half-transferred
4. If anything fails (network crash, power outage), the system must recover safely
5. Every single cent must be traceable and auditable
6. Must prevent: charging twice, losing money, or inconsistent balances

This is one of the hardest problems in backend engineering because **money can't disappear** and **you can't charge twice**.

---

## Real-World Use Cases

1. **E-Commerce**: Processing customer payments
2. **Marketplaces**: Split payments between platform and sellers
3. **Subscriptions**: Recurring billing systems
4. **International Transfers**: Multi-currency transactions
5. **Peer-to-Peer**: Venmo, Cash App style transfers

---

## Key Concepts You'll Learn

### 1. Distributed Transactions
- **Two-Phase Commit (2PC)**: Coordinated commits
- **Saga Pattern**: Long-running transactions
- **Compensating Transactions**: Undoing operations
- **Transaction Coordinator**: Orchestrating distributed ops

### 2. Idempotency
- **Idempotent Keys**: Prevent duplicate processing
- **Request Deduplication**: Detecting retries
- **State Machine**: Tracking transaction states
- **Exactly-Once Semantics**: Despite at-least-once delivery

### 3. Event Sourcing
- **Immutable Event Log**: All changes as events
- **Event Replay**: Reconstructing state
- **Audit Trail**: Complete history
- **Time Travel**: View state at any point

### 4. Financial Patterns
- **Double-Entry Bookkeeping**: Every debit has a credit
- **Ledger Design**: Immutable transaction records
- **Reconciliation**: Matching records
- **Money Type**: Handling decimals correctly

### 5. Reliability Patterns
- **Circuit Breaker**: Fail fast on downstream failures
- **Retry with Backoff**: Handle transient failures
- **Timeout**: Don't wait forever
- **Dead Letter Queue**: Handle permanent failures

### 6. Security
- **PCI DSS Compliance**: Credit card security
- **Encryption at Rest**: Protect sensitive data
- **Key Management**: Secure key storage
- **Fraud Detection**: Anomaly detection

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         CLIENT APPS                               │
│                  (Mobile, Web, Backend Services)                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY                                 │
│              (Rate Limiting, Auth, Idempotency)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   PAYMENT ORCHESTRATOR                           │
│              (Saga Coordinator, State Machine)                   │
└───────┬──────────┬──────────┬──────────┬──────────┬─────────────┘
        │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Account │  │ Payment │  │Fraud    │  │ Ledger  │  │Notifi-  │
│ Service │  │ Gateway │  │Detection│  │ Service │  │cation   │
└────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
     │            │            │            │            │
     ▼            ▼            ▼            ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│Account  │  │Stripe/  │  │ ML      │  │Event    │  │Email/   │
│   DB    │  │PayPal   │  │ Model   │  │ Store   │  │SMS      │
└─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘

           ┌──────────────────────────────────┐
           │        MESSAGE QUEUE (Kafka)      │
           │    (Event Bus, State Tracking)    │
           └──────────────────────────────────┘
```

---

## Saga Pattern Implementation

### What is a Saga?

A saga is a sequence of local transactions where each transaction updates a single service. If one transaction fails, the saga executes compensating transactions to undo the changes.

### Example: Payment Transaction Saga

```
Happy Path:
1. Reserve Funds (Account Service)
2. Validate Payment (Fraud Detection)
3. Charge Card (Payment Gateway)
4. Record Transaction (Ledger Service)
5. Commit Funds (Account Service)
6. Send Confirmation (Notification Service)

Failure Path (if step 3 fails):
1. Unreserve Funds (Compensating Transaction)
2. Mark Payment Failed (Ledger)
3. Send Failure Notification
```

### Saga State Machine

```
States:
- INITIATED: Saga started
- FUNDS_RESERVED: Step 1 complete
- FRAUD_CHECKED: Step 2 complete
- CARD_CHARGED: Step 3 complete
- LEDGER_RECORDED: Step 4 complete
- COMPLETED: All steps done
- COMPENSATING: Rolling back
- FAILED: Saga failed
```

---

## Database Schema

### Event Store (PostgreSQL)

```sql
-- Immutable event log
CREATE TABLE payment_events (
    id BIGSERIAL PRIMARY KEY,
    payment_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    metadata JSONB,

    UNIQUE(payment_id, version)
);

-- Example events:
-- PaymentInitiated, FundsReserved, FraudCheckPassed,
-- CardCharged, LedgerRecorded, PaymentCompleted,
-- PaymentFailed, FundsUnreserved

CREATE INDEX idx_payment_id ON payment_events(payment_id);
CREATE INDEX idx_event_type ON payment_events(event_type);
CREATE INDEX idx_created_at ON payment_events(created_at);
```

### Ledger (PostgreSQL)

```sql
-- Immutable ledger entries
CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    transaction_id UUID NOT NULL,
    account_id BIGINT NOT NULL,
    amount DECIMAL(19, 4) NOT NULL, -- NEVER use FLOAT for money
    currency VARCHAR(3) NOT NULL,
    entry_type VARCHAR(10) NOT NULL, -- 'DEBIT' or 'CREDIT'
    balance_after DECIMAL(19, 4) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    metadata JSONB
);

CREATE INDEX idx_transaction_id ON ledger_entries(transaction_id);
CREATE INDEX idx_account_id ON ledger_entries(account_id);
CREATE INDEX idx_created_at ON ledger_entries(created_at);

-- Double-entry bookkeeping: Every payment creates TWO entries
-- DEBIT from payer's account
-- CREDIT to payee's account
```

### Accounts (PostgreSQL)

```sql
CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    balance DECIMAL(19, 4) NOT NULL DEFAULT 0,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    version INT NOT NULL DEFAULT 0, -- For optimistic locking
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Reserved funds (for pending transactions)
CREATE TABLE account_holds (
    id BIGSERIAL PRIMARY KEY,
    account_id BIGINT NOT NULL REFERENCES accounts(id),
    payment_id UUID NOT NULL UNIQUE,
    amount DECIMAL(19, 4) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### Idempotency (Redis)

```
Key: idempotency:{idempotency_key}
Value: {
  "status": "PROCESSING" | "COMPLETED" | "FAILED",
  "payment_id": "uuid",
  "response": { /* cached response */ }
}
TTL: 24 hours
```

---

## API Design

### 1. Create Payment

```http
POST /api/v1/payments
Content-Type: application/json
Idempotency-Key: unique-key-12345
Authorization: Bearer <token>

{
  "amount": 10000, // Amount in cents (avoid floating point)
  "currency": "USD",
  "payment_method": "card_token_abc123",
  "recipient": "merchant_xyz",
  "description": "Purchase of Product X",
  "metadata": {
    "order_id": "order-123",
    "customer_email": "user@example.com"
  }
}

Response 201:
{
  "payment_id": "pay_abc123def456",
  "status": "processing",
  "amount": 10000,
  "currency": "USD",
  "created_at": "2026-02-16T10:30:00Z",
  "estimated_completion": "2026-02-16T10:30:30Z"
}
```

### 2. Get Payment Status

```http
GET /api/v1/payments/pay_abc123def456
Authorization: Bearer <token>

Response 200:
{
  "payment_id": "pay_abc123def456",
  "status": "succeeded",
  "amount": 10000,
  "currency": "USD",
  "payment_method": "card_****1234",
  "recipient": "merchant_xyz",
  "created_at": "2026-02-16T10:30:00Z",
  "completed_at": "2026-02-16T10:30:15Z",
  "fees": 300, // Platform fee in cents
  "net_amount": 9700,
  "events": [
    {
      "type": "payment.initiated",
      "timestamp": "2026-02-16T10:30:00Z"
    },
    {
      "type": "payment.succeeded",
      "timestamp": "2026-02-16T10:30:15Z"
    }
  ]
}
```

### 3. Refund Payment

```http
POST /api/v1/payments/pay_abc123def456/refund
Content-Type: application/json
Idempotency-Key: refund-unique-key-67890
Authorization: Bearer <token>

{
  "amount": 10000, // Full or partial refund
  "reason": "customer_request"
}

Response 201:
{
  "refund_id": "ref_xyz789abc123",
  "payment_id": "pay_abc123def456",
  "amount": 10000,
  "status": "processing",
  "created_at": "2026-02-16T11:00:00Z"
}
```

---

## Idempotency Implementation

### Why Idempotency?

Networks are unreliable. A client might:
1. Send payment request
2. Connection drops before response
3. Client retries
4. Without idempotency: **Customer charged twice!**

### Implementation

```typescript
class PaymentController {
  async createPayment(
    req: Request,
    idempotencyKey: string
  ) {
    // 1. Check if we've seen this request before
    const cached = await redis.get(
      `idempotency:${idempotencyKey}`
    );

    if (cached) {
      const data = JSON.parse(cached);

      if (data.status === "COMPLETED") {
        // Return cached response
        return data.response;
      }

      if (data.status === "PROCESSING") {
        // Still processing, ask client to poll
        return {
          status: "processing",
          payment_id: data.payment_id,
          message: "Payment is being processed"
        };
      }
    }

    // 2. First time seeing this request
    const paymentId = generateUUID();

    // 3. Mark as processing
    await redis.set(
      `idempotency:${idempotencyKey}`,
      JSON.stringify({
        status: "PROCESSING",
        payment_id: paymentId,
        timestamp: Date.now()
      }),
      "EX",
      86400 // 24 hours
    );

    try {
      // 4. Process payment
      const result = await this.paymentService.process({
        payment_id: paymentId,
        ...req.body
      });

      // 5. Cache successful response
      await redis.set(
        `idempotency:${idempotencyKey}`,
        JSON.stringify({
          status: "COMPLETED",
          payment_id: paymentId,
          response: result,
          timestamp: Date.now()
        }),
        "EX",
        86400
      );

      return result;
    } catch (error) {
      // 6. Mark as failed
      await redis.set(
        `idempotency:${idempotencyKey}`,
        JSON.stringify({
          status: "FAILED",
          payment_id: paymentId,
          error: error.message,
          timestamp: Date.now()
        }),
        "EX",
        86400
      );

      throw error;
    }
  }
}
```

---

## Money Type Implementation

### NEVER use Float for Money!

```typescript
// ❌ WRONG - Floating point errors
const amount = 0.1 + 0.2; // 0.30000000000000004

// ✅ CORRECT - Use integers (cents)
const amount = 10 + 20; // 30 cents = $0.30

// ✅ CORRECT - Use Decimal library
import Decimal from "decimal.js";
const amount = new Decimal("0.1")
  .plus(new Decimal("0.2")); // 0.3
```

### Money Class

```typescript
class Money {
  private amountInCents: number;
  private currency: string;

  constructor(amountInCents: number, currency: string) {
    if (!Number.isInteger(amountInCents)) {
      throw new Error("Amount must be integer (cents)");
    }
    this.amountInCents = amountInCents;
    this.currency = currency;
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(
      this.amountInCents + other.amountInCents,
      this.currency
    );
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(
      this.amountInCents - other.amountInCents,
      this.currency
    );
  }

  multiply(factor: number): Money {
    return new Money(
      Math.round(this.amountInCents * factor),
      this.currency
    );
  }

  divide(divisor: number): Money {
    return new Money(
      Math.round(this.amountInCents / divisor),
      this.currency
    );
  }

  toString(): string {
    return `${(this.amountInCents / 100).toFixed(2)} ${this.currency}`;
  }

  private assertSameCurrency(other: Money) {
    if (this.currency !== other.currency) {
      throw new Error(
        `Currency mismatch: ${this.currency} vs ${other.currency}`
      );
    }
  }
}

// Usage
const price = new Money(10000, "USD"); // $100.00
const tax = price.multiply(0.08); // $8.00
const total = price.add(tax); // $108.00
```

---

## Saga Orchestrator Implementation

```typescript
class PaymentSaga {
  private state: SagaState = "INITIATED";
  private paymentId: string;
  private compensations: (() => Promise<void>)[] = [];

  async execute(payment: PaymentRequest) {
    try {
      // Step 1: Reserve funds
      await this.reserveFunds(payment);
      this.state = "FUNDS_RESERVED";

      // Step 2: Fraud check
      await this.checkFraud(payment);
      this.state = "FRAUD_CHECKED";

      // Step 3: Charge card
      await this.chargeCard(payment);
      this.state = "CARD_CHARGED";

      // Step 4: Record in ledger
      await this.recordLedger(payment);
      this.state = "LEDGER_RECORDED";

      // Step 5: Commit funds
      await this.commitFunds(payment);
      this.state = "COMPLETED";

      // Step 6: Send notification
      await this.sendNotification(payment);

      return { status: "succeeded", paymentId: this.paymentId };
    } catch (error) {
      // Compensation phase
      await this.compensate();
      throw error;
    }
  }

  private async reserveFunds(payment: PaymentRequest) {
    const result = await accountService.reserve({
      account_id: payment.payer_account,
      amount: payment.amount,
      payment_id: this.paymentId
    });

    // Register compensation
    this.compensations.push(async () => {
      await accountService.unreserve({
        payment_id: this.paymentId
      });
    });

    await this.publishEvent("FundsReserved", result);
  }

  private async chargeCard(payment: PaymentRequest) {
    const result = await paymentGateway.charge({
      payment_method: payment.payment_method,
      amount: payment.amount,
      idempotency_key: this.paymentId
    });

    // Register compensation (refund)
    this.compensations.push(async () => {
      await paymentGateway.refund({
        charge_id: result.charge_id
      });
    });

    await this.publishEvent("CardCharged", result);
  }

  private async compensate() {
    this.state = "COMPENSATING";

    // Execute compensations in reverse order
    for (const compensation of this.compensations.reverse()) {
      try {
        await compensation();
      } catch (error) {
        // Log but continue compensating
        logger.error("Compensation failed", error);
      }
    }

    this.state = "FAILED";
    await this.publishEvent("PaymentFailed", {
      payment_id: this.paymentId,
      reason: "Saga compensation completed"
    });
  }
}
```

---

## Testing Strategy

### Unit Tests
- Money class arithmetic
- State machine transitions
- Compensation logic

### Integration Tests
- Full saga execution
- Idempotency key handling
- Event publishing

### End-to-End Tests
```typescript
test("Payment succeeds end-to-end", async () => {
  const idempotencyKey = uuid();

  // Create payment
  const payment = await api.createPayment(
    {
      amount: 10000,
      currency: "USD",
      payment_method: "test_card"
    },
    idempotencyKey
  );

  expect(payment.status).toBe("processing");

  // Poll until complete
  const completed = await waitFor(
    () => api.getPayment(payment.payment_id),
    (p) => p.status === "succeeded",
    { timeout: 30000 }
  );

  expect(completed.status).toBe("succeeded");

  // Verify ledger entries
  const ledger = await api.getLedger(payment.payment_id);
  expect(ledger.entries).toHaveLength(2); // Debit + Credit

  // Verify idempotency
  const duplicate = await api.createPayment(
    {
      amount: 10000,
      currency: "USD",
      payment_method: "test_card"
    },
    idempotencyKey // Same key!
  );

  expect(duplicate.payment_id).toBe(payment.payment_id);
});
```

### Chaos Engineering
```typescript
// Test saga compensation
test("Saga compensates on card charge failure", async () => {
  // Inject failure
  paymentGateway.setFailureMode("charge", true);

  const payment = await api.createPayment({
    amount: 10000,
    currency: "USD",
    payment_method: "test_card"
  });

  // Wait for failure
  await waitFor(
    () => api.getPayment(payment.payment_id),
    (p) => p.status === "failed"
  );

  // Verify compensation executed
  const holds = await accountService.getHolds(
    payment.payer_account
  );
  expect(holds).toHaveLength(0); // Funds unreserved
});
```

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Payment Processing Time | < 5s |
| Throughput | 1000 payments/sec |
| Saga Compensation Time | < 10s |
| Idempotency Check | < 10ms |
| Event Publishing | < 50ms |
| Data Consistency | 100% |

---

## Security Considerations

### PCI DSS Compliance
- Never store raw credit card numbers
- Use tokenization (Stripe, PayPal)
- Encrypt sensitive data
- Regular security audits

### Fraud Detection
- Velocity checks (too many attempts)
- Geolocation mismatch
- Unusual amounts
- ML-based risk scoring

### Access Control
- Role-based permissions
- API key management
- IP whitelisting
- Rate limiting per account

---

## Success Criteria

✅ Zero double charges
✅ Zero lost payments
✅ 100% audit trail
✅ Idempotency working correctly
✅ Saga compensation tested
✅ Sub-5s payment processing
✅ Comprehensive test coverage
✅ Security audit passed

---

## Next Project

After completing this advanced project, move to **Project 05: Stream Processing Analytics Platform** to learn about real-time data processing at scale.

---

Happy Building! 🚀
