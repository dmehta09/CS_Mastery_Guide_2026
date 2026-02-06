# STAFF ENGINEER SYSTEM DESIGN MASTERY CURRICULUM (2026)

> A comprehensive, deeply-researched curriculum for mastering system design at the Staff Engineer level. Covers 150+ core concepts organized into progressive tiers.

---

## **TIER 1: FOUNDATIONAL PILLARS**

### **1. System Design Thinking**

#### Requirements Engineering
- Functional vs non-functional requirements decomposition
- Constraint identification (budget, timeline, team skills, compliance)
- Stakeholder alignment techniques
- Writing technical requirements documents (TRDs)
- Identifying hidden requirements and edge cases

#### Back-of-Envelope Calculations
- QPS/RPS estimation from DAU/MAU
- Storage growth modeling (daily/monthly/yearly projections)
- Bandwidth calculations (peak vs sustained traffic)
- Memory requirements for caching layers
- Cost modeling per 1M requests
- Network throughput calculations
- Database storage estimation (row size × count × replication factor)

#### Trade-off Analysis
- Consistency vs availability trade-offs (quantified with examples)
- Latency vs throughput optimization curves
- Cost vs performance optimization
- Build vs buy decision frameworks
- Technical debt impact analysis
- Complexity vs flexibility trade-offs
- Operational overhead considerations

---

### **2. Scalability Principles**

#### Scaling Dimensions
- Horizontal scaling (scale-out): Adding more machines
- Vertical scaling (scale-up): Adding more resources to existing machines
- Diagonal scaling: Combining horizontal and vertical
- Auto-scaling strategies and policies

#### State Management
- Stateless vs stateful systems
- Session affinity and sticky sessions
- Externalizing state (Redis, databases)
- State serialization and transfer

#### Read vs Write Scaling
- Read replicas for read-heavy workloads
- Write sharding for write-heavy workloads
- CQRS for mixed workloads
- Read-after-write consistency challenges

#### Database Scaling Strategies
- Connection pooling
- Query optimization before scaling
- Vertical scaling limits
- Read replicas
- Sharding and partitioning strategies
- Caching layers

---

### **3. Load Balancing**

#### Load Balancing Algorithms
- Round-robin (simple, equal distribution)
- Weighted round-robin (capacity-aware)
- Least connections (dynamic load)
- Weighted least connections
- IP hash (session persistence)
- Least response time
- Random with two choices (power of two)

#### Layer 4 vs Layer 7 Load Balancing
- L4: TCP/UDP level, faster, no content inspection
- L7: Application level, content-aware routing, SSL termination
- When to use each layer
- Hybrid approaches

#### Health Checks and Failover
- Active health checks (synthetic probes)
- Passive health checks (real traffic analysis)
- Health check intervals and thresholds
- Graceful degradation
- Failover mechanisms

#### Global Load Balancing
- GeoDNS-based routing
- Anycast routing
- Latency-based routing
- Geolocation-based routing
- Failover across regions

#### Load Balancer Types
- Hardware load balancers (F5, Citrix)
- Software load balancers (HAProxy, NGINX, Envoy)
- Cloud load balancers (ALB, NLB, GLB)
- DNS load balancing

---

### **4. Caching Strategies**

#### Cache Patterns
- Cache-aside (lazy loading): App manages cache
- Write-through: Write to cache and DB simultaneously
- Write-behind (write-back): Write to cache, async to DB
- Read-through: Cache manages DB reads
- Refresh-ahead: Proactive cache refresh

#### Cache Invalidation Strategies
- TTL-based expiration
- Event-driven invalidation
- Write-invalidate vs write-update
- Versioned keys
- Cache stampede prevention (locking, probabilistic early expiration)
- Lease-based invalidation

#### Distributed Caching
- Redis: Data structures, clustering, persistence
- Memcached: Simple key-value, multi-threaded
- Cache sharding strategies
- Consistent hashing for cache distribution
- Cache replication vs partitioning

#### CDN and Edge Caching
- CDN architecture and PoPs
- Cache-Control headers
- Edge caching strategies
- Origin shield pattern
- Cache purging and invalidation
- Dynamic content caching

#### Cache Consistency Models
- Strong consistency (single cache)
- Eventual consistency (distributed cache)
- Cache coherency protocols
- Read-your-writes in caching

#### Cache Performance
- Hot key handling (local caching, key splitting)
- Cache warming strategies
- Eviction policies (LRU, LFU, ARC, FIFO)
- Memory fragmentation management
- Cache hit ratio optimization

---

### **5. Database Design Fundamentals**

#### SQL vs NoSQL Selection
- ACID requirements → SQL
- Flexible schema → Document stores
- High write throughput → Wide-column stores
- Relationship-heavy → Graph databases
- Simple key-value access → Key-value stores
- Decision matrix and trade-offs

#### ACID Properties Deep Dive
- Atomicity: All or nothing transactions
- Consistency: Valid state transitions
- Isolation: Concurrent transaction handling
- Durability: Committed data survives failures
- Implementation mechanisms (WAL, locking, MVCC)

#### Database Normalization vs Denormalization
- Normal forms (1NF through BCNF)
- When to denormalize
- Denormalization patterns
- Trade-offs: Storage vs query complexity

#### Indexing Strategies
- B-tree indexes (range queries)
- Hash indexes (equality lookups)
- Composite indexes (column ordering matters)
- Covering indexes
- Partial indexes
- Expression indexes
- Full-text indexes
- Spatial indexes (R-tree, GiST)

#### Query Optimization
- EXPLAIN plan analysis
- Index selection
- Query rewriting
- Join optimization
- Avoiding N+1 queries
- Pagination strategies (offset vs cursor)

---

### **6. Back-pressure and Flow Control**

#### Handling Overload Scenarios
- Recognizing overload conditions
- Graceful degradation strategies
- Load shedding decisions
- Priority-based processing

#### Backpressure Propagation
- Reactive streams backpressure
- TCP flow control
- Application-level backpressure
- Propagating pressure to upstream services

#### Queue Management Strategies
- Bounded vs unbounded queues
- Queue overflow policies (drop, reject, block)
- Priority queues
- Delay queues
- Dead letter queues

#### Producer-Consumer Patterns
- Single producer/single consumer
- Multiple producers/single consumer
- Single producer/multiple consumers
- Multiple producers/multiple consumers
- Work stealing patterns

#### Throttling Mechanisms
- Client-side throttling
- Server-side throttling
- Adaptive throttling
- Token bucket implementation
- Leaky bucket implementation

---

### **7. Data Modeling Fundamentals**

#### Domain-Driven Design Basics
- Ubiquitous language
- Bounded contexts
- Context mapping
- Strategic vs tactical DDD

#### Entity Relationships
- One-to-one relationships
- One-to-many relationships
- Many-to-many relationships
- Self-referential relationships
- Polymorphic relationships

#### Data Lifecycle Management
- Data creation and ingestion
- Data transformation and enrichment
- Data archival strategies
- Data deletion and retention
- Data recovery procedures

#### Schema Evolution Strategies
- Backward compatibility
- Forward compatibility
- Full compatibility
- Schema versioning
- Online schema migrations

