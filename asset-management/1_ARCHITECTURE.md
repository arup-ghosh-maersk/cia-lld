# 1. ARCHITECTURE — Asset Master System

> **Purpose:** Complete system design, data model, ER diagrams

---

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│           Asset Master System (Event-Driven)            │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  User Creates/Updates Asset in UI                       │
│           ↓                                              │
│  API validates & persists (ASSET_MASTER, etc.)          │
│           ↓                                              │
│  Event published to ASSET_EVENT_STORE (immutable)       │
│           ↓                                              │
│  AssetEventHandler consumes → generates audit message   │
│           ↓                                              │
│  NotificationHandler dispatches (UI, email, webhook)    │
│           ↓                                              │
│  DocumentHandler processes file uploads + Votiro scan   │
│           ↓                                              │
│  User sees real-time notification                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Complete ER Diagram (All 16 Entities)

```mermaid
erDiagram
    ASSET_TEMPLATE ||--o{ ASSET_TEMPLATE : "parent_of"
    ASSET_TEMPLATE ||--o{ ASSET_MASTER : "instantiated_as"
    ASSET_TEMPLATE ||--o{ TEMPLATE_COMPONENT : "defines"
    ASSET_TEMPLATE ||--o{ TEMPLATE_ATTACHMENT : "defines"
    ASSET_TEMPLATE ||--o{ TEMPLATE_ATTRIBUTE : "defines"
    ASSET_MASTER ||--o{ ASSET_MASTER : "parent_of"
    ASSET_MASTER ||--o{ MASTER_COMPONENT : "has"
    ASSET_MASTER ||--o{ MASTER_ATTACHMENT : "has"
    ASSET_MASTER ||--o{ MASTER_ATTRIBUTE : "has"
    ASSET_MASTER ||--o{ ASSET_EVENT_STORE : "generates"
    ATTRIBUTE_DEF ||--o{ TEMPLATE_ATTRIBUTE : "referenced_by"
    ATTRIBUTE_DEF ||--o{ MASTER_ATTRIBUTE : "referenced_by"
    TEMPLATE_ATTRIBUTE ||--o{ MASTER_ATTRIBUTE : "sourced_from"
    COMPONENT ||--o{ TEMPLATE_COMPONENT : "referenced_by"
    COMPONENT ||--o{ MASTER_COMPONENT : "referenced_by"
    ATTACHMENT ||--o{ TEMPLATE_ATTACHMENT : "referenced_by"
    ATTACHMENT ||--o{ MASTER_ATTACHMENT : "references"
    ATTACHMENT ||--o{ DOCUMENT_SCAN : "scanned_by"
    TEMPLATE_COMPONENT ||--o{ MASTER_COMPONENT : "sourced_from"
    TEMPLATE_ATTACHMENT ||--o{ MASTER_ATTACHMENT : "sourced_from"
    ASSET_EVENT_STORE ||--o{ ASSET_AUDIT : "produces"
    ASSET_AUDIT ||--o{ NOTIFICATION_LOG : "dispatched_as"

    ASSET_TEMPLATE {
        uuid id PK
        uuid parent_template_id FK
        string object_id UK "e.g. AG#####.24.141.528.119"
        string object_type "AG|AGV|AMS|RTG|STS"
        int object_level "200-600 (Lv1-Lv5)"
        string category "EQ|CIV|TOOL|L2.xx"
        string functional_type "Instrument|Structural|Rotary|Electrical"
        string hierarchy_path "materialized path"
    }

    ASSET_MASTER {
        uuid id PK
        uuid asset_template_id FK
        uuid parent_asset_id FK
        string asset_code UK
        string name
        string serial_number
        string status "ACTIVE|INACTIVE|DECOMMISSIONED"
        date installation_date
    }

    ATTRIBUTE_DEF {
        uuid id PK
        string attribute_key UK
        string display_name
        string data_type "STRING|NUMBER|DATE|BOOLEAN|ENUM|JSON"
        string default_value
        jsonb valid_values
        string unit "kg|kW|V|mm|rpm"
        string group_name "Electrical|Mechanical"
        int sort_order
        boolean is_required
        jsonb validation_rules "inline array"
        string notes
    }

    TEMPLATE_ATTRIBUTE {
        uuid id PK
        uuid asset_template_id FK
        uuid attribute_def_id FK
        boolean is_mandatory
        int sort_order
        string notes
    }

    MASTER_ATTRIBUTE {
        uuid id PK
        uuid asset_master_id FK
        uuid attribute_def_id FK
        uuid template_attribute_id FK
        string source "FROM_TEMPLATE|NEW"
        string value_string
        decimal value_number
        date value_date
        boolean value_boolean
        jsonb value_json
        string validation_status "VALID|INVALID|WARNING|PENDING"
        jsonb validation_errors
        boolean is_overridden
    }

    COMPONENT {
        uuid id PK
        string component_code UK
        string name
        jsonb specifications
    }

    ATTACHMENT {
        uuid id PK
        string file_name
        string file_path
        string file_type
        string mime_type
        string category "MANUAL|DRAWING|CERTIFICATE|IMAGE|OTHER"
        bigint file_size
        string scan_status "PENDING|CLEAN|INFECTED|FAILED"
        timestamp uploaded_at
        uuid uploaded_by FK
    }

    TEMPLATE_COMPONENT {
        uuid id PK
        uuid asset_template_id FK
        uuid component_id FK
        int quantity
        boolean is_mandatory
    }

    TEMPLATE_ATTACHMENT {
        uuid id PK
        uuid asset_template_id FK
        uuid attachment_id FK
        boolean is_mandatory
    }

    MASTER_COMPONENT {
        uuid id PK
        uuid asset_master_id FK
        uuid component_id FK
        uuid template_component_id FK
        string source "FROM_TEMPLATE|NEW"
        string status "ACTIVE|EXCLUDED"
    }

    MASTER_ATTACHMENT {
        uuid id PK
        uuid asset_master_id FK
        uuid attachment_id FK
        uuid template_attachment_id FK
        string source "FROM_TEMPLATE|NEW"
        string status "ACTIVE|EXCLUDED"
    }

    ASSET_EVENT_STORE {
        uuid EventId PK
        uint EventType "1=Create|2=Update|3=Delete"
        jsonb EventDetails "full payload"
        timestamp created_at
        string created_by
    }

    ASSET_AUDIT {
        uuid NotificationId PK
        uuid event_id FK
        string AuditMessage
        jsonb Detail
        string status "PENDING|DISPATCHED|FAILED"
        timestamp created_at
        string created_by
    }

    NOTIFICATION_LOG {
        uuid id PK
        uuid notification_id FK
        string channel "UI_PUSH|EMAIL|WEBHOOK"
        string recipient
        string status "SENT|FAILED|RETRYING"
        int retry_count
        string error_message
        timestamp sent_at
        timestamp created_at
    }

    DOCUMENT_SCAN {
        uuid id PK
        uuid attachment_id FK
        string scan_engine "VOTIRO"
        string scan_status "PENDING|CLEAN|INFECTED|FAILED"
        string scan_result_detail
        string sanitized_file_path
        timestamp scanned_at
        string scan_reference_id
    }
```

