<!-- filepath: DOCUMENT_SCAN_ARCHITECTURE.md -->
# Document Scan Architecture — Event-Driven with Polling

> **Purpose:** Complete explanation of how DocumentHandler integrates with EventStore and EventDispatcher for Votiro scanning

---

## Architecture Overview

The document scanning process is now fully event-driven:

```
User Upload
    ↓
DocumentHandler creates ATTACHMENT + DOCUMENT_SCAN records
    ↓
DocumentHandler emits "DocumentScanStarted" event → EventStore
    ↓
EventDispatcher receives event
    ↓
EventDispatcher resolves DocumentHandler as eligible handler
    ↓
EventDispatcher executes DocumentHandler (polling task)
    ↓
DocumentHandler polls Votiro API (async, background)
    ↓
Scan completes → DocumentHandler emits "DocumentScanCompleted" event
    ↓
EventDispatcher routes to NotificationHandler
    ↓
NotificationHandler sends result to UI (SignalR)
```

---

## Detailed Flow

### Phase 1: Upload & Initial Request (Synchronous)

**1.1 User uploads file**
```
POST /api/v1/assets/{assetId}/attachments
Content-Type: multipart/form-data
```

**1.2 DocumentHandler receives request**
- Validates file (size, type, virus scan enabled)
- Submits file directly to Votiro CDR API (no quarantine)
- Receives `correlationId` from Votiro

**1.3 Create database records**
```sql
INSERT INTO attachment (asset_master_id, file_name, correlation_id, scan_status)
VALUES ('asset-001', 'Manual.pdf', 'votiro-xyz789', 'PENDING');

INSERT INTO document_scan (attachment_id, correlation_id, scan_status)
VALUES ('att-001', 'votiro-xyz789', 'SUBMITTED');
```

**1.4 Emit DocumentScanStarted event**
```csharp
// DocumentHandler immediately emits event
await _eventStore.EmitEventAsync(new DocumentScanStarted
{
    AttachmentId = attachment.Id,
    CorrelationId = correlationId,
    FileName = file.FileName,
    FileSize = file.Length
});
```

**1.5 Respond to user**
```json
HTTP 202 Accepted
{
  "attachmentId": "att-001",
  "correlationId": "votiro-xyz789",
  "status": "submitted"
}
```

---

### Phase 2: Event Dispatch (Immediate)

**2.1 EventStore publishes DocumentScanStarted**
```
EventStore → EventDispatcher.PublishEvent(DocumentScanStarted)
```

**2.2 EventDispatcher resolves handlers**
```csharp
var handlers = _handlerRegistry.ResolveHandlers(DocumentScanStarted);
// Result: [DocumentHandler]  ← DocumentHandler can handle this event
```

**2.3 EventDispatcher executes DocumentHandler**
```csharp
await _handler.ExecuteHandlerAsync(DocumentScanStarted);
// This starts the async polling loop
```

---

### Phase 3: Async Polling (Background Task)

**3.1 DocumentHandler executes polling loop**
```csharp
public class DocumentHandler : IEventHandler
{
    public async Task ExecuteHandlerAsync(IEvent @event)
    {
        var scanStarted = (DocumentScanStarted)@event;
        var correlationId = scanStarted.CorrelationId;
        
        // Async polling (no await here - fire and forget)
        _ = PollVotiroAsync(correlationId);
        
        // Log audit immediately
        await LogAuditAsync("Polling started", "pending");
    }
    
    private async Task PollVotiroAsync(string correlationId)
    {
        var maxAttempts = 30;  // ~5 minutes with 10-second intervals
        var attempt = 0;
        
        while (attempt < maxAttempts)
        {
            attempt++;
            
            try
            {
                // Check DB for current status
                var scan = await _db.GetDocumentScanAsync(correlationId);
                
                if (scan.Status != "SUBMITTED")
                    break;  // Already completed by webhook
                
                // Poll Votiro for result
                var result = await _votiro.GetScanResultAsync(correlationId);
                
                if (result.IsComplete)
                {
                    // Update database
                    await UpdateScanResultAsync(correlationId, result);
                    
                    // Emit completion event
                    await _eventStore.EmitEventAsync(new DocumentScanCompleted
                    {
                        CorrelationId = correlationId,
                        ScanStatus = result.Status,
                        ThreatDetected = result.ThreatDetected
                    });
                    
                    break;  // Exit polling loop
                }
                
                // Wait before next poll
                await Task.Delay(10000);  // 10 seconds
            }
            catch (Exception ex)
            {
                _logger.LogError($"Polling failed: {ex.Message}");
                // Continue polling, don't fail
            }
        }
        
        // If max attempts reached without completion
        if (attempt >= maxAttempts)
        {
            await UpdateScanStatusAsync(correlationId, "TIMEOUT");
            await _eventStore.EmitEventAsync(new DocumentScanFailed
            {
                CorrelationId = correlationId,
                Reason = "Polling timeout after 30 attempts"
            });
        }
    }
}
```

