# AWS Storage Services - Complete Conceptual Guide (Part 2)

## Continuation of S3 (Simple Storage Service)

### S3 Inventory (continued from Part 1)

#### Inventory Report Contents

**CSV format example:**

```csv
Bucket,Key,VersionId,IsLatest,IsDeleteMarker,Size,LastModifiedDate,ETag,StorageClass,IsMultipartUploaded,ReplicationStatus,EncryptionStatus,ObjectLockRetainUntilDate,ObjectLockMode,ObjectLockLegalHoldStatus
my-data-bucket,documents/report.pdf,abc123,true,false,1048576,2026-02-06T10:30:00.000Z,"d41d8cd98f00b204e9800998ecf8427e",STANDARD,false,COMPLETED,SSE-S3,2033-02-06T00:00:00.000Z,COMPLIANCE,OFF
my-data-bucket,images/photo.jpg,def456,true,false,524288,2026-01-15T08:20:00.000Z,"e99a18c428cb38d5f260853678922e03",GLACIER,false,,,,,
```

**Report includes:**
- Bucket name
- Object key (full path)
- Version ID (if versioning enabled)
- Size in bytes
- Last modified timestamp
- Storage class
- Encryption status (SSE-S3, SSE-KMS, SSE-C, or none)
- Replication status
- Object Lock settings
- Multipart upload status
- ETag (MD5 hash)

---

#### Use Cases

**1. Storage Cost Analysis**

```
Query inventory with Athena:
  SELECT
    storage_class,
    SUM(size) / 1024 / 1024 / 1024 AS total_gb,
    COUNT(*) AS object_count
  FROM s3_inventory
  GROUP BY storage_class

Result:
  STANDARD: 500 GB, 10,000 objects
  STANDARD_IA: 1,200 GB, 5,000 objects
  GLACIER: 5,000 GB, 50,000 objects

Action: Identify objects to transition to cheaper storage
```

---

**2. Compliance Audit**

```
Find unencrypted objects:
  SELECT bucket, key, size
  FROM s3_inventory
  WHERE encryption_status = 'NOT-SSE'

Find objects without Object Lock:
  SELECT bucket, key, size
  FROM s3_inventory
  WHERE object_lock_mode IS NULL
    AND bucket = 'compliance-data'

Action: Identify compliance violations
```

---

**3. Lifecycle Policy Validation**

```
Find old objects still in Standard class:
  SELECT key, last_modified_date, storage_class
  FROM s3_inventory
  WHERE storage_class = 'STANDARD'
    AND last_modified_date < DATE '2025-01-01'

Expected: Should be in Standard-IA or Glacier
Actual: Still in Standard (lifecycle policy not working?)
```

---

**4. Version Cleanup**

```
Find buckets with excessive versions:
  SELECT key, COUNT(*) as version_count
  FROM s3_inventory
  WHERE is_latest = false
  GROUP BY key
  HAVING COUNT(*) > 100

Action: Clean up old versions to reduce costs
```

---

#### Inventory vs List Objects API

```
┌──────────────────────────────────────────────────────────────┐
│    List Objects API     │    S3 Inventory                    │
├─────────────────────────┼────────────────────────────────────┤
│ Real-time (current)     │ Snapshot (daily/weekly)            │
├─────────────────────────┼────────────────────────────────────┤
│ Paginated (1000/page)   │ Complete report (all objects)      │
├─────────────────────────┼────────────────────────────────────┤
│ API costs per request   │ Flat delivery cost                 │
├─────────────────────────┼────────────────────────────────────┤
│ Good for: Real-time     │ Good for: Analytics, audits,       │
│ queries, small buckets  │ large buckets (billions of objects)│
├─────────────────────────┼────────────────────────────────────┤
│ Slow for large buckets  │ Fast (pre-generated report)        │
│ (millions of API calls) │                                    │
└──────────────────────────────────────────────────────────────┘
```

**Cost comparison:**

```
Scenario: Analyze 1 billion objects

List Objects API:
  - 1,000,000,000 objects ÷ 1,000 per page = 1,000,000 requests
  - Cost: 1,000,000 × $0.0004 / 1,000 = $400
  - Time: Hours to days (serial pagination)

S3 Inventory:
  - Cost: ~$2.50 (inventory generation)
  - Time: Report generated in 24-48 hours
  - Result: CSV/Parquet files ready for Athena/Spark
```

---

#### Integration with Athena

**Query inventory directly with SQL:**

```sql
-- 1. Create Athena table pointing to inventory
CREATE EXTERNAL TABLE s3_inventory (
  bucket string,
  key string,
  version_id string,
  is_latest boolean,
  is_delete_marker boolean,
  size bigint,
  last_modified_date timestamp,
  e_tag string,
  storage_class string,
  is_multipart_uploaded boolean,
  replication_status string,
  encryption_status string
)
PARTITIONED BY (dt string)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://my-inventory-reports/my-data-bucket/inventory-config/';

-- 2. Query inventory
SELECT
  storage_class,
  COUNT(*) as objects,
  SUM(size) / 1073741824 as total_gb,
  SUM(size) * 0.023 / 1073741824 as monthly_cost_usd  -- Assuming Standard pricing
FROM s3_inventory
WHERE dt = '2026-02-06'
GROUP BY storage_class
ORDER BY total_gb DESC;
```

**Result:**
```
storage_class    | objects  | total_gb | monthly_cost_usd
GLACIER          | 50000000 | 5000.00  | 200.00
STANDARD_IA      | 5000000  | 1200.00  | 300.00
STANDARD         | 1000000  | 500.00   | 575.00
```

---

### S3 Batch Operations

**What is S3 Batch Operations?**

A service to perform **large-scale batch operations** on billions of S3 objects with a single request, tracking progress and providing completion reports.

**Why it exists:**
- Manual operations on millions of objects is impractical
- Need to modify objects in bulk (tags, storage class, ACLs)
- Copying/restoring large datasets requires orchestration
- Compliance actions need audit trails

**Problems it solves:**
- **Scale**: Process billions of objects without writing code
- **Reliability**: Automatic retries, progress tracking
- **Audit**: Completion reports showing what succeeded/failed
- **Permissions**: Centralized IAM role for all operations
- **Cost efficiency**: Optimized batch processing vs individual API calls

---

#### Supported Operations

**1. Copy Objects**
```
Use case: Replicate data to new bucket, change encryption
Example: Copy 100M objects to backup bucket with different KMS key
```

**2. Restore from Glacier**
```
Use case: Restore archived data for analysis
Example: Restore 1TB of Glacier objects to Standard for 30 days
```

**3. Invoke Lambda Function**
```
Use case: Custom processing (metadata extraction, validation)
Example: Run Lambda on 10M images to extract EXIF data
```

**4. Replace Object Tags**
```
Use case: Add/modify tags for lifecycle, cost allocation
Example: Tag 50M objects with "Department=Finance"
```

**5. Replace Object ACLs**
```
Use case: Bulk permission changes
Example: Make 1M objects publicly readable
```

**6. Delete Object Tags**
```
Use case: Remove obsolete tags
Example: Remove "Temporary=true" tag from 5M objects
```

**7. Object Lock Retention**
```
Use case: Apply legal holds or retention to existing objects
Example: Set 7-year retention on 100M compliance documents
```

---

#### How Batch Operations Work

