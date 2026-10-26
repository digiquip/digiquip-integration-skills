# Changelog

All notable changes to integration skills and templates in this repository are documented here.

## Unreleased

### Added

- **`registry-digiquip-qr-nfc`** skill: physical QR/NFC as `digiquip_qr` identifiers, release via null `organization_id`, equipment landing licensing.
- **`registry-catalog-and-asset-types`** skill: search-first brand/model/asset type flow; catalog read-only on Registry M2M; catalog writes only via Operator identity session.

### Changed

- **`registry-assets`**: recommended M2M scope checklist (identifiers `:own`, assets `:all`, including update and delete); cross-link to catalog/asset type skill.
- **`registry-m2m/scopes.md`**: brands and models are read-only for partners; no catalog create scopes on M2M.

## 0.2.0 — 2026-10-25

### Added

- **`registry-assets`** skill: assets without org owner; fleet access via identifiers (`internal_number`).

### Changed

- **`registry-m2m`** and **`scopes.md`**: cross-links to asset model; onboarding template notes internnummer.

## 0.1.0 — 2026-10-25

### Added

- Initial `registry-m2m` skill, scope reference, customer onboarding template, and `env.example`.
- `AGENTS.md`, README, and MIT license.
