# Chapter 6: Query Processing and the Optimizer

## The Story So Far

Your company has been running smoothly on PostgreSQL for several months. The employee database has grown, the events table is filling up with analytics data, and the project tracking system is in full swing. But recently, the team has noticed something concerning: **some queries that used to be instant are now taking several seconds**.

Before you can fix performance problems, you need to understand what's happening inside PostgreSQL. This chapter teaches you to diagnose query performance by connecting **EXPLAIN output** to what the planner and executor are actually doing.

In the next chapter, you'll use this knowledge to add indexes and solve the performance problems. But first, let's understand the problem.

## Learning Objectives

By the end of this chapter, you will be able to:
- Trace the complete lifecycle of a query from text to results
- Understand how the parser validates SQL syntax
- Explain how the optimizer evaluates and selects query plans
- Read and interpret EXPLAIN output confidently
- Predict when different scan and join methods will be used
- Diagnose slow queries before fixing them

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

This chapter builds on tables from Chapter 2. We'll also create extended test data to see realistic query plans.

### Chapter Setup: Creating Demonstration Tables

PostgreSQL's query planner makes smart decisions based on table size, data distribution, available indexes, and statistics. The `employees` table from earlier chapters has various indexes and autovacuum activity that can make query plans unpredictable for learning purposes.

For this chapter, we'll create dedicated **demonstration tables** with:
- Controlled data distribution
- **Autovacuum disabled** (we control ANALYZE and VACUUM explicitly)
- No indexes initially (we add them as needed)
- A parent/child relationship for join demonstrations

**🔬 Try It:** Create Demo Tables

```sql
-- Clean up any previous demo tables
DROP TABLE IF EXISTS plan_demo_child CASCADE;
DROP TABLE IF EXISTS plan_demo CASCADE;

-- ============================================
-- PARENT TABLE: plan_demo (100K rows)
-- ============================================
CREATE TABLE plan_demo (
    id SERIAL PRIMARY KEY,
    category_id INT NOT NULL,     -- FK to child table (10 values)
    status INT NOT NULL,          -- 5 distinct values (1-5) 
    amount NUMERIC(10,2) NOT NULL,-- Random values 0-10000
    created_date DATE NOT NULL,   -- Dates over 3 years
    description TEXT              -- Variable length text
);

-- Disable autovacuum for predictable statistics
ALTER TABLE plan_demo SET (autovacuum_enabled = false);

-- ============================================
-- CHILD TABLE: plan_demo_child (10 rows)
-- Small lookup table for join demonstrations
-- ============================================
CREATE TABLE plan_demo_child (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL,
    priority INT NOT NULL         -- 1=High, 2=Medium, 3=Low
);

ALTER TABLE plan_demo_child SET (autovacuum_enabled = false);

-- Insert 10 categories
INSERT INTO plan_demo_child (category_id, category_name, priority)
VALUES 
    (1, 'Electronics', 1),
    (2, 'Clothing', 2),
    (3, 'Food', 1),
    (4, 'Furniture', 3),
    (5, 'Books', 2),
    (6, 'Sports', 2),
    (7, 'Toys', 3),
    (8, 'Health', 1),
    (9, 'Garden', 3),
    (10, 'Automotive', 2);

-- Insert 100,000 rows into parent with predictable distribution
INSERT INTO plan_demo (category_id, status, amount, created_date, description)
SELECT 
    (gs % 10) + 1 AS category_id,           -- 10 categories, ~10K rows each
    (gs % 5) + 1 AS status,                 -- 5 statuses, ~20K rows each
    round((random() * 10000)::numeric, 2),  -- Random amounts
    DATE '2022-01-01' + (gs % 1095),        -- 3 years of dates
    'Item ' || gs
FROM generate_series(1, 100000) AS gs;

-- Gather statistics explicitly (autovacuum won't do it)
ANALYZE plan_demo;
ANALYZE plan_demo_child;

-- Update visibility map for Index Only Scans
VACUUM plan_demo;
VACUUM plan_demo_child;

-- Verify the distribution
SELECT 
    COUNT(*) AS total_rows,
    COUNT(DISTINCT category_id) AS categories,
    COUNT(DISTINCT status) AS statuses
FROM plan_demo;
-- Result: 100000 rows, 10 categories, 5 statuses

SELECT COUNT(*) AS child_rows FROM plan_demo_child;
-- Result: 10 rows
```

**Why these design choices?**

| Choice | Reason |
|--------|--------|
| **Autovacuum disabled** | Statistics and visibility map only change when WE run ANALYZE/VACUUM |
| **Separate demo tables** | No interference from indexes created in other chapters |
| **Parent (100K) + Child (10)** | Classic large/small table join scenario |
| **Known distributions** | category_id: 10% each, status: 20% each—easy to verify estimates |

Throughout this chapter:
- Run `ANALYZE plan_demo;` after creating indexes or inserting data
- Run `VACUUM plan_demo;` before testing Index Only Scans
- When we say "drop and recreate," run the setup above again

---

## Part 1: Query Lifecycle Overview

When you type a SQL query and press Enter, PostgreSQL transforms it through five distinct stages before returning results. Understanding this pipeline helps you diagnose performance issues and write more efficient queries.

```
                    SQL Query (text)
                          │
                          ▼
              ┌───────────────────────┐
              │        Parser         │  "Is this valid SQL syntax?"
              │   (Lexical/Syntax)    │
              └───────────┬───────────┘
                          │ Parse Tree
                          ▼
              ┌───────────────────────┐
              │       Analyzer        │  "Do these tables/columns exist?"
              │   (Semantic Check)    │  "Are the types compatible?"
              └───────────┬───────────┘
                          │ Query Tree
                          ▼
              ┌───────────────────────┐
              │       Rewriter        │  "Expand views and apply rules"
              │    (Views, Rules)     │
              └───────────┬───────────┘
                          │ Rewritten Query
                          ▼
              ┌───────────────────────┐
              │   Planner/Optimizer   │  "What's the fastest way to
              │                       │   execute this?"
              └───────────┬───────────┘
                          │ Plan Tree
                          ▼
              ┌───────────────────────┐
              │       Executor        │  "Run the plan, return results"
              └───────────┬───────────┘
                          │ Results
                          ▼
                       Client
```

**Why does this matter?**

Each stage can be a source of errors or performance issues:
- **Parser errors**: Syntax mistakes like missing commas or typos
- **Analyzer errors**: References to non-existent tables or type mismatches
- **Rewriter**: Views that expand into inefficient queries
- **Planner**: Poor plan choices due to stale statistics
- **Executor**: Memory pressure, lock contention, I/O bottlenecks

---

## Part 2: The Parser

**Source**: `src/backend/parser/`

The parser is PostgreSQL's first line of defense. It answers one question: "Is this syntactically valid SQL?" It doesn't check if tables exist or if types match—just whether the query follows SQL grammar rules.

### Lexical Analysis (Scanner)

