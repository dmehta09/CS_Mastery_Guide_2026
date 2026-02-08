# AWS Mental Models for Mastery

## 1. The Building Analogy - Understanding AWS Structure

```
AWS = A Giant Building Complex
├── Regions = Different Cities (us-east-1 = New York, eu-west-1 = Dublin)
│   ├── Availability Zones = Buildings in that City (us-east-1a, 1b, 1c)
│   │   ├── Data Centers = Floors in the Building
│   │   │   └── Your Resources = Rooms/Offices
│   │   └── Connected by private fiber (millisecond latency)
│   └── Isolated from other cities for disaster protection
├── Edge Locations = Branch Offices worldwide (CloudFront, Route 53)
├── Local Zones = Satellite Offices in specific metros
└── Wavelength = Offices inside telecom buildings (5G edge)
```

**Key Insight**: Always think "which city, which building, which floor" when placing resources.

---

## 2. The Company Org Chart - IAM Mental Model

```
AWS Account = Your Company
│
├── Root User = CEO (All-powerful, rarely used, MFA required!)
│
├── IAM Users = Employees with ID badges
│   └── Login credentials + programmatic keys
│
├── IAM Groups = Departments (Developers, Admins, Finance)
│   └── Attach policies to groups, not individual users
│
├── IAM Roles = Job Functions (anyone can "wear the hat")
│   ├── EC2 Role = "Server Administrator hat"
│   ├── Lambda Role = "Function Executor hat"
│   └── Cross-Account Role = "Consultant from partner company"
│
└── IAM Policies = Rule Books
    ├── Identity-based = "What can this person/role do?"
    └── Resource-based = "Who can access this resource?"
```

### Policy Evaluation Flow
```
Request comes in
       │
       ▼
┌──────────────────┐
│ Explicit DENY?   │──Yes──► DENIED (Always wins)
└────────┬─────────┘
         │ No
         ▼
┌──────────────────┐
│ Explicit ALLOW?  │──No───► DENIED (Implicit deny)
└────────┬─────────┘
         │ Yes
         ▼
     ALLOWED
```

**Golden Rules**:
- Explicit Deny > Explicit Allow > Implicit Deny
- Least privilege: Start with nothing, add permissions
- Use roles, not long-term credentials

---

## 3. The Highway System - VPC Networking

```
VPC = Your Private Highway Network
│
├── CIDR Block = Total address space (e.g., 10.0.0.0/16 = 65,536 IPs)
│
├── Subnets = Exit ramps to specific areas
│   ├── Public Subnet = Has route to Internet Gateway
│   │   └── Resources get public IPs, directly accessible
│   └── Private Subnet = No direct internet route
│       └── Resources hidden, use NAT for outbound
│
├── Route Tables = GPS/Navigation rules
│   ├── Local route = "Stay on your highway"
│   └── 0.0.0.0/0 → IGW = "Go to internet via this exit"
│
├── Internet Gateway (IGW) = Toll booth to public internet
│   └── One per VPC, highly available
│
├── NAT Gateway = One-way mirror
│   └── Private resources can see out, internet can't see in
│
├── Security Groups = Personal bodyguards (stateful)
│   └── "Allow traffic from X" - return traffic auto-allowed
│
└── NACLs = Building security at subnet entrance (stateless)
    └── Must explicitly allow both inbound AND outbound
```

### Traffic Flow Decision
```
Internet → Your App:

    Internet
        │
        ▼
  ┌─────────────┐
  │   IGW       │ (Must exist for public access)
  └─────┬───────┘
        │
        ▼
  ┌─────────────┐
  │   NACL      │ (Subnet-level filter)
  └─────┬───────┘
        │
        ▼
  ┌─────────────┐
  │ Route Table │ (Points to subnet)
  └─────┬───────┘
        │
        ▼
  ┌─────────────┐
  │ Security Grp│ (Instance-level filter)
  └─────┬───────┘
        │
        ▼
   Your EC2/Service
```

---

## 4. The Storage Warehouse - S3 Mental Model

```
S3 = Infinite Warehouse with Smart Storage

┌─────────────────────────────────────────────────────────────┐
│  BUCKET (globally unique name = warehouse address)          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  OBJECTS (files with metadata)                          ││
│  │  ├── Key = full path (folder/subfolder/file.txt)       ││
│  │  ├── Value = actual data (up to 5TB)                   ││
│  │  ├── Metadata = labels and properties                  ││
│  │  └── Version ID = snapshot identifier                  ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘

Storage Classes = Different warehouse sections:
┌──────────────────────┬─────────────┬───────────────────────┐
│ Class                │ Access      │ Use Case              │
├──────────────────────┼─────────────┼───────────────────────┤
│ Standard             │ Instant     │ Frequently accessed   │
│ Intelligent-Tiering  │ Auto-moves  │ Unknown patterns      │
│ Standard-IA          │ Instant     │ Monthly access        │
│ One Zone-IA          │ Instant     │ Reproducible data     │
│ Glacier Instant      │ Instant     │ Archive, quick access │
│ Glacier Flexible     │ Min-Hours   │ Archive, can wait     │
│ Glacier Deep Archive │ 12-48 Hours │ Compliance, rarely    │
│ Express One Zone     │ <10ms       │ ML training, analytics│
└──────────────────────┴─────────────┴───────────────────────┘
```

