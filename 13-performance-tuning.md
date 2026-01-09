# Chapter 13: Performance Tuning

## Learning Objectives

By the end of this chapter, you will be able to:
- Identify and diagnose performance bottlenecks
- Optimize queries using EXPLAIN ANALYZE
- Configure PostgreSQL for optimal performance
- Design schemas for performance
- Implement caching strategies

## Part 1: Performance Methodology

### The Tuning Process

The most important performance skill is avoiding “random tuning.” PostgreSQL is a complex system, so you want a repeatable loop: measure, change one thing, and verify.

```
1. MEASURE: Identify what's slow
   └─ Use pg_stat_statements, logs, monitoring
   
2. ANALYZE: Find the root cause
   └─ Use EXPLAIN ANALYZE, system stats
   
3. OPTIMIZE: Apply targeted fixes
   └─ Indexes, queries, configuration
   
4. VERIFY: Confirm improvement
   └─ Compare before/after metrics
   
5. REPEAT: Performance is ongoing
```

**🔬 Try It:** Check Key Metrics

```sql
-- Connection pressure
SELECT count(*) AS current_connections
FROM pg_stat_activity;

-- Background writer activity (not checkpoints; see Chapter 9 for checkpointer stats)
SELECT buffers_clean, maxwritten_clean, buffers_alloc, stats_reset
FROM pg_stat_bgwriter;

-- WAL activity
SELECT * FROM pg_stat_wal;

-- Checkpoint activity (PostgreSQL 17/18)
SELECT num_timed, num_requested, write_time, sync_time, buffers_written, stats_reset
FROM pg_stat_checkpointer;
```

**📖 Example:** If `pg_stat_statements` is enabled, use it to find slow/expensive queries

> ```sql
> -- Response time: How long queries take (requires pg_stat_statements)
> SELECT mean_exec_time, calls, rows
> FROM pg_stat_statements
> WHERE query LIKE '%SELECT%'
> ORDER BY mean_exec_time DESC
> LIMIT 20;
>
> -- Throughput: Queries per second since stats reset
> SELECT sum(calls) / EXTRACT(EPOCH FROM (now() - stats_reset)) AS qps
> FROM pg_stat_statements, pg_stat_statements_info;
> ```

## Part 2: Query Optimization

### EXPLAIN ANALYZE Deep Dive

Think of `EXPLAIN` as “what the planner *thinks* it will do,” and `EXPLAIN ANALYZE` as “what *actually happened*.” When those differ, you’ve found the most valuable debugging signal in PostgreSQL performance work.

**🔬 Try It:**
```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING, VERBOSE, FORMAT TEXT)
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 80000;
```

**Key things to look for:**
- Actual vs estimated rows (off by >10x needs ANALYZE)
- Sequential scans on large tables
- High buffer reads (disk I/O)
- Sort operations that spill to disk
- Nested loops with many iterations
- Through-away work where a child node is passing an excessive amount of rows that get filtered out or not passed on by the parent

### Common Query Problems

This section is a collection of patterns you’ll see in production. The goal isn’t to memorize fixes—it’s to recognize symptoms (plan shape + row estimates + I/O) and know the first diagnostic step to take.

**1. Missing Indexes**

**📖 Example:**
```sql
-- Sequential scan on large table
Seq Scan on timecards  (cost=0.00..25000.00 rows=500000)
   Filter: (emp_id = 42 AND status = 'approved')
   Rows Removed by Filter: 499999

-- Solution: Add index
CREATE INDEX idx_timecards_emp_status ON timecards(emp_id, status);
```

**2. Wrong Join Order**

**📖 Example:** What this looks like (plan-shape varies by data and settings)

Symptom: large outer loops, many rows produced then filtered away upstream

Fixes are usually data-driven:
- ANALYZE after large changes
- add/select better indexes
- rewrite the query (e.g., pre-filter earlier)

> ```sql
> -- In real tuning, avoid “forcing” join order unless you have a very specific reason.
> ANALYZE some_large_table;
> ```

