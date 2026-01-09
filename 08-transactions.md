# Chapter 8: Transactions and Concurrency Control

## The Story So Far

With the earlier performance issues addressed, the company is running smoothly again. But a new concern has emerged: **the HR team ran a salary update that partially failed, leaving some employees with incorrect pay**. Meanwhile, two project managers tried to assign the same contractor at the same time, causing a data conflict.

These aren't just bugs—they're signs that your application needs proper **transaction handling**. In a growing company with multiple users hitting the database simultaneously, you need guarantees that data stays consistent even when things go wrong.

This chapter teaches you how PostgreSQL keeps your data safe.

## Learning Objectives

By the end of this chapter, you will be able to:
- Explain ACID properties and how PostgreSQL implements them
- Understand MVCC (Multi-Version Concurrency Control)
- Use different transaction isolation levels effectively
- Handle concurrent access and prevent anomalies
- Implement proper locking strategies
- Debug deadlocks and contention issues

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

## Part 1: ACID Properties

PostgreSQL guarantees ACID transactions. ACID isn’t marketing vocabulary—it’s the contract that makes a database safe under failures and concurrency. In real applications, ACID is what prevents “partially applied” business operations (like half a payroll update) when something goes wrong.

### Atomicity

Atomicity treats a transaction as a single, indivisible unit. If a transaction consists of five steps and the fourth step fails, the entire transaction is canceled, and the database is rolled back to its state before the transaction started.

**📖 Example:** Classic bank transfer

> ```sql
> BEGIN;
> UPDATE accounts SET balance = balance - 100 WHERE id = 1;
> UPDATE accounts SET balance = balance + 100 WHERE id = 2;
> -- If anything fails, both updates are rolled back
> COMMIT;
> ```

**Implementation (high level)**: PostgreSQL uses MVCC tuple versioning, which makes rollback possible by keeping old versions until they’re no longer needed. (We’ll make this concrete in Part 3.)

### Consistency

Consistency ensures that a transaction only takes the database from one valid state to another. Any data written must follow all defined rules, including constraints (like "balance cannot be negative"), triggers, and cascades.

It does **not** mean “a consistent view of data across concurrent transactions.” That meaning belongs to **Isolation** (snapshots / visibility rules).

In practice, “rules and invariants” can include:
- declarative constraints (CHECK, UNIQUE, FOREIGN KEY)
- trigger-enforced rules
- application invariants (e.g., “hours_worked must stay within 0–24”)

**📖 Example:** Constraints enforce valid state

> ```sql
> -- Constraints are checked at transaction end
> CREATE TABLE transfers (
>     id SERIAL PRIMARY KEY,
>     from_account INTEGER REFERENCES accounts(id),
>     to_account INTEGER REFERENCES accounts(id),
>     amount NUMERIC CHECK (amount > 0)
> );
> ```

**Implementation (high level)**: Constraints, triggers, and foreign keys enforce rules at statement time and/or commit time (depending on whether they are deferrable).

### Isolation

Concurrent transactions don’t interfere with each other. You should not observe half-finished work from another session, and your reads/writes should behave predictably based on the chosen isolation level.

**📖 Example:** Concurrent sessions

> ```sql
> -- Session 1
> BEGIN;
> UPDATE accounts SET balance = 1000 WHERE id = 1;
> -- Not yet committed
> 
> -- Session 2 sees old value
> SELECT balance FROM accounts WHERE id = 1;  -- Still shows original value
> ```

**Implementation (high level)**: MVCC provides snapshots (“what I can see”) so readers don’t block writers and writers don’t block readers in the common case.

### Durability

Committed transactions survive crashes. If PostgreSQL reports `COMMIT` succeeded, that transaction’s effects will still be there after a crash and restart.

**📖 Example:** Crash safety

> ```sql
> -- Once COMMIT returns success, data is safe
> COMMIT;
> -- Even if server crashes immediately after, data is preserved
> ```

**Implementation (high level)**: Write-Ahead Logging (WAL) ensures durability by writing redo information to disk before dirty data pages are flushed.

## Part 2: Transaction Basics

**📖 Example:** Transaction Control

```sql
-- Start transaction
BEGIN;
-- or
BEGIN TRANSACTION;
-- or
START TRANSACTION;

-- Commit changes
COMMIT;
-- or
END;

-- Undo changes
ROLLBACK;
-- or
ABORT;
```

