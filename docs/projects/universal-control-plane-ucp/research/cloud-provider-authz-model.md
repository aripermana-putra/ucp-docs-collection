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

### Sub-variants — granular UCP authz on top of a single SA

The single-SA model does not prevent UCP from enforcing finer-grained access control at the UCP layer. Two directions are possible:

**Sub-variant 1A — UCP-native service-level roles**

UCP introduces per-resource-type role assignments on top of its existing 3-role model. A tenant-admin could assign a user `developer` for databases but `viewer` for VMs, independently of their OC service roles.

| Aspect | Detail |
|---|---|
| Credential model | Unchanged — still one SA |
| Authz model | UCP-owned — no dependency on OC service roles |
| Schema impact | New role assignment dimension: `(user, tenant, resource_type, role)` |
| Implementation effort | Medium — extend `tenant_role_assignments`, update `RequirePermission` middleware |
| OC alignment | None — UCP defines its own service-level roles independent of OC |
| Cross-provider | Natural — UCP's resource types already abstract over providers |

Benefit: consistent fine-grained control across all cloud providers using UCP's own model.
Risk: diverges from OC's role model, creating two parallel authorization systems a user must understand.

**Sub-variant 1B — OC service roles applied cross-provider**

UCP reads the user's OC service roles (already collected from JWT into `oc_roles.oc_service_roles`) and maps them to UCP resource types. An OC `dbaas:admin` grants database provisioning rights in UCP regardless of provider — Omnia DBaaS, GCP Cloud SQL, AWS RDS are all treated as `database` resources.

Concrete mapping:

| OC service role | UCP resource type | Permission granted |
|---|---|---|
| `dbaas:admin` or `dbaas:operator` | `database` | `PermProvision` |
| `dbaas:viewer` | `database` | `PermRead` |
| `caas:admin` or `caas:edit` | `kubernetes` | `PermProvision` |
| `caas:view` | `kubernetes` | `PermRead` |
| `computeapi:admin` | `compute` | `PermProvision` |
| `staas:admin` | `storage` | `PermProvision` |
| `lbaas:lbaas-operator` | `loadbalancer` | `PermProvision` |

| Aspect | Detail |
|---|---|
| Credential model | Unchanged — still one SA |
| Authz model | Derived from OC — no new role management UI needed |
| Schema impact | None — `oc_service_roles` JSONB already populated from JWT |
| Implementation effort | Low — middleware reads `oc_roles`, applies mapping at provisioning time |
| OC alignment | High — UCP reflects the user's OC standing without duplicating it |
| Cross-provider | Yes — the OC service role maps to a UCP resource type, not a specific provider |

Benefit: zero additional role management in UCP; OC admins stay in control; cross-provider consistency comes for free from the OC role model.
Risk: UCP's resource scope is constrained by OC's service taxonomy. GCP-only resources with no OC equivalent (e.g. GKE, GCE) require a fallback rule — likely tenant-level UCP role only.

Both sub-variants are compatible: 1A and 1B could be layered (OC service roles as the floor, UCP role assignments as overrides). The choice depends on how tightly UCP should track OC's authz model.

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

| Dimension | Option 1 (Single SA) | Option 1A (UCP service roles) | Option 1B (OC service roles) | Option 2A (SA per resource) | Option 2B (SA per role) |
|---|---|---|---|---|---|
| Credentials per tenant per provider | 1 | 1 | 1 | N | 2–3 |
| Operational burden on tenant-admin | Low | Low | Low | High | Medium |
| Implementation complexity | Low | Medium | Low | High | Medium |
| Granularity of UCP authz | Tenant-role only | Per resource type | OC service role | Per resource type | Per UCP role |
| OC role model alignment | Partial | None | High | None | None |
| Cross-provider consistency | ✅ | ✅ | ✅ | ❌ | Partial |
| Least privilege at cloud level | ❌ | ❌ | ❌ | ✅ | Partial |
| Blast radius on credential leak | Entire SA scope | Entire SA scope | Entire SA scope | Per resource type | Per role scope |
| UCP audit trail | ✅ | ✅ | ✅ | ✅ | ✅ |
| Aligns with UCP design philosophy | ✅ | ✅ | ✅ | ❌ | Partial |
| New role management UI needed | No | Yes | No | Yes | Yes |

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

**Option 2A and 2B** distribute authz across multiple credentials at the cloud provider level. This does not pay off for a multi-cloud control plane: it creates provider-specific coupling, increases operational burden, and does not fully achieve least privilege (the SA boundary is always coarser than UCP's role model). They contradict UCP's purpose.

**Option 1 (base)** is the right foundation. One SA per provider per tenant, UCP as the security boundary. The credential model stays simple and portable; security investment goes into hardening UCP.

**Option 1A and 1B** are enhancements to the UCP authz layer on top of the same single-SA credential model:

- **1B (OC service roles)** is the lowest-friction path to finer-grained control — the data is already collected from the JWT, no new role management UI is needed, and the mapping aligns with the existing OC role model tenants already understand. The only constraint is that GCP-only resources with no OC service equivalent need a fallback rule.
- **1A (UCP-native service roles)** gives UCP full control over its own authz model, independent of OC. More flexible long-term, but requires building a new role management layer that tenants must learn separately from OC.

The base Option 1 (tenant-level roles only) is what the PoC implements and is sufficient for MVP scope. 1B is the natural next step if finer-grained control is needed — it costs little to implement and keeps OC as the source of truth for roles. 1A is a longer-term option if UCP needs to diverge from OC's service taxonomy.

The threat that remains genuinely open regardless of sub-variant is **SA key exfiltration from the database** — the one scenario where UCP's authz provides no protection. Mitigations beyond encrypted storage (secrets management via Vault or GCP Secret Manager, credential rotation) are deferred post-PoC.

---

## Recommendation

**Option 1 for PoC and MVP.** One service account per provider per tenant, UCP as the authoritative security boundary.

If finer-grained authz is needed, **Option 1B** (leverage existing OC service roles) is the preferred path — lowest implementation cost, highest OC alignment, and cross-provider by design. The PM decision on whether to enforce OC service roles at provisioning time (referenced as Option 2 in the RBAC POC report) is the gate for this.
