---
title: "Go Code Guidelines"
space: UCP
parent_page_id: "../implementation.md"
---

# Go Code Guidelines

---

## 1. Error Handling

### Current State

```go
// api-server/compute_handler.go:89
respondError(w, http.StatusInternalServerError, "Failed to verify tenant admin status: "+err.Error())

// api-server/main.go:604
respondError(w, http.StatusInternalServerError, "Failed to list databases: "+err.Error())

// api-server/main.go:554
respondError(w, http.StatusInternalServerError, "Failed to start workflow: "+err.Error())
```

And in the CLI:
```go
// cli/cmd/db/create.go:101-104
db, err := c.CreateDatabase(context.Background(), req)
if err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)  // inside RunE — bypasses cobra's error handling
}
```

### Problems

1. Raw `err.Error()` sent to HTTP clients leaks internal details (K8s API error messages, DSNs, internal hostnames).
2. No machine-readable error code — clients can only string-match to detect specific errors.
3. No request ID in the error response — impossible to correlate client error with server log.
4. `os.Exit(1)` inside a cobra `RunE` function bypasses deferred cleanup and makes the function untestable. `RunE` exists specifically to return an error.
5. No distinction between client errors (4xx, fix your request) and server errors (5xx, retry later).

### Option A — Sentinel error types with HTTP mapping (recommended for API server)

Define a small set of named error types. The response helper maps them to status codes and sanitizes messages before sending:

```go
// errors.go
type APIError struct {
    Code    string // machine-readable, e.g. "TENANT_NOT_FOUND"
    Message string // human-readable, safe to send to client
    status  int
    cause   error  // internal — never serialized
}

func (e *APIError) Error() string { return e.Message }
func (e *APIError) Unwrap() error { return e.cause }

func NotFound(code, msg string) *APIError {
    return &APIError{Code: code, Message: msg, status: http.StatusNotFound}
}
func BadRequest(code, msg string) *APIError { ... }
func Internal(cause error) *APIError {
    // Never expose cause.Error() to the client
    return &APIError{Code: "INTERNAL_ERROR", Message: "an internal error occurred", status: 500, cause: cause}
}

func respondErr(w http.ResponseWriter, reqID string, err error) {
    var apiErr *APIError
    if !errors.As(err, &apiErr) {
        apiErr = Internal(err)
    }
    // log the cause (with reqID) here — never send it to the client
    respondJSON(w, apiErr.status, map[string]string{
        "error":     apiErr.Code,
        "message":   apiErr.Message,
        "requestId": reqID,
    })
}
```

Trade-offs:
- Requires defining the error taxonomy upfront (small, finite set of codes).
- Callers must construct typed errors instead of `fmt.Errorf(...)` — slightly more verbose.
- But: clients can switch on `error` field, not string-match on `message`. Logs contain the real cause. Request IDs make production debugging tractable.

### Option B — Structured errors via `fmt.Errorf` + sentinel values

Use `errors.Is` / `errors.As` with package-level sentinel errors:

```go
var ErrNotFound = errors.New("not found")
var ErrForbidden = errors.New("forbidden")

// handler:
if errors.Is(err, ErrNotFound) {
    respondError(w, 404, "resource not found")
    return
}
```

Trade-offs:
- Simpler, no new types. Works well for small APIs.
- But: loses the `Code` field (client still string-matches), and the status-mapping logic scatters across every handler.

### Option C — Middleware-level error recovery

Return errors from handlers (change handler signature), catch them in a middleware, map centrally:

```go
type Handler func(w http.ResponseWriter, r *http.Request) error

func (s *APIServer) wrap(h Handler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if err := h(w, r); err != nil {
            respondErr(w, requestID(r), err)
        }
    }
}
```

Trade-offs:
- Handlers return errors naturally — most idiomatic Go.
- Centralized error-to-response mapping eliminates the `respondError(w, code, msg)` call from every handler.
- Requires changing all handler signatures, which is a larger refactor.
- This is the cleanest long-term model. Combined with Option A's typed errors, it's the production target.

### For the CLI

Fix `os.Exit` inside `RunE`. `RunE` exists to return errors:

```go
// Before:
db, err := c.CreateDatabase(context.Background(), req)
if err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)  // WRONG
}

// After:
db, err := c.CreateDatabase(cmd.Context(), req)
if err != nil {
    return fmt.Errorf("create database: %w", err)  // cobra prints this
}
```

**Recommendation: Option C (middleware recovery) + Option A (typed errors)**. Option C cleans up all handlers at once; Option A makes errors machine-readable. Both together are ~100 lines of infrastructure code that pays off immediately.

---

## 2. Context Propagation

### Current State

```go
// api-server/compute_handler.go:119
we, err := s.temporalClient.ExecuteWorkflow(
    context.Background(),   // wrong
    ...
)

// api-server/main.go:599
list, err := s.k8sClient.Resource(gvr).List(
    ctx,  // but ctx := context.Background() on line 593
    ...
)

// cli/cmd/db/create.go:100
db, err := c.CreateDatabase(context.Background(), req)  // wrong
```

The correct context to pass is `r.Context()` (in HTTP handlers) or `cmd.Context()` (in cobra commands). `context.Background()` never cancels, so:
- If the client disconnects, the K8s/Temporal call keeps running.
- If the HTTP server shuts down gracefully, in-flight requests don't stop.
- Deadlines set on the request (e.g., via a timeout middleware) are silently ignored.

