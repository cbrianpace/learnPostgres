# Chapter 7: Indexing Deep Dive

## The Story So Far

Your company has been growing rapidly. What started as a small startup with a handful of employees has expanded to hundreds of team members across multiple departments. The HR system that was handling a few dozen time entries per week now has **years of historical timecard data**.

The `timecards` table that was manageable last year now contains **500,000 entries** and is growing daily. Payroll reports that used to run in milliseconds now take 10+ seconds. The finance team's monthly labor cost report is timing out.

The performance problems have become impossible to ignore. Let's fix them with indexes.

## Learning Objectives

By the end of this chapter, you will be able to:
- Understand how different index types work internally
- Choose the right index type for each use case
- Create efficient indexes for common query patterns
- Monitor and maintain indexes
- Avoid common indexing mistakes

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

This chapter builds on the tables we've created so far: `employees`, `departments`, `projects`, and `project_assignments`.

---

## Part 1: Setting Up the Timecards Table

First, let's create the timecards table with realistic data volume. In a real HR system, employees log their hours daily, creating a high-volume table perfect for demonstrating indexing strategies.

**🔬 Try It:** Create the Timecards Table

```sql
-- Timecards table (this will be large!)
CREATE TABLE timecards (
    timecard_id SERIAL PRIMARY KEY,
    emp_id INTEGER REFERENCES employees(emp_id),
    work_date DATE NOT NULL,
    hours_worked NUMERIC(4,2) NOT NULL CHECK (hours_worked >= 0 AND hours_worked <= 24),
    project_id INTEGER REFERENCES projects(project_id),
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- 'draft', 'submitted', 'approved', 'rejected'
    submitted_at TIMESTAMPTZ,
    approved_by INTEGER REFERENCES employees(emp_id),
    approved_at TIMESTAMPTZ
);

-- Populate timecards with 500,000 historical entries
-- This simulates a company with many employees over several years
-- We use generate_series for predictable row count

INSERT INTO timecards (emp_id, work_date, hours_worked, project_id, description, status, submitted_at, approved_at)
SELECT 
    -- Distribute across employees (cycle through existing emp_ids)
    (SELECT emp_id FROM employees ORDER BY emp_id OFFSET (n % (SELECT COUNT(*) FROM employees)) LIMIT 1),
    -- Spread dates across ~5 years of history
    (CURRENT_DATE - ((n / 50) || ' days')::INTERVAL)::DATE,
    -- Regular 8-hour days with some variation (6-10 hours)
    (6 + (random() * 4))::NUMERIC(4,2),
    -- Assign to random project (some entries null for non-project work)
    CASE WHEN random() > 0.3 
        THEN (SELECT project_id FROM projects ORDER BY random() LIMIT 1) 
        ELSE NULL 
    END,
    -- Work description
    (ARRAY['Development', 'Meetings', 'Code review', 'Documentation', 'Testing', 'Planning', 'Training', 'Support'])[1 + (random() * 7)::INT],
    -- Status distribution: most approved, some recent ones in other states
    CASE 
        WHEN n > 450000 THEN 
            (ARRAY['draft', 'submitted', 'approved'])[1 + (random() * 2)::INT]
        WHEN n > 400000 THEN 
            (ARRAY['approved', 'approved', 'approved', 'rejected'])[1 + (random() * 3)::INT]
        ELSE 'approved'
    END,
    -- Submitted timestamp (for non-draft entries)
    CASE WHEN random() > 0.1 
        THEN (CURRENT_DATE - ((n / 50) || ' days')::INTERVAL + INTERVAL '1 day')::TIMESTAMPTZ
        ELSE NULL 
    END,
    -- Approved timestamp (for older approved entries)
    CASE WHEN n < 400000 AND random() > 0.2 
        THEN (CURRENT_DATE - ((n / 50) || ' days')::INTERVAL + INTERVAL '3 days')::TIMESTAMPTZ
        ELSE NULL 
    END
FROM generate_series(1, 500000) AS n;

-- Gather statistics
ANALYZE timecards;

-- Verify the data
SELECT COUNT(*) AS total_timecards FROM timecards;

SELECT status, COUNT(*) FROM timecards GROUP BY status ORDER BY COUNT(*) DESC;

SELECT 
    MIN(work_date) AS earliest_date,
    MAX(work_date) AS latest_date,
    COUNT(DISTINCT emp_id) AS employees_with_timecards
FROM timecards;
```

