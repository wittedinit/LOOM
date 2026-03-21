# Cost Model

> Cost-aware orchestration contract for LOOM. Every placement decision, every resource allocation, every workload migration carries a cost. LOOM makes that cost visible, comparable, and enforceable.

---

## Core Principles

1. **Cost is a first-class dimension.** Cost is not an afterthought bolted onto placement. It sits alongside availability, latency, and capacity in every orchestration decision.
2. **All substrates are comparable.** Bare metal, cloud VMs, edge nodes -- LOOM normalizes cost into a common model so they can be compared on equal terms.
3. **Tenants own their costs.** Every resource consumed is attributed to exactly one tenant. Shared resources use proportional allocation. No unattributed costs.
4. **Budget enforcement is policy, not prayer.** Hard caps are enforced in the orchestration pipeline. Soft caps trigger alerts. Projections warn before overspend happens.
5. **LLM sees cost data.** The LLM decision engine receives cost estimates as structured context. It does not query cost APIs directly -- it receives pre-computed cost facts.

---

## Types

```go
package cost

import "time"

// CostDimension classifies a cost line item.
type CostDimension string

const (
    DimensionCompute     CostDimension = "compute"       // CPU, RAM, GPU
    DimensionStorage     CostDimension = "storage"        // disk, object store
    DimensionNetwork     CostDimension = "network"        // transit, peering, cross-connect
    DimensionPower       CostDimension = "power"          // electricity consumption
    DimensionCooling     CostDimension = "cooling"        // included in PUE, tracked separately for visibility
    DimensionRackSpace   CostDimension = "rack_space"     // physical U-space
    DimensionLicense     CostDimension = "license"        // software licenses (NOS, hypervisor, etc.)
    DimensionMaintenance CostDimension = "maintenance"    // hardware support contracts
    DimensionEgress      CostDimension = "egress"         // cloud data transfer out
    DimensionAPICall     CostDimension = "api_call"       // per-request cloud API charges
)

// CostProfile defines the cost characteristics of a resource or substrate.
type CostProfile struct {
    ID           string              `json:"id"`            // UUID v7
    TenantID     string              `json:"tenant_id"`
    Name         string              `json:"name"`          // "us-east-rack-12", "aws-us-east-1", "edge-site-chicago"
    SubstrateType string             `json:"substrate_type"` // "bare_metal", "cloud_vm", "cloud_container", "edge"
    Provider     string              `json:"provider"`      // "self", "aws", "gcp", "azure", "colo_provider"
    Location     string              `json:"location"`      // datacenter, region, or site identifier
    LineItems    []CostLineItem      `json:"line_items"`    // individual cost components
    EffectiveAt  time.Time           `json:"effective_at"`  // when this profile takes effect
    ExpiresAt    *time.Time          `json:"expires_at"`    // nil = no expiry
    UpdatedAt    time.Time           `json:"updated_at"`
}

// CostLineItem is a single cost component within a profile.
type CostLineItem struct {
    Dimension   CostDimension `json:"dimension"`
    Description string        `json:"description"`   // "Dell R750 amortized over 60 months"
    UnitCost    float64       `json:"unit_cost"`      // cost per unit per period
    Unit        string        `json:"unit"`           // "per_month", "per_hour", "per_gb", "per_mbps", "per_kwh", "per_request"
    Quantity    float64       `json:"quantity"`       // how many units (e.g., watts, GB, U-spaces)
    Currency    string        `json:"currency"`       // ISO 4217: "USD", "EUR", etc.
}

// CostEstimate is the projected cost for a specific workload on a specific substrate.
type CostEstimate struct {
    ID            string              `json:"id"`
    TenantID      string              `json:"tenant_id"`
    WorkloadRef   string              `json:"workload_ref"`    // workflow or workload ID
    ProfileID     string              `json:"profile_id"`      // which CostProfile was used
    SubstrateType string              `json:"substrate_type"`
    Period        string              `json:"period"`          // "hourly", "monthly", "yearly"
    Breakdown     []CostBreakdownItem `json:"breakdown"`       // per-dimension costs
    TotalCost     float64             `json:"total_cost"`
    Currency      string              `json:"currency"`
    Confidence    float64             `json:"confidence"`      // 0.0-1.0; lower for spot/variable pricing
    EstimatedAt   time.Time           `json:"estimated_at"`
    Assumptions   []string            `json:"assumptions"`     // "assumes 70% avg CPU utilization", etc.
}

// CostBreakdownItem shows cost for one dimension within an estimate.
type CostBreakdownItem struct {
    Dimension CostDimension `json:"dimension"`
    Amount    float64       `json:"amount"`
    Unit      string        `json:"unit"`
    Detail    string        `json:"detail"` // human-readable explanation
}

// CostComparison ranks substrates by cost for a given workload.
type CostComparison struct {
    ID          string          `json:"id"`
    TenantID    string          `json:"tenant_id"`
    WorkloadRef string          `json:"workload_ref"`
    Estimates   []CostEstimate  `json:"estimates"`    // sorted by TotalCost ascending
    CheapestID  string          `json:"cheapest_id"`  // profile ID of the lowest-cost option
    ComparedAt  time.Time       `json:"compared_at"`
}

// BudgetPolicy defines cost guardrails for a tenant.
type BudgetPolicy struct {
    ID              string        `json:"id"`
    TenantID        string        `json:"tenant_id"`
    Name            string        `json:"name"`           // "production-monthly", "dev-team-quarterly"
    Period          string        `json:"period"`          // "monthly", "quarterly", "yearly"
    HardCap         *float64      `json:"hard_cap"`        // reject operations that would exceed this
    SoftCap         *float64      `json:"soft_cap"`        // alert when current spend exceeds this
    ProjectedCap    *float64      `json:"projected_cap"`   // alert when projected spend exceeds this
    Currency        string        `json:"currency"`
    Dimensions      []CostDimension `json:"dimensions"`    // which dimensions this policy covers (empty = all)
    EnforcementMode string        `json:"enforcement_mode"` // "enforce", "alert_only", "disabled"
    CreatedAt       time.Time     `json:"created_at"`
    UpdatedAt       time.Time     `json:"updated_at"`
}
```

