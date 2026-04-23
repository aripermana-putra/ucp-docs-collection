---
title: "Command Reference"
space: UCP
parent_page_id: "../universal-control-plane-ucp.md"
---

# UCP Platform — Command Reference

## Colima (Kubernetes on macOS)

```bash
# Status
colima status

# Start
colima start --cpu 4 --memory 8 --kubernetes

# Stop
colima stop

# Restart
colima stop && colima start --cpu 4 --memory 8 --kubernetes

# Check kubectl is pointed at Colima
kubectl config current-context   # should be: colima
kubectl cluster-info
```

---

## Crossplane

```bash
# Install / upgrade Crossplane
./k8s/deploy-crossplane.sh

# Check Crossplane pods
kubectl get pods -n crossplane-system

# Check installed providers
kubectl get providers

# Check provider health
kubectl describe provider <name>

# Apply XRDs and Compositions
kubectl apply -f crossplane/xrd/ -R
kubectl apply -f crossplane/composition/ -R

# Check all XRDs
kubectl get xrd

# Check all Compositions
kubectl get composition
```

---

## XRD / XR / MR

```bash
# List XRs (e.g. XDatabase)
kubectl get xdatabase -A
kubectl describe xdatabase <name>

# Check XR conditions (Ready / Synced)
kubectl get xdatabase <name> -o jsonpath='{.status.conditions}' | jq .

# List MRs (DatabaseInstance, ComputeInstance, etc.)
kubectl get databaseinstance -A
kubectl get databaseinstance <name> -o yaml

# Check MR conditions
kubectl get databaseinstance <name> -o jsonpath='{.status.conditions}' | jq .

# forProvider vs atProvider — specific MR
kubectl get databaseinstance <name> -o jsonpath='{.spec.forProvider}' | jq .
kubectl get databaseinstance <name> -o jsonpath='{.status.atProvider}' | jq .

# forProvider vs atProvider side by side (requires jq)
kubectl get databaseinstance <name> -o json \
  | jq '{forProvider: .spec.forProvider, atProvider: .status.atProvider}'

# forProvider vs atProvider — all MRs
kubectl get databaseinstance -A -o json \
  | jq '.items[] | {name: .metadata.name, forProvider: .spec.forProvider, atProvider: .status.atProvider}'

# List MRs with drift-protection label
kubectl get databaseinstance -A -l platform.io/drift-protection=true

# Check managementPolicies on MRs
kubectl get databaseinstance -A \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.managementPolicies}{"\n"}{end}'

# Get XR's composed resource refs
kubectl get xdatabase <name> -o jsonpath='{.spec.crossplane.resourceRefs}' | jq .

# Watch XR until ready
kubectl get xdatabase <name> -w

# Delete MR directly (bypasses Crossplane — use with caution)
kubectl delete databaseinstance <name>

# Patch managementPolicies on an MR
kubectl patch databaseinstance <name> \
  --type merge \
  -p '{"spec":{"managementPolicies":["Observe"]}}'

# Unlock MR for recovery (Create + Observe + Update)
kubectl patch databaseinstance <name> \
  --type merge \
  -p '{"spec":{"managementPolicies":["Create","Observe","Update"]}}'

# Add or modify a label on an MR
kubectl label databaseinstance <name> platform.io/drift-protection=true
kubectl label databaseinstance <name> platform.io/drift-protection=true --overwrite

# Remove a label from an MR
kubectl label databaseinstance <name> platform.io/drift-protection-

# Add or modify an annotation on an MR
kubectl annotate databaseinstance <name> example.io/key=value
kubectl annotate databaseinstance <name> example.io/key=value --overwrite

# Remove an annotation from an MR
kubectl annotate databaseinstance <name> example.io/key-

# Delete XR (also cleans up composed MRs)
kubectl delete xdatabase <name>
```

---

## Temporal Stack

```bash
# Deploy full stack (PostgreSQL + Temporal server)
./k8s/deploy-temporal-stack.sh

# Check all pods in temporal-system
kubectl get pods -n temporal-system

# Check deployments (what should be running)
kubectl get deployment -n temporal-system
# temporal-worker          ← Temporal internal (Helm-managed, do not touch)
# ucp-provisioning-worker  ← provisioning worker
# drift-worker             ← drift detection worker

# Restart a deployment
kubectl rollout restart deployment <name> -n temporal-system

# Port-forward Temporal frontend gRPC (required for temporal CLI commands)
kubectl port-forward -n temporal-system svc/temporal-frontend 7233:7233

# Port-forward Temporal Web UI
kubectl port-forward -n temporal-system svc/temporal-web 8233:8080
# Open: http://localhost:8233
```

---

## Workers

```bash
# Deploy provisioning worker
cd k8s/ucp-provisioning-worker && bash deploy.sh && cd ../..

# Deploy drift worker
cd k8s/drift-worker && bash deploy.sh && cd ../..

# Logs
kubectl logs -n temporal-system -l app=ucp-provisioning-worker -f
kubectl logs -n temporal-system -l app=drift-worker -f

# Previous pod logs (if pod crashed)
kubectl logs -n temporal-system -l app=drift-worker --previous
```

