# AWS Storage Services - Complete Conceptual Guide (Part 3)

## EBS (Elastic Block Store)

### What is EBS?

**Amazon EBS** is a **block-level storage service** that provides persistent storage volumes for EC2 instances. Think of it as a **virtual hard drive** that can be attached to EC2 instances.

**Why it exists:**
- EC2 instance store is ephemeral (data lost on stop/termination)
- Need persistent storage that survives instance lifecycle
- Different workloads need different performance characteristics
- Applications require block-level access (filesystems, databases)

**Problems it solves:**
- **Data persistence**: Data survives instance stop/restart/termination
- **Performance flexibility**: Choose IOPS and throughput based on workload
- **Snapshots/backups**: Point-in-time backups to S3
- **Resizing**: Grow volumes without downtime
- **Encryption**: Built-in encryption at rest and in transit

---

### Block Storage vs Object Storage

```
┌────────────────────────────────────────────────────────────┐
│     Block Storage (EBS)  │   Object Storage (S3)           │
├──────────────────────────┼─────────────────────────────────┤
│ Raw blocks, no structure │ Objects with metadata, keys     │
├──────────────────────────┼─────────────────────────────────┤
│ Requires filesystem      │ No filesystem needed            │
│ (ext4, XFS, NTFS)        │                                 │
├──────────────────────────┼─────────────────────────────────┤
│ Attached to EC2 instance │ Accessed via HTTP API           │
├──────────────────────────┼─────────────────────────────────┤
│ Low latency (<1ms)       │ Higher latency (~10-100ms)      │
├──────────────────────────┼─────────────────────────────────┤
│ Limited size per volume  │ Unlimited size                  │
│ (up to 64 TB)            │                                 │
├──────────────────────────┼─────────────────────────────────┤
│ Good for: OS drives,     │ Good for: Backups, static files,│
│ databases, apps needing  │ media, data lakes               │
│ filesystem               │                                 │
└──────────────────────────────────────────────────────────────┘
```

---

### EBS Architecture

```
┌────────────────────────────────────────────────────────────┐
│                 EBS Volume Architecture                    │
└────────────────────────────────────────────────────────────┘

    Availability Zone: us-east-1a
   ┌─────────────────────────────────────────┐
   │                                         │
   │  ┌──────────────┐    Network           │
   │  │ EC2 Instance │◄────────────────┐    │
   │  │              │                 │    │
   │  │ /dev/xvdf ───┼─────────────────┤    │
   │  └──────────────┘                 │    │
   │                                   │    │
   │                           ┌───────▼────┐│
   │                           │ EBS Volume ││
   │                           │ (100 GB)   ││
   │                           │ gp3        ││
   │                           └────────────┘│
   │                                         │
   │  Replicated within AZ (multiple drives)│
   │  99.999% availability                  │
   └─────────────────────────────────────────┘

❌ Cannot attach: Volume to instance in different AZ
✅ Can attach: Volume to instance in same AZ
```

**Key characteristics:**
- **Single AZ**: Volume exists in one AZ only
- **Replicated within AZ**: Stored on multiple physical drives
- **Network-attached**: Connected via AWS network (not physical connection)
- **Persistent**: Data survives instance stop/termination (unless delete-on-termination enabled)

---

### Volume Types

EBS offers **6 volume types** optimized for different use cases:

```
┌────────────────────────────────────────────────────────────┐
│              EBS Volume Type Spectrum                      │
└────────────────────────────────────────────────────────────┘

SSD-based (Latency-sensitive):
  gp3      gp2           io2 Block Express    io2      io1
  │        │             │                    │        │
  ▼        ▼             ▼                    ▼        ▼
General  General     Highest IOPS         High IOPS  High IOPS
Purpose  Purpose     (ultra-performance)  (critical) (critical)
Balanced Balanced
$0.08/GB $0.10/GB    $0.125/GB           $0.125/GB  $0.125/GB
3K-16K   3K-16K      256K IOPS           64K IOPS   64K IOPS
IOPS     IOPS

HDD-based (Throughput-optimized):
  st1               sc1
  │                 │
  ▼                 ▼
Throughput        Cold HDD
Optimized         (Infrequent)
$0.045/GB         $0.015/GB
500 MB/s          250 MB/s
```

---

#### 1. gp3 (General Purpose SSD) - Recommended Default

**What it is**: Latest-generation SSD optimized for most workloads.

**Performance:**
```
IOPS: 3,000 - 16,000 (independent of volume size)
Throughput: 125 - 1,000 MB/s (independent of volume size)
Size: 1 GB - 16 TB
Latency: Single-digit milliseconds

Baseline (included in price):
  - 3,000 IOPS
  - 125 MB/s throughput

Additional cost:
  - Extra IOPS: $0.005 per IOPS-month (above 3,000)
  - Extra throughput: $0.04 per MB/s-month (above 125 MB/s)
```

**Use cases:**
- Boot volumes
- Virtual desktops
- Development/test environments
- Low-latency interactive applications
- MySQL, PostgreSQL databases (moderate workload)

**Example configuration:**

```
Volume: 500 GB gp3
  - Storage: 500 GB × $0.08 = $40/month
  - Baseline: 3,000 IOPS, 125 MB/s (free)

Need more performance?
  - Add 5,000 IOPS: 5,000 × $0.005 = $25/month
  - Add 375 MB/s: 375 × $0.04 = $15/month
  Total: $40 + $25 + $15 = $80/month

Result: 500 GB, 8,000 IOPS, 500 MB/s
```

---

#### 2. gp2 (General Purpose SSD) - Legacy

**What it is**: Previous-generation SSD (before gp3).

**Performance:**
```
IOPS: Scales with volume size (3 IOPS per GB)
  - Minimum: 100 IOPS (even for tiny volumes)
  - Maximum: 16,000 IOPS (at 5,334 GB)

Throughput: Up to 250 MB/s (volumes ≥334 GB)
Size: 1 GB - 16 TB
Latency: Single-digit milliseconds

IOPS calculation:
  Volume size × 3 IOPS/GB
  Example: 100 GB → 300 IOPS
           1,000 GB → 3,000 IOPS
           5,334 GB → 16,000 IOPS (max)
```

**Problem with gp2**: IOPS tied to size
```
Scenario: Need 10,000 IOPS

gp2 approach:
  - Need: 10,000 IOPS ÷ 3 = 3,334 GB volume
  - Cost: 3,334 GB × $0.10 = $333.40/month
  - Problem: Paying for unused storage to get IOPS

gp3 approach:
  - Need: Any size volume + 7,000 extra IOPS
  - Cost: 100 GB × $0.08 + 7,000 × $0.005 = $8 + $35 = $43/month
  - Savings: $290.40/month (87% cheaper)
```

**Recommendation**: Use gp3 instead (cheaper, more flexible). gp2 only for legacy workloads.

---

#### 3. io2 Block Express (Highest Performance SSD)

**What it is**: Ultra-high-performance SSD for the most demanding workloads.

**Performance:**
```
IOPS: Up to 256,000 IOPS per volume
Throughput: Up to 4,000 MB/s
Size: 4 GB - 64 TB
Latency: Sub-millisecond (microseconds)
Durability: 99.999% (5 nines) - Best in class

IOPS/GB ratio: Up to 1,000:1
  Example: 64 GB volume → 64,000 IOPS possible
```

**Use cases:**
- SAP HANA
- Large relational databases (Oracle, SQL Server)
- NoSQL databases (Cassandra, MongoDB) at scale
- Mission-critical applications requiring highest performance