The scanner (`scan.l`) breaks your SQL text into **tokens**—the smallest meaningful units:

> ```sql
> SELECT first_name FROM employees WHERE salary > 50000;
> ```

Becomes:
> ```
> Token Type      | Value
> ----------------|------------------
> Keyword         | SELECT
> Identifier      | first_name
> Keyword         | FROM
> Identifier      | employees
> Keyword         | WHERE
> Identifier      | salary
> Operator        | >
> Integer         | 50000
> ```

The scanner handles tricky cases like:
- String literals with quotes: `'Hello, World'`
- Escaped identifiers: `"column-with-dashes"`
- Comments: `-- single line` and `/* multi-line */`
- Numeric formats: `1.5e-10`, `0x1A`, `1_000_000`

### Syntax Analysis (Grammar)

The grammar (`gram.y`) validates that tokens appear in valid combinations and builds a **parse tree**—a structured representation of the query.

**📖 Example:** Invalid syntax fails here:

> ```sql
> -- Missing FROM keyword
> SELECT first_name employees WHERE salary > 50000;
> -- ERROR: syntax error at or near "employees"
> ```

### Viewing the Parse Tree

PostgreSQL doesn't expose parse trees via SQL, but you can see them in the server log:

**🔬 Try It:**
```sql
-- Enable parse tree output (goes to server log, not psql)
SET debug_print_parse = on;
SET client_min_messages = 'log';

-- Run a simple query
SELECT * FROM employees WHERE emp_id = 1;

-- Output in server log shows the raw parse tree:
-- {QUERY :commandType 1 :querySource 0 :canSetTag true ...}

-- Clean up
RESET debug_print_parse;
RESET client_min_messages;
```

**Tip:** If you started PostgreSQL in foreground mode (`postgres -D $PGDATA`), you'll see the output in that terminal. Otherwise, check `$PGDATA/log/` or your configured log destination.

---

## Part 3: The Analyzer (Semantic Analysis)

**Source**: `src/backend/parser/analyze.c`

While the parser checks syntax, the analyzer checks **semantics**—does the query make sense given your actual database?

### What the Analyzer Does

1. **Name Resolution**: Maps identifiers to catalog objects
   - Does table `employees` exist?
   - Does it have column `salary`?
   - Which schema is it in?

2. **Type Checking**: Validates operations make sense
   - Can you compare `salary` (numeric) with `50000` (integer)? Yes.
   - Can you compare `salary` with `'hello'`? No—type mismatch.

3. **Function Resolution**: Finds the right function when multiple exist
   - `length('hello')` → which `length` function? (The text one)
   - `abs(-5)` vs `abs(-5.5)` → different functions for integer vs numeric

4. **Implicit Type Coercion**: Adds automatic type conversions
   - `salary > 50000` where salary is `NUMERIC` and 50000 is `INTEGER`
   - Analyzer adds an implicit cast to make types compatible

### Observing the Analyzer

**🔬 Try It:**
```sql
-- Debug output goes to the SERVER LOG, not psql
-- To see it, check $PGDATA/log/ or run PostgreSQL in foreground mode

SET debug_print_rewritten = on;
SET client_min_messages = 'debug1';

-- Run a simple query
SELECT * FROM plan_demo WHERE id = 1;

RESET debug_print_rewritten;
RESET client_min_messages;
```

**To see the debug output:**
1. If you started PostgreSQL with `postgres -D $PGDATA` (foreground), output appears in that terminal
2. Otherwise, check `$PGDATA/log/` for the latest log file
3. Or temporarily enable logging to stderr:

**🔬 Try It:** Temporarily log to stderr (optional)

```sql
ALTER SYSTEM SET log_destination = 'stderr';
SELECT pg_reload_conf();
```

**📖 Example:** Identify Error Sources

> ```sql
> -- 1. Parser error (syntax) - "SELEC" is not a keyword
> SELEC * FROM employees;
> -- ERROR: syntax error at or near "SELEC"
> 
> -- 2. Analyzer error (semantic) - table doesn't exist  
> SELECT * FROM nonexistent_table;
> -- ERROR: relation "nonexistent_table" does not exist
> 
> -- 3. Analyzer error (type) - can't compare number to incompatible string
> SELECT * FROM employees WHERE salary > 'not-a-number';
> -- ERROR: invalid input syntax for type numeric: "not-a-number"
> ```

---

## Part 4: The Rewriter

**Source**: `src/backend/rewrite/`

The rewriter transforms the query tree by expanding views and applying rules. It's PostgreSQL's macro system for queries.

### View Expansion

When you query a view, the rewriter replaces the view reference with its definition:

**🔬 Try It:**
```sql
-- Create a view
CREATE OR REPLACE VIEW high_earners AS
SELECT emp_id, first_name, last_name, salary, dept_id
FROM employees
WHERE salary > 100000;

-- Query the view
EXPLAIN SELECT * FROM high_earners WHERE dept_id = 1;
```

The rewriter transforms this into:
> ```sql
> SELECT emp_id, first_name, last_name, salary
> FROM employees
> WHERE salary > 100000 AND dept_id = 1;
> ```

The planner never sees `high_earners`—it only sees the expanded query against `employees`.

### Rule System

PostgreSQL's rule system allows you to intercept and modify queries. Views are actually implemented using rules internally. While you can create custom rules, **triggers are generally preferred** for most use cases—they're easier to understand and debug.

**🔬 Try It:**
```sql
-- Example: Redirect INSERTs on a view to the base table
CREATE OR REPLACE VIEW active_employees AS
SELECT emp_id, first_name, last_name, salary, dept_id, hire_date
FROM employees
WHERE salary > 0;  -- Only "active" employees

-- By default, you can't INSERT into a view
-- A rule makes the view updatable:
CREATE OR REPLACE RULE insert_active_employee AS
ON INSERT TO active_employees
DO INSTEAD
INSERT INTO employees (first_name, last_name, salary, dept_id, hire_date)
VALUES (NEW.first_name, NEW.last_name, NEW.salary, NEW.dept_id, NEW.hire_date);

-- Now this works:
INSERT INTO active_employees (first_name, last_name, salary, dept_id, hire_date)
VALUES ('Rule', 'Test', 75000, 1, current_date);

-- Verify it went to the base table
SELECT * FROM employees WHERE first_name = 'Rule';

-- Clean up
DROP RULE insert_active_employee ON active_employees;
DROP VIEW active_employees;
```

### Security Expansion (Row-Level Security)

Row-Level Security (RLS) policies are applied during the rewrite phase. The rewriter automatically adds WHERE clauses to enforce access control.

The key insight: RLS is invisible to the application. The query `SELECT * FROM employees` is transparently rewritten to include the security filter. This happens in the rewriter, before the planner ever sees the query.

---

## Part 5: The Planner/Optimizer

**Source**: `src/backend/optimizer/`

The planner is PostgreSQL's brain. Given a query, it must decide *how* to execute it efficiently. There are usually many ways to execute the same query—the planner's job is to find the cheapest one.

