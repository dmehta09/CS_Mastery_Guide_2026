# AWS Security Services - Complete Conceptual Guide

## Table of Contents
1. [KMS (Key Management Service)](#kms-key-management-service)
2. [Secrets Manager](#secrets-manager)
3. [Certificate Manager (ACM)](#certificate-manager-acm)
4. [WAF (Web Application Firewall)](#waf-web-application-firewall)
5. [Shield](#shield)
6. [GuardDuty](#guardduty)
7. [Security Hub](#security-hub)
8. [Inspector](#inspector)
9. [Other Security Services](#other-security-services)

---

## Overview: The AWS Security Ecosystem

AWS security services form a **defense-in-depth architecture** — multiple layers of protection working together. Think of it like a medieval castle: you have outer walls (Shield), archers on guard (GuardDuty), gatekeepers checking visitors (WAF), secret vaults for treasures (KMS), and investigators tracking suspicious activity (Detective).

**Why these services exist:**
- **Shared Responsibility Model**: AWS secures the infrastructure; you secure what you put in it
- **Compliance Requirements**: Many industries require encryption, audit trails, and threat detection
- **Attack Surface Reduction**: As cloud adoption grew, so did sophisticated attacks
- **Automation**: Manual security doesn't scale; these services automate protection

```
┌─────────────────────────────────────────────────────────────┐
│                   AWS Security Layers                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Shield     │  │     WAF      │  │   Network    │      │
│  │ (DDoS Block) │  │ (App Filter) │  │   Firewall   │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │               │
│         └──────────────────┴──────────────────┘               │
│                      │                                        │
│              [Your Application]                               │
│                      │                                        │
│  ┌──────────────────┴──────────────────┐                    │
│  │  Data Layer Protection               │                    │
│  ├──────────────┬──────────────────────┤                    │
│  │     KMS      │   Secrets Manager    │                    │
│  │ (Encryption) │   (Credentials)      │                    │
│  └──────────────┴──────────────────────┘                    │
│                                                               │
│  ┌─────────────────────────────────────┐                    │
│  │  Monitoring & Detection              │                    │
│  ├──────────────┬──────────────────────┤                    │
│  │  GuardDuty   │   Security Hub       │                    │
│  │  Inspector   │   Detective          │                    │
│  └──────────────┴──────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

---

## KMS (Key Management Service)

### What It Is
KMS is AWS's **centralized key management system** for creating and controlling cryptographic keys used to encrypt your data. Think of it as a highly secure key factory and vault combined — it generates encryption keys, stores them safely, and controls who can use them.

### Why It Exists
**The Core Problem**: Without KMS, you'd need to:
- Generate encryption keys yourself (risky if done wrong)
- Store keys somewhere (hard drives? Environment variables? Both are terrible ideas)
- Rotate keys manually (tedious and error-prone)
- Track who used which key and when (audit nightmare)

**Real-World Scenario**: Imagine you're storing customer credit card data in S3. You need:
1. Strong encryption (AES-256)
2. Keys that attackers can't access even if they breach your S3 bucket
3. Proof that keys are only used by authorized services
4. Automatic key rotation for compliance

KMS solves all of this.

### How KMS Works: The Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    KMS Architecture                          │
└─────────────────────────────────────────────────────────────┘

  Application/Service                    KMS Service
        │                                     │
        │  1. "Encrypt this data"             │
        ├────────────────────────────────────>│
        │                                     │
        │                         ┌───────────┴──────────┐
        │                         │  Check Key Policy    │
        │                         │  (Is caller allowed?)│
        │                         └───────────┬──────────┘
        │                                     │
        │                         ┌───────────┴──────────┐
        │                         │  Use CMK to encrypt  │
        │                         │  (Key never leaves)  │
        │                         └───────────┬──────────┘
        │                                     │
        │  2. Encrypted data returned         │
        │<────────────────────────────────────┤
        │                                     │
        │  3. CloudTrail logs this event      │
        └─────────────────────────────────────┘
                                              │
                                    ┌─────────┴─────────┐
                                    │  Audit Trail:     │
                                    │  Who, What, When  │
                                    └───────────────────┘
```

**Critical Concept**: Your encryption keys **never leave KMS**. When you ask KMS to encrypt data, it uses the key internally and returns encrypted data. You never see or download the actual key.

---

### Customer Master Keys (CMK)

**What They Are**: The "root" encryption keys in KMS. All data encryption ultimately traces back to a CMK.

**Types of CMKs**:

1. **AWS Managed Keys** (Format: `aws/service-name`)
   - Created automatically when you enable encryption on AWS services
   - Example: When you enable S3 bucket encryption, AWS creates `aws/s3`
   - **You cannot**: Delete, rotate manually, or change key policies
   - **You can**: Use them for free (no monthly charge)
   - **Use case**: Quick encryption for standard AWS services

2. **Customer Managed Keys** (You create and name them)
   - **You control**: Key policies, rotation, deletion, grants
   - **You pay**: $1/month per key + usage fees
   - **Use case**: Fine-grained control, compliance requirements, cross-account access

```
┌────────────────────────────────────────────────────────┐
│           AWS Managed vs Customer Managed Keys         │
├────────────────────────────────────────────────────────┤
│                                                         │
│  AWS Managed Key (aws/s3)                              │
│  ┌──────────────────────────────────────┐             │
│  │  • Auto-created by AWS               │             │
│  │  • Auto-rotated every year           │             │
│  │  • Cannot be deleted                 │             │
│  │  • Policy managed by AWS             │             │
│  │  • Free to use                       │             │
│  └──────────────────────────────────────┘             │
│                                                         │
│  Customer Managed Key (my-database-key)                │
│  ┌──────────────────────────────────────┐             │
│  │  • You create and name it            │             │
│  │  • You control rotation              │             │
│  │  • You can schedule deletion         │             │
│  │  • You write key policies            │             │
│  │  • $1/month + usage fees             │             │
│  └──────────────────────────────────────┘             │
└────────────────────────────────────────────────────────┘
```

---

### Key Policies

**What They Are**: JSON documents that define **who can use and manage a CMK**. Think of them as the access control list for your encryption keys.

**Why They Matter**: Without a key policy, your encryption key is useless — no one can use it, not even you.

**Key Policy Structure**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow EC2 to use key for encryption",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:EncryptDataKey",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

**Simple Explanation**:
- **Principal**: WHO can access (IAM user, role, AWS service)
- **Action**: WHAT they can do (encrypt, decrypt, manage key)
- **Resource**: WHICH key (usually `*` meaning this key)
- **Effect**: Allow or Deny

**Real-World Example**: You have a database encryption key. You want:
- DBAs to manage the key (create backups, rotate it)
- RDS service to use it for encryption
- Lambda functions to decrypt database backups

You'd create three policy statements, one for each use case.

---

### Grants

**What They Are**: Temporary, programmatic permissions to use a KMS key. Think of them as "guest passes" for encryption keys.

**Why They Exist**: Key policies are permanent and require manual updates. Grants are:
- **Temporary**: Can be revoked instantly
- **Programmatic**: Created via API calls, not manual JSON editing
- **Delegatable**: The grantee can create sub-grants (with constraints)

**Use Case Comparison**:

| Scenario | Use Key Policy | Use Grant |
|----------|----------------|-----------|
| Permanent access for your Lambda function | ✅ | ❌ |
| Temporary access for a contractor's role | ❌ | ✅ |
| AWS service needs to encrypt data for you | ❌ | ✅ (AWS creates grants automatically) |
| Cross-account access with conditions | ✅ | ✅ |

**Example Flow**:
```
┌────────────────────────────────────────────────────────┐
│  Scenario: Sharing encrypted S3 snapshot cross-account │
└────────────────────────────────────────────────────────┘

Account A (Owner)                    Account B (Recipient)
    │                                        │
    │ 1. Create grant on CMK                 │
    │    "Account B can decrypt"             │
    │────────────────────────────────────────>│
    │                                        │
    │ 2. Share encrypted snapshot            │
    │────────────────────────────────────────>│
    │                                        │
    │                                    3. Use grant to
    │                                       decrypt snapshot
    │                                        │
    │ 4. Revoke grant when done              │
    │<────────────────────────────────────────│
```

---

### Envelope Encryption

**What It Is**: A two-layer encryption technique where:
1. Your data is encrypted with a **Data Key** (DEK)
2. The Data Key itself is encrypted with a **Customer Master Key** (CMK)

**Why This Architecture?**:
- **Performance**: CMKs never leave KMS (slow for large data). Data keys work locally (fast).
- **Security**: Even if someone steals encrypted data + encrypted data key, they can't decrypt without accessing KMS.
- **Scalability**: One CMK can generate millions of data keys.

**Visual Flow**:

```
┌──────────────────────────────────────────────────────────────┐
│              Envelope Encryption Process                      │
└──────────────────────────────────────────────────────────────┘

ENCRYPTION:
┌─────────────┐
│  Your Data  │  (e.g., 5GB file)
└──────┬──────┘
       │
       │ 1. Generate Data Key from KMS
       ▼
┌─────────────────────────────────────┐
│  KMS returns TWO things:            │
│  • Plaintext Data Key (256-bit)     │
│  • Encrypted Data Key (wrapped)     │
└──────┬──────────────────────────────┘
       │
       │ 2. Use plaintext key to encrypt data locally
       ▼
┌─────────────────┐
│  Encrypted Data │
└─────────────────┘
       │
       │ 3. Delete plaintext key from memory
       │ 4. Store encrypted data + encrypted data key together
       ▼
┌──────────────────────────────────────┐
│  Storage (S3, EBS, etc.)             │
│  ┌────────────────────────────────┐  │
│  │ Encrypted Data                 │  │
│  │ + Encrypted Data Key           │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

DECRYPTION:
       │
       │ 1. Retrieve encrypted data + encrypted data key
       ▼
┌─────────────────────────────────────┐
│  Send encrypted data key to KMS     │
│  KMS decrypts it with CMK           │
│  Returns plaintext data key         │
└──────┬──────────────────────────────┘
       │
       │ 2. Use plaintext key to decrypt data locally
       ▼
┌─────────────┐
│  Your Data  │
└─────────────┘
```

**Real-World Example**:
You upload a 10GB video to S3 with encryption:
1. S3 asks KMS: "Give me a data key for CMK `arn:aws:kms:...`"
2. KMS returns a 256-bit AES key (plaintext + encrypted version)
3. S3 encrypts the 10GB video with the plaintext key (fast, local operation)
4. S3 deletes the plaintext key from memory
5. S3 stores: `encrypted_video.mp4` + `encrypted_data_key` metadata
6. When you download, S3 asks KMS to decrypt the data key, then decrypts the video

**Key Insight**: The 10GB video never goes to KMS. Only the tiny 256-bit data key does. This is why envelope encryption is used for large data.

---

### Key Rotation

**What It Is**: Automatically generating a new version of your encryption key while keeping old versions available to decrypt existing data.

**Why It Matters**:
- **Compliance**: Many regulations (PCI-DSS, HIPAA) require periodic key rotation
- **Security**: Limits the exposure if a key is compromised (less data encrypted with one key)
- **Best Practice**: Reduces cryptanalysis attack surface

**How KMS Rotation Works**:

```
┌──────────────────────────────────────────────────────────┐
│          Automatic Key Rotation Timeline                  │
└──────────────────────────────────────────────────────────┘

Year 1:
CMK: my-database-key
└── Key Material Version 1 (active for encryption/decryption)

Year 2 (rotation triggered):
CMK: my-database-key
├── Key Material Version 1 (decryption only)
└── Key Material Version 2 (active for encryption/decryption)

Year 3:
CMK: my-database-key
├── Key Material Version 1 (decryption only)
├── Key Material Version 2 (decryption only)
└── Key Material Version 3 (active for encryption/decryption)
```

**Important Notes**:
- **The CMK ARN never changes** — your applications don't need updates
- **Old key material stays available** — you can still decrypt old data
- **AWS Managed Keys**: Automatically rotate every year (cannot disable)
- **Customer Managed Keys**: You enable rotation (every year once enabled)
- **Asymmetric keys and keys with imported material**: Cannot be auto-rotated

**Manual Rotation Alternative**:
For imported key material or asymmetric keys:
1. Create a new CMK
2. Update application to use new CMK
3. Keep old CMK for decrypting old data
4. Schedule deletion of old CMK after data is re-encrypted

---

### Multi-Region Keys

**What They Are**: A set of KMS keys in different AWS regions that have the same key material and key ID. Think of them as "synchronized copies" of the same encryption key.

**Why They Exist**:

**The Problem Without Multi-Region Keys**:
```
User in us-east-1 uploads encrypted file to S3
                    │
                    ▼
         File encrypted with key in us-east-1
                    │
                    │ You replicate file to eu-west-1
                    ▼
         User in eu-west-1 tries to download
                    │
                    ▼
         ❌ ERROR: Must call KMS in us-east-1
         (Cross-region KMS calls = slow + expensive)
```

**The Solution With Multi-Region Keys**:
```
Create multi-region key in us-east-1 (PRIMARY)
                    │
                    ▼
         Replicate to eu-west-1 (REPLICA)
         (Same key material, same key ID)
                    │
                    ▼
    Users in both regions use local KMS endpoint
             (Fast + no cross-region calls)
```

**Real-World Use Cases**:

1. **Disaster Recovery**:
   - Encrypt RDS snapshots in us-east-1
   - Replicate to eu-west-1
   - If us-east-1 fails, decrypt and restore in eu-west-1 immediately

2. **Global Applications**:
   - DynamoDB Global Tables encrypted in multiple regions
   - Users access encrypted data from nearest region (low latency)

3. **Data Sovereignty with Portability**:
   - Encrypt data in Europe (GDPR compliance)
   - If needed, decrypt in other regions without re-encryption

**Architecture**:

```
┌────────────────────────────────────────────────────────┐
│            Multi-Region Key Structure                   │
└────────────────────────────────────────────────────────┘

  us-east-1 (PRIMARY)                eu-west-1 (REPLICA)
┌─────────────────────┐           ┌─────────────────────┐
│ Multi-Region Key    │           │ Multi-Region Key    │
│                     │           │                     │
│ ID: mrk-abc123...   │◄─────────►│ ID: mrk-abc123...   │
│ Material: [SAME]    │  Synced   │ Material: [SAME]    │
│ Policies: Regional  │           │ Policies: Regional  │
└─────────────────────┘           └─────────────────────┘
         │                                   │
         ▼                                   ▼
   Encrypts/Decrypts                   Encrypts/Decrypts
   data in us-east-1                   data in eu-west-1
```

**Key Differences from Single-Region Keys**:
- **Key ID prefix**: `mrk-` instead of regular UUID
- **Replication**: Manual (you create replicas in other regions)
- **Policies**: Each replica can have different key policies
- **Deletion**: Must delete each replica separately

---

### External Key Store

**What It Is**: A feature that lets you use encryption keys stored in **your own hardware security module (HSM)** instead of AWS-managed HSMs.

**Why It Exists**:

Some organizations have requirements that say:
- "Encryption keys must never exist outside our physical data center"
- "We must use our certified HSM hardware"
- "We need to prove key material was generated on-premises"

**The Architecture**:

```
┌──────────────────────────────────────────────────────────┐
│         Standard KMS vs External Key Store                │
└──────────────────────────────────────────────────────────┘

STANDARD KMS:
  Your App → KMS (AWS HSM) → Encrypted Data
             └─ Key lives in AWS

EXTERNAL KEY STORE:
  Your App → KMS (proxy) → Your HSM → Encrypted Data
                           └─ Key lives on your hardware
                              (Connected via AWS CloudHSM)
```

**How It Works**:
1. You set up your own HSM (using AWS CloudHSM or compatible)
2. You create keys in YOUR HSM
3. You configure KMS to use your HSM as an "external key store"
4. When apps call KMS, KMS proxies cryptographic operations to your HSM
5. **Key material never leaves your HSM**

**Trade-offs**:

| Aspect | Standard KMS | External Key Store |
|--------|--------------|-------------------|
| Setup | Minutes | Days (HSM provisioning) |
| Cost | $1/key/month | $1/key + HSM costs (~$1,000+/month) |
| Performance | High | Lower (network calls to HSM) |
| Availability | AWS SLA (99.99%) | Your HSM availability |
| Control | AWS manages HSM | You manage HSM |
| Compliance | Most use cases | Strict key sovereignty requirements |

**Use Case Example**:
A bank has a regulatory requirement that encryption keys for customer data must be generated and stored on FIPS 140-2 Level 3 certified hardware that the bank physically controls. They use External Key Store to:
- Generate keys in their on-premises HSM
- Use KMS API for encryption (seamless integration)
- Maintain physical control and audit trail of key material

---

## Secrets Manager

### What It Is
AWS Secrets Manager is a **secure storage and lifecycle management service for sensitive information** like database passwords, API keys, OAuth tokens, and encryption keys.

### Why It Exists

**The Problem Without Secrets Manager**:

```
BAD PRACTICE #1: Hardcoded secrets
const dbPassword = "MyP@ssw0rd123";  // ❌ In source code
                                      // - Visible in Git history
                                      // - Exposed if code leaks

BAD PRACTICE #2: Environment variables
export DB_PASSWORD="MyP@ssw0rd123"    // ❌ In shell history
                                       // - Visible in process list
                                       // - Hard to rotate

BAD PRACTICE #3: Config files
db.config:                             // ❌ On disk unencrypted
  password: MyP@ssw0rd123              // - Backup files include it
                                       // - No rotation
```

**The Solution with Secrets Manager**:

```
GOOD PRACTICE:
1. Store secret in Secrets Manager (encrypted with KMS)
2. Application retrieves secret at runtime
3. Automatic rotation every 30 days
4. Audit trail of who accessed what and when
```

**Real-World Scenario**:
You have 50 Lambda functions accessing an RDS database. The database password needs to change monthly for compliance.

Without Secrets Manager:
- Update password in database
- Update password in 50 Lambda environment variables
- Redeploy all 50 functions
- Hope nothing breaks (it will)

With Secrets Manager:
- Password rotates automatically via Lambda function
- All 50 functions retrieve current password from Secrets Manager
- Zero downtime, zero manual work

---

### How Secrets Manager Works

```
┌──────────────────────────────────────────────────────────┐
│           Secrets Manager Architecture                    │
└──────────────────────────────────────────────────────────┘

Application/Lambda
       │
       │ 1. "Get secret: prod/db/password"
       ▼
┌─────────────────────────────┐
│   Secrets Manager API       │
│  - Check IAM permissions    │
│  - Retrieve encrypted blob  │
└──────────┬──────────────────┘
           │
           │ 2. Decrypt using KMS
           ▼
┌─────────────────────────────┐
│   KMS                       │
│  - Decrypt secret           │
│  - Return plaintext         │
└──────────┬──────────────────┘
           │
           │ 3. Return plaintext secret
           ▼
     Application
     (Uses password)
           │
           │ 4. Log access event
           ▼
     CloudTrail
```

**Key Features**:

1. **Encryption at Rest**: All secrets encrypted with KMS
2. **Encryption in Transit**: HTTPS/TLS for API calls
3. **Versioning**: Old secret versions kept (for rollback)
4. **Access Control**: IAM policies + resource policies
5. **Audit Logging**: CloudTrail tracks all access

---

### Secret Rotation

**What It Is**: Automatically changing a secret (like a password) on a schedule and updating it everywhere it's used.

**Why It Matters**:
- **Security**: Limits window of exposure if secret is compromised
- **Compliance**: Many regulations require periodic credential rotation
- **Best Practice**: Reduces risk of credential stuffing attacks

**How Automatic Rotation Works**:

```
┌──────────────────────────────────────────────────────────┐
│         Automatic Rotation Process (30-day cycle)        │
└──────────────────────────────────────────────────────────┘

Day 1:
  Current Secret: Password_v1
  Status: Active

Day 30 (Rotation Triggered):
  ┌─────────────────────────────────────────┐
  │ Secrets Manager invokes Rotation Lambda│
  └──────────────┬──────────────────────────┘
                 │
                 ▼
  Step 1: CREATE new secret (Password_v2)
  Step 2: SET new secret in database
  Step 3: TEST connection with new secret
  Step 4: FINISH - mark Password_v2 as active
                 │
                 ▼
  Current Secret: Password_v2 (active)
  Previous Secret: Password_v1 (deprecated, kept for rollback)
```

**The Rotation Lambda Function**:
AWS provides pre-built rotation functions for common databases:
- RDS (MySQL, PostgreSQL, SQL Server)
- DocumentDB
- Redshift
- Aurora

**Example Rotation Scenario**:

```python
# Your application code (no changes needed for rotation)
import boto3

secrets_client = boto3.client('secretsmanager')

def get_database_connection():
    # Always retrieves CURRENT version
    secret = secrets_client.get_secret_value(
        SecretId='prod/rds/master-password'
    )
    password = json.loads(secret['SecretString'])['password']

    # Use password to connect
    return psycopg2.connect(
        host='mydb.rds.amazonaws.com',
        user='admin',
        password=password  # This is the current password
    )
```

**What happens during rotation**:
1. Day 29: Your app uses `Password_v1`
2. Day 30 at 2 AM: Rotation starts
   - Lambda creates `Password_v2` in RDS
   - Tests it works
   - Marks `Password_v2` as current
3. Day 30 at 2:10 AM: Your app's next call gets `Password_v2`
4. No downtime, no manual intervention

---

### Cross-Region Replication

**What It Is**: Automatically replicating secrets to multiple AWS regions for disaster recovery and global applications.

**Why It Matters**:

**Without Replication**:
```
Primary Region (us-east-1) goes down
         │
         ▼
Applications in eu-west-1 can't access secrets
         │
         ▼
❌ Service outage until us-east-1 recovers
```

**With Replication**:
```
Secret in us-east-1 (primary)
         │
         │ Automatically replicated
         ▼
Secret in eu-west-1 (replica)
Secret in ap-south-1 (replica)
         │
         ▼
✅ Applications access local replica (failover works)
```

**How Replication Works**:

```
┌────────────────────────────────────────────────────────┐
│          Cross-Region Secret Replication                │
└────────────────────────────────────────────────────────┘

us-east-1 (Primary)              eu-west-1 (Replica)
┌──────────────────┐            ┌──────────────────┐
│ Secret:          │            │ Secret:          │
│ prod/db/password │───────────►│ prod/db/password │
│                  │  Auto-sync │                  │
│ Value: Pass_v1   │            │ Value: Pass_v1   │
│ KMS Key: key-A   │            │ KMS Key: key-B   │
└──────────────────┘            └──────────────────┘
         │                               │
         │ Rotation occurs               │
         ▼                               ▼
 Value: Pass_v2                  Value: Pass_v2
 (synced within minutes)
```

**Key Points**:
- **Each region uses its own KMS key** (data sovereignty)
- **Replication is near real-time** (usually < 1 minute)
- **Rotation in primary region triggers replication** automatically
- **Replicas are read-only** (update primary, replicas sync)

**Use Case**:
Multi-region application with RDS read replicas:
- Primary database in us-east-1
- Read replicas in eu-west-1 and ap-south-1
- Secret replicated to all three regions
- Each region's app reads from local Secrets Manager (low latency)

---

### Resource Policies

**What They Are**: JSON policies attached directly to secrets that control who can access them, similar to S3 bucket policies.

**Why Use Resource Policies Instead of IAM Policies**:

| IAM Policy (Attached to User/Role) | Resource Policy (Attached to Secret) |
|------------------------------------|--------------------------------------|
| "This user can access secret X" | "This secret allows user Y" |
| Managed per identity | Managed per secret |
| Limited to same AWS account | **Can grant cross-account access** |

**Example Resource Policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowProductionLambdaAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ProdLambdaRole"
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*"
    },
    {
      "Sid": "DenyAccessFromOutsideVPC",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-1234567890abcdef"
        }
      }
    }
  ]
}
```

**Simple Explanation**:
- **First statement**: Production Lambda can read the secret
- **Second statement**: Deny access unless request comes from specific VPC endpoint (prevents access from outside corporate network)

**Real-World Use Case**:
You have a shared database used by multiple AWS accounts:
- Account A owns the database
- Accounts B and C need access
- Create secret in Account A with resource policy allowing Accounts B and C
- Accounts B and C can retrieve secret without copying it

---

### Automatic Rotation (Lambda)

**What It Is**: The Lambda function that Secrets Manager invokes to perform the actual rotation steps.

**The Four-Step Rotation Process**:

```
┌──────────────────────────────────────────────────────────┐
│      Lambda Rotation Function Flow                        │
└──────────────────────────────────────────────────────────┘

Event: Secrets Manager triggers Lambda with:
       - SecretId
       - Token (version identifier)
       - Step (which rotation step)

┌─────────────────────────────────────────────────────────┐
│ STEP 1: createSecret                                    │
│ - Generate new password (random string)                 │
│ - Store in Secrets Manager with label "AWSPENDING"      │
└──────────┬──────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 2: setSecret                                       │
│ - Connect to database with CURRENT password             │
│ - Execute: ALTER USER admin PASSWORD 'new_password';    │
│ - Database now accepts both old and new passwords       │
└──────────┬──────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 3: testSecret                                      │
│ - Attempt database connection with NEW password         │
│ - If fails: Raise exception, rotation aborts            │
│ - If succeeds: Continue to finishSecret                 │
└──────────┬──────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 4: finishSecret                                    │
│ - Label new version "AWSCURRENT"                        │
│ - Label old version "AWSPREVIOUS"                       │
│ - Future GetSecretValue calls return new password       │
└─────────────────────────────────────────────────────────┘
```

**Custom Rotation Function Example** (simplified):

```python
import boto3
import psycopg2
import json

secrets = boto3.client('secretsmanager')

def lambda_handler(event, context):
    # Event contains: SecretId, Token, Step
    secret_id = event['SecretId']
    token = event['Token']
    step = event['Step']

    if step == "createSecret":
        create_secret(secret_id, token)
    elif step == "setSecret":
        set_secret(secret_id, token)
    elif step == "testSecret":
        test_secret(secret_id, token)
    elif step == "finishSecret":
        finish_secret(secret_id, token)

def create_secret(secret_id, token):
    # Generate random password (simplified)
    import random, string
    new_password = ''.join(random.choices(string.ascii_letters + string.digits, k=32))

    # Store in Secrets Manager with "AWSPENDING" label
    secrets.put_secret_value(
        SecretId=secret_id,
        ClientRequestToken=token,
        SecretString=json.dumps({'password': new_password}),
        VersionStages=['AWSPENDING']
    )

def set_secret(secret_id, token):
    # Get current password (to connect) and pending password (to set)
    current = secrets.get_secret_value(SecretId=secret_id, VersionStage='AWSCURRENT')
    pending = secrets.get_secret_value(SecretId=secret_id, VersionStage='AWSPENDING')

    current_password = json.loads(current['SecretString'])['password']
    new_password = json.loads(pending['SecretString'])['password']

    # Connect to database with current password
    conn = psycopg2.connect(
        host='mydb.rds.amazonaws.com',
        user='admin',
        password=current_password
    )

    # Set new password in database
    cursor = conn.cursor()
    cursor.execute(f"ALTER USER admin WITH PASSWORD '{new_password}';")
    conn.commit()
    conn.close()

def test_secret(secret_id, token):
    # Retrieve pending password
    pending = secrets.get_secret_value(SecretId=secret_id, VersionStage='AWSPENDING')
    new_password = json.loads(pending['SecretString'])['password']

    # Test connection with new password
    try:
        conn = psycopg2.connect(
            host='mydb.rds.amazonaws.com',
            user='admin',
            password=new_password  # Use NEW password
        )
        conn.close()  # Success!
    except Exception as e:
        raise Exception(f"Failed to authenticate with new password: {e}")

def finish_secret(secret_id, token):
    # Move version labels: AWSPENDING → AWSCURRENT
    # (Secrets Manager handles this automatically)
    pass
```

**Key Concepts**:
- **Version Stages**: Labels like `AWSCURRENT`, `AWSPENDING`, `AWSPREVIOUS`
- **Token**: Unique ID for this rotation attempt (prevents duplicate rotations)
- **Rollback**: If rotation fails, `AWSCURRENT` stays on old password
- **Zero Downtime**: Database accepts both old and new passwords during rotation window

---

## Certificate Manager (ACM)

### What It Is
AWS Certificate Manager is a service for **provisioning, managing, and deploying SSL/TLS certificates** for your AWS resources and internal applications.

### Why It Exists

**The Problem Without ACM**:

Traditional SSL certificate management:
1. Purchase certificate from Certificate Authority ($50-$300/year)
2. Prove domain ownership (email validation, DNS records)
3. Download certificate files (.pem, .crt, .key)
4. Manually upload to load balancers, CloudFront, API Gateway
5. Set calendar reminder for expiration (usually 1 year)
6. Repeat steps 1-4 every year (or certificate expires → site goes down)

**The Solution with ACM**:
1. Request certificate (free for AWS services)
2. Validate domain ownership (automated via DNS)
3. ACM automatically deploys to AWS resources
4. ACM automatically renews before expiration
5. **Never think about certificates again**

**Cost Comparison**:

| Traditional | ACM |
|-------------|-----|
| $50-300/year per certificate | **Free** for AWS-integrated services |
| Manual renewal | **Automatic renewal** |
| Manual upload to servers | **Automatic deployment** |
| Expires if you forget | **Never expires** (auto-renews) |

---

### How ACM Works

```
┌──────────────────────────────────────────────────────────┐
│         ACM Certificate Lifecycle                         │
└──────────────────────────────────────────────────────────┘

1. REQUEST CERTIFICATE
   User: "I want a cert for example.com"
         │
         ▼
   ┌────────────────────────────┐
   │ ACM creates certificate    │
   │ Status: PENDING_VALIDATION │
   └────────────┬───────────────┘
                │
                ▼
2. VALIDATION
   ACM: "Prove you own example.com"
   Options:
   ├─ DNS Validation (add CNAME record)
   └─ Email Validation (click link in email)
         │
         │ User completes validation
         ▼
   ┌────────────────────────────┐
   │ Status: ISSUED             │
   │ Valid for 13 months        │
   └────────────┬───────────────┘
                │
                ▼
3. DEPLOYMENT
   User attaches cert to:
   ├─ Load Balancer (ALB/NLB)
   ├─ CloudFront distribution
   ├─ API Gateway
   └─ Elastic Beanstalk
         │
         ▼
   ┌────────────────────────────┐
   │ HTTPS traffic enabled      │
   └────────────┬───────────────┘
                │
                ▼
4. AUTO-RENEWAL (60 days before expiry)
   ACM: "Cert expires soon, renewing..."
   ├─ DNS validation: Auto-renews (if CNAME still exists)
   └─ Email validation: Sends reminder email
         │
         ▼
   ┌────────────────────────────┐
   │ Certificate renewed        │
   │ No downtime                │
   └────────────────────────────┘
```

---

### Public Certificates

**What They Are**: SSL/TLS certificates signed by a **publicly trusted Certificate Authority** (CA) that browsers and clients trust by default.

**Use Cases**:
- Public websites (example.com)
- Public APIs (api.example.com)
- CloudFront distributions
- Anything accessed by external users/browsers

**How Browsers Trust Public Certs**:

```
┌─────────────────────────────────────────────────────────┐
│           Public Certificate Trust Chain                │
└─────────────────────────────────────────────────────────┘

Browser visits https://example.com
         │
         ▼
1. Server presents certificate
   Subject: example.com
   Issuer: Amazon (ACM CA)
         │
         ▼
2. Browser checks: "Do I trust Amazon CA?"
   ├─ Checks built-in list of trusted CAs
   └─ Amazon is in the list ✅
         │
         ▼
3. Browser validates:
   ├─ Certificate not expired ✅
   ├─ Domain matches (example.com) ✅
   └─ Signature valid ✅
         │
         ▼
4. Shows green lock 🔒 in address bar
```

**Features**:
- **Free** for AWS-integrated services
- **Automatic renewal** (every 13 months)
- **Unlimited subdomains** with wildcard certs (`*.example.com`)
- **Cannot be exported** (AWS keeps private key for security)

**Limitations**:
- Only works with AWS services (ALB, CloudFront, API Gateway, etc.)
- Cannot be used on EC2 instances directly (use ACM Private CA instead)
- Cannot be downloaded or exported

---

### Private CA (Certificate Authority)

**What It Is**: Your own **private Certificate Authority** for issuing internal certificates that only your organization trusts.

**Why Use Private CA**:

**Scenario**: You have 500 microservices communicating via mTLS (mutual TLS):
- Each service needs a certificate
- Buying 500 public certificates = expensive + management nightmare
- Public CAs don't issue certs for internal domains like `auth.internal.mycompany.local`

**Solution**: Create your own CA, issue unlimited internal certs

**Public vs Private Certificates**:

```
┌─────────────────────────────────────────────────────────┐
│        Public Cert vs Private Cert Use Cases            │
└─────────────────────────────────────────────────────────┘

PUBLIC CERTIFICATE:
- Used for: Internet-facing websites/APIs
- Trusted by: All browsers/clients globally
- Validation: Must prove domain ownership
- Cost: Free (in ACM)
- Example: https://www.example.com

PRIVATE CERTIFICATE:
- Used for: Internal applications, microservices, VPNs
- Trusted by: Only devices you configure
- Validation: None needed (you control CA)
- Cost: $400/month per CA + $0.75 per cert
- Example: https://api.internal.mycompany.local
```

**Private CA Architecture**:

```
┌──────────────────────────────────────────────────────────┐
│           ACM Private CA Hierarchy                        │
└──────────────────────────────────────────────────────────┘

Root CA (mycompany-root-ca)
  │
  │ Issues certificate for:
  ▼
Subordinate CA (mycompany-issuing-ca)
  │
  │ Issues certificates for:
  ├─ Service: auth.internal.mycompany.local
  ├─ Service: payments.internal.mycompany.local
  ├─ Service: users.internal.mycompany.local
  └─ Device: laptop-12345.corp.mycompany.local

Trust Configuration:
- Install Root CA cert on all employee devices
- All issued certs are automatically trusted
```

**Real-World Use Case**:
A company with 100 engineers, 1,000 laptops, and 200 internal services:
1. Create ACM Private CA
2. Issue certificate for each service and laptop
3. Configure devices to trust the Private CA root certificate
4. All internal HTTPS connections use mTLS (mutual authentication)
5. Cost: $400/month + $75 for 100 certs = $475/month (vs $10,000+ for public certs)

---

### Certificate Renewal

**What It Is**: The process of replacing an expiring certificate with a new one before it expires.

**Why Auto-Renewal Matters**:

```
WITHOUT AUTO-RENEWAL:
Day 1: Certificate issued (valid 395 days)
Day 365: Calendar reminder (renewal due in 30 days)
         ├─ Engineer on vacation ❌
         └─ Reminder ignored
Day 395: Certificate expires
         ├─ Website shows "Not Secure" warning
         ├─ API calls fail
         ├─ Customer calls start flooding in
         └─ All-hands-on-deck emergency

WITH ACM AUTO-RENEWAL:
Day 1: Certificate issued
Day 335: ACM starts renewal (60 days early)
Day 336: ACM validates domain automatically
Day 337: New certificate deployed
Day 338-395: Still valid (safety buffer)
Day 396+: New certificate active
         └─ Engineers never notice ✅
```

**How ACM Auto-Renewal Works**:

```
┌──────────────────────────────────────────────────────────┐
│         ACM Automatic Renewal Process                     │
└──────────────────────────────────────────────────────────┘

60 days before expiry:
┌────────────────────────────────────────┐
│ ACM: "Renewal time for example.com"   │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│ DNS Validated Cert:                    │
│ - ACM checks if CNAME record exists    │
│ - If yes: Auto-renews ✅               │
│ - No action needed from you            │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│ Email Validated Cert:                  │
│ - ACM sends email to domain owner      │
│ - Must click approval link within 72h  │
│ - If ignored: Cert not renewed ❌      │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│ New cert issued and deployed           │
│ - Attached resources updated           │
│ - Zero downtime                        │
└────────────────────────────────────────┘
```

**Best Practice**: Always use **DNS validation** instead of email validation for auto-renewal.

**Renewal Failures (Email Validation)**:
Common reasons email validation fails:
- Domain owner changed
- Email went to spam
- Company email system blocked it
- Engineer on vacation didn't click link

**Monitoring Renewal**:
- CloudWatch Metrics: `DaysToExpiry`
- EventBridge Events: ACM emits renewal success/failure events
- Set up alerts for `DaysToExpiry < 30` (safety net)

---

### DNS Validation

**What It Is**: Proving you own a domain by adding a specific CNAME record to your DNS configuration.

**Why DNS Validation > Email Validation**:

| DNS Validation | Email Validation |
|----------------|------------------|
| ✅ Fully automatic renewal | ❌ Requires clicking email link |
| ✅ No human intervention | ❌ Humans forget, go on vacation |
| ✅ Works with Route 53 (one-click) | ❌ Requires access to domain email |
| ✅ Renewal never fails (CNAME stays forever) | ❌ Email can be missed/blocked |

**How DNS Validation Works**:

```
┌──────────────────────────────────────────────────────────┐
│              DNS Validation Flow                          │
└──────────────────────────────────────────────────────────┘

1. Request cert for example.com
         │
         ▼
2. ACM provides validation record:
   ┌─────────────────────────────────────────────────┐
   │ Name: _abc123.example.com                       │
   │ Type: CNAME                                     │
   │ Value: _xyz789.acm-validations.aws.             │
   └─────────────────────────────────────────────────┘
         │
         ▼
3. Add CNAME to your DNS (Route 53 or external):
   ┌─────────────────────────────────────────────────┐
   │ Route 53 Hosted Zone: example.com               │
   │ ┌───────────────────────────────────────────┐   │
   │ │ _abc123.example.com → CNAME → _xyz789...  │   │
   │ └───────────────────────────────────────────┘   │
   └─────────────────────────────────────────────────┘
         │
         ▼
4. ACM queries DNS:
   "Does _abc123.example.com point to _xyz789...?"
         │
         ├─ Yes ✅ → Certificate ISSUED
         └─ No ❌ → Certificate stays PENDING
         │
         ▼
5. Future renewals:
   ACM checks CNAME again → Renews automatically
```

**Route 53 Integration** (Easiest Method):

If your domain is in Route 53:
1. Request certificate in ACM
2. Click "Create records in Route 53" button
3. ACM automatically creates CNAME record
4. Certificate validated in minutes
5. Auto-renewal works forever (CNAME stays in Route 53)

**External DNS Provider** (GoDaddy, Namecheap, etc.):
1. Request certificate in ACM
2. Copy CNAME name and value from ACM console
3. Log into DNS provider
4. Add CNAME record manually
5. Wait for DNS propagation (can take hours)
6. ACM validates and issues certificate

**Example CNAME Record**:
```
Name: _a79865eb4cd1a6ab990a45779b4e0b96.example.com
Type: CNAME
Value: _75c7dd05e8d62a6e3f1e0e5b6a1d1234.acm-validations.aws.
TTL: 300
```

**Important**: Keep the CNAME record forever for auto-renewal to work. Deleting it breaks renewal.

---

### Email Validation

**What It Is**: Proving domain ownership by clicking an approval link sent to specific domain email addresses.

**How Email Validation Works**:

```
┌──────────────────────────────────────────────────────────┐
│              Email Validation Flow                        │
└──────────────────────────────────────────────────────────┘

1. Request cert for example.com
         │
         ▼
2. ACM sends emails to:
   ├─ admin@example.com
   ├─ administrator@example.com
   ├─ hostmaster@example.com
   ├─ postmaster@example.com
   ├─ webmaster@example.com
   └─ Domain WHOIS contact (if public)
         │
         ▼
3. Email contains approval link:
   ┌───────────────────────────────────────────┐
   │ Subject: Certificate Approval for         │
   │          example.com                      │
   │                                           │
   │ Click this link to approve:               │
   │ https://certificates.amazon.com/...       │
   │                                           │
   │ Link expires in 72 hours                  │
   └───────────────────────────────────────────┘
         │
         ▼
4. Domain owner clicks link → Cert ISSUED
   Don't click → Cert stays PENDING
         │
         ▼
5. Renewal (every 13 months):
   ACM sends email again
   Must click link again ❌
```

**When to Use Email Validation**:
- You don't control DNS (IT department does)
- You can't add DNS records (bureaucracy)
- Domain is hosted externally and DNS updates take days
- Testing or temporary certificates

**Problems with Email Validation**:
1. **Renewal requires human action** (click email every year)
2. **Emails can be missed** (spam filter, vacation, employee left)
3. **72-hour deadline** (certificate fails if not approved)
4. **Not suitable for automation** (can't auto-renew)

**Best Practice**: Use DNS validation unless you have a specific reason not to.

---

## WAF (Web Application Firewall)

### What It Is
AWS WAF is a **web application firewall** that filters HTTP/HTTPS requests based on rules you define, protecting your applications from common web exploits and bots.

### Why It Exists

**The Problem**:
Traditional firewalls (network firewalls, security groups) work at OSI Layer 3/4 (IP addresses, ports):
- Block: "Deny all traffic from 192.168.1.1"
- Allow: "Allow HTTPS on port 443"

But web attacks happen at **Layer 7 (application layer)**:
- SQL injection: `https://example.com/search?q=' OR '1'='1`
- Cross-site scripting: `<script>alert('XSS')</script>`
- DDoS attacks from distributed IPs (can't block by IP)
- Credential stuffing: 10,000 login attempts from different IPs
- Scraping bots: Crawling your site 1000x/second

**Security groups can't detect or block these**. They only see "HTTPS request from valid IP" → Allow ✅

**The Solution**:
WAF inspects the **content** of HTTP requests:
- URL path
- Query strings
- HTTP headers
- Request body
- Geo-location
- Request rate

```
┌──────────────────────────────────────────────────────────┐
│       Security Groups vs WAF                              │
└──────────────────────────────────────────────────────────┘

Security Group (Layer 3/4):
Request from 203.0.113.5:45123 → port 443
├─ Source IP allowed? ✅
├─ Port 443 allowed? ✅
└─ ALLOW (doesn't look at content)

WAF (Layer 7):
Request: POST /login HTTP/1.1
         Content: username=admin' OR '1'='1
├─ Contains SQL injection pattern? ❌
├─ Rate: 100 requests/sec from this IP? ❌
└─ BLOCK (inspects actual request)
```

---

### How WAF Works: The Architecture

```
┌──────────────────────────────────────────────────────────┐
│              WAF Request Flow                             │
└──────────────────────────────────────────────────────────┘

User Request
     │
     ▼
┌─────────────────────┐
│  CloudFront / ALB   │
│  API Gateway        │
└──────┬──────────────┘
       │
       │ Forwards request to WAF
       ▼
┌─────────────────────────────────────────┐
│           AWS WAF                       │
│  ┌────────────────────────────────┐    │
│  │  Web ACL (Access Control List) │    │
│  │  ┌──────────────────────────┐  │    │
│  │  │  Rule 1: Block SQL inject│  │    │
│  │  │  Rule 2: Rate limit      │  │    │
│  │  │  Rule 3: Geo-blocking    │  │    │
│  │  └──────────┬───────────────┘  │    │
│  └─────────────┼──────────────────┘    │
│                │                        │
│     ┌──────────┴──────────┐            │
│     │ ALLOW  │  BLOCK     │            │
│     ▼        ▼            │            │
└─────┼────────┼─────────────────────────┘
      │        │
      │        │ Blocked request → 403 Forbidden
      │        └──────────────────────────────►
      │
      │ Allowed request
      ▼
┌──────────────────┐
│  Backend Server  │
│  (EC2, Lambda)   │
└──────────────────┘
```

**Key Concepts**:
- **Web ACL**: Container for rules attached to resources (ALB, CloudFront, etc.)
- **Rules**: Conditions that inspect requests (if condition matches → action)
- **Action**: ALLOW, BLOCK, COUNT, CAPTCHA, or CHALLENGE
- **Priority**: Rules evaluated in order (1, 2, 3...); first match wins

---

### Web ACLs (Access Control Lists)

**What They Are**: The top-level container that holds WAF rules and is attached to AWS resources.

**Structure**:

```
Web ACL: "production-waf"
├─ Rule Priority 1: AWS Managed Rule - Core Rule Set
├─ Rule Priority 2: Rate-based rule (1000 req/5min)
├─ Rule Priority 3: Block requests from China/Russia
├─ Rule Priority 4: Custom rule - Block /admin path
└─ Default Action: ALLOW
```

**How Rules Are Evaluated**:

```
┌──────────────────────────────────────────────────────────┐
│           Web ACL Rule Evaluation                         │
└──────────────────────────────────────────────────────────┘

Request arrives
      │
      ▼
┌──────────────────────────────┐
│ Rule 1 (Priority 1):         │
│ Block SQL injection patterns │
└──────┬───────────────────────┘
       │
       ├─ Match? → BLOCK (stop evaluation, return 403)
       └─ No match → Continue
      │
      ▼
┌──────────────────────────────┐
│ Rule 2 (Priority 2):         │
│ Rate limit: Max 1000/5min    │
└──────┬───────────────────────┘
       │
       ├─ Match? → BLOCK
       └─ No match → Continue
      │
      ▼
┌──────────────────────────────┐
│ Rule 3 (Priority 3):         │
│ Geo-block China              │
└──────┬───────────────────────┘
       │
       ├─ Match? → BLOCK
       └─ No match → Continue
      │
      ▼
┌──────────────────────────────┐
│ No rules matched             │
│ Apply Default Action: ALLOW  │
└──────────────────────────────┘
```

**Important**: First matching rule wins. Order matters!

**Where You Attach Web ACLs**:
- **CloudFront**: Protects global CDN distributions
- **Application Load Balancer (ALB)**: Protects regional apps
- **API Gateway**: Protects REST/HTTP APIs
- **AppSync**: Protects GraphQL APIs
- **Cognito User Pool**: Protects authentication endpoints
- **App Runner**: Protects containerized apps
- **Verified Access**: Protects VPN alternatives

---

### Rules & Rule Groups

**What They Are**:

**Rule**: A single condition + action
- Example: "If request contains `'OR 1=1'` → BLOCK"

**Rule Group**: Collection of related rules
- Example: "OWASP Top 10 Protection" (10 rules for different vulnerabilities)

**Types of Rules**:

```
┌──────────────────────────────────────────────────────────┐
│                 WAF Rule Types                            │
└──────────────────────────────────────────────────────────┘

1. REGULAR RULES (match conditions)
   ├─ String matching: "Block if URL contains /admin"
   ├─ Geo-matching: "Block if country = Russia"
   ├─ IP set matching: "Block if IP in blacklist"
   ├─ Size constraint: "Block if body > 1MB"
   └─ Regex matching: "Block if matches SQL pattern"

2. RATE-BASED RULES (request frequency)
   └─ "Block IP if > 1000 requests / 5 minutes"

3. MANAGED RULE GROUPS (AWS or third-party)
   ├─ AWS Managed Rules
   ├─ AWS Marketplace Rules (F5, Fortinet, etc.)
   └─ Your Own Managed Rule Groups
```

**Rule Example** (Block specific user-agent):

```json
{
  "Name": "BlockBadBot",
  "Priority": 10,
  "Action": {
    "Block": {}
  },
  "Statement": {
    "ByteMatchStatement": {
      "FieldToMatch": {
        "SingleHeader": {
          "Name": "user-agent"
        }
      },
      "SearchString": "BadBot/1.0",
      "PositionalConstraint": "CONTAINS",
      "TextTransformations": [
        {
          "Type": "LOWERCASE",
          "Priority": 0
        }
      ]
    }
  }
}
```

**Simple Explanation**:
- **FieldToMatch**: Look at the "User-Agent" header
- **SearchString**: Find "BadBot/1.0"
- **PositionalConstraint**: Anywhere in the header (CONTAINS)
- **TextTransformations**: Convert to lowercase first (case-insensitive match)
- **Action**: BLOCK if matched

---

### Managed Rules

**What They Are**: Pre-configured rule groups maintained by AWS or third-party security vendors. Think of them as "WAF plugins" you can enable with one click.

**Why Use Managed Rules**:
- **Expertise**: Written by security experts
- **Maintained**: Auto-updated when new threats emerge
- **Saves Time**: No need to write rules yourself
- **Tested**: Used by thousands of AWS customers

**AWS Managed Rule Groups**:

```
┌──────────────────────────────────────────────────────────┐
│           Popular AWS Managed Rule Groups                │
└──────────────────────────────────────────────────────────┘

1. Core Rule Set (CRS)
   - OWASP Top 10 protection
   - SQL injection, XSS, local file inclusion
   - Most common rule set (start here)

2. Known Bad Inputs
   - Blocks requests with known malicious patterns
   - CVE-based signatures
   - Zero-day exploit patterns

3. Anonymous IP List
   - Blocks requests from:
     ├─ VPNs
     ├─ Proxies
     ├─ Tor exit nodes
     └─ Hosting providers (prevent bot attacks)

4. SQL Database Protection
   - Advanced SQL injection patterns
   - Database-specific exploits (MySQL, PostgreSQL, etc.)

5. Linux/Windows Operating System
   - Blocks OS-level attacks
   - Command injection
   - File path traversal

6. PHP/WordPress Application
   - PHP-specific vulnerabilities
   - WordPress plugin exploits
   - Common CMS attacks
```

**Cost**:
- Most AWS Managed Rules: **Free** (included with WAF)
- Some advanced rules: **$10-20/month**
- Third-party rules: **$50-500/month** (e.g., F5, Imperva)

**Example: Enabling Core Rule Set**:

```
Web ACL: "my-app-waf"
├─ Rule Priority 1: AWS-AWSManagedRulesCommonRuleSet
│  ├─ Cost: Free
│  ├─ Rules: 15+ rules for OWASP Top 10
│  ├─ Action: BLOCK
│  └─ Auto-updated by AWS
└─ Default Action: ALLOW
```

**Managed Rule Actions**:
You can override actions for specific rules within a managed rule group:

```
Core Rule Set:
├─ Rule: SQLi_QUERYARGUMENTS (SQL injection in query string)
│  └─ Action: BLOCK (default)
├─ Rule: XSS_BODY (XSS in request body)
│  └─ Action: COUNT (override to just count, not block)
└─ Rule: SizeRestrictions_BODY (body > 8KB)
   └─ Action: CAPTCHA (override to show CAPTCHA)
```

**Why Override?**: Some rules cause false positives. Instead of disabling the entire rule group, you can:
- Change BLOCK → COUNT (monitor without blocking)
- Change BLOCK → CAPTCHA (allow humans, block bots)

---

### Rate-Based Rules

**What They Are**: Rules that block IPs exceeding a specified request rate within a time window.

**Why They Matter**:
- **DDoS Protection**: Single IP flooding your API with 10,000 requests/sec
- **Brute Force Prevention**: Attacker trying 1000 passwords on /login
- **Scraping Prevention**: Bots crawling your entire site
- **Cost Control**: Prevents expensive backend processing

**How Rate-Based Rules Work**:

```
┌──────────────────────────────────────────────────────────┐
│          Rate-Based Rule Mechanism                        │
└──────────────────────────────────────────────────────────┘

5-minute sliding window:
┌─────────────────────────────────────────────────────────┐
│ IP: 203.0.113.5                                         │
│                                                         │
│ 12:00 PM: 200 requests                                  │
│ 12:01 PM: 300 requests                                  │
│ 12:02 PM: 400 requests   } Total: 1100 in 5 min        │
│ 12:03 PM: 100 requests   } Exceeds 1000 → BLOCK        │
│ 12:04 PM: 100 requests   }                              │
│                                                         │
│ 12:05 PM: IP blocked for 10-30 minutes                  │
└─────────────────────────────────────────────────────────┘
```

**Example Rule**:

```json
{
  "Name": "RateLimitLogin",
  "Priority": 5,
  "Action": {
    "Block": {}
  },
  "Statement": {
    "RateBasedStatement": {
      "Limit": 100,  // 100 requests
      "AggregateKeyType": "IP",  // Per source IP
      "ScopeDownStatement": {
        // Only apply to /login path
        "ByteMatchStatement": {
          "FieldToMatch": {
            "UriPath": {}
          },
          "SearchString": "/login",
          "PositionalConstraint": "EXACTLY"
        }
      }
    }
  }
}
```

**Simple Explanation**:
- **Limit**: 100 requests per 5-minute window
- **AggregateKeyType**: Track per source IP (alternatives: per session, per custom header)
- **ScopeDownStatement**: Only count requests to `/login` path
- **Action**: BLOCK IP for 10-30 minutes if exceeded

**Advanced Rate Limiting**:

```
Aggregate by different keys:
├─ IP: Block single IP
├─ Forwarded-IP: Block client behind proxy
├─ Session Cookie: Block logged-in user
├─ API Key: Block abusive API client
└─ Custom Header: Block tenant ID in multi-tenant app
```

**Real-World Example**:
E-commerce site during Black Friday sale:
- Legitimate traffic: 1000 requests/sec
- Bot traffic: Scrapers checking prices 24/7

Rate-based rules:
1. Allow 100 requests/5min to product pages (humans browse slowly)
2. Allow 1000 requests/5min to homepage (legitimate traffic spikes)
3. Allow 10 requests/5min to /api/prices (prevent scraping)

---

### Bot Control

**What It Is**: AWS Managed Rule Group specifically designed to detect and block bot traffic.

**Types of Bots**:

```
┌──────────────────────────────────────────────────────────┐
│               Bot Classification                          │
└──────────────────────────────────────────────────────────┘

GOOD BOTS (allow):
├─ Search engine crawlers (Googlebot, Bingbot)
├─ Social media crawlers (Facebook, Twitter)
├─ Monitoring tools (Pingdom, UptimeRobot)
└─ Known benign bots

BAD BOTS (block):
├─ Scrapers (stealing content/prices)
├─ Credential stuffing bots (automated login attempts)
├─ Inventory hoarders (buying all stock instantly)
├─ Scalpers (concert tickets, limited editions)
└─ DDoS bots

UNKNOWN BOTS (challenge):
├─ Unverified automated traffic
└─ Suspicious patterns
```

**How Bot Control Works**:

```
Request arrives
      │
      ▼
┌──────────────────────────────────────┐
│ Bot Control inspects:                │
│ ├─ User-Agent                        │
│ ├─ TLS fingerprint                   │
│ ├─ JavaScript execution              │
│ ├─ Mouse movements                   │
│ ├─ Request timing patterns           │
│ └─ IP reputation                     │
└──────┬───────────────────────────────┘
       │
       ├─ Good bot (verified) → ALLOW
       ├─ Bad bot (known malicious) → BLOCK
       └─ Unknown → CAPTCHA/CHALLENGE
```

**Bot Control Rule Tiers**:

| Tier | Detection | Cost | Use Case |
|------|-----------|------|----------|
| **Common** | Basic bot signatures | $10/month + $1/million requests | Small sites, basic protection |
| **Targeted** | Advanced fingerprinting | $100/month + $10/million requests | E-commerce, high-value endpoints |

**Example Configuration**:

```
Web ACL: "ecommerce-waf"
├─ Rule: AWS-AWSManagedRulesBotControlRuleSet
│  ├─ Tier: Targeted
│  ├─ Actions:
│  │  ├─ Verified Bots (Googlebot) → ALLOW
│  │  ├─ Category: Search Engine → ALLOW
│  │  ├─ Category: Scraping → BLOCK
│  │  ├─ Category: Credential Stuffing → BLOCK
│  │  └─ Unverified Bots → CAPTCHA
│  └─ Scope: All requests
```

**Real-World Use Case**:
Sneaker e-commerce site during limited edition release:
- Without Bot Control: Bots buy all stock in 0.3 seconds
- With Bot Control:
  - Verified good bots (Google) → Can index site
  - Known bad bots → Blocked
  - Unknown suspicious traffic → Must solve CAPTCHA
  - Humans → Unaffected

---

### Fraud Control

**What It Is**: Managed rule group for detecting **account takeover** (ATO) and **account creation fraud**.

**Types of Fraud Detected**:

```
┌──────────────────────────────────────────────────────────┐
│              Fraud Control Protections                    │
└──────────────────────────────────────────────────────────┘

1. Account Takeover (ATP - Account Takeover Prevention)
   ├─ Credential stuffing (stolen password lists)
   ├─ Brute force login attempts
   ├─ Suspicious login patterns
   └─ Anomalous device fingerprints

2. Account Creation Fraud (ACFP)
   ├─ Bulk fake account creation
   ├─ Disposable email addresses
   ├─ VPN/proxy signup attempts
   └─ Bot-driven registrations
```

**How Account Takeover Prevention Works**:

```
User attempts login
      │
      ▼
┌──────────────────────────────────────────┐
│ ATP analyzes:                            │
│ ├─ Is this IP in stolen credential DB?   │
│ ├─ Has this IP tried 100 passwords?      │
│ ├─ Is login from new country suddenly?   │
│ ├─ Device fingerprint matches user?      │
│ └─ Time since last login suspicious?     │
└──────┬───────────────────────────────────┘
       │
       ├─ Low risk → ALLOW (normal login)
       ├─ Medium risk → CAPTCHA (verify human)
       └─ High risk → BLOCK (likely stolen credentials)
```

**Configuration Example**:

```json
{
  "Name": "AccountTakeoverPrevention",
  "Priority": 3,
  "Statement": {
    "ManagedRuleGroupStatement": {
      "VendorName": "AWS",
      "Name": "AWSManagedRulesATPRuleSet",
      "ManagedRuleGroupConfigs": [
        {
          "AWSManagedRulesATPRuleSet": {
            "LoginPath": "/api/login",  // Where login happens
            "RequestInspection": {
              "PayloadType": "JSON",
              "UsernameField": {
                "Identifier": "/username"  // JSON path to username
              },
              "PasswordField": {
                "Identifier": "/password"  // JSON path to password
              }
            },
            "ResponseInspection": {
              "StatusCode": {
                "SuccessCodes": [200],  // Successful login status
                "FailureCodes": [401]   // Failed login status
              }
            }
          }
        }
      ]
    }
  }
}
```

**Simple Explanation**:
ATP needs to know:
- **Where is your login endpoint?** (`/api/login`)
- **How do you send credentials?** (JSON with `username` and `password` fields)
- **How do you indicate success/failure?** (HTTP 200 = success, 401 = failure)

With this info, ATP learns normal patterns and detects anomalies.

**Cost**: $10/month + $1.50 per 1000 login attempts analyzed

**Real-World Example**:
Streaming service with 10 million users:
- Attackers try credentials from LinkedIn breach
- ATP detects:
  - 50,000 login attempts from same IP
  - Credentials match known breached database
  - Login attempts from unusual geolocations
- Result: Block 99% of account takeover attempts, save customer accounts

---

### Custom Rules

**What They Are**: Rules you write yourself for application-specific logic that managed rules don't cover.

**When to Use Custom Rules**:
- Block specific URL patterns unique to your app
- Implement custom authentication checks
- Enforce business logic (e.g., "only allow API calls with valid license key")
- Country-specific restrictions (GDPR compliance)
- Block specific referer domains (hotlinking protection)

**Example Custom Rules**:

**1. Block Admin Panel from Internet**:
```json
{
  "Name": "BlockAdminFromInternet",
  "Priority": 1,
  "Action": {"Block": {}},
  "Statement": {
    "AndStatement": {
      "Statements": [
        {
          // Match /admin path
          "ByteMatchStatement": {
            "FieldToMatch": {"UriPath": {}},
            "SearchString": "/admin",
            "PositionalConstraint": "STARTS_WITH"
          }
        },
        {
          // NOT from corporate IP range
          "NotStatement": {
            "Statement": {
              "IPSetReferenceStatement": {
                "Arn": "arn:aws:wafv2:::ipset/corporate-ips"
              }
            }
          }
        }
      ]
    }
  }
}
```
**Translation**: Block requests to `/admin/*` unless they come from corporate IP range.

**2. Require API Key Header**:
```json
{
  "Name": "RequireAPIKey",
  "Priority": 2,
  "Action": {"Block": {}},
  "Statement": {
    "AndStatement": {
      "Statements": [
        {
          // Applies to /api paths
          "ByteMatchStatement": {
            "FieldToMatch": {"UriPath": {}},
            "SearchString": "/api",
            "PositionalConstraint": "STARTS_WITH"
          }
        },
        {
          // Missing X-API-Key header
          "NotStatement": {
            "Statement": {
              "SizeConstraintStatement": {
                "FieldToMatch": {
                  "SingleHeader": {"Name": "x-api-key"}
                },
                "ComparisonOperator": "GT",
                "Size": 0
              }
            }
          }
        }
      ]
    }
  }
}
```
**Translation**: Block `/api/*` requests that don't have an `X-API-Key` header.

**3. Geo-Blocking**:
```json
{
  "Name": "BlockRussiaChinaNorthKorea",
  "Priority": 5,
  "Action": {"Block": {}},
  "Statement": {
    "GeoMatchStatement": {
      "CountryCodes": ["RU", "CN", "KP"]
    }
  }
}
```
**Translation**: Block all requests from Russia, China, and North Korea.

**Text Transformations** (important for evasion prevention):

```json
"TextTransformations": [
  {"Type": "LOWERCASE", "Priority": 0},      // Convert to lowercase
  {"Type": "URL_DECODE", "Priority": 1},     // Decode %20 → space
  {"Type": "HTML_ENTITY_DECODE", "Priority": 2},  // Decode &lt; → <
  {"Type": "NORMALIZE_PATH", "Priority": 3}  // Remove /./ and /../
]
```

**Why Transformations Matter**:
Attacker tries to bypass SQL injection filter:
- Without transformation: `SELECT * FROM users` → Blocked
- With evasion: `%53ELECT * FROM users` (URL-encoded S)
- WAF with URL_DECODE: Decodes → `SELECT * FROM users` → Blocked ✅

---

## Shield

### What It Is
AWS Shield is a **DDoS (Distributed Denial of Service) protection service** that safeguards applications running on AWS.

### Why It Exists

**The DDoS Problem**:

```
┌──────────────────────────────────────────────────────────┐
│         Typical DDoS Attack                               │
└──────────────────────────────────────────────────────────┘

Attacker controls botnet of 100,000 infected devices
         │
         │ All send requests simultaneously
         ▼
┌─────────────────────────────────────────┐
│  Your Website                           │
│  ├─ Normal capacity: 1,000 req/sec      │
│  ├─ Attack traffic: 100,000 req/sec     │
│  └─ Result: Server crashes              │
└─────────────────────────────────────────┘
         │
         ▼
Legitimate users can't access site
Business loses money
Reputation damaged
```

**Types of DDoS Attacks**:

1. **Infrastructure Layer (Layer 3/4)**:
   - **SYN Flood**: Send millions of TCP SYN packets, exhaust connection table
   - **UDP Flood**: Flood with UDP packets (DNS amplification)
   - **ICMP Flood**: Ping flood

2. **Application Layer (Layer 7)**:
   - **HTTP Flood**: Send millions of valid-looking HTTP requests
   - **Slowloris**: Open connections, keep them alive forever
   - **DNS Query Flood**: Overwhelm DNS servers

**Traditional DDoS Mitigation (Expensive)**:
- Buy dedicated DDoS scrubbing hardware: $50,000-500,000
- Contract with DDoS mitigation service: $3,000-10,000/month
- Deploy globally (multiple data centers): $100,000+/month

**AWS Shield Solution**:
- Shield Standard: **Free**, always on
- Shield Advanced: $3,000/month, enterprise protection

---

### Shield Standard

**What It Is**: Automatic, always-on DDoS protection for all AWS customers at **no additional cost**.

**What It Protects**:
- Layer 3/4 attacks (SYN floods, UDP floods, reflection attacks)
- ~96% of known infrastructure-layer DDoS attacks

**How It Works**:

```
┌──────────────────────────────────────────────────────────┐
│         Shield Standard Architecture                      │
└──────────────────────────────────────────────────────────┘

Internet (DDoS Attack)
         │
         │ 100,000 requests/sec
         ▼
┌─────────────────────────────────────┐
│  AWS Edge Locations                 │
│  (Shield Standard Auto-Detection)   │
│  ├─ Inspect traffic patterns        │
│  ├─ Identify anomalies               │
│  └─ Filter malicious packets         │
└──────┬──────────────────────────────┘
       │
       │ 1,000 requests/sec (legitimate traffic only)
       ▼
┌─────────────────────────────────┐
│  Your Application               │
│  (CloudFront, Route 53, ALB)    │
└─────────────────────────────────┘
```

**Included Services**:
- ✅ Amazon CloudFront
- ✅ Amazon Route 53
- ✅ AWS Global Accelerator
- ✅ Elastic Load Balancing (ALB, NLB, CLB)
- ❌ EC2 instances (not protected by Standard)

**Limitations**:
- No Layer 7 (application) protection
- No DDoS cost protection (you pay for attack traffic)
- No DDoS Response Team (DRT) support
- No real-time attack notifications
- Limited visibility into attacks

**Who Should Use Shield Standard**:
Everyone (it's free and automatic). You don't need to enable it.

---

### Shield Advanced

**What It Is**: Enterprise-grade DDoS protection with **24/7 DDoS Response Team, cost protection, and application-layer defense**.

**Cost**: $3,000/month + data transfer fees

**What You Get**:

```
┌──────────────────────────────────────────────────────────┐
│      Shield Advanced vs Shield Standard                  │
└──────────────────────────────────────────────────────────┘

Shield Standard (Free):
├─ Layer 3/4 protection
├─ Automatic detection
└─ Basic filtering

Shield Advanced ($3,000/month):
├─ Everything in Standard
├─ Layer 7 (application) protection
├─ DDoS Response Team (DRT) 24/7
├─ Cost protection (refund attack charges)
├─ Advanced attack visibility
├─ WAF credits ($3,000/month)
├─ Health-based detection
├─ Custom mitigations
└─ 1-year commitment
```

**Key Features**:

**1. DDoS Cost Protection**:

Without Shield Advanced:
```
DDoS attack generates 10TB of data transfer
AWS charges: 10,000 GB × $0.085/GB = $850
You pay: $850 ❌
```

With Shield Advanced:
```
DDoS attack generates 10TB of data transfer
AWS charges: $0
Shield Advanced refunds attack-related charges ✅
```

**2. DDoS Response Team (DRT)**:
- 24/7/365 phone support during attacks
- Expert team analyzes attack in real-time
- Creates custom WAF rules on your behalf
- Proactive mitigation recommendations

**3. Application Layer Protection**:
Shield Advanced includes:
- Automatic WAF rule creation
- Layer 7 attack detection (HTTP floods)
- Integration with AWS WAF

**4. Health-Based Detection**:

```
┌──────────────────────────────────────────────────────────┐
│      Health-Based Detection                               │
└──────────────────────────────────────────────────────────┘

Normal Operation:
├─ CloudWatch Alarm: TargetResponseTime < 500ms ✅
├─ Shield: No attack detected
└─ Traffic flowing normally

DDoS Attack Begins:
├─ CloudWatch Alarm: TargetResponseTime > 2000ms ❌
├─ Shield detects health degradation
├─ Shield escalates to DRT
└─ DRT applies mitigations
      │
      ▼
Response time returns to normal within minutes
```

**Real-World Use Case**:
E-commerce site during Black Friday:
- Attack: 500,000 requests/sec HTTP flood
- Shield Advanced detects health degradation
- DRT creates WAF rate-limiting rules in 5 minutes
- Attack mitigated, site stays online
- AWS refunds $12,000 in attack-related charges
- Total cost: $3,000/month Shield Advanced (vs $12,000 one-time + downtime)

---

### DDoS Protection Best Practices

**Architecture for DDoS Resilience**:

```
┌──────────────────────────────────────────────────────────┐
│       DDoS-Resistant Architecture                         │
└──────────────────────────────────────────────────────────┘

Internet
   │
   ├─► Route 53 (Shield Standard protected)
   │     │
   │     └─► CloudFront (Shield Standard + WAF)
   │           │
   │           └─► ALB (Shield Advanced protected)
   │                 │
   │                 └─► Auto Scaling Group (scales to absorb)
   │                       │
   │                       └─► EC2 Instances (stateless)
   │
   └─► Global Accelerator (Shield Standard)
         │
         └─► NLB (Shield Advanced protected)
               │
               └─► Backend servers
```

**Defense-in-Depth Strategy**:
1. **Route 53**: Anycast DNS prevents DNS flood attacks
2. **CloudFront**: Caches content, absorbs traffic at edge
3. **WAF**: Filters malicious Layer 7 requests
4. **Shield Advanced**: Detects and mitigates DDoS
5. **Auto Scaling**: Scales to meet legitimate demand
6. **Health Checks**: Quickly route around unhealthy resources

---

## GuardDuty

### What It Is
Amazon GuardDuty is an **intelligent threat detection service** that continuously monitors your AWS accounts and workloads for malicious activity using machine learning.

### Why It Exists

**The Problem**:
Security threats in the cloud:
- Stolen IAM credentials used from Russia
- EC2 instance mining cryptocurrency
- S3 bucket exfiltration (data theft)
- Compromised container executing malware
- Backdoor created by attacker

**Traditional Detection Requires**:
- Parse billions of CloudTrail logs manually
- Analyze VPC Flow Logs (IP traffic patterns)
- Monitor DNS queries for suspicious domains
- Correlate across multiple log sources
- Hire security analysts 24/7

**GuardDuty Solution**:
- Ingests logs automatically (CloudTrail, VPC Flow, DNS, S3, EKS, Lambda, RDS)
- Uses ML models trained on AWS threat intelligence
- Generates findings (alerts) for suspicious activity
- Integrates with EventBridge for automated response

```
┌──────────────────────────────────────────────────────────┐
│         GuardDuty Architecture                            │
└──────────────────────────────────────────────────────────┘

AWS Account
   ├─ CloudTrail Logs (API calls)
   ├─ VPC Flow Logs (network traffic)
   ├─ DNS Logs (DNS queries)
   ├─ S3 Data Events (object access)
   ├─ EKS Audit Logs (Kubernetes API)
   ├─ Lambda Network Activity
   └─ RDS Login Activity
         │
         │ Continuous ingestion (no config needed)
         ▼
┌─────────────────────────────────────┐
│  GuardDuty Service                  │
│  ├─ Machine Learning Models         │
│  ├─ Threat Intelligence Feeds       │
│  ├─ Anomaly Detection               │
│  └─ Known Attack Patterns           │
└──────┬──────────────────────────────┘
       │
       │ Generates findings
       ▼
┌─────────────────────────────────────┐
│  GuardDuty Findings                 │
│  ├─ Severity: High/Medium/Low       │
│  ├─ Type: UnauthorizedAccess        │
│  ├─ Resource: i-0123456789abcdef    │
│  └─ Recommendation: Isolate instance│
└──────┬──────────────────────────────┘
       │
       ├─► Security Hub (centralize)
       ├─► EventBridge (automate response)
       └─► SNS (alert SOC team)
```

**Cost**:
- $0.50 per 1M CloudTrail events analyzed
- $1.14 per GB of VPC Flow Logs analyzed
- $0.50 per GB of DNS logs analyzed
- Free trial: 30 days

**Key Features**:
- Enable with one click (no agents, no infrastructure)
- Analyzes billions of events across accounts
- Continuously updated threat intelligence
- Integrates with AWS Organizations (multi-account)
- Can be automated with EventBridge

---

### Threat Detection (Finding Types)

GuardDuty categorizes threats into finding types. Here are the most important ones:

**1. UnauthorizedAccess (Compromised Credentials)**:

```
Finding: UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration

Meaning: IAM credentials for an EC2 instance are being used from
         outside AWS (suspicious)

Example Scenario:
- Attacker compromises EC2 instance
- Steals IAM role credentials from metadata service
- Uses them from their laptop in Russia
- GuardDuty detects: "These credentials should only be used from EC2"

Recommendation:
- Revoke credentials
- Investigate EC2 instance for compromise
- Rotate IAM role
```

**2. Recon (Reconnaissance / Scanning)**:

```
Finding: Recon:EC2/PortProbeUnprotectedPort

Meaning: Someone is scanning your EC2 instances for open ports
         (attacker reconnaissance phase)

Example Scenario:
- Attacker scans IP range 203.0.113.0/24
- Finds port 22 (SSH) open on your EC2
- Tests for weak passwords
- GuardDuty detects abnormal port scanning patterns

Recommendation:
- Review security group rules
- Ensure SSH uses key-based auth only
- Block source IP if malicious
```

**3. CryptoCurrency (Mining)**:

```
Finding: CryptoCurrency:EC2/BitcoinTool.B!DNS

Meaning: EC2 instance is communicating with known cryptocurrency
         mining pool (likely compromised)

Example Scenario:
- Attacker exploits vulnerable web app
- Installs cryptocurrency miner
- Instance starts mining Bitcoin
- DNS queries to mining-pool.com detected

Recommendation:
- Terminate instance immediately
- Investigate: How was it compromised?
- Launch new instance from known-good AMI
```

**4. Trojan (Backdoor / Malware)**:

```
Finding: Trojan:EC2/DNSDataExfiltration

Meaning: Instance is using DNS queries to exfiltrate data
         (classic exfiltration technique)

Example Scenario:
- Attacker steals database
- Encodes data in DNS queries:
  DNS query: stolen-data-abc123.attacker.com
- Data sent via DNS (bypasses firewall)

Recommendation:
- Isolate instance
- Capture memory dump for forensics
- Terminate and rebuild
```

**5. Backdoor**:

```
Finding: Backdoor:EC2/C&CActivity.B!DNS

Meaning: Instance is communicating with known command-and-control
         (C&C) server (attacker controlling compromised host)

Example Scenario:
- Malware installed on EC2
- Connects to C&C server for instructions
- Waits for commands (download more malware, launch attacks)

Recommendation:
- Immediate isolation (remove from network)
- Forensic analysis
- Report to AWS Abuse team
```

**Severity Levels**:

```
┌─────────────────────────────────────────────────────────┐
│  Finding Severity                                       │
└─────────────────────────────────────────────────────────┘

HIGH (8.0 - 10.0):
- Immediate threat
- Active compromise likely
- Action required within 1 hour
- Examples: Backdoor, CryptoCurrency, UnauthorizedAccess

MEDIUM (4.0 - 7.9):
- Suspicious activity
- Potential threat
- Investigate within 24 hours
- Examples: Unusual API calls, Recon activity

LOW (0.1 - 3.9):
- Informational
- May be benign
- Review when convenient
- Examples: Port scan from known security scanner
```

---

### Malware Protection

**What It Is**: GuardDuty can scan EBS volumes and EC2 instances for malware.

**How It Works**:

```
┌──────────────────────────────────────────────────────────┐
│         Malware Protection Process                        │
└──────────────────────────────────────────────────────────┘

GuardDuty detects suspicious behavior
         │
         │ (e.g., instance communicating with C&C server)
         ▼
┌─────────────────────────────────────┐
│ Trigger Malware Scan                │
│ - Create EBS snapshot                │
│ - Scan snapshot for malware          │
│ - Delete snapshot after scan         │
└──────┬──────────────────────────────┘
       │
       ├─ Malware found → Detailed finding with:
       │                  ├─ Malware name
       │                  ├─ File path
       │                  └─ Threat severity
       │
       └─ No malware → Lower severity, investigate behavior
```

**Use Cases**:
- Automated malware scanning on suspicious findings
- Scan newly launched instances (ensure clean)
- Scan after security incident (forensics)

**Cost**: $0.18 per GB scanned

---

### S3 Protection

**What It Is**: Detects suspicious S3 bucket activity, especially data exfiltration.

**What It Detects**:

```
1. Exfiltration:S3/ObjectRead.Unusual
   - Unusual volume of data downloaded
   - Example: 500GB downloaded in 10 minutes (normal: 1GB/day)

2. Impact:S3/PermissionsModification.Unusual
   - Bucket policy changed to public
   - ACLs modified to allow external access

3. PenTest:S3/KaliLinux
   - Access from Kali Linux (penetration testing OS)
   - Likely unauthorized security testing

4. UnauthorizedAccess:S3/MaliciousIPCaller.Custom
   - Access from known malicious IP
   - Threat intelligence match
```

**Real-World Example**:

```
Normal Behavior:
- Your app reads 100 objects/day from S3
- Access always from us-east-1

Suspicious Behavior:
- Suddenly 10,000 objects read in 1 hour
- Access from new IP in Russia
- Objects contain customer PII

GuardDuty Finding:
- Type: Exfiltration:S3/ObjectRead.Unusual
- Severity: HIGH
- Recommendation: Revoke access, investigate credentials
```

---

### EKS Protection

**What It Is**: Monitors Kubernetes clusters for threats.

**What It Detects**:

```
1. PrivilegeEscalation:Kubernetes/PrivilegedContainer
   - Pod running with excessive privileges
   - Container running as root

2. Execution:Kubernetes/ExecInKubeSystemPod
   - Someone executing commands in kube-system namespace
   - Possible cluster compromise

3. Persistence:Kubernetes/ContainerWithSensitiveMount
   - Container mounted sensitive host paths (/etc, /var/run)
   - Potential persistence mechanism

4. Discovery:Kubernetes/SuccessfulAnonymousAccess
   - Unauthenticated API access succeeded
   - API server misconfiguration
```

**How EKS Protection Works**:

```
Kubernetes Cluster
   ├─ API Server Logs (kubectl commands, pod creation)
   ├─ Audit Logs (authentication, authorization)
   └─ Runtime Behavior (network connections)
         │
         ▼
GuardDuty EKS Protection
   ├─ Analyzes API calls
   ├─ Detects privilege escalation
   ├─ Monitors container behavior
   └─ Flags anomalous patterns
         │
         ▼
Finding: "Anonymous user created admin role"
```

---

### Lambda Protection

**What It Is**: Detects threats in Lambda function execution.

**What It Detects**:

```
1. CryptoCurrency:Lambda/BitcoinTool.B!DNS
   - Lambda function mining cryptocurrency

2. Execution:Lambda/MaliciousFile
   - Lambda executing known malware

3. UnauthorizedAccess:Lambda/MaliciousIPCaller.Custom
   - Lambda invoked from malicious IP

4. Backdoor:Lambda/C&CActivity.B!DNS
   - Lambda communicating with C&C server
```

**Use Case**:
Serverless API compromised:
- Attacker exploits code injection vulnerability
- Lambda function modified to exfiltrate data
- GuardDuty detects unusual network activity
- Alert sent → Automated response disables function

---

### RDS Protection

**What It Is**: Detects suspicious database login activity.

**What It Detects**:

```
1. CredentialAccess:RDS/AnomalousLogin.SuccessfulLogin
   - Login from unusual location (suddenly from China)
   - Login at unusual time (3 AM, user normally logs in at 9 AM)

2. UnauthorizedAccess:RDS/MaliciousIPCaller.Custom
   - Database accessed from known malicious IP

3. UnauthorizedAccess:RDS/TorIPCaller
   - Database accessed via Tor network
```

**How It Works**:

```
RDS Database Login Attempt
   ├─ Username: admin
   ├─ Source IP: 198.51.100.5 (Russia)
   ├─ Time: 3:00 AM UTC
   └─ Previous logins always from us-east-1, 9 AM EST
         │
         ▼
GuardDuty analyzes:
   ├─ IP geolocation (anomalous)
   ├─ Time of day (anomalous)
   ├─ Threat intelligence (IP flagged)
   └─ User behavior baseline (unusual pattern)
         │
         ▼
Finding: CredentialAccess:RDS/AnomalousLogin.SuccessfulLogin
Severity: HIGH
Recommendation: Rotate credentials, investigate
```

---

## Security Hub

### What It Is
AWS Security Hub is a **centralized security management service** that aggregates, organizes, and prioritizes security findings from multiple AWS services and third-party tools.

### Why It Exists

**The Problem**:

Without Security Hub:
```
Security Team's Daily Nightmare:
├─ Check GuardDuty console (threat detections)
├─ Check Inspector console (vulnerability scans)
├─ Check Macie console (data discovery)
├─ Check IAM Access Analyzer console
├─ Check Config console (compliance)
├─ Check third-party tools (CrowdStrike, Splunk)
└─ Manually correlate findings across 10+ consoles

Result: Alerts missed, slow response, analyst burnout
```

With Security Hub:
```
Single Dashboard:
All findings from all sources
Prioritized by severity
Automated workflows
Compliance status visible
```

**What Security Hub Does**:

```
┌──────────────────────────────────────────────────────────┐
│         Security Hub Architecture                         │
└──────────────────────────────────────────────────────────┘

AWS Security Services:
├─ GuardDuty (findings) ────┐
├─ Inspector (findings) ─────┤
├─ Macie (findings) ─────────┤
├─ IAM Access Analyzer ──────┤
├─ Firewall Manager ─────────┤
└─ Systems Manager ──────────┤
                             │
Third-Party Integrations:    │
├─ CrowdStrike ──────────────┤
├─ Palo Alto Networks ───────┤
├─ Splunk ───────────────────┤
└─ Tenable ──────────────────┤
                             │
                             ▼
                  ┌──────────────────────┐
                  │   Security Hub       │
                  │  ┌────────────────┐  │
                  │  │ All Findings   │  │
                  │  │ Normalized     │  │
                  │  │ Prioritized    │  │
                  │  └────────┬───────┘  │
                  └───────────┼──────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
              EventBridge          SIEM/Dashboard
              (automate)           (visualize)
```

---

### Security Standards

**What They Are**: Pre-configured rule sets that check your AWS environment against industry compliance frameworks.

**Available Standards**:

```
┌──────────────────────────────────────────────────────────┐
│         Security Hub Standards                            │
└──────────────────────────────────────────────────────────┘

1. AWS Foundational Security Best Practices (FSBP)
   - AWS's own security recommendations
   - 200+ automated checks
   - Examples:
     ├─ S3 buckets should not be public
     ├─ RDS should have encryption enabled
     ├─ IAM root user should have MFA
     └─ CloudTrail should be enabled

2. CIS AWS Foundations Benchmark
   - Industry-standard security baseline
   - 50+ checks
   - Examples:
     ├─ Password policy requires minimum 14 characters
     ├─ MFA enabled for all IAM users
     └─ VPC flow logging enabled

3. PCI DSS (Payment Card Industry Data Security Standard)
   - For handling credit card data
   - 100+ checks
   - Examples:
     ├─ Encryption in transit for all data
     ├─ Regular vulnerability scanning
     └─ Access control for cardholder data

4. NIST 800-53 Rev. 5
   - US government security controls
   - 200+ checks

5. GDPR (General Data Protection Regulation)
   - EU data privacy compliance
```

**How Standards Work**:

```
Enable CIS AWS Foundations Benchmark
         │
         ▼
Security Hub runs 50 checks across your account
         │
         ├─ Check 1.1: Root account MFA enabled?
         │    └─ Status: FAILED (no MFA)
         ├─ Check 2.1: CloudTrail enabled in all regions?
         │    └─ Status: PASSED
         ├─ Check 4.1: S3 buckets not public?
         │    └─ Status: FAILED (3 public buckets found)
         │
         ▼
Security Hub Score: 67% compliant
   ├─ 34 checks passed ✅
   ├─ 16 checks failed ❌
   └─ Remediation recommendations provided
```

**Compliance Dashboard**:

```
AWS Foundational Security Best Practices:
┌─────────────────────────────────────────┐
│ Overall Score: 78%                      │
│                                         │
│ Critical: 3 findings                    │
│ High: 12 findings                       │
│ Medium: 45 findings                     │
│ Low: 89 findings                        │
│                                         │
│ Top Issues:                             │
│ 1. S3.1 - Block public access (15 S3)  │
│ 2. EC2.8 - EBS encryption (42 volumes) │
│ 3. IAM.6 - Password policy weak        │
└─────────────────────────────────────────┘
```

---

### Findings

**What They Are**: Individual security issues detected by Security Hub or integrated services.

**Finding Format (ASFF - AWS Security Finding Format)**:

```json
{
  "SchemaVersion": "2018-10-08",
  "Id": "arn:aws:securityhub:...:finding/12345",
  "ProductArn": "arn:aws:securityhub:...:product/aws/guardduty",
  "GeneratorId": "arn:aws:guardduty:...:detector/abc123",
  "AwsAccountId": "123456789012",
  "Types": ["TTPs/Command and Control/Backdoor:EC2/C&CActivity.B!DNS"],
  "CreatedAt": "2026-02-06T10:15:30Z",
  "UpdatedAt": "2026-02-06T10:15:30Z",
  "Severity": {
    "Product": 8.0,
    "Label": "HIGH",
    "Normalized": 80
  },
  "Title": "EC2 instance communicating with C&C server",
  "Description": "Instance i-0123abc is contacting known C&C domain evil.com",
  "Remediation": {
    "Recommendation": {
      "Text": "Isolate instance, investigate compromise, terminate"
    }
  },
  "Resources": [
    {
      "Type": "AwsEc2Instance",
      "Id": "arn:aws:ec2:us-east-1:123456789012:instance/i-0123abc",
      "Region": "us-east-1"
    }
  ],
  "Workflow": {
    "Status": "NEW"
  },
  "RecordState": "ACTIVE"
}
```

**Finding Lifecycle**:

```
NEW → NOTIFIED → RESOLVED → SUPPRESSED

NEW: Finding just created
NOTIFIED: Sent to security team
RESOLVED: Issue fixed
SUPPRESSED: False positive, ignored
```

**Finding Aggregation**:
Security Hub de-duplicates identical findings:
- GuardDuty finds: "EC2 port scan from 203.0.113.5"
- Inspector finds: "EC2 vulnerable to port scan"
- Security Hub: Aggregates as 1 finding with 2 sources

---

### Automated Response

**What It Is**: Using EventBridge to automatically remediate security findings.

**Why It Matters**:
Manual remediation doesn't scale:
- GuardDuty finds 50 compromised instances
- Security analyst must manually isolate each one
- Takes hours, attacker has time to spread

Automated response:
- Finding generated → EventBridge rule triggers → Lambda isolates instance → All 50 isolated in seconds

**Architecture**:

```
┌──────────────────────────────────────────────────────────┐
│         Automated Response Flow                           │
└──────────────────────────────────────────────────────────┘

1. Security Finding Generated
   GuardDuty: "EC2 instance compromised"
         │
         ▼
2. Finding sent to Security Hub
   (normalized format)
         │
         ▼
3. Security Hub sends event to EventBridge
         │
         ▼
4. EventBridge Rule matches finding type
   "If finding.Severity = HIGH and
    finding.Type = Backdoor:EC2/*"
         │
         ▼
5. EventBridge invokes Lambda function
         │
         ▼
6. Lambda performs remediation:
   ├─ Create snapshot (forensics)
   ├─ Isolate instance (block all traffic)
   ├─ Tag instance (mark as compromised)
   ├─ Create JIRA ticket
   └─ Notify security team (SNS)
```

**Example EventBridge Rule**:

```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": {
        "Label": ["HIGH", "CRITICAL"]
      },
      "Workflow": {
        "Status": ["NEW"]
      },
      "ProductName": ["GuardDuty"]
    }
  }
}
```

**Common Automated Remediations**:

```
┌──────────────────────────────────────────────────────────┐
│         Remediation Examples                              │
└──────────────────────────────────────────────────────────┘

Finding: S3 bucket became public
Remediation:
├─ Block public access
├─ Remove bucket policy allowing public reads
├─ Tag bucket with "remediated-by-automation"
└─ Notify bucket owner

Finding: IAM user without MFA
Remediation:
├─ Disable access keys
├─ Force password reset
├─ Send email to user with MFA setup instructions
└─ Track in compliance dashboard

Finding: Unencrypted EBS volume
Remediation:
├─ Create snapshot
├─ Create encrypted volume from snapshot
├─ Attach encrypted volume
├─ Delete unencrypted volume
└─ Update instance tags

Finding: EC2 instance compromised
Remediation:
├─ Create EBS snapshot (forensics)
├─ Isolate instance (security group: deny all)
├─ Create memory dump (advanced forensics)
├─ Tag instance: "compromised-do-not-delete"
├─ Notify on-call engineer (PagerDuty)
└─ Create incident ticket (ServiceNow)
```

**Example Lambda Remediation Function**:

```python
import boto3

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    # Extract instance ID from Security Hub finding
    finding = event['detail']['findings'][0]
    instance_id = finding['Resources'][0]['Id'].split('/')[-1]

    # Create snapshot for forensics
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'attachment.instance-id', 'Values': [instance_id]}]
    )
    for volume in volumes['Volumes']:
        ec2.create_snapshot(
            VolumeId=volume['VolumeId'],
            Description=f'Forensic snapshot: {instance_id}',
            TagSpecifications=[{
                'ResourceType': 'snapshot',
                'Tags': [{'Key': 'Purpose', 'Value': 'Forensics'}]
            }]
        )

    # Isolate instance by attaching security group that denies all traffic
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=['sg-isolation-deny-all']  # Pre-created SG with no rules
    )

    # Tag instance
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'SecurityStatus', 'Value': 'Compromised'},
            {'Key': 'AutoRemediated', 'Value': 'true'},
            {'Key': 'RemediationTime', 'Value': str(datetime.now())}
        ]
    )

    # Send notification (SNS, Slack, PagerDuty)
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
        Subject='EC2 Instance Compromised and Isolated',
        Message=f'Instance {instance_id} has been isolated. Snapshot created.'
    )

    return {'status': 'remediated', 'instance': instance_id}
```

---

### Cross-Account Aggregation

**What It Is**: Consolidating security findings from **multiple AWS accounts** into a single Security Hub account.

**Why It Matters**:

**Without Aggregation**:
```
Company with 50 AWS accounts (dev, staging, prod per team)
Security team must:
├─ Log into Account 1 → Check Security Hub
├─ Log into Account 2 → Check Security Hub
├─ Log into Account 3 → Check Security Hub
└─ ... (47 more accounts)

Result: Impossible to maintain, findings missed
```

**With Aggregation**:
```
Security Hub Administrator Account
   ├─ Aggregates findings from all 50 accounts
   ├─ Single dashboard shows all security posture
   └─ Automated responses apply across all accounts
```

**Architecture**:

```
┌──────────────────────────────────────────────────────────┐
│      Cross-Account Security Hub                          │
└──────────────────────────────────────────────────────────┘

AWS Organization
├─ Security Account (Master)
│  └─ Security Hub (Aggregator)
│     ┌────────────────────────────────────┐
│     │ Receives findings from all members │
│     │ Enforces standards across org      │
│     │ Centralized compliance reporting   │
│     └────────────────────────────────────┘
│
├─ Production Account (Member)
│  ├─ GuardDuty → Findings → Security Hub → Sends to Master
│  └─ Inspector → Findings → Security Hub → Sends to Master
│
├─ Development Account (Member)
│  └─ Security Hub → Sends to Master
│
└─ Staging Account (Member)
   └─ Security Hub → Sends to Master
```

**Setup**:
1. Enable AWS Organizations
2. Designate Security Account as Security Hub administrator
3. Enable Security Hub in all member accounts
4. Findings automatically aggregate to administrator account

---

## Inspector

### What It Is
Amazon Inspector is an **automated vulnerability scanner** for EC2 instances, container images (ECR), and Lambda functions.

### Why It Exists

**The Problem**:
Software vulnerabilities:
- CVE-2021-44228 (Log4Shell): Critical vulnerability in Java logging library
- Affects thousands of applications
- Manual patching takes weeks
- How do you know which EC2 instances are vulnerable?

**Traditional Approach**:
- Run vulnerability scanner manually
- Schedule scans weekly
- Parse reports manually
- Track patching progress in spreadsheet
- Re-scan to verify fix

**Inspector Solution**:
- Continuous scanning (automatic, no schedule needed)
- Detects vulnerabilities the moment they're published
- Prioritizes by severity + exploitability
- Integrates with Security Hub (centralized)

---

### Vulnerability Scanning

**What Inspector Scans For**:

```
┌──────────────────────────────────────────────────────────┐
│           Inspector Vulnerability Types                   │
└──────────────────────────────────────────────────────────┘

1. CVE Vulnerabilities (Common Vulnerabilities and Exposures)
   - Known security flaws in software
   - Examples:
     ├─ CVE-2023-12345: OpenSSL buffer overflow
     ├─ CVE-2022-98765: Apache HTTP Server RCE
     └─ CVE-2024-11111: Kernel privilege escalation

2. CIS Benchmark Violations
   - Configuration best practices
   - Examples:
     ├─ SSH allows password authentication (should be key-only)
     ├─ Firewall disabled
     └─ Weak file permissions

3. Network Reachability Issues
   - Unintended network exposure
   - Examples:
     ├─ Database port 3306 open to internet
     ├─ Admin panel accessible from 0.0.0.0/0
     └─ Internal service exposed via misconfigured security group
```

**How Inspector Works**:

```
┌──────────────────────────────────────────────────────────┐
│         Inspector Scanning Process                        │
└──────────────────────────────────────────────────────────┘

1. EC2 Scanning:
   EC2 Instance
      │
      │ SSM Agent installed (no extra software needed)
      ▼
   Inspector queries:
   ├─ Installed packages (rpm -qa, dpkg -l)
   ├─ Operating system version
   ├─ Running services
   └─ Configuration files
      │
      ▼
   Compare against vulnerability database:
   ├─ Is package version vulnerable to CVE-2023-12345?
   ├─ Is SSH config following CIS benchmark?
   └─ Generate finding if vulnerable

2. ECR Scanning:
   Container Image pushed to ECR
      │
      ▼
   Inspector automatically scans layers:
   ├─ Extract all installed packages
   ├─ Check for vulnerable libraries
   ├─ Scan OS packages
   └─ Generate finding if vulnerable
      │
      ▼
   Image tagged with scan results:
   - sha256:abc123 → 15 CRITICAL, 42 HIGH, 89 MEDIUM

3. Lambda Scanning:
   Lambda function code + dependencies
      │
      ▼
   Inspector scans:
   ├─ Lambda runtime (Node.js, Python version)
   ├─ Dependencies (package.json, requirements.txt)
   ├─ Layers
   └─ Application code
```

---

### EC2 Scanning

**What It Scans**:
- Operating system packages (yum, apt packages)
- Installed applications
- Configuration files
- Network exposure

**How It Works**:

```
Continuous Scanning (No Manual Trigger Needed):

New CVE published: CVE-2024-99999 (OpenSSL 3.0.7 vulnerable)
         │
         ▼
Inspector checks all EC2 instances:
├─ Instance i-abc123: OpenSSL 3.0.7 installed → VULNERABLE ❌
├─ Instance i-def456: OpenSSL 3.1.0 installed → NOT AFFECTED ✅
└─ Instance i-ghi789: OpenSSL 3.0.7 installed → VULNERABLE ❌
         │
         ▼
Findings generated:
- Severity: CRITICAL
- CVSS Score: 9.8
- Recommendation: "Upgrade OpenSSL to 3.0.8"
- Exploitability: HIGH (public exploit available)
```

**Example Finding**:

```
Title: CVE-2024-99999 - OpenSSL vulnerability
Severity: CRITICAL (9.8)
Description: Buffer overflow in OpenSSL allows remote code execution
Affected Resource: i-0123abc (webserver-prod-1)
Package: openssl-3.0.7-1.amzn2023
Fixed Version: openssl-3.0.8-1.amzn2023
Remediation: Run `sudo yum update openssl`
Exploitability: Public exploit available
```

**Inspector Risk Score**:
Inspector calculates a risk score based on:
- CVSS score (industry-standard severity)
- Exploitability (is there a public exploit?)
- Network exposure (is instance internet-facing?)
- Data sensitivity (does it access databases?)

```
Example Risk Calculation:

CVE-2024-99999 on internal instance:
├─ CVSS: 9.8 (CRITICAL)
├─ Exploit available: Yes (+2)
├─ Internet-facing: No (-1)
└─ Inspector Risk Score: 8.5 (HIGH)

Same CVE on internet-facing instance:
├─ CVSS: 9.8 (CRITICAL)
├─ Exploit available: Yes (+2)
├─ Internet-facing: Yes (+1)
└─ Inspector Risk Score: 10.0 (CRITICAL)
```

This helps prioritize: Patch internet-facing instances first.

---

### ECR Scanning

**What It Is**: Automated vulnerability scanning for Docker container images in Elastic Container Registry.

**Why It Matters**:
- Containers package entire application + dependencies
- One vulnerable library affects every container using it
- Base images (ubuntu:20.04) may have hundreds of packages

**How ECR Scanning Works**:

```
┌──────────────────────────────────────────────────────────┐
│         ECR Image Scanning                                │
└──────────────────────────────────────────────────────────┘

docker push myrepo/app:v1.2.3
         │
         ▼
ECR receives image
         │
         ▼
Inspector automatically scans:
   ├─ Base image layers (ubuntu:20.04)
   ├─ Application layer (your code)
   ├─ Installed packages (apt, pip, npm)
   └─ Dependencies (package.json, requirements.txt)
         │
         ▼
Compare against vulnerability database:
   - CVE-2023-11111: Found in libcurl 7.68.0 → CRITICAL
   - CVE-2023-22222: Found in Node.js 14.17.0 → HIGH
   - CVE-2023-33333: Found in OpenSSL 1.1.1 → MEDIUM
         │
         ▼
Scan results stored with image:
myrepo/app:v1.2.3
├─ CRITICAL: 3
├─ HIGH: 12
├─ MEDIUM: 45
└─ Total: 60 vulnerabilities
```

**Continuous Scanning**:
```
Day 1: Image scanned, 5 vulnerabilities found
Day 30: New CVE published (affects your image)
        Inspector rescans → Now 6 vulnerabilities
        Finding generated → Alert sent
```

**Integration with CI/CD**:

```
CI/CD Pipeline:
1. Build Docker image
2. Push to ECR
3. Inspector scans
4. If CRITICAL vulnerabilities found:
   ├─ Fail deployment ❌
   └─ Block image from being used in production
5. If vulnerabilities acceptable:
   └─ Allow deployment to EKS ✅
```

**Example Policy** (Block deployment if critical vulnerabilities):

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "ecr:PutImage",
      "Condition": {
        "NumericGreaterThan": {
          "ecr:ImageScanFindingsCritical": 0
        }
      }
    }
  ]
}
```

---

### Lambda Scanning

**What It Scans**:
- Lambda runtime version (Python 3.9, Node.js 16, etc.)
- Dependencies (packages in requirements.txt, package.json)
- Lambda layers
- Application code patterns (insecure coding practices)

**How It Works**:

```
Lambda Function: myfunction
   ├─ Runtime: Python 3.9
   ├─ Dependencies: requests, boto3, numpy
   └─ Code: lambda_function.py
         │
         ▼
Inspector scans:
   ├─ Is Python 3.9 vulnerable? (runtime CVEs)
   ├─ Is requests library vulnerable? (CVE-2023-xxxxx)
   ├─ Are there insecure code patterns? (hardcoded secrets)
   └─ Generate findings
         │
         ▼
Finding: "requests library version 2.25.0 has CVE-2023-12345"
Severity: HIGH
Remediation: "Update requirements.txt to requests>=2.31.0"
```

**Scanning Trigger**:
- Automatic: Whenever function code or dependencies change
- Continuous: Daily rescans for new CVEs

---

### Network Reachability

**What It Is**: Inspector analyzes network configuration to find unintended exposure.

**What It Detects**:

```
┌──────────────────────────────────────────────────────────┐
│      Network Reachability Issues                          │
└──────────────────────────────────────────────────────────┘

1. Internet Gateway Exposure
   ┌────────────────────────────────┐
   │  Internet                      │
   └──────────┬─────────────────────┘
              │
              │ Security Group: Allow 0.0.0.0/0:22
              ▼
   ┌─────────────────────────────────┐
   │  EC2: Database Server           │
   │  Port 22 (SSH) OPEN             │
   └─────────────────────────────────┘

   Finding: "Database server SSH accessible from internet"
   Severity: HIGH
   Recommendation: "Restrict SSH to corporate IP range"

2. VPC Peering Misconfiguration
   Account A VPC ←──peering──→ Account B VPC
   │                              │
   │ Security Group allows all    │
   └──────────────────────────────┘

   Finding: "Unintended cross-account access"
   Severity: MEDIUM

3. Unprotected Management Ports
   - RDP (3389) open to 0.0.0.0/0
   - SSH (22) open to 0.0.0.0/0
   - Database (3306, 5432, 1433) open to internet
```

**How It Works**:

Inspector builds network path map:
```
Internet → IGW → Route Table → Subnet → Security Group → EC2
   ├─ Is Security Group allowing 0.0.0.0/0?
   ├─ Is Network ACL blocking?
   ├─ Is instance in public subnet?
   └─ Generate finding if misconfigured
```

---

## Other Security Services

### Macie (Data Discovery)

**What It Is**: Automated **sensitive data discovery service** that uses machine learning to find and protect personally identifiable information (PII) in S3.

**Why It Exists**:
- Company has 10,000 S3 buckets
- Which buckets contain customer credit cards?
- Which buckets contain social security numbers?
- Which are publicly accessible?
- Manual review: Impossible

**What Macie Detects**:

```
┌──────────────────────────────────────────────────────────┐
│         Macie Sensitive Data Types                        │
└──────────────────────────────────────────────────────────┘

Personal Information:
├─ Credit card numbers (Visa, MasterCard, Amex)
├─ Social security numbers
├─ Passport numbers
├─ Driver's license numbers
├─ Email addresses
├─ Phone numbers
└─ Names

Financial Data:
├─ Bank account numbers
├─ SWIFT codes
├─ IBAN numbers
└─ Tax IDs

Health Information (PHI):
├─ Medical record numbers
├─ Health insurance IDs
└─ Prescription data

Custom Patterns:
├─ Employee IDs (regex: EMP-\d{6})
├─ API keys
└─ Internal codes
```

**How Macie Works**:

```
S3 Bucket: customer-data
   ├─ File: customers.csv
   │  └─ Contains: Names, emails, credit cards
   ├─ File: logs.txt
   │  └─ Contains: No sensitive data
   └─ File: backups.zip
      └─ Contains: SSNs, addresses
            │
            ▼
Macie scans files:
   ├─ customers.csv: 5,000 credit card numbers found ❌
   ├─ logs.txt: No PII ✅
   └─ backups.zip: 3,000 SSNs found ❌
            │
            ▼
Findings generated:
- Bucket: customer-data
- Sensitive data: Credit cards (5,000), SSNs (3,000)
- Public access: No
- Encryption: No ❌
- Severity: CRITICAL
```

**Use Cases**:
- GDPR compliance (find EU citizen data)
- PCI-DSS compliance (find credit card data)
- Data migration (identify sensitive data before cloud migration)
- Incident response (what data was exposed in breach?)

**Cost**: $0.10 per GB scanned (one-time) + $0.001 per S3 object monitored (ongoing)

---

### Detective (Investigation)

**What It Is**: Security investigation tool that automatically analyzes and visualizes security data to help investigate incidents.

**Why It Exists**:

**The Problem**:
GuardDuty alerts: "EC2 instance i-abc123 compromised"

Security team needs to answer:
- When did the compromise start?
- What did the attacker do?
- Which other resources did they access?
- Is this part of a larger attack?

**Without Detective**:
```
Manual Investigation (takes hours):
1. Query CloudTrail logs (millions of events)
2. Parse VPC Flow Logs (gigabytes of network data)
3. Correlate GuardDuty findings
4. Draw timeline manually
5. Search for related IPs/users across logs
```

**With Detective**:
```
Automatic Investigation (takes minutes):
1. Open Detective console
2. View visual graph of attack
3. See timeline automatically
4. Click to explore related entities
5. Export report
```

**How Detective Works**:

```
┌──────────────────────────────────────────────────────────┐
│         Detective Data Analysis                           │
└──────────────────────────────────────────────────────────┘

Data Sources (Automatically Ingested):
├─ CloudTrail (API calls)
├─ VPC Flow Logs (network traffic)
├─ GuardDuty findings
└─ EKS audit logs
      │
      │ Detective builds graph database
      ▼
Entities and Relationships:
   EC2 Instance i-abc123
      ├─ Launched by: IAM role prod-app-role
      ├─ Connected to: IP 198.51.100.5 (Russia)
      ├─ API calls: AssumeRole, PutObject (S3)
      └─ Finding: Backdoor:EC2/C&C
            │
            ▼
   IAM Role: prod-app-role
      ├─ Assumed by: Instance i-abc123
      ├─ Also assumed by: Instance i-def456 ← Related incident?
      └─ Permissions: S3 Full Access
            │
            ▼
Visual Investigation Timeline:
Feb 1 10:00: Normal behavior baseline
Feb 3 14:23: First API call to suspicious IP
Feb 3 14:25: Role assumed from new instance
Feb 3 14:30: Large S3 data exfiltration
Feb 3 14:35: GuardDuty finding generated
```

**Use Cases**:
- Investigate GuardDuty findings
- Track lateral movement in attacks
- Find root cause of incidents
- Identify compromised credentials

**Cost**: $2 per GB of data ingested (CloudTrail, VPC Flow Logs)

---

### Firewall Manager

**What It Is**: Centralized management of firewall rules across multiple AWS accounts.

**Why It Exists**:
- Company has 50 AWS accounts
- Each account has WAF rules, Shield, Security Groups
- Inconsistent security policies → Gaps in protection

**What Firewall Manager Does**:
- Define security policy once
- Deploy to all accounts automatically
- Continuous compliance checking
- Remediate violations automatically

**Managed Policies**:

```
┌──────────────────────────────────────────────────────────┐
│         Firewall Manager Policy Types                     │
└──────────────────────────────────────────────────────────┘

1. WAF Policies
   - Deploy WAF rules to all ALBs/CloudFront
   - Example: Block SQL injection across all accounts

2. Shield Advanced Policies
   - Enable Shield Advanced on all resources
   - Example: Protect all ALBs automatically

3. Security Group Policies
   - Enforce common rules (block RDP from internet)
   - Example: "No security group allows 0.0.0.0/0:22"

4. Network Firewall Policies
   - Deploy firewall to all VPCs
   - Inspect traffic for malware

5. Route 53 Resolver DNS Firewall
   - Block malicious domains across organization
```

**Example Use Case**:

```
Central Security Team Policy:
"All public-facing ALBs must have WAF with Core Rule Set"

Firewall Manager:
   ├─ Scans all 50 accounts for ALBs
   ├─ Account 23: ALB without WAF found ❌
   ├─ Automatically attaches WAF with Core Rule Set ✅
   └─ Sends notification to account owner
```

---

### Network Firewall

**What It Is**: Managed stateful firewall for VPCs that inspects traffic at Layer 3-7.

**Why It Exists**:

**Security Groups/NACLs Limitations**:
```
Security Groups can:
├─ Allow/Deny based on IP, port
└─ That's it

Security Groups cannot:
├─ Inspect packet payload (HTTP content)
├─ Detect malware in downloads
├─ Block specific domains
└─ Deep packet inspection
```

**Network Firewall Can**:
```
├─ Stateful inspection (track connections)
├─ Intrusion prevention (IPS)
├─ Web filtering (block malicious domains)
├─ Protocol detection (block non-HTTP on port 80)
└─ Deep packet inspection (find malware signatures)
```

**Architecture**:

```
┌──────────────────────────────────────────────────────────┐
│         Network Firewall Deployment                       │
└──────────────────────────────────────────────────────────┘

Internet
   │
   ▼
Internet Gateway
   │
   ▼
Network Firewall
   ├─ Inspect inbound traffic
   │  ├─ Block malicious domains
   │  ├─ Detect malware
   │  └─ IPS signatures
   │
   ▼
Route to VPC subnets
   │
   ▼
Application servers
```

**Rules**:
```
1. Stateful Rules:
   - Block traffic to known C&C servers
   - Block traffic from specific countries
   - Allow only HTTPS on port 443

2. IPS Rules (Suricata format):
   - Detect SQL injection attempts
   - Block buffer overflow exploits
   - Alert on suspicious patterns

3. Domain Filtering:
   - Block *.malicious-domain.com
   - Allow only *.company.com
```

---

### Verified Access

**What It Is**: Secure access to corporate applications without VPN.

**Why It Exists**:

**VPN Problems**:
- Complex setup (client software)
- Full network access (user can access everything)
- Performance overhead (all traffic tunneled)
- User experience: Slow, disconnects

**Verified Access Solution**:
- Zero Trust Network Access (ZTNA)
- Access based on identity + device posture
- No client software needed
- Granular access (specific apps only)

**How It Works**:

```
Employee opens browser → https://internal-app.company.com
         │
         ▼
Verified Access checks:
   ├─ Is user authenticated? (SSO, Okta)
   ├─ Is device compliant? (antivirus updated, disk encrypted)
   ├─ Is location allowed? (not from blacklisted country)
   └─ Is time allowed? (only during work hours)
         │
         ├─ All checks pass → Allow access ✅
         └─ Any check fails → Deny access ❌
```

**Use Cases**:
- Replace corporate VPN
- Contractor access (limited to specific apps)
- BYOD (bring your own device) policies

---

### Resource Access Manager (RAM)

**What It Is**: Share AWS resources across accounts **without duplicating them**.

**Why It Exists**:

**The Problem**:
```
Company has 10 AWS accounts (teams)
All need access to shared services:
├─ Transit Gateway (networking)
├─ Route 53 Resolver (DNS)
└─ License Manager (software licenses)

Without RAM:
Create duplicate resources in each account → Expensive, hard to manage

With RAM:
Create once, share to all accounts ✅
```

**Shareable Resources**:
```
Network:
├─ VPC subnets
├─ Transit Gateway
├─ Route 53 Resolver rules

Storage:
├─ Aurora DB clusters
├─ EBS snapshots

Security:
├─ KMS keys (cross-account encryption)
├─ License Manager configurations

Compute:
├─ EC2 Capacity Reservations
├─ Dedicated Hosts
```

**Example**:

```
Account A (Shared Services):
   └─ Transit Gateway (connects all accounts)
         │
         │ Shared via RAM
         ▼
   ┌──────────────────────────────────┐
   │  Shared with:                    │
   │  - Account B (Production)        │
   │  - Account C (Development)       │
   │  - Account D (Staging)           │
   └──────────────────────────────────┘

Accounts B, C, D:
   - Attach VPCs to shared Transit Gateway
   - No need to create their own
   - Cost: $0 (only Account A pays)
```

---

## Summary: Choosing the Right Security Service

```
┌──────────────────────────────────────────────────────────┐
│         Security Service Decision Tree                    │
└──────────────────────────────────────────────────────────┘

Need to encrypt data?
└─► KMS (keys) + Secrets Manager (passwords)

Need SSL/TLS certificates?
└─► ACM (public) or ACM Private CA (internal)

Need to block web attacks?
└─► WAF (application layer) + Shield (DDoS)

Need threat detection?
└─► GuardDuty (continuous monitoring)

Need vulnerability scanning?
└─► Inspector (EC2, containers, Lambda)

Need compliance checking?
└─► Security Hub (central dashboard)

Need to find sensitive data?
└─► Macie (S3 data discovery)

Need to investigate incidents?
└─► Detective (forensics)

Need centralized firewall management?
└─► Firewall Manager (multi-account)

Need VPC firewall?
└─► Network Firewall (deep packet inspection)

Need VPN replacement?
└─► Verified Access (zero trust)

Need to share resources?
└─► Resource Access Manager (cross-account)
```

**Common Architectures**:

**Startup (Basic Security)**:
```
├─ KMS: Encrypt S3, RDS
├─ ACM: HTTPS certificates
├─ WAF: Basic web protection
├─ GuardDuty: Threat detection
└─ Security Hub: Central dashboard
   Cost: ~$100-300/month
```

**Enterprise (Advanced Security)**:
```
├─ KMS + Secrets Manager: Encryption + credential management
├─ ACM Private CA: Internal certificates
├─ WAF + Shield Advanced: DDoS + application protection
├─ GuardDuty + Inspector: Threat + vulnerability detection
├─ Security Hub: Multi-account compliance
├─ Macie: Data discovery
├─ Detective: Incident investigation
├─ Firewall Manager: Centralized policies
└─ Network Firewall: Deep packet inspection
   Cost: ~$5,000-20,000/month
```

**Compliance-Heavy (Finance, Healthcare)**:
```
├─ All Enterprise services
├─ CloudHSM: FIPS 140-2 Level 3 encryption
├─ Audit Manager: Continuous compliance
├─ Config: Resource compliance tracking
└─ CloudTrail: Comprehensive audit logging
   Cost: ~$20,000-50,000/month
```

---

## Final Best Practices

1. **Defense in Depth**: Use multiple security layers (WAF + Shield + GuardDuty)
2. **Automate Response**: Use EventBridge to remediate findings automatically
3. **Centralize Monitoring**: Use Security Hub for all accounts
4. **Encrypt Everything**: KMS for data at rest, ACM for data in transit
5. **Scan Continuously**: Enable Inspector for all workloads
6. **Least Privilege**: Use IAM policies with minimal permissions
7. **Monitor Threat Intelligence**: GuardDuty auto-updates with latest threats
8. **Test Incident Response**: Simulate attacks, practice remediation
9. **Tag Resources**: Use tags for cost allocation and security categorization
10. **Review Regularly**: Monthly security posture reviews in Security Hub

---

**END OF DOCUMENT**