---
title: Jobs & Async Processing
parent: Technical References
nav_order: 4
---

# Jobs & Asynchronous Processing

## Overview
The Liongard Data API v3 implements asynchronous processing for long-running operations like dataprint dataprint processing. This enables immediate API responses while processing continues in the background.

---

## Why Asynchronous Processing?

Dataprint processing involves multiple steps:
1. **Validation**: Check data against config schema
2. **Parsing**: Extract environments and assets using JSONPath
3. **Normalization**: Transform data into standard format
4. **Storage**: Persist dataprint and update asset inventory
5. **Change Detection**: Compare with previous dataprints
6. **Alert Evaluation**: Run metric evaluations and alert rules

This can take **30 seconds to several minutes** depending on data size and complexity.

---

## How It Works

### 1. Submit Data → Get Job ID
```bash
POST /v3/environments/env_8888/inspectors/inspector_13/dataprints
Content-Type: application/json

{
  "clients": [...],
  "devices": [...]
}
```

**Response: 202 Accepted**
```json
{
  "jobId": "job_abc123xyz",
  "status": "pending",
  "message": "Dataprint accepted for processing",
  "estimatedCompletionSeconds": 120,
  "pollUrl": "https://api.liongard.example.com/v3/jobs/job_abc123xyz"
}
```

### 2. Poll for Status
```bash
GET /v3/jobs/job_abc123xyz
```

**Response: 200 OK (Processing)**
```json
{
  "data": {
    "jobId": "job_abc123xyz",
    "type": "dataprint_processing",
    "status": "processing",
    "progress": {
      "current": 2,
      "total": 5,
      "percentage": 40,
      "message": "Processing assets"
    },
    "resourceType": "dataprint",
    "environmentId": "env_8888",
    "inspectorId": "inspector_13",
    "createdAt": "2024-02-06T10:30:00Z",
    "updatedAt": "2024-02-06T10:30:05Z",
    "startedAt": "2024-02-06T10:30:01Z",
    "estimatedCompletionAt": "2024-02-06T10:32:00Z"
  }
}
```

### 3. Job Completes
```bash
GET /v3/jobs/job_abc123xyz
```

**Response: 200 OK (Completed)**
```json
{
  "data": {
    "jobId": "job_abc123xyz",
    "type": "dataprint_processing",
    "status": "completed",
    "progress": {
      "current": 5,
      "total": 5,
      "percentage": 100,
      "message": "Processing complete"
    },
    "result": {
      "accepted": true,
      "environmentsProcessed": 1,
      "assetsProcessed": 5,
      "warnings": [],
      "errors": []
    },
    "resourceType": "dataprint",
    "resourceId": "dataprint_xyz789",
    "environmentId": "env_8888",
    "inspectorId": "inspector_13",
    "createdAt": "2024-02-06T10:30:00Z",
    "updatedAt": "2024-02-06T10:30:15Z",
    "startedAt": "2024-02-06T10:30:01Z",
    "completedAt": "2024-02-06T10:30:15Z"
  }
}
```

---

## Callback URLs (One-Time Notifications)

Instead of polling, provide a callback URL to receive a POST notification when processing completes.

### Submit with Callback
```bash
POST /v3/environments/env_8888/inspectors/inspector_13/dataprints?callbackUrl=https://partner.example.com/webhooks/dataprint-complete
```

### Callback Payload
When the job completes, we POST to your callback URL:

**POST https://partner.example.com/webhooks/dataprint-complete**
```json
{
  "data": {
    "jobId": "job_abc123xyz",
    "type": "dataprint_processing",
    "status": "completed",
    "result": {
      "accepted": true,
      "environmentsProcessed": 1,
      "assetsProcessed": 5,
      "warnings": [],
      "errors": []
    },
    "resourceType": "dataprint",
    "resourceId": "dataprint_xyz789",
    "environmentId": "env_8888",
    "inspectorId": "inspector_13",
    "createdAt": "2024-02-06T10:30:00Z",
    "completedAt": "2024-02-06T10:30:15Z"
  }
}
```

### Callback Requirements
- **HTTPS only** (no HTTP)
- Must return **2xx status code** within 10 seconds
- Retry: 3 attempts with exponential backoff (1s, 5s, 15s)
- No authentication headers (use signature verification)

### Callback Security
We include a signature header for verification:

