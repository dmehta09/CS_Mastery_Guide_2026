# Observability Mental Models 2026

## Mental Model 1: The Three Pillars Hospital Analogy

Think of observability like a hospital monitoring a patient:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE THREE PILLARS OF OBSERVABILITY                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │     METRICS     │  │      LOGS       │  │     TRACES      │             │
│  │   (Vital Signs) │  │(Medical Records)│  │  (MRI/CT Scan)  │             │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤             │
│  │                 │  │                 │  │                 │             │
│  │  Heart Rate: 72 │  │ 10:30 - Patient │  │  Blood Flow     │             │
│  │  BP: 120/80     │  │   complained of │  │  from Heart     │             │
│  │  Temp: 98.6°F   │  │   chest pain    │  │    ↓            │             │
│  │  O2 Sat: 98%    │  │ 10:35 - Gave    │  │  Through Artery │             │
│  │                 │  │   medication    │  │    ↓            │             │
│  │ "How is the     │  │                 │  │  To Organs      │             │
│  │  patient NOW?"  │  │ "What HAPPENED?"│  │    ↓            │             │
│  │                 │  │                 │  │  Back to Heart  │             │
│  │  AGGREGATED     │  │  DISCRETE       │  │                 │             │
│  │  NUMERIC DATA   │  │  EVENTS         │  │ "HOW does it    │             │
│  │                 │  │                 │  │  flow through?" │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  USE CASE:          USE CASE:            USE CASE:                         │
│  - Dashboards       - Debugging          - Request flow                    │
│  - Alerting         - Auditing           - Latency analysis                │
│  - Capacity plan    - Forensics          - Dependencies                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### When to Use Each Pillar

```
┌─────────────────────────────────────────────────────────────────┐
│                  QUESTION → PILLAR MAPPING                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Is the system healthy?"              → METRICS               │
│  "What's the error rate?"              → METRICS               │
│  "Are we meeting SLOs?"                → METRICS               │
│                                                                 │
│  "Why did this specific request fail?" → LOGS                  │
│  "What was the error message?"         → LOGS                  │
│  "Who did what and when?"              → LOGS                  │
│                                                                 │
│  "Where is the latency coming from?"   → TRACES                │
│  "Which service is the bottleneck?"    → TRACES                │
│  "What's the request path?"            → TRACES                │
│                                                                 │
│  "Why is the system slow?"             → ALL THREE!            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 2: Metrics Methodologies (USE, RED, Golden Signals)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CHOOSING YOUR METRICS METHODOLOGY                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  USE METHOD (For Resources/Infrastructure)                          │   │
│  │  "How healthy is this RESOURCE?"                                    │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                     │   │
│  │  U - Utilization: % of resource being used                         │   │
│  │      └─ "CPU at 80%", "Memory at 60%", "Disk at 90%"               │   │
│  │                                                                     │   │
│  │  S - Saturation: Work waiting in queue                             │   │
│  │      └─ "5 threads waiting", "Queue depth: 100"                    │   │
│  │                                                                     │   │
│  │  E - Errors: Error count/rate                                      │   │
│  │      └─ "10 disk errors", "5 network timeouts"                     │   │
│  │                                                                     │   │
│  │  APPLY TO: CPU, Memory, Disk, Network, DB Connections              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  RED METHOD (For Services/Microservices)                            │   │
│  │  "How healthy is this SERVICE?"                                     │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                     │   │
│  │  R - Rate: Requests per second                                     │   │
│  │      └─ "500 req/s", "Traffic volume"                              │   │
│  │                                                                     │   │
│  │  E - Errors: Failed requests per second                            │   │
│  │      └─ "5 errors/s", "1% error rate"                              │   │
│  │                                                                     │   │
│  │  D - Duration: Latency (usually percentiles)                       │   │
│  │      └─ "p50: 50ms, p99: 200ms"                                    │   │
│  │                                                                     │   │
│  │  APPLY TO: API endpoints, Microservices, Request handlers          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  GOLDEN SIGNALS (Google SRE - Comprehensive)                        │   │
│  │  "How healthy is the system from USER perspective?"                 │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                     │   │
│  │  Latency: Time to serve a request (success vs error)               │   │
│  │  Traffic: Demand on your system (req/s, concurrent users)          │   │
│  │  Errors: Rate of failed requests (explicit + implicit)             │   │
│  │  Saturation: How "full" the service is (capacity headroom)         │   │
│  │                                                                     │   │
│  │  APPLY TO: User-facing services, SLO-driven systems                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Methodology Selection Decision Tree

```
What are you monitoring?
          │
          ├── Infrastructure (CPU, Memory, Disk, Network)?
          │         │
          │         └── USE: USE METHOD
          │
          ├── Microservice/API?
          │         │
          │         ├── Internal service → USE: RED METHOD
          │         │
          │         └── User-facing with SLOs → USE: GOLDEN SIGNALS
          │
          └── Entire system health?
                    │
                    └── USE: GOLDEN SIGNALS + USE for resources
