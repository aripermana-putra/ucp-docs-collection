# Task Breakdown — MCUCP-129: US 1.1 — SSO Authentication

**Story:** [MCUCP-129](https://jira.rakuten-it.com/jira/browse/MCUCP-129)
**Parent task:** MCUCP-263

**Business scope:** UCP users log in via ROC Keycloak SSO — no separate UCP account required. Both CLI (interactive PKCE flow) and direct API access (bearer token, for CI/CD) authenticate against the same identity provider. UCP creates a user record on first login for audit attribution, validates every request via JWKS, and logs authentication events.

**Codebase:** monorepo at `ucp-platform/`. Feature slice implementation lives under `api-server/internal/<feature>/`. Shared cross-cutting code lives under `api-server/internal/shared/`. API contract is defined in `api-server/api/openapi.yaml` — server stubs (`api-server/gen/api.gen.go`) and CLI client (`cli/gen/client.gen.go`) are generated from it via `make generate`.

---

## Subtask 1: JWT validation middleware + JIT user provisioning
**Components:** API Server, Platform DB
**Blocks:** Subtask 2, Subtask 4

### API Contract
No new endpoints. Introduces the shared OpenAPI components that all subsequent subtasks depend on.

Add to `api-server/api/openapi.yaml`:

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: >
        Keycloak-issued JWT. Supplied by the CLI after login or via the
        UCP_TOKEN environment variable for CI/CD use.

  responses:
    Unauthorized:
      description: Missing or invalid authentication token
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    Forbidden:
      description: Authenticated but insufficient permissions for this operation
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    BadRequest:
      description: Invalid request payload or parameters
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    NotFound:
      description: Resource not found or not accessible to the caller
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
    InternalError:
      description: Unexpected server-side failure
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"

  schemas:
    ErrorResponse:
      type: object
      required: [code, message, requestId]
      properties:
        code:
          type: string
          description: Machine-readable error code.
          example: UNAUTHENTICATED
        message:
          type: string
          description: Human-readable description safe to display to the caller.
          example: "No valid session token. Run 'ucp auth login'."
        requestId:
          type: string
          format: uuid
          description: >
            Unique identifier for this request. Include this value when
            reporting an error — it correlates the response with the
            corresponding server-side log entry.
```

### Implementation
Implement in `api-server/internal/shared/middleware/`:

**JWT validation middleware (Echo middleware, applied to all protected routes):**
- Extract token from `Authorization: Bearer <token>` header or `UCP_TOKEN` env var (env var takes precedence, for CI/CD use)
- Keycloak configuration (`KEYCLOAK_ISSUER_URL`, `KEYCLOAK_JWKS_URI`, `KEYCLOAK_CLIENT_ID`) is read from environment variables at startup and passed as constructor arguments — no DB table
- Validate JWT signature against the configured JWKS URI with a 5-minute in-memory TTL cache
- On valid token: extract `sub`, `email`, `preferred_username`, `groups` claims → run JIT provisioning → inject `Principal` into Echo context
- On missing, malformed, or expired token: return `DomainError{Code: "UNAUTHENTICATED", Status: 401}` — routed through Echo's global `HTTPErrorHandler`

**JIT user provisioning (runs on every validated request):**

Every time a JWT is successfully validated, run an upsert to create or refresh the user record:

```sql
INSERT INTO users (external_id, email, username, display_name)
VALUES ($1, $2, $3, $4)
ON CONFLICT (external_id)
DO UPDATE SET
    email        = EXCLUDED.email,
    display_name = EXCLUDED.display_name,
    last_login_at = now(),
    login_count  = users.login_count + 1,
    updated_at   = now()
RETURNING id
```

The returned `id` is the `user_id` carried in the `Principal` for the rest of the request. It is the foreign key used in every audit log entry — no `user_id` means no audit attribution.

**DB unavailable during JIT provisioning:** fail closed — return `DomainError{Code: "INTERNAL_ERROR", Status: 500}`. Proceeding without a `user_id` would silently produce audit log entries with no attribution, which is a worse outcome than a temporary service interruption.

### DB Schema

```mermaid
erDiagram
    users {
        UUID id PK
        TEXT external_id UK
        TEXT email
        TEXT username
        TEXT display_name
        TEXT status
        TIMESTAMPTZ last_login_at
        INT login_count
        TIMESTAMPTZ created_at
        TIMESTAMPTZ updated_at
        TIMESTAMPTZ deleted_at
    }
```

```sql
CREATE TABLE users (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id   TEXT        NOT NULL,
    email         TEXT        NOT NULL,
    username      TEXT        NOT NULL,
    display_name  TEXT        NOT NULL DEFAULT '',
    status        TEXT        NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'suspended')),
    last_login_at TIMESTAMPTZ,
    login_count   INT         NOT NULL DEFAULT 0,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at    TIMESTAMPTZ,
    UNIQUE (external_id)
);

