# AWS Mastery Concepts 2026

## AWS Fundamentals
- Global Infrastructure (Regions, Availability Zones, Edge Locations)
- Local Zones & Wavelength Zones
- AWS Account Structure
- AWS Organizations
- Service Control Policies (SCPs)
- AWS Control Tower
- Landing Zones
- AWS Well-Architected Framework
- Shared Responsibility Model
- AWS Support Plans
- AWS Pricing Models
- Free Tier

---

## Identity & Access Management (IAM)

### Core Concepts
- IAM Users
- IAM Groups
- IAM Roles
- IAM Policies (Identity-based, Resource-based)
- Policy Evaluation Logic
- Principal & Actions
- Conditions in Policies
- Permission Boundaries
- Session Policies
- Service-Linked Roles
- Cross-Account Access
- AssumeRole

### Advanced IAM
- IAM Identity Center (SSO)
- SAML 2.0 Federation
- OIDC Federation
- Web Identity Federation
- Attribute-Based Access Control (ABAC)
- IAM Access Analyzer
- IAM Policy Simulator
- Credential Report
- Access Advisor
- IAM Roles Anywhere

---

## Compute Services

### EC2 (Elastic Compute Cloud)
- Instance Types & Families
- Instance Lifecycle
- AMIs (Amazon Machine Images)
- Launch Templates
- Spot Instances
- Reserved Instances
- Savings Plans
- On-Demand Capacity Reservations
- Placement Groups (Cluster, Spread, Partition)
- Elastic Network Interfaces (ENI)
- Elastic IP Addresses
- Instance Store vs EBS
- EC2 Instance Connect
- EC2 Serial Console
- Nitro System
- Graviton Processors
- EC2 Auto Recovery

### Auto Scaling
- Auto Scaling Groups
- Launch Configurations vs Launch Templates
- Scaling Policies (Target Tracking, Step, Simple)
- Predictive Scaling
- Scheduled Scaling
- Lifecycle Hooks
- Warm Pools
- Instance Refresh
- Mixed Instances Policy

### Elastic Load Balancing
- Application Load Balancer (ALB)
- Network Load Balancer (NLB)
- Gateway Load Balancer (GWLB)
- Classic Load Balancer (Legacy)
- Target Groups
- Health Checks
- Sticky Sessions
- Cross-Zone Load Balancing
- SSL/TLS Termination
- SNI (Server Name Indication)
- Connection Draining
- Access Logs

### Lambda (Serverless Compute)
- Function Configuration
- Execution Roles
- Memory & Timeout
- Invocation Types (Sync, Async, Event Source)
- Event Source Mappings
- Lambda Layers
- Lambda Extensions
- Provisioned Concurrency
- Reserved Concurrency
- Lambda@Edge
- Lambda SnapStart
- Container Image Support
- Lambda URLs
- Response Streaming
- Powertools for Lambda

---

## Container Services

### ECS (Elastic Container Service)
- Task Definitions
- Tasks & Services
- Launch Types (EC2, Fargate)
- Capacity Providers
- Service Auto Scaling
- Service Discovery
- Task Placement Strategies
- Task Networking (awsvpc, bridge, host)
- ECS Exec
- ECS Anywhere

### EKS (Elastic Kubernetes Service)
- EKS Cluster Architecture
- Managed Node Groups
- Fargate Profiles
- EKS Add-ons
- EKS Blueprints
- IRSA (IAM Roles for Service Accounts)
- Pod Identity
- EKS Anywhere
- EKS Connector
- Karpenter
- AWS Load Balancer Controller
- Cluster Autoscaler
- EKS Cost Optimization

### ECR (Elastic Container Registry)
- Repository Policies
- Image Scanning
- Lifecycle Policies
- Cross-Region Replication
- Pull-Through Cache
- OCI Artifact Support

### App Runner
- Automatic Scaling
- Source Connections
- Service Configuration
- VPC Connector

---

## Storage Services

### S3 (Simple Storage Service)
- Buckets & Objects
- Storage Classes (Standard, IA, One Zone-IA, Glacier, Deep Archive, Intelligent-Tiering)
- S3 Lifecycle Policies
- S3 Versioning
- S3 Replication (CRR, SRR)
- S3 Object Lock
- S3 Access Points
- S3 Multi-Region Access Points
- S3 Object Lambda
- S3 Event Notifications
- S3 Inventory
- S3 Batch Operations
- S3 Select & Glacier Select
- S3 Transfer Acceleration
- Presigned URLs
- Bucket Policies
- ACLs (Legacy)
- Block Public Access
- S3 Storage Lens
- S3 Express One Zone