```
┌────────────────────────────────────────────────────────────┐
│              Batch Operations Workflow                     │
└────────────────────────────────────────────────────────────┘

Step 1: Create Manifest (list of objects to process)
  Option A: S3 Inventory report (CSV)
  Option B: Manual CSV file

  Format:
    bucket,key
    my-bucket,documents/file1.pdf
    my-bucket,documents/file2.pdf
    ...1 billion more lines

Step 2: Upload manifest to S3
  s3://my-manifests/batch-job-2026-02-06.csv

Step 3: Create Batch Operations Job
  - Manifest location: s3://my-manifests/batch-job-2026-02-06.csv
  - Operation: Copy, Tag, Lambda, etc.
  - IAM Role: Role with permissions to perform operation
  - Priority: 1-10 (higher = more important)
  - Completion report: Where to save results

Step 4: Job Execution
  ┌─────────────────────────────────┐
  │   Batch Operations Service      │
  │   - Reads manifest              │
  │   - Processes objects in chunks │
  │   - Retries failures            │
  │   - Tracks progress             │
  └─────────────────────────────────┘
            │
            ├─ Success: 999,000,000 objects
            ├─ Failed: 1,000,000 objects
            └─ Total: 1,000,000,000 objects

Step 5: Completion Report
  s3://my-reports/batch-job-2026-02-06/
    ├─ succeeded.csv  (successful operations)
    └─ failed.csv     (failed operations with reasons)
```

---

#### Practical Example: Bulk Object Copy

**Scenario**: Copy 100 million objects from us-east-1 to eu-west-1 for disaster recovery.

```
Step 1: Generate manifest using S3 Inventory
  Source bucket: data-us-east-1
  Inventory report: s3://inventory-reports/data-us-east-1/2026-02-06/

Step 2: Create Batch Operations Job
  Job configuration:
    Manifest: s3://inventory-reports/data-us-east-1/2026-02-06/manifest.json
    Operation: Copy
    Destination bucket: data-eu-west-1
    Storage class: STANDARD_IA (cheaper in destination)
    Encryption: SSE-KMS with eu-west-1 KMS key
    IAM Role: BatchOperationsRole
      Permissions:
        - s3:GetObject on source bucket
        - s3:PutObject on destination bucket
        - kms:Decrypt on source KMS key
        - kms:Encrypt on destination KMS key

Step 3: Job Execution
  Progress:
    Hour 1: 10,000,000 objects copied (10%)
    Hour 5: 50,000,000 objects copied (50%)
    Hour 10: 100,000,000 objects copied (100%)

  Results:
    Success: 99,950,000 objects
    Failed: 50,000 objects (permissions errors, deleted during job)

Step 4: Review Completion Report
  s3://completion-reports/job-abc123/
    ├─ succeeded.csv (99,950,000 lines)
    └─ failed.csv (50,000 lines)

  Investigate failures:
    - Bucket policy blocking cross-region access
    - KMS key permissions missing
    - Objects deleted after manifest created

  Fix issues and rerun job on failed.csv
```

---

#### Lambda Invocation with Batch Operations

**Custom processing at scale:**

```
Scenario: Extract metadata from 10 million images

Lambda Function:
  exports.handler = async (event) => {
    // Event contains:
    // - tasks: Array of objects to process

    const results = await Promise.all(event.tasks.map(async (task) => {
      const bucket = task.s3BucketArn.split(':::')[1];
      const key = task.s3Key;

      try {
        // Download image from S3
        const image = await s3.getObject({ Bucket: bucket, Key: key }).promise();

        // Extract EXIF metadata
        const metadata = await extractExif(image.Body);

        // Save metadata to DynamoDB
        await dynamodb.putItem({
          TableName: 'ImageMetadata',
          Item: { imageKey: key, ...metadata }
        }).promise();

        return {
          taskId: task.taskId,
          resultCode: 'Succeeded',
          resultString: 'Metadata extracted'
        };
      } catch (error) {
        return {
          taskId: task.taskId,
          resultCode: 'PermanentFailure',
          resultString: error.message
        };
      }
    }));

    return {
      invocationSchemaVersion: '1.0',
      treatMissingKeysAs: 'PermanentFailure',
      invocationId: event.invocationId,
      results: results
    };
  };

Batch Operations Job:
  - Manifest: 10M image keys
  - Operation: Invoke Lambda
  - Lambda function: image-metadata-extractor
  - Execution: Processes in batches of 1000 objects per invocation
  - Total invocations: 10,000
  - Duration: ~2 hours (with adequate concurrency)
```

---

#### Job Priority and Concurrency

**Priority system:**

```
Multiple jobs running:
  Job A (Priority 10): Critical compliance tagging
  Job B (Priority 5):  Routine copy operation
  Job C (Priority 1):  Low-priority archive

Execution:
  - Job A gets highest resources
  - Job B runs with medium resources
  - Job C runs when A and B have spare capacity

Use case: Ensure urgent jobs complete faster
```

**Concurrency limits:**
- AWS controls parallelism automatically
- No explicit concurrency setting
- Scales based on job size and priority
- Typical: Thousands of objects per second

---

#### Cost Considerations

**Pricing:**
```
Batch Operations: $0.25 per million objects processed

Example: 100 million objects
  Cost: 100 × $0.25 = $25

Additional costs:
  - S3 requests (GET, PUT, COPY): Standard pricing
  - Data transfer: If cross-region
  - Lambda invocations: If using Lambda operation

Compare to custom script:
  - 100M API calls (individual operations): $40-100
  - EC2 instance running for days: $50+
  - Development time: Hours to days

Batch Operations: Cheaper, faster, more reliable
```

---

#### Monitoring and Troubleshooting

**Job states:**
```
New → Preparing → Ready → Active → Paused → Completing → Complete
                                      │
                                      └─→ Failed
                                      └─→ Cancelled
```

**CloudWatch metrics:**
- `NumberOfTasksSucceeded`
- `NumberOfTasksFailed`
- `PercentageComplete`

**Completion report:**
```
Success report (succeeded.csv):
  bucket,key,versionId,taskStatus,MD5Checksum
  my-bucket,file1.pdf,v123,succeeded,abc123def...

Failure report (failed.csv):
  bucket,key,versionId,taskStatus,errorCode,errorMessage
  my-bucket,file2.pdf,v456,failed,AccessDenied,Insufficient permissions
```

**Common failure reasons:**
- `AccessDenied`: IAM role lacks permissions
- `NoSuchKey`: Object deleted after manifest created
- `InvalidObjectState`: Object in Glacier, not restored
- `InvalidArgument`: Invalid operation parameters

---

### S3 Select & Glacier Select

**What is S3 Select?**

A feature that allows you to **retrieve only a subset of data** from an object using SQL-like queries, instead of downloading the entire object.

**Why it exists:**
- Downloading entire large files wastes bandwidth and time
- Applications often need only specific columns or rows
- Processing data locally is slow and expensive
- Network transfer costs add up

**Problems it solves:**
- **Bandwidth reduction**: Download only needed data (up to 80% less)
- **Latency reduction**: Faster queries (no full download)
- **Cost savings**: Less data transfer out
- **Compute efficiency**: Server-side filtering reduces client processing
- **Simplified architecture**: No need for dedicated query engine

---

#### How S3 Select Works

