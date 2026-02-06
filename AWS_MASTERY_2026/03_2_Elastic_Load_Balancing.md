# AWS Elastic Load Balancing - Conceptual Deep Dive

## Overview

**Elastic Load Balancing (ELB)** automatically distributes incoming application traffic across multiple targets (EC2 instances, containers, IP addresses, Lambda functions). Think of it as a "traffic cop" that directs requests to the healthiest, most available servers.

**The core problems ELB solves**:

**Without a load balancer**:
```
User requests → Single server (IP: 54.123.45.67)
Problems:
- Server fails → Entire application down
- High traffic → Single server overwhelmed
- Maintenance → Must take application offline
- Scaling → Users must know new server IPs
```

**With a load balancer**:
```
User requests → Load Balancer (single DNS name)
                    ↓
    ┌───────────────┼───────────────┐
    ↓               ↓               ↓
Server 1        Server 2        Server 3

Benefits:
- Server fails → Traffic routes to healthy servers (high availability)
- High traffic → Distributed across all servers (horizontal scaling)
- Maintenance → Drain one server, others continue serving
- Scaling → Add/remove servers transparently (same DNS name)
```

**AWS offers three types of load balancers** (as of 2026):
1. **Application Load Balancer (ALB)** - HTTP/HTTPS traffic (Layer 7)
2. **Network Load Balancer (NLB)** - TCP/UDP/TLS traffic (Layer 4)
3. **Gateway Load Balancer (GWLB)** - Network/security appliances (Layer 3)

*Classic Load Balancer (CLB) - Legacy, being phased out*

---

## Application Load Balancer (ALB)

**What ALB is**: A Layer 7 (HTTP/HTTPS) load balancer that routes traffic based on content of the request.

**Why "Layer 7" matters**: ALB understands HTTP, so it can route based on:
- URL path (`/api/*` → API servers, `/images/*` → image servers)
- Hostname (`api.example.com` → API servers, `www.example.com` → web servers)
- HTTP headers (User-Agent, custom headers)
- Query strings (`?version=v2` → new version servers)

**When to use ALB**:
- Modern web applications (REST APIs, microservices)
- Containerized applications (ECS, EKS)
- Applications needing advanced routing
- HTTP/HTTPS traffic

### ALB Architecture

```
Internet
    ↓
Application Load Balancer
├─ Listener 1: Port 80 (HTTP)
│  └─ Rule: Redirect to HTTPS
└─ Listener 2: Port 443 (HTTPS)
   ├─ Rule 1: Path /api/* → API Target Group
   ├─ Rule 2: Path /images/* → Image Server Target Group
   ├─ Rule 3: Host api.example.com → API Target Group
   └─ Default Rule → Web Server Target Group

Target Groups:
├─ API Target Group: [instance-1, instance-2, instance-3]
├─ Image Target Group: [instance-4, instance-5]
└─ Web Server Target Group: [instance-6, instance-7, instance-8]
```

### ALB Key Features

#### 1. Path-Based Routing

**Route requests based on URL path**:
```
User request: https://example.com/api/users
ALB evaluates:
- Path: /api/users
- Matches rule: /api/*
- Routes to: API Target Group

User request: https://example.com/checkout
ALB evaluates:
- Path: /checkout
- No specific rule matches
- Routes to: Default Target Group (web servers)
```

**Real-world use case - Microservices architecture**:
```
Single ALB fronting multiple microservices:
- /users/* → User Service (instances running user microservice)
- /orders/* → Order Service (instances running order microservice)
- /payments/* → Payment Service (instances running payment microservice)
- /* → Frontend Service (instances serving web UI)

Benefit: One load balancer, one domain, multiple backend services
Cost: $16/month for ALB vs $64/month for 4 separate ALBs
```

#### 2. Host-Based Routing

**Route requests based on hostname**:
```
ALB with two domains:
- api.example.com → API Target Group
- www.example.com → Web Target Group

DNS configuration:
- api.example.com → CNAME: alb-1234.us-east-1.elb.amazonaws.com
- www.example.com → CNAME: alb-1234.us-east-1.elb.amazonaws.com

Single ALB serves both domains, routes intelligently
```

**Real-world use case - Multi-tenant SaaS**:
```
tenant1.saas.com → Tenant 1 Target Group
tenant2.saas.com → Tenant 2 Target Group
tenant3.saas.com → Tenant 3 Target Group

Each tenant gets isolated backend infrastructure
Single ALB reduces costs, simplifies management
```

