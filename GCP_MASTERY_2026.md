# GCP Mastery Concepts 2026

## GCP Fundamentals
- Global Infrastructure (Regions, Zones, Edge Locations)
- Projects & Project Hierarchy
- Resource Manager
- Organizations
- Folders
- Labels & Tags
- Resource Quotas
- Google Cloud Console
- Cloud Shell
- gcloud CLI
- APIs & Services
- Billing Accounts
- Budgets & Alerts

---

## Identity & Access Management (IAM)

### Core Concepts
- Members (Google Account, Service Account, Group, Domain)
- Roles (Basic, Predefined, Custom)
- Permissions
- IAM Policies
- Policy Binding
- Policy Inheritance
- Resource Hierarchy & IAM
- Deny Policies

### Service Accounts
- User-Managed Service Accounts
- Default Service Accounts
- Service Account Keys
- Workload Identity
- Service Account Impersonation
- Short-Lived Credentials
- Service Account Best Practices

### Advanced IAM
- IAM Conditions
- IAM Recommender
- Policy Analyzer
- Policy Troubleshooter
- Organization Policies
- Organization Policy Constraints
- Access Context Manager
- VPC Service Controls
- BeyondCorp Enterprise

### Identity Services
- Cloud Identity
- Identity Platform
- Workforce Identity Federation
- Workload Identity Federation
- Identity-Aware Proxy (IAP)
- Managed Microsoft AD

---

## Compute Services

### Compute Engine
- Machine Types (General, Compute, Memory, Accelerator)
- Custom Machine Types
- Preemptible VMs
- Spot VMs
- Sole-Tenant Nodes
- Instance Templates
- Instance Groups (Managed, Unmanaged)
- Autoscaling
- Machine Images
- Snapshots
- Persistent Disks
- Local SSDs
- GPUs & TPUs
- Shielded VMs
- Confidential VMs
- OS Login
- Metadata Server
- Startup & Shutdown Scripts
- Live Migration
- Instance Scheduling

### Managed Instance Groups (MIG)
- Stateless vs Stateful MIGs
- Autoscaling Policies
- Health Checks
- Rolling Updates
- Canary Updates
- Regional vs Zonal MIGs
- Instance Redistribution

### Cloud Functions
- 1st Gen vs 2nd Gen
- Event-Driven Functions
- HTTP Functions
- CloudEvents
- Triggers (Pub/Sub, Cloud Storage, Firestore, etc.)
- Concurrency Settings
- Min/Max Instances
- VPC Connector
- Secrets Integration
- Cloud Functions Framework

### Cloud Run
- Container Deployment
- Services vs Jobs
- Autoscaling (0 to N)
- Concurrency
- CPU Allocation
- Revision Management
- Traffic Splitting
- Custom Domains
- VPC Access
- Cloud Run for Anthos
- Always-On CPU
- Direct VPC Egress
- Session Affinity
- Startup & Liveness Probes

### App Engine
- Standard Environment
- Flexible Environment
- Services & Versions
- Traffic Splitting
- Scaling Configuration
- App Engine Cron
- Task Queues
- Memcache

---

## Container & Kubernetes Services

### Google Kubernetes Engine (GKE)
- Cluster Architecture
- Control Plane
- Node Pools
- Autopilot Mode
- Standard Mode
- GKE Enterprise
- Release Channels (Rapid, Regular, Stable)
- Node Auto-Provisioning
- Cluster Autoscaler
- Vertical Pod Autoscaler
- Node Auto-Upgrade
- Node Auto-Repair
- Private Clusters
- Regional vs Zonal Clusters
- Multi-Zonal Clusters

### GKE Networking
- VPC-Native Clusters
- Alias IP Ranges
- Network Policies
- Dataplane V2 (Cilium)
- Gateway API
- GKE Ingress
- Container-Native Load Balancing
- Multi-Cluster Ingress
- Multi-Cluster Services
- Anthos Service Mesh

### GKE Security
- Workload Identity
- Binary Authorization
- GKE Sandbox (gVisor)
- Shielded GKE Nodes
- Pod Security Standards
- Config Connector
- Policy Controller
- Security Posture Dashboard

### GKE Operations
- GKE Monitoring
- GKE Logging
- Cost Optimization
- Backup for GKE
- Config Management
- Fleet Management

