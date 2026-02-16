# Project 02: Load Balancer & Health Check System

## Difficulty: Intermediate
**Estimated Time**: 2-3 weeks

## Problem Statement

Design a Load Balancer like HAProxy, NGINX, or AWS ELB that distributes incoming traffic across multiple backend servers, handles health checks, and ensures high availability.

**Real-World Analogy**: Think of a restaurant host who decides which table (server) to seat incoming customers (requests). They check which tables are available (healthy), not too busy, and assign customers accordingly.

## Functional Requirements

### Core Features:
1. **Traffic Distribution**: Route requests to backend servers using various algorithms
2. **Health Checking**: Monitor server health and remove unhealthy servers from pool
3. **Session Persistence**: Route same user to same server (sticky sessions)
4. **SSL Termination**: Handle HTTPS encryption/decryption
5. **Connection Pooling**: Reuse connections to backend servers
6. **Rate Limiting**: Limit requests per IP/user

### Out of Scope:
- DDoS protection
- WAF (Web Application Firewall)
- Content caching

## Non-Functional Requirements

| Requirement | Target | Measurement |
|------------|--------|-------------|
| **Availability** | 99.99% | < 53 minutes downtime/year |
| **Latency** | < 10ms overhead | Time added by load balancer |
| **Throughput** | 100K requests/sec | Per load balancer instance |
| **Connection Limit** | 1M concurrent | Active connections |
| **Failover Time** | < 5 seconds | Time to detect and remove failed server |
| **CPU Usage** | < 70% | At peak load |

## Capacity Estimation

### Assumptions:
- **100K requests/second** peak traffic
- **1 million concurrent connections**
- **Average request size**: 10 KB
- **Average response size**: 50 KB
- **100 backend servers** in pool

### Bandwidth Calculation:
```
Incoming: 100K RPS × 10 KB = 1 GB/s = 8 Gbps
Outgoing: 100K RPS × 50 KB = 5 GB/s = 40 Gbps
Total: 48 Gbps

With 10% overhead for health checks, metrics:
Required: ~55 Gbps
```

### Connection Calculation:
```
Concurrent connections: 1M

Connections per backend server:
  1M / 100 servers = 10K connections/server

Connection pool size (LB to backends):
  100 servers × 1K pooled connections = 100K connections
```

### Memory Calculation:
```
Per connection overhead: ~64 KB (buffers, state)
1M connections × 64 KB = 64 GB

Session state (sticky sessions):
  1M sessions × 128 bytes = 128 MB

Health check data:
  100 servers × 1 KB = 100 KB

Total memory: ~65 GB per LB instance
```

## High-Level Architecture

```
                        Internet
                            │
                            ▼
                  ┌──────────────────┐
                  │   DNS (Route53)  │
                  │  Returns LB IPs  │
                  └─────────┬────────┘
                            │
           ┌────────────────┴────────────────┐
           │                                 │
           ▼                                 ▼
    ┌──────────────┐                  ┌──────────────┐
    │  Load Bal #1 │                  │  Load Bal #2 │
    │  (Primary)   │◄────────────────►│  (Standby)   │
    └──────┬───────┘   Health Check   └──────┬───────┘
           │           VRRP/Keepalived        │
           │                                  │
    ┌──────┴──────────────────────────────────┴──────┐
    │                                                 │
    │            Backend Server Pool                  │
    │                                                 │
    │  ┌────────┐  ┌────────┐  ┌────────┐           │
    │  │Server 1│  │Server 2│  │Server N│           │
    │  │Healthy │  │Healthy │  │Unhealthy│          │
    │  └────────┘  └────────┘  └────────┘           │
    │                                                 │
    └─────────────────────────────────────────────────┘
```

## Load Balancing Algorithms

### 1. Round Robin

**How it works**: Distribute requests in circular order

