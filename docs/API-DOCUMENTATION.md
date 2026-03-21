# API Documentation Strategy

> How LOOM's REST API is documented, versioned, and consumed by clients.

---

## 1. OpenAPI Specification

### Generation Strategy

LOOM generates its OpenAPI 3.1 specification from Go source code using [huma](https://huma.rocks/). Huma is preferred over swaggo because:

- It generates OpenAPI 3.1 (not just 3.0).
- Schemas are derived directly from Go structs and validation tags — no comment-based annotations.
- Request/response types are enforced at compile time, not at documentation time.
- It supports the `Idempotency-Key` header, cursor pagination, and error response patterns from [API-CONVENTIONS.md](API-CONVENTIONS.md) natively.

### How It Works

```go
// API handler registration generates OpenAPI spec automatically
huma.Register(api, huma.Operation{
    OperationID: "list-devices",
    Method:      http.MethodGet,
    Path:        "/api/v1/devices",
    Summary:     "List devices",
    Description: "Returns a paginated list of devices for the authenticated tenant.",
    Tags:        []string{"Devices"},
}, func(ctx context.Context, input *ListDevicesInput) (*ListDevicesOutput, error) {
    // Handler implementation
})

// Input struct defines query parameters and headers
type ListDevicesInput struct {
    Cursor   string `query:"cursor" doc:"Pagination cursor from previous response"`
    Limit    int    `query:"limit" default:"50" minimum:"1" maximum:"200" doc:"Items per page"`
    Type     string `query:"type" enum:"server,switch,router,firewall,hypervisor,kvm,storage,appliance" doc:"Filter by device type"`
    Status   string `query:"status" enum:"discovered,active,degraded,unreachable,decommissioned" doc:"Filter by status"`
    VendorID string `query:"vendor" doc:"Filter by vendor name"`
}

// Output struct defines the response body
type ListDevicesOutput struct {
    Body struct {
        Items      []Device `json:"items"`
        NextCursor string   `json:"next_cursor,omitempty"`
        HasMore    bool     `json:"has_more"`
    }
}
```

### Spec Output

The OpenAPI spec is generated at build time and embedded in the binary:

```bash
# Generate the spec (CI runs this as part of the build)
./loom openapi > openapi.json

# Verify spec validity
npx @redocly/cli lint openapi.json
```

The generated spec is also committed to the repository at `api/openapi.json` so it can be diffed in PRs. Any PR that changes API handlers must include the updated spec.

---

## 2. API Versioning

### URL Path Versioning

```
/api/v1/devices
/api/v1/workflows
/api/v1/tenants
/api/v2/devices       (future, only when breaking changes are unavoidable)
```

### Versioning Rules

| Rule | Detail |
|------|--------|
| Version is in the URL path | `/api/v1/`, not in headers or query params |
| No breaking changes within a major version | Adding fields is fine. Removing or renaming fields requires `v2`. |
| Additive changes are always safe | New endpoints, new optional query params, new response fields |
| Deprecation before removal | Deprecated endpoints return `Sunset` and `Deprecation` headers for one minor release before removal in the next major |
| Old versions remain available | When `v2` launches, `v1` continues to work for the remainder of its support period |

### What Constitutes a Breaking Change

| Breaking | Not Breaking |
|----------|-------------|
| Removing a field from a response | Adding a new field to a response |
| Renaming a field | Adding a new optional query parameter |
| Changing a field's type | Adding a new endpoint |
| Removing an endpoint | Adding a new enum value |
| Changing error codes | Adding a new error code |
| Changing pagination from cursor to offset | Increasing the default page size |

---

## 3. Documentation Hosting

### Embedded Swagger UI

The LOOM binary serves interactive API documentation via an embedded Swagger UI:

```go
//go:embed swagger-ui/*
var swaggerUI embed.FS

//go:embed openapi.json
var openapiSpec []byte
```

Accessible at:
- `http://localhost:8080/docs` — Swagger UI (interactive)
- `http://localhost:8080/api/openapi.json` — Raw OpenAPI spec

This means **every LOOM deployment includes its own API documentation**, matching the exact version deployed. No external documentation hosting is needed. Air-gapped deployments get full API docs without internet access.

### Documentation Features

The embedded Swagger UI provides:
- Interactive request builder with authentication.
- Schema visualization for all request/response types.
- Example requests and responses auto-generated from Go struct tags.
- Try-it-out functionality against the running LOOM instance.

---

## 4. SDK Generation

### Client SDK Generation Pipeline

OpenAPI-generated client SDKs are published for three languages:

| Language | Generator | Package |
|----------|-----------|---------|
| Go | `oapi-codegen` | `github.com/org/loom-go-sdk` |
| Python | `openapi-generator` (python-pydantic-v1) | `loom-sdk` (PyPI) |
| TypeScript | `openapi-generator` (typescript-fetch) | `@loom/sdk` (npm) |

### Generation Process

```bash
# Generate Go client
oapi-codegen -package loom -generate types,client api/openapi.json > sdk/go/client.gen.go

# Generate Python client
openapi-generator generate \
  -i api/openapi.json \
  -g python \
  -o sdk/python \
  --additional-properties=packageName=loom_sdk

# Generate TypeScript client
openapi-generator generate \
  -i api/openapi.json \
  -g typescript-fetch \
  -o sdk/typescript \
  --additional-properties=npmName=@loom/sdk
```

### SDK Versioning

SDK versions track LOOM versions. A LOOM `v0.3.0` release produces SDK version `v0.3.0` for all languages.

### SDK Quality

- Generated SDKs include type definitions for all domain models (Device, Endpoint, Workflow, etc.).
- Error responses are typed (`LoomError` struct with `Code`, `Message`, `CorrelationID`).
- Pagination helpers handle cursor-based pagination automatically.
- SDKs are tested in CI against a running LOOM instance to catch generation regressions.

---

## 5. API Changelog

### Tracking API Changes

Every PR that modifies the OpenAPI spec triggers a diff report:

```bash
# CI step: diff the spec against main
npx @redocly/cli diff api/openapi.json main:api/openapi.json
```

The diff output is posted as a PR comment, categorized as:
- **Non-breaking**: New endpoints, new optional fields, new query params.
- **Breaking**: Removed fields, type changes, removed endpoints. These block merge unless the PR targets a new major version.

### Migration Guides

When a breaking change is introduced (major version bump), a migration guide is published:

```markdown
## Migrating from v1 to v2

### Device List Response

**v1 (deprecated)**:
```json
GET /api/v1/devices?offset=100&limit=50
{
  "devices": [...],
  "total": 1500
}
```

**v2**:
```json
GET /api/v2/devices?cursor=abc123&limit=50
{
  "items": [...],
  "next_cursor": "def456",
  "has_more": true
}
```

**What to change**: Replace `offset` with `cursor`. Store `next_cursor` from each response. Remove `total` count dependency.
```

---

## 6. Reference Documents

| Document | Relationship |
|----------|-------------|
| [API-CONVENTIONS.md](API-CONVENTIONS.md) | Request/response format rules, headers, error codes, pagination, rate limiting |
| [DOMAIN-MODEL.md](DOMAIN-MODEL.md) | Go struct definitions that generate the OpenAPI schemas |
| [SECURITY-MODEL.md](SECURITY-MODEL.md) | Authentication (JWT), authorization (RBAC), tenant isolation in API layer |
| [ERROR-MODEL.md](ERROR-MODEL.md) | Error classification and HTTP status mapping |
| [AUDIT-MODEL.md](AUDIT-MODEL.md) | How API calls are recorded in the audit trail |

---

## 7. API Design Principles

These principles govern all API design decisions:

1. **Predictable**: Every endpoint follows the same patterns from [API-CONVENTIONS.md](API-CONVENTIONS.md). If you know one endpoint, you know them all.
2. **Tenant-scoped**: Every response contains only data belonging to the authenticated tenant. Cross-tenant data is never returned.
3. **Idempotent**: All mutating operations require an `Idempotency-Key` header. Retrying a request with the same key produces the same result.
4. **Traceable**: Every request gets a `X-Correlation-ID` (generated if not provided) that propagates through Temporal workflows, adapter calls, and NATS events.
5. **Self-documenting**: The running binary serves its own docs. No external documentation to go stale.
6. **Evolvable**: Additive changes are always safe. Breaking changes require a major version bump and a migration guide.
