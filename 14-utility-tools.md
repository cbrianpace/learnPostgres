# Chapter 14: PostgreSQL Command-Line Utilities

## Learning Objectives

By the end of this chapter, you will be able to:
- Use `pg_dump` and `pg_restore` for logical backups
- Analyze WAL files with `pg_waldump`
- Inspect cluster control data with `pg_controldata`
- Verify data integrity with `pg_checksums`
- Manage tablespaces and file locations
- Use various diagnostic and maintenance utilities

## Prerequisites

This chapter assumes you have:
- A running PostgreSQL installation (from Chapter 1)
- The `learning` database with our company schema (from Chapters 1-2)
- Access to the PostgreSQL bin directory

**🔬 Try It:** Verify Your Installation

```bash
# Verify your PostgreSQL installation
which pg_dump
pg_dump --version
```

---

## Part 1: Backup Utilities

This chapter is about practical operations: the commands you reach for when you need to **backup**, **restore**, **inspect**, or **debug** a running PostgreSQL system.
Most tools here are “frontend” programs in `src/bin/` that speak to the server (or read files the server creates).

### pg_dump - Logical Database Backup

`pg_dump` creates a logical backup of a database. Unlike physical backups (copying files), logical backups contain SQL commands that can recreate the database.

**Why Use pg_dump?**

| Use Case | pg_dump | Physical Backup |
|----------|---------|-----------------|
| Single database backup | ✅ Best choice | Backs up entire cluster |
| Cross-version migration | ✅ Supported | ❌ Not supported |
| Selective table backup | ✅ Supported | ❌ Not supported |
| Point-in-time recovery | ❌ Limited | ✅ Full support |
| Large database speed | Slower | Faster |

**🔬 Try It:** Basic pg_dump Usage

```bash
# Dump entire database as SQL
pg_dump learning > learning_backup.sql

# Dump with compression (custom format - recommended)
pg_dump -Fc learning > learning_backup.dump

# Dump with directory format (parallel backup)
pg_dump -Fd -j 4 learning -f learning_backup_dir/
```

**Understanding Output Formats:**

| Format | Flag | File Extension | Parallel Restore | Compression | Use Case |
|--------|------|----------------|------------------|-------------|----------|
| Plain SQL | `-Fp` (default) | `.sql` | ❌ | ❌ (gzip externally) | Human readable, simple restore |
| Custom | `-Fc` | `.dump` | ✅ | ✅ Built-in | Most versatile, recommended |
| Directory | `-Fd` | folder | ✅ | ✅ Per-file | Large databases, parallel dump |
| Tar | `-Ft` | `.tar` | ❌ | ❌ | Archive compatibility |

**Selective Backup:**

```bash
# Dump specific tables
pg_dump -t employees -t departments learning > company_core.sql

# Dump all tables matching a pattern
pg_dump -t 'project*' learning > projects.sql

# Dump schema only (no data)
pg_dump --schema-only learning > schema.sql

# Dump data only (no schema)
pg_dump --data-only learning > data.sql

# Exclude specific tables
pg_dump --exclude-table='audit_*' learning > no_audit.sql
```

**Advanced Options:**

```bash
# Include CREATE DATABASE statement
pg_dump -C learning > with_create_db.sql

# Dump with INSERT statements instead of COPY (slower but more portable)
pg_dump --inserts learning > inserts_format.sql

# Include column names in INSERTs (for clarity and safety)
pg_dump --inserts --column-inserts learning > verbose_inserts.sql

# Dump with IF EXISTS (safer for restore)
pg_dump --if-exists --clean learning > clean_restore.sql

# Use multiple connections for parallel dump (directory format only)
pg_dump -Fd -j 4 learning -f learning_parallel/
```

**🔬 Try It:** Create Backups of Your Company Database

