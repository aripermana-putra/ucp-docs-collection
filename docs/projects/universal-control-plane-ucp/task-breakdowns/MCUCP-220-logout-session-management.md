# Task Breakdown — MCUCP-220: US 1.1b — Logout & Session Management

**Story:** [MCUCP-220](https://jira.rakuten-it.com/jira/browse/MCUCP-220)
**Parent task:** MCUCP-263

**Business scope:** Users can log out of their current UCP session and check their current session status from the CLI. Logout invalidates the session server-side via Keycloak. Only the current session is affected — other active sessions on other devices are not invalidated.

---

## API Gateway Logout

### Logout handler
Call the Keycloak `/logout` (end-session) endpoint with the current session's refresh token to revoke it server-side. On success: delete the session record from DB and clear both session and state cookies in the response. Return a confirmation message. Only the current session is revoked; other active sessions on other devices continue until their tokens expire naturally.

> **Note:** After logout, the access token remains valid until its natural expiry (Keycloak-configured TTL — ~10 min in QA). Active revocation of access tokens is not supported by Keycloak and is out of scope. This residual window is the accepted trade-off.

### Standard logout response
Returned to the caller after successful logout:
```
Logged out. Session token removed.
```
HTTP 200 with cleared cookies. If the session is already expired or not found, return the same response without error — idempotent behavior.

---

## CLI Logout

### `ucp auth logout` command
1. Read stored credentials from OS keychain / `~/.ucp/credentials`
2. Call Keycloak `/logout` endpoint with the stored refresh token to revoke the session server-side
3. On success: remove all stored credentials for the current server from the OS keychain / credential file
4. Print:
   ```
   Logged out. Session token removed.
   ```
5. If no stored credentials exist: print `Not logged in.` and exit cleanly (no error)

The Keycloak `/logout` endpoint must be called **before** removing local credentials — this ensures the server-side session is revoked even if the credential cleanup fails midway.

---

## Session Status

### `ucp auth status` command
Read stored credentials from OS keychain / credential file without making any API call.

When logged in:
```
Logged in as: taro.rakuten@rakuten.com
Token expires: 2026-06-17T18:00:00Z (in 6h)
```
Display absolute timestamp + human-readable duration from now.

When not logged in or token is expired:
```
Not logged in. Run 'ucp auth login' to authenticate.
```
Exit 0 in both cases (status check is not an error condition).

---

## Audit Logging

### Logout audit events
Write an `auth.logout` audit log entry on every successful logout (both API Gateway and CLI paths):

| Field | Value |
|---|---|
| `user_id` | resolved UCP user ID |
| `action` | `auth.logout` |
| `timestamp` | event time |

Reuse the same audit log writer/helper established in MCUCP-129 to ensure consistent format.

---

## Dependencies
- Depends on session model and credential storage from **MCUCP-129**
- Keycloak `/logout` endpoint behavior confirmed: revokes refresh tokens and offline tokens. `/revoke` (RFC 7009) is also valid but `/logout` is preferred as it ends the entire session.
