---
name: registry-digiquip-qr-nfc
description: |
  Connect and lifecycle-manage DigiQuip printed QR labels and NFC tags as Registry
  identifiers (source digiquip_qr), including release for adoption and equipment
  landing page licensing. Use when integrating physical codes, scans, or branding.
---

# DigiQuip QR and NFC (Registry identifiers)

Physical **DigiQuip QR labels** and **NFC tags** are not separate Registry entities. They are rows in **identifiers** with a fixed `source`. Pair with **`registry-m2m`** (auth) and **`registry-assets`** (linking codes to assets and org fleet semantics).

Human docs: [External identifiers — link a DigiQuip QR code](https://digiquip.no/en/developers/identifiers/#link-a-digi-quip-qr-code-to-an-asset-or-asset-type). Scan and landing behaviour: [dq-equipment-page — How DigiQuip QR and NFC work today](https://github.com/digiquip/dq-equipment-page/blob/main/docs/digiquip-qr-today.md) (reference implementation notes).

## What is on the label or tag

- DigiQuip ships **unique printed codes**. **No two labels share the same code.**
- The public URL is on **`qr.digiquip.com`** (staging: `qr.staging.digiquip.no`). Scanners open that URL; the last path segment is the code identity.

| Physical medium | URL pattern | Typical `identity` length |
| --------------- | ----------- | ------------------------- |
| Printed QR (current) | `https://qr.digiquip.com/d/{code}` | 7 alphanumeric characters (e.g. `PYZEV1Q`) |
| NFC tag | `https://qr.digiquip.com/nfc/{code}` | 8 alphanumeric characters |
| Legacy QR | `https://qr.digiquip.com/q/{uuid}` | UUID (still accepted, no longer issued) |

**NFC is not a separate Registry source.** Tags are programmed with `/nfc/…` in the URL, but Registry stores the same `source` as QR: **`digiquip_qr`**. There is no `digiquip_nfc` source. After connect, Registry does not record whether the code was applied as QR or NFC (products may infer 7 vs 8 characters only for UI).

Lookups are **case-insensitive**; many writers **uppercase** the code before save.

## Registry identifier shape

When a code is **connected** to equipment (or an asset type), create or update an identifier like:

| Field | Value |
| ----- | ----- |
| `source` | `digiquip_qr` |
| `identity` | The code from the label/tag (normalized, often uppercase) |
| `type` | `asset` (most integrations) or `asset_type` when linking at type level |
| `foreign_id` | Registry UUID of the asset or asset type |
| `organization_id` | Customer org UUID when this org “owns” the connect (see below) |

**Always check whether the identifier already exists** before create (`GET /identifiers/{identity}?source=digiquip_qr`, or list/filter). A code is globally unique: one physical code maps to one linked entity.

## Who registers codes on an organization?

| Approach | When |
| -------- | ---- |
| **Partner M2M (typical)** | You **POST** (or PATCH) the identifier when linking a physical label/NFC to an asset or asset type, with **`organization_id`** set to your Registry org UUID — same pattern as Kontoret / Stemplet connect. |
| **DigiQuip bulk** | DigiQuip can register many codes or pre-connect on request (support / ops). Not required for every integration. |
| **Scan-only** | Until a row exists, `qr.digiquip.com` resolve shows **not connected**; the code is still a valid printed identity once you create the identifier. |

There is no separate “register org on code” API beyond writing the identifier with `source=digiquip_qr`, `identity`, `foreign_id`, `type`, and **`organization_id`**.

## Does Registry validate “real issued” codes?

**No central allow-list** on `POST /identifiers/`. Registry checks **uniqueness** (`identity` + `source` globally for `digiquip_qr`) and your **scopes / org write rules**. It does **not** verify that the string was minted on a DigiQuip print run.

**Practical validation:**

- Read the code from the **URL** (`/d/{7}` or `/nfc/{8}`) or scan; normalize (often uppercase).
- **`GET /identifiers/{identity}?source=digiquip_qr`** before create — avoid typos that would mint a **new** row that **does not scan** on `qr.digiquip.com`.
- Wrong identity → orphan identifier; fix by PATCH/DELETE per release rules, not by reusing the physical sticker.

Format conventions: 7 alphanumeric (QR `/d/`), 8 (NFC `/nfc/`), legacy UUID (`/q/`). Case-insensitive lookup.

## `GET /identifiers/{identity}?source=digiquip_qr` with `:own` scopes

Query params: **`source=digiquip_qr`** required; `organization_id` / `type` are **not** required for lookup (unlike `internal_number`).

| Situation | HTTP | Notes |
| --------- | ---- | ----- |
| **No row** (never connected, or typo) | **404** | Safe “code is free” signal for that identity string. |
| **Row exists**, `organization_id` = **your** allowed org | **200** | Full identifier + relations. |
| **Row exists**, `organization_id` = **another** org | **403** | Tenancy: identifiers use **DIRECT** `organization_id` filter under `:own`. |
| **Row exists**, `organization_id` is **null** (released for adoption) | **403** with typical `:own` token | Row is visible to unrestricted readers only; you may **adopt** via PATCH with your org if you have **update:identifiers:own** and policy allows. |

There is **no** 200 “empty shell” for an unregistered code — unregistered means **404**. Do not infer availability from JWT org claims (orgs are not in the token — see **`registry-m2m/SKILL.md`**).

**Create:** with **`create:identifiers:own`**, **`organization_id` is required** on POST (cannot mint org-less `digiquip_qr` rows with `:own` only).

Example (asset), from developer docs:

```json
{
  "source": "digiquip_qr",
  "identity": "PYZEV1Q",
  "foreign_id": "22934840-6214-4b84-bfde-7b239c6d2e33",
  "type": "asset",
  "organization_id": "<organization UUID>"
}
```

Confirm required fields and query parameters in OpenAPI for your environment. Connect flows in Stemplet/Kontoret also set `organization_id` and may add `external_content` (integration metadata)—follow the contract your product uses.

`internal_number` is a **different** `source` (org fleet numbers). Do not confuse it with `digiquip_qr`.

## Setup workflow (integrator)

1. Resolve or create the **asset** (see **`registry-assets`**).
2. Read the **code** from the label or from the scan URL (`/d/` or `/nfc/` segment).
3. **Lookup** `digiquip_qr` + identity; if missing, **POST** identifier linking `foreign_id` and `organization_id`.
4. If the code should point at another asset, **update** `foreign_id` (and org if needed)—do not mint a second identity for the same physical code.
5. Test scan: `qr.digiquip.com` resolves via operator, then landing or redirect rules apply (below).

## Releasing a code for another organization (critical)

The **sticker or NFC tag stays on the machine** in almost all real-world handovers. The code printed on it is permanent.

| Do | Do not |
| -- | ------ |
| **Clear org association** on the identifier when your customer no longer operates that machine but the label remains | **DELETE** the identifier while the physical code still exists on the equipment |
| Use **`PATCH /identifiers/{identity}`** (with `source=digiquip_qr` query params per OpenAPI) to set **`organization_id` to `null`** | Remove the row and leave a orphaned scan that hits “not connected” |

After `organization_id` is null, the **`digiquip_qr` row remains**: same `identity`, same `foreign_id` (asset link) unless you also change equipment assignment. Another organization can **adopt** the code by connecting/updating with their `organization_id` (product-specific connect rules apply).

Only delete an identifier if the **physical label or tag is destroyed or removed** and the code will never be scanned again—or when DigiQuip support explicitly directs retirement.

Agents implementing offboarding: prefer **patch org to null**, not delete.

## Equipment landing page (`dq-equipment-page`)

Scans hit **dq-equipment-page** on `qr.digiquip.com`, not your integration server. The app resolves the code (operator → Registry `digiquip_qr`), then decides:

- Redirect to a legacy DigiQuip instance machine URL (when `digiquip` sync identifiers exist),
- Redirect to Kvipp Mobile,
- Or render **`/landing/{identifier}`** on the equipment site.

**Default experience without customer branding:** connected codes still get a **generic DigiQuip landing** (controls, Kvipp CTA, etc.). That is the normal unpaid / unsubscribed presentation.

**Customer-branded landing** (partner logo, contact, footer, configured sections) requires:

1. An active Registry subscription for service **`equipment_landing`** on the organization (plan limits apply—see DigiQuip commercial docs / data flow pricing).
2. **Equipment landing** settings in dq-admin, stored as Registry **`organization_options`** under `landing.*` (partner chrome, show/hide controls, destination preferences).

Without **`equipment_landing`**, scans do not show that org’s customized landing; users see the **standard DigiQuip** landing flow (or redirects per resolve), not a white-label company page.

Integrations that only write Registry identifiers **do not** configure landing UI—that is org admin / dq-admin work after licensing.

Optional add-ons (e.g. expert chat on landing) are separate **`equipment_landing`** subscription add-ons in Registry—see subscription catalog in OpenAPI or DigiQuip product docs.

## Recommended scopes

Same identifier and asset scopes as **`registry-assets`** for connect and fleet work:

- `read|create|update|delete:identifiers:own` (plus `allowed_organizations`)
- `read|create|update|delete:assets:all` when creating or re-linking assets

**Release for adoption** needs at least **`update:identifiers:own`** (or `:all` if provisioned) to PATCH `organization_id` to null.

Reading subscription or `landing.*` options for scans is usually **not** part of a partner M2M integration; equipment-page loads those when rendering.

## Related skills and docs

- **`registry-assets`** — assets without org owner; `internal_number` vs physical codes
- **`registry-m2m`** — token and OpenAPI-first workflow
- [Kvipp deep links](https://digiquip.no/en/developers/kvipp-deep-links/) — different entry routes (`kvipp.it`, external identifiers); not the same as `qr.digiquip.com` equipment codes