---

## Part 2: Understanding the Problem

### Why Queries Slow Down

Without indexes, PostgreSQL must scan every row in a table to find matches:

```
Without Index (Sequential Scan):     With Index:
┌──────────────────────────┐        ┌──────────┐
│  Read EVERY page         │        │  B-tree  │ → Find location
│  Check EVERY row         │        │  Index   │   in O(log n)
│  Time: O(n)              │        └────┬─────┘
└──────────────────────────┘             │
                                         ▼
                                    ┌──────────┐
                                    │ Direct   │ → Read only
                                    │ Access   │   matching rows
                                    └──────────┘
```

### The Cost of Indexes

| Benefit | Cost |
|---------|------|
| Faster reads | Slower writes (index must be updated) |
| Query optimization | Storage space |
| Uniqueness enforcement | Maintenance overhead (bloat, reindex) |

**Rule of Thumb**: Don't add indexes speculatively. Wait until you have a performance problem, then add the right index for that specific query pattern.

---

## Part 3: Timecard Lookups Are Slow

Payroll reports that looking up timecards by employee or status takes forever. They need to run payroll reports quickly! Let's investigate.

**🔬 Try It:** Diagnose the Problem

```sql
-- Payroll query: find all timecards for an employee
EXPLAIN ANALYZE
SELECT * FROM timecards WHERE emp_id = 1;
```

You'll see `Seq Scan` in the output—PostgreSQL is reading all 500,000 rows to find matches.

Check the execution time (look at "Execution Time" in output). It might be 20-100ms, but that adds up when you're processing payroll for all employees.


### B-Tree Indexes: The Workhorse

**Source**: `src/backend/access/nbtree/`

B-Tree is the default index type. It organizes data in a balanced tree structure, enabling **O(log n)** lookups.

**What does O(log n) mean?**

O(log n) describes how the number of operations grows as data size increases. With a B-tree index:

| Table Rows | Sequential Scan (O(n)) | Index Lookup (O(log n)) |
|------------|------------------------|-------------------------|
| 1,000 | ~1,000 comparisons | ~10 comparisons |
| 1,000,000 | ~1,000,000 comparisons | ~20 comparisons |
| 1,000,000,000 | ~1,000,000,000 comparisons | ~30 comparisons |

Each time you **10x the data**, a sequential scan does **10x more work**, but an index lookup only adds **~3 more comparisons** (because log₂(10) ≈ 3.3). This is why indexes are essential for large tables—the performance difference grows dramatically with scale.

```
                    [Root: M]
                   /         \
            [Internal: D,H]   [Internal: R,V]
           /     |     \       /     |     \
         [A-C] [E-G] [I-L]   [N-Q] [S-U] [W-Z]
           ↓     ↓     ↓       ↓     ↓     ↓
         heap   heap   heap   heap  heap  heap
```

**B-Tree supports these operations:**

| Operation | Supported | Example |
|-----------|-----------|---------|
| Equality | ✓ | `WHERE emp_id = 1` |
| Range | ✓ | `WHERE hours_worked > 8` |
| BETWEEN | ✓ | `WHERE work_date BETWEEN '2024-01-01' AND '2024-03-31'` |
| IN | ✓ | `WHERE status IN ('submitted', 'approved')` |
| IS NULL | ✓ | `WHERE project_id IS NULL` |
| Pattern (prefix) | ✓ | `WHERE description LIKE 'Meeting%'` |
| Pattern (suffix) | ✗ | `WHERE description LIKE '%review'` |

**🔬 Try It:** Add Index for Employee Lookups

```sql
-- Before: Check query plan (should show Seq Scan)
EXPLAIN ANALYZE
SELECT * FROM timecards WHERE emp_id = 1;
-- Note the execution time

-- Create index for emp_id lookups
CREATE INDEX idx_timecards_emp_id ON timecards(emp_id);

-- After: Check query plan again (should show Index Scan!)
EXPLAIN ANALYZE
SELECT * FROM timecards WHERE emp_id = 1;
-- Compare the execution time - should be much faster!

-- Check our new index
\d+ timecards
```

### Multi-Column Indexes: Optimizing Complex Queries

Payroll also filters by employee AND status (they only want approved timecards for payment):

