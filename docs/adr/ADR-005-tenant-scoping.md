# ADR-005: Tenant Scoping from Day One

## Status

Accepted

## Context

LOOM will serve multiple tenants (organizations, business units, or customers) from inception. Multi-tenancy can be added retroactively, but experience shows that bolting tenant isolation onto a single-tenant system creates pervasive technical debt that touches every layer: database queries, event routing, workflow scoping, API authorization, and audit logging.

## Decision

Tenant scoping is built into every layer from the first line of code. There is no single-tenant mode.

Implementation:

- **Every database table** has a `tenant_id` column, included in primary keys and all indexes. Row-level security (RLS) policies enforce isolation at the database level as a defense-in-depth measure.
- **Every event** carries `tenant_id` in its metadata. NATS subject hierarchy starts with tenant: `{tenant}.{domain}.{event_type}`.
- **Every Temporal workflow** is scoped to a tenant via search attributes. Workflow IDs include tenant prefix.
- **Every API request** has tenant context extracted from the JWT. The `X-Tenant-ID` header is set server-side, never by the client.
- **Every log entry** includes tenant context for audit and debugging.

## Consequences

- No code path exists that operates without tenant context (except system-level operations like migrations)
- Adding a new tenant is a configuration change, not a code change
- Cross-tenant queries are explicitly forbidden at the repository layer; admin/reporting queries require elevated privileges
- Slightly more complex initial development (every query includes `WHERE tenant_id = ?`)
- Significantly less technical debt compared to retrofitting multi-tenancy later
- Tenant isolation is testable from day one — integration tests run with multiple tenants to verify no data leakage
- Per-tenant resource quotas, rate limits, and billing are straightforward since tenant context is always available
