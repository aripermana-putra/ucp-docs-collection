# Research — Logging Policy and Audit Logging

**Jira:** [MCUCP-256](https://jira.rakuten-it.com/jira/browse/MCUCP-256) · [MCUCP-257](https://jira.rakuten-it.com/jira/browse/MCUCP-257)
**Date:** 2026-08-21

---

## Summary

Two related but distinct logging concerns need RFC-level definition before more features are built:

1. **Application logging policy (MCUCP-256)** — what log level is appropriate for each category of event, what standard fields every log line must carry, and what must never be logged.
2. **Audit logging scope (MCUCP-257)** — which user-visible actions across every UCP feature produce an audit log entry, and what the entry must contain.

The infrastructure decisions (tool: `log/slog`, format: JSON to stdout, transport: EaaS/Filebeat; storage: PostgreSQL `audit_logs` table) are already settled in ADR-007 and implemented in code. This research defines the **policy and coverage** on top of that infrastructure.

---

## Problem

### Application logging (MCUCP-256)

ADR-007 says "level-based structured logging with JSON output to stdout" but does not define what each level means in UCP's context. The current codebase already uses `slog` but without a written policy, level assignment is ad-hoc. Examples visible today:

- Audit write failures → `slog.ErrorContext`
- Keycloak logout revocation failure → `slog.WarnContext`
- Soft-delete race condition during WIF verify → `slog.WarnContext`
- HTTP 4xx responses → `Warn` via the global error handler
- HTTP 5xx responses → `Error` via the global error handler
- Server startup/shutdown → `slog.Info` (no context)
- Missing env var → `slog.Error` (then `os.Exit(1)`)

No written definition exists for:
- What belongs at `Debug`
- The distinction between `Warn` and `Error`
- Standard fields required on every line
- What must never appear in logs (credentials, PII)

Go's `slog` has no `Fatal` or `Panic` level — process termination uses `slog.Error` followed by `os.Exit(1)`.

### Audit logging (MCUCP-257)

The `audit_logs` table and `AuditService` exist and are used, but coverage is defined piecemeal per TRD. Currently implemented:

| Action | Written by | When |
|---|---|---|
| `auth.login` | JWT middleware | First API call of a new Keycloak session |
| `auth.logout` | Logout handler | On `POST /auth/logout` |
| `authorization_failed` | Permission middleware | On 403 |
| `gcp_project_registered` | GCP service | On successful registration |
| `gcp_credential_updated` | GCP service | On successful SA email update |

Not yet defined: which future operations need audit entries, what constitutes a complete audit event, and how the audit log is exposed to users (PRD-003 grants `viewer` role read access to audit logs, but no `GET /audit-logs` endpoint exists).

---

## Why it matters

**Logging policy** is a precondition for operational reliability. Without level definitions, on-call engineers cannot set alert thresholds (alert on `Error`, ignore `Info`), and log noise degrades the usefulness of the EaaS dashboard.

**Audit coverage** is a security and compliance requirement. Missing entries create gaps in the audit trail that are hard to backfill. Each new feature adds new operations — without a scope definition, coverage remains reactive rather than systematic.

---

## Findings

### Application logging: current field usage

The global error handler (`httpserver/error_handler.go`) already establishes a pattern for request logs:

```
method, uri, status, code, latency_ms, request_id, [cause if 5xx]
```

The request logger middleware adds:
```
method, uri, status, latency_ms, request_id
```

Service-layer log calls (e.g. audit write failures) currently use:
```
action, user_id, [tenant_rns], error
```

No uniform standard is documented.

### Audit log schema adequacy

The `audit_logs` table has `resource_type TEXT` and `resource_id TEXT` columns that are currently unpopulated. They are intended for resource-scoped operations. The `metadata JSONB` column carries arbitrary key-value pairs — no schema is enforced on it; each write defines its own shape. Both need a documented convention so implementations stay consistent.

RGR MON-03 requires that event logs include the **outcome of the event (success or failure)**. The current `audit_logs` table has no `result` field — a migration is required. Additionally, all current audit writes record only successful operations; security-significant failures (e.g. WIF verification failure) produce no audit entry today, which does not satisfy MON-03.

---

## Log format and error logging — deep dive

This section examines four questions that determine the standard format for every log line in UCP. Each is analysed from multiple angles before a recommendation is made.

---

### Q1 — Field naming convention: `snake_case` vs `camelCase` vs `dot.notation`

**The current state:** every field in the codebase today is `snake_case` — `user_id`, `request_id`, `gcp_project_id`, `latency_ms`, `tenant_rns`. This emerged organically from Go conventions and PostgreSQL column names.

**Option A — `snake_case` (keep current)**

- Consistent with the existing codebase without any migration
- Matches PostgreSQL column names (`audit_logs.user_id`, `audit_logs.tenant_rns`) — the same field name in the log and in the DB, making cross-referencing trivial
- Consistent with the OpenAPI response fields which use `camelCase` in JSON but `snake_case` in Go struct tags — aligning logs with DB rather than API feels more natural for operational queries
- Standard in Python/Go ecosystems. Elasticsearch maps `snake_case` naturally with no special config

**Option B — `camelCase`**

- More natural for JSON-native tooling and JavaScript consumers of log data
- Inconsistent with the existing codebase — would require a breaking change to all current log calls
- Inconsistent with DB column names — `userId` vs `user_id` adds cognitive overhead when correlating log fields against DB queries
- No meaningful benefit for UCP's stack (EaaS/Elasticsearch handles both equally)

**Option C — `dot.notation` (Elastic Common Schema — ECS)**

ECS is Elastic's standardised field naming scheme: `http.request.method`, `error.message`, `user.id`, `event.action`. Used by Elastic APM and Filebeat-native integrations.

- Pros: standardised across the Elastic ecosystem — dashboards, alerts, and correlations work across all ECS-compliant services without remapping. If Rakuten's EaaS ever ships pre-built UCP dashboards or cross-service correlation, ECS compliance means zero remapping.
- Cons: significant deviation from the current codebase. Every field must be renamed. `user_id` → `user.id`, `request_id` → `trace.id` or `event.id`, `gcp_project_id` → a custom `ucp.gcp.project_id`. The nesting adds ceremony at call sites: `"http.method", method` instead of `"method", method`. ECS is most valuable when you're ingesting many services into one Elasticsearch cluster and need cross-service dashboards — at UCP's current scale (one service family), the benefit is small.
- There's also a subtlety: ECS `dot.notation` in JSON logs gets parsed by Elasticsearch as nested objects (`{"http": {"method": "GET"}}`), while `snake_case` fields are flat. Nested objects have slightly different query syntax in Kibana.

**Recommendation: Option A — `snake_case`.**

The codebase is already there. The consistency with DB column names is a genuine operational benefit — when debugging a GCP registration failure, you query `audit_logs` with `WHERE gcp_project_id = 'x'` and you search Elasticsearch with `gcp_project_id: "x"`. Zero translation.

ECS is worth reconsidering if Rakuten's EaaS team provides pre-built dashboards that require it, but that's a future migration decision, not a day-one requirement.

---

### Q2 — Error field shape: `"error", err` vs `err.Error()` vs structured

This is the most nuanced question because of how `DomainError` interacts with `slog`.

**How `slog` renders errors today:**

When you write `slog.ErrorContext(ctx, "something failed", "error", err)`, slog calls `err.Error()` on the value and stores the result as a string in the `error` JSON field. For a wrapped error:

```go
err := fmt.Errorf("outer context: %w", pgErr)
// JSON output: {"error": "outer context: pq: connection refused"}
```

The full chain appears as one concatenated string. This is the Go idiomatic default.

**The `DomainError` trap:**

`DomainError.Error()` returns `Message` — the client-safe string, not the internal cause:

```go
de := domain.ErrInternal(pgErr)
// de.Error() = "An internal error occurred."   ← client-safe
// de.Cause() = pgErr                           ← actual root cause
```

If service code accidentally receives and logs a `DomainError`:
```go
slog.ErrorContext(ctx, "something failed", "error", de)
// JSON: {"error": "An internal error occurred."}  ← useless for debugging
```

The actual DB error is swallowed. The current error handler correctly avoids this by using `de.Cause()` explicitly. But there's no guardrail preventing a developer from logging a `DomainError` at the service layer.

**Option A — `"error", err` (current, keep)**

- Idiomatic `slog` — one field, one string, full chain
- Works correctly for all non-DomainError errors (DB errors, network errors, Go stdlib errors)
- Risk: if a `DomainError` flows into a `"error", err` call at the service layer, the root cause is invisible
- Mitigation: document the rule — **service code never receives DomainErrors** (DomainError wrapping happens at the handler layer); service code always logs raw errors

**Option B — `"error", err.Error()` (explicit string conversion)**

- Identical output to Option A for most errors
- Slightly more explicit about intent — "I want the string"
- No practical advantage; actually slightly worse because it discards the `slog.LogValuer` interface that custom error types can implement for richer output
- Not recommended

**Option C — separate `"error_type"` and `"error_message"` fields**

```go
slog.ErrorContext(ctx, "db failed",
    "error_type", fmt.Sprintf("%T", err),  // e.g. "*pgconn.PgError"
    "error_msg", err.Error(),
)
```

- Enables Elasticsearch queries like `error_type: "*pgconn.PgError"` — useful for alerting on specific error classes
- More verbose at call sites
- Overkill for UCP's current scale — most errors are `pgx` DB errors or network errors and the type name adds little actionable information that the message doesn't already carry
- Could be worth it once provisioning is in place and you want alerts on specific Temporal workflow error types

**Option D — custom `slog.LogValuer` on `DomainError`**

Go's `slog` supports a `LogValue() slog.Value` method. `DomainError` could implement it to output both the message and the cause as separate sub-fields:

```go
func (e *DomainError) LogValue() slog.Value {
    attrs := []slog.Attr{
        slog.String("code", e.Code),
        slog.String("message", e.Message),
    }
    if e.cause != nil {
        attrs = append(attrs, slog.String("cause", e.cause.Error()))
    }
    return slog.GroupValue(attrs...)
}
```

Output: `{"error": {"code": "INTERNAL_ERROR", "message": "An internal error occurred.", "cause": "pq: connection refused"}}`

- Pros: structured, queryable, makes the DomainError-at-service-layer trap safe — logging a DomainError always shows the cause
- Cons: changes the shape of the `error` field from a string to an object — breaks any Elasticsearch dashboards or alert rules that treat `error` as a flat string. Also means `"error", err` behaves differently depending on whether `err` is a `DomainError` or a raw error — inconsistent output for the same field name.

**Recommendation: Option A — `"error", err` — with an explicit rule.**

The rule: **at the service layer, always log raw errors** (`"error", dbErr`, `"error", networkErr`). DomainError wrapping is a handler concern. Service code never receives DomainErrors — it produces them. This means `"error", err` at the service layer is always safe. At the handler/middleware layer, where DomainErrors do exist, use `"error", de.Cause()` to log the root cause (as the error handler already does). Document this split explicitly in the RFC.

---

### Q3 — Contextual mandatory fields: what is required in which context

Three distinct call contexts exist in UCP:

**Context 1: HTTP request path (has `request_id`, may have `user_id`)**

These are log lines written inside a handler, middleware, or service called from a request. The request context carries `request_id` (injected by RequestID middleware) and `Principal` (injected by JWT middleware for authenticated endpoints).

Mandatory fields:
- `request_id` — always, via `slog.XxxContext(ctx, ...)` which picks it up automatically if the slog handler is wired to extract it, OR by passing it explicitly as `"request_id", middleware.RequestIDFromContext(ctx)`
- `user_id` — on any line where the principal is known (authenticated endpoints)

> **Note on automatic vs explicit `request_id`:** `slog`'s standard `JSONHandler` does NOT automatically extract `request_id` from context — it must be passed explicitly. The alternative is a custom `slog.Handler` wrapper that injects context values automatically — see Option B and its implementation details below.

**Context 2: Service layer — background or async operations (no HTTP context)**

Temporal activities, health checks, drift detection — no `request_id`. Have operation-specific identifiers instead.

Mandatory fields:
- Operation-specific identifier (`workflow_id`, `gcp_project_id`, etc.) — at least one that ties the log line to a traceable unit of work
- `user_id` if the operation was triggered by a user action (e.g. a Temporal activity knows the `user_id` from the workflow input)

**Context 3: Process lifecycle (startup, shutdown, fatal)**

No request context, no user.

Mandatory fields: none beyond the slog built-ins (`level`, `time`, `msg`). These lines are rare and self-explanatory.

**Option A — Explicit field passing everywhere (current approach)**

```go
slog.ErrorContext(ctx, "failed to update verification status",
    "error", updateErr,
    "gcp_project_id", gcpProjectID)
```

`request_id` must be explicitly passed every time it's needed, or injected into the context using `slog.InfoContext(ctx, ...)` where `ctx` carries the request ID in the context value (but as noted, the standard handler doesn't extract it automatically).

