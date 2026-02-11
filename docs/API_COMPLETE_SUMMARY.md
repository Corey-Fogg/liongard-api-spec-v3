---
title: Endpoint Reference
parent: Technical References
nav_order: 1
---

# Liongard Data API v3 - Complete Feature Summary

## Overview
This document summarizes the complete Liongard Data API v3 specification with all enhancements, including schema improvements from v1/v2 analysis, metrics support, and asynchronous job processing.

---

## API Statistics

- **OpenAPI Version**: 3.1.0
- **API Version**: 3.0.0  
- **Total Endpoints**: 27 unique paths
- **Total Schemas**: 62 defined
- **Authentication**: API Key (X-API-Key header)
- **Format**: JSON (application/json)
- **Date/Time Format**: ISO 8601 UTC

---

## Resource Categories

### 1. Alerts (5 endpoints)
Manage security and operational alerts with full lifecycle support.

- `GET /v3/alerts` - List alerts with filtering
- `POST /v3/alerts` - Create alert
- `GET /v3/alerts/{alertId}` - Get alert details
- `PATCH /v3/alerts/{alertId}` - Update alert (including close)
- ~~`DELETE /v3/alerts/{alertId}`~~ - NOT SUPPORTED (use PATCH to close)

**Key Features:**
- Priority levels: low, medium, high, critical
- Status workflow: open → in_progress → resolved/closed
- Auto-close after N days
- External system integration (PSA tickets)
- Audit trail (createdBy, updatedBy, closedBy)

**Enhanced Fields (26 total):**
- Timeline/launchpoint references
- Percentage complete tracking
- Keywords for search
- Common ID for grouping
- Detailed error context

### 2. Assets (4 endpoints) - READ ONLY
Device and hardware inventory automatically populated from inspector data.

- `GET /v3/assets` - List assets with filtering
- `GET /v3/assets/{assetId}` - Get asset details
- `GET /v3/environments/{environmentId}/assets` - Assets by environment
- `GET /v3/environments/{environmentId}/assets/{assetId}` - Env-scoped asset

**Key Features:**
- Read-only (populated via inspector mappings)
- Inventory states: inventory, archive, discovery
- Asset categories: compute, network, storage, security
- Asset types: server, workstation, firewall, router, etc.
- Lifecycle tracking (purchase, warranty, EOL dates)

**Enhanced Fields (41 total):**
- Hardware details (manufacturer, model, serial)
- Network info (IP, MAC addresses)
- Virtualization (host, cluster, hypervisor)
- Location and classification
- Management status

### 3. Environments (4 endpoints)
Client/customer organizations that contain assets and inspectors.

- `GET /v3/environments` - List environments
- `POST /v3/environments` - Create environment
- `GET /v3/environments/{environmentId}` - Get environment
- `PATCH /v3/environments/{environmentId}` - Update environment

**Key Features:**
- Hierarchical (parent/child relationships)
- Tiers: production, staging, development, test
- Status tracking
- Child environment support

### 4. Identities (4 endpoints) - READ ONLY
User accounts and service principals from inspector data.

- `GET /v3/identities` - List identities
- `GET /v3/identities/{identityId}` - Get identity details
- `GET /v3/environments/{environmentId}/identities` - Identities by environment
- `GET /v3/environments/{environmentId}/identities/{identityId}` - Env-scoped identity

**Key Features:**
- Read-only (populated via inspector mappings)
- Identity types: user, admin, service, guest, shared, system, application
- Inventory states: inventory, archive, discovery
- Security tracking (MFA, privileged access)
- Credential lifecycle (password expiration, account expiration)

**Enhanced Fields (31 total):**
- Authentication details (MFA, last login)
- Password tracking (last set, expiration)
- Organizational info (department, job title, manager)
- Source system tracking
- Privileged access flags

### 5. Inspectors (11 endpoints)
Data collection configurations and ingestion.

**Global Endpoints:**
- `GET /v3/inspectors` - List all inspectors
- `POST /v3/inspectors` - Create inspector
- `GET /v3/inspectors/{inspectorId}` - Get inspector
- `PATCH /v3/inspectors/{inspectorId}` - Update inspector
- `DELETE /v3/inspectors/{inspectorId}` - Delete inspector

