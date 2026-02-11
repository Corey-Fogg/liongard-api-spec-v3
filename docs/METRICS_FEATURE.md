---
title: Metrics & JMESPath
parent: Technical References
nav_order: 5
---

# Metrics API Feature Documentation

## Overview
Metrics have been added to the Liongard Data API v3 specification. Metrics are JMESPath expressions used to query specific information from dataprints (inspection data), enabling vendors to extract custom data points and build integrations.

---

## What are Metrics?

**Metrics** are named JMESPath queries that extract specific values from inspector dataprints. They enable:

1. **Custom Data Extraction**: Query any field from inspection data
2. **Aggregation**: Count, sum, or aggregate data (e.g., "Count of Domain Computers")
3. **Change Detection**: Track changes in metric values over time
4. **Alert Triggers**: Create alerts based on metric thresholds
5. **Reporting**: Build custom dashboards and reports

### Key Concepts

- **JMESPath**: JSON query language used to extract data
- **Dataprint**: Point-in-time snapshot of inspection data
- **System**: An inspected entity (e.g., Active Directory domain, M365 tenant)
- **Evaluation**: Running a metric's JMESPath query against a dataprint

---

## API Endpoints Added

### 1. List Metrics
```
GET /v3/metrics
```
Returns all available metrics with filtering and pagination.

**Query Parameters:**
- `limit`, `offset` - Pagination
- `sort` - Sort field
- `filter` - FastAPI Filter syntax

**Response:** List of metric objects

### 2. Create Metric
```
POST /v3/metrics
```
Create a custom metric with JMESPath query.

**Request Body:**
```json
{
  "name": "Active Directory: Count of Joined Computers",
  "description": "Number of computers joined to this AD domain",
  "inspectorId": "inspector_13",
  "environmentId": "env_8888",
  "queries": [
    {
      "query": "length(Computers)",
      "inspectorVersionId": "version_26"
    }
  ],
  "metricDisplay": true,
  "changesEnabled": true
}
```

**Key Fields:**
- `queries`: Array of JMESPath queries (one per inspector version)
- `inspectorId`: Which inspector this metric applies to
- `environmentId`: Scope (null = global)
- `metricDisplay`: Show in UI by default
- `changesEnabled`: Track changes over time

### 3. Get Metric
```
GET /v3/metrics/{metricId}
```
Retrieve a specific metric definition.

### 4. Update Metric
```
PATCH /v3/metrics/{metricId}
```
Update a custom metric (system metrics cannot be modified).

### 5. Delete Metric
```
DELETE /v3/metrics/{metricId}
```
Delete a custom metric (system metrics cannot be deleted).

### 6. Evaluate Metrics
```
POST /v3/metrics/evaluate
```
**Most Important Endpoint** - Evaluate metrics across multiple systems and return values.

**Request Body:**
```json
{
  "metrics": [
    "metric_1",
    "aa4eef4d-3dc5-4b3d-9d8a-00c1d9b68065"
  ],
  "filters": [
    {
      "field": "environmentId",
      "op": "equal_to",
      "value": "env_8888"
    }
  ],
  "sorting": [
    {
      "field": "environmentName",
      "direction": "asc"
    }
  ],
  "pagination": {
    "limit": 25,
    "offset": 0
  },
  "date": "2024-02-06T00:00:00Z"
}
```

**Response:**
```json
{
  "data": [
    {
      "systemId": "system_3517",
      "systemName": "Active Directory - Contoso",
      "environmentId": "env_8888",
      "environmentName": "Contoso Nation",
      "metricId": "metric_1",
      "metricName": "Active Directory: Count of Joined Computers",
      "timelineId": "timeline_1332180",
      "evaluatedAt": "2024-02-06T10:30:00Z",
      "value": 7
    }
  ],
  "pagination": {
    "limit": 25,
    "offset": 0,
    "count": 1,
    "hasMore": false
  }
}
```

**Parameters:**
- `includeNonVisible` (query) - Include metrics with display disabled

