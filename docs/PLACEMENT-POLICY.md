# Placement Policy

> Addresses ADVERSARIAL-REVIEW.md Finding 6.2: The LLM might be 95% confident but completely wrong. Confidence != correctness. Hard constraints must override LLM recommendations regardless of confidence score.

## Problem

LLM-driven placement optimization is dangerous when the LLM is confidently wrong. A confidence threshold catches "I don't know" but not "I'm sure and I'm wrong." Examples:

- LLM recommends placing a database on a spot instance (cost optimization, confidence 0.91). Spot instance is terminated 2 hours later.
- LLM recommends placing a workload in US-East (lowest latency, confidence 0.88). The workload has an EU data residency requirement.
- LLM recommends a GPU-less node for an ML workload (cheaper, confidence 0.85). The workload fails at runtime.

**Solution:** Hard constraints are enforced deterministically BEFORE and AFTER LLM recommendations. The LLM optimizes within the feasible region defined by hard constraints. It cannot override them.

---

## Constraint Priority Order

Constraints are evaluated in strict priority order. Higher priority constraints are never violated to satisfy lower priority ones.

| Priority | Category | Type | Example |
|----------|----------|------|---------|
| 1 | Compliance | Hard | "Workload X must stay in EU" |
| 2 | Security | Hard | "Production never on shared tenancy" |
| 3 | Capability | Hard | "Requires GPU", "Requires NVMe" |
| 4 | Exclusion | Hard | "Never on spot/preemptible compute" |
| 5 | Budget | Hard | "Tenant Y budget is $5K/month" |
| 6 | Availability | Configurable | SLA-based — 99.9% requires multi-AZ |
| 7 | Affinity/Anti-affinity | Soft | "Co-locate with service Y" |
| 8 | Cost | Soft | "Minimize cost" |
| 9 | Performance | Soft | "Minimize latency" |

**Hard constraints** are pass/fail. A placement either satisfies them or it does not. There is no "close enough."

**Soft preferences** are optimization targets. The LLM (or deterministic fallback) optimizes across soft preferences within the feasible region defined by hard constraints.

---

## Go Types

