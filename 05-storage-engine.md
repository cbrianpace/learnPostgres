# Chapter 5: The Storage Engine

## Learning Objectives

By the end of this chapter, you will be able to:
- Understand how PostgreSQL stores data on disk
- Explain the structure of heap files and pages
- Describe how tuples are organized within pages
- Understand TOAST for large values
- Work with the Free Space Map and Visibility Map
- Use pageinspect to examine page internals

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

This chapter references the `employees` table created in Chapter 2.

## Part 1: Storage Hierarchy

PostgreSQL organizes storage in a hierarchy:

```
Cluster (Data Directory)
└── Database
    └── Tablespace
        └── Schema
            └── Table (Relation)
                └── Data Files (main, fsm, vm)
                    └── Page (Block) - 8KB default
                        └── Tuple (Row)
```

### File Organization

Each table is stored as multiple related files:

| File | Suffix | Purpose |
|------|--------|---------|
| **Main data file** | (none) | The actual table rows (heap) |
| **Free Space Map** | `_fsm` | Tracks available space in each page |
| **Visibility Map** | `_vm` | Tracks which pages are fully visible to all transactions |

> **Source code note:** PostgreSQL internally calls these related files "forks" (see `src/include/common/relpath.h`). You'll see this term in documentation and function names like `pg_relation_filepath()`.

**🔬 Try It:**
```sql
-- Find the main data file path for a table
SELECT pg_relation_filepath('employees');
-- Returns something like: base/16384/16385
```

Given a path like `base/16384/16385`, PostgreSQL stores related files alongside it:
- **Main data**: `base/16384/16385`
- **Free Space Map (FSM)**: `base/16384/16385_fsm`
- **Visibility Map (VM)**: `base/16384/16385_vm`

### The 1GB File Limit

PostgreSQL splits large tables into 1GB segments:

For a large table, you might see:
  base/16384/16385      (first 1GB)
  base/16384/16385.1    (second 1GB)
  base/16384/16385.2    (third 1GB)

## Part 2: Page Structure

Every page in PostgreSQL is 8KB (configurable at compile time):

```
┌────────────────────────────────────────────────────────────┐
│                     Page Header (24 bytes)                 │
├────────────────────────────────────────────────────────────┤
│                     Item Pointers (Line Pointers)          │
│              [LP1][LP2][LP3]...[LPn] → grows down          │
├────────────────────────────────────────────────────────────┤
│                                                            │
│                     Free Space                             │
│                                                            │
├────────────────────────────────────────────────────────────┤
│              ← grows up   [Tuple n]...[Tuple 2][Tuple 1]   │
│                           Tuples                           │
├────────────────────────────────────────────────────────────┤
│                     Special Space (indexes only)           │
└────────────────────────────────────────────────────────────┘
```

### Page Header Fields

**File**: `src/include/storage/bufpage.h`

| Field | Size | Description |
|-------|------|-------------|
| pd_lsn | 8 bytes | Last WAL LSN affecting this page |
| pd_checksum | 2 bytes | Page checksum |
| pd_flags | 2 bytes | Flag bits |
| pd_lower | 2 bytes | Offset to start of free space |
| pd_upper | 2 bytes | Offset to end of free space |
| pd_special | 2 bytes | Offset to special space |
| pd_pagesize_version | 2 bytes | Page size and version |
| pd_prune_xid | 4 bytes | Oldest prunable XID |

**🔬 Try It:** Explore Pages with pageinspect

```sql
-- Install the extension
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Create a test table
CREATE TABLE page_demo (id SERIAL, name TEXT, value INTEGER);
INSERT INTO page_demo (name, value) 
SELECT 'item_' || i, i * 10 
FROM generate_series(1, 100) AS i;

-- View page header
SELECT * FROM page_header(get_raw_page('page_demo', 0));

-- lsn           | checksum | flags | lower | upper | special | pagesize | version | prune_xid
-- 0/1A234F80    |        0 |     0 |   428 |  5664 |    8192 |     8192 |       4 |         0
```

**Understanding the output:**
- `lower = 428`: Item pointers use bytes 24-428
- `upper = 5664`: Tuples start at byte 5664
- `special = 8192`: No special space (heap page)
- Free space: 5664 - 428 = 5236 bytes

### Line Pointers (Item Pointers)

Each line pointer is 4 bytes and points to a tuple:

