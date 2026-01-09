# Chapter 16: Extension Development & Server Extensibility

## Learning Objectives

By the end of this chapter, you will be able to:
- Explain what a PostgreSQL **extension** is (as a packaging and deployment mechanism)
- Create a simple SQL-only extension (control file + versioned SQL script)
- Understand how C extensions are built and loaded (`MODULE_PATHNAME`, `PGXS`, `PG_MODULE_MAGIC`)
- Recognize when custom types/operators/operator classes require C
- Understand what FDWs, background workers, and hooks are (and when they’re appropriate)

## Prerequisites

- You have a working PostgreSQL 18 environment and can connect with `psql`
- For the build sections, you have a local build toolchain and `pg_config` on your `PATH`
  - If you installed from source using this curriculum, you already have `pg_config` under `$HOME/pgsql/pg18/bin/pg_config`

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

## Part 1: What an Extension Is (and Isn’t)

An **extension** is primarily a *packaging* mechanism:

- It bundles SQL objects (schemas, tables, functions, types, operators, views) into a named unit.
- It can also ship compiled shared libraries (C code) that SQL objects can reference.
- It supports **versioning and upgrades** (`my_ext--1.0.sql`, `my_ext--1.0--1.1.sql`).

An extension is *not* required to create functions or triggers—you can create those directly in the database (Chapter 11). Extensions matter when you need repeatable installation, upgrades, and distribution across environments.

## Part 2: Creating an Extension (SQL-only)

### Extension Structure

```
my_extension/
├── my_extension.control       # Extension metadata
├── my_extension--1.0.sql      # Installation script (versioned)
├── my_extension--1.0--1.1.sql # Optional upgrade script(s)
├── Makefile                   # Build/install via PGXS
└── README.md                  # Documentation
```

### Control File

**📖 Example:** `my_extension.control`

```ini
# my_extension.control
comment = 'My useful PostgreSQL extension'
default_version = '1.0'
relocatable = true
requires = 'plpgsql'
superuser = false
```

**What the control file options mean**

The `.control` file is metadata that tells PostgreSQL how to install/upgrade your extension.

| Field | Meaning | Notes / gotchas |
|------|---------|-----------------|
| `comment` | Human-readable description | Shown in `\dx` and `pg_available_extensions` |
| `default_version` | Version installed when you run `CREATE EXTENSION my_extension` | Must correspond to an install script `my_extension--<version>.sql` |
| `relocatable` | Whether users can choose a different schema when installing | If `true`, you typically avoid hard-coding schema-qualified names; if `false`, consider setting `schema` |
| `requires` | Comma-separated list of required extensions | Ensures dependencies are installed first (e.g., `plpgsql`, `hstore`) |
| `superuser` | Whether installation requires superuser | Set `false` for “SQL-only, safe” extensions; set `true` if you need superuser-only actions (often the case for C extensions on many systems) |
| `schema` | Schema to install into (non-relocatable extensions) | Common for extensions that want a fixed namespace (e.g., `my_extension`) |
| `module_pathname` | Path to the shared library for C code | Usually used in SQL as `'MODULE_PATHNAME'` so the right library path is resolved at install time |

> **Practical guideline:** If you’re building a SQL-only extension, start with `superuser = false`, `relocatable = true`, and a clean schema design. If you’re shipping C code, you’ll often end up with `module_pathname` and `superuser = true` depending on your deployment environment.

### SQL Installation Script

**📖 Example:** `my_extension--1.0.sql`

> ```sql
> -- Complain if script is sourced in psql, rather than via CREATE EXTENSION
> \echo Use "CREATE EXTENSION my_extension" to load this file. \quit
>
> -- Create schema for extension objects
> CREATE SCHEMA IF NOT EXISTS my_extension;
>
> -- Create functions
> CREATE FUNCTION my_extension.useful_function(input TEXT)
> RETURNS TEXT AS $$
>     SELECT 'Processed: ' || input;
> $$ LANGUAGE SQL IMMUTABLE;
>
> -- Create views
> CREATE VIEW my_extension.summary AS
> SELECT count(*) AS total_tables
> FROM information_schema.tables
> WHERE table_schema = 'public';
> ```

### Makefile (PGXS)

**📖 Example:** `Makefile` (build/install via `pg_config` + PGXS)

