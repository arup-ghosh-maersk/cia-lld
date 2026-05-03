# Asset Master - Sequence Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Asset Create with Event Dispatcher

```mermaid
sequenceDiagram
    actor User
    participant UI as Asset UI
    participant API as Asset API
    participant AB as Asset Business Service
    participant DB as Asset DB
    participant ES as EventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant Audit as AuditStore
    participant Push as Notification

    User->>UI: Fill form and Submit
    UI->>API: POST /api/v1/assets
    API->>AB: ValidateAndCreate(assetData)
    AB->>DB: INSERT Asset
    DB-->>AB: AssetId ASSET001

    AB->>ES: EmitEvent Type AssetCreated
    ES-->>API: Event Stored EventId
    API-->>UI: 201 Created

    Note over ES,Push: Async Event Dispatch Processing
    ES->>ED: PublishEvent(AssetCreated)
    ED->>ED: ResolveHandlers(event)
    ED->>AEH: ExecuteHandlerAsync(event)
    AEH->>AEH: HandleAssetCreate(event)
    AEH->>AEH: GenerateAuditMessage()
    AEH->>Audit: LogAudit(assetCreated, handler=AssetEventHandler)
    Audit-->>AEH: AuditRecord Created
    AEH->>Push: NotifyOnCompletion()
    Push-->>UI: Push Asset ASSET001 Created
    AEH-->>ED: Handler Complete
```

---

## 2. Asset Update with Event Dispatch

```mermaid
sequenceDiagram
    actor User
    participant UI as Asset UI
    participant API as Asset API
    participant AB as Asset Business Service
    participant DB as Asset DB
    participant ES as EventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant NH as NotificationHandler
    participant Audit as AuditStore
    participant Push as Notification

    User->>UI: Modify Asset and Submit
    UI->>API: PUT /api/v1/assets/ASSET001
    API->>AB: ValidateAndUpdate(assetData)
    AB->>DB: UPDATE Asset Record
    DB-->>AB: Updated

    AB->>ES: EmitEvent Type AssetUpdated
    ES-->>API: Event Stored EventId
    API-->>UI: 200 OK

    Note over ES,Push: Async Event Dispatch Processing
    ES->>ED: PublishEvent(AssetUpdated)
    ED->>ED: ResolveHandlers(event)
    par AssetEventHandler
        ED->>AEH: ExecuteHandlerAsync(event)
        AEH->>AEH: ExtractModifiedFields(before, after)
        AEH->>Audit: LogAudit(fieldChanges, handler=AssetEventHandler)
        Audit-->>AEH: AuditRecord Created
        AEH-->>ED: Handler Complete
    and NotificationHandler
        ED->>NH: ExecuteHandlerAsync(event)
        NH->>NH: ResolveChannels(recipients)
        NH->>Push: DispatchToChannels(notification)
        Push-->>NH: Dispatched
        NH->>Audit: LogNotification(handler=NotificationHandler)
        NH-->>ED: Handler Complete
    end

    Push-->>UI: Push Asset ASSET001 Updated
```

---

## 3. EventDispatcher Coordination

```mermaid
sequenceDiagram
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
        AEH->>AA: LogAudit(eventDetails, handler=AssetEventHandler)
        AA-->>AEH: AuditRecord
        AEH-->>ED: Success
    and
        ED->>NH: ExecuteHandlerAsync(event)
        NH->>AA: LogNotification(handler=NotificationHandler)
        AA-->>NH: Logged
        NH-->>ED: Success
    and
        ED->>DH: ExecuteHandlerAsync(event)
        DH->>AA: LogScan(handler=DocumentHandler)
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
    participant ED as EventDispatcher
    participant NH as NotificationHandler
    participant SIGNALR as UINotificationChannel
    participant EMAIL as EmailNotificationChannel
    participant NL as NotificationLog
    participant UI as Asset Master UI
    participant EMAILSVC as Email Service

    ED->>NH: HandleAsync(event)
    NH->>NH: ResolveChannels(recipients)
    Note over NH: Determine channels UI and Email

    par UI Push
        NH->>SIGNALR: SendAsync(message, recipient)
        SIGNALR->>UI: SignalR Push Asset ASSET001 Updated
        UI-->>SIGNALR: Acknowledged
        SIGNALR->>NL: LogNotification(channel=UI_PUSH)
        NL-->>SIGNALR: Logged
        SIGNALR-->>NH: true
    and Email
        NH->>EMAIL: SendAsync(message, recipient)
        EMAIL->>EMAILSVC: SendEmailAsync(template, recipient)
        EMAILSVC-->>EMAIL: Delivered
        EMAIL->>NL: LogNotification(channel=EMAIL)
        NL-->>EMAIL: Logged
        EMAIL-->>NH: true
    end

    NH-->>ED: All channels dispatched
```

---

## 5. Document Upload and Direct Votiro Scan

### 5a. Upload and Submit

```mermaid
sequenceDiagram
    actor User
    participant UI as Asset Master UI
    participant API as Asset API
    participant DH as DocumentHandler
    participant VOTIRO as Votiro CDR API
    participant DB as Attachment DB
    participant ES as EventStore

    User->>UI: Upload file Manual.pdf
    UI->>API: POST /attachments
    API->>DH: UploadAndScanAsync

    DH->>VOTIRO: SubmitFileAsync
    VOTIRO-->>DH: correlationId

    DH->>DB: CreateAttachment status Pending
    DH->>DB: CreateDocumentScan status Submitted

    DH->>ES: Emit DocumentScanStarted
    API-->>UI: 202 Accepted
    UI-->>User: File submitted for scanning
```

### 5b. Background Polling

```mermaid
sequenceDiagram
    participant ES as EventStore
    participant ED as EventDispatcher
    participant DH as DocumentHandler
    participant DB as Attachment DB
    participant VOTIRO as Votiro CDR API

    ES->>ED: PublishEvent(DocumentScanStarted, correlationId)
    ED->>ED: ResolveHandlers(DocumentScanStarted)
    ED->>DH: ExecuteHandlerAsync(DocumentScanStarted)

    Note over DH,DB: EventDispatcher triggers DocumentHandler polling

    loop Poll every N seconds max 30 attempts
        DH->>DB: GetDocumentScan(correlationId)
        DB-->>DH: status

        Note over DH: If status is COMPLETED
        DH->>VOTIRO: GetScanResult(correlationId)
        VOTIRO-->>DH: ScanResult

        Note over DH,DB: Determine final status Clean or ThreatDetected
        DH->>DB: UpdateAttachment(finalStatus)
        DH->>DB: UpdateDocumentScan(finalStatus)

        DH->>ES: Emit DocumentScanCompleted
        break Stop polling
            DH-->>ED: Handler Complete
        end
    end
```

### 5c. Scan Completion Notification

```mermaid
sequenceDiagram
    actor User
    participant ES as EventStore
    participant ED as EventDispatcher
    participant NH as NotificationHandler
    participant Audit as AuditStore
    participant UI as Asset Master UI

    ES->>ED: Publish DocumentScanCompleted
    ED->>NH: ExecuteHandlerAsync

    NH->>Audit: Log notification event
    Audit-->>NH: Logged

    NH->>UI: SignalR scan completed
    UI-->>User: Display scan result
```