```
X-Liongard-Signature: sha256=abc123...
X-Liongard-Timestamp: 1707217800
```

Verify signature:
```python
import hmac
import hashlib

def verify_callback(payload, signature, timestamp, secret):
    # Construct signed payload
    signed_payload = f"{timestamp}.{payload}"
    
    # Compute HMAC
    expected = hmac.new(
        secret.encode(),
        signed_payload.encode(),
        hashlib.sha256
    ).hexdigest()
    
    # Compare
    return hmac.compare_digest(f"sha256={expected}", signature)
```

---

## Job Cancellation

Cancel pending or processing jobs via the API.

### Cancel a Job
```bash
DELETE /v3/jobs/job_abc123xyz
```

**Response: 200 OK**
```json
{
  "data": {
    "jobId": "job_abc123xyz",
    "type": "dataprint_processing",
    "status": "cancelled",
    "environmentId": "env_8888",
    "inspectorId": "inspector_13",
    "createdAt": "2024-02-06T10:30:00Z",
    "updatedAt": "2024-02-06T10:30:10Z",
    "startedAt": "2024-02-06T10:30:01Z",
    "completedAt": "2024-02-06T10:30:10Z"
  }
}
```

### Cancellation Behavior

| Job Status | Cancellable? | Behavior |
|------------|--------------|----------|
| **pending** | Yes | Removed from queue immediately |
| **processing** | Yes | Gracefully stopped (2-5 seconds) |
| **completed** | No | Returns 409 Conflict |
| **failed** | No | Returns 409 Conflict |
| **cancelled** | No | Returns 409 Conflict |

### When to Cancel

- **Wrong data submitted**: Uploaded incorrect file
- **Duplicate submission**: Accidentally submitted twice
- **Configuration changed**: Need to update config before processing
- **Stuck job**: Job taking too long (investigate first)
- **Testing**: Cancel test jobs during development

### Example: Cancel and Resubmit

```python
def resubmit_dataprint(env_id, inspector_id, data, api_key):
    """
    Cancel any existing job and submit new data
    """
    # Check for existing job
    jobs = requests.get(
        f"https://api.liongard.example.com/v3/jobs",
        headers={"X-API-Key": api_key},
        params={
            "filter": f"environmentId=={env_id};inspectorId=={inspector_id}&status__in=pending,processing"
        }
    )
    
    # Cancel if exists
    for job in jobs.json()["data"]:
        requests.delete(
            f"https://api.liongard.example.com/v3/jobs/{job['jobId']}",
            headers={"X-API-Key": api_key}
        )
        print(f"Cancelled existing job: {job['jobId']}")
    
    # Submit new data
    response = requests.post(
        f"https://api.liongard.example.com/v3/environments/{env_id}/inspectors/{inspector_id}/dataprints",
        headers={"X-API-Key": api_key},
        json=data
    )
    
    return response.json()["jobId"]
```

---

## Concurrency Rules

### Per-Inspector Per-Environment Limitation

**Rule**: Only **ONE** dataprint job can process at a time for a given inspector in a specific environment.

**Why?** 
- Prevents data race conditions
- Ensures correct change detection
- Maintains data integrity
- Avoids resource contention

### Conflict Response

If you attempt to submit a dataprint while a job is already running:

**Request:**
```bash
POST /v3/environments/env_8888/inspectors/inspector_13/dataprints
```

**Response: 409 Conflict**
```json
{
  "error": {
    "code": "CONCURRENT_JOB_CONFLICT",
    "message": "A dataprint job is already processing for inspector 'inspector_13' in environment 'env_8888'",
    "details": {
      "existingJobId": "job_existing123",
      "existingJobStatus": "processing",
      "existingJobCreatedAt": "2024-02-06T10:28:00Z",
      "suggestion": "Wait for the existing job to complete or cancel it using DELETE /v3/jobs/job_existing123"
    }
  }
}
```

### Handling Conflicts

**Option 1: Wait for Completion**
```python
def wait_for_slot(env_id, inspector_id, api_key, timeout=300):
    """
    Wait for any existing job to complete before submitting
    """
    start = time.time()
    
    while (time.time() - start) < timeout:
        # Check for existing jobs
        jobs = get_active_jobs(env_id, inspector_id, api_key)
        
        if not jobs:
            return True  # Slot available
        
        # Wait for existing job
        job_id = jobs[0]["jobId"]
        job = get_job_status(job_id, api_key)
        
        if job["status"] in ["completed", "failed", "cancelled"]:
            return True
        
        time.sleep(5)
    
    return False  # Timeout
```

