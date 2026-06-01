---
title: "Cloud Provider Authorization — Service Account Strategy"
space: UCP
---

# Cloud Provider Authorization — Service Account Strategy

| | |
|---|---|
| **Author** | aripermana.putra |
| **Date** | 2026-06-01 |
| **Purpose** | Evaluate how UCP should handle cloud provider credentials and authorization for MVP |

---

## Context and Constraints

UCP's identity and access anchor is **OC Core Data / Keycloak**. Authentication (who the user is) and tenant/member management are handled entirely through this channel. UCP does not manage its own IdP and does not want to replicate OC's authorization model.

The challenge is the gap between **UCP-level authorization** (what a user can do in UCP) and **cloud provider-level authorization** (what credentials UCP uses to act on the cloud on the user's behalf). Every supported cloud provider has its own authorization mechanism:

| Provider | Machine identity mechanism |
|---|---|
| GCP | Service Account + key file or Workload Identity |
| AWS | IAM User access key, or IAM Role via STS |
| Azure | Service Principal + client secret, or Managed Identity |
| OC | ROC service account token |

UCP cannot federate into each provider's native authz system using Keycloak JWTs — GCP Workload Identity Federation could technically do this, but it requires per-project GCP configuration and does not generalize across AWS and Azure. This is not UCP's job.

The question is therefore: **what credential(s) does the tenant upload into UCP, and how does UCP use them?**

---

## Option 1 — Single Service Account, UCP as the Security Boundary

The tenant creates one service account with sufficient access for all cloud services UCP supports (DB, VM, Storage, GKE, Load Balancer, etc.) and uploads it once per provider.

UCP uses this single credential for all provisioning, management, and monitoring operations for that tenant on that provider. UCP is the authoritative authz layer — the SA credential is an implementation detail, not an access control mechanism.

### Quantitative

| Metric | Value |
|---|---|
| Credentials to manage per tenant per provider | 1 |
| Backend components to build | Credential upload API, encrypted storage (1 table), provider config reference in provisioning handlers, route-level permission middleware, audit logging |
| CLI components to build | `ucp credentials set <provider>` command to upload SA key |
| Blast radius on credential exfiltration | Entire SA scope for that tenant |
| Blast radius on UCP auth bypass | Same as exfiltrated SA |
| Cloud-native audit attribution | All operations attributed to one SA — no per-UCP-user traceability at provider level |
| UCP audit trail | Full — every action recorded with UCP user identity, session, IP, timestamp |
| Portability across cloud providers | High — same model regardless of provider |

### Qualitative

**Pros:**
- Simplest operational setup for the tenant-admin — create one SA, upload once per provider
- No routing logic — no per-resource-type or per-role credential selection
- Works identically across GCP, AWS, Azure, Omnia
- UCP is the single pane of glass — all authz decisions in one place, auditable in one place
- Adding new resource types or new UCP roles requires no changes to credential management
- Aligns with UCP's design goal: abstract away provider differences including authz

**Cons:**
- SA is more powerful than any individual user's UCP role — a developer whose UCP role blocks VM provisioning still triggers VM operations via the SA
- If the SA key is exfiltrated from UCP, the attacker bypasses UCP's authz entirely
- No cloud-provider-level audit attribution per UCP user
- Anyone with direct cloud access outside UCP can modify UCP-managed resources without UCP enforcement

---

### Sub-variant 1A — UCP-native service-level roles

UCP introduces per-resource-type role assignments on top of its tenant-level 3-role model. A tenant-admin could assign a user `developer` for databases but `viewer` for VMs, independently of their OC service roles.

| Metric | Value |
|---|---|
| Additional backend components | Extended role assignment table `(user, tenant, resource_type, role)`; role management API per resource type; updated permission middleware to resolve resource type from request context |
| Additional CLI components | `ucp role assign --resource-type database <user>` and equivalent commands per resource type |
| Authz model | UCP-owned — no dependency on OC service roles |
| OC alignment | None — UCP defines its own service-level roles independent of OC |
| Cross-provider consistency | Yes — UCP resource types abstract over providers |

**Benefit:** consistent fine-grained control across all cloud providers using UCP's own model.

**Risk:** diverges from OC's role model, creating two parallel authorization systems a user must understand. Role management becomes a UCP-specific task on top of OC membership management.

---

### Sub-variant 1B — OC service roles applied cross-provider

UCP reads the user's OC service roles from the Keycloak JWT `groups` claim and maps them to UCP resource types. An OC `dbaas:admin` grants database provisioning rights in UCP regardless of provider — ROC DBaaS, GCP Cloud SQL, and AWS RDS are all treated as `database` resources.

| Metric | Value |
|---|---|
| Additional backend components | JWT groups parsing at login (oc_roles table); OC service → UCP resource type mapping (config); permission middleware reads oc_roles and applies mapping |
| Additional CLI components | None — role assignment is automatic from OC; no new CLI commands needed |
| Authz model | Derived from OC — follows the user's existing OC standing |
| OC alignment | High — UCP reflects OC service roles without duplicating management |
| Cross-provider consistency | Yes — OC service role maps to a UCP resource type, not a specific provider |

Concrete mapping:

| OC service role | UCP resource type | Permission granted |
|---|---|---|
| `dbaas:admin` or `dbaas:operator` | `database` | provision + manage |
| `dbaas:viewer` | `database` | read only |
| `caas:admin` or `caas:edit` | `kubernetes` | provision + manage |
| `caas:view` | `kubernetes` | read only |
| `computeapi:admin` | `compute` | provision + manage |
| `staas:admin` | `storage` | provision + manage |
| `lbaas:lbaas-operator` | `loadbalancer` | provision + manage |

**Benefit:** zero additional role management in UCP; OC admins stay in control; cross-provider consistency comes from the OC role model at no extra cost.

**Risk:** UCP's resource scope is constrained by OC's service taxonomy. GCP-only resources with no OC equivalent (GKE, GCE) need a fallback rule — likely fall back to tenant-level UCP role.

Both sub-variants are compatible and can be layered: OC service roles as the baseline, UCP role assignments as overrides for resources outside OC's taxonomy.

---

## Option 2 — Multiple Service Accounts, Role-Mapped

The tenant creates multiple service accounts with different scopes. UCP maps each service account to a UCP role or resource type.

**Sub-variant 2A — mapped by resource type:** tenant creates a DB SA, a VM SA, a storage SA, etc. UCP routes each operation to the appropriate SA.

**Sub-variant 2B — mapped by UCP role:** tenant creates a developer SA and a tenant-admin SA. UCP uses the matching SA for the user's resolved role.

### Quantitative

| Metric | Sub-variant 2A (per resource type) | Sub-variant 2B (per UCP role) |
|---|---|---|
| Credentials to manage per tenant per provider | N — one per resource type | 2–3 — one per UCP role |
| Additional backend components | Multi-SA storage; SA-to-resource-type routing table; per-handler SA selection logic per provider | Multi-SA storage; SA-to-role routing table; role-based SA selection in middleware |
| Additional CLI components | `ucp credentials set --resource-type database <key>` per type | `ucp credentials set --role developer <key>` per role |
| Blast radius on credential exfiltration | Limited to that resource type's scope | Limited to that role's scope |
| Portability across cloud providers | Low — each provider has different resource type taxonomies | Medium — role concept is more portable |

### Qualitative

**Sub-variant 2A:**

**Pros:**
- Closest to least privilege at the cloud provider level
- Limited blast radius per credential
- Clear cloud-native audit trail (which SA = which resource type)

**Cons:**
- High operational burden on tenant-admin — create and maintain N SAs, one per resource type UCP supports
- Tight coupling between UCP's resource model and each provider's IAM taxonomy
- Does not generalize cleanly across providers — each has its own way of scoping service accounts
- Adding a new resource type to UCP requires a new SA category and a new credential upload from every tenant
- Most complex option to build and maintain

**Sub-variant 2B:**

**Pros:**
- Smaller credential surface than 2A
- Role concept is provider-agnostic

**Cons:**
- Blast radius is defined by GCP IAM scope of the SA, not by UCP's role model — the boundary is less clean than it appears
- Still requires tenants to create and map multiple SAs
- Adding a new UCP role requires a new SA category from every tenant
- SA routing logic grows as the role model evolves

---

## Comparison Summary


| Dimension | Option 1 | Option 1A | Option 1B | Option 2A | Option 2B |
|---|---|---|---|---|---|
| Credentials per tenant per provider | 1 | 1 | 1 | N | 2–3 |
| Backend build effort | Low | Medium | Low–Medium | High | Medium |
| CLI build effort | Low | Low–Medium | Low | Medium | Low–Medium |
| Operational burden on tenant-admin | Low | Low | Low | High | Medium |
| Authz granularity | Tenant-role only | Per resource type | OC service role | Per resource type | Per UCP role |
| OC role model alignment | Partial | None | High | None | None |
| Cross-provider consistency | ✅ | ✅ | ✅ | ❌ | Partial |
| Least privilege at cloud level | ❌ | ❌ | ❌ | ✅ | Partial |
| Blast radius on credential exfiltration | Entire SA | Entire SA | Entire SA | Per resource type | Per role scope |
| Scales with new resource types | ✅ | Needs schema update | ✅ | ❌ (new SA per type) | ✅ |
| Scales with new cloud providers | ✅ | ✅ | ✅ | ❌ | Partial |

---

## STRIDE Threat Analysis

Option 1 and its sub-variants make UCP the security boundary. The threat model focuses on UCP itself.

| Threat | Attack scenario | Severity | Mitigation |
|---|---|---|---|
| **Spoofing** | Attacker impersonates a valid UCP session and acts as a legitimate user | High | Strong authn via Keycloak, HTTPS-only, session expiry, CSRF protection |
| **Elevation of Privilege** | User calls a higher-privilege endpoint by bypassing permission middleware | Critical | Permission middleware applied at route level — all routes must declare required permission explicitly |
| **Elevation of Privilege** | SQL injection or auth bypass extracts another tenant's SA credential | Critical | Parameterized queries, tenant-scoped DB queries, encrypted credential storage |
| **Information Disclosure** | SA key extracted from UCP database or server logs | Critical | Credentials encrypted at rest; never appear in logs or API responses; DB access restricted to api-server only |
| **Repudiation** | User denies having provisioned a resource | Medium | Audit log records every action with user ID, session ID, IP address, timestamp |
| **Tampering** | Attacker modifies a provisioned resource directly via cloud console | Low for UCP | Outside UCP's control — drift detection (MCUCP-158) surfaces this |

---

## Analysis and Rationale

**Options 2A and 2B** distribute authz across multiple credentials at the cloud provider level. This does not pay off for a multi-cloud control plane: it creates provider-specific coupling, increases operational burden for tenants, and does not fully achieve least privilege (the SA boundary is always coarser than UCP's role model regardless of how many SAs are used). They contradict UCP's purpose as a unified abstraction layer.

**Option 1 (base)** is the right foundation. One SA per provider per tenant, UCP as the security boundary. Simple, portable, and scalable — adding new providers, new resource types, or new UCP roles requires no changes to credential management.

**Option 1A and 1B** add authz granularity at the UCP layer without changing the credential model:

- **1B** is the lower-effort path — no new role management UI, no schema changes beyond storing service roles from JWT, and it keeps OC as the source of truth which tenants already manage. The constraint is OC's service taxonomy, but a fallback to tenant-level role for GCP-only resource types is straightforward.
- **1A** decouples UCP entirely from OC's service model, giving full control but requiring a new role management dimension that tenants must learn and maintain in UCP separately.

The genuinely open risk across all single-SA options is **SA key exfiltration from the database** — the one scenario where UCP's authz provides no protection because the attacker bypasses UCP entirely. This requires post-MVP investment in secrets management (Vault, GCP Secret Manager) and credential rotation, independent of which option is chosen.

---

## Recommendation

**Option 1 for MVP.** One service account per provider per tenant, UCP as the authoritative security boundary.

If finer-grained authz is required at MVP, **Option 1B** is the preferred path — it reuses OC's existing role model with minimal build effort, requires no new UI, and is cross-provider by design. The decision on whether to enforce OC service roles is a PM call.

**Option 1A** is a longer-term option if UCP needs to diverge from OC's service taxonomy or manage resource types with no OC equivalent.
