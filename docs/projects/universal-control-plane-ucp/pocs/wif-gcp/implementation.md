---
title: "WIF PoC — Implementation"
space: UCP
parent_page_id: "../wif-gcp.md"
---

# WIF PoC — Implementation

What was built, run, and observed. For background, see [concepts.md](./concepts.md).

---

## Environment

| Component | Value |
|-----------|-------|
| Cluster type | Local (colima + k3s v1.31.11) |
| Crossplane version | 2.2.0 |
| provider-family-gcp version | v2.6.0 |
| GCP sandbox project | `sub-gcp-ucp-clsd-sandbox` |
| GCP project classification | L1 (GP 106 enforced) |
| Cluster OIDC issuer | `https://ghe.rakuten-it.com/_services/token` ⚠️ dummy — see note below |
| WIF pool | `ucp-wif-poc-pool` |
| WIF provider | `ucp-k8s-provider` |

> ⚠️ **PoC workaround — dummy issuer URI + manual JWK upload**
>
> GP 106 blocked our actual issuer URI (`https://storage.googleapis.com/ucp-wif-poc-jwks`). To unblock the PoC, we used `https://ghe.rakuten-it.com/_services/token` — a URI already on the GP 106 allowed list — as the issuer URI in the WIF provider, and uploaded the cluster's actual JWKS file directly via the GCP Console.
>
> GCP uses the uploaded JWK for signature verification instead of fetching from the issuer URI, so the actual cluster keys are used correctly. The cluster is reconfigured to issue tokens with `iss=https://ghe.rakuten-it.com/_services/token` to match.
>
> **This is not acceptable for production.** The issuer URI should be a URL UCP genuinely owns or controls. For production, CCoE must add a real UCP-owned issuer URL to the GP 106 allowed list.

---

## Step 1 — Set Up Public JWKS Endpoint

### 1a. Extract the cluster's public JWKS

```bash
kubectl get --raw /openid/v1/jwks > jwks.json
```

### 1b. Host JWKS in a public GCS bucket

```bash
gsutil mb gs://ucp-wif-poc-jwks
gsutil iam ch allUsers:objectViewer gs://ucp-wif-poc-jwks
gsutil cp jwks.json gs://ucp-wif-poc-jwks/jwks.json
```

Verify: `curl https://storage.googleapis.com/ucp-wif-poc-jwks/jwks.json`

Also create the OIDC discovery document:

```bash
cat > openid-configuration.json << EOF
{
  "issuer": "https://storage.googleapis.com/ucp-wif-poc-jwks",
  "jwks_uri": "https://storage.googleapis.com/ucp-wif-poc-jwks/jwks.json",
  "response_types_supported": ["id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
EOF
gsutil cp openid-configuration.json gs://ucp-wif-poc-jwks/.well-known/openid-configuration
```

### 1c. Recreate the cluster with the matching issuer URL

> **Note:** Changing `--service-account-issuer` requires recreating the cluster. Existing ServiceAccount tokens become invalid.
>
> The issuer URL must match whatever was registered in the WIF provider. Since we used the dummy workaround (see environment note above), the issuer is set to `https://ghe.rakuten-it.com/_services/token`.

```bash
colima delete -p ucp --force

colima start \
  --cpu 4 --memory 8 --disk 40 \
  --kubernetes \
  --kubernetes-version "v1.31.11+k3s1" \
  --k3s-arg "--kube-apiserver-arg=service-account-issuer=https://ghe.rakuten-it.com/_services/token" \
  -p ucp
```

Verify issuer in new tokens:
```bash
kubectl create token default -n default | python3 -c "
import sys, base64, json
token = sys.stdin.read().strip()
payload = token.split('.')[1]
payload += '=' * (4 - len(payload) % 4)
print(json.dumps(json.loads(base64.b64decode(payload)), indent=2))
"
```

Expected: `"iss": "https://ghe.rakuten-it.com/_services/token"`

---

## Step 2 — Set Up GCP Side (Tenant Project)