### S3 Access Control Layers
```
Request → Can I access this object?

Layer 1: Block Public Access (Account/Bucket level kill switch)
    │
    ▼ (if not blocked)
Layer 2: Bucket Policy (Resource-based, JSON)
    │
    ▼
Layer 3: IAM Policy (Identity-based)
    │
    ▼
Layer 4: Object ACL (Legacy, avoid using)
    │
    ▼
Access Granted or Denied
```

---

## 5. The Compute Spectrum - Choose Your Compute

```
More Control ◄─────────────────────────────────────► Less Control
More Ops Work                                       Less Ops Work

┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
│ Bare    │   EC2   │   ECS   │   EKS   │ Fargate │ Lambda  │
│ Metal   │         │ on EC2  │ Managed │         │         │
├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ You     │ You     │ You     │ You     │ AWS     │ AWS     │
│ manage  │ manage  │ manage  │ manage  │ manages │ manages │
│ ALL     │ OS +    │ cluster │ K8s +   │ infra   │ ALL     │
│         │ up      │ + tasks │ nodes   │         │         │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘

Cost Model:
├── EC2: Pay for uptime (even if idle)
├── Fargate: Pay for vCPU + memory per second
└── Lambda: Pay only for execution time (100ms increments)
```

### Compute Decision Tree
```
                    ┌──────────────────┐
                    │ What's your app? │
                    └────────┬─────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   Stateless?          Containerized?       Legacy/Monolith?
        │                    │                    │
        ▼                    │                    ▼
   ┌─────────┐               │               ┌─────────┐
   │ Lambda  │◄──Short       │               │   EC2   │
   │ or      │   running     │               └─────────┘
   │ Fargate │   (<15min)    │
   └─────────┘               │
                             ▼
                    ┌────────────────┐
                    │ Need K8s APIs? │
                    └───────┬────────┘
                     Yes    │    No
                    ┌───────┴────────┐
                    ▼                ▼
               ┌─────────┐     ┌─────────┐
               │   EKS   │     │   ECS   │
               └─────────┘     └─────────┘
                    │                │
                    ▼                ▼
            Want to manage    Want to manage
            worker nodes?     EC2 instances?
              │     │           │     │
             Yes    No         Yes    No
              │     │           │     │
              ▼     ▼           ▼     ▼
           Managed  EKS      ECS on  ECS on
           Node Grp Fargate  EC2     Fargate
```

---

## 6. The Database Selection - Right Tool for the Job

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA TYPE DECISION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Structured Data (SQL)?                                         │
│   ├── Transactions important? ─────────► RDS / Aurora           │
│   ├── Massive scale needed? ───────────► Aurora Serverless v2   │
│   └── Global distribution? ────────────► Aurora Global Database │
│                                                                  │
│   Semi-structured (NoSQL)?                                       │
│   ├── Key-value + simple queries? ─────► DynamoDB               │
│   ├── Document store (MongoDB)? ───────► DocumentDB             │
│   └── Wide column (Cassandra)? ────────► Keyspaces              │
│                                                                  │
│   Specialized Data?                                              │
│   ├── Graph relationships? ────────────► Neptune                │
│   ├── Time series (IoT, logs)? ────────► Timestream             │
│   ├── Search (full-text)? ─────────────► OpenSearch             │
│   ├── Caching (low latency)? ──────────► ElastiCache / DAX      │
│   └── Ledger (immutable)? ─────────────► QLDB                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### RDS vs Aurora vs DynamoDB
```
                    RDS                Aurora              DynamoDB
                    ───                ──────              ────────
Type:               Traditional SQL    Cloud-native SQL    NoSQL Key-Value

Scaling:            Vertical           Auto (compute)      Infinite horizontal
                    (bigger box)       + read replicas     (partition magic)

Availability:       Multi-AZ           Multi-AZ + Global   Multi-Region
                    (standby)          (read replicas)     (Global Tables)

Price Model:        Instance hours     Instance + I/O      Capacity units
                                       or I/O-Optimized    (RCU/WCU or on-demand)

Best For:           Lift & shift       High-perf SQL       Massive scale
                    existing apps      cloud-native        simple access patterns
```

---

## 7. The Event Flow - Messaging & Integration

