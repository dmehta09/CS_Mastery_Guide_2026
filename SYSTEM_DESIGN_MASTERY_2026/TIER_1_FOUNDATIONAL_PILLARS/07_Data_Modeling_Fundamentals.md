# Data Modeling Fundamentals: A Comprehensive Guide

## Introduction

**Data modeling** is the process of creating a structured representation of data and its relationships within a system. Think of it as creating a blueprint for how information should be organized, stored, and accessed - similar to how an architect creates blueprints for a building before construction begins.

Data modeling is crucial because:
- **Clarity**: Provides a clear understanding of what data exists and how it relates
- **Quality**: Ensures data accuracy, consistency, and integrity
- **Efficiency**: Optimizes data storage and retrieval
- **Communication**: Creates a common language between business and technical teams
- **Scalability**: Enables systems to grow and evolve gracefully

---

## What is a Data Model?

A data model defines:
1. **Entities**: The "things" or objects we store information about (e.g., customers, orders, products)
2. **Attributes**: The characteristics or properties of entities (e.g., customer name, order date)
3. **Relationships**: How entities connect to each other (e.g., a customer places an order)
4. **Constraints**: The rules that maintain data quality (e.g., an email must be unique)

**Visual representation:**
```
┌─────────────────────────────────────────────────────────┐
│                    Data Model                           │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ENTITIES          ATTRIBUTES         RELATIONSHIPS     │
│  ┌──────────┐     ┌──────────┐       ┌──────────┐     │
│  │ Customer │     │ Name     │       │ Places   │     │
│  │ Order    │     │ Date     │       │ Contains │     │
│  │ Product  │     │ Price    │       │ Belongs  │     │
│  └──────────┘     └──────────┘       └──────────┘     │
│                                                          │
│  CONSTRAINTS                                            │
│  ┌────────────────────────────────────────────┐       │
│  │ • Uniqueness                                │       │
│  │ • Referential integrity                     │       │
│  │ • Data types and formats                    │       │
│  └────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

---

## Domain-Driven Design Basics

**Domain-Driven Design (DDD)** is an approach to software development that focuses on modeling the real-world domain (the business area) as accurately as possible. It bridges the gap between business requirements and technical implementation.

**Core principle**: The model should reflect how domain experts (business people) think about and talk about their work.

---

### Ubiquitous Language

**What it is**: A common vocabulary shared by everyone involved in the project - developers, business analysts, domain experts, and stakeholders. Every term has a precise, agreed-upon meaning.

**The problem without ubiquitous language:**
```
Business Person:  "When a customer reserves an item..."
Developer:        "Oh, you mean when they create a pending order?"
Business Person:  "No, that's different. A reservation holds..."
Developer:        "Right, so it's like a cart item?"
Business Person:  "Not exactly..."

Result: Confusion, miscommunication, bugs
```

**With ubiquitous language:**
```
Everyone agrees:
  - "Reservation" = Temporary hold on inventory (30 minutes)
  - "Order" = Confirmed purchase with payment
  - "Cart" = Collection of items before checkout

┌────────────────────────────────────────────────┐
│ Ubiquitous Language Dictionary                 │
├────────────────────────────────────────────────┤
│ Term         │ Definition                      │
├──────────────┼─────────────────────────────────┤
│ Reservation  │ 30-min inventory hold           │
│ Order        │ Confirmed purchase              │
│ Cart         │ Pre-checkout collection         │
│ Fulfillment  │ Process of delivering order     │
└────────────────────────────────────────────────┘

Business documents, code, database, and conversations
all use EXACTLY these terms
```

**Implementation example:**
```
Data Model uses business terms:

NOT:
  Table: "pending_transactions"
  Column: "user_id"

YES:
  Table: "reservations"  ← Matches business language
  Column: "customer_id"  ← Matches business language
```

**Benefits:**
- Eliminates translation errors between business and technical teams
- Code becomes self-documenting
- Easier onboarding for new team members
- Domain knowledge is captured in the system itself

**How to build it:**
```
Process:
1. Workshops with domain experts
2. Document key terms and definitions
3. Use terms consistently in:
   - Code (class names, methods, variables)
   - Database (table names, columns)
   - Documentation
   - Conversations
4. Refine as understanding deepens
5. Reject synonyms - one concept, one term
```

---

### Bounded Contexts

**What they are**: Explicit boundaries within which a particular model applies. Different parts of a system may need different models for the same concept. A bounded context is like a "jurisdiction" where specific rules and definitions apply.

**Why they matter**: In large systems, trying to use one unified model everywhere creates complexity and confusion. The same word can mean different things in different parts of the business.

**Example scenario - E-commerce system:**
```
The term "Product" means different things in different contexts:

┌─────────────────────────────────────────────────────────┐
│                    CATALOG CONTEXT                       │
│  (How customers browse and search)                      │
├─────────────────────────────────────────────────────────┤
│  Product:                                               │
│  - name                                                 │
│  - description                                          │
│  - images                                               │
│  - categories                                           │
│  - search keywords                                      │
│  - customer reviews                                     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   INVENTORY CONTEXT                      │
│  (How warehouse tracks stock)                           │
├─────────────────────────────────────────────────────────┤
│  Product:                                               │
│  - SKU                                                  │
│  - warehouse location                                   │
│  - quantity on hand                                     │
│  - reorder point                                        │
│  - supplier information                                 │
│  - dimensions and weight                                │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    PRICING CONTEXT                       │
│  (How pricing team manages prices)                      │
├─────────────────────────────────────────────────────────┤
│  Product:                                               │
│  - base price                                           │
│  - cost                                                 │
│  - margin                                               │
│  - promotional pricing rules                            │
│  - volume discounts                                     │
│  - competitor prices                                    │
└─────────────────────────────────────────────────────────┘

Same term "Product" but completely different attributes
and concerns in each context!
```

**Visual representation:**
```
                    Complete System
    ┌────────────────────────────────────────────┐
    │                                            │
    │  ┌──────────────┐    ┌──────────────┐    │
    │  │   CATALOG    │    │  INVENTORY   │    │
    │  │   CONTEXT    │    │   CONTEXT    │    │
    │  │              │    │              │    │
    │  │  Product as  │    │  Product as  │    │
    │  │  Searchable  │    │  Physical    │    │
    │  │  Item        │    │  Stock       │    │
    │  └──────────────┘    └──────────────┘    │
    │         │                    │            │
    │         └─────────┬──────────┘            │
    │                   │                       │
    │         ┌─────────┴─────────┐            │
    │         │   PRICING CONTEXT │            │
    │         │                    │            │
    │         │   Product as       │            │
    │         │   Priced Item      │            │
    │         └────────────────────┘            │
    │                                            │
    └────────────────────────────────────────────┘

Each context has its own database schema,
business rules, and team ownership
```

**Key principles:**

1. **One team per context**: Each bounded context has a dedicated team
2. **Independent deployment**: Contexts can be developed and deployed separately
3. **Clear boundaries**: Explicit interfaces between contexts
4. **Context-specific models**: No forced unification

**Real-world analogy:**
```
Think of a hospital:

Emergency Room Context:
  - "Patient" = Immediate medical status, triage level, allergies
  - Focus: Quick treatment decisions

Billing Context:
  - "Patient" = Insurance info, payment history, outstanding bills
  - Focus: Financial processing

Pharmacy Context:
  - "Patient" = Medication history, drug interactions, prescriptions
  - Focus: Safe medication dispensing

Same person, but each context needs different information
and has different responsibilities
```

---

### Context Mapping

**What it is**: A strategic design tool that documents the relationships between bounded contexts. It shows how different parts of the system communicate and depend on each other.

**Why it matters**: Prevents confusion, identifies integration points, and helps teams coordinate when their contexts need to interact.

**Types of relationships:**

**1. Shared Kernel**
```
Two contexts share a common model subset

Context A  ∩  Context B
    └──────┬──────┘
       Shared
       Model

Example:
  User Authentication context and User Profile context
  both share basic user identity information

Characteristics:
  - Changes require coordination between teams
  - Used sparingly (high coupling)
  - Common code/database
```

**2. Customer-Supplier**
```
Upstream context (Supplier) provides data/services
Downstream context (Customer) consumes them

  ┌──────────────┐
  │   Supplier   │  (Upstream)
  │   Context    │
  └───────┬──────┘
          │ API/Events
          ↓
  ┌──────────────┐
  │   Customer   │  (Downstream)
  │   Context    │
  └──────────────┘

Example:
  Inventory context (supplier) provides stock data
  → Order context (customer) checks availability

Characteristics:
  - Customer's needs influence supplier's roadmap
  - Negotiated interface
  - Dependencies managed explicitly
```

**3. Conformist**
```
Downstream context must conform to upstream model

  ┌──────────────┐
  │   Upstream   │  (Third-party or legacy)
  │   Context    │
  └───────┬──────┘
          │ Fixed API
          ↓
  ┌──────────────┐
  │ Conformist   │  (Must adapt)
  │   Context    │
  └──────────────┘

Example:
  Your system must integrate with external payment
  processor API - you adapt to their model

Characteristics:
  - No influence on upstream
  - Accept their model as-is
  - Common with third-party services
```

**4. Anti-Corruption Layer (ACL)**
```
Translation layer protects downstream from upstream complexity

  ┌──────────────┐
  │   External   │
  │   System     │
  └───────┬──────┘
          │
          ↓
  ┌──────────────┐
  │     ACL      │  ← Translates between models
  │  (Adapter)   │
  └───────┬──────┘
          │
          ↓
  ┌──────────────┐
  │  Your Clean  │
  │    Model     │
  └──────────────┘

Example:
  Legacy system uses cryptic codes
  → ACL translates to meaningful business terms
  → Your context uses clean, modern model

Characteristics:
  - Isolates your model from external complexity
  - Translation and adaptation logic
  - Protects against upstream changes
```

**5. Open Host Service**
```
Context provides well-defined API for multiple consumers

         ┌──────────────┐
         │ Open Host    │  ← Well-defined API
         │   Service    │
         └───────┬──────┘
                 │
       ┌─────────┼─────────┐
       │         │         │
       ↓         ↓         ↓
  [Context]  [Context]  [Context]
     A          B          C

Example:
  Product Catalog API serves multiple systems:
  - Website
  - Mobile app
  - Partner integrations

Characteristics:
  - Standard, documented interface
  - Multiple consumers
  - Versioned API
```

**6. Published Language**
```
Shared, standardized data format for integration

  Context A  →  [Standard Format]  →  Context B
  Context C  →  [Standard Format]  →  Context D

Example:
  All contexts exchange data using JSON with
  agreed-upon schema (OpenAPI, Protocol Buffers)

Characteristics:
  - Industry standards (JSON, XML, etc.)
  - Documented schemas
  - Reduces translation complexity
```

**Complete context map example:**
```
E-commerce System Context Map:

     ┌─────────────┐
     │   CATALOG   │ (Open Host Service)
     └──────┬──────┘
            │ REST API
      ┌─────┴─────┬──────────┐
      │           │          │
      ↓           ↓          ↓
┌──────────┐ ┌────────┐ ┌────────┐
│  ORDER   │ │ SEARCH │ │EXTERNAL│
│          │ │        │ │PARTNERS│
└────┬─────┘ └────────┘ └────────┘
     │                        ↑
     │ Customer-Supplier      │ Conformist
     ↓                        │
┌──────────┐            ┌─────────┐
│INVENTORY │            │   ACL   │
│          │            │(Adapter)│
└────┬─────┘            └─────────┘
     │
     │ Shared Kernel
     ↓
┌──────────┐
│WAREHOUSE │
│          │
└──────────┘

Legend:
→ Integration direction
ACL = Anti-Corruption Layer
```

---

### Strategic vs Tactical DDD

Domain-Driven Design operates at two levels: strategic (big picture planning) and tactical (implementation details).

**Strategic DDD** (What we've covered above)
```
Focus: High-level structure and organization

Components:
  - Ubiquitous Language
  - Bounded Contexts
  - Context Mapping
  - Core Domain identification
  - Subdomains

Goal: Ensure the right team boundaries and reduce complexity

Questions answered:
  ❓ How should we divide our system?
  ❓ Which parts are most important to the business?
  ❓ How do different parts communicate?
  ❓ What should each team focus on?
```

**Tactical DDD**
```
Focus: Implementation patterns within a bounded context