- Pros: no hidden magic, explicit control, no custom handler needed
- Cons: easy to forget `request_id` in a new log call; inconsistent coverage until the RFC enforces it

**Option B — Context-enriched slog handler**

A thin wrapper around `slog.JSONHandler` that automatically injects `request_id` and `user_id` from the context into every log line when they're present:

```go
type contextHandler struct {
    inner slog.Handler
}

func (h *contextHandler) Handle(ctx context.Context, r slog.Record) error {
    if reqID := middleware.RequestIDFromContext(ctx); reqID != "" {
        r.AddAttrs(slog.String("request_id", reqID))
    }
    if p, ok := middleware.PrincipalFromContext(ctx); ok {
        r.AddAttrs(slog.String("user_id", p.UserID.String()))
    }
    return h.inner.Handle(ctx, r)
}
```

- Pros: `request_id` and `user_id` appear in every `slog.XxxContext(ctx, ...)` call automatically — zero chance of forgetting them. Developers just call `slog.ErrorContext(ctx, ...)` and the mandatory fields are there.
- Cons: custom handler adds complexity; needs to be wired in `main.go`; slightly slower (two context lookups per log call). The handler must be careful about nil contexts (startup logs without a request context).
- The context-aware handler is the approach used by structured logging libraries like `log/slog`'s own documentation examples for adding per-request fields.