```
               PRODUCER                              CONSUMER
              (Sender)                               (Receiver)
                 │                                      ▲
                 │         Choose Your Pattern:         │
                 │                                      │
    ┌────────────┼──────────────────────────────────────┼────────────┐
    │            ▼                                      │            │
    │   ┌────────────────┐                              │            │
    │   │      SQS       │  Queue (Point-to-Point)     │            │
    │   │   ┌─┬─┬─┬─┐    │  • Messages wait in line    │            │
    │   │   │►│►│►│►│────┼──• Each message → 1 consumer│            │
    │   │   └─┴─┴─┴─┘    │  • Decoupling + buffering   │            │
    │   └────────────────┘                              │            │
    │                                                   │            │
    │   ┌────────────────┐                              │            │
    │   │      SNS       │  Pub/Sub (Fan-out)          │            │
    │   │       ┌─┐      │  • 1 message → N subscribers │            │
    │   │      ┌┴─┴┐     │  • Push to SQS, Lambda, HTTP│            │
    │   │     ┌┴───┴┐────┼──• No persistence           │            │
    │   │    ┌┴─────┴┐   │                              │            │
    │   └────────────────┘                              │            │
    │                                                   │            │
    │   ┌────────────────┐                              │            │
    │   │  EventBridge   │  Event Bus (Smart Router)   │            │
    │   │  ════════════  │  • Rules filter & route     │            │
    │   │  │ Rules │─────┼──• Schema registry          │            │
    │   │  │ │ │ │ │     │  • Archive & replay         │            │
    │   │  ▼ ▼ ▼ ▼ ▼     │  • 35+ AWS service sources  │            │
    │   └────────────────┘                              │            │
    │                                                   │            │
    │   ┌────────────────┐                              │            │
    │   │    Kinesis     │  Streaming (Real-time)      │            │
    │   │ ═══════════════│  • Ordered, replayable      │            │
    │   │ │█│█│█│█│█│█│──┼──• Multiple consumers       │            │
    │   │ shard  shard   │  • Analytics on the fly     │            │
    │   └────────────────┘                              │            │
    └───────────────────────────────────────────────────────────────┘
```

### When to Use What
```
┌─────────────────┬──────────────────────────────────────────────┐
│ Pattern         │ Use Case                                      │
├─────────────────┼──────────────────────────────────────────────┤
│ SQS             │ Decouple services, handle spikes, async jobs │
│ SNS             │ Notify multiple subscribers, alerts          │
│ SNS + SQS       │ Fan-out to multiple queues                   │
│ EventBridge     │ Event-driven architecture, AWS integrations  │
│ Kinesis Streams │ Real-time analytics, ordered events, replay  │
│ Kinesis Firehose│ Streaming to S3/Redshift (managed ETL)      │
│ Step Functions  │ Orchestrate workflows, state machines        │
│ MSK (Kafka)     │ Existing Kafka apps, high-throughput streams │
└─────────────────┴──────────────────────────────────────────────┘
```

---

## 8. The Security Onion - Defense in Depth

```
                    ┌─────────────────────────────────────┐
                    │           EDGE PROTECTION            │
                    │  (Shield, WAF, CloudFront)          │
                    │    DDoS, SQL injection, XSS         │
                    │                                      │
                    │   ┌─────────────────────────────┐   │
                    │   │      NETWORK LAYER          │   │
                    │   │  (VPC, NACLs, Security Grps)│   │
                    │   │    Firewall rules           │   │
                    │   │                             │   │
                    │   │   ┌─────────────────────┐   │   │
                    │   │   │   IDENTITY LAYER   │   │   │
                    │   │   │  (IAM, STS, SSO)   │   │   │
                    │   │   │  Who can do what   │   │   │
                    │   │   │                     │   │   │
                    │   │   │   ┌─────────────┐   │   │   │
                    │   │   │   │    DATA     │   │   │   │
                    │   │   │   │  (KMS, ACM) │   │   │   │
                    │   │   │   │ Encryption  │   │   │   │
                    │   │   │   └─────────────┘   │   │   │
                    │   │   └─────────────────────┘   │   │
                    │   └─────────────────────────────┘   │
                    └─────────────────────────────────────┘

Detection & Response:
┌────────────┬────────────────────────────────────────────┐
│ GuardDuty  │ Threat detection (ML-based anomalies)     │
│ Inspector  │ Vulnerability scanning (EC2, Lambda, ECR) │
│ Macie      │ Sensitive data discovery in S3            │
│ Detective  │ Investigation & root cause analysis       │
│ Security Hub │ Aggregated view + compliance standards │
└────────────┴────────────────────────────────────────────┘
```

### Encryption Mental Model
```
Encryption at Rest (stored data):
┌─────────────────────────────────────────────────────────┐
│  Your Data  ───►  KMS Key  ───►  Encrypted Data        │
│                                                         │
│  Key Types:                                             │
│  ├── AWS Managed: AWS creates & manages (easy)         │
│  ├── Customer Managed: You control (rotation, policy)  │
│  └── External: Your HSM via External Key Store         │
└─────────────────────────────────────────────────────────┘

Encryption in Transit (moving data):
┌─────────────────────────────────────────────────────────┐
│  Client ◄──── TLS/SSL ────► AWS Service                │
│               │                                         │
│  ACM provides free certificates for AWS services        │
│  (ALB, CloudFront, API Gateway)                        │
└─────────────────────────────────────────────────────────┘
```

