---
title: "Interview Template - Software Engineer"
space: UCP
parent_page_id: "../others.md"
---

## Usage Notes

Use this page as the master template for interviewer copy/paste. For each interview, duplicate the page, fill in the basic information, then select questions from the pool based on the category focus and the target level of the position: `Associate`, `Mid`, `Senior`, or `Lead`. A typical 60-minute interview should cover:

- 5 minutes for introductions
- 10 minutes for candidate background
- 35 to 40 minutes for technical and scenario questions
- 5 to 10 minutes for candidate questions and wrap-up

Recommended question mix per interview:

- 2 questions from software engineering and system design
- 2 questions from cloud-native platform topics
- 1 question from operations and observability
- 1 question from collaboration or ownership

## 1. Basic Info

| Field | Value |
| --- | --- |
| Candidate Name |  |
| Interview Date |  |
| Position | Software Development Engineer, Universal Control Plane Group - CLSD |
| Interview Round |  |
| Interviewer |  |
| Focus Area |  |
| Target Level | Associate/Mid/Senior/Lead |

## 2. Interviewer Checklist

- Review the candidate resume and identify 2 areas to probe deeply.
- Align question selection with the round focus.
- Prefer depth over breadth; ask follow-up questions.
- Capture concrete evidence, not only impressions.
- Check for hands-on experience vs. theoretical familiarity.
- Evaluate ownership, reliability, and production judgment.
- Leave 5 minutes for candidate questions.

## 3. Introduction Prompt

Use the following prompt to start the conversation:

> Please spend 3 to 5 minutes introducing yourself. Focus on your current role, the systems you own or contribute to, the cloud-native technologies you use most, and one or two projects that best represent your experience.

Follow-up prompts if needed:

- What is the scale of the system in terms of users, services, or traffic?
- What parts did you personally design, implement, or operate?
- What were the most difficult production issues you handled?
- What trade-offs did you have to make?
- What interested you in this role and in working on an internal cloud platform team?

## 4. Interviewer Assessment

>Copy the following block once per interviewer.

**Interviewer Name:** {Name}

-

**Round / Focus**

-

**Question**

-

Example: `A01, B03, F02`

**Strengths**

-

**Concerns**

-

**Notes**

-

**Hiring Recommendation**

>Strong Hire/Hire/No Hire/Strong No Hire 
>**Assessed level :** Associate/Mid/Senior/Lead
>**Summary :**

## 5. Question Pool

Use the tables below to choose questions. Use the question ID in the assessment section, for example `A01`, `B02`, or `F01`. Use the `Level` column to match the target role while keeping the original category-based structure. Use the `Rating` column to mark `Above`, `Meets`, or `Below`.

### A. Software Engineering Fundamentals

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| A01 | Associate | Tell me about a backend or platform feature you implemented. What part did you personally build? | Clear ownership, concrete implementation details, basic testing and deployment awareness |  |  |
| A02 | Associate | How do you make sure the code you write is safe to merge? | Basic review habits, testing, validation, asking for feedback, attention to quality |  |  |
| A03 | Associate | Walk me through how you typically deliver a feature from requirement to production. | End-to-end delivery awareness, communication, testing, rollout discipline |  |  |
| A04 | Mid | Tell me about a service you designed and implemented end-to-end. What decisions did you own? | Ownership across design, implementation, rollout, and support |  |  |
| A05 | Mid | Describe a time you improved a fragile or hard-to-maintain service. | Practical refactoring, prioritization, measurable improvements |  |  |
| A06 | Mid | Tell me about a time code review significantly improved a design or prevented a problem. | Learns from review, values feedback, can explain concrete technical impact |  |  |
| A07 | Senior | What makes a control-plane service truly production-ready? | Reliability, runbooks, observability, rollout strategy, alerting, security |  |  |
| A08 | Senior | When would you choose Go, Python, or Java for a control-plane service? | Trade-offs around performance, concurrency, ecosystem, team skill, operational simplicity |  |  |
| A09 | Senior | How do you approach testing for infrastructure or platform code that is hard to unit test? | Pragmatic testing strategy across integration, contract, and end-to-end validation |  |  |
| A10 | Lead | How do you raise the engineering bar in code review, design review, and operational readiness across a platform team? | Coaching, standards, enabling mechanisms, pragmatic consistency |  |  |