```bash
# Create the backups directory
mkdir -p ~/backups

# Create different backup formats
pg_dump learning > ~/backups/learning_plain.sql
pg_dump -Fc learning > ~/backups/learning_custom.dump
pg_dump -Fd -j 2 learning -f ~/backups/learning_dir/

# Compare file sizes
ls -lh ~/backups/

# Dump only the employee-related tables
pg_dump -t employees -t departments -t projects -t project_assignments \
    -Fc learning > ~/backups/company_schema.dump

# Create a schema-only backup for documentation
pg_dump --schema-only --no-owner --no-privileges learning > ~/backups/schema_docs.sql
```

### pg_dumpall - Cluster-Wide Backup

`pg_dumpall` backs up an entire PostgreSQL cluster, including roles and tablespaces that `pg_dump` cannot capture.

If you restore a cluster from only `pg_dump` files, you may discover missing roles or tablespace definitions. `pg_dumpall --globals-only` is the typical companion for that.

**What pg_dumpall Captures That pg_dump Cannot:**

- Roles (users and groups)
- Role memberships
- Tablespace definitions
- Per-role and per-database settings

```bash
# Dump entire cluster
pg_dumpall > full_cluster.sql

# Dump only global objects (roles, tablespaces)
pg_dumpall --globals-only > globals.sql

# Dump roles only
pg_dumpall --roles-only > roles.sql

# Dump tablespaces only
pg_dumpall --tablespaces-only > tablespaces.sql
```

**Best Practice: Combined Backup Strategy**

```bash
#!/bin/bash
# backup_strategy.sh - Complete backup approach

BACKUP_DIR=~/backups/$(date +%Y%m%d)
mkdir -p $BACKUP_DIR

# 1. Backup global objects (roles, tablespaces)
pg_dumpall --globals-only > $BACKUP_DIR/globals.sql

# 2. Backup each database in custom format
for db in $(psql -Atc "SELECT datname FROM pg_database WHERE datistemplate = false"); do
    echo "Backing up $db..."
    pg_dump -Fc $db > $BACKUP_DIR/${db}.dump
done

echo "Backup complete: $BACKUP_DIR"
ls -lh $BACKUP_DIR
```

### pg_restore - Restore from Backup

`pg_restore` restores databases from non-plain-text dumps created by `pg_dump`.

The biggest idea: **plain SQL dumps are restored with `psql`**, while **custom/directory dumps are restored with `pg_restore`**.

**Restore Scenarios:**

```bash
# Restore from custom format
pg_restore -d learning learning_custom.dump

# Restore from directory format
pg_restore -d learning learning_dir/

# Create database and restore
createdb learning_restored
pg_restore -d learning_restored learning_custom.dump

# Restore with parallelism (faster for large databases)
pg_restore -d learning -j 4 learning_dir/

# Restore specific tables only
pg_restore -d learning -t employees -t departments learning_custom.dump

# List contents of backup without restoring
pg_restore -l learning_custom.dump

# Restore using a list file (selective restore)
pg_restore -l learning_custom.dump > toc.txt
# Edit toc.txt to comment out unwanted items
pg_restore -d learning -L toc.txt learning_custom.dump
```

**Handling Restore Errors:**

```bash
# Stop on first error (default continues)
pg_restore -d learning --exit-on-error learning_custom.dump

# Drop objects before recreating
pg_restore -d learning --clean learning_custom.dump

# Don't restore ownership (useful when restoring as different user)
pg_restore -d learning --no-owner learning_custom.dump

# Don't restore privileges
pg_restore -d learning --no-privileges learning_custom.dump
```

**Plain SQL Restore:**

For plain text dumps, use `psql`:

```bash
# Restore plain SQL dump
psql learning < learning_plain.sql

# Restore with error handling
psql -v ON_ERROR_STOP=1 learning < learning_plain.sql
```

---

## Part 2: WAL Analysis Utilities

### pg_waldump - WAL File Inspector

`pg_waldump` decodes and displays Write-Ahead Log records. This is invaluable for understanding PostgreSQL's durability mechanism and debugging replication issues.

**Understanding WAL:**

WAL (Write-Ahead Logging) ensures durability by writing changes to a log before modifying data files. Every change in PostgreSQL generates WAL records.

