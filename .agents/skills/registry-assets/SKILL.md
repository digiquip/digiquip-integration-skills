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

Assets need an **asset type** first. Common paths:

- **`POST /assets/find`** with serial number, brand, and model (see developer docs). With `create=true`, Registry can create asset type and asset when missing.
- Explicit **`POST /asset_types`** then **`POST /assets`** when you manage IDs yourself.

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
| Physical DigiQuip QR | `source=digiquip_qr`, `identity` = code on label |

## Scopes and authorization

- Asset API access is governed by **`read` / `create` / `update` / `delete` scopes on `assets`** (and related dimensions)—see **`registry-m2m/scopes.md`** and your onboarding doc.
- Logto does **not** issue `*:assets:own` with org filtering on the asset table. “Our assets” in product language means **assets reachable through our identifiers** (and endpoints that filter that way), not “assets where `organization_id` = us”.
- **Identifier** writes often require **`organization_id`** when your token uses `:own` identifier scopes—plan creates so the org on the identifier matches an org in `allowed_organizations`.

When a call returns **403**, check both asset scopes and identifier scopes before widening access.

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

- **`registry-m2m`** — token, scopes, OpenAPI-first workflow
- **`registry-m2m/reference.md`** — environment URLs
- Repository **`templates/customer-onboarding.md`** — org UUIDs and scope list
