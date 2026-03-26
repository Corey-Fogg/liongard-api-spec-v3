---
title: Webhooks & Event Subscriptions
parent: Technical References
nav_order: 6
---

# Webhooks & Event Subscriptions

## Overview

Webhooks are how Liongard tells your application that something changed. Instead of polling endpoints, you register a URL and Liongard POSTs to it when events occur — data lands, assets change, metrics cross thresholds, alerts fire.

The webhook system supports:
- Object-level subscriptions (get notified when any alert/asset/dataprint changes)
- RSQL filters (only get events matching specific criteria)
- Metric-based filters (subscribe to metric threshold crossings)
- Delivery modes (full object, diff only, or notification-only)
- Field expansion (embed related objects to reduce follow-up API calls)

---

## Creating a Subscription

Subscriptions are organized by **object type**. Each subscription targets one endpoint URL and can watch multiple object types with independent filters.

```bash
POST /v3/webhooks/subscriptions
```

```json
{
  "name": "Production alerts and asset changes",
  "endpointUrl": "https://yourapp.com/webhooks/liongard",
  "secret": "whsec_your_signing_secret_here",
  "objects": [
    {
      "object": "alerts",
      "events": ["created", "updated"],
      "filter": "priority=in=(high,critical);status==open",
      "expand": ["environment"],
      "delivery": {
        "mode": "full",
        "includeSnapshot": "minimal"
      }
    },
    {
      "object": "assets",
      "events": ["created", "updated"],
      "filter": "environmentId==env_8888;assetType==server",
      "delivery": {
        "mode": "diff"
      }
    }
  ]
}
```

Each entry in `objects` controls:
- **object** — which resource type to watch
- **events** — which lifecycle events to receive
- **filter** — RSQL expression to narrow which events are delivered
- **expand** — embed related objects in the webhook payload
- **delivery** — how much data to include

---

## Subscribable Object Types

| Object | Events | Use Case |
|--------|--------|----------|
| `alerts` | created, updated, deleted | Ticket creation, escalation workflows |
| `assets` | created, updated, deleted | Inventory sync, status monitoring |
| `dataprints` | submitted, completed, failed | Data pipeline tracking |
| `environments` | created, updated, deleted | Tenant lifecycle management |
| `identities` | created, updated, deleted | User provisioning, security monitoring |
| `inspectors` | created, updated, deleted | Integration lifecycle |
| `metrics` | created, updated, deleted, evaluated | Dashboard refresh, threshold alerts |
| `systems` | created, updated, inspected | Inspection scheduling, status tracking |

---

## Filtering with RSQL

Filters use the same RSQL syntax as REST endpoints. Only events matching the filter are delivered.

**Alert examples:**
```
# Only critical alerts
priority==critical

# Open alerts for a specific environment
environmentId==env_8888;status==open

# Alerts from a specific inspector
inspectorId==inspector_13;priority=in=(high,critical)
```

**Asset examples:**
```
# Only servers
assetType==server

# Servers that went offline
assetType==server;status==inactive

# Assets in specific environments
environmentId=in=(env_8888,env_9999);category==compute
```

**Dataprint examples:**
```
# Dataprints for a specific inspector
inspectorId==inspector_13

# Only completed dataprints (skip submitted/failed)
# Use the "completed" event instead of a status filter
```

---

## Metric-Based Webhook Subscriptions

This is where webhooks get powerful. Instead of just watching CRUD events, you can subscribe to metric evaluations and set up threshold-based notifications.

### Subscribing to Metric Evaluations

Watch for when any metric is evaluated (runs after every dataprint completes):

```json
{
  "name": "Track all metric evaluations",
  "endpointUrl": "https://yourapp.com/webhooks/metrics",
  "secret": "whsec_secret",
  "objects": [
    {
      "object": "metrics",
      "events": ["evaluated"],
      "filter": "inspectorId==inspector_13"
    }
  ]
}
```

The `metric.evaluated` event fires after a dataprint is processed and metrics are recalculated. The payload includes the metric ID, the new value, and the previous value — so you can detect changes without polling.

**Payload for `metric.evaluated`:**
```json
{
  "id": "evt_m9876",
  "type": "metric.evaluated",
  "createdAt": "2024-02-06T10:30:15Z",
  "data": {
    "object": {
      "metricId": "metric_42",
      "name": "Active Directory: Count of Disabled Users",
      "inspectorId": "inspector_13",
      "environmentId": "env_8888",
      "systemId": "system_3454",
      "value": 12,
      "previousValue": 8,
      "evaluatedAt": "2024-02-06T10:30:15Z"
    }
  }
}
```

