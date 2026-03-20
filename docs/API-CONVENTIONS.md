# API Conventions

REST API contract stability rules.

## Versioning

- Prefix: `/api/v1/`
- Backward-compatible within major version
- Breaking changes require new major version (`/api/v2/`)

## Request/Response Format

- JSON only (`Content-Type: application/json`)
- UTC timestamps in RFC 3339 (`2024-03-20T10:00:00Z`)
- UUIDs for all resource IDs

## Pagination

Cursor-based (not offset-based — offset breaks with concurrent inserts).

```json
GET /api/v1/devices?cursor=<opaque_token>&limit=50

Response:
{
  "items": [...],
  "next_cursor": "abc123",
  "has_more": true
}
```

## Filtering

- Query params: `?tenant_id=&type=&status=&vendor=`
- Consistent naming across all endpoints

## Error Response

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Device with ID abc-123 not found",
    "details": [],
    "correlation_id": "req-456"
  }
}
```

HTTP status mapping:

| Status | Meaning |
|--------|---------|
| 400 | Validation error |
| 401 | Authentication failure |
| 403 | Authorization failure |
| 404 | Resource not found |
| 409 | Conflict |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

## Headers

- `X-Correlation-ID`: propagated through all layers (generated if not provided)
- `X-Tenant-ID`: extracted from JWT, not set by client
- `Idempotency-Key`: required for POST/PUT/DELETE (UUID)
- `Authorization: Bearer <JWT>`

## Rate Limiting

- Per-tenant, Valkey-backed
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- 429 response with `Retry-After` header
