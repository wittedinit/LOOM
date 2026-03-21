# ADR-012: Binary IP Protection and Anti-Reverse-Engineering

## Status

Accepted

## Context

LOOM is a commercial infrastructure orchestrator containing proprietary:

- Orchestration decision logic and placement algorithms
- LLM prompt engineering and heuristic tuning
- Cost optimization models
- Protocol adapter implementations
- Multi-vendor config translation templates

Go binaries are notoriously easy to reverse engineer — function names, type information, package paths, and string literals are all preserved by default. Tools like Ghidra, GoReSym, and redress can reconstruct near-complete source structure. This is an existential IP risk for a commercial product.

## Decision

### Build Pipeline Protection (Mandatory)

1. All release builds use `garble -literals -seed=random build -ldflags="-s -w" -trimpath`
2. This strips debug info, removes filesystem paths, obfuscates function/type/package names, and encrypts string literals
3. CI pipeline enforces: no release build without garble

### Architectural Protection (Defense in Depth)

4. LLM prompts, heuristics, and cost models stored as AES-256-GCM encrypted assets loaded at runtime — NOT compiled into the binary
5. Vendor config templates stored as encrypted config files — NOT hardcoded strings
6. Core decision engine communicates via gRPC with well-defined interface — if needed, could be reimplemented in Rust for stronger RE resistance without changing the architecture
7. Plugin adapters are separate binaries (via go-plugin gRPC) — IP distributed across multiple binaries

### Runtime Protection

8. Binary self-hash verification at startup (detect tampering)
9. Debugger detection on Linux (ptrace, TracerPid) — log warning, don't crash (avoids false positives)
10. License validation tied to machine fingerprint for commercial editions

### Distribution Protection

11. Binaries signed with cosign/sigstore
12. Container images signed and digest-pinned
13. Checksums published for offline verification

## Consequences

- Higher barrier to casual reverse engineering — garble removes function names and encrypts string literals, eliminating the easy wins for attackers
- Encrypted runtime assets mean the most valuable IP (prompts, heuristics, cost models) is not in the binary at all
- Plugin architecture distributes IP across multiple binaries, forcing attackers to reverse engineer each one independently
- garble adds ~20-30% to build time
- garble + literals increases binary size ~10-15%
- Encrypted assets require key management (tie to vault MEK from ADR-011)
- Determined attackers with sufficient resources can still reverse engineer — this is true for any language
- Go will never be as RE-resistant as Rust/C++ — architectural protection matters more than binary obfuscation

## Alternatives Considered

1. **Rust for core engine** — Rejected for MVP: protocol library gaps outweigh RE benefit. Reconsidered if IP theft becomes a demonstrated threat.
2. **SaaS-only (no binary distribution)** — Rejected: air-gapped customers require on-prem binary.
3. **Custom VM/bytecode** — Rejected: extreme complexity, performance impact, maintenance burden.
4. **Legal-only protection (no technical measures)** — Rejected: legal protection is necessary but insufficient. Technical measures raise the cost of reverse engineering.
