# Registry API scopes (integrator reference)

DigiQuip issues OAuth scopes on M2M access tokens via Logto. The Registry enforces them on REST endpoints. This document explains the **model**; your exact scope string is listed in your onboarding checklist.

**If this document disagrees with your token or OpenAPI operation security requirements, trust the token and OpenAPI.**

## Scope shape

Most data dimensions use:

```text
{operation}:{dimension}:{level}
```

| Part | Examples | Meaning |
| ---- | -------- | ------- |
| `operation` | `read`, `create`, `update`, `delete` | HTTP-style permission |
| `dimension` | `people`, `assets`, `actions`, `identifiers`, … | Resource family |
| `level` | `own`, `all` | Tenancy breadth |

Some elevated scopes use a two-part form, e.g. `all:subscriptions`, which gates subscription-related dimensions.

### `all:everything`

Broad internal-style access. **Rarely issued to partners.** When present, it still does **not** grant access to all sensitive surfaces (for example organization secrets and some subscription writes have additional gates).

Do not design an integration that requires `all:everything` unless DigiQuip explicitly approved it.

## `:own` vs `:all`

| Level | Behavior (simplified) |
| ----- | ------------------------ |
| `:all` | Cross-tenant access on that dimension (subject to endpoint rules). |
| `:own` | Limited to organizations Registry resolves into **`allowed_organizations`** (see below—not a JWT claim). |

For M2M, Registry fills **`allowed_organizations`** from Logto (**application ↔ organization** membership), not from anything inside the access token.

### Implicit read on some dimensions

For several **Pattern A** dimensions, authenticated callers may **read** within their allowed organizations even without an explicit `read:{dimension}:own` scope—but access is still bounded by `allowed_organizations`. Writes always require the appropriate `create` / `update` / `delete` scope.

Pattern A dimensions include (among others): `people`, `actions`, `asset_types`, `identifiers`, `groups`, `checklists`, `organizations`, `skills`, `acquired_skills`, `lessons`, `policies`, `authorities`, and related organization-scoped data.

### Dimensions without `:own` in Logto

Some dimensions do **not** have a meaningful org-tenancy scope in Logto on the entity itself. Effective access is typically `:{operation}:{dimension}:all` or scoped via related data. Examples include **`assets`**, **`documents`**, **`brands`**, and **`models`**.

**Assets:** there is no `*:assets:own` that filters the asset table by organization. A customer’s machines are linked through **identifiers** (often `source=internal_number` with `organization_id`). See **`../registry-assets/SKILL.md`**.

**Brands and models (catalog):** the shared manufacturer/model catalog is **read-only** on standard partner M2M tokens. Use **`read:brands:all`** and **`read:models:all`** for `/brands/` and nested model routes. Catalog data is global—**`:own` does not scope brands or models** to `allowed_organizations`, and implicit Pattern A read (e.g. `read:people:own` without `read:brands`) does **not** unlock the catalog. DigiQuip does **not** issue `create` / `update` / `delete` on catalog dimensions to integrators. New catalog rows go through controlled product flows (Operator **equipment identity session**), not direct Registry catalog writes. See **`../registry-catalog-and-asset-types/SKILL.md`**.

If your onboarding lists only `:own` scopes on a dimension that does not support `:own`, contact DigiQuip support—your client may need different scopes.

## Common dimensions (not exhaustive)

Use OpenAPI and your onboarding doc as the final list for **your** client.

| Dimension | Typical integrator use |
| --------- | ---------------------- |
| `people` | Workers, profiles, org membership |
| `asset_types` | Equipment type rows (brand + model); distinct from catalog entities |
| `brands` / `models` | Shared catalog (read for partners) |
| `assets` | Serialised equipment |
| `actions` | Inspections, controls, performed work |
| `identifiers` | Stable cross-system keys |
| `organizations` | Tenants, metadata (within granted access) |
| `groups` | Groupings, equipment categories (aliases may apply) |
| `checklists` | Checklist templates and structure |

Subscription and billing dimensions (`subscriptions`, `subscription_addons`) are usually restricted and not part of standard equipment/person sync integrations.

## Organization context

Partner M2M integrations should rely on:

- **`scope`** in the JWT (the only authorization list Logto puts in the token)
- **Logto app organization membership** (DigiQuip configures; Registry loads via Management API per request)
- **Registry organization UUIDs** from onboarding in paths and bodies
- API filters documented in OpenAPI (e.g. `organization_id` query parameters where supported)

Do not assume you can access every organization in Registry with a narrow M2M profile.

## Checking what you have

**JWT decode (debugging scopes only):** inspect `scope` — space-separated permission strings. You will **not** see `allowed_organizations` or org UUIDs in the token. That is normal.

**Verify org tenancy:** call Registry, e.g.  
`GET /organizations/{registry_org_uuid}/identifiers?limit=1` with your bearer token. See **`SKILL.md`** (organization access) and **`reference.md`** (organization tenancy).

In production code, rely on **API responses** and integration tests—not on org claims in the JWT.

## Requesting changes

To add a scope or organization, contact [support@digiquip.no](mailto:support@digiquip.no) with:

- The operation (read/create/update/delete)
- The dimension
- Why `:own` is enough or why `:all` is required
- Which Registry organization UUIDs the app must access (Logto org membership for the M2M application)
