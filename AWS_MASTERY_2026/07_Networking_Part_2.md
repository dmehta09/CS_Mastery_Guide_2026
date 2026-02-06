# AWS Networking: Part 2 - Transit Gateway, Direct Connect, VPN

> **Continuation from Part 1 (VPC)**. This part covers AWS Transit Gateway, Direct Connect, and Site-to-Site VPN services.

---

## Transit Gateway

### What is Transit Gateway?

**AWS Transit Gateway** is a highly scalable cloud router that connects VPCs, VPN connections, and Direct Connect gateways through a central hub. Instead of creating complex mesh networks with multiple VPC peering connections, Transit Gateway acts as a **central hub** for all network traffic.

Think of it as a **network router** that sits in the middle and connects everything together.

```
WITHOUT Transit Gateway (Mesh complexity):
┌──────┐     ┌──────┐     ┌──────┐
│VPC-A │◄───►│VPC-B │◄───►│VPC-C │
└───┬──┘     └──┬───┘     └───┬──┘
    │  ╲        │         ╱   │
    │   ╲       │       ╱     │
    │    ╲      │     ╱       │
    │     ╲     │   ╱         │
    │      ╲    │ ╱           │
┌───▼───┐   ╲  ▼  ╱      ┌───▼───┐
│VPC-D  │◄───►├◄───────►│VPC-E  │
└───────┘     └─────────┘└───────┘

5 VPCs = 10 peering connections
N VPCs = N*(N-1)/2 connections
(Becomes unmanageable quickly)

WITH Transit Gateway (Hub-and-Spoke):
┌──────┐       ┌──────┐       ┌──────┐
│VPC-A │       │VPC-B │       │VPC-C │
└───┬──┘       └───┬──┘       └───┬──┘
    │              │              │
    └──────┬───────┼───────┬──────┘
           │       │       │
      ┌────▼───────▼───────▼────┐
      │   Transit Gateway       │
      │   (Central Hub)         │
      └────┬───────────────┬────┘
           │               │
    ┌──────▼──────┐   ┌───▼──────┐
    │   VPC-D     │   │  VPC-E   │
    └─────────────┘   └──────────┘

5 VPCs = 5 attachments
N VPCs = N attachments
(Scales linearly)
```

### Why Transit Gateway Exists

**Problems it solves:**
- **Scalability**: Supports thousands of VPCs (vs. peering limits)
- **Transitive routing**: VPC-A can reach VPC-C through Transit Gateway
- **Centralized management**: One place to manage all network connections
- **Simplified architecture**: Hub-and-spoke vs. complex mesh
- **Multi-region connectivity**: Connect networks across AWS regions
- **Hybrid cloud**: Single connection point for on-premises networks

**Real-world scenarios**:
- **Enterprise with 100+ VPCs**: Each business unit, team, or environment has own VPC
- **Multi-region architecture**: Production in us-east-1, DR in eu-west-1, connect via Transit Gateway
- **Hybrid cloud**: On-premises datacenter connects to all AWS VPCs through single VPN/Direct Connect
- **Acquisition integration**: Quickly connect acquired company's VPCs to your network

---

### Transit Gateway Attachments

#### What are Attachments?

**Attachments** connect resources to the Transit Gateway. They're the "cables" that plug into the hub.

```
Transit Gateway Hub:
        ┌─────────────────────────┐
        │  Transit Gateway        │
        └───┬───┬───┬────┬────┬───┘
            │   │   │    │    │
   ┌────────┘   │   │    │    └──────────┐
   │            │   │    │               │
   │            │   │    │               │
┌──▼───┐  ┌────▼┐ ┌▼─┐ ┌▼────────┐  ┌──▼──────┐
│ VPC  │  │VPC  │ │VPN│ │Direct   │  │VPC      │
│Attach│  │Attach│ │   │ │Connect  │  │(Another │
│      │  │     │ │   │ │Gateway  │  │ region) │
└──────┘  └─────┘ └───┘ └─────────┘  └─────────┘
```

#### Types of Attachments

**1. VPC Attachment**
```
Connects a VPC to Transit Gateway

Configuration:
- Choose VPC
- Choose subnet in each AZ (for HA)
- Transit Gateway creates ENI in each subnet

┌─────────────────────────────────────┐
│         VPC (10.0.0.0/16)           │
│                                     │
│  AZ-1           AZ-2                │
│  ┌──────┐      ┌──────┐             │
│  │Subnet│      │Subnet│             │
│  │ ┌──┐│      │ ┌──┐ │             │
│  │ │ENI││      │ │ENI│ │ ← TGW ENIs │
│  │ └──┘│      │ └──┘ │             │
│  └───┬──┘      └───┬──┘             │
└──────┼─────────────┼────────────────┘
       └──────┬──────┘
              │
       ┌──────▼──────┐
       │   Transit   │
       │   Gateway   │
       └─────────────┘
```