### Why Planning Matters

Consider this simple query:
> ```sql
> SELECT * FROM employees WHERE salary > 100000;
> ```

Options include:
1. **Sequential scan**: Read every row, check the condition
2. **Index scan**: Use `idx_emp_salary` to find matching rows
3. **Bitmap scan**: Build a bitmap of matching pages, then fetch them

The best choice depends on:
- How many rows match? (selectivity)
- How large is the table?
- Is the data cached in memory?
- How selective is the condition?

### Planner Directory Structure

```
src/backend/optimizer/
├── geqo/       # Genetic Query Optimizer (for 12+ table joins)
├── path/       # Generates access paths (scan/join methods)
├── plan/       # Converts best path to executable plan
├── prep/       # Query preprocessing and simplification
└── util/       # Utility functions (statistics, etc.)
```

### Cost-Based Optimization

PostgreSQL uses **cost-based optimization**: it estimates the "cost" of each possible plan and picks the cheapest. Cost is measured in abstract units roughly corresponding to disk page fetches.

**🔬 Try It:**
```sql
-- View key cost parameters
SHOW seq_page_cost;        -- 1.0  (baseline: sequential disk read)
SHOW random_page_cost;     -- 4.0  (random I/O is ~4x more expensive)
SHOW cpu_tuple_cost;       -- 0.01 (processing one row)
SHOW cpu_index_tuple_cost; -- 0.005 (processing one index entry)
SHOW cpu_operator_cost;    -- 0.0025 (evaluating one operator)
```

**Reading these values:**
- Sequential I/O (reading pages in order) costs 1.0 per page
- Random I/O (jumping around) costs 4.0 per page—mechanical disks hate this
- CPU costs are much smaller than I/O costs
- For SSDs, you might lower `random_page_cost` to 1.1-2.0

---

## Part 6: Understanding EXPLAIN

`EXPLAIN` is your window into the planner's decisions. Master it, and you can diagnose almost any performance issue.

### Basic EXPLAIN

**🔬 Try It:**
```sql
-- Ensure we have an index for this example
CREATE INDEX IF NOT EXISTS idx_demo_category ON plan_demo(category_id);

ANALYZE plan_demo;

EXPLAIN SELECT * FROM plan_demo WHERE category_id = 1;
```

The plan should look something like the following:

> ```
> Bitmap Heap Scan on plan_demo  (cost=115.01..1854.51 rows=10000 width=52)
>   Recheck Cond: (category_id = 1)
>   ->  Bitmap Index Scan on idx_demo_category  (cost=0.00..112.51 rows=10000 width=0)
>         Index Cond: (category_id = 1)
> ```

### EXPLAIN Output Explained

**Node format**: `NodeType on table (cost=startup..total rows=estimate width=bytes)`

| Field | Meaning |
|-------|---------|
| `cost=0.00..112.51` | Estimated cost: startup cost (first row) to total cost |
| `rows=10000` | Estimated number of rows this node will return |
| `width=52` | Estimated average row size in bytes |

**Startup cost** is important for operations that must process all input before producing output (like Sort). **Total cost** is what matters for comparing plans.

### EXPLAIN ANALYZE

Add `ANALYZE` to actually run the query and see real numbers:

**🔬 Try It:**
```sql
EXPLAIN ANALYZE SELECT * FROM plan_demo WHERE category_id = 1;
```

The plan should look something like the following:

> ```
> Bitmap Heap Scan on plan_demo  (cost=115.01..1854.51 rows=10000 width=52) 
>                                (actual time=1.234..12.456 rows=10000 loops=1)
>   Recheck Cond: (category_id = 1)
>   Heap Blocks: exact=541
>   ->  Bitmap Index Scan on idx_demo_category  (cost=0.00..112.51 rows=10000 width=0) 
>                                               (actual time=0.789..0.790 rows=10000 loops=1)
>         Index Cond: (category_id = 1)
> Planning Time: 0.089 ms
> Execution Time: 13.234 ms
> ```

**New fields with ANALYZE:**
| Field | Meaning |
|-------|---------|
| `actual time=1.234..12.456` | Real time in ms (startup..total) |
| `rows=10000` | Actual rows returned (compare to estimate!) |
| `loops=1` | How many times this node executed |
| `Heap Blocks: exact=541` | Pages read from heap |

### EXPLAIN with BUFFERS

See memory and disk I/O details:

**🔬 Try It:**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM plan_demo WHERE category_id = 1;
```

The plan should look something like the following:

> ```
> Bitmap Heap Scan on plan_demo  (cost=115.01..1854.51 rows=10000 width=52) 
>                                (actual time=1.234..12.456 rows=10000 loops=1)
>   Recheck Cond: (category_id = 1)
>   Heap Blocks: exact=541
>   Buffers: shared hit=555
>   ...
> ```

**Buffer metrics:**
| Field | Meaning |
|-------|---------|
| `shared hit=555` | Pages found in PostgreSQL's buffer cache |
| `shared read=X` | Pages read from disk (OS may have cached) |
| `shared written=X` | Pages written (dirty buffers) |
| `temp read/written` | Temp file I/O (bad—means work_mem too small) |

**Rule of thumb**: `shared hit` is good (fast), `shared read` means disk I/O, `temp` means you're spilling to disk.

### All EXPLAIN Options

**🔬 Try It:**
```sql
-- JSON format (great for programmatic analysis)
EXPLAIN (FORMAT JSON) SELECT * FROM plan_demo;

-- YAML format
EXPLAIN (FORMAT YAML) SELECT * FROM plan_demo;

-- Everything
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING, FORMAT TEXT) 
SELECT * FROM plan_demo WHERE category_id = 1;
```

---

## Part 7: Statistics and Selectivity

The planner makes decisions based on statistics. Bad statistics → bad plans.

### Collecting Statistics

**🔬 Try It:**
```sql
-- Update statistics for a table
ANALYZE plan_demo;

-- Update statistics for specific columns
ANALYZE plan_demo(category_id, amount);

-- Increase statistics detail for a column (default is 100)
ALTER TABLE plan_demo ALTER COLUMN amount SET STATISTICS 500;

ANALYZE plan_demo;
```

### Viewing Statistics

**🔬 Try It:**
```sql
-- Table-level statistics
SELECT 
    relname AS table_name,
    reltuples::bigint AS row_estimate,
    relpages AS page_count,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relname = 'plan_demo';

-- Column statistics (plan_demo has predictable distribution)
SELECT 
    attname AS column_name,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    correlation
FROM pg_stats
WHERE tablename = 'plan_demo'
  AND attname IN ('category_id', 'status');
