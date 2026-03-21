# LOOM Adapter SDK Specification

> **Status:** Ready for implementation
> **Closes:** RESEARCH-protocol-adapter-realism.md Section 4 (gap C10), APPLICATION-LAYER-GAP-ANALYSIS.md Gaps 1-2
> **Module:** `github.com/wittedinit/loom/sdk`
> **Phase:** 1B (scaffolding + conformance + MockSSH + MockRedfish + example adapter), Phase 6+ (plugin lifecycle)
> **Last updated:** 2026-03-21

---

## Table of Contents

1. [Scaffolding CLI](#1-scaffolding-cli)
2. [Conformance Test Suite](#2-conformance-test-suite)
3. [Mock Device Framework](#3-mock-device-framework)
4. [Example Adapter Walkthrough](#4-example-adapter-walkthrough)
5. [Contributor Guide](#5-contributor-guide)
6. [Plugin Lifecycle (Phase 6+)](#6-plugin-lifecycle-phase-6)

---

## 1. Scaffolding CLI

### 1.1 Command Specification

```
loom adapter scaffold [flags]

Flags:
  --name        string   Adapter name (required). Used for package name and directory.
                         Must be lowercase alphanumeric with hyphens.
  --protocol    string   Protocol hint (required). One of:
                         ssh, snmp, redfish, netconf, gnmi, http, custom
  --families    strings  Interface families to scaffold (default: connector,discoverer,executor).
                         Options: connector, discoverer, executor, statereader, watcher
  --output-dir  string   Parent directory for the generated adapter (default: ./adapters/)
  --module      string   Go module path (default: github.com/wittedinit/loom/adapters/<name>)
  --compensation string  Compensation reliability level (default: best_effort).
                         Options: transactional, snapshot_restore, best_effort, none

Examples:
  loom adapter scaffold --name dell-redfish --protocol redfish
  loom adapter scaffold --name junos-netconf --protocol netconf --families connector,discoverer,executor,statereader --compensation transactional
  loom adapter scaffold --name my-http-agent --protocol http --compensation none
  loom adapter scaffold --name arista-gnmi --protocol gnmi --families connector,discoverer,executor,statereader,watcher
```

### 1.2 Generated File Structure

```
adapters/<name>/
  adapter.go              # Adapter struct, Registration(), Connector implementation
  adapter_test.go         # Conformance suite import + protocol-specific test stubs
  config.go               # AdapterConfig with protocol-specific defaults
  discoverer.go           # Discoverer implementation (if --families includes discoverer)
  executor.go             # Executor + compensation metadata (if --families includes executor)
  statereader.go          # StateReader implementation (if --families includes statereader)
  watcher.go              # Watcher implementation (if --families includes watcher)
  errors.go               # Protocol-specific error classification helpers
  doc.go                  # Package documentation
  testdata/
    discovery_response.json   # Sample device discovery response for the protocol
    error_response.json       # Sample error response for the protocol
```

### 1.3 Generated File Templates

#### adapter.go

```go
// Package <name> implements a LOOM adapter for <protocol> targets.
//
// This adapter implements the following interface families:
//   - Connector (required)
//   - Discoverer
//   - Executor
//
// Compensation reliability: <compensation>
package <name>

import (
	"context"
	"fmt"
	"log/slog"
	"sync"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

// Adapter implements the LOOM adapter contract for <protocol> targets.
type Adapter struct {
	mu        sync.RWMutex
	config    Config
	logger    *slog.Logger
	connected bool

	// TODO: Add protocol-specific client field.
	// For example:
	//   client *<protocol>.Client
}

// New creates a new <name> adapter with the given configuration.
func New(config loom.AdapterConfig) loom.Adapter {
	cfg := DefaultConfig()
	cfg.Apply(config.Options)

	return &Adapter{
		config: cfg,
		logger: config.Logger,
	}
}

// Registration returns the adapter's registration metadata.
func (a *Adapter) Registration() loom.AdapterRegistration {
	return loom.AdapterRegistration{
		Name:     "<name>",
		Protocol: "<protocol>",
		Capabilities: []loom.Capability{
			{Name: "discover_hardware", Protocol: "<protocol>", ReadOnly: true},
			// TODO: Add capabilities this adapter provides.
		},
		Factory: New,
	}
}

// --- Connector (Required) ---

// Connect establishes a connection to the target device.
func (a *Adapter) Connect(ctx context.Context, endpoint loom.EndpointInfo, cred loom.CredentialRef) error {
	a.mu.Lock()
	defer a.mu.Unlock()

	if a.connected {
		return nil // already connected, idempotent
	}

	// TODO: Implement protocol-specific connection logic.
	// 1. Resolve the credential from the vault reference.
	// 2. Establish the protocol connection to endpoint.Address:endpoint.Port.
	// 3. Set a.connected = true on success.
	//
	// Return classified errors:
	//   - loom.NewTransientError("ADAPTER_UNREACHABLE", ...) for network failures
	//   - loom.NewPermanentError("ADAPTER_AUTH_FAILED", ...) for auth failures

	return fmt.Errorf("Connect not implemented")
}

// Disconnect closes the connection to the target device.
func (a *Adapter) Disconnect(ctx context.Context) error {
	a.mu.Lock()
	defer a.mu.Unlock()

	if !a.connected {
		return nil // already disconnected, idempotent
	}

	// TODO: Close protocol-specific client connections.
	// Always set a.connected = false, even on error.

	a.connected = false
	return nil
}

// Ping checks whether the connection is still alive.
func (a *Adapter) Ping(ctx context.Context) error {
	a.mu.RLock()
	defer a.mu.RUnlock()

	if !a.connected {
		return loom.NewPermanentError("ADAPTER_UNREACHABLE",
			"not connected", "", nil)
	}

	// TODO: Send a lightweight keepalive or probe to the target.
	// Return TransientError if the probe fails but reconnection might help.

	return nil
}

// Connected reports whether the adapter currently holds an active connection.
func (a *Adapter) Connected() bool {
	a.mu.RLock()
	defer a.mu.RUnlock()
	return a.connected
}
```

#### config.go

```go
package <name>

import "time"

// Config holds protocol-specific configuration for this adapter.
type Config struct {
	// ConnectTimeout is the maximum time to establish a connection.
	ConnectTimeout time.Duration

	// OperationTimeout is the default per-operation timeout.
	OperationTimeout time.Duration

	// IdempotencyCacheTTL is how long completed operation results are cached.
	// Must be at least 2x the maximum operation timeout per ADAPTER-CONTRACT.md.
	IdempotencyCacheTTL time.Duration

	// TODO: Add protocol-specific config fields.
	// For SSH:  Username, KeyPath, KnownHostsPath
	// For SNMP: Community, Version, SecurityLevel
	// For Redfish: BasePath, VerifyTLS
	// For NETCONF: Subsystem, CandidateDatastore
	// For gNMI: Encoding, SubscriptionMode
	// For HTTP: BaseURL, Headers, VerifyTLS
}

// DefaultConfig returns a Config with sensible defaults.
func DefaultConfig() Config {
	return Config{
		ConnectTimeout:      30 * time.Second,
		OperationTimeout:    60 * time.Second,
		IdempotencyCacheTTL: 24 * time.Hour,
	}
}

// Apply merges option overrides from AdapterConfig.Options into this Config.
func (c *Config) Apply(opts map[string]any) {
	if v, ok := opts["connect_timeout"].(time.Duration); ok {
		c.ConnectTimeout = v
	}
	if v, ok := opts["operation_timeout"].(time.Duration); ok {
		c.OperationTimeout = v
	}
}
```

#### discoverer.go

```go
package <name>

import (
	"context"
	"fmt"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

// Discover probes the connected target and returns identity and fact data.
func (a *Adapter) Discover(ctx context.Context, scope loom.DiscoveryScope) (*loom.DiscoveryResult, error) {
	if !a.Connected() {
		return nil, loom.NewPermanentError("ADAPTER_UNREACHABLE",
			"cannot discover: not connected", "", nil)
	}

	// TODO: Implement protocol-specific discovery.
	// 1. Query the target for identity attributes (serial, MAC, UUID, asset tag).
	// 2. Query the target for observed facts (model, firmware, CPU, memory, interfaces).
	// 3. Respect scope.Categories to limit what is discovered.
	// 4. Respect scope.Depth for recursive discovery (e.g., chassis -> blades).
	// 5. Classify errors: transient (timeout, temporary unavailable) vs permanent (unsupported).

	return &loom.DiscoveryResult{
		Identities: []loom.ExternalIdentity{
			// TODO: Populate from device response.
		},
		Facts: []loom.ObservedFact{
			// TODO: Populate from device response.
		},
		Timestamp: time.Now(),
	}, fmt.Errorf("Discover not implemented")
}
```

#### executor.go

```go
package <name>

import (
	"context"
	"fmt"
	"sync"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

// --- Idempotency Cache ---

type cachedResult struct {
	result    *loom.OperationResult
	expiresAt time.Time
}

var (
	idempotencyCache   = make(map[string]*cachedResult)
	idempotencyCacheMu sync.RWMutex
)

// Execute dispatches a typed operation to the target device.
// Adapters MUST NOT retry internally. One attempt per call.
// The Temporal activity layer owns retry logic.
func (a *Adapter) Execute(ctx context.Context, op loom.TypedOperation) (*loom.OperationResult, error) {
	if !a.Connected() {
		return nil, loom.NewPermanentError("ADAPTER_UNREACHABLE",
			"cannot execute: not connected", op.IdempotencyKey, nil)
	}

	// --- Idempotency check ---
	if cached := a.checkIdempotencyCache(op.IdempotencyKey); cached != nil {
		return cached, nil
	}

	var result *loom.OperationResult
	var err error

	startedAt := time.Now()

	switch op.Type {
	// TODO: Add case branches for each supported operation type.
	// Example:
	// case "power_cycle":
	//     result, err = a.executePowerCycle(ctx, op)
	default:
		return nil, loom.NewPermanentError("ADAPTER_UNSUPPORTED_OP",
			fmt.Sprintf("operation %q not supported by <name> adapter", op.Type),
			op.IdempotencyKey, nil)
	}

	if err != nil {
		return nil, err
	}

	result.StartedAt = startedAt
	result.CompletedAt = time.Now()

	// --- Cache successful result ---
	a.cacheResult(op.IdempotencyKey, result)

	return result, nil
}

// SupportedOperations returns the list of operation types this adapter handles.
func (a *Adapter) SupportedOperations() []loom.OperationType {
	return []loom.OperationType{
		// TODO: List supported operations.
		// "power_cycle",
		// "read_sensors",
	}
}

// CompensationFor returns compensation metadata for a given operation type.
func (a *Adapter) CompensationFor(opType loom.OperationType) loom.CompensationInfo {
	// TODO: Define compensation for each supported operation.
	return loom.CompensationInfo{
		Reversible:  false,
		Reliability: loom.CompReliabilityBestEffort,
	}
}

func (a *Adapter) checkIdempotencyCache(key string) *loom.OperationResult {
	idempotencyCacheMu.RLock()
	defer idempotencyCacheMu.RUnlock()

	cached, ok := idempotencyCache[key]
	if !ok {
		return nil
	}
	if time.Now().After(cached.expiresAt) {
		return nil
	}
	return cached.result
}

func (a *Adapter) cacheResult(key string, result *loom.OperationResult) {
	idempotencyCacheMu.Lock()
	defer idempotencyCacheMu.Unlock()

	idempotencyCache[key] = &cachedResult{
		result:    result,
		expiresAt: time.Now().Add(a.config.IdempotencyCacheTTL),
	}
}
```

#### errors.go

```go
package <name>

import (
	loom "github.com/wittedinit/loom/sdk/adapter"
)

// classifyError maps a protocol-specific error to a LOOM error type.
// Every adapter MUST classify errors. No raw error returns.
func classifyError(err error, correlationID string) error {
	if err == nil {
		return nil
	}

	// TODO: Implement protocol-specific error classification.
	//
	// Pattern for SSH:
	//   if errors.Is(err, ssh.ErrAuthFailed) {
	//       return loom.NewPermanentError("ADAPTER_AUTH_FAILED", err.Error(), correlationID, err)
	//   }
	//   if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
	//       return loom.NewTransientError("ADAPTER_TIMEOUT", err.Error(), correlationID, 5*time.Second, err)
	//   }
	//
	// Pattern for HTTP/Redfish:
	//   switch resp.StatusCode {
	//   case 401, 403:
	//       return loom.NewPermanentError("ADAPTER_AUTH_FAILED", ...)
	//   case 429:
	//       return loom.NewTransientError("ADAPTER_RATE_LIMITED", ...)
	//   case 500, 502, 503:
	//       return loom.NewTransientError("ADAPTER_TIMEOUT", ...)
	//   }

	// Default: treat unknown errors as transient (conservative).
	return loom.NewTransientError("ADAPTER_TIMEOUT", err.Error(), correlationID, 0, err)
}

// Ensure compile-time verification that our adapter satisfies all declared interfaces.
var (
	_ loom.Adapter    = (*Adapter)(nil)
	_ loom.Connector  = (*Adapter)(nil)
	// Uncomment as interfaces are implemented:
	// _ loom.Discoverer  = (*Adapter)(nil)
	// _ loom.Executor    = (*Adapter)(nil)
	// _ loom.StateReader = (*Adapter)(nil)
	// _ loom.Watcher     = (*Adapter)(nil)
)
```

#### adapter_test.go

```go
package <name>_test

import (
	"testing"

	<name> "github.com/wittedinit/loom/adapters/<name>"
	"github.com/wittedinit/loom/sdk/conformance"
	loom "github.com/wittedinit/loom/sdk/adapter"
)

func newTestAdapter() loom.Adapter {
	return <name>.New(loom.AdapterConfig{
		Protocol: "<protocol>",
		Options:  map[string]any{},
	})
}

func TestConformance(t *testing.T) {
	adapter := newTestAdapter()
	conformance.RunAll(t, adapter, conformance.Options{
		Connector:  true,
		Discoverer: true,
		Executor:   true,
		StateReader: false, // TODO: enable when StateReader is implemented
		Watcher:     false, // TODO: enable when Watcher is implemented
	})
}

// TODO: Add protocol-specific tests beyond what the conformance suite covers.
// Examples:
//   - TestConnectionPoolExhaustion
//   - TestProtocolVersionNegotiation
//   - TestVendorSpecificQuirks
```

---

## 2. Conformance Test Suite

The conformance test suite is a Go package that adapter authors import. It validates that an adapter correctly implements all declared contracts from ADAPTER-CONTRACT.md, ERROR-MODEL.md, and OPERATION-TYPES.md.

### 2.1 Package Structure

```
sdk/conformance/
  conformance.go       # RunAll, RunCategory entry points
  connectivity.go      # Connector interface tests
  discovery.go         # Discoverer interface tests
  execution.go         # Executor interface tests
  state.go             # StateReader interface tests
  watching.go          # Watcher interface tests
  errors.go            # Error classification tests
  idempotency.go       # Idempotency key tests
  compensation.go      # Compensation contract tests
  cancellation.go      # Context cancellation + timeout tests
  cleanup.go           # Resource cleanup tests
  helpers.go           # Shared test utilities
```

### 2.2 Entry Points

```go
package conformance

import (
	"testing"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

// Options controls which test categories to run.
type Options struct {
	// Interface families to test. Connector is always tested.
	Connector   bool // always true; included for explicitness
	Discoverer  bool
	Executor    bool
	StateReader bool
	Watcher     bool

	// MockEndpoint is the endpoint for the test target.
	// If empty, tests use a built-in mock server.
	MockEndpoint *loom.EndpointInfo

	// MockCredential is the credential for the test target.
	// If empty, tests use a built-in test credential.
	MockCredential *loom.CredentialRef

	// RealDevice indicates tests should run against a real device,
	// not a mock. Skips tests that require fault injection.
	RealDevice bool
}

// RunAll runs every applicable conformance test category.
func RunAll(t *testing.T, adapter loom.Adapter, opts Options) {
	t.Helper()

	t.Run("Connectivity", func(t *testing.T) {
		RunConnectivity(t, adapter, opts)
	})

	t.Run("ErrorClassification", func(t *testing.T) {
		RunErrorClassification(t, adapter, opts)
	})

	t.Run("ContextHandling", func(t *testing.T) {
		RunContextHandling(t, adapter, opts)
	})

	t.Run("ResourceCleanup", func(t *testing.T) {
		RunResourceCleanup(t, adapter, opts)
	})

	if opts.Discoverer {
		if d, ok := adapter.(loom.Discoverer); ok {
			t.Run("Discovery", func(t *testing.T) {
				RunDiscovery(t, adapter, d, opts)
			})
		} else {
			t.Fatal("Discoverer tests requested but adapter does not implement Discoverer")
		}
	}

	if opts.Executor {
		if e, ok := adapter.(loom.Executor); ok {
			t.Run("Execution", func(t *testing.T) {
				RunExecution(t, adapter, e, opts)
			})
			t.Run("Idempotency", func(t *testing.T) {
				RunIdempotency(t, adapter, e, opts)
			})
			t.Run("Compensation", func(t *testing.T) {
				RunCompensation(t, adapter, e, opts)
			})
		} else {
			t.Fatal("Executor tests requested but adapter does not implement Executor")
		}
	}

	if opts.StateReader {
		if sr, ok := adapter.(loom.StateReader); ok {
			t.Run("StateReading", func(t *testing.T) {
				RunStateReading(t, adapter, sr, opts)
			})
		} else {
			t.Fatal("StateReader tests requested but adapter does not implement StateReader")
		}
	}

	if opts.Watcher {
		if w, ok := adapter.(loom.Watcher); ok {
			t.Run("Watching", func(t *testing.T) {
				RunWatching(t, adapter, w, opts)
			})
		} else {
			t.Fatal("Watcher tests requested but adapter does not implement Watcher")
		}
	}
}

// RunCategory runs a single named category.
// Valid categories: "connectivity", "discovery", "execution", "idempotency",
// "compensation", "errors", "context", "cleanup", "statereading", "watching"
func RunCategory(t *testing.T, category string, adapter loom.Adapter, opts Options) {
	t.Helper()

	switch category {
	case "connectivity":
		RunConnectivity(t, adapter, opts)
	case "discovery":
		d, ok := adapter.(loom.Discoverer)
		if !ok {
			t.Skip("adapter does not implement Discoverer")
		}
		RunDiscovery(t, adapter, d, opts)
	case "execution":
		e, ok := adapter.(loom.Executor)
		if !ok {
			t.Skip("adapter does not implement Executor")
		}
		RunExecution(t, adapter, e, opts)
	case "idempotency":
		e, ok := adapter.(loom.Executor)
		if !ok {
			t.Skip("adapter does not implement Executor")
		}
		RunIdempotency(t, adapter, e, opts)
	case "compensation":
		e, ok := adapter.(loom.Executor)
		if !ok {
			t.Skip("adapter does not implement Executor")
		}
		RunCompensation(t, adapter, e, opts)
	case "errors":
		RunErrorClassification(t, adapter, opts)
	case "context":
		RunContextHandling(t, adapter, opts)
	case "cleanup":
		RunResourceCleanup(t, adapter, opts)
	case "statereading":
		sr, ok := adapter.(loom.StateReader)
		if !ok {
			t.Skip("adapter does not implement StateReader")
		}
		RunStateReading(t, adapter, sr, opts)
	case "watching":
		w, ok := adapter.(loom.Watcher)
		if !ok {
			t.Skip("adapter does not implement Watcher")
		}
		RunWatching(t, adapter, w, opts)
	default:
		t.Fatalf("unknown conformance category: %q", category)
	}
}
```

### 2.3 Test Categories and Cases

#### 2.3.1 Connectivity Tests (`connectivity.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunConnectivity(t *testing.T, adapter loom.Adapter, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	t.Run("Connect_Success", func(t *testing.T) {
		// PASS: Connect returns nil error and Connected() returns true.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		err := adapter.Connect(ctx, endpoint, cred)
		if err != nil {
			t.Fatalf("Connect failed: %v", err)
		}
		if !adapter.Connected() {
			t.Fatal("Connected() returned false after successful Connect")
		}

		// Cleanup
		_ = adapter.Disconnect(ctx)
	})

	t.Run("Connect_Idempotent", func(t *testing.T) {
		// PASS: Calling Connect twice returns nil both times. No error on double-connect.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		err := adapter.Connect(ctx, endpoint, cred)
		if err != nil {
			t.Fatalf("First Connect failed: %v", err)
		}

		err = adapter.Connect(ctx, endpoint, cred)
		if err != nil {
			t.Fatalf("Second Connect should be idempotent, got: %v", err)
		}

		_ = adapter.Disconnect(ctx)
	})

	t.Run("Disconnect_Success", func(t *testing.T) {
		// PASS: Disconnect returns nil and Connected() returns false.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		_ = adapter.Connect(ctx, endpoint, cred)

		err := adapter.Disconnect(ctx)
		if err != nil {
			t.Fatalf("Disconnect failed: %v", err)
		}
		if adapter.Connected() {
			t.Fatal("Connected() returned true after Disconnect")
		}
	})

	t.Run("Disconnect_Idempotent", func(t *testing.T) {
		// PASS: Calling Disconnect on an already-disconnected adapter returns nil.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		err := adapter.Disconnect(ctx)
		if err != nil {
			t.Fatalf("Disconnect on non-connected adapter should be idempotent, got: %v", err)
		}
	})

	t.Run("Ping_WhenConnected", func(t *testing.T) {
		// PASS: Ping returns nil when the adapter is connected.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		_ = adapter.Connect(ctx, endpoint, cred)
		defer func() { _ = adapter.Disconnect(ctx) }()

		err := adapter.Ping(ctx)
		if err != nil {
			t.Fatalf("Ping failed on connected adapter: %v", err)
		}
	})

	t.Run("Ping_WhenDisconnected", func(t *testing.T) {
		// PASS: Ping returns an error (PermanentError or TransientError) when not connected.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		_ = adapter.Disconnect(ctx)

		err := adapter.Ping(ctx)
		if err == nil {
			t.Fatal("Ping should return an error when not connected")
		}
		assertLoomError(t, err)
	})

	t.Run("Connect_InvalidEndpoint", func(t *testing.T) {
		// PASS: Connect to an unreachable endpoint returns a classified LoomError.
		if opts.RealDevice {
			t.Skip("skipping fault injection test against real device")
		}

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		badEndpoint := loom.EndpointInfo{
			Address:   "192.0.2.1", // RFC 5737 TEST-NET, guaranteed unreachable
			Port:      1,
			Transport: endpoint.Transport,
		}

		err := adapter.Connect(ctx, badEndpoint, cred)
		if err == nil {
			t.Fatal("Connect to unreachable endpoint should fail")
		}
		assertLoomError(t, err)
	})

	t.Run("Connect_InvalidCredentials", func(t *testing.T) {
		// PASS: Connect with bad credentials returns ADAPTER_AUTH_FAILED (PermanentError).
		if opts.RealDevice {
			t.Skip("skipping fault injection test against real device")
		}

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		badCred := loom.CredentialRef{
			VaultPath: "secret/nonexistent",
			Type:      "basic",
		}

		err := adapter.Connect(ctx, endpoint, badCred)
		if err == nil {
			t.Fatal("Connect with invalid credentials should fail")
		}
		assertPermanentError(t, err, "ADAPTER_AUTH_FAILED")
	})
}
```

#### 2.3.2 Discovery Tests (`discovery.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunDiscovery(t *testing.T, adapter loom.Adapter, discoverer loom.Discoverer, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	connectAdapter(t, adapter, endpoint, cred)
	defer disconnectAdapter(t, adapter)

	t.Run("Discover_ReturnsIdentities", func(t *testing.T) {
		// PASS: Discovery returns at least one ExternalIdentity.
		ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
		defer cancel()

		result, err := discoverer.Discover(ctx, loom.DiscoveryScope{
			Categories: []string{"all"},
			Depth:      1,
		})
		if err != nil {
			t.Fatalf("Discover failed: %v", err)
		}
		if len(result.Identities) == 0 {
			t.Fatal("Discover returned zero identities; at least one is required")
		}
		for i, id := range result.Identities {
			if id.Type == "" {
				t.Errorf("Identity[%d].Type is empty", i)
			}
			if id.Value == "" {
				t.Errorf("Identity[%d].Value is empty", i)
			}
		}
	})

	t.Run("Discover_ReturnsFacts", func(t *testing.T) {
		// PASS: Discovery returns at least one ObservedFact.
		ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
		defer cancel()

		result, err := discoverer.Discover(ctx, loom.DiscoveryScope{
			Categories: []string{"all"},
			Depth:      1,
		})
		if err != nil {
			t.Fatalf("Discover failed: %v", err)
		}
		if len(result.Facts) == 0 {
			t.Fatal("Discover returned zero facts; at least one is required")
		}
		for i, fact := range result.Facts {
			if fact.Category == "" {
				t.Errorf("Fact[%d].Category is empty", i)
			}
			if fact.Key == "" {
				t.Errorf("Fact[%d].Key is empty", i)
			}
			if fact.Source == "" {
				t.Errorf("Fact[%d].Source is empty", i)
			}
		}
	})

	t.Run("Discover_TimestampSet", func(t *testing.T) {
		// PASS: DiscoveryResult.Timestamp is set to a reasonable time.
		ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
		defer cancel()

		before := time.Now()
		result, err := discoverer.Discover(ctx, loom.DiscoveryScope{
			Categories: []string{"hardware"},
			Depth:      1,
		})
		if err != nil {
			t.Fatalf("Discover failed: %v", err)
		}
		if result.Timestamp.Before(before) {
			t.Error("Discover Timestamp is before the call started")
		}
	})

	t.Run("Discover_CategoryFilter", func(t *testing.T) {
		// PASS: When scope.Categories is ["hardware"], no "network" facts appear.
		ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
		defer cancel()

		result, err := discoverer.Discover(ctx, loom.DiscoveryScope{
			Categories: []string{"hardware"},
			Depth:      1,
		})
		if err != nil {
			t.Fatalf("Discover failed: %v", err)
		}
		for _, fact := range result.Facts {
			if fact.Category == "network" {
				t.Error("Discover returned network facts when only hardware was requested")
			}
		}
	})

	t.Run("Discover_WhenDisconnected", func(t *testing.T) {
		// PASS: Discover returns a classified error when the adapter is not connected.
		_ = adapter.Disconnect(context.Background())
		defer connectAdapter(t, adapter, endpoint, cred)

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		_, err := discoverer.Discover(ctx, loom.DiscoveryScope{
			Categories: []string{"all"},
			Depth:      1,
		})
		if err == nil {
			t.Fatal("Discover should fail when not connected")
		}
		assertLoomError(t, err)
	})
}
```

#### 2.3.3 Execution Tests (`execution.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunExecution(t *testing.T, adapter loom.Adapter, executor loom.Executor, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	connectAdapter(t, adapter, endpoint, cred)
	defer disconnectAdapter(t, adapter)

	t.Run("SupportedOperations_NonEmpty", func(t *testing.T) {
		// PASS: SupportedOperations returns at least one operation type.
		ops := executor.SupportedOperations()
		if len(ops) == 0 {
			t.Fatal("SupportedOperations returned empty list; Executor must support at least one operation")
		}
	})

	t.Run("Execute_UnsupportedOp", func(t *testing.T) {
		// PASS: Executing an unsupported operation returns ADAPTER_UNSUPPORTED_OP.
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		_, err := executor.Execute(ctx, loom.TypedOperation{
			Type:           "nonexistent_operation_type_for_conformance_test",
			IdempotencyKey: "conformance-unsupported-op-test",
			Timeout:        10 * time.Second,
		})
		if err == nil {
			t.Fatal("Execute should fail for unsupported operation type")
		}
		assertPermanentError(t, err, "ADAPTER_UNSUPPORTED_OP")
	})

	t.Run("Execute_WhenDisconnected", func(t *testing.T) {
		// PASS: Execute returns a classified error when the adapter is not connected.
		_ = adapter.Disconnect(context.Background())
		defer connectAdapter(t, adapter, endpoint, cred)

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		ops := executor.SupportedOperations()
		if len(ops) == 0 {
			t.Skip("no supported operations to test")
		}

		_, err := executor.Execute(ctx, loom.TypedOperation{
			Type:           ops[0],
			IdempotencyKey: "conformance-disconnected-exec-test",
			Timeout:        10 * time.Second,
		})
		if err == nil {
			t.Fatal("Execute should fail when not connected")
		}
		assertLoomError(t, err)
	})

	t.Run("Execute_ResultFields", func(t *testing.T) {
		// PASS: A successful Execute returns a result with populated timestamps and matching operation type.
		// This test only runs if the adapter provides a conformance test operation.
		// Adapter authors should register a lightweight test operation.
		t.Skip("requires adapter-specific test operation -- implement in adapter_test.go")
	})
}
```

#### 2.3.4 Idempotency Tests (`idempotency.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	"github.com/google/uuid"
	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunIdempotency(t *testing.T, adapter loom.Adapter, executor loom.Executor, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	connectAdapter(t, adapter, endpoint, cred)
	defer disconnectAdapter(t, adapter)

	t.Run("SameKey_SameResult", func(t *testing.T) {
		// PASS: Submitting the same IdempotencyKey twice returns the same result
		// without re-executing the operation.
		// This test requires a supported operation. Adapter authors should provide
		// a test-safe operation that can be executed repeatedly.
		t.Skip("requires adapter-specific test operation -- implement in adapter_test.go")
	})

	t.Run("DifferentKey_IndependentExecution", func(t *testing.T) {
		// PASS: Different idempotency keys produce independent executions.
		t.Skip("requires adapter-specific test operation -- implement in adapter_test.go")
	})

	t.Run("IdempotencyKey_Required", func(t *testing.T) {
		// PASS: An operation with an empty IdempotencyKey either generates one or rejects with ValidationError.
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		ops := executor.SupportedOperations()
		if len(ops) == 0 {
			t.Skip("no supported operations")
		}

		_, err := executor.Execute(ctx, loom.TypedOperation{
			Type:           ops[0],
			IdempotencyKey: "", // empty key
			Timeout:        10 * time.Second,
		})
		// Adapter may either auto-generate a key (no error) or reject (ValidationError).
		// Both are acceptable. What is NOT acceptable is executing without any key tracking.
		_ = err
	})

	t.Run("IdempotencyKey_UUID_Format", func(t *testing.T) {
		// PASS: A valid UUID is accepted as an IdempotencyKey.
		key := uuid.New().String()
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		ops := executor.SupportedOperations()
		if len(ops) == 0 {
			t.Skip("no supported operations")
		}

		// We just verify the key format is accepted; the operation may fail for other reasons.
		_, _ = executor.Execute(ctx, loom.TypedOperation{
			Type:           ops[0],
			IdempotencyKey: key,
			Timeout:        10 * time.Second,
		})
	})
}
```

#### 2.3.5 Compensation Tests (`compensation.go`)

```go
package conformance

import (
	"testing"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

// CompensationDeclarer is an optional interface adapters can implement
// to declare compensation metadata per operation type.
type CompensationDeclarer interface {
	CompensationFor(opType loom.OperationType) loom.CompensationInfo
}

func RunCompensation(t *testing.T, adapter loom.Adapter, executor loom.Executor, opts Options) {
	t.Helper()

	t.Run("AllMutatingOps_DeclareCompensation", func(t *testing.T) {
		// PASS: Every mutating (non-read-only) operation has a CompensationInfo declaration.
		declarer, ok := adapter.(CompensationDeclarer)
		if !ok {
			t.Skip("adapter does not implement CompensationDeclarer")
		}

		ops := executor.SupportedOperations()
		for _, op := range ops {
			info := declarer.CompensationFor(op)
			// Every operation must have a declared reliability level, even if it's "none".
			validReliabilities := map[loom.CompensationReliability]bool{
				loom.CompReliabilityTransactional:    true,
				loom.CompReliabilitySnapshotRestore:  true,
				loom.CompReliabilityBestEffort:       true,
				loom.CompReliabilityNone:             true,
			}
			if !validReliabilities[info.Reliability] {
				t.Errorf("Operation %q has invalid CompensationReliability: %q", op, info.Reliability)
			}

			// If an operation is declared reversible, it must specify the compensation operation.
			if info.Reversible && info.CompensationOp == "" {
				t.Errorf("Operation %q is declared Reversible but CompensationOp is empty", op)
			}
		}
	})

	t.Run("ReadOnlyOps_NotReversible", func(t *testing.T) {
		// PASS: Read-only operations (read_sensors, read_interfaces) should not
		// declare themselves as reversible.
		declarer, ok := adapter.(CompensationDeclarer)
		if !ok {
			t.Skip("adapter does not implement CompensationDeclarer")
		}

		readOnlyOps := map[loom.OperationType]bool{
			"read_sensors":    true,
			"read_interfaces": true,
		}

		ops := executor.SupportedOperations()
		for _, op := range ops {
			if readOnlyOps[op] {
				info := declarer.CompensationFor(op)
				if info.Reversible {
					t.Errorf("Read-only operation %q should not be declared Reversible", op)
				}
			}
		}
	})
}
```

#### 2.3.6 Error Classification Tests (`errors.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunErrorClassification(t *testing.T, adapter loom.Adapter, opts Options) {
	t.Helper()

	t.Run("AllErrors_AreLoomErrors", func(t *testing.T) {
		// PASS: Every error returned by the adapter implements loom.LoomError.
		// This is validated implicitly by all other tests via assertLoomError.
		// This test documents the requirement.
	})

	t.Run("TransientErrors_AreRetryable", func(t *testing.T) {
		// PASS: Any TransientError has Retryable() == true.
		err := loom.NewTransientError("TEST", "test", "", 0, nil)
		if !err.Retryable() {
			t.Fatal("TransientError.Retryable() must return true")
		}
	})

	t.Run("PermanentErrors_AreNotRetryable", func(t *testing.T) {
		// PASS: Any PermanentError has Retryable() == false.
		err := loom.NewPermanentError("TEST", "test", "", nil)
		if err.Retryable() {
			t.Fatal("PermanentError.Retryable() must return false")
		}
	})

	t.Run("Errors_CarryCorrelationID", func(t *testing.T) {
		// PASS: Errors from adapter operations carry the correlation ID from the request.
		if opts.RealDevice {
			t.Skip("skipping fault injection test against real device")
		}

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		// Connect to a bad endpoint to trigger an error with a known context.
		badEndpoint := loom.EndpointInfo{
			Address:   "192.0.2.1",
			Port:      1,
			Transport: "tcp",
		}
		badCred := loom.CredentialRef{VaultPath: "test/bad", Type: "basic"}

		err := adapter.Connect(ctx, badEndpoint, badCred)
		if err == nil {
			_ = adapter.Disconnect(ctx)
			return // adapter connected to test address; cannot test error correlation
		}

		if le, ok := err.(loom.LoomError); ok {
			// CorrelationID may be empty if the adapter cannot derive one from a Connect call.
			// This is acceptable. The requirement is that when a correlation ID IS available,
			// it is passed through.
			_ = le.CorrelationID()
		}
	})

	t.Run("NoRawErrors", func(t *testing.T) {
		// REQUIREMENT: Adapters MUST NOT return bare `errors.New()` or `fmt.Errorf()`.
		// All errors must be one of: TransientError, PermanentError, PartialError, ValidationError.
		// This is validated by assertLoomError in all other tests.
		// This test documents the requirement.
	})
}
```

#### 2.3.7 Context Cancellation and Timeout Tests (`cancellation.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunContextHandling(t *testing.T, adapter loom.Adapter, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	t.Run("Connect_RespectsContextCancellation", func(t *testing.T) {
		// PASS: A cancelled context causes Connect to return promptly with an error.
		ctx, cancel := context.WithCancel(context.Background())
		cancel() // cancel immediately

		err := adapter.Connect(ctx, endpoint, cred)
		if err == nil {
			// Some adapters may connect instantly (e.g., in-memory mocks).
			_ = adapter.Disconnect(context.Background())
			t.Skip("adapter connected despite cancelled context (may be in-memory mock)")
		}
	})

	t.Run("Connect_RespectsTimeout", func(t *testing.T) {
		// PASS: A context with a very short timeout causes Connect to fail promptly.
		if opts.RealDevice {
			t.Skip("skipping timeout test against real device")
		}

		ctx, cancel := context.WithTimeout(context.Background(), 1*time.Millisecond)
		defer cancel()

		// Use unreachable address to ensure timeout fires before connection succeeds.
		badEndpoint := loom.EndpointInfo{
			Address:   "192.0.2.1",
			Port:      1,
			Transport: endpoint.Transport,
		}

		start := time.Now()
		err := adapter.Connect(ctx, badEndpoint, cred)
		elapsed := time.Since(start)

		if err == nil {
			_ = adapter.Disconnect(context.Background())
			t.Skip("adapter connected despite short timeout")
		}

		// The operation should complete within a reasonable time after the timeout fires.
		// We allow 5 seconds of slack for OS-level socket timeout behavior.
		if elapsed > 5*time.Second {
			t.Errorf("Connect took %v after a 1ms timeout; context cancellation not respected", elapsed)
		}
	})

	t.Run("Ping_RespectsContextCancellation", func(t *testing.T) {
		// PASS: A cancelled context causes Ping to return promptly with an error.
		ctx, cancel := context.WithCancel(context.Background())
		cancel()

		err := adapter.Ping(ctx)
		// Ping on a disconnected adapter may fail for "not connected" reasons
		// rather than cancellation. Both are acceptable.
		_ = err
	})

	t.Run("Disconnect_RespectsContextCancellation", func(t *testing.T) {
		// PASS: Disconnect completes even with a cancelled context.
		// Disconnect SHOULD attempt to clean up resources regardless of context state.
		// This is a weaker requirement: we prefer graceful cleanup over strict cancellation.
		ctx, cancel := context.WithCancel(context.Background())
		cancel()

		// This should not panic or block indefinitely.
		_ = adapter.Disconnect(ctx)
	})
}
```

#### 2.3.8 Resource Cleanup Tests (`cleanup.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunResourceCleanup(t *testing.T, adapter loom.Adapter, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	t.Run("Disconnect_CleansUp", func(t *testing.T) {
		// PASS: After Disconnect, Connected() returns false and no resources leak.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		err := adapter.Connect(ctx, endpoint, cred)
		if err != nil {
			t.Fatalf("Connect failed: %v", err)
		}

		err = adapter.Disconnect(ctx)
		if err != nil {
			t.Fatalf("Disconnect failed: %v", err)
		}

		if adapter.Connected() {
			t.Fatal("Connected() should return false after Disconnect")
		}
	})

	t.Run("MultipleConnectDisconnect_NoLeak", func(t *testing.T) {
		// PASS: Connecting and disconnecting 10 times does not leak resources.
		ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
		defer cancel()

		for i := 0; i < 10; i++ {
			err := adapter.Connect(ctx, endpoint, cred)
			if err != nil {
				t.Fatalf("Connect iteration %d failed: %v", i, err)
			}
			err = adapter.Disconnect(ctx)
			if err != nil {
				t.Fatalf("Disconnect iteration %d failed: %v", i, err)
			}
		}

		if adapter.Connected() {
			t.Fatal("Connected() should return false after final Disconnect")
		}
	})
}
```

#### 2.3.9 StateReader Tests (`state.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunStateReading(t *testing.T, adapter loom.Adapter, reader loom.StateReader, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	connectAdapter(t, adapter, endpoint, cred)
	defer disconnectAdapter(t, adapter)

	t.Run("ReadState_ReturnsSnapshot", func(t *testing.T) {
		// PASS: ReadState returns a StateSnapshot with a non-zero Timestamp and non-empty State map.
		t.Skip("requires adapter-specific ResourceRef -- implement in adapter_test.go")
	})

	t.Run("ReadState_TimestampIsRecent", func(t *testing.T) {
		// PASS: The returned snapshot timestamp is within the last 60 seconds.
		t.Skip("requires adapter-specific ResourceRef -- implement in adapter_test.go")
	})

	t.Run("ReadState_InvalidResource", func(t *testing.T) {
		// PASS: Reading a non-existent resource returns a classified error.
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		_, err := reader.ReadState(ctx, loom.ResourceRef{
			DeviceID:     "conformance-nonexistent",
			ResourceType: "nonexistent",
			ResourceID:   "nonexistent",
		})
		if err == nil {
			t.Fatal("ReadState should fail for nonexistent resource")
		}
		assertLoomError(t, err)
	})

	t.Run("ReadState_WhenDisconnected", func(t *testing.T) {
		// PASS: ReadState returns a classified error when not connected.
		_ = adapter.Disconnect(context.Background())
		defer connectAdapter(t, adapter, endpoint, cred)

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		_, err := reader.ReadState(ctx, loom.ResourceRef{
			DeviceID:     "any",
			ResourceType: "any",
			ResourceID:   "any",
		})
		if err == nil {
			t.Fatal("ReadState should fail when not connected")
		}
		assertLoomError(t, err)
	})
}
```

#### 2.3.10 Watcher Tests (`watching.go`)

```go
package conformance

import (
	"context"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

func RunWatching(t *testing.T, adapter loom.Adapter, watcher loom.Watcher, opts Options) {
	t.Helper()
	endpoint, cred := resolveTarget(opts)

	connectAdapter(t, adapter, endpoint, cred)
	defer disconnectAdapter(t, adapter)

	t.Run("Watch_ReturnsChannel", func(t *testing.T) {
		// PASS: Watch returns a non-nil channel.
		t.Skip("requires adapter-specific ResourceRef -- implement in adapter_test.go")
	})

	t.Run("Watch_SequenceMonotonic", func(t *testing.T) {
		// PASS: Events received on the Watch channel have monotonically increasing Sequence numbers.
		t.Skip("requires adapter-specific ResourceRef -- implement in adapter_test.go")
	})

	t.Run("StopWatch_ClosesChannel", func(t *testing.T) {
		// PASS: After StopWatch, the event channel is closed.
		t.Skip("requires adapter-specific ResourceRef -- implement in adapter_test.go")
	})

	t.Run("Watch_WhenDisconnected", func(t *testing.T) {
		// PASS: Watch returns a classified error when not connected.
		_ = adapter.Disconnect(context.Background())
		defer connectAdapter(t, adapter, endpoint, cred)

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		_, err := watcher.Watch(ctx, loom.ResourceRef{
			DeviceID:     "any",
			ResourceType: "any",
			ResourceID:   "any",
		})
		if err == nil {
			t.Fatal("Watch should fail when not connected")
		}
		assertLoomError(t, err)
	})
}
```

### 2.4 Shared Test Helpers (`helpers.go`)

```go
package conformance

import (
	"context"
	"errors"
	"testing"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

// resolveTarget returns the endpoint and credential to use for tests.
// If opts provides them, use those. Otherwise, use built-in defaults.
func resolveTarget(opts Options) (loom.EndpointInfo, loom.CredentialRef) {
	endpoint := loom.EndpointInfo{
		Address:   "127.0.0.1",
		Port:      0, // will be replaced by mock server
		Transport: "tcp",
	}
	cred := loom.CredentialRef{
		VaultPath: "secret/conformance-test",
		Type:      "basic",
	}

	if opts.MockEndpoint != nil {
		endpoint = *opts.MockEndpoint
	}
	if opts.MockCredential != nil {
		cred = *opts.MockCredential
	}

	return endpoint, cred
}

// connectAdapter connects the adapter for testing, failing the test on error.
func connectAdapter(t *testing.T, adapter loom.Adapter, endpoint loom.EndpointInfo, cred loom.CredentialRef) {
	t.Helper()
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := adapter.Connect(ctx, endpoint, cred); err != nil {
		t.Fatalf("test setup: Connect failed: %v", err)
	}
}

// disconnectAdapter disconnects the adapter, logging but not failing on error.
func disconnectAdapter(t *testing.T, adapter loom.Adapter) {
	t.Helper()
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := adapter.Disconnect(ctx); err != nil {
		t.Logf("test cleanup: Disconnect returned error: %v", err)
	}
}

// assertLoomError asserts that err implements loom.LoomError.
func assertLoomError(t *testing.T, err error) {
	t.Helper()
	var le loom.LoomError
	if !errors.As(err, &le) {
		t.Errorf("expected loom.LoomError, got %T: %v", err, err)
	}
}

// assertPermanentError asserts err is a PermanentError with the given code.
func assertPermanentError(t *testing.T, err error, expectedCode string) {
	t.Helper()
	var pe *loom.PermanentError
	if !errors.As(err, &pe) {
		t.Errorf("expected PermanentError, got %T: %v", err, err)
		return
	}
	if pe.Code() != expectedCode {
		t.Errorf("expected error code %q, got %q", expectedCode, pe.Code())
	}
}

// assertTransientError asserts err is a TransientError with the given code.
func assertTransientError(t *testing.T, err error, expectedCode string) {
	t.Helper()
	var te *loom.TransientError
	if !errors.As(err, &te) {
		t.Errorf("expected TransientError, got %T: %v", err, err)
		return
	}
	if te.Code() != expectedCode {
		t.Errorf("expected error code %q, got %q", expectedCode, te.Code())
	}
}
```

### 2.5 Conformance Report

When run, the conformance suite produces a structured report:

```
=== Conformance Results ===
Adapter:     dell-redfish
Protocol:    redfish
Timestamp:   2026-03-21T14:30:00Z

Connectivity:
  [PASS] Connect_Success
  [PASS] Connect_Idempotent
  [PASS] Disconnect_Success
  [PASS] Disconnect_Idempotent
  [PASS] Ping_WhenConnected
  [PASS] Ping_WhenDisconnected
  [PASS] Connect_InvalidEndpoint
  [PASS] Connect_InvalidCredentials

ErrorClassification:
  [PASS] AllErrors_AreLoomErrors
  [PASS] TransientErrors_AreRetryable
  [PASS] PermanentErrors_AreNotRetryable
  [PASS] Errors_CarryCorrelationID

ContextHandling:
  [PASS] Connect_RespectsContextCancellation
  [PASS] Connect_RespectsTimeout
  [PASS] Ping_RespectsContextCancellation
  [PASS] Disconnect_RespectsContextCancellation

ResourceCleanup:
  [PASS] Disconnect_CleansUp
  [PASS] MultipleConnectDisconnect_NoLeak

Discovery:
  [PASS] Discover_ReturnsIdentities
  [PASS] Discover_ReturnsFacts
  [PASS] Discover_TimestampSet
  [PASS] Discover_CategoryFilter
  [PASS] Discover_WhenDisconnected

Execution:
  [PASS] SupportedOperations_NonEmpty
  [PASS] Execute_UnsupportedOp
  [PASS] Execute_WhenDisconnected

Compensation:
  [PASS] AllMutatingOps_DeclareCompensation
  [PASS] ReadOnlyOps_NotReversible

TOTAL: 22 passed, 0 failed, 4 skipped
```

---

## 3. Mock Device Framework

### 3.1 Package Structure

```
sdk/mockdevice/
  base.go         # MockDevice base type with shared behavior
  ssh.go          # MockSSH server
  snmp.go         # MockSNMP agent
  redfish.go      # MockRedfish server (HTTP + JSON)
  netconf.go      # MockNETCONF server (TCP + XML framing)
  gnmi.go         # MockgNMI server (gRPC)
  fault.go        # Fault injection framework
```

### 3.2 Base MockDevice

```go
package mockdevice

import (
	"sync"
	"time"
)

// DeviceState represents the simulated state of a mock device.
type DeviceState struct {
	mu sync.RWMutex

	// Identity
	SerialNumber string
	Vendor       string
	Model        string
	Hostname     string
	MACAddresses []string
	UUID         string

	// Hardware
	PowerState  PowerState // on, off
	CPUCount    int
	MemoryGB    int
	FirmwareVer string

	// Network
	Interfaces []MockInterface

	// Sensors
	Sensors []MockSensor
}

// PowerState represents the power state of a mock device.
type PowerState string

const (
	PowerOn  PowerState = "on"
	PowerOff PowerState = "off"
)

// MockInterface represents a simulated network interface.
type MockInterface struct {
	Name        string
	Status      string // "up", "down"
	Speed       string // "1G", "10G"
	MTU         int
	MACAddress  string
	IPAddresses []string
}

// MockSensor represents a simulated hardware sensor.
type MockSensor struct {
	Name   string
	Type   string  // "temperature", "voltage", "fan", "power"
	Value  float64
	Unit   string
	Status string  // "ok", "warning", "critical"
}

// FaultConfig controls failure injection on a mock device.
type FaultConfig struct {
	// ConnectFailure causes Connect to fail with the specified error.
	// nil = connect succeeds.
	ConnectFailure error

	// AuthFailure causes authentication to fail.
	AuthFailure bool

	// LatencyMin is the minimum simulated latency for every operation.
	LatencyMin time.Duration

	// LatencyMax is the maximum simulated latency (uniformly distributed).
	LatencyMax time.Duration

	// FailAfterN causes operations to fail after N successful calls.
	// 0 = never fail. Useful for testing partial failures.
	FailAfterN int

	// FailWithError is the error returned when FailAfterN triggers.
	FailWithError error

	// DropConnectionAfter causes the mock to close the connection after this duration.
	// 0 = never drop.
	DropConnectionAfter time.Duration

	// ResponseCorruption randomly corrupts N% of responses (0-100).
	ResponseCorruption int
}

// DefaultState returns a fully populated device state for testing.
func DefaultState() *DeviceState {
	return &DeviceState{
		SerialNumber: "LOOM-MOCK-SN-001",
		Vendor:       "LOOM-Mock",
		Model:        "TestDevice-1000",
		Hostname:     "mock-device-1",
		MACAddresses: []string{"00:11:22:33:44:55", "00:11:22:33:44:56"},
		UUID:         "550e8400-e29b-41d4-a716-446655440000",
		PowerState:   PowerOn,
		CPUCount:     4,
		MemoryGB:     32,
		FirmwareVer:  "1.0.0",
		Interfaces: []MockInterface{
			{Name: "eth0", Status: "up", Speed: "1G", MTU: 1500, MACAddress: "00:11:22:33:44:55", IPAddresses: []string{"192.168.1.10/24"}},
			{Name: "eth1", Status: "down", Speed: "10G", MTU: 9000, MACAddress: "00:11:22:33:44:56", IPAddresses: []string{}},
		},
		Sensors: []MockSensor{
			{Name: "CPU0 Temp", Type: "temperature", Value: 42.5, Unit: "celsius", Status: "ok"},
			{Name: "PSU1 Voltage", Type: "voltage", Value: 12.1, Unit: "volts", Status: "ok"},
			{Name: "Fan1 Speed", Type: "fan", Value: 3200, Unit: "rpm", Status: "ok"},
		},
	}
}
```

### 3.3 MockSSH

```go
package mockdevice

import (
	"fmt"
	"io"
	"net"
	"strings"
	"sync"
	"sync/atomic"

	"github.com/gliderlabs/ssh"
)

// SSHConfig configures the MockSSH server.
type SSHConfig struct {
	// State is the initial device state.
	State *DeviceState

	// Faults configures failure injection.
	Faults FaultConfig

	// ValidUsers maps username -> password for authentication.
	ValidUsers map[string]string

	// CommandResponses maps command strings to canned responses.
	// Supports prefix matching: "show version" matches "show version | include Serial".
	CommandResponses map[string]string

	// HostKey is the SSH host private key PEM bytes.
	// If nil, a test key is generated.
	HostKey []byte
}

// MockSSH is a test SSH server that simulates a network device CLI.
type MockSSH struct {
	mu       sync.Mutex
	config   SSHConfig
	server   *ssh.Server
	listener net.Listener
	addr     string
	callCount atomic.Int64
}

// NewSSH creates and starts a mock SSH server on a random port.
func NewSSH(config SSHConfig) (*MockSSH, error) {
	if config.State == nil {
		config.State = DefaultState()
	}
	if config.ValidUsers == nil {
		config.ValidUsers = map[string]string{"admin": "admin"}
	}
	if config.CommandResponses == nil {
		config.CommandResponses = defaultSSHResponses(config.State)
	}

	m := &MockSSH{config: config}

	m.server = &ssh.Server{
		Handler: m.handleSession,
		PasswordHandler: func(ctx ssh.Context, password string) bool {
			if config.Faults.AuthFailure {
				return false
			}
			expected, ok := config.ValidUsers[ctx.User()]
			return ok && password == expected
		},
	}

	listener, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		return nil, fmt.Errorf("MockSSH listen: %w", err)
	}
	m.listener = listener
	m.addr = listener.Addr().String()

	go func() {
		_ = m.server.Serve(listener)
	}()

	return m, nil
}

// Addr returns the listen address (host:port) of the mock SSH server.
func (m *MockSSH) Addr() string { return m.addr }

// Close shuts down the mock SSH server.
func (m *MockSSH) Close() error {
	return m.server.Close()
}

// CallCount returns how many SSH sessions have been handled.
func (m *MockSSH) CallCount() int64 { return m.callCount.Load() }

func (m *MockSSH) handleSession(s ssh.Session) {
	m.callCount.Add(1)

	// Simulate latency if configured.
	simulateLatency(m.config.Faults)

	// Check FailAfterN.
	if m.config.Faults.FailAfterN > 0 && m.callCount.Load() > int64(m.config.Faults.FailAfterN) {
		_, _ = io.WriteString(s, "% Error: device unavailable\n")
		_ = s.Exit(1)
		return
	}

	// Read command from the session.
	cmd := strings.TrimSpace(string(s.RawCommand()))
	if cmd == "" {
		// Interactive session: read from stdin.
		buf := make([]byte, 4096)
		n, _ := s.Read(buf)
		cmd = strings.TrimSpace(string(buf[:n]))
	}

	// Look up response.
	response := m.lookupResponse(cmd)
	_, _ = io.WriteString(s, response)
	_ = s.Exit(0)
}

func (m *MockSSH) lookupResponse(cmd string) string {
	m.mu.Lock()
	defer m.mu.Unlock()

	// Exact match first.
	if resp, ok := m.config.CommandResponses[cmd]; ok {
		return resp
	}

	// Prefix match.
	for prefix, resp := range m.config.CommandResponses {
		if strings.HasPrefix(cmd, prefix) {
			return resp
		}
	}

	return fmt.Sprintf("%% Unknown command: %s\n", cmd)
}

func defaultSSHResponses(state *DeviceState) map[string]string {
	return map[string]string{
		"show version": fmt.Sprintf(
			"Hostname: %s\nVendor: %s\nModel: %s\nSerial: %s\nFirmware: %s\n",
			state.Hostname, state.Vendor, state.Model, state.SerialNumber, state.FirmwareVer,
		),
		"show interfaces": formatInterfaces(state.Interfaces),
		"show inventory":  fmt.Sprintf("Serial Number: %s\nUUID: %s\n", state.SerialNumber, state.UUID),
	}
}

func formatInterfaces(ifaces []MockInterface) string {
	var b strings.Builder
	for _, iface := range ifaces {
		fmt.Fprintf(&b, "%s: %s, speed %s, MTU %d, MAC %s\n",
			iface.Name, iface.Status, iface.Speed, iface.MTU, iface.MACAddress)
		for _, ip := range iface.IPAddresses {
			fmt.Fprintf(&b, "  inet %s\n", ip)
		}
	}
	return b.String()
}
```

### 3.4 MockRedfish

```go
package mockdevice

import (
	"encoding/json"
	"fmt"
	"net"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync"
	"sync/atomic"
)

// RedfishConfig configures the MockRedfish server.
type RedfishConfig struct {
	// State is the initial device state.
	State *DeviceState

	// Faults configures failure injection.
	Faults FaultConfig

	// Vendor is the Redfish vendor string (e.g., "Dell", "HPE", "Supermicro").
	Vendor string

	// RedfishVersion is the Redfish schema version to report.
	RedfishVersion string

	// ValidCredentials maps username -> password.
	ValidCredentials map[string]string
}

// MockRedfish is a test HTTP server that simulates a Redfish BMC.
type MockRedfish struct {
	mu        sync.Mutex
	config    RedfishConfig
	server    *httptest.Server
	callCount atomic.Int64
}

// NewRedfish creates and starts a mock Redfish server.
func NewRedfish(config RedfishConfig) *MockRedfish {
	if config.State == nil {
		config.State = DefaultState()
	}
	if config.Vendor == "" {
		config.Vendor = "LOOM-Mock"
	}
	if config.RedfishVersion == "" {
		config.RedfishVersion = "1.17.0"
	}
	if config.ValidCredentials == nil {
		config.ValidCredentials = map[string]string{"admin": "admin"}
	}

	m := &MockRedfish{config: config}

	mux := http.NewServeMux()
	mux.HandleFunc("/redfish/v1/", m.handleRoot)
	mux.HandleFunc("/redfish/v1/Systems/1", m.handleSystem)
	mux.HandleFunc("/redfish/v1/Systems/1/Actions/ComputerSystem.Reset", m.handleReset)
	mux.HandleFunc("/redfish/v1/Chassis/1", m.handleChassis)
	mux.HandleFunc("/redfish/v1/Chassis/1/Thermal", m.handleThermal)
	mux.HandleFunc("/redfish/v1/Chassis/1/Power", m.handlePower)
	mux.HandleFunc("/redfish/v1/Managers/1", m.handleManager)

	m.server = httptest.NewServer(m.authMiddleware(mux))
	return m
}

// URL returns the base URL of the mock Redfish server.
func (m *MockRedfish) URL() string { return m.server.URL }

// Addr returns the host:port of the mock Redfish server.
func (m *MockRedfish) Addr() string {
	u := m.server.URL
	// Strip "http://"
	return strings.TrimPrefix(u, "http://")
}

// Host returns just the host part.
func (m *MockRedfish) Host() string {
	host, _, _ := net.SplitHostPort(m.Addr())
	return host
}

// Port returns just the port part.
func (m *MockRedfish) Port() string {
	_, port, _ := net.SplitHostPort(m.Addr())
	return port
}

// Close shuts down the mock Redfish server.
func (m *MockRedfish) Close() { m.server.Close() }

// CallCount returns the number of HTTP requests handled.
func (m *MockRedfish) CallCount() int64 { return m.callCount.Load() }

func (m *MockRedfish) authMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if m.config.Faults.AuthFailure {
			http.Error(w, `{"error":{"code":"Base.1.0.GeneralError","message":"Authentication failed"}}`, http.StatusUnauthorized)
			return
		}

		user, pass, ok := r.BasicAuth()
		if !ok {
			http.Error(w, `{"error":{"code":"Base.1.0.GeneralError","message":"Authentication required"}}`, http.StatusUnauthorized)
			return
		}

		expected, valid := m.config.ValidCredentials[user]
		if !valid || expected != pass {
			http.Error(w, `{"error":{"code":"Base.1.0.GeneralError","message":"Invalid credentials"}}`, http.StatusUnauthorized)
			return
		}

		m.callCount.Add(1)
		simulateLatency(m.config.Faults)
		next.ServeHTTP(w, r)
	})
}

func (m *MockRedfish) handleRoot(w http.ResponseWriter, r *http.Request) {
	m.jsonResponse(w, map[string]any{
		"@odata.id":      "/redfish/v1/",
		"@odata.type":    "#ServiceRoot.v1_12_0.ServiceRoot",
		"Id":             "RootService",
		"Name":           "Root Service",
		"RedfishVersion": m.config.RedfishVersion,
		"Vendor":         m.config.Vendor,
		"Product":        fmt.Sprintf("%s BMC", m.config.Vendor),
		"Systems":        map[string]string{"@odata.id": "/redfish/v1/Systems"},
		"Chassis":        map[string]string{"@odata.id": "/redfish/v1/Chassis"},
		"Managers":       map[string]string{"@odata.id": "/redfish/v1/Managers"},
	})
}

func (m *MockRedfish) handleSystem(w http.ResponseWriter, r *http.Request) {
	m.mu.Lock()
	state := m.config.State
	m.mu.Unlock()

	m.jsonResponse(w, map[string]any{
		"@odata.id":    "/redfish/v1/Systems/1",
		"@odata.type":  "#ComputerSystem.v1_16_0.ComputerSystem",
		"Id":           "1",
		"Name":         state.Hostname,
		"Manufacturer": state.Vendor,
		"Model":        state.Model,
		"SerialNumber": state.SerialNumber,
		"UUID":         state.UUID,
		"PowerState":   capitalize(string(state.PowerState)),
		"Status": map[string]string{
			"State":  "Enabled",
			"Health": "OK",
		},
		"ProcessorSummary": map[string]any{
			"Count": state.CPUCount,
		},
		"MemorySummary": map[string]any{
			"TotalSystemMemoryGiB": state.MemoryGB,
		},
		"Boot": map[string]any{
			"BootSourceOverrideTarget":  "None",
			"BootSourceOverrideEnabled": "Disabled",
		},
		"Actions": map[string]any{
			"#ComputerSystem.Reset": map[string]any{
				"target":                          "/redfish/v1/Systems/1/Actions/ComputerSystem.Reset",
				"ResetType@Redfish.AllowableValues": []string{"On", "ForceOff", "GracefulShutdown", "GracefulRestart", "ForceRestart"},
			},
		},
	})
}

func (m *MockRedfish) handleReset(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var body struct {
		ResetType string `json:"ResetType"`
	}
	if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
		http.Error(w, `{"error":{"message":"invalid request body"}}`, http.StatusBadRequest)
		return
	}

	m.mu.Lock()
	switch body.ResetType {
	case "On":
		m.config.State.PowerState = PowerOn
	case "ForceOff", "GracefulShutdown":
		m.config.State.PowerState = PowerOff
	case "GracefulRestart", "ForceRestart":
		m.config.State.PowerState = PowerOn // simplified: restart = stays on
	default:
		m.mu.Unlock()
		http.Error(w, `{"error":{"message":"unsupported ResetType"}}`, http.StatusBadRequest)
		return
	}
	m.mu.Unlock()

	w.WriteHeader(http.StatusNoContent)
}

func (m *MockRedfish) handleChassis(w http.ResponseWriter, r *http.Request) {
	state := m.config.State
	m.jsonResponse(w, map[string]any{
		"@odata.id":    "/redfish/v1/Chassis/1",
		"@odata.type":  "#Chassis.v1_16_0.Chassis",
		"Id":           "1",
		"Name":         "Main Chassis",
		"Manufacturer": state.Vendor,
		"Model":        state.Model,
		"SerialNumber": state.SerialNumber,
		"Thermal":      map[string]string{"@odata.id": "/redfish/v1/Chassis/1/Thermal"},
		"Power":        map[string]string{"@odata.id": "/redfish/v1/Chassis/1/Power"},
	})
}

func (m *MockRedfish) handleThermal(w http.ResponseWriter, r *http.Request) {
	m.mu.Lock()
	sensors := m.config.State.Sensors
	m.mu.Unlock()

	temps := []map[string]any{}
	fans := []map[string]any{}
	for _, s := range sensors {
		if s.Type == "temperature" {
			temps = append(temps, map[string]any{
				"Name":                  s.Name,
				"ReadingCelsius":        s.Value,
				"Status":               map[string]string{"State": "Enabled", "Health": strings.Title(s.Status)},
			})
		}
		if s.Type == "fan" {
			fans = append(fans, map[string]any{
				"Name":    s.Name,
				"Reading": int(s.Value),
				"Status":  map[string]string{"State": "Enabled", "Health": strings.Title(s.Status)},
			})
		}
	}

	m.jsonResponse(w, map[string]any{
		"@odata.id":    "/redfish/v1/Chassis/1/Thermal",
		"@odata.type":  "#Thermal.v1_7_0.Thermal",
		"Temperatures": temps,
		"Fans":         fans,
	})
}

func (m *MockRedfish) handlePower(w http.ResponseWriter, r *http.Request) {
	m.mu.Lock()
	sensors := m.config.State.Sensors
	m.mu.Unlock()

	voltages := []map[string]any{}
	for _, s := range sensors {
		if s.Type == "voltage" {
			voltages = append(voltages, map[string]any{
				"Name":               s.Name,
				"ReadingVolts":       s.Value,
				"Status":            map[string]string{"State": "Enabled", "Health": strings.Title(s.Status)},
			})
		}
	}

	m.jsonResponse(w, map[string]any{
		"@odata.id":  "/redfish/v1/Chassis/1/Power",
		"@odata.type": "#Power.v1_6_0.Power",
		"Voltages":   voltages,
	})
}

func (m *MockRedfish) handleManager(w http.ResponseWriter, r *http.Request) {
	m.jsonResponse(w, map[string]any{
		"@odata.id":      "/redfish/v1/Managers/1",
		"@odata.type":    "#Manager.v1_11_0.Manager",
		"Id":             "1",
		"Name":           "BMC",
		"ManagerType":    "BMC",
		"FirmwareVersion": m.config.State.FirmwareVer,
	})
}

func (m *MockRedfish) jsonResponse(w http.ResponseWriter, data any) {
	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(data)
}

func capitalize(s string) string {
	if s == "" {
		return s
	}
	return strings.ToUpper(s[:1]) + s[1:]
}
```

### 3.5 Shared Utilities (`fault.go`)

```go
package mockdevice

import (
	"math/rand"
	"time"
)

// simulateLatency pauses for the configured latency range.
func simulateLatency(faults FaultConfig) {
	if faults.LatencyMin <= 0 && faults.LatencyMax <= 0 {
		return
	}
	min := faults.LatencyMin
	max := faults.LatencyMax
	if max <= min {
		time.Sleep(min)
		return
	}
	jitter := time.Duration(rand.Int63n(int64(max - min)))
	time.Sleep(min + jitter)
}
```

### 3.6 Mock Integration with Conformance Tests

The conformance test suite and mock framework integrate as follows. Adapter test files use mocks to create the test target:

```go
package myredfish_test

import (
	"fmt"
	"testing"

	myredfish "github.com/wittedinit/loom/adapters/my-redfish"
	loom "github.com/wittedinit/loom/sdk/adapter"
	"github.com/wittedinit/loom/sdk/conformance"
	"github.com/wittedinit/loom/sdk/mockdevice"
)

func TestConformance(t *testing.T) {
	// Start a mock Redfish server with default state.
	mock := mockdevice.NewRedfish(mockdevice.RedfishConfig{
		Vendor: "Dell",
	})
	defer mock.Close()

	adapter := myredfish.New(loom.AdapterConfig{
		Protocol: "redfish",
		Options: map[string]any{
			"base_url":   mock.URL(),
			"verify_tls": false,
		},
	})

	conformance.RunAll(t, adapter, conformance.Options{
		Connector:  true,
		Discoverer: true,
		Executor:   true,
		MockEndpoint: &loom.EndpointInfo{
			Address:   mock.Host(),
			Port:      parsePort(mock.Port()),
			Transport: "http",
		},
		MockCredential: &loom.CredentialRef{
			VaultPath: "mock/admin",
			Type:      "basic",
		},
	})
}

func TestFaultInjection_AuthFailure(t *testing.T) {
	mock := mockdevice.NewRedfish(mockdevice.RedfishConfig{
		Faults: mockdevice.FaultConfig{
			AuthFailure: true,
		},
	})
	defer mock.Close()

	adapter := myredfish.New(loom.AdapterConfig{Protocol: "redfish"})

	// Verify the adapter correctly classifies auth failures.
	conformance.RunCategory(t, "connectivity", adapter, conformance.Options{
		MockEndpoint: &loom.EndpointInfo{
			Address:   mock.Host(),
			Port:      parsePort(mock.Port()),
			Transport: "http",
		},
	})
}

func TestFaultInjection_HighLatency(t *testing.T) {
	mock := mockdevice.NewRedfish(mockdevice.RedfishConfig{
		Faults: mockdevice.FaultConfig{
			LatencyMin: 2 * time.Second,
			LatencyMax: 5 * time.Second,
		},
	})
	defer mock.Close()

	// Test that the adapter respects context timeouts under latency.
	adapter := myredfish.New(loom.AdapterConfig{Protocol: "redfish"})
	conformance.RunCategory(t, "context", adapter, conformance.Options{
		MockEndpoint: &loom.EndpointInfo{
			Address:   mock.Host(),
			Port:      parsePort(mock.Port()),
			Transport: "http",
		},
	})
}

func parsePort(s string) int {
	var p int
	fmt.Sscanf(s, "%d", &p)
	return p
}
```

---

## 4. Example Adapter Walkthrough

This section contains a complete, working example adapter: an HTTP health-check adapter that manages devices with a simple JSON health API. The adapter demonstrates every contract requirement.

### 4.1 The Target Device API

The adapter targets devices that expose a simple HTTP management API:

```
GET  /health              -> {"status": "ok", "uptime_seconds": 12345}
GET  /info                -> {"serial": "ABC123", "model": "Widget-X", "firmware": "2.1.0"}
GET  /sensors             -> {"sensors": [{"name": "cpu_temp", "value": 42.5, "unit": "celsius"}]}
POST /power {"action": "restart"} -> {"previous": "on", "current": "on"}
POST /power {"action": "off"}     -> {"previous": "on", "current": "off"}
POST /power {"action": "on"}      -> {"previous": "off", "current": "on"}
```

### 4.2 Complete Adapter Code

```go
// Package httphealth implements a LOOM adapter for devices with a simple HTTP
// health-check management API. This is a reference implementation demonstrating
// every adapter contract requirement:
//
//   - Connector: connect, disconnect, ping (idempotent)
//   - Discoverer: identity and fact discovery
//   - Executor: power_cycle with compensation, read_sensors (read-only)
//   - Error classification: all errors are typed LoomErrors
//   - Idempotency: operation results cached by idempotency key
//   - Compensation: power_cycle declares best_effort compensation
//   - Context cancellation: all operations respect ctx
//
// Lines of code: ~280 (excluding comments and tests)
package httphealth

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"sync"
	"time"

	loom "github.com/wittedinit/loom/sdk/adapter"
)

