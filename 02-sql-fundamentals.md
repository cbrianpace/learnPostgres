# Chapter 2: SQL Fundamentals

## Learning Objectives

By the end of this chapter, you will be able to:
- Write complex SELECT queries with joins, subqueries, and aggregations
- Use window functions for advanced analytics
- Work with Common Table Expressions (CTEs)
- Understand and use set operations
- Write efficient filtering and sorting queries

## Setting Up Our Learning Database

We'll continue using the `learning` database we created in Chapter 1. In Chapter 1, we created basic `departments` and `employees` tables. Now we'll expand these tables with additional columns and add new tables for our SQL exercises.

**🔬 Try It:** Connect to the Database

```sql
-- Connect to the learning database we created in Chapter 1
\c learning
```

---

## Chapter Setup: Evolving the Schema

In Chapter 1, we created basic `employees` and `departments` tables. As our company grows, we need to track more information. This is a common real-world scenario—schemas evolve over time.

**🔬 Try It:** Modify Existing Tables with ALTER TABLE

```sql
-- Add a budget column to departments
ALTER TABLE departments ADD COLUMN budget NUMERIC(12,2);

-- Add a manager_id column to employees (self-referencing for org hierarchy)
ALTER TABLE employees ADD COLUMN manager_id INTEGER REFERENCES employees(emp_id);

-- Verify the changes
\d departments
\d employees
```

> **Why ALTER TABLE?** In production, you rarely drop tables with existing data. `ALTER TABLE` lets you evolve your schema while preserving data. Common operations include `ADD COLUMN`, `DROP COLUMN`, `ALTER COLUMN`, and `ADD CONSTRAINT`.

**🔬 Try It:** Create New Tables for Project Tracking

```sql
CREATE TABLE projects (
    project_id SERIAL PRIMARY KEY,
    project_name VARCHAR(200) NOT NULL,
    start_date DATE,
    end_date DATE,
    budget NUMERIC(12,2),
    dept_id INTEGER REFERENCES departments(dept_id)
);

CREATE TABLE project_assignments (
    assignment_id SERIAL PRIMARY KEY,
    emp_id INTEGER REFERENCES employees(emp_id),
    project_id INTEGER REFERENCES projects(project_id),
    role VARCHAR(50),
    hours_allocated INTEGER,
    UNIQUE(emp_id, project_id)
);
```

**🔬 Try It:** Populate with Sample Data

For this chapter's examples, we need a richer dataset. Let's clear the sample data from Chapter 1 and insert a complete set:

**🔬 Try It:**
```sql
-- Clear existing sample data (keep the table structure!)
TRUNCATE employees, departments RESTART IDENTITY CASCADE;

-- Insert departments with budgets
INSERT INTO departments (dept_name, location, budget) VALUES
    ('Engineering', 'Building A', 1000000),
    ('Sales', 'Building B', 500000),
    ('Marketing', 'Building B', 300000),
    ('HR', 'Building C', 200000),
    ('Research', 'Building A', 800000);

-- Insert employees with manager hierarchy
INSERT INTO employees (first_name, last_name, email, hire_date, salary, dept_id, manager_id) VALUES
    ('John', 'Smith', 'john.smith@company.com', '2020-01-15', 85000, 1, NULL),
    ('Sarah', 'Johnson', 'sarah.j@company.com', '2019-03-22', 92000, 1, 1),
    ('Michael', 'Williams', 'michael.w@company.com', '2021-06-01', 78000, 1, 1),
    ('Emily', 'Brown', 'emily.b@company.com', '2018-11-10', 95000, 2, NULL),
    ('David', 'Jones', 'david.j@company.com', '2020-08-15', 72000, 2, 4),
    ('Lisa', 'Davis', 'lisa.d@company.com', '2022-02-28', 68000, 3, NULL),
    ('Robert', 'Miller', 'robert.m@company.com', '2019-07-20', 88000, 4, NULL),
    ('Jennifer', 'Wilson', 'jennifer.w@company.com', '2021-01-10', 75000, 5, NULL),
    ('James', 'Taylor', 'james.t@company.com', '2020-04-05', 82000, 1, 2),
    ('Amanda', 'Anderson', 'amanda.a@company.com', '2022-09-01', 65000, 2, 4);

INSERT INTO projects (project_name, start_date, end_date, budget, dept_id) VALUES
    ('Website Redesign', '2024-01-01', '2024-06-30', 150000, 1),
    ('Mobile App v2', '2024-03-01', '2024-12-31', 300000, 1),
    ('Sales CRM Integration', '2024-02-01', '2024-08-31', 100000, 2),
    ('Brand Refresh', '2024-04-01', '2024-07-31', 75000, 3),
    ('AI Research Project', '2024-01-15', '2025-01-15', 500000, 5);

INSERT INTO project_assignments (emp_id, project_id, role, hours_allocated) VALUES
    (1, 1, 'Project Lead', 200),
    (2, 1, 'Developer', 400),
    (3, 1, 'Developer', 400),
    (2, 2, 'Tech Lead', 600),
    (3, 2, 'Developer', 500),
    (9, 2, 'Developer', 400),
    (4, 3, 'Project Lead', 300),
    (5, 3, 'Sales Rep', 200),
    (6, 4, 'Designer', 250),
    (8, 5, 'Researcher', 800);
```

Why we reset data here: the rest of this chapter builds on predictable sample data so that joins, aggregates, and subqueries produce stable results. In real systems you wouldn’t usually TRUNCATE production tables—you’d run these exercises on a dev database or a sandbox.

---

## Part 1: Advanced SELECT Queries