-- category_id: n_distinct = 10, each value appears ~10% of time
-- status: n_distinct = 5, each value appears ~20% of time
```

**Key statistics:**
| Statistic | Meaning |
|-----------|---------|
| `n_distinct` | Number of distinct values (-1 = unique, negative = fraction of rows) |
| `most_common_vals` | Most frequent values (helps estimate equality conditions) |
| `most_common_freqs` | Frequencies of most common values |
| `histogram_bounds` | Value distribution (helps estimate range conditions) |
| `correlation` | Physical vs logical ordering (-1 to 1; high means clustered) |

### Selectivity Estimation

**Selectivity** is the fraction of rows a condition will match. The planner estimates this from statistics:


**🔬 Try It:**
```sql
-- plan_demo has 100K rows with 10 categories (evenly distributed)
EXPLAIN SELECT * FROM plan_demo WHERE category_id = 5;
-- Estimates ~10,000 rows (100K / 10 categories = 10%)

-- status has 5 values
EXPLAIN SELECT * FROM plan_demo WHERE status = 1;
-- Estimates ~20,000 rows (100K / 5 statuses = 20%)

-- Combined conditions
EXPLAIN SELECT * FROM plan_demo WHERE category_id = 5 AND status = 1;
-- Estimates ~2,000 rows (10% × 20% = 2%)
```

**Exercise 6.2: Statistics Accuracy**

**🔬 Try It:**
```sql
-- Check estimated vs actual rows
EXPLAIN ANALYZE SELECT * FROM plan_demo WHERE amount BETWEEN 1000 AND 2000;

-- Compare estimated "rows=" with actual "rows="
-- They should be close since we just ran ANALYZE

-- Force stale statistics by adding data without ANALYZE:
INSERT INTO plan_demo (category_id, status, amount, created_date, description)
SELECT 1, 1, 9999.99, current_date, 'New high-value item'
FROM generate_series(1, 1000);

-- Estimates will be slightly off now
EXPLAIN ANALYZE SELECT * FROM plan_demo WHERE amount > 9000;

-- Fix with ANALYZE
ANALYZE plan_demo;

EXPLAIN ANALYZE SELECT * FROM plan_demo WHERE amount > 9000;
```

---

## Part 8: Scan Methods

The planner chooses how to access table data. Each method has trade-offs.

**🔬 Try It:**
```sql
-- Start this section with a clean slate: drop any indexes from earlier sections
DROP INDEX IF EXISTS idx_demo_category;
DROP INDEX IF EXISTS idx_demo_cat_amt;
DROP INDEX IF EXISTS idx_demo_status;

ANALYZE plan_demo;
```

### Sequential Scan

Reads every page of the table in order. Simple and efficient for:
- Small tables
- Queries that need most rows
- No useful index exists

Even when an index exists, PostgreSQL may still choose a sequential scan if it estimates that reading a large portion of the table is cheaper than lots of random index lookups.

**🔬 Try It:**
```sql
-- With no indexes, planner must use Seq Scan
EXPLAIN SELECT * FROM plan_demo WHERE category_id = 5;
-- Seq Scan on plan_demo  (cost=0.00..2084.00 rows=9743 width=32)
--   Filter: (category_id = 5)
```

**When sequential scan wins:**
- Reading >10-20% of a large table
- Table is small (fits in a few pages)
- Sequential I/O is much faster than random I/O

### Index Scan

Uses an index to find specific rows, then fetches each row from the table (heap).

This is best when the predicate is highly selective (returns a small fraction of rows). The trade-off is random I/O: each matching row can require an extra heap page fetch.

**🔬 Try It:**
```sql
-- Index Scan is best for highly selective queries (few rows)
-- The primary key index always uses Index Scan for single-row lookups:
EXPLAIN SELECT * FROM plan_demo WHERE id = 12345;
-- Index Scan using plan_demo_pkey on plan_demo  (cost=0.29..8.31 rows=1 width=32)
--   Index Cond: (id = 12345)

-- For less selective conditions, you can force Index Scan:
CREATE INDEX idx_demo_category ON plan_demo(category_id);
ANALYZE plan_demo;

SET enable_seqscan = off;
SET enable_bitmapscan = off;
EXPLAIN SELECT * FROM plan_demo WHERE category_id = 5;
-- Index Scan using idx_demo_category on plan_demo  (cost=0.29..3513.09 rows=9840 width=32)
--   Index Cond: (category_id = 5)

RESET enable_seqscan;
RESET enable_bitmapscan;
```

**When index scan wins:**
- Fetching very few rows (< 1-2% of table)
- LIMIT clause limits rows needed
- Rows are clustered by index order

> **Note**: For moderate selectivity (5-20%), PostgreSQL often prefers Bitmap Scan.

### Bitmap Scan

A two-phase approach: build a bitmap of matching pages, then fetch them in physical order. Preferred for moderate selectivity.

Bitmap scans are a “compromise plan”: you still use indexes to find matches, but you batch the heap reads to reduce random I/O.

**🔬 Try It:**
```sql
EXPLAIN SELECT * FROM plan_demo WHERE category_id = 5;

-- Bitmap Scan shines with OR conditions (it can combine multiple indexes)
CREATE INDEX idx_demo_status ON plan_demo(status);

ANALYZE plan_demo;

EXPLAIN SELECT * FROM plan_demo 
WHERE category_id = 1 OR status = 5;
```

**What to look for in the plan output:**
- You should see **`Bitmap Heap Scan`** with one or more **`Bitmap Index Scan`** nodes underneath.
- For the OR query, you’ll typically see **`BitmapOr`** combining the two index bitmaps.

**When bitmap scan wins:**
- Moderate selectivity (5-20% of table)
- Multiple index conditions (OR, AND)
- Reduces random I/O by batching heap access

### Index Only Scan

An index-only scan can answer the query entirely from the index without touching the table:

The key is that the executor can return results from the index *only if it can prove the heap tuple is visible*. That’s what the visibility map is for, which is why `VACUUM` matters here.

**🔬 Try It:**
```sql
-- Create a covering index (includes all columns we need)
CREATE INDEX idx_demo_cat_amt ON plan_demo(category_id, amount);

VACUUM plan_demo;  -- Updates visibility map

-- Query only needs columns that are IN the index
EXPLAIN SELECT category_id, amount FROM plan_demo WHERE category_id = 5;
-- Index Only Scan using idx_demo_cat_amt on plan_demo  (cost=0.42..338.10 rows=10153 width=10)
--   Index Cond: (category_id = 5)
```

When you run this with `EXPLAIN ANALYZE`, look for `Heap Fetches: 0` which means no table access was needed.

**Requirements:**
- All columns in SELECT and WHERE must be in the index
- Table must be recently VACUUMed (visibility map up to date)
- `Heap Fetches: 0` means perfect index-only scan

**When bitmap scan wins:**
- Multiple index conditions (ORs or ANDs)
- Moderate selectivity (too many rows for index scan, too few for seq scan)
- Reduces random I/O by sorting page accesses

### TID Scan

Direct row access by physical location (tuple ID):

**🔬 Try It:**
```sql
-- Direct tuple access (rare in application code)
EXPLAIN SELECT * FROM plan_demo WHERE ctid = '(0,1)'::tid;
-- Tid Scan on plan_demo
--   TID Cond: (ctid = '(0,1)'::tid)
```

TID scans are mainly used internally (e.g., by UPDATE/DELETE after finding rows).

### Comparing Scan Methods

**🔬 Try It:**
```sql
-- Drop indexes to see Seq Scan
DROP INDEX IF EXISTS idx_demo_category;
DROP INDEX IF EXISTS idx_demo_cat_amt;
DROP INDEX IF EXISTS idx_demo_status;