```typescript
class RoundRobinLoadBalancer {
  private servers: Server[];
  private currentIndex: number = 0;

  selectServer(): Server {
    const server = this.servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.servers.length;
    return server;
  }
}
```

**Pros**: Simple, fair distribution
**Cons**: Doesn't consider server load or response time
**Use case**: Servers have similar capacity, requests have similar workload

### 2. Weighted Round Robin

**How it works**: Assign weights based on server capacity

```typescript
class WeightedRoundRobinLoadBalancer {
  private servers: Array<{ server: Server; weight: number }>;
  private currentIndex: number = 0;
  private currentWeight: number = 0;

  selectServer(): Server {
    while (true) {
      this.currentIndex = (this.currentIndex + 1) % this.servers.length;

      if (this.currentIndex === 0) {
        this.currentWeight = this.currentWeight - this.gcd();
        if (this.currentWeight <= 0) {
          this.currentWeight = this.maxWeight();
        }
      }

      if (this.servers[this.currentIndex].weight >= this.currentWeight) {
        return this.servers[this.currentIndex].server;
      }
    }
  }

  private maxWeight(): number {
    return Math.max(...this.servers.map(s => s.weight));
  }

  private gcd(): number {
    // Calculate GCD of all weights
    return this.servers.reduce((a, b) => this.gcdTwo(a, b.weight), 0);
  }

  private gcdTwo(a: number, b: number): number {
    return b === 0 ? a : this.gcdTwo(b, a % b);
  }
}

// Usage:
const lb = new WeightedRoundRobinLoadBalancer();
lb.addServer(server1, 5);  // Powerful server
lb.addServer(server2, 3);  // Medium server
lb.addServer(server3, 1);  // Weak server

// server1 gets 5/9 requests, server2 gets 3/9, server3 gets 1/9
```

**Use case**: Servers have different capacities (CPU, RAM)

### 3. Least Connections

**How it works**: Route to server with fewest active connections

```typescript
class LeastConnectionsLoadBalancer {
  private servers: Map<Server, number> = new Map(); // server → connection count

  selectServer(): Server {
    let minConnections = Infinity;
    let selectedServer: Server | null = null;

    for (const [server, connections] of this.servers) {
      if (connections < minConnections) {
        minConnections = connections;
        selectedServer = server;
      }
    }

    // Increment connection count
    this.servers.set(selectedServer!, minConnections + 1);
    return selectedServer!;
  }

  releaseConnection(server: Server) {
    const count = this.servers.get(server) || 0;
    this.servers.set(server, Math.max(0, count - 1));
  }
}
```

**Use case**: Long-lived connections (WebSockets, streaming)

### 4. Weighted Least Connections

```typescript
selectServer(): Server {
  let minRatio = Infinity;
  let selectedServer: Server | null = null;

  for (const [server, connections] of this.servers) {
    const weight = this.weights.get(server) || 1;
    const ratio = connections / weight;

    if (ratio < minRatio) {
      minRatio = ratio;
      selectedServer = server;
    }
  }

  return selectedServer!;
}
```

### 5. IP Hash (Consistent Hashing)

**How it works**: Hash client IP to determine server

```typescript
import crypto from 'crypto';

class IPHashLoadBalancer {
  private servers: Server[];

  selectServer(clientIP: string): Server {
    const hash = crypto
      .createHash('md5')
      .update(clientIP)
      .digest('hex');

    const index = parseInt(hash.substring(0, 8), 16) % this.servers.length;
    return this.servers[index];
  }
}
```

**Pros**: Same client always goes to same server (natural sticky sessions)
**Cons**: Uneven distribution if client IPs are not uniformly distributed
**Use case**: Session affinity without cookies

### 6. Least Response Time

**How it works**: Route to server with lowest average response time

