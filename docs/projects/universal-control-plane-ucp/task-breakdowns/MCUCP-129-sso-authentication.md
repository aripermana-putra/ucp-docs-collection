# Task Breakdown — MCUCP-129: US 1.1 — SSO Authentication

**Story:** [MCUCP-129](https://jira.rakuten-it.com/jira/browse/MCUCP-129)
**Parent task:** MCUCP-263

**Business scope:** UCP users authenticate via ROC Keycloak SSO — no separate UCP account or password required. Both the CLI (interactive PKCE flow) and direct API access (bearer token) must authenticate against the same identity provider. UCP creates user records on first login for stable audit attribution, validates tokens on every request, and logs authentication events.

---

## Authentication Middleware

### JWT validation middleware
Implement JWT signature validation via JWKS endpoint with an in-memory TTL cache (5-minute window). Extract `sub`, `email`, `preferred_username`, and `groups` claims from the validated token and inject a `Principal` into the request context. Reject requests with missing, malformed, or expired tokens with:
```
HTTP 401 — { "code": "UNAUTHENTICATED", "message": "No valid session token. Run 'ucp auth login'." }
```
Support both `Authorization: Bearer <token>` header and `UCP_TOKEN` environment variable (env var takes precedence, enabling CI/CD use without interactive login).

### Just-in-time user provisioning
On first authenticated request, create a user record via `INSERT ... ON CONFLICT (idp_id, external_id)` in the `users` table. Store `email`, `username`, `display_name` only — no roles. This record exists solely for stable `user_id` attribution in audit logs.

> **Open question (decide before MVP):** if the DB is unavailable during JIT provisioning, should the request fail closed (HTTP 500) or fail open (proceed without a `Principal`)? Fail-closed is safer but breaks all API access on DB downtime.

### TLS verification configuration
Remove all hardcoded `InsecureSkipVerify: true` from all files. Replace with per-endpoint deploy-time config flags:
- `TLS_SKIP_VERIFY_KEYCLOAK`
- `TLS_SKIP_VERIFY_HORIZON`
- `TLS_SKIP_VERIFY_OMNIA`

Default: `false` (verify). Production sets none. STG/QA sets relevant flags explicitly. Validate zero `InsecureSkipVerify: true` references remain in production-targeted code paths.

---

## API Gateway Login Flow

### PKCE login handler
Implement `LoginHandler`: generate PKCE code verifier + challenge, generate and sign state value, set HMAC-signed encrypted state cookie, redirect to Keycloak `/authorize`.

### PKCE callback handler
Implement `CallbackHandler`: validate state cookie, exchange authorization code + PKCE verifier for tokens, validate ID token signature via JWKS. Create a session record with AES-GCM encrypted tokens. Set `HttpOnly + SameSite=Strict + Secure` session cookie. A new login must **not** invalidate existing sessions on other devices — concurrent sessions are allowed in MVP.

### Session middleware (API Gateway)
Validate session cookie on each incoming request and inject `Principal` into request context. On expired or invalid session: return HTTP 401 with the standard `UNAUTHENTICATED` error body. Token TTL follows ROC Keycloak configuration — UCP does not extend or override it.

---

## CLI Login Flow

### `ucp auth login` command
Implement the PKCE login flow for the CLI:
1. Start a local callback server on port 18080
2. Open the system browser to Keycloak authorize URL with PKCE challenge
3. Receive the authorization code from the Keycloak redirect
4. Exchange code + PKCE verifier for tokens
5. Store tokens in OS keychain (macOS Keychain, Linux SecretService, Windows Credential Manager) with encrypted JSON fallback at `~/.ucp/credentials`, indexed by server URL for multi-server support

On success, print:
```
Logged in as taro.rakuten@rakuten.com
Run 'ucp help' to see all available commands.
```
On browser open failure, print the authorize URL as a fallback for manual navigation.

### Port conflict error handling
When port 18080 is unavailable, emit a diagnostic message identifying the port conflict and advising the user to free the port before retrying. Do not silently fail or print a generic error.

### Token expiry handling (CLI)
When any CLI command receives HTTP 401, display:
```
Session expired. Run 'ucp auth login' to re-authenticate.
```
Re-authentication is expected behavior — UCP does not proactively refresh tokens.

---

## Offline Token for Long-Running Temporal Workflows

### Offline token acquisition at workflow submission
At workflow submission time, request an `offline_access` scoped token from Keycloak. Store the offline token server-side associated with the workflow instance. When a paused workflow resumes, exchange the offline token for a fresh access token and re-evaluate the user's `groups` claim at resume time.

> **Open question (prerequisite):** confirm with the ROC Auth team that the UCP Keycloak client is configured to allow `offline_access` scope. Offline token TTL is indefinite (`exp=0`) — define the cleanup flow when the workflow completes or is cancelled. If `offline_access` is not supported, fall back to storing a regular refresh token with documented risk.

---

## Audit Logging

### Login audit events
Write an `auth.login` audit log entry on every successful login (both API Gateway and CLI paths):

| Field | Value |
|---|---|
| `user_id` | resolved UCP user ID |
| `action` | `auth.login` |
| `timestamp` | event time |

Implement via shared middleware or helper — not per-handler — to ensure consistent coverage across all authentication paths. `ip_address` and login failure tracking are out of scope for MVP.

---

## Dependencies
- `ucp auth status` command and `auth.logout` audit event are implemented as part of **MCUCP-220**
- Offline token flow depends on ROC Auth team confirmation

## Open Questions
1. **Fail-open vs fail-closed on DB unavailability during JIT provisioning** — must be decided before MVP.
2. **Offline token `offline_access` scope** — requires ROC Auth team confirmation before implementing the offline token flow.
3. **Multiple ROC roles for the same service in JWT `groups` claim** — current behavior is last-parsed wins. Define correct resolution strategy before MVP (e.g., highest role wins, explicit precedence list).
