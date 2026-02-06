# System Design Thinking

## Overview

System Design Thinking is the disciplined approach to planning, architecting, and building software systems that meet real-world requirements while balancing multiple competing constraints. It's the bridge between abstract business needs and concrete technical implementations.

Think of it like designing a building: you need to understand what the building will be used for (requirements), estimate costs and materials (calculations), and make choices about where to invest resources (trade-offs). System design applies this same rigorous thinking to software systems.

---

## 1. Requirements Engineering

Requirements Engineering is the systematic process of discovering, analyzing, documenting, and validating what a system must do and how well it must do it. It's the foundation upon which all design decisions are made.

### Why Requirements Engineering Matters

Without clear requirements, teams build the wrong thing, waste resources, and create systems that fail to meet user needs. Good requirements engineering prevents costly rework and ensures stakeholder alignment from day one.

```
Project Success Flow:

Clear Requirements → Aligned Design → Correct Implementation → Satisfied Users
       ↓
  Unclear/Missing Requirements → Misaligned Design → Rework Cycles → Failed Project
```

---

### 1.1 Functional vs Non-Functional Requirements Decomposition

#### Functional Requirements (FRs)

**Definition:** Functional requirements describe *what the system must do*—the specific behaviors, features, and operations it must support.

**Layman explanation:** These are the "action items" of your system. If you were designing a car, functional requirements would be: "The car must accelerate," "The car must brake," "The car must turn left and right."

**Characteristics:**
- Action-oriented (verbs: create, delete, update, send, process)
- Directly tied to user or system behaviors
- Testable through functional testing
- Often form the basis of user stories

**Examples:**

| System Type | Functional Requirement |
|-------------|------------------------|
| E-commerce | Users must be able to add items to a shopping cart |
| Banking App | System must transfer funds between accounts |
| Social Media | Users must be able to post photos with captions |
| Email Service | System must send emails within 5 seconds of user clicking "send" |

**Decomposition Strategy:**

```
High-Level FR: "Users can shop online"
        ↓
    Decompose into:
        ├─ User Authentication
        │   ├─ Register new account
        │   ├─ Login with credentials
        │   └─ Reset password
        │
        ├─ Product Browsing
        │   ├─ Search products by keyword
        │   ├─ Filter by category/price
        │   └─ View product details
        │
        ├─ Shopping Cart
        │   ├─ Add items to cart
        │   ├─ Update quantities
        │   └─ Remove items
        │
        └─ Checkout Process
            ├─ Enter shipping address
            ├─ Select payment method
            └─ Confirm order
```

#### Non-Functional Requirements (NFRs)

**Definition:** Non-functional requirements describe *how well the system must perform*—the quality attributes, constraints, and operational characteristics.

**Layman explanation:** These are the "quality standards" of your system. For the car example: "The car must accelerate from 0-60 mph in under 8 seconds," "The car must get at least 30 mpg," "The car must be safe in crashes."

**Common NFR Categories:**

| Category | Description | Example Metrics |
|----------|-------------|-----------------|
| **Performance** | Speed and responsiveness | Response time < 200ms, Throughput > 10,000 TPS |
| **Scalability** | Ability to handle growth | Support 10M concurrent users, Scale to 100TB data |
| **Availability** | System uptime | 99.99% uptime (52 minutes downtime/year) |
| **Reliability** | Consistency of correct operation | Mean Time Between Failures (MTBF) > 720 hours |
| **Security** | Protection from threats | Encryption at rest/transit, OAuth 2.0 authentication |
| **Maintainability** | Ease of updates and fixes | Code coverage > 80%, Deployment time < 15 minutes |
| **Usability** | User experience quality | Task completion in < 3 clicks, Mobile responsive |
| **Compatibility** | Integration with other systems | Support Chrome/Firefox/Safari, REST API compliance |

**Decomposition Strategy:**

```
High-Level NFR: "System must be highly available"
        ↓
    Decompose into measurable targets:
        ├─ Availability Target: 99.95% uptime
        │   └─ Translates to: ~4.38 hours downtime per year
        │
        ├─ Redundancy Requirements
        │   ├─ Multi-region deployment (3+ regions)
        │   ├─ Active-active configuration
        │   └─ Automated failover < 30 seconds
        │
        ├─ Monitoring Requirements
        │   ├─ Health checks every 10 seconds
        │   ├─ Alerting within 1 minute of issue
        │   └─ 24/7 on-call rotation
        │
        └─ Recovery Requirements
            ├─ Recovery Time Objective (RTO): 15 minutes
            ├─ Recovery Point Objective (RPO): 5 minutes
            └─ Backup frequency: Every 6 hours
```

**Key Differences Summary:**

```
Functional Requirements          Non-Functional Requirements
        ↓                                   ↓
   "WHAT to do"                        "HOW WELL to do it"
        ↓                                   ↓
  Feature-focused                      Quality-focused
        ↓                                   ↓
 "Add to cart"                       "Cart loads in <100ms"
 "Process payment"                   "Process 10K payments/sec"
 "Send notification"                 "99.9% delivery success rate"
```

---

### 1.2 Constraint Identification

Constraints are the boundaries within which your system must be designed and built. Identifying them early prevents unrealistic designs and costly pivots later.

#### Types of Constraints

**1. Budget Constraints**

**Layman explanation:** How much money you can spend on building and running the system—like having a budget for building a house.

**What to identify:**
- Total development budget (personnel, tools, licenses)
- Infrastructure budget (servers, cloud costs, bandwidth)
- Ongoing operational costs (maintenance, support, updates)
- Hidden costs (training, migration, technical debt repayment)

**Impact on design:**
```
Limited Budget Scenario:
    ├─ Cannot afford: Multi-region deployment, Premium databases
    ├─ Must choose: Open-source tools, Serverless architectures
    ├─ Trade-off: Accept higher latency for cost savings
    └─ Mitigation: Start simple, scale incrementally
```

**2. Timeline Constraints**

**Layman explanation:** The deadline by which the system must be delivered—like a construction project that must finish before winter.

**What to identify:**
- Hard deadlines (regulatory compliance, market windows)
- Milestone dates (beta launch, investor demos)
- Dependencies on other projects or teams
- Seasonal factors (e.g., must be ready before holiday shopping season)

**Impact on design:**
```
Tight Timeline Scenario:
    ├─ Cannot do: Build custom solutions from scratch
    ├─ Must choose: Leverage existing libraries/frameworks, Buy vs Build
    ├─ Trade-off: Accept technical debt for speed
    └─ Mitigation: Prioritize MVP features, defer nice-to-haves
```

**3. Team Skills Constraints**

**Layman explanation:** What your team knows how to do well—like a construction crew that specializes in wood framing but not steelwork.

**What to identify:**
- Programming languages/frameworks team knows
- Cloud platforms team has experience with
- Database technologies team can manage
- Learning curve for new technologies
- Availability of training or external expertise

**Impact on design:**
```
Team with Python/PostgreSQL expertise:
    ├─ Good fit: Django/Flask backend, PostgreSQL database
    ├─ Poor fit: Java Spring ecosystem, Oracle database
    ├─ Risk area: New tech like Rust or Kubernetes
    └─ Strategy: Build on strengths, minimize new learning
```

**4. Compliance Constraints**

**Layman explanation:** Legal rules and industry standards you must follow—like building codes that specify how strong a foundation must be.

**What to identify:**
- Data privacy regulations (GDPR, CCPA, HIPAA)
- Industry standards (PCI-DSS for payments, SOC 2 for SaaS)
- Geographic restrictions (data residency requirements)
- Audit and reporting requirements
- Accessibility standards (WCAG, ADA)

**Impact on design:**
```
GDPR Compliance Requirements:
    ├─ Data storage: EU region only for EU users
    ├─ Features required: User data export, Right to be forgotten
    ├─ Logging: Audit trail for all data access
    ├─ Security: Encryption, Access controls
    └─ Architecture impact: Multi-region deployment complexity
```

#### Constraint Documentation Template

```
CONSTRAINT ANALYSIS DOCUMENT
============================

Project: [System Name]
Date: [Date]
Owner: [Engineering Lead]

1. BUDGET CONSTRAINTS
   Development Budget: $X
   Infrastructure Budget: $Y/month
   Maximum acceptable burn rate: $Z/month
   Impact: [How this limits technology choices]

2. TIMELINE CONSTRAINTS
   Hard deadline: [Date] - [Reason]
   Milestone 1: [Date] - [Deliverable]
   Milestone 2: [Date] - [Deliverable]
   Impact: [Features to defer, MVP scope]

3. TEAM CONSTRAINTS
   Team size: X engineers
   Skill matrix: [Languages/frameworks known]
   Available for learning: [New technologies acceptable]
   Impact: [Technology stack limitations]

4. COMPLIANCE CONSTRAINTS
   Regulations: [GDPR, HIPAA, etc.]
   Standards: [ISO, SOC 2, etc.]
   Certifications required: [List]
   Impact: [Architectural requirements]

5. TECHNICAL CONSTRAINTS
   Existing systems to integrate: [List]
   Legacy technology debt: [What must be maintained]
   Approved vendors only: [Yes/No]
   Impact: [Integration complexity, vendor lock-in]
```

---

### 1.3 Stakeholder Alignment Techniques

Stakeholder alignment ensures everyone involved—executives, product managers, engineers, customers—shares the same understanding of what's being built and why.

**Layman explanation:** Getting everyone to agree on the same destination before starting the journey—like a family agreeing on vacation plans before booking flights.

#### Key Stakeholder Groups

```
Stakeholder Ecosystem:

    ┌─────────────────────────────────────────────┐
    │         EXECUTIVE LEADERSHIP                │
    │  (Budget, Strategy, Business outcomes)      │
    └────────────────┬────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
    ┌────▼─────┐          ┌─────▼────┐
    │ PRODUCT  │          │ BUSINESS │
    │ MANAGERS │          │  USERS   │
    │(Features)│          │ (Needs)  │
    └────┬─────┘          └─────┬────┘
         │                      │
         └──────────┬───────────┘
                    │
            ┌───────▼────────┐
            │   ENGINEERING  │
            │ (Technical     │
            │  feasibility)  │
            └───────┬────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
    ┌────▼─────┐        ┌─────▼─────┐
    │   OPS    │        │ SECURITY  │
    │(Ops work)│        │(Compliance)│
    └──────────┘        └───────────┘
```

