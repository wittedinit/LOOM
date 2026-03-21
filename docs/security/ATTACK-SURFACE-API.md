# LOOM API Attack Surface Analysis

**Date:** 2026-03-21
**Scope:** REST API authentication, authorization, and input handling
**Code:** `/tmp/loom-security-api/` (exploit + defense Go code, all tests passing)

---

## Summary

LOOM exposes a multi-tenant API that controls physical infrastructure (switches, BMCs, PiKVMs). Any API vulnerability could allow an attacker to access, modify, or disrupt devices belonging to another tenant. This document covers 10 attack categories with working exploit code, tested defenses, and findings.

**Test results: 22/22 tests passing.** All defenses validated.

---

## 1. JWT Token Attacks

**Risk: CRITICAL** -- JWT is the sole authentication mechanism for API access.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| `alg: none` confusion | `attacks/jwt_attacks.go:CraftAlgNoneToken` | Blocked |
| HS256-with-RSA-public-key switching | `attacks/jwt_attacks.go:CraftHS256WithPublicKeyToken` | Blocked |
| Expired token replay | `attacks/jwt_attacks.go:CraftExpiredToken` | Blocked |
| Tenant claim manipulation | `attacks/jwt_attacks.go:CraftTenantSwapToken` | **Requires authz layer** |
| `kid` header path traversal | `attacks/jwt_attacks.go:CraftKidInjectionToken` | Blocked |
| Token replay (same JTI reused) | Tested via `shared.MakeValidToken` | Blocked |

### Finding: Tenant Claim Manipulation

A properly-signed JWT with a forged `tenant_id` claim will pass JWT validation because the signature is valid. **JWT validation alone does not prevent cross-tenant access.** The authorization service must independently verify that `user_id` belongs to the claimed `tenant_id` (e.g., via database lookup on every request, or by having the auth service be the sole issuer of tenant claims after verifying membership).

### Defenses Implemented

**File:** `defenses/jwt_defenses.go`

- **Algorithm allowlist**: `jwt.WithValidMethods([]string{"RS256"})` -- no `none`, no `HS256`
- **kid validation**: Regex `^[a-zA-Z0-9_-]{1,64}$` blocks path traversal (`../../../../etc/passwd`)
- **Key store**: Only pre-registered public keys are accepted; kid maps to in-memory store, never to filesystem
- **Expiry enforcement**: `jwt.WithExpirationRequired()` with 5-second leeway
- **Issuer/audience validation**: Hardcoded checks for `loom-auth` and `loom-api`
- **JTI replay blacklist**: Each token ID is tracked; reuse is rejected
- **Required tenant_id claim**: Empty tenant_id is rejected

---

## 2. IDOR (Insecure Direct Object Reference)

**Risk: CRITICAL** -- Allows cross-tenant device access/modification.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| GET device from another tenant | `attacks/idor_attacks.go:IDORGetDevice` | Blocked |
| GET workflow from another tenant | `attacks/idor_attacks.go:IDORGetWorkflow` | Blocked |
| PATCH device from another tenant | `attacks/idor_attacks.go:IDORModifyDevice` | Blocked |
| UUID enumeration | `attacks/idor_attacks.go:IDOREnumerateDevices` | Mitigated |

### Key Design Requirement

Every database query MUST include `WHERE tenant_id = $1` sourced from the JWT, never from URL parameters or request body. The error response for "wrong tenant" and "does not exist" MUST be identical to prevent enumeration.

### Defenses Implemented

**File:** `defenses/idor_defenses.go`

- `GetDeviceForTenant(deviceID, tenantID)` returns the same `"device not found"` error regardless of whether the device exists but belongs to another tenant, or does not exist at all
- Handler extracts `tenant_id` from JWT context, never from the request
- UUID v4 (not v7) recommended for resource IDs to maximize enumeration resistance

---

## 3. Mass Assignment / Over-Posting