**🔬 Try It:** Multi-Column Indexes

```sql
-- This query filters on two columns
SELECT timecard_id, work_date, hours_worked 
FROM timecards 
WHERE emp_id = 1 AND status = 'approved';
```

A multi-column index can help, but **column order matters**:

**🔬 Try It:**
```sql
DROP INDEX idx_timecards_emp_id;

CREATE INDEX idx_timecards_emp_status ON timecards(emp_id, status);

EXPLAIN ANALYZE 
SELECT * FROM timecards WHERE emp_id = 1 AND status = 'approved';

EXPLAIN ANALYZE
SELECT * FROM timecards WHERE emp_id = 1;

EXPLAIN ANALYZE
SELECT * FROM timecards WHERE status = 'submitted';
```

What to look for:
- For `emp_id = 1 AND status = 'approved'`, you should see **`Index Scan using idx_timecards_emp_status`** and an `Index Cond` that includes **both columns**.
- For `emp_id = 1`, you should still see the same index used (because `emp_id` is the leading column).
- For `status = 'submitted'`, PostgreSQL usually can’t use this index efficiently, since the leading column (`emp_id`) is missing.

**Key Insight**: A multi-column index on `(emp_id, status)` can replace a single-column index on `emp_id`. It handles both:
- Queries filtering on `emp_id` only
- Queries filtering on `emp_id AND status`

But it **cannot** efficiently handle queries filtering on `status` alone.

**Rule**: Put equality columns before range columns. Put most selective columns first.

---

## Part 4: Case-Insensitive Search and Date Queries

HR reports that searching for timecards by description is case-sensitive, and payroll needs to query by work week efficiently.

### The Problem with Functions on Indexed Columns

**🔬 Try It:**
```sql
-- This search won't use a simple index on description!
EXPLAIN SELECT * FROM timecards WHERE LOWER(description) = 'meetings';
-- Shows Seq Scan because LOWER(description) doesn't match the indexed column
```

The index stores `description` values, but the query asks for `LOWER(description)` values—they don't match.

### Expression Indexes: The Solution

**🔬 Try It:**
```sql
-- Index the expression we're searching on
CREATE INDEX idx_timecards_description_lower ON timecards(LOWER(description));

-- Now this query uses the index
EXPLAIN SELECT * FROM timecards WHERE LOWER(description) = 'meetings';
-- Shows "Index Scan using idx_timecards_description_lower"
```

### Other Useful Expression Indexes

**🔬 Try It:**
```sql
-- Index for year-based queries on work_date
CREATE INDEX idx_timecards_work_year ON timecards(EXTRACT(YEAR FROM work_date));

-- Now this is fast - get all timecards from 2024
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(hours_worked) FROM timecards WHERE EXTRACT(YEAR FROM work_date) = 2024;

-- Index for week-based queries (useful for weekly payroll)
CREATE INDEX idx_timecards_work_week ON timecards(EXTRACT(WEEK FROM work_date), EXTRACT(YEAR FROM work_date));

-- Fast weekly summary
EXPLAIN ANALYZE
SELECT SUM(hours_worked) 
FROM timecards 
WHERE EXTRACT(WEEK FROM work_date) = 1 AND EXTRACT(YEAR FROM work_date) = 2024;
```

---

## Part 5: Event Analytics Are Crawling

The analytics team needs to query the `events` table by event type and user metadata. With thousands of events, queries are slow.

### GIN Indexes: For Multi-Valued Data

**Source**: `src/backend/access/gin/`

GIN (Generalized Inverted Index) is designed for values containing multiple elements—JSONB, arrays, full-text search vectors.

**Think of GIN like a book's index**: Instead of scanning every page for "PostgreSQL", you look in the index, find "PostgreSQL → pages 5, 23, 47", and jump directly there.

| Use Case | Data Type | Operators |
|----------|-----------|-----------|
| JSONB queries | `JSONB` | `@>`, `?`, `?&`, `?\|` |
| Array containment | `TEXT[]`, etc. | `@>`, `&&`, `<@` |
| Full-text search | `tsvector` | `@@` |

### Indexing the Events Table

