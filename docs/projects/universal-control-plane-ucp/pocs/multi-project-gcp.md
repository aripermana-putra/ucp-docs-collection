---
title: "Multi-Project GCP"
space: UCP
parent_page_id: "../pocs.md"
---

# Multi-Project GCP

A single OC tenant may own multiple GCP projects — for example, one project per
service or one per workload. This PoC validates that UCP can manage resources
across multiple GCP projects under the same tenant and that credential
registration includes both live key validation and environment isolation
enforcement.

---

## Goals

1. **Credential validation** — verify a GCP service account key is actually
   live at upload time, not just structurally valid.
2. **Environment isolation** — prevent a user from accidentally or intentionally
   registering production GCP credentials into a non-production UCP instance.
3. **Multi-project per tenant** — a tenant can register multiple GCP projects
   and select which project to use when provisioning a resource.

---

## Sub-Documents

- [Design](multi-project-gcp/design.md) — live validation flow, label
  enforcement, multi-project storage and ProviderConfig naming
- [Implementation](multi-project-gcp/implementation.md) — scope, status table,
  sequence diagrams, verification

---

## Related

- RBAC PoC — tenant registration and credential management
- MCUCP-192 — tenant isolation (ProviderConfig per tenant)
