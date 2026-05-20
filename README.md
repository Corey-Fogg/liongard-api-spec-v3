---
nav_exclude: true
---

# Liongard Data API v3

OpenAPI 3.1 specification for the Liongard Data API v3 — a structured datalake API that lets vendors **push data in**, **pull insights out**, and **react to changes** in real time.

> The user-facing documentation lives at the published [GitHub Pages site](https://corey-fogg.github.io/liongard-api-spec-v3/). This README is the GitHub-only repo guide.

---

## Repo Layout

```
liongard-api-spec-v3/
├── liongard-api-v3.yaml          ← OpenAPI 3.1 specification (source of truth)
├── inspector-manifest.schema.json← JSON Schema for the inspector creation framework
├── COMPREHENSIVE_API_GUIDE.md    ← Vendor integration walkthrough
├── DESIGN_HISTORY_AND_RATIONALE.md
├── index.md                      ← Landing page for the docs site (Jekyll/just-the-docs)
├── swagger-ui.html               ← Interactive API explorer
├── _config.yml                   ← Jekyll config
├── .clinerules                   ← Editing rules for Claude Code agents
├── docs/                         ← Long-form references
│   ├── INSPECTOR_MANIFEST_GUIDE.md  ← Manifest framework: single doc, one API call, full inspector
│   ├── RSQL_FILTER_GUIDE.md
│   ├── RESPONSE_FORMAT_GUIDE.md
│   ├── JOBS_ASYNC_PROCESSING.md
│   ├── METRICS_FEATURE.md
│   ├── WEBHOOKS_GUIDE.md
│   └── technical-references.md
└── examples/                     ← Reference payloads
    └── senteon-manifest.json     ← Complete Senteon inspector as one manifest
```

`README.md`, `.clinerules`, `examples/`, and `liongard-api-v3.yaml` are excluded from the published docs site by `_config.yml`.

---

## Editing the Spec

Conventions are formalised in `.clinerules`. Highlights:

1. OpenAPI 3.1.0 compliance
2. **No response wrappers** — list endpoints return arrays, single endpoints return objects, pagination lives in headers
3. **RSQL** for the `?filter=` parameter on every list endpoint (never `field__op=value`)
4. camelCase fields, ISO-8601 timestamps
5. Async (202 + job id) for any operation that can exceed 5 s
6. Use the terms **inspector**, **environment**, **dataprint**, **system**, **launchpoint** consistently

### Inspector Builder

Sub-resource endpoints under `/v3/inspectors/{inspectorId}/...` define the full inspector model: `ui-config-template`, `authentication`, `endpoints`, `discovery`, `views`, `asset-mappings`, `rules`. They can be authored piecewise or driven in one shot by posting an `inspectorManifest` to `POST /v3/inspectors/import`. See `docs/INSPECTOR_MANIFEST_GUIDE.md`.

The manifest standardises the **inputs needed to create an inspector** (auth, endpoints, mappings, views, rules). The dataprint payload an inspector emits is intentionally not standardised — each inspector chooses its own shape and the per-environment `dataprintConfig` (`PUT /v3/environments/{envId}/inspectors/{id}/config`) is how Liongard learns to parse it into environments and assets.

---

## Validate & Preview Locally

```bash
# YAML syntax
python3 -c "import yaml; yaml.safe_load(open('liongard-api-v3.yaml'))"

# OpenAPI 3.1 validation
npx @apidevtools/swagger-cli validate liongard-api-v3.yaml

# Validate the manifest schema against the bundled example
python3 -c "import json, jsonschema; jsonschema.validate(json.load(open('examples/senteon-manifest.json')), json.load(open('inspector-manifest.schema.json')))"

# Browse the spec in Swagger UI
docker run -p 8080:8080 \
  -e SWAGGER_JSON=/spec/liongard-api-v3.yaml \
  -v $(pwd):/spec swaggerapi/swagger-ui

# Preview the docs site (Jekyll)
bundle exec jekyll serve
```

---

## Contributing

1. Edit `liongard-api-v3.yaml` and any affected `.md` files in the same PR.
2. Run the validations above.
3. Keep the doc cross-references in sync — `COMPREHENSIVE_API_GUIDE.md`, the relevant `docs/*` file, and the spec should agree on field names and example payloads.
4. New endpoints belong to an existing tag when possible; add a new tag only when the surface is genuinely new (e.g. `Inspector Builder`).

For broader design context, read `DESIGN_HISTORY_AND_RATIONALE.md`.

---

## Links

- **Interactive docs:** [Swagger UI](swagger-ui)
- **OpenAPI 3.1:** https://spec.openapis.org/oas/v3.1.0
- **RSQL:** https://github.com/jirutka/rsql-parser
- **RFC 5988 (Link header):** https://tools.ietf.org/html/rfc5988
