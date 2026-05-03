<!-- filepath: 02_DISPATCH_AUDIT_EXPLAINED.md -->
# Dispatch Audit Simplified — Single Table Pattern

> **Purpose:** Simplified audit tracking without maintaining separate dispatch_audit table

---

## What is Dispatch Tracking?

Instead of maintaining a separate `dispatch_audit` table, handlers log directly to **ASSET_AUDIT** with a `handler` field.

**It answers:** *"What happened to this event?"* (Not: "which handler processed it")

### Simplified Approach
- ✅ Handlers log to ASSET_AUDIT (single source of truth)
- ✅ No separate dispatch_audit table to maintain
- ✅ Include handler type in audit message
- ✅ Use structured data in `detail` field for debugging
- ✅ Status field tracks: CREATED → DISPATCHED → COMPLETED/FAILED

### Flow
```
Event → EventDispatcher → [Parallel Handlers]
                            ├→ Handler 1 → LogAudit (ASSET_AUDIT)
                            ├→ Handler 2 → LogAudit (ASSET_AUDIT)
                            └→ Handler 3 → LogAudit (ASSET_AUDIT)
```

---

## Database Schema (Simplified)

**ASSET_AUDIT becomes the single audit log:**

```sql
CREATE TABLE asset_audit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_master_id UUID NOT NULL REFERENCES asset_master(id),
    event_id UUID REFERENCES asset_event_store(event_id),
    
    handler VARCHAR(50),  
    -- Handler that created this audit: AssetEventHandler, NotificationHandler, etc.
    
    change_type VARCHAR(50) NOT NULL,  
    -- CREATED, UPDATED, DELETED, NOTIFIED, SCANNED, etc.
    
    message TEXT NOT NULL,  
    -- Human-readable message
    
    detail JSONB,  
    -- Structured data: {handler_time_ms, status, error, etc}
    
    status VARCHAR(20),  
    -- CREATED, DISPATCHED, COMPLETED, FAILED
    
    changed_by UUID,
    changed_by_name VARCHAR(255),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_event_id (event_id),
    INDEX idx_handler (handler),
    INDEX idx_changed_at (changed_at DESC)
);
```

**No dispatch_audit table needed** — use ASSET_AUDIT for everything.

---

## Example #1: Asset Creation

**Scenario:** User creates Asset ASSET001

### Single Audit Log (ASSET_AUDIT only)

**Event Created:**
```json
{
  "event_id": "e123",
  "event_type": "AssetCreated",
  "asset_master_id": "asset-001",
  "created_at": "2026-05-03 09:00:00.000"
}
```

**AssetEventHandler logs to ASSET_AUDIT:**
```json
{
  "id": "audit-001",
  "asset_master_id": "asset-001",
  "event_id": "e123",
  "handler": "AssetEventHandler",
  "change_type": "CREATED",
  "message": "Created Asset ASSET001",
  "status": "COMPLETED",
  "detail": {
    "asset_code": "ASSET001",
    "asset_name": "Motor #123",
    "execution_time_ms": 150,
    "result": "success"
  },
  "changed_by": "user-123",
  "changed_by_name": "user@company.com",
  "created_at": "2026-05-03 09:00:00.050"
}
```

**NotificationHandler logs to ASSET_AUDIT:**
```json
{
  "id": "audit-002",
  "asset_master_id": "asset-001",
  "event_id": "e123",
  "handler": "NotificationHandler",
  "change_type": "NOTIFIED",
  "message": "Sent UI push notification",
  "status": "COMPLETED",
  "detail": {
    "channel": "UI_PUSH",
    "recipient": "user@company.com",
    "execution_time_ms": 45,
    "result": "success"
  },
  "created_at": "2026-05-03 09:00:00.095"
}
```

**Query all activities for one event:**
```sql
SELECT handler, change_type, message, detail->>'execution_time_ms' as time_ms, changed_at
FROM asset_audit
WHERE event_id = 'e123'
ORDER BY created_at ASC;

RESULT:
┌────────────────────┬──────────────┬───────────────────────────┬──────────┬──────────────┐
│ handler            │ change_type  │ message                   │ time_ms  │ changed_at   │
├────────────────────┼──────────────┼───────────────────────────┼──────────┼──────────────┤
│ AssetEventHandler   │ CREATED      │ Created Asset ASSET001    │ 150      │ 09:00:00.050 │
│ NotificationHandler │ NOTIFIED     │ Sent UI push notification │ 45       │ 09:00:00.095 │
└────────────────────┴──────────────┴───────────────────────────┴──────────┴──────────────┘
```

---

## Example #2: Asset Update (Parallel Handlers)

**Scenario:** User updates Asset ASSET001 (year: 2023 → 2024)