```
┌────────────────────────────────────────────────────────────┐
│          Traditional Approach (Without S3 Select)          │
└────────────────────────────────────────────────────────────┘

S3 Object: sales-data.csv (10 GB)
  - 100 million rows
  - 50 columns

Client Request: Get all sales in California

Process:
  1. Download entire 10 GB file from S3
  2. Parse all 100M rows locally
  3. Filter WHERE state = 'CA' (client-side)
  4. Get result: 5 million rows (~500 MB)

Cost:
  - Data transfer: 10 GB × $0.09 = $0.90
  - Time: Minutes (depending on bandwidth)
  - Compute: CPU cycles to parse 10 GB


┌────────────────────────────────────────────────────────────┐
│               S3 Select Approach                           │
└────────────────────────────────────────────────────────────┘

S3 Object: sales-data.csv (10 GB)

Client Request:
  SELECT * FROM S3Object WHERE state = 'CA'

Process:
  1. S3 scans object server-side
  2. S3 filters WHERE state = 'CA' (server-side)
  3. S3 returns only matching rows: 500 MB

Cost:
  - S3 Select: 10 GB scanned × $0.002 = $0.02
  - Data transfer: 500 MB × $0.09 = $0.045
  - Total: $0.065 (93% cost savings)
  - Time: Seconds (only 500 MB downloaded)
```

**Performance improvement**: Up to 400% faster, 80% cheaper for selective queries.

---

#### Supported File Formats

**Input formats:**
- **CSV**: With or without headers
- **JSON**: Lines or Documents format
- **Parquet**: Columnar format (best performance)
- **Apache Avro**: Binary format
- **BZIP2, GZIP compression**: Transparent decompression

**Output formats:**
- CSV
- JSON

---

#### SQL Syntax (Simplified)

**SELECT statement:**
```sql
SELECT [columns] FROM S3Object [alias] WHERE [condition] LIMIT [n]
```

**Examples:**

```sql
-- 1. Select specific columns
SELECT name, age, city FROM S3Object WHERE age > 30

-- 2. Count rows
SELECT COUNT(*) FROM S3Object WHERE status = 'active'

-- 3. Aggregate functions
SELECT AVG(price), MAX(quantity) FROM S3Object

-- 4. String functions
SELECT * FROM S3Object WHERE LOWER(name) LIKE '%smith%'

-- 5. Limit results
SELECT * FROM S3Object LIMIT 1000
```

**Supported functions:**
- Aggregates: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`
- String: `LOWER`, `UPPER`, `SUBSTRING`, `TRIM`, `CHAR_LENGTH`
- Operators: `=`, `!=`, `<`, `>`, `<=`, `>=`, `LIKE`, `IN`, `BETWEEN`
- Logical: `AND`, `OR`, `NOT`

---

#### Practical Example: CSV Query

**File: sales-2026.csv (5 GB)**

```csv
order_id,customer_id,product_id,quantity,price,state,date
1001,C123,P456,2,29.99,CA,2026-01-15
1002,C456,P789,1,49.99,NY,2026-01-15
1003,C789,P123,5,9.99,CA,2026-01-16
... 50 million more rows
```

**Query: California sales totals**

```python
import boto3

s3 = boto3.client('s3')

response = s3.select_object_content(
    Bucket='sales-data',
    Key='sales-2026.csv',
    ExpressionType='SQL',
    Expression="""
        SELECT state, SUM(quantity * price) AS total_sales
        FROM S3Object
        WHERE state = 'CA'
    """,
    InputSerialization={
        'CSV': {
            'FileHeaderInfo': 'USE',  # First row is header
            'FieldDelimiter': ',',
            'RecordDelimiter': '\n'
        },
        'CompressionType': 'NONE'
    },
    OutputSerialization={
        'JSON': {
            'RecordDelimiter': '\n'
        }
    }
)

# Stream results
for event in response['Payload']:
    if 'Records' in event:
        records = event['Records']['Payload'].decode('utf-8')
        print(records)  # {"state":"CA","total_sales":15000000.50}
```

**Result**: Returns only aggregated totals instead of downloading 5 GB file.

---

#### Parquet Performance Advantage

**Columnar format benefits:**

```
CSV Layout (Row-based):
  Row 1: [id][name][age][city][state][salary]
  Row 2: [id][name][age][city][state][salary]
  Row 3: [id][name][age][city][state][salary]

  Query: SELECT state FROM S3Object
  Must read: ALL columns from ALL rows

Parquet Layout (Column-based):
  Column: id    → [1,2,3,...]
  Column: name  → ['Alice','Bob','Charlie',...]
  Column: age   → [30,25,35,...]
  Column: state → ['CA','NY','CA',...]

  Query: SELECT state FROM S3Object
  Must read: ONLY state column (ignores others)

Performance:
  CSV:     Read 100% of data
  Parquet: Read 10% of data (only state column)
  Speedup: 10x faster, 90% less data scanned
```

**Best practice**: Convert large datasets to Parquet for S3 Select queries.

---

#### Glacier Select

**What is Glacier Select?**

S3 Select for **archived objects in Glacier**, allowing SQL queries on archived data without restoring the entire object to S3 Standard.

**Traditional Glacier retrieval:**
```
1. Initiate restore: Request entire 10 GB archive
2. Wait: 3-5 hours (Glacier Flexible Retrieval)
3. Download: Entire 10 GB restored to S3 Standard
4. Query: Filter locally
5. Cost: Restore fee + storage for restored copy

Total time: Hours
Total cost: High (restore + temp storage)
```

**Glacier Select approach:**
```
1. Query directly: SELECT ... FROM archive
2. Wait: Same retrieval time (3-5 hours)
3. Receive: Only matching rows (e.g., 100 MB)
4. Cost: Restore fee for data scanned (not entire archive)

Total time: Same hours, but less data transferred
Total cost: Lower (no temp storage, less transfer)
```

**Use case**: Query archived logs or historical data without full restoration.

---

#### S3 Select Limitations

**Size limits:**
- Object size: No hard limit, but large objects may time out
- Query complexity: Simple SQL only (no JOINs, subqueries, window functions)
- Result size: No limit, but streams back

**Performance limitations:**
- Not suitable for complex analytics (use Athena instead)
- No indexes (full scan always)
- No caching (every query rescans)

**When to use S3 Select:**
- Simple filtering, aggregation
- One-off ad-hoc queries
- Bandwidth/cost constrained
- Small subset of large object

**When NOT to use (use Athena instead):**
- Complex queries (JOINs, subqueries)
- Multiple objects (S3 Select = one object at a time)
- Repeated queries on same data (Athena caches results)
- Advanced analytics (window functions, CTEs)

---

### S3 Transfer Acceleration

**What is S3 Transfer Acceleration?**

A feature that speeds up **uploads to S3** by routing data through **AWS CloudFront edge locations** instead of directly to the S3 bucket region.

**Why it exists:**
- Long-distance internet connections are slow and unreliable
- Direct uploads to distant regions suffer high latency
- Global applications need fast uploads from anywhere
- Packet loss over public internet reduces throughput

**Problems it solves:**
- **Faster uploads**: Up to 100x faster for distant users
- **Reliability**: AWS backbone more reliable than public internet
- **Global reach**: Fast uploads from any location
- **No infrastructure changes**: Just change the endpoint URL

---

#### How Transfer Acceleration Works

```
┌────────────────────────────────────────────────────────────┐
│        Traditional Upload (No Acceleration)                │
└────────────────────────────────────────────────────────────┘

User in Tokyo
     │
     │  Public Internet
     │  (High latency, packet loss)
     │  Multiple hops: ISP → Transit → AWS
     │
     ▼
S3 Bucket in us-east-1 (Virginia)

Upload time: 10 minutes for 1 GB
Throughput: ~1.7 MB/s


