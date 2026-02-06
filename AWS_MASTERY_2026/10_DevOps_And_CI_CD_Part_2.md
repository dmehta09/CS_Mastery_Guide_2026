# AWS DevOps & CI/CD - Comprehensive Guide (Part 2)

## Continuation: CodeDeploy (Detailed Sections)

### Deployment Groups

**What is a Deployment Group?**
A Deployment Group defines **WHERE your application will be deployed**—the specific collection of EC2 instances, Lambda functions, or ECS services that receive the new version.

**Why Deployment Groups Matter:**

```
Single Application: MyWebApp
Multiple Deployment Groups:

Development Group
  └── Targets: 2 EC2 instances (dev environment)
       Strategy: All-at-once (fast, can have downtime)

Staging Group
  └── Targets: 5 EC2 instances (staging environment)
       Strategy: Rolling 50% (moderate safety)

Production Group
  └── Targets: 20 EC2 instances (live users)
       Strategy: Blue/Green (zero downtime, max safety)
```

**How Targets Are Selected:**

**1. EC2 Tag-Based Selection**
```yaml
Deployment Group: Production
Target Selection:
  Type: EC2 Instances
  Criteria: Tags
  Tags:
    - Key: Environment
      Value: Production
    - Key: Application
      Value: WebApp

# Result: Any EC2 instance with BOTH tags is included
# Dynamic: New instances with tags auto-join group
```

**Visual Representation:**

```
AWS Account with EC2 Instances:

Instance 1: Tags [Environment: Production, Application: WebApp] ✓ Included
Instance 2: Tags [Environment: Production, Application: API]    ✗ Not included
Instance 3: Tags [Environment: Staging, Application: WebApp]    ✗ Not included
Instance 4: Tags [Environment: Production, Application: WebApp] ✓ Included
Instance 5: Tags [Environment: Production, Application: WebApp] ✓ Included

Deployment Group "Production" targets: [Instance 1, 4, 5]
```

**2. Auto Scaling Group Selection**
```yaml
Deployment Group: Production-ASG
Target Selection:
  Type: Auto Scaling Group
  ASG Name: WebApp-Production-ASG

# Result: All instances in ASG (current + future)
# Dynamic: ASG scales → new instances automatically get deployment
```

**Auto Scaling Group Behavior:**

```
Scenario: ASG scales up during deployment

Initial state:
ASG has 3 instances → CodeDeploy deployment starts
    ↓
During deployment:
ASG scales to 5 instances (new instances launch)
    ↓
CodeDeploy behavior:
- Continues deploying to original 3 instances
- New instances (4 & 5) launch with OLD version
- Must trigger new deployment to update new instances

Alternative with "Auto Scaling Groups" setting:
- New instances automatically get latest deployed revision
- No manual re-deployment needed
```

**3. On-Premises Instances**
```yaml
Deployment Group: Hybrid-Deployment
Target Selection:
  Type: On-Premises
  Instance Names:
    - on-prem-server-01
    - on-prem-server-02
    - data-center-web-03

# Requires: CodeDeploy Agent installed on on-prem servers
# Registration: Instances must be registered with CodeDeploy
```

**Deployment Group Configuration Example:**

```yaml
Application: MyWebApp
Deployment Group: Production

# Target Selection
Instances:
  Tags:
    Environment: production
    Role: webserver
  # Selects all instances matching BOTH tags

# Load Balancer Integration
Load Balancer:
  Type: Application Load Balancer
  Target Group: webapp-prod-tg
  # CodeDeploy manages instance registration/deregistration

# Deployment Configuration
Config: CodeDeployDefault.HalfAtATime
# Deploys to 50% of instances at a time

# Health Check
Health Check:
  Type: ELB
  Timeout: 300 seconds
  # Wait up to 5 min for instance to become healthy

# Rollback Configuration
Auto Rollback:
  Enabled: true
  Trigger On:
    - Deployment Failure
    - Health Check Failure
  # Auto-rollback to previous version

# Alarms (Optional)
CloudWatch Alarms:
  - CPUUtilization-High
  - 5xx-Errors-Spike
  # Stop deployment if alarms trigger
```

**Load Balancer Integration:**

CodeDeploy integrates with load balancers for zero-downtime deployments:

```
Before Deployment:
ALB → Target Group
       ├── Instance 1 (healthy) ← serving traffic
       ├── Instance 2 (healthy) ← serving traffic
       └── Instance 3 (healthy) ← serving traffic

During Deployment (Instance 1):
ALB → Target Group
       ├── Instance 1 (draining) ← being updated
       ├── Instance 2 (healthy) ← serving 100% traffic
       └── Instance 3 (healthy) ← serving 100% traffic

Steps for Instance 1:
1. De-register Instance 1 from target group
2. Wait for connection draining (default 300 sec)
3. Deploy new version to Instance 1
4. Run lifecycle hooks (ApplicationStop → ValidateService)
5. Re-register Instance 1 to target group
6. Wait for healthy status (health checks pass)
7. Move to Instance 2

After Deployment:
ALB → Target Group
       ├── Instance 1 (healthy) ← new version
       ├── Instance 2 (healthy) ← new version
       └── Instance 3 (healthy) ← new version
```

**Multiple Deployment Groups - Real Use Case:**

```
Application: E-Commerce Platform

Group 1: Canary-Deployment
  Purpose: Test with 5% of users first
  Targets: 1 instance behind weighted target group
  Strategy: All-at-once (single instance)
  Monitoring: Intensive CloudWatch monitoring
  Duration: 1 hour observation

Group 2: Production-Primary-US
  Purpose: Main production fleet (US region)
  Targets: 15 instances (us-east-1)
  Strategy: Rolling 33% (5 instances at a time)
  Load Balancer: ALB-US-Production
  Monitoring: Standard CloudWatch alarms

Group 3: Production-Europe
  Purpose: European users
  Targets: 10 instances (eu-west-1)
  Strategy: Blue/Green (zero downtime)
  Load Balancer: ALB-EU-Production
  Monitoring: Standard CloudWatch alarms

Deployment Flow:
Step 1: Deploy to Canary-Deployment
  - Monitor for 1 hour
  - Check error rates, latency, user feedback

Step 2: If canary healthy → Deploy to Production-Primary-US
  - Rolling deployment (5 instances at a time)
  - Monitor during entire deployment

Step 3: If US deployment successful → Deploy to Production-Europe
  - Blue/Green deployment (instant switch)
  - Keep blue environment for 24h rollback window
```

**Deployment Group with Multiple Criteria:**

```yaml
# Advanced targeting
Deployment Group: Web-Tier-Production

Tag Sets (OR logic between sets, AND within set):
  Set 1:
    - Environment: production
    - Tier: web
    - Version: v2
  # Matches instances with ALL three tags

  Set 2:
    - Environment: production
    - Tier: api
    - Service: gateway
  # Matches instances with ALL three tags

# Result: Instances matching (Set 1) OR (Set 2) are targeted
# Allows deploying to multiple logical groups simultaneously
```

**Key Concepts:**
- Deployment groups are **environment-specific** (dev, staging, prod)
- **Tags** enable dynamic targeting (automatically includes new instances)
- **Load balancer integration** provides zero-downtime deployments
- Can have **multiple deployment groups** per application
- Each group can have different deployment strategies
- **Auto Scaling Groups** ensure new instances get latest revision

---

### Deployment Configurations

**What Are Deployment Configurations?**
Deployment Configurations define **HOW FAST** and **IN WHAT PATTERN** CodeDeploy rolls out your application. They control the speed vs. safety tradeoff.

**Pre-Defined Configurations (EC2/On-Premises):**

```
┌─────────────────────────────┬─────────────┬──────────────┬──────────────┐
│ Configuration               │ Speed       │ Safety       │ Use Case     │
├─────────────────────────────┼─────────────┼──────────────┼──────────────┤
│ CodeDeployDefault.          │ Fastest     │ Lowest       │ Dev/Test     │
│   AllAtOnce                 │ (1 wave)    │ (all at risk)│ environments │
│                             │             │              │              │
│ CodeDeployDefault.          │ Moderate    │ Moderate     │ Staging,     │
│   HalfAtATime               │ (2 waves)   │ (50% at risk)│ Low-traffic  │
│                             │             │              │              │
│ CodeDeployDefault.          │ Slowest     │ Highest      │ Production,  │
│   OneAtATime                │ (N waves)   │ (10% at risk)│ Mission-     │
│                             │             │              │ critical     │
└─────────────────────────────┴─────────────┴──────────────┴──────────────┘
```

**Detailed Behavior:**

**1. AllAtOnce** (Fastest, Risky)
```
10 Instances
    ↓
┌────────────────────────────────────┐
│  Deploy to ALL 10 simultaneously   │
│  [1][2][3][4][5]                   │
│  [6][7][8][9][10]                  │
│  All deploying at once             │
└────────────────────────────────────┘
    ↓
Timeline:
  0:00 - Start deployment to all 10
  0:05 - All instances updated
    ↓
Success: 5 minutes total
Failure: All 10 instances broken (100% outage) ❌

Use Cases:
✓ Development environment (fast iteration)
✓ Off-hours maintenance window
✓ When downtime is acceptable
✓ Non-production workloads

Risk Level: HIGH (entire fleet at risk)
```

**2. HalfAtATime** (Balanced)
```
10 Instances
    ↓
Wave 1: Deploy to 5 instances (50%)
┌────────────────────────────────────┐
│  [1][2][3][4][5] ← deploying       │
│  [6][7][8][9][10] ← old version    │
└────────────────────────────────────┘
    ↓
Health checks pass for Wave 1
    ↓
Wave 2: Deploy to remaining 5
┌────────────────────────────────────┐
│  [1][2][3][4][5] ← new version ✓   │
│  [6][7][8][9][10] ← deploying      │
└────────────────────────────────────┘
    ↓
Timeline:
  0:00 - Deploy to instances 1-5
  0:05 - Health checks pass
  0:06 - Deploy to instances 6-10
  0:11 - Deployment complete
    ↓
Success: 11 minutes total
Failure: Max 50% instances affected

Use Cases:
✓ Staging environment
✓ Moderate-risk production
✓ Balanced speed and safety
✓ Medium-traffic applications

Risk Level: MODERATE (half fleet at risk)
```

**3. OneAtATime** (Safest, Slowest)
```
10 Instances
    ↓
Wave 1: Instance 1
[1]← deploying
[2][3][4][5][6][7][8][9][10]← old version
    ↓
Health check passes
    ↓
Wave 2: Instance 2
[1]← new version ✓
[2]← deploying
[3][4][5][6][7][8][9][10]← old version
    ↓
Health check passes
    ↓
... (continues for each instance)
    ↓
Timeline:
  0:00 - Deploy to instance 1
  0:03 - Health check passes
  0:03 - Deploy to instance 2
  0:06 - Health check passes
  ... (repeat)
  0:30 - All instances updated
    ↓
Success: 30 minutes total
Failure: Max 1 instance affected (10%)

Use Cases:
✓ Critical production systems
✓ Financial applications
✓ Healthcare systems
✓ Maximum safety required

Risk Level: LOW (one instance at risk)
```

**Custom Deployment Configurations:**

Create your own deployment speed with precise control:

**Example 1: Deploy 25% at a time**
```yaml
Custom Configuration: Deploy25Percent

Minimum Healthy Hosts:
  Type: FLEET_PERCENT
  Value: 75
  # 75% must stay healthy during deployment
  # Therefore, deploy to 25% at a time

Mathematical breakdown with 12 instances:
  Total instances: 12
  Minimum healthy: 75% = 9 instances
  Can update at once: 12 - 9 = 3 instances (25%)

Deployment waves:
  Wave 1: Deploy to 3 instances (25%)
    - 9 instances remain healthy (75%) ✓
  Wave 2: Deploy to next 3 instances (25%)
    - 9 instances remain healthy ✓
  Wave 3: Deploy to next 3 instances (25%)
    - 9 instances remain healthy ✓
  Wave 4: Deploy to final 3 instances (25%)
    - 9 instances remain healthy ✓
```

