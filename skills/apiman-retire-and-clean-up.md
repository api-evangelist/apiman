---
name: Retire and clean up Apiman entities safely
description: Unwind an Apiman deployment in the correct order — break contracts, unregister clients, retire APIs, then delete — and know which steps cannot be undone.
api: openapi/apiman-organizations-api-openapi.yml
operations: [getApiVersionContracts, getClientVersionContracts, deleteContract, deleteAllContracts, performAction, deleteApi, deleteClient, deletePlan, deleteOrg, deleteApiDefinition, deleteApiImage, revoke, revokeAll, exportData, importData]
generated: '2026-09-06'
method: generated
source: openapi/_original/apiman-openapi.json + https://www.apiman.io/changelog.html
---

# Retire and clean up Apiman entities safely

Apiman enforces deletion preconditions and answers **409** when sub-elements are still
active — "still-published APIs" under an organization, "still-registered ClientVersions"
under a client. That 409 is the safety rail. Unwind in this order.

## Order of operations

1. **Snapshot first.** `exportData` (`GET /system/export`) is the only restore path
   Apiman gives you. Per-entity deletes have **no undo**. `importData`
   (`POST /system/import`) restores a whole instance, not one object.

2. **Find what depends on it.** `getApiVersionContracts` (consumers of an API version) or
   `getClientVersionContracts` (what a client consumes).

3. **Break contracts.** `deleteContract` for one, `deleteAllContracts` for every contract
   on a client version. This revokes access at the gateway.

4. **Unregister clients.** `performAction` with `type: unregisterClient` — the inverse of
   `registerClient`.

5. **Retire APIs.** `performAction` with `type: retireAPI` — the inverse of `publishAPI`.
   Retiring removes the API from the gateway. As of 3.1.2.Final an organization holding
   *retired* entities can be deleted; before that it could not.

6. **Delete.** `deleteApi`, `deleteClient`, `deletePlan`, then `deleteOrg`. A `409` at any
   step means you skipped one above.

7. **Revoke people.** `revoke` removes a single role from a user in an organization;
   `revokeAll` removes every membership they hold there.

## What cannot be undone

| Action | Reversal |
|---|---|
| `publishAPI` | `retireAPI` |
| `registerClient` | `unregisterClient` |
| `createContract` / `approveContract` | `deleteContract` |
| `grant` | `revoke` / `revokeAll` |
| `lockPlan` | **none** — create a new plan version |
| `updateClientApiKey` | **none** — the old key is gone |
| `deleteOrg` / `deleteApi` / `deleteClient` / `deletePlan` | **none** — only a full `importData` restore |

No time window is published for any reversal. Do not assume a grace period.

## Smaller cleanups

- `deleteApiDefinition` removes the stored spec from an API version without touching the
  version.
- `deleteApiImage` removes the API's image.

## Audit

Everything you did is recorded as `AuditEntryBean` rows readable through `getOrgActivity`,
`getApiActivity`, `getApiVersionActivity`, `getClientActivity`, `getPlanActivity` and the
per-user `getActivity`. There is no request-id correlation header on this API, so the
audit trail is the only after-the-fact trace.