Components:
  - Entities (objects with unique identity)
  - Value Objects (objects defined by attributes)
  - Aggregates (consistency boundaries)
  - Repositories (data access abstraction)
  - Domain Events (things that happened)
  - Domain Services (operations that don't belong to entities)

Goal: Write clean, maintainable code that reflects the domain

Questions answered:
  ❓ How do we structure our code?
  ❓ What are the consistency boundaries?
  ❓ How do we persist data?
  ❓ How do we maintain invariants?
```

**Relationship:**
```
Strategic DDD guides Tactical DDD

┌───────────────────────────────────────────────┐
│           STRATEGIC DDD                       │
│  (Architecture, Boundaries, Teams)            │
│                                               │
│  ┌─────────────┐      ┌─────────────┐       │
│  │  Bounded    │      │  Bounded    │       │
│  │  Context A  │      │  Context B  │       │
│  │             │      │             │       │
│  │  ┌───────────────────────────┐  │       │
│  │  │   TACTICAL DDD            │  │       │
│  │  │   (Implementation)        │  │       │
│  │  │                           │  │       │
│  │  │  - Entities               │  │       │
│  │  │  - Value Objects          │  │       │
│  │  │  - Aggregates             │  │       │
│  │  │  - Repositories           │  │       │
│  │  └───────────────────────────┘  │       │
│  └─────────────┘      └─────────────┘       │
└───────────────────────────────────────────────┘

Strategic = Architecture decisions
Tactical = Code organization within architecture
```

**When to use each:**

```
Starting a project:
  1. Strategic DDD first ← Define contexts and boundaries
  2. Then Tactical DDD   ← Implement within contexts

Large systems:
  Strategic DDD = Essential (prevents chaos)
  Tactical DDD = Optional (use where complexity warrants)

Small systems:
  Strategic DDD = Might be overkill
  Tactical DDD = Can still provide value
```

**Example - Order Management:**

**Strategic level:**
```
Identify bounded contexts:
  - Order Taking Context
  - Payment Processing Context
  - Fulfillment Context
  - Shipping Context

Define relationships:
  Order Taking → Payment (Customer-Supplier)
  Payment → Fulfillment (Events)
  Fulfillment → Shipping (Customer-Supplier)
```

**Tactical level within Order Taking Context:**
```
Entities:
  - Order (has unique ID, changes over time)
  - Customer (has unique ID)

Value Objects:
  - Address (defined by attributes, immutable)
  - Money (amount + currency)

Aggregate:
  - Order Aggregate (Order + OrderItems)
    Ensures: Total price consistency
    Rule: Can't add items if order is confirmed

Repository:
  - OrderRepository (persist/retrieve orders)

Domain Events:
  - OrderPlaced
  - OrderConfirmed
  - OrderCancelled
```

**Key differences:**
```
┌──────────────────┬─────────────────┬─────────────────┐
│ Aspect           │ Strategic DDD   │ Tactical DDD    │
├──────────────────┼─────────────────┼─────────────────┤
│ Scope            │ System-wide     │ Within context  │
│ Participants     │ All stakeholders│ Developers      │
│ Timeline         │ Early, ongoing  │ During coding   │
│ Changes          │ Infrequent      │ Frequent        │
│ Impact           │ High            │ Medium          │
│ Artifacts        │ Context map     │ Code structure  │
└──────────────────┴─────────────────┴─────────────────┘
```

---


## Entity Relationships

Entity relationships describe how different entities (objects or concepts) in a data model connect to each other. Understanding these relationships is fundamental to proper data modeling.

### Basic Relationship Concepts

**Entity**: A thing or object that exists independently and about which we store data.
```
Examples:
  - Customer
  - Order
  - Product
  - Employee
  - Department
```

**Relationship**: A connection or association between entities.
```
Examples:
  - Customer PLACES Order
  - Employee WORKS IN Department
  - Order CONTAINS Product
```

**Cardinality**: The numerical relationship between entities - how many of one entity relate to how many of another.

---

### One-to-One Relationships (1:1)

**Definition**: Each instance of Entity A is related to exactly one instance of Entity B, and vice versa.

**Visual representation:**
```
Entity A          Entity B
   |                 |
   1 ←────────────→ 1

One A relates to exactly one B
One B relates to exactly one A
```

**Real-world examples:**

**Example 1: Person and Passport**
```
┌──────────────┐         ┌──────────────┐
│   Person     │ 1 ─── 1 │   Passport   │
├──────────────┤         ├──────────────┤
│ person_id    │◄────────│ passport_id  │
│ name         │         │ person_id    │ ← Foreign key
│ birth_date   │         │ number       │
└──────────────┘         │ issue_date   │
                         └──────────────┘

Each person has one passport
Each passport belongs to one person
```

**Example 2: User and User Settings**
```
┌──────────────┐         ┌──────────────────┐
│    User      │ 1 ─── 1 │  User Settings   │
├──────────────┤         ├──────────────────┤
│ user_id      │◄────────│ settings_id      │
│ username     │         │ user_id          │ ← Foreign key
│ email        │         │ theme            │
└──────────────┘         │ language         │
                         │ notifications    │
                         └──────────────────┘

Each user has one settings record
Each settings record belongs to one user
```

**Implementation approaches:**

**Approach 1: Separate tables with foreign key**
```
User table:
  user_id (Primary Key)
  username
  email

UserSettings table:
  settings_id (Primary Key)
  user_id (Foreign Key, UNIQUE) ← Uniqueness enforces 1:1
  theme
  language
```

**Approach 2: Single table (denormalized)**
```
User table:
  user_id (Primary Key)
  username
  email
  theme        ← Settings included directly
  language     ← Settings included directly

Use when: Relationship is truly inseparable
Trade-off: Simpler but less flexible
```

**When to use separate tables:**
- Large attribute sets (performance: don't load settings with every user query)
- Optional relationships (not every user has settings yet)
- Different access patterns (settings modified more frequently)
- Security (different permissions for settings vs user data)

**Characteristics:**
```
✓ Relatively rare in practice
✓ Often indicates potential for table merger
✓ Usually involves optional or security-separated data
✓ Foreign key must be UNIQUE to enforce 1:1
```

---

### One-to-Many Relationships (1:N)

**Definition**: Each instance of Entity A can be related to multiple instances of Entity B, but each instance of Entity B is related to only one instance of Entity A.

**Visual representation:**
```
Entity A          Entity B
   |                 |
   1 ←────────────→ N (many)

One A relates to many B
Each B relates to only one A
```

**Real-world examples:**

**Example 1: Customer and Orders**
```
┌──────────────┐         ┌──────────────┐
│  Customer    │ 1     N │    Order     │
├──────────────┤  ─────  ├──────────────┤
│ customer_id  │◄────────│ order_id     │
│ name         │         │ customer_id  │ ← Foreign key
│ email        │         │ order_date   │
└──────────────┘         │ total        │
                         └──────────────┘

One customer can place many orders
Each order belongs to one customer
```

**Example 2: Department and Employees**
```
┌──────────────┐         ┌──────────────┐
│ Department   │ 1     N │  Employee    │
├──────────────┤  ─────  ├──────────────┤
│ dept_id      │◄────────│ emp_id       │
│ dept_name    │         │ dept_id      │ ← Foreign key
│ location     │         │ name         │
└──────────────┘         │ salary       │
                         └──────────────┘

One department has many employees
Each employee belongs to one department
```

**Data representation:**
```
Customer Table:
┌─────────────┬──────────────┬─────────────────────┐
│ customer_id │ name         │ email               │
├─────────────┼──────────────┼─────────────────────┤
│ 1           │ Alice Smith  │ alice@example.com   │
│ 2           │ Bob Jones    │ bob@example.com     │
└─────────────┴──────────────┴─────────────────────┘

Order Table:
┌──────────┬─────────────┬────────────┬────────┐
│ order_id │ customer_id │ order_date │ total  │
├──────────┼─────────────┼────────────┼────────┤
│ 101      │ 1           │ 2026-01-15 │ 150.00 │ ← Alice's order
│ 102      │ 1           │ 2026-01-20 │ 75.00  │ ← Alice's order
│ 103      │ 2           │ 2026-01-22 │ 200.00 │ ← Bob's order
└──────────┴─────────────┴────────────┴────────┘
                ↑
          Foreign key pointing to Customer
```

**Querying patterns:**
```
Find all orders for a customer:
  Look up customer_id = 1
  Find all orders WHERE customer_id = 1
  Result: Orders 101, 102

Find the customer for an order:
  Look up order_id = 103
  Get customer_id = 2
  Look up customer WHERE customer_id = 2
  Result: Bob Jones
```

**Cascade operations:**
```
What happens when parent is deleted?

Option 1: CASCADE DELETE
  Delete customer_id = 1
  → Automatically delete orders 101, 102
  Use when: Child records meaningless without parent

Option 2: RESTRICT
  Delete customer_id = 1
  → Error: "Cannot delete, orders exist"
  Use when: Want to prevent accidental data loss

Option 3: SET NULL
  Delete customer_id = 1
  → Set customer_id to NULL in orders 101, 102
  Use when: Child records can exist independently

Option 4: NO ACTION
  Delete customer_id = 1
  → Leave orders with invalid customer_id
  Use when: Application handles orphans
```

**Characteristics:**
```
✓ Most common relationship type
✓ Foreign key on "many" side
✓ Natural parent-child hierarchy
✓ One query to get parent, another to get children
```

---

### Many-to-Many Relationships (N:M)

**Definition**: Each instance of Entity A can be related to multiple instances of Entity B, and each instance of Entity B can be related to multiple instances of Entity A.

**Visual representation:**
```
Entity A          Entity B
   |                 |
   N ←────────────→ N (many-to-many)

Many A relate to many B
Many B relate to many A
```

**Real-world examples:**

**Example 1: Students and Courses**
```
A student can enroll in many courses
A course can have many students enrolled

┌──────────────┐         ┌──────────────┐
│   Student    │ N     N │   Course     │
├──────────────┤  ─────  ├──────────────┤
│ student_id   │         │ course_id    │
│ name         │         │ course_name  │
│ major        │         │ credits      │
└──────────────┘         └──────────────┘
```

**Example 2: Products and Tags**
```
A product can have many tags
A tag can be applied to many products

┌──────────────┐         ┌──────────────┐
│   Product    │ N     N │     Tag      │
├──────────────┤  ─────  ├──────────────┤
│ product_id   │         │ tag_id       │
│ name         │         │ tag_name     │
│ price        │         │ category     │
└──────────────┘         └──────────────┘
```

**Implementation: Junction Table (Bridge Table)**

Many-to-many relationships cannot be directly represented with foreign keys. We need an intermediate table.

```
Student Table:                 Course Table:
┌────────────┬───────┐        ┌───────────┬──────────────┐
│ student_id │ name  │        │ course_id │ course_name  │
├────────────┼───────┤        ├───────────┼──────────────┤
│ 1          │ Alice │        │ 101       │ Mathematics  │
│ 2          │ Bob   │        │ 102       │ Physics      │
│ 3          │ Carol │        │ 103       │ Chemistry    │
└────────────┴───────┘        └───────────┴──────────────┘
        ↑                              ↑
        │                              │
        └────────┬─────────────────────┘
                 │
        ┌────────┴───────────────┐
        │   Enrollment Table     │ ← Junction/Bridge Table
        │  (Connects the two)    │
        ├────────────┬───────────┤
        │ student_id │ course_id │ ← Composite Primary Key
        ├────────────┼───────────┤
        │ 1          │ 101       │ ← Alice takes Math
        │ 1          │ 102       │ ← Alice takes Physics
        │ 2          │ 101       │ ← Bob takes Math
        │ 3          │ 102       │ ← Carol takes Physics
        │ 3          │ 103       │ ← Carol takes Chemistry
        └────────────┴───────────┘
```

**Junction table breakdown:**
```
Enrollment Table Structure:
  Primary Key: (student_id, course_id) ← Composite
  Foreign Key 1: student_id → Student.student_id
  Foreign Key 2: course_id → Course.course_id

This ensures:
  ✓ Each student-course combination appears once
  ✓ Referential integrity to both tables
  ✓ Efficient querying in both directions
```

**Enriched junction table:**

Often, the relationship itself has attributes:

```
┌──────────────┬───────────┬──────────┬────────┐
│ student_id   │ course_id │ grade    │ year   │
├──────────────┼───────────┼──────────┼────────┤
│ 1            │ 101       │ A        │ 2025   │
│ 1            │ 102       │ B+       │ 2026   │
│ 2            │ 101       │ A-       │ 2025   │
└──────────────┴───────────┴──────────┴────────┘
              ↑            ↑        ↑
          Relationship  Additional attributes
           identifiers  of the relationship itself
```

**Query patterns:**

```
Find all courses for a student (Alice, student_id=1):
  1. Look in Enrollment WHERE student_id = 1
  2. Get course_ids: 101, 102
  3. Look in Course WHERE course_id IN (101, 102)
  Result: Mathematics, Physics

Find all students in a course (Math, course_id=101):
  1. Look in Enrollment WHERE course_id = 101
  2. Get student_ids: 1, 2
  3. Look in Student WHERE student_id IN (1, 2)
  Result: Alice, Bob
```

**Alternative: Separate one-to-many relationships**
```
Instead of thinking "many-to-many", think of two one-to-many:

Student 1───N Enrollment N───1 Course

Student has many Enrollments
Course has many Enrollments
Enrollment belongs to one Student and one Course
```

**Characteristics:**
```
✓ Requires junction/bridge table
✓ Junction table has composite primary key
✓ Can store relationship-specific attributes
✓ Common in e-commerce, social networks, tagging systems
```

---

### Self-Referential Relationships

**Definition**: A relationship where an entity has a relationship with instances of itself. The entity points back to its own table.

**Visual representation:**
```
┌──────────────┐
│   Entity     │
│              │
│      ↻       │ ← Points to itself
│              │
└──────────────┘
```

**Real-world examples:**

**Example 1: Employee Management Hierarchy**
```
An employee can manage other employees
An employee can have a manager (who is also an employee)

┌────────────────────────┐
│      Employee          │
├────────────────────────┤
│ emp_id  (PK)           │
│ name                   │
│ manager_id (FK)        │───┐
└────────────────────────┘   │
         ↑                    │
         └────────────────────┘
         Points to another Employee

Data:
┌────────┬──────────────┬────────────┐
│ emp_id │ name         │ manager_id │
├────────┼──────────────┼────────────┤
│ 1      │ Sarah (CEO)  │ NULL       │ ← Top of hierarchy
│ 2      │ Tom          │ 1          │ ← Reports to Sarah
│ 3      │ Jane         │ 1          │ ← Reports to Sarah
│ 4      │ Mike         │ 2          │ ← Reports to Tom
│ 5      │ Lisa         │ 2          │ ← Reports to Tom
└────────┴──────────────┴────────────┘

Hierarchy:
        Sarah (1)
        /       \
      Tom (2)   Jane (3)
      /    \
   Mike(4) Lisa(5)
```

**Example 2: Category Tree**
```
Categories can have subcategories
Each subcategory is also a category

┌────────────────────────┐
│      Category          │
├────────────────────────┤
│ category_id (PK)       │
│ name                   │
│ parent_id (FK)         │───┐
└────────────────────────┘   │
         ↑                    │
         └────────────────────┘

Data:
┌─────────────┬─────────────────┬───────────┐
│ category_id │ name            │ parent_id │
├─────────────┼─────────────────┼───────────┤
│ 1           │ Electronics     │ NULL      │ ← Root
│ 2           │ Computers       │ 1         │ ← Under Electronics
│ 3           │ Laptops         │ 2         │ ← Under Computers
│ 4           │ Desktops        │ 2         │ ← Under Computers
│ 5           │ Home Appliances │ 1         │ ← Under Electronics
└─────────────┴─────────────────┴───────────┘

Tree structure:
Electronics (1)
├── Computers (2)
│   ├── Laptops (3)
│   └── Desktops (4)
└── Home Appliances (5)
```

**Example 3: Social Network Friends**
```
Users can be friends with other users
Friendship is bidirectional

Many-to-many self-referential relationship:

┌────────────────────────┐
│        User            │
├────────────────────────┤
│ user_id (PK)           │
│ username               │
└────────────────────────┘
         ↑         ↑
         └─────┬───┘
               │
    ┌──────────┴──────────┐
    │   Friendship        │ ← Junction table
    ├─────────────────────┤
    │ user_id1 (FK)       │
    │ user_id2 (FK)       │
    │ created_date        │
    └─────────────────────┘

Data:
Friendship Table:
┌──────────┬──────────┬──────────────┐
│ user_id1 │ user_id2 │ created_date │
├──────────┼──────────┼──────────────┤
│ 1        │ 2        │ 2026-01-10   │ ← Alice & Bob
│ 1        │ 3        │ 2026-01-15   │ ← Alice & Carol
│ 2        │ 3        │ 2026-01-20   │ ← Bob & Carol
└──────────┴──────────┴──────────────┘

Note: Store each friendship once (1,2) not twice (1,2) and (2,1)
```

**Query challenges:**

**Finding all reports (direct and indirect):**
```
For employee hierarchy, find all people under Tom (id=2):

Direct reports:
  SELECT * FROM Employee WHERE manager_id = 2
  Result: Mike (4), Lisa (5)

All reports (including indirect):
  Requires recursive query:
  - Start with manager_id = 2
  - Find their reports
  - Find reports of those reports
  - Continue until no more found

  Result: Mike (4), Lisa (5), and anyone under them
```

**Self-join patterns:**
```
Finding manager name with employee:

SELECT
  e.name AS employee,
  m.name AS manager
FROM Employee e
LEFT JOIN Employee m ON e.manager_id = m.emp_id

Result:
┌──────────┬──────────┐
│ employee │ manager  │
├──────────┼──────────┤
│ Sarah    │ NULL     │
│ Tom      │ Sarah    │
│ Jane     │ Sarah    │
│ Mike     │ Tom      │
│ Lisa     │ Tom      │
└──────────┴──────────┘
```

**Characteristics:**
```
✓ Foreign key references same table
✓ NULL value often indicates root/top level
✓ Can represent hierarchies and networks
✓ Queries often need recursion or multiple joins
✓ Common in org charts, category trees, social graphs
```

---

### Polymorphic Relationships

**Definition**: A relationship where an entity can be related to multiple different types of entities. A single foreign key can reference different tables depending on context.

**Important note**: This pattern is controversial and often discouraged because it breaks referential integrity at the database level.

**Scenario: Comments on Multiple Entity Types**
```
Users can comment on:
  - Blog Posts
  - Videos
  - Photos

Each comment belongs to ONE of these, but which one varies
```

**Approach 1: Polymorphic Foreign Key (Not Recommended)**
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  BlogPost   │  │   Video     │  │    Photo    │
├─────────────┤  ├─────────────┤  ├─────────────┤
│ post_id     │  │ video_id    │  │ photo_id    │
│ title       │  │ title       │  │ filename    │
└─────────────┘  └─────────────┘  └─────────────┘
       ↑                ↑                ↑
       └────────────────┴────────────────┘
                        │
              ┌─────────┴──────────┐
              │      Comment       │
              ├────────────────────┤
              │ comment_id         │
              │ commentable_id     │ ← Generic FK
              │ commentable_type   │ ← Type indicator
              │ content            │
              └────────────────────┘

Example data:
┌────────────┬─────────────────┬──────────────────┬─────────┐
│ comment_id │ commentable_id  │ commentable_type │ content │
├────────────┼─────────────────┼──────────────────┼─────────┤
│ 1          │ 5               │ BlogPost         │ Great!  │
│ 2          │ 12              │ Video            │ Nice    │
│ 3          │ 8               │ Photo            │ Cool    │
└────────────┴─────────────────┴──────────────────┴─────────┘
                     ↑                  ↑
              Stores ID          Stores table name
```

**Problems with this approach:**
```
✗ Database cannot enforce referential integrity
  (Can't have FK to multiple tables)
✗ commentable_id=5 could reference non-existent record
✗ Can accidentally delete referenced entity
✗ No cascading deletes
✗ Queries are complex and slow
✗ Hard to maintain data consistency
```

**Approach 2: Separate Foreign Keys (Better)**
```
              ┌────────────────────┐
              │      Comment       │
              ├────────────────────┤
              │ comment_id         │
              │ post_id     (FK)   │ ← Nullable
              │ video_id    (FK)   │ ← Nullable
              │ photo_id    (FK)   │ ← Nullable
              │ content            │
              └────────────────────┘
              Constraint: Exactly ONE must be non-NULL

Example data:
┌────────────┬─────────┬──────────┬──────────┬─────────┐
│ comment_id │ post_id │ video_id │ photo_id │ content │
├────────────┼─────────┼──────────┼──────────┼─────────┤
│ 1          │ 5       │ NULL     │ NULL     │ Great!  │
│ 2          │ NULL    │ 12       │ NULL     │ Nice    │
│ 3          │ NULL    │ NULL     │ 8        │ Cool    │
└────────────┴─────────┴──────────┴──────────┴─────────┘

Benefits:
✓ Database can enforce referential integrity
✓ Proper foreign key constraints
✓ Cascade deletes work
✓ Simpler queries
```

**Approach 3: Separate Junction Tables (Best)**
```
Create dedicated tables for each relationship:

┌────────────────┐      ┌────────────────┐
│  PostComment   │      │  VideoComment  │
├────────────────┤      ├────────────────┤
│ comment_id (FK)│      │ comment_id (FK)│
│ post_id    (FK)│      │ video_id   (FK)│
└────────────────┘      └────────────────┘

┌────────────────┐
│  PhotoComment  │
├────────────────┤
│ comment_id (FK)│
│ photo_id   (FK)│
└────────────────┘

         ↑
    All point to:
┌────────────────┐
│    Comment     │
├────────────────┤
│ comment_id     │
│ content        │
│ author_id      │
└────────────────┘

Benefits:
✓ Clean referential integrity
✓ Explicit relationships
✓ Easy to query and maintain
✓ Can add relationship-specific attributes
```

**Approach 4: Common Table (Inheritance)**
```
Use a common parent table for all commentable entities:

┌────────────────┐
│  Commentable   │ ← Abstract base
├────────────────┤
│ entity_id  (PK)│
│ entity_type    │
└────────────────┘
     ↑   ↑   ↑
     │   │   │
┌────┴┐ ┌┴──┐ ┌┴────┐
│Post│ │Vid│ │Photo│
└────┘ └───┘ └─────┘

┌────────────────┐
│    Comment     │
├────────────────┤
│ comment_id     │
│ entity_id  (FK)│ → Points to Commentable
│ content        │
└────────────────┘

Benefits:
✓ Single foreign key
✓ Referential integrity maintained
✓ Consistent interface
```

**When to use polymorphic relationships:**
```
Consider carefully - usually there are better alternatives

Acceptable when:
  ✓ Rarely need to query by related entity
  ✓ Relationships are truly equivalent
  ✓ Application layer handles integrity

Avoid when:
  ✗ Database integrity is critical
  ✗ Frequent queries on related entities
  ✗ Relationships have different semantics
  ✗ Performance matters
```

---

### Relationship Design Best Practices

**1. Start with business requirements**
```
Ask domain experts:
  - What entities exist?
  - How do they relate?
  - What are the cardinalities?
  - What are the business rules?
```

**2. Normalize appropriately**
```
Avoid redundancy:
  ✗ Storing customer name in every order
  ✓ Store customer_id, look up name when needed

But don't over-normalize:
  ✗ Separate table for every attribute
  ✓ Group related attributes logically
```

**3. Choose appropriate cascade rules**
```
Delete cascades:
  Parent-child: Usually CASCADE
  Independent entities: Usually RESTRICT
  Optional relationships: Maybe SET NULL
```

**4. Index foreign keys**
```
Always index foreign key columns for performance:
  Customer.customer_id ← Indexed automatically (PK)
  Order.customer_id ← Must index manually

Without index: Slow joins and lookups
With index: Fast relationship traversal
```

**5. Name relationships clearly**
```
Junction table names should indicate relationship:
  ✗ student_course
  ✓ enrollment (what the relationship represents)

  ✗ product_tag
  ✓ product_tagging or tagged_products
```

---


## Data Lifecycle Management

Data lifecycle management encompasses all stages of data existence in a system - from initial creation through eventual deletion or archival. Proper lifecycle management ensures data remains accurate, accessible, compliant, and cost-effective throughout its lifetime.

### The Data Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│                  Data Lifecycle Stages                   │
└─────────────────────────────────────────────────────────┘

  1. Creation/        2. Active         3. Archive
     Ingestion           Use               Storage
       │                  │                  │
       ↓                  ↓                  ↓
   [Entry Point]    [Hot Storage]      [Cold Storage]
   - Validation     - Frequent         - Infrequent
   - Cleaning         access             access
   - Initial        - Transformation   - Compressed
     storage        - Enrichment       - Cheaper
       │                  │                  │
       └──────────────────┴──────────────────┘
                          │
                          ↓
                    4. Deletion/
                       Disposal
                          │
                          ↓
                   [Secure Erasure]
                   - Compliance
                   - Audit trail
```

---

### Data Creation and Ingestion

**What it is**: The process of introducing new data into the system. This is the entry point where data begins its lifecycle.

**Sources of data creation:**

```
┌──────────────────────────────────────────────────────┐
│              Data Creation Sources                    │
├──────────────────────────────────────────────────────┤
│                                                       │
│  1. User Input                                       │
│     - Web forms                                      │
│     - Mobile apps                                    │
│     - API requests                                   │
│                                                       │
│  2. System Generation                                │
│     - Logs                                           │
│     - Audit trails                                   │
│     - Calculated metrics                             │
│                                                       │
│  3. External Integration                             │
│     - Third-party APIs                               │
│     - File uploads (CSV, JSON, XML)                  │
│     - Data feeds                                     │
│                                                       │
│  4. Migration                                        │
│     - Legacy system imports                          │
│     - Data transfers                                 │
│     - System consolidation                           │
│                                                       │
└──────────────────────────────────────────────────────┘
```

**Ingestion pipeline:**

```
Raw Data Input
      │
      ↓
┌─────────────────┐
│   Validation    │ ← Check format, types, required fields
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│  Sanitization   │ ← Remove harmful content, normalize
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│  Enrichment     │ ← Add metadata, timestamps, computed fields
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│   Deduplication │ ← Identify and handle duplicates
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│     Storage     │ ← Persist to database
└─────────────────┘
```

**Example: User Registration Ingestion**

```
1. Raw Input:
   {
     "email": " USER@EXAMPLE.COM ",
     "password": "pass123",
     "name": "John"
   }

2. Validation:
   ✓ Email format valid
   ✓ Password meets requirements
   ✗ Name too short → Reject or request full name

3. Sanitization:
   email: "user@example.com" (trimmed, lowercased)
   password: [hashed]
   name: "John" (XSS patterns removed)

4. Enrichment:
   + created_at: 2026-02-06T10:30:00Z
   + user_id: auto-generated UUID
   + email_verified: false
   + last_login: NULL

5. Storage:
   INSERT INTO users (user_id, email, password_hash, ...)
```

**Validation strategies:**

```
┌──────────────────────────────────────────────────────────┐
│ Validation Level │ Example                               │
├──────────────────┼───────────────────────────────────────┤
│ Syntax           │ Valid JSON/XML format                 │
│ Type             │ Age is integer, date is ISO format    │
│ Format           │ Email matches regex pattern           │
│ Range            │ Age between 0-150                     │
│ Business Rules   │ Order quantity ≤ stock available      │
│ Referential      │ customer_id exists in customers table │
└──────────────────────────────────────────────────────────┘
```

**Handling ingestion failures:**

```
Strategy 1: Reject and Report
  Invalid data → Return error to user
  Use when: Interactive user input

Strategy 2: Dead Letter Queue
  Invalid data → Store in error queue for manual review
  Use when: Batch processing, async ingestion

Strategy 3: Partial Accept
  Accept valid fields, reject invalid ones
  Use when: Some data is better than none

Strategy 4: Transform and Accept
  Auto-fix common issues (e.g., trim whitespace)
  Use when: Issues are predictable and safe to auto-correct
```

**Ingestion patterns:**

```
Synchronous (Real-time):
  User Action → Immediate Validation → Immediate Storage
  ↓
  Response within seconds
  Used for: User registrations, orders, transactions

Asynchronous (Batch):
  Data Added to Queue → Background Processing → Storage
  ↓
  Response: "Processing..."
  Used for: File uploads, bulk imports, data migrations

Stream Processing:
  Continuous Data Stream → Process on-the-fly → Storage
  ↓
  Real-time but decoupled
  Used for: IoT sensors, clickstream data, logs
```

---

### Data Transformation and Enrichment

**What it is**: The process of modifying, enhancing, or deriving new data from existing data. This happens throughout the active lifecycle.

**Types of transformations:**

**1. Format Transformation**
```
Change data structure or representation

Examples:
  "2026-02-06" → Unix timestamp: 1738828800
  "John Doe" → {first: "John", last: "Doe"}
  "USD 100.00" → {amount: 100.00, currency: "USD"}
```

**2. Normalization**
```
Standardize data to consistent format

Examples:
  Phone numbers:
    "555-1234" → "+1-555-1234"
    "(555) 1234" → "+1-555-1234"
    "555.1234" → "+1-555-1234"

  Addresses:
    "123 Main St." → "123 Main Street"
    "Apt. 4B" → "Apartment 4B"
```

**3. Aggregation**
```
Combine multiple records into summary data

Example: Daily Sales Report
  Individual Orders → Daily Totals

  Order 1: $50  \
  Order 2: $75   } → Daily Total: $225
  Order 3: $100 /

  Store as:
  {
    date: "2026-02-06",
    total_sales: 225.00,
    order_count: 3,
    avg_order: 75.00
  }
