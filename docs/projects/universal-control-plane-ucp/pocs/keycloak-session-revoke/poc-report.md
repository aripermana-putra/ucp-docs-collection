---
title: "Keycloak Session Revoke PoC — Report"
space: UCP
parent_page_id: "../keycloak-session-revoke.md"
---

# PoC Report: Keycloak Session Revoke

**Research question answered:** MCUCP-238 B01 — Does Keycloak provide a working revocation endpoint that UCP can rely on for logout?

**Status:** Complete

---

## Verdict

**Yes — Keycloak provides working revocation endpoints.** Both `/logout` and `/revoke` successfully invalidate the refresh token. The current UCP PoC implementation using `/logout` is correct and sufficient for the CLI logout flow.

The residual risk window is **10 minutes** — the access token TTL configured in the QA realm.

---

## What This PoC Proved

| Criterion | Result |
|---|---|
| `/logout` revokes a regular refresh token | PASS — `204`, subsequent refresh returns `400 invalid_grant — Session not active` |
| `/revoke` revokes a regular refresh token | PASS — `200`, subsequent refresh returns `400 invalid_grant — Session not active` |
| `/logout` revokes an offline token | PASS — `204`, subsequent refresh returns `400 invalid_grant — Offline user session not found` |
| Access token TTL (QA) | **10 minutes** (residual risk window after logout) |
| Regular refresh token TTL (QA) | **1 hour** (session-bound) |
| Offline token TTL | **No expiry** — `exp = 0` (Unix epoch); persists until explicitly revoked |
| Both endpoints usable for UCP logout | Yes — for the single-session CLI flow, outcome is identical |

**Offline tokens are stored separately in Keycloak** — confirmed by the distinct error message (`Offline user session not found` vs `Session not active`). Explicit `/logout` reaches into that store and removes the token.

**Resolved open questions:**

| Question | Finding |
|---|---|
| Is `rns:roc:portal` a public client (no secret required for PKCE)? | Yes — PKCE login succeeded without a client secret |
| Does Keycloak follow RFC 7009 and return `200 OK` for `/revoke`? | Yes — `200 OK` returned; verify step confirms actual revocation |
| Does `/logout` revoke offline tokens? | Yes — `204` returned; offline token returns `400 invalid_grant — Offline user session not found` |

## What This PoC Did Not Prove

- Access token active revocation — not possible without token introspection on every request. The 10-minute residual window is an accepted trade-off.
- Production environment behavior — only QA was tested.
- Multi-client SSO behavior — `/logout` vs `/revoke` distinction only matters when multiple clients share a Keycloak session.
- Offline token behavior when `/logout` is called **without** passing the offline token (e.g., session expiry only) — this PoC only tested explicit revocation.

---

## Endpoint Comparison

| | `/logout` | `/revoke` (RFC 7009) |
|---|---|---|
| Scope | Session-level — ends the entire Keycloak session | Token-level — revokes only the specific token passed |
| Response on success | `204 No Content` | `200 OK` |
| Response for unknown token | `400` | `200` (RFC 7009 requires 200 always) |
| Triggers back-channel logout | Yes, if configured | No |
| Right choice for UCP logout | Yes | Possible, but `/logout` is more appropriate |

---

## Recommendation

- **Use `/logout`** for UCP logout — it ends the session entirely, which matches the intent of a logout action.
- **Accept the 10-minute residual window** for access tokens — this is standard OAuth2 behavior. Token introspection on every request would close the gap but adds a Keycloak round-trip per API call.
- **Confirm access token TTL in production** — QA is 10 minutes; production may differ. A shorter TTL reduces the residual window.
- **B01 is not a blocker** — the endpoint exists, works, and is already called correctly in the PoC codebase.
