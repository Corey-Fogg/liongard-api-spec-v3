# RSQL Filter Syntax Guide

## Overview
The Liongard Data API v3 uses **RSQL (RESTful Service Query Language)** for filtering. RSQL provides a clean, readable syntax using familiar comparison operators.

---

## Basic Syntax

### Simple Equality
```
?filter=field==value
```

Example:
```bash
GET /v3/jobs?filter=status==pending
```

### Not Equal
```
?filter=field!=value
```

Example:
```bash
GET /v3/jobs?filter=status!=cancelled
```

---

## Comparison Operators

### Numeric Comparisons
```
field>value         # Greater than
field>=value        # Greater than or equal
field<value         # Less than
field<=value        # Less than or equal
```

Examples:
```bash
?filter=assetsProcessed>10
?filter=createdAt>=2024-01-01T00:00:00Z
?filter=priority<critical
?filter=completedAt<=2024-02-06T23:59:59Z
```

---

## Set Operations

### IN (value is in list)
```
field=in=(val1,val2,val3)
```

Examples:
```bash
?filter=status=in=(pending,processing)
?filter=priority=in=(high,critical)
?filter=assetType=in=(server,workstation,firewall)
```

### OUT (value is not in list)
```
field=out=(val1,val2)
```

Examples:
```bash
?filter=status=out=(cancelled,failed)
?filter=environmentId=out=(env_archived1,env_archived2)
```

---

## Pattern Matching (Wildcards)

### Contains
```
field==*substring*
```

Example:
```bash
?filter=name==*server*
?filter=hostname==*acme*
```

### Starts With
```
field==prefix*
```

Example:
```bash
?filter=name==srv-*
?filter=serialNumber==SN-*
```

### Ends With
```
field==*suffix
```

Example:
```bash
?filter=name==*-prod
?filter=email==*@example.com
```

---

## Combining Conditions

### AND (semicolon)
```
condition1;condition2;condition3
```

Examples:
```bash
# Active critical alerts
?filter=status==open;priority==critical

# Jobs in environment for specific inspector
?filter=environmentId==env_8888;inspectorId==inspector_13

# Date range
?filter=createdAt>=2024-01-01;createdAt<2024-02-01
```

### OR (comma)
```
condition1,condition2,condition3
```

Examples:
```bash
# Open or in-progress
?filter=status==open,status==in_progress

# Multiple asset types
?filter=assetType==server,assetType==workstation

# Search multiple name patterns
?filter=name==*server*,name==*router*
```

### Grouping (parentheses)
```
condition1;(condition2,condition3)
```

Examples:
```bash
# Critical environment with either open or in-progress status
?filter=environmentId==env_8888;(status==open,status==in_progress)

# Multiple conditions with OR groups
?filter=(priority==high,priority==critical);status==open
```

---

## Jobs Endpoint Examples

### Check for Active Jobs
```bash
# Any active jobs
GET /v3/jobs?filter=status=in=(pending,processing)

# Active jobs for specific inspector/environment
GET /v3/jobs?filter=environmentId==env_8888;inspectorId==inspector_13;status=in=(pending,processing)

# Check if slot available
GET /v3/jobs?filter=environmentId==env_8888;inspectorId==inspector_13;status=in=(pending,processing)&limit=1
```

### Find Failed Jobs
```bash
# All failed jobs
GET /v3/jobs?filter=status==failed

# Failed dataprint jobs
GET /v3/jobs?filter=type==dataprint_processing;status==failed

# Failed jobs in last 24 hours
GET /v3/jobs?filter=status==failed;createdAt>=2024-02-05T00:00:00Z
```

### Monitor Specific Environment
```bash
# All jobs for an environment
GET /v3/jobs?filter=environmentId==env_8888

# Recent completed jobs for environment
GET /v3/jobs?filter=environmentId==env_8888;status==completed&sort=-completedAt&limit=10

# Jobs by type
GET /v3/jobs?filter=environmentId==env_8888;type==dataprint_processing
```

### Find Jobs with Callbacks
```bash
# Jobs that had callbacks (callback URL is not empty/null)
GET /v3/jobs?filter=callbackUrl!=null

# Jobs with multiple callback attempts
GET /v3/jobs?filter=callbackAttempts>1
```

### Date Range Queries
```bash
# Jobs created today
GET /v3/jobs?filter=createdAt>=2024-02-06T00:00:00Z

# Jobs completed in January
GET /v3/jobs?filter=completedAt>=2024-01-01T00:00:00Z;completedAt<2024-02-01T00:00:00Z

# Long-running jobs (started over 5 minutes ago, still processing)
GET /v3/jobs?filter=status==processing;startedAt<2024-02-06T10:25:00Z
```