```

**4. Derivation**
```
Calculate new fields from existing data

Examples:
  birth_date → age (calculated)
  order_total, tax_rate → tax_amount
  quantity, unit_price → line_total

  User Table:
  ┌────────────┬─────────────┬─────┐
  │ birth_date │ signup_date │ age │ ← Derived field
  ├────────────┼─────────────┼─────┤
  │ 1990-05-15 │ 2020-01-10  │ 35  │ ← Calculated
  └────────────┴─────────────┴─────┘
```

**5. Enrichment**
```
Add contextual information from other sources

Example: IP Address Enrichment
  Input:  IP address "203.0.113.45"
  Lookup: GeoIP database
  Add:    Country, City, ISP, Coordinates

  Before:
  {
    user_id: 123,
    ip_address: "203.0.113.45"
  }

  After:
  {
    user_id: 123,
    ip_address: "203.0.113.45",
    country: "United States",
    city: "San Francisco",
    lat: 37.7749,
    lon: -122.4194
  }
```

**Transformation pipeline example:**

```
E-commerce Order Processing:

1. Raw Order Data:
   {
     "items": [
       {"id": "P123", "qty": 2},
       {"id": "P456", "qty": 1}
     ],
     "customer": "C789"
   }

2. Product Lookup (Enrichment):
   {
     "items": [
       {"id": "P123", "qty": 2, "name": "Widget", "price": 19.99},
       {"id": "P456", "qty": 1, "name": "Gadget", "price": 49.99}
     ],
     "customer": "C789"
   }

