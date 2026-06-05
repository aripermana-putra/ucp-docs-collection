---
title: "Backend Architecture"
space: UCP
parent_page_id: "../implementation.md"
---

# UCP Go Codebase Design

A concrete design specification for the production rewrite of the UCP backend.

---

## 1. Architecture: Clean Architecture (not DDD)

DDD is overkill for UCP. UCP is a provisioning platform — it doesn't have a rich domain
with complex invariants. Its "domain logic" is mostly: validate the request, check auth,
check quota, start a workflow, return a workflow ID. The heavy lifting happens in
Temporal and Crossplane.

Clean Architecture with three layers is the right fit:

```
┌──────────────────────────────────────────────────────┐
│  Transport Layer (HTTP handlers, request/response)    │  ← Controllers in Spring
├──────────────────────────────────────────────────────┤
│  Use Case Layer (orchestration, business rules)       │  ← @Service in Spring
├──────────────────────────────────────────────────────┤
│  Domain Layer (entities, repository interfaces)       │  ← Domain model + Repository interfaces
├──────────────────────────────────────────────────────┤
│  Infrastructure Layer (K8s, Postgres, Temporal, etc.) │  ← @Repository + external clients
└──────────────────────────────────────────────────────┘
```

Dependency rule: inner layers know nothing about outer layers.
The domain layer defines the repository interfaces. Infrastructure implements them.
Use cases depend on the interfaces, not the implementations.

### Directory Layout

```
backend/api-server/
├── cmd/
│   └── server/
│       └── main.go          # Wiring + startup (the "ApplicationContext")
│
├── internal/
│   ├── api/                 # Transport layer
│   │   ├── handler/         # Echo handlers, one file per resource
│   │   │   ├── database.go
│   │   │   ├── compute.go
│   │   │   ├── storage.go
│   │   │   ├── kubernetes.go
│   │   │   ├── blueprint.go
│   │   │   ├── drift.go
│   │   │   ├── quota.go
│   │   │   ├── workflow.go
│   │   │   └── settings.go
│   │   ├── middleware/      # Auth, logging, metrics, request ID, CORS
│   │   │   ├── auth.go
│   │   │   ├── requestid.go
│   │   │   ├── logging.go
│   │   │   └── recovery.go
│   │   └── server.go        # Echo setup, route registration, error handler
│   │
│   ├── usecase/             # Business logic, one file per resource
│   │   ├── database.go
│   │   ├── compute.go
│   │   ├── storage.go
│   │   ├── kubernetes.go
│   │   ├── blueprint.go
│   │   ├── drift.go
│   │   └── quota.go
│   │
│   ├── domain/              # Entities + repository interfaces
│   │   ├── database.go      # Database entity + DatabaseRepository interface
│   │   ├── compute.go
│   │   ├── errors.go        # Domain error types
│   │   └── tenant.go        # Tenant entity
│   │
│   ├── repository/          # Infrastructure implementations
│   │   ├── k8s/             # K8s-backed resource repos
│   │   │   ├── database.go
│   │   │   ├── compute.go
│   │   │   └── gvr.go       # All GVR constants in one place
│   │   └── postgres/        # Postgres-backed repos
│   │       ├── audit.go
│   │       ├── session.go
│   │       └── migrations/  # SQL migration files
│   │
│   ├── gateway/             # External service clients (infra adapters)
│   │   ├── temporal.go      # Temporal client wrapper
│   │   ├── horizon.go       # Horizon API client
│   │   ├── omnia.go         # Omnia API client wrapper
│   │   └── gcp/             # GCP clients (quota, etc.)
│   │
│   ├── auth/                # BFF session auth (unchanged from POC, already solid)
│   │   ├── bff.go
│   │   └── session.go
│   │
│   └── config/              # Typed config from env
│       └── config.go
│
├── api/
│   └── openapi.yaml         # The spec — source of truth for the HTTP surface
│
└── gen/
    └── api.gen.go           # Generated from openapi.yaml — never edit by hand
```

### Spring Analogy Map

| Spring Boot | Go (UCP) |
|---|---|
| `@RestController` | `handler/database.go` |
| `@Service` | `usecase/database.go` |
| `@Repository` | `repository/k8s/database.go` |
| `@Entity` | `domain/database.go` (plain struct) |
| `ApplicationContext` | `cmd/server/main.go` (manual wiring) |
| `@Autowired` | constructor parameter |
| `@RestControllerAdvice` | `api/server.go` error handler |
| `application.properties` | `internal/config/config.go` (env vars) |

---

## 2. Framework: Echo

### Candidates

**go-chi:**
- Pure router, no extras. Handlers are standard `http.Handler` — zero new types to learn.
- Zero magic. Full stdlib compatibility.
- No built-in request binding — you still write `json.NewDecoder(r.Body).Decode(&req)` everywhere.
- No built-in global error handler — you build one yourself as middleware.

