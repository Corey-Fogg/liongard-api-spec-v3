# API Response Format Update Summary

## What Changed

The API has been updated to follow proper HTTP conventions by removing wrapper objects and moving pagination metadata to HTTP headers.

---

## Before & After

### Single Resource Endpoints

**Before:**
```json
{
  "data": {
    "alertId": "alert_123",
    "name": "Critical Alert",
    "status": "open"
  }
}
```

**After:**
```json
{
  "alertId": "alert_123",
  "name": "Critical Alert",
  "status": "open"
}
```

### List Endpoints

**Before:**
```json
{
  "data": [
    {"assetId": "asset_1", "name": "server-01"},
    {"assetId": "asset_2", "name": "server-02"}
  ],
  "pagination": {
    "limit": 100,
    "offset": 0,
    "count": 547,
    "hasMore": true
  }
}
```

**After:**

**Response Body:**
```json
[
  {"assetId": "asset_1", "name": "server-01"},
  {"assetId": "asset_2", "name": "server-02"}
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

## Impact on Code

### Python

**Before:**
```python
response = requests.get(url, headers=headers)
result = response.json()

# Single resource
item = result["data"]

# List
items = result["data"]
has_more = result["pagination"]["hasMore"]
```

**After:**
```python
response = requests.get(url, headers=headers)

# Single resource
item = response.json()

# List
items = response.json()
has_more = response.headers.get('X-Pagination-Has-More') == 'true'
```

### JavaScript

**Before:**
```javascript
const response = await fetch(url, { headers });
const result = await response.json();

// Single resource
const item = result.data;

// List
const items = result.data;
const hasMore = result.pagination.hasMore;
```

**After:**
```javascript
const response = await fetch(url, { headers });

// Single resource
const item = await response.json();

// List
const items = await response.json();
const hasMore = response.headers.get('X-Pagination-Has-More') === 'true';
```

---

## Pagination Headers

All list endpoints now include:

| Header | Description | Example |
|--------|-------------|---------|
| `X-Pagination-Limit` | Items per page | `100` |
| `X-Pagination-Offset` | Items skipped | `0` |
| `X-Pagination-Count` | Total items | `547` |
| `X-Pagination-Has-More` | More pages exist | `true` |
| `Link` | Pagination links (RFC 5988) | `<url>; rel="next"` |

---

## Link Header Format

The `Link` header follows RFC 5988:

```
Link: <https://api.liongard.com/v3/assets?limit=100&offset=100>; rel="next",
      <https://api.liongard.com/v3/assets?limit=100&offset=0>; rel="first",
      <https://api.liongard.com/v3/assets?limit=100&offset=400>; rel="last"
```

Relations:
- `next` - Next page URL
- `prev` - Previous page URL
- `first` - First page URL
- `last` - Last page URL

---

## Updated Files

### 1. OpenAPI Specification
**File:** `liongard-api-v3-enhanced.yaml`

Changes:
- ✅ All list endpoints return arrays directly
- ✅ All single resource endpoints return objects directly
- ✅ Pagination headers added to all list endpoints
- ✅ Link header (RFC 5988) added
- ✅ Removed 25 response wrapper schemas
- ✅ Kept only 36 core data schemas

**Stats:**
- Endpoints: 26
- Schemas: 36 (down from 61)
- Response wrappers: 0 (except errorResponse)

### 2. Comprehensive Guide
**File:** `COMPREHENSIVE_API_GUIDE.md`

Changes:
- ✅ Updated all code examples to remove `.json()["data"]`
- ✅ Updated pagination examples to use headers
- ✅ Added Link header examples
- ✅ Updated Quick Start guide
- ✅ Updated complete integration example

### 3. Response Format Guide (NEW)
**File:** `RESPONSE_FORMAT_GUIDE.md`

Contents:
- Complete before/after examples
- Pagination header documentation
- Code examples (Python, JavaScript, cURL)
- Migration guide
- HTTP client library usage

---

## Benefits

### ✅ HTTP Standards Compliance
Metadata in headers, content in body - proper HTTP usage.

### ✅ Cleaner Code
```python
# Before
alert_name = response.json()["data"]["name"]

# After
alert_name = response.json()["name"]
```

### ✅ Smaller Payloads
No wrapper objects = less data over the wire.

### ✅ Better Caching
Headers don't pollute response body, allowing better HTTP caching.

### ✅ Standard Link Header
Works automatically with HTTP clients that understand RFC 5988.

### ✅ More RESTful
Follows REST best practices and HTTP protocol design.

---

## Migration Checklist

For developers updating existing integrations:

- [ ] Remove `.json()["data"]` from single resource calls
- [ ] Remove `.json()["data"]` from list calls
- [ ] Update pagination logic to read headers instead of body
- [ ] Update `hasMore` checks to use header
- [ ] Consider using Link header for pagination
- [ ] Update tests to expect arrays/objects directly
- [ ] Update documentation/examples

---

## Examples

### Python - Complete Pagination

```python
def get_all_assets(api_key):
    """Get all assets with proper pagination"""
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
        
        # Check header
        if response.headers.get('X-Pagination-Has-More') != 'true':
            break
        
        offset += limit
    
    return all_assets
```

### Python - Using Link Header

```python
from requests.utils import parse_header_links

def get_all_using_links(url, headers):
    """Paginate using Link header (RFC 5988)"""
    all_items = []
    
    while url:
        response = requests.get(url, headers=headers)
        items = response.json()
        all_items.extend(items)
        
        # Parse Link header
        link_header = response.headers.get('Link')
        if not link_header:
            break
        
        links = parse_header_links(link_header)
        next_link = next((l for l in links if l['rel'] == 'next'), None)
        url = next_link['url'] if next_link else None
    
    return all_items
```

---

## Validation

### OpenAPI Spec
✅ **Valid YAML**  
✅ **26 endpoints** defined  
✅ **36 schemas** (core data only)  
✅ **Zero wrapper schemas** (except errorResponse)  
✅ **All list endpoints** return arrays with pagination headers  
✅ **All single endpoints** return objects directly  
✅ **Link header** (RFC 5988) on all list endpoints  
✅ **Ready for Swagger Editor**  

### Documentation
✅ **Comprehensive Guide** updated with new response format  
✅ **Response Format Guide** created with examples  
✅ **All code examples** updated to remove wrappers  
✅ **Pagination examples** updated to use headers  

---

## Testing the Changes

### Swagger Editor
1. Go to https://editor.swagger.io/
2. Paste the spec
3. Verify no errors/warnings
4. Check response schemas show arrays/objects directly
5. Verify headers are documented

### Sample Request
```bash
curl -i -X GET "https://api.liongard.com/v3/assets?limit=10&offset=0" \
  -H "X-API-Key: your-key"

# Verify:
# - Body is JSON array directly
# - X-Pagination-* headers present
# - Link header present
```

---

## Summary

🎉 **API now follows HTTP best practices!**

✅ **No wrapper objects** - Direct data access  
✅ **Pagination in headers** - Proper HTTP usage  
✅ **RFC 5988 Link header** - Standard pagination  
✅ **36 schemas** (down from 61) - Simpler spec  
✅ **Cleaner code** - Less nesting  
✅ **Better performance** - Smaller payloads  

**The API is now more RESTful, standards-compliant, and developer-friendly!**