### B. Distributed Systems and System Design

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| B01 | Associate | What is the difference between a synchronous API call and an asynchronous workflow? | Basic understanding of latency, retries, waiting, and decoupling |  |  |
| B02 | Mid | What is the difference between a request-response service and a reconciliation-based control plane? | Desired state vs. actual state, drift correction, eventual consistency |  |  |
| B03 | Mid | How would you design a highly available service that provisions cloud resources? | Redundancy, retries, idempotency, state management, failure handling |  |  |
| B04 | Mid | How do you handle API versioning and backward compatibility when platform consumers depend on your interfaces? | Contract discipline, migration planning, deprecation strategy, consumer empathy |  |  |
| B05 | Senior | Design a control-plane service that manages resources across private and public clouds. What are the key components? | API layer, reconciliation loop, workflow engine, persistence, policy, auth, observability, failure handling |  |  |
| B06 | Senior | How would you handle partial failure when provisioning a multi-step cloud resource workflow? | Saga or compensation thinking, retries, checkpoints, idempotency keys, failure visibility, human intervention paths |  |  |
| B07 | Senior | Walk me through how you would design a multi-tenant resource provisioning system from scratch. | Tenant isolation, authorization, quotas, data model, auditability, operations |  |  |
| B08 | Lead | How do you reason about consistency, correctness, and operational simplicity when designing distributed control-plane systems? | Mature trade-off thinking grounded in real examples |  |  |
| B09 | Lead | How would you evolve a universal control plane over the next 12 to 24 months as platform scope, team count, and cloud coverage increase? | Strategic thinking, boundaries, platform APIs, scale, organizational alignment |  |  |

### C. Kubernetes, Controllers, and Cloud-Native Platforms

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| C01 | Associate | What is Kubernetes used for in your current or past work? | Practical usage, not just textbook definitions |  |  |
| C02 | Associate | What do you understand by desired state and reconciliation? | Early understanding of controller-style thinking and eventual consistency |  |  |
| C03 | Mid | Have you built or extended a Kubernetes controller or operator? Walk through the design. | Watches, queues, retries, status conditions, finalizers, conflict handling, testing strategy |  |  |
| C04 | Mid | What parts of Kubernetes are most relevant when building a control plane product? | API server, CRDs, controllers, reconciliation, admission, RBAC, operators, namespaces |  |  |
| C05 | Senior | What common mistakes have you seen in controller design? | Non-idempotent behavior, poor status reporting, hot loops, missing finalizers, unsafe assumptions |  |  |
| C06 | Senior | How would you safely roll out a new control-plane feature that changes reconciliation behavior? | Feature flags, canary rollout, compatibility, migration plan, telemetry, rollback path |  |  |
| C07 | Lead | How do you balance platform standardization with team autonomy when building shared cloud services? | Product thinking, governance, abstraction boundaries, adoption mindset |  |  |

### D. Crossplane, Argo Workflows, and Workflow Orchestration

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| D01 | Associate | Tell me about a job, workflow, or automation you have built or supported. | Familiarity with multi-step tasks, retries, operational awareness |  |  |
| D02 | Mid | When would you use Argo Workflows or Temporal-style orchestration instead of plain controller logic? | Distinguishes long-running workflows, visibility, retries, fan-out, compensation, stateful execution |  |  |
| D03 | Mid | What problems do tools like Crossplane solve, and where do they fit in a platform architecture? | Control-plane abstraction, composable infrastructure, cloud resource lifecycle, platform APIs |  |  |
| D04 | Senior | Describe a workflow you built or operated that had multiple distributed steps and external dependencies. | Practical example, reliability concerns, retries, timeouts, observability, compensation |  |  |
| D05 | Senior | How do you decide what belongs in a controller, what belongs in a workflow engine, and what belongs in a service layer? | Strong boundary thinking, operational clarity, maintainability, failure domain awareness |  |  |
| D06 | Senior | What is your approach to idempotency in workflow steps, and why does it matter? | Understands retries, duplicate execution, safety, correctness, external side effects |  |  |
| D07 | Lead | You need to migrate from one workflow engine to another with minimal disruption. How would you plan and de-risk it? | Migration sequencing, compatibility, dual-run strategy, rollback, observability |  |  |
| D08 | Lead | What are the risks of long-running workflows, and how do you mitigate them at platform scale? | State explosion, stuck executions, versioning, timeout management, auditability, cleanup |  |  |

