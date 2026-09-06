---
name: Onboard a client app and issue an API key
description: Register a client application, contract it to a published API through a plan, handle the approval workflow, register it with the gateway and retrieve its API key.
api: openapi/apiman-organizations-api-openapi.yml
operations: [createClient, createClientVersion, getApiVersionPlans_1, createContract, approveContract, performAction, getClientApiKey, updateClientApiKey, getClientVersionContracts, getApiRegistryJSON, deleteContract]
generated: '2026-09-06'
method: generated
source: openapi/_original/apiman-openapi.json + https://www.apiman.io/apiman-docs/user-guide/latest/manager/consuming-apis.html
---

# Onboard a client app and issue an API key

A **Contract** is the three-way join at the centre of Apiman: a *Client version* consumes
an *API version* through a *Plan version*. Creating one grants access; breaking one
revokes it.

## Steps

1. **Create the client identity.** `createClient` (`NewClientBean`) in the consuming
   organization. This needs the *Client App Developer* or *Organization Owner* role.

2. **Create a client version.** `createClientVersion` (`NewClientVersionBean`). Apiman
   mints an API key for this version.

3. **Pick a plan.** `getApiVersionPlans_1` on the target API version lists the plans it
   offers. A plan version must be **locked** to be consumable.

4. **Create the contract.** `createContract` (`NewContractBean`: target org, API, API
   version, plan). If the plan sets `requiresApproval`, the contract lands pending and the
   API provider must call `approveContract` (`POST /actions/contracts`) before it is live.
   `404` here means the client version — not the API — was not found; read the message.

5. **Register the client with the gateway.** `performAction` with
   `{"type":"registerClient", …}`. Until this runs, the contract exists in the manager but
   the gateway will not honour the key.

6. **Retrieve the key.** `getClientApiKey` returns `ApiKeyBean.apiKey`. Callers present it
   to the gateway when invoking the managed API. `getApiRegistryJSON` (or
   `getApiRegistryXML`) returns the full set of endpoints plus the key — the single most
   useful call for wiring up a consumer.

7. **Verify.** `getClientVersionContracts` lists everything this client version consumes.

## Rotation and revocation

- **Rotate:** `updateClientApiKey` sets a new key. The old key is not recoverable, and the
  operation answers `409` if the client version is in the wrong status. Re-register the
  client afterwards so the gateway picks up the change.
- **Revoke one:** `deleteContract` ("Break Contract").
- **Revoke all:** `deleteAllContracts` ("Break All Contracts").
- **Take the client offline:** `performAction` with `type: unregisterClient` — the
  documented inverse of `registerClient`.

No published time window applies to any of these; they take effect on the next gateway
sync.

## Cautions

- The client API key authenticates traffic **through the gateway**, not calls to the
  Manager REST API. Those are different surfaces with different credentials.
- There is no idempotency key. A retried `createContract` after a network timeout may
  produce a second contract — list with `getClientVersionContracts` before retrying.
