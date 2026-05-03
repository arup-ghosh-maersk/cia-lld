# Architecture — Visual Summary

> **Focus:** Diagrams & crisp explanations | **No lengthy code** | **Visual-first**

---

## System Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASSET MASTER SYSTEM                          │
│                    (Event-Driven Architecture)                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────┐       ┌──────────┐       ┌─────────────┐
│  User   │──────▶│ Asset UI │──────▶│ Asset API   │
│ Creates │       │          │       │             │
│ Asset   │       └──────────┘       └──────┬──────┘
└─────────┘                                  │
                                             ▼
                                    ┌────────────────┐
                                    │  ASSET_MASTER  │
                                    │  COMPONENTS    │
                                    │  ATTRIBUTES    │
                                    └────────┬───────┘
                                             │
                                             ▼
                                    ┌────────────────────┐
                                    │ ASSET_EVENT_STORE  │
                                    │ (Immutable Events) │
                                    └────────┬───────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
                    ▼                        ▼                        ▼
            ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
            │ AssetEvent   │        │Notification │        │  Document    │
            │ Handler      │        │  Handler     │        │  Handler     │
            └──────┬───────┘        └──────┬───────┘        └──────┬───────┘
                   │                       │                       │
                   ▼                       ▼                       ▼
            ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
            │ ASSET_AUDIT  │        │NOTIFICATION │        │ DOCUMENT_SCAN│
            │  (Immutable) │        │   (UI/Email) │        │ (Votiro CDR) │
            └──────────────┘        └──────────────┘        └──────────────┘
                                             │
                                             ▼
                                        ┌─────────┐
                                        │   UI    │
                                        │Real-time│
                                        │Alerts   │
                                        └─────────┘