3. Calculate Totals (Derivation):
   {
     "items": [...],
     "customer": "C789",
     "subtotal": 89.97,
     "tax": 7.20,
     "total": 97.17
   }

4. Customer Info (Enrichment):
   {
     "items": [...],
     "customer": {
       "id": "C789",
       "name": "Jane Smith",
       "tier": "Gold",
       "discount": 0.10
     },
     "subtotal": 89.97,
     "discount": 8.99,
     "tax": 6.48,
     "total": 87.46
   }

5. Status Tracking (Augmentation):
   {
     ...,
     "order_id": "ORD-2026-001234",
     "status": "pending_payment",
     "created_at": "2026-02-06T10:30:00Z",
     "estimated_delivery": "2026-02-10"
   }
```

**When to transform:**

```
┌────────────────────┬─────────────────────────────────────┐
│ Timing             │ Use Case                            │
├────────────────────┼─────────────────────────────────────┤
│ On Write           │ Normalize before storage            │
│ (ETL)              │ Ensures consistency                 │
│                    │ Example: Format phone numbers       │
├────────────────────┼─────────────────────────────────────┤
│ On Read            │ Format for display                  │
│ (ELT)              │ Keeps raw data intact               │
│                    │ Example: Calculate age from DOB     │
├────────────────────┼─────────────────────────────────────┤
│ Scheduled Batch    │ Heavy aggregations                  │
│                    │ Not time-critical                   │
│                    │ Example: Daily/weekly reports       │
├────────────────────┼─────────────────────────────────────┤
│ Event-Triggered    │ When related data changes           │
│                    │ Maintain denormalized views         │
│                    │ Example: Update customer tier       │
└────────────────────┴─────────────────────────────────────┘
```

**Stored vs Computed Fields:**

```
STORED (Materialized):
  Pros:
    ✓ Fast reads (already calculated)
    ✓ Can be indexed
    ✓ Consistent snapshots
  Cons:
    ✗ Storage overhead
    ✗ Must keep synchronized
    ✗ Can become stale
  Example: Total sales per customer

COMPUTED (On-the-fly):
  Pros:
    ✓ Always current
    ✓ No storage overhead
    ✓ No synchronization needed
  Cons:
    ✗ Calculation overhead on every read
    ✗ Cannot be indexed
    ✗ Slower queries
  Example: Current age from birth date
```

---

### Data Archival Strategies

**What it is**: Moving older, less-frequently accessed data from active storage to cheaper, optimized long-term storage while maintaining accessibility when needed.

**Why archive:**
```
Benefits:
  ✓ Reduce costs (cold storage is cheaper)
  ✓ Improve performance (smaller active dataset)
  ✓ Meet compliance requirements (retain for N years)
  ✓ Simplify backups (smaller active data)
```

**Storage tiers:**

```
┌─────────────────────────────────────────────────────────┐
│                    Storage Tiers                         │
├───────────┬──────────────┬─────────────┬───────────────┤
│ Tier      │ Access Speed │ Cost/GB/mo  │ Use Case      │
├───────────┼──────────────┼─────────────┼───────────────┤
│ Hot       │ Milliseconds │ $0.15       │ Active data   │
│ (Active)  │              │             │ (last 30 days)│
├───────────┼──────────────┼─────────────┼───────────────┤
│ Warm      │ Seconds      │ $0.05       │ Recent data   │
│ (Recent)  │              │             │ (30-180 days) │
├───────────┼──────────────┼─────────────┼───────────────┤
│ Cold      │ Minutes/Hours│ $0.01       │ Old data      │
│ (Archive) │              │             │ (180+ days)   │
├───────────┼──────────────┼─────────────┼───────────────┤
│ Glacier   │ Hours/Days   │ $0.004      │ Compliance    │
│ (Deep)    │              │             │ (multi-year)  │
└───────────┴──────────────┴─────────────┴───────────────┘
```

**Archival criteria:**

```
Common rules for when to archive:

Time-based:
  - Orders older than 2 years
  - Logs older than 90 days
  - Resolved support tickets older than 1 year

Activity-based:
  - Inactive user accounts (no login for 12 months)
  - Completed projects
  - Closed cases

Business-based:
  - Fulfilled orders (after warranty period)
  - Past financial years
  - Expired contracts
```

**Archival patterns:**

**Pattern 1: Partition-Based Archival**
```
Data organized by date partitions

Active Database:
  orders_2026_01
  orders_2026_02  ← Current

Archive Storage:
  orders_2024_01
  orders_2024_02
  ...
  orders_2025_12

Process:
  1. Create new partition for new month
  2. When partition reaches age threshold (e.g., 2 years)
  3. Export partition to archive storage
  4. Drop partition from active database

Benefits:
  ✓ Easy to move entire partition
  ✓ No impact on active data
  ✓ Clear data boundaries
```

**Pattern 2: Incremental Archival**
```
Regular job moves qualifying records

Daily/Weekly job:
  1. Identify records meeting criteria:
     SELECT * FROM orders
     WHERE created_at < (NOW() - INTERVAL '2 years')
     AND archived = false

  2. Copy to archive storage

  3. Mark as archived or delete from active:
     UPDATE orders SET archived = true
     WHERE order_id IN (archived_ids)

Benefits:
  ✓ Gradual, controlled process
  ✓ Can be throttled
  ✓ Reversible (unarchive if needed)
```

**Pattern 3: Event-Triggered Archival**
```
Archive when specific events occur

Examples:
  - Order fulfilled + 30 days → Archive
  - User account deleted → Archive profile
  - Project closed → Archive project data

Process:
  Event occurs → Trigger archival workflow

Benefits:
  ✓ Immediate archival when appropriate
  ✓ Business logic-driven
  ✓ Automated based on state changes
```

**Archival storage options:**

```
1. Separate Database Schema
   Active DB: public.orders
   Archive DB: archive.orders

   Same technology, different namespace
   Easy to query if needed

2. Different Database Server
   Active: Primary PostgreSQL cluster
   Archive: Separate read-only PostgreSQL instance

   Physical separation, reduced load on primary

3. Data Warehouse
   Active: Transactional database
   Archive: Analytical warehouse (Snowflake, Redshift)

   Optimized for different access patterns

4. Object Storage
   Active: Database
   Archive: S3/Blob storage (Parquet, JSON files)

   Very cheap, but requires ETL to query

5. Tape/Glacier
   Active: Database
   Archive: AWS Glacier, tape backup

   Cheapest, but slow retrieval
```

**Accessing archived data:**

```
Option 1: Direct Query (if in database)
  - Union queries across active and archive
  - Transparent to application

Option 2: Manual Restore
  - User requests specific data
  - Admin restores from archive
  - Temporary access provided