### Artifact Registry
- Container Images
- Language Packages (Maven, npm, Python)
- OS Packages
- Helm Charts
- Vulnerability Scanning
- Remote Repositories
- Virtual Repositories
- Cleanup Policies

---

## Storage Services

### Cloud Storage
- Buckets & Objects
- Storage Classes (Standard, Nearline, Coldline, Archive)
- Autoclass
- Lifecycle Management
- Object Versioning
- Retention Policies
- Object Holds
- Bucket Lock
- Signed URLs
- Signed Policy Documents
- IAM vs ACLs
- Uniform Bucket-Level Access
- Public Access Prevention
- Requester Pays
- Cross-Region Replication (Dual-Region, Multi-Region)
- Turbo Replication
- Object Composition
- Parallel Composite Uploads
- gsutil & gcloud storage
- Transfer Service
- Storage Transfer Service

### Persistent Disk
- Zonal Persistent Disks
- Regional Persistent Disks
- Disk Types (pd-standard, pd-balanced, pd-ssd, pd-extreme)
- Hyperdisk (Extreme, Throughput, Balanced)
- Snapshots
- Snapshot Schedules
- Images
- Machine Images
- Disk Cloning
- Disk Resize

### Filestore
- Basic Tier
- High Scale Tier
- Enterprise Tier
- Zonal & Regional
- Backups
- Snapshots
- NFS Shares

### Other Storage
- Cloud Storage for Firebase
- Persistent Disk CSI Driver
- NetApp Cloud Volumes

---

## Database Services

### Cloud SQL
- MySQL, PostgreSQL, SQL Server
- High Availability (Regional)
- Read Replicas
- Cross-Region Replicas
- Point-in-Time Recovery
- Automated Backups
- On-Demand Backups
- Maintenance Windows
- Private IP & Public IP
- Cloud SQL Auth Proxy
- IAM Database Authentication
- Data Insights
- Cloud SQL Enterprise Plus
- Cloud SQL Editions

### Cloud Spanner
- Global Distribution
- Horizontal Scaling
- Multi-Region Configurations
- Regional Configurations
- Splits & Interleaving
- Secondary Indexes
- Read-Only Replicas
- Autoscaling
- Point-in-Time Recovery
- Backup & Restore
- Change Streams
- Spanner Graph
- PostgreSQL Interface

### Firestore
- Native Mode
- Datastore Mode
- Documents & Collections
- Queries & Indexes
- Composite Indexes
- Real-Time Listeners
- Offline Persistence
- Security Rules
- Multi-Region Replication
- Point-in-Time Recovery
- TTL Policies
- Firestore Aggregation Queries

### Bigtable
- Clusters & Nodes
- Tables, Column Families, Rows
- Row Key Design
- Replication
- Autoscaling
- Garbage Collection
- Data Boost
- Change Streams
- Bigtable Beam Connector

### AlloyDB
- PostgreSQL Compatible
- Columnar Engine
- AI/ML Integration
- High Availability
- Cross-Region Replication
- AlloyDB AI
- AlloyDB Omni

### Memorystore
- Redis
- Memcached
- Cluster Mode
- Read Replicas
- RDB Snapshots
- High Availability
- Memorystore for Valkey

### Other Databases
- Firebase Realtime Database
- Database Migration Service
- Datastream (CDC)
- Bare Metal Solution

---

## Networking

### Virtual Private Cloud (VPC)
- VPC Networks
- Subnets (Auto-Mode, Custom-Mode)
- IP Addressing
- Alias IP Ranges
- Private Google Access
- Private Service Access
- Firewall Rules
- Firewall Policies (Hierarchical, Network)
- Firewall Insights
- Routes & Routing
- VPC Network Peering
- Shared VPC
- VPC Flow Logs
- Packet Mirroring
- Private Service Connect
- Network Connectivity Center

### Hybrid Connectivity
- Cloud VPN (HA VPN, Classic VPN)
- Cloud Interconnect (Dedicated, Partner)
- Cross-Cloud Interconnect
- Network Connectivity Center
- Cloud Router
- BGP Sessions

### Load Balancing
- Global External Application Load Balancer
- Regional External Application Load Balancer
- Global External Proxy Network Load Balancer
- Regional External Proxy Network Load Balancer
- Regional External Passthrough Network Load Balancer
- Internal Application Load Balancer
- Internal Passthrough Network Load Balancer
- Cross-Region Internal Application Load Balancer
- Backend Services
- Backend Buckets
- URL Maps
- Health Checks
- SSL Certificates
- SSL Policies
- Cloud Armor Integration

