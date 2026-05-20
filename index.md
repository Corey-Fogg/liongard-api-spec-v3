---
title: Home
layout: home
nav_order: 1
---

# Liongard Data API v3

The structured datalake API for IT infrastructure intelligence. Push data in, pull insights out, and let AI discover what matters — all with rich metadata that makes every field queryable and meaningful.

---

## Getting Started

- **Vendors integrating with Liongard** — start with the [Complete Integration Guide]({% link COMPREHENSIVE_API_GUIDE.md %}) for the full walkthrough.
- **Authoring a custom inspector** — read the [Inspector Manifest Guide]({% link docs/INSPECTOR_MANIFEST_GUIDE.md %}) to describe a complete inspector in one JSON document and provision it with one API call.
- **Hands-on exploration** — open the **Interactive API Explorer** from the top navigation.

---

## Quick Start: push a dataprint

```python
import requests

# 1. Create an inspector (one-time per integration)
response = requests.post(
    "https://api.liongard.com/v3/inspectors",
    headers={"X-API-Key": "your-api-key"},
    json={
        "name": "my-integration",
        "alias": "My Integration",
        "category": "cloud"
    }
)
inspector_id = response.json()["inspectorId"]

# 2. Tell Liongard how to parse your dataprint for one customer environment
requests.put(
    f"https://api.liongard.com/v3/environments/env_123/inspectors/{inspector_id}/config",
    headers={"X-API-Key": "your-api-key"},
    json={
        "metadata": {
            "dataStructure": "split",
            "environments": {"arrayPath": "$.clients", "idPath": "$.id", "namePath": "$.name"},
            "assets":       {"arrayPath": "$.devices", "namePath": "$.hostname", "typePath": "$.type",
                             "uniqueIdPath": "$.serial", "environmentIdPath": "$.client_id"}
        }
    }
)

# 3. Push data
response = requests.post(
    f"https://api.liongard.com/v3/environments/env_123/inspectors/{inspector_id}/dataprints",
    headers={"X-API-Key": "your-api-key"},
    json={
        "clients": [{"id": "c1", "name": "Acme Corp"}],
        "devices": [{"client_id": "c1", "hostname": "srv-01", "type": "server", "serial": "SN001"}]
    }
)
print(f"Processing: {response.json()['jobId']}")
```

Each inspector defines its own dataprint shape — the `dataStructure` config above is how Liongard learns to extract environments and assets out of *your* schema.

---

## Quick Start: define a custom inspector from one manifest

For a vendor inspector with credentials, multiple endpoints, parent/child discovery, views, asset mappings, metrics, and alert rules, author the whole thing once and `POST /v3/inspectors/import`:

```bash
curl -X POST https://api.liongard.com/v3/inspectors/import \
  -H "X-API-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d @my-inspector.manifest.json
```

The manifest is a single JSON document with one section per sub-resource (`definition`, `authentication`, `uiConfigTemplate`, `endpoints`, `discovery`, `views`, `assetMappings`, `metrics`, `rules`). See [the worked Senteon example](https://github.com/Corey-Fogg/liongard-api-spec-v3/blob/main/examples/senteon-manifest.json) and the [Inspector Manifest Guide]({% link docs/INSPECTOR_MANIFEST_GUIDE.md %}).

---

## Key Features

| Feature | Details |
|---------|---------|
| **Inspector manifest framework** | Author a complete custom inspector — definition, auth, endpoints, discovery, views, asset mappings, metrics, alert rules — as one JSON document. `POST /v3/inspectors/import` runs the systematic builder pipeline. |
| **RSQL filtering** | Clean, powerful query syntax across every list endpoint (including each new Inspector Builder endpoint). |
| **Async processing** | Non-blocking dataprint operations with job tracking. |
| **Metrics system** | JMESPath-based data extraction from raw dataprints. |
| **Data Catalog** | Schema discovery and data dictionary for AI-accessible exploration. |
| **Systems** | First-class resource linking inspectors to environments. |
| **Bulk operations** | Push dataprints and evaluate metrics at scale. |
| **Webhooks** | Real-time event notifications for all resource types. |
| **OpenAPI 3.1** | Standards-compliant specification. |
| **No wrappers** | Direct arrays/objects in responses; pagination in RFC 5988 Link headers. |

---

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │          Liongard Datalake              │
Your Product ──────►│  Dataprints → Assets → Metrics          │──────► Your Dashboard
                    │  Environments → Systems → Identities    │──────► AI Agents
Webhooks ◄──────────│  Data Catalog → Schema Discovery        │──────► Reports
                    └─────────────────────────────────────────┘
```

**Core Workflow:**

1. **Create Inspector** — author it as a single [manifest]({% link docs/INSPECTOR_MANIFEST_GUIDE.md %}) or assemble it piece-by-piece via the `Inspector Builder` endpoints.
2. **Configure per Environment** — customer-specific dataprint mapping.
3. **Push Dataprints** — your data, single or bulk.
4. **Track Jobs** — async processing.
5. **Extract Metrics** — insights and aggregations.
6. **Explore Data Catalog** — discover schemas for AI consumption.
7. **Receive Webhooks** — real-time updates.

---

## Use Cases

| Scenario | Flow |
|---|---|
| MSP tool integration | MSP platform → push client data → Liongard → extract metrics → display in dashboard |
| Security tool integration | Security tool → push vulnerability data → Liongard → alerts → remediation |
| AI-powered analytics | AI agent → search data catalog → discover schemas → write JMESPath → generate insights |
| Monitoring integration | Monitoring tool → push device data → Liongard → track changes → create tickets |