### Option A — Pass `r.Context()` directly (correct, minimal change)

```go
// handler method:
we, err := s.temporalClient.ExecuteWorkflow(r.Context(), workflowOptions, ...)
list, err := s.k8sClient.Resource(gvr).List(r.Context(), metav1.ListOptions{})
```

Trade-offs: Trivially correct. One-line change per call site. No downsides.

### Option B — Derive a child context with a per-operation timeout

```go
ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
defer cancel()
list, err := s.k8sClient.Resource(gvr).List(ctx, metav1.ListOptions{})
```

Trade-offs: Adds explicit timeout per operation on top of request context. Prevents a slow K8s API from holding a goroutine indefinitely even when the client has not disconnected. More defensive, slightly more verbose.

### For the CLI

cobra attaches a context to the command via `cmd.Context()`. Use it:

```go
// Instead of context.Background():
db, err := c.CreateDatabase(cmd.Context(), req)
```

The CLI context respects OS signals (Ctrl+C) when wired correctly in `main()`:
```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
defer stop()
rootCmd.SetContext(ctx)
```

**Recommendation: Option B** — derive a child context with a conservative timeout per operation. This is defense-in-depth: the request context cancels if the client disconnects, and the timeout cancels if the downstream system hangs.

---

## 3. Logging

### Current State

```go
// All over api-server/
log.Printf("✅ Started workflow: %s for database: %s", workflowID, req.Name)
log.Printf("🗑️  Deleted compute instance: %s", name)
log.Printf("User %s is NOT admin of tenant %s", userEmail, tenantRNS)

// api-server/settings_handler.go:97
log.Printf("[db] connected to PostgreSQL")
```

`log.Printf` writes to stderr with a timestamp prefix. Problems:
1. No log level — debug noise mixed with errors.
2. No structured fields — you can't query "all requests for tenant X" in a log aggregator.
3. Emoji characters break log parsers and grep.
4. No request ID correlation between the log line and the HTTP response.

### Option A — `log/slog` (stdlib, Go 1.21+, recommended)

```go
import "log/slog"

// In main, configure once:
slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
})))

// In a middleware, attach request ID to every log from this request:
func withRequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        reqID := r.Header.Get("X-Request-ID")
        if reqID == "" {
            reqID = uuid.New().String()
        }
        w.Header().Set("X-Request-ID", reqID)
        ctx := context.WithValue(r.Context(), ctxKeyRequestID, reqID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// In a handler:
slog.InfoContext(r.Context(), "workflow started",
    "workflow_id", workflowID,
    "name", req.Name,
    "tenant_id", tenantID,
    "provider", req.Provider,
)
```

Output: `{"level":"INFO","time":"...","msg":"workflow started","workflow_id":"db-request-foo-...","name":"foo","tenant_id":"rns:...","provider":"omnia"}`

Trade-offs: Zero new dependencies (stdlib). JSON output works with every log aggregator. Structured fields enable queries. Adding `slog.InfoContext` with the right context automatically picks up the request ID from the context value. The only cost is slightly more verbose call sites than `log.Printf`.

### Option B — `zerolog` or `zap`

High-performance structured loggers with an allocation-free API. Widely used in production Go services.

```go
// zerolog example:
log.Info().Str("workflow_id", workflowID).Str("name", req.Name).Msg("workflow started")
```

Trade-offs: Faster than `slog` at high throughput (benchmarks show 3-5x). Requires a new dependency. For a platform like UCP where log volume is measured in thousands per minute (not millions), the performance difference is irrelevant. `slog` is sufficient and requires zero deps.

### Option C — Keep `log.Printf`, add a wrapper

Wrap `log.Printf` to prepend a request ID:

```go
func logf(ctx context.Context, format string, args ...interface{}) {
    if id, ok := ctx.Value(ctxKeyRequestID).(string); ok {
        log.Printf("[%s] "+format, append([]interface{}{id}, args...)...)
    } else {
        log.Printf(format, args...)
    }
}
```

Trade-offs: Minimal change. But output is still unstructured text — grep-able but not machine-parseable. Not worth it given that `slog` is now in stdlib.

**Recommendation: Option A (`log/slog`)**. Zero dependencies, structured, supports context propagation. Migration is mechanical: replace `log.Printf(...)` with `slog.InfoContext(ctx, msg, key, val, ...)`.

---

## 4. Dependency Injection and Interfaces

### Current State

```go
// api-server/settings_handler.go:64
client := NewHorizonAPIClient()  // creates a new client on EVERY call to isUserTenantAdmin
resp, err := client.httpClient.Do(req)
```

`isUserTenantAdmin` is called on every create/delete request. It creates a fresh `HorizonAPIClient` each time — no connection reuse, no mocking, no caching. The `APIServer` struct holds `horizonClient *HorizonAPIClient` as a field but `isUserTenantAdmin` ignores it and creates a new one.

Other concrete-type dependencies in `APIServer`:
```go
type APIServer struct {
    temporalClient      client.Client     // interface — correct
    k8sClient           dynamic.Interface // interface — correct
    db                  *ucpdb.DB         // concrete pointer — hard to mock
    horizonClient       *HorizonAPIClient // concrete pointer — hard to mock
    quotaProvider       QuotaProvider     // interface — correct
}
```

