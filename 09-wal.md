# Chapter 9: Write-Ahead Logging (WAL)

## Learning Objectives

By the end of this chapter, you will be able to:
- Explain how WAL provides durability and crash recovery
- Understand WAL record structure and flow
- Configure WAL for different workloads
- Implement Point-in-Time Recovery (PITR)
- Troubleshoot WAL-related issues

## Part 1: WAL Fundamentals

### Why WAL Exists

WAL solves the durability problem: How do we ensure committed transactions survive a crash?

**The Problem**:
- Writing directly to data files after each transaction is slow (random I/O)
- Keeping all data in memory is fast but loses data on crash

**The Solution**:
- Write a sequential log of all changes before modifying data
- Data files can be updated lazily (background writer / checkpointer)
- On crash, replay the log to recover

### WAL Guarantees

1. **Durability**: Committed transactions are never lost
2. **Atomicity**: Partial transactions can be rolled back
3. **Consistency**: Database recovers to a consistent state

### WAL Data Flow

This is the core mental model: commits are made durable by WAL first, then data pages are flushed later. This is why WAL is usually sequential I/O while data files are often random I/O.

```
Client
  │
  ▼
Backend process executes UPDATE/INSERT/DELETE
  │
  ├─(1) Generate WAL record(s) describing the change
  │
  ├─(2) Apply the change to the in-memory data page in shared buffers
  │       ┌──────────────────────────────┐
  │       │ Shared Buffers (data pages)  │  (shared memory)
  │       └──────────────────────────────┘
  │
  └─(3) Insert WAL record into WAL buffers
          ┌──────────────────────────────┐
          │ WAL Buffers (wal_buffers)    │  (shared memory)
          └──────────────┬───────────────┘
                         │
                         │ (4) WAL writer and/or committing backend flushes WAL
                         ▼
                ┌─────────────────────────┐
                │ WAL segment files       │  (on disk: $PGDATA/pg_wal/)
                │ sequential-ish I/O      │
                └──────────────┬──────────┘
                               │
                               │ (5) Background writer / checkpointer flush dirty data pages later
                               ▼
                ┌─────────────────────────┐
                │ Data files (heap/index) │  (on disk: $PGDATA/base/...)
                │ often random I/O        │
                └─────────────────────────┘

Optional paths:
  - Archiver copies completed WAL segments (archive_mode) for PITR
  - WAL sender streams WAL to standbys for replication
```

**Note:** This diagram intentionally omits many details (fsync timing, full-page images, commit/async commit differences, and how much WAL can be flushed by the backend vs the walwriter). The key invariant to remember is the WAL rule: **WAL must be durable before the corresponding data page is written**.

## Part 2: WAL Records

**Source**: `src/include/access/xlogrecord.h`, `src/backend/access/transam/xlog.c`

### WAL Record Structure

The important thing isn’t memorizing fields—it’s understanding that WAL records are *typed* (by resource manager) and contain enough information to redo changes during crash recovery or replication.

Each WAL record contains:

```
┌─────────────────────────────────────────────────┐
│            XLogRecord Header                    │
├─────────────────────────────────────────────────┤
│  xl_tot_len    │  Total length of record        │
│  xl_xid        │  Transaction ID                │
│  xl_prev       │  Pointer to previous record    │
│  xl_info       │  Resource manager info         │
│  xl_rmid       │  Resource manager ID           │
│  xl_crc        │  CRC checksum                  │
├─────────────────────────────────────────────────┤
│            XLogRecordBlockHeader(s)             │
│  (which blocks this record affects)             │
├─────────────────────────────────────────────────┤
│            Block Data                           │
│  (actual changes to apply)                      │
└─────────────────────────────────────────────────┘
```

### Resource Managers

Each type of change has its own resource manager:

**🔬 Try It:**
```sql
-- View resource managers
SELECT * FROM pg_get_wal_resource_managers();
```

Common resource managers:
- `Heap` - Table row operations
- `Btree` - B-tree index operations
- `Transaction` - Commit/abort records
- `XLOG` - Checkpoint records
- `Standby` - Hot standby info

**🔬 Try It:** Examine WAL Contents

