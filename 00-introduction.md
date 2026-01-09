# PostgreSQL Mastery: From Zero to Expert

## Welcome to Your PostgreSQL Journey

PostgreSQL is the world's most advanced open-source relational database. Whether you're a complete beginner or an experienced developer looking to deepen your understanding, this guide will help you through everything from basic SQL to the intricate details of PostgreSQL's source code.

## What is PostgreSQL?

PostgreSQL (often called "Postgres") is an object-relational database management system (ORDBMS) with over 35 years of active development. It began in 1986 as  at the University of California, Berkeley.

In this curriculum, you’ll learn PostgreSQL in two ways at the same time:
- **As a user**: writing SQL, designing schemas, and debugging slow queries
- **As an engineer**: connecting what you see in SQL to the real code paths in the PostgreSQL source tree

### Key Characteristics

- **ACID Compliant**: Full support for transactions with Atomicity, Consistency, Isolation, and Durability
- **SQL Standard Compliance**: Implements most of the SQL standard with extensions
- **Extensible**: Add custom data types, operators, functions, and index types
- **Robust Concurrency**: Multi-Version Concurrency Control (MVCC) allows readers and writers to never block each other
- **Advanced Features**: Full-text search, JSON/JSONB, arrays, range types, geometric types, and more
- **Reliability**: Write-Ahead Logging (WAL) ensures data integrity even after crashes

## Why Learn PostgreSQL?

1. **Industry Standard**: Used by thousands of organizations worldwide, from startups to Fortune 500 companies
2. **Career Growth**: PostgreSQL expertise is highly valued in the job market
3. **Open Source**: No licensing costs, vibrant community, transparent development
4. **Modern Features**: Constantly evolving with cutting-edge database capabilities
5. **Transferable Knowledge**: Concepts apply to other databases and distributed systems

## How This Guide is Organized

This guide is designed as a progressive learning path:

### Beginner Level (Chapters 1-3)
- Getting started with installation and basic operations
- SQL fundamentals and CRUD operations
- Understanding data types

### Intermediate Level (Chapters 4-8)
- Database architecture and internals
- Storage engine mechanics
- Query processing and optimization
- Indexing strategies
- Transactions and concurrency

### Advanced Level (Chapters 9-13)
- Write-Ahead Logging (WAL)
- Replication strategies
- Extensions development
- Database administration
- Performance tuning

### Expert Level (Chapters 14-16)
- Source code navigation
- Deep internals and contributing to PostgreSQL
- Command-line utilities and tools

## Learning Philosophy

This guide follows these principles:

### 1. Learn by Doing
Every chapter includes hands-on exercises. You'll write real queries, examine real data structures, and solve real problems.

### 2. Understand the "Why"
We don't just show you *how* to do something—we explain *why* PostgreSQL works that way. Understanding internals makes you a better practitioner.

### 3. Progressive Disclosure
Complex topics are introduced gradually. Each chapter builds on previous knowledge.

### 4. Source Code Connection
Since you have access to the PostgreSQL source code, we'll frequently reference the actual implementation. This gives you unparalleled insight into how the database really works.

## Prerequisites

To get the most from this curriculum:

- **Basic computer skills**: Comfortable with command line operations
- **Programming basics**: Understanding of variables, functions, and basic logic (any language)
- **No prior database experience required**: We start from the fundamentals

## Your Learning Environment

Throughout this guide, you'll need:

1. **A PostgreSQL installation** (we'll cover this in Chapter 1)
2. **A terminal/command prompt**
3. **A text editor** (for writing SQL scripts and examining source code)
4. **The PostgreSQL source code** (https://github.com/postgres/postgres)

## Quick Terminology

Before we dive in, let's establish some common terms:

| Term | Definition |
|------|------------|
| **Database** | A collection of tables, indexes, and other objects |
| **Table/Relation** | A structured collection of rows (records). |
| **Row/Tuple** | A single record in a table |
| **Column/Attribute** | A field within a table |
| **Query** | A request for data from the database |
| **Transaction** | A unit of work that is atomic |
| **Index** | A data structure for fast lookups |
| **Schema** | A namespace for database objects |

## Conventions Used in This Guide

Throughout this curriculum, we use visual markers to distinguish between hands-on exercises and illustrative examples:

**🔬 Try It** - Hands-On Exercises

When you see **"🔬 Try It:"** followed by SQL, **execute this code** in your psql session. These build your database incrementally.

**🔬 Try It:**
```sql
-- Execute this in psql
CREATE TABLE example (id SERIAL PRIMARY KEY);
```

**📖 Example** - Illustrations Only

When you see **"📖 Example"** or code in a blockquote, this is for **illustration only**—don't execute it. These demonstrate concepts or show what output looks like.

> ```sql
> -- This is an illustration, not meant to be executed
> SELECT * FROM hypothetical_table;
> ```

**Expected Output**

When we show expected results, they appear in plain text blocks:

```
 id | name
----+-------
  1 | Alice
  2 | Bob
```

**Quick Reference**

| Marker | Meaning |
|--------|---------|
| **🔬 Try It:** | Execute this SQL in psql |
| **📖 Example** | Illustration only, don't execute |
| `-- Expected output:` | What you should see after running |
| `-- Note:` | Important information about the code |

## The PostgreSQL Source Code Structure

Since you're learning from the source code repository, here's a quick overview of what you'll find:

You don’t need to understand all of these directories up front. The point is orientation: when a chapter says “this is implemented in `src/backend/access/nbtree/`”, you’ll know where to look.

```
postgres/
├── src/
│   ├── backend/          # The database server
│   │   ├── access/       # Storage access methods (heap, indexes)
│   │   ├── catalog/      # System catalog management
│   │   ├── commands/     # SQL command implementations
│   │   ├── executor/     # Query executor
│   │   ├── optimizer/    # Query planner/optimizer
│   │   ├── parser/       # SQL parser
│   │   ├── postmaster/   # Process management
│   │   ├── replication/  # Streaming replication
│   │   ├── storage/      # Buffer manager, lock manager
│   │   ├── tcop/         # Traffic cop (query dispatcher)
│   │   └── utils/        # Utility functions
│   ├── bin/              # Client tools (psql, pg_dump, etc.)
│   ├── include/          # Header files
│   ├── interfaces/       # Client libraries (libpq)
│   ├── pl/               # Procedural languages
│   └── test/             # Regression tests
├── contrib/              # Additional modules
└── doc/                  # Documentation
```

## How to Use This Guide

### Recommended Approach

1. **Read each chapter sequentially** - They build on each other
2. **Complete all exercises** - Practice is essential
3. **Explore the source code** - When we reference code, look at it yourself
4. **Take notes** - Write down questions and insights
5. **Experiment** - Try variations of examples

### Getting Help

If you get stuck:

1. Re-read the relevant section
2. Check the PostgreSQL documentation: https://www.postgresql.org/docs/
3. Search the PostgreSQL mailing list archives
4. Explore related source code files

## Your First Exercise

Before moving to Chapter 1, let's verify you can navigate the source code:

**Exercise 0.1: Get and Explore the Source**

1. Clone the PostgreSQL repository from GitHub:

```bash
git clone https://github.com/postgres/postgres.git
cd postgres
```

2. Checkout the PostgreSQL 18.1 release tag:

```bash
git checkout REL_18_1
```
   You should see: `HEAD is now at xxxxxxx PostgreSQL 18.1`

3. Find the main entry point of the server:

```bash
ls -la src/backend/main/
```

4. Look at the file `main.c` - this is where PostgreSQL starts.

5. Count how many C files are in the executor:

```bash
ls src/backend/executor/*.c | wc -l
```

**Expected results:**
- The repository should be on tag `REL_18_1`
- You should see `main.c` in the main directory
- There should be approximately 70+ C files in the executor

## What You'll Build

By completing this curriculum, you'll have built a comprehensive **Company Management System** from scratch. This isn't just a random collection of tables—it's a cohesive business application that a real company might use.

### Your `learning` Database - A Complete Company Schema:

**Human Resources Core (Chapters 1-2):**
| Table | Purpose |
|-------|---------|
| `departments` | Organizational units with locations and budgets |
| `employees` | Staff records with salaries, hire dates, and reporting structure |
| `projects` | Company initiatives with timelines and budgets |
| `project_assignments` | Employee-to-project mapping with roles and allocated hours |

**Time & Attendance (Chapter 7):**
| Table | Purpose |
|-------|---------|
| `timecards` | Daily work hours logged by employees (500K rows for realistic indexing demos) |

*Note: The timecards table provides high-volume data essential for demonstrating indexing strategies and query optimization.*

**Facilities & Security (Chapter 7):**
| Table | Purpose |
|-------|---------|
| `room_bookings` | Meeting room reservations (demonstrates GiST and exclusion constraints) |
| `badge_access` | Building entry/exit logs (demonstrates BRIN indexes for time-series) |

**System & Analytics Tables (Chapters 3, 11):**
| Table | Purpose |
|-------|---------|
| `events` | Application events as JSONB (demonstrates GIN indexes) |
| `audit_log` | Change tracking via triggers |
| `feature_flags` | Application settings (demonstrates BOOLEAN) |
| Various demo tables | Data type examples (`integer_examples`, `datetime_examples`, etc.) |

### The Company Growth Story

As you progress through this curriculum, you'll follow a realistic narrative: **your company is growing, and with growth comes new challenges**. This isn't just a collection of labs—it's a simulation of what a real database professional experiences.

```
Chapter 1:  🏢 Founding - Create the company with employees & departments
    ↓
Chapter 2:  📊 Operations - Add projects, learn to query and analyze
    ↓
Chapter 3:  🔧 Features - Track events with rich data types (JSONB, arrays, etc.)
    ↓
Chapters 4-6: 🔍 Growing Pains - Understand internals as queries slow down
    ↓
Chapter 7:  ⚡ Performance Week - Fix slow timecard queries with proper indexes
    ↓
Chapter 8:  🔒 Reliability - Ensure data consistency with transactions
    ↓
Chapters 9-13: 🏗️ Enterprise - Replication, extensions, security, backups
    ↓
Chapter 16: 📤 Operations - Export, backup, and manage your database
```

**What You'll Practice:**
- **Chapter 1**: Found the company—create base employee/department structure
- **Chapter 2**: Run the business—write complex JOINs, aggregations, analytics
- **Chapter 3**: Expand features—explore data types for events, configurations, and more
- **Chapters 4-6**: Investigate slowdowns—understand how PostgreSQL stores and retrieves data
- **Chapter 7**: Solve "The Week Everything Slowed Down"—add indexes to timecards, events, and room bookings
- **Chapter 8**: Ensure reliability—implement transaction-safe operations
- **Chapters 11-12**: Enterprise features—add audit triggers, security, backups
- **Chapter 16**: Day-to-day operations—export and manage your company data

**Extensions You'll Install:**
- `pageinspect` - Examine page internals
- `pg_buffercache` - View buffer cache contents
- `pg_freespacemap` - Track free space
- `btree_gist` - GiST support for scalar types
- `pg_stat_statements` - Query performance statistics

**Skills You'll Demonstrate:**
- Writing complex queries with window functions and CTEs
- Designing efficient indexes for your query patterns
- Implementing audit triggers and data validation
- Setting up replication for high availability
- Performance tuning based on EXPLAIN ANALYZE
- Using PostgreSQL command-line utilities for backup and analysis

## Ready to Begin?

You now have the foundation to start your PostgreSQL journey. In the next chapter, we'll install PostgreSQL and write our first queries.

Remember: becoming an expert takes time and practice. Don't rush. Enjoy the learning process, and celebrate each new concept you master.

---

**Next Chapter:** [Getting Started →](01-getting-started.md)

