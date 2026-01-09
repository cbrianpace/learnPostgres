# Appendix A: PostgreSQL Configuration Reference

This appendix provides a fully annotated `postgresql.conf` configuration file with detailed explanations of every parameter.

## How to Use This Reference

Each parameter entry includes:
- **Description**: What the parameter controls
- **Type**: Data type (boolean, integer, real, string, enum)
- **Default**: The default value
- **Context**: When the parameter can be changed
- **Range/Values**: Valid values or limits

### Context Legend

| Context | Meaning | How to Apply |
|---------|---------|--------------|
| **postmaster** | Requires server restart | Stop and start PostgreSQL |
| **sighup** | Reload without restart | `pg_ctl reload` or `SELECT pg_reload_conf()` |
| **superuser** | Superuser can change at runtime | `SET parameter = value;` (superuser only) |
| **user** | Any user can change at runtime | `SET parameter = value;` |
| **superuser-backend** | Superuser can set at connection start | Connection string or `PGOPTIONS` |
| **backend** | Can only be set at connection start | Connection string or `PGOPTIONS` |
| **internal** | Cannot be changed (compile-time) | Recompile PostgreSQL |

---

## Annotated Configuration File

```ini
# ==============================================================================
# PostgreSQL Configuration File - Fully Annotated Reference
# ==============================================================================
#
# This file contains all PostgreSQL configuration parameters with detailed
# annotations. Use this as a reference when tuning your database.
#
# File format:
#   parameter_name = value   # inline comment
#
# Memory units:  B (bytes), kB, MB, GB, TB
# Time units:    us (microseconds), ms, s, min, h, d
#
# To apply changes:
#   - Parameters marked "restart required": pg_ctl restart
#   - Parameters marked "reload": pg_ctl reload or SELECT pg_reload_conf()
#   - Parameters marked "user/superuser": SET parameter = value;

# ==============================================================================
# FILE LOCATIONS
# ==============================================================================
# These parameters define where PostgreSQL looks for configuration files
# and where it writes its PID file. All paths are relative to PGDATA unless
# specified as absolute paths.

# data_directory
# ------------------------------------------------------------------------------
# Description: Location of the database cluster's data files
# Type:        string (directory path)
# Default:     ConfigDir (the -D directory or PGDATA)
# Context:     postmaster (restart required)
# Notes:       Allows the config file to be stored separately from data files.
#              Useful in some deployment scenarios.
#data_directory = 'ConfigDir'

# hba_file
# ------------------------------------------------------------------------------
# Description: Location of the host-based authentication configuration file
# Type:        string (file path)
# Default:     ConfigDir/pg_hba.conf
# Context:     postmaster (restart required)
# Notes:       This file controls which hosts can connect and how they
#              authenticate. See pg_hba.conf documentation.
#hba_file = 'ConfigDir/pg_hba.conf'

# ident_file
# ------------------------------------------------------------------------------
# Description: Location of the ident configuration file
# Type:        string (file path)
# Default:     ConfigDir/pg_ident.conf
# Context:     postmaster (restart required)
# Notes:       Maps operating system usernames to PostgreSQL usernames
#              when using ident authentication.
#ident_file = 'ConfigDir/pg_ident.conf'

# external_pid_file
# ------------------------------------------------------------------------------
# Description: Location for an additional PID file written by the server
# Type:        string (file path)
# Default:     '' (none)
# Context:     postmaster (restart required)
# Notes:       Useful when you need the PID file in a location other than
#              the data directory. Often used with init scripts.
#external_pid_file = ''


# ==============================================================================
# CONNECTIONS AND AUTHENTICATION
# ==============================================================================

# ------------------------------------------------------------------------------
# Connection Settings
# ------------------------------------------------------------------------------

# listen_addresses
# ------------------------------------------------------------------------------
# Description: IP address(es) on which the server listens for connections
# Type:        string (comma-separated list)
# Default:     'localhost'
# Context:     postmaster (restart required)
# Values:      '*' = all available interfaces
#              '0.0.0.0' = all IPv4 interfaces
#              '::' = all IPv6 interfaces
#              'localhost' = loopback only
# Security:    CRITICAL - Exposing to all interfaces ('*') without proper
#              pg_hba.conf restrictions is a security risk!
# Example:     'localhost,192.168.1.100' for specific interfaces
#listen_addresses = 'localhost'

# port
# ------------------------------------------------------------------------------
# Description: TCP port number the server listens on
# Type:        integer
# Default:     5432
# Context:     postmaster (restart required)
# Range:       1-65535 (ports below 1024 require root on Unix)
# Notes:       Standard PostgreSQL port. Change if running multiple instances
#              or to obscure the service (security through obscurity).
#port = 5432

# max_connections
# ------------------------------------------------------------------------------
# Description: Maximum number of concurrent connections allowed
# Type:        integer
# Default:     100
# Context:     postmaster (restart required)
# Range:       1-262143 (limited by available shared memory)
# Notes:       Each connection uses memory (~10MB per connection for work_mem
#              and other session state). Consider connection pooling (PgBouncer)
#              for applications needing many connections.
# Formula:     max_connections * (work_mem + temp_buffers + ...) should fit in RAM
#max_connections = 100

# reserved_connections
# ------------------------------------------------------------------------------
# Description: Number of connection slots reserved for replication and admin
# Type:        integer
# Default:     0
# Context:     postmaster (restart required)
# Range:       0-262143
# Notes:       These slots are reserved from max_connections for non-regular
#              connections. New in PostgreSQL 16.
#reserved_connections = 0

# superuser_reserved_connections
# ------------------------------------------------------------------------------
# Description: Connection slots reserved exclusively for superusers
# Type:        integer
# Default:     3
# Context:     postmaster (restart required)
# Range:       0-262143 (should be < max_connections)
# Notes:       Ensures superusers can always connect for maintenance even when
#              max_connections is reached. Essential for emergency access.
#superuser_reserved_connections = 3

# unix_socket_directories
# ------------------------------------------------------------------------------
# Description: Directory(ies) where Unix-domain socket(s) are created
# Type:        string (comma-separated directory list)
# Default:     '/tmp' (or /var/run/postgresql on some systems)
# Context:     postmaster (restart required)
# Notes:       Unix sockets are faster than TCP for local connections.
#              Multiple directories allow different applications to use
#              different socket locations.
#unix_socket_directories = '/tmp'

# unix_socket_group
# ------------------------------------------------------------------------------
# Description: Group ownership of the Unix-domain socket
# Type:        string (group name)
# Default:     '' (server's default group)
# Context:     postmaster (restart required)
# Notes:       Combined with unix_socket_permissions, controls who can
#              connect via Unix socket.
#unix_socket_group = ''

# unix_socket_permissions
# ------------------------------------------------------------------------------
# Description: File permissions for the Unix-domain socket
# Type:        integer (octal)
# Default:     0777 (anyone can connect)
# Context:     postmaster (restart required)
# Notes:       Use 0770 to restrict to owner and group only.
#              Begin with 0 for octal notation.
# Security:    Consider restricting in multi-user environments.
#unix_socket_permissions = 0777

# bonjour
# ------------------------------------------------------------------------------
# Description: Advertise the server via Bonjour/Zeroconf
# Type:        boolean
# Default:     off
# Context:     postmaster (restart required)
# Notes:       Allows automatic discovery of PostgreSQL servers on the
#              local network. macOS feature (requires Bonjour support).
#bonjour = off

# bonjour_name
# ------------------------------------------------------------------------------
# Description: Bonjour service name for this server
# Type:        string
# Default:     '' (uses computer name)
# Context:     postmaster (restart required)
# Notes:       Only relevant if bonjour = on.
#bonjour_name = ''

# ------------------------------------------------------------------------------
# TCP Settings
# ------------------------------------------------------------------------------
# These control TCP keepalive behavior. See 'man tcp' for system-level details.
# Keepalives detect dead connections and prevent connection timeout issues.

# tcp_keepalives_idle
# ------------------------------------------------------------------------------
# Description: Seconds of inactivity before sending a TCP keepalive
# Type:        integer (seconds)
# Default:     0 (use system default, typically 2 hours)
# Context:     user
# Range:       0-2147483647
# Notes:       Lower values detect dead connections faster but create more
#              network traffic. Recommended: 60-300 for cloud environments
#              where connections may be silently dropped.
#tcp_keepalives_idle = 0

# tcp_keepalives_interval
# ------------------------------------------------------------------------------
# Description: Seconds between TCP keepalive retransmits
# Type:        integer (seconds)
# Default:     0 (use system default)
# Context:     user
# Range:       0-2147483647
# Notes:       How long to wait for keepalive acknowledgment before retrying.
#tcp_keepalives_interval = 0

# tcp_keepalives_count
# ------------------------------------------------------------------------------
# Description: Number of TCP keepalives before giving up
# Type:        integer
# Default:     0 (use system default)
# Context:     user
# Range:       0-2147483647
# Notes:       After this many unacknowledged keepalives, the connection
#              is considered dead.
#tcp_keepalives_count = 0

# tcp_user_timeout
# ------------------------------------------------------------------------------
# Description: Milliseconds before an unacknowledged TCP connection times out
# Type:        integer (milliseconds)
# Default:     0 (use system default)
# Context:     user
# Range:       0-2147483647
# Notes:       Maximum time to wait for acknowledgment of transmitted data.
#              Useful for detecting connection issues faster than keepalives.
#tcp_user_timeout = 0

# client_connection_check_interval
# ------------------------------------------------------------------------------
# Description: Interval to check if client has disconnected during long queries
# Type:        integer (milliseconds)
# Default:     0 (disabled)
# Context:     user
# Range:       0-2147483647
# Notes:       When enabled, cancels queries if the client has disconnected.
#              Useful to avoid wasting resources on abandoned queries.
#              Requires OS support for detecting connection state.
#client_connection_check_interval = 0

# ------------------------------------------------------------------------------
# Authentication
# ------------------------------------------------------------------------------

# authentication_timeout
# ------------------------------------------------------------------------------
# Description: Maximum time allowed for client authentication to complete
# Type:        integer (time units)
# Default:     1min
# Context:     sighup (reload)
# Range:       1s-600s
# Notes:       Prevents denial-of-service by clients that connect but don't
#              complete authentication. Increase if using slow auth methods.
#authentication_timeout = 1min

# password_encryption
# ------------------------------------------------------------------------------
# Description: Algorithm for encrypting passwords stored in pg_authid
# Type:        enum
# Default:     scram-sha-256
# Context:     user
# Values:      scram-sha-256, md5
# Notes:       SCRAM-SHA-256 is more secure than MD5. MD5 is deprecated and
#              should only be used for compatibility with old clients.
# Security:    Always use scram-sha-256 for new installations.
#password_encryption = scram-sha-256

# scram_iterations
# ------------------------------------------------------------------------------
# Description: Number of iterations for SCRAM-SHA-256 password hashing
# Type:        integer
# Default:     4096
# Context:     user
# Range:       1-2147483647
# Notes:       Higher values make password cracking harder but slow down
#              authentication. 4096 is a reasonable balance.
#scram_iterations = 4096

# ------------------------------------------------------------------------------
# GSSAPI/Kerberos Authentication
# ------------------------------------------------------------------------------

# krb_server_keyfile
# ------------------------------------------------------------------------------
# Description: Location of the Kerberos server key file
# Type:        string (file path)
# Default:     FILE:${sysconfdir}/krb5.keytab
# Context:     sighup (reload)
# Notes:       Only needed if using GSSAPI/Kerberos authentication.
#krb_server_keyfile = 'FILE:${sysconfdir}/krb5.keytab'

# krb_caseins_users
# ------------------------------------------------------------------------------
# Description: Whether Kerberos usernames are case-insensitive
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       Set to on if your Kerberos realm uses case-insensitive names.
#krb_caseins_users = off

# gss_accept_delegation
# ------------------------------------------------------------------------------
# Description: Accept delegated GSSAPI credentials from clients
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       When on, allows the server to use the client's credentials
#              to access other Kerberos-protected services.
#gss_accept_delegation = off

# ------------------------------------------------------------------------------
# SSL Configuration
# ------------------------------------------------------------------------------
# For detailed SSL setup, see the PostgreSQL documentation on secure connections.

# ssl
# ------------------------------------------------------------------------------
# Description: Enable SSL connections
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       Requires ssl_cert_file and ssl_key_file to be configured.
# Security:    Strongly recommended for production, especially over networks.
#ssl = off

# ssl_ca_file
# ------------------------------------------------------------------------------
# Description: Certificate Authority (CA) file for client certificate verification
# Type:        string (file path)
# Default:     '' (client certificates not verified)
# Context:     sighup (reload)
# Notes:       Required for client certificate authentication (clientcert=1).
#ssl_ca_file = ''

# ssl_cert_file
# ------------------------------------------------------------------------------
# Description: Server SSL certificate file
# Type:        string (file path)
# Default:     'server.crt'
# Context:     sighup (reload)
# Notes:       Must be a PEM-encoded certificate. Can include intermediate
#              certificates (in order from server cert to root).
#ssl_cert_file = 'server.crt'

# ssl_crl_file
# ------------------------------------------------------------------------------
# Description: SSL Certificate Revocation List file
# Type:        string (file path)
# Default:     '' (no CRL checking)
# Context:     sighup (reload)
# Notes:       Lists revoked client certificates that should be rejected.
#ssl_crl_file = ''

# ssl_crl_dir
# ------------------------------------------------------------------------------
# Description: Directory containing SSL CRL files
# Type:        string (directory path)
# Default:     '' (disabled)
# Context:     sighup (reload)
# Notes:       Alternative to ssl_crl_file for multiple CRL files.
#ssl_crl_dir = ''

# ssl_key_file
# ------------------------------------------------------------------------------
# Description: Server SSL private key file
# Type:        string (file path)
# Default:     'server.key'
# Context:     sighup (reload)
# Notes:       Must be PEM-encoded. Should be readable only by the server
#              user (chmod 600). Cannot be encrypted unless using
#              ssl_passphrase_command.
# Security:    Protect this file carefully!
#ssl_key_file = 'server.key'

# ssl_ciphers
# ------------------------------------------------------------------------------
# Description: Allowed SSL cipher suites
# Type:        string (OpenSSL cipher list format)
# Default:     'HIGH:MEDIUM:+3DES:!aNULL'
# Context:     sighup (reload)
# Notes:       Format follows OpenSSL cipher list syntax.
#              Use 'openssl ciphers' to see available ciphers.
# Security:    Review periodically as cipher recommendations change.
#ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'

# ssl_prefer_server_ciphers
# ------------------------------------------------------------------------------
# Description: Server chooses cipher (vs. client preference)
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Notes:       When on, server's cipher preference order is used.
#              Recommended to ensure strong ciphers are preferred.
#ssl_prefer_server_ciphers = on

# ssl_ecdh_curve
# ------------------------------------------------------------------------------
# Description: Curve for ECDH key exchange
# Type:        string
# Default:     'prime256v1'
# Context:     sighup (reload)
# Notes:       Use 'openssl ecparam -list_curves' to see available curves.
#ssl_ecdh_curve = 'prime256v1'

# ssl_min_protocol_version
# ------------------------------------------------------------------------------
# Description: Minimum SSL/TLS protocol version allowed
# Type:        enum
# Default:     'TLSv1.2'
# Context:     sighup (reload)
# Values:      TLSv1, TLSv1.1, TLSv1.2, TLSv1.3
# Security:    TLSv1.2 minimum is recommended. TLSv1 and TLSv1.1 are
#              considered insecure.
#ssl_min_protocol_version = 'TLSv1.2'

# ssl_max_protocol_version
# ------------------------------------------------------------------------------
# Description: Maximum SSL/TLS protocol version allowed
# Type:        enum
# Default:     '' (use highest available)
# Context:     sighup (reload)
# Values:      TLSv1, TLSv1.1, TLSv1.2, TLSv1.3, '' (no limit)
# Notes:       Usually leave empty to use the best available protocol.
#ssl_max_protocol_version = ''

# ssl_dh_params_file
# ------------------------------------------------------------------------------
# Description: File containing Diffie-Hellman parameters
# Type:        string (file path)
# Default:     '' (use compiled-in defaults)
# Context:     sighup (reload)
# Notes:       Custom DH params can be generated with:
#              openssl dhparam -out dhparams.pem 2048
#ssl_dh_params_file = ''

# ssl_passphrase_command
# ------------------------------------------------------------------------------
# Description: Command to obtain passphrase for encrypted SSL private key
# Type:        string (shell command)
# Default:     ''
# Context:     sighup (reload)
# Notes:       The command should output the passphrase to stdout.
#              Example: 'echo "mypassphrase"' (insecure, use vault instead)
#ssl_passphrase_command = ''

# ssl_passphrase_command_supports_reload
# ------------------------------------------------------------------------------
# Description: Whether ssl_passphrase_command can be run during reload
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       If on, allows SSL configuration reload without restart.
#ssl_passphrase_command_supports_reload = off
```