#### Alignment Techniques

**1. Requirements Review Meetings**

**Purpose:** Get all stakeholders to review, question, and approve requirements together.

**Structure:**
```
Meeting Flow (90 minutes):

    [0-10 min]  Introduction & Goals
                - Why we're building this
                - Success criteria

    [10-40 min] Functional Requirements Walkthrough
                - Demo mockups or prototypes
                - Q&A on each major feature

    [40-60 min] Non-Functional Requirements Review
                - Performance targets
                - Security/compliance needs
                - Scalability expectations

    [60-80 min] Constraints & Trade-offs
                - Budget realities
                - Timeline implications
                - What we're NOT building

    [80-90 min] Sign-off & Next Steps
                - Document decisions
                - Identify open questions
                - Assign action items
```

**Best Practices:**
- Share documents 48 hours before meeting
- Use visual aids (diagrams, mockups)
- Record decisions in writing immediately
- Have a facilitator keep discussion on track

**2. Prototyping and Demos**

**Purpose:** Show, don't tell—visual prototypes communicate better than written specs.

**Layman explanation:** Like showing someone a model of a house rather than just describing it in words.

**Types:**
- Low-fidelity wireframes (Figma, Balsamiq)
- Interactive clickable prototypes
- Technical proof-of-concepts for risky features
- Video walkthroughs of user flows

**3. User Story Mapping**

**Purpose:** Organize requirements around user journeys to ensure completeness.

```
User Story Map Structure:

User Activities (High Level):
[Discover Products] → [Add to Cart] → [Checkout] → [Track Order]
       ↓                  ↓              ↓             ↓
User Tasks:
- Search          - Review cart   - Enter address  - View status
- Filter          - Update qty    - Select payment - Get updates
- View details    - Apply coupon  - Confirm order  - Contact support
       ↓                  ↓              ↓             ↓
Prioritization:
[MVP - Must Have]
[Version 1.1 - Should Have]
[Future - Nice to Have]
```

**4. RACI Matrix for Decisions**

**Purpose:** Clarify who has what role in requirement decisions.

**Layman explanation:** RACI stands for Responsible, Accountable, Consulted, Informed—defining who does what in decision-making.

```
RACI Matrix Example:

Requirement Decision | Product Mgr | Eng Lead | Security | Exec
--------------------|-------------|----------|----------|------
Feature priority    |      A      |    R     |    C     |  I
Performance target  |      C      |    A     |    I     |  I
Security standards  |      I      |    C     |    A     |  I
Budget approval     |      C      |    C     |    I     |  A

R = Responsible (does the work)
A = Accountable (final decision maker)
C = Consulted (provides input)
I = Informed (kept in the loop)
```

**5. Decision Logs**

**Purpose:** Document what was decided, why, and by whom—prevents revisiting settled issues.

**Template:**
```
DECISION LOG ENTRY #42
======================
Date: 2026-01-15
Decision: Use PostgreSQL instead of MongoDB for primary database
Made by: Engineering Lead (Sarah Chen)
Stakeholders involved: Product (Alex), Ops (Jordan), Security (Priya)

CONTEXT:
Need to choose database for user data storage

OPTIONS CONSIDERED:
1. PostgreSQL (relational)
2. MongoDB (document)
3. DynamoDB (managed NoSQL)

DECISION:
PostgreSQL selected

RATIONALE:
- Team has 5 years PostgreSQL experience (low risk)
- ACID transactions needed for payment processing
- Structured schema fits our data model
- Cost: $500/month vs $1200/month for DynamoDB

TRADE-OFFS ACCEPTED:
- Slightly more complex to scale horizontally than MongoDB
- Need to manage schema migrations

IMPACT:
- Timeline: No delay (team expertise)
- Budget: Within allocated database budget
- Risk: Low, team confident

NEXT STEPS:
- Set up dev/staging PostgreSQL instances by 2026-01-20
- Create initial schema design by 2026-01-22
```

---

### 1.4 Writing Technical Requirements Documents (TRDs)

A Technical Requirements Document translates business needs into engineering specifications. It's the contract between what stakeholders want and what engineers will build.

**Layman explanation:** The TRD is like an architect's detailed blueprint—it shows exactly what to build, with measurements and specifications, so builders know what to construct.

#### TRD Structure

```
TECHNICAL REQUIREMENTS DOCUMENT
================================

1. EXECUTIVE SUMMARY
   - Problem statement (1 paragraph)
   - Proposed solution (1 paragraph)
   - Success metrics (3-5 key metrics)

2. SYSTEM OVERVIEW
   - High-level architecture diagram
   - Key components and their roles
   - Integration points

3. FUNCTIONAL REQUIREMENTS
   - Organized by feature/module
   - Each requirement with unique ID
   - Clear acceptance criteria

4. NON-FUNCTIONAL REQUIREMENTS
   - Performance targets (with metrics)
   - Scalability expectations
   - Security requirements
   - Availability/reliability targets

5. CONSTRAINTS
   - Budget constraints
   - Timeline constraints
   - Technology constraints
   - Compliance requirements

6. ASSUMPTIONS AND DEPENDENCIES
   - What we're assuming to be true
   - External systems we depend on
   - Third-party services required

7. OUT OF SCOPE
   - What we're explicitly NOT building
   - Future enhancements deferred

8. RISKS AND MITIGATIONS
   - Technical risks identified
   - Mitigation strategies

9. ACCEPTANCE CRITERIA
   - How we'll know we're done
   - Testing approach

10. APPENDICES
    - Glossary of terms
    - Reference materials
```

#### TRD Best Practices

**1. Use Unique, Traceable IDs**

```
FR-001: User Authentication
FR-002: Password Reset
FR-003: Shopping Cart Management
  FR-003.1: Add item to cart
  FR-003.2: Update item quantity
  FR-003.3: Remove item from cart

NFR-001: API response time < 200ms at 95th percentile
NFR-002: System uptime 99.9% measured monthly
```

**2. Make Requirements Testable**

**Bad (vague):** System should be fast
**Good (testable):** API endpoints must respond in <200ms for 95% of requests under 1000 RPS load

**Bad (subjective):** UI should be user-friendly
**Good (testable):** New users must complete account creation in <3 minutes with <5% abandonment rate

**3. Use Clear, Unambiguous Language**

**Avoid:** should, could, might, approximately, as appropriate
**Use:** must, shall, will, exactly, specific numbers

**Example:**
- ❌ "The system should probably handle around 1000 users"
- ✅ "The system must support 1000 concurrent users with <200ms response time"

**4. Include Visual Diagrams**

```
Example: User Authentication Flow

    ┌──────┐                ┌─────────┐              ┌──────────┐
    │ User │                │   API   │              │   Auth   │
    │      │                │ Gateway │              │ Service  │
    └───┬──┘                └────┬────┘              └────┬─────┘
        │                        │                        │
        │ POST /login            │                        │
        │ {email, password}      │                        │
        │───────────────────────>│                        │
        │                        │                        │
        │                        │ Validate credentials   │
        │                        │───────────────────────>│
        │                        │                        │
        │                        │                        │ Verify
        │                        │                        │ password
        │                        │                        │ hash
        │                        │                        │
        │                        │     JWT Token          │
        │                        │<───────────────────────│
        │                        │                        │
        │    200 OK              │                        │
        │    {token, userId}     │                        │
        │<───────────────────────│                        │
        │                        │                        │
```

---

### 1.5 Identifying Hidden Requirements and Edge Cases

Hidden requirements are the needs that stakeholders don't explicitly state but expect to work. Edge cases are unusual scenarios that can break your system if not handled.

**Layman explanation:** Hidden requirements are like expecting a car to have a spare tire—nobody mentions it explicitly, but everyone assumes it's there. Edge cases are like "what happens if someone puts diesel in a gasoline car?"

#### Common Hidden Requirements

**1. Data Migration**

When building a new system to replace an old one:
- Migrating existing user data
- Maintaining historical records
- Dual-running old and new systems during transition
- Rollback capability if migration fails

**2. Audit and Logging**

Often unstated but critical:
- Who did what, when (audit trail)
- Error logging for debugging
- Compliance logging (GDPR, HIPAA)
- Log retention policies

**3. User Access Management**

Beyond basic login:
- Role-based access control (RBAC)
- Password reset flows
- Account recovery mechanisms
- Session timeout policies
- Multi-factor authentication

**4. Error Handling and Graceful Degradation**

```
System Failure Scenarios:

Database Down:
    └─> Can system still serve cached data?
    └─> What functionality remains available?

Third-party API Down:
    └─> Retry logic with exponential backoff?
    └─> Fallback to alternative provider?
    └─> Queue requests for later processing?

High Traffic Spike:
    └─> Rate limiting to protect system?
    └─> Graceful degradation (disable non-critical features)?
    └─> User-friendly error messages?
```

#### Discovering Hidden Requirements

**Technique 1: "What if" Questioning**

```
For a payment system:

What if... the payment gateway is down?
    → Need: Retry logic, fallback processor, queue transactions

What if... user submits payment twice?
    → Need: Idempotency keys, duplicate detection

What if... payment succeeds but confirmation fails?
    → Need: Transaction reconciliation, manual review queue

What if... user disputes a charge?
    → Need: Dispute tracking, audit logs, refund workflow
```

**Technique 2: Walk Through User Journeys End-to-End**

```
E-commerce Purchase Journey:

1. Browse products
   └─> What if no products found? (Empty state handling)

2. Add to cart
   └─> What if item out of stock? (Real-time inventory check)

3. Checkout
   └─> What if address incomplete? (Validation, autocomplete)

4. Payment
   └─> What if card declined? (Error messages, alternative methods)

5. Confirmation
   └─> What if email fails to send? (Retry, alternative notification)

6. Fulfillment
   └─> What if shipment delayed? (Status updates, customer service)
```

**Technique 3: Study Existing System Failures**

Look at:
- Customer support tickets (what breaks?)
- System outage reports (what caused downtime?)
- User complaints (what frustrates users?)
- Production incident logs (what edge cases hit production?)

#### Edge Case Categories

**1. Boundary Conditions**

