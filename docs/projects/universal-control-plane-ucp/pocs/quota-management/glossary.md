---
title: "Quota Management Glossary"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota Management Glossary

Terms used across the quota management documentation, defined in plain language.

---

## Rate Quota

A limit on **how frequently** you can call a management API — measured in requests per
unit of time (usually per minute). Exceeding a rate quota means your API call was
rejected because you sent too many requests too fast, not because you have too many
resources.

**Example:** Cloud SQL's `connect` rate quota is 1,000 requests/min. If your application
calls the Cloud SQL Admin API more than 1,000 times in a minute, GCP returns HTTP 429.
The next minute, the counter resets and you can call again.

**Important:** using a Cloud SQL database — running SQL queries, reading and writing data
— does **not** count against rate quotas. Database traffic goes directly over a TCP
connection to the database instance itself. It never passes through the Admin API.
The Cloud SQL `connect` rate quota specifically measures calls to `GetConnectSettings`
and `GenerateEphemeralCertificate`, which are called by the Auth Proxy when opening a
new connection — not the SQL queries that flow through that connection afterward.

Rate quotas are transient — they self-recover. They matter to operations teams monitoring
API call volume, not to tenants asking "how many databases can I create?"

---

## Allocation Quota (Resource Quota)

A limit on **how many instances of a resource** can exist at one time. Unlike rate quotas,
allocation quotas do not reset — if you are at the limit, you must delete an existing
resource before you can create a new one.

**Example:** Cloud SQL has a limit of 100–1,000 instances per project. Once you reach
that ceiling, any attempt to create another Cloud SQL instance will be rejected by GCP
regardless of how infrequently you make API calls.

Allocation quotas are the kind UCP cares most about for pre-provision checks — they
directly determine whether a tenant can provision more resources.

Also called "resource quota" in some contexts. GCP's own documentation uses "allocation
quota" in the Cloud Monitoring metric naming (`quota/allocation/usage`).

---

## Soft Limit

A quota limit that is not enforced by a single hard API boundary, but is instead a
policy value managed manually by the cloud provider (typically via a support case
request). Soft limits often do not have a machine-readable metric ID in the quota APIs.

**Example:** Cloud SQL instances per project (100 or 1,000 depending on network
architecture) is a soft limit. It appears in the GCP Console Quotas page, but is not
returned by the Cloud Quotas API as a queryable metric. To increase it, you file a
support case with Google. There is no API to read the current limit programmatically.

Contrast with a hard quota, where the limit is returned by the quota API and enforced
automatically by GCP's infrastructure.

---

## Hard Quota

A quota limit that is programmatically readable via a quota API and automatically
enforced by the cloud provider's infrastructure. No manual intervention is needed for
enforcement — GCP will reject the request the moment usage hits the limit.

**Example:** Compute Engine CPUs per region is a hard quota. You can read its current
value via the Cloud Quotas API (`compute.googleapis.com/cpus`), and GCP will return an
error immediately if you try to create a VM that would exceed it.

---

## Quota Metric

The unique identifier for a specific quota within a cloud provider's quota system.
In GCP this is a dot-separated path that includes the service domain and the resource
name.

**Format:** `{service-domain}/{resource-name}`

**Examples:**
- `sqladmin.googleapis.com/connect` — Cloud SQL API connect rate quota
- `compute.googleapis.com/cpus` — Compute Engine CPU allocation quota
- `container.googleapis.com/clusters` — GKE cluster count quota

Quota metrics are the keys used when querying the Cloud Quotas API. Not every GCP limit
has a quota metric — soft limits (like Cloud SQL instance count) do not.

---

## Dimension

A qualifier that scopes a quota metric to a specific context such as a region, zone, or
user. A single quota metric can have different limit values for different dimensions.

**Example:** The `sqladmin.googleapis.com/connect` metric has a separate limit row for
each GCP region where Cloud SQL is available (`asia-east1`, `us-central1`,
`europe-west1`, etc.). Each row can have a different limit value.

This is why a single service can produce hundreds of rows in the quota table — one per
metric per dimension combination.

---

## Quota Limit

The ceiling value for a given quota metric (and dimension). This is the maximum allowed
usage. In GCP, limits are read from the Cloud Quotas API (`quotaInfos.list`).

---

## Quota Usage

The current consumption against the limit. For rate quotas, usage is the request rate
over the current time window. For allocation quotas, usage is the count of existing
resources.

In GCP, usage for some metrics is available via Cloud Monitoring time series queries. Not
all services expose usage metrics in Cloud Monitoring — Cloud SQL does not, for example.

---

## Cloud Quotas API

The GCP API for reading and requesting changes to quota limits
(`cloudquotas.googleapis.com`). GA since Q1 2024. Returns `QuotaInfo` objects containing
the metric ID, display name, current limit, and dimensional breakdowns.

This is the source for **limits** in UCP's quota listing.

Endpoint: `GET /v1/projects/{project}/locations/global/quotaInfos`

---

## Cloud Monitoring API

The GCP API for querying time series metrics, including quota usage metrics where
available (`monitoring.googleapis.com`). Returns data points over a time range.

This is the intended source for **usage** in UCP's quota listing — but only works for
services that expose quota usage as monitoring metrics. Cloud SQL does not; Compute
Engine does.

Endpoint: `GET /v3/projects/{project}/timeSeries`

---

## QuotaInfo

The object returned per quota metric by the Cloud Quotas API. Contains:
- `metric` — the quota metric ID (e.g. `sqladmin.googleapis.com/connect`)
- `quotaDisplayName` — human-readable name
- `service` — the GCP service this quota belongs to
- `dimensionInfos` — list of dimension-scoped limit values (one per region, etc.)
- `isPrecise` — whether the limit is an exact value or an approximation

---

## QuotaPreference

A Cloud Quotas API resource used to request a quota limit increase or adjustment. Not
part of the current PoC scope — UCP reads quotas but does not submit increase requests.
