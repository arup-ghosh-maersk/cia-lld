# Event Sourcing & Audit Trail

> **Date:** May 3, 2026 | **Version:** 1.0 | **Status:** Production Ready (Phase 2)

## Table of Contents
1. [Overview](#overview)
2. [Event Sourcing Pattern](#event-sourcing-pattern)
3. [Data Model](#data-model)
4. [Event Types](#event-types)
5. [Audit Trail](#audit-trail)
6. [Query Examples](#query-examples)
7. [Retention Policies](#retention-policies)

---

## Overview

The **Event Sourcing & Audit** system captures every change to assets as immutable events, enabling:
- ✅ **Complete History:** Every CRUD operation recorded
- ✅ **Immutable Log:** Events never deleted, only archived
- ✅ **Audit Trail:** Who, what, when, why changes
- ✅ **Compliance:** Regulatory audit requirements
- ✅ **Debugging:** Reconstruct state at any point in time
- ✅ **Event Replay:** Rebuild state from events if needed

### Architecture

```
┌──────────────────┐
│  Asset Changes   │
│  (UI/API)        │
└────────┬─────────┘
         ↓
┌──────────────────────────────┐
│ ASSET_EVENT_STORE            │
│ (Immutable event log)        │
└────────┬─────────────────────┘
         ↓
┌──────────────────────────────┐
│ Event Handlers (Async):      │
│ - AssetEventHandler          │
│ - NotificationHandler        │
│ - DocumentHandler            │
└────────┬─────────────────────┘
         ↓
┌──────────────────────────────┐
│ ASSET_AUDIT                  │
│ (Derived audit records)      │
│ NOTIFICATION_LOG             │
│ (Notification history)       │
└──────────────────────────────┘
```

---

## Event Sourcing Pattern

### Core Concept
Instead of updating a row in ASSET_MASTER, we **append events** to ASSET_EVENT_STORE:

```
Old (State-based):
  ASSET_MASTER [id=123, name="Motor A", status="ACTIVE"]
  → Update name to "Motor B"
  → Query: What changed? Need separate audit table

New (Event-sourced):
  Event 1: AssetCreated {id=123, name="Motor A", ...}
  Event 2: AssetAttributeChanged {id=123, attr=name, old="Motor A", new="Motor B"}
  Event 3: AssetStatusChanged {id=123, old="DRAFT", new="ACTIVE"}
  → Query: Read all events, get complete history
```

### Benefits
- **Traceability:** See entire change history
- **No data loss:** All events preserved
- **Debugging:** Replay events to debug issues
- **Compliance:** Immutable audit trail
- **Performance:** Fast append-only writes

### Immutability Guarantee
```sql
-- Events cannot be deleted or modified
CREATE TRIGGER asset_event_immutable
BEFORE UPDATE OR DELETE ON ASSET_EVENT_STORE
FOR EACH ROW EXECUTE FUNCTION raise_immutability_error();

-- Result: Only INSERT allowed, UPDATE/DELETE blocked
```

---

## Data Model

### ASSET_EVENT_STORE Table
Immutable event log.

```sql
CREATE TABLE ASSET_EVENT_STORE (
    id UUID PRIMARY KEY,
    event_id VARCHAR(50) NOT NULL,        -- UUID for idempotency
    asset_master_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,      -- AssetCreated, AssetAttributeChanged, etc.
    event_version INT DEFAULT 1,
    aggregate_version INT NOT NULL,       -- Sequence number for ordering
    event_timestamp TIMESTAMP NOT NULL,
    event_user_id UUID,
    event_user_name VARCHAR(255),
    event_correlation_id VARCHAR(255),    -- Link related events
    event_payload JSONB NOT NULL,         -- Complete event data
    event_metadata JSONB,                 -- Source IP, browser, etc.
    is_archived BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT asset_events_immutable CHECK (
        archive_date IS NULL OR archive_date >= created_at
    )
);

CREATE INDEX idx_asset_master_id ON ASSET_EVENT_STORE(asset_master_id);
CREATE INDEX idx_event_type ON ASSET_EVENT_STORE(event_type);
CREATE INDEX idx_event_timestamp ON ASSET_EVENT_STORE(event_timestamp DESC);
CREATE INDEX idx_aggregate_version ON ASSET_EVENT_STORE(asset_master_id, aggregate_version);
```

**Key Fields:**
- `event_id` — Unique identifier for idempotent processing (prevents duplicates)
- `aggregate_version` — Sequence number (1, 2, 3...) for total ordering
- `event_payload` — Complete event data (JSON)
- `is_archived` — Archived events not returned in queries (for performance)

### ASSET_AUDIT Table
Derived audit records (view-like).

```sql
CREATE TABLE ASSET_AUDIT (
    id UUID PRIMARY KEY,
    asset_master_id UUID NOT NULL,
    event_store_id UUID NOT NULL,
    change_type VARCHAR(50),              -- CREATED, UPDATED, DELETED
    entity_type VARCHAR(100),             -- ASSET, ATTRIBUTE, COMPONENT
    field_name VARCHAR(255),
    old_value VARCHAR(4000),
    new_value VARCHAR(4000),
    changed_by UUID,
    changed_by_name VARCHAR(255),
    changed_at TIMESTAMP DEFAULT NOW(),
    correlation_id VARCHAR(255),
    ip_address VARCHAR(50),
    notes TEXT,
    
    FOREIGN KEY (asset_master_id) REFERENCES ASSET_MASTER(id),
    FOREIGN KEY (event_store_id) REFERENCES ASSET_EVENT_STORE(id)
);

CREATE INDEX idx_asset_master_id ON ASSET_AUDIT(asset_master_id);
CREATE INDEX idx_changed_at ON ASSET_AUDIT(changed_at DESC);
CREATE INDEX idx_change_type ON ASSET_AUDIT(change_type);
```

---

## Event Types

### Asset Lifecycle Events

#### 1. AssetCreated
Fired when new asset created.

```json
{
  "event_type": "AssetCreated",
  "aggregate_version": 1,
  "event_timestamp": "2026-05-03T10:15:00Z",
  "event_user_id": "user-123",
  "event_user_name": "john.doe@company.com",
  "event_correlation_id": "corr-abc123",
  "event_payload": {
    "asset_master_id": "asset-123",
    "asset_code": "AST-001",
    "name": "Main Pump",
    "asset_template_id": "template-pump-001",
    "status": "DRAFT",
    "installation_date": "2026-05-03",
    "created_by": "john.doe@company.com"
  },
  "event_metadata": {
    "source_ip": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "request_id": "req-xyz789"
  }
}
```

#### 2. AssetAttributeChanged
Fired when attribute value changes.

```json
{
  "event_type": "AssetAttributeChanged",
  "aggregate_version": 5,
  "event_timestamp": "2026-05-03T11:30:00Z",
  "event_user_id": "user-123",
  "event_user_name": "john.doe@company.com",
  "event_payload": {
    "asset_master_id": "asset-123",
    "attribute_def_id": "attr-motor-speed",
    "attribute_key": "motor_speed_rpm",
    "old_value": "1500",
    "new_value": "3000",
    "change_reason": "Customer requested upgrade"
  }
}
```

#### 3. AssetComponentAdded
Fired when component added to asset.

```json
{
  "event_type": "AssetComponentAdded",
  "aggregate_version": 8,
  "event_payload": {
    "asset_master_id": "asset-123",
    "component_id": "comp-bearing-001",
    "master_component_id": "mcomp-456",
    "component_name": "Ball Bearing 6205",
    "serial_number": "BR-2024-0001",
    "quantity": 2,
    "installation_date": "2026-05-03"
  }
}
```

#### 4. AssetAttachmentUploaded
Fired when file uploaded.

```json
{
  "event_type": "AssetAttachmentUploaded",
  "aggregate_version": 12,
  "event_payload": {
    "asset_master_id": "asset-123",
    "attachment_id": "att-789",
    "file_name": "Motor_Manual.pdf",
    "file_size_bytes": 2048000,
    "mime_type": "application/pdf",
    "document_type": "MANUAL",
    "uploaded_by": "john.doe@company.com"
  }
}
```

#### 5. AssetStatusChanged
Fired when asset status changes.

```json
{
  "event_type": "AssetStatusChanged",
  "aggregate_version": 15,
  "event_payload": {
    "asset_master_id": "asset-123",
    "old_status": "DRAFT",
    "new_status": "ACTIVE",
    "status_reason": "Approved by engineering",
    "status_approved_by": "eng-lead@company.com"
  }
}
```

#### 6. DocumentScanned
Fired when file scan completes.

```json
{
  "event_type": "DocumentScanned",
  "aggregate_version": 13,
  "event_payload": {
    "attachment_id": "att-789",
    "scan_status": "CLEAN",
    "risk_level": "CLEAN",
    "threats_found": [],
    "scan_duration_ms": 3500,
    "scan_engine_version": "Votiro 2026.05"
  }
}
```

### Event Catalog

| Event | Trigger | Payload |
|-------|---------|---------|
| AssetCreated | New asset created | asset details |
| AssetUpdated | Asset properties changed | old/new values |
| AssetAttributeChanged | Attribute value changed | attribute key, old/new |
| AssetComponentAdded | Component added | component details |
| AssetComponentRemoved | Component removed | component ID |
| AssetAttachmentUploaded | File uploaded | file details |
| AssetAttachmentDeleted | File deleted | file ID |
| DocumentScanned | CDR scan complete | scan result |
| AssetStatusChanged | Status changed | old/new status |
| AssetHierarchyChanged | Parent changed | old/new parent |

---

## Audit Trail

### Audit Trail Generation
Events flow through event handlers which create audit records:

```csharp
public class AssetEventHandler : IEventHandler
{
    private readonly IRepository _repo;
    private readonly IAuditService _auditService;
    
    public async Task HandleAsync(AssetAttributeChanged @event)
    {
        // Create audit record from event
        var auditRecord = new AssetAudit
        {
            Id = Guid.NewGuid(),
            AssetMasterId = @event.AssetMasterId,
            EventStoreId = @event.EventId,
            ChangeType = "UPDATED",
            EntityType = "ATTRIBUTE",
            FieldName = @event.AttributeKey,
            OldValue = @event.OldValue,
            NewValue = @event.NewValue,
            ChangedBy = @event.EventUserId,
            ChangedByName = @event.EventUserName,
            ChangedAt = @event.EventTimestamp,
            CorrelationId = @event.EventCorrelationId,
            IpAddress = @event.EventMetadata["source_ip"]
        };
        
        await _repo.CreateAuditAsync(auditRecord);
    }
}
```

### Audit Trail Query

```sql
-- Complete history of asset changes
SELECT 
  aa.id,
  aa.entity_type,
  aa.change_type,
  aa.field_name,
  aa.old_value,
  aa.new_value,
  aa.changed_by_name,
  aa.changed_at,
  aa.ip_address
FROM ASSET_AUDIT aa
WHERE aa.asset_master_id = $1
ORDER BY aa.changed_at DESC;
```

### Compliance Reporting

```sql
-- Who changed what, when, why (last 30 days)
SELECT 
  DATE(aa.changed_at) as change_date,
  aa.changed_by_name,
  COUNT(*) as num_changes,
  STRING_AGG(DISTINCT aa.field_name, ', ') as fields_changed
FROM ASSET_AUDIT aa
WHERE aa.asset_master_id = $1
  AND aa.changed_at > NOW() - INTERVAL '30 days'
GROUP BY DATE(aa.changed_at), aa.changed_by_name
ORDER BY change_date DESC;
```

---

## Query Examples

### Query: Recreate Asset State at Specific Time

```sql
-- Get state of asset as it was on 2026-04-01 12:00:00
WITH asset_events AS (
  SELECT 
    event_type,
    aggregate_version,
    event_timestamp,
    event_payload
  FROM ASSET_EVENT_STORE
  WHERE asset_master_id = $1
    AND event_timestamp <= '2026-04-01 12:00:00'
  ORDER BY aggregate_version
)
SELECT 
  event_type,
  event_payload
FROM asset_events;

-- Then in application code: Replay events to reconstruct state
```

### Query: Get All Changes by User

```sql
SELECT 
  aa.changed_at,
  aa.entity_type,
  aa.field_name,
  aa.old_value,
  aa.new_value,
  aa.asset_master_id
FROM ASSET_AUDIT aa
WHERE aa.changed_by = $1
ORDER BY aa.changed_at DESC
LIMIT 100;
```

### Query: Audit Trail with Context

```sql
SELECT 
  aa.changed_at,
  aa.entity_type,
  aa.field_name,
  aa.old_value,
  aa.new_value,
  aa.changed_by_name,
  ase.event_payload->>'change_reason' as reason,
  ase.event_metadata->>'source_ip' as ip_address
FROM ASSET_AUDIT aa
LEFT JOIN ASSET_EVENT_STORE ase ON aa.event_store_id = ase.id
WHERE aa.asset_master_id = $1
ORDER BY aa.changed_at DESC;
```

### Query: Detect Suspicious Activity

```sql
-- Same user changing 100+ assets in 1 hour
SELECT 
  aa.changed_by_name,
  DATE_TRUNC('hour', aa.changed_at) as change_hour,
  COUNT(DISTINCT aa.asset_master_id) as assets_changed,
  COUNT(*) as total_changes
FROM ASSET_AUDIT aa
WHERE aa.changed_at > NOW() - INTERVAL '7 days'
GROUP BY aa.changed_by_name, DATE_TRUNC('hour', aa.changed_at)
HAVING COUNT(DISTINCT aa.asset_master_id) > 100
ORDER BY change_hour DESC;
```

---

## Retention Policies

### Event Archival
```
0-30 days    → Active in ASSET_EVENT_STORE
31-365 days  → In ASSET_EVENT_ARCHIVE
1-7 years    → Long-term BLOB storage
8+ years     → Deleted per policy
```

### SQL: Archive Old Events
```sql
BEGIN TRANSACTION;

-- Move events older than 1 year to archive
INSERT INTO ASSET_EVENT_ARCHIVE
SELECT * FROM ASSET_EVENT_STORE
WHERE created_at < NOW() - INTERVAL '1 year'
  AND is_archived = false;

-- Mark as archived
UPDATE ASSET_EVENT_STORE
SET is_archived = true
WHERE created_at < NOW() - INTERVAL '1 year';

COMMIT;
```

### Compliance Options

| Regulation | Retention | Action |
|-----------|-----------|--------|
| **GDPR** | Until right-to-be-forgotten | Delete events with personal data |
| **SOX** | 7 years | Retain full audit trail |
| **HIPAA** | 6 years | Retain all access logs |
| **ISO 9001** | 6 years | Quality record retention |

---

## Related Documents
- [1_ARCHITECTURE.md](./1_ARCHITECTURE.md) — System overview
- [7_HANDLERS.md](./7_HANDLERS.md) — Event processing handlers
- [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) — Event type quick reference
- [FAQ.md](./FAQ.md) — Event sourcing FAQs

---

**Last Updated:** May 3, 2026 | **Next Review:** Phase 2 completion
