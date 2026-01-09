# Chapter 4: PostgreSQL Database Architecture

## Learning Objectives

By the end of this chapter, you will be able to:
- Explain PostgreSQL's process architecture
- Understand shared memory and its components
- Navigate the data directory structure
- Describe the lifecycle of a connection
- Understand background processes and their roles

## Part 1: Process Architecture Overview

PostgreSQL uses a multi-process architecture (not multi-threaded). This design choice provides:
- **Stability**: One crashing connection doesn't affect others
- **Security**: Process isolation between clients
- **Simplicity**: Easier to debug and reason about

### The Process Model

This diagram is the “mental picture” to keep in your head for the rest of the internals chapters:
- the **postmaster** accepts connections and spawns processes
- each **backend process** is responsible for exactly one client connection
- all backends coordinate through **shared memory** and background processes

```
                   ┌─────────────────────────────┐
                   │       Client Applications   │
                   └─────────────┬───────────────┘
                                 │
                   ┌─────────────▼───────────────┐
                   │      Postmaster (main)      │
                   │   Listens for connections   │
                   └─────────────┬───────────────┘
                                 │ fork()
       ┌─────────────────────────┼─────────────────────────┐
       │                         │                         │
┌──────▼──────┐           ┌──────▼──────┐          ┌──────▼──────┐
│   Backend   │           │   Backend   │          │   Backend   │
│  Process 1  │           │  Process 2  │          │  Process N  │
└──────┬──────┘           └──────┬──────┘          └──────┬──────┘
       │                         │                         │
       └─────────────────────────┼─────────────────────────┘
                                 │
                   ┌─────────────▼───────────────┐
                   │       Shared Memory         │
                   │  (Buffer Pool, WAL, etc.)   │
                   └─────────────────────────────┘
```

**🔬 Try It:** View PostgreSQL Processes

```bash
# See all PostgreSQL processes
ps aux | grep postgres

# Or use the system catalog
psql -c "SELECT pid, usename, application_name, state, query 
         FROM pg_stat_activity"
```

You'll see something like:
```
  PID   USER     COMMAND
 1234   postgres postgres: checkpointer
 1235   postgres postgres: background writer
 1236   postgres postgres: walwriter
 1237   postgres postgres: autovacuum launcher
 1238   postgres postgres: stats collector
 1239   postgres postgres: logical replication launcher
 1240   postgres postgres: user dbname [local] idle
```

## Part 2: The Postmaster Process

The postmaster (`src/backend/postmaster/postmaster.c`) is PostgreSQL's main process:

### Responsibilities

1. **Listen for connections** on configured ports
2. **Authenticate clients** using pg_hba.conf rules
3. **Fork backend processes** for each connection
4. **Manage background workers** (spawn, monitor, restart)
5. **Handle shutdown signals** and coordinate cleanup

### Startup Sequence

Looking at `src/backend/main/main.c`:

This is a simplified sketch. The real startup also initializes subsystems like shared memory, statistics, and background workers.

```c
// Simplified startup flow
int main(int argc, char *argv[])
{
    // 1. Initialize memory contexts
    MemoryContextInit();
    
    // 2. Set up signal handlers
    pqsignal(SIGINT, die);
    pqsignal(SIGTERM, die);
    
    // 3. Load configuration
    InitializeGUCOptions();
    
    // 4. Choose mode based on arguments
    if (argc > 1 && strcmp(argv[1], "--single") == 0)
        PostgresMain(...);  // Single-user mode
    else
        PostmasterMain(...);  // Normal multi-user mode
}
```

### Connection Handling

When a client connects:

1. **Listen**: Postmaster accepts TCP connection
2. **Fork**: Creates new backend process
3. **Authenticate**: Backend validates credentials
4. **Initialize**: Sets up memory contexts, caches
5. **Ready**: Backend enters command loop

**Exercise 4.1: Watch Connection Lifecycle**