**Recommendation: Option B — context-enriched handler.**

The operational benefit is significant: `request_id` in every log line means any error in production is immediately correlatable to the HTTP response the user received (which carries the same `request_id`). Option A requires discipline at every call site — a discipline that breaks over time as the codebase grows. Option B makes the correct thing the default.

**How it would be implemented** (not yet in the codebase — currently `main.go` uses the standard `slog.NewJSONHandler` with no context injection):

```go
// shared/middleware/context_log_handler.go — ~20 lines
type contextHandler struct {
    inner slog.Handler
}

func (h *contextHandler) Handle(ctx context.Context, r slog.Record) error {
    if reqID := RequestIDFromContext(ctx); reqID != "" {
        r.AddAttrs(slog.String("request_id", reqID))
    }
    if p, ok := PrincipalFromContext(ctx); ok {
        r.AddAttrs(slog.String("user_id", p.UserID.String()))
    }
    return h.inner.Handle(ctx, r)
}
```

```go
// main.go — one line change
baseHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: level})
slog.SetDefault(slog.New(NewContextHandler(baseHandler)))
```

Every `slog.XxxContext(ctx, ...)` call in the codebase gets `request_id` and `user_id` injected automatically when they're in the context — without touching existing call sites. Calls without a context (`slog.Info(...)`, startup logs) skip the injection cleanly.

