# Appendix B: PostgreSQL Source Code Tour

## Learning Objectives

By the end of this chapter, you will be able to:
- Navigate the PostgreSQL source code effectively
- Understand key data structures and their purposes
- Trace query execution through the codebase
- Read and understand PostgreSQL C code conventions
- Set up a development environment for exploration

## Part 1: Source Code Organization

### Top-Level Structure

This section is an orientation map. You don’t need to memorize it—just learn “where to look” when a chapter references a subsystem (planner, executor, storage, replication, etc.).

```
postgres/
├── src/                    # Source code
│   ├── backend/           # The database server
│   ├── bin/               # Client programs
│   ├── common/            # Code shared by frontend/backend
│   ├── fe_utils/          # Frontend utilities
│   ├── include/           # Header files
│   ├── interfaces/        # Client libraries (libpq)
│   ├── pl/                # Procedural languages
│   ├── port/              # Platform portability
│   ├── template/          # Platform templates
│   ├── test/              # Test suites
│   └── timezone/          # Timezone data
├── contrib/               # Additional modules
├── doc/                   # Documentation
└── config/                # Build configuration
```

### Backend Directory Structure

The `src/backend/` tree is the PostgreSQL server. Most “how does Postgres do X?” answers eventually land somewhere under this directory.

```
src/backend/
├── access/                # Storage access methods
│   ├── brin/             # BRIN indexes
│   ├── common/           # Common access code
│   ├── gin/              # GIN indexes
│   ├── gist/             # GiST indexes
│   ├── hash/             # Hash indexes
│   ├── heap/             # Heap (table) access
│   ├── index/            # Index utilities
│   ├── nbtree/           # B-tree indexes
│   ├── spgist/           # SP-GiST indexes
│   ├── table/            # Table access methods
│   └── transam/          # Transaction manager, WAL
├── catalog/               # System catalog management
├── commands/              # SQL command implementations
├── executor/              # Query executor
├── libpq/                 # Server-side libpq
├── main/                  # Server entry point
├── nodes/                 # Node structures
├── optimizer/             # Query planner/optimizer
│   ├── geqo/             # Genetic optimizer
│   ├── path/             # Path generation
│   ├── plan/             # Plan generation
│   ├── prep/             # Query preprocessing
│   └── util/             # Optimizer utilities
├── parser/                # SQL parser
├── postmaster/            # Process management
├── replication/           # Replication code
├── rewrite/               # Query rewriting (views, rules)
├── storage/               # Storage management
│   ├── buffer/           # Buffer manager
│   ├── file/             # File operations
│   ├── freespace/        # Free space map
│   ├── ipc/              # IPC, shared memory
│   ├── large_object/     # Large objects
│   ├── lmgr/             # Lock manager
│   ├── page/             # Page operations
│   ├── smgr/             # Storage manager
│   └── sync/             # Synchronization
├── tcop/                  # Traffic cop (query dispatcher)
└── utils/                 # Utilities
    ├── adt/              # Abstract data types
    ├── cache/            # Caching
    ├── error/            # Error handling
    ├── fmgr/             # Function manager
    ├── hash/             # Hash utilities
    ├── init/             # Initialization
    ├── mb/               # Multibyte encoding
    ├── misc/             # Miscellaneous
    ├── mmgr/             # Memory management
    ├── sort/             # Sorting
    └── time/             # Time/date utilities
```

## Part 2: Key Entry Points

### Server Startup

The entry point is `main()`, but very quickly PostgreSQL hands off to either **single-user** mode or the normal **postmaster** mode. This is useful when you’re trying to understand what runs “once at startup” vs “once per connection.”

```c
// src/backend/main/main.c
int main(int argc, char *argv[])
{
    // Initialize memory contexts
    MemoryContextInit();
    
    // Parse command line
    // ...
    
    // Start appropriate mode
    if (strcmp(argv[1], "--single") == 0)
        PostgresMain(argc, argv, ...);  // Single-user mode
    else
        PostmasterMain(argc, argv);     // Normal mode
}
```