`db` and `horizonClient` are concrete. The inconsistency means DB and Horizon can't be replaced in tests; Temporal and K8s can.

### Option A — Interface all external dependencies

Define a minimal interface for each dependency. Use the concrete type in production; use a fake in tests.

```go
// DB interface (only what the server actually calls):
type store interface {
    CreateAuditLog(entry AuditEntry)
    GetSession(id string) (*Session, error)
    GetQuotaPolicy(tenantRNS, resource string) (*QuotaPolicy, error)
    // ... add as needed
}

// Horizon interface:
type tenantChecker interface {
    IsTenantAdmin(ctx context.Context, authToken, tenantRNS string) (bool, error)
}

// Use in APIServer:
type APIServer struct {
    temporalClient client.Client
    k8sClient      dynamic.Interface
    store          store         // was *ucpdb.DB
    tenantChecker  tenantChecker // was *HorizonAPIClient
    quotaProvider  QuotaProvider
}
```

Trade-offs:
- Requires extracting the interface definition (mechanical work).
- Fakes are hand-written — can drift from real implementation.
- But: every handler becomes unit-testable without a real DB or Horizon. This is the standard Go pattern and the most impactful testability improvement.

### Option B — Accept concrete types but add a test constructor

Keep concrete types in `APIServer` but add a constructor that accepts them as parameters, so tests can inject test doubles without interfaces:

```go
func NewAPIServer(tc client.Client, k8s dynamic.Interface, db *ucpdb.DB, ...) *APIServer
```

Trade-offs: No new interfaces. But `*ucpdb.DB` can only be constructed with a real PostgreSQL connection, so tests still can't run without infrastructure. Not a meaningful improvement for unit tests.

### Option C — Use `testify/mock` with generated mocks

Define interfaces, then use `mockery` to generate mocks automatically from the interface.

Trade-offs: Same testability benefits as Option A, but mock generation removes the hand-written fake maintenance burden. Adds `testify/mock` and `mockery` to the toolchain. Worth it if the interface surface is large.

**Recommendation: Option A for the three non-interface dependencies (`db`, `horizonClient`, and the in-handler `isUserTenantAdmin` pattern)**. The interface definitions are small (~5 methods each). Testability improvement is immediate.

Also fix `isUserTenantAdmin`: it should use `s.horizonClient` (the field), not create a new one:

```go
// Before (settings_handler.go:64):
client := NewHorizonAPIClient()

// After:
client := s.horizonClient  // reuse the pooled client on the struct
```

---

## 5. Validation

### Current State

Validation is inline in every handler, with no shared logic:

```go
// main.go CreateDatabase:
if req.Name == "" {
    respondError(w, http.StatusBadRequest, "name is required")
    return
}
if req.Provider == "" {
    req.Provider = "omnia"
}
switch req.Provider {
case "omnia":
    if req.OmniaServiceID == "" { ... }
    if req.EngineVersion == "" { ... }
    // defaults ...
case "gcp":
    if req.ProjectID == "" { ... }
    // defaults ...
default:
    respondError(...)
}

// compute_handler.go CreateComputeInstance:
if req.Name == "" { ... }
if req.Provider == "" { req.Provider = "gce" }
if req.Provider != "gce" { ... }
if req.ProjectID == "" { ... }
if req.TenantID == "" { ... }
```

Each handler has its own validation block. There's no reuse. No validation of field format (name goes directly into K8s resource names and Temporal workflow IDs with no sanitization). The only shared helper is `sanitizeTenantID`.

### Option A — Validation method on each request type

Each request type gets a `Validate() error` method. The handler calls it once.

```go
func (r *ComputeInstanceRequest) Validate() error {
    if r.Name == "" {
        return BadRequest("NAME_REQUIRED", "name is required")
    }
    if !validResourceName.MatchString(r.Name) {
        return BadRequest("NAME_INVALID", "name must be lowercase alphanumeric with hyphens, max 53 chars")
    }
    if r.TenantID == "" {
        return BadRequest("TENANT_REQUIRED", "tenantId is required")
    }
    // default-filling:
    if r.Provider == "" { r.Provider = "gce" }
    return nil
}

// In handler:
if err := req.Validate(); err != nil {
    respondErr(w, reqID, err)
    return
}
```

Trade-offs:
- Minimal: no new dependencies. Validation lives with the type that owns the data.
- Default-filling and validation are co-located — easier to reason about.
- Testable in isolation from the handler.

### Option B — `go-playground/validator` struct tags

```go
type ComputeInstanceRequest struct {
    Name      string `json:"name" validate:"required,dns_rfc1123_label"`
    TenantID  string `json:"tenantId" validate:"required"`
    ProjectID string `json:"projectId" validate:"required_if=Provider gce"`
    Provider  string `json:"provider" validate:"oneof=gce"`
}
```

And a decode+validate helper:

```go
func decodeAndValidate[T any](r *http.Request) (T, error) {
    var v T
    if err := json.NewDecoder(r.Body).Decode(&v); err != nil {
        return v, BadRequest("INVALID_JSON", err.Error())
    }
    if err := validate.Struct(v); err != nil {
        return v, BadRequest("VALIDATION_FAILED", err.Error())
    }
    return v, nil
}
```