**Environment-Scoped Endpoints:**
- `GET /v3/environments/{envId}/inspectors` - List for environment
- `GET /v3/environments/{envId}/inspectors/{inspectorId}` - Get config
- `GET /v3/environments/{envId}/inspectors/{inspectorId}/config` - Get metadata config
- `PUT /v3/environments/{envId}/inspectors/{inspectorId}/config` - Set metadata config
- `POST /v3/environments/{envId}/inspectors/{inspectorId}/dataprints` - Push data (async)
- `GET /v3/environments/{envId}/inspectors/{inspectorId}/dataprints/{dataprintId}` - Get dataprint

**Key Features:**
- Global inspector definitions used across environments
- Environment-specific configurations
- Metadata mapping (JSONPath-based)
- Two data structures: tiered and split
- Asynchronous dataprint processing
- Version tracking

**Enhanced Fields (20 total):**
- Publishing metadata (logo, icon, help links)
- Categorization and status
- Frequency recommendations
- Platform constraints
- Billing implications

### 6. Jobs (2 endpoints) - NEW
Asynchronous operation tracking for long-running tasks.

- `GET /v3/jobs/{jobId}` - Get job status and progress
- `DELETE /v3/jobs/{jobId}` - Cancel pending or processing job
- `GET /v3/jobs` - List all jobs with filtering (use filter parameter for environmentId, inspectorId, status, type, etc.)

**Key Features:**
- Async dataprint processing
- Real-time progress tracking (percentage, messages)
- Job statuses: pending, processing, completed, failed, cancelled
- Optional callback URLs for one-time notifications
- Error details for failed jobs
- 30-day retention
- Unified filtering via filter parameter

**Filtering Examples:**
- `?filter=status=in=pending,processing` - Active jobs
- `?filter=environmentId=env_8888&inspectorId=inspector_13` - Specific inspector
- `?filter=type=dataprint_processing&status=failed` - Failed dataprints

**Concurrency Rules:**
- **ONE job per inspector per environment** (enforced)
- 5 concurrent jobs per environment
- 10 concurrent jobs per API key
- 409 Conflict if concurrent job exists

**Use Cases:**
- Track dataprint processing
- Cancel incorrect submissions
- Check if slot available before submitting
- Monitor job history for debugging

### 7. Metrics (6 endpoints) - NEW
JMESPath-based data extraction from dataprints.

- `GET /v3/metrics` - List metrics
- `POST /v3/metrics` - Create custom metric
- `GET /v3/metrics/{metricId}` - Get metric definition
- `PATCH /v3/metrics/{metricId}` - Update custom metric
- `DELETE /v3/metrics/{metricId}` - Delete custom metric
- `POST /v3/metrics/evaluate` - Evaluate metrics across systems
- `GET /v3/environments/{envId}/systems/{systemId}/metrics` - Get all metrics for system

**Key Features:**
- JMESPath query language
- Custom metric creation
- Bulk evaluation (multiple metrics, multiple systems)
- Filtering by environment, system, inspector
- Historical evaluation (specify date)
- Change detection tracking
- Alert rule integration

**Use Cases:**
- Extract specific data points (e.g., "Count of Domain Computers")
- Build custom dashboards
- Push data to partner systems
- Create reports across all clients
- Track changes over time

### 8. Meta (2 endpoints)
API capability discovery.

- `GET /v3/meta/objects` - List filterable/expandable fields per object
- `GET /v3/meta/assets/types` - Asset type capabilities

**Key Features:**
- Dynamic field discovery
- No expand-all wildcard (explicit only)
- RSQL syntax
- Supported events for webhooks

### 9. Webhooks (4 endpoints)
Event subscription for real-time updates.

- `GET /v3/webhooks/subscriptions` - List subscriptions
- `POST /v3/webhooks/subscriptions` - Create subscription
- `GET /v3/webhooks/subscriptions/{subscriptionId}` - Get subscription
- `PATCH /v3/webhooks/subscriptions/{subscriptionId}` - Update subscription
- `DELETE /v3/webhooks/subscriptions/{subscriptionId}` - Delete subscription

**Key Features:**
- Subscribe to specific events (alert.created, dataprint.created, etc.)
- Delivery modes: full, diff, none
- HMAC signature verification
- Automatic retries with exponential backoff
- Status tracking (enabled/disabled)

---

## Key Design Patterns

### 1. Asynchronous Processing
**Pattern**: Long operations return 202 Accepted with job ID

```bash
POST /v3/.../dataprints
← 202 Accepted
{
  "jobId": "job_abc123",
  "pollUrl": "https://api.../v3/jobs/job_abc123"
}

GET /v3/jobs/job_abc123
← 200 OK
{
  "status": "completed",
  "result": { ... }
}
```

