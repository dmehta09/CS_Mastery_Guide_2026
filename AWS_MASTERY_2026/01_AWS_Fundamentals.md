# AWS Fundamentals: Complete Conceptual Guide

> **Last Updated:** February 2026
> **Audience:** Beginners to Intermediate Cloud Engineers
> **Focus:** Conceptual understanding of AWS core building blocks

---

## Table of Contents

1. [Global Infrastructure](#global-infrastructure)
2. [Local Zones & Wavelength Zones](#local-zones--wavelength-zones)
3. [AWS Account Structure](#aws-account-structure)
4. [AWS Organizations](#aws-organizations)
5. [Service Control Policies (SCPs)](#service-control-policies-scps)
6. [AWS Control Tower](#aws-control-tower)
7. [Landing Zones](#landing-zones)
8. [AWS Well-Architected Framework](#aws-well-architected-framework)
9. [Shared Responsibility Model](#shared-responsibility-model)
10. [AWS Support Plans](#aws-support-plans)
11. [AWS Pricing Models](#aws-pricing-models)
12. [Free Tier](#free-tier)

---

## Global Infrastructure

### What It Is

AWS Global Infrastructure is the **physical foundation** of AWS cloud services—the actual data centers, networking equipment, and connectivity that powers everything you build on AWS. It's organized into three main components:

1. **Regions** - Geographic areas containing multiple data centers
2. **Availability Zones (AZs)** - Isolated data center clusters within a region
3. **Edge Locations** - Content delivery and low-latency service endpoints worldwide

### Why It Exists

**Problem it solves:**
- Applications need to be close to users for low latency
- Systems must survive hardware failures, power outages, and natural disasters
- Different countries have data residency laws requiring data to stay within borders
- Global businesses need consistent infrastructure across continents

### Regions

**What they are:**
A Region is a **separate geographic area** (like "US East", "Europe Frankfurt", "Asia Pacific Tokyo") that contains multiple physically isolated data centers. As of 2026, AWS has 30+ Regions worldwide.

**Key characteristics:**
- Each Region is **completely independent** from other Regions
- Resources in one Region don't automatically replicate to others
- You choose which Region(s) to deploy your applications in
- Pricing, available services, and compliance certifications vary by Region

**Naming convention:**
- `us-east-1` (Northern Virginia, USA)
- `eu-west-1` (Ireland)
- `ap-southeast-1` (Singapore)

**When to use which Region:**
- Deploy in the Region **closest to your users** for lowest latency
- Choose Regions based on **data residency requirements** (e.g., GDPR in Europe)
- Use multiple Regions for **disaster recovery** (if one Region fails, another handles traffic)

```
ASCII Diagram: AWS Regions Globally

        North America              Europe              Asia Pacific
    ┌──────────────────┐      ┌──────────────┐    ┌──────────────┐
    │   us-east-1      │      │  eu-west-1   │    │ ap-south-1   │
    │   us-west-2      │      │  eu-central-1│    │ ap-southeast-1│
    │   ca-central-1   │      └──────────────┘    │ ap-northeast-1│
    └──────────────────┘                          └──────────────┘
           │                        │                     │
           └────────────────────────┴─────────────────────┘
                   Global AWS Backbone Network
                   (Private fiber connections)
```

### Availability Zones (AZs)

**What they are:**
An Availability Zone is **one or more discrete data centers** within a Region, each with independent power, cooling, and networking. Each Region has **2-6 AZs** (most have 3+).

**Key characteristics:**
- AZs within a Region are physically separated (typically 10s of miles apart)
- Connected by **low-latency, high-bandwidth private fiber** (sub-millisecond latency)
- Isolated from failures in other AZs (separate power grids, flood zones)
- Named with Region code + letter: `us-east-1a`, `us-east-1b`, `us-east-1c`

**Why multiple AZs matter:**

If you deploy your application in a **single AZ** and that data center loses power, your entire application goes down. By deploying across **multiple AZs**, you achieve **high availability**—if one AZ fails, your application continues running in the others.

```
ASCII Diagram: Availability Zones within a Region

        Region: us-east-1 (Northern Virginia)
    ┌──────────────────────────────────────────────┐
    │                                              │
    │  ┌────────────┐  ┌────────────┐  ┌────────────┐
    │  │   AZ-1a    │  │   AZ-1b    │  │   AZ-1c    │
    │  │            │  │            │  │            │
    │  │ Data       │  │ Data       │  │ Data       │
    │  │ Centers    │  │ Centers    │  │ Centers    │
    │  │            │  │            │  │            │
    │  │ Power: A   │  │ Power: B   │  │ Power: C   │
    │  │ Network: X │  │ Network: Y │  │ Network: Z │
    │  └────────────┘  └────────────┘  └────────────┘
    │         │               │               │
    │         └───────────────┴───────────────┘
    │          Low-latency private network
    │         (< 1ms between AZs typically)
    └──────────────────────────────────────────────┘

    If AZ-1a loses power → Applications in AZ-1b and AZ-1c
    continue running without interruption
```

**Real-world example:**

You run a web application:
- **Load balancer** distributes traffic across AZs
- **Web servers** deployed in AZ-1a, AZ-1b, AZ-1c
- **Database** automatically replicates data to standby in another AZ
- If AZ-1a experiences a power outage, traffic automatically routes to servers in AZ-1b and AZ-1c

### Edge Locations

**What they are:**
Edge Locations are **mini data centers** (300+ worldwide as of 2026) that cache and deliver content closer to end users. They're part of AWS's content delivery network called **CloudFront**.

**Key characteristics:**
- Many more Edge Locations than Regions (10x+)
- Don't host your full applications—only cache and deliver content
- Handle services like CloudFront (CDN), Route 53 (DNS), AWS WAF (firewall)
- Located in major cities globally, even where AWS has no full Regions

**Why they exist:**

If your website is hosted in `us-east-1` and a user in Australia requests your homepage, the data travels halfway around the world (~200ms latency). With Edge Locations, static content (images, videos, CSS) is cached in Sydney, reducing latency to ~10ms.

```
ASCII Diagram: Edge Locations in Action

User in Sydney          Edge Location Sydney      Origin (us-east-1)
     │                         │                         │
     │  1. Request image       │                         │
     ├────────────────────────>│                         │
     │                         │  2. Cache miss?         │
     │                         │  Check if cached        │
     │                         │         │               │
     │                         │  3. If not cached       │
     │                         ├────────────────────────>│
     │                         │  4. Fetch from origin   │
     │                         │<────────────────────────┤
     │                         │  5. Cache locally       │
     │  6. Deliver (10ms)      │     Store for 24hrs     │
     │<────────────────────────┤                         │
     │                         │                         │
     │  7. Next user requests  │                         │
     │     same image          │                         │
     ├────────────────────────>│                         │
     │  8. Served from cache   │   (No trip to origin)   │
     │<────────────────────────┤                         │
     │     (~10ms, not 200ms)  │                         │

First request: 200ms (Sydney → Virginia → Sydney)
Cached requests: 10ms (Sydney → Sydney Edge Location)
```

**What's cached at Edge Locations:**
- Website static assets (images, CSS, JavaScript)
- Video streaming content
- DNS query responses
- API responses (with configuration)

### Choosing Your Infrastructure Strategy

**Single Region, Multiple AZs (Most Common):**
- Deploy application across 2-3 AZs in one Region
- Protects against data center failures
- Good for: Most applications, cost-effective high availability

**Multi-Region (Advanced):**
- Deploy identical infrastructure in multiple Regions
- Protects against entire Region failures (extremely rare)
- Good for: Global applications, disaster recovery, compliance requirements
- More complex and expensive

**Global Edge Acceleration:**
- Use Edge Locations for content delivery
- Good for: Websites with global audience, video streaming, downloadable files

```
ASCII Diagram: Typical Architecture - Single Region, Multi-AZ

                          User Traffic
                               │
                               ▼
                    ┌──────────────────┐
                    │  CloudFront CDN  │  ← Edge Locations (Global)
                    │  (Edge Locations)│
                    └──────────────────┘
                               │
                               ▼
        ═══════════════════════════════════════════════
        Region: us-east-1
        ┌─────────────────────────────────────────────┐
        │                                             │
        │         ┌────────────────────┐              │
        │         │  Load Balancer     │              │
        │         │  (Multi-AZ)        │              │
        │         └────────────────────┘              │
        │           │            │                    │
        │           ▼            ▼                    │
        │    ┌──────────┐  ┌──────────┐              │
        │    │  AZ-1a   │  │  AZ-1b   │              │
        │    │          │  │          │              │
        │    │ Web      │  │ Web      │              │
        │    │ Servers  │  │ Servers  │              │
        │    │          │  │          │              │
        │    │ Database │  │ Database │              │
        │    │ Primary  │  │ Standby  │              │
        │    └──────────┘  └──────────┘              │
        │                                             │
        └─────────────────────────────────────────────┘
```

### Key Takeaways

✓ **Regions** = Geographic areas, fully independent, choose based on user location/compliance
✓ **Availability Zones** = Multiple data centers per Region, deploy across them for high availability
✓ **Edge Locations** = Global caching points for fast content delivery
✓ **Best practice:** Always use multiple AZs within a Region for production workloads
✓ **Global reach:** Combine Regional deployment with Edge Location caching for optimal performance

---

## Local Zones & Wavelength Zones

### What They Are

Local Zones and Wavelength Zones are **extensions of AWS Regions** that bring compute and storage services **closer to specific cities or mobile networks**, solving the "last mile" latency problem.

Think of them as **mini AWS data centers** placed in locations where AWS doesn't have a full Region, but where customers need ultra-low latency.

### Why They Exist

**Problem they solve:**

Even with Regions and Edge Locations, some applications need **single-digit millisecond latency**:
- Real-time gaming (multiplayer games with 100+ players)
- Live video production and broadcasting
- Machine learning inference for autonomous vehicles
- Augmented/Virtual Reality applications
- High-frequency trading

**The limitation:**
- Edge Locations only cache content—they can't run your custom applications
- Full Regions are expensive and take years to build
- Some cities have large populations but no nearby AWS Region

**The solution:**
Local Zones and Wavelength Zones provide a middle ground—enough AWS services (compute, storage, databases) to run latency-sensitive applications, without the full breadth of a complete Region.

### Local Zones

**What they are:**
A Local Zone is an **AWS infrastructure deployment** in a major city, connected to a parent AWS Region. As of 2026, AWS has Local Zones in cities like Los Angeles, Miami, Boston, Houston, and internationally.

**Key characteristics:**
- Provide core services: EC2 (compute), EBS (storage), VPC (networking), RDS (databases)
- Connected to parent Region via AWS private network
- Named like: `us-east-1-bos-1a` (Boston Local Zone, parent Region is us-east-1)
- Single-digit millisecond latency to end users in that city

**How they work:**

```
ASCII Diagram: Local Zone Architecture

Parent Region (us-east-1, Virginia)
┌────────────────────────────────────────┐
│  Full AWS Services (200+)             │
│  - All compute, storage, AI/ML, etc.  │
│                                        │
│  Your control plane:                  │
│  - IAM, CloudWatch, billing           │
│  - Management and orchestration       │
└────────────────────────────────────────┘
              │
              │ AWS Private Network
              │ (~10-20ms latency)
              ▼
Local Zone (Boston, us-east-1-bos-1a)
┌────────────────────────────────────────┐
│  Subset of services:                   │
│  - EC2 instances (compute)             │
│  - EBS volumes (storage)               │
│  - VPC subnets (networking)            │
│  - Application Load Balancers          │
│                                        │
│  Your workloads running HERE:          │
│  - Latency-sensitive applications      │
│  - Serve Boston metro area users       │
└────────────────────────────────────────┘
              │
              │ Internet
              │ (~1-5ms latency)
              ▼
       End Users in Boston
```

**Real-world use case:**

A video editing company in Los Angeles needs to:
- Render 4K video in real-time
- Collaborate with multiple editors simultaneously
- Upload/download large video files instantly

**Without Local Zone:**
Files travel to `us-west-1` (Northern California, 300 miles away) = 15-30ms latency
**With Local Zone:**
Files stay in `us-west-2-lax-1a` (Los Angeles) = 1-3ms latency

### Wavelength Zones

**What they are:**
Wavelength Zones are AWS infrastructure **embedded directly inside 5G mobile network operators' data centers**. They're even closer to end users than Local Zones—literally inside the mobile carrier's network.

**Key characteristics:**
- Hosted inside telecom partner facilities (Verizon, Vodafone, KDDI, SK Telecom)
- Ultra-low latency to mobile devices (typically <10ms)
- Traffic never leaves the carrier's network to reach your application
- Provide EC2, ECS, EKS for compute workloads

**Why they're different from Local Zones:**

| Aspect | Local Zones | Wavelength Zones |
|--------|-------------|------------------|
| **Location** | Major cities | Inside telecom carrier facilities |
| **Target users** | General city population | Mobile device users on specific carriers |
| **Latency** | 1-5ms | <10ms to mobile devices |
| **Network path** | Internet → Local Zone | Mobile network → Wavelength (no internet hop) |

**How they work:**

```
ASCII Diagram: Wavelength Zone in 5G Network

                      AWS Region
                   (Control Plane)
                          │
                          │ Management
                          ▼
    ┌───────────────────────────────────────────────┐
    │  Telecom Carrier Data Center (e.g., Verizon) │
    │                                               │
    │  ┌─────────────────────────────────┐          │
    │  │  Wavelength Zone                │          │
    │  │                                 │          │
    │  │  AWS Compute Resources:         │          │
    │  │  - EC2 instances                │          │
    │  │  - Your application             │          │
    │  │                                 │          │
    │  └─────────────────────────────────┘          │
    │               │                               │
    │               │ Direct connection             │
    │               │ (no internet routing)         │
    │               ▼                               │
    │  ┌─────────────────────────────────┐          │
    │  │  5G Core Network                │          │
    │  │  (Carrier infrastructure)       │          │
    │  └─────────────────────────────────┘          │
    │               │                               │
    └───────────────┼───────────────────────────────┘
                    │
                    │ 5G Signal
                    ▼
         Mobile Devices (Phones, IoT)
         Connected to this carrier

Traffic flow: Mobile device → Carrier 5G → Wavelength Zone
(Latency: 5-10ms, never touches public internet)
```

**Real-world use cases:**

**1. Connected Vehicles:**
Self-driving cars need to process sensor data instantly. Data goes from car → 5G network → Wavelength Zone for ML inference → response back to car, all in <10ms.

**2. AR/VR Gaming:**
VR headset streams game rendering. Graphics processing happens in Wavelength Zone, delivered over 5G—user doesn't experience motion sickness from lag.

**3. Smart Factories:**
Industrial robots on 5G network communicate with control systems in Wavelength Zone. Ultra-low latency ensures precise coordination.

### When to Use What

```
Latency Sensitivity Spectrum:

Standard Web App            Real-time Gaming        Autonomous Vehicles
Low latency needs           Medium latency needs    Ultra-low latency needs
        │                           │                        │
        ▼                           ▼                        ▼
┌──────────────┐          ┌──────────────┐        ┌──────────────┐
│   Standard   │          │    Local     │        │  Wavelength  │
│   Region     │          │    Zones     │        │    Zones     │
│   + AZs      │          │              │        │              │
│              │          │              │        │              │
│ 20-100ms     │          │  1-5ms       │        │  <10ms       │
│ latency      │          │  latency     │        │  latency     │
└──────────────┘          └──────────────┘        └──────────────┘

Add CloudFront Edge Locations for all scenarios to cache static content
```

**Decision matrix:**

- **Use standard Region + AZs:** Most applications (web, mobile backends, APIs)
- **Use Local Zones:** Latency-sensitive apps in specific cities (media, gaming, finance)
- **Use Wavelength Zones:** Mobile-first apps requiring carrier network integration (AR/VR, IoT, connected vehicles)

### Key Takeaways

✓ **Local Zones** = AWS services in major cities, 1-5ms latency to local users
✓ **Wavelength Zones** = AWS inside carrier networks, <10ms to mobile devices
✓ Both extend Regions without requiring full regional infrastructure
✓ You manage resources from parent Region, but workloads run locally
✓ Use only when standard Region latency (20-100ms) isn't good enough

---

## AWS Account Structure

### What It Is

An AWS Account is the **fundamental container** for all your AWS resources. Think of it as a **security boundary and billing unit**—everything inside one account is isolated from other accounts, and all costs are tracked separately.

### Why It Exists

**Problem it solves:**

Without account boundaries:
- Different teams would share resources → security risks
- Development could accidentally break production
- No way to separate billing between departments
- Impossible to apply different security rules to different environments

**The solution:**
AWS Accounts provide **isolation**, **security**, and **organizational structure**. Large enterprises use dozens or hundreds of accounts.

### Account Basics

**What's in an account:**
- **12-digit Account ID:** Unique identifier (e.g., `123456789012`)
- **Root user:** The email address that created the account (has unlimited permissions)
- **IAM users and roles:** Identities that can access account resources
- **All AWS resources:** EC2 instances, S3 buckets, databases, etc.
- **Billing:** Separate bill for all resources in this account

**Key characteristics:**
- Accounts are **completely isolated** by default
- Resources in one account can't access resources in another (unless explicitly permitted)
- Each account has its own **security controls** and **compliance boundaries**
- Billing is tracked per-account, but can be consolidated

```
ASCII Diagram: AWS Account Anatomy

AWS Account ID: 123456789012
┌───────────────────────────────────────────────────┐
│  Root User: company@example.com                   │
│  (Should NEVER be used day-to-day)               │
├───────────────────────────────────────────────────┤
│                                                   │
│  IAM Users & Roles:                              │
│  ├─ developer@company.com (limited permissions)  │
│  ├─ admin@company.com (admin permissions)        │
│  └─ ci-cd-role (for automated deployments)       │
│                                                   │
├───────────────────────────────────────────────────┤
│                                                   │
│  AWS Resources:                                   │
│  ├─ 20 EC2 instances (us-east-1)                 │
│  ├─ 5 S3 buckets                                 │
│  ├─ 3 RDS databases                              │
│  └─ VPCs, Lambda functions, etc.                 │
│                                                   │
├───────────────────────────────────────────────────┤
│                                                   │
│  Billing:                                         │
│  Monthly total: $5,432.10                        │
│  (All resources above)                           │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Single Account vs. Multi-Account Strategy

**Single Account (Small organizations):**
- Simple to manage
- Lower administrative overhead
- All resources share same security boundaries
- Good for: Startups, small teams, individual projects

**Multi-Account (Medium to large organizations):**
- Stronger security isolation
- Separate billing per team/department/environment
- Different compliance requirements per account
- Better blast radius containment (if one account is compromised, others are safe)

**Common multi-account patterns:**

```
ASCII Diagram: Typical Multi-Account Structure

Organization Root
        │
        ├─── Production Account (ID: 111111111111)
        │    └─ Only production workloads
        │    └─ Strict access controls
        │    └─ High availability requirements
        │
        ├─── Development Account (ID: 222222222222)
        │    └─ Developers have more permissions
        │    └─ Can create/delete resources freely
        │    └─ Lower cost, non-critical
        │
        ├─── Staging Account (ID: 333333333333)
        │    └─ Pre-production testing
        │    └─ Mirrors production setup
        │
        ├─── Security/Audit Account (ID: 444444444444)
        │    └─ Centralized logging
        │    └─ Security monitoring tools
        │    └─ Compliance reports
        │
        └─── Shared Services Account (ID: 555555555555)
             └─ CI/CD pipelines
             └─ Container registries
             └─ Shared networking (Transit Gateway)
```

### Why Multiple Accounts Matter

**Security example:**

**Single account problem:**
A developer accidentally grants public internet access to a production database because they were testing something.

**Multi-account solution:**
Developers work in the Development account (ID: 222222222222). They have zero access to the Production account (ID: 111111111111). Even if they make mistakes, production is completely unaffected.

**Cost example:**

**Single account problem:**
Your AWS bill is $50,000/month but you can't tell how much each department is spending.

**Multi-account solution:**
- Marketing account: $15,000/month
- Engineering account: $25,000/month
- Data Science account: $10,000/month

Each team gets their own bill and budget.

### Account Creation

**How accounts are created:**

```bash
# Accounts can be created via AWS Organizations (explained in next section)
# or manually through the AWS Console

# Manual creation:
# 1. Go to aws.amazon.com
# 2. Create account with unique email address
# 3. Provide payment method
# 4. Verify identity

# Programmatic creation (via Organizations):
aws organizations create-account \
  --email dev-team@company.com \     # Unique email required
  --account-name "Development"       # Human-readable name
```

**Important constraints:**
- Each account needs a **unique email address** (use email aliases like `aws+dev@company.com`)
- Root user credentials should be stored securely and rarely used
- Use IAM users or roles for day-to-day access, never the root user

### Root User vs. IAM Users

**Root user:**
- The email address that created the account
- Has **unlimited permissions** (can delete the entire account)
- Cannot be restricted by any policy
- Should only be used for initial setup and specific account-level tasks

**IAM users/roles:**
- Created inside the account
- Can have limited permissions
- Should be used for all day-to-day operations
- Can be audited and controlled

```
ASCII Diagram: Root User vs. IAM Hierarchy

AWS Account (123456789012)
        │
        ├─── Root User: ceo@company.com
        │    ├─ Can do ANYTHING
        │    ├─ Close account
        │    ├─ Change billing settings
        │    └─ Should be MFA-protected and locked away
        │
        └─── IAM (Identity & Access Management)
             │
             ├─── IAM Users
             │    ├─ alice@company.com (Developer)
             │    │   └─ Can create EC2 instances, S3 buckets
             │    │   └─ Cannot delete account or change billing
             │    │
             │    └─ bob@company.com (Admin)
             │        └─ Can manage users and resources
             │        └─ Cannot close account (only root can)
             │
             └─── IAM Roles
                  ├─ EC2-to-S3-Role
                  │   └─ Allows EC2 instances to read from S3
                  │
                  └─ Lambda-Execution-Role
                      └─ Allows Lambda functions to write logs
```

### Key Takeaways

✓ **AWS Account** = Security boundary, billing unit, resource container
✓ **12-digit Account ID** uniquely identifies each account
✓ **Root user** should be locked down with MFA and never used daily
✓ **Multi-account strategy** improves security, cost tracking, and compliance
✓ Common pattern: Separate accounts for Production, Development, Staging, Security
✓ Accounts can be managed centrally using **AWS Organizations** (next section)

---

## AWS Organizations

### What It Is

AWS Organizations is a **free service** that lets you centrally manage and govern multiple AWS accounts. Think of it as the **control center** for your entire AWS footprint—instead of managing 50 accounts individually, you manage them all from one place.

### Why It Exists

**Problem it solves:**

Imagine you have 20 AWS accounts for different teams and environments. Without Organizations:
- Each account bills separately → 20 different invoices
- No way to enforce company-wide security policies
- No centralized view of all resources
- Each account admin works independently → inconsistent configurations
- No way to share resources (like reserved instances) across accounts

**The solution:**
Organizations provides **centralized billing**, **policy enforcement**, **account hierarchy**, and **cross-account resource sharing**.

### How It Works

You designate one AWS account as the **management account** (formerly called "master account"). This account creates and manages all other accounts, called **member accounts**.

```
ASCII Diagram: AWS Organization Structure

        Management Account (111111111111)
        - Pays all bills
        - Creates and manages member accounts
        - Applies organization-wide policies
                    │
                    │
        ┌───────────┴───────────┐
        │  AWS Organization     │
        └───────────┬───────────┘
                    │
        ┌───────────┴────────────────┬──────────────┐
        │                            │              │
        ▼                            ▼              ▼
Production OU          Development OU        Security OU
        │                            │              │
    ┌───┴───┐                   ┌───┴───┐          │
    ▼       ▼                   ▼       ▼          ▼
  Prod     Prod              Dev-1   Dev-2     Log Archive
  Web      DB               Account  Account    Account
 (222..)  (333..)          (444..)  (555..)    (666..)

OU = Organizational Unit (folder grouping accounts)
```

### Key Components

**1. Management Account**
- The "parent" account that created the Organization
- Pays the consolidated bill for all member accounts
- Only account that can invite or create new accounts
- Cannot be restricted by Service Control Policies (SCPs)
- Should be highly secured (limited human access)

**2. Member Accounts**
- Regular AWS accounts joined to the Organization
- Inherit policies from the management account
- Can be grouped into Organizational Units (OUs)
- Billing rolls up to management account
- Can be moved between OUs or removed from Organization

**3. Organizational Units (OUs)**
- Logical groupings of accounts (like folders)
- Hierarchical structure (OUs can contain other OUs)
- Policies applied to an OU affect all accounts inside it
- Common OU patterns: by environment (Prod, Dev), by team (Engineering, Marketing), by compliance (PCI, HIPAA)

```
ASCII Diagram: Hierarchical OU Structure

Root (Organization)
    │
    ├─── Production OU
    │    ├─── Frontend Account
    │    ├─── Backend Account
    │    └─── Database Account
    │
    ├─── Non-Production OU
    │    │
    │    ├─── Development OU
    │    │    ├─── Dev Team A Account
    │    │    └─── Dev Team B Account
    │    │
    │    └─── Staging OU
    │         └─── Staging Account
    │
    └─── Security & Compliance OU
         ├─── Log Archive Account
         ├─── Security Tooling Account
         └─── Audit Account

Policy inheritance:
- Policy attached to "Production OU" applies to Frontend, Backend, AND Database accounts
- Policy attached to "Non-Production OU" applies to ALL Dev and Staging accounts
```

### Core Features

#### 1. Consolidated Billing

**What it is:**
All member accounts' charges roll up into a single bill paid by the management account.

**Benefits:**
- **Volume discounts:** AWS gives better pricing when you exceed usage thresholds. With consolidated billing, usage from ALL accounts counts toward thresholds.
- **Single payment:** One credit card charge instead of 50
- **Shared Reserved Instances:** If one account buys Reserved Instances (pre-paid compute capacity), unused capacity automatically benefits other accounts
- **Shared Savings Plans:** Same concept—cost savings shared across accounts

**Example:**

```
Without Consolidated Billing:
- Account A uses 800 GB data transfer = $0.09/GB = $72
- Account B uses 800 GB data transfer = $0.09/GB = $72
- Total: $144

With Consolidated Billing:
- Combined: 1,600 GB data transfer
- First 1 TB at $0.09/GB = $90
- Next 600 GB at $0.085/GB (volume discount) = $51
- Total: $141 (plus easier to manage)
```

#### 2. Centralized Account Management

**What it is:**
Create and invite accounts programmatically from the management account.

**Example CLI:**

```bash
# Create a new AWS account in your Organization
aws organizations create-account \
  --email marketing-aws@company.com \
  --account-name "Marketing Department" \
  --role-name OrganizationAccountAccessRole

# This creates:
# - New AWS account with unique Account ID
# - Automatically joined to your Organization
# - A role that allows management account to access it
# - No need to manually sign up or provide credit card

# Invite an existing external account to join
aws organizations invite-account-to-organization \
  --target Id=222222222222,Type=ACCOUNT

# Remove an account from Organization
aws organizations remove-account-from-organization \
  --account-id 333333333333
```

**Important notes:**
- Created accounts automatically get a role called `OrganizationAccountAccessRole` that allows management account admins to access them
- Each account still needs a unique email address
- Removing an account from Organization requires adding billing info to that account

#### 3. Service Control Policies (SCPs)

Service Control Policies are **guardrails** that limit what can be done in member accounts. They don't grant permissions—they restrict maximum available permissions. (Covered in detail in next section.)

#### 4. Shared Resources

**What it is:**
Share specific AWS resources across accounts without duplicating them.

**What can be shared:**
- VPC subnets (networking)
- Route 53 zones (DNS)
- License Manager licenses
- Resource Access Manager resources

**Why it matters:**

Without resource sharing:
- Each account creates its own VPC → complex networking
- Duplicate resources → higher costs
- Inconsistent configurations

With resource sharing:
- Shared Services account owns the VPC
- Other accounts use subnets from that VPC
- Centralized networking, lower costs

```
ASCII Diagram: Resource Sharing Example

Shared Services Account (111111111111)
    │
    └─── VPC (10.0.0.0/16)
         ├─── Subnet A (10.0.1.0/24) ─┐
         └─── Subnet B (10.0.2.0/24) ─┤
                                       │
                        RAM (Resource Access Manager)
                       Shares subnets to other accounts
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
            Production Account                    Development Account
            (222222222222)                       (333333333333)
                    │                                     │
            EC2 instances use                     EC2 instances use
            Subnet A (10.0.1.0/24)               Subnet B (10.0.2.0/24)

            Both accounts share the same VPC, no duplication needed
```

### Real-World Organization Example

**Company:** A fintech startup with 50 employees

**Organization structure:**

```
Management Account (finance@company.com)
    └─ Pays bills, rarely used for actual work

Production OU
    ├─ prod-web (customer-facing website)
    ├─ prod-api (backend services)
    └─ prod-data (databases and analytics)

Development OU
    ├─ dev-team-frontend
    ├─ dev-team-backend
    └─ sandbox (temporary testing)

Security OU
    ├─ security-logging (CloudTrail logs)
    ├─ security-monitoring (GuardDuty)
    └─ security-incident (forensics)

Shared Services OU
    └─ networking (Transit Gateway, VPN)
    └─ cicd (Jenkins, GitHub Actions runners)
```

**Policies applied:**

- **Production OU:** Deny deleting resources, require MFA, encrypt all data
- **Development OU:** Allow developers to experiment, cost limits
- **Security OU:** No one can modify logs, read-only access for auditors
- **All accounts:** Must use specific AWS regions, block unapproved services

### Benefits Summary

| Feature | Benefit |
|---------|---------|
| **Consolidated billing** | Volume discounts, single payment, shared savings |
| **Centralized management** | Create/invite/remove accounts from one place |
| **Policy enforcement** | Apply security rules to dozens of accounts at once |
| **Resource sharing** | Share VPCs, DNS zones without duplication |
| **Cost allocation** | Track spending by OU, team, or environment |
| **Audit & compliance** | Centralized logging, consistent security posture |

### Common Patterns

**By Environment:**
```
Root
├─── Production OU
├─── Staging OU
└─── Development OU
```

**By Department:**
```
Root
├─── Engineering OU
├─── Marketing OU
├─── Finance OU
└─── HR OU
```

**By Compliance Requirement:**
```
Root
├─── PCI-Compliant Workloads OU
├─── HIPAA-Compliant Workloads OU
└─── General Workloads OU
```

**Hybrid (Most Common):**
```
Root
├─── Production OU
│    ├─── Core Services
│    └─── Customer Data (HIPAA)
├─── Non-Production OU
│    ├─── Development
│    └─── Staging
├─── Security OU
└─── Shared Services OU
```

### Key Takeaways

✓ **AWS Organizations** = Free service to manage multiple AWS accounts centrally
✓ **Management account** pays consolidated bill and enforces policies
✓ **OUs (Organizational Units)** group accounts logically for policy application
✓ **Consolidated billing** provides volume discounts and shared savings
✓ Typically used with 10+ accounts, essential for large enterprises
✓ Enables **multi-account strategy** for security, compliance, and cost control

---

## Service Control Policies (SCPs)

### What They Are

Service Control Policies (SCPs) are **permission guardrails** that define the maximum permissions available in AWS accounts within an Organization. They act as a **security filter** that restricts what users and roles can do, even if they have full IAM admin permissions.

### Why They Exist

**Problem they solve:**

Imagine you have an AWS account admin with full permissions. Without SCPs:
- Admin could accidentally delete critical production resources
- Admin could launch expensive services in unapproved regions
- Admin could disable security logging
- Admin could use services that violate compliance requirements

Even with good intentions, humans make mistakes. SCPs provide **organizational guardrails** that prevent certain actions entirely.

### How They Work (Critical Concept)

**SCPs are a permission boundary, NOT a grant:**

```
Think of permissions as a Venn diagram:

    ┌─────────────────────────────────┐
    │  What SCP Allows (Max Boundary) │
    │   ┌─────────────────────┐       │
    │   │ What IAM Allows     │       │
    │   │                     │       │
    │   │   ┌──────────┐      │       │
    │   │   │ Actual   │      │       │
    │   │   │ Access   │      │       │
    │   │   └──────────┘      │       │
    │   └─────────────────────┘       │
    └─────────────────────────────────┘

For a user to perform an action, BOTH must allow it:
1. SCP must allow the action (organizational boundary)
2. IAM policy must allow the action (user/role permissions)

If EITHER denies, the action is blocked.
```

**Key principle:**
SCPs **restrict** the maximum available permissions. They don't grant permissions themselves. A user needs **both** SCP allowance AND IAM permissions to do anything.

### Example Scenario

**Setup:**
- Alice is an IAM admin in the Production account
- Her IAM policy grants `AdministratorAccess` (full permissions)

**Without SCP:**
Alice can do anything in AWS—create resources, delete databases, change region, disable logging.

**With SCP that denies deleting RDS databases:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "rds:Delete*",
      "Resource": "*"
    }
  ]
}
```

**Result:**
Alice still has admin permissions via IAM, BUT she physically cannot delete RDS databases. The SCP prevents it, regardless of her IAM permissions.

```
ASCII Diagram: SCP vs. IAM Permission Flow

User: Alice (IAM Admin in Production Account)
Tries to: Delete RDS database
        │
        ├─── Step 1: Check SCP
        │    │
        │    ├─ SCP attached to Production OU
        │    │  └─ Denies rds:Delete*
        │    │
        │    └─ Result: ❌ DENIED by SCP
        │
        └─── Step 2: Never reached
             (SCP denial blocks immediately,
              IAM permissions don't even matter)

User: Alice
Tries to: Create EC2 instance
        │
        ├─── Step 1: Check SCP
        │    └─ SCP allows ec2:RunInstances ✓
        │
        ├─── Step 2: Check IAM
        │    └─ IAM policy allows ec2:RunInstances ✓
        │
        └─── Result: ✅ ALLOWED (both permit it)
```

### SCP Types

AWS provides two approaches to SCPs:

#### 1. Allow List (Default, More Restrictive)

**How it works:**
By default, SCPs use an **implicit deny**—nothing is allowed unless explicitly permitted.

**Example: Allow only specific services**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",      // Allow all EC2 actions
        "s3:*",       // Allow all S3 actions
        "rds:*"       // Allow all RDS actions
      ],
      "Resource": "*"
    }
  ]
}
```

**Result:**
Only EC2, S3, and RDS can be used in this account. Services like Lambda, DynamoDB, EMR are completely blocked, even if users have IAM permissions for them.

**When to use:**
Highly regulated environments where you want tight control over which services can be used.

#### 2. Deny List (More Common, Less Restrictive)

**How it works:**
Start with full AWS access (`FullAWSAccess` managed SCP), then explicitly deny specific actions.

**Example: Block deleting CloudTrail logs**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

**Result:**
Everything in AWS works normally EXCEPT modifying CloudTrail (audit logs). Even account admins cannot tamper with logging.

**When to use:**
Most organizations use deny lists to block dangerous actions while keeping AWS fully functional.

### Common SCP Use Cases

#### Use Case 1: Enforce Region Restrictions

**Problem:** Company policy requires all resources in US regions for data sovereignty.

**Solution:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        }
      }
    }
  ]
}
```

**Effect:**
Users cannot create resources in Europe, Asia, or any non-US region. Attempts to do so are blocked.

#### Use Case 2: Prevent Disabling Security Services

**Problem:** Admins might accidentally disable GuardDuty (threat detection) or Config (compliance monitoring).

**Solution:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "config:DeleteConfigurationRecorder",
        "config:StopConfigurationRecorder"
      ],
      "Resource": "*"
    }
  ]
}
```

**Effect:**
Security services cannot be disabled, ensuring continuous monitoring.

#### Use Case 3: Block Root User Actions

**Problem:** Root user has unlimited permissions and should never be used daily.

**Solution:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

**Effect:**
Root user cannot perform ANY actions. Forces use of IAM users/roles.

#### Use Case 4: Require MFA for Sensitive Operations

**Problem:** Deleting resources should require multi-factor authentication.

**Solution:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "s3:DeleteBucket"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

**Effect:**
Users must have MFA active to delete EC2 instances, RDS databases, or S3 buckets.

### SCP Inheritance

SCPs are inherited down the OU tree:

```
ASCII Diagram: SCP Inheritance

Root (FullAWSAccess)
    │
    └─ SCP: Deny region outside us-east-1, us-west-2
    │
    ├─── Production OU
    │    │
    │    └─ SCP: Deny rds:Delete*, ec2:Terminate*
    │    │
    │    ├─── Prod-Web Account
    │    │    Effective policies:
    │    │    1. Deny non-US regions (from Root)
    │    │    2. Deny deleting RDS/EC2 (from Production OU)
    │    │
    │    └─── Prod-DB Account
    │         Effective policies:
    │         1. Deny non-US regions (from Root)
    │         2. Deny deleting RDS/EC2 (from Production OU)
    │
    └─── Development OU
         │
         └─ No additional SCP
         │
         └─── Dev Account
              Effective policies:
              1. Deny non-US regions (from Root only)
              2. CAN delete RDS/EC2 (no OU restriction)
```

**Key points:**
- SCPs are cumulative—accounts inherit all SCPs from parent OUs
- More restrictive policies win (if one SCP denies, it's denied)
- Management account is exempt from all SCPs (cannot restrict itself)

### SCP vs. IAM Policies

| Aspect | SCP | IAM Policy |
|--------|-----|------------|
| **Applied to** | Accounts or OUs | Users, groups, roles |
| **Purpose** | Set maximum boundary | Grant actual permissions |
| **Can grant** | No | Yes |
| **Can restrict** | Yes | Yes |
| **Scope** | Organization-wide | Single account |
| **Who manages** | Organization admin | Account admin |

### Important Limitations

**SCPs do NOT affect:**
- The management account (it's exempt)
- Service-linked roles (AWS-managed roles for services)
- Actions performed by AWS services on your behalf

**SCPs DO affect:**
- All IAM users and roles in member accounts
- Root user in member accounts
- Cross-account access from other accounts

### Testing SCPs Safely

**Best practice:** Test in a sandbox account first.

```bash
# Simulate what would happen if SCP is applied
# (Uses IAM Policy Simulator)

aws iam simulate-custom-policy \
  --policy-input-list file://test-scp.json \
  --action-names ec2:TerminateInstances \
  --resource-arns "arn:aws:ec2:us-east-1:123456789012:instance/*"

# Returns whether action would be allowed/denied
```

### Key Takeaways

✓ **SCPs** = Organization-wide guardrails that set maximum permissions
✓ **Do NOT grant permissions**—they only restrict what's possible
✓ Both SCP AND IAM policy must allow an action for it to succeed
✓ Applied to OUs or individual accounts, inherited down the tree
✓ Management account is exempt (cannot restrict itself)
✓ Common uses: Block regions, prevent deleting logs, require MFA, limit services
✓ Use **deny lists** (block specific actions) for most cases
✓ Always test SCPs in non-production accounts first

---

## AWS Control Tower

### What It Is

AWS Control Tower is an **automated setup and governance service** that creates and manages a secure, multi-account AWS environment based on AWS best practices. Think of it as a **pre-configured, production-ready AWS Organization** with built-in security guardrails and centralized dashboards.

Instead of manually creating accounts, configuring policies, and setting up logging for hours or days, Control Tower does it all in ~60 minutes.

### Why It Exists

**Problem it solves:**

Setting up a proper multi-account AWS environment requires:
- Creating dozens of accounts manually
- Configuring AWS Organizations and OUs
- Setting up centralized logging (CloudTrail, Config)
- Creating IAM roles and cross-account access
- Applying Service Control Policies
- Configuring security monitoring
- Establishing compliance tracking
- Creating account provisioning workflows

Doing this manually takes **weeks** and requires deep AWS expertise. Mistakes are common and costly.

**The solution:**
Control Tower automates the entire setup process and enforces ongoing governance. It's essentially **AWS's blueprint for a well-architected multi-account environment**.

### What Control Tower Provides

```
ASCII Diagram: Control Tower Components

                    AWS Control Tower
    ┌─────────────────────────────────────────────────┐
    │                                                 │
    │  1. Landing Zone (Base Environment)             │
    │     - Pre-configured OU structure               │
    │     - Centralized logging                       │
    │     - Cross-account networking                  │
    │     - Account factory for creating new accounts │
    │                                                 │
    │  2. Guardrails (Governance Rules)               │
    │     - Mandatory: Cannot be disabled             │
    │     - Strongly recommended: Best practices      │
    │     - Elective: Choose based on needs           │
    │                                                 │
    │  3. Account Factory (Self-Service Provisioning) │
    │     - Create new accounts via web form          │
    │     - Automatically applies baseline config     │
    │     - Pre-configured networking and IAM         │
    │                                                 │
    │  4. Dashboard (Centralized Visibility)          │
    │     - Compliance status across all accounts     │
    │     - Guardrail violations                      │
    │     - Drift detection                           │
    │                                                 │
    └─────────────────────────────────────────────────┘
```

### Core Concepts

#### 1. Landing Zone (Covered in detail in next section)

A Landing Zone is the **baseline multi-account environment** that Control Tower sets up. It includes:

**Pre-configured accounts:**
- **Management account:** Your existing AWS account (becomes the Control Tower home)
- **Log Archive account:** Stores all CloudTrail and Config logs (immutable, long-term retention)
- **Audit account:** Where security/compliance teams access read-only dashboards

**Pre-configured OUs:**
- **Security OU:** Contains Log Archive and Audit accounts
- **Sandbox OU:** For experimental workloads
- **Custom OUs:** You can create more (e.g., Production, Development)

```
ASCII Diagram: Control Tower Landing Zone Structure

Management Account (Control Tower Home)
    │
    ├─── AWS Organizations
    │    └─── Controls all accounts
    │
    └─── Control Tower Dashboard
         └─── Governance visibility

    ┌─────────────────────────────┐
    │  Security OU (Mandatory)    │
    │  ├─ Log Archive Account     │
    │  │  └─ All logs stored here │
    │  └─ Audit Account            │
    │     └─ Security dashboards   │
    └─────────────────────────────┘

    ┌─────────────────────────────┐
    │  Sandbox OU                 │
    │  └─ Experimentation accounts│
    └─────────────────────────────┘

    ┌─────────────────────────────┐
    │  Production OU (You create) │
    │  └─ Production workloads    │
    └─────────────────────────────┘
```

#### 2. Guardrails

Guardrails are **high-level rules** that enforce governance across all accounts. They're implemented using SCPs and AWS Config rules.

**Three types:**

**Mandatory (Cannot be disabled):**
- Disallow changing CloudTrail configuration
- Disallow deleting log archives
- Require MFA for root user
- Enable CloudTrail in all accounts

These exist to ensure baseline security and compliance. Even account admins cannot violate them.

**Strongly Recommended (Should enable):**
- Disallow public read access to S3 buckets
- Disallow changes to IAM roles created by Control Tower
- Enable encryption for EBS volumes
- Enable MFA for IAM users with console access

Best practices that AWS recommends for most organizations.

**Elective (Choose based on needs):**
- Disallow specific instance types (e.g., no GPU instances)
- Disallow launching resources in specific regions
- Require tagging for all resources
- Disallow internet gateways in certain accounts

Organization-specific rules you can enable as needed.

```
ASCII Diagram: Guardrail Enforcement Layers

Account: Production-Web (Member Account)
    │
    ├─ User tries: Delete CloudTrail logs
    │   └─> Blocked by Mandatory Guardrail (SCP)
    │       └─> "cloudtrail:DeleteTrail" is denied
    │
    ├─ User tries: Create S3 bucket with public access
    │   └─> Blocked by Strongly Recommended Guardrail (Config rule)
    │       └─> Config detects non-compliance → Alert sent
    │
    ├─ User tries: Launch t2.micro in us-east-1
    │   └─> Allowed (no guardrail restriction)
    │
    └─ User tries: Launch p3.8xlarge GPU instance
        └─> Blocked by Elective Guardrail (if enabled)
            └─> "ec2:RunInstances" with p3.* denied by SCP
```

**Guardrail status tracking:**

Control Tower dashboard shows:
- **Clear:** No violations, fully compliant
- **In violation:** Guardrail rule broken, needs remediation
- **Unknown:** Cannot determine status (rare)

#### 3. Account Factory

Account Factory is a **self-service portal** for creating new AWS accounts with pre-approved configurations.

**How it works:**

```
User Request (via AWS Service Catalog)
    │
    ├─ Fills web form:
    │   - Account name: "Marketing Production"
    │   - Email: marketing-prod@company.com
    │   - OU to join: Production OU
    │
    ├─ Submit request
    │
    ▼
Account Factory (Automated Process)
    │
    ├─ 1. Creates new AWS account
    ├─ 2. Joins it to specified OU
    ├─ 3. Applies all guardrails from that OU
    ├─ 4. Enables CloudTrail logging → Log Archive
    ├─ 5. Enables AWS Config → Audit account
    ├─ 6. Creates baseline IAM roles
    ├─ 7. Configures VPC (if specified)
    ├─ 8. Applies account-level SCPs
    │
    ▼
Ready-to-use account (15-30 minutes)
    └─ User can immediately deploy workloads
    └─ Already compliant with organization policies
```

**Benefits:**
- Consistent configuration across all accounts
- No manual setup errors
- Automatic compliance from day one
- Developers can request accounts without waiting for IT

**Example Account Factory request:**

```yaml
Account Name: Data Science Sandbox
Email: datascience-sandbox@company.com
OU: Sandbox
Managed Organizational Unit: Yes
VPC Configuration: Single VPC with public/private subnets
VPC CIDR: 10.1.0.0/16
Region: us-west-2
```

#### 4. Dashboard & Monitoring

Control Tower provides a **centralized dashboard** showing:

**Account Compliance:**
- Total accounts in organization
- Accounts in compliance vs. non-compliant
- Which guardrails are violated and where

**Drift Detection:**
- Has anyone manually changed Control Tower-managed resources?
- If yes, what changed and when?
- Ability to repair drift automatically

**Organizational View:**
```
Dashboard View (Simplified)

┌─────────────────────────────────────────────────┐
│ Control Tower Dashboard                         │
├─────────────────────────────────────────────────┤
│ Accounts:          45 total                     │
│ Compliant:         42 (93%)                     │
│ Non-Compliant:     3 (7%)                       │
│ Guardrails:        28 enabled                   │
│ Violations:        5 active                     │
├─────────────────────────────────────────────────┤
│ Recent Violations:                              │
│ ├─ dev-team-a: Public S3 bucket created        │
│ ├─ marketing: IAM user without MFA             │
│ └─ sandbox-2: Resource in eu-west-1 (disallowed)│
├─────────────────────────────────────────────────┤
│ Drift Detected:                                 │
│ └─ audit-account: CloudTrail modified manually  │
│    [Repair] [Ignore]                            │
└─────────────────────────────────────────────────┘
```

### When to Use Control Tower

**Use Control Tower if:**
- Starting fresh with AWS multi-account setup
- Have 10+ accounts (or planning to grow to that)
- Need consistent governance across accounts
- Want automated compliance monitoring
- Have limited AWS expertise (Control Tower embeds best practices)
- Need to meet compliance frameworks (SOC, PCI, HIPAA)

**Don't use Control Tower if:**
- You have a single AWS account with no plans to expand
- You already have complex custom Organizations setup (Control Tower requires specific structure)
- You need complete flexibility without guardrails (rare)

### Control Tower vs. Manual Organizations Setup

| Aspect | Control Tower | Manual Setup |
|--------|---------------|--------------|
| **Setup time** | ~60 minutes (mostly automated) | Days to weeks |
| **Expertise needed** | Basic AWS knowledge | Deep AWS expertise |
| **Governance** | Pre-configured guardrails | Build your own policies |
| **Compliance** | Built-in frameworks | Manual configuration |
| **Account provisioning** | Automated via Account Factory | Manual, error-prone |
| **Drift detection** | Automatic | Manual monitoring |
| **Cost** | Free (pay for underlying resources) | Free (pay for underlying resources) |
| **Flexibility** | Opinionated structure | Complete control |

### Real-World Setup Flow

**Step 1: Prerequisites**
- Have an AWS account (becomes management account)
- Not already using AWS Organizations (or willing to migrate)
- Choose home region (where Control Tower lives, usually us-east-1 or us-west-2)

**Step 2: Enable Control Tower**
```bash
# Via AWS Console (recommended for first time):
# 1. Navigate to AWS Control Tower service
# 2. Click "Set up landing zone"
# 3. Review and confirm OU structure
# 4. Provide Log Archive and Audit account email addresses
# 5. Wait ~60 minutes for setup
```

**Step 3: Post-Setup**
- Access Control Tower dashboard
- Enable additional guardrails based on requirements
- Create custom OUs (e.g., Production, Development)
- Provision new accounts via Account Factory
- Invite existing accounts to Control Tower (if applicable)

### Cost Considerations

**Control Tower itself is free**, but you pay for:
- AWS Config rules (~$2/rule/account/month)
- CloudTrail logs stored in S3 (~$0.20/GB)
- Service Catalog (Account Factory) usage (minimal)

**Typical costs:**
- Small org (5-10 accounts): ~$50-100/month
- Medium org (50 accounts): ~$500-800/month
- Large org (200+ accounts): ~$2,000-5,000/month

Most cost comes from AWS Config, which runs compliance checks continuously.

### Key Takeaways

✓ **Control Tower** = Automated multi-account setup + ongoing governance
✓ Creates **Landing Zone** with pre-configured accounts, OUs, and logging
✓ **Guardrails** enforce security and compliance (mandatory, recommended, elective)
✓ **Account Factory** enables self-service account provisioning
✓ **Dashboard** provides centralized visibility and drift detection
✓ Best for: Organizations with 10+ accounts needing governance
✓ Setup time: ~60 minutes vs. weeks for manual configuration
✓ Free service, but pay for underlying resources (Config, CloudTrail, S3)

---

## Landing Zones

### What It Is

A **Landing Zone** is a well-architected, multi-account AWS environment configured with **security, compliance, and governance best practices from day one**. It's the **foundation** upon which you build all your AWS workloads.

Think of it as the **blueprint for your AWS house**—before building rooms (applications), you need a solid foundation (Landing Zone) with electrical wiring (networking), plumbing (logging), and security systems (IAM/SCPs).

### Why It Exists

**Problem it solves:**

Organizations starting with AWS often make these mistakes:
- Deploy everything in one account → security nightmare
- Set up logging as an afterthought → no audit trail
- No standardized network architecture → connectivity chaos
- Inconsistent IAM configurations → security gaps
- No central governance → policy violations unnoticed

**The solution:**
A Landing Zone establishes **organizational standards** before you deploy any production workloads. It answers:
- How should accounts be structured?
- Where do logs go?
- What security policies apply everywhere?
- How do accounts communicate?
- How do we provision new accounts consistently?

### Landing Zone vs. Control Tower

**Important distinction:**

- **Landing Zone (concept):** The architectural pattern and set of best practices for multi-account AWS setup
- **AWS Control Tower (product):** AWS service that automatically creates and manages a Landing Zone

**You can have a Landing Zone without Control Tower** (build it manually), but Control Tower automates Landing Zone creation and maintenance.

### Core Components of a Landing Zone

Every well-designed Landing Zone includes these elements:

#### 1. Account Structure (Organizational Units)

**Purpose:** Logical grouping of accounts by function, environment, or compliance requirement.

```
ASCII Diagram: Typical Landing Zone Account Structure

Root (AWS Organization)
    │
    ├──── Security OU
    │     ├─ Log Archive Account
    │     │  └─ Centralized logging (immutable)
    │     │  └─ CloudTrail, Config, VPC Flow Logs
    │     │  └─ Retention: 7+ years
    │     │
    │     ├─ Audit/Compliance Account
    │     │  └─ Security monitoring (GuardDuty, Security Hub)
    │     │  └─ Compliance dashboards (AWS Config)
    │     │  └─ Read-only access for auditors
    │     │
    │     └─ Security Tooling Account
    │        └─ Vulnerability scanners
    │        └─ Threat detection tools
    │        └─ Incident response automation
    │
    ├──── Infrastructure/Shared Services OU
    │     ├─ Network Account
    │     │  └─ Transit Gateway (connects all VPCs)
    │     │  └─ Direct Connect / VPN
    │     │  └─ DNS (Route 53 Resolver)
    │     │
    │     └─ Shared Services Account
    │        └─ CI/CD pipelines (Jenkins, CodePipeline)
    │        └─ Artifact repositories (ECR, CodeArtifact)
    │        └─ Shared databases/caching
    │
    ├──── Production OU
    │     ├─ Prod-App-A Account
    │     ├─ Prod-App-B Account
    │     └─ Prod-Data Account (databases, analytics)
    │
    ├──── Staging/Pre-Production OU
    │     └─ Staging-App Account
    │
    ├──── Development OU
    │     ├─ Dev-Team-1 Account
    │     ├─ Dev-Team-2 Account
    │     └─ Dev-Shared Account
    │
    └──── Sandbox OU
          └─ Temporary experimentation accounts
          └─ Auto-deleted after 90 days
```

**Why this structure:**
- **Security accounts isolated:** Even if production is compromised, logs remain safe
- **Environment separation:** Dev mistakes can't break production
- **Shared services centralized:** Avoid duplicating CI/CD infrastructure
- **Clear billing:** See costs per team/environment/application

#### 2. Identity & Access Management (IAM) Baseline

**Purpose:** Standardized access patterns and role structure across all accounts.

**Key elements:**

**Cross-account roles:**
```
Management Account (Admins work here)
    │
    └─ IAM Role: OrganizationAccountAccessRole
       │
       ├─ Can assume role in ANY member account
       │  └─ Used by admins to access accounts
       │
       └─ Created automatically by Account Factory
```

**Break-glass access:**
Emergency access mechanism for account lockouts.

**Federated identity:**
Single Sign-On (SSO) integration with corporate identity provider (Okta, Azure AD, Google Workspace).

```
ASCII Diagram: Federated Access Flow

Corporate Identity Provider (Okta)
    │
    │ 1. User logs in: alice@company.com
    ├─> SAML assertion created
    │
    ▼
AWS SSO (Identity Center)
    │
    │ 2. Maps user to AWS accounts and roles
    ├─> alice@company.com → Production OU (ReadOnly)
    ├─> alice@company.com → Development OU (PowerUser)
    │
    ▼
AWS Accounts
    │
    └─ Alice can access:
       ├─ Production accounts (read-only)
       └─ Development accounts (full access)

No need to create separate IAM users in each account
```

#### 3. Centralized Logging

**Purpose:** All logs from all accounts flow to a single, immutable location for security and compliance.

**Log types collected:**

**CloudTrail (API audit logs):**
- Who did what, when, from where
- Every API call in every account
- Cannot be deleted by account admins

**AWS Config (Resource configuration history):**
- What resources exist in each account
- Configuration changes over time
- Compliance rule evaluations

**VPC Flow Logs (Network traffic):**
- IP traffic in and out of VPCs
- Security investigation and troubleshooting

**CloudWatch Logs (Application and service logs):**
- Lambda function logs
- ECS/EKS container logs
- Custom application logs

```
ASCII Diagram: Centralized Logging Architecture

┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Prod Account │   │ Dev Account  │   │ Staging Acct │
│              │   │              │   │              │
│ CloudTrail──┐│   │ CloudTrail──┐│   │ CloudTrail──┐│
│ Config─────┼┤   │ Config─────┼┤   │ Config─────┼┤
│ VPC Logs───┼┤   │ VPC Logs───┼┤   │ VPC Logs───┼┤
└────────────┼┘   └────────────┼┘   └────────────┼┘
             │                 │                 │
             └─────────────────┼─────────────────┘
                               │
                               ▼
                    ┌────────────────────┐
                    │  Log Archive       │
                    │  Account           │
                    │                    │
                    │  S3 Bucket         │
                    │  (Immutable)       │
                    │  - MFA delete      │
                    │  - Versioning      │
                    │  - Lifecycle policy│
                    │  - No write after  │
                    │    creation        │
                    └────────────────────┘
                               │
                               └─> Long-term storage
                                   └─> Glacier (7+ years)
```

**Why centralized:**
- If attacker compromises an account, they can't delete the logs (logs are in different account)
- Single source of truth for compliance audits
- Easier to search across all accounts
- Cost-efficient storage with lifecycle policies

#### 4. Network Architecture

**Purpose:** Standardized, secure, scalable network connectivity between accounts.

**Hub-and-spoke model:**

```
ASCII Diagram: Transit Gateway Network Architecture

              ┌────────────────────┐
              │  Network Account   │
              │                    │
              │  Transit Gateway   │
              │  (Central router)  │
              └────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│Production   │ │Development  │ │Shared       │
│VPC          │ │VPC          │ │Services VPC │
│             │ │             │ │             │
│10.0.0.0/16  │ │10.1.0.0/16  │ │10.2.0.0/16  │
└─────────────┘ └─────────────┘ └─────────────┘

Benefits:
- All VPCs connected through single point (Transit Gateway)
- No need for complex VPC peering mesh
- Centralized routing and firewall rules
- Easy to add/remove accounts
- Shared services accessible to all
```

**Internet egress:**
Centralized NAT Gateways in Network account for cost efficiency.

#### 5. Governance & Compliance

**Purpose:** Enforce security policies and track compliance automatically.

**Service Control Policies (SCPs):**
- Applied at OU level
- Define what can/cannot be done in accounts
- Examples: Block regions, prevent log deletion, require encryption

**AWS Config Rules:**
- Continuously check for compliance violations
- Examples: "All S3 buckets must be encrypted", "No public RDS instances"

**Detective controls:**
```
Automatic Checks (Every 15 minutes):
    │
    ├─> S3 bucket made public?
    │   └─> Alert to Security team via SNS
    │
    ├─> EC2 instance launched without encryption?
    │   └─> Non-compliant status in dashboard
    │
    └─> CloudTrail disabled?
        └─> Automatic remediation (re-enable)
        └─> Alert to security team
```

#### 6. Account Provisioning Process

**Purpose:** Consistent, repeatable way to create new accounts.

**Account Factory workflow:**

```
1. Request submitted (Service Catalog)
    │
    ├─ Account name
    ├─ Email address
    ├─ OU assignment
    └─ Network requirements
    │
    ▼
2. Automated provisioning (15-30 min)
    │
    ├─ Create AWS account
    ├─ Join to specified OU
    ├─ Enable CloudTrail → Log Archive
    ├─ Enable AWS Config → Audit account
    ├─ Apply OU-specific SCPs
    ├─ Create baseline IAM roles
    ├─ Configure VPC (if requested)
    ├─ Attach to Transit Gateway
    └─ Enable security services (GuardDuty, Security Hub)
    │
    ▼
3. Account ready for use
    └─> Already compliant with org policies
    └─> Already logged and monitored
    └─> Already networked with other accounts
```

### Building a Landing Zone (Manual vs. Automated)

**Option 1: AWS Control Tower (Recommended)**
- Automated setup in ~60 minutes
- Ongoing governance and drift detection
- Pre-configured best practices
- Good for: Most organizations

**Option 2: Manual build**
- Use Infrastructure as Code (Terraform, CloudFormation)
- Full customization
- More complex, requires expertise
- Good for: Large enterprises with specific requirements

**Option 3: AWS Landing Zone solution (Legacy)**
- Older CloudFormation-based solution
- Being replaced by Control Tower
- Not recommended for new deployments

### Landing Zone Maturity Model

```
Level 1: Basic
├─ Multiple accounts exist
├─ Basic AWS Organizations setup
├─ Some centralized billing
└─ Manual account creation

Level 2: Foundational
├─ Structured OUs (Security, Prod, Dev)
├─ CloudTrail enabled in all accounts
├─ Basic SCPs applied
└─ Some automation

Level 3: Operational (Typical Landing Zone)
├─ Control Tower or equivalent
├─ Centralized logging (CloudTrail, Config, VPC Flow)
├─ Account Factory for provisioning
├─ Network hub-and-spoke architecture
├─ Guardrails enforced via SCPs
└─ Compliance monitoring

Level 4: Advanced
├─ Automated compliance remediation
├─ Advanced network segmentation
├─ Multiple environments (regions/countries)
├─ Integration with SIEM tools
└─ Continuous security posture management

Level 5: Optimized
├─ Multi-region resilience
├─ Advanced threat detection and response
├─ Policy-as-code for all governance
├─ Self-healing infrastructure
└─ Zero-trust networking
```

### Key Takeaways

✓ **Landing Zone** = Multi-account foundation with built-in governance
✓ **Core components:** Account structure, IAM baseline, centralized logging, network architecture, compliance automation
✓ **Purpose:** Establish organizational standards BEFORE deploying workloads
✓ **Control Tower** automates Landing Zone creation and maintenance
✓ Typically includes: Security OU, Shared Services, Production, Development, Sandbox
✓ **Centralized logging** protects audit trails from compromise
✓ **Account Factory** enables consistent, automated account provisioning
✓ Investment upfront saves months of remediation work later

---

## AWS Well-Architected Framework

### What It Is

The AWS Well-Architected Framework is a **set of best practices and guiding principles** for designing and operating cloud systems. It's essentially **AWS's blueprint for building resilient, secure, efficient, and cost-effective architectures**.

Think of it as an **architectural review checklist** covering six key areas: security, reliability, performance, cost optimization, operational excellence, and sustainability.

### Why It Exists

**Problem it solves:**

Organizations building on AWS often ask:
- "Is my architecture secure enough?"
- "Will it scale when traffic spikes?"
- "Am I overpaying for resources?"
- "How do I handle failures gracefully?"
- "What happens during a disaster?"

Without standards, teams:
- Make inconsistent architectural decisions
- Miss critical security configurations
- Over-provision or under-provision resources
- Lack operational visibility
- Can't efficiently troubleshoot issues

**The solution:**
The Well-Architected Framework provides **proven patterns** and **architectural trade-offs** to help teams build better systems from the start and improve existing ones.

### The Six Pillars

```
ASCII Diagram: Well-Architected Framework Pillars

        AWS Well-Architected Framework
    ┌──────────────────────────────────────┐
    │                                      │
    │  ┌──────────────────────────┐        │
    │  │  1. Operational          │        │
    │  │     Excellence            │        │
    │  └──────────────────────────┘        │
    │                                      │
    │  ┌──────────────────────────┐        │
    │  │  2. Security             │        │
    │  └──────────────────────────┘        │
    │                                      │
    │  ┌──────────────────────────┐        │
    │  │  3. Reliability          │        │
    │  └──────────────────────────┘        │
    │                                      │
    │  ┌──────────────────────────┐        │
    │  │  4. Performance          │        │
    │  │     Efficiency            │        │
    │  └──────────────────────────┘        │
    │                                      │
    │  ┌──────────────────────────┐        │
    │  │  5. Cost                 │        │
    │  │     Optimization          │        │
    │  └──────────────────────────┘        │
    │                                      │
    │  ┌──────────────────────────┐        │
    │  │  6. Sustainability       │        │
    │  └──────────────────────────┘        │
    │                                      │
    └──────────────────────────────────────┘
```

### Pillar 1: Operational Excellence

**Focus:** Running and monitoring systems to deliver business value, and continually improving processes.

**Key principles:**
- **Perform operations as code:** Infrastructure, deployments, and operations should be automated (Infrastructure as Code)
- **Make frequent, small, reversible changes:** Deploy often, roll back quickly if issues occur
- **Refine operations procedures frequently:** Learn from failures, update runbooks
- **Anticipate failure:** Test failure scenarios (chaos engineering)
- **Learn from operational events:** Post-mortems after incidents

**Example questions:**
- How do you determine what your priorities are?
- How do you reduce defects and ease remediation?
- How do you know your operations are successful?

**Real-world application:**

```
Bad Architecture (Manual Operations):
- Deploy code by SSHing into servers
- Update configurations by hand
- No monitoring, discover issues from customer complaints
- Scared to make changes (might break production)

Good Architecture (Operational Excellence):
- CI/CD pipeline deploys automatically
- Infrastructure defined in Terraform/CloudFormation
- CloudWatch alarms notify team before users notice issues
- Blue/green deployments allow instant rollback
- Automated testing catches issues pre-production
```

### Pillar 2: Security

**Focus:** Protecting information, systems, and assets while delivering business value through risk assessments and mitigation strategies.

**Key principles:**
- **Implement a strong identity foundation:** Least privilege access, separation of duties, eliminate long-term credentials
- **Enable traceability:** Log and monitor all actions (CloudTrail, Config)
- **Apply security at all layers:** Defense in depth (network, host, application, data)
- **Automate security best practices:** Use services that enforce security automatically
- **Protect data in transit and at rest:** Encryption everywhere
- **Keep people away from data:** Minimize direct human access to sensitive data
- **Prepare for security events:** Incident response plans and automation

**Example questions:**
- How do you manage identities for people and machines?
- How do you detect and investigate security events?
- How do you protect your data at rest and in transit?

**Security layers:**

```
ASCII Diagram: Defense in Depth

External Layer
    ├─ AWS Shield (DDoS protection)
    ├─ AWS WAF (Web Application Firewall)
    └─ CloudFront (CDN with geo-blocking)
            │
            ▼
Network Layer
    ├─ VPC (Private network)
    ├─ Security Groups (Instance firewalls)
    ├─ Network ACLs (Subnet firewalls)
    └─ PrivateLink (No internet exposure)
            │
            ▼
Compute Layer
    ├─ IAM roles (No hardcoded credentials)
    ├─ Systems Manager (Patching)
    ├─ Inspector (Vulnerability scanning)
    └─ GuardDuty (Threat detection)
            │
            ▼
Application Layer
    ├─ Code scanning (SonarQube)
    ├─ Dependency checking (Snyk)
    └─ Secrets Manager (No passwords in code)
            │
            ▼
Data Layer
    ├─ Encryption at rest (KMS)
    ├─ Encryption in transit (TLS)
    ├─ S3 bucket policies
    └─ RDS encryption

If one layer fails, others still protect you
```

### Pillar 3: Reliability

**Focus:** Systems must perform their intended function consistently and correctly, recover from failures, and meet demand.

**Key principles:**
- **Automatically recover from failure:** Monitor KPIs, trigger automation when thresholds are breached
- **Test recovery procedures:** Regularly test failover and disaster recovery
- **Scale horizontally:** Distribute load across multiple smaller resources
- **Stop guessing capacity:** Use auto-scaling
- **Manage change through automation:** Infrastructure as Code reduces human error

**Example questions:**
- How do you monitor workload resources?
- How do you implement change?
- How do you back up data?
- How do you plan for disaster recovery?

**Reliability patterns:**

```
Single AZ (Not Reliable):
┌──────────────────┐
│   Availability   │
│   Zone A         │
│                  │
│  Load Balancer   │
│      ↓           │
│  Web Server 1    │
│      ↓           │
│  Database        │
└──────────────────┘

If AZ-A fails → Entire system down

Multi-AZ (Reliable):
┌──────────────────┐  ┌──────────────────┐
│   Availability   │  │   Availability   │
│   Zone A         │  │   Zone B         │
│                  │  │                  │
│  Web Server 1 ←──┼──┼→ Web Server 2    │
│      ↓           │  │      ↓           │
│  DB Primary ─────┼──┼→ DB Standby      │
└──────────────────┘  └──────────────────┘
       ↑                       ↑
       └─── Multi-AZ LB ───────┘

If AZ-A fails → Traffic routes to AZ-B automatically
Database automatically fails over to standby
```

**RTO vs. RPO:**

```
Recovery Time Objective (RTO):
- How long can you afford to be down?
- Example: "We can tolerate 1 hour downtime"

Recovery Point Objective (RPO):
- How much data loss is acceptable?
- Example: "We can lose max 5 minutes of data"

┌───────────────────────────────────────────────┐
│ Timeline of Disaster:                         │
│                                               │
│ Normal Operations ──┬── Disaster ──┬── Recovered│
│                     │                │         │
│                  [Disaster]      [System Up]  │
│                     ├────RPO────┤             │
│                     ├────────RTO────────┤     │
│                                               │
│ RPO: Last successful backup                   │
│ RTO: Time to restore and resume               │
└───────────────────────────────────────────────┘
```

### Pillar 4: Performance Efficiency

**Focus:** Using computing resources efficiently to meet requirements and maintain efficiency as demand changes.

**Key principles:**
- **Democratize advanced technologies:** Use managed services instead of building from scratch
- **Go global in minutes:** Deploy in multiple regions easily
- **Use serverless architectures:** No server management overhead
- **Experiment more often:** Test different instance types, services
- **Consider mechanical sympathy:** Understand how technologies work underneath

**Example questions:**
- How do you select appropriate resource types and sizes?
- How do you monitor performance?
- How do you evolve your workload to take advantage of new releases?

**Performance optimization example:**

```
Problem: Web application is slow during peak hours

Analysis:
┌─────────────────────────────────────────┐
│ Bottleneck Identification               │
├─────────────────────────────────────────┤
│ Database queries: 2000ms (SLOW)         │
│ Application logic: 50ms (OK)            │
│ External API calls: 500ms (OK)          │
│ Frontend rendering: 100ms (OK)          │
└─────────────────────────────────────────┘

Solutions (Performance Efficiency):
1. Add ElastiCache (Redis) for frequently accessed data
   - Database queries reduced to 10ms for cached data

2. Use Aurora Serverless for auto-scaling database
   - Database scales up during peak automatically

3. Use CloudFront CDN for static assets
   - Static content served from edge (10ms vs 200ms)

4. Implement database read replicas
   - Distribute read traffic across multiple databases

Result: Page load time 2650ms → 660ms (75% improvement)
```

### Pillar 5: Cost Optimization

**Focus:** Running systems to deliver business value at the lowest price point.

**Key principles:**
- **Implement cloud financial management:** Dedicated team to manage cloud costs
- **Adopt a consumption model:** Pay only for what you use
- **Measure overall efficiency:** Understand cost per business outcome
- **Stop spending money on undifferentiated heavy lifting:** Use managed services
- **Analyze and attribute expenditure:** Tag resources, track costs by team/project

**Example questions:**
- How do you govern usage?
- How do you monitor usage and cost?
- How do you decommission resources?

**Cost optimization strategies:**

```
Right-Sizing:
┌────────────────────────────────────────┐
│ Current: t3.2xlarge (8 vCPU, 32GB RAM)│
│ Actual usage: 10% CPU, 20% memory     │
│ Cost: $300/month                      │
└────────────────────────────────────────┘
                ↓ Right-size
┌────────────────────────────────────────┐
│ New: t3.medium (2 vCPU, 4GB RAM)      │
│ Usage: 40% CPU, 80% memory (optimal)  │
│ Cost: $30/month                       │
│ Savings: $270/month (90%)             │
└────────────────────────────────────────┘

Reserved Instances / Savings Plans:
On-Demand: $0.416/hour = $300/month
Reserved (1 year): $0.275/hour = $198/month (34% savings)
Reserved (3 years): $0.185/hour = $133/month (56% savings)

Spot Instances (for non-critical workloads):
On-Demand: $300/month
Spot: $90/month (70% savings, but can be interrupted)

Auto-Scaling:
Fixed 10 instances 24/7: $3,000/month
Auto-scale 2-15 instances: $600/month (80% savings)
```

### Pillar 6: Sustainability

**Focus:** Minimizing environmental impact of cloud workloads (added in 2021).

**Key principles:**
- **Understand your impact:** Measure carbon footprint
- **Establish sustainability goals:** Set targets for improvement
- **Maximize utilization:** Higher utilization = better efficiency
- **Anticipate and adopt more efficient hardware:** Use latest instance types
- **Use managed services:** AWS optimizes underlying infrastructure
- **Reduce downstream impact:** Efficient data transfer and storage

**Example questions:**
- How do you select Regions to support sustainability goals?
- How do you take advantage of software and architecture patterns?

**Sustainability practices:**

```
Energy-Efficient Patterns:

1. Use Graviton processors (ARM-based)
   - 60% better performance per watt than x86
   - Lower carbon footprint

2. Right-size resources
   - Idle capacity wastes energy
   - Auto-scale to match demand

3. Choose efficient storage
   - S3 Intelligent-Tiering (auto-moves old data to cheaper, more efficient tiers)
   - Archive to Glacier (uses less energy)

4. Optimize data transfer
   - Cache at edge locations (CloudFront)
   - Compress data before transfer
   - Avoid unnecessary data movement

5. Use serverless where possible
   - Lambda only runs when needed (no idle servers)
   - Scales to zero when not in use
```

### Well-Architected Review Process

**How it works:**

```
1. Workload Definition
   ├─ What are you building?
   ├─ What are the business requirements?
   └─ What are the compliance requirements?

2. Review Using Framework
   ├─ Answer questions for each pillar
   ├─ Identify high and medium risk issues
   └─ Document current architecture

3. Improvement Plan
   ├─ Prioritize risks
   ├─ Create remediation roadmap
   └─ Estimate effort and impact

4. Implementation
   ├─ Address high-risk items first
   ├─ Validate improvements
   └─ Document changes

5. Repeat Regularly
   └─ Re-review every 6-12 months
   └─ Review before major changes
```

**AWS provides:**
- **Well-Architected Tool:** Free service in AWS Console for self-service reviews
- **Well-Architected Reviews:** AWS solutions architects can perform reviews
- **Lenses:** Specialized guidance for specific workloads (SaaS, Machine Learning, IoT, etc.)

### Trade-offs Between Pillars

Often improving one pillar impacts another:

```
Example Trade-off Scenarios:

Reliability vs. Cost:
├─ High reliability: Multi-region, always-on standby = $$$$
└─ Balanced: Multi-AZ in single region = $$

Performance vs. Cost:
├─ Maximum performance: Largest instances, global CDN = $$$$
└─ Balanced: Right-sized instances, regional CDN = $$

Security vs. Performance:
├─ Maximum security: Every request scanned, encrypted = Slower
└─ Balanced: Encryption + selective scanning = Faster

Operational Excellence vs. Speed to Market:
├─ Perfect automation: Months to build = Slow launch
└─ Balanced: Iterative automation = Launch fast, improve over time
```

**The framework helps you make informed trade-offs** based on business priorities.

### Key Takeaways

✓ **Well-Architected Framework** = Best practices across six pillars
✓ **Six pillars:** Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability
✓ **Purpose:** Build better systems, identify risks, drive continuous improvement
✓ **Trade-offs are normal:** Framework helps you make informed decisions
✓ **Free tool available:** AWS Well-Architected Tool for self-service reviews
✓ **Review regularly:** Perform reviews at project start and every 6-12 months
✓ Not a checklist to complete once—it's an ongoing practice

---

## Shared Responsibility Model

### What It Is

The AWS Shared Responsibility Model defines **who is responsible for security of what** in the cloud. It's a clear division of security and compliance duties between AWS (the cloud provider) and you (the customer).

Think of it like renting an apartment:
- **Landlord (AWS):** Responsible for building security, electricity, plumbing
- **Tenant (You):** Responsible for locking your door, not letting strangers in, protecting your belongings

### Why It Exists

**Problem it solves:**

When moving to the cloud, organizations often have confusion:
- "Who patches the underlying servers?"
- "Who encrypts my data?"
- "Who controls network access?"
- "If there's a breach, who is liable?"

Without clear boundaries, critical security controls might be **nobody's responsibility** (each side assumes the other is handling it).

**The solution:**
The Shared Responsibility Model clearly defines: **AWS secures the cloud infrastructure, you secure what you put in the cloud**.

### The Core Principle

```
ASCII Diagram: Shared Responsibility Model Overview

YOU (Customer)                    AWS (Amazon)
Responsibility "IN" the Cloud     Responsibility "OF" the Cloud

┌───────────────────────┐         ┌──────────────────────────┐
│                       │         │                          │
│  Your Data            │         │  Physical Security       │
│  Your Applications    │         │  - Data centers          │
│  Your OS Config       │         │  - Power/cooling         │
│  Your Network Config  │         │  - Physical access       │
│  Your Access Controls │         │                          │
│  Your Encryption Keys │         │  Hardware                │
│                       │         │  - Servers               │
│                       │         │  - Storage devices       │
│                       │         │  - Network equipment     │
│                       │         │                          │
│                       │         │  Global Infrastructure   │
│                       │         │  - Regions               │
│                       │         │  - Availability Zones    │
│                       │         │  - Edge Locations        │
│                       │         │                          │
│                       │         │  Software Platform       │
│                       │         │  - Hypervisor            │
│                       │         │  - Managed services      │
│                       │         │  - API infrastructure    │
│                       │         │                          │
└───────────────────────┘         └──────────────────────────┘

Security IN the Cloud               Security OF the Cloud
(You configure and manage)          (AWS builds and maintains)
```

### AWS's Responsibilities ("OF" the Cloud)

**What AWS manages and secures:**

**1. Physical Infrastructure:**
- Data center facilities (24/7 guards, biometric access, cameras)
- Fire suppression systems
- Climate control (cooling, power backup)
- Physical server hardware

**2. Network Infrastructure:**
- Routers, switches, load balancers (physical equipment)
- DDoS protection (AWS Shield) at infrastructure level
- Network backbone connecting Regions

**3. Virtualization Layer:**
- Hypervisor (software separating customer virtual machines)
- Ensuring customers cannot access each other's resources

**4. Managed Services Infrastructure:**
- For services like RDS, Lambda, S3—AWS manages underlying OS, patching, backups

**AWS certifications:**
- SOC 1/2/3
- ISO 27001
- PCI DSS Level 1
- FedRAMP
- Many more (varies by service and Region)

**What this means:**
You don't need to worry about:
- Earthquakes destroying data (AWS has redundancy)
- Someone physically stealing a server
- Network equipment failing
- Hypervisor vulnerabilities

### Your Responsibilities ("IN" the Cloud)

**What you must configure and secure:**

**1. Data:**
- **Classification:** Mark data as public, internal, confidential
- **Encryption:** Choose to encrypt data at rest and in transit (YOU enable it)
- **Backup:** Configure backup schedules and test restores
- **Retention:** Define how long data is kept

**2. Platform and Applications:**
- **Operating System:** Patch and update EC2 instances
- **Application code:** Secure your code (no hardcoded passwords, input validation)
- **Dependencies:** Keep libraries and frameworks updated

**3. Identity & Access Management:**
- **IAM policies:** Who can access what
- **MFA:** Enable multi-factor authentication
- **Password policies:** Enforce strong passwords
- **Key management:** Rotate credentials regularly

**4. Network Configuration:**
- **Security Groups:** Configure firewall rules correctly
- **Network ACLs:** Subnet-level access controls
- **VPC design:** Private subnets for sensitive resources
- **Public access:** Don't accidentally expose databases to internet

**5. Client-Side Encryption:**
- Encrypt data before sending to AWS (if required)
- Manage encryption keys yourself (or use AWS KMS)

```
Bad Configuration (Your Responsibility):
- S3 bucket set to public (anyone can read your files)
- EC2 instance with Security Group allowing 0.0.0.0/0 on SSH port
- RDS database with weak password "admin123"
- No MFA on root account

AWS does NOT automatically prevent these—you must configure correctly
```

### Responsibility Varies by Service Type

The model shifts based on service abstraction level:

```
ASCII Diagram: Responsibility by Service Type

IaaS (Infrastructure)     PaaS (Platform)         SaaS (Software)
Example: EC2             Example: RDS            Example: S3
┌───────────────────┐    ┌───────────────────┐   ┌───────────────────┐
│ YOU                │    │ YOU                │   │ YOU                │
│ ├─ Data           │    │ ├─ Data           │   │ ├─ Data           │
│ ├─ Application    │    │ ├─ Access Control │   │ └─ Access Control │
│ ├─ Runtime        │    │ │                  │   │                   │
│ ├─ OS             │    │ AWS                │   │ AWS                │
│ │                 │    │ ├─ OS              │   │ ├─ Application    │
│ AWS               │    │ ├─ Runtime         │   │ ├─ Runtime        │
│ ├─ Virtualization │    │ ├─ Virtualization  │   │ ├─ OS             │
│ ├─ Hardware       │    │ ├─ Hardware        │   │ ├─ Virtualization │
│ └─ Infrastructure │    │ └─ Infrastructure  │   │ ├─ Hardware       │
└───────────────────┘    └───────────────────┘   │ └─ Infrastructure │
                                                  └───────────────────┘
More responsibility ←                         → Less responsibility
More control ←                                → Less control
```

**Examples:**

**EC2 (Infrastructure as a Service):**
- AWS: Hardware, hypervisor, physical network
- You: OS patches, firewall rules, data encryption, applications

**RDS (Platform as a Service):**
- AWS: Hardware, OS, database software, patches, backups
- You: Database configuration, access controls, encryption settings, query optimization

**S3 (Software as a Service):**
- AWS: Entire infrastructure, durability (11 9's), availability
- You: Bucket policies, encryption settings, versioning, lifecycle rules

### Common Security Misconfigurations (Your Responsibility to Prevent)

**1. Public S3 Buckets**
```
Problem: S3 bucket containing customer data set to public
Impact: Data breach, regulatory fines, reputation damage
AWS Responsibility: Provide tools to prevent this (Block Public Access)
Your Responsibility: Actually enable Block Public Access
```

**2. Overly Permissive Security Groups**
```
Problem: Security Group allows SSH from 0.0.0.0/0 (entire internet)
Impact: Brute force attacks, unauthorized access
AWS Responsibility: Provide Security Groups feature
Your Responsibility: Configure rules properly (allow only your office IP)
```

**3. Unencrypted Data**
```
Problem: S3 buckets, EBS volumes, RDS databases without encryption
Impact: Data exposure if physical media is accessed
AWS Responsibility: Provide encryption options (KMS, default encryption)
Your Responsibility: Enable encryption (it's often opt-in, not default)
```

**4. No MFA on Root Account**
```
Problem: Root account has unlimited permissions, no MFA
Impact: Account takeover if password is compromised
AWS Responsibility: Provide MFA capability
Your Responsibility: Enable MFA (AWS cannot force you to do this)
```

**5. Outdated Operating Systems**
```
Problem: Running EC2 with unpatched OS vulnerable to exploits
Impact: Malware, ransomware, data theft
AWS Responsibility: Provide patching tools (Systems Manager)
Your Responsibility: Apply patches (AWS doesn't auto-patch your OS)
```

### Inherited vs. Shared vs. Customer-Only Controls

**Inherited Controls:**
- Fully managed by AWS
- You inherit the benefit
- Examples: Physical security, environmental controls

**Shared Controls:**
- AWS provides the capability, you configure it
- Examples: Patching (AWS patches infrastructure, you patch your OS), Configuration management

**Customer-Only Controls:**
- You are solely responsible
- Examples: Data classification, IAM policies, application security

```
Control Responsibility Matrix:

Control Type          │ AWS │ Customer │ Example
──────────────────────┼─────┼──────────┼────────────────────────
Inherited             │ ✓   │          │ Physical data center security
Shared - Patching     │ ✓   │    ✓     │ AWS patches hypervisor / You patch OS
Shared - Config Mgmt  │ ✓   │    ✓     │ AWS provides tools / You configure
Customer-Only         │     │    ✓     │ IAM policies, data encryption
```

### Real-World Breach Scenario

**Capital One Breach (2019) - Shared Responsibility in Action:**

**What happened:**
- Application on EC2 had Server-Side Request Forgery (SSRF) vulnerability
- Attacker exploited vulnerability to access EC2 metadata service
- Metadata service contained IAM role credentials
- Credentials had excessive permissions to S3 buckets
- 100 million customer records stolen

**Who was responsible:**

| Aspect | Responsible Party | Why |
|--------|------------------|-----|
| EC2 infrastructure security | AWS | EC2 service was not compromised |
| Application code vulnerability | Customer | Customer wrote vulnerable code |
| IAM role with excessive permissions | Customer | Customer configured overly permissive policies |
| EC2 metadata service behavior | AWS | Working as designed |
| Firewall rules (Security Groups) | Customer | Should have been more restrictive |
| Data encryption | Shared | AWS provides tools, customer enables |

**Outcome:** Customer (Capital One) was liable because breach was due to **customer's security misconfigurations**, not AWS infrastructure failure.

### Key Takeaways

✓ **AWS** secures the cloud infrastructure (physical, network, hardware)
✓ **You** secure what you put in the cloud (data, apps, access controls)
✓ Responsibility shifts based on service type (EC2 vs RDS vs S3)
✓ **Most breaches are due to customer misconfiguration**, not AWS failures
✓ Enabling security features is YOUR responsibility (encryption, MFA, backups)
✓ AWS provides the tools—you must use them correctly
✓ Both parties share responsibility for overall security posture

---

## AWS Support Plans

### What They Are

AWS Support Plans are **subscription tiers** that provide different levels of technical support, architectural guidance, and response times from AWS. Think of them as **insurance policies** for your cloud infrastructure—you pay for access to AWS experts when things go wrong or when you need guidance.

### Why They Exist

**Problem they solve:**

When running production systems on AWS:
- Critical services go down (need 24/7 emergency support)
- Need expert guidance on architecture design
- Struggling with performance or cost optimization
- Regulatory compliance requirements need validation
- Complex multi-service integrations aren't working

Without support, you're on your own—reading documentation, forum posts, hoping to find answers before business impact escalates.

**The solution:**
Support plans provide direct access to AWS engineers, architectural reviews, faster response times, and proactive guidance based on your criticality needs and budget.

### The Five Support Tiers

```
ASCII Diagram: AWS Support Plans

┌──────────────┬─────────────┬──────────────┬─────────────┬──────────────┐
│ Basic        │ Developer   │ Business     │ Enterprise  │ Enterprise   │
│ (Free)       │             │              │ On-Ramp     │ Support      │
├──────────────┼─────────────┼──────────────┼─────────────┼──────────────┤
│ $0/month     │ $29/month   │ $100/month   │ $5,500/month│ $15,000/month│
│              │ or 3% bill  │ or 10% bill  │ (minimum)   │ (minimum)    │
├──────────────┼─────────────┼──────────────┼─────────────┼──────────────┤
│ Forum only   │ Business    │ 24/7 phone   │ 24/7 phone  │ 24/7 phone   │
│              │ hours email │ email, chat  │ email, chat │ email, chat  │
├──────────────┼─────────────┼──────────────┼─────────────┼──────────────┤
│ No response  │ <24 hours   │ <1 hour      │ <30 min     │ <15 min      │
│ SLA          │ (general)   │ (critical)   │ (critical)  │ (critical)   │
└──────────────┴─────────────┴──────────────┴─────────────┴──────────────┘
         Increasing Cost →
         Increasing Support →
```

### Plan Details

#### 1. Basic Support (Free)

**Included with all AWS accounts:**

**What you get:**
- **Customer Service:** Billing and account questions only
- **Documentation:** Access to all whitepapers, documentation, support forums
- **AWS Trusted Advisor:** 7 core checks (basic security and performance recommendations)
- **AWS Health Dashboard:** Service health notifications

**What you DON'T get:**
- No technical support cases
- No architectural guidance
- No direct access to AWS engineers

**Response times:** N/A (no technical support)

**Who it's for:**
- Personal projects and learning
- Non-production environments
- When you have internal AWS expertise

```
Basic Support Use Case:
├─ Scenario: Learning AWS, building side projects
├─ Bill: $50/month
├─ Support cost: $0
└─ If issue occurs: Google it, ask in forums
```

#### 2. Developer Support

**Cost:**
- Greater of $29/month or 3% of monthly AWS bill
- Example: $1,000 AWS bill = $30 support (3%)

**What you get:**
- **Business hours email access** to Cloud Support Associates
- **General guidance:** < 24 hours response
- **System impaired:** < 12 hours response
- **Trusted Advisor:** Same 7 core checks as Basic
- **Best practices guidance:** General architectural advice
- **1 primary contact** allowed to open cases

**Limitations:**
- No phone support
- Business hours only (not 24/7)
- Cases handled by junior support associates, not senior engineers

**Who it's for:**
- Startups and small businesses
- Development/testing environments
- Non-critical applications
- Single developer or small team

```
Developer Support Use Case:
├─ Scenario: Startup with dev environment
├─ Bill: $500/month
├─ Support cost: $29/month
├─ Issue: Lambda function timing out
└─ Response: Email within 12-24 hours with troubleshooting steps
```

#### 3. Business Support

**Cost:**
- Greater of $100/month or:
  - 10% of first $0-$10K
  - 7% of $10K-$80K
  - 5% of $80K-$250K
  - 3% over $250K
- Example: $10,000 AWS bill = $1,000 support (10%)

**What you get:**
- **24/7 phone, email, and chat** access to Cloud Support Engineers
- **Production system impaired:** < 4 hours response
- **Production system down:** < 1 hour response
- **Full Trusted Advisor:** All checks (cost optimization, security, fault tolerance)
- **Infrastructure Event Management:** AWS support for launches/events (additional fee)
- **Unlimited contacts** can open cases
- **Contextual architectural guidance:** Best practices for your use case
- **Interoperability support:** Third-party software integrations

**Response time SLAs:**

| Severity | Description | Response Time |
|----------|-------------|---------------|
| General guidance | How-to questions | < 24 hours |
| System impaired | Non-critical functions affected | < 12 hours |
| Production system impaired | Important functions degraded | < 4 hours |
| Production system down | Business significantly impacted | < 1 hour |

**Who it's for:**
- Production workloads
- Businesses with revenue-generating applications on AWS
- Teams needing 24/7 support
- Companies without deep AWS expertise

```
Business Support Use Case:
├─ Scenario: E-commerce site processing $100K/day
├─ Bill: $15,000/month
├─ Support cost: $1,500/month (10%)
├─ Issue: Website down, can't process orders
├─ Action: Call AWS support hotline
└─ Response: Engineer on phone within 1 hour
```

#### 4. Enterprise On-Ramp

**Cost:**
- Greater of $5,500/month or:
  - 10% of first $0-$150K
  - 7% of $150K-$500K
  - 5% of $500K-$1M
  - 3% over $1M

**What you get (in addition to Business):**
- **Business-critical system down:** < 30 minutes response
- **Pool of Technical Account Managers (TAMs):** Shared TAM for guidance
- **Access to proactive programs:**
  - Well-Architected reviews
  - Operations reviews
  - Cost optimization workshops
- **Incident and Event Management:** Support for planned events (product launches)
- **Access to AWS Incident Detection and Response** (additional cost)

**Who it's for:**
- Growing companies with critical workloads
- Organizations spending $50K-$150K+/month
- Teams wanting proactive guidance without full dedicated TAM cost

```
Enterprise On-Ramp Use Case:
├─ Scenario: SaaS company with $100K AWS bill
├─ Support cost: $10,000/month
├─ Benefit: Quarterly Well-Architected review
├─ Benefit: Pre-launch support for major feature release
└─ Benefit: 30-minute response for critical outages
```

#### 5. Enterprise Support

**Cost:**
- Greater of $15,000/month or:
  - 10% of first $0-$150K
  - 7% of $150K-$500K
  - 5% of $500K-$1M
  - 3% over $1M

**What you get (in addition to Enterprise On-Ramp):**
- **Business-critical system down:** < 15 minutes response
- **Dedicated Technical Account Manager (TAM):**
  - Single point of contact
  - Proactive reviews and guidance
  - Coordinates with AWS engineering teams
  - Escalation management
- **Concierge support team:** Billing and account assistance
- **Infrastructure Event Management:** Included (not additional fee)
- **Well-Architected reviews:** Unlimited
- **Operations reviews:** Unlimited
- **Training credits:** Included
- **Access to AWS Managed Services:** Option to have AWS operate your infrastructure

**Technical Account Manager (TAM) value:**

```
TAM Activities (Proactive):
├─ Monthly check-ins on architecture
├─ Quarterly Well-Architected reviews
├─ Cost optimization recommendations
├─ Access to AWS roadmap information (NDA)
├─ Review security posture
├─ Pre-launch support for major deployments
└─ Direct escalation path to AWS service teams

TAM During Incidents:
├─ Coordinates response across multiple AWS teams
├─ Stays on call until resolution
├─ Post-incident analysis and recommendations
└─ Prevents future similar incidents
```

**Who it's for:**
- Large enterprises with mission-critical workloads
- Organizations spending $150K+/month on AWS
- Companies needing strategic AWS partnership
- Businesses where downtime costs exceed $10K/hour

```
Enterprise Support Use Case:
├─ Scenario: Financial services company, $500K AWS bill
├─ Support cost: $50,000/month
├─ Benefit: Dedicated TAM named Sarah
├─ Sarah's work:
│   ├─ Monthly architecture review
│   ├─ Pre-approved access to AWS engineering teams
│   ├─ Saved $200K/year via cost optimization recommendations
│   └─ Coordinated resolution during critical database outage (15 min)
└─ ROI: Support pays for itself through cost savings alone
```

### Support Plan Feature Comparison

```
Feature Matrix:

Feature                     │ Basic │ Developer │ Business │ Ent. On-Ramp │ Enterprise
────────────────────────────┼───────┼───────────┼──────────┼──────────────┼────────────
24/7 Support                │   ✗   │     ✗     │    ✓     │      ✓       │     ✓
Phone Support               │   ✗   │     ✗     │    ✓     │      ✓       │     ✓
<1 hour response (critical) │   ✗   │     ✗     │    ✓     │      ✓       │     ✓
<30 min response (critical) │   ✗   │     ✗     │    ✗     │      ✓       │     ✓
<15 min response (critical) │   ✗   │     ✗     │    ✗     │      ✗       │     ✓
Full Trusted Advisor        │   ✗   │     ✗     │    ✓     │      ✓       │     ✓
Dedicated TAM               │   ✗   │     ✗     │    ✗     │      ✗       │     ✓
Shared TAM Pool             │   ✗   │     ✗     │    ✗     │      ✓       │     ✓
Well-Architected Reviews    │   ✗   │     ✗     │  Paid    │      ✓       │     ✓
Cost Optimization           │ Basic │   Basic   │   Full   │     Full     │    Full
Unlimited Contacts          │   ✗   │     ✗     │    ✓     │      ✓       │     ✓
```

### How to Choose

**Decision flowchart:**

```
Are you in production? ──── No ──── Learning/Testing? ──── Yes ──── Basic (Free)
         │                                   │
         │                                   No
         │                                   │
        Yes                                  ▼
         │                           Single developer? ──── Yes ──── Developer ($29)
         │                                   │
         ▼                                   No
Bill > $10K/month? ──── No ──── Developer         │
         │                                         ▼
        Yes                                  Business ($100+)
         │
         ▼
Bill > $150K/month? ──── No ──── Business Support
         │
        Yes
         │
         ▼
Need dedicated TAM? ──── No ──── Enterprise On-Ramp ($5.5K)
         │
        Yes
         │
         ▼
    Enterprise Support ($15K+)
```

### Cost Examples

**Scenario 1: Startup**
- AWS Bill: $2,000/month
- Basic: $0 (no tech support)
- Developer: $60/month (3% of $2K)
- Business: $200/month (10% of $2K)
- **Recommendation:** Developer (affordable, some support)

**Scenario 2: Growing SaaS**
- AWS Bill: $25,000/month
- Business: $2,500/month
- Enterprise On-Ramp: $5,500/month
- **Recommendation:** Business (24/7 support, reasonable cost)

**Scenario 3: Large Enterprise**
- AWS Bill: $300,000/month
- Business: $21,000/month (7% blended rate)
- Enterprise On-Ramp: $18,000/month (6% blended rate)
- Enterprise: $24,000/month (8% blended rate) + dedicated TAM
- **Recommendation:** Enterprise (TAM value exceeds cost difference)

### Key Takeaways

✓ **Basic (Free):** Documentation and forums only, no tech support
✓ **Developer ($29+):** Email support, business hours, for dev/test environments
✓ **Business ($100+):** 24/7 phone/chat, <1 hour response, for production workloads
✓ **Enterprise On-Ramp ($5.5K+):** <30 min response, shared TAM pool, proactive guidance
✓ **Enterprise ($15K+):** <15 min response, dedicated TAM, unlimited reviews
✓ **Rule of thumb:** If downtime costs more than support, upgrade to Business+
✓ **Cost scales with AWS bill** (percentage-based pricing)
✓ Support plans can be changed anytime (takes effect next billing cycle)

---

## AWS Pricing Models

### What They Are

AWS offers **multiple payment models** for consuming cloud resources, each optimized for different usage patterns and commitment levels. Unlike traditional IT (buy a server, use for 3-5 years), AWS lets you pay only for what you use, when you use it—or commit long-term for deep discounts.

### Why Multiple Models Exist

**Problem they solve:**

Different workloads have different characteristics:
- **Steady-state applications:** Run 24/7, predictable (databases, web servers)
- **Variable workloads:** Traffic spikes during business hours, weekends (retail sites)
- **Batch processing:** Only runs certain times (nightly data processing)
- **Dev/test:** Only needed during work hours
- **Experiments:** Might not work out, shouldn't commit long-term

**One-size-fits-all pricing would mean**:
- Paying for resources you don't use (over-provisioning)
- Or lacking resources when you need them (under-provisioning)

**The solution:**
Multiple pricing models let you optimize costs based on your specific usage patterns and risk tolerance.

### The Four Primary Pricing Models

```
ASCII Diagram: AWS Pricing Models Spectrum

High Flexibility                              High Savings
Low Commitment                                High Commitment
        │                                            │
        ▼                                            ▼
┌──────────┬──────────┬───────────────┬──────────────────┐
│ On-Demand│ Reserved │ Savings Plans │ Spot Instances   │
│          │ Instances│               │                  │
│ $1.00/hr │ $0.65/hr │ $0.70/hr     │ $0.30/hr         │
│ No commit│ 1-3 years│ 1-3 years    │ Can be terminated│
└──────────┴──────────┴───────────────┴──────────────────┘

Pay as you go  →  Lock in discount  →  Lock in discount  →  Auction pricing
(No commitment)    (Specific instance)  (Flexible usage)     (Interruptible)
```

### 1. On-Demand Pricing

**What it is:**
Pay by the hour or second for compute capacity, with **no long-term commitments or upfront payments**. This is the default pricing model.

**How it works:**
```
Start EC2 instance → Runs for 3 hours 27 minutes → Stop instance
You pay: $0.10/hour × 3.45 hours = $0.345

No contract, no commitment, can stop anytime
```

**Characteristics:**
- **No upfront cost:** Start/stop resources anytime
- **No commitment:** Use for minutes or years
- **Highest per-hour cost:** Most expensive option
- **Billed to the second** (for Linux instances, minimum 60 seconds)

**When to use:**
- **Short-term, spiky, or unpredictable workloads**
- Applications being developed/tested
- First-time applications with unknown demand
- Don't want any commitments
- Experimentation and proof-of-concepts

**Example use case:**
```
Scenario: E-commerce site traffic

Normal traffic: 10 instances (On-Demand)
Black Friday spike: +40 instances for 1 day (On-Demand)

On-Demand flexibility allows handling spike without
year-long commitment to 50 instances
```

**Real cost example:**
- Instance type: t3.medium
- On-Demand price: $0.0416/hour
- Running 24/7 for one month: $0.0416 × 730 hours = $30.37/month

### 2. Reserved Instances (RIs)

**What they are:**
Commit to using a specific instance type in a specific region for **1 or 3 years** in exchange for significant discounts (up to 72% off On-Demand).

**How it works:**
```
Purchase: 1-year commitment for t3.medium in us-east-1
Discount: 40% off On-Demand price
Your cost: $0.025/hour (vs. $0.0416 On-Demand)

If you run matching instance → Automatic discount applied
If you don't run instance → You still pay (commitment)
```

**Three payment options:**

| Payment Option | Upfront Payment | Monthly Payment | Total Discount |
|----------------|-----------------|-----------------|----------------|
| **All Upfront** | 100% paid now | $0 | Highest (~72%) |
| **Partial Upfront** | ~50% paid now | Monthly payments | Medium (~55%) |
| **No Upfront** | $0 paid now | Monthly payments | Lowest (~40%) |

**Two offering classes:**

**Standard RIs:**
- Up to 72% discount
- Cannot change instance family (if you buy t3, you're stuck with t3)
- Can change Availability Zone, instance size (within same family)
- Can sell unused RIs on Reserved Instance Marketplace

**Convertible RIs:**
- Up to 54% discount (less than Standard)
- Can change instance family, OS, tenancy during term
- More flexibility, less discount
- Cannot sell on Marketplace

**When to use:**
- **Steady-state workloads with predictable usage**
- Production databases that run 24/7
- Web servers with consistent baseline traffic
- When you're confident the workload will run for 1-3 years

**Example use case:**
```
Scenario: Production database server

Runs 24/7, 365 days/year
Requirement won't change for at least 1 year

On-Demand cost: $0.416/hour × 8,760 hours/year = $3,645/year
Reserved (1-year, no upfront): $0.275/hour × 8,760 = $2,409/year
Savings: $1,236/year (34%)

Reserved (3-year, all upfront): $0.185/hour equivalent
Savings: ~55% off On-Demand
```

**Important notes:**
- You pay for the commitment regardless of actual usage
- Best for resources running >50% of the time
- Billing discount applies automatically when matching instances run

### 3. Savings Plans

**What they are:**
Commit to a **consistent amount of usage (measured in $/hour)** for 1 or 3 years, get discounts up to 72%. More flexible than Reserved Instances.

**How it works:**
```
Commit: $10/hour of compute usage for 1 year
AWS gives: Up to 66% discount on that $10/hour commitment
Flexibility: Use any instance type, any region, any OS
             Automatically applies to EC2, Lambda, Fargate

Month 1: You use $12/hour actual → $10 at discounted rate, $2 at On-Demand
Month 2: You use $8/hour actual → $8 at discounted rate, $2 commitment wasted
```

**Two types:**

**Compute Savings Plans:**
- Most flexible
- Applies to: EC2, Lambda, Fargate
- Any instance family, any region, any OS
- Up to 66% discount

**EC2 Instance Savings Plans:**
- Less flexible than Compute, more than Reserved Instances
- Applies to: EC2 only
- Must choose instance family (e.g., t3) and region
- Can change OS, instance size within family
- Up to 72% discount

**Comparison:**

```
Flexibility Spectrum:

On-Demand ────────────────────────────────────────────────► Most Flexible
(No commitment, highest cost)                               Zero discount

Compute Savings Plans ────────────────────────────────────► Very Flexible
(Any instance, any region)                                  66% discount

EC2 Instance Savings Plans ───────────────────────────────► Moderately Flexible
(One instance family + region)                              72% discount

Reserved Instances ────────────────────────────────────────► Least Flexible
(Specific instance type + region)                           72% discount
```

**When to use:**
- **Predictable usage across diverse workloads**
- Mix of EC2, Lambda, Fargate
- Want flexibility to change instance types
- Steady baseline usage but workloads evolve

**Example use case:**
```
Scenario: Microservices company

Usage breakdown:
├─ EC2 for web servers
├─ Lambda for API processing
├─ Fargate for containers
└─ Mix of instance types as needs change

Compute Savings Plan ($50/hour commitment):
├─ Automatically applies to all three services
├─ No need to predict specific instance types
└─ 66% discount on $50/hour = savings ~$16.50/hour = $12K/month
```

### 4. Spot Instances

**What they are:**
Purchase **unused EC2 capacity at up to 90% discount** in exchange for AWS being able to **reclaim instances with 2-minute warning** when capacity is needed elsewhere.

**How it works:**
```
AWS has spare capacity → You bid your max price → If spot price < your bid, instance launches
Spot price increases → If spot price > your bid, AWS terminates your instance (2-min warning)

Example:
On-Demand: $0.10/hour
Spot price: $0.03/hour (70% discount)
Your max bid: $0.05/hour

Instance runs at $0.03 until spot price rises above $0.05,
then AWS terminates with 2-minute warning
```

**Key characteristics:**
- **Massive discounts:** 50-90% off On-Demand
- **Can be interrupted:** AWS terminates with 2-minute notice
- **Spot price fluctuates:** Based on supply/demand
- **No commitment:** Pay only when instances run

**When to use (workloads that can tolerate interruption):**
- **Batch processing:** Can restart from checkpoint
- **Data analysis:** Store intermediate results, resume later
- **CI/CD:** Test runs can restart if interrupted
- **Rendering:** Video frames can be processed independently
- **Web crawling:** Checkpointed progress, continue later

**When NOT to use:**
- Databases (interruption causes data unavailability)
- Production web servers (users see errors during interruption)
- Real-time applications
- Stateful applications without checkpointing

**Spot Best Practices:**

```
Strategy: Diversify to reduce interruption

Single Instance Type (Bad):
└─ 100 × c5.large Spot instances in us-east-1a
   └─ If c5.large spot price spikes, ALL instances terminated

Diversified (Good):
├─ 30 × c5.large in us-east-1a
├─ 30 × c5a.large in us-east-1b (AMD processor, different pricing)
├─ 20 × c5n.large in us-east-1c (network-optimized)
└─ 20 × c6i.large in us-east-1a (newer generation)

Result: Interruptions affect only subset, not entire fleet
```

**Spot interruption handling:**

```python
# Example: Detect Spot interruption (2-minute warning)

import requests

# AWS provides instance metadata endpoint
METADATA_URL = "http://169.254.169.254/latest/meta-data/spot/instance-action"

def check_for_interruption():
    try:
        response = requests.get(METADATA_URL, timeout=1)
        if response.status_code == 200:
            # Spot interruption notice received
            print("Spot instance being terminated in 2 minutes!")
            # Save work, upload to S3, update database, etc.
            save_checkpoint()
            graceful_shutdown()
    except:
        # No interruption notice, continue working
        pass

# Check every 5 seconds during workload
```

**Real cost example:**
```
Batch processing job:
├─ Needs: 100 vCPUs for 10 hours
├─ On-Demand: c5.4xlarge (16 vCPUs) × 7 instances = $1.19/hour
│   └─ Total: $1.19 × 10 hours × 7 = $83.30
├─ Spot: Same instances at 70% discount = $0.36/hour
│   └─ Total: $0.36 × 10 hours × 7 = $25.20
└─ Savings: $58.10 (70% reduction)
```

### Comparing All Four Models

**Cost comparison (t3.medium in us-east-1):**

| Model | Price/Hour | Annual Cost (24/7) | Discount | Commitment | Interruption Risk |
|-------|------------|-------------------|----------|------------|-------------------|
| On-Demand | $0.0416 | $364 | 0% | None | None |
| Spot | ~$0.0125 | ~$109 | ~70% | None | Yes (2-min warning) |
| Savings Plan (1yr) | $0.0274 | $240 | 34% | 1 year | None |
| Reserved (1yr, No upfront) | $0.0275 | $241 | 34% | 1 year | None |
| Reserved (3yr, All upfront) | $0.0185 | $162 | 56% | 3 years | None |

### Hybrid Strategy (Real-World Best Practice)

**Don't use just one model—combine them:**

```
ASCII Diagram: Hybrid Pricing Strategy

Web Application with Variable Traffic:

Baseline (24/7): 10 instances
    └─► Reserved Instances (3-year)
        └─► Cheapest for steady baseline
        └─► $1,620/year for all 10

Peak hours (8am-8pm weekdays): +20 instances
    └─► Savings Plans (1-year)
        └─► Flexible for business hour traffic
        └─► $3,000/year commitment

Unexpected spikes (sales, news): +50 instances
    └─► On-Demand
        └─► Only when needed, pay by hour
        └─► ~$500/month for occasional spikes

Batch processing (nightly): 100 instances
    └─► Spot Instances
        └─► 70% discount, interruptions OK at night
        └─► $1,500/month for nightly jobs

Total savings: ~60% vs. all On-Demand
```

### Free Tier (Special Pricing Model)

Covered in detail in next section, but worth noting:
- 750 hours/month EC2 t2.micro (first 12 months)
- 5GB S3 storage (always free)
- 1 million Lambda requests/month (always free)

### Other AWS Pricing Considerations

**Data Transfer:**
- **Inbound:** Free (data coming into AWS)
- **Outbound:** Charged (data leaving AWS to internet)
  - First 100 GB/month: Free
  - Next 10 TB: $0.09/GB
  - Scales down with volume
- **Between Regions:** Charged
- **Within same Region (usually):** Free

**Storage:**
- S3: $0.023/GB/month (Standard)
- EBS: $0.10/GB/month (gp3 volumes)
- Glacier: $0.004/GB/month (archive)

**Requests:**
- API calls, Lambda invocations, S3 requests all charged per million requests

### Key Takeaways

✓ **On-Demand:** No commitment, highest cost, maximum flexibility
✓ **Reserved Instances:** 1-3 year commitment, up to 72% discount, specific instance type
✓ **Savings Plans:** 1-3 year commitment, up to 72% discount, more flexible than RIs
✓ **Spot:** Up to 90% discount, can be interrupted with 2-min warning
✓ **Best practice:** Combine models (RIs for baseline, Spot for batch, On-Demand for spikes)
✓ **Rule of thumb:** If usage >50% of time → commit (RI/Savings Plan)
✓ **Spot best for:** Fault-tolerant, stateless, checkpointed workloads
✓ Always monitor usage and optimize—AWS Cost Explorer helps identify savings

---

## Free Tier

### What It Is

The AWS Free Tier is a **collection of free usage limits** for AWS services, designed to let you explore, experiment, and build without incurring costs. It includes three types of offers: 12-month free for new accounts, always free services, and limited free trials.

### Why It Exists

**Problem it solves:**

AWS wants to:
- **Lower barrier to entry:** Let people try AWS without credit card risk
- **Enable learning:** Build real projects without cost concern
- **Support small workloads:** Tiny applications can run permanently free
- **Encourage adoption:** Once you're comfortable with AWS, you'll scale up (and pay)

### Three Types of Free Tier Offers

```
ASCII Diagram: Free Tier Structure

┌────────────────────────────────────────────────────────────┐
│                    AWS Free Tier                           │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. 12-Month Free Tier                                     │
│     └─ Starts when you create AWS account                 │
│     └─ Expires after 12 months                            │
│     └─ Example: 750 hours/month EC2 t2.micro              │
│                                                            │
│  2. Always Free                                            │
│     └─ Never expires                                       │
│     └─ Available to all AWS accounts                      │
│     └─ Example: 1M Lambda requests/month                  │
│                                                            │
│  3. Short-Term Free Trials                                 │
│     └─ Starts when you first use service                  │
│     └─ Expires after trial period (30 days-12 months)     │
│     └─ Example: 2 months free Amazon Inspector            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 1. 12-Month Free Tier (New AWS Accounts)

**Duration:** First 12 months from account creation date

**Major offerings:**

**EC2 (Compute):**
- **750 hours/month** of Linux or Windows t2.micro instances
- Equivalent to: 1 instance running 24/7 for entire month
- Or: 2 instances running 12 hours/day each
- Instance type: t2.micro (1 vCPU, 1 GB RAM)
- Enough for: Small websites, dev environments, learning

**S3 (Storage):**
- **5 GB** standard storage
- **20,000 GET requests**
- **2,000 PUT requests**
- Enough for: Personal websites, small file storage

**RDS (Databases):**
- **750 hours/month** of db.t2.micro, db.t3.micro, or db.t4g.micro
- **20 GB database storage**
- **20 GB backup storage**
- Enough for: Development databases, small applications

**CloudFront (CDN):**
- **50 GB data transfer out**
- **2 million HTTP/HTTPS requests**
- Enough for: Small website with global distribution

**Other notable services:**
- **EBS:** 30 GB of SSD or Magnetic storage
- **Elastic Load Balancing:** 750 hours + 15 GB data processing
- **SNS:** 1 million publishes
- **SES:** 3,000 messages (if from EC2)
- **Glacier:** 10 GB retrieval

**What happens after 12 months:**
- Free tier expires
- You start paying standard On-Demand rates
- No automatic warnings (use Billing Alarms!)

**Example 12-month free usage:**

```
Scenario: Personal blog hosted on AWS

Month 1-12 (Free):
├─ 1 × EC2 t2.micro running 24/7 (web server)
├─ 5 GB S3 storage (images, static files)
├─ 20 GB RDS db.t2.micro (database)
└─ CloudFront for faster load times

Cost: $0/month (within free tier limits)

Month 13 onwards (Paid):
Same resources now cost:
├─ EC2: $8.40/month
├─ S3: $0.12/month
├─ RDS: $15.20/month
└─ CloudFront: ~$5/month
Total: ~$29/month
```

### 2. Always Free

**Duration:** Never expires, available to all AWS customers

**Major always-free offerings:**

**Lambda (Serverless Compute):**
- **1 million requests/month**
- **400,000 GB-seconds of compute time**
- Enough for: Moderate API workloads, automation scripts

**DynamoDB (NoSQL Database):**
- **25 GB storage**
- **25 read capacity units** (can handle ~200M requests/month for small items)
- **25 write capacity units**
- Enough for: Many production workloads at small scale

**CloudWatch (Monitoring):**
- **10 custom metrics**
- **10 alarms**
- **1 million API requests**
- **5 GB log data ingestion**
- Enough for: Basic monitoring of several resources

**SNS (Notifications):**
- **1 million publishes**
- **100,000 HTTP deliveries**
- **1,000 email deliveries**
- Enough for: Application notifications, alerts

**SQS (Message Queue):**
- **1 million requests**
- Enough for: Small-scale microservices communication

**CloudFront (CDN):**
- **1 TB data transfer out per month**
- **10 million HTTP/HTTPS requests**
- Enough for: Moderate traffic websites

**CodeBuild (CI/CD):**
- **100 build minutes/month** (general1.small)
- Enough for: Small projects, learning

**AWS Budgets:**
- **2 active budgets** (cost monitoring)
- Essential for preventing bill shock

**Other always-free services:**
- **Cognito:** 50,000 monthly active users
- **API Gateway:** 1 million REST API calls/month (first 12 months only)
- **Glue:** 1 million objects stored in Data Catalog
- **X-Ray:** 100,000 traces recorded, 1 million traces scanned

**Real-world example (always free):**

```
Scenario: Serverless API backend

Architecture:
├─ API Gateway (handles requests)
├─ Lambda (processes logic)
├─ DynamoDB (stores data)
├─ CloudWatch (monitors health)
└─ SNS (sends alerts)

Monthly usage:
├─ API Gateway: 500,000 requests
├─ Lambda: 500,000 invocations × 128MB × 200ms = 12,800 GB-seconds
├─ DynamoDB: 10 GB storage, 50M reads, 10M writes
├─ CloudWatch: 5 alarms, 2 GB logs
└─ SNS: 10,000 notifications

Cost with Always Free Tier: $0/month (all within limits!)

If this grows beyond free tier:
├─ Lambda beyond 1M requests: ~$0.20 per 1M
├─ DynamoDB beyond 25GB: $0.25/GB/month
└─ Still very cheap at small scale
```

### 3. Short-Term Free Trials

**Services with trial periods:**

**Amazon Inspector (Security):**
- **15-day free trial**
- Vulnerability scanning for EC2, container images
- After trial: Pay per assessment

**Amazon Macie (Data Security):**
- **30-day free trial**
- Discovers and protects sensitive data in S3
- After trial: Pay per GB scanned

**Amazon GuardDuty (Threat Detection):**
- **30-day free trial**
- Monitors for malicious activity
- After trial: Pay per GB analyzed

**Amazon Detective (Security Investigation):**
- **30-day free trial**
- Investigates security findings
- After trial: Pay per GB ingested

**Amazon Lightsail (Simple VPS):**
- **First month free** (select plans)
- Simple virtual private servers
- Includes compute, storage, and data transfer

**SageMaker (Machine Learning):**
- **250 hours/month free** (first 2 months)
- t2.medium or t3.medium notebook instances

### Free Tier Gotchas (Common Ways to Accidentally Exceed)

**1. Multiple instances**
```
Free Tier: 750 hours/month EC2 t2.micro

Mistake: Running 2 × t2.micro instances 24/7
Usage: 2 × 730 hours = 1,460 hours
Overage: 710 hours × $0.0116 = $8.24/month (charged)

Correct: Run 1 × t2.micro 24/7 OR 2 × t2.micro half-time each
```

**2. Wrong instance type**
```
Free Tier: t2.micro only

Mistake: Launch t2.small (thinking "it's still t2")
Result: $0.023/hour × 730 hours = $16.79/month (NO FREE TIER)

Correct: Use exactly t2.micro (or t3.micro for RDS)
```

**3. Data transfer out**
```
Free Tier: 15 GB/month combined data transfer out

Mistake: Host a popular video (100 MB) downloaded 200 times
Usage: 100 MB × 200 = 20 GB
Overage: 5 GB × $0.09/GB = $0.45 (charged)

Correct: Use CloudFront (1 TB free) for large file distribution
```

**4. Elastic IP addresses not attached**
```
Free Tier: 1 Elastic IP is free IF attached to running instance

Mistake: Create Elastic IP, don't attach it (or stop instance)
Cost: $0.005/hour × 730 = $3.65/month

Correct: Always attach Elastic IPs to running instances, release unused IPs
```

**5. EBS volumes after instance termination**
```
Free Tier: 30 GB EBS storage

Mistake: Terminate EC2 instance but EBS volume remains
Cost: 30 GB × $0.10/GB = $3/month (charged after free tier)

Correct: Delete EBS volumes when terminating instances (or check "delete on termination")
```

### Monitoring Free Tier Usage

**AWS provides tools to track usage:**

**1. Billing Dashboard**
- Shows current month usage
- Alerts when approaching limits
- Available at: console.aws.amazon.com/billing

**2. Budgets**
- Set budget (e.g., $1/month)
- Get email when approaching or exceeding
- 2 budgets free (Always Free)

**3. Cost Explorer**
- Visualize spending over time
- Filter by service, region, tag

**Example budget alert setup:**

```bash
# Create a budget that alerts at $1 (for free tier monitoring)
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json

# budget.json:
{
  "BudgetName": "Free-Tier-Monitor",
  "BudgetLimit": {
    "Amount": "1",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}

# Email alert when 80% of $1 budget reached
```

### Maximum Value from Free Tier Strategy

**Scenario: Student learning AWS for certifications**

```
Optimal Free Tier Setup:

Compute:
└─ 1 × EC2 t2.micro (24/7 study environment)
   └─ Install: AWS CLI, SDKs, practice tools

Storage:
├─ 5 GB S3 (store study materials, lab files)
├─ 30 GB EBS (attached to EC2)
└─ 20 GB RDS MySQL (practice database)

Networking:
└─ CloudFront (50 GB free, host static study site)

Serverless:
├─ Lambda (1M requests, practice serverless)
├─ DynamoDB (25 GB, practice NoSQL)
└─ API Gateway (practice building APIs)

Monitoring:
├─ CloudWatch (10 alarms, monitor everything)
└─ AWS Budgets (2 budgets, prevent overspend)

Total cost: $0/month for 12 months
(+Always Free services continue after 12 months)

All services needed for AWS certification practice included
```

### After Free Tier Expires

**Options to continue low-cost:**

**1. Stay within Always Free limits**
- Lambda, DynamoDB, CloudFront can handle small production workloads free forever

**2. Use Lightsail**
- $3.50/month for simple VPS (cheaper than EC2 after free tier)

**3. Optimize spending**
- Right-size instances (t3.nano instead of t3.micro for light workloads)
- Use Spot Instances (70-90% discount)
- Stop resources when not in use (dev environments overnight/weekends)

**4. AWS Educate / AWS Credits**
- Students get extended free tier through AWS Educate
- Startups get credits through AWS Activate program

### Key Takeaways

✓ **Three types:** 12-month free (new accounts), Always Free, Short-term trials
✓ **12-month free:** 750 hours/month EC2 t2.micro, 5 GB S3, 20 GB RDS
✓ **Always free:** Lambda (1M requests), DynamoDB (25 GB), CloudWatch basics
✓ **Common mistakes:** Wrong instance type, multiple instances, unused Elastic IPs
✓ **Monitoring essential:** Set up Budgets with $1 threshold to avoid surprise bills
✓ **After expiration:** Always Free continues, 12-month free converts to paid
✓ **Enough for:** Learning, small personal projects, MVP development
✓ **Not enough for:** Production workloads with significant traffic

---

**Document Complete**

This guide covered all AWS Fundamentals topics requested:
✓ Global Infrastructure (Regions, AZs, Edge Locations)
✓ Local Zones & Wavelength Zones
✓ AWS Account Structure
✓ AWS Organizations
✓ Service Control Policies (SCPs)
✓ AWS Control Tower
✓ Landing Zones
✓ AWS Well-Architected Framework
✓ Shared Responsibility Model
✓ AWS Support Plans
✓ AWS Pricing Models
✓ Free Tier

Each section includes:
- Clear conceptual explanations
- ASCII diagrams for visualization
- Real-world examples and use cases
- Common pitfalls and best practices
- Decision frameworks and comparisons