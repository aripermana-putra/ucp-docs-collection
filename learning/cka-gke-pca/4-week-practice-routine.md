# 4-Week Practice Routine

This guide is meant to answer one practical question:

"Where do I actually learn each topic from?"

It is designed for:
- MacBook with Colima
- A personal GCP sandbox project
- CKA as the main track
- GKE, networking, and light PCA as supporting tracks

## How to Use This Guide

Use a 4-part study loop for every topic:
1. Watch: use your Udemy or Pluralsight course for orientation
2. Read: use official docs for correctness and depth
3. Lab: do the task locally or in GCP
4. Write: summarize what you learned in your own words

Do not try to find a separate course for every small topic.
Instead, use:
- one main course per track
- official docs as the source of truth
- your own labs as retention work

## Your Source Ladder

When learning a topic, use sources in this order:

1. Main course
   Use your current Udemy CKA course as the main guide for Kubernetes topics.

2. Official docs
   Use the official docs to verify concepts, terminology, and edge cases.

3. Hands-on lab
   Practice first in Colima for speed, then in GCP for realism.

4. Your own notes
   Write one short page after each topic.

## Core Resource Map

### CKA Core
- CNCF CKA blueprint:
  [CKA Certification](https://www.cncf.io/training/certification/cka/)
- Kubernetes Basics tutorial:
  [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- Kubernetes Concepts:
  [Concepts](https://kubernetes.io/docs/concepts/)
- Kubernetes Tasks:
  [Tasks](https://kubernetes.io/docs/tasks/)
- kubectl quick reference:
  [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

### GKE and Networking Core
- GKE networking overview:
  [GKE Network Overview](https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview)
- GCP VPC overview:
  [VPC Overview](https://cloud.google.com/vpc/docs/vpc)
- GKE flexible Pod CIDR:
  [Flexible Pod CIDR](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr)
- GKE discontiguous multi-Pod CIDR:
  [Multi-Pod CIDR](https://cloud.google.com/kubernetes-engine/docs/how-to/multi-pod-cidr)
- GKE alias IPs overview:
  [Alias IPs and GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)

### PCA Core
- PCA certification page:
  [Professional Cloud Architect](https://cloud.google.com/learn/certification/cloud-architect)
- Google Cloud Architecture Framework:
  [Architecture Framework](https://cloud.google.com/architecture/framework)

## What to Use on Each Platform

### Udemy
Use your CKA course as the main teaching path for:
- Pods, Deployments, ReplicaSets
- Services and networking basics
- ConfigMaps and Secrets
- Scheduling
- Storage basics
- Troubleshooting drills

### Pluralsight
Use Pluralsight to fill gaps where you want a more structured explanation for:
- networking fundamentals
- Linux basics for Kubernetes admins
- Terraform fundamentals
- cloud architecture basics

Search using these phrases:
- `CIDR subnetting routing DNS NAT`
- `Kubernetes networking fundamentals`
- `Terraform fundamentals GCP`
- `Google Cloud architecture fundamentals`

### Google Cloud Skills Boost
Use this mainly for:
- GKE labs
- GCP networking labs
- PCA-related architecture exposure
- IAM, VPC, load balancing, observability, and Terraform on GCP

When searching Skills Boost, prioritize learning paths and labs around:
- Google Kubernetes Engine
- VPC networking
- Professional Cloud Architect
- Terraform on Google Cloud

## Study Model

Use a 2-lab system:
- Local lab with Colima for fast repetition and troubleshooting drills
- GCP sandbox for GKE, networking, IAM, load balancing, and Terraform practice

Each topic should be practiced in 3 passes:
1. Follow the lab once
2. Repeat from memory
3. Break it on purpose and troubleshoot it

## Weekly Rhythm

Use this structure each week:
- Session 1: Watch + local lab, 45-60 min
- Session 2: Read + local lab, 45-60 min
- Session 3: Troubleshooting drill, 30-45 min
- Session 4: GCP sandbox lab, 60-90 min
- Session 5: Review and notes, 20-30 min

Keep one note page per topic with:
- What the concept is
- YAML or commands you used
- What failed
- How you diagnosed it
- What signals mattered most
- What you still do not understand yet

## Week 1: Core Kubernetes + Networking Basics

### Goal
Get comfortable with the basic workflow of creating, inspecting, exposing, and debugging workloads.

### Watch
Use your CKA course sections that cover:
- Pods
- Deployments
- ReplicaSets
- Services
- Namespaces
- basic kubectl usage

### Read
- [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
  Focus on: Pods, Deployments, ReplicaSets, Services, Namespaces
- [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [VPC Overview](https://cloud.google.com/vpc/docs/vpc)
  Focus on: VPC, subnet, IP range

### Learn These Networking Basics
- What an IP address is
- What CIDR notation means
- What a subnet is
- The difference between Pod IP, Service IP, and node IP
- DNS basics inside and outside Kubernetes

### Local Lab with Colima
- Create a Deployment from YAML
- Scale it up and down
- Expose it with a ClusterIP Service
- Expose another app with NodePort
- Use `kubectl get`, `describe`, `logs`, and `exec`
- Delete and recreate the workload from memory

### GCP Sandbox Lab
- Create or inspect a small GKE cluster
- Identify the cluster VPC and subnet
- Check whether the cluster is VPC-native
- Deploy a sample app and expose it with a Service
- Observe internal vs external access paths

### Write
Create `week-1-notes.md` with:
- basic kubectl workflow
- ClusterIP vs NodePort
- CIDR explained in your own words
- Pod IP vs Service IP vs node IP

## Week 2: Scheduling, Configuration, and Failure Signals

### Goal
Understand how workloads land on nodes and how common misconfigurations show up.

### Watch
Use your CKA course sections that cover:
- resource requests and limits
- probes
- ConfigMaps and Secrets
- rolling updates and rollbacks
- taints, tolerations, and selectors

### Read
- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
  Focus on: scheduling, ConfigMaps, Secrets, probes, rollout behavior
- [Kubernetes Tasks](https://kubernetes.io/docs/tasks/)
  Focus on: ConfigMaps, Secrets, probes, deployment updates
- [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

### Learn These Networking Basics
- port vs targetPort
- selector matching
- why a Service can exist but send no traffic
- internal DNS naming for Services

### Local Lab with Colima
- Create a Deployment with requests and limits
- Add readiness and liveness probes
- Inject config through ConfigMaps and Secrets
- Roll out a bad image and recover with rollback
- Break a selector and troubleshoot why traffic fails

### GCP Sandbox Lab
- Create a second node pool if practical
- Use selectors or taints to influence scheduling
- Observe where Pods land
- Inspect events, logs, and rollout status during failures

### Write
Create `week-2-notes.md` with:
- common Service failure patterns
- probe failure symptoms
- how requests and limits affect scheduling
- how to debug a Deployment that is not becoming ready

## Week 3: GKE Networking and Platform Operations

### Goal
Connect Kubernetes concepts to cloud networking and scaling behavior.

### Watch
Use your CKA course sections that cover:
- Services and networking review
- Ingress or Gateway basics if included
- Jobs and CronJobs
- storage basics
- troubleshooting

Then use your GCP-focused material on:
- GKE networking
- VPC-native clusters
- load balancers
- node pools and autoscaling

### Read
- [GKE Network Overview](https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview)
- [Alias IPs and GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)
- [Flexible Pod CIDR](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr)
- [Multi-Pod CIDR](https://cloud.google.com/kubernetes-engine/docs/how-to/multi-pod-cidr)

### Learn These Networking Basics
- VPC-native cluster meaning
- Pod CIDR vs Service CIDR
- max Pods per node
- why Pod IP exhaustion can limit scaling
- how traffic reaches Pods through a load balancer at a high level

### Local Lab with Colima
- Deploy a multi-service app
- Route traffic to the right backend
- Create a Job and a CronJob
- Create a PVC and inspect storage objects if supported
- Practice fast troubleshooting with events and status fields

### GCP Sandbox Lab
- Deploy a workload behind a LoadBalancer Service or Ingress
- Inspect cluster networking settings
- Map Pod IP range, Service IP range, and node subnet
- Read the docs while comparing them with your actual cluster settings
- Write down how a Pod CIDR exhaustion scenario would happen

### Write
Create `week-3-notes.md` with:
- a diagram of user -> load balancer -> service -> pod
- a plain-English explanation of Pod CIDR exhaustion
- notes on how node pools and IP ranges interact
- one summary paragraph of the GKE thread scenario in your own words

## Week 4: Integration, Terraform, and Light PCA Thinking

### Goal
Tie together Kubernetes operations, GCP resources, and architecture thinking.

### Watch
Use:
- your CKA course for troubleshooting review
- Pluralsight or Skills Boost for Terraform basics and GCP architecture fundamentals
- PCA-oriented material for reliability, scalability, cost, and security tradeoffs

### Read
- [Professional Cloud Architect](https://cloud.google.com/learn/certification/cloud-architect)
- [Architecture Framework](https://cloud.google.com/architecture/framework)
- [VPC Overview](https://cloud.google.com/vpc/docs/vpc)

### Learn These Networking Basics
- firewall allow rules
- Cloud NAT purpose
- private vs public egress
- internal vs external load balancing

### Local Lab with Colima
- Run timed troubleshooting drills
- Rebuild core manifests from memory
- Practice fixing broken images, selectors, ports, and probes

### GCP Sandbox Lab
- Provision a simple environment with Terraform if possible:
  - VPC
  - subnet
  - GKE cluster or supporting resources
- Compare manual setup vs Terraform setup
- Document what should always be managed as code

### Write
Create `week-4-notes.md` with:
- what you can now do from memory
- where you still feel weak
- what topics should become month 2 priorities
- one short architecture recommendation for an internal platform workload

## What to Practice Locally with Colima
- writing YAML from scratch
- creating Deployments and Services quickly
- troubleshooting failed rollouts
- inspecting logs and events
- debugging DNS and selector issues
- practicing under time pressure

## What to Practice in GCP
- GKE cluster creation
- node pools and autoscaling behavior
- VPC and subnet concepts
- IAM and workload identity concepts
- load balancers and ingress behavior
- Pod and Service IP planning
- Terraform for repeatable infrastructure

## How to Judge Progress

At the end of 4 weeks, you should be able to:
- explain a Deployment and a Service without notes
- diagnose common selector, port, image, and probe issues
- describe Pod IP vs Service IP vs node IP
- explain why GKE networking design affects scaling limits
- speak more confidently in platform and SRE discussions

## Month 2 Direction

After this 4-week cycle, continue with:
- deeper troubleshooting
- storage and stateful workloads
- RBAC and security foundations
- observability basics
- more Terraform and GitOps patterns
- more structured PCA study
- CKS preparation after your CKA base feels stable
