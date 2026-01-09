# PostgreSQL Mastery: Complete Learning Curriculum

## From Zero to Expert in PostgreSQL

Welcome to the most comprehensive PostgreSQL learning resource available. This curriculum takes you from absolute beginner to deep internals expert, with hands-on exercises throughout.

## 📚 Curriculum Overview

### 🟢 Beginner Level
*Get started with PostgreSQL basics*

| Chapter | Topic | Time | Description |
|---------|-------|------|-------------|
| [00](00-introduction.md) | **Introduction** | 30 min | Welcome, philosophy, and course overview |
| [01](01-getting-started.md) | **Getting Started** | 2 hrs | Installation, first database, basic operations |
| [02](02-sql-fundamentals.md) | **SQL Fundamentals** | 4 hrs | SELECT, JOINs, aggregations, subqueries |
| [03](03-data-types.md) | **Data Types** | 3 hrs | Numbers, text, JSON, arrays, and more |

### 🟡 Intermediate Level
*Understand how PostgreSQL works*

| Chapter | Topic | Time | Description |
|---------|-------|------|-------------|
| [04](04-database-architecture.md) | **Architecture** | 3 hrs | Processes, memory, data directory |
| [05](05-storage-engine.md) | **Storage Engine** | 4 hrs | Pages, tuples, TOAST, FSM |
| [06](06-query-processing.md) | **Query Processing** | 4 hrs | Parser, planner, executor, EXPLAIN |
| [07](07-indexing.md) | **Indexing** | 4 hrs | B-tree, GiST, GIN, BRIN, strategies |
| [08](08-transactions.md) | **Transactions** | 3 hrs | ACID, MVCC, isolation levels, locking |

### 🟠 Advanced Level
*Master complex features*

| Chapter | Topic | Time | Description |
|---------|-------|------|-------------|
| [09](09-wal.md) | **Write-Ahead Logging** | 3 hrs | WAL internals, checkpoints, PITR |
| [10](10-replication.md) | **Replication** | 4 hrs | Streaming, logical, failover |
| [11](11-server-side-programming.md) | **Server-side Programming** | 4 hrs | Functions, triggers, PL/pgSQL, procedural languages |
| [12](12-security.md) | **Security** | 4 hrs | Roles/privileges, pg_hba.conf, auth methods, SSL/TLS, RLS |
| [13](13-performance-tuning.md) | **Performance Tuning** | 4 hrs | Optimization, configuration, benchmarking |

### 🔴 Expert Level
*Dive into the source code and master utilities*

| Chapter | Topic | Time | Description |
|---------|-------|------|-------------|
| [14](14-utility-tools.md) | **Utility Tools** | 3 hrs | pg_dump, pg_restore, pg_waldump, pg_controldata |
| [15](15-autovacuum-and-bloat.md) | **Autovacuum & Bloat** | 4 hrs | Autovacuum tuning, freezing, bloat diagnosis |
| [16](16-extension-development.md) | **Extension Development & Extensibility** | 5 hrs | PGXS, control files, C extensions, FDWs, background workers |

### 📎 Appendices
*Reference materials*

| Appendix | Topic | Description |
|----------|-------|-------------|
| [A](appendix-a-configuration-reference.md) | **Configuration Reference** | Fully annotated postgresql.conf with all parameters |
| [B](appendix-b-source-code-tour.md) | **Source Code Tour** | Orientation for navigating the PostgreSQL source tree |

## 🗺️ Learning Paths

### Path 1: Database Developer (20 hours)
Focus on using PostgreSQL effectively in applications:
```
00 → 01 → 02 → 03 → 06 → 07 → 08
```

### Path 2: Database Administrator (38 hours)
Learn to manage and maintain PostgreSQL:
```
00 → 01 → 02 → 04 → 05 → 09 → 10 → 12 → 13 → 14 → 15
```

### Path 3: Complete Journey (55+ hours)
Master everything from basics to internals:
```
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13 → 14 → 15 → 16
```

## 🛠️ Prerequisites

- Basic command line familiarity
- Any programming language experience
- Access to this PostgreSQL source code repository
- A computer with at least 4GB RAM

## 🚀 Quick Start

- **Build PostgreSQL from source** (Chapter 1):

