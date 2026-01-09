# Chapter 15: Autovacuum, Freezing, and Bloat

## Learning Objectives

By the end of this chapter, you will be able to:
- Explain why PostgreSQL needs VACUUM (MVCC + dead tuples)
- Understand autovacuum thresholds, cost-based throttling, and worker behavior
- Diagnose table/index bloat symptoms and know the first remediation steps
- Explain transaction ID wraparound and how freezing prevents it
- Connect SQL-level behavior to the key autovacuum/vacuum code paths

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

This chapter benefits from having a larger table like `timecards` (Chapter 7), but it also includes a small demo table so you can see behaviors quickly.

---

## Part 1: Why VACUUM Exists (MVCC Creates Dead Tuples)

In PostgreSQL, `UPDATE` and `DELETE` do not overwrite rows in place. They create new tuple versions and leave old versions behind for other transactions that might still need them.

That’s MVCC’s core trade-off:
- **Pros**: readers don’t block writers (and vice versa)
- **Cost**: old row versions accumulate until vacuum cleans them up

---

## Part 2: Key Observability Metrics (Dead Tuples, Vacuum History)

**🔬 Try It:** Spot dead tuples and vacuum activity

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

How to interpret this:
- **`n_dead_tup` rising over time** is normal for update-heavy tables; it’s a signal to check autovacuum effectiveness.
- **`last_autovacuum` missing** on busy tables can be a smell (settings too conservative, or vacuum can’t keep up).

---

## Part 3: Autovacuum Basics (Thresholds and Cost Throttling)

Autovacuum runs per table when “dead tuple pressure” crosses a threshold:
- `autovacuum_vacuum_threshold` (base)
- `autovacuum_vacuum_scale_factor` (percentage of table)

It’s also rate-limited by cost-based settings so it doesn’t monopolize I/O:
- `autovacuum_vacuum_cost_limit`
- `autovacuum_vacuum_cost_delay`

**🔬 Try It:** Inspect autovacuum configuration

```sql
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
  'autovacuum',
  'autovacuum_max_workers',
  'autovacuum_naptime',
  'autovacuum_vacuum_threshold',
  'autovacuum_vacuum_scale_factor',
  'autovacuum_analyze_threshold',
  'autovacuum_analyze_scale_factor',
  'autovacuum_vacuum_cost_limit',
  'autovacuum_vacuum_cost_delay'
)
ORDER BY name;
```

---

## Part 4: A Small, Safe Demo (Dead Tuples → VACUUM)

This demo creates a tiny table to make “dead tuples” and vacuum effects observable without stressing your main HR tables.

**🔬 Try It:** Create a vacuum demo table

```sql
DROP TABLE IF EXISTS vacuum_demo;
CREATE TABLE vacuum_demo (
  id BIGSERIAL PRIMARY KEY,
  payload TEXT NOT NULL
) WITH (autovacuum_enabled = false);

INSERT INTO vacuum_demo(payload)
SELECT 'row ' || i
FROM generate_series(1, 50000) AS i;

ANALYZE vacuum_demo;
```

**🔬 Try It:** Create dead tuples

```sql
-- Update many rows (creates new tuple versions)
UPDATE vacuum_demo
SET payload = payload || ' updated'
WHERE id % 2 = 0;

-- Delete some rows (creates dead tuples too)
DELETE FROM vacuum_demo
WHERE id % 10 = 0;
```

**🔬 Try It:** Observe dead tuples and vacuum impact

```sql
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'vacuum_demo';

VACUUM (ANALYZE) vacuum_demo;

SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'vacuum_demo';
```

Notes:
- Regular `VACUUM` makes space reusable inside PostgreSQL; it usually does **not** return space to the OS.
- `VACUUM (ANALYZE)` also refreshes statistics, which can significantly change plans.

---

## Part 5: Freezing and Transaction ID Wraparound

Transaction IDs are finite (32-bit) and wrap around. PostgreSQL must “freeze” very old tuples so their visibility metadata remains correct across wraparound.

**🔬 Try It:** Check wraparound risk at the database level

```sql
SELECT
  datname,
  age(datfrozenxid) AS xid_age,
  current_setting('autovacuum_freeze_max_age')::bigint AS freeze_max_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

What to look for:
- If `xid_age` approaches `freeze_max_age`, autovacuum should start being aggressive. If it can’t keep up, you’re heading toward emergency vacuum behavior.

---

## Part 6: Bloat (What It Is and What It Isn’t)

“Bloat” is overloaded. In practice it usually means one (or more) of:
- **Table bloat**: dead tuples and free space inside relation files
- **Index bloat**: indexes growing because of churn (esp. non-HOT updates)
- **TOAST bloat**: large values updated frequently

First-response checklist:
- Ensure autovacuum is running and not starved
- Confirm statistics are up to date (`ANALYZE`)
- Identify the churny tables and hot update patterns
- Only then consider heavy operations (`VACUUM FULL`, `CLUSTER`, `REINDEX`)

---

## Source Code Connection

Key places to read while studying this chapter:
- Autovacuum launcher/workers: `src/backend/postmaster/autovacuum.c`
- Manual VACUUM command: `src/backend/commands/vacuum.c`
- Heap visibility and pruning: `src/backend/access/heap/heapam_visibility.c`, `src/backend/access/heap/pruneheap.c`
- Transaction IDs / wraparound concepts: `src/backend/access/transam/`, `src/include/access/transam.h`
- Visibility map (index-only scans and vacuum interaction): `src/backend/access/heap/visibilitymap.c`

---

## Summary

In this chapter, you learned:
- MVCC creates dead tuples; VACUUM reclaims them for reuse
- Autovacuum is essential operational infrastructure, not an optional feature
- Freezing prevents transaction ID wraparound failures
- “Bloat” is usually a symptom; fix the workload + autovacuum first, then rebuild objects if needed

---

## Chapter Cleanup

**🔬 Try It:** Remove demo objects

```sql
DROP TABLE IF EXISTS vacuum_demo;
```

---

**Previous Chapter:** [Utility Tools ←](14-utility-tools.md)

**Next Chapter:** [Extension Development & Server Extensibility →](17-extension-development.md)  <!-- Chapter 16 -->

**Return to:** [Curriculum Index](README.md)


