---
name: registry-m2m
description: |
  Integrate with the DigiQuip Registry REST API using Logto machine-to-machine
  credentials. Organization tenancy for :own scopes is resolved by Registry from
  Logto app org membership—not from JWT claims. Use for sync jobs, webhooks, or
  services that read or write people, assets, actions, identifiers, or related data.
---

# Registry M2M integration

Integrate through the **public REST API** only. Treat OpenAPI as the contract; these instructions are guardrails and workflow.

## Before you write code

1. Confirm **environment** and base URL—see **`reference.md`**. **Staging**, **test**, and **`test-registry.kvipp.io`** are the **same** non-production stack; production is `registry.kvipp.io` only.
2. Load the customer onboarding checklist (local copy of `templates/customer-onboarding.md`): **token URL**, **`resource` (audience)**, **scopes**, **organization UUID(s)**, and intended use cases.
3. Fetch **`{base_url}/openapi.json`** and use **`{base_url}/docs`** to verify paths, methods, query parameters, and schemas. Do not invent endpoints.
4. Implement **token acquisition** and refresh before any data calls.

Human-oriented domain docs: [DigiQuip developers](https://digiquip.no/en/developers/) (structure, identifiers, organizations, actions, asset types and assets).

**Contract:** M2M access is governed by the customer’s main agreement plus [API and integration terms](https://digiquip.no/en/legal/api-terms/) (remediation of faulty registry writes, including third-party compensation, may be invoiced). Do not promise unlimited free cleanup to partners.

## Authentication (Logto client credentials)

Exchange M2M credentials for an access token at **`{logto_host}/oidc/token`**. Use the **token URL** and **`resource=`** from onboarding—see **`reference.md`** (staging vs production differ; staging audience is usually `http://test-registry.kvipp.io`, not the production URL).

Example (production-shaped; do not copy into staging without changing host and `resource`):

```http
POST https://login.digiquip.com/oidc/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=<CLIENT_ID>
&client_secret=<CLIENT_SECRET>
&resource=https://registry.kvipp.io
```

Logto has **two hostnames per environment** (custom domain and `*.logto.app` tenant URL). They are the same tenant; DigiQuip may give you either for admin work. For Registry API tokens, use the pair documented in onboarding—often the **custom domain** token endpoint so the JWT matches what Registry expects.

You normally **do not** need `…/oidc` discovery or `…/oidc/jwks` unless you validate JWTs yourself; Registry validates the bearer token on each API call.

Send API requests with:

```http
Authorization: Bearer <access_token>
```

Tokens are short-lived. Cache the token until shortly before `expires_in` elapses, then request a new one. On **401**, refresh the token once before failing the job.

Store `client_id` and `client_secret` in environment variables (see `examples/env.example`). Never commit secrets.

## Organization access — NOT in the JWT

**This confuses almost every new M2M integrator. Read before debugging `:own` scopes.**

Logto access tokens for M2M contain only standard OIDC claims, for example `aud`, `client_id`, `exp`, `iat`, `iss`, `jti`, `scope`, and `sub`. There is **no** `allowed_organizations` claim (and no other organization list) in the token body Logto returns from `POST …/oidc/token`.

| Question | Answer |
| -------- | ------ |
| Should we see org UUIDs when we decode the JWT? | **No.** That is expected. |
| Is there a `resource` or query parameter to request org in the token call? | **No.** Only `grant_type`, `client_id`, `client_secret`, and `resource` (API audience). |
| Does a new token “pick up” org after DigiQuip changes Logto? | Fetch a **new** token after provisioning, but org still will **not** appear in the JWT. Test with a **Registry API call**, not JWT decode. |
| Where does tenancy come from? | On **each Registry request**, after JWT validation, Registry calls Logto Management API: `GET /applications/{app_id}/organizations` for M2M apps (`sub` equals `client_id`). That list becomes internal `allowed_organizations` (Logto org ids; Registry maps to Registry UUIDs). |
| What must DigiQuip configure? | The M2M **application must be a member of the customer organization** in Logto (staging or production tenant). Scopes alone are not enough for `:own` on identifiers. |
| How do we verify it works? | `GET {registry}/organizations/{registry_org_uuid}/identifiers?limit=1` with `Authorization: Bearer …` — or another `:own` route documented in OpenAPI. **403** or empty where data should exist → missing scope, app not in org, or wrong org UUID. |

**Do not** tell customers or agents to “check `allowed_organizations` in the token.” Point them to **`reference.md`** (organization tenancy) and a concrete Registry test call.

Copy Registry **organization UUID(s)** from onboarding into API paths and identifier bodies (`organization_id`). DigiQuip links the Logto app to the matching Logto organization; you do not pass org id to the token endpoint.

## Authorization (scopes)

Scopes are space-separated strings in the JWT `scope` claim. DigiQuip provisions only what your integration needs.

Read **`scopes.md`** in this skill directory for:

- `:own` vs `:all` vs `all:everything`
- Which dimensions use organization tenancy
- Dimensions that do not use `:own` in Logto

**Rules for agents:**

- Assume **least privilege**. Only call endpoints your provisioned scopes allow.
- For **`:own`** (and implicit own-read where applicable), restrict reads and writes to organizations Registry resolves into **`allowed_organizations`** (from Logto app ↔ org membership—not from JWT claims). Use Registry **organization UUIDs** from the customer onboarding doc in URLs and payloads.
- Do not assume `all:everything` unless onboarding explicitly says so.
- **403** usually means missing scope or wrong org context—not a bug to “work around” by trying another path.

Staff-style headers such as `X-Digiquip-Organization-Id` are for DigiQuip products, not typical partner M2M clients—do not use them unless DigiQuip documents otherwise for your client.

## API usage conventions

### OpenAPI first

For every endpoint you implement:

1. Find the operation in OpenAPI.
2. Match HTTP method, path, required query parameters, and request body schema.
3. Handle documented error responses (409 for conflicts, 404 for missing resources, 422 for validation).

### Pagination and list responses

List endpoints often support `skip` and `limit` (and sometimes `order_by`). Paginated responses may include headers such as `X-Total-Count`, `X-Page`, `X-Per-Page`, and `X-Total-Pages`. Loop with `skip` until you have retrieved the full result set or reach your integration’s safety cap.

### Paths and trailing slashes

Registry routes can be sensitive to trailing slashes. If a call returns **404** or **307** unexpectedly, compare the path to OpenAPI exactly—including a trailing `/` on collection routes when the spec shows one.

### External identifiers

Prefer **identifiers** (`source` + `identity` pointing at a Registry UUID) to tie your system’s IDs to Registry records across merges and data quality improvements. See [identifiers](https://digiquip.no/en/developers/identifiers/).

Create and resolve identifiers within the scopes you were granted. Use stable `source` values agreed with the customer (often their system name).

**Assets are special:** equipment does not belong to an organization on the asset record. Customer fleet access uses identifiers (typically `source=internal_number`). Read **`registry-assets/SKILL.md`** before implementing asset sync or org-scoped equipment lists.

### Data quality mindset

The Registry is a **passive system of record**: allow incremental data entry, use external IDs for stable links, and expect enrichment over time—not perfect upfront records.

## Recommended implementation order

1. Token helper (fetch, cache, refresh).
2. Health check: small **read** call allowed by your scopes (e.g. list with `limit=1`).
3. Identifier strategy: map your IDs ↔ Registry UUIDs.
4. Core sync (create/update) for the entities in scope.
5. Idempotency: handle 409, use identifiers or GET-before-POST where appropriate.
6. Logging: log correlation IDs if your HTTP client exposes them; never log tokens or secrets.

## Testing

- Develop and test against **`https://test-registry.kvipp.io`** (staging = test) until explicitly approved for production.
- Use realistic but non-production data.
- After code changes, re-run against OpenAPI: agents often drift path or field names.

## Troubleshooting

| Symptom | Likely cause |
| ------- | ------------- |
| 401 on API | Expired token, wrong `resource` / audience, wrong environment (staging token on prod Registry), token from wrong Logto hostname, or malformed `Authorization` header |
| 403 | Missing scope, or `:own` access outside `allowed_organizations` |
| 404 | Wrong UUID, wrong path, or trailing slash mismatch |
| 409 | Unique constraint or duplicate identifier—make the operation idempotent |
| Empty lists with 200 | Valid but no data, or org filter excludes rows—check scopes and filters |
| “No org in JWT” after Logto change | **Not a bug.** Orgs are never in the JWT; verify Logto app org membership and call Registry |
| 403 on `…/identifiers` with `:own` scopes | M2M app not linked to org in Logto, wrong Registry org UUID, or missing scope |

## Out of scope for this skill

- Registry Development MCP or direct database access.
- DigiQuip internal services (Operator, Kvipp admin, Synchronizer) unless a separate skill is added for them.
- Provisioning M2M apps or scopes—contact [support@digiquip.no](mailto:support@digiquip.no).

## Related files

- `scopes.md` — authorization model
- `reference.md` — environment URLs and quick links
- Repository `templates/customer-onboarding.md` — per-customer scopes and orgs