This section is intentionally short: these keywords are the “control surface” for everything else in the chapter.
The rest of the chapter is about what those commands *mean* in PostgreSQL (MVCC versions, snapshots, locking, and durability).

### Savepoints

Savepoints create restore points within a transaction.

Savepoints are most useful when you want to do a multi-step operation where some steps are “optional” or may fail, but you don’t want to abort the entire transaction. They’re also a practical way to recover from an error in interactive sessions.

**📖 Example:** Using savepoints

> ```sql
> BEGIN;
> 
> UPDATE accounts SET balance = balance - 100 WHERE id = 1;
> SAVEPOINT sp1;
> 
> UPDATE accounts SET balance = balance + 100 WHERE id = 2;
> -- Oops, wrong account!
> 
> ROLLBACK TO SAVEPOINT sp1;
> -- First update preserved, second undone
> 
> UPDATE accounts SET balance = balance + 100 WHERE id = 3;
> COMMIT;
> ```

### Autocommit Mode

By default, each statement is its own transaction. This is great for simple operations, but it can be dangerous when you *intend* multiple statements to be “all-or-nothing.”

This is why you can run `INSERT` or `UPDATE` without writing `BEGIN/COMMIT` in most day-to-day usage—PostgreSQL wraps the statement in an implicit transaction.

**📖 Example:**

> ```sql
> -- This is implicitly:
> -- BEGIN;
> INSERT INTO logs (message) VALUES ('test');
> -- COMMIT;
> 
> -- Disable autocommit in psql:
> \set AUTOCOMMIT off
> ```

**Exercise 8.1: Transaction Practice**

**🔬 Try It:**
```sql
-- We'll use an HR table you already have: timecards
-- Create a single known timecard row we can safely reference throughout this chapter.

INSERT INTO timecards (emp_id, work_date, hours_worked, description, status, submitted_at)
SELECT
    (SELECT MIN(emp_id) FROM employees),
    CURRENT_DATE - INTERVAL '1 day',
    8.00,
    'Chapter 8 demo: transaction/savepoint',
    'submitted',
    NOW()
RETURNING timecard_id;

-- Copy the returned timecard_id and set it for the rest of this chapter (replace 500001 with the timecard_id from INSERT):
\set timecard_id 500001

-- Practice transaction + savepoint with an intentional failure
BEGIN;

-- Step 1: Make a change we want to keep
UPDATE timecards
SET status = 'approved',
    approved_at = NOW()
WHERE timecard_id = :timecard_id;

SAVEPOINT before_bad_insert;

-- Step 2: Trigger an error (violates CHECK hours_worked <= 24)
INSERT INTO timecards (emp_id, work_date, hours_worked, description, status)
VALUES (
    (SELECT MIN(emp_id) FROM employees),
    CURRENT_DATE - INTERVAL '2 day',
    25.00,
    'This insert should fail (hours_worked > 24)',
    'submitted'
);

-- Step 3: Recover from the error and continue
ROLLBACK TO SAVEPOINT before_bad_insert;

UPDATE timecards
SET description = description || ' (approved in same transaction)'
WHERE timecard_id = :timecard_id;

COMMIT;

-- Verify the final state:
--   timecards.status = approved
--   timecards.description = appended with approved statement
SELECT timecard_id, emp_id, work_date, hours_worked, status, approved_at, description
FROM timecards
WHERE timecard_id = :timecard_id;
```

## Part 3: MVCC Deep Dive

**Source**: `src/backend/access/heap/heapam.c`, `src/include/access/htup_details.h`

PostgreSQL uses Multi-Version Concurrency Control (MVCC). Instead of overwriting rows in place, it creates new versions. Each transaction reads from a **snapshot**, so readers can continue without being blocked by writers.

### Transaction Snapshots

Transaction **snapshots** are how PostgreSQL defines “what I can see” for MVCC.
At any moment there are transactions that have committed, transactions that are still in progress, and transactions that haven’t started yet.
When PostgreSQL takes a snapshot, it captures a view of the world like:

- **which transaction IDs are definitely committed and visible**
- **which transaction IDs are still running and should be treated as “not visible yet”**

That snapshot is then used to decide tuple visibility by comparing a row’s `xmin`/`xmax` against the snapshot.

Why this matters:
- In **Read Committed**, PostgreSQL effectively takes a new snapshot for each statement.
- In **Repeatable Read** and **Serializable**, PostgreSQL uses a consistent snapshot for the whole transaction.