```ini
# ==============================================================================
# RESOURCE USAGE (except WAL)
# ==============================================================================

# ------------------------------------------------------------------------------
# Memory
# ------------------------------------------------------------------------------

# shared_buffers
# ------------------------------------------------------------------------------
# Description: Memory dedicated to PostgreSQL's shared buffer cache
# Type:        integer (memory units)
# Default:     128MB
# Context:     postmaster (restart required)
# Range:       128kB - OS limit
# Tuning:      Start with 25% of total RAM for dedicated database servers.
#              Maximum ~8GB on Windows due to OS limitations.
#              Larger values don't always help due to OS cache interaction.
# Example:     For 64GB RAM server: shared_buffers = 16GB
#shared_buffers = 128MB

# huge_pages
# ------------------------------------------------------------------------------
# Description: Whether to use huge pages for shared memory
# Type:        enum
# Default:     try
# Context:     postmaster (restart required)
# Values:      off, on, try
# Notes:       'try' uses huge pages if available, falls back if not.
#              'on' fails startup if huge pages unavailable.
#              Huge pages reduce TLB misses, improving performance.
# Setup:       Linux: sysctl vm.nr_hugepages, check /proc/meminfo
#huge_pages = try

# huge_page_size
# ------------------------------------------------------------------------------
# Description: Size of huge pages (if using huge pages)
# Type:        integer (memory units)
# Default:     0 (use system default, typically 2MB)
# Context:     postmaster (restart required)
# Notes:       Only change if your system supports multiple huge page sizes.
#huge_page_size = 0

# temp_buffers
# ------------------------------------------------------------------------------
# Description: Memory for temporary tables (per session)
# Type:        integer (memory units)
# Default:     8MB
# Context:     user (only before first use of temp tables in session)
# Range:       800kB - unlimited
# Notes:       Each session allocates its own temp buffers as needed.
#              Increase if you use large temporary tables.
#temp_buffers = 8MB

# max_prepared_transactions
# ------------------------------------------------------------------------------
# Description: Maximum number of simultaneously prepared transactions
# Type:        integer
# Default:     0 (disabled)
# Context:     postmaster (restart required)
# Range:       0-262143
# Notes:       Only enable if using two-phase commit (PREPARE TRANSACTION).
#              Each prepared transaction uses shared memory.
# Warning:     Don't enable unless you need it - prepared transactions
#              can block vacuuming if not resolved.
#max_prepared_transactions = 0

# work_mem
# ------------------------------------------------------------------------------
# Description: Memory for query operations (sorts, hashes, etc.)
# Type:        integer (memory units)
# Default:     4MB
# Context:     user
# Range:       64kB - unlimited
# Tuning:      Total memory used can be: work_mem * (queries * operations)
#              A complex query might use multiple operations simultaneously.
#              Be conservative! 100 connections * 10 operations * 64MB = 64GB!
# Example:     For OLTP: 4-16MB. For data warehouse queries: 256MB-1GB
#work_mem = 4MB

# hash_mem_multiplier
# ------------------------------------------------------------------------------
# Description: Multiplier for work_mem when used by hash operations
# Type:        real
# Default:     2.0
# Context:     user
# Range:       1.0-1000.0
# Notes:       Hash joins and aggregations can use work_mem * this multiplier.
#              Higher values favor hash operations in query plans.
#hash_mem_multiplier = 2.0

# maintenance_work_mem
# ------------------------------------------------------------------------------
# Description: Memory for maintenance operations (VACUUM, CREATE INDEX, etc.)
# Type:        integer (memory units)
# Default:     64MB
# Context:     user
# Range:       64kB - unlimited
# Tuning:      Can be set much higher than work_mem since maintenance
#              operations are less frequent. 256MB-1GB is common.
#              Higher values speed up VACUUM and index creation.
#maintenance_work_mem = 64MB

# autovacuum_work_mem
# ------------------------------------------------------------------------------
# Description: Memory for autovacuum worker processes
# Type:        integer (memory units)
# Default:     -1 (use maintenance_work_mem)
# Context:     sighup (reload)
# Range:       -1, 64kB - unlimited
# Notes:       Separate setting allows limiting autovacuum memory usage
#              independently from manual VACUUM operations.
#autovacuum_work_mem = -1

# logical_decoding_work_mem
# ------------------------------------------------------------------------------
# Description: Memory for logical decoding before spilling to disk
# Type:        integer (memory units)
# Default:     64MB
# Context:     user
# Range:       64kB - unlimited
# Notes:       Used by logical replication. Higher values reduce disk I/O
#              but increase memory usage per replication slot.
#logical_decoding_work_mem = 64MB

# max_stack_depth
# ------------------------------------------------------------------------------
# Description: Maximum safe depth of the server's execution stack
# Type:        integer (memory units)
# Default:     2MB
# Context:     superuser
# Range:       100kB - unlimited
# Notes:       Should be less than kernel's stack size limit (ulimit -s).
#              Prevents stack overflow crashes from deeply nested queries.
#max_stack_depth = 2MB

# shared_memory_type
# ------------------------------------------------------------------------------
# Description: Type of shared memory implementation to use
# Type:        enum
# Default:     mmap (platform-dependent)
# Context:     postmaster (restart required)
# Values:      mmap, sysv, windows
# Notes:       mmap is usually best on modern systems.
#              sysv may be needed for some containers or older systems.
#shared_memory_type = mmap

# dynamic_shared_memory_type
# ------------------------------------------------------------------------------
# Description: Type of dynamic shared memory implementation
# Type:        enum
# Default:     posix (platform-dependent)
# Context:     postmaster (restart required)
# Values:      posix, sysv, windows, mmap
# Notes:       Used for parallel query workers and other dynamic allocations.
#dynamic_shared_memory_type = posix

# min_dynamic_shared_memory
# ------------------------------------------------------------------------------
# Description: Minimum dynamic shared memory allocated at startup
# Type:        integer (memory units)
# Default:     0MB
# Context:     postmaster (restart required)
# Notes:       Pre-allocating can improve parallel query startup time.
#min_dynamic_shared_memory = 0MB

# vacuum_buffer_usage_limit
# ------------------------------------------------------------------------------
# Description: Buffer pool size limit for VACUUM and ANALYZE
# Type:        integer (memory units)
# Default:     2MB
# Context:     user
# Range:       0, 128kB-16GB
# Notes:       Limits how much of shared_buffers vacuum can use.
#              0 disables the limit. Prevents vacuum from evicting hot data.
#vacuum_buffer_usage_limit = 2MB

# ------------------------------------------------------------------------------
# SLRU Buffers (change requires restart)
# ------------------------------------------------------------------------------

# commit_timestamp_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for commit timestamp cache (pg_commit_ts)
# Type:        integer (number of buffers or 0 for auto)
# Default:     0 (auto-tuned)
# Context:     postmaster (restart required)
# Notes:       Only relevant if track_commit_timestamp = on.
#commit_timestamp_buffers = 0

# multixact_offset_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for MultiXact offset cache
# Type:        integer (number of 8kB buffers)
# Default:     16
# Context:     postmaster (restart required)
# Notes:       MultiXacts track row locks from multiple transactions.
#multixact_offset_buffers = 16

# multixact_member_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for MultiXact member cache
# Type:        integer (number of 8kB buffers)
# Default:     32
# Context:     postmaster (restart required)
#multixact_member_buffers = 32

# notify_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for NOTIFY queue
# Type:        integer (number of 8kB buffers)
# Default:     16
# Context:     postmaster (restart required)
# Notes:       Increase if using LISTEN/NOTIFY heavily.
#notify_buffers = 16

# serializable_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for serializable transaction tracking
# Type:        integer (number of 8kB buffers)
# Default:     32
# Context:     postmaster (restart required)
# Notes:       Only relevant if using SERIALIZABLE isolation level.
#serializable_buffers = 32

# subtransaction_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for subtransaction cache (pg_subtrans)
# Type:        integer (number of buffers or 0 for auto)
# Default:     0 (auto-tuned)
# Context:     postmaster (restart required)
#subtransaction_buffers = 0

# transaction_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for transaction status cache (pg_xact)
# Type:        integer (number of buffers or 0 for auto)
# Default:     0 (auto-tuned)
# Context:     postmaster (restart required)
#transaction_buffers = 0

# ------------------------------------------------------------------------------
# Disk
# ------------------------------------------------------------------------------

# temp_file_limit
# ------------------------------------------------------------------------------
# Description: Maximum disk space for temporary files per process
# Type:        integer (kB)
# Default:     -1 (unlimited)
# Context:     superuser
# Range:       -1 or 0 - unlimited
# Notes:       Limits disk usage from sorts, hashes, and temp tables
#              that spill to disk. -1 means no limit.
# Warning:     Without limits, a runaway query can fill disk.
#temp_file_limit = -1

# max_notify_queue_pages
# ------------------------------------------------------------------------------
# Description: Maximum SLRU pages for NOTIFY queue
# Type:        integer
# Default:     1048576
# Context:     postmaster (restart required)
# Notes:       Limits memory for queued notifications.
#max_notify_queue_pages = 1048576

# ------------------------------------------------------------------------------
# Kernel Resources
# ------------------------------------------------------------------------------

# max_files_per_process
# ------------------------------------------------------------------------------
# Description: Maximum number of simultaneously open files per process
# Type:        integer
# Default:     1000
# Context:     postmaster (restart required)
# Range:       64 - unlimited
# Notes:       Should be less than the OS limit (ulimit -n).
#              Increase for databases with many tables/indexes.
#max_files_per_process = 1000

# ------------------------------------------------------------------------------
# Cost-Based Vacuum Delay
# ------------------------------------------------------------------------------
# These settings throttle vacuum to reduce I/O impact on production queries.

# vacuum_cost_delay
# ------------------------------------------------------------------------------
# Description: Time to sleep after vacuum reaches cost limit
# Type:        integer (milliseconds)
# Default:     0 (disabled)
# Context:     user
# Range:       0-100ms
# Notes:       0 disables cost-based delay. Enable to reduce vacuum I/O impact.
#              Typical value: 2-10ms for busy systems.
#vacuum_cost_delay = 0

# vacuum_cost_page_hit
# ------------------------------------------------------------------------------
# Description: Cost of vacuuming a page found in shared buffers
# Type:        integer
# Default:     1
# Context:     user
# Range:       0-10000
# Notes:       Lower than page_miss because no disk I/O required.
#vacuum_cost_page_hit = 1

# vacuum_cost_page_miss
# ------------------------------------------------------------------------------
# Description: Cost of vacuuming a page not in shared buffers
# Type:        integer
# Default:     2
# Context:     user
# Range:       0-10000
# Notes:       Higher than page_hit due to disk read required.
#vacuum_cost_page_miss = 2

# vacuum_cost_page_dirty
# ------------------------------------------------------------------------------
# Description: Additional cost when vacuum dirties a page
# Type:        integer
# Default:     20
# Context:     user
# Range:       0-10000
# Notes:       Dirty pages must be written back, causing I/O.
#vacuum_cost_page_dirty = 20

# vacuum_cost_limit
# ------------------------------------------------------------------------------
# Description: Total cost before vacuum sleeps
# Type:        integer
# Default:     200
# Context:     user
# Range:       1-10000
# Notes:       When accumulated cost reaches this, vacuum sleeps
#              for vacuum_cost_delay. Higher = more aggressive vacuum.
#vacuum_cost_limit = 200

# ------------------------------------------------------------------------------
# Background Writer
# ------------------------------------------------------------------------------
# The background writer pre-writes dirty buffers to reduce checkpoint load.

# bgwriter_delay
# ------------------------------------------------------------------------------
# Description: Time between background writer activity rounds
# Type:        integer (milliseconds)
# Default:     200ms
# Context:     sighup (reload)
# Range:       10-10000ms
# Notes:       Lower values = more frequent writing, smoother I/O.
#bgwriter_delay = 200ms

# bgwriter_lru_maxpages
# ------------------------------------------------------------------------------
# Description: Maximum buffers written per background writer round
# Type:        integer
# Default:     100
# Context:     sighup (reload)
# Range:       0-1073741823 (0 disables)
# Notes:       Caps the work done per round.
#bgwriter_lru_maxpages = 100

# bgwriter_lru_multiplier
# ------------------------------------------------------------------------------
# Description: Multiplier for estimating buffers needed before next round
# Type:        real
# Default:     2.0
# Context:     sighup (reload)
# Range:       0.0-10.0
# Notes:       Higher values pre-write more buffers ahead of demand.
#bgwriter_lru_multiplier = 2.0

# bgwriter_flush_after
# ------------------------------------------------------------------------------
# Description: Force OS flush after this many pages written by bgwriter
# Type:        integer (pages, 8kB each)
# Default:     0 (OS default behavior)
# Context:     sighup (reload)
# Notes:       Can reduce I/O spikes but may reduce throughput.
#bgwriter_flush_after = 0

# ------------------------------------------------------------------------------
# Asynchronous Behavior
# ------------------------------------------------------------------------------

# backend_flush_after
# ------------------------------------------------------------------------------
# Description: Force OS flush after backend writes this many pages
# Type:        integer (pages)
# Default:     0 (disabled)
# Context:     user
# Notes:       Limits dirty data accumulation per backend.
#backend_flush_after = 0

# effective_io_concurrency
# ------------------------------------------------------------------------------
# Description: Number of concurrent disk I/O operations expected
# Type:        integer
# Default:     1
# Context:     user
# Range:       0-1000 (0 disables prefetching)
# Tuning:      SSD/NVMe: 200. RAID array: number of drives.
#              Single disk: 1-2. Affects bitmap heap scans.
#effective_io_concurrency = 1

# maintenance_io_concurrency
# ------------------------------------------------------------------------------
# Description: I/O concurrency for maintenance operations
# Type:        integer
# Default:     10
# Context:     user
# Range:       0-1000
# Notes:       Like effective_io_concurrency but for VACUUM, etc.
#maintenance_io_concurrency = 10

# io_combine_limit
# ------------------------------------------------------------------------------
# Description: Maximum size of I/O operations combined into one
# Type:        integer (memory units)
# Default:     128kB
# Context:     user
# Notes:       Combining sequential reads improves throughput.
#io_combine_limit = 128kB

# max_worker_processes
# ------------------------------------------------------------------------------
# Description: Maximum number of background worker processes
# Type:        integer
# Default:     8
# Context:     postmaster (restart required)
# Range:       0-262143
# Notes:       Includes parallel query workers, logical replication,
#              extensions' workers. Set >= max_parallel_workers.
#max_worker_processes = 8

# max_parallel_workers_per_gather
# ------------------------------------------------------------------------------
# Description: Maximum parallel workers per Gather node
# Type:        integer
# Default:     2
# Context:     user
# Range:       0-1024
# Notes:       Limits parallelism per query. Total workers limited by
#              max_parallel_workers.
#max_parallel_workers_per_gather = 2

# max_parallel_maintenance_workers
# ------------------------------------------------------------------------------
# Description: Maximum parallel workers for maintenance operations
# Type:        integer
# Default:     2
# Context:     user
# Range:       0-1024
# Notes:       Affects parallel index builds and similar operations.
#max_parallel_maintenance_workers = 2

# max_parallel_workers
# ------------------------------------------------------------------------------
# Description: Maximum parallel workers available system-wide
# Type:        integer
# Default:     8
# Context:     user
# Range:       0-1024
# Notes:       Total cap on parallel workers. Should be <= max_worker_processes.
#max_parallel_workers = 8

# parallel_leader_participation
# ------------------------------------------------------------------------------
# Description: Whether leader process helps execute parallel plans
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       When on, the leader participates in parallel work.
#              When off, leader only coordinates.
#parallel_leader_participation = on


# ==============================================================================
# WRITE-AHEAD LOG (WAL)
# ==============================================================================

# ------------------------------------------------------------------------------
# Settings
# ------------------------------------------------------------------------------

# wal_level
# ------------------------------------------------------------------------------
# Description: How much information is written to WAL
# Type:        enum
# Default:     replica
# Context:     postmaster (restart required)
# Values:      minimal, replica, logical
# Notes:       
#   - minimal: Crash recovery only, no replication
#   - replica: Supports streaming replication and physical backup
#   - logical: Supports logical replication (includes replica features)
# Warning:     Cannot be changed without restart. Plan for future needs.
#wal_level = replica

# fsync
# ------------------------------------------------------------------------------
# Description: Force WAL updates to disk
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Warning:     NEVER set to off in production! Data loss WILL occur on crash.
#              Only disable for expendable data or bulk loading (restore on crash).
#fsync = on

# synchronous_commit
# ------------------------------------------------------------------------------
# Description: Level of synchronization for transaction commits
# Type:        enum
# Default:     on
# Context:     user
# Values:      off, local, remote_write, remote_apply, on
# Notes:
#   - off: Return before WAL flush (risk: lose recent transactions)
#   - local: Wait for local WAL flush only
#   - remote_write: Wait for standby to receive WAL
#   - remote_apply: Wait for standby to apply WAL (strongest)
#   - on: Same as local (or remote_write with sync replication)
# Tuning:      'off' can improve performance but risks data loss.
#synchronous_commit = on

# wal_sync_method
# ------------------------------------------------------------------------------
# Description: Method used to force WAL to disk
# Type:        enum
# Default:     fsync (platform-dependent)
# Context:     sighup (reload)
# Values:      open_datasync, fdatasync, fsync, fsync_writethrough, open_sync
# Notes:       fdatasync is usually best on Linux. Each method has different
#              performance and reliability characteristics.
#wal_sync_method = fsync

# full_page_writes
# ------------------------------------------------------------------------------
# Description: Write full pages to WAL on first modification after checkpoint
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Notes:       Protects against partial page writes during crashes.
#              Can be turned off only if filesystem guarantees atomic writes.
# Warning:     Turning off risks data corruption!
#full_page_writes = on

# wal_log_hints
# ------------------------------------------------------------------------------
# Description: Also do full page writes for hint bit updates
# Type:        boolean
# Default:     off
# Context:     postmaster (restart required)
# Notes:       Required for pg_rewind to work. Increases WAL volume slightly.
#              Enable if planning to use pg_rewind for failover.
#wal_log_hints = off

# wal_compression
# ------------------------------------------------------------------------------
# Description: Compress full-page writes in WAL
# Type:        enum
# Default:     off
# Context:     superuser
# Values:      off, pglz, lz4, zstd, on (alias for pglz)
# Notes:       Reduces WAL size and I/O at cost of CPU. lz4/zstd require
#              compilation with those libraries.
#wal_compression = off

# wal_init_zero
# ------------------------------------------------------------------------------
# Description: Zero-fill new WAL files
# Type:        boolean
# Default:     on
# Context:     superuser
# Notes:       Pre-allocates space and ensures atomic writes on some filesystems.
#wal_init_zero = on

# wal_recycle
# ------------------------------------------------------------------------------
# Description: Recycle WAL files by renaming
# Type:        boolean
# Default:     on
# Context:     superuser
# Notes:       Reduces file creation overhead. Disable only if filesystem
#              doesn't support efficient rename.
#wal_recycle = on

# wal_buffers
# ------------------------------------------------------------------------------
# Description: Shared memory for WAL data before flushing
# Type:        integer (memory units)
# Default:     -1 (auto-tuned based on shared_buffers, ~3% up to 64MB)
# Context:     postmaster (restart required)
# Range:       32kB - unlimited
# Notes:       -1 auto-calculates: approximately 1/32 of shared_buffers.
#              Larger values help with heavy write workloads.
#wal_buffers = -1

# wal_writer_delay
# ------------------------------------------------------------------------------
# Description: Time between WAL writer activity rounds
# Type:        integer (milliseconds)
# Default:     200ms
# Context:     sighup (reload)
# Range:       1-10000ms
# Notes:       Controls how often WAL is flushed to disk in background.
#wal_writer_delay = 200ms

# wal_writer_flush_after
# ------------------------------------------------------------------------------
# Description: WAL writer flushes WAL to OS after this much data
# Type:        integer (memory units)
# Default:     1MB
# Context:     sighup (reload)
# Notes:       Balances latency and throughput.
#wal_writer_flush_after = 1MB

# wal_skip_threshold
# ------------------------------------------------------------------------------
# Description: Skip WAL for new files larger than this
# Type:        integer (memory units)
# Default:     2MB
# Context:     user
# Notes:       When wal_level=minimal, new files bigger than this are
#              written directly, skipping WAL.
#wal_skip_threshold = 2MB

# commit_delay
# ------------------------------------------------------------------------------
# Description: Microseconds to wait before WAL flush, hoping for group commit
# Type:        integer (microseconds)
# Default:     0 (disabled)
# Context:     superuser
# Range:       0-100000
# Notes:       Allows multiple transactions to be flushed together.
#              Only effective with high transaction rates.
#commit_delay = 0

# commit_siblings
# ------------------------------------------------------------------------------
# Description: Minimum concurrent transactions before commit_delay applies
# Type:        integer
# Default:     5
# Context:     user
# Range:       1-1000
# Notes:       commit_delay only used if this many transactions are active.
#commit_siblings = 5

# ------------------------------------------------------------------------------
# Checkpoints
# ------------------------------------------------------------------------------

# checkpoint_timeout
# ------------------------------------------------------------------------------
# Description: Maximum time between automatic checkpoints
# Type:        integer (time units)
# Default:     5min
# Context:     sighup (reload)
# Range:       30s-1d
# Tuning:      Longer intervals reduce I/O but increase recovery time.
#              Typical production: 15min-30min.
#checkpoint_timeout = 5min

# checkpoint_completion_target
# ------------------------------------------------------------------------------
# Description: Target fraction of checkpoint interval for completion
# Type:        real
# Default:     0.9
# Context:     sighup (reload)
# Range:       0.0-1.0
# Notes:       0.9 means spread checkpoint I/O over 90% of interval.
#              Higher values reduce I/O spikes but risk incomplete checkpoints.
#checkpoint_completion_target = 0.9

# checkpoint_flush_after
# ------------------------------------------------------------------------------
# Description: Force OS flush after checkpointer writes this many pages
# Type:        integer (pages)
# Default:     0 (disabled)
# Context:     sighup (reload)
# Notes:       Can smooth out I/O but may reduce throughput.
#checkpoint_flush_after = 0

# checkpoint_warning
# ------------------------------------------------------------------------------
# Description: Warn if checkpoints occur more frequently than this
# Type:        integer (time units)
# Default:     30s
# Context:     sighup (reload)
# Notes:       0 disables warning. Frequent checkpoints indicate max_wal_size
#              may be too small.
#checkpoint_warning = 30s

# max_wal_size
# ------------------------------------------------------------------------------
# Description: Maximum WAL size before triggering checkpoint
# Type:        integer (memory units)
# Default:     1GB
# Context:     sighup (reload)
# Tuning:      Larger values reduce checkpoint frequency but increase
#              disk usage and recovery time. Typical: 4GB-64GB.
#max_wal_size = 1GB

# min_wal_size
# ------------------------------------------------------------------------------
# Description: Minimum WAL size to retain for recycling
# Type:        integer (memory units)
# Default:     80MB
# Context:     sighup (reload)
# Notes:       WAL files below this threshold are kept for reuse.
#min_wal_size = 80MB

# ------------------------------------------------------------------------------
# Prefetching during recovery
# ------------------------------------------------------------------------------

# recovery_prefetch
# ------------------------------------------------------------------------------
# Description: Prefetch pages referenced in WAL during recovery
# Type:        enum
# Default:     try
# Context:     sighup (reload)
# Values:      off, on, try
# Notes:       Can significantly speed up crash recovery.
#recovery_prefetch = try

# wal_decode_buffer_size
# ------------------------------------------------------------------------------
# Description: Buffer size for WAL decoding lookahead
# Type:        integer (memory units)
# Default:     512kB
# Context:     postmaster (restart required)
# Notes:       Larger values allow more lookahead for prefetching.
#wal_decode_buffer_size = 512kB

# ------------------------------------------------------------------------------
# Archiving
# ------------------------------------------------------------------------------

# archive_mode
# ------------------------------------------------------------------------------
# Description: Enable WAL archiving
# Type:        enum
# Default:     off
# Context:     postmaster (restart required)
# Values:      off, on, always
# Notes:       
#   - off: No archiving
#   - on: Archive when server is primary
#   - always: Archive even when server is standby
#archive_mode = off

# archive_library
# ------------------------------------------------------------------------------
# Description: Shared library for WAL archiving
# Type:        string
# Default:     '' (use archive_command instead)
# Context:     sighup (reload)
# Notes:       Alternative to archive_command. Uses a loadable module.
#archive_library = ''

# archive_command
# ------------------------------------------------------------------------------
# Description: Command to archive completed WAL files
# Type:        string
# Default:     '' (archiving disabled)
# Context:     sighup (reload)
# Placeholders: %p = full path to WAL file, %f = file name only
# Example:     'cp %p /archive/%f'
#              'test ! -f /archive/%f && cp %p /archive/%f'
# Notes:       Must return 0 on success. File should be durable when
#              command returns.
#archive_command = ''

# archive_timeout
# ------------------------------------------------------------------------------
# Description: Force WAL segment switch after this time
# Type:        integer (time units)
# Default:     0 (disabled)
# Context:     sighup (reload)
# Notes:       Ensures WAL is archived even with low activity.
#              Useful for ensuring point-in-time recovery granularity.
#archive_timeout = 0

# ------------------------------------------------------------------------------
# Archive Recovery
# ------------------------------------------------------------------------------
# Used only during recovery mode (standby or point-in-time recovery).

# restore_command
# ------------------------------------------------------------------------------
# Description: Command to retrieve archived WAL files
# Type:        string
# Default:     ''
# Context:     sighup (reload)
# Placeholders: %p = path where to restore, %f = file name
# Example:     'cp /archive/%f %p'
#restore_command = ''

# archive_cleanup_command
# ------------------------------------------------------------------------------
# Description: Command to clean up old archive files
# Type:        string
# Default:     ''
# Context:     sighup (reload)
# Notes:       Runs at each restartpoint. Example: pg_archivecleanup
#archive_cleanup_command = ''

# recovery_end_command
# ------------------------------------------------------------------------------
# Description: Command to run when recovery completes
# Type:        string
# Default:     ''
# Context:     sighup (reload)
# Notes:       Runs once when recovery ends and server becomes primary.
#recovery_end_command = ''

# ------------------------------------------------------------------------------
# Recovery Target
# ------------------------------------------------------------------------------
# Set these only when performing point-in-time recovery (PITR).

# recovery_target
# ------------------------------------------------------------------------------
# Description: Named recovery point
# Type:        string
# Default:     ''
# Context:     postmaster (restart required)
# Values:      'immediate' (stop as soon as consistent)
# Notes:       Use with recovery_target_action.
#recovery_target = ''

# recovery_target_name
# ------------------------------------------------------------------------------
# Description: Named restore point to recover to
# Type:        string
# Default:     ''
# Context:     postmaster (restart required)
# Notes:       Restore points are created with pg_create_restore_point().
#recovery_target_name = ''

# recovery_target_time
# ------------------------------------------------------------------------------
# Description: Timestamp to recover to
# Type:        string (timestamp)
# Default:     ''
# Context:     postmaster (restart required)
# Example:     '2024-01-15 14:30:00 America/New_York'
#recovery_target_time = ''

# recovery_target_xid
# ------------------------------------------------------------------------------
# Description: Transaction ID to recover to
# Type:        string
# Default:     ''
# Context:     postmaster (restart required)
#recovery_target_xid = ''

# recovery_target_lsn
# ------------------------------------------------------------------------------
# Description: WAL LSN to recover to
# Type:        string
# Default:     ''
# Context:     postmaster (restart required)
# Example:     '0/12345678'
#recovery_target_lsn = ''

# recovery_target_inclusive
# ------------------------------------------------------------------------------
# Description: Whether to stop after or before the recovery target
# Type:        boolean
# Default:     on
# Context:     postmaster (restart required)
# Notes:       on = include target transaction, off = stop just before
#recovery_target_inclusive = on

# recovery_target_timeline
# ------------------------------------------------------------------------------
# Description: Timeline to recover to
# Type:        string
# Default:     'latest'
# Context:     postmaster (restart required)
# Values:      'current', 'latest', or numeric timeline ID
#recovery_target_timeline = 'latest'

# recovery_target_action
# ------------------------------------------------------------------------------
# Description: Action when recovery target is reached
# Type:        enum
# Default:     'pause'
# Context:     postmaster (restart required)
# Values:      pause, promote, shutdown
# Notes:
#   - pause: Pause and wait for confirmation
#   - promote: Automatically become primary
#   - shutdown: Shut down cleanly
#recovery_target_action = 'pause'

# ------------------------------------------------------------------------------
# WAL Summarization
# ------------------------------------------------------------------------------

# summarize_wal
# ------------------------------------------------------------------------------
# Description: Run WAL summarizer process
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       Creates summary files that can speed up certain operations.
#summarize_wal = off

# wal_summary_keep_time
# ------------------------------------------------------------------------------
# Description: How long to keep WAL summary files
# Type:        integer (time units)
# Default:     '10d'
# Context:     sighup (reload)
# Notes:       0 = never delete summaries.
#wal_summary_keep_time = '10d'


# ==============================================================================
# REPLICATION
# ==============================================================================

# ------------------------------------------------------------------------------
# Sending Servers
# ------------------------------------------------------------------------------
# Configure these on the primary and any standby that sends replication data.

# max_wal_senders
# ------------------------------------------------------------------------------
# Description: Maximum concurrent WAL sender processes
# Type:        integer
# Default:     10
# Context:     postmaster (restart required)
# Range:       0-262143
# Notes:       Each standby and backup tool needs one slot.
#              Set to 0 to disable replication.
#max_wal_senders = 10

# max_replication_slots
# ------------------------------------------------------------------------------
# Description: Maximum number of replication slots
# Type:        integer
# Default:     10
# Context:     postmaster (restart required)
# Range:       0-262143
# Notes:       Slots prevent WAL removal until consumed.
# Warning:     Unused slots can cause WAL bloat!
#max_replication_slots = 10

# wal_keep_size
# ------------------------------------------------------------------------------
# Description: Minimum WAL to retain for standbys
# Type:        integer (memory units)
# Default:     0 (disabled)
# Context:     sighup (reload)
# Notes:       Keeps WAL even without replication slots.
#              Alternative/supplement to replication slots.
#wal_keep_size = 0

# max_slot_wal_keep_size
# ------------------------------------------------------------------------------
# Description: Maximum WAL retained by replication slots
# Type:        integer (memory units)
# Default:     -1 (unlimited)
# Context:     sighup (reload)
# Notes:       Prevents disk exhaustion from stale slots.
#              Slots exceeding this become invalid.
#max_slot_wal_keep_size = -1

# wal_sender_timeout
# ------------------------------------------------------------------------------
# Description: Timeout for inactive replication connections
# Type:        integer (milliseconds)
# Default:     60s
# Context:     user
# Notes:       0 disables timeout. Terminates stale connections.
#wal_sender_timeout = 60s

# track_commit_timestamp
# ------------------------------------------------------------------------------
# Description: Record commit timestamp of transactions
# Type:        boolean
# Default:     off
# Context:     postmaster (restart required)
# Notes:       Required for some conflict resolution in logical replication.
#              Adds small overhead.
#track_commit_timestamp = off

# ------------------------------------------------------------------------------
# Primary Server
# ------------------------------------------------------------------------------
# These settings are ignored on a standby server.

# synchronous_standby_names
# ------------------------------------------------------------------------------
# Description: Standby servers that provide synchronous replication
# Type:        string
# Default:     '' (no synchronous standbys)
# Context:     sighup (reload)
# Format:      [FIRST|ANY] num (name [, ...])
# Examples:    'standby1' - one sync standby
#              'FIRST 2 (s1, s2, s3)' - first 2 of listed standbys
#              'ANY 1 (s1, s2)' - any one standby
#              '*' - any standby
# Notes:       Names match application_name of standbys.
#synchronous_standby_names = ''

# synchronized_standby_slots
# ------------------------------------------------------------------------------
# Description: Replication slots that logical walsenders wait for
# Type:        string
# Default:     ''
# Context:     sighup (reload)
# Notes:       For coordinating logical and physical replication.
#synchronized_standby_slots = ''

# ------------------------------------------------------------------------------
# Standby Servers
# ------------------------------------------------------------------------------
# These settings are ignored on a primary server.

# primary_conninfo
# ------------------------------------------------------------------------------
# Description: Connection string to the primary server
# Type:        string
# Default:     ''
# Context:     sighup (reload)
# Example:     'host=primary port=5432 user=replication password=secret'
# Notes:       Used by standby to connect to primary.
#primary_conninfo = ''

# primary_slot_name
# ------------------------------------------------------------------------------
# Description: Replication slot to use on primary
# Type:        string
# Default:     ''
# Context:     sighup (reload)
# Notes:       Using a slot prevents WAL removal on primary.
#primary_slot_name = ''

# hot_standby
# ------------------------------------------------------------------------------
# Description: Allow queries on standby server
# Type:        boolean
# Default:     on
# Context:     postmaster (restart required)
# Notes:       When on, standbys accept read-only queries.
#hot_standby = on

# max_standby_archive_delay
# ------------------------------------------------------------------------------
# Description: Maximum delay before canceling queries on standby (archive WAL)
# Type:        integer (time units)
# Default:     30s
# Context:     sighup (reload)
# Notes:       -1 = wait indefinitely (may delay recovery).
#              Balances query completion vs. replication lag.
#max_standby_archive_delay = 30s

# max_standby_streaming_delay
# ------------------------------------------------------------------------------
# Description: Maximum delay before canceling queries on standby (streaming WAL)
# Type:        integer (time units)
# Default:     30s
# Context:     sighup (reload)
# Notes:       -1 = wait indefinitely.
#max_standby_streaming_delay = 30s

# wal_receiver_create_temp_slot
# ------------------------------------------------------------------------------
# Description: Create temporary slot if primary_slot_name not set
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       Temporary slots are removed on disconnect.
#wal_receiver_create_temp_slot = off

# wal_receiver_status_interval
# ------------------------------------------------------------------------------
# Description: How often standby reports status to primary
# Type:        integer (time units)
# Default:     10s
# Context:     sighup (reload)
# Notes:       0 disables status updates.
#wal_receiver_status_interval = 10s

# hot_standby_feedback
# ------------------------------------------------------------------------------
# Description: Send feedback from standby to prevent query conflicts
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       When on, primary won't remove rows needed by standby queries.
#              Can cause table bloat on primary.
#hot_standby_feedback = off

# wal_receiver_timeout
# ------------------------------------------------------------------------------
# Description: Timeout for standby waiting for primary
# Type:        integer (milliseconds)
# Default:     60s
# Context:     sighup (reload)
# Notes:       0 disables timeout.
#wal_receiver_timeout = 60s

# wal_retrieve_retry_interval
# ------------------------------------------------------------------------------
# Description: Wait time before retrying failed WAL retrieval
# Type:        integer (time units)
# Default:     5s
# Context:     sighup (reload)
#wal_retrieve_retry_interval = 5s

# recovery_min_apply_delay
# ------------------------------------------------------------------------------
# Description: Minimum delay before applying WAL on standby
# Type:        integer (time units)
# Default:     0 (no delay)
# Context:     sighup (reload)
# Notes:       Creates a time-delayed standby for recovering from mistakes.
#              Example: 1h delay gives 1 hour to recover from accidental DELETE.
#recovery_min_apply_delay = 0

# sync_replication_slots
# ------------------------------------------------------------------------------
# Description: Synchronize replication slots from primary to standby
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       Allows failover without losing slot state.
#sync_replication_slots = off

# ------------------------------------------------------------------------------
# Subscribers (Logical Replication)
# ------------------------------------------------------------------------------
# These settings are ignored on a publisher.

# max_logical_replication_workers
# ------------------------------------------------------------------------------
# Description: Maximum logical replication workers
# Type:        integer
# Default:     4
# Context:     postmaster (restart required)
# Notes:       Taken from max_worker_processes pool.
#max_logical_replication_workers = 4

# max_sync_workers_per_subscription
# ------------------------------------------------------------------------------
# Description: Maximum sync workers per subscription
# Type:        integer
# Default:     2
# Context:     sighup (reload)
# Notes:       Controls initial table sync parallelism.
#max_sync_workers_per_subscription = 2

# max_parallel_apply_workers_per_subscription
# ------------------------------------------------------------------------------
# Description: Maximum parallel apply workers per subscription
# Type:        integer
# Default:     2
# Context:     sighup (reload)
# Notes:       Controls ongoing replication parallelism.
#max_parallel_apply_workers_per_subscription = 2
```