### EBS (Elastic Block Store)
- Volume Types (gp3, gp2, io2, io1, st1, sc1)
- IOPS & Throughput
- EBS Snapshots
- EBS Fast Snapshot Restore
- EBS Multi-Attach (io1/io2)
- EBS Encryption
- EBS-Optimized Instances

### EFS (Elastic File System)
- Performance Modes (General Purpose, Max I/O)
- Throughput Modes (Bursting, Provisioned, Elastic)
- Storage Classes (Standard, IA, Archive)
- Lifecycle Management
- EFS Access Points
- EFS Replication
- Mount Targets

### FSx
- FSx for Windows File Server
- FSx for Lustre
- FSx for NetApp ONTAP
- FSx for OpenZFS

### Storage Gateway
- File Gateway
- Volume Gateway
- Tape Gateway

---

## Database Services

### RDS (Relational Database Service)
- Supported Engines (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server)
- Multi-AZ Deployments
- Read Replicas
- RDS Proxy
- Parameter Groups
- Option Groups
- Automated Backups
- Manual Snapshots
- Point-in-Time Recovery
- Performance Insights
- Enhanced Monitoring
- IAM Database Authentication
- RDS Custom

### Aurora
- Aurora MySQL & PostgreSQL
- Aurora Serverless v2
- Aurora Global Database
- Aurora Replicas
- Aurora Multi-Master
- Aurora Cloning
- Aurora Backtrack
- Aurora Machine Learning
- Aurora I/O-Optimized
- Aurora Limitless Database

### DynamoDB
- Tables, Items, Attributes
- Primary Keys (Partition Key, Sort Key)
- Secondary Indexes (LSI, GSI)
- Read/Write Capacity Modes (Provisioned, On-Demand)
- DynamoDB Streams
- Global Tables
- TTL (Time to Live)
- Transactions
- PartiQL
- DAX (DynamoDB Accelerator)
- Point-in-Time Recovery
- On-Demand Backup
- DynamoDB Zero-ETL to OpenSearch

### ElastiCache
- Redis (Cluster Mode, Replication)
- Memcached
- ElastiCache Serverless
- Global Datastore
- Data Tiering

### Other Database Services
- DocumentDB (MongoDB Compatible)
- Neptune (Graph Database)
- Timestream (Time Series)
- QLDB (Ledger Database)
- Keyspaces (Cassandra Compatible)
- MemoryDB for Redis

---

## Networking

### VPC (Virtual Private Cloud)
- CIDR Blocks
- Subnets (Public, Private)
- Route Tables
- Internet Gateway
- NAT Gateway
- NAT Instance
- Elastic IPs
- VPC Peering
- VPC Endpoints (Gateway, Interface)
- PrivateLink
- VPC Flow Logs
- Network ACLs
- Security Groups
- DHCP Options Sets
- DNS Resolution & Hostnames
- VPC Sharing
- IPv6 Support

### Transit Gateway
- Transit Gateway Attachments
- Route Tables
- Peering Attachments
- Multicast
- Connect Attachments
- Inter-Region Peering

### Direct Connect
- Dedicated Connections
- Hosted Connections
- Virtual Interfaces (Private, Public, Transit)
- Direct Connect Gateway
- Link Aggregation Groups (LAG)
- Site-to-Site VPN Backup

### Site-to-Site VPN
- Virtual Private Gateway
- Customer Gateway
- VPN Connections
- Accelerated VPN
- VPN CloudHub

### Route 53
- Hosted Zones (Public, Private)
- Record Types (A, AAAA, CNAME, MX, TXT, NS, SOA, Alias)
- Routing Policies (Simple, Weighted, Latency, Failover, Geolocation, Geoproximity, Multi-Value, IP-Based)
- Health Checks
- DNS Failover
- Route 53 Resolver
- Resolver Endpoints
- DNSSEC
- Route 53 Application Recovery Controller

