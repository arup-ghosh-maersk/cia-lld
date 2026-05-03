# Attribute Inheritance — Visual Quick Reference

> **Date:** May 3, 2026  
> **Version:** 1.0  
> **Purpose:** Visual guide to template attribute inheritance pattern

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Asset Master System                      │
│                 (Event-Driven Architecture)                 │
└─────────────────────────────────────────────────────────────┘

    ┌──────────────┐                    ┌──────────────┐
    │ ASSET        │                    │ ASSET        │
    │ TEMPLATE     │──────┬─────────────│ MASTER       │
    │              │      │             │              │
    │ ┌──────────┐ │      │             │ ┌──────────┐ │
    │ │ Attr Defs│ │      │ inherited   │ │ Attr     │ │
    │ │(Template)│ │◄─────┤ ──────────► │ │ Values   │ │
    │ │          │ │      │             │ │          │ │
    │ └──────────┘ │      │             │ └──────────┘ │
    │              │      │             │              │
    │              │      │             │ ┌──────────┐ │
    │              │      │             │ │ Custom   │ │
    │              │      └─────────────│ │ Attr Defs│ │
    │              │                    │ │(Asset)   │ │
    │              │                    │ └──────────┘ │
    └──────────────┘                    └──────────────┘
         (1 per template)           (N assets per template)
         (defines once)              (inherit + customize)
```

---

## 2. Data Flow Diagram

```
┌────────────────────────────────────────────────────────────────┐
│ ASSET_TEMPLATE "Motor"                                         │
│ id = TEMPLATE-MOTOR-ID                                         │
└────────────────────────────────────────────────────────────────┘
                              │
                              │ defines
                              ↓
┌────────────────────────────────────────────────────────────────┐
│ ATTRIBUTE_DEF (Shared)                                         │
│ ┌─────────────────────────────────────────────────────────┐   │
│ │ id: DEF-001                                             │   │
│ │ owner_id: TEMPLATE-MOTOR-ID                             │   │
│ │ scope: TEMPLATE                     ← One definition    │   │
│ │ attribute_key: "rated_capacity"                         │   │
│ │ default_value: "300"                                    │   │
│ │ validation_rules: [REQUIRED, MIN=10, MAX=500]           │   │
│ │ is_mandatory: true                                      │   │
│ └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
                              │
                  ┌───────────┼───────────┬──────────────┐
                  │           │           │              │
        inherited by each asset:
                  │           │           │              │
                  ↓           ↓           ↓              ↓
        ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────────┐
        │ASSET #1     │ │ASSET #2     │ │ASSET #3     │ │ASSET #N      │
        │(Motor A)    │ │(Motor B)    │ │(Motor C)    │ │(Motor N)     │
        └─────────────┘ └─────────────┘ └─────────────┘ └──────────────┘
             │                │                │              │
             ├─ ATTRIBUTE_VALUE (owns DEF-001 reference)
             │  value: "250"
             │  is_overridden: false
             │
             ├─ ATTRIBUTE_VALUE (for custom attr)
             │  attribute_def_id: DEF-002
             │  value: "custom_value"
             │
             └─ ATTRIBUTE_DEF (Custom, scope=ASSET)
                id: DEF-002
                owner_id: ASSET-001
                source: NEW

            ↓ Each asset can:
              • Use inherited template value as-is
              • Override inherited value
              • Add custom attributes
              • Set values on all of the above
