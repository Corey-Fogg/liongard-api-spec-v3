---
title: Inspector Manifest Framework
parent: Technical References
nav_order: 1
---

# Inspector Manifest Framework

A single JSON document that describes a complete custom Liongard inspector — every credential field, HTTP endpoint, UI view, asset mapping, alert rule, and metric — and the systematic sequence of API calls that provisions it.

The manifest is the v3 API's answer to the question: **"How do I create a complete custom inspector via API, without writing inspector code?"**

- The full structure is published as a standalone JSON Schema at `inspector-manifest.schema.json`.
- The same shape is published as the `inspectorManifest` schema in `liongard-api-v3.yaml`.
- The pipeline that consumes it is exposed at `POST /v3/inspectors/import`.

---

## Why a manifest

An inspector has at least ten logical pieces, and every one of them is independently editable in the v3 API:

| Inspector piece | v3 sub-resource endpoint |
|---|---|
| Identity (name, alias, category…) | `POST /v3/inspectors` |
| Credential form for users | `PUT /v3/inspectors/{id}/ui-config-template` |
| Authentication flow (apiKey, OAuth, …) | `PUT /v3/inspectors/{id}/authentication` |
| HTTP endpoints (preflight, extract, child) | `POST /v3/inspectors/{id}/endpoints` |
| Multi-tenancy / discovery | `PUT /v3/inspectors/{id}/discovery` |
| Dataprint → environments + assets mapping | `PUT /v3/environments/{envId}/inspectors/{id}/config` |
| UI views (tabs) | `POST /v3/inspectors/{id}/views` |
| Asset inventory mappings | `PUT /v3/inspectors/{id}/asset-mappings` |
| Metrics (JMESPath queries) | `POST /v3/metrics` |
| Alert rules | `POST /v3/inspectors/{id}/rules` |

Authoring one inspector by hand would mean roughly ten request bodies, all tied together by IDs that the previous responses return. The manifest collapses that into a single JSON document and a single API call. The server runs the pipeline in dependency order, threads IDs between steps, and returns the full `callPipeline` so the result is auditable — every operation that ran, with the exact payload and resource id it produced.

---

## Manifest shape

```jsonc
{
  "manifestVersion": "1.0",

  "definition":         { /* InspectorCreateRequest */ },
  "authentication":     { /* InspectorAuthenticationUpsertRequest */ },
  "uiConfigTemplate":   { "fields": [ /* UiConfigField, … */ ] },
  "endpoints":          [ /* EndpointDefinitionCreateRequest, … */ ],
  "discovery":          { /* DiscoveryConfigUpsertRequest */ },
  "dataprintMetadata":  { /* DataprintMetadata */ },
  "views":              [ /* ViewCreateRequest, … */ ],
  "assetMappings":      { /* AssetMappingsUpsertRequest */ },
  "metrics":            [ /* MetricCreateRequest, … */ ],
  "rules":              [ /* RuleCreateRequest, … */ ]
}
```

Every section is **optional except `definition`**. Sections that are present run; sections that are absent are skipped. The same shape comes back out of `GET /v3/inspectors/{id}/manifest` — manifests round-trip.

---

## The pipeline

`POST /v3/inspectors/import` runs the sections in dependency order. Each step is one call against the matching v3 sub-resource endpoint — exactly the call the caller could have made by hand.

| Step | Section | API call |
|------|---------|----------|
| 1 | `definition` | `POST /v3/inspectors` *(or `PATCH /v3/inspectors/{id}` when `upsert=true` matches an existing name)* |
| 2 | `uiConfigTemplate` | `PUT /v3/inspectors/{id}/ui-config-template` |
| 3 | `authentication` | `PUT /v3/inspectors/{id}/authentication` |
| 4 | each `endpoints[]` | `POST /v3/inspectors/{id}/endpoints` |
| 5 | `discovery` | `PUT /v3/inspectors/{id}/discovery` |
| 6 | `dataprintMetadata` *(when `environmentId` is provided)* | `PUT /v3/environments/{environmentId}/inspectors/{id}/config` |
| 7 | each `views[]` | `POST /v3/inspectors/{id}/views` |
| 8 | `assetMappings` | `PUT /v3/inspectors/{id}/asset-mappings` |
| 9 | each `metrics[]` | `POST /v3/metrics` *(the importer fills `inspectorId` and a default `inspectorVersionId`)* |
| 10 | each `rules[]` | `POST /v3/inspectors/{id}/rules` *(metric references by name are resolved to the IDs the importer captured in step 9)* |

The response carries the full `callPipeline`:

```json
{
  "inspectorId": "inspector_abc123",
  "dryRun": false,
  "callPipeline": [
    {
      "step": 1,
      "operationId": "createInspector",
      "method": "POST",
      "path": "/v3/inspectors",
      "payload": { "name": "acme-monitor", ... },
      "status": 201,
      "resourceId": "inspector_abc123",
      "durationMs": 48
    },
    {
      "step": 2,
      "operationId": "upsertInspectorUiConfigTemplate",
      "method": "PUT",
      "path": "/v3/inspectors/inspector_abc123/ui-config-template",
      "payload": { "fields": [ ... ] },
      "status": 200,
      "durationMs": 22
    },
    ...
  ],
  "warnings": [],
  "errors": []
}
```

`dryRun: true` runs the same pipeline through the templating engine, returns the rendered call list and any validation warnings, and mutates nothing. Use it to preview a manifest before pushing it to production.

---

## Templating placeholders

The manifest references runtime values in three flavors of placeholder. The importer validates that every placeholder has a target before it sends the call.

| Placeholder | Resolved from | Where it appears |
|---|---|---|
| `{FIELD_NAME}` | `uiConfigTemplate.fields[].name` | `authentication.headerFormat`, `authentication.tokenParameters`, `endpoints[].urlTemplate`, `endpoints[].queryParams`, `endpoints[].bodyTemplate`, `endpoints[].headers` |
| `{preflight.EndpointName}` | A preflight endpoint's resolved data | `endpoints[].urlTemplate`, `endpoints[].bodyTemplate` of `role=extract` endpoints |
| `{parent.fieldName}` | One row of the parent endpoint's response (per-row substitution) | `endpoints[].urlTemplate`, `endpoints[].queryParams`, `endpoints[].bodyTemplate` of `role=child` endpoints |

Examples:

```jsonc
// Authentication header for raw <id>:<secret> apiKey
{ "headerFormat": "{SENTEON_API_KEY_ID}:{SENTEON_API_KEY_SECRET}" }

// POST extract that fans out preflight ids
{
  "name": "Users",
  "role": "extract",
  "method": "POST",
  "urlTemplate": "/user-management/entities/users/GET/v1",
  "bodyTemplate": { "ids": "{preflight.UserIDs}" },
  "dependsOn": ["UserIDs"]
}

// Per-endpoint compliance fan-out
{
  "name": "Compliance",
  "role": "child",
  "parentEndpoint": "AccountEndpoint",
  "attachAs": "Compliance",
  "urlTemplate": "/Agent/Compliance?endpoint={parent.Id}"
}
```

---

## RSQL across the builder

Every list endpoint added under `Inspector Builder` accepts the same `?filter=` parameter that the rest of v3 does, using **RSQL** syntax (see [RSQL Filter Guide](RSQL_FILTER_GUIDE.md)):

```bash
# Find all preflight endpoints
GET /v3/inspectors/inspector_13/endpoints?filter=role==preflight

# Find extract endpoints touched in the last day
GET /v3/inspectors/inspector_13/endpoints?filter=role==extract;updatedAt>=2026-05-18T00:00:00Z

# Find inspectors authored by a specific team
GET /v3/inspectors?filter=author==*Liongard*

# Disabled rules for an inspector
GET /v3/inspectors/inspector_13/rules?filter=enabled==false

# Views shown on the child side of a parent/child pair
GET /v3/inspectors/inspector_13/views?filter=viewType==child
```

The `metricEvaluateRequest.filters` and `bulkMetricEvaluateRequest.filters` bodies keep their `{field, op, value}` shape because they support typed values; the URL-level filter parameter on every list endpoint — including the new builder endpoints — uses RSQL.

---

## End-to-end example: Pax8

A complete manifest for a Pax8 inspector — credentials, OAuth2 client credentials auth, four endpoints, per-customer discovery, six views, asset mappings, and three metrics — would look like this in skeleton form:

```jsonc
{
  "manifestVersion": "1.0",
  "definition": {
    "name": "pax8-inspector",
    "alias": "Pax8",
    "description": "Pax8 cloud marketplace data — companies, subscriptions, orders, invoices.",
    "category": "cloud",
    "defaultFrequency": { "type": "days", "interval": 1 }
  },
  "uiConfigTemplate": {
    "fields": [
      { "name": "PAX8_CLIENT_ID",     "label": "Pax8 Client ID",     "type": "string", "required": true, "secure": true, "parent": true, "help": "Pax8 partner portal → Settings → API." },
      { "name": "PAX8_CLIENT_SECRET", "label": "Pax8 Client Secret", "type": "secret", "required": true, "secure": true, "parent": true },
      { "name": "PAX8_COMPANY_ID",    "label": "Pax8 Company ID",    "type": "string", "required": false, "help": "Auto-populated by parent discovery." }
    ]
  },
  "authentication": {
    "type": "oauth2ClientCredentials",
    "baseUrl": "https://api.pax8.com",
    "tokenEndpoint": "/v1/oauth/token",
    "tokenContentType": "application/x-www-form-urlencoded",
    "tokenParameters": {
      "grant_type":    "client_credentials",
      "client_id":     "{PAX8_CLIENT_ID}",
      "client_secret": "{PAX8_CLIENT_SECRET}"
    },
    "credentialFields": ["PAX8_CLIENT_ID", "PAX8_CLIENT_SECRET"],
    "headerFormat": "Bearer {ACCESS_TOKEN}"
  },
  "endpoints": [
    { "name": "Companies",     "role": "discovery", "method": "GET", "urlTemplate": "/v1/companies",                                  "pagination": { "type": "pageSize", "pageParam": "page", "sizeParam": "size", "maxPerPage": 200 }, "valuePath": "content", "extractEnabled": "PAX8_COMPANY_ID==null" },
    { "name": "Subscriptions", "role": "extract",   "method": "GET", "urlTemplate": "/v1/subscriptions?companyId={PAX8_COMPANY_ID}",  "pagination": { "type": "pageSize", "pageParam": "page", "sizeParam": "size", "maxPerPage": 200 }, "valuePath": "content", "extractEnabled": "PAX8_COMPANY_ID!=null" },
    { "name": "Orders",        "role": "extract",   "method": "GET", "urlTemplate": "/v1/orders?companyId={PAX8_COMPANY_ID}",         "pagination": { "type": "pageSize" }, "valuePath": "content", "extractEnabled": "PAX8_COMPANY_ID!=null" },
    { "name": "Contacts",      "role": "extract",   "method": "GET", "urlTemplate": "/v1/companies/{PAX8_COMPANY_ID}/contacts",       "pagination": { "type": "pageSize" }, "extractEnabled": "PAX8_COMPANY_ID!=null" }
  ],
  "discovery": {
    "mode": "parentChild",
    "discoverySourceEndpoint": "Companies",
    "environmentSearchPath": "name",
    "aliasTemplate": "Pax8 - {name}",
    "configMapping": { "PAX8_COMPANY_ID": "id" }
  },
  "views": [
    { "title": "Overview", "viewType": "child", "items": [
      { "type": "keyValueTable", "rows": [
        ["Company Name", "{{Companies[0].name}}"],
        ["Subscriptions", "{{length(Subscriptions)}}"],
        ["Open Orders",   "{{length(Orders[?status=='open'])}}"]
      ]}
    ]},
    { "title": "Subscriptions", "viewType": "child", "items": [
      { "type": "dataTable", "columns": ["Product", "Quantity", "Term", "Status"],
        "path": "{{Subscriptions[].[productName, quantity, term, status]}}" }
    ]}
  ],
  "assetMappings": {
    "assetType": "Pax8 Customer",
    "mappings": [
      { "label": "Customer Name",    "query": "Companies[0].name" },
      { "label": "Subscription Count", "query": "length(Subscriptions)" }
    ]
  },
  "metrics": [
    { "name": "Pax8: Subscription Count",      "queries": [{ "query": "length(Subscriptions)",              "inspectorVersionId": "latest" }] },
    { "name": "Pax8: Open Order Count",        "queries": [{ "query": "length(Orders[?status=='open'])",   "inspectorVersionId": "latest" }] },
    { "name": "Pax8: Subscription Names",      "queries": [{ "query": "Subscriptions[].productName | join(', ', @)", "inspectorVersionId": "latest" }], "changesEnabled": true }
  ],
  "rules": [
    { "name": "Pax8 | Subscription list changed",
      "title": "Pax8 | Subscription list changed",
      "body": "Finding: The set of Pax8 subscriptions has changed.\nConcern: Unauthorized purchase / cancellation.\nAction: Confirm the change was intended.",
      "tags": ["pax8", "change-detection"],
      "conditionGroups": [
        { "groupCondition": "and", "priority": "medium",
          "conditions": [{ "metricId": "Pax8: Subscription Names", "operator": "changed", "value": null }] }
      ]
    }
  ]
}
```

`POST /v3/inspectors/import` with `dryRun=true` returns the full sequence of calls the server would issue. Drop `dryRun`, hit it again, and the inspector is provisioned end-to-end.

---

## Round-tripping

`GET /v3/inspectors/{id}/manifest` returns the manifest for an existing inspector. Pass `?environmentId=env_…` to include the environment-scoped `dataprintMetadata` section.

The output is the same shape that `/v3/inspectors/import` consumes — so manifests round-trip:

```bash
# Export
curl -H "X-API-Key: $KEY" \
  "https://api.liongard.com/v3/inspectors/inspector_13/manifest?environmentId=env_8888" \
  > pax8.manifest.json

# Re-import into a different tenant (upsert behaves like an idempotent replace)
curl -H "X-API-Key: $OTHER_KEY" \
  -H "Content-Type: application/json" \
  -X POST https://api.liongard.com/v3/inspectors/import \
  -d "{\"manifest\": $(cat pax8.manifest.json), \"upsert\": true}"
```

Secrets (any ui-config field with `secure: true`) are **not** included in the export; the caller supplies them at launchpoint creation time.
