# Chapter 10: Replication

## Learning Objectives

By the end of this chapter, you will be able to:
- Set up streaming replication for high availability
- Configure synchronous and asynchronous replication
- Implement logical replication for selective data copying
- Manage replication slots and monitor lag
- Design failover strategies

## Part 1: Replication Overview

PostgreSQL offers several replication mechanisms:

| Type | Description | Use Case |
|------|-------------|----------|
| **Streaming Replication** | Physical copy of entire database | High availability, read scaling |
| **Logical Replication** | Row-level changes via publications | Selective replication, upgrades |
| **Log Shipping** | Copy and apply WAL files | Simple standby setup |

### Physical vs Logical Replication

If you remember one thing: **physical replication copies bytes**, **logical replication copies row changes**.
That difference drives everything else: failover options, what you can replicate, and what kinds of upgrades/migrations are possible.

### HA Framing: RPO and RTO (Why Replication Settings Matter)

Replication choices should be driven by:
- **RPO (Recovery Point Objective)**: how much data loss is acceptable (time)
- **RTO (Recovery Time Objective)**: how long recovery can take (time)

Examples:
- Lower RPO often means stricter durability and/or **synchronous replication** (commit waits for a standby).
- Lower RTO often means automation + rehearsed failover procedures (and good observability).

```
Physical (Streaming):
Primary → WAL stream → Standby
- Exact byte-for-byte copy
- All databases replicated
- Same PostgreSQL version required

Logical:
Publisher → Decoded changes → Subscriber
- Row-level replication
- Selective tables/databases
- Cross-version possible
```

## Part 2: Streaming Replication Setup

This setup is easiest to understand as “two clusters”:
- **Primary**: accepts writes and generates WAL
- **Standby**: continuously replays WAL to stay in sync

**🔬 Try It:** Configure Primary Server

```sql
-- postgresql.conf on primary
-- wal_level = replica  (or 'logical' for logical replication)
-- max_wal_senders = 10
-- max_replication_slots = 10

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secret';
```

```bash
# pg_hba.conf - allow replication connections
# TYPE  DATABASE        USER           ADDRESS         METHOD
host    replication     replicator     10.0.0.0/24     scram-sha-256
```

### Create Replication Slot

Slots ensure WAL is retained until standby consumes it:

**🔬 Try It:**
```sql
-- On primary
SELECT pg_create_physical_replication_slot('standby1');

-- View slots
SELECT slot_name, slot_type, active, restart_lsn 
FROM pg_replication_slots;
```

### Initialize Standby

`pg_basebackup` takes a consistent snapshot of the primary’s data directory and prepares it to follow WAL from the primary. Think of it as “copy the base, then stream the changes.”

```bash
# Take base backup for standby
pg_basebackup -h primary_host -U replicator -D /var/lib/pgsql/data \
  -R -P -S standby1

# -R: Create standby.signal and configure recovery
# -P: Show progress
# -S: Use this replication slot
```

The `-R` flag creates `standby.signal` and adds to `postgresql.auto.conf`:

```
primary_conninfo = 'host=primary_host user=replicator password=secret'
primary_slot_name = 'standby1'
```

### Start Standby

```bash
pg_ctl start -D /var/lib/pgsql/data

# Standby will connect to primary and start streaming
```

### Verify Replication

**🔬 Try It:**
```sql
-- On primary: view connected standbys
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag
FROM pg_stat_replication;

-- On standby: check replication status
SELECT 
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS received,
    pg_last_wal_replay_lsn() AS replayed,
    pg_last_xact_replay_timestamp() AS last_replay_time;
```

## Part 3: Synchronous Replication

Ensure transactions are replicated before commit returns:

### Configure Synchronous Standby

**🔬 Try It:**
```sql
-- On primary (postgresql.conf)
-- synchronous_standby_names = 'standby1'  -- One sync standby
-- synchronous_standby_names = 'FIRST 2 (standby1, standby2, standby3)'  -- First 2 of 3
-- synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'   -- Any 2 of 3
```

