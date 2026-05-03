<!-- filepath: DISPATCH_SIMPLIFICATION.md -->
# Dispatch Audit Simplification — Migration Guide

> **Status:** Complete | **Date:** May 3, 2026 | **Impact:** Remove 2+ redundant tables

---

## The Change

**Before:** Multiple tables for handler tracking
- `ASSET_AUDIT` (business events)
- `DISPATCH_AUDIT` (handler execution) — **REMOVED**
- `NOTIFICATION_LOG` (notification history) — **REMOVED**

**After:** Single unified table
- `ASSET_AUDIT` (everything)

---

## Why Simplify?

### Pain Points with Multiple Tables

| Problem | Impact |
|---------|--------|
| **Data Duplication** | Same event logged in 3 places |
| **Complexity** | Complex UNION queries to get full picture |
| **Sync Issues** | Tables can get out of sync |
| **Maintenance Burden** | 3 schemas, 3 indexing strategies |
| **Performance** | Joins + unions slower than single table |

### Benefits of Unified Approach

| Benefit | Value |
|---------|-------|
| **Single Source of Truth** | One table, one story |
| **Simpler Queries** | No joins/unions needed |
| **Faster Development** | Less code to write |
| **Easier Debugging** | All activities in one place |
| **Better Consistency** | No sync issues possible |

---

## Migration Path

### Step 1: Update ASSET_AUDIT Schema

Add these columns to `ASSET_AUDIT`:

```sql
ALTER TABLE asset_audit ADD COLUMN handler VARCHAR(50);
ALTER TABLE asset_audit ADD COLUMN detail JSONB;

-- Create index for queries
CREATE INDEX idx_handler ON asset_audit(handler);
```

### Step 2: Deprecate Old Tables

Handlers no longer write to:
- ~~`dispatch_audit`~~ (drop in next release)
- ~~`notification_log`~~ (drop in next release)

Keep read-access for 1 release cycle for reporting.

### Step 3: Update Handler Code

**Old pattern:**
```csharp
await _auditRepo.CreateAuditAsync(audit);
await _dispatchAuditRepo.LogDispatchAsync(handler, time, status);
```

**New pattern:**
```csharp
await _auditRepo.CreateAuditAsync(new AssetAudit
{
    Handler = "AssetEventHandler",
    Detail = new { execution_time_ms = 150, result = "success" }
});
```

---

## Data Model

### ASSET_AUDIT (Updated)

```sql
CREATE TABLE asset_audit (
    id UUID PRIMARY KEY,
    asset_master_id UUID NOT NULL,
    event_id UUID REFERENCES asset_event_store(event_id),
    
    -- NEW: Handler that created this record
    handler VARCHAR(50),  -- AssetEventHandler, NotificationHandler, etc.
    
    change_type VARCHAR(50) NOT NULL,  -- CREATED, UPDATED, NOTIFIED, SCANNED, etc.
    message TEXT NOT NULL,  -- Human readable
    
    -- NEW: Structured data for debugging/monitoring
    detail JSONB,  
    -- Example:
    -- {
    --   "execution_time_ms": 150,
    --   "result": "success|failed",
    --   "error": "null or error message",
    --   "field_name": "year",
    --   "old_value": "2023",
    --   "new_value": "2024"
    -- }
    
    status VARCHAR(20),  -- CREATED, DISPATCHED, COMPLETED, FAILED
    changed_by UUID,
    changed_by_name VARCHAR(255),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_event_id (event_id),
    INDEX idx_handler (handler),  -- NEW
    INDEX idx_changed_at (changed_at DESC)
);
```

---

## Query Examples

### Before (Multiple Tables)

```sql
-- Had to UNION multiple tables
SELECT 'AUDIT' as type, handler_name, message, created_at
FROM asset_audit WHERE event_id = 'e123'
UNION ALL
SELECT 'DISPATCH' as type, handler_type, message, completed_at
FROM dispatch_audit WHERE event_id = 'e123'
UNION ALL
SELECT 'NOTIFICATION' as type, channel, message, sent_at
FROM notification_log WHERE event_id = 'e123'
ORDER BY created_at ASC;
```

### After (Single Table)

```sql
-- Simple, single table query
SELECT handler, change_type, message, created_at
FROM asset_audit
WHERE event_id = 'e123'
ORDER BY created_at ASC;
```

---

## Handler Implementation

### AssetEventHandler

