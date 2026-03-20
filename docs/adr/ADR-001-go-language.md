# ADR-001: Go as Primary Language

## Status

Accepted

## Context

LOOM manages physical and virtual infrastructure across multiple protocols (SNMP, Redfish, WS-Man, VMware, gNMI). The system must handle 1000+ concurrent device connections, deploy as a single binary to edge locations, and integrate with Anthropic's LLM APIs. Language choice affects every component.

## Decision

Go is the primary language for all LOOM services.

Key factors:

- **Protocol libraries exist and are mature**: gosnmp, gofish (Redfish), go-wsman-messages, govmomi (VMware), openconfig/gnmi. These are production-grade and actively maintained. No other language has this breadth of infrastructure protocol coverage.
- **Goroutines for concurrent connections**: Managing 1000+ simultaneous device connections with goroutines and channels is natural. Each discovery scan, state poll, or config push runs in its own goroutine with bounded concurrency via semaphores.
- **Single binary deployment**: `go build` produces a statically linked binary. No runtime, no dependency management at edge sites. Critical for deploying to locations with limited operational support.
- **Official Anthropic SDK**: The `anthropic-sdk-go` package provides first-class LLM integration without wrapping REST APIs manually.
- **Temporal SDK**: Temporal's Go SDK is the original and most mature. Workflow and activity code is native Go.

### Rejected Alternatives

- **Rust**: Superior performance and safety, but missing critical protocol libraries. gosnmp, gofish, and govmomi have no Rust equivalents at comparable maturity. Building them would add 6-12 months.
- **Python**: Rich protocol libraries exist, but deployment complexity (virtualenvs, dependency conflicts at edge sites) and GIL limitations for concurrent connections make it unsuitable for the core system. Acceptable for one-off scripts and notebooks.
- **TypeScript/Node.js**: Wrong ecosystem for infrastructure management. Protocol library coverage is minimal. Single-threaded event loop requires careful management for 1000+ connections.

## Consequences

- All core services are written in Go
- Team members need Go proficiency
- Protocol-specific adapters benefit from existing Go libraries without FFI or subprocess overhead
- Edge deployment is simplified to copying a single binary
- LLM integration uses the official SDK with type-safe request/response handling
- Some developer experience trade-offs (verbose error handling, no generics until recently) are accepted as worthwhile given the ecosystem fit