┌────────────────────────────────────────────────────────────┐
│          Transfer Acceleration                             │
└────────────────────────────────────────────────────────────┘

User in Tokyo
     │
     │  Short distance
     │  (Low latency, ~10ms)
     ▼
CloudFront Edge in Tokyo
     │
     │  AWS Private Network
     │  (Optimized backbone, no packet loss)
     │  TCP optimizations, parallel streams
     ▼
S3 Bucket in us-east-1 (Virginia)

Upload time: 2 minutes for 1 GB
Throughput: ~8.5 MB/s
Speedup: 5x faster
```

**Key concept**: Data travels quickly from user to nearest edge location (short distance), then uses AWS's optimized network (fast, reliable) to reach the S3 bucket.

---

#### Configuration

**Enable Transfer Acceleration:**

```
1. Navigate to S3 bucket settings
2. Enable Transfer Acceleration
   ⚠️ Bucket name must be DNS-compliant (no dots, no uppercase)
3. Use acceleration endpoint instead of standard endpoint

Standard endpoint:
  https://my-bucket.s3.us-east-1.amazonaws.com

Acceleration endpoint:
  https://my-bucket.s3-accelerate.amazonaws.com

That's it! No code changes beyond URL.
```

**Code example:**

```python
import boto3

# Standard upload (slow from distant locations)
s3 = boto3.client('s3', region_name='us-east-1')
s3.upload_file('large-file.zip', 'my-bucket', 'large-file.zip')

# Accelerated upload (fast from anywhere)
s3 = boto3.client('s3', region_name='us-east-1',
                  config=boto3.session.Config(s3={'use_accelerate_endpoint': True}))
s3.upload_file('large-file.zip', 'my-bucket', 'large-file.zip')
# Only difference: use_accelerate_endpoint=True
# S3 client automatically uses s3-accelerate.amazonaws.com
```

---

#### Performance Gains

**Real-world benchmarks:**

```
From distant locations (5000+ km from bucket region):
  - File size: 1 GB
  - Standard upload: 10-30 minutes
  - Accelerated upload: 2-5 minutes
  - Speedup: 3-10x

From nearby locations (<1000 km):
  - Minimal improvement (1.1-1.5x)
  - May even be slightly slower (overhead)

Optimal use case:
  - Global user base uploading to centralized bucket
  - Large files (>100 MB)
  - Users >3000 km from bucket region
```

**Speed Comparison Tool**: AWS provides a tool to test acceleration benefits for your location:
```
https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html
```

---

#### Cost Considerations

**Pricing:**

```
Standard S3 upload: FREE (no charge for PUT requests data transfer IN)

Transfer Acceleration:
  - Data transfer IN via acceleration: $0.04 - $0.08 per GB (varies by region)
  - PUT requests: Standard S3 pricing ($0.005 per 1,000)

Example: Upload 100 GB from Tokyo to us-east-1
  Standard: $0 (free)
  Accelerated: 100 GB × $0.04 = $4.00

Worth it if:
  - Time saved is valuable (customer experience, SLA)
  - Upload reliability is critical (fewer failures)
  - Standard upload would fail/timeout frequently
```

**Cost optimization:**
- Use acceleration only for distant users
- Detect user location and conditionally use acceleration endpoint
- Disable for users near bucket region

---

#### Transfer Acceleration vs CloudFront

```
┌──────────────────────────────────────────────────────────────┐
│  Transfer Acceleration  │    CloudFront (for downloads)      │
├─────────────────────────┼────────────────────────────────────┤
│ Speeds up: Uploads      │ Speeds up: Downloads               │
│ (PUT, POST)             │ (GET)                              │
├─────────────────────────┼────────────────────────────────────┤
│ Use case: User uploads  │ Use case: Content delivery         │
│ to S3                   │ to users                           │
├─────────────────────────┼────────────────────────────────────┤
│ No caching              │ Edge caching (faster repeated      │
│                         │ requests)                          │
├─────────────────────────┼────────────────────────────────────┤
│ Cost: Per GB uploaded   │ Cost: Per GB downloaded            │
└──────────────────────────────────────────────────────────────┘
```

**Best practice**: Use BOTH together:
- Transfer Acceleration for uploads (user → S3)
- CloudFront for downloads (S3 → user)

---

#### Limitations

**Bucket name restrictions:**
```
✅ Valid: my-bucket-name
✅ Valid: my-bucket-123
❌ Invalid: MyBucket (uppercase)
❌ Invalid: my.bucket.name (dots)
❌ Invalid: bucket_name (underscore at bucket level)

Reason: Acceleration uses CNAME DNS, requires DNS-compliant names
```

**Operation support:**
- ✅ Supported: PUT, POST, Multipart Upload
- ❌ Not supported: GET, DELETE, LIST (use standard endpoint)

**Regional limitations:**
- Not all regions support Transfer Acceleration
- Check AWS documentation for supported regions

---

### Presigned URLs

**What are Presigned URLs?**

Temporary, **signed URLs** that grant time-limited access to S3 objects without requiring AWS credentials or modifying bucket permissions.

**Why they exist:**
- Public access to entire buckets is insecure
- Users shouldn't have AWS credentials for temporary access
- Need time-limited sharing (expiring links)
- Want to grant access without changing bucket policies

**Problems they solve:**
- **Temporary access**: Grant access for limited time (minutes to days)
- **No credential sharing**: Users don't need AWS keys
- **Fine-grained control**: Per-object, per-operation access
- **Secure downloads**: Share private files without making bucket public
- **Secure uploads**: Allow users to upload without write permissions to bucket

---

#### How Presigned URLs Work

```
┌────────────────────────────────────────────────────────────┐
│              Presigned URL Creation                        │
└────────────────────────────────────────────────────────────┘

Application with AWS credentials
     │
     │  Generate presigned URL request
     │  - Bucket: my-private-bucket
     │  - Key: documents/report.pdf
     │  - Expiration: 1 hour
     │  - Operation: GET
     ▼
AWS SDK signs URL with your credentials
  - Adds signature parameters
  - Adds expiration timestamp
  - Returns URL
     │
     ▼
Presigned URL (example):
https://my-private-bucket.s3.us-east-1.amazonaws.com/documents/report.pdf?
X-Amz-Algorithm=AWS4-HMAC-SHA256&
X-Amz-Credential=AKIA.../20260206/us-east-1/s3/aws4_request&
X-Amz-Date=20260206T120000Z&
X-Amz-Expires=3600&
X-Amz-SignedHeaders=host&
X-Amz-Signature=abc123def456...

User clicks URL
     │
     ▼
S3 validates signature
  ✅ Signature valid?
  ✅ Expiration not passed?
  ✅ Requested operation matches signed operation?
     │
     └─ Yes → Grant access
     └─ No → 403 Forbidden
```

**Key insight**: The URL contains a cryptographic signature proving it was created by someone with valid AWS credentials, but the user doesn't need those credentials.

---

#### Generating Presigned URLs

**For downloads (GET):**

```python
import boto3
from datetime import timedelta

s3_client = boto3.client('s3')

# Generate presigned URL for download
url = s3_client.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'my-private-bucket',
        'Key': 'documents/report.pdf'
    },
    ExpiresIn=3600  # 1 hour in seconds
)