---

### Q4 — Sensitive field masking: convention vs enforcement

**What is sensitive in UCP's context:**
- JWT tokens (access_token, refresh_token, id_token)
- Keycloak client secrets
- SA email addresses in application logs (distinct from audit log metadata where they are intentional)
- User email/name beyond `user_id` UUID
- `old_sa_email` / `new_sa_email` in application error logs (these exist in audit metadata intentionally but should not leak into general application logs)

**Current state:** No tokens or emails appear in application logs today. The risk is future accidental logging by a developer during debugging (e.g. `slog.Debug("token exchange", "token", accessToken)`).

**Option A — Convention only (document the rules, rely on code review)**

- Pros: zero implementation cost, no complexity added
- Cons: convention breaks in two ways: (1) a developer logging under time pressure, (2) a future AI-generated code change that logs a token "helpfully". Code review catches these but not always immediately.
- Sufficient at current team size and codebase scale

**Option B — Custom slog handler scrubs known field names**

The context-enriched handler from Q3 can also scrub:

```go
var sensitiveFields = map[string]bool{
    "token": true, "access_token": true, "refresh_token": true,
    "secret": true, "password": true,
}

func (h *contextHandler) Handle(ctx context.Context, r slog.Record) error {
    // scrub known sensitive field names
    r.Attrs(func(a slog.Attr) bool {
        if sensitiveFields[strings.ToLower(a.Key)] {
            // replace value with "[REDACTED]"
        }
        return true
    })
    ...
}
```

- Pros: defence-in-depth — even if a developer logs `"token", accessToken`, it gets redacted before reaching Elasticsearch
- Cons: `slog.Record` fields are immutable — scrubbing requires building a new `Record`, which adds allocation per log call. The handler becomes more complex. False positives (a legitimate field named `"token_count"` being scrubbed) require careful field name matching.
- The `slog.Record` immutability problem is real — the Go team intentionally made records immutable for performance; scrubbing requires creating a new record with filtered attributes.

**Option C — Sensitive fields never passed to slog at all (strictest convention)**

The rule: sensitive values are never assigned to a local variable named anything resembling a log field. If you need to log "something about the token", log a hash or the first/last 4 chars.

- Pros: no runtime cost, no handler complexity
- Cons: purely a culture/review discipline — same as Option A, just more specific

**Recommendation: Option A + one rule from Option C.**

At UCP's current scale, a custom scrubbing handler adds complexity that isn't justified. The rule: **if a sensitive value must appear in a log for debugging, log a non-reversible representation** — token prefix (`token[:8] + "..."`) or a hash. This is documented in the RFC so it's explicit, not just assumed.

Option B is worth reconsidering when the team grows beyond ~5 engineers or when there's evidence of accidental sensitive field logging. The context-enriched handler from Q3 provides the right hook to add scrubbing later with minimal additional change.

---

## What other Rakuten teams do

### Rakuten Group Regulations (RGR) — mandatory compliance baseline

From the Cyber Security Defense Department's "Security Practices for Logging & Audit" ([Confluence](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=4775300409)), the following RGR policies apply to all Rakuten products including UCP:

The following policies are confirmed from the actual RGR document ([002411] Instruction for Continuous Monitoring, downloaded and reviewed):