**🔬 Try It:**
```sql
-- Create the events table if it doesn't exist (may exist from Chapter 3)
CREATE TABLE IF NOT EXISTS events (
    id SERIAL PRIMARY KEY,
    event_time TIMESTAMPTZ DEFAULT NOW(),
    data JSONB
);

-- Populate with realistic event data for GIN index demonstration
INSERT INTO events (data)
SELECT jsonb_build_object(
    'type', (ARRAY['click', 'view', 'purchase', 'login', 'logout'])[1 + (random() * 4)::int],
    'page', '/page/' || (random() * 100)::int,
    'user_id', (random() * 1000)::int,
    'timestamp', NOW() - (random() * 30 || ' days')::interval
)
FROM generate_series(1, 50000);

-- Without an index, containment queries are slow
EXPLAIN ANALYZE SELECT * FROM events WHERE data @> '{"type": "click"}';

-- Add a GIN index
CREATE INDEX idx_events_data ON events USING gin(data);

-- Now containment queries are fast
EXPLAIN ANALYZE SELECT * FROM events WHERE data @> '{"type": "click"}';
-- Shows "Bitmap Index Scan on idx_events_data"
```

### Key Existence Queries

**🔬 Try It:**
```sql
-- Find events that have a 'purchase_amount' field
EXPLAIN SELECT * FROM events WHERE data ? 'purchase_amount';
```

### Indexing Specific JSONB Paths

If you only query specific paths, a targeted index is smaller:

**🔬 Try It:**
```sql
-- Index just the 'type' field
CREATE INDEX idx_events_type ON events((data->>'type'));

-- This uses the expression index
EXPLAIN SELECT * FROM events WHERE data->>'type' = 'click';
```

---

## Part 6: Meeting Room Double-Bookings

Facilities reports that the meeting room booking system allows double-bookings. We need to prevent overlapping reservations.

### GiST Indexes: For Complex Comparisons

**Source**: `src/backend/access/gist/`

GiST (Generalized Search Tree) handles operations B-tree can't: "contains", "overlaps", "nearest neighbor".

| Data Type | Operations | Example |
|-----------|------------|---------|
| Range types | Contains, overlaps | Booking conflicts |
| Geometric | Distance, containment | Location queries |
| Full-text | Matches | Document search |

### Creating a Booking System with Overlap Prevention

**🔬 Try It:**
```sql
-- We need btree_gist for scalar types in exclusion constraints
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Create the bookings table
CREATE TABLE room_bookings (
    id SERIAL PRIMARY KEY,
    room_name TEXT NOT NULL,
    reserved_by INTEGER REFERENCES employees(emp_id),
    during TSTZRANGE NOT NULL
);

-- GiST index for range queries
CREATE INDEX idx_bookings_during ON room_bookings USING gist(during);

-- Exclusion constraint prevents overlapping bookings for same room
ALTER TABLE room_bookings
ADD CONSTRAINT no_double_booking
EXCLUDE USING gist (room_name WITH =, during WITH &&);

-- This booking succeeds
INSERT INTO room_bookings (room_name, reserved_by, during) 
VALUES ('Conference A', 1, '[2024-01-15 09:00, 2024-01-15 10:00)');

-- This booking for a DIFFERENT room succeeds
INSERT INTO room_bookings (room_name, reserved_by, during) 
VALUES ('Conference B', 2, '[2024-01-15 09:00, 2024-01-15 10:00)');
```

**🔬 Try It:** Overlapping booking fails
```sql
-- This booking FAILS - overlaps with first booking!
INSERT INTO room_bookings (room_name, reserved_by, during) 
VALUES ('Conference A', 3, '[2024-01-15 09:30, 2024-01-15 10:30)');
-- ERROR: conflicting key value violates exclusion constraint "no_double_booking"
```

### Finding Available Rooms

**🔬 Try It:**
```sql
-- Find rooms available during a time slot
SELECT DISTINCT room_name FROM room_bookings
WHERE NOT during && '[2024-01-15 14:00, 2024-01-15 15:00)'::TSTZRANGE;
```

---

## Part 7: Badge Access Log Queries Timing Out

Security needs to query badge swipe history for compliance audits. The `badge_access` table has millions of entries—every time an employee badges into a building, door, or secure area, it's logged. Date range queries for audits are extremely slow.

### BRIN Indexes: For Naturally Sorted Data

**Source**: `src/backend/access/brin/`

BRIN (Block Range INdex) is incredibly compact—often 1000x smaller than B-tree. Instead of indexing every row, it stores min/max values for ranges of physical blocks.

