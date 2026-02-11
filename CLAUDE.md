# ClaudeCode Instructions (Liongard API v3)

You are editing an OpenAPI 3.1 spec for the Liongard Data API v3.

## Canonical file
- Edit: `openapi/v3.yaml`
- Keep examples updated: `openapi/examples/`

## Non-negotiables (v3 design)
- camelCase JSON fields
- Pagination uses query params `limit` and `offset`
- List responses include `pagination: { limit, offset, count }`
- Filtering uses `where` query param; webhook subscriptions also use `where`
- `expand` is dynamic strings; no expand-all; discover via meta endpoints
- Inspectors are environment-scoped:
  - `/v3/environments/{environmentId}/inspectors/...`
- Dataprints are inspector-scoped (no global dataprints):
  - `/v3/environments/{environmentId}/inspectors/{inspectorId}/dataprints`
  - Config is `/dataprints/config`
- POST endpoints return 201 unless explicitly documented otherwise

## Change workflow
1) Update `openapi/v3.yaml`
2) Add/update JSON examples under `openapi/examples/`
3) Update `docs/decisions.md` if the contract changes
4) Run `npm run lint` and fix issues