### E. Databases and Persistence

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| E01 | Associate | When would you choose MySQL or PostgreSQL for a service? | Basic data-model reasoning, transactions, relational use cases |  |  |
| E02 | Mid | Tell me about a system you built using MySQL, PostgreSQL, or Cassandra. Why was that datastore chosen? | Clear workload-based reasoning, data model awareness, trade-offs, operational implications |  |  |
| E03 | Mid | What kinds of workloads are a good fit for Cassandra, and what are the trade-offs? | High write throughput, distributed scale, partition design, eventual consistency, query-driven modeling |  |  |
| E04 | Senior | How would you model state for a provisioning system that needs auditability and recovery? | Durable state transitions, event or history thinking, schema clarity, recovery and reconciliation |  |  |
| E05 | Senior | How do you avoid data integrity issues in distributed workflows that touch multiple systems? | Idempotency, transactional boundaries, outbox or event patterns, compensation, audit trails |  |  |
| E06 | Senior | How do you approach database migrations in a production system with zero-downtime requirements? | Schema evolution, compatibility windows, rollout safety, operational caution |  |  |
| E07 | Lead | What database operational issues have you personally handled in production, and how did you lead the response? | Real examples such as slow queries, replication lag, hotspotting, schema changes, backup and restore |  |  |

### F. Observability, Operations, and Reliability

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| F01 | Associate | If a service is failing in production, what would you check first? | Logs, dashboards, recent changes, basic triage discipline |  |  |
| F02 | Associate | Tell me about a bug or incident you fixed. How did you approach it? | Structured debugging, learning mindset, clear ownership |  |  |
| F03 | Mid | What signals would you monitor for a control-plane service? | Availability, latency, queue depth, reconciliation duration, workflow failure rates, dependency health |  |  |
| F04 | Mid | Tell me about a production incident you handled. How did you investigate and resolve it? | Calm debugging process, evidence-based steps, collaboration, remediation, follow-up actions |  |  |
| F05 | Mid | How do you decide what metrics, logs, and alerts are actually useful versus just noisy? | Signal-to-noise judgment, operator empathy, actionable alert design |  |  |
| F06 | Senior | How would you use OpenTelemetry in a multi-service platform? | Trace propagation, standard attributes, sampling trade-offs, linking traces with logs and metrics |  |  |
| F07 | Senior | How do you distinguish a symptom from a root cause in operational support? | Structured triage, narrowing hypotheses, data correlation, postmortem mindset |  |  |
| F08 | Senior | How do you approach defining and measuring SLOs for an internal platform service? | SLI selection, user impact thinking, realistic targets, operational follow-through |  |  |
| F09 | Senior | Describe a time a silent failure caused a real problem. How did you detect it and prevent recurrence? | Can detect missing signals, improve observability, and reason about blind spots |  |  |
| F10 | Lead | How do you design an observability strategy so that product teams and platform operators can both diagnose issues effectively? | Audience-aware observability, service ownership, shared standards, useful telemetry |  |  |
| F11 | Lead | A new platform feature causes reconciliation latency and backlog growth across multiple tenants. How would you lead the response? | Technical triage, communication, containment, prioritization, follow-through |  |  |

