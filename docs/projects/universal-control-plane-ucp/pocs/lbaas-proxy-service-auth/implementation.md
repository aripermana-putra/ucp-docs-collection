---
title: "LBaaS Proxy Service Auth PoC — Implementation"
space: UCP
parent_page_id: "../lbaas-proxy-service-auth.md"
---

# Implementation: LBaaS Proxy Service Auth PoC

## Source Code

| Item | Value |
|---|---|
| Repository | [aripermana-putra/kitchen-sink](https://github.com/aripermana-putra/kitchen-sink) |
| Path | `lbaas-proxy-auth-poc/main.go` |
| Entry point | `main()` |
| Base commit | `d81be3e` (the PoC directory is not yet committed; no PoC commit SHA exists) |
| Raw run output | `lbaas-proxy-auth-poc/run-output.txt`, `run-output-2.txt`, `run-output-variants.txt` |

---

## What Was Built

A single-file Go program using only the standard library. Ed25519 signing uses `crypto/ed25519`; the JWT is assembled by hand (base64url header, claims and signature), so no JOSE dependency is needed.

| Function | Responsibility |
|---|---|
| `signAssertor` | Reads the service account JWK (`kty=OKP`, `crv=Ed25519`), signs the assertor JWT with EdDSA. Claims: `iss`/`sub` = service account, `aud=rns:roc:lbaas`, `exp = iat + 20s`, random `jti`. |
| `exchange` | POSTs `grant_type=client_credentials` with the JWT-bearer `client_assertion` to the proxy `/token` endpoint. |
| `decodeJWT` | Decodes header and claims of a JWT without verifying it. |
| `checkIssuer` | Resolves the `iss` host, fetches `/.well-known/openid-configuration` and the `jwks_uri`. |
| `callLBaaS` | `GET` against LBaaS with an optional bearer token; prints status, body (truncated to 400 chars), `X-Request-Id`, `Www-Authenticate`, `Server`. |

Tokens are held in memory only. Output contains decoded claims and responses, never raw tokens or key material.

## How It Was Run

```bash
cd lbaas-proxy-auth-poc
JWK_PATH=<path to downloaded service account JWK> go run .
```

| Variable | Default |
|---|---|
| `JWK_PATH` | required |
| `SA_ACCOUNT` | `ucp-cli-auth@clsd-ucp.iam.service-accounts.roc` |
| `PROXY_TOKEN_URL` | `https://roc-tenant-sa-kc-jws-80-keycloak.jpe2-caas1-beta1.caas.jpe2z.r-local.net/token` |
| `LBAAS_BASE_URL` | `https://qa2-lbaas.r-local.net` |
| `TLS_INSECURE` | `true` (TLS verification skipped, as `curl -k` in the source test) |
| `ASSERTOR_AUD_ARRAY` | `false`; `true` signs the assertor with `aud` as `["rns:roc:lbaas"]` |
| `TOKEN_SCOPE` | unset; when set, sent as the `scope` form field on the token request |
| `CONTROLS` | `true`; `false` skips the control calls |

Steps executed in order: sign assertor, token exchange, access token decode, issuer checks, LBaaS call with the proxy token, control calls. The program was run twice with defaults, then three times with variants (`ASSERTOR_AUD_ARRAY`, `TOKEN_SCOPE`, both) and `CONTROLS=false`. Results are recorded in `poc-report.md`.

### Known-good comparison

User tokens from the OC portal were decoded and sent to LBaaS with `curl` and a `Authorization: Bearer` header, outside the Go program. The tokens are short-lived (600 s), supplied by the user, held in shell memory only and not stored in the repository or docs. The decode printed `iss`, `azp`, `aud` (type and entries), groups and claim names.

## Limitations

- The Go HTTP client follows redirects, so control requests that LBaaS redirects to the Keycloak login page are reported as `200` with an HTML body.
- Token claims are decoded but signatures are not verified locally.
- Only `GET` requests are made; no LBaaS write operations are exercised.