**Risk: HIGH** -- Could allow tenant escape or privilege escalation.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| PATCH with `tenant_id`, `is_admin` | `attacks/mass_assignment_attacks.go:MassAssignmentPatchDevice` | Blocked |
| POST with pre-set `id`, `tenant_id` | `attacks/mass_assignment_attacks.go:MassAssignmentCreateDevice` | Blocked |

### Defenses Implemented

**File:** `defenses/mass_assignment_defenses.go`

- **Explicit allowlist structs**: `DevicePatchRequest` has only `Name` and `Address` -- no `TenantID`, `IsAdmin`, `ID`
- **`json.Decoder.DisallowUnknownFields()`**: Any field not in the struct is rejected with HTTP 400
- **Server-generated fields**: `ID` is `uuid.New()`, `TenantID` comes from JWT, `IsAdmin` defaults to `false`
- **No trailing data**: Decoder checks `dec.More()` to reject payloads with extra content

---

## 4. Rate Limiting and Brute Force

**Risk: HIGH** -- Credential stuffing and API key enumeration.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| Password brute force | `attacks/ratelimit_attacks.go:BruteForceLogin` | Mitigated |
| API key timing analysis | `attacks/ratelimit_attacks.go:TimingAttackAPIKey` | Mitigated |

### Finding: Timing Side-Channel

`NaiveKeyCompare` (using `==`) short-circuits on the first mismatched byte, leaking information about valid key prefixes. An attacker making thousands of requests can statistically determine valid key bytes.

### Defenses Implemented

**File:** `defenses/ratelimit_defenses.go`

- **Per-IP rate limiting**: Token bucket via `golang.org/x/time/rate` with configurable rate and burst
- **Per-account lockout**: 5 failed attempts triggers 15-minute lockout
- **Constant-time comparison**: `crypto/subtle.ConstantTimeCompare` for all secret comparisons
- **Length-independent timing**: Even mismatched lengths use a dummy comparison to prevent length leaks

---

## 5. Request Smuggling / Header Injection

**Risk: HIGH** -- Can bypass authentication/authorization entirely.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| CL-TE request smuggling | `attacks/smuggling_attacks.go:HTTPRequestSmuggling` | Blocked |
| X-Tenant-ID injection | `attacks/smuggling_attacks.go:TenantHeaderInjection` | Blocked |
| CRLF header injection | `attacks/smuggling_attacks.go:CRLFHeaderInjection` | Blocked (Go stdlib) |
| X-Forwarded-For spoofing | Tested via header injection | Blocked |

### Defenses Implemented

**File:** `defenses/smuggling_defenses.go`

- **Strip untrusted headers**: Middleware deletes `X-Tenant-ID`, `X-Forwarded-For`, `X-Real-IP`, `X-Original-URL` before they reach handlers
- **Reject ambiguous encoding**: Requests with both `Content-Length` and `Transfer-Encoding` return 400
- **CRLF validation**: Header values containing `\r` or `\n` are rejected
- **Tenant from JWT only**: Middleware rejects any request that includes an `X-Tenant-ID` header

---

## 6. CORS Misconfiguration

**Risk: HIGH** -- Allows malicious websites to steal data using logged-in user's session.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| Preflight from evil origin | `attacks/cors_attacks.go:CORSProbe` | Blocked |
| Origin reflection check | `attacks/cors_attacks.go:CORSProbeResult.IsVulnerable` | Not vulnerable |
| Cross-origin credential theft | `attacks/cors_attacks.go:CORSExploitPayload` | Blocked |

### Defenses Implemented

**File:** `defenses/cors_defenses.go`

- **Strict origin allowlist**: Only `dashboard.loom.example.com` and `admin.loom.example.com`
- **No wildcard origin**: `Access-Control-Allow-Origin: *` is never used
- **No origin reflection**: Unknown origins get no CORS headers at all (browser blocks the response)
- **SameSite cookies**: `SameSite=Strict`, `HttpOnly=true`, `Secure=true` on all session cookies

---

## 7. GraphQL / API Introspection Abuse