### CloudFront
- Distributions
- Origins (S3, ALB, Custom)
- Behaviors
- Cache Policies
- Origin Request Policies
- Response Headers Policies
- Edge Functions (CloudFront Functions, Lambda@Edge)
- Origin Shield
- Origin Access Control (OAC)
- Field-Level Encryption
- Signed URLs & Cookies
- WebSocket Support
- Real-Time Logs

### Global Accelerator
- Accelerators
- Listeners
- Endpoint Groups
- Endpoints
- Client Affinity
- Custom Routing

### API Gateway
- REST APIs
- HTTP APIs
- WebSocket APIs
- API Keys & Usage Plans
- Stages & Deployments
- Request/Response Transformations
- Lambda Integration (Proxy, Non-Proxy)
- Authorizers (Lambda, Cognito, IAM)
- Throttling & Quotas
- Caching
- Custom Domain Names
- Private APIs
- Canary Deployments

---

## Security Services

### KMS (Key Management Service)
- Customer Master Keys (CMK)
- AWS Managed Keys
- Customer Managed Keys
- Key Policies
- Grants
- Envelope Encryption
- Key Rotation
- Multi-Region Keys
- External Key Store

### Secrets Manager
- Secret Rotation
- Cross-Region Replication
- Resource Policies
- Automatic Rotation (Lambda)

### Certificate Manager (ACM)
- Public Certificates
- Private CA
- Certificate Renewal
- DNS Validation
- Email Validation

### WAF (Web Application Firewall)
- Web ACLs
- Rules & Rule Groups
- Managed Rules
- Rate-Based Rules
- Bot Control
- Fraud Control
- Custom Rules

### Shield
- Shield Standard
- Shield Advanced
- DDoS Protection
- Cost Protection

### GuardDuty
- Threat Detection
- Finding Types
- Malware Protection
- S3 Protection
- EKS Protection
- Lambda Protection
- RDS Protection

### Security Hub
- Security Standards
- Findings
- Automated Response
- Cross-Account Aggregation

### Inspector
- Vulnerability Scanning
- EC2 Scanning
- ECR Scanning
- Lambda Scanning
- Network Reachability

### Other Security Services
- Macie (Data Discovery)
- Detective (Investigation)
- Firewall Manager
- Network Firewall
- Verified Access
- Resource Access Manager (RAM)

---

## Monitoring & Observability

### CloudWatch
- Metrics & Custom Metrics
- Metric Math
- Anomaly Detection
- Alarms (Metric, Composite)
- Dashboards
- Logs & Log Groups
- Log Insights
- Logs Live Tail
- Metric Filters
- Subscription Filters
- Contributor Insights
- ServiceLens
- Synthetics
- RUM (Real User Monitoring)
- Evidently (Feature Flags)
- Application Signals
- Internet Monitor

### X-Ray
- Traces & Segments
- Subsegments
- Sampling Rules
- Service Map
- Trace Map
- Insights
- Groups
- X-Ray SDK

### CloudTrail
- Management Events
- Data Events
- Insights Events
- Trail Configuration
- CloudTrail Lake
- Event History
- Organization Trail

### EventBridge
- Event Buses
- Rules & Targets
- Event Patterns
- Scheduler
- Pipes
- Schema Registry
- Archive & Replay
- Global Endpoints

### AWS Config
- Configuration Items
- Configuration Recorder
- Config Rules (Managed, Custom)
- Conformance Packs
- Remediation Actions
- Aggregators

---

## DevOps & CI/CD

### CodePipeline
- Stages & Actions
- Source Actions
- Build Actions
- Deploy Actions
- Approval Actions
- Pipeline Triggers
- Cross-Region Actions
- Pipeline Variables

### CodeBuild
- Build Projects
- Build Specifications (buildspec.yml)
- Build Environments
- Build Caching
- Batch Builds
- Reports

### CodeDeploy
- Deployment Groups
- Deployment Configurations
- AppSpec File
- Deployment Types (In-Place, Blue/Green)
- Rollback Configuration
- Lifecycle Hooks

### CodeCommit
- Repositories
- Branches
- Pull Requests
- Triggers & Notifications

### CodeArtifact
- Domains & Repositories
- Upstream Repositories
- Package Publishing

### CloudFormation
- Templates (YAML, JSON)
- Stacks & Stack Sets
- Change Sets
- Drift Detection
- Nested Stacks
- Cross-Stack References
- Custom Resources
- Macros
- Stack Policies
- Rollback Triggers
- Registry & Modules

