---
title: "HTTP Framework: chi vs echo"
space: UCP
parent_page_id: "../implementation.md"
---

# chi vs echo: In-Depth Comparison for UCP

Constraints fixed before this comparison starts:
- Spec-first is non-negotiable (OpenAPI → oapi-codegen strict mode)
- Team has Spring Boot background, not Go-native
- UCP is not high-throughput — framework performance is irrelevant
- AI writes most of the code; the framework should make AI output consistent and correct

---

## The Single Most Important Fact

With **oapi-codegen strict mode**, both frameworks become nearly identical at the
handler implementation layer. The generated wrapper handles request parsing, response
marshaling, and (with the right plugin) request validation. Your handler implements
a typed interface and returns typed objects.

This means echo's most-cited advantages — `c.Bind()`, `c.JSON()`, `c.Validate()` —
**do not apply in spec-first strict mode**. They're handled by generated code.

What remains different: middleware ecosystem, error handling hook, context model,
and testing ergonomics.

---

## 1. What oapi-codegen Strict Mode Generates

Given this spec fragment:

```yaml
# openapi.yaml
paths:
  /api/v1/databases:
    post:
      operationId: CreateDatabase
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateDatabaseRequest'
      responses:
        '202':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AsyncResponse'
        '400':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
```

### What chi strict generates:

```go
// gen/api.gen.go — generated, never edit

type StrictServerInterface interface {
    CreateDatabase(ctx context.Context, request CreateDatabaseRequestObject) (CreateDatabaseResponseObject, error)
    ListDatabases(ctx context.Context, request ListDatabasesRequestObject) (ListDatabasesResponseObject, error)
    // ... one method per operationId
}

// The generated wrapper registers chi routes and calls your interface:
func HandlerFromMux(si StrictServerInterface, r chi.Router) {
    wrapper := ServerInterfaceWrapper{Handler: si}
    r.Post("/api/v1/databases", wrapper.CreateDatabase)
    // ...
}

// Request and response types are generated from the spec:
type CreateDatabaseRequestObject struct {
    Body *CreateDatabaseRequest
}

type CreateDatabaseResponseObject interface {
    VisitCreateDatabaseResponse(w http.ResponseWriter) error
}

type CreateDatabase202JSONResponse AsyncResponse      // 202 response type
type CreateDatabase400JSONResponse ErrorResponse      // 400 response type
type CreateDatabase429JSONResponse ErrorResponse      // 429 response type
```

### What echo strict generates:

```go
// gen/api.gen.go — generated, never edit

type StrictServerInterface interface {
    CreateDatabase(ctx context.Context, request CreateDatabaseRequestObject) (CreateDatabaseResponseObject, error)
    // Same interface — identical
}

func RegisterHandlers(router EchoRouter, si StrictServerInterface) {
    wrapper := ServerInterfaceWrapper{Handler: si}
    router.POST("/api/v1/databases", wrapper.CreateDatabase)
}
```

**The interface is identical.** The only generated difference is that the chi version
uses `chi.Router` and the echo version uses `echo.Echo`. Your handler code is the same
either way:

```go
// api/handler/database.go — YOUR code, framework-agnostic because of strict mode

var _ api.StrictServerInterface = (*Handlers)(nil)

func (h *Handlers) CreateDatabase(ctx context.Context, req api.CreateDatabaseRequestObject) (api.CreateDatabaseResponseObject, error) {
    result, err := h.databaseUC.Create(ctx, req.Body)
    if err != nil {
        return nil, err  // error goes to global handler
    }
    return api.CreateDatabase202JSONResponse{
        Code:       "PROVISIONING_STARTED",
        WorkflowId: result.WorkflowID,
        Message:    "database provisioning started",
    }, nil
}
```

This handler compiles against chi. It also compiles against echo. The framework
choice does not affect this code at all.

---

## 2. Error Handling: The Real Differentiator

This is where chi and echo genuinely differ, and it matters for UCP.

### chi: You Build the Global Handler

chi has no built-in global error handler. Handlers return nothing — they write
to `http.ResponseWriter` directly. To get centralized error handling you either:

**Option A: Middleware that recovers from panics (not clean)**

