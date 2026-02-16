# Project 05: Stream Processing Analytics Platform

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language

---

## What You're Building

A real-time analytics system that processes millions of events per day (similar to Apache Kafka + Flink/Spark Streaming). Think of how LinkedIn processes activity streams or how Uber tracks real-time trip analytics.

## Layman Explanation

Imagine a **highway traffic monitoring system**:

1. Thousands of cars (events) pass sensors every second
2. System counts cars in real-time
3. Calculates average speed per hour
4. Detects traffic jams immediately
5. Shows live dashboards to traffic controllers
6. Stores historical data for analysis

But instead of cars, we're processing user clicks, purchases, sensor data, or IoT events.

---

## Key Concepts

### 1. Stream Processing
- Kafka producers and consumers
- Consumer groups for parallel processing
- Offset management
- Exactly-once semantics
- Event time vs processing time

### 2. Windowing
- Tumbling windows (fixed intervals)
- Sliding windows (overlapping)
- Session windows (activity-based)
- Late data handling
- Watermarks

### 3. Aggregations
- Count, sum, average in real-time
- Group by dimensions
- Time-based aggregations
- Complex event processing

### 4. Time-Series Storage
- InfluxDB or TimescaleDB
- Data retention policies
- Downsampling strategies
- Query optimization

---

## Architecture

```
[Event Sources] → [Kafka Topics] → [Stream Processors] → [Time-Series DB] → [Dashboard]
                       ↓
                  [Event Store]
                  (Archive)
```

---

## Use Cases

### 1. Website Analytics
```
Events: page_view, click, scroll, form_submit
Metrics:
- Page views per minute
- Unique visitors per hour
- Most popular pages
- Conversion funnel
```

### 2. E-Commerce
```
Events: product_view, add_to_cart, purchase, refund
Metrics:
- Revenue per minute
- Cart abandonment rate
- Popular products by region
- Real-time inventory tracking
```

### 3. IoT Sensor Data
```
Events: temperature, humidity, pressure readings
Metrics:
- Average temperature per hour
- Anomaly detection
- Device health monitoring
- Predictive maintenance
```

---

## Event Schema (Kafka)

```json
{
  "event_id": "evt_123456",
  "event_type": "purchase_completed",
  "timestamp": "2026-02-16T10:30:00.000Z",
  "user_id": "user_123",
  "session_id": "sess_abc",
  "properties": {
    "product_id": "prod_456",
    "amount": 99.99,
    "currency": "USD",
    "payment_method": "credit_card",
    "country": "US"
  },
  "metadata": {
    "ip_address": "1.2.3.4",
    "user_agent": "Mozilla/5.0...",
    "platform": "web"
  }
}
```

---

## Stream Processing Example

### Tumbling Window (1-minute aggregations)

```typescript
// Calculate revenue per minute
stream
  .filter((event) => event.event_type === "purchase_completed")
  .window(tumblingWindow(Duration.minutes(1)))
  .groupBy((event) => event.properties.currency)
  .aggregate({
    count: count(),
    total_revenue: sum((e) => e.properties.amount),
    avg_order_value: avg((e) => e.properties.amount)
  })
  .to("analytics.revenue_per_minute");
```

### Sliding Window (5-minute window, 1-minute slide)

```typescript
// Top products in last 5 minutes, updated every minute
stream
  .filter((event) => event.event_type === "product_view")
  .window(slidingWindow(
    Duration.minutes(5),
    Duration.minutes(1)
  ))
  .groupBy((event) => event.properties.product_id)
  .aggregate({ view_count: count() })
  .orderBy("view_count", "DESC")
  .limit(10)
  .to("analytics.top_products");
```

---

## Database Schema

### Time-Series DB (InfluxDB)

```
Measurement: website_metrics
Tags: page_url, country, device_type
Fields: page_views (int), unique_users (int), avg_load_time (float)
Time: timestamp

Measurement: purchase_metrics
Tags: product_id, currency, payment_method
Fields: count (int), total_amount (float), avg_amount (float)
Time: timestamp
```

### PostgreSQL (Metadata)

```sql
CREATE TABLE event_schemas (
    event_type VARCHAR(100) PRIMARY KEY,
    schema_version INT NOT NULL,
    json_schema JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE dashboards (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    config JSONB NOT NULL, -- Chart configurations
    created_by BIGINT NOT NULL,
    is_public BOOLEAN DEFAULT FALSE
);
```

---

## Implementation Steps

### Week 1: Event Ingestion
1. Set up Kafka cluster
2. Create producers (event ingestion API)
3. Define event schemas
4. Implement validation
5. Basic consumers

### Week 2: Stream Processing
1. Implement windowing logic
2. Real-time aggregations
3. Handle late events
4. Exactly-once processing
5. State management

### Week 3: Storage & Queries
1. Set up time-series database
2. Write processed data
3. Implement query API
4. Retention policies
5. Data archival

### Week 4: Dashboards & Advanced
1. Real-time dashboard API
2. WebSocket for live updates
3. Complex event processing
4. Anomaly detection
5. Performance tuning

---

## Kafka Configuration

```yaml
# Topics
topics:
  - name: raw_events
    partitions: 12
    replication_factor: 3
    retention_ms: 604800000 # 7 days

  - name: processed_metrics
    partitions: 6
    replication_factor: 3
    retention_ms: 2592000000 # 30 days

# Consumer Groups
consumer_groups:
  - name: analytics_processor
    processors: 12
    auto_commit: false # Manual offset management
    isolation_level: read_committed
```

---

## API Endpoints

```http
# Ingest Event
POST /api/v1/events
{
  "event_type": "purchase_completed",
  "user_id": "user_123",
  "properties": { ... }
}

# Query Metrics
GET /api/v1/metrics?
  metric=revenue&
  group_by=country&
  start_time=2026-02-16T00:00:00Z&
  end_time=2026-02-16T23:59:59Z&
  window=1h

# Real-Time Metrics (WebSocket)
WS /api/v1/stream/metrics
{
  "subscribe": ["revenue_per_minute", "active_users"]
}
```

---

## Technology Stack

### NestJS
- kafkajs
- @influxdata/influxdb-client
- bull (job queues)

### Golang
- confluent-kafka-go
- influxdb-client-go
- High performance

### Python
- kafka-python / confluent-kafka-python
- influxdb-client-py
- pandas (data processing)

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Event Ingestion | 100,000 events/sec |
| Processing Latency | < 1 second |
| Query Response | < 500ms |
| Storage Retention | 90 days hot, 1 year cold |

---

## Success Criteria

✅ Process 100K+ events/sec
✅ Sub-second processing latency
✅ Exactly-once semantics
✅ Handle late events correctly
✅ Real-time dashboards
✅ Scalable horizontally
✅ Data consistency

---

**Next**: Move to Project 06 (Distributed File Storage) for large-scale object storage!