### Cloud DNS
- Public Zones
- Private Zones
- Forwarding Zones
- Peering Zones
- Record Types
- DNS Policies
- DNSSEC
- Response Policies
- Cloud Domains

### Cloud CDN
- Cache Modes
- Cache Keys
- Cache Invalidation
- Signed URLs & Cookies
- Media CDN
- Edge Cache

### Cloud NAT
- NAT Gateway
- Port Allocation
- Endpoint-Independent Mapping
- Dynamic Port Allocation
- Logging

### Network Security
- Cloud Armor
- Security Policies
- WAF Rules
- Adaptive Protection
- Bot Management
- Rate Limiting
- Threat Intelligence
- Cloud IDS
- Secure Web Proxy

### Network Intelligence
- Network Topology
- Performance Dashboard
- Connectivity Tests
- Firewall Insights

### Service Networking
- Service Directory
- Traffic Director
- Cloud Service Mesh

---

## Security Services

### Cloud KMS
- Key Rings
- Crypto Keys
- Key Versions
- Key Rotation
- Symmetric & Asymmetric Keys
- Hardware Security Modules (Cloud HSM)
- External Key Manager (EKM)
- Autokey
- CMEK & CSEK

### Secret Manager
- Secrets & Versions
- Secret Rotation
- Regional Secrets
- Replication Policies
- IAM Integration
- Secret Manager SDK

### Certificate Manager
- DNS Authorization
- Load Balancer Authorization
- Certificate Maps
- Certificate Issuance

### Certificate Authority Service
- Private CA
- Certificate Templates
- CA Pools

### Security Command Center
- Security Health Analytics
- Event Threat Detection
- Container Threat Detection
- Web Security Scanner
- Virtual Machine Threat Detection
- Findings & Assets
- Security Posture
- Attack Path Simulation
- Toxic Combinations

### Other Security Services
- Binary Authorization
- BeyondCorp Enterprise
- Assured Workloads
- Access Transparency
- Confidential Computing
- Shielded VMs
- reCAPTCHA Enterprise
- Web Risk API
- Phishing Protection

---

## Observability (Cloud Operations Suite)

### Cloud Monitoring
- Metrics Explorer
- Metrics Scopes
- Dashboards
- Charts & Widgets
- Custom Metrics
- Alerting Policies
- Notification Channels
- Uptime Checks
- Synthetic Monitoring
- Service Monitoring (SLOs)
- Managed Service for Prometheus
- PromQL Support

### Cloud Logging
- Log Router
- Log Buckets
- Log Sinks
- Log Views
- Log-Based Metrics
- Log Analytics
- Logs Explorer
- Error Reporting
- Audit Logs (Admin, Data Access, System)

### Cloud Trace
- Distributed Tracing
- Trace Analysis
- Latency Reports
- Trace Sampling

### Cloud Profiler
- CPU Profiling
- Heap Profiling
- Continuous Profiling

### OpenTelemetry Integration
- Ops Agent
- OpenTelemetry Collector
- Auto-Instrumentation

### Application Performance Management
- Error Reporting
- Debugger (Deprecated)
- Application Integration Monitoring

---

## DevOps & CI/CD

### Cloud Build
- Build Triggers
- Build Configurations (cloudbuild.yaml)
- Build Steps
- Substitution Variables
- Private Pools
- Worker Pools
- Build Caching
- Artifact Management
- GitHub & GitLab Integration

### Cloud Deploy
- Delivery Pipelines
- Targets
- Releases
- Rollouts
- Deployment Strategies (Canary, Blue-Green)
- Approvals
- Rollback
- Deploy Policies

### Source Repositories
- Git Repositories
- Mirroring (GitHub, Bitbucket)

### Infrastructure as Code
- Terraform on GCP
- Deployment Manager (Legacy)
- Config Connector
- Pulumi
- Crossplane

### Artifact Management
- Artifact Registry
- Container Registry (Deprecated)
- Artifact Analysis

---

## Data Analytics