> ```makefile
> EXTENSION = my_extension
> DATA = my_extension--1.0.sql
>
> PG_CONFIG = pg_config
> PGXS := $(shell $(PG_CONFIG) --pgxs)
> include $(PGXS)
> ```

### Install and Use Extension

**📖 Example:** Build/install from your shell (not in `psql`)

> ```bash
> # In the extension directory:
> make
> sudo make install
> ```

**🔬 Try It:** Extension lifecycle in SQL

```sql
-- Create extension
CREATE EXTENSION my_extension;

-- Use extension
SELECT my_extension.useful_function('test');

-- List installed extensions
SELECT extname, extversion, extnamespace::regnamespace
FROM pg_extension
ORDER BY extname;

-- Drop extension
DROP EXTENSION my_extension;
```

## Part 3: C Extensions (High-performance, low-level integration)

SQL functions and PL/pgSQL are great for “database business logic.” C extensions are for:

- High-performance CPU-bound work
- Custom data types and operators
- Index support methods / operator classes
- Deep integration points (hooks, background workers)

### Simple C Function

**📖 Example:** `my_cfunc.c`

> ```c
> #include "postgres.h"
> #include "fmgr.h"
> #include "utils/builtins.h"
>
> PG_MODULE_MAGIC;
>
> PG_FUNCTION_INFO_V1(my_add);
> Datum
> my_add(PG_FUNCTION_ARGS)
> {
>     int32 a = PG_GETARG_INT32(0);
>     int32 b = PG_GETARG_INT32(1);
>     PG_RETURN_INT32(a + b);
> }
> ```

### SQL Definition for C Function

**📖 Example:** `my_cfunc--1.0.sql`

> ```sql
> CREATE FUNCTION my_add(integer, integer)
> RETURNS integer
> AS 'MODULE_PATHNAME', 'my_add'
> LANGUAGE C IMMUTABLE STRICT;
> ```

### Build C Extension (PGXS)

**📖 Example:** `Makefile` for a C extension (illustrative)

> ```makefile
> MODULES = my_cfunc
> EXTENSION = my_cfunc
> DATA = my_cfunc--1.0.sql
>
> PG_CONFIG = pg_config
> PGXS := $(shell $(PG_CONFIG) --pgxs)
> include $(PGXS)
> ```

> **Operational note:** Installing a C extension usually requires filesystem access on the server and often superuser privileges. Many managed PostgreSQL services restrict this.

## Part 4: Custom Types, Operators, and Operator Classes

You *can* create custom types and operators in SQL, but anything that needs tight integration with indexing typically requires C support functions.

### Custom Types (SQL-level)

**🔬 Try It:** Composite + domain types

```sql
-- Composite type
CREATE TYPE address AS (
    street TEXT,
    city TEXT,
    state CHAR(2),
    zip VARCHAR(10)
);

CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name TEXT,
    headquarters address
);

INSERT INTO companies (name, headquarters)
VALUES ('Acme Corp', ROW('123 Main St', 'Springfield', 'IL', '62701'));

SELECT name, (headquarters).city FROM companies;

-- Domain types
CREATE DOMAIN email_address AS TEXT
CHECK (VALUE ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');

CREATE DOMAIN positive_int AS INTEGER
CHECK (VALUE > 0);

CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    email email_address,
    priority positive_int
);

INSERT INTO contacts (email, priority) VALUES ('test@example.com', 5);
-- This fails:
-- INSERT INTO contacts (email, priority) VALUES ('invalid-email', 5);
```

### Custom Operators (SQL-level)

**🔬 Try It:** Define and use a custom operator

```sql
CREATE FUNCTION array_contains_all(arr1 ANYARRAY, arr2 ANYARRAY)
RETURNS BOOLEAN AS $$
    SELECT arr2 <@ arr1;
$$ LANGUAGE SQL IMMUTABLE;

CREATE OPERATOR @>@ (
    LEFTARG = ANYARRAY,
    RIGHTARG = ANYARRAY,
    FUNCTION = array_contains_all,
    COMMUTATOR = @<@
);

SELECT ARRAY[1,2,3,4,5] @>@ ARRAY[1,3,5];  -- true
```

### Operator Classes (for indexing)

Operator classes tell an index access method how to:
- compare values
- normalize keys
- support strategies (e.g., B-tree ordering, GiST consistency checks)

