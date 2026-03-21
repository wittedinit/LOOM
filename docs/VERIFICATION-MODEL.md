# Verification Model

> How LOOM proves convergence: desired state matches observed state.

## Core Principle

Verification is not a health check. Verification is convergence proof. Every workflow declares what success looks like (`ExpectedOutcome`), and verification compares observed reality against those expectations. If observed state does not match expected state, the workflow has failed -- regardless of whether the execution steps returned success.

## Rules

1. **Each workflow declares `ExpectedOutcome`** at submission time. This is a set of expected property values on target resources.
2. **Verification checks map to expected outcomes.** A check does not exist in isolation. Every check corresponds to one or more expected properties from the `ExpectedOutcome`.
3. **`VerificationResult` is structured evidence.** It contains the check name, expected value, observed value, pass/fail boolean, raw evidence, and timestamp.
4. **Verification results are stored separately from raw observed facts.** The facts table records what was observed. The verification table records whether observations matched expectations.
5. **Failed verification = failed workflow.** Even if every adapter call returned HTTP 200 and every activity completed without error, if verification says the device is not in the expected state, the workflow has failed.
6. **Verification is a Temporal activity.** It runs with its own timeout, retry policy, and heartbeat. It is not inlined into the main activity.
7. **Evidence is mandatory.** The `evidence` field contains raw adapter output -- the actual response, not a summary. This is the proof that the check was performed and what it found.

## Types

```go
package verification

import "time"

// VerificationResult is the outcome of a single verification check.
type VerificationResult struct {
    CheckName  string    `json:"check_name"`
    Expected   any       `json:"expected"`
    Observed   any       `json:"observed"`
    Passed     bool      `json:"passed"`
    Evidence   string    `json:"evidence"`    // raw adapter output
    CheckedAt  time.Time `json:"checked_at"`
}

// VerificationSummary aggregates all checks for a workflow.
type VerificationSummary struct {
    WorkflowID  string               `json:"workflow_id"`
    Converged   bool                 `json:"converged"`   // true only if ALL checks passed
    TotalChecks int                  `json:"total_checks"`
    Passed      int                  `json:"passed"`
    Failed      int                  `json:"failed"`
    Results     []VerificationResult `json:"results"`
    VerifiedAt  time.Time            `json:"verified_at"`
}
```

## Verification Check Types

### ReachabilityCheck

- **Verifies:** A device responds to network probes at its expected address.
- **Evidence collection:** ICMP ping, TCP connect to management port, or SNMP GET sysObjectID. Adapter returns round-trip time, response payload, and timestamp.
- **Pass criteria:** At least one probe method succeeds within timeout. Response originates from the expected IP address.
- **Fail criteria:** All probe methods time out or return errors. Response from unexpected address.

```go
type ReachabilityCheck struct {
    TargetIP    string   `json:"target_ip"`
    ProbeMethods []string `json:"probe_methods"` // "icmp", "tcp", "snmp"
    TimeoutSec  int      `json:"timeout_sec"`
}
```

### StateMatchCheck

- **Verifies:** A specific property of a resource matches the expected value.
- **Evidence collection:** Adapter reads the current value of the property from the device or controller. Returns the raw response containing the property value.
- **Pass criteria:** Observed value equals expected value (exact match or within tolerance for numeric values).
- **Fail criteria:** Observed value differs from expected value.

```go
type StateMatchCheck struct {
    ResourceType string `json:"resource_type"`
    ResourceID   string `json:"resource_id"`
    Property     string `json:"property"`       // "power_state", "hostname", "os_version", etc.
    Expected     any    `json:"expected"`
    Tolerance    *float64 `json:"tolerance"`     // for numeric comparisons, nil = exact match
}
```

### PortOpenCheck

- **Verifies:** A specific TCP/UDP port is open and accepting connections on a target host.
- **Evidence collection:** TCP connect or UDP probe to the target port. Returns connection result, any banner received, and timing.
- **Pass criteria:** Connection established within timeout. For services with known banners (SSH, HTTP), banner matches expected pattern.
- **Fail criteria:** Connection refused, timed out, or banner mismatch.

```go
type PortOpenCheck struct {
    TargetIP   string `json:"target_ip"`
    Port       int    `json:"port"`
    Protocol   string `json:"protocol"`    // "tcp", "udp"
    ExpectedBanner *string `json:"expected_banner"` // regex pattern, nil = any
    TimeoutSec int    `json:"timeout_sec"`
}
```

### SSHLoginCheck

- **Verifies:** SSH access to a host works with expected credentials/keys.
- **Evidence collection:** SSH connection attempt using the specified authentication method. Executes a simple command (e.g., `whoami`, `hostname`) and returns output.
- **Pass criteria:** SSH session established. Command executes and returns expected output.
- **Fail criteria:** Authentication failure, connection timeout, command returns unexpected output.

```go
type SSHLoginCheck struct {
    TargetIP       string `json:"target_ip"`
    Port           int    `json:"port"`            // default 22
    Username       string `json:"username"`
    AuthMethod     string `json:"auth_method"`     // "key", "password"
    CredentialRef  string `json:"credential_ref"`  // vault path, never inline secret
    TestCommand    string `json:"test_command"`     // command to execute after login
    ExpectedOutput string `json:"expected_output"` // regex pattern for command output
}
```

### HTTPHealthCheck