| Policy | Status | Requirement |
|---|---|---|
| **MON-03** | REQUIRED | Event logs must contain at minimum: (a) system account ID, (b) type of event, (c) date and time, (d) origination/source, (e) **outcome (success or failure)**, (f) individual/entity associated |
| **MON-03(b)** | REQUIRED | System access must be linkable to individual users — audit trails must trace activity back to a specific user |
| **MON-03(e)** | REQUIRED | Limit personal data in audit records to elements identified in the data privacy risk assessment |
| **MON-05** | REQUIRED | When an event log processing failure occurs: (1) immediately alert designated personnel, (2) take reasonable actions to remedy — minimum: overwrite oldest records. If no logs received by centralized collector within 24 hours, alert asset owners |
| **MON-07** | REQUIRED | Use an authoritative time source for event log timestamps |
| **MON-10** | REQUIRED | For critical/sensitive IT assets: minimum 90 days online (readily available) + minimum 1 year total (online, archival, or offline storage) |
| **IAC-01(a)** | REQUIRED | Maintain a history of user access activities for accountability and incident investigation |

**Key finding — MON-05 and write guarantees:** MON-05 requires alerting on audit write failures and taking reasonable remediation action. It does not mandate zero-loss delivery — retrying is "reasonable action." This is the compliance basis for the in-process async retry recommendation.

**Key finding — MON-03 outcome field:** The current `audit_logs` schema is missing a `result` field (success/failure). This is required by MON-03(e) which mandates that event logs capture the outcome of the event.

**PII rule (MON-03(e)):** Personal data in audit records must be limited to what is identified in the privacy risk assessment. Passwords, credentials, and raw identifiers must not appear.

---

### SECaaS (Link Security / RMCPS) — most detailed audit log standard found

From "22. Audit Logs" ([Confluence](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6766875206)), SECaaS explicitly separates three layers:

| Layer | What it covers |
|---|---|
| **Audit Logs** | State-changing operations. Append-only, compliance-oriented |
| **System KPIs / Application Monitoring** | Infrastructure health, API latency, crash rates |
| **Application KPIs / Analytics** | Feature usage, user funnels, business metrics |

Their canonical audit log JSON schema:
```json
{
  "eventId":       "<UUID v4>",
  "timestamp":     "<ISO 8601 UTC>",
  "eventType":     "<EVENT_TYPE>",
  "service":       "USS | PRA",
  "userId":        "<internal UUID> | null",
  "actorId":       "<internal UUID> | null",
  "actorType":     "USER | SYSTEM | BSS | TM_VENDOR",
  "result":        "SUCCESS | FAILURE | PENDING",
  "errorCode":     "<string>",
  "errorMessage":  "<string>",
  "correlationId": "<UUID>",
  "metadata":      { ... }
}
```

Key design decisions:
- `userId` is **nullable** — system-scoped events (scheduled jobs) have no user context
- **Append-only enforced at DB level** — service account has INSERT-only grant; no UPDATE, DELETE, or TRUNCATE
- **`retentionCategory` field** on every event so the purge job applies the correct TTL without fragile event-type enumeration
- **Privacy**: raw phone numbers replaced with HMAC-SHA256 hash. SMS content, blocked URLs, call numbers never stored.
- **Storage**: per-service DB tables over ELK — ELK's ILM delete phase is incompatible with append-only requirements
- **Retention proposals**: lifecycle 3 years, consent 5 years, admin actions 1 year, system events 90 days, security events 1 year

---

### SINDEV (Vault-grade platform standards)

From "09. Auditing and Logging Needs" ([Confluence](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6859637809)):

**Required audit fields per event:** event identifier, coordinated timestamp, request and correlation identifiers, actor identity and identity type, authentication method, tenant and namespace, resource identifier, operation, result and standardized error category.

**Hard rule:** "Secret payloads, passwords, private keys, bearer tokens, and recovery shares SHALL NOT be included."

**Sensitive field handling:** "For sensitive event fields such as token identifiers, keyed hashes may be used to preserve correlation without recording the original value."

**Audit failure behaviour** — notably stricter than UCP's current silently-ignore approach:
- Continue reads while buffering, up to a bounded limit
- Block sensitive administrative writes when no trustworthy audit path remains
- Alert immediately on audit destination failure
- **Avoid silently dropping events**

---

### Mobility (MPES) — log format standard for batch and API

From "Proposed Log Format Standard — Batch and External API" ([Confluence](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6723851353)):

- **`snake_case`** field names throughout
- **`error_type`** as a separate enum: `TIMEOUT`, `HTTP_4XX`, `HTTP_5XX`, `CONNECTION_REFUSED`, `PARSE_ERROR`, `UNKNOWN` — enables Elasticsearch alerting by error class
- "No full request/response bodies at INFO level. Full body logging permitted at DEBUG only."
- **"DEBUG must be disabled in production log sinks"** — explicit rule
- No PII, no credentials in log lines. `duration_ms` as integer.

---

### Mobile (RMCPS) — cross-platform log schema

From "Standard Log Schema — Mobile" ([Confluence](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6649078743)):