### Column Expressions and Aliases

**📖 Example:** You can create computed columns and give them readable names:

> ```sql
> SELECT 
>     first_name || ' ' || last_name AS full_name,
>     salary / 12 AS monthly_salary
> FROM employees;
> ```

**🔬 Try It:** Calculated Columns and CASE Expressions

```sql
-- Calculate derived columns
SELECT 
    first_name || ' ' || last_name AS full_name,
    salary AS annual_salary,
    salary / 12 AS monthly_salary,
    salary * 1.10 AS salary_with_10pct_raise
FROM employees;

-- Using CASE expressions to categorize employees
SELECT 
    first_name,
    last_name,
    salary,
    CASE 
        WHEN salary >= 90000 THEN 'Senior'
        WHEN salary >= 75000 THEN 'Mid-level'
        ELSE 'Junior'
    END AS level
FROM employees;
```

### Filtering with WHERE

**🔬 Try It:** WHERE Clause Variations

```sql
-- Multiple conditions with AND/OR
SELECT * FROM employees
WHERE dept_id = 1 AND salary > 80000;

SELECT * FROM employees
WHERE dept_id = 1 OR dept_id = 2;

-- IN operator (cleaner than multiple ORs)
SELECT * FROM employees
WHERE dept_id IN (1, 2, 5);

-- BETWEEN for ranges (inclusive)
SELECT * FROM employees
WHERE salary BETWEEN 70000 AND 90000;

-- LIKE for pattern matching
SELECT * FROM employees
WHERE email LIKE '%@company.com';

SELECT * FROM employees
WHERE last_name LIKE 'J%';  -- Starts with J

-- NULL handling
SELECT * FROM employees
WHERE manager_id IS NULL;  -- Find top-level managers

SELECT * FROM employees
WHERE manager_id IS NOT NULL;
```

**🔬 Try It:** Practice WHERE Clauses

Write and execute queries to find:
1. Employees hired in 2020
2. Employees with salary between 70000 and 85000 in Engineering
3. Employees whose email contains 'j'

**🔬 Try It:**
```sql
-- 1. Hired in 2020
SELECT * FROM employees
WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01';
-- Or using EXTRACT:
SELECT * FROM employees
WHERE EXTRACT(YEAR FROM hire_date) = 2020;

-- 2. Salary range in Engineering
SELECT e.* FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary BETWEEN 70000 AND 85000
  AND d.dept_name = 'Engineering';

-- 3. Email contains 'j' (case-insensitive)
SELECT * FROM employees
WHERE email ILIKE '%j%';
```

---

## Part 2: Joins

Joins combine rows from multiple tables. PostgreSQL supports all standard join types.

**Join Syntax Shortcuts:**

| Full Syntax | Shorthand | Notes |
|-------------|-----------|-------|
| `INNER JOIN` | `JOIN` | The `INNER` keyword is optional |
| `LEFT OUTER JOIN` | `LEFT JOIN` | The `OUTER` keyword is optional |
| `RIGHT OUTER JOIN` | `RIGHT JOIN` | The `OUTER` keyword is optional |
| `FULL OUTER JOIN` | `FULL JOIN` | The `OUTER` keyword is optional |

Most developers use the shorthand forms. You'll see both in production code.

**Understanding "Left" and "Right" Tables:**

In a JOIN, "left" and "right" refer to the **order tables appear in your query** relative to the JOIN clause:

> ```sql
> FROM left_table
> JOIN right_table ON ...
> ```

> ```sql
> SELECT ...
> FROM employees e          ← LEFT table (appears first, before JOIN)
> LEFT JOIN departments d   ← RIGHT table (appears after JOIN keyword)
> ON e.dept_id = d.dept_id;
> ```

Think of it like reading order:
- **Left table** = the table mentioned BEFORE the JOIN keyword
- **Right table** = the table mentioned AFTER the JOIN keyword

Visual representation:
> ```text
> FROM  A  LEFT JOIN  B  ON ...
>       ↑            ↑
>     LEFT         RIGHT
>    table         table
> ```

This matters for OUTER joins:
- `LEFT JOIN` keeps all rows from the **left** (first) table
- `RIGHT JOIN` keeps all rows from the **right** (second) table
- `FULL JOIN` keeps all rows from **both** tables

### INNER JOIN

Returns only rows that match in both tables.

You’ll use INNER JOIN when you only care about rows that have a valid relationship on both sides (for example, “employees that belong to a department”).

> **Alternate syntax:** `JOIN` (without `INNER`) means the same thing. Both are equivalent:
> - `FROM employees e INNER JOIN departments d ON ...`
> - `FROM employees e JOIN departments d ON ...`

**🔬 Try It:** Joins

```sql
-- Get employees with their department names
-- These two queries are identical:
SELECT 
    e.first_name,
    e.last_name,
    d.dept_name,
    d.location
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- Same query using shorthand (more common in practice)
SELECT 
    e.first_name,
    e.last_name,
    d.dept_name,
    d.location
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

### LEFT (OUTER) JOIN

Returns **all rows from the left table** (the one before JOIN), plus matching rows from the right table. Non-matching rows get NULL for the right table's columns.

In the HR story, LEFT JOIN is a great fit for “show everything even if optional related data is missing,” like employees who don’t have project assignments yet.

> **Alternate syntax:** `LEFT JOIN` and `LEFT OUTER JOIN` are identical. The `OUTER` keyword is optional.

**🔬 Try It:** LEFT JOIN

```sql
-- Get ALL employees, even those without project assignments
-- employees = LEFT table (all rows kept)
-- project_assignments = RIGHT table (only matching rows)
SELECT 
    e.first_name,
    e.last_name,
    p.project_name,      -- NULL if employee has no assignments
    pa.role              -- NULL if employee has no assignments
