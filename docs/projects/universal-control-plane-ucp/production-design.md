---
title: "Production Design"
space: UCP
parent_page_id: "../universal-control-plane-ucp.md"
---

# Production Design

System design specifications for the UCP production deployment.

---

## Documents

### [System Design](./production-design/system-design.md)

Full production architecture specification for the MVP milestone (2 cloud providers,
100 tenants, 1,000 active users). Covers NFRs, scale analysis, Kubernetes cluster
topology options with comparison by scalability, availability, blast radius, and
operational complexity, and per-component design with justification for every
technology choice including explicit reasoning for components not included.

### [Architecture Diagrams](./production-design/diagrams.md)

C1 (System Context) and C2 (Container) diagrams for all three cluster topology
options: single cluster, Platform + Operations split, and three-cluster separation.