### Postmaster Loop

The postmaster’s job is mostly orchestration: listen, accept, authenticate, spawn a backend, and keep the cluster’s background processes healthy.

```c
// src/backend/postmaster/postmaster.c
static void ServerLoop(void)
{
    for (;;)
    {
        // Accept new connections
        port = ConnCreate(ListenSocket[i]);
        
        // Fork new backend
        BackendStartup(port);
    }
}
```

### Backend Main Loop

Once a backend process is running, it sits in a loop reading protocol messages from the client and dispatching work. This is where “a SQL statement becomes actual work” at the server.

```c
// src/backend/tcop/postgres.c
void PostgresMain(...)
{
    for (;;)
    {
        // Read command from client
        firstchar = ReadCommand(&input_message);
        
        switch (firstchar)
        {
            case 'Q':  // Simple query
                exec_simple_query(query_string);
                break;
            case 'P':  // Parse
                exec_parse_message(...);
                break;
            // ... more message types
        }
        
        ReadyForQuery(whereToSendOutput);
    }
}
```

## Part 3: Query Execution Path

### Simple Query Flow

```c
// src/backend/tcop/postgres.c
static void exec_simple_query(const char *query_string)
{
    // 1. Parse
    parsetree_list = pg_parse_query(query_string);
    
    foreach(parsetree_item, parsetree_list)
    {
        // 2. Analyze
        querytree_list = pg_analyze_and_rewrite(parsetree, ...);
        
        // 3. Plan
        plantree_list = pg_plan_queries(querytree_list, ...);
        
        // 4. Execute
        foreach(plantree_item, plantree_list)
        {
            PortalRun(portal, ...);
        }
    }
}
```

### Parser Stage

```c
// src/backend/parser/

// Lexical analysis (scanner)
// scan.l - flex scanner definition

// Syntax analysis (grammar)
// gram.y - bison grammar definition

// Result: Parse tree (raw parsetree list)
```

### Analyzer Stage

```c
// src/backend/parser/analyze.c
Query *parse_analyze(RawStmt *parseTree, ...)
{
    // Transform parse tree to query tree
    // - Resolve names to catalog entries
    // - Check types and coercions
    // - Validate semantics
}
```

### Planner Stage

```c
// src/backend/optimizer/plan/planner.c
PlannedStmt *planner(Query *parse, ...)
{
    // 1. Preprocess query
    subquery_planner(glob, parse, ...);
    
    // 2. Generate paths
    // Find all possible ways to execute
    
    // 3. Create plan
    // Convert best path to plan tree
}
```

### Executor Stage

```c
// src/backend/executor/execMain.c
void ExecutorRun(QueryDesc *queryDesc, ...)
{
    // Initialize state
    ExecutorStart(queryDesc, ...);
    
    // Execute plan
    ExecutePlan(estate, planstate, ...);
    
    // Cleanup
    ExecutorEnd(queryDesc);
}
```

## Part 4: Key Data Structures

### Node System

All PostgreSQL data structures are nodes:

```c
// src/include/nodes/nodes.h
typedef struct Node
{
    NodeTag type;  // Identifies node type
} Node;

// Example: Query node
typedef struct Query
{
    NodeTag     type;
    CmdType     commandType;
    List       *rtable;        // Range table
    FromExpr   *jointree;      // Join tree
    List       *targetList;    // Target list
    // ... many more fields
} Query;
```

### List Structure

```c
// src/include/nodes/pg_list.h
typedef struct List
{
    NodeTag     type;
    int         length;
    ListCell   *head;
    ListCell   *tail;
} List;

// Usage
List *list = NIL;                    // Empty list
list = lappend(list, element);       // Add to end
list = lcons(element, list);         // Add to front
foreach(lc, list)                    // Iterate
{
    Node *node = lfirst(lc);
}
```

### Memory Contexts

