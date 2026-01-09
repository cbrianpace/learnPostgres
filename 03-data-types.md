# Chapter 3: PostgreSQL Data Types

## Learning Objectives

By the end of this chapter, you will be able to:
- Use all standard PostgreSQL data types effectively
- Work with advanced types like arrays, JSON, and hstore
- Create custom types and domains
- Understand type coercion and casting
- Choose the right data type for each use case

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

## Why Data Types Matter

Choosing the right data type affects:
- **Storage efficiency**: Some types use less disk space
- **Query performance**: Proper types enable index usage and faster comparisons
- **Data integrity**: Types enforce valid values
- **Functionality**: Different types support different operations

## Part 1: Numeric Types

PostgreSQL offers several numeric types optimized for different use cases—from small counters to precise financial calculations.

### Integer Types

Integers store whole numbers without decimals. Choose the smallest type that fits your data to save storage and improve cache efficiency.

**How can 2 bytes store 32,767?**

Computers store numbers in binary (base-2). Each byte has 8 bits, and each bit can be 0 or 1. Here's how the number **123** looks in binary:

```
Decimal: 123
Binary:  0111 1011  (one byte = 8 bits)

Bit positions:  128  64  32  16   8   4   2   1
Bit values:       0   1   1   1   1   0   1   1
                      ↓   ↓   ↓   ↓       ↓   ↓
Calculation:     64 + 32 + 16 + 8     + 2 + 1 = 123
```

Each bit position represents a power of 2. Add up the positions where the bit is "1" to get the decimal value.

The number of unique values you can represent is 2^(number of bits):

| Storage | Bits | Unique Values | Signed Range |
|---------|------|---------------|--------------|
| 2 bytes | 16 bits | 2^16 = 65,536 | -32,768 to +32,767 |
| 4 bytes | 32 bits | 2^32 = ~4.3 billion | -2,147,483,648 to +2,147,483,647 |
| 8 bytes | 64 bits | 2^64 = ~18 quintillion | -9.2×10^18 to +9.2×10^18 |

For **signed** integers (which can be negative), one bit indicates the sign (+/-), so half the values are negative and half are positive. For example, 16 bits gives you 65,536 values: from -32,768 to +32,767 (note: includes zero, so positive side has one less).

