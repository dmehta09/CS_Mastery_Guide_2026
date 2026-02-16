# Project 05: Time-Series Monitoring System

## Difficulty: Advanced
**Estimated Time**: 3-4 weeks

## Problem Statement

Design a time-series monitoring system like Prometheus, Datadog, or InfluxDB Cloud that collects metrics from distributed systems, stores them efficiently, and provides fast queries for dashboards and alerting.

**Real-World Example**: Monitor CPU usage across 1000 servers, storing data points every 15 seconds, and query "What was the average CPU usage for web servers in the last 7 days?"

## Functional Requirements

### Core Features:
1. **Metrics Collection**: Collect metrics from agents/applications
2. **Time-Series Storage**: Store metrics efficiently (compress, downsample)
3. **Querying**: Fast queries for dashboards (range queries, aggregations)
4. **Aggregation**: Downsample high-resolution data (15s → 1m → 5m → 1h → 1d)
5. **Retention**: Keep raw data for 7 days, aggregated for months/years
6. **Alerting**: Trigger alerts based on thresholds
7. **Dashboards**: Visualize metrics over time

### Out of Scope:
- Log aggregation (use ELK for that)
- Distributed tracing (use Jaeger)
- Anomaly detection (ML-based)

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Write Throughput** | 1M data points/sec | Across cluster |
| **Query Latency** | < 100ms | P95 for dashboard queries |
| **Retention** | 13 months | Raw: 7 days, Aggregated: 13 months |
| **Cardinality** | 100M unique series | Unique metric+labels combinations |
| **Compression** | 10:1 ratio | Compressed vs raw |
| **Availability** | 99.9% | ~8 hours downtime/year |

## Capacity Estimation

### Assumptions:
- **1000 servers** being monitored
- **100 metrics per server** (CPU, memory, disk, network, etc.)
- **15-second interval** (4 data points/minute)
- **8 bytes per data point** (timestamp + value)

### Storage Calculation:

```
Data points per second:
  1000 servers × 100 metrics × (1/15) = 6,666 points/sec
  = ~400K points/minute
  = ~24M points/hour
  = ~576M points/day

Raw storage (7 days retention):
  576M points/day × 8 bytes × 7 days = 32.3 GB

Downsampled storage (13 months):
  1-minute aggregates: 576M/4 × 8 bytes × 365 days = 1.68 TB
  5-minute aggregates: 1.68TB / 5 = 336 GB
  1-hour aggregates: 1.68TB / 60 = 28 GB

Total storage (with replication 3x):
  (32.3 GB + 1.68 TB) × 3 = ~5.2 TB

With compression (10:1):
  Required: ~520 GB
```

### Write Throughput:
```
Data points per second: 6,666
Data size: 6,666 × 8 bytes = 53 KB/sec = 4.3 Gbps

With metadata overhead (metric name, labels):
  Actual: ~200 KB/sec = 1.6 Gbps
```

### Query Patterns:
```
Dashboard refresh: Every 15 seconds
Queries per dashboard: 10 panels
Dashboards in use: 100 concurrent

QPS: 100 dashboards × 10 queries / 15 sec = 67 QPS

Alerting queries: 1000 rules × (1/minute) = 17 QPS

Total: ~100 QPS
```

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│              Monitored Applications                  │
│    ┌────────┐  ┌────────┐  ┌────────┐              │
│    │ App 1  │  │ App 2  │  │ App N  │              │
│    └───┬────┘  └───┬────┘  └───┬────┘              │
│        │  Push     │  Pull     │  Push             │
└────────┼───────────┼───────────┼────────────────────┘
         │           │           │
         ▼           ▼           ▼
    ┌────────────────────────────────┐
    │   Metrics Collection Layer     │
    │  ┌──────────┐  ┌──────────┐  │
    │  │Prometheus│  │Telegraf  │  │
    │  │  (Pull)  │  │  (Push)  │  │
    │  └────┬─────┘  └────┬─────┘  │
    └───────┼─────────────┼─────────┘
            │             │
            ▼             ▼
    ┌────────────────────────────────┐
    │    Time-Series Database        │
    │                                │
    │  ┌──────────┐  ┌──────────┐  │
    │  │  Shard 1 │  │  Shard 2 │  │
    │  │ (Hot)    │  │ (Hot)    │  │
    │  └──────────┘  └──────────┘  │
    │                                │
    │  ┌──────────────────────────┐ │
    │  │   Long-term Storage      │ │
    │  │   (Cold, Compressed)     │ │
    │  └──────────────────────────┘ │
    └────────┬───────────────────────┘
             │
             ├────────┬──────────────┬────────────┐
             │        │              │            │
             ▼        ▼              ▼            ▼
      ┌──────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐
      │Dashboard │ │Alerting │ │ Query   │ │Analytics │
      │(Grafana) │ │ Engine  │ │   API   │ │          │
      └──────────┘ └─────────┘ └─────────┘ └──────────┘
