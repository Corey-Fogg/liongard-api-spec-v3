# Liongard Vendor API v3 - Design History & Rationale

## Executive Summary

Liongard Vendor API v3 is the **first external-facing API** built specifically for vendor integrations. This is not a v1/v2 rewrite - it's a ground-up design that learned from internal API limitations while establishing a stable, production-grade contract for external partners.

**Why v3 Exists:**
- v1/v2 were internal APIs optimized for Liongard's UI
- Vendors need different patterns (async, webhooks, filtering)
- External APIs require stability contracts internal APIs don't need
- Vendor use cases differ fundamentally from UI use cases

**Key Differentiators:**
- ✅ Async processing (non-blocking)
- ✅ Proper HTTP semantics (headers for metadata)
- ✅ Standard filtering (RSQL)
- ✅ Webhook-first design
- ✅ No wrapper objects
- ✅ Production-grade patterns

---

## Design Evolution: How We Got Here

### Phase 1: Initial Vision - "Standard REST + Webhooks"

**Goal:** Define a clean REST API standard supporting both endpoints and webhooks.

**Initial Requirements:**
- Group data by Environments
- Asset Inventory and Inspector data
- Support pagination, filtering, expand
- Webhooks with full payloads and diffs

**Outcome:** Established the foundation but needed concrete conventions.

---

### Phase 2: Core Conventions Locked Down

**Key Decisions Made:**

#### Naming Convention: camelCase
```
✅ environmentId, assetType, createdAt
❌ environment_id, asset_type, created_at
```

**Rationale:** JavaScript-friendly, consistent with modern REST APIs.

#### Pagination Model
```
Query params: ?limit=100&offset=0
Response: In body (initially)
```

**Later Changed:** Moved to HTTP headers for proper REST compliance.

#### Filtering Philosophy
```
✅ Filter by model fields only
✅ Explicit field references
❌ No magic query parsing
❌ No expand-all wildcards
```

**Rationale:** Predictable, version-safe, discoverable.

#### Webhook Scoping
Webhooks must support conditional expressions:
```
EnvironmentId = 15243 AND AssetType = Workstation
```

**Impact:** Forced us to create a unified filtering model for both REST and webhooks. This became RSQL.

**Critical Decision:** Align REST filtering and webhook scoping to the same expression language.

---

### Phase 3: Alerts Became Global

**Problem:** Should alerts be per-environment or global?

**Internal Reality:** Alerts are tied to environments in Liongard's data model.

**Vendor Reality:** Vendors want to see ALL alerts across ALL environments.

**Decision:**
```
✅ GET /v3/alerts (global, query all)
✅ Filter by environmentId using where/filter
✅ Environment-scoped routes optional
```

**Rationale:** 
- Vendors integrate at the partner level, not per customer
- Filtering is easier than multiple API calls
- Aligns with webhook patterns (global subscriptions)

**Schema Alignment:**
- Based on v1 Task schema
- Removed "Task" terminology publicly
- Kept field parity without internal coupling

---

### Phase 4: Ingestion Integrated into Inspectors

**Original Exploration:**
1. Top-level ingestion endpoint `/v3/ingestion`
2. Separate ingestion category
3. Dataprint-based approach

**Problem Identified:**
- Dataprints are tied to Inspectors
- Inspectors are tied to Environments
- Therefore ingestion **must** be nested

**Final Decision:**
```
POST /v3/environments/{environmentId}/inspectors/{inspectorId}/dataprints
```

**Structure:**
```
/v3/environments/{envId}/inspectors/{inspectorId}/
  ├── GET /config          # Get mapping configuration
  ├── PUT /config          # Set mapping configuration
  ├── POST /dataprints     # Push data
  └── GET /dataprints/{id} # Get dataprint result
```

**Rationale:**
- Natural hierarchy (environment → inspector → data)
- Configuration and ingestion are tightly coupled
- Prevents orphaned dataprints
- Clear ownership model

**Major Impact:** This unified ingestion, inspector metadata, and data retrieval into a single coherent model.

---

### Phase 5: Expand Complexity Solved

**Problem Discovered:**
```
❌ expand: enum[installedSoftware, policies, rules]
```