**Option 2: Cancel Existing Job**
```python
def force_submit(env_id, inspector_id, data, api_key):
    """
    Cancel any existing job and submit new data
    """
    # Try to submit
    response = submit_dataprint(env_id, inspector_id, data, api_key)
    
    # Handle conflict
    if response.status_code == 409:
        error = response.json()["error"]
        existing_job_id = error["details"]["existingJobId"]
        
        # Cancel existing job
        cancel_job(existing_job_id, api_key)
        
        # Retry submission
        response = submit_dataprint(env_id, inspector_id, data, api_key)
    
    return response.json()["jobId"]
```

**Option 3: Queue Locally**
```python
class DataprintQueue:
    """
    Queue dataprints locally and submit when slot available
    """
    def __init__(self, env_id, inspector_id, api_key):
        self.env_id = env_id
        self.inspector_id = inspector_id
        self.api_key = api_key
        self.queue = []
    
    def enqueue(self, data):
        """Add to queue"""
        self.queue.append(data)
    
    def process_queue(self):
        """Process queue when slots available"""
        while self.queue:
            # Wait for slot
            if not wait_for_slot(self.env_id, self.inspector_id, self.api_key):
                break
            
            # Submit next in queue
            data = self.queue.pop(0)
            job_id = submit_dataprint(self.env_id, self.inspector_id, data, self.api_key)
            
            # Wait for completion before next
            wait_for_job(job_id, self.api_key)
```

### Global Concurrency Limits

In addition to per-inspector limits:

| Scope | Limit | Behavior on Exceed |
|-------|-------|-------------------|
| **Per Inspector Per Environment** | 1 job | 409 Conflict |
| **Per Environment** | 5 concurrent jobs | 429 Rate Limited |
| **Per API Key** | 10 concurrent jobs | 429 Rate Limited |

### Best Practices for Concurrency

**Do:**
- **Check before submitting**: Query active jobs first
- **Handle 409 gracefully**: Wait or cancel as appropriate
- **Use queuing**: For multiple dataprints to same inspector
- **Stagger submissions**: When uploading to multiple inspectors
- **Monitor job count**: Don't exceed API-wide limits

**Don't:**
- **Retry immediately**: Wait or cancel existing job
- **Submit duplicates**: Check if job already exists
- **Ignore 409s**: Will continue to fail without action

---

## Concurrency Checking Helper

```python
def can_submit_dataprint(env_id, inspector_id, api_key):
    """
    Check if a dataprint can be submitted for this inspector/environment
    
    Returns:
        (bool, str): (can_submit, reason)
    """
    # Check for active jobs
    response = requests.get(
        "https://api.liongard.example.com/v3/jobs",
        headers={"X-API-Key": api_key},
        params={
            "filter": f"environmentId=={env_id};inspectorId=={inspector_id}&status__in=pending,processing",
            "limit": 1
        }
    )
    
    jobs = response.json()["data"]
    
    if jobs:
        job = jobs[0]
        return (
            False, 
            f"Job {job['jobId']} is currently {job['status']}"
        )
    
    return (True, "Ready to submit")

# Usage
can_submit, reason = can_submit_dataprint("env_8888", "inspector_13", api_key)

if can_submit:
    job_id = submit_dataprint(env_id, inspector_id, data, api_key)
else:
    print(f"Cannot submit: {reason}")
```

---

## Job Statuses

| Status | Description | Final? |
|--------|-------------|--------|
| **pending** | Job queued, not started | No |
| **processing** | Job actively processing | No |
| **completed** | Job finished successfully | Yes |
| **failed** | Job encountered fatal error | Yes |
| **cancelled** | Job was cancelled | Yes |

---

## Polling Best Practices

### Recommended Polling Strategy

