# Project 09: API Gateway with Resilience Patterns

## Difficulty: Intermediate
**Estimated Time**: 2-3 weeks

## Problem Statement

Design an API Gateway like Kong, AWS API Gateway, or Apigee that serves as the single entry point for microservices, implements resilience patterns (circuit breaker, retry, timeout), handles authentication, and provides observability.

**Real-World Analogy**: The API Gateway is like a hotel concierge who handles all guest requests, routes them to appropriate departments, and ensures smooth service even when some departments are unavailable.

## Functional Requirements

1. **Request Routing**: Route requests to appropriate backend services
2. **Authentication**: Validate JWT tokens, API keys
3. **Rate Limiting**: Protect backends from overload
4. **Circuit Breaker**: Stop calling failing services
5. **Request/Response Transformation**: Modify headers, body
6. **Load Balancing**: Distribute load across service instances
7. **Caching**: Cache frequent responses
8. **Observability**: Logging, metrics, tracing

## Resilience Patterns

### 1. Circuit Breaker Pattern

**Problem**: When a service fails, stop sending requests (avoid cascading failures)

**States**: CLOSED → OPEN → HALF_OPEN → CLOSED

```typescript
class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime?: number;

  constructor(
    private threshold = 5,        // Open after 5 failures
    private timeout = 60000,      // Try again after 60s
    private halfOpenAttempts = 3  // Test with 3 requests
  ) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      // Check if timeout elapsed
      if (Date.now() - this.lastFailureTime! < this.timeout) {
        throw new Error('Circuit breaker is OPEN');
      }
      // Timeout elapsed, try half-open
      this.state = 'HALF_OPEN';
      this.successCount = 0;
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;

    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.halfOpenAttempts) {
        this.state = 'CLOSED';
      }
    }
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}
```

### 2. Retry Pattern with Backoff

```typescript
async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  maxAttempts = 3,
  baseDelay = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxAttempts || !isRetryable(error)) {
        throw error;
      }

      // Exponential backoff with jitter
      const delay = baseDelay * Math.pow(2, attempt - 1);
      const jitter = Math.random() * 1000;
      await sleep(delay + jitter);
    }
  }
}

function isRetryable(error: Error): boolean {
  // Retry on network errors, 5xx, but not 4xx
  return error.code === 'ETIMEDOUT' ||
         error.code === 'ECONNRESET' ||
         (error.status >= 500 && error.status < 600);
}
```

### 3. Timeout Pattern

```typescript
async function withTimeout<T>(
  operation: Promise<T>,
  timeoutMs: number
): Promise<T> {
  return Promise.race([
    operation,
    new Promise<T>((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), timeoutMs)
    )
  ]);
}

// Usage:
const response = await withTimeout(
  fetch('http://slow-service.com/api'),
  5000  // 5 second timeout
);
```

### 4. Bulkhead Pattern

**Concept**: Isolate resources so failure in one doesn't affect others

```typescript
class Bulkhead {
  private activeRequests = 0;
  private queue: Array<() => void> = [];

  constructor(private maxConcurrent: number) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    // Wait for available slot
    await this.acquire();

    try {
      return await operation();
    } finally {
      this.release();
    }
  }

  private async acquire(): Promise<void> {
    if (this.activeRequests < this.maxConcurrent) {
      this.activeRequests++;
      return;
    }

    // Wait in queue
    return new Promise<void>(resolve => {
      this.queue.push(resolve);
    });
  }

  private release(): void {
    this.activeRequests--;

    if (this.queue.length > 0) {
      const next = this.queue.shift()!;
      this.activeRequests++;
      next();
    }
  }
}

// Usage: Separate bulkheads for different services
const userServiceBulkhead = new Bulkhead(100);
const paymentServiceBulkhead = new Bulkhead(50);

// If payment service is slow, doesn't affect user service
```

## Architecture

```
           Client
             │
             ▼
    ┌────────────────┐
    │  API Gateway   │
    │  ┌──────────┐  │
    │  │Auth      │  │
    │  │Rate Limit│  │
    │  │Circuit   │  │
    │  │Breaker   │  │
    │  └──────────┘  │
    └────────┬───────┘
             │
    ┌────────┴────────┬──────────────┐
    ▼                 ▼              ▼
┌────────┐      ┌──────────┐   ┌──────────┐
│ Users  │      │ Products │   │ Orders   │
│Service │      │ Service  │   │ Service  │
└────────┘      └──────────┘   └──────────┘
```

## API Design

```http
# Gateway configuration
POST /api/v1/routes
Body: {
  "path": "/api/users/*",
  "backend": "http://user-service:8080",
  "methods": ["GET", "POST"],
  "auth": "jwt",
  "rate_limit": {
    "requests_per_minute": 100
  },
  "circuit_breaker": {
    "threshold": 5,
    "timeout": 60000
  }
}

# Request through gateway
GET /api/users/123
X-API-Key: abc123

→ Gateway routes to: http://user-service:8080/123
→ With resilience patterns applied
```

## Summary

### Key Concepts:
- ✅ Circuit Breaker (prevent cascading failures)
- ✅ Retry with exponential backoff
- ✅ Timeouts
- ✅ Bulkhead pattern (resource isolation)
- ✅ Rate limiting
- ✅ Request routing

### Real-World Examples:
- Kong: 10K+ RPS per instance
- AWS API Gateway: Managed, auto-scaling
- Envoy: Service mesh proxy

---

**Next Project**: [10. Multi-Region Active-Active System](../10-multi-region-system/README.md)
