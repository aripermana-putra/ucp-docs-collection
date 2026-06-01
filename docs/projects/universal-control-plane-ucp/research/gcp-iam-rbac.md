---
title: "GCP IAM — RBAC Research"
space: UCP
---

# GCP IAM — RBAC Research

| | |
|---|---|
| **Author** | aripermana.putra |
| **Date** | 2026-06-01 |
| **Purpose** | Understanding GCP's RBAC model and its implications for UCP |

---

## Overview

GCP's access control system is called **Cloud IAM (Identity and Access Management)**. It answers the question: *who can do what on which resource*. The three components map directly to that sentence:

| Component | GCP term | Description |
|---|---|---|
| Who | **Principal** | The identity requesting access |
| Can do what | **Role** | A named collection of permissions |
| On which resource | **Resource + Policy** | Where the role binding is attached |

---

## Principals

A principal is any authenticated identity that can be granted access. GCP supports:

| Type | Identifier format | Description |
|---|---|---|
| Google Account | `user:alice@example.com` | Individual human user |
| Service Account | `serviceAccount:sa@project.iam.gserviceaccount.com` | Machine identity for workloads |
| Google Group | `group:team@example.com` | Collection of Google Accounts |
| Google Workspace domain | `domain:example.com` | All users in a domain |
| All authenticated users | `allAuthenticatedUsers` | Any Google-authenticated identity |
| All users | `allUsers` | Public, no authentication required |
| Workload Identity | `principal://...` | Federated external identity (e.g. GitHub Actions, AWS) |

Service accounts are the standard identity for machine-to-machine access. Rather than using static long-lived keys, Workload Identity Federation allows external workloads to exchange their own identity tokens (e.g. OIDC from Keycloak, AWS STS) for short-lived GCP credentials — eliminating the need to manage service account key files.

---

## Permissions

Permissions are the atomic unit of access control. Every GCP API method maps to a permission. The naming format is:

```
service.resource.verb
```

Examples:
- `compute.instances.list` — list Compute Engine VMs
- `compute.instances.delete` — delete a Compute Engine VM
- `storage.objects.get` — read a Cloud Storage object
- `iam.roles.create` — create an IAM role

Permissions are never granted directly to principals. They are always bundled into roles, and roles are granted.

---

## Role Types

### Basic roles (formerly primitive)

Extremely broad. Exist for historical reasons and are not recommended for production.

| Role | Covers |
|---|---|
| `roles/viewer` | Read-only access to all resources |
| `roles/editor` | Read + write access to all resources |
| `roles/owner` | Full access including IAM management |

### Predefined roles

Maintained by Google, scoped to a specific service. Automatically updated when Google adds new permissions. Examples:

| Role | Service | What it allows |
|---|---|---|
| `roles/compute.instanceAdmin.v1` | Compute Engine | Full VM lifecycle management |
| `roles/compute.viewer` | Compute Engine | List and get VMs |
| `roles/storage.admin` | Cloud Storage | Full bucket and object management |
| `roles/storage.objectViewer` | Cloud Storage | Read objects only |
| `roles/cloudsql.admin` | Cloud SQL | Full database management |
| `roles/cloudsql.client` | Cloud SQL | Connect to databases |
| `roles/container.admin` | GKE | Full cluster management |
| `roles/container.developer` | GKE | Deploy to clusters, no admin |

There are hundreds of predefined roles — one per meaningful use case per service.

### Custom roles

User-defined collections of permissions. Created at organisation or project level, can only be granted within that same scope. Up to 300 per organisation and 300 per project.

