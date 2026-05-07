---
title: "Tenant Isolation"
space: UCP
parent_page_id: "../pocs.md"
---

# Tenant Isolation

## Background

MCUCP-119 attempted to migrate all Crossplane XRDs, XRs, and MRs from `scope: Cluster` to
`scope: Namespaced`, with the goal of using one Kubernetes namespace per tenant as the
isolation boundary. This was halted because Crossplane v2 enforces a hard constraint:

> A namespace-scoped XR can only compose resources in its own namespace.

All managed resources (MRs) from the providers we need to support are still cluster-scoped:

| Provider | MR Scope | Namespaced Support |
|---|---|---|
| provider-upjet-gcp | Cluster | Not yet (upstream in progress) |
| provider-upjet-aws | Namespace | Done in v2 |
| provider-upjet-azure | Cluster | Not yet (upstream in progress) |
| provider-roc (Omnia) | Cluster | Not yet (we own this — unblocked) |
| provider-terraform | Cluster | Not yet |

This document records the investigation and decision on how to achieve tenant isolation in
the interim, until all providers support namespaced MRs.

---

## Why Namespace Per Tenant Does Not Work Today

Creating a Kubernetes namespace per tenant only provides isolation if the resources
actually live inside that namespace. Cluster-scoped resources (XRs, MRs) exist outside
any namespace — they float at the cluster level with `namespace: ""`.

```
namespace: team-a   <- tenant A's namespace
namespace: team-b   <- tenant B's namespace

xdatabases.platform.io/team-a-db   <- cluster-scoped, namespace = ""
xdatabases.platform.io/team-b-db   <- cluster-scoped, namespace = ""
```

Kubernetes RBAC cannot say "only allow team-a to see resources in namespace team-a" for
resources that are not in any namespace. The namespaces are irrelevant for Crossplane
isolation as long as XRDs and provider MRs remain cluster-scoped.

**Conclusion:** namespace-per-tenant is the correct long-term model (MCUCP-119), but it
provides zero Crossplane isolation benefit today.

---

## Existing Auth and RBAC Infrastructure

Before deciding on an approach, it is important to understand what isolation mechanisms
are already in place.

### Identity Provider

Rakuten Keycloak (`accounts-onecloud.rakuten-it.com`) is the OIDC IdP. Two auth paths:

- **Browser:** BFF pattern — opaque session cookie, tokens never reach the browser
- **CLI:** PKCE flow, Bearer JWT on every request

### Tenant Identity

Tenant context is passed as an `X-Tenant-ID` header using Rakuten RNS format
(e.g. `rns:roc:iam::clsd-ucp`). The API server validates tenant admin status by calling
the Horizon API (`isUserTenantAdmin()`).

### The Critical Isolation Point

Tenants never have direct access to the Kubernetes API. The only access path is:

```
Tenant -> Keycloak auth -> API Server -> Temporal Workflow -> K8s/Crossplane -> Cloud Provider
```

The API server's `ServiceAccount` is the sole Kubernetes actor. This means the **API
server is already the primary isolation boundary** — Crossplane-level namespace scoping
is defense-in-depth, not the primary control.

### RBAC Design (Designed, Not Fully Implemented)

A 5-role per-tenant RBAC model is already designed (`docs/architecture/RBAC.md`):

| Role | Scope | Description |
|---|---|---|
| `platform-admin` | Platform | UCP operators |
| `tenant-admin` | Tenant | Full access within tenant |
| `deployer` | Tenant | Provision and delete resources |
| `approver` | Tenant | Approve/reject Temporal workflows |
| `viewer` | Tenant | Read-only |

Currently only `isUserTenantAdmin()` is live. The full permission middleware is designed
but not yet implemented.

### Resource Labeling (Designed, Not Applied)

All UCP-provisioned resources are designed to carry ownership labels:

```yaml
metadata:
  labels:
    ucp.platform/managed: "true"
    ucp.platform/tenant: "rns:roc:iam::clsd-ucp"
    ucp.platform/created-by: "<userID>"
    ucp.platform/workflow-id: "<id>"
```

This is defined in `RBAC.md §6` but not yet applied at XR creation time.

---

## Options Considered

### Option A — ProviderConfig Per Tenant (Selected for Interim)

Each tenant gets one `ProviderConfig` per provider, pointing to their dedicated cloud
account or project:

```
tenant rns:roc:iam::clsd-ucp ->
  ProviderConfig: clsd-ucp-gcp    -> GCP project:        clsd-ucp-prod
  ProviderConfig: clsd-ucp-aws    -> AWS account:         123456789
  ProviderConfig: clsd-ucp-azure  -> Azure subscription:  xxxxxxxx
  ProviderConfig: clsd-ucp-omnia  -> Omnia tenant:        clsd-ucp
```

When the API server creates an XR for a tenant, it injects the correct `providerConfig`
reference. Tenants interact only through the API — they cannot create XRs directly or
reference another tenant's ProviderConfig.

The API server `ServiceAccount` already has permission to manage ProviderConfigs
(`k8s/api-server/serviceaccount.yaml`) — this was anticipated in the existing design.