```typescript
class LeastResponseTimeLoadBalancer {
  private servers: Map<Server, ResponseTimeMetrics> = new Map();

  selectServer(): Server {
    let minTime = Infinity;
    let selectedServer: Server | null = null;

    for (const [server, metrics] of this.servers) {
      const avgTime = metrics.totalTime / metrics.requestCount;
      const weightedTime = avgTime * (1 + metrics.activeConnections * 0.1);

      if (weightedTime < minTime) {
        minTime = weightedTime;
        selectedServer = server;
      }
    }

    return selectedServer!;
  }

  recordResponse(server: Server, responseTime: number) {
    const metrics = this.servers.get(server)!;
    metrics.totalTime += responseTime;
    metrics.requestCount++;
  }
}
```

**Use case**: Complex requests with varying processing times

## Health Checking

### Active Health Checks

```typescript
interface HealthCheckConfig {
  interval: number;          // Check every 5 seconds
  timeout: number;           // 3 seconds timeout
  healthyThreshold: number;  // 2 successes = healthy
  unhealthyThreshold: number;// 3 failures = unhealthy
  path: string;              // "/health"
}

class HealthChecker {
  private serverHealth = new Map<Server, HealthStatus>();

  async checkHealth(server: Server, config: HealthCheckConfig) {
    const status = this.serverHealth.get(server) || {
      isHealthy: true,
      consecutiveSuccesses: 0,
      consecutiveFailures: 0
    };

    try {
      const response = await fetch(`${server.url}${config.path}`, {
        timeout: config.timeout
      });

      if (response.status === 200) {
        status.consecutiveSuccesses++;
        status.consecutiveFailures = 0;

        // Mark healthy after threshold
        if (status.consecutiveSuccesses >= config.healthyThreshold) {
          status.isHealthy = true;
        }
      } else {
        this.handleFailure(status, config);
      }
    } catch (error) {
      this.handleFailure(status, config);
    }

    this.serverHealth.set(server, status);
  }

  private handleFailure(status: HealthStatus, config: HealthCheckConfig) {
    status.consecutiveFailures++;
    status.consecutiveSuccesses = 0;

    // Mark unhealthy after threshold
    if (status.consecutiveFailures >= config.unhealthyThreshold) {
      status.isHealthy = false;
      this.notifyUnhealthy(server);
    }
  }

  isHealthy(server: Server): boolean {
    return this.serverHealth.get(server)?.isHealthy ?? false;
  }
}
```

### Passive Health Checks

Monitor actual traffic and mark servers unhealthy based on errors:

```typescript
class PassiveHealthChecker {
  private errorCounts = new Map<Server, number>();

  recordRequest(server: Server, success: boolean) {
    if (!success) {
      const errors = (this.errorCounts.get(server) || 0) + 1;
      this.errorCounts.set(server, errors);

      // If error rate > 50% in last 10 requests, mark unhealthy
      if (errors > 5) {
        this.markUnhealthy(server);
      }
    } else {
      // Reset error count on success
      this.errorCounts.set(server, 0);
    }
  }
}
```

### Health Check Endpoints

Backend servers should implement:

```typescript
// Simple health check
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Detailed health check
app.get('/health/detailed', async (req, res) => {
  const dbHealthy = await checkDatabase();
  const cacheHealthy = await checkRedis();
  const diskSpace = await checkDiskSpace();

  if (dbHealthy && cacheHealthy && diskSpace > 10) {
    res.status(200).json({
      status: 'healthy',
      checks: {
        database: 'ok',
        cache: 'ok',
        disk_space_gb: diskSpace
      }
    });
  } else {
    res.status(503).json({
      status: 'unhealthy',
      checks: {
        database: dbHealthy ? 'ok' : 'fail',
        cache: cacheHealthy ? 'ok' : 'fail',
        disk_space_gb: diskSpace
      }
    });
  }
});
```

## Session Persistence (Sticky Sessions)

### Cookie-based Stickiness

