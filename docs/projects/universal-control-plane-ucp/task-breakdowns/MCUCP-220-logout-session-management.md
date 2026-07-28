# Task Breakdown — MCUCP-220: US 1.1b — Logout & Session Management

**Story:** [MCUCP-220](https://jira.rakuten-it.com/jira/browse/MCUCP-220)
**Parent task:** MCUCP-263

**Business scope:** Users can log out of their current UCP session and check their session status from the CLI. Logout invalidates the session server-side via Keycloak. Only the current session is affected — other active sessions on other devices are not invalidated.

---

## Subtask 1: API Server logout handler + logout audit logging
**Components:** API Server, Platform DB
**Blocked by:** MCUCP-129 Subtask 3 (sessions table + login flow), MCUCP-129 Subtask 5 (audit_logs table)

### API Contract

**`POST /auth/logout`**

Revokes the current session server-side and clears the session cookie.

| | |
|---|---|
| Auth required | Yes (valid session cookie or Bearer token) |
| Request body | None |

| Status | Condition | Body / Headers |
|---|---|---|
| `200 OK` | Session revoked successfully | `{ "message": "Logged out. Session token removed." }` — `Set-Cookie: session=; Max-Age=0` (cleared) |
| `200 OK` | Session already expired or not found | Same response — idempotent, no error |
| `401 Unauthorized` | No valid session | Standard `UNAUTHENTICATED` body |

Flow:
1. Call Keycloak `/logout` (end-session) endpoint with the current session's refresh token to revoke it server-side
2. Delete the session record from `sessions`
3. Write `auth.logout` audit log entry (see below)
4. Clear session and state cookies in the response

> **Note:** after logout, the access token remains valid until its Keycloak-configured TTL (~10 min in QA). Active access token revocation is not supported by Keycloak and is out of scope. This residual window is the accepted trade-off.

**Audit event written on successful logout:**

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

```
ucp auth logout [--server <url>]

FLAGS:
  --server    UCP server URL (defaults to configured server)
```

Flow:
1. Read stored credentials from OS keychain / `~/.ucp/credentials`
2. Call `POST /auth/logout` with the stored refresh token (Keycloak revocation must happen **before** local credential removal)
3. Remove all stored credentials for the current server from the OS keychain / credential file

| Outcome | Output |
|---|---|
| Success | `Logged out. Session token removed.` |
| Not logged in (no stored credentials) | `Not logged in.` |
| API Server unreachable | `Warning: could not reach server to invalidate session. Local credentials removed.`<br>(local credentials are still removed) |

---

## Subtask 3: `ucp auth status` command
**Components:** CLI
**Blocked by:** MCUCP-129 Subtask 4 (credential storage format)

### CLI Definition

```
ucp auth status [--server <url>]

FLAGS:
  --server    UCP server URL (defaults to configured server)
```

Reads stored credentials from OS keychain / credential file without making any API call.

| Outcome | Output |
|---|---|
| Logged in, token valid | `Logged in as: taro.rakuten@rakuten.com`<br>`Token expires: 2026-06-17T18:00:00Z (in 6h)` |
| Not logged in or token expired | `Not logged in. Run 'ucp auth login' to authenticate.` |

Exits 0 in both cases — status check is not an error condition. Expiry display shows both absolute ISO timestamp and human-readable duration from now.
