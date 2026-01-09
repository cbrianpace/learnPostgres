# Chapter 1: Getting Started with PostgreSQL

## Learning Objectives

By the end of this chapter, you will be able to:
- Install PostgreSQL (from source or using Docker)
- Start and stop the PostgreSQL server
- Connect to the database using psql
- Create your first database and table
- Perform basic CRUD operations

## Choosing Your Installation Method

You have **two options** for running PostgreSQL locally. Choose the one that best fits your goals:

| | **Option A: From Source** | **Option B: Docker** |
|---|---------------------------|----------------------|
| **Best for** | Learning internals, debugging, contributing | Quick setup, testing, experimentation |
| **Setup time** | 15-30 minutes | 2-5 minutes |
| **Prerequisites** | C compiler, libraries | Docker Desktop |
| **Disk space** | ~500MB compiled | ~400MB image |
| **Flexibility** | Full control, can modify code | Easy version switching |
| **Recommended if** | You want deep understanding | You want to start coding SQL fast |

> **Our Recommendation:** If your primary goal is learning SQL and PostgreSQL features, start with **Docker** (Option B) to get running quickly. You can always build from source later when you want to explore internals.
>
> If you want the complete experience of understanding PostgreSQL from the ground up, start with **Option A**.

Both options will give you a fully functional PostgreSQL 18 installation. The rest of this curriculum works identically regardless of which option you choose.

---

## Option A: Installing PostgreSQL from Source

Building from source gives you the deepest understanding of how PostgreSQL works. You'll be able to modify the code, set breakpoints, and truly understand the internals covered in later chapters.

### Prerequisites

Before compiling, ensure you have these tools and libraries installed:

```bash
# On macOS (using Homebrew)
brew install readline openssl icu4c pkg-config

# On Ubuntu/Debian
sudo apt-get update
sudo apt-get install build-essential libreadline-dev zlib1g-dev \
    libicu-dev pkg-config bison flex

# On RHEL/CentOS/Fedora
sudo dnf install gcc make readline-devel zlib-devel \
    libicu-devel pkg-config bison flex
# (Use 'yum' instead of 'dnf' on older systems)
```

**Key dependencies:**
- **readline** - Command-line editing in psql
- **zlib** - Compression support
- **ICU** - Unicode collation support (required in modern PostgreSQL)
- **bison/flex** - Parser generators (needed to build from source)
- **pkg-config** - Library detection

### Building PostgreSQL

Navigate to the PostgreSQL source directory and run:

```bash
# Configure the build (see note for MacOS)
./configure --prefix=$HOME/pgsql/pg18
```

**Troubleshooting configure errors:**

If you get an **ICU library not found** error:
```bash
# macOS: ICU is installed via Homebrew but needs explicit paths
./configure --prefix=$HOME/pgsql/pg18 \
    ICU_CFLAGS="-I$(brew --prefix icu4c)/include" \
    ICU_LIBS="-L$(brew --prefix icu4c)/lib -licui18n -licuuc -licudata"

# Or disable ICU if you don't need it (not recommended for production)
./configure --prefix=$HOME/pgsql/pg18 --without-icu
```

If you get an **OpenSSL not found** error on macOS:
```bash
./configure --prefix=$HOME/pgsql/pg18 \
    --with-openssl \
    --with-includes=$(brew --prefix openssl)/include \
    --with-libraries=$(brew --prefix openssl)/lib
```

**Complete macOS configure with all Homebrew paths:**
```bash
./configure --prefix=$HOME/pgsql/pg18 \
    ICU_CFLAGS="-I$(brew --prefix icu4c)/include" \
    ICU_LIBS="-L$(brew --prefix icu4c)/lib -licui18n -licuuc -licudata" \
    --with-openssl \
    --with-includes="$(brew --prefix openssl)/include:$(brew --prefix readline)/include" \
    --with-libraries="$(brew --prefix openssl)/lib:$(brew --prefix readline)/lib"
```

Once configure succeeds, compile and install:

```bash
# Compile (use -j for parallel build)
make -j4

# Install
make install
```