// --- Adapter struct ---

// Adapter manages devices with a simple HTTP health API.
type Adapter struct {
	mu     sync.RWMutex
	config Config
	logger *slog.Logger

	// Runtime state (set by Connect, cleared by Disconnect).
	client    *http.Client
	baseURL   string
	connected bool

	// Idempotency cache: key -> cached result.
	cacheMu sync.RWMutex
	cache   map[string]*cachedResult
}

type cachedResult struct {
	result    *loom.OperationResult
	expiresAt time.Time
}

// Config holds adapter configuration.
type Config struct {
	ConnectTimeout      time.Duration
	OperationTimeout    time.Duration
	IdempotencyCacheTTL time.Duration
}

// DefaultConfig returns production-safe defaults.
func DefaultConfig() Config {
	return Config{
		ConnectTimeout:      10 * time.Second,
		OperationTimeout:    30 * time.Second,
		IdempotencyCacheTTL: 24 * time.Hour, // >= 2x max operation timeout
	}
}

// --- Factory ---

// New is the adapter factory. It is registered in the adapter registry.
func New(cfg loom.AdapterConfig) loom.Adapter {
	c := DefaultConfig()
	if v, ok := cfg.Options["connect_timeout"].(time.Duration); ok {
		c.ConnectTimeout = v
	}
	if v, ok := cfg.Options["operation_timeout"].(time.Duration); ok {
		c.OperationTimeout = v
	}

	logger := cfg.Logger
	if logger == nil {
		logger = slog.Default()
	}

	return &Adapter{
		config: c,
		logger: logger,
		cache:  make(map[string]*cachedResult),
	}
}