**Why This Fails:**
- Workstations have `installedSoftware`
- Firewalls have `policies`
- Servers have different arrays
- Asset types have different expandable fields

**Attempted Solution 1: Expand-all wildcard**
```
?expand=*
```
**Rejected:** Too expensive, unpredictable, version-unsafe.

**Final Solution: Dynamic Discovery**
```
✅ GET /v3/meta/objects             # Discover expandable fields per object
✅ GET /v3/meta/assets/types        # Discover asset-type-specific fields
✅ expand=field1,field2             # Explicit strings
✅ strictExpand=true                # Fail on invalid expands
```

**Rationale:**
- Prevents spec brittleness
- Self-documenting
- Version-safe (new fields don't break old clients)
- Enables runtime discovery

**Example:**
```python
# Discover what's expandable
meta = get("/v3/meta/objects")
alert_expandable = meta["objects"]["alert"]["expandableFields"]
# ["environment", "inspector", "asset"]

# Use it
alerts = get("/v3/alerts?expand=environment,inspector")
```

---

### Phase 6: POST Standardization

**Goal:** Production-grade conventions for resource creation.

**Changes:**
1. POST responses: `200` → `201 Created`
2. Standardized error handling
3. Removed ingestion duplication
4. Cleaned alphabetical structure
5. Removed unnecessary verbosity

**Before:**
```json
POST /v3/alerts
200 OK
{
  "data": {
    "alertId": "alert_123"
  }
}
```

**After:**
```json
POST /v3/alerts
201 Created
Location: /v3/alerts/alert_123
{
  "alertId": "alert_123",
  "name": "Critical Alert",
  "status": "open"
}
```

**Rationale:** Standard HTTP semantics, Location header for new resource, direct object (no wrapper).

---

### Phase 7: Environment and Alert Creation

**Requirements:**
- Vendors need to create environments
- Vendors need to create alerts (pass-through from their systems)

**Decisions:**
```
✅ POST /v3/environments
✅ POST /v3/alerts
```

**Schema Strategy:**
- Use v1/v2 field definitions as baseline
- Keep v3-clean naming (camelCase)
- Avoid "Task" terminology publicly
- Preserve field parity without coupling to internals

**Example:**
```
Internal: Task.DueDate
v1/v2: DueDate
v3: dueDate
```

---

### Phase 8: Webhooks as First-Class Citizens

**Philosophy:** Webhooks are not secondary. They're a primary integration method.

**Design Principles:**
1. **Object-scoped subscriptions**
   ```json
   {
     "url": "https://vendor.com/webhook",
     "events": ["alert.created", "alert.updated"],
     "filter": "priority==critical;status==open",
     "expand": "environment,inspector"
   }
   ```

2. **Per-object filters**
   - Same RSQL syntax as REST
   - Evaluated before delivery
   - Reduces webhook noise

3. **Per-object expand**
   - Related data embedded
   - Reduces follow-up API calls

4. **Delivery modes:**
   - `full` - Complete object
   - `diff` - Only changed fields
   - `none` - Just notification

5. **Snapshot behavior**
   - For diffs, includes before/after
   - Enables change tracking
   - Powers audit trails

**Symmetry:**
```
REST:    GET /v3/alerts?filter=priority==critical&expand=environment
Webhook: POST /v3/webhooks/subscriptions
         { "events": ["alert.created"], 
           "filter": "priority==critical",
           "expand": "environment" }
```

**Result:** Unified filtering model across REST and webhooks.

---

## Major Structural Decisions

### 1. Terminology: "Dataprint" not "Ingestion"

**Old (v1/v2):** Ingestion, ingestion endpoint, ingested data  
**New (v3):** Dataprint, dataprint endpoint, dataprint data

**Rationale:**
- "Ingestion" is internal jargon
- "Dataprint" is customer-facing term
- Aligns with Liongard's product terminology
- More intuitive for vendors

### 2. Async Processing Pattern

**v1/v2 Pattern:**
```
POST /v1/dataprints
[waits 30-120 seconds]
200 OK
{ "accepted": true, "assetsProcessed": 5 }
```

**v3 Pattern:**
```
POST /v3/.../dataprints
[returns immediately]
202 Accepted
{ "jobId": "job_123", "status": "pending", "pollUrl": "..." }

GET /v3/jobs/job_123
200 OK
{ "status": "completed", "result": {...} }
```

**Rationale:**
- Non-blocking for vendors
- Better error handling
- Progress tracking
- Timeout-free
- Webhook-compatible

### 3. Response Format: No Wrappers

**v1/v2 Pattern:**
```json
{
  "data": {
    "alertId": "alert_123"
  },
  "pagination": {
    "limit": 100,
    "offset": 0
  }
}
```

**v3 Pattern:**

**Single Resource:**
```json
{
  "alertId": "alert_123",
  "name": "Critical Alert"
}
```

**List:**
```json
[
  {"alertId": "alert_1"},
  {"alertId": "alert_2"}
]
```

**Headers:**
```
X-Pagination-Limit: 100
X-Pagination-Offset: 0
X-Pagination-Count: 547
X-Pagination-Has-More: true
Link: <url>; rel="next"
```

**Rationale:**
- Proper HTTP semantics
- Smaller payloads
- Better caching
- Standard Link header (RFC 5988)

### 4. Filtering: RSQL Standard

**v1/v2 Pattern:**
```
?status__in=open,in_progress&priority__gte=high
```

**v3 Pattern:**
```
?filter=status=in=(open,in_progress);priority>=high
```

**Operators:**
- `==` Equal
- `!=` Not equal
- `>` `>=` `<` `<=` Comparisons
- `=in=()` In list
- `=out=()` Not in list
- `;` AND
- `,` OR
- `*` Wildcards

**Rationale:**
- Industry standard (RSQL)
- More readable
- No weird underscores
- Library support
- Works in URLs

### 5. Concurrency Control

**Rule:** Only ONE dataprint job can process at a time per inspector per environment.

**Enforcement:**
```
POST /v3/.../dataprints (while job running)
409 Conflict
{
  "error": {
    "code": "CONCURRENT_JOB_CONFLICT",
    "existingJobId": "job_xyz",
    "suggestion": "Cancel using DELETE /v3/jobs/job_xyz"
  }
}
```

**Rationale:**
- Prevents data races
- Ensures correct change detection
- Maintains data integrity
- Clear error handling

---

## Unified Design Philosophy

### 1. REST and Webhooks Share Syntax

**Filtering:**
```
REST:    ?filter=status==open;priority==critical
Webhook: "filter": "status==open;priority==critical"
```

**Expansion:**
```
REST:    ?expand=environment,inspector
Webhook: "expand": "environment,inspector"
```

**Result:** Learn once, use everywhere.

### 2. Discoverable at Runtime

**Meta Endpoints:**
```
GET /v3/meta/objects          # What fields exist?
GET /v3/meta/assets/types     # What asset types exist?
```

**Benefit:** Clients can build dynamic UIs without hardcoding.

### 3. Hierarchical Resource Model

```
/v3/
├── environments/
│   ├── {envId}/
│   │   ├── inspectors/
│   │   │   ├── {inspectorId}/
│   │   │   │   ├── config
│   │   │   │   ├── dataprints/
│   │   │   │   └── jobs/
│   │   ├── assets/
│   │   └── systems/
├── alerts/
├── metrics/
├── webhooks/
└── jobs/
```

**Rationale:**
- Natural hierarchy
- Clear ownership
- Logical nesting
- RESTful

### 4. Separation of Concerns

**Inspector Definition (Global):**
```
POST /v3/inspectors
{
  "name": "acme-monitor",
  "displayName": "Acme Monitoring"
}
```

**Inspector Configuration (Per Environment):**
```
PUT /v3/environments/{envId}/inspectors/{inspectorId}/config
{
  "metadata": {
    "dataStructure": "split",
    "environments": {...},
    "assets": {...}
  }
}
```

**Rationale:**
- Define inspector once
- Configure per customer
- Reuse definitions
- Version independently

---

## Why v3 is Better Than v1/v2

### For Vendors

| Aspect | v1/v2 | v3 |
|--------|-------|-----|
| **Blocking** | Sync (30-120s wait) | Async (immediate return) |
| **Timeouts** | Frequent | None (job-based) |
| **Progress** | None | Real-time tracking |
| **Webhooks** | Limited | Full-featured |
| **Filtering** | Complex syntax | Standard RSQL |
| **Errors** | Generic | Detailed + actionable |
| **Concurrency** | Race conditions | Controlled + clear errors |
| **Response Size** | Wrapped objects | Direct + smaller |

### For Liongard

| Aspect | v1/v2 | v3 |
|--------|-------|-----|
| **Versioning** | Coupled to UI | Independent |
| **Breaking Changes** | Impact UI | Isolated to vendors |
| **Documentation** | Internal | Production-grade |
| **Support** | Ticket-heavy | Self-service |
| **Monitoring** | Limited | Job tracking built-in |
| **Rate Limiting** | Shared with UI | Vendor-specific |

---

## Key Design Principles

### 1. Vendor-First Thinking

**Question:** What do vendors need?
- Bulk operations (not one-at-a-time)
- Async processing (don't block)
- Webhooks (push, don't poll)
- Filtering (query what you need)
- Metrics (extract insights)

**Result:** Every feature designed for integration use cases.

### 2. Production-Grade Patterns

**Standards:**
- OpenAPI 3.1 compliant
- RFC 5988 Link headers
- RSQL filtering standard
- HTTP status codes correct
- Error responses actionable

**Result:** Familiar to developers, library-compatible.

### 3. Self-Documenting

**Discovery:**
- Meta endpoints for runtime discovery
- Examples in OpenAPI spec
- Comprehensive guide
- Interactive Swagger UI

**Result:** Developers can explore without support.

### 4. Version-Safe

**Strategies:**
- Explicit field expansion (no wildcards)
- Strict validation optional
- Additive changes safe
- No implicit behaviors

**Result:** New features don't break old clients.

### 5. Opinionated but Flexible

**Opinionated:**
- RSQL for filtering (not custom syntax)
- Async for long operations (not sync)
- Headers for metadata (not body)
- Direct responses (no wrappers)

**Flexible:**
- Multiple pagination methods (offset, Link header)
- Multiple job tracking methods (poll, callback, webhook)
- Multiple data structures (tiered, split)

**Result:** Best practices built-in, escape hatches available.

---

## Evolution Summary

### The Journey in Phases

1. **Generic REST + Webhooks** → Established vision
2. **Core Conventions** → Locked naming, pagination, filtering
3. **Alerts Global** → Vendor use case drove design
4. **Ingestion Under Inspectors** → Natural hierarchy discovered
5. **Dynamic Expand** → Prevented brittleness
6. **POST Standardization** → Production patterns
7. **Resource Creation** → Vendor pass-through enabled
8. **Webhooks First-Class** → Unified with REST

### Key Pivots

**Pivot 1: Pagination**
- Started in body → Moved to headers
- **Why:** HTTP standards, better caching

**Pivot 2: Filtering**
- Started with custom syntax → Adopted RSQL
- **Why:** Industry standard, better docs

**Pivot 3: Ingestion**
- Started separate → Moved under Inspectors
- **Why:** Natural hierarchy, clear ownership

**Pivot 4: Expand**
- Started with enums → Made dynamic + discoverable
- **Why:** Version-safe, asset types vary

**Pivot 5: Terminology**
- Started with "ingestion" → Changed to "dataprint"
- **Why:** Customer-facing, product alignment

### Decisions That Stuck

✅ camelCase naming  
✅ Async processing  
✅ Job-based tracking  
✅ RSQL filtering  
✅ No wrapper objects  
✅ Headers for metadata  
✅ Webhooks with same filter syntax  
✅ Hierarchical resources  
✅ Meta discovery endpoints  

---

## Comparison: v1/v2 vs v3

### Response Format

**v1/v2:**
```json
{
  "data": {
    "ID": 123,
    "Name": "Alert"
  }
}
```

**v3:**
```json
{
  "alertId": "alert_123",
  "name": "Alert"
}
```

### Pagination

**v1/v2:**
```
?page=1&pageSize=50
Response body: { data: [...], pagination: {...} }
```

**v3:**
```
?limit=50&offset=0
Response headers: X-Pagination-Limit, X-Pagination-Count, Link
Response body: [...]
```

### Filtering

**v1/v2:**
```
?conditions[]={"path":"Status","op":"eq","value":"active"}
```

**v3:**
```
?filter=status==active
```

### Dataprint Submission

**v1/v2:**
```
POST /v1/dataprints
[waits 30-120 seconds]
200 OK
{ "accepted": true }
```

**v3:**
```
POST /v3/.../dataprints
[immediate]
202 Accepted
{ "jobId": "job_123", "pollUrl": "..." }

GET /v3/jobs/job_123
200 OK
{ "status": "completed", "result": {...} }
```

### Identifiers

**v1/v2:**
```json
{ "ID": 1234 }
```

**v3:**
```json
{ "alertId": "alert_1234" }
```

---

## Future-Proofing

### What's Stable

✅ Resource models (alert, asset, environment)  
✅ URL structure (/v3/resource/{id})  
✅ RSQL filtering syntax  
✅ Async job pattern  
✅ Response format (no wrappers)  
✅ Header-based pagination  

### What Can Evolve

✅ New fields (additive)  
✅ New resource types  
✅ New webhook events  
✅ New metric queries  
✅ New asset types  

### Version Strategy

**v3.x.x:**
- x.x.x = Patch (bugs)
- x.X.x = Minor (new features, backward compatible)
- X.x.x = Major (breaking changes)

**Breaking Change Examples:**
- Removing fields
- Changing field types
- Changing URL structure
- Changing error codes

**Non-Breaking Examples:**
- Adding fields
- Adding endpoints
- Adding webhook events
- Adding asset types

---

## Lessons Learned

### 1. Start with Use Cases

**Don't:** Design API from data model  
**Do:** Design API from vendor workflows

**Example:** Vendors want all alerts globally, not per-environment queries.

### 2. Standards Over Custom

**Don't:** Invent custom syntax  
**Do:** Use industry standards (RSQL, RFC 5988)

**Example:** RSQL has docs, parsers, and developer familiarity.

### 3. Async is Essential

**Don't:** Block HTTP connections for long operations  
**Do:** Return immediately with job ID

**Example:** Dataprint processing takes 30-120s. Don't make vendors wait.

### 4. Webhooks = Events + Filters

**Don't:** Send all events, let vendors filter client-side  
**Do:** Filter server-side based on subscription

**Example:** Only send critical alerts, not every alert.

### 5. Discovery Prevents Brittleness

**Don't:** Hardcode expandable fields  
**Do:** Provide meta endpoints for runtime discovery

**Example:** New asset types don't require API version bump.

### 6. Hierarchy Matters

**Don't:** Flat resource structure  
**Do:** Logical nesting based on ownership

**Example:** Dataprints belong to inspectors, inspectors belong to environments.

### 7. No Wrapper Objects

**Don't:** `{ data: {...} }`  
**Do:** Direct objects/arrays

**Example:** Simpler client code, smaller payloads, better HTTP semantics.

### 8. Metadata in Headers

**Don't:** `{ data: [...], pagination: {...} }`  
**Do:** Response body for content, headers for metadata

**Example:** Proper HTTP design, better caching.

---

## Success Criteria

### For Vendors

✅ Can integrate in < 1 day  
✅ Don't need support for common tasks  
✅ Can discover capabilities at runtime  
✅ Never block on long operations  
✅ Get real-time updates via webhooks  
✅ Can filter to exactly what they need  
✅ Have working code examples  

### For Liongard

✅ Versioned independently from UI  
✅ Breaking changes don't affect internal systems  
✅ Rate limiting per vendor  
✅ Job tracking for debugging  
✅ Self-service documentation  
✅ Webhook delivery monitoring  
✅ Clear error messages reduce tickets  

---

## Conclusion

Liongard Vendor API v3 represents a complete rethinking of how external integrations work. By starting from vendor use cases rather than internal data models, and by applying modern REST and async patterns, we've created an API that is:

- **Fast** - Non-blocking, async
- **Flexible** - RSQL filtering, dynamic expansion
- **Discoverable** - Meta endpoints, OpenAPI spec
- **Standard** - HTTP semantics, industry patterns
- **Stable** - Version-safe, clear contracts
- **Powerful** - Webhooks, metrics, jobs

This is not just "v1/v2 with better docs" - it's a fundamentally different approach designed for the way vendors actually build integrations.

**The result:** An API that vendors want to use, not one they have to use.
