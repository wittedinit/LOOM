# LOOM Credential Vault: Attack Surface Analysis

> Security audit of LOOM's credential vault and cryptographic implementation.
> All attacks were implemented in Go, executed, and benchmarked.
> Test code: `/tmp/loom-security-vault/attack*_test.go`

**Date:** 2026-03-21
**Platform:** darwin/arm64 (Apple M2 Max)
**Go version:** go1.24
**Verdict:** Architecture is sound. All proposed defenses verified working. Implementation must follow the patterns documented here exactly.

---

## Summary of Findings

| # | Attack | Exploitable? | Defense Verified? | Severity if Undefended |
|---|--------|-------------|-------------------|----------------------|
| 1 | Envelope Encryption Key Recovery | No | Yes | Critical |
| 2 | Memory Dump Credential Extraction | Yes (without defense) | Yes | Critical |
| 3 | Timing Side-Channel Comparison | Yes (18x timing leak) | Yes | High |
| 4 | Swap File Credential Leakage | Yes (without mlock) | Yes | High |
| 5 | JSON Serialization Leak | Yes (without defense) | Yes | High |
| 6 | Error Message Credential Leak | Yes (without defense) | Yes | High |
| 7 | Log Injection via Credential Values | Yes (without defense) | Yes | Medium |
| 8 | AES-GCM Nonce Reuse | Yes (full plaintext recovery) | Yes | Critical |
| 9 | Encryption Downgrade | Yes (if configurable) | Yes | Critical |
| 10 | Vault Backend Substitution | Yes (if mutable) | Yes | Critical |

---

## Attack 1: Envelope Encryption Key Recovery

### Threat

Attacker steals the encrypted credential blob from the database and attempts to recover the plaintext without the master key.

### Results

```
PASS: Cannot decrypt ciphertext without correct DEK
PASS: Cannot unwrap DEK without correct KEK
PASS: Cannot unwrap KEK without correct MEK
PASS: Tried 256 partial key guesses, all rejected by GCM authentication
```

AES-256-GCM's authentication tag rejects every incorrect key with probability ~1 - 2^-128. Partial key knowledge provides zero advantage.

### Argon2id Brute-Force Resistance

Benchmarked on Apple M2 Max (single core):

| Passphrase Length | Keyspace | Time at 19 guesses/sec | Time at 1000 guesses/sec |
|-------------------|----------|----------------------|------------------------|
| 8 chars | 95^8 | 1.10e+07 years | 2.10e+05 years |
| 12 chars | 95^12 | 8.95e+14 years | 1.71e+13 years |
| 16 chars | 95^16 | 7.29e+22 years | 1.39e+21 years |
| 20 chars | 95^20 | 5.94e+30 years | 1.14e+29 years |

Single Argon2id derivation (time=3, mem=64MB, threads=4): **52ms**.

At 1000 guesses/sec (GPU cluster -- limited by Argon2id's 64MB memory-hardness):
- 16-char passphrase: **1.39 x 10^21 years** to exhaust keyspace
- Heat death of universe: ~10^10 years

### Required Defenses

1. **Minimum passphrase: 16 characters** (95^16 = 4.4e31 keyspace)
2. **HSM-sealed MEK preferred** -- passphrase never exposed to software
3. **Argon2id params: time=3, mem=64MB, threads=4** (memory-hard, GPU-resistant)
4. **No passphrase hints or error messages that reveal passphrase validity**

---

## Attack 2: Memory Dump Credential Extraction

### Threat

Attacker dumps process memory (via /proc/self/mem, debugger, or core dump) and searches for credential byte patterns.

### Results

```
VULNERABILITY: Found credential pattern 1 time(s) in process memory!
VULNERABILITY: Credential STILL in memory after nil + GC! Found 1 time(s)
WARNING: Go's GC does NOT zero freed memory
```

Go's garbage collector moves objects between heap generations but **never zeros freed memory**. A process memory dump can recover credentials that were "deleted" minutes or hours ago.

### Defense: SecureBytes Type

```go
type SecureBytes struct {
    data   []byte  // mmap'd, mlock'd pages
    locked bool
    closed bool
}
```

Verified behaviors:
- Source bytes zeroed immediately after copy to secure pages
- Memory pages mlock'd (won't swap to disk)
- `Close()` explicitly zeros every byte before `munmap`
- Returns `nil` after `Close()`
- `runtime.SetFinalizer` as safety net for forgotten `Close()`