```go
// Not recommended — panics are for truly exceptional situations
r.Use(func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                respondError(w, 500, "internal error")
            }
        }()
        next.ServeHTTP(w, r)
    })
})
```

**Option B: Wrap handlers to return errors (recommended)**

```go
// Define your own handler type
type Handler func(w http.ResponseWriter, r *http.Request) error

// Adapter that converts Handler to http.HandlerFunc
func H(h Handler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if err := h(w, r); err != nil {
            handleError(w, r, err)  // centralized error → response mapping
        }
    }
}

// Register:
r.Post("/api/v1/databases", H(handler.CreateDatabase))
```

But here's the problem: **oapi-codegen strict mode generates the route registration
for you**. The generated `HandlerFromMux` calls standard `http.HandlerFunc`, not your
custom `Handler` type. You can't inject the error-returning wrapper into the generated
routing code without forking the generated output or writing a post-processing step.

You end up with either:
- The error wrapper in the strict mode wrapper (requires customizing oapi-codegen templates)
- Per-handler error checking inside the generated interface methods
- The panic/recover approach

None of these are clean.

### echo: First-Class Global Error Handler

```go
// api/server.go — one place, handles all errors

e.HTTPErrorHandler = func(err error, c echo.Context) {
    if c.Response().Committed {
        return
    }

    reqID := c.Response().Header().Get("X-Request-ID")

    var de *domain.DomainError
    if errors.As(err, &de) {
        if de.Cause != nil {
            slog.ErrorContext(c.Request().Context(), "domain error",
                "code", de.Code,
                "cause", de.Cause,
                "request_id", reqID,
            )
        }
        _ = c.JSON(de.Status, api.ErrorResponse{
            Code:    de.Code,
            Status:  de.Status,
            Message: de.Message,
            TraceId: reqID,
        })
        return
    }

    slog.ErrorContext(c.Request().Context(), "unhandled error",
        "error", err,
        "request_id", reqID,
    )
    _ = c.JSON(500, api.ErrorResponse{
        Code:    "INTERNAL_ERROR",
        Status:  500,
        Message: "an internal error occurred",
        TraceId: reqID,
    })
}
```

This wires into the oapi-codegen strict mode wrapper cleanly because the strict
wrapper calls the echo error handler when your interface method returns an error:

```go
// Inside oapi-codegen generated strict wrapper (echo variant):
func (w *strictHandler) CreateDatabase(ctx echo.Context) error {
    // ... parse request ...
    response, err := w.ssi.CreateDatabase(ctx.Request().Context(), request)
    if err != nil {
        return err  // echo routes this to e.HTTPErrorHandler
    }
    return response.VisitCreateDatabaseResponse(ctx.Response())
}
```

Your handler returns `nil, domain.ErrQuotaExceeded(...)` → strict wrapper returns
it to echo → echo calls `HTTPErrorHandler` → consistent JSON error response.

**Verdict: echo wins on error handling, especially in spec-first strict mode.**
The `HTTPErrorHandler` hook is exactly the `@RestControllerAdvice` pattern, and it
integrates with oapi-codegen's generated code without any workaround.

---

## 3. Middleware: Compared Side by Side

Both have middleware for every common concern. The implementations are equivalent.

| Concern | chi | echo |
|---|---|---|
| Request ID | `middleware.RequestID` | `middleware.RequestID` |
| Structured logging | `middleware.Logger` (basic), or write your own slog middleware | `middleware.Logger` (basic), or write your own slog middleware |
| Panic recovery | `middleware.Recoverer` | `middleware.Recover` |
| CORS | `middleware.CORS` (via `rs/cors`) | `middleware.CORSWithConfig` (built-in) |
| Rate limiting | `go.uber.org/ratelimit` or `jub0bs/fcors` | `middleware.RateLimiterWithConfig` (built-in) |
| Gzip compression | `middleware.Compress` | `middleware.Gzip` |
| Timeout | `middleware.Timeout` | `middleware.TimeoutWithConfig` |
| Body size limit | `middleware.RequestSize` | `middleware.BodyLimitWithConfig` |

chi's middleware uses `func(http.Handler) http.Handler` — pure stdlib. Any middleware
written for Go's stdlib works with chi without adaptation.