**🔬 Try It:**
```sql
-- View current transaction's snapshot
SELECT pg_current_snapshot();

-- Export snapshot for another session
SELECT pg_export_snapshot();

-- In another session, use that snapshot
SET TRANSACTION SNAPSHOT '00000003-00000003-1';
```

Sample output:
```
 pg_current_snapshot 
---------------------
 1333:1333:
```

How to read the snapshot string returned by `pg_current_snapshot()`:
- It has the form **`xmin:xmax:xip_list`**
- **`xmin`**: the oldest transaction ID that is still considered in-progress for this snapshot
- **`xmax`**: the first not-yet-assigned transaction ID (upper bound)
- **`xip_list`**: a list of active transaction IDs at the moment the snapshot was taken

In practice, you rarely need to interpret the raw string—what matters is that the snapshot is what drives MVCC visibility rules.


### How MVCC Works

Each tuple has hidden system columns that track **which transaction** created it and which transaction replaced/deleted it:
- `xmin`: Transaction ID that created this version
- `xmax`: Transaction ID that deleted/updated this version (or 0)
- `cmin`/`cmax`: Command IDs within the transaction

A command ID (cmin/cmax) is a per-transaction counter PostgreSQL uses to order statements within a single transaction. 
- What it represents: “the Nth SQL command I ran in this transaction.”
- Why it exists: even inside one transaction, PostgreSQL needs to decide whether a tuple version created/modified by an earlier statement should be visible to a later statement in the same transaction.

**🔬 Try It:**
```sql
-- See hidden columns
SELECT xmin, xmax, cmin, cmax, ctid, * 
FROM timecards
WHERE timecard_id = :timecard_id;
```

### Tuple Visibility

Tuple visibility is the rule set that answers: “given my snapshot, should I see this row version?” This is the foundation for understanding why some rows “disappear” or “reappear” across sessions.

A tuple is visible to a transaction if:
1. `xmin` is committed AND started before our snapshot
2. `xmax` is 0 OR not committed OR started after our snapshot

**🔬 Try It:**
```sql
-- Demonstrate MVCC
BEGIN;
SELECT txid_current();  -- Note: 1000

-- See tuple with our xmin
SELECT xmin, xmax, ctid, status, description
FROM timecards
WHERE timecard_id = :timecard_id;

UPDATE timecards
SET description = description || ' (edited)'
WHERE timecard_id = :timecard_id;
-- Old tuple: xmax = 1000 (our xid)
-- New tuple: xmin = 1000

SELECT xmin, xmax, ctid, status, description
FROM timecards
WHERE timecard_id = :timecard_id;
COMMIT;
```

## Part 4: Isolation Levels

PostgreSQL supports four isolation levels (only three distinct behaviors):

Isolation levels define *when* your snapshot is taken and *how long* it remains stable. Higher isolation reduces anomalies but can introduce retries (especially under Serializable).

### Read Uncommitted

In PostgreSQL, behaves same as Read Committed:

PostgreSQL does not allow dirty reads (seeing uncommitted changes) even if you ask for Read Uncommitted. This is stricter than some databases and simplifies reasoning about correctness.