```bash
# Find WAL files location
psql -c "SHOW data_directory;"
# WAL files are in pg_wal/ subdirectory

# List WAL files
ls -la $PGDATA/pg_wal/

# Dump a WAL file (requires appropriate permissions)
pg_waldump $PGDATA/pg_wal/000000010000000000000001
```

**WAL Record Types:**

| Record Type | Description |
|-------------|-------------|
| Heap | Table row operations (INSERT, UPDATE, DELETE) |
| Btree | B-tree index changes |
| Transaction | COMMIT, ABORT |
| XLOG | Checkpoint, database startup |
| Standby | Replication-related |
| CLOG | Commit status changes |

**Filtering WAL Records:**

```bash
# Show only transaction records
pg_waldump --rmgr=Transaction $PGDATA/pg_wal/000000010000000000000001

# Show records in a lsn range
pg_waldump --start=0/1000000 --end=0/2000000 $PGDATA/pg_wal/000000010000000000000001

# Show specific transaction
pg_waldump --xid=12345 $PGDATA/pg_wal/000000010000000000000001

# Show records for a specific relation (table/index)
pg_waldump --relation=16384/16385/16386 $PGDATA/pg_wal/000000010000000000000001

# Output statistics instead of records
pg_waldump --stats $PGDATA/pg_wal/000000010000000000000001
```

**Exercise 16.3: Explore WAL Records**

**🔬 Try It:**
```sql
-- First, let's generate some WAL records
\c learning

-- Force a checkpoint to start fresh
CHECKPOINT;

-- Note the current WAL position
SELECT pg_current_wal_lsn();

-- Make some changes
INSERT INTO employees (first_name, last_name, email, hire_date, salary, dept_id)
VALUES ('Test', 'WALUser', 'wal@company.com', CURRENT_DATE, 50000, 1);

UPDATE employees SET salary = 55000 WHERE email = 'wal@company.com';

DELETE FROM employees WHERE email = 'wal@company.com';

-- Note the new WAL position
SELECT pg_current_wal_lsn();
```

```bash
# Now examine the WAL records (run as postgres user or with appropriate permissions)
# Find the WAL file containing your changes
pg_waldump --start=YOUR_START_LSN --end=YOUR_END_LSN $PGDATA/pg_wal/CURRENT_WAL_FILE
```

### pg_receivewal - WAL Streaming Archiver

`pg_receivewal` streams WAL from a running server, useful for maintaining WAL archives for point-in-time recovery.

```bash
# Stream WAL to a directory
pg_receivewal -D /path/to/wal_archive -h localhost -p 5432

# With slot (ensures no WAL is missed)
pg_receivewal -D /path/to/wal_archive --slot=archive_slot --create-slot

# Synchronous mode (waits for fsync)
pg_receivewal -D /path/to/wal_archive --synchronous
```

---

## Part 3: Cluster Information Utilities

### pg_controldata - Control File Inspector

`pg_controldata` displays control file information. The control file contains critical cluster metadata.

```bash
# Display control data
pg_controldata $PGDATA
```

**Key Information Displayed:**

| Field | Description |
|-------|-------------|
| `pg_control version` | Control file format version |
| `Catalog version` | System catalog version |
| `Database system identifier` | Unique cluster ID |
| `Database cluster state` | Current state (production, recovery, etc.) |
| `Latest checkpoint location` | Last checkpoint WAL position |
| `Latest checkpoint's REDO location` | Where recovery would start |
| `Latest checkpoint's TimeLineID` | Timeline for recovery |
| `Data page checksum version` | Whether checksums are enabled |

**Cluster States:**

| State | Meaning |
|-------|---------|
| `in production` | Normal operation |
| `shut down` | Clean shutdown |
| `in archive recovery` | Recovering from archive |
| `in crash recovery` | Recovering from crash |
| `shut down in recovery` | Standby was stopped cleanly |

### pg_checksums - Data Checksum Management

Data checksums help detect corruption caused by hardware or filesystem issues.