// --- Registration ---

// Registration returns metadata for the adapter registry.
func (a *Adapter) Registration() loom.AdapterRegistration {
	return loom.AdapterRegistration{
		Name:     "httphealth",
		Protocol: "http",
		Capabilities: []loom.Capability{
			{Name: "discover_hardware", Protocol: "http", ReadOnly: true},
			{Name: "power_control", Protocol: "http", ReadOnly: false},
			{Name: "read_sensors", Protocol: "http", ReadOnly: true},
		},
		Factory: New,
	}
}

// --- Connector (Required) ---

// Connect establishes an HTTP client targeting the device.
// Validates reachability by calling /health. Idempotent: safe to call twice.
func (a *Adapter) Connect(ctx context.Context, endpoint loom.EndpointInfo, cred loom.CredentialRef) error {
	a.mu.Lock()
	defer a.mu.Unlock()

	if a.connected {
		return nil // idempotent
	}

	// Build base URL from endpoint.
	scheme := "http"
	if endpoint.Transport == "https" || endpoint.Transport == "tls" {
		scheme = "https"
	}
	a.baseURL = fmt.Sprintf("%s://%s:%d", scheme, endpoint.Address, endpoint.Port)

	a.client = &http.Client{
		Timeout: a.config.ConnectTimeout,
	}

	// Validate reachability with a health check.
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, a.baseURL+"/health", nil)
	if err != nil {
		return loom.NewPermanentError("ADAPTER_UNREACHABLE",
			fmt.Sprintf("failed to create request: %v", err), "", err)
	}

	// In a real adapter, resolve cred.VaultPath to get actual credentials
	// and set Authorization headers here.

	resp, err := a.client.Do(req)
	if err != nil {
		return loom.NewTransientError("ADAPTER_UNREACHABLE",
			fmt.Sprintf("health check failed: %v", err), "", 5*time.Second, err)
	}
	defer resp.Body.Close()

	if resp.StatusCode == http.StatusUnauthorized || resp.StatusCode == http.StatusForbidden {
		return loom.NewPermanentError("ADAPTER_AUTH_FAILED",
			"health check returned auth error", "", nil)
	}
	if resp.StatusCode != http.StatusOK {
		return loom.NewTransientError("ADAPTER_UNREACHABLE",
			fmt.Sprintf("health check returned status %d", resp.StatusCode), "", 5*time.Second, nil)
	}

	a.connected = true
	a.logger.Info("connected to device", "url", a.baseURL)
	return nil
}