**2. VPN Attachment**
```
Connects Site-to-Site VPN to Transit Gateway

┌─────────────────┐
│  On-Premises    │
│  Datacenter     │
│  ┌───────────┐  │
│  │  Router   │  │
│  └─────┬─────┘  │
└────────┼────────┘
         │ VPN Tunnel
         │ (encrypted)
    ┌────▼─────────┐
    │ Transit      │
    │ Gateway      │
    │ VPN Attach   │
    └──────────────┘
```

**3. Direct Connect Gateway Attachment**
```
Connects Direct Connect to Transit Gateway

┌─────────────────┐
│  On-Premises    │
│  ┌───────────┐  │
│  │  Router   │  │
│  └─────┬─────┘  │
└────────┼────────┘
         │ Dedicated fiber
         │ (1-100 Gbps)
    ┌────▼─────────────┐
    │ Direct Connect   │
    │ Gateway          │
    └────┬─────────────┘
         │
    ┌────▼─────────┐
    │ Transit      │
    │ Gateway      │
    └──────────────┘
```

**4. Peering Attachment (Inter-Region)**
```
Connects Transit Gateways in different regions

us-east-1                          eu-west-1
┌─────────────┐                   ┌─────────────┐
│  Transit    │◄─────────────────►│  Transit    │
│  Gateway    │  Peering Attach   │  Gateway    │
│  (TGW-US)   │                   │  (TGW-EU)   │
└──────┬──────┘                   └──────┬──────┘
       │                                 │
   VPCs in US                        VPCs in EU
```

**5. Connect Attachment**
```
Connects third-party network appliances via GRE/IPsec

┌──────────────────────────────────┐
│     VPC                          │
│  ┌────────────────────────────┐  │
│  │  SD-WAN Appliance          │  │
│  │  (Cisco, Palo Alto, etc.)  │  │
│  └─────────┬──────────────────┘  │
└────────────┼─────────────────────┘
             │ GRE tunnel
        ┌────▼─────────┐
        │ Transit      │
        │ Gateway      │
        │ Connect      │
        └──────────────┘
```

---

### Transit Gateway Route Tables

#### What are TGW Route Tables?

Just like VPC route tables, **Transit Gateway route tables** determine where traffic is routed. However, they control routing **between attachments**, not within a VPC.

```
Routing Flow:
┌─────────────────────────────────────────────┐
│ Traffic from VPC-A going to 192.168.0.0/16  │
│              ↓                              │
│ VPC Route Table: Send to Transit Gateway   │
│              ↓                              │
│ Transit Gateway receives packet             │
│              ↓                              │
│ TGW Route Table: Where should this go?     │
│   Destination: 192.168.0.0/16              │
│   Target: VPC-B attachment                  │
│              ↓                              │
│ Forward to VPC-B                            │
└─────────────────────────────────────────────┘
```

#### Route Table Association and Propagation

**Two key concepts:**

**1. Association**: Which attachments use which route table
**2. Propagation**: Which attachments automatically advertise their routes

```
Example Setup:
┌──────────────────────────────────────────┐
│ Transit Gateway                          │
│                                          │
│ Route Table: Production-RT               │
│ ┌──────────────────────────────────────┐ │
│ │ Associated: VPC-Prod attachment      │ │
│ │ Propagated: VPC-DB, On-Prem-VPN      │ │
│ │                                      │ │
│ │ Routes (auto-learned):               │ │
│ │ 10.1.0.0/16 → VPC-DB                 │ │
│ │ 192.168.0.0/16 → On-Prem-VPN         │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘

Result:
VPC-Prod can reach:
- VPC-DB (10.1.0.0/16)
- On-Premises (192.168.0.0/16)
```

#### Network Segmentation with Route Tables

**Use case**: Isolate development from production

```
┌─────────────────────────────────────────────┐
│           Transit Gateway                   │
│                                             │
│  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Production-RT   │  │ Development-RT  │  │
│  │                 │  │                 │  │
│  │ Associated:     │  │ Associated:     │  │
│  │ - VPC-Prod      │  │ - VPC-Dev       │  │
│  │ - VPC-DB        │  │ - VPC-Test      │  │
│  │                 │  │                 │  │
│  │ Propagated:     │  │ Propagated:     │  │
│  │ - VPC-Prod      │  │ - VPC-Dev       │  │
│  │ - VPC-DB        │  │ - VPC-Test      │  │
│  │ - On-Prem       │  │ (Not on-prem)   │  │
│  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────┘

Effect:
✓ Production can reach database and on-premises
✓ Development can reach test environment
✗ Development CANNOT reach production or on-prem
```

---

### Transit Gateway Peering Attachments

#### What is TGW Peering?