**🔬 Try It:**
```sql
-- In one terminal, start watching
SELECT pid, usename, state, query 
FROM pg_stat_activity 
WHERE usename = current_user;

-- Refresh query every 5 seconds (ctrl-C to stop)
\watch 5

-- In another terminal, connect multiple times
psql -c "SELECT pg_sleep(10)" postgres
psql -c "SELECT pg_sleep(10)" postgres
```

Watch the processes appear and disappear.

## Part 3: Backend Processes

When you connect to PostgreSQL with `psql` or your application, the postmaster **forks a new process** just for your connection. This is called a **backend process**. Each connection = one backend = one OS process.

### Why One Process Per Connection?

| Aspect | Benefit |
|--------|---------|
| **Isolation** | One crashed query doesn't take down the server |
| **Security** | Process-level memory isolation between users |
| **Simplicity** | Each backend has its own memory, no complex locking for session state |
| **OS Integration** | Leverages OS scheduler, memory protection, and debugging tools |

The trade-off is that connections are "expensive"—each one uses memory (~5-10MB) and fork() has overhead. This is why connection poolers (like PgBouncer) are common in production.

### The Query Protocol: Simple vs Extended

PostgreSQL supports two ways to execute queries:

**Simple Query Protocol (`Q` message):**
- Client sends complete SQL string
- Server parses, plans, and executes in one step
- Used by: `psql`, simple scripts

**Extended Query Protocol (`P`/`B`/`E` messages):**
- Parse (`P`): Create a prepared statement
- Bind (`B`): Attach parameter values
- Execute (`E`): Run the prepared statement
- Used by: Application drivers (JDBC, libpq with prepared statements)

The extended protocol is more efficient for repeated queries because parsing and planning happen once.

### The Command Loop

The backend sits in an infinite loop waiting for client commands. The code lives in `src/backend/tcop/postgres.c` (tcop = "traffic cop"):

```c
// Simplified from postgres.c - the heart of PostgreSQL's backend
for (;;)
{
    // Block until client sends something
    ReadCommand(&input_message);
    
    // First byte tells us what kind of message
    switch (firstchar)
    {
        case 'Q':   // Simple query - "SELECT * FROM employees"
            exec_simple_query(query_string);
            break;
            
        case 'P':   // Parse - create prepared statement
            exec_parse_message(query_string, ...);
            break;
            
        case 'B':   // Bind - attach parameters to prepared statement
            exec_bind_message(...);
            break;
            
        case 'E':   // Execute - run the prepared statement
            exec_execute_message(...);
            break;
            
        case 'X':   // Terminate - client disconnecting
            proc_exit(0);
            break;
    }
    
    // Tell client we're ready for the next command
    ReadyForQuery(whereToSendOutput);
}
```

**Exercise 4.1: Watch Backend Processing**

**🔬 Try It:**
```sql
-- In one session, start a long query
SELECT pg_sleep(30);

-- In another session, watch what the backend is doing
SELECT pid, state, query, query_start 
FROM pg_stat_activity 
WHERE state = 'active';
```

### Memory Management in Backends

PostgreSQL uses **memory contexts**—hierarchical memory allocators that allow efficient bulk deallocation. Instead of freeing individual allocations, PostgreSQL frees entire contexts at once.

If you’re not a C developer, here’s the intuition:
- In most programs, every allocation you make has to be freed individually later. If you forget, you leak memory.
- A database backend handles **thousands of small allocations per query** (parse tree nodes, planner structures, executor tuples, sort buffers, etc.). Tracking and freeing each one individually would be slow and error-prone.

Memory contexts solve this by acting like “buckets”:
- The backend allocates lots of small objects into a bucket (a context).
- When the operation is over (end of statement, end of query, end of transaction), PostgreSQL can drop the whole bucket at once.

This matters operationally because it:
- prevents gradual memory growth in long-running sessions
- makes error handling safer (cleanup still happens even when a query errors out)
- keeps per-session memory usage more predictable under load