**BRIN works because**: Access log data is inserted in chronological order. Rows with similar timestamps are physically close together on disk. BRIN can skip entire block ranges that can't possibly contain matches.

| Index Type | Size (1M rows) | Best Use Case |
|------------|----------------|---------------|
| B-tree | ~20 MB | Random access patterns |
| BRIN | ~20 KB | Naturally sorted data |

### Creating Badge Access Log Infrastructure

**🔬 Try It:** Create badge access table

```sql
-- Badge access log table - tracks all building/door entry events
CREATE TABLE badge_access (
    id BIGSERIAL PRIMARY KEY,
    emp_id INTEGER REFERENCES employees(emp_id),
    access_point VARCHAR(50) NOT NULL,  -- 'Main Entrance', 'Server Room', 'Parking Garage', etc.
    access_time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    access_type VARCHAR(10) NOT NULL,   -- 'entry' or 'exit'
    granted BOOLEAN NOT NULL DEFAULT true
);

-- Insert 1 million time-ordered access records (simulating ~2 years of data)
INSERT INTO badge_access (emp_id, access_point, access_time, access_type, granted)
SELECT 
    (SELECT emp_id FROM employees ORDER BY random() LIMIT 1),
    (ARRAY['Main Entrance', 'Side Door', 'Parking Garage', 'Server Room', 'Executive Floor', 'Cafeteria'])[1 + (random() * 5)::int],
    '2023-01-01'::timestamptz + (i || ' seconds')::interval,
    (ARRAY['entry', 'exit'])[1 + (random() * 1)::int],
    random() > 0.02  -- 2% denied access attempts
FROM generate_series(1, 1000000) AS i;

ANALYZE badge_access;

-- Verify the data
SELECT COUNT(*) FROM badge_access;
SELECT MIN(access_time), MAX(access_time) FROM badge_access;
```

**🔬 Try It:** Compare B-tree vs BRIN

```sql
-- Create BRIN index (with default 128 pages per range)
CREATE INDEX idx_badge_access_brin ON badge_access USING brin(access_time);

-- Compare with B-tree for reference
CREATE INDEX idx_badge_access_btree ON badge_access(access_time);

-- Check the dramatic size difference!
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(quote_ident(indexname)::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'badge_access';
-- BRIN will be dramatically smaller (often 100x-1000x)

-- Both indexes work for this audit query, but BRIN is much smaller
EXPLAIN ANALYZE
SELECT * FROM badge_access
WHERE access_time BETWEEN '2024-01-15' AND '2024-01-16';

-- Drop the B-tree to save space - BRIN is sufficient for time-range queries
DROP INDEX idx_badge_access_btree;
```

### Tuning BRIN: Pages Per Range

**🔬 Try It:**
```sql
-- Smaller ranges = more precision but larger index
CREATE INDEX idx_badge_brin_precise 
ON badge_access USING brin(access_time) 
WITH (pages_per_range = 32);

-- Larger ranges = smaller index but less precision
CREATE INDEX idx_badge_brin_compact 
ON badge_access USING brin(access_time) 
WITH (pages_per_range = 256);
```

`pages_per_range` is the main BRIN trade-off knob:
- Smaller ranges (e.g., 32) = more precise skipping, larger BRIN index
- Larger ranges (e.g., 256) = smaller BRIN index, less precise skipping

**When BRIN fails**: If data isn't naturally sorted (e.g., random updates scatter timestamps across blocks), BRIN becomes useless—every block range might contain any value.

---

## Part 8: Advanced Indexing Techniques

### Hash Indexes: Equality-Only

Hash indexes are smaller than B-tree for equality checks on large values:

This is a niche index type:
- Only helps for `=` comparisons (no ranges, no ordering)
- Usually not worth it unless you have a very specific workload pattern
- In many real systems, B-tree is still the default choice even for equality

