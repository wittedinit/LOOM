# ADR-007: LLM Suggests, Never Executes

## Status

Accepted

## Context

LOOM integrates LLM capabilities (via Anthropic's Claude) for device classification, config generation, anomaly analysis, and optimization suggestions. LLMs can hallucinate, produce inconsistent outputs, and lack the determinism required for infrastructure operations. Combined with Temporal's durable execution (which replays workflows), non-deterministic LLM calls inside workflows would break replay semantics. The boundary between LLM intelligence and deterministic execution must be explicit and enforced.

## Decision

LLM is restricted to a suggestion/advisory role. It never executes operations, never bypasses validation, and every LLM output is typed and carries confidence metadata.

Enforcement rules:

- **LLM never performs execution directly**: LLM produces a typed recommendation struct. A deterministic dispatcher validates and executes it.
- **LLM never bypasses validation**: Every LLM-generated config or plan passes through the same validation pipeline as human-authored input.
- **LLM output is always typed**: Every LLM response is parsed into a concrete Go struct with confidence score and provenance. Never executed as freeform text.
- **Confidence thresholds gate action**: Low confidence (< 0.7) triggers human review. Very low (< 0.4) falls back to deterministic alternatives.
- **Every path works without LLM**: If the LLM is unavailable, degraded, or untrusted, deterministic fallbacks handle every operation (fingerprinting for classification, templates for config generation, rules for placement).
- **LLM calls are Temporal activities**: LLM interactions happen in activities (which tolerate non-determinism), never in workflow code (which must be deterministic for replay).

See `/docs/LLM-BOUNDARIES.md` for the complete enforcement specification.

## Consequences

- LLM enhances but never controls infrastructure operations
- Hallucinated configs are caught by the same validation that catches human errors
- Temporal workflow replay is safe because LLM calls are isolated in activities with cached results
- Every LLM-dependent feature has a tested deterministic fallback
- Slightly more development effort per feature (must build both LLM path and fallback path)
- LLM improvements are additive — they make suggestions better without changing execution guarantees
- Audit trail includes LLM reasoning alongside deterministic validation results
