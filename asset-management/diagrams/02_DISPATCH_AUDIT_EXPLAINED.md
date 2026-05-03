<!-- filepath: 02_DISPATCH_AUDIT_EXPLAINED.md -->
# LogDispatchAudit Explained — Sample Database Entries

> **Purpose:** Understand what LogDispatchAudit does and see sample database entries for each sequence diagram

---

## What is LogDispatchAudit?

`LogDispatchAudit` is a tracking mechanism that records **which handlers processed which events**, including execution time, status, and results.

**It answers:** *"What happened to this event and which handlers touched it?"*

### Purpose
- ✅ Track event dispatcher workflow completion
- ✅ Monitor handler execution for debugging
- ✅ Create audit trail of async handler invocations
- ✅ Identify bottlenecks or failures in the handler pipeline
- ✅ Measure performance of individual handlers

### Flow
```
Event → EventDispatcher → [Parallel Handlers]
                            ├→ Handler 1 (LogAudit) → LogDispatchAudit
                            ├→ Handler 2 (LogNotification) → LogDispatchAudit
                            └→ Handler 3 (LogScan) → LogDispatchAudit
```

---

## Database Schema

```sql
CREATE TABLE dispatch_audit (
    dispatch_audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL REFERENCES asset_event_store(event_id),
    handler_type VARCHAR(50) NOT NULL,  
    -- Values: AssetEventHandler, NotificationHandler, DocumentHandler
    
    handler_status VARCHAR(20) NOT NULL,  
    -- Values: Success, Failed, Timeout, PartialSuccess
    
    execution_time_ms INT NOT NULL,  
    -- How long the handler took to execute
    
    error_message VARCHAR(500),  
    -- If failed, what was the error?
    
    dispatched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  
    -- When EventDispatcher sent to handler
    
    completed_at TIMESTAMP,  
    -- When handler finished executing
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_event_id (event_id),
    INDEX idx_handler_type (handler_type),
    INDEX idx_dispatched_at (dispatched_at DESC)
);
```

---

## Sequence Diagram #1: Asset Creation

**Scenario:** User creates Asset ASSET001

### Step-by-Step Database Entries

#### Step 1: Event Created
**ASSET_EVENT_STORE:**
```json
{
  "event_id": "e123",
  "event_type": 1,        // AssetCreated enum
  "event_details": {
    "asset_master_id": "asset-001",
    "asset_code": "ASSET001",
    "name": "Motor #123",
    "components": [],
    "attachments": []
  },
  "created_at": "2026-05-03 09:00:00.000",
  "created_by": "user@company.com"
}
```

#### Step 2: AssetEventHandler Processes Event
**ASSET_AUDIT (Generated):**
```json
{
  "audit_id": "audit-001",
  "event_id": "e123",
  "handler_name": "AssetEventHandler",
  "audit_message": "Created Asset ASSET001",
  "detail": {
    "asset_code": "ASSET001",
    "asset_name": "Motor #123",
    "components_added": 0,
    "attachments_added": 0
  },
  "status": "DISPATCHED",
  "created_at": "2026-05-03 09:00:00.050"
}
```

#### Step 3: EventDispatcher Logs Handler Completion
**DISPATCH_AUDIT (Logged by EventDispatcher):**
```json
{
  "dispatch_audit_id": "da-001",
  "event_id": "e123",
  "handler_type": "AssetEventHandler",
  "handler_status": "Success",
  "execution_time_ms": 150,    // Handler took 150ms
  "error_message": null,
  "dispatched_at": "2026-05-03 09:00:00.050",
  "completed_at": "2026-05-03 09:00:00.200",
  "created_at": "2026-05-03 09:00:00.200"
}
```

#### Step 4: NotificationHandler Sends Notification
**NOTIFICATION_LOG (Generated):**
```json
{
  "notification_id": "notif-001",
  "event_id": "e123",
  "channel": "UI_PUSH",
  "recipient": "user@company.com",
  "message": "✅ Asset ASSET001 created",
  "status": "SENT",
  "sent_at": "2026-05-03 09:00:00.250",
  "created_at": "2026-05-03 09:00:00.250"
}
```

---

## Sequence Diagram #2: Asset Update

**Scenario:** User updates Asset ASSET001 (year: 2023 → 2024)

### Database Entries

#### Event & Audit
**ASSET_EVENT_STORE:**
```json
{
  "event_id": "e456",
  "event_type": 2,        // AssetUpdated enum
  "event_details": {
    "asset_master_id": "asset-001",
    "asset_code": "ASSET001",
    "changes": {
      "year": { "old": 2023, "new": 2024 }
    },
    "modified_fields": ["year"]
  },
  "created_at": "2026-05-03 10:30:00.000",
  "created_by": "user2@company.com"
}
```