**Example 2: Ensure minimum instance count**
```yaml
Custom Configuration: Keep10Healthy

Minimum Healthy Hosts:
  Type: HOST_COUNT
  Value: 10
  # Always keep at least 10 specific instances healthy

Example with 15 instances:
  Wave 1: Deploy to 5 instances (33%)
    - 10 instances remain healthy ✓ (15 total - 5 deploying)
  Wave 2: Deploy to next 5 instances (33%)
    - 10 instances remain healthy ✓
  Wave 3: Deploy to final 5 instances (33%)
    - 10 instances remain healthy ✓

Use case: SLA requires minimum 10 instances always available
```

**Deployment Configuration Visualization:**

```
Scenario: 8 instances, HalfAtATime configuration

Timeline Visualization:

0:00 ┤ [1][2][3][4][5][6][7][8] ← All running v1.0
     │ 100% capacity available
     │
0:01 ┤ [▓][▓][▓][▓][5][6][7][8]
     │  ↑ deploying v2.0
     │ 50% capacity (instances 5-8 serve all traffic)
     │
0:05 ┤ [✓][✓][✓][✓][5][6][7][8]
     │  ↑ v2.0 healthy
     │  [health checks passed]
     │ 100% capacity restored (mixed versions)
     │
0:06 ┤ [✓][✓][✓][✓][▓][▓][▓][▓]
     │               ↑ deploying v2.0
     │ 50% capacity (instances 1-4 serve all traffic)
     │
0:10 ┤ [✓][✓][✓][✓][✓][✓][✓][✓]
     │  ↑ All running v2.0
     │  Deployment Complete! ✓
     │ 100% capacity, all v2.0
```

**Lambda Deployment Configurations:**

Lambda uses **traffic shifting** instead of instance-by-instance deployment:

**1. Canary Deployments**
```
CodeDeployDefault.LambdaCanary10Percent5Minutes

Behavior:
  Minute 0: 10% of traffic → new version
           90% of traffic → old version
           [Monitor for 5 minutes]

  Minute 5: If healthy → 100% traffic → new version
           If unhealthy → Auto-rollback to old version

Traffic distribution:
  Old (v1): ████████████████████ 100%
  New (v2):

  ↓ (shift 10%)

  Old (v1): ██████████████████ 90%
  New (v2): ██ 10%
            [Canary period - watching for errors]

  ↓ (after 5 min, if healthy)

  Old (v1):
  New (v2): ████████████████████ 100%

Other canary options:
- LambdaCanary10Percent10Minutes (10% for 10 min)
- LambdaCanary10Percent15Minutes (10% for 15 min)
- LambdaCanary10Percent30Minutes (10% for 30 min)
```

**2. Linear Deployments**
```
CodeDeployDefault.LambdaLinear10PercentEvery1Minute

Behavior:
  Every 1 minute, shift another 10% of traffic
  Takes 10 minutes total for full migration

Traffic distribution over time:
  Minute 0:
    Old: ████████████████████ 100%
    New:

  Minute 1:
    Old: ██████████████████ 90%
    New: ██ 10%

  Minute 2:
    Old: ████████████████ 80%
    New: ████ 20%

  Minute 3:
    Old: ██████████████ 70%
    New: ██████ 30%

  ... continues ...

  Minute 10:
    Old:
    New: ████████████████████ 100%

Other linear options:
- LambdaLinear10PercentEvery2Minutes
- LambdaLinear10PercentEvery3Minutes
- LambdaLinear10PercentEvery10Minutes
```

**3. All-At-Once**
```
CodeDeployDefault.LambdaAllAtOnce

Behavior:
  Immediately switch 100% traffic to new version
  No gradual shift, instant cutover

Traffic distribution:
  Old: ████████████████████ 100%
  New:

  ↓ (instant switch)

  Old:
  New: ████████████████████ 100%

Use case: Non-critical functions, dev/test
```

**ECS Deployment Configurations:**

ECS uses similar patterns to EC2 but with task-based deployments:

```
CodeDeployDefault.ECSLinear10PercentEvery1Minute
- Shift 10% of traffic every 1 minute
- From old task set → new task set
- Takes 10 minutes for complete migration

CodeDeployDefault.ECSCanary10Percent5Minutes
- 10% traffic to new tasks for 5 minutes
- Then 100% if healthy

CodeDeployDefault.ECSAllAtOnce
- Instant switch to new task set
```

**Choosing the Right Configuration:**

```
Decision Matrix:

High Traffic + Mission Critical:
  → CodeDeployDefault.OneAtATime (EC2)
  → LambdaCanary10Percent5Minutes (Lambda)
  Example: Payment processing, user authentication

Moderate Traffic + Standard Production:
  → CodeDeployDefault.HalfAtATime (EC2)
  → LambdaLinear10PercentEvery3Minutes (Lambda)
  Example: Web applications, APIs

Low Risk + Development:
  → CodeDeployDefault.AllAtOnce
  Example: Internal tools, staging environments

Custom Requirements:
  → Create custom configuration
  Example: SLA requires 15 instances minimum healthy
```

**Configuration with Alarms:**

```yaml
Deployment Configuration: HalfAtATime
CloudWatch Alarms:
  - HTTPCode_Target_5XX_Count > 10
  - TargetResponseTime > 2000ms
  - UnhealthyHostCount > 1

Behavior during deployment:
  Wave 1: Deploy to 50% of instances
    ↓
  Monitor alarms during wave
    ↓
  If alarm triggers → STOP deployment
    ↓
  Automatic rollback initiated
    ↓
  Deployment fails, instances restored

Example timeline:
  0:00 - Start deployment (Wave 1)
  0:03 - Alarm: 5XX errors > 10
  0:03 - Stop deployment immediately
  0:04 - Rollback Wave 1 instances
  0:07 - All instances back to old version
  Result: Prevented 50% fleet from bad deployment
```

**Key Concepts:**
- **Slower deployments** = safer but take longer
- **Faster deployments** = riskier but complete quickly
- **Custom configurations** allow precise control (any percentage)
- **Lambda configs** use traffic shifting (gradual cutover)
- **Minimum healthy hosts** ensures availability during deployment
- Always test deployment configs in **staging first**

---

### AppSpec File

**What is the AppSpec File?**
The **AppSpec (Application Specification)** file is a YAML or JSON file that tells CodeDeploy:
1. **What files** to copy to the target
2. **What scripts** to run during deployment
3. **When** to run those scripts (lifecycle hooks)

Think of it as the **instruction manual** for your deployment.

**AppSpec File Location:**

```
Your Application Package:
myapp.zip
├── appspec.yml          ← Must be in root!
├── index.js
├── package.json
├── config/
│   └── production.json
└── scripts/
    ├── stop_server.sh
    ├── install_dependencies.sh
    └── start_server.sh
```

**Basic AppSpec Structure (EC2/On-Premises):**

```yaml
version: 0.0
# AppSpec version (always 0.0 for EC2/On-Premises)

os: linux
# Target OS: linux or windows

files:
  # What files to copy and where

permissions:
  # File permissions after copy (Linux only)

hooks:
  # Lifecycle scripts to run at specific points
```

**Real-World Example - Node.js Web Application:**

```yaml
version: 0.0
os: linux

files:
  # Source → Destination mapping
  - source: /
    destination: /var/www/myapp
    # Copies ALL files from deployment package root
    # to /var/www/myapp on target instance

  # Can specify specific files/directories:
  - source: /config/production.json
    destination: /var/www/myapp/config/
    # Copy only production config file

  - source: /public
    destination: /var/www/myapp/public
    # Copy entire public directory

permissions:
  # Set file permissions after copying
  - object: /var/www/myapp
    owner: webapp
    group: webapp
    mode: 755
    type:
      - directory
    # All directories: rwxr-xr-x (755)

  - object: /var/www/myapp
    owner: webapp
    group: webapp
    mode: 644
    type:
      - file
    # All files: rw-r--r-- (644)

  - object: /var/www/myapp/scripts
    owner: webapp
    group: webapp
    mode: 755
    type:
      - file
    # Script files: rwxr-xr-x (755, executable)

hooks:
  # Lifecycle event hooks (run in this order)

  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 60
      runas: root
      # Stop old application version before deployment
      # Timeout: Max 60 seconds for script to complete
      # Run as root user

  BeforeInstall:
    - location: scripts/backup_old_version.sh
      timeout: 120
      runas: root
      # Backup current version before new files copied

    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: webapp
      # Install system dependencies (5 min timeout)

  AfterInstall:
    - location: scripts/npm_install.sh
      timeout: 600
      runas: webapp
      # Install Node.js packages after files copied
      # Longer timeout for npm install (10 min)

  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 60
      runas: webapp
      # Start new application version

  ValidateService:
    - location: scripts/health_check.sh
      timeout: 120
      runas: webapp
      # Verify deployment succeeded
      # If this fails, deployment marked as FAILED
```

**Lifecycle Hook Order (Complete Flow):**

```
CodeDeploy Agent starts deployment
         ↓
┌───────────────────────────────────────────────────┐
│  1. ApplicationStop                               │
│     Purpose: Stop old version gracefully          │
│     Script: scripts/stop_server.sh                │
│     Example: systemctl stop myapp, pm2 stop       │
└───────────────────────────────────────────────────┘
         ↓
┌───────────────────────────────────────────────────┐
│  2. DownloadBundle                                │
│     Purpose: Download application from S3/GitHub  │
│     Action: Automatic (CodeDeploy managed)        │
│     Note: No custom scripts allowed               │
└───────────────────────────────────────────────────┘
         ↓
┌───────────────────────────────────────────────────┐
│  3. BeforeInstall                                 │
│     Purpose: Prepare for installation             │
│     Script: scripts/backup_old_version.sh         │
│     Example: Backup files, create directories     │
└───────────────────────────────────────────────────┘
         ↓
┌───────────────────────────────────────────────────┐
│  4. Install                                       │
│     Purpose: Copy files to destination            │
│     Action: Automatic (per 'files' section)       │
│     Note: Applies 'permissions' section           │
└───────────────────────────────────────────────────┘
         ↓
┌───────────────────────────────────────────────────┐
│  5. AfterInstall                                  │
│     Purpose: Configure newly installed files      │
│     Script: scripts/npm_install.sh                │
│     Example: npm install, chmod, config setup     │
└───────────────────────────────────────────────────┘
         ↓
┌───────────────────────────────────────────────────┐
│  6. ApplicationStart                              │
│     Purpose: Start new application version        │
│     Script: scripts/start_server.sh               │
│     Example: systemctl start, pm2 start           │
└───────────────────────────────────────────────────┘
         ↓
┌───────────────────────────────────────────────────┐
│  7. ValidateService                               │
│     Purpose: Verify deployment success            │
│     Script: scripts/health_check.sh               │
│     Example: curl health endpoint, check logs     │
│     CRITICAL: Failure here triggers rollback      │
└───────────────────────────────────────────────────┘
         ↓
      Deployment Complete ✓
```

**Example Lifecycle Scripts:**

