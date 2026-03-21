# LLM Integration Boundaries

Hard enforceable rules for LLM integration in LOOM. These are not guidelines — they are constraints enforced in code review and architecture.

## Rules

### 1. LLM never performs execution directly

LLM produces a recommendation struct. A deterministic dispatcher validates and executes it.

### 2. LLM never bypasses validation

Every LLM-generated config/plan passes through the same validation pipeline as human-authored input.

### 3. LLM output is always typed

Every LLM response is parsed into a concrete Go struct. Never executed as freeform text.

```go
type LLMRecommendation struct {
    Type           string          // "classification", "placement", "config_suggestion", "oversight", "optimization"
    Confidence     float64         // 0.0 to 1.0
    Provenance     []string        // what inputs/docs the LLM based this on
    Recommendation any             // typed struct matching Type
    Fallback       any             // deterministic fallback if LLM is unavailable or low confidence
    Reasoning      string          // LLM's explanation (for audit)
}
```

### 4. Confidence + provenance required

Every recommendation carries a confidence score and list of sources. Low confidence (< 0.7) triggers human review. Very low (< 0.4) uses deterministic fallback.

### 5. Every path works without LLM

If the LLM is unavailable, degraded, or untrusted, every operation must have a deterministic fallback:

- Classification: deterministic fingerprinting (sysObjectID, Redfish schema)
- Placement: rule-based placement (resource availability, affinity rules)
- Config generation: template-based generation (Jinja/Go templates)
- Oversight detection: checklist-based validation
- Cost optimization: threshold-based alerting

### 6. LLM suggests, deterministic systems execute

The execution pipeline is:

```
Request → LLM Suggests → Deterministic Validates → Deterministic Executes → Deterministic Verifies
```

### 7. Code review enforcement

These rules are checked in code review:

- No direct execution of LLM output without validation
- No `interface{}` or `any` for LLM response types (must be typed structs)
- Fallback path must exist and be tested
- Confidence threshold must be configurable per operation type

## LLM Use Cases and Boundaries

| Use Case | LLM Role | Deterministic Fallback | Confidence Threshold |
|----------|----------|----------------------|---------------------|
| Device classification | Suggest type/vendor/model | sysObjectID/Redfish schema lookup | 0.7 |
| Placement decision | Suggest optimal placement | Rule-based (capacity + affinity) | 0.8 |
| Config generation | Suggest vendor-specific config | Template-based generation | 0.9 (production) |
| Oversight detection | Flag missing requirements | Checklist validation | 0.6 |
| Anomaly correlation | Suggest root cause | Threshold alerting | 0.7 |
| Cost optimization | Suggest savings | Budget threshold alerts | 0.7 |

<!-- AUDIT-FIX: C-03 — Prompt injection defense for device-reported data -->

## Input Sanitization: Device-Reported Data

### Mandatory Rule

**All device-reported data (sysDescr, hostnames, firmware strings, Redfish schema fields, SNMP MIB values, LLDP neighbor names, banner text) MUST pass through `ContentSanitizer` before inclusion in any LLM prompt.** Device-reported strings are attacker-controlled input — a compromised or malicious device can embed prompt injection payloads in any field it reports.

### Defense-in-Depth Strategy

1. **Input sanitization (first layer):** `ContentSanitizer` strips known-dangerous patterns from device-reported data before it enters any LLM prompt. This is a necessary but insufficient defense — novel injection patterns will bypass it.

2. **Output validation (primary defense):** Every LLM response is parsed into a **strict Go struct** (Rule 3 above). The LLM cannot execute arbitrary actions because its output is validated against a typed schema. Freeform text is never executed. This is the primary defense against prompt injection — even if injection succeeds in manipulating the LLM's reasoning, the output must conform to the expected struct or it is rejected.

3. **External multi-signal scoring (authority):** LLM confidence scores are **NEVER trusted alone** as the basis for action. All placement, classification, and configuration decisions use external multi-signal scoring that combines LLM output with deterministic signals (device fingerprint match, network topology validation, capacity checks). The external scoring system is the authority, not the LLM. See PLACEMENT-POLICY.md for the full scoring model.

### Go Types

```go
// ContentSanitizer strips dangerous patterns from device-reported strings
// before they are included in LLM prompts.
type ContentSanitizer struct {
    MaxLength         int            // Maximum output length (default: 256 chars)
    StripControlChars bool           // Remove ASCII control characters 0x00-0x1F, 0x7F (default: true)
    StripHTMLTags     bool           // Remove HTML/XML tags (default: true)
    StripInstructions bool           // Remove instruction-like patterns (default: true)
    InstructionPatterns []string     // Patterns that look like prompt instructions
}

// SanitizeForLLM cleans a raw device-reported string for safe inclusion
// in an LLM prompt. It applies all configured sanitization steps and
// truncates the result to MaxLength.
//
// Sanitization steps (in order):
//   1. Strip control characters (0x00-0x1F except \n and \t, 0x7F)
//   2. Strip HTML/XML tags (<...> patterns)
//   3. Strip instruction-like patterns ("ignore previous", "system:", "you are", etc.)
//   4. Normalize whitespace (collapse runs of whitespace to single space)
//   5. Truncate to MaxLength (default: 256 characters)
//
// This function is NOT a security boundary on its own — it is one layer
// of defense-in-depth. The primary defense is output validation (typed structs).
func (cs *ContentSanitizer) SanitizeForLLM(raw string) string {
    // Implementation applies steps 1-5 in order.
    // Returns sanitized string, never empty (returns "[sanitized:empty]" if input
    // reduces to nothing, so the LLM knows a value existed but was stripped).
    panic("not implemented")
}
```

### What ContentSanitizer Does NOT Protect Against

- Novel prompt injection techniques not covered by the pattern list
- Semantic manipulation (e.g., a device name of "HighPriorityProductionServer" influencing placement)
- Data exfiltration via LLM output (mitigated by typed output structs and no network access from LLM)

These residual risks are addressed by output validation (Rule 3) and external multi-signal scoring.

<!-- END AUDIT-FIX: C-03 -->

## Testing Requirements

- Every LLM-dependent code path has a test that runs with LLM disabled
- Integration tests verify deterministic fallback produces correct (if less optimal) results
- Fuzz tests verify that malformed LLM output is rejected by validation