#### 3. HTTP Header Routing

**Route based on custom HTTP headers**:
```
Rule: If header "X-API-Version: v2" → New API Target Group
Otherwise → Legacy API Target Group

Use case: Gradual API migration
- Mobile app v1: Sends X-API-Version: v1 → Old API
- Mobile app v2: Sends X-API-Version: v2 → New API
- Allows running both versions simultaneously
```

#### 4. Query String Routing

**Route based on URL parameters**:
```
https://example.com/search?version=beta → Beta Target Group
https://example.com/search → Production Target Group

Use case: Feature flags, A/B testing
```

#### 5. WebSocket Support

**Maintain persistent connections**:
```
WebSocket connection: Client ↔ ALB ↔ Backend server
Connection remains open (bidirectional, real-time)

Use cases:
- Chat applications
- Real-time dashboards
- Gaming servers
- Collaborative editing tools
```

**How ALB handles WebSockets**:
- Detects WebSocket upgrade request
- Establishes persistent connection
- Routes all messages to same backend instance (sticky connection)
- Connection lasts until client/server closes it

#### 6. HTTP/2 and gRPC Support

**HTTP/2 benefits**:
- Multiplexing: Multiple requests over single connection
- Header compression: Reduced bandwidth
- Server push: Proactive resource delivery

**gRPC support**:
```
gRPC microservices:
- ALB terminates HTTP/2 (client to ALB)
- Routes gRPC calls to backend services
- Health checks via gRPC protocol

Use case: Modern microservices (Kubernetes + gRPC)
```

#### 7. Lambda as Target

**Route traffic to Lambda functions**:
```
ALB Listener:
├─ Rule: Path /api/* → Lambda Function (API handler)
└─ Rule: Path /* → EC2 Target Group (web servers)

ALB invokes Lambda, passes HTTP request as event
Lambda returns HTTP response
ALB forwards response to client

Benefit: Serverless backends mixed with traditional servers
```

**Use case - Hybrid architecture**:
```
E-commerce site:
- Product pages: EC2 instances (high throughput, consistent load)
- Search API: Lambda (variable load, short duration)
- Image resize: Lambda (on-demand, no idle cost)

One ALB routes to both EC2 and Lambda based on path
```

---

## Network Load Balancer (NLB)

**What NLB is**: A Layer 4 (TCP/UDP/TLS) load balancer optimized for ultra-high performance and low latency.

**Key difference from ALB**: NLB doesn't understand HTTP. It routes based on IP address and port only (network layer).

**When to use NLB**:
- Ultra-low latency required (microseconds matter)
- Millions of requests per second
- Static IP addresses needed
- Non-HTTP protocols (TCP, UDP, custom protocols)
- Preserve source IP address

### NLB vs ALB: Performance Comparison

| Metric | Network Load Balancer | Application Load Balancer |
|--------|----------------------|---------------------------|
| **Latency** | ~100 microseconds | ~10-20 milliseconds |
| **Throughput** | Millions requests/sec | Thousands requests/sec |
| **Connections** | Millions concurrent | Hundreds of thousands |
| **Protocol** | TCP, UDP, TLS (Layer 4) | HTTP, HTTPS (Layer 7) |
| **Routing** | IP + Port | URL, headers, host, etc. |
| **Cost** | $16/month + $0.006/LCU-hr | $16/month + $0.008/LCU-hr |

### NLB Key Features

#### 1. Ultra-Low Latency

**Why NLB is faster**:
```
ALB (Layer 7):
Client → ALB (terminate connection)
         ALB parses HTTP request
         ALB makes routing decision based on path/host
         ALB → Backend (new connection)
Overhead: Parse HTTP + routing logic = 10-20ms

NLB (Layer 4):
Client → NLB (passthrough)
         NLB looks at IP:Port only
         NLB → Backend (forward packets)
Overhead: Minimal packet forwarding = 0.1ms

Result: NLB is 100× lower latency
```

**Use case**: Financial trading systems, gaming, IoT, real-time bidding

#### 2. Static IP Addresses

**Problem with ALB**: ALB IP addresses change dynamically (you get DNS name only)
```
ALB DNS: alb-1234.us-east-1.elb.amazonaws.com
Resolves to: 54.123.45.67, 52.234.56.78, ... (changes over time)
```

**NLB provides static IPs**:
```
NLB per AZ:
- us-east-1a: 52.10.20.30 (static)
- us-east-1b: 52.10.20.31 (static)
- us-east-1c: 52.10.20.32 (static)

Benefits:
- Whitelist in firewalls (doesn't change)
- Compliance requirements (fixed IPs)
- DNS A records (instead of CNAME)
```