### CDK (Cloud Development Kit)
- Constructs (L1, L2, L3)
- Stacks & Apps
- Assets
- Context
- CDK Pipelines
- cdk synth & deploy

### Terraform on AWS
- AWS Provider
- State Management (S3 Backend)
- State Locking (DynamoDB)
- Workspaces
- Modules

---

## Application Integration

### SQS (Simple Queue Service)
- Standard Queues
- FIFO Queues
- Message Attributes
- Visibility Timeout
- Dead Letter Queues
- Delay Queues
- Long Polling
- Message Deduplication
- Message Grouping
- Temporary Queues

### SNS (Simple Notification Service)
- Topics (Standard, FIFO)
- Subscriptions
- Message Filtering
- Message Fanout
- SMS & Mobile Push
- Email Notifications

### Step Functions
- State Machines
- States (Task, Pass, Choice, Wait, Parallel, Map, Fail, Succeed)
- Standard vs Express Workflows
- Distributed Map
- Service Integrations
- Error Handling & Retries
- Workflow Studio

### EventBridge
- Event-Driven Architecture
- Custom Event Buses
- Partner Event Sources
- Schema Discovery

### AppSync
- GraphQL APIs
- Resolvers
- Data Sources
- Real-Time Subscriptions
- Caching
- Authentication Methods

### MQ (Message Broker)
- ActiveMQ
- RabbitMQ
- Broker Deployments

---

## Analytics & Big Data

### Athena
- SQL Queries on S3
- Data Formats (Parquet, ORC, JSON, CSV)
- Partitioning
- Workgroups
- Federated Queries
- ACID Transactions
- Apache Spark

### Redshift
- Clusters & Nodes
- Redshift Serverless
- Data Sharing
- Spectrum (Query S3)
- Materialized Views
- Workload Management
- Concurrency Scaling
- RA3 Instances
- Zero-ETL Integrations

### EMR (Elastic MapReduce)
- Cluster Modes
- EMR Serverless
- EMR on EKS
- Spark, Hive, Presto, Flink
- EMRFS
- Instance Fleets

### Kinesis
- Kinesis Data Streams
- Kinesis Data Firehose
- Kinesis Data Analytics
- Kinesis Video Streams
- Enhanced Fan-Out
- Shard Management

### Glue
- Data Catalog
- Crawlers
- ETL Jobs
- Glue Studio
- Glue DataBrew
- Glue Streaming
- Schema Registry
- Glue Data Quality

### Lake Formation
- Data Lakes
- Data Permissions
- LF-Tags
- Governed Tables
- Row & Cell-Level Security

### OpenSearch Service
- Domains
- Indexes
- OpenSearch Dashboards
- Serverless
- Ingestion

### QuickSight
- Dashboards
- SPICE
- ML Insights
- Embedded Analytics
- Q (Natural Language)

### Data Pipeline (Legacy)
### MSK (Managed Streaming for Kafka)
- Brokers
- MSK Serverless
- MSK Connect
- Tiered Storage

---

## Machine Learning & AI

### SageMaker
- Studio & Studio Lab
- Notebooks
- Training Jobs
- Hyperparameter Tuning
- Model Registry
- Endpoints (Real-Time, Serverless, Async)
- Batch Transform
- Pipelines
- Feature Store
- Data Wrangler
- Clarify
- Model Monitor
- Ground Truth
- JumpStart
- Canvas
- SageMaker HyperPod

### Bedrock
- Foundation Models (Claude, Titan, Llama, etc.)
- Model Customization
- Fine-Tuning
- Knowledge Bases
- Agents
- Guardrails
- Prompt Management
- Model Evaluation

### AI Services
- Rekognition (Image/Video Analysis)
- Textract (Document Processing)
- Comprehend (NLP)
- Translate
- Polly (Text-to-Speech)
- Transcribe (Speech-to-Text)
- Lex (Chatbots)
- Personalize (Recommendations)
- Forecast
- Kendra (Enterprise Search)
- Q Business

### Generative AI
- Amazon Q Developer
- CodeWhisperer
- PartyRock
- Bedrock Agents

---

## Cost Management

### Cost Explorer
- Cost Analysis
- Forecasting
- Savings Plans Recommendations
- Reserved Instance Recommendations