```
Input Validation Edge Cases:

Empty input:          "" (empty string)
Null values:          null, undefined
Zero values:          0, 0.00
Negative values:      -1, -999
Maximum values:       Integer.MAX_VALUE
Minimum values:       Integer.MIN_VALUE
Special characters:   <script>, '; DROP TABLE, Unicode characters
```

**2. Timing and Concurrency**

```
Concurrency Edge Cases:

Race conditions:
    User A and User B buy last item simultaneously
    └─> Need: Database-level locking or inventory reservation

Simultaneous updates:
    Two users edit same document at same time
    └─> Need: Conflict resolution, operational transforms

Time zones:
    Meeting scheduled across time zones
    └─> Need: UTC storage, timezone conversion, DST handling

Clock skew:
    Distributed systems with unsynchronized clocks
    └─> Need: Vector clocks, NTP synchronization
```

**3. Scale Edge Cases**

```
Scale Scenarios:

Volume:
    └─> What if 1 million users sign up in one day?
    └─> What if database grows to 100TB?

Velocity:
    └─> What if 100,000 requests hit in one second?
    └─> What if batch job processes 1 billion records?

Variety:
    └─> What if user uploads 10GB video file?
    └─> What if username contains emoji or special scripts?
```

**4. External Dependency Failures**

```
Dependency Failure Matrix:

┌────────────────┬──────────────────┬───────────────────┐
│   Dependency   │  Failure Mode    │  System Response  │
├────────────────┼──────────────────┼───────────────────┤
│ Payment API    │ Timeout (30s)    │ Retry 3x, queue   │
│ Email Service  │ Rate limit hit   │ Queue, backoff    │
│ Image CDN      │ 5xx errors       │ Fallback to S3    │
│ Auth Service   │ Total outage     │ Cached tokens OK  │
│ Database       │ Primary down     │ Failover replica  │
└────────────────┴──────────────────┴───────────────────┘
```

#### Edge Case Documentation Template

```
EDGE CASE ANALYSIS
==================

Scenario: User submits payment while internet disconnects

Trigger: Network interruption during payment processing

Current behavior (if any): Unknown / Not handled

Expected behavior:
- Transaction should complete if backend received request
- User should see "Processing..." state, not error
- Auto-retry on reconnection
- Show clear status once connection restored

Priority: HIGH (impacts revenue, user trust)

Test cases:
1. Disconnect after clicking "Pay" but before request sent
2. Disconnect after request sent, before response received
3. Disconnect after successful payment, before confirmation shown

Implementation notes:
- Use idempotency keys for payment requests
- Client-side retry with exponential backoff
- Server-side deduplication of payment requests
- Webhook for async payment confirmation
```

---

## 2. Back-of-Envelope Calculations

Back-of-envelope calculations are quick, approximate estimates used to validate whether a system design is feasible. They help you make informed decisions about architecture, infrastructure, and costs before writing a single line of code.

**Layman explanation:** These are "napkin math" calculations—rough estimates done quickly to check if your plan makes sense. Like estimating if you have enough money for a vacation by quickly adding up flight, hotel, and food costs.

**Why they matter:**
- Catch unrealistic designs early
- Estimate infrastructure costs
- Size databases, caches, and servers appropriately
- Identify bottlenecks before building
- Make build vs. buy decisions with data

### Core Principles

1. **Approximate, don't be exact:** Round numbers to make math easier (1 million ≈ 10^6, not 1,048,576)
2. **Show your work:** Document assumptions so others can adjust them
3. **Use standard units:** Stick to industry conventions (QPS, GB, ms)
4. **Sanity check:** Does the answer make sense in the real world?

### Common Reference Numbers (2026 Standards)

```
STORAGE:
    1 KB  = 1,000 bytes        ≈ Short email
    1 MB  = 1,000 KB           ≈ High-quality photo
    1 GB  = 1,000 MB           ≈ HD movie
    1 TB  = 1,000 GB           ≈ 250,000 photos
    1 PB  = 1,000 TB           ≈ 500 million photos

TIME:
    1 second     = 1,000 milliseconds (ms)
    1 ms         = 1,000 microseconds (μs)
    1 day        = 86,400 seconds
    1 month      ≈ 2.5 million seconds (30 days)
    1 year       ≈ 31.5 million seconds

NETWORK:
    Typical bandwidth:
        - Mobile 4G:        10-50 Mbps
        - Home broadband:   100-1000 Mbps
        - Data center:      1-100 Gbps

LATENCY:
    L1 cache:               0.5 ns
    L2 cache:               7 ns
    RAM access:             100 ns
    SSD read:               150 μs
    HDD seek:               10 ms
    Network within DC:      0.5 ms
    Network cross-region:   50-150 ms
    Network cross-continent: 100-300 ms
```

---

### 2.1 QPS/RPS Estimation from DAU/MAU

**QPS (Queries Per Second)** and **RPS (Requests Per Second)** measure how many operations your system must handle every second.

**DAU (Daily Active Users)** and **MAU (Monthly Active Users)** measure how many unique users interact with your system.

**Layman explanation:** If you run a coffee shop, DAU is how many different customers visit each day, and QPS is how many coffee orders you get per minute during rush hour. You need to know both to staff appropriately.

#### Basic Calculation Framework

```
Step 1: Estimate total daily requests
    Daily Requests = DAU × Average actions per user per day

Step 2: Account for traffic distribution
    Peak QPS = (Daily Requests / 86,400 seconds) × Peak factor

    Where Peak factor typically = 2-5×
        - Social media during events: 5-10×
        - E-commerce during sales: 3-8×
        - Normal business apps: 2-3×

Step 3: Add safety margin
    Provisioned Capacity = Peak QPS × Safety margin (1.5-2×)
```

#### Worked Example 1: Social Media App

**Given:**
- DAU: 10 million users
- Each user: 50 actions per day (scroll, like, comment, post)

**Calculation:**

```
Step 1: Total daily requests
    = 10,000,000 users × 50 actions/user
    = 500,000,000 requests/day

Step 2: Average QPS
    = 500,000,000 / 86,400 seconds
    ≈ 5,787 QPS average

Step 3: Peak QPS (peak factor = 5× for social media)
    = 5,787 × 5
    ≈ 29,000 QPS peak

Step 4: Provisioned capacity (2× safety margin)
    = 29,000 × 2
    = 58,000 QPS capacity needed
```

**Interpretation:**
- Need to handle ~58,000 QPS at peak
- Each web server handles ~1,000 QPS (typical)
- Need at least 58 web servers during peak
- With load balancing and redundancy: 70-80 servers

#### Worked Example 2: E-commerce Site

**Given:**
- DAU: 500,000 daily shoppers
- Each user: 20 page views per session
- 5% checkout (25,000 transactions/day)

**Calculation:**

```
Browse Requests:
    = 500,000 × 20 page views
    = 10,000,000 page views/day
    Average: 10M / 86,400 ≈ 116 QPS
    Peak (3× for shopping): 116 × 3 = 348 QPS

Checkout Requests:
    = 25,000 checkouts × 10 API calls per checkout
    = 250,000 checkout API calls/day
    Average: 250,000 / 86,400 ≈ 3 QPS
    Peak (8× during flash sales): 3 × 8 = 24 QPS

Total Peak QPS: 348 + 24 = 372 QPS
Provisioned capacity (1.5× margin): 372 × 1.5 ≈ 560 QPS
```

#### Converting Between DAU and MAU

```
Typical Relationship:
    MAU = DAU × 3 to 4

    Example:
        If DAU = 1 million
        Then MAU ≈ 3-4 million

    Meaning: Some users visit daily, others weekly/monthly
```

**Stickiness Ratio:**
```
Stickiness = DAU / MAU

High stickiness (daily habit):  0.3-0.6  (e.g., messaging apps)
Medium stickiness:              0.15-0.3  (e.g., social media)
Low stickiness:                 0.05-0.15 (e.g., utility apps)
```

#### Request Type Breakdown

Different actions have different costs:

```
Action Type Complexity:

READ (Low cost):
    - View profile:         1 DB read
    - Scroll feed:          10-50 DB reads
    - Search:               1 search query + 10 DB reads

WRITE (Medium cost):
    - Like/upvote:          1 DB write + cache invalidation
    - Comment:              3-5 DB writes (content, index, notifications)
    - Post photo:           Image upload + CDN + 5-10 DB writes

HEAVY (High cost):
    - Video upload:         Encoding, transcoding, CDN distribution
    - AI processing:        GPU compute, model inference
    - Report generation:    Aggregate queries across millions of rows
```

**Weighted QPS Calculation:**

```
Example: Video Platform

Assume DAU = 5 million
Actions per user per day:
    - 50 video views (read-heavy)
    - 5 searches
    - 2 uploads

Weighted calculation:
    Views:    5M × 50 = 250M lightweight reads → 2,894 QPS avg
    Searches: 5M × 5  = 25M medium reads       → 289 QPS avg
    Uploads:  5M × 2  = 10M heavy writes       → 116 QPS avg (but CPU-intensive)

Total: ~3,300 QPS average
Peak (4×): ~13,200 QPS
Capacity: ~20,000 QPS provisioned
```

---

### 2.2 Storage Growth Modeling

Storage growth modeling predicts how much disk space your system will need over time.

**Layman explanation:** Like estimating how many file cabinets you'll need over the next 5 years based on how many documents you file each month.

#### Basic Storage Formula

```
Total Storage = (Items × Size per item × Replication factor) + Overhead

Where:
    Items = number of records/files
    Size per item = average bytes per record
    Replication factor = copies for redundancy (typically 3×)
    Overhead = indexes, metadata, fragmentation (typically 1.2-1.5×)
```

#### Worked Example 1: Photo Sharing App

**Given:**
- 1 million active users
- Each user uploads 5 photos/month
- Average photo size: 2 MB
- Keep all photos forever
- 3× replication for reliability

**Calculation:**

```
YEAR 1:
    Photos uploaded = 1M users × 5 photos/month × 12 months
                   = 60 million photos

    Raw storage    = 60M × 2 MB = 120,000 MB = 120 TB

    With replication (3×):
                   = 120 TB × 3 = 360 TB

    With overhead (1.3× for metadata, thumbnails):
                   = 360 TB × 1.3 = 468 TB needed in Year 1

YEAR 2 (cumulative):
    New photos     = 60M (same upload rate)
    Total photos   = 120M
    Storage needed = 120M × 2 MB × 3 × 1.3 = 936 TB

YEAR 3:
    Storage needed = 180M × 2 MB × 3 × 1.3 = 1,404 TB ≈ 1.4 PB

GROWTH RATE:
    ~470 TB per year
    ~39 TB per month
    ~1.3 TB per day
```