Trade-offs:
- Declarative — the struct documents its own constraints.
- `required_if`, `oneof`, `min`, `max`, `regexp` are built-in.
- But: adds a dependency, tag syntax is harder to read than code for complex cross-field rules, and default-filling can't be done in tags (still needs code).

### Option C — OpenAPI-generated validation

If going spec-first (from the architectural doc), `oapi-codegen` generates handler stubs with request validation baked in. The spec is the single source of truth for field constraints.

Trade-offs: Best long-term solution — constraints defined once in the spec, enforced in Go server and TS client. Requires the spec to exist first.

**Recommendation: Option A now, migrate to Option C when the spec exists**. Option A costs almost nothing and pays off immediately in handler clarity. The regex for resource names is the most important missing constraint:

```go
var validResourceName = regexp.MustCompile(`^[a-z][a-z0-9-]{0,51}[a-z0-9]$`)
```

---

## 6. Code Duplication Across Handlers

### 6a. Audit Logging

Every create and delete handler contains this block verbatim:

```go
if s.db != nil {
    p, _ := authpkg.PrincipalFromContext(r.Context())
    entry := ucpdb.AuditEntry{
        Action:    "compute.create",
        Resource:  req.Name,
        IPAddress: r.RemoteAddr,
        UserAgent: r.UserAgent(),
        Metadata:  map[string]any{"provider": req.Provider, "workflow_id": we.GetID()},
    }
    if p != nil {
        entry.UserID = p.UserID
        entry.SessionID = p.SessionID
    }
    s.db.CreateAuditLog(entry)
}
```

This appears in: `CreateDatabase`, `DeleteDatabase`, `CreateComputeInstance`, `DeleteComputeInstance`, `CreateStorageBucket`, `DeleteStorageBucket`, `CreateKubernetesCluster`, `DeleteKubernetesCluster`, `CreateTerraformResource` — and every future resource handler.

**Extract to a helper:**

```go
func (s *APIServer) auditLog(r *http.Request, action, resource string, meta map[string]any) {
    if s.db == nil {
        return
    }
    p, _ := authpkg.PrincipalFromContext(r.Context())
    entry := ucpdb.AuditEntry{
        Action:    action,
        Resource:  resource,
        IPAddress: r.RemoteAddr,
        UserAgent: r.UserAgent(),
        Metadata:  meta,
    }
    if p != nil {
        entry.UserID = p.UserID
        entry.SessionID = p.SessionID
    }
    s.db.CreateAuditLog(entry)
}

// Usage:
s.auditLog(r, "compute.create", req.Name, map[string]any{
    "provider":    req.Provider,
    "workflow_id": we.GetID(),
})
```

This eliminates ~15 lines from each handler and makes the `db == nil` guard and principal extraction a single source of truth.

### 6b. XR YAML Construction

`createXDatabaseYAML`, `createXComputeInstanceYAML`, `createXStorageBucketYAML`, and equivalents all:
1. Build a `map[string]interface{}` for metadata
2. Build a `map[string]interface{}` for parameters
3. Build a `map[string]interface{}` for `spec.crossplane.compositionSelector.matchLabels`
4. Call `yaml.Marshal(xr)` with `_` for the error

The repeated structure:

```go
// Every handler:
metadata := map[string]interface{}{
    "name":      req.Name,
    "namespace": "default",
    "labels":    map[string]interface{}{"platform.ucp.io/tenant": sanitizeTenantID(req.TenantID)},
    "annotations": map[string]interface{}{
        "temporal.io/workflow-id":   workflowID,
        "platform.ucp.io/tenant-id": req.TenantID,
    },
}
// ... then marshal with ignored error
yamlBytes, _ := yaml.Marshal(xr)
```

**Option A: XR builder struct**

```go
type XRBuilder struct {
    APIVersion  string
    Kind        string
    Name        string
    Namespace   string
    TenantID    string
    WorkflowID  string
    Parameters  map[string]interface{}
    MatchLabels map[string]string
    ExtraAnnotations map[string]string
}

func (b XRBuilder) YAML() (string, error) {
    metadata := map[string]interface{}{
        "name":      b.Name,
        "namespace": b.Namespace,
        "labels":    map[string]interface{}{"platform.ucp.io/tenant": sanitizeTenantID(b.TenantID)},
        "annotations": map[string]interface{}{
            "temporal.io/workflow-id":   b.WorkflowID,
            "platform.ucp.io/tenant-id": b.TenantID,
        },
    }
    for k, v := range b.ExtraAnnotations {
        metadata["annotations"].(map[string]interface{})[k] = v
    }
    xr := map[string]interface{}{
        "apiVersion": b.APIVersion,
        "kind":       b.Kind,
        "metadata":   metadata,
        "spec": map[string]interface{}{
            "parameters": b.Parameters,
            "crossplane": map[string]interface{}{
                "compositionSelector": map[string]interface{}{
                    "matchLabels": b.MatchLabels,
                },
            },
        },
    }
    out, err := yaml.Marshal(xr)
    return string(out), err  // error is now surfaced, not swallowed
}
```

Trade-offs: One builder, all handlers use it. The marshal error is now returned (fixes the silent `_` swallow). Each handler only specifies what's unique to its resource type.

**Option B: Typed struct + yaml tags**