---

## Alerts Endpoint Examples

### By Status and Priority
```bash
# Open alerts
GET /v3/alerts?filter=status==open

# Open or in progress
GET /v3/alerts?filter=status=in=(open,in_progress)

# Critical priority only
GET /v3/alerts?filter=priority==critical

# Critical and high priority that are open
GET /v3/alerts?filter=priority=in=(high,critical);status==open
```

### By Environment
```bash
# All alerts for environment
GET /v3/alerts?filter=environmentId==env_8888

# Critical alerts for environment
GET /v3/alerts?filter=environmentId==env_8888;priority==critical;status==open
```

### Date Filters
```bash
# Alerts created today
GET /v3/alerts?filter=createdAt>=2024-02-06T00:00:00Z

# Overdue alerts (past due date)
GET /v3/alerts?filter=dueDate<2024-02-06T00:00:00Z;status=in=(open,in_progress)

# Alerts auto-closing soon (older than 30 days)
GET /v3/alerts?filter=autoClose>0;createdAt<=2024-01-06T00:00:00Z
```

### Text Search
```bash
# Search by name
GET /v3/alerts?filter=name==*vulnerability*

# Search by keywords
GET /v3/alerts?filter=keywords==*security*

# Case-sensitive exact match
GET /v3/alerts?filter=name==Critical Alert
```

---

## Assets Endpoint Examples

### By Type and Status
```bash
# All servers
GET /v3/assets?filter=assetType==server

# Active workstations
GET /v3/assets?filter=assetType==workstation;status==active

# Critical assets
GET /v3/assets?filter=class==critical

# Servers or firewalls
GET /v3/assets?filter=assetType=in=(server,firewall)
```

### By Inventory State
```bash
# Active inventory
GET /v3/assets?filter=inventoryState==inventory

# Archived assets
GET /v3/assets?filter=inventoryState==archive

# Recently discovered
GET /v3/assets?filter=inventoryState==discovery
```

### By Location/Management
```bash
# Managed devices
GET /v3/assets?filter=managedDevice==true

# Physical servers
GET /v3/assets?filter=assetType==server;physical==true

# Assets in specific location
GET /v3/assets?filter=location==*Dallas*

# Assets not in specific locations
GET /v3/assets?filter=location=out=(Archive,Decommissioned)
```

### Lifecycle Queries
```bash
# Warranty expiring soon (within 30 days)
GET /v3/assets?filter=warrantyExpiration>=2024-02-06T00:00:00Z;warrantyExpiration<=2024-03-06T00:00:00Z

# Assets past EOL
GET /v3/assets?filter=eolDate<2024-02-06T00:00:00Z

# Recently seen (active in last 24 hours)
GET /v3/assets?filter=lastSeenAt>=2024-02-05T00:00:00Z
```

---

## Identities Endpoint Examples

### By Type and Status
```bash
# Admin accounts
GET /v3/identities?filter=identityType==admin

# Active users
GET /v3/identities?filter=status==active

# Service accounts
GET /v3/identities?filter=identityType==service

# Admin or service accounts
GET /v3/identities?filter=identityType=in=(admin,service)
```

### Security Filters
```bash
# Privileged accounts without MFA
GET /v3/identities?filter=privileged==true;mfaEnabled==false

# All privileged accounts
GET /v3/identities?filter=privileged==true

# Accounts with expiring passwords (next 7 days)
GET /v3/identities?filter=passwordExpires>=2024-02-06T00:00:00Z;passwordExpires<=2024-02-13T00:00:00Z

# Accounts without MFA (high or critical)
GET /v3/identities?filter=mfaEnabled==false;(privileged==true,identityType==admin)
```

### Activity Filters
```bash
# Recently logged in (last 24 hours)
GET /v3/identities?filter=lastLogin>=2024-02-05T00:00:00Z

# Stale accounts (no login in 90+ days)
GET /v3/identities?filter=lastLogin<2024-11-07T00:00:00Z

# Accounts with old passwords (set over 180 days ago)
GET /v3/identities?filter=passwordLastSet<2023-08-08T00:00:00Z

# Never logged in
GET /v3/identities?filter=lastLogin==null
```

---

## Metrics Endpoint Examples