**Infrastructure Planning:**

```
Cloud Storage Costs (2026 estimate):
    S3 Standard: $0.023 per GB/month

    Year 1: 468 TB = 468,000 GB
    Cost: 468,000 × $0.023 = $10,764/month

    Year 3: 1.4 PB = 1,400,000 GB
    Cost: 1,400,000 × $0.023 = $32,200/month
```

#### Worked Example 2: Chat Application

**Given:**
- 10 million DAU
- Each user sends 50 messages/day
- Average message: 200 bytes (text only)
- Message retention: 1 year
- Database replication: 3×

**Calculation:**

```
DAILY MESSAGES:
    = 10M users × 50 messages = 500 million messages/day

DAILY STORAGE GROWTH:
    Raw data    = 500M × 200 bytes = 100 GB/day

    With replication (3×):
                = 100 GB × 3 = 300 GB/day

    With indexes (1.5× for message search, user indexes):
                = 300 GB × 1.5 = 450 GB/day

YEARLY PROJECTION:
    = 450 GB/day × 365 days = 164,250 GB ≈ 164 TB/year

RETENTION POLICY IMPACT:
    With 1-year retention: 164 TB max (steady state)
    Without deletion: 164 TB × number of years
```

---

### 2.3 Bandwidth Calculations

Bandwidth measures how much data flows through your network connections. You need to estimate both sustained (average) and peak traffic.

**Layman explanation:** Bandwidth is like the size of pipes carrying water. You need to know both the normal flow and the maximum flow during a flood to size your pipes correctly.

#### Basic Bandwidth Formula

```
Bandwidth (Mbps) = (Data transferred per second in MB) × 8

Convert to standard units:
    - 1 Mbps = 1,000,000 bits per second
    - 1 Gbps = 1,000 Mbps
    - 1 MB = 8 Megabits (Mb)
```

#### Worked Example 1: Video Streaming Service

**Given:**
- 100,000 concurrent viewers during peak
- Video quality: 5 Mbps per stream (1080p)

**Calculation:**

```
PEAK BANDWIDTH REQUIRED:
    = 100,000 streams × 5 Mbps per stream
    = 500,000 Mbps
    = 500 Gbps peak

SUSTAINED (assuming 40% average load):
    = 500 Gbps × 0.4
    = 200 Gbps sustained

MONTHLY DATA TRANSFER:
    Average viewers per hour: 40,000
    Hours per day: 24
    Days per month: 30

    Data per stream per hour:
        = 5 Mbps × 3,600 seconds
        = 18,000 Megabits = 2,250 MB ≈ 2.25 GB

    Monthly transfer:
        = 40,000 viewers × 24 hours × 30 days × 2.25 GB
        = 64,800,000 GB
        = 64.8 PB/month
```

**CDN Cost Estimate:**
```
CDN pricing (2026): ~$0.02 per GB
    64.8 PB = 64,800,000 GB
    Cost = 64,800,000 × $0.02 = $1,296,000/month
```

#### Worked Example 2: Image-Heavy Social App

**Given:**
- 5 million DAU
- Each user views 100 images/day
- Average image size: 500 KB (compressed, optimized)

**Calculation:**

```
DAILY BANDWIDTH:
    Images viewed = 5M users × 100 images = 500M images/day

    Data transferred = 500M × 500 KB = 250,000,000 KB
                     = 250,000 GB = 250 TB/day

AVERAGE BANDWIDTH:
    = 250 TB/day ÷ 86,400 seconds
    = 2.89 TB/second
    = 2,890 GB/second
    = 23,120 Gbps average

PEAK BANDWIDTH (3× average for social media):
    = 23,120 × 3 = 69,360 Gbps peak
    ≈ 70 Tbps (Terabits per second)
```

**Note:** This massive bandwidth is why CDNs are essential—they cache content at edge locations close to users.

#### Ingress vs Egress Bandwidth

```
INGRESS (Upload to your system):
    - User uploads
    - API requests
    - Typically much smaller than egress

EGRESS (Download from your system):
    - Media delivery (images, videos)
    - API responses
    - Typically dominates bandwidth costs

Example: Photo upload app
    Ingress:  Users upload 1M photos/day × 2 MB = 2 TB/day ingress
    Egress:   Users view 100M photos/day × 2 MB = 200 TB/day egress

    Ratio: 1:100 (egress is 100× larger)
```

---

### 2.4 Memory Requirements for Caching Layers

Caching stores frequently accessed data in fast memory (RAM) to reduce database load and improve response times.

**Layman explanation:** Caching is like keeping your most-used kitchen tools on the counter instead of in a drawer—faster to grab when you need them.

#### Cache Sizing Strategy

```
Step 1: Identify what to cache
    - Hot data (frequently accessed)
    - Expensive computations
    - Slow database queries

Step 2: Estimate working set size
    Working set = Data accessed in recent time window (e.g., last hour)

Step 3: Apply cache hit ratio target
    Typical targets: 80-95% cache hit ratio

Step 4: Add overhead
    Redis/Memcached overhead: ~20-30% for metadata
```

#### Worked Example 1: E-commerce Product Catalog

**Given:**
- 10 million products in database
- Each product record: 5 KB (JSON with details)
- 20% of products account for 80% of views (Pareto principle)
- Target: 90% cache hit ratio

**Calculation:**

```
FULL CATALOG SIZE:
    = 10M products × 5 KB = 50,000,000 KB = 50 GB

HOT PRODUCTS (20% most viewed):
    = 2M products × 5 KB = 10 GB

CACHE SIZE FOR 90% HIT RATIO:
    Need to cache ~25% of products
    = 2.5M products × 5 KB = 12.5 GB

WITH OVERHEAD (1.25×):
    = 12.5 GB × 1.25 = 15.6 GB

PROVISIONED CACHE MEMORY:
    = 16 GB (round up for safety)

INFRASTRUCTURE:
    Redis server with 16 GB RAM
    Or: Distributed cache across 4 servers × 4 GB each
```

#### Worked Example 2: User Session Cache

**Given:**
- 500,000 concurrent users at peak
- Each session: 10 KB (user data, preferences, cart)
- Session timeout: 30 minutes

**Calculation:**

```
PEAK CONCURRENT SESSIONS:
    = 500,000 users × 10 KB = 5,000,000 KB = 5 GB

WITH 30-MINUTE WINDOW:
    Users in last 30 min ≈ Peak concurrent × 1.5
    = 500,000 × 1.5 = 750,000 sessions
    = 750,000 × 10 KB = 7.5 GB

WITH OVERHEAD (1.3×):
    = 7.5 GB × 1.3 = 9.75 GB

PROVISIONED CACHE:
    = 12 GB Redis cluster
    OR: 16 GB for growth headroom
```

#### Cache Eviction Policies

```
Common Strategies:

LRU (Least Recently Used):
    └─> Evict items not accessed for longest time
    └─> Good for: General-purpose caching

LFU (Least Frequently Used):
    └─> Evict items accessed least often
    └─> Good for: Long-lived popular content

TTL (Time To Live):
    └─> Evict after fixed expiration time
    └─> Good for: Time-sensitive data (stock prices, session data)

Size-based:
    └─> Evict when cache reaches capacity limit
    └─> Good for: Preventing memory overflow
```

---

### 2.5 Cost Modeling per 1M Requests

Understanding cost per million requests helps you predict operational expenses and make build vs. buy decisions.

**Layman explanation:** Like calculating the cost per mile to drive your car—helps you decide if driving or taking the train is cheaper for a trip.

#### Cost Components

```
Per-Request Costs:
    ├─ Compute (CPU time)
    ├─ Memory allocation
    ├─ Database queries
    ├─ External API calls
    ├─ Data transfer (bandwidth)
    └─ Storage operations
```

#### Worked Example 1: Simple REST API

**Given:**
- Running on AWS Lambda
- Average execution: 200ms
- Memory allocated: 512 MB
- 1 database read per request
- 5 KB response payload

**AWS Pricing (2026 estimates):**
```
Lambda:
    - $0.20 per 1M requests
    - $0.0000166667 per GB-second of compute

RDS (PostgreSQL):
    - db.t3.medium: $0.068/hour = ~$50/month
    - Can handle ~5,000 QPS

Data transfer:
    - First 10 TB/month: $0.09/GB egress
```

**Calculation:**

```
FOR 1 MILLION REQUESTS:

Lambda invocation cost:
    = $0.20 per 1M requests = $0.20

Lambda compute cost:
    Compute time = 200ms × 1M requests = 200,000 seconds
    GB-seconds = 0.5 GB × 200,000 = 100,000 GB-seconds
    Cost = 100,000 × $0.0000166667 = $1.67

Database cost (amortized):
    Assuming Lambda handles all 1M requests in 1 hour at 278 RPS
    DB can handle this load easily
    Hourly cost = $0.068

Data transfer:
    = 1M requests × 5 KB = 5,000,000 KB = 5 GB
    = 5 × $0.09 = $0.45

TOTAL COST PER 1M REQUESTS:
    = $0.20 + $1.67 + $0.068 + $0.45
    ≈ $2.39 per million requests
```

#### Worked Example 2: Video Transcoding Service

**Given:**
- GPU instances for transcoding: g4dn.xlarge
- Average video: 10 minutes, 500 MB
- Transcoding time: 2 minutes on GPU
- Output formats: 3 (1080p, 720p, 480p)

**AWS Pricing:**
```
g4dn.xlarge: $0.526/hour
S3 storage: $0.023/GB/month
```

**Calculation:**

