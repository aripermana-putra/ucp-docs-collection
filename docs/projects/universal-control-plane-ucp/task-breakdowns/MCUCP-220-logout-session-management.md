# Task Breakdown — MCUCP-220: US 1.1b — Logout & Session Management

**Story:** [MCUCP-220](https://jira.rakuten-it.com/jira/browse/MCUCP-220)
**Parent task:** MCUCP-263

**Business scope:** Users can log out of their current UCP session and check their session status from the CLI. Logout revokes the session server-side via Keycloak. Only the current session is affected — other active sessions on other devices are not invalidated.

**Codebase:** monorepo at `ucp-platform/`. Feature slice implementation in `api-server/internal/auth/`. API contract defined in `api-server/api/openapi.yaml`.

---

## Subtask 1: API Server logout handler + logout audit logging
**Components:** API Server, Platform DB
**Blocked by:** MCUCP-129 Subtask 2 (sessions table + PKCE login flow), MCUCP-129 Subtask 4 (audit_logs table)

### API Contract
Add to `api-server/api/openapi.yaml`:

```yaml
paths:
  /auth/logout:
    post:
      operationId: logout
      summary: Revoke current session and clear session cookie
      tags: [auth]
      security:
        - sessionCookie: []
        - bearerAuth: []
      responses:
        "200":
          description: Session revoked. Always returns 200 — idempotent if session already expired.
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/LogoutResponse"
        "401":
          $ref: "#/components/responses/Unauthorized"

components:
  schemas:
    LogoutResponse:
      type: object
      required: [message]
      properties:
        message:
          type: string
          example: "Logged out. Session token removed."
```

### Implementation
Implement in `api-server/internal/auth/`:

- `logout` handler: call Keycloak `/logout` (end-session) endpoint with the current session's refresh token, delete the session row from `sessions`, write `auth.logout` audit log entry, clear session and state cookies. Returns `200` in all cases — idempotent if session is already expired or not found.
- Use the shared `AuditService` (from MCUCP-129 Subtask 4) to write the audit entry.

> **Note:** after logout, the access token remains valid until its Keycloak-configured TTL (~10 min in QA). Active access token revocation is not supported by Keycloak — this residual window is the accepted trade-off.

**Audit event:**

| Field | Value |
|---|---|
| `user_id` | resolved UCP user UUID |
| `action` | `auth.logout` |
| `session_id` | revoked session ID |
| `created_at` | event timestamp |

---

## Subtask 2: CLI logout command
**Components:** CLI
**Blocked by:** Subtask 1

### CLI Definition
Implement in `cli/`. Uses the generated client from `cli/gen/client.gen.go` to call `POST /auth/logout`.

```
ucp auth logout [--server <url>]

FLAGS:
  --server    UCP server URL (defaults to configured server)
```

Flow: call `POST /auth/logout` with the stored session token (Keycloak revocation happens server-side) → on success, remove stored credentials for this server from OS keychain / `~/.ucp/credentials`.

| Outcome | Output |
|---|---|
| Success | `Logged out. Session token removed.` |
| Not logged in (no stored credentials) | `Not logged in.` |
| Server unreachable | `Warning: could not reach server to invalidate session. Local credentials removed.` (credentials are still removed locally) |

---

## Subtask 3: `ucp auth status` command
**Components:** CLI
**Blocked by:** MCUCP-129 Subtask 3 (credential storage format must be defined first)

### CLI Definition
Implement in `cli/`. Reads stored credentials locally — no API call.

```
ucp auth status [--server <url>]

FLAGS:
  --server    UCP server URL (defaults to configured server)
```

| Outcome | Output |
|---|---|
| Logged in, token valid | `Logged in as: taro.rakuten@rakuten.com`<br>`Token expires: 2026-06-17T18:00:00Z (in 6h)` |
| Not logged in or token expired | `Not logged in. Run 'ucp auth login' to authenticate.` |

Exits 0 in both cases. Expiry shows both absolute ISO 8601 timestamp and human-readable duration from now.
