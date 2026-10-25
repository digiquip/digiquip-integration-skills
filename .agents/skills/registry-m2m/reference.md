# Registry integration quick reference

## Base URLs

| Environment | API base | OpenAPI | Swagger UI |
| ----------- | -------- | ------- | ---------- |
| Test | `https://test-registry.kvipp.io` | `/openapi.json` | `/docs` |
| Production | `https://registry.kvipp.io` | `/openapi.json` | `/docs` |

## Auth

| Item | Value |
| ---- | ----- |
| Token URL | `https://auth.digiquip.no/oidc/token` |
| Grant | `client_credentials` |
| Resource | Confirm with DigiQuip (often `https://registry.kvipp.io` for Registry API audience) |

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

- M2M provisioning, scope changes, secret rotation: [support@digiquip.no](mailto:support@digiquip.no)
- Pricing (data flow): https://digiquip.no/en/dataflyt/pricing/