**🔬 Try It:**
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- PostgreSQL never shows uncommitted data
```

### Read Committed (Default)

Each statement sees data committed before the statement started.

**📖 Example:** Two concurrent sessions

> ```sql
> -- Session 1
> BEGIN;
> UPDATE accounts SET balance = 100 WHERE id = 1;
> 
> -- Session 2
> BEGIN;
> SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
> SELECT balance FROM accounts WHERE id = 1;  -- Sees old value
> 
> -- Session 1
> COMMIT;
> 
> -- Session 2
> SELECT balance FROM accounts WHERE id = 1;  -- NOW sees 100
> COMMIT;
> ```

**Allowed anomalies** (plain English): because each statement takes a new snapshot, the “world can change” between two SELECTs in the same transaction.

- **Non-repeatable read**: you read the *same row twice* and get a different result the second time (because another transaction committed an update in between).
- **Phantom read**: you run the *same query twice* and the second run returns *more/fewer rows* (because another transaction inserted/deleted matching rows in between).

These are not bugs—they’re the trade-off that keeps Read Committed fast and concurrency-friendly.

### Repeatable Read

The transaction sees a consistent snapshot from start to finish. Re-running the same SELECT will return the same results, even if other sessions commit changes in the meantime.

**📖 Example:** Snapshot consistency

> ```sql
> -- Session 1
> BEGIN;
> SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
> SELECT balance FROM accounts WHERE id = 1;  -- 100
> 
> -- Session 2
> UPDATE accounts SET balance = 200 WHERE id = 1;
> COMMIT;
> 
> -- Session 1
> SELECT balance FROM accounts WHERE id = 1;  -- Still 100!
> COMMIT;
> ```

**Prevented anomalies**: Non-repeatable reads
**Allowed**: Serialization anomalies (but no phantoms in PostgreSQL)

### Serializable

Transactions behave as if executed serially. PostgreSQL uses **Serializable Snapshot Isolation (SSI)**, which detects dangerous dependency patterns and forces one transaction to retry to preserve correctness.

**🔬 Try It:**
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Source**: `src/backend/storage/lmgr/predicate.c`

Uses Serializable Snapshot Isolation (SSI):

**🔬 Try It:**
```sql
-- Example of serialization anomaly detection
-- Setup
CREATE TABLE counters (name TEXT PRIMARY KEY, value INTEGER);
INSERT INTO counters VALUES ('a', 0), ('b', 0);
```

```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT value FROM counters WHERE name = 'a';  -- 0
UPDATE counters SET value = 1 WHERE name = 'b';

-- Session 2
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT value FROM counters WHERE name = 'b';  -- 0
UPDATE counters SET value = 1 WHERE name = 'a';
COMMIT;  -- Success
 