**scripts/stop_server.sh:**
```bash
#!/bin/bash
# ApplicationStop hook - Stop the application gracefully

echo "=== ApplicationStop Hook ==="
echo "Stopping Node.js application..."

# Check if application is running
if pgrep -f "node server.js" > /dev/null; then
  echo "Application process found, initiating graceful shutdown..."

  # Send SIGTERM for graceful shutdown (allows cleanup)
  pkill -TERM -f "node server.js"

  # Wait up to 30 seconds for graceful shutdown
  timeout=30
  while pgrep -f "node server.js" > /dev/null && [ $timeout -gt 0 ]; do
    echo "Waiting for graceful shutdown... ($timeout seconds remaining)"
    sleep 1
    ((timeout--))
  done

  # Force kill if still running after timeout
  if pgrep -f "node server.js" > /dev/null; then
    echo "WARNING: Graceful shutdown timeout, force killing process..."
    pkill -9 -f "node server.js"
  fi

  echo "Application stopped successfully"
else
  echo "Application not running (first deployment or already stopped)"
fi

exit 0  # Always succeed - don't fail if app not running
```

**scripts/backup_old_version.sh:**
```bash
#!/bin/bash
# BeforeInstall hook - Backup current version

echo "=== BeforeInstall Hook ==="

# Check if application directory exists
if [ -d "/var/www/myapp" ]; then
  echo "Backing up current version..."

  # Create backup directory
  BACKUP_DIR="/var/backups/myapp"
  mkdir -p $BACKUP_DIR

  # Create timestamped backup
  timestamp=$(date +%Y%m%d_%H%M%S)
  backup_file="$BACKUP_DIR/backup_$timestamp.tar.gz"

  tar -czf $backup_file /var/www/myapp
  echo "Backup created: $backup_file"

  # Keep only last 5 backups (cleanup old ones)
  cd $BACKUP_DIR
  ls -t | tail -n +6 | xargs -r rm -f
  echo "Old backups cleaned (kept last 5)"
else
  echo "No existing installation found (first deployment)"
fi

# Create application directory structure
echo "Creating directory structure..."
mkdir -p /var/www/myapp
mkdir -p /var/www/myapp/logs
mkdir -p /var/www/myapp/tmp
mkdir -p /var/www/myapp/uploads

# Set ownership (prepare for file copy)
chown -R webapp:webapp /var/www/myapp

echo "BeforeInstall completed successfully"
exit 0
```

**scripts/npm_install.sh:**
```bash
#!/bin/bash
# AfterInstall hook - Install Node.js dependencies

echo "=== AfterInstall Hook ==="

cd /var/www/myapp

# Install Node.js dependencies
echo "Installing npm dependencies..."
npm ci --production
# npm ci = clean install, uses package-lock.json, faster and more reliable

# Verify installation
if [ $? -eq 0 ]; then
  echo "Dependencies installed successfully"
else
  echo "ERROR: npm install failed"
  exit 1  # Fail deployment if dependencies can't be installed
fi

# Run database migrations (if applicable)
if [ -f "migrate.js" ]; then
  echo "Running database migrations..."
  node migrate.js

  if [ $? -eq 0 ]; then
    echo "Migrations completed successfully"
  else
    echo "ERROR: Database migrations failed"
    exit 1  # Fail deployment if migrations fail
  fi
fi

# Build assets (if needed)
if grep -q '"build"' package.json; then
  echo "Building production assets..."
  npm run build
fi

# Set correct permissions
echo "Setting file permissions..."
find /var/www/myapp -type f -exec chmod 644 {} \;
find /var/www/myapp -type d -exec chmod 755 {} \;
chmod +x /var/www/myapp/scripts/*.sh

# Fetch secrets from AWS Systems Manager Parameter Store
echo "Fetching application secrets..."
aws ssm get-parameter \
  --name /myapp/prod/env \
  --with-decryption \
  --region us-east-1 \
  --query 'Parameter.Value' \
  --output text > /var/www/myapp/.env

echo "AfterInstall completed successfully"
exit 0
```

**scripts/start_server.sh:**
```bash
#!/bin/bash
# ApplicationStart hook - Start the application

echo "=== ApplicationStart Hook ==="

cd /var/www/myapp

# Start application with PM2 (process manager)
echo "Starting application with PM2..."

# PM2 ecosystem file configuration
pm2 start ecosystem.config.js --env production

# Verify PM2 started successfully
if [ $? -eq 0 ]; then
  echo "Application started successfully"

  # Save PM2 process list (persists across reboots)
  pm2 save

  # Configure PM2 to start on system boot
  pm2 startup systemd -u webapp --hp /home/webapp

  # Display PM2 status
  pm2 list
else
  echo "ERROR: Failed to start application"
  exit 1  # Fail deployment if app can't start
fi

exit 0
```

**scripts/health_check.sh:**
```bash
#!/bin/bash
# ValidateService hook - Verify deployment success

echo "=== ValidateService Hook ==="

# Wait for application to fully initialize
echo "Waiting for application to start..."
sleep 15

# Configuration
HEALTH_URL="http://localhost:3000/health"
MAX_ATTEMPTS=10
RETRY_DELAY=5
attempt=0

# Health check loop
while [ $attempt -lt $MAX_ATTEMPTS ]; do
  echo "Health check attempt $((attempt + 1))/$MAX_ATTEMPTS..."

  # Send HTTP request to health endpoint
  http_code=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_URL)

  if [ "$http_code" -eq 200 ]; then
    echo "✓ Health check passed (HTTP $http_code)"

    # Additional validation: check response body
    response=$(curl -s $HEALTH_URL)

    # Verify response contains expected content
    if echo "$response" | grep -q '"status":"healthy"'; then
      echo "✓ Application is healthy and responding correctly"

      # Optional: Check database connection
      if echo "$response" | grep -q '"database":"connected"'; then
        echo "✓ Database connection verified"
      fi

      # Optional: Check cache connection
      if echo "$response" | grep -q '"cache":"connected"'; then
        echo "✓ Cache connection verified"
      fi

      echo "========================================="
      echo "Deployment validation SUCCESSFUL"
      echo "========================================="
      exit 0  # Success - deployment proceeds
    else
      echo "✗ Response body invalid: $response"
    fi
  else
    echo "✗ Health check failed (HTTP $http_code)"
  fi

  echo "Retrying in $RETRY_DELAY seconds..."
  sleep $RETRY_DELAY
  ((attempt++))
done

# All attempts failed
echo "========================================="
echo "✗✗✗ Health check FAILED after $MAX_ATTEMPTS attempts"
echo "========================================="
echo "Deployment will be marked as FAILED"
echo "Automatic rollback will be triggered"
exit 1  # Failure → triggers rollback
```

**AppSpec for Windows:**

```yaml
version: 0.0
os: windows
# Note: 'os: windows' instead of 'os: linux'

files:
  - source: \
    destination: C:\inetpub\wwwroot\myapp
    # Windows paths use backslashes

permissions:
  # Permissions section not used on Windows
  # Use BeforeInstall hook to set ACLs if needed

hooks:
  ApplicationStop:
    - location: scripts\stop_iis.ps1
      timeout: 60
      # PowerShell script to stop IIS app pool

  BeforeInstall:
    - location: scripts\backup.ps1
      timeout: 120

  AfterInstall:
    - location: scripts\configure_iis.ps1
      timeout: 300
      # Configure IIS application

  ApplicationStart:
    - location: scripts\start_iis.ps1
      timeout: 60
      # Start IIS app pool

  ValidateService:
    - location: scripts\health_check.ps1
      timeout: 120
```

**Windows PowerShell Hook Example:**
```powershell
# scripts/stop_iis.ps1
# ApplicationStop hook for IIS

Write-Host "=== ApplicationStop Hook ==="

# Stop IIS Application Pool
$appPoolName = "MyAppPool"

if (Test-Path "IIS:\AppPools\$appPoolName") {
    Write-Host "Stopping IIS Application Pool: $appPoolName"
    Stop-WebAppPool -Name $appPoolName

    # Wait for app pool to stop
    $timeout = 30
    while ((Get-WebAppPoolState $appPoolName).Value -ne "Stopped" -and $timeout -gt 0) {
        Write-Host "Waiting for app pool to stop... ($timeout seconds)"
        Start-Sleep -Seconds 1
        $timeout--
    }

    Write-Host "Application Pool stopped"
} else {
    Write-Host "Application Pool not found (first deployment)"
}

exit 0
```

**AppSpec for Lambda:**

Lambda uses a completely different AppSpec format:

```yaml
version: 0.0
Resources:
  - MyFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: "my-lambda-function"
        Alias: "live"
        # Alias that receives traffic
        CurrentVersion: "1"
        # Current version serving traffic
        TargetVersion: "2"
        # New version to shift traffic to

Hooks:
  BeforeAllowTraffic:
    - BeforeTrafficShift
    # Lambda function to invoke before traffic shift
    # Use for: Pre-deployment tests, validation

  AfterAllowTraffic:
    - AfterTrafficShift
    # Lambda function to invoke after traffic shift
    # Use for: Post-deployment validation, notifications
```

**Lambda Hook Function Example:**
```python
# before_traffic_shift_function.py
import json
import boto3
import time

def lambda_handler(event, context):
    """
    Pre-deployment validation for Lambda traffic shift.
    Runs integration tests on new version before allowing traffic.
    """
    print("=== BeforeAllowTraffic Hook ===")
    print(f"Event: {json.dumps(event)}")

    # Extract deployment information
    deployment_id = event['DeploymentId']
    lifecycle_event_hook_execution_id = event['LifecycleEventHookExecutionId']

    # Get function details
    function_name = event.get('FunctionName')
    new_version = event.get('TargetVersion')

    print(f"Testing new version: {new_version}")

    # Run integration tests on new version
    lambda_client = boto3.client('lambda')
    codedeploy_client = boto3.client('codedeploy')

    test_cases = [
        {"action": "create", "data": {"id": 1}},
        {"action": "read", "data": {"id": 1}},
        {"action": "update", "data": {"id": 1}},
        {"action": "delete", "data": {"id": 1}}
    ]

    all_tests_passed = True

    for i, test_case in enumerate(test_cases):
        print(f"Running test case {i+1}/{len(test_cases)}")

        try:
            # Invoke new Lambda version with test payload
            response = lambda_client.invoke(
                FunctionName=f"{function_name}:{new_version}",
                InvocationType='RequestResponse',
                Payload=json.dumps(test_case)
            )

            # Parse response
            result = json.loads(response['Payload'].read())

            # Validate response
            if result.get('statusCode') != 200:
                print(f"✗ Test case {i+1} failed: {result}")
                all_tests_passed = False
                break
            else:
                print(f"✓ Test case {i+1} passed")

        except Exception as e:
            print(f"✗ Test case {i+1} exception: {str(e)}")
            all_tests_passed = False
            break

    # Report results to CodeDeploy
    if all_tests_passed:
        print("✓ All tests passed - approving deployment")
        status = 'Succeeded'
    else:
        print("✗ Tests failed - rejecting deployment")
        status = 'Failed'

    # Signal CodeDeploy with result
    codedeploy_client.put_lifecycle_event_hook_execution_status(
        deploymentId=deployment_id,
        lifecycleEventHookExecutionId=lifecycle_event_hook_execution_id,
        status=status
    )

    return {
        'statusCode': 200 if all_tests_passed else 500,
        'body': json.dumps({
            'tests_passed': all_tests_passed,
            'deployment_id': deployment_id
        })
    }
```

**AppSpec for ECS:**

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-east-1:123456789012:task-definition/myapp:2"
        # New task definition to deploy
        LoadBalancerInfo:
          ContainerName: "myapp-container"
          ContainerPort: 8080
        PlatformVersion: "LATEST"
        # For Fargate tasks

Hooks:
  BeforeInstall:
    - location: scripts/pre_deploy_check.sh
      # Run before new tasks created

  AfterInstall:
    - location: scripts/post_deploy_validation.sh
      # Run after new tasks created (before traffic)

  AfterAllowTestTraffic:
    - location: scripts/integration_tests.sh
      # Run tests on new tasks with test traffic

  BeforeAllowTraffic:
    - location: scripts/final_validation.sh
      # Final check before production traffic

  AfterAllowTraffic:
    - location: scripts/monitoring_setup.sh
      # Configure monitoring after traffic shift