### BigQuery
- Datasets & Tables
- Partitioned Tables (Time, Range, Integer)
- Clustered Tables
- External Tables
- Materialized Views
- Authorized Views
- Standard SQL
- Legacy SQL (Deprecated)
- BigQuery ML
- BigQuery BI Engine
- BigQuery Editions (Standard, Enterprise, Enterprise Plus)
- Slots & Reservations
- On-Demand vs Capacity Pricing
- Streaming Inserts
- Storage Write API
- BigQuery Data Transfer Service
- BigQuery Omni (Multi-Cloud)
- BigQuery Studio
- Duets AI in BigQuery
- Remote Functions
- Search Indexes
- Vector Search
- BigQuery DataFrames

### Dataflow
- Apache Beam
- Batch Processing
- Stream Processing
- Dataflow Templates
- Flex Templates
- Prime (Vertical Autoscaling)
- Streaming Engine
- Shuffle Service
- Dataflow SQL

### Pub/Sub
- Topics & Subscriptions
- Push & Pull Subscriptions
- Message Ordering
- Exactly-Once Delivery
- Dead Letter Topics
- Message Filtering
- BigQuery Subscriptions
- Cloud Storage Subscriptions
- Pub/Sub Lite
- Pub/Sub Schemas
- Message Retention
- Seek & Replay

### Dataproc
- Clusters
- Cluster Modes
- Autoscaling
- Dataproc Serverless
- Dataproc Metastore
- Spark, Hadoop, Flink, Hive
- Dataproc on GKE
- Personal Cluster Authentication

### Dataprep
- Data Wrangling
- Recipes
- Data Transformation
- Trifacta Integration

### Data Fusion
- Visual ETL
- Pipelines
- Data Integration
- CDAP Framework
- Replication

### Dataplex
- Data Lakes
- Data Zones
- Assets
- Data Discovery
- Data Quality
- Data Lineage
- Data Catalog
- Data Profiling
- Dataplex Universal Catalog

### Composer
- Apache Airflow
- Managed Airflow
- DAGs
- Environments
- Composer 2 & 3

### Looker
- LookML
- Explores
- Dashboards
- Looker Studio (Data Studio)
- Looker Modeler

### Analytics Hub
- Data Exchanges
- Listings
- Data Sharing

### Datastream
- Change Data Capture
- MySQL, PostgreSQL, Oracle Sources
- BigQuery, Cloud Storage Destinations

---

## Machine Learning & AI

### Vertex AI
- Vertex AI Workbench
- Custom Training
- AutoML
- Model Registry
- Model Deployment (Endpoints)
- Online & Batch Prediction
- Feature Store
- Pipelines (Kubeflow, TFX)
- Experiments & Metadata
- Vertex AI Vector Search
- Model Monitoring
- Model Evaluation
- Explainable AI
- Vertex AI Vision
- Vertex AI Search
- Vertex AI Conversation

### Generative AI
- Vertex AI Studio
- Gemini Models
- PaLM Models (Legacy)
- Model Garden
- Generative AI App Builder
- Grounding
- Extensions
- Prompt Design
- Fine-Tuning
- RLHF
- Context Caching
- Function Calling
- Embeddings API
- Imagen (Image Generation)
- Codey (Code Generation)

### AI Building Blocks
- Vision AI
- Video AI
- Natural Language API
- Translation API
- Speech-to-Text
- Text-to-Speech
- Document AI
- Contact Center AI (CCAI)
- Recommendations AI
- Retail API
- Healthcare API
- Timeseries Insights API

### ML Infrastructure
- TPUs
- GPUs
- Deep Learning VMs
- Deep Learning Containers
- TensorFlow Enterprise
- JAX on Cloud

---

## Application Integration

### Cloud Tasks
- Task Queues
- HTTP Targets
- App Engine Targets
- Task Scheduling
- Rate Limiting
- Retry Configuration

### Cloud Scheduler
- Cron Jobs
- HTTP Targets
- Pub/Sub Targets
- App Engine Targets

### Workflows
- YAML/JSON Workflow Definition
- Steps & Connectors
- Conditional Logic
- Parallel Execution
- Error Handling
- Callbacks
- Connectors

### Eventarc
- Event Routing
- Triggers
- CloudEvents
- Event Providers (GCP, Custom)

### API Gateway
- OpenAPI Specs
- API Configs
- API Keys
- Service Control
- Developer Portal

### Apigee
- API Proxies
- API Products
- Developer Portal
- Analytics
- Monetization
- Policies
- Environments
- Hybrid Deployment
- Apigee X

### Integration Connectors
- Pre-Built Connectors
- Application Integration

---

## Cost Management