**Transit Gateway Peering** connects Transit Gateways in different regions, enabling cross-region routing through the AWS global network.

```
Multi-Region Architecture:
┌────────────────────────────────────────────┐
│            us-east-1                       │
│  ┌──────┐  ┌──────┐  ┌──────┐             │
│  │VPC-A │  │VPC-B │  │VPC-C │             │
│  └───┬──┘  └───┬──┘  └───┬──┘             │
│      └──────┬──┴─────────┘                │
│             │                              │
│      ┌──────▼────────┐                     │
│      │ TGW-US-East   │                     │
│      └──────┬────────┘                     │
└─────────────┼──────────────────────────────┘
              │
              │ Peering Attachment
              │ (AWS Backbone)
              │
┌─────────────┼──────────────────────────────┐
│      ┌──────▼────────┐       eu-west-1    │
│      │  TGW-EU-West  │                     │
│      └──────┬────────┘                     │
│             │                              │
│      ┌──────┴──────────┐                   │
│  ┌───▼──┐  ┌──────┐  ┌─▼─────┐            │
│  │VPC-D │  │VPC-E │  │VPC-F  │            │
│  └──────┘  └──────┘  └───────┘            │
└────────────────────────────────────────────┘

VPC-A (US) can now communicate with VPC-D (EU)
```

#### Peering Characteristics

**Key properties:**
- **Cross-region only**: Can't peer TGWs in same region
- **AWS global network**: Traffic uses AWS backbone (not internet)
- **Bandwidth**: Up to 50 Gbps per peering
- **Static routes**: Must configure routes (not automatic propagation across regions)
- **No transitive peering**: Can't daisy-chain TGWs

```
Supported:
TGW-A (us-east-1) ◄──► TGW-B (eu-west-1)

Not supported (transitive):
TGW-A ◄──► TGW-B ◄──► TGW-C
   ▲                      ▲
   └──────────✗───────────┘
```

---

### Transit Gateway Multicast

#### What is Multicast?

**Multicast** allows one sender to transmit data to multiple receivers efficiently. Transit Gateway supports multicast, enabling this within your VPC networks.

```
Unicast (traditional):
Sender sends 3 separate copies
┌────────┐   ┌────────┐   ┌────────┐
│Receiver│   │Receiver│   │Receiver│
│   A    │   │   B    │   │   C    │
└───▲────┘   └───▲────┘   └───▲────┘
    │            │            │
    │            │            │
┌───┴────────────┴────────────┴───┐
│  Network sends 3 copies          │
│  Wastes bandwidth                │
└──────────────┬───────────────────┘
           ┌───▼───┐
           │Sender │
           └───────┘

Multicast:
Sender sends 1 copy, network replicates
┌────────┐   ┌────────┐   ┌────────┐
│Receiver│   │Receiver│   │Receiver│
│   A    │   │   B    │   │   C    │
└───▲────┘   └───▲────┘   └───▲────┘
    │            │            │
┌───┴────────────┴────────────┴───┐
│  Network replicates efficiently  │
│  Single transmission             │
└──────────────┬───────────────────┘
           ┌───▼───┐
           │Sender │
           └───────┘
```

#### Multicast Use Cases

**Real-world scenarios**:
- **Video streaming**: Corporate live broadcasts to thousands of employees
- **Financial data feeds**: Stock market data to trading applications
- **Gaming**: Multiplayer game state synchronization
- **Industrial IoT**: Sensor data to multiple monitoring systems

---

### Transit Gateway Connect Attachments

#### What is TGW Connect?

**Connect attachments** enable integration with SD-WAN and third-party network appliances using standard protocols (GRE, BGP).

```
Traditional VPN limitations:
- IPsec overhead
- Complex key management
- Limited throughput

Connect solution:
- GRE tunnels (lightweight)
- BGP for dynamic routing
- Higher performance
- Native SD-WAN integration

┌──────────────────────────────────────┐
│         VPC                          │
│  ┌────────────────────────────────┐  │
│  │  SD-WAN Appliance (EC2)        │  │
│  │  - Cisco Viptela               │  │
│  │  - VMware VeloCloud            │  │
│  │  - Silver Peak                 │  │
│  └──────────┬─────────────────────┘  │
└─────────────┼────────────────────────┘
              │ GRE + BGP
        ┌─────▼───────────┐
        │ Transit Gateway │
        │ Connect         │
        └─────┬───────────┘
              │
        All TGW attachments
        (VPCs, VPN, DX, etc.)
```

---

### Inter-Region Peering

#### Advanced Multi-Region Patterns

**Global backbone architecture:**
```
us-east-1              eu-west-1            ap-southeast-1
┌────────┐             ┌────────┐           ┌────────┐
│ TGW-US │◄───────────►│ TGW-EU │◄─────────►│ TGW-AP │
└───┬────┘  Peering    └───┬────┘  Peering  └───┬────┘
    │                      │                    │
  VPCs                   VPCs                 VPCs

Benefits:
- Any VPC can reach any other VPC globally
- Traffic stays on AWS backbone
- Centralized network management
- Disaster recovery across regions
```

