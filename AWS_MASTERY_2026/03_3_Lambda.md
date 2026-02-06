# AWS Lambda - Conceptual Deep Dive

## Overview

**AWS Lambda** is AWS's serverless compute service that runs code without provisioning or managing servers. You upload code, AWS executes it in response to events, and you pay only for compute time consumed.

**The fundamental shift Lambda represents**:

**Traditional (EC2)**:
```
1. Provision servers
2. Install runtime (Node.js, Python)
3. Deploy code
4. Monitor servers
5. Scale capacity
6. Pay 24/7 (even if idle 90% of time)
```

**Serverless (Lambda)**:
```
1. Upload code
2. Configure trigger
3. AWS handles: runtime, scaling, availability, monitoring
4. Pay only when code executes (per millisecond)
```

**Core value proposition**: **Zero server management + pay-per-use + infinite scale**

---

## What Problems Lambda Solves

**Problem 1: Idle Capacity Cost**
```
Traditional:
- Run 5 EC2 instances 24/7 for API
- Peak usage: 2 hours/day (8%)
- Idle: 22 hours/day (92%)
- Cost: $730/month (full 24/7)

Lambda:
- Code runs only during requests
- Actual usage: 2 hours/day
- Cost: $15/month (92% savings)
```

**Problem 2: Scaling Overhead**
```
Traditional:
- Traffic spike: Must add instances (5 minutes lag)
- Auto Scaling: Configure policies, test, monitor
- Over-provision: Buy capacity for peaks (waste money)

Lambda:
- Automatic scaling: 0 to thousands of concurrent executions
- No configuration needed
- Scale-to-zero: No cost when idle
```

**Problem 3: Operational Burden**
```
Traditional:
- OS patching, security updates
- Runtime updates (Node.js, Python versions)
- Server monitoring, log management
- High availability setup (multi-AZ)

Lambda:
- AWS handles all operations
- Automatic patching, built-in HA
- Integrated logging (CloudWatch)
- Focus on code, not infrastructure
```

---

## How Lambda Works

### Execution Model

**Function invocation lifecycle**:
```
1. Event triggers Lambda (API call, S3 upload, etc.)
2. AWS finds/creates execution environment
   - Cold start (new environment): 100-1000ms
   - Warm start (reuse environment): 1-10ms
3. AWS loads your code
4. Your function executes
5. Function returns response
6. Environment kept warm (10-15 minutes) for reuse
7. After idle period, environment destroyed
```

### Lambda Execution Environment

**What AWS provides**:
```
Execution Environment:
├─ Operating System: Amazon Linux 2023
├─ Runtime: Node.js 20, Python 3.12, Java 17, Go 1.x, .NET 8, Ruby 3.3
├─ Memory: 128 MB to 10,240 MB (configurable)
├─ CPU: Proportional to memory (1,769 MB = 1 vCPU)
├─ Ephemeral Storage: 512 MB to 10,240 MB (/tmp directory)
├─ Timeout: 1 second to 15 minutes (configurable)
└─ Network: VPC optional, internet access by default
```

**Resource allocation**:
```
Memory allocation impacts everything:
- 128 MB: 0.08 vCPU, slow execution
- 1,769 MB: 1 full vCPU (breakpoint)
- 3,008 MB: ~1.7 vCPUs
- 10,240 MB: ~6 vCPUs

Key insight: More memory = more CPU = faster execution (often cheaper despite higher per-ms cost)
```

---

## Cold Start vs Warm Start

**Cold start**: First invocation or after idle period
```
Timeline:
- 0ms: Event arrives
- 0-100ms: AWS finds capacity, creates environment
- 100-300ms: Downloads code, initializes runtime
- 300-500ms: Executes initialization code (imports, connections)
- 500ms+: Handler function executes

Total cold start: 500-1500ms (depends on language, code size, dependencies)
```

**Warm start**: Reusing existing environment
```
Timeline:
- 0ms: Event arrives
- 0-10ms: Reuse existing environment
- 10ms+: Handler function executes

Total warm start: 10-50ms (100× faster than cold start)
```

### Cold Start Impact by Language

| Runtime | Cold Start | Why |
|---------|------------|-----|
| **Python** | 200-500ms | Lightweight interpreter |
| **Node.js** | 300-600ms | Fast JavaScript engine |
| **Go** | 400-800ms | Compiled binary (larger) |
| **Java** | 1000-3000ms | JVM startup overhead |
| **.NET** | 800-1500ms | CLR initialization |
| **Ruby** | 400-700ms | Moderate interpreter |