ANALYZE plan_demo;

EXPLAIN SELECT * FROM plan_demo WHERE category_id = 5;
-- Seq Scan (no index available)

-- Recreate index
CREATE INDEX idx_demo_category ON plan_demo(category_id);

ANALYZE plan_demo;

EXPLAIN SELECT * FROM plan_demo WHERE category_id = 5;
-- Index Scan (planner chooses index for 10% selectivity)
```

---

## Part 9: Join Algorithms

PostgreSQL supports three join algorithms. The planner picks based on table sizes, indexes, and sort order. We'll use `plan_demo` (100K rows) and `plan_demo_child` (10 rows) for predictable join demonstrations.

**🔬 Try It:**
```sql
-- Start clean: drop any indexes on join columns
DROP INDEX IF EXISTS idx_demo_category;
DROP INDEX IF EXISTS idx_demo_child_cat;
ANALYZE plan_demo;
ANALYZE plan_demo_child;
```

### Nested Loop Join

The simplest algorithm: for each row in the outer table, scan the inner table for matches.

**🔬 Try It:**
```sql
-- Create index on child table for efficient inner lookups
CREATE INDEX idx_demo_child_cat ON plan_demo_child(category_id);

ANALYZE plan_demo_child;

-- The planner prefers Hash/Merge joins even for small results.
-- To demonstrate Nested Loop, disable both alternatives:
SET enable_hashjoin = off;
SET enable_mergejoin = off;

EXPLAIN SELECT p.id, p.amount, c.category_name
FROM plan_demo p
JOIN plan_demo_child c ON p.category_id = c.category_id
WHERE p.id < 10;
-- Nested Loop  (cost=0.29..10.63 rows=8 width=17)
--   Join Filter: (c.category_id = p.category_id)
--   ->  Index Scan using plan_demo_pkey on plan_demo p
--         Index Cond: (id < 10)
--   ->  Materialize
--         ->  Seq Scan on plan_demo_child c

RESET enable_hashjoin;
RESET enable_mergejoin;
```

**Pseudocode:**
```
for each row in outer_table:         # 9 rows (filtered by id < 10)
    for each row in inner_table:     # scans all 10 child rows
        if join_condition matches:
            output combined row
```

**Best when:**
- Outer table is very small (or heavily filtered)
- Inner table has an index on the join column
- LIMIT clause reduces rows needed

> **Note**: PostgreSQL often prefers Hash Join even for small tables because building a tiny hash table is very fast.

### Hash Join

Builds a hash table from the smaller table, then probes it with the larger table. This is often the default choice for equi-joins.

**🔬 Try It:**
```sql
-- Drop index so there's no sorted access path
DROP INDEX IF EXISTS idx_demo_child_cat;

ANALYZE plan_demo_child;

-- Hash Join: hash the small table (10 rows), probe with large (100K)
EXPLAIN SELECT p.id, p.amount, c.category_name
FROM plan_demo p
JOIN plan_demo_child c ON p.category_id = c.category_id;
-- Hash Join  (cost=1.23..2208.97 rows=100000 width=17)
--   Hash Cond: (p.category_id = c.category_id)
--   ->  Seq Scan on plan_demo p  (100K rows, probe side)
--   ->  Hash
--         ->  Seq Scan on plan_demo_child c  (10 rows, build side)
```

**Pseudocode:**
```
# Build phase: hash the smaller table (10 rows)
hash_table = {}
for each row in plan_demo_child:
    hash_table[category_id] = row

# Probe phase: scan larger table (100K), look up matches
for each row in plan_demo:
    if category_id in hash_table:
        output combined row
```

**Best when:**
- No useful indexes on join columns
- Joining medium-to-large tables
- Equi-join (=) conditions
- Smaller table fits in work_mem

### Merge Join

Sorts both inputs, then merges them like a zipper. Efficient when data is already sorted via indexes.

**🔬 Try It:**
```sql
-- Create indexes to provide sorted access
CREATE INDEX idx_demo_category ON plan_demo(category_id);
CREATE INDEX idx_demo_child_cat ON plan_demo_child(category_id);
ANALYZE plan_demo;
ANALYZE plan_demo_child;

-- Disable Hash and Nested Loop to force Merge Join
SET enable_hashjoin = off;
SET enable_nestloop = off;

EXPLAIN SELECT p.id, p.amount, c.category_name
FROM plan_demo p
JOIN plan_demo_child c ON p.category_id = c.category_id;
-- Merge Join  (cost=0.43..6421.98 rows=100000 width=17)
--   Merge Cond: (p.category_id = c.category_id)
--   ->  Index Scan using idx_demo_category on plan_demo p
--   ->  Index Scan using idx_demo_child_cat on plan_demo_child c

RESET enable_hashjoin;
RESET enable_nestloop;
```

**Pseudocode:**
```
sort left_table by join_key    # (or use index for pre-sorted access)
sort right_table by join_key   # (or use index for pre-sorted access)
while both have rows:
    if left_key == right_key:
        output combined row
        advance both
    elif left_key < right_key:
        advance left
    else:
        advance right
```

**Best when:**
- Inputs are already sorted (from indexes or prior Sort)
- Very large tables where hash table won't fit in memory
- Non-equi joins (<, >, BETWEEN)

### Join Algorithm Comparison

| Algorithm | Build Cost | Memory | Best For |
|-----------|------------|--------|----------|
| Nested Loop | None | Minimal | Small outer, indexed inner |
| Hash Join | O(smaller) | Hash table size | Equi-joins, no index |
| Merge Join | O(n log n) × 2 | Sort buffers | Pre-sorted data, large tables |

---

## Part 10: Aggregation

Using `plan_demo` for predictable aggregation plans.

### Plain Aggregate

Simple aggregation without grouping:

**🔬 Try It:**
```sql
EXPLAIN SELECT COUNT(*), AVG(amount), MAX(amount) FROM plan_demo;
-- Aggregate
--   -> Seq Scan on plan_demo
```

### Hash Aggregate vs Group Aggregate

PostgreSQL has two strategies for GROUP BY:

| Strategy | How it Works | Best When |
|----------|--------------|-----------|
| **HashAggregate** | Build hash table keyed by GROUP BY columns | Few to moderate groups, fits in work_mem |
| **GroupAggregate** | Process pre-sorted rows, output when key changes | Data already sorted, many groups, or low work_mem |

**🔬 Try It:**
```sql
-- First, ensure no index on category_id exists
DROP INDEX IF EXISTS idx_demo_category;
ANALYZE plan_demo;