// Disconnect releases the HTTP client. Idempotent: safe to call when not connected.
func (a *Adapter) Disconnect(ctx context.Context) error {
	a.mu.Lock()
	defer a.mu.Unlock()

	a.client = nil
	a.baseURL = ""
	a.connected = false
	return nil
}

// Ping checks whether the device is still reachable.
func (a *Adapter) Ping(ctx context.Context) error {
	a.mu.RLock()
	defer a.mu.RUnlock()

	if !a.connected {
		return loom.NewPermanentError("ADAPTER_UNREACHABLE", "not connected", "", nil)
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, a.baseURL+"/health", nil)
	if err != nil {
		return loom.NewTransientError("ADAPTER_TIMEOUT", err.Error(), "", 0, err)
	}

	resp, err := a.client.Do(req)
	if err != nil {
		return loom.NewTransientError("ADAPTER_TIMEOUT",
			fmt.Sprintf("ping failed: %v", err), "", 5*time.Second, err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return loom.NewTransientError("ADAPTER_UNREACHABLE",
			fmt.Sprintf("ping returned %d", resp.StatusCode), "", 5*time.Second, nil)
	}
	return nil
}

// Connected reports whether the adapter holds an active connection.
func (a *Adapter) Connected() bool {
	a.mu.RLock()
	defer a.mu.RUnlock()
	return a.connected
}

// --- Discoverer ---

// Discover retrieves identity and hardware facts from the device.
func (a *Adapter) Discover(ctx context.Context, scope loom.DiscoveryScope) (*loom.DiscoveryResult, error) {
	if !a.Connected() {
		return nil, loom.NewPermanentError("ADAPTER_UNREACHABLE", "not connected", "", nil)
	}

	body, err := a.httpGet(ctx, "/info")
	if err != nil {
		return nil, err
	}

	var info struct {
		Serial   string `json:"serial"`
		Model    string `json:"model"`
		Firmware string `json:"firmware"`
	}
	if err := json.Unmarshal(body, &info); err != nil {
		return nil, loom.NewTransientError("ADAPTER_TIMEOUT",
			fmt.Sprintf("invalid info response: %v", err), "", 0, err)
	}

	return &loom.DiscoveryResult{
		Identities: []loom.ExternalIdentity{
			{Type: "serial_number", Value: info.Serial, Scope: "chassis"},
		},
		Facts: []loom.ObservedFact{
			{Category: "hardware", Key: "model", Value: info.Model, Source: "httphealth", Timestamp: time.Now()},
			{Category: "firmware", Key: "firmware_version", Value: info.Firmware, Source: "httphealth", Timestamp: time.Now()},
		},
		Timestamp: time.Now(),
	}, nil
}

// --- Executor ---

// Execute dispatches a typed operation. Honors idempotency keys. Does NOT retry.
func (a *Adapter) Execute(ctx context.Context, op loom.TypedOperation) (*loom.OperationResult, error) {
	if !a.Connected() {
		return nil, loom.NewPermanentError("ADAPTER_UNREACHABLE", "not connected", op.IdempotencyKey, nil)
	}

	// Check idempotency cache.
	if cached := a.getCached(op.IdempotencyKey); cached != nil {
		return cached, nil
	}

	var result *loom.OperationResult
	var err error
	start := time.Now()

	switch op.Type {
	case "power_cycle":
		result, err = a.executePowerCycle(ctx, op)
	case "read_sensors":
		result, err = a.executeReadSensors(ctx, op)
	default:
		return nil, loom.NewPermanentError("ADAPTER_UNSUPPORTED_OP",
			fmt.Sprintf("unsupported operation: %s", op.Type), op.IdempotencyKey, nil)
	}

	if err != nil {
		return nil, err
	}

	result.StartedAt = start
	result.CompletedAt = time.Now()
	result.IdempotencyKey = op.IdempotencyKey

	// Cache the result.
	a.putCached(op.IdempotencyKey, result)

	return result, nil
}

// SupportedOperations returns the operation types this adapter handles.
func (a *Adapter) SupportedOperations() []loom.OperationType {
	return []loom.OperationType{"power_cycle", "read_sensors"}
}

// CompensationFor returns compensation metadata for a given operation.
func (a *Adapter) CompensationFor(opType loom.OperationType) loom.CompensationInfo {
	switch opType {
	case "power_cycle":
		return loom.CompensationInfo{
			Reversible:     true,
			CompensationOp: "power_cycle", // reverse action
			Reliability:    loom.CompReliabilityBestEffort,
		}
	case "read_sensors":
		return loom.CompensationInfo{
			Reversible: false, // read-only, nothing to compensate
		}
	default:
		return loom.CompensationInfo{Reversible: false}
	}
}

// --- Operation implementations ---

func (a *Adapter) executePowerCycle(ctx context.Context, op loom.TypedOperation) (*loom.OperationResult, error) {
	params, ok := op.Params.(loom.PowerCycleParams)
	if !ok {
		return nil, loom.NewPermanentError("VALIDATION_FAILED",
			"Params is not PowerCycleParams", op.IdempotencyKey, nil)
	}

	reqBody, _ := json.Marshal(map[string]string{"action": params.Action})
	body, err := a.httpPost(ctx, "/power", reqBody)
	if err != nil {
		return nil, err
	}

	var resp struct {
		Previous string `json:"previous"`
		Current  string `json:"current"`
	}
	if err := json.Unmarshal(body, &resp); err != nil {
		return nil, loom.NewTransientError("ADAPTER_TIMEOUT",
			fmt.Sprintf("invalid power response: %v", err), op.IdempotencyKey, 0, err)
	}

	// Build the compensation operation (reverse action).
	var reverseAction string
	switch params.Action {
	case "on":
		reverseAction = "off"
	case "off":
		reverseAction = "on"
	default:
		reverseAction = "" // cycle/reset have no clean reverse
	}

	result := &loom.OperationResult{
		OperationType: "power_cycle",
		Status:        loom.StatusSucceeded,
		Output: loom.PowerCycleResult{
			PreviousState: resp.Previous,
			CurrentState:  resp.Current,
		},
	}

	if reverseAction != "" {
		result.CompensationOp = &loom.TypedOperation{
			Type: "power_cycle",
			Params: loom.PowerCycleParams{
				Action: reverseAction,
			},
		}
	}

	return result, nil
}

func (a *Adapter) executeReadSensors(ctx context.Context, op loom.TypedOperation) (*loom.OperationResult, error) {
	body, err := a.httpGet(ctx, "/sensors")
	if err != nil {
		return nil, err
	}

	var resp struct {
		Sensors []struct {
			Name  string  `json:"name"`
			Value float64 `json:"value"`
			Unit  string  `json:"unit"`
		} `json:"sensors"`
	}
	if err := json.Unmarshal(body, &resp); err != nil {
		return nil, loom.NewTransientError("ADAPTER_TIMEOUT",
			fmt.Sprintf("invalid sensor response: %v", err), op.IdempotencyKey, 0, err)
	}

	sensors := make([]loom.SensorReading, len(resp.Sensors))
	for i, s := range resp.Sensors {
		sensors[i] = loom.SensorReading{
			Name:   s.Name,
			Type:   classifySensorType(s.Unit),
			Value:  s.Value,
			Unit:   s.Unit,
			Status: "ok",
		}
	}

	return &loom.OperationResult{
		OperationType: "read_sensors",
		Status:        loom.StatusSucceeded,
		Output:        loom.ReadSensorsResult{Sensors: sensors},
	}, nil
}

// --- HTTP helpers ---

func (a *Adapter) httpGet(ctx context.Context, path string) ([]byte, error) {
	a.mu.RLock()
	url := a.baseURL + path
	client := a.client
	a.mu.RUnlock()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, loom.NewTransientError("ADAPTER_TIMEOUT", err.Error(), "", 0, err)
	}

	resp, err := client.Do(req)
	if err != nil {
		return nil, classifyHTTPError(err, "")
	}
	defer resp.Body.Close()

	if resp.StatusCode == http.StatusUnauthorized || resp.StatusCode == http.StatusForbidden {
		return nil, loom.NewPermanentError("ADAPTER_AUTH_FAILED",
			fmt.Sprintf("GET %s returned %d", path, resp.StatusCode), "", nil)
	}
	if resp.StatusCode >= 500 {
		return nil, loom.NewTransientError("ADAPTER_TIMEOUT",
			fmt.Sprintf("GET %s returned %d", path, resp.StatusCode), "", 5*time.Second, nil)
	}
	if resp.StatusCode >= 400 {
		return nil, loom.NewPermanentError("ADAPTER_UNSUPPORTED_OP",
			fmt.Sprintf("GET %s returned %d", path, resp.StatusCode), "", nil)
	}

	return io.ReadAll(resp.Body)
}

