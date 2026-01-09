# Chapter 12: Security (Roles, Authentication, and Access Control)

## Learning Objectives

By the end of this chapter, you will be able to:
- Model access using **roles** (login roles vs group roles) and least privilege
- Understand the end-to-end **authentication flow** (connection → `pg_hba.conf` → auth method → role checks)
- Configure and reason about `pg_hba.conf` matching rules and common auth methods
- Use PostgreSQL access controls: **GRANT/REVOKE**, default privileges, and **Row Level Security (RLS)**
- Understand SSL/TLS options and how they interact with authentication

## Prerequisites

**🔬 Try It:** Connect to the Database

```sql
\c learning
```

## Part 1: Roles and Privileges (Authorization)

PostgreSQL uses **roles** for both users and groups. A role can:
- log in (user) via the `LOGIN` attribute
- own objects (tables, schemas, functions)
- be a member of other roles (group membership)

### Role Attributes (Global Capabilities)

Role attributes control what a role can do **system-wide** (separate from table permissions).

| Attribute | Meaning |
|-----------|---------|
| `LOGIN` | Can connect to the database |
| `SUPERUSER` | Bypasses permission checks (avoid unless absolutely required) |
| `CREATEDB` | Can create databases |
| `CREATEROLE` | Can create/manage other roles |
| `REPLICATION` | Can initiate streaming replication |
| `CONNECTION LIMIT n` | Maximum concurrent connections |

**🔬 Try It:** Create and inspect roles

```sql
-- Create a login role (user)
CREATE ROLE alice WITH LOGIN PASSWORD 'secure_password';

-- Create a group role (no login)
CREATE ROLE engineering;

-- Grant membership (alice inherits engineering's privileges)
GRANT engineering TO alice;

-- View roles
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolreplication, rolcanlogin, rolconnlimit
FROM pg_roles
ORDER BY rolname;
```

> **Security note:** In real environments, avoid embedding passwords in scripts. Prefer secret managers, rotation, and SSO/cert where appropriate.

### Object Privileges (GRANT/REVOKE)

Object privileges control what roles can do with **specific database objects** (tables, views, sequences, functions).

Common table privileges: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`.

**🔬 Try It:** Least-privilege grants on HR tables

```sql
-- A read-only role for HR reporting
CREATE ROLE hr_readonly;
GRANT CONNECT ON DATABASE learning TO hr_readonly;
GRANT USAGE ON SCHEMA public TO hr_readonly;

GRANT SELECT ON employees, departments, timecards TO hr_readonly;

-- Let alice assume hr_readonly
GRANT hr_readonly TO alice;

-- Inspect grants
SELECT grantee, privilege_type, table_name
FROM information_schema.table_privileges
WHERE table_schema = 'public'
  AND table_name IN ('employees','departments','timecards')
ORDER BY table_name, grantee, privilege_type;
```

### Default Privileges (Future-proofing)

Default privileges let you define what privileges should be applied to **objects created in the future** by a particular owner.

**🔬 Try It:** Default privileges for future tables

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO hr_readonly;
```

## Part 2: Row Level Security (RLS)

Row Level Security adds per-row access control policies. It’s commonly used for:
- multi-tenant apps (each customer sees only their rows)
- internal segmentation (departments, regions)

**How it works:**
- Enable RLS on a table
- Define policies (filters and check conditions)
- PostgreSQL automatically applies them to queries

> **Important:** Superusers and table owners bypass RLS by default. Use `FORCE ROW LEVEL SECURITY` to apply policies to the owner too.

**🔬 Try It:** Department-based isolation

```sql
-- This pattern uses a session setting as “application context”.
-- In real apps you might use JWT claims, `SET ROLE`, or a trusted auth proxy to set context.

ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

CREATE POLICY employees_dept_isolation ON employees
FOR SELECT
TO PUBLIC
USING (dept_id = current_setting('app.dept_id')::INTEGER);

-- Apply RLS even for the table owner (use carefully; can surprise admins)
ALTER TABLE employees FORCE ROW LEVEL SECURITY;

-- Act as alice and set department context
SET ROLE alice;
SET app.dept_id = '1';

SELECT current_user, count(*) AS visible_employees
FROM employees;

RESET ROLE;
RESET app.dept_id;
```