**ASSET_AUDIT:**
```json
{
  "audit_id": "audit-002",
  "event_id": "e456",
  "handler_name": "AssetEventHandler",
  "audit_message": "Updated Asset ASSET001, Modified Fields: year",
  "detail": {
    "asset_code": "ASSET001",
    "field_name": "year",
    "old_value": "2023",
    "new_value": "2024",
    "changed_by": "user2@company.com"
  },
  "status": "DISPATCHED",
  "created_at": "2026-05-03 10:30:00.050"
}
```

#### Parallel Handler Tracking
**DISPATCH_AUDIT (AssetEventHandler):**
```json
{
  "dispatch_audit_id": "da-002a",
  "event_id": "e456",
  "handler_type": "AssetEventHandler",
  "handler_status": "Success",
  "execution_time_ms": 120,
  "error_message": null,
  "dispatched_at": "2026-05-03 10:30:00.050",
  "completed_at": "2026-05-03 10:30:00.170",
  "created_at": "2026-05-03 10:30:00.170"
}
```

**DISPATCH_AUDIT (NotificationHandler):**
```json
{
  "dispatch_audit_id": "da-002b",
  "event_id": "e456",
  "handler_type": "NotificationHandler",
  "handler_status": "Success",
  "execution_time_ms": 85,
  "error_message": null,
  "dispatched_at": "2026-05-03 10:30:00.050",
  "completed_at": "2026-05-03 10:30:00.135",
  "created_at": "2026-05-03 10:30:00.135"
}
```

#### Multi-Channel Notifications
**NOTIFICATION_LOG (UI Push):**
```json
{
  "notification_id": "notif-002a",
  "event_id": "e456",
  "channel": "UI_PUSH",
  "recipient": "user@company.com",
  "message": "Asset ASSET001 updated: year changed (2023 → 2024)",
  "status": "SENT",
  "sent_at": "2026-05-03 10:30:00.150",
  "created_at": "2026-05-03 10:30:00.135"
}
```

**NOTIFICATION_LOG (Email):**
```json
{
  "notification_id": "notif-002b",
  "event_id": "e456",
  "channel": "EMAIL",
  "recipient": "user@company.com",
  "message": "Asset ASSET001 updated: year changed (2023 → 2024)",
  "status": "SENT",
  "retry_count": 0,
  "sent_at": "2026-05-03 10:30:00.350",
  "created_at": "2026-05-03 10:30:00.135"
}
```

---

## Sequence Diagram #3: EventDispatcher Coordination

**Scenario:** Single event processed by all 3 handlers in parallel

### Database Entries

**DISPATCH_AUDIT - All 3 Handlers:**

*Handler #1 - AssetEventHandler (Fastest):*
```json
{
  "dispatch_audit_id": "da-003a",
  "event_id": "e789",
  "handler_type": "AssetEventHandler",
  "handler_status": "Success",
  "execution_time_ms": 145,
  "error_message": null,
  "dispatched_at": "2026-05-03 11:00:00.000",
  "completed_at": "2026-05-03 11:00:00.145"
}
```

*Handler #2 - NotificationHandler (Medium):*
```json
{
  "dispatch_audit_id": "da-003b",
  "event_id": "e789",
  "handler_type": "NotificationHandler",
  "handler_status": "Success",
  "execution_time_ms": 95,
  "error_message": null,
  "dispatched_at": "2026-05-03 11:00:00.000",
  "completed_at": "2026-05-03 11:00:00.095"
}
```

*Handler #3 - DocumentHandler (Slowest):*
```json
{
  "dispatch_audit_id": "da-003c",
  "event_id": "e789",
  "handler_type": "DocumentHandler",
  "handler_status": "Success",
  "execution_time_ms": 200,
  "error_message": null,
  "dispatched_at": "2026-05-03 11:00:00.000",
  "completed_at": "2026-05-03 11:00:00.200"
}
```