```
TopMemoryContext (lives for backend lifetime)
├── MessageContext (freed after each command)
├── CacheMemoryContext (catalog caches, long-lived)
├── PortalContext (per-query, freed when query completes)
│   └── ExecutorState (per-executor)
└── ErrorContext (for error recovery)
```

**Why contexts matter:**
- **No memory leaks**: When a query finishes, free the entire PortalContext
- **Fast cleanup**: One `MemoryContextReset()` vs thousands of `free()` calls
- **Debugging**: Easy to see where memory is being used

**Where you’ll see this concept in practice:**
- **Per-statement memory**: parse/analyze/planning scratch data should not “stick around” after the statement finishes.
- **Per-query execution memory**: executor state and intermediate tuples should disappear when the query completes.
- **Long-lived caches**: some memory *should* persist (catalog caches), because it speeds up later queries.

**🔬 Try It:**
```sql
-- View memory contexts in your current backend
SELECT name, level, path, total_bytes, used_bytes 
FROM pg_backend_memory_contexts 
ORDER BY used_bytes DESC 
LIMIT 10;
```

Key memory contexts:
- **TopMemoryContext**: Lives for entire backend lifetime—never freed until disconnect
- **MessageContext**: Per-message, freed after each command completes
- **PortalContext**: Per-query execution—holds query plan and execution state
- **CacheMemoryContext**: System catalog caches (table definitions, etc.)

## Part 4: Background Processes

PostgreSQL runs several essential background processes:

### Checkpointer

**File**: `src/backend/postmaster/checkpointer.c`

Purpose: Periodically flush dirty buffers to disk, ensuring data durability

**🔬 Try It:**
```sql
-- View checkpoint information (PostgreSQL 17+ uses pg_stat_checkpointer)
SELECT 
    num_timed,           -- Checkpoints triggered by checkpoint_timeout
    num_requested,       -- Checkpoints triggered manually or by max_wal_size
    buffers_written,     -- Buffers written during checkpoints
    write_time,          -- Total write time (ms)
    sync_time            -- Total sync time (ms)
FROM pg_stat_checkpointer;

-- Configure checkpoint behavior
SHOW checkpoint_timeout;            -- Time between checkpoints (default: 5min)
SHOW checkpoint_completion_target;  -- Spread I/O over this fraction of interval
SHOW max_wal_size;                  -- WAL size that triggers checkpoint
```

### Background Writer (bgwriter)

**File**: `src/backend/postmaster/bgwriter.c`

Purpose: Periodically write some dirty buffers in the background to smooth disk I/O and keep a supply of reusable (clean) buffers, which helps reduce checkpoint write spikes

**🔬 Try It:**
```sql
-- Background writer stats
SELECT buffers_clean, maxwritten_clean, buffers_alloc 
FROM pg_stat_bgwriter;
```

### WAL Writer

**File**: `src/backend/postmaster/walwriter.c`

Purpose: Flush WAL buffers to disk

**What are WAL buffers?**

WAL buffers are a shared-memory area (configured by `wal_buffers`) where backends temporarily accumulate WAL data before it is written to the WAL files on disk (`$PGDATA/pg_wal/`). Think of them as the “staging area” between:
- many concurrent backend processes generating changes, and
- sequential writes to the WAL log on disk

**What is a WAL record?**

A WAL record is a *redo* description of a change. It contains enough information for PostgreSQL to re-apply the change during crash recovery or on a standby during replication (for example: “insert this tuple,” “split this B-tree page,” “commit transaction X”).

**Where this fits in the architecture**

- Backends generate WAL records as they modify data pages in shared buffers.
- WAL records are appended into WAL buffers.
- The WAL writer (and sometimes the committing backend) flushes WAL buffers to WAL segment files on disk.
- Later, the checkpointer writes the modified data pages to the main data files.

This ordering is enforced by the core WAL safety rule: **WAL must reach durable storage before the corresponding dirty data page is written** (the “write-ahead” in WAL).