```

---

## 3. State Diagram: Attribute Lifecycle

```
                    PHASE 1 (Current)
                    ─────────────────

    ┌──────────────────────────────────────────────────────┐
    │ User creates ASSET from template                     │
    │ POST /api/v1/assets { asset_template_id: ... }       │
    └──────────────────────────────────────────────────────┘
                             │
                             ↓
    ┌──────────────────────────────────────────────────────┐
    │ Asset inherits TEMPLATE attributes automatically     │
    │ (no explicit action needed)                          │
    │                                                      │
    │ GET /api/v1/assets/{assetId}/attribute-defs          │
    │   → Returns all TEMPLATE-scoped definitions          │
    │     from parent template                            │
    └──────────────────────────────────────────────────────┘
                             │
                    ┌────────┼────────┐
                    │                 │
                    ↓                 ↓
    ┌──────────────────────────┐  ┌──────────────────────────┐
    │ Set value on inherited   │  │ Create custom attribute  │
    │ template attribute       │  │ (asset-specific)        │
    │                          │  │                         │
    │PUT /api/v1/.../attr-val  │  │POST /api/v1/.../attr-   │
    │  def_id: DEF-001         │  │ defs (scope=ASSET)       │
    │  value: "350"            │  │                         │
    │                          │  │ → ATTRIBUTE_DEF with    │
    │→ ATTRIBUTE_VALUE created │  │   owner=ASSET_ID        │
    │  is_overridden: true     │  │   source: NEW           │
    └──────────────────────────┘  └──────────────────────────┘
                    │                     │
                    └────────────┬────────┘
                                 ↓
    ┌──────────────────────────────────────────────────────┐
    │ Asset now has both:                                  │
    │ • Inherited TEMPLATE attributes (with values)        │
    │ • Custom ASSET attributes (unique to this asset)     │
    │                                                      │
    │ GET /api/v1/assets/{assetId}/attribute-defs          │
    │   → Returns BOTH scopes in single list               │
    └──────────────────────────────────────────────────────┘


                    PHASE 2 (Future)
                    ────────────────

    ┌──────────────────────────────────────────────────────┐
    │ Admin creates TEMPLATE attributes                    │
    │ POST /api/v1/templates/{templateId}/attribute-defs   │
    │                                                      │
    │ → ATTRIBUTE_DEF with:                                │
    │   owner_id: TEMPLATE_ID                              │
    │   scope: TEMPLATE                                    │
    │   is_mandatory: true/false                           │
    └──────────────────────────────────────────────────────┘
                             │
                             ↓
    ┌──────────────────────────────────────────────────────┐
    │ New assets created from template                     │
    │ automatically inherit those definitions              │
    │                                                      │
    │ (Same flow as Phase 1 from here on)                  │
    └──────────────────────────────────────────────────────┘
```

---

## 4. Attribute Scope Hierarchy

```
                    ATTRIBUTE_DEF
                    (Single Table)
                           │
                ┌──────────┴──────────┐
                │                     │
                ↓                     ↓
        ┌─────────────────┐  ┌──────────────────┐
        │ TEMPLATE Scope  │  │ ASSET Scope      │
        │ (shared)        │  │ (per-asset)      │
        ├─────────────────┤  ├──────────────────┤
        │ owner_id =      │  │ owner_id =       │
        │ TEMPLATE_ID     │  │ ASSET_ID         │
        │                 │  │                  │
        │ Inherited by:   │  │ Visible to:      │
        │ All assets of   │  │ This asset only  │
        │ that template   │  │                  │
        │                 │  │ source:          │
        │ is_mandatory:   │  │ NEW              │
        │ true = required │  │ FROM_TEMPLATE*   │
        │ on all assets   │  │ (*Phase 2)       │
        │                 │  │                  │
        │ Stored once,    │  │ Each custom attr │
        │ used by many    │  │ stored once      │
        └─────────────────┘  └──────────────────┘
                │                     │
                │ referenced by       │ referenced by
                │                     │
                ↓                     ↓
        ┌─────────────────────────────────────┐
        │    ATTRIBUTE_VALUE                  │
        │                                     │
        │ asset_master_id ──→ Which asset    │
        │ attribute_def_id ──→ Which def     │
        │ value_* ──────────→ The value      │
        │ is_overridden ────→ Differs from  │
        │                    template def    │
        │ validation_status → VALID/INVALID  │
        └─────────────────────────────────────┘
```

---

## 5. SQL Relationship Pattern

```
                TEMPLATE-Level
                ════════════════