**Why Transit Gateway Matters (Summary)**

**Problems solved:**
- **Scale**: Manage thousands of VPCs from single control plane
- **Complexity reduction**: Replace mesh peering with hub-and-spoke
- **Global connectivity**: Seamlessly connect regions
- **Hybrid cloud**: Single integration point for on-premises
- **Cost efficiency**: Reduce data transfer costs with centralized routing
- **Security**: Centralized traffic inspection points

---

## Direct Connect

### What is AWS Direct Connect?

**AWS Direct Connect** is a dedicated, private network connection between your on-premises datacenter and AWS. Instead of using the public internet, you get a **physical fiber connection** directly to AWS.

Think of it as building a **private highway** between your office and AWS, bypassing all public roads (internet).

```
WITHOUT Direct Connect (Over Internet):
┌──────────────┐                     ┌────────────┐
│ On-Premises  │                     │    AWS     │
│ Datacenter   │                     │    VPC     │
│              │                     │            │
│  ┌────────┐  │                     │ ┌────────┐ │
│  │Servers │  │  Public Internet    │ │  EC2   │ │
│  └────┬───┘  │  ┌──────────────┐   │ └────────┘ │
└───────┼──────┘  │ Unpredictable│   └────────────┘
        │         │   latency    │         ▲
        │         │   bandwidth  │         │
        └────────►│   security   │─────────┘
                  │   risks      │
                  └──────────────┘

WITH Direct Connect:
┌──────────────┐                     ┌────────────┐
│ On-Premises  │                     │    AWS     │
│ Datacenter   │                     │    VPC     │
│              │                     │            │
│  ┌────────┐  │  ┌──────────────┐   │ ┌────────┐ │
│  │Servers │  │  │Direct Connect│   │ │  EC2   │ │
│  └────┬───┘  │  │Private Fiber │   │ └────────┘ │
└───────┼──────┘  │ - Predictable│   └────────────┘
        │         │ - Dedicated  │         ▲
        │         │ - Secure     │         │
        └────────►│ - Fast       │─────────┘
                  └──────────────┘
```

### Why Direct Connect Exists

**Problems it solves:**
- **Bandwidth**: Consistent, high-throughput connection (1 Gbps to 100 Gbps)
- **Latency**: Lower, more predictable latency than internet
- **Security**: Traffic never touches public internet
- **Cost**: Reduce data transfer costs for high-volume workloads
- **Compliance**: Meet regulatory requirements for private connectivity

**Real-world scenarios**:
- **Data migration**: Transfer petabytes to AWS (faster than internet)
- **Hybrid cloud**: Keep sensitive data on-premises, compute in AWS
- **Disaster recovery**: Real-time replication to AWS
- **Content distribution**: Media companies uploading large video files
- **Financial services**: Low-latency trading systems

---

### Dedicated Connections

#### What are Dedicated Connections?

A **Dedicated Connection** is a physical Ethernet port dedicated exclusively to you at an AWS Direct Connect location. You own the entire bandwidth.

```
Your dedicated port:
┌─────────────────────────────────────────┐
│  AWS Direct Connect Location (Equinix)  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  Your Dedicated Port              │  │
│  │  - 1 Gbps, 10 Gbps, or 100 Gbps   │  │
│  │  - Nobody else uses this          │  │
│  │  - Your router ◄──► AWS router    │  │
│  └───────────────┬───────────────────┘  │
└──────────────────┼──────────────────────┘
                   │
                   │ Fiber to AWS
                   │
            ┌──────▼──────┐
            │  AWS Region │
            └─────────────┘
```

**Bandwidth options:**
- **1 Gbps**: Most common for enterprises
- **10 Gbps**: High-traffic applications
- **100 Gbps**: Massive data transfer (video, genomics)

**Setup process:**
1. Request port at Direct Connect location
2. AWS assigns you a port
3. You arrange cross-connect (fiber from your rack to AWS rack)
4. Configure BGP peering
5. Create Virtual Interfaces (VIFs)

---

### Hosted Connections

#### What are Hosted Connections?

A **Hosted Connection** is provided by an AWS Direct Connect Partner. Instead of dedicating an entire port to you, the partner shares their connection and provisions a portion to you.

Think of it like **renting an apartment** (hosted) vs **buying a house** (dedicated).

```
Hosted Connection (via Partner):
┌──────────────────────────────────────────┐
│  AWS Direct Connect Location             │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  Partner's Dedicated Port (10 Gbps)│  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ Customer A: 500 Mbps         │  │  │
│  │  │ Customer B: 1 Gbps    ← You  │  │  │
│  │  │ Customer C: 2 Gbps           │  │  │
│  │  │ Partner uses rest            │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────┬───────────────────┘  │
└───────────────────┼──────────────────────┘
                    │
```