Required fields: `ts` (ISO-8601 UTC), `severity`, `component`, `correlation_id`, `event_type`.

**Error classification taxonomy** (enum, not free-form string):
`OFFLINE`, `NETWORK_TIMEOUT`, `HTTP_4XX`, `HTTP_5XX`, `AUTH_EXPIRED`, `AUTH_INVALID`, `VALIDATION`, `DB_IO`, `SDK_FAILURE`, `UNKNOWN`

**PII policy:** use `user_id_hash`, use `http_path_template` (`/v1/users/{id}/policy`) instead of raw URL with identifiers.

**#1 cross-team gap identified:** `correlation_id` is absent from most teams' current implementations — called out as the single most impactful missing field for end-to-end traceability.

---

### Cross-team patterns — summary

| Pattern | Prevalence |
|---|---|
| `snake_case` field names | Universal across all teams surveyed |
| `correlation_id` / `request_id` as mandatory | Called out as #1 gap; required in all standards |
| Error type as **enum**, not free-form string | MPES (`error_type`), Mobile (`error_class`), SECaaS (`errorCode`) |
| Separate audit log from application log | SECaaS, SINDEV — explicit architectural separation |
| Audit log append-only (no UPDATE/DELETE) | SECaaS, SINDEV — enforced at DB permission level |
| DEBUG disabled in production log sinks | MPES explicit rule; implied by others |
| PII must not appear in logs | Universal — RGR mandatory |
| Retention minimum 90 days for system events; 1 year for security; 3 years for lifecycle | SECaaS, CSDD guidance |

---

## Analysis and design

### MCUCP-256 — Logging policy

**Option A — Kubernetes/Cloud-native convention**

Widely used in cloud-native services:
- `Debug` — internal state useful only for debugging: SQL queries, cache hits/misses, JWT claim values during development
- `Info` — normal operational events: server start/stop, workflow submitted, request served
- `Warn` — unexpected but recoverable or non-blocking: dependency unavailable and retried, Keycloak logout failure, degraded health dependency
- `Error` — unexpected and requires attention: unrecoverable service error, 5xx, failed operations that affect correctness
- *(no Fatal in slog)* — process termination: `slog.Error` + `os.Exit(1)` for startup failures, `panic` for programming errors

This aligns with how the codebase already behaves (4xx → Warn, 5xx → Error). The main gap is `Debug` — nothing currently uses it.

**Option B — Stricter two-level policy (Info/Error only)**

Some teams use only Info and Error to reduce noise. `Warn` is dropped — every non-Info event is an Error. Simpler alerting threshold (alert on anything that is not Info), but produces noisier alerts for expected partial failures (e.g. a tenant with no IAM groups entry during `ListMyTenants`).

Not well-suited for UCP because non-blocking failures (logout revocation, health dependency degradation) should not trigger alerts but are currently logged at `Warn` — losing `Warn` would force a choice between silencing them or alerting on them.

**Recommendation:** Option A — Kubernetes convention. It matches existing codebase behaviour and is unambiguous for the cases UCP already has.

---

### MCUCP-257 — Audit logging design

**What should be audited**

Audit entries exist for **user-initiated state changes** with compliance or security significance. Three rules determine whether an operation needs an audit entry:

1. **State change** — something changed in the system (write operation). Read-only operations (`GET`, diagnostics) do not need audit entries.
2. **User-initiated** — triggered by an authenticated user action, not an automated background process (health checks, drift detection, scheduled jobs do not need entries).
3. **Security or compliance significance** — the action would matter to an auditor: authentication, authorization failures, credential changes, resource lifecycle events.

**How to write audit entries**

Every audit entry goes through `AuditService.InsertAuditLog` (the single write path). Each feature TRD defines the specific actions it produces and what goes in `metadata`. The convention for `metadata`:
- Holds **the minimum fields needed to reconstruct what changed** — not a full state snapshot
- For creates: include identifying fields (`gcp_project_id`, `sa_email`, etc.)
- For updates: include both old and new values (`old_sa_email`, `new_sa_email`)
- For deletes: include identifying fields sufficient for recovery or audit attribution

**Schema design — `audit_logs` table fields**

| Field | When populated | Notes |
|---|---|---|
| `user_id` | Always when authenticated; NULL for system-initiated events | UUID — never email or name |
| `request_id` | Always — correlates the entry to the HTTP response and application log | From request context |
| `action` | Always — machine-readable event name, `snake_case` verb | e.g. `auth.login`, `gcp_project_registered` |
| `tenant_rns` | When the event is scoped to a tenant | e.g. `rns:roc:iam::coupon-team`; NULL for system-wide events |
| `resource_type` | For resource-scoped operations | e.g. `gcp_project`, `compute`, `database`; NULL for auth/authz events |
| `resource_id` | For resource-scoped operations | Stable identifier of the affected resource; NULL for auth/authz events |
| `metadata` | Optional — operation-specific key-value pairs | See convention above |
| `created_at` | Always | The actual event time — for some events this is `auth_time` from the JWT, not `now()` |