Custom roles are useful when predefined roles are either too permissive (include things you don't want to grant) or not granular enough (you need a subset). The recommended approach is to start from a similar predefined role and remove what isn't needed.

Limitations:
- Not all permissions are supported in custom roles
- Cannot be granted across organisations or projects
- Cannot be created at folder level
- Role IDs are immutable after creation and cannot be reused for 44 days after deletion

---

## Resource Hierarchy and Policy Inheritance

GCP organises resources in a tree:

```
Organisation
  └── Folder (optional, can be nested)
        └── Project
              └── Service resource (e.g. VM, bucket, Cloud SQL instance)
```

An **allow policy** (formerly IAM policy) is a set of role bindings attached to a node in this hierarchy. The effective permissions on any resource are the **union** of all policies from that resource up to the organisation root.

```
Org policy
  + Folder policy
    + Project policy
      + Resource policy
= Effective permissions
```

This means granting a role at the organisation level gives it everywhere. Granting at the project level covers all resources in that project. Granting at the resource level covers only that resource.

**Important:** policies only add permissions — they never subtract. To restrict access that was granted higher up, use deny policies (see below).

---

## Allow Policies

An allow policy is a JSON/YAML document attached to a resource containing role bindings:

```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:alice@example.com",
        "serviceAccount:worker@project.iam.gserviceaccount.com"
      ]
    },
    {
      "role": "roles/storage.admin",
      "members": [
        "group:storage-admins@example.com"
      ]
    }
  ]
}
```

A binding maps a role to one or more principals. Any principal in the binding receives all permissions in that role on the resource the policy is attached to (and all descendants if the resource is a hierarchy node).

Bindings can include **conditions** — CEL expressions that restrict when the binding applies.

---

## Deny Policies

Deny policies are a separate mechanism that **explicitly blocks** access regardless of what allow policies grant. IAM always evaluates deny policies before allow policies — a deny rule wins unconditionally.

Deny policies attach to organisations, folders, or projects and cascade downward.

Use cases:
- Block specific operations organisation-wide (e.g. no one can create custom roles except the security team)
- Grant broad access at a high level but carve out exceptions for sensitive resources
- Condition-based blocking (e.g. deny access outside corporate network)

---

## IAM Conditions

Conditions add attribute-based logic to role bindings and deny policies using CEL expressions. They narrow when a binding applies without requiring separate roles.

Supported attribute categories:

| Category | Examples |
|---|---|
| Resource | resource type, name, tags |
| Request | timestamp, IP address, URL path, access level |
| Principal | principal type, identity |

Common use cases:
- Temporary access (`request.time < timestamp("2026-12-31T00:00:00Z")`)
- Network-restricted access (`request.auth.access_levels contains "accessPolicies/.../accessLevels/corpnet"`)
- Tag-based environment separation (prod vs staging resources under same project)
- Business hours only

---

## How Access Evaluation Works

When a principal makes a request to a GCP resource:

1. GCP collects all deny policies applicable to the resource (from resource up to org)
2. If any deny rule matches and there is no exception → **access denied**
3. GCP collects all allow policies applicable to the resource (from resource up to org)
4. If the union of allow policies includes the required permission → **access granted**
5. Otherwise → **access denied**

The evaluation is stateless per-request — no session concept, no caching of role state.

---

## Comparison to OC/UCP RBAC

| Aspect | GCP IAM | OC (Horizon) | UCP |
|---|---|---|---|
| Role granularity | Per service, per resource level | Per service per tenant | Per tenant (3 roles) |
| Permission model | Hundreds of atomic permissions | Tenant role + service role | 5-permission bitmask |
| Where roles attach | Any resource in the hierarchy | Tenant membership | Tenant (flat, no hierarchy) |
| Deny mechanism | Explicit deny policies | No equivalent | No equivalent |
| Conditions | Attribute-based (CEL) | None | None |
| Machine identity | Service accounts + Workload Identity | Service accounts | Not defined |
| Custom roles | Yes, up to 300 per org/project | No | Not applicable |
| Inheritance | Yes, down the org/folder/project tree | No | No |

---

## Implications for UCP

**GCP resources provisioned by UCP do not have per-user GCP IAM role assignments today.** UCP creates the resource (Cloud SQL, GKE, GCS) under a service account whose credentials are stored in the tenant's provider config. The GCP-level access policy on those resources is whatever the service account has — typically project-level admin for the relevant service.

This means:

- A UCP `developer` who provisions a Cloud SQL instance gets it created under the tenant's service account, but does not personally receive any GCP IAM role on that instance
- There is no GCP-level enforcement that only `tenant-admin` can delete a resource — that enforcement lives entirely in UCP's `RequirePermission` middleware
- If someone with direct GCP access (outside UCP) has the right IAM roles, they can modify or delete UCP-managed resources without going through UCP at all

**The open design question** (OQ8 in the RBAC POC report) — whether to introduce per-service UCP roles — is separate from GCP IAM. GCP IAM has no concept of "this user can provision DBs but not GKE clusters in this tenant." UCP would need to model that itself using its own permission layer, not GCP IAM bindings.

If UCP ever wanted to grant end users direct GCP console/API access to their provisioned resources (beyond what UCP exposes), Workload Identity Federation would be the mechanism — exchanging a Keycloak JWT for a scoped GCP access token tied to the appropriate predefined or custom role on the relevant resources.

---

## References

- [Cloud IAM overview](https://docs.cloud.google.com/iam/docs/overview)
- [Roles overview](https://docs.cloud.google.com/iam/docs/roles-overview)
- [Resource hierarchy and access control](https://docs.cloud.google.com/iam/docs/resource-hierarchy-access-control)
- [IAM Conditions overview](https://docs.cloud.google.com/iam/docs/conditions-overview)
- [Deny policies overview](https://docs.cloud.google.com/iam/docs/deny-overview)
- [Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation)
- [Understanding custom roles](https://docs.cloud.google.com/iam/docs/understanding-custom-roles)