```

## Data Model

### Time-Series Metrics

```
Metric Structure:
metric_name{label1="value1", label2="value2"} value timestamp

Example:
http_requests_total{method="GET", path="/api/users", status="200"} 1523 1708084800
│                   │                                               │    │
└─ Metric Name      └─ Labels (key-value pairs)                    │    └─ Timestamp
                                                                    └─ Value
```

### Labels and Cardinality

```
Cardinality = Unique combinations of labels

Example:
- metric: http_requests_total
- labels: method (3 values), path (100 values), status (5 values)
- cardinality: 3 × 100 × 5 = 1,500 time series

High cardinality problem:
  metric{user_id="123"}  ← BAD! User ID as label
  10M users = 10M time series = memory explosion

  Solution: Use fixed labels
  metric{user_tier="free"}  ← GOOD! Limited values
  3 tiers = 3 time series
```

### Storage Schema

#### InfluxDB Schema

```influx
Database: monitoring

Measurement: cpu_usage
Tags (indexed):
  - host: web-server-1
  - region: us-east-1
  - environment: production

Fields (not indexed):
  - value: 75.5 (float)
  - num_cores: 8 (int)

Timestamp: 1708084800 (nanoseconds)

Query:
SELECT mean(value) FROM cpu_usage
WHERE host='web-server-1' AND time > now() - 1h
GROUP BY time(1m)
```

#### Prometheus Schema (TSDB)

```
Directory structure:
data/
├── 01HQXXX/          # Block (2 hour chunk)
│   ├── chunks/       # Compressed time series
│   │   └── 000001
│   ├── index         # Inverted index (label → series)
│   └── meta.json     # Block metadata
├── 01HQYYY/
└── wal/              # Write-ahead log
```

## Time-Series Compression

### Delta-of-Delta Encoding

```
Original timestamps:
1708084800, 1708084815, 1708084830, 1708084845

Deltas:
15, 15, 15 (constant interval)

Delta-of-deltas:
0, 0, 0 (all zeros!) → Highly compressible

Savings: 8 bytes × 4 = 32 bytes → 2 bytes
```

### Gorilla Compression (Facebook)

```
Values: 72.5, 72.7, 72.6, 72.8

XOR between consecutive values:
72.5 XOR 72.7 = 0.2 (only few bits different)

Store: First value + XOR diffs
Compression ratio: ~12:1 for typical metrics
```

### Implementation:

```typescript
class TimeSeriesBlock {
  private timestamps: number[] = [];
  private values: number[] = [];
  private baseTimestamp: number;
  private baseValue: number;
  private lastTimestamp: number;
  private lastValue: number;

  addPoint(timestamp: number, value: number) {
    if (this.timestamps.length === 0) {
      // First point: store as-is
      this.baseTimestamp = timestamp;
      this.baseValue = value;
      this.lastTimestamp = timestamp;
      this.lastValue = value;
      return;
    }

    // Delta-of-delta for timestamp
    const delta = timestamp - this.lastTimestamp;
    const lastDelta = this.lastTimestamp - this.baseTimestamp;
    const deltaOfDelta = delta - lastDelta;

    // XOR for value
    const xor = this.floatToInt(value) ^ this.floatToInt(this.lastValue);

    this.timestamps.push(deltaOfDelta);
    this.values.push(xor);

    this.lastTimestamp = timestamp;
    this.lastValue = value;
  }

  compress(): Buffer {
    // Bit-pack delta-of-deltas and XORs
    const bits: number[] = [];

    for (let i = 0; i < this.timestamps.length; i++) {
      const dod = this.timestamps[i];

      if (dod === 0) {
        // Common case: constant interval
        bits.push(0);  // 1 bit
      } else if (dod >= -63 && dod <= 64) {
        // Small delta: 2-bit header + 7-bit value
        bits.push(1, 0, ...this.toBits(dod, 7));
      } else {
        // Large delta: 2-bit header + 32-bit value
        bits.push(1, 1, ...this.toBits(dod, 32));
      }
    }

    return this.packBits(bits);
  }
}
```

## Downsampling (Aggregation)

### Retention Policy

```
Resolution      Retention       Storage
──────────────────────────────────────────
Raw (15s)       7 days          32 GB
1-minute        30 days         140 GB
5-minute        90 days         336 GB
1-hour          1 year          60 GB
1-day           Forever         2 GB

