# DigiQuip Registry — customer onboarding (local copy)

Fill in this template when DigiQuip provisions M2M access. Keep it in **your** integration repository or password manager attachments—not in public git with secrets.

Copy to e.g. `docs/digiquip-onboarding.md` and add that path to `.gitignore` if it will contain internal notes you do not want published.

## Integration summary

| Field | Value |
| ----- | ----- |
| Customer / project name | |
| Contact at DigiQuip | |
| Date provisioned | |
| API terms acknowledged (see [API and integration terms](https://digiquip.no/en/legal/api-terms/)) | yes / date |
| Integration purpose (1–2 sentences) | |

## Environments

**Staging** and **test** mean the same non-production stack (`test-registry.kvipp.io`). Production is separate.

| Environment | Registry API (HTTPS) | Registry org UUID | Logto token URL (`POST …/oidc/token`) | OAuth `resource` (audience) | M2M client provisioned |
| ----------- | -------------------- | ----------------- | ------------------------------------- | --------------------------- | ---------------------- |
| Staging / test | `https://test-registry.kvipp.io` | *(fill in—Registry staging)* | e.g. `https://devlogin.digiquip.com/oidc/token` | e.g. `http://test-registry.kvipp.io` | yes / no / date |
| Production | `https://registry.kvipp.io` | *(fill in—Registry production)* | e.g. `https://login.digiquip.com/oidc/token` | e.g. `https://registry.kvipp.io` | yes / no / date |

**DigiQuip:** always record **both** Registry organization UUIDs. Staging and production are different tenants and often different UUIDs for the same customer name.

**Staging Logto** is reset on a regular schedule (not durable). Re-provision M2M app, scopes, and app ↔ org membership after a reset; **Registry org UUID in staging stays the same.**

Logto **tenant URL** (console / Management API): staging `https://k9rv6k.logto.app`, production `https://4wgi1e.logto.app` — same tenant as the custom domain. DigiQuip may reference either hostname; use the token URL and `resource` from the table above unless support specifies otherwise.

## Logto M2M application

| Field | Where stored (not in git) |
| ----- | ------------------------- |
| Client ID | e.g. `.env` → `DIGIQUIP_CLIENT_ID` |
| Client secret | e.g. secret manager / `.env` |
| Token endpoint | e.g. `.env` → `DIGIQUIP_TOKEN_ENDPOINT` |
| Token resource (`resource=` in token request) | e.g. `.env` → `DIGIQUIP_TOKEN_RESOURCE` |
| Notes from DigiQuip | |

## Scopes granted

Paste the exact scope string or list from DigiQuip:

```text
(paste scopes here)
```

### Allowed organizations (`:own` / tenancy)

**Not in the JWT.** Logto tokens only carry `scope` (and standard OIDC claims). Registry loads org access from **Logto: M2M app ↔ organization membership**. DigiQuip configures that in Logto; integrators verify with a Registry API call, not by decoding the token.

Registry **organization UUID** (for API paths and `organization_id` on identifiers)—**fill in both environments**:

| Environment | Registry organization UUID |
| ----------- | ------------------------ |
| Staging / test | |
| Production | |

Additional org UUIDs (if the client may access more than one tenant):

```text
(optional: one UUID per line, note staging vs production)
```

Logto M2M application id (client id)—**per environment** (staging credentials do not survive Logto resets):

| Environment | Client ID (`sub` / `client_id`) |
| ----------- | ------------------------------- |
| Staging / test | |
| Production | |

After provisioning: test e.g. `GET {registry}/organizations/{uuid}/identifiers?limit=1` with a fresh bearer token for **that** environment’s UUID and credentials.

### Endpoints or workflows explicitly in scope

Check or list what this integration is allowed to do:

- [ ] Read people
- [ ] Write people
- [ ] Read assets / asset types
- [ ] Write assets / asset types
- [ ] Read or write actions
- [ ] Read or write identifiers
- [ ] Other: ___________

**Equipment fleet (typical):** see recommended scopes in `.agents/skills/registry-assets/SKILL.md` — identifiers `read|create|update|delete:own` plus assets `read|create|update|delete:all`, with Logto app ↔ org membership for the Registry org UUID above.

## Identifier convention

| Field | Value |
| ----- | ----- |
| Org fleet / internnummer (`source`) | Usually `internal_number` — confirm with DigiQuip |
| `source` for your ERP / system id | e.g. `erp-companyname` (separate from `internal_number` if both are used) |
| Example `identity` (internnummer) | |
| Example `identity` in your ERP | |
| Example Registry asset UUID (test) | |

For assets, the link to your organization is the **identifier** (with `type=asset` and `organization_id`), not an owner field on the asset. See skill **`registry-assets`** in this repository.

## Operational notes

- Rate limits or batching guidance from DigiQuip:
- Expected sync direction (inbound / outbound / both):
- Maintenance window or contact for incidents:

## Agent instruction snippet

When using an AI agent on this integration, point it to:

1. DigiQuip skill: `registry-m2m` from [digiquip-integration-skills](https://github.com/DigiQuip/digiquip-integration-skills) (or your vendored copy under `.agents/skills/registry-m2m/`)
2. This filled onboarding document for scopes and organization IDs
3. OpenAPI at `{base_url}/openapi.json` for the environment you are targeting