print(url)
# Share this URL with user via email, chat, etc.
# User can download the file for 1 hour without AWS credentials
```

**For uploads (PUT):**

```python
# Generate presigned URL for upload
url = s3_client.generate_presigned_url(
    'put_object',
    Params={
        'Bucket': 'user-uploads',
        'Key': f'uploads/{user_id}/photo.jpg',
        'ContentType': 'image/jpeg'  # Enforce content type
    },
    ExpiresIn=300  # 5 minutes
)

# User can upload ONLY to this specific key
# URL expires after 5 minutes
```

---

#### Common Use Cases

**1. Secure File Sharing**

```
Scenario: Share confidential document with client

Traditional approach:
  - Make S3 bucket public (❌ insecure)
  - OR create IAM user for client (❌ complex)

Presigned URL approach:
  1. Generate presigned URL (1 day expiration)
  2. Email URL to client
  3. Client downloads using URL (no AWS account needed)
  4. URL expires after 1 day (automatic revocation)
```

---

**2. Direct Browser Uploads**

```
Scenario: User uploads profile photo from web app

Traditional approach:
  User → Web server → S3
  Problems:
    - Server must handle upload (bandwidth, processing)
    - Slower (two-hop: user→server→S3)
    - Server becomes bottleneck

Presigned URL approach:
  1. User requests upload permission
  2. Server generates presigned PUT URL
  3. User uploads DIRECTLY to S3 (browser → S3)
  4. S3 validates signature
  5. Server notified of completion (S3 event)

Benefits:
  - Faster (direct to S3)
  - Server offloaded (no upload bandwidth used)
  - Scalable (S3 handles uploads)
```

**Example frontend code:**

```javascript
// 1. Get presigned URL from backend
const response = await fetch('/api/get-upload-url', {
  method: 'POST',
  body: JSON.stringify({ fileName: 'photo.jpg' })
});
const { uploadUrl } = await response.json();

// 2. Upload file directly to S3 using presigned URL
const file = document.getElementById('file-input').files[0];
await fetch(uploadUrl, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': file.type }
});

// Upload complete! File is in S3.
```

---

**3. Time-Limited Access to Videos**

```
Scenario: Paid video platform (only paying users can watch)

Implementation:
  1. User authenticates with your app
  2. App verifies subscription status
  3. If valid, generate presigned URL (2-hour expiration)
  4. Return URL to video player
  5. Player streams from S3 using presigned URL

Benefits:
  - S3 serves video (no video server needed)
  - URLs expire (can't share indefinitely)
  - No public bucket access (secure)
```

---

**4. Temporary Write Access**

```
Scenario: Allow external partner to upload data

Configuration:
  1. Partner has no AWS account
  2. Generate presigned PUT URL for specific S3 path
  3. Share URL with partner
  4. Partner uploads data (API or curl)
  5. URL expires after defined period

Example with curl:
  curl -X PUT -T data.csv "https://bucket.s3.amazonaws.com/uploads/partner-data.csv?X-Amz-..."
```

---

#### Presigned URL Security

**Best practices:**

```
✅ DO:
  - Use short expiration times (minutes to hours, not days)
  - Generate URL per request (don't reuse)
  - Use HTTPS (presigned URLs work with HTTP but are insecure)
  - Validate user authorization BEFORE generating URL
  - Log URL generation (audit trail)

❌ DON'T:
  - Share URLs publicly (they grant access to anyone)
  - Use long expirations (years) for sensitive data
  - Generate URLs from client-side (exposes AWS credentials)
  - Reuse same URL for multiple users (tracking impossible)
```

**Expiration considerations:**

```
Too short (seconds):
  - User may not complete download/upload
  - Poor user experience (timeouts)

Too long (days):
  - URL may be shared/leaked
  - Access persists beyond intended duration

Recommended:
  - Downloads: 5 minutes - 1 hour (depending on file size)
  - Uploads: 5-15 minutes
  - Streaming: Duration of content + buffer
```

---

#### Presigned URL vs Presigned POST

```
┌──────────────────────────────────────────────────────────────┐
│  Presigned URL (PUT)    │    Presigned POST                  │
├─────────────────────────┼────────────────────────────────────┤
│ Simple GET/PUT          │ HTML form uploads                  │
├─────────────────────────┼────────────────────────────────────┤
│ URL-based               │ Form-based (multipart/form-data)   │
├─────────────────────────┼────────────────────────────────────┤
│ Single file             │ Can include metadata, conditions   │
├─────────────────────────┼────────────────────────────────────┤
│ Simpler                 │ More flexible (file size limits,   │
│                         │ content type enforcement)          │
└──────────────────────────────────────────────────────────────┘
```

**Presigned POST example:**

```python
# Generate presigned POST (form-based upload)
response = s3_client.generate_presigned_post(
    'my-bucket',
    'uploads/${filename}',
    Fields={'acl': 'private', 'Content-Type': 'image/jpeg'},
    Conditions=[
        {'acl': 'private'},
        {'Content-Type': 'image/jpeg'},
        ['content-length-range', 1024, 10485760]  # 1KB - 10MB only
    ],
    ExpiresIn=600
)

# Returns: { 'url': '...', 'fields': {...} }
# Use in HTML form
```

**HTML form:**

```html
<form action="${response.url}" method="post" enctype="multipart/form-data">
  <input type="hidden" name="key" value="${response.fields.key}">
  <input type="hidden" name="AWSAccessKeyId" value="${response.fields.AWSAccessKeyId}">
  <input type="hidden" name="policy" value="${response.fields.policy}">
  <input type="hidden" name="signature" value="${response.fields.signature}">
  <input type="file" name="file">
  <input type="submit" value="Upload">
</form>
```

---

#### Limitations

**Maximum expiration:**
- Presigned URLs using IAM user credentials: 7 days max
- Presigned URLs using STS temporary credentials: Limited by STS token expiration (typically 1-12 hours)
- Presigned URLs using IAM role: 6 hours max (for `generate_presigned_url`)

**Signature version:**
- Modern signature (SigV4): 7-day expiration
- Legacy signature (SigV2): Deprecated, don't use

**Credential invalidation:**
```
⚠️ If AWS credentials used to sign URL are revoked/rotated:
   - Existing presigned URLs become invalid immediately
   - Even if expiration time hasn't passed

Example:
  1. Generate URL with expiration = 24 hours
  2. Rotate IAM user access key (security requirement)
  3. URL stops working immediately (signature mismatch)
```

---

### Bucket Policies

**What are Bucket Policies?**

**JSON-based access control policies** attached to S3 buckets that define who can access bucket resources and what actions they can perform.

**Why they exist:**
- IAM policies are user-centric (attached to users/roles)
- Need resource-centric policies (attached to buckets)
- Cross-account access requires bucket-level permissions
- Public access requires bucket policies (IAM alone isn't enough)

**Problems they solve:**
- **Cross-account access**: Allow other AWS accounts to access bucket
- **Public/anonymous access**: Grant access without AWS credentials
- **IP-based restrictions**: Allow access only from specific IPs
- **Encryption enforcement**: Require encrypted uploads
- **Centralized permission management**: One place for all bucket access rules

---

#### Bucket Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadAccess",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

**Components:**

1. **Version**: Policy language version (always "2012-10-17")
2. **Statement**: Array of permission rules
3. **Sid**: Statement ID (optional, for documentation)
4. **Effect**: `Allow` or `Deny`
5. **Principal**: Who gets the permission (user, account, service, or `*` for everyone)
6. **Action**: What they can do (`s3:GetObject`, `s3:PutObject`, etc.)
7. **Resource**: What they can access (bucket ARN, object ARNs)
8. **Condition**: Optional constraints (IP address, encryption, MFA, etc.)

---

#### Common Bucket Policy Patterns

**Pattern 1: Public Read Access (Static Website)**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-website-bucket/*"
  }]
}
```

**Use case**: Hosting static website, allowing anyone to download files.

**Security note**: ⚠️ Makes ALL objects publicly readable. Use carefully.

---

**Pattern 2: Cross-Account Access**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAccountBAccess",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111122223333:root"
    },
    "Action": [
      "s3:GetObject",
      "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::shared-bucket/*"
  }]
}
```

**Use case**: Account A owns bucket, Account B can read/write objects.

**Important**: Account B users also need IAM permissions in their own account (dual permission requirement).

---

**Pattern 3: Require Encryption at Upload**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedUploads",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringNotEquals": {
        "s3:x-amz-server-side-encryption": "AES256"
      }
    }
  }]
}
```