```go
package placement

import "time"

// PlacementPolicy is a tenant-scoped or workload-scoped set of constraints
// and preferences that govern where infrastructure can be placed.
type PlacementPolicy struct {
    ID              string           `json:"id"`               // UUID v7
    TenantID        string           `json:"tenant_id"`
    Name            string           `json:"name"`             // "production-database-policy"
    Description     string           `json:"description"`
    HardConstraints []HardConstraint `json:"hard_constraints"` // must ALL be satisfied
    SoftPreferences []SoftPreference `json:"soft_preferences"` // optimize across these
    CreatedBy       string           `json:"created_by"`       // actor who created this policy
    CreatedAt       time.Time        `json:"created_at"`
    UpdatedAt       time.Time        `json:"updated_at"`
}

// ConstraintCategory classifies a hard constraint for priority ordering.
type ConstraintCategory string

const (
    ConstraintCategoryCompliance  ConstraintCategory = "compliance"
    ConstraintCategorySecurity    ConstraintCategory = "security"
    ConstraintCategoryCapability  ConstraintCategory = "capability"
    ConstraintCategoryExclusion   ConstraintCategory = "exclusion"
    ConstraintCategoryBudget      ConstraintCategory = "budget"
    ConstraintCategoryAvailability ConstraintCategory = "availability"
)

// HardConstraint is a non-negotiable placement rule. If a placement violates
// any hard constraint, the placement is rejected — regardless of LLM confidence.
type HardConstraint struct {
    ID          string             `json:"id"`           // UUID v7
    Category    ConstraintCategory `json:"category"`
    Name        string             `json:"name"`         // "eu_data_residency"
    Description string             `json:"description"`  // "All data must remain in EU regions"
    Rule        ConstraintRule     `json:"rule"`         // the actual constraint logic
    Enabled     bool               `json:"enabled"`
}

// ConstraintRule defines the logic of a hard constraint.
// Rules are evaluated deterministically — no LLM involvement.
type ConstraintRule struct {
    Field    string   `json:"field"`    // what to check: "region", "instance_type", "capability", "cost_monthly", "tenancy"
    Operator string   `json:"operator"` // "in", "not_in", "equals", "not_equals", "less_than", "greater_than", "has_capability"
    Values   []string `json:"values"`   // the constraint values: ["eu-west-1", "eu-central-1"], ["gpu"], ["dedicated"]
}

// SoftPreference is an optimization target. The LLM (or deterministic fallback)
// tries to satisfy these but they do not cause rejection.
type SoftPreference struct {
    ID          string  `json:"id"`
    Name        string  `json:"name"`        // "minimize_cost", "minimize_latency", "co_locate_with"
    Description string  `json:"description"`
    Objective   string  `json:"objective"`   // "minimize", "maximize", "target"
    Field       string  `json:"field"`       // "cost_monthly", "latency_ms", "availability_percent"
    Weight      float64 `json:"weight"`      // 0.0-1.0, relative importance among soft preferences
    TargetValue string  `json:"target_value"` // for "target" objective: the desired value
}

// PlacementRequest is submitted when a workload needs infrastructure.
type PlacementRequest struct {
    ID              string           `json:"id"`
    TenantID        string           `json:"tenant_id"`
    WorkloadType    string           `json:"workload_type"`     // "database", "web_server", "ml_training", "batch"
    Requirements    WorkloadRequirements `json:"requirements"`  // CPU, memory, storage, network
    PolicyID        string           `json:"policy_id"`         // which PlacementPolicy to apply
    CorrelationID   string           `json:"correlation_id"`
    RequestedBy     string           `json:"requested_by"`
    RequestedAt     time.Time        `json:"requested_at"`
}

// WorkloadRequirements describes what the workload needs from infrastructure.
type WorkloadRequirements struct {
    CPUCores      int      `json:"cpu_cores"`
    MemoryGB      int      `json:"memory_gb"`
    StorageGB     int      `json:"storage_gb"`
    StorageType   string   `json:"storage_type"`    // "ssd", "nvme", "hdd", "any"
    GPURequired   bool     `json:"gpu_required"`
    GPUType       string   `json:"gpu_type"`        // "nvidia_a100", "any", ""
    NetworkMbps   int      `json:"network_mbps"`
    Capabilities  []string `json:"capabilities"`    // additional required capabilities
}

// PlacementDecision is the output of the placement engine.
type PlacementDecision struct {
    ID                string            `json:"id"`
    RequestID         string            `json:"request_id"`
    TargetDeviceID    string            `json:"target_device_id"`   // where to place it
    TargetRegion      string            `json:"target_region"`
    Reasoning         string            `json:"reasoning"`          // why this placement was chosen
    Source            PlacementSource   `json:"source"`             // who made the decision
    Confidence        float64           `json:"confidence"`         // 0.0-1.0 (only meaningful for LLM source)
    ConstraintResults []ConstraintCheck `json:"constraint_results"` // every hard constraint, pass/fail
    PolicyViolations  []PolicyViolation `json:"policy_violations"`  // empty if placement is valid
    DecidedAt         time.Time         `json:"decided_at"`
}

// PlacementSource identifies whether the decision came from the LLM or
// the deterministic fallback.
type PlacementSource string

const (
    PlacementSourceLLM           PlacementSource = "llm"
    PlacementSourceDeterministic PlacementSource = "deterministic"
    PlacementSourceOperator      PlacementSource = "operator"  // human override
)

// ConstraintCheck records whether a specific hard constraint was satisfied.
type ConstraintCheck struct {
    ConstraintID   string `json:"constraint_id"`
    ConstraintName string `json:"constraint_name"`
    Category       ConstraintCategory `json:"category"`
    Passed         bool   `json:"passed"`
    ActualValue    string `json:"actual_value"`   // what the placement has
    ExpectedValues string `json:"expected_values"` // what the constraint requires
    Message        string `json:"message"`         // human-readable explanation
}

// PolicyViolation is produced when a placement decision violates a hard constraint.
type PolicyViolation struct {
    ConstraintID   string             `json:"constraint_id"`
    ConstraintName string             `json:"constraint_name"`
    Category       ConstraintCategory `json:"category"`
    Message        string             `json:"message"`   // "Placement in us-east-1 violates EU data residency constraint"
    Severity       string             `json:"severity"`  // always "hard" for hard constraints
}
```