```
PASS: Source bytes zeroed after NewSecureBytes
PASS: Memory pages are mlock'd (won't be swapped to disk)
PASS: SecureBytes returns nil after Close
```

### Required Implementation

- All credential handling MUST use `SecureBytes`, never raw `[]byte` or `string`
- Credentials MUST NOT be stored in Go strings (immutable, GC-managed)
- `defer cred.Close()` pattern is mandatory
- `runtime.SetFinalizer` logs a warning if `Close()` was forgotten

---

## Attack 3: Timing Side-Channel on Credential Comparison

### Threat

Attacker measures response times to determine how many bytes of a credential guess are correct, enabling byte-by-byte recovery.

### Benchmark Results (Apple M2 Max)

**Naive `==` comparison (VULNERABLE):**

| Scenario | Time (ns/op) |
|----------|-------------|
| First byte differs | **0.617** |
| Last byte differs | **11.13** |

**Timing ratio: 18.0x** -- a massive, easily exploitable side channel.

**`crypto/subtle.ConstantTimeCompare` (DEFENSE):**

| Scenario | Time (ns/op) |
|----------|-------------|
| First byte differs | **11.19** |
| Last byte differs | **11.05** |

**Timing ratio: 0.99x** -- no measurable timing leakage.

### Required Defense

- **ALL credential comparisons MUST use `crypto/subtle.ConstantTimeCompare`**
- This includes: HMAC verification, API key validation, password hash comparison
- Never use `==`, `bytes.Equal`, or any short-circuit comparison on credential material
- Lint rule: flag any `==` comparison involving types named `*Credential*`, `*Secret*`, `*Password*`

---

## Attack 4: Swap File Credential Leakage

### Threat

Under memory pressure, the OS swaps process pages to disk. An attacker with disk access (physical, backup, or decommissioned drive) can read credential bytes from the swap partition.

### Results

```
PASS: Credential pages are mlock'd -- OS cannot swap them to disk
PASS: RLIMIT_CORE=0 -- no core dump files created
```

### Required Defenses (Startup Checklist)

| Defense | Linux | macOS |
|---------|-------|-------|
| `mlock` on credential pages | Yes | Yes |
| `RLIMIT_CORE = 0` | Yes | Yes |
| `PR_SET_DUMPABLE = 0` | Yes | N/A |
| `RLIMIT_MEMLOCK` tuning | Yes (check limits) | Yes |
| seccomp `ptrace` filter | Optional (advanced) | N/A |

LOOM MUST refuse to start if `RLIMIT_MEMLOCK` is too low for the expected credential count.

---

## Attack 5: JSON Serialization Credential Leak

### Threat

Developer accidentally passes a credential struct to `json.Marshal`, `fmt.Sprintf`, or `log.Printf`, exposing plaintext in logs, API responses, or Temporal workflow history.

### Results

**Without defense:**
```
json.Marshal: {"password":"s3cret-IPMI-p@ss!"}
fmt.Sprintf("%+v"): {Password:s3cret-IPMI-p@ss!}
fmt.Sprintf("%#v"): VulnerableCredential{Password:"s3cret-IPMI-p@ss!"}
```

**With defense (SecureCredential):**
```
json.Marshal: {"password":"***REDACTED***"}
fmt.Sprintf("%+v"): SecureCredential{Password:"***REDACTED***"}
fmt.Sprintf("%#v"): SecureCredential{Password:"***REDACTED***"}
fmt.Sprintf("%s"): SecureCredential{Password:***REDACTED***}
```

### Required Defense

Every credential-bearing struct MUST implement:

1. `MarshalJSON()` -- returns `***REDACTED***` for credential fields
2. `String()` -- redacts for `%v` and `%s` format verbs
3. `GoString()` -- redacts for `%#v` format verb
4. `Format(fmt.State, rune)` -- catches ALL format verbs as a safety net
5. Credential field MUST be unexported (`password []byte`, not `Password string`)

---

## Attack 6: Error Message Credential Leak

### Threat

```go
err := fmt.Errorf("failed to connect with password: %s", password)
```

