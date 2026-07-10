---
title: "Keycloak Session Revoke PoC — Implementation"
space: UCP
parent_page_id: "../keycloak-session-revoke.md"
---

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

The script opens the browser three times — once per flow. All require a PKCE login against OneCloud Keycloak QA.

**Client ID note:** The default in the PoC codebase (`config.go`) is `ucp-cli`, but the actual registered client in the QA realm is `rns:roc:portal` (confirmed from `~/.ucp/config.yaml`).

---

## What the Script Does

Three independent flows run sequentially:

1. **Flow A** — login → `POST /logout` with regular refresh token → verify `400 invalid_grant`
2. **Flow B** — login → `POST /revoke` with regular refresh token → verify `400 invalid_grant`
3. **Flow C** — login with `offline_access` scope → `POST /logout` with offline token → verify `400 invalid_grant`

Each flow prints both the access token and refresh token TTL decoded from the JWT `iat`/`exp` claims.

---

## Results

### Flow A — `/logout` endpoint (regular refresh token)

```
access_token  TTL: 10m0s (expires at 2026-07-10T10:15:07+09:00)
refresh_token TTL: 1h0m0s (expires at 2026-07-10T11:05:07+09:00)

--- Step 2: POST /logout with refresh_token ---
status: 204
body:   (empty)

--- Step 3: Verify ---
status: 400
body:   {"error":"invalid_grant","error_description":"Session not active"}
RESULT: PASS — refresh token is revoked (400 invalid_grant)
```

### Flow B — `/revoke` endpoint (RFC 7009, regular refresh token)

```
access_token  TTL: 10m0s (expires at 2026-07-10T10:15:10+09:00)
refresh_token TTL: 1h0m0s (expires at 2026-07-10T11:05:10+09:00)

--- Step 2: POST /revoke with refresh_token ---
status: 200
body:   (empty)

--- Step 3: Verify ---
status: 400
body:   {"error":"invalid_grant","error_description":"Session not active"}
RESULT: PASS — refresh token is revoked (400 invalid_grant)
```

### Flow C — `/logout` endpoint (offline token)

```
access_token  TTL: 10m0s (expires at 2026-07-10T10:15:12+09:00)
refresh_token TTL: -495457h5m12s (expires at 1970-01-01T09:00:00+09:00)

--- Step 2: POST /logout with offline refresh_token ---
status: 204
body:   (empty)

--- Step 3: Verify ---
status: 400
body:   {"error":"invalid_grant","error_description":"Offline user session not found"}
RESULT: PASS — refresh token is revoked (400 invalid_grant)
```

---

## Key Observations

**Token TTL comparison (QA realm):**

| Token | Regular session | Offline (`offline_access`) |
|---|---|---|
| `access_token` | 10 minutes | 10 minutes |
| `refresh_token` | 1 hour (session-bound) | `exp = 0` (no expiry) |

**Offline token `exp = 0`** — Keycloak sets the expiry to Unix epoch (0), meaning it carries no session-based expiry. It persists indefinitely until explicitly revoked or removed via the Keycloak admin console.

**Different error messages confirm different storage:**
- Regular token: `"Session not active"` — the Keycloak session was ended
- Offline token: `"Offline user session not found"` — Keycloak stores offline tokens separately; `/logout` reaches into that store and removes it

**Response codes differ by design:**
- `/logout` → `204 No Content` (session ended)
- `/revoke` → `200 OK` (RFC 7009 — returns 200 even for unknown tokens)

**Behavioral difference:** `/logout` ends the entire Keycloak session (affects all clients sharing the session). `/revoke` is token-specific — only the passed token is invalidated. For UCP's single-session CLI flow the outcome is identical, but the distinction matters if multi-client SSO is introduced.