```bash
./configure --prefix=$HOME/pgsql/pg18
make
make install
```

- **Initialize your data directory**:

```bash
export PATH=$HOME/pgsql/pg18/bin:$PATH
export PGDATA=$HOME/pgsql/pg18/data
initdb -D $PGDATA
```

- **Start the server**:

```bash
pg_ctl -D $PGDATA start
```

- **Connect and start learning**:

```bash
psql postgres
```

## 📖 How to Use This Curriculum

### For Each Chapter:

1. **Read the content** - Understand concepts before practicing
2. **Run the examples** - Type them yourself, don't copy-paste
3. **Complete exercises** - Practice is essential
4. **Explore source code** - When referenced, look at the actual code
5. **Take notes** - Document your insights and questions

### Hands-On Learning:

Every chapter includes:
- 📝 **Code examples** you can run immediately
- 🏋️ **Exercises** to practice what you've learned
- 🔬 **Source code references** linking to actual PostgreSQL code
- 📊 **Diagrams** explaining complex concepts

## 🎯 Learning Outcomes

After completing this curriculum, you will be able to:

**Beginner Level:**
- Install and configure PostgreSQL
- Write complex SQL queries
- Design efficient database schemas
- Choose appropriate data types

**Intermediate Level:**
- Explain PostgreSQL's internal architecture
- Optimize queries using EXPLAIN
- Design effective indexing strategies
- Implement proper transaction handling

**Advanced Level:**
- Set up replication for high availability
- Perform database administration tasks
- Tune PostgreSQL for optimal performance

**Expert Level:**
- Navigate and understand the source code
- Debug internal PostgreSQL operations
- Contribute to PostgreSQL development
- Implement custom access methods
- Build and package custom extensions

## 📁 Source Code Connection

This curriculum is designed to be used alongside the PostgreSQL source code. Key directories you'll explore:

```
src/backend/          # Server implementation
├── access/          # Storage and indexes
├── catalog/         # System catalogs
├── commands/        # SQL commands
├── executor/        # Query execution
├── optimizer/       # Query planning
├── parser/          # SQL parsing
├── postmaster/      # Process management
├── replication/     # Replication
├── storage/         # Buffer and lock managers
└── utils/           # Utilities

src/include/         # Header files
src/bin/             # Client tools
contrib/             # Extensions
```

## 🤝 Contributing

Found an error or want to improve the curriculum? Contributions are welcome!

## 📚 Additional Resources

- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Wiki](https://wiki.postgresql.org/)
- [PostgreSQL Mailing Lists](https://www.postgresql.org/list/)
- [PostgreSQL Source Code](https://git.postgresql.org/)

## 📋 Chapter Status

| Chapter | Status | Last Updated |
|---------|--------|--------------|
| 00 - Introduction | ✅ Complete | 2026-01 |
| 01 - Getting Started | ✅ Complete | 2026-01 |
| 02 - SQL Fundamentals | ✅ Complete | 2026-01 |
| 03 - Data Types | ✅ Complete | 2026-01 |
| 04 - Architecture | ✅ Complete | 2026-01 |
| 05 - Storage Engine | ✅ Complete | 2026-01 |
| 06 - Query Processing | ✅ Complete | 2026-01 |
| 07 - Indexing | ✅ Complete | 2026-01 |
| 08 - Transactions | ✅ Complete | 2026-01 |
| 09 - WAL | ✅ Complete | 2026-01 |
| 10 - Replication | ✅ Complete | 2026-01 |
| 11 - Server-side Programming | ✅ Complete | 2026-01 |
| 12 - Security | ✅ Complete | 2026-01 |
| 13 - Performance | ✅ Complete | 2026-01 |
| 14 - Utility Tools | ✅ Complete | 2026-01 |
| 15 - Autovacuum & Bloat | ✅ Complete | 2026-01 |
| 16 - Extension Development | ✅ Complete | 2026-01 |
| A - Configuration Reference | ✅ Complete | 2026-01 |
| B - Source Code Tour | ✅ Complete | 2026-01 |

---

## Begin Your Journey

Ready to master PostgreSQL? Start with the [Introduction](00-introduction.md) and work through each chapter at your own pace. Remember: the goal isn't speed—it's deep understanding.

**Happy learning! 🐘**

