# Project 13: APM & Observability Platform

## Difficulty: Advanced
**Estimated Time**: 4-5 weeks

## Problem Statement

Design an Application Performance Monitoring (APM) platform like Datadog, New Relic, or Dynatrace that collects metrics, logs, and traces from distributed systems, provides real-time dashboards, and enables alerting.

**Three Pillars of Observability**: Metrics, Logs, Traces

## Functional Requirements

1. **Metrics Collection**: Collect CPU, memory, latency, throughput
2. **Log Aggregation**: Centralize logs from all services
3. **Distributed Tracing**: Track requests across services
4. **Dashboards**: Visualize metrics in real-time
5. **Alerting**: Trigger alerts on anomalies
6. **Service Map**: Visualize service dependencies

## Architecture

```
┌────────────────────────────────────────────────┐
│           Instrumented Applications            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │Service A│  │Service B│  │Service C│       │
│  └────┬────┘  └────┬────┘  └────┬────┘       │
└───────┼────────────┼────────────┼─────────────┘
        │            │            │
        │ Metrics    │ Logs       │ Traces
        │            │            │
        ▼            ▼            ▼
┌─────────────────────────────────────────────────┐
│               Collection Layer                  │
│  ┌──────────┐  ┌───────┐  ┌──────────┐        │
│  │Prometheus│  │Fluentd│  │  Jaeger  │        │
│  └────┬─────┘  └───┬───┘  └────┬─────┘        │
└───────┼────────────┼────────────┼──────────────┘
        │            │            │
        ▼            ▼            ▼
┌─────────────────────────────────────────────────┐
│               Storage Layer                     │
│  ┌──────────┐  ┌────────────┐  ┌───────────┐  │
│  │Time-Series│  │Elasticsearch│  │Cassandra │  │
│  │   DB      │  │  (Logs)    │  │ (Traces) │  │
│  └──────────┘  └────────────┘  └───────────┘  │
└─────────────────────────────────────────────────┘
                      │
                      ▼
              ┌──────────────┐
              │  Dashboards  │
              │   (Grafana)  │
              └──────────────┘
```

## 1. Metrics Collection

### Agent-based Collection

```typescript
import { register, Counter, Histogram } from 'prom-client';

// Counter: Only increases
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status']
});

// Histogram: Distribution
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'path'],
  buckets: [0.1, 0.5, 1, 2, 5]  // Response time buckets
});

// Middleware
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;

    httpRequestsTotal.inc({
      method: req.method,
      path: req.route?.path || 'unknown',
      status: res.statusCode
    });

    httpRequestDuration.observe({
      method: req.method,
      path: req.route?.path || 'unknown'
    }, duration);
  });

  next();
});

// Expose metrics endpoint
app.get('/metrics', (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(register.metrics());
});
```

### Push vs Pull Model

```
Pull (Prometheus):
  Prometheus → Scrape /metrics endpoint → Store

  Pros: Service discovery, scrape failures visible
  Cons: Need to expose endpoint, firewall issues

Push (StatsD):
  App → Send metrics → Collector → Store

  Pros: Works with firewalls, short-lived jobs
  Cons: Collector can be overwhelmed
```

## 2. Log Aggregation

### Structured Logging

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'app.log' }),
    new winston.transports.Console()
  ]
});

// Usage
logger.info('User logged in', {
  userId: 123,
  email: 'user@example.com',
  ipAddress: req.ip,
  traceId: req.headers['x-trace-id']
});

// Output:
{
  "timestamp": "2026-02-16T10:30:00.000Z",
  "level": "info",
  "message": "User logged in",
  "userId": 123,
  "email": "user@example.com",
  "ipAddress": "1.2.3.4",
  "traceId": "abc-123-def"
}
```

### Log Pipeline

```
App → File → Log Shipper → Processing → Storage → Query
              (Filebeat)   (Logstash)   (Elasticsearch)
```

### Elasticsearch Schema

```json
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" },
      "service": { "type": "keyword" },
      "traceId": { "type": "keyword" },
      "userId": { "type": "keyword" },
      "metadata": { "type": "object" }
    }
  }
}
```

## 3. Distributed Tracing

### OpenTelemetry

```typescript
import { NodeTracerProvider } from '@opentelemetry/sdk-trace-node';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

// Setup tracer
const provider = new NodeTracerProvider();
provider.addSpanProcessor(
  new SimpleSpanProcessor(
    new JaegerExporter({ endpoint: 'http://jaeger:14268/api/traces' })
  )
);
provider.register();

const tracer = trace.getTracer('my-service');