```

---

## Entity Relationship Diagram (Simplified)

```
┌────────────────────────┐
│   ASSET_TEMPLATE       │  ◄─── Reusable template
│ (asset code, type,     │       (AG#####.24.141)
│  object_level)         │
└────────┬───────────────┘
         │ instantiates
         ▼
┌────────────────────────────────────────────────┐
│         ASSET_MASTER (Individual Assets)       │
│   (id, asset_code, name, serial_number)        │
└────┬─────────────┬──────────────┬──────────────┘
     │ has         │ has          │ has
     ▼             ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐
│ATTRIBUTE │  │COMPONENT │  │ ATTACHMENT   │
│ Values   │  │ (Parts)  │  │ (Documents)  │
└──────────┘  └──────────┘  └──────┬───────┘
                                   │
                                   ▼
                            ┌──────────────┐
                            │DOCUMENT_SCAN │
                            │ (Votiro CDR) │
                            └──────────────┘
     │
     └─ generates
        ▼
   ┌──────────────┐
   │ASSET_EVENT   │ ◄─── Immutable event log
   │ (Every change)
   └──────┬───────┘
          │
          ├─ triggers
          │   ▼
          │ ASSET_AUDIT (Audit record)
          │
          └─ triggers
              ▼
            NOTIFICATION_LOG (Alert record)
```

---

## Data Model — Core Tables

### ASSET_TEMPLATE
```
┌─────────────────────────┐
│  ASSET_TEMPLATE         │
├─────────────────────────┤
│ id (PK)                 │
│ object_id (UK) ◄─ AG##### example
│ object_level ◄─ 200-600 (Lv1-Lv5)
│ object_type  ◄─ AG, AGV, AMS, RTG
│ category     ◄─ EQ, CIV, TOOL
│ functional_type ◄─ Instrument, Rotary
│ parent_template_id (FK) ◄─ Hierarchy
│ hierarchy_path ◄─ Materialized path
└─────────────────────────┘
```

### ASSET_MASTER
```
┌──────────────────────────┐
│  ASSET_MASTER            │
├──────────────────────────┤
│ id (PK)                  │
│ asset_code (UK)          │
│ name                     │
│ serial_number            │
│ status ◄─ ACTIVE/INACTIVE
│ asset_template_id (FK)   │
│ parent_asset_id (FK) ◄─ Hierarchy
│ installation_date        │
└──────────────────────────┘
```

### ATTRIBUTE_DEF (Reusable)
```
┌──────────────────────────────┐
│  ATTRIBUTE_DEF               │
├──────────────────────────────┤
│ id (PK)                      │
│ attribute_key (UK)           │
│ display_name                 │
│ data_type ◄─ STRING, NUMBER, DATE, ENUM
│ unit ◄─ RPM, kg, kW, V
│ group_name ◄─ Electrical, Mechanical
│ is_required                  │
│ default_value                │
│ valid_values (JSONB) ◄─ For ENUM
│ validation_rules (JSONB) ◄─ Array of rules
└──────────────────────────────┘
```

### MASTER_ATTRIBUTE (Asset-Level Values)
```
┌──────────────────────────────┐
│  MASTER_ATTRIBUTE            │
├──────────────────────────────┤
│ id (PK)                      │
│ asset_master_id (FK)         │
│ attribute_def_id (FK)        │
│ attribute_value              │
│ is_inherited ◄─ From template
│ validation_status ◄─ VALID/INVALID/WARNING
│ validation_errors (JSONB)    │
└──────────────────────────────┘
```

### ASSET_EVENT_STORE (Immutable Log)
```
┌──────────────────────────────┐
│  ASSET_EVENT_STORE           │
├──────────────────────────────┤
│ event_id (PK)                │
│ asset_master_id (FK)         │
│ event_type ◄─ Create, Update, Delete
│ event_timestamp              │
│ event_user_id & event_user_name
│ event_payload (JSONB) ◄─ Full snapshot
│ event_metadata ◄─ IP, browser, etc.
│ is_archived (immutable)      │
└──────────────────────────────┘
```

### ASSET_AUDIT (Derived from Events)
```
┌──────────────────────────────┐
│  ASSET_AUDIT                 │
├──────────────────────────────┤
│ id (PK)                      │
│ asset_master_id (FK)         │
│ event_store_id (FK)          │
│ change_type ◄─ CREATED, UPDATED
│ entity_type ◄─ ASSET, ATTRIBUTE
│ field_name                   │
│ old_value / new_value        │
│ changed_by / changed_at      │
└──────────────────────────────┘
```

---

## Component Diagram — Handler Architecture

```
┌────────────────────────────────────────────────────────┐
│           EVENT-DRIVEN HANDLER SYSTEM                  │
└────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────┐
    │    ASSET_EVENT_STORE (Event Queue)      │
    │    Published events from API             │
    └──────────────┬──────────────────────────┘
                   │ Subscribers
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
    ┌────────┐ ┌──────────┐ ┌──────────┐
    │ Asset  │ │Notif     │ │Document  │
    │Event   │ │Handler   │ │Handler   │
    │Handler │ │          │ │          │
    └────┬───┘ └────┬─────┘ └────┬─────┘
         │          │           │
         ▼          ▼           ▼
    ┌────────┐ ┌──────────┐ ┌──────────┐
    │ASSET   │ │NOTIF     │ │DOCUMENT  │
    │AUDIT   │ │LOG       │ │SCAN      │
    │(Record)│ │(UI/Email)│ │(Votiro)  │
    └────────┘ └──────────┘ └──────────┘
```

---

## Sequence Diagram — Asset Creation

```
User                API               Database          EventStore        Handler          UI
 │                  │                   │                   │              │               │
 ├─ Submit Form ───▶│                   │                   │              │               │
 │                  │                   │                   │              │               │
 │                  ├─ Validate ───────▶│                   │              │               │
 │                  │◀─ OK ─────────────┤                   │              │               │
 │                  │                   │                   │              │               │
 │                  ├─ INSERT Asset ───▶│                   │              │               │
 │                  │ INSERT Attributes │                   │              │               │
 │                  │ INSERT Components │                   │              │               │
 │                  │◀─ AssetId ────────┤                   │              │               │
 │                  │                   │                   │              │               │
 │                  ├─ Publish Event ───────────────────────▶│              │               │
 │                  │                   │                   │              │               │
 │◀─ 201 Created ───┤                   │                   │              │               │
 │                  │                   │                   │              │               │
 │                  │                   │                   │    Consume    │               │
 │                  │                   │                   │◀──────────────┤               │
 │                  │                   │                   │              │               │
 │                  │                   │   Generate Audit ─┼─────────────▶│               │
 │                  │                   │                   │              │               │
 │                  │                   │   Dispatch Notif ─┼──────────────────────────────▶
 │                  │                   │                   │              │       Push Alert
```

---

## Sequence Diagram — Attribute Validation

```
User            UI                API            Validation       Database        Handler
 │              │                  │               Engine           │               │
 ├─ Enter 5000  │                  │                │               │               │
 │ (RPM value)  │                  │                │               │               │
 │              ├─ POST Attr ─────▶│                │               │               │
 │              │ (value: 5000)    │                │               │               │
 │              │                  │                │               │               │
 │              │                  ├─ Get Rules ───────────────────▶│               │
 │              │                  │◀─ [MIN:100, MAX:3000] ─────────┤               │
 │              │                  │                │               │               │
 │              │                  ├─ Validate ────▶│               │               │
 │              │                  │                │               │               │
 │              │                  │  5000 > 3000? ✓ FAIL           │               │
 │              │                  │◀─ ERROR ───────┤               │               │
 │              │                  │                │               │               │
 │              │◀─ 400 Bad Req ───┤                │               │               │
 │              │    "Max 3000"    │                │               │               │
 │              │                  │                │               │               │
 ├─ Enter 2500  │                  │                │               │               │
 │              ├─ POST Attr ─────▶│                │               │               │
 │              │ (value: 2500)    │                │               │               │
 │              │                  ├─ Validate ────▶│               │               │
 │              │                  │ 100 ≤ 2500 ≤ 3000 ✓ PASS       │               │
 │              │                  │◀─ OK ──────────┤               │               │
 │              │                  │                │               │               │
 │              │                  ├─ INSERT ──────────────────────▶│               │
 │              │                  │◀─ OK ──────────────────────────┤               │
 │              │                  │                │               │               │
 │              │                  ├─ Publish Event ──────────────────────────────▶│
 │              │◀─ 200 OK ────────┤                │               │               │
```

---

## Validation Rules Flow

```
Attribute Value Submitted
    │
    ▼
Load ATTRIBUTE_DEF
    │
    ▼
Get validation_rules array
    │
    ▼
Sort by evaluation_order
    │
    ├─ Rule 1 (REQUIRED): value is not empty? 
    │   └─ NO ──▶ Add Error ──▶ STOP (severity: ERROR)
    │   └─ YES ─▶ Continue
    │
    ├─ Rule 2 (MIN_VALUE): value >= 100?
    │   └─ NO ──▶ Add Error ──▶ STOP
    │   └─ YES ─▶ Continue
    │
    ├─ Rule 3 (MAX_VALUE): value <= 3000?
    │   └─ NO ──▶ Add Warning ─▶ Continue (severity: WARNING)
    │   └─ YES ─▶ Continue
    │
    ▼
Return Result
    ├─ Has Errors?     ──▶ VALIDATION FAILED ❌
    ├─ Has Warnings?   ──▶ VALIDATION PASSED ⚠️  (with warnings)
    └─ No Issues?      ──▶ VALIDATION PASSED ✅
```

---

## File Processing Pipeline

```
User Uploads File
    │
    ▼
┌──────────────────┐
│ Validate File    │
├──────────────────┤
│ ✓ Size < 100MB   │
│ ✓ Type allowed   │
│ ✓ Signature OK   │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Upload to Azure  │
│ Blob Storage     │
└──────┬───────────┘
       │ (Encrypted at rest)
       ▼
┌──────────────────┐
│ Create ATTACHMENT│
│ record           │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Send to Votiro   │
│ for CDR Scanning │
└──────┬───────────┘
       │
       ├─ PENDING ──▶ (waiting)
       │
       ├─ SCANNING ─▶ (processing)
       │
       ├─ CLEAN ────▶ ✅ Download allowed
       │
       ├─ INFECTED ─▶ 🛡️  Remediate (if possible)
       │             └─▶ ✅ Download cleaned file
       │
       └─ FAILED ───▶ ❌ Manual review needed
```

---

## Integration Points

```
Asset Master System
    │
    ├─ Votiro CDR ◀──────▶ Malware scanning
    │  (File security)
    │
    ├─ Azure Blob ◀──────▶ Document storage
    │  (Encrypted)
    │
    ├─ SendGrid ◀────────▶ Email notifications
    │  (Email service)
    │
    ├─ Webhooks ◀────────▶ External systems
    │  (Event push)
    │
    └─ Azure Service Bus ◀▶ Event messaging
       (Async processing)
```

---

## Hierarchy Example (Materialized Path)

```
Lv1 (Level 200): AG00001
    │
    ├─ Lv2 (Level 300): AG00001.24 (Location)
    │   │
    │   ├─ Lv3 (Level 400): AG00001.24.141 (Sub-Location)
    │   │   │
    │   │   ├─ Lv4 (Level 500): AG00001.24.141.528 (Assembly)
    │   │   │   │
    │   │   │   ├─ Lv5 (Level 600): AG00001.24.141.528.119 (Motor)
    │   │   │   │   └─ Status: ACTIVE
    │   │   │   │
    │   │   │   └─ Lv5 (Level 600): AG00001.24.141.528.120 (Pump)
    │   │   │       └─ Status: ACTIVE
    │   │   │
    │   │   └─ Lv4 (Level 500): AG00001.24.141.529 (Frame)
    │   │
    │   └─ Lv3 (Level 400): AG00001.24.142
    │
    └─ Lv2 (Level 300): AG00001.25
```

**Materialized Path Storage:**
```
AG00001.24.141.528.119 → Fast traversal (no joins)
```

---

## Key Design Patterns

### 1. Catalog Pattern (3-Table Junction)
```
ATTRIBUTE_DEF (Catalog) ──┐
                          ├─ TEMPLATE_ATTRIBUTE ──┐
                                                   ├─ MASTER_ATTRIBUTE
(Reusable definition)     (Link + metadata)       (Value on asset)
```

### 2. Event Sourcing
```
Event Created ──▶ Event Stored ──▶ Handler Consumes ──▶ Audit Generated
                 (Immutable)       (Async)             (Derived record)
```

### 3. Polymorphic Scope
```
Attribute: "blade_count"
Scopes:
  ├─ FAN equipment
  └─ COMPRESSOR equipment
(Not applicable to MOTOR)
```

### 4. Materialized Path Hierarchy
```
Lv1.Lv2.Lv3.Lv4.Lv5 ──▶ Single string ──▶ Fast queries
(No recursive joins)
```

---

## Performance Targets

| Operation | Target | Achieved |
|-----------|--------|----------|
| Create Asset (with 10 attrs, 5 components) | <500ms | ✅ |
| List Attributes for Asset | <100ms | ✅ |
| Validate Single Attribute | <10ms | ✅ |
| Scan File (via Votiro) | <5min | ✅ |
| Audit Retrieval (paginated) | <200ms | ✅ |

---

## Summary

✅ **Event-Driven** — Real-time processing  
✅ **Immutable Audit** — Complete change history  
✅ **Reusable Catalog** — Attributes, components shared  
✅ **Hierarchical Assets** — GOS register structure  
✅ **Dynamic Validation** — 8 rule types, JSON-based  
✅ **Secure Files** — CDR scanning + encryption  
✅ **Multi-Handler** — Audit, notifications, CDR all async  

---

**Type:** Architecture | **Format:** Visual-First | **No Code**