ASSET_TEMPLATE
    ├─ id: TEMPLATE-MOTOR
    └─ ...

    ATTRIBUTE_DEF (scope=TEMPLATE, owner=TEMPLATE-MOTOR)
        ├─ id: DEF-001
        ├─ attribute_key: "rated_capacity"
        ├─ default_value: "300"
        └─ is_mandatory: true

        ATTRIBUTE_VALUE (via DEF-001 → any asset)
            ├─ asset_master_id: ASSET-001
            ├─ value: "250"
            └─ is_overridden: true

            ├─ asset_master_id: ASSET-002
            ├─ value: "350"
            └─ is_overridden: true

            ├─ asset_master_id: ASSET-003
            ├─ value: NULL (uses template default)
            └─ is_overridden: false

                    ASSET-Level
                    ═══════════

ASSET_MASTER
    ├─ id: ASSET-001
    ├─ asset_template_id: TEMPLATE-MOTOR
    └─ ...

    ATTRIBUTE_DEF (scope=ASSET, owner=ASSET-001, source=NEW)
        ├─ id: DEF-002
        ├─ attribute_key: "custom_mount_height"
        └─ validation_rules: [MIN_VALUE: 100]

        ATTRIBUTE_VALUE (via DEF-002 → ASSET-001 only)
            ├─ value: "150"
            └─ validation_status: VALID
```

---

## 6. Three Inheritance Patterns

```
Pattern A: Pure Inheritance
──────────────────────────

Template: rated_capacity (default: 300, TEMPLATE scope)
Asset:    Uses inherited, no override
Result:   ATTRIBUTE_VALUE.is_overridden = FALSE
Display:  "300" (uses template default)


Pattern B: Inheritance + Override
──────────────────────────────

Template: rated_capacity (default: 300, TEMPLATE scope)
Asset:    Inherits but sets value to 350
Result:   ATTRIBUTE_VALUE.is_overridden = TRUE
          ATTRIBUTE_VALUE.value_string = "350"
Display:  "350" (asset override)


Pattern C: Custom Asset Attribute
──────────────────────────────

Template: rated_capacity (TEMPLATE scope)
Asset:    Adds NEW custom_mount_height (ASSET scope)
Result:   Two separate ATTRIBUTE_DEF rows
          One from TEMPLATE (inherited)
          One from ASSET (custom)
Display:  Both attributes visible to user
```

---

## 7. Storage Comparison

```
❌ Inefficient (Separate Tables)
═══════════════════════════════

For 100 Motor assets, each with "rated_capacity":

TEMPLATE_ATTRIBUTE_DEF
  └─ DEF-100: rated_capacity

MASTER_ATTRIBUTE_DEF
  ├─ DEF-101: rated_capacity (for ASSET-001)  ← DUPLICATE
  ├─ DEF-102: rated_capacity (for ASSET-002)  ← DUPLICATE
  ├─ DEF-103: rated_capacity (for ASSET-003)  ← DUPLICATE
  │ ...
  └─ DEF-200: rated_capacity (for ASSET-100) ← DUPLICATE

Total Rows: 1 + 100 = 101 rows
Duplication: 99 redundant definition copies


✅ Efficient (Polymorphic ATTRIBUTE_DEF)
════════════════════════════════════════

ATTRIBUTE_DEF (single table, polymorphic owner)
  ├─ DEF-001 (scope=TEMPLATE, owner=MOTOR_TEMPLATE_ID)
  │   └─ rated_capacity ← ONE definition, shared by all 100 assets
  │
  ├─ DEF-002 (scope=ASSET, owner=ASSET-001)
  │   └─ custom_field_1 (only for ASSET-001)
  │
  ├─ DEF-003 (scope=ASSET, owner=ASSET-002)
  │   └─ custom_field_2 (only for ASSET-002)
  │
  └─ DEF-004 (scope=ASSET, owner=ASSET-003)
      └─ custom_field_3 (only for ASSET-003)

