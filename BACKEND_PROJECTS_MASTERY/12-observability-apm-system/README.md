# Project 12: Observability & APM System

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language

## What You're Building

An Application Performance Monitoring (APM) system like Datadog, New Relic, or Grafana Cloud that collects metrics, logs, and traces from distributed systems.

## Key Concepts (Three Pillars)

### 1. Metrics
- Counter, Gauge, Histogram
- Prometheus format
- Custom metrics
- Dashboards (Grafana)

### 2. Logs
- Structured logging (JSON)
- Log aggregation
- Correlation IDs
- Log levels

### 3. Traces
- Distributed tracing
- OpenTelemetry
- Span context propagation
- Service maps

## Architecture

```
Applications → Agents → Collectors → Storage → Visualization
                 ↓          ↓            ↓           ↓
             (Metrics)  (Aggregation) (Time-Series) (Dashboards)
             (Logs)     (Filtering)   (Elasticsearch)
             (Traces)   (Sampling)    (Jaeger)
```

## Implementation

### Metrics Collection

```typescript
import { Registry, Counter, Histogram } from "prom-client";

const registry = new Registry();

// Counter: Only goes up
const httpRequestsTotal = new Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "path", "status"],
  registers: [registry]
});

// Histogram: Distribution of values
const httpRequestDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration",
  labelNames: ["method", "path"],
  buckets: [0.1, 0.5, 1, 2, 5],
  registers: [registry]
});

// Usage
app.use((req, res, next) => {
  const start = Date.now();

  res.on("finish", () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestsTotal.inc({
      method: req.method,
      path: req.route?.path,
      status: res.statusCode
    });
    httpRequestDuration.observe(
      { method: req.method, path: req.route?.path },
      duration
    );
  });

  next();
});
```

### Structured Logging

```typescript
import winston from "winston";

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({
      filename: "application.log"
    })
  ]
});

// Usage
logger.info("User logged in", {
  user_id: 123,
  email: "user@example.com",
  ip_address: req.ip,
  trace_id: req.headers["x-trace-id"]
});

// Output:
// {
//   "timestamp": "2026-02-16T10:30:00.000Z",
//   "level": "info",
//   "message": "User logged in",
//   "user_id": 123,
//   "email": "user@example.com",
//   "ip_address": "1.2.3.4",
//   "trace_id": "abc123"
// }
```

### Distributed Tracing

```typescript
import { trace, context } from "@opentelemetry/api";
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";

const provider = new NodeTracerProvider();
provider.addSpanProcessor(
  new SimpleSpanProcessor(
    new JaegerExporter({ endpoint: "http://jaeger:14268/api/traces" })
  )
);
provider.register();

const tracer = trace.getTracer("my-service");

// Create spans
async function handleRequest(req, res) {
  const span = tracer.startSpan("handle_request", {
    attributes: {
      "http.method": req.method,
      "http.url": req.url
    }
  });

  try {
    // Do work
    await processRequest(req);
    span.setStatus({ code: SpanStatusCode.OK });
  } catch (error) {
    span.setStatus({
      code: SpanStatusCode.ERROR,
      message: error.message
    });
    throw error;
  } finally {
    span.end();
  }
}
```

## Alerting Rules

```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} (threshold: 0.01)"

      - alert: HighLatency
        expr: histogram_quantile(0.95, http_request_duration_seconds) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value }}s"
```

## Dashboard (Grafana)

```json
{
  "dashboard": {
    "title": "API Performance",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
          }
        ]
      },
      {
        "title": "Latency (P95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, http_request_duration_seconds)"
          }
        ]
      }
    ]
  }
}
```

## Success Criteria

✅ Collect metrics from all services
✅ Centralized log aggregation
✅ End-to-end distributed tracing
✅ Real-time dashboards
✅ Alert on anomalies
✅ Sub-100ms trace overhead

---

**Next**: Project 13 (Multi-Region Auth)