**🔬 Try It:**
```sql
-- View line pointers
SELECT lp, lp_off, lp_flags, lp_len 
FROM heap_page_items(get_raw_page('page_demo', 0))
LIMIT 10;

-- lp | lp_off | lp_flags | lp_len
-- ---+--------+----------+-------
--  1 |   8160 |        1 |     32
--  2 |   8128 |        1 |     32
--  3 |   8096 |        1 |     32
```

**lp_flags meanings:**
- 0 = LP_UNUSED (available for reuse)
- 1 = LP_NORMAL (normal tuple)
- 2 = LP_REDIRECT (HOT redirect)
- 3 = LP_DEAD (dead, can be reclaimed)

**Exercise 5.1: Page Space Calculation**

**🔬 Try It:**
```sql
-- Calculate free space in a page
-- Note: page_header() returns columns named 'lower', 'upper', 'special' (not pd_*)
SELECT 
    upper - lower AS free_space,
    special - lower AS potential_free,
    round((upper - lower)::NUMERIC / 8192 * 100, 2) AS free_pct
FROM page_header(get_raw_page('page_demo', 0));
```

## Part 3: Tuple Structure

Each tuple (row) has a header followed by data:

```
┌─────────────────────────────────────────────────────────────┐
│                  Tuple Header (23 bytes minimum)            │
├─────────────────────────────────────────────────────────────┤
│ t_xmin (4) | t_xmax (4) | t_cid (4) | t_ctid (6) |          │
│ t_infomask2 (2) | t_infomask (2) | t_hoff (1)               │
├─────────────────────────────────────────────────────────────┤
│                     Null Bitmap (optional)                  │
├─────────────────────────────────────────────────────────────┤
│                     User Data                               │
└─────────────────────────────────────────────────────────────┘
```

### Tuple Header Fields

| Field | Description |
|-------|-------------|
| t_xmin | Transaction ID that inserted this tuple |
| t_xmax | Transaction ID that deleted/updated this tuple (0 if alive) |
| t_cid | Command ID within transaction |
| t_ctid | Current tuple ID (block, offset) for this tuple version |
| t_infomask | Various flag bits |
| t_infomask2 | More flags + number of attributes |
| t_hoff | Offset to user data |

### Viewing Tuple Details

**🔬 Try It:**
```sql
-- Detailed tuple information
SELECT 
    lp AS line_pointer,
    t_xmin,
    t_xmax,
    t_ctid,
    t_infomask::BIT(16) AS infomask_bits,
    t_data
FROM heap_page_items(get_raw_page('page_demo', 0))
LIMIT 5;
```

### Understanding t_ctid

t_ctid identifies the "current" version of a tuple:

**🔬 Try It:**
```sql
-- View tuple chain for an updated row
BEGIN;
UPDATE page_demo SET value = 999 WHERE id = 1;

-- The old version points to the new version
SELECT t_ctid, t_xmin, t_xmax 
FROM heap_page_items(get_raw_page('page_demo', 0))
WHERE lp = 1;

COMMIT;
```

**Exercise 5.2: Track Tuple Versions**

**🔬 Try It:**
```sql
-- Create tracking table
CREATE TABLE tuple_tracker (id INTEGER PRIMARY KEY, val TEXT);
INSERT INTO tuple_tracker VALUES (1, 'original');

-- Check system columns
SELECT ctid, xmin, xmax, id, val FROM tuple_tracker;

-- Update and observe
UPDATE tuple_tracker SET val = 'updated' WHERE id = 1;

SELECT ctid, xmin, xmax, id, val FROM tuple_tracker;

-- What's in the page?
SELECT lp, t_xmin, t_xmax, t_ctid, lp_flags
FROM heap_page_items(get_raw_page('tuple_tracker', 0));
```

## Part 4: MVCC in Storage

PostgreSQL implements Multi-Version Concurrency Control at the storage level:

### How MVCC Works

