---
title: "Multi-Project GCP — Implementation"
space: UCP
parent_page_id: "../multi-project-gcp.md"
---

# Multi-Project GCP — Implementation

---

## Implementation

| Component | File | Status |
|---|---|---|
| Live GCP credential validation (OAuth2 + Cloud Resource Manager) | `settings_handler.go` | Not deployed |
| GCP project label check (`ucp-env` label enforcement) | `settings_handler.go` | Not deployed |
| Multi-project secret naming — include `project_id` in secret name | `settings_handler.go` | Not deployed |
| Multi-project ProviderConfig naming — include `project_id` | `settings_handler.go` | Not deployed |
| `GET /api/v1/settings/credentials/gcp` — list registered GCP projects | `settings_handler.go` | Not deployed |
| Create form project selector — fetches and renders available ProviderConfigs | frontend create forms | Not deployed |
| `providerConfig` field in create requests — no longer auto-derived | all create handlers + forms | Not deployed |

---

## Scope

### In Scope

- **Live key validation** — exchange private key for OAuth2 token, verify against Cloud Resource Manager; confirms key is active and SA has minimum required permissions
- **Environment label enforcement** — GCP project must carry `ucp-env: {qa|production}` label matching the UCP instance environment; enforced at credential upload time using the Cloud Resource Manager response
- **Multi-project secret + ProviderConfig naming** — `(tenant, project_id, env)` triple replaces the current `(tenant, env)` pair; prevents collision when a tenant registers multiple projects
- **Project list endpoint** — `GET /api/v1/settings/credentials/gcp` returns all registered GCP projects for a tenant with their ProviderConfig names
- **Project selector in create forms** — frontend fetches the project list and renders a dropdown; single-project tenants see it auto-selected (backwards compatible)
- **`providerConfig` no longer auto-derived** — create handlers accept explicit `providerConfig` from the request instead of calling `gcpProviderConfigName(tenantID, env)`

### Out of Scope

- **GCP project label auto-setting** — UCP does not set GCP project labels; the label must be set by the GCP project owner
- **Multi-cloud providers (AWS, Azure)** — this PoC focuses on GCP; the pattern generalises but is not implemented for other providers
- **Credential rotation** — detecting expired keys and triggering rotation remains out of scope

---

## API Sequence — Credential Upload with Live Validation

`POST /api/v1/settings/credentials`

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Handler as API Server
    participant GCPAuth as GCP OAuth2
    participant GCRM as Cloud Resource Manager

    Browser->>Handler: POST /api/v1/settings/credentials<br/>{provider: "gcp", tenantId: ..., credentials: {...}}

    Handler->>Handler: validateGCPCredentials() — structural check<br/>(type, project_id, client_email, private_key present)

    Handler->>GCPAuth: POST /token<br/>signed JWT (RS256, private_key from SA JSON)
    GCPAuth-->>Handler: access_token OR error

    alt key revoked or expired
        Handler-->>Browser: 400 Bad Request — "GCP credentials are invalid or revoked"
    end

    Handler->>GCRM: GET /v1/projects/{project_id}<br/>Authorization: Bearer {access_token}
    GCRM-->>Handler: project metadata including labels

    alt labels["ucp-env"] != current environment
        Handler-->>Browser: 400 Bad Request — "project is registered for {other-env}, not {current-env}"
    end

    Handler->>Handler: store K8s Secret<br/>name: gcp-credentials-{tenant}-{project_id}-{env}
    Handler->>Handler: upsertGCPProviderConfig()<br/>name: gcp-{tenant}-{project_id}-{env}
    Handler-->>Browser: 201 Created {projectId, providerConfig, clientEmail}
```

---

## API Sequence — Resource Creation with Project Selection

`POST /api/v1/storage` with explicit providerConfig

```mermaid
sequenceDiagram
    participant Browser
    participant Handler as API Server
    participant K8s as Kubernetes

    Browser->>Handler: GET /api/v1/settings/credentials/gcp?tenantId=...
    Handler->>K8s: list Secrets with label platform.ucp.io/tenant={tenant}
    K8s-->>Handler: [{projectId: "proj-a", providerConfig: "gcp-...-proj-a-qa"}, ...]
    Handler-->>Browser: {projects: [...]}

    Browser->>Handler: POST /api/v1/storage<br/>{name: "my-bucket", tenantId: ..., providerConfig: "gcp-...-proj-a-qa"}

    Handler->>Handler: stamp XR with providerConfig from request body
    Handler->>K8s: CREATE XObjectStorage (providerConfigRef: gcp-...-proj-a-qa)
    Note over K8s: Crossplane patches providerConfigRef to the MR<br/>MR uses that ProviderConfig's credentials → provisions in proj-a
```

---

## Verification

### Prerequisite: set the GCP project label

```bash
gcloud projects update <project-id> \
  --update-labels ucp-env=qa
```

### Test 1 — Valid key + correct label → accepted

```bash
curl -s -X POST -H "$COOKIE" -H "X-Environment: QA" \
  -d "{\"provider\":\"gcp\",\"tenantId\":\"$TENANT\",\"credentials\":$(cat sa-key.json | jq -c .)}" \
  http://localhost:8080/api/v1/settings/credentials
# → 201 Created with projectId and providerConfig name
```

### Test 2 — Valid key + wrong label → rejected

```bash
# Project has label ucp-env=production, UCP environment is QA
curl -s -X POST -H "$COOKIE" -H "X-Environment: QA" ...
# → 400 Bad Request: "project is registered for production, not qa"
```

### Test 3 — Revoked/invalid key → rejected

```bash
# Use a key with private_key modified to be invalid
curl -s -X POST -H "$COOKIE" -H "X-Environment: QA" ...
# → 400 Bad Request: "GCP credentials are invalid or revoked"
```

### Test 4 — List registered projects

```bash
curl -s -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/settings/credentials/gcp?tenantId=$TENANT" | jq .
# → { "projects": [{ "projectId": "...", "providerConfig": "..." }] }
```

### Test 5 — Create resource with explicit project selection

```bash
curl -s -X POST -H "$COOKIE" -H "X-Environment: QA" -H "X-Tenant-ID: $TENANT" \
  -d '{"name":"test-bucket","provider":"gcs","projectId":"...","providerConfig":"gcp-...-proj-a-qa"}' \
  http://localhost:8080/api/v1/storage
# → 202 Accepted; verify XR has correct providerConfigRef
kubectl get xobjectstorage test-bucket -o jsonpath='{.spec.parameters.providerConfig}'
```

---

## Verification Results

*To be filled in after tests are run.*

| Test | Expected | Actual | Status |
|---|---|---|---|
| Test 1 — valid key + correct label | 201 Created | | Pending |
| Test 2 — valid key + wrong label | 400 env mismatch | | Pending |
| Test 3 — invalid/revoked key | 400 invalid credentials | | Pending |
| Test 4 — list registered projects | project list with ProviderConfig names | | Pending |
| Test 5 — create resource with project selection | XR carries correct providerConfigRef | | Pending |
