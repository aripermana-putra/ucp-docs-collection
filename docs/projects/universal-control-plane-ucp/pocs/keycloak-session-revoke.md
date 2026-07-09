# PoC: Keycloak Session Revoke

**Research question:** Does Keycloak provide a working token revocation endpoint that UCP can rely on for logout?

**Status:** Complete — both `/logout` and `/revoke` confirmed working; access token TTL is 10 minutes in QA; B01 is not a blocker

**Jira:** [MCUCP-238 B01](https://jira.rakuten-it.com/jira/browse/MCUCP-238)

---

## Sub-documents

| Document | Purpose |
|---|---|
| [design.md](keycloak-session-revoke/design.md) | Scope, hypothesis, approach, and success criteria — read before implementation |
| [concepts.md](keycloak-session-revoke/concepts.md) | Keycloak token lifecycle and revocation background |
| [implementation.md](keycloak-session-revoke/implementation.md) | What was built and run |
| [evidence.md](keycloak-session-revoke/evidence.md) | External references and Keycloak docs |
| [poc-report.md](keycloak-session-revoke/poc-report.md) | Verdict and recommendation |