---

## Validation Pipeline

Every placement decision — whether from the LLM, deterministic engine, or human operator — passes through the same validation pipeline:

```
PlacementRequest
    |
    v
[1. Load PlacementPolicy for tenant/workload]
    |
    v
[2. LLM generates recommendation (if available)]
    |  - LLM receives: available resources, soft preferences, hard constraint DESCRIPTIONS
    |  - LLM does NOT enforce constraints — it only optimizes
    |
    v
[3. Deterministic Constraint Validator]
    |  - Evaluates EVERY hard constraint against the proposed placement
    |  - Hard constraints are checked in priority order
    |  - First violation stops evaluation (fail-fast)
    |  - Returns: all ConstraintCheck results + any PolicyViolations
    |
    v
[4. Decision Gate]
    |  - If zero PolicyViolations: placement is approved, proceed to execution
    |  - If any PolicyViolation: placement is REJECTED
    |     - If LLM source: log the override, try deterministic fallback
    |     - If deterministic source: escalate to operator (no valid placement exists)
    |     - If operator source: warn but allow (operator override is explicit)
    |
    v
[5. Execution via Temporal workflow]
```

### LLM Override Logging

Every time a hard constraint overrides an LLM recommendation, the event is logged:

```go
// LLMOverrideEvent is published to NATS when a hard constraint rejects
// an LLM placement recommendation. This data is used to improve LLM
// prompt engineering and identify systematic LLM failures.
type LLMOverrideEvent struct {
    PlacementRequestID string            `json:"placement_request_id"`
    LLMRecommendation  string            `json:"llm_recommendation"`  // what the LLM suggested
    LLMConfidence      float64           `json:"llm_confidence"`
    Violations         []PolicyViolation `json:"violations"`           // why it was rejected
    FallbackUsed       PlacementSource   `json:"fallback_used"`       // what replaced it
    Timestamp          time.Time         `json:"timestamp"`
}
```

---

## Escalation: No Valid Placement

If no placement satisfies all hard constraints, the request **fails with an explanation**. LOOM does not force a placement into a bad position.

```go
// NoFeasiblePlacementError is returned when no available resource satisfies
// all hard constraints. The request cannot be fulfilled without relaxing constraints.
type NoFeasiblePlacementError struct {
    Code             string            `json:"code"`              // "NO_FEASIBLE_PLACEMENT"
    Message          string            `json:"message"`
    RequestID        string            `json:"request_id"`
    UnsatisfiableConstraints []PolicyViolation `json:"unsatisfiable_constraints"`
    CorrelationID    string            `json:"correlation_id"`
}
```

The operator sees: "No placement satisfies all constraints. The following constraints could not be met: [list]. Options: (1) relax constraint X, (2) add capacity in region Y, (3) contact admin to review policy."

---

## Default Policies

LOOM ships with a set of default hard constraints that apply unless overridden:

| Constraint | Category | Rule |
|-----------|----------|------|
| no_spot_for_stateful | exclusion | Stateful workloads (database, queue) never on spot/preemptible instances |
| no_shared_tenancy_prod | security | Production workloads never on shared-tenancy hosts |
| budget_cap_enforcement | budget | Placement cost cannot exceed tenant's remaining monthly budget |
| gpu_capability_match | capability | Workloads requiring GPU must be placed on GPU-equipped hosts |
| storage_type_match | capability | NVMe-required workloads must be placed on NVMe-equipped hosts |

These defaults can be disabled or extended per tenant via the PlacementPolicy API.

---

## Relationship to LLM-BOUNDARIES.md

This document extends Rule 6 from LLM-BOUNDARIES.md ("LLM suggests, deterministic systems execute") with concrete constraint types and a validation pipeline. The placement policy is the deterministic system that validates LLM suggestions.

The LLM's role in placement: **optimize within the feasible region defined by hard constraints.** It receives hard constraint descriptions as context (so it can try to avoid proposing invalid placements), but hard constraints are always enforced by the deterministic validator, never by the LLM itself.