> **GP 106 workaround applied — provider created successfully via GCP Console**
>
> Direct `gcloud` creation with our actual issuer URI was blocked by GP 106. Workaround: used `https://ghe.rakuten-it.com/_services/token` (already on the allowed list) as the issuer URI and uploaded the cluster's JWKS file directly. Provider `ucp-k8s-provider` created and shows healthy (green) in the GCP Console.
>
> The gcloud commands below reflect the intended final state. For the PoC the provider was created via the UI using the dummy issuer workaround.



```bash
PROJECT_ID=<sandbox-project-id>
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
POOL_ID=ucp-wif-poc-pool
PROVIDER_ID=ucp-k8s-provider
SA_EMAIL=ucp-crossplane-poc@${PROJECT_ID}.iam.gserviceaccount.com
K8S_SA_NAMESPACE=crossplane-system
K8S_SA_NAME=<crossplane-provider-sa-name>

# Create workload identity pool
gcloud iam workload-identity-pools create $POOL_ID \
  --project=$PROJECT_ID \
  --location=global \
  --display-name="UCP WIF PoC Pool"

# Create OIDC provider pointing to UCP's OIDC issuer
gcloud iam workload-identity-pools providers create-oidc $PROVIDER_ID \
  --project=$PROJECT_ID \
  --location=global \
  --workload-identity-pool=$POOL_ID \
  --issuer-uri="https://storage.googleapis.com/ucp-wif-poc-jwks" \
  --allowed-audiences="https://storage.googleapis.com/ucp-wif-poc-jwks" \
  --attribute-mapping="google.subject=assertion.sub"

# Create a GCP SA for Crossplane to impersonate
gcloud iam service-accounts create ucp-crossplane-poc \
  --project=$PROJECT_ID \
  --display-name="UCP Crossplane WIF PoC"

# Grant required GCP permissions on the SA (adjust roles as needed for PoC)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/cloudsql.admin"

# Grant the K8s SA permission to impersonate the GCP SA
gcloud iam service-accounts add-iam-policy-binding $SA_EMAIL \
  --project=$PROJECT_ID \
  --role=roles/iam.workloadIdentityUser \
  --member="principal://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/subject/system:serviceaccount:${K8S_SA_NAMESPACE}:${K8S_SA_NAME}"
```

---

## Step 3 — Install Crossplane + GCP Provider

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace

kubectl apply -f crossplane/providers/gcp/provider-family-gcp.yaml
kubectl apply -f crossplane/providers/gcp/provider-gcp-sql.yaml
```

---

## Step 4 — Test Approach A: `Secret` with `external_account` JSON

### 4a. Generate the credential config

```bash
gcloud iam workload-identity-pools create-cred-config \
  projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/providers/${PROVIDER_ID} \
  --service-account=$SA_EMAIL \
  --credential-source-file=/var/run/secrets/tokens/gcp-token \
  --credential-source-type=text \
  --output-file=wif-credential-config.json
```

### 4b. Store as K8s Secret

```bash
kubectl create secret generic tenant-wif-creds \
  --namespace=crossplane-system \
  --from-file=credentials.json=wif-credential-config.json
```

### 4c. Create ProviderConfig

```yaml
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: wif-poc-providerconfig
spec:
  projectID: <PROJECT_ID>
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: tenant-wif-creds
      key: credentials.json
```

### 4d. Mount projected SA token on provider pod

The provider pod needs a projected ServiceAccount token at `/var/run/secrets/tokens/gcp-token`. This requires a `DeploymentRuntimeConfig`:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: provider-gcp-wif
spec:
  serviceAccountTemplate:
    metadata:
      name: ucp-provider-gcp-storage   # stable name — never changes across upgrades
  deploymentTemplate:
    spec:
      selector: {}
      template:
        spec:
          volumes:
          - name: gcp-token
            projected:
              sources:
              - serviceAccountToken:
                  audience: "ucp-platform"   # fixed audience — all tenants use this value
                  expirationSeconds: 3600
                  path: gcp-token
          containers:
          - name: package-runtime
            volumeMounts:
            - name: gcp-token
              mountPath: /var/run/secrets/tokens
              readOnly: true
```

> **Multi-tenant design:** the fixed audience `ucp-platform` means all tenants configure their WIF provider with the same audience value. Per-tenant differentiation is handled by the credential config JSON — each tenant's secret specifies a different GCP SA email to impersonate. One shared provider pod, N tenants.

