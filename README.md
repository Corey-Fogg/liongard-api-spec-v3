# Liongard Data API v3 (Spec Repo)

This repo contains the OpenAPI spec and supporting docs for the Liongard vendor consumption API (v3).

## Source of truth
- `openapi/v3.yaml`

## Commands
- `npm install`
- `npm run lint` (Spectral)
- `npm run bundle` (Redocly -> `dist/openapi.json`)
- `npm run preview` (Redocly docs server)

## Workflow
When changing the contract:
1) Update `openapi/v3.yaml`
2) Update examples under `openapi/examples/`
3) Note the contract change in `docs/decisions.md`
4) Run `npm run lint`
