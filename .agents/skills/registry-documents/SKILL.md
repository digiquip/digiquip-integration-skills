---
name: registry-documents
description: |
  Upload and list files on Registry assets and asset types (manuals, certificates,
  etc.). Implicit read on documents.organization_id; nested writes parent-gate on
  assets and asset_types—not create:documents:all for partners.
---

# Registry documents (assets and asset types)

Read this skill when an integration **uploads, lists, or updates files** attached to **assets** or **asset types** (manuals, CE documents, photos, etc.). Pair with **`registry-m2m`**, **`registry-assets`**, and **`registry-catalog-and-asset-types`**.

**Registry behaviour** (parent-gate nested writes + implicit document read): merged in dq-registry PR [#1504](https://github.com/digiquip/dq-registry/pull/1504).

Human docs: [Asset types and assets](https://digiquip.no/en/developers/asset_types_and_assets/). Manual layers: [manuals-and-catalog-documents](https://github.com/DigiQuip/dq-registry/blob/dev/docs/features/asset-type-data-quality/manuals-and-catalog-documents.md).

Confirm paths, form fields, and enums in **`{base_url}/openapi.json`**.

## Two authorization models (do not mix them up)

| Operation | How Registry authorizes |
| --------- | ------------------------ |
| **Read** metadata and lists (`GET /documents/{id}`, `GET /assets/{id}/documents`, `GET /documents/`, …) | **Implicit `read:documents:own`** — Logto does **not** issue `read:documents:own` or require `read:documents:all` for org-scoped partner reads. Any **authenticated** token reads rows where **`documents.organization_id`** matches **`allowed_organizations`** (Pattern A). |
| **Create / update / delete** on **`POST …/assets/{id}/documents`**, **`POST …/asset_types/{id}/documents`**, and **`PATCH` / `DELETE /documents/{id}`** when the token has **no** `create|update|delete:documents:all` | **Parent-gate** on **`assets`** or **`asset_types`** (row must have `asset_id` or `asset_type_id`). Org-scoped rows need form/body **`organization_id`** plus parent **`:own`** for that org (or parent **`:all`** / `all:everything`). |
| **Create / update / delete** on **`POST /documents/`** (no parent) and **steward catalog** manual routes | **`create|update|delete:documents:all`** (or `all:everything`) — not issued to standard partners. |

**Read does not flow through `read:asset_types:own` or `read:assets:own`.** There is no `read:assets:own` in Logto. Document read is always the **documents** dimension (implicit own), filtered by **`organization_id`** on the document row.

## Partner endpoints (typical)

| Goal | Method | Notes |
| ---- | ------ | ----- |
| Upload file on a **machine** | `POST /assets/{asset_id}/documents` | Multipart: `file`, `document_type`, optional `id`; set **`organization_id`** to customer Registry UUID |
| List files on a **machine** | `GET /assets/{asset_id}/documents?viewer_organization_id={org}` | Returns **public** docs (`organization_id` null) plus **your** org’s rows |
| Upload **manual** on **asset type** (org overlay) | `POST /asset_types/{asset_type_id}/documents` | Manual types: **`organization_id` required** (`user_manual`, `service_manual`, `quick_guide`) |
| List **manuals** (catalog + org overlay) | `GET /asset_types/{asset_type_id}/manuals?organization_id={org}` | Does not require `read:documents` scope; returns document ids and metadata |
| Document metadata | `GET /documents/{document_id}` | Implicit document read |
| Download bytes | `GET /documents/{document_id}/file` | **No bearer token** (UUID capability URL) |
| Patch metadata | `PATCH /documents/{document_id}` | Parent **`update:asset_types:own`** or **`update:assets:all`** if no `update:documents:all` |
| Delete | `DELETE /documents/{document_id}` | Parent **`delete:asset_types:own`** or **`delete:assets:all`** if no `delete:documents:all` |

Do **not** use **`POST /documents/`** for partner integrations unless DigiQuip explicitly provisioned **`create:documents:all`**.

## Recommended Logto scopes (partner)

Minimum for upload + org-scoped read (no documents dimension on the token):

```text
create:assets:all
create:asset_types:own
```

Optional if the integration **updates or deletes** its uploads without `*:documents:*`:

```text
update:asset_types:own
delete:asset_types:own
```

Asset-linked documents use **`create:assets:all`** (and **`delete:assets:all`** for delete/make-public paths when parent-gating applies) — there is no `create:assets:own`.

**Do not provision for standard partners:**

- `read:documents:all` — not needed for org-scoped read (implicit own)
- `create:documents:all` — **does not** authorize nested upload without parent scopes; only bare `/documents/` and steward flows

DigiQuip still withholds **`update:assets:all`** / **`delete:assets:all`** for changing **serial number / production year** on the asset row — see **`registry-assets`**. Document upload is separate.

## Upload rules

- **Always set `organization_id`** on partner uploads (customer Registry UUID) so files stay **org-scoped** and appear in lists with `viewer_organization_id`.
- **Public asset documents** (`organization_id` omitted): only with **`create:assets:all`** (or elevated); visible to all viewers on that asset — avoid for tenant-specific integrations.
- **Manual on asset type:** `organization_id` is **required** for manual document types; duplicate content vs catalog may return **409** with existing catalog `document_id` — do not store a second blob.
- **Multipart:** send `file` + `document_type`; optional client-generated `id` (UUID4). See OpenAPI for deprecated document types (`finance`, `leasing`, … — legacy sync only).

## Prerequisites

- **Asset** exists and is the one you intend (Registry does not parent-gate “this asset belongs to your fleet” on upload — use **identifiers** and your own idempotency; see **`registry-assets`**).
- **Asset type** is linked to the customer org when uploading type-level manuals (`organization_asset_types`).
- M2M app is a **member of the organization in Logto** so implicit document read and `:own` parent writes resolve **`allowed_organizations`**.

## Anti-patterns

| Mistake | Why it fails |
| ------- | ------------ |
| Grant only `create:documents:all` for nested upload | **403** on `POST …/asset_types/…/documents` without parent scope |
| Expect `read:asset_types:own` to gate document GET | Read uses **implicit `read:documents:own`** on the document row |
| Omit `organization_id` on partner manual upload | **400** for manual types on asset types |
| `POST /documents/` from ERP | Wrong surface; needs steward `create:documents:all` |
| Confuse asset field edits with documents | Serial/year changes are **not** document uploads; no `update:assets` for partners |

## Related

- **`registry-assets`** — fleet via identifiers; `create:assets:all`
- **`registry-catalog-and-asset-types`** — resolve type; `create:asset_types:own` + org link
- **`registry-m2m/scopes.md`** — implicit read, global catalog read, `:own` / `:all`