#### Data Integrity Constraints
- Primary keys and uniqueness
- Foreign key constraints
- Check constraints
- NOT NULL constraints
- Application-level constraints

---

## **TIER 2: DISTRIBUTED SYSTEMS CORE**

### **8. Distributed Systems Basics**

#### CAP Theorem
- Consistency: All nodes see same data
- Availability: Every request receives response
- Partition tolerance: System works despite network partitions
- CA, CP, AP system examples
- CAP theorem limitations and misconceptions

#### PACELC Theorem
- Extension of CAP
- Partition: Availability vs Consistency
- Else (no partition): Latency vs Consistency
- Real-world system classification

#### BASE Properties
- Basically Available
- Soft state
- Eventual consistency
- BASE vs ACID trade-offs

#### Distributed System Challenges
- Network unreliability
- Clock synchronization
- Partial failures
- Non-determinism
- Debugging complexity

#### Network Partitions and Failure Modes
- Network partition types (asymmetric, symmetric)
- Byzantine failures
- Crash failures
- Omission failures
- Timing failures

#### Fallacies of Distributed Computing
1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

---

### **9. Consistency Models**

#### Strong Consistency
- Linearizability: Real-time ordering guarantee
- Serializability: Transaction isolation
- Strict serializability: Combining both
- Implementation costs and trade-offs

#### Sequential Consistency
- All operations appear in some sequential order
- Weaker than linearizability
- Program order preserved per process

#### Causal Consistency
- Causally related operations ordered
- Concurrent operations may be seen differently
- Vector clocks for tracking causality

#### Session Consistency Guarantees
- Read-your-writes
- Monotonic reads
- Monotonic writes
- Writes-follow-reads

#### Eventual Consistency
- Convergence guarantee
- Anti-entropy protocols
- Read repair
- Merkle trees for synchronization
- Conflict resolution strategies

#### Strong Eventual Consistency (CRDTs)
- Conflict-free replicated data types
- G-Counter, PN-Counter
- G-Set, OR-Set, LWW-Register
- Operational vs state-based CRDTs
- Use cases: Collaborative editing, offline-first apps

---

### **10. Replication & Sharding**

#### Replication Topologies
- Single-leader (primary-backup/master-slave)
- Multi-leader (active-active/master-master)
- Leaderless (Dynamo-style)
- Chain replication

#### Synchronization Mechanisms
- Synchronous replication
- Asynchronous replication
- Semi-synchronous replication
- Quorum-based replication

#### Read Replicas
- Lag and staleness
- Read-after-write consistency
- Monotonic reads
- Replica selection strategies

#### Database Sharding Strategies
- Hash-based sharding
- Range-based sharding
- Directory-based sharding
- Composite sharding
- Geographic sharding

#### Consistent Hashing
- Basic consistent hashing
- Virtual nodes (vnodes)
- Jump consistent hash
- Rendezvous hashing (HRW)
- Bounded-load consistent hashing

#### Rebalancing Strategies
- Fixed number of partitions
- Dynamic partitioning
- Proportional to nodes
- Minimizing data movement

#### Cross-Partition Operations
- Scatter-gather queries
- Secondary indexes (local vs global)
- Cross-shard transactions
- Two-phase commit limitations

---

### **11. Distributed Transactions**

#### Two-Phase Commit (2PC)
- Prepare phase
- Commit phase
- Coordinator failure handling
- Participant failure handling
- Blocking nature and limitations

#### Three-Phase Commit (3PC)
- Pre-commit phase addition
- Non-blocking properties
- Limitations in practice

#### Saga Pattern
- Choreography-based sagas
- Orchestration-based sagas
- Compensating transactions
- Saga execution coordinator
- Isolation challenges

#### Distributed Locks
- Lock acquisition and release
- Lock expiration and renewal
- Fencing tokens
- Redlock algorithm (and controversies)
- ZooKeeper locks

#### Idempotency
- Idempotent operations definition
- Idempotency keys
- Implementing idempotency
- At-least-once with idempotency = exactly-once

#### Event Sourcing
- Event store design
- Event schema evolution
- Snapshotting strategies
- Rebuilding state from events
- Event sourcing vs CRUD

---

### **12. Message Queues & Event Streaming**

#### Message Queue Patterns
- Point-to-point (competing consumers)
- Publish-subscribe (fan-out)
- Request-reply
- Priority queues

#### Message Ordering and Delivery Guarantees
- At-most-once delivery
- At-least-once delivery
- Exactly-once delivery (and myths)
- FIFO ordering
- Partition-level ordering

#### Dead Letter Queues
- Purpose and use cases
- DLQ processing strategies
- Alerting on DLQ growth
- Reprocessing from DLQ

#### Event Streaming Platforms
- Apache Kafka architecture
- Kafka partitions and consumer groups
- Kafka Streams
- RabbitMQ patterns
- AWS SQS/SNS
- Google Pub/Sub

#### Message Durability and Persistence
- In-memory vs persistent queues
- Replication for durability
- Write-ahead logging
- Compaction strategies

#### Event Schema Management
- Schema registry
- Avro, Protobuf, JSON Schema
- Schema evolution compatibility
- Schema versioning strategies

---

### **13. Consensus Algorithms**

#### Raft Algorithm
- Leader election
- Log replication
- Safety guarantees
- Membership changes
- Raft optimizations

#### Paxos Algorithm
- Basic Paxos (single-decree)
- Multi-Paxos
- Roles: proposers, acceptors, learners
- Paxos Made Simple

#### EPaxos (Egalitarian Paxos)
- Leaderless consensus
- Commit ordering
- Geo-distributed optimization

#### Byzantine Fault Tolerance
- Byzantine generals problem
- PBFT algorithm
- BFT in blockchain
- When BFT is needed

#### Leader Election Patterns
- Bully algorithm
- Ring-based election
- Lease-based leadership
- Leadership transfer
- Split-brain prevention

#### Quorum-Based Systems
- Read and write quorums
- Quorum intersection
- Sloppy quorums
- Hinted handoff

---

### **14. Clock Synchronization & Time**

#### Logical Clocks
- Lamport timestamps
- Happens-before relation
- Vector clocks
- Version vectors
- Interval Tree Clocks

#### Hybrid Logical Clocks (HLC)
- Combining physical and logical time
- Bounded clock skew
- NTP dependency reduction
- Use in CockroachDB, MongoDB

#### Physical Clock Synchronization
- NTP (Network Time Protocol)
- PTP (Precision Time Protocol)
- Clock skew and drift
- GPS-based time synchronization

#### Google TrueTime
- Confidence intervals
- Spanner's use of TrueTime
- Wait-out uncertainty
- External consistency

#### Timestamp Ordering
- Total ordering with timestamps
- Tie-breaking strategies
- Timestamp-based concurrency control

#### Causality Tracking
- Detecting concurrent operations
- Causal consistency implementation
- Conflict detection

---

### **15. Distributed Coordination**

#### Service Discovery
- Client-side discovery
- Server-side discovery
- Service registry (Consul, etcd, ZooKeeper)
- DNS-based discovery
- Health checking and deregistration

#### Distributed Configuration Management
- Centralized configuration
- Configuration versioning
- Dynamic configuration updates
- Configuration consistency