```ini
# ==============================================================================
# QUERY TUNING
# ==============================================================================

# ------------------------------------------------------------------------------
# Planner Method Configuration
# ------------------------------------------------------------------------------
# These enable/disable specific execution strategies. Primarily for debugging
# or working around planner issues. Generally leave all enabled (on).

# enable_async_append
# ------------------------------------------------------------------------------
# Description: Enable asynchronous append nodes (for parallel foreign scans)
# Type:        boolean
# Default:     on
# Context:     user
#enable_async_append = on

# enable_bitmapscan
# ------------------------------------------------------------------------------
# Description: Enable bitmap scan plan type
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Bitmap scans combine multiple index scans efficiently.
#enable_bitmapscan = on

# enable_gathermerge
# ------------------------------------------------------------------------------
# Description: Enable gather merge nodes (parallel merge)
# Type:        boolean
# Default:     on
# Context:     user
#enable_gathermerge = on

# enable_hashagg
# ------------------------------------------------------------------------------
# Description: Enable hash aggregation
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Hash agg is often faster for large groups.
#enable_hashagg = on

# enable_hashjoin
# ------------------------------------------------------------------------------
# Description: Enable hash join plan type
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Hash joins are typically fastest for large tables.
#enable_hashjoin = on

# enable_incremental_sort
# ------------------------------------------------------------------------------
# Description: Enable incremental sort nodes
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Exploits partial ordering from indexes.
#enable_incremental_sort = on

# enable_indexscan
# ------------------------------------------------------------------------------
# Description: Enable index scan plan type
# Type:        boolean
# Default:     on
# Context:     user
#enable_indexscan = on

# enable_indexonlyscan
# ------------------------------------------------------------------------------
# Description: Enable index-only scan plan type
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Very fast when all columns are in the index.
#enable_indexonlyscan = on

# enable_material
# ------------------------------------------------------------------------------
# Description: Enable materialization nodes
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Materialize caches results for reuse in nested loops.
#enable_material = on

# enable_memoize
# ------------------------------------------------------------------------------
# Description: Enable memoize nodes (cache nested loop inner results)
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       New in PG14. Caches results to avoid redundant scans.
#enable_memoize = on

# enable_mergejoin
# ------------------------------------------------------------------------------
# Description: Enable merge join plan type
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Efficient when inputs are sorted.
#enable_mergejoin = on

# enable_nestloop
# ------------------------------------------------------------------------------
# Description: Enable nested loop join plan type
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Good for small tables or indexed inner tables.
#enable_nestloop = on

# enable_parallel_append
# ------------------------------------------------------------------------------
# Description: Enable parallel append nodes
# Type:        boolean
# Default:     on
# Context:     user
#enable_parallel_append = on

# enable_parallel_hash
# ------------------------------------------------------------------------------
# Description: Enable parallel hash joins
# Type:        boolean
# Default:     on
# Context:     user
#enable_parallel_hash = on

# enable_partition_pruning
# ------------------------------------------------------------------------------
# Description: Enable partition pruning
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Skips partitions that can't contain matching rows.
#enable_partition_pruning = on

# enable_partitionwise_join
# ------------------------------------------------------------------------------
# Description: Enable partition-wise joins
# Type:        boolean
# Default:     off
# Context:     user
# Notes:       Joins partitions individually. Can improve performance
#              but increases planning time.
#enable_partitionwise_join = off

# enable_partitionwise_aggregate
# ------------------------------------------------------------------------------
# Description: Enable partition-wise grouping/aggregation
# Type:        boolean
# Default:     off
# Context:     user
# Notes:       Similar tradeoff to partitionwise_join.
#enable_partitionwise_aggregate = off

# enable_presorted_aggregate
# ------------------------------------------------------------------------------
# Description: Enable pre-sorted aggregation optimization
# Type:        boolean
# Default:     on
# Context:     user
#enable_presorted_aggregate = on

# enable_seqscan
# ------------------------------------------------------------------------------
# Description: Enable sequential scan plan type
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Disabling forces index use (usually a bad idea).
#enable_seqscan = on

# enable_sort
# ------------------------------------------------------------------------------
# Description: Enable explicit sort nodes
# Type:        boolean
# Default:     on
# Context:     user
#enable_sort = on

# enable_tidscan
# ------------------------------------------------------------------------------
# Description: Enable TID scan plan type
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Direct row access by ctid. Very fast when applicable.
#enable_tidscan = on

# enable_group_by_reordering
# ------------------------------------------------------------------------------
# Description: Enable GROUP BY column reordering optimization
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Reorders GROUP BY to match available indexes.
#enable_group_by_reordering = on

# ------------------------------------------------------------------------------
# Planner Cost Constants
# ------------------------------------------------------------------------------
# These affect how the planner estimates query costs.
# Tune these based on your actual hardware characteristics.

# seq_page_cost
# ------------------------------------------------------------------------------
# Description: Planner's estimate of sequential page fetch cost
# Type:        real
# Default:     1.0
# Context:     user
# Notes:       Base unit for other cost estimates.
#seq_page_cost = 1.0

# random_page_cost
# ------------------------------------------------------------------------------
# Description: Planner's estimate of random (non-sequential) page fetch cost
# Type:        real
# Default:     4.0
# Context:     user
# Tuning:      SSD: 1.1-1.5. HDD: 4.0. RAID: 2-3.
#              Lower values favor index scans over sequential scans.
#random_page_cost = 4.0

# cpu_tuple_cost
# ------------------------------------------------------------------------------
# Description: Planner's estimate of processing one row cost
# Type:        real
# Default:     0.01
# Context:     user
#cpu_tuple_cost = 0.01

# cpu_index_tuple_cost
# ------------------------------------------------------------------------------
# Description: Planner's estimate of processing one index entry cost
# Type:        real
# Default:     0.005
# Context:     user
#cpu_index_tuple_cost = 0.005

# cpu_operator_cost
# ------------------------------------------------------------------------------
# Description: Planner's estimate of processing one operator/function cost
# Type:        real
# Default:     0.0025
# Context:     user
#cpu_operator_cost = 0.0025

# parallel_setup_cost
# ------------------------------------------------------------------------------
# Description: Planner's estimate of parallel worker startup cost
# Type:        real
# Default:     1000.0
# Context:     user
# Notes:       Higher values make planner less likely to choose parallelism.
#parallel_setup_cost = 1000.0

# parallel_tuple_cost
# ------------------------------------------------------------------------------
# Description: Planner's estimate of passing a tuple to parallel leader
# Type:        real
# Default:     0.1
# Context:     user
#parallel_tuple_cost = 0.1

# min_parallel_table_scan_size
# ------------------------------------------------------------------------------
# Description: Minimum table size to consider parallel scan
# Type:        integer (memory units)
# Default:     8MB
# Context:     user
# Notes:       Tables smaller than this won't use parallel seq scan.
#min_parallel_table_scan_size = 8MB

# min_parallel_index_scan_size
# ------------------------------------------------------------------------------
# Description: Minimum index size to consider parallel scan
# Type:        integer (memory units)
# Default:     512kB
# Context:     user
#min_parallel_index_scan_size = 512kB

# effective_cache_size
# ------------------------------------------------------------------------------
# Description: Planner's assumption about total cache size
# Type:        integer (memory units)
# Default:     4GB
# Context:     user
# Tuning:      Set to ~75% of total RAM on dedicated database server.
#              This is just an estimate for planning, not an allocation.
# Notes:       Includes shared_buffers + OS file cache.
#effective_cache_size = 4GB

# ------------------------------------------------------------------------------
# JIT (Just-in-Time Compilation) Cost Thresholds
# ------------------------------------------------------------------------------
# NOTE: JIT requires PostgreSQL to be compiled with LLVM support (--with-llvm).
# These settings have no effect if PostgreSQL was built without LLVM.

# jit_above_cost
# ------------------------------------------------------------------------------
# Description: Query cost threshold to enable JIT compilation
# Type:        real
# Default:     100000
# Context:     user
# Notes:       -1 disables JIT. Lower values use JIT more aggressively.
#              JIT helps with CPU-bound queries. Requires LLVM.
#jit_above_cost = 100000

# jit_inline_above_cost
# ------------------------------------------------------------------------------
# Description: Query cost threshold for JIT function inlining
# Type:        real
# Default:     500000
# Context:     user
# Notes:       -1 disables inlining. Inlining improves performance but
#              increases compilation time. Requires LLVM.
#jit_inline_above_cost = 500000

# jit_optimize_above_cost
# ------------------------------------------------------------------------------
# Description: Query cost threshold for expensive JIT optimizations
# Type:        real
# Default:     500000
# Context:     user
# Notes:       -1 disables optimization. Optimization takes longer but
#              produces faster code. Requires LLVM.
#jit_optimize_above_cost = 500000

# ------------------------------------------------------------------------------
# Genetic Query Optimizer (GEQO)
# ------------------------------------------------------------------------------
# For queries with many joins, GEQO uses a genetic algorithm to find
# a good (not necessarily optimal) plan quickly.

# geqo
# ------------------------------------------------------------------------------
# Description: Enable genetic query optimizer
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Useful for queries with many tables.
#geqo = on

# geqo_threshold
# ------------------------------------------------------------------------------
# Description: Number of FROM items to trigger GEQO
# Type:        integer
# Default:     12
# Context:     user
# Range:       2-2147483647
# Notes:       Below this threshold, exhaustive search is used.
#geqo_threshold = 12

# geqo_effort
# ------------------------------------------------------------------------------
# Description: GEQO effort (controls iterations/population)
# Type:        integer
# Default:     5
# Context:     user
# Range:       1-10
# Notes:       Higher = better plans but slower planning.
#geqo_effort = 5

# geqo_pool_size
# ------------------------------------------------------------------------------
# Description: GEQO population size
# Type:        integer
# Default:     0 (auto based on effort)
# Context:     user
#geqo_pool_size = 0

# geqo_generations
# ------------------------------------------------------------------------------
# Description: GEQO number of generations
# Type:        integer
# Default:     0 (auto based on effort)
# Context:     user
#geqo_generations = 0

# geqo_selection_bias
# ------------------------------------------------------------------------------
# Description: GEQO selection bias (favoring fitter individuals)
# Type:        real
# Default:     2.0
# Context:     user
# Range:       1.5-2.0
#geqo_selection_bias = 2.0

# geqo_seed
# ------------------------------------------------------------------------------
# Description: GEQO random seed
# Type:        real
# Default:     0.0
# Context:     user
# Range:       0.0-1.0
# Notes:       0.0 uses random seed. Fixed value gives reproducible plans.
#geqo_seed = 0.0

# ------------------------------------------------------------------------------
# Other Planner Options
# ------------------------------------------------------------------------------

# default_statistics_target
# ------------------------------------------------------------------------------
# Description: Default statistics target for table columns
# Type:        integer
# Default:     100
# Context:     user
# Range:       1-10000
# Tuning:      Higher values = more accurate statistics but slower ANALYZE.
#              Increase for columns with skewed distributions.
#              Can be set per-column with ALTER TABLE.
#default_statistics_target = 100

# constraint_exclusion
# ------------------------------------------------------------------------------
# Description: Use constraints for query optimization
# Type:        enum
# Default:     partition
# Context:     user
# Values:      on, off, partition
# Notes:       'partition' enables only for partition pruning (safest).
#constraint_exclusion = partition

# cursor_tuple_fraction
# ------------------------------------------------------------------------------
# Description: Expected fraction of cursor rows to be retrieved
# Type:        real
# Default:     0.1
# Context:     user
# Range:       0.0-1.0
# Notes:       Affects planning for cursors. Lower values optimize for
#              fetching few rows.
#cursor_tuple_fraction = 0.1

# from_collapse_limit
# ------------------------------------------------------------------------------
# Description: Maximum FROM items to collapse subqueries into
# Type:        integer
# Default:     8
# Context:     user
# Notes:       Above this, subqueries are planned separately.
#from_collapse_limit = 8

# jit
# ------------------------------------------------------------------------------
# Description: Enable JIT compilation
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Master switch for JIT. Has no effect without LLVM support.
#              PostgreSQL must be compiled with --with-llvm for JIT to work.
#jit = on

# join_collapse_limit
# ------------------------------------------------------------------------------
# Description: Maximum FROM items before explicit JOIN order is preserved
# Type:        integer
# Default:     8
# Context:     user
# Notes:       1 = always preserve explicit JOIN order.
#              Higher values allow more optimization freedom.
#join_collapse_limit = 8

# plan_cache_mode
# ------------------------------------------------------------------------------
# Description: How to use cached plans for prepared statements
# Type:        enum
# Default:     auto
# Context:     user
# Values:      auto, force_generic_plan, force_custom_plan
# Notes:       'auto' chooses based on cost comparison.
#plan_cache_mode = auto

# recursive_worktable_factor
# ------------------------------------------------------------------------------
# Description: Multiplier for estimating recursive CTE working table size
# Type:        real
# Default:     10.0
# Context:     user
# Range:       0.001-1000000
#recursive_worktable_factor = 10.0


# ==============================================================================
# REPORTING AND LOGGING
# ==============================================================================

# ------------------------------------------------------------------------------
# Where to Log
# ------------------------------------------------------------------------------

# log_destination
# ------------------------------------------------------------------------------
# Description: Where to send log output
# Type:        string (comma-separated list)
# Default:     'stderr'
# Context:     sighup (reload)
# Values:      stderr, csvlog, jsonlog, syslog, eventlog (Windows)
# Notes:       csvlog and jsonlog require logging_collector = on.
#              Multiple destinations can be combined.
#log_destination = 'stderr'

# logging_collector
# ------------------------------------------------------------------------------
# Description: Enable log file collection
# Type:        boolean
# Default:     off
# Context:     postmaster (restart required)
# Notes:       Required for csvlog/jsonlog. Captures stderr output to files.
#              Recommended for production.
#logging_collector = off

# log_directory
# ------------------------------------------------------------------------------
# Description: Directory for log files
# Type:        string
# Default:     'log'
# Context:     sighup (reload)
# Notes:       Relative to PGDATA or absolute path.
#log_directory = 'log'

# log_filename
# ------------------------------------------------------------------------------
# Description: Log file name pattern
# Type:        string
# Default:     'postgresql-%Y-%m-%d_%H%M%S.log'
# Context:     sighup (reload)
# Notes:       Supports strftime() format codes.
# Examples:    'postgresql-%a.log' (day of week, rotates weekly)
#              'postgresql-%Y-%m-%d.log' (daily)
#log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

# log_file_mode
# ------------------------------------------------------------------------------
# Description: File permissions for log files
# Type:        integer (octal)
# Default:     0600
# Context:     sighup (reload)
# Notes:       Use octal notation (start with 0).
#log_file_mode = 0600

# log_rotation_age
# ------------------------------------------------------------------------------
# Description: Auto-rotate log after this time
# Type:        integer (time units)
# Default:     1d
# Context:     sighup (reload)
# Notes:       0 disables time-based rotation.
#log_rotation_age = 1d

# log_rotation_size
# ------------------------------------------------------------------------------
# Description: Auto-rotate log after this size
# Type:        integer (memory units)
# Default:     10MB
# Context:     sighup (reload)
# Notes:       0 disables size-based rotation.
#log_rotation_size = 10MB

# log_truncate_on_rotation
# ------------------------------------------------------------------------------
# Description: Truncate (vs. append) same-named log file on rotation
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       Only applies to time-based rotation.
#log_truncate_on_rotation = off

# syslog_facility
# ------------------------------------------------------------------------------
# Description: Syslog facility to use
# Type:        enum
# Default:     'LOCAL0'
# Context:     sighup (reload)
# Values:      LOCAL0-LOCAL7
#syslog_facility = 'LOCAL0'

# syslog_ident
# ------------------------------------------------------------------------------
# Description: Program name for syslog messages
# Type:        string
# Default:     'postgres'
# Context:     sighup (reload)
#syslog_ident = 'postgres'

# syslog_sequence_numbers
# ------------------------------------------------------------------------------
# Description: Add sequence numbers to syslog messages
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Notes:       Helps detect lost or reordered messages.
#syslog_sequence_numbers = on

# syslog_split_messages
# ------------------------------------------------------------------------------
# Description: Split long messages for syslog
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
#syslog_split_messages = on

# event_source
# ------------------------------------------------------------------------------
# Description: Event source name for Windows Event Log
# Type:        string
# Default:     'PostgreSQL'
# Context:     postmaster (restart required)
#event_source = 'PostgreSQL'

# ------------------------------------------------------------------------------
# When to Log
# ------------------------------------------------------------------------------

# log_min_messages
# ------------------------------------------------------------------------------
# Description: Minimum message severity to log
# Type:        enum
# Default:     warning
# Context:     superuser
# Values:      debug5, debug4, debug3, debug2, debug1, info, notice, warning,
#              error, log, fatal, panic (decreasing detail)
#log_min_messages = warning

# log_min_error_statement
# ------------------------------------------------------------------------------
# Description: Log SQL for errors at this level or higher
# Type:        enum
# Default:     error
# Context:     superuser
# Values:      Same as log_min_messages
# Notes:       Shows the query that caused the error.
#log_min_error_statement = error

# log_min_duration_statement
# ------------------------------------------------------------------------------
# Description: Log statements running at least this long
# Type:        integer (milliseconds)
# Default:     -1 (disabled)
# Context:     superuser
# Notes:       0 = log all statements with duration.
#              Useful for finding slow queries.
# Example:     1000 = log queries taking > 1 second
#log_min_duration_statement = -1

# log_min_duration_sample
# ------------------------------------------------------------------------------
# Description: Log sampled statements running at least this long
# Type:        integer (milliseconds)
# Default:     -1 (disabled)
# Context:     superuser
# Notes:       Used with log_statement_sample_rate for sampling slow queries.
#log_min_duration_sample = -1

# log_statement_sample_rate
# ------------------------------------------------------------------------------
# Description: Fraction of long statements to log
# Type:        real
# Default:     1.0
# Context:     superuser
# Range:       0.0-1.0
# Notes:       1.0 = log all, 0.0 = log none.
#log_statement_sample_rate = 1.0

# log_transaction_sample_rate
# ------------------------------------------------------------------------------
# Description: Fraction of transactions to log all statements for
# Type:        real
# Default:     0.0
# Context:     superuser
# Range:       0.0-1.0
#log_transaction_sample_rate = 0.0

# log_startup_progress_interval
# ------------------------------------------------------------------------------
# Description: Interval for logging startup progress
# Type:        integer (time units)
# Default:     10s
# Context:     sighup (reload)
# Notes:       0 disables. Useful for monitoring long recovery.
#log_startup_progress_interval = 10s

# ------------------------------------------------------------------------------
# What to Log
# ------------------------------------------------------------------------------

# debug_print_parse
# ------------------------------------------------------------------------------
# Description: Log parse trees for debugging
# Type:        boolean
# Default:     off
# Context:     user
#debug_print_parse = off

# debug_print_rewritten
# ------------------------------------------------------------------------------
# Description: Log rewritten parse trees
# Type:        boolean
# Default:     off
# Context:     user
#debug_print_rewritten = off

# debug_print_plan
# ------------------------------------------------------------------------------
# Description: Log query execution plans
# Type:        boolean
# Default:     off
# Context:     user
#debug_print_plan = off

# debug_pretty_print
# ------------------------------------------------------------------------------
# Description: Indent debug output for readability
# Type:        boolean
# Default:     on
# Context:     user
#debug_pretty_print = on

# log_autovacuum_min_duration
# ------------------------------------------------------------------------------
# Description: Log autovacuum activity taking at least this long
# Type:        integer (time units)
# Default:     10min
# Context:     sighup (reload)
# Notes:       -1 = never log, 0 = always log.
#log_autovacuum_min_duration = 10min

# log_checkpoints
# ------------------------------------------------------------------------------
# Description: Log checkpoint activity
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Notes:       Useful for monitoring checkpoint frequency and duration.
#log_checkpoints = on

# log_connections
# ------------------------------------------------------------------------------
# Description: Log successful connections
# Type:        boolean
# Default:     off
# Context:     superuser-backend
# Notes:       Useful for security auditing.
#log_connections = off

# log_disconnections
# ------------------------------------------------------------------------------
# Description: Log session terminations
# Type:        boolean
# Default:     off
# Context:     superuser-backend
# Notes:       Includes session duration.
#log_disconnections = off

# log_duration
# ------------------------------------------------------------------------------
# Description: Log duration of all completed statements
# Type:        boolean
# Default:     off
# Context:     superuser
# Notes:       Use log_min_duration_statement for selective logging.
#log_duration = off

# log_error_verbosity
# ------------------------------------------------------------------------------
# Description: Detail level for error messages
# Type:        enum
# Default:     default
# Context:     superuser
# Values:      terse, default, verbose
# Notes:       'verbose' includes SQLSTATE, source file, line, function.
#log_error_verbosity = default

# log_hostname
# ------------------------------------------------------------------------------
# Description: Log client hostname (vs. just IP)
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
# Notes:       Causes DNS lookup, may slow connections.
#log_hostname = off

# log_line_prefix
# ------------------------------------------------------------------------------
# Description: Printf-style format string for log line prefix
# Type:        string
# Default:     '%m [%p] '
# Context:     sighup (reload)
# Escapes:     %a=app_name, %u=user, %d=database, %r=remote_host:port,
#              %h=remote_host, %b=backend_type, %p=PID, %P=parallel_leader_PID,
#              %t=timestamp, %m=timestamp+ms, %n=unix_epoch, %Q=query_id,
#              %i=command_tag, %e=SQLSTATE, %c=session_id, %l=session_line,
#              %s=session_start, %v=virtual_xid, %x=xid, %q=non-session_stop, %%=%
# Example:     '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
#log_line_prefix = '%m [%p] '

# log_lock_waits
# ------------------------------------------------------------------------------
# Description: Log lock waits longer than deadlock_timeout
# Type:        boolean
# Default:     off
# Context:     superuser
# Notes:       Useful for detecting lock contention issues.
#log_lock_waits = off

# log_recovery_conflict_waits
# ------------------------------------------------------------------------------
# Description: Log recovery conflicts on standby
# Type:        boolean
# Default:     off
# Context:     sighup (reload)
#log_recovery_conflict_waits = off

# log_parameter_max_length
# ------------------------------------------------------------------------------
# Description: Max bytes of bind parameters to log
# Type:        integer
# Default:     -1 (unlimited)
# Context:     superuser
# Notes:       0 = don't log parameters.
#log_parameter_max_length = -1

# log_parameter_max_length_on_error
# ------------------------------------------------------------------------------
# Description: Max bytes of bind parameters in error messages
# Type:        integer
# Default:     0 (don't log)
# Context:     user
#log_parameter_max_length_on_error = 0

# log_statement
# ------------------------------------------------------------------------------
# Description: Which SQL statements to log
# Type:        enum
# Default:     'none'
# Context:     superuser
# Values:      none, ddl, mod, all
# Notes:       'ddl' = schema changes, 'mod' = ddl + data changes, 'all' = everything
# Security:    Useful for auditing.
#log_statement = 'none'

# log_replication_commands
# ------------------------------------------------------------------------------
# Description: Log replication commands
# Type:        boolean
# Default:     off
# Context:     superuser
#log_replication_commands = off

# log_temp_files
# ------------------------------------------------------------------------------
# Description: Log temp files above this size
# Type:        integer (kB)
# Default:     -1 (disabled)
# Context:     superuser
# Notes:       0 = log all temp files. Helps identify queries needing work_mem.
#log_temp_files = -1

# log_timezone
# ------------------------------------------------------------------------------
# Description: Timezone for log timestamps
# Type:        string
# Default:     'GMT'
# Context:     sighup (reload)
#log_timezone = 'GMT'

# ------------------------------------------------------------------------------
# Process Title
# ------------------------------------------------------------------------------

# cluster_name
# ------------------------------------------------------------------------------
# Description: Name added to process titles
# Type:        string
# Default:     ''
# Context:     postmaster (restart required)
# Notes:       Helps identify processes when running multiple clusters.
#cluster_name = ''

# update_process_title
# ------------------------------------------------------------------------------
# Description: Update process title with current activity
# Type:        boolean
# Default:     on
# Context:     superuser
# Notes:       Shows query state in 'ps' output. Small overhead.
#update_process_title = on


# ==============================================================================
# STATISTICS
# ==============================================================================

# ------------------------------------------------------------------------------
# Cumulative Query and Index Statistics
# ------------------------------------------------------------------------------

# track_activities
# ------------------------------------------------------------------------------
# Description: Track current command of each session
# Type:        boolean
# Default:     on
# Context:     superuser
# Notes:       Enables pg_stat_activity view. Very useful, tiny overhead.
#track_activities = on

# track_activity_query_size
# ------------------------------------------------------------------------------
# Description: Max bytes of query text to track
# Type:        integer
# Default:     1024
# Context:     postmaster (restart required)
# Notes:       Increase for long queries.
#track_activity_query_size = 1024

# track_counts
# ------------------------------------------------------------------------------
# Description: Track table/index access statistics
# Type:        boolean
# Default:     on
# Context:     superuser
# Notes:       Required for autovacuum to work!
#track_counts = on

# track_io_timing
# ------------------------------------------------------------------------------
# Description: Track I/O timing statistics
# Type:        boolean
# Default:     off
# Context:     superuser
# Notes:       Small overhead from timing calls. Very useful for I/O analysis.
#track_io_timing = off

# track_wal_io_timing
# ------------------------------------------------------------------------------
# Description: Track WAL I/O timing statistics
# Type:        boolean
# Default:     off
# Context:     superuser
#track_wal_io_timing = off

# track_functions
# ------------------------------------------------------------------------------
# Description: Track function call statistics
# Type:        enum
# Default:     none
# Context:     superuser
# Values:      none, pl, all
# Notes:       'pl' = procedural language functions, 'all' = include SQL/C.
#track_functions = none

# stats_fetch_consistency
# ------------------------------------------------------------------------------
# Description: Consistency of statistics snapshots
# Type:        enum
# Default:     cache
# Context:     user
# Values:      none, cache, snapshot
#stats_fetch_consistency = cache

# ------------------------------------------------------------------------------
# Monitoring
# ------------------------------------------------------------------------------

# compute_query_id
# ------------------------------------------------------------------------------
# Description: Compute query identifiers
# Type:        enum
# Default:     auto
# Context:     superuser
# Values:      off, on, auto, regress
# Notes:       Query IDs help correlate EXPLAIN with pg_stat_statements.
#compute_query_id = auto

# log_statement_stats
# ------------------------------------------------------------------------------
# Description: Log cumulative statistics per statement
# Type:        boolean
# Default:     off
# Context:     superuser
#log_statement_stats = off

# log_parser_stats
# ------------------------------------------------------------------------------
# Description: Log parser performance statistics
# Type:        boolean
# Default:     off
# Context:     superuser
#log_parser_stats = off

# log_planner_stats
# ------------------------------------------------------------------------------
# Description: Log planner performance statistics
# Type:        boolean
# Default:     off
# Context:     superuser
#log_planner_stats = off

# log_executor_stats
# ------------------------------------------------------------------------------
# Description: Log executor performance statistics
# Type:        boolean
# Default:     off
# Context:     superuser
#log_executor_stats = off


# ==============================================================================
# AUTOVACUUM
# ==============================================================================
# Autovacuum automatically maintains table health by reclaiming dead tuples
# and updating statistics.

# autovacuum
# ------------------------------------------------------------------------------
# Description: Enable autovacuum subprocess
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Notes:       Requires track_counts = on. NEVER disable in production!
#autovacuum = on

# autovacuum_max_workers
# ------------------------------------------------------------------------------
# Description: Maximum autovacuum worker processes
# Type:        integer
# Default:     3
# Context:     postmaster (restart required)
# Notes:       More workers = more tables vacuumed in parallel.
#autovacuum_max_workers = 3

# autovacuum_naptime
# ------------------------------------------------------------------------------
# Description: Time between autovacuum runs on each database
# Type:        integer (time units)
# Default:     1min
# Context:     sighup (reload)
# Notes:       How often autovacuum checks for work.
#autovacuum_naptime = 1min

# autovacuum_vacuum_threshold
# ------------------------------------------------------------------------------
# Description: Minimum row updates before vacuum
# Type:        integer
# Default:     50
# Context:     sighup (reload)
# Notes:       Table vacuumed when dead rows exceed threshold + scale_factor * size.
#autovacuum_vacuum_threshold = 50

# autovacuum_vacuum_insert_threshold
# ------------------------------------------------------------------------------
# Description: Minimum row inserts before vacuum
# Type:        integer
# Default:     1000
# Context:     sighup (reload)
# Notes:       -1 disables insert-based vacuum.
#autovacuum_vacuum_insert_threshold = 1000

# autovacuum_analyze_threshold
# ------------------------------------------------------------------------------
# Description: Minimum row changes before analyze
# Type:        integer
# Default:     50
# Context:     sighup (reload)
#autovacuum_analyze_threshold = 50

# autovacuum_vacuum_scale_factor
# ------------------------------------------------------------------------------
# Description: Fraction of table to add to vacuum threshold
# Type:        real
# Default:     0.2
# Context:     sighup (reload)
# Notes:       0.2 = vacuum when 20% of rows are dead (plus threshold).
#autovacuum_vacuum_scale_factor = 0.2

# autovacuum_vacuum_insert_scale_factor
# ------------------------------------------------------------------------------
# Description: Fraction of table to add to insert vacuum threshold
# Type:        real
# Default:     0.2
# Context:     sighup (reload)
#autovacuum_vacuum_insert_scale_factor = 0.2

# autovacuum_analyze_scale_factor
# ------------------------------------------------------------------------------
# Description: Fraction of table to add to analyze threshold
# Type:        real
# Default:     0.1
# Context:     sighup (reload)
#autovacuum_analyze_scale_factor = 0.1

# autovacuum_freeze_max_age
# ------------------------------------------------------------------------------
# Description: Maximum transaction age before forced vacuum for freeze
# Type:        integer
# Default:     200000000
# Context:     postmaster (restart required)
# Notes:       Prevents transaction ID wraparound. Don't set too high!
#autovacuum_freeze_max_age = 200000000

# autovacuum_multixact_freeze_max_age
# ------------------------------------------------------------------------------
# Description: Maximum multixact age before forced vacuum
# Type:        integer
# Default:     400000000
# Context:     postmaster (restart required)
#autovacuum_multixact_freeze_max_age = 400000000

# autovacuum_vacuum_cost_delay
# ------------------------------------------------------------------------------
# Description: Cost delay for autovacuum (throttling)
# Type:        integer (milliseconds)
# Default:     2ms
# Context:     sighup (reload)
# Notes:       -1 = use vacuum_cost_delay.
#autovacuum_vacuum_cost_delay = 2ms

# autovacuum_vacuum_cost_limit
# ------------------------------------------------------------------------------
# Description: Cost limit for autovacuum workers
# Type:        integer
# Default:     -1 (use vacuum_cost_limit)
# Context:     sighup (reload)
# Notes:       Shared across all autovacuum workers.
#autovacuum_vacuum_cost_limit = -1


# ==============================================================================
# CLIENT CONNECTION DEFAULTS
# ==============================================================================

# ------------------------------------------------------------------------------
# Statement Behavior
# ------------------------------------------------------------------------------

# client_min_messages
# ------------------------------------------------------------------------------
# Description: Minimum message severity sent to client
# Type:        enum
# Default:     notice
# Context:     user
# Values:      debug5-debug1, log, notice, warning, error
#client_min_messages = notice

# search_path
# ------------------------------------------------------------------------------
# Description: Schema search path for unqualified names
# Type:        string
# Default:     '"$user", public'
# Context:     user
# Notes:       $user = schema matching session user name.
#search_path = '"$user", public'

# row_security
# ------------------------------------------------------------------------------
# Description: Enable row-level security
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       When off, queries bypass RLS policies (if permitted).
#row_security = on

# default_table_access_method
# ------------------------------------------------------------------------------
# Description: Default storage method for tables
# Type:        string
# Default:     'heap'
# Context:     user
# Notes:       Can be changed by extensions providing other storage methods.
#default_table_access_method = 'heap'

# default_tablespace
# ------------------------------------------------------------------------------
# Description: Default tablespace for new objects
# Type:        string
# Default:     '' (pg_default)
# Context:     user
#default_tablespace = ''

# default_toast_compression
# ------------------------------------------------------------------------------
# Description: Default compression for TOAST values
# Type:        enum
# Default:     'pglz'
# Context:     user
# Values:      pglz, lz4
# Notes:       lz4 requires compilation with --with-lz4.
#default_toast_compression = 'pglz'

# temp_tablespaces
# ------------------------------------------------------------------------------
# Description: Tablespace(s) for temporary objects
# Type:        string
# Default:     '' (default tablespace)
# Context:     user
# Notes:       Comma-separated list. Objects distributed randomly.
#temp_tablespaces = ''

# check_function_bodies
# ------------------------------------------------------------------------------
# Description: Check function bodies during CREATE FUNCTION
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Disable to load functions referencing undefined objects.
#check_function_bodies = on

# default_transaction_isolation
# ------------------------------------------------------------------------------
# Description: Default transaction isolation level
# Type:        enum
# Default:     'read committed'
# Context:     user
# Values:      serializable, repeatable read, read committed, read uncommitted
#default_transaction_isolation = 'read committed'

# default_transaction_read_only
# ------------------------------------------------------------------------------
# Description: Default to read-only transactions
# Type:        boolean
# Default:     off
# Context:     user
#default_transaction_read_only = off

# default_transaction_deferrable
# ------------------------------------------------------------------------------
# Description: Default to deferrable transactions
# Type:        boolean
# Default:     off
# Context:     user
# Notes:       Only meaningful for serializable read-only transactions.
#default_transaction_deferrable = off

# session_replication_role
# ------------------------------------------------------------------------------
# Description: Session's replication role for triggers
# Type:        enum
# Default:     'origin'
# Context:     superuser
# Values:      origin, replica, local
# Notes:       Controls which triggers fire.
#session_replication_role = 'origin'

# statement_timeout
# ------------------------------------------------------------------------------
# Description: Maximum time for any statement
# Type:        integer (milliseconds)
# Default:     0 (disabled)
# Context:     user
# Notes:       Protects against runaway queries.
#statement_timeout = 0

# transaction_timeout
# ------------------------------------------------------------------------------
# Description: Maximum time for any transaction
# Type:        integer (milliseconds)
# Default:     0 (disabled)
# Context:     user
#transaction_timeout = 0

# lock_timeout
# ------------------------------------------------------------------------------
# Description: Maximum time to wait for locks
# Type:        integer (milliseconds)
# Default:     0 (disabled)
# Context:     user
# Notes:       Prevents indefinite waits for locked resources.
#lock_timeout = 0

# idle_in_transaction_session_timeout
# ------------------------------------------------------------------------------
# Description: Maximum idle time in an open transaction
# Type:        integer (milliseconds)
# Default:     0 (disabled)
# Context:     user
# Notes:       Terminates sessions that hold transactions open too long.
#              Prevents table bloat from long-open transactions.
#idle_in_transaction_session_timeout = 0

# idle_session_timeout
# ------------------------------------------------------------------------------
# Description: Maximum idle time for any session
# Type:        integer (milliseconds)
# Default:     0 (disabled)
# Context:     user
# Notes:       Terminates idle connections.
#idle_session_timeout = 0

# vacuum_freeze_table_age
# ------------------------------------------------------------------------------
# Description: Age at which vacuum scans whole table to freeze
# Type:        integer
# Default:     150000000
# Context:     user
#vacuum_freeze_table_age = 150000000

# vacuum_freeze_min_age
# ------------------------------------------------------------------------------
# Description: Minimum age for vacuum to freeze a row
# Type:        integer
# Default:     50000000
# Context:     user
#vacuum_freeze_min_age = 50000000

# vacuum_failsafe_age
# ------------------------------------------------------------------------------
# Description: Age at which vacuum becomes aggressive
# Type:        integer
# Default:     1600000000
# Context:     user
# Notes:       Triggers aggressive mode to prevent wraparound.
#vacuum_failsafe_age = 1600000000

# vacuum_multixact_freeze_table_age
# ------------------------------------------------------------------------------
# Description: Multixact age for full table freeze scan
# Type:        integer
# Default:     150000000
# Context:     user
#vacuum_multixact_freeze_table_age = 150000000

# vacuum_multixact_freeze_min_age
# ------------------------------------------------------------------------------
# Description: Minimum multixact age for vacuum to freeze
# Type:        integer
# Default:     5000000
# Context:     user
#vacuum_multixact_freeze_min_age = 5000000

# vacuum_multixact_failsafe_age
# ------------------------------------------------------------------------------
# Description: Multixact age triggering aggressive vacuum
# Type:        integer
# Default:     1600000000
# Context:     user
#vacuum_multixact_failsafe_age = 1600000000

# bytea_output
# ------------------------------------------------------------------------------
# Description: Output format for bytea values
# Type:        enum
# Default:     'hex'
# Context:     user
# Values:      hex, escape
#bytea_output = 'hex'

# xmlbinary
# ------------------------------------------------------------------------------
# Description: Encoding for XML binary data
# Type:        enum
# Default:     'base64'
# Context:     user
# Values:      base64, hex
#xmlbinary = 'base64'

# xmloption
# ------------------------------------------------------------------------------
# Description: Default for XML parsing
# Type:        enum
# Default:     'content'
# Context:     user
# Values:      content, document
#xmloption = 'content'

# gin_pending_list_limit
# ------------------------------------------------------------------------------
# Description: Maximum size of GIN pending list
# Type:        integer (memory units)
# Default:     4MB
# Context:     user
# Notes:       Larger values mean faster inserts but slower searches
#              until the pending list is merged.
#gin_pending_list_limit = 4MB

# createrole_self_grant
# ------------------------------------------------------------------------------
# Description: Privileges granted by CREATEROLE to self
# Type:        string
# Default:     ''
# Context:     user
# Values:      '', 'set', 'inherit', 'set, inherit'
#createrole_self_grant = ''

# event_triggers
# ------------------------------------------------------------------------------
# Description: Enable event triggers
# Type:        boolean
# Default:     on
# Context:     superuser
#event_triggers = on

# ------------------------------------------------------------------------------
# Locale and Formatting
# ------------------------------------------------------------------------------

# datestyle
# ------------------------------------------------------------------------------
# Description: Date display format
# Type:        string
# Default:     'iso, mdy'
# Context:     user
# Notes:       Format: 'style, order'. Styles: iso, sql, postgres, german.
#              Orders: dmy, mdy, ymd.
#datestyle = 'iso, mdy'

# intervalstyle
# ------------------------------------------------------------------------------
# Description: Interval display format
# Type:        enum
# Default:     'postgres'
# Context:     user
# Values:      postgres, postgres_verbose, sql_standard, iso_8601
#intervalstyle = 'postgres'

# timezone
# ------------------------------------------------------------------------------
# Description: Session timezone
# Type:        string
# Default:     'GMT' (or system timezone after initdb)
# Context:     user
# Example:     'America/New_York', 'UTC'
#timezone = 'GMT'

# timezone_abbreviations
# ------------------------------------------------------------------------------
# Description: Which timezone abbreviation set to use
# Type:        string
# Default:     'Default'
# Context:     user
# Values:      Default, Australia, India (or custom file)
#timezone_abbreviations = 'Default'

# extra_float_digits
# ------------------------------------------------------------------------------
# Description: Extra precision for floating point display
# Type:        integer
# Default:     1
# Context:     user
# Range:       -15 to 3
# Notes:       > 0 enables precise output mode.
#extra_float_digits = 1

# client_encoding
# ------------------------------------------------------------------------------
# Description: Client-side character encoding
# Type:        string
# Default:     sql_ascii (or database encoding)
# Context:     user
# Notes:       Usually auto-detected from client.
#client_encoding = sql_ascii

# lc_messages
# ------------------------------------------------------------------------------
# Description: Locale for system messages
# Type:        string
# Default:     '' (use OS default)
# Context:     superuser
#lc_messages = ''

# lc_monetary
# ------------------------------------------------------------------------------
# Description: Locale for monetary formatting
# Type:        string
# Default:     'C'
# Context:     user
#lc_monetary = 'C'

# lc_numeric
# ------------------------------------------------------------------------------
# Description: Locale for number formatting
# Type:        string
# Default:     'C'
# Context:     user
#lc_numeric = 'C'

# lc_time
# ------------------------------------------------------------------------------
# Description: Locale for time formatting
# Type:        string
# Default:     'C'
# Context:     user
#lc_time = 'C'

# icu_validation_level
# ------------------------------------------------------------------------------
# Description: Level for ICU locale validation warnings
# Type:        enum
# Default:     warning
# Context:     user
# Values:      disabled, debug5-debug1, log, notice, warning, error
#icu_validation_level = warning

# default_text_search_config
# ------------------------------------------------------------------------------
# Description: Default text search configuration
# Type:        string
# Default:     'pg_catalog.simple'
# Context:     user
# Notes:       Set to appropriate language config (english, german, etc.)
#default_text_search_config = 'pg_catalog.simple'

# ------------------------------------------------------------------------------
# Shared Library Preloading
# ------------------------------------------------------------------------------

# local_preload_libraries
# ------------------------------------------------------------------------------
# Description: Libraries to preload into each session
# Type:        string (comma-separated)
# Default:     ''
# Context:     user (but requires files in $libdir/plugins/)
# Notes:       User-controllable preloading. Limited locations for safety.
#local_preload_libraries = ''

# session_preload_libraries
# ------------------------------------------------------------------------------
# Description: Libraries to preload into new sessions
# Type:        string (comma-separated)
# Default:     ''
# Context:     superuser
# Notes:       Loaded at connection start.
#session_preload_libraries = ''

# shared_preload_libraries
# ------------------------------------------------------------------------------
# Description: Libraries to preload at server start
# Type:        string (comma-separated)
# Default:     ''
# Context:     postmaster (restart required)
# Notes:       Required for extensions like pg_stat_statements, timescaledb.
# Example:     'pg_stat_statements,auto_explain'
#shared_preload_libraries = ''

# jit_provider
# ------------------------------------------------------------------------------
# Description: JIT provider library
# Type:        string
# Default:     'llvmjit'
# Context:     postmaster (restart required)
#jit_provider = 'llvmjit'

# ------------------------------------------------------------------------------
# Other Defaults
# ------------------------------------------------------------------------------

# dynamic_library_path
# ------------------------------------------------------------------------------
# Description: Path for dynamically loaded modules
# Type:        string
# Default:     '$libdir'
# Context:     superuser
#dynamic_library_path = '$libdir'

# gin_fuzzy_search_limit
# ------------------------------------------------------------------------------
# Description: Limit on GIN exact search results
# Type:        integer
# Default:     0 (no limit)
# Context:     user
#gin_fuzzy_search_limit = 0


# ==============================================================================
# LOCK MANAGEMENT
# ==============================================================================

# deadlock_timeout
# ------------------------------------------------------------------------------
# Description: Time to wait before checking for deadlock
# Type:        integer (time units)
# Default:     1s
# Context:     superuser
# Notes:       Deadlock detection is expensive. Don't set too low.
#deadlock_timeout = 1s

# max_locks_per_transaction
# ------------------------------------------------------------------------------
# Description: Average locks per transaction (for sizing lock table)
# Type:        integer
# Default:     64
# Context:     postmaster (restart required)
# Range:       10-unlimited
# Notes:       Lock table size = max_connections * max_locks_per_transaction.
#              Increase for DDL-heavy workloads.
#max_locks_per_transaction = 64

# max_pred_locks_per_transaction
# ------------------------------------------------------------------------------
# Description: Average predicate locks per transaction
# Type:        integer
# Default:     64
# Context:     postmaster (restart required)
# Notes:       For SERIALIZABLE isolation level.
#max_pred_locks_per_transaction = 64

# max_pred_locks_per_relation
# ------------------------------------------------------------------------------
# Description: Max predicate locks per relation before escalation
# Type:        integer
# Default:     -2
# Context:     sighup (reload)
# Notes:       Negative values are calculated from max_pred_locks_per_transaction.
#max_pred_locks_per_relation = -2

# max_pred_locks_per_page
# ------------------------------------------------------------------------------
# Description: Max predicate locks per page before escalation
# Type:        integer
# Default:     2
# Context:     sighup (reload)
#max_pred_locks_per_page = 2


# ==============================================================================
# VERSION AND PLATFORM COMPATIBILITY
# ==============================================================================

# ------------------------------------------------------------------------------
# Previous PostgreSQL Versions
# ------------------------------------------------------------------------------

# array_nulls
# ------------------------------------------------------------------------------
# Description: Enable NULL elements in arrays from string input
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Pre-8.2 compatibility if set to off.
#array_nulls = on

# backslash_quote
# ------------------------------------------------------------------------------
# Description: How to handle \' in string literals
# Type:        enum
# Default:     safe_encoding
# Context:     user
# Values:      on, off, safe_encoding
# Security:    safe_encoding prevents SQL injection in some encodings.
#backslash_quote = safe_encoding

# escape_string_warning
# ------------------------------------------------------------------------------
# Description: Warn about backslash escapes in string literals
# Type:        boolean
# Default:     on
# Context:     user
#escape_string_warning = on

# lo_compat_privileges
# ------------------------------------------------------------------------------
# Description: Enable large object compatibility mode
# Type:        boolean
# Default:     off
# Context:     superuser
# Notes:       Pre-9.0 behavior if set to on.
#lo_compat_privileges = off

# quote_all_identifiers
# ------------------------------------------------------------------------------
# Description: Quote all identifiers in dumps
# Type:        boolean
# Default:     off
# Context:     user
# Notes:       Makes dumps more portable but less readable.
#quote_all_identifiers = off

# standard_conforming_strings
# ------------------------------------------------------------------------------
# Description: Enable standard-conforming string literals
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       When on, backslash is not an escape character.
#              Use E'...' for escape strings.
#standard_conforming_strings = on

# synchronize_seqscans
# ------------------------------------------------------------------------------
# Description: Synchronize concurrent sequential scans
# Type:        boolean
# Default:     on
# Context:     user
# Notes:       Multiple scans share disk reads. Usually beneficial.
#synchronize_seqscans = on

# ------------------------------------------------------------------------------
# Other Platforms and Clients
# ------------------------------------------------------------------------------

# transform_null_equals
# ------------------------------------------------------------------------------
# Description: Treat "expr = NULL" as "expr IS NULL"
# Type:        boolean
# Default:     off
# Context:     user
# Notes:       For compatibility with non-standard clients.
# Warning:     SQL standard says = NULL should always be NULL.
#transform_null_equals = off

# allow_alter_system
# ------------------------------------------------------------------------------
# Description: Allow ALTER SYSTEM command
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Security:    Set to off to prevent config changes via SQL.
#allow_alter_system = on


# ==============================================================================
# ERROR HANDLING
# ==============================================================================

# exit_on_error
# ------------------------------------------------------------------------------
# Description: Terminate session on any error
# Type:        boolean
# Default:     off
# Context:     user
# Notes:       Useful for batch processing where any error should stop.
#exit_on_error = off

# restart_after_crash
# ------------------------------------------------------------------------------
# Description: Reinitialize all backends after one crashes
# Type:        boolean
# Default:     on
# Context:     sighup (reload)
# Notes:       When off, server stays down after crash. Useful for debugging.
#restart_after_crash = on

# data_sync_retry
# ------------------------------------------------------------------------------
# Description: Retry fsync failures (vs. panic)
# Type:        boolean
# Default:     off
# Context:     postmaster (restart required)
# Notes:       On some systems, retry is safe. On others, data may be lost.
# Warning:     Only set to on if you understand your storage system!
#data_sync_retry = off

# recovery_init_sync_method
# ------------------------------------------------------------------------------
# Description: Method for syncing data directory at recovery start
# Type:        enum
# Default:     fsync
# Context:     sighup (reload)
# Values:      fsync, syncfs (Linux 5.8+)
#recovery_init_sync_method = fsync


# ==============================================================================
# CONFIG FILE INCLUDES
# ==============================================================================

# These are directives, not variables. Can appear multiple times.

# include_dir = '...'
# ------------------------------------------------------------------------------
# Description: Include all .conf files from a directory
# Context:     config file (processed at load)
# Example:     include_dir = 'conf.d'
# Notes:       Files processed in alphabetical order.

# include_if_exists = '...'
# ------------------------------------------------------------------------------
# Description: Include file if it exists (no error if missing)
# Context:     config file
# Example:     include_if_exists = 'optional.conf'

# include = '...'
# ------------------------------------------------------------------------------
# Description: Include configuration file
# Context:     config file
# Example:     include = 'tuning.conf'


# ==============================================================================
# CUSTOMIZED OPTIONS
# ==============================================================================

# Add settings for extensions here. Extension parameters follow the format:
# extension_name.parameter = value
#
# Examples:
# pg_stat_statements.max = 10000
# pg_stat_statements.track = all
# auto_explain.log_min_duration = '1s'
# timescaledb.max_background_workers = 4
```