**🔬 Try It:**
```sql
-- Employee badges table - maps physical badge IDs to employees
-- Includes employees, contractors, visitors, and historical badges
CREATE TABLE employee_badges (
    id SERIAL PRIMARY KEY,
    badge_serial TEXT UNIQUE NOT NULL,  -- Long alphanumeric badge ID from manufacturer
    emp_id INTEGER REFERENCES employees(emp_id),
    badge_type VARCHAR(20) NOT NULL,    -- 'employee', 'contractor', 'visitor', 'temporary'
    issued_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at DATE,
    status VARCHAR(20) DEFAULT 'active'  -- 'active', 'lost', 'revoked', 'expired'
);

-- Generate 50,000 badge records (employees, contractors, visitors over many years)
-- This volume is realistic for a mid-size company and ensures index usage
INSERT INTO employee_badges (badge_serial, emp_id, badge_type, issued_at, expires_at, status)
SELECT 
    'BADGE-' || TO_CHAR('2015-01-01'::DATE + (n || ' days')::INTERVAL, 'YYYY') || '-' || 
    UPPER(SUBSTRING(MD5(n::TEXT || RANDOM()::TEXT) FROM 1 FOR 10)),
    -- Most badges link to employees, some are visitor/contractor (NULL emp_id)
    CASE WHEN RANDOM() > 0.3 
        THEN (SELECT emp_id FROM employees ORDER BY RANDOM() LIMIT 1)
        ELSE NULL 
    END,
    (ARRAY['employee', 'employee', 'employee', 'contractor', 'visitor', 'temporary'])[1 + (RANDOM() * 5)::INT],
    '2015-01-01'::TIMESTAMPTZ + (n || ' hours')::INTERVAL,
    '2015-01-01'::DATE + (n || ' days')::INTERVAL + INTERVAL '1 year',
    CASE 
        WHEN n > 45000 THEN 'active'
        WHEN RANDOM() < 0.7 THEN 'expired'
        WHEN RANDOM() < 0.85 THEN 'lost'
        ELSE 'revoked'
    END
FROM generate_series(1, 50000) AS n;

ANALYZE employee_badges;

-- Check how many badges we have
SELECT COUNT(*) FROM employee_badges;
SELECT status, COUNT(*) FROM employee_badges GROUP BY status ORDER BY COUNT(*) DESC;

-- Hash index for fast badge lookups at door readers
CREATE INDEX idx_badge_serial_hash ON employee_badges USING hash(badge_serial);

-- Get a real badge serial to test with
SELECT badge_serial FROM employee_badges WHERE status = 'active' LIMIT 1;

-- Badge reader lookup (exact match only - hash indexes excel at this)
-- Copy a badge_serial from the query above and use it here:
EXPLAIN ANALYZE SELECT * FROM employee_badges 
WHERE badge_serial = 'BADGE-2138-ABE78C29EC';  -- Replace with actual badge serial
-- Should show: Index Scan using idx_badge_serial_hash
```

This ties into our `badge_access` table - when someone swipes their badge, the system looks up the badge serial to identify them and check if it's active.

### Partial Indexes: Index Only What You Need

Partial indexes only include rows that match a condition. They're smaller and faster when you frequently query a subset of data.

**🔬 Try It:**
```sql
-- Clean up Index
DROP INDEX idx_timecards_emp_status;

-- Payroll only cares about submitted timecards
-- Instead of indexing all 500K timecards, only index the ~10% that are submitted
CREATE INDEX idx_timecards_submitted 
ON timecards(emp_id, work_date)
WHERE status = 'submitted';

-- Check the index size vs a full index
SELECT pg_size_pretty(pg_relation_size('idx_timecards_submitted')) AS partial_size;

-- Create a full index for comparison
CREATE INDEX idx_timecards_full ON timecards(emp_id, work_date);

SELECT pg_size_pretty(pg_relation_size('idx_timecards_full')) AS full_size;

-- This query uses the partial index (matches the WHERE condition)
EXPLAIN ANALYZE 
SELECT * FROM timecards 
WHERE status = 'submitted' AND emp_id = 1 
ORDER BY work_date DESC;

-- This query CANNOT use the partial index (different status)
EXPLAIN ANALYZE
SELECT * FROM timecards 
WHERE status = 'approved' AND emp_id = 1;
-- Must use full index or sequential scan

-- Clean up the comparison index
DROP INDEX idx_timecards_full;
```

**When to use partial indexes:**
- Frequently querying a specific subset (e.g., "approved" records, "active" status)
- The subset is much smaller than the full table
- You want to save space and improve write performance

### Covering Indexes: Avoid Heap Lookups

An "index-only scan" reads data directly from the index without touching the table. This is faster because it skips the heap lookup entirely.