#### Leader Election Services
- ZooKeeper recipes
- etcd leader election
- Consul leader election
- Kubernetes leader election

#### Distributed Semaphores and Barriers
- Counting semaphores
- Barriers for coordination
- Double barriers
- Implementation patterns

#### Coordination Patterns
- Distributed queues
- Distributed counters
- Group membership
- Watch and notify

---

## **TIER 3: ARCHITECTURE PATTERNS**

### **16. Microservices Architecture**

#### Microservices vs Monolith
- When to use microservices
- Migration triggers
- Team size considerations
- Complexity trade-offs

#### Service Decomposition Strategies
- By business capability
- By subdomain (DDD)
- By data ownership
- Strangler fig pattern
- Avoiding distributed monolith

#### Service Communication
- Synchronous (HTTP, gRPC)
- Asynchronous (message queues, events)
- Hybrid approaches
- Communication patterns selection

#### API Gateway Pattern
- Single entry point
- Cross-cutting concerns
- Request routing
- Protocol translation
- Rate limiting and authentication

#### Service Mesh
- Sidecar proxy pattern
- Control plane vs data plane
- Traffic management
- Security (mTLS)
- Observability

#### Data Management in Microservices
- Database per service
- Shared database (anti-pattern)
- Data consistency patterns
- Saga pattern for transactions

---

### **17. Modular Monolith Architecture**

#### When to Use Monoliths
- Early-stage startups
- Small teams
- Simple domains
- Strong consistency requirements

#### Modular Monolith Design
- Module boundaries
- Internal APIs between modules
- Shared nothing between modules
- Database schema isolation

#### Monolith Scaling Strategies
- Vertical scaling
- Horizontal scaling with session management
- Read replicas
- Caching layers

#### Migration Path to Microservices
- Identifying seams
- Strangler fig application
- Database decomposition
- Incremental extraction

---

### **18. Event-Driven Architecture**

#### Event Sourcing Deep Dive
- Events as source of truth
- Event store design
- Event schema evolution
- Snapshotting for performance
- Replaying events
- Temporal queries

#### CQRS (Command Query Responsibility Segregation)
- Separate read and write models
- Eventually consistent reads
- Projection patterns
- Combining with event sourcing

#### Event Choreography vs Orchestration
- Choreography: Events trigger reactions
- Orchestration: Central coordinator
- Hybrid approaches
- Trade-offs and selection criteria

#### Event Streaming Patterns
- Event notification
- Event-carried state transfer
- Event sourcing
- CQRS with event streaming

#### Eventual Consistency in Event-Driven Systems
- Handling out-of-order events
- Idempotent event handlers
- Compensation patterns
- User experience considerations

---

### **19. Serverless Architecture**

#### Function-as-a-Service (FaaS)
- AWS Lambda, Azure Functions, Google Cloud Functions
- Function lifecycle
- Stateless execution
- Triggering mechanisms

#### Serverless Patterns
- API backends
- Event processing
- Scheduled tasks
- Stream processing

#### Cold Start Optimization
- Provisioned concurrency
- Function warming
- Package size optimization
- Runtime selection

#### Serverless Limitations
- Execution time limits
- State management
- Vendor lock-in
- Cold start latency
- Debugging complexity

#### Serverless Databases
- DynamoDB on-demand
- Aurora Serverless
- PlanetScale
- Neon (serverless Postgres)

#### Cost Optimization
- Right-sizing memory
- Execution time optimization
- Avoiding unnecessary invocations
- Reserved concurrency

---

### **20. Layered Architecture**

#### N-Tier Architecture
- Presentation layer
- Business logic layer
- Data access layer
- Database layer

#### Separation of Concerns
- Single responsibility
- Layer dependencies
- Cross-cutting concerns
- Dependency injection

#### Anti-Patterns
- Leaky abstractions
- Anemic domain model
- Big ball of mud
- Circular dependencies

---

### **21. Domain-Driven Design (DDD)**

#### Strategic Design
- Bounded contexts
- Context mapping
- Ubiquitous language
- Subdomains (core, supporting, generic)

#### Tactical Design
- Entities and value objects
- Aggregates and aggregate roots
- Domain events
- Repositories
- Domain services
- Factories

#### Anti-Corruption Layers
- Translating between contexts
- Protecting domain model
- Adapter patterns

#### Event Storming
- Discovery workshops
- Event identification
- Command and aggregate discovery
- Context boundary identification

---

### **22. Hexagonal/Ports & Adapters Architecture**

#### Clean Architecture Principles
- Dependency rule
- Entities at center
- Use cases layer
- Interface adapters
- Frameworks and drivers

#### Ports and Adapters
- Primary/driving ports
- Secondary/driven ports
- Adapters implementation
- Dependency inversion

#### Testability Patterns
- Testing with fake adapters
- Integration testing boundaries
- Contract testing

#### Core Domain Isolation
- Business logic isolation
- Framework independence
- Database independence

---

### **23. Strangler Fig Pattern**

#### Legacy System Modernization
- Incremental replacement
- Risk mitigation
- Business continuity

#### Implementation Strategies
- Routing layer (facade)
- Event interception
- Database decomposition
- API versioning

#### Parallel Run Patterns
- Shadow traffic
- Comparison testing
- Gradual cutover
- Rollback strategies

---

## **TIER 4: DATABASE & STORAGE SYSTEMS**

### **24. SQL Database Internals**

#### Storage Engines
- B-tree structure and operations
- LSM-tree (Log-Structured Merge-tree)
- B-tree vs LSM-tree trade-offs
- Page structure and buffer pool
- Write-ahead logging (WAL)

#### MVCC (Multi-Version Concurrency Control)
- Version chains
- Snapshot isolation
- Garbage collection
- Write skew anomaly

#### Transaction Processing
- Transaction lifecycle
- Lock management
- Deadlock detection
- Two-phase locking (2PL)
- Optimistic concurrency control

#### Isolation Levels
- Read Uncommitted
- Read Committed
- Repeatable Read
- Serializable
- Snapshot Isolation
- Trade-offs and selection

#### Connection Management
- Connection pooling (PgBouncer, ProxySQL)
- Pool sizing calculations
- Connection lifecycle
- Connection draining

---

### **25. NoSQL Databases - Document Stores**

#### MongoDB Patterns
- Document modeling
- Embedding vs referencing
- Schema design patterns
- Indexes and query optimization

#### Document Modeling
- One-to-one: Embed
- One-to-many: Embed or reference
- Many-to-many: Reference
- Denormalization strategies

#### Sharding Strategies
- Shard key selection
- Range vs hash sharding
- Zone sharding
- Balancing and chunks

#### Use Cases
- Content management
- User profiles
- Product catalogs
- Real-time analytics

---

### **26. NoSQL Databases - Key-Value Stores**

#### Redis Patterns
- Data structures (strings, lists, sets, hashes, sorted sets)
- Pub/Sub
- Lua scripting
- Redis Cluster
- Persistence (RDB, AOF)

#### DynamoDB Patterns
- Single-table design
- Partition key and sort key
- GSI and LSI
- Capacity modes
- Transactions