**Risk: MEDIUM** -- Schema leak enables targeted attacks; depth bombs cause DoS.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| Introspection query | `attacks/graphql_attacks.go:GraphQLIntrospection` | Blocked |
| Depth bomb (nested query) | `attacks/graphql_attacks.go:GraphQLDepthBomb` | Blocked |
| Batch query amplification | `attacks/graphql_attacks.go:GraphQLBatchAttack` | Blocked |

### Defenses Implemented

**File:** `defenses/graphql_defenses.go`

- **Introspection disabled**: Queries containing `__schema` or `__type` are rejected
- **Max query depth**: 5 levels (configurable)
- **Max batch size**: 5 queries per request
- **Middleware pattern**: `GraphQLSecurityMiddleware` wraps the GraphQL handler

---

## 8. Webhook / Callback SSRF

**Risk: CRITICAL** -- SSRF can leak cloud credentials, access internal services, read local files.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| AWS metadata (169.254.169.254) | `attacks/ssrf_attacks.go:WebhookSSRFPayloads` | Blocked |
| GCP/Azure metadata | Same | Blocked |
| Internal services (localhost) | Same | Blocked |
| `file:///etc/passwd` | Same | Blocked |
| IPv6 loopback (`[::1]`) | Same | Blocked |
| Hex/octal IP bypass | Same | Blocked |
| DNS rebinding | Same | Blocked |
| Redirect chain SSRF | `attacks/ssrf_attacks.go:RedirectChainSSRF` | Mitigated |

### Defenses Implemented

**File:** `defenses/ssrf_defenses.go`

- **HTTPS only**: No `http://`, `file://`, `gopher://`, etc.
- **No IP-based URLs**: `net.ParseIP` rejects all IP addresses (prevents hex/octal/IPv6 bypass)
- **DNS resolution check**: After resolving the hostname, all returned IPs are checked against private ranges
- **Private IP deny list**: RFC1918, link-local (169.254.0.0/16), loopback, IPv6 ULA/link-local, carrier-grade NAT
- **Blocked hostnames**: `metadata.google.internal` explicitly blocked
- **No redirect following**: HTTP client should not follow redirects (prevents open-redirect chaining)
- **No credentials in URL**: `user:pass@host` syntax is rejected

---

## 9. Pagination / Export Data Leakage

**Risk: MEDIUM** -- Mass data exfiltration, cross-tenant cursor manipulation.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| `page_size=999999999` | `attacks/pagination_attacks.go:PaginationDump` | Capped to 100 |
| Negative page/size | `attacks/pagination_attacks.go:NegativePageAttack` | Sanitized to defaults |
| Cursor tampering | `attacks/pagination_attacks.go:CursorManipulation` | Blocked (HMAC) |
| Export-all bypass | `attacks/pagination_attacks.go:ExportAllAttack` | Requires tenant filter |

### Defenses Implemented

**File:** `defenses/pagination_defenses.go`

- **Max page size**: Capped at 100 (configurable), default 25
- **Input sanitization**: Negative values default to 1 (page) and 25 (size)
- **HMAC-signed cursors**: Cursors contain `tenant_id`, `offset`, and `resource` signed with a server-side secret
- **Cross-tenant cursor check**: Cursor's embedded `tenant_id` must match the requesting JWT's `tenant_id`
- **Tamper detection**: Modified cursors fail HMAC verification

---

## 10. Error Message Information Disclosure

**Risk: MEDIUM** -- Leaks guide attackers to further exploits.

### Attacks Attempted

| Attack | File | Result |
|--------|------|--------|
| SQL error trigger | `attacks/error_disclosure_attacks.go:TriggerSQLError` | Generic response |
| Panic/stack trace | `attacks/error_disclosure_attacks.go:TriggerPanicError` | Generic response |
| Path traversal 404 | `attacks/error_disclosure_attacks.go:TriggerNotFoundWithDetails` | Generic response |
| Server header leak | `attacks/error_disclosure_attacks.go:CheckServerHeaders` | Headers stripped |