### 2. Concurrency Control
**Pattern**: One job per resource at a time

```bash
POST /v3/.../dataprints
← 409 Conflict
{
  "error": {
    "code": "CONCURRENT_JOB_CONFLICT",
    "details": {
      "existingJobId": "job_xyz",
      "suggestion": "Cancel using DELETE /v3/jobs/job_xyz"
    }
  }
}
```

### 3. Pagination
**Pattern**: Consistent limit/offset with hasMore indicator

```json
{
  "data": [...],
  "pagination": {
    "limit": 100,
    "offset": 0,
    "count": 547,
    "hasMore": true
  }
}
```

### 4. Filtering
**Pattern**: RSQL (RESTful Service Query Language)

```
?filter=status==active
?filter=priority==critical;status==open
?filter=status=in=(pending,processing)
?filter=createdAt>=2024-01-01;createdAt<2024-02-01
```

**Operators:**
- `==` Equal
- `!=` Not equal
- `>` `>=` `<` `<=` Comparisons
- `=in=()` In list
- `*` Wildcards
- `;` AND
- `,` OR

### 5. Field Selection
**Pattern**: Explicit expand parameter

```
?expand=environment,inspector
```

### 6. Resource Expansion
**Pattern**: Related resources embedded

```json
{
  "alertId": "alert_123",
  "environment": {
    "environmentId": "env_456",
    "name": "Acme Corp"
  }
}
```

---

## Authentication & Security

### API Key Authentication
```http
X-API-Key: your-api-key-here
```

### Rate Limiting
- Standard: Varies by endpoint
- Metric Evaluation: 100 requests/minute
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

### Webhook Signatures
```http
X-Liongard-Signature: sha256=abc123...
X-Liongard-Timestamp: 1707217800
```

Verify using HMAC-SHA256 with webhook secret.

---

## Data Structures

### Inspector Data Ingestion

**Tiered Structure** (environments contain assets):
```json
{
  "clients": [
    {
      "id": "client_001",
      "name": "Acme Corp",
      "devices": [
        { "hostname": "dc01", "type": "server" }
      ]
    }
  ]
}
```

**Split Structure** (flat arrays with foreign keys):
```json
{
  "clients": [
    { "id": "client_001", "name": "Acme Corp" }
  ],
  "devices": [
    { "client_id": "client_001", "hostname": "dc01" }
  ]
}
```

### Metadata Mapping
```json
{
  "dataStructure": "split",
  "environments": {
    "arrayPath": "$.clients",
    "namePath": "$.name",
    "idPath": "$.id"
  },
  "assets": {
    "arrayPath": "$.devices",
    "namePath": "$.hostname",
    "typePath": "$.type",
    "uniqueIdPath": "$.serial_number",
    "environmentIdPath": "$.client_id"
  }
}
```

---

## Error Handling

### Standard Error Response
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "field": "priority",
      "issue": "Must be one of: low, medium, high, critical"
    }
  }
}
```

### HTTP Status Codes
- `200` OK - Success
- `201` Created - Resource created
- `202` Accepted - Async operation started
- `204` No Content - Deleted successfully
- `400` Bad Request - Invalid input
- `401` Unauthorized - Invalid API key
- `404` Not Found - Resource doesn't exist
- `409` Conflict - Concurrent operation or version mismatch
- `422` Unprocessable Entity - Validation failed
- `429` Too Many Requests - Rate limited

---

## Breaking Changes from v1/v2

### Dataprint Submission
**v1/v2**: Synchronous, returns result immediately
```bash
POST /v1/dataprints
← 201 Created (after 30-120s)
{ "accepted": true, "assetsProcessed": 5 }
```

**v3**: Asynchronous, returns job ID immediately
```bash
POST /v3/.../dataprints
← 202 Accepted (instant)
{ "jobId": "job_123", "status": "pending" }
```

### Pagination
**v1/v2**: `page` and `pageSize`
```bash
?page=1&pageSize=50
```

**v3**: `limit` and `offset`
```bash
?limit=50&offset=0
```

### Filtering
**v1/v2**: `conditions[]` with complex syntax
```bash
?conditions[]={"path":"Status","op":"eq","value":"active"}
```

**v3**: RSQL syntax
```bash
?filter=status==active
```

### ID Format
**v1/v2**: Integers
```json
{ "ID": 1234 }
```

**v3**: Strings
```json
{ "alertId": "alert_1234" }
```

---

## Best Practices

### 1. Dataprint Submission
- Check for active jobs before submitting
```python
can_submit, reason, job_id = can_submit_dataprint(env_id, inspector_id, api_key)
if not can_submit:
    cancel_job(job_id, api_key)