#### Key Design Strategies
- Key naming conventions
- Key distribution
- Avoiding hot keys
- TTL and expiration

#### Use Cases
- Session storage
- Caching
- Rate limiting
- Leaderboards
- Real-time inventory

---

### **27. NoSQL Databases - Wide-Column Stores**

#### Cassandra Patterns
- Partition key design
- Clustering columns
- Materialized views
- Lightweight transactions

#### Data Modeling
- Query-driven design
- Denormalization
- Time-series patterns
- Bucketing strategies

#### Partitioning Strategies
- Avoiding hot partitions
- Partition size limits
- Compound partition keys

#### Compaction Strategies
- Size-tiered compaction
- Leveled compaction
- Time-window compaction

#### Use Cases
- Time-series data
- IoT sensor data
- User activity logs
- Messaging systems

---

### **28. NoSQL Databases - Graph Databases**

#### Neo4j Patterns
- Property graph model
- Cypher query language
- Index-free adjacency
- Graph algorithms

#### Graph Modeling
- Nodes and relationships
- Properties
- Labels and types
- Graph schema design

#### Traversal Algorithms
- Breadth-first search
- Depth-first search
- Shortest path
- PageRank
- Community detection

#### Use Cases
- Social networks
- Recommendation engines
- Fraud detection
- Knowledge graphs
- Network topology

#### When NOT to Use Graph Databases
- Simple key-value access
- High-volume writes
- Full-text search
- OLAP workloads

---

### **29. Time-Series Databases**

#### InfluxDB/TimescaleDB Patterns
- Time-series data modeling
- Tags vs fields
- Retention policies
- Continuous queries

#### Data Modeling
- Measurement/metric naming
- Tag cardinality management
- Downsampling strategies
- Aggregation functions

#### Retention and Compression
- Retention policies
- Compression algorithms
- Tiered storage
- Data lifecycle management

#### Use Cases
- Monitoring and metrics
- IoT sensor data
- Financial data
- Log analytics

---

### **30. Search Engines**

#### Elasticsearch Patterns
- Index design
- Mapping and analyzers
- Query DSL
- Aggregations

#### Full-Text Search
- Inverted index
- Tokenization and analysis
- Relevance scoring (BM25)
- Fuzzy matching

#### Indexing Strategies
- Index per time period
- Index aliases
- Reindexing strategies
- Index lifecycle management

#### Search Relevance
- Boosting and scoring
- Function score queries
- Learning to rank
- A/B testing search

#### Use Cases
- Product search
- Log analysis
- Content discovery
- Autocomplete

---

### **31. Vector Databases (2024-2026 Critical)**

#### Vector Database Fundamentals
- Embedding vectors
- Similarity search
- Distance metrics (cosine, euclidean, dot product)
- Approximate nearest neighbor (ANN)

#### Vector Database Options
- Pinecone: Managed, scalable
- Weaviate: Open-source, hybrid search
- Milvus: Open-source, scalable
- pgvector: PostgreSQL extension
- Qdrant: Rust-based, performant

#### Indexing Algorithms
- HNSW (Hierarchical Navigable Small World)
- IVF (Inverted File Index)
- PQ (Product Quantization)
- Trade-offs: Speed vs recall vs memory

#### Use Cases
- Semantic search
- RAG (Retrieval-Augmented Generation)
- Recommendation systems
- Image similarity
- Anomaly detection

---

### **32. Database Migration & Schema Evolution**

#### Online Schema Changes
- pt-online-schema-change
- gh-ost
- Native online DDL
- Trade-offs and risks

#### Zero-Downtime Migrations
- Expand-contract pattern
- Dual writes
- Background migration
- Cutover strategies

#### Backward Compatibility
- Adding columns (nullable or with defaults)
- Removing columns (deprecate first)
- Renaming (add new, migrate, remove old)
- Type changes

#### Migration Testing
- Schema migration testing
- Data migration testing
- Rollback testing
- Performance testing

---

### **33. Multi-Model and Polyglot Persistence**

#### Polyglot Persistence
- Right database for the job
- Data synchronization
- Consistency challenges
- Operational complexity

#### Database per Service Pattern
- Service data ownership
- Cross-service queries
- Data duplication
- Event-driven sync

#### Cross-Database Transactions
- Saga pattern
- Eventual consistency
- Compensating transactions
- Outbox pattern

---

### **34. Object Storage**

#### S3/Blob Storage Patterns
- Object naming conventions
- Prefix design for performance
- Multipart uploads
- Versioning

#### Object Lifecycle Management
- Lifecycle policies
- Storage classes/tiers
- Intelligent tiering
- Glacier/Archive access

#### Access Patterns
- Pre-signed URLs
- CloudFront/CDN integration
- S3 Select for querying
- Event notifications

---

### **35. Data Lakes & Data Warehousing**

#### OLTP vs OLAP
- Transaction processing characteristics
- Analytical processing characteristics
- Data flow between systems

#### Lakehouse Architecture (Modern)
- Delta Lake
- Apache Iceberg
- Apache Hudi
- ACID on data lakes
- Time travel queries

#### ETL/ELT Pipelines
- Extract, Transform, Load
- Extract, Load, Transform
- Streaming ETL
- Data quality checks

#### Data Warehouse Design
- Star schema
- Snowflake schema
- Slowly changing dimensions
- Fact and dimension tables

#### Real-Time OLAP (2024-2026)
- ClickHouse
- Apache Druid
- Apache Pinot
- Use cases and trade-offs

---

## **TIER 5: API DESIGN & COMMUNICATION**

### **36. RESTful API Design**

#### REST Principles
- Resource-based design
- Statelessness
- Uniform interface
- HATEOAS

#### Resource Modeling
- Noun-based URLs
- Hierarchical resources
- Collection vs item resources
- Resource relationships

#### HTTP Methods and Status Codes
- GET, POST, PUT, PATCH, DELETE
- Idempotency of methods
- 2xx, 3xx, 4xx, 5xx codes
- Proper status code selection

#### API Versioning Strategies
- URL versioning (/v1/)
- Header versioning
- Query parameter versioning
- Content negotiation

---

### **37. GraphQL**

#### GraphQL Fundamentals
- Schema definition language
- Types and fields
- Queries and mutations
- Subscriptions

#### GraphQL vs REST
- Over-fetching and under-fetching
- Single endpoint
- Strong typing
- Introspection

#### GraphQL at Scale
- Schema stitching
- Federation
- Persisted queries
- Query complexity analysis

#### N+1 Problem and Solutions
- DataLoader pattern
- Batching and caching
- Query optimization

#### Security Considerations
- Depth limiting
- Query complexity limits
- Rate limiting
- Authentication and authorization

---

### **38. gRPC**

#### Protocol Buffers
- Message definition
- Backward compatibility
- Code generation
- Efficient serialization

#### gRPC Streaming
- Unary RPC
- Server streaming
- Client streaming
- Bidirectional streaming

#### gRPC vs REST
- Performance comparison
- Use case selection
- Browser support (gRPC-Web)
- Tooling ecosystem

#### Load Balancing gRPC
- Client-side load balancing
- Proxy-based load balancing
- Service mesh integration

---

### **39. API Gateway Pattern**

