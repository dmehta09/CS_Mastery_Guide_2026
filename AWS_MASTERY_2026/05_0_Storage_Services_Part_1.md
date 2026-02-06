# AWS Storage Services - Complete Conceptual Guide

## Table of Contents

1. [S3 (Simple Storage Service)](#s3-simple-storage-service)
2. [EBS (Elastic Block Store)](#ebs-elastic-block-store)
3. [EFS (Elastic File System)](#efs-elastic-file-system)
4. [FSx](#fsx)
5. [Storage Gateway](#storage-gateway)

---

## S3 (Simple Storage Service)

### What is S3?

**Amazon S3** is an **object storage service** that stores data as objects within containers called buckets. Unlike traditional file systems (hierarchical) or block storage (raw volumes), S3 is a **flat namespace** where each object is accessed via a unique key (path-like identifier).

**Why it exists:**
- Traditional storage struggles with **scalability** (finite disk space, manual scaling)
- Need for **durable, available storage** accessible from anywhere via HTTP/HTTPS
- Demand for **cost-effective archival** and backup solutions
- Requirements for **unlimited storage** without capacity planning

**Problems it solves:**
- **Infinite scalability**: Store unlimited data without provisioning
- **Durability**: 99.999999999% (11 nines) - data won't be lost
- **Availability**: Accessible globally via API, no server management
- **Cost optimization**: Pay only for what you use, with multiple pricing tiers
- **Integration**: Native integration with virtually all AWS services

---

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Region                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Availability Zone 1                     │  │
│  │  ┌────────────────────────────────────────┐          │  │
│  │  │  S3 Storage Infrastructure             │          │  │
│  │  │  (Replicated across multiple devices) │          │  │
│  │  └────────────────────────────────────────┘          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Availability Zone 2                     │  │
│  │  ┌────────────────────────────────────────┐          │  │
│  │  │  S3 Storage Infrastructure             │          │  │
│  │  │  (Automatic replication)               │          │  │
│  │  └────────────────────────────────────────┘          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Availability Zone 3                     │  │
│  │  ┌────────────────────────────────────────┐          │  │
│  │  │  S3 Storage Infrastructure             │          │  │
│  │  │  (Automatic replication)               │          │  │
│  │  └────────────────────────────────────────┘          │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

         ▲                                      ▲
         │                                      │
         │  HTTPS Requests                      │
         │                                      │
    ┌────┴─────┐                          ┌────┴─────┐
    │  Client  │                          │   AWS    │
    │  (App)   │                          │ Services │
    └──────────┘                          └──────────┘
```

**Key principle**: S3 automatically replicates data across **at least 3 Availability Zones** (for Standard class) within a region, ensuring data survives facility failures.

---

### Buckets & Objects

#### Buckets

A **bucket** is a container (namespace) for objects. Think of it as a top-level folder, but globally unique.

**Key characteristics:**
- **Globally unique name** across all AWS accounts (e.g., `my-company-logs-2026`)
- **Region-specific**: Created in a specific AWS region
- **Flat namespace**: No true folder hierarchy (though console simulates it)
- **Unlimited objects**: Can store billions of objects per bucket
- **100 bucket limit** per account (soft limit, can be increased)

**Naming rules:**
- 3-63 characters, lowercase, no underscores
- Must start with letter or number
- Cannot be formatted as IP address (e.g., `192.168.1.1`)

#### Objects

An **object** is the fundamental entity stored in S3, consisting of:

1. **Object Data** (the file content) - 0 bytes to 5TB
2. **Metadata** (key-value pairs describing the object)
3. **Key** (unique identifier within the bucket)
4. **Version ID** (if versioning enabled)

**Object Key Structure:**

```
Bucket: my-app-data
Key: users/2026/january/user-12345.json
      └─┬──┘ └┬─┘ └──┬───┘ └─────┬──────┘
     prefix  prefix prefix    filename

Full S3 URI: s3://my-app-data/users/2026/january/user-12345.json
HTTPS URL: https://my-app-data.s3.amazonaws.com/users/2026/january/user-12345.json
```

**Important concept**: The `/` characters are part of the key name, not actual folders. S3 is a flat structure that *simulates* hierarchy for human convenience.

---

### Storage Classes

S3 offers **multiple storage classes** to optimize cost based on access patterns. Each class has different pricing for storage, retrieval, and availability.

**Why multiple classes exist:**
- Not all data is accessed equally (80/20 rule: 20% of data accounts for 80% of access)
- Storing rarely-accessed data at "hot storage" prices wastes money
- Different compliance/archival needs require different durability/retrieval characteristics

#### Storage Class Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│                    Access Frequency Spectrum                     │
└──────────────────────────────────────────────────────────────────┘

Hot (Frequent) ←─────────────────────────────────────→ Cold (Rare)

Standard    Standard-IA   One Zone-IA   Glacier    Glacier Deep
  │            │              │         Flexible    Archive
  │            │              │         Instant      │
  │            │              │         Retrieval    │
  │            │              │            │         │
  ▼            ▼              ▼            ▼         ▼
$$$$$       $$$$           $$$          $$         $
(high       (medium        (medium      (low       (lowest
storage)    storage)       storage)     storage)   storage)

millisec    millisec      millisec     millisec   12-48 hrs
latency     latency       latency      latency    retrieval
```

#### 1. **S3 Standard** (Default)

**Use case**: Frequently accessed data, production workloads

**Characteristics:**
- **Durability**: 99.999999999% (11 nines)
- **Availability**: 99.99%
- **Availability Zones**: ≥3 AZs
- **Retrieval**: Milliseconds, no retrieval fee
- **Cost**: Highest storage cost, no retrieval cost

**Best for:**
- Active application data (user uploads, content delivery)
- Frequently accessed analytics data
- Mobile/gaming applications

**Example scenario**: A social media platform storing user profile pictures accessed thousands of times daily.

---

#### 2. **S3 Standard-IA (Infrequent Access)**

**Use case**: Data accessed less than once per month but needs immediate access when requested

**Characteristics:**
- **Durability**: 99.999999999%
- **Availability**: 99.9%
- **Availability Zones**: ≥3 AZs
- **Retrieval**: Milliseconds + per-GB retrieval fee
- **Minimum storage duration**: 30 days
- **Minimum object size**: 128KB

**Best for:**
- Disaster recovery backups
- Older media files (previous year's photos)
- Compliance archives requiring quick access

**Cost consideration**: If you retrieve more than ~25% of stored data monthly, Standard becomes cheaper.

---

#### 3. **S3 One Zone-IA**

**Use case**: Infrequently accessed, reproducible data where availability is less critical

**Characteristics:**
- **Durability**: 99.999999999% (within the single AZ)
- **Availability**: 99.5%
- **Availability Zones**: 1 AZ (risk: data lost if AZ destroyed)
- **Retrieval**: Milliseconds + per-GB retrieval fee
- **Cost**: 20% cheaper than Standard-IA

**Best for:**
- Secondary backup copies (you have the primary elsewhere)
- Data that can be regenerated (thumbnails, transcoded media)
- Development/test environments

**Risk**: If the Availability Zone fails, data is permanently lost.

---

#### 4. **S3 Glacier Flexible Retrieval** (formerly Glacier)

**Use case**: Long-term archives accessed 1-2 times per year

**Characteristics:**
- **Durability**: 99.999999999%
- **Availability**: 99.99% (after retrieval)
- **Availability Zones**: ≥3 AZs
- **Retrieval times**:
  - Expedited: 1-5 minutes (expensive)
  - Standard: 3-5 hours (moderate)
  - Bulk: 5-12 hours (cheapest)
- **Minimum storage duration**: 90 days

**Best for:**
- Financial records (7-year retention)
- Medical imaging archives
- Digital media preservation

**Important**: You must initiate a retrieval job; data isn't immediately accessible.

---

#### 5. **S3 Glacier Instant Retrieval**

**Use case**: Archival data needing instant access, accessed once per quarter

**Characteristics:**
- **Retrieval**: Milliseconds (like Standard)
- **Cost**: Lower storage than Standard-IA, higher retrieval cost
- **Minimum storage duration**: 90 days

**Best for:**
- Medical images requiring instant access for legal/compliance
- News media archives

**Difference from Glacier Flexible**: No waiting for retrieval jobs—data available instantly.

---

#### 6. **S3 Glacier Deep Archive**

**Use case**: Data accessed less than once per year, long-term regulatory retention

**Characteristics:**
- **Durability**: 99.999999999%
- **Availability Zones**: ≥3 AZs
- **Retrieval times**:
  - Standard: 12 hours
  - Bulk: 48 hours
- **Minimum storage duration**: 180 days
- **Cost**: Lowest storage cost in S3

**Best for:**
- Regulatory archives (10+ year retention)
- Scientific research data preservation
- Historical records

**Example**: A bank storing transaction records for 10 years to meet regulatory compliance.

---

#### 7. **S3 Intelligent-Tiering**

**Use case**: Unpredictable or changing access patterns

**How it works:**
S3 automatically moves objects between access tiers based on usage:

```
┌─────────────────────────────────────────────────────────┐
│          S3 Intelligent-Tiering Architecture            │
└─────────────────────────────────────────────────────────┘

Object uploaded to bucket
         │
         ▼
┌────────────────────┐
│  Frequent Access   │ ◄─── No monitoring fee
│  (30 days)         │      (Auto-moved if accessed)
└─────────┬──────────┘
          │ (Not accessed for 30 days)
          ▼
┌────────────────────┐
│ Infrequent Access  │ ◄─── Lower storage cost
│  (30-90 days)      │      (Auto-moved if accessed)
└─────────┬──────────┘
          │ (Not accessed for 90 days)
          ▼
┌────────────────────┐
│  Archive Instant   │ ◄─── Even lower cost
│  Access (90-180)   │      (Optional tier)
└─────────┬──────────┘
          │ (Not accessed for 180 days)
          ▼
┌────────────────────┐
│  Archive Access    │ ◄─── Minutes to hours retrieval
│  (180-270 days)    │      (Optional tier)
└─────────┬──────────┘
          │ (Not accessed for 270+ days)
          ▼
┌────────────────────┐
│ Deep Archive       │ ◄─── Lowest cost, hours retrieval
│  Access (270+)     │      (Optional tier)
└────────────────────┘
```

**Characteristics:**
- **Small monitoring fee**: Per 1,000 objects
- **No retrieval fees**: Unlike manual IA classes
- **No minimum storage duration**: Unlike other Glacier tiers
- **Automatic optimization**: No manual intervention

**Best for:**
- Data lakes with unknown access patterns
- Mixed workloads (some hot, some cold data)
- When you don't want to manage lifecycle policies manually

**Cost consideration**: Cost-effective if you store objects >128KB for >30 days.

---

#### Storage Class Decision Tree

```
                    Start: Storing data in S3
                              │
                              ▼
                   How often is data accessed?
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
     Daily/Weekly         Monthly          Rarely/Never
          │                   │                   │
          ▼                   ▼                   ▼
    S3 Standard    Need immediate access?    How fast retrieval?
                              │                   │
                        ┌─────┴─────┐        ┌────┼────┐
                        │           │        │    │    │
                       Yes          No    Instant 3-5hr 12-48hr
                        │           │        │    │    │
                        ▼           ▼        ▼    ▼    ▼
                  Can lose if   Standard-IA  Glacier  Glacier
                  AZ fails?     (or One      Instant  Flexible
                        │       Zone-IA)     Retrieval  Deep
                   ┌────┴────┐                        Archive
                   │         │
                  Yes       No
                   │         │
                   ▼         ▼
              One Zone   Standard-IA
                 -IA

              Don't know access pattern?
                        │
                        ▼
                Intelligent-Tiering
```

---

### S3 Lifecycle Policies

**What are Lifecycle Policies?**

Automated rules that **transition objects between storage classes** or **delete objects** based on age or other criteria. This eliminates manual data management and optimizes costs automatically.

**Why they exist:**
- Data value decreases over time (hot → cold)
- Manual data management doesn't scale (millions of objects)
- Cost optimization requires moving old data to cheaper tiers
- Compliance requires automatic deletion after retention periods

**Problems they solve:**
- **Cost reduction**: Automatically move aging data to cheaper storage
- **Compliance**: Auto-delete data after legal retention period
- **Operational overhead**: No manual intervention needed
- **Storage optimization**: Remove incomplete multipart uploads

---

#### How Lifecycle Rules Work

```
┌────────────────────────────────────────────────────────────┐
│               Lifecycle Policy Execution Flow              │
└────────────────────────────────────────────────────────────┘

Day 0: Object created in S3 Standard
  │
  │  ┌──────────────────────────────────────┐
  │  │  Lifecycle Rule 1: Transition        │
  │  │  - After 30 days → Standard-IA       │
  │  │  - After 90 days → Glacier Flexible  │
  │  │  - After 365 days → Deep Archive     │
  │  └──────────────────────────────────────┘
  │
  ├─ Day 30: Auto-transition to Standard-IA
  │           (Lower storage cost, retrieval fees apply)
  │
  ├─ Day 90: Auto-transition to Glacier Flexible
  │           (Even lower cost, 3-5 hour retrieval)
  │
  ├─ Day 365: Auto-transition to Deep Archive
  │            (Lowest cost, 12-48 hour retrieval)
  │
  │  ┌──────────────────────────────────────┐
  │  │  Lifecycle Rule 2: Expiration        │
  │  │  - After 2555 days (7 years) DELETE  │
  │  └──────────────────────────────────────┘
  │
  └─ Day 2555: Object permanently deleted
```

**Execution timing**: Lifecycle actions happen at **midnight UTC** and may take up to 24-48 hours to complete.

---

#### Lifecycle Rule Components

A lifecycle rule consists of:

1. **Scope** (which objects to apply to)
   - Entire bucket
   - Prefix filter (e.g., `logs/2025/`)
   - Tag filter (e.g., `Archive=true`)
   - Object size filter (min/max bytes)

2. **Actions**
   - **Transition actions**: Move to different storage class
   - **Expiration actions**: Delete objects
   - **Version actions**: Handle previous versions

3. **Timing**
   - Days after object creation
   - Specific date

---

#### Example Configuration (Conceptual)

```
Rule Name: "Archive Application Logs"
Scope:
  - Prefix: "logs/"
  - Tag: "Type=application"

Transition Actions:
  - After 30 days → Standard-IA
  - After 90 days → Glacier Flexible Retrieval

Expiration Action:
  - After 365 days → DELETE

Result:
  logs/2026/app.log (Day 0)   → S3 Standard
  logs/2026/app.log (Day 30)  → Standard-IA (75% storage savings)
  logs/2026/app.log (Day 90)  → Glacier (90% storage savings)
  logs/2026/app.log (Day 365) → DELETED
```

---

#### Common Lifecycle Patterns

**Pattern 1: Progressive Archival**

```
0 days     → S3 Standard (active data)
30 days    → Standard-IA (recent but infrequent)
90 days    → Glacier Flexible (compliance archive)
7 years    → DELETE (end of retention)
```

**Use case**: Financial transaction records, audit logs

---

**Pattern 2: Temporary Data Cleanup**

```
0 days     → S3 Standard
7 days     → DELETE
```

**Use case**: Temporary video processing files, cache data, staging uploads

---

**Pattern 3: Incomplete Multipart Upload Cleanup**

```
Special Action: Delete incomplete multipart uploads after 7 days
```

**Why this matters**: Failed multipart uploads leave orphaned parts that cost money. This rule automatically cleans them up.

---

**Pattern 4: Versioned Object Management**

```
Current version:
  0 days    → S3 Standard
  30 days   → Standard-IA

Previous versions (non-current):
  0 days    → Immediate transition to Glacier
  90 days   → DELETE
```

**Use case**: Document management with versioning where only the latest version needs quick access.

---

#### Transition Constraints

**Important**: Not all transitions are allowed. S3 enforces a **waterfall model**:

```
Allowed Transition Waterfall (One-way only):

S3 Standard
    ↓
Standard-IA / One Zone-IA
    ↓
Intelligent-Tiering
    ↓
Glacier Instant Retrieval
    ↓
Glacier Flexible Retrieval
    ↓
Glacier Deep Archive

❌ Cannot transition: Deep Archive → Standard (must restore first)
❌ Cannot transition: Glacier → Standard-IA (must restore first)
✅ Can transition: Standard → Deep Archive (skip intermediate classes)
```

**Minimum storage durations** must be respected:
- Standard-IA / One Zone-IA: 30 days
- Glacier Instant Retrieval: 90 days
- Glacier Flexible: 90 days
- Deep Archive: 180 days

**Early deletion charges**: If you delete objects before minimum duration, you're charged for the full minimum period.

---

#### Lifecycle Policy Practical Considerations

**Cost calculation example:**

```
Scenario: 1 TB of log data
- Month 0: Store in Standard: $23/month
- Month 1: Transition to Standard-IA: $12.50/month (46% savings)
- Month 3: Transition to Glacier: $4/month (83% savings)

Annual cost with lifecycle: $23 + $12.50 + ($4 × 10) = $75.50
Annual cost without lifecycle: $23 × 12 = $276

Savings: $200.50 (73%)
```

**Transition costs**: Each transition has a small per-request fee (~$0.01 per 1,000 objects). For millions of small objects, this can add up.

**Best practices:**
- Use prefix/tag filters to avoid transitioning frequently-accessed data
- Consolidate small objects before transitioning (reduces request costs)
- Test policies on a small dataset first
- Monitor S3 Storage Lens to verify expected behavior
- Don't transition objects accessed frequently (defeats the purpose)

---

#### Monitoring Lifecycle Actions

AWS doesn't directly notify when lifecycle actions occur, but you can:

1. **S3 Inventory**: Generate reports showing storage class of all objects
2. **CloudWatch Metrics**: Monitor storage bytes per storage class
3. **S3 Storage Lens**: Dashboard showing lifecycle transition trends
4. **CloudTrail**: Logs API calls (transitions appear as S3 internal calls)

---

### S3 Versioning

**What is S3 Versioning?**

A bucket-level setting that **preserves every version of every object** stored in the bucket, including all writes and deletions. When enabled, S3 assigns a unique **version ID** to each object version.

**Why it exists:**
- Accidental deletions are permanent in unversioned buckets
- Overwrites destroy previous data (no "undo")
- Compliance requirements mandate audit trails
- Ransomware can encrypt/destroy all accessible data

**Problems it solves:**
- **Accidental deletion protection**: Deleted objects can be restored
- **Accidental overwrite protection**: Previous versions remain accessible
- **Audit trail**: Complete history of all changes
- **Ransomware protection**: Previous clean versions survive encryption attacks
- **Compliance**: Regulatory requirements for data immutability

---

#### Versioning States

A bucket can be in one of three states:

```
┌──────────────────────────────────────────────────────────┐
│                Versioning State Machine                  │
└──────────────────────────────────────────────────────────┘

    Unversioned (Default)
           │
           │ Enable Versioning
           │ (One-way, cannot fully reverse)
           ▼
       Enabled ←──────────┐
           │              │
           │ Suspend      │ Enable
           │              │
           ▼              │
      Suspended ──────────┘

❌ Cannot return to: Unversioned (once enabled)
✅ Can toggle between: Enabled ↔ Suspended
```

**Unversioned** (default):
- No version IDs
- Overwrites replace objects
- Deletes are permanent
- Version ID is `null`

**Enabled**:
- All new uploads get unique version IDs
- Previous versions are preserved
- Delete operations create a delete marker (not permanent)
- Cannot be reversed to unversioned

**Suspended**:
- New uploads get version ID `null`
- Previous versions remain accessible
- Doesn't delete existing versions
- Deletes create delete markers

---

#### How Versioning Works

**Scenario 1: Multiple Uploads (Same Key)**

```
Bucket: my-documents (Versioning: Enabled)

Upload #1: document.pdf
  └─ Version ID: abc123 (current version)

Upload #2: document.pdf (updated content)
  ├─ Version ID: def456 (current version) ← Latest
  └─ Version ID: abc123 (previous version) ← Still exists!

Upload #3: document.pdf (another update)
  ├─ Version ID: ghi789 (current version) ← Latest
  ├─ Version ID: def456 (previous version)
  └─ Version ID: abc123 (oldest version)

GET s3://my-documents/document.pdf → Returns ghi789 (latest)
GET s3://my-documents/document.pdf?versionId=abc123 → Returns abc123
```

**Key insight**: Each upload creates a new version, but the object key remains the same. The latest version is always returned unless you specify a version ID.

---

**Scenario 2: Delete Operations**

```
Before Delete:
  document.pdf (Version: ghi789) ← Current

After DELETE request:
  ├─ Delete Marker (new "version", no content) ← Current
  ├─ Version ghi789 (still exists, hidden)
  ├─ Version def456 (still exists, hidden)
  └─ Version abc123 (still exists, hidden)

GET s3://my-documents/document.pdf → 404 Not Found

To restore:
  DELETE the delete marker (by version ID)
    → Version ghi789 becomes current again

To permanently delete:
  DELETE s3://my-documents/document.pdf?versionId=ghi789
    → Version ghi789 is permanently deleted
    → Delete marker still exists (object still appears deleted)
```

**Important**: A standard delete in a versioned bucket doesn't actually delete data—it just hides it behind a delete marker.

---

#### Delete Marker Deep Dive

A **delete marker** is a special placeholder with:
- No data content (0 bytes)
- A unique version ID
- Object key of the deleted object
- Metadata indicating it's a delete marker

```
┌────────────────────────────────────────────────────┐
│         Delete Marker vs Permanent Delete          │
└────────────────────────────────────────────────────┘

Standard DELETE (no version ID specified):
  document.pdf
    Before: [v3] [v2] [v1]
    After:  [DeleteMarker] [v3] [v2] [v1]
              ↑ Current (GET returns 404)
              └─ All previous versions still exist

Permanent DELETE (version ID specified):
  DELETE document.pdf?versionId=v2
    Before: [DeleteMarker] [v3] [v2] [v1]
    After:  [DeleteMarker] [v3] [v1]
              └─ v2 is permanently gone (unrecoverable)
```

---

#### Versioning + Lifecycle Policies

Lifecycle rules can manage both current and non-current versions:

```
Current Version Rule:
  - Transition to Standard-IA after 30 days
  - Transition to Glacier after 90 days

Non-Current Version Rule:
  - Transition to Glacier immediately (becomes non-current)
  - Delete after 90 days (non-current)

Example Timeline:
  Day 0:   Upload v1 → S3 Standard (current)
  Day 10:  Upload v2 → S3 Standard (current)
           v1 becomes non-current → Immediate to Glacier
  Day 30:  v2 → Standard-IA (current version lifecycle)
  Day 100: v1 deleted (90 days as non-current)
  Day 120: v2 → Glacier (current version lifecycle)
```

**NoncurrentVersionTransition**: Applies when a version becomes non-current (new version uploaded)

**NoncurrentVersionExpiration**: Deletes versions after they've been non-current for X days

---

#### Versioning Use Cases

**1. Accidental Deletion Protection**

```
User accidentally deletes entire folder:
  DELETE /reports/2025/

Result:
  - Delete markers created for all objects
  - All data still exists as previous versions

Recovery:
  - Remove delete markers
  - Or copy previous versions to new keys
```

---

**2. Ransomware Protection**

```
Ransomware encrypts all accessible files:

Without versioning:
  - Files overwritten with encrypted versions
  - Original data lost forever

With versioning:
  - Encrypted versions become current
  - Original clean versions still exist as previous versions
  - Restore by removing encrypted versions or accessing version IDs

Additional protection: Enable MFA Delete (requires MFA to delete versions)
```

---

**3. Audit Trail / Compliance**

```
Regulatory requirement: Maintain 7-year history of all documents

Configuration:
  - Versioning: Enabled
  - Lifecycle: Delete non-current versions after 7 years
  - S3 Object Lock: Prevent deletion during retention period

Result:
  - Complete change history for every object
  - Immutable audit trail
  - Automatic cleanup after retention period
```

---

#### Versioning Cost Considerations

**Storage costs multiply:**

```
Example:
  Original file: document.pdf (10 MB)

  Update 1: document.pdf (10 MB, slightly modified)
  Update 2: document.pdf (10 MB, slightly modified)
  Update 3: document.pdf (10 MB, slightly modified)

  Total storage: 40 MB (4 versions × 10 MB)
  Monthly cost: 4× the cost of a single file
```

**Cost optimization strategies:**

1. **Lifecycle policies for non-current versions**:
   - Transition to Glacier immediately
   - Delete after retention period

2. **Version expiration**:
   - Keep only last N versions
   - Delete versions older than X days

3. **Intelligent-Tiering for versions**:
   - Automatically move infrequently accessed versions

**Best practice**: Enable versioning + lifecycle policies together to balance protection and cost.

---

#### MFA Delete

**MFA Delete** requires **multi-factor authentication** to:
- Permanently delete object versions
- Suspend/enable versioning

**Configuration:**
```
Requirement: Root account credentials + MFA device
Enable MFA Delete → Can only be done via AWS CLI/API, not Console

Effect:
  - DELETE operations require MFA code
  - Prevents accidental/malicious permanent deletion
  - Additional protection against compromised credentials
```

**Use case**: Highly sensitive data, compliance requirements, protection against insider threats.

---

#### Versioning Best Practices

✅ **Enable for critical buckets**: Production data, backups, compliance documents

✅ **Combine with lifecycle policies**: Auto-manage costs for non-current versions

✅ **Use S3 Inventory**: Monitor version counts and storage consumption

✅ **Enable MFA Delete**: For sensitive data requiring extra protection

✅ **Monitor version count**: High version counts may indicate issues (e.g., automated overwrites)

❌ **Don't enable for high-churn buckets**: Temporary data, cache, high-frequency updates (cost explosion)

❌ **Don't ignore non-current versions**: They consume storage and cost money

---

### S3 Replication (CRR & SRR)

**What is S3 Replication?**

Automatic, **asynchronous copying** of objects from a source bucket to one or more destination buckets. There are two types:

- **CRR (Cross-Region Replication)**: Destination bucket in a different AWS region
- **SRR (Same-Region Replication)**: Destination bucket in the same AWS region

**Why it exists:**
- Single-region storage creates regional failure risk
- Compliance may require data in specific geographies
- Low-latency access needed from multiple locations
- Disaster recovery requires off-site copies
- Production/test environment separation needed

**Problems it solves:**
- **Disaster recovery**: Regional failure doesn't lose data
- **Latency reduction**: Serve users from geographically closer buckets
- **Compliance**: Meet data residency requirements (e.g., GDPR)
- **Account isolation**: Replicate to different AWS accounts
- **Operational separation**: Separate prod and dev data

---

#### CRR vs SRR Comparison

```
┌──────────────────────────────────────────────────────────────┐
│            Cross-Region Replication (CRR)                    │
└──────────────────────────────────────────────────────────────┘

    Region: us-east-1                  Region: eu-west-1
   ┌─────────────────┐                ┌─────────────────┐
   │  Source Bucket  │  ============> │  Dest Bucket    │
   │  (my-app-data)  │  (Async copy)  │  (my-app-eu)    │
   └─────────────────┘                └─────────────────┘
          │                                   │
          ▼                                   ▼
    US-based users                      EU-based users

Use cases:
  - Disaster recovery
  - Compliance (data residency)
  - Latency reduction
  - Geographic redundancy

┌──────────────────────────────────────────────────────────────┐
│            Same-Region Replication (SRR)                     │
└──────────────────────────────────────────────────────────────┘

         Region: us-east-1
   ┌─────────────────┐                ┌─────────────────┐
   │  Source Bucket  │  ============> │  Dest Bucket    │
   │  (production)   │  (Async copy)  │  (backup)       │
   └─────────────────┘                └─────────────────┘
          │                                   │
     AWS Account A                       AWS Account B

Use cases:
  - Log aggregation
  - Prod/test environment sync
  - Account isolation
  - Compliance (separate accounts)
```

---

#### How Replication Works

**Replication Flow:**

```
┌────────────────────────────────────────────────────────────┐
│                  Replication Process                       │
└────────────────────────────────────────────────────────────┘

1. Object uploaded to source bucket
         │
         ▼
2. S3 checks replication rule
         │
         │  Does object match rule criteria?
         │  (prefix, tags, storage class)
         │
         ├─ No → Replication skipped
         │
         └─ Yes → Continue
              │
              ▼
3. S3 assumes replication IAM role
         │
         ▼
4. S3 reads object from source
         │
         ▼
5. S3 encrypts data in transit (TLS)
         │
         ▼
6. S3 writes object to destination
         │
         ▼
7. Metadata replicated:
   - Object key
   - Metadata
   - ACLs (optional)
   - Tags (optional)
   - Object Lock settings (if enabled)
         │
         ▼
8. Replication complete
   (typically within 15 minutes, can be seconds)
```

**Important**: Replication is **asynchronous**. Most objects replicate within minutes, but there's no SLA guaranteeing instant replication.

---

#### Replication Requirements

To enable replication, you MUST have:

1. **Versioning enabled** on both source and destination buckets
2. **IAM role** with permissions to:
   - Read objects from source bucket
   - Write objects to destination bucket
   - Read bucket replication configuration
3. **Replication rule** defining what to replicate

**Optional but common**:
- S3 Batch Replication (for existing objects)
- Replication Time Control (RTC) for SLA
- Metrics and event notifications

---

#### Replication Rule Configuration

A replication rule specifies:

**1. Scope (what to replicate)**:
```
Option A: Entire bucket
  → All objects replicate

Option B: Prefix filter
  → Only objects matching prefix replicate
  → Example: "logs/2026/" → Only 2026 logs

Option C: Tag filter
  → Only objects with specific tags replicate
  → Example: Tag "Replicate=true"

Option D: Combination
  → Prefix: "documents/" AND Tag: "Critical=yes"
```

**2. Destination**:
```
- Destination bucket ARN (can be in different account)
- Storage class for replicas (can differ from source)
- Encryption (can re-encrypt with different KMS key)
- Replication ownership (who owns replicas)
```

**3. Additional options**:
```
- Replicate delete markers: Yes/No
- Replicate previous versions: Yes/No
- Replicate replica modifications: Yes/No
- Replication Time Control (RTC): Yes/No
```

---

#### What Gets Replicated (and What Doesn't)

**✅ Automatically replicated** (after rule creation):

- Newly uploaded objects
- Object metadata
- Object tags (if enabled in rule)
- Object ACLs (if enabled)
- S3 Object Lock retention (if enabled)

**❌ NOT replicated** by default:

- **Existing objects** (uploaded before rule created)
  - Solution: Use S3 Batch Replication

- **Delete markers** (unless explicitly enabled)
  - Protects against accidental deletion cascading

- **Permanent version deletes**
  - Prevents malicious deletion from replicating

- **Objects encrypted with SSE-C** (customer-provided keys)
  - S3 doesn't have the key to decrypt/replicate

- **Objects in Glacier or Deep Archive**
  - Must restore to Standard first

- **Lifecycle actions**
  - Lifecycle runs independently on each bucket

**Objects replicated from another bucket**:
  - Not replicated by default (prevents replication loops)
  - Can enable "replica modification replication" if needed

---

#### Replication Time Control (RTC)

**What is RTC?**

An optional feature providing a **15-minute SLA** for replication, with monitoring and notifications.

```
┌──────────────────────────────────────────────────────────┐
│         Without RTC         │         With RTC           │
├─────────────────────────────┼────────────────────────────┤
│ No SLA                      │ 99.99% of objects within   │
│                             │ 15 minutes                 │
├─────────────────────────────┼────────────────────────────┤
│ No metrics                  │ Detailed CloudWatch metrics│
│                             │ showing replication lag    │
├─────────────────────────────┼────────────────────────────┤
│ Best effort                 │ S3 Event notifications for │
│                             │ missed SLA objects         │
├─────────────────────────────┼────────────────────────────┤
│ Lower cost                  │ Higher cost (per GB)       │
└──────────────────────────────────────────────────────────┘
```

**Use cases for RTC**:
- Compliance requiring strict replication timelines
- Critical disaster recovery scenarios
- Real-time analytics requiring synchronized data

**Monitoring**: CloudWatch metric `ReplicationLatency` tracks time to replicate.

---

#### Batch Replication

**Purpose**: Replicate **existing objects** that existed before the replication rule was created.

```
Scenario:
  Bucket "my-data" has 10 TB of data
  Enable replication rule on Day 0

  Without Batch Replication:
    - Only new objects (uploaded after Day 0) replicate
    - 10 TB of existing data does NOT replicate

  With Batch Replication:
    - Create a Batch Replication job
    - S3 copies all 10 TB to destination
    - Can filter by prefix, tags, creation date
    - Shows progress and completion status
```

**How it works**:
1. S3 generates manifest (list of objects to replicate)
2. S3 Batch Operations processes the manifest
3. Objects are replicated respecting the replication rule
4. You pay for batch operations + data transfer

**Use case**: Migrating existing data lake to new region, disaster recovery setup.

---

#### Cross-Account Replication

**Why replicate to a different AWS account?**
- Organizational separation (dev vs prod accounts)
- Compliance (separate control of data)
- Shared services (centralized logs from multiple accounts)

**Permission Model:**

```
┌────────────────────────────────────────────────────────────┐
│           Cross-Account Replication Setup                  │
└────────────────────────────────────────────────────────────┘

Account A (Source):
  - Bucket: my-app-logs
  - Replication IAM Role: SourceReplicationRole
    → Trust policy: Allow S3 to assume role
    → Permissions: Read my-app-logs, Write to Account B bucket

Account B (Destination):
  - Bucket: centralized-logs
  - Bucket Policy:
    → Allow Account A's SourceReplicationRole to write objects
  - Optional: Change replica ownership to Account B
    (otherwise Account A still owns the objects)
```

**Ownership consideration**: By default, replicas are owned by the source account. Enable **Replica Ownership Override** to transfer ownership to destination account.

---

#### Replication vs Backup

Many people confuse replication with backup. They serve different purposes:

```
┌──────────────────────────────────────────────────────────┐
│         Replication          │         Backup            │
├──────────────────────────────┼───────────────────────────┤
│ Real-time sync               │ Point-in-time snapshot    │
├──────────────────────────────┼───────────────────────────┤
│ Active-active copies         │ Historical archive        │
├──────────────────────────────┼───────────────────────────┤
│ DELETE replicates (optional) │ DELETEs don't affect      │
│                              │ historical backups        │
├──────────────────────────────┼───────────────────────────┤
│ Protects against:            │ Protects against:         │
│ - Regional failure           │ - Accidental deletion     │
│ - Latency                    │ - Ransomware              │
│                              │ - Corruption              │
│                              │ - Historical recovery     │
└──────────────────────────────────────────────────────────┘
```

**Best practice**: Use BOTH. Replication for disaster recovery, versioning/lifecycle for backup.

---

#### Replication Cost Considerations

**Costs involved**:

1. **Data transfer**:
   - CRR: $0.02 per GB transferred out of source region
   - SRR: Lower transfer costs (same region)

2. **Storage**:
   - You pay for storage in BOTH buckets
   - Can use different storage classes to optimize

3. **Replication requests**:
   - PUT requests in destination bucket
   - GET requests from source (during replication)

4. **RTC**: Additional per-GB fee if enabled

**Cost optimization example:**

```
Scenario: Replicate 1 TB of infrequent data

Option 1: Standard → Standard (both buckets)
  Source: $23/month
  Destination: $23/month
  Transfer: $20 (one-time)
  Total: $66/month ongoing

Option 2: Standard → Standard-IA (different class)
  Source: $23/month
  Destination: $12.50/month
  Transfer: $20 (one-time)
  Total: $55.50/month ongoing (16% savings)
```

---

#### Replication Monitoring

**Key metrics to monitor**:

1. **Replication latency**: Time between source upload and replica availability
   - Metric: `ReplicationLatency` (if RTC enabled)

2. **Pending bytes**: How much data is queued for replication
   - Metric: `BytesPendingReplication`

3. **Operations pending**: Number of objects queued
   - Metric: `OperationsPendingReplication`

4. **Failed operations**: Objects that failed to replicate
   - Metric: `OperationsFailedReplication`
   - Requires investigation (permissions, encryption issues)

**S3 Event Notifications**: Can trigger Lambda/SNS when replication fails or completes.

---

#### Common Replication Issues

**Problem 1: Objects not replicating**

Checklist:
- ✅ Versioning enabled on both buckets?
- ✅ IAM role has correct permissions?
- ✅ Object matches replication rule filter?
- ✅ Object not encrypted with SSE-C?
- ✅ Object not in Glacier/Deep Archive?

**Problem 2: Cross-account replication permission denied**

Solution:
- Verify destination bucket policy allows source account
- Enable replica ownership override if needed
- Check KMS key permissions (if using encryption)

**Problem 3: High replication lag**

Causes:
- Large object sizes (multi-GB files)
- High upload rate overwhelming replication queue
- Network congestion

Solution:
- Enable RTC for guaranteed 15-min SLA
- Monitor `BytesPendingReplication` metric
- Contact AWS Support if persistent

---

### S3 Object Lock

**What is S3 Object Lock?**

A feature that implements **Write-Once-Read-Many (WORM)** model, preventing object versions from being deleted or overwritten for a specified retention period or indefinitely.

**Why it exists:**
- Regulatory compliance (SEC 17a-4, FINRA, HIPAA) requires immutable records
- Legal hold requirements during litigation
- Protection against insider threats or malicious deletion
- Ransomware protection (attackers can't delete/modify data)

**Problems it solves:**
- **Compliance**: Meet regulatory requirements for data immutability
- **Legal protection**: Preserve evidence during investigations
- **Accidental deletion**: Even administrators can't delete during retention
- **Malicious activity**: Compromised credentials can't destroy audit logs
- **Ransomware**: Attackers can't encrypt or delete protected data

---

#### Object Lock Architecture

```
┌────────────────────────────────────────────────────────────┐
│                 Object Lock Protection                     │
└────────────────────────────────────────────────────────────┘

Object: financial-record.pdf (Version: v1)
  │
  ├─ Object Lock Enabled
  │    │
  │    ├─ Retention Period: 7 years
  │    │    └─ Mode: Compliance (cannot override)
  │    │
  │    └─ Legal Hold: No
  │
  ▼
Protection Active:
  ❌ DELETE version v1 → Access Denied
  ❌ OVERWRITE v1 → Access Denied (creates new version instead)
  ✅ READ v1 → Allowed
  ✅ Upload new version → Allowed (creates v2, v1 still locked)

After 7 years:
  ✅ DELETE v1 → Allowed
```

**Critical concept**: Object Lock protects at the **version level**, not the object key level. You can still upload new versions, but locked versions cannot be deleted.

---

#### Object Lock Requirements

**MUST be enabled at bucket creation**:
```
Cannot enable Object Lock on existing bucket
Must create new bucket with Object Lock enabled
Versioning is automatically enabled (required)
```

**Once enabled**:
- Cannot be disabled
- Versioning cannot be suspended
- This is permanent for the bucket

**Why so strict?** Regulatory compliance requires immutability from the start—enabling it later wouldn't protect historical data.

---

#### Retention Modes

Object Lock offers two retention modes:

#### 1. **Governance Mode**

**Permissions-based protection**: Users with special permissions can override.

```
Characteristics:
  - Most users: Cannot delete or overwrite
  - Users with "s3:BypassGovernanceRetention" permission: Can override
  - Root account: Can grant bypass permissions

Use case:
  - Internal policies (not regulatory)
  - Testing Object Lock before compliance mode
  - Need for emergency overrides by authorized personnel

Example:
  Retention: 1 year (Governance)

  Regular user DELETE → Access Denied

  Admin with bypass permission DELETE with header:
    x-amz-bypass-governance-retention: true
    → Deletion succeeds
```

**When to use**: Internal data retention policies where authorized overrides may be needed.

---

#### 2. **Compliance Mode**

**Absolute protection**: No one can delete or shorten retention, not even root account.

```
Characteristics:
  - Nobody can delete protected versions
  - Nobody can shorten retention period
  - Root account: Cannot override
  - AWS Support: Cannot override

  Can extend retention: Yes
  Can shorten retention: No
  Can delete version: No (until retention expires)

Use case:
  - Regulatory compliance (SEC, FINRA, HIPAA)
  - Legal requirements
  - Maximum protection against deletion

Example:
  Retention: 7 years (Compliance)

  Any user DELETE → Access Denied
  Root account DELETE → Access Denied
  AWS Support ticket → Cannot override

  Only option: Wait 7 years
```

**When to use**: Regulatory compliance, legal requirements, data that must never be deleted prematurely.

---

#### Retention Period

**Retention period** specifies how long an object version is protected.

```
Retention Configuration:

Option 1: Days
  Retain for: 365 days
  Start: Object version creation
  End: 365 days later

Option 2: Years
  Retain for: 7 years
  Start: Object version creation
  End: 7 years later

Example Timeline:
  Day 0: Upload financial-record.pdf (v1)
         Retention: 7 years (Compliance)

  Day 1-2555: DELETE v1 → Access Denied

  Day 2556: DELETE v1 → Allowed (retention expired)
```

**Default retention**: Can set bucket default retention that applies to all new objects automatically.

---

#### Legal Hold

**Legal Hold** is a separate, indefinite lock independent of retention period.

```
Legal Hold vs Retention Period:

┌──────────────────────────────────────────────────────────┐
│         Retention Period    │      Legal Hold            │
├─────────────────────────────┼────────────────────────────┤
│ Time-based (days/years)     │ No expiration (indefinite) │
├─────────────────────────────┼────────────────────────────┤
│ Automatically expires       │ Manual removal required    │
├─────────────────────────────┼────────────────────────────┤
│ Set at upload or later      │ Can add/remove anytime     │
├─────────────────────────────┼────────────────────────────┤
│ Compliance/Governance mode  │ No modes (on/off only)     │
└──────────────────────────────────────────────────────────┘
```

**How Legal Hold works:**

```
Object: lawsuit-evidence.pdf (v1)

Legal Hold: ON
  ❌ DELETE → Access Denied
  ❌ OVERWRITE → Access Denied
  ✅ READ → Allowed

Legal Hold: OFF (removed by authorized user)
  ✅ DELETE → Allowed (if no retention period)
```

**Use case**:
- Active litigation (preserve evidence)
- Regulatory investigation
- Need to hold data without knowing duration upfront
- Override retention period expiration (e.g., retention expires but legal hold remains)

**Permission**: Requires `s3:PutObjectLegalHold` permission to enable/disable.

---

#### Retention + Legal Hold Interaction

Objects can have **both** retention period and legal hold:

```
Scenario:
  Object: contract.pdf (v1)
  Retention: 1 year (Compliance)
  Legal Hold: ON

Timeline:
  Month 0-12: Protected by BOTH retention + legal hold
              ❌ Cannot delete

  Month 12: Retention expires
            ❌ Still cannot delete (legal hold active)

  Month 18: Legal hold removed by authorized user
            ✅ Can now delete

Rule: If EITHER retention OR legal hold is active → Protected
```

---

#### Object Lock Practical Example

**Scenario**: Financial services company must retain transaction records for 7 years per SEC Rule 17a-4(f).

```
Configuration:

1. Create bucket: financial-records-2026
   ✅ Enable Object Lock at creation
   ✅ Versioning: Automatically enabled

2. Set default retention:
   Mode: Compliance
   Period: 7 years

3. Upload transaction record:
   PUT transaction-20260206.json
   → Version ID: abc123
   → Retention: 7 years (Compliance) - applied automatically

4. Attempt to delete after 1 year:
   DELETE transaction-20260206.json?versionId=abc123
   → Error: Access Denied (retention active)

5. After 7 years (2033):
   DELETE transaction-20260206.json?versionId=abc123
   → Success (retention expired)

6. Emergency litigation in 2027:
   PUT Legal Hold ON transaction-20260206.json?versionId=abc123
   → Now protected by legal hold EVEN AFTER retention expires
   → Must manually remove legal hold before deletion allowed
```

---

#### Object Lock vs Versioning vs MFA Delete

Many people confuse these features. Here's how they differ:

```
┌──────────────────────────────────────────────────────────────────┐
│  Feature         │ Versioning │ MFA Delete │ Object Lock         │
├──────────────────┼────────────┼────────────┼─────────────────────┤
│ Protects against │ Accidental │ Malicious  │ Any deletion        │
│                  │ overwrites │ deletion   │ (time-based)        │
├──────────────────┼────────────┼────────────┼─────────────────────┤
│ Delete marker    │ Yes (soft  │ Requires   │ No delete markers   │
│                  │ delete)    │ MFA        │ (hard protection)   │
├──────────────────┼────────────┼────────────┼─────────────────────┤
│ Can admin delete?│ Yes        │ Yes (with  │ No (Compliance)     │
│                  │            │ MFA)       │ Special perm (Gov.) │
├──────────────────┼────────────┼────────────┼─────────────────────┤
│ Regulatory       │ No         │ No         │ Yes (WORM)          │
│ compliance       │            │            │                     │
├──────────────────┼────────────┼────────────┼─────────────────────┤
│ Time-based       │ No         │ No         │ Yes                 │
│ protection       │            │            │                     │
└──────────────────────────────────────────────────────────────────┘
```

**Best practice**: Use together for defense-in-depth:
- Versioning: Protects against accidental changes
- MFA Delete: Adds authentication requirement
- Object Lock: Provides time-based immutability for compliance

---

#### Object Lock Cost Considerations

**Additional costs**:
- **No extra charge for Object Lock itself**
- **Storage costs**: All versions consume storage
  - Cannot delete versions during retention = storage accumulates

**Cost optimization**:
```
Problem: 10 years of data, each year 1 TB
         10 TB total storage costs

Solution: Lifecycle policies + Object Lock
  - Years 0-7: S3 Standard (compliance retention)
  - Years 7-10: Cannot delete (retention active)
              But can transition to Glacier Deep Archive

Configuration:
  Object Lock: 7 years (Compliance)
  Lifecycle: Transition to Deep Archive after upload

Result:
  - Still compliant (retention preserved)
  - Storage in Deep Archive: 90% cost savings
  - After 7 years: Can delete from Deep Archive
```

**Important**: Storage class transitions are allowed during retention period, but deletion is not.

---

#### Monitoring Object Lock

**CloudTrail events** to monitor:
- `PutObjectRetention`: When retention is applied/modified
- `PutObjectLegalHold`: When legal hold is added/removed
- `GetObjectRetention`: When retention info is retrieved
- `BypassGovernanceRetention`: When override is used (Governance mode)

**S3 Inventory reports**: Include retention and legal hold status for all objects.

**Best practice**: Set up CloudWatch Alarms for:
- Legal hold additions (may indicate litigation)
- Governance bypass usage (audit trail for overrides)

---

### S3 Access Points

**What are S3 Access Points?**

Named network endpoints with **dedicated permissions policies** that simplify access management to shared S3 buckets. Each Access Point has its own hostname and access policy.

**Why they exist:**
- Bucket policies become complex with many applications/teams (100+ statements)
- Different apps need different access patterns to same bucket
- Managing cross-account access in bucket policies is cumbersome
- Audit trails are difficult with single bucket policy

**Problems they solve:**
- **Simplified permissions**: Each app/team gets its own access point with tailored policy
- **Policy scalability**: Avoid single massive bucket policy
- **Access isolation**: Separate network paths for different use cases
- **Delegation**: Teams manage their own access point policies
- **Auditing**: Clear separation of which app/team accessed what

---

#### Access Points Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Traditional Bucket Access                     │
└────────────────────────────────────────────────────────────┘

                    Bucket Policy
                   (100+ statements)
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
    App A              App B              App C
  (read-only)       (write logs/)      (full access)

Problem: Complex bucket policy, hard to audit, single point of failure


┌────────────────────────────────────────────────────────────┐
│              Access Points Architecture                    │
└────────────────────────────────────────────────────────────┘

                    S3 Bucket: shared-data
                    (Simple bucket policy:
                     "Delegate to Access Points")
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
  Access Point A      Access Point B     Access Point C
  (analytics-ap)      (logs-ap)          (admin-ap)
  Policy: Read        Policy: Write      Policy: Full
  Prefix: data/       Prefix: logs/      Prefix: *
       │                   │                   │
    App A              App B              App C
  (Analytics team)   (Logging team)     (Admin team)

Benefits:
  - Simple, focused policies per access point
  - Clear ownership and auditing
  - Independent policy management
```

---

#### How Access Points Work

**Creation process:**

```
1. Create S3 bucket: shared-data-bucket

2. Update bucket policy to delegate to Access Points:
   {
     "Statement": [{
       "Effect": "Allow",
       "Principal": { "AWS": "*" },
       "Action": "s3:*",
       "Resource": "*",
       "Condition": {
         "StringEquals": {
           "s3:DataAccessPointAccount": "123456789012"  // Your account ID
         }
       }
     }]
   }
   // This says: "Allow access only through Access Points in this account"

3. Create Access Point: analytics-readonly-ap
   Bucket: shared-data-bucket
   Network origin: Internet

4. Create Access Point policy:
   {
     "Statement": [{
       "Effect": "Allow",
       "Principal": { "AWS": "arn:aws:iam::123456789012:role/AnalyticsRole" },
       "Action": "s3:GetObject",
       "Resource": "arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-readonly-ap/object/analytics/*"
     }]
   }
   // Analytics team can only read objects under analytics/ prefix

5. Application uses Access Point ARN instead of bucket name:
   OLD: s3://shared-data-bucket/analytics/report.csv
   NEW: arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-readonly-ap/object/analytics/report.csv
```

---

#### Access Point Policies

**Key differences from bucket policies:**

```
Bucket Policy:
  - Applies to entire bucket
  - Single policy for all access
  - Maximum 20 KB size
  - Becomes complex with many users/apps

Access Point Policy:
  - Applies only to that access point
  - Separate policy per access point
  - Maximum 20 KB per access point policy
  - Simple, focused policies
  - Up to 10,000 access points per account
```

**Example multi-tenant scenario:**

```
Bucket: customer-data

Access Point 1: customer-a-ap
  Policy: Customer A can only access customer-a/* prefix

Access Point 2: customer-b-ap
  Policy: Customer B can only access customer-b/* prefix

Access Point 3: analytics-ap
  Policy: Analytics team can read all customer-* prefixes

Access Point 4: admin-ap
  Policy: Admin can do everything

Result:
  - Each customer completely isolated
  - Analytics has read-only cross-customer view
  - Admin maintains full control
  - Simple, auditable policies
```

---

#### VPC Access Points

**VPC Access Points** restrict access to only within a specific VPC (no internet access).

```
┌────────────────────────────────────────────────────────────┐
│                 VPC Access Point                           │
└────────────────────────────────────────────────────────────┘

    Your VPC (vpc-abc123)
   ┌─────────────────────────────────────┐
   │                                     │
   │  ┌──────────────┐                  │
   │  │   EC2        │                  │
   │  │   Instance   │                  │
   │  └──────┬───────┘                  │
   │         │                           │
   │         │ s3:GetObject via          │
   │         │ VPC Access Point          │
   │         ▼                           │
   │  ┌──────────────────┐              │
   │  │ VPC Endpoint     │              │
   │  │ (S3 Gateway)     │              │
   │  └──────┬───────────┘              │
   └─────────┼──────────────────────────┘
             │
             │ (Traffic stays within AWS network)
             ▼
   ┌──────────────────────┐
   │  Access Point        │
   │  (VPC-only)          │
   │  Network: VPC only   │
   └──────┬───────────────┘
          │
          ▼
   ┌──────────────────┐
   │  S3 Bucket       │
   └──────────────────┘

Internet user attempts access:
  ❌ Access Denied (Access Point only accepts VPC traffic)

EC2 instance in VPC:
  ✅ Access Granted (within allowed VPC)
```

**Use cases**:
- Private data that should never be accessible from internet
- Compliance requirements (data must stay within VPC)
- Additional security layer for sensitive buckets

**Configuration**:
```
Access Point settings:
  Network origin: Virtual Private Cloud (VPC)
  VPC: vpc-abc123

Access Point policy:
  Automatically restricts to specified VPC
  Additional policies layer on top
```

---

### S3 Multi-Region Access Points

**What are Multi-Region Access Points?**

A **global endpoint** that dynamically routes requests to the lowest-latency S3 bucket replica across multiple AWS regions.

**Why they exist:**
- Global applications need low latency worldwide
- Managing multiple regional endpoints is complex
- Failover between regions requires application changes
- Users in different geographies have varying latencies

**Problems they solve:**
- **Latency optimization**: Auto-route to closest bucket
- **Simplified architecture**: One global endpoint, not region-specific URLs
- **Automatic failover**: If region is down, route to next closest
- **Active-active setup**: All regions serve traffic simultaneously
- **No application changes**: Apps use single endpoint

---

#### Multi-Region Access Point Architecture

```
┌────────────────────────────────────────────────────────────┐
│       Multi-Region Access Point (Global Endpoint)          │
└────────────────────────────────────────────────────────────┘

Global Endpoint:
  my-app.mrap.accesspoint.s3-global.amazonaws.com
                │
                │ AWS Global Accelerator
                │ (Anycast routing)
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
Region:     Region:     Region:
us-east-1   eu-west-1   ap-south-1
┌─────┐     ┌─────┐     ┌─────┐
│Bucket│    │Bucket│    │Bucket│
│(rep) │◄──►│(rep) │◄──►│(rep) │
└─────┘     └─────┘     └─────┘
  │           │           │
  │           │           │
User A      User B      User C
(New York)  (London)    (Mumbai)
  │           │           │
  └───────────┴───────────┘
  All use same global endpoint,
  automatically routed to closest replica

Replication: Bi-directional (multi-way) between all regions
```

**Key insight**: Users always use the SAME endpoint (`*.mrap.accesspoint.s3-global.amazonaws.com`), but AWS routes them to the geographically nearest bucket automatically.

---

#### How Multi-Region Access Points Work

**Request flow:**

```
1. User in Mumbai makes request:
   GET https://my-app.mrap.accesspoint.s3-global.amazonaws.com/photo.jpg

2. AWS Global Accelerator receives request

3. Anycast routing determines:
   - User location: Mumbai, India
   - Available replicas: us-east-1, eu-west-1, ap-south-1
   - Closest replica: ap-south-1

4. Request routed to ap-south-1 bucket:
   GET s3://my-app-ap-south-1/photo.jpg

5. Response returned through Global Accelerator:
   photo.jpg served to user with low latency

If ap-south-1 fails:
   - Automatic failover to next closest: eu-west-1
   - No application changes needed
   - Transparent to user
```

---

#### Setup Requirements

To use Multi-Region Access Points:

1. **Multiple S3 buckets** in different regions with **same data**
2. **S3 Replication** configured bi-directionally between all buckets
3. **Multi-Region Access Point** created referencing all buckets

```
Example Setup:

Step 1: Create buckets in 3 regions
  - us-east-1: my-app-us
  - eu-west-1: my-app-eu
  - ap-south-1: my-app-ap

Step 2: Enable versioning on all buckets
  (Required for replication)

Step 3: Configure bi-directional replication
  - my-app-us ↔ my-app-eu
  - my-app-us ↔ my-app-ap
  - my-app-eu ↔ my-app-ap

Step 4: Create Multi-Region Access Point
  Name: my-app-mrap
  Buckets:
    - my-app-us (us-east-1)
    - my-app-eu (eu-west-1)
    - my-app-ap (ap-south-1)

Step 5: Application uses MRAP endpoint
  OLD: https://my-app-us.s3.us-east-1.amazonaws.com/photo.jpg
  NEW: https://my-app-mrap.mrap.accesspoint.s3-global.amazonaws.com/photo.jpg
```

---

#### Failover and Routing

**Automatic failover:**

```
Normal operation:
  User → Closest region bucket

Region failure (e.g., eu-west-1 degraded):
  Users routed to eu-west-1 → Auto-fail to next closest region
  No manual intervention
  No application changes

Latency-based routing:
  AWS measures real-time latency to each replica
  Routes to lowest latency (not just geographic proximity)
```

**Routing controls** (optional):

```
Can configure routing policies:
  - Active-Active: All regions serve traffic (default)
  - Active-Passive: Only use certain regions unless failure

Replication policies:
  - Can adjust replication priority per region
  - Can temporarily pause replication
```

---

#### Multi-Region Access Point vs CloudFront

People often confuse these. They serve different purposes:

```
┌────────────────────────────────────────────────────────────┐
│   Multi-Region Access Point │    CloudFront CDN            │
├──────────────────────────────┼──────────────────────────────┤
│ Routes to S3 buckets         │ Caches content at edge       │
├──────────────────────────────┼──────────────────────────────┤
│ Always fetches from S3       │ Serves from cache (edge)     │
├──────────────────────────────┼──────────────────────────────┤
│ Good for: Dynamic content,   │ Good for: Static content,    │
│ writes, uploads              │ reads, frequently accessed   │
├──────────────────────────────┼──────────────────────────────┤
│ Latency: Tens of ms          │ Latency: Single-digit ms     │
│ (S3 regional endpoint)       │ (edge location)              │
├──────────────────────────────┼──────────────────────────────┤
│ Cost: Standard S3 pricing    │ Cost: Data transfer out from │
│ + request charges            │ CloudFront (cheaper than S3) │
└────────────────────────────────────────────────────────────┘
```

**Best practice**: Use BOTH together:
- MRAP for S3 uploads/writes (low latency, multi-region failover)
- CloudFront for content delivery (caching, lower latency reads)

---

#### Multi-Region Access Point Use Cases

**Use Case 1: Global application with regional failover**

```
Application: Video streaming platform
Regions: us-east-1 (primary), eu-west-1 (secondary), ap-south-1 (secondary)

Without MRAP:
  - Application hardcoded to use us-east-1 endpoint
  - If us-east-1 fails, app needs code deployment to switch endpoint

With MRAP:
  - Application uses MRAP global endpoint
  - Automatic failover to eu-west-1 if us-east-1 fails
  - Zero code changes, zero downtime
```

**Use Case 2: Compliance with data residency + global access**

```
Requirement: EU users' data must stay in EU, but admins worldwide need access

Solution:
  - Bucket in eu-west-1: EU user data
  - Bucket in us-east-1: US user data
  - Bucket in ap-south-1: APAC user data
  - MRAP routing policy: Route users to their regional bucket
  - Admins use MRAP to access all regions transparently

Compliance maintained, global access simplified
```

---

#### Cost Considerations

**Additional costs**:

1. **Data routing charge**: $0.0033 per GB routed through MRAP (in addition to standard S3 costs)
2. **Replication costs**: Multi-way replication between all regions
3. **Storage costs**: Data stored in multiple regions

**Cost comparison**:

```
Scenario: 100 GB uploaded globally, 1 TB downloads monthly

Option 1: Single region (us-east-1) + CloudFront
  Storage: $2.30/month
  CloudFront data transfer: ~$85/month
  Total: ~$87.30/month

Option 2: Multi-Region Access Point (3 regions)
  Storage: $2.30 × 3 = $6.90/month
  Replication: One-time transfer ~$6
  MRAP routing: 1 TB × $0.0033 = $3.30/month
  Total: ~$16.20/month + $6 one-time

Option 3: Both (MRAP for writes, CloudFront for reads)
  Best of both worlds but higher complexity
```

**When to use**:
- Global writes (user uploads, APIs)
- Strict failover requirements
- Multi-region compliance needs

**When NOT to use**:
- Read-only static content (use CloudFront instead, cheaper)
- Single-region applications
- Cost-sensitive workloads with low traffic

---

### S3 Object Lambda

**What is S3 Object Lambda?**

A feature that allows you to **process and transform objects on-the-fly** during S3 GET requests using AWS Lambda functions, without changing the stored object or creating modified copies.

**Why it exists:**
- Different applications need different formats of same data (JSON vs XML, full vs redacted)
- Creating multiple copies of data wastes storage (cost and management)
- Pre-processing all possible variations is impractical
- Users need personalized views of data (redaction, watermarking)

**Problems it solves:**
- **Storage efficiency**: One source object, many transformed views (no duplicate storage)
- **Dynamic transformation**: Process only when requested (on-demand)
- **Personalization**: Different users get different views of same object
- **Data protection**: Redact sensitive fields without storing multiple versions
- **Format conversion**: Serve different formats (CSV→JSON, image resizing) without permanent copies

---

#### Object Lambda Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Traditional Approach (Without Object Lambda)  │
└────────────────────────────────────────────────────────────┘

Source Data: customer-data.json (full PII)
  ↓
Preprocessing:
  ├─ Create redacted-customer-data.json (PII removed)
  ├─ Create masked-customer-data.json (PII masked)
  └─ Create summary-customer-data.json (aggregated)

Storage:
  - Original: 100 MB
  - Redacted: 100 MB
  - Masked: 100 MB
  - Summary: 10 MB
  Total: 310 MB stored

Maintenance:
  - Update all 4 copies when source changes
  - Manage separate lifecycles
  - Risk of inconsistency


┌────────────────────────────────────────────────────────────┐
│          Object Lambda Approach (On-Demand Transform)      │
└────────────────────────────────────────────────────────────┘

Source Data: customer-data.json (100 MB)
  │
  │  Request via Object Lambda Access Point
  │  (Redacted version)
  ▼
┌─────────────────────────┐
│  Lambda Function        │
│  (Redaction Logic)      │
│  - Read original object │
│  - Remove PII fields    │
│  - Return redacted data │
└─────────────────────────┘
  │
  │  Transformed object returned
  ▼
User receives: customer-data.json (redacted, no PII)

Storage: Only 100 MB (original)
Transform: On-demand, per request
Consistency: Always uses latest source
```

**Key principle**: The Lambda function runs during the GET request, transforms the object in-flight, and returns the modified version. The original object is never changed.

---

#### How Object Lambda Works

**Request flow:**

```
Step 1: User makes GET request to Object Lambda Access Point
  GET https://my-redaction-ap-123456.s3-object-lambda.us-east-1.amazonaws.com/data.json

Step 2: Object Lambda Access Point triggers Lambda function
  Event payload includes:
    - GetObjectContext (used to return response)
    - UserRequest (original request details)
    - Configuration (Access Point settings)

Step 3: Lambda function runs
  const AWS = require('aws-sdk');
  const s3 = new AWS.S3();

  exports.handler = async (event) => {
    const s3Url = event.getObjectContext.inputS3Url;  // Presigned URL to original

    // Fetch original object from S3
    const original = await fetch(s3Url);
    const data = await original.json();

    // Transform: Remove PII
    const redacted = {
      ...data,
      ssn: undefined,          // Remove SSN
      email: 'REDACTED',       // Mask email
      phone: 'REDACTED'        // Mask phone
    };

    // Return transformed object
    await s3.writeGetObjectResponse({
      RequestRoute: event.getObjectContext.outputRoute,
      RequestToken: event.getObjectContext.outputToken,
      Body: JSON.stringify(redacted)  // Transformed data
    }).promise();
  };

Step 4: Transformed object returned to user
  User receives: Redacted version (PII removed)
  S3 original: Unchanged (still has PII)
```

---

#### Object Lambda Access Points

Object Lambda requires a **two-tier Access Point structure**:

```
┌────────────────────────────────────────────────────────────┐
│          Object Lambda Access Point Structure              │
└────────────────────────────────────────────────────────────┘

┌─────────────────────────────────┐
│ Object Lambda Access Point      │  ← User requests here
│ - Invokes Lambda function       │
│ - Passes context to function    │
└────────────┬────────────────────┘
             │ (Supporting Access Point)
             ▼
┌─────────────────────────────────┐
│ Standard S3 Access Point        │  ← Lambda fetches original here
│ - Points to source bucket       │
│ - Standard permissions          │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│ Source S3 Bucket                │  ← Original data stored here
│ - Unchanged by Lambda           │
└─────────────────────────────────┘

Configuration:
  1. Create S3 bucket: customer-data
  2. Create supporting Access Point: customer-data-ap
  3. Create Object Lambda Access Point: customer-data-redacted-olap
     - Supporting Access Point: customer-data-ap
     - Lambda function: redaction-function
```

---

#### Common Use Cases

**1. PII Redaction**

```
Scenario: Analytics team needs customer data WITHOUT PII

Original object:
{
  "id": "12345",
  "name": "John Doe",
  "ssn": "123-45-6789",      ← Sensitive
  "email": "john@email.com",  ← Sensitive
  "purchase_amount": 99.99
}

Lambda transformation:
{
  "id": "12345",
  "name": "REDACTED",
  "ssn": "REDACTED",
  "email": "REDACTED",
  "purchase_amount": 99.99    ← Keep for analytics
}

Benefit:
  - Analytics team gets safe data
  - Original PII preserved for authorized teams
  - No duplicate storage
```

---

**2. Image Resizing**

```
Scenario: Mobile app needs thumbnails, web app needs full size

Original: photo.jpg (5 MB, 4000×3000)

Mobile request:
  GET via thumbnail-olap → Lambda resizes to 400×300 → Returns 50 KB thumbnail

Web request:
  GET via full-size-olap → Lambda passes through → Returns 5 MB original

Benefit:
  - No pre-generated thumbnails stored
  - On-demand sizing (any dimension requested)
  - Storage: Only 5 MB original
```

---

**3. Format Conversion**

```
Scenario: Legacy app needs XML, modern app needs JSON

Original: data.json (JSON format)

Legacy app request:
  GET via xml-olap → Lambda converts JSON→XML → Returns XML

Modern app request:
  GET via source-ap → Returns JSON directly (no transformation)

Benefit:
  - Support multiple formats from single source
  - No duplicate storage for XML copy
```

---

**4. Data Enrichment**

```
Scenario: Add metadata from external API to stored objects

Original object:
{
  "product_id": "ABC123",
  "price": 29.99
}

Lambda enrichment:
  - Fetch product details from external API
  - Merge with S3 object
  - Return enriched response

Returned object:
{
  "product_id": "ABC123",
  "price": 29.99,
  "product_name": "Widget",        ← From API
  "category": "Electronics",       ← From API
  "in_stock": true                 ← From API
}

Benefit:
  - S3 object stays small (static data only)
  - Dynamic data added at request time
  - Always fresh external data
```

---

**5. Compression/Decompression**

```
Scenario: Store compressed, serve decompressed

Original: logs.json.gz (compressed, 1 MB)

User request:
  GET via decompression-olap → Lambda decompresses → Returns logs.json (10 MB)

Benefit:
  - Storage: 1 MB (compressed)
  - User receives: 10 MB (decompressed, ready to use)
  - 90% storage savings
```

---

#### Lambda Function Payload

The Lambda function receives a detailed event:

```javascript
{
  "xAmzRequestId": "request-id",
  "getObjectContext": {
    "inputS3Url": "https://presigned-url-to-original",  // Presigned URL to original object
    "outputRoute": "route-to-return-data",
    "outputToken": "token-for-response"
  },
  "configuration": {
    "accessPointArn": "arn:...",
    "supportingAccessPointArn": "arn:...",
    "payload": "{...}"  // Optional custom payload from Access Point config
  },
  "userRequest": {
    "url": "https://original-request-url",
    "headers": {
      "Host": "...",
      "Range": "bytes=0-1023"  // If range request
    }
  },
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "...",
    "arn": "arn:aws:iam::..."
  },
  "protocolVersion": "1.00"
}
```

**Important fields**:
- `inputS3Url`: Presigned URL to fetch the original object
- `outputRoute` + `outputToken`: Required to return the transformed object
- `userRequest`: Details of the original request (headers, range)

---

#### Performance Considerations

**Latency impact:**

```
Without Object Lambda:
  S3 GET request: ~10-50 ms

With Object Lambda:
  S3 GET request: ~10-50 ms
  + Lambda cold start: ~100-3000 ms (first invocation)
  + Lambda execution: ~50-500 ms (transformation logic)
  Total: ~160-3550 ms

Optimization:
  - Provisioned Concurrency (eliminates cold start): +$0.015/hr per provisioned instance
  - Optimize Lambda function (smaller package, faster code)
  - Cache transformed results in CloudFront (for read-heavy workloads)
```

**Cost considerations:**

```
Costs:
  1. Lambda invocations: $0.20 per 1M requests
  2. Lambda duration: $0.0000166667 per GB-second
  3. S3 GET requests: $0.0004 per 1,000 requests (to fetch original)
  4. S3 data transfer: $0.09 per GB (from S3 to Lambda, within region = free)

Example:
  1M requests/month
  Transform: 500ms @ 512MB memory
  Object size: 100 KB

  Lambda invocations: $0.20
  Lambda duration: 1M × 0.5s × 0.5GB × $0.0000166667 = $4.17
  S3 GET (original): $0.40
  Total: $4.77/month

Compare to storing 2 copies:
  Original + Transformed: 200 KB × 1M objects = 200 GB
  Storage: 200 GB × $0.023 = $4.60/month

Object Lambda cheaper if: Low request rate, high transformation complexity
Pre-generated copies cheaper if: High request rate, simple transformation
```

---

#### Limitations

**Function constraints:**
- Lambda timeout: 15 minutes max (for large object processing)
- Memory: 128 MB - 10 GB
- Object size: No hard limit, but large objects = longer processing

**Feature limitations:**
- Cannot use with S3 Select or Glacier Select
- Cannot use with S3 Batch Operations
- Only supports GET requests (no PUT, POST, DELETE transformations)
- Range requests supported but complex (Lambda must handle byte ranges)

**Best practices:**
- Keep transformations lightweight (sub-second)
- Use Provisioned Concurrency for latency-sensitive apps
- Monitor Lambda errors (failed transformations)
- Consider caching transformed results in CloudFront
- Test with maximum expected object size

---

### S3 Event Notifications

**What are S3 Event Notifications?**

Automated messages sent to other AWS services when specific **events occur in an S3 bucket** (object created, deleted, restored, etc.). Think of it as "webhooks" for S3.

**Why they exist:**
- Applications need to react to S3 changes in real-time
- Polling S3 for changes is inefficient and expensive
- Event-driven architectures require decoupling
- Processing pipelines need automatic triggering

**Problems they solve:**
- **Real-time processing**: Trigger workflows immediately when files arrive
- **Automation**: No manual intervention or polling needed
- **Decoupling**: S3 doesn't know about downstream systems
- **Scalability**: Handles millions of events automatically
- **Cost efficiency**: No constant polling costs

---

#### Event Notification Architecture

```
┌────────────────────────────────────────────────────────────┐
│              S3 Event Notification Flow                    │
└────────────────────────────────────────────────────────────┘

   S3 Bucket: uploads
        │
        │  User uploads: photo.jpg
        ▼
   Event: s3:ObjectCreated:Put
        │
        │  S3 publishes event to configured targets
        │
        ├──────────┬──────────┬──────────┐
        │          │          │          │
        ▼          ▼          ▼          ▼
    Lambda      SNS Topic  SQS Queue  EventBridge
    Function    (Email)    (Queue)    (Advanced routing)
        │          │          │          │
        ▼          ▼          ▼          ▼
   Resize     Send email  Process   Trigger Step
   image      to admin    async     Functions workflow
```

**Supported destinations:**
1. **Lambda**: Invoke function directly
2. **SNS**: Publish to topic (fan-out to multiple subscribers)
3. **SQS**: Send to queue (decoupled processing)
4. **EventBridge**: Route to multiple targets with advanced filtering

---

#### Event Types

**Object creation events:**
```
s3:ObjectCreated:*          All creation events (wildcard)
s3:ObjectCreated:Put        PUT, POST, or COPY operations
s3:ObjectCreated:Post       POST operation
s3:ObjectCreated:Copy       COPY operation
s3:ObjectCreated:CompleteMultipartUpload  Multipart upload completion
```

**Object deletion events:**
```
s3:ObjectRemoved:*          All deletion events
s3:ObjectRemoved:Delete     DELETE operation (permanent)
s3:ObjectRemoved:DeleteMarkerCreated  Delete marker created (versioned bucket)
```

**Object restoration events:**
```
s3:ObjectRestore:Post       Glacier restore initiated
s3:ObjectRestore:Completed  Glacier restore completed
s3:ObjectRestore:Delete     Temporary restored copy expired
```

**Other events:**
```
s3:ReducedRedundancyLostObject  Object in RRS lost
s3:Replication:*                Replication events
s3:LifecycleTransition:*        Lifecycle transition events
s3:LifecycleExpiration:*        Lifecycle expiration events
s3:IntelligentTiering:*         Intelligent-Tiering events
s3:ObjectTagging:*              Tagging events
s3:ObjectAcl:Put                ACL modified
```

---

#### Event Notification Configuration

**Example: Trigger Lambda on new uploads**

```
Bucket: image-uploads
Event Configuration:
  Name: process-new-images
  Events: s3:ObjectCreated:*
  Prefix: uploads/
  Suffix: .jpg
  Destination: Lambda function (image-processor)

Result:
  ✅ Upload: uploads/photo.jpg → Lambda triggered
  ✅ Upload: uploads/2026/vacation.jpg → Lambda triggered (prefix matches)
  ❌ Upload: uploads/document.pdf → NOT triggered (suffix doesn't match)
  ❌ Upload: downloads/photo.jpg → NOT triggered (prefix doesn't match)
```

**Filtering:**
- **Prefix**: Only trigger for objects with specific prefix (folder path)
- **Suffix**: Only trigger for specific file extensions
- **Both**: Combine for precise filtering

---

#### Event Message Format

**Lambda receives:**

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2026-02-06T12:34:56.789Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "AWS:AIDAI..."
      },
      "requestParameters": {
        "sourceIPAddress": "203.0.113.1"
      },
      "responseElements": {
        "x-amz-request-id": "ABC123",
        "x-amz-id-2": "DEF456..."
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "process-new-images",
        "bucket": {
          "name": "image-uploads",
          "ownerIdentity": {
            "principalId": "A1B2C3..."
          },
          "arn": "arn:aws:s3:::image-uploads"
        },
        "object": {
          "key": "uploads/photo.jpg",    // ⚠️ URL-encoded
          "size": 1048576,               // Bytes
          "eTag": "abc123...",
          "versionId": "v123",           // If versioning enabled
          "sequencer": "005C..."         // Ordering token
        }
      }
    }
  ]
}
```

**Important**: `object.key` is **URL-encoded**. Decode before using:
```javascript
// Object key: "uploads/my file.jpg" (with space)
// Event contains: "uploads/my%20file.jpg"

const objectKey = decodeURIComponent(event.Records[0].s3.object.key);
// Result: "uploads/my file.jpg"
```

---

#### Common Event Notification Patterns

**Pattern 1: Image Processing Pipeline**

```
User uploads image to S3
  │
  ▼
S3 Event → Lambda (resize-function)
  │
  ├─ Download original from S3
  ├─ Generate thumbnail
  ├─ Upload thumbnail to S3 (different prefix)
  └─ Update database with thumbnail path

Avoiding infinite loops:
  ⚠️ Problem: Upload thumbnail → Triggers same event → Infinite loop

  ✅ Solution: Different prefixes
     Event filter: Prefix = "uploads/"
     Thumbnails go to: "thumbnails/"
     Event NOT triggered for "thumbnails/" prefix
```

---

**Pattern 2: Data Lake ETL**

```
Raw data arrives in S3 data lake
  │
  ▼
S3 Event → SQS Queue → Lambda (ETL processor)
  │
  ├─ Read raw data
  ├─ Transform, clean, enrich
  └─ Write to processed/ prefix

Why SQS?
  - Decouples upload from processing
  - Handles bursts (1000 uploads → queue buffers)
  - Retry failed processing automatically
  - Lambda polls queue at controlled rate
```

---

**Pattern 3: Fan-out Notifications**

```
Document uploaded to S3
  │
  ▼
S3 Event → SNS Topic
  │
  ├─── Subscription 1: Email to admins
  ├─── Subscription 2: Lambda (virus scan)
  ├─── Subscription 3: SQS (indexing queue)
  └─── Subscription 4: HTTP endpoint (external system)

Result: Single upload triggers multiple independent workflows
```

---

**Pattern 4: Cross-Account Event Handling**

```
Account A (Bucket owner)
  Bucket: shared-data
  Event: s3:ObjectCreated:*
  Destination: SNS Topic in Account B

Account B (Processor)
  SNS Topic: data-processing-topic
  Subscription: Lambda function

Configuration:
  1. Account A bucket policy: Allow SNS to receive events
  2. Account B SNS policy: Allow Account A S3 to publish
  3. SNS subscription: Lambda in Account B
```

---

#### EventBridge Integration

**S3 + EventBridge** provides more advanced capabilities than basic event notifications:

```
┌────────────────────────────────────────────────────────────┐
│     Basic S3 Events      │    EventBridge Integration      │
├──────────────────────────┼──────────────────────────────────┤
│ 3 destinations max       │ Multiple targets per rule       │
│ (Lambda, SNS, SQS)       │ (20+ AWS services)              │
├──────────────────────────┼──────────────────────────────────┤
│ Simple prefix/suffix     │ Advanced content filtering      │
│ filtering                │ (metadata, tags, size)          │
├──────────────────────────┼──────────────────────────────────┤
│ No transformation        │ Input transformation            │
│                          │ (modify event before target)    │
├──────────────────────────┼──────────────────────────────────┤
│ Immediate delivery       │ Can schedule/delay              │
├──────────────────────────┼──────────────────────────────────┤
│ Manual configuration     │ Archive, replay events          │
└────────────────────────────────────────────────────────────┘
```

**EventBridge example:**

```
EventBridge Rule:
  Event pattern:
    {
      "source": ["aws.s3"],
      "detail-type": ["Object Created"],
      "detail": {
        "bucket": { "name": ["my-bucket"] },
        "object": {
          "size": [{ "numeric": [">", 1048576] }],  // Files > 1 MB
          "key": [{ "prefix": "uploads/" }]
        }
      }
    }

  Targets:
    1. Lambda: process-large-files
    2. Step Functions: video-encoding-workflow
    3. SNS: admin-alerts
    4. CloudWatch Logs: audit-log-group
```

---

#### Limitations & Considerations

**Rate limits:**
- S3 event notifications have no explicit rate limit
- Destination services have limits:
  - Lambda: 1000 concurrent executions (soft limit)
  - SQS: 3000 messages/second (FIFO), unlimited (Standard)
  - SNS: 30,000 messages/second (soft limit)

**Event delivery:**
- **Best-effort delivery**: Events typically delivered in seconds
- **No guaranteed ordering**: Events may arrive out of order
- **Possible duplicates**: Same event may be delivered more than once
- **No retry by S3**: If destination is unavailable, event may be lost

**Best practices for reliability:**

```
Idempotency:
  - Lambda functions must handle duplicate events
  - Use DynamoDB or database to track processed events

Example:
  const processedEvents = new Set();  // Or DynamoDB

  if (processedEvents.has(event.s3.object.sequencer)) {
    console.log('Duplicate event, skipping');
    return;
  }

  // Process event
  // ...

  processedEvents.add(event.s3.object.sequencer);
```

**Avoiding infinite loops:**
- Never write to the same prefix that triggers the event
- Use different buckets for source and destination
- Use prefix filters carefully

**Cost:**
- Event notifications themselves: **FREE**
- Destination costs apply:
  - Lambda invocations
  - SQS/SNS message charges
  - EventBridge events

---

### S3 Inventory

**What is S3 Inventory?**

A scheduled **report of all objects** in a bucket, delivered as CSV, ORC, or Parquet files, providing metadata about every object.

**Why it exists:**
- Listing millions of objects via API is slow and expensive
- Need periodic audits of bucket contents
- Analytics on storage patterns, costs, compliance
- Lifecycle policy validation

**Problems it solves:**
- **Bulk listing**: Report on billions of objects efficiently
- **Cost analysis**: Identify storage costs by prefix, storage class
- **Compliance**: Audit encryption, versioning, retention status
- **Lifecycle validation**: Verify objects transitioned as expected
- **Storage optimization**: Find old, large, or unnecessary objects

---

#### Inventory Report Configuration

```
Configuration:
  Source bucket: my-data-bucket
  Destination bucket: my-inventory-reports
  Frequency: Daily or Weekly
  Format: CSV, ORC, or Parquet

  Include:
    ✅ Object size
    ✅ Last modified date
    ✅ Storage class
    ✅ Encryption status
    ✅ Replication status
    ✅ E-Tag
    ✅ Object Lock retention
    ✅ Versioning
    Optional: Prefix filter (only report specific paths)
```

**Report delivery:**
```
Destination bucket structure:
  my-inventory-reports/
    ├─ my-data-bucket/
    │   ├─ inventory-config-name/
    │   │   ├─ 2026-02-06T00-00Z/
    │   │   │   ├─ manifest.json
    │   │   │   ├─ manifest.checksum
    │   │   │   └─ data/
    │   │   │       ├─ part-00000.csv.gz
    │   │   │       ├─ part-00001.csv.gz
    │   │   │       └─ part-00002.csv.gz
```

**manifest.json** contains metadata about the inventory report (file locations, count, size).