### By Inspector
```bash
# Metrics for specific inspector
GET /v3/metrics?filter=inspectorId==inspector_13

# Custom metrics only
GET /v3/metrics?filter=isCustom==true

# Metrics with active alert rules
GET /v3/metrics?filter=hasActiveAlertRule==true
```

### By Display State
```bash
# Visible metrics
GET /v3/metrics?filter=metricDisplay==true

# Change-enabled metrics
GET /v3/metrics?filter=changesEnabled==true

# Both visible and change-enabled
GET /v3/metrics?filter=metricDisplay==true;changesEnabled==true
```

### By Scope
```bash
# Global metrics (no environment)
GET /v3/metrics?filter=environmentId==null

# Environment-specific metrics
GET /v3/metrics?filter=environmentId==env_8888

# Metrics for multiple environments
GET /v3/metrics?filter=environmentId=in=(env_8888,env_9999)
```

---

## Combining with Other Parameters

### Pagination
```bash
GET /v3/jobs?filter=status==completed&limit=50&offset=100
```

### Sorting
```bash
GET /v3/alerts?filter=priority==critical&sort=-createdAt
```

### Field Expansion
```bash
GET /v3/alerts?filter=status==open&expand=environment,inspector
```

### Combined Example
```bash
GET /v3/jobs?filter=environmentId==env_8888;status=in=(pending,processing)&sort=-createdAt&limit=20&offset=0
```

---

## Implementation Examples

### Python
```python
import requests
from urllib.parse import quote

def get_active_jobs(env_id, inspector_id, api_key):
    """Get active jobs for a specific inspector/environment"""
    # Build RSQL filter
    filter_expr = f"environmentId=={env_id};inspectorId=={inspector_id};status=in=(pending,processing)"
    
    params = {
        "filter": filter_expr,
        "limit": 10
    }
    
    response = requests.get(
        "https://api.liongard.example.com/v3/jobs",
        headers={"X-API-Key": api_key},
        params=params
    )
    
    return response.json()["data"]

def get_critical_alerts(env_id, api_key):
    """Get open critical alerts for environment"""
    filter_expr = f"environmentId=={env_id};priority==critical;status==open"
    
    params = {
        "filter": filter_expr,
        "sort": "-createdAt",
        "limit": 50
    }
    
    response = requests.get(
        "https://api.liongard.example.com/v3/alerts",
        headers={"X-API-Key": api_key},
        params=params
    )
    
    return response.json()["data"]

def search_assets_by_name(search_term, api_key):
    """Search assets by name pattern"""
    filter_expr = f"name==*{search_term}*"
    
    response = requests.get(
        "https://api.liongard.example.com/v3/assets",
        headers={"X-API-Key": api_key},
        params={"filter": filter_expr, "limit": 100}
    )
    
    return response.json()["data"]
```

### JavaScript
```javascript
async function getActiveJobs(envId, inspectorId, apiKey) {
  // Build RSQL filter
  const filter = `environmentId==${envId};inspectorId==${inspectorId};status=in=(pending,processing)`;
  
  const params = new URLSearchParams({
    filter: filter,
    limit: 10
  });
  
  const response = await fetch(
    `https://api.liongard.example.com/v3/jobs?${params}`,
    { headers: { "X-API-Key": apiKey } }
  );
  
  const json = await response.json();
  return json.data;
}

async function getCriticalAlerts(envId, apiKey) {
  const filter = `environmentId==${envId};priority==critical;status==open`;
  
  const params = new URLSearchParams({
    filter: filter,
    sort: "-createdAt",
    limit: 50
  });
  
  const response = await fetch(
    `https://api.liongard.example.com/v3/alerts?${params}`,
    { headers: { "X-API-Key": apiKey } }
  );
  
  const json = await response.json();
  return json.data;
}
```

### cURL
```bash
# Active jobs for inspector/environment
curl -X GET "https://api.liongard.example.com/v3/jobs?filter=environmentId==env_8888;inspectorId==inspector_13;status=in=(pending,processing)" \
  -H "X-API-Key: your-api-key"

# Critical open alerts
curl -X GET "https://api.liongard.example.com/v3/alerts?filter=priority==critical;status==open&sort=-createdAt" \
  -H "X-API-Key: your-api-key"

# Assets with name pattern
curl -X GET "https://api.liongard.example.com/v3/assets?filter=name==*server*" \
  -H "X-API-Key: your-api-key"
