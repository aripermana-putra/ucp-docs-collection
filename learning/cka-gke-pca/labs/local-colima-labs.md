# Local Colima Labs

Use these labs for fast, repeatable CKA-style practice.

## Lab 1: Deployment and Service Basics
- Create a namespace for practice
- Deploy an nginx app from YAML
- Scale replicas up and down
- Expose it with a ClusterIP Service
- Expose another app with NodePort
- Verify endpoints and connectivity
- Delete and recreate from memory

## Lab 2: Troubleshooting by Breaking Things
- Break the image name
- Break the Service selector
- Break the containerPort / targetPort mapping
- Break the readiness probe
- Use `kubectl get`, `describe`, `logs`, and `get events`
- Write down the exact symptom and fix

## Lab 3: Config and Rollouts
- Create a ConfigMap and mount it
- Create a Secret and inject it as env vars
- Perform a rolling update
- Intentionally deploy a bad image tag
- Roll back the Deployment

## Lab 4: Scheduling Basics
- Add resource requests and limits
- Use nodeSelector if your setup supports multiple nodes
- Practice taints and tolerations if available
- Observe where pods land and why

## Lab 5: Jobs and Storage
- Create a Job
- Create a CronJob
- Create a PVC if your local setup supports it
- Inspect Job and PVC status fields

## Speed Drill Rules
- Do one run while following notes
- Do one run from memory
- Do one run while timing yourself
- Do one run where you intentionally break it first