**Pricing:**
```
Storage: $0.125/GB-month
IOPS: $0.065 per IOPS-month

Example: 1 TB volume, 100,000 IOPS
  Storage: 1,000 GB × $0.125 = $125
  IOPS: 100,000 × $0.065 = $6,500
  Total: $6,625/month
```

**Expensive but necessary for:**
- Sub-millisecond latency requirements
- Extremely high IOPS workloads
- Mission-critical databases

---

#### 4. io2 (Provisioned IOPS SSD)

**What it is**: High-performance SSD for I/O-intensive workloads (predecessor to io2 Block Express).

**Performance:**
```
IOPS: Up to 64,000 IOPS per volume
Throughput: Up to 1,000 MB/s
Size: 4 GB - 16 TB
Latency: Single-digit milliseconds
Durability: 99.999% (5 nines)

IOPS/GB ratio: Up to 500:1
  Example: 100 GB volume → 50,000 IOPS max
```

**Use cases:**
- Production databases (moderate to high workload)
- I/O-intensive applications
- When io2 Block Express not needed (cost savings)

**Pricing:**
```
Storage: $0.125/GB-month
IOPS: $0.065 per IOPS-month (same as io2 Block Express)
```

---

#### 5. io1 (Provisioned IOPS SSD) - Legacy

**What it is**: Original provisioned IOPS SSD (superseded by io2).

**Performance:**
```
IOPS: Up to 64,000 IOPS
Throughput: Up to 1,000 MB/s
Durability: 99.8 - 99.9% (lower than io2)
```

**Recommendation**: Use io2 instead (same price, better durability).

---

#### 6. st1 (Throughput Optimized HDD)

**What it is**: Low-cost HDD optimized for **throughput** (MB/s), not IOPS.

**Performance:**
```
Throughput: 500 MB/s max
IOPS: Up to 500 IOPS (much lower than SSD)
Size: 125 GB - 16 TB
Cost: $0.045/GB-month

Baseline throughput: 40 MB/s per TB
Burst throughput: 250 MB/s per TB
```

