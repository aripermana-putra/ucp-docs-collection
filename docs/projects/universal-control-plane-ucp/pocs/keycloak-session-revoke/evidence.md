# Evidence: Keycloak Session Revoke

## Keycloak Official Documentation

| Claim | Source |
|---|---|
| Keycloak logout endpoint accepts refresh token and invalidates the session | [Keycloak Docs — Session and Token Revocation](https://www.keycloak.org/docs/latest/securing_apps/#logout) |
| RFC 7009 `/revoke` endpoint supported since Keycloak 12 | [Keycloak Docs — OpenID Connect endpoints](https://www.keycloak.org/docs/latest/securing_apps/#endpoints) |
| Access tokens are not actively revocable by default (JWT validation is local) | [Keycloak Docs — Token Introspection](https://www.keycloak.org/docs/latest/securing_apps/#_token_introspection_endpoint) |
| Offline tokens persist across user logout and session expiry | [Keycloak Docs — Offline Access](https://www.keycloak.org/docs/latest/server_admin/#_offline-access) |

## RFC References

| RFC | Relevance |
|---|---|
| [RFC 7009 — OAuth 2.0 Token Revocation](https://www.rfc-editor.org/rfc/rfc7009) | Defines the `/revoke` endpoint semantics; server returns 200 even for unknown tokens |
| [RFC 8252 — OAuth 2.0 for Native Apps](https://www.rfc-editor.org/rfc/rfc8252) | Basis for PKCE flow used by UCP CLI |

## UCP Codebase Observations

From code review of `ucp-platform-poc` (not external, recorded for traceability):

| Observation | Location |
|---|---|
| CLI logout calls `POST /protocol/openid-connect/logout` with refresh token | `cli/internal/auth/login.go:142-154` |
| API Gateway logout calls same endpoint via `revokeRefreshToken` | `backend/api-server/bff_auth.go:721-751` |
| CLI silently ignores revocation error (`_ = ucpauth.Logout(...)`) | `cli/cmd/auth/logout.go:29` |
| Both use `/logout`, not RFC 7009 `/revoke` | Both files above |