#### Gateway Responsibilities
- Request routing
- Protocol translation
- Authentication/authorization
- Rate limiting
- Request/response transformation
- Caching

#### Gateway Options
- Kong, Ambassador, Traefik
- AWS API Gateway
- Azure API Management
- Cloud-native gateways

#### Gateway Patterns
- Edge gateway
- Internal gateway
- BFF (Backend for Frontend)

---

### **40. Service-to-Service Communication**

#### Synchronous Communication
- HTTP/REST
- gRPC
- GraphQL
- Trade-offs

#### Asynchronous Communication
- Message queues
- Event streaming
- Pub/Sub patterns
- When to use async

#### Resilience Patterns
- Circuit breaker
- Retry with backoff
- Timeout handling
- Bulkhead isolation

---

### **41. WebSockets & Real-time Communication**

#### WebSocket Protocol
- Connection lifecycle
- Frame types
- Heartbeats and keepalive
- Scaling challenges

#### Server-Sent Events (SSE)
- Unidirectional streaming
- Reconnection handling
- Event types

#### Long Polling
- Implementation pattern
- Timeout handling
- Scaling considerations

#### Real-time Scaling Strategies
- Connection management
- Pub/Sub backends
- Sticky sessions
- Horizontal scaling

---

### **42. Async APIs (2024-2026)**

#### AsyncAPI Specification
- Event-driven API documentation
- Channel and message definitions
- Protocol bindings
- Code generation

#### Webhook Design Patterns
- Callback registration
- Payload design
- Retry strategies
- Security (signatures, mTLS)
- Idempotency

---

### **43. BFF (Backend for Frontend) Pattern**

#### Client-Specific APIs
- Mobile BFF
- Web BFF
- Third-party BFF

#### Aggregation Patterns
- Data aggregation
- API composition
- Response shaping

#### Maintenance Considerations
- Code duplication
- Team ownership
- Versioning per client

---

## **TIER 6: RELIABILITY & FAULT TOLERANCE**

### **44. High Availability (HA)**

#### Redundancy Strategies
- N+1 redundancy
- N+2 redundancy
- 2N redundancy
- Geographic redundancy

#### Active-Passive vs Active-Active
- Failover mechanisms
- Data synchronization
- DNS switching
- Load balancer failover

#### Health Checks and Monitoring
- Liveness checks
- Readiness checks
- Deep health checks
- Synthetic monitoring

#### Availability Calculations
- Uptime percentage
- SLA calculations
- Nines of availability
- MTBF and MTTR

---

### **45. Fault Tolerance Patterns**

#### Circuit Breaker Pattern
- Closed state (normal)
- Open state (failing fast)
- Half-open state (testing)
- Failure threshold configuration
- Recovery strategies

#### Bulkhead Pattern
- Thread pool isolation
- Semaphore bulkheads
- Resource partitioning
- Blast radius containment

#### Retry Strategies
- Immediate retry
- Fixed interval retry
- Exponential backoff
- Exponential backoff with jitter
- Retry budgets

#### Timeout Patterns
- Connection timeout
- Read timeout
- Write timeout
- Request timeout
- Deadline propagation

#### Graceful Degradation
- Feature degradation
- Fallback responses
- Cached responses
- Static fallbacks

---

### **46. Rate Limiting & Throttling**

#### Rate Limiting Algorithms
- Token bucket (allows bursts)
- Leaky bucket (smooth rate)
- Fixed window (simple, bursty edges)
- Sliding window log
- Sliding window counter

#### Distributed Rate Limiting
- Central rate limiter (Redis)
- Local + global limits
- Eventual consistency
- Synchronization lag handling

#### Rate Limiting Strategies
- Per user/API key
- Per IP address
- Per endpoint
- Tiered limits
- Adaptive limits

---

### **47. Idempotency & Exactly-Once Processing**

#### Idempotent Operations
- GET, PUT, DELETE are idempotent
- POST requires explicit handling
- Mathematical idempotency

#### Idempotency Keys
- Client-generated keys
- Key storage and lookup
- Key expiration
- Response caching

#### Exactly-Once Semantics
- At-least-once + idempotency
- Transactional outbox
- Deduplication strategies

---

### **48. Load Shedding**

#### Load Shedding Strategies
- Random shedding
- Priority-based shedding
- Age-based shedding (LIFO)
- Cost-based shedding

#### Overload Detection
- Queue depth monitoring
- Latency thresholds
- Error rate thresholds
- Resource utilization

#### Graceful Overload Handling
- Returning cached data
- Simplified responses
- Retry-After headers
- Client backoff signaling

---

### **49. Chaos Engineering**

#### Chaos Principles
- Build hypothesis around steady state
- Vary real-world events
- Run experiments in production
- Automate to run continuously
- Minimize blast radius

#### Failure Injection Types
- Service failure
- Network latency
- Network partition
- Resource exhaustion
- Dependency failure

#### Chaos Tools
- Chaos Monkey (Netflix)
- Gremlin
- LitmusChaos
- Chaos Mesh

#### Game Days
- Planned failure exercises
- Cross-team coordination
- Runbook validation
- Learning and improvement

---

### **50. Deployment Strategies**

#### Blue-Green Deployment
- Two production environments
- Instant switchover
- Easy rollback
- Resource overhead

#### Canary Deployment
- Gradual traffic shift
- Monitoring and validation
- Automatic rollback
- Progressive delivery

#### Rolling Deployment
- Gradual instance updates
- Zero downtime
- Mixed versions temporarily
- Rollback complexity

#### Feature Flags
- Deployment vs release separation
- Gradual rollout
- A/B testing
- Kill switches

---

### **51. Disaster Recovery (DR)**

#### Recovery Objectives
- RPO: Recovery Point Objective (data loss tolerance)
- RTO: Recovery Time Objective (downtime tolerance)
- Cost vs recovery speed trade-offs

#### Backup Strategies
- Full backups
- Incremental backups
- Differential backups
- Point-in-time recovery

#### DR Patterns
- Backup and restore
- Pilot light
- Warm standby
- Multi-site active-active

#### DR Testing
- Tabletop exercises
- Failover drills
- Full DR tests
- Chaos engineering

---

### **52. Graceful Shutdown & Startup**

#### Graceful Shutdown
- SIGTERM handling
- Connection draining
- In-flight request completion
- Resource cleanup

#### Health Check Integration
- Failing health checks before shutdown
- Deregistration from load balancer
- Drain period

#### Startup Patterns
- Dependency checking
- Warm-up period
- Gradual traffic ramp
- Readiness signaling

---

## **TIER 7: PERFORMANCE & OPTIMIZATION**

### **53. Latency Optimization**

#### Latency Analysis
- P50, P95, P99, P99.9 percentiles
- Latency distribution analysis
- Tail latency causes
- Coordinated omission problem

#### Optimization Techniques
- Reducing network round trips
- Connection reuse (keep-alive)
- Request batching and coalescing
- Prefetching
- Async operations

#### CDN and Edge
- Static content caching
- Dynamic content at edge
- Edge computing
- Geographic distribution

---

### **54. Throughput Optimization**