-- Session 1
COMMIT;  -- ERROR: could not serialize access due to read/write dependencies
```

**Why did Session 1 fail?**

This is a classic **serialization anomaly** under snapshot isolation:
- Session 1 **read** row `a` and then **wrote** row `b`.
- Session 2 **read** row `b` and then **wrote** row `a`.

If PostgreSQL allowed both transactions to commit, there would be **no single “serial order”** (Session 1 then Session 2, or Session 2 then Session 1) that could produce what both sessions observed while preserving serializable correctness.

With SSI, PostgreSQL tracks read/write dependencies and detects this dangerous cycle. To preserve true serializable behavior, it aborts one transaction (here, Session 1) with a serialization failure so the application can **retry the transaction**.

### Isolation Level Comparison

| Level | Dirty Read | Non-Repeatable Read | Phantom | Serialization Anomaly |
|-------|------------|---------------------|---------|----------------------|
| Read Uncommitted | No* | Yes | Yes | Yes |
| Read Committed | No | Yes | Yes | Yes |
| Repeatable Read | No | No | No* | Yes |
| Serializable | No | No | No | No |

\* PostgreSQL is stricter than SQL standard requires

## Part 5: Locking

### Lock Types

Locks are how PostgreSQL coordinates access to shared resources (tables, rows, pages, transaction IDs). Locks prevent *unsafe* concurrency, but they also create waiting and contention when many sessions need the same resources.

PostgreSQL has several table-level lock modes:

| Lock Mode | Conflicts With | Typical command |
|-----------|---------------|-----------------|
| ACCESS SHARE | ACCESS EXCLUSIVE | `SELECT` |
| ROW SHARE | EXCLUSIVE, ACCESS EXCLUSIVE | `SELECT ... FOR UPDATE` |
| ROW EXCLUSIVE | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | `UPDATE` |
| SHARE UPDATE EXCLUSIVE | SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | `VACUUM` |
| SHARE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | `CREATE INDEX` |
| SHARE ROW EXCLUSIVE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | `ALTER TABLE` |
| EXCLUSIVE | All except ACCESS SHARE | `REFRESH MATERIALIZED VIEW` |
| ACCESS EXCLUSIVE | All | `DROP TABLE` |

**Lock modes in plain English (what it means for other sessions):**

- **ACCESS SHARE**: “I’m reading.” Other sessions can read and write normally, but DDL that requires `ACCESS EXCLUSIVE` will wait.
- **ROW SHARE**: “I’m selecting rows with a locking clause.” Allows normal reads/writes, but conflicts with the strongest table-level locks (`EXCLUSIVE` / `ACCESS EXCLUSIVE`).
- **ROW EXCLUSIVE**: “I’m modifying rows.” This is the common lock for `INSERT/UPDATE/DELETE`. Other sessions can still read and write, but some operations that require a stable table state may wait.
- **SHARE UPDATE EXCLUSIVE**: “I’m doing maintenance that needs coordination.” Common for `VACUUM`/`ANALYZE`. It allows normal reads/writes but prevents certain concurrent maintenance/DDL combinations.
- **SHARE**: “I’m building or validating something and need stability.” Common for `CREATE INDEX`. It allows reads, but blocks concurrent row modifications that would change what’s being built.
- **SHARE ROW EXCLUSIVE**: “I’m doing a schema change that must coordinate with writers.” It blocks many concurrent modifications and other schema/maintenance operations; you typically see it around some `ALTER TABLE` operations.
- **EXCLUSIVE**: “I need strong table-level isolation.” Rare in application code; blocks most other table-level locks except simple reads.
- **ACCESS EXCLUSIVE**: “I need to be alone with this table.” Common for DDL (`DROP/TRUNCATE/most ALTER`). It blocks **all** other accesses (reads and writes) until the operation finishes.

### Automatic Locks

Most locks are acquired automatically based on what your statements do. You don’t typically “turn on” locking—you learn which statements implicitly take which locks so you can predict blocking.

**📖 Example:** Lock modes acquired automatically

> ```sql
> -- SELECT acquires ACCESS SHARE
> SELECT * FROM accounts;
> 
> -- UPDATE/DELETE/INSERT acquires ROW EXCLUSIVE
> UPDATE accounts SET balance = 100 WHERE id = 1;
> 
> -- DDL acquires ACCESS EXCLUSIVE
> ALTER TABLE accounts ADD COLUMN status TEXT;
> ```

### Explicit Table Locks

Manual table locks are rare in application code but can be useful for administrative operations where you need a stable view of a table while performing a multi-step change.

**📖 Example:** Manual table locking

> ```sql
> -- Lock table manually
> BEGIN;
> LOCK TABLE accounts IN EXCLUSIVE MODE;
> -- Now no other transaction can modify accounts
> UPDATE accounts SET balance = balance * 1.1;
> COMMIT;
> ```

### Row-Level Locks

Row-level locks (`SELECT ... FOR UPDATE`, etc.) are the most common concurrency control tool in OLTP applications. They let you safely read-modify-write a row while making concurrent sessions wait.

**📖 Example:** Row-level lock options

> ```sql
> -- Lock specific rows for update
> BEGIN;
> SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
> -- Row is locked, others wait
> 
> -- Other options:
> SELECT * FROM accounts FOR SHARE;           -- Shared lock
> SELECT * FROM accounts FOR NO KEY UPDATE;   -- Less restrictive update lock
> SELECT * FROM accounts FOR KEY SHARE;       -- Shared lock on key only
> 
> -- Don't wait for locks
> SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
> SELECT * FROM accounts WHERE id = 1 FOR UPDATE SKIP LOCKED;
> ```

### Advisory Locks

Advisory locks are **application-managed locks**. PostgreSQL will track them and make other sessions wait (or fail fast), but **PostgreSQL does not automatically acquire them for you** the way it does with row/table locks.

That makes them useful when you need to protect a logical resource that *isn't* a single row or table lock, for example:
- Ensure only one worker processes a given payroll run ID
- Prevent two sessions from generating the same “next employee number”
- Serialize access to an external system integration step (e.g., “sync-department-42”)

Key points:
- **They don’t lock rows**: holding an advisory lock does *not* block normal `SELECT/UPDATE` statements unless your application also tries to take the same advisory lock.
- **They’re identified by a key** (either a BIGINT or two INTs). Pick a stable mapping, like `hashtext('payroll-run-2026-01')`.
- **Two scopes**:
  - **Session-level** (`pg_advisory_lock`): held until you explicitly unlock it or disconnect.
  - **Transaction-level** (`pg_advisory_xact_lock`): automatically released at `COMMIT`/`ROLLBACK` (usually safer).

Common pitfalls:
- Session-level locks can accidentally be held “forever” if your app forgets to unlock.
- You can still deadlock if two sessions take multiple advisory locks in different orders—use consistent ordering.

Application-level locks:

**🔬 Try It:**
```sql
-- Session-level locks
SELECT pg_advisory_lock(12345);  -- Block until acquired
SELECT pg_try_advisory_lock(12345);  -- Return false if not available
SELECT pg_advisory_unlock(12345);

-- Transaction-level locks (auto-released on commit)
SELECT pg_advisory_xact_lock(12345);

