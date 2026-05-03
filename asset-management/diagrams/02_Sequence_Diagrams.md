# Asset Master — Sequence Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Asset Create with Event Dispatcher

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Asset UI
    participant API as Asset API
    participant AB as Asset Business Service
    participant DB as Database
    participant ES as EventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant Audit as AuditStore
    participant Push as Notification

    User->>UI: Fill form & Submit
    UI->>API: POST /api/v1/assets
    API->>AB: ValidateAndCreate(assetData)
    AB->>DB: INSERT Asset
    DB-->>AB: AssetId: ASSET001    AB->>ES: EmitEvent {Type: AssetCreated, Details: {...}}
    ES-->>API: Event Stored, EventId
    API-->>UI: 201 Created

    Note over ES,Push: Async Event Dispatch Processing
    ES->>ED: PublishEvent(AssetCreated)
    ED->>ED: ResolveHandlers(event)
    ED->>AEH: ExecuteHandlerAsync(event)
    AEH->>AEH: HandleAssetCreate(event)
    AEH->>AEH: GenerateAuditMessage()
    AEH->>Audit: LogAudit(assetCreated, handler="AssetEventHandler", detail={...})
    Audit-->>AEH: AuditRecord Created
    AEH->>Push: NotifyOnCompletion()
    Push-->>UI: Push — "Asset ASSET001 Created"
    AEH-->>ED: Handler Complete
```

---

## 2. Asset Update with Event Dispatch

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Asset UI
    participant API as Asset API
    participant AB as Asset Business Service
    participant DB as Database
    participant ES as EventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant NH as NotificationHandler
    participant Audit as AuditStore
    participant Push as Notification

    User->>UI: Modify Asset & Submit
    UI->>API: PUT /api/v1/assets/ASSET001
    API->>AB: ValidateAndUpdate(assetData)
    AB->>DB: UPDATE Asset Record
    DB-->>AB: Updated

    AB->>ES: EmitEvent {Type: AssetUpdated, Details: {...}}
    ES-->>API: Event Stored, EventId
    API-->>UI: 200 OK

    Note over ES,Push: Async Event Dispatch Processing
    ES->>ED: PublishEvent(AssetUpdated)
    ED->>ED: ResolveHandlers(event)
      par AssetEventHandler
        ED->>AEH: ExecuteHandlerAsync(event)
        AEH->>AEH: ExtractModifiedFields(before, after)
        AEH->>Audit: LogAudit(fieldChanges, handler="AssetEventHandler")
        Audit-->>AEH: AuditRecord Created
        AEH-->>ED: Handler Complete
    and NotificationHandler
        ED->>NH: ExecuteHandlerAsync(event)
        NH->>NH: ResolveChannels(recipients)
        NH->>Push: DispatchToChannels(notification)
        Push-->>NH: Dispatched
        NH->>Audit: LogNotification(handler="NotificationHandler", channels=[...])
        NH-->>ED: Handler Complete
    end
    
    Push-->>UI: Push — "Asset ASSET001 Updated"
```

---

## 3. EventDispatcher Coordination

```mermaid
sequenceDiagram
    autonumber
    participant ES as AssetEventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant NH as NotificationHandler
    participant DH as DocumentHandler
    participant AA as AuditStore
    
    ES->>ED: PublishEvent(event)
    ED->>ED: ResolveHandlers(event)
    Note over ED: Check CanHandle() for each handler

    par Parallel Execution
        ED->>AEH: ExecuteHandlerAsync(event)
        AEH->>AA: LogAudit(eventDetails, handler="AssetEventHandler")
        AA-->>AEH: AuditRecord
        AEH-->>ED: Success
    and
        ED->>NH: ExecuteHandlerAsync(event)
        NH->>AA: LogNotification(handler="NotificationHandler")
        AA-->>NH: Logged
        NH-->>ED: Success
    and
        ED->>DH: ExecuteHandlerAsync(event)
        DH->>AA: LogScan(handler="DocumentHandler")
        AA-->>DH: Logged
        DH-->>ED: Success
    end

    ED-->>ES: Dispatch Complete
    Note over AA: All audit activities logged to ASSET_AUDIT
```

---

## 4. Notification Handler Multi-Channel Dispatch