### Synchronous Commit Levels

**🔬 Try It:**
```sql
-- On primary
SET synchronous_commit = 'remote_apply';  -- Wait for standby to apply

-- Options:
-- 'on'            - Wait for local flush (no standby wait)
-- 'remote_write'  - Wait for standby to receive (not flush)
-- 'remote_apply'  - Wait for standby to apply (strongest)
```

### Monitor Synchronous Replication

**🔬 Try It:**
```sql
SELECT 
    application_name,
    sync_state,  -- 'sync', 'async', 'potential', 'quorum'
    sent_lsn,
    replay_lsn
FROM pg_stat_replication;
```

## Part 4: Failover and Switchover

### Promoting Standby to Primary

```bash
# Promote standby
pg_ctl promote -D /var/lib/pgsql/data

# Or using SQL (PostgreSQL 12+)
SELECT pg_promote();
```

After promotion:
- `standby.signal` is removed
- Standby starts accepting writes
- Timeline increments (creates new WAL timeline)

### Split Brain (Two Primaries) — Detection and Recovery

**Split brain** is when two nodes both believe they are primary and **accept writes independently**. This is catastrophic because the clusters diverge and there is no automatic, conflict-free way to merge them back together at the physical replication level.

**Why it happens**
- network partition (primary and standby cannot see each other)
- failover automation promotes a standby, but the old primary was not properly fenced off
- clients continue writing to the old primary due to DNS/connection string caching

**How to determine if it occurred**

The most direct indicators are operational:
- You observe successful writes on **two nodes** during the same incident window.
- Both nodes report `pg_is_in_recovery() = false` (both acting as primary).

Useful technical signals:
- Replication attempts fail with errors like “requested timeline is not in this server’s history”.
- The nodes show different WAL timelines (first 8 hex chars of WAL filename differ):

**📖 Example:** Check timeline quickly on each node

> ```sql
> SELECT
>   pg_is_in_recovery() AS is_standby,
>   pg_walfile_name(pg_current_wal_lsn()) AS wal_file;
> ```

If the `wal_file` timeline differs between nodes that should be in the same cluster history, you have a divergence.

**Recovery approach (high-level)**
- **Pick a winner** primary (the one you will keep).
- **Fence** the other node immediately (stop Postgres, block network access, remove it from service discovery).
- Reconfigure the losing node as a standby:
  - Try **`pg_rewind`** (fastest) if you have sufficient WAL and the timelines allow rewind.
  - Otherwise, take a new base backup and rebuild the replica.
- **Reconcile data**: any writes that happened on the losing side must be recovered at the application level (replay from audit logs, queues, or business-system reconciliation). Physical replication will not merge those writes.

**Prevention**
- Use a reliable failover manager with strong fencing semantics (STONITH / power fencing where possible).
- Prefer a single source of truth for primary identity (DCS/consensus) and make clients follow it.
- Monitor for “two primaries” conditions and page immediately.

### Planned Switchover

1. Stop writes on primary
2. Wait for standby to catch up
3. Promote standby
4. Reconfigure old primary as standby

**🔬 Try It:**
```sql
-- On primary: check all WAL is sent
SELECT client_addr, sent_lsn = flush_lsn AS caught_up
FROM pg_stat_replication;

-- On standby: verify caught up
SELECT pg_last_wal_replay_lsn() = pg_last_wal_receive_lsn();
```

### Using pg_rewind for Failback

After failover, rewind old primary to become standby:

```bash
# Stop old primary
pg_ctl stop -D /old_primary

# Rewind to match new primary's timeline
pg_rewind --target-pgdata=/old_primary --source-server="host=new_primary"

# Configure as standby
touch /old_primary/standby.signal
# Add primary_conninfo to postgresql.auto.conf

# Start as standby
pg_ctl start -D /old_primary
```