```sql
-- Enable pg_walinspect extension
CREATE EXTENSION IF NOT EXISTS pg_walinspect;

-- View recent WAL records
SELECT 
    start_lsn,
    end_lsn,
    resource_manager,
    record_type,
    record_length,
    main_data_length
FROM pg_get_wal_records_info(
    -- pg_lsn arithmetic uses byte offsets (numeric), not strings like '1MB'
    -- 1MB = 1024*1024 bytes
    pg_current_wal_lsn() - (1024*1024)::numeric,
    pg_current_wal_lsn()
)
LIMIT 20;

-- Detailed record info (replace LSN with a valid LSN from previous statement)
SELECT * FROM pg_get_wal_record_info('0/3E3AE788');
```

## Part 3: WAL Segments

### Segment Files

WAL is stored in segment files in `$PGDATA/pg_wal/`:

```bash
ls -la $PGDATA/pg_wal/

# Files named: TTTTTTTTXXXXXXYYYYYYYY
# T = timeline
# X = high 32 bits of LSN
# Y = segment number
```

Default segment size: 16MB (configurable at initdb with `--wal-segsize`)

### WAL Timelines (Why the “T” Exists)

WAL filenames start with a **timeline ID** (TLI). A timeline is a logical “branch” of WAL history for a cluster.

You usually see multiple timelines when:
- A standby is **promoted** (it starts generating WAL that diverges from the old primary)
- You perform **PITR** to a point in time and then start the server (recovery can create a new branch)

Conceptually:
- Timeline **1** is the original history.
- When you promote/recover, PostgreSQL writes a `.history` file and starts writing WAL on a new timeline (2, 3, …).
- This is how Postgres prevents ambiguity: after a fork in history, WAL is identified by both **timeline + segment + offset**.

**Why timelines matter operationally**

- After a failover, the old primary is now on the **wrong timeline**. To re-join it as a standby, you typically need **`pg_rewind`** (if possible) or a fresh base backup.
- Tools like replication managers depend on timeline changes to reason about safe failover/failback.

**🔬 Try It:** See the timeline in your current WAL filename

```sql
SELECT
  pg_walfile_name(pg_current_wal_lsn()) AS wal_file,
  substring(pg_walfile_name(pg_current_wal_lsn()) from 1 for 8) AS timeline_hex;
```

> The timeline is shown in hex in the first 8 characters of the WAL segment filename.

### Log Sequence Number (LSN)

LSN is the position in the WAL stream:

**🔬 Try It:**
```sql
-- Current WAL position
SELECT pg_current_wal_lsn();

-- LSN of last checkpoint
SELECT checkpoint_lsn FROM pg_control_checkpoint();

-- Convert LSN to segment filename
SELECT pg_walfile_name('0/1234ABCD');

-- Extract segment and offset
SELECT pg_walfile_name_offset('0/1234ABCD');
```

### WAL Segment Lifecycle

```
1. Pre-allocated (pg_wal/)
        │
        ▼
2. Current segment (being written)
        │
        ▼
3. Filled segment
        │
        ▼
4a. Archived (if archiving enabled)
    → archive_command copies to safe location
        │
        ▼
4b. Recycled or Removed
    → When checkpoint makes it unnecessary
```

## Part 4: WAL Configuration

### Key Parameters

**🔬 Try It:**
```sql
-- Buffer configuration
SHOW wal_buffers;           -- Size of WAL buffer (default: -1, auto)
SHOW wal_writer_delay;      -- Delay between WAL flushes (200ms)

-- Checkpoint configuration
SHOW checkpoint_timeout;    -- Maximum time between checkpoints (5min)
SHOW max_wal_size;          -- WAL size to trigger checkpoint (1GB)
SHOW min_wal_size;          -- Minimum WAL to keep (80MB)
SHOW checkpoint_completion_target;  -- Spread I/O (0.9)

-- Synchronization
SHOW wal_sync_method;       -- How to sync WAL to disk
SHOW synchronous_commit;    -- When to confirm commit
SHOW fsync;                 -- Whether to fsync (never disable!)

-- Replication/Archiving
SHOW wal_level;             -- Detail level (replica, logical)
SHOW archive_mode;          -- Enable WAL archiving
SHOW archive_command;       -- Command to archive segments
```

### Commit Synchronization Options