**Note:** Checksums must be enabled at cluster initialization (`initdb --data-checksums`) or enabled offline on an existing cluster.

```bash
# Check if checksums are enabled
pg_controldata $PGDATA | grep checksum

# Enable checksums on existing cluster (requires server to be stopped!)
pg_ctl stop -D $PGDATA
pg_checksums --enable -D $PGDATA
pg_ctl start -D $PGDATA

# Verify checksums
pg_checksums --check -D $PGDATA

# Show progress during verification
pg_checksums --check --progress -D $PGDATA
```

---

## Part 4: Server Management Utilities

### pg_ctl - Server Control

`pg_ctl` is the primary utility for managing PostgreSQL server processes.

```bash
# Start server
pg_ctl -D $PGDATA start

# Start with specific options
pg_ctl -D $PGDATA -o "-p 5433" start

# Stop server (different modes)
pg_ctl -D $PGDATA stop -m smart     # Wait for clients to disconnect
pg_ctl -D $PGDATA stop -m fast      # Disconnect clients, clean shutdown
pg_ctl -D $PGDATA stop -m immediate # Abort all processes (requires recovery)

# Restart server
pg_ctl -D $PGDATA restart

# Reload configuration
pg_ctl -D $PGDATA reload

# Check server status
pg_ctl -D $PGDATA status

# Promote standby to primary
pg_ctl -D $PGDATA promote
```

**Shutdown Modes Comparison:**

| Mode | Behavior | Use Case |
|------|----------|----------|
| `smart` | Waits for all clients to disconnect | Graceful maintenance |
| `fast` | Disconnects clients, clean shutdown | Normal shutdown |
| `immediate` | Aborts all processes | Emergency (requires recovery on restart) |

### initdb - Initialize Database Cluster

`initdb` creates a new PostgreSQL database cluster.

```bash
# Basic initialization
initdb -D /path/to/new/datadir

# With checksums (recommended)
initdb -D /path/to/new/datadir --data-checksums

# Specify locale and encoding
initdb -D /path/to/new/datadir --locale=en_US.UTF-8 --encoding=UTF8

# Specify authentication method
initdb -D /path/to/new/datadir --auth-local=peer --auth-host=scram-sha-256

# Set superuser password during init
initdb -D /path/to/new/datadir --pwprompt

# Specify WAL directory separately (for performance)
initdb -D /path/to/new/datadir --waldir=/fast/ssd/pg_wal
```

### pg_upgrade - Major Version Upgrade

`pg_upgrade` upgrades a PostgreSQL cluster to a new major version without full dump/restore.

```bash
# Check upgrade compatibility (dry run)
pg_upgrade \
    --old-datadir=/path/to/old/data \
    --new-datadir=/path/to/new/data \
    --old-bindir=/path/to/old/bin \
    --new-bindir=/path/to/new/bin \
    --check

# Perform upgrade
pg_upgrade \
    --old-datadir=/path/to/old/data \
    --new-datadir=/path/to/new/data \
    --old-bindir=/path/to/old/bin \
    --new-bindir=/path/to/new/bin

# Use link mode (faster, but old cluster becomes unusable)
pg_upgrade \
    --old-datadir=/path/to/old/data \
    --new-datadir=/path/to/new/data \
    --old-bindir=/path/to/old/bin \
    --new-bindir=/path/to/new/bin \
    --link
```

---

## Part 5: Maintenance Utilities

### vacuumdb - VACUUM from Command Line

`vacuumdb` runs VACUUM from the command line, useful for scripting and cron jobs.

```bash
# Vacuum a specific database
vacuumdb learning

# Vacuum and analyze
vacuumdb --analyze learning

# Vacuum all databases
vacuumdb --all

# Vacuum specific table
vacuumdb -t employees learning

# Full vacuum (rewrites table, requires exclusive lock)
vacuumdb --full learning

# Parallel vacuum (PostgreSQL 13+)
vacuumdb --jobs=4 learning

# Verbose output
vacuumdb --verbose learning
```