**Bandwidth options:**
- 50 Mbps, 100 Mbps, 200 Mbps, 300 Mbps, 400 Mbps, 500 Mbps
- 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps

**Dedicated vs Hosted:**
```
┌──────────────────┬─────────────┬──────────────┐
│ Feature          │ Dedicated   │ Hosted       │
├──────────────────┼─────────────┼──────────────┤
│ Bandwidth        │ 1/10/100 Gbps│ 50 Mbps-10 Gbps│
│ Setup time       │ Weeks       │ Days         │
│ Provider         │ You + AWS   │ Partner      │
│ Cost             │ Higher      │ Lower        │
│ Flexibility      │ Full control│ Partner mgmt │
│ Use case         │ Enterprise  │ SMB/startup  │
└──────────────────┴─────────────┴──────────────┘
```

---

### Virtual Interfaces (VIFs)

#### What are Virtual Interfaces?

**Virtual Interfaces** are logical connections configured on top of your physical Direct Connect. They define what you can access and how.

```
Physical Connection (1 dedicated port):
┌──────────────────────────────────────┐
│  Direct Connect (10 Gbps port)       │
│  ┌────────────────────────────────┐  │
│  │  Private VIF #1                │  │ ← Access VPC-A
│  │  VLAN 101, BGP ASN 65001       │  │
│  ├────────────────────────────────┤  │
│  │  Private VIF #2                │  │ ← Access VPC-B
│  │  VLAN 102, BGP ASN 65001       │  │
│  ├────────────────────────────────┤  │
│  │  Public VIF                    │  │ ← Access S3, DynamoDB
│  │  VLAN 103, Public IPs          │  │
│  ├────────────────────────────────┤  │
│  │  Transit VIF                   │  │ ← Access Transit Gateway
│  │  VLAN 104, BGP ASN 65002       │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

One physical connection → Multiple logical connections
```

#### Types of Virtual Interfaces

**1. Private VIF:**
```
Connects to: VPCs via Virtual Private Gateway

Use case: Access EC2, RDS in private subnets
┌──────────────┐         ┌─────────────┐
│ On-Premises  │         │    VPC      │
│              │ Private │             │
│  Servers ────┼─────────┼──► EC2      │
│              │  VIF    │             │
└──────────────┘         └─────────────┘
```

**2. Public VIF:**
```
Connects to: AWS public services

Use case: Access S3, DynamoDB, AWS APIs
┌──────────────┐         ┌──────────────┐
│ On-Premises  │         │ AWS Services │
│              │ Public  │              │
│  Backup ─────┼─────────┼──► S3        │
│  System      │  VIF    │  DynamoDB    │
└──────────────┘         └──────────────┘

Note: Uses AWS public IPs, but traffic stays on
AWS network (doesn't touch internet)
```

**3. Transit VIF:**
```
Connects to: Transit Gateway

Use case: Access multiple VPCs through Transit Gateway
┌──────────────┐         ┌───────────────┐
│ On-Premises  │ Transit │ Transit GW    │
│              │  VIF    │   ├─► VPC-A   │
│  Servers ────┼─────────┤   ├─► VPC-B   │
│              │         │   └─► VPC-C   │
└──────────────┘         └───────────────┘

Benefit: One VIF accesses all VPCs
```

---

### Direct Connect Gateway

#### What is Direct Connect Gateway?

**Direct Connect Gateway** allows you to connect your Direct Connect to multiple VPCs in different regions from a single connection. Without it, you'd need separate connections per region.

```
WITHOUT Direct Connect Gateway:
┌─────────────┐   ┌────────────────┐   ┌─────────────┐
│On-Premises  │   │ Direct Connect │   │  us-east-1  │
│             │──►│  Connection 1  │──►│   VPC-A     │
└─────────────┘   └────────────────┘   └─────────────┘
┌─────────────┐   ┌────────────────┐   ┌─────────────┐
│On-Premises  │   │ Direct Connect │   │  eu-west-1  │
│             │──►│  Connection 2  │──►│   VPC-B     │
└─────────────┘   └────────────────┘   └─────────────┘
Need multiple connections (expensive!)

WITH Direct Connect Gateway:
┌─────────────┐   ┌────────────────┐
│On-Premises  │   │ Direct Connect │
│             │──►│  Connection    │
└─────────────┘   └────────┬───────┘
                           │
                  ┌────────▼─────────┐
                  │Direct Connect    │
                  │Gateway (Global)  │
                  └────┬────────┬────┘
                       │        │
              ┌────────▼─┐   ┌─▼─────────┐
              │us-east-1 │   │ eu-west-1 │
              │  VPC-A   │   │  VPC-B    │
              └──────────┘   └───────────┘
One connection → Multiple regions!
```

