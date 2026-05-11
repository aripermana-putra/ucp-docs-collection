---
title: "Quota Management PoC Findings"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota Management PoC Findings

## The Most Important Finding: Requirements Must Come First

The single most critical takeaway from this PoC is that **the technical feasibility of
quota management cannot be fully assessed without knowing what the user actually needs to
see.**

We built the PoC assuming "show quota usage and limits, similar to the GCP Console." But
the GCP Console quota page exists to serve GCP operators managing a project. UCP tenants
have a different context — they are provisioning resources through an abstraction layer
and may not know what a rate quota or a quota metric ID is.

Until we answer "what does a UCP tenant need to know about quotas, and when?", we cannot
know:
- Which metrics to fetch
- Which limits actually matter for provisioning decisions
- Whether the data we need is even available from the provider APIs
- Whether the same quota page design works across GCP, AWS, and Azure

The PoC proved the API plumbing works. It did not — and cannot — answer the product
question. That question must be answered first.

---

## Finding 1: GCP Quota Data Is Far More Complex Than Anticipated

The original PoC plan assumed a small, flat list of quota metrics (one per resource type).
In practice:

- A single GCP service (Cloud SQL) exposes dozens of quota metrics
- Each metric can produce multiple rows due to **dimensions** — for example, the `connect`
  metric has a separate limit per GCP region, which means a single metric becomes 20+
  rows in a project that has instances across regions
- A typical GCP project with a few services enabled will have hundreds of quota rows in
  total

This is not a problem with GCP's API. It is a mismatch between the assumption we started
with and the reality of how cloud provider quota systems are structured.

The implication: we cannot simply "fetch all quotas and display them." Without curation,
the quota page becomes as complex as the GCP Console itself, which defeats the purpose of
having UCP as an abstraction layer.

---

## Finding 2: Not All Quota Data Is Available via API

Two classes of GCP limits exist, and only one is available programmatically:

| Class | Example | Limit via API | Usage via API |
|---|---|---|---|
| Hard quota with metric ID | Cloud SQL `connect` rate (1,000 req/min) | Yes — Cloud Quotas API | Potentially — Cloud Monitoring (where supported) |
| Soft limit without metric ID | Cloud SQL instances per project (100–1,000) | **No** | Partial — count via Admin API, but no ceiling to compare against |

**Cloud SQL instance count** — the quota that matters most for UCP's pre-provision gate —
falls in the second category. GCP does not expose this limit via the Cloud Quotas API.
It is a soft limit managed via support cases. There is no metric ID for it.

Additionally, **Cloud SQL does not support quota usage metrics in Cloud Monitoring at
all** (confirmed in GCP documentation). The metric
`sqladmin.googleapis.com/quota/instances/usage` does not exist. This was the original
assumption in the PoC plan and it was wrong.

The pre-provision gate for Cloud SQL databases cannot be implemented as originally
designed — there is no programmatic way to read the instance limit to compare against.

---

## Finding 3: Rate Quotas Are Largely Irrelevant to UCP Tenants

The six Cloud SQL quota metrics that *do* have proper metric IDs
(`connect`, `get`, `list`, `mutate`, `default`, `default_per_region`) are all **rate
quotas** — limits on how many times the Cloud SQL Admin API can be called per minute.

These are not meaningful for UCP tenant capacity planning because:

1. They limit Admin API call frequency, not the number of databases a tenant can have
2. **Using a Cloud SQL database (running SQL queries) does not count against these
   quotas.** Database traffic goes over a direct TCP connection to the instance and never
   touches the Admin API. The `connect` quota specifically measures calls to
   `GetConnectSettings` and `GenerateEphemeralCertificate` — used by the Auth Proxy when
   establishing connections — not the SQL queries that flow through those connections.
3. A typical tenant using UCP to manage a handful of databases will never approach these
   rate limits. They become relevant only for automation that calls the Admin API at very
   high frequency.

Showing these metrics prominently in the UCP quota page would likely confuse tenants and
suggest concerns they do not have.

---

## Finding 4: The GCP Console Is Not the Right Reference

The original goal — "show quotas similar to the GCP Console" — was the wrong anchor. The
GCP Console quota page is designed for GCP project administrators who need to audit and
manage all API quotas across all services. That is a different job than what a UCP tenant
needs.

A UCP tenant needs to answer one question: **"Can I provision more of resource X?"**

That question requires:
1. How many of resource X do I currently have?
2. What is my ceiling for resource X?

Neither question maps cleanly to a GCP quota metric — as Finding 2 and Finding 3 show,
the data that matters most (instance count vs. limit) is either not a quota metric at all,
or is a soft limit with no API support.

---

## What the PoC Did Prove

Despite the gaps above, the PoC established several things that are valid regardless of
which direction requirements take:

- The `QuotaProvider` interface correctly abstracts GCP-specific logic behind a
  provider-agnostic contract. Adding AWS or Azure quota support means implementing the
  interface, not changing the API handler.
- The Cloud Quotas API is callable with the tenant's existing stored GCP credentials. No
  additional credential setup is needed.
- The quota UI component (table, usage bar, service filter, tenant selector) renders
  correctly and can be pointed at any data the backend returns.
- The `fetchCloudQuotasLimit` function correctly queries and parses the Cloud Quotas API
  response when the metric ID is correct.

---

## What Must Be Decided Before This Feature Moves Forward

1. **What does a UCP tenant actually need to know about quotas?**
   Is it "can I create more databases?" (allocation), or "am I hitting API limits?"
   (rate), or both? The answer determines which data to fetch.

2. **How do we handle limits that GCP does not expose via API?**
   For Cloud SQL instance count: do we count current instances via the Admin API and
   show the limit as "contact your GCP administrator"? Or do we not show it until GCP
   provides a proper API?

3. **Is the quota page provider-specific or provider-agnostic?**
   If a tenant has both GCP and Omnia resources, do they see one unified quota view or
   separate tabs per provider? This affects the data model significantly.

4. **Which resource types matter for the pre-provision gate?**
   The pre-provision gate is the highest-value outcome of this PoC. But it only works
   for allocation quotas with API-readable limits. We need to identify which resource
   types GCP exposes as hard quotas with proper metric IDs before wiring the gate.