```

---

## URL Encoding Notes

Special characters in RSQL filters should be URL-encoded when used in URLs:

| Character | URL Encoded | Example |
|-----------|-------------|---------|
| `==` | `%3D%3D` | (usually not needed in query strings) |
| `;` | `%3B` | (AND separator) |
| `,` | `%2C` | (OR separator) |
| `>` | `%3E` | (greater than) |
| `<` | `%3C` | (less than) |
| `*` | `%2A` | (wildcard) |

Most HTTP libraries handle this automatically, but if constructing URLs manually, ensure proper encoding.

---

## Common Patterns

### Check if Resource Exists
```bash
GET /v3/jobs?filter=environmentId==env_8888;inspectorId==inspector_13;status=in=(pending,processing)&limit=1
# If data array is empty, no active job exists
```

### Get Recent Items
```bash
GET /v3/alerts?filter=createdAt>=2024-02-06T00:00:00Z&sort=-createdAt&limit=20
```

### Exclude Values
```bash
GET /v3/jobs?filter=status=out=(cancelled,failed)
# Or using NOT equal for single value
GET /v3/jobs?filter=status!=cancelled
```

### Complex Business Logic
```bash
# High/critical open alerts in production environments
GET /v3/alerts?filter=priority=in=(high,critical);status==open;environmentId==*-prod*

# Privileged accounts without MFA that logged in recently
GET /v3/identities?filter=privileged==true;mfaEnabled==false;lastLogin>=2024-02-01T00:00:00Z
```

---

## Best Practices

### ✅ Do
- Use `==` for equality (double equals)
- Use `;` for AND conditions
- Use `,` for OR conditions  
- Use parentheses `()` for grouping complex logic
- Use `*` wildcards for pattern matching
- Use `=in=()` for multiple values instead of multiple OR conditions
- URL-encode special characters when constructing URLs manually
- Add `limit` to prevent large responses

### ❌ Don't
- Use single `=` for equality (use `==`)
- Use `&` for AND (use `;`)
- Use `|` for OR (use `,`)
- Forget to enclose IN/OUT values in parentheses
- Mix AND/OR without parentheses in complex expressions
- Request all data and filter client-side

---

## RSQL Operator Quick Reference

| Operator | Meaning | Example |
|----------|---------|---------|
| `==` | Equal | `status==pending` |
| `!=` | Not equal | `status!=cancelled` |
| `>` | Greater than | `priority>low` |
| `>=` | Greater than or equal | `createdAt>=2024-01-01` |
| `<` | Less than | `assetsProcessed<10` |
| `<=` | Less than or equal | `completedAt<=2024-02-06` |
| `=in=()` | In list | `status=in=(pending,processing)` |
| `=out=()` | Not in list | `status=out=(cancelled,failed)` |
| `==*text*` | Contains | `name==*server*` |
| `==prefix*` | Starts with | `name==srv-*` |
| `==*suffix` | Ends with | `name==*-prod` |
| `;` | AND | `status==open;priority==high` |
| `,` | OR | `status==open,status==in_progress` |
| `()` | Grouping | `priority==high;(status==open,status==in_progress)` |

---

## RSQL vs Other Syntaxes

### Why RSQL?

**Compared to OData:**
- ✅ Simpler syntax (no `$filter=`)
- ✅ No spaces in operators
- ✅ More intuitive for developers

**Compared to MongoDB:**
- ✅ More readable for URLs
- ✅ No JSON in query strings
- ✅ Simpler to construct

**Compared to GraphQL:**
- ✅ Works with REST
- ✅ No schema language needed
- ✅ Standard HTTP GET requests

**Compared to Custom Syntax:**
- ✅ Well-documented standard
- ✅ Libraries available in many languages
- ✅ Familiar to developers who've used it before

---

## Libraries & Tools

### Parser Libraries
- **Java**: rsql-parser
- **Node.js**: rsql-builder, node-rsql-parser
- **Python**: py-rsql-parser
- **.NET**: RSQLParser.NET
- **PHP**: rsql-php-parser

### Testing
Test RSQL filters using:
- Postman/Insomnia
- cURL
- Browser DevTools

---

## Summary

✅ **Clean, readable syntax** using familiar operators (`==`, `!=`, `>`, `<`)  
✅ **Powerful combinators** (`;` for AND, `,` for OR)  
✅ **Pattern matching** with wildcards (`*`)  
✅ **Set operations** (`=in=`, `=out=`)  
✅ **Standard specification** with library support  
✅ **URL-friendly** - works great in query strings  

RSQL provides the perfect balance of power and simplicity for REST API filtering!