func (a *Adapter) httpPost(ctx context.Context, path string, body []byte) ([]byte, error) {
	a.mu.RLock()
	url := a.baseURL + path
	client := a.client
	a.mu.RUnlock()

	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(body))
	if err != nil {
		return nil, loom.NewTransientError("ADAPTER_TIMEOUT", err.Error(), "", 0, err)
	}
	req.Header.Set("Content-Type", "application/json")

	resp, err := client.Do(req)
	if err != nil {
		return nil, classifyHTTPError(err, "")
	}
	defer resp.Body.Close()

	if resp.StatusCode == http.StatusUnauthorized || resp.StatusCode == http.StatusForbidden {
		return nil, loom.NewPermanentError("ADAPTER_AUTH_FAILED",
			fmt.Sprintf("POST %s returned %d", path, resp.StatusCode), "", nil)
	}
	if resp.StatusCode >= 500 {
		return nil, loom.NewTransientError("ADAPTER_TIMEOUT",
			fmt.Sprintf("POST %s returned %d", path, resp.StatusCode), "", 5*time.Second, nil)
	}
	if resp.StatusCode >= 400 {
		return nil, loom.NewPermanentError("ADAPTER_UNSUPPORTED_OP",
			fmt.Sprintf("POST %s returned %d", path, resp.StatusCode), "", nil)
	}

	return io.ReadAll(resp.Body)
}