**Result**: Uploads without `x-amz-server-side-encryption: AES256` header are rejected.

**Use case**: Compliance requirement for data encryption.

---

**Pattern 4: IP Allowlist (VPN/Office Only)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFromOfficeIP",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": [
            "203.0.113.0/24",
            "198.51.100.0/24"
          ]
        }
      }
    },
    {
      "Sid": "DenyAllOthers",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": [
            "203.0.113.0/24",
            "198.51.100.0/24"
          ]
        }
      }
    }
  ]
}
```

**Use case**: Bucket accessible only from corporate network.

---

**Pattern 5: Require MFA for Delete Operations**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireMFAForDelete",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:DeleteObject",
    "Resource": "arn:aws:s3:::critical-data/*",
    "Condition": {
      "Bool": {
        "aws:MultiFactorAuthPresent": "false"
      }
    }
  }]
}
```

**Result**: DELETE operations without MFA are blocked.

**Use case**: Extra protection for sensitive data.

---

**Pattern 6: CloudFront-Only Access (Private Content)**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

**Use case**: Block direct S3 access, allow only via CloudFront distribution.

---

#### Bucket Policy vs IAM Policy

```
┌──────────────────────────────────────────────────────────────┐
│     IAM Policy          │      Bucket Policy                 │
├─────────────────────────┼────────────────────────────────────┤
│ Attached to: IAM users, │ Attached to: S3 bucket             │
│ roles, groups           │                                    │
├─────────────────────────┼────────────────────────────────────┤
│ Scope: What this user   │ Scope: Who can access this bucket  │
│ can do                  │                                    │
├─────────────────────────┼────────────────────────────────────┤
│ Cross-account: No       │ Cross-account: Yes                 │
│                         │ (can grant to other accounts)      │
├─────────────────────────┼────────────────────────────────────┤
│ Public access: No       │ Public access: Yes                 │
│                         │ (Principal: "*")                   │
├─────────────────────────┼────────────────────────────────────┤
│ Managed by: IAM team    │ Managed by: Bucket owner           │
└──────────────────────────────────────────────────────────────┘
```

**Both required for access:**

```
User from Account B wants to access bucket in Account A

Account A (Bucket owner):
  Bucket Policy: Allow Account B access ✅

Account B (User's account):
  IAM Policy: Allow user to access Account A bucket ✅

Both required: Bucket policy + IAM policy
Missing either: Access Denied
```

---

#### Policy Evaluation Logic

**How AWS decides Allow or Deny:**

```
┌────────────────────────────────────────────────────────────┐
│               S3 Access Decision Flow                      │
└────────────────────────────────────────────────────────────┘

Request: User A tries to GetObject from bucket

Step 1: Any explicit DENY?
  ├─ Yes → Access Denied (stop here)
  └─ No → Continue

Step 2: Any explicit ALLOW?
  Check:
    ├─ User's IAM policy
    ├─ Bucket policy
    ├─ S3 Access Point policy
    ├─ Resource-based policies

  ├─ Yes → Access Allowed
  └─ No → Default Deny (Access Denied)

Rule: Explicit DENY always wins
      Need at least one explicit ALLOW
      Default is DENY
```

**Example:**

```
Scenario: User has IAM policy allowing s3:GetObject
          Bucket policy denies s3:GetObject from user's IP

Result: Access Denied (Deny wins)

Scenario: User has IAM policy allowing s3:GetObject
          Bucket policy has no statements about this user

Result: Access Allowed (IAM Allow + no Deny)

Scenario: User has no IAM policy
          Bucket policy allows s3:GetObject for user's account

Result: Access Allowed (Bucket policy Allow + no Deny)
```

---

### ACLs (Access Control Lists) - Legacy

**What are S3 ACLs?**

An **older, legacy access control mechanism** for S3 that predates bucket policies and IAM. ACLs are per-object or per-bucket permissions.

**Why they exist:**
- S3's original permission system (before bucket policies)
- Backward compatibility with old applications

**Why they're legacy:**
- Limited functionality (no conditions, complex policies)
- Bucket policies are more powerful and flexible
- AWS recommends disabling ACLs for new buckets

---

#### ACL Grants

**Predefined permissions:**

- `READ`: List objects (bucket) or read object (object)
- `WRITE`: Create/delete objects (bucket only)
- `READ_ACP`: Read ACL
- `WRITE_ACP`: Modify ACL
- `FULL_CONTROL`: All permissions

**Grantees:**

- Specific AWS account (canonical user ID)
- Predefined groups:
  - `AllUsers`: Public access (anyone)
  - `AuthenticatedUsers`: Any AWS account
  - `LogDelivery`: S3 log delivery service

---

#### When ACLs Are Still Used

**Valid use case: S3 Server Access Logging**

```
Log delivery bucket requires ACL:
  - Grantee: LogDelivery group
  - Permission: WRITE

S3 writes access logs to this bucket
ACL required (bucket policy doesn't work for this)
```

**Most other cases:** Use bucket policies instead.

---

#### AWS Recommendation: Disable ACLs

**Modern best practice:**

```
Bucket settings:
  Object Ownership: Bucket owner enforced

Effect:
  - ACLs disabled for new objects
  - All objects owned by bucket owner
  - Simpler permission model (bucket policies only)
  - No ACL confusion
```

**Migration path:**
1. Audit existing ACL usage
2. Convert ACL permissions to bucket policies
3. Enable "Bucket owner enforced"
4. ACLs no longer apply

---

### Block Public Access

**What is Block Public Access?**

A **protective layer** that overrides bucket policies and ACLs to prevent accidental public exposure of S3 buckets and objects.

**Why it exists:**
- Accidental public buckets cause data breaches (major security risk)
- Developers may add public policies unintentionally
- Auditing all bucket policies is difficult
- Need a safety net to prevent public access

**Problems it solves:**
- **Data breach prevention**: Blocks public access even if policy allows it
- **Defense in depth**: Extra security layer beyond policies
- **Compliance**: Meet requirements for private data
- **Mistake protection**: Overrides incorrect policies

---

#### Block Public Access Settings

**Four settings (applied at account or bucket level):**

```
1. BlockPublicAcls
   Blocks: PUT operations that set public ACLs on objects/bucket

2. IgnorePublicAcls
   Ignores: All public ACLs (even existing ones)
   Effect: Public ACLs have no effect

3. BlockPublicPolicy
   Blocks: PUT operations that create public bucket policies

4. RestrictPublicBuckets
   Restricts: Public bucket policies to only authorized AWS principals
   Effect: Public policies work only for AWS users, not anonymous
```