**🔬 Try It:**
```sql
-- Fully synchronous (default, safest)
SET synchronous_commit = on;

-- Return after write to WAL buffer (risky: data loss window)
SET synchronous_commit = off;

-- Wait for WAL write but not fsync
SET synchronous_commit = local;

-- Wait for standby acknowledgment
SET synchronous_commit = remote_write;  -- Standby received
SET synchronous_commit = remote_apply;  -- Standby applied
```

**Exercise 9.1: Measure Commit Latency**

**🔬 Try It:**
```sql
-- Compare synchronous vs asynchronous commits
\timing on

-- We'll use an existing HR table so this works in the cohesive project database.
-- We'll also tag demo rows so we can clean them up after measuring.

-- Synchronous (default)
BEGIN;
INSERT INTO timecards (emp_id, work_date, hours_worked, status, description, submitted_at)
SELECT
  (SELECT MIN(emp_id) FROM employees),
  CURRENT_DATE - INTERVAL '1 day',
  8.00,
  'submitted',
  'wal_commit_demo_sync',
  NOW()
FROM generate_series(1, 1000);
COMMIT;  -- Note the time

-- Asynchronous
SET synchronous_commit = off;
BEGIN;
INSERT INTO timecards (emp_id, work_date, hours_worked, status, description, submitted_at)
SELECT
  (SELECT MIN(emp_id) FROM employees),
  CURRENT_DATE - INTERVAL '1 day',
  8.00,
  'submitted',
  'wal_commit_demo_async',
  NOW()
FROM generate_series(1, 1000);
COMMIT;  -- Note the time difference

RESET synchronous_commit;

-- Cleanup demo rows (keeps the HR dataset tidy)
DELETE FROM timecards
WHERE description IN ('wal_commit_demo_sync', 'wal_commit_demo_async');
```

## Part 5: Checkpoints

**Source**: `src/backend/postmaster/checkpointer.c`

Checkpoints write all dirty buffers to disk:

### What Happens During Checkpoint

1. Write all dirty shared buffers to data files
2. Write a checkpoint record to WAL
3. Update `pg_control` with checkpoint location
4. Recycle old WAL segments

Checkpoints do **not** mean “flush everything for every transaction.” They’re periodic and are a major source of write bursts if not tuned correctly.

**How the checkpointer relates to the background writer**

They are separate background processes that **complement** each other:
- **Background writer (`bgwriter`)**: writes *some* dirty buffers continuously in the background, mostly to smooth I/O and keep a pool of clean buffers available for backends.
- **Checkpointer**: is responsible for the checkpoint itself—making sure that, at checkpoint time, the system writes out the remaining dirty buffers needed and records the checkpoint safely.

So yes, they “work together” in the sense that `bgwriter` reduces the amount of work the checkpointer must do during a checkpoint, which helps reduce sudden I/O spikes. But they don’t run as a single combined unit: **checkpoints can still be heavy** if the workload dirties pages faster than the background writer can flush them.

### Checkpoint Triggers

- `checkpoint_timeout` exceeded
- `max_wal_size` worth of WAL generated
- Manual `CHECKPOINT` command
- Server shutdown

### Monitoring Checkpoints

**🔬 Try It:**
```sql
-- Checkpointer statistics (checkpoint activity lives here in modern PostgreSQL)
SELECT * FROM pg_stat_checkpointer;
```

Key fields to understand:
- **`num_timed`**: checkpoints triggered by `checkpoint_timeout`
- **`num_requested`**: “forced” checkpoints (commonly due to WAL pressure / `max_wal_size`)
- **`write_time`**: time spent writing dirty buffers during checkpoints (ms)
- **`sync_time`**: time spent in fsync during checkpoints (ms)
- **`buffers_written`**: number of buffers written during checkpoints

> **Name change note:** In older releases you may see names like `checkpoints_timed` / `checkpoints_req` / `checkpoint_write_time` / `checkpoint_sync_time`. In PostgreSQL 17/18 these are exposed as `num_timed` / `num_requested` / `write_time` / `sync_time`.

If `num_requested` grows frequently relative to `num_timed`, consider increasing `max_wal_size` (and tune checkpoint settings) to avoid constant forced checkpoints.

**🔬 Try It:** Background writer statistics (separate from checkpoints)