---

## 9. The Monitoring Dashboard - Observability Stack

```
                    ┌─────────────────────────────────────┐
                    │         WHAT TO OBSERVE              │
                    └─────────────────────────────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           ▼                        ▼                        ▼
    ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
    │   METRICS   │          │    LOGS     │          │   TRACES    │
    │ (Numbers)   │          │  (Events)   │          │ (Requests)  │
    │ CloudWatch  │          │  CloudWatch │          │   X-Ray     │
    │  Metrics    │          │    Logs     │          │             │
    └─────────────┘          └─────────────┘          └─────────────┘
           │                        │                        │
           ▼                        ▼                        ▼
    CPU at 80%?             Error in line 42?         Request took 5s
    Memory spike?           User login failed?        Bottleneck at DB?
    Request count?          API returned 500?         Lambda cold start?

    ─────────────────────────────────────────────────────────────────
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │             ACTIONS                  │
                    ├─────────────────────────────────────┤
                    │ CloudWatch Alarms → SNS → PagerDuty │
                    │ EventBridge → Lambda → Auto-remediate│
                    │ Dashboard → Visual → Human Decision │
                    └─────────────────────────────────────┘
```

### CloudWatch Metrics Flow
```
Your App/Service
       │
       │ Emits metrics (default or custom)
       ▼
┌────────────────┐
│ CloudWatch     │
│ Metrics        │◄── Aggregated by namespace, dimension
├────────────────┤
│ Metric Math    │◄── Combine/transform metrics
├────────────────┤
│ Alarms         │◄── Threshold triggers
└───────┬────────┘
        │
        ▼
┌────────────────┐    ┌────────────────┐
│ SNS Topic      │───►│ Lambda/Email   │
└────────────────┘    │ Auto Scaling   │
                      │ EC2 Actions    │
                      └────────────────┘
```

---

## 10. The Cost Optimization Pyramid

```
                         ▲
                        /│\
                       / │ \        ARCHITECTURE OPTIMIZATION
                      /  │  \       • Serverless where possible
                     /   │   \      • Right-size everything
                    /    │    \     • Multi-AZ only where needed
                   /─────┼─────\
                  /      │      \   PRICING MODEL OPTIMIZATION
                 /       │       \  • Reserved Instances (1-3 yr)
                /        │        \ • Savings Plans
               /         │         \• Spot Instances (flexible workloads)
              /──────────┼──────────\
             /           │           \ RESOURCE OPTIMIZATION
            /            │            \• Delete unused resources
           /             │             \• Lifecycle policies (S3)
          /              │              \• Scheduled start/stop
         /───────────────┼───────────────\
        /                │                \ VISIBILITY
       /                 │                 \• Cost Explorer
      /                  │                  \• Budgets & Alerts
     /                   │                   \• Tags for allocation
    ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔
```

### Cost Decision: Compute Pricing
```
┌──────────────────────────────────────────────────────────────────┐
│                  EC2 PRICING DECISION TREE                        │
└──────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
       Predictable Load?              Variable/Spiky?
              │                               │
              ▼                               ▼
    ┌─────────────────┐              ┌─────────────────┐
    │Can commit 1-3yr?│              │ Can be stopped? │
    └────────┬────────┘              └────────┬────────┘
        Yes  │  No                       Yes  │  No
             │                               │
    ┌────────┴────────┐              ┌───────┴───────┐
    ▼                 ▼              ▼               ▼
 Reserved          Savings        Spot           On-Demand
 Instances         Plans         (up to 90%      (full price)
 (specific         (flexible     discount)
 instance)         compute)

Savings:
├── Reserved: Up to 72% (specific instance type)
├── Savings Plans: Up to 66% (flexible across instances)
├── Spot: Up to 90% (can be interrupted)
└── On-Demand: 0% (pay as you go)
```

---

## 11. The High Availability Blueprint

