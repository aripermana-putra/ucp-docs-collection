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

## LBaaS v3 API

| Fact | Source |
|---|---|
| The v3 swagger is Swagger 2.0 with `basePath: /v3`. Its endpoints are `/ping`, `/version`, `/info/...`, `/search`, `/tenants`, and per-tenant `acl_rules`, `certificates`, `load_balancers`, `load_balancer_statuses`, `server_groups`, `quotas`, `max_conn_limit`, `migrate` and `rollback`. Its OAuth2 token URL is `https://qa2-accounts-onecloud.rakuten-it.com/auth/realms/roc/protocol/openid-connect/token` in the QA spec file `user-swagger-qa.yaml`. | [LBaaS v3 Swagger UI (jpw2 prod host)](https://jpw2-user-dlb.r-local.net/v3/swagger/docs/user/index.html) |
| The generated Go client `lbaas-client` uses an unversioned spec and takes the regional host as `BaseURL` (`jpw2-user-dlb` or `jpe2-user-dlb`). | [roc-api-client](https://github.com/clsd-ucp/roc-api-client) |
| v3 endpoint patterns: Prod `https://<region>-user-dlb.r-local.net/v3/`; Dev `https://dev-<region>-user-dlb.r-local.net/v3/` (v3 only). Prod regions: jpe1, jpe2, jpw1, jpw2, jpc1, use1, usw1, euc1. Dev regions: jpe1, jpe2, jpw2. v2 is served at `https://lbaas.r-local.net/v2/`. `/v3/version` confirms the deployed version. | OneCloud documentation repo (docusaurus), `website/docs/Network/LBaaS/api/v3/lbaas-api-specification-v3.md`, `lbaas-api-version-history-v3.md` and `lbaas-v2-v3-differences.md` |
| QA v3 hosts `https://qa-jpc1-user-dlb.r-local.net/v3`, `https://qa-jpe1-user-dlb.r-local.net/v3` and `https://qa-jpe2-user-dlb.r-local.net/v3` were supplied by the PoC owner; the OneCloud documentation does not list them. | PoC owner |

## References

- [New Proxy Service - LBaaS Test](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6977413500/New+Proxy+Service+-+LBaaS+Test)
- [RFC 7519 — JSON Web Token, §4.1.3 "aud" Claim](https://www.rfc-editor.org/rfc/rfc7519#section-4.1.3)
- [OpenID Connect Core 1.0 — ID Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)
- [LBaaS v3 Swagger UI (jpw2 prod host)](https://jpw2-user-dlb.r-local.net/v3/swagger/docs/user/index.html)
- [roc-api-client](https://github.com/clsd-ucp/roc-api-client)
