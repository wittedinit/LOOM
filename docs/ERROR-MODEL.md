# Error Model

> Error taxonomy, propagation rules, and retry policies for all LOOM components.

## Core Principle

Every error in LOOM is categorized, carries a correlation ID for tracing, and declares whether it is retryable. There are no bare `error` strings. Every error that crosses a boundary (adapter to activity, activity to workflow, workflow to API) is a structured `LoomError`.

## Error Types

```go
package errors

import (
    "fmt"
    "time"
)

// LoomError is the base interface for all LOOM errors.
// Every error in the system must implement this interface.
type LoomError interface {
    error
    Code() string          // machine-readable error code
    Retryable() bool       // whether the caller should retry
    CorrelationID() string // request correlation ID for tracing
}

// --- TransientError ---
// Retry with backoff. The operation may succeed if tried again.
// Examples: network timeout, temporary service unavailable, rate limit.

type TransientError struct {
    code          string
    message       string
    correlationID string
    retryAfter    time.Duration // suggested wait before retry
    cause         error         // underlying error
}

func NewTransientError(code, message, correlationID string, retryAfter time.Duration, cause error) *TransientError {
    return &TransientError{
        code:          code,
        message:       message,
        correlationID: correlationID,
        retryAfter:    retryAfter,
        cause:         cause,
    }
}

func (e *TransientError) Error() string        { return fmt.Sprintf("[%s] %s", e.code, e.message) }
func (e *TransientError) Code() string         { return e.code }
func (e *TransientError) Retryable() bool      { return true }
func (e *TransientError) CorrelationID() string { return e.correlationID }
func (e *TransientError) RetryAfter() time.Duration { return e.retryAfter }
func (e *TransientError) Unwrap() error        { return e.cause }

// --- PermanentError ---
// Do not retry. The operation will never succeed with the same inputs.
// Examples: authentication failure, resource not found, invalid input.

type PermanentError struct {
    code          string
    message       string
    correlationID string
    cause         error
}

func NewPermanentError(code, message, correlationID string, cause error) *PermanentError {
    return &PermanentError{
        code:          code,
        message:       message,
        correlationID: correlationID,
        cause:         cause,
    }
}

func (e *PermanentError) Error() string        { return fmt.Sprintf("[%s] %s", e.code, e.message) }
func (e *PermanentError) Code() string         { return e.code }
func (e *PermanentError) Retryable() bool      { return false }
func (e *PermanentError) CorrelationID() string { return e.correlationID }
func (e *PermanentError) Unwrap() error        { return e.cause }

// --- PartialError ---
// Some operations succeeded, others failed. Compensation may be needed.
// Examples: multi-device provisioning where 3 of 5 succeeded, saga step failure.

type PartialError struct {
    code            string
    message         string
    correlationID   string
    succeededOps    []string // operations that completed successfully
    failedOps       []string // operations that failed
    compensationReq bool     // whether compensation is needed for succeeded ops
    cause           error
}

func NewPartialError(code, message, correlationID string, succeeded, failed []string, needsCompensation bool, cause error) *PartialError {
    return &PartialError{
        code:            code,
        message:         message,
        correlationID:   correlationID,
        succeededOps:    succeeded,
        failedOps:       failed,
        compensationReq: needsCompensation,
        cause:           cause,
    }
}

func (e *PartialError) Error() string        { return fmt.Sprintf("[%s] %s (succeeded: %d, failed: %d)", e.code, e.message, len(e.succeededOps), len(e.failedOps)) }
func (e *PartialError) Code() string         { return e.code }
func (e *PartialError) Retryable() bool      { return false } // partial errors need compensation, not retry
func (e *PartialError) CorrelationID() string { return e.correlationID }
func (e *PartialError) SucceededOps() []string { return e.succeededOps }
func (e *PartialError) FailedOps() []string    { return e.failedOps }
func (e *PartialError) NeedsCompensation() bool { return e.compensationReq }
func (e *PartialError) Unwrap() error          { return e.cause }

// --- ValidationError ---
// Input rejected before execution. Never retryable with the same input.
// Examples: missing required field, invalid IP format, unknown device type.

type ValidationError struct {
    code          string
    message       string
    correlationID string
    field         string // which field failed validation
    reason        string // why it failed
}

func NewValidationError(code, message, correlationID, field, reason string) *ValidationError {
    return &ValidationError{
        code:          code,
        message:       message,
        correlationID: correlationID,
        field:         field,
        reason:        reason,
    }
}

func (e *ValidationError) Error() string        { return fmt.Sprintf("[%s] %s: field '%s' %s", e.code, e.message, e.field, e.reason) }
func (e *ValidationError) Code() string         { return e.code }
func (e *ValidationError) Retryable() bool      { return false }
func (e *ValidationError) CorrelationID() string { return e.correlationID }
func (e *ValidationError) Field() string         { return e.field }
func (e *ValidationError) Reason() string        { return e.reason }
```

