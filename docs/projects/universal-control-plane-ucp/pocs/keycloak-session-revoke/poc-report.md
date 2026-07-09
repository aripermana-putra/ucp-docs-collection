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
| `/logout` revokes the refresh token | PASS — `204`, subsequent refresh returns `400 invalid_grant` |
| `/revoke` revokes the refresh token | PASS — `200`, subsequent refresh returns `400 invalid_grant` |
| Access token TTL (QA) | **10 minutes** |
| Both endpoints usable for UCP logout | Yes — for the single-session CLI flow, outcome is identical |

## What This PoC Did Not Prove

- Access token active revocation — not possible without token introspection on every request. The 10-minute residual window is an accepted trade-off.
- Production environment behavior — only QA was tested.
- Multi-client SSO behavior — `/logout` vs `/revoke` distinction only matters when multiple clients share a Keycloak session.

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