### reindexdb - Rebuild Indexes

`reindexdb` rebuilds indexes, useful when indexes become bloated or corrupted.

```bash
# Reindex specific database
reindexdb learning

# Reindex all databases
reindexdb --all

# Reindex specific table
reindexdb -t employees learning

# Reindex specific index
reindexdb -i employees_pkey learning

# Reindex system catalogs
reindexdb --system learning

# Concurrent reindex (PostgreSQL 12+, doesn't block writes)
reindexdb --concurrently learning
```

### clusterdb - Cluster Tables

`clusterdb` physically reorders table data based on an index, improving query performance for range scans.

```bash
# Cluster all tables in database
clusterdb learning

# Cluster specific table
clusterdb -t employees learning

# Verbose output
clusterdb --verbose learning
```

**Note:** Clustering requires an exclusive lock and rewrites the entire table.

---

## Part 6: Client Utilities

### createdb / dropdb - Database Management

```bash
# Create database
createdb mydb

# Create with specific owner
createdb -O myuser mydb

# Create with encoding and locale
createdb -E UTF8 --locale=en_US.UTF-8 mydb

# Create from template
createdb -T template_postgis mydb

# Drop database
dropdb mydb

# Drop with force (disconnect active connections - PostgreSQL 13+)
dropdb --force mydb
```

### createuser / dropuser - Role Management

```bash
# Create user (interactive)
createuser myuser

# Create superuser
createuser --superuser admin

# Create user with specific permissions
createuser --createdb --createrole manager

# Create with password prompt
createuser --pwprompt myuser

# Create login-disabled role (group)
createuser --no-login developers

# Drop user
dropuser myuser
```

### psql - Command-line Client

While covered in Chapter 1, here are advanced `psql` features:

```bash
# Execute single command
psql -c "SELECT version();" learning

# Execute file
psql -f setup.sql learning

# Execute multiple commands
psql -c "\\dt" -c "SELECT COUNT(*) FROM employees;" learning

# Output formatting
psql -A -t -c "SELECT email FROM employees;" learning  # Unaligned, tuples only

# HTML output
psql -H -c "SELECT * FROM employees;" learning > report.html

# CSV output (PostgreSQL 12+)
psql -c "\\copy employees TO STDOUT WITH CSV HEADER" learning > employees.csv
```

**Useful psql Variables:**

```bash
# Set variables on command line
psql -v dept=1 -c "SELECT * FROM employees WHERE dept_id = :dept;" learning

# Auto-commit off for safety
psql -v AUTOCOMMIT=off learning
```

---

## Part 7: Diagnostic Utilities

### pg_isready - Connection Check

`pg_isready` checks if a PostgreSQL server is ready to accept connections.

```bash
# Basic check
pg_isready

# Check specific host/port
pg_isready -h localhost -p 5432

# Check specific database
pg_isready -d learning

# Quiet mode (only exit code)
pg_isready -q && echo "PostgreSQL is ready"

# Use in scripts
if pg_isready -h localhost -p 5432 -q; then
    echo "Database is available"
else
    echo "Database is not available"
fi
```

**Exit Codes:**

| Code | Meaning |
|------|---------|
| 0 | Server is accepting connections |
| 1 | Server is rejecting connections |
| 2 | No response from server |
| 3 | No attempt was made (bad parameters) |

### pg_config - Build Configuration

`pg_config` displays information about the installed PostgreSQL.

```bash
# Show all configuration
pg_config

# Show specific values
pg_config --bindir
pg_config --includedir
pg_config --pkglibdir
pg_config --configure

# Useful for extension compilation
gcc -I$(pg_config --includedir) -c myextension.c
```

---

## Part 8: Replication Utilities

### pg_basebackup - Physical Base Backup

`pg_basebackup` creates a physical copy of a database cluster, the foundation for streaming replication and point-in-time recovery.