This error propagates through error chains to logs, API responses, Temporal history, distributed tracing, and alerting systems.

### Results

```
VULNERABILITY: Error chain: "workflow step 3 failed: BMC operation failed:
  failed to connect with password: R3dfish-Admin-P@55w0rd!"
```

### Defense: OpaquePassword Type

All format verbs (`%v`, `%s`, `%+v`, `%#v`, `%q`) produce `[REDACTED]`:

```
PASS: Error with OpaquePassword: "connection failed with credential: [REDACTED]"
```

### Required Rules

- Credentials MUST be wrapped in `OpaquePassword` before any operation that could produce an error
- Error messages describe the *operation* and *target*, never the *credential*
- `credentialError` type provides operation context without credential material
- Code review: reject any `fmt.Errorf` or `errors.New` that references a credential variable

---

## Attack 7: Log Injection via Credential Values

### Threat

A credential value containing `\n`, `"`, shell metacharacters, or ANSI escape codes can:
- Create fake log entries (newline injection)
- Execute commands if logs are piped to a shell
- Inject fields into JSON-structured logs
- Clear/corrupt terminal output (ANSI escapes)

### Results

**Unstructured logging (VULNERABLE):**
```
Line 0: [INFO] Auth attempt with password: real_pass
Line 1: [CRITICAL] SYSTEM BREACH DETECTED - shutting down  <-- INJECTED
```

**JSON injection:**
```json
{"level":"info","password":"","severity":"CRITICAL","message":"BREACH"}
```
Injected `severity` field into the log entry.

**Structured logging with json.Marshal (DEFENSE):**
```
PASS: Command injection safely escaped
PASS: Newline injection escaped
PASS: JSON injection safely escaped
PASS: Credential fields automatically redacted
PASS: ANSI escape codes safely escaped
```

### Required Defense

- **Structured JSON logging only** (no `fmt.Sprintf`-based log lines)
- `json.Marshal` handles all escaping automatically
- Logger middleware auto-redacts fields named `password`, `secret`, `token`, `key`, `credential`, `api_key`
- Credential types refuse to serialize (see Attack 5)

---

## Attack 8: Nonce Reuse in AES-GCM

### Threat

AES-GCM is **catastrophically broken** if a nonce is reused with the same key. Full plaintext recovery is trivial.

### Results

```
CRITICAL: Full plaintext recovery achieved via nonce reuse!
Recovered plaintext2: "SSH private key: -----BEGIN"
```

The attack is simple XOR algebra:
```
ciphertext1 XOR ciphertext2 = plaintext1 XOR plaintext2
```
With any known plaintext, all other plaintexts under the same key+nonce are recoverable. The GHASH authentication key H is also leaked, allowing message forgery.

### Nonce Collision Probability (96-bit random nonces)

| Volume | 1 year | 10 years | 100 years |
|--------|--------|----------|-----------|
| 1,000 ops/day | 8.42e-19 | 8.42e-17 | 8.42e-15 |
| 10,000 ops/day | 8.42e-17 | 8.42e-15 | 8.42e-13 |
| 100,000 ops/day | 8.42e-15 | 8.42e-13 | 8.42e-11 |

At LOOM's expected volume (1,000 ops/day):
- **16,600 years** before collision probability reaches 2^-32 (1 in 4 billion)
- NIST 2^32 invocation limit per key reached in **11,759 years**

### Required Defense

1. **Every nonce generated from `crypto/rand`** (96-bit random)
2. **Never use a counter-based nonce** (risk of counter reset or wrap)
3. **DEK rotation every 90 days** (well within safety margin)
4. **Hard limit: 2^31 encryptions per key triggers mandatory rotation**
5. **Encryption counter stored alongside key metadata**

---

## Attack 9: Encryption Downgrade

### Threat

Attacker modifies config file to force weaker encryption:
```json
{"algorithm": "AES-128-GCM", "hash_func": "SHA-1", "kdf_func": "pbkdf2", "kdf_iters": 1000}
```

Or null encryption:
```json
{"algorithm": "NONE"}
```

### Results

```
VULNERABILITY: Config accepted AES-128-GCM, SHA-1, PBKDF2 with 1000 iterations
VULNERABILITY: Config accepted NULL encryption
```