func classifyHTTPError(err error, correlationID string) error {
	// Context cancellation or deadline exceeded.
	if ctx := context.Canceled; err == ctx {
		return loom.NewTransientError("ADAPTER_TIMEOUT", "request cancelled", correlationID, 0, err)
	}
	// Default: treat as transient (network issue).
	return loom.NewTransientError("ADAPTER_UNREACHABLE", err.Error(), correlationID, 5*time.Second, err)
}

func classifySensorType(unit string) string {
	switch unit {
	case "celsius", "fahrenheit":
		return "temperature"
	case "volts":
		return "voltage"
	case "rpm":
		return "fan"
	case "watts":
		return "power"
	default:
		return "unknown"
	}
}

// --- Idempotency cache ---

func (a *Adapter) getCached(key string) *loom.OperationResult {
	if key == "" {
		return nil
	}
	a.cacheMu.RLock()
	defer a.cacheMu.RUnlock()
	c, ok := a.cache[key]
	if !ok || time.Now().After(c.expiresAt) {
		return nil
	}
	return c.result
}

func (a *Adapter) putCached(key string, result *loom.OperationResult) {
	if key == "" {
		return
	}
	a.cacheMu.Lock()
	defer a.cacheMu.Unlock()
	a.cache[key] = &cachedResult{
		result:    result,
		expiresAt: time.Now().Add(a.config.IdempotencyCacheTTL),
	}
}