**3. Poor Selectivity Estimates**

**📖 Example:** When estimates are off (illustrative)

Symptom: Estimated rows: 100, Actual rows: 100000

Fix: refresh stats; optionally raise stats target for important columns

> ```sql
> ANALYZE some_table;
> ALTER TABLE some_table ALTER COLUMN some_col SET STATISTICS 1000;
> ANALYZE some_table;
> ```

**4. Correlated Subqueries**

**📖 Example:**
```sql
-- Slow: Subquery runs for each row
SELECT e.emp_id,
       e.first_name || ' ' || e.last_name AS employee,
       (SELECT count(*) FROM timecards t WHERE t.emp_id = e.emp_id) AS timecards_count
FROM employees e;

-- Better: Use JOIN
SELECT e.emp_id,
       e.first_name || ' ' || e.last_name AS employee,
       count(t.timecard_id) AS timecards_count
FROM employees e
LEFT JOIN timecards t ON t.emp_id = e.emp_id
GROUP BY e.emp_id, e.first_name, e.last_name;
```

**5. N+1 Queries**

**📖 Example:**
```sql
-- Bad: Query per employee (in application code)
-- for employee in employees:
--     timecards = SELECT * FROM timecards WHERE emp_id = ?

-- Better: Batch query
SELECT * FROM timecards WHERE emp_id IN (1,2,3,4,5);

-- Or join in single query
SELECT e.*, t.* FROM employees e
JOIN timecards t ON e.emp_id = t.emp_id;
```

### Index Optimization

Indexes are one of the highest-leverage performance tools, but you want to add and keep them **based on evidence**.

**🔬 Try It:** Find tables that are being sequentially scanned heavily (index candidates)

```sql
SELECT
    relname,
    seq_scan,
    idx_scan,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_seq_rows
FROM pg_stat_user_tables
WHERE seq_scan > 10
  AND seq_tup_read / NULLIF(seq_scan, 0) > 10000
ORDER BY seq_tup_read DESC;
```

This query helps you find tables where PostgreSQL is repeatedly scanning lots of rows. In our HR storyline, you’ll often see large tables like `timecards` show up here.

**🔬 Try It:** Find unused indexes (candidates for removal)

```sql
SELECT
    indexrelid::regclass AS index,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint);
```

**Index-only scans** depend on the visibility map: PostgreSQL can only skip heap reads for pages marked all-visible.
If you expect index-only scans but aren’t getting them, a `VACUUM` often helps.

## Part 3: Configuration Tuning

### Memory Configuration

**🔬 Try It:**
```sql
SELECT name, setting, unit 
FROM pg_settings 
WHERE name IN ('shared_buffers', 'effective_cache_size', 'work_mem', 'maintenance_work_mem');
```

These four settings are the “big knobs” for memory behavior:

- **`shared_buffers`**: PostgreSQL’s main buffer cache (data pages). A common starting point is ~25% of RAM.
- **`effective_cache_size`**: what PostgreSQL *assumes* is cached by the OS + shared buffers (planner hint). Often 50–75% of RAM.
- **`work_mem`**: memory per sort/hash *operation* (can be used multiple times per query). Mis-sizing this is one of the most common causes of memory pressure.
- **`maintenance_work_mem`**: memory for maintenance operations like `VACUUM` and `CREATE INDEX`.

**📖 Example:** Illustrative settings (validate with your workload)

> ```sql
> ALTER SYSTEM SET shared_buffers = '4GB';
> ALTER SYSTEM SET effective_cache_size = '12GB';
> ALTER SYSTEM SET work_mem = '64MB';
> ALTER SYSTEM SET maintenance_work_mem = '1GB';
> SELECT pg_reload_conf();
> ```

### Connection Configuration

Connections are expensive in PostgreSQL because each connection typically maps to a backend process.
In production, you usually keep `max_connections` reasonable and use a pooler (e.g., PgBouncer).

**🔬 Try It:** Inspect connection limits and current usage