**3.2 Polling states**

| Attempt | Time | Votiro Status | DB Status | Action |
|---------|------|---------------|-----------|--------|
| 1 | 10s | PROCESSING | SUBMITTED | Continue |
| 2 | 20s | PROCESSING | SUBMITTED | Continue |
| 3 | 30s | COMPLETE | SUBMITTED | Get result |
| - | - | - | CLEAN | Emit DocumentScanCompleted |

---

### Phase 4: Scan Completion (Event Dispatch)

**4.1 DocumentHandler emits DocumentScanCompleted**
```csharp
await _eventStore.EmitEventAsync(new DocumentScanCompleted
{
    CorrelationId = correlationId,
    ScanStatus = "CLEAN",        // or INFECTED, FAILED
    ThreatDetected = false,
    PollingAttempts = 3
});
```

**4.2 EventStore publishes DocumentScanCompleted**
```
EventStore → EventDispatcher.PublishEvent(DocumentScanCompleted)
```

**4.3 EventDispatcher resolves handlers**
```csharp
var handlers = _handlerRegistry.ResolveHandlers(DocumentScanCompleted);
// Result: [NotificationHandler]  ← NotificationHandler handles completion
```

**4.4 EventDispatcher executes NotificationHandler**
```csharp
await notificationHandler.ExecuteHandlerAsync(DocumentScanCompleted);
// Sends result to user via SignalR
```

---

## Database Schema

### ATTACHMENT table
```sql
CREATE TABLE attachment (
    id UUID PRIMARY KEY,
    asset_master_id UUID NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_size BIGINT,
    correlation_id VARCHAR(100) UNIQUE,  -- Votiro's ID
    scan_status VARCHAR(20),              -- PENDING, CLEAN, INFECTED, TIMEOUT
    threat_name VARCHAR(255),             -- If threat detected
    risk_level INT,                       -- 0-100
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (asset_master_id) REFERENCES asset_master(id)
);
```

### DOCUMENT_SCAN table
```sql
CREATE TABLE document_scan (
    id UUID PRIMARY KEY,
    attachment_id UUID NOT NULL,
    correlation_id VARCHAR(100),
    scan_status VARCHAR(20),              -- SUBMITTED, PROCESSING, CLEAN, INFECTED, FAILED
    polling_attempts INT DEFAULT 0,
    last_polled_at TIMESTAMP,
    threat_detail JSONB,                  -- Detailed threat info from Votiro
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (attachment_id) REFERENCES attachment(id),
    INDEX (correlation_id)
);
```

---

## ASSET_AUDIT Entries

### DocumentScanStarted
```json
{
  "handler": "DocumentHandler",
  "change_type": "SCANNED",
  "message": "File submitted to Votiro: Manual.pdf",
  "detail": {
    "file_name": "Manual.pdf",
    "correlation_id": "votiro-xyz789",
    "file_size": 2048000,
    "execution_time_ms": 150,
    "result": "success"
  }
}
```