### Billing
- Billing Accounts
- Billing Export to BigQuery
- Cost Table & Pricing
- Cloud Billing Reports
- Cost Breakdown
- Cost Trends

### Budgets & Alerts
- Budget Creation
- Budget Alerts
- Programmatic Notifications

### Cost Optimization
- Committed Use Discounts (CUDs)
- Sustained Use Discounts
- Spot VMs
- Preemptible VMs
- Rightsizing Recommendations
- Active Assist
- Idle Resource Detection
- Cost Attribution (Labels)
- FinOps Hub

### Pricing
- Pricing Calculator
- SKUs
- Pricing Tiers

---

## Governance & Compliance

### Resource Manager
- Projects
- Folders
- Organization
- IAM at Each Level
- Resource Hierarchy

### Organization Policies
- Constraints
- Custom Constraints
- Policy Inheritance
- Dry Run Mode

### Policy Intelligence
- IAM Recommender
- Policy Analyzer
- Policy Simulator
- Policy Troubleshooter

### Compliance
- Assured Workloads
- Compliance Reports Manager
- Access Transparency
- VPC Service Controls
- Data Residency

### Asset Inventory
- Cloud Asset Inventory
- Asset Search
- Asset Export
- Real-Time Feed

---

## Migration & Transfer

### Migration Services
- Migrate to Virtual Machines
- Migrate to Containers
- Database Migration Service
- BigQuery Data Transfer Service
- Transfer Appliance

### Data Transfer
- Storage Transfer Service
- Transfer Service for On-Premises
- gsutil & gcloud storage
- Offline Transfer (Transfer Appliance)

### Modernization
- Anthos
- Cloud Run
- GKE Autopilot

---

## Hybrid & Multi-Cloud

### Anthos
- Anthos Clusters (GKE, On-Prem, AWS, Azure)
- Anthos Config Management
- Anthos Service Mesh
- Anthos Identity Service
- Fleet Management
- Connect Gateway
- Policy Controller

### Distributed Cloud
- Google Distributed Cloud Edge
- Google Distributed Cloud Hosted
- Google Distributed Cloud Virtual

### Multi-Cloud
- BigQuery Omni
- Anthos Multi-Cloud
- Cross-Cloud Network

---

## Architecture Patterns

### High Availability
- Multi-Zonal Deployments
- Regional Deployments
- Global Load Balancing
- Health Checks
- Failover Strategies

### Scalability
- Horizontal vs Vertical Scaling
- Autoscaling
- Stateless Architecture
- Caching with Memorystore

### Security Patterns
- Defense in Depth
- Least Privilege
- VPC Service Controls
- Private Google Access
- Zero Trust (BeyondCorp)

### Resilience
- Disaster Recovery
- Backup Strategies
- Multi-Region Deployments
- Chaos Engineering

### Microservices
- Cloud Run Services
- GKE Workloads
- Service Mesh
- Event-Driven with Pub/Sub

### Data Architecture
- Data Lake (Dataplex)
- Data Warehouse (BigQuery)
- Data Mesh
- Lakehouse Architecture

---

## Important Services Summary 2026

### Must-Know Core Services
- Compute Engine, Cloud Run, Cloud Functions, GKE
- Cloud Storage, Persistent Disk, Filestore
- VPC, Cloud Load Balancing, Cloud DNS, Cloud CDN
- Cloud SQL, Cloud Spanner, Firestore, Bigtable, BigQuery
- IAM, Cloud KMS, Secret Manager
- Cloud Monitoring, Cloud Logging, Cloud Trace
- Pub/Sub, Cloud Tasks, Eventarc
- Cloud Build, Cloud Deploy, Artifact Registry

### High-Demand 2026 Services
- Vertex AI (ML Platform)
- Gemini Models (Generative AI)
- BigQuery ML & Vector Search
- GKE Autopilot
- Cloud Run (Serverless Containers)
- AlloyDB (PostgreSQL)
- Dataplex (Data Governance)
- Eventarc (Event-Driven)
- Cloud Deploy (CD)
- Security Command Center
- Managed Prometheus

### Emerging & Strategic Services
- Vertex AI Search & Conversation
- BigQuery Studio
- Gemini in Google Cloud Console
- Document AI
- Duet AI for Developers
- Application Integration
- Dataplex Universal Catalog
- Memorystore for Valkey
- Confidential Computing
- Cross-Cloud Network
- FinOps Hub