Total Rows: 1 (template) + ~3 (custom) = 4 rows
Savings: 97 rows (97% reduction)

Formula:
  Inefficient: 1 + N rows (N = number of assets)
  Efficient: 1 + C rows (C = number of custom attrs)
```

---

## 8. Query Flow: Get Attributes for an Asset

```
User opens Asset #123 detail page

                    │
                    ↓

GET /api/v1/assets/ASSET-123/attribute-defs

                    │
                    ↓
        ┌───────────────────────────────────────┐
        │  Backend Query Logic:                 │
        │                                       │
        │  1. Find asset's template_id:         │
        │     template_id = TEMPLATE-MOTOR      │
        │                                       │
        │  2. Fetch TEMPLATE-scoped attrs:      │
        │     SELECT * FROM ATTRIBUTE_DEF       │
        │     WHERE scope = 'TEMPLATE'          │
        │     AND owner_id = TEMPLATE-MOTOR     │
        │     → DEF-001, DEF-002, ... (shared) │
        │                                       │
        │  3. Fetch ASSET-scoped attrs:         │
        │     SELECT * FROM ATTRIBUTE_DEF       │
        │     WHERE scope = 'ASSET'             │
        │     AND owner_id = ASSET-123          │
        │     → DEF-N, DEF-M, ... (custom)     │
        │                                       │
        │  4. For each ATTRIBUTE_DEF:           │
        │     LEFT JOIN ATTRIBUTE_VALUE         │
        │     WHERE asset_master_id = ASSET-123 │
        │                                       │
        │  5. Merge results (template + custom) │
        └───────────────────────────────────────┘
                    │
                    ↓

Response: 200 OK
[
  {
    id: "DEF-001",
    attribute_key: "rated_capacity",
    scope: "TEMPLATE",
    value: "250" (from ATTRIBUTE_VALUE)
    is_overridden: true
  },
  {
    id: "DEF-002",
    attribute_key: "rpm_limit",
    scope: "TEMPLATE",
    value: NULL → "3000" (template default)
    is_overridden: false
  },
  {
    id: "DEF-003",
    attribute_key: "custom_mount_height",
    scope: "ASSET",
    source: "NEW",
    value: "150"
  }
]

                    │
                    ↓

UI displays all attributes in one list
(no distinction between inherited/custom to end user)
```

---

## 9. Phase 1 vs. Phase 2 Feature Matrix

```
┌──────────────────────────┬──────────────┬──────────────────┐
│ Feature                  │ Phase 1      │ Phase 2          │
├──────────────────────────┼──────────────┼──────────────────┤
│ Create attributes        │ Asset only   │ Template + Asset │
│ Default values           │ Manual       │ Via template     │
│ Inheritance              │ Placeholder  │ ✅ Full support  │
│ Override tracking        │ Partial      │ ✅ Full          │
│ Mandatory attributes     │ Manual       │ ✅ Automatic     │
│ Parent linking           │ Not used     │ ✅ Used          │
│ Cross-field rules        │ ❌ No        │ 🔲 Planned       │
│ Bulk attribute updates   │ ❌ No        │ 🔲 Planned       │
└──────────────────────────┴──────────────┴──────────────────┘
```

---

## 10. Key Concepts Checklist

```
Understanding Attribute Inheritance:

□ Understand polymorphic owner pattern (owner_id + scope)
□ Know the difference between TEMPLATE and ASSET scope
□ Can explain why this saves storage (1 def per 100 assets)
□ Can write query to fetch inherited + custom attrs together
□ Know that is_overridden tracks asset overrides
□ Understand parent_attribute_def_id (Phase 2) for linking
□ Know Phase 1 limitation (asset-only creation)
□ Can explain Phase 2 roadmap (template-level creation)
□ Know how values are resolved (ATTRIBUTE_VALUE → default)
□ Understand mandatory attributes (is_mandatory flag)
```

---

*End of Visual Quick Reference*
