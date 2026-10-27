# Agent instructions

Read and follow **`.agents/skills/`** when building or maintaining integrations against DigiQuip APIs.

## Skills

| Skill | Path | When |
| ----- | ---- | ---- |
| **`registry-m2m`** | `.agents/skills/registry-m2m/` | Any Registry integration: M2M auth, scopes, REST calls, pagination |
| **`registry-assets`** | `.agents/skills/registry-assets/` | Equipment (assets): no org owner on asset, fleet via identifiers (`internal_number`) |
| **`registry-digiquip-qr-nfc`** | `.agents/skills/registry-digiquip-qr-nfc/` | Printed QR / NFC tags: `digiquip_qr` identifiers, release/adopt, equipment landing |
| **`registry-catalog-and-asset-types`** | `.agents/skills/registry-catalog-and-asset-types/` | Merke/modell catalog (read on M2M), search-first `asset_type_id`; catalog writes only via Operator identity session |

Start with **`registry-m2m/SKILL.md`**. **Never expect organization UUIDs or `allowed_organizations` inside the M2M JWT**—Registry resolves org tenancy from Logto per API request. When the work involves **assets or equipment fleets**, also read **`registry-assets/SKILL.md`**. When resolving **brand, model, or asset type** before creating equipment, read **`registry-catalog-and-asset-types/SKILL.md`**. For **physical DigiQuip QR or NFC codes**, read **`registry-digiquip-qr-nfc/SKILL.md`**. Use **`registry-m2m/scopes.md`** for authorization semantics. Use **`templates/customer-onboarding.md`** for customer-specific scopes and organization IDs (local copy; no secrets in git).

## Sources of truth

| Topic | Where |
| ----- | ----- |
| Concepts (people, assets, actions, identifiers) | [digiquip.no developers](https://digiquip.no/nb/developers/) |
| M2M token flow | [Machine-to-machine](https://digiquip.no/en/developers/machine-to-machine/) |
| Endpoints, request/response shapes | Registry **`/openapi.json`** and **`/docs`** for the environment you target |
| Agent guardrails and integration workflow | This repository (`.agents/skills/`) |

Do **not** use the Registry **Development MCP** (`/mcp` in dq-registry) for customer integrations. Integrate only via the public REST API and the scopes provisioned in Logto.

## Secrets

- Store `client_id`, `client_secret`, and tokens in environment variables or a secret manager.
- Never commit credentials, JWTs, or filled onboarding documents that contain secrets.
- Use **staging / test** (`https://test-registry.kvipp.io`) first unless the task explicitly requires production. Staging and test are the same environment—see **`registry-m2m/reference.md`** for Logto hostnames and OAuth `resource` values.

## When editing this repository

- Keep skills concise; link to developer docs and OpenAPI instead of duplicating reference material.
- Update **`CHANGELOG.md`** when guidance changes in a way integrators should notice.
- Norwegian and English developer pages on digiquip.no remain the canonical human docs; skills may be English-only for maintainability.
