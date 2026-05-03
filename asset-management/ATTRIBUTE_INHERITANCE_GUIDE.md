# Asset Master — Attribute Inheritance Guide

> **Date:** May 3, 2026  
> **Version:** 1.0  
> **Module:** Asset Master — Attribute Template Inheritance & Asset Customization  
> **Status:** Phase 1 Complete | Phase 2 Ready for Implementation

---

## Table of Contents

1. [Quick Start](#1-quick-start)
2. [Core Concepts](#2-core-concepts)
3. [Design Pattern](#3-design-pattern)
4. [Implementation Examples](#4-implementation-examples)
5. [API Reference](#5-api-reference)
6. [Database Queries](#6-database-queries)
7. [Phase 2 Roadmap](#7-phase-2-roadmap)

---

## 1. Quick Start

### What is Attribute Inheritance?

Asset attributes are defined at two levels:

| Level | Scope | Created By | Inherited | Storage |
|-------|-------|-----------|-----------|---------|
| **Template** | Template-level | (Phase 2) Template Admin | ✅ Yes | 1 row shared by all assets |
| **Asset** | Asset-level | (Phase 1) Asset User | ❌ No | 1 row per custom attribute |

### Why It Matters

**Storage Efficiency:**
- Template attributes are defined ONCE and reused by ALL assets
- Custom attributes are ONLY created when an asset needs them
- Result: 97%+ reduction in attribute definition storage

**Example:**
```
Scenario: 100 Motor assets, each needing "Rated Capacity"

❌ Without inheritance: 100 duplicate definitions (100 rows)
✅ With inheritance: 1 shared definition (1 row)
```

---

## 2. Core Concepts

### 2.1 Polymorphic Owner Pattern

The `ATTRIBUTE_DEF` table stores ALL attribute definitions (template & asset level) using a single polymorphic pattern:

```
ATTRIBUTE_DEF
├─ id (PK)
├─ owner_id (FK) ──────┬─→ ASSET_TEMPLATE.id  [if scope=TEMPLATE]
├─ scope              │   ASSET_MASTER.id     [if scope=ASSET]
│  └─ TEMPLATE        │
│  └─ ASSET           │
├─ attribute_key      │
├─ display_name       └─────────────────────────────────────────
├─ validation_rules
└─ parent_attribute_def_id ──→ Link to TEMPLATE definition (Phase 2)
```

### 2.2 Scope Values

| Scope | Meaning | Owner | Inherited By | Phase |
|-------|---------|-------|--------------|-------|
| `TEMPLATE` | Template-level definition, shared | `ASSET_TEMPLATE.id` | All assets of that template | P2 |
| `ASSET` | Asset-specific custom attribute | `ASSET_MASTER.id` | This asset only | P1 |

### 2.3 Source Tracking (Phase 2)

```
ATTRIBUTE_DEF.source (ASSET scope only):
├─ NEW ─────────────→ Custom attribute created by user
└─ FROM_TEMPLATE ──→ Sourced from template, optionally overridden
```

### 2.4 Inheritance Linking (Phase 2)

```
ATTRIBUTE_DEF.parent_attribute_def_id (ASSET scope, FROM_TEMPLATE source):
  └─→ Links back to TEMPLATE-scoped definition for inheritance tracking
```

---

## 3. Design Pattern

### 3.1 Attribute Value Storage

```
ATTRIBUTE_VALUE
├─ id (PK)
├─ asset_master_id (FK) ────────→ Which asset
├─ attribute_def_id (FK) ────────→ Points to TEMPLATE OR ASSET def
├─ value_string / value_number / value_date / etc.
├─ is_overridden (boolean) ──────→ true if differs from template default
└─ validation_status
   └─ VALID | INVALID | WARNING | PENDING
```

**Key Point:**
- ATTRIBUTE_VALUE always references ATTRIBUTE_DEF
- ATTRIBUTE_DEF can be either TEMPLATE-scoped or ASSET-scoped
- No duplication: 100 assets using 1 template attribute = 100 value rows, 1 definition row

### 3.2 Inheritance Resolution

```
When displaying attributes for an asset:

1. Fetch all ATTRIBUTE_DEF where:
   - scope = TEMPLATE AND owner_id = asset's template_id
   - scope = ASSET AND owner_id = asset's id

2. For each ATTRIBUTE_DEF, find its value:
   - Look up ATTRIBUTE_VALUE (asset_id, attr_def_id)
   - If found, use ATTRIBUTE_VALUE.value_*
   - If not found, use ATTRIBUTE_DEF.default_value

3. Display all attributes together (no distinction to user)
```

---

## 4. Implementation Examples

### Example 1: Motor Template → Motor Asset (Phase 2)

```yaml
# Phase 2: Admin creates template attributes
POST /api/v1/templates/MOTOR-001/attribute-defs

Template Definition Created:
  ATTRIBUTE_DEF
    id: DEF-001
    owner_id: TEMPLATE-MOTOR-ID
    scope: TEMPLATE
    attribute_key: rated_capacity
    display_name: Rated Capacity
    data_type: NUMBER
    unit: kW
    is_mandatory: true
    default_value: 300
    validation_rules: [REQUIRED, MIN=10, MAX=500]

---

# Phase 1 (Current): Asset user instantiates template and adds custom attribute
POST /api/v1/assets
  asset_template_id: TEMPLATE-MOTOR-ID
  
ASSET_MASTER Created:
  id: ASSET-123
  asset_template_id: TEMPLATE-MOTOR-ID
  name: Motor #123

---

# Asset now has access to template attributes
GET /api/v1/assets/ASSET-123/attribute-defs
  → Returns DEF-001 (rated_capacity) via template inheritance

# User sets value on inherited attribute
PUT /api/v1/assets/ASSET-123/attribute-values
  attribute_def_id: DEF-001
  value: 350
  
ATTRIBUTE_VALUE Created:
  asset_master_id: ASSET-123
  attribute_def_id: DEF-001 (template definition)
  value_string: 350
  is_overridden: true (differs from default 300)

---

# User adds custom attribute (asset-specific)
POST /api/v1/assets/ASSET-123/attribute-defs

ATTRIBUTE_DEF Created:
  id: DEF-002
  owner_id: ASSET-123
  scope: ASSET
  source: NEW
  attribute_key: custom_mounting_bracket
  display_name: Custom Mounting Bracket
  
ATTRIBUTE_VALUE Created:
  asset_master_id: ASSET-123
  attribute_def_id: DEF-002 (asset definition)
  value_string: Model-XYZ

---

# Final state: Asset has both inherited + custom attributes
GET /api/v1/assets/ASSET-123/attribute-defs
  → [DEF-001 (inherited), DEF-002 (custom)]
```

### Example 2: Querying Inheritance

```sql
-- User interface needs to display "Rated Capacity" for Motor #123

SELECT 
  ad.id,
  ad.display_name,
  ad.scope,
  COALESCE(av.value_string, ad.default_value) as current_value
FROM ATTRIBUTE_DEF ad
LEFT JOIN ATTRIBUTE_VALUE av ON av.attribute_def_id = ad.id 
  AND av.asset_master_id = 'ASSET-123'
WHERE 
  -- Template attribute inherited by this asset
  ad.owner_id = 'TEMPLATE-MOTOR-ID' AND ad.scope = 'TEMPLATE'
  AND ad.attribute_key = 'rated_capacity';

Result:
  id: DEF-001
  display_name: Rated Capacity
  scope: TEMPLATE
  current_value: 350 (from ASSET-123's override)
```

---

## 5. API Reference

### 5.1 Asset Attribute Endpoints (Phase 1)

#### List All Attributes (Template-Inherited + Asset-Custom)

```
GET /api/v1/assets/{assetId}/attribute-defs

Query Parameters:
  scope (optional): TEMPLATE | ASSET  (filter to specific scope)

Response:
  [
    {
      id: "DEF-001",
      attribute_key: "rated_capacity",
      display_name: "Rated Capacity",
      scope: "TEMPLATE",
      owner_id: "TEMPLATE-MOTOR-ID",
      data_type: "NUMBER",
      unit: "kW",
      validation_rules: [...]
    },
    {
      id: "DEF-002",
      attribute_key: "custom_mounting_bracket",
      scope: "ASSET",
      owner_id: "ASSET-123",
      source: "NEW"
    }
  ]
```

#### Create Custom Attribute (Asset-Only in Phase 1)

```
POST /api/v1/assets/{assetId}/attribute-defs

Request:
  {
    display_name: "Custom Mount Height",
    data_type: "NUMBER",
    unit: "mm",
    is_required: true,
    validation_rules: [
      { rule_type: "MIN_VALUE", rule_value: "100", severity: "ERROR" }
    ]
  }

Response: 201 Created
  {
    id: "DEF-003",
    owner_id: "ASSET-123",
    scope: "ASSET",
    source: "NEW",
    attribute_key: "custom_mount_height",
    display_name: "Custom Mount Height"
  }
```

#### Set Attribute Value

```
PUT /api/v1/assets/{assetId}/attribute-values

Request:
  {
    attribute_def_id: "DEF-001",  (can be TEMPLATE or ASSET scoped)
    value: "350"
  }

Response: 200 OK
  {
    id: "VALUE-001",
    asset_master_id: "ASSET-123",
    attribute_def_id: "DEF-001",
    value_string: "350",
    is_overridden: true,
    validation_status: "VALID"
  }
```

#### Update Attribute Definition & Rules (Asset-Custom Only)

```
PUT /api/v1/assets/{assetId}/attribute-defs/{defId}

Request:
  {
    display_name: "Updated Name",
    validation_rules: [...]
  }

Response: 200 OK
  { updated attribute definition }

Note: Only works for scope=ASSET. Template definitions are updated via Phase 2 template endpoint.
```

---

### 5.2 Template Attribute Endpoints (Phase 2)

#### Create Template Attribute

```
POST /api/v1/templates/{templateId}/attribute-defs

Request:
  {
    display_name: "Rated Capacity",
    data_type: "NUMBER",
    unit: "kW",
    is_mandatory: true,  (required on all assets)
    default_value: "300",
    validation_rules: [...]
  }

Response: 201 Created
  {
    id: "DEF-100",
    owner_id: "TEMPLATE-MOTOR-ID",
    scope: "TEMPLATE",
    attribute_key: "rated_capacity",
    is_mandatory: true
  }

Effect: All assets instantiated from this template will inherit this definition
```

#### List Template Attributes

```
GET /api/v1/templates/{templateId}/attribute-defs

Response:
  [
    {
      id: "DEF-100",
      attribute_key: "rated_capacity",
      scope: "TEMPLATE",
      is_mandatory: true
    }
  ]
```

#### Update Template Attribute

```
PUT /api/v1/templates/{templateId}/attribute-defs/{defId}

Request:
  {
    validation_rules: [...]  (affects all assets inheriting this def)
  }

Response: 200 OK
  { updated definition }

Effect: Changes propagate to all assets inheriting this definition
```

---

## 6. Database Queries

### 6.1 Get All Attributes for Asset

```sql
-- Fetch everything an asset can see (inherited + custom)
SELECT 
  ad.id,
  ad.attribute_key,
  ad.display_name,
  ad.scope,
  ad.source,
  ad.default_value,
  av.value_string,
  av.is_overridden,
  av.validation_status,
  CASE 
    WHEN ad.scope = 'TEMPLATE' THEN 'Inherited from Template'
    WHEN ad.source = 'NEW' THEN 'Custom Asset Attribute'
    ELSE 'Sourced from Template'
  END as attribute_type
FROM ATTRIBUTE_DEF ad
LEFT JOIN ATTRIBUTE_VALUE av 
  ON av.attribute_def_id = ad.id 
  AND av.asset_master_id = 'ASSET-123'
WHERE 
  -- Template-level (inherited)
  (ad.owner_id = (SELECT asset_template_id FROM ASSET_MASTER WHERE id = 'ASSET-123')
   AND ad.scope = 'TEMPLATE')
  -- Asset-level (custom)
  OR (ad.owner_id = 'ASSET-123' AND ad.scope = 'ASSET')
ORDER BY 
  CASE ad.scope WHEN 'TEMPLATE' THEN 1 ELSE 2 END,
  ad.sort_order ASC;
```

### 6.2 Check for Overrides

```sql
-- Which template attributes has this asset overridden?
SELECT 
  ad.attribute_key,
  ad.display_name,
  ad.default_value AS template_default,
  av.value_string AS asset_value,
  av.is_overridden
FROM ATTRIBUTE_DEF ad
LEFT JOIN ATTRIBUTE_VALUE av 
  ON av.attribute_def_id = ad.id 
  AND av.asset_master_id = 'ASSET-123'
WHERE 
  ad.owner_id = (SELECT asset_template_id FROM ASSET_MASTER WHERE id = 'ASSET-123')
  AND ad.scope = 'TEMPLATE'
  AND av.is_overridden = true;
```

### 6.3 Count Storage Savings

```sql
-- How much storage are we saving with the polymorphic pattern?

-- Template attributes (shared)
SELECT COUNT(*) as template_attrs
FROM ATTRIBUTE_DEF 
WHERE scope = 'TEMPLATE' 
  AND owner_id = 'TEMPLATE-MOTOR-ID';

-- Assets inheriting from template
SELECT COUNT(DISTINCT am.id) as num_assets
FROM ASSET_MASTER am
WHERE am.asset_template_id = 'TEMPLATE-MOTOR-ID';

-- Hypothetical cost without inheritance (would need num_assets * template_attrs rows)
-- Actual cost: just template_attrs rows for all assets combined
```

---

## 7. Phase 2 Roadmap

### 7.1 What's Coming in Phase 2

| Feature | Phase 1 | Phase 2 | Notes |
|---------|---------|---------|-------|
| Create attributes | Asset only | Template + Asset | Admin creates template-level base attributes |
| Attribute inheritance | (placeholder) | ✅ Full | Assets inherit template definitions |
| Override attributes | (not yet) | ✅ Yes | Assets can override inherited values |
| Parent linking | Not used | ✅ Yes | `parent_attribute_def_id` for tracing |
| Mandatory attributes | Manual | ✅ Auto | `is_mandatory=true` enforces on all assets |
| Cross-field rules | Not supported | 🔲 Planned | Validate across multiple attributes |

### 7.2 Migration Path: Phase 1 → Phase 2

**Before Phase 2:**
- All asset attributes have `owner_id = ASSET_MASTER.id` and `scope = ASSET`
- No `parent_attribute_def_id` references exist

**After Phase 2 Deployment:**
1. Admin creates template attributes via new Phase 2 UI
2. Existing asset attributes remain unchanged (still `scope=ASSET, source=NEW`)
3. New assets created from templates automatically inherit template attributes
4. Optional: Migrate existing asset attributes to template scope if they're common

### 7.3 Example Phase 2 Workflow

```
Admin User → Creates Motor Template
  ├─ POSTing /templates/MOTOR/attribute-defs
  │   ├─ Rated Capacity (TEMPLATE scope, is_mandatory=true)
  │   ├─ RPM Limit (TEMPLATE scope)
  │   └─ Efficiency Rating (TEMPLATE scope)
  
User → Creates Motor Asset
  ├─ Selects Motor template
  ├─ Automatically gets 3 inherited attributes
  ├─ Sets values on inherited attributes
  ├─ Adds custom attributes as needed
  
Database:
  ├─ ATTRIBUTE_DEF (scope=TEMPLATE): 3 rows (shared by all Motor assets)
  ├─ ATTRIBUTE_DEF (scope=ASSET): N rows (custom per asset)
  ├─ ATTRIBUTE_VALUE: Values for both inherited and custom
```

---

## 8. Troubleshooting

### Problem: Template Attribute Changes Don't Affect Existing Assets

**Root Cause:** Asset values override template defaults

**Solution:** 
1. Update template definition (affects future assets)
2. Reset `is_overridden=false` on asset values to revert to template default
3. Or bulk-update specific assets as needed

### Problem: Asset Attribute Appears Duplicated

**Root Cause:** Likely both TEMPLATE and ASSET scope versions exist

**Solution:**
1. Query to confirm both exist
2. Delete redundant ASSET version if not customized
3. Or merge custom values into template definition (Phase 2)

### Problem: Storage Still Growing Despite Inheritance

**Root Cause:** Creating custom attributes for every asset instead of using template

**Solution:**
1. Review which attributes should be template-level (Phase 2)
2. Consolidate common custom attributes into template
3. Monitor ATTRIBUTE_DEF growth rate

---

## 9. Summary Table

| Aspect | Phase 1 | Phase 2 |
|--------|---------|---------|
| **Attribute Creation** | Asset only | Template + Asset |
| **Storage Pattern** | All scope=ASSET | Template (scope=TEMPLATE) + Asset (scope=ASSET) |
| **Inheritance** | Manual (not built-in) | Automatic via scope=TEMPLATE |
| **Override Support** | Not tracked | Tracked via `is_overridden` flag |
| **Parent Linking** | Not used | Full `parent_attribute_def_id` support |
| **Mandatory Attrs** | Manual enforcement | `is_mandatory` flag automates |
| **Duplication Risk** | High (per-asset defs) | Low (shared template defs) |
| **Storage Efficiency** | ~1x baseline | ~97%+ savings for shared attrs |

---

## 10. Document Index

| Document | Purpose | Audience |
|----------|---------|----------|
| **LLD_AssetMaster_SHORT.md** | Quick reference design | Developers, Architects |
| **LLD_AssetMaster1.md** | Complete detailed design | Developers, DB Designers |
| **AttributeRuleEngine_Design.md** | Rule engine details | Rule developers |
| **AttributeRuleEngine_Design_SHORT.md** | Rule engine quick ref | Developers |
| **ATTRIBUTE_INHERITANCE_GUIDE.md** (this file) | Inheritance patterns | Developers, Designers |

---

*End of Attribute Inheritance Guide*