**Elastic IP support**:
```
Assign your own Elastic IPs to NLB:
- us-east-1a: Your EIP (52.50.60.70)
- us-east-1b: Your EIP (52.50.60.71)

Use case: Migration from on-premises (keep same IPs)
```

#### 3. Preserve Source IP Address

**ALB behavior** (connection termination):
```
Client (IP: 203.0.113.5) → ALB → Backend
Backend sees: ALB's private IP (10.0.1.50), not client IP

To get client IP: Parse X-Forwarded-For header
```

**NLB behavior** (passthrough):
```
Client (IP: 203.0.113.5) → NLB → Backend
Backend sees: Client's actual IP (203.0.113.5)

No header parsing needed, transparent to backend
```

**Use cases**:
- Application needs true client IP (geolocation, access control)
- Legacy applications that don't parse headers
- Non-HTTP protocols (can't use X-Forwarded-For)

#### 4. TLS Termination (NLB)

**NLB can terminate TLS**:
```
Client -(HTTPS)→ NLB -(HTTP)→ Backend
NLB decrypts traffic (SSL/TLS offload)
Backend receives plaintext HTTP

Benefits:
- Offload CPU-intensive crypto from backends
- Centralized certificate management
- Similar to ALB TLS termination
```

**NLB can also passthrough TLS**:
```
Client -(HTTPS)→ NLB -(HTTPS)→ Backend
NLB forwards encrypted traffic (no decryption)
Backend handles TLS termination

Use case: End-to-end encryption required (compliance)
```

#### 5. UDP Support

**Use cases for UDP load balancing**:
- **DNS servers**: DNS queries use UDP
- **Gaming**: Low-latency game servers
- **VoIP/video**: Real-time communication
- **IoT**: Sensor data transmission
- **Syslog**: Log aggregation

**Example - DNS server fleet**:
```
Network Load Balancer (UDP port 53)
    ↓
Target Group: [DNS-1, DNS-2, DNS-3]

Client sends DNS query → NLB routes to healthy DNS server
Low latency, high throughput, fault tolerance
```

---

## Gateway Load Balancer (GWLB)

**What GWLB is**: A specialized load balancer for deploying, scaling, and managing **third-party network appliances** (firewalls, intrusion detection, deep packet inspection).

**The problem GWLB solves**:
```
Without GWLB:
- Deploy firewall appliance: Manual configuration, complex routing
- Scale firewall: Hard (must update routing tables)
- High availability: Requires custom failover logic
```

**With GWLB**:
```
Traffic flow:
Internet → GWLB → Firewall Appliances → GWLB → Your Application

GWLB handles:
- Load distribution across firewalls
- Health checks (remove failed firewalls)
- Automatic scaling
- Transparent traffic routing (GENEVE encapsulation)
```

### GWLB Architecture

```
Internet Gateway
    ↓
GWLB Endpoint (in application VPC)
    ↓
Gateway Load Balancer
    ↓
Target Group: [Firewall-1, Firewall-2, Firewall-3]
    ↓ (traffic inspected/filtered)
    ↓
Gateway Load Balancer
    ↓
GWLB Endpoint (return path)
    ↓
Application instances
```

**Key insight**: Traffic goes through appliances transparently. Application doesn't know inspection happened.

### GWLB Use Cases

**1. Centralized Security Inspection**
```
Multi-account AWS organization:
- GWLB in security account
- All traffic routes through security appliances
- Firewalls, IDS/IPS inspect all traffic
- Centralized threat detection

Benefit: Consistent security across all accounts
```

**2. Compliance Requirements**
```
Regulation: All traffic must be inspected by approved firewall
Solution: Route all traffic through GWLB + certified firewall appliances
Audit: Logs prove all traffic inspected
```

**3. Third-Party Appliance Scaling**
```
Palo Alto / Fortinet / Check Point firewalls:
- Deploy as EC2 instances
- Register with GWLB Target Group
- Auto Scaling: Add/remove firewall instances based on load
- GWLB distributes traffic automatically

Benefit: Firewall fleet scales like application fleet
```

---

## Target Groups: The Backend Pool

**What Target Groups are**: A logical grouping of targets (instances, IPs, Lambda functions) that receive traffic from a load balancer.

**Target types**:
1. **Instance**: EC2 instance IDs
2. **IP**: IP addresses (instances, on-premises servers, containers)
3. **Lambda**: Lambda function ARN
4. **ALB**: Application Load Balancer (for NLB → ALB chaining)

### Target Group Configuration

```
Target Group: "web-app-tg"
├─ Protocol: HTTP
├─ Port: 80
├─ VPC: vpc-abc123
├─ Health Check:
│  ├─ Protocol: HTTP
│  ├─ Path: /health
│  ├─ Interval: 30 seconds
│  ├─ Timeout: 5 seconds
│  ├─ Healthy threshold: 2
│  └─ Unhealthy threshold: 3
├─ Targets:
│  ├─ i-0abc123 (healthy)
│  ├─ i-0def456 (healthy)
│  ├─ i-0ghi789 (unhealthy) ← Not receiving traffic
│  └─ i-0jkl012 (healthy)
└─ Load Balancing Algorithm: Round robin
```

### Load Balancing Algorithms

**1. Round Robin** (default for ALB)
```
Requests distributed evenly across all healthy targets:
Request 1 → Target A
Request 2 → Target B
Request 3 → Target C
Request 4 → Target A (cycle repeats)

Use case: Targets have equal capacity
```

**2. Least Outstanding Requests** (ALB only)
```
Route to target with fewest active requests:
Target A: 10 active requests
Target B: 5 active requests ← Send here
Target C: 12 active requests

Use case: Requests have variable duration
```

**3. Flow Hash** (NLB only)
```
Hash based on: protocol, source IP, source port, dest IP, dest port
Same hash → Same target (connection affinity)

Use case: Stateful connections (client always routes to same backend)
```

### Target Registration and Deregistration

**Registration**:
```
1. Add target to Target Group (manual or Auto Scaling)
2. Target enters "initial" state
3. Health checks begin
4. After passing health checks → "healthy" state
5. Load balancer starts sending traffic

Timeline: ~30-90 seconds from registration to receiving traffic
```

**Deregistration** (graceful shutdown):
```
1. Mark target for deregistration
2. Target enters "draining" state
3. Load balancer stops sending NEW requests
4. Existing connections allowed to complete
5. After drain timeout (default: 300s) → Force close remaining connections
6. Target fully deregistered

Connection draining prevents dropped requests during shutdown
```

---

## Health Checks: Determining Target Availability

**What health checks do**: Periodically test if targets are healthy and able to receive traffic.

### Health Check Configuration

```
Health Check Settings:
├─ Protocol: HTTP (or HTTPS, TCP)
├─ Port: 80 (or "traffic port" = same as target)
├─ Path: /health (HTTP/HTTPS only)
├─ Interval: 30 seconds
├─ Timeout: 5 seconds
├─ Healthy threshold: 2 consecutive successes
└─ Unhealthy threshold: 3 consecutive failures
```

### Health Check Process

```
Timeline example:
00:00 - Check 1: HTTP GET /health → 200 OK (1/2 healthy)
00:30 - Check 2: HTTP GET /health → 200 OK (2/2 healthy) ✓ Mark healthy
01:00 - Check 3: HTTP GET /health → Timeout (1/3 unhealthy)
01:30 - Check 4: HTTP GET /health → 500 Error (2/3 unhealthy)
02:00 - Check 5: HTTP GET /health → Timeout (3/3 unhealthy) ✗ Mark unhealthy
02:01 - Load balancer stops sending traffic to this target
```

### Health Check Best Practices

**1. Deep Health Checks** (Recommended)
```
Endpoint: GET /health
Logic:
- Check database connection: SELECT 1 (pass/fail)
- Check external API: Test critical dependency
- Check disk space: Ensure not full
- Return: 200 OK if all pass, 503 if any fail

Benefit: Catches application-level failures, not just "is server responding"
```

**2. Shallow Health Checks** (Basic)
```
Endpoint: GET /health
Logic: Return 200 OK (server is running)

Problem: Server running but database down → Health check passes, but app broken
```

**3. Balance Sensitivity**
```
Too sensitive (1 failure = unhealthy):
- Transient network blip → Target marked unhealthy
- Constant flapping (healthy → unhealthy → healthy)

Too lenient (10 failures = unhealthy):
- Target broken for 5 minutes before detected
- Users experience errors during detection window

Recommended: 2-3 failures for production
```

**4. Separate Health Check Port** (Advanced)
```
Application port: 80 (serves user traffic)
Health check port: 8080 (dedicated health endpoint)

Benefits:
- Health endpoint doesn't consume application resources
- Can implement different logic (admin-only checks)
- Better monitoring/debugging
```

---

## Sticky Sessions (Session Affinity)

**What sticky sessions are**: Ensure requests from same client always go to same target.

**Why needed**:
```
Without stickiness:
User session stored on Target A
Request 1 → Target A (session data available) ✓
Request 2 → Target B (no session data) ✗ User logged out

With stickiness:
User session on Target A
Request 1 → Target A ✓
Request 2 → Target A (same target) ✓
Request 3 → Target A (same target) ✓
```

### Sticky Session Types

**1. Duration-Based** (ALB)
```
Load balancer generates cookie: AWSALB=<target-id>
Cookie duration: 1 hour (configurable: 1 second to 7 days)

Process:
1. First request → Route to Target A
2. ALB sets cookie: AWSALB=target-a
3. Subsequent requests with cookie → Always route to Target A
4. After 1 hour, cookie expires → Load balancer assigns new target
```

**2. Application-Based** (ALB)
```
Your application sets cookie: SESSION_ID=abc123
ALB reads your cookie, routes based on it

Configuration:
- Cookie name: SESSION_ID
- ALB ensures same SESSION_ID → Same target

Benefit: Works with existing application session cookies
```

### When to Use Sticky Sessions

**Good use cases**:
- Legacy applications with local session storage
- Applications with server-side caching
- Temporary solution during migration to stateless architecture

**Bad use cases (prefer stateless)**:
- Modern cloud-native apps (use external session store)
- High availability requirements (target failure loses sessions)
- Optimal load distribution (stickiness can cause imbalance)

**Modern alternative**:
```
Instead of sticky sessions:
- Store sessions in: ElastiCache Redis, DynamoDB
- Any target can retrieve session from external store
- Benefits: Better HA, better load distribution, easier scaling
```

---

## Cross-Zone Load Balancing

**What it solves**: Uneven distribution across AZs.

**Without Cross-Zone**:
```
ALB (2 AZs):
- AZ-A: 1 target, receives 50% of traffic
- AZ-B: 3 targets, receive 50% of traffic (16.7% each)

Result: AZ-A target overloaded, AZ-B targets underutilized
```

**With Cross-Zone** (Enabled by default on ALB):
```
ALB distributes evenly across all 4 targets:
- AZ-A: 1 target, receives 25% of traffic
- AZ-B: 3 targets, receive 75% total (25% each)

Result: Even distribution regardless of AZ
```

**Cross-Zone Load Balancing by Load Balancer Type**:

| Load Balancer | Cross-Zone Default | Cost |
|---------------|-------------------|------|
| **ALB** | Enabled | Free |
| **NLB** | Disabled | $0.01/GB if enabled |
| **GWLB** | Disabled | Free if enabled |

**Recommendation**: Enable for even distribution, disable only if cost-sensitive (NLB high traffic volumes).

---

## SSL/TLS Termination

**What SSL/TLS termination is**: Load balancer decrypts HTTPS traffic, forwards HTTP to backends.

```
Client -(HTTPS)→ ALB (decrypts) -(HTTP)→ Backend

Benefits:
- Offload CPU-intensive crypto from backends
- Centralized certificate management (one place to update certs)
- Simpler backend configuration (no TLS setup)
```

### SSL/TLS Certificates

**Certificate sources**:

**1. AWS Certificate Manager (ACM)** (Recommended)
```
Benefits:
- Free SSL/TLS certificates
- Automatic renewal (no expiration issues)
- Easy integration with ALB/NLB
- Managed by AWS (no manual renewal)

Limitation: Can only use with AWS services (not exportable)
```

**2. Import Your Own Certificate**
```
Use cases:
- Organization policy requires specific CA
- Extended Validation (EV) certificates
- Wildcard certificates from third-party CA

Process: Upload certificate + private key to ACM or IAM
```

### SSL/TLS Policies

**What SSL policies are**: Define supported TLS versions and cipher suites.

**Policy selection trade-off**:
```
Strict policy (TLS 1.3 only):
- Maximum security
- Problem: Old browsers/clients can't connect

Lenient policy (TLS 1.0, 1.1, 1.2, 1.3):
- Maximum compatibility
- Problem: Vulnerable to older protocol attacks

Recommended (2026): TLS 1.2 and 1.3 only
- Modern browsers supported since 2015
- Good balance of security and compatibility
```

**AWS managed policies**:
- **ELBSecurityPolicy-TLS13-1-2-2021-06**: TLS 1.3 and 1.2 (recommended)
- **ELBSecurityPolicy-FS-1-2-Res-2020-10**: Forward secrecy, TLS 1.2+
- **ELBSecurityPolicy-2016-08**: Legacy policy (includes TLS 1.0, 1.1)

---

## Server Name Indication (SNI)

**What SNI solves**: Hosting multiple HTTPS sites (different domains) on same load balancer.

**The old problem** (before SNI):
```
Load balancer has one IP address
Client connects via HTTPS
Client must specify domain AFTER TLS handshake
But TLS handshake needs certificate BEFORE knowing domain
Result: Can't host multiple HTTPS domains on same IP
```

**SNI solution**:
```
Client connects to ALB
Client sends domain name (api.example.com) during TLS handshake (SNI extension)
ALB selects correct certificate based on domain
TLS handshake completes with right certificate
```

### Multi-Domain ALB Configuration

```
ALB with SNI:
├─ Listener: Port 443 (HTTPS)
├─ Certificate 1 (Default): api.example.com
├─ Certificate 2: www.example.com
├─ Certificate 3: admin.example.com
└─ Rules:
   ├─ Host: api.example.com → API Target Group
   ├─ Host: www.example.com → Web Target Group
   └─ Host: admin.example.com → Admin Target Group

Single ALB, single IP, multiple domains with different certificates
```

**Real-world use case**:
```
SaaS platform:
- customer1.saas.com (Certificate 1)
- customer2.saas.com (Certificate 2)
- ...
- customer100.saas.com (Certificate 100)

One ALB with 100 certificates via SNI
Cost: $16/month for ALB
vs. $1,600/month for 100 separate ALBs
```

**SNI support**:
- **ALB**: Full SNI support (multiple certificates)
- **NLB**: SNI support for TLS listeners
- **CLB**: No SNI support (legacy issue)

---

## Connection Draining (Deregistration Delay)

**What connection draining is**: Gracefully complete existing requests before removing a target.

**Without connection draining**:
```
1. Mark target for removal
2. Immediately stop all traffic → Close existing connections
3. Users: "500 error" mid-request

Result: Poor user experience
```

**With connection draining**:
```
1. Mark target for removal (enters "draining" state)
2. Stop NEW connections to target
3. Allow existing connections to complete
4. Wait up to "drain timeout" (default: 300 seconds)
5. After timeout, force-close remaining connections
6. Remove target

Result: Existing requests complete successfully
```

### Connection Draining Timeline

```
Target deregistration initiated:
00:00 - Target enters "draining" state
00:00 - Load balancer stops routing NEW requests
00:00 - Existing connections continue:
        - Active request (started at -0:30, ETA: +0:10) → Completes at 00:10
        - Long-polling connection (started -5:00) → Continues
05:00 - Drain timeout reached (300 seconds)
05:00 - Force close remaining connections
05:00 - Target fully deregistered

Requests after 00:00: Routed to other targets (no impact)
Requests before 00:00: Allowed to complete (up to 5 minutes)
```

### Drain Timeout Configuration

**Short timeout (30-60 seconds)**:
```
Use case: Short-lived requests (typical web pages, APIs)
- Most requests complete in <10 seconds
- 60-second timeout catches stragglers
- Faster deployments (instance replaced in ~1 minute)
```

**Long timeout (300+ seconds)**:
```
Use case: Long-running requests
- File uploads (multi-GB)
- Video streaming
- WebSocket connections
- Database migrations

Timeout: 5-10 minutes to allow completion
```

**Zero timeout** (Not recommended):
```
Immediate termination
Use case: Development only (don't care about dropped connections)
```

**Best practice**: Set based on your p99 request duration + buffer
- If p99 latency = 2 seconds → Timeout = 60 seconds (30× buffer)
- If p99 latency = 30 seconds → Timeout = 180 seconds

---

## Load Balancer Deletion Protection

**What it is**: Prevent accidental deletion of load balancer.

**Without protection**:
```
Engineer accidentally runs: aws elbv2 delete-load-balancer --load-balancer-arn <arn>
Result: Production load balancer deleted → Complete outage
```

**With protection enabled**:
```
Engineer runs delete command
AWS rejects: "Load balancer has deletion protection enabled"
Must first disable protection, then delete

Adds extra step = prevents accidents
```

**When to enable**:
- All production load balancers
- Critical infrastructure
- Load balancers managing revenue-generating traffic

**When optional**:
- Development environments
- Temporary load balancers
- Non-critical testing

---

## Load Balancer Access Logs

**What access logs are**: Detailed logs of every request processed by load balancer.

**Log contents**:
```
Each log entry includes:
- Timestamp (2026-02-06T12:34:56.789Z)
- Client IP (203.0.113.5)
- Backend IP (10.0.1.50)
- Request processing time (0.123 seconds)
- Backend processing time (0.100 seconds)
- Response status (200, 404, 500)
- User agent (browser/device)
- Request URL (https://example.com/api/users)
- SSL cipher (TLS_AES_128_GCM_SHA256)
- TLS version (TLS1.3)
```

**Storage**:
```
Access logs → S3 bucket
Path structure: s3://my-logs/AWSLogs/{account-id}/elasticloadbalancing/{region}/{year}/{month}/{day}/

Cost:
- ALB logging: Free (no charge for log generation)
- S3 storage: $0.023/GB/month
- Example: 1M requests/day, 1 KB/request = 30 GB/month = $0.69/month
```

### Use Cases for Access Logs

**1. Security Analysis**
```
Detect attacks:
- High request rate from single IP (DDoS attempt)
- SQL injection patterns in URLs
- Brute force login attempts
- Suspicious user agents (bots, scrapers)

Tool: Amazon Athena to query logs, detect patterns
```

**2. Debugging**
```
Customer report: "I got error at 2 PM"
Access logs: Find exact request, see:
- Which backend handled it
- Response status code
- Processing time
- Error details

Faster troubleshooting vs guessing
```

**3. Analytics**
```
Business insights:
- Most popular pages (by request count)
- Geographic distribution (by IP geolocation)
- Device breakdown (mobile vs desktop, via User-Agent)
- API usage patterns

Tool: Athena or Redshift for analysis
```

**4. Compliance**
```
Regulations require:
- Log all access to sensitive data
- Retain logs for 7 years
- Audit trail of who accessed what

Load balancer access logs = compliance requirement met
```

---

## Advanced Load Balancing Patterns

### 1. Blue-Green Deployments

**Concept**: Two identical environments (blue = current, green = new), switch traffic instantly.

```
Setup:
- Blue Target Group: 10 instances running v1.0
- Green Target Group: 10 instances running v2.0
- ALB: 100% traffic → Blue

Deployment:
1. Test Green environment (internal testing)
2. Update ALB listener: 100% traffic → Green
3. Traffic switches instantly (seconds)
4. Monitor metrics
5. If issues: Switch back to Blue (instant rollback)

Benefit: Zero-downtime deployments, instant rollback
```

### 2. Canary Deployments

**Concept**: Gradually shift traffic from old to new version.

```
Setup:
- Prod Target Group (v1.0): Weight 90
- Canary Target Group (v2.0): Weight 10

ALB routes:
- 90% traffic → v1.0
- 10% traffic → v2.0 (canary)

Monitor canary:
- Error rate, latency, business metrics
- If good: Shift to 50/50, then 0/100
- If bad: Shift back to 100/0

Benefit: Limit blast radius of bad deployments
```

### 3. Weighted Target Groups

**Concept**: Control traffic split between target groups.

```
Use case: A/B testing new feature
- Control group (no feature): 50% traffic
- Treatment group (new feature): 50% traffic

Measure: Conversion rates, engagement
Data-driven decisions on feature launch
```

### 4. Multi-Region Failover

**Concept**: Route 53 + ALBs in multiple regions for disaster recovery.

```
Route 53 Health Check → ALB in us-east-1
If health check fails:
- Route 53 switches DNS to ALB in us-west-2
- Traffic now goes to DR region
- Automatic failover, no manual intervention

RTO: 5-10 minutes (DNS TTL + propagation)
```

---

## Load Balancer Pricing (2026)

### Cost Structure

**All load balancers**:
```
Base cost: $16/month ($0.0225/hour)
+ LCU cost (Load Balancer Capacity Units)

LCU measured by:
- New connections per second
- Active connections per minute
- Bandwidth (GB processed)
- Rule evaluations (ALB only)

You pay for highest dimension
```

### ALB Pricing Example

```
Application Load Balancer:
- Base: $16/month
- LCU pricing: $0.008/LCU-hour

Usage:
- 25 new connections/second (dimension 1)
- 3,000 active connections (dimension 2)
- 10 GB/hour processed (dimension 3)
- 2,000 rule evaluations/second (dimension 4)

LCU calculation:
- Dimension 1: 25 / 25 = 1 LCU
- Dimension 2: 3,000 / 3,000 = 1 LCU
- Dimension 3: 10 / 1 = 10 LCUs ← Highest
- Dimension 4: 2,000 / 1,000 = 2 LCUs

Billable: 10 LCUs/hour
Cost: $16 + (10 × $0.008 × 730 hours) = $16 + $58.40 = $74.40/month
```

### NLB Pricing Example

```
Network Load Balancer:
- Base: $16/month
- LCU pricing: $0.006/LCU-hour (25% cheaper than ALB)

Usage:
- 800 new connections/second
- 100,000 active connections
- 50 GB/hour processed

LCU calculation:
- New connections: 800 / 800 = 1 LCU
- Active connections: 100,000 / 100,000 = 1 LCU
- Bandwidth: 50 / 1 = 50 LCUs ← Highest

Billable: 50 LCUs/hour
Cost: $16 + (50 × $0.006 × 730) = $16 + $219 = $235/month
```

### Cost Optimization Strategies

**1. Consolidate Load Balancers**
```
Before: 5 ALBs (one per microservice) = 5 × $16 = $80/month base
After: 1 ALB with path-based routing = $16/month base
Savings: $64/month (80% reduction in base cost)
```

**2. Use NLB for Non-HTTP**
```
TCP/UDP workload:
- NLB LCU: $0.006/hour
- ALB LCU: $0.008/hour
- Savings: 25% per LCU (if you don't need HTTP features)
```

**3. Optimize Rule Evaluations (ALB)**
```
Rule evaluations count as LCU dimension
- 100 complex rules per request = expensive
- Simplify rules, reduce evaluation count
- Use path-based over query-string when possible
```

**4. Connection Reuse**
```
Enable keep-alive (HTTP persistent connections)
- Reduces new connections per second
- Lower LCU consumption
- Better performance
```

---

## ELB Best Practices Summary

**Architecture**:
1. **Multi-AZ**: Always deploy in ≥2 AZs (high availability)
2. **Cross-Zone Load Balancing**: Enable for even distribution
3. **Deletion Protection**: Enable on production load balancers

**Health Checks**:
1. **Deep health checks**: Check application dependencies, not just HTTP 200
2. **Appropriate thresholds**: 2-3 failures for prod, faster for dev
3. **Separate health endpoint**: Dedicated port/path for health (optional)

**Security**:
1. **TLS 1.2+**: Disable TLS 1.0/1.1 (vulnerable)
2. **ACM certificates**: Use AWS Certificate Manager (free, auto-renewal)
3. **WAF integration**: Attach AWS WAF for application-layer protection (DDoS, SQL injection)

**Monitoring**:
1. **Enable access logs**: S3 storage for debugging, compliance, analytics
2. **CloudWatch metrics**: Monitor request count, latency, error rates
3. **CloudWatch Alarms**: Alert on unhealthy targets, high latency, errors

**Performance**:
1. **Choose right type**: ALB for HTTP, NLB for TCP/UDP/ultra-low-latency
2. **Connection draining**: Set based on request duration (p99 + buffer)
3. **Sticky sessions**: Avoid if possible (use external session store)

**Cost**:
1. **Consolidate**: Use path-based routing instead of multiple ALBs
2. **Right-size**: Monitor LCU usage, optimize rule evaluations
3. **Reserved capacity**: For predictable, high-traffic loads (contact AWS)

---

## Choosing the Right Load Balancer

| Requirement | Load Balancer | Why |
|-------------|---------------|-----|
| HTTP/HTTPS web app | ALB | Layer 7 routing, WebSocket, HTTP/2 |
| Microservices | ALB | Path-based routing, host-based routing |
| Ultra-low latency | NLB | Microsecond latency, millions of req/sec |
| Static IP required | NLB | Elastic IP support |
| TCP/UDP protocol | NLB | Layer 4 support |
| Preserve source IP | NLB | Passthrough mode |
| Security appliances | GWLB | Firewall, IDS/IPS, DPI |
| WebSocket | ALB or NLB | Both support, ALB easier for HTTP-based |
| gRPC | ALB | Native gRPC routing |
| Lambda targets | ALB | Serverless integration |

**Default recommendation (2026)**: Start with ALB for most web applications. Consider NLB only if you need Layer 4 features or ultra-low latency.

---

**Elastic Load Balancing is the foundation of highly available, scalable AWS architectures. Understanding when to use ALB vs NLB, how to configure health checks, and how to leverage advanced features like SNI and connection draining is critical for production systems.**