```
FOR 1 MILLION VIDEOS:

Compute cost:
    Transcode time per video = 2 min × 3 formats = 6 min
    Total GPU hours = 1M videos × 6 min ÷ 60 = 100,000 hours
    Cost = 100,000 × $0.526 = $52,600

Storage cost (per month):
    Input: 1M × 500 MB = 500 TB
    Output: 1M × 500 MB × 3 formats = 1,500 TB
    Total: 2,000 TB = 2,000,000 GB
    Cost = 2,000,000 × $0.023 = $46,000/month

Data transfer (delivery):
    Assuming each video viewed 10 times on average
    = 1M videos × 3 formats × 500 MB × 10 views
    = 15,000 TB = 15 PB
    CDN cost ≈ $0.02/GB = 15M GB × $0.02 = $300,000

TOTAL COST FOR 1M VIDEOS:
    One-time: $52,600 (transcoding)
    Monthly recurring: $46,000 (storage) + $300,000 (CDN)

PER VIDEO COST:
    Transcoding: $0.053 per video
    Monthly storage: $0.046 per video per month
    Delivery (if viewed 10× avg): $0.30 per video
```

---

### 2.6 Network Throughput Calculations

Network throughput measures the actual data transfer rate achieved, accounting for protocol overhead, packet loss, and network congestion.

**Layman explanation:** Throughput is like the actual water flowing through a pipe, which is always less than the pipe's theoretical maximum due to friction and bends.

#### Theoretical vs. Actual Throughput

```
Typical Throughput Efficiency:

1 Gbps network link:
    Theoretical maximum: 1,000 Mbps
    TCP/IP overhead: ~5%
    Actual application throughput: ~950 Mbps (95%)

Real-world factors:
    - Protocol headers (TCP/IP, HTTP): 5-10% overhead
    - Network congestion: 10-30% reduction during peak
    - Packet retransmission (1-3% packet loss): 5-15% impact
    - Multiple concurrent flows: Sharing reduces per-flow throughput
```

#### Worked Example 1: File Download Service

**Given:**
- 10 Gbps network link
- Serving 1 GB files to users
- Average users downloading concurrently: 50

**Calculation:**

```
EFFECTIVE THROUGHPUT:
    Theoretical: 10 Gbps = 10,000 Mbps
    With overhead (95% efficiency): 9,500 Mbps
    = 9,500 ÷ 8 = 1,187.5 MB/s

PER-USER THROUGHPUT (50 concurrent):
    = 1,187.5 MB/s ÷ 50 users
    = 23.75 MB/s per user
    = 190 Mbps per user

DOWNLOAD TIME FOR 1 GB FILE:
    = 1,000 MB ÷ 23.75 MB/s
    = 42 seconds per file

MAXIMUM CONCURRENT DOWNLOADS:
    = 1,187.5 MB/s ÷ 10 MB/s (minimum acceptable speed)
    = 118 concurrent users max

    At 50 concurrent: Comfortable margin
    At 100 concurrent: Still acceptable
    At 150 concurrent: Degraded performance
```

#### Worked Example 2: Database Replication

**Given:**
- Primary database in US-East
- Replica in EU-West
- Cross-region latency: 80ms
- Network link: 1 Gbps
- Database write rate: 50,000 transactions/day
- Average transaction size: 5 KB

**Calculation:**

```
REPLICATION THROUGHPUT NEEDED:

Daily data:
    = 50,000 transactions × 5 KB = 250,000 KB = 250 MB/day

Peak write rate (assuming 5× peak factor):
    Average: 250 MB ÷ 86,400s = 2.9 KB/s
    Peak: 2.9 KB/s × 5 = 14.5 KB/s = 0.116 Mbps

Network capacity check:
    Available: 1 Gbps = 1,000 Mbps
    Required: 0.116 Mbps
    Utilization: 0.012% (plenty of headroom)

LATENCY IMPACT:
    Network RTT: 80ms
    Async replication: No user-facing impact
    Sync replication: Every write adds 80ms latency
        → User writes take 80ms+ longer
        → Not recommended for cross-region
```

---

### 2.7 Database Storage Estimation

Accurately estimating database storage prevents costly migrations and performance issues as data grows.

**Layman explanation:** Like estimating how many books will fit in a library—you need to account for the books themselves, the shelves (indexes), and aisles (overhead).

#### Storage Formula

```
Total DB Storage = (Raw Data + Indexes + Overhead) × Replication Factor

Where:
    Raw data = Rows × Average row size
    Indexes = 20-50% of raw data (varies by index count)
    Overhead = 15-25% for fragmentation, metadata
    Replication = 2-3× for high availability
```

#### Row Size Calculation

```
Example: User table

CREATE TABLE users (
    id           BIGINT,        -- 8 bytes
    email        VARCHAR(255),  -- avg 30 bytes + 1 length byte
    password_hash CHAR(64),     -- 64 bytes
    created_at   TIMESTAMP,     -- 8 bytes
    updated_at   TIMESTAMP,     -- 8 bytes
    profile_json TEXT           -- avg 500 bytes + 4 length bytes
);

Row size = 8 + 31 + 64 + 8 + 8 + 504 = 623 bytes
```

#### Worked Example 1: Social Media User Database

**Given:**
- 50 million users
- User table row size: 623 bytes (calculated above)
- Indexes: id (primary), email (unique), created_at
- 3× replication

**Calculation:**

```
RAW DATA:
    = 50M rows × 623 bytes
    = 31,150,000,000 bytes
    ≈ 31.15 GB

INDEXES:
    Primary key (id): 50M × 8 bytes = 400 MB
    Email index: 50M × 31 bytes = 1,550 MB
    Created_at index: 50M × 8 bytes = 400 MB
    Total indexes: 2,350 MB ≈ 2.35 GB

OVERHEAD (20%):
    = (31.15 + 2.35) × 0.20
    = 6.7 GB

TOTAL SINGLE INSTANCE:
    = 31.15 + 2.35 + 6.7 = 40.2 GB

WITH REPLICATION (3×):
    = 40.2 GB × 3 = 120.6 GB

PROVISIONED STORAGE:
    = 150 GB (add 25% growth buffer)
```

#### Worked Example 2: E-commerce Order History

**Given:**
- 1 million orders per month
- Order table: 1 KB per order
- Order items table: 300 bytes per item, avg 3 items per order
- Keep 3 years of history
- 2× replication

**Calculation:**

```
ORDERS TABLE (3 years):
    Orders = 1M/month × 36 months = 36M orders
    Size = 36M × 1 KB = 36 GB

ORDER_ITEMS TABLE:
    Items = 36M orders × 3 items = 108M items
    Size = 108M × 300 bytes = 32.4 GB

RAW DATA TOTAL:
    = 36 + 32.4 = 68.4 GB

INDEXES (30% for order_id, user_id, created_at, product_id):
    = 68.4 × 0.30 = 20.5 GB

OVERHEAD (15%):
    = (68.4 + 20.5) × 0.15 = 13.3 GB

TOTAL SINGLE INSTANCE:
    = 68.4 + 20.5 + 13.3 = 102.2 GB

WITH REPLICATION (2×):
    = 102.2 × 2 = 204.4 GB

MONTHLY GROWTH:
    New orders: 1M × 1 KB = 1 GB
    New items: 3M × 300 bytes = 900 MB
    Total: ~1.9 GB raw + indexes/overhead
    ≈ 2.5 GB per month growth
    ≈ 5 GB with replication
```

**Storage Planning:**
```
Year 1: 102 GB (historical) + 60 GB (new) = 162 GB
Year 2: 162 GB + 60 GB = 222 GB
Year 3: 222 GB + 60 GB = 282 GB

Provision: 350 GB (includes buffer for spikes)
```

#### Partition Strategy Impact

```
Example: Orders table partitioned by month

WITHOUT PARTITIONING:
    - Single 100 GB table
    - Query scans entire table for recent orders
    - Slow as table grows

WITH MONTHLY PARTITIONS:
    - Each month: ~2.5 GB partition
    - Query only scans current month partition
    - Fast queries, easy to archive old partitions

Storage optimization:
    - Keep recent 12 months on SSD: 30 GB
    - Archive older data to cheaper HDD: 70 GB
    - Cost savings: 40% reduction
```

#### Database Growth Over Time

```
LINEAR GROWTH MODEL:

Year 1: 100 GB
Year 2: 200 GB
Year 3: 300 GB

Formula: Storage(year) = Initial + (Growth_rate × year)


EXPONENTIAL GROWTH MODEL:

Year 1: 100 GB (1M users)
Year 2: 200 GB (2M users, viral growth)
Year 3: 450 GB (4.5M users)

Formula: Storage(year) = Initial × Growth_factor^year


STEPPED GROWTH MODEL:

Year 1: 100 GB (steady state)
Year 2: 500 GB (new feature launch, 5× spike)
Year 3: 600 GB (back to steady growth)

Requires: Capacity planning for known launch dates
```

---

## 3. Trade-off Analysis

Trade-off analysis is the systematic evaluation of competing design choices, quantifying what you gain and what you sacrifice with each option.

**Layman explanation:** Trade-offs are like choosing between speed and fuel efficiency in a car—you can't have both maximized at the same time, so you decide which matters more for your needs.

### The Fundamental Reality of Trade-offs

```
In system design, you CANNOT optimize everything simultaneously:

    Performance  ←→  Cost
    Consistency  ←→  Availability
    Simplicity   ←→  Flexibility
    Speed        ←→  Accuracy
```

Every design decision involves choosing which qualities to prioritize and which to sacrifice or compromise.

---

### 3.1 Consistency vs Availability Trade-offs

This is the famous CAP theorem dilemma: in distributed systems with network partitions, you must choose between Consistency and Availability.

**Layman explanation:** Consistency means everyone sees the same data at the same time. Availability means the system always responds to requests. In a distributed system during a network failure, you can't guarantee both—you must pick which is more important.

#### The CAP Theorem

```
CAP Theorem States: Pick 2 of 3

    C - Consistency
    A - Availability
    P - Partition Tolerance (network failures WILL happen)

Since P is unavoidable in distributed systems:
    → Choose CP (Consistent but may be unavailable)
    OR
    → Choose AP (Available but may be inconsistent)
```

#### Consistency Levels Spectrum

```
STRONG CONSISTENCY (CP systems):
    ↓
    Linearizability
    - Reads see most recent write
    - Example: Bank account balance must be exact

    Sequential Consistency
    - All nodes see operations in same order
    - Example: Distributed locks

    ↓
EVENTUAL CONSISTENCY (AP systems):
    ↓
    Causal Consistency
    - Related operations stay ordered
    - Example: Social media comments stay in thread order

    Read-your-writes
    - Users see their own updates immediately
    - May not see others' updates right away

    Eventual Consistency
    - All nodes eventually agree (seconds/minutes later)
    - Example: DNS propagation, CDN cache updates
```