```

- Use callback URLs for production
```python
submit_dataprint(env_id, inspector_id, data, api_key, 
                 callback_url="https://yourapp.com/webhook")
```

- Handle 409 Conflicts gracefully
```python
try:
    submit_dataprint(...)
except ConflictError as e:
    existing_job = e.details["existingJobId"]
    # Wait or cancel
```

### 2. Job Polling
- Use adaptive intervals based on progress
```python
if progress < 20:
    interval = 5  # Slow start
elif progress < 80:
    interval = 3  # Normal
else:
    interval = 1  # Near completion
```

- Set reasonable timeouts
```python
wait_for_job(job_id, timeout=300)  # 5 minutes
```

### 3. Metrics
- Use bulk evaluation for efficiency
```python
# Evaluate multiple metrics across all systems at once
evaluate_metrics(metrics=["metric_1", "metric_2", "metric_3"])
```

- Cache metric definitions
```python
# Fetch once, use many times
metrics = list_metrics()
metric_map = {m["metricId"]: m for m in metrics}
```

### 4. Webhooks
- Always verify signatures
```python
if not verify_signature(payload, signature, secret):
    return 401
```

- Return 2xx quickly, process asynchronously
```python
@webhook_handler
def handle_webhook(payload):
    queue.enqueue(process_webhook, payload)
    return 200  # Return immediately
```

### 5. Error Handling
- Check error codes for specific actions
```python
if error["code"] == "CONCURRENT_JOB_CONFLICT":
    job_id = error["details"]["existingJobId"]
    cancel_job(job_id)
    retry()
```

---

## Migration Checklist

- [ ] Update authentication to use `X-API-Key` header
- [ ] Change pagination from page/pageSize to limit/offset
- [ ] Update filter syntax to RSQL
- [ ] Handle async dataprint responses (202 + job polling)
- [ ] Convert integer IDs to string IDs
- [ ] Update error handling for new error format
- [ ] Implement job cancellation logic
- [ ] Add concurrency checking before dataprint submission
- [ ] Update timestamp parsing to ISO 8601
- [ ] Test webhook signature verification

---

## Production Readiness Checklist

### API Implementation
- [ ] All endpoints return correct status codes
- [ ] Error responses include actionable messages
- [ ] Rate limiting implemented and documented
- [ ] CORS configured if needed
- [ ] API versioning strategy defined

### Async Processing
- [ ] Job queue infrastructure in place
- [ ] Progress tracking implemented
- [ ] Callback delivery with retries
- [ ] Job retention policy (30 days)
- [ ] Concurrency limits enforced

### Data Integrity
- [ ] Prevent concurrent dataprint processing per inspector/environment
- [ ] Validate data against config schema
- [ ] Handle partial failures gracefully
- [ ] Rollback mechanism for failed jobs

### Monitoring
- [ ] Job status dashboard
- [ ] Failed job alerts
- [ ] Rate limit monitoring
- [ ] Webhook delivery monitoring
- [ ] Slow job detection

### Security
- [ ] API keys securely generated and stored
- [ ] Webhook signature verification
- [ ] HTTPS only for callbacks
- [ ] Input validation on all endpoints
- [ ] SQL injection prevention

### Documentation
- [ ] API reference published
- [ ] Code examples for each language
- [ ] Migration guide from v1/v2
- [ ] Troubleshooting guide
- [ ] Changelog maintained

---

## Summary

**Liongard Data API v3** provides a comprehensive, production-ready API for vendor integrations with:

- **27 endpoints** covering all core resources  
- **62 schemas** with rich field definitions  
- **Asynchronous processing** for scalability  
- **Job management** with cancellation support  
- **Metrics system** for flexible data extraction  
- **Webhook subscriptions** for real-time updates  
- **Concurrency control** preventing data conflicts  
- **OpenAPI 3.1 compliance** with zero warnings  

**Key Differentiators from v1/v2:**
- Non-blocking dataprint ingestion
- Comprehensive job tracking
- Custom metrics support
- Enhanced schema definitions
- Modern REST patterns
- Better error handling

**Ready for:**
- Production vendor integrations
- MSP tool integrations
- Partner data exchanges
- Custom dashboard development
- Automated workflows

**Next Steps:**
1. Review endpoint documentation
2. Test in sandbox environment
3. Implement async job handling
4. Add concurrency checks
5. Deploy to production