---

## Bare Metal Cost Model

Bare metal costs are amortized and normalized to monthly values.

### Hardware Amortization

```
monthly_hardware_cost = purchase_price / useful_life_months
```

| Component | Typical Useful Life | Example |
|-----------|-------------------|---------|
| Server chassis | 60 months (5 years) | $12,000 / 60 = $200/month |
| Network switch | 60 months | $8,000 / 60 = $133/month |
| Storage array | 48 months (4 years) | $30,000 / 48 = $625/month |
| GPU accelerator | 36 months (3 years) | $15,000 / 36 = $417/month |

### Power

```
monthly_power_cost = avg_watts * hours_per_month * cost_per_kwh * PUE / 1000
```

| Parameter | Typical Value |
|-----------|--------------|
| `avg_watts` | Measured via IPMI/Redfish sensors, or estimated from TDP |
| `hours_per_month` | 730 (24 * 365 / 12) |
| `cost_per_kwh` | $0.08 - $0.15 (varies by region) |
| `PUE` | 1.2 - 1.8 (Power Usage Effectiveness; includes cooling overhead) |

PUE incorporates cooling costs. A PUE of 1.4 means 40% overhead for cooling and power distribution on top of IT load. Cooling is not calculated separately -- it is embedded in PUE.

### Rack Space

```
monthly_rack_cost = rack_units * cost_per_u_per_month
```

| Parameter | Typical Value |
|-----------|--------------|
| `rack_units` | 1U, 2U, 4U per server; 1U per switch |
| `cost_per_u_per_month` | $50 - $200 (varies by facility tier) |

### Maintenance Contracts

Hardware support contracts (next-business-day, 4-hour, etc.) are a monthly cost per device.

```
monthly_maintenance = annual_contract_cost / 12
```

---

## Cloud Cost Model

Cloud costs are pulled from provider pricing APIs and normalized.

### Compute Pricing Tiers

| Tier | Description | Discount vs On-Demand | Confidence |
|------|-------------|----------------------|------------|
| On-Demand | Pay per hour, no commitment | Baseline | 1.0 |
| Reserved (1-year) | Committed capacity, billed monthly | ~30-40% off | 0.95 |
| Reserved (3-year) | Committed capacity, upfront or monthly | ~50-60% off | 0.95 |
| Spot/Preemptible | Spare capacity, can be reclaimed | ~60-90% off | 0.3 |

Spot pricing confidence is low because availability is not guaranteed.

### Data Transfer

| Direction | Cost |
|-----------|------|
| Ingress | Free (all major providers) |
| Egress (to internet) | $0.05 - $0.12 per GB (varies by volume and provider) |
| Egress (cross-region) | $0.01 - $0.02 per GB |
| Egress (same region, cross-AZ) | $0.01 per GB |

### Storage

Billed per GB per month. Type determines price: SSD ($0.08-0.17/GB), HDD ($0.02-0.05/GB), object storage ($0.02-0.03/GB).

### API Calls

Some cloud resources incur per-request charges (e.g., S3 GET/PUT, Lambda invocations). These are tracked under the `api_call` dimension.

---

## Network Cost Model

| Component | Pricing Model | Typical Cost |
|-----------|--------------|-------------|
| Transit (commodity internet) | $/Mbps committed | $0.50 - $2.00 per Mbps/month |
| Peering (direct) | Free or nominal | $0 - $500/month per peer |
| Cross-connect (colo) | $/month per cable | $200 - $500/month |
| Bandwidth commitment | Committed rate + burst | Base rate + $0.01-0.05/GB overage |

Network costs are attributed to the tenant whose traffic traverses the link. For shared uplinks, costs are split proportionally by measured bandwidth consumption.

---

