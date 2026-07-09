# Concepts: Keycloak Session Revoke

## Keycloak Token Types

| Token | Lifetime | Revocable? |
|---|---|---|
| Access token | Short (minutes) | No — expires naturally |
| Refresh token | Long (hours/days) | Yes — via `/logout` or `/revoke` |
| Offline token | Indefinite | Yes — via `/logout` or `/revoke`, or admin console |
| ID token | Short | No — informational only |

## Revocation Endpoints

Keycloak exposes two endpoints for token invalidation:

**`POST /protocol/openid-connect/logout`**
- Logs out the Keycloak session associated with the refresh token
- Invalidates the refresh token and the session
- May trigger back-channel logout notifications to other clients (if configured)
- Returns `204 No Content` on success

**`POST /protocol/openid-connect/revoke`** (RFC 7009)
- Revokes a specific token (refresh or access)
- Does not necessarily end the Keycloak session
- Returns `200 OK` even for unknown tokens (RFC 7009 compliance)

## Access Token Behavior After Logout

Access tokens are JWTs validated locally via JWKS. Keycloak does not maintain a blocklist for access tokens by default. After logout:

- The refresh token is revoked — no new access tokens can be issued
- The current access token remains valid until its `exp` claim
- Any caller holding the access token can continue making authenticated API calls until it expires

This creates a residual risk window between logout and access token expiry:

```
logout called
    │
    ├─ refresh token → revoked immediately
    │
    └─ access token → still valid until exp
                      (anyone holding it can still call the API)
```

The window size equals the access token TTL configured in the Keycloak realm. A shorter TTL reduces exposure but does not eliminate it.

To eliminate the window entirely, every API request would need to call Keycloak's `/introspect` endpoint to check whether the token is still active. This adds a Keycloak round-trip on every request and is not implemented in the current UCP design.

The accepted trade-off: revoke the refresh token on logout to prevent new tokens from being issued, and rely on a short access token TTL to limit the residual window.

## PKCE Flow (Public Client)

UCP uses Authorization Code + PKCE for CLI and browser login. PKCE eliminates the need for a client secret on public clients (CLI, SPAs). The `code_verifier` generated client-side proves ownership of the authorization code without a secret.

## OneCloud Keycloak (ROC Realm)

UCP authenticates against the `roc` realm of OneCloud Keycloak:

| Environment | Realm URL |
|---|---|
| QA | `https://qa2-accounts-onecloud.rakuten-it.com/auth/realms/roc` |
| Production | `https://accounts-onecloud.rakuten-it.com/auth/realms/roc` |