#### Direct Connect Gateway Benefits

**Key capabilities:**
- Connect to **up to 10 VPCs** across any AWS region
- **Global resource**: Not tied to a specific region
- **No additional charges**: Included with Direct Connect
- **Private connectivity**: All traffic stays on AWS backbone

**Limitations:**
- VPCs must have **non-overlapping CIDRs**
- Cannot use for VPC-to-VPC communication (use Transit Gateway for that)
- Maximum 10 VPC associations

---

### Link Aggregation Groups (LAG)

#### What is LAG?

**Link Aggregation Group** bundles multiple physical Direct Connect connections into a single logical connection for higher bandwidth and redundancy.

```
Single Connection (10 Gbps):
┌──────────┐      10 Gbps       ┌─────┐
│Your      │◄─────────────────►│ AWS │
│Router    │                    └─────┘
└──────────┘
Max: 10 Gbps
If fails: Total outage

LAG (4 x 10 Gbps = 40 Gbps):
┌──────────┐     ┌────────┐     ┌─────┐
│Your      │◄───►│  LAG   │◄───►│ AWS │
│Router    │     │ 10 Gbps│     └─────┘
│          │◄───►│ 10 Gbps│
│          │     │ 10 Gbps│
│          │◄───►│ 10 Gbps│
└──────────┘     └────────┘
Max: 40 Gbps
If 1 fails: 30 Gbps remains (HA)
```

**LAG requirements:**
- All connections must be **same bandwidth**
- All connections must terminate at **same Direct Connect location**
- Minimum 1, maximum 4 connections per LAG

---

### Site-to-Site VPN as Backup

#### Hybrid Approach: Direct Connect + VPN

For **maximum availability**, combine Direct Connect with VPN backup:

```
Normal operation (Direct Connect):
┌──────────────┐                  ┌────────────┐
│ On-Premises  │ Direct Connect   │    AWS     │
│              │◄────────────────►│    VPC     │
│              │  (Primary)       │            │
└──────────────┘                  └────────────┘

Direct Connect fails:
┌──────────────┐                  ┌────────────┐
│ On-Premises  │ VPN over Internet│    AWS     │
│              │◄────────────────►│    VPC     │
│              │  (Backup)        │            │
└──────────────┘                  └────────────┘

Both active (active-active):
┌──────────────┐  Direct Connect  ┌────────────┐
│ On-Premises  │◄────────────────►│    AWS     │
│              │  VPN (backup)    │    VPC     │
│              │◄────────────────►│            │
└──────────────┘                  └────────────┘
BGP prefers Direct Connect (lower metrics)
VPN takes over if DC fails
```

**Configuration:**
- Use BGP to advertise routes over both connections
- Configure **route preferences** (AS path prepending or local preference)
- Direct Connect routes preferred by default
- VPN routes used only if Direct Connect down

---

### Why Direct Connect Matters (Summary)

**Problems solved:**
- **Bandwidth**: Dedicated, consistent throughput (not shared internet)
- **Latency**: Predictable, lower latency than internet
- **Cost savings**: Reduce data transfer charges
- **Security**: Private connection, never touches internet
- **Compliance**: Meet regulatory requirements
- **Reliability**: SLA-backed availability with LAG and VPN backup

**When to use Direct Connect:**
- Transferring large amounts of data regularly (> 1 TB/month)
- Latency-sensitive applications (video streaming, trading)
- Hybrid cloud architectures
- Compliance requirements for private connectivity

**When VPN might be sufficient:**
- Low-bandwidth needs (< 1 Gbps)
- Budget-constrained projects
- Temporary/proof-of-concept workloads
- Backup connectivity only

---

## Site-to-Site VPN

### What is AWS Site-to-Site VPN?

**AWS Site-to-Site VPN** creates encrypted IPsec tunnels between your on-premises network and your AWS VPC over the public internet. It's a cost-effective way to establish hybrid cloud connectivity.

Think of it as a **secure tunnel through the internet** rather than a dedicated fiber (Direct Connect).

```
VPN Architecture:
┌──────────────────┐                    ┌──────────────────┐
│  On-Premises     │   Encrypted IPsec  │    AWS VPC       │
│  Datacenter      │        Tunnel      │                  │
│                  │  ╔════════════════╗ │                  │
│  ┌────────────┐  │  ║    Internet    ║ │  ┌────────────┐ │
│  │  Customer  │──┼──╫────────────────╫─┼──│  Virtual   │ │
│  │  Gateway   │  │  ║  (Public, but  ║ │  │  Private   │ │
│  │  (Router)  │  │  ║   encrypted)   ║ │  │  Gateway   │ │
│  └────────────┘  │  ╚════════════════╝ │  └────────────┘ │
└──────────────────┘                    └──────────────────┘

Traffic is encrypted end-to-end
Travels over internet (cost-effective)
AWS manages VPN endpoints
```