```c
// src/include/utils/palloc.h
// All allocations happen in memory contexts

MemoryContext oldcxt = MemoryContextSwitchTo(TopMemoryContext);
void *ptr = palloc(size);
// ... use memory ...
pfree(ptr);  // Optional: freed when context is destroyed
MemoryContextSwitchTo(oldcxt);

// Create child context
MemoryContext mycxt = AllocSetContextCreate(
    parent,
    "MyContext",
    ALLOCSET_DEFAULT_SIZES
);
```

### Tuple Descriptor

```c
// src/include/access/tupdesc.h
typedef struct TupleDescData
{
    int         natts;          // Number of attributes
    Oid         tdtypeid;       // Type OID
    int32       tdtypmod;       // Type modifier
    bool        tdhasoid;       // Has OID column
    FormData_pg_attribute attrs[FLEXIBLE_ARRAY_MEMBER];
} TupleDescData;
```

### Buffer Management

```c
// src/include/storage/buf.h
typedef int Buffer;  // Buffer identifier

// Reading a buffer
Buffer buf = ReadBuffer(relation, blockno);
Page page = BufferGetPage(buf);
// ... work with page ...
ReleaseBuffer(buf);

// Writing a buffer
buf = ReadBuffer(relation, blockno);
LockBuffer(buf, BUFFER_LOCK_EXCLUSIVE);
page = BufferGetPage(buf);
// ... modify page ...
MarkBufferDirty(buf);
UnlockReleaseBuffer(buf);
```

## Part 5: Code Conventions

### Naming Conventions

```c
// Functions: CamelCase with module prefix
ExecInitNode()          // Executor function
HeapTupleSatisfies()    // Heap access function
LockAcquire()           // Lock manager function

// Variables: lowercase with underscores
int num_items;
Buffer buf;
HeapTuple tuple;

// Macros: UPPERCASE
#define BUFFER_LOCK_SHARE 1
#define InvalidOid ((Oid) 0)

// Type names: end with _t or Data
typedef int32 CommandId;
typedef struct RelationData *Relation;
```

### Error Handling

```c
// src/backend/utils/error/elog.c

// Report error (aborts transaction)
ereport(ERROR,
    (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
     errmsg("invalid value for parameter \"%s\": %d", name, value)));

// Just log a message
elog(LOG, "starting autovacuum worker");

// Assert (debug builds only)
Assert(condition);
```

### Common Patterns

```c
// Check for null
if (ptr == NULL)
    ereport(ERROR, ...);

// Switch on node type
switch (nodeTag(node))
{
    case T_SelectStmt:
        return transformSelectStmt((SelectStmt *) node);
    case T_InsertStmt:
        return transformInsertStmt((InsertStmt *) node);
    default:
        elog(ERROR, "unrecognized node type: %d", nodeTag(node));
}

// List iteration
foreach(lc, list)
{
    Node *item = lfirst(lc);
    // process item
}
```

## Part 6: Tracing Query Execution

### Enable Debug Output

**🔬 Try It:**
```sql
-- In postgresql.conf or session
SET debug_print_parse = on;
SET debug_print_rewritten = on;
SET debug_print_plan = on;
SET client_min_messages = 'debug1';

-- Run query and check server log
SELECT * FROM employees WHERE emp_id = 1;
```

### Using gdb

```bash
# Find postgres process
ps aux | grep postgres

# Attach debugger
gdb -p <pid>

# Set breakpoints
break exec_simple_query
break ExecScan

# Continue and execute query
continue
```

### Key Functions to Trace

```c
// Query lifecycle
exec_simple_query()     // Entry point
pg_parse_query()        // Parsing
pg_analyze_and_rewrite() // Analysis
pg_plan_queries()       // Planning
PortalRun()             // Execution

// Executor
ExecInitNode()          // Initialize plan node
ExecProcNode()          // Execute one tuple
ExecEndNode()           // Cleanup

// Access methods
heap_getnext()          // Get next tuple
index_getnext()         // Index scan
```

## Part 7: Modifying PostgreSQL