---

## Domain Splits (4 Focus Areas)

### 1. Core Asset Domain

**Tables:** ASSET_TEMPLATE, ASSET_MASTER

**Purpose:** Hierarchical asset structure mirroring GOS Register (APMT-MnR-SOP-0080-102)

**Levels:**
- Lv1 (200): e.g., AG##### (equipment group)
- Lv2 (300): e.g., AG#####.24 (location)
- Lv3 (400): e.g., AG#####.24.141 (sub-location)
- Lv4 (500): e.g., AG#####.24.141.528 (assembly)
- Lv5 (600): e.g., AG#####.24.141.528.119 (component)

**Self-referencing:** `parent_template_id` enables hierarchy

---

### 2. Attributes Domain

**Tables:** ATTRIBUTE_DEF, TEMPLATE_ATTRIBUTE, MASTER_ATTRIBUTE

**Key Concept:** Shared catalog pattern (same as Components)

- **ATTRIBUTE_DEF:** Reusable definition (key, type, rules, defaults)
- **TEMPLATE_ATTRIBUTE:** Template → Attribute link + mandatory flag
- **MASTER_ATTRIBUTE:** Asset → Attribute link + value + validation status

**Storage:** Single ATTRIBUTE_DEF shared across 100+ assets = 98% storage savings

---