```

---

## Mental Model 3: Prometheus Architecture & PromQL

### Prometheus Pull Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     PROMETHEUS PULL-BASED ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐                                                          │
│  │  Your App    │──┐                                                       │
│  │  /metrics    │  │                                                       │
│  └──────────────┘  │                                                       │
│                    │     ┌─────────────────────────────────┐               │
│  ┌──────────────┐  │     │        PROMETHEUS SERVER        │               │
│  │  Database    │  │     │  ┌─────────┐    ┌──────────┐   │               │
│  │  Exporter    │──┼────▶│  │ Scraper │───▶│   TSDB   │   │               │
│  │  /metrics    │  │     │  │(Pull)   │    │(Storage) │   │               │
│  └──────────────┘  │     │  └─────────┘    └──────────┘   │               │
│                    │     │       │              │         │               │
│  ┌──────────────┐  │     │       ▼              ▼         │               │
│  │  Node        │  │     │  ┌─────────┐    ┌──────────┐   │    ┌────────┐ │
│  │  Exporter    │──┘     │  │ Service │    │  Query   │───┼───▶│Grafana │ │
│  │  /metrics    │        │  │Discovery│    │ Engine   │   │    └────────┘ │
│  └──────────────┘        │  └─────────┘    │(PromQL)  │   │               │
│                          │                  └──────────┘   │               │
│                          │        │                        │               │
│                          │        ▼                        │               │
│                          │  ┌──────────────┐              │               │
│                          │  │ Alertmanager │              │               │
│                          │  │   (Rules)    │              │               │
│                          │  └──────────────┘              │               │
│                          └─────────────────────────────────┘               │
│                                                                             │
│  WHY PULL?                                                                 │
│  ✓ Prometheus controls scrape rate                                         │
│  ✓ Easy to detect if target is down (scrape fails)                        │
│  ✓ Can manually scrape for debugging                                       │
│  ✓ Simpler security (no inbound connections to apps)                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### PromQL Mental Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PromQL QUERY STRUCTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BASIC PATTERN:                                                            │
│                                                                             │
│    metric_name{label="value"}[time_range] @ timestamp offset duration      │
│    ─────┬─────  ──────┬──────  ────┬────   ────┬────  ─────┬─────          │
│         │             │            │           │           │               │
│         │             │            │           │           └─ Go back in   │
│         │             │            │           │              time         │
│         │             │            │           └─ Fixed timestamp          │
│         │             │            └─ Range vector (for rate())            │
│         │             └─ Filter by labels                                  │
│         └─ Metric name (what you're measuring)                             │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  COMMON PATTERNS:                                                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ RATE OF CHANGE (for counters)                                       │   │
│  │                                                                     │   │
│  │   rate(http_requests_total[5m])                                    │   │
│  │        ─────┬─────────────  ──┬                                    │   │
│  │             │                 └─ Look back 5 minutes               │   │
│  │             └─ Counter (always increases)                          │   │
│  │                                                                     │   │
│  │   Result: Requests per second (averaged over 5 min)                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PERCENTILES (for histograms)                                        │   │
│  │                                                                     │   │
│  │   histogram_quantile(0.99, rate(request_duration_bucket[5m]))      │   │
│  │                      ────  ──────────────┬────────────────         │   │
│  │                       │                  └─ Must use rate() first  │   │
│  │                       └─ 99th percentile                           │   │
│  │                                                                     │   │
│  │   Result: p99 latency over last 5 minutes                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AGGREGATION (group by)                                              │   │
│  │                                                                     │   │
│  │   sum by (status_code) (rate(http_requests_total[5m]))             │   │
│  │   ───┬── ───────┬─────                                             │   │
│  │      │          └─ Group by this label                             │   │
│  │      └─ Aggregation function                                       │   │
│  │                                                                     │   │
│  │   Result: Request rate broken down by status code                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ERROR RATE CALCULATION                                              │   │
│  │                                                                     │   │
│  │   sum(rate(http_requests_total{status=~"5.."}[5m]))                │   │
│  │   ─────────────────────────────────────────────────                │   │
│  │   sum(rate(http_requests_total[5m]))                               │   │
│  │                                                                     │   │
│  │   Result: Percentage of 5xx errors                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Metric Types Decision Tree

```
What kind of value are you measuring?
              │
              ├── A value that only goes UP (cumulative)?
              │         │
              │         └── USE: COUNTER
              │             Examples: total_requests, bytes_sent, errors_total
              │             Query with: rate(), increase()
              │
              ├── A value that can go UP or DOWN?
              │         │
              │         └── USE: GAUGE
              │             Examples: temperature, queue_size, active_connections
              │             Query directly or with avg_over_time()
              │
              ├── Distribution of values (latency, sizes)?
              │         │
              │         ├── Need percentiles (p50, p99)?
              │         │         │
              │         │         └── USE: HISTOGRAM
              │         │             Server-side aggregation, approximate percentiles
              │         │
              │         └── Need exact percentiles (pre-calculated)?
              │                   │
              │                   └── USE: SUMMARY
              │                       Client-side calculation, cannot aggregate
              │
              └── Unsure?
                        │
                        └── USE: HISTOGRAM (most flexible)
