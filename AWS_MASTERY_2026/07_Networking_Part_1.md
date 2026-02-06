# AWS Networking: Comprehensive Conceptual Guide

> **Purpose**: This guide explains AWS networking services and concepts with a focus on understanding what they are, why they exist, and what problems they solve in real-world cloud architectures.

---

## Table of Contents

1. [VPC (Virtual Private Cloud)](#vpc-virtual-private-cloud)
2. [Transit Gateway](#transit-gateway)
3. [Direct Connect](#direct-connect)
4. [Site-to-Site VPN](#site-to-site-vpn)
5. [Route 53](#route-53)
6. [CloudFront](#cloudfront)
7. [Global Accelerator](#global-accelerator)
8. [API Gateway](#api-gateway)

---

## VPC (Virtual Private Cloud)

### What is VPC?

**VPC (Virtual Private Cloud)** is your own isolated, private network within AWS. Think of it as having your own data center in the cloud, but without the physical hardware. It's a logically isolated section of the AWS cloud where you can launch AWS resources in a virtual network that you define.

### Why VPC Exists

**Problems it solves:**
- **Network Isolation**: Multiple customers need to run workloads on AWS without interfering with each other
- **Security Control**: Organizations need to control exactly who can access their resources and how
- **Custom Network Design**: Businesses need to replicate their on-premises network architecture in the cloud
- **IP Address Management**: Companies need control over their IP address ranges for compliance and integration

**Real-world scenario**: An e-commerce company wants to run their database in an isolated network segment that's not accessible from the internet, while their web servers need internet access. VPC makes this possible.

---

### CIDR Blocks

#### What are CIDR Blocks?

**CIDR (Classless Inter-Domain Routing)** is a way to specify a range of IP addresses. When you create a VPC, you assign it a CIDR block like `10.0.0.0/16`, which defines all possible IP addresses that can be used within that VPC.

#### Understanding CIDR Notation

```
10.0.0.0/16
│       └── Prefix length (how many bits are "fixed")
└── Base IP address

/16 means: First 16 bits are fixed, remaining 16 bits are variable
This gives: 2^16 = 65,536 possible IP addresses
```

**Common CIDR ranges:**
- `/16` → 65,536 IPs (e.g., `10.0.0.0/16` = `10.0.0.0` to `10.0.255.255`)
- `/24` → 256 IPs (e.g., `10.0.1.0/24` = `10.0.1.0` to `10.0.1.255`)
- `/28` → 16 IPs (e.g., `10.0.1.0/28` = `10.0.1.0` to `10.0.1.15`)

#### Why CIDR Matters

**Problems it solves:**
- **IP Planning**: Prevents IP address conflicts when connecting multiple networks
- **Scalability**: Larger CIDR blocks (/16) give more room for growth
- **Segmentation**: Smaller CIDR blocks (/24, /28) create isolated network segments

**Real-world scenario**: A company plans to have 100 subnets across multiple availability zones. They choose `10.0.0.0/16` for their VPC, which gives them 256 possible `/24` subnets (each with 256 IPs).

#### VPC CIDR Rules

```
Valid Private IP Ranges (RFC 1918):
┌─────────────────────────────────────┐
│ 10.0.0.0/8      (10.0.0.0 - 10.255.255.255)      │
│ 172.16.0.0/12   (172.16.0.0 - 172.31.255.255)    │
│ 192.168.0.0/16  (192.168.0.0 - 192.168.255.255)  │
└─────────────────────────────────────┘

AWS VPC CIDR Constraints:
- Minimum: /28 (16 IPs)
- Maximum: /16 (65,536 IPs)
- Can add secondary CIDR blocks later
```

---

### Subnets (Public and Private)

#### What are Subnets?

A **subnet** is a subdivision of your VPC's IP address range. It's a smaller network segment within your larger VPC network. Each subnet exists within a single Availability Zone.

```
VPC: 10.0.0.0/16
│
├── Subnet A (AZ-1): 10.0.1.0/24  (256 IPs)
├── Subnet B (AZ-1): 10.0.2.0/24  (256 IPs)
├── Subnet C (AZ-2): 10.0.3.0/24  (256 IPs)
└── Subnet D (AZ-2): 10.0.4.0/24  (256 IPs)
```

#### Public vs Private Subnets

The distinction between **public** and **private** subnets is not a checkbox or setting—it's based on **routing configuration**.

**Public Subnet:**
- Has a route to an Internet Gateway
- Resources can receive traffic directly from the internet (if they have public IPs)
- Use case: Web servers, load balancers, bastion hosts

**Private Subnet:**
- Does NOT have a direct route to an Internet Gateway
- Resources cannot be reached directly from the internet
- Can access internet via NAT Gateway/Instance
- Use case: Databases, application servers, backend processing

```
┌─────────────────────────────────────────────────┐
│                    Internet                      │
└────────────────┬────────────────────────────────┘
                 │
                 │
         ┌───────▼────────┐
         │ Internet Gateway│
         └───────┬────────┘
                 │
┌────────────────┼────────────────────────────────┐
│                VPC (10.0.0.0/16)                │
│                │                                 │
│  ┌─────────────┴──────────┐   ┌────────────┐   │
│  │  Public Subnet         │   │  Private   │   │
│  │  (10.0.1.0/24)         │   │  Subnet    │   │
│  │                        │   │(10.0.2.0/24)│  │
│  │  ┌──────────┐          │   │            │   │
│  │  │Web Server│◄─────────┼───│ No direct  │   │
│  │  │(Public IP)          │   │ internet   │   │
│  │  └──────────┘          │   │ route      │   │
│  │      ▲                 │   │            │   │
│  │      │ Internet Access │   │ ┌────────┐ │   │
│  │      │                 │   │ │Database│ │   │
│  └──────┼─────────────────┘   │ └────────┘ │   │
│         │                     └────────────┘   │
└─────────┼──────────────────────────────────────┘
          │
      (Public traffic)
```

#### Why Subnets Matter

**Problems they solve:**
- **Security Isolation**: Separate public-facing resources from sensitive backend systems
- **High Availability**: Distribute resources across multiple AZs for redundancy
- **Network Segmentation**: Organize resources by function, environment, or team
- **Compliance**: Meet regulatory requirements for network isolation

**Real-world scenario**: A three-tier application has:
- **Public subnets**: Load balancers (accept traffic from internet)
- **Private subnets (app tier)**: Application servers (accessed only by load balancers)
- **Private subnets (data tier)**: Databases (accessed only by app servers)

#### AWS Reserved IPs in Subnets

In every subnet, **AWS reserves 5 IP addresses**:

```
Subnet: 10.0.1.0/24 (256 total IPs)

Reserved:
10.0.1.0   → Network address
10.0.1.1   → VPC router
10.0.1.2   → DNS server
10.0.1.3   → Future use
10.0.1.255 → Broadcast address

Usable: 251 IPs (256 - 5)
```

---

### Route Tables

#### What are Route Tables?

A **route table** contains a set of rules (routes) that determine where network traffic from your subnet or gateway is directed. Every subnet must be associated with a route table.

Think of a route table as a GPS for network packets—it tells them which way to go to reach their destination.

#### How Route Tables Work

```
Route Table Structure:
┌──────────────────┬─────────────────┬──────────┐
│ Destination      │ Target          │ Status   │
├──────────────────┼─────────────────┼──────────┤
│ 10.0.0.0/16      │ local           │ active   │ ← VPC internal traffic
│ 0.0.0.0/0        │ igw-xxxxx       │ active   │ ← Internet traffic
└──────────────────┴─────────────────┴──────────┘

Explanation:
- Destination: IP range the rule applies to
- Target: Where to send matching traffic
- local: Special target for VPC-internal communication
```

**Key concepts:**
- **Local route**: Automatically created for VPC CIDR, enables communication within VPC
- **Default route (0.0.0.0/0)**: Catches all traffic not matching other routes (typically internet traffic)
- **Most specific route wins**: `/32` beats `/24` beats `/16` beats `/0`

#### Public vs Private Route Tables

```
PUBLIC ROUTE TABLE (makes subnet public):
┌──────────────────┬─────────────────┐
│ Destination      │ Target          │
├──────────────────┼─────────────────┤
│ 10.0.0.0/16      │ local           │ ← Internal VPC traffic
│ 0.0.0.0/0        │ igw-12345       │ ← Route to Internet Gateway
└──────────────────┴─────────────────┘

PRIVATE ROUTE TABLE (keeps subnet private):
┌──────────────────┬─────────────────┐
│ Destination      │ Target          │
├──────────────────┼─────────────────┤
│ 10.0.0.0/16      │ local           │ ← Internal VPC traffic
│ 0.0.0.0/0        │ nat-67890       │ ← Route to NAT Gateway (optional)
└──────────────────┴─────────────────┘
```

#### Why Route Tables Matter

**Problems they solve:**
- **Traffic Control**: Direct traffic to the right destination (internet, VPN, peering)
- **Security**: Control which subnets can access the internet
- **Network Architecture**: Implement complex routing scenarios (hybrid cloud, multi-VPC)

**Real-world scenario**: A company has:
- Public subnets route `0.0.0.0/0` to Internet Gateway (web tier)
- Private subnets route `0.0.0.0/0` to NAT Gateway (app tier needs outbound internet)
- Database subnets have no `0.0.0.0/0` route (completely isolated)

#### Route Table Association

```
┌────────────────────────────────────────┐
│            VPC (10.0.0.0/16)           │
│                                        │
│  ┌──────────────┐  ┌──────────────┐   │
│  │Public Subnet │  │Private Subnet│   │
│  │              │  │              │   │
│  └──────┬───────┘  └──────┬───────┘   │
│         │                 │            │
│         │                 │            │
│    ┌────▼──────┐     ┌───▼────────┐   │
│    │Public RT  │     │Private RT  │   │
│    │(IGW route)│     │(NAT route) │   │
│    └───────────┘     └────────────┘   │
└────────────────────────────────────────┘

Each subnet → Associated with exactly 1 route table
Each route table → Can be associated with multiple subnets
```

---

### Internet Gateway (IGW)

#### What is an Internet Gateway?

An **Internet Gateway** is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet. It's the doorway between your private VPC network and the public internet.

```
┌──────────────────────────────────────┐
│           Internet                    │
│         (Public IPs)                  │
└───────────────┬──────────────────────┘
                │
                │ ↕ Bidirectional traffic
                │
        ┌───────▼───────┐
        │    Internet   │  ← Managed by AWS
        │    Gateway    │  ← Highly available
        │  (igw-xxxxx)  │  ← No single point of failure
        └───────┬───────┘
                │
┌───────────────┼──────────────────────┐
│              VPC                     │
│               │                      │
│     ┌─────────▼─────────┐            │
│     │  Public Subnet    │            │
│     │                   │            │
│     │  ┌─────────────┐  │            │
│     │  │ EC2 Instance│  │            │
│     │  │ Public IP:  │  │            │
│     │  │ 54.x.x.x    │  │            │
│     │  └─────────────┘  │            │
│     └───────────────────┘            │
└──────────────────────────────────────┘
```

#### How Internet Gateway Works

The IGW performs two critical functions:

**1. NAT (Network Address Translation)** for instances with public IPs:
```
Outbound (EC2 → Internet):
Private IP (10.0.1.5) → IGW translates → Public IP (54.x.x.x)

Inbound (Internet → EC2):
Public IP (54.x.x.x) → IGW translates → Private IP (10.0.1.5)
```

**2. Route Target** for internet-bound traffic:
- Route table entry: `0.0.0.0/0 → igw-xxxxx` directs all internet traffic to IGW

#### Requirements for Internet Access

For an EC2 instance to access the internet via IGW, you need **all four**:

```
✓ 1. Internet Gateway attached to VPC
✓ 2. Public IP or Elastic IP assigned to instance
✓ 3. Subnet route table has route: 0.0.0.0/0 → igw-xxxxx
✓ 4. Security Group allows outbound traffic
```

#### Why Internet Gateway Matters

**Problems it solves:**
- **Public Internet Access**: Resources need to serve content to internet users
- **Outbound Connectivity**: Instances need to download updates, access APIs
- **Bidirectional Communication**: Both inbound (SSH, HTTP) and outbound traffic

**Real-world scenario**:
- Web application servers in public subnet receive HTTP/HTTPS traffic from users worldwide
- These servers download software updates from the internet
- Load balancer accepts incoming connections from any IP address

#### Internet Gateway Characteristics

**Key properties:**
- **One per VPC**: You attach one IGW to one VPC (1:1 relationship)
- **Horizontally Scaled**: AWS automatically scales based on bandwidth needs
- **No Bandwidth Constraints**: No throughput limits imposed by IGW itself
- **No Additional Cost**: No hourly charge or data transfer charge for IGW

```
VPC Limits:
┌─────────────────────────────────────┐
│ 1 VPC ↔ 1 Internet Gateway          │
│                                     │
│ Cannot attach multiple IGWs to VPC  │
│ Cannot attach IGW to multiple VPCs  │
└─────────────────────────────────────┘
```

---

### NAT Gateway

#### What is a NAT Gateway?

A **NAT (Network Address Translation) Gateway** is a managed AWS service that allows instances in private subnets to initiate outbound connections to the internet while preventing the internet from initiating inbound connections to those instances.

Think of it as a **one-way door**: your private resources can reach out to the internet, but the internet cannot reach in.

```
┌──────────────────────────────────────┐
│           Internet                    │
└───────────────┬──────────────────────┘
                │
        ┌───────▼───────┐
        │Internet Gateway│
        └───────┬───────┘
                │
┌───────────────┼──────────────────────┐
│              VPC                     │
│               │                      │
│  ┌────────────▼──────────┐           │
│  │    Public Subnet      │           │
│  │                       │           │
│  │  ┌─────────────────┐  │           │
│  │  │  NAT Gateway    │◄─┼─┐         │
│  │  │  (Elastic IP)   │  │ │         │
│  │  └─────────────────┘  │ │         │
│  └───────────────────────┘ │         │
│                            │         │
│  ┌─────────────────────────┼───┐     │
│  │    Private Subnet       │   │     │
│  │                         │   │     │
│  │  ┌───────────────┐      │   │     │
│  │  │ EC2 Instance  │──────┘   │     │
│  │  │ (Private IP)  │ Outbound │     │
│  │  └───────────────┘   only   │     │
│  └─────────────────────────────┘     │
└──────────────────────────────────────┘

Flow:
1. Private instance sends request to internet
2. Request routes to NAT Gateway
3. NAT Gateway translates private IP to its Elastic IP
4. NAT Gateway forwards to Internet Gateway
5. Response comes back and NAT translates back to private IP
```

#### How NAT Gateway Works

**Outbound traffic flow:**
```
1. EC2 (10.0.2.15) → wants to reach www.example.com
2. Route table: 0.0.0.0/0 → nat-gateway
3. NAT Gateway: Translates 10.0.2.15 → NAT's Elastic IP (54.x.x.x)
4. Internet sees request from 54.x.x.x (not 10.0.2.15)
5. Response returns to 54.x.x.x
6. NAT Gateway translates back: 54.x.x.x → 10.0.2.15
```

**Why inbound connections are blocked:**
- Internet only knows NAT Gateway's Elastic IP
- NAT Gateway only allows responses to connections it initiated
- No way for external systems to directly address private instances

#### Why NAT Gateway Matters

**Problems it solves:**
- **Outbound Internet Access**: Private instances need software updates, API calls
- **Security**: Keep instances private while allowing necessary outbound connectivity
- **Compliance**: Meet requirements for non-publicly accessible infrastructure
- **Simplicity**: No need to manage NAT instances or patches

**Real-world scenario**:
- Application servers in private subnet need to call external payment API (Stripe, PayPal)
- Database servers need to download security patches
- Backend workers need to send emails via external SMTP service
- None of these should be directly accessible from internet

#### NAT Gateway Characteristics

**Key properties:**
- **Managed Service**: AWS handles availability, scaling, patching
- **High Availability**: Within a single AZ (deploy one per AZ for full redundancy)
- **Elastic IP**: Requires one Elastic IP address
- **Bandwidth**: Scales up to 100 Gbps
- **Placement**: Must be deployed in a **public subnet**

**Cost considerations:**
- Hourly charge per NAT Gateway
- Data processing charge per GB transferred
- Multiple NAT Gateways needed for HA (one per AZ)

```
High Availability Setup:
┌──────────────────────────────────────────────┐
│                   VPC                        │
│                                              │
│  AZ-1                      AZ-2              │
│  ┌──────────────┐         ┌──────────────┐  │
│  │Public Subnet │         │Public Subnet │  │
│  │┌────────────┐│         │┌────────────┐│  │
│  ││NAT Gateway ││         ││NAT Gateway ││  │
│  │└──────▲─────┘│         │└──────▲─────┘│  │
│  └───────┼──────┘         └───────┼──────┘  │
│          │                        │          │
│  ┌───────┼──────┐         ┌───────┼──────┐  │
│  │Private│Subnet│         │Private│Subnet│  │
│  │   EC2 │      │         │   EC2 │      │  │
│  └───────┴──────┘         └───────┴──────┘  │
│                                              │
│  Each AZ's private subnet uses its own NAT   │
│  If AZ-1 fails, AZ-2 continues working       │
└──────────────────────────────────────────────┘
```

---

### NAT Instance

#### What is a NAT Instance?

A **NAT Instance** is an EC2 instance configured to perform NAT (Network Address Translation). It's the **legacy, self-managed alternative** to NAT Gateway.

**Important**: AWS recommends NAT Gateway over NAT Instance in almost all cases. NAT Instances are mostly covered here for legacy understanding and exam purposes.

#### NAT Gateway vs NAT Instance

```
┌────────────────────┬─────────────────┬────────────────┐
│     Feature        │  NAT Gateway    │  NAT Instance  │
├────────────────────┼─────────────────┼────────────────┤
│ Management         │ Fully managed   │ You manage     │
│ Availability       │ HA within AZ    │ Manual config  │
│ Bandwidth          │ Up to 100 Gbps  │ Instance size  │
│ Maintenance        │ AWS handles     │ You patch OS   │
│ Cost               │ Higher          │ Lower          │
│ Security Groups    │ Not supported   │ Supported      │
│ Bastion use        │ No              │ Yes (dual use) │
│ Port forwarding    │ No              │ Yes            │
└────────────────────┴─────────────────┴────────────────┘
```

#### When to Use NAT Instance

**Valid use cases (rare):**
- **Cost optimization**: Very low traffic scenarios where NAT Gateway cost is prohibitive
- **Port forwarding**: Need to forward specific ports to private instances
- **Bastion host**: Use same instance as both NAT and bastion (jump box)
- **Security group inspection**: Need to filter traffic at instance level
- **Compliance**: Specific requirements for self-managed NAT

**Configuration requirements:**
```
1. Launch EC2 instance in public subnet
2. Disable "Source/Destination Check" ← Critical!
3. Attach Elastic IP to instance
4. Configure security groups for NAT traffic
5. Update route tables: 0.0.0.0/0 → eni-xxxxx (instance ENI)
6. Manually configure HA/failover scripts (optional)
```

#### Why Source/Destination Check Must Be Disabled

```
Normal EC2 behavior:
- Instance drops packets not addressed to its IP (source check)
- Instance drops packets it didn't originate (destination check)

NAT Instance needs to:
- Accept packets from other instances (different source)
- Forward packets to internet (different destination)

Solution: Disable source/destination check
```

#### NAT Instance Diagram

```
┌────────────────────────────────────────────┐
│              VPC (10.0.0.0/16)             │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │         Public Subnet                │  │
│  │                                      │  │
│  │  ┌────────────────────────────────┐  │  │
│  │  │    NAT Instance (EC2)          │  │  │
│  │  │    - Elastic IP: 54.x.x.x      │  │  │
│  │  │    - Private IP: 10.0.1.10     │  │  │
│  │  │    - Source/Dest Check: OFF    │  │  │
│  │  │    - Security Group: NAT-SG    │  │  │
│  │  └───────────▲────────────────────┘  │  │
│  └──────────────┼───────────────────────┘  │
│                 │                          │
│  ┌──────────────┼───────────────────────┐  │
│  │         Private Subnet               │  │
│  │              │                       │  │
│  │    ┌─────────┴─────────┐             │  │
│  │    │  Application EC2  │             │  │
│  │    │  Route: 0.0.0.0/0 │             │  │
│  │    │  Target: eni-nat  │             │  │
│  │    └───────────────────┘             │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘

If NAT instance fails:
- Manual intervention required
- No automatic failover (unless you script it)
- All private instances lose internet access
```

#### Why NAT Gateway Is Preferred

**NAT Gateway advantages:**
- No instance management, patches, or monitoring
- Automatic high availability within AZ
- Better performance and scaling
- No capacity planning needed
- No source/destination check concerns

**When NAT Instance might still make sense:**
- Extremely cost-sensitive environments with minimal traffic
- Need for advanced traffic inspection or routing
- Legacy systems already using NAT instances

---

### Elastic IPs (EIP)

#### What are Elastic IPs?

An **Elastic IP (EIP)** is a static, public IPv4 address allocated to your AWS account. Unlike regular public IPs that change when you stop/start an instance, an Elastic IP stays with your account until you explicitly release it.

Think of it like owning a phone number—you can transfer it between devices without changing the number.

#### How Elastic IPs Work

```
Regular Public IP (ephemeral):
┌─────────────────────────────────────┐
│ Launch EC2 → Gets 54.123.45.67      │
│ Stop EC2   → IP released            │
│ Start EC2  → Gets 54.198.23.45  ← Different! │
└─────────────────────────────────────┘

Elastic IP (persistent):
┌─────────────────────────────────────┐
│ Allocate EIP → 52.100.200.50        │
│ Associate to EC2-A → 52.100.200.50  │
│ Disassociate → 52.100.200.50        │
│ Associate to EC2-B → 52.100.200.50  ← Same! │
└─────────────────────────────────────┘
```

#### Elastic IP Association

```
Elastic IP can be associated with:

1. EC2 Instance (primary use)
   ┌──────────┐
   │   EC2    │
   │   +EIP   │ ← Direct association
   └──────────┘

2. Network Interface (ENI)
   ┌──────────┐
   │   EC2    │
   │   ENI-1  │ ← EIP attached to ENI
   │   ENI-2  │ ← Different IP
   └──────────┘

3. NAT Gateway
   ┌─────────────┐
   │ NAT Gateway │ ← Requires EIP
   └─────────────┘
```

#### Why Elastic IPs Matter

**Problems they solve:**
- **Fixed IP Address**: Applications/clients need a stable IP that doesn't change
- **DNS Management**: Point DNS records to a fixed IP
- **Quick Recovery**: Rapidly remap IP to standby instance during failures
- **IP Whitelisting**: Third parties can whitelist your specific IP

**Real-world scenarios**:
- **API endpoint**: External partners whitelist your IP for API access
- **Email server**: Maintain IP reputation for email deliverability
- **Failover**: Quickly move IP from failed primary to secondary instance
- **Compliance**: Regulations require static IP for audit logs

#### Elastic IP Costs and Limits

**Pricing model:**
```
✓ FREE when:
  - Associated with a running instance
  - Only one EIP per running instance

✗ CHARGED when:
  - Not associated with any resource
  - Associated with stopped instance
  - Multiple EIPs on same instance (second+ charged)

Why? AWS wants to discourage IP hoarding
```

**Limits:**
- **5 Elastic IPs per region** by default (soft limit, can request increase)
- Each EIP can associate with one resource at a time
- Can quickly remap between resources (< 5 minutes)

#### Elastic IP Best Practices

**When to use:**
- Load balancer alternative for single-instance applications
- NAT Gateway (required component)
- Bastion hosts that need fixed SSH access point
- Legacy applications requiring static IPs

**When NOT to use:**
- **Use Load Balancer instead**: For production web apps (better availability)
- **Use Route 53 instead**: For DNS-based routing and failover
- **Use NAT Gateway instead**: Already includes managed EIP

```
Architecture Evolution:
┌──────────────────────────────────────────┐
│ ❌ Anti-pattern:                         │
│    EIP → Single EC2 (web server)         │
│    Problem: Single point of failure      │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ ✓ Better:                                │
│    ALB → Multiple EC2 (auto-scaling)     │
│    Problem solved: HA, auto-scaling      │
└──────────────────────────────────────────┘
```

#### Elastic IP Remapping for High Availability

```
Manual failover scenario:
┌──────────────────────────────────────────┐
│                                          │
│  ┌──────────┐            ┌──────────┐   │
│  │Primary   │            │Secondary │   │
│  │EC2       │            │EC2       │   │
│  │          │            │(standby) │   │
│  └────▲─────┘            └──────────┘   │
│       │                                  │
│    ┌──┴──┐                               │
│    │ EIP │ 52.x.x.x                      │
│    └─────┘                               │
│                                          │
│  Primary fails →                         │
│                                          │
│  ┌──────────┐            ┌──────────┐   │
│  │Primary   │            │Secondary │   │
│  │EC2       │            │EC2       │   │
│  │(failed)  │            │          │   │
│  └──────────┘            └────▲─────┘   │
│                               │          │
│                          ┌────┴───┐      │
│                          │  EIP   │      │
│                          └────────┘      │
│                          52.x.x.x        │
│                          (same IP!)      │
└──────────────────────────────────────────┘
```

---

### VPC Peering

#### What is VPC Peering?

**VPC Peering** is a networking connection between two VPCs that enables routing traffic between them using private IP addresses. Instances in either VPC can communicate as if they're within the same network.

Think of it as building a private bridge between two separate networks.

```
Before Peering:
┌──────────────┐              ┌──────────────┐
│   VPC-A      │              │   VPC-B      │
│  10.0.0.0/16 │              │ 172.16.0.0/16│
│              │   ✗ No       │              │
│   ┌──────┐   │   connection │   ┌──────┐   │
│   │ EC2  │   │              │   │ EC2  │   │
│   └──────┘   │              │   └──────┘   │
└──────────────┘              └──────────────┘

After Peering:
┌──────────────┐              ┌──────────────┐
│   VPC-A      │◄────────────►│   VPC-B      │
│  10.0.0.0/16 │   Peering    │ 172.16.0.0/16│
│              │   Connection │              │
│   ┌──────┐   │              │   ┌──────┐   │
│   │ EC2  │───┼──────────────┼──►│ EC2  │   │
│   └──────┘   │   Private    │   └──────┘   │
│ 10.0.1.5     │   routing    │ 172.16.1.10  │
└──────────────┘              └──────────────┘
```

#### How VPC Peering Works

**Connection setup:**
```
1. VPC-A owner creates peering request → VPC-B
2. VPC-B owner accepts request
3. Update route tables in both VPCs:

   VPC-A Route Table:
   ┌──────────────────┬─────────────────┐
   │ Destination      │ Target          │
   ├──────────────────┼─────────────────┤
   │ 10.0.0.0/16      │ local           │
   │ 172.16.0.0/16    │ pcx-xxxxx       │ ← Peering connection
   └──────────────────┴─────────────────┘

   VPC-B Route Table:
   ┌──────────────────┬─────────────────┐
   │ Destination      │ Target          │
   ├──────────────────┼─────────────────┤
   │ 172.16.0.0/16    │ local           │
   │ 10.0.0.0/16      │ pcx-xxxxx       │ ← Same peering connection
   └──────────────────┴─────────────────┘

4. Update security groups to allow traffic from peer VPC CIDR
```

#### VPC Peering Constraints

**Critical limitations:**

**1. No Transitive Peering**
```
❌ This DOES NOT work:
┌──────┐     ┌──────┐     ┌──────┐
│VPC-A │◄───►│VPC-B │◄───►│VPC-C │
└──────┘     └──────┘     └──────┘
    ▲                         ▲
    │      ✗ No direct        │
    └─────────path────────────┘

VPC-A cannot reach VPC-C through VPC-B

✓ Solution: Create direct peering
┌──────┐     ┌──────┐     ┌──────┐
│VPC-A │◄───►│VPC-B │◄───►│VPC-C │
└───┬──┘     └──────┘     └───▲──┘
    │                          │
    └──────────────────────────┘
         Direct peering
```

**2. No Overlapping CIDR Blocks**
```
❌ Cannot peer:
VPC-A: 10.0.0.0/16
VPC-B: 10.0.0.0/24  ← Overlaps with VPC-A!

✓ Can peer:
VPC-A: 10.0.0.0/16
VPC-B: 172.16.0.0/16  ← No overlap
```

**3. One Peering Connection Per VPC Pair**
```
VPC-A can peer with VPC-B once
Cannot create multiple peering connections between same two VPCs
```

#### Why VPC Peering Matters

**Problems it solves:**
- **Service Isolation**: Separate VPCs for different environments (dev, staging, prod)
- **Account Isolation**: Different AWS accounts but need private communication
- **Region Isolation**: VPCs in different regions can peer
- **Cost Efficiency**: No internet gateway, NAT, or VPN needed

**Real-world scenarios**:
- **Shared services**: Central VPC with Active Directory, accessed by multiple app VPCs
- **Multi-tenant**: Each customer has isolated VPC, connects to shared backend VPC
- **Data analytics**: Production VPC peers with analytics VPC for data processing
- **Cross-account**: Separate AWS accounts for billing, but need resource access

#### Peering Across Regions and Accounts

```
Inter-Region Peering:
┌────────────────────────────────────────┐
│ US-East-1                              │
│  ┌──────────┐                          │
│  │  VPC-A   │                          │
│  │10.0.0.0  │◄────────┐                │
│  └──────────┘         │                │
└───────────────────────┼────────────────┘
                        │ Peering
┌───────────────────────┼────────────────┐
│ EU-West-1             │                │
│  ┌──────────┐         │                │
│  │  VPC-B   │◄────────┘                │
│  │172.16.0.0│                          │
│  └──────────┘                          │
└────────────────────────────────────────┘

Cross-Account Peering:
┌────────────────────────────────────────┐
│ Account: 111111111111                  │
│  ┌──────────┐                          │
│  │  VPC-A   │◄────────┐                │
│  └──────────┘         │                │
└───────────────────────┼────────────────┘
                        │ Peering
┌───────────────────────┼────────────────┐
│ Account: 222222222222 │                │
│  ┌──────────┐         │                │
│  │  VPC-B   │◄────────┘                │
│  └──────────┘                          │
└────────────────────────────────────────┘
```

#### VPC Peering vs Alternatives

```
┌──────────────────┬────────────┬─────────────┬───────────┐
│ Feature          │ Peering    │ Transit GW  │ PrivateLink│
├──────────────────┼────────────┼─────────────┼───────────┤
│ Max VPCs         │ Many (1-1) │ Thousands   │ N/A       │
│ Transitive       │ No         │ Yes         │ No        │
│ Cross-region     │ Yes        │ Yes         │ No*       │
│ Complexity       │ Low        │ Medium      │ Low       │
│ Cost             │ Low        │ Higher      │ Medium    │
│ Use case         │ Simple     │ Hub-spoke   │ Services  │
└──────────────────┴────────────┴─────────────┴───────────┘

*PrivateLink endpoints can be cross-region with additional setup
```

---

### VPC Endpoints

#### What are VPC Endpoints?

**VPC Endpoints** allow you to privately connect your VPC to supported AWS services without requiring an internet gateway, NAT device, VPN connection, or AWS Direct Connect. Traffic between your VPC and the AWS service does not leave the Amazon network.

Think of it as creating a **private tunnel** directly to AWS services, bypassing the public internet entirely.

```
WITHOUT VPC Endpoint:
┌──────────────────────────────────────┐
│            VPC                       │
│  ┌──────────────┐                    │
│  │Private Subnet│                    │
│  │  ┌────────┐  │                    │
│  │  │  EC2   │──┼────┐               │
│  │  └────────┘  │    │               │
│  └──────────────┘    │               │
└──────────────────────┼───────────────┘
                       │
                  NAT Gateway
                       │
                 Internet Gateway
                       │
                  Public Internet ← Security risk, cost, latency
                       │
              ┌────────▼────────┐
              │   AWS S3 API    │
              │ s3.amazonaws.com│
              └─────────────────┘

WITH VPC Endpoint:
┌──────────────────────────────────────┐
│            VPC                       │
│  ┌──────────────┐                    │
│  │Private Subnet│                    │
│  │  ┌────────┐  │    ┌────────────┐  │
│  │  │  EC2   │──┼───►│VPC Endpoint│──┼──┐
│  │  └────────┘  │    │  (Private) │  │  │
│  └──────────────┘    └────────────┘  │  │
└──────────────────────────────────────┘  │
                                          │
                    AWS Private Network   │
                                          │
                                   ┌──────▼──────┐
                                   │   AWS S3    │
                                   │   (Private) │
                                   └─────────────┘
```

#### Types of VPC Endpoints

AWS provides two types:
1. **Gateway Endpoints** (for S3 and DynamoDB only)
2. **Interface Endpoints** (for everything else)

---

#### Gateway Endpoints

**What are Gateway Endpoints?**

Gateway Endpoints are a **route table target** that routes traffic destined for S3 or DynamoDB to these services via Amazon's private network.

**Supported services:**
- Amazon S3
- DynamoDB

**How Gateway Endpoints work:**
```
Configuration:
1. Create Gateway Endpoint for S3 in your VPC
2. Associate with route table(s)
3. AWS automatically adds route:

Route Table Update:
┌─────────────────────────┬──────────────┐
│ Destination             │ Target       │
├─────────────────────────┼──────────────┤
│ 10.0.0.0/16             │ local        │
│ pl-12345678 (S3 prefix) │ vpce-xxxxx   │ ← Auto-added
└─────────────────────────┴──────────────┘

pl-12345678 = Managed prefix list for S3 IP ranges
```

**Request flow:**
```
1. EC2 instance: aws s3 cp file.txt s3://my-bucket/
2. Route table: Traffic to S3 → vpce-xxxxx
3. Gateway Endpoint: Routes to S3 via AWS network
4. S3: Returns data via same private path
```

**Gateway Endpoint characteristics:**
- **No cost**: Free to use (still pay for data transfer)
- **No ENI**: Doesn't create network interface in subnet
- **Route-based**: Works via route table entries
- **Regional**: Only works within same region as VPC
- **Scalable**: AWS handles bandwidth and availability

---

#### Interface Endpoints (AWS PrivateLink)

**What are Interface Endpoints?**

Interface Endpoints create an **Elastic Network Interface (ENI)** in your subnet with a private IP address that serves as an entry point for traffic destined to supported AWS services.

**Supported services:** 100+ AWS services including:
- EC2, ECS, Lambda, API Gateway
- CloudWatch, CloudFormation, Systems Manager
- Kinesis, SNS, SQS, Secrets Manager
- And many more...

**How Interface Endpoints work:**
```
┌──────────────────────────────────────────┐
│              VPC (10.0.0.0/16)           │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │      Private Subnet                │  │
│  │                                    │  │
│  │  ┌──────────┐     ┌─────────────┐ │  │
│  │  │   EC2    │────►│  Interface  │ │  │
│  │  │          │     │  Endpoint   │ │  │
│  │  └──────────┘     │  (ENI)      │ │  │
│  │                   │  10.0.1.50  │─┼──┼─┐
│  │                   └─────────────┘ │  │ │
│  └────────────────────────────────────┘  │ │
└──────────────────────────────────────────┘ │
                                             │
                      AWS PrivateLink        │
                                             │
                                      ┌──────▼──────┐
                                      │  AWS SNS    │
                                      │  (Service)  │
                                      └─────────────┘
```

**Request flow:**
```
1. EC2: aws sns publish --topic-arn arn:aws:sns:...
2. DNS: sns.us-east-1.amazonaws.com → 10.0.1.50 (endpoint ENI)
3. Endpoint ENI: Forwards to SNS via PrivateLink
4. SNS: Processes request and returns via PrivateLink
```

**Interface Endpoint characteristics:**
- **Cost**: Hourly charge + data processing fee
- **ENI**: Creates network interface in your subnet
- **DNS**: Optionally creates private DNS names
- **Multi-AZ**: Deploy one ENI per AZ for high availability
- **Security Groups**: Apply security groups to control access

---

#### Gateway vs Interface Endpoints

```
┌─────────────────────┬─────────────────┬─────────────────┐
│ Feature             │ Gateway         │ Interface       │
├─────────────────────┼─────────────────┼─────────────────┤
│ Services            │ S3, DynamoDB    │ 100+ services   │
│ Cost                │ Free            │ Hourly + data   │
│ Implementation      │ Route table     │ ENI in subnet   │
│ IP address          │ No              │ Yes (private)   │
│ Security groups     │ No (use policy) │ Yes             │
│ On-premises access  │ No              │ Yes (via DX/VPN)│
│ Subnet placement    │ No (route only) │ Yes             │
│ HA                  │ Automatic       │ Multi-AZ ENIs   │
└─────────────────────┴─────────────────┴─────────────────┘
```

#### Why VPC Endpoints Matter

**Problems they solve:**
- **Security**: Keep traffic off public internet (compliance requirement)
- **Cost**: Avoid NAT Gateway data processing fees
- **Performance**: Lower latency via AWS backbone network
- **Simplicity**: No need to manage NAT/IGW for AWS service access

**Real-world scenarios**:
- **Compliance**: Financial data must not traverse public internet
- **Cost optimization**: High S3 traffic from private subnets (NAT Gateway expensive)
- **Hybrid cloud**: On-premises systems access AWS services via Direct Connect + endpoints
- **Microservices**: Services communicate with SQS/SNS without internet exposure

---

### AWS PrivateLink

#### What is AWS PrivateLink?

**AWS PrivateLink** is the underlying technology that powers Interface Endpoints. More broadly, it enables you to **privately expose your own services** to other VPCs or AWS accounts without using VPC peering, internet gateways, or NAT.

Think of it as creating a **private, scalable service endpoint** that others can consume securely.

```
Service Provider (Your Service):
┌──────────────────────────────────────────┐
│         Provider VPC                     │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  Network Load Balancer (NLB)      │  │
│  │  ┌──────┐  ┌──────┐  ┌──────┐    │  │
│  │  │ EC2  │  │ EC2  │  │ EC2  │    │  │
│  │  └──────┘  └──────┘  └──────┘    │  │
│  └─────────────────┬──────────────────┘  │
│                    │                     │
│          ┌─────────▼──────────┐          │
│          │  VPC Endpoint      │          │
│          │  Service           │          │
│          │  com.amazonaws.    │          │
│          │  vpce.us-east-1... │          │
│          └─────────┬──────────┘          │
└────────────────────┼─────────────────────┘
                     │
              AWS PrivateLink
                     │
┌────────────────────┼─────────────────────┐
│ Consumer VPC       │                     │
│                    │                     │
│          ┌─────────▼──────────┐          │
│          │  Interface Endpoint│          │
│          │  (ENI)             │          │
│          │  10.1.1.50         │          │
│          └─────────▲──────────┘          │
│                    │                     │
│           ┌────────┴────────┐            │
│           │   EC2 Instance  │            │
│           │   (Consumer)    │            │
│           └─────────────────┘            │
└──────────────────────────────────────────┘
```

#### PrivateLink Architecture Components

**Provider side (your service):**
1. **Network Load Balancer (NLB)**: Fronts your service
2. **Endpoint Service**: Links NLB to PrivateLink
3. **Service Name**: Unique identifier consumers use to connect

**Consumer side:**
1. **Interface Endpoint**: ENI in consumer VPC
2. **Private DNS**: (Optional) Friendly DNS name
3. **Security Groups**: Control access to endpoint

#### How PrivateLink Works

**Setup flow:**
```
Provider:
1. Deploy service (e.g., API) behind NLB
2. Create VPC Endpoint Service: nlb-xxxxx
3. Get service name: com.amazonaws.vpce.us-east-1.vpce-svc-xxxxx
4. Configure permissions (which accounts can connect)

Consumer:
1. Create Interface Endpoint using service name
2. AWS creates ENI in consumer subnet
3. Traffic to service goes via PrivateLink
4. No IP overlap concerns (uses PrivateLink addressing)
```

#### PrivateLink vs VPC Peering

```
VPC Peering:
┌──────────┐ Full network ┌──────────┐
│ VPC-A    │◄────────────►│ VPC-B    │
│ All      │   access     │ All      │
│ resources│              │ resources│
└──────────┘              └──────────┘
- Requires non-overlapping CIDRs
- Exposes entire VPC network
- Transitive routing not supported

PrivateLink:
┌──────────┐              ┌──────────┐
│ VPC-A    │              │ VPC-B    │
│ ┌──────┐ │   Specific   │  All     │
│ │ NLB  │◄┼──────────────┤ resources│
│ │Service│ │   service    │ can      │
│ └──────┘ │   only       │ access   │
└──────────┘              └──────────┘
- CIDRs can overlap (no conflict)
- Exposes only specific service
- Scalable to thousands of consumers
```

#### Why PrivateLink Matters

**Problems it solves:**
- **Service Exposure**: Provide services to many VPCs/accounts without peering
- **Security**: Consumers don't need access to your entire VPC
- **Scalability**: Support thousands of consumers (vs. peering limits)
- **CIDR Independence**: Works even with overlapping IP ranges

**Real-world scenarios**:
- **SaaS providers**: Offer private connectivity to enterprise customers
- **Shared services**: Central IT provides services (auth, logging) to many business units
- **Marketplace**: Sell services via AWS Marketplace with private endpoints
- **Partner integration**: Allow partners to access your API without VPN

---

### VPC Flow Logs

#### What are VPC Flow Logs?

**VPC Flow Logs** capture information about the IP traffic going to and from network interfaces in your VPC. It's like having a **detailed call log** for all network communications.

Flow Logs help you troubleshoot connectivity issues, monitor traffic patterns, and meet compliance requirements.

```
Flow Log captures:
┌────────────────────────────────────────────────┐
│ Source IP → Destination IP                     │
│ Source Port → Destination Port                 │
│ Protocol (TCP/UDP/ICMP)                        │
│ Packets, Bytes transferred                     │
│ Action (ACCEPT/REJECT)                         │
│ Timestamp                                      │
└────────────────────────────────────────────────┘
```

#### Flow Log Levels

You can create Flow Logs at three levels:

```
1. VPC Level:
   ┌────────────────────────┐
   │        VPC             │ ← All traffic in VPC
   │  ┌──────┐   ┌──────┐  │
   │  │Subnet│   │Subnet│  │
   │  └──────┘   └──────┘  │
   └────────────────────────┘

2. Subnet Level:
   ┌────────────────────────┐
   │        VPC             │
   │  ┌──────┐   ┌──────┐  │
   │  │Subnet│   │Subnet│  │ ← Only this subnet
   │  └──────┘   └──────┘  │
   └────────────────────────┘

3. Network Interface (ENI) Level:
   ┌────────────────────────┐
   │        VPC             │
   │  ┌──────┐              │
   │  │ ENI  │ ← Only this ENI
   │  └──────┘              │
   └────────────────────────┘
```

#### Flow Log Record Format

**Standard record example:**
```
2 123456789012 eni-abc123 172.31.16.5 172.31.16.10
20641 22 6 20 4249 1418530010 1418530070 ACCEPT OK

Fields:
- version: 2
- account-id: 123456789012
- interface-id: eni-abc123
- srcaddr: 172.31.16.5
- dstaddr: 172.31.16.10
- srcport: 20641
- dstport: 22 (SSH)
- protocol: 6 (TCP)
- packets: 20
- bytes: 4249
- start: 1418530010
- end: 1418530070
- action: ACCEPT
- log-status: OK
```

#### Flow Log Destinations

Flow logs can be published to:

**1. CloudWatch Logs:**
```
┌──────────────────────────────────────┐
│            VPC                       │
│  ┌────────────┐                      │
│  │ Flow Logs  │──────┐               │
│  └────────────┘      │               │
└──────────────────────┼───────────────┘
                       │
                ┌──────▼────────┐
                │  CloudWatch   │
                │  Log Group    │
                │  /aws/vpc/fl  │
                └───────────────┘
Use case: Real-time monitoring, alarms
```

**2. S3 Bucket:**
```
┌──────────────────────────────────────┐
│            VPC                       │
│  ┌────────────┐                      │
│  │ Flow Logs  │──────┐               │
│  └────────────┘      │               │
└──────────────────────┼───────────────┘
                       │
                ┌──────▼────────┐
                │  S3 Bucket    │
                │  flow-logs/   │
                └───────────────┘
Use case: Long-term storage, batch analysis
```

**3. Kinesis Data Firehose:**
```
┌──────────────────────────────────────┐
│            VPC                       │
│  ┌────────────┐                      │
│  │ Flow Logs  │──────┐               │
│  └────────────┘      │               │
└──────────────────────┼───────────────┘
                       │
                ┌──────▼──────────┐
                │ Kinesis Firehose│
                │      ↓          │
                │  S3/Redshift/   │
                │  OpenSearch     │
                └─────────────────┘
Use case: Real-time analytics, streaming
```

#### What Flow Logs Capture (and Don't)

**Captured:**
- Traffic to/from EC2 instances
- Traffic to/from ELB, RDS, ElastiCache
- Accepted AND rejected traffic
- Traffic between VPCs (peering)

**NOT captured:**
- Traffic to Amazon DNS server (Route 53 Resolver)
- Windows license activation traffic
- Instance metadata (169.254.169.254)
- DHCP traffic
- Traffic to VPC router (reserved IP)

#### Why Flow Logs Matter

**Problems they solve:**
- **Security Monitoring**: Detect unusual traffic patterns, potential attacks
- **Troubleshooting**: Understand why connections fail (rejected packets)
- **Compliance**: Audit trails for regulatory requirements
- **Cost Analysis**: Identify high-traffic sources

**Real-world scenarios**:
- **Security breach**: Analyze flow logs to trace attacker's lateral movement
- **Connection issues**: Check if traffic is being rejected by security groups/NACLs
- **Cost investigation**: Find EC2 instances with unexpected high data transfer
- **Compliance audit**: Prove no unauthorized access to sensitive subnets

#### Flow Log Analysis Example

**Scenario**: SSH connection failing

```
Flow Log Analysis:
┌─────────────────────────────────────────────┐
│ srcaddr: 203.0.113.5 (your laptop)          │
│ dstaddr: 10.0.1.10 (EC2 instance)           │
│ srcport: 54321                              │
│ dstport: 22 (SSH)                           │
│ protocol: 6 (TCP)                           │
│ action: REJECT ← Problem found!             │
└─────────────────────────────────────────────┘

Diagnosis:
- Traffic is being REJECTED
- Check Security Group rules for port 22
- Check Network ACL rules
- Check route tables
```

---

### Network ACLs (NACLs)

#### What are Network ACLs?

**Network Access Control Lists (NACLs)** are stateless firewalls that control traffic at the **subnet boundary**. They act as a security layer that filters traffic entering or leaving a subnet.

Think of NACLs as a **gatekeeper** at the subnet entrance—checking every packet in both directions.

```
┌────────────────────────────────────────────┐
│                VPC                         │
│                                            │
│   ┌─────────────────────────────────────┐  │
│   │         Subnet                      │  │
│   │                                     │  │
│   │  ┌─────┐        ┌─────┐  ┌─────┐  │  │
│   │  │ EC2 │        │ EC2 │  │ EC2 │  │  │
│   │  └─────┘        └─────┘  └─────┘  │  │
│   │                                     │  │
│   └───────────┬─────────────────────────┘  │
│               │                            │
│       ┌───────▼─────────┐                  │
│       │  Network ACL    │ ← Subnet boundary
│       │  (NACL)         │    firewall      │
│       └───────┬─────────┘                  │
│               │                            │
└───────────────┼────────────────────────────┘
                │
         Internet/Other subnets
```

#### How NACLs Work

**NACL Structure:**
```
Network ACL Rules (Inbound):
┌──────┬─────────┬──────────┬──────┬─────┬────────┐
│ Rule │ Type    │ Protocol │ Port │ Src │ Action │
├──────┼─────────┼──────────┼──────┼─────┼────────┤
│ 100  │ HTTP    │ TCP      │ 80   │ All │ ALLOW  │
│ 110  │ HTTPS   │ TCP      │ 443  │ All │ ALLOW  │
│ 120  │ SSH     │ TCP      │ 22   │ All │ ALLOW  │
│ *    │ All     │ All      │ All  │ All │ DENY   │ ← Default
└──────┴─────────┴──────────┴──────┴─────┴────────┘

Network ACL Rules (Outbound):
┌──────┬─────────┬──────────┬──────┬─────┬────────┐
│ Rule │ Type    │ Protocol │ Port │ Dst │ Action │
├──────┼─────────┼──────────┼──────┼─────┼────────┤
│ 100  │ All     │ All      │ All  │ All │ ALLOW  │
│ *    │ All     │ All      │ All  │ All │ DENY   │
└──────┴─────────┴──────────┴──────┴─────┴────────┘

Rules are evaluated in order (lowest number first)
First match wins, processing stops
```

#### NACLs are Stateless

**Critical concept**: NACLs are **stateless**—they don't remember connections.

```
Stateless behavior:
┌──────────────────────────────────────────┐
│ Inbound request:                         │
│   Client (1.2.3.4:54321) → EC2 (22)      │
│   Inbound rule: ALLOW src=0.0.0.0/0:22   │
│                                          │
│ Response:                                │
│   EC2 (22) → Client (1.2.3.4:54321)      │
│   Outbound rule: MUST ALSO ALLOW         │
│   dst=0.0.0.0/0:1024-65535 (ephemeral)   │
└──────────────────────────────────────────┘

You must allow:
1. Inbound rule for request
2. Outbound rule for response (including ephemeral ports)
```

**Ephemeral ports**: Temporary ports (1024-65535) used for client connections

#### NACL vs Security Groups

```
┌─────────────────────┬──────────────┬─────────────────┐
│ Feature             │ NACL         │ Security Group  │
├─────────────────────┼──────────────┼─────────────────┤
│ Level               │ Subnet       │ Instance (ENI)  │
│ State               │ Stateless    │ Stateful        │
│ Rules               │ Allow + Deny │ Allow only      │
│ Rule processing     │ Order matters│ All rules eval  │
│ Applies to          │ All instances│ Assigned only   │
│ Default behavior    │ Allow all    │ Deny all        │
└─────────────────────┴──────────────┴─────────────────┘
```

**Layered defense:**
```
┌─────────────────────────────────────────┐
│           Internet                      │
└──────────────┬──────────────────────────┘
               │
       ┌───────▼─────────┐
       │  Network ACL    │ ← First layer (subnet)
       │  Stateless      │
       └───────┬─────────┘
               │
┌──────────────┼──────────────────────────┐
│     Subnet   │                          │
│              │                          │
│      ┌───────▼─────────┐                │
│      │ Security Group  │ ← Second layer (instance)
│      │ Stateful        │                │
│      └───────┬─────────┘                │
│              │                          │
│        ┌─────▼─────┐                    │
│        │    EC2    │                    │
│        └───────────┘                    │
└─────────────────────────────────────────┘
```

#### Why Network ACLs Matter

**Problems they solve:**
- **Subnet-level protection**: Block traffic before it reaches instances
- **Explicit deny**: Block specific IP ranges or ports
- **Compliance**: Additional security layer for regulatory requirements
- **DDoS mitigation**: Quickly block attack sources at subnet level

**Real-world scenarios**:
- **Block malicious IPs**: Quickly deny traffic from known attack sources
- **Subnet isolation**: Prevent subnets from communicating
- **Defense in depth**: Add subnet-level rules in addition to security groups
- **Emergency response**: Rapidly block traffic during active attack

#### NACL Best Practices

```
✓ Use NACLs for explicit deny rules (block bad IPs)
✓ Keep rules simple and minimal
✓ Remember to configure both inbound AND outbound
✓ Consider ephemeral ports for outbound responses

✗ Don't rely solely on NACLs (use Security Groups too)
✗ Don't forget rule number ordering
✗ Don't create overly complex rule sets
```

---

### Security Groups

#### What are Security Groups?

**Security Groups** act as virtual firewalls at the **instance level** (technically at the ENI level). They control inbound and outbound traffic for EC2 instances, RDS databases, load balancers, and other AWS resources.

Think of a Security Group as a **bouncer** for each instance—deciding who gets in and out.

```
┌──────────────────────────────────────────┐
│           VPC / Subnet                   │
│                                          │
│   ┌──────────────────────────────────┐   │
│   │    Security Group "web-sg"       │   │
│   │                                  │   │
│   │  ┌────────────────────────────┐  │   │
│   │  │  EC2 Instance              │  │   │
│   │  │  - Web Server              │  │   │
│   │  │  - ENI: eni-12345          │  │   │
│   │  │  - Private IP: 10.0.1.10   │  │   │
│   │  └────────────────────────────┘  │   │
│   │                                  │   │
│   │  Rules:                          │   │
│   │  In: Allow TCP/80 from 0.0.0.0/0 │   │
│   │  In: Allow TCP/443 from 0.0.0.0/0│   │
│   │  Out: Allow All                  │   │
│   └──────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

#### How Security Groups Work

**Rule structure:**
```
Security Group: web-sg

Inbound Rules:
┌──────────┬──────────┬──────┬────────────────────┐
│ Type     │ Protocol │ Port │ Source             │
├──────────┼──────────┼──────┼────────────────────┤
│ HTTP     │ TCP      │ 80   │ 0.0.0.0/0          │ ← Anywhere
│ HTTPS    │ TCP      │ 443  │ 0.0.0.0/0          │
│ SSH      │ TCP      │ 22   │ 203.0.113.0/24     │ ← Specific range
│ Custom   │ TCP      │ 8080 │ sg-app-tier        │ ← Another SG
└──────────┴──────────┴──────┴────────────────────┘

Outbound Rules:
┌──────────┬──────────┬──────┬────────────────────┐
│ Type     │ Protocol │ Port │ Destination        │
├──────────┼──────────┼──────┼────────────────────┤
│ All      │ All      │ All  │ 0.0.0.0/0          │ ← Default
└──────────┴──────────┴──────┴────────────────────┘
```

#### Security Groups are Stateful

**Critical concept**: Security Groups are **stateful**—they remember connections.

```
Stateful behavior:
┌────────────────────────────────────────────┐
│ Inbound request:                           │
│   Client → EC2:80 (HTTP)                   │
│   Inbound rule checked: ALLOW port 80      │
│                                            │
│ Response:                                  │
│   EC2:80 → Client (ephemeral port)         │
│   NO outbound rule check! ← Automatic      │
│   Connection tracked, response allowed     │
└────────────────────────────────────────────┘

You only need to specify:
- Inbound rule for incoming connections
- Outbound rule is automatic for responses
```

#### Security Group References

**Powerful feature**: Reference other security groups instead of IP addresses.

```
Architecture:
┌──────────────────────────────────────────────┐
│  ┌─────────────────────┐                     │
│  │  sg-web (Web tier)  │                     │
│  │  ┌──────┐  ┌──────┐ │                     │
│  │  │ EC2  │  │ EC2  │ │                     │
│  │  └──────┘  └──────┘ │                     │
│  └──────────┬──────────┘                     │
│             │ Can talk to                    │
│  ┌──────────▼──────────┐                     │
│  │  sg-app (App tier)  │                     │
│  │  Inbound:           │                     │
│  │  Port 8080          │                     │
│  │  Source: sg-web     │ ← Reference!        │
│  │  ┌──────┐  ┌──────┐ │                     │
│  │  │ EC2  │  │ EC2  │ │                     │
│  │  └──────┘  └──────┘ │                     │
│  └──────────┬──────────┘                     │
│             │ Can talk to                    │
│  ┌──────────▼──────────┐                     │
│  │  sg-db (Data tier)  │                     │
│  │  Inbound:           │                     │
│  │  Port 3306          │                     │
│  │  Source: sg-app     │ ← Reference!        │
│  │  ┌──────┐           │                     │
│  │  │ RDS  │           │                     │
│  │  └──────┘           │                     │
│  └─────────────────────┘                     │
└──────────────────────────────────────────────┘

Benefits:
- No need to update rules when instances scale
- Instances automatically get access when added to SG
- Clear, manageable security architecture
```

#### Why Security Groups Matter

**Problems they solve:**
- **Instance-level security**: Fine-grained control per resource
- **Dynamic scaling**: Security follows instances (auto-scaling friendly)
- **Least privilege**: Only allow necessary traffic
- **Simplified management**: SG references eliminate IP management

**Real-world scenarios**:
- **Three-tier app**: Web → App → Database with layered security
- **Auto-scaling**: New web servers automatically get same security rules
- **Microservices**: Each service has its own SG with specific allow rules
- **Compliance**: Enforce network segmentation requirements

#### Security Group Characteristics

**Key properties:**
- **Default deny**: All inbound traffic denied by default
- **Allow rules only**: Cannot create explicit deny rules (use NACLs for that)
- **Multiple SGs**: Instance can have up to 5 security groups
- **Live changes**: Rule changes apply immediately
- **Connection tracking**: Stateful—responses automatically allowed

**Limits:**
- 5 security groups per ENI (default)
- 60 inbound rules per security group (default)
- 60 outbound rules per security group (default)
- Soft limits—can request increases

---

### DHCP Options Sets

#### What are DHCP Options Sets?

**DHCP (Dynamic Host Configuration Protocol) Options Sets** define network configuration settings that EC2 instances receive when they boot up. This includes DNS servers, domain names, NTP servers, and NetBIOS settings.

Think of it as the **network configuration package** automatically delivered to every instance in your VPC.

```
When EC2 instance boots:
┌──────────────────────────────────────┐
│ EC2 Instance starts                  │
│       ↓                              │
│ DHCP client broadcasts:              │
│ "I need network configuration!"      │
│       ↓                              │
│ VPC DHCP server responds with:       │
│ ┌──────────────────────────────────┐ │
│ │ - IP address: 10.0.1.15          │ │
│ │ - Subnet mask: 255.255.255.0     │ │
│ │ - Default gateway: 10.0.1.1      │ │
│ │ - DNS server: 10.0.0.2           │ │ ← From DHCP Options
│ │ - Domain name: ec2.internal      │ │ ← From DHCP Options
│ └──────────────────────────────────┘ │
│       ↓                              │
│ Instance configures itself           │
└──────────────────────────────────────┘
```

#### DHCP Options You Can Configure

```
Available options:
┌────────────────────────────────────────────┐
│ domain-name: Custom domain (e.g., company.local)
│ domain-name-servers: DNS server IPs       │
│ ntp-servers: NTP server IPs               │
│ netbios-name-servers: NetBIOS servers     │
│ netbios-node-type: NetBIOS node type      │
└────────────────────────────────────────────┘
```

**Default DHCP Options Set:**
```
domain-name:
  - us-east-1: ec2.internal
  - other regions: region.compute.internal

domain-name-servers:
  - AmazonProvidedDNS (VPC CIDR +2 address)
  - Example: VPC 10.0.0.0/16 → DNS at 10.0.0.2
```

#### Custom DHCP Options Example

**Scenario**: Use your own DNS servers instead of AWS-provided DNS

```
Custom DHCP Options Set:
┌───────────────────────────────────────┐
│ domain-name: corp.example.com         │
│ domain-name-servers:                  │
│   - 10.0.1.100 (Active Directory)     │
│   - 10.0.2.100 (Secondary DNS)        │
│   - AmazonProvidedDNS (fallback)      │
└───────────────────────────────────────┘

Result:
- Instances get domain: corp.example.com
- DNS queries go to your DNS servers first
- Fallback to AWS DNS if yours fail
```

#### Why DHCP Options Matter

**Problems they solve:**
- **Custom DNS**: Use corporate DNS servers (Active Directory, on-premises)
- **Domain naming**: Set custom domain for internal hostnames
- **Time sync**: Configure specific NTP servers for compliance
- **Hybrid cloud**: Integrate with on-premises infrastructure

**Real-world scenarios**:
- **Active Directory**: EC2 instances join AD domain using custom DNS
- **Hybrid DNS**: Resolve both AWS and on-premises hostnames
- **Compliance**: Use specific NTP servers for time synchronization

**Important limitation**: DHCP Options Sets **cannot be modified**—you must create a new one and associate it with the VPC.

---

### DNS Resolution & Hostnames

#### What is VPC DNS?

AWS provides DNS servers within every VPC to resolve:
- **Public DNS names** (e.g., ec2-54-123-45-67.compute-1.amazonaws.com)
- **Private DNS names** (e.g., ip-10-0-1-15.ec2.internal)
- **Custom Route 53 private hosted zones**

```
DNS Resolution in VPC:
┌──────────────────────────────────────────┐
│              VPC (10.0.0.0/16)           │
│                                          │
│  AWS DNS Server: 10.0.0.2 (VPC +2)      │
│  ┌───────────────────────────────────┐   │
│  │                                   │   │
│  │  EC2: 10.0.1.15                   │   │
│  │  ┌─────────────────────────────┐  │   │
│  │  │ Query: google.com           │  │   │
│  │  │    ↓                        │  │   │
│  │  │ Query: 10.0.0.2 (VPC DNS)   │  │   │
│  │  │    ↓                        │  │   │
│  │  │ Forward to public DNS       │  │   │
│  │  │    ↓                        │  │   │
│  │  │ Response: 142.250.x.x       │  │   │
│  │  └─────────────────────────────┘  │   │
│  └───────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

#### DNS Settings in VPC

Two critical settings control DNS behavior:

**1. enableDnsHostnames:**
- Determines if instances get public DNS names
- Default: `false` (except default VPC)
- When enabled: Instances with public IPs get public DNS names

**2. enableDnsSupport:**
- Determines if DNS resolution is supported
- Default: `true`
- Must be `true` for VPC DNS to work

```
Settings combinations:
┌──────────────────┬──────────────┬──────────────────┐
│ enableDnsSupport │ DnsHostnames │ Result           │
├──────────────────┼──────────────┼──────────────────┤
│ true             │ true         │ Full DNS support │
│ true             │ false        │ DNS works, no    │
│                  │              │ public hostnames │
│ false            │ true         │ No DNS resolution│
│ false            │ false        │ No DNS at all    │
└──────────────────┴──────────────┴──────────────────┘
```

#### Private DNS Names

```
Private DNS Naming:
┌────────────────────────────────────────────┐
│ Region: us-east-1                          │
│ Instance: 10.0.1.15                        │
│                                            │
│ Private DNS:                               │
│ ip-10-0-1-15.ec2.internal                  │
│                                            │
│ Pattern: ip-{private-ip}.ec2.internal      │
│ (dashes replace dots in IP)                │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│ Region: eu-west-1                          │
│ Instance: 172.31.10.20                     │
│                                            │
│ Private DNS:                               │
│ ip-172-31-10-20.eu-west-1.compute.internal │
│                                            │
│ Pattern: ip-{private-ip}.region.compute... │
└────────────────────────────────────────────┘
```

#### Route 53 Private Hosted Zones

```
Integration with Route 53:
┌──────────────────────────────────────────┐
│         Route 53 Private Hosted Zone     │
│         Domain: internal.company.com     │
│         Associated with: VPC-A, VPC-B    │
│                                          │
│  Records:                                │
│  app.internal.company.com → 10.0.1.15    │
│  db.internal.company.com  → 10.0.2.20    │
└──────────────┬───────────────────────────┘
               │
    ┌──────────┴──────────┐
    │                     │
┌───▼──────┐      ┌───────▼────┐
│  VPC-A   │      │   VPC-B    │
│          │      │            │
│  EC2     │      │   EC2      │
│  can     │      │   can      │
│  resolve │      │   resolve  │
└──────────┘      └────────────┘

Only instances in associated VPCs can resolve
```

---

### VPC Sharing

#### What is VPC Sharing?

**VPC Sharing** allows multiple AWS accounts to create resources (EC2, RDS, Lambda) in a shared, centrally-managed VPC. The VPC owner shares subnets with participant accounts.

Think of it as **renting out apartments** (subnets) in a building (VPC) you own.

```
Without VPC Sharing (Traditional):
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Account A    │  │ Account B    │  │ Account C    │
│  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
│  │ VPC-A  │  │  │  │ VPC-B  │  │  │  │ VPC-C  │  │
│  └────────┘  │  │  └────────┘  │  │  └────────┘  │
│              │  │              │  │              │
│ Need peering │  │ Need peering │  │ Need peering │
└──────────────┘  └──────────────┘  └──────────────┘
Complex mesh, many VPCs, many connections

With VPC Sharing:
┌──────────────────────────────────────────┐
│      Account A (Owner)                   │
│      ┌───────────────────────────────┐   │
│      │  Shared VPC                   │   │
│      │  ┌─────────┐  ┌─────────┐    │   │
│      │  │Subnet-1 │  │Subnet-2 │    │   │
│      │  └────┬────┘  └────┬────┘    │   │
│      └───────┼────────────┼─────────┘   │
└──────────────┼────────────┼─────────────┘
               │            │
        Shared │            │ Shared
               │            │
    ┌──────────▼──┐   ┌─────▼──────┐
    │ Account B   │   │ Account C  │
    │ (Participant)│   │(Participant)│
    │  ┌────────┐ │   │ ┌────────┐ │
    │  │  EC2   │ │   │ │  RDS   │ │
    │  └────────┘ │   │ └────────┘ │
    └─────────────┘   └────────────┘
Single VPC, centrally managed
```

#### How VPC Sharing Works

**Using AWS Resource Access Manager (RAM):**
```
1. VPC Owner (Account A):
   - Creates VPC with subnets
   - Shares specific subnets via RAM
   - Manages: VPC, IGW, route tables, NACLs

2. Participant (Account B, C):
   - Accepts share invitation
   - Launches resources in shared subnets
   - Manages: Only their own resources (EC2, RDS, etc.)
   - Cannot modify VPC/subnet configuration
```

**Ownership model:**
```
┌─────────────────────────────────────────────┐
│ VPC Owner Controls:                         │
│ ✓ VPC CIDR blocks                           │
│ ✓ Subnets                                   │
│ ✓ Route tables                              │
│ ✓ Internet Gateway                          │
│ ✓ NAT Gateway                               │
│ ✓ VPC endpoints                             │
│ ✓ Network ACLs                              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Participants Control:                       │
│ ✓ Their EC2 instances, RDS, Lambda, etc.    │
│ ✓ Security groups (their own)               │
│ ✓ Can reference owner's security groups     │
│ ✗ Cannot modify VPC configuration           │
└─────────────────────────────────────────────┘
```

#### Why VPC Sharing Matters

**Problems it solves:**
- **Simplified networking**: One VPC instead of many
- **Reduced complexity**: No VPC peering needed
- **Cost efficiency**: Share NAT Gateways, VPC endpoints
- **Centralized control**: Network team manages VPC, app teams deploy resources
- **Better IP management**: Single CIDR space, easier planning

**Real-world scenarios**:
- **Enterprise multi-account**: Central IT owns VPC, business units deploy apps
- **Development teams**: Each team has separate AWS account, shares network infra
- **Cost optimization**: Multiple accounts share expensive NAT Gateways
- **Compliance**: Central team enforces network policies across all accounts

---

### IPv6 Support

#### What is IPv6 in VPC?

AWS VPCs can operate in **dual-stack mode**, supporting both IPv4 and IPv6 addresses simultaneously. IPv6 provides a vastly larger address space and eliminates the need for NAT.

```
Dual-Stack VPC:
┌──────────────────────────────────────────┐
│ VPC                                      │
│ IPv4: 10.0.0.0/16                        │
│ IPv6: 2600:1f13:xxx::/56 (AWS-provided) │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Subnet                             │  │
│  │ IPv4: 10.0.1.0/24                  │  │
│  │ IPv6: 2600:1f13:xxx:1::/64         │  │
│  │                                    │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ EC2 Instance                 │  │  │
│  │  │ IPv4: 10.0.1.15              │  │  │
│  │  │ IPv6: 2600:1f13:xxx:1::5a    │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘

Both protocols work simultaneously
```

#### IPv6 Characteristics in AWS

**Key differences from IPv4:**
```
┌────────────────────┬─────────────────┬─────────────────┐
│ Feature            │ IPv4            │ IPv6            │
├────────────────────┼─────────────────┼─────────────────┤
│ Address format     │ 10.0.1.15       │ 2600:1f13:x::5a │
│ CIDR size (VPC)    │ /16 to /28      │ /56 (fixed)     │
│ CIDR size (Subnet) │ /16 to /28      │ /64 (fixed)     │
│ Addresses          │ Private + Public│ All public      │
│ NAT needed         │ Yes (for private)│ No             │
│ Internet access    │ Via IGW         │ Via IGW or EIGW │
│ Bring your own     │ Yes (BYOIP)     │ Yes (BYOIPv6)   │
└────────────────────┴─────────────────┴─────────────────┘
```

**All IPv6 addresses are public:**
- No concept of private IPv6 in AWS VPC
- All IPv6 addresses are globally unique
- Security controlled by security groups/NACLs (not address privacy)

#### Egress-Only Internet Gateway (EIGW)

```
Problem: IPv6 addresses are all public
Solution: Egress-Only Internet Gateway

┌──────────────────────────────────────────┐
│              VPC                         │
│  ┌────────────────────────────────────┐  │
│  │  Subnet (IPv6)                     │  │
│  │                                    │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ EC2 (IPv6-only instance)     │  │  │
│  │  │ 2600:1f13:xxx:1::10          │  │  │
│  │  └──────────┬───────────────────┘  │  │
│  └─────────────┼──────────────────────┘  │
│                │                         │
│       ┌────────▼──────────┐              │
│       │ Egress-Only IGW   │              │
│       │  (IPv6 only)      │              │
│       └────────┬──────────┘              │
└────────────────┼─────────────────────────┘
                 │
                 │ ← Outbound: Allowed
                 │ → Inbound: Blocked
                 │
         ┌───────▼──────┐
         │   Internet   │
         └──────────────┘

Behavior:
✓ EC2 can initiate connections to internet
✗ Internet cannot initiate connections to EC2
(Similar to NAT Gateway but for IPv6)
```

#### Why IPv6 Matters

**Problems it solves:**
- **Address exhaustion**: Unlimited public IPs (practically)
- **No NAT overhead**: Direct routing, better performance
- **Future-proofing**: Support for IPv6-only clients
- **Simplified architecture**: No private/public IP complexity

**Real-world scenarios**:
- **Mobile apps**: Many cellular networks prefer IPv6
- **IoT devices**: Large number of devices need unique addresses
- **Cost savings**: Eliminate NAT Gateway costs for IPv6 traffic
- **Global reach**: Better support for IPv6-only regions

**Adoption strategy:**
```
Phase 1: Dual-stack (IPv4 + IPv6)
  - Add IPv6 to existing VPC
  - Test applications
  - Gradual rollout

Phase 2: IPv6-preferred
  - New resources use IPv6 primarily
  - IPv4 as fallback

Phase 3: IPv6-only (future)
  - Some workloads go IPv6-only
  - Significant cost reduction
```

---