#### Quantified Trade-off Examples

**Example 1: Banking System (Choose Consistency)**

```
SCENARIO: User checks balance, then withdraws $100

STRONG CONSISTENCY (CP) Approach:

    Architecture:
        ┌─────────┐         ┌──────────┐
        │ Primary │◄────────│ Client   │
        │   DB    │         └──────────┘
        └────┬────┘
             │ Sync replication
             ▼
        ┌─────────┐
        │ Replica │ (Cannot serve reads during partition)
        └─────────┘

    Trade-offs:
        ✓ Guarantee: Balance always accurate
        ✓ No overdrafts, no double-spending
        ✗ During network partition: System unavailable (5-15 sec outage)
        ✗ Higher latency: Must wait for sync replication (50-100ms extra)
        ✗ Lower throughput: Single writer bottleneck

    Metrics:
        - Availability: 99.9% (account for partition handling)
        - Latency: 100-200ms per transaction
        - Throughput: ~5,000 TPS per database
        - Cost: Higher (need reliable network, multiple regions)

    Business justification:
        - Regulatory requirement: Financial accuracy mandatory
        - Trust: Users expect exact balances
        - Risk: Inconsistent balance could cause lawsuits
        → Consistency is NON-NEGOTIABLE

EVENTUAL CONSISTENCY (AP) Approach - NOT ACCEPTABLE:
    ✗ Could show wrong balance
    ✗ Could allow overdrafts
    ✗ Violates financial regulations
    → NEVER use for financial transactions
```

**Example 2: Social Media "Like" Counter (Choose Availability)**

```
SCENARIO: Viral post gets 10,000 likes per minute

EVENTUAL CONSISTENCY (AP) Approach:

    Architecture:
        ┌─────────┐     ┌─────────┐     ┌─────────┐
        │ Region  │◄───►│ Region  │◄───►│ Region  │
        │  US     │     │  EU     │     │  ASIA   │
        └─────────┘     └─────────┘     └─────────┘
              ↑               ↑               ↑
              └───────────────┴───────────────┘
                  Async replication (100-500ms)
                  Users can like even if regions disconnected

    Trade-offs:
        ✓ Always available: Users can always like/unlike
        ✓ Low latency: <50ms response time
        ✓ High throughput: 100,000+ likes/second
        ✗ Count temporarily inaccurate (off by 100-500 likes)
        ✗ Brief double-like possible (fixed within seconds)

    Metrics:
        - Availability: 99.99% (independent regions)
        - Latency: 30-50ms average
        - Throughput: 100,000+ operations/sec
        - Cost: Lower (no need for coordination)
        - Convergence time: 500ms-2s (how long until all regions agree)

    Business justification:
        - User impact: Seeing "1,247 likes" vs "1,251 likes" doesn't matter
        - Engagement: Users frustrated by "unavailable" button
        - Scale: Can't handle viral traffic with strong consistency
        → Availability is more important than exact count

STRONG CONSISTENCY (CP) Approach - OVERKILL:
    ✗ Added latency frustrates users (200ms+ per like)
    ✗ Single point of failure (one region down = feature down globally)
    ✗ Can't scale to viral load
    → Wrong choice for this use case
```

#### Decision Framework

```
CHOOSE CONSISTENCY (CP) when:

    ✓ Financial transactions (payments, transfers, credits)
    ✓ Inventory management (prevent double-booking)
    ✓ Access control (security can't be "eventually correct")
    ✓ Legal/compliance requirements
    ✓ User expectations of exactness

    Cost: Accept lower availability during failures

CHOOSE AVAILABILITY (AP) when:

    ✓ Social engagement (likes, follows, views)
    ✓ Analytics/metrics (pageviews, click counts)
    ✓ User-generated content feeds
    ✓ Recommendation engines
    ✓ Session data, shopping carts (can sync later)

    Cost: Accept temporary inconsistency
```

#### Hybrid Approaches

```
SPLIT BY DATA TYPE:

E-commerce system:

    CP (Strong Consistency):
        - Payment processing
        - Inventory count
        - Order placement

    AP (Eventual Consistency):
        - Product reviews
        - View counts
        - Recommendation feed
        - Session data

This allows: Critical data stays consistent, everything else stays fast
```

#### Quantifying the Trade-off

```
Availability Loss from Consistency:

Example metrics from real system:

Eventual consistency (AP):
    - Availability: 99.99% (52 min downtime/year)
    - Latency P99: 45ms
    - Cost: $10,000/month

Strong consistency (CP):
    - Availability: 99.9% (8.7 hours downtime/year)
    - Latency P99: 180ms
    - Cost: $25,000/month (coordination overhead)

Trade-off quantified:
    Lose: 8 hours of availability per year
    Lose: 135ms latency per request
    Lose: $15,000/month in extra costs
    Gain: Zero data inconsistency

Question: Is the guarantee worth the cost for THIS data?
```

---

### 3.2 Latency vs Throughput Optimization

Latency (response time) and throughput (requests per second) are often in tension—optimizing one can hurt the other.

**Layman explanation:** Latency is how fast one customer gets served; throughput is how many customers you serve per hour. A restaurant could serve each customer super fast but limit total customers, OR batch multiple orders together for higher throughput but slower individual service.

#### The Core Trade-off

```
OPTIMIZE FOR LOW LATENCY:
    Strategy: Process each request immediately, minimize queuing
    Result: Fast individual responses, but lower total throughput
    Example: Handle each request in dedicated thread

OPTIMIZE FOR HIGH THROUGHPUT:
    Strategy: Batch requests, share resources efficiently
    Result: More total work done, but higher individual latency
    Example: Batch database writes every 100ms
```

#### Latency-Throughput Curve

```
The Performance Curve:

Throughput
    ↑
    │     ┌────── Saturation point
    │    ╱       (latency explodes)
    │   ╱
    │  ╱
    │ ╱
    │╱______________ Sweet spot
    │ ╲             (balanced)
    │  ╲
    │   ╲__________ Low utilization
    │               (fast but wasteful)
    └──────────────────────────→ Latency
   Low                         High

Key insight: As you push throughput higher, latency increases
```

#### Quantified Example 1: Database Batch Writes

**Scenario:** Write user actions to database

```
APPROACH A: Individual Writes (Low Latency)

    Process: Write each action immediately to database

    Metrics:
        - Latency: 5ms per write
        - Throughput: 200 writes/second per connection
        - Database load: 200 connections × 200 writes = 40,000 writes/sec
        - User experience: Instant feedback

    Trade-offs:
        ✓ User sees confirmation immediately
        ✗ High database connection overhead
        ✗ Can only handle 40,000 writes/sec before DB overload
        ✗ Expensive: Need larger DB instance

APPROACH B: Batched Writes (High Throughput)

    Process: Accumulate writes for 100ms, then batch insert

    Metrics:
        - Latency: 50-150ms (wait for batch + write)
        - Throughput: 500,000 writes/second (10× higher)
        - Database load: 5,000 batch operations/sec (manageable)
        - User experience: Small delay before confirmation

    Trade-offs:
        ✓ Can handle 10× more writes with same hardware
        ✓ Cheaper: Smaller DB instance suffices
        ✓ Better resource utilization
        ✗ User waits 50-150ms for confirmation
        ✗ More complex error handling (what if batch fails?)
        ✗ Risk: Buffer overflow if writes spike

DECISION CRITERIA:
    Use Approach A when: User expects instant feedback (chat send, payment)
    Use Approach B when: Slight delay acceptable (analytics, logs, metrics)
```

#### Quantified Example 2: API Request Handling

```
APPROACH A: One Thread Per Request (Low Latency)

    Architecture:
        Request arrives → Spawn dedicated thread → Process → Return

    Metrics:
        - Latency P50: 20ms
        - Latency P99: 45ms
        - Max throughput: 1,000 concurrent requests
            (Limited by thread pool size, context switching overhead)
        - Memory: 1 MB per thread × 1,000 = 1 GB RAM

    Trade-offs:
        ✓ Very responsive under normal load
        ✓ Simple programming model
        ✗ Collapses under heavy load (>1,000 concurrent requests)
        ✗ Wastes CPU on context switching
        ✗ High memory footprint

APPROACH B: Async Event Loop (High Throughput)

    Architecture:
        All requests share event loop → Non-blocking I/O → Process many concurrently

    Metrics:
        - Latency P50: 35ms (+15ms queueing)
        - Latency P99: 120ms (requests wait in queue)
        - Max throughput: 50,000 concurrent requests
            (Limited by CPU, not threads)
        - Memory: 10 KB per request × 50,000 = 500 MB RAM

    Trade-offs:
        ✓ Handles 50× more concurrent requests
        ✓ Lower memory usage
        ✓ Better CPU utilization
        ✗ Higher latency due to queueing
        ✗ More complex error handling
        ✗ One slow request can block others (if not careful)

DECISION CRITERIA:
    Use Approach A when: <1,000 concurrent users, simplicity valued
    Use Approach B when: >10,000 concurrent users, scale is critical
```

#### The "Knee" of the Curve

```
Every system has a throughput "knee" where latency explodes:

Example: API Server

Throughput (RPS)  →  Latency P99
    0 - 5,000         50ms    (Comfortable, plenty of headroom)
    5,000 - 8,000     75ms    (Getting busy)
    8,000 - 9,000     120ms   (Approaching limits)
    9,000 - 9,500     250ms   (THE KNEE - latency explodes)
    9,500 - 10,000    800ms   (Saturation, requests queue up)
    >10,000           Timeout (System collapses, errors spike)

Best practice: Operate at 60-70% of knee
    For this system: Target 5,500-6,500 RPS max
    Provides: Safety margin for traffic spikes
```

---

### 3.3 Cost vs Performance Optimization

The eternal question: How much performance are you willing to pay for?

**Layman explanation:** Like choosing between economy and first-class flights—first class is faster and more comfortable, but is it worth 5× the cost?

#### Cost-Performance Curves

