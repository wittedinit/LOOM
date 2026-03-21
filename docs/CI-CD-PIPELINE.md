# CI/CD Pipeline

> Build, test, scan, and release pipeline for LOOM using GitHub Actions.

---

## 1. GitHub Actions Workflow Structure

```
.github/workflows/
├── ci.yml              # Runs on every PR and push to main
├── release.yml         # Runs on version tags (v*)
├── security-scan.yml   # Runs daily + on every PR
└── nightly.yml         # Nightly E2E and performance tests
```

### Trigger Rules

| Workflow | Trigger |
|----------|---------|
| `ci.yml` | Push to any branch, PR opened/updated against `main` |
| `release.yml` | Tag push matching `v*` (e.g., `v0.3.0`) |
| `security-scan.yml` | Daily at 06:00 UTC + every PR |
| `nightly.yml` | Daily at 02:00 UTC (main branch only) |

---

## 2. Build Stage

### Static Analysis and Compilation

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with:
        go-version-file: go.mod

    # Compilation
    - run: go build ./...

    # Vet — catches common mistakes
    - run: go vet ./...

    # Staticcheck — advanced static analysis
    - run: go install honnef.co/go/tools/cmd/staticcheck@latest
    - run: staticcheck ./...

    # golangci-lint — comprehensive linter suite
    - uses: golangci/golangci-lint-action@v6
      with:
        version: latest
        args: --timeout=5m
```

### golangci-lint Configuration

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck        # unchecked errors
    - govet           # vet checks
    - staticcheck     # advanced static analysis
    - unused          # unused code
    - gosimple        # simplification suggestions
    - ineffassign     # ineffective assignments
    - typecheck       # type checking
    - gocritic        # opinionated checks
    - gosec           # security-oriented checks
    - bodyclose       # unclosed HTTP response bodies
    - sqlclosecheck   # unclosed sql.Rows
    - contextcheck    # context.Context misuse
    - noctx           # HTTP requests without context
    - exhaustive      # non-exhaustive enum switches

linters-settings:
  gosec:
    excludes:
      - G104  # Allow unhandled errors in test files
    severity: high

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - gosec
        - errcheck
```

---

## 3. Test Stage

Tests run in dependency order. Each stage gates the next.

```yaml
test-unit:
  needs: build
  runs-on: ubuntu-latest
  steps:
    - run: go test -race -coverprofile=coverage.out -covermode=atomic ./...
    - uses: codecov/codecov-action@v4
      with:
        files: coverage.out

test-integration:
  needs: test-unit
  runs-on: ubuntu-latest
  services:
    postgres:
      image: timescale/timescaledb-ha:pg16
      env:
        POSTGRES_DB: loom_test
        POSTGRES_USER: loom_test
        POSTGRES_PASSWORD: test
      ports:
        - 5432:5432
      options: >-
        --health-cmd pg_isready
        --health-interval 10s
        --health-timeout 5s
        --health-retries 5
    nats:
      image: nats:2-alpine
      ports:
        - 4222:4222
      options: >-
        --health-cmd "wget --spider http://localhost:8222/healthz || exit 1"
  steps:
    - run: go test -race -tags=integration ./...

test-security:
  needs: test-unit
  runs-on: ubuntu-latest
  steps:
    - run: go test -race -tags=security ./...
```

### Test Parallelism

- Unit tests and security regression tests run in parallel (no shared dependencies).
- Integration tests run after unit tests pass (they require Docker services).
- E2E tests run nightly, not on every PR (they take ~10 minutes and require the full simulated fleet).

---

## 4. Security Stage

### Vulnerability Scanning

```yaml
security:
  needs: build
  runs-on: ubuntu-latest
  steps:
    # Go vulnerability database check
    - run: go install golang.org/x/vuln/cmd/govulncheck@latest
    - run: govulncheck ./...

    # Dependency audit — check for known vulnerable dependencies
    - run: go list -m -json all | go install github.com/sonatype-nexus-community/nancy@latest && nancy sleuth

    # SBOM generation — Software Bill of Materials
    - run: |
        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
        syft . -o spdx-json > sbom.spdx.json
    - uses: actions/upload-artifact@v4
      with:
        name: sbom
        path: sbom.spdx.json

    # Container image scanning (if building container)
    - run: |
        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
        trivy image --severity HIGH,CRITICAL --exit-code 1 loom:${{ github.sha }}
```

### Scanning Summary