-- With no index, planner uses HashAggregate (10 groups easily fit in memory)
EXPLAIN SELECT category_id, COUNT(*), AVG(amount) 
FROM plan_demo 
GROUP BY category_id;
-- HashAggregate
--   Group Key: category_id
--   -> Seq Scan on plan_demo

-- Now add an index on category_id
CREATE INDEX idx_demo_category ON plan_demo(category_id);
ANALYZE plan_demo;

-- With an index providing sorted access, planner may choose GroupAggregate
-- To guarantee GroupAggregate, disable hash aggregation:
SET enable_hashagg = off;
EXPLAIN SELECT category_id, COUNT(*), AVG(amount) 
FROM plan_demo 
GROUP BY category_id;
-- GroupAggregate
--   Group Key: category_id
--   -> Index Only Scan using idx_demo_category (or Index Scan + Sort)

RESET enable_hashagg;
```

**HashAggregate internals:**
1. Scan rows and hash by GROUP BY columns
2. Update running aggregates for each group in the hash table
3. Output all groups when scan completes

**GroupAggregate internals:**
1. Read rows in sorted order (from index or Sort node)
2. Accumulate aggregates while key matches
3. Output group when key changes, start new group

---

## Part 11: Subqueries and CTEs

Continuing with `plan_demo` and `plan_demo_child`.

### Subquery Processing

PostgreSQL is smart about subqueries—it often transforms them into joins:

**🔬 Try It:**
```sql
-- Subquery with IN: find items in high-priority categories
EXPLAIN SELECT * FROM plan_demo p
WHERE p.category_id IN (SELECT category_id FROM plan_demo_child WHERE priority = 1);
```

What to look for:
- PostgreSQL often rewrites `IN (subquery)` as a **semi-join**.
- In the plan, you’ll typically see a **`Hash Semi Join`** or **`Nested Loop Semi Join`** depending on sizes and settings.

**Semi-join**: Returns outer rows that have at least one match (no duplicates from inner).

### Correlated Subqueries

A correlated subquery references the outer query—it must execute once per outer row:

**🔬 Try It:**
```sql
EXPLAIN SELECT p.id, p.amount,
    (SELECT AVG(amount) FROM plan_demo p2 WHERE p2.category_id = p.category_id) AS cat_avg
FROM plan_demo p
WHERE p.id < 20;
```

What to look for:
- The plan will usually show a **`SubPlan`** node for the correlated subquery.
- The correlated filter (`p2.category_id = p.category_id`) appears inside the subplan.

**Performance warning**: SubPlan executes for each outer row. Without the `WHERE p.id < 20`, that's 100K subquery executions! Consider rewriting as a JOIN:

**🔬 Try It:**
```sql
-- More efficient: JOIN with derived table
EXPLAIN SELECT p.id, p.amount, cat_avgs.avg_amount
FROM plan_demo p
JOIN (
    SELECT category_id, AVG(amount) AS avg_amount 
    FROM plan_demo 
    GROUP BY category_id
) cat_avgs ON p.category_id = cat_avgs.category_id
WHERE p.id < 20;
-- Hash Join (much faster—subquery runs once, not 19 times)
```

### Common Table Expressions (CTEs)

CTEs (WITH clauses) can be **inlined** or **materialized**:

This matters because it changes both performance and plan shape:
- **Inlined CTE**: treated like a subquery; filters can be pushed down; often faster
- **Materialized CTE**: computed once and stored; can be faster when reused; acts as an optimization barrier

**🔬 Try It:**
```sql
-- PostgreSQL 12+ inlines single-reference CTEs by default
EXPLAIN WITH high_value AS (
    SELECT * FROM plan_demo WHERE amount > 8000
)
SELECT * FROM high_value WHERE category_id = 1;
-- Bitmap Heap Scan on plan_demo  (cost=112.56..1098.10 rows=1979 width=32)
--   Recheck Cond: (category_id = 1)
--   Filter: (amount > '8000'::numeric)
--   ->  Bitmap Index Scan on idx_demo_category
-- Note: CTE was inlined—both conditions are applied to plan_demo!

-- Force materialization with MATERIALIZED keyword
EXPLAIN WITH high_value AS MATERIALIZED (
    SELECT * FROM plan_demo WHERE amount > 8000
)
SELECT * FROM high_value WHERE category_id = 1;
-- CTE Scan on high_value  (cost=2084.00..2524.84 rows=1980 width=64)
--   Filter: (category_id = 1)
--   CTE high_value
--     ->  Seq Scan on plan_demo
--           Filter: (amount > '8000'::numeric)
-- Note: CTE is computed first, then filtered
```

**When to use MATERIALIZED:**
- CTE is expensive and referenced multiple times
- You want to prevent the planner from pushing filters into the CTE
- For optimization barriers (rare)

---

## Part 12: Parallel Query Execution

PostgreSQL can split work across multiple CPU cores for large operations.

### Parallel Query Basics

**🔬 Try It:**
```sql
-- Check parallel settings
SHOW max_parallel_workers_per_gather;  -- Workers per query (default: 2)
SHOW min_parallel_table_scan_size;     -- Min size for parallel (default: 8MB)

-- Check our table size
SELECT pg_size_pretty(pg_relation_size('plan_demo')) as table_size;
-- ~6.7 MB (less than 8MB threshold, so no parallel by default)
```

### Parallel Scan in Action

Our `plan_demo` table (100K rows, ~6.7MB) is smaller than the default 8MB threshold. To demonstrate parallel execution, we lower the thresholds:

**🔬 Try It:**
```sql
-- Lower thresholds to enable parallel on our table
SET min_parallel_table_scan_size = 0;
SET parallel_tuple_cost = 0;
SET parallel_setup_cost = 0;

EXPLAIN SELECT COUNT(*) FROM plan_demo;
-- Finalize Aggregate  (cost=1354.85..1354.86 rows=1 width=8)
--   ->  Gather  (cost=1354.83..1354.84 rows=2 width=8)
--         Workers Planned: 2
--         ->  Partial Aggregate  (cost=1354.83..1354.84 rows=1 width=8)
--               ->  Parallel Seq Scan on plan_demo