```sql
SHOW max_connections;
SHOW superuser_reserved_connections;
SELECT count(*) AS current_connections FROM pg_stat_activity;
```

**📖 Example:** Lower connections + use pooling

> ```sql
> ALTER SYSTEM SET max_connections = 100;
> SELECT pg_reload_conf();
> ```

### Checkpoint Tuning

Checkpoint tuning is about reducing “I/O spikes” and avoiding overly frequent checkpoints.  Managing checkpoints can make data loads more efficient by reducing the number of full page writes.

Two common levers:

- **`checkpoint_completion_target`**: spreads checkpoint writes over more time (smoother disk usage)
- **`max_wal_size` / `min_wal_size`**: controls how much WAL can accumulate before a checkpoint is forced

**🔬 Try It:** Check how often checkpoints are happening

```sql
-- Checkpoint stats live here in PostgreSQL 17/18
SELECT num_timed, num_requested, write_time, sync_time, buffers_written, stats_reset
FROM pg_stat_checkpointer;
```

If `num_requested` is frequently increasing, PostgreSQL is being forced to checkpoint (commonly due to WAL pressure / `max_wal_size`).

**📖 Example:** Typical checkpoint settings for write-heavy workloads (apply with care)

> ```sql
> ALTER SYSTEM SET checkpoint_completion_target = 0.9;
> ALTER SYSTEM SET max_wal_size = '4GB';
> ALTER SYSTEM SET min_wal_size = '1GB';
> SELECT pg_reload_conf();
> ```

### Planner Configuration

Planner tuning should be done carefully and validated with `EXPLAIN (ANALYZE, BUFFERS)`.
Many “bad plans” come from incorrect cost assumptions, especially around storage.

**📖 Example:** Common planner settings (illustrative)

> ```sql
> -- SSD vs HDD cost assumptions
> ALTER SYSTEM SET random_page_cost = 1.1;          -- SSD-ish
> ALTER SYSTEM SET effective_io_concurrency = 200;  -- SSD-ish
> 
> -- Parallel query capacity (depends on CPU and workload)
> ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
> ALTER SYSTEM SET max_parallel_workers = 8;
> ALTER SYSTEM SET parallel_tuple_cost = 0.01;
> ALTER SYSTEM SET parallel_setup_cost = 1000;
> 
> SELECT pg_reload_conf();
> ```

**🔬 Try It:** Inspect current values

```sql
SHOW random_page_cost;
SHOW effective_io_concurrency;
SHOW max_parallel_workers_per_gather;
SHOW max_parallel_workers;
```

## Part 4: Schema Design for Performance

### Normalization vs Denormalization

**📖 Example:**
```sql
-- Normalized (good for writes): separate employee identity from time entries
CREATE TABLE employees (emp_id, first_name, last_name, email);
CREATE TABLE timecards (timecard_id, emp_id, work_date, hours_worked);

-- Denormalized (good for reads): embed employee name into the time entry (trade-offs!)
CREATE TABLE timecards (timecard_id, emp_id, employee_name, work_date, hours_worked);
```

### Partitioning

**🔬 Try It:** Partition pruning with an HR-themed time-series table

```sql
-- Create a small, self-contained partitioned table for the demo so it doesn't
-- conflict with other chapters (and doesn't require repartitioning `timecards`).
--
-- Note: on partitioned tables, PRIMARY KEY / UNIQUE constraints must include the
-- partition key (`created_at`) so PostgreSQL can enforce uniqueness across partitions.
DROP TABLE IF EXISTS timecard_events_part_demo CASCADE;

CREATE TABLE timecard_events_part_demo (
    id BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    emp_id INT NOT NULL,
    event JSONB NOT NULL
    ,
    PRIMARY KEY (created_at, id)
) PARTITION BY RANGE (created_at);

CREATE TABLE timecard_events_2024_01 PARTITION OF timecard_events_part_demo
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE timecard_events_2024_02 PARTITION OF timecard_events_part_demo
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Seed a few rows so the plan isn't entirely hypothetical
INSERT INTO timecard_events_part_demo (created_at, emp_id, event)
VALUES
  ('2024-01-15 10:00:00+00', (SELECT MIN(emp_id) FROM employees), '{"type":"submitted"}'),
  ('2024-02-10 12:00:00+00', (SELECT MIN(emp_id) FROM employees), '{"type":"approved"}');

-- Queries automatically prune partitions based on the date range
EXPLAIN (COSTS OFF)
SELECT *
FROM timecard_events_part_demo
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-20';
```