### G. Service Communication, Security, and Platform Boundaries

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| G01 | Mid | What problems do service mesh technologies like Envoy help solve? | Traffic management, mTLS, observability, policy enforcement, resilience patterns |  |  |
| G02 | Mid | What role does an API gateway like Kong play in a platform environment? | Authn or authz, routing, rate limiting, policy, visibility, consumer management |  |  |
| G03 | Senior | How would you secure a control plane that provisions resources across multiple environments or tenants? | Identity, RBAC, secrets handling, isolation, auditing, least privilege, network boundaries |  |  |
| G04 | Senior | How do you approach mTLS between services, and when is it necessary? | Practical security judgment, cert lifecycle awareness, trade-offs in implementation |  |  |
| G05 | Senior | How do you approach secret management for automation-heavy systems? | Rotation, short-lived credentials, vault patterns, auditability, minimizing blast radius |  |  |
| G06 | Lead | What are the risks of centralizing too much logic in a gateway or mesh layer? | Hidden complexity, ownership ambiguity, debugging difficulty, policy sprawl, performance trade-offs |  |  |
| G07 | Lead | How do you think about platform boundaries between control plane, gateway, and service mesh responsibilities? | Clear ownership model, simpler operations, debuggability, maintainability |  |  |

### H. Scenario-Based Questions

Use these scenarios intentionally vaguely at first. Do not immediately add constraints unless the candidate asks. The goal is to see whether they begin by asking clarifying questions and whether they can justify trade-offs instead of listing tools or jargon.

**What to look for before they propose a solution**

- Do they ask about users, consumers, or tenants?
- Do they ask about expected traffic, throughput, workload shape, or scale?
- Do they ask about latency, availability targets, or SLA/SLO expectations?
- Do they ask about consistency, failure handling, auditability, security, or compliance needs?
- Do they ask about operational ownership, team constraints, timeline, or existing platform dependencies?
- When they suggest a tool or pattern, do they explain why it fits the problem and what trade-offs it introduces?

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| H01 | Mid | Design a service that allows internal teams to request and manage shared infrastructure resources through a common platform API. | Starts with clarifying questions, identifies actors and workflows, reasons about API, state, failure handling, and operations before naming tools |  |  |
| H02 | Mid | A platform capability is becoming hard to operate and users say it feels slow and unreliable. How would you approach the problem? | Clarifies symptoms vs impact, asks for usage patterns and expectations, narrows scope, proposes measurement before solution |  |  |
| H03 | Senior | Design a control-plane style system that manages long-running operations across multiple dependent components. | Asks about scale, correctness, latency expectations, recovery, idempotency, visibility, and ownership; chooses patterns with rationale |  |  |
| H04 | Senior | You need to introduce a major change into a critical platform service without disrupting users. How would you plan and execute it? | Clarifies blast radius, compatibility constraints, user expectations, rollback needs, observability, and release strategy |  |  |
| H05 | Lead | Design a shared internal platform capability that must support multiple teams with different needs while remaining operable and governable. | Explores tenant variation, governance, golden paths, ownership boundaries, support model, cost, and long-term maintainability |  |  |
| H06 | Lead | A platform area has grown organically and different teams are proposing different tools and architectures to solve similar problems. How would you lead the decision? | Clarifies business context and constraints, avoids premature tool bias, evaluates trade-offs explicitly, and drives toward coherent platform strategy |  |  |

### I. Ownership, Collaboration, and Behavioral

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| I01 | Associate | How do you ask for help when you are blocked on a technical issue? | Healthy collaboration, communication, self-awareness |  |  |
| I02 | Mid | How do you work with product managers, QA, and other developers when requirements are ambiguous? | Clarification, slicing scope, explicit assumptions, delivery mindset, partnership |  |  |
| I03 | Mid | Tell me about a time you had to support a service you did not originally build. | Ownership mindset, learning speed, respectful collaboration, operational follow-through |  |  |
| I04 | Senior | Tell me about a time you disagreed with a technical direction. What did you do? | Healthy communication, evidence-based discussion, willingness to align after decision |  |  |
| I05 | Senior | Tell me about a time you prevented a risky change or identified a hidden operational risk. | Good judgment, influence, concrete action, measurable avoided impact |  |  |
| I06 | Senior | Tell me about a time you had to influence a decision without direct authority. | Influence, credibility, stakeholder management, outcome orientation |  |  |
| I07 | Lead | How do you balance shipping speed with reliability in platform engineering? | Pragmatic trade-offs, staged rollout, risk classification, clear accountability |  |  |
| I08 | Lead | How do you build trust with engineering teams that depend on infrastructure you control? | Internal customer orientation, transparency, consistency, follow-through |  |  |
| I09 | Lead | What distinguishes a strong senior engineer from a lead engineer in a cloud platform organization? | Clear level expectations, mentoring, system ownership, multiplier behavior |  |  |