```
┌─────────────────────────────────────────────────────────────────┐
│                       REGION: us-east-1                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    AZ-a (us-east-1a)              AZ-b (us-east-1b)            │
│    ┌─────────────────┐            ┌─────────────────┐           │
│    │  Public Subnet  │            │  Public Subnet  │           │
│    │  ┌───────────┐  │            │  ┌───────────┐  │           │
│    │  │    NAT    │  │            │  │    NAT    │  │           │
│    │  └───────────┘  │            │  └───────────┘  │           │
│    │  ┌───────────┐  │            │  ┌───────────┐  │           │
│    │  │    ALB    │◄─┼────────────┼─►│    ALB    │  │           │
│    │  └───────────┘  │            │  └───────────┘  │           │
│    └────────┬────────┘            └────────┬────────┘           │
│             │                              │                     │
│    ┌────────┴────────┐            ┌────────┴────────┐           │
│    │ Private Subnet  │            │ Private Subnet  │           │
│    │  ┌───────────┐  │            │  ┌───────────┐  │           │
│    │  │   EC2/ECS │  │◄──ASG────►│  │   EC2/ECS │  │           │
│    │  └───────────┘  │            │  └───────────┘  │           │
│    └────────┬────────┘            └────────┬────────┘           │
│             │                              │                     │
│    ┌────────┴────────┐            ┌────────┴────────┐           │
│    │  Data Subnet    │            │  Data Subnet    │           │
│    │  ┌───────────┐  │            │  ┌───────────┐  │           │
│    │  │RDS Primary│  │◄─Sync────►│  │RDS Standby│  │           │
│    │  └───────────┘  │            │  └───────────┘  │           │
│    └─────────────────┘            └─────────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Key HA Components:
├── Multi-AZ: Survive data center failure
├── Auto Scaling: Replace failed instances
├── ALB: Health checks + traffic distribution
├── RDS Multi-AZ: Automatic failover (<60s)
└── S3: 99.999999999% durability (11 9's)
```

---

## 12. The DR Strategy Spectrum

```
Lower Cost ◄──────────────────────────────────────► Higher Cost
Higher RTO/RPO                                   Lower RTO/RPO

┌─────────────┬─────────────┬─────────────┬─────────────┐
│   BACKUP    │   PILOT     │    WARM     │  MULTI-SITE │
│  & RESTORE  │   LIGHT     │   STANDBY   │ACTIVE/ACTIVE│
├─────────────┼─────────────┼─────────────┼─────────────┤
│             │ ┌───┐       │ ┌───┐       │ ┌───┐ ┌───┐ │
│  Backups    │ │min│       │ │med│       │ │100│ │100│ │
│  in S3     │ │env│       │ │env│       │ │ % │ │ % │ │
│  ┌───┐      │ └───┘       │ └───┘       │ └───┘ └───┘ │
│  │ S3│      │             │             │             │
│  └───┘      │             │             │             │
├─────────────┼─────────────┼─────────────┼─────────────┤
│ RTO: Hours  │ RTO: 10min+ │ RTO: Minutes│ RTO: ~0     │
│ RPO: Hours  │ RPO: Minutes│ RPO: Seconds│ RPO: ~0     │
├─────────────┼─────────────┼─────────────┼─────────────┤
│ Restore     │ Scale up    │ Scale up    │ Already     │
│ from backup │ minimal     │ running     │ running     │
│             │ infra       │ smaller env │ full env    │
└─────────────┴─────────────┴─────────────┴─────────────┘

RTO = Recovery Time Objective (how long to recover?)
RPO = Recovery Point Objective (how much data can we lose?)
```

---

## 13. The Serverless Event Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVERLESS APPLICATION                        │
└─────────────────────────────────────────────────────────────────┘

                        ┌──────────────┐
    User Request ──────►│ API Gateway  │
                        │   (REST/WS)  │
                        └──────┬───────┘
                               │
                               ▼
                        ┌──────────────┐
                        │   Lambda     │◄── Auto-scales 0 → thousands
                        │  (Compute)   │
                        └──────┬───────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  DynamoDB    │      │     S3       │      │     SQS      │
│  (Data)      │      │  (Storage)   │      │  (Queue)     │
└──────────────┘      └──────┬───────┘      └──────┬───────┘
                             │                      │
                             ▼                      ▼
                      S3 Event Trigger       SQS Event Source
                             │                      │
                             └──────────┬───────────┘
                                        │
                                        ▼
                               ┌──────────────┐
                               │   Lambda     │
                               │ (Processing) │
                               └──────┬───────┘
                                      │
                                      ▼
                               ┌──────────────┐
                               │ Step Function│◄── Orchestration
                               │  (Workflow)  │
                               └──────────────┘
```

### Lambda Concurrency Model
```
Request comes in:
        │
        ▼
Is there a warm container?
        │
   Yes ─┴─ No
    │       │
    │       ▼
    │   Cold Start (init code runs)
    │       │
    └───────┤
            ▼
    Handler executes
            │
            ▼
    Container stays warm (~15 min)