**Rate Limit:** 100 requests per minute per API key

### 7. Get System Metrics
```
GET /v3/environments/{environmentId}/systems/{systemId}/metrics
```
Get all evaluated metrics for a specific system.

**Response:**
```json
{
  "data": {
    "systemId": "system_3517",
    "systemName": "Active Directory - Contoso",
    "environmentId": "env_8888",
    "latestTimeline": "timeline_1332180",
    "evaluatedAt": "2024-02-06T10:30:00Z",
    "metrics": [
      {
        "metricId": "metric_1",
        "metricName": "Active Directory: Count of Joined Computers",
        "value": 7
      },
      {
        "metricId": "metric_2",
        "metricName": "Active Directory: Privileged Users",
        "value": ["Administrator", "DomainAdmin"]
      }
    ]
  }
}
```

---

## Schema Details

### Metric Object
```yaml
metric:
  metricId: "metric_1"
  uuid: "aa4eef4d-3dc5-4b3d-9d8a-00c1d9b68065"
  name: "Active Directory: Count of Joined Computers"
  description: "Number of computers joined to this Active Directory domain"
  inspectorId: "inspector_13"
  environmentId: "env_8888"  # null for global
  queries:
    - query: "length(Computers)"
      inspectorVersionId: "version_26"
  metricDisplay: true
  changesEnabled: true
  isCustom: false
  hasActiveAlertRule: true
  belongsToChangeRule: false
  createdAt: "2021-09-10T19:21:03Z"
  updatedAt: "2024-02-06T10:30:00Z"
```

### Key Fields Explained

**metricId** - Unique identifier  
**uuid** - UUID for external reference/stability  
**queries** - Array of JMESPath expressions (one per inspector version)  
**metricDisplay** - Whether shown in UI by default  
**changesEnabled** - Track changes over time  
**isCustom** - Custom vs system-provided metric  
**hasActiveAlertRule** - Used in alert rules  
**belongsToChangeRule** - Part of change detection  

---

## Use Cases

### 1. Vendor Integration - Pull Metrics
**Scenario:** MSP tool wants to display AD computer count

```bash
# Step 1: Find the metric
GET /v3/metrics?filter=name__contains=Count of Joined Computers

# Step 2: Evaluate for all clients
POST /v3/metrics/evaluate
{
  "metrics": ["aa4eef4d-3dc5-4b3d-9d8a-00c1d9b68065"],
  "pagination": {"limit": 100, "offset": 0}
}

# Returns values for all systems
```

### 2. Custom Dashboard - Create Metric
**Scenario:** Create custom metric for unlicensed users

```bash
POST /v3/metrics
{
  "name": "M365: Unlicensed Users Count",
  "inspectorId": "inspector_m365",
  "queries": [{
    "query": "length(Users[?LicenseAssignmentStates == `[]`])",
    "inspectorVersionId": "version_latest"
  }],
  "metricDisplay": true,
  "changesEnabled": true
}
```

### 3. Reporting - Evaluate Multiple Metrics
**Scenario:** Generate security report across all clients

```bash
POST /v3/metrics/evaluate
{
  "metrics": [
    "metric_mfa_enabled_users",
    "metric_privileged_users", 
    "metric_stale_accounts"
  ],
  "sorting": [{"field": "environmentName", "direction": "asc"}],
  "pagination": {"limit": 1000, "offset": 0}
}
```

### 4. System Details - Get All Metrics
**Scenario:** Show all metrics for a specific client system

```bash
GET /v3/environments/env_8888/systems/system_3517/metrics?includeNonVisible=true
```

### 5. Partner Integration - Push to External System
**Scenario:** Push metric values to partner platform

```python
# Evaluate metrics
response = requests.post(
    "https://api.liongard.com/v3/metrics/evaluate",
    headers={"X-API-Key": api_key},
    json={
        "metrics": ["metric_1", "metric_2"],
        "pagination": {"limit": 100, "offset": 0}
    }
)

# Push to partner system
for result in response.json()["data"]:
    partner_api.push_metric(
        client_id=result["environmentId"],
        metric_name=result["metricName"],
        value=result["value"],
        timestamp=result["evaluatedAt"]
    )
```

