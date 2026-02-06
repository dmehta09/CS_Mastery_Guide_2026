# Database Design Fundamentals

## Introduction

Database design is the process of organizing data into structured storage systems that allow efficient retrieval, modification, and management. Think of it like designing a library: you need to decide how to categorize books, where to place them, and how people will find what they need quickly.

Good database design is foundational to building scalable, maintainable software systems. Poor design choices early on can lead to performance bottlenecks, data inconsistencies, and expensive rewrites later.

This guide covers the essential concepts every software engineer should understand when designing database systems in 2026.

---

## SQL vs NoSQL Selection

Choosing between SQL (relational) and NoSQL (non-relational) databases is one of the first critical decisions in database design. This choice impacts how you structure data, how your application queries it, and how it scales.

### Understanding the Options

**SQL Databases** (also called relational databases) organize data into tables with predefined schemas (structure). Examples: PostgreSQL, MySQL, Oracle, SQL Server.

**NoSQL Databases** come in several types, each optimized for different use cases:

- **Document Stores**: Store data as JSON-like documents (MongoDB, CouchDB)
- **Wide-Column Stores**: Store data in columns rather than rows (Cassandra, HBase)
- **Key-Value Stores**: Simple lookup by unique keys (Redis, DynamoDB)
- **Graph Databases**: Optimized for relationship-heavy data (Neo4j, Amazon Neptune)

### Decision Matrix

```
┌─────────────────────────────────────────────────────────────────┐
│                    SELECTION CRITERIA                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Need ACID guarantees?                                          │
│  (all-or-nothing operations, strict consistency)                │
│         YES → SQL Database                                      │
│                                                                 │
│  Schema changes frequently?                                     │
│  (adding new fields, varying structure)                         │
│         YES → Document Store (MongoDB, CouchDB)                 │
│                                                                 │
│  Extremely high write throughput?                               │
│  (millions of writes per second, time-series data)              │
│         YES → Wide-Column Store (Cassandra, HBase)              │
│                                                                 │
│  Complex relationships between entities?                        │
│  (social networks, recommendation engines)                      │
│         YES → Graph Database (Neo4j, Neptune)                   │
│                                                                 │
│  Simple lookup by ID/key?                                       │
│  (caching, session storage, real-time metrics)                  │
│         YES → Key-Value Store (Redis, DynamoDB)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### ACID Requirements → SQL

**ACID** stands for Atomicity, Consistency, Isolation, Durability (explained in detail later). If your application requires these guarantees—such as financial transactions, inventory management, or booking systems—SQL databases are typically the right choice.

**Why?** SQL databases are designed from the ground up to enforce ACID properties. A bank transfer must either complete fully (debit one account, credit another) or not happen at all—no partial states allowed.

**Example scenarios:**
- E-commerce checkout and payment processing
- Banking and financial systems
- Healthcare record systems
- Reservation and booking platforms

### Flexible Schema → Document Stores

If your data structure evolves rapidly or varies between records, document stores offer flexibility that SQL schemas cannot match easily.

**Layman explanation:** Imagine storing information about different products. Some products have "color" and "size", others have "memory" and "processor". In SQL, you'd need to accommodate all possible fields. In a document store, each product can have its own unique set of fields.

**Example scenarios:**
- Content management systems (varying article types)
- User profiles with custom fields
- Product catalogs with diverse attributes
- IoT sensor data (different sensor types)

### High Write Throughput → Wide-Column Stores

When your system needs to handle millions of writes per second—often distributed across multiple data centers—wide-column stores excel.

**Layman explanation:** Think of traditional databases like notebooks where you write one entry at a time. Wide-column stores are like having thousands of people simultaneously writing into different sections of a massive ledger, with the system automatically organizing everything.

**How they work:** Data is stored in columns grouped into column families, distributed across many servers. Writes are extremely fast because they're append-only operations (adding new data rather than modifying existing data).

**Example scenarios:**
- Time-series data (metrics, logs, sensor readings)
- Event logging and analytics pipelines
- Real-time recommendation engines
- Messaging applications at massive scale

**Trade-off:** You sacrifice some consistency guarantees and query flexibility for incredible write performance and horizontal scalability (adding more servers to handle more load).

### Relationship-Heavy → Graph Databases

When your data is primarily about connections between things—who knows whom, what influences what, how items relate—graph databases are purpose-built for this.

**Layman explanation:** Imagine trying to find "friends of friends who like the same movies as you" in a traditional database. You'd need multiple complex queries joining many tables. In a graph database, you simply "walk" the relationship paths, which is far more efficient.

**Structure:**
```
    Person ─────[KNOWS]────→ Person
      │                        │
   [LIKES]                  [LIKES]
      │                        │
      ↓                        ↓
    Movie ←────[SIMILAR_TO]─── Movie
```

**Example scenarios:**
- Social networks (connections, recommendations)
- Fraud detection (finding suspicious patterns)
- Knowledge graphs (Wikipedia-style linked information)
- Supply chain and logistics networks

**Trade-off:** Graph databases are optimized for traversing relationships but may not be ideal for simple tabular data or aggregation queries.

### Simple Key-Value Access → Key-Value Stores

If you primarily need to store and retrieve data by a unique identifier (like looking up a word in a dictionary), key-value stores offer the simplest and fastest solution.

**Structure:**
```
┌──────────────┬────────────────────────────────┐
│     KEY      │            VALUE               │
├──────────────┼────────────────────────────────┤
│ user:12345   │ {"name":"Alice","age":28,...}  │
│ session:abc  │ {"cart":[...], "loggedIn":true}│
│ cache:query1 │ [result data...]               │
└──────────────┴────────────────────────────────┘
```

**Layman explanation:** It's like a locker system—you have a key (locker number), and you can instantly store or retrieve whatever's inside. You don't search through the contents; you just know the key.

**Example scenarios:**
- Caching layer (storing frequently accessed data)
- Session management (user login states)
- Rate limiting counters
- Real-time leaderboards and counters

**Trade-off:** Extremely fast and simple, but you can only query by the key. You can't search within values or perform complex queries.

### Comprehensive Trade-offs Analysis

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        DATABASE TYPE COMPARISON                            │
├──────────────┬──────────┬───────────┬──────────────┬────────────┬──────────┤
│              │   SQL    │ Document  │ Wide-Column  │   Graph    │ Key-Val  │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ ACID         │   ✓✓✓    │    ✓      │      -       │     ✓✓     │    -     │
│ Guarantees   │          │           │              │            │          │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ Schema       │ Rigid    │ Flexible  │   Flexible   │  Flexible  │  None    │
│ Flexibility  │          │           │              │            │          │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ Write        │   ✓✓     │    ✓✓     │     ✓✓✓      │     ✓      │   ✓✓✓    │
│ Performance  │          │           │              │            │          │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ Read         │   ✓✓     │    ✓✓     │      ✓       │     ✓✓     │   ✓✓✓    │
│ Performance  │          │           │              │            │          │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ Complex      │   ✓✓✓    │    ✓      │      -       │     ✓✓     │    -     │
│ Queries      │          │           │              │            │          │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ Relationship │   ✓✓     │    ✓      │      -       │    ✓✓✓     │    -     │
│ Queries      │          │           │              │            │          │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ Horizontal   │    ✓     │    ✓✓     │     ✓✓✓      │     ✓      │   ✓✓✓    │
│ Scaling      │          │           │              │            │          │
├──────────────┼──────────┼───────────┼──────────────┼────────────┼──────────┤
│ Learning     │   Easy   │   Easy    │   Moderate   │  Moderate  │   Easy   │
│ Curve        │          │           │              │            │          │
└──────────────┴──────────┴───────────┴──────────────┴────────────┴──────────┘

Legend: ✓✓✓ Excellent | ✓✓ Good | ✓ Fair | - Poor/Limited
```