Total: ~570 GB (with 3x replication: 1.7 TB)
```

### Downsampling Process

```typescript
// Background job: Downsample every hour
async function downsampleMetrics() {
  const now = Date.now();
  const hourAgo = now - 3600000;

  // Get all 15s data points from last hour
  const rawData = await tsdb.query({
    metric: 'cpu_usage',
    start: hourAgo,
    end: now,
    resolution: '15s'
  });

  // Group by 1-minute windows
  const oneMinuteData = groupBy(rawData, 60000);

  // Calculate aggregates for each window
  for (const [timestamp, points] of oneMinuteData) {
    const aggregated = {
      timestamp,
      min: Math.min(...points.map(p => p.value)),
      max: Math.max(...points.map(p => p.value)),
      avg: average(points.map(p => p.value)),
      sum: sum(points.map(p => p.value)),
      count: points.length
    };

    // Write to 1-minute table
    await tsdb.writeDownsampled(aggregated, '1m');
  }

  // Delete raw data older than 7 days
  await tsdb.deleteOlderThan('15s', 7 * 86400000);
}
```

### Multi-Resolution Queries

```typescript
class TimeSeriesQuery {
  async query(metric: string, start: number, end: number): Promise<DataPoint[]> {
    const range = end - start;

    // Choose resolution based on time range
    let resolution: string;

    if (range <= 3600000) {
      // Last hour: use raw data (15s)
      resolution = '15s';
    } else if (range <= 86400000) {
      // Last day: use 1-minute data
      resolution = '1m';
    } else if (range <= 7 * 86400000) {
      // Last week: use 5-minute data
      resolution = '5m';
    } else if (range <= 30 * 86400000) {
      // Last month: use 1-hour data
      resolution = '1h';
    } else {
      // Longer: use 1-day data
      resolution = '1d';
    }

    return await this.queryResolution(metric, start, end, resolution);
  }
}
```

## Query Language

### PromQL (Prometheus Query Language)

```promql
# Instant query: Current value
http_requests_total{job="api-server"}

# Range query: Last 5 minutes
http_requests_total{job="api-server"}[5m]

# Rate: Requests per second
rate(http_requests_total[5m])

# Aggregation: Sum across all servers
sum(rate(http_requests_total[5m])) by (status)

# Arithmetic: Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
  * 100

# Functions:
- rate(): Calculate per-second rate
- increase(): Total increase over time range
- avg_over_time(): Average value over time
- max_over_time(): Maximum value
- histogram_quantile(): Calculate percentiles
```

### InfluxQL Examples

```influx
-- Average CPU over last hour
SELECT MEAN(value) FROM cpu_usage
WHERE time > now() - 1h
GROUP BY time(1m), host

-- Top 5 servers by CPU
SELECT TOP(value, 5) FROM cpu_usage
WHERE time > now() - 5m
GROUP BY host

-- Downsampling query
SELECT MEAN(value) as mean, MAX(value) as max
INTO cpu_usage_1h
FROM cpu_usage
GROUP BY time(1h), host
```

## Alerting

### Alert Rules

```yaml
groups:
  - name: api_alerts
    interval: 15s
    rules:
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            / sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighCPU
        expr: avg(cpu_usage) by (host) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.host }}"
          description: "CPU usage is {{ $value }}%"

      - alert: DiskSpaceLow
        expr: disk_free_gb < 10
        for: 1h
        labels:
          severity: warning
        annotations:
          description: "Only {{ $value }}GB free on {{ $labels.host }}"
```

### Alert State Machine

```
State transitions:
INACTIVE → PENDING → FIRING → RESOLVED
             ↓         ↓
          [for: 5m] [condition false]

Example:
1. Error rate > 5%: State = PENDING (start timer)
2. Still > 5% after 5 minutes: State = FIRING (send notification)
3. Error rate < 5%: State = RESOLVED (send recovery notification)
```

### Alert Manager

```typescript
class AlertManager {
  private activeAlerts = new Map<string, Alert>();

  async evaluate(rule: AlertRule) {
    const result = await this.queryEngine.execute(rule.expr);

    for (const series of result) {
      const alertId = this.getAlertId(rule, series.labels);
      const existingAlert = this.activeAlerts.get(alertId);

      if (series.value > rule.threshold) {
        if (!existingAlert) {
          // New alert: Create in PENDING state
          this.activeAlerts.set(alertId, {
            id: alertId,
            rule: rule.name,
            state: 'PENDING',
            value: series.value,
            labels: series.labels,
            startsAt: Date.now()
          });
        } else if (existingAlert.state === 'PENDING') {
          // Check if exceeded 'for' duration
          const duration = Date.now() - existingAlert.startsAt;

          if (duration >= rule.for * 1000) {
            // Promote to FIRING
            existingAlert.state = 'FIRING';
            await this.notify(existingAlert);
          }
        }
      } else {
        // Condition no longer met
        if (existingAlert && existingAlert.state === 'FIRING') {
          existingAlert.state = 'RESOLVED';
          await this.notify(existingAlert);
        }

        this.activeAlerts.delete(alertId);
      }
    }
  }

