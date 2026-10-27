# Registry integration quick reference

## Environment names

DigiQuip uses **staging**, **test**, and **non-production** interchangeably for the **same** stack:

| What you may hear | Same environment |
| ----------------- | ---------------- |
| Staging | Yes — one non-production stack |
| Test | Yes |
| `test-registry.kvipp.io` | Yes — Registry hostname for that stack |

**Production** is separate: `registry.kvipp.io`, Logto production tenant, production M2M credentials.

Do not point a **test/staging** M2M client at production Registry (or the reverse).

## Registry API base URLs

| Environment | API base (HTTPS) | OpenAPI | Swagger UI |
| ----------- | ---------------- | ------- | ---------- |
| Staging / test | `https://test-registry.kvipp.io` | `/openapi.json` | `/docs` |
| Production | `https://registry.kvipp.io` | `/openapi.json` | `/docs` |

All API requests use the **HTTPS** base URL above. The OAuth `resource` / audience for staging is a separate string (see below)—it is **not** always the same characters as the browser URL.

## Logto: two hostnames per environment

Each environment uses one Logto **tenant** with two public hostnames:

| Role | Staging / test | Production |
| ---- | -------------- | ---------- |
| **Custom domain** (sign-in, many token calls) | `https://devlogin.digiquip.com` | `https://login.digiquip.com` |
| **Tenant URL** (Logto console, Management API, OIDC discovery) | `https://k9rv6k.logto.app` | `https://4wgi1e.logto.app` |

They refer to the **same** tenant. DigiQuip staff often use the **tenant URL** in the Logto admin UI (`k9rv6k` / `4wgi1e`). Integrations usually call **`{hostname}/oidc/token`** on the hostname DigiQuip listed in onboarding.

### Token endpoint (`POST …/oidc/token`)

| Environment | Typical token URL | OAuth `resource` (audience) for Registry |
| ----------- | ----------------- | ---------------------------------------- |
| Staging / test | `https://devlogin.digiquip.com/oidc/token` | `http://test-registry.kvipp.io` |
| Production | `https://login.digiquip.com/oidc/token` | `https://registry.kvipp.io` |

Use the **exact** `resource=` string from your onboarding email—especially in staging (`http` vs `https` matters for Registry validation).

If a token request against the tenant URL works in Logto but Registry returns **401**, switch to the **custom-domain** token URL for that environment and confirm `resource` matches onboarding.

### OIDC metadata (`/oidc`, `/oidc/jwks`)

Examples (staging tenant):

- Discovery: `https://k9rv6k.logto.app/oidc`
- JWKS: `https://k9rv6k.logto.app/oidc/jwks`

The custom domain exposes the same paths under `https://devlogin.digiquip.com/oidc` and `…/oidc/jwks`.

**Typical M2M integrations do not need these URLs.** Flow: `POST /oidc/token` → send `access_token` to Registry with `Authorization: Bearer`. Registry validates the JWT.

Use `/oidc` or `/oidc/jwks` only if you **decode or verify tokens locally** (debugging claims, custom middleware). They are not a substitute for the token endpoint.

## Organization tenancy (`:own` scopes) — not in the JWT

### What integrators see in the access token

After `POST …/oidc/token`, decoding the JWT typically shows only:

`aud`, `client_id`, `exp`, `iat`, `iss`, `jti`, `scope`, `sub`

There is **no** `allowed_organizations` (or equivalent) in that payload. **If org claims are missing, that is correct.**

### What Registry does on each API call

1. Validates the JWT (issuer, audience, signature).
2. For M2M (`sub` == `client_id`), calls Logto Management API:  
   **`GET /applications/{application_id}/organizations`**
3. Builds internal **`allowed_organizations`** from that response (Logto organization ids).
4. Maps Logto org ids to **Registry organization UUIDs** when authorizing `:own` / Pattern A reads.

Partners never receive that list inside the token. There is **no** token-request parameter to add it.

### What DigiQuip must configure (Logto)

- M2M application created with the right **scopes**.
- Same application **added to the customer organization** in Logto (organization membership for the app).

Registry organization UUID (for API paths and `organization_id` on identifiers) is documented in onboarding. The Logto org row must correspond to that Registry org (via `logto_id` in Registry). DigiQuip ops configures membership; integrators use the UUID in API calls.

### How to verify (do this instead of decoding JWT)

```http
GET https://test-registry.kvipp.io/organizations/{registry_org_uuid}/identifiers?limit=1
Authorization: Bearer <access_token>
```

Use the staging base URL and org UUID from onboarding in test. Success (200, with or without rows) means auth + org resolution worked for that route. **403** → scopes, app not in org, or wrong org id.

Fetching a new access token does not add org claims to the JWT. After Logto membership changes, get a new token **and** retest a Registry call.

### Registry organization UUID

DigiQuip provisions one or more **Registry organization UUIDs** per customer (e.g. to tie machines to a rental company). You need them for:

- **`:own`** and implicit own-read (tenancy resolved server-side as above)
- **Identifiers** (`organization_id` on create/list; fleet via `internal_number`)

Copy UUIDs into `templates/customer-onboarding.md` (local). They are not secret, but they are customer-specific—do not guess them from documentation.

## Developer documentation (human)

| Topic | URL |
| ----- | --- |
| Overview | https://digiquip.no/en/developers/basics/ |
| Structure | https://digiquip.no/en/developers/structure/ |
| Identifiers | https://digiquip.no/en/developers/identifiers/ |
| M2M | https://digiquip.no/en/developers/machine-to-machine/ |
| Organizations | https://digiquip.no/en/developers/organizations/ |
| Actions | https://digiquip.no/en/developers/actions/ |
| Asset types and assets | https://digiquip.no/en/developers/asset_types_and_assets/ |

Norwegian pages use `/nb/developers/` instead of `/en/developers/`.

## Support

- M2M provisioning, scope changes, organization UUIDs, secret rotation: [support@digiquip.no](mailto:support@digiquip.no)
- Pricing (data flow): https://digiquip.no/en/dataflyt/pricing/