```bash
# Basic backup
pg_basebackup -D /path/to/backup -h localhost -p 5432

# With WAL files included (standalone backup)
pg_basebackup -D /path/to/backup -X stream

# Checkpoint control
pg_basebackup -D /path/to/backup --checkpoint=fast

# Progress reporting
pg_basebackup -D /path/to/backup --progress

# Compressed backup
pg_basebackup -D /path/to/backup -Z 9

# Create replication slot
pg_basebackup -D /path/to/backup --slot=backup_slot --create-slot

# Backup with tablespace mapping
pg_basebackup -D /path/to/backup \
    --tablespace-mapping=/old/tblspc=/new/tblspc
```

**Exercise 16.5: Create a Standby Server**

```bash
# 1. Create base backup
pg_basebackup -D /path/to/standby_data -X stream --progress

# 2. Configure standby (PostgreSQL 12+)
echo "primary_conninfo = 'host=localhost port=5432'" >> /path/to/standby_data/postgresql.auto.conf
touch /path/to/standby_data/standby.signal

# 3. Start standby on different port
pg_ctl -D /path/to/standby_data -o "-p 5433" start

# 4. Verify replication
psql -p 5433 -c "SELECT pg_is_in_recovery();"  # Should return 't'
```

### pg_rewind - Resynchronize Diverged Cluster

`pg_rewind` synchronizes a data directory with another copy, useful after failover.

```bash
# Rewind old primary to follow new primary
pg_rewind \
    --target-pgdata=/path/to/old_primary \
    --source-server="host=new_primary port=5432"

# Dry run
pg_rewind \
    --target-pgdata=/path/to/old_primary \
    --source-server="host=new_primary port=5432" \
    --dry-run
```

---

## Part 9: Practical Exercises

### Exercise 16.1: Complete Backup and Recovery Drill

This exercise simulates a disaster recovery scenario:

```bash
#!/bin/bash
# disaster_recovery_drill.sh

# Setup
BACKUP_DIR=~/dr_drill
mkdir -p $BACKUP_DIR

echo "=== Step 1: Create baseline backup ==="
pg_dump -Fc learning > $BACKUP_DIR/learning_baseline.dump
pg_dumpall --globals-only > $BACKUP_DIR/globals.sql

echo "=== Step 2: Verify backup contents ==="
pg_restore -l $BACKUP_DIR/learning_baseline.dump | head -20

echo "=== Step 3: Simulate data change ==="
psql learning -c "INSERT INTO departments (dept_name, location, budget) 
                  VALUES ('Recovery Test', 'DR Site', 100000) RETURNING *;"

echo "=== Step 4: Take incremental backup ==="
pg_dump -Fc learning > $BACKUP_DIR/learning_updated.dump

echo "=== Step 5: Create test restore database ==="
createdb learning_dr_test

echo "=== Step 6: Restore from baseline ==="
pg_restore -d learning_dr_test $BACKUP_DIR/learning_baseline.dump

echo "=== Step 7: Verify baseline restore ==="
psql learning_dr_test -c "SELECT * FROM departments WHERE dept_name = 'Recovery Test';"
# Should return 0 rows (not in baseline)

echo "=== Step 8: Restore from updated backup ==="
dropdb learning_dr_test
createdb learning_dr_test
pg_restore -d learning_dr_test $BACKUP_DIR/learning_updated.dump

echo "=== Step 9: Verify updated restore ==="
psql learning_dr_test -c "SELECT * FROM departments WHERE dept_name = 'Recovery Test';"
# Should show the Recovery Test department

echo "=== Cleanup ==="
dropdb learning_dr_test
psql learning -c "DELETE FROM departments WHERE dept_name = 'Recovery Test';"
rm -rf $BACKUP_DIR

echo "=== Disaster Recovery Drill Complete ==="
```

### Exercise 16.2: Performance Analysis with Utilities