// Create span
async function handleRequest(req, res) {
  const span = tracer.startSpan('handle_request', {
    attributes: {
      'http.method': req.method,
      'http.url': req.url,
      'http.user_agent': req.headers['user-agent']
    }
  });

  try {
    // Propagate context to child operations
    await context.with(trace.setSpan(context.active(), span), async () => {
      // Call other services
      const user = await fetchUser(req.userId);
      const orders = await fetchOrders(user.id);

      res.json({ user, orders });
    });

    span.setStatus({ code: SpanStatusCode.OK });
  } catch (error) {
    span.setStatus({
      code: SpanStatusCode.ERROR,
      message: error.message
    });
    span.recordException(error);
    throw error;
  } finally {
    span.end();
  }
}

// Child span
async function fetchUser(userId: string) {
  const span = tracer.startSpan('fetch_user', {
    attributes: { 'user.id': userId }
  });

  try {
    const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    return user;
  } finally {
    span.end();
  }
}
```

### Trace Visualization

```
Request ID: abc-123

Service A (10ms)
├─ Database Query (5ms)
└─ Call Service B (5ms)
   └─ Service B (15ms)
      ├─ Cache Lookup (1ms)
      └─ Call Service C (14ms)
         └─ Service C (50ms)
            └─ External API (48ms)

Total: 75ms
Bottleneck: Service C → External API (48ms)
```

## 4. Alerting

### Alert Rules

```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{status=~"5.."}[5m])
            / rate(http_requests_total[5m])
          ) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value }}s"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
```

### Alert Manager

```typescript
class AlertManager {
  private activeAlerts = new Map<string, Alert>();

  async evaluate(rule: AlertRule) {
    const result = await prometheus.query(rule.expr);

    for (const series of result) {
      const alertId = this.getAlertId(rule, series.labels);
      const value = series.value;

      if (value > rule.threshold) {
        const existing = this.activeAlerts.get(alertId);

        if (!existing) {
          // New alert
          this.activeAlerts.set(alertId, {
            id: alertId,
            rule: rule.name,
            state: 'PENDING',
            startsAt: Date.now(),
            value
          });
        } else if (existing.state === 'PENDING') {
          // Check if exceeded 'for' duration
          if (Date.now() - existing.startsAt >= rule.for * 1000) {
            existing.state = 'FIRING';
            await this.notify(existing);
          }
        }
      } else {
        // Alert resolved
        const existing = this.activeAlerts.get(alertId);
        if (existing && existing.state === 'FIRING') {
          existing.state = 'RESOLVED';
          await this.notify(existing);
          this.activeAlerts.delete(alertId);
        }
      }
    }
  }

  private async notify(alert: Alert) {
    // Send notifications
    await Promise.all([
      this.sendEmail(alert),
      this.sendSlack(alert),
      this.sendPagerDuty(alert)
    ]);
  }
}
```

## 5. Dashboards

### Grafana Dashboard JSON

```json
{
  "dashboard": {
    "title": "API Performance",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{path}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
          }
        ]
      },
      {
        "title": "Latency (P95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      }
    ]
  }
}
```

## 6. Service Map

```typescript
interface ServiceDependency {
  from: string;
  to: string;
  requestsPerSecond: number;
  errorRate: number;
  avgLatency: number;
}

class ServiceMapBuilder {
  async buildMap(): Promise<ServiceDependency[]> {
    // Query traces to find service dependencies
    const traces = await jaeger.getTraces({ limit: 10000 });

    const dependencies = new Map<string, ServiceDependency>();

    for (const trace of traces) {
      for (let i = 0; i < trace.spans.length - 1; i++) {
        const fromService = trace.spans[i].service;
        const toService = trace.spans[i + 1].service;

        const key = `${fromService}→${toService}`;

        if (!dependencies.has(key)) {
          dependencies.set(key, {
            from: fromService,
            to: toService,
            requestsPerSecond: 0,
            errorRate: 0,
            avgLatency: 0
          });
        }

        const dep = dependencies.get(key)!;
        dep.requestsPerSecond++;
        if (trace.spans[i + 1].error) dep.errorRate++;
        dep.avgLatency += trace.spans[i + 1].duration;
      }
    }

    // Calculate averages
    for (const dep of dependencies.values()) {
      dep.errorRate = dep.errorRate / dep.requestsPerSecond;
      dep.avgLatency = dep.avgLatency / dep.requestsPerSecond;
    }

    return Array.from(dependencies.values());
  }
}
```

## Summary

### What We Built:
- APM platform with metrics, logs, traces
- Real-time dashboards
- Alerting system
- Service dependency map

### Key Concepts:
- ✅ Three pillars of observability
- ✅ Metrics collection (Prometheus)
- ✅ Log aggregation (ELK stack)
- ✅ Distributed tracing (OpenTelemetry)
- ✅ Alert rules and notification
- ✅ Service map generation

### Real-World Examples:
- Datadog: 2T+ data points/day
- New Relic: APM for 1000s of apps
- Dynatrace: AI-powered monitoring

---

**Next Project**: [14. Streaming Analytics Platform](../14-streaming-analytics/README.md)