```

**Common AppSpec Mistakes:**

```yaml
❌ WRONG - appspec.yml in subdirectory:
myapp/
├── src/
│   └── appspec.yml  # ← Wrong location!
└── index.js

✓ CORRECT - appspec.yml in root:
myapp/
├── appspec.yml      # ← Correct location!
├── src/
└── index.js

❌ WRONG - Script doesn't handle failure:
hooks:
  ValidateService:
    - location: scripts/test.sh
      # Script exits with 0 even if tests fail

✓ CORRECT - Script exits with proper code:
hooks:
  ValidateService:
    - location: scripts/test.sh
      # Script: exit 1 if tests fail, exit 0 if pass

❌ WRONG - Timeout too short:
hooks:
  AfterInstall:
    - location: scripts/npm_install.sh
      timeout: 30  # npm install takes 5 minutes!

✓ CORRECT - Realistic timeout:
hooks:
  AfterInstall:
    - location: scripts/npm_install.sh
      timeout: 600  # 10 minutes for npm install

❌ WRONG - Running as wrong user:
hooks:
  ApplicationStart:
    - location: scripts/start.sh
      runas: root  # App shouldn't run as root!

✓ CORRECT - Running as app user:
hooks:
  ApplicationStart:
    - location: scripts/start.sh
      runas: webapp  # Run as dedicated app user
```

**Key Concepts:**
- AppSpec **must be in root** of deployment package (appspec.yml)
- **Hooks run in order**; if any fails, deployment fails
- Scripts must **exit 0** for success, **non-zero** for failure
- **timeout** specifies max execution time (script killed after timeout)
- **runas** defines which user executes the script
- Failed hooks trigger **automatic rollback**
- **ValidateService** is critical—last chance to detect issues before deployment succeeds
- Different platforms (EC2, Lambda, ECS) use different AppSpec formats

---


### Deployment Types (In-Place vs Blue/Green)

**What Are Deployment Types?**
Deployment types define the **fundamental strategy** for how the new version replaces the old version. This is one of the most important architectural decisions in your deployment pipeline.

**Two Main Types:**
1. **In-Place Deployment** - Update existing instances
2. **Blue/Green Deployment** - Create new infrastructure, switch traffic

---

**In-Place Deployment**

**How It Works:**
```
Existing Infrastructure (running old version)
         ↓
Update instances in-place (one at a time or in batches)
         ↓
Same Infrastructure (now running new version)
```

**Detailed Flow:**

```
Step 1: Initial State
ALB → [Instance 1: v1.0] ← Serving traffic
      [Instance 2: v1.0] ← Serving traffic
      [Instance 3: v1.0] ← Serving traffic

Step 2: Update Instance 1
ALB → [Instance 1: Deploying v2.0] ← Temporarily unavailable
      [Instance 2: v1.0] ← Serving ALL traffic
      [Instance 3: v1.0] ← Serving ALL traffic

      During this time:
      - Instance 1 de-registered from load balancer
      - Application stopped
      - New version deployed
      - Application started
      - Health checks performed

Step 3: Instance 1 Updated
ALB → [Instance 1: v2.0] ← Back in service
      [Instance 2: v1.0] ← Serving traffic
      [Instance 3: v1.0] ← Serving traffic

Step 4: Update Instance 2
ALB → [Instance 1: v2.0] ← Serving traffic
      [Instance 2: Deploying v2.0] ← Temporarily unavailable
      [Instance 3: v1.0] ← Serving traffic

Step 5: Update Instance 3
ALB → [Instance 1: v2.0] ← Serving traffic
      [Instance 2: v2.0] ← Serving traffic
      [Instance 3: Deploying v2.0] ← Temporarily unavailable

Step 6: Final State
ALB → [Instance 1: v2.0] ← Serving traffic
      [Instance 2: v2.0] ← Serving traffic
      [Instance 3: v2.0] ← Serving traffic
```

**Advantages:**
```
✓ Cost-effective (reuses existing instances)
  - No temporary infrastructure
  - No double instance costs

✓ Simple infrastructure (no complex setup)
  - Uses existing Auto Scaling Groups
  - No need for multiple target groups

✓ Fast (no new instance provisioning)
  - Just deploy code, don't launch new instances

✓ Works with limited capacity
  - Good for small instance counts
  - Minimal AWS resource requirements
```

**Disadvantages:**
```
✗ Reduced capacity during deployment
  - Some instances offline during update
  - May impact performance under high load

✗ Potential downtime (if configured AllAtOnce)
  - All instances offline simultaneously

✗ Slower rollback (redeploy old version)
  - Must re-run deployment with old revision
  - Takes same time as original deployment

✗ Both versions running simultaneously (temporarily)
  - v1.0 and v2.0 both serving traffic during deployment
  - Potential compatibility issues (database schema, API contracts)

✗ Cannot test new version before going live
  - New version immediately gets real traffic
```

**Best For:**
- Development/staging environments
- Non-critical applications
- Cost-sensitive deployments
- Small instance fleets (< 10 instances)
- Applications with backwards-compatible changes

**Configuration Example:**
```yaml
Deployment Type: In-Place
Deployment Configuration: HalfAtATime
Load Balancer: Enabled

Result:
- Deploy to 50% of instances
- Health check
- Deploy to remaining 50%
- Total time: ~15 minutes for 10 instances
- Cost: $0 extra (uses existing instances)
```

---

**Blue/Green Deployment**

**How It Works:**
```
Blue Environment (current production, v1.0)
         ↓
Green Environment created (new instances, v2.0)
         ↓
Test Green Environment (verify it works)
         ↓
Switch traffic: Blue → Green
         ↓
Green Environment is now production
         ↓
Optionally terminate Blue (or keep for rollback)
```

**Detailed Flow:**

```
Step 1: Initial State (Blue environment serving traffic)
┌─────────────────────────────────────────┐
│            Production Traffic           │
│                   ↓                     │
│   ALB → Blue Target Group (100%)        │
│          ├── Instance 1: v1.0           │
│          ├── Instance 2: v1.0           │
│          └── Instance 3: v1.0           │
│                                         │
│         Green Target Group              │
│          (empty)                        │
└─────────────────────────────────────────┘

Step 2: Create Green Environment
┌─────────────────────────────────────────┐
│            Production Traffic           │
│                   ↓                     │
│   ALB → Blue Target Group (100%)        │
│          ├── Instance 1: v1.0 ← Traffic │
│          ├── Instance 2: v1.0 ← Traffic │
│          └── Instance 3: v1.0 ← Traffic │
│                                         │
│         Green Target Group (0%)         │
│          ├── Instance 4: v2.0 ← No traffic
│          ├── Instance 5: v2.0 ← No traffic
│          └── Instance 6: v2.0 ← No traffic
│             (being deployed & tested)   │
└─────────────────────────────────────────┘

Step 3: Test Green Environment
┌─────────────────────────────────────────┐
│   Production traffic still on Blue      │
│                                         │
│   Blue: Serving 100% of real users      │
│                                         │
│   Green: Receiving test traffic         │
│   - Health checks                       │
│   - Smoke tests                         │
│   - Integration tests                   │
│   - Manual validation                   │
│                                         │
│   Duration: 5-30 minutes (configurable) │
└─────────────────────────────────────────┘

Step 4: Traffic Switch (Instant Cutover)
┌─────────────────────────────────────────┐
│            Production Traffic           │
│                   ↓                     │
│   ALB → Blue Target Group (0%)          │
│          ├── Instance 1: v1.0 (idle)    │
│          ├── Instance 2: v1.0 (idle)    │
│          └── Instance 3: v1.0 (idle)    │
│                                         │
│   ALB → Green Target Group (100%)       │
│          ├── Instance 4: v2.0 ← Traffic │
│          ├── Instance 5: v2.0 ← Traffic │
│          └── Instance 6: v2.0 ← Traffic │
│                                         │
│   Switch time: < 1 second               │
└─────────────────────────────────────────┘

Step 5: Monitor Green (Post-Deployment)
┌─────────────────────────────────────────┐
│   Green serving production traffic      │
│   - Monitor CloudWatch metrics          │
│   - Watch error rates                   │
│   - Check application logs              │
│   - Verify business metrics             │
│                                         │
│   Blue environment kept for rollback    │
│   - Ready for instant traffic switch   │
│   - Duration: 1 hour - 24 hours         │
└─────────────────────────────────────────┘

Step 6: Terminate Blue (After Validation)
┌─────────────────────────────────────────┐
│            Production Traffic           │
│                   ↓                     │
│   ALB → Green Target Group (100%)       │
│          ├── Instance 4: v2.0           │
│          ├── Instance 5: v2.0           │
│          └── Instance 6: v2.0           │
│                                         │
│         Blue Target Group               │
│          (instances terminated)         │
│                                         │
│   Deployment complete ✓                 │
└─────────────────────────────────────────┘
```

**Advantages:**
```
✓ Zero downtime (instant traffic switch)
  - < 1 second cutover
  - Users don't notice deployment

✓ Fast rollback (instant traffic switch back)
  - Switch traffic back to Blue
  - < 1 minute rollback time

✓ Test new version before production traffic
  - Full testing on Green environment
  - Verify everything works before users affected

✓ Full capacity maintained throughout
  - Both environments running
  - No performance degradation

✓ Clean separation (no mixed versions)
  - Either all v1.0 or all v2.0
  - No compatibility issues between versions

✓ Easy rollback window
  - Keep Blue environment for hours/days
  - Instant rollback available anytime
```

**Disadvantages:**
```
✗ Higher cost (temporary double infrastructure)
  - Running both Blue and Green simultaneously
  - 2x instance costs during deployment
  - Example: 10 instances → 20 instances temporarily

✗ More complex setup
  - Requires load balancer
  - Need two target groups
  - Auto Scaling Group coordination

✗ Requires more AWS resources
  - Additional ENIs, IPs
  - More complex IAM permissions

✗ Database migration challenges
  - Schema changes must be backwards compatible
  - Both versions may access same database
  - Need careful migration planning
```

**Best For:**
- Production environments
- Mission-critical applications
- Zero-downtime requirements
- High-traffic applications
- When fast rollback is essential
- Customer-facing services

**Configuration Example:**
```yaml
Deployment Type: Blue/Green
Traffic Rerouting: Load Balancer
Original Instances: Terminate after 1 hour

Result:
- Create new Auto Scaling Group (Green)
- Deploy to Green instances
- Test Green environment
- Switch ALB target group: Blue → Green
- Monitor Green for 1 hour
- Terminate Blue instances
- Total time: ~20 minutes deployment + 1 hour validation
- Cost: 2x instances for ~1.5 hours
```

---

**Comparison Matrix:**

```
┌─────────────────────────┬────────────────────┬────────────────────┐
│ Aspect                  │ In-Place           │ Blue/Green         │
├─────────────────────────┼────────────────────┼────────────────────┤
│ Downtime                │ Possible (minimal) │ Zero               │
│ Rollback Speed          │ Slow (redeploy)    │ Instant (< 1 min)  │
│ Rollback Method         │ Redeploy old ver   │ Traffic switch     │
│ Cost                    │ Lower              │ Higher (2x temp)   │
│ Complexity              │ Simple             │ Complex            │
│ Infrastructure          │ Existing instances │ New instances      │
│ Testing Before Live     │ Limited            │ Full               │
│ Mixed Versions          │ Yes (temporarily)  │ No                 │
│ Capacity During Deploy  │ Reduced            │ Full (2x)          │
│ Instance Requirement    │ N instances        │ 2N (temporarily)   │
│ Deployment Duration     │ 10-30 min          │ 15-25 min          │
│ Supported Platforms     │ EC2, On-premises   │ EC2, Lambda, ECS   │
│ Load Balancer Required  │ No (recommended)   │ Yes (mandatory)    │
│ Database Migrations     │ Easier             │ Challenging        │
└─────────────────────────┴────────────────────┴────────────────────┘
```

**Decision Tree:**

```
Need zero downtime?
├─ Yes → Blue/Green
└─ No → Consider cost
    ├─ Cost sensitive? → In-Place
    └─ Need fast rollback? → Blue/Green