```
Performance
    ↑
    │         ┌──── Diminishing returns
    │        ╱      (10× cost for 20% gain)
    │       ╱
    │      ╱
    │     ╱
    │    ╱_________ Sweet spot
    │   ╱           (good perf at reasonable cost)
    │  ╱
    │ ╱____________ Budget option
    │╱              (acceptable perf, low cost)
    └──────────────────────────→ Cost
   Low                         High
```

#### Quantified Example 1: Database Tier Selection

**Scenario:** Choose database instance for 50,000 daily active users

```
OPTION A: Minimal Cost

    Instance: db.t3.medium (2 vCPU, 4 GB RAM)
    Cost: $50/month

    Performance:
        - Handles: 500 QPS comfortably
        - Latency P99: 80ms
        - Storage: 100 GB

    Limitations:
        ✗ Cannot burst beyond 1,000 QPS
        ✗ Slow for complex queries (no CPU headroom)
        ✗ No read replicas (single point of failure)

    Best for:
        - MVP, proof of concept
        - Low traffic apps (<50K users)
        - When budget is primary constraint

OPTION B: Balanced

    Instance: db.m5.xlarge (4 vCPU, 16 GB RAM) + 2 read replicas
    Cost: $280/month primary + $140/month replicas = $420/month

    Performance:
        - Handles: 5,000 QPS (10× Option A)
        - Latency P99: 30ms (2.6× faster)
        - Storage: 500 GB
        - Read scalability: 15,000 read QPS across replicas

    Benefits:
        ✓ Handles growth to 500K users
        ✓ High availability (failover to replica)
        ✓ Fast even under load

    Best for:
        - Production systems
        - 100K-1M user range
        - When reliability and performance matter

OPTION C: Maximum Performance

    Instance: db.r5.4xlarge (16 vCPU, 128 GB RAM) + 5 read replicas
    Cost: $1,850/month primary + $925/month replicas = $2,775/month

    Performance:
        - Handles: 20,000 QPS (40× Option A)
        - Latency P99: 10ms (8× faster)
        - Storage: 2 TB
        - Read scalability: 100,000 read QPS

    Benefits:
        ✓ Supports millions of users
        ✓ Sub-10ms query latency
        ✓ Massive query complexity capacity

    Trade-offs:
        ✗ 55× more expensive than Option A
        ✗ Only 2× faster than Option B for typical queries
        ✗ Most capacity unused unless at scale

    Best for:
        - Multi-million user scale
        - Latency-critical applications
        - When cost is not primary concern

DECISION MATRIX:

User Scale     Monthly Budget   Recommendation   Cost/User
---------------------------------------------------------
<100K          <$500            Option A         $0.0005
100K-500K      $500-$1,000      Option B         $0.0008
500K-2M        $1,000-$3,000    Option B→C       $0.0014
>2M            $3,000+          Option C         $0.0014
```

#### Quantified Example 2: Caching Strategy

```
OPTION A: No Cache (Lowest Cost)

    Setup: Direct database queries for everything
    Cost: $0/month additional

    Performance:
        - Database load: 10,000 QPS
        - Latency P99: 150ms (every request hits DB)
        - Database cost: $600/month (need larger instance)

    Total cost: $600/month
    User experience: Slow page loads

OPTION B: Redis Cache (Balanced)

    Setup: Cache frequent queries, 90% hit rate
    Cost: $150/month (Redis cluster)

    Performance:
        - Database load: 1,000 QPS (10× reduction)
        - Latency P99: 25ms (cached) / 150ms (cache miss)
        - Effective latency: 35ms average
        - Database cost: $200/month (smaller instance suffices)

    Total cost: $350/month
    User experience: Fast
    Savings: $250/month vs Option A (42% cheaper + way faster!)

    ROI: Cache pays for itself through DB savings + better UX

OPTION C: CDN + Redis + Database Query Cache (Maximum Performance)

    Setup: Multi-layer caching
        - CDN for static assets (images, CSS, JS)
        - Redis for API responses
        - Database query cache for complex queries

    Cost: $150 Redis + $300 CDN = $450/month

    Performance:
        - CDN hit rate: 95% (static assets never hit origin)
        - Redis hit rate: 90% (API responses)
        - Database load: 100 QPS (100× reduction)
        - Latency P99: 15ms (CDN) / 25ms (Redis) / 150ms (DB)
        - Effective latency: 20ms average
        - Database cost: $100/month (tiny instance)

    Total cost: $550/month
    User experience: Blazing fast

    When it's worth it:
        ✓ Global user base (CDN reduces cross-region latency)
        ✓ High read-to-write ratio (10:1 or higher)
        ✓ User retention depends on speed
```

#### Build vs Buy Cost Analysis

**Scenario:** Need video transcoding service

```
OPTION A: Build In-House

    Development:
        - 3 engineers × 3 months = 9 engineer-months
        - Cost: 9 × $15,000 = $135,000

    Infrastructure:
        - GPU servers: 10 × g4dn.xlarge = $3,780/month
        - Storage: 50 TB = $1,150/month
        - Total: $4,930/month

    Maintenance:
        - 0.5 engineer ongoing = $7,500/month

    Total Year 1: $135,000 + $59,160 + $90,000 = $284,160

    Benefits:
        ✓ Full control over algorithms
        ✓ Customizable to exact needs
        ✓ No per-video fees

    Drawbacks:
        ✗ High upfront cost
        ✗ Ongoing maintenance burden
        ✗ Need expertise in video encoding

OPTION B: Buy SaaS (e.g., AWS MediaConvert)

    Cost: $0.03 per minute of video transcoded

    Usage: 100,000 videos/month × 10 min avg = 1M minutes
    Monthly cost: 1M × $0.03 = $30,000/month
    Year 1 cost: $360,000

    Benefits:
        ✓ Zero development time
        ✓ Zero maintenance
        ✓ Scales automatically
        ✓ Always up-to-date encoders

    Drawbacks:
        ✗ Higher long-term cost
        ✗ Vendor lock-in
        ✗ Less customization

BREAK-EVEN ANALYSIS:

Year 1: Buy cheaper if <284K transcoded minutes ($9,500/month usage)
Year 2+: Build cheaper if >4,930/month usage (pays back dev costs)

Decision factors:
    - <10K videos/month: Always buy (build not worth it)
    - 10K-50K videos/month: Buy (let someone else solve it)
    - >100K videos/month: Build (you'll save long-term)
    - Caveat: Only build if video is core competency
```

---

### 3.4 Technical Debt Impact Analysis

Technical debt is the implied cost of future rework caused by choosing quick solutions now instead of better approaches.

**Layman explanation:** Like fixing a leaky roof with duct tape instead of replacing shingles—fast and cheap now, but will cost you more later when the whole ceiling collapses.

#### Quantifying Technical Debt

```
Technical Debt Formula:

Total Cost = Initial shortcut savings + Accumulated interest

Where "interest" =
    - Slower feature development (working around debt)
    - More bugs (fragile code)
    - Higher maintenance costs
    - Engineer frustration (turnover cost)
```

#### Worked Example: Database Schema Design

**Scenario:** E-commerce product catalog

```
QUICK & DIRTY (Technical Debt Approach):

    Schema:
        products table: id, name, details_json
        (Store everything in JSON blob)

    Time to implement: 2 days

    Initial benefits:
        ✓ Fast to build
        ✓ Flexible (just add fields to JSON)
        ✓ No migrations needed initially

    Technical debt accumulates:

        Month 1-3:
            - Works fine for MVP
            - Query performance acceptable (<10K products)

        Month 4-6:
            - Cannot efficiently query by product attributes
            - Full table scans on searches (slow)
            - Need hacky workarounds for filtering
            Time cost: +2 hours per new feature

        Month 7-12:
            - Performance degraded (100ms → 2 second queries)
            - Cannot add database indexes on JSON fields
            - Business requests complex reports (impossible)
            Time cost: +8 hours per new feature

        Year 2:
            - MUST refactor to proper schema
            - Migration time: 4 engineer-weeks
            - Downtime risk during migration
            - Rewrite query code: 2 engineer-weeks
            Total: 6 engineer-weeks = $90,000

    Total cost over 2 years:
        Initial savings: 3 days ($3,600)
        Interest paid: $90,000 + slower development
        Net loss: -$86,400

PROPER APPROACH (No Technical Debt):

    Schema:
        products: id, name, category_id, brand, price
        product_attributes: product_id, key, value
        (Normalized, indexed)

    Time to implement: 5 days

    Costs:
        Extra upfront: 3 days ($3,600)
        Ongoing: Fast queries, easy to extend
        Year 2 refactor: $0

    Total cost over 2 years:
        Initial investment: $3,600
        Interest paid: $0
        Net savings: +$86,400 vs technical debt approach
```

#### Technical Debt Decision Matrix

```
ACCEPTABLE Technical Debt:

✓ Prototyping/MVP validation
    - Don't know if product will succeed
    - Need fast market feedback
    - Can throw away entirely if fails

✓ Time-critical launches
    - Hard external deadline (conference, event)
    - Competitive advantage from being first
    - Plan to refactor immediately after launch

✓ Isolated, non-core features
    - Edge case handling
    - Admin tools (low usage)
    - Easy to replace later without system-wide impact

UNACCEPTABLE Technical Debt:

✗ Core business logic
    - Payment processing (never cut corners)
    - User authentication (security critical)
    - Data integrity (can't "fix later")

✗ Public APIs
    - Breaking changes anger users
    - Hard to deprecate once released
    - Sets long-term maintenance obligations

✗ Infrastructure/architecture
    - Difficult to change later
    - System-wide impact
    - Migration is expensive and risky
```

---

### 3.5 Complexity vs Flexibility Trade-offs

More flexibility usually means more complexity. Simple systems are rigid; flexible systems are complex.

**Layman explanation:** A Swiss Army knife is more flexible than a regular knife, but it's also more complex, harder to use, and easier to break.

#### The Complexity Spectrum

```
SIMPLE (Inflexible):
    One way to do things
    Few configuration options
    Easy to understand
    Hard to extend

    ↓

BALANCED:
    Common paths easy
    Extension points for custom needs
    Moderate learning curve

    ↓

COMPLEX (Flexible):
    Many ways to do things
    Highly configurable
    Steep learning curve
    Easy to extend (if you understand it)
```

