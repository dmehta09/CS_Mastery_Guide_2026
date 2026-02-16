# Project 14: Streaming Analytics Platform

## Difficulty: Advanced
**Estimated Time**: 4-5 weeks

## Problem Statement

Design a real-time streaming analytics platform like Apache Flink, Kafka Streams, or AWS Kinesis Analytics that processes millions of events per second, performs windowed aggregations, and maintains stateful computations.

**Use Cases**: Real-time dashboards, fraud detection, IoT analytics, clickstream analysis

## Functional Requirements

1. **Event Ingestion**: Ingest millions of events/sec
2. **Stream Processing**: Transform, filter, aggregate events
3. **Windowing**: Time-based, session-based windows
4. **Stateful Processing**: Maintain state across events
5. **Exactly-Once Processing**: No duplicate processing
6. **Low Latency**: Sub-second processing

## Architecture

```
Event Sources
├─ IoT Devices
├─ Clickstream
├─ Application Logs
└─ Financial Transactions
       │
       ▼
┌──────────────────┐
│  Apache Kafka    │
│  (Message Broker)│
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│   Stream Processor           │
│   (Flink / Kafka Streams)    │
│                              │
│  ┌────────────────────────┐ │
│  │ Windowing & Aggregation│ │
│  └────────────────────────┘ │
│  ┌────────────────────────┐ │
│  │   Stateful Operators   │ │
│  └────────────────────────┘ │
└────────┬─────────────────────┘
         │
         ├─────────┬──────────┐
         ▼         ▼          ▼
   ┌─────────┐ ┌───────┐ ┌────────┐
   │Dashboard│ │Alerts │ │Storage │
   └─────────┘ └───────┘ └────────┘
```

## Streaming Concepts

### 1. Windowing

**Tumbling Window**: Fixed, non-overlapping intervals

```
Window size: 1 minute

Events: [1, 5, 12, 18, 25, 62, 70]

Windows:
[0-60s]:  [1, 5, 12, 18, 25]  → Count: 5
[60-120s]: [62, 70]            → Count: 2
```

**Sliding Window**: Overlapping intervals

```
Window size: 10 seconds
Slide: 5 seconds

Windows:
[0-10s]:   [events...]
[5-15s]:   [events...]  (overlaps with previous)
[10-20s]:  [events...]
```

**Session Window**: Dynamic, based on inactivity

```
Gap: 5 minutes

User clicks: 10:00, 10:02, 10:04 → Session 1
User idle for 5+ minutes
User clicks: 10:15, 10:17 → Session 2
```

### 2. Watermarks

**Problem**: Events arrive out of order

```
Wall clock time: 10:05
Events received:
  - Event A: timestamp 10:00 (5 min old)
  - Event B: timestamp 10:04 (1 min old)
  - Event C: timestamp 10:02 (3 min old, out of order!)

Question: When to close window [10:00-10:05]?
```

**Solution**: Watermarks

```typescript
class Watermark {
  private watermark: number = 0;

  update(eventTime: number, maxLateness: number) {
    // Watermark = max event time - max lateness
    this.watermark = Math.max(this.watermark, eventTime - maxLateness);
  }

  shouldCloseWindow(windowEnd: number): boolean {
    return this.watermark >= windowEnd;
  }
}

// Example:
maxLateness = 10 seconds

Event times: 100, 105, 102, 110
Watermarks:   90,  95,  92, 100

Window [100-110]:
- Watermark reaches 100 → Close window
- Event with time 102 arrives → Late event, discard or side output
```

## Kafka Streams Implementation

### Word Count Example

```typescript
import { KafkaStreams } from 'kafka-streams';

const kafkaStreams = new KafkaStreams({
  noptions: {
    'metadata.broker.list': 'localhost:9092',
    'group.id': 'word-count',
    'client.id': 'word-count-client'
  }
});

const stream = kafkaStreams.getKStream('input-topic');

stream
  // 1. Parse JSON
  .mapJSONConvenience()

  // 2. Extract text field
  .map(event => event.text)

  // 3. Split into words
  .flatMap(text => text.toLowerCase().split(/\s+/))

  // 4. Filter empty words
  .filter(word => word.length > 0)

  // 5. Count occurrences (stateful!)
  .countByKey('word', 60000)  // 60 second tumbling window

  // 6. Output results
  .to('output-topic');

stream.start();
```

### Clickstream Analytics

```typescript
interface ClickEvent {
  userId: string;
  pageUrl: string;
  timestamp: number;
  sessionId?: string;
}

const clickStream = kafkaStreams.getKStream<ClickEvent>('clicks');

// Session window: Group clicks into sessions
clickStream
  .map(event => ({
    key: event.userId,
    value: event
  }))

  // Session window with 30 minute gap
  .window({
    type: 'session',
    gap: 30 * 60 * 1000  // 30 minutes
  })

  // Aggregate clicks per session
  .reduce((agg, event) => {
    return {
      userId: event.userId,
      sessionStart: Math.min(agg?.sessionStart || Infinity, event.timestamp),
      sessionEnd: Math.max(agg?.sessionEnd || 0, event.timestamp),
      pageViews: (agg?.pageViews || 0) + 1,
      pages: [...(agg?.pages || []), event.pageUrl]
    };
  })

  .to('sessions-topic');
```

## Apache Flink Implementation

### Real-Time Dashboard

