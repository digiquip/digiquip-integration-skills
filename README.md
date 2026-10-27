# digiquip-integration-skills

Agent skills and onboarding templates for integrating with DigiQuip APIs. This repository is for **customers and partners** building machine-to-machine (M2M) integrations—not for DigiQuip internal product development.

## What is here

| Path | Purpose |
| ---- | ------- |
| [`AGENTS.md`](AGENTS.md) | Entry point for coding agents working in a clone of this repo |
| [`.agents/skills/registry-m2m/`](.agents/skills/registry-m2m/) | Registry API integration skill (auth, scopes, OpenAPI-first workflow) |
| [`.agents/skills/registry-assets/`](.agents/skills/registry-assets/) | Assets: global equipment records, org access via identifiers |
| [`.agents/skills/registry-digiquip-qr-nfc/`](.agents/skills/registry-digiquip-qr-nfc/) | DigiQuip QR/NFC labels, `digiquip_qr`, landing page licensing |
| [`.agents/skills/registry-catalog-and-asset-types/`](.agents/skills/registry-catalog-and-asset-types/) | Shared brand/model catalog (M2M read-only), asset type resolution, identity session funnel |
| [`templates/customer-onboarding.md`](templates/customer-onboarding.md) | Per-customer checklist (scopes, orgs, environments)—fill in locally; do not commit secrets |
| [`examples/env.example`](examples/env.example) | Environment variable names for M2M credentials |

Human-readable API documentation lives on [digiquip.no developers](https://digiquip.no/nb/developers/) (Norwegian) and [/en/developers/](https://digiquip.no/en/developers/) (English).

## Using these skills in your project

1. Copy the skill into your integration codebase, or add this repository as a submodule.
2. Point your agent at **`AGENTS.md`** (or the skill path below) so it loads Registry integration rules before writing API code.
3. Copy [`templates/customer-onboarding.md`](templates/customer-onboarding.md) to your project (e.g. `docs/digiquip-onboarding.md`) and fill in the scopes and organization IDs DigiQuip provisioned for you.
4. Copy [`examples/env.example`](examples/env.example) to `.env` and add your Logto M2M credentials. Never commit `.env`.

Recommended skill path for agents:

```text
.agents/skills/registry-m2m/SKILL.md
```

If you only copy one folder, copy **`.agents/skills/registry-m2m/`** and keep `scopes.md` beside `SKILL.md`.

## Registry environments

**Staging** and **test** refer to the **same** non-production environment (`test-registry.kvipp.io`). See [`.agents/skills/registry-m2m/reference.md`](.agents/skills/registry-m2m/reference.md) for Logto hostnames, token URLs, and OAuth `resource` values.

| Environment | Base URL | OpenAPI |
| ----------- | -------- | ------- |
| Staging / test | `https://test-registry.kvipp.io` | `https://test-registry.kvipp.io/openapi.json` |
| Production | `https://registry.kvipp.io` | `https://registry.kvipp.io/openapi.json` |

Interactive docs: append `/docs` to the base URL.

Token endpoint and M2M setup: [Machine-to-machine access](https://digiquip.no/en/developers/machine-to-machine/). Contact [support@digiquip.no](mailto:support@digiquip.no) for credentials.

## Versioning

Current release: **v0.3.2** (see [CHANGELOG.md](CHANGELOG.md)).

Tag releases when integration guidance changes (scope model, auth URLs, or breaking API conventions). Pin your integration project to a tag when you need stable agent instructions.

## Contributing (DigiQuip)

Changes to skills should stay aligned with the live Registry API and published developer docs. When in doubt, **`openapi.json` for the target environment is authoritative** for endpoints and schemas; skills describe process and guardrails, not a full API listing.
