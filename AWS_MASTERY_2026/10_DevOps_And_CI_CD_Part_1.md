# AWS DevOps & CI/CD - Comprehensive Guide

## Table of Contents
1. [CodePipeline](#codepipeline)
2. [CodeBuild](#codebuild)
3. [CodeDeploy](#codedeploy)
4. [CodeCommit](#codecommit)
5. [CodeArtifact](#codeartifact)
6. [CloudFormation](#cloudformation)
7. [CDK (Cloud Development Kit)](#cdk-cloud-development-kit)
8. [Terraform on AWS](#terraform-on-aws)

---

## Introduction to AWS DevOps & CI/CD

**What is DevOps?**
DevOps is a cultural and technical approach that combines software development (Dev) and IT operations (Ops) to shorten the development lifecycle and deliver high-quality software continuously. It emphasizes automation, collaboration, and rapid feedback loops.

**What is CI/CD?**
- **CI (Continuous Integration)**: Automatically building and testing code every time a developer commits changes
- **CD (Continuous Delivery/Deployment)**: Automatically deploying tested code to staging or production environments

**Why AWS DevOps Services?**
Traditional software delivery is slow and error-prone:
- Manual builds lead to "works on my machine" problems
- Manual deployments cause human errors and downtime
- Lack of automation slows release cycles from months to hours

AWS provides a complete suite of tools to automate the entire software delivery pipeline—from code commit to production deployment.

---

## CodePipeline

### What is CodePipeline?

**AWS CodePipeline** is a fully managed **continuous delivery service** that orchestrates and automates the entire software release process. Think of it as a **conveyor belt for your code**—it automatically moves your code through different stages (build, test, deploy) every time you make a change.

**Real-World Problem It Solves:**
Without CodePipeline, teams manually coordinate:
- Pulling code from repositories
- Running build scripts
- Executing tests
- Deploying to different environments
- Waiting for approvals

This manual process is slow, error-prone, and doesn't scale. CodePipeline automates this entire workflow, turning hours of manual work into minutes of automated execution.

**How It Works (Conceptual):**

```
Developer commits code
         ↓
    CodePipeline detects change
         ↓
    ┌─────────────────────────────────────────┐
    │         PIPELINE EXECUTION              │
    ├─────────────────────────────────────────┤
    │  Stage 1: Source                        │
    │  [Get code from GitHub/CodeCommit]      │
    │            ↓                            │
    │  Stage 2: Build                         │
    │  [Compile, run tests via CodeBuild]     │
    │            ↓                            │
    │  Stage 3: Manual Approval               │
    │  [Wait for human approval]              │
    │            ↓                            │
    │  Stage 4: Deploy                        │
    │  [Deploy to EC2/ECS via CodeDeploy]     │
    └─────────────────────────────────────────┘
         ↓
    Application running in production
```

---

### Stages & Actions

**What Are Stages?**
A **stage** is a logical grouping of actions in your pipeline. Each stage represents a phase in your release process. Stages execute **sequentially**—one after another.

**Common Stage Pattern:**
1. **Source Stage** - Get the latest code
2. **Build Stage** - Compile and test
3. **Staging Stage** - Deploy to test environment
4. **Production Stage** - Deploy to live environment

**What Are Actions?**
An **action** is a specific task performed within a stage. Multiple actions within a stage can run in **parallel** or **sequentially**.

**Example:**
```
Stage: Build
├── Action 1: Run unit tests (parallel)
├── Action 2: Run integration tests (parallel)
└── Action 3: Create Docker image (runs after tests pass)
```

**Visual Representation:**

```
┌──────────────────────────────────────────────────────┐
│                     PIPELINE                         │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────────┐   ┌─────────────┐   ┌───────────┐  │
│  │   Stage 1   │ → │   Stage 2   │ → │  Stage 3  │  │
│  │   Source    │   │    Build    │   │   Deploy  │  │
│  └─────────────┘   └─────────────┘   └───────────┘  │
│       │                  │                  │        │
│  ┌────▼────┐        ┌────▼────┐        ┌────▼────┐  │
│  │ Action: │        │ Action: │        │ Action: │  │
│  │ GitHub  │        │CodeBuild│        │CodeDeploy│ │
│  └─────────┘        └─────────┘        └─────────┘  │
│                          │                           │
│                     ┌────▼────┐                      │
│                     │ Action: │                      │
│                     │  Test   │                      │
│                     └─────────┘                      │
└──────────────────────────────────────────────────────┘
```

**Key Concepts:**
- Stages run **one at a time** (sequential)
- Actions within a stage can run **in parallel** (faster execution)
- If any action fails, the entire stage fails and pipeline stops
- Each stage can have 1-50 actions

---

### Source Actions

**What Are Source Actions?**
Source actions are the **entry point** of your pipeline. They detect changes in your code repository and trigger the pipeline execution. This is where CodePipeline "watches" for new commits.

**Supported Source Providers:**
1. **AWS CodeCommit** - AWS's native Git repository
2. **GitHub** - Public and private repositories
3. **Bitbucket** - Atlassian's Git service
4. **Amazon S3** - For pre-packaged artifacts
5. **Amazon ECR** - For Docker container images

**How Source Detection Works:**

```
Developer pushes code to GitHub
         ↓
GitHub sends webhook to CodePipeline
         ↓
CodePipeline detects change
         ↓
Downloads source code as ZIP artifact
         ↓
Passes artifact to next stage
```

**Example Scenario:**
```
Repository: github.com/mycompany/webapp
Branch: main
Pipeline Configuration:
- Detects any push to 'main' branch
- Downloads entire repository
- Creates artifact: SourceOutput.zip
- Passes to Build stage
```

**Key Concepts:**
- Source actions create **output artifacts** (ZIP files containing your code)
- These artifacts are stored in an **S3 bucket** automatically created by CodePipeline
- Subsequent stages consume these artifacts as **input artifacts**

**Detection Methods:**
1. **Webhooks** (recommended) - Near real-time, repository sends notification to AWS
2. **Polling** (legacy) - CodePipeline checks repository every few minutes
3. **CloudWatch Events** - Event-driven triggers for CodeCommit

---

### Build Actions

**What Are Build Actions?**
Build actions take your source code and transform it into a deployable artifact. This includes:
- Compiling code
- Running tests
- Creating Docker images
- Packaging applications
- Generating documentation

**Primary Build Provider: AWS CodeBuild**

**How Build Actions Work:**

```
Input: Source code (from Source stage)
         ↓
    Build Action starts
         ↓
┌────────────────────────────────┐
│     CodeBuild Environment      │
│  ┌──────────────────────────┐  │
│  │ 1. Install dependencies  │  │
│  │ 2. Compile code          │  │
│  │ 3. Run unit tests        │  │
│  │ 4. Create JAR/WAR/ZIP    │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
         ↓
Output: Built artifact (app.jar, Docker image, etc.)
         ↓
    Pass to Deploy stage
```

**Real-World Example:**

Imagine you're building a Java web application:

```
Source Stage Output: Raw .java files, pom.xml
         ↓
Build Action (CodeBuild):
  - Runs: mvn clean package
  - Compiles .java → .class files
  - Runs JUnit tests
  - Creates: webapp.war
         ↓
Build Stage Output: webapp.war (deployable)
```

**Build Action Configuration:**

```yaml
# What CodePipeline tells CodeBuild:
Input Artifact: SourceOutput
Build Project: MyAppBuildProject
Environment Variables:
  - ENV: production
  - VERSION: 1.2.3
Output Artifact: BuildOutput
```

**Multiple Build Actions in Parallel:**

```
Source Code
     ↓
┌────────────────────────────┐
│      Build Stage           │
├────────────────────────────┤
│  ┌──────────┐  ┌─────────┐ │
│  │ Build    │  │ Build   │ │
│  │ Backend  │  │Frontend │ │
│  │ (Java)   │  │ (React) │ │
│  └──────────┘  └─────────┘ │
│       ↓             ↓      │
│   API.jar      bundle.js   │
└────────────────────────────┘
```

---

### Deploy Actions

**What Are Deploy Actions?**
Deploy actions take your built artifacts and deploy them to target environments (servers, containers, Lambda functions). This is where your code becomes a running application.

**Supported Deploy Providers:**
1. **AWS CodeDeploy** - Deploy to EC2, on-premises servers, Lambda, ECS
2. **AWS CloudFormation** - Deploy infrastructure as code
3. **AWS Elastic Beanstalk** - Deploy to managed platform
4. **Amazon ECS** - Deploy Docker containers
5. **AWS S3** - Deploy static websites
6. **AWS Service Catalog** - Deploy approved products

**How Deploy Actions Work:**

```
Input: Built artifact (from Build stage)
         ↓
    Deploy Action triggered
         ↓
┌──────────────────────────────────┐
│      Deployment Provider         │
│   (CodeDeploy/ECS/Lambda)        │
│  ┌────────────────────────────┐  │
│  │ 1. Stop old application    │  │
│  │ 2. Install new version     │  │
│  │ 3. Run health checks       │  │
│  │ 4. Route traffic to new    │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
         ↓
    Application running in target environment
```

**Real-World Deployment Pattern:**

```
Pipeline with Multiple Deployment Stages:

Source → Build → Deploy to Staging → Manual Approval → Deploy to Production
                      ↓                                         ↓
                 staging.myapp.com                       www.myapp.com
                 (Testing environment)                   (Live users)
```

**Example: Deploying to EC2 via CodeDeploy:**

```
Build Stage produces: application.zip
         ↓
Deploy Action configuration:
  - Provider: CodeDeploy
  - Application: MyWebApp
  - Deployment Group: Production-Servers
  - Input Artifact: BuildOutput
         ↓
CodeDeploy:
  1. Downloads application.zip to EC2 instances
  2. Runs deployment scripts (stop app, install, start app)
  3. Performs health checks
  4. Completes deployment or rolls back
```

**Blue/Green Deployment Example:**

```
Before Deployment:
  Load Balancer → Blue Environment (v1.0)
                  Green Environment (empty)

During Deployment:
  Load Balancer → Blue Environment (v1.0)
                  Green Environment (v2.0 deploying...)

After Deployment:
  Load Balancer → Green Environment (v2.0) ✓
                  Blue Environment (v1.0, ready for rollback)
```

**Key Concepts:**
- Deploy actions can target **multiple environments** (dev, staging, production)
- Each environment can have different configurations
- Failed deployments can trigger **automatic rollbacks**
- Health checks ensure new version is working before completing

---

### Approval Actions

**What Are Approval Actions?**
Approval actions are **human decision points** in your pipeline. They pause automated execution and wait for someone to manually approve or reject the deployment before continuing.

**Why Approval Actions Exist:**
Automation is powerful, but some decisions require human judgment:
- Final review before production deployment
- Security/compliance checks
- Business stakeholder sign-off
- Budget approval for infrastructure changes

**How Approval Actions Work:**

```
Build Stage completes successfully
         ↓
Deploy to Staging (automatic)
         ↓
┌─────────────────────────────────┐
│     APPROVAL ACTION             │
│  Pipeline execution PAUSED      │
│                                 │
│  Notification sent to:          │
│  - Email                        │
│  - SNS topic                    │
│  - Slack (via SNS)              │
│                                 │
│  Waiting for approval...        │
│  [ Approve ] [ Reject ]         │
└─────────────────────────────────┘
         ↓ (if approved)
Deploy to Production (automatic)
```

**Real-World Scenario:**

```
Company: E-commerce Platform
Requirement: Legal team must approve changes affecting payment processing

Pipeline:
1. Source (GitHub) → automatic
2. Build (CodeBuild) → automatic
3. Deploy to Dev → automatic
4. Integration Tests → automatic
5. Deploy to Staging → automatic
6. MANUAL APPROVAL ← Legal team reviews
7. Deploy to Production → automatic (only if approved)
```

**Approval Configuration:**

```yaml
Stage: ProductionApproval
Action:
  Name: LegalReview
  Type: Manual
  Configuration:
    NotificationArn: arn:aws:sns:us-east-1:123456789:approvals
    CustomData: "Payment processing changes - requires legal review"
    ExternalEntityLink: https://wiki.company.com/deployment-123
```

**Notification Flow:**

```
Approval Action triggered
         ↓
SNS notification sent
         ↓
┌────────────────────────────────┐
│  Email to: legal@company.com   │
│  Subject: Pipeline Approval    │
│  Body: Review deployment #123  │
│  Link: AWS Console approval    │
└────────────────────────────────┘
         ↓
Reviewer clicks link → AWS Console
         ↓
Reviews changes, logs, test results
         ↓
Clicks "Approve" or "Reject"
         ↓
Pipeline continues or stops
```

**Key Concepts:**
- Approvals can have **time limits** (auto-reject after X hours)
- Multiple approvers can be configured via **SNS topic subscriptions**
- Approval **comments** can be added (reason for approval/rejection)
- Pipeline execution **costs nothing** while waiting for approval

---

### Pipeline Triggers

**What Are Pipeline Triggers?**
Triggers define **when and how** a pipeline starts executing. They answer the question: "What event should kick off this pipeline?"

**Trigger Types:**

**1. Source Repository Triggers (Most Common)**
```
Developer commits to 'main' branch
         ↓
Webhook triggers pipeline
         ↓
Pipeline executes automatically
```

**2. Schedule-Based Triggers**
```
CloudWatch Events Rule: "Every night at 2 AM"
         ↓
Triggers pipeline
         ↓
Nightly build and deployment
```

**3. Manual Triggers**
```
Developer clicks "Release Change" in console
         ↓
Pipeline executes on-demand
```

**4. Event-Based Triggers (Advanced)**
```
S3 bucket receives new file
         ↓
CloudWatch Event detects
         ↓
Triggers pipeline
```

**Webhook vs Polling:**

```
WEBHOOK (Recommended):
GitHub → (instant) → AWS → Pipeline starts immediately
Latency: < 5 seconds
Cost: Free

POLLING (Legacy):
AWS checks GitHub every 5 minutes → "Any changes?" → Pipeline starts
Latency: Up to 5 minutes
Cost: API calls count against rate limits
```

**Real-World Example - Multiple Triggers:**

```
Pipeline: MyWebApp-Pipeline

Trigger 1: Webhook
  - Repository: github.com/mycompany/webapp
  - Branch: main
  - Action: Any push triggers deployment

Trigger 2: Schedule
  - Cron: 0 2 * * ? (Every day at 2 AM UTC)
  - Action: Rebuild and redeploy (fresh deployment)

Trigger 3: Manual
  - Console button
  - Action: Emergency hotfix deployment
```

**Branch Filtering:**

```yaml
# Only trigger on specific branches
Trigger Configuration:
  Source: GitHub
  Repository: mycompany/webapp
  Branch: main  # Only 'main' branch triggers pipeline

# Other branches (feature/*, dev) do NOT trigger
```

**Tag-Based Triggers:**

```
Developer creates Git tag: v1.2.3
         ↓
Pipeline detects tag creation
         ↓
Builds release version 1.2.3
         ↓
Deploys to production with version tag
```

**Key Concepts:**
- Pipelines can have **multiple trigger sources**
- Webhooks are **preferred** over polling (faster, more efficient)
- You can **disable triggers** temporarily without deleting pipeline
- **Manual approval** is different from **manual trigger** (trigger starts pipeline, approval pauses it)

---

### Cross-Region Actions

**What Are Cross-Region Actions?**
Cross-region actions allow your pipeline to deploy resources **across multiple AWS regions** from a single pipeline. This is essential for global applications that need to serve users worldwide with low latency.

**Why Cross-Region Actions Matter:**

**Problem:**
- Your users are in US, Europe, and Asia
- Single-region deployment = high latency for distant users
- Manually deploying to each region = slow and error-prone

**Solution:**
- One pipeline deploys to **all regions simultaneously**
- Consistent deployment across all regions
- Global application updates in minutes

**How Cross-Region Actions Work:**

```
Pipeline in us-east-1 (Primary Region)
         ↓
Source: GitHub (us-east-1)
         ↓
Build: CodeBuild (us-east-1) → Creates artifact
         ↓
┌────────────────────────────────────────────┐
│         Deploy Stage (Multi-Region)        │
├────────────────────────────────────────────┤
│  Action 1: Deploy to us-east-1 (parallel)  │
│  Action 2: Deploy to eu-west-1 (parallel)  │
│  Action 3: Deploy to ap-south-1 (parallel) │
└────────────────────────────────────────────┘
         ↓
Application running in 3 regions simultaneously
```

**Artifact Replication:**

CodePipeline automatically replicates artifacts to target regions:

```
Build produces app.zip in us-east-1
         ↓
CodePipeline copies to:
  - S3 bucket in us-east-1 (source)
  - S3 bucket in eu-west-1 (replica)
  - S3 bucket in ap-south-1 (replica)
         ↓
Each region's deploy action uses local artifact
```

**Real-World Example - Global Web Application:**

```yaml
Pipeline: GlobalWebApp

Stage 1: Source (us-east-1)
  - GitHub repository

Stage 2: Build (us-east-1)
  - CodeBuild creates Docker image
  - Pushes to ECR in us-east-1

Stage 3: Deploy-North-America (parallel)
  - Deploy to us-east-1 (N. Virginia)
  - Deploy to us-west-2 (Oregon)

Stage 4: Deploy-Europe (parallel)
  - Deploy to eu-west-1 (Ireland)
  - Deploy to eu-central-1 (Frankfurt)

Stage 5: Deploy-Asia (parallel)
  - Deploy to ap-southeast-1 (Singapore)
  - Deploy to ap-northeast-1 (Tokyo)
```

**Visual Representation:**

```
                    ┌─────────────────┐
                    │  Source & Build │
                    │   (us-east-1)   │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │   USA    │      │  Europe  │      │   Asia   │
    │ us-east-1│      │eu-west-1 │      │ap-south-1│
    │ us-west-2│      │eu-central│      │ap-ne-1   │
    └──────────┘      └──────────┘      └──────────┘
```

**Key Concepts:**
- Artifacts are **automatically replicated** to S3 buckets in target regions
- You pay for **cross-region data transfer** and **S3 storage** in each region
- Actions in the same stage can deploy to **different regions in parallel**
- Requires **IAM permissions** in each target region

---

### Pipeline Variables

**What Are Pipeline Variables?**
Pipeline variables are **dynamic values** that can be used throughout your pipeline. They allow you to:
- Pass information between stages
- Inject configuration at runtime
- Make pipelines more flexible and reusable

**Types of Variables:**

**1. Namespace Variables (Automatic)**
AWS automatically provides variables from each action:

```
Source Action Variables:
  - #{SourceVariables.BranchName}     → "main"
  - #{SourceVariables.CommitId}       → "a3f5c8d..."
  - #{SourceVariables.CommitMessage}  → "Fix login bug"
  - #{SourceVariables.AuthorDate}     → "2026-02-06"

Build Action Variables:
  - #{BuildVariables.BuildId}         → "webapp:12345"
  - #{BuildVariables.BuildTag}        → "v1.2.3"

Deploy Action Variables:
  - #{DeployVariables.DeploymentId}   → "d-ABC123"
```

**2. Custom Variables (User-Defined)**
You can define your own variables:

```yaml
Variables:
  - Name: Environment
    DefaultValue: staging

  - Name: Version
    DefaultValue: 1.0.0
```

**How Variables Are Used:**

**Example 1: Dynamic Docker Tagging**

```yaml
Build Stage - CodeBuild:
  Environment Variables:
    - COMMIT_ID: #{SourceVariables.CommitId}
    - BUILD_NUMBER: #{BuildVariables.BuildId}

  Build Command:
    # Creates Docker image with unique tag
    docker build -t myapp:${COMMIT_ID}-${BUILD_NUMBER} .

Result: myapp:a3f5c8d-12345
```

**Example 2: CloudFormation Parameters**

```yaml
Deploy Stage - CloudFormation:
  Template: infrastructure.yaml
  Parameters:
    AppVersion: #{Variables.Version}
    Environment: #{Variables.Environment}
    DeploymentTime: #{BuildVariables.BuildStartTime}
```

**Example 3: Conditional Deployment**

```yaml
# Deploy to production only if branch is 'main'
Deploy Action:
  Condition: #{SourceVariables.BranchName} == "main"
  Target: Production
```

**Real-World Use Case - Version Tracking:**

```
Complete Flow with Variables:

1. Developer commits:
   Message: "Add payment feature"
   Commit: a3f5c8d

2. Source Action captures:
   #{SourceVariables.CommitId} = "a3f5c8d"
   #{SourceVariables.CommitMessage} = "Add payment feature"

3. Build Action uses variables:
   docker build -t myapp:#{SourceVariables.CommitId}
   Result: myapp:a3f5c8d

4. Deploy Action tags resources:
   CloudFormation Parameters:
     - ImageTag: #{SourceVariables.CommitId}
     - DeploymentNotes: #{SourceVariables.CommitMessage}

5. Production deployment shows:
   Version: a3f5c8d
   Change: "Add payment feature"
   Timestamp: 2026-02-06 14:30:00
```

**Variable Resolution:**

```
Pipeline execution starts
         ↓
Variables are resolved BEFORE action execution
         ↓
Example Resolution:
  "Deploy version #{Variables.Version} to #{Variables.Environment}"
         ↓
  "Deploy version 1.2.3 to production"
         ↓
Resolved string passed to action
```

**Key Concepts:**
- Variables are **immutable** during pipeline execution
- Use **#{Namespace.VariableName}** syntax
- Variables can be used in: environment variables, parameters, artifact names
- **Default values** prevent pipeline failures if variable is missing

---

## CodeBuild

### What is CodeBuild?

**AWS CodeBuild** is a fully managed **build service** that compiles source code, runs tests, and produces deployable artifacts. Think of it as a **temporary, disposable build server** that appears when you need it and vanishes when done.

**Real-World Problem It Solves:**

**Traditional Build Process (The Old Way):**
```
Company maintains dedicated build servers:
- Purchase and maintain physical/virtual servers
- Install and update build tools (Java, Maven, Node.js)
- Scale servers manually for peak times
- Pay for servers 24/7 even if used 1 hour/day
- Deal with "works on my machine" inconsistencies

Cost: $500-2000/month for idle servers
Complexity: High (server maintenance, security patches)
```

**CodeBuild Approach (The Modern Way):**
```
On-demand build containers:
- No servers to maintain
- Pre-configured build environments
- Auto-scales to any number of concurrent builds
- Pay only for build minutes used
- Consistent, reproducible builds

Cost: $0.005 per build minute (e.g., 5-min build = $0.025)
Complexity: Low (AWS manages everything)
```

**How CodeBuild Works (High Level):**

```
┌─────────────────────────────────────────────────────┐
│              CodeBuild Execution Flow               │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. Trigger received (from CodePipeline or manual) │
│                      ↓                              │
│  2. Provision build container                      │
│     (Docker container with build tools)            │
│                      ↓                              │
│  3. Download source code                           │
│     (from S3, GitHub, CodeCommit)                  │
│                      ↓                              │
│  4. Execute buildspec.yml                          │
│     ┌────────────────────────────┐                 │
│     │ Install dependencies       │                 │
│     │ Run pre-build scripts      │                 │
│     │ Compile code               │                 │
│     │ Run tests                  │                 │
│     │ Package artifacts          │                 │
│     └────────────────────────────┘                 │
│                      ↓                              │
│  5. Upload artifacts to S3                         │
│                      ↓                              │
│  6. Terminate container (cleanup)                  │
│                      ↓                              │
│  7. Return success/failure status                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Visual Representation of Build Lifecycle:**

```
Source Code (GitHub)
         ↓
    ┌────────────────────────────┐
    │   Build Container Starts   │
    │   (Fresh Ubuntu + tools)   │
    ├────────────────────────────┤
    │  Phase 1: INSTALL          │
    │  - npm install             │
    │  - pip install -r req.txt  │
    ├────────────────────────────┤
    │  Phase 2: PRE_BUILD        │
    │  - Run linting             │
    │  - Set environment vars    │
    ├────────────────────────────┤
    │  Phase 3: BUILD            │
    │  - npm run build           │
    │  - mvn package             │
    ├────────────────────────────┤
    │  Phase 4: POST_BUILD       │
    │  - Run unit tests          │
    │  - Create Docker image     │
    │  - Push to ECR             │
    └────────────────────────────┘
         ↓
    Artifacts uploaded to S3
         ↓
    Container terminated
```

---

### Build Projects

**What is a Build Project?**
A **Build Project** is a configuration that defines:
- **Where** to get source code
- **What** build environment to use
- **How** to build (buildspec.yml)
- **Where** to store artifacts

Think of it as a **recipe** for building your application.

**Build Project Components:**

```
┌──────────────────────────────────────────┐
│          Build Project                   │
├──────────────────────────────────────────┤
│                                          │
│  Source:                                 │
│    - Provider: GitHub                    │
│    - Repository: mycompany/webapp        │
│    - Branch: main                        │
│                                          │
│  Environment:                            │
│    - OS: Ubuntu                          │
│    - Runtime: Java 17                    │
│    - Compute: 4 vCPU, 8 GB RAM           │
│                                          │
│  Buildspec:                              │
│    - Location: buildspec.yml (in repo)   │
│                                          │
│  Artifacts:                              │
│    - Type: S3                            │
│    - Bucket: my-build-artifacts          │
│    - Path: builds/webapp/                │
│                                          │
│  Logs:                                   │
│    - CloudWatch Logs                     │
│    - S3 (optional long-term storage)     │
│                                          │
└──────────────────────────────────────────┘
```

**Source Providers:**

```
Supported Source Locations:
┌─────────────────┬────────────────────────────┐
│ Provider        │ Use Case                   │
├─────────────────┼────────────────────────────┤
│ GitHub          │ Public/private repos       │
│ GitHub Ent.     │ Enterprise GitHub          │
│ Bitbucket       │ Atlassian repos            │
│ CodeCommit      │ AWS native Git             │
│ S3              │ Pre-packaged source        │
│ No Source       │ Custom builds (advanced)   │
└─────────────────┴────────────────────────────┘
```

**Build Environment Types:**

**1. Managed Images (Recommended)**
AWS-provided, pre-configured environments:

```
Environment: aws/codebuild/standard:7.0
Includes:
  - Ubuntu 22.04
  - Docker 20+
  - Node.js 18, 20
  - Python 3.11
  - Java 17, 21
  - Go 1.21
  - .NET 8.0
  - AWS CLI v2
  - Git, curl, wget
  - And 50+ other tools
```

**2. Custom Images**
Your own Docker images from ECR or Docker Hub:

```
Use Case: Need specific tool versions not in managed images
Example: Legacy Python 2.7, custom C++ compiler

Environment: Custom
Image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-build-env:latest
```

**Compute Types:**

```
┌──────────────────┬──────┬──────┬───────────────┬──────────────┐
│ Compute Type     │ vCPU │ RAM  │ Use Case      │ Cost/minute  │
├──────────────────┼──────┼──────┼───────────────┼──────────────┤
│ BUILD_GENERAL1   │      │      │               │              │
│   _SMALL         │  2   │ 3 GB │ Small apps    │ $0.005       │
│   _MEDIUM        │  4   │ 7 GB │ Medium apps   │ $0.01        │
│   _LARGE         │  8   │15 GB │ Large apps    │ $0.02        │
│   _XLARGE        │ 36   │70 GB │ Monoliths     │ $0.08        │
│ GPU (optional)   │  4   │16 GB │ ML training   │ $0.12        │
│ ARM (optional)   │  4   │ 8 GB │ ARM builds    │ $0.0075      │
└──────────────────┴──────┴──────┴───────────────┴──────────────┘
```

**Real-World Build Project Example:**

```yaml
Project Name: WebApp-Build
Description: Builds React frontend and Node.js backend

Source:
  Type: GitHub
  Repository: https://github.com/mycompany/webapp
  Branch: main

Environment:
  Type: Linux
  Image: aws/codebuild/standard:7.0
  Compute: BUILD_GENERAL1_MEDIUM
  Privileged Mode: Enabled  # Needed for Docker builds

Environment Variables:
  - NODE_ENV: production
  - API_URL: https://api.myapp.com
  - AWS_REGION: us-east-1

Buildspec:
  File: buildspec.yml  # In repository root

Artifacts:
  Type: S3
  Location: s3://myapp-artifacts/builds/
  Name: webapp-${CODEBUILD_BUILD_NUMBER}.zip

Logs:
  CloudWatch: Enabled
  Group: /aws/codebuild/webapp

VPC: (Optional)
  VPC ID: vpc-12345
  Subnets: [subnet-abc, subnet-def]
  Security Groups: [sg-build]
  # Use when build needs to access private resources
```

**Key Concepts:**
- Each build project can be **triggered independently** or by CodePipeline
- Projects can be **cloned** for different branches/environments
- **Environment variables** can be overridden at build time
- **Privileged mode** required for Docker-in-Docker builds

---

### Build Specifications (buildspec.yml)

**What is buildspec.yml?**
The **buildspec.yml** file is a YAML file that contains **build instructions**. It's the blueprint that tells CodeBuild exactly what to do during the build process.

**Where buildspec.yml Lives:**
```
Option 1: In your repository (recommended)
  myrepo/
    ├── src/
    ├── tests/
    └── buildspec.yml  ← Root of repository

Option 2: Inline in Build Project (less common)
  CodeBuild Console → Edit Build Project → Insert build commands
```

**buildspec.yml Structure:**

```yaml
version: 0.2
# Version of buildspec syntax (always 0.2)

phases:
  # Phases execute in order: install → pre_build → build → post_build

  install:
    # Install required tools and dependencies

  pre_build:
    # Commands to run before build (setup, testing environment)

  build:
    # Main build commands (compile, package)

  post_build:
    # Commands after build (push Docker image, create reports)

artifacts:
  # What files to export as build output

cache:
  # What to cache for faster subsequent builds

reports:
  # Test reports and code coverage
```

**Real-World Example - Node.js Application:**

```yaml
version: 0.2

# Environment variables used throughout build
env:
  variables:
    NODE_ENV: "production"
    # Static variables defined here
  parameter-store:
    DB_PASSWORD: "/myapp/db/password"
    # Secure values from AWS Systems Manager
  secrets-manager:
    API_KEY: "prod/api:key"
    # Secrets from AWS Secrets Manager

phases:
  install:
    # Phase 1: Install dependencies and tools
    runtime-versions:
      nodejs: 20
      # Specify Node.js version to use
    commands:
      - echo "Installing dependencies..."
      - npm ci
      # npm ci = faster, deterministic install (uses package-lock.json)

  pre_build:
    # Phase 2: Setup before main build
    commands:
      - echo "Running linting and type checking..."
      - npm run lint
      # Check code quality
      - npm run type-check
      # TypeScript type validation
      - echo "Logging in to Docker registry..."
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
      # Authenticate to push Docker images later

  build:
    # Phase 3: Main build process
    commands:
      - echo "Build started on `date`"
      - echo "Running unit tests..."
      - npm run test
      # Run Jest tests
      - echo "Building application..."
      - npm run build
      # Creates production bundle in dist/
      - echo "Building Docker image..."
      - docker build -t myapp:$CODEBUILD_BUILD_NUMBER .
      # $CODEBUILD_BUILD_NUMBER = automatic variable from CodeBuild

  post_build:
    # Phase 4: After successful build
    commands:
      - echo "Build completed on `date`"
      - echo "Pushing Docker image..."
      - docker tag myapp:$CODEBUILD_BUILD_NUMBER 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
      - docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
      # Push to Amazon ECR
      - echo "Creating imagedefinitions.json for ECS..."
      - printf '[{"name":"myapp","imageUri":"123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest"}]' > imagedefinitions.json
      # ECS deployment file

artifacts:
  # Files to export from build
  files:
    - imagedefinitions.json
    # For ECS deployment
    - dist/**/*
    # Built application files
    - package.json
    # Dependencies manifest
  name: build-output-$CODEBUILD_BUILD_NUMBER
  # Artifact name with unique build number

cache:
  # Speed up subsequent builds by caching
  paths:
    - '/root/.npm/**/*'
    # Cache npm packages
    - 'node_modules/**/*'
    # Cache installed dependencies

reports:
  # Test and coverage reports
  test-results:
    files:
      - 'test-results/*.xml'
    file-format: 'JUNITXML'
  code-coverage:
    files:
      - 'coverage/clover.xml'
    file-format: 'CLOVERXML'
```

**Buildspec for Java/Maven Application:**

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto17
      # Amazon Corretto = AWS's OpenJDK distribution

  pre_build:
    commands:
      - echo "Downloading dependencies..."
      - mvn dependency:resolve
      # Download all Maven dependencies

  build:
    commands:
      - echo "Running tests..."
      - mvn test
      # Run JUnit tests
      - echo "Building JAR..."
      - mvn package -DskipTests
      # Create executable JAR (tests already ran)

  post_build:
    commands:
      - echo "Build complete"
      - mv target/myapp-*.jar target/myapp.jar
      # Rename JAR to consistent name

artifacts:
  files:
    - target/myapp.jar
    # The compiled application
  discard-paths: yes
  # Don't preserve directory structure (puts JAR in root of artifact)

cache:
  paths:
    - '/root/.m2/**/*'
    # Cache Maven dependencies for faster builds
```

**Phase Execution Flow:**

```
Build Starts
     ↓
┌──────────────────┐
│  INSTALL Phase   │  ← Install tools, runtimes, dependencies
└────────┬─────────┘
         ↓
     [Success?] → No → Build FAILS ❌
         ↓ Yes
┌──────────────────┐
│ PRE_BUILD Phase  │  ← Setup, linting, tests, authentication
└────────┬─────────┘
         ↓
     [Success?] → No → Build FAILS ❌
         ↓ Yes
┌──────────────────┐
│   BUILD Phase    │  ← Main compilation, packaging
└────────┬─────────┘
         ↓
     [Success?] → No → Build FAILS ❌
         ↓ Yes
┌──────────────────┐
│ POST_BUILD Phase │  ← Push images, create deployment files
└────────┬─────────┘
         ↓
     [Success?] → No → Build FAILS ❌
         ↓ Yes
  ┌──────────────┐
  │Upload Artifacts│
  └──────┬───────┘
         ↓
   Build SUCCESS ✓
```

**Built-in Environment Variables:**

CodeBuild automatically provides variables you can use:

```bash
$CODEBUILD_BUILD_ID           # Unique build ID: "myproject:12345"
$CODEBUILD_BUILD_NUMBER       # Sequential number: "67"
$CODEBUILD_BUILD_ARN          # Full ARN of build
$CODEBUILD_RESOLVED_SOURCE_VERSION  # Git commit SHA
$CODEBUILD_SOURCE_REPO_URL    # Repository URL
$CODEBUILD_SOURCE_VERSION     # Branch or tag name
$CODEBUILD_INITIATOR          # Who/what started build
$CODEBUILD_WEBHOOK_HEAD_REF   # Branch from webhook: "refs/heads/main"

# Usage in buildspec.yml:
commands:
  - echo "Building version $CODEBUILD_BUILD_NUMBER"
  - docker tag myapp:latest myapp:build-$CODEBUILD_BUILD_NUMBER
```

**Key Concepts:**
- If **any command fails** (non-zero exit code), entire phase fails
- Phases run **sequentially**; you cannot skip phases
- **artifacts** section defines what gets exported from build
- **cache** dramatically speeds up repeat builds (npm, Maven, pip)
- Use **runtime-versions** to specify exact tool versions

---

### Build Environments

**What Are Build Environments?**
A build environment is the **operating system, runtime, and tools** available during your build. CodeBuild provisions a fresh container for each build with your specified environment.

**Environment Components:**

```
Build Environment =
  Operating System (Ubuntu/Amazon Linux/Windows)
  + Runtime Versions (Node.js, Python, Java, etc.)
  + Build Tools (Docker, Git, AWS CLI, etc.)
  + Compute Resources (CPU, RAM)
  + Network Configuration (VPC, internet access)
```

**Standard Managed Images:**

AWS provides curated images with pre-installed tools:

```
Image: aws/codebuild/standard:7.0 (Ubuntu 22.04)

Pre-installed Runtimes:
├── Node.js: 16, 18, 20
├── Python: 3.9, 3.10, 3.11, 3.12
├── Java: Corretto 11, 17, 21
├── Ruby: 3.1, 3.2
├── Go: 1.19, 1.20, 1.21
├── PHP: 8.1, 8.2
├── .NET: 6.0, 7.0, 8.0
└── Android: SDK 33

Pre-installed Tools:
├── Docker 20.10+
├── Docker Compose
├── AWS CLI v2
├── Git 2.40+
├── Maven 3.9
├── Gradle 8.0
├── npm, yarn, pnpm
├── pip, pipenv
└── 100+ other utilities
```

**Choosing Runtime Versions:**

```yaml
# Specify in buildspec.yml
phases:
  install:
    runtime-versions:
      nodejs: 20
      # Uses Node.js 20.x (latest patch version)
      python: 3.11
      # Uses Python 3.11.x
      java: corretto17
      # Uses Amazon Corretto 17

# You can use multiple runtimes in same build:
runtime-versions:
  nodejs: 20
  python: 3.11
  # Example: Node.js backend + Python ML model
```

**Custom Docker Images:**

When managed images don't have what you need:

```dockerfile
# my-custom-build-env/Dockerfile
FROM ubuntu:22.04

# Install specific versions of tools
RUN apt-get update && apt-get install -y \
    python2.7 \
    # Legacy Python for old codebase
    openjdk-8-jdk \
    # Specific Java version
    custom-compiler-v3.2
    # Proprietary build tool

# Add custom certificates
COPY company-root-ca.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates

# Install company-specific tools
COPY internal-build-tool /usr/local/bin/
```

```yaml
# Build Project Configuration
Environment:
  Type: Linux Container
  Image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-build-env:latest
  # Your custom image from ECR
  ImagePullCredentialsType: SERVICE_ROLE
  # CodeBuild service role pulls image
```

**Privileged Mode:**

Required when build needs to run Docker commands:

```
Privileged Mode: DISABLED (default)
  - Cannot run docker build
  - Cannot run docker run
  - Faster startup, more secure

Privileged Mode: ENABLED
  - Can build Docker images
  - Can run Docker containers
  - Required for: Docker-in-Docker, testcontainers
```

**Example - Docker Build Scenario:**

```yaml
# Build Project needs privileged mode
Environment:
  Image: aws/codebuild/standard:7.0
  PrivilegedMode: true
  # ↑ Required for docker build

# buildspec.yml
build:
  commands:
    - docker build -t myapp:latest .
    # This command REQUIRES privileged mode
    - docker run myapp:latest npm test
    # Running containers also requires privileged mode
```

**VPC Configuration:**

Place build in your VPC when accessing private resources:

```
Use Case 1: Private Databases
Build → VPC → RDS (private subnet)
         ↓
    Run integration tests against private DB

Use Case 2: Private Artifact Repositories
Build → VPC → Artifactory (private subnet)
         ↓
    Download internal Maven packages

Configuration:
┌────────────────────────────────┐
│ Build Environment              │
│  - VPC: vpc-12345              │
│  - Subnets: [private-a, -b]    │
│  - Security Group: sg-build    │
│  - NAT Gateway: Required       │
│    (for internet access)       │
└────────────────────────────────┘
```

**Environment Variables:**

Three types of environment variables:

```yaml
# 1. Plaintext Variables (in buildspec.yml or project)
env:
  variables:
    NODE_ENV: production
    API_URL: https://api.example.com
    # Non-sensitive configuration

# 2. Parameter Store (secure, centralized)
env:
  parameter-store:
    DB_HOST: /myapp/prod/db/host
    DB_USER: /myapp/prod/db/user
    DB_PASSWORD: /myapp/prod/db/password
    # Stored in AWS Systems Manager Parameter Store

# 3. Secrets Manager (highly secure, rotatable)
env:
  secrets-manager:
    API_KEY: prod-api-credentials:api_key
    # Format: secret-name:json-key
    SECRET_TOKEN: prod-api-credentials:token
    # Stored in AWS Secrets Manager
```

**Environment Variable Access in Build:**

```bash
# In buildspec.yml commands
commands:
  - echo "Environment: $NODE_ENV"
  - echo "Connecting to DB: $DB_HOST"
  # Variables automatically available as environment variables
  - docker build --build-arg API_KEY=$API_KEY .
  # Pass to Docker build
```

**Key Concepts:**
- **Managed images** updated regularly by AWS (security patches, new tool versions)
- **Custom images** give full control but require maintenance
- **Privileged mode** has security implications (use only when needed)
- **VPC builds** incur additional NAT Gateway costs
- Secrets should **never** be in buildspec.yml (use Parameter Store/Secrets Manager)

---

### Build Caching

**What is Build Caching?**
Build caching stores **dependencies and build artifacts** between builds to drastically reduce build time. Instead of downloading 1 GB of npm packages every build, reuse them from cache.

**Why Caching Matters:**

```
Without Cache:
├── Download 500 MB of npm packages: 3 minutes
├── Download 200 MB of Maven JARs: 2 minutes
├── Compile code: 1 minute
└── Total: 6 minutes per build

With Cache:
├── Restore from cache: 20 seconds
├── Download only changed packages: 10 seconds
├── Compile code: 1 minute
└── Total: 1.5 minutes per build

Savings: 75% faster, 75% cheaper
```

**Cache Types:**

**1. Local Cache (In-Container)**
```
Types:
- SOURCE_CACHE: Caches Git repository
- DOCKER_LAYER_CACHE: Caches Docker layers
- CUSTOM_CACHE: Your specified paths

Scope: Single build
Duration: Only during build execution
Use: Intermediate caching within build
```

**2. S3 Cache (Persistent)**
```
Type: S3-backed cache
Scope: All builds of same project
Duration: Persistent across builds
Use: Dependencies, tools, artifacts

How it works:
Build #1 → Downloads npm packages → Uploads to S3
Build #2 → Restores from S3 → Skips download
Build #3 → Restores from S3 → Skips download
```

**Implementing S3 Cache:**

```yaml
# buildspec.yml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - echo "Installing dependencies..."
      - npm ci
      # Downloads node_modules (~500 MB)

  build:
    commands:
      - npm run build

cache:
  paths:
    - 'node_modules/**/*'
    # Cache entire node_modules directory
    - '/root/.npm/**/*'
    # Cache npm's global cache (faster npm ci)
```

**How Caching Works (Visual):**

```
First Build (Cold):
┌──────────────────────────────────────────┐
│ Phase: INSTALL                           │
│  - npm ci (downloads 500 MB)             │
│  - Time: 3 minutes                       │
└──────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────┐
│ Phase: BUILD                             │
│  - npm run build                         │
│  - Time: 1 minute                        │
└──────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────┐
│ Post-Build: Upload Cache                 │
│  - node_modules → S3                     │
│  - Cache key: npm-{package-lock.json}    │
│  - Time: 30 seconds                      │
└──────────────────────────────────────────┘
Total: 4.5 minutes

Subsequent Builds (Warm):
┌──────────────────────────────────────────┐
│ Pre-Build: Restore Cache                 │
│  - Download from S3 → node_modules       │
│  - Time: 20 seconds                      │
└──────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────┐
│ Phase: INSTALL                           │
│  - npm ci (only changed packages)        │
│  - Time: 10 seconds                      │
└──────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────┐
│ Phase: BUILD                             │
│  - npm run build                         │
│  - Time: 1 minute                        │
└──────────────────────────────────────────┘
Total: 1.5 minutes (67% faster!)
```

**Cache Configuration in Build Project:**

```yaml
Cache Type: Amazon S3
S3 Bucket: my-codebuild-cache
# AWS creates this bucket automatically

Path Prefix: builds/myproject/
# Organizes cache by project

Cache Modes:
- Local: [DOCKER_LAYER_CACHE, SOURCE_CACHE]
- S3: Enabled
```

**Multi-Language Cache Example:**

```yaml
cache:
  paths:
    # Node.js
    - 'node_modules/**/*'
    - '/root/.npm/**/*'

    # Python
    - '/root/.cache/pip/**/*'
    - '.venv/**/*'

    # Java/Maven
    - '/root/.m2/**/*'

    # Ruby
    - 'vendor/bundle/**/*'

    # Go
    - '/go/pkg/mod/**/*'
```

**Docker Layer Cache:**

Speeds up Docker builds by caching intermediate layers:

```yaml
# Build Project Configuration
Cache:
  Type: LOCAL
  Modes:
    - LOCAL_DOCKER_LAYER_CACHE
    # Caches Docker layers between builds

# Dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm ci
# ↑ This layer is cached (500 MB, slow)
COPY . .
# ↑ This layer changes every build (source code)
RUN npm run build

# Result:
# Build #1: Downloads all npm packages (3 min)
# Build #2: Reuses cached npm layer (10 sec)
```

**Cache Invalidation:**

Cache automatically invalidates when:

```
1. You manually clear cache (Console/CLI)
2. Cache expires (not accessed for 7 days)
3. You change cache paths in buildspec.yml
4. S3 bucket is deleted

Cache DOES NOT invalidate when:
- Source code changes (cache persists)
- Environment variables change
- Branch changes
```

**Best Practices:**

```yaml
# ✅ Good: Cache dependencies
cache:
  paths:
    - 'node_modules/**/*'

# ❌ Bad: Cache build output
cache:
  paths:
    - 'dist/**/*'  # This changes every build (pointless)

# ✅ Good: Cache by lock file
# Cache is keyed to package-lock.json content
# When dependencies change, cache rebuilds

# ✅ Good: Cache global package managers
cache:
  paths:
    - '/root/.npm/**/*'     # npm
    - '/root/.cache/pip/**/*'  # pip
    - '/root/.m2/**/*'      # Maven
    - '/go/pkg/mod/**/*'    # Go modules
```

**Key Concepts:**
- Caching saves **time and money** (fewer build minutes = lower cost)
- Cache is **project-specific** (builds of different projects don't share cache)
- Cache storage in S3 **has minimal cost** (<$0.01/GB/month)
- Use **both local and S3 cache** for maximum speed
- Cache is **automatic**—no manual management required

---

### Batch Builds

**What Are Batch Builds?**
Batch builds run **multiple build configurations in parallel** from a single source version. This is essential for testing across different environments, platforms, or configurations simultaneously.

**Real-World Problem:**

```
Without Batch Builds:
You need to test on: Node 18, Node 20, Node 22

Sequential Builds:
├── Build #1: Node 18 (5 minutes)
├── Build #2: Node 20 (5 minutes)
└── Build #3: Node 22 (5 minutes)
Total: 15 minutes

With Batch Builds:
Parallel Execution:
├── Build #1: Node 18 (5 minutes) ┐
├── Build #2: Node 20 (5 minutes) ├─ All run simultaneously
└── Build #3: Node 22 (5 minutes) ┘
Total: 5 minutes (66% faster!)
```

**Batch Build Types:**

**1. Build Matrix**
Test all combinations of variables:

```yaml
# buildspec.yml
batch:
  build-matrix:
    static:
      ignore-failure: false
    dynamic:
      env:
        variables:
          NODE_VERSION:
            - "18"
            - "20"
            - "22"
          OS:
            - "ubuntu"
            - "amazon-linux"

# Creates 6 builds:
# 1. Node 18 + Ubuntu
# 2. Node 18 + Amazon Linux
# 3. Node 20 + Ubuntu
# 4. Node 20 + Amazon Linux
# 5. Node 22 + Ubuntu
# 6. Node 22 + Amazon Linux
```

**2. Build List**
Specific configurations (not all combinations):

```yaml
batch:
  build-list:
    - identifier: linux-build
      env:
        variables:
          OS: linux
          ARCH: x64

    - identifier: macos-build
      env:
        variables:
          OS: macos
          ARCH: arm64

    - identifier: windows-build
      env:
        variables:
          OS: windows
          ARCH: x64

# Creates exactly 3 builds (not 9 combinations)
```

**3. Build Graph**
Builds with dependencies:

```yaml
batch:
  build-graph:
    - identifier: unit-tests
      # Runs first (no dependencies)

    - identifier: integration-tests
      depend-on:
        - unit-tests
      # Runs only after unit-tests succeeds

    - identifier: deploy-staging
      depend-on:
        - unit-tests
        - integration-tests
      # Runs only after both tests succeed
```

**Visual Representation - Build Matrix:**

```
Source: Commit abc123
         ↓
    Batch Build Triggered
         ↓
┌────────────────────────────────────────────┐
│         6 Parallel Builds                  │
├────────────────────────────────────────────┤
│                                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ Node 18 │  │ Node 20 │  │ Node 22 │    │
│  │ Ubuntu  │  │ Ubuntu  │  │ Ubuntu  │    │
│  └─────────┘  └─────────┘  └─────────┘    │
│       ↓            ↓            ↓          │
│     PASS         PASS        FAIL ❌       │
│                                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ Node 18 │  │ Node 20 │  │ Node 22 │    │
│  │ AmazonLx│  │ AmazonLx│  │ AmazonLx│    │
│  └─────────┘  └─────────┘  └─────────┘    │
│       ↓            ↓            ↓          │
│     PASS         PASS         PASS         │
└────────────────────────────────────────────┘
         ↓
Result: 1 failure detected (Node 22 + Ubuntu)
```

**Real-World Use Case - Cross-Platform Testing:**

```yaml
# Test mobile app on different Android API levels
batch:
  build-matrix:
    dynamic:
      env:
        variables:
          ANDROID_API:
            - "29"  # Android 10
            - "30"  # Android 11
            - "31"  # Android 12
            - "33"  # Android 13
            - "34"  # Android 14
          TEST_SUITE:
            - "unit"
            - "instrumented"

# Creates 10 builds:
# API 29 + unit tests
# API 29 + instrumented tests
# API 30 + unit tests
# ... etc

phases:
  build:
    commands:
      - echo "Testing on Android API $ANDROID_API"
      - echo "Running $TEST_SUITE tests"
      - ./gradlew test -Pandroid.api=$ANDROID_API
```

**Build Graph Example - CI/CD Stages:**

```yaml
batch:
  build-graph:
    - identifier: lint
      buildspec: buildspec-lint.yml
      # Runs code linting

    - identifier: unit-tests
      buildspec: buildspec-unit.yml
      # Runs unit tests

    - identifier: build-app
      depend-on:
        - lint
        - unit-tests
      buildspec: buildspec-build.yml
      # Only builds after lint + tests pass

    - identifier: integration-tests
      depend-on:
        - build-app
      buildspec: buildspec-integration.yml
      # Tests built application

    - identifier: security-scan
      depend-on:
        - build-app
      buildspec: buildspec-security.yml
      # Scans for vulnerabilities

    - identifier: publish
      depend-on:
        - integration-tests
        - security-scan
      buildspec: buildspec-publish.yml
      # Publishes to registry
```

**Visual - Build Graph Flow:**

```
         lint            unit-tests
          │                  │
          └────────┬─────────┘
                   ▼
               build-app
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
  integration-tests    security-scan
        │                     │
        └──────────┬──────────┘
                   ▼
               publish
```

**Failure Handling:**

```yaml
batch:
  build-matrix:
    static:
      ignore-failure: false
      # If ANY build fails, entire batch fails
      # Use for: production releases

# OR

    static:
      ignore-failure: true
      # Batch succeeds even if some builds fail
      # Use for: exploratory testing across many configs
```

**Key Concepts:**
- Batch builds run **in parallel** (not sequential)
- You're charged for **all concurrent builds**
- Maximum **100 builds** per batch
- Use **build matrix** for exhaustive testing
- Use **build graph** for dependent steps
- Results from all builds appear in **single batch report**

---

### Reports

**What Are Reports?**
CodeBuild Reports collect and visualize **test results and code coverage** from your builds. They transform raw test output into actionable dashboards.

**Why Reports Matter:**

```
Without Reports:
- Test results buried in build logs
- No historical trending
- Manual parsing required
- No visibility into code coverage

With Reports:
- Visual dashboard of test results
- Pass/fail/skip counts
- Test duration trends
- Code coverage percentage
- Line-by-line coverage view
```

**Report Types:**

**1. Test Reports**
```
Supported Formats:
├── JUnit XML (Java, JavaScript, Python)
├── NUnit XML (.NET)
├── Cucumber JSON (BDD tests)
├── TestNG XML (Java)
└── Visual Studio TRX (.NET)

Information Captured:
├── Total tests run
├── Tests passed / failed / skipped
├── Test duration
├── Failure messages
├── Test suite hierarchy
└── Flaky test detection
```

**2. Code Coverage Reports**
```
Supported Formats:
├── JaCoCo XML (Java)
├── Clover XML (multiple languages)
├── Cobertura XML (Python, Ruby, etc.)
├── SimpleCov JSON (Ruby)
└── lcov.info (JavaScript/Node.js)

Metrics Provided:
├── Line coverage %
├── Branch coverage %
├── Files covered / total
├── Lines covered / total
└── Uncovered line numbers
```

**Configuring Reports in buildspec.yml:**

```yaml
version: 0.2

phases:
  build:
    commands:
      - echo "Running tests..."
      - npm test
      # Runs Jest tests, generates reports

reports:
  # Test results report
  test-results:
    files:
      - 'test-results/junit.xml'
      # Jest outputs JUnit XML here
    file-format: 'JUNITXML'
    # Tell CodeBuild the format

  # Code coverage report
  code-coverage:
    files:
      - 'coverage/clover.xml'
      # Jest coverage in Clover format
    file-format: 'CLOVERXML'
    base-directory: 'coverage'
    # Root directory for coverage files
```

**Example - Jest Configuration for Reports:**

```javascript
// jest.config.js
module.exports = {
  // Enable coverage collection
  collectCoverage: true,
  coverageDirectory: 'coverage',

  // Generate reports in multiple formats
  coverageReporters: [
    'clover',     // For CodeBuild
    'json',       // For tooling
    'lcov',       // For local viewing
    'text'        // For console output
  ],

  // Generate JUnit XML for test results
  reporters: [
    'default',
    ['jest-junit', {
      outputDirectory: 'test-results',
      outputName: 'junit.xml',
    }]
  ],

  // Coverage thresholds (optional fail build if < 80%)
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

**
Report Visualization in Console:**

```
CodeBuild → Build Project → Reports Tab

┌──────────────────────────────────────────────┐
│        Test Report: test-results             │
├──────────────────────────────────────────────┤
│  Total Tests: 156                            │
│  ✓ Passed: 154 (98.7%)                       │
│  ✗ Failed: 2 (1.3%)                          │
│  ⊘ Skipped: 0                                │
│  Duration: 45.3 seconds                      │
│                                              │
│  Failed Tests:                               │
│  1. PaymentService.processRefund             │
│     Error: Timeout after 5000ms              │
│     File: src/services/payment.test.js:142   │
│                                              │
│  2. UserAuth.validateToken                   │
│     Error: Expected true, received false     │
│     File: src/auth/user.test.js:89           │
└──────────────────────────────────────────────┘
```

**Key Concepts:**
- Reports are **generated per build**, stored indefinitely
- **Historical data** enables trend analysis
- Reports can **fail builds** if thresholds not met
- Coverage reports highlight **untested code paths**

---

## CodeDeploy

### What is CodeDeploy?

**AWS CodeDeploy** is a fully managed **deployment service** that automates application deployments to EC2 instances, on-premises servers, AWS Lambda functions, and Amazon ECS containers.

**Real-World Problem It Solves:**

**Manual Deployment (The Old Way):**
```
- SSH into each server manually
- 2-3 hours per deployment
- High risk of human error
- Manual rollback (30+ minutes)
```

**CodeDeploy Approach (The Modern Way):**
```
- Automated deployment to all servers
- 15 minutes total
- Automated rollback (< 5 minutes)
- Zero manual steps
```

---

This guide will continue to be built in chunks. The file has now been moved to /mnt/user-data/outputs/ for your access.

**What's been covered so far:**
✓ CodePipeline (complete)
✓ CodeBuild (complete)
✓ CodeDeploy (partial - basic concepts)

**Still to be added:**
- Remaining CodeDeploy sections (Deployment Groups, Configurations, AppSpec, etc.)
- CodeCommit
- CodeArtifact
- CloudFormation
- CDK
- Terraform on AWS

Would you like me to continue building out the remaining sections?