**🔬 Try It:**
```sql
CREATE INDEX idx_timecards_emp_id ON timecards (emp_id);

-- Traditional index on emp_id (already exists from earlier)
-- Query needs to fetch hours_worked and work_date from heap
EXPLAIN ANALYZE 
SELECT emp_id, work_date, hours_worked 
FROM timecards 
WHERE emp_id = 1;
-- Shows "Index Scan" - must visit heap for work_date and hours_worked

-- Covering index INCLUDEs the columns we SELECT
CREATE INDEX idx_timecards_emp_covering 
ON timecards(emp_id) 
INCLUDE (work_date, hours_worked);

-- Update visibility map for index-only scans
VACUUM timecards;

-- Now query can be satisfied entirely from the index
EXPLAIN ANALYZE
SELECT emp_id, work_date, hours_worked 
FROM timecards 
WHERE emp_id = 1;
-- Shows "Index Only Scan" - no heap access needed!

-- Compare the "Heap Fetches" in both plans - covering index should show 0
```

**When to use covering indexes:**
- Queries that SELECT a small, predictable set of columns
- High-frequency queries where heap access is a bottleneck
- Trade-off: larger index size for faster reads

### Concurrent Index Operations

In production, avoid locking the table during index creation:

**🔬 Try It:**
```sql
-- Create without blocking writes (takes longer)
CREATE INDEX CONCURRENTLY idx_employees_hire_date ON employees(hire_date);

-- Drop without blocking reads
DROP INDEX CONCURRENTLY idx_employees_hire_date;
```

---

## Part 9: Examining Index Internals

### B-Tree Structure

**🔬 Try It:**
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- View B-tree metadata for our covering index
SELECT * FROM bt_metap('idx_timecards_emp_covering');
```

**`bt_metap` columns explained:**

| Column | Meaning | What to Look For |
|--------|---------|------------------|
| `magic` | B-tree magic number | Should be 340322 (validates it's a B-tree) |
| `version` | B-tree version | Current version is 4 |
| `root` | Page number of root | Starting point for all searches |
| `level` | Tree height | **Key metric!** More levels = more I/O per lookup |
| `fastroot` | Optimized root page | May differ from root after deletions |
| `fastlevel` | Level of fastroot | Used for optimization |
| `last_cleanup_num_delpages` | Deleted pages from last cleanup | High = consider REINDEX |

> **Rule of thumb:** Level 2-3 is normal. Level 4+ on a reasonably-sized table may indicate bloat.

**🔬 Try It:**
```sql
-- View statistics for a specific page
SELECT * FROM bt_page_stats('idx_timecards_emp_covering', 1);
```

**`bt_page_stats` columns explained:**

| Column | Meaning | What to Look For |
|--------|---------|------------------|
| `blkno` | Block/page number | The page you're examining |
| `type` | Page type | 'l' = leaf, 'i' = internal, 'r' = root |
| `live_items` | Number of live index entries | How full is this page |
| `dead_items` | Deleted entries not yet cleaned | High = needs VACUUM |
| `avg_item_size` | Average entry size in bytes | Larger = fewer entries per page |
| `page_size` | Total page size (usually 8192) | Standard PostgreSQL page |
| `free_size` | Unused space in bytes | Low free_size = full page |
| `btpo_prev/btpo_next` | Links to sibling pages | For range scans |
| `btpo_level` | Level in tree (0 = leaf) | Leaves hold actual data pointers |

**🔬 Try It:**
```sql
-- View actual index entries on a page
SELECT itemoffset, ctid, itemlen, data 
FROM bt_page_items('idx_timecards_emp_covering', 1) 
LIMIT 10;
```

**`bt_page_items` columns explained:**

| Column | Meaning | What to Look For |
|--------|---------|------------------|
| `itemoffset` | Position within page | 1-based offset |
| `ctid` | Tuple ID (page, offset) | Points to actual table row |
| `itemlen` | Entry size in bytes | Larger keys = fewer per page |
| `nulls` | Which columns are NULL | 'f' = not null |
| `vars` | Which columns are variable-length | 't' = variable |
| `data` | Raw key data (hex) | The actual indexed values |

### Monitoring Index Usage

**🔬 Try It:**
```sql
-- Which indexes are being used?
SELECT 
    schemaname,
    relname,
    indexrelname,
    idx_scan AS times_used,
    idx_tup_read AS rows_read,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Find UNUSED indexes (candidates for removal)
