---
title: "LBaaS Proxy Service Auth PoC"
space: UCP
parent_page_id: "../pocs.md"
---

# PoC: LBaaS Proxy Service Auth

**Research question:** Why does an access token issued by the new proxy service for a service account get rejected with HTTP 500 by the LBaaS API, and which part of the service-to-service flow causes it?

**Source test:** [New Proxy Service - LBaaS Test](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6977413500/New+Proxy+Service+-+LBaaS+Test)

---

## Sub-documents

| Document | Purpose |
|---|---|
| design.md | Scope, hypotheses, approach, and success criteria — read before implementation |
| poc-report.md | Verdict, observed data, and recommendation |
| implementation.md | The Go program, how it was run, and limitations |
| evidence.md | Facts taken from the source Confluence test |

**Status:** Failure reproduced; the 500 occurs in the LBaaS entry layer after signature validation. A QA2 portal user token is accepted. The differing claim (`aud` type, `azp`, or missing claims) is not identified and needs a proxy-side change to test — see poc-report.md.