**Use cases:**
- Big data processing (Hadoop, MapReduce)
- Log processing (large sequential writes)
- Data warehouses
- ETL workloads (sequential throughput matters, latency doesn't)

**Not suitable for:**
- Boot volumes (too slow)
- Databases (low IOPS)
- Random I/O workloads

**Example:**
```
Workload: Process 5 TB log files daily (sequential read)

st1 (Throughput Optimized):
  - Volume: 5 TB × $0.045 = $225/month
  - Throughput: 5 TB × 40 MB/s = 200 MB/s baseline
  - Time to read: 5 TB ÷ 200 MB/s ≈ 7 hours

gp3 (for comparison):
  - Volume: 5 TB × $0.08 = $400/month
  - Throughput: 125 MB/s baseline
  - Time to read: 5 TB ÷ 125 MB/s ≈ 11 hours
  - Cost: 78% more expensive

st1 wins: Cheaper, faster for sequential throughput
```

---

#### 7. sc1 (Cold HDD)

**What it is**: Lowest-cost HDD for infrequently accessed data.

**Performance:**
```
Throughput: 250 MB/s max
IOPS: Up to 250 IOPS
Size: 125 GB - 16 TB
Cost: $0.015/GB-month (cheapest EBS option)

Baseline throughput: 12 MB/s per TB
Burst throughput: 80 MB/s per TB
```

**Use cases:**
- Infrequently accessed data
- Archive storage (accessible, not Glacier-slow)
- File servers with cold data
- Secondary backups

**Not suitable for:**
- Anything requiring performance
- Frequently accessed data

**Example:**
```
Scenario: Store 10 TB of archived logs (accessed quarterly)

sc1:
  - Cost: 10 TB × $0.015 = $150/month
  - Performance: Acceptable for infrequent access

S3 Glacier (alternative):
  - Cost: 10 TB × $0.004 = $40/month
  - Retrieval time: 3-5 hours

Decision:
  - sc1 if need instant access
  - Glacier if retrieval delay acceptable (cheaper)
```

---

### Volume Type Decision Tree

```
                Start: What's your workload?
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
    Throughput-heavy   IOPS-heavy      General-purpose
    (sequential)       (random I/O)    (balanced)
          │                 │                 │
          ▼                 ▼                 ▼
    Frequently        How many IOPS?     Use gp3
    accessed?               │            (default choice)
          │            ┌────┼────┐
      ┌───┴───┐        │    │    │
      │       │      <5K  5K-  >64K
     Yes      No           16K
      │       │        │    │    │
      ▼       ▼        ▼    ▼    ▼
     st1     sc1      gp3  io2  io2 Block
                                  Express
```

---

### IOPS & Throughput

**What are IOPS?**

**IOPS (Input/Output Operations Per Second)**: Number of read/write operations per second.

```
IOPS measures: How many operations
Throughput measures: How much data

Example:
  1,000 IOPS × 4 KB per operation = 4 MB/s throughput
  1,000 IOPS × 256 KB per operation = 256 MB/s throughput

Different workloads:
  - Database: Small random I/O (4-16 KB) → IOPS matters
  - Video streaming: Large sequential I/O (1 MB+) → Throughput matters
```

---

**IOPS calculation:**

```
Block size × IOPS = Throughput

gp3 example:
  3,000 IOPS baseline
  Block size: 16 KB (typical database)
  Throughput: 3,000 × 16 KB = 48 MB/s

  To max out 125 MB/s baseline throughput:
    Need: 125 MB/s ÷ 16 KB = 8,000 IOPS
    Solution: Provision 8,000 IOPS (5,000 extra)
```

---

**Throughput limits by volume type:**

```
Volume Type    Max Throughput   Max IOPS
gp3            1,000 MB/s       16,000
gp2            250 MB/s         16,000
io2 Block Exp  4,000 MB/s       256,000
io2            1,000 MB/s       64,000
io1            1,000 MB/s       64,000
st1            500 MB/s         500
sc1            250 MB/s         250
```

---

### EBS Snapshots

**What are EBS Snapshots?**

**Point-in-time backups** of EBS volumes stored in S3 (managed by AWS, not accessible as S3 objects).

**Why they exist:**
- Need backups for disaster recovery
- Want to copy volumes across AZs/regions
- Create new volumes from existing data
- Restore to previous state

**Problems they solve:**
- **Disaster recovery**: Restore volume if failure occurs
- **Data migration**: Move volumes across AZs/regions
- **Volume cloning**: Create copies for testing
- **Cost savings**: Incremental backups (only changed blocks)

---

#### How Snapshots Work

```
┌────────────────────────────────────────────────────────────┐
│              Snapshot Incremental Backup                   │
└────────────────────────────────────────────────────────────┘

Day 0: Create 100 GB volume
  - Data: 100 GB

Day 1: Take snapshot (Snapshot-1)
  - Stores: 100 GB (full backup)
  - Cost: 100 GB × $0.05 = $5/month

Day 5: Modify 10 GB of data
  - Volume: Still 100 GB

Day 6: Take snapshot (Snapshot-2)
  - Stores: Only 10 GB changed (incremental)
  - Total stored: Snapshot-1 (100 GB) + Snapshot-2 (10 GB delta)
  - Cost: 110 GB × $0.05 = $5.50/month

Day 10: Modify 5 GB

Day 11: Take snapshot (Snapshot-3)
  - Stores: Only 5 GB changed
  - Total stored: 100 + 10 + 5 = 115 GB
  - Cost: 115 GB × $0.05 = $5.75/month

Delete Snapshot-1:
  - AWS automatically consolidates Snapshot-2 to become full
  - Total stored: 115 GB (no change)
  - Billing: Only for 115 GB
```

**Key insight**: Snapshots are incremental, but AWS manages dependencies. Deleting old snapshots doesn't break newer ones.

---

#### Snapshot Operations

**Create snapshot:**
```
Volume: /dev/xvdf (100 GB, 60 GB used)

Snapshot creation:
  1. Initiate snapshot
  2. EBS takes instant point-in-time copy
  3. Snapshot available immediately (lazy loading)
  4. Background process copies data to S3

Snapshot stores: Only used blocks (60 GB), not full 100 GB
Cost: 60 GB × $0.05 = $3/month
```

**Restore from snapshot:**
```
Create new volume from Snapshot-1:
  1. Select snapshot
  2. Choose volume type (can differ from original)
     Original: gp2
     New: gp3 (upgrade during restore)
  3. Choose AZ (can differ from original)
     Original: us-east-1a
     New: us-east-1b (migration)
  4. Volume created and ready
```

**Copy snapshot across regions:**
```
Disaster recovery setup:
  1. Create snapshot in us-east-1
  2. Copy snapshot to eu-west-1
  3. If us-east-1 fails, restore in eu-west-1

Cost: Pay for storage in both regions
```

---

#### Fast Snapshot Restore (FSR)

**Problem**: Volumes restored from snapshots have high latency initially (lazy loading from S3).

```
Traditional snapshot restore:
  Create volume from snapshot
    → Volume available immediately
    → But blocks loaded from S3 on first access
    → First read of each block: High latency (S3 fetch)
    → Subsequent reads: Normal latency (cached)

  Result: "Warming up" period with poor performance
```

**Fast Snapshot Restore (FSR) solution:**

```
Enable FSR on snapshot:
  - Snapshot data pre-loaded into EBS
  - Volumes created from FSR snapshot: Full performance immediately
  - No warm-up period

Cost: $0.75 per snapshot per AZ per hour
  Example: 1 snapshot, 2 AZs, 24 hours
           1 × 2 × 24 × $0.75 = $36/day

Use case:
  - Production database restore (need full performance now)
  - Critical workloads (can't wait for warm-up)
  - DR scenarios requiring immediate performance
```

---

### EBS Multi-Attach (io1/io2)

**What is Multi-Attach?**

Feature allowing **multiple EC2 instances** to attach to the **same io1/io2 volume** simultaneously for read/write access.

**Why it exists:**
- Some clustered applications need shared block storage
- High availability requires multiple instances accessing same data
- Legacy applications designed for SAN (Storage Area Network)

**Problems it solves:**
- **Shared storage**: Multiple instances access same volume
- **High availability**: Failover between instances
- **Clustered applications**: Shared lock management

---

#### How Multi-Attach Works

```
┌────────────────────────────────────────────────────────────┐
│                  EBS Multi-Attach                          │
└────────────────────────────────────────────────────────────┘

       Availability Zone: us-east-1a
      ┌──────────────────────────────────────┐
      │                                      │
      │  ┌──────────────┐                   │
      │  │ EC2 Instance │                   │
      │  │     #1       │                   │
      │  └──────┬───────┘                   │
      │         │                           │
      │         └───────────┐               │
      │                     ▼               │
      │              ┌─────────────┐        │
      │              │ io2 Volume  │        │
      │              │ Multi-Attach│        │
      │              │ Enabled     │        │
      │              └─────────────┘        │
      │                     ▲               │
      │         ┌───────────┘               │
      │         │                           │
      │  ┌──────┴───────┐                   │
      │  │ EC2 Instance │                   │
      │  │     #2       │                   │
      │  └──────────────┘                   │
      │                                      │
      └──────────────────────────────────────┘

Both instances read/write same volume simultaneously
```

---

#### Limitations and Requirements

**Requirements:**
- Volume type: io1 or io2 only (not gp3, gp2, or HDD)
- Same AZ: All instances must be in same AZ
- Max instances: 16 instances per volume
- Cluster-aware filesystem: Application must handle concurrent access

**Application requirements:**
```
⚠️ Multi-Attach does NOT provide:
  - Automatic locking
  - Conflict resolution
  - Data consistency (application's responsibility)

Application MUST:
  - Use cluster-aware filesystem (GFS2, OCFS2)
  - OR implement own locking mechanism
  - Handle concurrent writes

Standard filesystems (ext4, XFS) will CORRUPT data with Multi-Attach
```

---

#### Use Cases

**1. High Availability Clustered Applications**

```
Oracle RAC (Real Application Clusters):
  - Multiple database instances
  - Shared storage requirement
  - Built-in lock management

Setup:
  Instance 1 → io2 volume (Multi-Attach)
  Instance 2 → Same io2 volume

If Instance 1 fails:
  Instance 2 continues with no data loss
```

---

**2. Linux Cluster (High Availability)**

```
Pacemaker/Corosync cluster:
  - Shared configuration/data
  - Automatic failover
  - Cluster-aware filesystem (GFS2)
```

---

**When NOT to use Multi-Attach:**

```
❌ General file sharing:
   Use EFS instead (designed for shared access)

❌ Standard applications:
   Most apps don't support concurrent block access

❌ Read-only sharing:
   Take snapshots and create separate volumes instead
```

---

### EBS Encryption

**What is EBS Encryption?**

Built-in encryption for EBS volumes using **AWS KMS keys**, encrypting data at rest and in transit between EC2 and EBS.

**Encryption scope:**
- Data at rest (on volume)
- Data in transit (EC2 ↔ EBS)
- Snapshots (encrypted if source volume encrypted)
- Volumes restored from encrypted snapshots

---

#### How Encryption Works

```
┌────────────────────────────────────────────────────────────┐
│              EBS Encryption Architecture                   │
└────────────────────────────────────────────────────────────┘

EC2 Instance                          EBS Service
     │                                     │
     │ 1. Request: Write data             │
     ▼                                     │
┌──────────┐                              │
│ Plaintext│                              │
│   Data   │                              │
└────┬─────┘                              │
     │                                     │
     │ 2. Encrypted in-transit (TLS)      │
     │                                     ▼
     └─────────────────────────────►  ┌─────────┐
                                      │ KMS Key │
                                      │ (Master)│
                                      └────┬────┘
                                           │
                                      3. Generate
                                      Data Key
                                           │
                                           ▼
                                      ┌────────────┐
                                      │ Data Key   │
                                      │ (Encrypted)│
                                      └─────┬──────┘
                                           │
                                      4. Encrypt data
                                           │
                                           ▼
                                      ┌────────────┐
                                      │ Encrypted  │
                                      │ Volume     │
                                      └────────────┘

Read operation: Reverse process (decrypt with data key)
```

---

#### Encryption Configuration

**Enable encryption:**

```
Option 1: Enable encryption when creating volume
  Create volume settings:
    ✅ Encrypted: Yes
    Key: (choose KMS key)
      - AWS managed key (free): aws/ebs
      - Customer managed key: my-custom-key

Option 2: Enable encryption by default (account level)
  EC2 Settings → EBS Encryption
    ✅ Enable encryption by default

  Effect: All new volumes automatically encrypted
```

**Migrate unencrypted to encrypted:**

```
Cannot encrypt existing volume in-place

Migration process:
  1. Create snapshot of unencrypted volume
  2. Copy snapshot with encryption enabled
  3. Create new volume from encrypted snapshot
  4. Attach encrypted volume to instance
  5. Delete unencrypted volume

Downtime: Required (must detach old, attach new)
```

---

#### Encryption Performance

**Impact on performance:**
```
Encryption overhead: Minimal (<5% latency impact)
Modern EC2 instances: Hardware-accelerated encryption (negligible impact)

Best practice: Always enable encryption (security benefit >> minimal cost)
```

**Cost:**
```
Encryption itself: FREE (no additional charge)
KMS key usage:
  - AWS managed key (aws/ebs): FREE
  - Customer managed key: $1/month + API calls
```

---

### EBS-Optimized Instances

**What are EBS-Optimized Instances?**

EC2 instances with **dedicated bandwidth** for EBS I/O, separate from network traffic.

**Why it exists:**
- Shared network impacts EBS performance
- Storage-intensive workloads need guaranteed bandwidth
- Consistent performance required

---

**Non-optimized vs Optimized:**

```
Non-EBS-Optimized Instance:
  ┌────────────────┐
  │ EC2 Instance   │
  └────────┬───────┘
           │ Shared Network
     ┌─────┴─────┐
     │           │
  Network     EBS I/O
  Traffic     ← Competes with network traffic

Result: Network traffic impacts EBS performance


EBS-Optimized Instance:
  ┌────────────────┐
  │ EC2 Instance   │
  └─────┬────┬─────┘
        │    │ Dedicated EBS Bandwidth
     Network EBS I/O ← No competition
     Traffic

Result: Consistent EBS performance
```

**Modern instances**: Most instance types are EBS-optimized by default (no extra cost).

**Dedicated bandwidth examples:**
```
Instance Type    EBS Bandwidth
m5.large         4,750 Mbps
m5.xlarge        6,800 Mbps
m5.2xlarge       10,000 Mbps
c5.9xlarge       10,000 Mbps
r5.4xlarge       10,000 Mbps
```

---

## EFS (Elastic File System)

### What is EFS?

**Amazon EFS** is a **fully managed NFS (Network File System)** that provides **shared file storage** for multiple EC2 instances simultaneously across multiple Availability Zones.

**Why it exists:**
- EBS volumes can only attach to one instance (except Multi-Attach with limitations)
- Need shared file storage accessible by many instances
- Applications require POSIX-compliant filesystem
- Want elastic, auto-scaling storage (no capacity planning)

**Problems it solves:**
- **Shared access**: Multiple instances access same files simultaneously
- **High availability**: Automatically replicated across multiple AZs
- **Elastic scaling**: Storage grows/shrinks automatically
- **POSIX compliance**: Standard Linux filesystem semantics
- **Concurrent access**: Thousands of connections simultaneously

---

### EFS vs EBS vs S3

```
┌──────────────────────────────────────────────────────────────────┐
│      EBS           │       EFS           │        S3             │
├────────────────────┼─────────────────────┼───────────────────────┤
│ Block storage      │ File storage (NFS)  │ Object storage        │
├────────────────────┼─────────────────────┼───────────────────────┤
│ Single instance    │ Multiple instances  │ Unlimited clients     │
│ (except Multi-     │ (shared)            │ (via API)             │
│ Attach)            │                     │                       │
├────────────────────┼─────────────────────┼───────────────────────┤
│ Single AZ          │ Multi-AZ            │ Regional (multi-AZ)   │
├────────────────────┼─────────────────────┼───────────────────────┤
│ Fixed size         │ Auto-scaling        │ Unlimited             │
│ (provision)        │ (elastic)           │                       │
├────────────────────┼─────────────────────┼───────────────────────┤
│ Sub-ms latency     │ Low ms latency      │ Higher latency        │
├────────────────────┼─────────────────────┼───────────────────────┤
│ Use: OS drives,    │ Use: Shared file    │ Use: Backups, static  │
│ databases, apps    │ systems, content    │ content, data lakes   │
│                    │ repos, web serving  │                       │
└──────────────────────────────────────────────────────────────────┘
```

---

### EFS Architecture

```
┌────────────────────────────────────────────────────────────┐
│                  EFS Multi-AZ Architecture                 │
└────────────────────────────────────────────────────────────┘

                    EFS File System
                  (Regional resource)
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
      AZ-1               AZ-2               AZ-3
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│              │   │              │   │              │
│ Mount Target │   │ Mount Target │   │ Mount Target │
│  (ENI with   │   │  (ENI with   │   │  (ENI with   │
│   IP address)│   │   IP address)│   │   IP address)│
│              │   │              │   │              │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
   ┌───▼────┐        ┌────▼───┐        ┌────▼───┐
   │  EC2   │        │  EC2   │        │  EC2   │
   │Instance│        │Instance│        │Instance│
   └────────┘        └────────┘        └────────┘

All instances mount same filesystem:
  sudo mount -t nfs4 fs-abc123.efs.us-east-1.amazonaws.com:/ /mnt/efs

Data automatically replicated across all AZs
If AZ-1 fails, instances in AZ-2 and AZ-3 continue with no data loss
```

**Key characteristics:**
- **Regional service**: Spans all AZs in region
- **Mount targets**: One per AZ for low-latency local access
- **Automatic replication**: Data stored redundantly across AZs
- **Shared access**: Thousands of instances can mount simultaneously
- **Elastic**: Storage capacity grows/shrinks automatically

---

### Performance Modes

EFS offers two performance modes (set at filesystem creation, cannot change):

#### 1. General Purpose (Default)

**What it is**: Optimized for latency-sensitive workloads.

**Performance:**
```
Latency: Low (single-digit milliseconds)
Operations: Up to 7,000 file operations per second
Use case: Most workloads (web serving, content management, dev/test)

Throughput scaling:
  - Bursting mode: 100 MB/s per TB stored (baseline)
  - Baseline: Grows with filesystem size

Example:
  1 TB filesystem → 100 MB/s baseline
  5 TB filesystem → 500 MB/s baseline
```

**Best for:**
- Web servers
- Content management systems
- Home directories
- General file sharing

---

#### 2. Max I/O

**What it is**: Optimized for high aggregate throughput and operations per second.

**Performance:**
```
Latency: Higher (slightly higher than General Purpose)
Operations: >7,000 file operations per second (hundreds of thousands)
Throughput: Higher aggregate throughput

Trade-off: Higher latency per operation, but more total throughput
```

**Best for:**
- Big data analytics
- Media processing (many files processed in parallel)
- Genomics analysis
- High-throughput workloads with many concurrent clients

**Decision:**
```
General Purpose: Default choice for most applications
Max I/O: Only if you hit 7,000 ops/sec limit on General Purpose
```

---

### Throughput Modes

EFS offers three throughput modes (can be changed):

#### 1. Bursting Throughput (Default)

**How it works**: Throughput scales with filesystem size, with ability to burst.

```
Baseline throughput: 50 MB/s per TB stored (minimum 1 MB/s)
Burst throughput: 100 MB/s (available using burst credits)

Burst credits:
  - Earn: 50 MB/s per TB of storage
  - Spend: When exceeding baseline
  - Maximum: 2.1 TB burst credit balance

Example:
  Filesystem: 100 GB
  Baseline: 100 GB / 1024 GB × 50 MB/s = 5 MB/s
  Burst: Up to 100 MB/s (using credits)

  If idle: Accumulate burst credits
  If busy: Use burst credits to exceed 5 MB/s baseline

  Credit depletion: If sustained > baseline → throttled to baseline
```

**Best for:**
- Workloads with spiky traffic
- Small filesystems with occasional high throughput needs
- Cost-sensitive applications (no extra charge)

---

#### 2. Provisioned Throughput

**What it is**: Pay for specific throughput regardless of storage size.

```
Configuration:
  Provision: 200 MB/s throughput
  Storage: 50 GB

Cost:
  Storage: 50 GB × $0.30 = $15/month (Standard class)
  Provisioned throughput: (200 - 2.5) MB/s × $6 = $1,185/month
    └─ 2.5 MB/s included (50 GB / 20 = 2.5 baseline)

Total: $1,200/month

Use case: Small filesystem needing high throughput
  (Otherwise would need huge filesystem for baseline throughput)
```

**Best for:**
- Small filesystems requiring high throughput
- Predictable, sustained high throughput workloads
- When bursting mode insufficient

---

#### 3. Elastic Throughput (Recommended for Most)

**What it is**: Automatically scales throughput up and down based on workload (pay only for throughput used).

```
How it works:
  - Automatically scales to 3 GB/s reads, 1 GB/s writes
  - No baseline, no bursting, no provisioning
  - Pay only for throughput used (per GB transferred)

Pricing:
  Read: $0.30 per GB transferred
  Write: $0.60 per GB transferred

Example:
  Transfer 100 GB reads, 50 GB writes monthly
  Cost: (100 × $0.30) + (50 × $0.60) = $30 + $30 = $60

  Compare Provisioned Throughput at 100 MB/s:
    Fixed cost: ~$600/month (regardless of actual usage)

  Elastic wins: If usage variable or unpredictable
```

**Best for:**
- Unpredictable workloads
- Variable throughput requirements
- Spiky traffic patterns
- Default choice for new applications (2024+)

---

### Storage Classes

EFS has three storage classes for cost optimization:

#### 1. Standard

**What it is**: Frequently accessed files.

```
Performance: Low latency (single-digit ms)
Availability: 99.99%
Durability: 11 nines (replicated across multiple AZs)
Cost: $0.30 per GB-month
```

---

#### 2. Infrequent Access (IA)

**What it is**: Infrequently accessed files (lower storage cost, retrieval fee).

```
Performance: First-byte latency slightly higher
Availability: 99.99%
Cost:
  Storage: $0.016 per GB-month (95% cheaper than Standard)
  Retrieval: $0.01 per GB read

Break-even: If accessed <30 times per month, IA cheaper

Example:
  File: 10 GB
  Access: 10 times per month

  Standard: 10 GB × $0.30 = $3/month

  IA:
    Storage: 10 GB × $0.016 = $0.16/month
    Retrieval: 10 GB × 10 accesses × $0.01 = $1/month
    Total: $1.16/month (61% cheaper)
```

---

#### 3. Archive

**What it is**: Rarely accessed archival data (lowest cost, highest retrieval fee).

```
Cost:
  Storage: $0.008 per GB-month (97% cheaper than Standard)
  Retrieval: $0.03 per GB read

Break-even: If accessed <10 times per month

Use case: Historical data, compliance archives, long-term backups
```

---

### Lifecycle Management

**Automatic transition** between storage classes based on access patterns.

```
Configuration:
  Lifecycle policy: Move to IA after 30 days of no access

How it works:
  Day 0: File created in Standard class
  Day 1-30: File accessed regularly → Stays in Standard
  Day 31: File not accessed for 30 days → Moved to IA
  Day 60: File accessed → Moved BACK to Standard
  Day 90: File not accessed for 30 days → Moved to IA again

Automatic optimization:
  - Frequently accessed: Stay in Standard (low latency)
  - Infrequently accessed: Move to IA (cost savings)
```

**Policy options:**
```
Transition to IA after:
  - 7 days of no access
  - 14 days
  - 30 days (recommended)
  - 60 days
  - 90 days

Transition to Archive after:
  - 90 days of no access (requires IA enabled first)
  - 180 days
  - 270 days
  - 365 days
```

---

### EFS Access Points

**What are Access Points?**

Application-specific entry points into EFS with **enforced user identity and path**.

**Why they exist:**
- Need to enforce specific permissions per application
- Want to enforce different root directories for different apps
- Simplify IAM policy management

**How they work:**

```
┌────────────────────────────────────────────────────────────┐
│                  EFS Access Points                         │
└────────────────────────────────────────────────────────────┘

EFS Filesystem: /
  ├── /app-a/
  │    └── data/
  ├── /app-b/
  │    └── data/
  └── /shared/

Access Point 1: app-a-access
  - Root directory: /app-a
  - POSIX user: 1001
  - POSIX group: 1001
  - Permissions: UID 1001 can read/write

Access Point 2: app-b-access
  - Root directory: /app-b
  - POSIX user: 1002
  - POSIX group: 1002
  - Permissions: UID 1002 can read/write

Application A mounts: fs-abc123::fsap-111 (Access Point 1)
  → Sees /app-a as root (/)
  → Cannot access /app-b or /shared
  → All operations run as UID 1001

Application B mounts: fs-abc123::fsap-222 (Access Point 2)
  → Sees /app-b as root (/)
  → Cannot access /app-a or /shared
  → All operations run as UID 1002
```

**Benefits:**
- **Isolation**: Each app confined to its directory
- **Simplified management**: No manual permission management in apps
- **Security**: Enforced POSIX users/groups
- **Multi-tenancy**: Safe sharing of one filesystem across applications

---

### EFS Replication

**What is EFS Replication?**

**Automatic, continuous replication** of EFS filesystem to another region for disaster recovery.

```
┌────────────────────────────────────────────────────────────┐
│                  EFS Replication                           │
└────────────────────────────────────────────────────────────┘

Source Region: us-east-1
  EFS Filesystem: fs-source
    └─ Data: 1 TB

    │ Continuous replication
    │ (changes replicated in ~15 min)
    ▼

Destination Region: eu-west-1
  EFS Filesystem: fs-replica (read-only)
    └─ Data: 1 TB (synchronized)

Failover scenario:
  1. us-east-1 region fails
  2. Delete replication configuration
  3. fs-replica becomes writable
  4. Point applications to fs-replica
  5. Continue operations in eu-west-1
```

**Characteristics:**
- **RPO (Recovery Point Objective)**: ~15 minutes (typical replication lag)
- **RTO (Recovery Time Objective)**: Minutes (manual failover)
- **Replication**: One-way (source → destination)
- **Cost**: Pay for storage in both regions + data transfer

---

### Mount Targets

**What are Mount Targets?**

**Network interfaces** in each AZ that allow EC2 instances to mount the EFS filesystem.

```
Best practice: One mount target per AZ

Why?
  - Instances mount to local AZ mount target (lowest latency)
  - Cross-AZ mount increases latency and costs money

Correct setup:
  AZ-1: Instance-1 → Mount target in AZ-1
  AZ-2: Instance-2 → Mount target in AZ-2

  Latency: ~1-5 ms (local AZ)
  Cost: No cross-AZ transfer charges

Incorrect setup:
  AZ-1: Instance-1 → Mount target in AZ-2 (cross-AZ)

  Latency: ~10-20 ms (higher)
  Cost: $0.01 per GB cross-AZ transfer
```

**Mount target configuration:**
```
Each mount target has:
  - IP address in VPC subnet
  - Security group (controls access)
  - DNS name (fs-abc123.efs.us-east-1.amazonaws.com)
```

---

### EFS Use Cases

**1. Shared Content Repository**

```
Scenario: WordPress site on multiple EC2 instances

Traditional problem:
  - Instance-1 has uploads/ directory
  - Instance-2 doesn't see Instance-1's uploads
  - User uploads to Instance-1, not visible on Instance-2

EFS solution:
  - All instances mount same EFS filesystem at /var/www/html/wp-content/uploads
  - User uploads to Instance-1 → Visible on all instances
  - Shared state across all web servers
```

---

**2. Home Directories**

```
Scenario: Linux users need home directories accessible from any instance

Setup:
  - EFS mounted at /home
  - All instances mount same EFS
  - User logs into any instance → Same home directory

Benefit: Consistent user environment, no data duplication
```

---

**3. Big Data Analytics**

```
Scenario: Spark cluster processing large datasets

Setup:
  - Input data on EFS
  - Multiple Spark workers (EC2 instances)
  - All workers read from EFS in parallel

Performance:
  - Max I/O mode for high throughput
  - Elastic throughput for bursting
  - Thousands of concurrent reads
```

---

### EFS vs Amazon FSx

```
When to use EFS:
  ✅ Linux-based applications
  ✅ Need NFS protocol
  ✅ Multi-AZ by default (high availability)
  ✅ POSIX-compliant filesystem

When to use FSx:
  ✅ Windows applications (FSx for Windows)
  ✅ High-performance computing (FSx for Lustre)
  ✅ NetApp ONTAP features needed (FSx for NetApp)
  ✅ OpenZFS filesystem (FSx for OpenZFS)
```

---

## FSx

### What is Amazon FSx?

**Amazon FSx** is a family of **fully managed** third-party file systems optimized for specific workloads. Unlike EFS (AWS-native NFS), FSx provides popular third-party filesystems (Windows File Server, Lustre, NetApp ONTAP, OpenZFS).

**Why it exists:**
- Applications built for specific filesystems (SMB, Lustre, ONTAP, ZFS)
- Migration from on-premises requires same filesystem
- Specialized performance characteristics needed
- Enterprise features (deduplication, compression, snapshots) required

---

### FSx for Windows File Server

**What it is**: Fully managed **Windows-native** shared file storage built on **Windows Server** with **SMB** protocol.

**Why it exists:**
- Windows applications require SMB protocol
- Active Directory integration needed
- Need Windows-specific features (NTFS, DFS, quotas)
- Migrate Windows file servers to AWS

**Problems it solves:**
- **Windows compatibility**: Native SMB, NTFS permissions
- **Active Directory**: Integrated authentication
- **High availability**: Multi-AZ deployment
- **Managed**: No Windows server maintenance

---

#### Architecture

```
┌────────────────────────────────────────────────────────────┐
│        FSx for Windows File Server Architecture            │
└────────────────────────────────────────────────────────────┘

      Multi-AZ Deployment
 ┌──────────────────────────────────┐
 │  AZ-1            AZ-2             │
 │ ┌──────┐       ┌──────┐          │
 │ │Active│◄─────►│Standby│         │
 │ │Server│  Sync │Server│          │
 │ └───┬──┘       └──────┘          │
 │     │ Automatic                  │
 │     │ Failover                   │
 └─────┼────────────────────────────┘
       │
       ▼
 ┌────────────────┐
 │ Windows Clients│
 │ (EC2, On-prem) │
 │ Mount via SMB  │
 └────────────────┘

Active Directory:
  - Integrates with AWS Managed AD or on-premises AD
  - Windows authentication
  - NTFS ACLs
```

---

#### Key Features

**1. SMB Protocol Support**
```
Supported versions: SMB 2.0, 2.1, 3.0, 3.1.1
Encryption: SMB 3.0+ encryption in transit
Access: Windows, Linux (with CIFS utils), macOS
```

**2. Active Directory Integration**
```
Authentication:
  - AWS Managed Microsoft AD
  - Self-managed AD (on-premises)
  - AD Connector

Permissions:
  - NTFS ACLs
  - Windows file permissions
  - Group policies
```

**3. DFS (Distributed File System)**
```
DFS Namespaces: Group multiple shares into namespace
DFS Replication: (Not supported, use FSx replication)
```

**4. Shadow Copies**
```
User-initiated restore:
  - Previous Versions (Windows feature)
  - Restore files from snapshots
  - No admin intervention
```

---

#### Performance Tiers

**SSD Storage** (Latency-sensitive workloads):
```
Throughput: Up to 2 GB/s
IOPS: Tens of thousands
Latency: Sub-millisecond
Use case: Databases, user home directories, dev/test
Cost: Higher
```

**HDD Storage** (Throughput-optimized):
```
Throughput: Up to 2 GB/s
IOPS: Lower
Latency: Single-digit milliseconds
Use case: Home directories, department shares, content repos
Cost: Lower (up to 50% cheaper)
```

---

#### Use Cases

**1. Lift-and-Shift Windows File Servers**
```
On-premises: Windows File Server (SMB shares)
Migration: FSx for Windows File Server (drop-in replacement)
Benefits: No application changes, managed service
```

**2. SharePoint Server**
```
SharePoint requires: SMB-based storage
FSx provides: SMB with Active Directory integration
```

**3. SQL Server (file-based storage)**
```
SQL Server can store databases on SMB shares
FSx provides: High-performance SMB with Multi-AZ
```

---

### FSx for Lustre

**What it is**: High-performance **parallel filesystem** designed for **compute-intensive workloads** (HPC, ML, big data).

**Why it exists:**
- HPC workloads need extreme throughput (hundreds of GB/s)
- Machine learning training requires fast access to datasets
- Big data analytics needs parallel I/O
- Sub-millisecond latency required

**Problems it solves:**
- **Massive throughput**: Hundreds of GB/s aggregate throughput
- **Low latency**: Sub-millisecond
- **S3 integration**: Seamless data loading from S3
- **Parallel I/O**: Thousands of clients simultaneously

---

#### Architecture

```
┌────────────────────────────────────────────────────────────┐
│              FSx for Lustre Architecture                   │
└────────────────────────────────────────────────────────────┘

    S3 Bucket (Data Lake)
         │
         │ Lazy load or preload
         ▼
   ┌────────────────┐
   │ FSx for Lustre │
   │  (Cache layer) │
   │                │
   │ - SSD: 1 GB/s+ │
   │ - HDD: 100s    │
   │   MB/s         │
   └───────┬────────┘
           │
     ┌─────┴─────┬─────────┬─────────┐
     │           │         │         │
     ▼           ▼         ▼         ▼
Compute 1   Compute 2  Compute 3  Compute N
(EC2/EKS)   (EC2/EKS)  (EC2/EKS)  (EC2/EKS)

Parallel access: All compute reads/writes in parallel
Throughput scales: Linear with filesystem size
```

---

#### Deployment Types

**1. Scratch**
```
Purpose: Temporary, short-term processing
Durability: NOT replicated (single copy)
Cost: Lowest ($0.14/GB-month for 200 MB/s/TB)
Performance: Same as Persistent
Use case:
  - Temporary workloads
  - Data in S3 (can reload if lost)
  - Cost-sensitive HPC jobs

Risk: If storage server fails, data lost (no replication)
```

**2. Persistent**
```
Purpose: Long-term storage, production workloads
Durability: Replicated within AZ
Cost: Higher ($0.17/GB-month for 125 MB/s/TB)
Performance: Same as Scratch
Use case:
  - Production analytics
  - Long-running training
  - Critical data processing

Benefits: Data survives hardware failures
```

---

#### S3 Integration

**Seamless S3 data loading:**

```
Setup:
  1. Create FSx for Lustre filesystem
  2. Link to S3 bucket: s3://my-data-lake/

Data access:
  Lazy loading (default):
    - File metadata loaded immediately
    - File content loaded on first access
    - Subsequent reads from FSx cache (fast)

  Preload (optional):
    - Load all data from S3 to FSx upfront
    - Use when: Need all data immediately

  Write-back:
    - Write to FSx
    - Optionally sync back to S3
    - S3 becomes persistent storage

Workflow:
  1. Load dataset from S3 to FSx (lazy or preload)
  2. Compute instances process data on FSx (high speed)
  3. Export results back to S3 (persistent storage)
  4. Delete FSx filesystem (no longer needed)
```

---

#### Performance Characteristics

**Throughput per TB:**
```
SSD-based:
  - 200 MB/s per TB of storage
  - 1,000 MB/s per TB (optional, higher cost)

HDD-based:
  - 12 MB/s per TB (baseline)
  - 40 MB/s per TB (optional)

Scaling:
  10 TB filesystem with 200 MB/s/TB → 2,000 MB/s (2 GB/s)
  100 TB filesystem → 20,000 MB/s (20 GB/s)
```

---

#### Use Cases

**1. Machine Learning Training**
```
Dataset: 50 TB on S3
Training: 1000 GPU instances need fast data access

FSx for Lustre:
  - Load dataset from S3 to FSx (preload)
  - All GPUs read training data from FSx in parallel
  - Throughput: 10+ GB/s aggregate
  - Training time: Hours instead of days
```

**2. Genomics Analysis**
```
Workflow: Process millions of genome sequences
FSx provides: Parallel I/O for thousands of compute cores
Benefit: Reduce analysis time from weeks to days
```

**3. Video Rendering**
```
Scenario: Render thousands of video frames in parallel
FSx: High-throughput storage for frame data
```

---

### FSx for NetApp ONTAP

**What it is**: Fully managed **NetApp ONTAP** filesystem with enterprise features (deduplication, compression, snapshots, cloning).

**Why it exists:**
- Organizations use NetApp on-premises
- Need NetApp-specific features in cloud
- Multi-protocol support required (NFS, SMB, iSCSI)
- Advanced data management features needed

---

#### Key Features

**1. Multi-Protocol Support**
```
Protocols supported:
  - NFS (v3, v4, v4.1, v4.2)
  - SMB (2.0, 2.1, 3.0, 3.1.1)
  - iSCSI

Use case: Mixed environment (Linux + Windows)
  Linux clients: Mount via NFS
  Windows clients: Mount via SMB
  Same data accessible via both protocols
```

**2. Storage Efficiency**
```
Deduplication: Eliminate duplicate blocks
Compression: Reduce storage footprint
Thin provisioning: Only use actual space consumed

Example:
  Provisioned: 10 TB
  Actual data: 5 TB (after dedup/compression)
  Storage used: 5 TB
  Cost: Pay for 5 TB, not 10 TB
```

**3. Snapshots and Clones**
```
Snapshots: Point-in-time copies (space-efficient)
Clones: Instant writable copies from snapshots

Use case: Development environments
  1. Take snapshot of production data
  2. Create 10 clones from snapshot
  3. Developers work on clones (isolated)
  4. Storage used: Snapshot + only changed blocks in clones
```

**4. SnapMirror Replication**
```
Cross-region replication:
  Source: FSx ONTAP in us-east-1
  Destination: FSx ONTAP in eu-west-1

  Incremental replication for DR
```

---

#### Use Cases

**1. Hybrid Cloud**
```
On-premises: NetApp ONTAP
Cloud: FSx for NetApp ONTAP

Benefits:
  - Same filesystem technology
  - Seamless replication (SnapMirror)
  - Familiar management tools
```

**2. Database Workloads**
```
Oracle, SQL Server, PostgreSQL:
  - iSCSI block storage
  - Snapshots for backups
  - Clones for dev/test
```

**3. Multi-Protocol File Sharing**
```
Mixed environment:
  Engineering (Linux): Access via NFS
  Business users (Windows): Access via SMB
  Same files, different protocols
```

---

### FSx for OpenZFS

**What it is**: Fully managed **OpenZFS** filesystem with high performance and ZFS features (snapshots, compression, data integrity).

**Why it exists:**
- Applications built for ZFS
- Need ZFS features (copy-on-write snapshots, checksums)
- Linux-based workloads requiring high performance
- Data integrity critical

---

#### Key Features

**1. Copy-on-Write Snapshots**
```
Instant snapshots: Zero copy, instant creation
Space-efficient: Only changed blocks consume space
Read-only snapshots: Immutable point-in-time copies
```

**2. Data Integrity**
```
End-to-end checksums: Detect data corruption
Self-healing: Automatically fix corrupt blocks (with redundancy)
```

**3. Compression**
```
LZ4 compression: Fast compression algorithm
Transparent: Automatic compression/decompression
Storage savings: 30-50% typical
```

---

#### Performance

```
Throughput: Up to 12.5 GB/s
IOPS: Up to 1 million
Latency: Sub-millisecond (NVMe SSDs)

Use case: High-performance Linux workloads
```

---

#### Use Cases

**1. Linux Application Migration**
```
On-premises: ZFS-based storage
Migration: FSx for OpenZFS (compatible)
```

**2. Financial Services**
```
Requirements: Data integrity, snapshots, performance
FSx OpenZFS: Meets all requirements natively
```

---

## Storage Gateway

### What is AWS Storage Gateway?

**AWS Storage Gateway** is a **hybrid cloud storage service** that connects on-premises applications to AWS cloud storage (S3, EBS, Glacier) via standard protocols (NFS, SMB, iSCSI).

**Why it exists:**
- On-premises applications need cloud storage
- Gradual migration to cloud (hybrid model)
- Need local cache for low latency + cloud backup
- Existing apps can't be modified to use S3 API

**Problems it solves:**
- **Hybrid integration**: Bridge on-premises and cloud storage
- **Backup to cloud**: Automated backups to S3/Glacier
- **Disaster recovery**: Replicate data to AWS
- **Local caching**: Low-latency access with cloud durability
- **Protocol translation**: Apps use NFS/SMB/iSCSI, data stored in S3

---

### Storage Gateway Types

AWS offers **three types** of Storage Gateway:

```
┌────────────────────────────────────────────────────────────┐
│                Storage Gateway Types                       │
└────────────────────────────────────────────────────────────┘

1. File Gateway (S3 File)
   Protocol: NFS, SMB
   Backend: S3 buckets
   Use case: File sharing, backups, archives

2. Volume Gateway
   Protocol: iSCSI (block storage)
   Backend: S3 snapshots
   Use case: Block storage, disk backups, DR

3. Tape Gateway (VTL)
   Protocol: iSCSI VTL
   Backend: S3, Glacier
   Use case: Backup software integration, tape replacement
```

---

### File Gateway (S3 File)

**What it is**: Presents **S3 buckets as NFS/SMB shares** to on-premises applications.

**Architecture:**

```
┌────────────────────────────────────────────────────────────┐
│                 File Gateway Architecture                  │
└────────────────────────────────────────────────────────────┘

On-Premises Data Center:
  ┌────────────────────────────┐
  │ Applications               │
  │ (File servers, apps)       │
  └──────────┬─────────────────┘
             │ NFS/SMB
             ▼
  ┌────────────────────────────┐
  │ File Gateway (VM or        │
  │ Hardware Appliance)        │
  │ - Local cache (hot data)   │
  │ - Async upload to S3       │
  └──────────┬─────────────────┘
             │ HTTPS
             │ (Internet or Direct Connect)
             ▼
AWS Cloud:
  ┌────────────────────────────┐
  │ S3 Bucket                  │
  │ - All files stored         │
  │ - Lifecycle policies       │
  │ - Versioning, encryption   │
  └────────────────────────────┘
```

**How it works:**
```
1. Application writes file via NFS/SMB
2. File Gateway caches file locally (fast access)
3. File Gateway uploads to S3 asynchronously
4. File appears as S3 object (with metadata)
5. Subsequent reads from local cache (fast)
6. If cache miss, download from S3
```

---

#### Use Cases

**1. Backup and Archive**
```
Problem: On-premises backup infrastructure expensive

Solution:
  - Mount File Gateway as NFS share
  - Backup software writes to share
  - Data automatically uploaded to S3
  - Apply lifecycle policies (transition to Glacier)

Benefit: Cloud economics, no tape management
```

**2. Disaster Recovery**
```
Primary site: File Gateway replicating to S3
DR site: Mount same S3 bucket via File Gateway in AWS

Failover:
  1. Primary site fails
  2. Launch File Gateway in AWS
  3. Mount same S3 bucket
  4. Applications access files (low latency in AWS)
```

**3. Cloud Data Processing**
```
Workflow:
  1. On-premises generates files
  2. File Gateway uploads to S3
  3. S3 event triggers Lambda/EC2 processing
  4. Results written back to S3
  5. Available on-premises via File Gateway
```

---

### Volume Gateway

**What it is**: Provides **iSCSI block storage** volumes backed by S3 snapshots.

**Two modes:**

#### 1. Cached Volumes

```
Architecture:
  On-Premises:
    ┌────────────────────┐
    │ Application        │
    │ (Database, etc.)   │
    └─────────┬──────────┘
              │ iSCSI
              ▼
    ┌────────────────────┐
    │ Volume Gateway     │
    │ - Cache: Hot data  │
    │ - Full data: S3    │
    └─────────┬──────────┘
              │
  AWS Cloud:  ▼
    ┌────────────────────┐
    │ S3 (Primary        │
    │ storage)           │
    │ - EBS snapshots    │
    └────────────────────┘

Characteristics:
  - Primary storage: S3 (all data)
  - Local cache: Frequently accessed data only
  - Volume size: Up to 32 TB per volume
  - Total capacity: Up to 1 PB (32 volumes)

Use case: Large datasets, infrequent access pattern
```

---

#### 2. Stored Volumes

```
Architecture:
  On-Premises:
    ┌────────────────────┐
    │ Application        │
    └─────────┬──────────┘
              │ iSCSI
              ▼
    ┌────────────────────┐
    │ Volume Gateway     │
    │ - Primary storage  │
    │   (local)          │
    └─────────┬──────────┘
              │ Async snapshots
  AWS Cloud:  ▼
    ┌────────────────────┐
    │ S3 (Backup storage)│
    │ - EBS snapshots    │
    └────────────────────┘

Characteristics:
  - Primary storage: On-premises (full dataset local)
  - S3: Async snapshots only (backup)
  - Low-latency: All data local
  - Volume size: Up to 16 TB per volume

Use case: Low-latency required, full dataset on-premises
```

---

#### Use Cases

**1. Disaster Recovery (Cached Volumes)**
```
Normal operation:
  - Applications use iSCSI volumes
  - Data stored in S3, cached locally

Disaster scenario:
  - On-premises fails
  - Create EBS volume from S3 snapshot
  - Attach to EC2 instance
  - Resume operations in AWS
```

**2. Database Backup (Stored Volumes)**
```
Production database:
  - Local storage for performance
  - Volume Gateway creates snapshots
  - Snapshots stored in S3
  - Point-in-time recovery available
```

---

### Tape Gateway (VTL - Virtual Tape Library)

**What it is**: Emulates **physical tape library** using S3 and Glacier, compatible with existing backup software.

**Architecture:**

```
┌────────────────────────────────────────────────────────────┐
│               Tape Gateway Architecture                    │
└────────────────────────────────────────────────────────────┘

On-Premises:
  ┌────────────────────────┐
  │ Backup Software        │
  │ (Veeam, Veritas, etc.) │
  └──────────┬─────────────┘
             │ iSCSI (VTL interface)
             ▼
  ┌────────────────────────┐
  │ Tape Gateway           │
  │ - Virtual tape library │
  │ - Virtual tape drives  │
  └──────────┬─────────────┘
             │
AWS Cloud:   │
  ┌──────────▼──────────────┐
  │ Virtual Tape Library    │
  │ (S3 - active tapes)     │
  │ - Fast retrieval        │
  └──────────┬──────────────┘
             │ Archive (tape ejected)
             ▼
  ┌─────────────────────────┐
  │ Virtual Tape Shelf      │
  │ (Glacier - archived     │
  │ tapes)                  │
  │ - Hours retrieval       │
  └─────────────────────────┘
```

**How it works:**
```
1. Backup software detects "tape library" (Tape Gateway)
2. Writes backup to "virtual tape" (appears as physical tape)
3. Tape data stored in S3 (Virtual Tape Library)
4. "Eject tape" → Move to Glacier (Virtual Tape Shelf)
5. Restore: "Load tape" → Retrieve from Glacier
```

---

#### Use Cases

**Replace Physical Tapes**
```
Problem:
  - Physical tapes expensive
  - Tape rotation manual
  - Off-site storage logistics
  - Tape degradation over time

Solution: Tape Gateway
  - Same backup software (no changes)
  - Cloud storage (durable, scalable)
  - No physical tape management
  - Lower cost
```

---

### Storage Gateway Deployment Options

**1. Virtual Machine (VMware, Hyper-V)**
```
Deploy: Gateway as VM in on-premises data center
Requirements:
  - CPU: 4 cores minimum
  - RAM: 16 GB minimum
  - Disk: Cache + upload buffer
```

**2. Hardware Appliance**
```
Purchase: Physical appliance from AWS
Pre-configured: Plug and play
Use case: No virtualization infrastructure
```

**3. EC2 Instance**
```
Deploy: Gateway as EC2 instance
Use case: Cloud-to-cloud migration, testing
```

---

## Storage Services Comparison Summary

```
┌──────────────────────────────────────────────────────────────────┐
│ Service  │ Type  │ Use Case                                      │
├──────────┼───────┼───────────────────────────────────────────────┤
│ S3       │ Object│ Backups, static content, data lakes           │
│ EBS      │ Block │ EC2 boot drives, databases                    │
│ EFS      │ File  │ Shared file storage (Linux/NFS)               │
│ FSx Win  │ File  │ Shared file storage (Windows/SMB)             │
│ FSx Lus  │ File  │ HPC, ML training, high throughput             │
│ FSx ONTAP│ File  │ Enterprise features, multi-protocol           │
│ FSx ZFS  │ File  │ Linux high-performance, ZFS features          │
│ Gateway  │ Hybrid│ On-premises to cloud bridge                   │
└──────────────────────────────────────────────────────────────────┘
```

**Decision tree:**
```
Need storage for EC2? → EBS
Need shared file storage?
  ├─ Linux (NFS)? → EFS
  ├─ Windows (SMB)? → FSx for Windows
  ├─ HPC/ML? → FSx for Lustre
  └─ Enterprise features? → FSx for NetApp ONTAP

Need object storage? → S3
Need hybrid (on-prem + cloud)? → Storage Gateway
```