**echo:**
- Router + request binding (`c.Bind(&req)`) + response helpers (`c.JSON(status, body)`).
- First-class `HTTPErrorHandler` — the exact hook for centralized error handling.
- `echo.Context` is not stdlib-compatible, but it's thin and well-documented.
- oapi-codegen has an `--generate echo/server` target that outputs echo-native stubs.

**gin:**
- Fastest of the three (benchmarks), large ecosystem.
- Custom context (`*gin.Context`), heavier API surface than echo.
- More "magic" around middleware chaining and context handling.
- The speed advantage is irrelevant for UCP — every request triggers a 20-30 min
  Temporal workflow. The HTTP layer is not the bottleneck.

### Decision: Echo

The deciding factor is `echo.HTTPErrorHandler`. This single hook gives you the
`@RestControllerAdvice` pattern without writing middleware from scratch.

The team comes from Spring Boot. `c.Bind(&req)` maps to `@RequestBody`, `c.JSON(200, data)`
maps to `ResponseEntity.ok(data)`, `c.Param("name")` maps to `@PathVariable`.
The mental model transfer is direct.

go-chi is the better choice if you're going stdlib-only and have strong Go instincts.
For a team relearning Go patterns, echo's guardrails reduce the surface area for mistakes.

---

## 3. Dependency Injection: Manual Constructor Injection

### Why Not Wire or Dig

uber/dig and google/wire are reflection-based or codegen-based containers.
They solve a real problem (wiring 50+ dependencies in a large service), but they
introduce a layer of indirection that obscures the actual dependency graph.

For UCP: the api-server has roughly 10-12 top-level dependencies. You can wire
all of them explicitly in ~80 lines of `main.go`. That's not a problem that needs
a solution.

### The Pattern: Constructor Injection

Every type in the use case and handler layers receives its dependencies as constructor
parameters. No global variables. No `init()` that reaches into package state.

```go
// domain/database.go — defines what the use case needs (interface, not implementation)
type DatabaseRepository interface {
    List(ctx context.Context, tenantID string) ([]Database, error)
    Get(ctx context.Context, name string) (*Database, error)
    Create(ctx context.Context, xrYAML string) error
    Delete(ctx context.Context, name string) error
}

// usecase/database.go — depends on the interface
type DatabaseUseCase struct {
    repo     domain.DatabaseRepository
    temporal gateway.TemporalGateway
    horizon  gateway.TenantChecker
    quota    domain.QuotaChecker
    audit    domain.AuditLogger
}

func NewDatabaseUseCase(
    repo domain.DatabaseRepository,
    temporal gateway.TemporalGateway,
    horizon gateway.TenantChecker,
    quota domain.QuotaChecker,
    audit domain.AuditLogger,
) *DatabaseUseCase {
    return &DatabaseUseCase{
        repo:     repo,
        temporal: temporal,
        horizon:  horizon,
        quota:    quota,
        audit:    audit,
    }
}

// api/handler/database.go — depends on the use case (via interface)
type DatabaseHandler struct {
    uc DatabaseUseCasePort  // interface, not *DatabaseUseCase
}

func NewDatabaseHandler(uc DatabaseUseCasePort) *DatabaseHandler {
    return &DatabaseHandler{uc: uc}
}
```