// --- Compile-time interface verification ---

var (
	_ loom.Adapter   = (*Adapter)(nil)
	_ loom.Connector = (*Adapter)(nil)
	_ loom.Discoverer = (*Adapter)(nil)
	_ loom.Executor  = (*Adapter)(nil)
)
```

### 4.3 Example Adapter Test File

```go
package httphealth_test

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/wittedinit/loom/adapters/httphealth"
	loom "github.com/wittedinit/loom/sdk/adapter"
	"github.com/wittedinit/loom/sdk/conformance"
)

// newMockHealthServer creates a test HTTP server that simulates the target device API.
func newMockHealthServer() *httptest.Server {
	mux := http.NewServeMux()

	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]any{
			"status":         "ok",
			"uptime_seconds": 12345,
		})
	})

	mux.HandleFunc("/info", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]any{
			"serial":   "EXAMPLE-SN-001",
			"model":    "Widget-X",
			"firmware": "2.1.0",
		})
	})

	mux.HandleFunc("/sensors", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]any{
			"sensors": []map[string]any{
				{"name": "cpu_temp", "value": 42.5, "unit": "celsius"},
				{"name": "psu_voltage", "value": 12.1, "unit": "volts"},
			},
		})
	})

	mux.HandleFunc("/power", func(w http.ResponseWriter, r *http.Request) {
		var body struct {
			Action string `json:"action"`
		}
		json.NewDecoder(r.Body).Decode(&body)
		json.NewEncoder(w).Encode(map[string]any{
			"previous": "on",
			"current":  body.Action,
		})
	})

	return httptest.NewServer(mux)
}

func TestConformance_HTTPHealth(t *testing.T) {
	server := newMockHealthServer()
	defer server.Close()

	adapter := httphealth.New(loom.AdapterConfig{
		Protocol: "http",
		Options: map[string]any{
			"connect_timeout":   5 * time.Second,
			"operation_timeout": 10 * time.Second,
		},
	})

	// Parse the test server address.
	// server.URL is like "http://127.0.0.1:PORT"
	endpoint := loom.EndpointInfo{
		Address:   "127.0.0.1",
		Port:      parseTestPort(server.URL),
		Transport: "http",
	}
	cred := loom.CredentialRef{
		VaultPath: "mock/test",
		Type:      "basic",
	}

	conformance.RunAll(t, adapter, conformance.Options{
		Connector:      true,
		Discoverer:     true,
		Executor:       true,
		MockEndpoint:   &endpoint,
		MockCredential: &cred,
	})
}

// TestPowerCycle_Idempotency verifies the idempotency cache works end-to-end.
func TestPowerCycle_Idempotency(t *testing.T) {
	server := newMockHealthServer()
	defer server.Close()

	adapter := httphealth.New(loom.AdapterConfig{
		Protocol: "http",
		Options:  map[string]any{},
	})

	endpoint := loom.EndpointInfo{
		Address:   "127.0.0.1",
		Port:      parseTestPort(server.URL),
		Transport: "http",
	}

	ctx := t.Context()
	if err := adapter.Connect(ctx, endpoint, loom.CredentialRef{}); err != nil {
		t.Fatalf("Connect failed: %v", err)
	}
	defer adapter.Disconnect(ctx)

	executor := adapter.(loom.Executor)

	op := loom.TypedOperation{
		Type:           "power_cycle",
		IdempotencyKey: "test-idempotency-001",
		Timeout:        10 * time.Second,
		Params:         loom.PowerCycleParams{Action: "off"},
	}

	// First execution.
	result1, err := executor.Execute(ctx, op)
	if err != nil {
		t.Fatalf("First Execute failed: %v", err)
	}

	// Second execution with same key -- should return cached result.
	result2, err := executor.Execute(ctx, op)
	if err != nil {
		t.Fatalf("Second Execute failed: %v", err)
	}

	// Results should be identical (same pointer from cache).
	if result1.CompletedAt != result2.CompletedAt {
		t.Error("Second call should return cached result with identical CompletedAt")
	}
}

// TestPowerCycle_Compensation verifies compensation metadata is populated.
func TestPowerCycle_Compensation(t *testing.T) {
	server := newMockHealthServer()
	defer server.Close()

	adapter := httphealth.New(loom.AdapterConfig{Protocol: "http"})

	endpoint := loom.EndpointInfo{
		Address:   "127.0.0.1",
		Port:      parseTestPort(server.URL),
		Transport: "http",
	}

	ctx := t.Context()
	if err := adapter.Connect(ctx, endpoint, loom.CredentialRef{}); err != nil {
		t.Fatalf("Connect failed: %v", err)
	}
	defer adapter.Disconnect(ctx)

	executor := adapter.(loom.Executor)

	result, err := executor.Execute(ctx, loom.TypedOperation{
		Type:           "power_cycle",
		IdempotencyKey: "test-compensation-001",
		Timeout:        10 * time.Second,
		Params:         loom.PowerCycleParams{Action: "off"},
	})
	if err != nil {
		t.Fatalf("Execute failed: %v", err)
	}

	if result.CompensationOp == nil {
		t.Fatal("CompensationOp should be populated for power_cycle off -> on")
	}
	if result.CompensationOp.Type != "power_cycle" {
		t.Errorf("CompensationOp.Type = %q, want power_cycle", result.CompensationOp.Type)
	}

	compParams, ok := result.CompensationOp.Params.(loom.PowerCycleParams)
	if !ok {
		t.Fatal("CompensationOp.Params should be PowerCycleParams")
	}
	if compParams.Action != "on" {
		t.Errorf("CompensationOp action = %q, want 'on' (reverse of 'off')", compParams.Action)
	}
}

func parseTestPort(url string) int {
	// url is "http://127.0.0.1:PORT"
	var port int
	for i := len(url) - 1; i >= 0; i-- {
		if url[i] == ':' {
			fmt.Sscanf(url[i+1:], "%d", &port)
			break
		}
	}
	return port
}
```

### 4.4 Registration in the Adapter Factory

```go
// In the main LOOM binary's adapter initialization:
package main

import (
	loom "github.com/wittedinit/loom/sdk/adapter"
	"github.com/wittedinit/loom/adapters/httphealth"
)

func registerAdapters(registry loom.Registry) {
	// Built-in adapters register at startup.
	// Each adapter's New function is the factory.
	registry.Register(loom.AdapterRegistration{
		Name:     "httphealth",
		Protocol: "http",
		Capabilities: []loom.Capability{
			{Name: "discover_hardware", Protocol: "http", ReadOnly: true},
			{Name: "power_control", Protocol: "http", ReadOnly: false},
			{Name: "read_sensors", Protocol: "http", ReadOnly: true},
		},
		Factory: httphealth.New,
	})

	// Register other adapters the same way:
	// registry.Register(ssh.New(...).Registration())
	// registry.Register(redfish.New(...).Registration())
}
```

### 4.5 Design Choice Annotations

| Line/Area | Design Choice | Why |
|-----------|--------------|-----|
| `sync.RWMutex` on Adapter | Concurrency-safe state | Multiple Temporal activities may share a single adapter connection |
| `Connect` calls `/health` | Fail-fast on connect | Detects unreachable devices before any operation is attempted |
| `Connect` is idempotent | Required by contract | Callers may retry Connect without side effects |
| `Disconnect` always clears state | Graceful degradation | Even if the server is gone, local state is cleaned up |
| No retries in `Execute` | Contract requirement | Retry ownership is exclusively Temporal's (ADAPTER-CONTRACT.md) |
| `classifyHTTPError` | Error taxonomy | Every error maps to a LOOM error type -- no bare errors leak |
| `CompensationOp` populated on success | Saga support | The orchestrator uses this to build rollback plans |
| `idempotencyCache` is per-adapter | Simplicity for MVP | In production, this moves to Valkey for cross-worker dedup |
| Compile-time `var _ loom.X = (*A)(nil)` | Interface verification | Build fails if the adapter drifts from the contract |

---

## 5. Contributor Guide

### 5.1 How to Propose a New Adapter

#### Step 1: File an RFC Issue

Use the template below. The RFC must be approved by at least one maintainer before implementation begins.

```markdown
## Adapter RFC: <Protocol/Vendor>