-- Use case: prevent duplicate job processing
BEGIN;
SELECT pg_try_advisory_xact_lock(hashtext('job-123'));
-- Returns true if we got the lock
-- Process job...
COMMIT;  -- Lock automatically released
```

**Exercise 8.2: Observe Locking**

**🔬 Try It:**
```sql
-- Session 1
BEGIN;
SELECT * FROM timecards WHERE timecard_id = :timecard_id FOR UPDATE;

-- Session 2 (in another terminal)
BEGIN;
SELECT * FROM timecards WHERE timecard_id = :timecard_id FOR UPDATE;
-- This blocks!

-- Session 3: View locks
SELECT 
    l.locktype,
    l.relation::regclass,
    l.mode,
    l.granted,
    l.pid,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation = 'timecards'::regclass;
```

## Part 6: Deadlock Detection

**Source**: `src/backend/storage/lmgr/deadlock.c`

Deadlocks happen when two sessions each hold a lock the other needs. PostgreSQL automatically detects this cycle and aborts one transaction so the system can make progress.

This is why deadlocks show up as errors you must handle at the application layer: the correct response is typically “retry the transaction.”

**📖 Example:** Deadlock scenario

> ```sql
> -- Session 1
> BEGIN;
> UPDATE accounts SET balance = 100 WHERE id = 1;
> -- Has lock on id=1
> 
> -- Session 2
> BEGIN;
> UPDATE accounts SET balance = 200 WHERE id = 2;
> -- Has lock on id=2
> 
> -- Session 1
> UPDATE accounts SET balance = 100 WHERE id = 2;
> -- Waits for Session 2's lock
> 
> -- Session 2
> UPDATE accounts SET balance = 200 WHERE id = 1;
> -- Waits for Session 1's lock = DEADLOCK!
> -- PostgreSQL detects this and kills one transaction
> 
> -- ERROR:  deadlock detected
> -- DETAIL:  Process 12345 waits for ShareLock on transaction 678;
> --          blocked by process 12346.
> --          Process 12346 waits for ShareLock on transaction 679;
> --          blocked by process 12345.
> ```

### Preventing Deadlocks

Deadlocks are usually prevented by **discipline**, not configuration:
- take locks in a consistent order
- keep transactions short
- use `NOWAIT` / `SKIP LOCKED` when building worker queues

1. **Lock resources in consistent order**:

**📖 Example:** Consistent lock ordering

> ```sql
> -- Always lock lower IDs first
> BEGIN;
> SELECT * FROM accounts WHERE id IN (1, 5, 3) ORDER BY id FOR UPDATE;
> -- Locks acquired: 1, then 3, then 5
> ```

2. **Use lock timeouts**:

**🔬 Try It:**
```sql
SET lock_timeout = '5s';
```

3. **Use NOWAIT or SKIP LOCKED**:

**🔬 Try It:**
```sql
-- Treat submitted timecards as a work queue (multiple workers can run this safely)
BEGIN;

SELECT timecard_id, emp_id, work_date, status
FROM timecards
WHERE status = 'submitted'
ORDER BY submitted_at NULLS LAST, timecard_id
FOR UPDATE SKIP LOCKED
LIMIT 1;

COMMIT;
```

## Part 7: Monitoring Concurrency

When systems become highly concurrent, “what is it waiting on?” becomes one of the most important debugging skills. PostgreSQL exposes this information through system views.

### View Active Transactions

**🔬 Try It:**
```sql
-- Current transactions
SELECT 
    pid,
    usename,
    state,
    xact_start,
    NOW() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;
```

### View Locks and Waiting

This query helps answer: “who is blocked, and who is blocking them?” In practice, you’ll use it during incidents to find the root blocker.
**🔬 Try It:**
```sql
-- Current locks
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### Transaction ID Wraparound

Transaction IDs (XIDs) are finite and **wrap around**. Internally, PostgreSQL uses 32-bit transaction IDs, so after roughly \(2^{32}\) transactions the counter cycles back to the beginning.

That sounds abstract, but it has a very concrete consequence: MVCC visibility depends on comparing XIDs (“is this insert/delete older than my snapshot?”). Once XIDs wrap, a very old XID can look “new” again unless PostgreSQL takes special action.

**What happens if wraparound isn’t prevented?**

PostgreSQL can start making incorrect visibility decisions (treating extremely old rows as if they were created by a “future” transaction). That is effectively **data corruption / data loss**, so PostgreSQL treats wraparound prevention as a critical safety requirement.