```typescript
class StickySessionLoadBalancer {
  private sessionToServer = new Map<string, Server>();

  selectServer(request: Request): Server {
    // Check for existing session cookie
    const sessionId = request.cookies['LB_SESSION'];

    if (sessionId && this.sessionToServer.has(sessionId)) {
      const server = this.sessionToServer.get(sessionId)!;

      // Verify server is still healthy
      if (this.isHealthy(server)) {
        return server;
      }
    }

    // No session or server unhealthy, select new server
    const newServer = this.loadBalancingAlgorithm.selectServer();
    const newSessionId = this.generateSessionId();

    this.sessionToServer.set(newSessionId, newServer);

    // Set cookie in response
    response.setCookie('LB_SESSION', newSessionId, {
      httpOnly: true,
      secure: true,
      maxAge: 3600 // 1 hour
    });

    return newServer;
  }

  private generateSessionId(): string {
    return crypto.randomBytes(32).toString('hex');
  }
}
```

### IP-based Stickiness

```typescript
class IPStickyLoadBalancer {
  selectServer(clientIP: string): Server {
    // Hash IP to consistent server
    const hash = crypto.createHash('md5').update(clientIP).digest('hex');
    const index = parseInt(hash.substring(0, 8), 16) % this.servers.length;
    return this.servers[index];
  }
}
```

## Connection Pooling

```typescript
class ConnectionPool {
  private pools = new Map<Server, Connection[]>();
  private maxPoolSize = 100;
  private minPoolSize = 10;

  async getConnection(server: Server): Promise<Connection> {
    let pool = this.pools.get(server);

    if (!pool) {
      pool = [];
      this.pools.set(server, pool);
    }

    // Try to get existing connection
    if (pool.length > 0) {
      const conn = pool.pop()!;

      // Verify connection is still alive
      if (conn.isAlive()) {
        return conn;
      }
    }

    // Create new connection
    return await this.createConnection(server);
  }

  releaseConnection(server: Server, conn: Connection) {
    const pool = this.pools.get(server)!;

    // Return to pool if under max size
    if (pool.length < this.maxPoolSize) {
      pool.push(conn);
    } else {
      conn.close();
    }
  }

  // Maintain minimum pool size
  async warmUp(server: Server) {
    const pool = this.pools.get(server) || [];

    while (pool.length < this.minPoolSize) {
      const conn = await this.createConnection(server);
      pool.push(conn);
    }

    this.pools.set(server, pool);
  }
}
```

## High Availability

### Active-Passive Setup

```
┌──────────────┐           ┌──────────────┐
│  Load Bal #1 │           │  Load Bal #2 │
│  (Primary)   │◄─────────►│  (Standby)   │
│  Active      │  Heartbeat│  Passive     │
└──────────────┘   VRRP    └──────────────┘
      │                          │
   Virtual IP               Standby
   1.2.3.4
```

**VRRP (Virtual Router Redundancy Protocol)**:
```
1. Both LBs monitor each other via heartbeat
2. Primary holds the Virtual IP (VIP)
3. If primary fails, standby takes over VIP
4. Failover time: ~2-5 seconds
```

### Active-Active Setup

```
┌──────────────┐           ┌──────────────┐
│  Load Bal #1 │           │  Load Bal #2 │
│  Active      │           │  Active      │
│  50% traffic │           │  50% traffic │
└──────────────┘           └──────────────┘
      │                          │
      └──────────┬───────────────┘
                 │
            DNS Round Robin
         (lb1.example.com + lb2.example.com)
```

**Benefits**:
- Better resource utilization
- No wasted standby capacity
- Higher total throughput

## SSL/TLS Termination

```typescript
class SSLTerminationLB {
  async handleRequest(request: HTTPSRequest): Promise<void> {
    // 1. Terminate SSL at load balancer
    const decrypted = await this.decryptSSL(request);

    // 2. Select backend server
    const server = this.selectServer();

    // 3. Forward as HTTP (or re-encrypt)
    const response = await this.forwardToBackend(server, decrypted);

    // 4. Encrypt response
    const encrypted = await this.encryptSSL(response);

    // 5. Send to client
    return encrypted;
  }
}
```