### Sample Query Result
```sql
SELECT handler_type, handler_status, execution_time_ms, completed_at
FROM dispatch_audit
WHERE event_id = 'e789'
ORDER BY completed_at ASC;

RESULT:
┌────────────────────┬────────────────┬──────────────────┬─────────────────────────────┐
│ handler_type       │ handler_status │ execution_time_ms │ completed_at                │
├────────────────────┼────────────────┼──────────────────┼─────────────────────────────┤
│ NotificationHandler │ Success        │ 95               │ 2026-05-03 11:00:00.095     │
│ AssetEventHandler   │ Success        │ 145              │ 2026-05-03 11:00:00.145     │
│ DocumentHandler     │ Success        │ 200              │ 2026-05-03 11:00:00.200     │
└────────────────────┴────────────────┴──────────────────┴─────────────────────────────┘

NOTE: All handlers started at 11:00:00.000
      Slowest handler (DocumentHandler) finished at 11:00:00.200
      Total dispatch time: 200ms
```

---

## Sequence Diagram #4: Notification Multi-Channel

**Scenario:** Notification sent via UI and Email

### Database Entries

**NOTIFICATION_LOG (UI Channel):**
```json
{
  "notification_id": "notif-004a",
  "event_id": "e012",
  "channel": "UI_PUSH",
  "recipient": "user@company.com",
  "message": "Asset ASSET001 updated",
  "status": "SENT",
  "retry_count": 0,
  "error_message": null,
  "sent_at": "2026-05-03 12:00:00.050",
  "created_at": "2026-05-03 12:00:00.000"
}
```

**NOTIFICATION_LOG (Email Channel):**
```json
{
  "notification_id": "notif-004b",
  "event_id": "e012",
  "channel": "EMAIL",
  "recipient": "user@company.com",
  "message": "Asset ASSET001 updated",
  "status": "SENT",
  "retry_count": 0,
  "error_message": null,
  "sent_at": "2026-05-03 12:00:00.250",
  "created_at": "2026-05-03 12:00:00.000"
}
```

### Query: Channel Performance
```sql
SELECT 
  channel,
  COUNT(*) as total_sent,
  AVG(EXTRACT(EPOCH FROM (sent_at - created_at))) as avg_latency_seconds,
  MAX(EXTRACT(EPOCH FROM (sent_at - created_at))) as max_latency_seconds
FROM notification_log
WHERE created_at > NOW() - INTERVAL '1 day'
GROUP BY channel;

RESULT:
┌──────────┬────────────┬──────────────────────┬─────────────────────┐
│ channel  │ total_sent │ avg_latency_seconds  │ max_latency_seconds │
├──────────┼────────────┼──────────────────────┼─────────────────────┤
│ UI_PUSH  │ 4521       │ 0.08                 │ 0.25                │
│ EMAIL    │ 4521       │ 2.15                 │ 5.30                │
└──────────┴────────────┴──────────────────────┴─────────────────────┘
```

---

## Sequence Diagram #5: Document Upload & Votiro Scan

**Scenario:** User uploads Manual.pdf, Votiro scans and returns CLEAN

### Database Entries

#### Step 1: File Uploaded
**ASSET_EVENT_STORE:**
```json
{
  "event_id": "e345",
  "event_type": 5,        // FileUploaded enum
  "event_details": {
    "asset_master_id": "asset-001",
    "attachment_id": "att-001",
    "file_name": "Manual.pdf",
    "file_size": 2048000,
    "content_type": "application/pdf",
    "correlation_id": "votiro-corr-xyz789"
  },
  "created_at": "2026-05-03 13:00:00.000",
  "created_by": "user@company.com"
}
```

#### Step 2: Attachment Recorded
**ATTACHMENT:**
```json
{
  "attachment_id": "att-001",
  "asset_master_id": "asset-001",
  "file_name": "Manual.pdf",
  "file_size": 2048000,
  "mime_type": "application/pdf",
  "correlation_id": "votiro-corr-xyz789",
  "scan_status": "PENDING",
  "created_at": "2026-05-03 13:00:00.000"
}
```

#### Step 3: DocumentHandler Starts Polling
**DOCUMENT_SCAN (Initial):**
```json
{
  "scan_id": "scan-001",
  "attachment_id": "att-001",
  "correlation_id": "votiro-corr-xyz789",
  "scan_status": "SUBMITTED",
  "polling_attempts": 0,
  "last_polled_at": null,
  "created_at": "2026-05-03 13:00:00.050"
}
```

#### Step 4: After Polling Completes (CLEAN)
**DOCUMENT_SCAN (Final):**
```json
{
  "scan_id": "scan-001",
  "attachment_id": "att-001",
  "correlation_id": "votiro-corr-xyz789",
  "scan_status": "CLEAN",
  "scan_result_detail": {
    "threat_detected": false,
    "threat_name": null,
    "risk_level": 0,
    "scan_duration": "2500ms",
    "processed_at": "2026-05-03 13:00:02.500"
  },
  "polling_attempts": 3,
  "last_polled_at": "2026-05-03 13:00:02.500",
  "updated_at": "2026-05-03 13:00:02.550"
}
```