```python
import time
import requests

def wait_for_job(job_id, api_key, timeout=300):
    """
    Poll job until completion or timeout
    
    Args:
        job_id: Job identifier
        api_key: API key
        timeout: Max wait time in seconds
    
    Returns:
        Final job status
    """
    start = time.time()
    interval = 2  # Start with 2 second intervals
    
    while (time.time() - start) < timeout:
        response = requests.get(
            f"https://api.liongard.example.com/v3/jobs/{job_id}",
            headers={"X-API-Key": api_key}
        )
        
        job = response.json()["data"]
        status = job["status"]
        
        # Check if done
        if status in ["completed", "failed", "cancelled"]:
            return job
        
        # Progress-based intervals
        progress = job.get("progress", {}).get("percentage", 0)
        if progress < 20:
            interval = 5  # Slow start
        elif progress < 80:
            interval = 3  # Normal pace
        else:
            interval = 1  # Near completion
        
        time.sleep(interval)
    
    raise TimeoutError(f"Job {job_id} did not complete within {timeout}s")
```

### Polling Guidelines
- **Start slow**: Begin with 2-5 second intervals
- **Adapt**: Adjust based on progress percentage
- **Respect rate limits**: Don't poll more than once per second
- **Timeout**: Set reasonable max wait time (e.g., 5-10 minutes)
- **Handle errors**: Check for `failed` status and inspect `error` field

---

## Webhooks vs Callbacks

| Feature | Webhooks | Callbacks |
|---------|----------|-----------|
| **Scope** | All events of a type | Single job completion |
| **Setup** | One-time subscription | Per-request parameter |
| **Use Case** | Continuous monitoring | One-off operations |
| **Delivery** | Multiple retries | 3 attempts |
| **Best For** | Long-term integrations | Ad-hoc requests |

### When to Use What?

**Use Webhooks when:**
- Monitoring all dataprint updates continuously
- Building real-time dashboards
- Triggering workflows on any dataprint change
- Long-term production integration

**Use Callbacks when:**
- One-off data import
- Testing/debugging
- User-initiated sync operations
- Don't want persistent webhook configuration

**Use Polling when:**
- Interactive UI operations
- Need real-time progress updates
- Callback URL not available
- Simpler implementation preferred

---

## List Jobs Endpoint

Query recent jobs for monitoring and debugging:

```bash
GET /v3/jobs?status=failed&limit=10&sort=-createdAt
```

**Response:**
```json
{
  "data": [
    {
      "jobId": "job_def456",
      "type": "dataprint_processing",
      "status": "failed",
      "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid JSONPath in assets.uniqueIdPath"
      },
      "environmentId": "env_9999",
      "createdAt": "2024-02-06T10:25:00Z",
      "completedAt": "2024-02-06T10:25:02Z"
    }
  ],
  "pagination": {
    "limit": 10,
    "offset": 0,
    "count": 1,
    "hasMore": false
  }
}
```

**Query Parameters:**
- `status` - Filter by status (pending, processing, completed, failed)
- `limit`, `offset` - Pagination
- `sort` - Sort field (createdAt, updatedAt, completedAt)
- `filter` - FastAPI Filter syntax

---

## Error Handling

### Failed Jobs

When `status` is `failed`, the job includes error details:

```json
{
  "jobId": "job_xyz",
  "status": "failed",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Data structure does not match config",
    "details": {
      "field": "assets.uniqueIdPath",
      "path": "$.serial_number",
      "issue": "Path did not resolve for some assets"
    }
  },
  "result": {
    "accepted": false,
    "environmentsProcessed": 1,
    "assetsProcessed": 3,
    "warnings": ["Device 'acme-ws01' has no IP address"],
    "errors": [
      "Device 'acme-srv02' missing required field: serial_number"
    ]
  }
}
```

### Common Error Codes

| Code | Description | Recovery |
|------|-------------|----------|
| `VALIDATION_ERROR` | Data doesn't match schema | Fix data format |
| `CONFIG_VERSION_MISMATCH` | Config changed during processing | Re-submit with new config |
| `JSONPATH_ERROR` | Invalid JSONPath expression | Fix config metadata |
| `RATE_LIMIT_EXCEEDED` | Too many concurrent jobs | Wait and retry |
| `TIMEOUT` | Processing took too long | Break into smaller batches |

---

## Implementation Examples

### Example 1: Synchronous-Style Wrapper

```python
def push_dataprint_sync(env_id, inspector_id, data, api_key, timeout=300):
    """
    Push dataprint and wait for completion (blocking)
    """
    # Submit data
    response = requests.post(
        f"https://api.liongard.example.com/v3/environments/{env_id}/inspectors/{inspector_id}/dataprints",
        headers={"X-API-Key": api_key},
        json=data
    )
    
    job_id = response.json()["jobId"]
    
    # Wait for completion
    job = wait_for_job(job_id, api_key, timeout)
    
    if job["status"] == "failed":
        raise Exception(f"Dataprint processing failed: {job['error']['message']}")
    
    return job["result"]
```

