---
title: "Multi-Project GCP — Design"
space: UCP
parent_page_id: "../multi-project-gcp.md"
---

# Multi-Project GCP — Design

---

## Current State

The current credential upload flow (`POST /api/v1/settings/credentials`) has
two gaps:

**1. Structural-only validation.** `validateGCPCredentials()` checks that
required fields exist and that `client_email` looks like a service account
address. It does not call any GCP API, so a valid-looking but revoked, expired,
or permission-less key passes the check and only fails later when Crossplane
attempts to use it.

**2. No environment isolation.** A `tenant-admin` can upload production GCP
credentials into a development UCP instance and provision production resources
through it. GCP does not know or care which UCP environment the call comes from —
it only validates the JWT signature. OC/network-level environment separation does
not extend to GCP project access.

**3. One project per tenant.** The secret and ProviderConfig are named using
`(tenant_id, env)` only. Uploading a second GCP project for the same tenant
would overwrite the first.

---

## Live Credential Validation

When credentials are uploaded, UCP performs a live verification:

```
1. Build a signed JWT using the private_key from the service account JSON
   (RS256, sub = client_email, aud = https://oauth2.googleapis.com/token)

2. POST https://oauth2.googleapis.com/token
   grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   assertion=<signed JWT>
   → obtain a short-lived access token

3. GET https://cloudresourcemanager.googleapis.com/v1/projects/{project_id}
   Authorization: Bearer <access token>
   → proves: key is valid, SA has at least resourcemanager.projects.get
```

If step 2 fails → key is syntactically correct but revoked or expired.
If step 3 fails → key works but the SA lacks the minimum required permissions.

This replaces the current `validateGCPCredentials()` structural check with an
end-to-end live check. The Cloud Resource Manager response also returns the
project's labels, which are used for environment isolation (see below).

---

## Environment Isolation via GCP Project Labels

To prevent cross-environment credential misuse, UCP requires the GCP project
to carry a label that matches the UCP environment the credentials are being
uploaded to:

| UCP environment (`X-Environment`) | Required GCP project label |
|---|---|
| QA | `ucp-env: qa` |
| Production | `ucp-env: production` |

The label is checked from the Cloud Resource Manager response returned in step 3
of the live validation. If the label is absent or does not match, the upload is
rejected.

```
Upload GCP credentials for project "coupon-prod-gcp" to QA UCP
  → live validation succeeds
  → Cloud Resource Manager returns: labels: { "ucp-env": "production" }
  → UCP environment is QA
  → mismatch → 400 Bad Request: "project is registered for production, not qa"
```

This requires the GCP project owner to explicitly set the label on the GCP
project side before credentials can be registered in UCP, proving that they
both control the project and intend it for that environment.

---

## Multi-Project Storage

To support multiple GCP projects per tenant, the secret name and ProviderConfig
name are extended to include the GCP project ID:

**Secret naming:**
```
gcp-credentials-{sanitized-tenant-id}-{sanitized-project-id}-{env}
```
Example: `gcp-credentials-rns-roc-iam--clsd-ucp-coupon-prod-gcp-qa`

**ProviderConfig naming:**
```
gcp-{sanitized-tenant-id}-{sanitized-project-id}-{env}
```
Example: `gcp-rns-roc-iam--clsd-ucp-coupon-prod-gcp-qa`

The `project_id` is already present in the service account JSON — no extra
input required from the user. The sanitization replaces non-alphanumeric
characters with `-`.

---

## Project Selection at Resource Creation

When a tenant has multiple registered GCP projects, the user must explicitly
select which project to provision into. The create form fetches the list of
registered ProviderConfigs for the current tenant and renders a selector:

```
GET /api/v1/settings/credentials/gcp?tenantId={tenantRNS}

Response:
{
  "projects": [
    { "projectId": "coupon-prod-gcp", "providerConfig": "gcp-...-coupon-prod-gcp-qa", "clientEmail": "..." },
    { "projectId": "coupon-shared-gcp", "providerConfig": "gcp-...-coupon-shared-gcp-qa", "clientEmail": "..." }
  ]
}
```

The selected `providerConfig` value is sent in the create request body and
injected into the XR `spec.parameters.providerConfig` field. Crossplane patches
this through to each composed MR's `spec.providerConfigRef.name`.

If only one project is registered, the selector defaults to it (backwards
compatible with the current single-project behaviour).

---

## OC → UCP Role Mapping

No changes required. Multi-project is a credential and provisioning concern,
not a role concern. The existing per-tenant role model applies regardless of
how many GCP projects the tenant has.

---

## Open Questions

### 1. Who sets the GCP project label?

The label (`ucp-env: qa`) must be set by someone with `resourcemanager.projects.update`
permission on the GCP project. In practice this is the team's GCP admin or the
TAM team at project creation time. UCP cannot set it — UCP only reads it.

This creates a dependency on the GCP project setup process. Should UCP's
onboarding documentation include instructions for setting this label, or should
there be a self-service way for the tenant-admin to trigger label setting via
the GCP console?

### 2. Sanitization collisions

Two different project IDs could produce the same sanitized string (e.g.
`my-project-1` and `my.project.1` both become `my-project-1`). A collision
check should be added when registering a new project.

### 3. Default project concept

Should UCP support a "default project" per tenant so that users who don't
care about multi-project don't need to see a selector? The first registered
project could be marked as default.