FROM employees e                                              -- ← LEFT table
LEFT JOIN project_assignments pa ON e.emp_id = pa.emp_id      -- ← RIGHT table
LEFT JOIN projects p ON pa.project_id = p.project_id;
```

Visual: LEFT JOIN keeps everything from the left side
```
employees (LEFT)     project_assignments (RIGHT)
┌──────────────┐     ┌──────────────┐
│ Alice    ────┼─────┼─→ Project A  │  ✓ Match
│ Bob      ────┼─────┼─→ Project B  │  ✓ Match  
│ Amanda       │     │              │  ✓ Kept (with NULLs)
└──────────────┘     └──────────────┘
   ALL KEPT           Only matches
```

### RIGHT (OUTER) JOIN

Returns **all rows from the right table** (the one after JOIN), plus matching rows from the left table.

RIGHT JOIN is less common in day-to-day work because you can usually rewrite it as a LEFT JOIN by swapping table order. We include it so you can read existing SQL and understand the “left vs right table” rule.

> **Alternate syntax:** `RIGHT JOIN` and `RIGHT OUTER JOIN` are identical.

**🔬 Try It:** Right Join

```sql
-- Get ALL departments, even those with no employees
-- employees = LEFT table (only matching rows)
-- departments = RIGHT table (all rows kept)
SELECT 
    d.dept_name,
    d.location,
    e.first_name,       -- NULL if department has no employees
    e.last_name         -- NULL if department has no employees
FROM employees e                                    -- LEFT table
RIGHT JOIN departments d ON e.dept_id = d.dept_id;  -- RIGHT table (all kept)
```

Visual: RIGHT JOIN keeps everything from the right side

*(Illustrative example - imagine a "Legal" department with no employees yet)*
```
employees (LEFT)     departments (RIGHT)
┌──────────────┐     ┌──────────────┐
│ John     ────┼─────┼─→ Engineering│  ✓ Match
│ Lisa     ────┼─────┼─→ Marketing  │  ✓ Match  
│              │     │   Legal      │  ✓ Kept (NULLs for employee)
└──────────────┘     └──────────────┘
  Only matches          ALL KEPT
```

In our sample data, all departments have employees. To see a RIGHT JOIN with NULLs:

**🔬 Try It:** RIGHT JOIN with Empty Department

```sql
-- Add a department with no employees
INSERT INTO departments (dept_name, location, budget) 
VALUES ('Legal', 'Building D', 150000);

-- Now this RIGHT JOIN will show Legal with NULL employee names
SELECT d.dept_name, e.first_name, e.last_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

**Chained JOINs:** When you have multiple JOINs, each one operates on the *result* of the previous join:

> ```sql
> FROM A
> JOIN B ON ...      -- A is left, B is right
> JOIN C ON ...      -- (A+B result) is left, C is right
> JOIN D ON ...      -- (A+B+C result) is left, D is right
> ```

**Tip:** RIGHT JOINs can always be rewritten as LEFT JOINs by swapping table order:

> ```sql
> -- These two queries produce the same result:
> 
> -- Using RIGHT JOIN
> FROM employees e RIGHT JOIN departments d ON ...
> 
> -- Equivalent LEFT JOIN (swap the table order)
> FROM departments d LEFT JOIN employees e ON ...
> ```

Many developers prefer using LEFT JOINs exclusively for consistency and readability.

### FULL (OUTER) JOIN

Returns **all rows from both tables**, matching where possible. Unmatched rows from either side get NULLs.

FULL JOIN is useful for reconciliation-style questions—finding missing relationships in either direction (e.g., departments with no employees AND employees with no department).

> **Alternate syntax:** `FULL JOIN` and `FULL OUTER JOIN` are identical.

**🔬 Try It:** FULL JOIN

```sql
-- See ALL departments and ALL employees, even unmatched ones
SELECT 
    d.dept_name,        -- NULL if employee has no department
    e.first_name,       -- NULL if department has no employees
    e.last_name
FROM departments d                                   -- LEFT table
FULL JOIN employees e ON d.dept_id = e.dept_id;      -- RIGHT table
```

Visual: FULL JOIN keeps everything from both sides

*(Illustrative - shows what happens with unmatched rows on BOTH sides)*
```
departments (LEFT)   employees (RIGHT)
┌──────────────┐     ┌──────────────┐
│ Engineering ─┼─────┼─→ John       │  ✓ Match
│ Marketing   ─┼─────┼─→ Lisa       │  ✓ Match  
│ Legal        │     │              │  ✓ Kept (NULL employee)
│              │     │   Contractor │  ✓ Kept (NULL department)
└──────────────┘     └──────────────┘
   ALL KEPT            ALL KEPT
```

In our sample data, all employees have departments and all departments have employees. To see FULL JOIN with NULLs on both sides:

**🔬 Try It:** FULL JOIN with Unmatched Rows

```sql
-- Add an employee with no department (NULL dept_id)
INSERT INTO employees (first_name, last_name, email, hire_date, salary, dept_id, manager_id)
VALUES ('Alex', 'Contractor', 'alex.c@company.com', '2024-01-01', 50000, NULL, NULL);

-- Now FULL JOIN shows unmatched rows from BOTH sides
SELECT d.dept_name, e.first_name, e.last_name
FROM departments d
FULL JOIN employees e ON d.dept_id = e.dept_id
ORDER BY d.dept_name NULLS LAST, e.first_name;
-- Legal will have NULL for employee columns
-- Alex will have NULL for department columns
```