  private async notify(alert: Alert) {
    // Send to notification channels
    await Promise.all([
      this.sendEmail(alert),
      this.sendSlack(alert),
      this.sendPagerDuty(alert)
    ]);
  }
}
```

## Sharding Strategy

### Time-based Sharding

```
Shard assignment: hash(metric_name + labels) % num_shards

Shard 1: Metrics A-F
Shard 2: Metrics G-M
Shard 3: Metrics N-Z

Queries:
- Single metric: Hit 1 shard
- Wildcard (all metrics): Fan-out to all shards, merge results

Benefits:
- Horizontal scaling
- Parallel writes

Drawbacks:
- Hotspots if some metrics have high cardinality
```

### Consistent Hashing

```typescript
class ConsistentHashSharding {
  private ring = new Map<number, Shard>();
  private virtualNodes = 150;  // Per physical node

  addShard(shard: Shard) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${shard.id}-${i}`);
      this.ring.set(hash, shard);
    }

    // Sort ring by hash
    this.sortedKeys = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }

  getShard(metric: string): Shard {
    const hash = this.hash(metric);

    // Find next shard on ring
    const index = this.sortedKeys.findIndex(k => k >= hash);

    if (index === -1) {
      // Wrap around
      return this.ring.get(this.sortedKeys[0])!;
    }

    return this.ring.get(this.sortedKeys[index])!;
  }
}
```

## API Design

```http
# Write metrics (Push model)
POST /api/v1/write
Content-Type: application/x-protobuf
Body: <Prometheus remote write format>

# Query (PromQL)
GET /api/v1/query?query=up&time=1708084800
Response: {
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {"instance": "server1", "job": "api"},
        "value": [1708084800, "1"]
      }
    ]
  }
}

# Range query
GET /api/v1/query_range?query=rate(http_requests[5m])&start=1708084800&end=1708088400&step=15
Response: {
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {...},
        "values": [
          [1708084800, "123.45"],
          [1708084815, "125.32"],
          ...
        ]
      }
    ]
  }
}

# Get series metadata
GET /api/v1/series?match[]=http_requests_total{job="api"}
Response: {
  "data": [
    {"__name__": "http_requests_total", "job": "api", "instance": "server1"},
    {"__name__": "http_requests_total", "job": "api", "instance": "server2"}
  ]
}
```

## Trade-offs

### Pull vs Push

| Aspect | Pull (Prometheus) | Push (InfluxDB, Datadog) |
|--------|------------------|--------------------------|
| **Who initiates** | Server scrapes targets | Targets push to server |
| **Discovery** | Need service discovery | Targets register themselves |
| **Network** | Server must reach targets | Targets must reach server |
| **Debugging** | Can check /metrics endpoint | Harder to debug |
| **Short-lived** | Bad (jobs finish before scrape) | Good (push before exit) |
| **Security** | Firewall-friendly (server initiates) | Need to allow inbound |

### Storage: Row vs Column

| Aspect | Row-oriented | Column-oriented |
|--------|-------------|----------------|
| **Write** | Fast (append row) | Slower (update columns) |
| **Read** | Slow (scan all columns) | Fast (scan relevant columns) |
| **Compression** | Lower | Higher (similar values) |
| **Use case** | OLTP | OLAP (analytics) |

Time-series is read-heavy analytics → **Column-oriented better**

## Monitoring the Monitor

```yaml
Metrics to track:
  - ingestion_rate: Data points written/sec
  - query_latency_p95: Query latency
  - storage_bytes: Disk usage
  - cardinality: Number of unique time series
  - compaction_duration: Time to compress blocks

Alerts:
  - name: TSDBDown
    expr: up{job="tsdb"} == 0
    severity: critical

  - name: HighCardinality
    expr: tsdb_series_count > 10000000
    severity: warning
    description: "High cardinality can cause OOM"
```

## Summary

### What We Built:
- Time-series monitoring system
- Efficient compression (delta-of-delta, XOR)
- Multi-resolution storage (raw + downsampled)
- Fast queries with indexing
- Alerting engine

### Key Concepts:
- ✅ Time-series data model
- ✅ Compression algorithms
- ✅ Downsampling and retention
- ✅ PromQL query language
- ✅ Sharding strategies
- ✅ Pull vs Push models

### Real-World Examples:
- Prometheus: 1M samples/sec per instance
- InfluxDB: 10M writes/sec (clustered)
- Datadog: 2T+ metrics/day

---

**Next Project**: [06. Distributed Search Engine](../06-search-engine/README.md)