```csharp
public class AssetEventHandler : IConsumer<AssetCreatedEvent>
{
    private readonly IRepository _repo;
    
    public async Task Consume(ConsumeContext<AssetCreatedEvent> context)
    {
        var @event = context.Message;
        var startTime = DateTime.UtcNow;
        
        try 
        {
            // Process event
            var assetId = @event.AssetMasterId;
            
            // Single audit log entry
            await _repo.CreateAuditAsync(new AssetAudit
            {
                AssetMasterId = @event.AssetMasterId,
                EventId = @event.EventId,
                Handler = "AssetEventHandler",
                ChangeType = "CREATED",
                Message = $"Created Asset {assetId}",
                Status = "COMPLETED",
                Detail = new 
                { 
                    execution_time_ms = (int)(DateTime.UtcNow - startTime).TotalMilliseconds,
                    asset_code = @event.AssetCode,
                    result = "success"
                },
                ChangedBy = @event.EventUserId,
                ChangedByName = @event.EventUserName
            });
        }
        catch (Exception ex)
        {
            await _repo.CreateAuditAsync(new AssetAudit
            {
                AssetMasterId = @event.AssetMasterId,
                EventId = @event.EventId,
                Handler = "AssetEventHandler",
                ChangeType = "CREATED",
                Message = "Error processing asset creation",
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
```

### NotificationHandler

```csharp
public class NotificationHandler : IConsumer<AssetUpdatedEvent>
{
    private readonly IRepository _repo;
    private readonly INotificationService _service;
    
    public async Task Consume(ConsumeContext<AssetUpdatedEvent> context)
    {
        var @event = context.Message;
        var startTime = DateTime.UtcNow;
        
        try 
        {
            var channels = ResolveChannels(@event);
            await _service.SendToChannelsAsync(channels);
            
            // Single entry in ASSET_AUDIT
            await _repo.CreateAuditAsync(new AssetAudit
            {
                AssetMasterId = @event.AssetMasterId,
                EventId = @event.EventId,
                Handler = "NotificationHandler",
                ChangeType = "NOTIFIED",
                Message = $"Sent {channels.Count} notifications",
                Status = "COMPLETED",
                Detail = new 
                { 
                    channels = channels.Select(c => c.Name).ToList(),
                    execution_time_ms = (int)(DateTime.UtcNow - startTime).TotalMilliseconds,
                    result = "success"
                }
            });
        }
        catch (Exception ex)
        {
            await _repo.CreateAuditAsync(new AssetAudit
            {
                Handler = "NotificationHandler",
                ChangeType = "NOTIFIED",
                Status = "FAILED",
                Detail = new { error = ex.Message, result = "failed" }
            });
        }
    }
}
```

---

## Monitoring & Alerts

### Handler Performance

```sql
-- Which handlers are slow?
SELECT 
  handler,
  COUNT(*) as invocations,
  AVG((detail->>'execution_time_ms')::int) as avg_ms,
  MAX((detail->>'execution_time_ms')::int) as max_ms
FROM asset_audit
WHERE created_at > NOW() - INTERVAL '1 hour'
  AND detail IS NOT NULL
GROUP BY handler
ORDER BY avg_ms DESC;
```

### Failure Rate

```sql
-- Handler failure rate
SELECT 
  handler,
  detail->>'result' as result,
  COUNT(*) as count
FROM asset_audit
WHERE created_at > NOW() - INTERVAL '1 day'
  AND detail->>'result' IS NOT NULL
GROUP BY handler, detail->>'result';
```

### Complete Event Timeline

```sql
-- Full story of one event, one query
SELECT 
  handler, 
  change_type,
  message, 
  detail->>'execution_time_ms' as ms,
  created_at
FROM asset_audit
WHERE event_id = 'e123'
ORDER BY created_at ASC;
```

---

## FAQ

**Q: What if I need historical dispatch_audit data?**
A: Keep the old tables in read-only mode for 1 release cycle. Create a one-time migration script to archive old data if needed.

**Q: How do I track notification channels now?**
A: Include in the `detail` JSONB field: `{ "channels": ["UI_PUSH", "EMAIL"], ... }`

**Q: Will this break reporting?**
A: No. Update report queries to use single ASSET_AUDIT table instead of joining 3 tables.

**Q: How do I measure total dispatch time?**
A: Find the MAX(created_at) for all audit entries for one event.

---

## Migration Checklist

- [ ] Update ASSET_AUDIT schema (add `handler`, `detail`)
- [ ] Update AssetEventHandler code
- [ ] Update NotificationHandler code
- [ ] Update DocumentHandler code
- [ ] Update monitoring queries
- [ ] Update reporting dashboards
- [ ] Test end-to-end flow
- [ ] Deploy to staging
- [ ] Verify all queries work
- [ ] Deploy to production
- [ ] Monitor for 1 week
- [ ] Drop old tables (next release)

---

**Last Updated:** May 3, 2026 | **Version:** 1.0
