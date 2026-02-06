# Distributed Coordination

## Table of Contents
1. [Introduction](#introduction)
2. [Service Discovery](#service-discovery)
3. [Distributed Configuration Management](#distributed-configuration-management)
4. [Leader Election Services](#leader-election-services)
5. [Distributed Semaphores and Barriers](#distributed-semaphores-and-barriers)
6. [Coordination Patterns](#coordination-patterns)

---

## Introduction

**Distributed coordination** is the set of mechanisms that enable multiple independent processes or services to work together in a coordinated manner across a network. In distributed systems, coordination is essential for tasks like discovering services, electing leaders, managing shared configuration, and synchronizing actions.

### Why Distributed Coordination?

In a distributed system, you face challenges that don't exist in single-machine systems:

```
Single Machine (Simple):              Distributed System (Complex):

    [Process A]                       [Service A] ─┐
        |                                          ├─ Network
    [Process B]                       [Service B] ─┤  (can fail)
        |                                          ├─
    [Process C]                       [Service C] ─┘

All in same memory                 Each has own memory
Direct communication               Network communication
Single point of failure            Multiple points of failure
Shared state easy                  Shared state hard
```

**Key coordination challenges**:
1. **Service discovery**: How does Service A find Service B's address?
2. **Configuration**: How do all services get the same config changes?
3. **Leadership**: Who coordinates actions when multiple services exist?
4. **Synchronization**: How do services wait for each other?
5. **Membership**: Which services are currently alive?

**Layman explanation**: Think of distributed coordination like managing a large orchestra. Each musician (service) needs to know where to sit (discovery), what version of the music to play (configuration), who's conducting (leader election), and when to start/stop together (synchronization). Without coordination, you get chaos instead of harmony.

### The CAP Theorem Context

Distributed coordination systems must operate under the constraints of the CAP theorem:

```
CAP Theorem Triangle:

         Consistency
              /\
             /  \
            /    \
           /  CP  \
          / (e.g., \
         /  etcd,   \
        / ZooKeeper) \
       /──────────────\
   Partition        Available
   Tolerance
       \──────────────/
        \    AP      /
         \ (e.g.,  /
          \ Consul)/
           \     /
            \   /
             \ /

You can only pick 2 of 3!
```

**Layman explanation**: It's like having a group project where you can't have all three of: (1) everyone has the exact same information, (2) everyone can always participate, and (3) the group still works when communication breaks down. Most coordination systems choose consistency + partition tolerance, meaning they prefer accurate data over always being available.

---

## Service Discovery

**Service discovery** is the mechanism by which services find and communicate with each other in a dynamic distributed system where service locations change frequently.

### The Problem Service Discovery Solves

```
Without Service Discovery:                With Service Discovery:

[Service A]                               [Service A]
    |                                         |
    needs to call                         1. Query registry
    "user-service"                           "where is user-service?"
    |                                         |
    hardcoded:                                v
    http://192.168.1.10:8080          [Service Registry]
    |                                         |
    ✗ What if it moves?                  2. Returns current location
    ✗ What if it scales to 10 instances?     |
    ✗ What if it's down?                     v
                                         [Service A]
                                             |
                                         3. Call correct instance
                                             |
                                             v
                                         [User Service]
                                          (current location)
```

**Problems with hardcoding**:
- Services change IP addresses when they restart
- Services scale horizontally (multiple instances)
- Services may be deployed to different hosts
- Failed services need to be avoided

**Layman explanation**: Service discovery is like a phone book that automatically updates. Instead of memorizing your friend's phone number, you look them up each time you want to call. If they change numbers, the phone book updates automatically.

### Client-Side Discovery

In client-side discovery, the client is responsible for determining available service instances and load balancing requests.

```
Client-Side Discovery Flow:

1. Client queries registry
   [Client] ──query──> [Service Registry]
                            |
                       Returns list:
                       [instance-1: 192.168.1.10:8080]
                       [instance-2: 192.168.1.11:8080]
                       [instance-3: 192.168.1.12:8080]
                            |
                            v
2. Client chooses instance (load balancing)
   [Client] ──decides to call──> instance-2
                            |
3. Direct call to service
   [Client] ──HTTP request──> [Service instance-2]
```

**Characteristics**:
- Client knows about all available instances
- Client implements load balancing logic
- Client makes direct connections to services
- No intermediate proxy

**Load balancing strategies** (client implements):
```
Round Robin:
Request 1 → Instance 1
Request 2 → Instance 2
Request 3 → Instance 3
Request 4 → Instance 1 (cycle repeats)

Random:
Request 1 → Instance 2 (random pick)
Request 2 → Instance 1 (random pick)
Request 3 → Instance 3 (random pick)

Least Connections:
Request → Instance with fewest active connections
```

**Advantages**:
- ✓ Fewer network hops (direct connection)
- ✓ Client controls load balancing strategy
- ✓ Better performance (no proxy overhead)
- ✓ Client can implement smart routing (sticky sessions, affinity)

**Disadvantages**:
- ✗ Client must implement discovery logic
- ✗ Client must implement load balancing
- ✗ Each client language needs implementation
- ✗ More complexity in client applications

**Examples**:
- Netflix Eureka with Ribbon (client-side load balancing)
- Consul with client libraries
- Custom implementations with etcd/ZooKeeper

**When to use**:
- Microservices within same network
- Performance-critical applications
- When you control all clients
- When client-side intelligence is needed

### Server-Side Discovery

In server-side discovery, the client makes requests to a load balancer/proxy, which queries the registry and forwards requests.

```
Server-Side Discovery Flow:

1. Client calls load balancer (fixed address)
   [Client] ──request──> [Load Balancer]
                             |
2. Load balancer queries registry
                         [LB] ──query──> [Service Registry]
                                              |
                                         Returns:
                                         [instance-1: healthy]
                                         [instance-2: healthy]
                                         [instance-3: unhealthy]
                                              |
3. Load balancer selects healthy instance
                         [LB] ──decides──> instance-2
                             |
4. Load balancer forwards request
   [Load Balancer] ──forward──> [Service instance-2]
                             |
5. Response flows back through load balancer
   [Client] <──response──< [Load Balancer] <── [Service]
```

**Characteristics**:
- Client only knows load balancer address
- Load balancer handles discovery and routing
- Centralized load balancing logic
- Additional network hop

**Load balancer types**:
```
Hardware Load Balancers:
- F5, Citrix NetScaler
- Expensive, high performance
- Typically at network edge

Software Load Balancers:
- HAProxy, Nginx, Envoy
- Flexible, programmable
- Can run anywhere

Cloud Load Balancers:
- AWS ELB/ALB, Azure Load Balancer
- Managed service
- Integrated with cloud infrastructure
```

**Advantages**:
- ✓ Simple client implementation
- ✓ Centralized load balancing logic
- ✓ Language-agnostic (any client works)
- ✓ Easy to change load balancing strategy
- ✓ Can add features (SSL termination, caching)

**Disadvantages**:
- ✗ Additional network hop (latency)
- ✗ Load balancer is single point of failure
- ✗ Load balancer can be bottleneck
- ✗ More infrastructure to manage

**Examples**:
- Kubernetes Service with kube-proxy
- AWS Elastic Load Balancer (ELB)
- Nginx with dynamic upstreams
- Envoy proxy with service mesh

**When to use**:
- Public-facing APIs
- Multiple client types/languages
- Centralized control preferred
- When infrastructure supports it

### Service Registry

A **service registry** is a database of available service instances with their locations and metadata.

#### Service Registry Architecture

```
Service Registry Components:

   Service Instances           Registry           Clients

[Service A-1] ──register──┐
                          ├──> [Registry DB]
[Service A-2] ──register──┤         |
                          │    [instance-1: host, port, health]
[Service A-3] ──register──┘    [instance-2: host, port, health]
                                [instance-3: host, port, health]
        |                           |
    heartbeat                       |
    every 10s                   [Query API]
        |                           |
        └────────────────────────> query ───> [Client]
                                              discovers
                                              all instances
```

**Core operations**:

**1. Registration**: Service registers itself on startup
```
POST /register
{
  "service": "user-service",
  "instance_id": "user-service-abc123",
  "host": "192.168.1.10",
  "port": 8080,
  "metadata": {
    "version": "1.2.0",
    "region": "us-east-1"
  }
}
```

**2. Deregistration**: Service removes itself on shutdown
```
DELETE /deregister/user-service/abc123
```

**3. Heartbeat/Health check**: Prove service is alive
```
PUT /heartbeat/user-service/abc123
(sent every 10 seconds)
```

**4. Discovery**: Query for available instances
```
GET /discover/user-service
→ Returns list of healthy instances
```

#### Popular Service Registry Implementations

**Consul by HashiCorp**:
```
Consul Features:

[Service Mesh Support]
     |
     ├── Service Registration/Discovery ✓
     ├── Health Checking ✓
     ├── Key/Value Store ✓
     ├── Multi-datacenter ✓
     └── DNS Interface ✓

Architecture:

   [Consul Server 1] ──┐
   [Consul Server 2] ──┼── Raft consensus
   [Consul Server 3] ──┘
          |
   Consul Clients
          |
   ├── [Service A]
   ├── [Service B]
   └── [Service C]
```

**Consul characteristics**:
- Multi-datacenter support out of the box
- Built-in health checking
- DNS interface (services discoverable via DNS)
- Service mesh capabilities (Consul Connect)
- Key/value store for configuration
- Strong consistency (CP in CAP)

**etcd (by CoreOS/Cloud Native Computing Foundation)**:
```
etcd Architecture:

[etcd node 1] ──┐
[etcd node 2] ──┼── Raft consensus cluster
[etcd node 3] ──┘
      |
   HTTP/gRPC API
      |
   [Kubernetes] ← primary user
   [Custom apps]
   [Service discovery]

Key features:
- Distributed key-value store
- Strong consistency
- Watch mechanism (real-time updates)
- TTL for keys (auto-expiration)
- Atomic operations
```

**etcd characteristics**:
- Kubernetes' backing store
- Simple key-value model
- Strong consistency guarantees
- Watch API for real-time updates
- Not specifically built for service discovery (but commonly used)

**ZooKeeper (by Apache)**:
```
ZooKeeper Ensemble:

[ZK Server 1] ──┐
[ZK Server 2] ──┼── ZAB protocol (consensus)
[ZK Server 3] ──┘
      |
   Hierarchical namespace
      |
   /services
      ├── /user-service
      │   ├── instance-1
      │   ├── instance-2
      │   └── instance-3
      └── /order-service
          ├── instance-1
          └── instance-2
```

**ZooKeeper characteristics**:
- Oldest and most mature
- Hierarchical namespace (like filesystem)
- Strong consistency
- Watches for change notifications
- Ephemeral nodes (auto-delete when client disconnects)
- Java-based, mature ecosystem

**Comparison**:
```
Feature           Consul    etcd       ZooKeeper
──────────────────────────────────────────────────────
Service discovery Native    Manual     Manual
Health checking   Built-in  DIY        DIY
DNS interface     Yes       No         No
Multi-DC          Native    Manual     Manual
HTTP API          Yes       Yes        No (Java)
Consistency       CP        CP         CP
Maturity          Medium    Medium     High
Learning curve    Medium    Low        High
```

### DNS-Based Discovery

**DNS-based discovery** uses the Domain Name System for service discovery.

```
DNS Service Discovery Flow:

1. Service registers DNS record
   [Service instance] ──register──> [DNS Server]
                                         |
                                    Creates records:
                                    user-service.local → 192.168.1.10
                                    user-service.local → 192.168.1.11
                                    user-service.local → 192.168.1.12

2. Client queries DNS
   [Client] ──DNS lookup: user-service.local──> [DNS Server]
                                                      |
3. DNS returns IP addresses (round-robin)
   [Client] <──192.168.1.10, 192.168.1.11, 192.168.1.12──┘

4. Client connects to one of the IPs
   [Client] ──connect──> [192.168.1.10]
```

**DNS record types for service discovery**:

**A record** (Address):
```
user-service.local.  IN  A  192.168.1.10
user-service.local.  IN  A  192.168.1.11
user-service.local.  IN  A  192.168.1.12

Client gets multiple IPs, typically round-robin
```

**SRV record** (Service):
```
_http._tcp.user-service.local. IN SRV 10 60 8080 instance1.local.
_http._tcp.user-service.local. IN SRV 10 40 8080 instance2.local.

Contains: priority, weight, port, target
More metadata than A records
```

**Advantages**:
- ✓ Universal (every platform supports DNS)
- ✓ Simple client implementation
- ✓ Built-in caching
- ✓ No special libraries needed
- ✓ Works across network boundaries

**Disadvantages**:
- ✗ DNS caching can be stale
- ✗ No health checking (DNS returns all records)
- ✗ TTL (time-to-live) not always respected by clients
- ✗ Limited metadata
- ✗ Slow updates (DNS propagation)

**Examples**:
- Consul (provides DNS interface)
- Kubernetes DNS (CoreDNS)
- AWS Route 53 with health checks
- SkyDNS

**When to use**:
- Simple deployments
- Cross-platform compatibility needed
- When DNS infrastructure already exists
- For external services (not just internal)

### Health Checking and Deregistration

**Health checking** ensures only healthy service instances are discoverable. **Deregistration** removes unhealthy or shutdown instances.

#### Health Check Types

**1. Heartbeat (passive checks)**:
```
Service sends periodic heartbeats:

t=0s  : [Service] ──heartbeat──> [Registry] (healthy ✓)
t=10s : [Service] ──heartbeat──> [Registry] (healthy ✓)
t=20s : [Service] ──heartbeat──> [Registry] (healthy ✓)
t=30s : [Service] X (crashed)
t=40s : [Registry checks: no heartbeat received]
t=40s : [Registry marks service unhealthy ✗]
t=40s : [Registry deregisters service]

If no heartbeat for N seconds → mark unhealthy
```

**2. Active health checks (registry probes service)**:
```
Registry actively checks service:

t=0s  : [Registry] ──HTTP GET /health──> [Service] ──200 OK──> ✓
t=10s : [Registry] ──HTTP GET /health──> [Service] ──200 OK──> ✓
t=20s : [Registry] ──HTTP GET /health──> [Service] X timeout ✗
t=30s : [Registry] ──HTTP GET /health──> [Service] X timeout ✗
t=30s : [Registry marks unhealthy and deregisters]
```

**3. Hybrid approach**:
```
Combine both:

Service sends heartbeats (lightweight, frequent)
    +
Registry performs deep health checks (heavy, infrequent)

Example: Heartbeat every 5s, deep check every 30s
```

#### Health Check Endpoints

**Simple health check**:
```
GET /health
Response: 200 OK
{
  "status": "UP"
}

Service is alive
```

**Detailed health check**:
```
GET /health
Response: 200 OK
{
  "status": "UP",
  "checks": {
    "database": "UP",
    "cache": "UP",
    "disk_space": "UP"
  },
  "details": {
    "uptime": "3600s",
    "version": "1.2.0"
  }
}

Service and dependencies are healthy
```

**Degraded health**:
```
GET /health
Response: 503 Service Unavailable
{
  "status": "DOWN",
  "checks": {
    "database": "DOWN",
    "cache": "UP"
  }
}

Service has issues, should be deregistered
```

#### Deregistration Strategies

**1. Graceful deregistration (clean shutdown)**:
```
Shutdown sequence:

1. Service receives shutdown signal (SIGTERM)
2. Service stops accepting new requests
3. Service finishes processing ongoing requests
4. Service deregisters from registry
5. Service confirms shutdown → Registry removes entry
6. Service exits
```

**2. Ephemeral nodes (auto-deregistration)**:
```
ZooKeeper/etcd pattern:

Service connects → Creates ephemeral node
   [Service] ──create /services/user/instance-1 (ephemeral)

If connection lost:
   [Service] X (crashes or network partition)
        |
   Connection dropped
        |
   [Registry] automatically deletes ephemeral node
        |
   Instance immediately deregistered
```

**3. TTL-based expiration**:
```
etcd lease pattern:

Service registers with TTL=30s
   [Service] ──register with lease (30s TTL)
        |
   Must renew lease every 30s
        |
   t=0s : register ✓
   t=15s: renew lease ✓
   t=30s: renew lease ✓
   t=45s: [Service crashes, no renewal]
   t=60s: [Registry expires lease, deregisters]
```

**Layman explanation**: Health checking is like a teacher taking attendance. If students (services) don't respond when called (health check), they're marked absent (unhealthy). After enough missed checks, they're removed from the class roster (deregistered). Students can also tell the teacher when they're leaving early (graceful deregistration).

---

## Distributed Configuration Management

**Distributed configuration management** enables multiple services across a distributed system to share, update, and maintain consistent configuration settings.

### The Configuration Problem

```
Without Centralized Config:         With Centralized Config:

Each service has local config:     All services use central config:

[Service A]                        [Service A] ─┐
  config.yaml                                   │
  - db_host: 10.0.1.5                          │
  - timeout: 30s                                ├──> [Config Server]
                                                │      db_host: 10.0.1.5
[Service B]                        [Service B] ─┤      timeout: 30s
  config.yaml                                   │      feature_x: true
  - db_host: 10.0.1.5 (duplicate!)             │
  - timeout: 30s                                │
                                   [Service C] ─┘
[Service C]
  config.yaml
  - db_host: 10.0.1.5 (again!)

Problem: Database moves to 10.0.1.20    Solution: Update once in config server
Must update ALL services manually       All services get new value automatically
Error-prone, slow, inconsistent        Fast, consistent, auditable
```

**Problems with local configuration**:
- Duplicate configuration across services
- Inconsistent values (typos, outdated values)
- Hard to update (requires redeployment)
- No audit trail
- Environment-specific differences (dev vs prod)
- Secrets management challenges

**Layman explanation**: Think of configuration like phone numbers for pizza delivery. Without centralized config, each family member has their own copy of the number written down. If the pizzeria changes numbers, you have to tell everyone individually (and hope nobody forgot to update). With centralized config, everyone checks the same phone book—update it once, everyone sees the new number.

### Centralized Configuration

**Centralized configuration** stores all configuration in a single source of truth that services read from.

#### Centralized Configuration Architecture

```
Configuration Flow:

1. Config stored centrally
   [Config Repository]
        |
        ├── application.yaml
        ├── database.yaml
        ├── features.yaml
        └── secrets (encrypted)

2. Services fetch config on startup
   [Service A] ──fetch──> [Config Server] ──reads──> [Repository]
   [Service B] ──fetch──> [Config Server]
   [Service C] ──fetch──>

3. Services can also watch for changes
   [Config Server] ──notify──> [Service A] (config changed!)
                   ──notify──> [Service B]
                   ──notify──> [Service C]

4. Services reload config dynamically
   [Services] ──apply new config──> (no restart needed)
```

**Configuration storage backends**:

**Git repository**:
```
Repository structure:
/configs
  ├── production
  │   ├── service-a.yaml
  │   ├── service-b.yaml
  │   └── common.yaml
  ├── staging
  │   ├── service-a.yaml
  │   └── common.yaml
  └── development
      └── common.yaml

Benefits:
✓ Version control built-in
✓ Audit trail (commit history)
✓ Code review for config changes
✓ Rollback capability
✓ Familiar workflow for developers
```

**Key-value stores** (etcd, Consul):
```
Hierarchical structure:
/config
  ├── /production
  │   ├── /database
  │   │   ├── host = "db.prod.internal"
  │   │   └── port = 5432
  │   └── /features
  │       └── new_ui = true
  └── /staging
      └── /database
          └── host = "db.staging.internal"

Benefits:
✓ Real-time updates
✓ Watch mechanism
✓ High availability
✓ Distributed consensus
```

**Database**:
```
Config table:
┌──────────────┬─────────┬───────────────┬─────────┐
│ service      │ env     │ key           │ value   │
├──────────────┼─────────┼───────────────┼─────────┤
│ user-service │ prod    │ db.host       │ 10.0.1.5│
│ user-service │ prod    │ db.timeout    │ 30      │
│ user-service │ staging │ db.host       │ 10.0.2.5│
└──────────────┴─────────┴───────────────┴─────────┘

Benefits:
✓ SQL queries for config
✓ Relational integrity
✓ Familiar technology
✗ Not designed for distributed reads
✗ Slower updates
```

#### Configuration Hierarchy

**Layered configuration** (precedence from highest to lowest):

```
Configuration Layers:

[Service-specific overrides]  ← Highest precedence
         |
         v
[Environment-specific]
  (production, staging, dev)
         |
         v
[Application defaults]
         |
         v
[Global defaults]              ← Lowest precedence

Example:
Global:     timeout = 30s
App:        timeout = 60s       (overrides global)
Production: timeout = 120s      (overrides app)
Service-A:  timeout = 10s       (overrides everything)

Result for Service-A in production: timeout = 10s
```

**Profile-based configuration**:
```
Application can run with different profiles:

Development profile:
  - database: localhost
  - debug_logging: true
  - cache: disabled

Production profile:
  - database: db.prod.internal
  - debug_logging: false
  - cache: enabled
  - monitoring: enabled

Services load config based on active profile
```

### Configuration Versioning

**Configuration versioning** tracks changes to configuration over time.

#### Version Control with Git

```
Git-based versioning:

Commit history:

commit abc123 (HEAD -> main)
Author: Alice
Date: 2026-02-06
    Increase database timeout to 60s

    database.timeout: 30s → 60s

commit def456
Author: Bob
Date: 2026-02-05
    Enable feature flag for new UI

    features.new_ui: false → true

commit ghi789
Author: Alice
Date: 2026-02-04
    Update database host for migration

    database.host: 10.0.1.5 → 10.0.1.20
```

**Benefits of Git versioning**:
- Complete change history
- Who changed what, when, and why
- Ability to diff versions
- Easy rollback (revert commit)
- Branch for testing config changes
- Code review for configuration

#### Semantic Versioning

```
Configuration versions:

v1.2.3
│ │ │
│ │ └─ Patch: backward-compatible bug fixes
│ └─── Minor: backward-compatible new features
└───── Major: breaking changes

Examples:
v1.0.0 → v1.0.1  (fix typo in config key)
v1.0.1 → v1.1.0  (add new optional parameter)
v1.1.0 → v2.0.0  (rename config keys, breaking change)
```

**Version compatibility**:
```
Service version compatibility matrix:

Config Version    Compatible Service Versions
─────────────────────────────────────────────
v1.x.x            v1.0.0 - v1.9.9
v2.x.x            v2.0.0+
v3.x.x            v3.0.0+

Service v1.5.0 can use config v1.x.x ✓
Service v1.5.0 cannot use config v2.x.x ✗
```

#### Configuration Rollback

```
Rollback scenario:

t=0:  Deploy config v2.0.0 (breaking change)
      [Services] ──apply config v2.0.0

t=5:  Services start failing ✗
      Error: unknown config key "new_setting"

t=6:  Rollback decision made
      [Admin] ──rollback to v1.9.0

t=7:  Services receive v1.9.0
      [Services] ──apply config v1.9.0

t=8:  Services healthy again ✓

Git rollback:
git revert abc123  (creates new commit that undoes abc123)
or
git reset --hard def456  (reset to previous commit)
```

### Dynamic Configuration Updates

**Dynamic configuration** allows changing config without restarting services.

#### Push vs Pull Models

**Pull model** (services poll for changes):
```
Services periodically check for updates:

t=0s  : [Service] ──fetch config──> [Config Server]
        (config version: v1.0.0)

t=30s : [Service] ──fetch config──> [Config Server]
        (config version: v1.0.0) ← unchanged

t=60s : [Service] ──fetch config──> [Config Server]
        (config version: v1.1.0) ← NEW VERSION!
        [Service] ──applies new config

Polling interval: 30-60 seconds typical

Pros:
✓ Simple implementation
✓ Service controls update timing
✓ No persistent connection needed

Cons:
✗ Delay before updates applied
✗ Unnecessary network calls
✗ All services poll (scalability)
```

**Push model** (config server notifies services):
```
Config server pushes changes:

[Config Server] maintains connections with services
        |
        ├── [Service A] (websocket connection)
        ├── [Service B] (websocket connection)
        └── [Service C] (websocket connection)

Config change occurs:
[Admin] ──update config──> [Config Server]
                                |
                           Immediately notify all:
                                |
                    ┌───────────┼───────────┐
                    v           v           v
              [Service A]  [Service B]  [Service C]
                    |           |           |
              Apply update  Apply update  Apply update
              (instant)     (instant)     (instant)

Pros:
✓ Instant updates
✓ No polling overhead
✓ Efficient

Cons:
✗ Requires persistent connections
✗ More complex implementation
✗ Connection management overhead
```

**Watch mechanism** (etcd/Consul/ZooKeeper):
```
Watch-based updates:

1. Service establishes watch on config key
   [Service] ──watch /config/database/timeout──> [etcd]

2. Service receives current value
   [etcd] ──returns: 30s──> [Service]

3. Config changes
   [Admin] ──set /config/database/timeout = 60s──> [etcd]

4. etcd notifies watching service
   [etcd] ──notification: timeout changed to 60s──> [Service]

5. Service applies new config
   [Service] ──updates timeout to 60s

Pros:
✓ Real-time updates (milliseconds)
✓ No polling needed
✓ Event-driven
✓ Multiple services can watch same key

Cons:
✗ Requires connection maintenance
✗ Watch must be re-established on disconnect
```

#### Gradual Rollout (Canary Configuration)

```
Phased configuration rollout:

Phase 1: Update 10% of instances
   [10% of services] ──new config (v2.0)
   [90% of services] ──old config (v1.0)
   Monitor for errors...

Phase 2: If healthy, update 50%
   [50% of services] ──new config (v2.0)
   [50% of services] ──old config (v1.0)
   Monitor for errors...

Phase 3: If still healthy, update all
   [100% of services] ──new config (v2.0)
   Done ✓

If errors at any phase:
   Rollback all to v1.0
   Investigate issue
```

**Implementation strategies**:
```
Instance-based routing:
- Tag instances as canary or stable
- Config server returns different config based on tag

Percentage-based:
- Use instance ID hash to determine which get new config
- Example: if hash(instance_id) % 100 < 10, use new config

Feature flags as config:
- Config contains feature flags
- Gradually increase percentage of users with flag enabled
```

### Configuration Consistency

Ensuring all services see consistent configuration is critical.

#### Consistency Models

**Strong consistency** (linearizable):
```
All services see config changes in same order:

t=0: [Admin] ──sets timeout=60s──> [Config Store]
t=1: [Config Store] ──replicates to all nodes
t=2: [Service A] ──reads timeout── gets 60s ✓
t=3: [Service B] ──reads timeout── gets 60s ✓
t=4: [Service C] ──reads timeout── gets 60s ✓

All services see same value
No service sees old value (30s) after update
```

**Eventual consistency**:
```
Services may temporarily see different values:

t=0: [Admin] ──sets timeout=60s──> [Config Node 1]
t=1: [Service A] ──reads from Node 1── gets 60s ✓
t=1: [Service B] ──reads from Node 2── gets 30s ⚠️ (old value)
t=2: [Config replicates Node 1 → Node 2]
t=3: [Service B] ──reads from Node 2── gets 60s ✓

Eventually all services converge to same value
Temporary inconsistency acceptable
```

**When each is appropriate**:
```
Strong consistency needed:
- Database connection settings
- Security policies
- Rate limits
- Critical business rules

Eventual consistency acceptable:
- UI themes
- Feature flags (non-critical)
- Logging levels
- Cache TTLs
```

#### Configuration Validation

**Schema validation** (prevent invalid config):
```
Config schema definition:
{
  "type": "object",
  "properties": {
    "database": {
      "type": "object",
      "properties": {
        "host": {"type": "string", "pattern": "^[a-z0-9.-]+$"},
        "port": {"type": "integer", "minimum": 1, "maximum": 65535},
        "timeout": {"type": "integer", "minimum": 1, "maximum": 300}
      },
      "required": ["host", "port"]
    }
  }
}

Validation flow:
[Admin] ──submits config──> [Config Server]
                                |
                           Validate against schema
                                |
         ┌──────────────────────┴──────────────────────┐
         v                                             v
    Valid config                                  Invalid config
         |                                             |
    Accept & propagate                           Reject with errors
         |                                             |
   [Services updated]                      [Admin notified of errors]
```

**Dry-run validation**:
```
Test configuration before applying:

1. [Admin] ──submit config (dry-run mode)──> [Config Server]
2. [Config Server] ──validate schema
3. [Config Server] ──test apply to canary instance
4. [Canary Instance] ──loads config, validates internally
5. [Canary Instance] ──reports: valid ✓ or errors ✗
6. If valid: [Admin] ──apply to production
   If errors: [Admin] ──fix issues, retry
```

**Layman explanation**: Dynamic configuration is like adjusting the temperature in a smart home. Instead of going to each room's thermostat (restarting services), you change it once on your phone app (config server), and all rooms update automatically. You can also test the temperature in one room first (canary) before applying it everywhere.

---

## Leader Election Services

**Leader election** is the process of designating one node among many as the coordinator or decision-maker. This is essential in distributed systems to avoid conflicts and coordinate actions.

### Why Leader Election?

```
Without Leader (chaos):              With Leader (coordinated):

All nodes try to coordinate:        One leader coordinates:

[Node A] ──write to DB              [Leader]
[Node B] ──write to DB (conflict!)      |
[Node C] ──write to DB (conflict!)      ├── instructs [Follower A]
                                         ├── instructs [Follower B]
Result: Data corruption ✗               └── instructs [Follower C]

                                   Result: Consistent actions ✓
```

**Common use cases**:
1. **Database writes**: Only leader writes, followers read
2. **Job scheduling**: Leader distributes jobs to workers
3. **Cache invalidation**: Leader coordinates cache updates
4. **Distributed locks**: Leader manages lock acquisition
5. **Cluster coordination**: Leader makes cluster-wide decisions

**Layman explanation**: Leader election is like choosing a team captain in sports. Everyone can participate, but one person (the leader) calls the plays and coordinates the team. If the captain gets injured (fails), the team elects a new captain to continue the game.

### Leader Election Requirements

```
Key properties of leader election:

1. Safety: At most one leader at any time
   ✓ [Leader A] at time t
   ✗ [Leader A] and [Leader B] at same time (split-brain!)

2. Liveness: Eventually there is a leader
   After failure, new leader elected within bounded time

3. Fairness: All candidates have equal chance
   Not always required, but desirable

4. Efficiency: Minimal overhead
   Election should be fast
   Network traffic should be low
```

**Split-brain problem** (must be avoided):
```
Network partition scenario:

[Node A] ──X── network partition ──X── [Node B]
[Node C] ──/                          \── [Node D]

Partition 1:                    Partition 2:
[Node A] elects itself leader   [Node B] elects itself leader
[Node C] follows A              [Node D] follows B

Result: TWO LEADERS! ✗
Both partitions act independently
Causes data inconsistency

Prevention:
- Require quorum (majority) to elect leader
- Partition with <50% nodes cannot elect leader
```

### ZooKeeper Recipes

**ZooKeeper** provides primitives for implementing distributed coordination patterns, including leader election.

#### ZooKeeper Leader Election Pattern

```
ZooKeeper namespace structure:

/election
  ├── candidate-0000000001  (Node A)
  ├── candidate-0000000002  (Node B)
  ├── candidate-0000000003  (Node C)
  └── candidate-0000000004  (Node D)

Leader election algorithm:

1. Each node creates sequential ephemeral node
   [Node A] ──create /election/candidate-──> [ZK]
   [ZK] assigns sequence: /election/candidate-0000000001

2. Each node lists children of /election
   [Node A] ──getChildren /election──> [ZK]
   Returns: [candidate-0000000001, candidate-0000000002, ...]

3. Node checks if its node is smallest sequence number
   Node A: My node is 0000000001
          Smallest node? YES → I am leader ✓

   Node B: My node is 0000000002
          Smallest node? NO → I am follower
          Watch node 0000000001 (watch previous)

4. If leader fails:
   [Node A] X (leader fails)
   |
   [ZK] deletes ephemeral node 0000000001
   |
   [Node B] receives watch notification
   |
   [Node B] ──getChildren /election
   Now 0000000002 is smallest → I am new leader ✓
```

**Why this approach works**:
- Ephemeral nodes automatically deleted on disconnect
- Sequential numbers provide total ordering
- Watching previous node avoids "herd effect"
- Only one node becomes leader (smallest sequence)

**Herd effect prevention**:
```
Without watching previous (BAD):

Leader fails:
[Node-0001] X
     |
All nodes notified simultaneously
     |
├── [Node-0002] ──getChildren
├── [Node-0003] ──getChildren
├── [Node-0004] ──getChildren  (thundering herd!)
└── [Node-0005] ──getChildren

With watching previous (GOOD):

[Node-0002] watches [Node-0001]
[Node-0003] watches [Node-0002]
[Node-0004] watches [Node-0003]

Leader fails:
[Node-0001] X
     |
Only [Node-0002] notified (watched 0001)
     |
[Node-0002] becomes leader
     |
No unnecessary notifications
```

#### ZooKeeper Leader Responsibilities

```
Leader lifecycle:

1. Become leader
   [Node A] ──smallest sequence number
   [Node A] ──perform leader initialization
               - Load state
               - Acquire resources
               - Start coordinating

2. Perform leader duties
   [Node A] ──coordinates cluster operations
   [Node A] ──responds to client requests
   [Node A] ──maintains leadership (heartbeat)

3. Detect loss of leadership
   [Node A] ──loses connection to ZooKeeper
   [Node A] ──immediately stop leader duties
   [Node A] ──cleanup resources
   [Node A] ──re-join election as candidate

Leadership takeover:
   [Node B] ──detects A's node deleted
   [Node B] ──becomes new leader
   [Node B] ──performs leader initialization
```

### etcd Leader Election

**etcd** uses distributed locks and leases for leader election.

#### etcd Election API

```
etcd leader election flow:

1. Create a lease (time-bound resource)
   [Node A] ──create lease (TTL=10s)──> [etcd]
   [etcd] ──returns lease ID: 123

2. Campaign for leadership with lease
   [Node A] ──campaign on key "/election/leader"
               with lease 123──> [etcd]

3. First node to write key becomes leader
   [etcd] checks: key exists? NO
   [etcd] writes key "/election/leader" = Node A (lease 123)
   [Node A] ──elected as leader! ✓

4. Other nodes block on campaign
   [Node B] ──campaign on "/election/leader"──> [etcd]
   [etcd] key already exists (Node A is leader)
   [Node B] ──waits (blocks) until key deleted

5. Leader keeps lease alive
   [Node A] ──keep-alive lease 123 every 3s

6. If leader fails:
   [Node A] X (crashed, no keep-alive)
   Lease 123 expires after 10s
   [etcd] ──deletes key "/election/leader"
   [Node B] ──unblocked from campaign
   [Node B] ──writes key "/election/leader" = Node B
   [Node B] ──elected as new leader! ✓
```

**etcd election primitives**:

**Campaign** (try to become leader):
```
Campaign(election_key, value, lease)
  - Attempts to write election_key with value
  - Blocks if key already exists
  - Returns when elected as leader
```

**Observe** (watch current leader):
```
Observe(election_key)
  - Watch for changes to election_key
  - Returns current leader value
  - Notified when leader changes
```

**Resign** (step down as leader):
```
Resign(election_key)
  - Delete election_key
  - Allows new leader election
```

#### etcd Lease-Based Elections

```
Lease mechanism:

[Node A creates lease]
     |
     v
[etcd Lease ID: 123, TTL: 10s]
     |
     ├── Keep-alive (t=3s)  → Lease renewed
     ├── Keep-alive (t=6s)  → Lease renewed
     ├── Keep-alive (t=9s)  → Lease renewed
     └── [Node A crashes at t=11s]
          |
          No keep-alive received
          |
     [etcd expires lease at t=13s]
          |
     All keys attached to lease deleted
          |
     New leader can be elected

Advantages:
✓ Automatic cleanup on failure
✓ Bounded time to detect failure (TTL)
✓ No manual cleanup needed
```

**Why leases are important**:
```
Without lease:
  Leader crashes → key remains forever
  Manual intervention required
  Delayed new leader election

With lease:
  Leader crashes → lease expires automatically
  Key deleted by etcd
  New leader elected within TTL
  No manual intervention needed
```

### Consul Leader Election

**Consul** provides leader election through session and key-value mechanisms.

#### Consul Session-Based Elections

```
Consul leader election flow:

1. Create a session
   [Node A] ──create session (TTL=10s)──> [Consul]
   [Consul] ──returns session ID: abc123

2. Try to acquire lock on key with session
   [Node A] ──PUT /v1/kv/election/leader
               ?acquire=abc123
               value="NodeA"

3. First session to acquire becomes leader
   [Consul] ──no existing lock → grant lock
   [Node A] ──lock acquired! I am leader ✓

4. Other nodes try to acquire, fail
   [Node B] ──PUT /v1/kv/election/leader?acquire=def456
   [Consul] ──lock already held by abc123
   [Node B] ──lock acquisition failed
   [Node B] ──watch key for changes

5. Leader renews session periodically
   [Node A] ──session TTL renewal every 5s

6. If leader fails:
   [Node A] X (no session renewal)
   Session abc123 expires
   [Consul] ──releases lock on key
   [Node B] ──watch triggered, retries acquisition
   [Node B] ──PUT /v1/kv/election/leader?acquire=def456
   [Consul] ──no lock → grant to def456
   [Node B] ──lock acquired! I am new leader ✓
```

**Consul session features**:
```
Session configuration:
{
  "TTL": "10s",          // Time-to-live
  "Behavior": "delete",  // delete or release locks on expiry
  "Checks": [           // Health checks linked to session
    "service:node-a"
  ]
}

If health check fails:
  Session invalidated immediately
  Locks released
  Faster failure detection than TTL
```

**Lock acquisition with blocking**:
```
Blocking query (wait for lock release):

[Node B] ──GET /v1/kv/election/leader
            ?wait=30s&index=42

If lock held:
  [Consul] blocks request until:
    - Lock released (leader failure)
    - Timeout (30s)
    - Index changes

When lock released:
  [Consul] returns immediately
  [Node B] tries to acquire lock
```

### Kubernetes Leader Election

**Kubernetes** uses lease-based leader election for controller high availability.

#### Kubernetes Lease API

```
Kubernetes leader election:

1. Create/update Lease resource
   apiVersion: coordination.k8s.io/v1
   kind: Lease
   metadata:
     name: my-controller-leader
     namespace: default
   spec:
     holderIdentity: pod-a
     leaseDurationSeconds: 15
     acquireTime: "2026-02-06T10:00:00Z"
     renewTime: "2026-02-06T10:00:05Z"

2. Pod A acquires lease
   [Pod A] ──create/update lease with holderIdentity="pod-a"
   [Pod A] ──becomes leader ✓

3. Pod A renews lease periodically
   [Pod A] ──update renewTime every 5s

4. Other pods try to acquire, see lease held
   [Pod B] ──read lease
   [Pod B] ──holderIdentity="pod-a", renewTime=recent
   [Pod B] ──wait as follower

5. If leader fails:
   [Pod A] X (crashed, no renewals)
   Lease renewTime becomes stale (>15s old)
   [Pod B] ──read lease, see stale renewTime
   [Pod B] ──update lease with holderIdentity="pod-b"
   [Pod B] ──becomes new leader ✓
```

#### Kubernetes Leader Election Library

```
Leader election using client-go library:

Components:

[LeaderElector]
     |
     ├── OnStartedLeading() callback
     |   └── Execute leader responsibilities
     |
     ├── OnStoppedLeading() callback
     |   └── Cleanup leader resources
     |
     └── OnNewLeader() callback
         └── Notify when new leader elected

Workflow:

1. Start election process
   [Controller] ──Run LeaderElector

2. Competing for lease
   Multiple pods try to acquire same lease

3. Winner becomes leader
   [Pod A] ──acquires lease
   [Pod A] OnStartedLeading() triggered
   [Pod A] ──starts controller logic

4. Follower behavior
   [Pod B] OnNewLeader("pod-a") triggered
   [Pod B] ──waits, monitors lease

5. Leader maintains lease
   [Pod A] ──automatically renews lease

6. Leader failure
   [Pod A] X
   [Pod A] OnStoppedLeading() triggered (if graceful)
   [Pod B] ──detects stale lease
   [Pod B] ──acquires lease
   [Pod B] OnStartedLeading() triggered
   [Pod B] ──takes over controller logic
```

**Kubernetes use cases**:
```
Controller high availability:

[kube-scheduler replica 1] ───┐
[kube-scheduler replica 2] ───┼─ One leader, others standby
[kube-scheduler replica 3] ───┘

Only leader schedules pods
Followers wait as hot standby
Automatic failover on leader failure

Same pattern for:
- kube-controller-manager
- Custom controllers
- Operators
```

### Comparison of Leader Election Systems

```
Feature              ZooKeeper    etcd        Consul      Kubernetes
────────────────────────────────────────────────────────────────────────
Consistency model    CP           CP          CP          CP
Quorum required      Yes          Yes         Yes         Yes
Automatic cleanup    Yes          Yes         Yes         Yes
Split-brain safe     Yes          Yes         Yes         Yes
Session/Lease TTL    Yes          Yes         Yes         Yes
Watch mechanism      Yes          Yes         Yes         Yes
External dependency  ZK cluster   etcd        Consul      K8s API
Maturity             High         High        High        Medium
Ease of use          Medium       Easy        Medium      Easy (in K8s)
```

**Choosing a system**:
```
Use ZooKeeper when:
- Need mature, battle-tested solution
- Complex coordination patterns
- Already using Kafka/Hadoop ecosystem

Use etcd when:
- Need simple key-value semantics
- Using Kubernetes (etcd is backing store)
- Want modern, easy-to-use API

Use Consul when:
- Need service mesh capabilities
- Multi-datacenter deployment
- Want integrated service discovery + election

Use Kubernetes Lease when:
- Building Kubernetes controllers/operators
- Already running in Kubernetes
- Need high availability for controllers
```

**Layman explanation**: Leader election is like a game of "hot potato." Everyone passes around a token (lease). Whoever holds the token when the music stops (lease acquired) becomes the leader and makes decisions. If the leader drops the token (crashes), someone else picks it up (new leader elected). The system ensures only one person holds the token at a time, preventing conflicts.

---

## Distributed Semaphores and Barriers

**Distributed semaphores** and **barriers** are coordination primitives that control concurrent access to resources and synchronize multiple processes across a distributed system.

### Distributed Semaphores

A **semaphore** is a counter that controls access to a limited resource. In distributed systems, semaphores coordinate access across multiple machines.

#### What Problem Do Semaphores Solve?

```
Without Semaphore (resource overload):

Limited resource: Database (max 10 connections)

[Service 1] ──┐
[Service 2] ──┤
[Service 3] ──┤
[Service 4] ──┼──> [Database]
[Service 5] ──┤
    ...       │    (15 services × 10 connections each
[Service 15] ─┘     = 150 connection attempts!)

Result: Database overwhelmed ✗
        Connection pool exhausted
        Services fail


With Semaphore (controlled access):

Semaphore: max 10 permits

[Service 1] ──acquire permit 1──┐
[Service 2] ──acquire permit 2──┤
[Service 3] ──acquire permit 3──┤
    ...                          ├──> [Database]
[Service 10] ─acquire permit 10─┤    (exactly 10 connections)
                                 │
[Service 11] ─wait (no permits)─┘
[Service 12] ─wait (no permits)
    ...
[Service 15] ─wait (no permits)

When Service 1 finishes:
  ──release permit 1
  [Service 11] acquires permit 1

Result: Database protected ✓
        Fair resource distribution
```

**Layman explanation**: A semaphore is like a coat check with limited hooks. The coat check has 10 hooks (permits). When all hooks are full, new arrivals must wait until someone leaves and frees up a hook. This prevents overcrowding and ensures everyone gets fair access.

#### Counting Semaphores

**Counting semaphores** allow N concurrent accessors.

```
Counting Semaphore:

Initial state:
  Available permits: 5
  ┌─┬─┬─┬─┬─┐
  │✓│✓│✓│✓│✓│  (5 free permits)
  └─┴─┴─┴─┴─┘

After 3 processes acquire:
  Available permits: 2
  ┌─┬─┬─┬─┬─┐
  │X│X│X│✓│✓│  (3 in use, 2 free)
  └─┴─┴─┴─┴─┘
  [P1] [P2] [P3] holding permits

Process 4 tries to acquire:
  Available permits: 1
  ┌─┬─┬─┬─┬─┐
  │X│X│X│X│✓│  (4 in use, 1 free)
  └─┴─┴─┴─┴─┘
  [P1] [P2] [P3] [P4] holding permits

Process 5 tries to acquire:
  Available permits: 0
  ┌─┬─┬─┬─┬─┐
  │X│X│X│X│X│  (all in use)
  └─┴─┴─┴─┴─┘
  [P1] [P2] [P3] [P4] [P5] holding permits

Process 6 tries to acquire:
  [P6] ──blocks, waits for permit release

Process 2 releases permit:
  Available permits: 1
  ┌─┬─┬─┬─┬─┐
  │X│✓│X│X│X│  (permit freed)
  └─┴─┴─┴─┴─┘
  [P6] ──acquires freed permit
```

**Semaphore operations**:
```
acquire(n=1):
  - Decrease available permits by n
  - Block if insufficient permits
  - Resume when permits available

release(n=1):
  - Increase available permits by n
  - Wake up waiting processes

tryAcquire(timeout):
  - Try to acquire permit
  - Return false if unavailable within timeout
  - Don't block indefinitely
```

#### ZooKeeper Semaphore Implementation

```
ZooKeeper-based semaphore:

Structure:
/semaphores
  └── /database-pool
        ├── limit = 10                    (max permits)
        ├── /locks
        │   ├── lock-0000000001  (Process A)
        │   ├── lock-0000000002  (Process B)
        │   ├── lock-0000000003  (Process C)
        │   └── ...

Algorithm:

1. Process wants to acquire permit
   [Process D] ──create /locks/lock-NNNNNN (ephemeral, sequential)
   [ZK] creates: /locks/lock-0000000011

2. Get all locks and check position
   [Process D] ──getChildren /locks
   Returns: [lock-0000000001, lock-0000000002, ..., lock-0000000011]
   Count: 11 locks exist

3. Check if within limit
   If position ≤ limit (10):
     Permit acquired ✓
   Else:
     [Process D] watches lock at position (current_pos - limit)
     [Process D] waits for that lock to be deleted

4. When Process A finishes and releases:
   [Process A] ──delete /locks/lock-0000000001
   [Process D] ──watch triggered
   [Process D] ──rechecks count: 10 locks exist
   [Process D] ──permit acquired ✓

5. Process D finishes:
   [Process D] ──delete /locks/lock-0000000011
   [Process E] (if waiting) ──permit acquired
```

**Advantages of ZooKeeper approach**:
- Automatic cleanup (ephemeral nodes)
- FIFO ordering (sequential nodes)
- No central counter to update
- Handles failures gracefully

#### etcd Semaphore Implementation

```
etcd-based semaphore using leases:

Structure:
Key prefix: /semaphores/database-pool/
Max permits: 10

Permit tracking:
/semaphores/database-pool/permit-1  (lease: 123, holder: process-a)
/semaphores/database-pool/permit-2  (lease: 456, holder: process-b)
...
/semaphores/database-pool/permit-10 (lease: 789, holder: process-j)

Algorithm:

1. Process wants permit
   [Process K] ──create lease (TTL=10s)
   [Process K] ──list keys with prefix /semaphores/database-pool/

2. Check available permits
   If (count of keys) < max_permits:
     [Process K] ──PUT /semaphores/database-pool/permit-X
                    with lease, value=process-k
     Permit acquired ✓
   Else:
     [Process K] ──wait, watch for deletions

3. Keep lease alive
   [Process K] ──keep-alive lease every 3s

4. Release permit
   [Process K] ──delete key /semaphores/database-pool/permit-X
   or
   [Process K] crashes → lease expires → key deleted

5. Waiting process acquires freed permit
   [Process L] ──watch triggered
   [Process L] ──tries to acquire permit
```

### Barriers for Coordination

A **barrier** is a synchronization point where all processes must arrive before any can proceed. Think of it as a "wait for everyone" checkpoint.

#### Basic Barrier

```
Barrier with 3 processes:

Goal: Start job only when all 3 processes ready

t=0: [Process A] ──ready, wait at barrier
     Barrier count: 1/3

t=1: [Process B] ──ready, wait at barrier
     Barrier count: 2/3

t=2: [Process C] ──ready, wait at barrier
     Barrier count: 3/3

     ALL READY!
     ┌────────────┐
     │  BARRIER   │
     │  RELEASED  │
     └────────────┘
           |
     ┌─────┼─────┐
     v     v     v
  [Proc A] [Proc B] [Proc C]
  all proceed together
```

**Use cases**:
- Distributed MapReduce: All mappers finish before reducers start
- Parallel computations: Wait for all workers before aggregating results
- Synchronized starts: Begin processing data at same time
- Batch processing: Process next batch only when all nodes ready

**Layman explanation**: A barrier is like a tour group meeting point. The tour guide (barrier) waits until all tourists (processes) arrive before continuing the tour. Nobody goes ahead until everyone is ready.

#### ZooKeeper Barrier Implementation

```
ZooKeeper barrier:

Structure:
/barriers
  └── /job-123
        ├── ready-count = 0
        ├── size = 3               (expected number of processes)
        └── /ready
              ├── process-a
              ├── process-b
              └── process-c

Enter barrier flow:

1. Process arrives at barrier
   [Process A] ──create /barriers/job-123/ready/process-a (ephemeral)
   [Process A] ──increment ready-count

2. Process checks if barrier released
   [Process A] ──watch ready-count
   [Process A] ──if ready-count < size: wait

3. Last process arrives
   [Process C] ──create /barriers/job-123/ready/process-c
   [Process C] ──increment ready-count
   ready-count = 3 (equals size)

4. Barrier released
   [Barrier] ──notifies all waiting processes
   All processes ──proceed

Leave barrier flow:

1. Process completes work
   [Process A] ──delete /barriers/job-123/ready/process-a
   [Process A] ──decrement ready-count

2. Last process to leave cleans up
   [Process C] ──delete /barriers/job-123/ready/process-c
   ready-count = 0
   [Process C] ──delete /barriers/job-123 (barrier node)
```

### Double Barriers

**Double barriers** synchronize both the start AND the end of a distributed computation.

```
Double Barrier:

Phase 1: Enter barrier (wait for all to start)

t=0: [Process A] ──enters barrier, waits
t=1: [Process B] ──enters barrier, waits
t=2: [Process C] ──enters barrier, waits
     All present: enter_count = 3 = size ✓

     ┌────────────┐
     │ ALL READY  │
     │   START!   │
     └────────────┘
           |
     ┌─────┼─────┐
     v     v     v
  [Proc A] [Proc B] [Proc C]
  all start work together

Phase 2: Leave barrier (wait for all to finish)

t=5: [Process B] ──finishes work, enters leave barrier, waits
t=6: [Process A] ──finishes work, enters leave barrier, waits
t=7: [Process C] ──finishes work, enters leave barrier, waits
     All finished: leave_count = 3 = size ✓

     ┌────────────┐
     │ ALL DONE!  │
     │  RELEASE   │
     └────────────┘
           |
     ┌─────┼─────┐
     v     v     v
  [Proc A] [Proc B] [Proc C]
  all exit together
```

**Implementation pattern**:
```
ZooKeeper double barrier:

/barriers
  └── /computation-456
        ├── size = 5
        ├── /enter
        │   └── (processes create nodes here)
        └── /leave
            └── (processes create nodes here)

Enter phase:
  [Process] ──create /enter/process-X
  [Process] ──wait until count(/enter) = size
  [All present] ──start work

Computation phase:
  [Processes] ──do work independently

Leave phase:
  [Process] ──create /leave/process-X
  [Process] ──wait until count(/leave) = size
  [All done] ──exit together
```

**Use cases**:
- Synchronized batch processing
- Parallel algorithms requiring coordination
- Distributed simulations (lockstep execution)
- Testing (all services start/stop together)

### Implementation Patterns

#### Pattern 1: Fair Queueing Semaphore

```
Ensures FIFO ordering for semaphore acquisition:

Traditional semaphore (unfair):
  [P1] waits → [P2] waits → [P3] waits
  Permit released → [P2] acquires (random/fastest)
  Not fair!

Fair queue semaphore:
  [P1] joins queue at position 1
  [P2] joins queue at position 2
  [P3] joins queue at position 3

  Queue: [P1] [P2] [P3]

  Permit released → [P1] acquires (first in queue)
  Fair ✓

Implementation:
  Use sequential ephemeral nodes (ZooKeeper)
  Watch previous node in sequence
  Acquire permit when at front of queue
```

#### Pattern 2: Timed Barriers

```
Barrier with timeout:

Standard barrier:
  Wait indefinitely until all arrive
  Problem: One slow/failed process blocks everyone

Timed barrier:
  Wait up to T seconds
  If not all present by timeout:
    Release anyway
    Or fail entire batch
    Or proceed with partial set

Timeline:
  t=0:  [P1] arrives
  t=1:  [P2] arrives
  t=2:  [P3] arrives
  t=30: Timeout! (expected P4, P5 didn't arrive)

  Options:
    1. Release [P1] [P2] [P3] anyway
    2. Fail and retry batch
    3. Mark batch partial, proceed
```

#### Pattern 3: Reentrant Semaphores

```
Allow same process to acquire multiple times:

Problem with regular semaphore:
  [Process A] ──acquires permit 1
  [Process A] ──calls function that tries to acquire again
  [Process A] ──deadlocks! (waiting for itself)

Reentrant semaphore:
  [Process A] ──acquires permit 1
  Counter: {Process A: 1}

  [Process A] ──acquires again (same process)
  Counter: {Process A: 2}
  ✓ Succeeds without blocking

  [Process A] ──release
  Counter: {Process A: 1}

  [Process A] ──release
  Counter: {Process A: 0}
  Permit fully released

Implementation:
  Track holder identity with permit
  Allow re-acquisition by same holder
  Require matching number of releases
```

#### Pattern 4: Hierarchical Barriers

```
Nested barriers for phased computation:

Barrier hierarchy:

[Global Barrier]          (all processes)
  ├── [Group 1 Barrier]   (processes 1-3)
  │     ├── Phase 1a
  │     ├── Phase 1b
  │     └── Phase 1c
  ├── [Group 2 Barrier]   (processes 4-6)
  │     ├── Phase 2a
  │     └── Phase 2b
  └── [Group 3 Barrier]   (processes 7-9)
        └── Phase 3a

Execution:
  1. All groups work independently within their barriers
  2. Each group waits at global barrier when done
  3. When all groups ready, all proceed to next stage

Use case: MapReduce
  Group barriers: Individual map/reduce phases
  Global barrier: All maps done before any reduce starts
```

**Layman explanation**: Distributed semaphores and barriers are like traffic control systems. Semaphores are like traffic lights controlling how many cars (processes) can enter an intersection (resource) at once. Barriers are like parade coordination—all marching bands must wait at the starting line until everyone's ready, then they all start together. Both prevent chaos and ensure orderly, synchronized operation.

---

## Coordination Patterns

Beyond basic primitives, distributed systems use higher-level **coordination patterns** to solve common distributed problems.

### Distributed Queues

A **distributed queue** allows multiple producers and consumers to communicate asynchronously across a distributed system.

#### Priority Queue Pattern

```
ZooKeeper-based priority queue:

Structure:
/queue
  ├── priority-1
  │   ├── task-0000000001
  │   ├── task-0000000005
  │   └── task-0000000009
  ├── priority-2
  │   ├── task-0000000002
  │   └── task-0000000007
  └── priority-3
      ├── task-0000000003
      └── task-0000000008

Producer adds task:

[Producer] ──create task with priority 1
[ZK] creates: /queue/priority-1/task-NNNNNN (sequential)
Task added to priority 1 queue ✓

Consumer processes tasks:

1. [Consumer] ──list all priority levels
2. [Consumer] ──get children of highest priority (priority-1)
3. [Consumer] ──sort by sequence number
4. [Consumer] ──process smallest sequence first (FIFO within priority)
5. [Consumer] ──delete task node after processing
6. Repeat from step 2
```

**Fair work distribution**:
```
Multiple consumers competing for work:

Queue:
  [Task 1] [Task 2] [Task 3] [Task 4] [Task 5]

Consumers:
  [Consumer A] ──tries to delete Task 1
  [Consumer B] ──tries to delete Task 1 (same task!)

  Only one succeeds (atomic delete in ZK)
  [Consumer A] ──delete succeeds ✓ (owns Task 1)
  [Consumer B] ──delete fails ✗ (already deleted)

  [Consumer B] ──tries Task 2
  [Consumer B] ──delete succeeds ✓ (owns Task 2)

Each task processed by exactly one consumer
No coordination overhead between consumers
```

#### Delayed Queue Pattern

```
Tasks that should execute only after a delay:

Structure:
/delayed-queue
  └── /tasks
        ├── execute_at=1709723400-task-0001
        ├── execute_at=1709723500-task-0002
        └── execute_at=1709723600-task-0003

Consumer watches for ready tasks:

[Consumer] ──periodically scan queue
           ──current_time = 1709723450
           ──find tasks where execute_at ≤ current_time
           ──process: task-0001 (ready)
           ──skip: task-0002, task-0003 (not ready yet)

Alternative: Use TTL
[Producer] ──create task with TTL until execute_at
[Task] invisible until TTL expires
[Consumer] ──sees task only when ready
```

### Distributed Counters

**Distributed counters** track shared counts across multiple processes without conflicts.

#### Optimistic Counter (Eventually Consistent)

```
Each process maintains local count, periodically syncs:

Process A counter: 5
Process B counter: 3
Process C counter: 7

Sync to central store:
[Process A] ──increment /counter/process-a by 5
[Process B] ──increment /counter/process-b by 3
[Process C] ──increment /counter/process-c by 7

Total count:
[Reader] ──sum all counters
Total = 5 + 3 + 7 = 15

Advantages:
✓ Low coordination overhead
✓ High performance (local increments)
✓ Scales well

Disadvantages:
✗ Eventually consistent (not exact in real-time)
✗ Requires periodic sync
```

**Use cases**: Page views, metrics, approximate counts

#### Distributed Counter with CAS (Compare-And-Swap)

```
Strong consistency using atomic operations:

Current counter value: 42

Process A wants to increment:
1. [Process A] ──read counter value: 42
2. [Process A] ──compute new value: 43
3. [Process A] ──CAS(expected=42, new=43)
   If current value still 42:
     Update to 43 ✓
   Else:
     Retry (someone else modified it)

Concurrent increment:

t=0: Counter = 42
     [Process A] reads 42
     [Process B] reads 42

t=1: [Process A] ──CAS(expected=42, new=43) ✓ succeeds
     Counter = 43

t=2: [Process B] ──CAS(expected=42, new=43) ✗ fails
     (counter is now 43, not 42)
     [Process B] reads current value (43)
     [Process B] ──CAS(expected=43, new=44) ✓ succeeds
     Counter = 44

Result: No lost updates ✓
```

**etcd implementation**:
```
Atomic increment using etcd transaction:

Transaction:
  IF counter.value == 42:    (compare)
    PUT counter.value = 43   (swap)
  ELSE:
    fail, retry

Guarantees:
✓ Atomic operation
✓ Strong consistency
✓ No lost updates
```

#### Sharded Counter (High Throughput)

```
Split counter across multiple shards:

Counter shards:
/counters
  ├── shard-0: 1,234
  ├── shard-1: 2,456
  ├── shard-2: 1,789
  └── shard-3: 3,012

Increment operation:
[Process A] ──hash(process_id) % num_shards = 2
[Process A] ──increment /counters/shard-2
No contention with other shards

Read total:
[Reader] ──sum all shards
Total = 1,234 + 2,456 + 1,789 + 3,012 = 8,491

Advantages:
✓ Parallel increments (no contention)
✓ High throughput
✓ Scales with shard count

Disadvantages:
✗ Reading total requires aggregation
✗ More complex
```

**Choosing strategy**:
```
Use Case                 Strategy
────────────────────────────────────────────────
Metrics, analytics       Optimistic (eventually consistent)
Click counters           Sharded (high throughput)
Financial balances       CAS (strong consistency)
Rate limiting            Sharded with periodic sync
Page views               Optimistic
Inventory counts         CAS
```

### Group Membership

**Group membership** tracks which processes are currently alive and part of a group.

#### Ephemeral Membership Pattern

```
ZooKeeper ephemeral nodes for membership:

Group structure:
/groups
  └── /web-servers
        ├── server-1 (ephemeral) → "192.168.1.10:8080"
        ├── server-2 (ephemeral) → "192.168.1.11:8080"
        └── server-3 (ephemeral) → "192.168.1.12:8080

Process joins group:

[Server-1] ──connects to ZooKeeper
[Server-1] ──creates /groups/web-servers/server-1 (ephemeral)
           with value "192.168.1.10:8080"
[Server-1] ──joined group ✓

Process monitors group:

[Monitor] ──getChildren /groups/web-servers
          ──watch for changes

Current members: [server-1, server-2, server-3]

Process leaves group (graceful):

[Server-2] ──explicitly deletes /groups/web-servers/server-2
[Monitor] ──watch triggered
[Monitor] ──current members: [server-1, server-3]

Process crashes (ungraceful):

[Server-3] X (crashes, loses ZK connection)
[ZK] ──automatically deletes ephemeral node server-3
[Monitor] ──watch triggered
[Monitor] ──current members: [server-1]
```

**Membership with metadata**:
```
Store rich information in member nodes:

/groups/web-servers/server-1:
{
  "host": "192.168.1.10",
  "port": 8080,
  "version": "1.2.0",
  "region": "us-east-1",
  "capacity": 100,
  "current_load": 45
}

Use cases:
- Load balancing (choose lowest load)
- Version-aware routing
- Region-based routing
- Health status tracking
```

#### Membership with Heartbeat

```
Active heartbeat pattern:

Members send periodic heartbeats:

[Server-1] ──heartbeat every 5s──> [Coordinator]
           ──update timestamp

[Server-2] ──heartbeat every 5s──> [Coordinator]
           ──update timestamp

Coordinator checks liveness:

[Coordinator] ──every 10s:
                check last_heartbeat for all members

[Server-1] last_heartbeat: 3s ago ✓ alive
[Server-2] last_heartbeat: 2s ago ✓ alive
[Server-3] last_heartbeat: 25s ago ✗ dead

[Coordinator] ──removes Server-3 from group
              ──notifies watchers

Implementation with etcd:

[Member] ──creates lease (TTL=10s)
[Member] ──register with lease
[Member] ──keep-alive lease every 5s
         ──if crashes, lease expires
         ──registration auto-deleted
```

### Watch and Notify

**Watch mechanisms** allow processes to be notified when data changes.

#### ZooKeeper Watch Pattern

```
Watch types in ZooKeeper:

1. Data watch (value changes):
   [Client] ──exists /config/timeout, watch=true
   [ZK] ──notifies if node created/deleted/modified

2. Children watch (child nodes change):
   [Client] ──getChildren /services/user-svc, watch=true
   [ZK] ──notifies if children added/removed

Watch characteristics:
- One-time trigger (must re-set after trigger)
- Ordered (watch notifications in order)
- Guaranteed delivery (before seeing new state)

Example flow:

t=0: [Client] ──getData /config/timeout, watch=true
     Returns: 30
     Watch set ✓

t=1: [Admin] ──setData /config/timeout = 60
     [ZK] triggers watch for Client

t=2: [Client] receives notification
     [Client] ──getData /config/timeout, watch=true (re-set watch)
     Returns: 60
     Watch re-set ✓

If watch not re-set:
     Future changes won't notify ✗
```

**One-time watch limitation**:
```
Problem: Lost updates if not careful

t=0: [Client] ──getData /config, watch=true
     Value: A

t=1: [Admin] ──setData /config = B
     [ZK] triggers watch

t=2: [Client] processes notification slowly...

t=3: [Admin] ──setData /config = C
     No watch set! Client misses update ✗

t=4: [Client] ──getData /config, watch=true
     Value: C (sees C, never saw B!)

Solution: Re-set watch BEFORE processing

t=2: [Client] receives notification
     [Client] ──getData /config, watch=true (re-set FIRST)
     Then process value
     Won't miss intermediate updates ✓
```

#### etcd Watch Pattern

```
etcd persistent watch (automatic re-watch):

[Client] ──watch key="/config/timeout"

[etcd] ──returns watch stream (persistent connection)

Changes stream automatically:

t=0: /config/timeout = 30
t=1: /config/timeout = 60
     [Client] ←─ notification: old=30, new=60
t=2: /config/timeout = 90
     [Client] ←─ notification: old=60, new=90

No manual re-watch needed ✓
Stream remains open until closed
```

**Range watch** (watch multiple keys):
```
Watch all keys with prefix:

[Client] ──watch prefix="/config/"

Notifications for:
  /config/database/host
  /config/database/port
  /config/timeout
  /config/features/new_ui
  ... (all keys under /config/)

Use case:
  Monitor entire configuration subtree
  Get notified of any config change
```

**Watch with revision** (historical replay):
```
Watch from specific point in history:

etcd maintains revision history:
  rev 1: /config/timeout = 10
  rev 2: /config/timeout = 20
  rev 3: /config/timeout = 30
  rev 4: /config/timeout = 40 (current)

[Client] ──watch /config/timeout, start_revision=2

[etcd] ──replays changes from rev 2 onward:
       ──notification: rev 2, value = 20
       ──notification: rev 3, value = 30
       ──notification: rev 4, value = 40
       ──continues watching for future changes

Use case:
  Catch up on missed events
  Replay history for recovery
```

#### Consul Watch Pattern

```
Consul blocking queries (long-polling):

[Client] ──GET /v1/kv/config/timeout?wait=5m&index=42

If no change within 5 minutes:
  [Consul] ──returns: timeout, index=42
  [Client] ──re-query with same index

If change occurs:
  [Consul] ──immediately returns: new value, index=43
  [Client] ──processes update
  [Client] ──re-query with index=43

Advantages:
✓ HTTP-based (no special protocol)
✓ Efficient (server holds connection)
✓ Automatic retry logic

Blocking query parameters:
  wait: max time to block (default 5m)
  index: last seen modification index
```

**Consul watch types**:
```
Service watch:
  Watch for changes to service instances
  [Client] ──watch service="user-service"
  Notified when instances added/removed

Key watch:
  Watch single key-value pair
  [Client] ──watch key="config/timeout"

Key prefix watch:
  Watch all keys with prefix
  [Client] ──watch prefix="config/"

Health watch:
  Watch service health status
  [Client] ──watch health="user-service"
  Notified on health changes
```

### Coordination Pattern Comparison

```
Pattern              ZooKeeper        etcd             Consul
─────────────────────────────────────────────────────────────────
Distributed Queue    Sequential       Manual           KV + Sessions
                     nodes            ordering

Counter              CAS              Transactions     CAS
                     ZNode version

Group Membership     Ephemeral        Leases           Sessions
                     nodes

Watch/Notify         One-time         Persistent       Blocking
                     re-watch         stream           queries

Barriers             Built-in         Manual           Manual
                     recipe

Semaphores           Sequential       Prefix           Sessions
                     nodes            counting         + KV
```

**Choosing the right pattern**:
```
Need                          Best Pattern
──────────────────────────────────────────────────
Task distribution             Distributed Queue
Metrics aggregation           Distributed Counter (sharded)
Service discovery             Group Membership + Watch
Real-time notifications       Watch/Notify
Synchronized execution        Barriers
Resource pooling              Semaphores
Leader election               Built-in leader election
Configuration management      Watch + Versioning
```

---

## Summary

**Distributed coordination** enables independent services to work together effectively. Key takeaways:

1. **Service Discovery**: Allows services to find each other dynamically (client-side vs server-side, with health checking)

2. **Configuration Management**: Centralized, versioned configuration with dynamic updates and consistency guarantees

3. **Leader Election**: Ensures single coordinator with automatic failover (ZooKeeper, etcd, Consul, Kubernetes)

4. **Semaphores & Barriers**: Control concurrent access (semaphores) and synchronize multiple processes (barriers)

5. **Coordination Patterns**: Higher-level patterns like distributed queues, counters, group membership, and watch/notify mechanisms

The right coordination approach depends on your consistency requirements, performance needs, and existing infrastructure. Most modern systems use a combination of these patterns to achieve robust distributed coordination.