### Budgets
- Cost Budgets
- Usage Budgets
- Savings Plans Budgets
- Budget Alerts
- Budget Actions

### Cost Optimization
- Trusted Advisor
- Compute Optimizer
- S3 Storage Lens
- Right Sizing
- Spot Instances Strategy
- Savings Plans vs Reserved Instances

### Billing
- Consolidated Billing
- Cost Allocation Tags
- AWS Cost and Usage Reports
- AWS Pricing Calculator

---

## Governance & Compliance

### Organizations
- Organizational Units (OUs)
- Service Control Policies (SCPs)
- Tag Policies
- Backup Policies
- AI Services Opt-Out Policies

### Control Tower
- Landing Zone
- Guardrails (Preventive, Detective)
- Account Factory
- Account Customizations

### Service Catalog
- Portfolios
- Products
- Provisioned Products
- Launch Constraints

### License Manager
- License Tracking
- License Rules

### Systems Manager
- Session Manager
- Run Command
- Patch Manager
- Parameter Store
- State Manager
- OpsCenter
- Automation
- Inventory
- Application Manager
- Fleet Manager
- Change Manager

---

## Migration & Transfer

### Migration Services
- Migration Hub
- Application Discovery Service
- Application Migration Service
- Database Migration Service (DMS)
- Schema Conversion Tool (SCT)
- Mainframe Modernization

### Transfer Services
- DataSync
- Transfer Family (SFTP, FTPS, FTP)
- Snow Family (Snowcone, Snowball, Snowmobile)

---

## Disaster Recovery & Backup

### Backup
- Backup Plans
- Backup Vaults
- Recovery Points
- Cross-Region & Cross-Account Backup
- Backup Audit Manager

### DR Strategies
- Backup & Restore
- Pilot Light
- Warm Standby
- Multi-Site Active/Active

### Elastic Disaster Recovery
- Source Servers
- Recovery Instances
- Failback

---

## Hybrid & Edge

### Outposts
- Outpost Racks
- Outpost Servers
- Local Gateway

### Wavelength
- 5G Edge Computing
- Carrier Gateways

### Local Zones
- Low-Latency Applications

### Snow Family
- Snowcone
- Snowball Edge
- AWS OpsHub

---

## Architecture Patterns

### High Availability
- Multi-AZ Deployments
- Auto Scaling
- Load Balancing
- Health Checks
- Failover Strategies

### Scalability
- Horizontal vs Vertical Scaling
- Stateless Architecture
- Caching Strategies
- Database Scaling

### Security Patterns
- Defense in Depth
- Least Privilege
- Encryption at Rest & Transit
- Network Segmentation
- Zero Trust Architecture

### Resilience
- Chaos Engineering
- Circuit Breaker Pattern
- Bulkhead Pattern
- Retry with Exponential Backoff
- Graceful Degradation

### Microservices
- Service Discovery
- API Gateway Pattern
- Event-Driven Architecture
- Saga Pattern
- CQRS

### Serverless Patterns
- Event-Driven Processing
- Fan-Out Pattern
- Strangler Fig Pattern

---

## Important Services Summary 2026

### Must-Know Core Services
- EC2, Lambda, ECS, EKS
- S3, EBS, EFS
- VPC, Route 53, CloudFront, API Gateway
- RDS, Aurora, DynamoDB
- IAM, KMS, Secrets Manager
- CloudWatch, CloudTrail, X-Ray
- SQS, SNS, EventBridge, Step Functions
- CloudFormation, CDK

### High-Demand 2026 Services
- Bedrock (Generative AI)
- SageMaker (ML Platform)
- EKS & Karpenter (Kubernetes)
- Aurora Serverless v2
- Lambda SnapStart
- EventBridge Pipes & Scheduler
- S3 Express One Zone
- ElastiCache Serverless
- Application Signals (APM)
- Verified Access (Zero Trust)
- Amazon Q (AI Assistant)

### Emerging & Strategic Services
- Graviton Processors (Cost Optimization)
- PrivateLink & Lattice (Service Networking)
- Clean Rooms (Data Collaboration)
- Entity Resolution
- HealthLake
- IoT Core & Greengrass
- Amplify (Full-Stack Development)
- AppConfig (Feature Flags)
- Proton (Platform Engineering)