echo's middleware uses `func(echo.HandlerFunc) echo.HandlerFunc` — echo-specific.
Third-party middleware written for stdlib needs a wrapper. This is a minor friction
point but not a showstopper; the echo community is large enough that most common
middleware has an echo variant.

**Verdict: tie in capability. chi has a slight edge in ecosystem compatibility
(stdlib-compatible middleware runs everywhere). echo has a slight edge in built-in
options (rate limiter, body limit are built-in vs external dependency).**

---

## 4. Context Model

### chi: stdlib `context.Context`

chi passes values through stdlib `context.Context` using typed keys:

```go
// chi puts URL path params into the request context:
func (h *Handlers) GetDatabase(w http.ResponseWriter, r *http.Request) {
    name := chi.URLParam(r, "name")  // reads from r.Context()
    // ...
}

// Middleware injects values the same way:
type ctxKey struct{}
ctx := context.WithValue(r.Context(), ctxKey{}, principal)
r = r.WithContext(ctx)

// Handler reads:
principal, _ := r.Context().Value(ctxKey{}).(*domain.Principal)
```

### echo: `echo.Context` wrapping stdlib

echo wraps the request in `echo.Context`, which has convenience methods:

```go
func (h *Handlers) GetDatabase(c echo.Context) error {
    name := c.Param("name")          // path param, cleaner than chi.URLParam
    principal := c.Get("principal")  // echo's key-value store on context
    // ...
}
```

The echo.Context convenience is real — `c.Param()` is cleaner than `chi.URLParam(r, "name")`.
But in strict mode this doesn't matter — the generated code extracts path params
before calling your interface method, which receives a typed request object:

```go
// Your handler in strict mode — no framework context at all:
func (h *Handlers) GetDatabase(ctx context.Context, req api.GetDatabaseRequestObject) (api.GetDatabaseResponseObject, error) {
    name := req.DatabaseName  // generated from the spec path parameter
    // No chi.URLParam, no c.Param — the generated wrapper handled it
    db, err := h.uc.Get(ctx, name)
    // ...
}
```

**Verdict: in spec-first strict mode, context differences are irrelevant. The generated
wrapper insulates your handler from framework-specific context access entirely.**

In middleware (which you still write manually), chi's stdlib context is slightly
more portable. Echo's `c.Get/Set` is slightly more convenient but creates an
implicit key-value store that's harder to type-check.

---

## 5. Testing Ergonomics

### chi handlers with httptest

chi handlers are plain `http.Handler`. You test them exactly as you would test any
Go HTTP handler:

```go
func TestListDatabases_FiltersByTenant(t *testing.T) {
    handler := NewHandlers(fakeDatabaseUC{...}, ...)

    req := httptest.NewRequest("GET", "/api/v1/databases?tenantId=rns:roc:iam::clsd-ucp", nil)
    w := httptest.NewRecorder()

    // In strict mode, the generated router wraps the handler — test via router:
    r := chi.NewRouter()
    api.HandlerFromMux(handler, r)
    r.ServeHTTP(w, req)

    assert.Equal(t, 200, w.Code)
    var resp api.ListDatabasesResponse
    json.Unmarshal(w.Body.Bytes(), &resp)
    assert.Len(t, resp.Items, 2)
}
```

### echo handlers with httptest

echo handlers require an echo instance. Setting up the same test:

```go
func TestListDatabases_FiltersByTenant(t *testing.T) {
    handler := NewHandlers(fakeDatabaseUC{...}, ...)

    e := echo.New()
    api.RegisterHandlers(e, handler)
    // Important: must also register the error handler:
    e.HTTPErrorHandler = customErrorHandler

    req := httptest.NewRequest("GET", "/api/v1/databases?tenantId=rns:roc:iam::clsd-ucp", nil)
    w := httptest.NewRecorder()
    e.ServeHTTP(w, req)  // echo implements http.Handler

    assert.Equal(t, 200, w.Code)
    var resp api.ListDatabasesResponse
    json.Unmarshal(w.Body.Bytes(), &resp)
    assert.Len(t, resp.Items, 2)
}
```

Testing error handling is where echo shows an advantage:

```go
// With echo: error handler is testable end-to-end in the same test
func TestCreateDatabase_QuotaExceeded_Returns429(t *testing.T) {
    e := echo.New()
    e.HTTPErrorHandler = customErrorHandler  // the real error handler
    api.RegisterHandlers(e, NewHandlers(fakeDatabaseUC{
        createErr: domain.ErrQuotaExceeded("database", "rns:..."),
    }))

    req := httptest.NewRequest("POST", "/api/v1/databases", body)
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    e.ServeHTTP(w, req)

    assert.Equal(t, 429, w.Code)
    var resp api.ErrorResponse
    json.Unmarshal(w.Body.Bytes(), &resp)
    assert.Equal(t, "QUOTA_EXCEEDED", resp.Code)
}
```

With chi, you'd test the error handler separately from the handler (two tests instead
of one end-to-end test). Not a dealbreaker, but more setup.

**Verdict: comparable for happy-path tests. Echo has a marginal advantage for
error-path tests because the error handler wires in cleanly. With chi, you need
more ceremony to test the full request → error → response pipeline.**

---

## 6. AI Code Generation Consistency

The user's point is correct and worth making precise: **spec-first means the spec
is the ground truth that AI must conform to, not the code**.

With strict mode oapi-codegen, the generated interface is the contract. AI writes
implementations of that interface. The framework doesn't change what the AI writes
at the handler level — it writes implementations of the same typed interface either way.

Where framework choice affects AI output is in:

1. **Middleware code** — AI knows both patterns well. For chi, it writes
   `func(http.Handler) http.Handler`. For echo, it writes
   `func(echo.HandlerFunc) echo.HandlerFunc`. Neither is harder for AI.

2. **Error handling** — This is where echo wins for AI consistency.
   The echo pattern is `return domain.ErrXxx(...)` from any handler, anywhere,
   and it's handled globally. The chi pattern requires either a panic (wrong)
   or a custom wrapper that the AI must be told about explicitly.
   If AI generates a handler that returns an error value in chi, the code
   won't compile (chi handlers have no return value). Echo handlers return `error`
   natively. **This is a real advantage for AI-written code: echo's handler
   signature `func(echo.Context) error` is natural for AI to produce correct
   error propagation.**

3. **Generated stubs** — oapi-codegen strict mode generates `context.Context` +
   typed request → typed response for both. AI writes the same thing either way.

**Verdict: echo is marginally better for AI-written code because the error
propagation model (`return err`) is natural, whereas chi requires explaining
the wrapper pattern to AI every time.**

---

## 7. Ecosystem and Maintenance

| Dimension | chi | echo |
|---|---|---|
| GitHub stars | ~18k | ~29k |
| Latest release cadence | Active (v5 in progress) | Active |
| Major version stability | v5 is breaking (v4 is stable) | v4 is stable |
| oapi-codegen support | ✅ official target | ✅ official target |
| Internal Rakuten use | GADP, Rakuten Energy (docs) | MNO investigation (recommendation only) |
| Spring Boot mental model | Weaker mapping | Stronger mapping |

Note: chi v5 is a breaking change from v4 (drops stdlib compatibility for some features).
Use v4 for production. oapi-codegen targets chi v5 in newer versions — verify compatibility.

---

## 8. The Specific Case for UCP

UCP characteristics that inform the choice:

- **Team has Spring Boot background** → echo's `HTTPErrorHandler` = `@RestControllerAdvice`.
  Concepts map directly. No need to explain "you wrap all handlers with an adapter function."
- **Spec-first strict mode** → handler signatures are identical between frameworks.
  The framework becomes a thin routing and middleware layer.
- **AI writes most code** → echo's `return err` error model is natural for AI.
  Chi's errorless handler signature requires a wrapper that AI must know about.
- **BFF auth, metrics, rate limiting, CORS, request ID** → both have these. Echo
  has them more built-in; chi needs some external packages but they're well-known.
- **UCP is not high-throughput** → performance is a non-argument.
- **The existing POC uses gorilla/mux** → migration to either is similar effort.
  Neither gorilla handler code is reusable; it's a rewrite either way.

---

## 9. Side-by-Side: The Same API Server in Both

To make this concrete, here's the wiring code for the same server in both frameworks.

### chi version