```mermaid
sequenceDiagram
    autonumber
    participant ED as EventDispatcher
    participant NH as NotificationHandler
    participant SIGNALR as UINotificationChannel
    participant EMAIL as EmailNotificationChannel
    participant NL as NotificationLog
    participant UI as Asset Master UI
    participant EMAILSVC as Email Service

    ED->>NH: HandleAsync(event)
    NH->>NH: ResolveChannels(recipients)
    Note over NH: Determine channels (UI, Email)

    par UI Push
        NH->>SIGNALR: SendAsync(message, recipient)
        SIGNALR->>UI: SignalR Push — "Asset ASSET001 Updated"
        UI-->>SIGNALR: Acknowledged
        SIGNALR->>NL: LogNotification(handler="NotificationHandler", channel="UI_PUSH")
        NL-->>SIGNALR: Logged
        SIGNALR-->>NH: true
    and Email
        NH->>EMAIL: SendAsync(message, recipient)
        EMAIL->>EMAILSVC: SendEmailAsync(template, recipient)
        EMAILSVC-->>EMAIL: Delivered
        EMAIL->>NL: LogNotification(handler="NotificationHandler", channel="EMAIL")
        NL-->>EMAIL: Logged
        EMAIL-->>NH: true
    end

    NH-->>ED: All channels dispatched
```

---

## 5. Document Upload & Direct Votiro Scan

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Asset Master UI
    participant API as Asset API
    participant DH as DocumentHandler
    participant ES as EventStore
    participant VOTIRO as Votiro CDR API
    participant DB as Database
    participant ED as EventDispatcher
    participant NH as NotificationHandler
    participant Audit as AuditStore

    User->>UI: Upload file (e.g. Manual.pdf)
    UI->>API: POST /api/v1/assets/{assetId}/attachments
    API->>DH: UploadAndScanAsync(file, metadata)

    DH->>VOTIRO: SubmitFileAsync(file, metadata)
    VOTIRO-->>DH: VotiroResponse {correlationId}
    
    DH->>DB: CreateAttachment {fileName, correlationId, status=Pending}
    DB-->>DH: Attachment Created
    DH->>DB: CreateDocumentScan {correlationId, status=Submitted}
    DB-->>DH: DocumentScan Created

    Note over DH,ES: Emit Document Scan Event
    DH->>ES: EmitEvent(DocumentScanStarted, correlationId)
    ES-->>API: Event Stored, EventId
    
    DH-->>API: Submitted for Scan
    API-->>UI: 202 Accepted {attachmentId, correlationId}
    UI-->>User: ⏳ "File submitted for scanning..."

    Note over ES,ED: EventDispatcher triggers DocumentHandler polling
    ES->>ED: PublishEvent(DocumentScanStarted, correlationId)
    ED->>ED: ResolveHandlers(DocumentScanStarted)
    ED->>DH: ExecuteHandlerAsync(DocumentScanStarted)

    Note over DH,DB: Async Polling & Processing
    loop Poll Every N Seconds (Max 30)
        DH->>DB: GetDocumentScan(correlationId)
        DB-->>DH: DocumentScan {status, pollingAttempts}
        
        opt Scan Complete
            DH->>VOTIRO: GetScanResultAsync(correlationId)
            VOTIRO-->>DH: ScanResult {status, threat}
            
            alt Threat Detected
                DH->>DB: UpdateAttachment {status=ThreatDetected}
                DH->>DB: UpdateDocumentScan {status=ThreatDetected}
            else Clean
                DH->>DB: UpdateAttachment {status=Clean}
                DH->>DB: UpdateDocumentScan {status=Clean}
            else Failed
                DH->>DB: UpdateDocumentScan {status=Failed}
            end
            break Exit Loop
        end
    end

    Note over DH,ES: Emit Scan Completion Event
    DH->>ES: EmitEvent(DocumentScanCompleted, result)
    ES-->>DH: Event Stored
    
    DH->>Audit: LogScan(handler="DocumentHandler", scan_result)
    Audit-->>DH: Logged    Note over ES,NH: EventDispatcher routes to NotificationHandler
    ES->>ED: PublishEvent(DocumentScanCompleted)
    ED->>ED: ResolveHandlers(DocumentScanCompleted)
    ED->>NH: ExecuteHandlerAsync(DocumentScanCompleted)
    NH->>Audit: LogNotification(handler="NotificationHandler", result)
    Audit-->>NH: Logged
    NH->>UI: SignalR: ✅/❌ "Manual.pdf scan complete"
    UI-->>User: Display result
```
