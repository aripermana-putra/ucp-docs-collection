# Resource Map

This file turns the weekly plan into concrete places to learn from.

## Core Official Resources
- CNCF CKA blueprint:
  https://www.cncf.io/training/certification/cka/
- Kubernetes Basics:
  https://kubernetes.io/docs/tutorials/kubernetes-basics/
- Kubernetes Concepts:
  https://kubernetes.io/docs/concepts/
- Kubernetes Tasks:
  https://kubernetes.io/docs/tasks/
- kubectl Quick Reference:
  https://kubernetes.io/docs/reference/kubectl/quick-reference/
- GKE Network Overview:
  https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview
- GKE Alias IPs:
  https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips
- Flexible Pod CIDR:
  https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr
- Multi-Pod CIDR:
  https://cloud.google.com/kubernetes-engine/docs/how-to/multi-pod-cidr
- GCP VPC Overview:
  https://cloud.google.com/vpc/docs/vpc
- PCA certification page:
  https://cloud.google.com/learn/certification/cloud-architect
- Google Cloud Architecture Framework:
  https://cloud.google.com/architecture/framework

## How to Use Udemy
Use your CKA course as the main guided path for:
- Pods, Deployments, ReplicaSets
- Services and networking basics
- ConfigMaps and Secrets
- Scheduling
- Storage basics
- Troubleshooting

When a section feels unclear:
- pause the course
- read the official Kubernetes docs for that exact object
- redo the lab locally in Colima

## How to Use Pluralsight
Use Pluralsight to add structure where your CKA course is thin.
Search these phrases:
- `CIDR subnetting routing DNS NAT`
- `Kubernetes networking fundamentals`
- `Linux fundamentals for Kubernetes`
- `Terraform fundamentals GCP`
- `Google Cloud architecture fundamentals`

Use Pluralsight mostly for:
- networking foundations
- Terraform basics
- cloud architecture background
- Linux command-line reinforcement

## How to Use Google Cloud Skills Boost
Use Skills Boost for hands-on GCP/GKE labs and architecture exposure.
Search these phrases:
- `Google Kubernetes Engine`
- `Kubernetes in Google Cloud`
- `VPC networking`
- `Professional Cloud Architect`
- `Terraform Google Cloud`
- `Cloud Load Balancing`
- `Cloud NAT`
- `IAM Google Cloud`

Prioritize these kinds of content:
- learning paths
- skill badges
- hands-on labs
- architecture case-study style modules

## Weekly Resource Map

### Week 1
Watch:
- Udemy CKA sections on Pods, Deployments, Services, Namespaces, kubectl basics

Read:
- Kubernetes Basics
- Kubernetes Concepts for Pods, Deployments, Services
- kubectl Quick Reference
- GCP VPC Overview

Search in Pluralsight if needed:
- `CIDR subnetting basics`
- `DNS basics`

Search in Skills Boost if needed:
- `Kubernetes in Google Cloud`
- `Google Kubernetes Engine basics`

### Week 2
Watch:
- Udemy CKA sections on ConfigMaps, Secrets, probes, rollouts, scheduling

Read:
- Kubernetes Concepts and Tasks for ConfigMaps, Secrets, probes, updates
- kubectl Quick Reference

Search in Pluralsight if needed:
- `Kubernetes scheduling basics`
- `health checks containers kubernetes`

Search in Skills Boost if needed:
- `GKE workload deployment`
- `IAM Google Cloud basics`

### Week 3
Watch:
- Udemy networking review sections
- Any GKE/networking modules available in your learning platforms

Read:
- GKE Network Overview
- Alias IPs and GKE
- Flexible Pod CIDR
- Multi-Pod CIDR

Search in Pluralsight if needed:
- `Kubernetes networking fundamentals`
- `load balancing basics`

Search in Skills Boost if needed:
- `Google Kubernetes Engine networking`
- `VPC networking`
- `Cloud Load Balancing`

### Week 4
Watch:
- troubleshooting review
- Terraform basics
- PCA or GCP architecture overview material

Read:
- PCA certification page
- Google Cloud Architecture Framework
- VPC Overview

Search in Pluralsight if needed:
- `Terraform fundamentals GCP`
- `cloud architecture fundamentals`

Search in Skills Boost if needed:
- `Professional Cloud Architect`
- `Terraform Google Cloud`
- `Cloud NAT`
- `Cloud Load Balancing`

## Recommended Study Pattern
For each topic, do this:
1. watch 20-40 minutes
2. read 15-30 minutes
3. lab 30-60 minutes
4. write notes for 10 minutes

## Minimum Viable Weekly Output
At the end of each week, you should have:
- one completed notes file
- one local lab completed at least twice
- one GCP lab completed or partly completed
- one short explanation written in your own words