**Pros:** Works today across all 4 providers. Real cloud-level isolation. No new tooling.

**Cons:** Cluster-scoped XRs globally visible to cluster-admins (acceptable — tenants
have no direct cluster access).

### Option B — vCluster Per Tenant

Each tenant gets a virtual Kubernetes cluster with its own Crossplane installation.

**Pros:** True API-server-level isolation — tenants cannot see each other at all.

**Cons:** One Crossplane install per tenant. High operational overhead. Overkill for an
internal platform serving known teams.

### Option C — Namespace Per Tenant with Namespace-Scoped XRDs (Future)

Resume MCUCP-119 once all required providers ship namespaced MR support. This is the
correct long-term model.

**Blocked on:** provider-upjet-gcp, provider-upjet-azure, provider-roc, provider-terraform.

Note: `provider-roc` is owned by the platform team — making `OmniaDatabase`
namespace-scoped is unblocked today and can be done independently.

---

## Current Implementation State

The ProviderConfig-per-tenant approach is already partially live. When a tenant admin
uploads cloud credentials via the UI, the API server automatically creates the
corresponding ProviderConfig.

### Credential Upload Flow

```
Tenant admin uploads credential file via UI (CloudCredentials.jsx)
  -> POST /api/v1/settings/credentials
  -> API server validates file format
  -> Stores as K8s Secret in crossplane-system:
       cloud-credentials-{provider}-{sanitized-tenant-id}-{env}
       labels:
         platform.ucp.io/type: cloud-credentials
         platform.ucp.io/provider: gcp
         platform.ucp.io/env: production
         platform.ucp.io/tenant: rns-roc-iam--clsd-ucp
  -> Automatically creates/updates ProviderConfig:
       gcp-{sanitized-tenant-id}-{env}
       spec.projectID: <from service account JSON>
       spec.credentials.secretRef: <points to above secret>
```

Only tenant admins (verified via Horizon API) can upload credentials. The raw key
material is never returned via any API endpoint — only metadata (project ID, client
email, key ID) is surfaced back to the UI.

### Current State Per Provider

| Provider | Credential UI | K8s Secret | ProviderConfig Auto-Created |
|---|---|---|---|
| GCP | Yes | Yes (service account JSON) | Yes — `upsertGCPProviderConfig()` |
| ROC/Omnia | Yes | Yes (Ed25519 JWK) | No — uses on-demand JWT generation instead |
| AWS | No | No | No |
| Azure | No | No | No |

### ROC/Omnia Auth Difference

Omnia uses a different auth pattern. Instead of a long-lived ProviderConfig credential,
the API server stores the tenant's Ed25519 JWK private key and generates a fresh
short-lived JWT (10-minute TTL) each time Omnia needs to be called:

```
K8s Secret (JWK private key)
  -> generateROCTokenForTenant()
  -> signs JWT with Ed25519 key
  -> JWT passed to Temporal workflow -> Omnia API call
```

This matches Omnia's requirement for short-lived tokens and avoids the token expiry
problems that come with storing long-lived credentials.

---

## Isolation Layers with Option A

| Layer | Mechanism | Status |
|---|---|---|
| Authentication | Keycloak OIDC | Implemented |
| Tenant identity | `X-Tenant-ID` + Horizon API | Implemented |
| API-level RBAC | 5-role permission model | Designed, not implemented |
| Resource ownership | `ucp.platform/tenant` labels + API-layer filtering | Designed, not applied |
| Cloud isolation (GCP) | ProviderConfig per tenant — auto-created on credential upload | Implemented |
| Cloud isolation (Omnia) | Per-tenant JWK secret + on-demand JWT generation | Implemented |
| Cloud isolation (AWS/Azure) | ProviderConfig per tenant | Not yet — credential UI needed |
| K8s defense-in-depth | `ValidatingAdmissionPolicy` (native k8s 1.35, no extra tooling) | Not done |

---

## Recommended Path

| Timeframe | Action | Status |
|---|---|---|
| Now | ProviderConfig per tenant for GCP — auto-created on credential upload | Done |
| Now | Per-tenant JWK + JWT generation for Omnia | Done |
| Now | Add credential UI + ProviderConfig provisioning for AWS and Azure | Not done |
| Now | Apply `ucp.platform/tenant` labels on all XR creation | Not done |
| Now | Implement RBAC.md role model (replace `isUserTenantAdmin()`) | Not done |
| Now | Add `ValidatingAdmissionPolicy` for tenant label + ProviderConfig enforcement | Not done |
| Short-term | Change `provider-roc` OmniaDatabase to `scope=Namespaced` | Not done |
| Long-term | Resume MCUCP-119 when provider-upjet-gcp and provider-upjet-azure ship namespaced MR support | Blocked on upstream |

---

## Related

- `MCUCP-119` — namespace-scoped XRDs branch (on hold)
- `.local-docs/MCUCP-119-namespace-scoped-xrds.md` — detailed blocker analysis
- `docs/architecture/RBAC.md` — full RBAC role and permission design
- `docs/architecture/BFF_AUTH.md` — authentication architecture
- `.local-docs/AUTH_FLOW.md` — end-to-end auth flow reference
