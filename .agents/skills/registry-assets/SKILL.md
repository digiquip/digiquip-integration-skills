---
name: registry-assets
description: |
  Model and integrate Registry assets (physical equipment) and how customer access
  works without organization ownership on the asset record. Use when listing, creating,
  or syncing equipment, linking ERP IDs, or filtering a fleet by organization.
---

# Registry assets and organization access

Read this skill when your integration touches **assets** (individual pieces of equipment) or ties them to a **customer organization**. Pair it with **`registry-m2m`** for auth and scopes.

Human docs: [Asset types and assets](https://digiquip.no/en/developers/asset_types_and_assets/), [External identifiers](https://digiquip.no/en/developers/identifiers/).

## Core idea: an asset has no owner organization

In the Registry, an **asset** is a physical object identified mainly by **serial number** and **asset type** (brand, model, optional version). The asset record is **not** owned by a single organization.

That is intentional. In the real world many parties relate to the same machine over its lifetime—manufacturer, importer, dealer, operating company, lessor, and others. The Registry does not try to pick one “owner” on the asset row.

**Do not** look for `organization_id` on the asset as the way to scope your customer’s fleet. **Do not** assume `:own` scopes on the `assets` dimension behave like `people` or `actions` with Pattern A org filtering on the asset table—assets are a special case.

## How your customer’s equipment becomes visible (`:own`)

Tenancy for equipment is expressed through **identifiers**, not through an owner field on the asset.

For many integrations, the customer’s internal fleet number uses:

| Field | Typical value |
| ----- | ------------- |
| `source` | `internal_number` |
| `identity` | The customer’s internal number (your “internnummer”) |
| `type` | `asset` |
| `foreign_id` | Registry UUID of the asset |
| `organization_id` | Registry UUID of the customer organization |

Creating or maintaining that identifier is what **links** the global asset to **your** organization for list filters, org-scoped identifier APIs, and downstream products (for example Kvipp entry via identifier).

Confirm the exact `source` string in your onboarding checklist. DigiQuip may also use other sources (`digiquip_qr`, integration-specific sources, or your own `source` for ERP IDs—see identifiers docs). **`internal_number`** is the standard pattern for org-scoped fleet numbers.

### Identifiers are polymorphic

Identifiers can point to `person`, `asset`, or `asset_type`. For equipment integrations you usually work with **`type: asset`** (and sometimes **`asset_type`** for catalog links).

## Mental model (short)

```text
Asset (global)  ←── foreign_id ──  Identifier (org-scoped when source = internal_number)
     │
     └── asset_type_id → Asset type (catalog; may be linked to orgs separately)
```

Your stable business key is often **`(source, identity)`** or **`(organization_id, source, identity)`** for `internal_number`—not only the asset UUID. UUIDs remain correct after data merges; identifiers preserve your links when you use them consistently.

## Recommended workflows

### 1. Resolve or create the asset

Assets need an **`asset_type_id`** (brand + model in the Registry sense). **Search the shared catalog first**, then resolve or create an asset type—see **`registry-catalog-and-asset-types`**. Partners do **not** create catalog brands or models via Registry; missing catalog rows require DigiQuip stewardship or an identity-session product flow.

Common paths:

- **Preferred for data quality:** catalog search → `GET /asset_types/resolve` or `create_preview` → **`POST /assets`** with `asset_type_id`.
- **`POST /asset_types`** then **`POST /assets`** when onboarding grants `create:asset_types` and catalog already matches.
- **`POST /assets/find?create=true`** — convenient but can create types without the same controls as search-first; use only when DigiQuip explicitly accepts that trade-off for your sync.

Always confirm exact paths and bodies in OpenAPI for your environment.

### 2. Attach the organization link (identifier)

After you have the asset UUID:

1. Check whether an identifier already exists (avoid duplicates—developer docs stress this for QR and other sources).
2. Create an identifier with `foreign_id` = asset id, `type` = `asset`, `organization_id` = customer org, and the agreed `source` / `identity`.

For **`internal_number`**, Registry requires **`organization_id`** and **`type`** on create/lookup (see OpenAPI and identifier endpoints).

### 3. List or sync “our machines”

To list equipment for one organization, use asset list endpoints that filter **via identifiers**, for example:

- `organization_id` query parameter on asset list routes that document identifier-based filtering
- `organization_association_type` — default **`internal_number`** limits to identifiers with `source=internal_number`; **`all`** includes other identifier sources for that org

Read the route description in OpenAPI—filtering is implemented through the Identifiers table (`foreign_id` = asset, `type` = asset, matching `organization_id`).

### 4. Keep your ERP id separately (optional)

You may also maintain a second identifier with your own `source` (for example `erp_acme`) and `identity` = primary key in your system. That supports [find-by-identifier](https://digiquip.no/en/developers/identifiers/) and Kvipp deep links without embedding Registry UUIDs in your URLs.

| Purpose | Typical pattern |
| ------- | ---------------- |
| Org fleet / internnummer | `source=internal_number`, `organization_id` set |
| Your system’s primary key | Custom `source` agreed with customer, `identity` = your id |
| Physical DigiQuip QR / NFC | `source=digiquip_qr`, `identity` = code on label/tag — see **`registry-digiquip-qr-nfc`** |

## Recommended scopes (M2M)

Provision scopes in Logto on the customer’s M2M application. The token must also list the customer’s Registry **organization UUID(s)** in **`allowed_organizations`** when using `:own` on identifiers—without that, org-scoped identifier access is empty.

Logto does **not** issue `*:assets:own`. Only **`:all`** exists on the `assets` dimension. Org-specific “ownership” of equipment is always through **identifiers** (`:own` on `identifiers`).

### Standard fleet integration (one org, internnummer)

Use this as the default profile when an integration creates assets, links them with `internal_number`, lists the org fleet, and maintains those links over time.

| Scope | Why you need it |
| ----- | ---------------- |
| `read:identifiers:own` | List and look up identifiers for your org (including `internal_number`); org-nested routes such as `GET /organizations/{id}/identifiers` |
| `create:identifiers:own` | Create the `internal_number` (and other org-scoped) links to assets |
| `update:identifiers:own` | Change identifier fields (e.g. correct `identity`, metadata) without recreating the asset |
| `delete:identifiers:own` | Remove an org’s link to an asset (does not delete the global asset row) |
| `read:assets:all` | `POST /assets/find` (read path), `POST /assets/find_by_serial_number`, and other asset reads that call the assets dimension |
| `create:assets:all` | `POST /assets/find?create=true`, `POST /assets/`, merge flows that require `create:assets` |
| `update:assets:all` | `PUT` / `PATCH` on assets when you change serial number, status, asset type, etc. |
| `delete:assets:all` | Retire or remove asset records when your process requires Registry deletes (uncommon for typical fleet sync—confirm with DigiQuip) |

Copy-paste checklist for onboarding docs:

```text
read:identifiers:own
create:identifiers:own
update:identifiers:own
delete:identifiers:own
read:assets:all
create:assets:all
update:assets:all
delete:assets:all
```

### Often needed alongside assets

| Scope | When |
| ----- | ---- |
| `read:brands:all`, `read:models:all` | Catalog search (read-only; no catalog writes on M2M) |
| `read:asset_types:own` or `read:asset_types:all` | Resolve/list asset types after brand/model are known |
| `create:asset_types:all` or `:own` | `POST /asset_types` when allowed—after **`registry-catalog-and-asset-types`** search rules |

Asset types use organization junction tenancy (`:own` is valid). They are separate from the asset row.

### Broader access (unusual for a single customer)

| Scope | Effect |
| ----- | ------ |
| `read:identifiers:all` / `create:identifiers:all` / … | Cross-tenant identifier access—only for hub or operator-style clients |
| `all:everything` | Internal-style; not a default for partners |

DigiQuip should narrow this list per customer; do not request `delete:assets:all` or `:all` identifier scopes unless the use case requires it.

### Which layer answers “is this our machine?”

| Question | Enforced by |
| -------- | ----------- |
| May this client touch this **identifier** row for our org? | `*:identifiers:own` + `allowed_organizations` + `identifiers.organization_id` |
| May this client **read/create/update/delete** the global **asset**? | `*:assets:all` (and valid token on routes that only check authentication today) |
| Does this asset appear in **our fleet list**? | Identifier link (`internal_number` + org), not an asset scope |

When a call returns **403**, check identifier scopes and `allowed_organizations` first for org-link operations; check `*:assets:all` for find/create/update/delete on the asset itself. See **`registry-m2m/scopes.md`** for general scope syntax.

## Common agent mistakes

| Mistake | Why it fails |
| ------- | ------------- |
| Setting or expecting `organization_id` on the asset body | Not the tenancy model |
| Listing all assets and filtering client-side | Misses org linkage; may violate least privilege |
| Skipping identifier after `POST /assets` | Asset exists globally but is not linked to the customer org |
| Duplicate identifiers for the same source/identity/org | Conflicts or ambiguous lookups—search first |
| Using only serial number as sole key across brands | Collisions—use asset type + serial or Registry UUID |

## Contrasts (when reasoning about other entities)

| Entity | How org tenancy usually works |
| ------ | ------------------------------ |
| **People** | Organization membership (junction), Pattern A `:own` |
| **Actions** | `organization_id` on the action (with documented edge cases) |
| **Assets** | **Identifiers** (especially `internal_number`), not owner on asset |

## Related

- **`registry-catalog-and-asset-types`** — merke/modell catalog, asset type resolution, identity session funnel
- **`registry-m2m`** — token, scopes, OpenAPI-first workflow
- **`registry-m2m/reference.md`** — environment URLs
- Repository **`templates/customer-onboarding.md`** — org UUIDs and scope list