### CROSS JOIN

Cartesian product of both tables (every row from table A paired with every row from table B).

**📖 Example:** CROSS JOIN syntax (be careful with large tables!):

> ```sql
> -- Every possible combination
> SELECT d.dept_name, p.project_name
> FROM departments d
> CROSS JOIN projects p;
> ```
> **Warning:** 1000 rows × 1000 rows = 1,000,000 rows!


### SELF JOIN

Join a table to itself to find hierarchical relationships:

**🔬 Try It:**
```sql
-- Get employees with their manager names
SELECT 
    e.first_name || ' ' || e.last_name AS employee,
    m.first_name || ' ' || m.last_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

### Legacy Join Syntax (SQL-89)

You may encounter older code using comma-separated tables with conditions in the WHERE clause:

**📖 Example:** Comparing modern and legacy join syntax:

> ```sql
> -- Modern syntax (SQL-92, recommended)
> SELECT e.first_name, d.dept_name
> FROM employees e
> INNER JOIN departments d ON e.dept_id = d.dept_id
> WHERE e.salary > 50000;
> 
> -- Legacy syntax (SQL-89, still works but not recommended)
> SELECT e.first_name, d.dept_name
> FROM employees e, departments d
> WHERE e.dept_id = d.dept_id
>   AND e.salary > 50000;
> ```

**Why prefer modern JOIN syntax:**
- Separates join conditions (`ON`) from filter conditions (`WHERE`)
- Makes LEFT/RIGHT/FULL OUTER joins explicit and readable
- Harder to accidentally create a CROSS JOIN by forgetting a condition
- Easier to understand query structure at a glance

---

## Part 3: Aggregation and GROUP BY

Aggregation functions let you compute a single result from a set of rows. Instead of seeing every individual row, you can answer questions like "How many?", "What's the total?", or "What's the average?"

**Common use cases:**
- **Reporting**: Calculate totals, averages, counts for dashboards and reports
- **Analytics**: Find trends, outliers, distributions in your data
- **Data validation**: Check for duplicates, missing values, data quality issues
- **Business metrics**: Revenue totals, customer counts, average order values

Aggregations are often combined with `GROUP BY` to calculate results for each category (e.g., sales per region, employees per department).

**🔬 Try It:** Basic Aggregations

```sql
-- Count, Sum, Average, Min, Max
SELECT 
    COUNT(*) AS total_employees,
    SUM(salary) AS total_salaries,
    AVG(salary) AS avg_salary,
    MIN(salary) AS lowest_salary,
    MAX(salary) AS highest_salary
FROM employees;

-- Count distinct values
SELECT COUNT(DISTINCT dept_id) AS num_departments
FROM employees;
```

### GROUP BY

`GROUP BY` splits your data into groups based on one or more columns, then applies aggregate functions to each group separately. Without `GROUP BY`, aggregates operate on the entire result set as one group.

**Key rule:** Every column in your SELECT must either:
1. Be in the `GROUP BY` clause, OR
2. Be inside an aggregate function (COUNT, SUM, AVG, etc.)

```
Without GROUP BY:              With GROUP BY dept_id:
┌─────────────────────┐       ┌─────────────────────┐
│ All employees       │       │ Engineering │ 3 emp │
│ as ONE group        │  →    │ Sales       │ 3 emp │
│ COUNT(*) = 10       │       │ Marketing   │ 1 emp │
└─────────────────────┘       │ HR          │ 1 emp │
                              │ Research    │ 1 emp │
                              └─────────────────────┘
```

**🔬 Try It:** GROUP BY

```sql
-- Aggregate by department
SELECT 
    d.dept_name,                           -- In GROUP BY ✓
    COUNT(e.emp_id) AS num_employees,      -- Aggregate function ✓
    AVG(e.salary)::NUMERIC(10,2) AS avg_salary,  -- Aggregate function ✓
    SUM(e.salary) AS total_salary          -- Aggregate function ✓
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name            -- Columns to group by
ORDER BY num_employees DESC;
```

**📖 Example:** Common mistake—forgetting to include a non-aggregated column in GROUP BY:

> ```sql
> -- ERROR: column "e.first_name" must appear in GROUP BY clause
> SELECT d.dept_name, e.first_name, COUNT(*)
> FROM departments d JOIN employees e ON d.dept_id = e.dept_id
> GROUP BY d.dept_name;
> -- Fix: Either add e.first_name to GROUP BY, or remove it from SELECT
> ```

HAVING filters groups (after aggregation), while WHERE filters rows (before aggregation):

**🔬 Try It:**
```sql
-- Departments with more than 2 employees
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS num_employees
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
HAVING COUNT(e.emp_id) > 2;