**Unsigned integers** (PostgreSQL doesn't have these natively) could use all bits for magnitude: 0 to 65,535 for 16 bits.

| Type | Storage | Range | Use Case |
|------|---------|-------|----------|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Small enums, flags |
| `INTEGER` | 4 bytes | ±2.1 billion | Most IDs, counts |
| `BIGINT` | 8 bytes | ±9.2 quintillion | Large sequences, analytics |

`SERIAL` and `BIGSERIAL` are shorthand for auto-incrementing integers (they create a sequence behind the scenes).

**🔬 Try It:** Integer Types

```sql
-- Create table to explore integer types
CREATE TABLE integer_examples (
    tiny_val SMALLINT,      -- 2 bytes, -32768 to +32767
    normal_val INTEGER,     -- 4 bytes, -2147483648 to +2147483647
    big_val BIGINT,         -- 8 bytes, -9223372036854775808 to +9223372036854775807
    serial_val SERIAL,      -- Auto-incrementing INTEGER
    bigserial_val BIGSERIAL -- Auto-incrementing BIGINT
);

-- Insert examples
INSERT INTO integer_examples (tiny_val, normal_val, big_val) VALUES
    (100, 1000000, 9223372036854775807),
    (-32768, -2147483648, -9223372036854775808);

-- Check storage sizes
SELECT pg_column_size(tiny_val) as smallint_size,
       pg_column_size(normal_val) as int_size,
       pg_column_size(big_val) as bigint_size
FROM integer_examples LIMIT 1;
```

### Decimal Types

PostgreSQL has two categories of decimal numbers:

**Exact types (`NUMERIC`/`DECIMAL`)**: Store values precisely as specified. Use for money, financial data, or anywhere rounding errors are unacceptable. Slower but accurate.

**Approximate types (`REAL`/`DOUBLE PRECISION`)**: Store floating-point numbers using IEEE 754 format. Faster for scientific calculations but can have tiny rounding errors (e.g., 0.1 + 0.2 might equal 0.30000000000000004).

| Type | Storage | Precision | Use Case |
|------|---------|-----------|----------|
| `NUMERIC(p,s)` | Variable | Exact, user-defined | Money, precise calculations |
| `REAL` | 4 bytes | ~6 decimal digits | Scientific data |
| `DOUBLE PRECISION` | 8 bytes | ~15 decimal digits | Scientific data needing more range |

**🔬 Try It:** Decimal and Floating-Point Types

```sql
-- Exact precision types
CREATE TABLE decimal_examples (
    money_val NUMERIC(10,2),    -- Exact, 10 digits total, 2 after decimal
    precise_val DECIMAL(20,10), -- Same as NUMERIC
    float_val REAL,             -- 4 bytes, 6 decimal digits precision
    double_val DOUBLE PRECISION -- 8 bytes, 15 decimal digits precision
);

-- Demonstrate precision differences
INSERT INTO decimal_examples VALUES 
    (12345678.99, 1234567890.1234567890, 1.23456789, 1.234567890123456);

SELECT * FROM decimal_examples;

-- NUMERIC is exact, floating point is approximate
SELECT 0.1::NUMERIC + 0.2::NUMERIC AS numeric_result,
       0.1::REAL + 0.2::REAL AS float_result;
```

**When to use each:**
- `SMALLINT`: Enumeration-like values, small counts
- `INTEGER`: Most common choice for IDs and counts
- `BIGINT`: Very large numbers, large auto-increment sequences
- `NUMERIC`: Money, precise scientific calculations
- `REAL/DOUBLE`: Scientific data where some imprecision is acceptable

### Money Type

PostgreSQL's `MONEY` type stores currency values with fixed fractional precision based on the database's `lc_monetary` locale setting.

**Pros**: Convenient formatting, handles currency symbols  
**Cons**: Locale-dependent, difficult to change precision, awkward for multi-currency systems

**Recommendation**: Most applications should use `NUMERIC(12,2)` or `NUMERIC(19,4)` instead for better control and portability. Use `MONEY` only for simple, single-currency applications.

**🔬 Try It:** Money Type

```sql
-- PostgreSQL has a built-in money type
CREATE TABLE expense_reimbursements (
    id SERIAL PRIMARY KEY,
    emp_id INTEGER REFERENCES employees(emp_id),
    description VARCHAR(200),
    amount MONEY
);

INSERT INTO expense_reimbursements (emp_id, description, amount) VALUES
    ((SELECT MIN(emp_id) FROM employees), 'Home office equipment', '$199.99'),
    ((SELECT MIN(emp_id) FROM employees), 'Team lunch', 89.50);

SELECT description, amount, amount * 2 AS doubled FROM expense_reimbursements;

-- Convert to numeric for calculations
SELECT description, amount::NUMERIC AS amount_numeric FROM expense_reimbursements;
```

**Note:** Many prefer using `NUMERIC(10,2)` for money because of locale issues with the MONEY type.

## Part 2: Character Types

Text data is fundamental to most applications. PostgreSQL offers three character types with important tradeoffs.

### Character Type Comparison

| Type | Behavior | Storage | Use Case |
|------|----------|---------|----------|
| `CHAR(n)` | Fixed length, **space-padded** | Always n bytes | Legacy systems, fixed-format data |
| `VARCHAR(n)` | Variable length, max n | Actual length + overhead | When you need length validation |
| `TEXT` | Unlimited length | Actual length + overhead | Most common choice |

**Key insight**: In PostgreSQL, `TEXT` and `VARCHAR` perform identically. Unlike some databases, there's no performance penalty for using `TEXT`. The only reason to use `VARCHAR(n)` is if you want the database to enforce a maximum length.

**🔬 Try It:**
```sql
CREATE TABLE text_examples (
    fixed_char CHAR(10),        -- Fixed length, padded with spaces
    var_char VARCHAR(100),      -- Variable length with limit
    unlimited_text TEXT         -- Unlimited length
);

INSERT INTO text_examples VALUES 
    ('Hello', 'Hello World', 'This can be any length');

-- Try to see the padding (but there's a gotcha!)
SELECT fixed_char, 
       LENGTH(fixed_char) AS len,
       fixed_char || '|' AS with_trailing
FROM text_examples;
-- Result: len = 5, with_trailing = 'Hello|'
-- Wait, where's the padding?
```

**PostgreSQL CHAR Gotcha:** PostgreSQL aggressively strips trailing spaces from `CHAR` values in almost all operations. Even casting to `TEXT` doesn't preserve them!  This behavior will vary based on the client.  For example, the JDBC driver will preserve the spaces in the result set.

**🔬 Try It:**
```sql
-- Watch the spaces disappear
SELECT 
    fixed_char,
    LENGTH(fixed_char) AS len,                    -- 5 (stripped!)
    OCTET_LENGTH(fixed_char) AS bytes,            -- 10 (storage shows padding!)
    fixed_char || '|' AS concat,                  -- 'Hello|' (stripped!)
    fixed_char::TEXT || '|' AS cast_concat        -- 'Hello|' (still stripped!)
FROM text_examples;
```

The `OCTET_LENGTH` of 10 proves padding exists in storage, but PostgreSQL strips it on retrieval. To truly see the raw bytes:

**🔬 Try It:**
```sql
-- See the actual stored bytes (48656c6c6f = "Hello", 20 = space)
SELECT encode(fixed_char::bytea, 'hex') AS raw_bytes FROM text_examples;
-- Result: 48656c6c6f2020202020  (5 bytes for "Hello" + 5 space bytes 0x20)
```

**The moral:** `CHAR(n)` wastes storage on padding you can never actually use. This is why experienced PostgreSQL developers avoid `CHAR(n)` entirely—just use `TEXT` or `VARCHAR`.

**Bottom line:** `CHAR(n)` behavior in PostgreSQL is confusing and rarely useful. Just use `TEXT` or `VARCHAR`.  The exception being for columns that have exact length values and are short in size (one or two character values like state abbreviations).

**Best Practice:** In PostgreSQL, `TEXT` and `VARCHAR` have essentially identical performance. Use:
- `VARCHAR(n)` when you need to enforce a maximum length
- `TEXT` when there's no natural limit

### String Functions

PostgreSQL provides a rich library of functions for manipulating text. These are essential for data cleaning, formatting output, and text processing.

| Function | Purpose | Example |
|----------|---------|---------|
| `||` or `CONCAT()` | Join strings | `'a' || 'b'` → `'ab'` |
| `UPPER/LOWER/INITCAP` | Change case | `INITCAP('hello')` → `'Hello'` |
| `LENGTH` | Count characters | `LENGTH('hello')` → `5` |
| `SUBSTRING` | Extract portion | `SUBSTRING('hello' FROM 1 FOR 2)` → `'he'` |
| `POSITION` | Find substring | `POSITION('l' IN 'hello')` → `3` |
| `TRIM/LTRIM/RTRIM` | Remove whitespace | `TRIM('  hi  ')` → `'hi'` |

**🔬 Try It:**
```sql
-- Common string operations
SELECT 
    'hello' || ' ' || 'world' AS concatenation,
    CONCAT('hello', ' ', 'world') AS concat_func,
    UPPER('hello') AS upper_case,
    LOWER('HELLO') AS lower_case,
    INITCAP('hello world') AS title_case,
    LENGTH('hello') AS length,
    SUBSTRING('hello world' FROM 1 FOR 5) AS substring,
    POSITION('world' IN 'hello world') AS position,
    REPLACE('hello world', 'world', 'PostgreSQL') AS replaced,
    TRIM('  hello  ') AS trimmed,
    LPAD('42', 5, '0') AS left_padded,
    REVERSE('hello') AS reversed;
```

### Regular Expressions

PostgreSQL supports POSIX regular expressions for powerful pattern matching. Regex is useful for validation, extraction, and complex text transformations.

| Operator | Meaning | Case |
|----------|---------|------|
| `~` | Matches regex | Case-sensitive |
| `~*` | Matches regex | Case-insensitive |
| `!~` | Does not match | Case-sensitive |
| `!~*` | Does not match | Case-insensitive |

Key functions:
- `REGEXP_REPLACE(string, pattern, replacement, flags)` - Replace matches
- `REGEXP_MATCHES(string, pattern, flags)` - Return all matches as array
- `SUBSTRING(string FROM pattern)` - Extract first match

**🔬 Try It:**
```sql
-- PostgreSQL has powerful regex support
SELECT 
    'hello world' ~ 'world' AS matches,      -- POSIX regex match
    'Hello World' ~* 'world' AS matches_ci,  -- Case-insensitive
    'hello world' !~ 'foo' AS not_matches,
    REGEXP_REPLACE('hello 123 world 456', '\d+', '#', 'g') AS replaced,
    REGEXP_MATCHES('hello 123 world 456', '\d+', 'g') AS matches;

-- Extracting with regex
SELECT SUBSTRING('Product Code: ABC-123' FROM '[A-Z]+-\d+') AS extracted;
```

## Part 3: Date and Time Types

Date and time handling is notoriously tricky. PostgreSQL provides robust types and functions, but you need to understand timezone behavior to avoid subtle bugs.

### Date/Time Types Overview

| Type | Storage | Description | Example |
|------|---------|-------------|---------|
| `DATE` | 4 bytes | Calendar date only | `'2024-01-15'` |
| `TIME` | 8 bytes | Time of day only | `'14:30:00'` |
| `TIMESTAMP` | 8 bytes | Date + time, no timezone | `'2024-01-15 14:30:00'` |
| `TIMESTAMPTZ` | 8 bytes | Date + time, timezone-aware | `'2024-01-15 14:30:00-05'` |
| `INTERVAL` | 16 bytes | Time duration | `'2 hours 30 minutes'` |

**Critical distinction**: `TIMESTAMP` stores exactly what you give it. `TIMESTAMPTZ` converts to UTC for storage and back to your session timezone for display. For most applications, **use `TIMESTAMPTZ`** to avoid timezone confusion.

**🔬 Try It:**
```sql
CREATE TABLE datetime_examples (
    date_only DATE,                          -- 4 bytes, date without time
    time_only TIME,                          -- 8 bytes, time without date
    timestamp_local TIMESTAMP,               -- 8 bytes, without timezone
    timestamp_tz TIMESTAMPTZ,                -- 8 bytes, with timezone
    time_interval INTERVAL                   -- 16 bytes, time duration
);

INSERT INTO datetime_examples VALUES (
    '2024-01-15',
    '14:30:00',
    '2024-01-15 14:30:00',
    '2024-01-15 14:30:00-05',
    '2 hours 30 minutes'
);

SELECT * FROM datetime_examples;
```

### Working with Dates

PostgreSQL provides many functions for getting current date/time, doing arithmetic, extracting components, and formatting output.

**Getting current date/time:**
- `CURRENT_DATE`, `CURRENT_TIME`, `CURRENT_TIMESTAMP` - SQL standard
- `NOW()` - PostgreSQL alias for `CURRENT_TIMESTAMP`
- `LOCALTIMESTAMP` - Without timezone info

**Date arithmetic** is intuitive: add/subtract integers (days) or intervals.

**`EXTRACT(field FROM source)`** pulls out components: YEAR, MONTH, DAY, HOUR, DOW (day of week), EPOCH (Unix timestamp), etc.

**`TO_CHAR(timestamp, format)`** formats dates as strings for display.

**🔬 Try It:**
```sql
-- Current date/time functions
SELECT 
    CURRENT_DATE AS today,
    CURRENT_TIME AS time_now,
    CURRENT_TIMESTAMP AS timestamp_now,
    NOW() AS now_alias,
    LOCALTIME AS local_time,
    LOCALTIMESTAMP AS local_timestamp;

-- Date arithmetic
SELECT 
    CURRENT_DATE + 7 AS week_later,
    CURRENT_DATE - INTERVAL '30 days' AS month_ago,
    CURRENT_DATE + INTERVAL '1 year' AS next_year,
    AGE(CURRENT_DATE, '1990-01-15') AS age;

-- Extracting components
SELECT 
    EXTRACT(YEAR FROM NOW()) AS year,
    EXTRACT(MONTH FROM NOW()) AS month,
    EXTRACT(DAY FROM NOW()) AS day,
    EXTRACT(DOW FROM NOW()) AS day_of_week,
    EXTRACT(EPOCH FROM NOW()) AS unix_timestamp;

-- Formatting
SELECT 
    TO_CHAR(NOW(), 'YYYY-MM-DD') AS iso_date,
    TO_CHAR(NOW(), 'Month DD, YYYY') AS long_date,
    TO_CHAR(NOW(), 'HH24:MI:SS') AS time_24h,
    TO_CHAR(NOW(), 'Day') AS day_name;

-- Parsing strings to dates
SELECT 
    TO_DATE('01/15/2024', 'MM/DD/YYYY') AS parsed_date,
    TO_TIMESTAMP('2024-01-15 14:30', 'YYYY-MM-DD HH24:MI') AS parsed_ts;
```

### Intervals

`INTERVAL` represents a duration of time—useful for date arithmetic, scheduling, and calculating differences.

**Creating intervals:**
- String format: `INTERVAL '1 year 2 months 3 days 4 hours'`
- ISO 8601: `INTERVAL 'P1Y2M3DT4H'`
- Shorthand: `INTERVAL '1 week'`, `INTERVAL '30 days'`

**Interval arithmetic:**
- Add/subtract intervals from dates/timestamps
- Multiply intervals by numbers: `2 * INTERVAL '1 hour'`
- Subtract dates to get an integer (days) or use `AGE()` to get an interval

**🔬 Try It:**
```sql
-- Interval creation and arithmetic
SELECT 
    INTERVAL '1 year 2 months 3 days' AS complex_interval,
    INTERVAL '1 week' AS one_week,
    2 * INTERVAL '1 hour' AS two_hours,
    '2024-12-31'::DATE - '2024-01-01'::DATE AS days_between,
    AGE('2024-12-31'::DATE, '2024-01-01'::DATE) AS age_between;

-- Date ranges (useful for scheduling)
SELECT daterange('2024-01-01', '2024-01-31', '[]') AS january;
```

## Part 4: Boolean Type

**🔬 Try It:**
```sql
CREATE TABLE feature_flags (
    name VARCHAR(50) PRIMARY KEY,
    enabled BOOLEAN DEFAULT false,
    description TEXT
);

INSERT INTO feature_flags VALUES 
    ('dark_mode', true, 'Enable dark mode UI'),
    ('beta_features', false, 'Show beta features'),
    ('analytics', null, 'Track analytics');  -- NULL means unknown

-- Boolean operations
SELECT 
    name,
    enabled,
    NOT enabled AS inverted,
    enabled IS TRUE AS explicitly_true,
    enabled IS NOT FALSE AS not_false,  -- TRUE or NULL
    enabled IS UNKNOWN AS is_null
FROM feature_flags;

-- Boolean in WHERE (implicit IS TRUE)
SELECT * FROM feature_flags WHERE enabled;
SELECT * FROM feature_flags WHERE NOT enabled;  -- Only returns false, not NULL
SELECT * FROM feature_flags WHERE enabled IS NOT TRUE;  -- FALSE and NULL
```

## Part 5: Binary Data

The `BYTEA` type stores binary data—raw bytes that aren't interpreted as text. This is essential for storing files, images, encrypted data, hashes, or any non-textual content.

### When to Use BYTEA

| Use Case | Why BYTEA |
|----------|-----------|
| **File storage** | Store PDFs, images, documents directly in the database |
| **Cryptographic hashes** | SHA-256, MD5 checksums are binary, not text |
| **Encrypted data** | Ciphertext is binary and may contain any byte value |
| **Serialized objects** | Protocol buffers, MessagePack, etc. |
| **Small binary blobs** | Thumbnails, icons, certificates |

**⚠️ Large files:** For files over ~1MB, consider PostgreSQL's Large Objects (`lo_*` functions) or external storage (S3, filesystem) with a path/URL in the database. BYTEA works but can impact performance.

### BYTEA vs TEXT

**🔬 Try It:**
```sql
-- TEXT interprets bytes as characters (may fail on invalid UTF-8)
SELECT 'Hello'::TEXT;   -- Stored as UTF-8 text

-- BYTEA stores raw bytes (any byte value 0-255 is valid)
SELECT 'Hello'::BYTEA;  -- Stored as raw bytes: \x48656c6c6f
```

### Input Formats

PostgreSQL accepts binary data in two formats:

**Hex format (recommended):** Prefix with `\x` followed by hexadecimal pairs:

**🔬 Try It:**
```sql
SELECT '\x48656c6c6f'::BYTEA;  -- "Hello" in hex (48=H, 65=e, 6c=l, 6f=o)
```

**Escape format:** Traditional format using `\\` for backslash and `\nnn` for octal:

**🔬 Try It:**
```sql
SELECT E'\\x48656c6c6f'::BYTEA;  -- Same thing, escape syntax
```

### Working with BYTEA

**🔬 Try It:**
```sql
-- Create a table for file storage
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    content BYTEA,
    checksum BYTEA
);

-- Insert binary data (text is automatically converted to bytes)
INSERT INTO files (name, content, checksum) VALUES 
    ('test.txt', 'Hello World'::BYTEA, sha256('Hello World'::BYTEA));

-- Insert using hex format
INSERT INTO files (name, content) VALUES 
    ('hex_example', '\x48656c6c6f'::BYTEA);  -- "Hello" in hex
```

### Essential BYTEA Functions

| Function | Description | Example |
|----------|-------------|---------|
| `encode(bytea, format)` | Convert binary to text | `encode('\x48656c6c6f', 'base64')` → `'SGVsbG8='` |
| `decode(text, format)` | Convert text to binary | `decode('SGVsbG8=', 'base64')` → `\x48656c6c6f` |
| `length(bytea)` | Number of bytes | `length('\x48656c6c6f')` → `5` |
| `sha256(bytea)` | SHA-256 hash | `sha256('password'::bytea)` |
| `md5(bytea)` | MD5 hash (text result) | `md5('test'::bytea)` |
| `substring(bytea, start, len)` | Extract bytes | `substring('\x48656c6c6f', 1, 2)` → `\x4865` |
| `\|\|` | Concatenate binary | `'\x48'::bytea \|\| '\x65'::bytea` → `\x4865` |

### Encoding Formats

The `encode()` and `decode()` functions support several formats:

**🔬 Try It:**
```sql
SELECT 
    content,
    encode(content, 'hex') AS hex,        -- Hexadecimal
    encode(content, 'base64') AS base64,  -- Base64 (compact for transport)
    encode(content, 'escape') AS escaped  -- PostgreSQL escape format
FROM files WHERE name = 'test.txt';
```

**Output:**
```
    content     |        hex         |    base64     |   escaped   
----------------+--------------------+---------------+-------------
 \x48656c6c6f...| 48656c6c6f576f726c64| SGVsbG8gV29ybGQ= | Hello World
```

### Practical Example: Storing and Verifying File Checksums

**🔬 Try It:**
```sql
-- Store a file with its SHA-256 checksum
INSERT INTO files (name, content, checksum)
VALUES (
    'config.json',
    '{"setting": "value"}'::BYTEA,
    sha256('{"setting": "value"}'::BYTEA)
);

-- Verify file integrity
SELECT 
    name,
    CASE 
        WHEN sha256(content) = checksum THEN 'VALID'
        ELSE 'CORRUPTED!'
    END AS integrity_check
FROM files;

-- Display checksum as hex string
SELECT name, encode(checksum, 'hex') AS sha256_hex FROM files;
```

## Part 6: Network Types

PostgreSQL has native network address types:

**🔬 Try It:**
```sql
CREATE TABLE network_config (
    id SERIAL PRIMARY KEY,
    ip_address INET,          -- IP address with optional netmask
    network CIDR,             -- Network specification
    mac_address MACADDR       -- MAC address
);

INSERT INTO network_config (ip_address, network, mac_address) VALUES 
    ('192.168.1.100', '192.168.1.0/24', '08:00:2b:01:02:03'),
    ('10.0.0.1/8', '10.0.0.0/8', '08:00:2b:01:02:04'),
    ('2001:db8::1', '2001:db8::/32', '08:00:2b:01:02:05');

-- Network functions
SELECT 
    ip_address,
    network,
    host(ip_address) AS host_only,
    netmask(network) AS netmask,
    broadcast(network) AS broadcast,
    network_with_mask AS network_string,
    ip_address << '192.168.0.0/16'::INET AS in_private_range
FROM (SELECT ip_address, network, network::TEXT AS network_with_mask 
      FROM network_config) sub;

-- Find IPs in a network
SELECT * FROM network_config
WHERE ip_address << '192.168.0.0/16'::INET;
```

## Part 7: UUID Type

UUIDs (Universally Unique Identifiers) are 128-bit values useful for distributed systems where you can't rely on a central sequence generator.

**PostgreSQL 13+ has built-in UUID generation**—no extension needed!

**🔬 Try It:**
```sql
-- Modern approach: use built-in gen_random_uuid() (PostgreSQL 13+)
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO sessions (user_id) VALUES (1), (2), (3);

SELECT * FROM sessions;

-- Generate UUIDs directly
SELECT gen_random_uuid() AS random_uuid;
```

**Legacy approach (requires uuid-ossp extension):**

If you need other UUID versions (v1 time-based, etc.) or are on PostgreSQL < 13, you need the `uuid-ossp` extension. This requires compiling PostgreSQL with `--with-uuid=e2fsprogs` or `--with-uuid=ossp` and having the uuid library installed.

**🔬 Try It:**
```sql
-- Only if you need uuid-ossp features (and compiled with uuid support)
   CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
   SELECT uuid_generate_v1() AS v1_time_based;   -- Time + MAC based
   SELECT uuid_generate_v4() AS v4_random;       -- Same as gen_random_uuid()
```

| UUID Version | Method | Use Case |
|--------------|--------|----------|
| v4 (random) | `gen_random_uuid()` | Most common, no extension needed |
| v1 (time-based) | `uuid_generate_v1()` | Sortable by time, requires extension |
| v7 (time-ordered) | Not built-in yet | Modern sortable alternative (use extension or app-side) |

## Part 8: Arrays

PostgreSQL supports multi-dimensional arrays:

**🔬 Try It:**
```sql
CREATE TABLE products_v2 (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[],              -- Array of text
    prices NUMERIC(10,2)[],   -- Array of prices (different regions)
    matrix INTEGER[][]        -- 2D array
);

INSERT INTO products_v2 (name, tags, prices, matrix) VALUES 
    ('Widget', ARRAY['electronic', 'gadget', 'popular'], 
     ARRAY[19.99, 24.99, 29.99],
     ARRAY[[1,2],[3,4]]),
    ('Gadget', '{tool,handy}', '{9.99,14.99}', '{{5,6},{7,8}}');

-- Array access
SELECT 
    name,
    tags[1] AS first_tag,           -- 1-indexed!
    prices[2:3] AS price_range,     -- Slice
    array_length(tags, 1) AS num_tags
FROM products_v2;

-- Array searching
SELECT * FROM products_v2 WHERE 'popular' = ANY(tags);
SELECT * FROM products_v2 WHERE tags @> ARRAY['electronic']; -- Contains

-- Array functions
SELECT 
    array_append(ARRAY[1,2], 3) AS appended,
    array_prepend(0, ARRAY[1,2]) AS prepended,
    array_cat(ARRAY[1,2], ARRAY[3,4]) AS concatenated,
    array_remove(ARRAY[1,2,3,2], 2) AS removed,
    array_position(ARRAY['a','b','c'], 'b') AS position,
    unnest(ARRAY['a','b','c']) AS unnested;

-- Aggregate into arrays
SELECT dept_id, array_agg(first_name ORDER BY first_name) AS employees
FROM employees
GROUP BY dept_id;
```

**Exercise 3.1: Array Operations**

**🔬 Try It:**
```sql
-- Create a table of blog posts with tags
CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    tags TEXT[]
);

INSERT INTO blog_posts (title, tags) VALUES
    ('Getting Started with PostgreSQL', ARRAY['postgresql', 'database', 'tutorial']),
    ('Advanced SQL Techniques', ARRAY['sql', 'database', 'advanced']),
    ('Python Web Development', ARRAY['python', 'web', 'tutorial']);

-- Find posts with specific tags
SELECT * FROM blog_posts WHERE 'tutorial' = ANY(tags);

-- Find posts with ALL of these tags
SELECT * FROM blog_posts WHERE tags @> ARRAY['database', 'tutorial'];

-- Find posts with ANY of these tags  
SELECT * FROM blog_posts WHERE tags && ARRAY['python', 'postgresql'];

-- Count tags per post
SELECT title, array_length(tags, 1) AS tag_count FROM blog_posts;

-- Unnest and count tag frequency
SELECT tag, COUNT(*) 
FROM blog_posts, unnest(tags) AS tag
GROUP BY tag
ORDER BY count DESC;
```

## Part 9: JSON and JSONB

PostgreSQL has excellent JSON support with two data types: `JSON` and `JSONB`. Understanding the difference is critical for performance.

### JSON vs JSONB: What's the Difference?

| Aspect | `JSON` | `JSONB` |
|--------|--------|---------|
| **Storage** | Stores exact text (including whitespace) | Parsed binary format |
| **Speed on insert** | Faster (no parsing) | Slightly slower (must parse) |
| **Speed on read/query** | Slower (reparsed every access) | Much faster (already parsed) |
| **Indexing** | ❌ Cannot be indexed | ✅ GIN indexes supported |
| **Operators** | Basic (`->`, `->>`) | Full set (`@>`, `?`, `?&`, `?|`) |
| **Preserves order** | ✅ Key order preserved | ❌ Keys may be reordered |
| **Preserves duplicates** | ✅ Duplicate keys kept | ❌ Last value wins |
| **Preserves whitespace** | ✅ Exact formatting | ❌ Normalized |

**When to use each:**

- **Use `JSONB` (almost always):** Faster queries, supports indexing, smaller storage for most data
- **Use `JSON` (rare):** Only when you must preserve exact formatting, key order, or duplicate keys (e.g., logging raw API responses for debugging)

### Practical Example

**🔬 Try It:**
```sql
-- Same data, different types
SELECT '{"b": 2,  "a": 1}'::JSON;   -- Keeps: {"b": 2,  "a": 1} (spaces, order)
SELECT '{"b": 2,  "a": 1}'::JSONB;  -- Returns: {"a": 1, "b": 2} (sorted, normalized)

-- Duplicate keys
SELECT '{"x": 1, "x": 2}'::JSON;    -- Keeps both (but accessing is unpredictable)
SELECT '{"x": 1, "x": 2}'::JSONB;   -- Returns: {"x": 2} (last value wins)
```

### Creating a JSON Table

**Recommendation:** Use `JSONB` for both columns unless you have a specific reason for `JSON`:

**🔬 Try It:**
```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_time TIMESTAMPTZ DEFAULT NOW(),
    data JSONB,        -- Use JSONB for queryable data
    metadata JSONB     -- Binary format, faster queries, indexable
);

INSERT INTO events (data, metadata) VALUES 
    ('{"type": "click", "x": 100, "y": 200}',
     '{"user_id": 1, "session": "abc123", "tags": ["important", "reviewed"]}'),
    ('{"type": "scroll", "direction": "down", "amount": 500}',
     '{"user_id": 2, "session": "def456", "tags": ["spam"]}');

-- JSON operators
SELECT 
    data->>'type' AS event_type,           -- Get as text
    data->'x' AS x_value,                  -- Get as JSON
    metadata->'user_id' AS user_id,
    metadata->>'session' AS session,
    metadata->'tags'->0 AS first_tag,      -- Array access
    metadata#>'{tags,0}' AS path_access    -- Path access
FROM events;

-- JSONB specific operators
SELECT * FROM events 
WHERE metadata @> '{"user_id": 1}'::JSONB;  -- Contains

SELECT * FROM events
WHERE metadata ? 'session';  -- Has key

SELECT * FROM events
WHERE metadata ?& ARRAY['user_id', 'session'];  -- Has all keys

-- JSON functions
SELECT 
    jsonb_pretty(metadata) AS formatted,
    jsonb_typeof(metadata->'user_id') AS type,
    jsonb_array_length(metadata->'tags') AS num_tags,
    jsonb_object_keys(metadata) AS keys
FROM events;

-- Build JSON
SELECT jsonb_build_object(
    'name', first_name,
    'email', email,
    'hire_year', EXTRACT(YEAR FROM hire_date)
) AS employee_json
FROM employees LIMIT 3;

-- Aggregate to JSON
SELECT jsonb_agg(jsonb_build_object('name', first_name, 'salary', salary))
FROM employees;
```

### JSONB Modification

**🔬 Try It:**
```sql
-- Update JSONB values
UPDATE events 
SET metadata = metadata || '{"reviewed": true}'::JSONB
WHERE id = 1;

-- Remove a key
UPDATE events
SET metadata = metadata - 'reviewed'
WHERE id = 1;

-- Set nested value
UPDATE events
SET metadata = jsonb_set(metadata, '{priority}', '"high"')
WHERE id = 1;

SELECT metadata FROM events WHERE id = 1;
```

### JSONB Indexing

JSONB columns can be indexed using GIN (Generalized Inverted Index) for fast containment queries. We'll cover this in detail in **Chapter 7: Indexing**, but here's a preview:

This preview is intentionally non-executable: we don’t want to create indexes yet, because Chapter 7 is where we build enough data volume to make indexing trade-offs visible and measurable.

**📖 Example:** What JSONB indexing often looks like

> ```sql
> CREATE INDEX idx_events_metadata ON events USING gin (metadata);
> SELECT * FROM events WHERE metadata @> '{"user_id": 1}'::jsonb;
> ```

For now, our `events` table is small enough that sequential scans are fast. As the company grows and we accumulate more event data, indexing will become essential.

## Part 10: Composite Types

**Composite types** let you define custom structured types—like a `struct` in C or a class in object-oriented languages. They group related fields into a single reusable type.

**When to use composite types:**

| Approach | Best For | Trade-offs |
|----------|----------|------------|
| Composite Type | Repeated structures (address, coordinates), type safety | Rigid schema, harder to query/index individual fields |
| JSONB | Flexible/dynamic data, varying fields | Less type safety, larger storage |
| Separate Columns | Simple cases, frequently queried fields | More columns, repetitive for repeated structures |

**Example use case:** A customer has both billing and shipping addresses. Instead of 8 separate columns (`billing_street`, `billing_city`, `shipping_street`, etc.), define an `address` type once and use it twice:

**🔬 Try It:**
```sql
-- Create a composite type (like defining a struct)
CREATE TYPE address AS (
    street VARCHAR(100),
    city VARCHAR(50),
    state CHAR(2),
    zip VARCHAR(10)
);

-- Use the type in a table (one column holds all 4 address fields)
CREATE TABLE employee_profiles (
    emp_id INTEGER PRIMARY KEY REFERENCES employees(emp_id),
    home_address address,      -- One composite column
    mailing_address address    -- Another composite column
);

-- Insert using ROW() constructor
INSERT INTO employee_profiles (emp_id, home_address, mailing_address) VALUES
    ((SELECT MIN(emp_id) FROM employees),
     ROW('123 Main St', 'Springfield', 'IL', '62701'),
     ROW('456 Oak Ave', 'Chicago', 'IL', '60601'));
```

### Accessing Composite Fields

**Important syntax quirk:** You must wrap the column name in parentheses when accessing fields, otherwise PostgreSQL thinks you're referencing a table:

**🔬 Try It:**
```sql
-- Access individual fields (parentheses required!)
SELECT 
    emp_id,
    (home_address).street AS home_street,      -- ✓ Correct
    (home_address).city AS home_city,          -- ✓ Correct
    -- home_address.street                      -- ✗ Would fail!
    (mailing_address).*                        -- Expand all fields
FROM employee_profiles;
```

Why parentheses? Without them, PostgreSQL parses "billing_address.street" as "table_name.column_name", not "column_name.field_name"


### Updating Composite Fields

**🔬 Try It:**
```sql
-- Update a single field within the composite
UPDATE employee_profiles
SET home_address.zip = '62702'
WHERE emp_id = (SELECT MIN(emp_id) FROM employees);

-- Update the entire composite value
UPDATE employee_profiles
SET home_address = ROW('789 New St', 'Boston', 'MA', '02101')
WHERE emp_id = (SELECT MIN(emp_id) FROM employees);
```

### Composite Types vs JSONB

**📖 Example:** Type checking differences:

> ```sql
> -- Composite: strict schema, type-checked at insert time
> INSERT INTO employee_profiles (emp_id, home_address, mailing_address)
> VALUES (1, ROW(123, 'City', 'XX', '00000'), ...);
> -- ✗ Error: 123 is not VARCHAR
> 
> -- JSONB: flexible, errors only when you query wrong types
> -- No error at insert, but may fail later when processing
> ```

**Recommendation:** Use composite types when the structure is stable and you want compile-time type checking. Use JSONB when structure varies or you need flexibility.

## Part 11: Range Types

**Range types** represent a range of values with a single column instead of two separate columns (`start_date`, `end_date`). This is powerful because PostgreSQL provides built-in operators for overlap detection, containment, and more.

**Why use ranges instead of two columns?**

| Approach | Overlap Check | Constraint |
|----------|---------------|------------|
| Two columns (`start_time`, `end_time`) | `a.start < b.end AND a.end > b.start` (easy to get wrong!) | Complex trigger needed |
| Range type (`TSTZRANGE`) | `a.time_range && b.time_range` (simple!) | Built-in exclusion constraint |

**Built-in Range Types:**

| Type | Contains | Example |
|------|----------|---------|
| `INT4RANGE` | integers | `[1, 10)` |
| `INT8RANGE` | bigints | `[1, 1000000)` |
| `NUMRANGE` | numeric/decimal | `[1.5, 9.9]` |
| `TSRANGE` | timestamp without tz | `[2024-01-01, 2024-12-31)` |
| `TSTZRANGE` | timestamp with tz | `[2024-01-01 00:00+00, 2024-01-02 00:00+00)` |
| `DATERANGE` | dates | `[2024-01-01, 2024-01-31]` |

**Bound Notation (like math intervals):**

| Notation | Meaning | Example |
|----------|---------|---------|
| `[` | Inclusive (includes this value) | `[1, 10]` includes 1 and 10 |
| `(` | Exclusive (excludes this value) | `(1, 10)` excludes 1 and 10 |
| `[1, 10)` | Half-open (common for time) | Includes 1, excludes 10 |

**Half-open ranges `[start, end)` are recommended for time** because they don't overlap at boundaries: `[9:00, 12:00)` and `[12:00, 15:00)` are adjacent, not overlapping.

**🔬 Try It:**
```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    room_name VARCHAR(50),
    reserved_during TSTZRANGE,  -- Timestamp range with timezone
    price_range NUMRANGE        -- Numeric range
);

INSERT INTO reservations (room_name, reserved_during, price_range) VALUES 
    ('Conference A', 
     '[2024-01-15 09:00, 2024-01-15 12:00)',  -- Includes start, excludes end
     '[100, 500]'),                            -- Includes both bounds
    ('Conference B',
     '[2024-01-15 14:00, 2024-01-15 17:00)',
     '(50, 200)');                             -- Excludes both bounds
```

### Range Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `@>` | Contains element | `range @> value` |
| `<@` | Contained by | `value <@ range` |
| `&&` | Overlaps | `range1 && range2` |
| `-\|-` | Adjacent | `range1 -\|- range2` |
| `+` | Union | `range1 + range2` |
| `*` | Intersection | `range1 * range2` |
| `-` | Difference | `range1 - range2` |

**🔬 Try It:**
```sql
-- Range operators in action
SELECT 
    room_name,
    lower(reserved_during) AS start_time,
    upper(reserved_during) AS end_time,
    reserved_during @> '2024-01-15 10:00'::TIMESTAMPTZ AS contains_10am,
    reserved_during && '[2024-01-15 11:00, 2024-01-15 15:00)'::TSTZRANGE AS overlaps
FROM reservations;
```

### Exclusion Constraints (Preventing Overlaps)

**Exclusion constraints** are like unique constraints on steroids. While a unique constraint prevents duplicate values, an exclusion constraint prevents rows that "conflict" based on any operators you define—like overlapping ranges.

Think of it this way:
- **UNIQUE constraint:** "No two rows can have the same value"
- **Exclusion constraint:** "No two rows can overlap/conflict according to these rules"

This is perfect for scheduling systems where you need to prevent double-booking.

**🔬 Try It:**
```sql
-- Prevent overlapping reservations with exclusion constraint
-- First, enable btree_gist extension (required for using = with GiST on VARCHAR)
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- GiST natively supports range overlap (&&), but NOT scalar equality (=)
-- btree_gist adds GiST operator classes for text, varchar, int, etc.
ALTER TABLE reservations 
ADD CONSTRAINT no_overlap 
EXCLUDE USING GIST (room_name WITH =, reserved_during WITH &&);

-- Now overlapping reservations for the SAME room are prevented:
-- This will fail due to overlap with existing Conference A reservation:
INSERT INTO reservations (room_name, reserved_during, price_range) VALUES 
    ('Conference A', '[2024-01-15 11:00, 2024-01-15 13:00)', '[100,200]');

-- But different rooms can overlap (this would succeed):
INSERT INTO reservations (room_name, reserved_during, price_range) VALUES 
    ('Conference C', '[2024-01-15 10:00, 2024-01-15 14:00)', '[150,300]');
```

## Part 12: Enumerated Types

**Enumerated types (ENUMs)** restrict a column to a fixed set of allowed values. Think of them as a dropdown menu for your data—only the listed options are valid.

### When to Use ENUMs

| Use Case | Example |
|----------|---------|
| Status fields | `'pending'`, `'approved'`, `'rejected'` |
| Categories with fixed options | `'small'`, `'medium'`, `'large'` |
| Days, priorities, ratings | `'low'`, `'medium'`, `'high'`, `'critical'` |

### ENUM vs VARCHAR with CHECK vs Lookup Table

| Approach | Pros | Cons |
|----------|------|------|
| **ENUM** | Type-safe, compact storage (4 bytes), fast | Hard to modify, values embedded in schema |
| **VARCHAR + CHECK** | Easy to modify constraint | No type safety, larger storage |
| **Lookup table** | Most flexible, can add metadata | Requires JOIN, more complex |

**Rule of thumb:** Use ENUMs for stable, rarely-changing value sets. Use a lookup table if values might change frequently or need additional attributes (descriptions, sort order, etc.).

### Creating and Using ENUMs

**🔬 Try It:**
```sql
-- Create an enum type
CREATE TYPE mood AS ENUM ('sad', 'neutral', 'happy', 'ecstatic');

CREATE TABLE mood_log (
    id SERIAL PRIMARY KEY,
    logged_at TIMESTAMPTZ DEFAULT NOW(),
    feeling mood
);

INSERT INTO mood_log (feeling) VALUES ('happy'), ('neutral'), ('ecstatic');

-- Enums are ordered
SELECT * FROM mood_log WHERE feeling > 'neutral';

-- Add a new value
ALTER TYPE mood ADD VALUE 'anxious' BEFORE 'neutral';
```

## Part 13: Domains

**Domains** are user-defined data types built on top of existing types, with optional constraints attached. Think of them as "named, reusable column definitions" that enforce validation rules consistently across your database.

### Why Use Domains?

Without domains, you'd repeat the same validation logic everywhere:

> ```sql
> -- Without domains: repetitive, error-prone
> CREATE TABLE users (
>     email VARCHAR(255) CHECK (email ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
> );
> CREATE TABLE contacts (
>     email VARCHAR(255) CHECK (email ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')  -- Duplicated!
> );
> ```

With domains, define once, use everywhere:

> ```sql
> -- With domains: DRY (Don't Repeat Yourself)
> CREATE DOMAIN email AS VARCHAR(255)
>     CHECK (VALUE ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');
> 
> CREATE TABLE domain_example1 (email email);      -- Clean!
> CREATE TABLE domain_example2 (email email);   -- Consistent!
> ```

### Domain vs CHECK Constraint

| Aspect | Domain | CHECK Constraint |
|--------|--------|------------------|
| **Reusability** | ✅ Define once, use in many tables | ❌ Must repeat in each table |
| **Consistency** | ✅ Guaranteed same rules everywhere | ⚠️ Risk of copy-paste errors |
| **Modification** | ⚠️ Can't easily change (must drop/recreate) | ✅ ALTER TABLE to change |
| **Documentation** | ✅ Self-documenting type name | ❌ Constraint logic scattered |

### Creating and Using Domains

**🔬 Try It:**
```sql
-- Create domains for common validation patterns
CREATE DOMAIN email AS VARCHAR(255)
    CHECK (VALUE ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');

CREATE DOMAIN positive_integer AS INTEGER
    CHECK (VALUE > 0);

CREATE DOMAIN us_zip AS VARCHAR(10)
    CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

-- Domains can have NOT NULL and DEFAULT too
CREATE DOMAIN percentage AS NUMERIC(5,2)
    CHECK (VALUE >= 0 AND VALUE <= 100)
    DEFAULT 0
    NOT NULL;

-- Use domains in tables (just like any data type)
CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    contact_email email,
    priority positive_integer,
    postal_code us_zip
);

-- Valid inserts work
INSERT INTO contacts VALUES (1, 'test@example.com', 5, '12345');
INSERT INTO contacts VALUES (2, 'other@test.org', 1, '12345-6789');
```

**📖 Example:** Invalid inserts fail with clear error messages:

> ```sql
> INSERT INTO contacts VALUES (3, 'invalid-email', 1, '12345');
> --   ERROR: value for domain email violates check constraint "email_check"
> 
> INSERT INTO contacts VALUES (4, 'test@example.com', -1, '12345');
> --   ERROR: value for domain positive_integer violates check constraint
> 
> INSERT INTO contacts VALUES (5, 'test@example.com', 1, '1234');
> --   ERROR: value for domain us_zip violates check constraint
> ```

### Common Domain Patterns

**🔬 Try It:**
```sql
-- Money that can't be negative
CREATE DOMAIN money_positive AS NUMERIC(10,2) CHECK (VALUE >= 0);

-- Percentage (0-100)
CREATE DOMAIN percentage AS NUMERIC(5,2) CHECK (VALUE BETWEEN 0 AND 100);

-- Non-empty text
CREATE DOMAIN nonempty_text AS TEXT CHECK (LENGTH(TRIM(VALUE)) > 0);

-- URL pattern
CREATE DOMAIN url AS TEXT CHECK (VALUE ~ '^https?://');
```

## Part 14: Type Casting and Coercion

**Type casting** converts a value from one data type to another. PostgreSQL needs this because operations often require specific types—you can't add a string to a number without converting it first.

### Explicit vs Implicit Casting

| Type | Description | Example |
|------|-------------|---------|
| **Explicit** | You request the conversion | `'42'::INTEGER` or `CAST('42' AS INTEGER)` |
| **Implicit** | PostgreSQL converts automatically | `'10' + 5` → PostgreSQL casts '10' to integer |

**Explicit casting** is preferred because it's clear and won't surprise you. Implicit casting can cause unexpected behavior or hide bugs.

### Two Syntaxes for Explicit Casting

**🔬 Try It:**
```sql
-- PostgreSQL-style (shorter, commonly used)
SELECT '42'::INTEGER;
SELECT NOW()::DATE;

-- SQL-standard CAST function (more portable)
SELECT CAST('42' AS INTEGER);
SELECT CAST(NOW() AS DATE);
```

Both produce identical results. Use `::` for brevity in PostgreSQL-specific code, `CAST()` for portability.

### Common Type Conversions

**🔬 Try It:**
```sql
SELECT 
    -- Text to numbers
    '42'::INTEGER AS text_to_int,
    '3.14'::NUMERIC AS text_to_numeric,
    
    -- Numbers to text
    42::TEXT AS int_to_text,
    3.14159::TEXT AS float_to_text,
    
    -- Boolean conversions
    'true'::BOOLEAN AS text_to_bool,
    1::BOOLEAN AS int_to_bool,        -- 1 = true, 0 = false
    
    -- Numeric precision changes (may truncate!)
    3.14::INTEGER AS float_to_int,    -- Result: 3 (truncated, not rounded)
    3.99::INTEGER AS also_truncates,  -- Result: 3
    
    -- Date/time conversions
    '2024-01-15'::DATE AS text_to_date,
    NOW()::DATE AS timestamp_to_date, -- Strips time portion
    NOW()::TIME AS timestamp_to_time; -- Strips date portion
```

### Implicit Coercion (Automatic Casting)

PostgreSQL automatically converts types in certain situations:

**🔬 Try It:**
```sql
-- Arithmetic with mixed types
SELECT '10' + 5;              -- '10' implicitly cast to INTEGER → 15
SELECT 10 + 2.5;              -- 10 implicitly cast to NUMERIC → 12.5

-- Comparison with different types  
SELECT * FROM employees WHERE emp_id = '1';  -- '1' cast to INTEGER
```

### ⚠️ Type Casting and Index Performance

**This is critical for query performance!** When you cast a column in a WHERE clause, PostgreSQL often **cannot use an index** on that column.

**🔬 Try It:**
```sql
-- Setup: table with an indexed INTEGER column
CREATE TABLE payroll_runs (
    id SERIAL PRIMARY KEY,
    run_number INTEGER,
    created_at TIMESTAMPTZ
);
CREATE INDEX idx_payroll_run_number ON payroll_runs(run_number);
CREATE INDEX idx_payroll_runs_created_at ON payroll_runs(created_at);

-- Insert sample data
INSERT INTO payroll_runs (run_number, created_at)
SELECT i, NOW() - (i || ' days')::INTERVAL 
FROM generate_series(1, 100000) i;

ANALYZE payroll_runs;
```

**❌ BAD: Casting the column prevents index usage**

**🔬 Try It:**
```sql
-- This forces a full table scan!
EXPLAIN ANALYZE 
SELECT * FROM payroll_runs WHERE run_number::TEXT = '12345';
--                         ^^^^^^^^^^^^^^^^^^^
-- The index on order_number CANNOT be used because we're 
-- casting every row's value to TEXT before comparing

-- Same problem with dates - casting the column
EXPLAIN ANALYZE
SELECT * FROM payroll_runs WHERE created_at::DATE = '2024-01-15';
--                         ^^^^^^^^^^^^^^^
-- Index on created_at cannot be used
```

**✅ GOOD: Cast the constant instead, or use proper types**

**🔬 Try It:**
```sql
-- Cast the search value, not the column
EXPLAIN ANALYZE 
SELECT * FROM payroll_runs WHERE run_number = '12345'::INTEGER;
--                                        ^^^^^^^^^^^^^^^^
-- Index CAN be used - we're comparing integers to integers

-- For dates, use a range instead of casting
EXPLAIN ANALYZE
SELECT * FROM payroll_runs
WHERE created_at >= '2024-01-15'::DATE 
  AND created_at < '2024-01-16'::DATE;
-- Index CAN be used - no function applied to the column
```

**Why does this happen?**

When you write `column::SOMETYPE = value`, PostgreSQL must:
1. Read every row from the table
2. Cast each row's column value
3. Then compare

This defeats the purpose of an index. But when you write `column = value::SOMETYPE`, PostgreSQL:
1. Casts the constant once
2. Uses the index to find matching rows directly

**Rule of thumb:** Never cast or apply functions to indexed columns in WHERE clauses. Cast the comparison value instead.

## Source Code Connection

Data types are defined in `src/include/catalog/pg_type.dat` and implemented throughout:

- **Type definitions**: `src/include/catalog/pg_type.h`
- **Array support**: `src/backend/utils/adt/arrayfuncs.c`
- **JSON/JSONB**: `src/backend/utils/adt/json.c`, `jsonb.c`
- **Date/Time**: `src/backend/utils/adt/datetime.c`, `timestamp.c`
- **Network types**: `src/backend/utils/adt/network.c`
- **Type coercion**: `src/backend/parser/parse_coerce.c`

Explore the type OIDs:

**🔬 Try It:**
```sql
SELECT oid, typname, typlen, typtype 
FROM pg_type 
WHERE typtype = 'b'  -- Base types
ORDER BY oid 
LIMIT 20;
```

## Summary

In this chapter, you learned:

1. **Numeric types**: INTEGER, NUMERIC, REAL for different precision needs
2. **Text types**: VARCHAR, TEXT, and string functions
3. **Date/Time**: DATE, TIMESTAMP, INTERVAL with rich function support
4. **Boolean**: Three-valued logic (TRUE, FALSE, NULL)
5. **Binary**: BYTEA for raw data
6. **Network**: INET, CIDR, MACADDR
7. **UUID**: Universally unique identifiers
8. **Arrays**: Native array support with powerful operators
9. **JSON/JSONB**: Full JSON document support
10. **Composite types**: Custom structured types
11. **Range types**: For intervals and scheduling
12. **Enums**: Ordered enumerated types
13. **Domains**: Constrained type aliases

## Data Type Selection Guide

| Use Case | Recommended Type |
|----------|-----------------|
| Primary key | SERIAL, BIGSERIAL, or UUID |
| Money | NUMERIC(precision, scale) |
| Free-form text | TEXT |
| Limited text | VARCHAR(n) |
| Timestamps | TIMESTAMPTZ |
| True/False | BOOLEAN |
| Structured data | JSONB |
| IP addresses | INET |
| Tags/Categories | TEXT[] or ENUM |
| Time periods | Range types |

---

## Chapter Cleanup

The following tables and types were created for demonstration purposes in this chapter. Run this cleanup to remove them:

**🔬 Try It:** Clean up demonstration objects

```sql
-- Drop demonstration tables (in dependency order)
DROP TABLE IF EXISTS 
    integer_examples,
    decimal_examples,
    expense_reimbursements,
    text_examples,
    datetime_examples,
    feature_flags,
    files,
    network_config,
    sessions,
    products_v2,
    blog_posts,
    events,
    employee_profiles,
    reservations,
    mood_log,
    contacts,
    payroll_runs
CASCADE;

-- Drop custom types (must drop dependent tables first)
DROP TYPE IF EXISTS address CASCADE;
DROP TYPE IF EXISTS mood CASCADE;

-- Drop domains
DROP DOMAIN IF EXISTS email CASCADE;
DROP DOMAIN IF EXISTS positive_integer CASCADE;
DROP DOMAIN IF EXISTS us_zip CASCADE;
DROP DOMAIN IF EXISTS percentage CASCADE;
DROP DOMAIN IF EXISTS money_positive CASCADE;
DROP DOMAIN IF EXISTS nonempty_text CASCADE;
DROP DOMAIN IF EXISTS url CASCADE;
```

> **Note:** The `CASCADE` option automatically drops any objects that depend on the dropped object. Be careful with this in production!

---

**Previous Chapter:** [← SQL Fundamentals](02-sql-fundamentals.md)

**Next Chapter:** [Database Architecture →](04-database-architecture.md)

