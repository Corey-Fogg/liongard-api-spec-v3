---
title: Response Format
parent: Technical References
nav_order: 3
---

# API Response Format - No Wrappers, Headers for Metadata

## Overview

The Liongard Vendor API v3 follows proper HTTP conventions:
- **Response bodies** contain only the actual data
- **Metadata** (pagination, etc.) goes in HTTP headers
- **No wrapper objects** - cleaner, more RESTful

---

## Single Resource Endpoints

### GET Single Resource

**Endpoint:** `GET /v3/alerts/{alertId}`

**Response Body (200 OK):**
```json
{
  "alertId": "alert_7586",
  "environmentId": "env_8888",
  "name": "Critical vulnerability detected",
  "status": "open",
  "priority": "critical",
  "createdAt": "2024-02-06T10:30:00Z"
}
```

**No wrapper** - just the object directly.

### POST/PUT/PATCH Single Resource

**Endpoint:** `POST /v3/inspectors`

**Response Body (201 Created):**
```json
{
  "inspectorId": "inspector_abc123",
  "name": "acme-monitor",
  "displayName": "Acme Monitoring Tool",
  "status": "active"
}
```

---

## List Endpoints (Collections)

### GET Collection

**Endpoint:** `GET /v3/assets?limit=100&offset=0`

**Response Body (200 OK):**
```json
[
  {
    "assetId": "asset_1",
    "name": "server-01",
    "assetType": "server",
    "status": "active"
  },
  {
    "assetId": "asset_2",
    "name": "server-02", 
    "assetType": "server",
    "status": "active"
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
      <https://api.liongard.com/v3/assets?limit=100&offset=400>; rel="last"
```

---

## Pagination Headers

### All Pagination Headers

| Header | Type | Description |
|--------|------|-------------|
| `X-Pagination-Limit` | integer | Items per page (from request) |
| `X-Pagination-Offset` | integer | Items skipped (from request) |
| `X-Pagination-Count` | integer | Total items available |
| `X-Pagination-Has-More` | boolean | Whether more pages exist |
| `Link` | string | RFC 5988 pagination links |

### Link Header Format

The `Link` header follows RFC 5988 and includes:

```
<url>; rel="next"    - Next page
<url>; rel="prev"    - Previous page
<url>; rel="first"   - First page
<url>; rel="last"    - Last page
```

**Example:**
```
Link: <https://api.liongard.com/v3/alerts?limit=100&offset=100>; rel="next",
      <https://api.liongard.com/v3/alerts?limit=100&offset=0>; rel="first",
      <https://api.liongard.com/v3/alerts?limit=100&offset=900>; rel="last"
```

---

## Code Examples

### Python - Single Resource

```python
import requests

# GET single resource
response = requests.get(
    "https://api.liongard.com/v3/alerts/alert_123",
    headers={"X-API-Key": api_key}
)

# Response is object directly
alert = response.json()
print(alert["name"])  # No ["data"] wrapper
```

### Python - List with Pagination

```python
def get_all_assets(api_key):
    """Get all assets across all pages"""
    all_assets = []
    offset = 0
    limit = 100
    
    while True:
        response = requests.get(
            "https://api.liongard.com/v3/assets",
            headers={"X-API-Key": api_key},
            params={"limit": limit, "offset": offset}
        )
        
        # Response is array directly
        assets = response.json()
        all_assets.extend(assets)
        
        # Check pagination header
        has_more = response.headers.get('X-Pagination-Has-More')
        if has_more != 'true':
            break
        
        # Get total count from headers
        total_count = int(response.headers.get('X-Pagination-Count', 0))
        print(f"Retrieved {len(all_assets)}/{total_count} assets")
        
        offset += limit
    
    return all_assets

# Usage
assets = get_all_assets(api_key)
print(f"Total assets: {len(assets)}")
```

### Python - Using Link Header

