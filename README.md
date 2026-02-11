# Liongard Vendor API v3 Specification

The first external-facing API for Liongard, designed specifically for vendors and partners to integrate their products with the Liongard platform.

**This is NOT a v1/v2 rewrite** - it's a ground-up design built for external vendor integrations.

## 📖 Documentation

**[🔗 Interactive API Explorer](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/Corey-Fogg/liongard-api-spec-v3/refs/heads/main/liongard-api-v3.yaml)**

Browse and test the API directly in Swagger UI.

## 📚 Documentation Files

### Core Documentation
- **[COMPREHENSIVE_API_GUIDE.md](COMPREHENSIVE_API_GUIDE.md)** - Complete vendor integration guide (2,200+ lines)
- **[DESIGN_HISTORY_AND_RATIONALE.md](DESIGN_HISTORY_AND_RATIONALE.md)** - Why v3 exists and how we got here
- **[liongard-api-v3.yaml](liongard-api-v3.yaml)** - OpenAPI 3.1 specification

### Technical References
- **[RSQL_FILTER_GUIDE.md](docs/RSQL_FILTER_GUIDE.md)** - Complete filtering syntax reference
- **[RESPONSE_FORMAT_GUIDE.md](docs/RESPONSE_FORMAT_GUIDE.md)** - Response structure and pagination
- **[JOBS_ASYNC_PROCESSING.md](docs/JOBS_ASYNC_PROCESSING.md)** - Async operations deep dive
- **[METRICS_FEATURE.md](docs/METRICS_FEATURE.md)** - JMESPath-based data extraction
- **[API_COMPLETE_SUMMARY.md](docs/API_COMPLETE_SUMMARY.md)** - Full endpoint reference

### Migration Guides
- **[RSQL_MIGRATION_SUMMARY.md](docs/RSQL_MIGRATION_SUMMARY.md)** - Moving from old filter syntax
- **[RESPONSE_UPDATE_SUMMARY.md](docs/RESPONSE_UPDATE_SUMMARY.md)** - Response format changes

## 🚀 Quick Start

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

## 🔑 Key Features

- ✅ **26 endpoints** - Complete API surface
- ✅ **41 schemas** - Fully defined data models
- ✅ **RSQL filtering** - Clean, powerful query syntax
- ✅ **Async processing** - Non-blocking dataprint operations
- ✅ **Metrics system** - JMESPath-based data extraction
- ✅ **Webhooks** - Real-time event notifications
- ✅ **OpenAPI 3.1** - Standards-compliant specification
- ✅ **No wrappers** - Direct arrays/objects in responses
- ✅ **Header pagination** - RFC 5988 Link header

## 📦 What's Different from v1/v2?

| Feature | v1/v2 (Internal) | v3 (Vendor) |
|---------|------------------|-------------|
| **Purpose** | Internal Liongard UI | External vendor integrations |
| **Responses** | Wrapped in `{data: ...}` | Direct arrays/objects |
| **Pagination** | In response body | In HTTP headers (RFC 5988) |
| **Filtering** | FastAPI syntax (`__in`, `__gte`) | RSQL (`=in=`, `>=`) |
| **Dataprints** | Synchronous (blocks) | Asynchronous (jobs) |
| **Terminology** | "Ingestion" | "Dataprint" |
| **Design Goal** | Optimize for UI | Optimize for vendors |

## 🏗️ Architecture

```
Your Product → Push Dataprints → Liongard API → Process Async → Extract Metrics → Your Dashboard
```

**Core Workflow:**
1. Create Inspector (your integration definition)
2. Configure per Environment (customer-specific mapping)
3. Push Dataprints (your data)
4. Track Jobs (async processing)
5. Extract Metrics (insights and aggregations)
6. Receive Webhooks (real-time updates)

## 🛠️ Development

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

## 🤝 Contributing

When making changes:

1. ✅ Maintain OpenAPI 3.1.0 compliance
2. ✅ Keep response format consistent (no wrappers)
3. ✅ Use RSQL for filtering (not double underscores)
4. ✅ Add pagination headers to list endpoints
5. ✅ Update documentation
6. ✅ Validate before committing

See [.clinerules](.clinerules) for detailed guidelines for Claude Code.

## 📖 Design Philosophy

Read [DESIGN_HISTORY_AND_RATIONALE.md](DESIGN_HISTORY_AND_RATIONALE.md) to understand:
- Why v3 exists
- How design decisions were made
- Evolution from initial concept to current state
- Lessons learned
- Comparison with v1/v2

## 🎯 Use Cases

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

## 📊 API Stats

- **Endpoints**: 26
- **Schemas**: 41
- **OpenAPI Version**: 3.1.0
- **API Version**: 3.0.0
- **Response Wrapper Schemas**: 0 (direct responses)
- **Special Response Schemas**: 6 (jobs, metrics, meta)
- **Validation**: ✅ Zero errors

## 🔗 Links

- **Interactive Docs**: https://petstore.swagger.io/?url=https://raw.githubusercontent.com/Corey-Fogg/liongard-api-spec-v3/refs/heads/main/liongard-api-v3.yaml
- **OpenAPI 3.1 Spec**: https://spec.openapis.org/oas/v3.1.0
- **RSQL Spec**: https://github.com/jirutka/rsql-parser
- **RFC 5988 (Link Header)**: https://tools.ietf.org/html/rfc5988

## 📝 License

[Your License Here]

## 🆘 Support

For questions or issues:
1. Check the [Comprehensive API Guide](COMPREHENSIVE_API_GUIDE.md)
2. Browse existing issues
3. Create a new issue with details

---

**Built for vendors, by vendors.** 🚀