### Defense: Hardcoded Algorithm Constants

```go
const AllowedAlgorithm = "AES-256-GCM"  // compile-time constant
const AllowedHash      = "BLAKE3"
const AllowedKDF       = "Argon2id"
```

```
PASS: Algorithm is always AES-256-GCM (hardcoded constant)
PASS: Hash is always BLAKE3 (hardcoded constant)
PASS: KDF is always Argon2id (hardcoded constant)
PASS: Rejected 9999-day key rotation
PASS: Rejected zero-day key rotation
```

### Required Rules

- Cryptographic algorithms are **compile-time constants**, never configurable
- Config file only controls operational parameters (rotation period, backend type)
- Config validation rejects values outside safe bounds
- Config file integrity verified via Ed25519 signature at startup
- Binary includes hardcoded security floor that applies even without config

---

## Attack 10: Vault Backend Substitution

### Threat

Attacker changes vault backend from HSM to local file-based storage, weakening security without operator awareness.

### Results

```
VULNERABILITY: Backend silently changed from hsm to local
```

### Defense: Immutable Backend After Initialization

```
PASS: Backend initialized to hsm
PASS: Backend change rejected: vault backend is immutable after initialization
PASS: Backend still hsm after rejected change attempt
Audit: {"event":"BACKEND_CHANGE_REJECTED","severity":"critical",
        "details":"Attempted to change backend from hsm to local"}
```

Backend security levels:
| Backend | Level | Production? |
|---------|-------|------------|
| HSM (PKCS#11) | 5/5 | Recommended |
| TPM 2.0 | 4/5 | Acceptable |
| AWS KMS / GCP KMS / Azure KV | 3/5 | Acceptable |
| HashiCorp Vault Transit | 3/5 | Acceptable |
| Local file-based | 1/5 | **Air-gapped only** |

### Required Rules

- Backend type is **immutable after first initialization** (`sync.Once` pattern)
- Backend change attempts generate CRITICAL audit events
- Config hash (SHA-256) is recorded at startup and verified on restart
- Operations teams alert on `BACKEND_INITIALIZED` with `backend=local` in production
- Operations teams alert on `BACKEND_CHANGE_REJECTED` (indicates active attack)

---

## Implementation Priority

### P0 -- Must Have Before Any Credential Touches the System

1. **SecureBytes type** with mlock, explicit zeroing, finalizer (Attack 2, 4)
2. **`crypto/subtle.ConstantTimeCompare`** for all credential comparisons (Attack 3)
3. **Random nonce from `crypto/rand`** for every AES-GCM encryption (Attack 8)
4. **Hardcoded algorithm constants** -- AES-256-GCM, BLAKE3, Argon2id (Attack 9)
5. **Immutable backend after initialization** (Attack 10)

### P1 -- Must Have Before Production

6. **Custom MarshalJSON / String / GoString / Format** on all credential types (Attack 5)
7. **OpaquePassword wrapper** -- credential values never appear in errors (Attack 6)
8. **Structured JSON logging** with auto-redaction of credential fields (Attack 7)
9. **RLIMIT_CORE=0 and PR_SET_DUMPABLE=0** at startup (Attack 4)
10. **Argon2id with time=3, mem=64MB** for passphrase-derived keys (Attack 1)

### P2 -- Should Have

11. DEK rotation counter with hard limit at 2^31 (Attack 8)
12. Config file Ed25519 signature verification (Attack 9)
13. Backend security level logging and alerting (Attack 10)
14. `seccomp` filter to prevent ptrace (Attack 4)
15. Lint rules to flag `==` on credential types and `fmt.Errorf` with credential variables

---

## Test Execution Summary

```
Platform: darwin/arm64 (Apple M2 Max)
Tests:    30 passed, 0 failed
Duration: 0.477s (tests) + 11.588s (benchmarks)

Benchmark: Naive compare (first byte differs):    0.617 ns/op
Benchmark: Naive compare (last byte differs):    11.13  ns/op  <-- 18x slower
Benchmark: ConstantTime (first byte differs):    11.19  ns/op
Benchmark: ConstantTime (last byte differs):     11.05  ns/op  <-- no difference

Argon2id single derivation: 52ms (time=3, mem=64MB, threads=4)
Argon2id guesses/sec (single core): 19
```