RESET min_parallel_table_scan_size;
RESET parallel_tuple_cost;
RESET parallel_setup_cost;
```

> **Note**: In production with larger tables (>8MB), parallel query kicks in automatically.

**How parallel query works:**
1. **Gather** node coordinates parallel workers
2. Each worker runs **Parallel Seq Scan** on a portion of the table
3. **Partial Aggregate** computes partial results per worker
4. **Finalize Aggregate** combines partial results

### Parallel Operations

Not all operations can be parallelized:

| Operation | Parallel? | Notes |
|-----------|-----------|-------|
| Seq Scan | ✅ | Splits table pages among workers |
| Index Scan | ✅ | Each worker scans different index ranges |
| Hash Join | ✅ | Parallel hash build and probe |
| Merge Join | ✅ | Workers produce sorted streams |
| Aggregate | ✅ | Partial aggregate → finalize |
| Sort | ⚠️ | Limited parallelism |
| Writes (INSERT/UPDATE) | ❌ | Not parallelized |

---

## Part 13: Plan Caching and Prepared Statements

Prepared statements cache query plans to avoid repeated planning:

**🔬 Try It:**
```sql
-- Prepare a parameterized statement
PREPARE demo_by_category(int) AS
SELECT * FROM plan_demo WHERE category_id = $1;

-- First 5 executions: PostgreSQL creates custom plans
EXECUTE demo_by_category(1);
EXECUTE demo_by_category(2);
EXECUTE demo_by_category(3);
EXECUTE demo_by_category(4);
EXECUTE demo_by_category(5);
EXECUTE demo_by_category(6);

-- After 5 executions: may switch to generic plan
-- Generic plans work for any parameter value

-- View prepared statements
SELECT name, statement, parameter_types, 
       generic_plans, custom_plans
FROM pg_prepared_statements;

-- Deallocate when done
DEALLOCATE demo_by_category;
```

**Custom vs Generic plans:**
- **Custom plan**: Optimized for specific parameter value (e.g., knows `category_id=1` matches 10K rows)
- **Generic plan**: Works for any value (uses average selectivity estimate)

Generic plans save planning time but may be suboptimal for skewed distributions.

---

## Part 14: The Executor

**Source**: `src/backend/executor/`

The executor runs the plan tree using a **pull model** (also called "iterator" or "Volcano" model):

```
            Request tuple
         ┌──────────────────┐
         │   Client/Cursor  │
         └────────┬─────────┘
                  │ "Give me a row"
                  ▼
         ┌──────────────────┐
         │  Aggregate Node  │──────▶ Returns aggregated row
         └────────┬─────────┘
                  │ "Give me rows to aggregate"
                  ▼
         ┌──────────────────┐
         │  Seq Scan Node   │──────▶ Returns one row at a time
         └──────────────────┘
                  │ Reads from table
```

### Key Executor Source Files

| Node | File | Purpose |
|------|------|---------|
| SeqScan | `nodeSeqscan.c` | Sequential table scan |
| IndexScan | `nodeIndexscan.c` | Index-based scan |
| BitmapHeapScan | `nodeBitmapHeapscan.c` | Bitmap scan heap access |
| NestLoop | `nodeNestloop.c` | Nested loop join |
| HashJoin | `nodeHashjoin.c` | Hash join |
| MergeJoin | `nodeMergejoin.c` | Merge join |
| Sort | `nodeSort.c` | Sorting |
| Aggregate | `nodeAgg.c` | Aggregation |
| Gather | `nodeGather.c` | Parallel query coordination |

### Advanced: JIT Compilation

PostgreSQL supports JIT (Just-In-Time) compilation, which compiles query expressions to native machine code at runtime. This can speed up CPU-bound analytical queries with complex expressions.

JIT requires PostgreSQL to be compiled with LLVM support (`--with-llvm`). Since LLVM configuration can be complex and version-sensitive, our build in Chapter 1 does not include JIT. This has no impact on learning PostgreSQL—JIT is a performance optimization primarily beneficial for data warehouse workloads.

If you're interested in JIT, see the PostgreSQL documentation: https://www.postgresql.org/docs/current/jit.html

---

## Part 15: Practical Query Optimization

### Finding Slow Queries

**Method 1: Query Logging**

**🔬 Try It:**
```sql
-- Log queries taking longer than 1000ms
ALTER SYSTEM SET log_min_duration_statement = '1000ms';
SELECT pg_reload_conf();
-- Check $PGDATA/log/ for slow query logs
```

**Method 2: pg_stat_statements (Recommended)**

If you set up `pg_stat_statements` in Chapter 1:

**🔬 Try It:**
```sql
-- Top 10 queries by total time
SELECT 
    substring(query, 1, 60) AS query_preview,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with worst average time
SELECT 
    substring(query, 1, 60) AS query_preview,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms
FROM pg_stat_statements
WHERE calls > 100
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Reset after optimization changes
SELECT pg_stat_statements_reset();
```

> **Note:** If `pg_stat_statements` isn't available, see [Chapter 1](01-getting-started.md#setting-up-pg_stat_statements-recommended) for setup instructions.

### Common Optimization Techniques

Using `plan_demo` for demonstrations:

**🔬 Try It:**
```sql
-- Start fresh for these examples
DROP INDEX IF EXISTS idx_demo_category;
DROP INDEX IF EXISTS idx_demo_status;
ANALYZE plan_demo;

-- 1. Identify missing indexes (seq scan on large table with selective filter)
EXPLAIN ANALYZE SELECT * FROM plan_demo WHERE description LIKE 'Item 999%';
-- Seq Scan (no index on description)