CREATE INDEX idx_users_email      ON users(email);
CREATE INDEX idx_users_active     ON users(id) WHERE deleted_at IS NULL;
```

> `external_id` is the Keycloak `sub` claim — unique per user across the ROC realm. `status` and `deleted_at` are distinct: `suspended` means temporarily blocked with the record intact; `deleted_at IS NOT NULL` means the account is gone but the row is preserved for audit log attribution.

---

## Subtask 2: API Server login flow (PKCE)
**Components:** API Server, Platform DB
**Blocked by:** Subtask 1
**Blocks:** Subtask 3, Subtask 4, MCUCP-220 Subtask 1

### API Contract
Add to `api-server/api/openapi.yaml`:

```yaml
paths:
  /auth/login:
    get:
      operationId: initiateLogin
      summary: Initiate Keycloak PKCE login flow
      tags: [auth]
      responses:
        "302":
          description: Redirect to Keycloak authorize endpoint
          headers:
            Location:
              schema:
                type: string
            Set-Cookie:
              description: HMAC-signed encrypted state cookie
              schema:
                type: string
        "500":
          $ref: "#/components/responses/InternalError"

  /auth/callback:
    get:
      operationId: handleCallback
      summary: Keycloak OIDC redirect callback
      tags: [auth]
      parameters:
        - name: code
          in: query
          required: true
          schema:
            type: string
        - name: state
          in: query
          required: true
          schema:
            type: string
      responses:
        "302":
          description: Login succeeded — redirect to app root with session cookie set (HttpOnly, SameSite=Strict, Secure, AES-GCM encrypted)
          headers:
            Location:
              schema:
                type: string
            Set-Cookie:
              schema:
                type: string
        "400":
          $ref: "#/components/responses/BadRequest"
        "503":
          description: Keycloak is unreachable or failed to exchange the authorization code
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
```

### Implementation
Implement in `api-server/internal/auth/`:

- `initiateLogin`: generate PKCE code verifier + challenge, generate state, set HMAC-signed encrypted state cookie, redirect to Keycloak `/authorize`
- `handleCallback`: validate state cookie, exchange code + PKCE verifier for tokens via Keycloak, validate ID token signature via JWKS, run JIT provisioning (via the shared middleware's upsert), create session record with AES-GCM encrypted tokens, set `HttpOnly + SameSite=Strict + Secure` session cookie, redirect to app root. A new login does **not** invalidate existing sessions — concurrent sessions are allowed in MVP.
- Session middleware (in `internal/shared/middleware/`): validate session cookie on each request, extract `sub` and `groups` claims from the stored access token, inject `Principal` into context. On invalid or expired session: return `DomainError{Code: "UNAUTHENTICATED", Status: 401}`.

### DB Schema

```mermaid
erDiagram
    users {
        UUID id PK
    }
    sessions {
        TEXT id PK
        UUID user_id FK
        TEXT access_token
        TEXT refresh_token
        TIMESTAMPTZ expires_at
        TIMESTAMPTZ created_at
        TIMESTAMPTZ last_seen_at
        INET ip_address
        TEXT user_agent
    }
    users ||--o{ sessions : "user_id"
```

```sql
CREATE TABLE sessions (
    id            TEXT        PRIMARY KEY,
    user_id       UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    access_token  TEXT        NOT NULL,
    refresh_token TEXT        NOT NULL,
    expires_at    TIMESTAMPTZ NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address    INET,
    user_agent    TEXT
);

CREATE INDEX idx_sessions_user_id    ON sessions(user_id);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

---

## Subtask 3: CLI login flow
**Components:** CLI
**Blocked by:** Subtask 1

### CLI Definition
Implement in `cli/`. This command talks directly to Keycloak — it does not go through the UCP API. No generated client is involved.

```
ucp auth login [--server <url>]

FLAGS:
  --server    UCP server URL (defaults to configured server)
```

Flow: start local callback server on port 18080 → open system browser to Keycloak authorize URL → receive authorization code on redirect → exchange code + PKCE verifier for tokens → store tokens in OS keychain (macOS Keychain, Linux SecretService, Windows Credential Manager) with encrypted JSON fallback at `~/.ucp/credentials`, indexed by server URL.

| Outcome | Output |
|---|---|
| Success | `Logged in as taro.rakuten@rakuten.com`<br>`Run 'ucp help' to see all available commands.` |
| Port 18080 unavailable | `Error: port 18080 is already in use. Free the port and run 'ucp auth login' again.` |
| Browser could not open | `Opening browser for login...`<br>`If your browser did not open, visit the following URL to complete login:`<br>`<authorize-url>` |
| Login cancelled or timed out | `Error: login was cancelled or timed out. Run 'ucp auth login' to try again.` |
| Token expired (on any subsequent command) | `Session expired. Run 'ucp auth login' to re-authenticate.` |

---

## Subtask 4: Auth login audit logging
**Components:** API Server, Platform DB
**Blocked by:** Subtask 2

### Implementation
No new endpoints. Implement a shared `AuditService` in `api-server/internal/shared/audit/` injected as a constructor dependency. Write an `auth.login` entry from inside the `handleCallback` handler (session-based login) and from the JWT middleware on first successful bearer token validation.

| Field | Value |
|---|---|
| `user_id` | resolved UCP user UUID from JIT provisioning |
| `action` | `auth.login` |
| `session_id` | session ID for session-based login; `null` for bearer token |
| `metadata` | `{}` (ip_address and failure tracking are Phase 2) |
| `created_at` | event timestamp |

### DB Schema

```mermaid
erDiagram
    users {
        UUID id PK
    }
    sessions {
        TEXT id PK
    }
    audit_logs {
        UUID id PK
        UUID user_id FK
        TEXT session_id FK
        TEXT action
        TEXT resource_type
        TEXT resource_id
        TEXT tenant_rns
        JSONB metadata
        TIMESTAMPTZ created_at
    }
    users    ||--o{ audit_logs : "user_id"
    sessions ||--o{ audit_logs : "session_id"
```

```sql
CREATE TABLE audit_logs (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID        REFERENCES users(id) ON DELETE SET NULL,
    session_id    TEXT        REFERENCES sessions(id) ON DELETE SET NULL,
    action        TEXT        NOT NULL,
    resource_type TEXT,
    resource_id   TEXT,
    tenant_rns    TEXT,
    metadata      JSONB,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_logs_user_id    ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action     ON audit_logs(action);
CREATE INDEX idx_audit_logs_tenant_rns ON audit_logs(tenant_rns);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
```

---

## Subtask 5: Offline token flow for Temporal workflows
**Components:** API Server, Platform DB
**Blocked by:** Subtask 2

> **External dependency:** confirm with the ROC Auth team that the UCP Keycloak client allows `offline_access` scope before this subtask starts. If not supported, fall back to a regular refresh token and document the accepted risk.

### Implementation
No new endpoints. Implement inside the workflow submission handler (separate story) as an extension point. Defined here for the DB schema.

At workflow submission: request `offline_access` scoped token from Keycloak, persist in `workflow_offline_tokens` linked to the workflow ID. On workflow resume: exchange the offline token for a fresh access token and re-evaluate the user's `groups` claim. On workflow completion or cancellation: revoke the offline token via Keycloak `/logout` and delete the row.

### DB Schema

```mermaid
erDiagram
    users {
        UUID id PK
    }
    workflow_offline_tokens {
        TEXT workflow_id PK
        TEXT offline_token
        UUID user_id FK
        TEXT tenant_rns
        TIMESTAMPTZ created_at
        TIMESTAMPTZ last_used_at
    }
    users ||--o{ workflow_offline_tokens : "user_id"
```

```sql
CREATE TABLE workflow_offline_tokens (
    workflow_id   TEXT        PRIMARY KEY,
    offline_token TEXT        NOT NULL,
    user_id       UUID        NOT NULL REFERENCES users(id),
    tenant_rns    TEXT        NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_used_at  TIMESTAMPTZ
);

CREATE INDEX idx_workflow_tokens_user_id ON workflow_offline_tokens(user_id);
```

---

## Open Questions
1. **Offline token `offline_access` scope** — requires ROC Auth team confirmation before Subtask 5 starts.
2. **Multiple ROC roles for the same service in JWT `groups` claim** — define the correct resolution strategy (e.g. last-parsed, highest role) before MVP.
