---
name: Read Apiman usage metrics and audit history
description: Pull per-API, per-client and per-plan usage and response statistics out of Apiman, search the catalogue, and read the immutable activity trail.
api: openapi/apiman-organizations-api-openapi.yml
operations: [getUsage, getUsagePerClient, getUsagePerPlan, getResponseStats, getResponseStatsSummary, getResponseStatsPerClient, getResponseStatsPerPlan, getClientUsagePerApi, getOrgActivity, getApiActivity, getApiVersionActivity, getClientActivity, getPlanActivity, getActivity, searchApis_2, searchClients, searchOrgs, searchUsers, getStatus]
generated: '2026-09-06'
method: generated
source: openapi/_original/apiman-openapi.json
---

# Read Apiman usage metrics and audit history

## Metrics

Seven read-only operations hang off an API version under
`…/apis/{apiId}/versions/{version}/metrics/`:

| Operation | Returns |
|---|---|
| `getUsage` | `UsageHistogramBean` — request counts bucketed over a time range |
| `getUsagePerClient` | `UsagePerClientBean` — which consumers drove the traffic |
| `getUsagePerPlan` | `UsagePerPlanBean` — which service tier drove the traffic |
| `getResponseStats` | `ResponseStatsHistogramBean` — success/failure/error over time |
| `getResponseStatsSummary` | `ResponseStatsSummaryBean` — one rolled-up figure |
| `getResponseStatsPerClient` | `ResponseStatsPerClientBean` |
| `getResponseStatsPerPlan` | `ResponseStatsPerPlanBean` |

From the consumer side, `getClientUsagePerApi` gives a client version's usage broken down
by the APIs it calls.

These read from whatever metrics store the deployment is configured with —
Elasticsearch, InfluxDB, Prometheus, or a JSON log file (the `write-to` option added in
3.1.0.Final). **If the operator configured no metrics store, these return empty rather
than failing.** Check `getStatus` (`GET /system/status`, `SystemStatusBean`) before
concluding an API has no traffic.

## Search

`searchApis_2`, `searchClients`, `searchOrgs`, `searchUsers`, `searchRoles` and
`searchApiCatalog` are **POST with a body**, not GET with query parameters. The body is a
`SearchCriteriaBean`:

```json
{
  "filters": [{"name": "name", "value": "payments", "operator": "like"}],
  "orderBy": {"name": "name", "ascending": true},
  "paging":  {"page": 1, "pageSize": 25}
}
```

Operators: `bool_eq`, `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`. Responses are
`SearchResultsBean<T>` envelopes. Note that the plain list endpoints (`listApis`,
`listClients`, `listPlans`, `listMembers`) take **no** paging parameters and return the
whole collection — only `/search/*` is paged.

## Audit trail

Every entity exposes an activity feed of `AuditEntryBean` records
(`who`, `entityType`, `entityId`, `entityVersion`, `createdOn`, `what`, `data`):
`getOrgActivity`, `getApiActivity`, `getApiVersionActivity`, `getClientActivity`,
`getClientVersionActivity`, `getPlanActivity`, `getPlanVersionActivity`, and `getActivity`
for a single user. This is the system of record for who changed what — there is no
request-id correlation header on this API.

## Notes

- All of these are `GET` and safe to retry.
- `403` means the caller lacks view permission on the organization; check with
  `getPermissionsForUser`.
