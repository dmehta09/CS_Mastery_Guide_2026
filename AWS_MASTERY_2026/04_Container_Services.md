# AWS Container Services: Comprehensive Conceptual Guide

**Last Updated**: February 2026
**Target Audience**: Backend engineers, cloud architects, DevOps engineers, and developers

---

## Table of Contents

1. [Introduction](#introduction)
2. [ECS - Elastic Container Service](#ecs)
3. [EKS - Elastic Kubernetes Service](#eks)
4. [ECR - Elastic Container Registry](#ecr)
5. [App Runner](#app-runner)
6. [Comparison & Best Practices](#comparison)
7. [Real-World Examples](#examples)

---

<a name="introduction"></a>
## 1. Introduction to AWS Container Services

### What Are Containers?

Containers package applications with all their dependencies into a single, portable unit. Unlike VMs that virtualize hardware, containers virtualize the operating system, making them:

- **Lightweight**: Share the host OS kernel
- **Fast**: Start in milliseconds
- **Portable**: Run consistently across environments
- **Isolated**: Each container has its own filesystem and process space

### AWS Container Service Ecosystem

```
┌─────────────────────────────────────────────────────┐
│                                                      │
│  ┌──────────────┐  ┌──────────────┐               │
│  │  App Runner  │  │     ECS      │               │
│  │ (Simplest)   │  │ (AWS-native) │               │
│  └──────────────┘  └──────────────┘               │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐               │
│  │     EKS      │  │     ECR      │               │
│  │(Kubernetes)  │  │(Image Store) │               │
│  └──────────────┘  └──────────────┘               │
│                                                      │
│  All services integrate seamlessly                  │
└─────────────────────────────────────────────────────┘
```

**When to Use Each**:
- **App Runner**: Simple web apps, fastest deployment
- **ECS**: Microservices, AWS-first organizations
- **EKS**: Kubernetes expertise, multi-cloud portability
- **ECR**: Image storage for all above services

---

<a name="ecs"></a>
## 2. ECS - Elastic Container Service

### Overview

ECS is AWS's proprietary container orchestration service. It manages where and how containers run across a fleet of servers.

**Key Concept**: ECS handles scheduling, scaling, monitoring, and recovery automatically.

### Core Components

#### Task Definitions

A **blueprint** describing your containerized application:

```json
{
  "family": "web-app",
  "containerDefinitions": [{
    "name": "nginx",
    "image": "nginx:latest",
    "cpu": 256,              // 1024 = 1 vCPU
    "memory": 512,           // MB
    "essential": true,       // Stop all if this fails
    "portMappings": [{
      "containerPort": 80
    }],
    "environment": [{
      "name": "ENV",
      "value": "production"
    }]
  }],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512"
}
```

**Analogy**: Like a recipe - it doesn't cook the meal, it tells you how.

#### Tasks vs Services

```
TASK = Single running instance (one-time job)
SERVICE = Maintains desired count of tasks (long-running)

Example:
┌─────────────────────────────────────┐
│  ECS SERVICE (Desired: 3)           │
│                                      │
│  Task1   Task2   Task3              │
│   ✓       ✓       ✓                 │
│                                      │
│  Task2 crashes ❌                    │
│                                      │
│  Task1   Task4   Task3              │
│   ✓      ✓(new)   ✓                 │
│                                      │
│  Service auto-replaces failed tasks │
└─────────────────────────────────────┘
```

### Launch Types

#### EC2 Launch Type

**You manage servers**:

```
┌──────────────────────────────────┐
│  EC2 Instances (You manage)      │
│  ┌────────┐  ┌────────┐         │
│  │  t3.md │  │  t3.md │         │
│  │ ┌────┐ │  │ ┌────┐ │         │
│  │ │C1  │ │  │ │C3  │ │         │
│  │ │C2  │ │  │ │C4  │ │         │
│  │ └────┘ │  │ └────┘ │         │
│  └────────┘  └────────┘         │
│                                  │
│  You handle:                     │
│  - OS patches                    │
│  - Capacity planning             │
│  - Instance types                │
└──────────────────────────────────┘
```

**Pros**: More control, can be cheaper at scale
**Cons**: More operational overhead

#### Fargate Launch Type

**AWS manages infrastructure**:

```
┌──────────────────────────────────┐
│  No visible servers!             │
│  ┌────────┐  ┌────────┐         │
│  │ Task 1 │  │ Task 2 │         │
│  │(Fargate│  │(Fargate│         │
│  └────────┘  └────────┘         │
│                                  │
│  AWS handles:                    │
│  - Server provisioning           │
│  - Patching                      │
│  - Scaling infrastructure        │
│                                  │
│  You pay per-second              │
└──────────────────────────────────┘
```

**Pros**: Serverless, no management, pay-per-use
**Cons**: Slightly higher per-container cost

### Capacity Providers

Define infrastructure pools for task placement:

```
Strategy: Run 70% on-demand, 30% spot

┌────────────────────────────────────────┐
│  FARGATE (70% - Stable)                │
│  7 tasks at full price                 │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│  FARGATE_SPOT (30% - 70% cheaper)      │
│  3 tasks, can be interrupted           │
│  If interrupted → auto-replacement     │
└────────────────────────────────────────┘
```

### Service Auto Scaling

Automatically adjust task count based on metrics:

```
10 AM: Low traffic          2 PM: Traffic spike
┌──────────┐               ┌────────────────────┐
│ 2 Tasks  │               │    8 Tasks         │
│ CPU: 20% │  ──────────>  │    CPU: 75%        │
└──────────┘               │    Auto-scaled!    │
                           └────────────────────┘
```

**Scaling Policies**:
1. **Target Tracking**: Maintain target metric (e.g., 70% CPU)
2. **Step Scaling**: Add X tasks when metric crosses threshold
3. **Scheduled**: Scale at specific times

### Service Discovery

Containers find each other using DNS instead of IP addresses:

```
Without Service Discovery:
Frontend → 10.0.1.54:3000  (IP changes on restart ❌)

With Service Discovery:
Frontend → backend.local:3000  (DNS always resolves ✓)

┌────────────────────────────────┐
│  AWS Cloud Map                 │
│  backend.local → [10.0.1.5,    │
│                   10.0.1.8]    │
└────────────────────────────────┘
         ↑              ↓
    Auto-register   DNS query
         │              │
    Backend Tasks  Frontend Task
```

### Task Placement Strategies

Control where tasks run on EC2 instances:

1. **Spread**: Distribute across AZs (high availability)
2. **Binpack**: Pack densely (cost optimization)
3. **Random**: Random placement

```
SPREAD (High Availability):
AZ-1a      AZ-1b      AZ-1c
Task1      Task2      Task3
Task4      Task5      Task6

BINPACK (Cost Optimization):
Instance1  Instance2  Instance3
Task1       Empty     Empty
Task2
Task3      (Fill one before using next)
Task4
```

### Task Networking Modes

#### awsvpc (Recommended)

Each task gets its own network interface and IP:

```
┌────────────────────────────────┐
│  VPC (10.0.0.0/16)             │
│  ┌──────────┐  ┌──────────┐   │
│  │ Task1    │  │ Task2    │   │
│  │ ENI      │  │ ENI      │   │
│  │10.0.1.10 │  │10.0.1.11 │   │
│  └──────────┘  └──────────┘   │
│                                │
│  Each task = its own IP        │
│  Can apply security groups     │
└────────────────────────────────┘
```

**Required for Fargate**

#### bridge (Legacy)

Containers share host network, use port mapping:

```
┌────────────────────────────────┐
│  EC2 Instance (10.0.1.5)       │
│  Container1:80 → Host:32768    │
│  Container2:80 → Host:32769    │
└────────────────────────────────┘
```

### ECS Exec

Run commands inside running containers without SSH:

```
$ aws ecs execute-command \
    --cluster my-cluster \
    --task abc123 \
    --container nginx \
    --interactive \
    --command "/bin/bash"

bash-4.2$ ps aux
bash-4.2$ curl localhost:80
bash-4.2$ cat /var/log/app.log
```

**Features**:
- No SSH keys needed
- Works with Fargate
- All sessions logged to CloudWatch

### ECS Anywhere

Run ECS on your own servers (on-premises or other clouds):

```
┌─────────────────────────────────┐
│  AWS Cloud - ECS Control Plane  │
│  (Manages all containers)       │
└─────────────────────────────────┘
         │              │
         ▼              ▼
┌──────────────┐  ┌──────────────┐
│ AWS Fargate  │  │ Your Data    │
│              │  │ Center       │
│ Tasks        │  │ Tasks        │
└──────────────┘  └──────────────┘
```

**Use Cases**:
- Hybrid cloud deployments
- Gradual cloud migration
- On-premises compliance requirements

---

<a name="eks"></a>
## 3. EKS - Elastic Kubernetes Service

### Overview

EKS is AWS's managed Kubernetes service. **Kubernetes** is an industry-standard container orchestration platform.

**ECS vs EKS**:
- **ECS**: Simpler, AWS-specific
- **EKS**: More complex, portable across clouds

### Cluster Architecture

```
┌────────────────────────────────────┐
│  AWS-MANAGED CONTROL PLANE         │
│                                    │
│  - API Server (kubectl commands)   │
│  - Scheduler (pod placement)       │
│  - etcd (cluster state database)   │
│  - Multi-AZ for high availability  │
└────────────────────────────────────┘
              ↓
┌────────────────────────────────────┐
│  DATA PLANE (Worker Nodes)         │
│  You manage (or AWS with Fargate)  │
│                                    │
│  Node1     Node2     Node3         │
│  ┌────┐   ┌────┐   ┌────┐         │
│  │Pod │   │Pod │   │Pod │         │
│  │Pod │   │Pod │   │Pod │         │
│  └────┘   └────┘   └────┘         │
└────────────────────────────────────┘
```

**Kubernetes Terms**:
- **Pod**: Smallest unit, 1+ containers
- **Node**: Worker machine (EC2 or Fargate)
- **Deployment**: Desired state for pods
- **Service**: Network access to pods

### Managed Node Groups

EC2 instances that AWS helps you manage:

```
┌────────────────────────────────┐
│  MANAGED NODE GROUP            │
│  ┌──────────┐  ┌──────────┐   │
│  │ t3.large │  │ t3.large │   │
│  │ ┌──────┐ │  │ ┌──────┐ │   │
│  │ │ Pods │ │  │ │ Pods │ │   │
│  │ └──────┘ │  │ └──────┘ │   │
│  └──────────┘  └──────────┘   │
│                                │
│  AWS manages:                  │
│  - OS updates                  │
│  - Node registration           │
│  - Auto Scaling Group          │
└────────────────────────────────┘
```

**You configure**: Instance type, update strategy, scaling

### Fargate Profiles

Run pods on serverless Fargate compute:

```
Fargate Profile:
- Namespace: production
- Labels: tier=frontend

┌────────────────────────────────┐
│  Pods matching profile run     │
│  on Fargate (serverless)       │
│  ┌──────────┐  ┌──────────┐   │
│  │Fargate   │  │Fargate   │   │
│  │Pod       │  │Pod       │   │
│  └──────────┘  └──────────┘   │
│                                │
│  No EC2 instances to manage!   │
└────────────────────────────────┘
```

**When to Use**: Microservices, batch jobs, dev/test

### EKS Add-ons

Operational software managed by AWS:

1. **VPC-CNI**: Networking (pods get VPC IPs)
2. **CoreDNS**: DNS resolution
3. **kube-proxy**: Network routing
4. **EBS CSI Driver**: Persistent storage

```
┌────────────────────────────────┐
│  EKS Add-ons (AWS manages)     │
│  ┌────────┐  ┌──────────┐     │
│  │VPC-CNI │  │ CoreDNS  │     │
│  └────────┘  └──────────┘     │
│  ┌────────┐  ┌──────────┐     │
│  │kube-   │  │ EBS CSI  │     │
│  │proxy   │  │ Driver   │     │
│  └────────┘  └──────────┘     │
│                                │
│  Auto-updated & compatible     │
└────────────────────────────────┘
```

### IRSA (IAM Roles for Service Accounts)

Pods get their own IAM roles:

```
OLD: All pods share node role ❌
NEW: Each pod has its own role ✓

┌────────────────────────────────┐
│  Pod A (Web App)               │
│  IRSA: S3-ReadOnly-Role        │
│  Can: Read S3                  │
│  Can't: Write S3, Delete DB    │
└────────────────────────────────┘
┌────────────────────────────────┐
│  Pod B (Admin)                 │
│  IRSA: Admin-Role              │
│  Can: Everything               │
└────────────────────────────────┘
```

**Setup**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3Read
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: s3-reader  # Pod gets this role
```

### Karpenter

Intelligent, fast autoscaler for EC2 nodes:

```
Traditional Cluster Autoscaler:
Pod needs: 2 CPU, 4GB RAM
→ Waits 3-5 minutes
→ Adds m5.xlarge (4 CPU, 16GB) [Overkill!]

Karpenter:
Pod needs: 2 CPU, 4GB RAM, GPU: No, Spot: OK
→ Provisions c6i.large (2 CPU, 4GB) spot
→ Launches in 30 seconds
→ Perfect fit, 70% cheaper!
```

**Karpenter Features**:
- Just-in-time provisioning (~30 sec)
- 600+ instance types to choose from
- Intelligent bin-packing
- Automatic consolidation
- Spot instance support

```yaml
# Karpenter Provisioner
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
  limits:
    resources:
      cpu: 1000
  ttlSecondsAfterEmpty: 30  # Terminate empty nodes
  consolidation:
    enabled: true  # Optimize utilization
```

### AWS Load Balancer Controller

Automatically creates ALB/NLB for Kubernetes services:

```
You create Kubernetes Ingress:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: my-service
            port: 80

Controller automatically creates:
┌────────────────────────────────┐
│  Application Load Balancer     │
│  - Public URL: app.example.com │
│  - SSL/TLS                     │
│  - Routes to pods              │
└────────────────────────────────┘
```

### EKS Cost Optimization

**1. Right-Size Nodes**:
```
Before: m5.2xlarge (8 vCPU, 32 GB) - $0.384/hr
After:  c6i.xlarge  (4 vCPU, 8 GB)  - $0.17/hr
Savings: 56%
```

**2. Spot Instances**:
```
Critical pods → On-demand (100% price, stable)
Stateless pods → Spot (30% price, may interrupt)

With Karpenter: automatic spot fallback
```

**3. Fargate for Batch Jobs**:
```
Traditional: EC2 running 24/7 for 2hr/day jobs
Fargate: Pay only for 2hr/day
Savings: 90%
```

**4. Cluster Consolidation**:
```
Before: 5 small clusters (dev, staging, test, prod, ml)
After:  1 cluster with 5 namespaces
Savings: 4x control plane costs + overhead
```

**5. Monitor with Kubecost**:
- Per-pod cost visibility
- Identify waste
- Optimization recommendations

---

<a name="ecr"></a>
## 4. ECR - Elastic Container Registry

### Overview

ECR is AWS's container image registry (like private Docker Hub).

**Why ECR**:
- Private by default
- IAM integration
- Fast (same region pulls)
- Encrypted
- Multi-AZ replication

### Image Workflow

```
Developer Laptop:
$ docker build -t myapp:v1.0 .
$ docker tag myapp:v1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
$ docker push ...
        ↓
┌────────────────────────────────┐
│  ECR Repository                │
│  Images:                       │
│  - myapp:v1.0 (latest)         │
│  - myapp:v0.9                  │
│  - Encrypted at rest           │
│  - Vulnerability scanned       │
└────────────────────────────────┘
        ↓
ECS/EKS/Lambda pulls and runs
```

### Repository Policies

Control who can push/pull images:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCrossAccountPull",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::999999999999:root"
    },
    "Action": [
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage"
    ]
  }]
}
```

**Use Case**: CI/CD account pushes, production account pulls

### Image Scanning

Automatically detect vulnerabilities:

```
Push myapp:v1.0
    ↓
ECR scans for CVEs
    ↓
Scan Results:
✓ 0 Critical
⚠ 3 High (e.g., OpenSSL CVE-2024-1234)
⚠ 12 Medium
• 45 Low

Details:
CVE-2024-1234: OpenSSL vulnerability
Package: openssl 1.1.1
Fix: Upgrade to 1.1.1w
```

**Enhanced Scanning** (Amazon Inspector):
- OS packages
- Programming language libraries (Python, Node, Java)
- Continuous rescanning as new CVEs published

**CI/CD Integration**:
```yaml
steps:
  - build-image
  - push-to-ecr
  - wait-for-scan
  - check-critical-vulns:
      if critical: FAIL BUILD
      else: deploy
```

### Lifecycle Policies

Automatically delete old images:

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Keep last 10 images",
    "selection": {
      "tagStatus": "any",
      "countType": "imageCountMoreThan",
      "countNumber": 10
    },
    "action": {"type": "expire"}
  }, {
    "rulePriority": 2,
    "description": "Delete untagged images > 7 days",
    "selection": {
      "tagStatus": "untagged",
      "countType": "sinceImagePushed",
      "countUnit": "days",
      "countNumber": 7
    },
    "action": {"type": "expire"}
  }]
}
```

**Result**:
```
500 images (50GB) → 50 images (5GB)
Savings: $2/month/repo
```

### Cross-Region Replication

Automatically copy images to multiple regions:

```
Source: us-east-1
┌────────────────┐
│  myapp:v1.0    │
└────────────────┘
        ↓ (auto-replicate)
    ┌───┴───┐
    ↓       ↓
┌────────┐ ┌────────┐
│us-west-│ │eu-west-│
│   2    │ │   1    │
│myapp:  │ │myapp:  │
│v1.0    │ │v1.0    │
└────────┘ └────────┘

ECS in Oregon   ECS in Dublin
pulls locally   pulls locally
(fast!)         (fast!)
```

**Use Cases**:
- Disaster recovery
- Reduced latency
- Compliance (data residency)

### Pull-Through Cache

ECR caches images from public registries:

```
Without Cache:
ECS → Docker Hub (every pull, rate limited)

With Pull-Through Cache:
ECS → ECR Cache → Docker Hub (first pull only)
ECS → ECR Cache (subsequent pulls, fast!)

Setup:
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix dockerhub \
  --upstream-registry-url registry-1.docker.io

Usage:
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/dockerhub/nginx:latest
```

**Benefits**:
- No Docker Hub rate limits
- Faster pulls (cached in region)
- Improved reliability

### OCI Artifact Support

Store non-container artifacts:

```
ECR Repository can store:
┌────────────────────────────────┐
│  - Container images            │
│  - Helm charts                 │
│  - Terraform modules           │
│  - ML models                   │
│  - Policy bundles              │
│  - Any OCI-compliant artifact  │
└────────────────────────────────┘

Example: Push Helm chart
$ helm push my-chart-1.0.0.tgz \
    oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/
```

**Benefit**: One registry for everything

---

<a name="app-runner"></a>
## 5. App Runner

### Overview

App Runner is the **simplest** way to deploy containerized web apps. You point to code or an image, AWS handles everything else.

```
Traditional:                 App Runner:
1. Create VPC                1. Point to code/image
2. Configure subnets         2. Click deploy
3. Set up security groups
4. Create load balancer      Done! ✓
5. Configure ECS/EKS
6. Set up auto-scaling
7. Configure logging
...
```

**Target Audience**: Developers who want speed over control

### Architecture

```
┌────────────────────────────────┐
│  App Runner Service            │
│  (AWS manages all this)        │
│                                │
│  Public HTTPS Endpoint         │
│  https://abc.awsapprunner.com  │
│         ↓                      │
│  Built-in Load Balancer        │
│  (SSL, auto-scaling)           │
│         ↓                      │
│  Auto-scaled Containers        │
│  ┌────────┐  ┌────────┐       │
│  │Instance│  │Instance│       │
│  │   1    │  │   2    │       │
│  └────────┘  └────────┘       │
│  (Scales 0-25 automatically)   │
│         ↓                      │
│  Logs → CloudWatch             │
│  Metrics → CloudWatch          │
└────────────────────────────────┘
```

**You DON'T configure**:
- Load balancers
- VPCs
- SSL certificates
- Scaling policies

**You DO configure**:
- Source (GitHub or ECR)
- CPU/memory
- Environment variables

### Automatic Scaling

```
9 AM: Low traffic          12 PM: Lunch rush
┌──────────┐              ┌────────────────┐
│1 Instance│              │  10 Instances  │
│10 req/s  │  ──────────> │  500 req/s     │
└──────────┘              │  Auto-scaled!  │
                          └────────────────┘
                                  ↓
                          3 PM: Back to normal
                          ┌──────────┐
                          │2 Instance│
                          │50 req/s  │
                          │Scaled    │
                          │down to   │
                          │save $    │
                          └──────────┘
```

**Configuration**:
```json
{
  "maxConcurrency": 100,  // Requests per instance
  "maxSize": 25,          // Max instances
  "minSize": 1            // Min instances (always running)
}
```

### Source Connections

#### 1. GitHub Repository

App Runner builds from code:

```
┌────────────────────────────────┐
│  GitHub Repo                   │
│  - Source code (Node/Python/   │
│    Ruby/Go/.NET)               │
│  - apprunner.yaml              │
└────────────────────────────────┘
        ↓ (auto-deploy on push)
┌────────────────────────────────┐
│  App Runner Build              │
│  1. Pull code                  │
│  2. Install dependencies       │
│  3. Build app                  │
│  4. Create container           │
│  5. Deploy                     │
└────────────────────────────────┘
```

**apprunner.yaml**:
```yaml
version: 1.0
runtime: nodejs16
build:
  commands:
    build:
      - npm install
      - npm run build
run:
  command: npm start
  network:
    port: 8080
  env:
    - name: NODE_ENV
      value: production
```

#### 2. ECR Image

Pre-built container image:

```
┌────────────────────────────────┐
│  ECR Repository                │
│  myapp:v1.0                    │
└────────────────────────────────┘
        ↓ (App Runner pulls)
┌────────────────────────────────┐
│  App Runner Service            │
│  - Runs container              │
│  - Auto-scales                 │
│  - Manages HTTPS               │
└────────────────────────────────┘
```

**Auto-deploy**: New ECR push = automatic deployment

### VPC Connector

Access private VPC resources:

```
Without VPC Connector:
App Runner → Can only access public internet

With VPC Connector:
App Runner → VPC Connector → Your VPC
                               ↓
                          ┌────────────┐
                          │ RDS        │
                          │ ElastiCache│
                          │ Internal   │
                          │ Services   │
                          └────────────┘
```

**Configuration**:
```json
{
  "egressConfiguration": {
    "vpcConnectorArn": "arn:aws:apprunner:...:vpcconnector/my-connector",
    "egressType": "VPC"
  }
}
```

### When to Use App Runner

**✓ Use when**:
- Simple web APIs/frontends
- Want fastest deployment
- Variable traffic
- Small-medium scale

**✗ Don't use when**:
- Need fine-grained infrastructure control
- Non-HTTP workloads
- Complex networking
- Very high scale (>1000 req/s)

---

<a name="comparison"></a>
## 6. Comparison & Best Practices

### Decision Tree

```
Need containers?
  ├─ Yes → Priority?
  │   ├─ Simplicity → App Runner
  │   ├─ AWS-native → ECS
  │   │   ├─ No servers → Fargate
  │   │   └─ Cost optimization → EC2
  │   └─ Kubernetes → EKS
  │       ├─ No servers → Fargate
  │       └─ Maximum efficiency → Managed Nodes + Karpenter
  └─ No → Lambda
```

### Feature Matrix

| Feature | App Runner | ECS | EKS |
|---------|-----------|-----|-----|
| Ease of Use | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Control | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Learning Curve | 1 day | 1 week | 2-4 weeks |
| Portability | ✗ | ✗ | ✓ |
| Cost (small) | $$ | $$ | $$$ |
| Cost (large) | $$$ | $$ | $ |

### Best Practices

#### Security

**1. Least Privilege IAM**:
```
✓ Use IRSA/Pod Identity (EKS)
✓ Separate roles per task (ECS)
✗ Don't use broad permissions
```

**2. Network Isolation**:
```
VPC Structure:
┌────────────────────────────────┐
│  Public Subnet                 │
│  - Load balancers only         │
└────────────────────────────────┘
┌────────────────────────────────┐
│  Private Subnet                │
│  - Containers (no internet)    │
└────────────────────────────────┘
┌────────────────────────────────┐
│  Data Subnet                   │
│  - Databases (most restricted) │
└────────────────────────────────┘
```

**3. Image Security**:
```
✓ Scan images before deploy
✓ Use minimal base images (alpine, distroless)
✓ Update regularly
✗ Never include secrets in images
```

**4. Secrets Management**:
```
✓ AWS Secrets Manager
✓ SSM Parameter Store
✓ Inject at runtime
✗ Never hardcode
```

#### Cost Optimization

**1. Right-Size**:
```
Monitor actual usage → Adjust resources

Before: 2 vCPU, 4 GB → $70/month
After:  500m CPU, 1 GB → $18/month (74% savings)
```

**2. Use Spot Wisely**:
```
Critical services → On-demand
Stateless services → Spot (70% cheaper)
```

**3. Auto-Scale**:
```
Scale down off-hours
Set appropriate min/max
Use target tracking
```

**4. Reserved Capacity**:
```
Predictable workloads → Savings Plans (40-60% off)
Base load → Reserved
Bursts → On-demand/spot
```

#### Observability

**The Three Pillars**:

1. **Metrics**: CPU, memory, requests, errors
2. **Logs**: Application logs, error messages
3. **Traces**: Request flow through services

**Structured Logging**:
```json
// Good
{
  "timestamp": "2026-02-06T10:30:00Z",
  "level": "ERROR",
  "service": "user-api",
  "message": "DB connection failed",
  "retry_count": 3
}

// Bad
"2026-02-06 ERROR: DB failed after 3 retries"
```

**Meaningful Alarms**:
```
- High error rate (>10 errors/5min)
- High latency (>2s p99)
- Low health (<2 healthy instances)
```

#### High Availability

**Multi-AZ**:
```
Region: us-east-1
┌────────┐ ┌────────┐ ┌────────┐
│ AZ-1a  │ │ AZ-1b  │ │ AZ-1c  │
│ Task1  │ │ Task2  │ │ Task3  │
│ Task4  │ │ Task5  │ │        │
└────────┘ └────────┘ └────────┘

If AZ-1a fails → Tasks move to other AZs
```

**Health Checks**:
```javascript
app.get('/health', async (req, res) => {
  try {
    await db.ping();
    await redis.ping();
    res.status(200).json({status: 'healthy'});
  } catch (error) {
    res.status(503).json({status: 'unhealthy'});
  }
});
```

**Graceful Shutdown**:
```javascript
process.on('SIGTERM', async () => {
  server.close();              // Stop new requests
  await finishCurrentRequests();
  await db.close();
  process.exit(0);
});
```

#### CI/CD

**Deployment Strategies**:

1. **Rolling**: Replace instances one by one
2. **Blue-Green**: Instant switch, easy rollback
3. **Canary**: Test with small traffic first

**Example Pipeline**:
```yaml
steps:
  - build-image
  - push-to-ecr
  - scan-image:
      if critical vulns: FAIL
  - update-task-definition
  - deploy-to-ecs
  - wait-for-stable
```

---

<a name="examples"></a>
## 7. Real-World Examples

### Example 1: Startup MVP (App Runner)

```
┌────────────────────────────────┐
│  App Runner Service            │
│  - Source: GitHub              │
│  - Auto HTTPS                  │
│  - Auto-scaling (1-10)         │
└────────────────────────────────┘
        ↓
┌────────────────────────────────┐
│  Secrets Manager               │
│  - DB password                 │
│  - API keys                    │
└────────────────────────────────┘

Cost: $10-50/month
Setup: 30 minutes
```

### Example 2: E-commerce (ECS Fargate)

```
Route 53 → ALB → ECS Services
                  ├─ User API (2-10 tasks)
                  ├─ Product API (2-10 tasks)
                  └─ Order API (2-10 tasks)
                        ↓
                  Aurora PostgreSQL

All images in ECR
CloudWatch monitoring
Auto-scaling on CPU/memory

Cost: $200-500/month
Setup: 1-2 days
```

### Example 3: ML Platform (EKS)

```
┌────────────────────────────────┐
│  EKS Cluster                   │
│  ┌────────────────────────┐   │
│  │ Training Namespace     │   │
│  │ - GPU nodes (p3.8xl)   │   │
│  │ - Fargate batch jobs   │   │
│  └────────────────────────┘   │
│  ┌────────────────────────┐   │
│  │ Inference Namespace    │   │
│  │ - CPU nodes (c6i.2xl)  │   │
│  │ - API servers          │   │
│  └────────────────────────┘   │
│                                │
│  Karpenter: Auto-scale GPU     │
│  Spot: Cost optimization       │
│  Models: S3                    │
└────────────────────────────────┘

Cost: $2,000-10,000/month
Setup: 1-2 weeks
```

---

## Key Takeaways

1. **Start Simple**: App Runner → ECS → EKS (evolve as needed)
2. **Choose by Expertise**: ECS (AWS-native) vs EKS (K8s)
3. **Always Use ECR**: Integrated, secure image storage
4. **Automate Everything**: IaC, CI/CD, auto-scaling
5. **Monitor Proactively**: Metrics, logs, traces
6. **Optimize Costs**: Right-size, spot, auto-scale
7. **Design for Failure**: Multi-AZ, health checks
8. **Security First**: IAM, secrets, isolation, scanning

---

## Additional Resources

### AWS Documentation
- [ECS Docs](https://docs.aws.amazon.com/ecs/)
- [EKS Docs](https://docs.aws.amazon.com/eks/)
- [ECR Docs](https://docs.aws.amazon.com/ecr/)
- [App Runner Docs](https://docs.aws.amazon.com/apprunner/)

### Community
- [ECS Patterns](https://github.com/aws-samples/aws-ecs-patterns)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Container Roadmap](https://github.com/aws/containers-roadmap)

### Tools
- [AWS Pricing Calculator](https://calculator.aws/)
- [Kubecost](https://www.kubecost.com/) (EKS cost visibility)

---

**End of Guide**
This guide covers AWS Container Services conceptually with practical insights for 2026. Use it as a reference for understanding, designing, and implementing container workloads on AWS.