```bash
#!/bin/bash
# performance_analysis.sh

echo "=== Cluster Status ==="
pg_ctl -D $PGDATA status

echo ""
echo "=== Control Data Summary ==="
pg_controldata $PGDATA | grep -E "(cluster state|checkpoint|checksum)"

echo ""
echo "=== Database Sizes ==="
psql -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) as size 
         FROM pg_database ORDER BY pg_database_size(datname) DESC;"

echo ""
echo "=== Table Bloat Analysis ==="
vacuumdb --analyze --verbose learning 2>&1 | head -30

echo ""
echo "=== Index Health ==="
psql learning -c "SELECT schemaname, tablename, indexname, 
                         pg_size_pretty(pg_relation_size(indexrelid)) as size
                  FROM pg_stat_user_indexes 
                  ORDER BY pg_relation_size(indexrelid) DESC LIMIT 10;"
```

---

## Part 10: Utility Reference Quick Guide

### Backup & Restore

| Utility | Purpose | Key Options |
|---------|---------|-------------|
| `pg_dump` | Dump single database | `-Fc` (custom), `-Fd` (directory), `-t` (table) |
| `pg_dumpall` | Dump entire cluster | `--globals-only`, `--roles-only` |
| `pg_restore` | Restore from dump | `-j` (parallel), `--clean`, `-l` (list) |
| `pg_basebackup` | Physical backup | `-X stream`, `--progress`, `-Z` (compress) |

### Server Management

| Utility | Purpose | Key Options |
|---------|---------|-------------|
| `pg_ctl` | Start/stop/reload | `start`, `stop -m fast`, `reload`, `status` |
| `initdb` | Create cluster | `--data-checksums`, `--locale`, `--waldir` |
| `pg_upgrade` | Major version upgrade | `--check`, `--link`, `--clone` |

### Maintenance

| Utility | Purpose | Key Options |
|---------|---------|-------------|
| `vacuumdb` | Run VACUUM | `--analyze`, `--full`, `--jobs` |
| `reindexdb` | Rebuild indexes | `--concurrently`, `--system` |
| `clusterdb` | Cluster tables | `-t` (specific table) |

### Diagnostics

| Utility | Purpose | Key Options |
|---------|---------|-------------|
| `pg_waldump` | Decode WAL | `--stats`, `--rmgr`, `--xid` |
| `pg_controldata` | Control file info | (no options, just data directory) |
| `pg_checksums` | Verify checksums | `--check`, `--enable`, `--disable` |
| `pg_isready` | Connection check | `-h`, `-p`, `-d`, `-q` |
| `pg_config` | Build info | `--bindir`, `--configure` |

### Replication

| Utility | Purpose | Key Options |
|---------|---------|-------------|
| `pg_receivewal` | Stream WAL | `--slot`, `--create-slot` |
| `pg_rewind` | Resync cluster | `--source-server`, `--dry-run` |

---

## Summary

In this chapter, you learned to use PostgreSQL's command-line utilities for:

1. **Backup & Restore**: `pg_dump`, `pg_dumpall`, `pg_restore`, `pg_basebackup`
2. **WAL Analysis**: `pg_waldump`, `pg_receivewal`
3. **Cluster Info**: `pg_controldata`, `pg_checksums`
4. **Server Management**: `pg_ctl`, `initdb`, `pg_upgrade`
5. **Maintenance**: `vacuumdb`, `reindexdb`, `clusterdb`
6. **Diagnostics**: `pg_isready`, `pg_config`

These utilities are essential for:
- Implementing robust backup strategies
- Managing database clusters
- Troubleshooting issues
- Performing maintenance tasks
- Setting up replication

**Key Takeaways:**

- Use `pg_dump -Fc` for most logical backups (flexible and compressed)
- Use `pg_basebackup` for physical backups and replication setup
- Check cluster state with `pg_controldata` before maintenance
- Use `pg_isready` in scripts to verify database availability
- Regular `vacuumdb --analyze` keeps statistics current

---

**Previous Chapter:** [← Performance Tuning](13-performance-tuning.md)

**Next Chapter:** [Autovacuum, Freezing, and Bloat →](15-autovacuum-and-bloat.md)

**Appendix:** [Configuration Reference →](appendix-a-configuration-reference.md)

