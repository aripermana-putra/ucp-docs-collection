---
title: "WIF PoC — Evidence"
space: UCP
parent_page_id: "../wif-gcp.md"
---

# WIF PoC — Evidence

Externally sourced facts, provider source code findings, and references. For what was actually run and observed, see [implementation.md](./implementation.md).

---

## Provider Source Code Findings

### Supported credential sources (`provider-upjet-gcp-beta` — `apis/cluster/v1beta1/types.go`)

Source: [upbound/provider-upjet-gcp-beta — apis/cluster/v1beta1/types.go](https://github.com/upbound/provider-upjet-gcp-beta/blob/main/apis/cluster/v1beta1/types.go)

The `ProviderCredentials` struct defines:

```go
// +kubebuilder:validation:Enum=None;Secret;AccessToken;ImpersonateServiceAccount;InjectedIdentity;Environment;Filesystem;Upbound
Source xpv1.CredentialsSource `json:"source"`
```

And a `Federation` struct under `Upbound`:

```go
type Federation struct {
    ProviderID     string `json:"providerID"`
    ServiceAccount string `json:"serviceAccount"`
}
```

`ProviderID` format: `projects/<project-id>/locations/global/workloadIdentityPools/<pool>/providers/<provider>`

**Implication:** WIF is supported natively through the `Upbound`+`Federation` source. `InjectedIdentity` is present for GKE Workload Identity. `Secret` with `external_account` JSON is feasible via the GCP SDK's ADC support.

**Caveat:** `provider-upjet-gcp-beta` is versioned at v1.x. UCP uses `provider-family-gcp:v2.6.0`. The types are assumed to be the same codebase — to be verified during PoC.

---

## GCP WIF — How OIDC Verification Works

Source: [GCP WIF with Kubernetes — official guide](https://cloud.google.com/iam/docs/workload-identity-federation-with-kubernetes)

- GCP STS fetches the JWKS from `<issuer>/.well-known/openid-configuration` and then from the `jwks_uri` declared there
- The issuer URL must be publicly reachable by GCP
- Only the JWKS (public keys) needs to be public — the K8s API server itself does not need to be exposed
- The projected ServiceAccount token's `audience` must match the WIF pool provider's `allowed-audiences`

---

## GCP `external_account` Credential Type

Source: [GCP Application Default Credentials — external_account](https://google.aip.dev/auth/4117)

The `external_account` JSON credential config contains no secret material. It instructs the GCP SDK to:
1. Read a subject token from a file or URL
2. Exchange it at `token_url` (GCP STS)
3. Optionally impersonate a service account via `service_account_impersonation_url`

All major GCP client libraries (Go, Python, Java, Node) support this credential type natively.

---

## CCoE Guardrail GP 106

Source: [CCoE Public Cloud Infrastructure Guardrails v2](https://confluence.rakuten-it.com/confluence/spaces/CLOUDSOL/pages/5411030282/Public+Cloud+Infrastructure+Guardrails+Version+2)

- Constraint: `constraints/iam.workloadIdentityPoolProviders`
- Type: list constraint — defines which IDP URIs are **allowed** (not a blanket ban on WIF)
- Scope: **L1 Compliant Systems only**
- Policy source on `sub-gcp-ucp-clsd-sandbox`: inherited from parent — no project-level overrides

**Confirmed active on sandbox project.** WIF provider creation with an unlisted issuer URI returns:
```
FAILED_PRECONDITION: Org Policy violated for value: 'https://storage.googleapis.com/ucp-wif-poc-jwks'
type: constraints/iam.workloadIdentityPoolProviders
```

### Observed allowed list on `sub-gcp-ucp-clsd-sandbox`

Inspected via GCP Console → IAM & Admin → Organization Policies:

| URI | Identity |
|-----|----------|
| `https://sts.amazon.com` | AWS STS |
| `https://ghe.rakuten-it.com/_services/token` | Rakuten GHE |
| `https://login.microsoftonline.com/53a8b0d9-d900-48cc-9d7e-5935dc8d5b17/v2.0` | Azure AD |
| `https://token.actions.githubusercontent.com` | GitHub Actions |
| `https://sts.windows.net/53a8b0d9-d900-48cc-9d7e-5935dc8d5b17` | Azure STS |
| `https://aks-cluste-cped-jpeast-rg-88a072-4188q7in.hcp.japaneast.azmk8s.io` | AKS cluster (Japan East) |

**Key observation:** the AKS entry is a K8s cluster OIDC issuer — the same pattern UCP needs. CCoE has already approved this pattern once. This is a direct precedent for requesting UCP's cluster issuer to be added.

Reference: [GCP WIF best practices — disable pool creation](https://cloud.google.com/iam/docs/best-practices-for-using-workload-identity-federation#disable-pool-creation)

---

## GKE Workload Identity

Source: [GKE Workload Identity — official guide](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)

- When running on GKE, the `InjectedIdentity` credential source uses the GKE metadata server
- No public OIDC endpoint required — GKE handles OIDC discovery internally
- The K8s SA is annotated with the GCP SA email; GKE links them automatically
- This is the cleanest WIF path but requires UCP to run on GKE

---

## References

- [upbound/provider-upjet-gcp-beta — types.go](https://github.com/upbound/provider-upjet-gcp-beta/blob/main/apis/cluster/v1beta1/types.go)
- [GCP Workload Identity Federation — overview](https://cloud.google.com/iam/docs/workload-identity-federation)
- [GCP WIF with Kubernetes — official guide](https://cloud.google.com/iam/docs/workload-identity-federation-with-kubernetes)
- [GKE Workload Identity — official guide](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
- [GCP external_account credential type (ADC)](https://google.aip.dev/auth/4117)
- [GCP WIF best practices — disable pool creation](https://cloud.google.com/iam/docs/best-practices-for-using-workload-identity-federation#disable-pool-creation)
- [CCoE Public Cloud Infrastructure Guardrails v2](https://confluence.rakuten-it.com/confluence/spaces/CLOUDSOL/pages/5411030282/Public+Cloud+Infrastructure+Guardrails+Version+2)
- [UCP Policy Enforcement — guardrail coverage](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6671968824/Policy+enforcement)
- [Crossplane GCP ProviderConfig — credential sources](https://docs.crossplane.io/latest/concepts/providers/)