## Part 5: Replication Slots

### Physical Slots

**🔬 Try It:**
```sql
-- Create physical slot
SELECT pg_create_physical_replication_slot('standby1');

-- Drop slot (important: releases WAL!)
SELECT pg_drop_replication_slot('standby1');

-- View all slots
SELECT 
    slot_name,
    slot_type,
    active,
    active_pid,
    restart_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

### Slot Management

**Warning**: Inactive slots retain WAL forever!

**🔬 Try It:**
```sql
-- Find inactive slots with high WAL retention
SELECT slot_name, active, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots
WHERE NOT active;

-- Set maximum WAL retained by slots (PostgreSQL 13+)
-- max_slot_wal_keep_size = 100GB  -- In postgresql.conf
```

## Part 6: Logical Replication

**Source**: `src/backend/replication/logical/`

Logical replication decodes WAL into row changes:

### Setting Up Publisher

**🔬 Try It:**
```sql
-- On publisher
-- wal_level = logical  (in postgresql.conf)

-- Create publication
CREATE PUBLICATION my_pub FOR TABLE employees, departments;

-- Or all tables
-- CREATE PUBLICATION all_tables FOR ALL TABLES;

-- View publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

### Setting Up Subscriber

**🔬 Try It:**
```sql
-- On subscriber (must create tables first!)
-- CREATE TABLE employees (...);  -- Same structure
-- CREATE TABLE departments (...);

-- Create subscription
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=publisher dbname=learning user=replicator password=secret'
PUBLICATION my_pub;

-- View subscriptions
SELECT * FROM pg_subscription;
SELECT * FROM pg_stat_subscription;
```

### Logical Replication Features

**📖 Example:**
```sql
-- Publish only specific operations
CREATE PUBLICATION insert_only FOR TABLE orders
WITH (publish = 'insert');  -- Only INSERT operations

-- Publish with row filter (PostgreSQL 15+)
CREATE PUBLICATION us_only FOR TABLE customers
WHERE (country = 'US');

-- Publish specific columns (PostgreSQL 15+)
CREATE PUBLICATION public_data FOR TABLE users (id, name, email);
```

### Monitoring Logical Replication

**🔬 Try It:**
```sql
-- On publisher: view replication connections
SELECT * FROM pg_stat_replication WHERE application_name LIKE 'pg_%';

-- On subscriber: view subscription status
SELECT 
    subname,
    received_lsn,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;

-- Replication lag
SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), received_lsn)
) AS lag
FROM pg_stat_subscription;
```

## Part 7: Replication Conflicts and Resolution

### Physical Replication Conflicts

Hot standby can have query/recovery conflicts:

**🔬 Try It:**
```sql
SELECT * FROM pg_stat_database_conflicts;
```

Most “conflicts” here are not data conflicts—they’re **standby replay vs. long-running queries**.
Two common mitigation settings (configured on the standby) are:

- **`max_standby_streaming_delay`**: how long standby is willing to delay WAL replay to let queries finish
- **`hot_standby_feedback`**: reduces query cancellations but can increase bloat on the primary

**📖 Example:** Standby settings (in `postgresql.conf`)

> ```conf
> max_standby_streaming_delay = 30s
> hot_standby_feedback = on
> ```

### Logical Replication Conflicts

The “disable subscription / advance origin” technique is a last-resort recovery technique. Use it only when you understand which changes you are skipping and why.

**🔬 Try It:**
```sql
ALTER SUBSCRIPTION my_sub DISABLE;
-- Fix issue...
SELECT pg_replication_origin_advance('pg_<subscription_oid>', 'target_lsn');
ALTER SUBSCRIPTION my_sub ENABLE;
```

Logical replication conflicts are typically **data consistency** issues between publisher/subscriber.
Before “skipping” anything, check subscriber logs and `pg_stat_subscription` to understand what failed.

## Part 8: Cascading Replication