**Key takeaway**: Python and Node.js have fastest cold starts (preferred for latency-sensitive apps).

### Reducing Cold Starts

**1. Provisioned Concurrency**
```
Keep N execution environments always warm
- Cold start: 0ms (environments pre-initialized)
- Cost: Pay for provisioned instances 24/7 (like EC2)

Use case: Latency-critical applications
- API with <100ms SLA
- Real-time gaming backend
- Financial trading systems

Trade-off: Higher cost for guaranteed performance
```

**2. Increase Invocation Frequency**
```
High-traffic functions: Cold starts rare
- 100 req/sec → Environments constantly warm
- Cold start: <1% of invocations

Low-traffic functions: Frequent cold starts
- 1 req/hour → Every invocation likely cold start

Strategy: CloudWatch Events ping every 5 minutes (keep warm)
```

**3. Optimize Initialization Code**
```
Bad: Initialize on every invocation
def lambda_handler(event, context):
    db = connect_to_database()  # Slow, runs every time
    return process_request(db, event)

Good: Initialize outside handler
db = connect_to_database()  # Once per environment (cold start only)
def lambda_handler(event, context):
    return process_request(db, event)  # Fast, reuses connection

Result: Warm invocations 10× faster
```

**4. Reduce Package Size**
```
Smaller deployment package = faster download = faster cold start
- Remove unused dependencies
- Use Lambda Layers for common libraries
- Minimize node_modules (use production builds only)
```

---

## Lambda Invocation Models

### 1. Synchronous Invocation

**Request-response model** (caller waits for result):
```
Client → API Gateway → Lambda → Process → Return response → Client

Timeline:
- Client sends request
- Waits for Lambda to complete
- Receives response
- Total time: Lambda execution time + network latency

Use cases:
- REST APIs (user waiting for response)
- Mobile app backends
- GraphQL APIs
```

**Error handling**:
```
Lambda throws error:
- Client receives error response immediately
- Retry: Client's responsibility
```

### 2. Asynchronous Invocation