**🔬 Try It:**
```sql
-- WAL statistics
SELECT * FROM pg_stat_wal;
```

### Autovacuum Launcher and Workers

**Files**: `src/backend/postmaster/autovacuum.c`

Purpose: Automatic maintenance (VACUUM and ANALYZE)

**🔬 Try It:**
```sql
-- Current autovacuum activity
SELECT * FROM pg_stat_progress_vacuum;

-- Autovacuum settings
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_threshold;
```

### Stats Collector

**File**: `src/backend/postmaster/pgstat.c`

Purpose: Collect statistics about database activity

**🔬 Try It:**
```sql
-- Statistics views
SELECT * FROM pg_stat_database;
SELECT * FROM pg_stat_user_tables;
SELECT * FROM pg_stat_user_indexes;
```

### Logical Replication Launcher

Purpose: Manage logical replication workers

**🔬 Try It:**
```sql
-- Replication status
SELECT * FROM pg_stat_replication;
```

**Exercise 4.2: Monitor Background Processes**

> **Note:** In PostgreSQL 17+, checkpoint statistics were moved from `pg_stat_bgwriter` to a new `pg_stat_checkpointer` view. The examples below use the PostgreSQL 18 view structure.

**🔬 Try It:**
```sql
-- View checkpointer statistics (checkpoint-related activity)
SELECT 
    num_timed AS checkpoints_timed,      -- Scheduled checkpoints
    num_requested AS checkpoints_requested,  -- Requested checkpoints (e.g., CHECKPOINT command)
    num_done AS checkpoints_completed,
    buffers_written AS checkpoint_buffers,
    write_time,                          -- Time spent writing (ms)
    sync_time                            -- Time spent syncing (ms)
FROM pg_stat_checkpointer;

-- View background writer statistics
SELECT 
    buffers_clean,       -- Buffers written by bgwriter
    maxwritten_clean,    -- Times bgwriter stopped due to writing too many buffers
    buffers_alloc        -- Total buffers allocated
FROM pg_stat_bgwriter;
```

## Part 5: Shared Memory

All PostgreSQL processes share memory for communication and caching:

### Shared Memory Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Shared Memory                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │
│  │  Shared Buffer  │  │   WAL Buffers   │  │  Lock Table │  │
│  │     Pool        │  │                 │  │             │  │
│  │(shared_buffers) │  │  (wal_buffers)  │  │             │  │
│  └─────────────────┘  └─────────────────┘  └─────────────┘  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │
│  │   CLOG Buffers  │  │   Proc Array    │  │ Two-Phase   │  │
│  │                 │  │  (process info) │  │  State      │  │
│  └─────────────────┘  └─────────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Shared Buffer Pool

**Files**: `src/backend/storage/buffer/`

The buffer pool caches data pages in memory:

**🔬 Try It:**
```sql
-- See buffer pool configuration
SHOW shared_buffers;  -- Total size

-- What's in the buffer cache?
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

SELECT 
    c.relname,
    COUNT(*) AS buffers,
    pg_size_pretty(COUNT(*) * 8192) AS size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 10;
```

### WAL Buffers

**Files**: `src/backend/access/transam/xlog.c`

WAL (Write-Ahead Log) buffers hold transaction log records in memory before they're written to disk. This is critical for PostgreSQL's durability guarantee: **every change is logged before it's considered committed**.

**🔬 Try It:**
```sql
-- See WAL buffer configuration
SHOW wal_buffers;  -- Size of WAL buffer (default: -1 means auto-tuned)

-- WAL activity statistics
SELECT 
    wal_records,           -- Total WAL records generated
    wal_bytes,             -- Total WAL bytes generated
    wal_buffers_full      -- Times WAL buffers were full (should be low)
FROM pg_stat_wal;
```

