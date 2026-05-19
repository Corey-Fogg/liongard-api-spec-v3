---
nav_exclude: true
---

# Liongard Data API v3

The structured datalake API for IT infrastructure intelligence. Liongard captures, normalizes, and structures operational data from across your IT environment. This API enables vendors and integrations to **push data in**, **pull insights out**, and **react to changes** in real time.

Every piece of data is tagged with rich metadata — environment context, inspector lineage, asset classification, and schema mappings — so it can be discovered, queried, and consumed by both humans and AI systems.

## Documentation

**[Interactive API Explorer](swagger-ui)**

Browse and test the API directly in Swagger UI.

## Documentation Files

### Core Documentation
- **[COMPREHENSIVE_API_GUIDE.md](COMPREHENSIVE_API_GUIDE.md)** - Complete vendor integration guide
- **[DESIGN_HISTORY_AND_RATIONALE.md](DESIGN_HISTORY_AND_RATIONALE.md)** - Why v3 exists and how we got here
- **[liongard-api-v3.yaml](liongard-api-v3.yaml)** - OpenAPI 3.1 specification

### Technical References
- **[INSPECTOR_MANIFEST_GUIDE.md](docs/INSPECTOR_MANIFEST_GUIDE.md)** - Single-document JSON framework for provisioning a complete custom inspector via API
- **[RSQL_FILTER_GUIDE.md](docs/RSQL_FILTER_GUIDE.md)** - Complete filtering syntax reference
- **[RESPONSE_FORMAT_GUIDE.md](docs/RESPONSE_FORMAT_GUIDE.md)** - Response structure and pagination
- **[JOBS_ASYNC_PROCESSING.md](docs/JOBS_ASYNC_PROCESSING.md)** - Async operations deep dive
- **[METRICS_FEATURE.md](docs/METRICS_FEATURE.md)** - JMESPath-based data extraction
- **[WEBHOOKS_GUIDE.md](docs/WEBHOOKS_GUIDE.md)** - Webhooks, metric thresholds, and event subscriptions

### Schemas & Examples
- **[inspector-manifest.schema.json](inspector-manifest.schema.json)** - Standalone JSON Schema for the inspector manifest framework
- **[examples/senteon-manifest.json](examples/senteon-manifest.json)** - Worked example: a complete Senteon inspector authored as one manifest document


## Quick Start

```python
import requests

# 1. Create inspector
response = requests.post(
    "https://api.liongard.com/v3/inspectors",
    headers={"X-API-Key": "your-api-key"},
    json={
        "name": "my-integration",
        "displayName": "My Integration",
        "category": "monitoring"
    }
)
inspector_id = response.json()["inspectorId"]

# 2. Configure for environment
requests.put(
    f"https://api.liongard.com/v3/environments/env_123/inspectors/{inspector_id}/config",
    headers={"X-API-Key": "your-api-key"},
    json={
        "metadata": {
            "dataStructure": "split",
            "environments": {
                "arrayPath": "$.clients",
                "idPath": "$.id",
                "namePath": "$.name"
            },
            "assets": {
                "arrayPath": "$.devices",
                "namePath": "$.hostname",
                "typePath": "$.type",
                "uniqueIdPath": "$.serial",
                "environmentIdPath": "$.client_id"
            }
        }
    }
)

# 3. Push data
response = requests.post(
    f"https://api.liongard.com/v3/environments/env_123/inspectors/{inspector_id}/dataprints",
    headers={"X-API-Key": "your-api-key"},
    json={
        "clients": [{"id": "c1", "name": "Acme Corp"}],
        "devices": [{
            "client_id": "c1",
            "hostname": "srv-01",
            "type": "server",
            "serial": "SN001"
        }]
    }
)

job_id = response.json()["jobId"]
print(f"Processing: {job_id}")
```

## Key Features

- **Inspector manifest framework** - Author an entire custom inspector (definition, auth, endpoints, discovery, views, asset mappings, metrics, rules) in one JSON document; `POST /v3/inspectors/import` runs the systematic pipeline
- **RSQL filtering** - Clean, powerful query syntax across all list endpoints (including every new builder endpoint)
- **Async processing** - Non-blocking dataprint operations with job tracking
- **Metrics system** - JMESPath-based data extraction from raw dataprints
- **Data Catalog** - Schema discovery and data dictionary for AI-accessible exploration
- **Systems** - First-class resource linking inspectors to environments
- **Bulk operations** - Push dataprints and evaluate metrics at scale
- **Webhooks** - Real-time event notifications for all resource types
- **OpenAPI 3.1** - Standards-compliant specification
- **No wrappers** - Direct arrays/objects in responses
- **Header pagination** - RFC 5988 Link header

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │          Liongard Datalake               │
Your Product ──────►│  Dataprints → Assets → Metrics          │──────► Your Dashboard
                    │  Environments → Systems → Identities    │──────► AI Agents
Webhooks ◄──────────│  Data Catalog → Schema Discovery        │──────► Reports
                    └─────────────────────────────────────────┘
```

**Core Workflow:**
1. Create Inspector (your integration definition)
   - Author a complete inspector with a single manifest document — see the [Inspector Manifest Guide](docs/INSPECTOR_MANIFEST_GUIDE.md) — or assemble it piece-by-piece via the `Inspector Builder` endpoints
2. Configure per Environment (customer-specific mapping)
3. Push Dataprints (your data — single or bulk)
4. Track Jobs (async processing)
5. Extract Metrics (insights and aggregations)
6. Explore Data Catalog (discover schemas for AI consumption)
7. Receive Webhooks (real-time updates)

## Development

### Validate Spec

```bash
# YAML syntax
python3 -c "import yaml; yaml.safe_load(open('liongard-api-v3.yaml'))"

# OpenAPI validation
npx @apidevtools/swagger-cli validate liongard-api-v3.yaml
```

### View Locally

```bash
# Swagger UI
docker run -p 8080:8080 -e SWAGGER_JSON=/spec/liongard-api-v3.yaml \
  -v $(pwd):/spec swaggerapi/swagger-ui

# Redoc
npx @redocly/cli preview-docs liongard-api-v3.yaml
```

## Contributing

When making changes:

1. Maintain OpenAPI 3.1.0 compliance
2. Keep response format consistent (no wrappers)
3. Use RSQL for filtering (not double underscores)
4. Add pagination headers to list endpoints
5. Update documentation
6. Validate before committing

See [.clinerules](.clinerules) for detailed guidelines for Claude Code.

## Design Philosophy

Read [DESIGN_HISTORY_AND_RATIONALE.md](DESIGN_HISTORY_AND_RATIONALE.md) to understand the design decisions behind the API.

## Use Cases

**MSP Tool Integration:**
```
Your MSP Platform → Push client data → Liongard → Extract metrics → Display in dashboard
```

**Security Tool Integration:**
```
Your Security Tool → Push vulnerability data → Liongard → Get alerts → Trigger remediation
```

**Monitoring Integration:**
```
Your Monitoring Tool → Push device data → Liongard → Track changes → Create tickets
```

## Links

- **Interactive Docs**: [Swagger UI](swagger-ui)
- **OpenAPI 3.1 Spec**: https://spec.openapis.org/oas/v3.1.0
- **RSQL Spec**: https://github.com/jirutka/rsql-parser
- **RFC 5988 (Link Header)**: https://tools.ietf.org/html/rfc5988

## License

[Your License Here]

## Support

For questions or issues:
1. Check the [Comprehensive API Guide](COMPREHENSIVE_API_GUIDE.md)
2. Browse existing issues
3. Create a new issue with details

---

**Built for vendors. Structured for AI. Designed for scale.**