**Fire-and-forget model** (caller doesn't wait):
```
Client → SQS/SNS/S3 → Lambda (async) → Process
                     ↓
                  Client receives ACK immediately

Timeline:
- Client triggers event
- Receives acknowledgment immediately (event queued)
- Lambda processes in background
- Client doesn't wait or receive result

Use cases:
- Image processing (S3 upload triggers Lambda)
- Email sending (SNS topic triggers Lambda)
- Data processing pipelines
```

**Error handling**:
```
Lambda fails:
- AWS automatically retries (2 times)
- If still fails, event sent to Dead Letter Queue (DLQ)
- DLQ = SQS or SNS for failed events
```

### 3. Event Source Mapping (Stream/Queue Processing)

**Lambda polls event source**:
```
Lambda service polls:
- SQS queue
- Kinesis stream
- DynamoDB stream

Process:
1. Lambda polls source every second
2. Retrieves batch of records (up to 10,000)
3. Invokes function with batch
4. On success, deletes messages/advances stream position
5. On failure, retries batch

Use cases:
- SQS queue processing (background jobs)
- Real-time stream processing (Kinesis)
- Database change capture (DynamoDB Streams)
```

**Batch processing**:
```
Lambda receives: [message1, message2, ..., message100]
Processes: All messages in single invocation

Benefits:
- Fewer invocations = lower cost
- Higher throughput

Trade-off:
- One message fails → Entire batch retries (partial failure handling needed)
```

---

## Lambda Pricing (2026)

### Cost Components

**1. Request Charges**
```
First 1 million requests/month: Free
Additional requests: $0.20 per 1 million

Example:
- 10 million requests/month
- Cost: (10M - 1M) × $0.20 / 1M = $1.80
```

**2. Compute Charges** (Duration × Memory)
```
Unit: GB-seconds
Calculation: (Memory in GB) × (Duration in seconds)

Free tier: 400,000 GB-seconds/month

Pricing: $0.0000166667 per GB-second
(or $0.00001667 per GB-second for ARM/Graviton)

Example 1: 128 MB function, 100ms execution
- Memory: 0.125 GB
- Duration: 0.1 seconds
- GB-seconds: 0.125 × 0.1 = 0.0125
- Cost per invocation: 0.0125 × $0.0000166667 = $0.00000021 (0.02 cents)

Example 2: 1024 MB function, 1 second execution
- Memory: 1 GB
- Duration: 1 second
- GB-seconds: 1
- Cost per invocation: $0.0000166667 (0.0017 cents)
```

**3. Provisioned Concurrency Charges** (Optional)
```
Cost: $0.0000041667 per GB-second
(Nearly 4× more expensive than on-demand)

Example:
- 10 provisioned instances × 1024 MB × 730 hours
- = 10 × 1 GB × 2,628,000 seconds
- = 26,280,000 GB-seconds
- Cost: 26,280,000 × $0.0000041667 = $109.50/month

vs On-Demand (same usage):
- Cost: 26,280,000 × $0.0000166667 = $438/month

Savings? No, but you get zero cold starts
```

### Real-World Cost Examples

**Example 1: Low-Traffic API**
```
Usage:
- 100,000 requests/month
- Average: 512 MB, 200ms execution

Request cost:
- 100,000 requests (within free tier) = $0

Compute cost:
- 0.5 GB × 0.2 sec × 100,000 = 10,000 GB-seconds
- 10,000 GB-seconds (within free tier 400,000) = $0

Total: $0/month (entirely free tier)
```

**Example 2: Medium-Traffic API**
```
Usage:
- 10 million requests/month
- Average: 1024 MB, 500ms execution

Request cost:
- (10M - 1M free) × $0.20 / 1M = $1.80

Compute cost:
- 1 GB × 0.5 sec × 10M = 5,000,000 GB-seconds
- (5M - 400K free) × $0.0000166667 = $76.67

Total: $78.47/month
```

**Example 3: High-Traffic API (Compare EC2)**
```
Lambda:
- 100 million requests/month
- 1024 MB, 100ms avg

Cost:
- Requests: (100M - 1M) × $0.20 / 1M = $19.80
- Compute: 1 GB × 0.1 × 100M = 10M GB-sec → $166.67
- Total: $186.47/month

Equivalent EC2:
- Need ~5 instances (m5.large) to handle load
- 5 × $70/month = $350/month
- Lambda savings: 47%

But: Lambda includes HA, scaling, no ops overhead
```

---

## Lambda Concurrency

**What concurrency is**: Number of function instances executing simultaneously.

**Concurrency limits**:
```
Account-level default: 1,000 concurrent executions (per region)
- Can request increase to tens of thousands
- Soft limit, easily increased

Function-level:
- Reserved concurrency: Guarantee N executions always available
- Unreserved: Share from account pool
```

### Reserved vs Unreserved Concurrency

**Unreserved (default)**:
```
Account has 1,000 concurrency
- Function A uses 100
- Function B uses 200
- Remaining 700 available for other functions

Risk: One function can consume all concurrency (starve others)
```

**Reserved Concurrency**:
```
Function A: 200 reserved concurrency
- Guarantees: A can always run 200 concurrent executions
- Limits: A cannot exceed 200 (protects other functions)
- Reduces account pool: 1,000 - 200 = 800 remaining

Use case: Critical functions that must always have capacity
```

### Concurrency Scaling Example

```
API receiving traffic spike:
- Normal: 10 requests/second → 10 concurrent Lambdas (if 1 sec execution)
- Spike: 1,000 requests/second → 1,000 concurrent Lambdas

Lambda auto-scaling:
- 0-100: Immediate (burst capacity)
- 100-1000: Adds 500 per minute until limit reached
- Timeline: Reach 1,000 in ~2 minutes

If traffic > concurrency limit:
- Synchronous: Throttling errors (429 Too Many Requests)
- Asynchronous: Events queued, processed when capacity available
```

---

## Lambda Layers

**What Layers are**: Packages of libraries, dependencies, or custom runtimes that multiple functions can share.

**Without Layers**:
```
Function 1: 50 MB (code + dependencies)
Function 2: 50 MB (same dependencies)
Function 3: 50 MB (same dependencies)

Problems:
- Duplicate storage (150 MB total)
- Slower deployments (upload 50 MB each time)
- Harder to update dependencies (must update 3 functions)
```

**With Layers**:
```
Layer: Common dependencies (40 MB)
Function 1: 10 MB (just code)
Function 2: 10 MB (just code)
Function 3: 10 MB (just code)

All functions reference same Layer

Benefits:
- Reduced storage (70 MB vs 150 MB)
- Faster deployments (update Layer once, affects all functions)
- Smaller function packages (faster cold starts)
```

### Real-World Layer Use Cases

**1. Common Libraries**
```
Layer: Python pandas, numpy, scikit-learn (200 MB)
Functions: data-processor-1, data-processor-2, ml-inference

Update: New pandas version → Update Layer once, all functions get it
```

**2. Company-Wide Utilities**
```
Layer: Custom logging, authentication, API clients
Functions: All microservices (100+ functions)

Benefit: Consistent behavior, centralized updates
```

**3. Runtime Extensions**
```
Layer: Custom runtime (e.g., Rust, Swift)
Functions: Use languages not natively supported by Lambda
```

---

## Lambda Environment Variables

**What they are**: Key-value pairs available to function code at runtime.

**Use cases**:
```
Environment Variables:
- DATABASE_URL: Postgres connection string
- API_KEY: Third-party service key
- ENVIRONMENT: "production" or "staging"
- FEATURE_FLAGS: JSON string of enabled features

Code reads: os.environ['DATABASE_URL']
```

**Encryption**:
```
Sensitive data: Encrypted at rest with AWS KMS
- API keys, passwords, secrets
- Decrypted when function initializes
- Never stored in plaintext

Best practice: Use AWS Secrets Manager for highly sensitive data
- Rotate secrets automatically
- Audit access
```

---

## Lambda Best Practices

**1. Right-Size Memory**
```
Test with different memory allocations:
- 512 MB: Execution time = 2 seconds
- 1024 MB: Execution time = 1 second (2× faster)
- Cost comparison:
  - 512 MB: 0.5 GB × 2 sec = 1 GB-second
  - 1024 MB: 1 GB × 1 sec = 1 GB-second
  - Same cost, but 1024 MB finishes 2× faster

Recommendation: Use AWS Lambda Power Tuning tool to find optimal memory
```

**2. Reuse Connections**
```
Initialize outside handler:
- Database connections
- HTTP clients
- SDK clients

Reused across warm invocations (10× faster)
```

**3. Use Async for Background Tasks**
```
Synchronous API response:
- Process: 100ms
- Send email: 500ms
- Return response after 600ms (user waits)

Better approach:
- Process: 100ms
- Queue email task (SNS/SQS): 10ms
- Return response after 110ms
- Separate Lambda sends email asynchronously
```

**4. Monitor and Alert**
```
CloudWatch metrics:
- Invocation count
- Error rate
- Duration (p50, p99)
- Throttles
- Cold start percentage

Set alarms: Error rate > 1%, Duration p99 > 3 seconds
```

**5. Handle Partial Batch Failures**
```
Processing SQS batch:
- 100 messages in batch
- Message 50 fails

Bad: Return error → Entire batch retries (waste)

Good: reportBatchItemFailures
- Process 1-49: Success
- Process 50: Fail, report to AWS
- Process 51-100: Success
- Only message 50 retries
```

---

## When to Use Lambda vs EC2

| Use Case | Choose Lambda | Choose EC2 |
|----------|---------------|------------|
| **Short-lived tasks** (< 15 min) | ✅ Perfect fit | ❌ Wasteful |
| **Long-running** (hours) | ❌ Can't run >15 min | ✅ EC2 or ECS |
| **Variable traffic** | ✅ Scale-to-zero | ⚠️ Need Auto Scaling |
| **Consistent traffic 24/7** | ⚠️ Expensive at scale | ✅ More cost-effective |
| **Minimal ops** | ✅ Zero management | ❌ Patching, monitoring |
| **Stateful** | ❌ Stateless only | ✅ Supports state |
| **Custom runtime** | ⚠️ Limited runtimes | ✅ Install anything |
| **Low latency** | ⚠️ Cold starts | ✅ Always warm |

---

## Lambda Integration Patterns

**Pattern 1: API Backend (Lambda + API Gateway)**
```
Client → API Gateway → Lambda → DynamoDB → Response

Perfect for: REST APIs, GraphQL, mobile backends
```

**Pattern 2: Event Processing (S3 + Lambda)**
```
User uploads image → S3 → Lambda (resize) → S3 (thumbnails)

Perfect for: File processing, ETL pipelines
```

**Pattern 3: Stream Processing (Kinesis + Lambda)**
```
Application → Kinesis Stream → Lambda (process) → DynamoDB

Perfect for: Real-time analytics, log processing
```

**Pattern 4: Scheduled Jobs (EventBridge + Lambda)**
```
EventBridge (cron) → Lambda (cleanup old records)

Perfect for: Backups, report generation, maintenance tasks
```

---

**AWS Lambda transforms compute from "always-on servers" to "code that runs on-demand." For workloads that fit Lambda's model (short-lived, stateless, event-driven), it offers unparalleled simplicity, cost-efficiency, and scale.**