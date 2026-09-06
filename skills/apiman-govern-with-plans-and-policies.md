---
name: Govern an API with plans and policies
description: Build a service tier out of Apiman policies, lock it, inspect the resolved policy chain, and understand which policy level wins.
api: openapi/apiman-organizations-api-openapi.yml
operations: [list_2, getPolicyDefs, getAvailablePlugins, create_2, createPlan, createPlanVersion, createPlanPolicy, listPlanPolicies_1, updatePlanPolicy, reorderPlanPolicies, performAction, createApiPolicy, listApiPolicies_1, reorderApiPolicies, createClientPolicy, getApiPolicyChain, probeContractPolicy]
generated: '2026-09-06'
method: generated
source: openapi/_original/apiman-openapi.json + https://www.apiman.io/apiman-docs/user-guide/latest/manager/data-model.html
---

# Govern an API with plans and policies

A **policy** is Apiman's unit of governance, executed at runtime as a chain in front of
the backend. A **plan** is a named set of policies representing a level of service — the
Gold-1000-per-day / Silver-500-per-day pattern from Apiman's own docs.

## Know your policy catalogue first

- `list_2` (`GET /policyDefs`) — every policy type this Apiman knows about.
- `getPolicyDefs` — the policy definitions contributed by one installed plugin.
- `getAvailablePlugins` / `create_2` — browse and install plugins (admin only; `403` if
  you are not `apiadmin`). Official plugins ship on Maven Central under
  `io.apiman.plugins` — JWT, Keycloak OAuth, transformation, circuit breaker, API key,
  HTTP security, header allow/deny, URL whitelist, JSONP.

Each `PolicyDefinitionBean` carries a `form`/`formType` describing the shape of its
`configuration` blob. Read it before authoring a policy; the configuration is
policy-specific and unvalidated by the generic schema.

## Build the tier

1. `createPlan` (`NewPlanBean`), then `createPlanVersion`.
2. `createPlanPolicy` for each policy — e.g. a rate-limiting policy at 1000/day for Gold
   and 500/day for Silver. Same policy type, different `configuration`.
3. `listPlanPolicies_1` to confirm, `updatePlanPolicy` to adjust, `reorderPolicies`
   (`reorderPlanPolicies`) to set execution order. **Order matters** — the chain is
   evaluated in sequence.
4. `performAction` with `type: lockPlan`.

> **Locking is irreversible by design.** Apiman's docs: locking exists "so that API
> providers can't change the details of the plan out from underneath the client app
> developers who are using it." There is no unlock. To change a locked plan, create a new
> plan version. This is the one action in this skill with no reversal path — see
> `conventions/apiman-conventions.yml` → `reversibility`.

## The three levels

Policies attach at three levels and all three end up in one chain:

| Level | Create with | List with |
|---|---|---|
| API | `createApiPolicy` | `listApiPolicies_1` |
| Plan | `createPlanPolicy` | `listPlanPolicies_1` |
| Client | `createClientPolicy` | `listClientPolicies` |

Resolve the actual runtime order with `getApiPolicyChain`
(`…/plans/{planId}/policyChain`) — call it before publishing rather than reasoning about
precedence from memory. `probeContractPolicy` inspects one policy's live state on a
contract without modifying it.

## Notes

- Rate limiting and quota here are limits **you** impose on **your** consumers. They are
  not limits on the Apiman Manager API, which publishes none. See
  `rate-limits/apiman-rate-limits.yml`.
- Rate-limit and quota state needs a backing store: Elasticsearch, Redis, JDBC, Hazelcast,
  Vert.x shared data, filesystem or in-memory.
