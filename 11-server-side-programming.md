# Chapter 11: Server-side Programming (Functions, Triggers, Procedural Languages)

## Learning Objectives

By the end of this chapter, you will be able to:
- Create custom functions in SQL and PL/pgSQL
- Implement triggers for auditing, validation, and notifications
- Understand when to use additional procedural languages (PL/Python, PL/Perl)

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

Some examples reference tables from earlier chapters.

## Part 1: User-Defined Functions

Functions encapsulate reusable logic in the database. They can simplify complex queries, enforce business rules, and improve performance by reducing round-trips between application and database.

PostgreSQL supports functions in multiple languages:
- **SQL** - Simple, declarative, automatically inlined
- **PL/pgSQL** - Full procedural language with variables, loops, conditionals
- **PL/Python, PL/Perl, etc.** - Use familiar scripting languages
- **C** - Maximum performance for compute-intensive operations

### SQL Functions

SQL functions are the simplest form—just a SQL query with parameters. PostgreSQL can often "inline" them, replacing the function call with the actual query for better optimization.

**When to use SQL functions:**
- Simple transformations or calculations
- Encapsulating complex SELECT queries
- When you want the planner to see through the function

**🔬 Try It:** SQL Functions

```sql
-- Simple SQL function
CREATE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER AS $$
    SELECT a + b;
$$ LANGUAGE SQL;

SELECT add_numbers(5, 3);  -- Returns 8

-- Function returning table
CREATE FUNCTION get_high_earners(min_salary NUMERIC)
RETURNS TABLE(name TEXT, salary NUMERIC) AS $$
    SELECT first_name || ' ' || last_name, salary
    FROM employees
    WHERE salary >= min_salary;
$$ LANGUAGE SQL;

SELECT * FROM get_high_earners(80000);
```

### PL/pgSQL Functions

PL/pgSQL is PostgreSQL's native procedural language. It adds variables, control flow (IF/THEN/ELSE, loops), exception handling, and more to SQL.

One practical way to choose between SQL and PL/pgSQL:
- If it’s “one SELECT with parameters,” prefer **SQL functions** (better optimizer visibility).
- If you need branching, multiple statements, or explicit error handling, use **PL/pgSQL**.

**Structure of a PL/pgSQL function:**
```
DECLARE
    variable_name type;        -- Variable declarations
BEGIN
    -- Function body with SQL and procedural code
    RETURN value;
EXCEPTION
    WHEN error_type THEN       -- Optional error handling
        -- Handle error
END;
```

**When to use PL/pgSQL:**
- Complex business logic with conditionals
- Iterating over result sets
- Need for exception handling
- Performing multiple operations atomically

**🔬 Try It:**
```sql
-- PL/pgSQL with control flow
CREATE OR REPLACE FUNCTION calculate_bonus(emp_id INTEGER)
RETURNS NUMERIC AS $$
DECLARE
    emp_salary NUMERIC;
    bonus_rate NUMERIC;
    years_employed INTEGER;
BEGIN
    -- Get employee info
    SELECT salary, 
           EXTRACT(YEAR FROM AGE(NOW(), hire_date))::INTEGER
    INTO emp_salary, years_employed
    FROM employees
    WHERE employees.emp_id = calculate_bonus.emp_id;
    
    -- Calculate bonus rate based on tenure
    IF years_employed >= 10 THEN
        bonus_rate := 0.15;
    ELSIF years_employed >= 5 THEN
        bonus_rate := 0.10;
    ELSIF years_employed >= 2 THEN
        bonus_rate := 0.05;
    ELSE
        bonus_rate := 0.02;
    END IF;
    
    RETURN emp_salary * bonus_rate;
END;
$$ LANGUAGE plpgsql;

-- Function with exception handling
CREATE OR REPLACE FUNCTION safe_divide(a NUMERIC, b NUMERIC)
RETURNS NUMERIC AS $$
BEGIN
    RETURN a / b;
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Division by zero, returning NULL';
        RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

### Function Volatility

Function volatility tells PostgreSQL how the function behaves, enabling important optimizations. Getting this wrong can cause incorrect results or missed optimizations.

| Category | Meaning | Can Use in Index? | Example |
|----------|---------|-------------------|---------|
| `IMMUTABLE` | Same inputs → same output, always | Yes | `square(x)`, `lower(text)` |
| `STABLE` | Same within one query | Expression indexes only | `current_user`, settings lookups |
| `VOLATILE` | Can change anytime (default) | No | `random()`, `now()`, any INSERT/UPDATE |

**Why it matters:**
- `IMMUTABLE` functions can be pre-evaluated, used in index expressions, and cached
- Marking a function incorrectly (e.g., `IMMUTABLE` when it reads from tables) can return stale data
- `VOLATILE` functions must be called every time, preventing many optimizations

**🔬 Try It:**
```sql
-- IMMUTABLE: Same inputs always give same outputs, can be optimized
CREATE FUNCTION square(x INTEGER)
RETURNS INTEGER AS $$
    SELECT x * x;
