---
title: "Core Data UCP Subscription Discovery"
space: UCP
parent_page_id: "../pocs.md"
---

# Core Data — UCP Subscription Discovery

Proves whether the Horizon Core Data API can serve as the source of truth for UCP service subscription status and user role resolution, eliminating the need for a UCP-maintained `ucp_registered_tenants` table.

---

## Research Question

Can `GET /v0/members/{email}/tenants?subscriptions=true` reliably determine which of a user's ROC tenants are subscribed to the UCP service, and how should the `ucp tenants list` command combine this with JWT `groups` claim data to show the user's UCP role per tenant?

---

## Sub-docs

| Doc | Purpose |
|---|---|
| [design.md](core-data-ucp-subscription/design.md) | Scope, hypothesis, approach, success criteria |
| [poc-report.md](core-data-ucp-subscription/poc-report.md) | Verdict and recommendation |
| [implementation.md](core-data-ucp-subscription/implementation.md) | What was built and tested |
| [evidence.md](core-data-ucp-subscription/evidence.md) | Core Data API reference and external sources |