### Defenses Implemented

**File:** `defenses/error_defenses.go`

- **Recovery middleware**: Catches panics, logs full details server-side, returns generic `{"error":"internal server error","correlation_id":"..."}` to client
- **Correlation IDs**: Every response includes `X-Correlation-ID` so operators can find detailed logs without exposing them
- **Server header stripping**: `Server`, `X-Powered-By`, `X-AspNet-Version`, `X-Runtime`, `X-Debug-Token` are removed
- **SafeErrorResponse helper**: Centralized function that logs internal errors and returns generic messages

---

## Architecture Recommendations

### Middleware Stack Order

Apply middleware in this order (outermost first):

```
1. RecoveryMiddleware          -- catch panics
2. StripServerHeaders          -- remove tech headers
3. StripUntrustedHeaders       -- remove X-Tenant-ID etc.
4. ValidateHeaderValues        -- CRLF check
5. RejectAmbiguousTE           -- CL+TE check
6. RateLimitMiddleware         -- per-IP throttling
7. CORSMiddleware              -- origin validation
8. JWTValidationMiddleware     -- authentication
9. TenantFromJWTOnlyMiddleware -- authz setup
10. Handler                    -- business logic
```

### Database Query Pattern

Every query MUST be tenant-scoped:

```sql
-- CORRECT: tenant from JWT, never from request
SELECT * FROM devices WHERE id = $1 AND tenant_id = $2

-- WRONG: no tenant filter
SELECT * FROM devices WHERE id = $1
```

### Checklist for New Endpoints

- [ ] JWT validation with strict algorithm allowlist
- [ ] Tenant_id extracted from JWT, not request
- [ ] Strict struct binding with DisallowUnknownFields
- [ ] Page size capped, cursors signed
- [ ] Generic error responses with correlation IDs
- [ ] Rate limiting applied
- [ ] No technology-revealing headers

---

## File Index

| File | Purpose |
|------|---------|
| `shared/types.go` | Claims, device/workflow models, test key generation |
| `attacks/jwt_attacks.go` | JWT alg-none, alg-switching, replay, tenant swap, kid injection |
| `attacks/idor_attacks.go` | Cross-tenant GET/PATCH, UUID enumeration |
| `attacks/mass_assignment_attacks.go` | Over-posting with tenant_id, is_admin |
| `attacks/ratelimit_attacks.go` | Brute force, timing side-channel |
| `attacks/smuggling_attacks.go` | CL-TE smuggling, header injection, CRLF |
| `attacks/cors_attacks.go` | Origin probing, credential theft payload |
| `attacks/graphql_attacks.go` | Introspection, depth bomb, batch amplification |
| `attacks/ssrf_attacks.go` | Cloud metadata, internal services, file://, DNS rebinding |
| `attacks/pagination_attacks.go` | Page size dump, cursor tampering, negative values |
| `attacks/error_disclosure_attacks.go` | SQL error, panic, path traversal, header leak |
| `defenses/jwt_defenses.go` | StrictJWTValidator, KeyStore, JTIBlacklist |
| `defenses/idor_defenses.go` | Tenant-scoped store, uniform error messages |
| `defenses/mass_assignment_defenses.go` | StrictJSONDecode, allowlist structs |
| `defenses/ratelimit_defenses.go` | IPRateLimiter, AccountRateLimiter, constant-time compare |
| `defenses/smuggling_defenses.go` | Header stripping, CL+TE rejection, CRLF validation |
| `defenses/cors_defenses.go` | Strict CORS, SameSite cookies |
| `defenses/graphql_defenses.go` | Introspection block, depth/batch limits |
| `defenses/ssrf_defenses.go` | URL validation, private IP deny list, HTTPS-only |
| `defenses/pagination_defenses.go` | Page size cap, HMAC-signed tenant-scoped cursors |
| `defenses/error_defenses.go` | Recovery middleware, SafeErrorResponse, header stripping |
| `security_test.go` | 22 tests exercising all attacks and defenses |