**Sizing guideline:** WAL buffers are typically auto-sized to 1/32 of `shared_buffers`, capped at 64MB. For write-heavy workloads, you might increase this, but usually the default is fine.

### Lock Management

**Files**: `src/backend/storage/lmgr/` (lmgr = Lock Manager)

PostgreSQL uses locks to coordinate access between concurrent transactions. The lock table lives in shared memory so all backends can see who's holding what.

**Lock types (from lightest to heaviest):**

| Lock Mode | Conflicts With | Used For |
|-----------|----------------|----------|
| `AccessShareLock` | AccessExclusive | Simple SELECT |
| `RowShareLock` | Exclusive, AccessExclusive | SELECT FOR UPDATE/SHARE |
| `RowExclusiveLock` | Share, ShareRowExclusive, Exclusive, AccessExclusive | INSERT, UPDATE, DELETE |
| `ShareLock` | RowExclusive, ShareRowExclusive, Exclusive, AccessExclusive | CREATE INDEX (non-concurrent) |
| `AccessExclusiveLock` | Everything | DROP TABLE, ALTER TABLE, VACUUM FULL |

**Key insight:** Readers don't block readers, and readers don't block writers (thanks to MVCC). But some DDL operations need exclusive locks.

**🔬 Try It:**
```sql
-- View current locks
SELECT 
    locktype,
    relation::regclass AS table_name,
    mode,
    granted,
    pid,
    pg_blocking_pids(pid) AS blocked_by
FROM pg_locks 
WHERE relation IS NOT NULL
ORDER BY granted, relation;

-- Find blocking queries
SELECT 
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.pid != blocking.pid;
```

**Deadlock detection:** PostgreSQL automatically detects deadlocks and aborts one of the transactions. You can see deadlock events in the server log.

**Exercise 4.3: Explore Shared Memory**

**🔬 Try It:**
```sql
-- Calculate recommended shared_buffers
SELECT 
    pg_size_pretty(setting::BIGINT * 8192) AS current_shared_buffers
FROM pg_settings 
WHERE name = 'shared_buffers';

-- Buffer hit ratio (should be > 99%)
SELECT 
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit) AS heap_hit,
    CASE WHEN sum(heap_blks_read) + sum(heap_blks_hit) > 0
         THEN round(sum(heap_blks_hit) / 
              (sum(heap_blks_hit) + sum(heap_blks_read))::NUMERIC * 100, 2)
         ELSE 0
    END AS ratio
FROM pg_statio_user_tables;
```

## Part 6: The Data Directory ($PGDATA)

The data directory contains all database files:

```
$PGDATA/
├── base/                    # Database files
│   ├── 1/                   # template1
│   ├── 13067/               # template0  
│   └── 16384/               # Your databases (OID)
├── global/                  # Cluster-wide tables
├── pg_commit_ts/            # Commit timestamp data
├── pg_dynshmem/             # Dynamic shared memory
├── pg_hba.conf              # Client authentication
├── pg_ident.conf            # User name mapping
├── pg_logical/              # Logical replication data
├── pg_multixact/            # Multi-transaction status
├── pg_notify/               # LISTEN/NOTIFY data
├── pg_replslot/             # Replication slot data
├── pg_serial/               # Serializable transaction info
├── pg_snapshots/            # Exported snapshots
├── pg_stat/                 # Statistics files
├── pg_stat_tmp/             # Temporary statistics
├── pg_subtrans/             # Subtransaction status
├── pg_tblspc/               # Tablespace symlinks
├── pg_twophase/             # Two-phase commit state
├── PG_VERSION               # PostgreSQL version
├── pg_wal/                  # WAL files
│   ├── 000000010000000000000001
│   └── archive_status/
├── pg_xact/                 # Transaction status (CLOG)
├── postgresql.auto.conf     # Auto-generated config
├── postgresql.conf          # Main configuration
├── postmaster.opts          # Last startup options
└── postmaster.pid           # Postmaster lock file
```

### Exploring the Data Directory