-- Departments with average salary over 75000
SELECT 
    d.dept_name,
    AVG(e.salary)::NUMERIC(10,2) AS avg_salary
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
HAVING AVG(e.salary) > 75000;
```

### GROUPING SETS, ROLLUP, and CUBE

These advanced GROUP BY features let you calculate **multiple levels of aggregation in a single query** - perfect for reports that need subtotals and grand totals.

**Without these features**, you'd need multiple queries with UNION ALL:

> ```sql
> -- The hard way: 3 separate queries
> SELECT dept_name, hire_year, COUNT(*) FROM ... GROUP BY dept_name, hire_year
> UNION ALL
> SELECT dept_name, NULL, COUNT(*) FROM ... GROUP BY dept_name  -- subtotals
> UNION ALL
> SELECT NULL, NULL, COUNT(*) FROM ...;  -- grand total
> ```

**With GROUPING SETS**, you define exactly which groupings you want:

| Feature | What it does | Use case |
|---------|--------------|----------|
| `GROUPING SETS` | Specify exact grouping combinations | Custom report layouts |
| `ROLLUP` | Hierarchical subtotals (left to right) | Drill-down reports (year → quarter → month) |
| `CUBE` | All possible combinations | Cross-tabulation, pivot tables |

**🔬 Try It:** GROUPING SETS

```sql
-- Get department+year breakdown, department totals, AND grand total
SELECT 
    d.dept_name,
    EXTRACT(YEAR FROM e.hire_date) AS hire_year,
    COUNT(*) AS num_employees,
    AVG(e.salary)::NUMERIC(10,2) AS avg_salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY GROUPING SETS (
    (d.dept_name, EXTRACT(YEAR FROM e.hire_date)),  -- Detail rows
    (d.dept_name),                                   -- Subtotal per dept (hire_year = NULL)
    ()                                               -- Grand total (both = NULL)
)
ORDER BY d.dept_name NULLS LAST, hire_year;
```

ROLLUP is shorthand for hierarchical subtotals:

**🔬 Try It:**
```sql
-- ROLLUP(a, b, c) is equivalent to:
-- GROUPING SETS ((a, b, c), (a, b), (a), ())

-- Get counts per department, plus a grand total row
SELECT 
    d.dept_name,
    COUNT(*) AS num_employees
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY ROLLUP(d.dept_name);
-- Results: Engineering: 4, Sales: 3, ... , NULL: 10 (grand total)
```

CUBE generates **all possible subtotal combinations** across the listed grouping columns.

Think of it as: “Give me the full breakdown, *plus* every subtotal I could reasonably want for a report.”

For \(n\) grouping columns, `CUBE` produces \(2^n\) grouping sets. For two columns `(dept, year)`, that means:
- Detail rows: `(dept, year)`
- Subtotal by department: `(dept)`
- Subtotal by year: `(year)`
- Grand total: `()`

In the output, PostgreSQL uses `NULL` in the grouped columns to represent subtotal/grand-total rows. (In later chapters, you’ll see `GROUPING()` used to distinguish “real NULL” from “subtotal NULL”.)

**🔬 Try It:**
```sql
-- CUBE(a, b) is equivalent to:
-- GROUPING SETS ((a, b), (a), (b), ())

SELECT 
    d.dept_name,
    EXTRACT(YEAR FROM e.hire_date) AS hire_year,
    COUNT(*) AS num_employees
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY CUBE(d.dept_name, EXTRACT(YEAR FROM e.hire_date));
-- Results include: each dept+year, totals per dept, totals per year, grand total
```

## Part 4: Subqueries

### Scalar Subqueries

A **scalar** value is a single, atomic value—one row and one column. Think of it as a single cell in a spreadsheet, not a range. The term comes from linear algebra where a scalar is a single number (as opposed to a vector or matrix).

A scalar subquery returns exactly one value and can be used anywhere a single value is expected:

**🔬 Try It:**

```sql
-- Employees earning above average
SELECT first_name, last_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Each employee's salary compared to department average
SELECT 
    e.first_name,
    e.last_name,
    e.salary,
    (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id)::NUMERIC(10,2) AS dept_avg
FROM employees e;
```

### Table Subqueries

Returns multiple rows:

**🔬 Try It:**

```sql
-- Employees in departments with budget over 500000
SELECT first_name, last_name, dept_id
FROM employees
WHERE dept_id IN (
    SELECT dept_id FROM departments WHERE budget > 500000
);

