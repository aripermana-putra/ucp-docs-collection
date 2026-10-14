---
title: "LBaaS Proxy Service Auth PoC — Evidence"
space: UCP
parent_page_id: "../lbaas-proxy-service-auth.md"
---

# Evidence: LBaaS Proxy Service Auth PoC

Externally sourced facts only. Observed results from the PoC run are in `poc-report.md`.

## Source Test

| Fact | Source |
|---|---|
| The test flow is: create a service account in OC QA with roles, download its JWK, sign an assertor token, exchange it at the new proxy service for an access token, call the LBaaS API with that token. | [New Proxy Service - LBaaS Test](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6977413500/New+Proxy+Service+-+LBaaS+Test) |
| The assertor token uses `alg=EdDSA`, `aud=rns:roc:lbaas`, a 20-second lifetime and a unique `jti`. | same |
| The proxy `/token` call returns HTTP 200 with a 43200 s bearer access token (`aud=rns:roc:lbaas`, `azp=rns:roc:iam`, groups `lbaas-operator` and `lbaas-viewer` for `clsd-ucp`). | same |
| `GET https://qa2-lbaas.r-local.net/v2/tenants` with that token returns HTTP 500 `{"message":"Internal Server Error"}`; recorded `x-request-id` is `ia-v2-wWQCzhl41RaRRF95i25FQyvS9w`. The test is recorded as failed. | same |

## JWT and OIDC Claim Definitions

| Fact | Source |
|---|---|
| The `aud` claim is either a single case-sensitive string or an array of such strings. | [RFC 7519 §4.1.3](https://www.rfc-editor.org/rfc/rfc7519#section-4.1.3) |
| The `azp` (authorized party) claim identifies the OAuth client the ID token was issued to. | [OpenID Connect Core 1.0 §2](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) |

## References

- [New Proxy Service - LBaaS Test](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6977413500/New+Proxy+Service+-+LBaaS+Test)
- [RFC 7519 — JSON Web Token, §4.1.3 "aud" Claim](https://www.rfc-editor.org/rfc/rfc7519#section-4.1.3)
- [OpenID Connect Core 1.0 — ID Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)