**🔬 Try It:**
```bash
# Find your data directory
psql -c "SHOW data_directory" postgres

# Explore base directory
ls -la $PGDATA/base/

# Find a specific database's directory
psql -c "SELECT oid, datname FROM pg_database" postgres
ls -la $PGDATA/base/<oid from sql output>/
```

### Database File Organization

Each table is stored as one or more files:

**🔬 Try It:**
```sql
-- Find the file for a table
SELECT pg_relation_filepath('employees');
-- Returns: base/16384/16385

-- Get the actual file size
SELECT pg_relation_size('employees') AS heap_size,
       pg_table_size('employees') AS total_size,
       pg_total_relation_size('employees') AS with_indexes;
```

```bash
# View the actual files (your numbers may vary)
ls -la $PGDATA/base/16384/16385*
```

You might see:
- `16385` - Main heap file
- `16385_fsm` - Free Space Map
- `16385_vm` - Visibility Map

## Part 7: System Catalogs

PostgreSQL stores metadata in system tables (catalogs):

### Key System Catalogs

**🔬 Try It:**
```sql
-- Tables (relations)
SELECT relname, relkind, reltuples 
FROM pg_class 
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace;

-- Columns (attributes)
SELECT attname, atttypid::regtype, attnum 
FROM pg_attribute 
WHERE attrelid = 'employees'::regclass AND attnum > 0;

-- Indexes
SELECT indexrelid::regclass, indrelid::regclass 
FROM pg_index 
WHERE indrelid = 'employees'::regclass;

-- Namespaces (schemas)
SELECT * FROM pg_namespace;

-- Data types
SELECT typname, typlen, typtype FROM pg_type LIMIT 20;

-- Functions/Procedures
SELECT proname, pronargs, prorettype::regtype 
FROM pg_proc 
WHERE pronamespace = 'public'::regnamespace;
```

### Information Schema (SQL Standard)

**🔬 Try It:**
```sql
-- Tables
SELECT table_name, table_type 
FROM information_schema.tables 
WHERE table_schema = 'public';

-- Columns
SELECT column_name, data_type, is_nullable 
FROM information_schema.columns 
WHERE table_name = 'employees';

-- Constraints
SELECT constraint_name, constraint_type 
FROM information_schema.table_constraints 
WHERE table_name = 'employees';
```

## Part 8: Configuration Management

### Configuration Files

1. **postgresql.conf** - Main configuration
2. **pg_hba.conf** - Client authentication
3. **pg_ident.conf** - User name mapping
4. **postgresql.auto.conf** - ALTER SYSTEM changes

### Viewing and Changing Settings

**🔬 Try It:**
```sql
-- View all settings
SELECT name, setting, unit, context 
FROM pg_settings;

-- Settings by category
SELECT name, setting, context 
FROM pg_settings 
WHERE category = 'Resource Usage / Memory';

-- Check if a setting requires restart
SELECT name, context 
FROM pg_settings 
WHERE context = 'postmaster';  -- Requires restart

-- Change a setting
ALTER SYSTEM SET work_mem = '256MB';
SELECT pg_reload_conf();  -- Apply without restart

-- Or for session only
SET work_mem = '256MB';
```

### Context Types

| Context | When Applied |
|---------|--------------|
| internal | Cannot be changed |
| postmaster | Requires server restart |
| sighup | Requires reload (pg_reload_conf()) |
| superuser | Can be set by superuser |
| user | Can be set by any user |

**Exercise 4.4: Configuration Exploration**

**🔬 Try It:**
```sql
-- Find settings that differ from defaults
SELECT name, setting, boot_val, 
       CASE WHEN setting <> boot_val THEN 'CHANGED' ELSE 'DEFAULT' END
FROM pg_settings
WHERE setting <> boot_val
ORDER BY name;

-- Find memory-related settings
SELECT name, setting, unit, 
       pg_size_pretty(setting::BIGINT * 
           CASE unit 
               WHEN '8kB' THEN 8192 
               WHEN 'kB' THEN 1024 
               WHEN 'MB' THEN 1048576 
               ELSE 1 
           END) AS readable_value
FROM pg_settings
WHERE unit IN ('8kB', 'kB', 'MB')
ORDER BY name;
```