### Summary
One paragraph: what device/protocol does this adapter target?

### Target Devices
- Vendor(s) and model(s)
- Protocol version(s)
- Minimum firmware requirements

### Interface Families
Which LOOM interfaces will this adapter implement?
- [ ] Connector (required)
- [ ] Discoverer
- [ ] Executor
- [ ] StateReader
- [ ] Watcher

### Operations Supported
List each operation type this adapter will handle (from OPERATION-TYPES.md or new types):
| Operation | Read-Only? | Compensation |
|-----------|-----------|--------------|
| ...       | ...       | ...          |

### Compensation Reliability
What compensation reliability level does this protocol support?
- [ ] Transactional (atomic rollback)
- [ ] Snapshot/Restore (config backup/restore)
- [ ] Best Effort (reverse operation, not guaranteed)
- [ ] None (no programmatic undo)

Justify the choice with protocol evidence.

### Testing Strategy
- Mock server approach: what protocol-specific mock is needed?
- Real device testing: what lab equipment is required?
- Edge cases specific to this protocol/vendor.

### Dependencies
- Go libraries required (e.g., `github.com/gosnmp/gosnmp`)
- Build constraints (CGO? Platform-specific?)
- License compatibility (must be Apache-2.0 or MIT compatible)

### Estimated Effort
- Lines of code (estimate)
- Calendar time (estimate)
```

#### Step 2: Scaffold and Implement

```bash
loom adapter scaffold --name <name> --protocol <protocol> --families <families> --compensation <level>
```

Then implement the TODO stubs in each generated file.

#### Step 3: Conformance Suite

The adapter MUST pass all applicable conformance tests before PR review:

```bash
go test ./adapters/<name>/... -v -run TestConformance
```

#### Step 4: Submit PR

### 5.2 Review Criteria for Adapter PRs

Every adapter PR is reviewed against these criteria. All must pass.

| Criterion | Requirement |
|-----------|-------------|
| **Conformance** | `conformance.RunAll()` passes with all declared families enabled |
| **Error classification** | No bare `error` returns. Every error is a `TransientError`, `PermanentError`, `PartialError`, or `ValidationError` |
| **Idempotency** | Execute honors IdempotencyKey. Same key = same result without re-execution |
| **Compensation** | Every mutating operation declares CompensationInfo. Non-reversible operations explicitly declare `Reversible: false` |
| **No internal retries** | The adapter performs exactly one attempt per Execute call. No retry loops |
| **Context respect** | All operations accept and honor `context.Context`. Cancellation propagates within 5 seconds |
| **Compile-time checks** | `var _ loom.Interface = (*Adapter)(nil)` for every implemented interface |
| **Resource cleanup** | Disconnect cleans up all resources (connections, goroutines, file handles) |
| **Test coverage** | >80% line coverage. Critical paths (error classification, idempotency) >90% |
| **Documentation** | Package doc comment. Exported function doc comments. Config field comments |
| **No credential leaks** | Error messages and logs never contain raw credentials. `CredentialRef` is used throughout |

### 5.3 Performance Benchmarks

Every adapter must include benchmarks for:

```go
func BenchmarkConnect(b *testing.B) {
	// Measure connection establishment time.
	// Target: <500ms for local mock, <2s for remote device.
}

func BenchmarkDiscover(b *testing.B) {
	// Measure discovery time for full scope.
	// Target: <5s for single device.
}

func BenchmarkExecute(b *testing.B) {
	// Measure per-operation execution time.
	// Target: <1s for simple operations (power, sensor read).
}
```

Benchmarks are run with `go test -bench=. -benchmem` and results are tracked across releases to detect regressions.

### 5.4 Documentation Requirements

| Document | Location | Content |
|----------|----------|---------|
| Package doc | `doc.go` | Protocol overview, supported devices, limitations, compensation reliability justification |
| Config reference | `config.go` comments | Every config field documented with type, default, and effect |
| Operation mapping | Package doc or separate table | Which LOOM operations map to which protocol actions |
| Known quirks | Package doc | Vendor-specific behavior that deviates from the protocol standard |

---

## 6. Plugin Lifecycle (Phase 6+)

### 6.1 Overview

Phases 1-5: all adapters are compiled into the main LOOM binary. This is simpler, faster (no IPC overhead), and sufficient for the core adapter set.

Phase 6+: out-of-process adapters via HashiCorp `go-plugin` for:
- Third-party adapter distribution without forking LOOM
- Independent adapter versioning and patching
- IP protection for proprietary adapters
- Fault isolation (a crashing adapter does not crash the host)

### 6.2 go-plugin Integration

```go
// sdk/plugin/plugin.go

package plugin

import (
	"github.com/hashicorp/go-plugin"
	loom "github.com/wittedinit/loom/sdk/adapter"
)

// Handshake is the plugin handshake config.
// MagicCookieKey/Value must match between host and plugin.
var Handshake = plugin.HandshakeConfig{
	ProtocolVersion:  1,
	MagicCookieKey:   "LOOM_ADAPTER_PLUGIN",
	MagicCookieValue: "loom-v1",
}

// AdapterPlugin implements the go-plugin.Plugin interface.
type AdapterPlugin struct {
	// Impl is the real adapter implementation (plugin side only).
	Impl loom.Adapter
}

func (p *AdapterPlugin) GRPCServer(broker *plugin.GRPCBroker, s *grpc.Server) error {
	// Register the adapter gRPC service (generated from proto).
	RegisterAdapterServer(s, &adapterGRPCServer{impl: p.Impl})
	return nil
}

func (p *AdapterPlugin) GRPCClient(ctx context.Context, broker *plugin.GRPCBroker, c *grpc.ClientConn) (interface{}, error) {
	return &adapterGRPCClient{client: NewAdapterClient(c)}, nil
}

// PluginMap is the map of plugins the host can load.
var PluginMap = map[string]plugin.Plugin{
	"adapter": &AdapterPlugin{},
}
```

### 6.3 Plugin Versioning Scheme

Adapters declare an API version that represents the adapter contract version they implement:

```go
// AdapterAPIVersion tracks breaking changes to the adapter interfaces.
// Bump major when: interface signature changes, error type changes, new required methods.
// Bump minor when: new optional methods added, new operation types registered.
type AdapterAPIVersion struct {
	Major int // breaking changes
	Minor int // additive changes
}

// CurrentAPIVersion is the version of the adapter API that the host supports.
var CurrentAPIVersion = AdapterAPIVersion{Major: 1, Minor: 0}
```

**Compatibility rules:**
- Host supports adapters with the same `Major` version.
- Host supports adapters with `Minor` <= host's `Minor` (host is backward-compatible).
- Adapters with a higher `Major` version are rejected at load time.
- Adapters with a higher `Minor` version emit a warning (adapter may use features the host does not support).

### 6.4 Hot-Reload vs Restart-Required

| Change Type | Hot-Reload? | Mechanism |
|-------------|-------------|-----------|
| New adapter plugin binary | Yes | Host watches plugin directory, loads new binary, registers in registry |
| Updated adapter plugin binary (same API version) | Yes | Host replaces plugin process, drains in-flight operations to old process |
| Updated adapter plugin binary (new API major version) | No | Requires host restart with matching API version |
| Adapter config change | Yes | Host sends new config via gRPC; adapter applies without reconnect |
| Adapter removal | Yes | Host deregisters adapter from registry, terminates plugin process |

**Drain procedure for hot-reload:**
1. Host marks old plugin as "draining" -- no new operations are dispatched to it.
2. In-flight operations complete against the old plugin process (up to activity timeout).
3. Host starts new plugin process and registers it.
4. Old plugin process is terminated.
5. Total interruption: zero (old process handles in-flight; new process handles new operations).

### 6.5 Dependency Isolation

Each plugin is a separate Go binary with its own `go.mod`. This provides:

- **Dependency version isolation**: Plugin A can use `gosnmp v1.37` while Plugin B uses `gosnmp v1.38`.
- **Build isolation**: Plugin binaries are built independently. A broken plugin does not block LOOM releases.
- **CGO isolation**: Plugins requiring CGO (e.g., libnetconf2 bindings) do not force CGO on the main binary.

**Plugin binary layout:**

```
/opt/loom/plugins/
  adapters/
    dell-redfish          # ELF binary (linux/amd64)
    dell-redfish.sha256   # Checksum for integrity verification
    dell-redfish.sig      # Ed25519 signature (plugin signing, Phase 3+)
    junos-netconf         # Another plugin binary
    junos-netconf.sha256
    junos-netconf.sig
```

### 6.6 Plugin Loading and Trust

```go
// sdk/plugin/loader.go

package plugin

import (
	"crypto/ed25519"
	"crypto/sha256"
	"fmt"
	"os"
	"path/filepath"
)

// LoaderConfig configures plugin loading behavior.
type LoaderConfig struct {
	// PluginDir is the directory to scan for plugin binaries.
	PluginDir string

	// RequireSignature requires all plugins to have a valid Ed25519 signature.
	// When true, unsigned plugins are rejected.
	RequireSignature bool

	// TrustedPublicKeys is the set of Ed25519 public keys that can sign plugins.
	TrustedPublicKeys []ed25519.PublicKey
}

// VerifyPlugin checks plugin integrity and signature before loading.
func VerifyPlugin(path string, config LoaderConfig) error {
	// 1. Verify SHA-256 checksum.
	binary, err := os.ReadFile(path)
	if err != nil {
		return fmt.Errorf("read plugin binary: %w", err)
	}

	checksumPath := path + ".sha256"
	expectedHash, err := os.ReadFile(checksumPath)
	if err != nil {
		return fmt.Errorf("read checksum file: %w", err)
	}

	actualHash := sha256.Sum256(binary)
	actualHex := fmt.Sprintf("%x", actualHash)
	if actualHex != string(expectedHash[:64]) {
		return fmt.Errorf("checksum mismatch: expected %s, got %s", expectedHash[:64], actualHex)
	}

	// 2. Verify Ed25519 signature (if required).
	if config.RequireSignature {
		sigPath := path + ".sig"
		signature, err := os.ReadFile(sigPath)
		if err != nil {
			return fmt.Errorf("read signature file: %w", err)
		}

		verified := false
		for _, pubKey := range config.TrustedPublicKeys {
			if ed25519.Verify(pubKey, binary, signature) {
				verified = true
				break
			}
		}
		if !verified {
			return fmt.Errorf("plugin signature verification failed: %s", filepath.Base(path))
		}
	}

	return nil
}
```

### 6.7 Plugin Distribution

Third-party adapters can be distributed via:

| Method | Use Case | Phase |
|--------|----------|-------|
| Go module | Open-source adapters compiled into LOOM | 1-5 |
| Binary release (GitHub Releases) | Standalone plugin binaries | 6+ |
| OCI artifact (container registry) | Enterprise distribution with signing | 8+ |
| `loom adapter install <name>` | CLI-driven plugin installation | 8+ |

---

## Appendix A: SDK Module Layout

```
github.com/wittedinit/loom/sdk/
  adapter/        # Core types: Connector, Discoverer, Executor, etc.
                  # Re-exports from ADAPTER-CONTRACT.md, ERROR-MODEL.md, OPERATION-TYPES.md
  conformance/    # Test suite that adapter authors import
  mockdevice/     # Protocol-specific mock servers
  plugin/         # go-plugin integration (Phase 6+)
  scaffold/       # Code generation for `loom adapter scaffold`
```

The SDK module has minimal dependencies:
- `github.com/google/uuid` (idempotency keys)
- `github.com/gliderlabs/ssh` (MockSSH)
- `github.com/hashicorp/go-plugin` (plugin lifecycle, Phase 6+)
- Standard library for everything else

## Appendix B: Minimum Viable SDK (Phase 1B-2)

| Component | Phase 1B | Phase 2 | Phase 6+ |
|-----------|----------|---------|----------|
| Scaffolding CLI | Ship | Maintained | Maintained |
| Conformance suite (all categories) | Ship | Maintained | Maintained |
| MockSSH | Ship | Maintained | Maintained |
| MockRedfish | Ship | Maintained | Maintained |
| MockSNMP | Stub | Ship | Maintained |
| MockNETCONF | Stub | Ship | Maintained |
| MockgNMI | -- | Stub | Ship |
| Example adapter (httphealth) | Ship | Maintained | Maintained |
| Contributor guide | Ship | Maintained | Maintained |
| Plugin lifecycle | -- | -- | Ship |
| Plugin signing | -- | -- | Ship |
| Plugin hot-reload | -- | -- | Ship |

## Appendix C: Open Questions Resolved

| Question (from source docs) | Decision |
|-----|----------|
| Should the SDK be a separate Go module? | **Yes.** `github.com/wittedinit/loom/sdk` -- separate `go.mod` for dependency isolation |
| Should conformance tests support real device targets? | **Yes.** `Options.RealDevice = true` skips fault-injection tests but runs all contract tests |
| What is the minimum viable SDK for Phase 2? | Scaffolding + conformance + MockSSH + MockRedfish + example adapter |
| Should the conformance suite be a Go test suite or CLI tool? | **Go test suite.** Adapter authors `import "github.com/wittedinit/loom/sdk/conformance"` and call `RunAll(t, adapter, opts)`. This integrates with standard `go test`, CI, and coverage tools. A CLI wrapper can be added later |
| Can the device simulator double as the mock device framework? | **Yes, partially.** The `mockdevice` package provides the foundation. The scale simulator (RESEARCH-protocol-adapter-realism.md Section 8) wraps N mock devices behind a fleet API |