1. **INSERT**: Creates tuple with t_xmin = current XID, t_xmax = 0
2. **UPDATE**: Sets old tuple's t_xmax, creates new tuple with new t_xmin
3. **DELETE**: Sets t_xmax on existing tuple (doesn't remove it)

**🔬 Try It:**
```sql
-- Demonstrate MVCC
BEGIN;
SELECT txid_current();  -- Note your transaction ID

INSERT INTO tuple_tracker VALUES (2, 'test');

SELECT ctid, xmin, xmax, * FROM tuple_tracker WHERE id = 2;
-- xmin = your transaction ID, xmax = 0

COMMIT;

-- Now update
BEGIN;

UPDATE tuple_tracker SET val = 'modified' WHERE id = 2;
-- Old tuple: xmax = current XID
-- New tuple: xmin = current XID
COMMIT;
```

### Visibility Rules

A tuple is visible to a transaction if:
- t_xmin is committed AND before snapshot
- t_xmax is 0 OR not committed OR after snapshot

Transaction status lookup uses pg_xact (formerly pg_clog).  Files in $PGDATA/pg_xact/

## Part 5: HOT Updates (Heap-Only Tuples)

**HOT (Heap-Only Tuples)** is an optimization that avoids creating new index entries when a row is updated. This is a big performance win because index updates are expensive.

### The Problem HOT Solves

Without HOT, every UPDATE creates:
1. A new tuple version in the heap (table)
2. New entries in **every index** pointing to the new tuple

For a table with 5 indexes, updating one non-indexed column would create 5 new index entries—even though the indexed values didn't change!

### How HOT Works

When PostgreSQL can use a HOT update:
1. The new tuple is created on the **same page** as the old one
2. No index entries are updated
3. A **HOT chain** links old tuple → new tuple within the page

**Requirements for HOT:**
1. Updated columns are **not indexed** (indexed columns must stay the same)
2. New tuple **fits on the same page** (enough free space)

### HOT Chain Explained

A HOT chain is a linked list of tuple versions within a single page. Indexes still point to the original line pointer, which redirects to the current version.

The key idea: **the index points to a stable location**, and PostgreSQL uses in-page redirection to find the newest version. This is why HOT updates avoid index writes when only non-indexed columns change.

**Before any updates:**
```
Index → Line Pointer 1 → Tuple v1 (id=1, data='original')
```

**After HOT update:**
```
Index → Line Pointer 1 (REDIRECT to LP 2)
                ↓
        Line Pointer 2 → Tuple v2 (id=1, data='updated')
        
        (Old Tuple v1 is marked dead, waiting for VACUUM)
```

**After another HOT update:**
```
Index → Line Pointer 1 (REDIRECT to LP 3)
                ↓
        Line Pointer 3 → Tuple v3 (id=1, data='newest')
        
        (Tuples v1 and v2 are dead)
```

The index **never changes**—it always points to Line Pointer 1. PostgreSQL follows the chain to find the current version.

### Demonstrating HOT Updates

**🔬 Try It:**
```sql
-- Create table with index
CREATE TABLE hot_demo (
    id SERIAL PRIMARY KEY,
    indexed_col INTEGER,
    non_indexed_col TEXT
);
CREATE INDEX idx_hot_indexed ON hot_demo(indexed_col);

INSERT INTO hot_demo (indexed_col, non_indexed_col) VALUES (100, 'original');

-- Update non-indexed column (should be HOT)
UPDATE hot_demo SET non_indexed_col = 'updated' WHERE id = 1;

-- Check if it was a HOT update
SELECT n_tup_hot_upd FROM pg_stat_user_tables WHERE relname = 'hot_demo';
-- n_tup_hot_upd should be 1

-- Look at the page - see the HOT chain via t_ctid
SELECT lp, lp_flags, t_ctid 
FROM heap_page_items(get_raw_page('hot_demo', 0));

-- After VACUUM or page pruning, lp=1 can become a REDIRECT:
VACUUM hot_demo;

SELECT lp, lp_flags, t_ctid 
FROM heap_page_items(get_raw_page('hot_demo', 0));
```

How to interpret this output:
- **`t_ctid`** forms the HOT chain: the old tuple points to the location of the newer tuple.
- **Right after an update**, the old line pointer can still be `NORMAL` while `t_ctid` points forward.
- After **VACUUM / pruning**, PostgreSQL may convert the old line pointer to a **REDIRECT** (`lp_flags=2`) so future index lookups jump directly to the newest tuple.

### When HOT Fails

HOT update **cannot** be used when:

**🔬 Try It:**
```sql
-- Updating an INDEXED column - NOT HOT (index must be updated)
UPDATE hot_demo SET indexed_col = 200 WHERE id = 1;

-- Page is too full - NOT HOT (no room for new tuple on same page)
-- (This happens naturally as pages fill up)
```

### Why HOT Chains Matter

| Benefit | Impact |
|---------|--------|
| **Fewer index writes** | Dramatically faster updates for wide tables with many indexes |
| **Less WAL** | Smaller write-ahead log = faster replication |
| **Less bloat** | Index doesn't grow with each update |
| **Better cache efficiency** | Related tuple versions on same page |

**Tip:** Design tables with HOT in mind—put frequently updated columns in non-indexed fields when possible.

## Part 6: TOAST (The Oversized-Attribute Storage Technique)

PostgreSQL can't store values larger than ~2KB directly in a page. TOAST handles large values:

### TOAST Strategies

| Strategy | Description |
|----------|-------------|
| PLAIN | No TOAST (small fixed-length types) |
| EXTENDED | Compress then store out-of-line if needed |
| EXTERNAL | Store out-of-line without compression |
| MAIN | Compress but avoid out-of-line if possible |

**🔬 Try It:**
```sql
-- Check TOAST strategy for columns
SELECT 
    attname,
    atttypid::regtype,
    CASE attstorage
        WHEN 'p' THEN 'PLAIN'
        WHEN 'e' THEN 'EXTERNAL'
        WHEN 'm' THEN 'MAIN'
        WHEN 'x' THEN 'EXTENDED'
    END AS storage
FROM pg_attribute
WHERE attrelid = 'employees'::regclass AND attnum > 0;

-- Change storage strategy
ALTER TABLE employees ALTER COLUMN email SET STORAGE EXTERNAL;
```

### TOAST Tables

**🔬 Try It:**
```sql
-- Create table with large values
CREATE TABLE toast_demo (id SERIAL, big_data TEXT);

-- Insert large value
INSERT INTO toast_demo (big_data) 
VALUES (repeat('x', 200000));

-- Find the TOAST table
SELECT reltoastrelid::regclass FROM pg_class WHERE relname = 'toast_demo';

-- Examine TOAST table (change oid with value from previous query)
SELECT chunk_id, chunk_seq, length(chunk_data) 
FROM pg_toast.pg_toast_<oid>
LIMIT 5;

-- Compare sizes (change oid with value from previous query)
SELECT 
    pg_relation_size('toast_demo') AS main_size,
    pg_relation_size('pg_toast.pg_toast_<oid>') AS toast_size;
```

### TOAST Compression

**🔬 Try It:**
```sql
-- Check compression
SELECT 
    id,
    pg_column_size(big_data) AS in_row_size,
    length(big_data) AS actual_length
FROM toast_demo;

-- The in-row size is much smaller due to compression!
```

## Part 7: Free Space Map (FSM)

The FSM tracks free space in each page:

**🔬 Try It:**
```sql
-- Install pg_freespacemap
CREATE EXTENSION pg_freespacemap;

-- View free space per page
SELECT blkno, avail 
FROM pg_freespace('page_demo') 
LIMIT 20;

-- Total free space in a table
SELECT 
    sum(avail) AS total_free,
    count(*) AS num_pages,
    avg(avail)::INTEGER AS avg_free_per_page
FROM pg_freespace('page_demo');
```

### FSM Structure

The FSM is a tree structure:

```
Level 2: [max of children]
         /              \
Level 1: [max] ... ... [max]
         /  \           /  \
Level 0: [page free] [page free] ...
```

**Exercise 5.3: Observe FSM Changes**

**🔬 Try It:**
```sql
-- Insert data
INSERT INTO page_demo (name, value) 
SELECT 'bulk_' || i, i 
FROM generate_series(1, 1000) AS i;

-- Force writes to file
CHECKPOINT;

-- Check FSM
SELECT blkno, avail FROM pg_freespace('page_demo') WHERE avail > 0;

-- Delete some data
DELETE FROM page_demo WHERE id % 3 = 0;

-- FSM not immediately updated
SELECT blkno, avail FROM pg_freespace('page_demo') WHERE avail > 0;

-- After VACUUM, FSM is updated
VACUUM page_demo;
SELECT blkno, avail FROM pg_freespace('page_demo') WHERE avail > 0;
```

## Part 8: Visibility Map (VM)

The VM tracks which pages contain only visible tuples:

**Uses:**
1. **Index-only scans**: Skip heap fetch for visible pages
2. **VACUUM**: Skip pages with no dead tuples

**🔬 Try It:**
```sql
-- Create Extensions
CREATE EXTENSION pg_visibility;

-- View visibility map
SELECT * FROM pg_visibility('page_demo') LIMIT 20;

-- Summary
SELECT 
    count(*) AS total_pages,
    count(*) FILTER (WHERE all_visible) AS visible_pages,
    count(*) FILTER (WHERE all_frozen) AS frozen_pages
FROM pg_visibility('page_demo');
```

What the `pg_visibility()` output means:
- **`blkno`**: page number within the relation
- **`all_visible`**: true when all tuples on that page are visible to all transactions (enables index-only scans to skip heap fetches)
- **`all_frozen`**: true when all tuples are frozen (helps prevent transaction ID wraparound)

### Making Pages All-Visible

**🔬 Try It:**
```sql
-- After bulk insert, pages might not be all-visible
INSERT INTO page_demo (name, value) 
SELECT 'new_' || i, i FROM generate_series(1, 100) AS i;

SELECT count(*) FILTER (WHERE all_visible) FROM pg_visibility('page_demo');

-- VACUUM sets the visibility map bits
VACUUM page_demo;

SELECT count(*) FILTER (WHERE all_visible) FROM pg_visibility('page_demo');
```

## Part 9: Tablespaces

Tablespaces allow storing data in different filesystem locations.  Here is an **example**:

> ```sql
> -- Create a tablespace (requires filesystem directory)
> -- First create directory: mkdir /data/fast_ssd
> CREATE TABLESPACE fast_storage LOCATION '/data/fast_ssd';
> 
> -- Create table in specific tablespace
> CREATE TABLE hot_data (id SERIAL, data TEXT) TABLESPACE fast_storage;
> 
> -- Move table to different tablespace
> ALTER TABLE cold_data SET TABLESPACE pg_default;
> 
> -- View tablespace usage
> SELECT 
>     spcname,
>     pg_tablespace_location(oid) AS location,
>     pg_size_pretty(pg_tablespace_size(oid)) AS size
> FROM pg_tablespace;
> ```

### Default Tablespaces

**Example:**

> ```sql
> -- Set default tablespace for database
> ALTER DATABASE learning SET TABLESPACE fast_storage;
> 
> -- Set default for session
> SET default_tablespace = 'fast_storage';
> ```

## Part 10: Examining Storage Internals

### Raw Page Analysis

**🔬 Try It:**
```sql
-- Full page dump (hex)
SELECT encode(get_raw_page('page_demo', 0), 'hex');

-- Parse specific tuple data
SELECT t_data FROM heap_page_items(get_raw_page('page_demo', 0)) WHERE lp = 2;

-- Decode tuple data (varies by table structure)
SELECT 
    lp,
    t_data,
    substring(t_data from 1 for 4) AS first_4_bytes
FROM heap_page_items(get_raw_page('page_demo', 0))
LIMIT 5;
```

### Index Page Structure

B-tree indexes have their own page structure. The `pageinspect` extension provides functions to examine them.

**🔬 Try It:**
```sql
-- Create an index to examine
CREATE INDEX idx_page_demo ON page_demo(id);

-- First, let's understand B-tree structure:
-- Page 0 = metapage (contains tree metadata)
-- Page 1+ = root/internal/leaf pages

-- View the metapage
SELECT * FROM bt_metap('idx_page_demo');
```

**Metapage fields explained:**

| Field | Meaning | What to Look For |
|-------|---------|------------------|
| `magic` | B-tree identifier | Should be 340322 (validates it's a B-tree) |
| `version` | B-tree version | 4 = current PostgreSQL version |
| `root` | Root page number | Higher numbers after many splits = deeper tree |
| `level` | Tree depth | 0 = single page, 1-2 = normal, 3+ = large index |
| `fastlevel` | Fast-path root | Used for optimization |

**🔬 Try It:**
```sql
-- View B-tree page statistics (use page 1 for the root in small indexes)
SELECT * FROM bt_page_stats('idx_page_demo', 1);
```

**bt_page_stats fields explained:**

| Field | Meaning | Warning Signs |
|-------|---------|---------------|
| `blkno` | Page number | — |
| `type` | Page type: 'l'=leaf, 'r'=root, 'i'=internal | — |
| `live_items` | Active index entries | — |
| `dead_items` | Deleted but not vacuumed | **High count = needs VACUUM** |
| `avg_item_size` | Average entry size in bytes | — |
| `page_size` | Always 8192 | — |
| `free_size` | Available space | **Low = page nearly full, may split** |
| `btpo_prev/next` | Sibling page links | — |
| `btpo_level` | Level in tree (0=leaf) | — |
| `btpo_flags` | Page flags | — |

**🔬 Try It:**
```sql
-- View individual index entries
SELECT itemoffset, ctid, itemlen, data 
FROM bt_page_items('idx_page_demo', 1) 
LIMIT 10;
```

**bt_page_items fields explained:**

| Field | Meaning | What It Tells You |
|-------|---------|-------------------|
| `itemoffset` | Position in page | Entry number (1-based) |
| `ctid` | Points to heap tuple | (page, item) of the actual row |
| `itemlen` | Entry size in bytes | Larger = more index bloat per entry |
| `data` | Indexed value (hex) | The actual key value stored |

### Diagnosing Index Problems

**Problem: Index bloat (too many dead entries)**

**🔬 Try It:**
```sql
-- Check dead item ratio across all pages
SELECT 
    s.blkno,
    s.live_items,
    s.dead_items,
    CASE WHEN s.live_items > 0 
         THEN round(s.dead_items::numeric / s.live_items * 100, 1)
         ELSE 0 
    END AS dead_pct,
    s.free_size
FROM generate_series(1, pg_relation_size('idx_page_demo')::bigint / 8192 - 1) AS page_num,
     LATERAL bt_page_stats('idx_page_demo', page_num::int) AS s
ORDER BY s.dead_items DESC
LIMIT 10;

-- If dead_pct is high (>20%), run:
-- REINDEX INDEX idx_page_demo;
```

**Problem: Tree too deep (slow lookups)**

**🔬 Try It:**
```sql
-- Check tree depth
SELECT level AS tree_depth FROM bt_metap('idx_page_demo');
-- 0-2 = normal, 3+ = consider if index is necessary or data type is optimal
```

**Problem: Low fill factor (wasted space)**

**🔬 Try It:**
```sql
-- Check average page utilization (dynamically determines page count)
SELECT 
    count(*) AS pages_checked,
    round(avg(s.free_size)) AS avg_free_bytes,
    round(avg(s.free_size)::numeric / 8192 * 100, 1) AS avg_free_pct
FROM generate_series(
    1, 
    GREATEST(1, (pg_relation_size('idx_page_demo') / 8192)::int - 1)
) AS page_num,
LATERAL bt_page_stats('idx_page_demo', page_num::int) AS s;
-- High free_pct (>50%) after VACUUM might indicate fillfactor issue
```

## Source Code References

| Component | Source File |
|-----------|-------------|
| Page layout | `src/include/storage/bufpage.h` |
| Tuple header | `src/include/access/htup_details.h` |
| Heap access | `src/backend/access/heap/heapam.c` |
| TOAST | `src/backend/access/heap/tuptoaster.c` |
| FSM | `src/backend/storage/freespace/` |
| Visibility map | `src/backend/access/heap/visibilitymap.c` |

## Summary

In this chapter, you learned:

1. **Storage Hierarchy**: Clusters → Databases → Tables → Pages → Tuples
2. **Page Structure**: 8KB pages with header, item pointers, tuples
3. **Tuple Layout**: Header with MVCC info + null bitmap + data
4. **MVCC Storage**: xmin, xmax, ctid for version tracking
5. **HOT Updates**: Optimization to avoid index updates
6. **TOAST**: Large value storage and compression
7. **FSM**: Tracking free space for efficient inserts
8. **VM**: Tracking visibility for index-only scans
9. **Tablespaces**: Storing data on different devices

## Storage Quick Reference

**🔬 Try It:**
```sql
-- Table size information
SELECT 
    pg_size_pretty(pg_relation_size('tablename')) AS heap_size,
    pg_size_pretty(pg_table_size('tablename')) AS total_size,
    pg_size_pretty(pg_indexes_size('tablename')) AS index_size,
    pg_size_pretty(pg_total_relation_size('tablename')) AS everything;

-- Page inspection
SELECT * FROM page_header(get_raw_page('tablename', 0));
SELECT * FROM heap_page_items(get_raw_page('tablename', 0));

-- Free space
SELECT * FROM pg_freespace('tablename');

-- Visibility
SELECT * FROM pg_visibility('tablename');
```

---

## Chapter Cleanup

The following tables were created for demonstration purposes in this chapter:

**🔬 Try It:** Clean up demonstration objects

```sql
-- Drop demonstration tables (indexes are dropped automatically with tables)
DROP TABLE IF EXISTS page_demo CASCADE;
DROP TABLE IF EXISTS tuple_tracker CASCADE;
DROP TABLE IF EXISTS hot_demo CASCADE;
DROP TABLE IF EXISTS toast_demo CASCADE;
```

> **Note:** The extensions (`pageinspect`, `pg_freespacemap`, `pg_visibility`) are kept as they're useful for ongoing database analysis and are used in later chapters.

---

**Previous Chapter:** [← Database Architecture](04-database-architecture.md)

**Next Chapter:** [Query Processing →](06-query-processing.md)