SELECT 
    schemaname || '.' || indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS wasted_space
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint WHERE conindid IS NOT NULL);
```

### Detecting and Fixing Bloat

**🔬 Try It:**
```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Check an index for bloat
SELECT * FROM pgstatindex('idx_timecards_emp_covering');
-- avg_leaf_density: should be > 90% (lower = bloated)
-- leaf_fragmentation: should be low

-- Rebuild a bloated index (locks table briefly)
REINDEX INDEX idx_timecards_emp_covering;

-- Rebuild without locking (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_timecards_emp_covering;
```

---

## Part 10: Index Selection Strategy

### Decision Framework

```
1. What queries need optimization?
   └─ Identify slow queries from pg_stat_statements or logs

2. What columns are filtered/joined/sorted?
   └─ Check WHERE, JOIN ON, ORDER BY clauses

3. What operations are performed?
   ├─ Equality (=, IN)         → B-tree or Hash
   ├─ Range (<, >, BETWEEN)    → B-tree or BRIN (if sorted)
   ├─ Contains/Overlap         → GIN or GiST
   └─ Pattern (LIKE 'x%')      → B-tree with pattern_ops

4. What's the data distribution?
   ├─ High cardinality (unique-ish) → B-tree
   ├─ Naturally sorted             → BRIN
   └─ Few distinct values          → Consider partial index

5. What are the trade-offs?
   ├─ Write frequency vs read speed
   ├─ Storage space available
   └─ Maintenance overhead
```

### Quick Reference: Which Index Type?

| Query Pattern | Index Type | Example |
|---------------|------------|---------|
| `WHERE x = 5` | B-tree | Primary keys, foreign keys |
| `WHERE x > 5` | B-tree | Salary ranges |
| `WHERE x = 5 AND y > 10` | B-tree (x, y) | Multi-condition filters |
| `WHERE LOWER(x) = 'abc'` | B-tree expression | Case-insensitive search |
| `WHERE x @> '{...}'` | GIN | JSONB containment |
| `WHERE tags && ARRAY[...]` | GIN | Array overlap |
| `WHERE tsv @@ query` | GIN | Full-text search |
| `WHERE range && other` | GiST | Time range overlap |
| `WHERE point <-> center < 10` | GiST | Nearest neighbor |
| `WHERE time > '2024-01-01'` (sorted data) | BRIN | Time-series ranges |

---

## Summary: What We Built

| Problem | Solution | Index Type |
|---------|----------|------------|
| Slow timecard lookups | B-tree on emp_id | B-tree |
| Complex filter on emp+status | Multi-column index | B-tree |
| Case-insensitive description search | Expression index on LOWER() | B-tree |
| Weekly payroll queries | Expression indexes on date parts | B-tree |
| Slow event analytics | GIN on JSONB data | GIN |
| Double-booking prevention | Exclusion constraint | GiST |
| Badge access log audits | BRIN on timestamp | BRIN |

**Indexes created on timecards table:**
- `idx_timecards_description_lower` (expression index for case-insensitive search)
- `idx_timecards_work_year` (expression index for year-based queries)
- `idx_timecards_work_week` (expression index for weekly payroll)
- `idx_timecards_submitted` (partial index for submitted timecards)
- `idx_timecards_emp_covering` (covering index for index-only scans)

**Indexes created on other tables:**
- `idx_events_data` (GIN on JSONB)
- `idx_bookings_during` (GiST for range queries)

**New tables created:**
- `timecards` (500,000 rows - main HR data for indexing demonstration)
- `room_bookings` (with exclusion constraint)
- `badge_access` (1M rows with BRIN index for access log queries)

---

## Source Code References

| Index Type | Source Directory | Key File |
|------------|-----------------|----------|
| B-Tree | `src/backend/access/nbtree/` | `nbtinsert.c`, `nbtsearch.c` |
| Hash | `src/backend/access/hash/` | `hashinsert.c` |
| GiST | `src/backend/access/gist/` | `gist.c` |
| GIN | `src/backend/access/gin/` | `gininsert.c` |
| BRIN | `src/backend/access/brin/` | `brin.c` |

---

**Previous Chapter:** [← Query Processing](06-query-processing.md)

**Next Chapter:** [Transactions →](08-transactions.md)