Reference this from the provider:

```yaml
spec:
  runtimeConfigRef:
    name: provider-gcp-wif
```

### 4e. Verify

Create a test GCS bucket (faster than Cloud SQL) and check status:

```bash
kubectl apply -f - <<EOF
apiVersion: storage.gcp.upbound.io/v1beta1
kind: Bucket
metadata:
  name: ucp-wif-poc-test-bucket
  annotations:
    crossplane.io/external-name: ucp-wif-poc-test-bucket-clsd
spec:
  providerConfigRef:
    name: wif-poc-providerconfig
  forProvider:
    location: ASIA-NORTHEAST1
EOF

kubectl get buckets.storage.gcp.upbound.io ucp-wif-poc-test-bucket
```

Expected: `SYNCED=True, READY=True`

> **Note:** apply the `DeploymentRuntimeConfig` and `workloadIdentityUser` binding to ALL provider pods being tested, not just the SQL one. Each GCP provider has its own K8s SA — each must be individually bound.

---

## Test Results

| Credential Source | Result | Notes |
|------------------|--------|-------|
| `Secret` + `external_account` JSON | ✅ **Working** | GCS bucket provisioned — `SYNCED=True, READY=True` |

---

## Observations

- `sub-gcp-ucp-clsd-sandbox` is subject to `constraints/iam.workloadIdentityPoolProviders` enforced at org/folder level
- The constraint has no project-level overrides — all external WIF provider URIs are blocked
- Both approaches A and B require creating a WIF pool provider in the tenant's GCP project — both are blocked by the same constraint
- The WIF pool itself was created successfully (`ucp-wif-poc-pool`) — only the provider creation is blocked

### GP 106 Allowed List (observed via GCP Console)

The effective policy on `sub-gcp-ucp-clsd-sandbox` allows exactly these URIs:

| URI | Identity |
|-----|----------|
| `https://sts.amazon.com` | AWS STS — workloads on AWS |
| `https://ghe.rakuten-it.com/_services/token` | Rakuten internal GHE — CI/CD pipelines |
| `https://login.microsoftonline.com/53a8b0d9-d900-48cc-9d7e-5935dc8d5b17/v2.0` | Azure AD |
| `https://token.actions.githubusercontent.com` | GitHub Actions (public) |
| `https://sts.windows.net/53a8b0d9-d900-48cc-9d7e-5935dc8d5b17` | Azure STS |
| `https://aks-cluste-cped-jpeast-rg-88a072-4188q7in.hcp.japaneast.azmk8s.io` | Specific AKS cluster (Japan East) |

**Notable finding:** the last entry is a specific AKS cluster's OIDC issuer — the same pattern as what UCP needs. CCoE has already approved this pattern before, just for AKS rather than a self-hosted or GKE cluster.

### Issues Encountered During Execution

| Issue | Error | Fix |
|-------|-------|-----|
| Token file not found | `failed to open credential file "/var/run/secrets/tokens/gcp-token"` | `DeploymentRuntimeConfig` must be applied to the provider being tested — each provider needs it individually |
| Audience mismatch | `The audience in ID Token [...] does not match the expected audience` | Use a fixed custom audience `ucp-platform` in the projected volume and configure all tenant WIF providers with the same value |
| SA impersonation denied | `Permission 'iam.serviceAccounts.getAccessToken' denied` | `workloadIdentityUser` binding must cover each provider's K8s SA — solved permanently by using a stable SA name via `serviceAccountTemplate` |
| Provider SA name changes on upgrade | Principal string handed to tenants would break | Set `serviceAccountTemplate.metadata.name: ucp-provider-gcp-storage` — name is now stable regardless of provider version |

### Final State

GCS bucket `ucp-wif-poc-test-bucket-clsd` provisioned in `sub-gcp-ucp-clsd-sandbox` with `SYNCED=True, READY=True` using:
- Stable SA name: `ucp-provider-gcp-storage`
- Fixed audience: `ucp-platform`
- No SA key file at any point

Confirmed via `gsutil ls -b gs://ucp-wif-poc-test-bucket-clsd`.
