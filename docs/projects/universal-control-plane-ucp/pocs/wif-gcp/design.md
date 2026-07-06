---
title: "WIF PoC — Design"
space: UCP
parent_page_id: "../wif-gcp.md"
---

# WIF PoC — Design

Human review document. Read this before the PoC starts executing.

| | |
|---|---|
| **Ticket** | MCUCP-217 |
| **Parent research** | [Workload Identity Federation — Feasibility for UCP](../../research/workload-identity-federation-gcp.md) |

---

## Research Question

> Which credential source in `provider-upjet-gcp` works for Workload Identity Federation on a self-hosted, non-GKE Kubernetes cluster?

---

## Hypothesis

`Secret` with `external_account` JSON is the most likely approach to work. It relies purely on the GCP SDK's Application Default Credentials support — not on provider-specific infrastructure — meaning it should work on any cluster where the provider pod can read a file and reach GCP STS over the internet.

`Upbound` + `Federation` may also work but is suspected to have a dependency on Upbound's managed control plane. This needs to be tested to confirm or rule out.

`InjectedIdentity` is GKE-only and is not applicable to this PoC.

---

## Scope

| Item | In scope | Out of scope |
|------|----------|--------------|
| Local cluster (kind / k3d) | ✅ | |
| GKE cluster | | ✅ — separate path if needed |
| `provider-family-gcp` v2.6.0 | ✅ | |
| Cloud SQL provisioning as verification target | ✅ | |
| Full tenant onboarding automation | | ✅ — post-PoC |
| GP 106 CCoE allowlist approval | | ✅ — use L0 sandbox only |
| Migration path for existing SA key tenants | | ✅ — post-PoC |

---

## Approach

### Phase 1 — Unblock the OIDC issuer problem

Local clusters use `kubernetes.default.svc.cluster.local` as the OIDC issuer — unreachable by GCP STS. The workaround is to host the cluster's public JWKS on a publicly reachable URL (e.g GCS bucket) and configure the cluster to use that URL as its issuer.

Steps:
1. Extract the cluster's public JWKS (`kubectl get --raw /openid/v1/jwks`)
2. Host the JWKS and OIDC discovery document on a public GCS bucket
3. Recreate the cluster with `--service-account-issuer=<gcs-url>`

### Phase 2 — Configure GCP side (one-time per sandbox project)

1. Create a Workload Identity Pool in the sandbox GCP project
2. Create a WIF provider pointing to the GCS-hosted OIDC issuer
3. Create a GCP Service Account with Cloud SQL admin permissions
4. Grant the Crossplane provider's K8s ServiceAccount `roles/iam.workloadIdentityUser` on the GCP SA

### Phase 3 — Test Approach A (`Secret` + `external_account`)

1. Generate the `external_account` credential config with `gcloud iam workload-identity-pools create-cred-config`
2. Store it as a K8s Secret (no private key material)
3. Create a `ProviderConfig` with `source: Secret` referencing it
4. Add a `DeploymentRuntimeConfig` to mount a projected ServiceAccount token volume on the provider pod
5. Provision a Cloud SQL instance through Crossplane and confirm success

### Phase 4 — Test Approach B (`Upbound` + `Federation`)

1. Create a `ProviderConfig` with `source: Upbound` and the `federation` block
2. Observe provider pod behavior — check if it exchanges tokens or errors immediately
3. Determine whether `Upbound` source requires Upbound managed platform infrastructure

---

## Success Criteria

| Criterion | Pass condition |
|-----------|---------------|
| OIDC issuer workaround works | GCP STS successfully verifies a K8s ServiceAccount JWT signed by the local cluster |
| At least one credential source works end-to-end | Crossplane successfully provisions a Cloud SQL instance without a SA key file |
| Approach A verdict | Confirmed working or confirmed broken with root cause |
| Approach B verdict | Confirmed working, or confirmed limited to Upbound managed platform |
| Tenant setup steps documented | Clear list of what the tenant must configure in their GCP project |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| GP 106 blocks WIF pool creation in sandbox project | Low (likely L0) | High | Confirm project classification first; use L0 sandbox |
| `Upbound` source requires Upbound managed platform | Medium | Low — Approach A still available | Test and document; fall back to Approach A |
| Projected token volume not supported on self-hosted provider pod | Medium | High for Approach A | Check `DeploymentRuntimeConfig` support in v2.6.0 |
| JWKS key rotation breaks WIF after cluster recreation | Certain on recreation | Low for PoC | Note in docs; re-upload JWKS after cluster recreate |
| Provider types differ between `provider-upjet-gcp-beta` v1.x and `provider-family-gcp` v2.6.0 | Low | High | Inspect the running CRD schema early in Phase 3 |

---

## Open Questions

- Does `provider-family-gcp` v2.6.0 have the same `Upbound`/`Federation` credential source as `provider-upjet-gcp-beta` v1.x?
- Does `DeploymentRuntimeConfig` in v2.6.0 support adding projected volume mounts to provider pods?
- Is the sandbox GCP project classified as L0 or L1? (determines whether GP 106 applies)
- Does the `Upbound` credential source work without Upbound's managed control plane?