Standbys can cascade to other standbys:

This is a practical scaling tool: you can keep the primary focused on write workload while allowing read scale-out and geographic distribution through “fan-out” from an intermediate standby.

```
Primary → Standby1 → Standby2
                   → Standby3
```

**📖 Example:** Key settings for Standby1 to act as an upstream

> ```conf
> # On Standby1
> max_wal_senders = 5
> ```

**📖 Example:** Point Standby2 at Standby1

> ```conf
> # On Standby2
> primary_conninfo = 'host=standby1 user=replicator password=secret'
> ```

Benefits:
- Reduce load on primary
- Geographic distribution
- Hierarchical architecture

## Part 9: Delayed Replication

Intentionally delay standby for accidental deletion recovery:

**🔬 Try It:**
```sql
-- On standby (postgresql.conf or postgresql.auto.conf)
-- recovery_min_apply_delay = '1h'  -- 1 hour delay

-- Check current delay
SELECT 
    now() - pg_last_xact_replay_timestamp() AS actual_delay,
    '1 hour'::interval AS configured_delay;
```

Use case: If bad data is committed, you have 1 hour to recover from delayed standby.

## Part 10: Monitoring Replication

### Comprehensive Monitoring Query

**🔬 Try It:**
```sql
-- Replication overview
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)) AS send_lag,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, write_lsn)) AS write_lag,
    pg_size_pretty(pg_wal_lsn_diff(write_lsn, flush_lsn)) AS flush_lag,
    pg_size_pretty(pg_wal_lsn_diff(flush_lsn, replay_lsn)) AS replay_lag,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS total_lag
FROM pg_stat_replication;
```

### Replication Alerts

**🔬 Try It:**
```sql
-- Check for lagging standbys
SELECT slot_name
FROM pg_replication_slots
WHERE NOT active
   OR pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 1073741824;  -- 1GB

-- Check for conflict-prone standbys
SELECT datname, confl_tablespace + confl_lock + confl_snapshot + 
       confl_bufferpin + confl_deadlock AS total_conflicts
FROM pg_stat_database_conflicts
WHERE confl_tablespace + confl_lock + confl_snapshot + 
      confl_bufferpin + confl_deadlock > 0;
```

## Source Code References

| Component | Source Directory |
|-----------|-----------------|
| Streaming replication | `src/backend/replication/` |
| WAL sender | `src/backend/replication/walsender.c` |
| WAL receiver | `src/backend/replication/walreceiver.c` |
| Logical decoding | `src/backend/replication/logical/` |
| Slot management | `src/backend/replication/slot.c` |

## Summary

In this chapter, you learned:

1. **Streaming Replication**: Physical replication via WAL streaming
2. **Synchronous Replication**: Durability guarantees across replicas
3. **Failover/Switchover**: Promoting standby, planned transitions
4. **Replication Slots**: Ensuring WAL retention
5. **Logical Replication**: Row-level, selective replication
6. **Conflict Handling**: Physical and logical conflicts
7. **Cascading**: Hierarchical replication topologies
8. **Delayed Replication**: Protection against mistakes
9. **Monitoring**: Tracking lag and health

## Quick Reference

```sql
-- Physical replication slot
SELECT pg_create_physical_replication_slot('name');

-- Logical publication/subscription
CREATE PUBLICATION pub FOR TABLE t1, t2;
CREATE SUBSCRIPTION sub CONNECTION '...' PUBLICATION pub;

-- Check replication status
SELECT * FROM pg_stat_replication;
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_replication_slots;

-- Promote standby
SELECT pg_promote();

-- WAL position functions
SELECT pg_current_wal_lsn();
SELECT pg_last_wal_receive_lsn();
SELECT pg_last_wal_replay_lsn();
```

---

**Previous Chapter:** [← WAL](09-wal.md)

**Next Chapter:** [Server-side Programming →](11-server-side-programming.md)

