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

| Environment | Registry API (HTTPS) | Logto token URL (`POST …/oidc/token`) | OAuth `resource` (audience) | M2M client |
| ----------- | -------------------- | ------------------------------------- | --------------------------- | ---------- |
| Staging / test | `https://test-registry.kvipp.io` | e.g. `https://devlogin.digiquip.com/oidc/token` | e.g. `http://test-registry.kvipp.io` | Test client (yes/no) |
| Production | `https://registry.kvipp.io` | e.g. `https://login.digiquip.com/oidc/token` | e.g. `https://registry.kvipp.io` | Prod client (yes/no) |

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

Registry **organization UUIDs** this client may access (DigiQuip provisions these—e.g. one UUID per customer org such as a rental company). Required for fleet/identifier work and `:own` scopes:

```text
(one UUID per line)
```

### Endpoints or workflows explicitly in scope

Check or list what this integration is allowed to do:

- [ ] Read people
- [ ] Write people
- [ ] Read assets / asset types
- [ ] Write assets / asset types
- [ ] Read or write actions
- [ ] Read or write identifiers
- [ ] Other: ___________

**Equipment fleet (typical):** see recommended scopes in `.agents/skills/registry-assets/SKILL.md` — identifiers `read|create|update|delete:own` plus assets `read|create|update|delete:all`, with org UUID(s) in `allowed_organizations`.

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