Concurrency:
├── Reserved: Guaranteed capacity for this function
├── Provisioned: Pre-warmed containers (no cold start)
└── Unreserved: Shared pool (up to account limit)
```

---

## 14. The EKS Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         EKS CLUSTER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              CONTROL PLANE (AWS Managed)                 │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐              │    │
│  │  │API Server │ │Controller │ │  Scheduler │              │    │
│  │  └───────────┘ └───────────┘ └───────────┘              │    │
│  │                      │                                   │    │
│  │              ┌───────┴───────┐                          │    │
│  │              │     etcd      │ (Multi-AZ)               │    │
│  │              └───────────────┘                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                    VPC Endpoint                                  │
│                           │                                      │
│  ┌────────────────────────┴─────────────────────────────────┐   │
│  │                DATA PLANE (You manage or Fargate)         │   │
│  │                                                           │   │
│  │   AZ-a                    AZ-b                    AZ-c   │   │
│  │   ┌──────────────┐       ┌──────────────┐       ┌──────┐│   │
│  │   │ Node Group   │       │ Node Group   │       │Fargate││   │
│  │   │┌────┐┌────┐  │       │┌────┐┌────┐  │       │Profile││   │
│  │   ││Pod ││Pod │  │       ││Pod ││Pod │  │       │┌────┐ ││   │
│  │   │└────┘└────┘  │       │└────┘└────┘  │       ││Pod │ ││   │
│  │   └──────────────┘       └──────────────┘       │└────┘ ││   │
│  │                                                  └──────┘│   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  AWS Integrations:                                               │
│  ├── ALB Ingress Controller ─► Creates ALBs from Ingress        │
│  ├── IRSA ─► IAM Roles for Service Accounts                     │
│  ├── CSI Drivers ─► EBS, EFS, FSx volumes                       │
│  └── Karpenter/Cluster Autoscaler ─► Node scaling               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 15. The Well-Architected Pillars

```
                    ┌─────────────────────────────┐
                    │   WELL-ARCHITECTED REVIEW   │
                    └─────────────────────────────┘
                                  │
    ┌─────────┬─────────┬────────┼────────┬─────────┬─────────┐
    ▼         ▼         ▼        ▼        ▼         ▼         ▼
┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐
│OPERATION│SECURITY│RELIABIL │PERFORM │  COST  │SUSTAIN │
│  EXCEL  │        │  ITY    │ EFFIC  │ OPTIM  │ABILITY │
├────────┤├────────┤├────────┤├────────┤├────────┤├────────┤
│Automate│IAM     │Multi-AZ │Right   │Reserved│Efficient│
│IaC     │Encrypt │Auto heal│size    │Spot    │resources│
│Monitor │Network │Backup   │Cache   │Delete  │Region   │
│Improve │Detect  │Test DR  │CDN     │unused  │selection│
└────────┘└────────┘└────────┘└────────┘└────────┘└────────┘

Questions to ask for each pillar:
├── Operations: How do we know when something breaks?
├── Security: Who can access what, and is data protected?
├── Reliability: What happens when things fail?
├── Performance: Are we using the right resources?
├── Cost: Are we spending money efficiently?
└── Sustainability: Are we minimizing environmental impact?
```

---

## 16. Service Selection Decision Trees

### Storage Decision
```
                    ┌─────────────────┐
                    │  What's the     │
                    │  access pattern?│
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   Object/File?         Block Storage?       File System?
        │                    │                    │
        ▼                    │                    ▼
┌───────────────┐            │            ┌───────────────┐
│ Web assets?   │            │            │ Shared across │
│ Backups?      │            │            │ instances?    │
│ Data lake?    │            │            └───────┬───────┘
└───────┬───────┘            │               Yes  │  No
        ▼                    │                    │
       S3                    │            ┌──────┴──────┐
  (or S3 Glacier             │            ▼             ▼
   for archive)              │           EFS      Instance
                             │         (Linux)     Store
                             │          FSx        (temp)
                             │       (Windows/
                             ▼        Lustre)
                    ┌───────────────┐
                    │  Database or  │
                    │  boot volume? │
                    └───────┬───────┘
                            │
                            ▼
                           EBS
                    (gp3 for general,
                     io2 for IOPS)
```

### Load Balancer Decision
```
                    ┌────────────────────┐
                    │ What type of       │
                    │ traffic?           │
                    └─────────┬──────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   HTTP/HTTPS?          TCP/UDP/TLS?         Third-party
        │                     │               appliances?
        ▼                     ▼                     │
       ALB                   NLB                    ▼
  (Layer 7)             (Layer 4)                 GWLB
  • Path routing        • Ultra low latency       (inline
  • Host routing        • Static IP               inspection)
  • Cognito auth        • Millions of req/s
  • WebSocket           • PrivateLink
```

---

## 17. The Request Journey - Full Stack Trace

```
User types: www.example.com
                │
                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. DNS Resolution (Route 53)                                     │
│    Browser → Route 53 → Returns CloudFront IP                    │
└─────────────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. CDN Edge (CloudFront)                                         │
│    Cache hit? → Return cached content                            │
│    Cache miss? → Forward to origin                               │
└─────────────────────────────────────────────────────────────────┘
                │ (if cache miss)
                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Web Application Firewall (WAF)                                │
│    Check rules → Block bad requests, allow good ones             │
└─────────────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Load Balancer (ALB)                                           │
│    SSL termination → Route to target group → Health check        │
└─────────────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Compute (EC2/ECS/Lambda)                                      │
│    Process request → Call other services → Generate response     │
└─────────────────────────────────────────────────────────────────┘
                │
        ┌───────┴───────┬───────────────┐
        ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ 6. Database │ │ 7. Cache    │ │ 8. Storage  │