### Materialized Views

**📖 Example:**
```sql
-- Create materialized view for expensive aggregations
CREATE MATERIALIZED VIEW daily_hours AS
SELECT 
    work_date AS day,
    sum(hours_worked) AS total_hours,
    count(*) AS timecards
FROM timecards
GROUP BY work_date;

-- Index the materialized view
CREATE INDEX ON daily_hours(day);

-- Select
SELECT * FROM daily_hours WHERE day = '2025-12-31';

-- Refresh periodically
REFRESH MATERIALIZED VIEW daily_hours;
```

### Efficient Data Types

**📖 Example:** Data types that affect performance (illustrative)

> ```sql
> -- Use appropriate integer sizes
> -- SMALLINT: 2 bytes, -32768 to 32767
> -- INTEGER: 4 bytes
> -- BIGINT: 8 bytes
>
> -- UUIDs are great for distributed ID generation, but larger than BIGINT keys
> -- (and can increase index size / cache pressure).
> CREATE TABLE some_events (
>     id UUID DEFAULT gen_random_uuid()
> );
> ```

## Part 5: Write Optimization

### Batch Operations

**📖 Example:** Batch writes (illustrative; replace `table_name` with a real table)

> ```sql
> -- Slow: Individual inserts
> INSERT INTO table_name VALUES (1);
> INSERT INTO table_name VALUES (2);
> INSERT INTO table_name VALUES (3);
>
> -- Fast: Batch insert
> INSERT INTO table_name VALUES (1), (2), (3);
> ```

**📖 Example:** `COPY` is typically fastest for bulk load (shell/file dependent)

> ```sql
> COPY table_name FROM '/path/to/file.csv' WITH CSV HEADER;
> ```

### Unlogged Tables

**📖 Example:** Unlogged tables (illustrative)

> ```sql
> -- For temporary/regenerable data (not crash-safe)
> CREATE UNLOGGED TABLE cache_demo (
>     key TEXT PRIMARY KEY,
>     value JSONB
> );
> ```

### Bulk Loading Optimization

**📖 Example:** Bulk load checklist (illustrative)

> ```sql
> -- Disable or defer expensive indexes/constraints (case-by-case)
> -- Load data (often via COPY)
> -- Recreate indexes
> -- ANALYZE to refresh planner statistics
> ```

## Part 6: Read Optimization

### Connection Pooling

**📖 Example:** PgBouncer configuration (illustrative)

```bash
# PgBouncer configuration (pgbouncer.ini)
[databases]
learning = host=127.0.0.1 port=5432 dbname=learning

[pgbouncer]
listen_port = 6432
auth_type = md5
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

### Query Caching

**📖 Example:** Application-level caching pattern (illustrative)

> ```sql
> -- Keep cached results in a dedicated table
> CREATE TABLE report_cache (
>     key TEXT PRIMARY KEY,
>     value JSONB,
>     expires_at TIMESTAMPTZ
> );
>
> CREATE INDEX ON report_cache(expires_at);
>
> -- Invalidate cache keys when underlying tables change (requires careful design)
> CREATE OR REPLACE FUNCTION invalidate_cache()
> RETURNS TRIGGER AS $$
> BEGIN
>     DELETE FROM report_cache WHERE key LIKE TG_TABLE_NAME || ':%';
>     RETURN NULL;
> END;
> $$ LANGUAGE plpgsql;
> ```

### Read Replicas

**📖 Example:** Replica lag check (run on a standby)

> ```sql
> SELECT pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() AS caught_up;
> ```

## Part 7: Monitoring for Performance

### Key Performance Indicators

**🔬 Try It:**
```sql
-- Create monitoring view
CREATE OR REPLACE VIEW performance_metrics AS
SELECT
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_queries,
    (SELECT count(*) FROM pg_stat_activity WHERE wait_event IS NOT NULL) AS waiting_queries,
    (SELECT round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2) 
     FROM pg_stat_database) AS cache_hit_ratio,
    (SELECT round(100.0 * sum(idx_scan) / NULLIF(sum(idx_scan) + sum(seq_scan), 0), 2)
     FROM pg_stat_user_tables) AS index_usage_ratio,
    (SELECT max(now() - xact_start) FROM pg_stat_activity WHERE state = 'active') AS longest_query;