## Part 3: Authentication (How Logins Actually Work)

Authentication answers: “**who are you**?” Authorization answers: “**what are you allowed to do**?”

PostgreSQL’s connection-time flow is roughly:

1. **Client connects** (local socket or TCP).
2. Server evaluates `pg_hba.conf` top-to-bottom and picks the **first matching rule**.
3. That rule selects an **authentication method** (password, cert, GSS, etc.).
4. If authentication succeeds, PostgreSQL sets `session_user` and `current_user` and finishes startup.
5. For every SQL statement, PostgreSQL performs **privilege checks** on referenced objects (schemas, tables, columns, functions), plus any active RLS policies.

### `pg_hba.conf` Matching (Order Matters)

`pg_hba.conf` is evaluated **top to bottom**. The first rule that matches:
- connection type (`local`, `host`, `hostssl`, `hostnossl`)
- database (`all`, `learning`, `replication`, etc.)
- user/role
- client address (`127.0.0.1/32`, `10.0.0.0/8`, etc.)

…wins, and determines the auth method.

**📖 Example:** `pg_hba.conf` sketch (illustrative)

> ```conf
> # TYPE   DATABASE   USER     ADDRESS         METHOD
> local    all        postgres                 peer
> local    all        all                      scram-sha-256
>
> host     learning   all      127.0.0.1/32    scram-sha-256
> hostssl  all        all      0.0.0.0/0       scram-sha-256
> ```

### Common Authentication Methods

| Method | What it means | Notes |
|--------|---------------|-------|
| `scram-sha-256` | Password-based auth using SCRAM | Modern default; supports secure password storage and challenge-response |
| `peer` | OS user must match DB user (local sockets) | Great for local admin ergonomics; not for remote |
| `cert` | Client certificate authentication | Typically used with `hostssl` plus CA configuration |
| `trust` | No authentication | Avoid except in tightly controlled local dev |
| `md5` | Legacy password method | Kept for compatibility; prefer SCRAM |

### Password Storage (SCRAM)

With `password_encryption = scram-sha-256`, PostgreSQL stores a **verifier** (not the raw password) and performs a challenge-response exchange during login.

**🔬 Try It:** Check password settings (no secrets)

```sql
SHOW password_encryption;

-- Password hash access is restricted; use pg_roles for safe inspection
SELECT rolname, rolcanlogin
FROM pg_roles
WHERE rolname IN ('alice', 'hr_readonly')
ORDER BY rolname;
```

> Accessing `pg_authid` requires elevated privileges because it contains sensitive auth data.

## Part 4: SSL/TLS and Authentication

SSL/TLS protects credentials and data-in-transit. It also enables certificate-based authentication.

**📖 Example:** Minimal SSL settings (in `postgresql.conf`)

> ```conf
> ssl = on
> ssl_cert_file = 'server.crt'
> ssl_key_file = 'server.key'
> # ssl_ca_file = 'ca.crt'  # needed if you require/verify client certificates
> ```

**🔬 Try It:** Is my current session using SSL?

```sql
SELECT ssl, version, cipher, bits, client_dn
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
```

## Part 5: Practical Security Checklist

- **Use SCRAM** (`scram-sha-256`) for password auth
- **Prefer `hostssl`** for remote connections (encrypt in transit)
- **Least privilege**: roles per app/service, not shared superusers
- **Lock down schemas**: `REVOKE CREATE ON SCHEMA public FROM PUBLIC;` in real deployments
- **Avoid `SECURITY DEFINER` unless you audit it** (and set a safe `search_path`)
- **Use RLS** for multi-tenant row separation (with careful bypass rules)

---

## Chapter Cleanup

**🔬 Try It:** Clean up demo roles (optional)

```sql
-- Undo roles created in this chapter (run only if you don't want them around)
REASSIGN OWNED BY alice TO current_user;
DROP ROLE IF EXISTS alice;
DROP ROLE IF EXISTS engineering;
DROP ROLE IF EXISTS hr_readonly;
```

---

**Previous Chapter:** [← Server-side Programming](11-server-side-programming.md)

**Next Chapter:** [Performance Tuning →](13-performance-tuning.md)