In practice, non-trivial operator classes almost always require C functions.

## Part 5: Foreign Data Wrappers (FDWs)

FDWs let you expose external systems as tables. Conceptually:
- Planner builds a plan that includes **foreign scans**
- The FDW implements callbacks that fetch rows / push down predicates

**📖 Example:** `postgres_fdw` configuration (illustrative)

> ```sql
> CREATE EXTENSION postgres_fdw;
>
> CREATE SERVER remote_server
> FOREIGN DATA WRAPPER postgres_fdw
> OPTIONS (host 'remote_host', dbname 'remote_db', port '5432');
>
> CREATE USER MAPPING FOR local_user
> SERVER remote_server
> OPTIONS (user 'remote_user', password 'secret');
> ```

## Part 6: Background Workers and Hooks (Expert-only)

These are powerful but come with sharp edges:
- They run *inside* the PostgreSQL server process space
- Bugs can crash the whole server
- Hooks can change behavior in ways that are hard to debug

### Background Workers (Conceptual)

**📖 Example:** Skeleton (illustrative)

> ```c
> #include "postgres.h"
> #include "postmaster/bgworker.h"
> #include "storage/latch.h"
>
> PG_MODULE_MAGIC;
>
> void _PG_init(void);
> PGDLLEXPORT void my_worker_main(Datum main_arg);
> ```

### Hook System (Conceptual)

**📖 Example:** Executor hook (illustrative)

> ```c
> static ExecutorRun_hook_type prev_ExecutorRun = NULL;
>
> void _PG_init(void)
> {
>     prev_ExecutorRun = ExecutorRun_hook;
>     ExecutorRun_hook = my_ExecutorRun;
> }
> ```

## Part 7: Contrib Extensions You Should Know

Contrib modules live under `contrib/` in the source tree. Many are worth learning even if you never develop extensions yourself.

**🔬 Try It:** Install and sample a few contrib extensions

```sql
-- pg_trgm: Trigram text similarity
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SELECT similarity('PostgreSQL', 'Postgres');

-- hstore: Key-value store
CREATE EXTENSION IF NOT EXISTS hstore;
SELECT 'a=>1,b=>2'::hstore;

-- pg_prewarm: Buffer cache warming
CREATE EXTENSION IF NOT EXISTS pg_prewarm;
SELECT pg_prewarm('employees');
```

> **Note:** Some contrib extensions require `shared_preload_libraries` or special build/install steps (you set up `pg_stat_statements` earlier in Chapter 1).

## Source Code References

| Component | Source Directory / File |
|-----------|--------------------------|
| Extension infrastructure | `src/backend/commands/extension.c` |
| PGXS build system | `src/makefiles/pgxs.mk` |
| C function manager | `src/backend/utils/fmgr/` |
| FDW infrastructure | `src/backend/foreign/` |
| Background workers | `src/backend/postmaster/bgworker.c` |
| Hooks | Search for `_hook` types under `src/include/` and assignments under `src/backend/` |
| Contrib modules | `contrib/` |

## Summary

In this chapter, you learned:
- How extensions package SQL and/or C into installable, versioned units
- How C extensions are built and loaded
- Where advanced extensibility features fit (operator classes, FDWs, background workers, hooks)

## Exercises for Mastery

1. Build a SQL-only extension that creates a schema and 2 functions, and practice `DROP EXTENSION ... CASCADE`.
2. Read `contrib/pg_trgm/` and locate where the SQL objects are defined vs the C code.
3. Sketch an FDW design for a “remote HR system” (what predicates can you push down?).

---

## Chapter Cleanup

The following objects were created for demonstration purposes in this chapter:

**🔬 Try It:** Clean up demonstration objects

```sql
DROP TABLE IF EXISTS companies CASCADE;
DROP TABLE IF EXISTS contacts CASCADE;

DROP TYPE IF EXISTS address CASCADE;
DROP DOMAIN IF EXISTS email_address CASCADE;
DROP DOMAIN IF EXISTS positive_int CASCADE;

DROP FUNCTION IF EXISTS array_contains_all(ANYARRAY, ANYARRAY);
```

---

**Previous Chapter:** [← Autovacuum & Bloat](15-autovacuum-and-bloat.md)