**Write failure policy**

The question is: what happens when the DB is temporarily unreachable when an audit event needs to be written? Three options were evaluated.

**Option A — Same-transaction as business operation**

Write the audit entry in the same DB transaction as the business operation. If the audit write fails, the entire transaction rolls back.

- (+) Atomic — audit entry exists if and only if the business operation committed
- (-) A bug in the audit write (wrong data type, constraint violation) rolls back the business operation — user's action fails because of an audit logging bug. This is worse than a missing audit entry.
- (-) Couples audit logic to every business transaction; service layer must pass the transaction down to the audit write
- (-) Increases transaction duration on every write

**Option B — Outbox pattern (durable, independent)**

Write a "pending audit event" row in the same transaction as the business op. A background goroutine reads pending rows, writes to `audit_logs`, then deletes from the outbox. Survives restarts — if DB recovers after an outage, the backlog drains automatically.

- (+) Durable — event is persisted atomically with the business op; nothing is lost on crash or outage
- (+) Audit is fully independent from the business transaction
- (-) Requires an additional `audit_outbox` table and a background worker
- (-) No specific compliance mandate requiring this level of durability for UCP (confirmed via RGR research — MON-3/8/10 do not mandate zero-loss delivery for internal platform tools)
- (-) Added complexity for a risk level that has not been confirmed as required

**Option C — In-process async retry (recommended)**

After the business op commits, push the audit event to an in-memory channel. A background goroutine reads from the channel and writes to the DB, retrying with backoff on transient failures. If retries are exhausted, log at `Error` and discard.

- (+) Business op is fully independent — a bug or failure in the audit write never affects the user-facing operation
- (+) Handles the realistic failure mode (transient DB blip) without extra infrastructure
- (+) `AuditService` stays a self-contained injected dependency; callers fire-and-forget, retry is an implementation detail invisible to the rest of the codebase
- (-) If the process crashes in the millisecond window between business op commit and channel push, the event is lost. This is an accepted gap — the window is tiny, and no compliance mandate requires closing it for UCP.
- (-) In-flight events in the channel are lost on process restart. Same accepted gap.

**Recommendation: Option C.**

RGR MON-05 (REQUIRED) requires that when an event log processing failure occurs, designated personnel are **immediately alerted** and reasonable remediation actions are taken — with overwriting oldest records cited as the minimum acceptable response. It does not mandate zero-loss delivery.

Option C satisfies MON-05: retries handle transient failures, and exhausted retries produce an `Error` log that the monitoring stack can be configured to alert on. The alerting itself is an observability configuration concern, not a code architecture change.

Option A is rejected because coupling the audit write to the business transaction means an audit bug can break user operations. Option B (outbox) is the right upgrade path if a stricter compliance requirement arrives. For MVP, Option C is the correct trade-off: simple, independent, handles transient failures, satisfies MON-05, and upgradeable.

**Retention**

RGR MON-10 mandates the following for critical or sensitive IT assets (confirmed from the actual RGR document):
- **90 days minimum** in online, readily-available format
- **1 year minimum total** in online, archival, or offline storage format

The 1-year archival tier does not need to be online or queryable — archived or offline storage satisfies the requirement. UCP's retention baseline: 90 days hot, 1 year total.

The current `audit_logs` DDL is a plain unpartitioned table — no `PARTITION BY` clause exists. ADR-002 notes that table partitioning by month is the recommended approach for `audit_logs` at scale to enable cold-storage archival, but this is a future concern. It does not need to be addressed until the table grows large enough to make queries slow or storage cost meaningful.

---

## Recommendation

### Logging policy (MCUCP-256)

Adopt the Kubernetes/cloud-native convention with these UCP-specific rules:

| Level | UCP definition | Examples |
|---|---|---|
| `Debug` | Internal state, only useful with `LOG_LEVEL=debug` | JWT claims during validation, SQL queries, cache hit/miss |
| `Info` | Normal operational events | Server start/stop, workflow submitted, request completed (non-error) |
| `Warn` | Unexpected but recoverable or non-blocking | Keycloak logout revocation failure (non-blocking), health dependency degraded, soft-delete race condition |
| `Error` | Unexpected failure requiring attention | WIF verification error (blocks operation), 5xx HTTP response, DB error on critical write (e.g. `InsertSession`), audit write retries exhausted |

**Debug in production:** `LOG_LEVEL=debug` is not permitted in production. Production uses `LOG_LEVEL=info` (default). Dev/QA may set `LOG_LEVEL=debug`. This aligns with MPES's explicit rule and eliminates the risk of JWT claims leaking into production EaaS.

**Log format standard:**

