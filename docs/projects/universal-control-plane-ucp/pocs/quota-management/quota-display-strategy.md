---
title: "Quota Display Strategy"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota Display Strategy

## Context

During the initial PoC implementation, we discovered that the GCP quota system is
significantly more complex than the single `instances` metric we originally assumed.
This document captures those findings and frames the design decision that must be made
before the quota UI can be considered production-ready.

---

## Finding 1: Quota Metrics Are Numerous and Multi-Dimensional

For a single GCP service, the Cloud Quotas API returns a large number of metrics. Cloud
SQL alone has dozens, spanning multiple categories:

| Category | Example metrics |
|---|---|
| Rate quotas (API calls) | `sqladmin.googleapis.com/connect`, `/get`, `/list`, `/mutate`, `/default`, `/default_per_region` |
| Streaming | `sqladmin.googleapis.com/streamsqldata_concurrent_connections`, `/streamsqldata_bytes_transferred` |
| Operations | `sqladmin.googleapis.com/operations` |

Beyond the metric count, each metric may have **multiple rows** because of dimensions.
For example, `connect` returns one row per region (`asia-east1`, `us-central1`,
`europe-west1`, etc.), and potentially per user. A single service can easily produce
hundreds of rows in the quota table.

This is exactly what the GCP Console Quotas page shows — and why it has a rich filter
bar with service, metric name, and dimension filters.

---

## Finding 2: Not All Quota Data Is Uniformly Available

During the PoC we discovered two categories of quota data with different availability:

### Hard quotas with full API support

These are standard rate or allocation quotas exposed through the Cloud Quotas API with a
proper metric ID. Both the limit and, in some cases, current usage are available
programmatically.

Example: all six Cloud SQL rate quotas (`connect`, `get`, `list`, etc.) have metric IDs
documented and are returned by `quotaInfos.list`.

### Soft limits without API support

Some GCP limits are described in documentation but are **not exposed as Cloud Quotas API
metrics**. They are managed as soft limits via support cases and are only visible in the
console Quotas page through a different mechanism.

Example: **Cloud SQL instances per project** (100 or 1,000 depending on network
architecture). This limit has no metric ID in the Cloud Quotas API. The Cloud Monitoring
metric `sqladmin.googleapis.com/quota/instances/usage` does not exist either — Cloud SQL
does not support quota metrics in Cloud Monitoring at all
(ref: [GCP quota alert docs](https://cloud.google.com/docs/quotas/set-up-quota-alerts)).

Implication: for soft limits, UCP cannot read the limit programmatically. It can only
count actual resources (e.g. by listing Cloud SQL instances via the Admin API) but has no
reliable way to fetch the ceiling to compare against.

---

## Finding 3: The Metric Taxonomy Differs Per Provider

AWS, Azure, GCP, and Omnia all have different quota systems, naming conventions, and
API shapes. There is no universal "instances per project" concept across providers.

What this means for the `QuotaProvider` interface: the interface must abstract over
these differences, but the mapping from GCP metric IDs to something meaningful in the UCP
context must be defined per-provider.

---

## Design Decision: What to Display

Three options, each with trade-offs:

### Option A — Mirror GCP Console (show everything)

Return every `quotaInfo` entry the Cloud Quotas API gives back, for every service the
tenant has enabled. The UI renders them all with filters matching the GCP console.

**Pros:**
- Complete — users see exactly what GCP sees
- No curation effort; automatically picks up new GCP quotas

**Cons:**
- Hundreds of rows for a typical project, most irrelevant to UCP provisioning decisions
- Multi-dimensional metrics produce duplicate-looking rows that are hard to read without
  deep GCP knowledge
- Completely GCP-specific — the UI and data model cannot generalize to AWS or Azure
- Soft limits (like instance count) would still be missing or show 0, causing confusion

### Option B — Curated per-service list (manually define what to show)

Define a static list of metric IDs per GCP service that UCP explicitly monitors. Only
those metrics appear in the UI.

Current PoC state is effectively this option (the `gcpMonitoredQuotas` slice).

**Pros:**
- Clean, focused, easy to read
- Easy to explain to users ("these are the quotas that matter for UCP")

**Cons:**
- High maintenance burden: must manually add entries as new resource types are added
- Requires deep knowledge of each provider's quota system upfront
- Misses quotas the operator cares about but that we haven't anticipated
- Soft limits (no metric ID) cannot be included at all

### Option C — UCP resource-type grouping (middle ground)

Group quotas by **UCP resource type** rather than by GCP service. Each provider
implementation is responsible for mapping its relevant metrics into UCP's resource
categories. The UI groups and filters by UCP resource type (Database, Compute, Storage,
Kubernetes).

Within each resource type, show only the metrics relevant to provisioning decisions for
that type. Rate quotas (API call limits) are surfaced under a separate "API Limits"
category rather than mixed in with resource count quotas.

Concretely for GCP Cloud SQL as "Database":
- Resource count: fetch from Admin API (instance list count) — no monitoring metric
- Limit: attempt Cloud Quotas API; if not available, display as "not available"
- Rate quotas: shown separately under "API Limits" with their documented metric IDs

**Pros:**
- Provider-agnostic framing — AWS and Azure implementations map into the same UCP
  resource type categories
- Focused on what UCP users actually need to make provisioning decisions
- Explicit about what is and is not available (no silent 0s for missing data)
- Rate quotas are visible but not mixed with resource count quotas

**Cons:**
- Still requires mapping work per provider per resource type
- May miss quotas that are important but do not map to a UCP resource type

---

## Recommendation

**Option C** is the right direction for a multi-cloud platform. The quota UI should be
organized around UCP concepts (what resource are you trying to provision?) not around
provider concepts (which GCP service is this metric for?).

However, the full implementation of Option C requires decisions and mapping work that is
out of scope for the current PoC sprint. The PoC should:

1. Implement Option B for GCP Cloud SQL rate quotas (the six documented metrics with real
   metric IDs) — this proves the API connectivity and UI rendering work
2. Show "not available" explicitly when limit or usage cannot be fetched — never show 0
   as if it were real data
3. Document the resource-type mapping table as the extension point, leaving it empty for
   non-Cloud-SQL resources until the mapping decision is made

The full Option C design (resource-type grouping, multi-provider mapping table, handling
of soft limits) should be a follow-on design spike before moving the quota feature to
production.

---

## Open Questions

1. **Rate quotas vs. resource quotas in the UI**: Should API rate quotas (req/min) be
   shown in the same quota page as resource count quotas (instances, clusters)? They serve
   different audiences — rate quotas matter for operations, resource quotas for capacity
   planning. Consider separate tabs or groupings.

2. **Soft limits with no API support**: For quotas like Cloud SQL instances per project
   where GCP does not expose the limit via API, should UCP show the usage-only row (count
   from Admin API, limit = "contact GCP support"), or omit the row entirely until the
   limit is available?

3. **Dimension handling**: For metrics that vary by region, which dimension's value should
   UCP display? The default (global) value, the value for the tenant's configured region,
   or all dimensions as separate rows?

4. **Staleness and caching**: Cloud Monitoring rate quota usage reflects API call rates
   over the last minutes. Cloud Quotas API limits are near-static. Should UCP cache the
   limit separately from the usage to reduce API calls?

5. **Cross-provider quota view**: When a tenant has both GCP and Omnia resources, should
   the quota page show quotas from both providers in one view, or is it per-provider?
