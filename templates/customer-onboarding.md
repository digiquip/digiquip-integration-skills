# DigiQuip Registry — customer onboarding (local copy)

Fill in this template when DigiQuip provisions M2M access. Keep it in **your** integration repository or password manager attachments—not in public git with secrets.

Copy to e.g. `docs/digiquip-onboarding.md` and add that path to `.gitignore` if it will contain internal notes you do not want published.

## Integration summary

| Field | Value |
| ----- | ----- |
| Customer / project name | |
| Contact at DigiQuip | |
| Date provisioned | |
| Integration purpose (1–2 sentences) | |

## Environments

| Environment | Registry base URL | Use M2M credentials |
| ----------- | ----------------- | --------------------- |
| Test | `https://test-registry.kvipp.io` | Test client (yes/no) |
| Production | `https://registry.kvipp.io` | Prod client (yes/no) |

## Logto M2M application

| Field | Where stored (not in git) |
| ----- | ------------------------- |
| Client ID | e.g. `.env` → `DIGIQUIP_CLIENT_ID` |
| Client secret | e.g. secret manager / `.env` |
| Token resource (`resource=` in token request) | |
| Notes from DigiQuip | |

## Scopes granted

Paste the exact scope string or list from DigiQuip:

```text
(paste scopes here)
```

### Allowed organizations (`:own` / tenancy)

Registry organization UUIDs this client may access:

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

## Identifier convention

| Field | Value |
| ----- | ----- |
| `source` string for your system | e.g. `erp-companyname` |
| Example `identity` in your system | |
| Example Registry UUID (test) | |

## Operational notes

- Rate limits or batching guidance from DigiQuip:
- Expected sync direction (inbound / outbound / both):
- Maintenance window or contact for incidents:

## Agent instruction snippet

When using an AI agent on this integration, point it to:

1. DigiQuip skill: `registry-m2m` from [digiquip-integration-skills](https://github.com/DigiQuip/digiquip-integration-skills) (or your vendored copy under `.agents/skills/registry-m2m/`)
2. This filled onboarding document for scopes and organization IDs
3. OpenAPI at `{base_url}/openapi.json` for the environment you are targeting