#### Horizontal Scaling
- Stateless service scaling
- Auto-scaling policies
- Load balancer configuration

#### Async Processing
- Background jobs
- Message queues
- Event-driven processing

#### Batch Processing
- Batch size optimization
- Parallel processing
- MapReduce patterns

#### Resource Optimization
- Connection pooling
- Thread pool tuning
- Memory management
- I/O optimization

---

### **55. Database Performance**

#### Query Optimization
- EXPLAIN plan analysis
- Index selection and creation
- Query rewriting
- Avoiding full table scans
- Join optimization

#### Connection Management
- Connection pool sizing
- Connection lifecycle
- Prepared statements
- Connection health checks

#### Read Optimization
- Read replicas
- Caching strategies
- Denormalization
- Materialized views

#### Write Optimization
- Batch writes
- Async writes
- Write-behind caching
- Partitioning for write distribution

---

### **56. Caching Performance**

#### Cache Hit Ratio
- Monitoring hit ratios
- Identifying cache misses
- Cache warming

#### Eviction Policies
- LRU (Least Recently Used)
- LFU (Least Frequently Used)
- ARC (Adaptive Replacement Cache)
- FIFO

#### Multi-Tier Caching
- L1: In-process cache
- L2: Distributed cache
- L3: CDN/Edge cache
- Cache coherency

---

### **57. Memory Management**

#### Memory Profiling
- Heap analysis
- Memory leak detection
- Object allocation tracking

#### Garbage Collection Tuning
- GC algorithm selection
- Heap sizing
- GC pause optimization
- GC logging and analysis

#### Off-Heap Storage
- Direct memory allocation
- Memory-mapped files
- Off-heap caches

---

### **58. Compression & Serialization**

#### Compression Algorithms
- Gzip/Deflate
- LZ4 (fast)
- Zstd (balanced)
- Brotli (high ratio)
- Trade-offs: Speed vs ratio

#### Serialization Formats
- JSON (human-readable, verbose)
- Protocol Buffers (efficient, typed)
- Avro (schema evolution)
- MessagePack (binary JSON)
- Comparison and selection

---

### **59. CDN & Edge Computing**

#### CDN Architecture
- Points of Presence (PoPs)
- Origin servers
- Edge servers
- Cache hierarchy

#### CDN Strategies
- Static content caching
- Dynamic content acceleration
- Origin shield
- Cache key design

#### Edge Computing (2024-2026)
- Cloudflare Workers
- Lambda@Edge
- Fastly Compute
- Use cases and limitations

---

## **TIER 8: SECURITY & AUTHENTICATION**

### **60. Authentication**

#### OAuth 2.0 / OpenID Connect
- Authorization code flow
- Client credentials flow
- PKCE for public clients
- Token types (access, refresh, ID)

#### JWT (JSON Web Tokens)
- Structure (header, payload, signature)
- Signing algorithms
- Token validation
- Token revocation challenges

#### Session Management
- Session storage
- Session expiration
- Session fixation prevention
- Distributed sessions

#### Passkeys/WebAuthn (2024-2026)
- FIDO2 standard
- Public key authentication
- Passwordless authentication
- Platform authenticators

#### Multi-Factor Authentication
- Something you know
- Something you have
- Something you are
- TOTP, WebAuthn, SMS

---

### **61. Authorization**

#### RBAC (Role-Based Access Control)
- Roles and permissions
- Role hierarchy
- Permission assignment

#### ABAC (Attribute-Based Access Control)
- Attribute-based policies
- Context-aware authorization
- Fine-grained control

#### ReBAC (Relationship-Based Access Control)
- Relationship modeling
- Graph-based authorization
- Zanzibar-style systems

#### Policy Engines
- Open Policy Agent (OPA)
- AWS Cedar
- Policy as code
- Centralized authorization

---

### **62. Security Best Practices**

#### Input Validation
- Whitelist validation
- Input sanitization
- Type coercion
- Length limits

#### Injection Prevention
- SQL injection
- NoSQL injection
- Command injection
- Template injection

#### XSS Prevention
- Output encoding
- Content Security Policy
- HTTPOnly cookies
- Sanitization libraries

#### CSRF Protection
- CSRF tokens
- SameSite cookies
- Double submit cookies

---

### **63. Encryption & Data Protection**

#### Encryption at Rest
- Transparent data encryption
- Application-level encryption
- Key management
- Encrypted backups

#### Encryption in Transit
- TLS 1.3
- Certificate management
- mTLS for service-to-service
- Certificate rotation

#### Key Management
- Key rotation
- Key hierarchy
- HSM (Hardware Security Modules)
- Cloud KMS (AWS KMS, GCP KMS)

#### Data Protection
- PII identification
- Data masking
- Tokenization
- Anonymization vs pseudonymization

---

### **64. API Security**

#### API Authentication
- API keys
- OAuth for APIs
- JWT validation
- mTLS

#### Rate Limiting for Security
- Brute force prevention
- DDoS mitigation
- Per-user limits

#### API Security Monitoring
- Anomaly detection
- Abuse detection
- Security logging

---

### **65. Zero Trust Architecture**

#### Zero Trust Principles
- Never trust, always verify
- Assume breach
- Verify explicitly
- Least privilege access

#### Implementation
- Identity verification
- Device verification
- Micro-segmentation
- Continuous validation

---

### **66. Secrets Management**

#### Secret Storage
- HashiCorp Vault
- AWS Secrets Manager
- GCP Secret Manager
- Azure Key Vault

#### Secret Injection
- Environment variables
- Sidecar containers
- Init containers
- CSI secrets driver

#### Secret Rotation
- Automatic rotation
- Rotation workflows
- Zero-downtime rotation

---

### **67. Supply Chain Security (2024-2026)**

#### SBOM (Software Bill of Materials)
- Dependency tracking
- Vulnerability scanning
- License compliance

#### SLSA Framework
- Build integrity
- Source integrity
- Provenance tracking
- Supply chain levels

#### Container Security
- Image signing (Sigstore, Cosign)
- Image scanning
- Base image management
- Runtime security

---

### **68. Compliance & Regulatory**

#### GDPR Compliance
- Data subject rights
- Consent management
- Data processing agreements
- Cross-border transfers

#### Data Residency
- Geographic restrictions
- Data sovereignty
- Multi-region architecture

#### Audit Logging
- Audit trail requirements
- Log integrity
- Retention requirements
- Access logging

---

## **TIER 9: OBSERVABILITY & MONITORING**

### **69. Monitoring & Metrics**

#### Metric Types
- Counters (monotonic)
- Gauges (point-in-time)
- Histograms (distributions)
- Summaries (quantiles)

#### Monitoring Methods
- RED method (Rate, Errors, Duration)
- USE method (Utilization, Saturation, Errors)
- Golden signals (latency, traffic, errors, saturation)

#### Alerting Best Practices
- Alert on symptoms, not causes
- Actionable alerts
- Alert fatigue prevention
- Escalation policies

#### Observability Pillars
- Metrics
- Logs
- Traces
- (Profiles - 4th pillar)

---

### **70. Logging Strategies**

#### Structured Logging
- JSON logging
- Consistent field names
- Contextual information
- Correlation IDs