### DocumentScanCompleted
```json
{
  "handler": "DocumentHandler",
  "change_type": "SCANNED",
  "message": "Scan complete: Manual.pdf is CLEAN",
  "detail": {
    "correlation_id": "votiro-xyz789",
    "scan_status": "CLEAN",
    "threat_detected": false,
    "polling_attempts": 3,
    "total_polling_time_ms": 30000,
    "execution_time_ms": 2650,
    "result": "success"
  }
}
```

### NotificationHandler Dispatch
```json
{
  "handler": "NotificationHandler",
  "change_type": "NOTIFIED",
  "message": "Scan result notification sent: CLEAN",
  "detail": {
    "channel": "UI_PUSH",
    "recipient": "user@company.com",
    "execution_time_ms": 25,
    "result": "success"
  }
}
```

---

## Advantages of This Architecture

### ✅ Event-Driven
- Clear separation of concerns
- Decoupled components (DocumentHandler, NotificationHandler)
- EventStore maintains immutable audit trail

### ✅ Async Polling
- Non-blocking file uploads
- Immediate 202 response to user
- Background polling doesn't block other requests
- Max 30 attempts (~5 minutes) prevents infinite loops

### ✅ No Quarantine Storage
- Direct upload to Votiro CDR
- Reduces storage overhead
- Faster scan initiation
- Cleaner security model

### ✅ Correlation ID Tracking
- Votiro provides unique correlation ID per file
- Used to poll for results
- Enables reliable async processing
- Easy to trace requests end-to-end

### ✅ Single Audit Log
- All activities logged to ASSET_AUDIT
- No separate dispatch_audit table
- Single source of truth
- Simple queries without joins

---

## Error Scenarios

### Scenario 1: File Too Large
```
Upload → Validation fails → 413 Payload Too Large
No event emitted, no polling
User sees error immediately
```

### Scenario 2: Votiro Unavailable
```
Submit to Votiro → Exception → Log error → EventStore records failure
DocumentHandler.ExecuteHandlerAsync() handles gracefully
Admin can retry manually
```

### Scenario 3: Polling Timeout (30 attempts, ~5 minutes)
```
DocumentHandler polling completes without result
→ EmitEvent(DocumentScanFailed, reason="Polling timeout")
→ EventDispatcher routes to NotificationHandler
→ User notified: "Scan timeout, contact support"
```

### Scenario 4: Threat Detected
```
Polling complete → Votiro returns INFECTED
→ Update ATTACHMENT status=INFECTED, threat_name="Trojan.XYZ"
→ EmitEvent(DocumentScanCompleted, threat=true)
→ NotificationHandler sends: "⚠️ Threat detected: Trojan.XYZ"
```

---

## Monitoring & Debugging

### Query: Find all pending scans
```sql
SELECT a.file_name, ds.correlation_id, ds.polling_attempts, ds.last_polled_at
FROM document_scan ds
JOIN attachment a ON ds.attachment_id = a.id
WHERE ds.scan_status = 'SUBMITTED';
```

### Query: Find timeout scans
```sql
SELECT a.file_name, ds.correlation_id, EXTRACT(EPOCH FROM (NOW() - ds.created_at)) as age_seconds
FROM document_scan ds
JOIN attachment a ON ds.attachment_id = a.id
WHERE ds.scan_status = 'SUBMITTED'
  AND (NOW() - ds.created_at) > INTERVAL '5 minutes';
```

### Query: Audit trail for one attachment
```sql
SELECT aa.handler, aa.change_type, aa.message, aa.detail->>'polling_attempts' as attempts, aa.created_at
FROM asset_audit aa
WHERE aa.asset_master_id = 'asset-001'
  AND aa.change_type = 'SCANNED'
ORDER BY aa.created_at ASC;
```

---

## Configuration

```csharp
// Polling configuration
public class DocumentScanConfig
{
    public int PollingIntervalSeconds { get; set; } = 10;
    public int MaxPollingAttempts { get; set; } = 30;     // ~5 minutes
    public int PollingTimeoutSeconds { get; set; } = 300; // 5 minutes
    public bool EnableVotiroCDR { get; set; } = true;
}
```

---

**Last Updated:** May 3, 2026 | **Version:** 1.0