### Why Site-to-Site VPN Exists

**Problems it solves:**
- **Quick setup**: Operational in hours vs weeks (Direct Connect)
- **Cost-effective**: No dedicated circuit costs
- **Hybrid cloud**: Extend on-premises network to AWS
- **Disaster recovery**: Backup connectivity for Direct Connect
- **Multi-site**: Connect branch offices to AWS

**Comparison with Direct Connect:**
```
┌──────────────────┬────────────────┬────────────────┐
│ Feature          │ Site-to-Site   │ Direct Connect │
│                  │ VPN            │                │
├──────────────────┼────────────────┼────────────────┤
│ Setup time       │ Hours          │ Weeks          │
│ Bandwidth        │ Up to 1.25 Gbps│ 1-100 Gbps     │
│ Latency          │ Variable       │ Consistent     │
│ Cost             │ Low            │ High           │
│ Encryption       │ Built-in       │ Optional       │
│ Use internet     │ Yes            │ No             │
│ Best for         │ Small/medium   │ Enterprise     │
└──────────────────┴────────────────┴────────────────┘
```

---

### Virtual Private Gateway (VGW)

#### What is Virtual Private Gateway?

**Virtual Private Gateway (VGW)** is the VPN concentrator on the AWS side. It's attached to your VPC and terminates VPN connections from your on-premises network.

```
VPN Connection Components:
┌───────────────────────────────────────────┐
│            AWS VPC                        │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │  Virtual Private Gateway (VGW)      │  │ ← AWS-managed VPN endpoint
│  │  - Two redundant endpoints          │  │
│  │  - Highly available                 │  │
│  └──────┬──────────────────────┬───────┘  │
│         │                      │          │
│    ┌────▼────┐           ┌────▼────┐     │
│    │ Tunnel 1│           │ Tunnel 2│     │ ← Redundant tunnels
│    └────┬────┘           └────┬────┘     │
└─────────┼───────────────────────┼─────────┘
          │    Encrypted IPsec    │
          │                       │
┌─────────▼───────────────────────▼─────────┐
│         Public Internet                   │
└───────────┬───────────────────────────────┘
            │
┌───────────▼────────────────────────────────┐
│  On-Premises                               │
│  ┌──────────────────────────────────────┐  │
│  │  Customer Gateway Device             │  │ ← Your router
│  │  (Cisco, Juniper, Palo Alto, etc.)   │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

**VGW characteristics:**
- **Highly available**: AWS manages redundancy automatically
- **Two VPN endpoints**: In different Availability Zones
- **Route propagation**: Can automatically update VPC route tables
- **No additional cost**: Included with VPN connection pricing

---

### Customer Gateway (CGW)

#### What is Customer Gateway?

**Customer Gateway** represents your on-premises VPN device in AWS configuration. It's metadata about your router, not a physical device AWS provides.

```
Customer Gateway Configuration:
┌─────────────────────────────────────┐
│  AWS Console/API Input:             │
│                                     │
│  1. Public IP: 203.0.113.5          │ ← Your router's internet IP
│  2. BGP ASN: 65000                  │ ← For dynamic routing
│  3. Device type: (optional)         │ ← Help docs selection
└─────────────────────────────────────┘

AWS creates:
┌─────────────────────────────────────┐
│  Customer Gateway Resource          │
│  cgw-0123456789abcdef0              │
│                                     │
│  - Stores your router info          │
│  - Used to create VPN connection    │
│  - No actual device in AWS          │
└─────────────────────────────────────┘
```

---

### VPN Connections

#### Creating a VPN Connection

**Setup process:**
```
1. Create Customer Gateway
   └─ Enter your router's public IP and BGP ASN

2. Create Virtual Private Gateway
   └─ Attach to your VPC

3. Create VPN Connection
   ├─ Choose CGW and VGW
   ├─ Select routing type (static or dynamic)
   └─ AWS creates two tunnels automatically

4. Download configuration file
   ├─ AWS generates router config
   ├─ Specific to your device (Cisco, Juniper, etc.)
   └─ Contains tunnel IPs, pre-shared keys, BGP settings

5. Configure your router
   └─ Apply downloaded configuration
```

#### Routing: Static vs Dynamic (BGP)

**Static routing:**
```
You manually specify routes:
┌────────────────────────────────────┐
│ Your router config:                │
│ - Route 10.0.0.0/16 to AWS tunnel  │
│                                    │
│ AWS VPN config:                    │
│ - Route 192.168.0.0/16 to your CGW │
└────────────────────────────────────┘