│ (RDS/Dynamo)│ │(ElastiCache)│ │    (S3)     │
└─────────────┘ └─────────────┘ └─────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. Response travels back through the same path                   │
│    Compute → ALB → CloudFront (cache) → User                     │
└─────────────────────────────────────────────────────────────────┘

Observability at each step:
├── Route 53: Query logs
├── CloudFront: Access logs, real-time logs
├── WAF: Request logs
├── ALB: Access logs, target group metrics
├── Compute: CloudWatch metrics/logs, X-Ray traces
├── Database: Performance Insights, slow query logs
└── All: CloudTrail for API calls
```

---

## 18. The Migration Strategies - 6 R's

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE 6 R'S OF MIGRATION                        │
└─────────────────────────────────────────────────────────────────┘

    Less Effort ◄────────────────────────────────► More Effort
    Less Benefit                                   More Benefit

┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ RETIRE   │ RETAIN   │ REHOST   │REPLATFORM│REPURCHASE│REFACTOR  │
│          │          │(Lift &   │(Lift &   │(Replace) │(Re-arch) │
│          │          │ Shift)   │ Optimize)│          │          │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Turn off │ Keep     │ Move     │ Move +   │ Buy SaaS │ Rewrite  │
│ not      │ on-prem  │ as-is    │ small    │ replace  │ cloud-   │
│ needed   │ for now  │ to EC2   │ changes  │ on-prem  │ native   │
│          │          │          │ (RDS)    │ app      │          │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ $0       │ Existing │ Quick    │ Some     │ Variable │ Highest  │
│          │ costs    │ savings  │ optim    │          │ benefit  │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

Examples:
├── Retire: Old test environments nobody uses
├── Retain: Mainframe with 2-year replacement plan
├── Rehost: Web server → EC2 (same OS, same app)
├── Replatform: MySQL on-prem → RDS MySQL (managed)
├── Repurchase: Exchange Server → Microsoft 365
└── Refactor: Monolith → Microservices on EKS
```

---

## 19. Troubleshooting Decision Tree

```
                    ┌─────────────────────┐
                    │   ISSUE REPORTED    │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │ Can users reach app?│
                    └──────────┬──────────┘
                          No   │   Yes
                    ┌──────────┴──────────┐
                    ▼                     ▼
           Network Issue            App Issue
                │                        │
    ┌───────────┴───────────┐           │
    ▼                       ▼           │
DNS working?          Security Groups    │
(nslookup)            allow traffic?     │
    │                       │           │
    ▼                       ▼           │
Route 53              Check inbound      │
health checks         rules for ports    │
    │                       │           │
    ▼                       ▼           │
Check ALB/NLB         NACLs allow        │
target health         both directions?   │
                                        │
                    ┌───────────────────┘
                    ▼
           ┌───────────────────┐
           │ What's the error? │
           └─────────┬─────────┘
                     │
    ┌────────────────┼────────────────┐
    ▼                ▼                ▼
  5xx?            4xx?           Timeout?
    │                │                │
    ▼                ▼                ▼
Server error    Client error     Resource
Check:          Check:           exhaustion
• App logs      • Auth/IAM       Check:
• Memory/CPU    • API params     • CPU/Mem
• Dependencies  • Permissions    • Connections
• Lambda cold                    • Throttling
  start

Tools to use:
├── CloudWatch Logs: Application errors
├── CloudWatch Metrics: Resource utilization
├── X-Ray: Request tracing, latency
├── VPC Flow Logs: Network issues
├── CloudTrail: API call failures
└── Access Advisor: Permission issues
```

---

## 20. AWS Service Naming Patterns

```
Understanding the names helps remember what they do:

┌─────────────────┬───────────────────────────────────────────────┐
│ Pattern         │ Examples                                       │
├─────────────────┼───────────────────────────────────────────────┤
│ "Elastic"       │ Elastic = Auto-scaling/flexible               │
│                 │ ELB, EBS, EFS, ElastiCache, Elastic Beanstalk │
├─────────────────┼───────────────────────────────────────────────┤
│ "Simple"        │ Simple = Easy to use, managed                 │
│                 │ S3, SQS, SNS, SES, SWF                        │
├─────────────────┼───────────────────────────────────────────────┤
│ "Cloud"         │ Cloud = AWS-specific cloud service            │
│                 │ CloudWatch, CloudTrail, CloudFront, CloudFormation │
├─────────────────┼───────────────────────────────────────────────┤
│ "Amazon"        │ Amazon = Usually a unique AWS service         │
│                 │ Aurora, Athena, Neptune, Bedrock, Q           │
├─────────────────┼───────────────────────────────────────────────┤
│ "AWS"           │ AWS = Platform/management service             │
│                 │ AWS Organizations, AWS Config, AWS Backup     │
├─────────────────┼───────────────────────────────────────────────┤
│ Nature/Space    │ Database/Analytics services                   │
│                 │ Aurora, Neptune, Redshift, Kinesis, Glue     │
├─────────────────┼───────────────────────────────────────────────┤
│ AI/ML names     │ Human-like abilities                          │
│                 │ Lex (speech), Polly (voice), Rekognition     │
└─────────────────┴───────────────────────────────────────────────┘
```

