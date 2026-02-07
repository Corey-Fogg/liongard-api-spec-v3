# Liongard Vendor API v3 - Complete Guide

## Table of Contents
1. [What is This API?](#what-is-this-api)
2. [Who Should Use This?](#who-should-use-this)
3. [Quick Start](#quick-start)
4. [Core Concepts](#core-concepts)
5. [The Data Flow](#the-data-flow)
6. [Inspector Configuration](#inspector-configuration)
7. [Pushing Data (Dataprints)](#pushing-data-dataprints)
8. [Asynchronous Processing](#asynchronous-processing)
9. [Extracting Data with Metrics](#extracting-data-with-metrics)
10. [Real-Time Updates with Webhooks](#real-time-updates-with-webhooks)
11. [Querying Resources](#querying-resources)
12. [Complete Integration Example](#complete-integration-example)
13. [Production Deployment](#production-deployment)
14. [Troubleshooting](#troubleshooting)

---

## What is This API?

The Liongard Vendor API v3 is **the first external-facing API** designed specifically for vendors and partners to integrate their products with Liongard. This is NOT a rewrite of internal v1/v2 APIs - it's a brand new API built from the ground up for vendor integrations.

### What It Does

**Think of it as a two-way data bridge:**

1. **Vendors push data INTO Liongard** (your product's data)
2. **Vendors pull insights OUT of Liongard** (metrics, aggregations, changes)

### Real-World Use Cases

**Scenario 1: MSP Tool Integration**
```
Your MSP Platform → Push client data → Liongard → Extract metrics → Display in your dashboard
```

**Scenario 2: Security Tool Integration**
```
Your Security Tool → Push vulnerability data → Liongard → Get alerts → Trigger remediation
```

**Scenario 3: Monitoring Integration**
```
Your Monitoring Tool → Push device data → Liongard → Track changes → Create tickets
```

---

## Who Should Use This?

### You Should Use This API If You Are:

✅ **A vendor** building an integration with Liongard  
✅ **An MSP platform** wanting to sync data with Liongard  
✅ **A security tool** pushing security data  
✅ **A monitoring solution** sending infrastructure data  
✅ **A partner** building custom integrations  

### You Should NOT Use This If You Are:

❌ Building internal Liongard features (use internal APIs)  
❌ Creating Liongard UI components (this is headless)  
❌ Looking for a GraphQL API (this is REST)  

---

## Quick Start

### Step 1: Get Your API Key

Contact Liongard to obtain your API key:
```
X-API-Key: lg_vendor_abc123def456...
```

### Step 2: Create an Inspector

An **Inspector** defines what kind of data you're sending.

```bash
curl -X POST "https://api.liongard.com/v3/inspectors" \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "acme-monitoring",
    "displayName": "Acme Monitoring Tool",
    "description": "Device monitoring data from Acme platform",
    "category": "monitoring"
  }'
```

**Response:**
```json
{
  "inspectorId": "inspector_abc123",
  "name": "acme-monitoring",
  "displayName": "Acme Monitoring Tool",
  "status": "active"
}
```

### Step 3: Configure the Inspector for an Environment

Tell Liongard HOW to parse your data structure.

```bash
curl -X PUT "https://api.liongard.com/v3/environments/env_8888/inspectors/inspector_abc123/config" \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {
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
  }'
```

### Step 4: Push Your First Dataprint

```bash
curl -X POST "https://api.liongard.com/v3/environments/env_8888/inspectors/inspector_abc123/dataprints" \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "clients": [
      {
        "id": "client_001",
        "name": "Acme Corp"
      }
    ],
    "devices": [
      {
        "client_id": "client_001",
        "hostname": "acme-srv-01",
        "type": "server",
        "serial_number": "SN12345",
        "ip_address": "192.168.1.10",
        "status": "online"
      }
    ]
  }'
```

**Response (Async Job):**
```json
{
  "jobId": "job_xyz789",
  "status": "pending",
  "message": "Dataprint accepted for processing",
  "pollUrl": "https://api.liongard.com/v3/jobs/job_xyz789"
}
```

### Step 5: Check Job Status

```bash
curl -X GET "https://api.liongard.com/v3/jobs/job_xyz789" \
  -H "X-API-Key: your-api-key"
```

**Response:**
```json
{
  "jobId": "job_xyz789",
  "status": "completed",
  "result": {
    "accepted": true,
    "environmentsProcessed": 1,
    "assetsProcessed": 1
  }
}
```

🎉 **Done!** Your data is now in Liongard.

---

## Core Concepts

### 1. Inspectors

**What:** A definition of your integration  
**Think of it as:** The "plugin" for your product  
**Created once:** Yes, reused across all customers  

```
Inspector = "What kind of data am I sending?"
```

**Example Inspectors:**
- `acme-monitoring` - Device monitoring data
- `acme-security` - Vulnerability scans
- `acme-backup` - Backup status reports

### 2. Environments

**What:** A customer/client/tenant in Liongard  
**Think of it as:** Your customer's workspace  
**Created by:** Liongard (you reference them)  

```
Environment = "Which customer is this data for?"
```

**Example:**
- `env_8888` = "Contoso Corporation"
- `env_9999` = "Fabrikam Industries"

### 3. Dataprints

**What:** A snapshot of your data at a point in time  
**Think of it as:** A JSON payload you send to Liongard  
**Frequency:** Whenever your data changes (hourly, daily, on-demand)  

```
Dataprint = "Here's my current data"
```

**Contains:**
- Environments (your customers/tenants)
- Assets (devices, servers, workstations, etc.)
- Any other data you want to track

### 4. Assets

**What:** Devices, servers, users, or any trackable entity  
**Extracted from:** Your dataprints (automatically)  
**Used for:** Inventory, change detection, alerting  

```
Asset = "A thing being tracked" (server, router, user account, etc.)
```

### 5. Identities

**What:** User accounts, service principals, API keys  
**Extracted from:** Your dataprints (if you include them)  
**Used for:** User tracking, security monitoring  

```
Identity = "A user or account"
```

### 6. Metrics

**What:** Custom queries against your dataprint data  
**Think of it as:** SQL-like queries but using JMESPath  
**Used for:** Extracting specific values, building dashboards  

```
Metric = "Give me the count of offline servers"
```

### 7. Jobs

**What:** Background tasks processing your dataprints  
**Think of it as:** Async operations that may take time  
**Status:** pending → processing → completed/failed  

```
Job = "Is my dataprint done processing?"
```

### 8. Webhooks

**What:** Real-time notifications when things change  
**Think of it as:** "Call me when dataprint is processed"  
**Used for:** Event-driven integrations  

```
Webhook = "Tell me when something happens"
```

---

## The Data Flow

### The Complete Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR PRODUCT                             │
│                                                                 │
│  1. Collect Data (devices, users, whatever you track)          │
│  2. Format as JSON                                             │
│  3. POST to /dataprints                                        │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│                     LIONGARD API                                │
│                                                                 │
│  4. Accept dataprint → Return job ID (202)                     │
│  5. Process asynchronously:                                     │
│     • Validate structure                                        │
│     • Extract environments                                      │
│     • Extract assets                                            │
│     • Detect changes                                            │
│     • Run metrics                                               │
│  6. Complete job → Store results                                │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│                     YOUR PRODUCT (Pull)                         │
│                                                                 │
│  7. Query metrics → Get aggregated data                        │
│  8. List assets → Get inventory                                 │
│  9. Check alerts → Get notifications                            │
│  10. Display in your UI                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Example

**Your Product's Perspective:**

1. **You collect data** from your customers
   ```javascript
   const data = {
     clients: await getClients(),
     devices: await getDevices()
   };
   ```

2. **You push to Liongard**
   ```javascript
   const job = await pushDataprint(envId, inspectorId, data);
   // Returns: { jobId: "job_123", status: "pending" }
   ```

3. **You poll or wait for webhook**
   ```javascript
   // Option A: Poll
   while (true) {
     const status = await getJobStatus(job.jobId);
     if (status === 'completed') break;
   }
   
   // Option B: Webhook (we call you)
   app.post('/webhook', (req, res) => {
     console.log('Dataprint processed!', req.body);
   });
   ```

4. **You query results**
   ```javascript
   // Get asset count
   const metrics = await evaluateMetrics(['metric_device_count']);
   
   // Show in your UI
   dashboard.show(`${metrics[0].value} devices tracked`);
   ```

---

## Inspector Configuration

### Understanding Data Structures

Liongard supports two data patterns:

#### Pattern 1: Split (Recommended)

**When to use:** Your data has separate arrays for environments and assets

```json
{
  "clients": [
    { "id": "c1", "name": "Acme Corp" },
    { "id": "c2", "name": "Beta Inc" }
  ],
  "devices": [
    { "client_id": "c1", "hostname": "srv1", "serial": "SN001" },
    { "client_id": "c1", "hostname": "srv2", "serial": "SN002" },
    { "client_id": "c2", "hostname": "srv3", "serial": "SN003" }
  ]
}
```

**Configuration:**
```json
{
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
```

**How it works:**
1. Liongard reads `$.clients` array → Creates environments
2. Liongard reads `$.devices` array → Creates assets
3. Links assets to environments via `client_id`

#### Pattern 2: Tiered (Nested)

**When to use:** Your data has nested structure (environments contain assets)

```json
{
  "clients": [
    {
      "id": "c1",
      "name": "Acme Corp",
      "devices": [
        { "hostname": "srv1", "serial": "SN001" },
        { "hostname": "srv2", "serial": "SN002" }
      ]
    },
    {
      "id": "c2",
      "name": "Beta Inc",
      "devices": [
        { "hostname": "srv3", "serial": "SN003" }
      ]
    }
  ]
}
```

**Configuration:**
```json
{
  "metadata": {
    "dataStructure": "tiered",
    "environments": {
      "arrayPath": "$.clients",
      "idPath": "$.id",
      "namePath": "$.name"
    },
    "assets": {
      "arrayPath": "$.devices",
      "namePath": "$.hostname",
      "typePath": "$.type",
      "uniqueIdPath": "$.serial"
    }
  }
}
```

**How it works:**
1. Liongard reads `$.clients` array → Creates environments
2. For each client, reads nested `$.devices` array → Creates assets
3. Automatically links assets to parent environment

### JSONPath Primer

JSONPath is used to tell Liongard where to find data in your JSON.

**Basic Examples:**
```
$                    Root of document
$.clients            Array named "clients"
$.clients[0]         First client
$.clients[*]         All clients
$.clients[*].name    Name field of all clients
```

**Your Configuration Uses:**
- `arrayPath` - Where to find the array
- `idPath` - Where to find ID (relative to array item)
- `namePath` - Where to find name (relative to array item)

**Example:**
```json
{
  "customers": [
    { "customer_id": "123", "customer_name": "Acme" }
  ]
}
```

Configuration:
```json
{
  "environments": {
    "arrayPath": "$.customers",
    "idPath": "$.customer_id",
    "namePath": "$.customer_name"
  }
}
```

### Configuration Workflow

```
1. Create Inspector (once)
   POST /v3/inspectors
   
2. For each customer environment:
   PUT /v3/environments/{envId}/inspectors/{inspectorId}/config
   
3. Start pushing dataprints:
   POST /v3/environments/{envId}/inspectors/{inspectorId}/dataprints
```

---

## Pushing Data (Dataprints)

### When to Push

Push a dataprint when:
- ✅ Your data changes (event-driven)
- ✅ On a schedule (e.g., every hour)
- ✅ User triggers sync in your UI
- ✅ After significant events (new device added, etc.)

### Dataprint Structure

Your dataprint should contain:

1. **Environments** (your customers/tenants)
   - Required: ID, Name
   - Optional: Any metadata you want

2. **Assets** (devices, servers, etc.)
   - Required: Name, Type, Unique ID
   - Optional: Any metadata you want

3. **Identities** (optional - users/accounts)
   - Required: Type, Name
   - Optional: Email, department, etc.

### Example: Monitoring Tool Dataprint

```json
{
  "clients": [
    {
      "id": "acme_001",
      "name": "Acme Corporation",
      "tier": "enterprise",
      "contact_email": "admin@acme.com"
    }
  ],
  "devices": [
    {
      "client_id": "acme_001",
      "hostname": "acme-dc01",
      "type": "server",
      "serial_number": "SN-ACME-001",
      "ip_address": "10.0.1.10",
      "os": "Windows Server 2022",
      "status": "online",
      "cpu_usage": 45,
      "memory_usage": 60,
      "disk_usage": 75,
      "last_seen": "2024-02-06T10:30:00Z"
    },
    {
      "client_id": "acme_001",
      "hostname": "acme-ws-01",
      "type": "workstation",
      "serial_number": "SN-ACME-101",
      "ip_address": "10.0.2.101",
      "os": "Windows 11 Pro",
      "status": "online",
      "last_seen": "2024-02-06T10:25:00Z"
    }
  ]
}
```

### Pushing Code Example

```python
import requests
import json

def push_dataprint(env_id, inspector_id, data, api_key):
    """
    Push a dataprint to Liongard
    
    Returns job ID for tracking
    """
    url = f"https://api.liongard.com/v3/environments/{env_id}/inspectors/{inspector_id}/dataprints"
    
    response = requests.post(
        url,
        headers={
            "X-API-Key": api_key,
            "Content-Type": "application/json"
        },
        json=data
    )
    
    if response.status_code == 202:
        result = response.json()
        return result["jobId"]
    else:
        raise Exception(f"Failed to push dataprint: {response.text}")

# Usage
data = {
    "clients": [...],
    "devices": [...]
}

job_id = push_dataprint("env_8888", "inspector_abc123", data, "your-api-key")
print(f"Dataprint submitted: {job_id}")
```

### What Happens After You Push

```
1. API validates JSON structure (instant)
   └─ Returns 202 Accepted + Job ID

2. Background processor starts (async):
   ├─ Validates against your config
   ├─ Extracts environments
   ├─ Extracts assets
   ├─ Links relationships
   ├─ Detects changes from previous dataprint
   ├─ Evaluates metrics
   └─ Triggers alerts/webhooks

3. Job completes (30s - 2min):
   └─ Status changes to "completed"
```

---

## Asynchronous Processing

### Why Async?

Dataprint processing can take 30 seconds to 2 minutes because:
- Parsing large JSON payloads
- Extracting thousands of assets
- Comparing with previous dataprints (change detection)
- Running metrics and alert rules

**Instead of making you wait, we return immediately with a job ID.**

### The Job Lifecycle

```
pending → processing → completed
                    ↘ failed
```

### Tracking Jobs: Three Options

#### Option 1: Polling (Simple)

```python
import time

def wait_for_job(job_id, api_key, timeout=300):
    """Poll until job completes"""
    start = time.time()
    
    while (time.time() - start) < timeout:
        response = requests.get(
            f"https://api.liongard.com/v3/jobs/{job_id}",
            headers={"X-API-Key": api_key}
        )
        
        job = response.json()
        
        if job["status"] == "completed":
            return job["result"]
        elif job["status"] == "failed":
            raise Exception(f"Job failed: {job['error']['message']}")
        
        # Show progress
        progress = job.get("progress", {})
        print(f"Progress: {progress.get('percentage', 0)}% - {progress.get('message', '')}")
        
        time.sleep(5)  # Poll every 5 seconds
    
    raise TimeoutError(f"Job {job_id} timed out")

# Usage
job_id = push_dataprint(...)
result = wait_for_job(job_id, api_key)
print(f"Processed {result['assetsProcessed']} assets")
```

#### Option 2: Callback URL (Recommended)

```python
# When pushing dataprint, provide callback URL
response = requests.post(
    f"https://api.liongard.com/v3/environments/{env_id}/inspectors/{inspector_id}/dataprints?callbackUrl=https://yourapp.com/webhook/dataprint-complete",
    headers={"X-API-Key": api_key},
    json=data
)

# Your webhook endpoint
@app.route("/webhook/dataprint-complete", methods=["POST"])
def handle_dataprint_complete():
    payload = request.json
    job = payload["data"]
    
    if job["status"] == "completed":
        print(f"Success! Processed {job['result']['assetsProcessed']} assets")
    else:
        print(f"Failed: {job['error']['message']}")
    
    return "", 200  # Return 2xx quickly
```

#### Option 3: Webhook Subscription (Continuous)

```python
# Subscribe to all dataprint events (one-time setup)
requests.post(
    "https://api.liongard.com/v3/webhooks/subscriptions",
    headers={"X-API-Key": api_key},
    json={
        "url": "https://yourapp.com/webhook/liongard",
        "events": ["dataprint.created", "dataprint.processing", "dataprint.completed"],
        "deliveryMode": "full"
    }
)

# Your webhook receives ALL dataprint events
@app.route("/webhook/liongard", methods=["POST"])
def handle_webhook():
    event = request.json
    
    if event["type"] == "dataprint.completed":
        dataprint = event["data"]["object"]
        print(f"Dataprint {dataprint['dataprintId']} completed")
    
    return "", 200
```

### Handling Concurrency

**Important Rule:** Only ONE dataprint job can process at a time per inspector per environment.

**Why?** Prevents data conflicts and ensures change detection works correctly.

**Check Before Pushing:**

```python
def can_push_dataprint(env_id, inspector_id, api_key):
    """Check if we can push (no active job)"""
    response = requests.get(
        "https://api.liongard.com/v3/jobs",
        headers={"X-API-Key": api_key},
        params={
            "filter": f"environmentId=={env_id};inspectorId=={inspector_id};status=in=(pending,processing)",
            "limit": 1
        }
    )
    
    jobs = response.json()
    return len(jobs) == 0

# Usage
if can_push_dataprint(env_id, inspector_id, api_key):
    job_id = push_dataprint(...)
else:
    print("Job already running, waiting...")
    # Wait or cancel existing job
```

**Cancel Existing Job:**

```python
# Cancel the active job
requests.delete(
    f"https://api.liongard.com/v3/jobs/{existing_job_id}",
    headers={"X-API-Key": api_key}
)

# Now push new dataprint
job_id = push_dataprint(...)
```

### 409 Conflict Response

If you try to push while a job is running:

```json
{
  "error": {
    "code": "CONCURRENT_JOB_CONFLICT",
    "message": "A dataprint job is already processing for inspector 'inspector_abc123' in environment 'env_8888'",
    "details": {
      "existingJobId": "job_xyz789",
      "existingJobStatus": "processing",
      "suggestion": "Wait for the existing job to complete or cancel it using DELETE /v3/jobs/job_xyz789"
    }
  }
}
```

---

## Extracting Data with Metrics

### What Are Metrics?

Metrics let you query specific data from your dataprints using **JMESPath** expressions.

**Think of it as:**
- SQL for JSON
- A custom report query
- A calculated field

### JMESPath Basics

**Simple field access:**
```
Query: name
Data:  {"name": "Server-01"}
Result: "Server-01"
```

**Array length:**
```
Query: length(devices)
Data:  {"devices": [1, 2, 3, 4, 5]}
Result: 5
```

**Filter and count:**
```
Query: length(devices[?status=='offline'])
Data:  {"devices": [
         {"name": "srv1", "status": "online"},
         {"name": "srv2", "status": "offline"},
         {"name": "srv3", "status": "offline"}
       ]}
Result: 2
```

**Extract field from array:**
```
Query: devices[*].name
Data:  {"devices": [{"name": "srv1"}, {"name": "srv2"}]}
Result: ["srv1", "srv2"]
```

### Creating a Metric

```python
# Create metric: "Count of offline devices"
response = requests.post(
    "https://api.liongard.com/v3/metrics",
    headers={"X-API-Key": api_key},
    json={
        "name": "Device Count - Offline",
        "description": "Number of devices with status=offline",
        "inspectorId": "inspector_abc123",
        "queries": [
            {
                "query": "length(devices[?status=='offline'])",
                "inspectorVersionId": "version_1"
            }
        ],
        "metricDisplay": True,
        "changesEnabled": True
    }
)

metric_id = response.json()["metricId"]
```

### Evaluating Metrics

**Get values across all systems:**

```python
response = requests.post(
    "https://api.liongard.com/v3/metrics/evaluate",
    headers={"X-API-Key": api_key},
    json={
        "metrics": ["metric_123", "metric_456"],
        "pagination": {
            "limit": 100,
            "offset": 0
        }
    }
)

results = response.json()

for result in results:
    print(f"{result['environmentName']}: {result['metricName']} = {result['value']}")

# Output:
# Acme Corp: Device Count - Offline = 2
# Beta Inc: Device Count - Offline = 0
```

**Get all metrics for a specific system:**

```python
response = requests.get(
    f"https://api.liongard.com/v3/environments/{env_id}/systems/{system_id}/metrics",
    headers={"X-API-Key": api_key}
)

metrics = response.json()["metrics"]

for metric in metrics:
    print(f"{metric['metricName']}: {metric['value']}")
```

### Common Metric Patterns

#### Count Metrics
```python
# Total devices
"length(devices)"

# Devices by type
"length(devices[?type=='server'])"

# Devices by status
"length(devices[?status=='offline'])"
```

#### Percentage Metrics
```python
# Percentage offline
"length(devices[?status=='offline']) / length(devices) * `100`"
```

#### List Metrics
```python
# All offline device names
"devices[?status=='offline'].name"

# Result: ["srv-01", "srv-03", "srv-07"]
```

#### Complex Metrics
```python
# Devices with high CPU and low disk
"devices[?cpu_usage>`80` && disk_free<`10`]"

# Count by multiple conditions
"length(devices[?type=='server' && status=='online' && os==`Windows*`])"
```

### Using Metrics in Your Product

**Example: Dashboard Widget**

```python
def get_dashboard_stats(api_key):
    """Get aggregated stats for dashboard"""
    
    response = requests.post(
        "https://api.liongard.com/v3/metrics/evaluate",
        headers={"X-API-Key": api_key},
        json={
            "metrics": [
                "metric_total_devices",
                "metric_offline_devices",
                "metric_critical_alerts"
            ],
            "pagination": {"limit": 1000, "offset": 0}
        }
    )
    
    results = response.json()
    
    # Aggregate across all customers
    stats = {
        "total_devices": 0,
        "offline_devices": 0,
        "critical_alerts": 0
    }
    
    for result in results:
        if "total_devices" in result["metricName"]:
            stats["total_devices"] += result["value"]
        elif "offline" in result["metricName"]:
            stats["offline_devices"] += result["value"]
        elif "critical" in result["metricName"]:
            stats["critical_alerts"] += result["value"]
    
    return stats

# Display in UI
stats = get_dashboard_stats(api_key)
dashboard.show(f"""
    Total Devices: {stats['total_devices']}
    Offline: {stats['offline_devices']}
    Critical Alerts: {stats['critical_alerts']}
""")
```

---

## Real-Time Updates with Webhooks

### What Are Webhooks?

Webhooks let Liongard notify YOUR application when events happen.

**Instead of:**
```
Your App → (polling) → "Are you done yet?" → Liongard
Your App → (polling) → "Are you done yet?" → Liongard
Your App → (polling) → "Are you done yet?" → Liongard
```

**You get:**
```
Liongard → (event occurs) → POST https://yourapp.com/webhook → Your App
```

### Subscribe to Events

```python
response = requests.post(
    "https://api.liongard.com/v3/webhooks/subscriptions",
    headers={"X-API-Key": api_key},
    json={
        "url": "https://yourapp.com/webhooks/liongard",
        "events": [
            "dataprint.created",
            "dataprint.completed",
            "alert.created",
            "asset.created",
            "asset.updated"
        ],
        "deliveryMode": "full",
        "enabled": True
    }
)

subscription_id = response.json()["subscriptionId"]
```

### Handle Webhook Deliveries

```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)

@app.route("/webhooks/liongard", methods=["POST"])
def handle_webhook():
    # 1. Verify signature (security)
    signature = request.headers.get("X-Liongard-Signature")
    timestamp = request.headers.get("X-Liongard-Timestamp")
    payload = request.get_data()
    
    if not verify_signature(payload, signature, timestamp, webhook_secret):
        return "Invalid signature", 401
    
    # 2. Get event data
    event = request.json
    event_type = event["type"]
    
    # 3. Handle different event types
    if event_type == "dataprint.completed":
        handle_dataprint_completed(event)
    elif event_type == "alert.created":
        handle_alert_created(event)
    elif event_type == "asset.updated":
        handle_asset_updated(event)
    
    # 4. Return 2xx quickly (< 10 seconds)
    return "", 200

def verify_signature(payload, signature, timestamp, secret):
    """Verify webhook came from Liongard"""
    signed_payload = f"{timestamp}.{payload.decode()}"
    expected = hmac.new(
        secret.encode(),
        signed_payload.encode(),
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(f"sha256={expected}", signature)

def handle_dataprint_completed(event):
    """Process completed dataprint"""
    dataprint = event["data"]["object"]
    env_id = dataprint["environmentId"]
    
    print(f"Dataprint completed for {env_id}")
    
    # Update your database
    db.update_last_sync(env_id, dataprint["createdAt"])
    
    # Trigger downstream processes
    analytics.process_environment(env_id)

def handle_alert_created(event):
    """Process new alert"""
    alert = event["data"]["object"]
    
    if alert["priority"] == "critical":
        # Create ticket in your system
        ticket_system.create_ticket(
            title=alert["name"],
            description=alert["message"],
            priority="high"
        )

def handle_asset_updated(event):
    """Process asset change"""
    changes = event["data"]["changes"]
    asset = event["data"]["object"]
    
    # Notify if status changed
    if "status" in changes:
        old_status = changes["status"]["old"]
        new_status = changes["status"]["new"]
        
        if old_status == "online" and new_status == "offline":
            notifications.send(
                f"Device {asset['name']} went offline!"
            )
```

### Webhook Event Types

**Dataprint Events:**
- `dataprint.created` - New dataprint submitted
- `dataprint.processing` - Started processing
- `dataprint.completed` - Finished processing
- `dataprint.failed` - Processing failed

**Alert Events:**
- `alert.created` - New alert generated
- `alert.updated` - Alert status/priority changed
- `alert.resolved` - Alert marked resolved

**Asset Events:**
- `asset.created` - New asset discovered
- `asset.updated` - Asset properties changed
- `asset.deleted` - Asset removed

**Identity Events:**
- `identity.created` - New user/account found
- `identity.updated` - Identity properties changed
- `identity.deleted` - Identity removed

### Webhook Payload Example

```json
{
  "eventId": "evt_abc123",
  "type": "dataprint.completed",
  "createdAt": "2024-02-06T10:30:15Z",
  "data": {
    "object": {
      "dataprintId": "dataprint_xyz789",
      "environmentId": "env_8888",
      "inspectorId": "inspector_abc123",
      "createdAt": "2024-02-06T10:30:00Z",
      "status": "completed",
      "result": {
        "accepted": true,
        "environmentsProcessed": 1,
        "assetsProcessed": 15,
        "warnings": [],
        "errors": []
      }
    }
  }
}
```

---

## Querying Resources

### Filtering with RSQL

All list endpoints support filtering using RSQL syntax.

**Basic Examples:**

```bash
# Equal
GET /v3/assets?filter=assetType==server

# Not equal
GET /v3/assets?filter=status!=offline

# Greater than
GET /v3/assets?filter=diskUsage>80

# In list
GET /v3/assets?filter=assetType=in=(server,workstation)

# AND (semicolon)
GET /v3/assets?filter=assetType==server;status==online

# OR (comma)
GET /v3/assets?filter=status==offline,status==error

# Wildcard
GET /v3/assets?filter=name==*server*

# Complex
GET /v3/assets?filter=assetType==server;(status==offline,status==error);location==*datacenter*
```

**Python Helper:**

```python
def build_filter(**conditions):
    """Build RSQL filter string"""
    parts = []
    for field, value in conditions.items():
        if isinstance(value, list):
            values = ",".join(value)
            parts.append(f"{field}=in=({values})")
        else:
            parts.append(f"{field}=={value}")
    return ";".join(parts)

# Usage
filter_str = build_filter(
    assetType="server",
    status=["online", "warning"],
    location="datacenter"
)
# Result: "assetType==server;status=in=(online,warning);location==datacenter"

response = requests.get(
    "https://api.liongard.com/v3/assets",
    headers={"X-API-Key": api_key},
    params={"filter": filter_str}
)
```

### Pagination

**List endpoints return arrays directly with pagination in headers:**

```bash
GET /v3/assets?limit=100&offset=0
```

**Response Body:**
```json
[
  {
    "assetId": "asset_1",
    "name": "server-01",
    "assetType": "server"
  },
  {
    "assetId": "asset_2", 
    "name": "server-02",
    "assetType": "server"
  }
]
```

**Response Headers:**
```
X-Pagination-Limit: 100
X-Pagination-Offset: 0
X-Pagination-Count: 547
X-Pagination-Has-More: true
Link: <https://api.liongard.com/v3/assets?limit=100&offset=100>; rel="next",
      <https://api.liongard.com/v3/assets?limit=100&offset=0>; rel="first",
      <https://api.liongard.com/v3/assets?limit=100&offset=500>; rel="last"
```

**Python Helper:**

```python
def get_all_assets(api_key, filters=None):
    """Get all assets with pagination"""
    all_assets = []
    offset = 0
    limit = 100
    
    while True:
        params = {"limit": limit, "offset": offset}
        if filters:
            params["filter"] = filters
        
        response = requests.get(
            "https://api.liongard.com/v3/assets",
            headers={"X-API-Key": api_key},
            params=params
        )
        
        # Response is array directly
        assets = response.json()
        all_assets.extend(assets)
        
        # Check pagination header
        has_more = response.headers.get('X-Pagination-Has-More') == 'true'
        if not has_more:
            break
        
        offset += limit
    
    return all_assets
```

**Using Link Header:**

```python
import requests
from requests.utils import parse_header_links

def get_next_page(response):
    """Get next page URL from Link header"""
    link_header = response.headers.get('Link')
    if not link_header:
        return None
    
    links = parse_header_links(link_header)
    for link in links:
        if link.get('rel') == 'next':
            return link['url']
    
    return None

# Usage
response = requests.get(url, headers=headers)
assets = response.json()

next_url = get_next_page(response)
if next_url:
    response = requests.get(next_url, headers=headers)
    more_assets = response.json()
```

### Sorting

```python
# Sort by name ascending
GET /v3/assets?sort=name

# Sort by createdAt descending (most recent first)
GET /v3/assets?sort=-createdAt

# Multiple sorts
GET /v3/assets?sort=-priority,name
```

### Field Expansion

```python
# Get alerts with environment and inspector details embedded
response = requests.get(
    "https://api.liongard.com/v3/alerts",
    headers={"X-API-Key": api_key},
    params={
        "filter": "status==open",
        "expand": "environment,inspector"
    }
)

for alert in response.json():
    print(f"{alert['name']} in {alert['environment']['name']}")
```

---

## Complete Integration Example

### Scenario: Monitoring Tool Integration

**Your Product:** Network monitoring tool  
**Goal:** Sync device data to Liongard every hour  
**Features:** Push data, track jobs, display metrics  

### Step 1: Initialize (One-Time Setup)

```python
import requests
import time
from datetime import datetime

class LiongardIntegration:
    def __init__(self, api_key, base_url="https://api.liongard.com"):
        self.api_key = api_key
        self.base_url = base_url
        self.headers = {
            "X-API-Key": api_key,
            "Content-Type": "application/json"
        }
    
    def create_inspector(self):
        """Create inspector definition (once)"""
        response = requests.post(
            f"{self.base_url}/v3/inspectors",
            headers=self.headers,
            json={
                "name": "acme-monitor",
                "displayName": "Acme Network Monitor",
                "description": "Device monitoring and health data",
                "category": "monitoring"
            }
        )
        
        return response.json()["inspectorId"]
    
    def configure_inspector(self, env_id, inspector_id):
        """Configure inspector for environment (per customer)"""
        response = requests.put(
            f"{self.base_url}/v3/environments/{env_id}/inspectors/{inspector_id}/config",
            headers=self.headers,
            json={
                "metadata": {
                    "dataStructure": "split",
                    "environments": {
                        "arrayPath": "$.clients",
                        "idPath": "$.client_id",
                        "namePath": "$.client_name"
                    },
                    "assets": {
                        "arrayPath": "$.devices",
                        "namePath": "$.hostname",
                        "typePath": "$.device_type",
                        "uniqueIdPath": "$.serial_number",
                        "environmentIdPath": "$.client_id"
                    }
                }
            }
        )
        
        return response.json()
    
    def create_metrics(self, inspector_id):
        """Create custom metrics (once)"""
        metrics = [
            {
                "name": "Total Devices",
                "query": "length(devices)"
            },
            {
                "name": "Offline Devices",
                "query": "length(devices[?status=='offline'])"
            },
            {
                "name": "High CPU Devices",
                "query": "length(devices[?cpu_usage>`80`])"
            },
            {
                "name": "Critical Devices",
                "query": "devices[?status=='offline' || cpu_usage>`90`].hostname"
            }
        ]
        
        metric_ids = []
        
        for metric in metrics:
            response = requests.post(
                f"{self.base_url}/v3/metrics",
                headers=self.headers,
                json={
                    "name": metric["name"],
                    "inspectorId": inspector_id,
                    "queries": [{
                        "query": metric["query"],
                        "inspectorVersionId": "version_1"
                    }],
                    "metricDisplay": True,
                    "changesEnabled": True
                }
            )
            metric_ids.append(response.json()["metricId"])
        
        return metric_ids

# One-time setup
integration = LiongardIntegration("your-api-key")
inspector_id = integration.create_inspector()
metric_ids = integration.create_metrics(inspector_id)

print(f"Inspector: {inspector_id}")
print(f"Metrics: {metric_ids}")
```

### Step 2: Regular Data Sync

```python
class MonitoringSync:
    def __init__(self, integration, inspector_id):
        self.integration = integration
        self.inspector_id = inspector_id
    
    def collect_data_from_monitoring_tool(self):
        """Get data from your monitoring system"""
        # This is YOUR code to get data from your product
        clients = []
        devices = []
        
        for customer in your_monitoring_tool.get_customers():
            clients.append({
                "client_id": customer.id,
                "client_name": customer.name
            })
            
            for device in customer.get_devices():
                devices.append({
                    "client_id": customer.id,
                    "hostname": device.hostname,
                    "device_type": device.type,
                    "serial_number": device.serial,
                    "ip_address": device.ip,
                    "status": device.status,
                    "cpu_usage": device.cpu_percent,
                    "memory_usage": device.memory_percent,
                    "disk_usage": device.disk_percent,
                    "last_seen": device.last_contact.isoformat()
                })
        
        return {
            "clients": clients,
            "devices": devices
        }
    
    def push_dataprint(self, env_id, data):
        """Push dataprint and track job"""
        response = requests.post(
            f"{self.integration.base_url}/v3/environments/{env_id}/inspectors/{self.inspector_id}/dataprints",
            headers=self.integration.headers,
            json=data
        )
        
        if response.status_code == 202:
            return response.json()["jobId"]
        elif response.status_code == 409:
            # Concurrent job - cancel and retry
            error = response.json()["error"]
            existing_job = error["details"]["existingJobId"]
            
            print(f"Cancelling existing job: {existing_job}")
            requests.delete(
                f"{self.integration.base_url}/v3/jobs/{existing_job}",
                headers=self.integration.headers
            )
            
            time.sleep(2)
            return self.push_dataprint(env_id, data)
        else:
            raise Exception(f"Failed: {response.text}")
    
    def wait_for_job(self, job_id, timeout=300):
        """Wait for job to complete"""
        start = time.time()
        
        while (time.time() - start) < timeout:
            response = requests.get(
                f"{self.integration.base_url}/v3/jobs/{job_id}",
                headers=self.integration.headers
            )
            
            job = response.json()
            status = job["status"]
            
            if status == "completed":
                return job["result"]
            elif status == "failed":
                raise Exception(f"Job failed: {job['error']['message']}")
            
            progress = job.get("progress", {})
            print(f"  Progress: {progress.get('percentage', 0)}%")
            
            time.sleep(5)
        
        raise TimeoutError(f"Job {job_id} timed out")
    
    def sync_customer(self, env_id):
        """Complete sync for one customer"""
        print(f"Syncing customer {env_id}...")
        
        # 1. Collect data
        print("  Collecting data...")
        data = self.collect_data_from_monitoring_tool()
        print(f"  Found {len(data['clients'])} clients, {len(data['devices'])} devices")
        
        # 2. Push to Liongard
        print("  Pushing to Liongard...")
        job_id = self.push_dataprint(env_id, data)
        print(f"  Job started: {job_id}")
        
        # 3. Wait for completion
        print("  Waiting for processing...")
        result = self.wait_for_job(job_id)
        print(f"  ✓ Complete: {result['assetsProcessed']} assets processed")
        
        return result
    
    def sync_all_customers(self):
        """Sync all customers"""
        customers = your_monitoring_tool.get_customers()
        
        for customer in customers:
            try:
                result = self.sync_customer(customer.liongard_env_id)
            except Exception as e:
                print(f"  ✗ Failed: {e}")

# Run sync
sync = MonitoringSync(integration, inspector_id)
sync.sync_all_customers()
```

### Step 3: Display Metrics in Your UI

```python
class Dashboard:
    def __init__(self, integration, metric_ids):
        self.integration = integration
        self.metric_ids = metric_ids
    
    def get_aggregate_stats(self):
        """Get stats across all customers"""
        response = requests.post(
            f"{self.integration.base_url}/v3/metrics/evaluate",
            headers=self.integration.headers,
            json={
                "metrics": self.metric_ids,
                "pagination": {"limit": 1000, "offset": 0}
            }
        )
        
        results = response.json()
        
        # Aggregate
        stats = {
            "total_devices": 0,
            "offline_devices": 0,
            "high_cpu_devices": 0,
            "critical_devices": []
        }
        
        for result in results:
            name = result["metricName"]
            value = result["value"]
            
            if "Total Devices" in name:
                stats["total_devices"] += value
            elif "Offline" in name:
                stats["offline_devices"] += value
            elif "High CPU" in name:
                stats["high_cpu_devices"] += value
            elif "Critical" in name:
                stats["critical_devices"].extend(value)
        
        return stats
    
    def get_customer_details(self, env_id):
        """Get detailed stats for one customer"""
        response = requests.post(
            f"{self.integration.base_url}/v3/metrics/evaluate",
            headers=self.integration.headers,
            json={
                "metrics": self.metric_ids,
                "filters": [{
                    "field": "environmentId",
                    "op": "equal_to",
                    "value": env_id
                }],
                "pagination": {"limit": 100, "offset": 0}
            }
        )
        
        results = response.json()
        
        details = {}
        for result in results:
            details[result["metricName"]] = result["value"]
        
        return details
    
    def display(self):
        """Render dashboard"""
        stats = self.get_aggregate_stats()
        
        print("=" * 50)
        print("MONITORING DASHBOARD")
        print("=" * 50)
        print(f"Total Devices:     {stats['total_devices']}")
        print(f"Offline:           {stats['offline_devices']}")
        print(f"High CPU:          {stats['high_cpu_devices']}")
        print(f"Critical Devices:  {len(stats['critical_devices'])}")
        
        if stats['critical_devices']:
            print("\nCritical Devices:")
            for device in stats['critical_devices'][:10]:
                print(f"  - {device}")

# Show dashboard
dashboard = Dashboard(integration, metric_ids)
dashboard.display()
```

### Step 4: Handle Webhooks

```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)

# Subscribe to webhooks (one-time)
requests.post(
    f"{integration.base_url}/v3/webhooks/subscriptions",
    headers=integration.headers,
    json={
        "url": "https://yourapp.com/webhook/liongard",
        "events": [
            "dataprint.completed",
            "alert.created",
            "asset.updated"
        ],
        "deliveryMode": "full"
    }
)

@app.route("/webhook/liongard", methods=["POST"])
def handle_webhook():
    # Verify signature
    signature = request.headers.get("X-Liongard-Signature")
    timestamp = request.headers.get("X-Liongard-Timestamp")
    payload = request.get_data()
    
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        f"{timestamp}.{payload.decode()}".encode(),
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(f"sha256={expected}", signature):
        return "Invalid signature", 401
    
    event = request.json
    
    if event["type"] == "dataprint.completed":
        # Update sync status in your DB
        env_id = event["data"]["object"]["environmentId"]
        your_db.update_last_sync(env_id, datetime.now())
        
        # Refresh customer dashboard
        dashboard_cache.invalidate(env_id)
    
    elif event["type"] == "alert.created":
        # Create ticket
        alert = event["data"]["object"]
        if alert["priority"] == "critical":
            ticket_system.create(
                title=alert["name"],
                customer=alert["environmentId"],
                priority="high"
            )
    
    elif event["type"] == "asset.updated":
        # Check for status changes
        changes = event["data"]["changes"]
        if "status" in changes:
            asset = event["data"]["object"]
            if changes["status"]["new"] == "offline":
                notifications.send(
                    f"Device {asset['name']} went offline!",
                    customer=asset["environmentId"]
                )
    
    return "", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### Step 5: Scheduled Sync

```python
from apscheduler.schedulers.background import BackgroundScheduler

def sync_job():
    """Hourly sync job"""
    print(f"[{datetime.now()}] Starting sync...")
    
    try:
        sync = MonitoringSync(integration, inspector_id)
        sync.sync_all_customers()
        print(f"[{datetime.now()}] Sync complete")
    except Exception as e:
        print(f"[{datetime.now()}] Sync failed: {e}")

# Schedule every hour
scheduler = BackgroundScheduler()
scheduler.add_job(sync_job, 'interval', hours=1)
scheduler.start()

print("Scheduler started. Syncing every hour...")
```

---

## Production Deployment

### Configuration Management

```python
import os
from dataclasses import dataclass

@dataclass
class LiongardConfig:
    api_key: str
    base_url: str
    inspector_id: str
    metric_ids: list
    webhook_secret: str
    
    @classmethod
    def from_env(cls):
        return cls(
            api_key=os.getenv("LIONGARD_API_KEY"),
            base_url=os.getenv("LIONGARD_BASE_URL", "https://api.liongard.com"),
            inspector_id=os.getenv("LIONGARD_INSPECTOR_ID"),
            metric_ids=os.getenv("LIONGARD_METRIC_IDS", "").split(","),
            webhook_secret=os.getenv("LIONGARD_WEBHOOK_SECRET")
        )

config = LiongardConfig.from_env()
```

### Error Handling

```python
import logging
from requests.exceptions import RequestException, Timeout

logger = logging.getLogger(__name__)

class LiongardClient:
    def __init__(self, config):
        self.config = config
    
    def push_dataprint(self, env_id, data, retries=3):
        """Push with retries"""
        for attempt in range(retries):
            try:
                response = requests.post(
                    f"{self.config.base_url}/v3/environments/{env_id}/inspectors/{self.config.inspector_id}/dataprints",
                    headers={"X-API-Key": self.config.api_key},
                    json=data,
                    timeout=30
                )
                
                if response.status_code == 202:
                    return response.json()["jobId"]
                elif response.status_code == 409:
                    # Handle concurrency conflict
                    return self._handle_conflict(response, env_id, data)
                elif response.status_code == 429:
                    # Rate limited - exponential backoff
                    wait = int(response.headers.get("Retry-After", 2 ** attempt))
                    logger.warning(f"Rate limited, waiting {wait}s")
                    time.sleep(wait)
                    continue
                else:
                    response.raise_for_status()
            
            except Timeout:
                logger.error(f"Timeout on attempt {attempt + 1}/{retries}")
                if attempt == retries - 1:
                    raise
                time.sleep(2 ** attempt)
            
            except RequestException as e:
                logger.error(f"Request failed: {e}")
                if attempt == retries - 1:
                    raise
                time.sleep(2 ** attempt)
    
    def _handle_conflict(self, response, env_id, data):
        """Handle concurrent job conflict"""
        error = response.json()["error"]
        existing_job = error["details"]["existingJobId"]
        
        logger.info(f"Cancelling existing job: {existing_job}")
        
        # Cancel and retry
        requests.delete(
            f"{self.config.base_url}/v3/jobs/{existing_job}",
            headers={"X-API-Key": self.config.api_key}
        )
        
        time.sleep(2)
        return self.push_dataprint(env_id, data, retries=1)
```

### Monitoring & Logging

```python
import logging
from prometheus_client import Counter, Histogram, Gauge

# Metrics
dataprints_pushed = Counter('liongard_dataprints_pushed_total', 'Total dataprints pushed')
dataprints_failed = Counter('liongard_dataprints_failed_total', 'Failed dataprints')
dataprint_duration = Histogram('liongard_dataprint_duration_seconds', 'Dataprint processing time')
active_jobs = Gauge('liongard_active_jobs', 'Number of active jobs')

# Logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

class InstrumentedSync:
    def __init__(self, client):
        self.client = client
        self.logger = logging.getLogger(__name__)
    
    def sync_customer(self, env_id):
        """Sync with instrumentation"""
        self.logger.info(f"Starting sync for {env_id}")
        
        try:
            with dataprint_duration.time():
                # Collect data
                data = self.collect_data()
                
                # Push
                job_id = self.client.push_dataprint(env_id, data)
                dataprints_pushed.inc()
                active_jobs.inc()
                
                # Wait
                result = self.wait_for_job(job_id)
                active_jobs.dec()
                
                self.logger.info(f"Sync complete: {result['assetsProcessed']} assets")
                return result
        
        except Exception as e:
            dataprints_failed.inc()
            self.logger.error(f"Sync failed for {env_id}: {e}", exc_info=True)
            raise
```

### Health Checks

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/health", methods=["GET"])
def health_check():
    """Kubernetes health check"""
    checks = {
        "liongard_api": check_liongard_api(),
        "database": check_database(),
        "monitoring_tool": check_monitoring_tool()
    }
    
    healthy = all(checks.values())
    status_code = 200 if healthy else 503
    
    return jsonify({
        "status": "healthy" if healthy else "unhealthy",
        "checks": checks
    }), status_code

def check_liongard_api():
    """Check if Liongard API is reachable"""
    try:
        response = requests.get(
            f"{config.base_url}/v3/meta/objects",
            headers={"X-API-Key": config.api_key},
            timeout=5
        )
        return response.status_code == 200
    except:
        return False
```

---

## Troubleshooting

### Common Issues

#### Issue 1: 409 Conflict - Job Already Running

**Problem:**
```json
{
  "error": {
    "code": "CONCURRENT_JOB_CONFLICT",
    "message": "A dataprint job is already processing..."
  }
}
```

**Solution:**
```python
# Option 1: Wait for job to complete
def wait_for_slot(env_id, inspector_id, timeout=300):
    start = time.time()
    while (time.time() - start) < timeout:
        response = requests.get(
            f"{base_url}/v3/jobs",
            headers=headers,
            params={
                "filter": f"environmentId=={env_id};inspectorId=={inspector_id};status=in=(pending,processing)"
            }
        )
        if not response.json():
            return True  # Slot available
        time.sleep(10)
    return False

# Option 2: Cancel existing job
existing_job_id = error["details"]["existingJobId"]
requests.delete(f"{base_url}/v3/jobs/{existing_job_id}", headers=headers)
time.sleep(2)
# Retry push
```

#### Issue 2: Job Failed - Validation Error

**Problem:**
```json
{
  "jobId": "job_123",
  "status": "failed",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid JSONPath in assets.uniqueIdPath"
  }
}
```

**Solution:**
1. Check your config JSONPath expressions
2. Test JSONPath against sample data
3. Ensure all required fields are present

```python
import jsonpath_ng

# Test your JSONPath
expr = jsonpath_ng.parse("$.devices[*].serial_number")
matches = expr.find(your_data)
print(f"Found {len(matches)} matches")
```

#### Issue 3: Empty Results from Metrics

**Problem:** Metric returns no value or null

**Solution:**
1. Check if dataprint was processed successfully
2. Verify JMESPath query syntax
3. Test query against actual dataprint data

```bash
# Test JMESPath online
https://jmespath.org/

# Your data
{
  "devices": [
    {"status": "online"},
    {"status": "offline"}
  ]
}

# Query
length(devices[?status=='offline'])

# Result: 1
```

#### Issue 4: Webhook Not Receiving Events

**Problem:** Webhook endpoint not being called

**Solution:**
1. Check subscription is active
   ```python
   response = requests.get(
       f"{base_url}/v3/webhooks/subscriptions",
       headers=headers
   )
   ```

2. Verify webhook URL is publicly accessible
   ```bash
   curl -X POST https://yourapp.com/webhook/test
   ```

3. Check webhook endpoint returns 2xx within 10 seconds

4. Review webhook delivery logs (if available)

5. Test signature verification

#### Issue 5: Rate Limiting (429)

**Problem:**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests"
  }
}
```

**Solution:**
```python
def make_request_with_backoff(url, **kwargs):
    """Request with exponential backoff"""
    max_retries = 5
    
    for attempt in range(max_retries):
        response = requests.request(**kwargs, url=url)
        
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 2 ** attempt))
            print(f"Rate limited, waiting {retry_after}s")
            time.sleep(retry_after)
            continue
        
        return response
    
    raise Exception("Max retries exceeded")
```

### Debugging Tips

**1. Enable Detailed Logging**
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

**2. Inspect Full Responses**
```python
response = requests.post(...)
print(f"Status: {response.status_code}")
print(f"Headers: {response.headers}")
print(f"Body: {response.text}")
```

**3. Test Individual Components**
```python
# Test 1: Can you reach API?
response = requests.get(f"{base_url}/v3/meta/objects", headers=headers)
assert response.status_code == 200

# Test 2: Is your config valid?
response = requests.get(
    f"{base_url}/v3/environments/{env_id}/inspectors/{inspector_id}/config",
    headers=headers
)
config = response.json()
print(json.dumps(config, indent=2))

# Test 3: Can you push minimal dataprint?
minimal_data = {
    "clients": [{"id": "test", "name": "Test"}],
    "devices": [{"client_id": "test", "hostname": "test", "serial": "123"}]
}
response = requests.post(...)
```

**4. Validate Your Data**
```python
def validate_dataprint(data, config):
    """Validate dataprint matches config"""
    import jsonpath_ng
    
    # Check environments
    env_path = jsonpath_ng.parse(config["metadata"]["environments"]["arrayPath"])
    envs = env_path.find(data)
    print(f"Found {len(envs)} environments")
    
    # Check assets
    asset_path = jsonpath_ng.parse(config["metadata"]["assets"]["arrayPath"])
    assets = asset_path.find(data)
    print(f"Found {len(assets)} assets")
    
    # Check required fields
    for asset in assets[:5]:  # Sample first 5
        print(f"Asset: {asset.value}")
```

---

## Summary

### What You Learned

✅ **The Vendor API is brand new** - Built specifically for external integrations  
✅ **Core workflow:** Create Inspector → Configure → Push Dataprints → Extract Metrics  
✅ **Async processing:** All dataprints are processed in background with job tracking  
✅ **Two-way bridge:** Push data in, pull insights out  
✅ **Real-time updates:** Webhooks notify you of events  
✅ **Flexible queries:** RSQL for filtering, JMESPath for metrics  

### Key Endpoints

```
POST /v3/inspectors                                              # Create inspector
PUT  /v3/environments/{envId}/inspectors/{id}/config             # Configure
POST /v3/environments/{envId}/inspectors/{id}/dataprints         # Push data
GET  /v3/jobs/{jobId}                                            # Check status
POST /v3/metrics/evaluate                                        # Get metrics
POST /v3/webhooks/subscriptions                                  # Subscribe to events
```

### Next Steps

1. **Get API key** from Liongard
2. **Create your inspector** definition
3. **Configure** for test environment
4. **Push test dataprint** and verify processing
5. **Create metrics** for your use cases
6. **Set up webhooks** for real-time updates
7. **Build UI** to display metrics
8. **Deploy to production**

### Resources

- **RSQL Filter Guide** - Complete filtering reference
- **RSQL Migration Guide** - If coming from other syntax
- **API Complete Summary** - Full endpoint reference
- **Jobs & Async Processing** - Deep dive on jobs

### Support

Questions? Issues? Contact Liongard support with:
- API key (last 4 characters only)
- Environment ID
- Inspector ID  
- Job ID (if applicable)
- Example request/response

---

**You're ready to build your Liongard integration!** 🚀