### Metric Threshold Subscriptions

Subscribe to get notified when a metric value crosses a threshold. Use the `metricThresholds` field on a webhook subscription to define conditions:

```json
{
  "name": "Alert when offline devices exceed 5",
  "endpointUrl": "https://yourapp.com/webhooks/thresholds",
  "secret": "whsec_secret",
  "objects": [
    {
      "object": "metrics",
      "events": ["evaluated"],
      "filter": "metricId==metric_offline_count"
    }
  ],
  "metricThresholds": [
    {
      "metricId": "metric_offline_count",
      "condition": "greater_than",
      "value": 5,
      "cooldownMinutes": 60
    }
  ]
}
```

When a `metric.evaluated` event fires and the metric's value matches the threshold condition, the webhook is delivered. The `cooldownMinutes` prevents repeated notifications — after firing once, the threshold won't fire again for that metric+environment until the cooldown expires.

**Threshold conditions:**
- `greater_than` — value > threshold
- `greater_than_or_equal` — value >= threshold
- `less_than` — value < threshold
- `less_than_or_equal` — value <= threshold
- `equal_to` — value == threshold
- `not_equal_to` — value != threshold
- `changed` — value differs from previous evaluation
- `changed_by_more_than` — absolute change exceeds threshold

**Examples:**

```json
// Alert when MFA-disabled users appear
{
  "metricId": "metric_mfa_disabled",
  "condition": "greater_than",
  "value": 0,
  "cooldownMinutes": 30
}

// Alert when device count drops (devices going offline)
{
  "metricId": "metric_total_devices",
  "condition": "changed_by_more_than",
  "value": 10,
  "cooldownMinutes": 120
}

// Alert when a boolean compliance metric flips
{
  "metricId": "metric_password_policy_compliant",
  "condition": "equal_to",
  "value": false,
  "cooldownMinutes": 0
}
```

### Combining Metric Filters with RSQL

You can scope metric threshold subscriptions to specific environments using RSQL filters. The filter applies first (only matching events are considered), then the threshold condition is checked.

```json
{
  "objects": [
    {
      "object": "metrics",
      "events": ["evaluated"],
      "filter": "environmentId=in=(env_8888,env_9999);inspectorId==inspector_13"
    }
  ],
  "metricThresholds": [
    {
      "metricId": "metric_privileged_users",
      "condition": "greater_than",
      "value": 10,
      "cooldownMinutes": 60
    },
    {
      "metricId": "metric_stale_accounts",
      "condition": "greater_than",
      "value": 50,
      "cooldownMinutes": 1440
    }
  ]
}
```

This subscribes to metric evaluations for two specific environments running the Active Directory inspector. It will notify when privileged users exceed 10 (checked hourly) or stale accounts exceed 50 (checked daily).

---

## Delivery Modes

Control how much data is included in each webhook delivery.

### Full Mode
Every webhook includes the complete object:
```json
{
  "delivery": {
    "mode": "full"
  }
}
```

Good for: syncing to external systems, rebuilding state from events.

### Diff Mode
Only changed fields are included (for update events):
```json
{
  "delivery": {
    "mode": "diff"
  }
}
```

Payload includes `previousAttributes` showing what changed:
```json
{
  "type": "asset.updated",
  "data": {
    "object": {
      "assetId": "asset_123",
      "status": "inactive"
    },
    "previousAttributes": {
      "status": "active"
    }
  }
}
```

Good for: change logs, audit trails, lightweight integrations.

### Snapshot Control

The `includeSnapshot` option adds a complete object snapshot alongside diff payloads:

- `none` — no snapshot (just the diff)
- `minimal` — core identifying fields only (ID, name, environment)
- `full` — complete object snapshot

---

## Signature Verification

Every webhook POST includes signature headers for verification:

```
X-Liongard-Signature: sha256=abc123def456...
X-Liongard-Timestamp: 1707217800
```

**Verification steps:**
1. Get the raw request body (before any JSON parsing)
2. Construct the signed string: `{timestamp}.{raw_body}`
3. Compute HMAC-SHA256 using your webhook secret
4. Compare with the `X-Liongard-Signature` header
5. Reject if timestamp is older than 5 minutes (replay protection)

```python
import hmac
import hashlib
import time

def verify_webhook(payload_bytes, signature, timestamp, secret):
    # Replay protection
    if abs(time.time() - int(timestamp)) > 300:
        return False

    signed_payload = f"{timestamp}.{payload_bytes.decode()}"
    expected = hmac.new(
        secret.encode(),
        signed_payload.encode(),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(f"sha256={expected}", signature)
```