---

## 21. Key AWS Limits to Remember

```
┌─────────────────────────────────────────────────────────────────┐
│              CRITICAL LIMITS (Soft = can request increase)       │
├─────────────────────────────────────────────────────────────────┤
│ SERVICE          │ LIMIT                     │ TYPE             │
├──────────────────┼───────────────────────────┼──────────────────┤
│ S3 bucket        │ 100 per account           │ Soft             │
│ S3 object        │ 5TB max                   │ Hard             │
│ S3 PUT           │ 5GB (single), 5TB (multi) │ Hard             │
├──────────────────┼───────────────────────────┼──────────────────┤
│ EC2 instances    │ Varies by type/region     │ Soft             │
│ EBS volume       │ 64TB (gp3)                │ Hard             │
│ Security Groups  │ 5 per ENI (default)       │ Soft             │
│ Rules per SG     │ 60 inbound + 60 outbound  │ Soft             │
├──────────────────┼───────────────────────────┼──────────────────┤
│ Lambda memory    │ 128MB - 10GB              │ Hard             │
│ Lambda timeout   │ 15 minutes                │ Hard             │
│ Lambda payload   │ 6MB (sync), 256KB (async) │ Hard             │
│ Lambda /tmp      │ 10GB                      │ Hard             │
├──────────────────┼───────────────────────────┼──────────────────┤
│ DynamoDB item    │ 400KB                     │ Hard             │
│ DynamoDB RCU/WCU │ 40,000 per table          │ Soft             │
├──────────────────┼───────────────────────────┼──────────────────┤
│ SQS message      │ 256KB                     │ Hard             │
│ SQS retention    │ 14 days max               │ Hard             │
│ SNS message      │ 256KB                     │ Hard             │
├──────────────────┼───────────────────────────┼──────────────────┤
│ API Gateway      │ 29 seconds (timeout)      │ Hard             │
│ API Gateway      │ 10MB (payload)            │ Hard             │
├──────────────────┼───────────────────────────┼──────────────────┤
│ VPCs per region  │ 5 (default)               │ Soft             │
│ Subnets per VPC  │ 200                       │ Soft             │
│ IGW per VPC      │ 1                         │ Hard             │
└──────────────────┴───────────────────────────┴──────────────────┘
```

---

## 22. Learning Path Recommendation

```
                    ┌─────────────────────┐
                    │   AWS MASTERY PATH  │
                    └──────────┬──────────┘
                               │
    ┌──────────────────────────┼──────────────────────────┐
    │                          │                          │
    ▼                          ▼                          ▼

PHASE 1: FOUNDATION        PHASE 2: BUILD           PHASE 3: OPTIMIZE
(Weeks 1-4)                (Weeks 5-10)             (Weeks 11-16)

├── IAM (critical!)        ├── Lambda               ├── Cost optimization
├── VPC networking         ├── API Gateway          ├── Well-Architected
├── EC2 basics             ├── DynamoDB             ├── Security deep dive
├── S3 fundamentals        ├── SQS/SNS/EventBridge  ├── DR strategies
├── RDS basics             ├── ECS/EKS basics       ├── Multi-account
└── CloudWatch basics      ├── CloudFormation/CDK   └── Observability stack
                           └── CI/CD pipeline

    │                          │                          │
    ▼                          ▼                          ▼

BUILD:                     BUILD:                     BUILD:
• 3-tier web app          • Serverless API           • Cost dashboard
• Multi-AZ setup          • Event-driven app         • Security audit
• Basic monitoring        • Container deployment     • DR runbook

                    ┌─────────────────────┐
                    │ PHASE 4: SPECIALIZE │
                    │   (Ongoing)         │
                    └──────────┬──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
   ML/AI Path            DevOps Path           Architect Path
   ├── SageMaker         ├── EKS deep dive     ├── Multi-region
   ├── Bedrock           ├── GitOps            ├── Landing zones
   └── AI Services       └── Platform eng      └── Hybrid cloud
```

---

## Quick Reference: The 80/20 of AWS

```
20% of services that solve 80% of use cases:

COMPUTE:      EC2, Lambda, ECS/Fargate
STORAGE:      S3, EBS, EFS
DATABASE:     RDS, DynamoDB, ElastiCache
NETWORKING:   VPC, ALB/NLB, Route 53, CloudFront
SECURITY:     IAM, KMS, WAF, Security Hub
MONITORING:   CloudWatch, X-Ray, CloudTrail
INTEGRATION:  SQS, SNS, EventBridge, Step Functions
DEVOPS:       CodePipeline, CloudFormation/CDK

Master these first. Everything else builds on top.
```
