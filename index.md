---
title: Home
layout: home
nav_order: 1
---

# Liongard Vendor API v3

The first external-facing API for Liongard, designed specifically for vendors and partners to integrate their products with the Liongard platform.

---

## Getting Started

Start with the [Complete Integration Guide]({% link COMPREHENSIVE_API_GUIDE.md %}) for a full walkthrough of how to build a vendor integration, from creating an inspector to handling webhooks in production.

For a hands-on look at the endpoints, use the **Interactive API Explorer** link in the top navigation.

---

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

---

## Key Features

| Feature | Details |
|---------|---------|
| **RSQL filtering** | Clean, powerful query syntax |
| **Async processing** | Non-blocking dataprint operations |
| **Metrics system** | JMESPath-based data extraction |
| **Webhooks** | Real-time event notifications |
| **OpenAPI 3.1** | Standards-compliant specification |
| **No wrappers** | Direct arrays/objects in responses |
| **Header pagination** | RFC 5988 Link header |

---

## Architecture

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

---

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