#### Log Aggregation
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Grafana Loki
- Splunk
- Cloud logging services

#### Log Levels
- DEBUG, INFO, WARN, ERROR, FATAL
- Level selection guidelines
- Dynamic log levels

#### Log Retention
- Retention policies
- Storage tiering
- Compliance requirements
- Cost management

---

### **71. Distributed Tracing**

#### OpenTelemetry
- Traces, spans, and context
- Auto-instrumentation
- Manual instrumentation
- Exporters and collectors

#### Trace Context Propagation
- W3C Trace Context
- B3 propagation
- Cross-service correlation

#### Span Design
- Span naming conventions
- Attributes and events
- Span status
- Span links

#### Tracing Analysis
- Service maps
- Critical path analysis
- Error tracking
- Performance analysis

---

### **72. Continuous Profiling (2024-2026)**

#### Profiling Types
- CPU profiling
- Memory profiling
- Allocation profiling
- Lock contention profiling

#### Always-On Profiling
- Low-overhead sampling
- Production profiling
- Historical analysis
- Flame graph analysis

#### Profiling Tools
- Pyroscope
- Parca
- Google Cloud Profiler
- Datadog Continuous Profiler

---

### **73. Health Checks**

#### Health Check Types
- Liveness: Is the process alive?
- Readiness: Can it handle traffic?
- Startup: Has it initialized?

#### Health Check Design
- Dependency checks
- Timeout configuration
- Failure thresholds
- Health check endpoints

---

### **74. SLI/SLO/SLA**

#### SLI (Service Level Indicator)
- Availability SLI
- Latency SLI
- Throughput SLI
- Error rate SLI
- Correctness SLI

#### SLO (Service Level Objective)
- Setting appropriate targets
- SLO negotiation
- SLO documentation

#### Error Budgets
- Error budget calculation
- Error budget policies
- Burn rate alerting
- SLO-based decision making

#### SLA (Service Level Agreement)
- Contractual commitments
- Penalties and credits
- SLA vs SLO relationship

---

### **75. Incident Management**

#### Incident Response
- Detection and triage
- Communication
- Mitigation
- Resolution
- Post-incident review

#### On-Call Practices
- On-call rotations
- Escalation policies
- Handoff procedures
- On-call compensation

#### Post-Mortems
- Blameless culture
- Timeline reconstruction
- Root cause analysis
- Action items tracking

#### Runbooks
- Runbook creation
- Runbook automation
- Runbook testing
- Knowledge sharing

---

### **76. Capacity Planning**

#### Resource Forecasting
- Historical analysis
- Growth projections
- Seasonal patterns
- Event-based spikes

#### Load Testing
- Load testing strategies
- Performance baselines
- Capacity limits
- Stress testing

#### Cost Optimization
- Right-sizing
- Reserved capacity
- Spot instances
- Auto-scaling efficiency

---

## **TIER 10: SPECIALIZED PATTERNS**

### **77. Stream Processing**

#### Stream vs Batch Processing
- Real-time requirements
- Data freshness
- Processing complexity
- Resource utilization

#### Stream Processing Frameworks
- Apache Kafka Streams
- Apache Flink
- Apache Spark Streaming
- Cloud streaming services

#### Windowing Strategies
- Tumbling windows
- Sliding windows
- Session windows
- Global windows

#### Stateful Stream Processing
- State management
- Checkpointing
- Exactly-once processing
- State recovery

---

### **78. Data Consistency Patterns**

#### Optimistic Concurrency Control
- Version numbers
- Compare-and-swap
- Conflict detection
- Retry strategies

#### Pessimistic Locking
- Exclusive locks
- Shared locks
- Lock granularity
- Deadlock prevention

#### MVCC Deep Dive
- Version management
- Garbage collection
- Snapshot isolation
- Write conflicts

#### Conflict Resolution
- Last-write-wins
- Application-specific merge
- CRDTs
- Operational transformation

---

### **79. Multi-Tenancy Patterns**

#### Tenant Isolation Strategies
- Shared nothing (instance per tenant)
- Shared everything (logical isolation)
- Hybrid approaches

#### Database Strategies
- Database per tenant
- Schema per tenant
- Shared schema with tenant ID

#### Noisy Neighbor Problem
- Resource limits per tenant
- Fair scheduling
- Throttling
- Isolation techniques

---

### **80. Feature Flags & Configuration**

#### Feature Flag Patterns
- Release flags
- Experiment flags
- Ops flags
- Permission flags

#### Feature Flag Systems
- LaunchDarkly
- Split.io
- Unleash
- Custom implementations

#### Configuration Management
- Configuration as code
- Dynamic configuration
- Configuration versioning
- Configuration validation

---

### **81. Anti-Patterns**

#### Distributed Monolith
- Symptoms
- Causes
- Solutions

#### Chatty Services
- Too many calls
- N+1 patterns
- Batching solutions

#### God Services
- Too many responsibilities
- Decomposition strategies

#### Shared Database
- Coupling problems
- Migration strategies

#### Premature Optimization
- When to optimize
- Measurement first
- Simple solutions

---

### **82. Cost Optimization (FinOps)**

#### Cloud Cost Management
- Resource tagging
- Cost allocation
- Budget alerts
- Cost anomaly detection

#### Resource Optimization
- Right-sizing instances
- Reserved vs on-demand
- Spot instances
- Serverless economics

#### Storage Cost Optimization
- Storage tiering
- Lifecycle policies
- Compression
- Deduplication

#### Data Transfer Costs
- Cross-region transfer
- CDN optimization
- Private connectivity

---

### **83. ML System Design (2024-2026)**

#### ML Model Serving
- Online inference
- Batch inference
- Model versioning
- A/B testing models

#### LLM Infrastructure
- RAG architecture
- Vector databases
- Embedding pipelines
- Prompt management
- Token management
- LLM caching

#### Feature Stores
- Online feature serving
- Offline feature serving
- Feature freshness
- Point-in-time correctness

#### MLOps
- Model training pipelines
- Model registry
- Model monitoring
- Model retraining

---

## **TIER 11: ORGANIZATIONAL & OPERATIONAL**

### **84. Team Topologies & Conway's Law**

#### Conway's Law
- Architecture reflects organization
- Inverse Conway maneuver
- Communication patterns

#### Team Types
- Stream-aligned teams
- Platform teams
- Enabling teams
- Complicated-subsystem teams

#### Cognitive Load
- Intrinsic cognitive load
- Extraneous cognitive load
- Team boundaries
- Domain complexity

---

### **85. DevOps & CI/CD**

#### CI/CD Practices
- Continuous integration
- Continuous delivery
- Continuous deployment
- Pipeline optimization

#### Infrastructure as Code
- Terraform
- Pulumi
- CloudFormation
- GitOps practices

#### GitOps
- Declarative infrastructure
- Git as source of truth
- Automated reconciliation
- ArgoCD, Flux

---

### **86. Container Orchestration**

#### Kubernetes Architecture
- Control plane components
- Node components
- Networking model
- Storage model

#### Kubernetes Patterns
- Sidecar
- Ambassador
- Adapter
- Init container

#### Kubernetes Operations
- Resource management
- Auto-scaling (HPA, VPA, Cluster)
- Network policies
- Security contexts