---

## API Server

```bash
# Deploy
cd k8s/api-server && bash deploy.sh && cd ../..

# Logs
kubectl logs -n temporal-system -l app=api-server -f

# Port-forward
kubectl port-forward -n temporal-system svc/api-server 8080:8080
```

---

## Frontend

```bash
# Deploy
cd k8s/frontend && bash deploy.sh && cd ../..

# Logs
kubectl logs -n temporal-system -l app=frontend -f

# Port-forward
kubectl port-forward -n temporal-system svc/frontend 3000:80
# Open: http://localhost:3000
```

---

## Temporal Workflows

```bash
# List running workflows
temporal workflow list --query 'ExecutionStatus="Running"'

# List by workflow type
temporal workflow list --query 'WorkflowType="RequestDatabaseWorkflow" AND ExecutionStatus="Running"'
temporal workflow list --query 'WorkflowType="DriftApprovalWorkflow" AND ExecutionStatus="Running"'

# Describe a workflow
temporal workflow describe --workflow-id <id>

# View workflow history
temporal workflow show --workflow-id <id>

# Query approval status
temporal workflow query --workflow-id <id> --type "approval-status"

# Send approval signal
temporal workflow signal \
  --workflow-id <id> \
  --name "approval-signal" \
  --input '{"approved":true}'

# Send rejection signal
temporal workflow signal \
  --workflow-id <id> \
  --name "approval-signal" \
  --input '{"approved":false,"rejected":true,"reason":"not approved"}'

# Terminate a stuck workflow
temporal workflow terminate --workflow-id <id> --reason "manual termination"
```

---

## Drift Detection

```bash
# Create schedule (every 30s)
bash scripts/setup-drift-scan-schedule.sh

# Delete and recreate schedule
temporal schedule delete --schedule-id drift-scan --yes
bash scripts/setup-drift-scan-schedule.sh

# List schedules
temporal schedule list

# Describe schedule
temporal schedule describe --schedule-id drift-scan

# Trigger one scan immediately
temporal schedule trigger --schedule-id drift-scan

# Check running approval workflows
temporal workflow list --query 'WorkflowType="DriftApprovalWorkflow" AND ExecutionStatus="Running"'

# Workflow ID format:
#   cluster-scoped XR:   drift-approval-<xrkind>-<xrname>-<mrname>
#   namespace-scoped XR: drift-approval-<xrnamespace>-<xrkind>-<xrname>-<mrname>
#
# Each drifted MR gets its own workflow — a multi-MR XR (e.g. GKE cluster+nodepool)
# produces two separate approval workflows, one per MR.
#
# Examples (current — all XRs are cluster-scoped):
#   drift-approval-xdatabase-my-db-my-db-instance
#   drift-approval-xcluster-my-gke-my-gke-cluster
#   drift-approval-xcluster-my-gke-my-gke-nodepool

# Approve drift recovery (replace <workflow-id> with the full ID from the list above)
temporal workflow signal \
  --workflow-id "<workflow-id>" \
  --name "approval-signal" \
  --input '{"approved":true}'

# Reject drift recovery
temporal workflow signal \
  --workflow-id "<workflow-id>" \
  --name "approval-signal" \
  --input '{"approved":false,"rejected":true,"reason":"not approved"}'

# Verify managementPolicies flipped back to Observe after recovery
kubectl get databaseinstance <name> -o jsonpath='{.spec.managementPolicies}'
# Expected: ["Observe"]
```

---

## RBAC / Permissions

```bash
# Check drift-worker service account permissions
kubectl auth can-i list databaseinstances.sql.gcp.upbound.io \
  --as=system:serviceaccount:temporal-system:drift-worker

kubectl auth can-i patch databaseinstances.sql.gcp.upbound.io \
  --as=system:serviceaccount:temporal-system:drift-worker

# View ClusterRole rules for a service account
kubectl describe clusterrole drift-worker
kubectl describe clusterrole ucp-provisioning-worker
```

---

## Debugging

```bash
# Crossplane provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/revision -f

# Describe a stuck XR or MR
kubectl describe xdatabase <name>
kubectl describe databaseinstance <name>

# Check events
kubectl get events -n temporal-system --sort-by='.lastTimestamp'
kubectl get events -n crossplane-system --sort-by='.lastTimestamp'

# Check connection secret written by Crossplane
kubectl get secret <name>-conn -n default -o jsonpath='{.data}' | jq 'map_values(@base64d)'
```

---

## Cleanup

```bash
# Delete test XR
kubectl delete xdatabase <name>

# Delete drift schedule
temporal schedule delete --schedule-id drift-scan --yes

# Terminate all running drift approval workflows
temporal workflow list --query 'WorkflowType="DriftApprovalWorkflow" AND ExecutionStatus="Running"' \
  | awk 'NR>1 {print $1}' \
  | xargs -I{} temporal workflow terminate --workflow-id {} --reason "cleanup"

# Full teardown
./k8s/cleanup-all.sh
```
