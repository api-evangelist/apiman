---
name: Publish an API through Apiman
description: Create an organization, register an API, attach its definition, verify readiness and publish it to a gateway using the Apiman Manager REST API.
api: openapi/apiman-organizations-api-openapi.yml
operations: [getInfo, getOrganizations, createOrg, getOrg, list, test, create_1, createApi, createApiVersion, updateApiDefinitionFromURL, updateApiDefinition, getApiVersionPlans_1, getApiVersionStatus, performAction]
generated: '2026-09-06'
method: generated
source: openapi/_original/apiman-openapi.json + https://www.apiman.io/apiman-docs/user-guide/latest/manager/providing-apis.html
---

# Publish an API through Apiman

Apiman is self-hosted. The base URL is your own deployment: `https://{apiman_host}/apiman`
(the project documents `http://localhost:8080/apiman` as the local default). There is no
vendor host and no vendor sandbox — see `sandbox/apiman-sandbox.yml` for the Docker
Compose quickstart.

## Before you start

- **Auth.** The whole Manager API is protected by the operator's Keycloak. Get an OIDC
  access token for the `apiman` client and send `Authorization: Bearer <token>`. The
  published OpenAPI declares **no** `securitySchemes`, so do not conclude from the spec
  that the API is open — it is not. See `authentication/apiman-authentication.yml`.
- **Confirm who you are** with `getInfo` (`GET /users/currentuser/info`) and what you may
  do with `getPermissionsForUser`. API work needs the *API Developer* or *Organization
  Owner* role in the target organization.
- **No idempotency.** There is no `Idempotency-Key` on this surface. Do not blindly retry
  a POST; on a duplicate create you will get `409`, which is your signal that the first
  call landed. See `conventions/apiman-conventions.yml`.

## Steps

1. **Find or create the organization.**
   `getOrganizations` lists yours. `getOrg` fetches one; `404` means it does not exist.
   `createOrg` creates it (`NewOrganizationBean`). You are granted Organization Owner
   automatically.

2. **Make sure a gateway exists.**
   `list` (`GET /gateways`) shows registered gateways. If you are adding one, validate the
   configuration first with `test` (`PUT /gateways`) — a real dry run — then `create_1`.
   Publishing fails without at least one running, registered gateway.

3. **Create the API identity.** `createApi` (`NewApiBean`) inside the organization. The
   `id` you choose is the path segment used from here on.

4. **Create a version.** `createApiVersion` (`NewApiVersionBean`) with the backend
   `endpoint`, `endpointType` and whether it is `publicAPI`. `409` means that version
   string already exists — pick another or fetch it with `getApiVersion_1`.

5. **Attach the definition.** `updateApiDefinitionFromURL` pulls the spec from a URL;
   `updateApiDefinition` uploads the body directly. Apiman stores OpenAPI, Swagger and
   WSDL — `getApiDefinition_2` will serve it back as `application/json`,
   `application/x-yaml` or `application/wsdl+xml`.

6. **Attach plans (unless the API is public).** `getApiVersionPlans_1` lists what is
   attached; add plan bindings through `updateApiVersion`, and order them with
   `reorderApiPlans`. Only **locked** plan versions can be attached. A non-public API with
   no plan cannot be published.

7. **Check readiness before acting.** `getApiVersionStatus` returns an
   `ApiVersionStatusBean` listing exactly what is still blocking publication. Always call
   this before step 8 — it is cheaper than a failed action.

8. **Publish.** `performAction` (`POST /actions`) with
   `{"type":"publishAPI","organizationId":…,"entityId":…,"entityVersion":…}`.
   Success is `204` with no body.

## Reversal

`performAction` with `type: retireAPI` is the documented inverse of `publishAPI`. No time
window is published for it, so treat the window as unknown rather than unlimited. See
`conventions/apiman-conventions.yml` → `reversibility`.

## Error handling

Apiman's spec binds **no schema** to any 4xx, so branch on the status code:

- `403` — authenticated but not permitted; re-check `getPermissionsForUser`.
- `404` — one of the composite path segments is wrong. The message does not say which;
  resolve outward-in (`getOrg` → `listApis` → `listApiVersions_1`).
- `409` — the version already exists, or a precondition is unmet.
- No `5xx` is documented anywhere in the contract; treat any as retryable-with-backoff.
