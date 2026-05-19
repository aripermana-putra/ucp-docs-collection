---
title: "Tenant Isolation — Concepts"
space: UCP
parent_page_id: "../tenant-isolation.md"
---

# Tenant Isolation — Concepts

---

## ProviderConfig Per Tenant

A `ProviderConfig` is a Crossplane resource that holds connection credentials for a cloud
provider. UCP creates one `ProviderConfig` per tenant per environment (QA and production).
All XRs provisioned for a tenant reference that tenant's `ProviderConfig`, which directs
Crossplane to act on the tenant's cloud account, not a shared one.

The `ProviderConfig` name is deterministic:

```
gcp-{sanitized-tenant-id}-{env}
```

For example: `gcp-rns-roc-iam--clsd-ucp-qa`.

This name is computed server-side and injected into the XR at creation time. Callers
cannot supply or override it.

---

## Tenant Identity

Tenant context is expressed as an RNS (Rakuten Name Space) string: `rns:roc:iam::clsd-ucp`.

The API server verifies tenant admin membership for every mutating and tenant-scoped read
operation by calling the Horizon API:

```
GET https://qa-horizon-data-api.r-local.net/v0/tenants/{tenantRNS}
Authorization: Bearer {caller's Keycloak access token}

Response: {"admins": [{"email": "user@example.com"}, ...]}
```

The caller's email is checked against the `admins` list. Any operation that targets a
specific tenant rejects non-admins with HTTP 403.

---

## Ownership Labels and Annotations

Every XR created by the API server carries two tenant ownership markers:

| Type | Key | Value example | Purpose |
|---|---|---|---|
| Label | `platform.ucp.io/tenant` | `rns-roc-iam--clsd-ucp` | Kubernetes label selectors — `:` is not a valid label value character |
| Annotation | `platform.ucp.io/tenant-id` | `rns:roc:iam::clsd-ucp` | Raw RNS format — required to call `isUserTenantAdmin()` and for exact string comparison |

Both are set at XR creation time. The label is used for list filtering. The annotation is
used in delete ownership checks and in-flight workflow matching, where the raw RNS string
is needed.

---

## sanitizeTenantID

Converts a raw RNS tenant ID into a Kubernetes-safe string by replacing all `:` with `-`:

```go
func sanitizeTenantID(tenantID string) string {
    return strings.NewReplacer(":", "-").Replace(strings.ToLower(tenantID))
}
```

`rns:roc:iam::clsd-ucp` → `rns-roc-iam--clsd-ucp`

---

## xrBelongsToTenant

`xrBelongsToTenant` determines whether a Kubernetes XR object belongs to a given tenant.
It checks three fallbacks in order:

1. `platform.ucp.io/tenant-id` annotation (exact RNS match)
2. `platform.ucp.io/tenant` label (sanitized match)
3. `spec.parameters.providerConfig` name match against `gcpProviderConfigName(tenantID, env)`

The third fallback covers XRs provisioned before the label/annotation stamping was
introduced. Once all active XRs carry the annotation, the third check becomes redundant.

---

## BFF Authentication

The API server implements the Backend For Frontend pattern. Keycloak JWTs never reach
the browser. The browser holds only an opaque session cookie (`session=<hex-id>`), which
maps to an encrypted token pair in the API server's PostgreSQL session store.

On each authenticated request:
1. The session middleware reads the cookie and looks up the session record.
2. It decrypts the stored access token (AES-GCM).
3. If the access token is expired, it transparently refreshes via Keycloak.
4. It injects a `Principal` (including the decrypted access token) into the request context.

Downstream handlers read `principal.AccessToken` to make Horizon API calls
(`isUserTenantAdmin`) and other authenticated operations.

---

## Why Namespace Per Tenant Does Not Isolate Crossplane Resources

Kubernetes RBAC can restrict access to namespaced resources by namespace. However,
Crossplane XRs and managed resources (MRs) from the providers UCP uses are cluster-scoped
(`scope: Cluster`). Cluster-scoped resources have no namespace — they exist outside any
namespace boundary:

```
namespace: tenant-a    <- tenant A's namespace
namespace: tenant-b    <- tenant B's namespace

xdatabases.platform.io/tenant-a-db   <- cluster-scoped, namespace = ""
xdatabases.platform.io/tenant-b-db   <- cluster-scoped, namespace = ""
```

Namespace-per-tenant provides zero Crossplane isolation benefit while provider MRs remain
cluster-scoped. The API-server label filtering approach is the correct interim strategy.

See `ucp-tenant-isolation-design.md` for the path to namespace-per-tenant.