## Per-Tenant Cost Attribution

### Dedicated Resources

Resources exclusively assigned to a tenant are directly attributed. The full cost of the resource is assigned to the tenant.

### Shared Resources

Shared infrastructure (network switches, shared storage, management plane) is allocated proportionally.

```
tenant_share = tenant_usage / total_usage * total_cost
```

Usage metrics vary by resource type:

| Resource Type | Usage Metric |
|---------------|-------------|
| Shared compute | CPU-hours consumed |
| Shared storage | GB-hours stored |
| Shared network | Bytes transferred |
| Management plane | Number of managed devices |

### Unattributable Costs

Some costs cannot be attributed to a single tenant (e.g., facility lease, staff). These are tracked under a `_platform` pseudo-tenant and excluded from tenant-facing cost reports. They may be allocated to tenants as a platform fee if the operator configures one.

---

## Cost Comparison

Given a workload specification (CPU, RAM, storage, network, duration), LOOM calculates cost on every available substrate and returns a ranked list.

**Comparison inputs:**

```go
type WorkloadSpec struct {
    CPUCores    int           `json:"cpu_cores"`
    MemoryGB    int           `json:"memory_gb"`
    StorageGB   int           `json:"storage_gb"`
    StorageType string        `json:"storage_type"`   // "ssd", "hdd", "nvme"
    NetworkMbps int           `json:"network_mbps"`
    EgressGB    int           `json:"egress_gb"`       // expected monthly egress
    Duration    time.Duration `json:"duration"`        // how long the workload runs
    GPUs        int           `json:"gpus"`
    Preemptible bool          `json:"preemptible"`     // can tolerate interruption
}
```

**Comparison procedure:**

1. Enumerate all `CostProfile` entries for the tenant.
2. For each profile, calculate the cost of running the workload for the specified duration.
3. Filter out profiles that cannot satisfy the workload requirements (e.g., not enough CPU, no GPU).
4. Sort remaining profiles by `TotalCost` ascending.
5. Return the full `CostComparison` including breakdown per substrate.

---

## Budget Enforcement

Budget policies are evaluated at workflow submission time.

### Enforcement Modes

| Mode | Behavior |
|------|----------|
| **Hard cap** | Workflow is rejected if estimated cost would push spend over the cap. API returns `409 Conflict` with cost details. |
| **Soft cap** | Workflow is accepted but an alert event is emitted: `loom.{tenant}.cost.soft_cap_exceeded`. |
| **Projected cap** | Workflow is accepted. If projected end-of-period spend (current + estimated remaining) exceeds the cap, an alert is emitted: `loom.{tenant}.cost.projected_cap_exceeded`. |
| **Disabled** | Policy exists but is not enforced. Used for tracking without blocking. |

### Evaluation Order

1. Check hard caps first. If any hard cap is exceeded, reject immediately.
2. Check soft caps. Emit alerts for any exceeded soft caps.
3. Check projected caps. Emit alerts for any exceeded projections.
4. If no hard cap is violated, the workflow proceeds.

---

## Cost Projection

LOOM projects future spend using historical data.

**Method:** Linear regression over the trailing 90-day window, with optional seasonal adjustment.

```
projected_end_of_period = current_spend + (daily_rate * remaining_days)
```

**Seasonal adjustment:** If the tenant has 12+ months of history, LOOM applies a seasonal factor derived from the same period in the prior year.

```
seasonal_factor = last_year_this_month / last_year_average_month
projected_adjusted = projected_end_of_period * seasonal_factor
```

Projections are recalculated daily and stored for trend analysis. If a projection exceeds a `ProjectedCap`, an event is emitted.

---

## LLM Integration

The LLM decision engine receives cost data as structured context when making placement and optimization decisions. The LLM never queries cost APIs directly.

**Context provided to LLM:**

```go
type CostContext struct {
    WorkloadSpec    WorkloadSpec    `json:"workload_spec"`
    Comparison      CostComparison `json:"cost_comparison"`      // ranked substrates
    BudgetRemaining float64        `json:"budget_remaining"`      // how much of the hard cap is left
    CurrentSpend    float64        `json:"current_spend"`         // spend so far this period
    ProjectedSpend  float64        `json:"projected_spend"`       // projected end-of-period spend
    CheapestOption  string         `json:"cheapest_option"`       // substrate name
    Constraints     []string       `json:"constraints"`           // "no spot instances", "prefer bare metal", etc.
}
```

The LLM may recommend a substrate that is not the cheapest if other factors (latency, availability, compliance) justify the premium. The recommendation must include cost reasoning in its `Reasoning` field (see [LLM-BOUNDARIES.md](LLM-BOUNDARIES.md)).

**Rules:**
- LLM recommendations that exceed a hard cap are rejected by the deterministic dispatcher, regardless of LLM confidence.
- If the LLM is unavailable, the cost comparison engine selects the cheapest substrate that meets workload requirements. No LLM means no cost-optimization intelligence, but the system still functions.