### J. Platform Mindset and Internal Developer Platform

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| J01 | Mid | What do you think makes internal platform engineering different from building end-user product features? | Understands internal customers, enablement, abstraction boundaries, and platform adoption |  |  |
| J02 | Mid | Tell me about a time you improved developer experience for internal users. | Practical developer empathy, simplification, measurable outcome |  |  |
| J03 | Senior | How do you know whether a platform is actually helping its users? | Outcome metrics, adoption, support burden, delivery impact, user feedback loops |  |  |
| J04 | Senior | How do you gather feedback from platform consumers, and how do you prioritize what to act on? | Structured feedback loops, prioritization, balancing strategy with user pain |  |  |
| J05 | Lead | What is your view on golden paths or paved roads? How would you implement one? | Opinionated but pragmatic platform thinking, enablement at scale, sensible defaults |  |  |
| J06 | Lead | How do you prevent platform sprawl, where too many tools or overlapping capabilities emerge? | Platform coherence, governance, simplification, lifecycle management |  |  |

### K. Public Cloud and Managed Services

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| K01 | Associate | Which public cloud platforms have you worked with, and at what depth? | Honest scope, practical usage, clarity on hands-on versus superficial exposure |  |  |
| K02 | Mid | Describe a system you built or operated that relied on managed cloud services. What worked well and what were the limitations? | Concrete experience, trade-offs, managed service boundaries, operational realism |  |  |
| K03 | Mid | Have you used infrastructure as code to manage public cloud resources? How did you handle environment parity across dev, staging, and prod? | IaC discipline, environment consistency, deployment process awareness |  |  |
| K04 | Senior | How do you manage IAM and access control across cloud resources at scale? | Role design, least privilege, operational manageability, auditing |  |  |
| K05 | Lead | How do you approach cost visibility and optimization for cloud resources used by multiple teams? | FinOps awareness, chargeback or showback thinking, usage transparency, prioritization |  |  |

### L. Infrastructure as Code and Platform Configuration

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| L01 | Associate | Have you used Terraform, Crossplane, or another infrastructure-as-code tool? In what context? | Honest baseline, hands-on scope, understands what they personally operated |  |  |
| L02 | Mid | How do you manage secrets and sensitive variables in infrastructure-as-code without exposing them in version control or pipelines? | Basic security hygiene, separation of secrets, tooling awareness, operational care |  |  |
| L03 | Mid | Describe a time a `terraform apply` or infrastructure change caused an unexpected issue. What happened and what changed afterward? | Change safety, learning, rollback mindset, practical debugging |  |  |
| L04 | Mid | How do you handle Terraform state drift or unexpected divergence between declared and actual infrastructure? | Drift detection, investigation discipline, safe reconciliation, awareness of state management risks |  |  |
| L05 | Senior | How would you describe the difference between managing infrastructure with Crossplane versus Terraform? When would you choose one over the other? | Control-plane vs apply-based thinking, reconciliation model, team workflow fit, abstraction trade-offs |  |  |
| L06 | Senior | Have you worked with Crossplane Compositions or XRDs? What did you find useful or limiting? | Hands-on platform API design thinking, composability, constraints, consumer experience |  |  |
| L07 | Senior | What is your approach to testing infrastructure-as-code before changes reach production? | Validation strategy, plan review, pre-prod checks, policy/testing discipline |  |  |
| L08 | Lead | If you were deciding whether to adopt Crossplane for a new internal platform, what factors would you evaluate? | Strategic evaluation, operational cost, platform API design, team capability, long-term maintainability |  |  |
| L09 | Lead | What are the limits of Terraform, Crossplane, or workflow-driven infrastructure automation in your experience? Where would you avoid each? | Tool judgment, boundaries, operational realism, avoids one-size-fits-all thinking |  |  |