## Part 9: Connection Pooling (External)

PostgreSQL creates a new process per connection, which is expensive. Production deployments often use connection pooling:

### Popular Poolers

1. **PgBouncer** - Lightweight, single-threaded
2. **Pgpool-II** - Feature-rich, supports load balancing
3. **Built-in pooling** - Some ORMs provide pooling

### Connection Limits

**🔬 Try It:**
```sql
-- View connection settings
SHOW max_connections;
SHOW superuser_reserved_connections;

-- Current connections
SELECT count(*) FROM pg_stat_activity;

-- Connections by state
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;
```

The `state` breakdown is a great sanity check: a large number of `idle` sessions often means the application isn’t pooling connections well.

**📖 Example:** Terminate long-idle sessions (use with caution)

> ```sql
> SELECT pg_terminate_backend(pid)
> FROM pg_stat_activity
> WHERE state = 'idle'
>   AND state_change < NOW() - INTERVAL '10 minutes'
>   AND pid <> pg_backend_pid();
> ```

## Source Code Deep Dive

### Key Files for Architecture Understanding

| Component | Source File |
|-----------|-------------|
| Main entry point | `src/backend/main/main.c` |
| Postmaster | `src/backend/postmaster/postmaster.c` |
| Backend command loop | `src/backend/tcop/postgres.c` |
| Shared memory init | `src/backend/storage/ipc/shmem.c` |
| Buffer manager | `src/backend/storage/buffer/bufmgr.c` |
| Lock manager | `src/backend/storage/lmgr/lock.c` |
| Background writer | `src/backend/postmaster/bgwriter.c` |
| Checkpointer | `src/backend/postmaster/checkpointer.c` |
| WAL writer | `src/backend/postmaster/walwriter.c` |
| Autovacuum | `src/backend/postmaster/autovacuum.c` |

### Tracing a Connection

1. `postmaster.c:ServerLoop()` - Accept connection
2. `postmaster.c:BackendStartup()` - Fork new process
3. `postmaster.c:BackendInitialize()` - Setup backend
4. `postgres.c:PostgresMain()` - Enter command loop

## Summary

In this chapter, you learned:

1. **Process Architecture**: Multi-process model with postmaster
2. **Backend Processes**: One per connection, running the command loop
3. **Background Processes**: Checkpointer, bgwriter, walwriter, autovacuum
4. **Shared Memory**: Buffer pool, WAL buffers, lock tables
5. **Data Directory**: Organization of $PGDATA
6. **System Catalogs**: Metadata storage in pg_* tables
7. **Configuration**: postgresql.conf, pg_hba.conf, settings management

## Architecture Quick Reference

```
PostgreSQL Server
├── Postmaster (parent process)
│   ├── Accepts connections
│   ├── Forks backends
│   └── Manages background workers
├── Backend Processes (one per connection)
│   ├── Parse → Plan → Execute queries
│   └── Own memory contexts
├── Background Workers
│   ├── Checkpointer
│   ├── Background Writer
│   ├── WAL Writer
│   ├── Autovacuum
│   └── Stats Collector
└── Shared Memory
    ├── Buffer Pool (shared_buffers)
    ├── WAL Buffers (wal_buffers)
    ├── Lock Tables
    └── Process Array
```

---

## Chapter Cleanup

The following table was created for demonstration purposes in this chapter:

**🔬 Try It:** Clean up demonstration objects

```sql
-- Drop demonstration table
DROP TABLE IF EXISTS file_test;
```

---

**Previous Chapter:** [← Data Types](03-data-types.md)

**Next Chapter:** [Storage Engine →](05-storage-engine.md)