Mission-critical application?
├─ Yes → Blue/Green
└─ No → In-Place

High traffic / revenue impact?
├─ Yes → Blue/Green
└─ No → In-Place

Development/Staging environment?
├─ Yes → In-Place
└─ No (Production) → Blue/Green
```

**Real-World Scenario - E-Commerce Site:**

```
Scenario: Black Friday deployment

Application: E-commerce website
Traffic: 10,000 requests/second
Revenue: $100,000/hour
Downtime cost: $1,666/minute

Decision: Blue/Green

Reasoning:
✓ Zero downtime = no revenue loss
✓ Instant rollback if issues
✓ Full capacity during deployment
✓ Can test before production traffic

Cost Analysis:
- In-Place potential downtime: 5 minutes = $8,330 lost revenue
- Blue/Green extra cost: $50 (1 hour of extra instances)
- ROI: $8,330 revenue saved - $50 cost = $8,280 benefit

Clear winner: Blue/Green
```

**Blue/Green with Auto Scaling Groups:**

```yaml
Blue/Green Deployment Process with ASG:

1. Original Auto Scaling Group (Blue):
   Name: webapp-blue-asg
   Instances: 10
   Launch Template: webapp-lt-v1
   Target Group: webapp-blue-tg

2. CodeDeploy creates new ASG (Green):
   Name: webapp-green-asg
   Instances: 10 (matches Blue capacity)
   Launch Template: webapp-lt-v2
   Target Group: webapp-green-tg

3. Deployment:
   - Green instances launch
   - CodeDeploy deploys application to Green
   - Health checks on Green

4. Traffic Switch:
   ALB Rule:
     Before: Route to webapp-blue-tg (100%)
     After:  Route to webapp-green-tg (100%)

5. Cleanup Options:
   a) Terminate Blue immediately
   b) Keep Blue for X hours (rollback window)
   c) Keep Blue at 0 capacity (instant scale-up if needed)
```

**Handling Database Migrations:**

```
Challenge: Schema changes between versions

Scenario:
v1.0: users table [id, name, email]
v2.0: users table [id, name, email, phone] ← new column

