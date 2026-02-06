# AWS Database Services - Comprehensive Guide

## Table of Contents
1. [Introduction to AWS Database Services](#introduction)
2. [RDS (Relational Database Service)](#rds)
3. [Aurora](#aurora)
4. [DynamoDB](#dynamodb)
5. [ElastiCache](#elasticache)
6. [Other Specialized Database Services](#other-services)

---

## Introduction to AWS Database Services {#introduction}

AWS offers a broad portfolio of purpose-built database services designed to solve different data storage, retrieval, and processing challenges. Unlike managing databases on EC2 instances, these managed services handle infrastructure provisioning, patching, backups, and scaling—allowing you to focus on application logic rather than database operations.

### Why Multiple Database Types?

Different applications have different data access patterns:

- **Relational databases** excel at structured data with complex relationships and ACID transactions
- **NoSQL databases** handle massive scale, flexible schemas, and high-throughput workloads
- **In-memory databases** provide microsecond latency for caching and session management
- **Specialized databases** (graph, time-series, ledger) optimize for specific use cases

```
Application Requirements → Choose the Right Database

┌─────────────────────────────────────────────────────────────┐
│  Access Pattern Analysis                                     │
├─────────────────────────────────────────────────────────────┤
│  • Structured + ACID?        → RDS / Aurora                 │
│  • Key-value at scale?       → DynamoDB                     │
│  • Sub-millisecond reads?    → ElastiCache / DAX            │
│  • Graph relationships?      → Neptune                      │
│  • Time-series analytics?    → Timestream                   │
│  • Immutable audit trail?    → QLDB                         │
└─────────────────────────────────────────────────────────────┘
```

---

## RDS (Relational Database Service) {#rds}

### What is RDS?

**Amazon RDS** is a managed relational database service that automates time-consuming administrative tasks like hardware provisioning, database setup, patching, and backups. It runs traditional SQL database engines in the cloud without requiring you to manage the underlying infrastructure.

### Why RDS Exists

**Problem:** Running your own database servers requires expertise in:
- OS patching and security updates
- Database engine upgrades
- Backup scheduling and restoration
- Hardware failure recovery
- Scaling storage and compute
- High availability configuration

**Solution:** RDS handles all operational overhead while you retain full control over database schema, queries, and application logic.

### Supported Database Engines

RDS supports six major database engines, each with its own licensing model and feature set:

```
┌────────────────────────────────────────────────────────────┐
│ RDS Supported Engines                                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  MySQL          ─┐                                         │
│  PostgreSQL     ─┤  Open Source (no license fees)         │
│  MariaDB        ─┘                                         │
│                                                            │
│  Oracle         ─┐  Commercial (BYOL or License Included) │
│  SQL Server     ─┘                                         │
│                                                            │
│  Aurora          → AWS-optimized (separate service)       │
└────────────────────────────────────────────────────────────┘
```

**Why choose each engine:**

- **MySQL**: Most popular open-source database, wide ecosystem support
- **PostgreSQL**: Advanced features (JSON, full-text search, geospatial)
- **MariaDB**: MySQL fork with additional performance optimizations
- **Oracle**: Enterprise features, existing Oracle workload migration
- **SQL Server**: Windows/.NET stack integration, T-SQL compatibility

### Multi-AZ Deployments

**What it is:** Multi-AZ (Availability Zone) creates a synchronous standby replica in a different physical datacenter within the same AWS region.

**Why it exists:** Protects against infrastructure failure, not just database crashes.

**How it works:**

```
Primary AZ (us-east-1a)              Standby AZ (us-east-1b)
┌─────────────────────┐              ┌─────────────────────┐
│   RDS Primary       │              │   RDS Standby       │
│   (Active)          │              │   (Passive)         │
│                     │              │                     │
│  ┌──────────────┐   │              │  ┌──────────────┐   │
│  │  Database    │   │ Synchronous  │  │  Database    │   │
│  │  Engine      │───┼──Replication─┼─→│  Replica     │   │
│  └──────────────┘   │              │  └──────────────┘   │
│         │           │              │         ▲           │
└─────────┼───────────┘              └─────────┼───────────┘
          │                                    │
          └──────── DNS Endpoint ──────────────┘
                   (Automatic failover)

Application connects to: mydb.c9akciq32.us-east-1.rds.amazonaws.com
  - Points to Primary during normal operation
  - Auto-switches to Standby during failure (1-2 min)
```

**Key characteristics:**
- **Synchronous replication**: Every write is committed to both instances
- **Automatic failover**: DNS endpoint switches to standby (typically 60-120 seconds)
- **No data loss**: Standby is always current with primary
- **Use case**: Production databases requiring high availability
- **Cost**: ~2x single-AZ (you pay for both instances)

### Read Replicas

**What it is:** Asynchronously replicated copies of your database that handle read-only queries.

**Why it exists:** Offload read traffic from the primary database to improve performance and scale horizontally.

**Key differences from Multi-AZ:**

```
Multi-AZ vs Read Replicas

Multi-AZ (High Availability)          Read Replicas (Read Scaling)
┌─────────────────────────┐           ┌─────────────────────────┐
│ Synchronous replication │           │ Asynchronous replication│
│ Same region only        │           │ Cross-region supported  │
│ Standby not accessible  │           │ Replica is readable     │
│ Automatic failover      │           │ Manual promotion        │
│ No performance benefit  │           │ Distributes read load   │
└─────────────────────────┘           └─────────────────────────┘
```

**Architecture example:**

```
                        Application Layer
                        ┌───────────────┐
                        │  App Server   │
                        └───────┬───────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
         Writes │                        Reads  │
                ▼                               ▼
        ┌───────────────┐              ┌────────────────┐
        │ RDS Primary   │              │ Read Replica 1 │
        │ (us-east-1a)  │─────────────→│ (us-east-1b)   │
        │               │  Async       │                │
        │  READ + WRITE │  Replication │  READ ONLY     │
        └───────┬───────┘              └────────────────┘
                │                               │
                │                               ▼
                │                      ┌────────────────┐
                └─────────────────────→│ Read Replica 2 │
                          Async        │ (us-west-2)    │
                          Replication  │  READ ONLY     │
                                       └────────────────┘
                                       (Cross-region replica)
```

**Use cases:**
- **Read-heavy workloads**: Analytics, reporting dashboards
- **Geographic distribution**: Serve users closer to their location
- **Disaster recovery**: Promote replica to primary if region fails
- **Backup without impacting primary**: Run backups against replica

**Replica lag:** Typically milliseconds to seconds (monitor via `ReplicaLag` CloudWatch metric)

### RDS Proxy

**What it is:** A fully managed database proxy that sits between your application and RDS database, pooling and managing database connections.

**Why it exists:**

**Problem:** Applications (especially serverless functions) create and destroy database connections rapidly. Each connection consumes memory on the database server, and connection creation is slow (100-200ms).

**Solution:** RDS Proxy maintains a warm pool of database connections and multiplexes application requests across them.

```
Without RDS Proxy                     With RDS Proxy

App ─── New Connection ──→ RDS        App ──→ RDS Proxy ──→ RDS
App ─── New Connection ──→ RDS        App ──→     │         │
App ─── New Connection ──→ RDS        App ──→     │         │
App ─── New Connection ──→ RDS        App ──→     │         │
                                      App ──→     │         │
100 connections                                Pool of      │
                                              10 connections
Problem:
- Connection creation overhead        Benefits:
- Database memory exhaustion          - Connection reuse (faster)
- Lambda cold starts slow             - Reduced database load
                                      - Automatic failover handling
```

**Key benefits:**
- **Connection pooling**: Reuses existing database connections
- **Failover handling**: Automatically connects to standby during Multi-AZ failover (reduces failover time by ~66%)
- **IAM authentication**: Centralized credential management
- **Serverless-friendly**: Essential for Lambda functions that scale rapidly

**When to use:**
- Serverless architectures (Lambda + RDS)
- Applications with unpredictable connection patterns
- Microservices with many concurrent connections

### Parameter Groups

**What it is:** A collection of database engine configuration values that control database behavior.

**Why it exists:** Different database engines have hundreds of tunable parameters (buffer sizes, timeouts, query limits). Parameter groups let you manage these configurations as reusable templates.

```
┌──────────────────────────────────────────────────────────┐
│ RDS Parameter Group Structure                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  default.mysql8.0          ← AWS-provided default       │
│    ├─ max_connections = 150                             │
│    ├─ innodb_buffer_pool_size = {DBInstanceClassMemory} │
│    └─ query_cache_size = 0                              │
│                                                          │
│  custom-production-mysql   ← Your custom group          │
│    ├─ max_connections = 500        (modified)           │
│    ├─ slow_query_log = 1           (enabled)            │
│    ├─ long_query_time = 2          (log queries > 2s)   │
│    └─ inherits other defaults                           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Parameter types:**
- **Static parameters**: Require database reboot to apply (e.g., `innodb_log_file_size`)
- **Dynamic parameters**: Apply immediately without reboot (e.g., `max_connections`)

**Example configuration:**

```sql
-- View current parameter values
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- These are READ-ONLY in RDS - modify via Parameter Groups in AWS Console/CLI
```

```bash
# AWS CLI: Create custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name prod-mysql-params \
  --db-parameter-group-family mysql8.0 \
  --description "Production MySQL parameters"

# Modify parameters
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-mysql-params \
  --parameters "ParameterName=max_connections,ParameterValue=500,ApplyMethod=immediate"
```

### Option Groups

**What it is:** Enable optional features or plugins for specific database engines (mainly Oracle and SQL Server).

**Why it exists:** Some database engines have add-on features that aren't enabled by default (security, auditing, replication tools).

**Examples:**
- **Oracle**: Transparent Data Encryption (TDE), Oracle Enterprise Manager
- **SQL Server**: SQL Server Audit, mirroring
- **MySQL**: MariaDB Audit Plugin

```
┌─────────────────────────────────────────────────────────┐
│ Option Group Example (Oracle)                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  oracle-option-group                                    │
│    ├─ OEM (Oracle Enterprise Manager) - enabled        │
│    ├─ TDE (Transparent Data Encryption) - enabled      │
│    └─ Timezone - set to US/Pacific                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Automated Backups

**What it is:** RDS automatically takes daily snapshots of your database and captures transaction logs throughout the day.

**Why it exists:** Provides point-in-time recovery without manual intervention.

**How it works:**

```
Backup Timeline (Retention: 7 days)

Day 1       Day 2       Day 3       Day 4       Day 5
  │           │           │           │           │
  ▼           ▼           ▼           ▼           ▼
┌───┐       ┌───┐       ┌───┐       ┌───┐       ┌───┐
│ S │       │ S │       │ S │       │ S │       │ S │  ← Daily Snapshots
└───┘       └───┘       └───┘       └───┘       └───┘

Transaction Logs (Captured every 5 minutes)
├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │

Point-in-Time Recovery Window
├─────────────────────────────────────────────────────┤
                     Any 5-minute increment
```

**Key characteristics:**
- **Retention period**: 1-35 days (default: 7 days)
- **Backup window**: 30-minute window you specify (or AWS chooses)
- **Performance impact**: Minimal on Multi-AZ (backup from standby); brief I/O pause on single-AZ
- **Storage**: Backups stored in S3 (free up to database size)
- **Automatic deletion**: Deleted when you delete the database (unless you create final snapshot)

### Manual Snapshots

**What it is:** User-initiated full backups of your RDS instance.

**Why use them:**
- **Long-term archival**: Retained indefinitely until manually deleted
- **Before major changes**: Take snapshot before schema migrations or upgrades
- **Cross-region copy**: Copy snapshots to other regions for disaster recovery
- **Database cloning**: Create new databases from snapshots

```
Backup Strategy

Automated Backups              Manual Snapshots
┌────────────────┐             ┌────────────────┐
│ Daily (7 days) │             │ On-demand      │
│ Auto-deleted   │             │ Kept forever   │
│ PITR enabled   │             │ Full copy only │
└────────────────┘             └────────────────┘
        │                              │
        └──────────┬───────────────────┘
                   ▼
        Comprehensive Backup Strategy
```

```bash
# Create manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-before-migration-2026-02-06

# Copy snapshot to another region (disaster recovery)
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789012:snapshot:mydb-snap \
  --target-db-snapshot-identifier mydb-snap-dr \
  --region us-west-2
```

### Point-in-Time Recovery (PITR)

**What it is:** Restore your database to any specific second within your backup retention period.

**Why it exists:** Recover from human errors (accidental DELETE, DROP TABLE) without losing all recent data.

**How it works:**

```
Disaster Scenario: Accidental data deletion

Timeline:
09:00 AM - Normal operations
10:30 AM - Developer runs DELETE without WHERE clause 😱
10:32 AM - Error discovered
10:35 AM - Initiate PITR to 10:29 AM

┌────────────────────────────────────────────────────┐
│ Point-in-Time Recovery Process                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. Latest daily snapshot (09:00 AM)              │
│  2. + Transaction logs (09:00 - 10:29 AM)         │
│  3. = New database instance at 10:29 AM state     │
│                                                    │
└────────────────────────────────────────────────────┘

Result: New RDS instance with data as it existed at 10:29 AM
        (1 minute before the deletion)
```

**Important notes:**
- **Creates new instance**: PITR doesn't modify existing database
- **Granularity**: 5-minute increments
- **Time to restore**: Depends on database size (minutes to hours)

### Performance Insights

**What it is:** A database performance monitoring tool that visualizes database load and helps identify performance bottlenecks.

**Why it exists:** Traditional database monitoring shows system metrics (CPU, memory) but doesn't explain *why* the database is slow. Performance Insights shows which SQL queries are consuming resources.

**Dashboard visualization:**

```
Performance Insights Dashboard

Database Load (Average Active Sessions)
  ▲
5 │                                    ╱╲
  │                    ┌────┐        ╱  ╲
4 │                    │    │      ╱      ╲
  │        ┌────┐      │    │    ╱          ╲
3 │        │    │      │    │  ╱              ╲
  │        │    │      │    │╱                  ╲
2 │    ┌───┤    │  ┌───┤    │                    ╲
  │    │   │    │  │   │    │                      ╲
1 │────┤   │    │──┤   │    │────────────────────────
  │    │   │    │  │   │    │
  └────┴───┴────┴──┴───┴────┴─────────────────────────→
      Time

Top SQL Queries by Load:
1. SELECT * FROM orders WHERE status = 'pending'  (35%)
2. UPDATE inventory SET count = count - 1         (28%)
3. SELECT * FROM users WHERE email LIKE '%@%'     (22%)
```

**Key features:**
- **Wait event analysis**: See what queries are waiting for (locks, I/O, CPU)
- **Query-level metrics**: Identify specific slow queries
- **Historical data**: Retain up to 2 years of performance data
- **No agent required**: Built into RDS

### Enhanced Monitoring

**What it is:** Real-time operating system metrics collected at 1-second granularity.

**Why it exists:** CloudWatch provides database-level metrics (CPU, I/O) every 60 seconds. Enhanced Monitoring shows OS-level details (per-process CPU, memory) in near real-time.

**What it monitors:**

```
Enhanced Monitoring Metrics

OS Level                          Standard CloudWatch
┌─────────────────┐              ┌─────────────────┐
│ Per-process CPU │              │ Overall CPU %   │
│ Memory detail   │              │ Storage space   │
│ Swap usage      │              │ IOPS            │
│ Network I/O     │              │ Connections     │
│ Disk I/O        │              └─────────────────┘
└─────────────────┘
Granularity: 1 sec               Granularity: 60 sec
```

**Use cases:**
- Diagnose performance spikes with second-level precision
- Identify resource contention between processes
- Troubleshoot sudden CPU or memory exhaustion

### IAM Database Authentication

**What it is:** Use AWS IAM credentials instead of database passwords to authenticate to MySQL and PostgreSQL.

**Why it exists:**

**Problems with traditional database passwords:**
- Passwords stored in application config files
- Manual password rotation
- Shared passwords across teams
- Hard to audit who accessed the database

**Solution:** Generate temporary authentication tokens via IAM.

```
Traditional Authentication        IAM Database Authentication
┌──────────────────────┐         ┌──────────────────────┐
│ App Config File      │         │ IAM Role            │
│ ┌──────────────────┐ │         │ ┌────────────────┐  │
│ │ DB_USER=admin    │ │         │ │ db:connect     │  │
│ │ DB_PASS=secret123│ │         │ │ permission     │  │
│ └──────────────────┘ │         │ └────────────────┘  │
└──────────────────────┘         └──────────────────────┘
          ▼                                  ▼
   ┌─────────────┐                  ┌─────────────┐
   │ RDS MySQL   │                  │ RDS MySQL   │
   └─────────────┘                  └─────────────┘

Risks:                            Benefits:
- Password leakage                - No passwords stored
- Manual rotation                 - Auto-rotation (15 min)
- Hard to audit                   - IAM audit trail
```

**How it works:**

```python
import boto3
import pymysql

# Generate authentication token (valid for 15 minutes)
rds_client = boto3.client('rds')
token = rds_client.generate_db_auth_token(
    DBHostname='mydb.c9akciq32.us-east-1.rds.amazonaws.com',
    Port=3306,
    DBUsername='iam_user',
    Region='us-east-1'
)

# Connect using token instead of password
connection = pymysql.connect(
    host='mydb.c9akciq32.us-east-1.rds.amazonaws.com',
    user='iam_user',
    password=token,  # ← Temporary token, not a password
    database='myapp',
    ssl={'ca': '/path/to/rds-ca-cert.pem'}  # SSL required
)
```

### RDS Custom

**What it is:** Managed database service that gives you access to the underlying EC2 instance and operating system.

**Why it exists:**

Some enterprise applications require:
- Custom database extensions or plugins
- OS-level access for compliance agents
- Non-standard database configurations
- Legacy application compatibility

**Standard RDS:** Fully managed, no OS access
**RDS Custom:** Managed infrastructure, but you can access OS and database files

```
RDS Standard vs RDS Custom

Standard RDS                      RDS Custom
┌─────────────────┐              ┌─────────────────┐
│ ❌ No OS access  │              │ ✅ SSH access    │
│ ❌ No root       │              │ ✅ Install agents│
│ ✅ Auto patching │              │ ⚠️ Manual control│
│ ✅ Easy          │              │ ⚠️ More complex │
└─────────────────┘              └─────────────────┘

Use when:                        Use when:
- Standard workloads             - Need OS customization
- Maximum automation             - Compliance requirements
- Minimal management             - Legacy app support
```

**Supported engines:** Oracle, SQL Server (as of 2026)

---

## Aurora {#aurora}

### What is Aurora?

**Amazon Aurora** is a MySQL and PostgreSQL-compatible relational database built from scratch for the cloud. It's up to 5x faster than standard MySQL and 3x faster than standard PostgreSQL while providing the same compatibility.

### Why Aurora Exists (vs Standard RDS)

**Problem with traditional databases in the cloud:**
- Storage and compute are tightly coupled
- Replication copies entire database (slow and inefficient)
- Single-node writes limit write scalability
- Regional failures require complex failover

**Aurora's architectural innovations:**

```
Traditional RDS Architecture       Aurora Architecture
┌────────────────────────┐        ┌────────────────────────┐
│  DB Instance           │        │  DB Instance           │
│  ├─ Compute            │        │  ├─ Compute            │
│  └─ Storage (EBS)      │        │  └─ (Separated)        │
└────────┬───────────────┘        └────────┬───────────────┘
         │                                  │
         │ 1:1 Coupling                     │ Networked
         ▼                                  ▼
   ┌───────────┐               ┌─────────────────────────┐
   │ EBS Volume│               │ Aurora Storage Cluster  │
   │ (Single)  │               │ ┌───┬───┬───┬───┬───┬─┐ │
   └───────────┘               │ │ 1 │ 2 │ 3 │ 4 │ 5 │6│ │
                               │ └───┴───┴───┴───┴───┴─┘ │
                               │  6 copies across 3 AZs  │
                               └─────────────────────────┘

Limitations:                    Benefits:
- Storage scales with instance  - Auto-scaling (10GB→128TB)
- Slow replication              - Instant replication (redo)
- Failover minutes              - Failover < 30 seconds
```

### Aurora MySQL & PostgreSQL

**What it is:** Aurora is offered in two flavors, each compatible with open-source engines:

- **Aurora MySQL**: Compatible with MySQL 5.7, 8.0
- **Aurora PostgreSQL**: Compatible with PostgreSQL 11, 12, 13, 14, 15, 16

**Compatibility means:** You can use standard MySQL/PostgreSQL drivers, tools, and applications without modification.

```sql
-- Same SQL works on Aurora MySQL and standard MySQL
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Same PostgreSQL extensions work on Aurora PostgreSQL
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

**Aurora-specific enhancements:**
- Faster crash recovery (Aurora: seconds, MySQL: minutes)
- Read replica lag: typically < 10ms (vs 100ms+ in RDS)
- Automatic storage scaling
- Advanced replication features

### Aurora Serverless v2

**What it is:** Aurora that automatically scales compute capacity up and down based on application demand, measured in Aurora Capacity Units (ACUs).

**Why it exists:**

**Problem:** Traditional databases require you to provision fixed compute capacity (e.g., db.r6g.xlarge). You either:
- Over-provision (waste money during low traffic)
- Under-provision (poor performance during spikes)

**Solution:** Aurora Serverless v2 scales incrementally in 0.5 ACU increments.

```
Traditional Aurora Provisioned    Aurora Serverless v2

Fixed Instance Size               Auto-Scaling Capacity
┌─────────────────────┐          ┌─────────────────────┐
│ db.r6g.2xlarge      │          │ Min: 0.5 ACUs       │
│ (8 vCPU, 64GB RAM)  │          │ Max: 128 ACUs       │
│                     │          │                     │
│ ████████████████    │          │ ▁▃▅██████▅▃▁        │
│ Cost: $$$$ 24/7     │          │ Cost: Pay per use   │
└─────────────────────┘          └─────────────────────┘

Usage Pattern:                    Serverless Response:
08:00 - Low traffic  →  2 ACUs   08:00 - Scales to 2 ACUs
12:00 - Lunch spike  →  64 ACUs  12:00 - Auto-scales to 64 ACUs
18:00 - Low traffic  →  2 ACUs   18:00 - Scales down to 2 ACUs
02:00 - Near zero    →  0.5 ACUs 02:00 - Minimal capacity
```

**What is an ACU?**
- 1 ACU ≈ 2GB RAM + proportional CPU and networking
- Scaling happens in seconds (not minutes like RDS instance resizing)

**Use cases:**
- **Variable workloads**: E-commerce (peak during sales)
- **Dev/test environments**: Only use capacity when developers are active
- **Infrequently accessed apps**: Save costs during idle periods
- **Multi-tenant SaaS**: Different tenants have different usage patterns

**v1 vs v2:**
- **v1**: Paused when idle (cold start delay), coarse scaling
- **v2**: Never pauses, instant scaling, much more granular control

### Aurora Global Database

**What it is:** A single Aurora database that spans multiple AWS regions, with sub-second replication latency.

**Why it exists:**

**Problems with single-region databases:**
- Regional disasters affect availability
- High latency for users in distant geographies
- Complex disaster recovery procedures

**Solution:** Aurora Global Database replicates to secondary regions with minimal lag.

```
Aurora Global Database Architecture

Primary Region (us-east-1)         Secondary Region (ap-southeast-1)
┌─────────────────────────┐       ┌─────────────────────────┐
│ Primary DB Cluster      │       │ Secondary DB Cluster    │
│ ┌─────────────────────┐ │       │ ┌─────────────────────┐ │
│ │ Writer Instance     │ │       │ │ Read-Only Replicas  │ │
│ │ (Read + Write)      │ │       │ │ (Read-Only)         │ │
│ └─────────────────────┘ │       │ └─────────────────────┘ │
│ ┌─────────────────────┐ │       │                         │
│ │ Reader Instance     │ │       │                         │
│ │ (Read-Only)         │ │       │                         │
│ └─────────────────────┘ │       │                         │
└─────────┬───────────────┘       └───────▲─────────────────┘
          │                               │
          └──── Physical Replication ─────┘
               (< 1 second latency)

Application in US         Application in Asia
     │                           │
     └─── Writes ──→ Primary     │
                                 │
     ┌─── Reads ──→ Primary      │
                                 │
                         Reads ──┴─→ Secondary
```

**Key characteristics:**
- **Replication latency**: Typically < 1 second
- **Cross-region read replicas**: Up to 5 secondary regions
- **Disaster recovery**: Promote secondary region to primary in < 1 minute
- **Global read scaling**: Serve low-latency reads from local regions
- **Dedicated replication**: Uses Aurora storage layer (not instance resources)

**RPO and RTO:**
- **RPO** (Recovery Point Objective): < 1 second of data loss
- **RTO** (Recovery Time Objective): < 1 minute failover

### Aurora Replicas

**What it is:** Read-only copies of your Aurora database within the same region.

**Aurora Replicas vs RDS Read Replicas:**

```
┌──────────────────────────────────────────────────────────┐
│ Feature Comparison                                       │
├──────────────────────────────────────────────────────────┤
│                     Aurora        RDS Read Replicas      │
│ Replication lag     < 10ms        100ms - seconds        │
│ Storage             Shared        Separate (copied)      │
│ Failover target     Yes (auto)    Yes (manual promote)   │
│ Max replicas        15            5 (MySQL), 5 (Postgres)│
│ Performance impact  Minimal       Higher                 │
└──────────────────────────────────────────────────────────┘
```

**Shared storage architecture:**

```
Aurora Cluster with Replicas

┌────────────────────────────────────────────────────┐
│ Aurora Storage (Shared)                            │
│ ┌──────────────────────────────────────────────┐   │
│ │  6 copies across 3 AZs                      │   │
│ └──────────────────────────────────────────────┘   │
└─────┬─────────────┬─────────────┬─────────────┘
      │             │             │
      ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Writer   │  │ Reader 1 │  │ Reader 2 │
│ (Primary)│  │ (Replica)│  │ (Replica)│
└──────────┘  └──────────┘  └──────────┘
      │             │             │
      │             │             │
   Writes      Read      Read
   + Reads     Only      Only
```

**Use cases:**
- **Read scaling**: Distribute SELECT queries across replicas
- **High availability**: Automatic failover to replica if primary fails
- **Mixed workloads**: Analytical queries on replicas, transactional on writer

**Endpoint types:**
- **Cluster endpoint**: Points to writer (for writes)
- **Reader endpoint**: Load-balances reads across all replicas
- **Instance endpoint**: Connect to specific instance

### Aurora Multi-Master

**What it is:** Aurora cluster where all instances can perform writes simultaneously (no single writer limitation).

**Why it exists:**

**Standard Aurora limitation:** Only one writer instance at a time
**Multi-Master solution:** Multiple writer instances with conflict detection

```
Standard Aurora                Aurora Multi-Master
┌────────────────┐            ┌────────────────┐
│ Writer (1)     │            │ Writer A       │
│ ↓ Writes       │            │ ↕ Writes       │
└────────────────┘            └────────────────┘
┌────────────────┐                   │
│ Reader (many)  │            ┌──────┴──────┐
│ ← Reads only   │            ▼             ▼
└────────────────┘     ┌────────────┐ ┌────────────┐
                       │ Writer B   │ │ Writer C   │
Single point of        │ ↕ Writes   │ │ ↕ Writes   │
write failure          └────────────┘ └────────────┘

                       All instances can write
                       (Quorum-based conflict resolution)
```

**Use cases:**
- **High write availability**: If one writer fails, others continue
- **Write scaling**: Distribute writes across instances
- **Zero-downtime failover**: No promotion needed

**Trade-offs:**
- More complex (application must handle conflicts)
- Limited to 2 writer instances per cluster (as of 2026)
- Not compatible with Aurora Serverless or Global Database

### Aurora Cloning

**What it is:** Create a full copy of an Aurora database in minutes without copying all the data.

**Why it exists:**

**Traditional database copying:**
```
Standard Snapshot Restore        Aurora Cloning
┌─────────────────────┐         ┌─────────────────────┐
│ Source DB (100GB)   │         │ Source DB (100GB)   │
└─────────┬───────────┘         └─────────┬───────────┘
          │                               │
          │ Copy 100GB                    │ Copy-on-write
          │ (Hours)                       │ (Minutes)
          ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│ New DB (100GB)      │         │ Clone DB (0GB→...)  │
│ Full copy           │         │ Shares unchanged    │
└─────────────────────┘         └─────────────────────┘

Cost: 2x storage                Cost: Only changed data
Time: Hours                     Time: Minutes
```

**How it works (copy-on-write):**
1. Clone created instantly pointing to same storage
2. When clone writes data, only modified pages are copied
3. Original database unaffected by clone changes

**Use cases:**
- **Development/testing**: Clone production for QA testing
- **Data analysis**: Run heavy queries without affecting production
- **Schema migration testing**: Test migrations on exact copy
- **Quick rollback**: If deployment fails, switch to clone

```bash
# Create clone via CLI
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier production-cluster \
  --db-cluster-identifier test-clone \
  --restore-type copy-on-write \
  --use-latest-restorable-time
```

### Aurora Backtrack

**What it is:** "Rewind" your Aurora MySQL database to a previous point in time without restoring from backup.

**Why it exists:**

**Problem:** Point-in-time recovery creates a new database instance (slow, requires DNS changes)
**Solution:** Backtrack rewinds the same database in minutes

```
Traditional PITR                  Aurora Backtrack
┌──────────────────┐             ┌──────────────────┐
│ Discover error   │             │ Discover error   │
│ at 10:30 AM      │             │ at 10:30 AM      │
└────────┬─────────┘             └────────┬─────────┘
         │                                │
         ▼                                ▼
┌──────────────────┐             ┌──────────────────┐
│ Create new DB    │             │ Backtrack same   │
│ from snapshot    │             │ DB to 10:15 AM   │
│ (30-60 min)      │             │ (Minutes)        │
└────────┬─────────┘             └────────┬─────────┘
         │                                │
         ▼                                ▼
┌──────────────────┐             ┌──────────────────┐
│ Update app to    │             │ No DNS change    │
│ new endpoint     │             │ needed           │
└──────────────────┘             └──────────────────┘
```

**How it works:**
- Aurora keeps a continuous redo log
- Backtrack "replays" or "unreplays" transactions
- Same database endpoint, no new instance

**Limitations:**
- **MySQL only** (not available for Aurora PostgreSQL)
- **Backtrack window**: Up to 72 hours (you configure)
- **Database downtime**: Few minutes while rewinding
- **Not a replacement for backups**: Can't backtrack beyond retention window

**Use cases:**
- Quickly undo accidental data modifications
- Test time-sensitive bugs ("what did the data look like at 3:00 AM?")
- Faster recovery from human errors

```bash
# Enable backtrack when creating cluster
aws rds create-db-cluster \
  --db-cluster-identifier my-cluster \
  --engine aurora-mysql \
  --backtrack-window 72  # Hours

# Backtrack to specific time
aws rds backtrack-db-cluster \
  --db-cluster-identifier my-cluster \
  --backtrack-to "2026-02-06T10:15:00Z"
```

### Aurora Machine Learning

**What it is:** Run machine learning inference directly from SQL queries using pre-trained models.

**Why it exists:**

**Traditional ML workflow:**
```
Query DB → Export data → Send to ML service → Get results → Write back to DB
```

**Aurora ML workflow:**
```
Query DB with ML function → Results returned inline
```

**Supported ML services:**
- **Amazon SageMaker**: Custom models
- **Amazon Comprehend**: Sentiment analysis, entity detection
- **Amazon Bedrock**: Generative AI

**Example use case - Sentiment analysis:**

```sql
-- Analyze customer feedback sentiment directly in SQL
SELECT
  feedback_id,
  feedback_text,
  aws_comprehend_detect_sentiment(feedback_text, 'en') AS sentiment
FROM
  customer_feedback
WHERE
  created_at > NOW() - INTERVAL 7 DAY;

-- Result:
-- feedback_id | feedback_text              | sentiment
-- 1           | "Great service!"           | {"Sentiment": "POSITIVE", "Score": 0.95}
-- 2           | "Terrible experience"      | {"Sentiment": "NEGATIVE", "Score": 0.89}
```

**How it works:**
```
Aurora Database                AWS Machine Learning Service
┌─────────────────┐           ┌──────────────────────────┐
│ SQL Query       │           │ SageMaker / Comprehend   │
│ SELECT          │           │                          │
│   id,           │──────────→│ Model inference          │
│   ML_FUNCTION() │←──────────│ Returns predictions      │
└─────────────────┘           └──────────────────────────┘
```

**Security:** Uses IAM roles (Aurora doesn't send your data outside AWS account)

### Aurora I/O-Optimized

**What it is:** A pricing option where you pay more for compute but zero for I/O operations.

**Why it exists:**

**Standard Aurora pricing:**
- Compute cost (instance hours)
- Storage cost (GB-month)
- **I/O cost (per million requests)**

**Problem:** I/O-intensive workloads can have unpredictable costs

**Aurora I/O-Optimized pricing:**
- Higher compute cost (+30-40%)
- Zero I/O charges

```
Cost Comparison Example

Standard Aurora             I/O-Optimized Aurora
┌────────────────────┐     ┌────────────────────┐
│ Compute:  $500/mo  │     │ Compute:  $700/mo  │
│ Storage:  $100/mo  │     │ Storage:  $100/mo  │
│ I/O:      $400/mo  │     │ I/O:      $0       │
├────────────────────┤     ├────────────────────┤
│ Total:   $1,000/mo │     │ Total:    $800/mo  │
└────────────────────┘     └────────────────────┘

When to use I/O-Optimized:
✓ I/O costs > 25% of total Aurora bill
✓ High read/write workloads
✓ Need cost predictability
```

**When to switch:**
- Monitor your Aurora I/O charges in Cost Explorer
- If I/O > 25-30% of total cost, I/O-Optimized likely saves money

### Aurora Limitless Database

**What it is:** Horizontally scale Aurora beyond single-instance write limits by automatically sharding data across multiple writer instances.

**Why it exists:**

**Standard Aurora limitation:**
- Single writer instance (limited write throughput)
- Readers can scale to 15, but writes bottleneck at one instance

**Aurora Limitless:**
- Distributes data across multiple shards
- Each shard has its own writer
- Application sees single database

```
Standard Aurora                Aurora Limitless
┌─────────────────┐           ┌──────────────────────────────┐
│ Single Writer   │           │ Distributed Writers          │
│ Max: ~200K      │           │                              │
│ writes/sec      │           │ Shard 1    Shard 2    Shard 3│
└─────────────────┘           │ Writer     Writer     Writer │
        ▲                     │   ▲          ▲          ▲    │
        │                     └───┼──────────┼──────────┼────┘
      Writes                      │          │          │
                                  └──────────┴──────────┘
Write scaling                           Sharded writes
limited                            (Millions of writes/sec)
```

**How it works:**
1. Aurora automatically partitions data across shards
2. Router layer directs queries to correct shard
3. Cross-shard queries are coordinated
4. Application uses standard PostgreSQL or MySQL drivers

**Use cases:**
- **Extreme write throughput**: Gaming leaderboards, IoT sensor data
- **Large datasets**: Multi-terabyte databases with high concurrency
- **SaaS platforms**: Millions of tenants with isolated data

**Limitations:**
- **PostgreSQL only** (as of 2026)
- Requires schema design consideration for optimal sharding
- Some PostgreSQL features may have limitations

**Note:** Aurora Limitless is a newer feature (public preview/GA timeline varies by region).

---

## DynamoDB {#dynamodb}

### What is DynamoDB?

**Amazon DynamoDB** is a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability. It's designed for applications that need consistent, single-digit millisecond latency at any scale.

### Why DynamoDB Exists

**Problems with traditional databases at web scale:**
- Difficult to scale horizontally
- Schema changes require downtime
- Performance degrades as data grows
- Complex sharding and replication management

**DynamoDB solutions:**
- Automatic horizontal scaling
- Schema-less (flexible data model)
- Consistent performance at petabyte scale
- Fully managed replication and backups

```
Traditional RDBMS at Scale        DynamoDB
┌──────────────────────┐         ┌──────────────────────┐
│ Fixed schema         │         │ Flexible schema      │
│ Vertical scaling     │         │ Horizontal scaling   │
│ Manual sharding      │         │ Auto-sharding        │
│ Complex replication  │         │ Built-in replication │
└──────────────────────┘         └──────────────────────┘
```

### Tables, Items, and Attributes

**Data model basics:**

```
DynamoDB Table Structure

Table: "Users"
┌─────────────────────────────────────────────────────────┐
│ Item 1 (like a row)                                     │
│ {                                                       │
│   "UserID": "12345",          ← Attribute (column)      │
│   "Email": "alice@example.com",                         │
│   "Age": 28,                                            │
│   "Interests": ["hiking", "photography"]  ← Can be list│
│ }                                                       │
├─────────────────────────────────────────────────────────┤
│ Item 2                                                  │
│ {                                                       │
│   "UserID": "67890",                                    │
│   "Email": "bob@example.com",                           │
│   "Country": "Canada"  ← Different attributes OK!      │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Table**: Collection of items (like a table in RDBMS)
- **Item**: Collection of attributes (like a row), max size 400KB
- **Attribute**: Name-value pair (like a column)
- **Schema-less**: Each item can have different attributes

**Supported data types:**
- **Scalar**: String, Number, Binary, Boolean, Null
- **Document**: List, Map (nested JSON-like structures)
- **Set**: String Set, Number Set, Binary Set

### Primary Keys (Partition Key, Sort Key)

**What it is:** Every DynamoDB table must have a primary key that uniquely identifies each item.

**Two types of primary keys:**

#### 1. Partition Key Only (Simple Primary Key)

```
Table: "Products"
┌──────────────────────────────────────────┐
│ ProductID (Partition Key) │ Name   │Price│
├───────────────────────────┼────────┼─────┤
│ "P123"                    │ Laptop │$999 │
│ "P456"                    │ Mouse  │$25  │
└──────────────────────────────────────────┘

• ProductID must be unique
• Items distributed across partitions by hashing ProductID
```

#### 2. Partition Key + Sort Key (Composite Primary Key)

```
Table: "Orders"
┌────────────────────────────────────────────────────────┐
│ CustomerID │ OrderDate  │ Items      │ Total           │
│ (Partition)│ (Sort Key) │            │                 │
├────────────┼────────────┼────────────┼─────────────────┤
│ "C001"     │ 2026-01-15 │ [...]      │ $150            │
│ "C001"     │ 2026-02-03 │ [...]      │ $89   ← Same    │
│ "C002"     │ 2026-01-20 │ [...]      │ $200    customer│
└────────────────────────────────────────────────────────┘

• Items with same Partition Key stored together
• Sorted by Sort Key within partition
• Combination (CustomerID + OrderDate) must be unique
```

**Why use composite keys:**
- Query all items with same partition key efficiently
- Items automatically sorted by sort key
- Enables range queries (e.g., "all orders for C001 in February 2026")

**How DynamoDB distributes data:**

```
Data Distribution by Partition Key

Application writes item with PartitionKey = "C001"
                │
                ▼
        Hash("C001") = 12345678
                │
                ▼
        ┌───────────────────────┐
        │ DynamoDB determines   │
        │ which partition based │
        │ on hash value         │
        └───────────┬───────────┘
                    │
        ┌───────────┴───────────┬────────────┐
        ▼                       ▼            ▼
  ┌──────────┐           ┌──────────┐  ┌──────────┐
  │Partition │           │Partition │  │Partition │
  │    1     │           │    2     │  │    3     │
  └──────────┘           └──────────┘  └──────────┘
```

**Design considerations:**
- **Even distribution**: Choose partition keys that distribute data evenly
- **Avoid hot partitions**: Don't use keys where one value gets most traffic (e.g., "Status" with 90% = "Active")
- **Access patterns**: Design keys based on how you'll query data

### Secondary Indexes (LSI, GSI)

**What it is:** Additional ways to query your data beyond the primary key.

**Problem without indexes:**
```
Table: "Users" (Primary Key: UserID)

Query 1: Get user by UserID        ✓ Fast (primary key)
Query 2: Get all users in "Canada" ✗ Slow (full table scan)
Query 3: Get users by email        ✗ Slow (full table scan)
```

**Solution:** Create secondary indexes

#### Local Secondary Index (LSI)

**What it is:** Alternate sort key with the same partition key as the table.

```
Base Table                        LSI: "UsersByJoinDate"
Partition: UserID                 Partition: UserID (same)
Sort: (none)                      Sort: JoinDate (new)

┌────────┬───────┬──────────┐   ┌────────┬──────────┬───────┐
│ UserID │ Email │ JoinDate │   │ UserID │ JoinDate │ Email │
├────────┼───────┼──────────┤   ├────────┼──────────┼───────┤
│ U001   │ ...   │ 2025-01  │   │ U001   │ 2025-01  │ ...   │
│ U002   │ ...   │ 2026-01  │   │ U001   │ 2026-01  │ ...   │  ← Same
└────────┴───────┴──────────┘   └────────┴──────────┴───────┘  UserID

Query: Get all items for UserID ordered by JoinDate
```

**LSI characteristics:**
- Must be created at table creation time (cannot add later)
- Same partition key as table
- Max 5 LSIs per table
- Shares throughput with base table
- Strongly consistent reads available

#### Global Secondary Index (GSI)

**What it is:** Completely different partition and sort keys from the table.

```
Base Table                        GSI: "UsersByCountry"
Partition: UserID                 Partition: Country (different!)
                                  Sort: Email (different!)

┌────────┬─────────┬─────────┐  ┌─────────┬───────┬────────┐
│ UserID │ Country │ Email   │  │ Country │ Email │ UserID │
├────────┼─────────┼─────────┤  ├─────────┼───────┼────────┤
│ U001   │ USA     │ a@...   │  │ Canada  │ b@... │ U002   │
│ U002   │ Canada  │ b@...   │  │ USA     │ a@... │ U001   │
│ U003   │ Canada  │ c@...   │  │ Canada  │ c@... │ U003   │
└────────┴─────────┴─────────┘  └─────────┴───────┴────────┘

Query: Get all users in Canada (fast with GSI)
```

**GSI characteristics:**
- Can be added/deleted after table creation
- Has its own partition and sort key
- Max 20 GSIs per table (as of 2026)
- **Eventually consistent** (not strongly consistent)
- Has its own provisioned throughput (separate from table)

**When to use each:**

```
┌──────────────────────────────────────────────────────┐
│ LSI vs GSI Decision                                  │
├──────────────────────────────────────────────────────┤
│ Use LSI when:                                        │
│ • Need strong consistency                            │
│ • Queries use same partition key                     │
│ • Decided at table design time                       │
│                                                      │
│ Use GSI when:                                        │
│ • Need different partition key                       │
│ • Need to query by attributes not in primary key     │
│ • May need to add indexes later                      │
└──────────────────────────────────────────────────────┘
```


### Read/Write Capacity Modes (Provisioned, On-Demand)

**What it is:** How you pay for and configure DynamoDB throughput.

#### Provisioned Capacity Mode

**What it is:** You specify the number of reads and writes per second your application needs.

**Capacity units:**
- **Read Capacity Unit (RCU)**: 1 strongly consistent read/sec of up to 4KB (or 2 eventually consistent reads/sec)
- **Write Capacity Unit (WCU)**: 1 write/sec of up to 1KB

**Example calculations:**

```
Provisioned Capacity Planning

Scenario: E-commerce product catalog
• 100 reads/sec of 8KB items
• 50 writes/sec of 2KB items

Read capacity needed:
  8KB ÷ 4KB = 2 RCUs per read
  100 reads/sec × 2 RCUs = 200 RCUs

Write capacity needed:
  2KB ÷ 1KB = 2 WCUs per write (round up)
  50 writes/sec × 2 WCUs = 100 WCUs

Provisioned: 200 RCUs + 100 WCUs
Cost: ~$100/month (varies by region)
```

**Auto-scaling:**
```
Capacity Over Time (with Auto Scaling)

Throughput
  ▲
  │     ┌─────┐
  │     │     │        ┌──────
  │     │     │        │
  │─────┘     └────────┘
  │
  └─────────────────────────→ Time
        ▲               ▲
    Scale Up        Scale Down
    (traffic spike) (low usage)

Auto Scaling Settings:
• Min: 50 RCUs/WCUs
• Max: 500 RCUs/WCUs
• Target utilization: 70%
```

**When to use provisioned:**
- Predictable workloads
- Consistent traffic patterns
- Cost optimization (cheaper than on-demand at sustained load)

#### On-Demand Capacity Mode

**What it is:** Pay per request with no capacity planning required.

**How it works:**
```
On-Demand Pricing

No provisioning needed:
• DynamoDB scales automatically
• Pay only for reads/writes you use

Cost model:
• Write: $1.25 per million write request units
• Read: $0.25 per million read request units
  (Prices approximate, vary by region)

Example:
  1 million reads of 4KB  = $0.25
  1 million writes of 1KB = $1.25
```

**Comparison:**

```
┌────────────────────────────────────────────────────────┐
│ Provisioned vs On-Demand                               │
├────────────────────────────────────────────────────────┤
│ Provisioned:                                           │
│ ✓ Lower cost at sustained high throughput             │
│ ✓ Predictable billing                                 │
│ ✗ Requires capacity planning                          │
│ ✗ Can throttle if exceed provisioned capacity         │
│                                                        │
│ On-Demand:                                             │
│ ✓ No capacity planning                                │
│ ✓ Scales automatically to any load                    │
│ ✗ ~5x more expensive than provisioned (at scale)      │
│ ✓ Perfect for unpredictable workloads                 │
└────────────────────────────────────────────────────────┘
```

**When to use on-demand:**
- New applications (unknown traffic patterns)
- Unpredictable or spiky workloads
- Development/test environments
- "Serverless" architecture preference

**You can switch between modes once every 24 hours**

### DynamoDB Streams

**What it is:** A time-ordered sequence of item-level modifications (create, update, delete) in a DynamoDB table.

**Why it exists:** Enable event-driven architectures and real-time processing of data changes.

**How it works:**

```
DynamoDB Table                  DynamoDB Stream
┌───────────────┐              ┌─────────────────────────┐
│ Item inserted │──────────────→│ Event 1: INSERT         │
│ Item updated  │──────────────→│ Event 2: MODIFY         │
│ Item deleted  │──────────────→│ Event 3: REMOVE         │
└───────────────┘              └────────┬────────────────┘
                                        │
                                        ▼
                              ┌──────────────────┐
                              │ Stream Consumer  │
                              │ (Lambda, etc.)   │
                              └──────────────────┘
```

**Stream record content options:**
- **KEYS_ONLY**: Only primary key attributes
- **NEW_IMAGE**: Entire item after modification
- **OLD_IMAGE**: Entire item before modification
- **NEW_AND_OLD_IMAGES**: Both before and after states

**Example stream record:**

```json
{
  "eventID": "1",
  "eventName": "MODIFY",
  "eventVersion": "1.1",
  "eventSource": "aws:dynamodb",
  "awsRegion": "us-east-1",
  "dynamodb": {
    "Keys": {
      "UserID": {"S": "U001"}
    },
    "NewImage": {
      "UserID": {"S": "U001"},
      "Status": {"S": "Premium"},
      "Email": {"S": "user@example.com"}
    },
    "OldImage": {
      "UserID": {"S": "U001"},
      "Status": {"S": "Free"},
      "Email": {"S": "user@example.com"}
    },
    "SequenceNumber": "123456789"
  }
}
```

**Use cases:**

```
Common DynamoDB Streams Patterns

1. Real-time Analytics
   DynamoDB → Stream → Lambda → Kinesis → Analytics

2. Data Replication
   DynamoDB (us-east-1) → Stream → Lambda → DynamoDB (eu-west-1)

3. Elasticsearch Sync
   DynamoDB → Stream → Lambda → Elasticsearch (search index)

4. Audit Trail
   DynamoDB → Stream → Lambda → S3 / CloudWatch Logs

5. Trigger Business Logic
   Order table → Stream → Lambda → Send confirmation email
```

**Retention:** Stream records are available for 24 hours


### Global Tables

**What it is:** Multi-region, multi-active replication for DynamoDB tables.

**Why it exists:** Provide low-latency access to data for globally distributed applications and disaster recovery.

**Architecture:**

```
Global Table: "Users"

Region: us-east-1              Region: eu-west-1
┌─────────────────┐           ┌─────────────────┐
│ DynamoDB Table  │←─────────→│ DynamoDB Table  │
│                 │ Bi-direct │                 │
│ Read + Write    │   Repl.   │ Read + Write    │
└────────┬────────┘           └────────┬────────┘
         │                             │
         │                             │
    App (US users)               App (EU users)
    Low latency                  Low latency

         │                             │
         ▼                             ▼
Region: ap-southeast-1
┌─────────────────┐
│ DynamoDB Table  │
│                 │
│ Read + Write    │
└────────┬────────┘
         │
    App (Asia users)
    Low latency
```

**Key characteristics:**
- **Multi-active writes**: Applications can write to any region
- **Automatic replication**: Changes replicated typically within 1 second
- **Conflict resolution**: Last Writer Wins (LWW) based on timestamp
- **Consistency**: Eventually consistent across regions
- **No code changes**: Application uses local table endpoint

**Conflict resolution example:**

```
Conflict Scenario

Time: 10:00:00.000
User writes to us-east-1: {"UserID": "U1", "Status": "Active", "timestamp": 10:00:00.000}

Time: 10:00:00.100
User writes to eu-west-1: {"UserID": "U1", "Status": "Inactive", "timestamp": 10:00:00.100}

Resolution (Last Write Wins):
After replication completes, both regions have:
{"UserID": "U1", "Status": "Inactive", "timestamp": 10:00:00.100}
                  ▲
                  Latest timestamp wins
```

**Use cases:**
- Global user base (serve users from nearest region)
- Business continuity (if one region fails, others continue)
- Compliance (data residency in multiple regions)

### TTL (Time to Live)

**What it is:** Automatically delete items after a specified timestamp without consuming write capacity.

**Why it exists:** Remove expired data (session tokens, temporary data) without manual cleanup jobs.

**How it works:**

```
DynamoDB Table with TTL

┌────────┬──────────────┬──────────────┐
│ ItemID │ Data         │ ExpiryTime   │ ← TTL attribute
├────────┼──────────────┼──────────────┤
│ I001   │ Session data │ 1675612800   │ ← Expired (will be deleted)
│ I002   │ Session data │ 1707235200   │ ← Active (future timestamp)
└────────┴──────────────┴──────────────┘
                              ▲
                              Unix timestamp (seconds since epoch)

DynamoDB Background Process:
• Scans for items where ExpiryTime < current_time
• Deletes expired items within 48 hours
• No additional cost (doesn't consume WCUs)
```

**Implementation example:**

```python
import time

# Enable TTL on table (one-time setup via AWS CLI/Console)
# aws dynamodb update-time-to-live \
#   --table-name Sessions \
#   --time-to-live-specification "Enabled=true, AttributeName=ExpiryTime"

# Application code: Set expiry when writing items
current_time = int(time.time())
expiry_time = current_time + (30 * 60)  # 30 minutes from now

dynamodb.put_item(
    TableName='Sessions',
    Item={
        'SessionID': {'S': 'session123'},
        'UserData': {'S': 'encrypted_data'},
        'ExpiryTime': {'N': str(expiry_time)}  # TTL attribute
    }
)

# DynamoDB automatically deletes this item ~30 minutes later
# (Actual deletion may occur within 48 hours after expiry)
```

**Important notes:**
- **Deletion timing**: Not immediate (typically within 48 hours)
- **Free**: Doesn't consume write capacity
- **Deletions visible in streams**: TTL deletions appear as REMOVE events
- **Not for time-critical deletions**: Use for "soft" expiry (sessions, cache)

### Transactions

**What it is:** ACID transactions across multiple DynamoDB items, even across different tables.

**Why it exists:** Ensure data consistency when multiple items must be updated together (all succeed or all fail).

**Example without transactions (problematic):**

```
Bank Transfer Problem (Without Transactions)

Step 1: Deduct $100 from Account A  ✓ Success
Step 2: Add $100 to Account B       ✗ Failure (network error)

Result: $100 disappeared! 💸

┌────────────┐                    ┌────────────┐
│ Account A  │                    │ Account B  │
│ Balance:   │                    │ Balance:   │
│ $1000      │                    │ $500       │
│   ↓        │                    │            │
│ $900 ✓     │                    │ $500 ✗     │
└────────────┘                    └────────────┘
             Money lost!
```

**With DynamoDB transactions:**

```
Bank Transfer (With Transaction)

TransactWriteItems:
  1. Deduct $100 from Account A
  2. Add $100 to Account B
  3. Create transaction log entry

If ANY step fails → ALL steps rolled back

┌────────────┐                    ┌────────────┐
│ Account A  │                    │ Account B  │
│ Balance:   │                    │ Balance:   │
│ $1000      │                    │ $500       │
│   ↓        │                    │   ↓        │
│ $900 ✓     │                    │ $600 ✓     │
└────────────┘                    └────────────┘
        Both succeed or both fail (atomic)
```

**Transaction APIs:**

```python
# TransactWriteItems (up to 100 items per transaction)
dynamodb.transact_write_items(
    TransactItems=[
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'AccountID': {'S': 'A001'}},
                'UpdateExpression': 'SET Balance = Balance - :amount',
                'ExpressionAttributeValues': {':amount': {'N': '100'}},
                'ConditionExpression': 'Balance >= :amount'  # Ensure sufficient funds
            }
        },
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'AccountID': {'S': 'A002'}},
                'UpdateExpression': 'SET Balance = Balance + :amount',
                'ExpressionAttributeValues': {':amount': {'N': '100'}}
            }
        }
    ]
)

# TransactGetItems (read multiple items atomically)
response = dynamodb.transact_get_items(
    TransactItems=[
        {'Get': {'TableName': 'Accounts', 'Key': {'AccountID': {'S': 'A001'}}}},
        {'Get': {'TableName': 'Accounts', 'Key': {'AccountID': {'S': 'A002'}}}}
    ]
)
```

**Limitations:**
- Max 100 items per transaction
- 2x cost compared to standard writes
- Cannot mix with batch operations

**Use cases:**
- Financial transactions
- Inventory management (reserve items across multiple warehouses)
- Multi-step workflows requiring consistency

### PartiQL

**What it is:** SQL-compatible query language for DynamoDB.

**Why it exists:** Provide familiar SQL syntax for developers while maintaining NoSQL performance.

**Standard DynamoDB API vs PartiQL:**

```
Standard DynamoDB API (Verbose)

response = dynamodb.query(
    TableName='Orders',
    KeyConditionExpression='CustomerID = :custid',
    FilterExpression='Total > :minTotal',
    ExpressionAttributeValues={
        ':custid': {'S': 'C001'},
        ':minTotal': {'N': '100'}
    }
)

PartiQL (SQL-like, Concise)

response = dynamodb.execute_statement(
    Statement="""
        SELECT * FROM Orders
        WHERE CustomerID = 'C001'
        AND Total > 100
    """
)
```

**Supported SQL operations:**

```sql
-- SELECT (query items)
SELECT * FROM Users WHERE Country = 'USA';

-- INSERT (put item)
INSERT INTO Users VALUE {
    'UserID': 'U123',
    'Email': 'user@example.com',
    'Country': 'USA'
};

-- UPDATE (modify item)
UPDATE Users
SET Status = 'Premium'
WHERE UserID = 'U123';

-- DELETE (remove item)
DELETE FROM Users WHERE UserID = 'U123';

-- Batch operations
SELECT * FROM Users WHERE UserID IN ['U001', 'U002', 'U003'];
```

**Limitations:**
- Still requires primary key for efficient queries (no full table scans without key)
- JOINs not supported (DynamoDB is NoSQL)
- Subset of SQL syntax (not all SQL features)

**When to use:**
- SQL familiarity (easier for RDBMS developers)
- Ad-hoc queries and exploration
- Simpler syntax for common operations

### DAX (DynamoDB Accelerator)

**What it is:** In-memory cache for DynamoDB that provides microsecond read latency.

**Why it exists:**

**Problem:** DynamoDB provides single-digit millisecond latency, but some applications need even faster reads (gaming leaderboards, real-time bidding).

**Solution:** DAX sits in front of DynamoDB and caches frequently accessed data.

```
Without DAX                       With DAX

Application                       Application
     │                                 │
     │ Read request                    │ Read request
     │ (5-10 ms latency)               │
     ▼                                 ▼
┌─────────────┐                 ┌─────────────┐
│ DynamoDB    │                 │ DAX Cluster │
│             │                 │ (Cache)     │
└─────────────┘                 │ (~400 μs)   │
                                └──────┬──────┘
                                       │ Cache miss
                                       │ (rare)
                                       ▼
                                ┌─────────────┐
                                │ DynamoDB    │
                                │             │
                                └─────────────┘

Latency: ~5-10 milliseconds      Latency: ~400 microseconds
                                 (10-20x faster)
```

**How DAX works:**

```
DAX Cluster Architecture

┌────────────────────────────────────────────────┐
│ Application                                    │
└───────┬────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────┐
│ DAX Cluster (Multi-node, Multi-AZ)             │
│ ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│ │ DAX Node│  │ DAX Node│  │ DAX Node│         │
│ │ (Cache) │  │ (Cache) │  │ (Cache) │         │
│ └─────────┘  └─────────┘  └─────────┘         │
└───────┬────────────────────────────────────────┘
        │ Cache miss / Write-through
        ▼
┌────────────────────────────────────────────────┐
│ DynamoDB Table                                 │
└────────────────────────────────────────────────┘
```

**Cache types:**
1. **Item cache**: Caches individual items (GetItem, BatchGetItem)
2. **Query cache**: Caches query results

**Write behavior:**
- **Write-through**: Writes go to DynamoDB first, then DAX invalidates cache
- **Eventually consistent**: DAX cache may be slightly stale (milliseconds)

**When to use DAX:**
- Read-heavy workloads with hot items (same items read repeatedly)
- Need microsecond latency
- Eventually consistent reads acceptable
- Budget for additional infrastructure

**When NOT to use DAX:**
- Strongly consistent reads required
- Write-heavy workloads (no benefit)
- Uniform access patterns (no hot items)

**Cost:** Additional hourly charges for DAX cluster nodes

### Point-in-Time Recovery (PITR)

**What it is:** Restore DynamoDB table to any second within the last 35 days.

**Why it exists:** Protect against accidental deletions or data corruption.

```
PITR Timeline

Day 1         Day 10        Day 20        Day 30     Day 35
  │             │             │             │           │
  ▼             ▼             ▼             ▼           ▼
  ●─────────────●─────────────●─────────────●───────────●

  └──────────────── Recovery Window ─────────────────────┘
                 (Any second)

Scenario:
• Accidental DELETE on Day 25 at 3:47 PM
• Restore to Day 25 at 3:46 PM (1 minute before)
```

**How it works:**
- **Continuous backups**: DynamoDB tracks all changes
- **No performance impact**: Backup process uses DynamoDB's internal mechanisms
- **Restore creates new table**: Original table unaffected
- **Includes GSIs and LSIs**: Entire table structure restored

```bash
# Enable PITR
aws dynamodb update-continuous-backups \
  --table-name MyTable \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# Restore to specific time
aws dynamodb restore-table-to-point-in-time \
  --source-table-name MyTable \
  --target-table-name MyTable-Restored \
  --restore-date-time "2026-02-06T15:46:00Z"
```

**Cost:** Additional charge (~20% of table storage cost)

### On-Demand Backup

**What it is:** Manual, full table backups that you trigger on-demand.

**Why it exists:** Long-term archival, pre-deployment snapshots, compliance.

**PITR vs On-Demand Backup:**

```
┌──────────────────────────────────────────────────────┐
│ PITR vs On-Demand Backup                             │
├──────────────────────────────────────────────────────┤
│ Point-in-Time Recovery:                              │
│ • Continuous (any second within 35 days)             │
│ • Automatic                                          │
│ • More expensive (~20% storage cost)                 │
│ • Short-term protection                              │
│                                                      │
│ On-Demand Backup:                                    │
│ • Manual snapshots                                   │
│ • Retained indefinitely (until deleted)              │
│ • Less expensive (actual storage only)               │
│ • Long-term archival                                 │
└──────────────────────────────────────────────────────┘
```

**Use cases:**
- Compliance (retain data for years)
- Major application upgrades (backup before deployment)
- Disaster recovery across regions (copy backups to other regions)

```bash
# Create on-demand backup
aws dynamodb create-backup \
  --table-name MyTable \
  --backup-name MyTable-BeforeUpgrade-2026-02-06

# Restore from backup
aws dynamodb restore-table-from-backup \
  --target-table-name MyTable-Restored \
  --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/MyTable/backup/01234567890123-abcdefgh
```

### DynamoDB Zero-ETL to OpenSearch

**What it is:** Automatically replicate DynamoDB data to Amazon OpenSearch for advanced search and analytics.

**Why it exists:**

**Problem:** DynamoDB is great for key-value lookups but limited for:
- Full-text search
- Complex aggregations
- Faceted search
- Analytics queries

**Solution:** Zero-ETL integration automatically syncs data to OpenSearch.

```
Traditional ETL Pipeline         Zero-ETL Integration

DynamoDB                          DynamoDB
    │                                 │
    ▼                                 │ Automatic
Lambda/Glue (custom code)             │ replication
    │                                 │
    ▼                                 ▼
Transform data                    OpenSearch
    │                             (No code!)
    ▼
OpenSearch

• Manual code                     • Fully managed
• Maintenance burden              • No Lambda functions
• Potential data loss             • Real-time sync
```

**How it works:**

```
DynamoDB → DynamoDB Streams → OpenSearch Integration → OpenSearch

Item written to DynamoDB
        ↓
Stream captures change
        ↓
Automatically indexed in OpenSearch
        ↓
Available for search in seconds
```

**Use cases:**
- **E-commerce**: Product catalog with full-text search and filters
- **Logging**: Store logs in DynamoDB, analyze in OpenSearch
- **User search**: Search users by name, email, bio (not just ID)

**Example workflow:**

```
DynamoDB Table: "Products"
┌────────┬──────────────┬─────────┬────────────┐
│ ItemID │ Name         │ Category│ Tags       │
├────────┼──────────────┼─────────┼────────────┤
│ P001   │ Laptop       │ Tech    │ ["work"]   │
│ P002   │ Running Shoe │ Sports  │ ["fitness"]│
└────────┴──────────────┴─────────┴────────────┘
                ↓
            Zero-ETL sync
                ↓
OpenSearch Index: "products"
{
  "_source": {
    "ItemID": "P001",
    "Name": "Laptop",
    "Category": "Tech",
    "Tags": ["work"]
  }
}

Now searchable with queries like:
• "Find all Tech products"
• "Products with tag 'work' or 'fitness'"
• Full-text search: "running"
```

**Setup:** Enable via AWS Console or CLI (no custom code required)

---

## ElastiCache {#elasticache}

### What is ElastiCache?

**Amazon ElastiCache** is a fully managed in-memory caching service that supports two open-source engines: Redis and Memcached. It provides sub-millisecond latency for frequently accessed data.

### Why ElastiCache Exists

**Problem:** Databases are persistent but slower (milliseconds)
**Solution:** In-memory cache provides microsecond access to hot data

```
Without Cache                    With ElastiCache

User request                     User request
     ▼                                ▼
Application                      Application
     ▼                                ▼
Database query                   ElastiCache check
(10-50 ms)                       (< 1 ms) ✓ Cache hit!
     │
     │                           Cache miss?
     │                                ▼
     │                           Database (10-50 ms)
     │                                ▼
     │                           Store in cache
     ▼                                ▼
Return data                      Return data

Latency: 10-50 ms every time     Latency: < 1 ms (90%+ requests)
Database load: 100%              Database load: 5-10%
```

**Common caching patterns:**

```
1. Cache-Aside (Lazy Loading)
   App → Check cache → Cache miss → Query DB → Store in cache

2. Write-Through
   App → Write to DB → Write to cache (always in sync)

3. Session Store
   App → Store user sessions in cache (fast access, auto-expiry)
```

### Redis vs Memcached

**When to choose each:**

```
┌────────────────────────────────────────────────────────┐
│ Redis vs Memcached Comparison                          │
├────────────────────────────────────────────────────────┤
│ Use Redis when you need:                               │
│ ✓ Complex data types (lists, sets, sorted sets)       │
│ ✓ Persistence (save data to disk)                     │
│ ✓ Pub/Sub messaging                                   │
│ ✓ Replication (high availability)                     │
│ ✓ Transactions                                         │
│ ✓ Geospatial data                                      │
│ ✓ Lua scripting                                        │
│                                                        │
│ Use Memcached when you need:                           │
│ ✓ Simplest caching (string key-value only)            │
│ ✓ Multi-threaded performance                          │
│ ✓ Horizontal scaling (sharding)                       │
│ ✗ No persistence needed                               │
│ ✗ No replication needed                               │
└────────────────────────────────────────────────────────┘

Summary: Redis is feature-rich, Memcached is simpler/faster for basic caching
```


### Redis (Cluster Mode, Replication)

#### Cluster Mode Disabled (Replication)

**What it is:** Single Redis cluster with one primary node and up to 5 read replicas.

```
Cluster Mode Disabled (Replication Only)

┌──────────────────────────────────────────────────┐
│ Redis Replication Group                          │
│                                                  │
│  ┌─────────────┐                                 │
│  │ Primary     │                                 │
│  │ (Read/Write)│                                 │
│  └──────┬──────┘                                 │
│         │ Async replication                      │
│         ├──────────┬──────────┐                  │
│         ▼          ▼          ▼                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │ Replica 1│ │ Replica 2│ │ Replica 3│        │
│  │(Read-only│ │(Read-only│ │(Read-only│        │
│  └──────────┘ └──────────┘ └──────────┘        │
└──────────────────────────────────────────────────┘

Failover: If primary fails, replica promoted automatically
```

**Use when:**
- Single dataset fits in memory of one node
- Need read scaling
- Need high availability

#### Cluster Mode Enabled (Sharding + Replication)

**What it is:** Data distributed across multiple shards, each with its own replication group.

```
Cluster Mode Enabled (Sharding + Replication)

┌──────────────────────────────────────────────────────┐
│ Redis Cluster                                        │
│                                                      │
│ Shard 1              Shard 2              Shard 3    │
│ ┌─────────┐         ┌─────────┐         ┌─────────┐ │
│ │Primary 1│         │Primary 2│         │Primary 3│ │
│ │ (Keys   │         │ (Keys   │         │ (Keys   │ │
│ │  A-G)   │         │  H-N)   │         │  O-Z)   │ │
│ └────┬────┘         └────┬────┘         └────┬────┘ │
│      │                   │                   │       │
│      ▼                   ▼                   ▼       │
│ ┌─────────┐         ┌─────────┐         ┌─────────┐ │
│ │Replica 1│         │Replica 2│         │Replica 3│ │
│ └─────────┘         └─────────┘         └─────────┘ │
└──────────────────────────────────────────────────────┘

Data partitioned by key hash:
  Key "apple"  → Hash → Shard 1
  Key "mango"  → Hash → Shard 2
  Key "zebra"  → Hash → Shard 3
```

**Use when:**
- Dataset > 500 GB (exceeds single node capacity)
- Need write scaling (distribute across shards)
- Need to partition data

**Scaling:**
- **Online resharding**: Add/remove shards without downtime
- **Online replica scaling**: Add/remove replicas per shard

### Memcached

**Architecture:**

```
Memcached Cluster (Multi-node)

┌────────────────────────────────────────────┐
│ Application (Client-side sharding)         │
└──────┬─────────────┬─────────────┬─────────┘
       │             │             │
       ▼             ▼             ▼
 ┌──────────┐  ┌──────────┐  ┌──────────┐
 │ Node 1   │  │ Node 2   │  │ Node 3   │
 │          │  │          │  │          │
 │ Keys:    │  │ Keys:    │  │ Keys:    │
 │ A-G      │  │ H-N      │  │ O-Z      │
 └──────────┘  └──────────┘  └──────────┘

• No replication (if node fails, data lost)
• No persistence
• Simple, fast, multi-threaded
• Horizontal scaling via sharding
```

**Use cases:**
- Simple object caching
- Session storage (with session persistence elsewhere)
- High-throughput, low-latency workloads
- Don't need persistence or high availability

### ElastiCache Serverless

**What it is:** Fully serverless, auto-scaling ElastiCache for Redis (launched 2023-2024).

**Why it exists:** Eliminate capacity planning and manual scaling.

```
Traditional ElastiCache         ElastiCache Serverless

Fixed node types                Auto-scaling capacity
┌────────────────┐             ┌────────────────┐
│ cache.r6g.large│             │ Scales 0→TBs   │
│ (13.07 GB RAM) │             │                │
│                │             │ Pay per use    │
│ Fixed cost     │             │                │
└────────────────┘             └────────────────┘

Manual scaling needed           Automatic scaling
```

**Features:**
- Automatic scaling based on workload
- Pay only for consumed resources (ECPUs - ElastiCache Processing Units)
- Maintains sub-millisecond latency
- High availability built-in

**Use when:**
- Unpredictable workloads
- Simplify operations
- Cost optimization for variable traffic

### Global Datastore (Redis)

**What it is:** Cross-region replication for Redis clusters.

**Why it exists:** Disaster recovery and low-latency reads for global users.

```
Redis Global Datastore

Primary Region (us-east-1)     Secondary Region (eu-west-1)
┌─────────────────────┐        ┌─────────────────────┐
│ Redis Cluster       │────────│ Redis Cluster       │
│ (Read + Write)      │ Async  │ (Read-Only)         │
│                     │ Replic.│                     │
└─────────────────────┘        └─────────────────────┘
         │                              │
         │                              │
    US Customers                   EU Customers
    (Read + Write)                 (Read-Only,
                                   low latency)

Replication lag: Typically < 1 second
Failover: Promote secondary to primary in < 1 minute
```

**Use cases:**
- Multi-region applications
- Disaster recovery
- Global read scaling

### Data Tiering (Redis)

**What it is:** Automatically move less-frequently accessed data from memory to SSD.

**Why it exists:** Reduce costs while maintaining fast access to hot data.

```
Redis Data Tiering

┌────────────────────────────────────────┐
│ Hot Data (Memory) - Microsecond access │
│ ┌──────────────────────────────────┐   │
│ │ Frequently accessed keys         │   │
│ │ (e.g., active user sessions)     │   │
│ └──────────────────────────────────┘   │
└────────────────────────────────────────┘
              ▲
              │ Auto-promoted on access
              ▼
┌────────────────────────────────────────┐
│ Cold Data (SSD) - Sub-millisecond      │
│ ┌──────────────────────────────────┐   │
│ │ Less-accessed keys               │   │
│ │ (e.g., old session data)         │   │
│ └──────────────────────────────────┘   │
└────────────────────────────────────────┘

Cost: SSD tier ~90% cheaper than memory
Performance: Still < 1 ms for cold data access
```

**Use when:**
- Large dataset with uneven access patterns
- Want to reduce costs without evicting data
- Need more capacity than fits in memory

**Example:** 200 GB dataset:
- Traditional: 200 GB in memory = $$$
- With tiering: 50 GB memory + 150 GB SSD = $$

---

## Other Specialized Database Services {#other-services}

AWS provides purpose-built databases for specific use cases beyond general relational and NoSQL workloads.

### DocumentDB (MongoDB Compatible)

**What it is:** Fully managed document database compatible with MongoDB API.

**Why it exists:**

**Problem:** Running self-managed MongoDB clusters requires operational overhead (sharding, replication, backups).

**Solution:** DocumentDB provides MongoDB compatibility with AWS-managed infrastructure.

```
Self-Managed MongoDB              Amazon DocumentDB
┌──────────────────────┐         ┌──────────────────────┐
│ Manual sharding      │         │ Auto-scaling         │
│ Complex replication  │         │ Built-in replication │
│ Backup management    │         │ Automated backups    │
│ Scaling challenges   │         │ Easy scaling         │
└──────────────────────┘         └──────────────────────┘
```

**Architecture:**

```
DocumentDB Cluster

┌─────────────────────────────────────────────┐
│ Cluster Storage (Shared, Auto-scaling)      │
│ Replicates across 3 AZs                     │
└──────┬──────────────┬──────────────────────┘
       │              │
       ▼              ▼
┌────────────┐  ┌────────────┐
│ Primary    │  │ Replica 1  │
│ Instance   │  │ (Read)     │
│(Read+Write)│  └────────────┘
└────────────┘

Endpoint types:
• Cluster endpoint (writes)
• Reader endpoint (reads)
```

**Compatibility:**
- MongoDB 3.6, 4.0, 5.0 APIs
- Use standard MongoDB drivers
- Most MongoDB features supported (collections, documents, indexes)

**Use cases:**
- Content management systems
- User profiles and catalogs
- Mobile/gaming backends
- Document-based applications

**Differences from MongoDB:**
- Not 100% compatible (some features differ)
- Storage decoupled from compute (like Aurora)
- Max document size: 16 MB (same as MongoDB)

### Neptune (Graph Database)

**What it is:** Fully managed graph database supporting Gremlin and SPARQL query languages.

**Why it exists:**

**Problem:** Relational databases struggle with highly connected data (social networks, recommendation engines, fraud detection).

**Solution:** Graph databases optimize for relationships.

```
Relational (Complex JOINs)        Graph (Native Relationships)
┌──────────────────────┐          ┌──────────────────────┐
│ Users Table          │          │    Alice             │
│ Friends Table        │          │    /   \             │
│ Multiple JOINs       │          │ Bob     Carol        │
│ Slow for deep paths  │          │  |       |           │
│                      │          │ Dave   Emma          │
└──────────────────────┘          └──────────────────────┘

Query: "Friends of friends?"      Direct traversal
Requires: Complex recursive JOINs Fast graph traversal
```

**Graph model example:**

```
Social Network Graph

(Alice)─[FOLLOWS]→(Bob)
   │
   └─[FOLLOWS]→(Carol)─[FOLLOWS]→(Dave)
                  │
                  └─[LIKES]→(Post: "AWS Tips")
                              ↑
                              │
                        (Emma)─┘ [LIKES]

Gremlin query: Find who Alice's friends follow
g.V().has('name','Alice')
     .out('FOLLOWS')
     .out('FOLLOWS')
     .dedup()
     .values('name')

Result: Dave (via Carol)
```

**Supported standards:**
- **Property Graph**: Gremlin (Apache TinkerPop)
- **RDF**: SPARQL (W3C standard)

**Use cases:**
- Social networks (friend recommendations)
- Fraud detection (pattern recognition)
- Knowledge graphs (Wikipedia-like data)
- Network/IT operations (dependency mapping)
- Recommendation engines

**Architecture:**
- Shared storage across 3 AZs
- Read replicas for high availability
- Up to 15 read replicas

### Timestream (Time Series Database)

**What it is:** Purpose-built time-series database for IoT, DevOps, and analytics workloads.

**Why it exists:**

**Problem:** Traditional databases inefficient for time-series data (IoT sensors, metrics, logs).

**Solution:** Optimized storage and queries for timestamped data.

```
Traditional Database              Timestream
┌──────────────────────┐         ┌──────────────────────┐
│ timestamp | metric   │         │ Automatic tiering     │
│ 10:00:00  | 25.3     │         │ Memory (recent data)  │
│ 10:00:01  | 25.4     │         │ ↓                     │
│ ... millions of rows │         │ Magnetic (historical) │
│                      │         │                       │
│ Slow aggregations    │         │ Fast time-based       │
│ Expensive storage    │         │ queries & aggregates  │
└──────────────────────┘         └──────────────────────┘
```

**Data lifecycle:**

```
Timestream Data Tiering

New data → Memory store (< 1 day) → Fast queries
              ↓ Automatic tiering
           Magnetic store (days→years) → Cost-efficient storage

┌────────────────────────────────────────────────┐
│ Query spans both tiers seamlessly             │
│ SELECT avg(temperature)                        │
│ FROM sensors                                   │
│ WHERE time > ago(30d)  ← Queries both tiers   │
└────────────────────────────────────────────────┘
```

**Built-in analytics:**
```sql
-- Time-based aggregations (optimized)
SELECT bin(time, 1h) as hour,
       avg(cpu_utilization) as avg_cpu
FROM metrics
WHERE time > ago(7d)
GROUP BY bin(time, 1h)
ORDER BY hour DESC;

-- Interpolation (fill missing data points)
SELECT CREATE_TIME_SERIES(time, value) as interpolated
FROM sensor_data
WHERE measure_name = 'temperature';
```

**Use cases:**
- IoT sensor data (temperature, humidity, location)
- DevOps metrics (CPU, memory, network)
- Application monitoring (request rates, latencies)
- Financial tick data

**Cost model:**
- Pay for data ingested
- Pay for data scanned in queries
- Automatic data compression and tiering

### QLDB (Quantum Ledger Database)

**What it is:** Immutable, cryptographically verifiable transaction log database.

**Why it exists:**

**Problem:** Need tamper-proof audit trail where changes cannot be altered or deleted.

**Solution:** QLDB provides append-only ledger with cryptographic verification.

```
Traditional Database             QLDB (Immutable Ledger)
┌──────────────────┐            ┌──────────────────────┐
│ UPDATE allowed   │            │ Append-only          │
│ DELETE allowed   │            │ No UPDATE/DELETE     │
│ No audit trail   │            │ Complete history     │
│ Data can change  │            │ Cryptographic proof  │
└──────────────────┘            └──────────────────────┘

Example ledger:
┌────────┬──────────┬─────────┬──────────────┐
│ Version│ Timestamp│ Data    │ Hash         │
├────────┼──────────┼─────────┼──────────────┤
│ 0      │ 10:00:00 │ Bal:100 │ hash_abc123  │
│ 1      │ 10:15:00 │ Bal:80  │ hash_def456  │ ← Linked
│ 2      │ 10:30:00 │ Bal:120 │ hash_ghi789  │ ← Chain
└────────┴──────────┴─────────┴──────────────┘

Cannot alter history - cryptographically protected
```

**How verification works:**

```
Cryptographic Verification

Transaction 1 → Hash A
Transaction 2 → Hash B (includes Hash A)
Transaction 3 → Hash C (includes Hash B)
                  ▲
                  Any tampering breaks the chain

QLDB can prove:
• Document existed at timestamp X
• Document has not been altered
• Complete change history available
```

**Use cases:**
- **Finance**: Transaction history, regulatory compliance
- **Supply chain**: Track product provenance
- **HR systems**: Immutable employee records
- **Insurance**: Claims processing audit trail
- **Government**: Vehicle registrations, identity records

**Query language:** PartiQL (same as DynamoDB)

```sql
-- Query current state
SELECT * FROM Accounts WHERE AccountID = 'A123';

-- Query historical versions
SELECT * FROM history(Accounts)
WHERE AccountID = 'A123';

-- Verify document integrity
SELECT blockAddress, hash FROM _ql_committed_Accounts;
```

**Key characteristics:**
- **Immutable**: No UPDATE or DELETE operations
- **Transparent**: Full history always available
- **Verifiable**: Cryptographic proof of data integrity
- **SQL-like**: PartiQL for queries

### Keyspaces (Cassandra Compatible)

**What it is:** Fully managed Cassandra-compatible database service.

**Why it exists:**

**Problem:** Apache Cassandra is powerful for distributed, high-throughput workloads but complex to manage (cluster sizing, replication, repairs).

**Solution:** Keyspaces provides Cassandra compatibility without operational overhead.

```
Self-Managed Cassandra          Amazon Keyspaces
┌──────────────────────┐       ┌──────────────────────┐
│ Manual cluster       │       │ Serverless           │
│ Capacity planning    │       │ Auto-scaling         │
│ Node management      │       │ Fully managed        │
│ Repair operations    │       │ No node management   │
└──────────────────────┘       └──────────────────────┘
```

**Cassandra data model:**

```
Keyspace: "ecommerce"
  └─ Table: "orders"
      ├─ Partition Key: customer_id (distributes data)
      ├─ Clustering Key: order_date (sorts within partition)
      └─ Data: order details

CQL (Cassandra Query Language):
CREATE TABLE orders (
  customer_id UUID,
  order_date TIMESTAMP,
  order_id UUID,
  total DECIMAL,
  PRIMARY KEY (customer_id, order_date)
);

SELECT * FROM orders
WHERE customer_id = '123e4567-e89b-12d3-a456-426614174000'
AND order_date > '2026-01-01';
```

**Features:**
- **CQL support**: Use standard Cassandra drivers
- **Serverless**: Pay per request (or provision throughput)
- **Point-in-time recovery**: 35-day retention
- **Encryption**: At-rest and in-transit
- **Multi-region replication**: Optional (additional cost)

**Use cases:**
- High-throughput applications (millions of writes/sec)
- Time-series data (IoT, logs)
- Wide-column data models
- Applications already using Cassandra

**Compatibility:**
- Cassandra 3.11 API (as of 2026)
- Most CQL features supported
- Use existing Cassandra drivers

### MemoryDB for Redis

**What it is:** Redis-compatible, in-memory database with durability (data persisted to disk).

**Why it exists:**

**ElastiCache vs MemoryDB:**

```
ElastiCache for Redis            MemoryDB for Redis
┌──────────────────────┐         ┌──────────────────────┐
│ Cache (ephemeral)    │         │ Primary database     │
│ Optional persistence │         │ Durable (always)     │
│ Async replication    │         │ Strong consistency   │
│ Microsec latency     │         │ Microsec latency     │
│                      │         │                      │
│ Use as: Cache layer  │         │ Use as: Main DB      │
└──────────────────────┘         └──────────────────────┘
```

**Key difference:** MemoryDB is a **primary database** (not just cache), with durability guarantees.

**Architecture:**

```
MemoryDB Cluster

┌────────────────────────────────────────────────┐
│ In-Memory Data (Multi-AZ)                      │
│         ↓                                      │
│    Transaction Log (Durable)                   │
│         ↓                                      │
│    Replicated across AZs                       │
└────────────────────────────────────────────────┘

Every write:
1. Committed to transaction log (persistent)
2. Replicated to in-memory replicas
3. Acknowledged to client

Recovery: Fast restart from transaction log
```

**Use cases:**
- **Primary database** for applications needing:
  - Sub-millisecond latency
  - Redis data structures
  - Durability guarantees
- Gaming leaderboards (persistent)
- Session stores (cannot afford data loss)
- Real-time analytics dashboards

**Comparison to ElastiCache:**
- **ElastiCache**: Cache layer, optional persistence
- **MemoryDB**: Primary database, always durable

**Redis compatibility:**
- Redis 6.2+ compatible
- Supports Redis data structures (strings, lists, sets, sorted sets, hashes)
- Pub/Sub, transactions, Lua scripting

---

## Choosing the Right Database Service

**Decision framework:**

```
┌──────────────────────────────────────────────────────────┐
│ Database Selection Guide                                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Need ACID transactions + SQL?                           │
│  → RDS (MySQL, PostgreSQL, etc.) or Aurora              │
│                                                          │
│ Need extreme performance for relational?                │
│  → Aurora                                                │
│                                                          │
│ Need massive scale + flexible schema?                   │
│  → DynamoDB                                              │
│                                                          │
│ Need sub-millisecond caching?                           │
│  → ElastiCache (Redis or Memcached)                     │
│                                                          │
│ Need durable in-memory database?                        │
│  → MemoryDB for Redis                                   │
│                                                          │
│ Working with documents/JSON?                            │
│  → DocumentDB (MongoDB) or DynamoDB                     │
│                                                          │
│ Need graph relationships?                               │
│  → Neptune                                               │
│                                                          │
│ Time-series data (IoT, metrics)?                        │
│  → Timestream                                            │
│                                                          │
│ Need immutable audit trail?                             │
│  → QLDB                                                  │
│                                                          │
│ Already using Cassandra?                                │
│  → Keyspaces                                             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Multi-database architecture example:**

```
Modern Application Architecture

┌─────────────────────────────────────────────────┐
│ Application Layer                               │
└──┬──────────┬──────────┬──────────┬─────────┬──┘
   │          │          │          │         │
   ▼          ▼          ▼          ▼         ▼
┌─────┐  ┌─────────┐ ┌────────┐ ┌──────┐ ┌───────┐
│ RDS │  │DynamoDB │ │ElastiC.│ │OpenS.│ │Neptune│
│     │  │         │ │        │ │      │ │       │
│User │  │Sessions │ │Cache   │ │Search│ │Social │
│Data │  │Orders   │ │        │ │      │ │Graph  │
└─────┘  └─────────┘ └────────┘ └──────┘ └───────┘

• RDS: User accounts, profiles (structured, ACID)
• DynamoDB: Session data, high-throughput orders
• ElastiCache: Frequently accessed data
• OpenSearch: Product search, full-text
• Neptune: Friend relationships, recommendations
```

**Cost optimization tips:**
1. **Right-size instances**: Don't over-provision
2. **Use reserved instances/savings plans**: For predictable workloads (30-70% savings)
3. **Leverage serverless**: Aurora Serverless, DynamoDB On-Demand, Keyspaces for variable workloads
4. **Enable auto-scaling**: Only pay for what you use
5. **Monitor and optimize**: Use CloudWatch, Cost Explorer to identify waste
6. **Data tiering**: Use Aurora I/O-Optimized, ElastiCache data tiering appropriately

---

## Best Practices Summary

### Security
- **Encryption**: Enable at-rest and in-transit encryption for all databases
- **IAM authentication**: Use IAM database auth where supported (RDS, Aurora)
- **VPC isolation**: Deploy databases in private subnets
- **Secret Manager**: Store credentials in AWS Secrets Manager (auto-rotation)
- **Least privilege**: Grant minimal permissions via IAM policies

### High Availability
- **Multi-AZ**: Enable for production workloads (RDS, Aurora, ElastiCache)
- **Read replicas**: Scale reads and provide failover targets
- **Automated backups**: Configure appropriate retention periods
- **Global tables/datastores**: For disaster recovery across regions

### Performance
- **Right query patterns**: Design schema around access patterns (especially DynamoDB)
- **Indexes**: Use GSIs/LSIs (DynamoDB), proper indexing (RDS/Aurora)
- **Caching**: Layer ElastiCache for frequently accessed data
- **Connection pooling**: Use RDS Proxy for serverless architectures
- **Monitoring**: Enable Performance Insights, Enhanced Monitoring, CloudWatch metrics

### Cost Management
- **Reserved capacity**: For predictable workloads
- **Serverless**: For unpredictable or low-traffic workloads
- **Data lifecycle**: Use TTL (DynamoDB), appropriate backup retention
- **Right-size**: Continuously review and adjust instance sizes
- **Delete unused resources**: Snapshots, old tables, idle instances

---

## Conclusion

AWS provides a comprehensive suite of database services designed for different use cases:

- **Relational workloads**: RDS and Aurora provide managed MySQL, PostgreSQL, Oracle, and SQL Server with high availability and performance
- **NoSQL at scale**: DynamoDB offers predictable performance for key-value and document data at any scale
- **Caching**: ElastiCache and DAX provide microsecond latency for hot data
- **Specialized needs**: Purpose-built databases (Neptune, Timestream, QLDB, DocumentDB, Keyspaces, MemoryDB) optimize for specific access patterns

**Key takeaway**: Choose the database that matches your access patterns, consistency requirements, and scale needs—don't force all workloads into a single database type.

The modern approach is **polyglot persistence**: using the best database for each workload within your application architecture.