The application context (Spring's `@Configuration` class) lives in `cmd/server/main.go`:

```go
func main() {
    cfg := config.Load()  // reads env vars, panics if required vars missing

    // Infrastructure
    db          := postgres.NewDB(cfg.DatabaseURL)
    k8sClient   := k8s.NewClient()
    temporal    := gateway.NewTemporalGateway(cfg.Temporal)
    horizon     := gateway.NewHorizonClient(cfg.Horizon)

    // Repositories
    databaseRepo  := k8srepo.NewDatabaseRepository(k8sClient)
    computeRepo   := k8srepo.NewComputeRepository(k8sClient)
    auditRepo     := pgrepo.NewAuditRepository(db)

    // Use cases
    databaseUC := usecase.NewDatabaseUseCase(databaseRepo, temporal, horizon, auditRepo)
    computeUC  := usecase.NewComputeUseCase(computeRepo, temporal, horizon, auditRepo)

    // Handlers
    databaseH := handler.NewDatabaseHandler(databaseUC)
    computeH  := handler.NewComputeHandler(computeUC)

    // Server
    srv := api.NewServer(cfg, databaseH, computeH, ...)
    srv.Start()
}
```

This is exactly Spring Boot DI minus the reflection. You give up auto-wiring.
You gain: the complete dependency graph is visible in one file, compile-time errors
if you forget a dependency, trivial to test (just pass fakes to the constructors).

### When to Add uber/fx

If the dependency graph exceeds ~30 components and main.go becomes hard to read,
add uber/fx. It's the closest thing to Spring's application context in Go — it uses
`fx.Provide` and `fx.Invoke` with reflection-based wiring, and critically, it handles
lifecycle (OnStart/OnStop hooks) cleanly. But it's opt-in later, not a day-one requirement.

---

## 4. Error Handling: Centralized with Domain Errors

### The Spring Boot Pattern in Go

Spring Boot:
```java
// BusinessException thrown from service layer
throw new BusinessException("QUOTA_EXCEEDED", "quota exceeded for tenant " + tenantRNS);

// Caught globally by:
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handle(BusinessException e) {
        return ResponseEntity.status(e.getStatus()).body(new ErrorResponse(e.getCode(), e.getMessage()));
    }
}
```

Go with echo:
```go
// domain/errors.go — the BusinessException equivalent

type DomainError struct {
    Code    string // machine-readable, e.g. "QUOTA_EXCEEDED"
    Message string // human-readable, safe to send to client
    Status  int    // HTTP status code
    cause   error  // internal — never serialized, only logged
}

func (e *DomainError) Error() string { return e.Message }
func (e *DomainError) Unwrap() error { return e.cause }

// Constructor functions — like static factory methods in Java
func ErrNotFound(resource, name string) *DomainError {
    return &DomainError{
        Code:    "RESOURCE_NOT_FOUND",
        Message: fmt.Sprintf("%s %q not found", resource, name),
        Status:  http.StatusNotFound,
    }
}

func ErrQuotaExceeded(resource, tenantID string) *DomainError {
    return &DomainError{
        Code:    "QUOTA_EXCEEDED",
        Message: fmt.Sprintf("quota exceeded for resource %q in tenant %q", resource, tenantID),
        Status:  http.StatusTooManyRequests,
    }
}

func ErrBadRequest(code, msg string) *DomainError {
    return &DomainError{Code: code, Message: msg, Status: http.StatusBadRequest}
}

func ErrForbidden(msg string) *DomainError {
    return &DomainError{Code: "FORBIDDEN", Message: msg, Status: http.StatusForbidden}
}

func ErrInternal(cause error) *DomainError {
    // NEVER set Message to cause.Error() — internal details stay internal
    return &DomainError{
        Code:    "INTERNAL_ERROR",
        Message: "an internal error occurred",
        Status:  http.StatusInternalServerError,
        cause:   cause,
    }
}
```

The global error handler (the `@RestControllerAdvice` equivalent) is registered on the
echo instance. This is the single place in the entire codebase that decides how errors
become HTTP responses:

```go
// api/server.go
func customErrorHandler(err error, c echo.Context) {
    if c.Response().Committed {
        return
    }

    reqID := requestID(c)

    var de *domain.DomainError
    if errors.As(err, &de) {
        // Business error — safe to send to client
        if de.cause != nil {
            // Log the internal cause for debugging
            slog.ErrorContext(c.Request().Context(), "domain error",
                "code", de.Code, "cause", de.cause, "request_id", reqID)
        }
        _ = c.JSON(de.Status, ErrorResponse{
            Code:    de.Code,
            Message: de.Message,
            TraceID: reqID,
        })
        return
    }

    // echo's own errors (e.g. from Bind validation)
    var he *echo.HTTPError
    if errors.As(err, &he) {
        _ = c.JSON(he.Code, ErrorResponse{
            Code:    "REQUEST_ERROR",
            Message: fmt.Sprintf("%v", he.Message),
            TraceID: reqID,
        })
        return
    }

    // System error — log full details, never send to client
    slog.ErrorContext(c.Request().Context(), "unhandled error",
        "error", err, "request_id", reqID)
    _ = c.JSON(http.StatusInternalServerError, ErrorResponse{
        Code:    "INTERNAL_ERROR",
        Message: "an internal error occurred",
        TraceID: reqID,
    })
}

e.HTTPErrorHandler = customErrorHandler
```

Handlers and use cases just return errors. They never touch `http.ResponseWriter`
for error responses:

```go
// usecase/database.go
func (u *DatabaseUseCase) Create(ctx context.Context, req CreateDatabaseRequest) (*domain.Database, error) {
    if err := u.quota.CheckQuota(ctx, req.TenantID, "database"); err != nil {
        return nil, domain.ErrQuotaExceeded("database", req.TenantID)
    }
    // ... if something unexpected goes wrong:
    if err := u.repo.Create(ctx, xrYAML); err != nil {
        return nil, domain.ErrInternal(fmt.Errorf("create database XR: %w", err))
    }
    return db, nil
}

// api/handler/database.go
func (h *DatabaseHandler) CreateDatabase(c echo.Context) error {
    var req CreateDatabaseRequest
    if err := c.Bind(&req); err != nil {
        return domain.ErrBadRequest("INVALID_REQUEST", "request body is malformed")
    }
    if err := c.Validate(&req); err != nil {
        return err  // validation errors handled by global handler
    }
    db, err := h.uc.Create(c.Request().Context(), req)
    if err != nil {
        return err  // DomainError propagates up to global handler
    }
    return c.JSON(http.StatusAccepted, AsyncResponse{
        Code:       "PROVISIONING_STARTED",
        WorkflowID: db.WorkflowID,
        Message:    "database provisioning started",
    })
}
```

This is the exact Spring Boot pattern. Your Go colleagues who "don't know this pattern"
are treating errors as pure values that must be handled at the call site. That's valid
Go style too, but centralized mapping is equally valid and significantly cleaner for HTTP APIs.
The key is that echo's `HTTPErrorHandler` is the `@ControllerAdvice` equivalent.

### Validation

Register a validator on the echo instance:

```go
import "github.com/go-playground/validator/v10"

type echoValidator struct{ v *validator.Validate }
func (ev *echoValidator) Validate(i interface{}) error {
    return ev.v.Struct(i)
}

e.Validator = &echoValidator{v: validator.New()}
```

Request types declare constraints via struct tags:

```go
type CreateDatabaseRequest struct {
    Name      string `json:"name" validate:"required,min=2,max=53,hostname_rfc1123"`
    TenantID  string `json:"tenantId" validate:"required"`
    Provider  string `json:"provider" validate:"required,oneof=omnia gcp"`
    ProjectID string `json:"projectId" validate:"required_if=Provider gcp"`
}
```

Validation errors flow through the global handler. No per-handler validation blocks.

---

## 5. Logging: log/slog

### Decision

`log/slog` is stdlib since Go 1.21. Level-based (Debug/Info/Warn/Error), structured
(key-value pairs), JSON output for machine parsing, context-aware via `InfoContext`.
Zero new dependencies. This is the correct answer for production Go logging.

Avoid zerolog/zap unless you have evidence of logging being a performance bottleneck.
For UCP (low request volume, long-running provisioning), it never will be.

### Setup

```go
// cmd/server/main.go
logLevel := slog.LevelInfo
if cfg.Debug {
    logLevel = slog.LevelDebug
}
slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: logLevel,
    // AddSource adds file:line to every log entry — useful in production
    AddSource: !cfg.Debug,
})))
```

### Request ID Middleware

Inject a request ID into every request context. The global error handler reads it.
Every log line from a handler carries it automatically via `slog.InfoContext(ctx, ...)`.

```go
// middleware/requestid.go
type ctxKey struct{}

func RequestID(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        id := c.Request().Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.New().String()
        }
        c.Response().Header().Set("X-Request-ID", id)
        ctx := context.WithValue(c.Request().Context(), ctxKey{}, id)
        c.SetRequest(c.Request().WithContext(ctx))
        return next(c)
    }
}

func RequestIDFromContext(ctx context.Context) string {
    id, _ := ctx.Value(ctxKey{}).(string)
    return id
}
```

### Structured Logging in Use Cases

```go
// Use case logs carry structured fields — no string formatting
slog.InfoContext(ctx, "database provisioning started",
    "name", req.Name,
    "tenant_id", req.TenantID,
    "provider", req.Provider,
    "workflow_id", workflowID,
    "request_id", middleware.RequestIDFromContext(ctx),
)

slog.ErrorContext(ctx, "horizon API call failed",
    "tenant_id", tenantID,
    "status_code", resp.StatusCode,
    "error", err,
    "request_id", middleware.RequestIDFromContext(ctx),
)
```

Log levels:
- `Debug` — internal flow (e.g., "polling K8s for ready status, attempt 3")
- `Info` — business events (workflow started, resource created)
- `Warn` — recoverable issues (token refresh retry, degraded dependency)
- `Error` — unhandled errors, system-level failures

---

## 6. Fault Tolerance: failsafe-go

### What Needs Protection

Not every external call needs a circuit breaker. K8s API is in-cluster (reliable).
Temporal is in-cluster or co-located. The calls that need protection are:

| External Call | Risk | Protection Needed |
|---|---|---|
| Horizon API (tenant check) | External network, auth dependency | Circuit breaker + retry + timeout |
| Omnia token endpoint | External, auth-critical | Retry + timeout |
| GCP Cloud Monitoring (quota) | External, non-critical path | Timeout + fallback |
| Temporal workflow submission | In-cluster, reliable | Timeout only |

### Library: failsafe-go

`github.com/failsafe-go/failsafe-go` — the Go equivalent of Resilience4j.
Supports circuit breaker, retry, timeout, rate limiter, fallback, all composable.

```go
// gateway/horizon.go

import "github.com/failsafe-go/failsafe-go"
import "github.com/failsafe-go/failsafe-go/circuitbreaker"
import "github.com/failsafe-go/failsafe-go/retrypolicy"
import "github.com/failsafe-go/failsafe-go/timeout"

type HorizonClient struct {
    http    *http.Client
    baseURL string
    cb      circuitbreaker.CircuitBreaker[*tenantResponse]
    retry   retrypolicy.RetryPolicy[*tenantResponse]
}

func NewHorizonClient(cfg HorizonConfig) *HorizonClient {
    cb := circuitbreaker.Builder[*tenantResponse]().
        WithFailureThreshold(3, 5).     // open after 3 failures in last 5 calls
        WithSuccessThreshold(2).         // close after 2 successes in half-open state
        WithDelay(30 * time.Second).     // stay open for 30s
        Build()

    retry := retrypolicy.Builder[*tenantResponse]().
        HandleErrors(ErrTransient).
        WithBackoff(100*time.Millisecond, 2*time.Second).
        WithJitter(50*time.Millisecond).
        WithMaxRetries(3).
        Build()

    return &HorizonClient{
        http:    &http.Client{Timeout: 10 * time.Second},
        baseURL: cfg.BaseURL,
        cb:      cb,
        retry:   retry,
    }
}

func (c *HorizonClient) IsTenantAdmin(ctx context.Context, token, tenantRNS string) (bool, error) {
    result, err := failsafe.NewExecutor[*tenantResponse](c.retry, c.cb).
        WithContext(ctx).
        Get(func() (*tenantResponse, error) {
            return c.fetchTenant(ctx, token, tenantRNS)
        })
    if err != nil {
        if circuitbreaker.IsOpen(err) {
            return false, domain.ErrInternal(fmt.Errorf("horizon API circuit open: %w", err))
        }
        return false, domain.ErrInternal(fmt.Errorf("horizon API call failed: %w", err))
    }
    return result.isAdmin(token), nil
}
```

Resilience4j users will feel at home. The composable builder pattern is identical.

---

## 7. Health Checks: Liveness and Readiness

### K8s Probe Semantics

- **Liveness** (`/health/live`): is the process alive? Should almost never fail.
  If it fails, K8s kills and restarts the pod. Only check things that indicate
  the process is broken beyond recovery — e.g., deadlock, OOM.
  Correct answer for UCP: always return 200 unless the process is truly stuck.

- **Readiness** (`/health/ready`): can this pod serve traffic?
  If it fails, K8s removes the pod from the service load balancer.
  Check: can we reach PostgreSQL? Can we reach Temporal? Do we have valid K8s connectivity?
  These checks should be fast and cached.

- **Startup** (`/health/startup`): is the initial startup complete?
  Relevant if your service has a slow startup (migrations, cache warm-up).

### Implementation

The POC already has a health prober package with the right idea. What's missing is:
1. Separate liveness from readiness endpoints
2. Per-check timeouts
3. Result caching (K8s probes every 10s — don't hit Postgres on every probe)

```go
// internal/health/health.go

type Status string
const (
    StatusOK       Status = "ok"
    StatusDegraded Status = "degraded"
    StatusDown     Status = "down"
)

type CheckResult struct {
    Status  Status        `json:"status"`
    Message string        `json:"message,omitempty"`
    Latency time.Duration `json:"latencyMs"`
}

type Check struct {
    Name    string
    Timeout time.Duration
    Run     func(ctx context.Context) CheckResult
}

type HealthService struct {
    livenessChecks  []Check
    readinessChecks []Check
    cache           *resultCache  // TTL-based cache, avoids hammering deps
}

// GET /health/live — simple, no caching, no external calls
func (h *HealthService) Live(c echo.Context) error {
    return c.JSON(200, map[string]string{"status": "ok"})
}

// GET /health/ready — runs readiness checks (cached for 15s)
func (h *HealthService) Ready(c echo.Context) error {
    results := h.cache.Get()
    if results == nil {
        results = h.runChecks(c.Request().Context(), h.readinessChecks)
        h.cache.Set(results)
    }

    overall := StatusOK
    for _, r := range results {
        if r.Status == StatusDown {
            overall = StatusDown
            break
        }
        if r.Status == StatusDegraded {
            overall = StatusDegraded
        }
    }

    code := http.StatusOK
    if overall == StatusDown {
        code = http.StatusServiceUnavailable
    }
    return c.JSON(code, map[string]interface{}{
        "status":     overall,
        "components": results,
    })
}
```

Readiness check response (mirrors Spring Boot Actuator format):
```json
{
  "status": "ok",
  "components": {
    "database":   { "status": "ok",   "latencyMs": 3 },
    "temporal":   { "status": "ok",   "latencyMs": 8 },
    "kubernetes": { "status": "ok",   "latencyMs": 12 },
    "horizon":    { "status": "degraded", "message": "latency elevated" }
  }
}
```

This is the Spring Boot Actuator `/actuator/health` response structure, reproduced
without the Actuator dependency.

---

## 8. Metrics and Tracing: OpenTelemetry

### Why OTel

Prometheus client directly is what the POC uses. It works, but it couples your
instrumentation to Prometheus. OpenTelemetry is now the industry standard — you
instrument once with OTel, then configure exporters for any backend (Prometheus,
Datadog, Grafana Cloud, etc.).

For UCP: instrument with OTel, export to Prometheus (already deployed) for metrics
and to Grafana Tempo for traces (extends the existing Grafana stack).

### Metrics: What to Instrument

The POC has infrastructure metrics (Crossplane resource counts, dependency health gauges)
but no application metrics. What's missing:

```
http_request_duration_seconds{method, path, status}   — latency histogram (P50/P95/P99)
http_requests_total{method, path, status}              — request counter
workflow_submitted_total{resource_type, provider}      — provisioning submission counter
workflow_active_count{resource_type}                   — active workflow gauge (poll Temporal)
quota_check_total{resource_type, result}               — quota pass/fail counter
external_call_duration_seconds{service, operation}     — Horizon/Omnia/GCP latency
external_call_errors_total{service, operation, code}   — external error counter
```

### Setup

```go
// OTel provider setup (in main.go)
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/prometheus"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
    sdktrace  "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/bridge/opentelemetry-go-contrib/instrumentation/github.com/labstack/echo/otelecho"
)

func setupOTel(ctx context.Context, cfg config.OTelConfig) (func(), error) {
    // Metrics: export to Prometheus
    promExporter, _ := prometheus.New()
    meterProvider := sdkmetric.NewMeterProvider(sdkmetric.WithReader(promExporter))
    otel.SetMeterProvider(meterProvider)

    // Traces: export to Grafana Tempo (or any OTLP endpoint)
    traceExporter, _ := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint(cfg.OTLPEndpoint),
    )
    tracerProvider := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(traceExporter),
        sdktrace.WithSampler(sdktrace.TraceIDRatioBased(cfg.SampleRate)),
    )
    otel.SetTracerProvider(tracerProvider)

    shutdown := func() {
        meterProvider.Shutdown(ctx)
        tracerProvider.Shutdown(ctx)
    }
    return shutdown, nil
}

// Echo: add OTel middleware for automatic HTTP metrics + traces
e.Use(otelecho.Middleware("ucp-api-server"))
// This gives you http_server_request_duration_seconds and http_server_request_total
// automatically for every endpoint, with trace span creation per request.
```

### Application Metrics

```go
// internal/metrics/metrics.go
type Metrics struct {
    WorkflowSubmitted metric.Int64Counter
    WorkflowActive    metric.Int64UpDownCounter
    QuotaChecked      metric.Int64Counter
    ExternalDuration  metric.Float64Histogram
}

func New(meter metric.Meter) (*Metrics, error) {
    submitted, _ := meter.Int64Counter("workflow.submitted",
        metric.WithDescription("Number of workflows submitted"),
    )
    // ... etc
    return &Metrics{WorkflowSubmitted: submitted, ...}, nil
}

// In use case:
func (u *DatabaseUseCase) Create(ctx context.Context, req CreateDatabaseRequest) (*domain.Database, error) {
    // ...
    u.metrics.WorkflowSubmitted.Add(ctx, 1,
        metric.WithAttributes(
            attribute.String("resource_type", "database"),
            attribute.String("provider", req.Provider),
        ),
    )
    return db, nil
}
```

### Tracing

OTel traces propagate across service boundaries via HTTP headers (`traceparent`).
For Temporal, pass the trace context in the workflow input so Temporal UI can link
to the trace:

```go
// gateway/temporal.go
func (g *TemporalGateway) StartWorkflow(ctx context.Context, opts WorkflowOptions, input interface{}) (string, error) {
    // Propagate trace context into workflow via header
    span := trace.SpanFromContext(ctx)
    traceID := span.SpanContext().TraceID().String()

    we, err := g.client.ExecuteWorkflow(ctx, client.StartWorkflowOptions{
        ID:        opts.WorkflowID,
        TaskQueue: opts.TaskQueue,
        Memo: map[string]interface{}{
            "trace_id": traceID,  // visible in Temporal UI
        },
    }, opts.WorkflowFn, input)
    // ...
}
```

### Grafana LGTM Stack

Complete observability with existing Grafana:
- **L** — Loki (log aggregation, slog JSON → Loki)
- **G** — Grafana (existing, dashboards)
- **T** — Tempo (traces from OTel)
- **M** — Mimir or Prometheus (metrics from OTel Prometheus exporter)

All four components are under the Grafana umbrella and integrate natively.
The OTel approach means you're not locked in — any OTLP-compatible backend works.

---

## 9. RESTful API: OpenAPI Spec-First

### The Approach

Write the OpenAPI 3.x spec first. Generate server stubs from it. Implement the stubs.
Never write an endpoint that isn't in the spec.

This is API contract-first development. The spec is the contract between:
- API server (producer) and frontend/CLI (consumers)
- Backend team and frontend team (different repos, independent deploys)

The same approach already exists for `omnia-client` — the Omnia spec generates the Go client.
Apply the same pattern to the UCP API itself.

### Tooling

`oapi-codegen` — already used in the project for omnia-client.

```yaml
# api/oapi-codegen.yaml
package: api
generate:
  echo-server:    true   # generates echo handler interface
  models:         true   # generates request/response types
  strict-server:  true   # enforces that all handlers return typed responses
output: gen/api.gen.go
```

The generated file contains:
1. All request/response types (no hand-writing them)
2. A `StrictServerInterface` — one method per endpoint
3. Echo route registration via `RegisterHandlers`

Your handler file implements the interface:

```go
// gen/api.gen.go (generated — never edit)
type StrictServerInterface interface {
    CreateDatabase(ctx context.Context, request CreateDatabaseRequestObject) (CreateDatabaseResponseObject, error)
    ListDatabases(ctx context.Context, request ListDatabasesRequestObject) (ListDatabasesResponseObject, error)
    GetDatabase(ctx context.Context, request GetDatabaseRequestObject) (GetDatabaseResponseObject, error)
    DeleteDatabase(ctx context.Context, request DeleteDatabaseRequestObject) (DeleteDatabaseResponseObject, error)
    // ... one method per endpoint
}

// api/handler/database.go (hand-written)
var _ api.StrictServerInterface = (*Handlers)(nil)  // compile-time interface check

func (h *Handlers) CreateDatabase(ctx context.Context, req api.CreateDatabaseRequestObject) (api.CreateDatabaseResponseObject, error) {
    db, err := h.databaseUC.Create(ctx, req.Body)
    if err != nil {
        return nil, err  // DomainError → global handler
    }
    return api.CreateDatabase202JSONResponse{
        Code:       "PROVISIONING_STARTED",
        WorkflowId: db.WorkflowID,
        Message:    "database provisioning started",
    }, nil
}
```

The compile-time `var _ api.StrictServerInterface = (*Handlers)(nil)` check means:
if you add an endpoint to the spec but don't implement it, the build fails.
You can't forget an endpoint. This is the equivalent of Spring's `@RestController`
implementing an interface from your API module.

### CI Enforcement

```makefile
# Makefile
generate:
    oapi-codegen --config api/oapi-codegen.yaml api/openapi.yaml

check-generated:
    $(MAKE) generate
    git diff --exit-code gen/  # fails CI if generated code is out of sync with spec
```

Running `make check-generated` in CI ensures the spec and implementation never diverge.

---

## 10. BaseResponse: Consistent API Response Structure

### The Spring Boot BaseResponse Pattern

```java
// Spring equivalent we're replicating
public class BaseResponse<T> {
    private String code;
    private int status;
    private String message;
    private T data;
}
```

### Go Implementation

```go
// internal/api/response.go

// Response wraps a successful response with data.
// T is the resource type (Database, ComputeInstance, etc.)
type Response[T any] struct {
    Code    string `json:"code"`              // "OK"
    Status  int    `json:"status"`            // HTTP status code mirrored in body
    Message string `json:"message,omitempty"` // human-readable, optional
    Data    T      `json:"data"`
}

// ListResponse wraps a list of resources.
type ListResponse[T any] struct {
    Code   string `json:"code"`   // "OK"
    Status int    `json:"status"`
    Items  []T    `json:"items"`
    Total  int    `json:"total"`
}

// AsyncResponse is returned for requests that start a long-running workflow.
// The client should poll GET /{resource}/{name} for completion.
type AsyncResponse struct {
    Code       string `json:"code"`       // "PROVISIONING_STARTED", "DELETION_STARTED"
    Status     int    `json:"status"`     // 202
    WorkflowID string `json:"workflowId"` // for status tracking
    Message    string `json:"message"`
    TraceID    string `json:"traceId,omitempty"`
}

// ErrorResponse is returned on all error paths.
// Populated by the global error handler — never by individual handlers.
type ErrorResponse struct {
    Code    string `json:"code"`    // "RESOURCE_NOT_FOUND", "QUOTA_EXCEEDED", "INTERNAL_ERROR"
    Status  int    `json:"status"`  // HTTP status code mirrored in body
    Message string `json:"message"` // safe for client consumption
    TraceID string `json:"traceId,omitempty"` // correlate with server logs
}

// Helper constructors used in handlers
func OK[T any](data T) Response[T] {
    return Response[T]{Code: "OK", Status: 200, Data: data}
}

func OKList[T any](items []T) ListResponse[T] {
    if items == nil { items = []T{} }
    return ListResponse[T]{Code: "OK", Status: 200, Items: items, Total: len(items)}
}

func Accepted(workflowID, msg string) AsyncResponse {
    return AsyncResponse{Code: "PROVISIONING_STARTED", Status: 202, WorkflowID: workflowID, Message: msg}
}
```

Usage in handlers (after oapi-codegen generates the typed response objects,
these helpers fill them):

```go
// List: GET /api/v1/databases
func (h *Handlers) ListDatabases(ctx context.Context, req api.ListDatabasesRequestObject) (api.ListDatabasesResponseObject, error) {
    dbs, err := h.databaseUC.List(ctx, string(req.Params.TenantId))
    if err != nil {
        return nil, err
    }
    return api.ListDatabases200JSONResponse(api.OKList(dbs)), nil
}

// Create: POST /api/v1/databases → 202 Accepted
func (h *Handlers) CreateDatabase(ctx context.Context, req api.CreateDatabaseRequestObject) (api.CreateDatabaseResponseObject, error) {
    result, err := h.databaseUC.Create(ctx, req.Body)
    if err != nil {
        return nil, err
    }
    return api.CreateDatabase202JSONResponse(api.Accepted(result.WorkflowID, "database provisioning started")), nil
}
```

---

## 11. Putting It All Together: A Request's Full Lifecycle

A `POST /api/v1/databases` request in the production codebase:

```
1. Request arrives at echo router
2. Middleware chain:
   - RequestID middleware: assigns/reads X-Request-ID, injects into context
   - OTel middleware: starts HTTP span, records request attributes
   - Auth middleware: validates session cookie, injects Principal into context
   - Logging middleware: logs request start (method, path, request_id)
3. Handler (DatabaseHandler.CreateDatabase):
   - c.Bind(&req) — echo deserializes JSON body
   - c.Validate(&req) — go-playground/validator checks struct tags
   - h.uc.Create(c.Request().Context(), req) — delegates to use case
4. Use case (DatabaseUseCase.Create):
   - horizon.IsTenantAdmin(ctx, token, req.TenantID) — with retry + circuit breaker
   - quota.CheckQuota(ctx, req.TenantID, "database") — K8s label-filtered list
   - xr := xr.Build(req) — typed XR builder, returns (string, error)
   - temporal.StartWorkflow(ctx, opts, input) — submits to Temporal
   - audit.Log(ctx, "database.create", req.Name, principal) — async fire-and-forget
   - return &domain.Database{WorkflowID: we.GetID()}, nil
5. Handler: returns api.CreateDatabase202JSONResponse(api.Accepted(workflowID, msg))
6. Global handler writes the response (happy path — no error to handle)
7. OTel middleware: records HTTP span end (status 202, duration)
8. Logging middleware: logs request completion (method, path, status, duration, request_id)

Error path (quota exceeded):
4b. quota.CheckQuota returns → use case returns domain.ErrQuotaExceeded(...)
5b. Handler returns the DomainError to echo
6b. echo.HTTPErrorHandler receives the error:
    - errors.As(err, &de) → true
    - Logs: slog.WarnContext(ctx, "quota exceeded", "code", de.Code, "request_id", reqID)
    - Writes: 429 {"code":"QUOTA_EXCEEDED","status":429,"message":"...","traceId":"..."}
```

---

## Summary of Decisions

| Concern | Decision | Rationale |
|---|---|---|
| Architecture | Clean Architecture (handler → usecase → domain → repository) | Right size for UCP. DDD would be overengineering for a provisioning platform. |
| HTTP Framework | echo | `HTTPErrorHandler` maps directly to `@RestControllerAdvice`. c.Bind/c.Validate reduce boilerplate. oapi-codegen echo support. |
| DI | Manual constructor injection in `main.go` | Full control, no magic, compile-time errors. ~10-15 dependencies — no need for a container. Add uber/fx if it grows. |
| Error Handling | `DomainError` + echo `HTTPErrorHandler` | Exact Spring Boot pattern: business exceptions + global handler. Centralized, consistent error format. |
| Validation | go-playground/validator via echo.Validator | Declarative struct tags. Single call to c.Validate. Errors flow through global handler. |
| Logging | log/slog (stdlib) | JSON structured output, level-based, context-aware, zero deps. |
| Fault Tolerance | failsafe-go | Resilience4j equivalent. Circuit breaker + retry + timeout, composable. Applied only to external HTTP calls (Horizon, Omnia, GCP). |
| Health Checks | Custom, two-endpoint (live + ready) | Spring Boot Actuator-style response. Readiness checks cached at 15s TTL. |
| Metrics | OTel → Prometheus exporter | Vendor-neutral instrumentation, feeds existing Prometheus. OTel echo middleware for automatic HTTP metrics. |
| Tracing | OTel → Grafana Tempo | Extends existing Grafana stack. W3C traceparent propagation. Trace ID in every error response. |
| API Contract | OpenAPI spec-first → oapi-codegen | Spec is the contract. Generated stubs prevent drift. Compile-time interface check enforces implementation completeness. |
| Response Format | Generic `Response[T]` / `ListResponse[T]` / `AsyncResponse` / `ErrorResponse` | One format per scenario. Clients never see per-resource envelopes. TraceID in every error. |