Define a proper Go struct for XDatabase, XComputeInstance, etc. with `yaml:` tags. Marshal it directly instead of constructing a `map[string]interface{}`.

Trade-offs: Type-safe, no string-key typos, no `_` swallow possible. More upfront work to define the structs. This is the clean long-term approach but requires more initial code.

**Recommendation: Option A for now** — it's a mechanical extraction that fixes the silent error and removes duplication with minimal restructuring.

### 6c. Condition Parsing (Status Determination)

`parseXDatabase` and `parseXComputeInstance` both iterate `status.conditions`, extract `type`, `status`, `reason`, `message`, and determine a human-readable status string. They do this with identical code, except `parseXDatabase` has more complex status logic (unavailable, failed, provisioning cases).

Extract the common part:

```go
type xrConditions struct {
    ReadyStatus  string
    ReadyReason  string
    ReadyMessage string
    SyncedStatus string
    Messages     []string
}

func extractConditions(obj *unstructured.Unstructured) xrConditions {
    var result xrConditions
    conds, found, _ := unstructured.NestedSlice(obj.Object, "status", "conditions")
    if !found {
        return result
    }
    for _, c := range conds {
        m, ok := c.(map[string]interface{})
        if !ok { continue }
        // ... extract fields, populate result
    }
    return result
}
```

Each handler then calls `extractConditions(obj)` and applies its own status logic on top. This removes ~30 lines of identical condition-iteration code from every `parseX` function.

---

## 7. Package Structure: Everything in `package main`

### Current State

All API server code — handlers, types, utilities, YAML builders, Temporal helpers — lives in `package main`. This means:

1. You can't import any of it from tests in a separate package (forces `package main` tests only).
2. You can't reuse types between the API server and other tools.
3. Compiler sees all files as one compilation unit — no encapsulation at all.
4. `var xComputeInstanceGVR = ...` is a package-level var in `compute_handler.go`, visible everywhere.

Compare with the CLI and temporal worker, which both use proper package hierarchy:
- `cli/internal/client/` — clean package with a public `Client` type
- `cli/internal/auth/` — encapsulated auth logic
- `temporal-worker/internal/activities/` — activities as their own package
- `temporal-worker/internal/workflows/` — workflows as their own package

The api-server is the outlier. Everything in `package main` is an anti-pattern for anything larger than a ~200 line program.

### Option A — Internal packages per concern

```
backend/api-server/
├── main.go              (wiring only, <100 lines)
├── internal/
│   ├── handler/         (HTTP handlers — no business logic)
│   │   ├── database.go
│   │   ├── compute.go
│   │   ├── storage.go
│   │   └── ...
│   ├── service/         (business logic — tenant check, YAML build, etc.)
│   │   ├── provisioner.go
│   │   └── xr_builder.go
│   ├── middleware/      (auth, logging, CORS, metrics)
│   │   └── ...
│   └── model/           (request/response types, shared across handler+service)
│       └── types.go
```

Trade-offs: Clean separation, each package is independently testable, `internal/` prevents accidental external imports. Cost: moving files and updating import paths. This is non-trivial but purely mechanical.

### Option B — Separate files, keep `package main`

Keep `package main` but enforce one file per resource. Types in `types.go`, utilities in `util.go`.

Trade-offs: Zero import-path changes. Compiler still sees everything. Can't write `package handler_test` external tests. Discipline degrades without structural enforcement. This is what the codebase already does for some resources (compute, drift, etc.) and it's better than having everything in `main.go`, but it doesn't solve the testability problem.

**Recommendation: Option A**. The CLI and temporal-worker already demonstrate the right structure. The api-server should match them. The migration is mechanical (move code, update imports), and the payoff is that handlers become testable without booting the full server.

---

## 8. Temporal Workflow: Typed Inputs vs Map Inputs

### Current State

Mixed: `RequestDatabaseWorkflow` now uses a typed `RequestDatabaseInput` struct (good), but other workflows still use `map[string]interface{}` passed from the API server:

```go
// api-server/compute_handler.go:112
workflowInput := map[string]interface{}{
    "namespace": "default",
    "xrYaml":    xrYAML,
    "name":      req.Name,
    "provider":  req.Provider,
}

we, err := s.temporalClient.ExecuteWorkflow(
    context.Background(),
    workflowOptions,
    "RequestComputeWorkflow",
    workflowInput,   // untyped map
)
```

On the worker side, `RequestComputeWorkflow` presumably receives this as a `map[string]interface{}` and accesses fields by string key. If a key is misspelled, the error appears at runtime in the workflow execution, not at compile time in the API server.

`RequestDatabaseWorkflow` is better — it defines `RequestDatabaseInput` which is serialized through Temporal. But even that struct has a comment in Japanese:
```go
SecretData map[string]string `json:"secretData"` // base64 decode済みにして返す方がUIに優しい
```

### Option A — Typed structs for all workflow inputs (recommended)

All workflows should define their own input/output types in the worker package. The API server imports these types (or duplicates them — see below on coupling).

```go
// temporal-worker/internal/workflows/request_compute.go
type RequestComputeInput struct {
    Namespace  string `json:"namespace"`
    XRYAML     string `json:"xrYaml"`
    Name       string `json:"name"`
    Provider   string `json:"provider"`
    TenantID   string `json:"tenantId"`
}

// api-server calls with:
s.executeWorkflow(r.Context(), options, "RequestComputeWorkflow", workflows.RequestComputeInput{...})
```