SELECT * FROM performance_metrics;
```

### Alerting Thresholds

**🔬 Try It:**
```sql
-- Long-running queries (>1 minute)
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' 
  AND now() - query_start > INTERVAL '1 minute';

-- Connection saturation
SELECT 
    count(*) AS current,
    current_setting('max_connections')::INT AS max,
    round(100.0 * count(*) / current_setting('max_connections')::INT, 2) AS pct
FROM pg_stat_activity;

-- Bloated tables
SELECT relname, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

## Part 8: Benchmarking

### pgbench

```bash
# Initialize test database
pgbench -i -s 100 testdb  # Scale factor 100 (~1.5GB)

# Run benchmark
pgbench -c 10 -j 2 -T 60 testdb
# -c: clients (connections)
# -j: threads
# -T: duration in seconds

# Custom benchmark
pgbench -f custom.sql -c 10 -T 60 testdb
```

### Creating Custom Benchmarks

**📖 Example:**
```sql
-- custom.sql
\set id random(1, 500000)
SELECT * FROM timecards WHERE timecard_id = :id;
UPDATE timecards SET hours_worked = hours_worked WHERE timecard_id = :id;
```
 
> This is intentionally “no-op” on the UPDATE to avoid distorting the dataset; it’s mainly a pattern for wiring pgbench to an existing table.

## Part 9: Performance Checklist

### Quick Wins
- [ ] Run ANALYZE on all tables
- [ ] Check for missing indexes on foreign keys
- [ ] Enable pg_stat_statements
- [ ] Review random_page_cost for SSDs
- [ ] Configure work_mem appropriately

### Database Level
- [ ] shared_buffers = 25% of RAM
- [ ] effective_cache_size = 50-75% of RAM
- [ ] Checkpoint tuning for write-heavy workloads
- [ ] Connection pooling if >100 connections

### Query Level
- [ ] EXPLAIN ANALYZE suspicious queries
- [ ] Check estimated vs actual rows
- [ ] Add indexes for frequently filtered columns
- [ ] Avoid SELECT * when possible
- [ ] Use covering indexes for frequent queries

### Schema Level
- [ ] Partition large tables
- [ ] Use appropriate data types
- [ ] Consider materialized views for aggregations
- [ ] Review normalization level

## Summary

In this chapter, you learned:

1. **Methodology**: Measure, Analyze, Optimize, Verify
2. **Query Optimization**: EXPLAIN ANALYZE, common problems
3. **Index Strategy**: When and what to index
4. **Configuration**: Memory, connections, checkpoints
5. **Schema Design**: Partitioning, denormalization
6. **Write Optimization**: Batching, bulk loading
7. **Read Optimization**: Caching, pooling, replicas
8. **Monitoring**: KPIs and alerting
9. **Benchmarking**: pgbench, custom tests

## Chapter Cleanup

This chapter creates a small number of objects for demonstration. If you want to remove them:

**🔬 Try It:** Clean up demonstration objects

```sql
DROP VIEW IF EXISTS performance_metrics;

DROP TABLE IF EXISTS timecard_events_part_demo CASCADE;
```

---

**Previous Chapter:** [← Security](12-security.md)

**Next Chapter:** [Utility Tools →](14-utility-tools.md)

