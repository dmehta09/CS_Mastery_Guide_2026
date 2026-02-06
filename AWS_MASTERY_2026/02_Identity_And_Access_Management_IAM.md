# AWS Identity & Access Management (IAM) - Complete Conceptual Guide

## Table of Contents
1. [Introduction to IAM](#introduction-to-iam)
2. [Core Concepts](#core-concepts)
   - [IAM Users](#iam-users)
   - [IAM Groups](#iam-groups)
   - [IAM Roles](#iam-roles)
   - [IAM Policies](#iam-policies)
   - [Policy Evaluation Logic](#policy-evaluation-logic)
   - [Principal & Actions](#principal--actions)
   - [Conditions in Policies](#conditions-in-policies)
   - [Permission Boundaries](#permission-boundaries)
   - [Session Policies](#session-policies)
   - [Service-Linked Roles](#service-linked-roles)
   - [Cross-Account Access](#cross-account-access)
   - [AssumeRole](#assumerole)
3. [Advanced IAM](#advanced-iam)
   - [IAM Identity Center (SSO)](#iam-identity-center-sso)
   - [SAML 2.0 Federation](#saml-20-federation)
   - [OIDC Federation](#oidc-federation)
   - [Web Identity Federation](#web-identity-federation)
   - [Attribute-Based Access Control (ABAC)](#attribute-based-access-control-abac)
   - [IAM Access Analyzer](#iam-access-analyzer)
   - [IAM Policy Simulator](#iam-policy-simulator)
   - [Credential Report](#credential-report)
   - [Access Advisor](#access-advisor)
   - [IAM Roles Anywhere](#iam-roles-anywhere)

---

## Introduction to IAM

### What is IAM?

AWS Identity and Access Management (IAM) is a **centralized security service** that controls **who** can access **what** resources in your AWS account, and **how** they can use them. Think of IAM as the security guard and key management system for your entire AWS infrastructure.

### Why Does IAM Exist?

In any cloud environment, security is paramount. Without proper access control:
- Anyone could launch expensive resources and rack up bills
- Unauthorized users could access sensitive data
- Accidental deletions or modifications could cause outages
- Compliance requirements would be impossible to meet

IAM solves these problems by providing:
1. **Authentication** - Verifying who you are (identity verification)
2. **Authorization** - Determining what you're allowed to do (permission management)
3. **Auditing** - Tracking who did what and when (accountability)

### Real-World Problem IAM Solves

Imagine a company with:
- Developers who need to deploy applications
- Data scientists who need read-only access to databases
- Finance team who should only view billing information
- External contractors who need temporary access to specific projects

Without IAM, you'd either:
- Give everyone full access (massive security risk)
- Create separate AWS accounts for each person (management nightmare)
- Manually control access to each resource (impossible at scale)

**IAM provides granular, scalable, centralized control** over all these access scenarios.

### Key Characteristics of IAM

- **Global Service**: IAM is not region-specific; identities and policies work across all AWS regions
- **Eventually Consistent**: Changes to IAM policies propagate globally but may take a few seconds
- **Free**: No additional cost for using IAM itself
- **Integrated**: Works with virtually all AWS services
- **Fine-Grained**: Control access down to specific API actions on specific resources

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Account                              │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │                    IAM Service                          │    │
│  │                                                          │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │    │
│  │  │  Users   │  │  Groups  │  │  Roles   │             │    │
│  │  └─────┬────┘  └─────┬────┘  └─────┬────┘             │    │
│  │        │             │             │                    │    │
│  │        └─────────────┴─────────────┘                    │    │
│  │                      │                                   │    │
│  │                ┌─────▼──────┐                           │    │
│  │                │  Policies  │  (Define Permissions)     │    │
│  │                └─────┬──────┘                           │    │
│  │                      │                                   │    │
│  └──────────────────────┼───────────────────────────────────┘    │
│                         │                                        │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              AWS Resources & Services                     │  │
│  │   (EC2, S3, RDS, Lambda, DynamoDB, etc.)                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

## IAM Users

### What Are IAM Users?

An IAM User is a **permanent identity** that represents a **person or application** that needs to interact with AWS. Each user has:
- A unique name within your AWS account
- Credentials (password and/or access keys)
- Permissions defined by attached policies

### Why IAM Users Exist

**The Root User Problem**: When you create an AWS account, you get a root user with unlimited access. Using the root user for daily tasks is dangerous because:
- No restrictions on what it can do
- If credentials are compromised, entire account is at risk
- Can't audit who did what when multiple people share credentials

**IAM Users solve this** by creating individual identities with specific, limited permissions.

### Real-World Scenarios

**Scenario 1: Development Team**
```
Company has 5 developers who need to:
- Deploy applications to EC2
- Access application logs in CloudWatch
- Cannot delete production databases

Solution: Create 5 IAM users, each with policies that:
  ✓ Allow EC2 deployment
  ✓ Allow CloudWatch read access
  ✗ Deny RDS delete operations
```

**Scenario 2: Application Credentials**
```
A Python application needs to upload files to S3

Solution: Create an IAM user specifically for this app with:
  - No password (not a human)
  - Access keys for programmatic access
  - Policy allowing only S3 PutObject on specific bucket
```

### IAM User Components

```
┌─────────────────────────────────────────────────────────┐
│                    IAM User: "john-developer"            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Identity Information:                                   │
│  • Unique Name: john-developer                           │
│  • ARN: arn:aws:iam::123456789012:user/john-developer   │
│  • User ID: AIDACKCEVSQ6C2EXAMPLE                        │
│                                                          │
│  Credentials (Authentication):                           │
│  ┌──────────────────────────────────────────┐           │
│  │ Console Access:                           │           │
│  │ • Password: ****************              │           │
│  │ • MFA Device: arn:aws:iam::...           │           │
│  └──────────────────────────────────────────┘           │
│  ┌──────────────────────────────────────────┐           │
│  │ Programmatic Access:                      │           │
│  │ • Access Key ID: AKIAIOSFODNN7EXAMPLE     │           │
│  │ • Secret Access Key: (hidden)             │           │
│  └──────────────────────────────────────────┘           │
│                                                          │
│  Permissions (Authorization):                            │
│  ┌──────────────────────────────────────────┐           │
│  │ Attached Policies:                        │           │
│  │ • DeveloperAccessPolicy                   │           │
│  │ • S3ReadOnlyAccess                        │           │
│  └──────────────────────────────────────────┘           │
│  ┌──────────────────────────────────────────┐           │
│  │ Group Memberships:                        │           │
│  │ • Developers                              │           │
│  └──────────────────────────────────────────┘           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Authentication Methods for IAM Users

**1. Console Password**
- For human users accessing AWS Management Console
- Can enforce password policies (complexity, rotation)
- Should always enable MFA (Multi-Factor Authentication)

**2. Access Keys**
- For programmatic access (API calls, CLI, SDKs)
- Consists of Access Key ID (public) and Secret Access Key (private)
- Can have up to 2 active access keys per user (for rotation)
- Never hardcode in application code

### Best Practices for IAM Users

**Do:**
- Create individual users for each person (no sharing)
- Enable MFA for all users, especially those with elevated permissions
- Use groups to assign permissions, not direct user policies
- Rotate access keys regularly
- Apply least privilege principle

**Don't:**
- Use root user for everyday tasks
- Share user credentials between people
- Hardcode access keys in code (use roles instead)
- Create users when roles would work better (e.g., for EC2 applications)

### When to Use IAM Users vs. Roles

| Use IAM Users When | Use IAM Roles When |
|-------------------|-------------------|
| Permanent human identities needed | AWS service needs to access another service |
| Long-term programmatic access required | Temporary access needed |
| Individual accountability is critical | Cross-account access required |
| Third-party tools require static credentials | Application running on EC2/Lambda |

---

## IAM Groups

### What Are IAM Groups?

An IAM Group is a **collection of IAM users**. Groups let you manage permissions for multiple users simultaneously. Think of groups as job titles or departments in a company.

### Why IAM Groups Exist

**The Permission Management Problem**:
Imagine managing permissions for 100 developers individually:
- Attaching the same policy to 100 users one by one
- When permissions change, updating 100 users manually
- Risk of inconsistent permissions across team members

**IAM Groups solve this** by allowing you to:
- Define permissions once for the entire group
- Add/remove users from groups instantly
- Ensure consistent permissions across team members

### Real-World Organization Structure

```
┌──────────────────────────────────────────────────────────────┐
│                  AWS Account: MyCompany                       │
│                                                               │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────┐ │
│  │  Administrators │  │   Developers     │  │  Data Team  │ │
│  │                 │  │                  │  │             │ │
│  │  Users:         │  │  Users:          │  │  Users:     │ │
│  │  • alice-admin  │  │  • john-dev      │  │  • sarah-ds │ │
│  │  • bob-admin    │  │  • jane-dev      │  │  • mike-ds  │ │
│  │                 │  │  • tom-dev       │  │             │ │
│  │  Policies:      │  │                  │  │  Policies:  │ │
│  │  • Admin Access │  │  Policies:       │  │  • S3 Read  │ │
│  │                 │  │  • EC2 Full      │  │  • Athena   │ │
│  └─────────────────┘  │  • RDS Read      │  │    Full     │ │
│                       │  • S3 Limited    │  └─────────────┘ │
│                       └──────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

### How Groups Work

```
Permission Flow Diagram:

┌─────────────────────────────────────────────────────────────┐
│  IAM Group: "Developers"                                     │
│                                                              │
│  Attached Policies:                                          │
│  ┌────────────────────────────────────────────────┐         │
│  │ Policy 1: EC2FullAccess                        │         │
│  │ Policy 2: S3ReadOnlyAccess                     │         │
│  │ Policy 3: CloudWatchLogsReadAccess             │         │
│  └────────────────────────────────────────────────┘         │
│         │                                                    │
│         └─────────┬───────────────┬──────────────┐          │
│                   ▼               ▼              ▼          │
│            ┌───────────┐   ┌───────────┐  ┌───────────┐    │
│            │ john-dev  │   │ jane-dev  │  │  tom-dev  │    │
│            └───────────┘   └───────────┘  └───────────┘    │
│                                                              │
│  All three users inherit:                                   │
│  ✓ Full EC2 permissions                                     │
│  ✓ Read-only S3 permissions                                 │
│  ✓ CloudWatch Logs read permissions                         │
└─────────────────────────────────────────────────────────────┘
```

### Key Characteristics

**1. Users Can Belong to Multiple Groups**
```
Example: Sarah is both a developer and a team lead

sarah-dev ──┬──> "Developers" group (deploy permissions)
            └──> "TeamLeads" group (billing view permissions)

Sarah gets permissions from BOTH groups combined
```

**2. Groups Cannot Be Nested**
```
❌ INVALID:
   AllEngineers
      ├─> BackendDevelopers
      └─> FrontendDevelopers

✓ VALID:
   AllEngineers (with common permissions)
   BackendDevelopers (with backend-specific permissions)
   FrontendDevelopers (with frontend-specific permissions)

   A user joins whichever groups apply to them
```

**3. Groups Are Not Identities**
```
❌ Cannot do: "Allow group Developers to assume role X"
✓ Can do: "Allow users in group Developers to assume role X"

Groups cannot be referenced in resource policies
Groups are just permission containers for users
```

### Practical Example: Managing a Growing Team

**Initial Setup (5 developers)**
```
Manual approach (without groups):
  Attach EC2FullAccess to john-dev
  Attach EC2FullAccess to jane-dev
  Attach EC2FullAccess to tom-dev
  Attach EC2FullAccess to alice-dev
  Attach EC2FullAccess to bob-dev
  = 5 policy attachments

Group approach:
  Create "Developers" group
  Attach EC2FullAccess to "Developers" group
  Add all 5 users to group
  = 1 policy attachment + 5 group memberships
```

**Scaling (50 developers)**
```
Manual approach:
  Modify permissions = Update 50 individual users
  New requirement = Attach new policy 50 times
  Remove access = Detach from 50 users

Group approach:
  Modify permissions = Update 1 group policy
  New requirement = Attach policy to 1 group once
  Remove access = Remove user from 1 group
```

**Permission Change Example**
```
Requirement: Developers now need CloudWatch access

Without groups:
  1. Find all developer users
  2. Attach CloudWatchReadOnlyAccess to user 1
  3. Attach CloudWatchReadOnlyAccess to user 2
  4. ... (repeat 50 times)
  5. Risk: Missing someone or inconsistent permissions

With groups:
  1. Attach CloudWatchReadOnlyAccess to "Developers" group
  2. Done - all members instantly have access
```

### Best Practices for IAM Groups

**Organize by Job Function**
```
Good group structure:
├─ Administrators (full account access)
├─ Developers (deploy and debug)
├─ DataScientists (data analysis tools)
├─ Auditors (read-only access)
├─ Billing (cost management)
└─ Support (limited troubleshooting)
```

**Combine Groups for Complex Permissions**
```
Example: Senior Backend Developer

User: john-senior-backend
Groups:
  • Developers (basic dev tools)
  • BackendTeam (database access)
  • Seniors (production access)

Total permissions = Union of all three groups
```

**Use Descriptive Naming**
```
❌ Bad: group1, team-a, engineers
✓ Good: EC2-Developers, S3-Admins, ReadOnly-Auditors
```

---

## IAM Roles

### What Are IAM Roles?

An IAM Role is a **temporary identity** that can be **assumed** by trusted entities to gain specific permissions. Unlike users, roles don't have permanent credentials. Think of a role as a "hat" that someone or something can wear temporarily to gain certain abilities.

### Why IAM Roles Exist

**The Credential Management Problem**:
```
Without Roles:
  EC2 instance needs S3 access
  ❌ Create IAM user for the instance
  ❌ Store access keys on the instance
  Problems:
    • Keys can be stolen if instance is compromised
    • Keys need manual rotation
    • Hard to manage at scale
    • Keys might be hardcoded or logged
```

**Roles solve this** by providing:
- Temporary security credentials (auto-rotated)
- No long-term keys to manage
- Automatic credential refresh
- Fine-grained access control

### Core Difference: Users vs. Roles

```
┌──────────────────────────────────────────────────────────┐
│                    IAM User                               │
├──────────────────────────────────────────────────────────┤
│ • ONE specific person or application                      │
│ • Permanent credentials (password/access keys)            │
│ • Long-term identity                                      │
│ • Credentials stored securely by user                     │
│                                                           │
│ Use when: Specific individual needs permanent access      │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                    IAM Role                               │
├──────────────────────────────────────────────────────────┤
│ • ANYONE/ANYTHING that's trusted can use it               │
│ • Temporary credentials (auto-generated)                  │
│ • Assumed when needed, released when done                 │
│ • AWS manages credential lifecycle                        │
│                                                           │
│ Use when: Temporary or delegated access needed            │
└──────────────────────────────────────────────────────────┘
```

### Role Components

Every IAM role has two critical policies:

```
┌─────────────────────────────────────────────────────────────┐
│              IAM Role: "EC2-S3-Access-Role"                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Trust Policy (Who can assume this role?)                │
│  ┌────────────────────────────────────────────────────┐    │
│  │ {                                                   │    │
│  │   "Effect": "Allow",                                │    │
│  │   "Principal": {                                    │    │
│  │     "Service": "ec2.amazonaws.com"  ◄─ Only EC2     │    │
│  │   },                                    can assume  │    │
│  │   "Action": "sts:AssumeRole"            this role   │    │
│  │ }                                                   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  2. Permission Policy (What can they do after assuming?)    │
│  ┌────────────────────────────────────────────────────┐    │
│  │ {                                                   │    │
│  │   "Effect": "Allow",                                │    │
│  │   "Action": "s3:GetObject",         ◄─ Can read     │    │
│  │   "Resource": "arn:aws:s3:::my-bucket/*"  from S3   │    │
│  │ }                                                   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Real-World Use Cases

**Use Case 1: EC2 Instance Accessing S3**

```
Problem: Web application on EC2 needs to read files from S3

Bad Solution (❌):
┌──────────────┐
│ EC2 Instance │
│              │  Has hardcoded access keys
│  app.py:     │  Access Key: AKIAI...
│  AWS_KEY=... │  Secret Key: wJalr...
└──────────────┘
Problems: Keys can leak, need rotation, security risk

Good Solution (✓):
┌──────────────────────────────────────────────────────────┐
│  1. Create Role "WebApp-S3-Reader"                        │
│     Trust: ec2.amazonaws.com                              │
│     Permissions: S3 read access                           │
│                                                           │
│  2. Attach role to EC2 instance                           │
│                                                           │
│  3. Application automatically gets temporary credentials  │
│     (refreshed every hour by AWS)                         │
└──────────────────────────────────────────────────────────┘

┌──────────────┐           ┌─────────────────┐
│ EC2 Instance │──assumes──>│ WebApp-S3-Reader│
│              │<──gets────│  (Role)         │
│  No keys     │  temp      └─────────────────┘
│  stored!     │  creds            │
└──────────────┘                   │ allows
                                   ▼
                            ┌─────────────┐
                            │ S3 Bucket   │
                            └─────────────┘
```

**Use Case 2: Lambda Function Accessing DynamoDB**

```
Scenario: Serverless function needs to write to database

┌─────────────────┐         ┌──────────────────────┐
│ Lambda Function │─assumes─>│ Lambda-DynamoDB-Role │
│                 │         └──────────────────────┘
│  No credentials │                    │
│  in code        │                    │ grants
└─────────────────┘                    ▼
                              ┌─────────────────┐
                              │ DynamoDB Table  │
                              └─────────────────┘

Trust Policy: lambda.amazonaws.com can assume
Permission Policy: dynamodb:PutItem allowed
```

**Use Case 3: Cross-Account Access**

```
Scenario: Dev account needs to access Production S3 bucket

Account A (Production - 111111111111)
┌────────────────────────────────────┐
│  S3 Bucket: prod-data              │
│                                    │
│  Role: CrossAccountS3Role          │
│  Trust: Account B (222222222222)   │
│  Permissions: S3 read access       │
└────────────────────────────────────┘
                ▲
                │ assumes
                │
Account B (Development - 222222222222)
┌────────────────────────────────────┐
│  IAM User: alice-developer         │
│                                    │
│  Permission: Can assume            │
│  CrossAccountS3Role in Account A   │
└────────────────────────────────────┘
```

**Use Case 4: Federated Users (Corporate Login)**

```
Scenario: Employees use company login to access AWS

┌────────────────────┐
│  Corporate IdP     │ (Identity Provider)
│  (Okta, Azure AD)  │
│                    │
│  Employee: john@company.com
└─────────┬──────────┘
          │ authenticates
          ▼
┌─────────────────────┐
│  AWS STS            │ (Security Token Service)
│  (Token Generator)  │
└─────────┬───────────┘
          │ provides temp credentials
          ▼
┌─────────────────────┐
│  IAM Role           │
│  "EngineerAccess"   │
│                     │
│  Trust: SAML IdP    │
│  Permissions: Dev   │
│  environment access │
└─────────────────────┘
```

### Types of IAM Roles

**1. Service Roles**
```
Purpose: Let AWS services act on your behalf
Examples:
  • Lambda execution role (for Lambda to write logs)
  • EC2 instance role (for EC2 to access S3)
  • ECS task role (for containers to access AWS resources)

Trust Policy Principal: AWS service (e.g., "lambda.amazonaws.com")
```

**2. Cross-Account Roles**
```
Purpose: Share resources between AWS accounts securely
Examples:
  • Production account sharing data with dev account
  • Central logging account collecting logs from all accounts
  • Third-party vendor needing limited access

Trust Policy Principal: AWS account ID or organization
```

**3. Federated User Roles**
```
Purpose: Let external users access AWS without IAM users
Examples:
  • Corporate employees using SSO
  • Mobile app users (Facebook/Google login)
  • SAML 2.0 enterprise authentication

Trust Policy Principal: SAML provider or Web Identity provider
```

**4. Instance Profile Role**
```
Purpose: Special wrapper for EC2 roles
Note: When you create a role for EC2, AWS creates both:
  • The IAM role (logical permissions entity)
  • An instance profile (container for the role)

The instance profile is what you actually attach to EC2
```

### How Role Assumption Works (Technical Flow)

```
Step-by-Step: EC2 Instance Assuming a Role

1. Initial Setup:
   ┌──────────────┐
   │ EC2 Instance │  Has instance profile attached
   └──────┬───────┘
          │
          │ Instance Profile points to
          ▼
   ┌─────────────────┐
   │ IAM Role        │
   │ "MyEC2Role"     │
   └─────────────────┘

2. Application Makes AWS API Call:
   ┌──────────────┐
   │ EC2 Instance │
   │  Running App │
   │              │  app.py: s3.get_object(...)
   │              │  (No keys in code!)
   └──────┬───────┘
          │
          │ SDK checks: Where are credentials?
          ▼
   ┌────────────────────┐
   │ EC2 Metadata       │  http://169.254.169.254/latest/meta-data/
   │ Endpoint           │  iam/security-credentials/MyEC2Role
   └────────┬───────────┘
          │ returns
          ▼
   Temporary Credentials:
   {
     "AccessKeyId": "ASIA...",
     "SecretAccessKey": "...",
     "Token": "...",
     "Expiration": "2026-02-06T15:30:00Z"  ← Valid for 1 hour
   }

3. SDK Uses Temp Credentials:
   ┌──────────────┐
   │ AWS SDK      │  Uses temporary credentials automatically
   └──────┬───────┘
          │
          │ Makes API call with temp creds
          ▼
   ┌─────────────────┐
   │ S3 Service      │  Validates credentials & checks permissions
   └─────────────────┘

4. Automatic Refresh:
   ✓ Credentials expire after 1 hour
   ✓ SDK automatically requests new ones (5-15 min before expiration)
   ✓ No manual rotation needed
```

### Role Session Duration

```
Session Duration Settings:

Default: 1 hour
Maximum: 12 hours (configurable per role)
Minimum: 15 minutes (for AssumeRole API)

┌────────────────────────────────────────────────────┐
│  Role Assumption Timeline                          │
├────────────────────────────────────────────────────┤
│                                                    │
│  0 min    AssumeRole called                        │
│  │        Temporary credentials issued             │
│  │                                                  │
│  │        ┌─────────────────────────┐              │
│  │        │  Credentials Valid      │              │
│  │        │  Application Works      │              │
│  │        └─────────────────────────┘              │
│  │                                                  │
│  45 min   SDK starts refreshing (proactive)        │
│  │        New credentials requested in background  │
│  │                                                  │
│  60 min   Old credentials expire                   │
│  │        New credentials already in use           │
│  │        ✓ No interruption to application         │
│  │                                                  │
└────────────────────────────────────────────────────┘
```

### When to Use Roles vs. Users

| Scenario | Use This |
|----------|----------|
| EC2 instance needs AWS access | ✓ Role (instance profile) |
| Lambda function needs permissions | ✓ Role (execution role) |
| Cross-account access | ✓ Role (cross-account role) |
| Human accessing AWS Console | User (with MFA) |
| Long-running script on your laptop | User (with access keys) |
| Application on non-AWS server | User or Role (see IAM Roles Anywhere) |
| Mobile app users | ✓ Role (Web Identity Federation) |
| Corporate SSO users | ✓ Role (SAML/OIDC federation) |
| Third-party vendor needs access | ✓ Role (with external ID) |

### Security Features of Roles

**1. Automatic Credential Rotation**
```
✓ New credentials every hour
✓ Old credentials become invalid
✓ No manual key rotation needed
✓ Reduced risk of credential theft
```

**2. No Long-Term Secrets**
```
✓ No access keys stored anywhere
✓ Can't accidentally commit to Git
✓ Can't be stolen from configuration files
✓ Even if stolen, expires within hours
```

**3. Auditability**
```
CloudTrail logs show:
  • Who assumed the role
  • When they assumed it
  • What actions they performed
  • Source IP address
  • Session duration
```

**4. Session Tagging** (Advanced)
```
When assuming role, can pass session tags:
  Department=Engineering
  Project=WebApp

Policies can then require:
  "Condition": {
    "StringEquals": {
      "aws:PrincipalTag/Department": "Engineering"
    }
  }
```

### Common Role Patterns

**Pattern 1: Service-to-Service Communication**
```
┌──────────┐         ┌──────────┐         ┌──────────┐
│ Lambda   │─role────>│ SQS      │─role────>│ DynamoDB │
│ Function │  allows  │ Queue    │  allows  │ Table    │
└──────────┘  send    └──────────┘  read    └──────────┘

Each service has its own role with least-privilege permissions
```

**Pattern 2: Break-Glass Admin Access**
```
Normal Operations:
  Developers ──> Developer Role (limited permissions)

Emergency Only:
  Senior Eng ──> Emergency Admin Role (MFA required)
                 ↓
                 Logs alert sent
                 ↓
                 Full access (audited)
```

**Pattern 3: Temporary Elevated Access**
```
User: john-developer
Normal Role: ReadOnly (default)

When needed:
  1. Request elevated access
  2. Manager approves
  3. Assume "Developer-Elevated" role
  4. Role expires after 2 hours
  5. Automatically reverts to ReadOnly
```

---

## IAM Policies

### What Are IAM Policies?

IAM Policies are **JSON documents** that define **permissions**. They specify:
- **Who** can do something (implicit - based on where policy is attached)
- **What** actions they can perform
- **Which** resources they can access
- **When** and **under what conditions** access is allowed

Think of policies as rule books that say "You can read files from this bucket, but only if you're connecting from the office network."

### Why IAM Policies Exist

Without policies, you'd have binary access control: either full access or no access. Policies solve this by providing:
- **Fine-grained control**: Specify exact actions allowed
- **Conditional access**: Add context-based restrictions
- **Reusability**: One policy can apply to many users/roles
- **Centralized management**: Update permissions in one place

### Policy Types Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    IAM Policy Types                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Identity-Based Policies                                   │
│     ┌────────────────────────────────────────────┐           │
│     │ Attached to: Users, Groups, Roles          │           │
│     │ Controls: What the identity can do         │           │
│     │ Types:                                     │           │
│     │   • AWS Managed (created by AWS)           │           │
│     │   • Customer Managed (you create)          │           │
│     │   • Inline (embedded in identity)          │           │
│     └────────────────────────────────────────────┘           │
│                                                               │
│  2. Resource-Based Policies                                   │
│     ┌────────────────────────────────────────────┐           │
│     │ Attached to: Resources (S3, SQS, etc.)     │           │
│     │ Controls: Who can access the resource      │           │
│     │ Example: S3 bucket policy                  │           │
│     └────────────────────────────────────────────┘           │
│                                                               │
│  3. Permission Boundaries                                     │
│     ┌────────────────────────────────────────────┐           │
│     │ Attached to: Users or Roles                │           │
│     │ Controls: Maximum permissions allowed      │           │
│     │ Use: Delegated admin control               │           │
│     └────────────────────────────────────────────┘           │
│                                                               │
│  4. Service Control Policies (SCPs)                           │
│     ┌────────────────────────────────────────────┐           │
│     │ Attached to: AWS Organizations accounts    │           │
│     │ Controls: Account-wide permission limits   │           │
│     │ Use: Multi-account governance              │           │
│     └────────────────────────────────────────────┘           │
│                                                               │
│  5. Session Policies                                          │
│     ┌────────────────────────────────────────────┐           │
│     │ Passed during: AssumeRole operation        │           │
│     │ Controls: Further limit role permissions   │           │
│     │ Use: Temporary permission reduction        │           │
│     └────────────────────────────────────────────┘           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Policy Structure (Anatomy)

```json
{
  "Version": "2012-10-17",              ← Policy language version (always this)
  "Statement": [                         ← Array of permission statements
    {
      "Sid": "AllowS3Read",              ← Statement ID (optional, for reference)
      "Effect": "Allow",                 ← "Allow" or "Deny"
      "Action": [                        ← What operations are allowed
        "s3:GetObject",                  ← Specific action
        "s3:ListBucket"                  ← Another action
      ],
      "Resource": [                      ← Which resources this applies to
        "arn:aws:s3:::my-bucket/*",      ← Objects in bucket
        "arn:aws:s3:::my-bucket"         ← Bucket itself
      ],
      "Condition": {                     ← When this applies (optional)
        "IpAddress": {                   ← Condition type
          "aws:SourceIp": "203.0.113.0/24"  ← Condition value
        }
      }
    }
  ]
}
```

### Identity-Based Policies

These are attached to IAM users, groups, or roles.

**AWS Managed Policies**
```
Pre-built by AWS, ready to use

Examples:
┌─────────────────────────────────────────────────────┐
│ Policy Name: ReadOnlyAccess                         │
│ Description: Provides read-only access to all       │
│              AWS services and resources             │
│ Use Case: Auditors, viewers                         │
│ ARN: arn:aws:iam::aws:policy/ReadOnlyAccess         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Policy Name: PowerUserAccess                        │
│ Description: Full access except IAM management      │
│ Use Case: Developers who shouldn't manage users     │
│ ARN: arn:aws:iam::aws:policy/PowerUserAccess        │
└─────────────────────────────────────────────────────┘

Pros:
  ✓ Maintained by AWS (updated automatically)
  ✓ Best practices built-in
  ✓ Quick to implement

Cons:
  ✗ Can't modify (too broad or too narrow)
  ✗ May grant more than needed
```

**Customer Managed Policies**
```
Created and managed by you

Example: Custom developer policy

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:InstanceType": ["t2.micro", "t2.small"]
        }
      }
    }
  ]
}

This allows:
  ✓ Launching EC2 instances
  ✓ Terminating EC2 instances
  ✗ Only t2.micro and t2.small types
  ✗ No other instance types

Pros:
  ✓ Exact permissions you need
  ✓ Reusable across users/groups/roles
  ✓ Version-controlled

Cons:
  ✗ More maintenance required
  ✗ Need deep AWS knowledge
```

**Inline Policies**
```
Embedded directly in a specific user, group, or role

When to use:
  • Strict 1:1 relationship needed
  • Policy should be deleted with identity
  • Testing/temporary permissions

Example Use Case:
  Emergency access policy for specific incident
  Attached directly to incident-responder role
  Deleted when role is removed

Warning:
  ⚠ Hard to track at scale
  ⚠ Can't reuse
  ⚠ Prefer customer managed policies
```

### Resource-Based Policies

Attached directly to resources (not identities).

**S3 Bucket Policy Example**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountARead",
      "Effect": "Allow",
      "Principal": {                    ← WHO can access (key difference!)
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Sid": "DenyNonSSL",
      "Effect": "Deny",
      "Principal": "*",                 ← Everyone
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"  ← Except when using HTTPS
        }
      }
    }
  ]
}
```

**Key Difference: Principal**

```
Identity-Based Policy:
  Attached to user "alice"
  Says: "Alice can read S3"
  No "Principal" field (implicit - it's Alice)

Resource-Based Policy:
  Attached to S3 bucket "my-bucket"
  Says: "Alice can access this bucket"
  Has "Principal" field (explicit - specifies Alice)
```

**Real-World Example: Cross-Account S3 Access**

```
Scenario: Account A wants to share bucket with Account B

Account A (111111111111) - Bucket Owner
┌────────────────────────────────────────┐
│  S3 Bucket: shared-data                │
│                                        │
│  Bucket Policy:                        │
│  {                                     │
│    "Effect": "Allow",                  │
│    "Principal": {                      │
│      "AWS": "arn:aws:iam::222222222222:role/DataAnalyst"
│    },                                  │
│    "Action": "s3:GetObject",           │
│    "Resource": "arn:aws:s3:::shared-data/*"
│  }                                     │
└────────────────────────────────────────┘
                    ▲
                    │ can access
                    │
Account B (222222222222) - Consumer
┌────────────────────────────────────────┐
│  IAM Role: DataAnalyst                 │
│                                        │
│  Identity Policy:                      │
│  {                                     │
│    "Effect": "Allow",                  │
│    "Action": "s3:GetObject",           │
│    "Resource": "arn:aws:s3:::shared-data/*"
│  }                                     │
└────────────────────────────────────────┘

Both policies must allow for access to work!
```

### AWS Managed vs. Customer Managed Policies

| Aspect | AWS Managed | Customer Managed |
|--------|-------------|------------------|
| Created by | Amazon Web Services | You |
| Maintainence | AWS updates automatically | You update manually |
| Versioning | AWS maintains versions | You control versions |
| Reusability | Across AWS accounts | Within your account |
| Specificity | General purpose | Tailored to needs |
| Best for | Common use cases | Custom requirements |

**Example Comparison**

```
AWS Managed: AmazonS3ReadOnlyAccess
  Allows:
    ✓ s3:Get*
    ✓ s3:List*
    ✓ s3:Describe*
    ✓ All S3 buckets

  Too broad if you only want one bucket

Customer Managed: MyApp-S3-ReadOnly
  Allows:
    ✓ s3:GetObject
    ✓ s3:ListBucket
    ✗ Only "arn:aws:s3:::my-app-bucket/*"

  Exact permissions needed
```

---

## Policy Evaluation Logic

### What is Policy Evaluation?

Policy evaluation is the **decision-making process** AWS uses to determine whether a request should be **allowed** or **denied**. When you make an AWS API call, AWS evaluates all applicable policies to make this decision.

### Why Understanding Evaluation Logic Matters

```
Scenario: Developer can't access S3 bucket

Without understanding evaluation:
  ❓ Check if they have S3 permissions (they do)
  ❓ Still denied - why?
  😕 Confused, frustrated

With understanding evaluation:
  ✓ Check identity policy (allows S3)
  ✓ Check resource policy (not present)
  ✓ Check permission boundary (restricts S3!)
  ✓ Found the issue!
```

### The Policy Evaluation Flow

```
Request: IAM user "alice" wants to s3:GetObject from "my-bucket/file.txt"

┌─────────────────────────────────────────────────────────────────┐
│                    STEP 1: AUTHENTICATION                        │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Is the requester who they claim to be?               │      │
│  │ ✓ Valid credentials (access keys, password, etc.)     │      │
│  └───────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STEP 2: CONTEXT GATHERING                     │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Collect all relevant policies:                        │      │
│  │ • Identity-based policies                             │      │
│  │ • Resource-based policies                             │      │
│  │ • Permission boundaries                               │      │
│  │ • Service control policies (SCPs)                     │      │
│  │ • Session policies (if assuming role)                 │      │
│  └───────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STEP 3: EXPLICIT DENY CHECK                   │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Is there an explicit "Deny" anywhere?                 │      │
│  │                                                        │      │
│  │ If YES → ❌ REQUEST DENIED (game over)                │      │
│  │ If NO  → Continue to next step                        │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                  │
│  ⚠️  DENY ALWAYS WINS - Even one deny overrides all allows     │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STEP 4: EXPLICIT ALLOW CHECK                  │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Is there an explicit "Allow"?                         │      │
│  │                                                        │      │
│  │ Consider:                                             │      │
│  │ • Identity policy allows?                             │      │
│  │ • Resource policy allows?                             │      │
│  │ • Either one sufficient (within same account)         │      │
│  │                                                        │      │
│  │ If YES → ✅ REQUEST ALLOWED                           │      │
│  │ If NO  → ❌ REQUEST DENIED (implicit deny)            │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                  │
│  📝 Default is deny - must have explicit allow                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Golden Rule: Deny > Allow

```
POLICY EVALUATION HIERARCHY:

1. Explicit DENY          ← Highest Priority (Overrides everything)
2. Explicit ALLOW         ← Only checked if no denies
3. Implicit DENY (default) ← If no explicit allow found

Example:
┌────────────────────────────────────────┐
│ Policy A: Allow s3:*                   │
│ Policy B: Deny s3:DeleteBucket         │
│ Policy C: Allow s3:DeleteBucket        │
└────────────────────────────────────────┘

Result: ❌ DENIED
Reason: Policy B's Deny overrides Policy A and C's Allow
```

### Detailed Evaluation Examples

**Example 1: Simple Allow**

```
User: john-developer
Request: s3:GetObject on my-bucket/file.txt

Policies to evaluate:
┌──────────────────────────────────────────┐
│ Identity Policy (attached to john)       │
│ {                                        │
│   "Effect": "Allow",                     │
│   "Action": "s3:GetObject",              │
│   "Resource": "arn:aws:s3:::my-bucket/*" │
│ }                                        │
└──────────────────────────────────────────┘

Evaluation:
  ✓ Authenticated as john-developer
  ✓ No explicit deny found
  ✓ Explicit allow found (identity policy)
  → ✅ ALLOWED
```

**Example 2: Resource Policy Allow**

```
User: alice (from external account 222222222222)
Request: s3:GetObject on shared-bucket/data.txt

Bucket Owner Account: 111111111111

Policies to evaluate:
┌──────────────────────────────────────────┐
│ Alice's Identity Policy (Account 222222) │
│ {                                        │
│   "Effect": "Allow",                     │
│   "Action": "s3:GetObject",              │
│   "Resource": "arn:aws:s3:::shared-bucket/*"
│ }                                        │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│ Bucket Policy (Account 111111)           │
│ {                                        │
│   "Effect": "Allow",                     │
│   "Principal": {                         │
│     "AWS": "arn:aws:iam::222222:user/alice"
│   },                                     │
│   "Action": "s3:GetObject",              │
│   "Resource": "arn:aws:s3:::shared-bucket/*"
│ }                                        │
└──────────────────────────────────────────┘

Evaluation:
  ✓ Authenticated as alice
  ✓ No explicit deny
  ✓ Identity policy allows (account 222222)
  ✓ Resource policy allows (account 111111)
  → ✅ ALLOWED

Note: Cross-account requires BOTH identity and resource policies to allow
```

**Example 3: Explicit Deny Wins**

```
User: bob-admin
Request: s3:DeleteBucket on production-data

Policies to evaluate:
┌──────────────────────────────────────────┐
│ Policy 1: AdministratorAccess            │
│ {                                        │
│   "Effect": "Allow",                     │
│   "Action": "*",                         │
│   "Resource": "*"                        │
│ }                                        │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│ Policy 2: DenyProductionDelete           │
│ {                                        │
│   "Effect": "Deny",                      │
│   "Action": "s3:DeleteBucket",           │
│   "Resource": "arn:aws:s3:::production-*"│
│ }                                        │
└──────────────────────────────────────────┘

Evaluation:
  ✓ Authenticated as bob-admin
  ❌ EXPLICIT DENY FOUND (Policy 2)
  → ❌ DENIED

Even though bob has Administrator access (allows everything),
the explicit deny prevents deletion of production buckets.
```

**Example 4: Permission Boundary Restriction**

```
User: contractor-jane
Request: ec2:TerminateInstances

Policies to evaluate:
┌──────────────────────────────────────────┐
│ Identity Policy (allows everything)      │
│ {                                        │
│   "Effect": "Allow",                     │
│   "Action": "*",                         │
│   "Resource": "*"                        │
│ }                                        │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│ Permission Boundary (limits to S3 only)  │
│ {                                        │
│   "Effect": "Allow",                     │
│   "Action": "s3:*",                      │
│   "Resource": "*"                        │
│ }                                        │
└──────────────────────────────────────────┘

Evaluation:
  ✓ Authenticated as contractor-jane
  ✓ No explicit deny
  ✓ Identity policy allows ec2:TerminateInstances
  ❌ Permission boundary does NOT allow ec2:*
  → ❌ DENIED

Effective permissions = Identity Policy ∩ Permission Boundary
Intersection includes only S3 actions, not EC2
```

### Policy Evaluation Within vs. Across Accounts

**Same Account**

```
┌─────────────────────────────────────────────────────────┐
│            Account: 111111111111                         │
│                                                          │
│  User: alice                                             │
│  Identity Policy: Allow s3:GetObject                     │
│           OR                                             │
│  Resource Policy: Allow s3:GetObject                     │
│                                                          │
│  Result: ALLOWED                                         │
│  (Either policy alone is sufficient)                     │
└─────────────────────────────────────────────────────────┘
```

**Cross Account**

```
Account A: 111111111111                Account B: 222222222222
┌─────────────────────────┐           ┌──────────────────────┐
│ S3 Bucket: shared-data  │           │ User: bob            │
│                         │           │                      │
│ Bucket Policy:          │           │ Identity Policy:     │
│ Allow bob from          │           │ Allow s3:GetObject   │
│ Account B               │           │ on shared-data       │
└─────────────────────────┘           └──────────────────────┘
             │                                    │
             └────────────────┬───────────────────┘
                              │
                    BOTH must allow
                              │
                              ▼
                        ✅ ALLOWED

If either is missing → ❌ DENIED
```

### Visual Decision Tree

```
                     ┌─────────────────┐
                     │  Request Made   │
                     └────────┬────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │ Authenticated?       │
                   └──────┬───────────────┘
                          │
                 ┌────────┴────────┐
                 │                 │
              NO │                 │ YES
                 ▼                 ▼
         ┌──────────┐      ┌──────────────┐
         │ ❌ DENY  │      │ Any Explicit │
         │ (AuthN)  │      │    Deny?     │
         └──────────┘      └──────┬───────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                      YES│                 │NO
                         ▼                 ▼
                  ┌──────────┐     ┌────────────────┐
                  │ ❌ DENY  │     │ Any Explicit   │
                  │  (Deny)  │     │    Allow?      │
                  └──────────┘     └────────┬───────┘
                                            │
                                   ┌────────┴────────┐
                                   │                 │
                                YES│                 │NO
                                   ▼                 ▼
                          ┌────────────┐     ┌──────────┐
                          │ ✅ ALLOW   │     │ ❌ DENY  │
                          │            │     │(Implicit)│
                          └────────────┘     └──────────┘
```

### Common Evaluation Scenarios

**Scenario 1: Conflicting Policies**

```
Question: What if different policies allow different actions?

Answer: Union (combine) all allows, unless there's a deny

Example:
  Policy A: Allow s3:GetObject
  Policy B: Allow s3:PutObject
  Policy C: Allow s3:DeleteObject

  Result: User can Get, Put, and Delete objects
```

**Scenario 2: Wildcards in Deny**

```
Policy:
{
  "Effect": "Deny",
  "Action": "ec2:*",
  "Resource": "*"
}

This denies ALL EC2 actions, including:
  ❌ ec2:RunInstances
  ❌ ec2:TerminateInstances
  ❌ ec2:DescribeInstances
  ❌ Every single EC2 action

Cannot be overridden by any allow policy
```

**Scenario 3: Missing Resource Policy**

```
Same account access to S3:

Identity Policy: ✓ Allows
Resource Policy: (none)

Result: ✅ ALLOWED (resource policy not required within account)

Cross-account access:

Identity Policy: ✓ Allows
Resource Policy: (none)

Result: ❌ DENIED (resource policy required for cross-account)
```

### Debugging Permission Issues

**Step-by-Step Troubleshooting**

```
Issue: User can't perform action

1. Verify Authentication
   Q: Can they log in / Are credentials valid?
   Tool: Check sign-in logs

2. Check for Explicit Denies
   Q: Is there any deny statement?
   Look at:
     • Identity policies
     • Resource policies
     • Permission boundaries
     • SCPs (if using AWS Organizations)

3. Check for Explicit Allows
   Q: Is there an allow statement?
   Look at:
     • All attached identity policies
     • Resource policy (if cross-account or specific resources)
     • Inherited group policies

4. Check Permission Boundaries
   Q: Is there a permission boundary limiting them?
   Note: Boundary must also allow the action

5. Check SCPs (Organizations)
   Q: Does the account SCP allow this action?
   Note: SCPs can restrict entire accounts

6. Check Conditions
   Q: Are there conditions that aren't being met?
   Examples:
     • IP address restrictions
     • MFA requirements
     • Time-based restrictions
     • Resource tags
```

**Using IAM Policy Simulator (covered in detail later)**

```
Tool: IAM Policy Simulator (console or CLI)

What it does:
  • Simulates policy evaluation
  • Shows which policies apply
  • Identifies why request was denied/allowed
  • Tests without making actual requests

Usage:
  1. Select user/role
  2. Choose service and action
  3. Optionally specify resource and conditions
  4. Run simulation
  5. See detailed results
```

### Best Practices for Policy Evaluation

```
1. Principle of Least Privilege
   ✓ Start with minimum permissions
   ✓ Add more only when needed
   ✓ Use explicit denies for critical protections

2. Use Permission Boundaries
   ✓ Delegate user creation safely
   ✓ Ensure contractors can't escalate privileges

3. Test Policies Before Deployment
   ✓ Use Policy Simulator
   ✓ Test in non-production first
   ✓ Review CloudTrail logs

4. Document Deny Statements
   ✓ Denies can be confusing
   ✓ Add comments (Sid field) explaining why

5. Regular Audits
   ✓ Review unused permissions (Access Advisor)
   ✓ Remove excessive permissions
   ✓ Check for orphaned policies
```

---

## Principal & Actions

### What Are Principals?

A **Principal** is an entity that can make requests to AWS. It represents the **"who"** in access control. Principals can authenticate and are assigned permissions.

### Types of Principals

```
┌──────────────────────────────────────────────────────────────┐
│                    Principal Types                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. AWS Account                                               │
│     arn:aws:iam::123456789012:root                           │
│     Represents the entire AWS account                         │
│                                                               │
│  2. IAM User                                                  │
│     arn:aws:iam::123456789012:user/john-developer            │
│     Specific individual or application identity               │
│                                                               │
│  3. IAM Role                                                  │
│     arn:aws:iam::123456789012:role/MyEC2Role                 │
│     Assumable identity                                        │
│                                                               │
│  4. Federated User (via SAML)                                │
│     arn:aws:sts::123456789012:federated-user/alice           │
│     External identity provider user                           │
│                                                               │
│  5. Federated User (via Web Identity)                        │
│     Authenticated via Google, Facebook, Amazon, etc.          │
│                                                               │
│  6. AWS Service                                               │
│     ec2.amazonaws.com                                        │
│     lambda.amazonaws.com                                     │
│     Service acting on your behalf                            │
│                                                               │
│  7. Anonymous Users                                           │
│     "*"  (wildcard - everyone, including unauthenticated)    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Principal in Different Policy Types

**Identity-Based Policies (No Principal Field)**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
    // NO "Principal" - it's implied (whoever this policy is attached to)
  }]
}

Attached to: IAM user "alice"
Meaning: Alice can perform s3:GetObject
```

**Resource-Based Policies (Principal Required)**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {                              ← Explicitly WHO
      "AWS": "arn:aws:iam::123456789012:user/alice"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}

Attached to: S3 bucket "my-bucket"
Meaning: Alice (from account 123456789012) can read from this bucket
```

### Principal Formats and Examples

**Specific IAM User**

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:user/alice"
  }
}

Allows: Only user alice from account 123456789012
```

**Entire AWS Account**

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:root"
  }
}

Allows: Any IAM identity in account 123456789012
Note: "root" here means the account, not the root user
```

**Multiple Principals**

```json
{
  "Principal": {
    "AWS": [
      "arn:aws:iam::111111111111:user/alice",
      "arn:aws:iam::222222222222:user/bob",
      "arn:aws:iam::333333333333:root"
    ]
  }
}

Allows:
  • User alice from account 111111111111
  • User bob from account 222222222222
  • Anyone from account 333333333333
```

**AWS Service as Principal**

```json
{
  "Principal": {
    "Service": "lambda.amazonaws.com"
  }
}

Allows: AWS Lambda service to assume this role
Common in role trust policies
```

**Federated User (SAML)**

```json
{
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:saml-provider/ExampleCorpIdP"
  }
}

Allows: Users authenticated via corporate SAML identity provider
```

**Everyone (Public Access)**

```json
{
  "Principal": "*"
}

Allows: Anyone (including anonymous users)
⚠️ WARNING: Use very carefully - public access!
```

**Specific Service in Specific Account**

```json
{
  "Principal": {
    "Service": "ec2.amazonaws.com"
  },
  "Condition": {
    "StringEquals": {
      "aws:SourceAccount": "123456789012"
    }
  }
}

Allows: EC2 service, but only when acting on behalf of account 123456789012
Prevents confused deputy attacks
```

### What Are Actions?

**Actions** are the specific operations that can be performed on AWS resources. They represent the **"what"** in access control.

### Action Format

```
Service:Operation

Examples:
  s3:GetObject          ← S3 service, GetObject operation
  ec2:RunInstances      ← EC2 service, RunInstances operation
  iam:CreateUser        ← IAM service, CreateUser operation
  dynamodb:PutItem      ← DynamoDB service, PutItem operation
```

### Action Categories

**Read Actions** (data retrieval, viewing)
```
s3:GetObject
s3:ListBucket
ec2:DescribeInstances
rds:DescribeDBInstances
cloudwatch:GetMetricStatistics
```

**Write Actions** (data modification, creation)
```
s3:PutObject
s3:DeleteObject
ec2:RunInstances
ec2:TerminateInstances
dynamodb:PutItem
dynamodb:DeleteItem
```

**Permission Actions** (access control)
```
iam:CreateUser
iam:AttachUserPolicy
s3:PutBucketPolicy
lambda:AddPermission
```

**Tagging Actions** (metadata management)
```
ec2:CreateTags
s3:PutBucketTagging
rds:AddTagsToResource
```

### Using Wildcards in Actions

**Wildcard Patterns**

```json
{
  "Effect": "Allow",
  "Action": "s3:*"              ← All S3 actions
}

Grants: Every S3 operation
  • s3:GetObject
  • s3:PutObject
  • s3:DeleteBucket
  • s3:CreateBucket
  • etc. (100+ actions)
```

```json
{
  "Effect": "Allow",
  "Action": "s3:Get*"           ← All S3 Get operations
}

Grants:
  ✓ s3:GetObject
  ✓ s3:GetBucketLocation
  ✓ s3:GetObjectAcl
  ✓ s3:GetBucketVersioning
  ✗ s3:PutObject (doesn't start with Get)
```

```json
{
  "Effect": "Allow",
  "Action": "*"                 ← ALL actions in ALL services
}

Grants: Everything
⚠️ Administrator-level access
```

**Multiple Actions**

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:DeleteObject"
  ]
}

Grants: Exactly these three S3 operations
```

### Read vs. Write Actions

**Read-Only Access Pattern**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:Get*",                  ← All Get operations
      "s3:List*",                 ← All List operations
      "s3:Describe*"              ← All Describe operations (some services)
    ],
    "Resource": "*"
  }]
}

Allows: Viewing/reading data
Denies: Any modifications
```

**Power User Pattern** (Full access except IAM)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",              ← Everything
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:*",                  ← Except IAM
        "organizations:*",        ← And Organizations
        "account:*"               ← And Account management
      ],
      "Resource": "*"
    }
  ]
}

Allows: Full access to services
Denies: User/permission management
```

### Action Examples by Service

**EC2 Actions**

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:RunInstances",           // Launch new instances
    "ec2:TerminateInstances",     // Stop instances
    "ec2:DescribeInstances",      // View instance details
    "ec2:StartInstances",         // Start stopped instances
    "ec2:StopInstances",          // Stop running instances
    "ec2:CreateTags",             // Add tags to resources
    "ec2:DescribeImages"          // View AMI catalog
  ]
}
```

**S3 Actions**

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",               // Download files
    "s3:PutObject",               // Upload files
    "s3:DeleteObject",            // Delete files
    "s3:ListBucket",              // List bucket contents
    "s3:GetBucketLocation",       // Get bucket region
    "s3:PutBucketPolicy",         // Modify bucket policy
    "s3:GetObjectVersion"         // Access versioned objects
  ]
}
```

**IAM Actions**

```json
{
  "Effect": "Allow",
  "Action": [
    "iam:CreateUser",             // Create new IAM users
    "iam:DeleteUser",             // Remove users
    "iam:AttachUserPolicy",       // Attach policies to users
    "iam:CreateRole",             // Create roles
    "iam:PassRole",               // Allow passing roles to services
    "iam:GetUser",                // View user details
    "iam:ListUsers"               // List all users
  ]
}
```

### The Special "PassRole" Permission

**What is PassRole?**

PassRole is a unique IAM permission required when you need to give an AWS service permission to assume a role.

**Why It Exists**

```
Without PassRole protection:

  User: alice (has ec2:RunInstances)

  Alice could:
    1. Launch EC2 instance
    2. Attach ANY role to it (including admin role)
    3. SSH into instance
    4. Use admin role credentials
    5. Do anything in the account!

  This would be privilege escalation!
```

**How PassRole Protects**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "ec2:RunInstances",
    "Resource": "*"
  },
  {
    "Effect": "Allow",
    "Action": "iam:PassRole",
    "Resource": "arn:aws:iam::123456789012:role/LimitedEC2Role"
  }]
}

This policy allows:
  ✓ Launch EC2 instances
  ✓ Attach only the "LimitedEC2Role"
  ✗ Cannot attach any other roles (including admin roles)
```

**Common PassRole Scenarios**

```
Scenario 1: Lambda Function Creation
  User creates Lambda function
  User must pass Lambda an execution role
  User needs: lambda:CreateFunction + iam:PassRole

Scenario 2: ECS Task Definition
  User creates ECS task
  User must pass task an IAM role
  User needs: ecs:RegisterTaskDefinition + iam:PassRole

Scenario 3: EC2 Instance Launch
  User launches EC2 instance
  User attaches instance profile
  User needs: ec2:RunInstances + iam:PassRole
```

### Combining Principals and Actions

**Real-World Policy Example: Cross-Account Lambda Trigger**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAccountBLambdaInvoke",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::222222222222:role/ExternalAppRole"
    },
    "Action": "lambda:InvokeFunction",
    "Resource": "arn:aws:lambda:us-east-1:111111111111:function:MyFunction"
  }]
}

Who: Role from account 222222222222
What: Can invoke Lambda function
Where: Specific function in account 111111111111
```

**Service Principal Example: CloudWatch Logs to S3**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AWSCloudTrailWrite",
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudtrail.amazonaws.com"
    },
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-cloudtrail-bucket/*",
    "Condition": {
      "StringEquals": {
        "s3:x-amz-acl": "bucket-owner-full-control"
      }
    }
  }]
}

Who: AWS CloudTrail service
What: Can write objects to S3
Where: Specific S3 bucket
When: Only with correct ACL
```

---


## Conditions in Policies

### What Are Policy Conditions?

**Conditions** add context-based logic to IAM policies. They specify **when** or **under what circumstances** a policy statement applies. Think of conditions as "if-then" rules: "If certain criteria are met, then this policy applies."

### Why Conditions Exist

```
Without Conditions:
  Policy: Allow s3:GetObject on my-bucket/*
  Problem: Anyone with this policy can access from anywhere, anytime

With Conditions:
  Policy: Allow s3:GetObject on my-bucket/*
          BUT ONLY IF:
            - Request comes from corporate IP range
            - User has MFA enabled
            - Request uses HTTPS
            - It's during business hours

This enables:
  ✓ Context-aware security
  ✓ Compliance requirements
  ✓ Conditional access patterns
  ✓ Dynamic permission adjustments
```

### Condition Structure

```json
{
  "Condition": {
    "ConditionOperator": {
      "ConditionKey": "ConditionValue"
    }
  }
}

Components:
  ConditionOperator: How to compare (equals, greater than, etc.)
  ConditionKey: What to check (IP address, MFA status, etc.)
  ConditionValue: What value to compare against
```


[Continuing from Conditions section...]

## Session Policies

### What Are Session Policies?

**Session Policies** are optional IAM policies that you pass when you programmatically create a temporary session using AWS STS (Security Token Service). They further limit the permissions of a role session beyond what the role's identity-based policy already allows.

Session policies are used with:
- `AssumeRole`
- `AssumeRoleWithSAML`
- `AssumeRoleWithWebIdentity`
- `GetFederationToken`

### Why Session Policies Exist

They enable **just-in-time permission reduction** without modifying the base role.

```
Scenario: Shared role with varying access needs

Base Role "DataAccessRole":
  - Allows: s3:GetObject on all buckets (s3://*)

Problem: Sometimes you need broader access, sometimes narrower

Traditional Solution (inflexible):
  - Create multiple roles (DataAccessAll, DataAccessProjectA, etc.)
  - Manage many roles

Session Policy Solution (flexible):
  - One role with broad base permissions
  - Pass session policy at assume-time to narrow permissions

Example:
  AssumeRole + Session Policy limiting to specific bucket
  Result: Temporary credentials only work for that bucket
```

### How Session Policies Work

```
Effective Permissions = Role Policy ∩ Session Policy

┌────────────────────────────────────────────────────────────┐
│ Role: DataAccessRole                                        │
│ Identity Policy: Allow s3:* on *                            │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ∩ (intersection)
                      │
┌─────────────────────▼──────────────────────────────────────┐
│ Session Policy (passed at AssumeRole):                      │
│ Allow s3:GetObject on arn:aws:s3:::project-alpha/*         │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
        ┌──────────────────────────────┐
        │ Effective Temporary Creds:    │
        │ s3:GetObject on project-alpha │
        │ (for duration of session)     │
        └──────────────────────────────┘
```

---

## Service-Linked Roles

### What Are Service-Linked Roles?

**Service-Linked Roles** are predefined IAM roles that are linked directly to specific AWS services. They include all the permissions that the service requires to call other AWS services on your behalf.

### Why Service-Linked Roles Exist

Many AWS services need to access other AWS services to function properly:

```
Example: Amazon Lex needs to:
  - Write logs to CloudWatch
  - Invoke Lambda functions
  - Access Polly for text-to-speech

Without Service-Linked Roles:
  ❌ You manually create role
  ❌ You figure out exact permissions needed
  ❌ AWS updates service, you need to update role
  ❌ Easy to misconfigure

With Service-Linked Roles:
  ✓ AWS creates the role automatically
  ✓ AWS defines the perfect permissions
  ✓ AWS updates permissions as service evolves
  ✓ Cannot be misconfigured
```

### Characteristics

- **AWS-Managed**: Permissions defined by the service, not by you
- **Auto-Created**: Created automatically when you enable a service (or manually via API)
- **Limited Deletion**: Cannot be deleted if service is using it
- **Specific Trust Policy**: Only the service can assume it
- **Predefined Permissions**: You cannot modify the permissions policy

### Common Service-Linked Roles

```
Amazon EC2 Auto Scaling
  Role: AWSServiceRoleForAutoScaling
  Allows: Launching/terminating instances, modifying load balancers

Amazon RDS
  Role: AWSServiceRoleForRDS
  Allows: Managing DB instances, snapshots, monitoring

AWS Lambda
  Role: AWSServiceRoleForLambdaReplicator
  Allows: Replicating functions across regions

Amazon ECS
  Role: AWSServiceRoleForECS
  Allows: Managing container instances, load balancers

AWS Config
  Role: AWSServiceRoleForConfig
  Allows: Recording configuration, delivering to S3
```

---

## Cross-Account Access

### What Is Cross-Account Access?

Cross-account access allows identities in one AWS account to access resources in a different AWS account. This is essential for multi-account architectures.

### Why Cross-Account Access Exists

```
Organization Structure:
  - Production Account (111111111111)
  - Development Account (222222222222)
  - Logging Account (333333333333)
  - Security Account (444444444444)

Needs:
  • Developers access production (read-only)
  • All accounts send logs to logging account
  • Security team audits all accounts

Without cross-account: Each account is isolated island
With cross-account: Controlled sharing between accounts
```

### Cross-Account Access Patterns

**Pattern 1: Role Assumption (Most Common)**

```
Account A (Production)                Account B (Development)
┌────────────────────────┐           ┌─────────────────────────┐
│ IAM Role:              │           │ IAM User: alice-dev     │
│ "ProdReadOnlyRole"     │           │                         │
│                        │           │ Policy: Can assume      │
│ Trust Policy:          │◄──────────│ ProdReadOnlyRole in     │
│ Allow Account B        │  assumes  │ Account A               │
│ (222222222222)         │           │                         │
│                        │           └─────────────────────────┘
│ Permission Policy:     │
│ Allow S3, CloudWatch   │
│ Read operations        │
└────────────────────────┘

Flow:
  1. alice-dev calls sts:AssumeRole
  2. Specifies ProdReadOnlyRole ARN
  3. Receives temporary credentials
  4. Uses temp creds to access Prod Account resources
  5. Credentials expire (1-12 hours)
```

**Pattern 2: Resource Policy (S3, SQS, SNS, etc.)**

```
Account A (Data Owner)                Account B (Consumer)
┌────────────────────────┐           ┌────────────────────────┐
│ S3 Bucket:             │           │ IAM Role:              │
│ "shared-analytics"     │           │ "DataProcessorRole"    │
│                        │           │                        │
│ Bucket Policy:         │           │ Identity Policy:       │
│ Allow Account B        │◄──────────│ Allow s3:GetObject     │
│ (222222222222)         │  accesses │ on shared-analytics    │
│ s3:GetObject           │           │                        │
└────────────────────────┘           └────────────────────────┘

Both policies must allow!
```

---

## AssumeRole

### What Is AssumeRole?

`AssumeRole` is an AWS STS (Security Token Service) API that returns temporary security credentials that you can use to access AWS resources.

### The AssumeRole Process

```
┌──────────────────────────────────────────────────────────────┐
│                     AssumeRole Flow                           │
└──────────────────────────────────────────────────────────────┘

Step 1: Initial Authentication
┌────────────────┐
│ IAM User/Role  │  "I want to assume arn:aws:iam::XXXX:role/MyRole"
│ alice-dev      │
└────────┬───────┘
         │ (1) Call sts:AssumeRole API
         ▼
┌────────────────────────────────┐
│ AWS STS Service                │
│ (Security Token Service)       │
└────────┬───────────────────────┘
         │ (2) Checks:
         │     - Is caller authenticated?
         │     - Does caller have sts:AssumeRole permission?
         │     - Does role's trust policy allow caller?
         ▼
┌────────────────────────────────┐
│ Target Role: MyRole            │
│ Trust Policy                   │
└────────┬───────────────────────┘
         │ (3) Trust policy evaluation
         │     ✓ Caller is allowed
         ▼
┌────────────────────────────────┐
│ STS Returns Temporary Creds   │
│ - AccessKeyId (temp)           │
│ - SecretAccessKey (temp)       │
│ - SessionToken                 │
│ - Expiration (default 1 hour)  │
└────────┬───────────────────────┘
         │ (4) Use credentials
         ▼
┌────────────────────────────────┐
│ Access AWS Resources           │
│ with role's permissions        │
└────────────────────────────────┘
```

### Example: Assuming a Role (CLI)

```bash
# Assume a role and get temporary credentials
aws sts assume-role \
  --role-arn "arn:aws:iam::123456789012:role/MyS3AccessRole" \
  --role-session-name "my-session"

# Response (temporary credentials):
{
  "Credentials": {
    "AccessKeyId": "ASIAXXX...",        # Temporary access key
    "SecretAccessKey": "abc123...",      # Temporary secret
    "SessionToken": "FwoGZXIv...",       # Session token (required!)
    "Expiration": "2026-02-06T15:00:00Z" # When it expires
  },
  "AssumedRoleUser": {
    "AssumedRoleId": "AROAIDPPEZS35WEXAMPLE:my-session",
    "Arn": "arn:aws:sts::123456789012:assumed-role/MyS3AccessRole/my-session"
  }
}
```

---

# Advanced IAM

## IAM Identity Center (SSO)

### What Is IAM Identity Center?

Formerly AWS Single Sign-On (SSO), **IAM Identity Center** is a centralized service for managing access to multiple AWS accounts and business applications using a single set of credentials.

### Why IAM Identity Center Exists

```
Traditional Multi-Account Problem:
  - 50 AWS accounts
  - 200 employees
  - Each employee needs credentials for multiple accounts

Without Identity Center:
  ❌ 200 users × 50 accounts = 10,000 IAM users to manage
  ❌ Different passwords for each account
  ❌ Password rotation nightmare
  ❌ Onboarding takes days
  ❌ Offboarding risks (forgot one account)

With Identity Center:
  ✓ One identity per employee
  ✓ One login for all accounts
  ✓ Centralized user management
  ✓ Automatic propagation across accounts
  ✓ Onboarding in minutes
  ✓ Offboarding removes all access instantly
```

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│            Corporate Identity Provider (Optional)             │
│               (Active Directory, Okta, etc.)                  │
└────────────────────────────┬─────────────────────────────────┘
                             │ SAML/SCIM
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                   AWS IAM Identity Center                     │
│  ┌────────────────────────────────────────────────────┐      │
│  │ Users & Groups                                     │      │
│  │ • john@company.com (Engineering group)             │      │
│  │ • alice@company.com (Data team)                    │      │
│  └────────────────────────────────────────────────────┘      │
│  ┌────────────────────────────────────────────────────┐      │
│  │ Permission Sets (reusable role templates)          │      │
│  │ • DeveloperAccess (EC2, S3, Lambda full)           │      │
│  │ • ReadOnlyAccess (all services read-only)          │      │
│  │ • DatabaseAdmin (RDS, DynamoDB full)               │      │
│  └────────────────────────────────────────────────────┘      │
└────────────────────────────┬─────────────────────────────────┘
                             │ Deploys as IAM roles
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                   AWS Organization                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Account 1    │  │ Account 2    │  │ Account 3    │       │
│  │ (Production) │  │ (Development)│  │ (Staging)    │       │
│  │              │  │              │  │              │       │
│  │ Roles auto-  │  │ Roles auto-  │  │ Roles auto-  │       │
│  │ created from │  │ created from │  │ created from │       │
│  │ permission   │  │ permission   │  │ permission   │       │
│  │ sets         │  │ sets         │  │ sets         │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────────────────────────────────────────────┘
```

### Key Concepts

**Permission Sets**
Templates that define what users can do in specific accounts.

```
Permission Set: "DeveloperAccess"
  AWS Managed Policy: PowerUserAccess
  Custom Inline Policy:
    - Allow: cloudwatch:PutMetricData
    - Deny: ec2:*SpotInstance*
  Session Duration: 8 hours

When assigned to user "john" in account "Production":
  → Creates IAM role in Production account
  → john can assume this role via SSO portal
```

---

## SAML 2.0 Federation

### What Is SAML 2.0 Federation?

SAML (Security Assertion Markup Language) 2.0 is a standard for exchanging authentication and authorization data between identity providers and service providers. In AWS context, it allows corporate users to access AWS using their existing corporate credentials.

### Why SAML Federation Exists

```
Corporate Environment:
  - Employees already have company credentials
  - Company uses Active Directory, Okta, Azure AD, etc.
  - Don't want separate AWS passwords
  - Need centralized access control

SAML enables:
  ✓ Single Sign-On to AWS
  ✓ Use existing corporate credentials
  ✓ Centralized user management
  ✓ Compliance with corporate policies
```

### SAML Authentication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    SAML 2.0 Flow                             │
└─────────────────────────────────────────────────────────────┘

1. User accesses corporate SSO portal
   ┌────────────┐
   │   User     │  Opens https://sso.company.com
   └──────┬─────┘
          │ (1) Requests AWS access
          ▼
   ┌────────────────────┐
   │ Corporate IdP      │  (Identity Provider)
   │ (Okta/Azure AD)    │
   └──────┬─────────────┘
          │ (2) Authenticates user (username/password/MFA)
          │     ✓ Verified
          │
          │ (3) Generates SAML assertion
          │     (XML document with user identity & permissions)
          ▼
   ┌────────────────────┐
   │ AWS Sign-In        │
   │ Endpoint           │
   └──────┬─────────────┘
          │ (4) Posts SAML assertion
          ▼
   ┌────────────────────┐
   │ AWS STS            │
   │ (AssumeRoleWithSAML)
   └──────┬─────────────┘
          │ (5) Validates SAML assertion
          │     Checks:
          │     - Signature valid?
          │     - Not expired?
          │     - Trusted IdP?
          │
          │ (6) Returns temporary credentials
          ▼
   ┌────────────────────┐
   │ User gets AWS      │
   │ Console Access     │
   │ or temp credentials│
   └────────────────────┘
```

---

## OIDC Federation

### What Is OIDC Federation?

**OpenID Connect (OIDC)** is an identity layer on top of OAuth 2.0. In AWS, OIDC federation allows you to authenticate users through identity providers like GitHub, Google, Azure AD, or any OIDC-compliant provider.

### Common Use Cases

```
1. GitHub Actions deploying to AWS
   - GitHub OIDC → AWS Role
   - No long-term AWS credentials in GitHub secrets

2. GitLab CI/CD pipelines
   - GitLab OIDC → AWS Role
   - Secure, temporary credentials

3. Kubernetes Service Accounts (EKS)
   - Pod identity → AWS Role
   - Fine-grained permissions per pod
```

### Example: GitHub Actions with OIDC

```
Setup:
1. Create OIDC Identity Provider in AWS
   Provider URL: https://token.actions.githubusercontent.com
   Audience: sts.amazonaws.com

2. Create IAM Role with trust policy:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:MyOrg/MyRepo:*"
      }
    }
  }]
}

3. GitHub Actions workflow:
   - Workflow requests OIDC token from GitHub
   - Presents token to AWS
   - AWS validates and issues temp credentials
   - Workflow uses credentials to deploy
```

---

## Attribute-Based Access Control (ABAC)

### What Is ABAC?

**Attribute-Based Access Control** uses tags (attributes) to control permissions. Instead of creating different policies for different resources, you use tag-based conditions.

### RBAC vs ABAC

```
RBAC (Role-Based Access Control) - Traditional:
┌────────────────────────────────────────────────┐
│ DeveloperRole:                                 │
│   Allow: s3:* on project-alpha-*               │
│   Allow: s3:* on project-beta-*                │
│   Allow: s3:* on project-gamma-*               │
│   ...add more as projects grow                 │
└────────────────────────────────────────────────┘

Problem: Need to update policy for every new project

ABAC (Attribute-Based Access Control) - Dynamic:
┌────────────────────────────────────────────────┐
│ DeveloperRole:                                 │
│   Allow: s3:*                                  │
│   Condition:                                   │
│     aws:PrincipalTag/Team = s3:ExistingObjectTag/Team
└────────────────────────────────────────────────┘

Solution: If developer's Team tag matches resource's Team tag, allow
         No policy updates needed for new projects!
```

### ABAC Example

```
Setup:
1. Tag IAM users with attributes
   alice-dev:
     Team = Engineering
     Project = Alpha

   bob-dev:
     Team = Engineering
     Project = Beta

2. Tag Resources
   s3://project-alpha-data/
     Team = Engineering
     Project = Alpha

   ec2-instance-123:
     Team = Engineering
     Project = Beta

3. Create ABAC Policy:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "s3:ExistingObjectTag/Project": "${aws:PrincipalTag/Project}"
      }
    }
  }]
}

Result:
  alice-dev (Project=Alpha) → Can access project-alpha-data ✓
  alice-dev → Cannot access project-beta-data ✗
  bob-dev (Project=Beta) → Can access project-beta-data ✓
  bob-dev → Cannot access project-alpha-data ✗

No policy changes needed when creating project-gamma!
```

---

## IAM Access Analyzer

### What Is IAM Access Analyzer?

IAM Access Analyzer helps identify resources in your organization that are shared with external entities. It uses automated reasoning to analyze resource policies.

### What It Detects

```
Analyzes:
  • S3 buckets
  • IAM roles
  • KMS keys
  • Lambda functions
  • SQS queues
  • SNS topics
  • Secrets Manager secrets
  • EBS volume snapshots
  • RDS snapshots
  • ECR repositories

Finds:
  ✓ Public access
  ✓ Cross-account access
  ✓ Access from other AWS Organizations
  ✓ Overly permissive policies
```

### How It Works

```
┌──────────────────────────────────────────────────────────┐
│                IAM Access Analyzer                        │
├──────────────────────────────────────────────────────────┤
│                                                           │
│ 1. Define Trust Zone                                     │
│    (Your AWS Organization or specific accounts)          │
│                                                           │
│ 2. Continuous Analysis                                   │
│    Automatically scans resource policies                 │
│                                                           │
│ 3. Generates Findings                                    │
│    ┌──────────────────────────────────────┐             │
│    │ Finding: S3 bucket public            │             │
│    │ Resource: arn:aws:s3:::my-bucket     │             │
│    │ External Access: Internet (*)        │             │
│    │ Severity: High                       │             │
│    │ Actions: s3:GetObject                │             │
│    └──────────────────────────────────────┘             │
│                                                           │
│ 4. Recommendations                                       │
│    • Review and remove public access                     │
│    • Restrict to specific principals                     │
│    • Add condition for source IP                         │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

---

## IAM Policy Simulator

### What Is Policy Simulator?

The IAM Policy Simulator lets you test and troubleshoot IAM policies before deploying them or to understand why access is denied.

### Use Cases

```
1. Testing New Policies
   Before: Attach policy → User tries → Fails → Debug
   With Simulator: Test policy → See results → Fix → Deploy

2. Debugging Access Issues
   "Why can't alice access this S3 bucket?"
   → Simulate alice accessing bucket
   → See which policies applied
   → Identify blocking policy

3. Validating Permissions
   "Does this contractor have EC2 delete access?"
   → Simulate EC2 TerminateInstances
   → Confirm allowed or denied
```

### How to Use

```
Via AWS Console:
  1. Navigate to IAM → Policy Simulator
  2. Select User/Role/Group
  3. Select AWS Service (e.g., S3)
  4. Select Actions (e.g., GetObject)
  5. Optionally: Specify resource ARN
  6. Run Simulation

Results:
  ✅ Allowed - Shows which policy granted access
  ❌ Denied - Shows which policy denied or lack of allow

Details include:
  • Matching policies
  • Evaluation logic
  • Condition evaluation results
```

---

## Credential Report

### What Is Credential Report?

A CSV report listing all IAM users in your account and the status of their credentials (passwords, access keys, MFA).

### Information Included

```
For each IAM user:
  • User name and ARN
  • User creation time
  • Password enabled (yes/no)
  • Password last used
  • Password last changed
  • Password next rotation
  • MFA active (yes/no)
  • Access key 1 active (yes/no)
  • Access key 1 last rotated
  • Access key 1 last used
  • Access key 2 active (yes/no)
  • Access key 2 last rotated
  • Access key 2 last used
  • Signing certificates
```

### Use Cases

```
Security Audits:
  • Find users without MFA
  • Identify old, unused access keys
  • Detect inactive accounts
  • Verify password rotation compliance

Compliance:
  • Prove MFA enforcement
  • Show credential rotation
  • Document access key usage

Cleanup:
  • Remove unused credentials
  • Disable inactive users
  • Rotate old access keys
```

---

## Access Advisor

### What Is Access Advisor?

Access Advisor shows service permissions granted to a user, role, or group and when those services were last accessed. This helps you implement least privilege.

### What It Shows

```
For a given IAM entity:

Service: Amazon S3
  Last Accessed: 2 days ago
  Granted By: DeveloperPolicy

Service: Amazon EC2
  Last Accessed: Never
  Granted By: DeveloperPolicy

Service: AWS Lambda
  Last Accessed: 5 hours ago
  Granted By: DeveloperPolicy

Insight: EC2 permissions granted but never used → Remove them!
```

### Use Cases

```
1. Right-Size Permissions
   - User has 20 service permissions
   - Only uses 8 services
   - Remove unused 12 services

2. Identify Excessive Access
   - DeveloperRole has Admin access
   - Actually only uses EC2, S3, Lambda
   - Create narrower policy

3. Compliance Documentation
   - Prove principle of least privilege
   - Show regular permission reviews
   - Demonstrate access reduction over time
```

---

## IAM Roles Anywhere

### What Is IAM Roles Anywhere?

IAM Roles Anywhere allows workloads running **outside AWS** (on-premises servers, other clouds, edge devices) to obtain temporary AWS credentials by using X.509 certificates.

### Why It Exists

```
Traditional Problem:
  On-premises server needs AWS access
  ❌ Create IAM user with access keys
  ❌ Store keys on server
  ❌ Keys are long-lived
  ❌ Keys can be stolen
  ❌ Manual rotation needed

IAM Roles Anywhere Solution:
  ✓ Server has X.509 certificate (from trusted CA)
  ✓ Uses certificate to authenticate
  ✓ Gets temporary AWS credentials (like roles)
  ✓ Credentials auto-rotate
  ✓ No long-term keys
```

### How It Works

```
┌──────────────────────────────────────────────────────────┐
│        On-Premises Server (or other cloud VM)            │
│                                                           │
│  Has: X.509 Certificate from Company CA                  │
│       Private Key (kept secret)                          │
└────────────────────┬─────────────────────────────────────┘
                     │
                     │ (1) Request credentials
                     │     - Present certificate
                     │     - Sign request with private key
                     ▼
┌──────────────────────────────────────────────────────────┐
│              AWS IAM Roles Anywhere                       │
│                                                           │
│  (2) Validates:                                           │
│      • Certificate from trusted CA?                       │
│      • Certificate not expired?                           │
│      • Signature valid?                                   │
│      • Certificate matches trust anchor?                  │
│                                                           │
│  (3) If valid → Issue temporary credentials               │
└────────────────────┬─────────────────────────────────────┘
                     │
                     │ Returns temporary credentials
                     ▼
┌──────────────────────────────────────────────────────────┐
│  Server now has:                                         │
│  • AccessKeyId (temp)                                    │
│  • SecretAccessKey (temp)                                │
│  • SessionToken                                          │
│  • Expiration (typically 1 hour)                         │
│                                                           │
│  Uses these to access AWS services                       │
└──────────────────────────────────────────────────────────┘
```

### Setup Components

```
1. Trust Anchor
   - References your certificate authority (CA)
   - Can be AWS Certificate Manager Private CA
   - Or external CA certificate

2. Profile
   - Links to IAM role
   - Defines permissions workload will receive
   - Can specify session duration, tags

3. X.509 Certificate
   - Issued by trusted CA
   - Installed on workload
   - Used to authenticate
```

### Example Use Case

```
Scenario: Data center servers backup to S3

Traditional (Insecure):
  • IAM user with access keys on every server
  • Keys in config files
  • Manual rotation every 90 days
  • Audit trail shows "iam_user" (which server?)

With Roles Anywhere (Secure):
  • Each server has unique X.509 cert
  • Certificate identifies the server
  • Temporary credentials (auto-rotate every hour)
  • Audit trail shows specific server via cert serial number
  • Revoke one certificate if server compromised
  • No long-term keys to steal
```

---

## Summary and Best Practices

### IAM Best Practices

```
1. Root Account
   ✓ Enable MFA
   ✓ Don't use for daily tasks
   ✓ Lock away credentials
   ✓ Set up billing alerts

2. Users
   ✓ Create individual users (no sharing)
   ✓ Enable MFA for privileged users
   ✓ Rotate credentials regularly
   ✓ Remove unnecessary permissions

3. Groups
   ✓ Assign permissions to groups
   ✓ Add users to groups
   ✓ Don't attach policies directly to users

4. Roles
   ✓ Use roles for applications
   ✓ Use roles for cross-account access
   ✓ Prefer roles over users where possible

5. Policies
   ✓ Principle of least privilege
   ✓ Use AWS managed policies where possible
   ✓ Test policies with Policy Simulator
   ✓ Review with Access Advisor

6. Monitoring
   ✓ Enable CloudTrail
   ✓ Review credential reports monthly
   ✓ Use IAM Access Analyzer
   ✓ Set up alerts for suspicious activity

7. Compliance
   ✓ Enforce MFA for sensitive operations
   ✓ Require password complexity
   ✓ Set password expiration
   ✓ Restrict from untrusted networks
```

### Common Mistakes to Avoid

```
❌ Using root account for daily operations
❌ Sharing IAM users between people
❌ Hardcoding access keys in code
❌ Overly permissive policies ("*")
❌ Not enabling MFA
❌ Not rotating credentials
❌ Not reviewing permissions regularly
❌ Granting more permissions than needed
❌ Ignoring Access Advisor recommendations
❌ Not monitoring with CloudTrail
```

---

## Conclusion

IAM is the foundation of AWS security. Understanding these concepts deeply enables you to:
- Build secure, scalable access control
- Implement least privilege effectively
- Troubleshoot permission issues quickly
- Meet compliance requirements
- Delegate administration safely
- Integrate with corporate identity systems

The investment in learning IAM pays dividends across every AWS service and architecture you build.

---

**End of Document**