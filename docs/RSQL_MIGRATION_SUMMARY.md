---
title: RSQL Migration Guide
parent: Migration Guides
nav_order: 1
---

# RSQL Filter Syntax - Migration Summary

## What Changed

The API has been updated from FastAPI Filter syntax (with double underscores) to **RSQL (RESTful Service Query Language)** for cleaner, more readable filtering.

---

## Syntax Comparison

### Old (FastAPI Filter with double underscores)
```bash
# Equality
?filter=status=pending

# Multiple values
?filter=status__in=pending,processing

# Comparison
?filter=createdAt__gte=2024-01-01

# Multiple conditions
?filter=environmentId=env_8888&inspectorId=inspector_13&status__in=pending,processing

# Contains
?filter=name__icontains=server
```

### New (RSQL)
```bash
# Equality
?filter=status==pending

# Multiple values
?filter=status=in=(pending,processing)

# Comparison
?filter=createdAt>=2024-01-01

# Multiple conditions (AND with semicolon)
?filter=environmentId==env_8888;inspectorId==inspector_13;status=in=(pending,processing)

# Contains (wildcard)
?filter=name==*server*
```

---

## Key Differences

| Feature | Old (FastAPI) | New (RSQL) |
|---------|---------------|------------|
| **Equality** | `field=value` | `field==value` |
| **Not Equal** | `field__not=value` | `field!=value` |
| **Greater Than** | `field__gt=value` | `field>value` |
| **Greater/Equal** | `field__gte=value` | `field>=value` |
| **Less Than** | `field__lt=value` | `field<value` |
| **Less/Equal** | `field__lte=value` | `field<=value` |
| **In List** | `field__in=a,b` | `field=in=(a,b)` |
| **Not In List** | `field__not_in=a,b` | `field=out=(a,b)` |
| **Contains** | `field__icontains=text` | `field==*text*` |
| **Starts With** | `field__startswith=text` | `field==text*` |
| **Ends With** | `field__endswith=text` | `field==*text` |
| **AND** | `&` | `;` |
| **OR** | Multiple filters | `,` |

---

## Common Queries - Before & After

### Check for Active Jobs
```bash
# OLD
GET /v3/jobs?filter=environmentId=env_8888&inspectorId=inspector_13&status__in=pending,processing

# NEW
GET /v3/jobs?filter=environmentId==env_8888;inspectorId==inspector_13;status=in=(pending,processing)
```

### Find Critical Alerts
```bash
# OLD
GET /v3/alerts?filter=priority=critical&status__in=open,in_progress

# NEW
GET /v3/alerts?filter=priority==critical;status=in=(open,in_progress)
```

### Date Range Query
```bash
# OLD
GET /v3/jobs?filter=createdAt__gte=2024-01-01&createdAt__lt=2024-02-01

# NEW
GET /v3/jobs?filter=createdAt>=2024-01-01;createdAt<2024-02-01
```

### Search by Name Pattern
```bash
# OLD
GET /v3/assets?filter=name__icontains=server

# NEW
GET /v3/assets?filter=name==*server*
```

### Multiple Asset Types
```bash
# OLD
GET /v3/assets?filter=assetType__in=server,workstation,firewall

# NEW
GET /v3/assets?filter=assetType=in=(server,workstation,firewall)
```

---

## Benefits of RSQL

### More Readable
```bash
# Easier to understand
status==pending;priority==critical

# vs
status=pending&priority=critical
```

### Familiar Operators
Uses standard comparison symbols (`>`, `<`, `>=`, `<=`, `==`, `!=`) that developers already know.

### Cleaner Syntax
No weird double underscores (`__in`, `__gte`, `__icontains`).

### Better AND/OR Logic
- `;` for AND (clearer than `&`)
- `,` for OR (explicit grouping)
- `()` for complex logic

### Standard Specification
RSQL is a documented standard with libraries in multiple languages.

---

## Migration Checklist

For developers updating existing integrations:

- [ ] Replace `=` with `==` for equality checks
- [ ] Replace `field__in=a,b` with `field=in=(a,b)`
- [ ] Replace `&` with `;` for AND conditions
- [ ] Replace comparison operators:
  - [ ] `__gt=` → `>`
  - [ ] `__gte=` → `>=`
  - [ ] `__lt=` → `<`
  - [ ] `__lte=` → `<=`
  - [ ] `__not=` → `!=`
- [ ] Replace text matching:
  - [ ] `__icontains=text` → `==*text*`
  - [ ] `__startswith=text` → `==text*`
  - [ ] `__endswith=text` → `==*text`
- [ ] Test all filter queries
- [ ] Update documentation/examples
- [ ] Update client libraries

---

## Code Examples - Before & After

### Python

**Before:**
```python
def get_active_jobs(env_id, inspector_id, api_key):
    params = {
        "filter": f"environmentId={env_id}&inspectorId={inspector_id}&status__in=pending,processing",
        "limit": 10
    }
    # ...
```

**After:**
```python
def get_active_jobs(env_id, inspector_id, api_key):
    filter_expr = f"environmentId=={env_id};inspectorId=={inspector_id};status=in=(pending,processing)"
    params = {
        "filter": filter_expr,
        "limit": 10
    }
    # ...
```

### JavaScript

**Before:**
```javascript
const filter = `environmentId=${envId}&inspectorId=${inspectorId}&status__in=pending,processing`;
```

**After:**
```javascript
const filter = `environmentId==${envId};inspectorId==${inspectorId};status=in=(pending,processing)`;
```

---

## Testing Your Migration

### 1. Simple Equality Test
```bash
# Test basic equality
curl "https://api.liongard.example.com/v3/jobs?filter=status==pending" \
  -H "X-API-Key: your-key"
```

### 2. AND Condition Test
```bash
# Test multiple conditions
curl "https://api.liongard.example.com/v3/jobs?filter=status==pending;type==dataprint_processing" \
  -H "X-API-Key: your-key"
```

### 3. IN Operator Test
```bash
# Test IN operator
curl "https://api.liongard.example.com/v3/jobs?filter=status=in=(pending,processing)" \
  -H "X-API-Key: your-key"
```

### 4. Comparison Test
```bash
# Test date comparison
curl "https://api.liongard.example.com/v3/jobs?filter=createdAt>=2024-01-01" \
  -H "X-API-Key: your-key"
```

### 5. Wildcard Test
```bash
# Test pattern matching
curl "https://api.liongard.example.com/v3/assets?filter=name==*server*" \
  -H "X-API-Key: your-key"
```

---

## Quick Reference Card

### RSQL Operators
```
==          Equal
!=          Not equal
>           Greater than
>=          Greater than or equal
<           Less than
<=          Less than or equal
=in=()      In list
=out=()     Not in list
==*text*    Contains
==prefix*   Starts with
==*suffix   Ends with
;           AND
,           OR
()          Group
```

### Common Patterns
```bash
# Single condition
field==value

# Multiple AND
field1==value1;field2==value2

# Multiple OR
field==value1,field==value2

# IN list
field=in=(value1,value2,value3)

# Date range
date>=2024-01-01;date<2024-02-01

# Complex
priority=in=(high,critical);(status==open,status==in_progress)
```

---

## Support

For questions or issues:
1. Check the **RSQL_FILTER_GUIDE.md** for comprehensive examples
2. Reference the RSQL specification at https://github.com/jirutka/rsql-parser
3. Test filters using the API sandbox environment
4. Contact support with specific filter examples that aren't working

---

## Summary

- **Cleaner syntax** - No more double underscores
- **Standard operators** - Uses `==`, `>`, `<`, etc.
- **Better readability** - `;` for AND, `,` for OR
- **Wildcards** - `*` for pattern matching
- **Well documented** - RSQL is a standard specification

The migration is straightforward - update your filter strings to use RSQL syntax and test!