Option 3: Lazy Loading
  - Check active storage first
  - If not found, check archive
  - Cache retrieved data temporarily

Option 4: Historical View
  - Separate interface for archived data
  - Clear indication data is archived
  - May have different SLA
```

---

### Data Deletion and Retention

**What it is**: Permanently removing data from the system according to business needs, legal requirements, and compliance regulations.

**Why delete data:**
```
Reasons:
  ✓ Legal compliance (GDPR "right to be forgotten")
  ✓ Security (reduce attack surface)
  ✓ Cost optimization (storage costs)
  ✓ Privacy (don't keep unnecessary personal data)
  ✓ Business policy (no need for ancient data)
```

**Retention policies:**

```
Define how long each data type is kept:

┌──────────────────────┬─────────────┬─────────────────┐
│ Data Type            │ Retention   │ Reason          │
├──────────────────────┼─────────────┼─────────────────┤
│ Customer orders      │ 7 years     │ Tax compliance  │
│ Access logs          │ 90 days     │ Security audit  │
│ Marketing emails     │ 2 years     │ Business need   │
│ Temp files           │ 24 hours    │ No long-term use│
│ Financial records    │ 10 years    │ Legal req.      │
│ Employee records     │ 7yrs after  │ Labor law       │
│                      │ termination │                 │
└──────────────────────┴─────────────┴─────────────────┘
```

**Deletion types:**

**1. Soft Delete (Logical)**
```
Mark as deleted, don't actually remove

Implementation:
  Add column: deleted_at TIMESTAMP NULL

  Mark deleted:
    UPDATE users
    SET deleted_at = NOW()
    WHERE user_id = 123

  Query active users:
    SELECT * FROM users
    WHERE deleted_at IS NULL

Advantages:
  ✓ Recoverable (accidental deletion)
  ✓ Audit trail maintained
  ✓ Foreign key relationships preserved
  ✓ Can analyze deletion patterns

Disadvantages:
  ✗ Data still consumes storage
  ✗ PII still exists (compliance risk)
  ✗ Queries more complex (always filter)
  ✗ Indexes larger
```

**2. Hard Delete (Physical)**
```
Permanently remove from database

Implementation:
  DELETE FROM users WHERE user_id = 123

Advantages:
  ✓ Data truly gone
  ✓ Compliant with privacy laws
  ✓ Frees storage space
  ✓ Simpler queries

Disadvantages:
  ✗ Irreversible
  ✗ May break foreign keys
  ✗ No audit trail (unless logged separately)
  ✗ Can't analyze deletion patterns
```

**Deletion strategies:**

**Strategy 1: Immediate Deletion**
```
Delete as soon as criteria met

User requests deletion → Delete immediately

Use when:
  ✓ Legal requirement (GDPR)
  ✓ Security concern
  ✓ User-initiated deletion

Risks:
  ✗ Accidental deletion
  ✗ No grace period
```

**Strategy 2: Grace Period**
```
Mark for deletion, actually delete later

User requests deletion
  ↓
Mark deleted_at = NOW()
  ↓
30-day grace period
  ↓
If not restored → Hard delete after 30 days

Use when:
  ✓ Want to prevent accidents
  ✓ Business process needs time
  ✓ Can afford storage temporarily
```

**Strategy 3: Cascade Deletion**
```
Delete related data automatically

DELETE user
  ↓ Cascades to
  - User's orders
  - User's sessions
  - User's preferences

Configuration:
  FOREIGN KEY (user_id)
  REFERENCES users(user_id)
  ON DELETE CASCADE  ← Automatic deletion

Benefits:
  ✓ Maintains referential integrity
  ✓ Automatic cleanup
  ✓ No orphaned records

Risks:
  ✗ Can delete more than intended
  ✗ May violate retention policies
```

**Strategy 4: Anonymization (Alternative to Deletion)**
```
Remove identifying information instead of deleting

Before:
  {
    user_id: 123,
    name: "John Doe",
    email: "john@example.com",
    order_total: 1500.00
  }

After anonymization:
  {
    user_id: 123,
    name: "[deleted]",
    email: "[deleted]",
    order_total: 1500.00  ← Keep for analytics
  }

Benefits:
  ✓ Maintains referential integrity
  ✓ Preserves aggregated data
  ✓ Complies with privacy laws
  ✓ Historical analysis still possible
```

**Deletion workflow example:**

```
GDPR Deletion Request:

1. Receive Request
   User submits "delete my data" request
   ↓
2. Verification
   Confirm user identity
   ↓
3. Impact Analysis
   Identify all data to be deleted:
   - User profile
   - Order history (keep financial, anonymize)
   - Reviews and comments (anonymize)
   - Session data (delete)
   - Logs containing PII (delete)
   ↓
4. Grace Period (Optional)
   30-day waiting period
   User can cancel
   ↓
5. Execution
   a. Anonymize order data
   b. Hard delete profile
   c. Soft delete reviews (mark as [deleted user])
   d. Purge logs
   ↓
6. Confirmation
   Log deletion action
   Notify user
   Update compliance records
   ↓
7. Verification
   Audit that all PII removed
   Confirm compliance
```

**Compliance considerations:**

```
GDPR (EU):
  - Right to erasure ("right to be forgotten")
  - Must delete within 30 days of request
  - Some data exempt (legal obligations)

CCPA (California):
  - Similar deletion rights
  - Must verify requester identity

HIPAA (Healthcare):
  - Cannot delete medical records during retention period
  - Specific retention requirements

Financial Regulations:
  - Must retain transaction records
  - Typically 7-10 years
  - Can anonymize PII, keep transactions
```

---

### Data Recovery Procedures

**What it is**: Processes and mechanisms to restore data after accidental deletion, corruption, or system failure.

**Recovery scenarios:**

```
1. Accidental Deletion
   User/admin deletes wrong data
   Recovery: Restore from backup

2. Data Corruption
   Hardware failure, software bug
   Recovery: Restore last known good state

3. Security Breach
   Ransomware, malicious deletion
   Recovery: Clean backup restoration

4. Operational Error
   Bad deployment, schema change gone wrong
   Recovery: Rollback or point-in-time restore

5. Disaster
   Data center failure, natural disaster
   Recovery: Failover to secondary site
```

**Recovery mechanisms:**

**1. Database Backups**
```
Types:

Full Backup:
  Complete copy of entire database
  Size: Large
  Speed: Slow to backup, fast to restore
  Frequency: Daily or weekly

Incremental Backup:
  Only changes since last backup
  Size: Small
  Speed: Fast to backup, slower to restore (need full + all increments)
  Frequency: Hourly or more frequent

Differential Backup:
  Changes since last full backup
  Size: Medium
  Speed: Medium for both
  Frequency: Daily
```

**Backup strategy example:**
```
Sunday:    Full Backup    (100 GB)
Monday:    Incremental    (5 GB)   ← Changes since Sunday
Tuesday:   Incremental    (6 GB)   ← Changes since Monday
Wednesday: Incremental    (4 GB)   ← Changes since Tuesday
...

Recovery for Wednesday:
  Restore: Sunday full + Mon + Tue + Wed incrementals
```

**2. Point-in-Time Recovery (PITR)**
```
Restore database to any specific moment

How it works:
  Full Backup + Transaction Logs

  Backup at midnight → All changes logged → Disaster at 2:37 PM

  Recovery:
    1. Restore full backup (midnight state)
    2. Replay transaction logs up to 2:36 PM
    3. Database restored to just before disaster

Benefits:
  ✓ Minimal data loss (seconds/minutes)
  ✓ Can recover to exact moment
  ✓ Useful for pinpointing issues
```

**3. Soft Delete Recovery**
```
If using soft deletes, recovery is simple:

Soft deleted:
  UPDATE users
  SET deleted_at = NOW()
  WHERE user_id = 123

Recover:
  UPDATE users
  SET deleted_at = NULL
  WHERE user_id = 123

Benefits:
  ✓ Instant recovery
  ✓ No backup needed
  ✓ User-level capability
```

**4. Archive Retrieval**
```
Restore data from archive storage

Process:
  1. Identify required data
  2. Initiate retrieval from cold storage
  3. Wait (minutes to hours)
  4. Data available in staging area
  5. Copy to active database if needed

Timeline example:
  AWS Glacier:
    - Expedited: 1-5 minutes ($$$)
    - Standard: 3-5 hours ($$)
    - Bulk: 5-12 hours ($)
```

**5. Replica/Standby Promotion**
```
If using replication:

Primary fails:
  Promote standby to primary

  Primary (failed) ----X

  Standby → Promoted → New Primary

  Minimal downtime (seconds)
  No data loss if synchronous replication
```

**Recovery procedures:**

**Procedure 1: Single Record Recovery**
```
Steps:
  1. Identify when record was deleted
  2. Find backup containing the record
  3. Extract specific record
  4. Restore to production

  Tools: Backup software with granular recovery
  Time: Minutes
```

**Procedure 2: Table/Schema Recovery**
```
Steps:
  1. Create temporary database
  2. Restore full backup to temp
  3. Export target table
  4. Import to production

  Avoids disrupting entire database
  Time: Hours
```

**Procedure 3: Full Database Recovery**
```
Steps:
  1. Take database offline (or create new)
  2. Restore latest full backup
  3. Apply incremental backups
  4. Replay transaction logs (if PITR)
  5. Validate data integrity
  6. Bring online

  Time: Hours to days (depends on size)
```

**Recovery Time Objective (RTO) and Recovery Point Objective (RPO):**

```
RTO: How long to recover (downtime tolerance)
RPO: How much data loss is acceptable

┌───────────────────────────────────────────────────────┐
│                                                        │
│  Last Backup         Disaster         Recovery        │
│      │                  │                 │           │
│      v                  v                 v           │
│  ────▪──────────────────X─────────────────▪────       │
│      ↑                  ↑                 ↑           │
│      │                  │                 │           │
│      │                  │                 │           │
│      │                  └─────RPO─────────┘           │
│      │                  (Data Loss)                   │
│      │                                                │
│      └────────────────────RTO─────────────────────────┘
│                        (Downtime)                     │
└───────────────────────────────────────────────────────┘

Examples:

Critical System:
  RTO: 1 hour (must be back up quickly)
  RPO: 5 minutes (minimal data loss acceptable)
  Strategy: Hot standby, continuous replication

Normal System:
  RTO: 24 hours (next business day)
  RPO: 24 hours (daily backup acceptable)
  Strategy: Daily backups

Non-critical:
  RTO: 1 week
  RPO: 1 week
  Strategy: Weekly backups
```

**Testing recovery procedures:**

```
Recovery Drill Schedule:

Quarterly:
  - Test restore of sample database
  - Verify backup integrity
  - Time the recovery process
  - Document any issues

Annually:
  - Full disaster recovery simulation
  - Failover to backup site
  - All teams participate
  - Update runbooks

Continuous:
  - Automated backup verification
  - Restore tests in staging
  - Alert on backup failures
```

**Recovery best practices:**

```
1. Test backups regularly
   ✓ Backup is only good if restore works
   ✓ Test in non-production environment
   ✓ Verify data integrity after restore

2. Automate backup processes
   ✓ Scheduled, consistent backups
   ✓ Alerts on failures
   ✓ Rotation and cleanup

3. Multiple backup locations
   ✓ On-site (fast recovery)
   ✓ Off-site (disaster protection)
   ✓ Cloud (additional redundancy)

4. Document procedures
   ✓ Step-by-step runbooks
   ✓ Contact information
   ✓ Decision trees

5. Monitor and alert
   ✓ Backup success/failure
   ✓ Storage capacity
   ✓ Backup age
```

---


## Schema Evolution Strategies

Schema evolution is the process of changing a database schema over time while maintaining system functionality and data integrity. As business requirements change, the data model must evolve, but this must be done carefully to avoid breaking existing functionality.

### Why Schema Evolution Matters

```
Scenario: Adding a new required field

Version 1 (Old):          Version 2 (New):
┌─────────────┐          ┌─────────────┐
│    User     │          │    User     │
├─────────────┤          ├─────────────┤
│ user_id     │          │ user_id     │
│ email       │          │ email       │
│ name        │          │ name        │
└─────────────┘          │ phone       │ ← New field!
                         └─────────────┘

Challenge: What happens to:
  - Old applications still running?
  - Existing data without phone numbers?
  - Migration of millions of records?
  - Rollback if there's a problem?
```

---

### Schema Compatibility Types

**Compatibility** defines how different schema versions interact with each other. There are three main types:

**1. Backward Compatibility**
```
Definition: New schema can read data written by old schema

┌──────────────────────────────────────────────────────┐
│              Backward Compatible                      │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Old Schema (v1)  ────writes────>  [Data]           │
│                                       │              │
│  New Schema (v2)  ────reads──────────┘              │
│                                                       │
│  New schema understands old data                     │
└──────────────────────────────────────────────────────┘

Example: Adding an optional field
  Old schema writes: {id: 1, name: "Alice"}
  New schema reads:  {id: 1, name: "Alice", phone: null}
                                             ↑
                                    Handles missing field

Use case:
  - Gradual deployment (new code, old data)
  - Reading historical data
  - During migration period
```

**2. Forward Compatibility**
```
Definition: Old schema can read data written by new schema

┌──────────────────────────────────────────────────────┐
│              Forward Compatible                       │
├──────────────────────────────────────────────────────┤
│                                                       │
│  New Schema (v2)  ────writes────>  [Data]           │
│                                       │              │
│  Old Schema (v1)  ────reads──────────┘              │
│                                                       │
│  Old schema understands new data                     │
└──────────────────────────────────────────────────────┘

Example: Adding a new field that old schema can ignore
  New schema writes: {id: 1, name: "Alice", phone: "555-1234"}
  Old schema reads:  {id: 1, name: "Alice"}
                                           ↑
                                  Ignores unknown field

Use case:
  - Rollback capability
  - Multi-version deployments
  - A/B testing
```

**3. Full Compatibility**
```
Definition: Both backward AND forward compatible

┌──────────────────────────────────────────────────────┐
│               Full Compatible                         │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Old Schema (v1)  ←────reads/writes────>  [Data]    │
│                                             ↕         │
│  New Schema (v2)  ←────reads/writes────────┘         │
│                                                       │
│  Both versions can read and write each other's data  │
└──────────────────────────────────────────────────────┘

Example: Adding optional field with default value
  Old schema: Ignores new field when reading
  New schema: Provides default for missing field

Both can coexist!

Use case:
  - Zero-downtime deployments
  - Blue-green deployments
  - Long migration periods
```

**Compatibility matrix:**

```
┌──────────────────────┬────────────┬──────────────────────┐
│ Change Type          │ Compatible │ Notes                │
├──────────────────────┼────────────┼──────────────────────┤
│ Add optional field   │ ✓ Backward │ New ignores missing  │
│                      │ ✓ Forward  │ Old ignores new      │
│                      │ ✓ Full     │                      │
├──────────────────────┼────────────┼──────────────────────┤
│ Remove optional      │ ✓ Backward │ New sees old data    │
│ field                │ ✗ Forward  │ Old breaks on new    │
├──────────────────────┼────────────┼──────────────────────┤
│ Add required field   │ ✗ Backward │ Old data invalid     │
│                      │ ✗ Forward  │ Old can't provide it │
│                      │ ✗ Full     │                      │
├──────────────────────┼────────────┼──────────────────────┤
│ Rename field         │ ✗ Backward │ Breaking change      │
│                      │ ✗ Forward  │                      │
│                      │ ✗ Full     │                      │
├──────────────────────┼────────────┼──────────────────────┤
│ Change field type    │ ✗ Usually  │ May work if          │
│                      │            │ types compatible     │
└──────────────────────┴────────────┴──────────────────────┘
```

---

### Schema Versioning

**What it is**: Explicitly tracking and managing different versions of your schema over time.

**Versioning approaches:**

**Approach 1: Version Number in Schema**
```
Include version metadata

Table: schema_versions
┌─────────┬──────────┬────────────┬──────────────────┐
│ version │ applied  │ script     │ description      │
├─────────┼──────────┼────────────┼──────────────────┤
│ 1       │ 2024-01  │ v001.sql   │ Initial schema   │
│ 2       │ 2024-02  │ v002.sql   │ Add phone field  │
│ 3       │ 2024-03  │ v003.sql   │ Add index on...  │
└─────────┴──────────┴────────────┴──────────────────┘

Benefits:
  ✓ Track what's applied
  ✓ Idempotent migrations
  ✓ Audit trail
```

**Approach 2: Migration Scripts**
```
Each change is a numbered migration file

migrations/
  001_create_users_table.sql
  002_add_users_phone.sql
  003_create_orders_table.sql
  004_add_orders_status_index.sql

Each migration:
  - Has unique ID/number
  - Up migration (apply change)
  - Down migration (revert change)

Example migration 002:

-- Up
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Down
ALTER TABLE users DROP COLUMN phone;
```

**Approach 3: Version in Application**
```
Application declares schema version it expects

Application v2.5:
  expects_schema_version: 15

Startup check:
  IF database_schema_version != 15:
    ERROR: "Schema mismatch! Expected v15, found v12"
    EXIT

Forces deployment coordination
```

**Approach 4: Schema Registry**
```
Central repository of schema definitions

Schema Registry:
  - Stores all schema versions
  - Validates new schemas
  - Enforces compatibility rules

Common in: Event-driven systems, Kafka, microservices

Example (Apache Avro):
  {
    "type": "record",
    "name": "User",
    "version": 2,
    "fields": [
      {"name": "id", "type": "int"},
      {"name": "email", "type": "string"},
      {"name": "phone", "type": ["null", "string"], "default": null}
    ]
  }
```

---

### Online Schema Migrations

**What it is**: Changing the database schema without downtime. The database remains available during the migration.

**Why it's challenging:**
```
During migration, you have:
  - Old application code running (reads/writes old schema)
  - New application code deploying (reads/writes new schema)
  - Database in transition (partially migrated)

All three must coexist!
```

**Online migration strategies:**

**Strategy 1: Expand and Contract (3-Phase Migration)**

The safest approach for production systems:

```
Phase 1: EXPAND
  Add new schema elements alongside old ones
  Both old and new exist simultaneously

Phase 2: MIGRATE
  Dual-write to both old and new
  Backfill existing data
  New code uses new schema

Phase 3: CONTRACT
  Remove old schema elements
  Clean up

Example: Renaming 'name' to 'full_name'

┌────────────────────────────────────────────────────────┐
│ Phase 1: EXPAND (Deploy #1)                            │
├────────────────────────────────────────────────────────┤
│ ALTER TABLE users ADD COLUMN full_name VARCHAR(100);  │
│                                                         │
│ Table now has BOTH:                                    │
│   - name (old)                                         │
│   - full_name (new, initially NULL)                    │
└────────────────────────────────────────────────────────┘
                        ↓
┌────────────────────────────────────────────────────────┐
│ Phase 2: MIGRATE (Deploy #2)                           │
├────────────────────────────────────────────────────────┤
│ Application changes:                                   │
│   - Write to BOTH columns                              │
│   - Read from full_name (fallback to name)             │
│                                                         │
│ Background job:                                        │
│   UPDATE users                                         │
│   SET full_name = name                                 │
│   WHERE full_name IS NULL;                             │
│                                                         │
│ Wait for all rows migrated...                          │
└────────────────────────────────────────────────────────┘
                        ↓
┌────────────────────────────────────────────────────────┐
│ Phase 3: CONTRACT (Deploy #3)                          │
├────────────────────────────────────────────────────────┤
│ ALTER TABLE users DROP COLUMN name;                   │
│                                                         │
│ Remove old column                                      │
└────────────────────────────────────────────────────────┘

Timeline: 3 separate deployments over days/weeks
```

**Strategy 2: Shadow Writing**

```
Write to new schema, read from old, gradually switch

Step 1: Deploy code that writes to BOTH
  Old application:
    - Reads from old schema
    - Writes to old schema

  New code deployed:
    - Reads from old schema (same behavior)
    - Writes to BOTH old and new ← Shadow write

Step 2: Backfill
  Copy old data to new schema

Step 3: Switch read path
  - Read from new schema
  - Still write to both

Step 4: Stop writing to old
  - Only write to new

Step 5: Remove old schema

Benefits:
  ✓ Gradual, safe transition
  ✓ Easy rollback at each step
  ✓ Can test new schema with real data
```

**Strategy 3: Blue-Green Schema**

```
Create complete new schema alongside old

Blue (Old):                Green (New):
┌─────────────┐           ┌─────────────┐
│   users     │           │   users_v2  │
│   orders    │           │   orders_v2 │
└─────────────┘           └─────────────┘
      ↑                          ↑
  Old app                   New app

Process:
  1. Create new schema
  2. Replicate data from blue to green
  3. Deploy new app pointing to green
  4. Switch traffic
  5. Verify
  6. Remove blue

Benefits:
  ✓ Complete isolation
  ✓ Easy rollback (switch back)
  ✓ Test thoroughly before switch

Challenges:
  ✗ 2x storage during transition
  ✗ Complex data synchronization
```

**Online migration best practices:**

```
1. Never delete columns/tables immediately
   ✓ Add new, migrate, remove old (expand-contract)
   ✗ Remove directly

2. Make changes in small steps
   ✓ One change per deployment
   ✗ Big bang migrations

3. Always have rollback plan
   Each step should be reversible

4. Monitor actively
   - Query performance
   - Error rates
   - Data consistency

5. Use feature flags
   Toggle between old/new behavior without redeployment

6. Lock-free when possible
   Avoid long-running locks that block operations
```

**Handling data type changes:**

```
Example: Change user_id from INT to BIGINT

Risky approach (downtime):
  ALTER TABLE users ALTER COLUMN user_id TYPE BIGINT;
  (Locks table during conversion)

Safe approach (online):
  1. Add new column:
     ALTER TABLE users ADD COLUMN user_id_new BIGINT;

  2. Populate:
     UPDATE users SET user_id_new = user_id;

  3. Dual-write phase:
     Write to both user_id and user_id_new

  4. Switch reads to user_id_new

  5. Rename:
     ALTER TABLE users RENAME COLUMN user_id TO user_id_old;
     ALTER TABLE users RENAME COLUMN user_id_new TO user_id;

  6. Clean up:
     ALTER TABLE users DROP COLUMN user_id_old;

Multiple deployments, no downtime!
```

---

### Breaking vs Non-Breaking Changes

**Non-Breaking Changes (Safe):**

```
✓ Adding optional columns
  ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;
  - Old code: Ignores new column
  - New code: Uses it
  - Existing data: NULL is fine

✓ Adding tables
  CREATE TABLE notifications (...);
  - Doesn't affect existing functionality

✓ Adding indexes
  CREATE INDEX idx_users_email ON users(email);
  - Improves performance
  - No application changes needed

✓ Widening column constraints
  VARCHAR(50) → VARCHAR(100)
  - All old data still valid

✓ Adding default values
  ALTER TABLE orders
  ALTER COLUMN status SET DEFAULT 'pending';
  - Old code unaffected
```

**Breaking Changes (Dangerous):**

```
✗ Removing columns
  ALTER TABLE users DROP COLUMN phone;
  - Old code expecting phone breaks
  - Data lost permanently

✗ Renaming columns
  ALTER TABLE users RENAME COLUMN name TO full_name;
  - Old code breaks immediately
  - Must use expand-contract

✗ Making columns non-nullable
  ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
  - Existing NULL values cause errors
  - Insertions without phone fail

✗ Narrowing column types
  VARCHAR(100) → VARCHAR(50)
  - Existing data may be truncated
  - Risk of data loss

✗ Changing column types
  VARCHAR → INT
  - Existing data may be incompatible
  - Application logic breaks

✗ Removing tables
  DROP TABLE old_table;
  - Any code using it breaks
  - Data lost

✗ Changing relationships
  Removing foreign keys, changing cardinality
  - Breaks referential integrity
  - Application logic assumptions violated
```

**Making breaking changes safely:**

```
Step 1: Announce deprecation
  - Document the change
  - Set timeline (e.g., "column removed in 30 days")
  - Give teams time to adapt

Step 2: Add new schema elements
  - Expand phase
  - Dual support period

Step 3: Migrate applications
  - Update all consumers
  - Verify no one uses old schema

Step 4: Contract
  - Remove old schema elements
  - Only after ALL applications updated
```

---

### Schema Migration Tools

**Popular tools for managing schema evolution:**

**1. Flyway**
```
Java-based migration tool

Structure:
  db/migration/
    V1__initial_schema.sql
    V2__add_users_phone.sql
    V3__create_orders_table.sql

Features:
  - Version-based migrations
  - Automatic execution
  - Rollback support
  - Works with most databases

Usage:
  flyway migrate  ← Apply pending migrations
  flyway info     ← Show status
  flyway repair   ← Fix issues
```

**2. Liquibase**
```
Database-agnostic change management

Format: XML, YAML, JSON, or SQL

Example (YAML):
  databaseChangeLog:
    - changeSet:
        id: 1
        author: alice
        changes:
          - createTable:
              tableName: users
              columns:
                - column:
                    name: id
                    type: bigint
                    autoIncrement: true

Features:
  - Multiple formats
  - Conditional execution
  - Preconditions
  - Rollback generation
```

**3. Alembic (Python)**
```
Database migration tool for SQLAlchemy

Generate migration:
  alembic revision -m "add users phone"

Creates file:
  def upgrade():
      op.add_column('users', sa.Column('phone', sa.String(20)))

  def downgrade():
      op.drop_column('users', 'phone')

Execute:
  alembic upgrade head  ← Apply all
  alembic downgrade -1  ← Rollback one
```

**4. Rails Migrations (Ruby)**
```
Built into Ruby on Rails

Generate:
  rails generate migration AddPhoneToUsers phone:string

Creates:
  class AddPhoneToUsers < ActiveRecord::Migration[7.0]
    def change
      add_column :users, :phone, :string
    end
  end

Execute:
  rails db:migrate        ← Apply
  rails db:rollback       ← Revert
```

**5. Prisma Migrate (TypeScript)**
```
Modern ORM with built-in migrations

Schema defined in Prisma file:
  model User {
    id    Int     @id @default(autoincrement())
    email String  @unique
    phone String? ← Add this
  }

Generate migration:
  prisma migrate dev --name add_user_phone

Automatically creates SQL and tracks state
```

---

### Schema Evolution Patterns

**Pattern 1: Additive Changes Only**
```
Philosophy: Only add, never remove or change

Rules:
  ✓ Add new columns (mark old as deprecated)
  ✓ Add new tables
  ✓ Add new relationships
  ✗ Never remove anything

Benefits:
  - Always backward compatible
  - No breaking changes
  - Simple to reason about

Drawbacks:
  - Schema bloat over time
  - Deprecated fields remain
  - Requires discipline
```

**Pattern 2: Parallel Schemas**
```
Run multiple schema versions simultaneously

API Gateway:
  ↓
  ├─ v1 API → Schema v1
  ├─ v2 API → Schema v2
  └─ v3 API → Schema v3

Each API version uses compatible schema version

Benefits:
  - Clear version boundaries
  - Independent evolution
  - Client chooses version

Drawbacks:
  - Multiple schemas to maintain
  - Data sync between versions
  - Complexity
```

**Pattern 3: Feature Flags + Schema**
```
Use feature flags to control schema usage

Schema has new field:
  users.preferences JSON

Old code (flag OFF):
  Doesn't use preferences field

New code (flag ON):
  Uses preferences field

Gradually roll out:
  10% users → 50% → 100%

Benefits:
  - Gradual rollout
  - Easy rollback
  - A/B testing

Requirement:
  Schema must support both code paths
```

**Pattern 4: Event Sourcing**
```
Store changes as events, derive current state

Instead of:
  UPDATE users SET name = 'Alice'

Store event:
  UserNameChanged { user_id: 123, new_name: 'Alice' }

Schema evolution:
  - Add new event types
  - Old events still valid
  - Rebuild state from all events

Benefits:
  - Complete history
  - Schema-agnostic (events are data)
  - Natural evolution

Drawbacks:
  - More complex
  - Event replay required
```

---

### Testing Schema Changes

**Testing strategies:**

```
1. Unit Tests
   Test migration scripts themselves
   - Does it run without error?
   - Does reverse migration work?

2. Integration Tests
   Test application with new schema
   - Can old code read new schema?
   - Can new code read old data?

3. Load Tests
   Test migration performance
   - How long does it take?
   - Does it lock tables?
   - Impact on production?

4. Staging Environment
   Run migration on production-like data
   - Catch issues before production
   - Verify data integrity
   - Time the process

5. Canary Deployment
   Deploy to small subset first
   - 1% traffic
   - Monitor for errors
   - Gradually increase

6. Shadow Testing
   Run new schema in parallel
   - Replicate writes
   - Compare results
   - Verify correctness
```

**Schema change checklist:**

```
Before:
  □ Document the change
  □ Plan rollback procedure
  □ Test on staging data
  □ Estimate migration time
  □ Check for lock duration
  □ Review with team
  □ Schedule maintenance window (if needed)

During:
  □ Take backup
  □ Start migration
  □ Monitor progress
  □ Watch for errors/locks
  □ Verify data integrity

After:
  □ Validate application functionality
  □ Monitor error rates
  □ Check performance metrics
  □ Verify data consistency
  □ Document results
  □ Keep rollback plan ready (24-48 hours)
```

---


## Data Integrity Constraints

Data integrity constraints are rules enforced at the database level to ensure data accuracy, consistency, and reliability. They act as gatekeepers, preventing invalid data from entering the system.

### Why Constraints Matter

```
Without Constraints:          With Constraints:
┌──────────────────┐         ┌──────────────────┐
│ Application      │         │ Application      │
│ (Tries to        │         │ (Tries to        │
│  maintain rules) │         │  maintain rules) │
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         ↓                            ↓
┌─────────────────────┐      ┌─────────────────────┐
│ Database            │      │ Database            │
│ (Accepts anything)  │      │ (Enforces rules)    │
│                     │      │                     │
│ ✗ Bugs let bad     │      │ ✓ Invalid data      │
│   data through      │      │   rejected          │
│ ✗ Direct DB access │      │ ✓ All entry points  │
│   bypasses checks   │      │   protected         │
│ ✗ Multiple apps    │      │ ✓ Consistent rules  │
│   = inconsistent    │      │   everywhere        │
└─────────────────────┘      └─────────────────────┘
```

**Defense in depth:**
```
Layer 1: User Interface Validation (UX feedback)
         ↓
Layer 2: Application Logic (Business rules)
         ↓
Layer 3: Database Constraints (Last line of defense)
         ↑
    Critical safety net
```

---

### Primary Keys and Uniqueness

**Primary Key Constraint:**

**What it is**: Ensures each row in a table can be uniquely identified. A primary key is a column (or combination of columns) that uniquely identifies each record.

**Properties:**
```
1. UNIQUE: No two rows have the same value
2. NOT NULL: Must have a value
3. IMMUTABLE: Shouldn't change (best practice)
4. ONE per table: Each table has exactly one primary key
```

**Examples:**

**Single-column primary key:**
```
CREATE TABLE users (
  user_id BIGINT PRIMARY KEY,  ← Single column PK
  email VARCHAR(255),
  name VARCHAR(100)
);

Data:
┌─────────┬──────────────────┬────────────┐
│ user_id │ email            │ name       │
├─────────┼──────────────────┼────────────┤
│ 1       │ alice@email.com  │ Alice      │
│ 2       │ bob@email.com    │ Bob        │
│ 3       │ carol@email.com  │ Carol      │
└─────────┴──────────────────┴────────────┘

Invalid attempts:
  INSERT INTO users VALUES (2, 'dave@email.com', 'Dave');
  ✗ Error: Duplicate primary key '2'

  INSERT INTO users VALUES (NULL, 'eve@email.com', 'Eve');
  ✗ Error: Primary key cannot be NULL
```

**Composite primary key:**
```
CREATE TABLE order_items (
  order_id BIGINT,
  product_id BIGINT,
  quantity INT,
  PRIMARY KEY (order_id, product_id)  ← Composite PK
);

Data:
┌──────────┬────────────┬──────────┐
│ order_id │ product_id │ quantity │
├──────────┼────────────┼──────────┤
│ 100      │ 1          │ 2        │ ← Unique combination
│ 100      │ 2          │ 1        │ ← Unique combination
│ 101      │ 1          │ 5        │ ← Unique combination
└──────────┴────────────┴──────────┘

Valid: Order 100 can have multiple products
Valid: Product 1 can be in multiple orders
Invalid: Same order + same product twice
```

**Choosing primary keys:**

```
Natural Keys:
  Use existing unique business identifier

  Examples:
    - Social Security Number
    - Email address
    - ISBN (books)
    - VIN (vehicles)

  Pros:
    ✓ Meaningful
    ✓ No extra column needed

  Cons:
    ✗ May change (people change email)
    ✗ May not be truly unique
    ✗ May be sensitive (privacy)
    ✗ May be composite

Surrogate Keys:
  Generate artificial unique identifier

  Examples:
    - Auto-incrementing integer
    - UUID/GUID
    - Custom generated ID

  Pros:
    ✓ Guaranteed unique
    ✓ Immutable
    ✓ Simple
    ✓ Performance (integer indexes)

  Cons:
    ✗ Not meaningful
    ✗ Extra column

Recommendation: Use surrogate keys
```

**Auto-increment vs UUID:**

```
Auto-increment (Sequential):
  CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,  -- PostgreSQL
    ...
  );

  Generated IDs: 1, 2, 3, 4, 5, ...

  Pros:
    ✓ Small (4 or 8 bytes)
    ✓ Indexed efficiently
    ✓ Human-readable
    ✓ Sortable by creation time

  Cons:
    ✗ Exposes record count
    ✗ Predictable (security concern)
    ✗ Coordination needed in distributed systems

UUID (Random):
  CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ...
  );

  Generated IDs:
    550e8400-e29b-41d4-a716-446655440000
    6ba7b810-9dad-11d1-80b4-00c04fd430c8

  Pros:
    ✓ Globally unique (no coordination)
    ✓ Non-sequential (security)
    ✓ Works across distributed systems
    ✓ Can be generated client-side

  Cons:
    ✗ Large (16 bytes)
    ✗ Random = poor index locality
    ✗ Not human-readable
    ✗ Not sortable
```

**Unique Constraint:**

**What it is**: Ensures values in a column (or columns) are unique, but unlike primary keys, allows NULL values and you can have multiple unique constraints per table.

```
CREATE TABLE users (
  user_id BIGINT PRIMARY KEY,
  email VARCHAR(255) UNIQUE,      ← Unique constraint
  username VARCHAR(50) UNIQUE,    ← Another unique constraint
  bio TEXT
);

Behavior:
  ✓ email must be unique across all users
  ✓ username must be unique
  ✓ Both can be NULL (unless also NOT NULL)
  ✓ NULL is special: Multiple NULLs allowed
    (NULL != NULL in SQL)

Examples:
  INSERT INTO users VALUES (1, 'alice@email.com', 'alice');
  ✓ Valid

  INSERT INTO users VALUES (2, 'alice@email.com', 'bob');
  ✗ Error: Duplicate email

  INSERT INTO users VALUES (2, NULL, 'bob');
  ✓ Valid (NULL email allowed)

  INSERT INTO users VALUES (3, NULL, 'carol');
  ✓ Valid (Multiple NULLs OK)
```

**Composite unique constraint:**
```
CREATE TABLE follows (
  follower_id BIGINT,
  followed_id BIGINT,
  created_at TIMESTAMP,
  UNIQUE (follower_id, followed_id)  ← Composite unique
);

Enforces: User can follow another user only once
Allows: User can follow multiple people
Allows: User can be followed by multiple people
```

---

### Foreign Key Constraints

**What it is**: Enforces referential integrity between tables. Ensures a value in one table must exist in another table.

**Basic foreign key:**
```
Parent Table:                    Child Table:
┌─────────────┐                 ┌──────────────┐
│   users     │                 │    orders    │
├─────────────┤                 ├──────────────┤
│ user_id (PK)│◄────────────────│ order_id (PK)│
│ name        │                 │ customer_id  │ ← FK to users
└─────────────┘                 │ order_date   │
                                 │ total        │
                                 └──────────────┘

CREATE TABLE orders (
  order_id BIGINT PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  order_date DATE,
  total DECIMAL(10,2),
  FOREIGN KEY (customer_id) REFERENCES users(user_id)
);

Enforced rules:
  ✓ customer_id must exist in users table
  ✗ Cannot insert order for non-existent customer
  ✗ Cannot delete user who has orders (by default)
```

**Foreign key actions:**

Control what happens when referenced row is updated or deleted:

```
┌──────────────────┬────────────────────────────────────────┐
│ Action           │ Behavior                               │
├──────────────────┼────────────────────────────────────────┤
│ CASCADE          │ Delete/update child rows automatically │
│ RESTRICT         │ Prevent delete/update if children exist│
│ SET NULL         │ Set child FK to NULL                   │
│ SET DEFAULT      │ Set child FK to default value          │
│ NO ACTION        │ Like RESTRICT (check at end of txn)    │
└──────────────────┴────────────────────────────────────────┘
```

**Example: CASCADE DELETE**
```
CREATE TABLE orders (
  ...
  customer_id BIGINT,
  FOREIGN KEY (customer_id)
    REFERENCES users(user_id)
    ON DELETE CASCADE  ← Cascade deletes
);

Scenario:
  Users:
    user_id=1 (Alice)

  Orders:
    order_id=100, customer_id=1
    order_id=101, customer_id=1

Action:
  DELETE FROM users WHERE user_id = 1;

Result:
  ✓ User 1 deleted
  ✓ Orders 100 and 101 AUTOMATICALLY deleted

Use when: Child records meaningless without parent
```

**Example: RESTRICT**
```
CREATE TABLE orders (
  ...
  customer_id BIGINT,
  FOREIGN KEY (customer_id)
    REFERENCES users(user_id)
    ON DELETE RESTRICT  ← Prevent deletion
);

Action:
  DELETE FROM users WHERE user_id = 1;

Result:
  ✗ Error: "Cannot delete user, orders exist"
  Must delete orders first, then user

Use when: Want to prevent accidental data loss
```

**Example: SET NULL**
```
CREATE TABLE orders (
  ...
  sales_rep_id BIGINT NULL,  ← Nullable
  FOREIGN KEY (sales_rep_id)
    REFERENCES employees(emp_id)
    ON DELETE SET NULL  ← Set to NULL
);

Scenario:
  Employee sales_rep_id=5 leaves company
  Orders 100, 101 have sales_rep_id=5

Action:
  DELETE FROM employees WHERE emp_id = 5;

Result:
  ✓ Employee deleted
  ✓ Orders 100, 101 now have sales_rep_id = NULL

Use when: Child can exist without parent reference
```

**Multiple foreign keys:**
```
One table can have multiple FKs:

CREATE TABLE order_items (
  item_id BIGINT PRIMARY KEY,
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  quantity INT,
  FOREIGN KEY (order_id) REFERENCES orders(order_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);

Relationships:
  ┌─────────┐
  │ orders  │
  └────┬────┘
       │ 1
       │
       │ N
  ┌────┴─────────┐
  │ order_items  │
  └────┬─────────┘
       │ N
       │
       │ 1
  ┌────┴────┐
  │products │
  └─────────┘
```

**Self-referencing foreign key:**
```
Table references itself:

CREATE TABLE employees (
  emp_id BIGINT PRIMARY KEY,
  name VARCHAR(100),
  manager_id BIGINT,
  FOREIGN KEY (manager_id)
    REFERENCES employees(emp_id)  ← Self-reference
);

Creates hierarchy:
  emp_id=1, manager_id=NULL (CEO)
  emp_id=2, manager_id=1    (Reports to CEO)
  emp_id=3, manager_id=1    (Reports to CEO)
  emp_id=4, manager_id=2    (Reports to emp 2)
```

---

### Check Constraints

**What it is**: Custom validation rules on column values or row data.

**Column-level checks:**
```
CREATE TABLE products (
  product_id BIGINT PRIMARY KEY,
  name VARCHAR(100),
  price DECIMAL(10,2) CHECK (price > 0),  ← Must be positive
  stock INT CHECK (stock >= 0)            ← Cannot be negative
);

Invalid attempts:
  INSERT INTO products VALUES (1, 'Widget', -10.00, 5);
  ✗ Error: Check constraint violated (price > 0)

  INSERT INTO products VALUES (1, 'Widget', 19.99, -5);
  ✗ Error: Check constraint violated (stock >= 0)
```

**Table-level checks:**
```
CREATE TABLE orders (
  order_id BIGINT PRIMARY KEY,
  order_date DATE,
  ship_date DATE,
  total DECIMAL(10,2),
  CONSTRAINT valid_dates CHECK (ship_date >= order_date),
  CONSTRAINT valid_total CHECK (total > 0)
);

Enforces:
  ✗ Cannot ship before order date
  ✗ Total must be positive

Example:
  INSERT INTO orders VALUES (
    1,
    '2026-02-06',
    '2026-02-05',  ← Before order date!
    100.00
  );
  ✗ Error: Check constraint 'valid_dates' violated
```

**Common check constraint patterns:**

**1. Range validation**
```
age INT CHECK (age BETWEEN 0 AND 150)
rating INT CHECK (rating IN (1, 2, 3, 4, 5))
percentage DECIMAL CHECK (percentage >= 0 AND percentage <= 100)
```

**2. Format validation** (limited in SQL)
```
-- Email must contain @
email VARCHAR(255) CHECK (email LIKE '%@%')

-- Phone format (US)
phone VARCHAR(15) CHECK (phone ~ '^\d{3}-\d{3}-\d{4}$')

-- Postal code
zip VARCHAR(10) CHECK (LENGTH(zip) = 5 OR LENGTH(zip) = 10)
```

**3. Business rules**
```
-- Discount cannot exceed price
CREATE TABLE order_items (
  ...
  price DECIMAL(10,2),
  discount DECIMAL(10,2),
  CONSTRAINT discount_valid CHECK (discount <= price)
);

-- End date must be after start date
CREATE TABLE subscriptions (
  ...
  start_date DATE,
  end_date DATE,
  CONSTRAINT dates_valid CHECK (end_date > start_date)
);

-- Status transitions
CREATE TABLE orders (
  ...
  status VARCHAR(20) CHECK (
    status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')
  )
);
```

**4. Conditional constraints**
```
-- If type is 'employee', employee_id must be set
CREATE TABLE users (
  user_id BIGINT PRIMARY KEY,
  user_type VARCHAR(20),
  employee_id BIGINT,
  CONSTRAINT emp_id_required CHECK (
    user_type != 'employee' OR employee_id IS NOT NULL
  )
);

-- If discount applied, discount_reason must be provided
CREATE TABLE orders (
  ...
  discount DECIMAL(10,2),
  discount_reason TEXT,
  CONSTRAINT discount_reason_required CHECK (
    discount = 0 OR discount_reason IS NOT NULL
  )
);
```

**Limitations:**
```
✗ Cannot reference other tables (use triggers for that)
✗ Cannot use subqueries
✗ Cannot call functions (database-dependent)
✗ Checked at row level, not during transaction
```

---

### NOT NULL Constraints

**What it is**: Ensures a column must always have a value - NULL is not allowed.

**Basic usage:**
```
CREATE TABLE users (
  user_id BIGINT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,     ← Required field
  name VARCHAR(100) NOT NULL,      ← Required field
  bio TEXT                          ← Optional (NULL allowed)
);

Valid:
  INSERT INTO users VALUES (1, 'alice@email.com', 'Alice', NULL);
  ✓ bio can be NULL

Invalid:
  INSERT INTO users VALUES (2, NULL, 'Bob', 'Bio text');
  ✗ Error: email cannot be NULL

  INSERT INTO users (user_id, email) VALUES (3, 'carol@email.com');
  ✗ Error: name cannot be NULL (not provided)
```

**NULL vs empty string:**
```
These are DIFFERENT:

NULL:           No value, unknown, not applicable
Empty string:   A value, it just happens to be empty ('')

CREATE TABLE products (
  description TEXT NOT NULL
);

Valid:
  INSERT INTO products VALUES ('');  ← Empty string
  ✓ Empty string is a value

Invalid:
  INSERT INTO products VALUES (NULL);
  ✗ NULL is not allowed
```

**Adding NOT NULL to existing column:**
```
Problem: Column has NULL values

ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
✗ Error: Column contains NULL values

Solution:
  1. Update NULLs to valid values:
     UPDATE users SET phone = '' WHERE phone IS NULL;

  2. Then add constraint:
     ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
     ✓ Success

Or provide default:
  ALTER TABLE users ALTER COLUMN phone SET DEFAULT '';
  UPDATE users SET phone = DEFAULT WHERE phone IS NULL;
  ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

**When to use NOT NULL:**
```
✓ Required business data
  - User must have email
  - Order must have customer
  - Product must have price

✗ Optional data
  - Phone number (not everyone has one)
  - Middle name (not everyone has one)
  - Company name (for individual customers)

✓ Foreign keys (usually)
  - Order MUST belong to customer

  Exceptions:
    - Optional relationships
    - Self-referencing (manager_id for CEO is NULL)
```

---

### Application-Level Constraints

**What they are**: Validation rules enforced by application code, not the database. These complement database constraints.

**Why both levels?**

```
┌───────────────────────┬───────────────────────────────┐
│ Database Constraints  │ Application Constraints       │
├───────────────────────┼───────────────────────────────┤
│ Last line of defense  │ First line of defense         │
│ Protects data         │ Provides UX                   │
│ All entry points      │ Specific application          │
│ Simple rules          │ Complex business logic        │
│ Performance cost      │ No DB round-trip              │
│ Hard to change        │ Easy to change                │
└───────────────────────┴───────────────────────────────┘

Best practice: Use BOTH
  - Application for UX and complex rules
  - Database for data integrity guarantee
```

**Examples:**

**1. Complex business rules**
```
Database: Hard to express

Application: Easy
  def validate_order(order):
      # Can only order alcohol if customer is 21+
      if has_alcohol(order.items):
          customer = get_customer(order.customer_id)
          if customer.age < 21:
              raise ValidationError("Must be 21+ to order alcohol")

      # Cannot exceed customer credit limit
      if order.total > customer.credit_limit:
          raise ValidationError("Exceeds credit limit")

      # Shipping address must be in delivery area
      if not in_delivery_area(order.shipping_address):
          raise ValidationError("Address outside delivery area")
```

**2. Cross-table validation**
```
Database: Would need triggers (complex)

Application: Straightforward
  def create_booking(user_id, resource_id, start_time, end_time):
      # User can't have overlapping bookings
      existing = get_user_bookings(user_id, start_time, end_time)
      if existing:
          raise ValidationError("You have conflicting booking")

      # Resource must be available
      if not resource_available(resource_id, start_time, end_time):
          raise ValidationError("Resource not available")
```

**3. External system validation**
```
Cannot be done in database

Application only:
  def validate_address(address):
      # Call address validation API
      result = address_service.validate(address)
      if not result.valid:
          raise ValidationError(f"Invalid address: {result.error}")

      # Normalize address to validated format
      return result.normalized_address
```

**4. Async validation**
```
Some checks take time

Application with background processing:
  def submit_document(document):
      # Quick checks
      if not document.has_content():
          raise ValidationError("Document is empty")

      # Accept document
      document.status = 'pending'
      save(document)

      # Queue slow validation
      queue.enqueue(async_validate_document, document.id)

  def async_validate_document(document_id):
      document = load(document_id)

      # Slow checks
      if virus_detected(document):
          document.status = 'rejected'
      elif format_invalid(document):
          document.status = 'rejected'
      else:
          document.status = 'approved'

      save(document)
```

**5. User experience validation**
```
Provide immediate feedback

Client-side (JavaScript):
  - Email format check (before submit)
  - Password strength meter
  - Real-time field validation

  Benefits:
    ✓ Instant feedback
    ✓ Better UX
    ✓ Reduces server load

  But ALWAYS validate server-side too!
  (Client can be bypassed)
```

**Validation layers:**

```
Request Flow:

┌──────────────────────┐
│ Client-Side          │ ← UX, immediate feedback
│ (JavaScript)         │
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Application Layer    │ ← Complex business logic
│ (API/Server Code)    │   Cross-table validation
└──────────┬───────────┘   External API calls
           │
           ↓
┌──────────────────────┐
│ Database Layer       │ ← Data integrity
│ (Constraints)        │   Last line of defense
└──────────────────────┘

Each layer catches different classes of errors
```

---

## Summary: Data Modeling Best Practices

**1. Domain-Driven Design**
```
✓ Use ubiquitous language throughout
✓ Define clear bounded contexts
✓ Map context relationships explicitly
✓ Start with strategic design, then tactical
```

**2. Entity Relationships**
```
✓ Model real-world relationships accurately
✓ Use appropriate cardinality (1:1, 1:N, N:M)
✓ Normalize to reduce redundancy
✓ Denormalize strategically for performance
✓ Document relationship semantics
```

**3. Data Lifecycle**
```
✓ Validate thoroughly on ingestion
✓ Enrich data to add value
✓ Archive old data to optimize costs
✓ Implement retention policies
✓ Have robust recovery procedures
```

**4. Schema Evolution**
```
✓ Maintain backward compatibility when possible
✓ Use versioned migrations
✓ Test changes on staging first
✓ Use expand-contract for breaking changes
✓ Never delete immediately - deprecate first
```

**5. Integrity Constraints**
```
✓ Use primary keys on every table
✓ Enforce foreign key relationships
✓ Add NOT NULL to required fields
✓ Use CHECK constraints for business rules
✓ Combine database + application validation
```

**Final Principle:**
```
Database constraints are your safety net
  - Prevent invalid data at the source
  - Protect against bugs in application code
  - Guard against direct database manipulation
  - Maintain data quality over time

They're harder to change than application code,
so design them carefully and thoughtfully.
```

---

**End of Document**