```sql
SELECT buffers_clean, maxwritten_clean, buffers_alloc, stats_reset
FROM pg_stat_bgwriter;
```

## Part 6: Crash Recovery

When PostgreSQL starts after a crash:

### Recovery Process

```
1. Read pg_control to find last checkpoint
2. Start reading WAL from checkpoint location
3. For each WAL record:
   - Identify affected page
   - Apply change if page is older than record
4. Continue until end of WAL
5. Database is now consistent
6. Start accepting connections
```

### Full-Page Writes

**Problem**: A crash during page write could leave a torn page.  A torn page is where only a portion of the database block was written to disk.  This is possible where a database page/block which is 8k is written to a file system using 4k blocks.  In a torn page scenario, the O/S only rights to one of the underlying file system blocks before the crash happens.

**Solution**: First modification to a page after checkpoint writes full page image to the WAL stream:

**🔬 Try It:**
```sql
SHOW full_page_writes;
```

`full_page_writes` is normally **on** because it protects against torn pages after a checkpoint. The trade-off is higher WAL volume, but it’s a key safety feature.

### WAL and Buffer Management

The “WAL rule” (log must be durable before a dirty page is written) was introduced in **Part 1: WAL Data Flow**. This section focuses on the part that matters for crash recovery:

**How recovery decides whether to apply a WAL record**

Each data page stores a **page LSN** (`pd_lsn`) — the LSN of the most recent WAL record applied to that page.
During crash recovery (or standby replay), PostgreSQL uses a simple rule:

- If the page’s LSN is **older than** the WAL record’s LSN, the record must be applied (redo).
- If the page’s LSN is **newer than or equal to** the record’s LSN, the record is skipped (the change is already present).

This is one of the reasons WAL replay can be idempotent and safe.

> If you want to see `pd_lsn` on a real page, Chapter 5 (`pageinspect`) shows how to inspect page headers.

## Part 7: Continuous Archiving

Archive WAL segments for backup/recovery:

### Setting Up Archiving

Archiving is configured in `postgresql.conf` (or `ALTER SYSTEM`) and is typically used with a real archive destination (local disk, NFS, object storage via a script, etc.).

**📖 Example:** `postgresql.conf` archiving settings

> ```conf
> archive_mode = on
> archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
> ```

**What the placeholders mean:**
- `%p`: full path to the WAL segment file
- `%f`: WAL segment filename only

**🔬 Try It:** Check current archiving settings

```sql
SHOW archive_mode;
SHOW archive_command;
SHOW archive_timeout;
```

### Archive Status

```bash
# Files waiting to be archived
ls $PGDATA/pg_wal/archive_status/

# .ready files need archiving
# .done files are archived
```

### Testing Archive Command

**🔬 Try It:**
```sql
-- Force archive command execution
SELECT pg_switch_wal();

-- Check if archiving is working
SELECT * FROM pg_stat_archiver;
-- archived_count: successful archives
-- failed_count: failed attempts
-- last_archived_wal: most recent archived file
```

## Part 8: Point-in-Time Recovery (PITR)

Restore database to any point in time using base backup + WAL archives:

### Creating a Base Backup

```bash
# Start backup
psql -c "SELECT pg_backup_start('my_backup', false)"

# Copy data directory (while database is running!)
rsync -a --exclude='pg_wal/*' $PGDATA/ /backup/base/

# Stop backup
psql -c "SELECT pg_backup_stop()"
```

Or use `pg_basebackup`:

```bash
pg_basebackup -D /backup/base -Ft -z -P

# -D: destination directory
# -Ft: tar format
# -z: compress
# -P: progress reporting
```

### Performing PITR

1. **Stop PostgreSQL** (if running)

2. **Replace data directory** with base backup

3. **Create recovery signal**:
```bash
touch $PGDATA/recovery.signal
```

4. **Configure recovery in postgresql.conf**:

**🔬 Try It:**
```sql
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
# Or:
# recovery_target_xid = '12345'
# recovery_target_name = 'my_restore_point'
# recovery_target_lsn = '0/1234ABCD'
```

5. **Start PostgreSQL** - it will recover to the target

### Recovery Targets