#### Quantified Example 1: Configuration System

**Scenario:** Application configuration management

```
APPROACH A: Simple Hard-coded Config

    Implementation:
        config.js:
            export const DB_HOST = "localhost";
            export const DB_PORT = 5432;
            export const MAX_CONNECTIONS = 10;

    Complexity: 1/10 (trivial to understand)
    Flexibility: 2/10 (requires code changes for any config)

    Pros:
        ✓ Zero configuration complexity
        ✓ Cannot be misconfigured
        ✓ Easy to debug (values in code)

    Cons:
        ✗ Different values for dev/staging/prod require code changes
        ✗ Cannot change settings without redeployment
        ✗ No runtime reconfiguration

    Best for: Hobby projects, single-deployment apps

APPROACH B: Environment Variables

    Implementation:
        .env file:
            DB_HOST=localhost
            DB_PORT=5432
            MAX_CONNECTIONS=10

        Code:
            const config = {
                dbHost: process.env.DB_HOST,
                dbPort: parseInt(process.env.DB_PORT),
                maxConnections: parseInt(process.env.MAX_CONNECTIONS)
            };

    Complexity: 3/10 (simple concept, some parsing needed)
    Flexibility: 6/10 (easy per-environment config)

    Pros:
        ✓ Different configs per environment
        ✓ No code changes needed
        ✓ Industry standard (12-factor app)

    Cons:
        ✗ Cannot change without restart
        ✗ No validation (wrong values cause runtime errors)
        ✗ Secret management needed separately

    Best for: Most production applications

APPROACH C: Dynamic Config Service

    Implementation:
        Config stored in database or service (e.g., AWS AppConfig)
        Application polls for changes every 30 seconds
        Schema validation for all config values
        Rollback capability
        Audit log of all changes

    Complexity: 8/10 (infrastructure + client library)
    Flexibility: 10/10 (change anything anytime)

    Pros:
        ✓ Change config without redeployment
        ✓ Gradual rollout (canary configs)
        ✓ Instant rollback if problems
        ✓ Central management across services
        ✓ Audit trail

    Cons:
        ✗ External dependency (config service must be up)
        ✗ Complexity in client code (caching, refresh logic)
        ✗ Potential for mid-execution config changes (bugs)
        ✗ Additional infrastructure cost

    Best for: Large-scale distributed systems, feature flags

COMPLEXITY COST:

Approach A:
    Development: 1 hour
    Maintenance: 2 hours/month (manual changes)
    Bugs from misconfiguration: Low

Approach B:
    Development: 4 hours (setup + validation)
    Maintenance: 0.5 hours/month (automated)
    Bugs from misconfiguration: Medium

Approach C:
    Development: 40 hours (service setup + client library)
    Maintenance: 2 hours/month (infrastructure)
    Bugs from misconfiguration: High (complex system)

Decision:
    <10 config values: Use Approach B
    >50 config values or >10 services: Consider Approach C
    Need frequent changes without deploy: Use Approach C
```

#### Quantified Example 2: API Design

```
SIMPLE API (Inflexible):

    Endpoint: GET /users/{id}
    Response: Returns fixed user object

    {
        "id": 123,
        "name": "Alice",
        "email": "alice@example.com",
        "createdAt": "2025-01-15"
    }

    Complexity: 1/10
    Flexibility: 2/10

    Pros:
        ✓ Predictable responses
        ✓ Easy to cache
        ✓ Simple client code

    Cons:
        ✗ Always returns all fields (bandwidth waste)
        ✗ Cannot customize for different clients
        ✗ Breaking changes if schema evolves

FLEXIBLE API (Complex):

    Endpoint: POST /graphql
    Query:
        query {
            user(id: 123) {
                name
                email
                posts(limit: 10) {
                    title
                    comments { author }
                }
            }
        }

    Complexity: 8/10
    Flexibility: 10/10

    Pros:
        ✓ Clients request exactly what they need
        ✓ No over-fetching or under-fetching
        ✓ Single endpoint for all queries
        ✓ Strongly typed schema

    Cons:
        ✗ Steep learning curve (GraphQL language)
        ✗ Complex server implementation
        ✗ Caching is difficult (queries are dynamic)
        ✗ Can enable inefficient queries (N+1 problem)
        ✗ More room for client errors

COST COMPARISON:

Simple REST API:
    Server development: 2 weeks
    Client integration: 1 week per client
    Performance optimization: Easy (caching)

    Total for 3 clients: 5 weeks

GraphQL API:
    Server development: 6 weeks (schema, resolvers, optimization)
    Client integration: 2 weeks per client (learning curve)
    Performance optimization: Hard (query complexity)

    Total for 3 clients: 12 weeks

Break-even: Flexible API pays off at ~5+ diverse clients
```

---

### 3.6 Operational Overhead Considerations

Every system design choice has operational costs—the work required to keep it running in production.

**Layman explanation:** Operational overhead is like home maintenance. A simple apartment is easy to maintain; a complex smart home with many automated systems requires constant tweaking and updates.

#### Operational Overhead Components

```
OPERATIONAL OVERHEAD:
    ├─ Monitoring (are systems healthy?)
    ├─ Alerting (when things break)
    ├─ Incident response (fixing issues)
    ├─ Updates & patching (security, dependencies)
    ├─ Backup & recovery (data protection)
    ├─ Capacity planning (will we run out of resources?)
    ├─ Cost optimization (are we overspending?)
    └─ Documentation (how does this work?)
```

#### Quantified Example: Self-Managed vs Managed Services

**Scenario:** PostgreSQL database for production app

```
OPTION A: Self-Managed on EC2

    Setup:
        - Launch EC2 instance
        - Install PostgreSQL
        - Configure replication
        - Set up monitoring
        - Configure backups

    Complexity: 9/10

    Monthly operational tasks:
        - Security patching: 4 hours/month
        - Backup verification: 2 hours/month
        - Performance tuning: 4 hours/month
        - Monitoring review: 8 hours/month
        - Incident response: 10 hours/month (average)
        Total: 28 hours/month

    Cost:
        Infrastructure: $200/month (EC2 + storage)
        Operations: 28 hours × $75/hour = $2,100/month
        Total: $2,300/month

    Risks:
        - Human error in configuration
        - Missed security patches
        - Backup failures undetected
        - Data loss from operator mistakes

OPTION B: Managed Service (AWS RDS)

    Setup:
        - Configure RDS via console (1 hour)
        - Backups automatic
        - Monitoring included
        - Patching automatic

    Complexity: 3/10

    Monthly operational tasks:
        - Monitoring review: 2 hours/month
        - Scaling decisions: 1 hour/month
        - Incident response: 2 hours/month (reduced by automation)
        Total: 5 hours/month

    Cost:
        Infrastructure: $400/month (RDS premium)
        Operations: 5 hours × $75/hour = $375/month
        Total: $775/month

    Benefits:
        ✓ Automated backups with point-in-time recovery
        ✓ Automated security patches
        ✓ 99.95% uptime SLA
        ✓ Monitoring dashboards included

COMPARISON:

Self-managed:
    Cost: $2,300/month
    Engineer time: 28 hours/month (full week)
    Risk: High (human error)

Managed service:
    Cost: $775/month
    Engineer time: 5 hours/month
    Risk: Low (automated, SLA-backed)

Savings: $1,525/month + 23 engineer hours
ROI: 66% cost reduction + engineers focus on product
```

#### Hidden Operational Costs

```
Example: Microservices Architecture

VISIBLE COSTS:
    - Infrastructure: $10,000/month

HIDDEN COSTS:
    - Service discovery maintenance: 10 hours/month
    - Network debugging (service-to-service): 20 hours/month
    - Distributed tracing setup & maintenance: 8 hours/month
    - Inter-service API versioning coordination: 12 hours/month
    - Multiple deployment pipelines: 5 hours/month
    Total hidden: 55 hours/month = $4,125/month

TRUE TOTAL: $14,125/month (41% more than visible costs)

Alternative (Monolith):
    - Infrastructure: $5,000/month
    - Operational overhead: 10 hours/month = $750/month
    Total: $5,750/month

Trade-off: Microservices cost 2.5× more operationally
Worth it when: Team size >30 engineers, independent deploys needed
Not worth it when: Team size <10 engineers, low traffic
```

#### Operational Overhead Decision Framework

```
LOW OPERATIONAL OVERHEAD (Preferred for small teams):
    ✓ Managed services (RDS, Lambda, etc.)
    ✓ Serverless architectures
    ✓ Monolithic applications
    ✓ Simple architectures

    Cost: Higher infrastructure costs
    Benefit: Engineers focus on product

HIGH OPERATIONAL OVERHEAD (Only when justified):
    ✓ Self-managed infrastructure
    ✓ Custom-built solutions
    ✓ Microservices
    ✓ Complex distributed systems

    Cost: Lower infrastructure, higher labor
    Benefit: Full control, optimized performance

JUSTIFICATION CHECKLIST:
    ⬜ Team >20 engineers (can afford specialists)
    ⬜ Cost savings >$50K/year (worth the effort)
    ⬜ Unique requirements (managed services insufficient)
    ⬜ Core competitive advantage (worth investment)

    If <2 checked: Use managed services
    If 2-3 checked: Consider hybrid approach
    If 4 checked: Self-manage makes sense
```

---

## Conclusion

System Design Thinking synthesizes three critical skills:

1. **Requirements Engineering:** Understanding WHAT to build through stakeholder alignment, constraint identification, and rigorous documentation

2. **Back-of-Envelope Calculations:** Estimating WHETHER your design is feasible through QPS, storage, bandwidth, and cost modeling

3. **Trade-off Analysis:** Deciding HOW to build by quantifying competing priorities—consistency vs availability, latency vs throughput, cost vs performance

**The master system designer:**
- Documents requirements with specificity (testable, measurable, traceable)
- Estimates capacity needs before building (prevents costly over/under-provisioning)
- Quantifies every trade-off (makes decisions with data, not hunches)
- Communicates choices clearly (stakeholders understand what they're getting and giving up)

Every design decision involves trade-offs. The goal isn't perfection—it's making informed choices that align with business priorities, user needs, and resource constraints.

**Remember:** The best system design is the one that delivers the required outcomes at acceptable cost, not the one that maximizes every metric or uses the fanciest technology.