# Changelog

All notable changes to integration skills and templates in this repository are documented here.

## Unreleased

### Changed

- **`registry-m2m`**: staging Logto tenant is periodically reset; M2M access there is not durable; Registry org UUIDs persist.
- **`customer-onboarding.md`**: separate Registry organization UUID (and client id) for staging and production.
- **`reference.md`** / **`SKILL.md`**: `X-Digiquip-Organization-Id` for staff impersonation in DigiQuip apps vs M2M; Kontoret does not send org header for normal users.
- **`registry-catalog-and-asset-types`**: `create:asset_types:own` vs org junction linking after `POST /asset_types/`.
- **`registry-digiquip-qr-nfc`**: who registers codes, no issued-code allow-list, GET lookup outcomes with `:own` scopes.
- **`registry-assets`** / **`registry-m2m/scopes.md`**: partner M2M does not get `update:assets` / `delete:assets`; shared DB; corrections via support; planned `:sub` tenancy.
- **`registry-catalog-and-asset-types`** / **`scopes.md`**: catalog `brands` / `models` **read** needs no Logto scopes (`IMPLICIT_GLOBAL_READ_DIMENSIONS`).

## 0.3.1 — 2026-10-27

### Changed

- **`registry-m2m`**: prominent **organization access — NOT in the JWT** section; Logto app org membership vs `:own` scopes; verify via Registry API, not token decode.
- **`reference.md`**, **`scopes.md`**, **`registry-assets`**, **`customer-onboarding.md`**, **`AGENTS.md`**: align tenancy docs (remove misleading “org in JWT” guidance).

## 0.3.0 — 2026-10-26

### Added

- **`registry-digiquip-qr-nfc`** skill: physical QR/NFC as `digiquip_qr` identifiers, release via null `organization_id`, equipment landing licensing.
- **`registry-catalog-and-asset-types`** skill: search-first brand/model/asset type flow; catalog read-only on Registry M2M; catalog writes only via Operator identity session.

### Changed

- **`registry-m2m`**: document that staging = test; Logto custom domain vs tenant URL; per-environment token URL and OAuth `resource`; organization UUID onboarding; remove outdated `auth.digiquip.no` example; expand **`reference.md`**.
- **`templates/customer-onboarding.md`** and **`examples/env.example`**: environment table with Logto and `resource` columns; API terms acknowledgment.
- **`registry-assets`**: recommended M2M scope checklist (identifiers `:own`, assets `:all`, including update and delete); cross-link to catalog/asset type skill.
- **`registry-m2m/scopes.md`**: brands and models are read-only for partners; no catalog create scopes on M2M.
- **`registry-catalog-and-asset-types`** / **`scopes.md`**: clarify `:all` (not `:own`) for catalog; resolve/preview vs `/brands/`; no implicit catalog read from other dimensions.

## 0.2.0 — 2026-10-25

### Added

- **`registry-assets`** skill: assets without org owner; fleet access via identifiers (`internal_number`).

### Changed

- **`registry-m2m`** and **`scopes.md`**: cross-links to asset model; onboarding template notes internnummer.

## 0.1.0 — 2026-10-25

### Added

- Initial `registry-m2m` skill, scope reference, customer onboarding template, and `env.example`.
- `AGENTS.md`, README, and MIT license.
