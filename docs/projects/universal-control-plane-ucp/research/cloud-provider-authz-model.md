---
title: "Cloud Provider Authorization — Service Account Strategy"
space: UCP
---

# Cloud Provider Authorization — Service Account Strategy

| | |
|---|---|
| **Author** | aripermana.putra |
| **Date** | 2026-06-01 |
| **Purpose** | Evaluate how UCP should handle cloud provider credentials and authorization |

---

## Context and Constraints

UCP's identity and access anchor is **OC Core Data / Keycloak**. Authentication (who the user is) and tenant/member management are handled entirely through this channel. UCP does not manage its own IdP and does not want to replicate OC's authorization model.

The challenge is the gap between **UCP-level authorization** (what a user can do in UCP) and **cloud provider-level authorization** (what credentials UCP uses to act on the cloud on the user's behalf). Every supported cloud provider has its own authorization mechanism:

| Provider | Machine identity mechanism |
|---|---|
| GCP | Service Account + key file or Workload Identity |
| AWS | IAM User access key, or IAM Role via STS |
| Azure | Service Principal + client secret, or Managed Identity |
| OC (Omnia) | ROC service account token |

UCP cannot federate into each provider's native authz system using Keycloak JWTs — GCP Workload Identity Federation could technically do this, but it requires per-project GCP configuration and does not generalize across AWS and Azure. This is not UCP's job.

The question is therefore: **what credential(s) does the tenant upload into UCP, and how does UCP use them?**

Two options are evaluated below.

---

## Option 1 — Single Service Account, UCP as the Security Boundary

The tenant creates one service account with sufficient access for all cloud services UCP supports (DB, VM, Storage, GKE, Load Balancer, etc.) and uploads it once.

UCP uses this single credential for all provisioning, management, and monitoring operations for that tenant on that provider. UCP is the authoritative authz layer — the SA credential is an implementation detail, not an access control mechanism.

### Quantitative

| Metric | Value |
|---|---|
| Credentials to manage per tenant per provider | 1 |
| UCP schema changes required | None — current provider config model already supports this |
| Implementation effort | None additional — already implemented in PoC |
| Blast radius on credential compromise (key exfiltration) | Entire SA scope for that tenant |
| Blast radius on UCP auth bypass | Same as compromised SA |
| Cloud-native audit attribution | All operations attributed to one SA — no per-UCP-user traceability at provider level |
| UCP audit trail | Full — every action recorded with UCP user identity, session, IP, timestamp |
| Portability across cloud providers | High — same model regardless of provider |

### Qualitative

**Pros:**
- Simplest operational setup for the tenant-admin — create one SA, upload once
- No routing logic in UCP — no per-resource-type or per-role credential selection
- Works identically across GCP, AWS, Azure, Omnia
- UCP is the single pane of glass — all authz decisions in one place, auditable in one place
- Adding new resource types or new UCP roles requires no changes to credential management
- Aligns with UCP's design goal: abstract away provider differences including authz

**Cons:**
- SA is more powerful than any individual user's UCP role — a developer whose UCP role blocks VM provisioning still triggers VM operations via the SA
- If the SA key is exfiltrated from UCP, the attacker bypasses UCP's authz entirely — no UCP enforcement applies to direct cloud API access
- No cloud-provider-level audit attribution per UCP user
- Anyone with direct cloud access (outside UCP) can modify UCP-managed resources without UCP enforcement

---

## Option 2 — Multiple Service Accounts, Role-Mapped

The tenant creates multiple service accounts with different scopes. UCP maps each service account to a UCP role or resource type.

**Sub-variant A — mapped by resource type:** tenant creates a DB SA, a VM SA, a storage SA, etc. UCP routes each operation to the appropriate SA.

**Sub-variant B — mapped by UCP role:** tenant creates a developer SA (provision only) and a tenant-admin SA (provision + manage + monitor). UCP uses the matching SA for the user's role.

### Quantitative

| Metric | Sub-variant A (per resource type) | Sub-variant B (per UCP role) |
|---|---|---|
| Credentials to manage per tenant per provider | N (one per resource type UCP supports) | 2–3 (one per UCP role) |
| UCP schema changes required | Significant — SA-to-resource-type mapping table | Moderate — SA-to-role mapping table |
| Implementation effort | High — routing logic per handler, per provider | Medium — route by resolved role |
| Blast radius on credential compromise | Limited to that resource type | Limited to that role's scope |
| Portability across cloud providers | Low — each provider has different resource types | Medium — role concept is provider-agnostic |

### Qualitative

**Sub-variant A:**

**Pros:**
- Closest to least privilege at the cloud provider level
- Limited blast radius per credential
- Clear cloud-native audit trail (which SA = which resource type)

**Cons:**
- High operational burden on tenant-admin — must create and maintain N SAs
- Tight coupling between UCP's internal resource model and each provider's IAM model
- Does not generalize cleanly across providers — AWS uses IAM policies, Azure uses role assignments on service principals; per-resource-type mapping differs on each
- Adding a new resource type to UCP requires a new SA category and credential upload from the tenant
- Complex to implement, complex to reason about, complex to support

**Sub-variant B:**

**Pros:**
- Aligns with UCP's existing role model
- Smaller credential surface than Sub-variant A
- Role concept is more portable than resource-type concept across providers

**Cons:**
- Still requires tenants to create and manage multiple SAs
- Blast radius is defined by GCP IAM scope of the SA, not by UCP's role model — the boundary is less clean than it appears
- Requires UI for tenant-admin to upload and map multiple credentials
- Adds schema complexity and routing logic that grows as roles evolve
- Adding a new UCP role means a new SA category and credential management burden for tenants

---

## Comparison Summary

| Dimension | Option 1 (Single SA, UCP boundary) | Option 2A (Per resource type) | Option 2B (Per UCP role) |
|---|---|---|---|
| Credentials per tenant per provider | 1 | N | 2–3 |
| Operational burden on tenant-admin | Low | High | Medium |
| Implementation complexity | Low | High | Medium |
| Cloud-provider portability | High | Low | Medium |
| Least privilege at cloud level | ❌ | ✅ | Partial |
| Blast radius on credential leak | Entire SA scope | Per resource type | Per role scope |
| UCP audit trail | ✅ | ✅ | ✅ |
| Cloud-native audit attribution per user | ❌ | ✅ | Partial |
| Aligns with UCP design philosophy | ✅ | ❌ | Partial |
| Works uniformly across GCP/AWS/Azure | ✅ | ❌ | Partial |

---

## STRIDE Threat Analysis (Option 1)

Since Option 1 makes UCP the security boundary, the threat model focuses on UCP itself.

| Threat | Attack scenario | Severity | Mitigation |
|---|---|---|---|
| **Spoofing** | Attacker impersonates a valid UCP session → acts as a legitimate user | High | Strong authn via Keycloak, HTTPS-only, session expiry, CSRF protection |
| **Elevation of Privilege** | UCP developer calls a tenant-admin endpoint by bypassing `RequirePermission` middleware | Critical | Middleware applied at route level — all routes require explicit permission declaration |
| **Elevation of Privilege** | SQL injection or auth bypass extracts another tenant's SA credential | Critical | Parameterized queries only, tenant-scoped DB queries, encrypted credential storage |
| **Information Disclosure** | SA key extracted from UCP DB or logs | Critical | Credentials stored encrypted at rest; never logged; DB access restricted to api-server only |
| **Repudiation** | User claims they did not provision a resource | Medium | Audit log records every action with user ID, session ID, IP, timestamp |
| **Tampering** | Attacker modifies a provisioned resource directly via cloud console | Low for UCP | Outside UCP's control; drift detection (MCUCP-158) detects this |

---

## Analysis and Rationale

**Option 2A** is the most secure at the cloud provider level but the least practical for UCP's scope. It creates tight coupling between UCP's resource model and each provider's IAM model, puts significant operational burden on tenants, and contradicts UCP's purpose as a unified control plane.

**Option 2B** adds complexity without achieving true least privilege — the SA boundary is still coarser than UCP's role model, and the model still doesn't generalize cleanly across providers.

**Option 1** is the right choice for UCP's design philosophy. The SA is not over-privileged by accident — it is deliberately scoped to everything UCP needs, because UCP is the authz layer. This means UCP's security posture matters more than any individual provider's access control. The security investment belongs in hardening UCP itself:

1. Encrypted credential storage
2. Comprehensive permission enforcement on all routes
3. Tenant-scoped DB queries preventing cross-tenant credential access
4. Audit logging for every state-changing operation
5. Credentials never appearing in logs or API responses

The threat that remains genuinely open is **SA key exfiltration from the database** — this is the one scenario where UCP's authz provides no protection. Mitigations beyond encrypted storage (secrets management via Vault or GCP Secret Manager, credential rotation policies) are deferred post-PoC.

---

## Recommendation

**Option 1.** One service account per provider per tenant, uploaded by the tenant-admin, used exclusively by UCP for all operations. UCP is the authoritative security boundary — cloud provider credentials are an implementation detail, not an access control mechanism.

The multi-SA models add complexity that does not pay off for a multi-cloud control plane: they create provider-specific coupling, increase operational burden, and do not fully achieve least privilege anyway.