**Recommended configuration (most secure):**

```
✅ BlockPublicAcls: ON
✅ IgnorePublicAcls: ON
✅ BlockPublicPolicy: ON
✅ RestrictPublicBuckets: ON

Result: Bucket is completely private, cannot be made public
```

---

#### How Block Public Access Works

```
┌────────────────────────────────────────────────────────────┐
│         Attempt to Make Bucket Public                      │
└────────────────────────────────────────────────────────────┘

Developer adds bucket policy:
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*"
}

Block Public Access enabled:
  ├─ BlockPublicPolicy: ON
  │    └─ Result: Policy update REJECTED
  │         Error: "Access Denied - Public policy blocked"
  │
  └─ If policy somehow exists:
       RestrictPublicBuckets: ON
         └─ Result: Policy IGNORED for anonymous users
              Effect: Bucket remains private despite policy

Protection: Even if mistake happens, data stays private
```

---

#### Account-Level vs Bucket-Level

**Account-level Block Public Access:**

```
Applies to: ALL buckets in the account (existing + future)
Cannot be overridden: Individual buckets can't disable it
Use case: Organization-wide security policy

Configuration:
  Account Settings → Block Public Access → All settings ON

Effect: No bucket in account can ever be public
```

**Bucket-level Block Public Access:**

```
Applies to: Individual bucket only
Can be different: Per bucket
Use case: Most buckets private, some public (e.g., website)

Configuration:
  Bucket → Permissions → Block Public Access → Configure per bucket
```

**Best practice:**
- Account-level ON (default protection)
- Explicitly disable per-bucket only when necessary (e.g., static website hosting)
- Require approval for disabling (security review)

---

#### Public Access Indicator

**S3 Console shows public status:**

```
Bucket list:
  ┌────────────────────┬────────┐
  │ Bucket Name        │ Access │
  ├────────────────────┼────────┤
  │ my-private-bucket  │ 🔒     │  (Private)
  │ my-website-bucket  │ 🌐     │  (Public)
  │ my-shared-bucket   │ ⚠️     │  (Public warning)
  └────────────────────┴────────┘

🔒 Private: Block Public Access enabled
🌐 Public: Publicly accessible (policy/ACL grants access)
⚠️  Warning: Potential public access
```

---

### S3 Storage Lens

**What is S3 Storage Lens?**

An **analytics and insights service** that provides organization-wide visibility into S3 usage, storage patterns, and cost optimization opportunities.

**Why it exists:**
- Organizations have thousands of S3 buckets across accounts
- Understanding total storage usage is difficult
- Finding cost optimization opportunities manually is impractical
- Need centralized visibility for governance

**Problems it solves:**
- **Visibility**: Understand storage across entire organization
- **Cost optimization**: Identify savings opportunities (storage class transitions, deletions)
- **Usage trends**: Track growth, access patterns
- **Anomaly detection**: Spot unusual activity
- **Recommendations**: Automated suggestions for improvements

---

#### Storage Lens Dashboards

**Default Dashboard (Free):**

```
Metrics included:
  - Total storage (bytes)
  - Object count
  - Storage by class (Standard, IA, Glacier, etc.)
  - Average object size
  - Encryption status

Granularity: Daily, at bucket level
Retention: 14 days
Cost: Free
```

**Advanced Dashboard (Paid):**

```
Additional metrics:
  - Activity metrics (GET, PUT request counts)
  - Detailed cost metrics
  - Access patterns (frequent vs infrequent)
  - Lifecycle rule effectiveness
  - Replication metrics

Granularity: Daily, configurable prefixes
Retention: 15 months
Cost: ~$0.20 per million objects analyzed
```

---

#### Dashboard Structure

```
┌────────────────────────────────────────────────────────────┐
│                Storage Lens Hierarchy                      │
└────────────────────────────────────────────────────────────┘

Organization (All accounts)
  │
  ├─ Account 1 (Production)
  │   ├─ Region: us-east-1
  │   │   ├─ Bucket: app-data
  │   │   │   └─ Prefix: logs/
  │   │   └─ Bucket: user-uploads
  │   │
  │   └─ Region: eu-west-1
  │       └─ Bucket: eu-data
  │
  └─ Account 2 (Development)
      └─ Region: us-west-2
          └─ Bucket: dev-data

View metrics at any level:
  - Entire organization
  - Specific account
  - Specific region
  - Specific bucket
  - Specific prefix
```

---

#### Key Insights and Recommendations

**1. Storage Class Optimization**

```
Insight:
  "500 GB in Standard class not accessed in 90+ days"

Recommendation:
  "Transition to Standard-IA → Save $192/year"

Action:
  Create lifecycle policy to automate transition
```

---

**2. Incomplete Multipart Uploads**

```
Insight:
  "25 GB in incomplete multipart uploads (orphaned parts)"

Recommendation:
  "Delete after 7 days → Save $6/year"

Action:
  Create lifecycle rule to abort incomplete uploads
```

---

**3. Versioning Costs**

```
Insight:
  "bucket-xyz has 10 TB in non-current versions"
  "Only 2 TB in current versions"

Recommendation:
  "Delete non-current versions after 30 days → Save $2,300/year"

Action:
  Review versioning policy, add expiration for old versions
```

---

**4. Encryption Status**

```
Insight:
  "20% of objects unencrypted in compliance-bucket"

Recommendation:
  "Enable default encryption + deny unencrypted uploads"

Action:
  1. Enable bucket default encryption
  2. Add bucket policy requiring encryption
```

---

#### Storage Lens Metrics Export

**Export to S3 for deeper analysis:**

```
Configuration:
  Export format: CSV or Parquet
  Destination: s3://storage-lens-exports/
  Frequency: Daily

Use cases:
  - Query with Athena
  - Visualize with QuickSight
  - Custom analytics with Python/Spark
  - Integrate with BI tools
```

**Example Athena query:**

```sql
SELECT
  account_id,
  bucket_name,
  storage_class,
  SUM(total_storage_bytes) / 1073741824 AS total_gb,
  SUM(total_storage_bytes * 0.023) / 1073741824 AS monthly_cost_usd
FROM storage_lens_export
WHERE date = '2026-02-06'
  AND storage_class = 'STANDARD'
  AND last_access_date < DATE '2025-11-06'  -- Not accessed in 90 days
GROUP BY account_id, bucket_name, storage_class
ORDER BY total_gb DESC;
```

---

### S3 Express One Zone

**What is S3 Express One Zone?**

A **high-performance S3 storage class** designed for latency-sensitive applications, providing **single-digit millisecond** request latency and up to **10x faster** performance than S3 Standard.

**Why it exists:**
- S3 Standard latency: 10-100ms (sufficient for most apps)
- Some applications need <10ms latency (real-time, high-frequency)
- Traditional S3 not optimized for millions of requests per second
- Need co-location with compute for ultra-low latency

**Problems it solves:**
- **Ultra-low latency**: Sub-10ms request latency
- **High throughput**: Millions of requests per second per bucket
- **Consistent performance**: Predictable latency at scale
- **Compute co-location**: Available in same AZ as EC2/Lambda for lowest latency

---

#### Architecture Differences