```

---

## Mental Model 4: OpenTelemetry Collector Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPENTELEMETRY COLLECTOR PIPELINE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Think of it like a WATER TREATMENT PLANT:                                 │
│                                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐    │
│  │  RECEIVERS  │──▶│ PROCESSORS  │──▶│ PROCESSORS  │──▶│  EXPORTERS  │    │
│  │   (Intake)  │   │  (Filter)   │   │  (Enrich)   │   │  (Output)   │    │
│  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘    │
│                                                                             │
│    Water comes     Remove debris    Add minerals     Send to homes         │
│    from sources    & contaminants   & treatment      via pipes             │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ACTUAL PIPELINE EXAMPLE:                                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                           RECEIVERS                                  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐               │   │
│  │  │   OTLP   │ │Prometheus│ │  Jaeger  │ │ Filelog  │               │   │
│  │  │ (gRPC/   │ │(Scrape   │ │(Legacy   │ │(Read log │               │   │
│  │  │  HTTP)   │ │ targets) │ │ traces)  │ │  files)  │               │   │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘               │   │
│  └───────┼────────────┼────────────┼────────────┼───────────────────────┘   │
│          │            │            │            │                          │
│          └────────────┴────────────┴────────────┘                          │
│                            │                                               │
│                            ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          PROCESSORS                                  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐               │   │
│  │  │  Batch   │▶│ Memory   │▶│Attributes│▶│ Filter   │               │   │
│  │  │(Group    │ │ Limiter  │ │(Add/     │ │(Drop     │               │   │
│  │  │ data)    │ │(Prevent  │ │ Remove)  │ │ unwanted)│               │   │
│  │  │          │ │  OOM)    │ │          │ │          │               │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                               │
│                            ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          EXPORTERS                                   │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐               │   │
│  │  │   OTLP   │ │Prometheus│ │   Loki   │ │   AWS    │               │   │
│  │  │ (To      │ │(Remote   │ │(Logs to  │ │X-Ray     │               │   │
│  │  │  Tempo)  │ │  write)  │ │  Loki)   │ │          │               │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  YAML CONFIG STRUCTURE:                                                    │
│                                                                             │
│  receivers:              # Define data sources                             │
│    otlp:                                                                   │
│      protocols:                                                            │
│        grpc:                                                               │
│        http:                                                               │
│                                                                             │
│  processors:             # Define transformations                          │
│    batch:                                                                  │
│    memory_limiter:                                                         │
│      limit_mib: 512                                                        │
│                                                                             │
│  exporters:              # Define destinations                             │
│    otlp:                                                                   │
│      endpoint: tempo:4317                                                  │
│                                                                             │
│  service:                # Wire it all together                            │
│    pipelines:                                                              │
│      traces:                                                               │
│        receivers: [otlp]                                                   │
│        processors: [batch, memory_limiter]                                 │
│        exporters: [otlp]                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 5: Distributed Tracing Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DISTRIBUTED TRACE ANATOMY                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  A TRACE = The entire journey of ONE request through your system           │
│  A SPAN  = One operation/step in that journey                              │
│                                                                             │
│  TRACE ID: abc123                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  [API Gateway]─────────────────────────────────────────────────────│   │
│  │  Span ID: 001   Parent: none   Duration: 250ms                     │   │
│  │       │                                                             │   │
│  │       ├──[Auth Service]────────────────────────────────            │   │
│  │       │  Span ID: 002   Parent: 001   Duration: 50ms               │   │
│  │       │       │                                                     │   │
│  │       │       └──[Redis Cache]─────────                            │   │
│  │       │          Span ID: 003   Parent: 002   Duration: 5ms        │   │
│  │       │                                                             │   │
│  │       └──[Order Service]─────────────────────────────────────      │   │
│  │          Span ID: 004   Parent: 001   Duration: 180ms              │   │
│  │               │                                                     │   │
│  │               ├──[Inventory Service]──────────────────             │   │
│  │               │  Span ID: 005   Parent: 004   Duration: 80ms       │   │
│  │               │       │                                             │   │
│  │               │       └──[PostgreSQL]──────                        │   │
│  │               │          Span ID: 006   Parent: 005   Duration: 30ms│   │
│  │               │                                                     │   │
│  │               └──[Payment Service]────────────                     │   │
│  │                  Span ID: 007   Parent: 004   Duration: 100ms      │   │
│  │                       │                                             │   │
│  │                       └──[Stripe API]─────                         │   │
│  │                          Span ID: 008   Parent: 007   Duration: 90ms│   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  TIMELINE VIEW (Gantt-style):                                              │
│  ─────────────────────────────────────────────────────────────────────     │
│  0ms        50ms       100ms      150ms      200ms      250ms              │
│  │          │          │          │          │          │                  │
│  [===================API Gateway===================]                       │
│  [==Auth==]                                                                │
│    [Redis]                                                                 │
│       [===============Order Service================]                       │
│          [=====Inventory=====]                                             │
│             [==DB==]                                                       │
│                  [========Payment========]                                 │
│                     [======Stripe======]                                   │
│                                                                             │
│  CRITICAL PATH: API Gateway → Order Service → Payment → Stripe (bottleneck)│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Span Attributes & Context Propagation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CONTEXT PROPAGATION FLOW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Service A                           Service B                             │
│  ┌──────────────────┐               ┌──────────────────┐                   │
│  │                  │               │                  │                   │
│  │  Create Span     │   HTTP/gRPC   │  Extract Context │                   │
│  │       │          │   ─────────▶  │       │          │                   │
│  │       ▼          │    Headers    │       ▼          │                   │
│  │  Inject Context  │               │  Create Child    │                   │
│  │  into Headers    │               │  Span            │                   │
│  │                  │               │                  │                   │
│  └──────────────────┘               └──────────────────┘                   │
│                                                                             │
│  HEADERS EXAMPLE (W3C Trace Context):                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01│   │
│  │               ──  ────────────────┬───────────────  ───────┬──────  ──│   │
│  │               │                   │                        │         │   │
│  │               │                   │                        │         │   │
│  │           version             trace-id                 span-id    flags │
│  │           (00)            (32 hex chars)           (16 hex chars)       │
│  │                                                                     │   │
│  │  tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE                │   │
│  │              (vendor-specific key-value pairs)                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 6: SLOs, SLIs, and Error Budgets

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SLO HIERARCHY: FROM SLA TO SLI                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SLA (Service Level AGREEMENT)                                      │   │
│  │  "Legal contract with consequences"                                 │   │
│  │                                                                     │   │
│  │  Example: "99.9% uptime or customer gets credit"                   │   │
│  │           "Response within 4 hours or refund"                      │   │
│  │                                                                     │   │
│  │  WHO CARES: Legal, Sales, Customers                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                               │
│                            ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SLO (Service Level OBJECTIVE)                                      │   │
│  │  "Internal target, stricter than SLA"                               │   │
│  │                                                                     │   │
│  │  Example: "99.95% availability (leaves buffer for SLA)"            │   │
│  │           "p99 latency < 200ms"                                    │   │
│  │                                                                     │   │
│  │  WHO CARES: Engineering, Product, SRE                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                               │
│                            ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SLI (Service Level INDICATOR)                                      │   │
│  │  "The actual measurement/metric"                                    │   │
│  │                                                                     │   │
│  │  Example: successful_requests / total_requests                     │   │
│  │           histogram_quantile(0.99, request_duration)               │   │
│  │                                                                     │   │
│  │  WHO CARES: SRE, DevOps, Monitoring systems                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Error Budget Mental Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ERROR BUDGET VISUALIZATION                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SLO: 99.9% availability over 30 days                                      │
│                                                                             │
│  30 days = 43,200 minutes                                                  │
│  Error Budget = 0.1% = 43.2 minutes of allowed downtime                    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Day 1        Day 10       Day 20       Day 30                      │   │
│  │  │            │            │            │                           │   │
│  │  ████████████████████████████████████████  100% budget remaining   │   │
│  │                                                                     │   │
│  │  Day 5: 15-minute outage                                           │   │
│  │  ████████████████████████████░░░░░░░░░░░   65% budget remaining    │   │
│  │                                                                     │   │
│  │  Day 12: 10-minute degradation                                     │   │
│  │  ████████████████████░░░░░░░░░░░░░░░░░░░   42% budget remaining    │   │
│  │                                                                     │   │
│  │  Day 20: 20-minute incident                                        │   │
│  │  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   -4% OVER BUDGET!        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ERROR BUDGET POLICY:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  Budget > 50%  → Ship features freely, experiment                  │   │
│  │  Budget 20-50% → Ship with caution, limit risky changes            │   │
│  │  Budget < 20%  → Focus on reliability, freeze features             │   │
│  │  Budget = 0%   → ALL HANDS ON RELIABILITY, no new features         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Multi-Window Multi-Burn-Rate Alerts

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  MULTI-WINDOW BURN RATE ALERTING                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BURN RATE = How fast you're consuming error budget                        │
│                                                                             │
│  Burn Rate 1x  = Using budget exactly as planned (will exhaust at month end)│
│  Burn Rate 10x = Using 10x faster (will exhaust in 3 days!)               │
│  Burn Rate 36x = Using 36x faster (will exhaust in 20 hours!)             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ALERTING WINDOWS (Google SRE recommended)                          │   │
│  │                                                                     │   │
│  │  Severity   │ Burn Rate │ Long Window │ Short Window │ Action      │   │
│  │  ───────────┼───────────┼─────────────┼──────────────┼─────────────│   │
│  │  PAGE       │    14.4   │   1 hour    │   5 min      │ Wake up!    │   │
│  │  PAGE       │    6      │   6 hours   │   30 min     │ Urgent      │   │
│  │  TICKET     │    3      │   1 day     │   2 hours    │ Fix today   │   │
│  │  TICKET     │    1      │   3 days    │   6 hours    │ Investigate │   │
│  │                                                                     │   │
│  │  WHY TWO WINDOWS?                                                   │   │
│  │  - Long window: Catches sustained problems                         │   │
│  │  - Short window: Prevents alerting on already-recovered issues     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PROMQL EXAMPLE:                                                           │
│                                                                             │
│  # Alert if 14.4x burn rate over 1 hour AND 5 minutes                     │
│  (                                                                         │
│    sum(rate(http_requests_total{status=~"5.."}[1h]))                      │
│    / sum(rate(http_requests_total[1h]))                                   │
│  ) > (14.4 * 0.001)  # 14.4x the 0.1% error budget                        │
│  AND                                                                       │
│  (                                                                         │
│    sum(rate(http_requests_total{status=~"5.."}[5m]))                      │
│    / sum(rate(http_requests_total[5m]))                                   │
│  ) > (14.4 * 0.001)                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 7: The Observability Stack Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY STACK COMPARISON                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    GRAFANA STACK (LGTM)                              │   │
│  │                                                                     │   │
│  │   L oki    - Logs (LogQL)        "Like Prometheus, but for logs"   │   │
│  │   G rafana - Visualization       "The dashboard layer"             │   │
│  │   T empo   - Traces (TraceQL)    "Cost-efficient trace storage"    │   │
│  │   M imir   - Metrics (PromQL)    "Scalable Prometheus"             │   │
│  │                                                                     │   │
│  │   + Grafana Alloy (formerly Agent) - Collection                    │   │
│  │   + Pyroscope - Continuous Profiling                               │   │
│  │                                                                     │   │
│  │   STRENGTHS: Unified UI, cost-effective, open source               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PROMETHEUS + JAEGER                               │   │
│  │                                                                     │   │
│  │   Prometheus    - Metrics (PromQL)                                  │   │
│  │   Alertmanager  - Alerting                                          │   │
│  │   Jaeger        - Traces                                            │   │
│  │   ELK/Loki      - Logs                                              │   │
│  │   Grafana       - Visualization                                     │   │
│  │                                                                     │   │
│  │   STRENGTHS: Mature, battle-tested, large community                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ELASTIC STACK (ELK)                               │   │
│  │                                                                     │   │
│  │   E lasticsearch - Storage & Search                                 │   │
│  │   L ogstash      - Data processing                                  │   │
│  │   K ibana        - Visualization                                    │   │
│  │   Beats          - Data collection                                  │   │
│  │   APM            - Application monitoring                           │   │
│  │                                                                     │   │
│  │   STRENGTHS: Full-text search, single stack, APM included          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    CLOUD PROVIDER NATIVE                             │   │
│  │                                                                     │   │
│  │   AWS:   CloudWatch, X-Ray, CloudWatch Logs                        │   │
│  │   GCP:   Cloud Monitoring, Cloud Trace, Cloud Logging              │   │
│  │   Azure: Azure Monitor, Application Insights                        │   │
│  │                                                                     │   │
│  │   STRENGTHS: No infra to manage, deep cloud integration            │   │
│  │   WEAKNESS:  Vendor lock-in, can be expensive at scale             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 8: Alerting Strategy Pyramid

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ALERTING STRATEGY PYRAMID                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              /\                                            │
│                             /  \      PAGE (Wake someone up)               │
│                            /    \     - Customer-impacting NOW             │
│                           / PAGE \    - Requires immediate action          │
│                          /________\   - SLO burn rate critical             │
│                         /          \                                       │
│                        /   TICKET   \  TICKET (Fix within hours/day)       │
│                       /              \ - Degraded but not critical         │
│                      /________________\- Trending toward SLO breach        │
│                     /                  \                                   │
│                    /     DASHBOARD      \ DASHBOARD (Informational)        │
│                   /                      \- Useful for investigation       │
│                  /__________INFO__________\- No immediate action needed    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  ALERT FATIGUE PREVENTION                            │   │
│  │                                                                     │   │
│  │  ✗ BAD: "CPU > 80%"          → May not indicate real problem       │   │
│  │  ✓ GOOD: "Error rate > 1%"    → Symptom-based, user-impacting      │   │
│  │                                                                     │   │
│  │  ✗ BAD: "Disk > 90%"          → Not immediately actionable         │   │
│  │  ✓ GOOD: "Disk will fill in <4h" → Predictive, actionable          │   │
│  │                                                                     │   │
│  │  ✗ BAD: Alert on every error  → Too noisy                          │   │
│  │  ✓ GOOD: Alert on error RATE  → Shows trend, reduces noise         │   │
│  │                                                                     │   │
│  │  RULE: Every alert should have a RUNBOOK                           │   │
│  │        If you can't write a runbook, don't create the alert        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Symptom vs Cause-Based Alerting

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   SYMPTOM vs CAUSE-BASED ALERTING                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CAUSE-BASED (Avoid as primary alerts)                              │   │
│  │                                                                     │   │
│  │  "CPU > 80%"           → May be normal under load                  │   │
│  │  "Memory > 90%"        → JVM might be designed this way            │   │
│  │  "Pod restarted"       → Could be normal scaling                   │   │
│  │  "Connection pool low" → Might handle it gracefully                │   │
│  │                                                                     │   │
│  │  PROBLEM: These don't tell you if users are affected               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SYMPTOM-BASED (Prefer these)                                       │   │
│  │                                                                     │   │
│  │  "Error rate > 1%"              → Users seeing errors              │   │
│  │  "p99 latency > 2s"             → Users experiencing slowness      │   │
│  │  "Checkout success rate < 99%"  → Revenue impact                   │   │
│  │  "API availability < 99.9%"     → SLO violation                    │   │
│  │                                                                     │   │
│  │  BENEFIT: Directly tied to user experience                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BEST PRACTICE:                                                            │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                                                                   │     │
│  │  PAGE on SYMPTOMS  → "Error rate spiked"                         │     │
│  │                          │                                        │     │
│  │                          ▼                                        │     │
│  │  DEBUG with CAUSES → Dashboard shows "CPU: 100%, DB: slow"       │     │
│  │                                                                   │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 9: Correlation - Connecting the Three Pillars

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PILLAR CORRELATION STRATEGIES                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  THE DEBUGGING FLOW:                                                       │
│                                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │  ALERT   │───▶│ METRICS  │───▶│  TRACES  │───▶│   LOGS   │             │
│  │          │    │          │    │          │    │          │             │
│  │"Error    │    │"Error    │    │"Find     │    │"Read     │             │
│  │ rate up!"│    │ rate     │    │ slow     │    │ error    │             │
│  │          │    │ graph"   │    │ trace"   │    │ message" │             │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘             │
│                                                                             │
│                                                                             │
│  CORRELATION MECHANISMS:                                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. EXEMPLARS (Metrics → Traces)                                    │   │
│  │                                                                     │   │
│  │  Metric: http_request_duration_seconds (histogram)                 │   │
│  │           │                                                         │   │
│  │           └─ Exemplar: {trace_id="abc123"} → Links to slow trace   │   │
│  │                                                                     │   │
│  │  In Grafana: Click on high latency point → Jump to trace           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  2. TRACE CONTEXT IN LOGS (Traces ↔ Logs)                           │   │
│  │                                                                     │   │
│  │  Log entry:                                                        │   │
│  │  {                                                                 │   │
│  │    "message": "Payment failed",                                    │   │
│  │    "trace_id": "abc123",     ← Links to trace                     │   │
│  │    "span_id": "def456",      ← Links to specific span             │   │
│  │    "error": "Card declined"                                        │   │
│  │  }                                                                 │   │
│  │                                                                     │   │
│  │  In Grafana: View trace → See "Logs for this span" panel           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  3. DERIVED METRICS FROM TRACES (Traces → Metrics)                  │   │
│  │                                                                     │   │
│  │  Tempo Metrics Generator / Span Metrics:                           │   │
│  │                                                                     │   │
│  │  Traces ──▶ traces_spanmetrics_latency_bucket{service="api"}       │   │
│  │             traces_spanmetrics_calls_total{service="api"}          │   │
│  │                                                                     │   │
│  │  Benefit: Get RED metrics automatically from traces                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 10: Sampling Strategies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SAMPLING DECISION TREE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WHY SAMPLE?                                                               │
│  - Traces are expensive (storage, network, processing)                     │
│  - 100% sampling at scale = bankruptcy                                     │
│  - Most traces are "normal" (not interesting)                              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     SAMPLING TYPES                                   │   │
│  │                                                                     │   │
│  │  HEAD-BASED                      TAIL-BASED                         │   │
│  │  (Decision at START)             (Decision at END)                  │   │
│  │                                                                     │   │
│  │  Request arrives                 Request completes                  │   │
│  │       │                                │                            │   │
│  │       ▼                                ▼                            │   │
│  │  ┌─────────┐                    ┌─────────────┐                     │   │
│  │  │ Random  │                    │ Analyze the │                     │   │
│  │  │ 10%?    │                    │ whole trace │                     │   │
│  │  └────┬────┘                    └──────┬──────┘                     │   │
│  │    Y  │  N                          │                               │   │
│  │    ▼  ▼                             ▼                               │   │
│  │  Keep Discard               Is it interesting?                      │   │
│  │                             - Errors? Keep!                         │   │
│  │  PROS:                      - Slow? Keep!                           │   │
│  │  - Simple                   - Normal? Maybe discard                 │   │
│  │  - Predictable                                                      │   │
│  │                             PROS:                                   │   │
│  │  CONS:                      - Keeps interesting traces              │   │
│  │  - Misses rare errors       - Smart about what matters              │   │
│  │  - Can't know if trace                                              │   │
│  │    is interesting yet       CONS:                                   │   │
│  │                             - Complex (needs all spans first)       │   │
│  │                             - Higher resource usage                 │   │
│  │                             - Requires collector buffering          │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RECOMMENDED STRATEGY:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. ALWAYS SAMPLE:                                                  │   │
│  │     - Errors (100%)                                                 │   │
│  │     - Slow requests > p99 (100%)                                    │   │
│  │     - New deployments (higher rate for first hour)                  │   │
│  │                                                                     │   │
│  │  2. PROBABILISTIC SAMPLE:                                           │   │
│  │     - Normal requests (1-10%)                                       │   │
│  │                                                                     │   │
│  │  3. RATE LIMIT:                                                     │   │
│  │     - Cap at X traces/second per service                           │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 11: Cardinality Management

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CARDINALITY: THE SILENT KILLER                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CARDINALITY = Number of unique time series                                │
│                                                                             │
│  FORMULA:                                                                  │
│  Total Series = metric_name × label1_values × label2_values × ...         │
│                                                                             │
│  EXAMPLE:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  http_requests_total{                                               │   │
│  │    method="GET|POST|PUT|DELETE",     # 4 values                    │   │
│  │    status="200|201|400|401|404|500", # 6 values                    │   │
│  │    endpoint="/api/v1/users|..."       # 50 endpoints               │   │
│  │  }                                                                  │   │
│  │                                                                     │   │
│  │  Series = 4 × 6 × 50 = 1,200 series   ✓ Manageable                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CARDINALITY EXPLOSION (BAD):                                       │   │
│  │                                                                     │   │
│  │  http_requests_total{                                               │   │
│  │    method="...",                                                    │   │
│  │    status="...",                                                    │   │
│  │    endpoint="...",                                                  │   │
│  │    user_id="..."        # 1,000,000 users!!!                       │   │
│  │  }                                                                  │   │
│  │                                                                     │   │
│  │  Series = 4 × 6 × 50 × 1,000,000 = 1.2 BILLION series   ✗ DEATH   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  HIGH-CARDINALITY LABELS TO AVOID IN METRICS:                       │   │
│  │                                                                     │   │
│  │  ✗ user_id                                                          │   │
│  │  ✗ request_id / trace_id                                            │   │
│  │  ✗ session_id                                                       │   │
│  │  ✗ email addresses                                                  │   │
│  │  ✗ IP addresses (unbounded)                                         │   │
│  │  ✗ timestamp as label                                               │   │
│  │  ✗ UUIDs of any kind                                                │   │
│  │                                                                     │   │
│  │  THESE BELONG IN:                                                   │   │
│  │  ✓ Logs (user_id, request_id, etc.)                                │   │
│  │  ✓ Traces (as span attributes)                                     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SAFE CARDINALITY GUIDELINES:                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  Per metric:     < 1,000 series (ideal)                            │   │
│  │  Per service:    < 10,000 series                                   │   │
│  │  Total system:   < 1,000,000 series (Prometheus comfortable)       │   │
│  │                                                                     │   │
│  │  If higher → Use Mimir/Thanos/VictoriaMetrics for horizontal scale │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 12: Grafana Dashboard Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DASHBOARD LAYOUT BEST PRACTICES                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  THE "NEWSPAPER" LAYOUT:                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ROW 1: THE HEADLINE (Overview stats)                               │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │  99.9%  │ │  250ms  │ │ 1.2K/s  │ │  0.1%   │ │   OK    │       │   │
│  │  │  Avail  │ │  p99    │ │  RPS    │ │ Errors  │ │  Status │       │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ROW 2: THE STORY (Time series trends)                              │   │
│  │  ┌─────────────────────────┐ ┌─────────────────────────┐            │   │
│  │  │     Request Rate        │ │    Error Rate           │            │   │
│  │  │   ╱╲    ╱╲             │ │                         │            │   │
│  │  │  ╱  ╲  ╱  ╲            │ │  ─────────────────────  │            │   │
│  │  │ ╱    ╲╱    ╲           │ │                    ╱    │            │   │
│  │  └─────────────────────────┘ └─────────────────────────┘            │   │
│  │  ┌─────────────────────────┐ ┌─────────────────────────┐            │   │
│  │  │     Latency (p50/p99)   │ │    Saturation           │            │   │
│  │  │   ___________________   │ │                         │            │   │
│  │  │  ─────────────────────  │ │  ███████░░░ 70%        │            │   │
│  │  └─────────────────────────┘ └─────────────────────────┘            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ROW 3: THE DETAILS (Breakdown by dimension)                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  Requests by Endpoint                                       │    │   │
│  │  │  /api/users     ████████████████████  45%                  │    │   │
│  │  │  /api/orders    ████████████  30%                          │    │   │
│  │  │  /api/products  ████████  20%                              │    │   │
│  │  │  /api/auth      ██  5%                                     │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ROW 4: THE INFRASTRUCTURE (Resource metrics)                       │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐           │   │
│  │  │  CPU 45%  │ │  Mem 60%  │ │  Disk 30% │ │  Net 20%  │           │   │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DASHBOARD TYPES:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. SERVICE DASHBOARD - One per service                            │   │
│  │     Focus: RED metrics, dependencies, resources                    │   │
│  │                                                                     │   │
│  │  2. OVERVIEW DASHBOARD - System-wide view                          │   │
│  │     Focus: All services at a glance, drill-down links              │   │
│  │                                                                     │   │
│  │  3. DEBUG DASHBOARD - For incident investigation                   │   │
│  │     Focus: Detailed metrics, logs integration, traces              │   │
│  │                                                                     │   │
│  │  4. BUSINESS DASHBOARD - For stakeholders                          │   │
│  │     Focus: Business metrics (orders, revenue, conversions)         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 13: Kubernetes Observability

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES OBSERVABILITY LAYERS                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 4: APPLICATION (Your Code)                                   │   │
│  │  ─────────────────────────────────                                  │   │
│  │  Metrics: Custom app metrics, business metrics                     │   │
│  │  Source:  App instrumentation (OpenTelemetry SDK)                  │   │
│  │  Example: orders_total, payment_latency_seconds                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 3: CONTAINER/POD                                             │   │
│  │  ────────────────────────                                           │   │
│  │  Metrics: CPU, memory, network per container                       │   │
│  │  Source:  cAdvisor (built into kubelet)                            │   │
│  │  Example: container_cpu_usage_seconds_total                        │   │
│  │           container_memory_usage_bytes                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 2: KUBERNETES OBJECTS                                        │   │
│  │  ─────────────────────────────                                      │   │
│  │  Metrics: Deployment replicas, pod phases, PVC status              │   │
│  │  Source:  kube-state-metrics                                        │   │
│  │  Example: kube_deployment_status_replicas_available                │   │
│  │           kube_pod_status_phase                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 1: NODE/INFRASTRUCTURE                                       │   │
│  │  ─────────────────────────────                                      │   │
│  │  Metrics: Node CPU, memory, disk, network                          │   │
│  │  Source:  node-exporter (Linux), kubelet metrics                   │   │
│  │  Example: node_cpu_seconds_total                                   │   │
│  │           node_memory_MemAvailable_bytes                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KUBERNETES MONITORING STACK:                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  kube-prometheus-stack (Helm chart) includes:                      │   │
│  │  ├── Prometheus Operator                                           │   │
│  │  ├── Prometheus (metrics storage)                                  │   │
│  │  ├── Alertmanager                                                  │   │
│  │  ├── Grafana (dashboards)                                          │   │
│  │  ├── node-exporter (node metrics)                                  │   │
│  │  ├── kube-state-metrics (K8s object metrics)                       │   │
│  │  └── Pre-built dashboards & alerts                                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 14: LogQL Query Patterns (Loki)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LogQL QUERY STRUCTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LogQL = PromQL concepts applied to logs                                   │
│                                                                             │
│  BASIC PATTERN:                                                            │
│  {stream_selector} |= "filter" | parser | label_filter | aggregation       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. STREAM SELECTOR (required) - Like PromQL label matchers         │   │
│  │                                                                     │   │
│  │  {job="api", env="prod"}           # Exact match                   │   │
│  │  {job=~"api|web"}                  # Regex match                   │   │
│  │  {job!="test"}                     # Not equal                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  2. LINE FILTERS (fast filtering)                                   │   │
│  │                                                                     │   │
│  │  |= "error"                        # Line contains "error"         │   │
│  │  != "debug"                        # Line doesn't contain "debug"  │   │
│  │  |~ "error|warning"                # Regex match                   │   │
│  │  !~ "health.*check"                # Regex doesn't match           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  3. PARSERS (extract labels from log content)                       │   │
│  │                                                                     │   │
│  │  | json                            # Parse JSON logs               │   │
│  │  | logfmt                          # Parse key=value format        │   │
│  │  | pattern "<_> <level> <msg>"     # Pattern extraction            │   │
│  │  | regexp "status=(?P<status>\\d+)" # Regex extraction             │   │
│  │                                                                     │   │
│  │  JSON Example:                                                      │   │
│  │  Log: {"level":"error","msg":"failed","user_id":"123"}             │   │
│  │                                                                     │   │
│  │  {job="api"} | json                                                │   │
│  │  # Creates labels: level="error", msg="failed", user_id="123"      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  4. LABEL FILTERS (filter on extracted labels)                      │   │
│  │                                                                     │   │
│  │  | level="error"                   # Exact match                   │   │
│  │  | status_code >= 400              # Numeric comparison            │   │
│  │  | duration > 1s                   # Duration comparison           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  5. METRIC QUERIES (convert logs to metrics)                        │   │
│  │                                                                     │   │
│  │  # Count errors per minute                                         │   │
│  │  sum(rate({job="api"} |= "error" [1m]))                            │   │
│  │                                                                     │   │
│  │  # P99 latency from JSON logs                                      │   │
│  │  quantile_over_time(0.99,                                          │   │
│  │    {job="api"} | json | unwrap duration [5m]                       │   │
│  │  )                                                                  │   │
│  │                                                                     │   │
│  │  # Bytes processed                                                  │   │
│  │  sum(bytes_rate({job="api"}[5m]))                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMON PATTERNS:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Find errors in last hour                                        │   │
│  │  {job="api"} |= "error" | json | line_format "{{.msg}}"           │   │
│  │                                                                     │   │
│  │  # Count 5xx errors by endpoint                                    │   │
│  │  sum by (endpoint) (                                               │   │
│  │    rate({job="api"} | json | status_code >= 500 [5m])             │   │
│  │  )                                                                  │   │
│  │                                                                     │   │
│  │  # Find slow requests (>1s) with context                           │   │
│  │  {job="api"} | json | duration > 1s                                │   │
│  │    | line_format "{{.method}} {{.path}} took {{.duration}}"       │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 15: Observability Cost Optimization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY COST REDUCTION STRATEGIES                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  COST DRIVERS:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. INGESTION VOLUME   (data coming in)                            │   │
│  │  2. STORAGE            (data retention)                            │   │
│  │  3. QUERY LOAD         (analysis/dashboards)                       │   │
│  │  4. CARDINALITY        (unique time series)                        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  OPTIMIZATION STRATEGIES BY PILLAR:                                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  METRICS COST REDUCTION                                             │   │
│  │                                                                     │   │
│  │  ┌────────────────────┬───────────────────────────────────────┐    │   │
│  │  │ Strategy           │ Implementation                        │    │   │
│  │  ├────────────────────┼───────────────────────────────────────┤    │   │
│  │  │ Drop unused metrics│ metric_relabel_configs: drop          │    │   │
│  │  │ Reduce cardinality │ Remove high-cardinality labels        │    │   │
│  │  │ Increase scrape    │ scrape_interval: 60s (from 15s)       │    │   │
│  │  │ interval           │ 4x reduction in data points           │    │   │
│  │  │ Use recording rules│ Pre-aggregate expensive queries       │    │   │
│  │  │ Downsample old data│ Thanos/Mimir compaction               │    │   │
│  │  └────────────────────┴───────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LOGS COST REDUCTION                                                │   │
│  │                                                                     │   │
│  │  ┌────────────────────┬───────────────────────────────────────┐    │   │
│  │  │ Strategy           │ Implementation                        │    │   │
│  │  ├────────────────────┼───────────────────────────────────────┤    │   │
│  │  │ Filter at source   │ Don't ship DEBUG in production        │    │   │
│  │  │ Drop noisy logs    │ Filter health checks, heartbeats      │    │   │
│  │  │ Reduce labels      │ Only essential labels (job, level)    │    │   │
│  │  │ Shorter retention  │ 7 days instead of 30 days             │    │   │
│  │  │ Compress           │ Enable gzip compression               │    │   │
│  │  │ Sample verbose logs│ Keep 10% of INFO logs                 │    │   │
│  │  └────────────────────┴───────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  TRACES COST REDUCTION                                              │   │
│  │                                                                     │   │
│  │  ┌────────────────────┬───────────────────────────────────────┐    │   │
│  │  │ Strategy           │ Implementation                        │    │   │
│  │  ├────────────────────┼───────────────────────────────────────┤    │   │
│  │  │ Head-based sampling│ Sample 1-10% of normal traces         │    │   │
│  │  │ Tail-based sampling│ Keep 100% errors, slow requests       │    │   │
│  │  │ Reduce span count  │ Don't trace internal loops            │    │   │
│  │  │ Limit attributes   │ Cap attribute count per span          │    │   │
│  │  │ Shorter retention  │ 3-7 days for most cases               │    │   │
│  │  └────────────────────┴───────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RETENTION TIERS:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  HOT   (0-7 days)   │ Full resolution, fast queries          $$$  │   │
│  │  WARM  (7-30 days)  │ Downsampled, slower storage            $$   │   │
│  │  COLD  (30-90 days) │ Highly compressed, archive             $    │   │
│  │  DELETE (90+ days)  │ Purge old data                         FREE │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 16: Observability Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY QUICK REFERENCE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROMETHEUS METRIC TYPES:                                                  │
│  ┌──────────────┬─────────────────────────────────────────────────────┐   │
│  │ Counter      │ Only increases (requests_total)       Use: rate()  │   │
│  │ Gauge        │ Goes up/down (temperature)            Use: direct  │   │
│  │ Histogram    │ Distribution (latency_bucket)         Use: quantile│   │
│  │ Summary      │ Pre-calculated percentiles            Use: direct  │   │
│  └──────────────┴─────────────────────────────────────────────────────┘   │
│                                                                             │
│  ESSENTIAL PromQL:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ rate(counter[5m])                   # Per-second rate              │   │
│  │ increase(counter[1h])               # Total increase               │   │
│  │ histogram_quantile(0.99, rate(...)) # P99 latency                  │   │
│  │ sum by (label) (metric)             # Aggregate by label           │   │
│  │ avg_over_time(gauge[1h])            # Average over time            │   │
│  │ absent(metric)                      # Alert if metric missing      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ESSENTIAL LogQL:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ {job="api"} |= "error"              # Find errors                  │   │
│  │ {job="api"} | json | level="error"  # Parse JSON, filter level     │   │
│  │ sum(rate({job="api"}[5m]))          # Log rate                     │   │
│  │ {job="api"} | json | unwrap latency # Extract numeric field        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SLO MATH CHEAT SHEET:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ SLO 99.9%  → 43.2 min/month  → 8.64 sec/day  downtime budget      │   │
│  │ SLO 99.95% → 21.6 min/month  → 4.32 sec/day  downtime budget      │   │
│  │ SLO 99.99% → 4.32 min/month  → 0.86 sec/day  downtime budget      │   │
│  │                                                                     │   │
│  │ Burn rate 1x  = exhaust budget in 30 days                          │   │
│  │ Burn rate 10x = exhaust budget in 3 days                           │   │
│  │ Burn rate 36x = exhaust budget in 20 hours (PAGE!)                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  OTEL COLLECTOR COMPONENTS:                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Receivers:   otlp, prometheus, jaeger, filelog                     │   │
│  │ Processors:  batch, memory_limiter, attributes, filter             │   │
│  │ Exporters:   otlp, prometheus, loki, debug                         │   │
│  │ Extensions:  health_check, pprof, zpages                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEBUGGING WORKFLOW:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Alert fires (symptom)                                           │   │
│  │ 2. Check dashboard (scope the problem)                             │   │
│  │ 3. Find example trace (identify bottleneck)                        │   │
│  │ 4. Read logs for that trace_id (root cause)                        │   │
│  │ 5. Fix → Monitor → Close                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 17: Learning Path

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY LEARNING PATH                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PHASE 1: FOUNDATIONS (Week 1-2)                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Understand three pillars (metrics, logs, traces)                 │   │
│  │ □ Learn USE, RED, Golden Signals methodologies                     │   │
│  │ □ Set up local Prometheus + Grafana                                │   │
│  │ □ Create your first dashboard with 4 Golden Signals                │   │
│  │ □ Write basic PromQL queries (rate, sum, histogram_quantile)       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 2: METRICS DEEP DIVE (Week 3-4)                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Instrument an app with Prometheus client library                 │   │
│  │ □ Create custom metrics (counter, gauge, histogram)                │   │
│  │ □ Configure Alertmanager with routing & receivers                  │   │
│  │ □ Write recording rules for expensive queries                      │   │
│  │ □ Understand cardinality and its impact                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 3: LOGGING (Week 5-6)                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Set up Loki + Promtail                                           │   │
│  │ □ Configure structured logging in your app                         │   │
│  │ □ Learn LogQL (filters, parsers, aggregations)                     │   │
│  │ □ Correlate logs with metrics in Grafana                           │   │
│  │ □ Set up log-based alerts                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 4: DISTRIBUTED TRACING (Week 7-8)                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Set up Tempo or Jaeger                                           │   │
│  │ □ Instrument app with OpenTelemetry SDK                            │   │
│  │ □ Propagate context across service boundaries                      │   │
│  │ □ Analyze traces to find bottlenecks                               │   │
│  │ □ Configure sampling strategies                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 5: OPENTELEMETRY & INTEGRATION (Week 9-10)                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Deploy OpenTelemetry Collector                                   │   │
│  │ □ Configure receivers, processors, exporters                       │   │
│  │ □ Add exemplars for metrics-to-traces correlation                  │   │
│  │ □ Include trace_id in logs for full correlation                    │   │
│  │ □ Build unified debugging workflow                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 6: SRE PRACTICES (Week 11-12)                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Define SLIs and SLOs for your services                           │   │
│  │ □ Implement error budgets and burn rate alerting                   │   │
│  │ □ Create runbooks for every alert                                  │   │
│  │ □ Practice incident response with observability tools              │   │
│  │ □ Optimize costs (retention, sampling, cardinality)                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 7: PRODUCTION EXCELLENCE (Ongoing)                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Scale with Mimir/Thanos for metrics                              │   │
│  │ □ Implement tail-based sampling                                    │   │
│  │ □ Add continuous profiling (Pyroscope)                             │   │
│  │ □ Explore eBPF-based observability                                 │   │
│  │ □ Implement GitOps for observability config                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  HANDS-ON PROJECTS:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Instrument a microservices demo (e.g., Online Boutique)         │   │
│  │ 2. Create an SLO dashboard with burn-rate alerts                   │   │
│  │ 3. Build a debugging workflow: alert → metrics → traces → logs     │   │
│  │ 4. Simulate an incident and trace it to root cause                 │   │
│  │ 5. Optimize observability costs by 50%                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

This mental models guide covers:

1. **Three Pillars Hospital Analogy** - Metrics, Logs, Traces explained
2. **USE/RED/Golden Signals** - Choosing the right methodology
3. **Prometheus Architecture** - Pull model, PromQL, metric types
4. **OpenTelemetry Collector** - Pipeline architecture
5. **Distributed Tracing** - Spans, context propagation
6. **SLOs & Error Budgets** - SLA → SLO → SLI hierarchy, burn rates
7. **Stack Comparison** - Grafana LGTM, ELK, Cloud native
8. **Alerting Strategy** - Pyramid, symptom vs cause
9. **Correlation** - Connecting pillars with exemplars, trace context
10. **Sampling Strategies** - Head vs tail-based
11. **Cardinality Management** - Avoiding metric explosion
12. **Dashboard Design** - Newspaper layout, types
13. **Kubernetes Observability** - Four layers
14. **LogQL Patterns** - Loki query building
15. **Cost Optimization** - Reduction strategies by pillar
16. **Quick Reference Card** - Essential commands
17. **Learning Path** - 12-week progression