---

## Webhook Payload Structure

All webhook deliveries use a consistent envelope:

```json
{
  "id": "evt_1234567890",
  "type": "asset.updated",
  "createdAt": "2024-02-06T10:30:00Z",
  "data": {
    "object": { ... },
    "previousAttributes": { ... },
    "snapshot": { ... }
  }
}
```

| Field | Description |
|-------|-------------|
| `id` | Unique event ID (use for deduplication) |
| `type` | Event type in `object.event` format |
| `createdAt` | When the event occurred |
| `data.object` | The affected object (or changed fields in diff mode) |
| `data.previousAttributes` | Previous values of changed fields (update events, diff mode) |
| `data.snapshot` | Full object snapshot (if `includeSnapshot` is set) |

---

## Handling Webhooks in Production

### Respond Quickly
Return a 2xx status within 10 seconds. Do heavy processing asynchronously:

```python
@app.route("/webhooks/liongard", methods=["POST"])
def handle_webhook():
    event = request.json

    # Queue for async processing — don't block
    task_queue.enqueue(process_event, event)

    return "", 200
```

### Idempotency
Events may be delivered more than once. Use the `id` field to deduplicate:

```python
def process_event(event):
    if redis.sismember("processed_events", event["id"]):
        return  # Already handled

    redis.sadd("processed_events", event["id"])
    redis.expire(f"processed_events", 86400)  # 24h TTL

    # Process the event
    handle(event)
```

### Retry Behavior
Failed deliveries (non-2xx response or timeout) are retried:
- Attempt 1: immediate
- Attempt 2: after 1 minute
- Attempt 3: after 5 minutes
- Attempt 4: after 15 minutes
- Attempt 5: after 60 minutes

After 5 failed attempts, the event is dropped and the subscription is flagged. Consecutive failures may disable the subscription.

---

## Managing Subscriptions

### List Subscriptions
```bash
GET /v3/webhooks/subscriptions
```

### Update a Subscription
```bash
PATCH /v3/webhooks/subscriptions/{subscriptionId}
```

```json
{
  "objects": [
    {
      "object": "alerts",
      "events": ["created", "updated"],
      "filter": "priority==critical"
    }
  ]
}
```

### Disable Without Deleting
```bash
PATCH /v3/webhooks/subscriptions/{subscriptionId}
```

```json
{
  "status": "disabled"
}
```

### Delete a Subscription
```bash
DELETE /v3/webhooks/subscriptions/{subscriptionId}
```

---

## Common Patterns

### Pattern 1: Sync Pipeline Status
Get notified when data ingestion completes or fails:

```json
{
  "objects": [
    {
      "object": "dataprints",
      "events": ["completed", "failed"]
    }
  ]
}
```

### Pattern 2: Security Monitoring
Watch for identity changes and privilege escalation:

```json
{
  "objects": [
    {
      "object": "identities",
      "events": ["created", "updated"],
      "filter": "privileged==true"
    }
  ],
  "metricThresholds": [
    {
      "metricId": "metric_admin_count",
      "condition": "changed",
      "value": 0,
      "cooldownMinutes": 0
    }
  ]
}
```

### Pattern 3: Infrastructure Drift Detection
Know when assets go offline or new ones appear:

```json
{
  "objects": [
    {
      "object": "assets",
      "events": ["created", "updated"],
      "filter": "category==compute",
      "delivery": { "mode": "diff" }
    }
  ],
  "metricThresholds": [
    {
      "metricId": "metric_offline_servers",
      "condition": "greater_than",
      "value": 0,
      "cooldownMinutes": 30
    }
  ]
}
```

### Pattern 4: Compliance Reporting
Get metric evaluation results for compliance dashboards:

```json
{
  "objects": [
    {
      "object": "metrics",
      "events": ["evaluated"],
      "filter": "inspectorId=in=(inspector_13,inspector_15,inspector_22)"
    }
  ]
}
```

Then process each evaluation to build compliance scorecards without ever polling.

---

## Field Expansion in Webhooks

Reduce follow-up API calls by expanding related objects in the webhook payload:

```json
{
  "objects": [
    {
      "object": "alerts",
      "events": ["created"],
      "expand": ["environment", "inspector"]
    }
  ]
}
```

The webhook payload will include the full environment and inspector objects inline, so you don't need to make separate GET calls to resolve them.