### Multiple Handlers in ASSET_AUDIT

**AssetEventHandler logs:**
```json
{
  "id": "audit-201",
  "asset_master_id": "asset-001",
  "event_id": "e456",
  "handler": "AssetEventHandler",
  "change_type": "UPDATED",
  "message": "Updated Asset ASSET001, field: year",
  "status": "COMPLETED",
  "detail": {
    "field_name": "year",
    "old_value": "2023",
    "new_value": "2024",
    "execution_time_ms": 120
  },
  "changed_by_name": "user2@company.com",
  "created_at": "2026-05-03 10:30:00.050"
}
```

**NotificationHandler logs (async, runs in parallel):**
```json
{
  "id": "audit-202",
  "asset_master_id": "asset-001",
  "event_id": "e456",
  "handler": "NotificationHandler",
  "change_type": "NOTIFIED",
  "message": "Sent notifications via 2 channels",
  "status": "COMPLETED",
  "detail": {
    "channels": ["UI_PUSH", "EMAIL"],
    "recipients": 1,
    "execution_time_ms": 85
  },
  "created_at": "2026-05-03 10:30:00.135"
}
```

**Query to see both handlers' work:**
```sql
SELECT handler, change_type, message, detail->>'execution_time_ms' as time_ms
FROM asset_audit
WHERE event_id = 'e456'
ORDER BY created_at ASC;

RESULT:
┌────────────────────┬──────────────┬──────────────────────────────────────┬─────────┐
│ handler            │ change_type  │ message                              │ time_ms │
├────────────────────┼──────────────┼──────────────────────────────────────┼─────────┤
│ AssetEventHandler   │ UPDATED      │ Updated Asset ASSET001, field: year  │ 120     │
│ NotificationHandler │ NOTIFIED     │ Sent notifications via 2 channels    │ 85      │
└────────────────────┴──────────────┴──────────────────────────────────────┴─────────┘

Total execution time: max(120, 85) = 120ms (parallel execution)
```

---

## Example #3: Multiple Handlers (3-Way Parallel)

**Scenario:** Single event processed by AssetEventHandler, NotificationHandler, DocumentHandler

### All Logged in ASSET_AUDIT

**AssetEventHandler:**
```json
{
  "id": "audit-301",
  "asset_master_id": "asset-001",
  "event_id": "e789",
  "handler": "AssetEventHandler",
  "change_type": "CREATED",
  "message": "Asset event logged",
  "detail": { "execution_time_ms": 145, "result": "success" },
  "created_at": "2026-05-03 11:00:00.145"
}
```

**NotificationHandler:**
```json
{
  "id": "audit-302",
  "asset_master_id": "asset-001",
  "event_id": "e789",
  "handler": "NotificationHandler",
  "change_type": "NOTIFIED",
  "message": "Notifications sent",
  "detail": { "execution_time_ms": 95, "channels": 2, "result": "success" },
  "created_at": "2026-05-03 11:00:00.095"
}
```

**DocumentHandler:**
```json
{
  "id": "audit-303",
  "asset_master_id": "asset-001",
  "event_id": "e789",
  "handler": "DocumentHandler",
  "change_type": "SCANNED",
  "message": "Document scan initiated",
  "detail": { "execution_time_ms": 200, "correlation_id": "votiro-xyz", "result": "success" },
  "created_at": "2026-05-03 11:00:00.200"
}
```

**Single query shows all three:**
```sql
SELECT handler, change_type, detail->>'execution_time_ms' as time_ms, created_at
FROM asset_audit
WHERE event_id = 'e789'
ORDER BY created_at ASC;

RESULT:
┌────────────────────┬──────────────┬─────────┬──────────────────┐
│ handler            │ change_type  │ time_ms │ created_at       │
├────────────────────┼──────────────┼─────────┼──────────────────┤
│ NotificationHandler │ NOTIFIED     │ 95      │ 11:00:00.095     │
│ AssetEventHandler   │ CREATED      │ 145     │ 11:00:00.145     │
│ DocumentHandler     │ SCANNED      │ 200     │ 11:00:00.200     │
└────────────────────┴──────────────┴─────────┴──────────────────┘

No separate table to join!
```

---

## Example #4: Document Upload & Scan

**Scenario:** User uploads file, DocumentHandler polls Votiro, returns result

### Document Tracking in ASSET_AUDIT

**Initial upload event:**
```json
{
  "id": "audit-401",
  "asset_master_id": "asset-001",
  "event_id": "e345",
  "handler": "DocumentHandler",
  "change_type": "SCANNED",
  "message": "File uploaded to Votiro: Manual.pdf",
  "status": "COMPLETED",
  "detail": {
    "file_name": "Manual.pdf",
    "correlation_id": "votiro-corr-xyz789",
    "file_size": 2048000,
    "execution_time_ms": 2650,
    "polling_attempts": 3
  },
  "created_at": "2026-05-03 13:00:02.700"
}
```