## Error Codes

### Adapter Errors

| Code | Type | Description |
|------|------|-------------|
| `ADAPTER_TIMEOUT` | Transient | Adapter call exceeded its timeout. The target device or controller did not respond in time. |
| `ADAPTER_AUTH_FAILED` | Permanent | Authentication to the target device or controller failed. Credentials are invalid or expired. |
| `ADAPTER_UNREACHABLE` | Transient | The target device or controller is not reachable on the network. May be temporary (network flap) or permanent (device down). Treated as transient for retry purposes; becomes permanent after max retries. |
| `ADAPTER_UNSUPPORTED_OP` | Permanent | The adapter does not support the requested operation for this device type or firmware version. |

### Resource Errors

| Code | Type | Description |
|------|------|-------------|
| `RESOURCE_NOT_FOUND` | Permanent | The requested resource does not exist. |
| `RESOURCE_CONFLICT` | Permanent | The operation conflicts with current resource state. Examples: creating a device with a hostname that already exists, updating a resource that was modified since it was read (optimistic concurrency violation). |

### Validation Errors

| Code | Type | Description |
|------|------|-------------|
| `VALIDATION_FAILED` | Permanent | Input did not pass validation. The `field` and `reason` provide specifics. |

### Workflow Errors

| Code | Type | Description |
|------|------|-------------|
| `WORKFLOW_CANCELLED` | Permanent | The workflow was cancelled by a user or system policy. |
| `APPROVAL_REJECTED` | Permanent | An approval gate was rejected by the approver. |
| `BUDGET_EXCEEDED` | Permanent | The operation would exceed the tenant's budget. |

### LLM Errors

| Code | Type | Description |
|------|------|-------------|
| `LLM_UNAVAILABLE` | Transient | The LLM service is not responding. Retry with backoff. |
| `LLM_CONFIDENCE_LOW` | Permanent | The LLM returned a response but with confidence below the acceptable threshold. Requires human review. |

## Retry Policy by Error Type

```go
package errors

import "time"

// RetryPolicy defines how retries are handled for different error types.
type RetryPolicy struct {
    MaxAttempts     int           // total attempts including the first try
    InitialInterval time.Duration // delay before first retry
    MaxInterval     time.Duration // cap on backoff growth
    BackoffFactor   float64       // multiplier per retry (exponential backoff)
}

// Default retry policies by error category.
var (
    // TransientRetryPolicy: retry aggressively with backoff.
    TransientRetryPolicy = RetryPolicy{
        MaxAttempts:     5,
        InitialInterval: 1 * time.Second,
        MaxInterval:     30 * time.Second,
        BackoffFactor:   2.0,
    }

    // AdapterTimeoutRetryPolicy: fewer retries, longer intervals.
    AdapterTimeoutRetryPolicy = RetryPolicy{
        MaxAttempts:     3,
        InitialInterval: 5 * time.Second,
        MaxInterval:     60 * time.Second,
        BackoffFactor:   3.0,
    }

    // LLMRetryPolicy: retry with longer backoff (LLM outages tend to be longer).
    LLMRetryPolicy = RetryPolicy{
        MaxAttempts:     3,
        InitialInterval: 10 * time.Second,
        MaxInterval:     2 * time.Minute,
        BackoffFactor:   2.0,
    }

    // NoRetryPolicy: for permanent errors.
    NoRetryPolicy = RetryPolicy{
        MaxAttempts: 1,
    }
)

// PolicyForCode returns the retry policy for a given error code.
func PolicyForCode(code string) RetryPolicy {
    switch code {
    case "ADAPTER_TIMEOUT":
        return AdapterTimeoutRetryPolicy
    case "ADAPTER_UNREACHABLE":
        return TransientRetryPolicy
    case "LLM_UNAVAILABLE":
        return LLMRetryPolicy
    case "ADAPTER_AUTH_FAILED", "ADAPTER_UNSUPPORTED_OP",
         "RESOURCE_NOT_FOUND", "RESOURCE_CONFLICT",
         "VALIDATION_FAILED", "WORKFLOW_CANCELLED",
         "APPROVAL_REJECTED", "BUDGET_EXCEEDED",
         "LLM_CONFIDENCE_LOW":
        return NoRetryPolicy
    default:
        return TransientRetryPolicy // unknown errors get transient treatment
    }
}
```

## Error Propagation

Errors propagate upward through three boundaries: adapter to activity, activity to workflow, workflow to API. At each boundary, the error is wrapped with additional context but the original error code and retryability are preserved.

### Adapter to Activity

```
Adapter returns error
  --> Activity catches error
  --> Activity wraps as LoomError (if not already)
  --> Activity returns ActivityResult{Success: false, Error: wrapped error}
  --> Temporal applies retry policy based on error.Retryable()
```

