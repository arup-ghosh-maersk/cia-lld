# 2. DESIGN PATTERNS — Asset Master System

> **Purpose:** Core patterns, principles, and architectural decisions

---

## Pattern 1: Catalog + 3-Table Junction

### Problem
How to reuse attribute definitions across 100+ assets without duplication?

### Solution
Adopt the **catalog pattern** used for Components:

```
ATTRIBUTE_DEF (Catalog)
    ↓ (referenced by)
TEMPLATE_ATTRIBUTE (Template Link)
    ↓ (sourced from)
MASTER_ATTRIBUTE (Asset Link + Value)
```

### Example

```sql
-- 1. Define attribute once (catalog)
INSERT INTO attribute_def (id, attribute_key, display_name, data_type)
VALUES ('DEF-001', 'rated_capacity', 'Rated Capacity', 'NUMBER');

-- 2. Link to template (indicates required for all assets)
INSERT INTO template_attribute (asset_template_id, attribute_def_id, is_mandatory)
VALUES ('TEMPLATE-001', 'DEF-001', TRUE);

-- 3. Store value on asset (multiple assets, single definition)
INSERT INTO master_attribute (asset_master_id, attribute_def_id, value_number)
VALUES 
  ('ASSET-001', 'DEF-001', 250),
  ('ASSET-002', 'DEF-001', 350),
  ('ASSET-003', 'DEF-001', 500);
```

**Storage:**
- Without pattern: 3 rows (100% duplication)
- With pattern: 1 row in ATTRIBUTE_DEF + 3 rows in MASTER_ATTRIBUTE (98% savings)

### Benefits
- ✅ Single source of truth (validation rules in one place)
- ✅ Efficient storage (no duplication)
- ✅ Easy updates (change rule once, applies to all assets)
- ✅ Same pattern as Components & Attachments (consistency)

---

## Pattern 2: Event Sourcing (Immutable Event Log)

### Problem
How to maintain a complete audit trail without performance overhead?

### Solution
Store all changes as **immutable events** in ASSET_EVENT_STORE:

```
User Action → Event Published → Handler Processes → Audit Generated
   (1 write)      (append-only)      (async)          (audit trail)
```

### Example

```sql
-- Asset created: Write once to event store
INSERT INTO asset_event_store (event_type, event_details, created_by)
VALUES 
  (1, 
   {
     "assetId": "ASSET-001",
     "assetCode": "ASSET001",
     "name": "Motor #123",
     "components": [{"id": "COMP-01"}, {"id": "COMP-02"}],
     "attachments": [{"id": "ATT-01"}]
   },
   'user@company.com');

-- Event handler reads and generates audit
-- (separate transaction, eventual consistency)
INSERT INTO asset_audit (event_id, audit_message, detail)
VALUES 
  ('EVENT-001',
   'Created Asset ASSET001, Added Components (2), Attachments (1)',
   {...});
```

### Benefits
- ✅ **Immutable:** Cannot be modified, audit trail is legally compliant
- ✅ **Async:** Event processing doesn't block user
- ✅ **Traceable:** Every change has source event
- ✅ **Scalable:** Append-only log, no complex locking

### Event Types
| Type | Code | Trigger |
|------|------|---------|
| AssetCreation | 1 | POST /api/v1/assets |
| AssetUpdate | 2 | PUT /api/v1/assets/{id} |
| AssetDeletion | 3 | DELETE /api/v1/assets/{id} |

---

## Pattern 3: Polymorphic Scope (Template vs. Asset Attributes)

### Problem
How to distinguish between template-level and asset-custom attributes?

### Solution
Use **source field** in MASTER_ATTRIBUTE:

```
source = FROM_TEMPLATE → inherited from template definition
source = NEW → custom attribute created directly on asset
```

### Example

```sql
-- Template attribute (inherited by all assets)
SELECT * FROM master_attribute
WHERE source = 'FROM_TEMPLATE'
  AND asset_master_id = 'ASSET-001'
  AND template_attribute_id IS NOT NULL;

-- Asset custom attribute (only on this asset)
SELECT * FROM master_attribute
WHERE source = 'NEW'
  AND asset_master_id = 'ASSET-001'
  AND template_attribute_id IS NULL;
```

### Benefits
- ✅ Clear inheritance chain (track source)
- ✅ Easy to override (set is_overridden = TRUE)
- ✅ Revert to template (delete MASTER_ATTRIBUTE row)

---

## Pattern 4: JSONB Inline Validation Rules

### Problem
How to store flexible validation rules without a separate table?

### Solution
Store rules as **inline JSONB array** in ATTRIBUTE_DEF:

```json
{
  "validation_rules": [
    {
      "type": "REQUIRED",
      "message": "Rated Capacity is required",
      "severity": "ERROR",
      "active": true
    },
    {
      "type": "MIN_VALUE",
      "value": "10",
      "message": "Must be ≥ 10",
      "severity": "ERROR",
      "active": true
    },
    {
      "type": "MAX_VALUE",
      "value": "500",
      "message": "Cannot exceed 500",
      "severity": "WARNING",
      "active": true
    }
  ]
}
```

### Benefits
- ✅ **Flexible:** Add rule types without schema changes
- ✅ **Self-contained:** Attribute definition includes its rules
- ✅ **Fast:** No JOINs needed to fetch rules
- ✅ **Queryable:** PostgreSQL JSONB supports operators (@>, ->>)

### Rule Evaluation

```
For each active rule:
  ├─ Evaluate against value
  ├─ If ERROR severity → Stop, mark INVALID, don't save
  ├─ If WARNING severity → Continue, mark WARNING
  └─ If pass → Continue
  
Final status:
  ├─ Has ERROR → INVALID
  ├─ Has WARNING → WARNING
  └─ No failures → VALID
```

---

## Pattern 5: Handler Pipeline (Async Processing)

### Problem
How to process events without blocking user requests?

### Solution
Implement **handler pipeline** with async event bus:

```
API Request → Event Published → Async Handler → Audit Generated → Notification Sent
(sync)            (sync)           (async)          (async)          (async)
```

### Handlers

| Handler | Consumes | Produces | Latency |
|---------|----------|----------|---------|
| **AssetEventHandler** | ASSET_EVENT_STORE | ASSET_AUDIT | 100–500 ms |
| **NotificationHandler** | ASSET_AUDIT | NOTIFICATION_LOG | 50–200 ms |
| **DocumentHandler** | File upload | DOCUMENT_SCAN | 1–60 s (Votiro) |

### Code Structure

```csharp
// 1. User creates asset (sync)
public async Task<AssetDto> CreateAssetAsync(CreateAssetRequest request)
{
    var asset = new AssetMaster { /* ... */ };
    await _db.SaveAsync(asset);
    
    // 2. Publish event (sync, just queuing)
    await _eventBus.PublishAsync(new AssetEvent 
    { 
        Type = EventType.Create,
        Details = asset.ToJson()
    });
    
    // 3. Return immediately (event processing is async)
    return MapToDto(asset);
}

// 4. Handler consumes event (async, background)
public class AssetEventHandler : IEventHandler<AssetEvent>
{
    public async Task HandleAsync(AssetEvent @event)
    {
        var audit = GenerateAudit(@event);
        await _auditDb.InsertAsync(audit);
        await _notificationHandler.DispatchAsync(audit);
    }
}
```

### Benefits
- ✅ **Fast:** User gets response immediately (< 500 ms)
- ✅ **Resilient:** Event processing fails don't impact user
- ✅ **Scalable:** Process events in parallel
- ✅ **Auditable:** Full history in ASSET_AUDIT

---

## Pattern 6: Notification Dispatch (Multi-Channel)

### Problem
How to send notifications via multiple channels (UI, email, webhook)?

### Solution
Use **strategy pattern** with channel implementations:

```
NotificationHandler
    ├─ UINotificationChannel (SignalR push)
    ├─ EmailNotificationChannel (SMTP)
    └─ WebhookNotificationChannel (HTTP POST)
```

### Code

```csharp
public class NotificationHandler
{
    private readonly Dictionary<Channel, INotificationChannel> _channels;
    
    public async Task DispatchAsync(NotificationRequest request)
    {
        var tasks = request.Recipients
            .Select(recipient => DispatchToRecipientAsync(request, recipient))
            .ToList();
        
        await Task.WhenAll(tasks);
    }
    
    private async Task DispatchToRecipientAsync(NotificationRequest request, string recipient)
    {
        var recipientChannels = ResolveChannels(recipient); // UI, email, webhook
        
        foreach (var channel in recipientChannels)
        {
            try
            {
                var success = await _channels[channel]
                    .SendAsync(request.Message, recipient);
                
                await _notificationLog.LogAsync(new NotificationLog 
                { 
                    Channel = channel,
                    Status = success ? Status.Sent : Status.Failed
                });
            }
            catch (Exception ex)
            {
                // Log failure, implement retry
            }
        }
    }
}
```

