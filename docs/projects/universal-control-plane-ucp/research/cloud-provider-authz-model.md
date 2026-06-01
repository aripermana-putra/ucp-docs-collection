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

Three options are evaluated below.

---

## Option 1 — Single Admin Service Account per Provider per Tenant

The tenant creates one service account with broad admin access to all cloud services UCP supports (DB, VM, Storage, GKE, Load Balancer, etc.) and uploads it once.

UCP uses this single credential for all provisioning, management, and monitoring operations for that tenant on that provider.

### Quantitative

| Metric | Value |
|---|---|
| Credentials to manage per tenant per provider | 1 |
| UCP schema changes required | Minimal — current provider config model already supports this |
| Implementation effort | Low — no routing logic needed |
| Blast radius on credential compromise | Entire GCP project for that tenant |
| GCP audit log attribution | All operations attributed to one SA — no per-operation-type traceability |

### Qualitative

**Pros:**
- Simplest possible operational setup for the tenant-admin — create one SA, upload once
- Simplest UCP implementation — no routing, no per-resource-type credential selection
- Works identically across GCP, AWS, Azure (each has an equivalent to "one credential with broad access")
- Easy to reason about: UCP either has access or it doesn't

**Cons:**
- Violates least privilege — the SA can do far more than UCP exposes. A UCP developer role cannot provision a VM, but the underlying SA credential can; if it leaks, the restriction disappears
- High blast radius — a compromised or leaked credential exposes the entire GCP project
- No cloud-native audit trail — GCP Cloud Audit Logs shows the SA performed the action, not which UCP user triggered it
- Over-privileged credentials are the primary attack vector for lateral movement in cloud environments (GCP's own best practice guidance explicitly warns against this)

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
- Clear audit trail (which SA = which resource type)

**Cons:**
- High operational burden on tenant-admin — must create and maintain N SAs
- Tight coupling between UCP's internal resource model and GCP's IAM model
- Does not generalize cleanly: AWS uses IAM policies, Azure uses role assignments on service principals — the per-resource-type model maps differently on each
- UCP must know which operations map to which SA. Adding a new resource type in UCP requires a new SA category and credential upload from the tenant
- Complex to implement, complex to reason about, complex to support

**Sub-variant B:**

**Pros:**
- Aligns with UCP's existing role model
- Smaller credential surface than Sub-variant A
- Role concept is more portable than resource-type concept across providers

**Cons:**
- Still requires tenants to create and manage multiple SAs
- If a developer SA is compromised, it has GCP-level permissions for everything a developer can do — that scope is defined by GCP IAM, not UCP, so the boundary is less clean than it appears
- Requires UI for tenant-admin to upload and map multiple credentials
- Adds schema complexity and routing logic that grows as roles evolve
- Adding a new UCP role means a new SA category and a new credential management burden

---

## Option 3 — Single Service Account, UCP as the Security Boundary

The tenant creates one service account with sufficient scope for all UCP operations and uploads it. UCP enforces all access control using its own permission model (`developer`, `tenant-admin`, `platform-admin`). The cloud provider credential is opaque to the end user — they interact only with UCP.

This is the current PoC approach.

### Quantitative

| Metric | Value |
|---|---|
| Credentials to manage per tenant per provider | 1 |
| UCP schema changes required | None — current model |
| Implementation effort | None additional — already implemented |
| Blast radius on credential compromise (key exfiltration from UCP) | Entire scope of the SA |
| Blast radius on UCP auth bypass | Same as compromised SA |
| Portability across cloud providers | High — same model regardless of provider |
| Audit trail in UCP | Full — every action recorded with UCP user identity |

### Qualitative

**Pros:**
- Uniform model across all cloud providers — implementation is identical for GCP, AWS, Azure, Omnia
- Zero additional operational burden on tenant-admin beyond uploading one credential
- UCP is the single pane of glass — all access control decisions are in one place, auditable in one place
- Aligns with UCP's design goal: UCP abstracts away provider differences, including authz
- Simpler to implement, maintain, and evolve
- Adding new resource types or new UCP roles requires no changes to credential management

**Cons:**
- UCP becomes a high-value target: the SA credential stored in UCP grants cloud access to everything UCP supports for that tenant; if stolen, the attacker bypasses UCP's authz entirely
- Violates least privilege at the cloud provider level — the SA is more powerful than any individual user's UCP role
- Anyone with direct GCP access (outside UCP) and the right GCP IAM role can modify or delete UCP-managed resources without any UCP enforcement
- No cloud-provider-level audit trail per UCP user (all cloud actions attributed to the SA)

### STRIDE threat analysis (Option 3 specific)

| Threat | Attack scenario | Severity | Mitigation |
|---|---|---|---|
| **Spoofing** | Attacker impersonates a valid UCP session → acts as a legitimate user | High | Strong authn via Keycloak, HTTPS-only, session expiry, CSRF protection |
| **Elevation of Privilege** | UCP developer calls a tenant-admin endpoint by bypassing `RequirePermission` middleware | Critical | Middleware applied at route level, not handler level — all routes require explicit permission declaration |
| **Elevation of Privilege** | SQL injection or auth bypass extracts another tenant's SA credential | Critical | Parameterized queries only, tenant-scoped DB queries, encrypted credential storage |
| **Information Disclosure** | SA key extracted from UCP DB or logs | Critical | Credentials stored encrypted at rest; never logged; access to DB restricted to api-server only |
| **Repudiation** | User claims they did not provision a resource | Medium | Audit log (`audit_logs` table) records every action with user ID, session ID, IP, timestamp |
| **Tampering** | Attacker modifies a provisioned resource directly via GCP console | Low for UCP | Outside UCP's control; drift detection (MCUCP-158) detects this |

---

## Comparison Summary

| Dimension | Option 1 (Single admin SA) | Option 2A (Per resource type) | Option 2B (Per UCP role) | Option 3 (UCP boundary) |
|---|---|---|---|---|
| Credentials per tenant per provider | 1 | N (per resource type) | 2–3 | 1 |
| Operational burden on tenant-admin | Low | High | Medium | Low |
| Implementation complexity | Low | High | Medium | Low (done) |
| Cloud-provider portability | High | Low | Medium | High |
| Least privilege at cloud level | ❌ | ✅ | Partial | ❌ |
| Blast radius on credential leak | Entire project | Per resource type | Per role scope | Entire SA scope |
| UCP audit trail | ✅ | ✅ | ✅ | ✅ |
| Cloud-native audit attribution | ❌ | ✅ | Partial | ❌ |
| Aligns with UCP design philosophy | ✅ | ❌ | Partial | ✅ |
| Works uniformly across GCP/AWS/Azure | ✅ | ❌ | Partial | ✅ |

---

## Analysis and Rationale

**Option 2A is the most secure at the cloud provider level but the least practical** for UCP's scope. It creates tight coupling between UCP's resource model and each provider's IAM model — coupling that must be maintained as UCP adds new resource types and new providers. It also puts significant operational burden on tenant-admins who just want to use UCP as a control plane. It contradicts UCP's purpose.

**Option 2B is a compromise that satisfies neither goal well.** It adds complexity without achieving true least privilege (the SA boundary is still coarser than the UCP role boundary), and it still doesn't generalize cleanly across providers.

**Options 1 and 3 are functionally identical in terms of credentials and implementation** — both use one SA with broad access. The difference is framing: Option 1 is described as the design intent; Option 3 acknowledges explicitly that UCP is the security boundary and owns the risk.

**The right framing is Option 3**, because it forces the design to honestly account for the security posture UCP is taking on. The SA is not "admin by accident" — it is deliberately scoped to everything UCP needs, because UCP is the authz layer. This means UCP's security posture matters more than any individual provider's access control, and the design should invest accordingly in:

1. Encrypted credential storage (already done)
2. Comprehensive `RequirePermission` coverage on all routes (already done)
3. Tenant-scoped DB queries preventing cross-tenant credential access (already done)
4. Audit logging for every state-changing operation (already done)
5. Credential never appearing in logs or API responses (already done)

The threat that remains genuinely open is **SA key exfiltration from the database** — this is the one scenario where UCP's authz provides no protection, because the attacker bypasses UCP entirely. Mitigations beyond encrypted storage (e.g. secrets management via Vault or GCP Secret Manager, credential rotation) are deferred post-PoC.

---

## Recommendation

**Proceed with Option 3.** One service account per provider per tenant, uploaded by the tenant-admin, used exclusively by UCP for all operations. UCP is the authoritative security boundary — cloud provider credentials are an implementation detail, not an access control mechanism.

The per-resource-type or per-role SA models add complexity that does not pay off for a multi-cloud control plane: they create provider-specific coupling, increase operational burden, and do not fully achieve least privilege anyway (the SA boundary is always coarser than UCP's role model).

The security investment belongs in hardening UCP itself — not in distributing authz across multiple SA credentials that each provider handles differently.