### Example 2: Async with Callback

```python
def push_dataprint_async(env_id, inspector_id, data, api_key, callback_url):
    """
    Push dataprint with callback (non-blocking)
    """
    response = requests.post(
        f"https://api.liongard.example.com/v3/environments/{env_id}/inspectors/{inspector_id}/dataprints",
        headers={"X-API-Key": api_key},
        params={"callbackUrl": callback_url},
        json=data
    )
    
    return response.json()["jobId"]

# Later, in your webhook handler:
@app.route("/webhooks/dataprint-complete", methods=["POST"])
def handle_completion():
    job = request.json["data"]
    
    if job["status"] == "completed":
        print(f"Success: {job['result']['assetsProcessed']} assets processed")
    else:
        print(f"Failed: {job['error']['message']}")
    
    return "", 200
```

### Example 3: Progress Monitoring

```python
def push_with_progress(env_id, inspector_id, data, api_key, progress_callback):
    """
    Push dataprint with real-time progress updates
    """
    # Submit
    response = requests.post(
        f"https://api.liongard.example.com/v3/environments/{env_id}/inspectors/{inspector_id}/dataprints",
        headers={"X-API-Key": api_key},
        json=data
    )
    
    job_id = response.json()["jobId"]
    
    # Poll with progress callback
    while True:
        job_response = requests.get(
            f"https://api.liongard.example.com/v3/jobs/{job_id}",
            headers={"X-API-Key": api_key}
        )
        
        job = job_response.json()["data"]
        
        # Call progress callback
        progress = job.get("progress", {})
        progress_callback(
            percentage=progress.get("percentage", 0),
            message=progress.get("message", "Processing...")
        )
        
        # Check completion
        if job["status"] in ["completed", "failed"]:
            return job
        
        time.sleep(2)
```

---

## Job Retention

- Jobs are retained for **30 days** after completion
- Older jobs are automatically purged
- Download/store results if needed for longer retention

---

## Concurrency Limits

- **Per API Key**: 10 concurrent dataprint processing jobs
- **Per Environment**: 5 concurrent jobs
- Exceeding limits returns `429 Too Many Requests`

---

## Best Practices Summary

**Do:**
- **Use callbacks** for most production use cases
- **Poll intelligently** - adapt interval based on progress
- **Handle failures** - check status and error details
- **Set timeouts** - don't poll forever
- **Monitor job list** - track failures for debugging
- **Batch operations** - don't overload with too many concurrent jobs
- **Store job IDs** - for audit trails and troubleshooting

**Don't:**
- **Poll aggressively** - respect rate limits
- **Ignore errors** - always check job status
- **Forget timeouts** - prevent hanging operations
- **Use HTTP callbacks** - HTTPS only for security

---

## Migration from Synchronous v1/v2

### v1/v2 Behavior
```bash
POST /v1/dataprints
→ Waits for processing (blocks 30-120s)
→ Returns final result directly
```

### v3 Behavior
```bash
POST /v3/.../dataprints
→ Returns immediately (202 Accepted)
→ Returns job ID
→ Poll or callback for result
```

### Migration Strategy

**Option 1: Add wrapper function** (easiest)
```python
# Wrap async API to behave synchronously
result = push_dataprint_sync(env_id, inspector_id, data, api_key)
```

**Option 2: Refactor to async** (recommended)
```python
# Push data
job_id = push_dataprint_async(env_id, inspector_id, data, api_key, callback_url)

# Continue with other work...
# Result arrives via callback later
```

---

## Summary

- **Dataprint POST returns 202 Accepted** with job ID
- **Poll GET /v3/jobs/{jobId}** for status
- **Optional callback URL** for one-time notification
- **Progress tracking** with percentage and messages
- **Webhooks** for continuous monitoring
- **Job history** via GET /v3/jobs
- **Error details** in failed jobs
- **30-day retention** for completed jobs

**API Changes:**
- Dataprint POST: `201 Created` → `202 Accepted`
- Returns: `dataprint processingResultResponse` → `dataprintJobResponse`
- New: 2 job endpoints for tracking

This enables non-blocking integrations while providing multiple ways to track completion!
