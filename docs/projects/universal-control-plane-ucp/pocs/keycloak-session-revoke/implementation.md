# Implementation: Keycloak Session Revoke PoC

## Repository

- **Repo:** kitchen-sink
- **Branch:** `poc/MCUCP-238-keycloak-session-revoke`
- **Script:** `keycloak-session-revoke/main.go`

## How to Run

```bash
cd keycloak-session-revoke

KEYCLOAK_ISSUER=https://qa2-accounts-onecloud.rakuten-it.com/auth/realms/roc \
KEYCLOAK_CLIENT_ID=rns:roc:portal \
go run .
```

The script opens the browser twice — once per flow. Both require a PKCE login against OneCloud Keycloak QA.

**Client ID note:** The default in the PoC codebase (`config.go`) is `ucp-cli`, but the actual registered client in the QA realm is `rns:roc:portal` (confirmed from `~/.ucp/config.yaml`).

---

## What the Script Does

Two independent flows run sequentially:

1. **Flow A** — login → `POST /logout` with refresh token → attempt refresh → verify `400 invalid_grant`
2. **Flow B** — login → `POST /revoke` with refresh token → attempt refresh → verify `400 invalid_grant`

Each flow also prints the access token TTL decoded from the JWT `iat`/`exp` claims.

---

## Results

### Flow A — `/logout` endpoint

```
access_token TTL: 10m0s (expires at 2026-07-09T17:46:45+09:00)

--- Step 2: POST /logout with refresh_token ---
status: 204
body:   (empty)

--- Step 3: Verify ---
status: 400
body:   {"error":"invalid_grant","error_description":"Session not active"}
RESULT: PASS — refresh token is revoked (400 invalid_grant)
```

### Flow B — `/revoke` endpoint (RFC 7009)

```
access_token TTL: 10m0s (expires at 2026-07-09T17:46:50+09:00)

--- Step 2: POST /revoke with refresh_token ---
status: 200
body:   (empty)

--- Step 3: Verify ---
status: 400
body:   {"error":"invalid_grant","error_description":"Session not active"}
RESULT: PASS — refresh token is revoked (400 invalid_grant)
```

---

## Key Observations

**Access token TTL is 10 minutes** in the QA realm. This is the residual risk window — an access token obtained before logout remains valid for up to 10 minutes after the refresh token is revoked.

**Both endpoints revoke the refresh token** — the verify step returns `400 invalid_grant` with `"Session not active"` for both.

**Response codes differ by design:**
- `/logout` → `204 No Content` (session ended)
- `/revoke` → `200 OK` (RFC 7009 — returns 200 even for unknown tokens)

**Behavioral difference:** `/logout` ends the entire Keycloak session (affects all clients sharing the session). `/revoke` is token-specific — only the passed token is invalidated, other tokens in the same session are unaffected. For UCP's single-session CLI flow the outcome is identical, but the distinction matters if multi-client SSO is introduced.
