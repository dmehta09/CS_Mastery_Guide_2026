# AWS Auto Scaling - Conceptual Deep Dive

## Overview

**Auto Scaling** is AWS's service for automatically adjusting the number of EC2 instances based on demand. Think of it as your application's "automatic capacity manager" - it adds servers when traffic increases and removes them when traffic decreases.

**The core problem Auto Scaling solves**:
- **Over-provisioning**: Running 50 servers 24/7 when you only need them during business hours → wasting money
- **Under-provisioning**: Running 10 servers when you suddenly get 10× traffic → site crashes, revenue loss
- **Manual scaling**: Having engineers wake up at 3 AM to launch servers → slow, error-prone, expensive

**Auto Scaling provides**:
- **Elasticity**: Capacity matches demand automatically
- **Cost optimization**: Pay only for what you need, when you need it
- **Reliability**: Replace unhealthy instances automatically
- **Performance**: Maintain consistent application performance under varying load

---

## Auto Scaling Groups (ASG)

**What an ASG is**: A logical grouping of EC2 instances that Auto Scaling manages as a single unit. The ASG defines:
- **Minimum capacity**: Never go below this (e.g., 2 instances for basic availability)
- **Maximum capacity**: Never exceed this (cost protection)
- **Desired capacity**: Target number of instances (Auto Scaling adjusts to meet this)

### ASG Core Concepts

**Configuration elements**:
```
Auto Scaling Group: "web-app-asg"
├─ Launch Template: "web-app-lt-v5" (defines what to launch)
├─ Min: 2 instances (always running, even at night)
├─ Max: 20 instances (cost ceiling, never exceed)
├─ Desired: 5 instances (current target, changes with scaling policies)
├─ VPC + Subnets: [subnet-a, subnet-b, subnet-c] (where to launch)
├─ Load Balancer: "web-app-alb" (register instances here)
├─ Health Check: ELB + EC2 (how to determine if instance is healthy)
└─ Scaling Policies: CPU > 70% → scale out (when to add/remove)
```

**How ASG maintains desired capacity**:
```
Scenario 1: Initial launch
- Desired: 5
- Current: 0
- Action: Launch 5 instances across subnets

Scenario 2: Instance fails health check
- Desired: 5
- Current: 5, but 1 unhealthy
- Action: Terminate unhealthy, launch replacement → maintain 5 healthy

Scenario 3: Scaling policy triggers (high CPU)
- Desired: 5 → Updated to 8 (by scaling policy)
- Current: 5
- Action: Launch 3 new instances

Scenario 4: Manual adjustment
- Admin changes desired to 3
- Current: 8
- Action: Terminate 5 instances (oldest first by default)
```

### Multi-AZ Distribution

**Why multi-AZ matters**: Single AZ failure shouldn't take down your application.

**How ASG distributes instances**:
```
ASG configured with 3 subnets:
- us-east-1a (Subnet A)
- us-east-1b (Subnet B)
- us-east-1c (Subnet C)

Desired capacity: 6 instances
Default distribution: 2 instances per AZ (balanced)

AZ-a: Instance 1, Instance 2
AZ-b: Instance 3, Instance 4
AZ-c: Instance 5, Instance 6

If AZ-a fails:
- Instances 1 & 2 lost (detected via health checks)
- ASG launches 2 replacement instances in AZ-b and AZ-c
- Application continues running (4 instances during recovery, then 6)
```

**Rebalancing**: If imbalance occurs (e.g., 3 instances in AZ-a, 2 in AZ-b, 1 in AZ-c), ASG gradually rebalances to achieve equal distribution.

### Health Checks: Determining Instance Health

**ASG supports two health check types**:

**1. EC2 Health Check** (Default)
- **What it checks**: Is the instance running? (AWS's view)
- **Failure detection**: Instance stopped, terminated, or impaired hardware
- **Limitation**: Doesn't detect application failures (app crashed but instance still running)

**2. ELB Health Check** (Recommended for load-balanced apps)
- **What it checks**: Is the application responding? (application-level)
- **How it works**: Load balancer sends HTTP/HTTPS requests to instance
  - Response 200 OK → Healthy
  - Timeout or 500 error → Unhealthy
- **Benefit**: Detects application failures (web server crashed, app hung, database connection lost)

**Health check configuration**:
```
ELB Health Check Settings:
- Target: HTTP /health or /
- Interval: 30 seconds (check every 30s)
- Timeout: 5 seconds (wait max 5s for response)
- Healthy threshold: 2 (2 consecutive successes = healthy)
- Unhealthy threshold: 3 (3 consecutive failures = unhealthy)

Timeline example:
00:00 - Check 1: Timeout (0/3 unhealthy)
00:30 - Check 2: Timeout (1/3 unhealthy)
01:00 - Check 3: Timeout (2/3 unhealthy)
01:30 - Check 4: Timeout (3/3 unhealthy) → Mark unhealthy
01:31 - ASG detects unhealthy instance
01:32 - ASG terminates instance, launches replacement
```

**Grace period**: After launching new instance, ASG waits before performing health checks (default: 300 seconds). Gives application time to boot and become ready.

### Instance Replacement Behavior

**When ASG replaces instances**:
1. Health check fails (EC2 or ELB)
2. Manual termination via API
3. Spot instance interrupted
4. AZ availability issues

**Replacement process**:
```
Step 1: Detect unhealthy instance (Instance A)
Step 2: Mark for termination
Step 3: Launch new instance (Instance B)
Step 4: Wait for Instance B to pass health checks
Step 5: Register Instance B with load balancer
Step 6: Terminate Instance A

Graceful shutdown (if configured):
- Load balancer stops sending new connections to Instance A
- Existing connections allowed to complete (drain timeout: 300s default)
- After drain timeout, force terminate
```

---

## Launch Configurations vs Launch Templates

**Both define HOW to launch instances** (AMI, instance type, security groups, etc.). But Launch Templates are superior in every way.

### Launch Configurations (Legacy, Deprecated)

**Characteristics**:
- **Immutable**: Can't modify after creation (must create new version)
- **Single instance type**: Can only specify one type (e.g., m5.large)
- **ASG only**: Can only be used with Auto Scaling Groups
- **Missing features**: No T2/T3 unlimited mode, no spot/on-demand mix, no multiple network interfaces

**In 2026**: AWS recommends migrating all Launch Configurations to Launch Templates. Support will eventually be removed.

### Launch Templates (Modern, Recommended)

**Why Launch Templates are better**:

**1. Versioning** (vs immutable)
```
Launch Template: "web-app-lt"
├─ Version 1: m5.large, AMI-v1.0
├─ Version 2: m5.large, AMI-v1.1 (updated AMI)
├─ Version 3: m6i.large, AMI-v1.2 (upgraded instance type + AMI)
└─ Version 4: m7g.large, AMI-v1.3 (Graviton + latest AMI)

ASG can use v3 (stable) while testing v4 in dev
Easy rollback: v4 has issues → revert ASG to v3 in one click
```

**2. Multiple Instance Types** (vs single type)
```
Launch Template: Supports multiple types
Instance types: [m7g.large, m6i.large, m5.large]
Priority: Prefer m7g.large (cheapest), fallback to others

Benefits:
- Flexibility: If m7g capacity unavailable, use m6i
- Cost optimization: Mix Spot (cheap) and On-Demand (reliable)
- Availability: More instance type options = higher launch success rate
```

**3. Spot + On-Demand Mix** (vs On-Demand only)
```
ASG with Launch Template:
├─ 30% On-Demand (baseline, always available)
└─ 70% Spot (cost-optimized, interruption-tolerant)

Cost savings: 50-60% overall (Spot is 70% cheaper than On-Demand)
Reliability: On-Demand provides stable baseline
```

**4. Multiple Network Interfaces** (advanced)
```
Launch Template: Can specify multiple ENIs
- ENI 1: Application traffic
- ENI 2: Management/monitoring traffic

Use case: Network appliances, security tools
```

**5. Reusable** (vs ASG-only)
```
Same Launch Template used for:
├─ Auto Scaling Group (automatic scaling)
├─ Manual instance launch (testing)
└─ Spot Fleet (batch processing)

Single source of truth for instance configuration
```

### Migration Path: Launch Configuration → Launch Template

**AWS provides automatic migration**:
```
1. Select Launch Configuration in console
2. Click "Copy to Launch Template"
3. Review generated Launch Template
4. Update ASG to use new Launch Template
5. Test scaling operations
6. Delete old Launch Configuration
```

---

## Scaling Policies: When to Add/Remove Instances

**Scaling policies define the "when" and "how much" to scale**. Three main types:

### 1. Target Tracking Scaling (Simplest, Recommended)

**Concept**: "Keep this metric at this target value" - ASG automatically adds/removes instances to maintain the target.

**How it works**:
```
Policy: "Keep average CPU utilization at 60%"

Scenario 1: CPU rises to 80%
- ASG calculates: Need more instances to bring CPU down to 60%
- Action: Add instances

Scenario 2: CPU drops to 30%
- ASG calculates: Too many instances, scale in to reach 60%
- Action: Remove instances (gradually, cooldown periods)

Scenario 3: CPU stays at 58-62%
- Within tolerance → No action
```

**Common target tracking metrics**:

**CPU Utilization** (most common)
```
Target: 60% average CPU
Why 60%?: Leaves headroom for spikes (40% capacity buffer)
Use case: CPU-bound apps (web servers, API servers)

Policy creation:
- Metric: CPUUtilization
- Target: 60
- Result: ASG maintains CPU around 60% by adjusting instance count
```

**Request Count Per Target** (load-balanced apps)
```
Target: 1000 requests/minute per instance
Calculation:
- Current load: 5000 requests/minute
- Target per instance: 1000
- Required instances: 5

Auto-adjusts:
- Load increases to 8000 req/min → Scale to 8 instances
- Load drops to 2000 req/min → Scale to 2 instances

Use case: HTTP applications where request count = good load indicator
```

**Network In/Out**
```
Target: 50 MB/s average network throughput
Use case: Network-intensive apps (streaming, file transfer)
```

**Custom CloudWatch Metrics**
```
Target: 100 active database connections per instance
Your application publishes: Current connection count to CloudWatch
ASG scales based on your custom metric

Use case: Application-specific load indicators
```

**Benefits of Target Tracking**:
- **Simple**: One configuration value (target), ASG figures out the rest
- **Self-adjusting**: No need to define "add 2 instances" rules
- **Predictable**: Maintains consistent performance
- **Multiple metrics**: Can have multiple target tracking policies (ASG uses most aggressive)

**Limitations**:
- **Scale-in delay**: Conservative scale-in (waits to ensure sustained low load)
- **Can't handle complex logic**: If you need "scale differently on weekdays vs weekends", use scheduled or step scaling

---

### 2. Step Scaling (Granular Control)

**Concept**: Define specific actions based on metric ranges (steps).

**How it works**:
```
Alarm: High CPU (> 60%)
Step 1: CPU 60-70% → Add 1 instance
Step 2: CPU 70-85% → Add 2 instances
Step 3: CPU 85-100% → Add 4 instances

Alarm: Low CPU (< 30%)
Step 1: CPU 20-30% → Remove 1 instance
Step 2: CPU 10-20% → Remove 2 instances
Step 3: CPU < 10% → Remove 3 instances
```

**Real-world example - E-commerce during flash sale**:
```
Scale Out Policy:
- CPU 50-60%: +2 instances (moderate load)
- CPU 60-70%: +5 instances (significant load)
- CPU 70-80%: +10 instances (heavy load)
- CPU 80%+: +20 instances (emergency capacity)

Why steps?: Aggressive scaling during flash sale to prevent outages
```

**Step Adjustments - Three modes**:

**1. ChangeInCapacity** (add/remove exact number)
```
Current: 10 instances
Action: Add 3 instances
Result: 13 instances
```

**2. ExactCapacity** (set to exact number)
```
Current: 10 instances
Action: Set to 20 instances
Result: 20 instances (regardless of current)
```

**3. PercentChangeInCapacity** (adjust by percentage)
```
Current: 10 instances
Action: Add 50%
Result: 15 instances (10 + 50% of 10)

Min increment: Round up (10.5 → 11)
```

**When to use Step Scaling**:
- Need precise control over scaling speed
- Different urgency at different thresholds
- Complex scaling logic based on metric ranges
- Combining multiple metrics (CPU + memory + requests)

---

### 3. Simple Scaling (Legacy, Not Recommended)

**Concept**: Single action based on single alarm.

```
Alarm: CPU > 70%
Action: Add 2 instances
Cooldown: Wait 300 seconds before next scaling action
```

**Why it's inferior to Step Scaling**:
- **Single action**: Can't have nuanced responses (CPU 75% vs 95% → same action)
- **Cooldown required**: Must wait even if additional scaling needed urgently
- **Less responsive**: Can't react quickly to rapidly changing load

**In 2026**: Use Target Tracking or Step Scaling instead. Simple Scaling exists for legacy compatibility only.

---

### 4. Predictive Scaling (ML-Powered)

**Concept**: AWS uses machine learning to predict future load based on historical patterns and schedules instances proactively.

**How it works**:
```
Historical data (2 weeks minimum):
- Mon-Fri 9 AM: Traffic increases
- Mon-Fri 6 PM: Traffic decreases
- Saturday: 50% lower traffic
- Sunday: 70% lower traffic

Predictive Scaling learns:
- Patterns repeat weekly
- Ramp-up takes 10 minutes

Predictive Action:
- Thursday 8:50 AM: Pre-launch instances (before 9 AM traffic)
- By 9:00 AM: Capacity ready (no lag, no performance degradation)
```

**The problem Predictive Scaling solves**:

**Without predictive**:
```
9:00 AM: Traffic spike arrives
9:00 AM: Reactive scaling policy triggers
9:02 AM: New instances launching
9:05 AM: Instances ready, registered with LB
9:05-9:00 = 5 minutes of degraded performance
```

**With predictive**:
```
8:50 AM: Predictive Scaling launches instances proactively
9:00 AM: Traffic spike arrives
9:00 AM: Capacity already available → no performance impact
```

**Configuration**:
```
Predictive Scaling Policy:
- Metric: CPUUtilization
- Target: 60%
- Mode: "Forecast and scale" (vs "Forecast only" for testing)
- Schedule: Scale out 10 minutes before predicted load
```

**Use cases**:
- **Daily patterns**: Office hours traffic (9 AM - 6 PM spikes)
- **Weekly patterns**: Higher weekend traffic for consumer apps
- **Seasonal**: Retail apps (holiday shopping patterns)
- **Event-driven**: Sports streaming (game times predictable)

**Requirements**:
- Minimum 2 weeks of historical data
- Consistent patterns (not random traffic)
- Target Tracking policy also configured (Predictive augments reactive scaling)

**Benefits**:
- **Zero-lag scaling**: Capacity ready before demand arrives
- **Better performance**: Users never experience degradation
- **Cost optimization**: Scale in predictively too (remove capacity before low-traffic periods)

**Limitations**:
- Requires predictable patterns (doesn't help with random viral traffic)
- Takes 2 weeks to learn patterns (not instant)
- Works best combined with reactive policies (Target Tracking as backup)

---

### 5. Scheduled Scaling

**Concept**: Scale at specific times based on a schedule (no metrics, just time-based).

**How it works**:
```
Scheduled Action 1:
- Name: "morning-scale-out"
- Time: Monday-Friday, 8:30 AM
- Action: Set desired capacity to 20

Scheduled Action 2:
- Name: "evening-scale-in"
- Time: Monday-Friday, 7:00 PM
- Action: Set desired capacity to 5

Scheduled Action 3:
- Name: "weekend-baseline"
- Time: Saturday-Sunday
- Action: Set desired capacity to 3
```

**When to use Scheduled Scaling**:

**Use Case 1: Predictable Business Hours**
```
B2B SaaS application:
- Customers are businesses (use during work hours)
- Traffic pattern: High 9AM-5PM, dead overnight
- Solution: Schedule 15 instances during business hours, 3 overnight
- Savings: 60% cost reduction vs running 15 instances 24/7
```

**Use Case 2: Batch Processing Windows**
```
Nightly ETL jobs:
- Job runs: 2 AM - 4 AM daily
- Solution:
  - 1:50 AM: Scale to 50 instances
  - 4:10 AM: Scale back to 5 instances
- Benefit: High capacity during job, minimal cost otherwise
```

**Use Case 3: Known Events**
```
TV broadcast app:
- Popular show airs: Thursdays 8 PM - 9 PM
- Solution:
  - Thursday 7:50 PM: Scale to 100 instances
  - Thursday 9:30 PM: Scale to 20 instances
- Benefit: Ready for predictable surge
```

**Use Case 4: Testing/Development Environments**
```
Dev environment:
- Developers work: Monday-Friday, 8 AM - 6 PM
- Solution:
  - Weekdays 7:30 AM: Scale to 10 instances
  - Weekdays 6:30 PM: Scale to 0 instances (terminate all)
  - Weekends: Keep at 0
- Savings: 72% (50 hours vs 168 hours/week)
```

**Schedule syntax options**:

**Cron Expression** (flexible)
```
0 8 * * MON-FRI (8 AM weekdays)
0 20 * * * (8 PM daily)
0 */4 * * * (Every 4 hours)
```

**Recurring schedule**
```
Frequency: Daily, Weekly, Monthly
Start time: 08:00
End time: (optional, or use second scheduled action)
```

**One-time schedule**
```
Date: 2026-11-29 (Black Friday)
Time: 00:00
Action: Set desired to 200 (prepare for massive traffic)
```

**Combining scheduled with other policies**:
```
Scheduled Action (8 AM): Set desired capacity to 20
Target Tracking Policy: Keep CPU at 60%

Result:
- 8 AM: Scheduled action sets baseline to 20 instances
- If load exceeds capacity → Target Tracking adds more instances
- If load drops → Target Tracking removes instances (but respects min capacity)

Best of both worlds: Scheduled baseline + dynamic adjustment
```

---

## Lifecycle Hooks: Custom Actions During Scaling

**What Lifecycle Hooks are**: Pause points in the instance launch/termination process that let you run custom actions before instances enter service or are terminated.

**Why they exist**: Sometimes you need to do more than just "launch instance and add to load balancer":
- Download large datasets before serving traffic
- Deregister from service discovery
- Backup local data before termination
- Run custom health checks
- Register with external monitoring systems

### Lifecycle Hook Stages

**Launch Lifecycle**:
```
Instance launching
   ↓
Pending → (Lifecycle Hook: Pending:Wait)
   ↓ [Your custom actions run here - up to 2 hours]
   ↓
   ↓ COMPLETE or ABANDON
   ↓
InService (receiving traffic) or Terminated (if ABANDON)
```

**Termination Lifecycle**:
```
Instance being terminated
   ↓
Terminating → (Lifecycle Hook: Terminating:Wait)
   ↓ [Your custom actions run here - up to 2 hours]
   ↓ Complete log shipping, backup data, graceful shutdown
   ↓
   ↓ COMPLETE or CONTINUE
   ↓
Terminated
```

### Real-World Lifecycle Hook Use Cases

**Use Case 1: Pre-Loading Cache Before Serving Traffic**
```
Problem: Redis cache server takes 5 minutes to load data from database

Without Lifecycle Hook:
- Instance launches
- Immediately added to load balancer
- Cold cache → slow queries for 5 minutes

With Lifecycle Hook (Launch):
1. Instance launches → Enters "Pending:Wait"
2. Hook triggers Lambda function
3. Lambda runs script on instance: "warm up cache"
4. Script loads data from database into Redis
5. After 5 minutes, script completes
6. Lambda signals: COMPLETE
7. Instance enters "InService" → Fully warmed cache
```

**Lambda function example**:
```python
import boto3

def lambda_handler(event, context):
    # Instance launched, in Pending:Wait state
    instance_id = event['detail']['EC2InstanceId']

    # Run SSM command on instance to warm cache
    ssm = boto3.client('ssm')
    ssm.send_command(
        InstanceIds=[instance_id],
        DocumentName='AWS-RunShellScript',
        Parameters={'commands': ['/opt/app/warm-cache.sh']}
    )

    # Script completion triggers another Lambda to signal COMPLETE
    # (or use timeout + auto-complete)
```

**Use Case 2: Graceful Shutdown - Draining Connections**
```
Problem: Terminate instance immediately → active user sessions lost

With Lifecycle Hook (Termination):
1. ASG decides to terminate instance
2. Instance enters "Terminating:Wait"
3. Hook triggers notification (SNS, Lambda, EventBridge)
4. Custom script on instance:
   - Stop accepting new connections
   - Wait for existing connections to finish (up to 10 minutes)
   - Flush logs to S3
   - Backup any local state
5. After completion, signal COMPLETE
6. Instance terminates

Result: Zero dropped user sessions
```

**Use Case 3: Backup Local Data Before Termination**
```
Instance with local SSD (instance store):
- Application writes temporary results to local disk
- Need to backup to S3 before termination

Termination Hook:
1. Instance entering termination
2. Hook pauses termination
3. Script runs: aws s3 sync /local-data/ s3://backup-bucket/
4. After sync completes, signal COMPLETE
5. Instance terminates (data safe in S3)
```

**Use Case 4: Service Discovery Registration/Deregistration**
```
Launch Hook:
- Register instance with Consul/Etcd service discovery
- Update DNS records
- Signal COMPLETE when registration confirmed

Termination Hook:
- Deregister from service discovery
- Drain connections
- Signal COMPLETE after clean removal
```

### Configuring Lifecycle Hooks

**Hook configuration**:
```
Lifecycle Hook: "launch-cache-warm"
├─ Lifecycle Transition: Instance Launch (autoscaling:EC2_INSTANCE_LAUNCHING)
├─ Heartbeat Timeout: 600 seconds (10 minutes max wait)
├─ Default Result: ABANDON (if timeout, don't add instance)
├─ Notification Target: SNS Topic or EventBridge
└─ Role: IAM role for publishing notifications

Process:
1. Instance launches
2. ASG publishes event to SNS/EventBridge
3. Lambda subscribes to SNS, receives notification
4. Lambda runs custom logic
5. Lambda calls complete_lifecycle_action API
6. ASG proceeds with instance launch
```

**Heartbeat timeout**:
- Max time instance can stay in "Wait" state: 2 hours (7200 seconds)
- If timeout expires without COMPLETE/ABANDON signal → Default Result applies
- Can extend timeout by sending "heartbeat" (for long-running operations)

**Default result**:
- **CONTINUE**: If timeout, proceed anyway (for non-critical hooks)
- **ABANDON**: If timeout, terminate instance (for critical operations like cache warming)

### Lifecycle Hook Best Practices

**1. Set Appropriate Timeouts**
```
Fast operations (register with service): 60 seconds
Medium operations (warm cache): 300 seconds
Slow operations (backup data): 1800 seconds

Don't: Set 2-hour timeout for 30-second operation (delays scaling)
```

**2. Implement Idempotency**
```
Hook might trigger multiple times (retries, failures)
Your script should handle: "Already warmed cache, skipping"
```

**3. Monitor Hook Completions**
```
CloudWatch Metrics:
- Hooks entering Wait state
- Hooks completing
- Hooks timing out

Alert: If >10% hooks timeout → investigate (script failure?)
```

**4. Test Hook Failures**
```
What happens if your script crashes?
- Timeout expires → Default Result applies
- Ensure Default Result is sensible
- Have monitoring to detect failures
```

**5. Use EventBridge Over SNS** (Modern approach)
```
EventBridge benefits:
- Filter events precisely
- Route to multiple targets
- Built-in retry logic
- Better integration with AWS services

Example EventBridge rule:
Event Pattern:
{
  "source": ["aws.autoscaling"],
  "detail-type": ["EC2 Instance-launch Lifecycle Action"],
  "detail": {
    "AutoScalingGroupName": ["web-app-asg"]
  }
}
Target: Lambda function
```

---

## Warm Pools: Pre-Initialized Instances

**What Warm Pools are**: A pool of pre-initialized instances kept in a stopped state, ready to be quickly started when scaling out.

**The problem Warm Pools solve**:
```
Traditional scaling timeline:
1. Scaling policy triggers (0:00)
2. Launch new instance (0:00 - 0:30) - Slow: boot OS, run user data
3. Application initialization (0:30 - 2:00) - Slow: download code, warm caches
4. Health checks pass (2:00 - 2:30)
5. Instance in service (2:30) - Total: 2.5 minutes

Problem: 2.5 minutes of degraded performance during scale-out
```

**With Warm Pool**:
```
Pre-initialization (done ahead of time):
1. Launch instances, run user data, initialize app
2. Stop instances → Add to Warm Pool
3. Instances sitting in "stopped" state (no compute cost)

Scale-out with Warm Pool:
1. Scaling policy triggers (0:00)
2. Start instance from Warm Pool (0:00 - 0:20) - Fast: already initialized
3. Health checks pass (0:20 - 0:30)
4. Instance in service (0:30) - Total: 30 seconds

Result: 5× faster scale-out
```

### Warm Pool States

**Stopped** (default, recommended):
- Instance fully stopped (like EBS-backed instance stop)
- Cost: Only EBS storage ($0.08/GB/month), no compute cost
- Start time: 30-60 seconds
- **Best for most use cases**: Balance of cost and speed

**Running**:
- Instance running but not in service (not receiving load balancer traffic)
- Cost: Full compute cost (same as InService instance)
- Start time: Instant (already running, just add to LB)
- **Use case**: Absolutely need instant capacity (financial trading, gaming launches)

**Hibernated**:
- Instance hibernated (RAM saved to EBS)
- Cost: EBS storage only (no compute)
- Start time: 60-90 seconds (restore RAM from disk)
- **Use case**: Applications with long initialization (large caches in RAM)

### Warm Pool Configuration

```
Warm Pool Settings:
├─ Minimum Size: 5 (always keep 5 pre-initialized instances)
├─ Max Prepared Capacity: 10 (never exceed 10 warm instances)
├─ Instance State: Stopped
└─ Instance Reuse Policy: Yes (return instances to pool after scale-in)

Combined with ASG:
├─ ASG Min: 10 (10 instances always InService)
├─ ASG Desired: 10
├─ ASG Max: 50
└─ Warm Pool: 5 instances pre-initialized

Total capacity:
- InService: 10 instances (serving traffic)
- Warm Pool: 5 instances (ready to go)
- Can scale to: 50 instances max
```

### Real-World Warm Pool Use Cases

**Use Case 1: Gaming - Server Launch Events**
```
Game DLC release at 12:00 PM:
- Normal load: 20 servers
- Launch day load: 200 servers needed
- Initialization time: 5 minutes per server (download 10 GB game assets)

Without Warm Pool:
- 12:00: Traffic spike
- 12:00-12:05: Launching 180 servers (staggered)
- 12:05-12:30: Servers slowly coming online
- Result: 30 minutes of degraded experience

With Warm Pool (100 pre-initialized servers):
- 11:50 AM: Pre-warm 100 instances (cost: $80/month storage only)
- 12:00 PM: Traffic spike
- 12:01 PM: 100 warm instances in service
- 12:02-12:10: Additional 80 instances launched normally
- Result: Most users served immediately, minimal degradation
```

**Use Case 2: Machine Learning Inference**
```
ML model serving:
- Initialization: 3 minutes (download 2 GB model, load into memory)
- Traffic pattern: Unpredictable spikes

Warm Pool Configuration:
- Keep 10 instances pre-initialized with model loaded
- Cost: 10 instances × 30 GB EBS = $24/month storage
- Benefit: Instant capacity for inference spikes
```

**Use Case 3: Enterprise Application with Heavy Bootstrap**
```
Java application:
- User data: Install Java, download 500 MB JAR, configure
- Application startup: Load 1 GB data into cache
- Total: 4 minutes

Warm Pool:
- Pre-initialize 20 instances
- Stopped state (no compute cost)
- Scale-out time: 30 seconds vs 4 minutes
- Business benefit: Better user experience during traffic spikes
```

### Warm Pool Best Practices

**1. Balance Cost vs Speed**
```
Stopped: $24/month for 10 instances (30 GB EBS each)
Running: $1,400/month for 10 instances (m5.large, 24/7)

Recommendation: Use Stopped unless sub-second scaling required
```

**2. Right-Size Warm Pool**
```
Too small: Not enough capacity for scale-out events
Too large: Wasting money on unused pre-initialized instances

Strategy:
- Analyze scaling patterns: How many instances added during typical scale-out?
- Set Warm Pool to 50-80% of typical scale-out amount
- Example: Usually scale out by 20 instances → Warm Pool of 10-15
```

**3. Instance Reuse**
```
Enable reuse: Instances scaled-in return to Warm Pool
- Scale out: 10 → 30 (take 20 from Warm Pool)
- Scale in: 30 → 15 (return 15 to Warm Pool)
- Benefit: Continuously recycled capacity

Disable reuse: Instances terminated on scale-in
- Use case: Instances accumulate state, prefer fresh instances
```

**4. Lifecycle Hooks + Warm Pool**
```
Combined power:
1. Instance launches, enters Warm Pool (stopped)
2. Lifecycle hook runs initialization script
3. After script completes, instance added to Warm Pool
4. On scale-out, start warm instance (already initialized)
5. Instance enters InService immediately

Result: Pre-initialized AND custom setup logic
```

---

## Instance Refresh: Safe Deployment of Configuration Changes

**What Instance Refresh is**: Automated, gradual replacement of all instances in an ASG with new instances (usually after updating Launch Template).

**The problem it solves**:
```
Scenario: Update Launch Template (new AMI with application v2.0)
Manual approach:
1. Update Launch Template
2. Terminate old instances one by one
3. ASG launches new instances with new template
4. Risk: If you terminate all at once → outage
5. Tedious: Manually terminating 50 instances slowly

Instance Refresh: Automated, safe, gradual rollout
```

### How Instance Refresh Works

```
ASG before refresh:
- 20 instances running with Launch Template v1 (app v1.0)

Trigger Instance Refresh:
- Use Launch Template v2 (app v2.0)
- Configuration: Replace 5 instances at a time, 60-second wait

Process:
Step 1: Terminate instances 1-5
Step 2: ASG launches 5 new instances (Launch Template v2)
Step 3: Wait for new instances to pass health checks
Step 4: Wait 60 seconds (warmup)
Step 5: Terminate instances 6-10
Step 6: Repeat until all 20 instances replaced

Result: Gradual rollout, zero downtime, automatic rollback if failures
```

### Instance Refresh Configuration

**Key settings**:

**1. Min Healthy Percentage**
```
Setting: 90% (default)
Meaning: Always keep at least 90% of desired capacity healthy

Example (10 instances, 90% min healthy):
- Desired: 10
- Min healthy: 9
- Process: Terminate 1, launch 1, wait for healthy, terminate next 1
- Never drops below 9 healthy instances

Setting: 50%
- More aggressive: Terminate 5, launch 5
- Faster deployment, but risk of degraded performance

Recommendation: 90% for production, 50% for dev/test
```

**2. Instance Warmup**
```
Setting: 60 seconds (default: use ASG health check grace period)
Meaning: After instance passes health checks, wait 60s before continuing

Purpose:
- Give instance time to fully warm up (caches, connections)
- Don't immediately load new instance heavily
- Prevents cascading failures

Example: Web server
- Health check passes at 0:30 (server responding)
- But cache still warming, responses slow
- Wait 60s (total 1:30) → Cache warm, full performance
- Now safe to replace next batch
```

**3. Checkpoint Percentages**
```
Setting: Pause at 25%, 50%, 75%
Meaning: Wait for manual approval before continuing

Use case: Risky deployment
- Rollout new version
- Check metrics at 25% (5 of 20 instances)
- If error rate increased → Rollback
- If metrics good → Approve, continue to 50%

Manual approval process:
1. Instance Refresh pauses
2. Engineer reviews metrics (errors, latency, CPU)
3. Engineer approves or cancels via console/API
4. If approved, continues; if canceled, rollback
```

**4. Rollback on Alarm**
```
Setting: CloudWatch Alarm (HTTP 5xx errors > threshold)
Behavior: If alarm triggers during refresh → Automatic rollback

Process:
- Deploying app v2.0 via Instance Refresh
- 10 instances replaced (v2.0), 10 still on v1.0
- v2.0 has bug → 5xx errors spike
- CloudWatch alarm triggers
- Instance Refresh automatically cancels
- ASG terminates v2.0 instances, launches v1.0 instances
- Rollback to safe state

Benefit: Automatic safety net for bad deployments
```

### Real-World Instance Refresh Use Cases

**Use Case 1: Application Deployment**
```
Deploying new application version:
1. Build new AMI with app v2.0
2. Create Launch Template v2 with new AMI
3. Start Instance Refresh:
   - Min healthy: 90%
   - Warmup: 120 seconds
   - Checkpoints: 25%, 50%
   - Rollback alarm: HTTP 5xx errors
4. Monitor deployment:
   - 25%: Check metrics → Approve
   - 50%: Check metrics → Approve
   - 100%: Deployment complete

If any issues: Automatic rollback or manual cancel
```

**Use Case 2: Security Patching**
```
Monthly OS security patches:
1. Create new AMI with latest patches
2. Update Launch Template
3. Schedule Instance Refresh:
   - Saturday 2 AM (low traffic)
   - Min healthy: 80% (can tolerate more aggression at low traffic)
   - Duration: 2 hours
4. Automated, safe patching of entire fleet

Benefit: No manual SSH to each server, consistent patching
```

**Use Case 3: Infrastructure Upgrade**
```
Upgrading instance types (m5.large → m7g.large):
1. Update Launch Template: Change instance type
2. Start Instance Refresh
3. Gradual replacement: Old instances → New instances
4. Result: Zero downtime upgrade to Graviton (20% cost savings)
```

### Instance Refresh Best Practices

**1. Always Set Rollback Alarms**
```
Critical alarms:
- HTTP 5xx errors > 1%
- Application errors > threshold
- Average latency > 500ms
- Health check failures > 5%

If any alarm triggers → Rollback automatically
```

**2. Test in Non-Prod First**
```
Process:
1. Refresh dev environment (validate process works)
2. Refresh staging environment (validate app works)
3. Refresh production (confident in process + app)
```

**3. Use Checkpoints for High-Risk Changes**
```
Major version upgrades: Checkpoint at 10%, 25%, 50%
Minor patches: No checkpoints (fully automated)
```

**4. Monitor Key Metrics During Refresh**
```
Dashboard during refresh:
- Instance Refresh progress (%)
- Healthy instance count
- Error rates (5xx, 4xx)
- Latency (p50, p99)
- Request rate

Watch for: Degradation as new instances roll out
```

**5. Cancel If Issues Detected**
```
Instance Refresh is running, but:
- Error rates increasing
- Latency spiking
- User complaints

Action: Cancel refresh immediately via console/API
Result: Stops further replacements, keeps mix of old/new instances
Manual action: Rollback by starting new refresh with old template
```

---

## Mixed Instances Policy: Optimizing Cost and Availability

**What Mixed Instances Policy is**: ASG feature that allows launching multiple instance types and purchasing options (On-Demand + Spot) to optimize cost and availability.

**Why it exists**: Relying on single instance type has problems:
- **Capacity risk**: m5.large unavailable in AZ → scale-out fails
- **Cost inefficiency**: Running 100% On-Demand when Spot could save 70%
- **Rigidity**: Locked into specific instance type (what if better option available?)

**Mixed Instances Policy solves**:
- **Flexibility**: Try multiple instance types
- **Cost optimization**: Mix Spot + On-Demand
- **Availability**: If one type unavailable, use another

### Core Concepts

**Instance Type Diversification**:
```
Mixed Instances Policy:
Instance types: [m7g.large, m6i.large, m5.large, m5a.large]
Priority: Try in order

ASG needs to launch 10 instances:
- Tries m7g.large first (cheapest, Graviton)
- If m7g capacity unavailable in AZ → tries m6i.large
- If m6i unavailable → tries m5.large
- Fallback to m5a.large

Result: Maximize chances of successful launch
```

**Spot + On-Demand Mix**:
```
Mixed Instances Policy:
- Base On-Demand Instances: 3 (always On-Demand)
- On-Demand Percentage Above Base: 30%

Scenario (Desired capacity: 10):
- First 3 instances: On-Demand (base)
- Remaining 7 instances: 30% On-Demand, 70% Spot
  - 2 instances On-Demand (30% of 7 = 2.1)
  - 5 instances Spot (70% of 7)

Total:
- 5 On-Demand (3 base + 2 from mix)
- 5 Spot

Cost comparison:
- All On-Demand: 10 × $0.10/hr = $1.00/hr
- Mixed (5 On-Demand + 5 Spot): (5 × $0.10) + (5 × $0.03) = $0.65/hr
- Savings: 35%
```

### Allocation Strategies

**How ASG chooses which instance types to launch**:

**1. lowest-price** (Maximum cost savings)
```
Strategy: Launch cheapest Spot instances
Instance types: [m5.large, c5.large, r5.large]
Spot prices: m5=$0.03, c5=$0.025, r5=$0.04

Result: ASG launches c5.large (cheapest)

Use case: Cost is primary concern, instances are fungible
Example: Batch processing, dev environments
```

**2. capacity-optimized** (Maximum availability)
```
Strategy: Launch instance types with most available capacity
AWS analyzes: Which instance types have deepest Spot capacity pools

Result: Launch types least likely to be interrupted

Use case: Availability matters more than cost
Example: Production apps on Spot (want minimal interruptions)
```

**3. capacity-optimized-prioritized** (Balance availability + preferences)
```
Strategy: Follow your priority order, but consider capacity
Your priority: [m7g.large, m6i.large, m5.large]

AWS: Checks capacity for each type
- m7g.large: Low capacity (likely interruptions)
- m6i.large: High capacity (stable)
- m5.large: Medium capacity

Result: Launch m6i.large (balance your preference + capacity)

Use case: Prefer specific types but want availability protection
```

**4. price-capacity-optimized** (Best balance, recommended)
```
Strategy: Optimize for both price and capacity (AWS recommended)
Considers:
- Spot prices (launch cheaper instances)
- Available capacity (avoid likely interruptions)

Result: Cheapest instances with good availability

Use case: Production workloads on Spot (default choice)
```

### Real-World Mixed Instances Examples

**Example 1: Cost-Optimized Web Application**
```
Mixed Instances Policy:
- Instance types: [m7g.large, m6i.large, m5.large]
- Allocation: price-capacity-optimized
- Base On-Demand: 5
- On-Demand %: 20%

Desired capacity: 25 instances
- 5 On-Demand (base)
- 20 additional: 20% On-Demand, 80% Spot
  - 4 On-Demand
  - 16 Spot

Total: 9 On-Demand + 16 Spot
Cost: $2,190/month On-Demand + $350/month Spot = $2,540
vs All On-Demand: $4,380/month
Savings: 42%

Availability: 16 Spot instances diversified across 3 instance types + 3 AZs = 9 Spot pools
Interruption risk: <1% (high diversification)
```

**Example 2: Batch Processing (Maximum Cost Savings)**
```
Mixed Instances Policy:
- Instance types: [m5.large, m5a.large, m5n.large, m4.large]
- Allocation: lowest-price
- Base On-Demand: 0 (100% Spot)
- On-Demand %: 0%

Workload: Fault-tolerant data processing
- Job interrupted → Requeued automatically
- Cost savings: 70-90% vs On-Demand

Risk mitigation:
- SQS queue holds jobs (interrupted jobs return to queue)
- Auto Scaling based on queue depth
- Multiple instance types = capacity flexibility
```

**Example 3: Production with High Availability Requirements**
```
Mixed Instances Policy:
- Instance types: [m7g.large, m7g.xlarge, m6i.large, m6i.xlarge]
- Allocation: capacity-optimized-prioritized
- Base On-Demand: 10 (50% of typical load)
- On-Demand %: 50%

Desired capacity: 20 instances
- 10 On-Demand (base)
- 10 additional: 50/50 split
  - 5 On-Demand
  - 5 Spot

Total: 15 On-Demand + 5 Spot
Benefit: Stable baseline (15 On-Demand), cost savings on burst capacity (5 Spot)
Interruption handling: Spot interruption → ASG launches On-Demand replacement
```

### Spot Instance Interruption Handling

**When Spot interrupted**:
```
1. AWS sends 2-minute warning (EC2 instance metadata)
2. ASG receives interruption notice
3. ASG immediately launches replacement (On-Demand or different Spot pool)
4. Load balancer drains connections from interrupted instance
5. After 2 minutes, instance terminated
6. Replacement instance takes over

Downtime: Minimal (replacement launching proactively)
```

**Capacity Rebalancing** (Proactive replacement):
```
Enable rebalancing recommendation:
- AWS predicts: This Spot instance likely to be interrupted soon
- ASG proactively launches replacement before interruption
- Gradual transition: New instance ready before old instance interrupted
- Result: Zero downtime from Spot interruptions

Trade-off: Slightly higher cost (brief period with extra instance)
Benefit: Near-zero impact from Spot interruptions
```

### Mixed Instances Best Practices

**1. Choose Similar Instance Types**
```
Good: [m7g.large, m6i.large, m5.large]
- Similar vCPU/RAM ratio
- Application performs consistently

Bad: [m5.large, c5.large, r5.large]
- Different characteristics (CPU vs memory-optimized)
- Application performance varies widely
```

**2. Balance Cost and Availability**
```
Development: 100% Spot (maximize savings)
Staging: 30% On-Demand, 70% Spot
Production: 50% On-Demand, 50% Spot (or 70/30 for critical apps)
```

**3. Diversify Across AZs + Instance Types**
```
Maximum diversification:
- 3 AZs × 4 instance types = 12 Spot pools
- Highly unlikely all 12 pools interrupted simultaneously
- Result: Stable Spot capacity
```

**4. Test Application on All Instance Types**
```
Before production:
- Launch test instances of each type in policy
- Verify application works correctly
- Check performance (especially if mixing x86 + Graviton)
```

**5. Monitor Spot Savings and Interruption Rate**
```
CloudWatch Metrics:
- Spot instance count
- On-Demand instance count
- Spot interruptions (count per day)

Target: <5% interruption rate
If higher: Add more instance types for diversification
```

---

## Summary: Auto Scaling Best Practices

**1. Use Launch Templates** (not Launch Configurations)
**2. Default to Target Tracking Scaling** (simplest, most effective)
**3. Enable Predictive Scaling** (for predictable patterns)
**4. Use Mixed Instances Policy** (cost + availability)
**5. Implement Lifecycle Hooks** (for custom initialization/cleanup)
**6. Consider Warm Pools** (for fast scale-out)
**7. Use Instance Refresh** (for safe deployments)
**8. Monitor key metrics**: Healthy instance count, scaling activity, costs
**9. Multi-AZ always** (resilience to AZ failures)
**10. Test scaling policies** (simulate load, verify behavior)

---

**Auto Scaling transforms EC2 from static capacity to dynamic, cost-optimized infrastructure that adapts to your needs in real-time.**