$$ LANGUAGE SQL IMMUTABLE;

-- STABLE: Returns same value within single query
CREATE FUNCTION get_current_user_id()
RETURNS INTEGER AS $$
    SELECT current_setting('app.user_id')::INTEGER;
$$ LANGUAGE SQL STABLE;

-- VOLATILE (default): Can return different values each call
CREATE FUNCTION get_random_employee()
RETURNS TEXT AS $$
    SELECT first_name FROM employees ORDER BY random() LIMIT 1;
$$ LANGUAGE SQL VOLATILE;
```

## Part 2: Triggers

Triggers automatically execute functions when specific events occur on a table. They're powerful for:
- **Audit logging**: Track who changed what and when
- **Data validation**: Enforce complex business rules
- **Derived data**: Automatically update computed columns or summary tables
- **Notifications**: Alert systems of changes

**Trigger timing:**
- `BEFORE`: Run before the operation (can modify or cancel it)
- `AFTER`: Run after the operation (row is already modified)
- `INSTEAD OF`: Replace the operation (for views)

**Trigger scope:**
- `FOR EACH ROW`: Fire once per affected row
- `FOR EACH STATEMENT`: Fire once per SQL statement

### Row-Level Triggers

Row-level triggers fire once for each row affected by the operation. They have access to `OLD` (the row before) and `NEW` (the row after) records.

Conceptually, a row-level trigger is “middleware inside the database”: every row change passes through your function, which can log, validate, or transform it consistently no matter which application issued the SQL.

Special variables available in trigger functions:
- `TG_OP`: The operation ('INSERT', 'UPDATE', 'DELETE')
- `TG_TABLE_NAME`: Name of the table that fired the trigger
- `OLD`: The row before the operation (UPDATE/DELETE)
- `NEW`: The row after the operation (INSERT/UPDATE)

**🔬 Try It:**
```sql
-- Audit log table
CREATE TABLE audit_log ( 
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMPTZ DEFAULT NOW(),
    changed_by TEXT DEFAULT current_user
);

-- Audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log(table_name, operation, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log(table_name, operation, old_data, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log(table_name, operation, old_data)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD));
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Attach trigger to table
CREATE TRIGGER employees_audit
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

**🔬 Try It:** See the audit trigger in action

```sql
-- Insert a single “demo” employee so we don't disturb the main dataset
INSERT INTO employees (first_name, last_name, email, hire_date, salary, dept_id)
VALUES ('Audit', 'Demo', 'audit.demo@company.com', CURRENT_DATE, 70000, (SELECT MIN(dept_id) FROM departments))
RETURNING emp_id \gset

-- Update the row (should log UPDATE with old_data + new_data)
UPDATE employees
SET salary = salary + 1000
WHERE emp_id = :emp_id;

-- Delete the row (should log DELETE with old_data)
DELETE FROM employees
WHERE emp_id = :emp_id;

-- Inspect the audit trail
SELECT table_name, operation, changed_at, changed_by
FROM audit_log
WHERE table_name = 'employees'
ORDER BY changed_at DESC
LIMIT 10;
```

### Statement-Level Triggers