### 3. Components & Attachments Domain

**Tables:** COMPONENT, ATTACHMENT, TEMPLATE_*, MASTER_*

**Pattern:** 3-table junctions (identical structure for both)

```
COMPONENT (catalog)
    ↓
TEMPLATE_COMPONENT (link)
    ↓
MASTER_COMPONENT (value on asset)
```

---

### 4. Event & Audit Domain

**Tables:** ASSET_EVENT_STORE, ASSET_AUDIT, NOTIFICATION_LOG, DOCUMENT_SCAN

**Pattern:** Immutable event log → audit generation → notification dispatch

```
Event → Audit → Notification
(1 row)   (1 row)   (1-3 rows)
```

---

## Data Model Details

### ASSET_TEMPLATE
```sql
CREATE TABLE asset_template (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_template_id UUID REFERENCES asset_template(id),
    object_id VARCHAR(50) UNIQUE NOT NULL,  -- e.g., AG#####.24.141.528.119
    object_type VARCHAR(20) NOT NULL,        -- AG, AGV, AMS, RTG, STS
    object_level INT NOT NULL,               -- 200=Lv1, ..., 600=Lv5
    category VARCHAR(20) NOT NULL,           -- EQ, CIV, TOOL, L2.xx
    functional_type VARCHAR(50),             -- Instrument, Structural, Rotary
    hierarchy_path VARCHAR(255) NOT NULL,    -- materialized path for indexing
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Indexes:**
- `object_id` (UNIQUE)
- `hierarchy_path` (for hierarchy traversal)
- `object_level` (for filtering by level)

---

### ASSET_MASTER
```sql
CREATE TABLE asset_master (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_template_id UUID NOT NULL REFERENCES asset_template(id),
    parent_asset_id UUID REFERENCES asset_master(id),
    asset_code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    serial_number VARCHAR(100),
    status VARCHAR(20) DEFAULT 'ACTIVE',     -- ACTIVE, INACTIVE, DECOMMISSIONED
    installation_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Indexes:**
- `asset_code` (UNIQUE)
- `asset_template_id` (foreign key)
- `parent_asset_id` (hierarchy)
- `status` (for filtering)

---

### ATTRIBUTE_DEF
```sql
CREATE TABLE attribute_def (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attribute_key VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    data_type VARCHAR(20) NOT NULL,          -- STRING, NUMBER, DATE, BOOLEAN, ENUM, JSON
    default_value VARCHAR(500),
    valid_values JSONB,                      -- For ENUM: ["Low", "Medium", "High"]
    unit VARCHAR(50),                        -- kg, kW, V, mm, rpm
    group_name VARCHAR(100),                 -- UI grouping: Electrical, Mechanical
    sort_order INT DEFAULT 0,
    is_required BOOLEAN DEFAULT FALSE,
    validation_rules JSONB NOT NULL DEFAULT '[]',  -- Array of rule objects
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Validation Rules Structure:**
```json
[
  {
    "type": "REQUIRED|MIN_VALUE|MAX_VALUE|MIN_LENGTH|MAX_LENGTH|REGEX|ENUM|DATE_RANGE",
    "value": "depends on type",
    "message": "User-facing error message",
    "severity": "ERROR|WARNING",
    "active": true
  }
]
```

**Indexes:**
- `attribute_key` (UNIQUE)
- `group_name` (for UI grouping)

---

### MASTER_ATTRIBUTE
```sql
CREATE TABLE master_attribute (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_master_id UUID NOT NULL REFERENCES asset_master(id),
    attribute_def_id UUID NOT NULL REFERENCES attribute_def(id),
    template_attribute_id UUID REFERENCES template_attribute(id),
    source VARCHAR(20) NOT NULL,             -- FROM_TEMPLATE or NEW
    value_string VARCHAR(500),
    value_number DECIMAL(18, 4),
    value_date DATE,
    value_boolean BOOLEAN,
    value_json JSONB,
    validation_status VARCHAR(20) DEFAULT 'PENDING',  -- VALID, INVALID, WARNING, PENDING
    validation_errors JSONB,                 -- Snapshot of failed rules
    is_overridden BOOLEAN DEFAULT FALSE,     -- True if differs from default
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(asset_master_id, attribute_def_id)
);
```

**Indexes:**
- `asset_master_id` (for asset queries)
- `attribute_def_id` (for attribute queries)
- `validation_status` (for filtering invalid records)

---

### ASSET_EVENT_STORE
```sql
CREATE TABLE asset_event_store (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type SMALLINT NOT NULL,            -- 1=Create, 2=Update, 3=Delete
    event_details JSONB NOT NULL,            -- Full asset snapshot at time of event
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,
    INDEX event_created_at (created_at DESC) -- For efficient time-range queries
);
```

**Event Details Structure:**
```json
{
  "assetId": "uuid",
  "assetCode": "ASSET001",
  "modifiedFields": ["serial_number", "status"],
  "componentCount": 5,
  "attachmentCount": 4,
  "before": { /* full snapshot before */ },
  "after": { /* full snapshot after */ }
}
```

---

### ASSET_AUDIT
```sql
CREATE TABLE asset_audit (
    notification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL REFERENCES asset_event_store(event_id),
    audit_message VARCHAR(500) NOT NULL,
    detail JSONB NOT NULL,                   -- Snapshot of event
    status VARCHAR(20) DEFAULT 'PENDING',    -- PENDING, DISPATCHED, FAILED
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,
    INDEX audit_created_at (created_at DESC)
);
```

---

## Sequence Diagrams

### Asset Create Flow

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Asset UI
    participant API as Asset API
    participant DB as Database
    participant ES as EventStore
    participant Handler as EventHandler
    participant Audit as AuditStore
    participant Push as Notification

    User->>UI: Fill form & Submit
    UI->>API: POST /api/v1/assets
    API->>DB: INSERT Asset, Components, Attachments
    DB-->>API: AssetId: ASSET001

    API->>ES: INSERT Event {Type: Create, Details: {...}}
    ES-->>API: EventId
    API-->>UI: 201 Created

    Note over ES,Push: Async Processing
    Handler->>ES: Consume Create Event
    Handler->>Audit: INSERT "Created Asset ASSET001, Added 5 Components, 4 Attachments"
    Handler->>Push: Dispatch Notification
    Push-->>UI: Real-time notification
```

### Asset Update Flow

```mermaid
sequenceDiagram
    autonumber
    User->>UI: Modify Asset & Submit
    UI->>API: PUT /api/v1/assets/ASSET001
    API->>DB: UPDATE Asset Record
    API->>ES: INSERT Event {Type: Update, Details: {...}}
    API-->>UI: 200 OK
    Note over ES: Async
    Handler->>Audit: INSERT "Updated ASSET001, Added 2 Components"
    Handler->>Push: Dispatch
```

---

## Performance Optimization

### Indexes Strategy

| Table | Index | Reason |
|-------|-------|--------|
| ASSET_MASTER | asset_code | Unique lookups |
| ASSET_MASTER | asset_template_id | Template filtering |
| MASTER_ATTRIBUTE | asset_master_id | Asset attribute queries |
| MASTER_ATTRIBUTE | validation_status | Find invalid records |
| ASSET_EVENT_STORE | created_at DESC | Time-range queries |
| ASSET_AUDIT | created_at DESC | Audit trail pagination |

### Partitioning Strategy

For tables exceeding 100M rows (ASSET_AUDIT, MASTER_ATTRIBUTE):
- **Partition by:** created_at (monthly or quarterly)
- **Benefits:** Faster archival, parallel scans, reduced index size

---

## Scalability Considerations

| Scenario | Estimated Scale | Mitigation |
|----------|-----------------|------------|
| 100K assets × 50 attributes | 5M MASTER_ATTRIBUTE rows | Partition, index on asset_id |
| 10M audit entries/year | Growing table | Archive to historical DB |
| 1000 concurrent users | Peak load | Message queue, read replicas |
| 10 TB file storage | File blobs | S3/Azure Blob, CDN caching |

---

## Next Steps

1. **Implementation:** See [8_HANDLERS_IMPLEMENTATION.md](./8_HANDLERS_IMPLEMENTATION.md)
2. **API Design:** See [9_API_ENDPOINTS.md](./9_API_ENDPOINTS.md)
3. **Integration:** See [10_INTEGRATION_GUIDE.md](./10_INTEGRATION_GUIDE.md)

---

**File:** 1_ARCHITECTURE.md | **Lines:** ~300