In-Place Deployment:
1. Deploy v2.0 code
2. Run migration: ADD COLUMN phone
3. Application uses new column
Problem: If rollback, v1.0 code ignores 'phone' column ✓
Solution: Works fine (v1.0 doesn't break with extra column)

Blue/Green Deployment:
Problem: Both environments access same database
1. Green (v2.0) needs 'phone' column
2. Blue (v1.0) doesn't know about 'phone' column
3. What happens during traffic switch?

Solution 1 - Backwards Compatible Migration:
Step 1: Add 'phone' column as NULLABLE
  - Both v1.0 and v2.0 work with database
  - v1.0 ignores column
  - v2.0 uses column
Step 2: Deploy v2.0 (Blue/Green)
Step 3: After stable, make column NOT NULL if needed

Solution 2 - Separate Databases:
  - Blue: Database instance 1 (old schema)
  - Green: Database instance 2 (new schema)
  - Migrate data before traffic switch
  - Complex, expensive, rarely used

Solution 3 - Expand-Contract Pattern:
Phase 1 (Expand): Add new column, both versions work
Phase 2 (Deploy): Blue/Green to v2.0
Phase 3 (Contract): Remove old columns after stable
```

**Blue/Green Traffic Shifting Strategies:**

```
Strategy 1: All-At-Once
Time 0: Blue 100%, Green 0%
Time 1: Blue 0%, Green 100%
Duration: Instant switch
Risk: High (all users immediately on new version)
Use: After thorough Green testing

Strategy 2: Canary
Time 0: Blue 100%, Green 0%
Time 5: Blue 90%, Green 10% (canary)
Wait 30 min, monitor metrics
Time 35: Blue 0%, Green 100% (if healthy)
Duration: 35 minutes
Risk: Low (10% of users test first)
Use: High-risk deployments

Strategy 3: Linear
Time 0: Blue 100%, Green 0%
Time 5: Blue 80%, Green 20%
Time 10: Blue 60%, Green 40%
Time 15: Blue 40%, Green 60%
Time 20: Blue 20%, Green 80%
Time 25: Blue 0%, Green 100%
Duration: 25 minutes
Risk: Moderate (gradual migration)
Use: Standard production deployments
```

**Key Concepts:**
- **In-Place** = update existing infrastructure (cheaper, some downtime risk)
- **Blue/Green** = new infrastructure (expensive, zero downtime)
- Blue/Green requires **load balancer** for traffic switch
- **Rollback**: In-Place (redeploy old version ~10-15 min), Blue/Green (traffic switch < 1 min)
- Choose based on **downtime tolerance**, **budget**, and **application criticality**
- **Database migrations** require careful planning with Blue/Green
- Blue/Green provides **testing opportunity** before production traffic
- Always test deployment strategy in **staging first**

---

### Rollback Configuration

**What is Rollback?**
Rollback is the process of **reverting to a previous, working version** when a deployment fails or causes issues. CodeDeploy supports **automatic rollback** based on configurable triggers.

**Why Rollback Matters:**

```
Scenario: Deployment introduces critical bug

WITHOUT Auto-Rollback:
0:00 - Deployment starts
0:05 - Deployment completes
0:10 - Users report 500 errors
0:25 - Team investigates logs
0:40 - Root cause identified
0:45 - Decision made to rollback
0:50 - Manual rollback initiated
1:00 - Rollback complete
Total downtime: 55 minutes
User impact: HIGH (hundreds of users affected)

WITH Auto-Rollback:
0:00 - Deployment starts
0:02 - Deployment to first batch
0:03 - Health checks fail on first batch
0:03 - Auto-rollback triggered immediately
0:06 - Rollback complete
Total downtime: 6 minutes
User impact: LOW (only users on first batch affected)
```

**Rollback Triggers:**

```yaml
Automatic Rollback Configuration:

Trigger 1: Deployment Failure
  When:
    - Any lifecycle hook fails (exit code != 0)
    - Instance health checks fail
    - Deployment timeout exceeded
    - CodeDeploy agent errors

Trigger 2: CloudWatch Alarms
  When:
    - CPU > 90% for 5 minutes
    - HTTP 5xx errors > threshold
    - Response time > 3 seconds
    - Custom application metrics breach

Trigger 3: Manual Rollback
  When:
    - Developer clicks "Stop and rollback"
    - AWS Console, CLI, or API
    - Human decision required
```

**Rollback Configuration Example:**

```yaml
Application: MyWebApp
Deployment Group: Production

Rollback Configuration:
  Enable Auto-Rollback: true

  Rollback When Deployment Fails: true
  # Any deployment failure triggers rollback
  # Examples:
  #   - AfterInstall hook fails
  #   - ValidateService hook times out
  #   - Instance doesn't become healthy

  Rollback When Alarm Threshold Met: true
  # CloudWatch alarms trigger rollback

  Linked CloudWatch Alarms:
    - Name: HighErrorRate
      Metric: HTTPCode_Target_5XX_Count
      Namespace: AWS/ApplicationELB
      Statistic: Sum
      Threshold: > 10 errors
      Period: 300 seconds (5 minutes)
      Evaluation Periods: 1
      # Triggers if > 10 5XX errors in 5-minute window

    - Name: HighLatency
      Metric: TargetResponseTime
      Namespace: AWS/ApplicationELB
      Statistic: Average
      Threshold: > 2000 (ms)
      Period: 300 seconds
      Evaluation Periods: 2
      # Triggers if avg response > 2s for 2 consecutive periods

    - Name: LowHealthyHosts
      Metric: HealthyHostCount
      Namespace: AWS/ApplicationELB
      Statistic: Minimum
      Threshold: < 3
      Period: 60 seconds
      Evaluation Periods: 1
      # Triggers if < 3 healthy hosts

Rollback Behavior:
  Type: ROLL_BACK
  # Redeploys last successful revision
  # Alternative: STOP_AND_FAIL (stop without rollback)
```

**How Rollback Works (In-Place Deployment):**

```
Deployment Timeline with Rollback:

0:00 - Start deployment v2.0
       Instances: [v1.0][v1.0][v1.0][v1.0]

0:02 - Deploy to first instance (HalfAtATime = 2 instances)
       Instances: [v2.0][v2.0][v1.0][v1.0]
       Status: Deploying

0:04 - Health check on first batch
       Instance 1: Health check PASS ✓
       Instance 2: Health check FAIL ✗ (returning 500 errors)
       CloudWatch Alarm: HighErrorRate triggered

0:04 - AUTOMATIC ROLLBACK INITIATED
       Reason: Alarm "HighErrorRate" in ALARM state
       Action: Stop deployment, revert to v1.0

0:05 - Rollback in progress
       Instances: [Reverting][Reverting][v1.0][v1.0]
       Running: ApplicationStop hook for v2.0
       Running: ApplicationStop → Install v1.0 → ApplicationStart

0:07 - Rollback complete
       Instances: [v1.0][v1.0][v1.0][v1.0]
       Status: FAILED (rolled back)
       All instances back to previous working version

Timeline Summary:
  - Deployment started: 0:00
  - Issue detected: 0:04 (4 minutes)
  - Rollback complete: 0:07 (7 minutes)
  - Total disruption: 7 minutes (only 2 instances affected)
```

**How Rollback Works (Blue/Green Deployment):**

```
Blue/Green Rollback (Much Faster):

0:00 - Traffic on Blue (v1.0)
       ALB → Blue Target Group (100%)
             ├── Instance 1: v1.0
             ├── Instance 2: v1.0
             └── Instance 3: v1.0

       Green Target Group (0%)
             (empty)

0:05 - Green environment created and deployed
       ALB → Blue Target Group (100%) ← Still serving traffic
             ├── Instance 1: v1.0
             ├── Instance 2: v1.0
             └── Instance 3: v1.0

       Green Target Group (0%) ← No traffic yet
             ├── Instance 4: v2.0 (deployed)
             ├── Instance 5: v2.0 (deployed)
             └── Instance 6: v2.0 (deployed)

0:10 - Traffic switched to Green
       ALB → Blue Target Group (0%)
             ├── Instance 1: v1.0 (idle)
             ├── Instance 2: v1.0 (idle)
             └── Instance 3: v1.0 (idle)

       Green Target Group (100%) ← Now serving traffic
             ├── Instance 4: v2.0 ← Serving
             ├── Instance 5: v2.0 ← Serving
             └── Instance 6: v2.0 ← Serving

0:12 - CloudWatch Alarm triggers
       Alarm: HighErrorRate (5XX errors from Green)

0:12 - AUTOMATIC ROLLBACK INITIATED
       Action: Switch traffic back to Blue

0:12 - Traffic switched back (< 30 seconds)
       ALB → Blue Target Group (100%) ← Restored
             ├── Instance 1: v1.0 ← Serving again
             ├── Instance 2: v1.0 ← Serving again
             └── Instance 3: v1.0 ← Serving again

       Green Target Group (0%)
             ├── Instance 4: v2.0 (terminated)
             ├── Instance 5: v2.0 (terminated)
             └── Instance 6: v2.0 (terminated)

Timeline Summary:
  - Deployment started: 0:00
  - Traffic switched to Green: 0:10
  - Issue detected: 0:12 (2 minutes after switch)
  - Rollback complete: 0:12.5 (30 seconds!)
  - Total disruption: 2.5 minutes
```

**CloudWatch Alarm Integration:**

```
Create CloudWatch Alarm for Rollback:

Alarm Name: deployment-5xx-errors
Metric: HTTPCode_Target_5XX_Count
Namespace: AWS/ApplicationELB
Dimension: TargetGroup = webapp-prod-tg

Statistic: Sum
Period: 60 seconds
Evaluation Periods: 2
Datapoints to Alarm: 2 out of 2
Threshold: > 5

Meaning:
  "Alarm if more than 5 5XX errors occur in a 60-second window,
   for 2 consecutive periods (2 minutes total)"

Link to Deployment Group:
  Deployment Group: Production
  Rollback Alarms:
    - deployment-5xx-errors

  Behavior during deployment:
    1. Deployment starts
    2. CodeDeploy monitors alarm state
    3. If alarm = ALARM state → Stop deployment
    4. Trigger automatic rollback
    5. Return to last successful revision
```

**Multiple Alarms Example:**

```yaml
Deployment Group: Production
Rollback Alarms:
  - cpu-utilization-high
  - memory-utilization-high
  - error-rate-high
  - response-time-high
  - queue-depth-high

Logic: ANY alarm triggers rollback
  If cpu-utilization-high = ALARM → Rollback
  OR error-rate-high = ALARM → Rollback
  OR response-time-high = ALARM → Rollback

Result: Maximum protection against bad deployments
```

**Rollback Timeline Visualization:**

```
Normal Deployment (No Issues):
0:00 ┤ Deploy Start
     │ ████████████ Deploying
0:10 ┤ Deploy Complete ✓
     │ Monitoring (no alarms)
0:20 ┤ Deployment Successful
     └─────────────────────────

Failed Deployment with Rollback:
0:00 ┤ Deploy Start
     │ ████████ Deploying
0:03 ┤ Alarm Triggered! ⚠
     │ ▓▓▓ Rolling Back
0:06 ┤ Rollback Complete ✓
     │ Back to v1.0
0:10 ┤ Deployment Failed (safely)
     └─────────────────────────

Visual Legend:
████ = Deployment in progress
▓▓▓▓ = Rollback in progress
⚠ = Alarm triggered
✓ = Success
```

**Manual Rollback:**

```bash
# Stop current deployment and rollback
aws deploy stop-deployment \
  --deployment-id d-ABCDEFGH123 \
  --auto-rollback-enabled

# Result:
# - Current deployment stops immediately
# - Last successful revision redeployed
# - Takes 5-10 minutes for in-place
# - Takes < 1 minute for blue/green

# View deployment status
aws deploy get-deployment \
  --deployment-id d-ABCDEFGH123

# Output shows:
# - Status: STOPPED
# - Auto-rollback: TRIGGERED
# - Rollback deployment ID: d-NEWROLLBACK456
```

**Rollback Best Practices:**

```yaml
✅ DO:
Production:
  - Enable automatic rollback: true
  - Configure CloudWatch alarms for:
    * Error rates (5XX)
    * Response times
    * CPU/Memory utilization
    * Custom application metrics
  - Test rollback procedures regularly (monthly)
  - Keep minimum 2 previous revisions available
  - Document rollback decision criteria
  - Monitor deployments actively during rollback window

Staging:
  - Test deployment before production
  - Validate alarms trigger correctly
  - Practice rollback scenarios

❌ DON'T:
  - Disable rollback in production (never!)
  - Set alarm thresholds too sensitive (false positives)
  - Delete previous deployment revisions
  - Ignore failed deployments without investigation
  - Deploy to production without staging test
  - Configure rollback without testing it
```

**Rollback with Database Migrations:**

```
Challenge: Database schema changed in v2.0

Scenario:
v1.0 Database Schema:
  users: [id, name, email]

v2.0 Database Schema:
  users: [id, name, email, phone]  ← Added column
  Migration: ALTER TABLE users ADD COLUMN phone VARCHAR(20);

Deployment:
1. Deploy v2.0 application code
2. Run migration (adds 'phone' column)
3. Application uses new schema

Rollback Scenario:
Problem: How to rollback application when schema changed?

Solution 1: Forward-Compatible Migrations (Recommended)
  Phase 1: Add column as NULLABLE
    ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;
    - v1.0 code ignores 'phone' column (no error)
    - v2.0 code uses 'phone' column
    - Rollback to v1.0: Works fine! ✓

  Phase 2: (After v2.0 stable for weeks)
    Make column NOT NULL if needed
    Remove old code paths

Solution 2: Separate Migration Deployment
  Deployment 1: Run migration only
    - Deploy schema change
    - Wait, monitor
  Deployment 2: Deploy application code
    - Uses new schema
    - Easier to rollback (schema already in place)

Solution 3: Blue/Green with Separate Databases (Complex)
  - Blue environment: Old database (old schema)
  - Green environment: New database (new schema)
  - Data migration before traffic switch
  - Rollback: Switch back to Blue database
  - Rarely used (expensive, complex)
```

**Rollback Decision Tree:**

```
Deployment completed
         ↓
Monitor for issues
         ↓
Are there errors? ──No──→ Deployment successful ✓
         │
         Yes
         ↓
Is it affecting users? ──No──→ Monitor closely, fix in next deployment
         │
         Yes
         ↓
How many users affected?
         ├─ <5% ──→ Can you hotfix quickly? ──Yes──→ Deploy hotfix
         │                                   └─No───→ Rollback
         │
         ├─ 5-20% ──→ Rollback (moderate impact)
         │
         └─ >20% ──→ IMMEDIATE ROLLBACK (high impact)

Rollback Time:
  - In-Place: 5-15 minutes
  - Blue/Green: <1 minute (traffic switch)
```

**Key Concepts:**
- **Automatic rollback** prevents prolonged outages (minutes vs. hours)
- **CloudWatch alarms** enable health-based rollback decisions
- **Blue/Green rollback** is significantly faster than in-place (seconds vs. minutes)
- Configure **multiple alarms** for comprehensive protection
- **Rollback is not a substitute** for proper testing (test in staging!)
- Always ensure **previous revisions** are available
- **Database migrations** require special planning for safe rollbacks
- **Test your rollback procedures** regularly (don't wait for emergency)
- Blue/Green keeps old environment, enabling **instant rollback**

---


### Lifecycle Hooks (Advanced Patterns)

**What Are Lifecycle Hooks?**
Lifecycle hooks are **specific points during deployment** where CodeDeploy pauses to execute your custom scripts. They provide complete control over the deployment process.

**Complete Hook Lifecycle:**

```
┌────────────────────────────────────────────────────────┐
│              DEPLOYMENT LIFECYCLE                      │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Phase            Hook Name          Your Script      │
│  ─────            ─────────          ───────────      │
│                                                        │
│  ① Stop       ApplicationStop        stop_app.sh      │
│  Old App      └─ Stop old version gracefully          │
│                                                        │
│  ② Download   DownloadBundle         (automatic)      │
│  Package      └─ CodeDeploy managed, no custom script │
│                                                        │
│  ③ Prepare    BeforeInstall          prepare.sh       │
│  Install      └─ Backup, create dirs, setup           │
│                                                        │
│  ④ Copy       Install                (automatic)      │
│  Files        └─ CodeDeploy managed, copies files     │
│                                                        │
│  ⑤ Configure  AfterInstall           configure.sh     │
│  App          └─ npm install, permissions, secrets    │
│                                                        │
│  ⑥ Start      ApplicationStart       start_app.sh     │
│  New App      └─ Start new version                    │
│                                                        │
│  ⑦ Validate   ValidateService        health_check.sh  │
│  Health       └─ Verify deployment success            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Hook Execution with Error Handling:**

```
Hook Execution Flow:

1. ApplicationStop
   ├─ Execute: scripts/stop_app.sh
   ├─ Timeout: 60 seconds
   ├─ Exit Code: 0 ✓
   └─ Status: SUCCESS → Continue

2. BeforeInstall
   ├─ Execute: scripts/backup.sh
   ├─ Timeout: 120 seconds
   ├─ Exit Code: 1 ✗ (backup failed)
   └─ Status: FAILURE → Deployment FAILED
          ↓
      Rollback triggered
          ↓
      Previous version restored

Alternative Success Path:
1. ApplicationStop → Success ✓
2. DownloadBundle → Success ✓
3. BeforeInstall → Success ✓
4. Install → Success ✓
5. AfterInstall → Success ✓
6. ApplicationStart → Success ✓
7. ValidateService → Success ✓
   └─ Deployment SUCCESS ✓
```

**Advanced Hook Patterns:**

**Pattern 1: Database Migration Hook**

```yaml
# appspec.yml
hooks:
  AfterInstall:
    - location: scripts/run_migrations.sh
      timeout: 600
      runas: webapp
      # Run database migrations after files copied
```

```bash
#!/bin/bash
# scripts/run_migrations.sh
# AfterInstall hook - Run database migrations

echo "=== Running Database Migrations ==="

cd /var/www/myapp

# Get database credentials from AWS Systems Manager
DB_HOST=$(aws ssm get-parameter --name /myapp/prod/db/host --query 'Parameter.Value' --output text)
DB_NAME=$(aws ssm get-parameter --name /myapp/prod/db/name --query 'Parameter.Value' --output text)
DB_USER=$(aws ssm get-parameter --name /myapp/prod/db/user --query 'Parameter.Value' --output text)
DB_PASSWORD=$(aws ssm get-parameter --name /myapp/prod/db/password --with-decryption --query 'Parameter.Value' --output text)

# Export for migration tool
export DATABASE_URL="postgresql://$DB_USER:$DB_PASSWORD@$DB_HOST/$DB_NAME"

# Backup database before migration (safety)
echo "Creating database backup..."
pg_dump $DATABASE_URL > /var/backups/db_$(date +%Y%m%d_%H%M%S).sql

# Run migrations
echo "Running migrations..."
npx sequelize-cli db:migrate

if [ $? -eq 0 ]; then
  echo "✓ Migrations completed successfully"

  # Log migration details
  echo "Migration: $(npx sequelize-cli db:migrate:status)" | tee -a /var/log/myapp/migrations.log

  exit 0
else
  echo "✗ Migration failed!"

  # Attempt to rollback migration
  echo "Rolling back last migration..."
  npx sequelize-cli db:migrate:undo:all

  exit 1  # Fail deployment
fi
```

**Pattern 2: Cache Warming Hook**

```bash
#!/bin/bash
# scripts/warm_cache.sh
# ValidateService hook - Warm up application caches

echo "=== Warming Application Caches ==="

API_URL="http://localhost:3000"

# Warm up critical endpoints (frequently accessed data)
echo "Warming product catalog cache..."
curl -s "$API_URL/api/products?preload=true" > /dev/null

echo "Warming user preferences cache..."
curl -s "$API_URL/api/preferences/preload" > /dev/null

echo "Warming recommendation engine..."
curl -s "$API_URL/api/recommendations/preload" > /dev/null

# Warm up static assets
echo "Warming CDN cache..."
curl -s "$API_URL/assets/bundle.js" > /dev/null
curl -s "$API_URL/assets/styles.css" > /dev/null

# Verify cache warming worked
CACHE_STATUS=$(curl -s "$API_URL/api/health/cache" | jq -r '.status')

if [ "$CACHE_STATUS" = "warm" ]; then
  echo "✓ Caches warmed successfully"
  exit 0
else
  echo "⚠ Cache warming incomplete (non-critical)"
  exit 0  # Don't fail deployment for cache issues
fi
```

**Pattern 3: Notification Hook**

```bash
#!/bin/bash
# scripts/notify_deployment.sh
# Multiple hooks - Send deployment notifications

echo "=== Sending Deployment Notifications ==="

# Get deployment details
DEPLOYMENT_ID=$(cat /opt/codedeploy-agent/deployment-root/deployment-instructions/*.json | jq -r '.DeploymentId')
DEPLOYMENT_GROUP=$(cat /opt/codedeploy-agent/deployment-root/deployment-instructions/*.json | jq -r '.DeploymentGroupName')

# Get Git commit info
GIT_COMMIT=$(git rev-parse --short HEAD)
GIT_AUTHOR=$(git log -1 --pretty=format:'%an')
GIT_MESSAGE=$(git log -1 --pretty=format:'%s')

# Send to Slack
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

SLACK_MESSAGE=$(cat <<EOF
{
  "text": "Deployment Started",
  "attachments": [
    {
      "color": "good",
      "fields": [
        {"title": "Deployment Group", "value": "$DEPLOYMENT_GROUP", "short": true},
        {"title": "Deployment ID", "value": "$DEPLOYMENT_ID", "short": true},
        {"title": "Commit", "value": "$GIT_COMMIT", "short": true},
        {"title": "Author", "value": "$GIT_AUTHOR", "short": true},
        {"title": "Message", "value": "$GIT_MESSAGE", "short": false}
      ]
    }
  ]
}
EOF
)

curl -X POST -H 'Content-type: application/json' \
  --data "$SLACK_MESSAGE" \
  "$SLACK_WEBHOOK"

echo "✓ Notification sent to Slack"
exit 0
```

**Pattern 4: Feature Flag Verification**

```bash
#!/bin/bash
# scripts/verify_feature_flags.sh
# ValidateService hook - Verify feature flags configured correctly

echo "=== Verifying Feature Flags ==="

API_URL="http://localhost:3000"

# Check feature flags endpoint
RESPONSE=$(curl -s "$API_URL/api/feature-flags")

# Verify critical features
PAYMENT_ENABLED=$(echo $RESPONSE | jq -r '.payment_enabled')
NEW_CHECKOUT=$(echo $RESPONSE | jq -r '.new_checkout_flow')

echo "Payment System: $PAYMENT_ENABLED"
echo "New Checkout: $NEW_CHECKOUT"

# Ensure payment system is enabled in production
if [ "$PAYMENT_ENABLED" != "true" ]; then
  echo "✗ ERROR: Payment system disabled in production!"
  exit 1  # Fail deployment
fi

# Verify feature flag service is accessible
if [ -z "$NEW_CHECKOUT" ]; then
  echo "✗ ERROR: Cannot connect to feature flag service!"
  exit 1
fi

echo "✓ Feature flags verified"
exit 0
```

**Pattern 5: Load Balancer Drain Wait**

```bash
#!/bin/bash
# scripts/wait_for_draining.sh
# ApplicationStop hook - Wait for connection draining

echo "=== Waiting for Connection Draining ==="

# Get instance ID
INSTANCE_ID=$(ec2-metadata --instance-id | cut -d " " -f 2)

# Get target group ARN (from environment or tags)
TARGET_GROUP_ARN=$(aws ec2 describe-tags \
  --filters "Name=resource-id,Values=$INSTANCE_ID" "Name=key,Values=TargetGroupArn" \
  --query 'Tags[0].Value' --output text)

echo "Instance: $INSTANCE_ID"
echo "Target Group: $TARGET_GROUP_ARN"

# Check draining status
MAX_WAIT=300  # 5 minutes
WAITED=0

while [ $WAITED -lt $MAX_WAIT ]; do
  # Get target health
  HEALTH=$(aws elbv2 describe-target-health \
    --target-group-arn $TARGET_GROUP_ARN \
    --targets Id=$INSTANCE_ID \
    --query 'TargetHealthDescriptions[0].TargetHealth.State' \
    --output text)

  echo "Target Health: $HEALTH (waited ${WAITED}s)"

  # If draining complete or not registered, proceed
  if [ "$HEALTH" = "unused" ] || [ "$HEALTH" = "draining" ]; then
    # Get active connection count
    CONNECTIONS=$(netstat -an | grep ESTABLISHED | wc -l)
    echo "Active connections: $CONNECTIONS"

    if [ $CONNECTIONS -eq 0 ]; then
      echo "✓ All connections drained"
      exit 0
    fi
  fi

  sleep 10
  WAITED=$((WAITED + 10))
done

echo "⚠ Draining timeout reached, proceeding anyway"
exit 0  # Don't fail deployment
```

**Hook Timeout Strategies:**

```yaml
# appspec.yml with strategic timeouts

hooks:
  ApplicationStop:
    - location: scripts/stop_app.sh
      timeout: 60
      # 60s: Should be quick (just stop process)

  BeforeInstall:
    - location: scripts/backup.sh
      timeout: 300
      # 5min: Backup might be large

  AfterInstall:
    - location: scripts/npm_install.sh
      timeout: 600
      # 10min: Package installation can be slow

    - location: scripts/run_migrations.sh
      timeout: 900
      # 15min: Database migrations can be complex

  ApplicationStart:
    - location: scripts/start_app.sh
      timeout: 120
      # 2min: Application startup time

  ValidateService:
    - location: scripts/health_check.sh
      timeout: 300
      # 5min: Health checks with retries
```

**Parallel Hooks (ECS Blue/Green):**

```yaml
# appspec.yml for ECS
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:...:task-definition/app:2"
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 8080

Hooks:
  BeforeInstall:
    - location: scripts/pre_deploy.sh
    - location: scripts/backup_db.sh
    # Both run in parallel

  AfterInstall:
    - location: scripts/configure.sh

  AfterAllowTestTraffic:
    - location: scripts/integration_tests.sh
    # Runs after test traffic routed to new tasks

  BeforeAllowTraffic:
    - location: scripts/smoke_tests.sh
    - location: scripts/security_scan.sh
    # Both run in parallel before production traffic

  AfterAllowTraffic:
    - location: scripts/monitor_metrics.sh
```

**Hook Debugging:**

```bash
# Enable verbose logging in hooks

#!/bin/bash
# scripts/debug_hook.sh

# Enable debug mode
set -x  # Print each command before executing
set -e  # Exit on first error
set -o pipefail  # Catch errors in pipes

# Logging function
log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a /var/log/codedeploy/hooks.log
}

log "=== AfterInstall Hook Starting ==="

# Log environment
log "Current user: $(whoami)"
log "Working directory: $(pwd)"
log "Environment variables:"
env | sort | tee -a /var/log/codedeploy/hooks.log

# Your hook logic
log "Running npm install..."
npm ci --production 2>&1 | tee -a /var/log/codedeploy/hooks.log

if [ ${PIPESTATUS[0]} -eq 0 ]; then
  log "✓ npm install succeeded"
else
  log "✗ npm install failed"
  exit 1
fi

log "=== AfterInstall Hook Complete ==="
exit 0
```

**Hook Execution Monitoring:**

```bash
# scripts/health_check.sh with detailed monitoring

#!/bin/bash

HEALTH_URL="http://localhost:3000/health"
METRICS_FILE="/var/log/codedeploy/health_metrics.json"

echo "=== ValidateService Hook ==="

# Initialize metrics
START_TIME=$(date +%s)
ATTEMPTS=0
MAX_ATTEMPTS=10

# Health check loop with metrics
while [ $ATTEMPTS -lt $MAX_ATTEMPTS ]; do
  ATTEMPT_START=$(date +%s)
  ATTEMPTS=$((ATTEMPTS + 1))

  echo "Health check attempt $ATTEMPTS/$MAX_ATTEMPTS..."

  # Make request with timing
  HTTP_CODE=$(curl -w "%{http_code}" -o /tmp/health_response.json \
    -s --max-time 10 $HEALTH_URL)

  ATTEMPT_END=$(date +%s)
  RESPONSE_TIME=$((ATTEMPT_END - ATTEMPT_START))

  # Log metrics
  cat >> $METRICS_FILE <<EOF
{
  "attempt": $ATTEMPTS,
  "timestamp": $(date +%s),
  "http_code": $HTTP_CODE,
  "response_time_seconds": $RESPONSE_TIME,
  "url": "$HEALTH_URL"
}
EOF

  # Check result
  if [ "$HTTP_CODE" -eq 200 ]; then
    TOTAL_TIME=$((ATTEMPT_END - START_TIME))
    echo "✓ Health check passed after $ATTEMPTS attempts ($TOTAL_TIME seconds total)"

    # Send metrics to CloudWatch
    aws cloudwatch put-metric-data \
      --namespace "CustomMetrics/CodeDeploy" \
      --metric-name HealthCheckAttempts \
      --value $ATTEMPTS \
      --unit Count

    aws cloudwatch put-metric-data \
      --namespace "CustomMetrics/CodeDeploy" \
      --metric-name HealthCheckDuration \
      --value $TOTAL_TIME \
      --unit Seconds

    exit 0
  fi

  echo "Health check failed (HTTP $HTTP_CODE), waiting 5 seconds..."
  sleep 5
done

echo "✗ Health check failed after $MAX_ATTEMPTS attempts"
exit 1
```

**Key Concepts:**
- Hooks run in **specific order** (cannot skip or reorder)
- **Any hook failure** fails entire deployment
- Use hooks for: migrations, caching, notifications, validation
- **timeout** prevents indefinite waits
- **exit 0** = success, **exit 1** = failure
- ValidateService is your **last line of defense**
- Hooks can run **in parallel** (within same lifecycle event)
- Always include **error handling** and **logging**
- Test hooks in **non-production** first

---

## CodeCommit

### What is CodeCommit?

**AWS CodeCommit** is a fully managed **source control service** that hosts secure and scalable **Git repositories**. Think of it as AWS's version of GitHub, GitLab, or Bitbucket.

**Real-World Problem It Solves:**

```
Traditional Git Hosting Challenges:

Self-Hosted GitLab/Gitea:
✗ Maintain servers (updates, security patches)
✗ Scale infrastructure for team growth
✗ Handle backups and disaster recovery
✗ Manage user authentication/authorization
✗ Configure CI/CD integrations
Cost: $200-500/month (servers + admin time)

Third-Party Services (GitHub, GitLab):
✗ Data stored outside AWS (egress costs)
✗ Separate authentication from AWS
✗ Integration complexity with AWS services
✗ Pricing scales with users
Cost: $10-50/user/month

CodeCommit Solution:
✓ Fully managed (no servers)
✓ Integrated with AWS IAM
✓ Automatic scaling
✓ Encrypted at rest and in transit
✓ Pay per request (not per user)
✓ Native AWS service integration
Cost: $1/month per active user (first 5 users free!)
```

**How CodeCommit Works:**

```
┌─────────────────────────────────────────────────────┐
│                  CodeCommit                         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Developer's Computer                               │
│  ┌──────────────────┐                               │
│  │ Local Git Repo   │                               │
│  │                  │                               │
│  │ $ git add .      │                               │
│  │ $ git commit -m  │                               │
│  │ $ git push       │                               │
│  └────────┬─────────┘                               │
│           │ HTTPS/SSH                               │
│           ↓                                         │
│  ┌─────────────────────────────────┐                │
│  │   CodeCommit Repository         │                │
│  │   (AWS Managed Git Server)      │                │
│  │                                 │                │
│  │   main ─────────────────○       │                │
│  │   develop ──────○               │                │
│  │   feature/login ──○             │                │
│  │                                 │                │
│  │   Encryption: AES-256           │                │
│  │   Redundancy: 3 AZs             │                │
│  │   Backups: Automatic            │                │
│  └─────────────────────────────────┘                │
│           │                                         │
│           │ Integration                             │
│           ↓                                         │
│  ┌─────────────────────────────────┐                │
│  │   AWS Services                  │                │
│  │   ├─ CodePipeline (CI/CD)       │                │
│  │   ├─ CodeBuild (Build)          │                │
│  │   ├─ Lambda (Triggers)          │                │
│  │   ├─ SNS (Notifications)        │                │
│  │   └─ CloudWatch (Monitoring)    │                │
│  └─────────────────────────────────┘                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Key Features:**

```
1. Fully Managed Git
   - Standard Git operations (clone, push, pull, branch, merge)
   - No server maintenance
   - Automatic scaling
   - 99.99% availability SLA

2. Security
   - Encryption at rest (AWS KMS)
   - Encryption in transit (HTTPS, SSH)
   - IAM integration (no separate user database)
   - VPC endpoint support (private connectivity)
   - IP allowlisting

3. Collaboration
   - Pull requests
   - Code reviews
   - Comments on code
   - Approval rules
   - Branch protections

4. Integration
   - CodePipeline (automatic CI/CD triggers)
   - CodeBuild (build on commit)
   - Lambda (custom triggers)
   - SNS (notifications)
   - CloudWatch Events (automation)

5. Scalability
   - Unlimited repositories
   - Unlimited branches
   - Files up to 2 GB
   - Repositories up to 2 TB
```

---

### Repositories

**What is a Repository?**
A repository in CodeCommit is a **storage location for your code**, including all versions and history, branches, and metadata.

**Creating a Repository:**

```bash
# Via AWS CLI
aws codecommit create-repository \
  --repository-name my-web-app \
  --repository-description "E-commerce web application" \
  --tags Team=Engineering,Environment=Production

# Response:
{
  "repositoryMetadata": {
    "repositoryName": "my-web-app",
    "repositoryId": "12345678-1234-1234-1234-123456789012",
    "Arn": "arn:aws:codecommit:us-east-1:123456789012:my-web-app",
    "cloneUrlHttp": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-web-app",
    "cloneUrlSsh": "ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-web-app",
    "creationDate": "2026-02-06T10:30:00.000Z",
    "lastModifiedDate": "2026-02-06T10:30:00.000Z"
  }
}
```

**Cloning a Repository:**

```bash
# HTTPS (recommended - uses IAM credentials)
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-web-app

# Credential Helper (stores AWS credentials for Git)
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

# Now git push/pull work automatically with IAM credentials
```

**Repository Permissions (IAM-Based):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codecommit:GitPull",
        "codecommit:GitPush"
      ],
      "Resource": "arn:aws:codecommit:us-east-1:123456789012:my-web-app"
    }
  ]
}

// Granular permissions:
// - GitPull: Clone, fetch, pull
// - GitPush: Push commits
// - CreateBranch: Create new branches
// - DeleteBranch: Delete branches
// - GetBranch: View branch info
// - MergePullRequestByFastForward: Merge PRs
```

**Repository Structure:**

```
my-web-app (CodeCommit Repository)
├── Branches
│   ├── main (default branch)
│   ├── develop
│   ├── feature/user-auth
│   ├── feature/payment-gateway
│   └── hotfix/security-patch
│
├── Commits
│   ├── abc123 - "Add payment feature" (latest)
│   ├── def456 - "Fix login bug"
│   └── ghi789 - "Initial commit"
│
├── Pull Requests
│   ├── PR #1: feature/user-auth → develop
│   ├── PR #2: feature/payment-gateway → develop
│   └── PR #3: hotfix/security-patch → main
│
└── Settings
    ├── Default branch: main
    ├── Triggers: Lambda on push
    ├── Notifications: SNS topic
    └── Approval rules: 2 reviewers required
```

---

### Branches

**What Are Branches?**
Branches in CodeCommit work exactly like standard Git branches—they're independent lines of development.

**Common Branching Strategies:**

**Strategy 1: GitFlow**
```
Production (main)
  │
  ├─── develop (integration branch)
  │     │
  │     ├─── feature/add-payment
  │     ├─── feature/user-profile
  │     └─── feature/notifications
  │
  └─── hotfix/critical-bug (emergency fixes)

Workflow:
1. Features developed in feature/* branches
2. Merge to develop for testing
3. Release from develop to main
4. Hotfixes go directly to main

Use case: Scheduled releases, multiple features in parallel
```

**Strategy 2: Trunk-Based Development**
```
main (trunk)
  ├─── short-lived-feature-1 (< 2 days)
  ├─── short-lived-feature-2 (< 2 days)
  └─── short-lived-fix (< 1 day)

Workflow:
1. Create short-lived branches from main
2. Merge back to main quickly (< 2 days)
3. Deploy main frequently (multiple times/day)

Use case: Continuous deployment, small teams
```

**Branch Protection:**

```bash
# Get branch information
aws codecommit get-branch \
  --repository-name my-web-app \
  --branch-name main

# Create protected branch (via approval rules)
aws codecommit put-approval-rule \
  --approval-rule-name "Require-2-Reviewers" \
  --approval-rule-content '{
    "Version": "2018-11-08",
    "DestinationReferences": ["refs/heads/main"],
    "Statements": [{
      "Type": "Approvers",
      "NumberOfApprovalsNeeded": 2,
      "ApprovalPoolMembers": [
        "arn:aws:iam::123456789012:user/senior-dev-1",
        "arn:aws:iam::123456789012:user/senior-dev-2"
      ]
    }]
  }'

# Result: main branch requires 2 approvals before merge
```

---

### Pull Requests

**What Are Pull Requests?**
Pull Requests (PRs) are proposals to merge code changes from one branch to another, enabling **code review** and **approval workflows**.

**Pull Request Workflow:**

```
Developer creates feature:
  1. Create branch: feature/add-payment
  2. Write code, commit changes
  3. Push to CodeCommit
  4. Create pull request: feature/add-payment → develop
     ↓
Code Review:
  5. Reviewers notified (SNS/Email)
  6. Reviewers examine code changes
  7. Reviewers add comments
  8. Developer addresses feedback
  9. Reviewers approve
     ↓
Merge:
  10. PR meets approval requirements (2 approvers)
  11. Automated tests pass (CodeBuild)
  12. Merge to develop branch
  13. Feature branch optionally deleted
  14. Notifications sent (Slack/Email)
```

**Creating a Pull Request:**

```bash
# Via AWS CLI
aws codecommit create-pull-request \
  --title "Add payment gateway integration" \
  --description "Implements Stripe payment processing

  Changes:
  - Added Stripe SDK
  - Created payment service
  - Added webhook handlers
  - Updated checkout flow

  Testing:
  - Unit tests: 45 passing
  - Integration tests: 12 passing
  - Manual testing completed in staging" \
  --targets repositoryName=my-web-app,sourceReference=feature/payment,destinationReference=develop \
  --client-request-token $(uuidgen)

# Response includes PR ID for tracking
```

**Pull Request with Approval Rules:**

```json
{
  "approvalRuleName": "Production-Merge-Requirements",
  "approvalRuleContent": {
    "Version": "2018-11-08",
    "DestinationReferences": ["refs/heads/main"],
    "Statements": [
      {
        "Type": "Approvers",
        "NumberOfApprovalsNeeded": 2,
        "ApprovalPoolMembers": [
          "arn:aws:iam::123456789012:user/tech-lead",
          "arn:aws:iam::123456789012:user/senior-dev-1",
          "arn:aws:iam::123456789012:user/senior-dev-2"
        ]
      }
    ]
  }
}

// Meaning:
// - Merges to 'main' require 2 approvals
// - Approvals must come from specified pool
// - PR cannot be merged until requirements met
```

---

### Triggers & Notifications

**What Are Triggers?**
Triggers are **automated actions** that execute when repository events occur (push, PR creation, etc.).

**Common Trigger Use Cases:**

**1. Trigger CodePipeline on Push:**
```yaml
Trigger: Branch Update
Event: Push to 'main' branch
Action: Start CodePipeline deployment

Configuration:
  Repository: my-web-app
  Branch: main
  Target: CodePipeline (MyAppPipeline)

Result:
  Developer pushes to main
  → CodePipeline automatically starts
  → Build → Test → Deploy to production
```

**2. Trigger Lambda for Custom Validation:**
```javascript
// Lambda function triggered on PR creation
exports.handler = async (event) => {
  const pr = event.detail.pullRequestId;
  const repo = event.detail.repositoryName;

  // Custom validation: Check commit message format
  const commits = await getCommits(repo, pr);

  for (const commit of commits) {
    if (!commit.message.match(/^(feat|fix|docs|style|refactor|test|chore):/)) {
      // Post comment on PR
      await postComment(repo, pr,
        "❌ Commit message must follow conventional commits format");
      return { statusCode: 400 };
    }
  }

  await postComment(repo, pr, "✓ All commit messages follow conventions");
  return { statusCode: 200 };
};
```

**3. Send Notifications:**
```yaml
Trigger: Pull Request State Change
Events:
  - PR Created
  - PR Updated
  - PR Merged
  - PR Closed

Action: Publish to SNS Topic

SNS Topic Subscriptions:
  - Email: dev-team@company.com
  - Slack: #code-reviews channel
  - Lambda: Custom notification logic

Message Format:
  Subject: "[CodeCommit] Pull Request #42 Created"
  Body:
    Repository: my-web-app
    PR: #42
    Title: "Add payment gateway"
    Author: john.doe
    Branch: feature/payment → develop
    URL: https://console.aws.amazon.com/...
```

**Creating a Trigger:**

```bash
# Trigger Lambda on push
aws codecommit put-repository-triggers \
  --repository-name my-web-app \
  --triggers '[
    {
      "name": "ValidateCommits",
      "destinationArn": "arn:aws:lambda:us-east-1:123456789012:function:ValidateCommits",
      "events": ["all"],
      "branches": ["main", "develop"]
    }
  ]'

# Trigger SNS on PR events
aws codecommit put-repository-triggers \
  --repository-name my-web-app \
  --triggers '[
    {
      "name": "PRNotifications",
      "destinationArn": "arn:aws:sns:us-east-1:123456789012:CodeCommitPRs",
      "events": ["pullRequestCreated", "pullRequestStatusChanged"],
      "branches": []
    }
  ]'
```

**Key Concepts:**
- CodeCommit is **fully managed Git** (no servers to maintain)
- **IAM-based authentication** (no separate user management)
- **Pull requests** enable code review workflows
- **Triggers** automate actions on repository events
- **Integrated** with CodePipeline, CodeBuild, Lambda
- **Cost-effective**: $1/user/month (first 5 free)
- **Secure**: Encrypted at rest and in transit
- Standard Git operations work identically

---


## Summary of Part 2

**Covered in this part:**

### CodeDeploy (Complete)
✓ Deployment Groups - Target selection strategies
✓ Deployment Configurations - Speed vs. safety tradeoffs
✓ AppSpec File - Complete deployment instructions
✓ Deployment Types - In-Place vs. Blue/Green comparison
✓ Rollback Configuration - Automatic failure recovery
✓ Lifecycle Hooks - Advanced deployment patterns

### CodeCommit (Complete)
✓ Repositories - Git repository management
✓ Branches - Branching strategies
✓ Pull Requests - Code review workflows
✓ Triggers & Notifications - Automation

---

## Coming in Part 3:
- CodeArtifact (Package management)
- CloudFormation (Infrastructure as Code)
- CDK (Cloud Development Kit)
- Terraform on AWS

---

**Guide Statistics:**
- Pages: ~85 (estimated when printed)
- Code Examples: 100+
- Diagrams: 50+
- Real-world scenarios: 30+