Pros: Simple
Cons: Manual updates, no failover automation
```

**Dynamic routing (BGP):**
```
Routes learned automatically via BGP:
┌────────────────────────────────────┐
│ Your router:                       │
│ - Advertises 192.168.0.0/16 via BGP│
│                                    │
│ AWS VGW:                           │
│ - Advertises 10.0.0.0/16 via BGP   │
│ - Auto-failover between tunnels    │
└────────────────────────────────────┘

Pros: Automatic updates, tunnel failover
Cons: Requires BGP-capable router
```

#### Tunnel Redundancy

**High availability design:**
```
AWS provides TWO tunnels per VPN connection:
┌──────────────────────────────────────┐
│  Tunnel 1: 52.1.2.3 ◄─► Your Router  │ ← Active
│  Tunnel 2: 52.4.5.6 ◄─► Your Router  │ ← Standby
└──────────────────────────────────────┘

If Tunnel 1 fails:
┌──────────────────────────────────────┐
│  Tunnel 1: 52.1.2.3 ◄─► Your Router  │ ← Failed
│  Tunnel 2: 52.4.5.6 ◄─► Your Router  │ ← Now active
└──────────────────────────────────────┘

With BGP: Automatic failover (seconds)
Without BGP: Manual failover required
```

---

### Accelerated VPN

#### What is Accelerated VPN?

**Accelerated Site-to-Site VPN** uses AWS Global Accelerator to route VPN traffic through the AWS global network instead of the public internet, providing better performance.

```
Regular VPN:
┌──────────┐                              ┌────────┐
│  Your    │    Public Internet           │  AWS   │
│  Office  │◄────────────────────────────►│  VPC   │
└──────────┘  (Variable latency/quality)  └────────┘

Accelerated VPN:
┌──────────┐                              ┌────────┐
│  Your    │  Nearest AWS edge location   │  AWS   │
│  Office  │────┐                    ┌───►│  VPC   │
└──────────┘    │  AWS Global        │    └────────┘
                │  Accelerator       │
                └────────────────────┘
                (Optimized AWS backbone)
```

**Benefits:**
- **Lower latency**: Traffic on AWS network ASAP
- **Better performance**: More predictable throughput
- **Improved jitter**: Consistent packet delivery

**Cost:**
- Additional hourly charge + data transfer fees
- Worth it for latency-sensitive applications

---

### VPN CloudHub

#### What is VPN CloudHub?

**VPN CloudHub** enables communication between multiple on-premises sites through AWS as a hub. It's like using AWS as a central router for your branch offices.

```
Traditional (site-to-site mesh):
┌─────────┐     ┌─────────┐
│ Office A│◄───►│ Office B│
└────┬────┘     └────┬────┘
     │   ╲        ╱  │
     │    ╲      ╱   │
     │     ╲    ╱    │
     │      ╲  ╱     │
     │       ╲╱      │
     │       ╱╲      │
     │      ╱  ╲     │
     │     ╱    ╲    │
     │    ╱      ╲   │
┌────▼───╱        ╲──▼────┐
│Office C│◄───────►│Office D│
└────────┘         └────────┘
Complex: N*(N-1)/2 connections

VPN CloudHub (hub-and-spoke):
┌─────────┐       ┌─────────┐
│ Office A│       │ Office B│
└────┬────┘       └────┬────┘
     │                 │
     │    ┌────────┐   │
     └───►│  AWS   │◄──┘
          │  VPC   │
          │(CloudHub)
          └───┬────┘
              │
     ┌────────┴────────┐
     │                 │
┌────▼────┐       ┌────▼────┐
│ Office C│       │ Office D│
└─────────┘       └─────────┘
Simple: N connections
```

**Configuration:**
```
1. Create one Virtual Private Gateway
2. Create multiple Customer Gateways (one per site)
3. Create multiple VPN connections to same VGW
4. Enable BGP on all connections
5. Sites learn each other's routes via BGP

Result:
Office A can reach Office B through AWS
Office C can reach Office D through AWS
All traffic encrypted
```

**Use cases:**
- Branch office connectivity
- Multi-region corporate network
- Backup for MPLS network
- Disaster recovery sites

---

### Why Site-to-Site VPN Matters (Summary)

**Problems solved:**
- **Quick deployment**: Hours instead of weeks
- **Cost-effective**: No dedicated circuit
- **Encrypted**: Built-in security
- **Flexible**: Easy to add/remove sites
- **Backup**: Ideal failover for Direct Connect

**When to use VPN:**
- Budget-constrained projects
- Temporary connections
- Low-bandwidth requirements (< 1 Gbps)
- Backup/DR connectivity
- Branch offices with intermittent needs

**When to upgrade to Direct Connect:**
- Consistent high bandwidth needed (> 1 Gbps)
- Latency-critical applications
- Large data transfers
- Production workloads with SLA requirements

---