-- For LIKE with prefix patterns, use text_pattern_ops operator class
-- (Regular B-tree indexes don't support LIKE due to collation)
CREATE INDEX idx_demo_description ON plan_demo(description text_pattern_ops);

ANALYZE plan_demo;

EXPLAIN ANALYZE SELECT * FROM plan_demo WHERE description LIKE 'Item 9999%';
-- Now uses Index Scan (with text_pattern_ops)

-- 2. Use covering indexes to avoid heap access
CREATE INDEX idx_demo_cat_covering ON plan_demo(category_id) INCLUDE (amount, status);

VACUUM plan_demo;  -- Update visibility map

EXPLAIN SELECT category_id, amount, status FROM plan_demo WHERE category_id = 1;
-- Index Only Scan (Heap Fetches: 0)

-- 3. Partial indexes for common queries
CREATE INDEX idx_demo_high_value ON plan_demo(amount) WHERE amount > 9000;

ANALYZE plan_demo;

EXPLAIN SELECT * FROM plan_demo WHERE amount > 9000;
-- Uses the smaller partial index

-- 4. Keep statistics fresh (especially with autovacuum disabled!)
ANALYZE plan_demo;

-- 5. Check for tables with too many seq scans
SELECT 
    relname,
    seq_scan,
    idx_scan,
    CASE WHEN idx_scan > 0 
         THEN round(seq_scan::numeric / idx_scan, 2) 
         ELSE seq_scan 
    END AS seq_to_idx_ratio
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_to_idx_ratio DESC;
```

### Exercise 6.3: Optimize a Multi-Table Query

**🔬 Try It:**
```sql
-- Start with this query using our demo tables
EXPLAIN ANALYZE
SELECT p.id, p.amount, c.category_name, c.priority
FROM plan_demo p
JOIN plan_demo_child c ON p.category_id = c.category_id
WHERE p.amount > 5000
  AND c.priority = 1
GROUP BY p.id, p.amount, c.category_name, c.priority;

-- Steps:
-- 1. Note the execution time and plan type
-- 2. Look for Seq Scans that could be Index Scans
-- 3. Check join order and algorithm
-- 4. Try adding indexes:
--    CREATE INDEX idx_demo_amount ON plan_demo(amount);
--    CREATE INDEX idx_demo_child_priority ON plan_demo_child(priority);
--    ANALYZE plan_demo; ANALYZE plan_demo_child;
-- 5. Re-run EXPLAIN ANALYZE and compare
```

---

## Source Code Deep Dive

| Stage | Key Files |
|-------|-----------|
| Lexer | `src/backend/parser/scan.l` |
| Parser | `src/backend/parser/gram.y` |
| Analyzer | `src/backend/parser/analyze.c` |
| Rewriter | `src/backend/rewrite/rewriteHandler.c` |
| Planner Entry | `src/backend/optimizer/plan/planner.c` |
| Cost Estimation | `src/backend/optimizer/path/costsize.c` |
| Join Planning | `src/backend/optimizer/path/joinpath.c` |
| Executor Main | `src/backend/executor/execMain.c` |

---

## Part 15: Planner Deep Dive (Structures, Estimates, and Extended Statistics)

Everything earlier in this chapter helps you **read plans** and understand the major planner decisions. This section goes one level deeper: it explains the internal “building blocks” the optimizer uses, and shows a practical expert tool for stabilizing estimates when columns are correlated.

### The Planner’s High-level Phases (Mental Model)

At a high level, planning looks like:

- **Rewrite / preprocess**: expand views, apply rules, normalize query forms
- **Path generation**: enumerate many possible execution strategies (“paths”) for each relation and join
- **Plan generation**: pick the cheapest paths and produce an executable plan tree

The planner is not trying to “be correct” in the human sense—it’s trying to choose a plan that is **cheap according to its estimates**.

### Core Optimizer Data Structures (What You’ll See in Source)

When you read optimizer code, these names come up constantly:

| Concept | What it represents (roughly) |
|--------|-------------------------------|
| `RelOptInfo` | One relation being planned (base table, join relation, subquery) and all known facts about it |
| `Path` | One possible way to produce rows for a relation (Seq Scan, Index Scan, Hash Join, etc.) |
| `RestrictInfo` | A WHERE/JOIN condition with planner metadata (selectivity, security level, moved quals, etc.) |
| `PathKey` | Ordering properties a path provides or requires (used for ORDER BY and Merge Join planning) |

You don’t need to memorize every field—just recognize these as the planner’s “working vocabulary.”

### The Independence Assumption (Why Correlations Break Estimates)

By default, PostgreSQL often assumes predicates on different columns are independent.
That fails when columns are correlated (very common in real data).

When estimates are wrong by large factors, you’ll often see:
- a join type that’s great for small inputs (Nested Loop) chosen for huge inputs
- a plan that filters late instead of early
- unstable plan choices as data shifts

### Extended Statistics (A Practical Expert Tool)

Extended statistics let PostgreSQL learn relationships across columns (dependencies, ndistinct, multi-column MCVs) so estimates for combined predicates are more stable.

We’ll demonstrate this using `plan_demo`, because it exists in this chapter and has controlled data.

**🔬 Try It:** Compare estimates before/after extended statistics

```sql
-- A combined predicate on two columns (category_id + status)
EXPLAIN
SELECT *
FROM plan_demo
WHERE category_id = 1 AND status = 1;

-- Create extended stats to teach the planner about the relationship
CREATE STATISTICS IF NOT EXISTS st_plan_demo_cat_status (dependencies)
ON category_id, status
FROM plan_demo;

ANALYZE plan_demo;

-- Re-check the estimate after ANALYZE
EXPLAIN
SELECT *
FROM plan_demo
WHERE category_id = 1 AND status = 1;
```

> If your estimates don’t visibly change, try different value pairs (e.g., `category_id = 2 AND status = 3`) or inspect `pg_stats` and confirm the table was analyzed.

### Source Code Connection (Where to Read)

If you want to follow the planner’s logic in the source tree, start here:
- Entry point: `src/backend/optimizer/plan/planner.c`
- Path generation: `src/backend/optimizer/path/`
- Join planning: `src/backend/optimizer/path/joinpath.c`, `src/backend/optimizer/path/allpaths.c`
- Clause selectivity: `src/backend/optimizer/path/clausesel.c`
- Statistics/selectivity functions: `src/backend/utils/adt/selfuncs.c`
- Core structs: `src/include/nodes/pathnodes.h`

---

## Summary

In this chapter, you learned:

1. **Query Lifecycle**: Parse → Analyze → Rewrite → Plan → Execute
2. **Parser**: Validates SQL syntax and builds parse tree
3. **Analyzer**: Resolves names, checks types, adds implicit casts
4. **Rewriter**: Expands views and applies rules
5. **Planner**: Cost-based optimizer that chooses the best execution plan
6. **EXPLAIN**: How to read and interpret query plans
7. **Statistics**: How the planner estimates selectivity and costs
8. **Scan Methods**: Sequential, Index, Index Only, Bitmap, TID
9. **Join Algorithms**: Nested Loop, Hash Join, Merge Join
10. **Aggregation**: HashAggregate vs GroupAggregate
11. **Parallel Query**: Multi-worker execution for large tables
12. **Optimization**: Finding and fixing slow queries

---

## Quick Reference

**🔬 Try It:**
```sql
-- EXPLAIN variations
EXPLAIN query;                           -- Basic plan
EXPLAIN ANALYZE query;                   -- Actual execution times
EXPLAIN (ANALYZE, BUFFERS) query;       -- Buffer/cache statistics
EXPLAIN (ANALYZE, VERBOSE) query;        -- Output columns, extra details
EXPLAIN (FORMAT JSON) query;             -- JSON output

-- Planner controls (use sparingly, mainly for debugging)
SET enable_seqscan = off;                -- Disable sequential scan
SET enable_indexscan = off;              -- Disable index scan
SET enable_bitmapscan = off;             -- Disable bitmap scan
SET enable_hashjoin = off;               -- Disable hash join
SET enable_mergejoin = off;              -- Disable merge join
SET enable_nestloop = off;               -- Disable nested loop
SET enable_hashagg = off;                -- Disable hash aggregate
RESET ALL;                               -- Reset all to defaults

-- Statistics
ANALYZE tablename;                       -- Update statistics
ALTER TABLE t ALTER COLUMN c SET STATISTICS 500;  -- Increase detail
```

---

## Cleanup (Optional)

The demo tables are only used in this chapter. Drop them when done:

**🔬 Try It:**
```sql
DROP TABLE IF EXISTS plan_demo_child;
DROP TABLE IF EXISTS plan_demo;
```

To recreate them, run the setup SQL at the beginning of this chapter.

---

**Previous Chapter:** [← Storage Engine](05-storage-engine.md)

**Next Chapter:** [Indexing →](07-indexing.md)