### Adding a Simple Function

```c
// src/backend/utils/adt/myfuncs.c
#include "postgres.h"
#include "fmgr.h"

PG_FUNCTION_INFO_V1(my_add_one);

Datum
my_add_one(PG_FUNCTION_ARGS)
{
    int32 arg = PG_GETARG_INT32(0);
    PG_RETURN_INT32(arg + 1);
}
```

**🔬 Try It:**
```sql
-- Register in system catalog
CREATE FUNCTION my_add_one(integer)
RETURNS integer
AS 'MODULE_PATHNAME'
LANGUAGE C IMMUTABLE STRICT;
```

### Adding a GUC Variable

```c
// src/backend/utils/misc/guc.c

// Define variable
static bool my_feature_enabled = false;

// Register in InitializeGUCOptions()
DefineCustomBoolVariable(
    "my_feature_enabled",
    "Enable my feature",
    NULL,
    &my_feature_enabled,
    false,
    PGC_USERSET,
    0,
    NULL, NULL, NULL
);
```

## Part 8: Testing

### Running Tests

```bash
# Regression tests
make check

# Specific test
make check TESTS=test_name

# Isolation tests
make isolation-check

# TAP tests
make prove_installcheck
```

### Writing Tests

**🔬 Try It:**
```sql
-- src/test/regress/sql/my_test.sql
-- Test my feature
SELECT my_add_one(5);  -- Should return 6
SELECT my_add_one(0);  -- Should return 1
SELECT my_add_one(-1); -- Should return 0
```

```
-- src/test/regress/expected/my_test.out
-- Expected output
 my_add_one 
------------
          6
(1 row)

 my_add_one 
------------
          1
(1 row)

 my_add_one 
------------
          0
(1 row)
```

## Part 9: Essential Files Reference

### Must-Know Files

| File | Purpose |
|------|---------|
| `src/backend/main/main.c` | Server entry point |
| `src/backend/tcop/postgres.c` | Query dispatcher |
| `src/backend/parser/gram.y` | SQL grammar |
| `src/backend/optimizer/plan/planner.c` | Query planner |
| `src/backend/executor/execMain.c` | Executor entry |
| `src/backend/storage/buffer/bufmgr.c` | Buffer manager |
| `src/backend/access/transam/xlog.c` | WAL implementation |
| `src/backend/postmaster/postmaster.c` | Process manager |

### Key Header Files

| File | Contains |
|------|----------|
| `src/include/postgres.h` | Main include |
| `src/include/nodes/nodes.h` | Node definitions |
| `src/include/storage/buf.h` | Buffer types |
| `src/include/access/htup_details.h` | Tuple structures |
| `src/include/catalog/pg_*.h` | System catalogs |
| `src/include/utils/palloc.h` | Memory allocation |

## Part 10: Development Setup

### Building from Source

```bash
# Configure with debug flags
./configure --enable-debug --enable-cassert CFLAGS="-O0 -g"

# Build
make -j4

# Install to custom location
make install DESTDIR=/opt/pg-dev

# Initialize test cluster
initdb -D /tmp/pgdata

# Start with debugging
postgres -D /tmp/pgdata &
```

### Useful Tools

```bash
# Code navigation
ctags -R src/
cscope -Rb

# Static analysis
make scan-build

# Memory checking
valgrind --leak-check=full postgres -D /tmp/pgdata
```

## Summary

In this chapter, you learned:

1. **Source Organization**: Directory structure and key files
2. **Entry Points**: main(), PostmasterMain(), PostgresMain()
3. **Query Path**: Parse → Analyze → Plan → Execute
4. **Data Structures**: Nodes, Lists, Memory Contexts
5. **Code Conventions**: Naming, error handling, patterns
6. **Tracing**: Debug output, gdb
7. **Modification**: Adding functions, GUCs
8. **Testing**: Regression tests
9. **Development Setup**: Build configuration

---

**Return to:** [Curriculum Index](README.md)