-- Using EXISTS
SELECT first_name, last_name
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM project_assignments pa 
    WHERE pa.emp_id = e.emp_id
);
```

**IN vs EXISTS: When to Use Each**

| Aspect | `IN` | `EXISTS` |
|--------|------|----------|
| **How it works** | Builds a list, then checks membership | Checks for existence row-by-row |
| **NULL handling** | `IN` with NULLs can give unexpected results | Handles NULLs predictably |
| **Best for** | Small subquery result sets | Large outer tables, correlated checks |
| **Short-circuits** | No - evaluates entire subquery | Yes - stops at first match |

**Performance considerations:**

- **Use `IN`** when the subquery returns a small, distinct list of values. PostgreSQL can optimize this into a hash or merge join.

- **Use `EXISTS`** when:
  - You only care whether *any* matching row exists (not the actual values)
  - The subquery is correlated (references the outer table)
  - The subquery might return many duplicate values
  - The subquery result set is large but matches are found early

> ```sql
> -- IN: Good when subquery returns few distinct values
> WHERE dept_id IN (SELECT dept_id FROM departments WHERE location = 'Building A')
> 
> -- EXISTS: Good for "has any" checks, especially with indexes on the correlation
> WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.total > 1000)
> ```

**NULL gotcha with IN:**

> ```sql
> -- If subquery returns NULL values, NOT IN may return no rows!
> SELECT * FROM employees WHERE dept_id NOT IN (SELECT dept_id FROM old_depts);
> -- If old_depts has a NULL dept_id, this returns NOTHING (not what you expect)
> 
> -- NOT EXISTS handles NULLs correctly
> SELECT * FROM employees e 
> WHERE NOT EXISTS (SELECT 1 FROM old_depts o WHERE o.dept_id = e.dept_id);
> ```

**In practice:** Modern PostgreSQL's query planner is smart and often generates similar execution plans for equivalent IN and EXISTS queries. Use EXPLAIN ANALYZE to verify performance in your specific case.

### Correlated Subqueries

A **correlated subquery** references columns from the outer query, creating a dependency between them. Unlike a regular subquery (which runs once), a correlated subquery is conceptually re-evaluated for *each row* of the outer query.

**Regular vs. Correlated:**

> ```sql
> -- REGULAR subquery: runs ONCE, returns one value used for all rows
> SELECT * FROM employees 
> WHERE salary > (SELECT AVG(salary) FROM employees);
> --              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ doesn't reference outer query
> 
> -- CORRELATED subquery: conceptually runs ONCE PER ROW of outer query
> SELECT * FROM employees e1
> WHERE salary > (SELECT AVG(salary) FROM employees e2 WHERE e2.dept_id = e1.dept_id);
> --                                                         ^^^^^^^^^ references outer query!
> ```

**How it works:**

For each row in the outer query, PostgreSQL:
1. Takes the current row's values (e.g., `e1.dept_id = 1`)
2. Executes the inner subquery using those values
3. Compares the result to determine if the row qualifies

**🔬 Try It:**

```sql
-- Employees who earn more than their department average
SELECT e1.first_name, e1.last_name, e1.salary, e1.dept_id
FROM employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e1.dept_id  -- This links inner to outer
);
```

**Performance consideration:** Correlated subqueries can be slow on large tables because they appear to run O(n) times. However, PostgreSQL's optimizer often rewrites them as JOINs internally. Always check with `EXPLAIN ANALYZE` if performance matters.

### Subqueries in FROM (Derived Tables)

**🔬 Try It:**

```sql
-- Top earner in each department
SELECT dept_stats.dept_name, dept_stats.top_salary
FROM (
    SELECT 
        d.dept_name,
        MAX(e.salary) AS top_salary
    FROM departments d
    JOIN employees e ON d.dept_id = e.dept_id
    GROUP BY d.dept_id, d.dept_name
) AS dept_stats
WHERE dept_stats.top_salary > 80000;
```

## Part 5: Common Table Expressions (CTEs)

A **Common Table Expression (CTE)** is a temporary named result set that exists only for the duration of a single query. Think of it as creating a temporary view that you define inline with your query.

**Why use CTEs?**

| Benefit | Description |
|---------|-------------|
| **Readability** | Break complex queries into logical, named steps |
| **Reusability** | Reference the same subquery multiple times without repeating it |
| **Maintainability** | Easier to understand, debug, and modify |
| **Recursion** | Enable recursive queries (impossible with regular subqueries) |

### Basic CTE Syntax

The `WITH` clause defines one or more CTEs before your main query:

> ```sql
> WITH cte_name AS (
>     -- This query runs first and creates a temporary result set
>     SELECT ...
> )
> -- Main query can reference cte_name like a table
> SELECT * FROM cte_name WHERE ...;
> ```

**Example:** Calculate department statistics, then join with department names:

**🔬 Try It:**
```sql
-- Basic CTE
WITH dept_stats AS (
    -- Step 1: Calculate stats (this runs first)
    SELECT 
        dept_id,
        COUNT(*) AS emp_count,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
-- Step 2: Join with departments (uses the CTE result)
SELECT 
    d.dept_name,
    ds.emp_count,
    ds.avg_salary::NUMERIC(10,2)
FROM departments d
JOIN dept_stats ds ON d.dept_id = ds.dept_id;
```

### Multiple CTEs

You can define multiple CTEs separated by commas. Each CTE can reference CTEs defined before it, allowing you to build up complex logic step by step:

**🔬 Try It:**
```sql
WITH 
-- First CTE: Calculate salary totals by department
dept_totals AS (
    SELECT dept_id, SUM(salary) AS total_salary
    FROM employees
    GROUP BY dept_id
),
-- Second CTE: Calculate project budgets by department
project_costs AS (
    SELECT dept_id, SUM(budget) AS total_budget
    FROM projects
    GROUP BY dept_id
)
-- Main query: Combine both CTEs with department info
SELECT 
    d.dept_name,
    COALESCE(dt.total_salary, 0) AS salary_cost,
    COALESCE(pc.total_budget, 0) AS project_budget
FROM departments d
LEFT JOIN dept_totals dt ON d.dept_id = dt.dept_id
LEFT JOIN project_costs pc ON d.dept_id = pc.dept_id;
```

**Tip:** Reading a multi-CTE query from top to bottom tells a story: "First calculate X, then calculate Y, finally combine them."

### Recursive CTEs

A **recursive CTE** references itself, allowing you to traverse hierarchical or graph-like data structures. This is perfect for:
- Organizational charts (employees → managers)
- Category trees (subcategories → parent categories)  
- Bill of materials (components → assemblies)
- Network paths (node → connected nodes)

**Structure of a recursive CTE:**

> ```sql
> WITH RECURSIVE cte_name AS (
>     -- 1. BASE CASE: Starting point (non-recursive)
>     SELECT ... WHERE <starting condition>
>     
>     UNION ALL
>     
>     -- 2. RECURSIVE CASE: How to find the next level
>     SELECT ... 
>     FROM table 
>     JOIN cte_name ON <relationship to previous level>
> )
> SELECT * FROM cte_name;
> ```

**How it works:**
1. Execute the base case → produces initial rows
2. Execute the recursive case using those rows → produces more rows
3. Repeat step 2 until no new rows are produced
4. Combine all rows as the final result

Build an organizational hierarchy showing reporting chains:

**🔬 Try It:**
```sql
-- Build the organizational hierarchy
WITH RECURSIVE org_hierarchy AS (
    -- Base case: top-level managers
    SELECT 
        emp_id,
        first_name,
        last_name,
        manager_id,
        1 AS level,
        first_name || ' ' || last_name AS path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees who report to someone
    SELECT 
        e.emp_id,
        e.first_name,
        e.last_name,
        e.manager_id,
        h.level + 1,
        h.path || ' -> ' || e.first_name || ' ' || e.last_name
    FROM employees e
    JOIN org_hierarchy h ON e.manager_id = h.emp_id
)
SELECT level, path
FROM org_hierarchy
ORDER BY path;
```

**Sample output:**
```
 level |                     path                      
-------+-----------------------------------------------
     1 | Emily Brown
     2 | Emily Brown -> Amanda Anderson
     2 | Emily Brown -> David Jones
     1 | John Smith
     2 | John Smith -> Sarah Johnson
     3 | John Smith -> Sarah Johnson -> James Taylor
     2 | John Smith -> Michael Williams
```

**⚠️ Infinite recursion warning:** If your data has cycles (A reports to B, B reports to A), the recursive CTE will run forever. PostgreSQL doesn't detect this automatically. Protect yourself by adding a depth limit:

> ```sql
> WITH RECURSIVE org_hierarchy AS (
>     SELECT ..., 1 AS level FROM employees WHERE manager_id IS NULL
>     UNION ALL
>     SELECT ..., h.level + 1 
>     FROM employees e JOIN org_hierarchy h ON ...
>     WHERE h.level < 10  -- Safety limit!
> )
> ```

## Part 6: Window Functions

Window functions perform calculations across a set of rows related to the current row—without collapsing them into a single output row like GROUP BY does. They're essential for analytics, reporting, and any time you need to compare rows to their neighbors or compute running calculations.

**Key concept:** The `OVER()` clause defines the "window" of rows the function operates on:
- `PARTITION BY` divides rows into groups (like GROUP BY, but keeps all rows)
- `ORDER BY` determines the order for ranking or running calculations
- Frame clauses (`ROWS BETWEEN...`) define exactly which rows to include

### ROW_NUMBER, RANK, DENSE_RANK

These functions assign a number to each row based on its position in the ordered result. They differ in how they handle ties (rows with equal values):

| Function | Ties | Example: values 100, 90, 90, 80 |
|----------|------|--------------------------------|
| `ROW_NUMBER()` | Each row gets unique number | 1, 2, 3, 4 |
| `RANK()` | Ties get same rank, next rank skipped | 1, 2, 2, 4 |
| `DENSE_RANK()` | Ties get same rank, no gaps | 1, 2, 2, 3 |

**🔬 Try It:**
```sql
-- Different ranking functions
SELECT 
    first_name,
    last_name,
    salary,
    dept_id,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK() OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- Ranking within partitions
SELECT 
    d.dept_name,
    e.first_name,
    e.last_name,
    e.salary,
    RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS dept_rank
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

### Running Totals and Averages

Running (or cumulative) calculations show a value that accumulates as you move through the rows. For each row, the function considers all rows from the start up to (and including) the current row.

Common uses:
- **Running totals**: Track cumulative sales, expenses, or inventory over time
- **Running averages**: Show how an average changes as more data arrives
- **Year-to-date calculations**: Sum values from January 1 to the current date

The key is `ORDER BY` in the `OVER()` clause—it defines the sequence for accumulation.

**🔬 Try It:**
```sql
-- Running total of salaries by hire date
SELECT 
    hire_date,
    first_name,
    last_name,
    salary,
    SUM(salary) OVER (ORDER BY hire_date) AS running_total,
    AVG(salary) OVER (ORDER BY hire_date) AS running_avg
FROM employees
ORDER BY hire_date;

-- Department running totals
SELECT 
    d.dept_name,
    e.first_name,
    e.salary,
    SUM(e.salary) OVER (
        PARTITION BY e.dept_id 
        ORDER BY e.salary
    ) AS dept_running_total
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

### LAG and LEAD

`LAG()` and `LEAD()` let you access values from other rows relative to the current row—without needing a self-join.

- **`LAG(column, offset, default)`**: Look backward (previous rows)
- **`LEAD(column, offset, default)`**: Look forward (next rows)

The `offset` (default 1) is how many rows to look back/ahead. The `default` is returned when there's no row at that offset (e.g., LAG on the first row).

Common uses:
- Calculate change from previous period (sales growth, price changes)
- Compare current row to previous/next values
- Detect gaps or anomalies in sequences

**🔬 Try It:**
```sql
-- Compare each employee's salary to previous hire
SELECT 
    hire_date,
    first_name,
    salary,
    LAG(salary) OVER (ORDER BY hire_date) AS prev_salary,
    salary - LAG(salary) OVER (ORDER BY hire_date) AS salary_diff
FROM employees
ORDER BY hire_date;

-- Look ahead
SELECT 
    hire_date,
    first_name,
    salary,
    LEAD(salary) OVER (ORDER BY hire_date) AS next_hire_salary
FROM employees
ORDER BY hire_date;
```

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

These functions return a value from a specific position within the window frame:

- **`FIRST_VALUE(column)`**: Returns the value from the first row in the frame
- **`LAST_VALUE(column)`**: Returns the value from the last row in the frame
- **`NTH_VALUE(column, n)`**: Returns the value from the nth row in the frame

Common uses:
- Compare each row to the best/worst in its group
- Find the first or most recent value in a sequence
- Calculate difference from the top performer

**Gotcha with LAST_VALUE:** By default, the frame ends at the current row, so `LAST_VALUE` often just returns the current row's value! Use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` to include all rows in the partition.

**🔬 Try It:**
```sql
-- Compare to highest/lowest in department
SELECT 
    d.dept_name,
    e.first_name,
    e.salary,
    FIRST_VALUE(e.salary) OVER (
        PARTITION BY e.dept_id 
        ORDER BY e.salary DESC
    ) AS highest_in_dept,
    e.salary - FIRST_VALUE(e.salary) OVER (
        PARTITION BY e.dept_id 
        ORDER BY e.salary DESC
    ) AS diff_from_highest
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

### Frame Specifications

Frame specifications let you precisely control which rows are included in each window function calculation. They define the "sliding window" of rows relative to the current row.

**Syntax:** `ROWS BETWEEN start AND end` or `RANGE BETWEEN start AND end`

| Bound | Meaning |
|-------|---------|
| `UNBOUNDED PRECEDING` | First row of partition |
| `n PRECEDING` | n rows before current |
| `CURRENT ROW` | The current row |
| `n FOLLOWING` | n rows after current |
| `UNBOUNDED FOLLOWING` | Last row of partition |

**ROWS vs RANGE:**
- `ROWS`: Physical row count (exact number of rows)
- `RANGE`: Logical value range (rows with equal ORDER BY values treated as one)

Common frame patterns:
- `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — Running total (default)
- `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` — 3-row moving average
- `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` — Entire partition


**🔬 Try It:**
```sql
-- Moving averages with specific windows
SELECT 
    hire_date,
    first_name,
    salary,
    AVG(salary) OVER (
        ORDER BY hire_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    )::NUMERIC(10,2) AS moving_avg_3
FROM employees
ORDER BY hire_date;

-- All rows in partition up to current
SELECT 
    d.dept_name,
    e.first_name,
    e.salary,
    AVG(e.salary) OVER (
        PARTITION BY e.dept_id
        ORDER BY e.salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )::NUMERIC(10,2) AS cumulative_avg
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

## Part 7: Set Operations

Set operations combine the results of two or more SELECT queries. They treat each query result as a mathematical set and perform set operations (union, intersection, difference).

**Requirements for set operations:**
- All queries must return the **same number of columns**
- Corresponding columns must have **compatible data types**
- Column names come from the **first query**

| Operation | Returns | Duplicates |
|-----------|---------|------------|
| `UNION` | All rows from both queries | Removed |
| `UNION ALL` | All rows from both queries | Kept |
| `INTERSECT` | Rows that appear in both queries | Removed |
| `EXCEPT` | Rows in first query but not in second | Removed |

### UNION and UNION ALL

`UNION` combines results from multiple queries into a single result set. By default, it removes duplicate rows (like `SELECT DISTINCT`).

`UNION ALL` keeps all rows, including duplicates. It's faster than `UNION` because it skips the duplicate-elimination step. Use it when you know there are no duplicates or when duplicates are acceptable.

**🔬 Try It:**
```sql
-- Combine employee names with manager names (removes duplicates)
SELECT first_name, last_name FROM employees
UNION
SELECT first_name, last_name FROM employees WHERE manager_id IS NULL;

-- Keep duplicates
SELECT dept_id FROM employees
UNION ALL
SELECT dept_id FROM projects;
```

### INTERSECT

`INTERSECT` returns only rows that appear in **both** query results. Think of it as finding the common elements between two sets.

Use cases:
- Find customers who bought Product A AND Product B
- Find users who visited Page X AND Page Y
- Find items that match multiple criteria from different tables

**🔬 Try It:**
```sql
-- Departments that have both employees AND projects
SELECT dept_id FROM employees
INTERSECT
SELECT dept_id FROM projects;
```

### EXCEPT

`EXCEPT` returns rows from the first query that **don't appear** in the second query. It's like set subtraction: A - B.

**Order matters!** `A EXCEPT B` is different from `B EXCEPT A`.

Use cases:
- Find customers who haven't made a purchase recently
- Find products not in any order
- Find records in one table missing from another (data validation)

Note: `EXCEPT` is similar to `NOT EXISTS` or `NOT IN`, but often more readable. The optimizer may generate similar plans.

**🔬 Try It:**
```sql
-- Departments with employees but no projects
SELECT dept_id FROM employees
EXCEPT
SELECT dept_id FROM projects;
```

## Summary

In this chapter, you learned:

1. **Advanced SELECT**: Expressions, CASE, filtering
2. **Joins**: INNER, LEFT, RIGHT, FULL, CROSS, and self joins
3. **Aggregation**: GROUP BY, HAVING, GROUPING SETS
4. **Subqueries**: Scalar, table, correlated subqueries
5. **CTEs**: Regular and recursive Common Table Expressions
6. **Window Functions**: Ranking, running totals, LAG/LEAD, frames
7. **Set Operations**: UNION, INTERSECT, EXCEPT

## Source Code Connection

The query processing for all these operations happens in:

- **Parser**: `src/backend/parser/` - SQL text to parse tree
- **Analyzer**: `src/backend/parser/analyze.c` - Semantic analysis
- **Optimizer**: `src/backend/optimizer/` - Query planning
- **Executor**: `src/backend/executor/` - Running the query

Key executor files for what we learned:
- `nodeAgg.c` - Aggregation
- `nodeHash.c`, `nodeHashjoin.c` - Hash joins
- `nodeMergejoin.c` - Merge joins
- `nodeNestloop.c` - Nested loop joins
- `nodeWindowAgg.c` - Window functions
- `nodeSetOp.c` - Set operations

---

**Previous Chapter:** [← Getting Started](01-getting-started.md)

**Next Chapter:** [Data Types →](03-data-types.md)