| Tool | Purpose | Runs On |
|------|---------|---------|
| `govulncheck` | Go standard library and dependency CVEs | Every PR |
| `nancy` | Dependency audit against OSS Index | Every PR |
| `syft` | SBOM generation (SPDX format) | Every PR + release |
| `trivy` | Container image CVE scan | Every PR (if Dockerfile exists) + release |
| `gosec` | Source-level security anti-patterns | Every PR (via golangci-lint) |

---

## 5. Release Stage

### Semantic Versioning

Releases follow [semver](https://semver.org/):

- **MAJOR**: Breaking API changes, incompatible database migrations
- **MINOR**: New features, backward-compatible changes (monthly cadence)
- **PATCH**: Bug fixes, security patches (as needed)

See [RELEASE-PROCESS.md](RELEASE-PROCESS.md) for the full release procedure.

### GoReleaser

Multi-platform binary builds and packaging:

```yaml
# .goreleaser.yml
project_name: loom

builds:
  - id: loom
    main: ./cmd/loom
    binary: loom
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w
      - -X main.version={{.Version}}
      - -X main.commit={{.Commit}}
      - -X main.date={{.Date}}

archives:
  - id: default
    formats: ['tar.gz']
    name_template: "loom_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    files:
      - LICENSE
      - README.md
      - docs/RELEASE-PROCESS.md

dockers:
  - image_templates:
      - "ghcr.io/{{ .Env.GITHUB_REPOSITORY }}:{{ .Version }}"
      - "ghcr.io/{{ .Env.GITHUB_REPOSITORY }}:latest"
    dockerfile: Dockerfile
    build_flag_templates:
      - "--label=org.opencontainers.image.version={{.Version}}"
      - "--label=org.opencontainers.image.revision={{.Commit}}"

signs:
  - artifacts: checksum
    cmd: cosign
    args:
      - "sign-blob"
      - "--yes"
      - "${artifact}"
      - "--output-signature=${signature}"
```

### Release Workflow

```yaml
release:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Full history for changelog generation

    - uses: actions/setup-go@v5
      with:
        go-version-file: go.mod

    # Run full test suite before releasing
    - run: go test -race ./...
    - run: go test -race -tags=integration ./...
    - run: govulncheck ./...

    # Build and publish
    - uses: goreleaser/goreleaser-action@v6
      with:
        version: latest
        args: release --clean
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    # Sign container images
    - run: |
        cosign sign --yes ghcr.io/${{ github.repository }}:${{ github.ref_name }}
```

---

## 6. Branch Strategy

### Trunk-Based Development

- **`main`** is the trunk. All development happens here.
- **Feature branches** are short-lived (< 3 days). Created from `main`, merged back via PR.
- **No long-lived branches.** Feature flags are used instead of feature branches for incomplete work.
- **Release branches** are created only for patch releases on older minor versions (e.g., `release/v0.2.x`).

### Feature Flags

Incomplete features are gated by feature flags, not long-lived branches:

```go
if featureflag.Enabled("llm-cost-prediction", tenant.ID) {
    // New cost prediction feature — not yet GA
    plan.CostPrediction = llm.PredictCost(ctx, plan)
}
```

Feature flags are stored in PostgreSQL and evaluated per-tenant. This allows:
- Rolling out features to specific tenants before GA.
- Instant rollback by disabling the flag.
- A/B testing of different implementations.

---

## 7. Artifacts

### Signed Binaries

All release binaries are signed using [cosign](https://github.com/sigstore/cosign) (keyless, Sigstore-backed):

```bash
# Verify a LOOM binary
cosign verify-blob \
  --signature loom_0.3.0_linux_amd64.tar.gz.sig \
  --certificate-identity-regexp ".*github.com/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  loom_0.3.0_linux_amd64.tar.gz
```

### Container Images

- Published to GitHub Container Registry (`ghcr.io`).
- Tagged with version and `latest`.
- Signed with cosign.
- Digest-pinned in all deployment manifests (never `:latest` in production).

### Air-Gapped Release Tarball

For environments with no internet access, each release includes an air-gapped bundle:

```
loom-airgap-v0.3.0.tar.gz
├── loom                          # Go binary (linux/amd64)
├── loom-arm64                    # Go binary (linux/arm64)
├── container-images/
│   ├── loom-v0.3.0.tar           # docker save output
│   ├── postgres-16.tar
│   ├── nats-2.tar
│   ├── temporal-server.tar
│   └── valkey-7.tar
├── migrations/
│   └── *.sql                     # Database migration files
├── checksums.sha256
├── checksums.sha256.sig          # Signed checksums
└── INSTALL-AIRGAP.md             # Air-gapped installation instructions
```

Operators verify integrity with:

```bash
sha256sum -c checksums.sha256
cosign verify-blob --signature checksums.sha256.sig checksums.sha256
```
