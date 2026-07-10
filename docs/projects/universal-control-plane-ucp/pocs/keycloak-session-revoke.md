---
title: "Keycloak Session Revoke PoC"
space: UCP
parent_page_id: "../pocs.md"
---

# PoC: Keycloak Session Revoke

**Research question:** Does Keycloak provide a working token revocation endpoint that UCP can rely on for logout?

**Status:** Complete — both `/logout` and `/revoke` confirmed working; access token TTL is 10 minutes in QA; B01 is not a blocker

**Jira:** [MCUCP-238 B01](https://jira.rakuten-it.com/jira/browse/MCUCP-238)

---

## Sub-documents

| Document | Purpose |
|---|---|
| [design.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6768760118) | Scope, hypothesis, approach, and success criteria — read before implementation |
| [concepts.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6768367629) | Keycloak token lifecycle and revocation background |
| [implementation.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6768367641) | What was built and run |
| [evidence.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6768367654) | External references and Keycloak docs |
| [poc-report.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6766882872) | Verdict and recommendation |