#### Step 5: EventDispatcher Logs DocumentHandler
**DISPATCH_AUDIT (DocumentHandler):**
```json
{
  "dispatch_audit_id": "da-005",
  "event_id": "e345",
  "handler_type": "DocumentHandler",
  "handler_status": "Success",
  "execution_time_ms": 2650,    // Includes polling time!
  "error_message": null,
  "dispatched_at": "2026-05-03 13:00:00.050",
  "completed_at": "2026-05-03 13:00:02.700",
  "created_at": "2026-05-03 13:00:02.700"
}
```

#### Step 6: Notification Sent
**NOTIFICATION_LOG:**
```json
{
  "notification_id": "notif-005",
  "event_id": "e345",
  "channel": "UI_PUSH",
  "recipient": "user@company.com",
  "message": "✅ Manual.pdf scan complete - File is clean",
  "status": "SENT",
  "sent_at": "2026-05-03 13:00:02.750",
  "created_at": "2026-05-03 13:00:02.700"
}
```

---

## Useful Queries

### Query 1: Performance Analysis
```sql
-- Identify slow handlers
SELECT 
  handler_type,
  COUNT(*) as invocations,
  AVG(execution_time_ms) as avg_time_ms,
  MAX(execution_time_ms) as max_time_ms,
  MIN(execution_time_ms) as min_time_ms
FROM dispatch_audit
WHERE dispatched_at > NOW() - INTERVAL '1 day'
GROUP BY handler_type
ORDER BY avg_time_ms DESC;
```

### Query 2: Failure Analysis
```sql
-- Check for handler failures
SELECT 
  handler_type,
  handler_status,
  COUNT(*) as count,
  error_message
FROM dispatch_audit
WHERE dispatched_at > NOW() - INTERVAL '1 day'
  AND handler_status != 'Success'
GROUP BY handler_type, handler_status, error_message
ORDER BY count DESC;
```

### Query 3: Complete Event Lifecycle
```sql
-- Track one event through entire system
SELECT 
  'ASSET_EVENT' as stage,
  event_id,
  '' as handler,
  created_at as timestamp
FROM asset_event_store
WHERE event_id = 'e123'

UNION ALL

SELECT 
  'ASSET_AUDIT' as stage,
  event_id,
  handler_name as handler,
  created_at as timestamp
FROM asset_audit
WHERE event_id = 'e123'

UNION ALL

SELECT 
  'DISPATCH' as stage,
  event_id,
  handler_type as handler,
  completed_at as timestamp
FROM dispatch_audit
WHERE event_id = 'e123'

UNION ALL

SELECT 
  'NOTIFICATION' as stage,
  event_id,
  channel as handler,
  sent_at as timestamp
FROM notification_log
WHERE event_id = 'e123'

ORDER BY timestamp ASC;
```

### Query 4: Notification Delivery Rate
```sql
-- Track notification success rate
SELECT 
  channel,
  status,
  COUNT(*) as count,
  ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY channel), 2) as percentage
FROM notification_log
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY channel, status
ORDER BY channel, status;
```

---

## Key Insights from LogDispatchAudit

| Metric | What It Tells You |
|--------|---|
| `execution_time_ms` | How fast is this handler? Is it a bottleneck? |
| `handler_status` | Did handler succeed or fail? |
| `error_message` | Why did handler fail? (For debugging) |
| `dispatched_at` → `completed_at` | What's the latency for this handler? |
| Count of handlers for one event | How many handlers are processing this event? |

### Common Use Cases

**Monitor Performance:**
```sql
-- Which handler is slowest?
SELECT handler_type, AVG(execution_time_ms) 
FROM dispatch_audit 
GROUP BY handler_type 
ORDER BY 2 DESC LIMIT 1;
```

**Track Failures:**
```sql
-- Did DocumentHandler fail in last hour?
SELECT * FROM dispatch_audit
WHERE handler_type = 'DocumentHandler'
  AND handler_status = 'Failed'
  AND dispatched_at > NOW() - INTERVAL '1 hour';
```

**Audit Trail:**
```sql
-- Show complete audit trail for one asset
SELECT d.handler_type, d.handler_status, d.execution_time_ms, d.completed_at
FROM dispatch_audit d
WHERE d.event_id IN (
  SELECT event_id FROM asset_event_store 
  WHERE event_payload->>'asset_master_id' = 'asset-001'
)
ORDER BY d.completed_at DESC;
```

---

**Last Updated:** May 3, 2026 | **Version:** 1.0