---

## JMESPath Examples

### Simple Field Access
```
Query: name
Result: "Active Directory - Contoso"
```

### Array Length
```
Query: length(Computers)
Result: 42
```

### Filter and Count
```
Query: length(Users[?MfaEnabled == `true`])
Result: 15
```

### Extract Field from Array
```
Query: Users[*].Email
Result: ["user1@example.com", "user2@example.com"]
```

### Complex Query
```
Query: {
  total: length(Users),
  privileged: length(Users[?IsPrivileged == `true`]),
  mfa_enabled: length(Users[?MfaEnabled == `true`])
}
Result: {
  "total": 50,
  "privileged": 5,
  "mfa_enabled": 45
}
```

---

## Best Practices

### 1. Metric Naming
- Use pattern: `{Inspector}: {Description}`
- Examples:
  - Good: "Active Directory: Count of Joined Computers"
  - Good: "M365: Users Without MFA"
  - Bad: "my_metric_1"

### 2. JMESPath Performance
- Keep queries simple when possible
- Avoid deep nesting
- Use filters efficiently
- Test queries on sample data first

### 3. Versioning
- Maintain queries for each inspector version
- Update when data structure changes
- Test new versions before deploying

### 4. Scope
- Use global metrics for common queries
- Use environment-scoped for client-specific metrics
- Consider reusability

### 5. Change Detection
- Enable `changesEnabled` for metrics that should trigger alerts
- Use for compliance tracking
- Monitor critical security settings

---

## Integration Patterns

### Pattern 1: Periodic Sync
```python
# Run every hour
def sync_metrics():
    # Get all metric values
    response = evaluate_metrics(
        metrics=["metric_1", "metric_2"],
        pagination={"limit": 1000}
    )
    
    # Update external system
    for metric_value in response["data"]:
        external_db.upsert(metric_value)
```

### Pattern 2: On-Demand Query
```python
# When user requests data
def get_client_metrics(environment_id):
    return evaluate_metrics(
        metrics=["metric_1", "metric_2"],
        filters=[{
            "field": "environmentId",
            "op": "equal_to",
            "value": environment_id
        }]
    )
```

### Pattern 3: Webhook + Metrics
```python
# When webhook fires for new dataprint
@webhook_handler
def on_dataprint_created(event):
    system_id = event["data"]["object"]["systemId"]
    
    # Evaluate metrics for this system
    metrics = get_system_metrics(
        environment_id=event["environmentId"],
        system_id=system_id
    )
    
    # Process/store/alert based on values
    process_metrics(metrics)
```

---

## Rate Limits

- **Metric Evaluate**: 100 requests/minute per API key
- **Other Endpoints**: Standard API rate limits apply

---

## Error Handling

### Evaluation Errors
If a JMESPath query fails, the response includes an error:

```json
{
  "metricId": "metric_1",
  "metricName": "...",
  "value": null,
  "error": "JMESPath query failed: invalid syntax at position 10"
}
```

### Common Errors
- **Invalid JMESPath**: Syntax error in query
- **Missing Data**: Query references non-existent field
- **Type Mismatch**: Query expects different data type
- **No Dataprint**: System has no inspection data

---

## Summary

- **JMESPath-based** querying for flexibility
- **Custom metrics** support for vendor-specific needs
- **Bulk evaluation** for efficient data retrieval
- **Change tracking** for monitoring over time
- **Rate limited** to ensure API stability

Metrics enable vendors to:
- Extract specific data points from inspections
- Build custom dashboards and reports
- Create integrations with external systems
- Track changes and trigger alerts
- Aggregate data across multiple clients

See the [Interactive API Explorer](swagger-ui) for the full list of metric endpoints and schemas.