**🔬 Try It:**
```sql
-- Create named restore point
SELECT pg_create_restore_point('before_migration');

-- Recovery targets (in postgresql.conf):
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_xid = '12345'
recovery_target_lsn = '0/1234ABCD'
recovery_target_name = 'before_migration'
recovery_target = 'immediate'  -- Stop at end of valid backup

-- What to do when target reached:
recovery_target_action = 'pause'    -- Pause for inspection
recovery_target_action = 'promote'  -- Promote to primary
recovery_target_action = 'shutdown' -- Shutdown cleanly
```



## Part 9: WAL Statistics and Monitoring

**🔬 Try It:**
```sql
-- WAL statistics
SELECT * FROM pg_stat_wal;

-- Key metrics:
-- wal_records: Total WAL records generated
-- wal_bytes: Total bytes written
-- wal_buffers_full: Times WAL buffer was full
-- wal_write: Number of WAL writes
-- wal_sync: Number of syncs

-- Write activity per table (DML volume; this correlates with WAL volume but is not “WAL per table”)
SELECT 
    schemaname,
    relname,
    n_tup_ins,
    n_tup_upd,
    n_tup_del
FROM pg_stat_user_tables
ORDER BY n_tup_ins + n_tup_upd + n_tup_del DESC;

-- Current WAL generation rate
WITH wal_rate AS (
    SELECT 
        pg_current_wal_lsn() AS lsn,
        now() AS ts
)
SELECT 
    pg_size_pretty(
        pg_wal_lsn_diff(w2.lsn, w1.lsn) / 
        EXTRACT(EPOCH FROM w2.ts - w1.ts)::NUMERIC * 3600
    ) AS wal_per_hour
FROM wal_rate w1, 
     (SELECT pg_sleep(10), pg_current_wal_lsn() AS lsn, now() AS ts) w2;
```

## Part 10: WAL Troubleshooting

### Common Issues

**1. WAL directory filling up**

```bash
# Check WAL directory size
du -sh $PGDATA/pg_wal/

# Causes:
# - Archive command failing
# - Long-running transactions
# - Replication slot lag
```

**🔬 Try It:**
```sql
-- Check for long-running transactions
SELECT pid, xact_start, state, query 
FROM pg_stat_activity 
WHERE xact_start < NOW() - INTERVAL '1 hour';

-- Check replication slot lag
SELECT slot_name, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;
```

**2. Slow checkpoints**

**🔬 Try It:**
```sql
-- If checkpoints are slow, check:
-- - Disk I/O capacity
-- - shared_buffers size
-- - checkpoint_completion_target

-- Checkpoint activity (PostgreSQL 17/18)
SELECT * FROM pg_stat_checkpointer;
```

**3. WAL replay slow on standby**

**🔬 Try It:**
```sql
-- On standby:
SELECT pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_last_xact_replay_timestamp();
```

## Source Code References

| Component | Source File |
|-----------|-------------|
| WAL records | `src/include/access/xlogrecord.h` |
| WAL insert | `src/backend/access/transam/xloginsert.c` |
| WAL write | `src/backend/access/transam/xlog.c` |
| Recovery | `src/backend/access/transam/xlogrecovery.c` |
| Checkpointer | `src/backend/postmaster/checkpointer.c` |

## Summary

In this chapter, you learned:

1. **WAL Purpose**: Durability through logging before data modification
2. **WAL Records**: Structure and resource managers
3. **WAL Segments**: File organization and lifecycle
4. **Configuration**: Tuning for different workloads
5. **Checkpoints**: How dirty pages get written to disk
6. **Crash Recovery**: Replaying WAL to restore consistency
7. **Continuous Archiving**: Saving WAL for backup
8. **PITR**: Restoring to any point in time
9. **Monitoring**: Statistics and troubleshooting

## Quick Reference

```sql
-- WAL position
SELECT pg_current_wal_lsn();
SELECT pg_walfile_name('0/1234ABCD');

-- Force WAL flush
SELECT pg_switch_wal();

-- Checkpoint
CHECKPOINT;

-- Restore points
SELECT pg_create_restore_point('name');

-- Archive status
SELECT * FROM pg_stat_archiver;

-- WAL stats
SELECT * FROM pg_stat_wal;
```


---

**Previous Chapter:** [← Transactions](08-transactions.md)

**Next Chapter:** [Replication →](10-replication.md)