**How PostgreSQL prevents wraparound**

PostgreSQL prevents wraparound by **freezing** old tuples:
- Vacuum marks old tuples as “frozen,” meaning they are visible to everyone regardless of XID age.
- Autovacuum continuously advances this safety boundary.

Important related settings/limits (high level):
- `autovacuum_freeze_max_age`: when the database approaches this age, autovacuum becomes aggressive to freeze.
- `vacuum_freeze_min_age` / `vacuum_freeze_table_age`: control when vacuum freezes tuples during routine operation.

**Why long-running transactions are dangerous**

A long-running transaction holds an old snapshot open. Vacuum must preserve tuple versions that could still be visible to that snapshot, which can block freezing and accelerate wraparound risk.

**If you are approaching wraparound (or see “wraparound” warnings)**

1. **Find and end long-running transactions** (idle-in-transaction sessions are common culprits).
2. **Run vacuum with freezing** on the worst offending tables/databases (often via `vacuumdb --freeze` operationally).
3. **Increase maintenance capacity temporarily** (e.g., `maintenance_work_mem`, autovacuum workers/cost limits) so vacuum can keep up.

If PostgreSQL ever refuses commands with messages about preventing wraparound data loss, treat it as an incident: the immediate goal is to get vacuum/freezing caught up so normal operation can resume.

**🔬 Try It:**
```sql
-- Check for wraparound danger
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    current_setting('autovacuum_freeze_max_age')::BIGINT AS freeze_max
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Find long-running transactions (often the real reason vacuum can't freeze)
SELECT
  pid,
  usename,
  state,
  xact_start,
  now() - xact_start AS xact_age,
  query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- Tables needing vacuum
SELECT 
    schemaname,
    relname,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

## Part 8: Transaction Patterns

### Optimistic Locking

Optimistic locking assumes conflicts are rare. Instead of taking a lock up front, you detect whether the row changed since you read it and retry if needed.

In PostgreSQL, a very practical optimistic locking technique uses the hidden `xmin` system column as a “version token.”

**🔬 Try It:** Optimistic locking using `xmin` on `timecards`

```sql
-- Capture a version token (xmin) for the row you intend to update
SELECT xmin::text AS xmin_token
FROM timecards
WHERE timecard_id = :timecard_id  \gset

-- Try to update only if the version token still matches
UPDATE timecards
SET description = description || ' (optimistic update)'
WHERE timecard_id = :timecard_id
  AND xmin::text = :'xmin_token';

-- If UPDATE affects 0 rows, someone else updated the row first.
-- In application code, the typical response is: re-read and retry.
```

### Pessimistic Locking

Pessimistic locking assumes conflicts are likely. You lock the row before making decisions, ensuring no one else can modify it until you commit/rollback.

**🔬 Try It:** Lock a `timecards` row before updating
```sql
BEGIN;

SELECT timecard_id, status, description
FROM timecards
WHERE timecard_id = :timecard_id
FOR UPDATE;

-- Row is now locked; another session trying to lock it will wait.
UPDATE timecards
SET description = description || ' (pessimistic update)'
WHERE timecard_id = :timecard_id;

COMMIT;
```

### Queue Processing Pattern

This pattern is common for background workers: you want multiple workers to safely take “the next item” without double-processing.

**🔬 Try It:** Treat submitted timecards as a work queue (SKIP LOCKED)
```sql
BEGIN;

-- Pick one submitted timecard to process without blocking other workers
SELECT timecard_id
FROM timecards
WHERE status = 'submitted'
ORDER BY submitted_at NULLS LAST, timecard_id
FOR UPDATE SKIP LOCKED
LIMIT 1;

-- In a real worker, you'd now process that timecard_id.
-- Tip: in psql, you can copy the returned timecard_id into :picked_id.

-- For the lab, approve one row you selected above (replace :picked_id accordingly).
\set picked_id 12345

UPDATE timecards
SET status = 'approved',
    approved_at = NOW()
WHERE timecard_id = :picked_id;
COMMIT;
```

## Part 9: Common Pitfalls

### Long-Running Transactions

Long transactions are dangerous because they hold snapshots open. That prevents VACUUM from removing dead tuples that might still be visible to that old snapshot, which leads to bloat and can increase wraparound risk.

**🔬 Try It:**
```sql
-- Problem: Prevents VACUUM from cleaning up
BEGIN;
SELECT COUNT(*) FROM timecards;
-- ... hours later, still in transaction