| Decision | Recommendation | Rationale |
|---|---|---|
| Field naming | `snake_case` | Consistent with codebase, DB columns, no migration cost |
| Error field | `"error", err` | Idiomatic slog; safe because service code never holds DomainErrors |
| DomainError cause | `"error", de.Cause()` at handler/middleware layer | Shows root cause, not client-safe message |
| Auto-inject context fields | Context-enriched `slog.Handler` wrapper | Zero-effort `request_id` + `user_id` in every contextual log call |
| Sensitive fields | Convention: never log raw tokens; use `token[:8]+"..."` if needed | Handler scrubbing adds complexity not yet justified at current scale |

**Standard fields per context:**

| Context | Mandatory fields |
|---|---|
| HTTP request (authenticated) | `request_id` (auto), `user_id` (auto) |
| HTTP request (unauthenticated) | `request_id` (auto) |
| Service / background operation | At least one operation-scoped identifier (`gcp_project_id`, `workflow_id`, etc.) |
| Process lifecycle | None beyond slog built-ins |

**Error logging rule:**
- Service layer: `"error", rawErr` — always a raw Go error, never a `DomainError`
- Handler/middleware layer: `"error", de.Cause()` — the internal root cause of a `DomainError`
- This split holds because `DomainError` wrapping is exclusively a handler concern; service code produces `DomainErrors` but does not receive them

**Must never appear in logs:**
- JWT token strings or any substring
- Keycloak credentials or client secrets
- User PII beyond `user_id` (UUID) — no emails, usernames, display names in application logs
- SA email addresses in application logs (intentional in audit log `metadata` only)
- If a sensitive value must be referenced for debugging: log a non-reversible representation (`value[:8] + "..."` or a hash)

### Audit logging (MCUCP-257)

The RFC defines the design principles above and documents the existing implementation. Key decisions:

- **Viewer access to audit logs**: deferred — no `GET /audit-logs` endpoint planned near-term; DB-only access for now
- **`resource_type` and `resource_id`**: populate for all resource-scoped writes going forward; each feature TRD defines the values for its operations
- **Retention**: 90 days hot + 1 year total (RGR MON-10 mandated minimums). Archival/offline storage satisfies the 1-year tier. Table partitioning by month (per ADR-002) deferred until scale requires it; current DDL is unpartitioned
- **Write failure policy**: in-process async retry with backoff (Option C) — independent from business transactions, handles transient failures, no extra infrastructure; upgradeable to outbox pattern if a compliance mandate requires it
- **Provisioning audit entries**: each provisioning feature TRD defines its own entries following the schema convention here

---

## References

- [ADR-007 Observability Stack](../../../source/ucp-platform/docs/adr/ADR-007-observability-stack.md) — `log/slog` decision, EaaS transport
- [RFC-002 Go Codebase Standard](../../../source/ucp-platform/docs/rfcs/RFC-002-go-codebase-standard.md) — Group 5: Observability Stack, logging rationale
- [ADR-002 Platform Database Engine](../../../source/ucp-platform/docs/adr/ADR-002-platform-database-engine.md) — audit log scale estimate (~10,000 events/day), table partitioning
- [PRD-003 RBAC](../../../source/ucp-platform/docs/prd/PRD-003-rbac/PRD-003-rbac.md) — viewer access to audit logs requirement
- [Google SRE Book — Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/) — signal classification reference
- [Go log/slog documentation](https://pkg.go.dev/log/slog) — stdlib logger API, including `LogValuer` interface and `slog.Handler` contract
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html) — field naming standard for Elasticsearch; evaluated and deferred for UCP
- [Go slog context injection pattern](https://pkg.go.dev/log/slog#hdr-Attrs_and_Values) — `slog.Handler` wrapping for context-aware field injection
- [Rakuten CSDD — Security Practices for Logging & Audit](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=4775300409) — RGR policies overview and references
- [RGR [002411] Instruction for Continuous Monitoring (MON)](https://officerakuten.sharepoint.com/sites/Team-CCoE/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2FTeam%2DCCoE%2FShared%20Documents%2FSecurity%2F%5B002411%5DInstruction%20for%5FContinuous%20Monitoring%20%28MON%29%2Epdf&parent=%2Fsites%2FTeam%2DCCoE%2FShared%20Documents%2FSecurity&p=true&ga=1) — primary source for MON-03, MON-05, MON-07, MON-10 requirements (reviewed directly)
- [RMCPS — 22. Audit Logs (SECaaS / Link Security)](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6766875206) — canonical audit log schema, retention categories, append-only design
- [SINDEV — 09. Auditing and Logging Needs](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6859637809) — required audit fields, sensitive field hashing, audit failure behaviour
- [MPES — Proposed Log Format Standard — Batch and External API](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6723851353) — `snake_case`, `error_type` enum, DEBUG-in-production rule
- [RMCPS — Standard Log Schema — Mobile](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6649078743) — error classification taxonomy, correlation_id gap analysis
