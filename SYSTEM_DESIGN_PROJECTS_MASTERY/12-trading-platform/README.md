# Project 12: High-Frequency Trading Platform

## Difficulty: Expert
**Estimated Time**: 5-6 weeks

## Problem Statement

Design a high-frequency trading platform that executes trades in microseconds, maintains order book consistency, handles millions of orders per second, and ensures financial accuracy.

**Challenge**: Ultra-low latency (<1ms), strong consistency, high throughput

## Functional Requirements

1. **Order Management**: Place, cancel, modify orders
2. **Order Book**: Maintain buy/sell order books per symbol
3. **Matching Engine**: Match orders at best price
4. **Market Data**: Real-time price updates
5. **Risk Management**: Pre-trade checks, position limits
6. **Audit Trail**: Complete trade history

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Latency** | < 1ms | Order placement to execution |
| **Throughput** | 1M orders/sec | Peak capacity |
| **Consistency** | 100% | Exactly-once execution |
| **Availability** | 99.999% | < 5 min downtime/year |
| **Audit** | 100% | All trades logged |

## Order Book Data Structure

```typescript
interface Order {
  orderId: string;
  symbol: string;
  side: 'BUY' | 'SELL';
  price: number;
  quantity: number;
  timestamp: number;
}

class OrderBook {
  private bids: PriorityQueue<Order>;  // Max heap (highest price first)
  private asks: PriorityQueue<Order>;  // Min heap (lowest price first)

  constructor() {
    this.bids = new PriorityQueue((a, b) => b.price - a.price);
    this.asks = new PriorityQueue((a, b) => a.price - b.price);
  }

  addOrder(order: Order) {
    if (order.side === 'BUY') {
      this.bids.push(order);
    } else {
      this.asks.push(order);
    }
  }

  matchOrders(): Trade[] {
    const trades: Trade[] = [];

    while (this.bids.length > 0 && this.asks.length > 0) {
      const topBid = this.bids.peek();
      const topAsk = this.asks.peek();

      // Can match if bid price >= ask price
      if (topBid.price >= topAsk.price) {
        const quantity = Math.min(topBid.quantity, topAsk.quantity);
        const price = topAsk.price;  // Taker pays maker's price

        trades.push({
          buyOrderId: topBid.orderId,
          sellOrderId: topAsk.orderId,
          price,
          quantity,
          timestamp: Date.now()
        });

        // Update quantities
        topBid.quantity -= quantity;
        topAsk.quantity -= quantity;

        // Remove filled orders
        if (topBid.quantity === 0) this.bids.pop();
        if (topAsk.quantity === 0) this.asks.pop();
      } else {
        break;  // No more matches possible
      }
    }

    return trades;
  }
}
```

## Low-Latency Optimizations

### 1. Memory Layout

```cpp
// Cache-line aligned struct (64 bytes)
struct __attribute__((aligned(64))) Order {
    uint64_t orderId;
    uint32_t price;      // Fixed-point (price * 10000)
    uint32_t quantity;
    uint64_t timestamp;
    uint8_t  side;       // 0=BUY, 1=SELL
    uint8_t  padding[39]; // Align to 64 bytes
};

// Benefits:
// - Fits in single cache line
// - No false sharing between CPU cores
// - Predictable memory access patterns
```

### 2. Lock-Free Data Structures

```cpp
#include <atomic>

template<typename T>
class LockFreeQueue {
private:
    struct Node {
        T data;
        std::atomic<Node*> next;
    };

    std::atomic<Node*> head;
    std::atomic<Node*> tail;

public:
    void enqueue(T data) {
        Node* node = new Node{data, nullptr};
        Node* oldTail = tail.load();

        // Atomic compare-and-swap
        while (!tail.compare_exchange_weak(oldTail, node)) {
            // Retry if another thread modified tail
        }

        oldTail->next.store(node);
    }

    bool dequeue(T& result) {
        Node* oldHead = head.load();
        Node* next = oldHead->next.load();

        if (next == nullptr) {
            return false;  // Queue empty
        }

        result = next->data;
        head.store(next);
        delete oldHead;
        return true;
    }
};
```

### 3. CPU Pinning

```cpp
#include <pthread.h>
#include <sched.h>

void pinToCPU(int cpuId) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpuId, &cpuset);

    pthread_t thread = pthread_self();
    pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
}

// Pin matching engine to dedicated CPU core
// Avoid context switches, cache misses
```