---

## Quick Reference: Context Summary

| Change Required | Parameters Affected | Examples |
|----------------|---------------------|----------|
| **Restart** | Memory allocation, process limits, WAL level | `shared_buffers`, `max_connections`, `wal_level` |
| **Reload** | Most runtime settings | `work_mem`, `checkpoint_timeout`, logging |
| **Superuser SET** | Security-sensitive runtime | `log_statement`, timeouts |
| **User SET** | Per-session tuning | `work_mem`, `enable_*`, `search_path` |

## Quick Reference: Memory Settings

For a 64GB RAM dedicated database server:

```ini
# Memory
shared_buffers = 16GB          # 25% of RAM
effective_cache_size = 48GB    # 75% of RAM
work_mem = 256MB               # Depends on query complexity
maintenance_work_mem = 2GB     # For VACUUM, CREATE INDEX
wal_buffers = 64MB             # -1 auto-calculates

# Parallelism
max_worker_processes = 16
max_parallel_workers_per_gather = 4
max_parallel_workers = 16
max_parallel_maintenance_workers = 4
```

## Quick Reference: Logging for Production

```ini
# Recommended production logging
log_destination = 'csvlog'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_min_duration_statement = 1000    # Log queries > 1 second
log_checkpoints = on
log_connections = on
log_disconnections = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_lock_waits = on
```

---

## Next Steps

- **[Chapter 12: Security](12-security.md)** - Roles, authentication, and access control
- **[Chapter 13: Performance Tuning](13-performance-tuning.md)** - Optimizing your configuration
- **[Appendix B: Source Code Tour](14-source-code-tour.md)** - Orientation for navigating the source tree

