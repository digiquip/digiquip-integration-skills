---
name: registry-catalog-and-asset-types
description: |
  Shared brand and model catalog (read-only on Registry M2M), search-first
  resolution of asset_type_id, and when catalog rows may be created (Operator
  identity session only—not direct Registry catalog API).
---

# Catalog, asset types, and Merke og Modell

Read this skill when an integration must **resolve brand and model** before creating or updating **assets**. Pair it with **`registry-m2m`** (auth, scopes) and **`registry-assets`** (assets and identifiers).

Human docs: [Asset types and assets](https://digiquip.no/en/developers/asset_types_and_assets/), [Merke og modell (NO)](https://digiquip.no/nb/kvipp/merker-og-modeller/).

Product rules in Registry: [brand-and-model-principles](https://github.com/DigiQuip/dq-registry/blob/dev/docs/features/asset-type-data-quality/brand-and-model-principles.md).

## Three layers (do not conflate them)

| Layer | Registry surface | Partner M2M |
| ----- | ---------------- | ----------- |
| **Catalog brand** | `GET /brands/`, search, similar | **Read only** |
| **Catalog model** | `GET /brands/{brand_id}/models/`, search, similar | **Read only** |
| **Asset type** | `asset_types` (brand + model + optional version) | **Read**; **create** only when your onboarding explicitly allows it and catalog already matches |

An **asset** always needs an **`asset_type_id`** (or a flow that returns one, such as `POST /assets/find`). You cannot treat free-text brand/model on the asset as a substitute for resolving the type.

## Catalog writes: identity session, not Registry M2M

DigiQuip maintains the **shared** manufacturer and model catalog. **Partners do not create or update catalog brands or models through the Registry REST API** (`POST` / `PUT` / `PATCH` / `DELETE` on `/brands/` or `/brands/{id}/models/`). Those routes are not part of standard M2M onboarding; direct catalog API use would bypass the controls DigiQuip relies on.

When a **new** manufacturer or model must enter the catalog, that happens through a **controlled funnel**:

- **Operator equipment identity session** — interactive brand-then-model flow with narrowing, optional web/LLM on commit, confidence thresholds, review flags, and steward workflow. Documented in dq-operator (`docs/EQUIPMENT_IDENTITY_SESSION.md`). This is **not** a substitute for Registry M2M; it is for product flows (Kvipp, Operator-backed UIs) that forward a user Registry token and Operator auth.

**Integrations must not** replicate catalog creation by calling Registry brand/model write endpoints. If ERP data has no catalog match, **stop**, surface the gap to the user or DigiQuip (support / admin), or route the user through an identity-session-capable product—not silent `POST /brands/`.

## Search-first flow (API equivalent of narrowing)

Mirror the **order** used in identity session: **brand → model → asset type**. Do not invent identity from category names, machine shape, or guesses.

```text
1. BRAND (catalog read)
   GET /brands/?search_term=…
   GET /brands/similar?name=…
   GET /brands/search?name=…
   → lock brand_id or canonical name from an existing row

2. MODEL (catalog read, only after brand is known)
   GET /brands/{brand_id}/models/?search_term=…
   GET /brands/{brand_id}/models/similar?name=…
   GET …/models/search?name=…
   → lock model_id or canonical model name

3. ASSET TYPE
   GET /asset_types/?brand=…&model=…
   GET /asset_types/by_brand_and_model?brand=…&model=…
   GET /asset_types/resolve?brand=…&model=…
   GET /asset_types/create_preview?brand=…&model=…
   → use existing asset_type_id, or POST /asset_types/ only if allowed and preview/resolve says it is correct

4. ASSET
   POST /assets/ with asset_type_id
   (see registry-assets; avoid blind POST /assets/find?create=true if catalog discipline matters)
```

### Practical rules

1. **Search before create.** Prefer `only_verified=true` on brand/model lists when you want steward-approved rows only.
2. **Minimum search length** — UX often waits until three characters before search (identity session); pasted or scanned text may search immediately.
3. **Brand before model** — never search models without a locked `brand_id` / brand name.
4. **Use resolve and create_preview** — `resolve` returns exact match vs candidates; `create_preview` shows canonical strings and whether a row already exists **without** writing.
5. **Variants stay distinct** — do not fold `E` / `H` drive suffixes onto a base model; when several models match, ask the user—do not pick silently.
6. **Hidden types** — default list/search omits `not_searchable` asset types; staff may pass `include_not_searchable=true` when documented.

## Asset type create vs catalog

`POST /asset_types/` is the **operational** row (brand + model) assets point to. It is separate from **catalog** maintenance:

- Partners with `create:asset_types:…` may create types **after** catalog search shows the right merke and modell (or an existing type id).
- Registry may **link** asset types to catalog and apply canonical names on create; behavior is defined in OpenAPI and `link_asset_type_to_catalog` in dq-registry. **Do not** use asset type create to bypass the rule “no catalog brand/model writes via API”—if brand or model is missing from the catalog, treat it as a **catalog gap**, not something to paper over with arbitrary strings.

`POST /assets/find?create=true` can auto-create types and assets. It is convenient for legacy sync but **weakens** the same catalog discipline as a guided search. Prefer explicit resolve → `asset_type_id` → `POST /assets/` when DigiQuip asked for controlled data quality.

## Recommended scopes (M2M)

| Scope | Typical need |
| ----- | ------------ |
| `read:brands:all` | `GET /brands/` — search, similar, `only_verified`, pagination, brand `id` |
| `read:models:all` | `GET /brands/{brand_id}/models/` — model search under a locked brand |
| `read:asset_types:own` or `read:asset_types:all` | `GET /asset_types/`, by id, org links — **not** a substitute for catalog read |
| `create:asset_types:…` | Only if onboarding allows creating types after catalog match |

**Do not request** `create:brands:all`, `update:brands:all`, `create:models:all`, or similar for partner integrations—they are not issued for catalog maintenance.

**`:own` on `brands` or `models`:** Registry treats these as **global** dimensions (no org filter on catalog rows). Even if a token contained `read:brands:own`, it would not limit results to one customer. Partner tokens should use **`read:brands:all`** and **`read:models:all`** only. **`read:people:own`** or **`read:asset_types:own`** does **not** grant catalog access—you need explicit `read:brands` / `read:models`.

Asset types may use `:own` (org junction) or `:all` depending on customer; confirm in **`templates/customer-onboarding.md`**.

### Resolve helpers vs catalog API

Some routes under `/asset_types/` only require a **valid bearer token** (authentication), not `read:brands`—for example **`GET /asset_types/resolve`**, **`GET /asset_types/create_preview`**, and the legacy string lists **`GET /asset_types/brands`** / **`…/brands/{brand}/models`**.

Use them for quick match/preview when you already have brand + model text. For **guided catalog search** (verified flag, catalog UUIDs, alternative names, similar-name thresholds), call **`/brands/`** and **`/brands/{brand_id}/models/`** with **`read:brands:all`** / **`read:models:all`**.

## Common agent mistakes

| Mistake | Why it fails |
| ------- | ------------ |
| `POST /brands/` or `POST …/models/` when ERP has a new name | Violates catalog policy; no M2M scope |
| Skipping catalog search and going straight to `POST /asset_types/` | Duplicates, wrong variants, review noise |
| Using global asset type text search instead of brand-locked model search | Wrong manufacturer context |
| `POST /assets/find?create=true` on every sync line | Creates types without human/catalog funnel |
| Treating org category or KREG class as model | Conflicts with brand-and-model principles |

## Related

- **`registry-assets`** — assets, identifiers, fleet listing
- **`registry-m2m`** — token, OpenAPI-first workflow
- **`registry-m2m/scopes.md`** — `:own` / `:all`, brands/models read-only for partners
- Operator **equipment identity session** — catalog writes and review funnel (product API, not Registry M2M)