### 4. Kernel Bypass Networking

```
Traditional network stack:
  App → System call → Kernel → Network driver → NIC
  Latency: ~50-100μs

Kernel bypass (DPDK, Solarflare):
  App → User-space driver → NIC
  Latency: ~1-5μs

10-50x faster!
```

## Matching Engine

```typescript
class MatchingEngine {
  private orderBooks = new Map<string, OrderBook>();

  async processOrder(order: Order): Promise<MatchResult> {
    // 1. Pre-trade risk checks
    await this.riskCheck(order);

    // 2. Get order book for symbol
    let book = this.orderBooks.get(order.symbol);
    if (!book) {
      book = new OrderBook();
      this.orderBooks.set(order.symbol, book);
    }

    // 3. Add order to book
    book.addOrder(order);

    // 4. Match orders
    const trades = book.matchOrders();

    // 5. Update positions
    for (const trade of trades) {
      await this.updatePositions(trade);
      await this.publishTrade(trade);
    }

    return {
      orderId: order.orderId,
      status: trades.length > 0 ? 'FILLED' : 'OPEN',
      fills: trades
    };
  }

  private async riskCheck(order: Order): Promise<void> {
    // Check account balance
    const balance = await this.getAccountBalance(order.userId);
    const cost = order.price * order.quantity;

    if (balance < cost) {
      throw new Error('INSUFFICIENT_BALANCE');
    }

    // Check position limits
    const position = await this.getPosition(order.userId, order.symbol);
    if (Math.abs(position + order.quantity) > MAX_POSITION) {
      throw new Error('POSITION_LIMIT_EXCEEDED');
    }
  }
}
```

## Time Synchronization

### Problem: Clock Skew

```
Server 1 time: 10:00:00.000000
Server 2 time: 10:00:00.000100 (100μs ahead)

Order arrives at Server 2, timestamp: 10:00:00.000100
Order arrives at Server 1, timestamp: 10:00:00.000050

Which order is first? Wrong answer = wrong trade execution!
```

### Solution: Hardware Timestamps

```cpp
#include <time.h>

uint64_t getHardwareTimestamp() {
    struct timespec ts;

    // Use hardware clock (from NIC)
    clock_gettime(CLOCK_REALTIME_COARSE, &ts);

    return ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}

// Precision: ~10ns
// Synchronized via PTP (Precision Time Protocol)
```

## Market Data Distribution

```typescript
class MarketDataPublisher {
  private subscribers = new Map<string, Set<WebSocket>>();

  subscribe(symbol: string, ws: WebSocket) {
    if (!this.subscribers.has(symbol)) {
      this.subscribers.set(symbol, new Set());
    }
    this.subscribers.get(symbol)!.add(ws);
  }

  publishTrade(trade: Trade) {
    const subscribers = this.subscribers.get(trade.symbol);

    if (!subscribers) return;

    const message = JSON.stringify({
      type: 'TRADE',
      symbol: trade.symbol,
      price: trade.price,
      quantity: trade.quantity,
      timestamp: trade.timestamp
    });

    // Broadcast to all subscribers
    for (const ws of subscribers) {
      ws.send(message);
    }
  }

  publishOrderBook(symbol: string, book: OrderBookSnapshot) {
    const subscribers = this.subscribers.get(symbol);

    if (!subscribers) return;

    const message = JSON.stringify({
      type: 'ORDER_BOOK',
      symbol,
      bids: book.bids.slice(0, 10),  // Top 10 levels
      asks: book.asks.slice(0, 10),
      timestamp: Date.now()
    });

    for (const ws of subscribers) {
      ws.send(message);
    }
  }
}
```

## Summary

### What We Built:
- High-frequency trading platform
- Order book with priority queues
- Matching engine with <1ms latency
- Low-latency optimizations (cache alignment, lock-free)
- Real-time market data distribution

### Key Concepts:
- ✅ Order book data structure
- ✅ Order matching algorithm
- ✅ Memory optimization (cache lines)
- ✅ Lock-free concurrent data structures
- ✅ CPU pinning and kernel bypass
- ✅ Time synchronization

### Real-World Examples:
- NASDAQ: 3M orders/sec, 10μs latency
- CME: 7M messages/sec
- NYSE: 100M+ orders/day

---

**Next Project**: [13. APM & Observability Platform](../13-apm-platform/README.md)