Trade-offs: Compile-time type checking. Temporal serializes the struct to JSON anyway, so it's compatible with the existing worker. The API server and worker must share the type — either as a shared package, or by duplicating the struct (the `drift_handler.go` pattern: `driftWorkflowInput` mirrors `DriftApprovalInput` locally).

The `drift_handler.go` approach of local duplication is actually pragmatic: the API server doesn't need to import the worker binary, it just needs to know the JSON shape. Define the type in both places, document it as "mirrors X in worker module". This is already done for drift.

### Option B — Keep maps, add JSON schema validation

Validate the map against a schema before sending:

```go
requiredKeys := []string{"namespace", "xrYaml", "name", "provider"}
for _, k := range requiredKeys {
    if _, ok := workflowInput[k]; !ok {
        return fmt.Errorf("missing required workflow input: %s", k)
    }
}
```

Trade-offs: Runtime validation instead of compile-time. Better than nothing but does not give type safety. Not worth it when Option A costs almost the same.

### Activity Options Duplication

Every workflow defines activity options inline:

```go
// request_database.go:
ao := workflow.ActivityOptions{
    StartToCloseTimeout: 25 * time.Minute,
    RetryPolicy: &temporal.RetryPolicy{MaximumAttempts: 1},
}

// request_compute.go:
ao := workflow.ActivityOptions{
    StartToCloseTimeout: 15 * time.Minute,
    RetryPolicy: &temporal.RetryPolicy{MaximumAttempts: 1},
}

// request_kubernetes_cluster.go:
ao := workflow.ActivityOptions{
    StartToCloseTimeout: 30 * time.Minute,
    RetryPolicy: &temporal.RetryPolicy{MaximumAttempts: 1},
}
```

Extract to named constants and constructors:

```go
// activities/options.go
func ProvisioningOptions(timeout time.Duration) workflow.ActivityOptions {
    return workflow.ActivityOptions{
        StartToCloseTimeout: timeout,
        RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 1},
    }
}

func SecretReadOptions() workflow.ActivityOptions {
    return workflow.ActivityOptions{
        StartToCloseTimeout: 30 * time.Second,
        RetryPolicy: &temporal.RetryPolicy{
            InitialInterval:    5 * time.Second,
            BackoffCoefficient: 2.0,
            MaximumInterval:    30 * time.Second,
            MaximumAttempts:    3,
        },
    }
}
```

If the retry policy for provisioning activities needs to change, it changes in one place.

---

## 9. CLI: `mustClient()` and Exit Patterns

### Current State

```go
// cli/cmd/root.go:53
func mustClient() *client.Client {
    cfg, err := config.Load()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error loading config: %v\n", err)
        os.Exit(1)   // hard exit in a library-style function
    }
    // ...
    store := auth.NewStore(storePath)
    creds, err := store.Load(serverURL)
    if err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)   // hard exit again
    }
    // ...
}

// cli/cmd/db/create.go:100-104
db, err := c.CreateDatabase(context.Background(), req)
if err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
}
```

`os.Exit(1)` in cobra `RunE` functions (and functions called from them) bypasses `defer`, prevents clean shutdown, and makes the code untestable. `RunE` exists specifically to allow returning an error that cobra formats and exits with.

### Option A — Return errors through RunE

```go
// cli/cmd/root.go
func buildClient() (*client.Client, error) {
    cfg, err := config.Load()
    if err != nil {
        return nil, fmt.Errorf("load config: %w", err)
    }
    // ...
    creds, err := store.Load(serverURL)
    if err != nil {
        return nil, err  // already has user-facing message
    }
    // ...
    return client.New(serverURL, creds.AccessToken, ctx), nil
}

// cli/cmd/db/create.go
RunE: func(cmd *cobra.Command, args []string) error {
    c, err := buildClient()
    if err != nil {
        return err  // cobra handles exit
    }
    db, err := c.CreateDatabase(cmd.Context(), req)
    if err != nil {
        return err  // cobra prints and exits with code 1
    }
    fmt.Printf("Database %q created (workflow: %s)\n", db.Name, db.WorkflowID)
    return nil
},
```

Trade-offs: cobra calls `os.Exit(1)` after printing the returned error. Same user-visible behavior, but: deferred cleanup runs, the function is testable by calling `RunE` directly and checking the returned error, no `os.Stderr` duplication.

### Option B — Keep `mustClient()` but move it to `PersistentPreRunE`

cobra has a `PersistentPreRunE` hook that runs before all subcommands. Build the client once there and store it:

```go
var ucpClient *client.Client

rootCmd.PersistentPreRunE = func(cmd *cobra.Command, args []string) error {
    var err error
    ucpClient, err = buildClient()
    return err
}
```

Trade-offs: Cleaner than `mustClient()` in every command. But auth commands (login, logout) don't need a client — they should not run `PersistentPreRunE`. This requires selective skipping, which complicates the root setup.

**Recommendation: Option A**. Mechanically replace all `os.Exit(1)` inside command functions with `return err`. Rename `mustClient()` to `buildClient() (*client.Client, error)`. The test improvement alone justifies it.

---

## 10. Testing: What Exists and What's Missing

### What the Tests Look Like