**Alternative: Minimal build (if library issues persist):**
```bash
# Minimal build that avoids library detection issues
./configure --prefix=$HOME/pgsql/pg18 --without-icu --without-readline
make -j4
make install
```
Note: This disables Unicode collation (ICU) and command-line editing (readline), but gives you a working PostgreSQL for learning.

### Installing Contrib Modules (Important!)

PostgreSQL comes with many useful extensions in the `contrib/` directory, but they aren't installed by default. These include:

| Extension | Purpose |
|-----------|---------|
| `btree_gist` | GiST support for scalar types (needed for exclusion constraints) |
| `pg_stat_statements` | Query performance statistics |
| `uuid-ossp` | UUID generation functions |
| `pg_trgm` | Trigram text similarity |
| `hstore` | Key-value storage |
| `tablefunc` | Crosstab/pivot tables |
| `pgcrypto` | Cryptographic functions |

**To install all contrib modules:**

```bash
# From the PostgreSQL source directory
cd contrib
make -j4
make install
```

### Installing pgaudit (Third-Party Extension)

`pgaudit` provides detailed session and object audit logging—essential for compliance and security monitoring. Unlike contrib modules, it must be downloaded separately:

```bash
# From your home directory or a source directory
cd ~
git clone https://github.com/pgaudit/pgaudit.git
cd pgaudit

# Checkout the branch matching your PostgreSQL version
git checkout REL_18_STABLE

# Build and install (requires pg_config in your PATH)
make install USE_PGXS=1
```

> **Note**: `USE_PGXS=1` tells the build system to use PostgreSQL's extension build infrastructure. This works for most third-party extensions.

