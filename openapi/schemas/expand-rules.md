# expand rules

- `expand` is an array of strings (query) or per-object array in webhook scopes
- No expand-all
- Discover supported expansions via:
  - `GET /v3/meta/objects`
  - `GET /v3/meta/assets/types`
- If `strictExpand=true`, unsupported expansions return 422