```go
// api-server/main_test.go — tests for YAML construction (good):
func TestCreateXDatabaseYAMLGCPLabelsAndParameters(t *testing.T) {
    server := &APIServer{}
    xrYAML := server.createXDatabaseYAML(DatabaseRequest{...}, "wf-gcp-456", "qa")
    // parse and assert fields
}

// temporal-worker/request_database_test.go (single test for a helper function):
func TestTokenRefreshWorkflowID(t *testing.T) {
    // tests fmt.Sprintf format string
}
```

What's tested: YAML output shape, token refresh workflow ID format. Both are pure functions — no dependencies — so they're easy to test.

What's not tested:
- Any HTTP handler (`CreateDatabase`, `ListDatabases`, `DeleteDatabase`, etc.)
- The workflow execution path
- Status parsing from K8s conditions
- Error paths (K8s not found, Temporal failure, auth failure)
- Drift handler logic
- Quota handler

### Option A — `httptest` with fake dependencies

For handlers, `net/http/httptest` is already in stdlib. Combine with Option 4 (interfaces for dependencies) to inject fakes:

```go
func TestCreateComputeInstance_MissingName(t *testing.T) {
    server := &APIServer{
        temporalClient: &fakeTemporalClient{},
        k8sClient:      fake.NewSimpleDynamicClient(scheme),
        db:             nil,
        horizonClient:  &fakeTenantChecker{isAdmin: true},
    }
    body := `{"provider":"gce","projectId":"proj","tenantId":"rns:..."}`
    req := httptest.NewRequest("POST", "/api/v1/compute", strings.NewReader(body))
    w := httptest.NewRecorder()
    server.CreateComputeInstance(w, req)

    if w.Code != http.StatusBadRequest {
        t.Fatalf("expected 400, got %d", w.Code)
    }
    var resp map[string]string
    json.Unmarshal(w.Body.Bytes(), &resp)
    if resp["error"] != "NAME_REQUIRED" {
        t.Fatalf("expected NAME_REQUIRED, got %s", resp["error"])
    }
}
```

Trade-offs: Fast (no real server), tests the actual handler logic, exercises status code and response shape. Requires extracting interfaces first (section 4).

### Option B — Table-driven tests for status parsing

`parseXDatabase` and `parseXComputeInstance` are pure functions (take a `*unstructured.Unstructured`, return a response struct). They can be tested without any server setup:

```go
func TestParseXDatabaseStatus(t *testing.T) {
    cases := []struct{
        name       string
        conditions []map[string]string
        wantStatus string
    }{
        {"ready", []map[string]string{{"type":"Ready","status":"True"}}, "ready"},
        {"provisioning", []map[string]string{{"type":"Synced","status":"True"}}, "provisioning"},
        {"reconcile error", []map[string]string{{"type":"Ready","status":"False","reason":"ReconcileError"}}, "failed"},
    }
    for _, tc := range cases {
        obj := buildUnstructured(tc.conditions)
        got := (&APIServer{}).parseXDatabase(obj)
        if got.Status != tc.wantStatus {
            t.Errorf("%s: want %s got %s", tc.name, tc.wantStatus, got.Status)
        }
    }
}
```

Trade-offs: Zero dependencies, extremely fast. These tests can be written today without any refactoring. Most valuable coverage: the status determination logic has many branches and string comparisons, all prone to regression.

### For Temporal workflows — use the test suite

The Temporal SDK test framework lets you run workflows and activities in process:

```go
func TestRequestDatabaseWorkflow_RejectsOnTimeout(t *testing.T) {
    suite := &testsuite.WorkflowTestSuite{}
    env := suite.NewTestWorkflowEnvironment()
    env.RegisterWorkflow(RequestDatabaseWorkflow)

    // Send rejection signal after 1 tick
    env.RegisterDelayedCallback(func() {
        env.SignalWorkflow("approval-signal", ApprovalSignal{Rejected: true})
    }, time.Millisecond)

    env.ExecuteWorkflow(RequestDatabaseWorkflow, RequestDatabaseInput{...})
    require.True(t, env.IsWorkflowCompleted())
    require.Error(t, env.GetWorkflowError())
}
```

`request_blueprint_test.go` already uses this pattern correctly. Expand it to the approval/rejection path, the provider routing path, and the token refresh path.

**Recommendation: Start with Option B (pure function tests) — they can be written this week. Then Option A (handler tests) after interfaces are extracted. Then Temporal test suite expansion.**

---

## 11. K8s GVR Constants: Scattered vs Centralized

### Current State

Every handler declares its own GVR constant:

```go
// compute_handler.go:
var xComputeInstanceGVR = schema.GroupVersionResource{
    Group: "platform.example.io", Version: "v1alpha1", Resource: "xcomputeinstances",
}

// main.go (inside ListDatabases):
gvr := schema.GroupVersionResource{
    Group: "platform.example.io", Version: "v1alpha1", Resource: "xdatabases",
}

// main.go (inside DeleteDatabase):
secretGVR := schema.GroupVersionResource{
    Group: "", Version: "v1", Resource: "secrets",
}
```

`xDatabaseGVR` is not a package-level var at all — it's constructed inline in each function. If `platform.example.io` needs to change to `platform.ucp.io`, you change it in 15 places.

### Fix: A single `gvr.go` file