#### Service Mesh
- Istio architecture
- Linkerd architecture
- Cilium (eBPF-based)
- Ambient mesh (sidecar-less)

---

### **87. Platform Engineering (2024-2026)**

#### Internal Developer Platforms
- Self-service infrastructure
- Golden paths
- Developer portals
- Platform as a product

#### Developer Experience
- Inner loop optimization
- Development environments
- Preview environments
- Documentation and guides

#### Platform Tools
- Backstage
- Port
- Humanitec
- Custom platforms

---

### **88. Technical Debt Management**

#### Debt Identification
- Code quality metrics
- Architecture reviews
- Team feedback
- Incident analysis

#### Prioritization
- Business impact
- Development velocity impact
- Risk assessment
- Effort estimation

#### Paydown Strategies
- Dedicated time allocation
- Boy scout rule
- Refactoring sprints
- Rewrite vs refactor

---

### **89. Architecture Decision Records (ADRs)**

#### ADR Structure
- Title
- Status
- Context
- Decision
- Consequences

#### ADR Templates
- MADR (Markdown ADR)
- Y-statements
- Custom templates

#### ADR Practices
- When to write ADRs
- Review process
- ADR lifecycle
- Knowledge sharing

---

### **90. System Design Interview Practice**

#### Interview Framework
1. Requirements clarification (5 min)
2. Capacity estimation (5 min)
3. High-level design (10 min)
4. Component deep dive (15 min)
5. Trade-offs discussion (5-10 min)

#### Common Design Problems
- URL Shortener
- Rate Limiter
- Notification System
- News Feed / Timeline
- Chat System
- Video Streaming (YouTube/Netflix)
- Search Autocomplete
- Distributed Cache
- Job Scheduler
- Payment System
- Ride-sharing
- Hotel Booking
- Stock Exchange
- Web Crawler
- Social Network
- File Storage (Dropbox/Google Drive)
- Typeahead Suggestion
- Location-based Service
- Metrics Monitoring

---

## **TIER 12: CROSS-CUTTING CONCERNS**

### **91. Data Privacy & Governance**

#### Data Classification
- Public, internal, confidential, restricted
- Classification automation
- Handling policies

#### Privacy by Design
- Data minimization
- Purpose limitation
- Consent management
- Privacy impact assessments

#### Data Governance
- Data lineage tracking
- Data catalogs
- Data quality management
- Data stewardship

#### Right to Be Forgotten
- Data deletion workflows
- Cascading deletes
- Backup handling
- Audit compliance

---

### **92. Testing Strategies**

#### Testing Pyramid
- Unit tests
- Integration tests
- End-to-end tests
- Contract tests

#### Advanced Testing
- Chaos testing
- Performance testing
- Security testing
- Shadow traffic testing

#### Contract Testing
- Consumer-driven contracts
- Pact
- Spring Cloud Contract

---

### **93. Networking Fundamentals**

#### TCP/IP
- TCP vs UDP
- Connection lifecycle
- Flow control
- Congestion control

#### DNS
- DNS resolution
- DNS caching
- DNS-based load balancing
- DNS failover

#### HTTP Evolution
- HTTP/1.1 limitations
- HTTP/2 multiplexing
- HTTP/3 and QUIC
- gRPC over HTTP/2

#### Service Mesh Networking
- Sidecar proxies
- Traffic management
- mTLS
- Network policies

#### eBPF (2024-2026)
- Kernel-level observability
- Network filtering
- Security enforcement
- Cilium and eBPF

---

### **94. Queue Design Patterns**

#### Queue Types
- Standard queues
- FIFO queues
- Priority queues
- Delay queues

#### Queue Patterns
- Competing consumers
- Fan-out
- Request-reply
- Dead letter handling

#### Queue Monitoring
- Queue depth
- Processing latency
- Error rates
- Consumer lag

---

### **95. Data Pipeline Patterns**

#### Batch Processing
- MapReduce
- Spark batch
- Scheduled jobs

#### Stream Processing
- Real-time pipelines
- Micro-batch
- Continuous processing

#### Lambda Architecture
- Batch layer
- Speed layer
- Serving layer
- Complexity trade-offs

#### Kappa Architecture
- Stream-only processing
- Reprocessing from logs
- Simpler architecture

#### Data Mesh (2024-2026)
- Domain-oriented data ownership
- Data as a product
- Self-serve data platform
- Federated governance

---

## **MASTERY PATH RECOMMENDATIONS**

### **Phase 1: Core Foundations (Must Know Cold)**
1. Distributed consensus (Raft, Paxos)
2. Consistency models (linearizability through eventual)
3. Replication and partitioning strategies
4. CAP/PACELC trade-offs
5. Back-of-envelope calculations

### **Phase 2: Data Layer Mastery**
6. SQL internals (transactions, indexing, query optimization)
7. NoSQL selection and modeling
8. Caching patterns and invalidation
9. Event sourcing and CQRS
10. Data pipeline architectures

### **Phase 3: Reliability & Performance**
11. Fault tolerance patterns (circuit breaker, bulkhead, retry)
12. Rate limiting and load shedding
13. Latency optimization
14. SLI/SLO/Error budgets
15. Chaos engineering

### **Phase 4: Modern Architecture**
16. Service mesh and API gateway patterns
17. Event-driven architecture
18. Kubernetes deep dive
19. Observability (traces, metrics, logs, profiling)
20. Security architecture (zero trust, secrets management)

### **Phase 5: Staff-Level Differentiation**
21. Platform engineering
22. ML/LLM infrastructure
23. Architecture decision-making
24. Technical debt and migration strategies
25. Organizational patterns (Conway's Law, team topologies)

---

## **INTERVIEW PREPARATION PRIORITY**

### **Critical (Must Master)**
- Tiers 1-6: Foundations through Reliability
- Topic 90: System Design Interview Practice

### **Important (Should Know Well)**
- Tiers 7-9: Performance, Security, Observability

### **Differentiating (Staff+ Level)**
- Tiers 10-12: Specialized patterns, Organizational, Cross-cutting

---

## **2024-2026 CRITICAL ADDITIONS**

| Topic | Why Critical |
|-------|--------------|
| **CRDTs** | Collaborative apps, offline-first |
| **Hybrid Logical Clocks** | Modern distributed time |
| **LLM/RAG Infrastructure** | AI-native applications |
| **Vector Databases** | Semantic search everywhere |
| **Platform Engineering** | Staff engineers build platforms |
| **eBPF** | Modern observability/networking |
| **Ambient Service Mesh** | Sidecar-less future |
| **Passkeys/WebAuthn** | Password replacement |
| **Supply Chain Security** | SBOM, SLSA mandatory |
| **Data Mesh** | Data ownership patterns |
| **Lakehouse Architecture** | OLAP evolution |
| **Continuous Profiling** | Always-on performance |
| **FinOps** | Cloud cost optimization |
| **Serverless Databases** | Neon, PlanetScale patterns |

---

**Total Concepts: 150+ core system design topics**

This comprehensive curriculum provides a complete roadmap for mastering system design at the Staff Engineer level, updated for 2026 with modern patterns and technologies.