- If the adapter returns a raw `error`, the activity wraps it as a `TransientError` with code `ADAPTER_TIMEOUT` (conservative default).
- If the adapter returns a structured `LoomError`, the activity preserves it.
- The activity adds its own context (which adapter, which operation, which resource) to the error message.

### Activity to Workflow

```
Activity fails after all retries exhausted
  --> Temporal marks activity as failed
  --> Workflow receives activity error
  --> Workflow decides: compensate (saga), skip (optional step), or fail
```

- For saga workflows: a failed activity triggers compensation of all previously completed steps.
- For non-saga workflows: a failed activity fails the workflow.
- `PartialError` from a multi-target activity triggers compensation for succeeded targets.

### Workflow to API

```
Workflow fails
  --> Workflow status updated to "failed"
  --> Error code and message projected to WorkflowStatus in DB
  --> API returns error to caller on next status check
```

- The API does not expose raw error details to external callers. It maps error codes to appropriate HTTP responses.
- Internal error details are available in the audit trail and workflow history.

## HTTP Status Code Mapping

When errors reach the API boundary, they map to HTTP status codes:

| Error Code | HTTP Status | Response |
|------------|-------------|----------|
| `VALIDATION_FAILED` | 400 Bad Request | `{"error": {"code": "VALIDATION_FAILED", "message": "...", "field": "...", "reason": "..."}}` |
| `ADAPTER_AUTH_FAILED` | 401 Unauthorized | `{"error": {"code": "ADAPTER_AUTH_FAILED", "message": "..."}}` |
| `APPROVAL_REJECTED` | 403 Forbidden | `{"error": {"code": "APPROVAL_REJECTED", "message": "..."}}` |
| `RESOURCE_NOT_FOUND` | 404 Not Found | `{"error": {"code": "RESOURCE_NOT_FOUND", "message": "..."}}` |
| `RESOURCE_CONFLICT` | 409 Conflict | `{"error": {"code": "RESOURCE_CONFLICT", "message": "..."}}` |
| `BUDGET_EXCEEDED` | 402 Payment Required | `{"error": {"code": "BUDGET_EXCEEDED", "message": "..."}}` |
| `ADAPTER_TIMEOUT` | 504 Gateway Timeout | `{"error": {"code": "ADAPTER_TIMEOUT", "message": "..."}}` |
| `ADAPTER_UNREACHABLE` | 502 Bad Gateway | `{"error": {"code": "ADAPTER_UNREACHABLE", "message": "..."}}` |
| `ADAPTER_UNSUPPORTED_OP` | 501 Not Implemented | `{"error": {"code": "ADAPTER_UNSUPPORTED_OP", "message": "..."}}` |
| `WORKFLOW_CANCELLED` | 409 Conflict | `{"error": {"code": "WORKFLOW_CANCELLED", "message": "..."}}` |
| `LLM_UNAVAILABLE` | 503 Service Unavailable | `{"error": {"code": "LLM_UNAVAILABLE", "message": "..."}}` |
| `LLM_CONFIDENCE_LOW` | 422 Unprocessable Entity | `{"error": {"code": "LLM_CONFIDENCE_LOW", "message": "..."}}` |

## API Error Response Format

```go
// APIError is the standard error response returned by the LOOM API.
type APIError struct {
    Code          string `json:"code"`
    Message       string `json:"message"`
    CorrelationID string `json:"correlation_id"`
    Field         string `json:"field,omitempty"`   // only for validation errors
    Reason        string `json:"reason,omitempty"`  // only for validation errors
    RetryAfter    *int   `json:"retry_after,omitempty"` // seconds, only for transient errors
}
```

## Error Handling Rules

1. **Never swallow errors.** Every error must be logged, returned, or explicitly handled with a comment explaining why it is safe to ignore.
2. **Never retry permanent errors.** Check `Retryable()` before retrying. Retrying permanent errors wastes resources and delays failure reporting.
3. **Always include correlation ID.** Every error must carry the correlation ID from the request that caused it. This is how errors are traced across services.
4. **Always wrap raw errors at boundaries.** When an error crosses from one layer to another (adapter to activity, activity to workflow), wrap it as a `LoomError` with additional context.
5. **Never expose internal error details to external callers.** The API returns error codes and safe messages. Stack traces, adapter names, and internal IPs stay in logs.
6. **Log errors at the boundary where they are handled, not where they are created.** An error created in an adapter is logged by the activity that catches it, not by the adapter itself. This prevents duplicate logging.
7. **Use `PartialError` for multi-target operations.** If an operation targets 5 devices and 3 succeed, return a `PartialError` with the succeeded and failed lists. Do not return a generic error that loses the success information.
8. **Compensation errors do not mask the original error.** If compensation also fails, log the compensation error separately. The original error is what gets reported to the caller.