### M. Go (Golang)

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| M01 | Associate | How long have you been writing Go, and what kinds of systems have you built with it? | Honest scope, concrete examples, distinguishes toy use from production use |  |  |
| M02 | Associate | What Go libraries or frameworks have you used for HTTP APIs, gRPC, or service development? | Practical ecosystem familiarity, can explain why they chose them |  |  |
| M03 | Mid | How do you structure a Go project for a production service? What packages, patterns, or conventions do you follow? | Thoughtful code organization, maintainability, team conventions, pragmatic patterns |  |  |
| M04 | Mid | How do you handle errors in Go across service boundaries? What patterns work well for you? | Explicit error handling discipline, observability/context propagation, practical trade-offs |  |  |
| M05 | Mid | What is your approach to writing testable Go code? | Dependency boundaries, interfaces where useful, table-driven tests, pragmatic testability |  |  |
| M06 | Senior | Explain how you use goroutines and channels in practice. Describe a case where concurrency caused a bug. | Real concurrency experience, understands synchronization pitfalls, can debug race conditions or leaks |  |  |
| M07 | Senior | Describe your experience with `controller-runtime` or `client-go` for building Kubernetes controllers. | Go plus Kubernetes depth, controller patterns, reconciliation-specific implementation details |  |  |
| M08 | Senior | How do you profile and optimize a Go service for CPU or memory when needed? | Uses profiling tools, measurement-first approach, avoids premature optimization |  |  |
| M09 | Lead | Describe a time a goroutine leak, memory leak, or runtime behavior issue caused a production problem. How did you lead the investigation and fix? | Runtime debugging depth, calm diagnosis, systemic prevention, leadership under pressure |  |  |

### N. Learning and Adaptability

| ID | Level | Question | What Good Looks Like | Rating | Notes |
| --- | --- | --- | --- | --- | --- |
| N01 | Associate | Tell me about a time you had to learn a new technology or domain quickly to deliver a project. | Learns effectively under pressure, gives concrete steps, reflects on outcome |  |  |
| N02 | Associate | Tell me about a time feedback from a peer or manager changed how you work. | Receptive to feedback, self-awareness, improvement mindset |  |  |
| N03 | Mid | How do you evaluate whether a new tool or framework is worth adopting? | Balanced evaluation, avoids hype, considers team fit and operational cost |  |  |
| N04 | Mid | Describe a project where your original approach turned out to be wrong. What changed your mind? | Intellectual honesty, evidence-based adjustment, learning from mistakes |  |  |
| N05 | Senior | In a role like this, technologies evolve quickly. How do you keep your judgment current without chasing every trend? | Practical learning habits, discernment, principled adoption |  |  |
| N06 | Senior | What is an area of platform or infrastructure engineering you still need to grow in? | Honest self-assessment, curiosity, realistic growth plan |  |  |
| N07 | Lead | Describe a time you had to unlearn a habit or assumption from a previous role because it did not fit the new environment. | Adaptability at senior scope, contextual judgment, ability to recalibrate |  |  |
| N08 | Lead | How do you help a team evaluate and adopt new technology without creating churn or trend-driven decisions? | Leadership in change management, technical judgment, team enablement |  |  |

## 6. Suggested Round Focus

### Technical Screen

- Prioritize sections A, B, C, and F
- Look for hands-on depth and communication clarity
- Use the `Level` column to match the target role
- Add at least 1 scenario or incident question

### Hiring Manager or Final Technical Round

- Prioritize sections B, C, D, F, G, and I
- Probe ownership, architecture judgment, and production maturity
- Use the `Level` column to match the target role
- Add at least 1 scenario question and 1 collaboration question

### Operational or Platform-Focused Round

- Prioritize sections D, E, F, G, and H
- Probe incident handling, workflow reliability, and multi-system reasoning
- Use the `Level` column to match the target role

## 7. Red Flags and Strong Signals

### Strong Signals

- Gives specific examples with clear personal ownership
- Understands reconciliation, idempotency, and failure recovery
- Can reason about production systems with real trade-offs
- Treats observability and operations as first-class engineering work
- Balances abstraction with practical delivery

### Red Flags

- Speaks only in general theory without hands-on examples
- Cannot explain failure handling or production incidents clearly
- Confuses Kubernetes usage with control-plane design experience
- Focuses only on feature delivery and ignores reliability concerns
- Avoids trade-offs or claims there were none