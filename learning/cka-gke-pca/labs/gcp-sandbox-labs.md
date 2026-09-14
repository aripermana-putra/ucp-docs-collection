# GCP Sandbox Labs

Use these labs to connect Kubernetes concepts to real GCP and GKE behavior.

## Lab 1: Basic GKE Cluster Walkthrough
- Create or inspect a small GKE cluster
- Identify the VPC and subnet used
- Check whether the cluster is VPC-native
- Deploy a sample app
- Expose it with a Service
- Compare internal and external access paths

## Lab 2: Node Pools and Scheduling
- Create or inspect multiple node pools
- Label nodes or use selectors where practical
- Deploy workloads with placement rules
- Observe scheduling results
- Record what changed and what did not

## Lab 3: Service, Load Balancer, and Ingress
- Deploy a simple app behind a LoadBalancer Service
- If practical, add Ingress
- Inspect how traffic reaches the workload
- Note which GCP resources appear outside Kubernetes

## Lab 4: GKE Networking Inspection
- Inspect Pod IP range
- Inspect Service IP range
- Inspect node subnet range
- Compare max Pods per node with the available Pod range
- Write a short explanation of how address exhaustion could happen

## Lab 5: Terraform Baseline
- Provision a small VPC and subnet with Terraform
- Provision a simple GKE-related environment if practical for cost
- Compare manual setup vs IaC setup
- Write down what should always be codified

## Lab 6: Architecture Reflection
- Sketch a small internal platform workload on GCP
- Identify reliability, security, cost, and operability concerns
- Write one recommendation and one tradeoff you accepted
