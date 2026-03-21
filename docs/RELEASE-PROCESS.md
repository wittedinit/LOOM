# Release Process

> How LOOM versions are managed, released, upgraded, and rolled back.

---

## 1. Semantic Versioning

LOOM follows [Semantic Versioning 2.0.0](https://semver.org/):

```
MAJOR.MINOR.PATCH
  │      │     └── Bug fixes, security patches. No API changes. No migration changes.
  │      └──────── New features, backward-compatible API additions. May include migrations.
  └─────────────── Breaking API changes, incompatible migrations, architectural shifts.
```

### Version Examples

| Version | Meaning |
|---------|---------|
| `v0.1.0` | First development release. API is unstable. |
| `v0.3.0` | Third minor release. New adapters, new API endpoints. |
| `v0.3.1` | Patch on 0.3. Security fix or bug fix only. |
| `v1.0.0` | First stable release. API contract is locked for the `v1` lifetime. |
| `v2.0.0` | Breaking change. API path moves to `/api/v2/`. Migration may be irreversible. |

### Pre-1.0 Rules

While LOOM is pre-1.0 (`v0.x.y`):
- Minor versions may include breaking changes (documented in changelog).
- Patch versions never include breaking changes.
- Each minor release includes an upgrade guide.

### Post-1.0 Rules

After `v1.0.0`:
- No breaking API changes within a major version.
- Database migrations are always forward-compatible within a major version.
- Deprecated features are marked in one minor release and removed no earlier than the next major.

---

## 2. Release Cadence

| Release Type | Cadence | Content |
|-------------|---------|---------|
| Minor (`v0.x.0`) | Monthly | New features, new adapters, enhancements |
| Patch (`v0.x.y`) | As needed | Bug fixes, security patches, dependency updates |
| Major (`vX.0.0`) | When required | Breaking changes, architectural shifts |

### Release Timeline

```
Day 1-25:  Development on main (trunk-based)
Day 25:    Feature freeze — only bug fixes and docs merge after this point
Day 26-28: Release candidate testing (RC tags: v0.3.0-rc.1)
Day 29:    Release candidate approval
Day 30:    Tag v0.3.0, GoReleaser builds artifacts, GitHub release published
```

---

## 3. Changelog Generation

Changelogs are generated from [Conventional Commits](https://www.conventionalcommits.org/):

### Commit Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Commit Types

| Type | Changelog Section | Example |
|------|------------------|---------|
| `feat` | Features | `feat(adapter): add Nokia SR Linux gNMI adapter` |
| `fix` | Bug Fixes | `fix(workflow): prevent saga compensation loop on timeout` |
| `perf` | Performance | `perf(discovery): batch SNMP walks for 3x throughput` |
| `security` | Security | `security(vault): rotate envelope keys on credential update` |
| `refactor` | (not in changelog) | `refactor(domain): extract verification into separate package` |
| `docs` | (not in changelog) | `docs: update FAILURE-MODES.md with NATS partition behavior` |
| `test` | (not in changelog) | `test(adapter): add IPMI mock server for power cycle tests` |
| `chore` | (not in changelog) | `chore: update Go to 1.23.1` |

### Breaking Changes

Breaking changes include `BREAKING CHANGE:` in the commit footer or use `!` after the type:

```
feat(api)!: change device list response to use cursor pagination

BREAKING CHANGE: The `offset` parameter is removed from GET /api/v1/devices.
Use `cursor` parameter instead. See migration guide in CHANGELOG.md.
```

### Generated Changelog

```markdown
## v0.3.0 (2026-04-01)

### Features
- **adapter**: Add Nokia SR Linux gNMI adapter (#142)
- **llm**: Support Ollama model hot-swap at runtime (#156)
- **api**: Add bulk device import endpoint (#161)

### Bug Fixes
- **workflow**: Prevent saga compensation loop on timeout (#148)
- **auth**: Fix JWT tenant extraction for nested org structures (#153)

### Security
- **vault**: Rotate envelope keys on credential update (#159)

### Performance
- **discovery**: Batch SNMP walks for 3x throughput (#164)

### Breaking Changes
- **api**: Device list response uses cursor pagination (#161)
  Migration: Replace `offset` query param with `cursor`. See upgrade guide below.
```

---

## 4. Upgrade Path

### Database Migrations

LOOM uses [golang-migrate](https://github.com/golang-migrate/migrate) for schema migrations:

```
migrations/
├── 000001_initial_schema.up.sql
├── 000001_initial_schema.down.sql
├── 000002_add_observed_facts_confidence.up.sql
├── 000002_add_observed_facts_confidence.down.sql
├── 000003_add_topology_graph.up.sql
├── 000003_add_topology_graph.down.sql
└── ...
```

#### Migration Rules

1. **Every migration has both `up` and `down` files.** No exceptions.
2. **Migrations are forward-only in production.** Down migrations exist for development and rollback emergencies, not for routine use.
3. **Migrations must be backward-compatible within a minor version.** The old binary must be able to run against the new schema during rolling upgrades.
4. **Large data migrations run as background jobs**, not in the migration transaction. The migration adds the new column; a separate Temporal workflow backfills existing rows.
5. **Migrations are tested in CI.** Every PR that includes a migration runs: apply up, verify schema, apply down, verify rollback, apply up again.

#### Running Migrations

```bash
# Apply all pending migrations
./loom migrate up

# Migrate to a specific version
./loom migrate goto 000005

# Rollback the last migration
./loom migrate down 1

# Show current migration version
./loom migrate version
```

### Config Compatibility

Configuration changes follow these rules:

| Change Type | Handling |
|------------|---------|
| New config key | Default value provided. Old configs work without modification. |
| Renamed config key | Old key accepted with deprecation warning for one minor version. |
| Removed config key | Removal announced in previous minor release changelog. LOOM logs a warning and ignores the key. |
| Changed default value | Documented in changelog. Explicit values in existing configs are not affected. |

### Rolling Upgrades

LOOM supports rolling upgrades with zero downtime for distributed deployments:

```
1. Apply database migration (compatible with both old and new binary)
2. Upgrade edge agents first (they are independently versioned)
3. Upgrade LOOM hub instances one at a time
4. Verify health after each instance upgrade
5. If any instance fails health checks, stop and investigate
```

---

## 5. Multi-Instance Coordination

### Hub Version Skew

During rolling upgrades, multiple LOOM hub instances may run different versions simultaneously.

**Rules:**
- The **database schema** is the coordination point. Both old and new binaries must work against the current schema.
- **NATS events** use a versioned envelope (see [EVENT-MODEL.md](EVENT-MODEL.md)). Old consumers ignore unknown event versions.
- **Temporal workflows** started on the old version run to completion on old workers. New workflows start on new workers. Both can run concurrently.

**Maximum supported version skew:** One minor version. Running `v0.3.x` and `v0.4.x` simultaneously is supported. Running `v0.2.x` and `v0.4.x` simultaneously is not.

### Edge Agent Version Skew

Edge agents may run older versions than the hub. This is expected in large deployments where agents are upgraded gradually.

**Rules:**
- Edge agents communicate with the hub via NATS. The message format is versioned.
- The hub accepts messages from agents up to **two minor versions behind**.
- Edge agents that are more than two minor versions behind receive an `upgrade_required` event and refuse to process new operations until upgraded.
- Edge agents can be upgraded independently of the hub (the agent binary is self-contained).

---

## 6. Rollback Procedure

### When to Roll Back

Roll back if:
- Health checks fail after upgrade and do not recover within 5 minutes.
- Error rate exceeds 5% of requests post-upgrade.
- A critical bug is discovered that cannot be patched forward quickly.

### Rollback Steps

#### Binary Rollback

```bash
# Stop the current version
systemctl stop loom

# Replace the binary with the previous version
cp /opt/loom/bin/loom.previous /opt/loom/bin/loom

# Start the previous version
systemctl start loom

# Verify health
curl -s http://localhost:8080/healthz | jq .
```

#### Database Migration Rollback

```bash
# Check current migration version
./loom migrate version

# Roll back the last migration
./loom migrate down 1

# Verify schema matches the previous binary's expectations
./loom migrate version
```

**Caution:** Down migrations may lose data if the up migration added columns that have since been populated. Rollback is safe within minutes of upgrade. After hours of production traffic, data reconciliation may be needed.

#### State Reconciliation

After rollback, run state reconciliation to ensure consistency:

```bash
# Compare Temporal workflow state against DB projections
./loom admin db check

# Reconcile any gaps (replay unacknowledged NATS events)
./loom admin db reconcile

# Verify device state matches last known good state
./loom admin devices verify-all --tenant=<tenant_id>
```

---

## 7. LTS Policy

### Long-Term Support Versions

After `v1.0.0`, LOOM designates every **fourth minor release** as an LTS release:

| Version | LTS | Support Period |
|---------|-----|---------------|
| `v1.0.0` | Yes | 18 months from release |
| `v1.1.0` | No | 3 months (until v1.2.0) |
| `v1.2.0` | No | 3 months (until v1.3.0) |
| `v1.3.0` | No | 3 months (until v1.4.0) |
| `v1.4.0` | Yes | 18 months from release |
| `v1.5.0` | No | 3 months (until v1.6.0) |

### LTS Guarantees

- **Security patches** backported for the full 18-month LTS period.
- **Critical bug fixes** backported on a case-by-case basis.
- **No new features** backported to LTS branches.
- LTS releases get their own release branch: `release/v1.0.x`, `release/v1.4.x`.

### Pre-1.0 Policy

Before `v1.0.0`:
- No LTS designations.
- Each minor release is supported until the next minor release.
- Patch releases are provided for the current minor version only.
- Operators are expected to upgrade monthly.

---

## 8. Release Checklist

```
[ ] All CI checks pass on main
[ ] Changelog generated and reviewed
[ ] Migration tested: up → verify → down → verify → up
[ ] Upgrade guide written for any breaking changes
[ ] Security scan clean (govulncheck, trivy)
[ ] SBOM generated and included in release artifacts
[ ] Release candidate tagged (v0.x.0-rc.1)
[ ] RC tested in staging environment for 48 hours
[ ] RC approved by maintainers
[ ] Final tag pushed (v0.x.0)
[ ] GoReleaser builds and publishes artifacts
[ ] Container images signed with cosign
[ ] Air-gapped tarball assembled and checksummed
[ ] GitHub release published with changelog
[ ] Announcement posted to project communication channels
```