Statement-level triggers fire once per SQL statement, regardless of how many rows are affected. They're useful for:
- Sending notifications after bulk operations
- Updating summary tables after batch changes
- Logging statement-level activity

Note: Statement-level triggers don't have access to individual `OLD`/`NEW` rows (in older PostgreSQL versions). They return NULL instead of a row.

**🔬 Try It:**
```sql
-- Notify on bulk changes
CREATE OR REPLACE FUNCTION notify_changes()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('table_changed', TG_TABLE_NAME);
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employees_notify
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH STATEMENT EXECUTE FUNCTION notify_changes();
```

**🔬 Try It:** See `LISTEN/NOTIFY` with the statement-level trigger

```sql
-- psql will print “Asynchronous notification …” messages after each statement commits.
LISTEN table_changed;

INSERT INTO employees (first_name, last_name, email, hire_date, salary, dept_id)
VALUES ('Notify', 'Demo', 'NOTIFY.DEMO@company.com', CURRENT_DATE, 65000, (SELECT MIN(dept_id) FROM departments))
RETURNING emp_id \gset

UPDATE employees
SET salary = salary + 500
WHERE emp_id = :emp_id;

DELETE FROM employees
WHERE emp_id = :emp_id;

UNLISTEN table_changed;
```

### BEFORE Triggers for Validation

BEFORE triggers run before the data is written, allowing you to:
- **Validate data**: Reject invalid inserts/updates by raising an exception
- **Modify data**: Change `NEW` values before they're written (e.g., set timestamps, normalize text)
- **Cancel operations**: Return NULL to silently skip the row

Returning `NEW` allows the operation to proceed (possibly with your modifications). Raising an exception aborts the entire transaction.

**🔬 Try It:**
```sql
-- Validate salary before insert/update
CREATE OR REPLACE FUNCTION validate_salary()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.salary < 0 THEN
        RAISE EXCEPTION 'Salary cannot be negative';
    END IF;
    
    IF NEW.salary > 1000000 THEN
        RAISE WARNING 'Very high salary: %', NEW.salary;
    END IF;
    
    -- Normalize data
    NEW.email := LOWER(NEW.email);
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER validate_employee_salary
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW EXECUTE FUNCTION validate_salary();
```

**🔬 Try It:** Demonstrate validation + normalization

```sql
-- Create a single demo row (email will be normalized to lowercase by the trigger)
INSERT INTO employees (first_name, last_name, email, hire_date, salary, dept_id)
VALUES ('Validate', 'Demo', 'VALIDATE.DEMO@company.com', CURRENT_DATE, 70000, (SELECT MIN(dept_id) FROM departments))
RETURNING emp_id \gset

-- Email normalized by trigger
SELECT emp_id, email
FROM employees
WHERE emp_id = :emp_id;

-- Negative salary should fail, but we'll recover using a SAVEPOINT so we can continue
BEGIN;
SAVEPOINT bad_salary;
UPDATE employees
SET salary = -1
WHERE emp_id = :emp_id;
ROLLBACK TO SAVEPOINT bad_salary;
COMMIT;

-- High salary should raise a WARNING (but still succeed)
UPDATE employees
SET salary = 1000001
WHERE emp_id = :emp_id;

-- Cleanup
DELETE FROM employees
WHERE emp_id = :emp_id;
```

> If you want to package database objects into installable, versioned extensions (or write C extensions/FDWs/background workers), see **Chapter 16: Extension Development & Server Extensibility**.

## Part 3: Procedural Languages

PostgreSQL can run stored procedures/functions in multiple languages. This is powerful when:

- You want richer control flow or better ergonomics than PL/pgSQL for a specific task.
- You want to reuse an existing library ecosystem (parsing, stats, ML, etc.).
- You want to keep certain business rules close to the data (with the same transactional semantics as SQL).

That power comes with an important security distinction the PostgreSQL community and docs emphasize:

| Category (common wording) | PostgreSQL term | What it means | Typical privilege |
|---|---|---|---|
| “Safe” | **Trusted language** | Code is restricted so it *can’t* escape the database sandbox (no arbitrary file/OS access). | Often installable/usable by non-superusers (depends on policy) |
| “Unsafe” | **Untrusted language** | Code can generally do what the server OS user can do (filesystem, process execution, network, etc.). | **Requires superuser** to install/use |

Examples:
- **PL/pgSQL** is trusted (safe) and ships with PostgreSQL.
- **`plpython3u`** is explicitly **untrusted** (the `u` suffix stands for untrusted).
- Perl historically has both forms (`plperl` trusted vs `plperlu` untrusted), but availability depends on how PostgreSQL was built/installed.

**Best practice:** treat untrusted languages like server-side code deployment. Only enable them when you control the host and you’re comfortable granting superuser-equivalent capabilities. Many managed PostgreSQL services restrict or disallow them.

### PL/Python

Here is an example of deploying and using PL/Python:

```sql
CREATE EXTENSION plpython3u;

CREATE FUNCTION py_max(a INTEGER, b INTEGER)
RETURNS INTEGER AS $$
    return max(a, b)
$$ LANGUAGE plpython3u;

-- Access database from Python
CREATE FUNCTION py_get_employees()
RETURNS SETOF employees AS $$
    result = plpy.execute("SELECT * FROM employees")
    return result
$$ LANGUAGE plpython3u;

-- Use Python libraries
CREATE FUNCTION calculate_statistics(numbers FLOAT[])
RETURNS JSONB AS $$
    import json
    import statistics
    
    return json.dumps({
        'mean': statistics.mean(numbers),
        'median': statistics.median(numbers),
        'stdev': statistics.stdev(numbers) if len(numbers) > 1 else 0
    })
$$ LANGUAGE plpython3u;
```

### PL/Perl

Here is an example of deploying and using PL/Perl:

```sql
CREATE EXTENSION plperl;

CREATE FUNCTION perl_reverse(text)
RETURNS text AS $$
    return scalar reverse $_[0];
$$ LANGUAGE plperl;
```

## Source Code References

| Component | Source Directory |
|-----------|-----------------|
| PL/pgSQL | `src/pl/plpgsql/` |
| Trigger infrastructure | `src/backend/commands/trigger.c` |
| Trigger execution | `src/backend/commands/trigger.c`, `src/backend/executor/execMain.c` |
| Procedural languages | `src/pl/` |

## Summary

In this chapter, you learned:

1. **Functions**: SQL, PL/pgSQL, volatility categories
2. **Triggers**: Row-level, statement-level, BEFORE/AFTER
3. **Procedural Languages**: PL/Python, PL/Perl

---

## Chapter Cleanup

The following objects were created for demonstration purposes in this chapter:

**🔬 Try It:** Clean up demonstration objects

```sql
-- Drop demonstration tables 
DROP TABLE IF EXISTS audit_log CASCADE;

-- Drop demonstration functions
DROP FUNCTION IF EXISTS add_numbers(INTEGER, INTEGER);
DROP FUNCTION IF EXISTS get_high_earners(NUMERIC);
DROP FUNCTION IF EXISTS square(INTEGER);
DROP FUNCTION IF EXISTS get_current_user_id();
DROP FUNCTION IF EXISTS get_random_employee();
DROP FUNCTION IF EXISTS calculate_bonus(INTEGER);

-- Drop demonstration trigger functions / triggers
DROP TRIGGER IF EXISTS employees_audit ON employees;
DROP TRIGGER IF EXISTS employees_notify ON employees;
DROP TRIGGER IF EXISTS validate_employee_salary ON employees;
DROP FUNCTION IF EXISTS audit_trigger_func();
DROP FUNCTION IF EXISTS notify_changes();
DROP FUNCTION IF EXISTS validate_salary();

-- Drop the statement-level notification channel (optional)
-- (LISTEN/UNLISTEN is session-local, so there's nothing persistent to drop)
```

> **Note:** PL/Python and PL/Perl objects require those languages to be installed, so they may not exist in your environment.

---

**Previous Chapter:** [← Replication](10-replication.md)

**Next Chapter:** [Security →](12-security.md)