### Benefits
- ✅ **Flexible:** Add new channels without changing core logic
- ✅ **Parallel:** Send to multiple channels simultaneously
- ✅ **Traceable:** Log each channel's result in NOTIFICATION_LOG
- ✅ **Resilient:** One channel failure doesn't block others

---

## Pattern 7: Quarantine + Scan (File Security)

### Problem
How to safely handle untrusted file uploads?

### Solution
Implement **quarantine + scan pattern** with Votiro CDR:

```
Upload → Quarantine → Votiro Scan → Decision → Archive or Delete
(temp)      (temp)        (async)     (clean?)     (secure)
```

### Flow

```
User uploads file
    ↓ (1)
Store in quarantine bucket (PENDING scan)
    ↓ (2)
Submit to Votiro API for CDR scan
    ↓ (3)
Receive webhook callback
    ├─ If CLEAN → Move to permanent storage
    ├─ If INFECTED → Delete from quarantine, notify user
    └─ If FAILED → Retry or notify admin
```

### Database Tracking

```sql
-- ATTACHMENT: tracks file metadata + scan status
INSERT INTO attachment (file_name, scan_status, uploaded_at)
VALUES ('Manual.pdf', 'PENDING', NOW());

-- DOCUMENT_SCAN: tracks scan result
INSERT INTO document_scan (attachment_id, scan_engine, scan_status, scan_reference_id)
VALUES ('ATT-001', 'VOTIRO', 'PENDING', 'VOTIRO-REF-123');

-- Later: Scan completes
UPDATE attachment SET scan_status = 'CLEAN' WHERE id = 'ATT-001';
UPDATE document_scan SET 
  scan_status = 'CLEAN',
  sanitized_file_path = '/docs/ATT-001/Manual.pdf',
  scanned_at = NOW()
WHERE attachment_id = 'ATT-001';
```

### Benefits
- ✅ **Secure:** Files cannot be accessed until scan passes
- ✅ **Traceable:** Full scan history in DOCUMENT_SCAN
- ✅ **Compliant:** CDR sanitizes threats automatically
- ✅ **Transparent:** User sees scan status in real-time

---

## Pattern 8: Materialized Path (Hierarchy Navigation)

### Problem
How to efficiently query asset hierarchy without recursive queries?

### Solution
Store **materialized path** in each row:

```sql
-- Asset hierarchy
AG#####
  ├─ AG#####.24
  │   ├─ AG#####.24.141
  │   │   ├─ AG#####.24.141.528
  │   │   │   └─ AG#####.24.141.528.119

-- Query: Get all children of AG#####.24
SELECT * FROM asset_template 
WHERE hierarchy_path LIKE 'AG#####.24.%'
ORDER BY hierarchy_path;
```

### Benefits
- ✅ **Fast:** No recursive queries (LIKE on indexed path)
- ✅ **Simple:** One WHERE clause instead of CTE
- ✅ **Scalable:** Works at any depth
- ✅ **Debuggable:** Path visible in every row

---

## Design Principles

| Principle | How It's Applied |
|-----------|-----------------|
| **Single Responsibility** | Each handler has one job (event → audit, audit → notify, file → scan) |
| **DRY (Don't Repeat Yourself)** | ATTRIBUTE_DEF shared across all assets, validation rules in one place |
| **Immutability** | ASSET_EVENT_STORE is append-only, cannot be modified |
| **Eventual Consistency** | Event → Audit → Notification are async, may take seconds |
| **Separation of Concerns** | Validation rules separate from asset creation logic |
| **YAGNI (You Aren't Gonna Need It)** | No cross-field rules (Phase 2), no bulk operations (Phase 2) |

---

## Comparison: Before vs. After Design

### Before (Monolithic)
```
Asset saved → Inline validation → Inline audit → Inline email
               (blocking)         (blocking)      (blocking)

Problems:
- User waits for email to send (slow)
- Validation logic is procedural (hard to change)
- No audit trail if email fails
- Cannot replay events
```

### After (Event-Driven)
```
Asset saved → Event published → Handler consumes → Audit generated → Email sent
(fast)       (async)              (async)           (async)           (async)

Benefits:
- User gets response in <500 ms
- Validation rules are declarative (easy to change)
- Full audit trail even if email fails
- Can replay/rescan events
```

---

## Next Steps

1. **Implement handlers:** See [7_HANDLERS.md](./7_HANDLERS.md)
2. **Configure rules:** See [4_VALIDATION_RULES.md](./4_VALIDATION_RULES.md)
3. **Set up databases:** See [1_ARCHITECTURE.md](./1_ARCHITECTURE.md)

---

**File:** 2_DESIGN_PATTERNS.md | **Lines:** ~200
