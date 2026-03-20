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

## Testing Requirements

- Every LLM-dependent code path has a test that runs with LLM disabled
- Integration tests verify deterministic fallback produces correct (if less optimal) results
- Fuzz tests verify that malformed LLM output is rejected by validation