**Benefits**:
- Offload SSL/TLS processing from backend servers
- Centralized certificate management
- Inspect traffic for security

**Considerations**:
- Need to re-encrypt if backend traffic must be secure (end-to-end encryption)
- SSL/TLS adds ~5-10ms latency

## API Endpoints

```http
# Admin API

# Add server to pool
POST /api/v1/servers
Body: {
  "host": "backend-1.example.com",
  "port": 8080,
  "weight": 5,
  "health_check_path": "/health"
}
Response: 201 Created

# Remove server
DELETE /api/v1/servers/{server_id}

# Get server status
GET /api/v1/servers
Response: {
  "servers": [
    {
      "id": "srv-1",
      "host": "backend-1.example.com",
      "port": 8080,
      "status": "healthy",
      "active_connections": 1234,
      "requests_per_second": 500,
      "avg_response_time_ms": 45
    }
  ]
}

# Get metrics
GET /api/v1/metrics
Response: {
  "total_requests": 1000000000,
  "requests_per_second": 50000,
  "active_connections": 500000,
  "avg_response_time_ms": 50,
  "error_rate": 0.01,
  "backends_healthy": 98,
  "backends_unhealthy": 2
}
```

## Trade-off Analysis

### Layer 4 vs Layer 7 Load Balancing

| Aspect | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
|--------|------------------|---------------------|
| **What it sees** | IP, Port | Full HTTP request (URL, headers, cookies) |
| **Routing** | Based on IP/Port | Based on URL path, headers, cookies |
| **Performance** | Faster (~100K RPS) | Slower (~50K RPS) due to parsing |
| **Features** | Limited | Advanced (path routing, sticky sessions) |
| **SSL Termination** | No | Yes |
| **Use Case** | TCP/UDP protocols, max performance | HTTP applications, need advanced routing |

**Example**:
```
Layer 4:
- Client connects to LB
- LB forwards TCP packets to backend
- Cannot route based on URL path

Layer 7:
- Client sends HTTP request to LB
- LB parses request, can route:
  - /api/* → API servers
  - /static/* → Static file servers
  - /admin/* → Admin servers
```

## Monitoring & Alerting

### Key Metrics

```yaml
Health Metrics:
  - healthy_backends: Number of healthy servers
  - unhealthy_backends: Number of unhealthy servers
  - backend_response_time: P50, P95, P99 latency

Load Metrics:
  - requests_per_second: Current RPS
  - active_connections: Current connections
  - connections_per_backend: Connections to each server

Error Metrics:
  - error_rate: % of requests with 5xx errors
  - timeout_rate: % of requests timing out
  - health_check_failures: Failed health checks
```

### Alerting Rules

```yaml
alerts:
  - name: TooFewHealthyBackends
    condition: healthy_backends < 3
    severity: critical
    action: Add more servers or investigate failures

  - name: HighErrorRate
    condition: error_rate > 5%
    duration: 5 minutes
    severity: critical

  - name: HighLatency
    condition: p95_response_time > 500ms
    duration: 5 minutes
    severity: warning
```

## Summary

### What We Built:
- Load balancer handling 100K RPS
- Multiple load balancing algorithms
- Active/passive health checking
- Sticky sessions
- Connection pooling
- High availability with failover

### Key Concepts:
- ✅ Load balancing algorithms
- ✅ Health checking strategies
- ✅ Session persistence
- ✅ Connection pooling
- ✅ High availability (active-passive, active-active)
- ✅ SSL/TLS termination

### Real-World Examples:
- HAProxy: 2M+ concurrent connections
- NGINX: 10K-100K RPS per instance
- AWS ELB: Auto-scaling, managed service

---

**Next Project**: [03. Distributed Rate Limiting Service](../03-rate-limiter/README.md)