-- Solution: Set timeouts
SET idle_in_transaction_session_timeout = '5min';
```

### Lock Contention on Hot Rows

“Hot row” contention happens when many sessions update the same row repeatedly (counters, “last seen” timestamps, single-row summary tables). This can become a throughput bottleneck.

**🔬 Try It:**
```sql
-- Ensure the demo table exists (it was introduced earlier in the Serializable section)
-- This pattern matters most under concurrency (many sessions doing this at once).

-- 1) Hot-row pattern: everyone updates the SAME row
UPDATE counters
SET value = value + 1
WHERE name = 'a';

-- 2) Sharded counters: spread the updates across N rows to reduce contention
-- Add a shard column once (safe if already added)
ALTER TABLE counters ADD COLUMN IF NOT EXISTS shard INT;

-- Make the primary key (name, shard) so each shard is a distinct counter
DELETE from counters;
ALTER TABLE counters DROP CONSTRAINT IF EXISTS counters_pkey;
ALTER TABLE counters ADD PRIMARY KEY (name, shard);

-- Seed 10 shards for counter 'a' (0..9). Use ON CONFLICT for re-runs.
INSERT INTO counters (name, shard, value)
SELECT 'a', s, 0
FROM generate_series(0, 9) AS s
ON CONFLICT (name, shard) DO NOTHING;

-- Now each update hits one random shard (integer 0..9)
UPDATE counters
SET value = value + 1
WHERE name = 'a'
  AND shard = floor(random() * 10)::int;

-- Sanity check: total value across shards (should increase over repeated runs)
SELECT name, SUM(value) AS total_value
FROM counters
WHERE name = 'a'
GROUP BY name;
```

### SELECT FOR UPDATE Without Index

`SELECT ... FOR UPDATE` can be expensive if the predicate can’t use an index. PostgreSQL must scan rows to find matches, and row locking adds overhead on top of the scan. On large tables, this pattern can cause long lock waits and poor throughput.

**📖 Example:**
> ```sql
> -- Problem: Locks entire table scan
> SELECT * FROM orders WHERE customer_name = 'John' FOR UPDATE;
> -- Without index, scans all rows and locks them
> 
> -- Solution: Add index
> CREATE INDEX idx_orders_customer ON orders(customer_name);
> ```

## Source Code References

| Component | Source File |
|-----------|-------------|
| Transaction manager | `src/backend/access/transam/xact.c` |
| MVCC visibility | `src/backend/access/heap/heapam_visibility.c` |
| Lock manager | `src/backend/storage/lmgr/lock.c` |
| Deadlock detection | `src/backend/storage/lmgr/deadlock.c` |
| Serializable SI | `src/backend/storage/lmgr/predicate.c` |

## Summary

In this chapter, you learned:

1. **ACID Properties**: Atomicity, Consistency, Isolation, Durability
2. **MVCC**: Multi-Version Concurrency Control via tuple versioning
3. **Isolation Levels**: Read Committed, Repeatable Read, Serializable
4. **Locking**: Table locks, row locks, advisory locks
5. **Deadlocks**: Detection, prevention, resolution
6. **Monitoring**: Viewing transactions, locks, and waiters
7. **Patterns**: Optimistic vs pessimistic locking

## Quick Reference

```sql
-- Transaction control
BEGIN;
SAVEPOINT name;
ROLLBACK TO name;
COMMIT;

-- Isolation levels
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Row locks
SELECT ... FOR UPDATE;
SELECT ... FOR SHARE;
SELECT ... FOR UPDATE NOWAIT;
SELECT ... FOR UPDATE SKIP LOCKED;

-- Table locks
LOCK TABLE t IN mode MODE;

-- Advisory locks
SELECT pg_advisory_lock(key);
SELECT pg_advisory_unlock(key);

-- Monitoring
SELECT * FROM pg_stat_activity;
SELECT * FROM pg_locks;
```

---

## Chapter Cleanup

The following tables were created for demonstration purposes in this chapter:

**🔬 Try It:** Clean up demonstration objects

```sql
-- Drop demonstration tables
DROP TABLE IF EXISTS counters CASCADE;
```

---

**Previous Chapter:** [← Indexing](07-indexing.md)

**Next Chapter:** [Write-Ahead Logging →](09-wal.md)