```python
from requests.utils import parse_header_links

def get_all_using_links(url, headers):
    """Paginate using Link header"""
    all_items = []
    
    while url:
        response = requests.get(url, headers=headers)
        
        # Response is array
        items = response.json()
        all_items.extend(items)
        
        # Parse Link header for next page
        link_header = response.headers.get('Link')
        if not link_header:
            break
        
        links = parse_header_links(link_header)
        next_link = next((l for l in links if l['rel'] == 'next'), None)
        url = next_link['url'] if next_link else None
    
    return all_items

# Usage
alerts = get_all_using_links(
    "https://api.liongard.com/v3/alerts",
    {"X-API-Key": api_key}
)
```

### JavaScript - Fetch with Headers

```javascript
async function getAllAssets(apiKey) {
  const allAssets = [];
  let offset = 0;
  const limit = 100;
  let hasMore = true;
  
  while (hasMore) {
    const response = await fetch(
      `https://api.liongard.com/v3/assets?limit=${limit}&offset=${offset}`,
      {
        headers: { "X-API-Key": apiKey }
      }
    );
    
    // Response is array directly
    const assets = await response.json();
    allAssets.push(...assets);
    
    // Check pagination headers
    hasMore = response.headers.get('X-Pagination-Has-More') === 'true';
    const totalCount = response.headers.get('X-Pagination-Count');
    
    console.log(`Retrieved ${allAssets.length}/${totalCount} assets`);
    
    offset += limit;
  }
  
  return allAssets;
}
```

### cURL - Inspecting Headers

```bash
# Get assets with pagination info
curl -i -X GET "https://api.liongard.com/v3/assets?limit=10&offset=0" \
  -H "X-API-Key: your-api-key"

# Response includes:
HTTP/1.1 200 OK
Content-Type: application/json
X-Pagination-Limit: 10
X-Pagination-Offset: 0
X-Pagination-Count: 547
X-Pagination-Has-More: true
Link: <https://api.liongard.com/v3/assets?limit=10&offset=10>; rel="next"

[
  { "assetId": "asset_1", ... },
  { "assetId": "asset_2", ... }
]
```

---

## Special Cases

### Empty Collections

**Endpoint:** `GET /v3/assets` (with no results)

**Response Body:**
```json
[]
```

**Headers:**
```
X-Pagination-Limit: 100
X-Pagination-Offset: 0
X-Pagination-Count: 0
X-Pagination-Has-More: false
```

### Async Operations (Jobs)

**Endpoint:** `POST /v3/environments/{id}/inspectors/{id}/dataprints`

**Response (202 Accepted):**
```json
{
  "jobId": "job_abc123",
  "status": "pending",
  "message": "Dataprint accepted for processing",
  "pollUrl": "https://api.liongard.com/v3/jobs/job_abc123"
}
```

No wrapper because this is a direct job status object.

### Error Responses

**Response (400 Bad Request):**
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

Error responses have an `error` wrapper for clarity.

---

## HTTP Client Library Support

### requests (Python)

```python
import requests

response = requests.get(url, headers=headers)

# Body
data = response.json()

# Headers
limit = response.headers['X-Pagination-Limit']
has_more = response.headers['X-Pagination-Has-More'] == 'true'
```

### fetch (JavaScript)

```javascript
const response = await fetch(url, { headers });

// Body
const data = await response.json();

// Headers
const limit = response.headers.get('X-Pagination-Limit');
const hasMore = response.headers.get('X-Pagination-Has-More') === 'true';
```

### axios (JavaScript)

```javascript
const response = await axios.get(url, { headers });

// Body
const data = response.data;

// Headers
const limit = response.headers['x-pagination-limit'];
const hasMore = response.headers['x-pagination-has-more'] === 'true';
```

### curl (Bash)

```bash
# Show headers with -i
curl -i GET url -H "X-API-Key: key"

# Headers only with -I
curl -I GET url -H "X-API-Key: key"
```

---

## Summary

- **No wrapper objects** - Direct access to data
- **Pagination in headers** - Proper HTTP usage
- **Standard Link header** - RFC 5988 compliant
- **Cleaner responses** - Less nesting
- **Smaller payloads** - Less data transfer

This is the proper RESTful way to design APIs.
