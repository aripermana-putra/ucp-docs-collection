---
title: "Implementation"
space: UCP
parent_page_id: "../universal-control-plane-ucp.md"
---

# Implementation

Engineering decisions and design specifications for the UCP production implementation.

---

## Documents

### [Repository Structure](./implementation/repo-structure.md)
Decision document for how `api-server`, `temporal-worker`, and `cli` are organized
across Git repositories. Compares umbrella-with-submodules, true monorepo, and polyrepo
with quantitative workflow analysis. **Status: Proposed.**

### [Backend Architecture](./implementation/backend-architecture.md)
Concrete design specification for the Go backend rewrite. Covers architecture pattern,
HTTP framework selection, dependency injection, error handling, logging, fault tolerance,
health checks, metrics and tracing, API contract approach, and response format.

### [HTTP Framework: chi vs echo](./implementation/http-framework.md)
In-depth comparison of chi and echo for the spec-first, `oapi-codegen` strict-mode
context. Covers error handling integration, middleware, context model, testing
ergonomics, and AI code generation consistency.

### [Go Code Guidelines](./implementation/go-code-guidelines.md)
Code-level engineering practices for the Go backend. Covers error handling patterns,
context propagation, logging, dependency injection, validation, handler structure,
Temporal workflow design, and CLI conventions.