**After scan completes:**
```json
{
  "id": "audit-402",
  "asset_master_id": "asset-001",
  "event_id": "e345",
  "handler": "NotificationHandler",
  "change_type": "NOTIFIED",
  "message": "Scan complete: Manual.pdf is CLEAN",
  "status": "COMPLETED",
  "detail": {
    "channel": "UI_PUSH",
    "threat_detected": false,
    "execution_time_ms": 25
  },
  "created_at": "2026-05-03 13:00:02.750"
}
```

**Complete audit trail for document:**
```sql
SELECT handler, change_type, message, detail
FROM asset_audit
WHERE event_id = 'e345'
ORDER BY created_at ASC;
```

---

## Simplified Queries

Instead of joining multiple tables, query ASSET_AUDIT directly:

### Query 1: All Activities for One Event
```sql
-- Single table, no joins
SELECT 
  handler, 
  change_type, 
  message,
  detail->>'execution_time_ms' as time_ms,
  status
FROM asset_audit
WHERE event_id = 'e123'
ORDER BY created_at ASC;
```

### Query 2: Handler Performance
```sql
-- Fastest/slowest handlers
SELECT 
  handler,
  COUNT(*) as invocations,
  AVG((detail->>'execution_time_ms')::int) as avg_time_ms,
  MAX((detail->>'execution_time_ms')::int) as max_time_ms
FROM asset_audit
WHERE created_at > NOW() - INTERVAL '1 day'
  AND detail IS NOT NULL
GROUP BY handler
ORDER BY avg_time_ms DESC;
```

### Query 3: Failed Operations
```sql
-- Find failures across all handlers
SELECT 
  handler,
  change_type,
  message,
  detail->>'result' as result,
  created_at
FROM asset_audit
WHERE created_at > NOW() - INTERVAL '1 hour'
  AND (status = 'FAILED' OR detail->>'result' = 'failed')
ORDER BY created_at DESC;
```

### Query 4: Notification Summary
```sql
-- Count notifications by channel
SELECT 
  handler,
  detail->>'channel' as channel,
  COUNT(*) as count
FROM asset_audit
WHERE handler = 'NotificationHandler'
  AND created_at > NOW() - INTERVAL '1 day'
GROUP BY handler, detail->>'channel';
```

---

## Benefits of Simplified Approach

| Aspect | Before (3+ tables) | After (1 table) |
|--------|-------------------|-----------------|
| **Tables** | ASSET_AUDIT, DISPATCH_AUDIT, NOTIFICATION_LOG | ASSET_AUDIT only |
| **Joins** | Multiple for complete view | None (single table) |
| **Queries** | Complex UNION statements | Simple SELECT |
| **Maintenance** | 3+ schemas to maintain | 1 schema |
| **Data consistency** | Can get out of sync | Guaranteed consistency |
| **Query performance** | Slower (joins + unions) | Fast (single table) |

---

## Handler Implementation Pattern

Each handler follows this pattern:

```csharp
public class AssetEventHandler : IConsumer<AssetCreatedEvent>
{
    private readonly IRepository _repo;
    private readonly ILogger _logger;
    
    public async Task Consume(ConsumeContext<AssetCreatedEvent> context)
    {
        var @event = context.Message;
        var startTime = DateTime.UtcNow;
        
        try 
        {
            // Do work
            await ProcessEventAsync(@event);
            
            // Log to ASSET_AUDIT (single table)
            await _repo.CreateAuditAsync(new AssetAudit
            {
                EventId = @event.EventId,
                Handler = "AssetEventHandler",
                ChangeType = "CREATED",
                Message = $"Processed asset {id}",
                Status = "COMPLETED",
                Detail = new 
                { 
                    execution_time_ms = (int)(DateTime.UtcNow - startTime).TotalMilliseconds,
                    result = "success"
                }
            });
        }
        catch (Exception ex)
        {
            await _repo.CreateAuditAsync(new AssetAudit
            {
                EventId = @event.EventId,
                Handler = "AssetEventHandler",
                ChangeType = "CREATED",
                Message = $"Error processing asset",
                Status = "FAILED",
                Detail = new 
                { 
                    execution_time_ms = (int)(DateTime.UtcNow - startTime).TotalMilliseconds,
                    error = ex.Message,
                    result = "failed"
                }
            });
            throw;
        }
    }
}

---

**Last Updated:** May 3, 2026 | **Version:** 1.0