```
┌────────────────────────────────────────────────────────────┐
│       S3 Standard       │    S3 Express One Zone           │
├─────────────────────────┼──────────────────────────────────┤
│ Regional service        │ Single AZ (zonal)                │
│ (spans 3+ AZs)          │                                  │
├─────────────────────────┼──────────────────────────────────┤
│ 99.999999999% durability│ 99.95% durability                │
│ (11 nines)              │ (lower, single AZ risk)          │
├─────────────────────────┼──────────────────────────────────┤
│ 10-100ms latency        │ <10ms latency (single-digit)     │
├─────────────────────────┼──────────────────────────────────┤
│ Thousands req/sec       │ Millions req/sec per bucket      │
├─────────────────────────┼──────────────────────────────────┤
│ Bucket naming: Global   │ Bucket naming: Regional          │
├─────────────────────────┼──────────────────────────────────┤
│ Pay per GB stored       │ Pay per GB stored + per GB       │
│                         │ processed (request data)         │
└──────────────────────────────────────────────────────────────┘
```

---

#### Performance Characteristics

**Latency comparison:**

```
Workload: 1 million GET requests

S3 Standard:
  - Average latency: 50ms
  - p99 latency: 200ms
  - Throughput: ~5,000 req/sec (per prefix)

S3 Express One Zone:
  - Average latency: 5ms (10x faster)
  - p99 latency: 15ms (13x faster)
  - Throughput: 100,000+ req/sec (per bucket)

Use case benefit:
  - Real-time analytics processing
  - High-frequency trading data
  - ML training with millions of small files
  - Interactive applications requiring instant access
```

---

#### Directory Buckets

**Key difference: S3 Express uses "Directory Buckets"**

```
Traditional S3 Bucket:
  - Name: my-bucket (global namespace)
  - Scope: Regional (spans 3+ AZs)
  - Naming: must be globally unique

S3 Express Directory Bucket:
  - Name: my-bucket--usw2-az1--x-s3 (includes AZ, suffix)
  - Scope: Single AZ only
  - Naming: Regional uniqueness (not global)

Directory Bucket naming:
  {base-name}--{az-id}--x-s3

  Example: analytics-data--use1-az2--x-s3
           └───┬────┘  └──┬───┘  └┬┘
           Base name    AZ ID   Suffix (required)
```

---

#### Optimized for Co-Location

**Best performance when compute and storage in same AZ:**

```
┌────────────────────────────────────────────────────────────┐
│            S3 Express Optimal Architecture                 │
└────────────────────────────────────────────────────────────┘

    Availability Zone: us-east-1a
   ┌─────────────────────────────────────────┐
   │                                         │
   │  ┌──────────────┐   ┌──────────────┐   │
   │  │ EC2 Instance │◄─►│ S3 Express   │   │
   │  │              │   │ Directory    │   │
   │  │ (Analytics)  │   │ Bucket       │   │
   │  └──────────────┘   └──────────────┘   │
   │        │                    │           │
   │        └────────────────────┘           │
   │        Low latency: <5ms                │
   │        (Same AZ, optimized network)     │
   └─────────────────────────────────────────┘

Cross-AZ access:
  EC2 in us-east-1b → S3 Express in us-east-1a
  Latency: Higher (~10-20ms) + cross-AZ data transfer costs
```

**Best practice**: Deploy compute resources in the same AZ as S3 Express bucket for lowest latency.

---

#### Use Cases

**1. Machine Learning Training (Small Files)**

```
Problem:
  - ML training: millions of small images (10-100 KB each)
  - S3 Standard: 50ms per request
  - Training time: Days (bottlenecked by I/O)

S3 Express solution:
  - Directory bucket in same AZ as training instances
  - 5ms per request (10x faster)
  - Training time: Hours (I/O no longer bottleneck)
  - Millions of requests per second supported
```

---

**2. Real-Time Analytics**

```
Application: Financial market data processing
  - Requirement: Sub-10ms latency for data retrieval
  - Request rate: 100,000 req/sec
  - Data size: Small objects (1-10 KB)

S3 Express:
  - Low latency: 3-7ms p50
  - High throughput: Handles request rate
  - Co-located with processing tier
```

---

**3. Interactive Applications**

```
Use case: Photo editing SaaS
  - Users load/save frequently (autosave every 30 sec)
  - Require instant feedback (<10ms)
  - High concurrency (thousands of users)

S3 Express:
  - Instant load/save operations
  - Consistent performance under load
  - Better user experience
```

---

#### Pricing Model

**S3 Express pricing (more expensive than Standard):**

```
Storage:
  - $0.16 per GB-month (7x more than Standard)

Data processing:
  - $0.0008 per GB processed (for GET requests)
  - Data uploaded (PUT): No processing charge

Requests:
  - GET: $0.40 per million (cheaper than Standard's $0.40 per million)
  - PUT: $2.00 per million (same as Standard)

Data transfer:
  - Same-AZ: FREE
  - Cross-AZ: $0.01 per GB (standard AWS cross-AZ rate)

Example cost: 100 GB storage, 10 million GETs (1 KB each)
  Storage: 100 × $0.16 = $16
  Data processing: 10 GB (10M × 1KB) × $0.0008 = $0.008
  Requests: 10 × $0.40 = $4
  Total: $20.008/month

Compare S3 Standard same workload:
  Storage: 100 × $0.023 = $2.30
  Requests: 10 × $0.0004 = $0.004
  Total: $2.304/month

Express cost: 8-9x higher, but worth it for latency-critical workloads
```

**When to use:**
- Latency is critical (<10ms required)
- High request rates (millions per second)
- Cost of latency exceeds storage cost (e.g., trading, real-time apps)

**When NOT to use:**
- Cost-sensitive workloads
- Latency not critical (batch processing)
- Standard S3 latency sufficient (most applications)

---

#### Limitations

**Durability trade-off:**
```
S3 Standard: 11 nines (99.999999999%) across 3+ AZs
S3 Express: 99.95% within single AZ

Risk: If AZ fails, data is lost (lower durability)

Mitigation:
  - Use for temporary/reproducible data
  - Replicate critical data to S3 Standard (different region)
  - Accept risk for performance-critical workloads
```

**Feature compatibility:**
```
✅ Supported:
  - Versioning
  - Encryption (SSE-S3, SSE-KMS)
  - Lifecycle policies
  - S3 Batch Operations
  - Access Points

❌ Not supported:
  - S3 Replication (CRR/SRR)
  - S3 Inventory
  - S3 Analytics
  - Public access (always private)
  - Bucket policies (use IAM policies + access points)
```

**Access patterns:**
```
Optimized for: Small objects, high request rate
Not optimized for: Large objects, sequential scans

Good fit:
  - Millions of 1-10 KB objects
  - Random access patterns
  - Low latency requirements

Poor fit:
  - Few large objects (GB+)
  - Sequential streaming
  - Cost-sensitive workloads
```

---

## S3 Summary

S3 is AWS's foundational object storage service with comprehensive features for:

✅ **Durability & Availability**: 11 nines durability, regional redundancy
✅ **Cost Optimization**: 7 storage classes from hot (Standard) to cold (Deep Archive)
✅ **Data Protection**: Versioning, Object Lock, Replication, Encryption
✅ **Access Control**: Bucket policies, ACLs, Access Points, Block Public Access
✅ **Performance**: Transfer Acceleration, S3 Select, S3 Express One Zone
✅ **Management**: Lifecycle policies, Inventory, Batch Operations, Storage Lens
✅ **Integration**: Event Notifications, Object Lambda, Presigned URLs

**Key design principles:**
- Infinite scalability (no capacity planning)
- Pay only for what you use
- Designed for 11 nines durability
- Multiple storage classes for cost optimization
- Rich ecosystem of features and integrations