```java
// Source: Read from Kafka
DataStream<Event> events = env
    .addSource(new FlinkKafkaConsumer<>("events", schema, properties));

// Tumbling window: 1 minute
events
    .keyBy(event -> event.getProductId())
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new SalesAggregator())
    .addSink(new DashboardSink());

class SalesAggregator implements AggregateFunction<Event, SalesStats, SalesStats> {
    @Override
    public SalesStats createAccumulator() {
        return new SalesStats(0, 0.0);
    }

    @Override
    public SalesStats add(Event event, SalesStats acc) {
        return new SalesStats(
            acc.count + 1,
            acc.revenue + event.getAmount()
        );
    }

    @Override
    public SalesStats merge(SalesStats a, SalesStats b) {
        return new SalesStats(
            a.count + b.count,
            a.revenue + b.revenue
        );
    }

    @Override
    public SalesStats getResult(SalesStats acc) {
        return acc;
    }
}
```

### Fraud Detection

```java
// Pattern matching: Detect suspicious transactions
Pattern<Transaction, ?> pattern = Pattern
    .<Transaction>begin("first")
    .where(new SimpleCondition<Transaction>() {
        @Override
        public boolean filter(Transaction t) {
            return t.amount > 1000;  // Large transaction
        }
    })
    .next("second")
    .where(new SimpleCondition<Transaction>() {
        @Override
        public boolean filter(Transaction t) {
            return t.amount > 1000;
        }
    })
    .within(Time.minutes(10));  // Two large transactions within 10 min

PatternStream<Transaction> patternStream = CEP.pattern(
    transactions.keyBy(Transaction::getUserId),
    pattern
);

DataStream<Alert> alerts = patternStream
    .select((PatternSelectFunction<Transaction, Alert>) pattern -> {
        Transaction first = pattern.get("first").get(0);
        Transaction second = pattern.get("second").get(0);

        return new Alert(
            first.getUserId(),
            "Suspicious activity: Two large transactions in 10 minutes",
            Arrays.asList(first, second)
        );
    });

alerts.addSink(new AlertSink());
```

## Stateful Processing

### State Backend

```java
// Configure state backend (where to store state)
env.setStateBackend(new RocksDBStateBackend("hdfs:///flink/checkpoints"));

// Enable checkpointing (for fault tolerance)
env.enableCheckpointing(60000);  // Checkpoint every 60s
```

### Stateful Operator

```typescript
class RunningAverageOperator extends ProcessFunction<number, number> {
  private state: ValueState<{ sum: number; count: number }>;

  open(context: RuntimeContext) {
    const descriptor = new ValueStateDescriptor(
      'running-average',
      { sum: 0, count: 0 }
    );
    this.state = context.getState(descriptor);
  }

  async processElement(value: number, ctx: Context): Promise<void> {
    const current = await this.state.value();

    const updated = {
      sum: current.sum + value,
      count: current.count + 1
    };

    await this.state.update(updated);

    const average = updated.sum / updated.count;
    ctx.output(average);
  }
}
```

## Exactly-Once Processing

### Challenge: Avoid Duplicates

```
Problem:
1. Process event E1
2. Update state
3. Send result to Kafka
4. Crash before committing offset
5. Restart → Process E1 again (duplicate!)
```

### Solution: Two-Phase Commit

```java
// Kafka Streams with exactly-once
Properties props = new Properties();
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG,
          StreamsConfig.EXACTLY_ONCE_V2);

// Flink with exactly-once
env.getCheckpointConfig().setCheckpointingMode(
    CheckpointingMode.EXACTLY_ONCE
);

// How it works:
// 1. Process events
// 2. Write to state backend
// 3. Create checkpoint barrier
// 4. Write to Kafka (transactional)
// 5. Commit checkpoint + Kafka transaction atomically
```

## Performance Optimization

### Parallelism

```java
// Set parallelism per operator
events
    .filter(...)
    .setParallelism(10)  // 10 parallel instances
    .keyBy(...)
    .window(...)
    .setParallelism(20)  // 20 parallel instances
    .aggregate(...);
```

### Backpressure Handling

```
Problem: Downstream can't keep up with upstream

Solution: Backpressure
  ┌─────────┐  slow  ┌─────────┐  fast  ┌─────────┐
  │ Source  │───X───→│Operator │───────→│  Sink   │
  └─────────┘        └─────────┘        └─────────┘
                          ↑
                     Buffer full!

  Source slows down automatically
```

## API Design

```http
# Submit streaming job
POST /api/v1/jobs
Body: {
  "name": "clickstream-analytics",
  "source": "kafka://clicks",
  "processing": {
    "window": {
      "type": "tumbling",
      "size": "1m"
    },
    "aggregation": "count"
  },
  "sink": "dashboard"
}

# Get job status
GET /api/v1/jobs/job-123
Response: {
  "id": "job-123",
  "status": "RUNNING",
  "uptime": 3600,
  "recordsProcessed": 1000000,
  "throughput": 10000,  # records/sec
  "checkpoints": {
    "latest": 1708084800,
    "count": 60
  }
}

# Query processed data
GET /api/v1/analytics/clicks?window=1h
Response: {
  "timestamp": "2026-02-16T10:00:00Z",
  "metrics": {
    "totalClicks": 1000000,
    "uniqueUsers": 50000,
    "topPages": [
      { "url": "/home", "clicks": 100000 },
      { "url": "/products", "clicks": 80000 }
    ]
  }
}
```

## Summary

### What We Built:
- Real-time streaming analytics platform
- Windowed aggregations (tumbling, sliding, session)
- Stateful stream processing
- Exactly-once processing guarantees
- Fraud detection with pattern matching

### Key Concepts:
- ✅ Event time vs processing time
- ✅ Watermarks for late events
- ✅ Windowing strategies
- ✅ Stateful operators
- ✅ Exactly-once semantics
- ✅ Backpressure handling

### Real-World Examples:
- Netflix: Kafka Streams for real-time recommendations
- Uber: Flink for real-time pricing
- LinkedIn: Kafka for activity streams

---

**Next Project**: [15. Vector Database & RAG System](../15-vector-rag-system/README.md)