- **Verifies:** An HTTP(S) endpoint returns an expected response.
- **Evidence collection:** HTTP request to the target URL. Returns status code, response headers, response body (truncated if large), and timing.
- **Pass criteria:** Status code matches expected code. Response body matches expected pattern (if specified).
- **Fail criteria:** Unexpected status code, body mismatch, TLS error, or timeout.

```go
type HTTPHealthCheck struct {
    URL            string            `json:"url"`
    Method         string            `json:"method"`          // default "GET"
    Headers        map[string]string `json:"headers"`
    ExpectedStatus int               `json:"expected_status"` // default 200
    ExpectedBody   *string           `json:"expected_body"`   // regex pattern, nil = any
    TLSVerify      bool              `json:"tls_verify"`
    TimeoutSec     int               `json:"timeout_sec"`
}
```

### VLANExistsCheck

- **Verifies:** A VLAN exists on a switch or set of switches with the expected configuration.
- **Evidence collection:** Adapter queries the switch for VLAN database/config. Returns VLAN ID, name, tagged/untagged ports, and status.
- **Pass criteria:** VLAN ID exists, name matches, ports match expected membership.
- **Fail criteria:** VLAN not found, name mismatch, port membership mismatch.

```go
type VLANExistsCheck struct {
    SwitchID       string   `json:"switch_id"`
    VLANID         int      `json:"vlan_id"`
    ExpectedName   string   `json:"expected_name"`
    ExpectedPorts  []string `json:"expected_ports"`  // port names that should be members
    PortMode       string   `json:"port_mode"`       // "tagged", "untagged", "any"
}
```

### ACLEnforcementCheck

- **Verifies:** An ACL (access control list) is applied and enforcing the expected rules on a network device.
- **Evidence collection:** Adapter reads the running ACL configuration from the device. Optionally performs a traffic test to confirm enforcement. Returns the ACL rules as parsed and any test results.
- **Pass criteria:** ACL exists on the expected interface/direction. Rules match the expected set (order-sensitive comparison). Traffic test (if performed) confirms expected permit/deny behavior.
- **Fail criteria:** ACL missing, rules differ from expected, traffic test shows unexpected behavior.

```go
type ACLEnforcementCheck struct {
    DeviceID       string    `json:"device_id"`
    ACLName        string    `json:"acl_name"`
    Interface      string    `json:"interface"`       // interface where ACL is applied
    Direction      string    `json:"direction"`       // "inbound", "outbound"
    ExpectedRules  []ACLRule `json:"expected_rules"`
    TrafficTest    bool      `json:"traffic_test"`    // whether to perform active traffic test
}

type ACLRule struct {
    Sequence int    `json:"sequence"`
    Action   string `json:"action"`   // "permit", "deny"
    Protocol string `json:"protocol"` // "tcp", "udp", "icmp", "ip"
    Source   string `json:"source"`   // CIDR or "any"
    Dest     string `json:"dest"`     // CIDR or "any"
    Port     *int   `json:"port"`     // nil = any port
}
```

### DNSResolutionCheck

- **Verifies:** A DNS name resolves to the expected address(es).
- **Evidence collection:** DNS query to the configured resolver(s). Returns all A/AAAA/CNAME records, TTL values, and the authoritative nameserver.
- **Pass criteria:** At least one returned record matches the expected address. If multiple expected addresses are specified, all must be present.
- **Fail criteria:** NXDOMAIN, no matching records, or unexpected records present.

```go
type DNSResolutionCheck struct {
    Hostname          string   `json:"hostname"`
    RecordType        string   `json:"record_type"`       // "A", "AAAA", "CNAME"
    ExpectedAddresses []string `json:"expected_addresses"` // IPs or CNAME targets
    Resolvers         []string `json:"resolvers"`          // DNS servers to query, nil = system default
    TimeoutSec        int      `json:"timeout_sec"`
}
```

## Verification Lifecycle

1. Workflow submits `ExpectedOutcome` with a list of `ExpectedCheck` entries.
2. After the execution steps complete, the workflow schedules a `VerifyOutcome` activity.
3. The verification activity iterates over each expected check, calling the appropriate adapter to collect evidence.
4. Each check produces a `VerificationResult` with raw evidence.
5. All results are aggregated into a `VerificationSummary`.
6. If `Converged == true`, the workflow succeeds. If `Converged == false`, the workflow fails.
7. Verification results are stored in the verification table (separate from the facts table).
8. Events are emitted: `verification.completed` (with summary) or `verification.failed` (with details of which checks failed).

## Retry Semantics for Verification

- Verification activities have their own retry policy (separate from the main activity).
- Default: 3 attempts with exponential backoff (1s, 2s, 4s).
- Transient failures (network timeout, adapter unavailable) are retried.
- Permanent failures (property mismatch, resource not found) are not retried.
- If verification fails on all retries, the workflow fails and compensation begins.

## Anti-Patterns

- **Do not verify inside the main activity.** Verification is always a separate activity.
- **Do not skip verification for "simple" forward operations.** Every forward side effect gets verified. Compensation steps are exempt from individual verification -- they are verified by the final state check after all compensations complete (see WORKFLOW-CONTRACT.md principle 4).
- **Do not use verification as monitoring.** Verification runs once per workflow, not continuously.
- **Do not store expected outcomes after the fact.** Expected outcomes are declared at submission, before execution begins.
- **Do not summarize evidence.** Store the raw adapter response. Summaries lose forensic value.