After installation, `pgaudit` will be available to load via `shared_preload_libraries` (we'll configure this during `initdb`).

**What's happening here?**

1. `./configure` - Detects your system's capabilities and creates build files
2. `make` - Compiles all source code into executable binaries
3. `make install` - Copies the binaries to the installation directory

### Understanding the Configure Script

Let's peek at what configure does. Open `configure.ac` in the root directory:

```bash
# This is an Autoconf input file that generates the configure script
# Key things it checks:
# - C compiler availability
# - Required libraries (readline, zlib)
# - Optional features (SSL, LDAP, etc.)
```

### Setting Up Your Environment

Add PostgreSQL to your PATH:

```bash
# Add to your ~/.bashrc or ~/.zshrc
export PATH=$HOME/pgsql/pg18/bin:$PATH
export LD_LIBRARY_PATH=$HOME/pgsql/pg18/lib:$LD_LIBRARY_PATH
export PGDATA=$HOME/pgsql/pg18/data

# Reload your shell
source ~/.bashrc  # or source ~/.zshrc
```

---

## Option B: Running PostgreSQL with Docker

> **Skip this section** if you already built PostgreSQL from source (Option A).

Docker provides a quick, clean way to run PostgreSQL without compiling anything. This is ideal for:
- Quick experimentation and learning SQL
- Testing different PostgreSQL versions
- Clean, isolated environments
- Avoiding system dependency issues

### Prerequisites

Install Docker:
- **macOS**: Download [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- **Linux**: `sudo apt-get install docker.io` (Ubuntu/Debian) or `sudo dnf install docker` (Fedora)
- **Windows**: Download [Docker Desktop](https://www.docker.com/products/docker-desktop/)

### Quick Start with Docker

```bash
# Pull the official PostgreSQL image
docker pull postgres:18

# Run PostgreSQL container
docker run --name learning-postgres \
    -e POSTGRES_PASSWORD=learningpass \
    -e POSTGRES_DB=learning \
    -p 5432:5432 \
    -v pgdata:/var/lib/postgresql/data \
    -d postgres:18
```

**What these options mean:**
- `--name learning-postgres` - Names the container for easy reference
- `-e POSTGRES_PASSWORD=learningpass` - Sets the superuser password (required)
- `-e POSTGRES_DB=learning` - Creates our learning database automatically
- `-p 5432:5432` - Maps container port to host port
- `-v pgdata:/var/lib/postgresql/data` - Persists data in a Docker volume
- `-d` - Runs in detached (background) mode

### Connecting to Docker PostgreSQL

```bash
# Connect using psql inside the container
docker exec -it learning-postgres psql -U postgres -d learning

# Or connect from your host (if you have psql installed)
psql -h localhost -U postgres -d learning
# Password: learningpass
```

### Managing the Docker Container

```bash
# Stop PostgreSQL
docker stop learning-postgres

# Start it again
docker start learning-postgres

# View logs
docker logs learning-postgres

# Remove container (data persists in volume)
docker rm learning-postgres

# Remove data volume (WARNING: deletes all data!)
docker volume rm pgdata
```

### Docker Compose (Recommended)

For a more maintainable setup, create a `docker-compose.yml` file:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:18
    container_name: learning-postgres
    environment:
      POSTGRES_PASSWORD: learningpass
      POSTGRES_DB: learning
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Then run:
```bash
# Start PostgreSQL
docker-compose up -d

# Stop PostgreSQL
docker-compose down

# Stop and remove data
docker-compose down -v
```

### Accessing PostgreSQL Configuration in Docker

```bash
# View current configuration
docker exec learning-postgres cat /var/lib/postgresql/data/postgresql.conf

# Or connect and use SHOW commands
docker exec -it learning-postgres psql -U postgres -c "SHOW all;"
```

### You're Ready!

If you followed Option B (Docker), your PostgreSQL is already running with the `learning` database created. **Skip ahead to [Part 5: Creating Your First Database](#part-5-creating-your-first-database)** — the `learning` database already exists, so you can jump straight to creating tables.

To connect:
```bash
docker exec -it learning-postgres psql -U postgres -d learning
```

---

## Part 2: Initializing the Database Cluster

> **Docker users**: Skip to [Part 5](#part-5-creating-your-first-database). Docker handles initialization automatically.

If you built from source (Option A), you need to initialize a database cluster (also called a data directory):

```bash
# Create the data directory with recommended extensions pre-configured
initdb -D $PGDATA \
    -c shared_preload_libraries='pg_stat_statements, pgaudit'
```

> **What's `-c`?** This sets configuration parameters in `postgresql.conf` during initialization. `pg_stat_statements` is essential for query performance monitoring (used in Chapter 6). You can add more libraries later by editing `postgresql.conf`.

**Alternative: Minimal initialization (no extensions)**
```bash
initdb -D $PGDATA
```

**Exercise 1.1: Explore the Data Directory**

After initialization, explore what was created:

```bash
ls -la $PGDATA
```

You should see:
```
base/           # Where your databases live
global/         # Cluster-wide tables
pg_hba.conf     # Client authentication configuration
pg_ident.conf   # User name mapping
postgresql.conf # Server configuration
pg_wal/         # Write-ahead log files
```

**What the initdb command does:**

Looking at `src/bin/initdb/initdb.c`, you'll find it:
1. Creates the directory structure
2. Initializes the system catalogs
3. Creates the template databases
4. Sets up default configuration files


## Part 3: Starting the Server

> **Docker users**: Your server is already running. Skip to [Part 5](#part-5-creating-your-first-database).

For source installations, start PostgreSQL with:

```bash
# Start in the foreground (useful for learning)
postgres -D $PGDATA

# Or start in the background
pg_ctl -D $PGDATA -l logfile start
```

**Understanding the Startup Process**

When PostgreSQL starts, it executes code starting from `src/backend/main/main.c`:

```c
// Simplified startup flow:
// 1. main() in main.c
// 2. PostmasterMain() - the main server process
// 3. ServerLoop() - listens for connections
// 4. For each connection: fork() -> BackendStartup()
```

**Exercise 1.2: Watch the Server Start**

Start PostgreSQL in verbose mode:

```bash
# Debug level 1
postgres -D $PGDATA -d 1  
```

You'll see detailed logs about initialization steps.

## Part 4: Connecting with psql

psql is PostgreSQL's command-line client for running queries and managing databases. Let's connect:

**Source installation:**
```bash
# Connect to the default database
psql postgres
```

**Docker installation:**
```bash
# Connect via docker exec
docker exec -it learning-postgres psql -U postgres -d learning

# Or if you have psql installed locally
psql -h localhost -U postgres -d learning
# Password: learningpass
```

You should see:
```
psql (18.x)
Type "help" for help.

postgres=#
```

### Essential psql Commands

`psql` is PostgreSQL’s command-line client. It runs SQL, but it also has meta-commands (the backslash commands) that query system catalogs and present the results in a readable way. You’ll use these constantly while learning.

| Command | Description |
|---------|-------------|
| `\?` | Help for psql commands |
| `\h` | Help for SQL commands |
| `\l` | List databases |
| `\dt` | List tables |
| `\d tablename` | Describe a table |
| `\du` | List users/roles |
| `\c dbname` | Connect to a database |
| `\q` | Quit |

**🔬 Try It:** Explore psql

```sql
-- Try these commands in psql:

\l
-- Lists all databases. You should see: postgres, template0, template1

\du
-- Lists all roles. You should see your username as a superuser.

\h CREATE TABLE
-- Shows syntax for CREATE TABLE
```

---

## Part 5: Creating Your First Database

> **Docker users**: The `learning` database was already created when you started the container. Just connect and verify:
> ```bash
> docker exec -it learning-postgres psql -U postgres -d learning
> ```
> Then skip the CREATE DATABASE step below.

This section establishes the single database used throughout the curriculum: **`learning`**. Keeping one database makes the story cohesive and avoids “which database has my tables?” confusion later.

**🔬 Try It:** Create the Learning Database (Source Installation)

```sql
-- Create pgaudit extension
CREATE EXTENSION pgaudit;

-- Create a new database
CREATE DATABASE learning;

-- Connect to it
\c learning

-- Create pg_stat_statements extension
CREATE EXTENSION pg_stat_statements;

-- Verify you're connected
SELECT current_database();
```

Why we install extensions here: later chapters assume you have a stable base environment. Some extensions are pure SQL, while others require server configuration (like preloading). We set this up early so later labs are predictable.

**What happens when you CREATE DATABASE?**

Looking at `src/backend/commands/dbcommands.c`:

1. PostgreSQL copies the template database (template1 by default)
2. Creates a new subdirectory in `$PGDATA/base/`
3. Registers the database in the `pg_database` system catalog

**🔬 Try It:** Find Your Database on Disk

```sql
-- Find the OID (Object ID) of your database
SELECT oid, datname FROM pg_database WHERE datname = 'learning';
```

Now look at your filesystem:

```bash
ls $PGDATA/base/
# You should see a directory matching the OID
```

---

## Part 6: Creating Tables

Let's create our first table. We'll use a company theme throughout this curriculum—starting simple here and expanding in later chapters.

**🔬 Try It:** Create the Company Tables

```sql
-- Create the departments table first (employees will reference it)
CREATE TABLE departments (
    dept_id SERIAL PRIMARY KEY,
    dept_name VARCHAR(100) NOT NULL,
    location VARCHAR(100)
);

-- Create an employees table
CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    hire_date DATE DEFAULT CURRENT_DATE,
    salary NUMERIC(10,2),
    dept_id INTEGER REFERENCES departments(dept_id)
);

-- Verify the tables were created
\d employees
\d departments
```

**Understanding the CREATE TABLE Process**

When you execute CREATE TABLE, PostgreSQL:

1. **Parses** the SQL (`src/backend/parser/`)
2. **Analyzes** the statement (`src/backend/parser/analyze.c`)
3. **Plans** execution (trivial for DDL)
4. **Executes** the command (`src/backend/commands/tablecmds.c`)

**🔬 Try It:** Explore the System Catalog

```sql
-- Find information about your table in the system catalog
SELECT 
    c.relname as table_name,
    c.relfilenode as file_node,
    c.reltuples as row_estimate
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relname = 'employees' 
  AND n.nspname = 'public';
```

---

## Part 7: CRUD Operations

CRUD stands for Create, Read, Update, Delete - the four basic operations on data.

**🔬 Try It:** CREATE (Insert)

```sql
-- First, insert some departments
INSERT INTO departments (dept_name, location) VALUES
    ('Engineering', 'Building A'),
    ('Sales', 'Building B'),
    ('HR', 'Building C');

-- Insert a single employee
INSERT INTO employees (first_name, last_name, email, salary, dept_id)
VALUES ('Alice', 'Johnson', 'alice.johnson@company.com', 85000, 1);

-- Insert multiple employees
INSERT INTO employees (first_name, last_name, email, salary, dept_id) VALUES
    ('Bob', 'Smith', 'bob.smith@company.com', 72000, 1),
    ('Carol', 'White', 'carol.white@company.com', 95000, 2),
    ('David', 'Brown', 'david.brown@company.com', 68000, 3);

-- Insert with RETURNING to see what was inserted
INSERT INTO employees (first_name, last_name, email, salary, dept_id)
VALUES ('Eve', 'Davis', 'eve.davis@company.com', 78000, 1)
RETURNING *;
```

**🔬 Try It:** READ (Select)

```sql
-- Select all columns, all rows
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, salary FROM employees;

-- Filter rows with WHERE
SELECT * FROM employees WHERE salary > 75000;

-- Sort results
SELECT * FROM employees ORDER BY salary DESC;

-- Limit results
SELECT * FROM employees ORDER BY salary DESC LIMIT 3;

-- Count rows
SELECT COUNT(*) FROM employees;

-- Aggregate functions
SELECT AVG(salary) as average_salary, MAX(salary) as highest_salary FROM employees;
```

**🔬 Try It:** UPDATE

```sql
-- Update a single row
UPDATE employees 
SET salary = 98000 
WHERE first_name = 'Carol' AND last_name = 'White';

-- Update multiple columns
UPDATE employees 
SET salary = 75000
WHERE first_name = 'Bob';

-- Update with RETURNING to see changes
UPDATE employees 
SET salary = salary * 1.05 
WHERE salary < 70000
RETURNING first_name, last_name, salary;
```

**🔬 Try It:** DELETE

```sql
-- Delete a specific row
DELETE FROM employees WHERE first_name = 'Eve' AND last_name = 'Davis';
```

**📖 Example:** Dangerous DELETE operations (don't execute):

> ```sql
> -- Delete with a condition (be careful!)
> DELETE FROM employees WHERE salary < 50000;
> 
> -- Delete all rows (very dangerous!)
> DELETE FROM employees;
> 
> -- Safer: Use TRUNCATE to remove all rows quickly
> TRUNCATE TABLE employees CASCADE;
> ```

**🔬 Try It:** Practice CRUD Operations

Complete these exercises:

1. Insert 3 new employees with different salaries
2. Select all employees with salary between 70000 and 90000
3. Update the hire_date for all employees to '2024-01-15'
4. Give everyone in Engineering (dept_id = 1) a 10% raise
5. Count how many employees remain

**🔬 Try It:** One possible solution

```sql
-- 1. Insert new employees
INSERT INTO employees (first_name, last_name, email, salary, dept_id) VALUES
    ('Frank', 'Miller', 'frank.miller@company.com', 82000, 2),
    ('Grace', 'Lee', 'grace.lee@company.com', 91000, 1),
    ('Henry', 'Wilson', 'henry.wilson@company.com', 65000, 3);

-- 2. Select with range
SELECT * FROM employees WHERE salary BETWEEN 70000 AND 90000;

-- 3. Update all hire dates
UPDATE employees SET hire_date = '2024-01-15';

-- 4. Give Engineering a raise
UPDATE employees SET salary = salary * 1.10 WHERE dept_id = 1;

-- 5. Count
SELECT COUNT(*) FROM employees;
```

---

## Part 8: Transactions

PostgreSQL supports full ACID transactions. Let's explore:

**🔬 Try It:** Basic Transaction

```sql
-- Start a transaction
BEGIN;

-- Make some changes
INSERT INTO employees (first_name, last_name, email, salary, dept_id) 
VALUES ('Test', 'User', 'test.user@company.com', 50000, 1);
UPDATE employees SET salary = 55000 WHERE email = 'test.user@company.com';

-- See the changes (within transaction)
SELECT * FROM employees WHERE email = 'test.user@company.com';

-- Commit to make changes permanent
COMMIT;
```

**📖 Example:** Rolling back a transaction (illustration):

> ```sql
> BEGIN;
> DELETE FROM employees WHERE dept_id = 1;
> -- Oops, that was a mistake!
> ROLLBACK;  -- Undoes everything since BEGIN
> ```

**🔬 Try It:** Observe Transaction Isolation

Open **two** psql sessions to the same database to see isolation in action:

**Session 1:**

**🔬 Try It:** Session 1 (hold the transaction open)
```sql
BEGIN;
UPDATE employees SET salary = 1 WHERE first_name = 'Alice' AND last_name = 'Johnson';
-- Don't commit yet! Leave this session open.
```

**Session 2:**

**🔬 Try It:** Session 2 (observe what you can see)
```sql
SELECT * FROM employees WHERE first_name = 'Alice' AND last_name = 'Johnson';
-- What salary do you see? (Should still be the old value!)
```

**Session 1:**

**🔬 Try It:** Session 1 (commit)
```sql
COMMIT;
```

**Session 2:**

**🔬 Try It:** Session 2 (observe again after commit)
```sql
SELECT * FROM employees WHERE first_name = 'Alice' AND last_name = 'Johnson';
-- Now you see the new salary
```

This demonstrates PostgreSQL's isolation—uncommitted changes are invisible to other sessions.

---

## Part 9: Stopping the Server

**Source installation:**

**🔬 Try It:** Stop PostgreSQL (source install)
```bash
# Graceful shutdown
pg_ctl -D $PGDATA stop

# Or specify shutdown mode
pg_ctl -D $PGDATA stop -m fast   # Disconnect clients, shutdown
pg_ctl -D $PGDATA stop -m smart  # Wait for clients to disconnect
pg_ctl -D $PGDATA stop -m immediate  # Abort connections, may require recovery
```

**Docker:**

**🔬 Try It:** Stop PostgreSQL (Docker)
```bash
# Stop the container (keeps data)
docker stop learning-postgres

# Start it again later
docker start learning-postgres

# Remove container but keep data volume
docker rm learning-postgres

# Remove everything including data (careful!)
docker rm learning-postgres && docker volume rm pgdata
```

## Summary

In this chapter, you learned:

1. **Installation**: Two options—building from source OR using Docker
2. **Initialization**: Setting up the data directory (or letting Docker handle it)
3. **Server Management**: Starting and stopping PostgreSQL
4. **psql**: Using the command-line client
5. **Database Creation**: Creating databases and tables
6. **CRUD Operations**: Insert, Select, Update, Delete
7. **Transactions**: Basic transaction control

## Quick Reference

**Source Installation:**
```bash
# Initialize database cluster
initdb -D $PGDATA

# Start server
pg_ctl -D $PGDATA start

# Stop server
pg_ctl -D $PGDATA stop

# Connect with psql
psql dbname
```

**Docker Installation:**
```bash
# Start PostgreSQL
docker run --name learning-postgres \
    -e POSTGRES_PASSWORD=learningpass \
    -e POSTGRES_DB=learning \
    -p 5432:5432 \
    -v pgdata:/var/lib/postgresql/data \
    -d postgres:18

# Connect with psql
docker exec -it learning-postgres psql -U postgres -d learning

# Stop PostgreSQL
docker stop learning-postgres
```

**🔬 Try It:**
```sql
-- Create database
CREATE DATABASE dbname;

-- Create table
CREATE TABLE tablename (
    column1 TYPE CONSTRAINT,
    column2 TYPE
);

-- CRUD
INSERT INTO table VALUES (...);
SELECT * FROM table WHERE ...;
UPDATE table SET col = val WHERE ...;
DELETE FROM table WHERE ...;

-- Transactions
BEGIN;
-- operations
COMMIT; -- or ROLLBACK;
```

---

**Previous Chapter:** [← Introduction](00-introduction.md)

**Next Chapter:** [SQL Fundamentals →](02-sql-fundamentals.md)