### Key Trade-offs to Consider

**1. Consistency vs Availability**

SQL databases typically choose consistency (everyone sees the same data immediately). Many NoSQL databases choose availability (the system stays responsive even if some data might be slightly out of date).

**Layman explanation:** Imagine a bank account. SQL ensures both you and your spouse see the same balance instantly (consistency). Some NoSQL systems would let you both make withdrawals even if they haven't synced yet (availability), which could cause problems.

**2. Vertical vs Horizontal Scaling**

- **Vertical scaling** (SQL's traditional strength): Make one server bigger and more powerful
- **Horizontal scaling** (NoSQL's strength): Add more servers to distribute the load

**Layman explanation:** Vertical is like replacing your car with a bigger truck. Horizontal is like having a fleet of cars. Eventually, you can't build a bigger truck, but you can always add more cars to the fleet.

**3. Query Flexibility vs Performance**

SQL databases let you query data in countless ways (any combination of filters, sorts, joins). NoSQL databases often require you to know your query patterns upfront and optimize for those specific patterns.

**Trade-off:** SQL gives you maximum flexibility but may be slower for specific queries. NoSQL is blazing fast for designed queries but inflexible for ad-hoc analysis.

**4. Data Integrity vs Development Speed**

SQL's strict schemas catch errors early (you can't insert a string where a number is expected). NoSQL's flexibility lets developers iterate faster but can lead to inconsistent data if not carefully managed.

### Making the Right Choice

**Start with SQL if:**
- Your data has clear, stable relationships
- You need strong consistency guarantees
- Your team is familiar with relational concepts
- You're building traditional business applications

**Consider NoSQL if:**
- You need massive horizontal scalability
- Your schema changes frequently
- You have specialized access patterns (key lookups, graph traversals, time-series)
- You're willing to manage consistency in your application layer

**Hybrid approach (common in 2026):**
Many modern applications use **polyglot persistence**—different databases for different parts of the system:
- SQL for transactional data (orders, payments)
- Document store for user profiles and content
- Key-value store for caching and sessions
- Graph database for recommendations and social features

---

## ACID Properties Deep Dive

ACID is an acronym for four properties that guarantee reliable database transactions. These properties ensure that even when multiple operations happen simultaneously or systems crash, your data remains correct and consistent.

**Layman explanation:** Think of ACID like safety features in a car. Atomicity is the airbag (all-or-nothing protection), Consistency is the alignment (everything stays properly balanced), Isolation is your lane (other drivers don't interfere with you), and Durability is the black box recorder (everything important is permanently saved).

### Atomicity: All or Nothing Transactions

**Atomicity** ensures that a transaction (a group of database operations) either completes entirely or doesn't happen at all. There are no partial successes.

**Real-world example:**

Imagine transferring $100 from Account A to Account B:
1. Deduct $100 from Account A
2. Add $100 to Account B

If the system crashes after step 1 but before step 2, atomicity ensures both steps are rolled back (undone). The money doesn't vanish.

**How it works:**

```
┌─────────────────────────────────────────────────────────────┐
│                    TRANSACTION LIFECYCLE                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  BEGIN TRANSACTION                                          │
│       │                                                     │
│       ├──→ Operation 1: Deduct $100 from Account A         │
│       │                                                     │
│       ├──→ Operation 2: Add $100 to Account B              │
│       │                                                     │
│       ├──→ Operation 3: Log transaction                    │
│       │                                                     │
│       ├──→ All succeeded?                                  │
│       │         │                                           │
│       │         ├── YES → COMMIT (make permanent)          │
│       │         │                                           │
│       │         └── NO → ROLLBACK (undo everything)        │
│       │                                                     │
│  END TRANSACTION                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation:** Databases achieve atomicity using transaction logs and rollback mechanisms. Before making changes, the database writes what it's about to do. If something fails, it uses this log to undo changes.

### Consistency: Valid State Transitions

**Consistency** ensures the database always moves from one valid state to another valid state, respecting all defined rules, constraints, and relationships.

**Layman explanation:** It's like a chess game's rules. A knight can only move in an L-shape. Consistency prevents you from making illegal moves—you can't suddenly move a knight like a queen. Similarly, the database enforces your business rules.

**Types of consistency rules:**

1. **Data type constraints:** Age must be a number, email must contain "@"
2. **Uniqueness constraints:** No two users can have the same email address
3. **Referential integrity:** You can't place an order for a product that doesn't exist
4. **Domain constraints:** Quantity must be positive, percentage must be 0-100
5. **Custom business rules:** Account balance cannot go below zero

**Example violation prevented by consistency:**

```
Initial State (VALID):
  Products Table: [Product ID: 123, Name: "Laptop"]
  Orders Table:   [Order ID: 1, Product ID: 123, Qty: 1]

Attempted Invalid Operation:
  DELETE Product 123

Result:
  ❌ REJECTED - Would violate referential integrity
     (Order 1 would reference non-existent product)

Alternative Valid Operations:
  ✓ First delete/update Order 1, THEN delete Product 123
  ✓ Or use CASCADE DELETE (automatically delete related orders)
```

**How it works:** Databases check all constraints before committing a transaction. If any rule would be violated, the entire transaction is rolled back.

### Isolation: Concurrent Transaction Handling

**Isolation** ensures that concurrent transactions (multiple operations happening at the same time) don't interfere with each other. Each transaction appears to run in isolation, even though many may execute simultaneously.

**Layman explanation:** Imagine 10 people editing the same Google Doc simultaneously. Without isolation, you'd see half-written words, conflicting changes, and chaos. Isolation is like giving each person their own temporary copy, then merging changes intelligently.

**Problems isolation prevents:**

**1. Dirty Reads** - Reading uncommitted changes from another transaction
```
Timeline:
  T1: BEGIN
  T1: UPDATE balance = 500 (from 1000)
  T2: BEGIN
  T2: READ balance → sees 500 (DIRTY READ - T1 not committed yet!)
  T1: ROLLBACK (changed mind, undo to 1000)
  T2: Now working with wrong data (500 instead of 1000)
```

**2. Non-Repeatable Reads** - Same query returns different results during one transaction
```
Timeline:
  T1: BEGIN
  T1: READ balance → 1000
  T2: BEGIN
  T2: UPDATE balance = 500
  T2: COMMIT
  T1: READ balance → 500 (DIFFERENT from first read!)
  T1: Confused - expected same value
```

**3. Phantom Reads** - New rows appear during a transaction
```
Timeline:
  T1: BEGIN
  T1: SELECT COUNT(*) FROM accounts → returns 100
  T2: BEGIN
  T2: INSERT new account
  T2: COMMIT
  T1: SELECT COUNT(*) FROM accounts → returns 101 (PHANTOM!)
  T1: Confused - count changed during transaction
```

**Isolation Levels** (trade-off between safety and performance):

```
┌────────────────────────────────────────────────────────────────┐
│                    ISOLATION LEVELS                            │
├────────────────────┬───────────┬───────────┬───────────────────┤
│                    │  Dirty    │   Non-    │    Phantom        │
│  Level             │  Reads    │ Repeatable│     Reads         │
├────────────────────┼───────────┼───────────┼───────────────────┤
│ READ UNCOMMITTED   │  Allowed  │  Allowed  │    Allowed        │
│ (fastest)          │           │           │                   │
├────────────────────┼───────────┼───────────┼───────────────────┤
│ READ COMMITTED     │ Prevented │  Allowed  │    Allowed        │
│ (common default)   │           │           │                   │
├────────────────────┼───────────┼───────────┼───────────────────┤
│ REPEATABLE READ    │ Prevented │ Prevented │    Allowed        │
│                    │           │           │                   │
├────────────────────┼───────────┼───────────┼───────────────────┤
│ SERIALIZABLE       │ Prevented │ Prevented │   Prevented       │
│ (safest, slowest)  │           │           │                   │
└────────────────────┴───────────┴───────────┴───────────────────┘
```

**Implementation mechanisms** (how databases achieve isolation):

**Lock-based Concurrency Control:**
- Transactions acquire locks on data they're reading/writing
- Other transactions wait until locks are released
- Simple but can cause performance bottlenecks

**Multiversion Concurrency Control (MVCC):**
- Database keeps multiple versions of data
- Readers see a snapshot from when their transaction started
- Writers create new versions rather than blocking readers
- Used by PostgreSQL, MySQL InnoDB, Oracle

```
MVCC Example:

  Actual Data: balance = 1000 (version 1)

  T1 starts → Gets snapshot at version 1
  T2 updates balance = 500 → Creates version 2
  T2 commits
  T1 still reads version 1 (balance = 1000) → Consistent!

  New transactions see version 2
  Old transactions see version 1 until they commit
```

### Durability: Committed Data Survives Failures

**Durability** guarantees that once a transaction is committed (confirmed complete), the changes are permanent—even if the system crashes immediately afterward.

**Layman explanation:** It's like saving your document. Before clicking "Save," a power outage loses your work. After "Save," the file survives even if your computer crashes. Durability is the database's "Save" guarantee.

**Implementation: Write-Ahead Logging (WAL)**

The most common mechanism for durability is the Write-Ahead Log:

```
┌────────────────────────────────────────────────────────────┐
│              WRITE-AHEAD LOGGING PROCESS                   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. Transaction executes                                   │
│     UPDATE balance = 500 WHERE account = 'A'               │
│                                                            │
│  2. BEFORE changing actual database:                       │
│     → Write to WAL: "Will change account A to 500"        │
│     → Flush WAL to disk (permanent storage)                │
│                                                            │
│  3. NOW safe to modify actual database                     │
│     → Change in-memory data                                │
│     → Eventually flush to disk (can happen later)          │
│                                                            │
│  4. COMMIT confirmed to application                        │
│                                                            │
│  ┌──────────────────────────────────────────┐             │
│  │  If crash happens at any point:          │             │
│  │  → On restart, replay WAL                │             │
│  │  → Redo committed transactions            │             │
│  │  → Undo uncommitted transactions          │             │
│  │  → Database restored to correct state    │             │
│  └──────────────────────────────────────────┘             │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Why WAL works:**

1. **Sequential writes are fast:** WAL writes happen sequentially (like writing in a notebook), which is much faster than random writes to different parts of the database
2. **Small writes protect large data:** WAL entry might be 100 bytes, protecting a 1GB change
3. **Recovery is possible:** Even if the actual database file is corrupted, you can rebuild it from the WAL

**Other durability mechanisms:**

- **Replication:** Copies of data on multiple servers
- **Checksums:** Detect corrupted data
- **Battery-backed caches:** Protect in-memory changes during power loss
- **Synchronous commits:** Don't acknowledge commit until data is on disk

### ACID in Practice: Real-World Scenarios

**E-commerce Order Processing:**
- **Atomicity:** Reduce inventory AND create order record (both or neither)
- **Consistency:** Can't order 5 items if only 3 in stock
- **Isolation:** Two customers can't both buy the last item
- **Durability:** Order confirmed to customer means it's permanently recorded

**Banking Transfers:**
- **Atomicity:** Debit source, credit destination (both or neither)
- **Consistency:** Total money in system never changes
- **Isolation:** Two transfers from same account don't cause negative balance
- **Durability:** Transfer confirmation means money moved permanently

**Flight Booking:**
- **Atomicity:** Reserve seat AND charge card (both or neither)
- **Consistency:** Can't double-book same seat
- **Isolation:** Multiple people can't book same seat simultaneously
- **Durability:** Booking confirmation survives server crashes

### ACID Trade-offs

**Benefits:**
- Data integrity guaranteed
- Predictable behavior
- Simpler application code (database handles concurrency)

**Costs:**
- Reduced throughput (transactions wait for locks/validation)
- Limited horizontal scalability (coordinating across servers is complex)
- Increased latency (WAL writes, lock acquisition)

**When to relax ACID:**
- Social media likes/views (approximate counts acceptable)
- Analytics pipelines (eventual consistency fine)
- Caching layers (stale data temporarily acceptable)
- Logging systems (losing a few log entries acceptable)

This is why NoSQL databases often trade some ACID properties for performance and scalability—the trade-off makes sense for certain applications.

---

## Database Normalization vs Denormalization

Normalization and denormalization are opposing design strategies for organizing relational database tables. Understanding both—and when to use each—is crucial for effective database design.

### What is Normalization?

**Normalization** is the process of organizing data to reduce redundancy (duplicate information) and improve data integrity (correctness and consistency).

**Layman explanation:** Imagine keeping customer addresses. Instead of writing "John Smith, 123 Main St, Boston, MA" on every order John places, you write it once in a customers table and just reference "Customer #42" on each order. This saves space and ensures if John moves, you update one place—not 100 different orders.

**Goals of normalization:**
1. Eliminate redundant data (same information stored multiple times)
2. Ensure data dependencies make logical sense
3. Make updates, inserts, and deletes easier and safer
4. Reduce storage requirements

### The Normal Forms (1NF through BCNF)

Normal forms are progressive levels of normalization, each building on the previous one.

### First Normal Form (1NF): Atomic Values

**Rule:** Each column must contain atomic (indivisible) values, and each column must contain values of a single type.

**Violation example:**
```
┌──────────┬─────────────────────────────┬─────────────┐
│ OrderID  │      Products               │    Total    │
├──────────┼─────────────────────────────┼─────────────┤
│   101    │  Laptop, Mouse, Keyboard    │   $1,299    │
│   102    │  Monitor                    │    $399     │
└──────────┴─────────────────────────────┴─────────────┘
```

**Problem:** The "Products" column contains multiple values (a list). You can't easily query "all orders containing Laptop."

**1NF compliant:**
```
┌──────────┬─────────────┬─────────────┐
│ OrderID  │   Product   │    Price    │
├──────────┼─────────────┼─────────────┤
│   101    │   Laptop    │    $999     │
│   101    │   Mouse     │     $49     │
│   101    │   Keyboard  │    $129     │
│   102    │   Monitor   │    $399     │
└──────────┴─────────────┴─────────────┘
```

**Now:** Each cell contains exactly one value. You can easily find all orders with Laptops.

### Second Normal Form (2NF): No Partial Dependencies

**Rule:** Must be in 1NF, AND all non-key columns must depend on the entire primary key (not just part of it).

**Layman explanation:** If your table uses a compound key (multiple columns together as the unique identifier), every other column should depend on ALL parts of that key, not just some parts.

**Violation example:**
```
┌──────────┬─────────────┬──────────────┬────────────────┐
│ OrderID  │  ProductID  │  ProductName │    Quantity    │
├──────────┼─────────────┼──────────────┼────────────────┤
│   101    │     5       │    Laptop    │       1        │
│   101    │     8       │    Mouse     │       2        │
│   102    │     5       │    Laptop    │       1        │
└──────────┴─────────────┴──────────────┴────────────────┘

Primary Key: (OrderID, ProductID) — combination uniquely identifies each row
```

**Problem:** "ProductName" depends only on "ProductID", not on the full key (OrderID + ProductID). If Product #5 changes name from "Laptop" to "Notebook", you'd have to update multiple rows.

**2NF compliant (split into two tables):**

```
Orders Table:
┌──────────┬─────────────┬────────────┐
│ OrderID  │  ProductID  │  Quantity  │
├──────────┼─────────────┼────────────┤
│   101    │      5      │     1      │
│   101    │      8      │     2      │
│   102    │      5      │     1      │
└──────────┴─────────────┴────────────┘

Products Table:
┌─────────────┬──────────────┐
│  ProductID  │  ProductName │
├─────────────┼──────────────┤
│      5      │    Laptop    │
│      8      │    Mouse     │
└─────────────┴──────────────┘
```

**Now:** ProductName is stored once per product. Update the product name in one place.

### Third Normal Form (3NF): No Transitive Dependencies

**Rule:** Must be in 2NF, AND no non-key column should depend on another non-key column.

**Layman explanation:** Information about one thing shouldn't be determined by information about another thing (unless that other thing is the key). Each table should be about one specific entity.

**Violation example:**
```
┌───────────┬──────────────┬─────────────┬─────────────────┐
│ EmployeeID│     Name     │  Department │  DeptLocation   │
├───────────┼──────────────┼─────────────┼─────────────────┤
│    1      │   Alice      │     IT      │   Building A    │
│    2      │   Bob        │     IT      │   Building A    │
│    3      │   Carol      │     HR      │   Building B    │
└───────────┴──────────────┴─────────────┴─────────────────┘
```

**Problem:** "DeptLocation" depends on "Department", not directly on "EmployeeID". If IT moves to Building C, you update multiple employee records.

**3NF compliant (split into two tables):**

```
Employees Table:
┌───────────┬──────────────┬─────────────┐
│ EmployeeID│     Name     │  Department │
├───────────┼──────────────┼─────────────┤
│    1      │   Alice      │     IT      │
│    2      │   Bob        │     IT      │
│    3      │   Carol      │     HR      │
└───────────┴──────────────┴─────────────┘

Departments Table:
┌─────────────┬─────────────────┐
│  Department │   DeptLocation  │
├─────────────┼─────────────────┤
│     IT      │   Building A    │
│     HR      │   Building B    │
└─────────────┴─────────────────┘
```

**Now:** Department location is stored once. If IT moves, update one row.

### Boyce-Codd Normal Form (BCNF): Stronger 3NF

**Rule:** Must be in 3NF, AND for every functional dependency X → Y, X must be a candidate key (a potential unique identifier).

**Layman explanation:** BCNF is a stricter version of 3NF that handles certain edge cases. In practice, most properly designed 3NF databases are already in BCNF.

**Violation example:**
```
┌──────────┬─────────────┬─────────────┐
│ StudentID│  Professor  │   Subject   │
├──────────┼─────────────┼─────────────┤
│   101    │   Dr. Lee   │   Physics   │
│   101    │   Dr. Shah  │   Math      │
│   102    │   Dr. Lee   │   Physics   │
└──────────┴─────────────┴─────────────┘

Constraint: Each professor teaches only one subject
```

**Problem:** "Professor → Subject" (knowing the professor tells you the subject), but Professor is not a candidate key of the table. This creates update anomalies.

**BCNF compliant:**

```
Student_Enrollment:
┌──────────┬─────────────┐
│ StudentID│  Professor  │
├──────────┼─────────────┤
│   101    │   Dr. Lee   │
│   101    │   Dr. Shah  │
│   102    │   Dr. Lee   │
└──────────┴─────────────┘

Professor_Subject:
┌─────────────┬─────────┐
│  Professor  │ Subject │
├─────────────┼─────────┤
│   Dr. Lee   │ Physics │
│   Dr. Shah  │  Math   │
└─────────────┴─────────┘
```

### Benefits of Normalization

1. **Data integrity:** Update one place, not many
2. **Storage efficiency:** No duplicate data
3. **Consistency:** Impossible to have contradictory information
4. **Easier maintenance:** Clear structure, logical organization

### When to Denormalize

**Denormalization** is the intentional introduction of redundancy to improve query performance.

**Why denormalize?**

While normalized databases are clean and consistent, they often require joining multiple tables to answer simple questions. Joins can be slow, especially with large datasets.

**Example:** Getting an order with customer details

**Normalized (3 tables, 2 joins):**
```
Query: Get order #101 with customer name and address

Orders → Join Customers (by CustomerID) → Join Addresses (by AddressID)
```

**Denormalized (1 table, no joins):**
```
┌─────┬────────┬────────────┬─────────────┬──────────┐
│Order│Customer│   Name     │   Address   │  Total   │
├─────┼────────┼────────────┼─────────────┼──────────┤
│ 101 │  42    │ John Smith │ 123 Main St │  $500    │
└─────┴────────┴────────────┴─────────────┴──────────┘

Query: SELECT * FROM Orders WHERE OrderID = 101
(instant, no joins needed)
```

**Trade-off:** Faster reads, but if John moves, you must update ALL his orders.

### Denormalization Patterns

**1. Duplicate frequently accessed data**

Store commonly-needed fields directly in the main table:
```
Orders table includes: CustomerName, CustomerEmail
(instead of joining to Customers table every time)
```

**2. Pre-calculate aggregates**

Store computed values to avoid expensive calculations:
```
ProductsTable includes: AverageRating, TotalReviews
(updated when new reviews come in, instead of calculating on every query)
```

**3. Store historical snapshots**

Keep point-in-time data even if source changes:
```
OrderItems includes: ProductPriceAtPurchase
(price customer actually paid, even if current price changed)
```

**4. Flatten hierarchies**

Combine parent-child data in one table:
```
Products includes: CategoryName, CategoryDescription
(instead of joining to Categories table)
```

### Trade-offs: Storage vs Query Complexity

```
┌────────────────────────────────────────────────────────────┐
│              NORMALIZATION VS DENORMALIZATION              │
├──────────────────────┬─────────────────┬───────────────────┤
│                      │   Normalized    │  Denormalized     │
├──────────────────────┼─────────────────┼───────────────────┤
│ Data Redundancy      │      Low        │      High         │
├──────────────────────┼─────────────────┼───────────────────┤
│ Storage Requirements │      Low        │      High         │
├──────────────────────┼─────────────────┼───────────────────┤
│ Read Performance     │   Slower        │     Faster        │
│                      │ (many joins)    │  (fewer joins)    │
├──────────────────────┼─────────────────┼───────────────────┤
│ Write Performance    │    Faster       │     Slower        │
│                      │ (update 1 row)  │ (update many)     │
├──────────────────────┼─────────────────┼───────────────────┤
│ Data Consistency     │  Guaranteed     │   Must manage     │
│                      │                 │   manually        │
├──────────────────────┼─────────────────┼───────────────────┤
│ Complexity           │  Schema complex │  Queries simple   │
│                      │  Queries complex│  Schema complex   │
├──────────────────────┼─────────────────┼───────────────────┤
│ Best For             │ OLTP systems    │  OLAP systems     │
│                      │ (many writes)   │ (analytics, BI)   │
└──────────────────────┴─────────────────┴───────────────────┘
```

**OLTP (Online Transaction Processing):** Systems handling many small transactions (e-commerce, banking). Favor normalization.

**OLAP (Online Analytical Processing):** Systems for analysis and reporting (data warehouses, business intelligence). Often use denormalized "star schemas" for fast queries.

### Practical Decision Framework

**Normalize when:**
- Data integrity is critical (financial, medical systems)
- Many write operations (frequently updated data)
- Storage costs are high
- Data warehouse source systems

**Denormalize when:**
- Read performance is critical (dashboards, public APIs)
- Joins are killing query performance
- Data is relatively static (product catalogs)
- Reporting and analytics databases

**Hybrid approach (common in 2026):**
- Normalized operational database (source of truth)
- Denormalized read replicas or caching layer (fast queries)
- Sync changes from normalized to denormalized periodically

This gives you both data integrity and performance where you need it.

---

## Indexing Strategies

Database indexes are data structures that improve the speed of data retrieval operations. Think of them like the index at the back of a textbook—instead of reading every page to find "photosynthesis," you check the index and jump directly to page 247.

**Without an index:** Database scans every row (called a "table scan" or "full scan")
**With an index:** Database jumps directly to matching rows

### How Indexes Work (Conceptual Overview)

```
┌────────────────────────────────────────────────────────────┐
│                WITHOUT INDEX (Table Scan)                  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Query: Find user with email = "alice@example.com"        │
│                                                            │
│  Users Table (1 million rows):                            │
│  Row 1: Check email... not alice                          │
│  Row 2: Check email... not alice                          │
│  Row 3: Check email... not alice                          │
│  ...                                                       │
│  Row 247,891: Check email... FOUND alice! ✓               │
│  ...continue checking remaining 752,109 rows...           │
│                                                            │
│  Time: ~2 seconds (scanned all 1M rows)                   │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                  WITH INDEX (Index Lookup)                 │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Query: Find user with email = "alice@example.com"        │
│                                                            │
│  Index on email column (sorted, searchable):              │
│  Binary search through index... found in 20 comparisons   │
│  Jump directly to Row 247,891                             │
│                                                            │
│  Time: ~0.002 seconds (1000x faster!)                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Trade-off:** Indexes speed up reads but slow down writes (INSERT, UPDATE, DELETE) because the index must be updated too. Also, indexes consume storage space.

### B-tree Indexes (Most Common)

**B-tree** (Balanced Tree) indexes are the default in most databases. They're optimized for range queries and sorted access.

**Structure visualization:**
```
                    [50]
                   /    \
                  /      \
              [25]        [75]
             /    \      /    \
           [10]  [35]  [60]  [90]
          /  \   / \   / \   / \
        [5][15][30][40][55][65][80][95]

Each box contains: pointer to rows with that value
```

**How B-tree search works:**

Finding value 65:
1. Start at root: 65 > 50, go right
2. At node: 65 < 75, go left
3. At node: 65 > 60, go right
4. Found! (took 3 comparisons for millions of rows)

**Best for:**
- Equality searches: `WHERE age = 25`
- Range queries: `WHERE age BETWEEN 18 AND 65`
- Sorted results: `ORDER BY created_at DESC`
- Prefix matching: `WHERE name LIKE 'John%'`

**Not good for:**
- Suffix matching: `WHERE name LIKE '%son'` (can't use index efficiently)
- Negations: `WHERE status != 'active'` (might be faster to scan)

### Hash Indexes (Exact Matches Only)

**Hash indexes** use a hash function to create a lookup table. Extremely fast for equality checks but cannot handle ranges.

**How it works:**
```
Hash Function: email → hash value → bucket location

"alice@example.com" → hash → 42891 → points to Row 247,891
"bob@example.com"   → hash → 17234 → points to Row 89,234

Query: WHERE email = 'alice@example.com'
→ Hash 'alice@example.com' → 42891 → instant lookup ✓
```

**Best for:**
- Exact equality: `WHERE id = 12345`
- Key-value lookups in memory databases (Redis)

**Cannot do:**
- Range queries: `WHERE id > 1000` (no ordering in hash table)
- Sorting: `ORDER BY id` (hash values aren't sorted)
- Prefix matching: `WHERE email LIKE 'alice%'`

**In practice:** Hash indexes are less common in disk-based SQL databases. B-tree indexes handle equality well enough and offer more flexibility. Hash indexes shine in in-memory databases.

### Composite Indexes (Multiple Columns)

**Composite indexes** (also called multi-column or concatenated indexes) index multiple columns together. Column ordering matters significantly.

**Example:** Index on (last_name, first_name, age)

```
Index Structure (conceptually sorted):
┌────────────┬────────────┬─────┬──────────┐
│ last_name  │ first_name │ age │  row_ptr │
├────────────┼────────────┼─────┼──────────┤
│ Johnson    │ Alice      │  28 │   →row1  │
│ Johnson    │ Alice      │  35 │   →row2  │
│ Johnson    │ Bob        │  42 │   →row3  │
│ Smith      │ Alice      │  29 │   →row4  │
│ Smith      │ Carol      │  31 │   →row5  │
└────────────┴────────────┴─────┴──────────┘
```

**The Left-Prefix Rule (Critical!):**

The index (last_name, first_name, age) can efficiently support:

✓ `WHERE last_name = 'Smith'`
✓ `WHERE last_name = 'Smith' AND first_name = 'Alice'`
✓ `WHERE last_name = 'Smith' AND first_name = 'Alice' AND age = 29`
✓ `WHERE last_name = 'Smith' AND age = 29` (can use index partially)

✗ `WHERE first_name = 'Alice'` (skips left-most column, can't use index)
✗ `WHERE age = 29` (skips left-most columns, can't use index)
✗ `WHERE first_name = 'Alice' AND age = 29` (skips last_name, can't use index)

**Layman explanation:** Think of a phone book sorted by last name, then first name. You can quickly find "Smith, Alice" but you cannot quickly find "everyone named Alice" (you'd have to scan every last name). The sorting only works from left to right.

**Column ordering strategy:**

Put columns in this order:
1. Columns used in WHERE clauses (most selective first)
2. Columns used in JOIN conditions
3. Columns used in ORDER BY

**Example decision:**

Query: `WHERE country = 'USA' AND city = 'Boston' AND zip = '02101'`

Analysis:
- country: ~200 distinct values (low selectivity)
- city: ~20,000 distinct values (medium selectivity)
- zip: ~40,000 distinct values (high selectivity)

**Best index:** (zip, city, country)

Why? Start with most selective filter (zip narrows down fastest), then city, then country.

### Covering Indexes (Include All Needed Columns)

A **covering index** contains all columns needed to satisfy a query—the database never needs to access the actual table.

**Example:**

Query: `SELECT first_name, last_name, email FROM users WHERE last_name = 'Smith'`

**Non-covering index:** Index on (last_name)
1. Use index to find matching row IDs
2. Go to actual table rows to get first_name and email
3. Two I/O operations (index + table)

**Covering index:** Index on (last_name, first_name, email)
1. Index contains all needed data
2. Return results directly from index
3. One I/O operation (index only)

**Performance gain:** 2-5x faster queries by avoiding table lookups

**Trade-off:** Larger indexes (storing more columns), slower writes (more data to update)

**Use when:**
- Query is run frequently
- Query accesses only a few columns
- Performance gain justifies larger index size

### Partial Indexes (Index Subset of Rows)

**Partial indexes** (also called filtered indexes) only index rows that meet certain criteria.

**Example:**

Table: orders (10 million rows)
- 9.5 million with status = 'completed'
- 500,000 with status = 'pending'

Frequent query: `WHERE status = 'pending'`

**Full index:** Index all 10M rows
**Partial index:** Index only pending rows

```
CREATE INDEX idx_pending_orders
ON orders(order_id)
WHERE status = 'pending'
```

**Benefits:**
- 95% smaller index (500K rows vs 10M rows)
- Faster queries on pending orders
- Faster writes (most INSERTs are 'completed', skip index)
- Less storage and memory

**Use when:**
- Queries frequently filter on same condition
- That condition selects small percentage of rows
- Common pattern: `WHERE deleted_at IS NULL` (only index active records)

### Expression Indexes (Index Computed Values)

**Expression indexes** (also called functional indexes) index the result of an expression or function, not just column values.

**Problem:** Query uses function on column

```
Query: WHERE LOWER(email) = 'alice@example.com'

Regular index on email won't help because the query transforms email before comparing
```

**Solution:** Index the transformed value

```
CREATE INDEX idx_email_lower
ON users(LOWER(email))

Now the query can use the index efficiently!
```

**Other common use cases:**

- Date parts: `INDEX ON orders(EXTRACT(YEAR FROM created_at))`
- Concatenation: `INDEX ON users(first_name || ' ' || last_name)`
- JSON fields: `INDEX ON data((properties->>'category'))`
- Calculations: `INDEX ON products(price * quantity)`

**Trade-off:** Index updates whenever underlying columns change, and uses more CPU for writes.

### Full-Text Indexes (Text Search)

**Full-text indexes** enable fast searching within text content—finding documents that contain certain words or phrases.

**Use cases:**
- Search articles by keywords
- Find products by description
- Search support tickets by content

**How it differs from LIKE:**

```
LIKE query (slow on large text):
WHERE article_text LIKE '%machine learning%'
→ Scans every row, searches entire text field

Full-text index (fast):
WHERE article_text @@ 'machine learning'
→ Pre-indexed word positions, instant lookup
```

**Features:**
- **Stemming:** "running" matches "run", "runs", "ran"
- **Stop words:** Ignores common words like "the", "and", "is"
- **Relevance ranking:** Returns best matches first
- **Phrase search:** Find exact phrases "machine learning"
- **Boolean operators:** "machine AND learning NOT deep"

**Trade-off:** Large indexes (words, positions, metadata), complex maintenance, slower writes.

**Modern alternative (2026):** Vector embeddings with approximate nearest neighbor (ANN) indexes for semantic search (finding similar meaning, not just exact words).

### Spatial Indexes (Geographic Data)

**Spatial indexes** optimize queries on geographic coordinates, shapes, and spatial relationships.

**Structure types:**

**R-tree indexes:** Group nearby geographic objects into hierarchical bounding boxes

```
World Map divided into boxes:

[North America]
    ├─ [USA]
    │    ├─ [West Coast] (contains LA, SF, Seattle)
    │    └─ [East Coast] (contains NYC, Boston)
    └─ [Canada]

Query: Find restaurants within 5 miles of (lat, long)
→ Check bounding boxes first
→ Only calculate distance for nearby candidates
```

**GiST (Generalized Search Tree):** Flexible framework supporting various spatial types (PostgreSQL's approach)

**Use cases:**
- Find nearby locations: "restaurants within 5 miles"
- Geographic containment: "which postal code contains this address?"
- Route optimization: "shortest path between points"
- Geofencing: "alert when device enters region"

**Query example:**

```
Without spatial index:
Calculate distance to EVERY point in database (millions of haversine calculations)

With spatial index:
1. Find bounding box containing search area
2. Calculate distance only for candidates in that box
→ 1000x faster
```

### Index Selection Strategy

**How to decide which indexes to create:**

**1. Analyze your queries (not your tables)**

Don't guess—measure:
- What queries run most frequently?
- Which queries are slowest?
- Check slow query logs
- Monitor production database metrics

**2. Index selection priorities:**

```
High Priority:
  ✓ Primary keys (usually auto-indexed)
  ✓ Foreign keys (JOIN performance)
  ✓ Columns in WHERE clauses of frequent queries
  ✓ Columns in ORDER BY for large result sets

Medium Priority:
  ✓ Columns in GROUP BY
  ✓ Columns in HAVING clauses
  ✓ Composite indexes for common multi-column filters

Low Priority:
  ✓ Rarely-queried columns
  ✓ Columns with low selectivity (few distinct values)
  ✓ Columns that change frequently
```

**3. Avoid over-indexing**

Each index has costs:
- **Storage:** Indexes consume disk space (sometimes more than the data itself)
- **Write performance:** Every INSERT/UPDATE/DELETE must update all indexes
- **Memory:** Databases cache hot indexes in RAM; too many indexes = less effective caching

**Rule of thumb:** Most tables need 3-5 indexes. If you have 15+ indexes on one table, you're probably over-indexed.

**4. Monitor index usage**

Databases track index usage statistics:
- Which indexes are actually used?
- Which indexes are never touched?

**Drop unused indexes** to improve write performance and reduce storage.

### Index Maintenance Best Practices

**1. Rebuild fragmented indexes**

Over time, indexes become fragmented (disorganized), reducing performance. Periodic rebuilding reorganizes them.

**2. Update statistics**

Databases use statistics (data distribution, cardinality estimates) to choose optimal query plans. Keep statistics current, especially after bulk data changes.

**3. Monitor index size**

Indexes growing much larger than expected may indicate:
- Too many columns in composite index
- Unnecessary expression indexes
- Bloat from deleted rows (vacuum/analyze needed)

**4. Test before production**

Before creating indexes on production:
- Test on copy of production data
- Measure actual performance improvement
- Consider index creation time (locks table in some databases)
- Use `CREATE INDEX CONCURRENTLY` in PostgreSQL to avoid blocking writes

### Quick Reference: Index Type Selection

```
┌────────────────────────────────────────────────────────────┐
│                  WHEN TO USE EACH INDEX TYPE               │
├───────────────┬────────────────────────────────────────────┤
│  Index Type   │              Best Use Case                 │
├───────────────┼────────────────────────────────────────────┤
│  B-tree       │ Default choice for most queries           │
│               │ Range queries, sorting, general purpose    │
├───────────────┼────────────────────────────────────────────┤
│  Hash         │ Exact equality matches only                │
│               │ In-memory databases, key-value lookups     │
├───────────────┼────────────────────────────────────────────┤
│  Composite    │ Queries filtering on multiple columns      │
│               │ Remember left-prefix rule!                 │
├───────────────┼────────────────────────────────────────────┤
│  Covering     │ Frequently-run queries needing few columns │
│               │ Performance-critical read operations       │
├───────────────┼────────────────────────────────────────────┤
│  Partial      │ Queries always filtering on same condition │
│               │ Condition selects small % of rows          │
├───────────────┼────────────────────────────────────────────┤
│  Expression   │ Queries using functions/calculations       │
│               │ WHERE LOWER(col), WHERE YEAR(date), etc.   │
├───────────────┼────────────────────────────────────────────┤
│  Full-text    │ Searching within text content              │
│               │ Articles, descriptions, documents          │
├───────────────┼────────────────────────────────────────────┤
│  Spatial      │ Geographic queries                         │
│  (R-tree/GiST)│ "Find nearby", containment, routing       │
└───────────────┴────────────────────────────────────────────┘
```

---

## Query Optimization

Query optimization is the art and science of making database queries run faster. Even with proper indexes, poorly-written queries can bring a database to its knees.

### EXPLAIN Plan Analysis

The **EXPLAIN** command shows you how the database will execute your query—the "query plan" or "execution plan." This is your most powerful tool for understanding query performance.

**Layman explanation:** EXPLAIN is like asking for directions before a trip. Instead of just driving and hoping, you see the route beforehand, identify traffic jams, and choose a better path.

**Basic usage:**

```
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com'
```

**Output (simplified):**
```
Seq Scan on users  (cost=0.00..18334.00 rows=1 width=200)
  Filter: (email = 'alice@example.com')
```

**Reading the plan:**

- **Seq Scan** (Sequential Scan): Reading every row in the table (BAD for large tables)
- **cost=0.00..18334.00**: Estimated cost (arbitrary units, higher = slower)
- **rows=1**: Estimated rows returned
- **width=200**: Average row size in bytes

**Better plan with index:**
```
Index Scan using idx_users_email on users
  (cost=0.42..8.44 rows=1 width=200)
  Index Cond: (email = 'alice@example.com')
```

- **Index Scan**: Using index (GOOD!)
- **cost=0.42..8.44**: Much lower cost (faster)

### Common Query Plan Operations

**Table Scan (Sequential Scan):**
```
Seq Scan on orders
```
- Reads every row in table
- **When acceptable:** Small tables, or query needs most rows anyway
- **When problematic:** Large tables, query needs few rows

**Index Scan:**
```
Index Scan using idx_order_date on orders
```
- Uses index to find specific rows
- **Good:** Direct lookup, efficient

**Index Only Scan:**
```
Index Only Scan using idx_order_date_total on orders
```
- All needed columns are in index (covering index)
- **Excellent:** No table access needed

**Bitmap Index Scan:**
```
Bitmap Heap Scan on orders
  -> Bitmap Index Scan on idx_status
```
- Combines multiple indexes or large result sets
- **Good:** Efficient for moderate selectivity

**Nested Loop:**
```
Nested Loop
  -> Seq Scan on customers
  -> Index Scan using idx_orders_customer on orders
```
- For each row in outer table, look up matching rows in inner table
- **Good for:** Small outer table, indexed inner table
- **Bad for:** Large outer table (inner lookup repeats many times)

**Hash Join:**
```
Hash Join
  -> Seq Scan on customers
  -> Hash
    -> Seq Scan on orders
```
- Build hash table from one table, probe with other
- **Good for:** Medium to large tables with equality joins
- **Bad for:** Memory constraints (hash table must fit in RAM)

**Merge Join:**
```
Merge Join
  -> Index Scan on customers (ordered)
  -> Index Scan on orders (ordered)
```
- Both inputs sorted on join key, merge together
- **Good for:** Both sides pre-sorted or have indexes
- **Best for:** Large tables with existing sort order

### EXPLAIN ANALYZE (Actual Execution)

`EXPLAIN ANALYZE` actually runs the query and reports real timing:

```
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending'
```

**Output:**
```
Seq Scan on orders (cost=0.00..18334.00 rows=1000 width=200)
                   (actual time=0.045..156.789 rows=1247 loops=1)
  Filter: (status = 'pending')
  Rows Removed by Filter: 998753
Planning Time: 0.234 ms
Execution Time: 157.123 ms
```

**Key metrics:**
- **actual time:** Real time in milliseconds
- **rows:** Actual rows (compare to estimate—big differences indicate stale statistics)
- **Rows Removed by Filter:** Rows examined but discarded (high = inefficient)
- **Execution Time:** Total query time

**Warning:** `EXPLAIN ANALYZE` runs the query for real. Be careful with:
- Queries that modify data (INSERT/UPDATE/DELETE)
- Expensive operations on production databases

### Index Selection (Database's Choice)

The database's **query optimizer** chooses which index to use (if any). Sometimes it makes surprising choices.

**Factors influencing index selection:**

1. **Selectivity:** How many rows the query returns
   - High selectivity (few rows): Use index
   - Low selectivity (many rows): Table scan may be faster

**Example:**
```
Table: 1 million users

Query 1: WHERE country = 'Vatican'  (returns 10 rows)
→ Use index (high selectivity)

Query 2: WHERE country = 'USA'  (returns 350,000 rows)
→ Table scan may be faster (low selectivity)
→ Why? Reading 35% of table randomly via index
   is slower than reading entire table sequentially
```

2. **Index size vs table size**
   - Small indexes in memory: Prefer index
   - Large indexes on disk: May prefer table scan

3. **Statistics accuracy**
   - Database estimates based on statistics
   - Outdated statistics = poor decisions
   - **Solution:** Regularly run ANALYZE/UPDATE STATISTICS

4. **Query predicates**
   - `=` (equality): Strong index usage
   - `<`, `>`, `BETWEEN` (range): Good index usage
   - `!=`, `NOT IN`: Often can't use index efficiently
   - `LIKE 'prefix%'`: Can use index
   - `LIKE '%suffix'`: Cannot use B-tree index

### Query Rewriting for Performance

Small changes to how you write queries can dramatically impact performance.

**1. Use specific columns instead of SELECT ***

```
Bad:  SELECT * FROM users WHERE id = 123
Good: SELECT id, name, email FROM users WHERE id = 123
```

**Why better:**
- Smaller data transfer
- May enable covering index
- Clearer intent, easier maintenance

**2. Avoid functions on indexed columns in WHERE**

```
Bad:  WHERE YEAR(created_at) = 2024
Good: WHERE created_at >= '2024-01-01'
      AND created_at < '2025-01-01'
```

**Why:** First query can't use index on created_at (function transforms the column). Second query can use the index directly.

**3. Use EXISTS instead of COUNT when checking existence**

```
Bad:  SELECT COUNT(*) FROM orders WHERE customer_id = 123
      (then check if count > 0)

Good: SELECT EXISTS(SELECT 1 FROM orders WHERE customer_id = 123)
```

**Why:** COUNT must count all rows. EXISTS stops at first match.

**4. Avoid OR conditions across different columns**

```
Bad:  WHERE first_name = 'John' OR last_name = 'Smith'

Good: Split into two queries with UNION:
      SELECT * FROM users WHERE first_name = 'John'
      UNION
      SELECT * FROM users WHERE last_name = 'Smith'
```

**Why:** OR across columns can't use indexes efficiently. UNION allows each query to use its own index.

**Exception:** OR on same column is fine: `WHERE status = 'pending' OR status = 'active'`

**5. Push filtering to subqueries**

```
Bad:  SELECT * FROM orders o
      JOIN customers c ON o.customer_id = c.id
      WHERE o.status = 'pending'

Good: SELECT * FROM
      (SELECT * FROM orders WHERE status = 'pending') o
      JOIN customers c ON o.customer_id = c.id
```

**Why:** Filter before joining reduces rows processed in join operation.

### Join Optimization

Joins are often the most expensive part of queries. Optimization strategies:

**1. Join order matters**

```
Join small tables first, then larger tables:

Bad:  FROM large_table (1M rows)
      JOIN small_table (100 rows)

Good: FROM small_table (100 rows)
      JOIN large_table (1M rows)
```

**Why:** Smaller intermediate result sets mean less work for subsequent joins.

**Note:** Modern optimizers often reorder joins automatically, but understanding this principle helps you structure queries.

**2. Ensure join columns are indexed**

```
Query: SELECT * FROM orders o
       JOIN customers c ON o.customer_id = c.id

Required indexes:
- orders.customer_id (foreign key index)
- customers.id (primary key, usually auto-indexed)
```

**Without indexes:** Nested loop with table scans = catastrophic performance

**3. Use appropriate join types**

**INNER JOIN:** Only matching rows from both tables
```
SELECT * FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
```
Most efficient when you want only matches.

**LEFT JOIN:** All rows from left table + matching from right
```
SELECT * FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
```
More expensive than INNER (must check all left rows even without matches).

**Optimization:** If you don't actually need non-matching rows, use INNER JOIN instead.

**4. Avoid unnecessary joins**

```
Bad:  SELECT o.id, o.total, c.name
      FROM orders o
      JOIN customers c ON o.customer_id = c.id
      WHERE o.id = 123

Good: SELECT o.id, o.total, o.customer_id
      FROM orders o
      WHERE o.id = 123
      -- Fetch customer.name separately if really needed
```

Only join when you actually need columns from both tables.

### Avoiding N+1 Queries

The **N+1 query problem** is one of the most common performance killers in application code.

**The problem:**

```
Application code:
1. Query: Get all customers (1 query)
2. For each customer, query their orders (N queries, where N = number of customers)

Total: 1 + N queries

Example with 100 customers:
  1 query to get customers
  + 100 queries to get each customer's orders
  = 101 database round trips
```

**Visualization:**
```
┌──────────────────────────────────────────────┐
│  N+1 QUERY PATTERN (SLOW)                    │
├──────────────────────────────────────────────┤
│                                              │
│  App → DB: Get all customers                 │
│  DB → App: [Customer 1, Customer 2, ...]     │
│                                              │
│  App → DB: Get orders for Customer 1         │
│  DB → App: [Order A, Order B]                │
│                                              │
│  App → DB: Get orders for Customer 2         │
│  DB → App: [Order C]                         │
│                                              │
│  App → DB: Get orders for Customer 3         │
│  DB → App: [Order D, Order E, Order F]       │
│                                              │
│  ... (97 more round trips) ...               │
│                                              │
│  Total: 101 queries, 101 network round trips │
│                                              │
└──────────────────────────────────────────────┘
```

**The solution: Eager loading (fetch all at once)**

```
┌──────────────────────────────────────────────┐
│  EAGER LOADING (FAST)                        │
├──────────────────────────────────────────────┤
│                                              │
│  App → DB: Get customers WITH their orders   │
│            (JOIN or IN clause)               │
│                                              │
│  DB → App: All customers + all orders        │
│                                              │
│  Total: 1 query, 1 network round trip        │
│  (100x faster!)                              │
│                                              │
└──────────────────────────────────────────────┘
```

**Implementation options:**

**Option 1: JOIN query**
```
SELECT c.*, o.*
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
```

Returns all data in one query. Application must de-duplicate customers (each customer repeats for each order).

**Option 2: IN clause query**
```
-- First query
SELECT * FROM customers

-- Second query (gets ALL orders at once)
SELECT * FROM orders
WHERE customer_id IN (1, 2, 3, ..., 100)
```

Two queries total (instead of 101). Application groups orders by customer_id.

**Modern ORMs** (Object-Relational Mappers) often handle this automatically:
- Rails: `.includes(:orders)`
- Django: `.prefetch_related('orders')`
- SQLAlchemy: `.options(joinedload('orders'))`

**When to use each:**

- **1:1 or 1:few relationships:** Use JOIN (customers → primary address)
- **1:many with lots of children:** Use IN clause (customers → potentially hundreds of orders)
- **Deep nesting:** Combination approach

**Red flags in application code:**
- Loop executing queries
- Query inside a loop
- ORM loading related objects inside iteration

### Pagination Strategies

Pagination displays data in pages (10, 50, 100 items at a time). Two common approaches with very different performance characteristics.

### Offset-Based Pagination (OFFSET/LIMIT)

**How it works:**
```
Page 1: LIMIT 10 OFFSET 0     (rows 1-10)
Page 2: LIMIT 10 OFFSET 10    (rows 11-20)
Page 3: LIMIT 10 OFFSET 20    (rows 21-30)
...
Page 100: LIMIT 10 OFFSET 990 (rows 991-1000)
```

**The problem: Deep pagination is slow**

```
┌────────────────────────────────────────────────┐
│      HOW OFFSET WORKS (INEFFICIENT)            │
├────────────────────────────────────────────────┤
│                                                │
│  Query: SELECT * FROM posts                    │
│         ORDER BY created_at DESC               │
│         LIMIT 10 OFFSET 990                    │
│                                                │
│  Database process:                             │
│  1. Scan and sort 1000 rows                    │
│  2. Skip first 990 rows (discarded!)           │
│  3. Return next 10 rows                        │
│                                                │
│  Page 100 processes 1000 rows to return 10!    │
│                                                │
└────────────────────────────────────────────────┘
```

**Performance characteristics:**
- Page 1: Fast (OFFSET 0)
- Page 10: Slower (OFFSET 90)
- Page 100: Much slower (OFFSET 990)
- Page 10,000: Unusably slow (OFFSET 99,990 - processes 100K rows!)

**When OFFSET/LIMIT is acceptable:**
- Small datasets (thousands of rows, not millions)
- Users rarely go past page 3-5
- Random access needed ("jump to page 50")

### Cursor-Based Pagination (Keyset Pagination)

**How it works:** Instead of counting rows, use the last seen value as a starting point.

```
Page 1:
  SELECT * FROM posts
  ORDER BY created_at DESC, id DESC
  LIMIT 10

  Last item: created_at = '2024-01-15 10:30', id = 12345

Page 2:
  SELECT * FROM posts
  WHERE (created_at, id) < ('2024-01-15 10:30', 12345)
  ORDER BY created_at DESC, id DESC
  LIMIT 10
```

**Visualization:**
```
┌────────────────────────────────────────────────┐
│      HOW CURSOR WORKS (EFFICIENT)              │
├────────────────────────────────────────────────┤
│                                                │
│  Cursor: "Start after created_at = X, id = Y"  │
│                                                │
│  Database process:                             │
│  1. Use index to jump directly to position X,Y │
│  2. Return next 10 rows                        │
│  3. No scanning/counting required!             │
│                                                │
│  SAME performance for page 1, 100, or 10,000!  │
│                                                │
└────────────────────────────────────────────────┘
```

**Performance characteristics:**
- Consistent speed regardless of page depth
- Can handle billions of rows efficiently
- Requires appropriate index on cursor columns

**Trade-offs:**

**Advantages:**
- Constant-time performance
- Handles real-time data (new items don't break pagination)
- Scales to any dataset size

**Disadvantages:**
- No "jump to page 50" (can only go next/previous)
- Slightly more complex to implement
- Needs unique, sequential cursor (timestamp + id)
- Can't show total page count

**When to use cursor pagination:**
- Infinite scroll UI (social media feeds)
- Large datasets (millions+ rows)
- Real-time data (items being added/removed frequently)
- Mobile apps (next/previous navigation)

**When to use offset pagination:**
- Small datasets (<10K rows)
- Need page numbers/total pages
- Random access required
- Traditional UI with page number links

**Hybrid approach (common in 2026):**
- Cursor pagination for feeds/lists
- Cached total counts (updated periodically, not real-time)
- Show "Load more" instead of page numbers

### Query Optimization Checklist

**Before optimizing:**
1. ✓ Measure: Profile actual production queries
2. ✓ Identify: Find the slowest queries (80/20 rule applies)
3. ✓ Set goals: Define acceptable response times

**Optimization steps:**
1. ✓ Run EXPLAIN ANALYZE on slow query
2. ✓ Check for table scans on large tables → Add indexes
3. ✓ Check for missing JOIN indexes → Add foreign key indexes
4. ✓ Look for function calls on indexed columns → Rewrite query
5. ✓ Check for N+1 queries in application → Add eager loading
6. ✓ Check for inefficient pagination → Switch to cursor-based
7. ✓ Verify statistics are current → Run ANALYZE
8. ✓ Re-measure: Confirm improvement

**Red flags in EXPLAIN output:**
- Seq Scan on tables with >10,000 rows
- "Rows Removed by Filter" in thousands
- Nested Loop with large outer table
- Hash joins not fitting in memory
- Estimated rows wildly different from actual

**Advanced techniques (beyond this guide):**
- Materialized views (pre-computed query results)
- Query hints (forcing specific execution plans)
- Partitioning (splitting large tables)
- Read replicas (offload read queries)
- Caching layer (Redis, Memcached)

---

## Conclusion

Database design fundamentals encompass choosing the right database type, ensuring ACID properties, normalizing (or denormalizing) appropriately, creating strategic indexes, and optimizing queries. Each decision involves trade-offs between consistency, performance, scalability, and complexity.

**Key takeaways:**

1. **Choose your database type** based on your access patterns, consistency needs, and scale requirements
2. **Understand ACID properties** to make informed decisions about when to relax them
3. **Normalize for data integrity** but denormalize strategically for read performance
4. **Index thoughtfully**—not too few, not too many, just what your queries need
5. **Optimize queries** through EXPLAIN analysis, proper joins, avoiding N+1, and smart pagination

Master these fundamentals, and you'll build database systems that are fast, reliable, and scalable.