---
title: "UCP Assistant"
space: UCP
parent_page_id: "../universal-control-plane-ucp.md"
---

# UCP Assistant

## Overview

UCP Assistant is an internal AI assistant for the Universal Control Plane (UCP)
documentation domain. It provides a consistent interface for asking questions
about UCP and related internal platforms by retrieving and citing trusted
documentation at answer time.

The assistant operates as a documentation copilot, not as a general-purpose
chatbot. Every answer is grounded in indexed source material, and every answer
ends with references to the documents used.

The primary interface is a CLI application. The same assistant engine also
supports other interfaces such as Slack, Teams, and web clients through a
session-aware backend.

---

## Scope

UCP Assistant covers operational and platform knowledge distributed across
multiple trusted internal documentation sources.

Initial source domains:

- UCP documentation
- OneCloud documentation

Primary interface:

- CLI chat application

Additional source domains:

- runbooks
- architecture documents
- onboarding guides
- RFCs and ADRs
- API references
- exported wiki content
- PDF documentation bundles

---

## Documents

- [Design](ucp-assistant/design.md) — architecture, grounding model, session model,
  source model, answer contract, and interface model
- [Implementation Plan](ucp-assistant/implementation-plan.md) — execution order,
  deliverables, interfaces, rollout sequence, and operational milestones

---

## Core Characteristics

- grounded retrieval on every answer
- mandatory references at the end of every answer
- session-based conversations
- multi-source indexing
- read-only source synchronization
- shared backend for CLI, API, and messaging clients

---

## Related

- [Universal Control Plane (UCP)](../universal-control-plane-ucp.md)