```go
// cmd/server/main.go

r := chi.NewRouter()

// Middleware
r.Use(middleware.RequestID)
r.Use(middleware.RealIP)
r.Use(slogMiddleware(logger))     // custom — 10 lines
r.Use(middleware.Recoverer)
r.Use(cors.Handler(cors.Options{...}))

// Auth middleware on the API subrouter
apiRouter := chi.NewRouter()
apiRouter.Use(authMiddleware(bffAuth))
r.Mount("/api/v1", apiRouter)

// Register generated routes
api.HandlerFromMux(handlers, apiRouter)

// Health (no auth)
r.Get("/health/live", health.Live)
r.Get("/health/ready", health.Ready)
r.Handle("/metrics", promhttp.Handler())

// Centralized error handling — THE PROBLEM:
// chi has no HTTPErrorHandler hook. You need a custom wrapper:
//
// Option 1: wrap every handler registration (doesn't work with generated HandlerFromMux)
// Option 2: write a custom oapi-codegen template that injects error handling
// Option 3: put error handling in middleware via panic/recover (wrong approach)
// Option 4: use strict mode's error return but write a custom strict handler wrapper

// This is the chi error handling gap — it requires solving before you can use it.
```

### echo version

```go
// cmd/server/main.go

e := echo.New()
e.HideBanner = true

// Global error handler — THE RESTCONTROLLERADVICE EQUIVALENT
e.HTTPErrorHandler = errorHandler(logger)

// Middleware
e.Use(echomiddleware.RequestID())
e.Use(echomiddleware.RealIP())
e.Use(slogMiddleware(logger))     // custom — 10 lines
e.Use(echomiddleware.Recover())
e.Use(echomiddleware.CORSWithConfig(echomiddleware.CORSConfig{...}))

// Auth on the API group
apiGroup := e.Group("/api/v1", authMiddleware(bffAuth))

// Register generated routes
api.RegisterHandlers(apiGroup, handlers)

// Health (no auth, root level)
e.GET("/health/live", health.Live)
e.GET("/health/ready", health.Ready)
e.GET("/metrics", echo.WrapHandler(promhttp.Handler()))

// Done. Error handling just works.
```

The echo version has no unsolved problems. The chi version requires solving
the error handling gap, which requires either forking oapi-codegen templates
or accepting a suboptimal pattern.

---

## 10. Verdict

| Criterion | chi | echo | Winner |
|---|---|---|---|
| Spec-first + strict mode handler code | Identical | Identical | Tie |
| Centralized error handling with oapi-codegen | Requires workaround | First-class hook | Echo |
| Spring Boot mental model | Weaker | Stronger | Echo |
| AI error propagation naturalness | Errorless handlers (needs wrapper) | `return err` native | Echo |
| Middleware ecosystem | stdlib-compatible | Slightly more built-in | Slight chi |
| Testing (error paths) | Two tests needed | One end-to-end test | Slight echo |
| Internal Rakuten adoption | GADP, Energy (actual use) | MNO (recommendation only) | Slight chi |
| stdlib lock-in risk | None | Mild echo.Context coupling | Slight chi |
| Framework complexity | Minimal | Moderate | chi |

**Recommendation: echo for UCP.**

The decisive factors are:
1. `HTTPErrorHandler` solves the centralized error handling requirement without any
   workaround, and it integrates cleanly with oapi-codegen strict mode.
2. The Spring Boot team knows `@RestControllerAdvice`. The echo mental model is a
   direct transfer. No new concept to explain.
3. `return err` from handlers is natural for AI-written code. Explaining the chi
   error-wrapper pattern to AI every session is friction.

The counter-argument — that chi is more idiomatic Go and more Rakuten-internally-used
— is valid but secondary given the team's background and the spec-first constraint.
chi shines when you're writing handlers by hand with Go instincts. In spec-first
strict mode with AI-written implementations, the framework difference largely disappears,
and echo's error handling hook is the remaining decisive factor.

**Caveat:** if the team has a Go-native engineer who will own the codebase long-term,
reconsider chi. The stdlib compatibility, lack of framework lock-in, and Rakuten
internal usage are real advantages for someone who knows Go well. For a team that
knows Spring Boot and is using AI to write Go, echo's guardrails are worth more.