```go
// internal/k8s/gvr.go (or a gvr.go at package level)
var (
    XDatabaseGVR = schema.GroupVersionResource{
        Group: "platform.example.io", Version: "v1alpha1", Resource: "xdatabases",
    }
    XComputeInstanceGVR = schema.GroupVersionResource{
        Group: "platform.example.io", Version: "v1alpha1", Resource: "xcomputeinstances",
    }
    XStorageBucketGVR = schema.GroupVersionResource{
        Group: "platform.example.io", Version: "v1alpha1", Resource: "xobjectstorages",
    }
    XKubernetesClusterGVR = schema.GroupVersionResource{
        Group: "platform.example.io", Version: "v1alpha1", Resource: "xkubernetesclusters",
    }
    SecretGVR = schema.GroupVersionResource{
        Group: "", Version: "v1", Resource: "secrets",
    }
    ProviderConfigGVR = schema.GroupVersionResource{
        Group: "gcp.upbound.io", Version: "v1beta1", Resource: "providerconfigs",
    }
)
```

Trade-offs: One change to update the group name across all handlers. No functional difference, pure maintainability. Zero cost.

---

## 12. HTTP Response Envelope Inconsistency

### Current State

```go
// databases:
respondJSON(w, 200, map[string]interface{}{"databases": databases, "count": len(databases)})

// compute:
respondJSON(w, 200, map[string]interface{}{"instances": instances, "count": len(instances)})

// drift:
respondJSON(w, 200, map[string]interface{}{"drifts": items})  // no count field

// approve workflow:
respondJSON(w, 200, map[string]string{"message": "Workflow approved successfully", "workflowId": workflowID})

// delete:
respondJSON(w, 200, map[string]string{"message": "Compute instance deleted successfully", "name": name})
```

Four different envelope shapes. The CLI's `client.go` handles this by defining per-response anonymous structs:

```go
var resp struct { Databases []DatabaseResponse `json:"databases"` }
// vs
var resp struct { Instances []ComputeInstanceResponse `json:"instances"` }
// vs
var resp struct { Clusters []KubernetesClusterResponse `json:"clusters"` }
```

The key name (`databases`, `instances`, `clusters`) differs per resource, so the CLI must know the key name for each endpoint. This is a client-side burden.

### Option A — Unified `items` envelope for list responses

```go
type ListResponse[T any] struct {
    Items []T `json:"items"`
    Total int  `json:"total"`
}

// Usage:
respondJSON(w, 200, ListResponse[ComputeInstanceResponse]{Items: instances, Total: len(instances)})
```

Trade-offs: Breaking change for the frontend (currently reads `.databases`, `.instances`, etc.). Requires frontend and CLI to update. But: worth doing before production. Client code becomes uniform:

```go
var resp ListResponse[DatabaseResponse]
c.do(ctx, "GET", "/api/v1/databases", nil, &resp)
return resp.Items, nil
```

### Option B — Keep resource-specific key, standardize everything else

Keep `{"databases": [...], "count": N}` per resource but enforce that all list responses have a `count` field (drift currently omits it) and all async responses have a `Location` header pointing to the workflow status.

Trade-offs: Non-breaking. Smaller change. But clients still need per-resource deserialization structs.

**Recommendation: Option A for the rewrite** — the unified `items` key makes client code dramatically simpler. Option B if you're making incremental changes and can't afford breaking changes.

---

## Summary Table

| # | Issue | Impacted Code | Fix Complexity | Impact |
|---|-------|---------------|----------------|--------|
| 1 | Error handling: raw errors to clients, no codes, no request ID | All handlers, CLI `RunE` | Medium | High |
| 2 | `context.Background()` in handlers | All handlers, CLI commands | Low (mechanical) | Medium |
| 3 | `log.Printf` with emoji, unstructured | All handlers | Low (mechanical) | Medium |
| 4 | Concrete deps (`db`, `horizonClient`) not injectable | `APIServer` struct | Medium | High (testability) |
| 5 | Inline validation, no name format check | All handlers | Low | High (security) |
| 6a | Audit log block copy-pasted in 9+ handlers | All create/delete handlers | Low | Medium |
| 6b | `yaml.Marshal` error swallowed, YAML construction duplicated | All `createX*YAML` functions | Low | Medium |
| 6c | Condition-parsing duplicated per resource | All `parseX*` functions | Low | Medium |
| 7 | `package main` god package | api-server/ | High | High (testability, structure) |
| 8a | `map[string]interface{}` workflow inputs (compute, k8s, storage) | compute_handler, k8s_handler, storage_handler | Low | Medium |
| 8b | Activity options duplicated across workflows | All workflow files | Low | Low |
| 9 | `os.Exit(1)` in cobra `RunE` functions | CLI cmd/ | Low (mechanical) | Medium (testability) |
| 10 | No handler-level tests | api-server/ | High | High |
| 11 | GVR constants scattered across handlers | All handlers | Low | Low |
| 12 | Inconsistent list response envelope | All list handlers | Medium | Low |

The three highest-leverage changes, in order:

1. **Extract interfaces for `db` and `horizonClient`** (enables all subsequent handler testing)
2. **Fix `context.Background()` → `r.Context()`** (correctness, one-line per call site)
3. **Extract `auditLog` helper and `extractConditions` helper** (removes the most copy-pasted code)
