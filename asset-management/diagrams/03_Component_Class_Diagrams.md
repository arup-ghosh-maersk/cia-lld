# Asset Master — Sequence Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Asset Creation Flow

```mermaid
sequenceDiagram
    actor User
    participant UI as Asset Master UI
    participant API as Asset API
    participant AB as Asset Business Service
    participant AES as AssetEventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant AUDITDB as Audit Log DB
    participant ASSETDB as Asset Master DB

    User->>UI: Create Asset
    UI->>API: POST /assets
    API->>AB: ValidateAndCreate(assetData)
    AB->>ASSETDB: Save Asset
    ASSETDB-->>AB: Asset Created
    AB->>AES: EmitEvent(AssetCreated)
    AES-->>API: Event Stored
    API-->>UI: Asset Created Response
    UI-->>User: Show Confirmation

    AES->>ED: PublishEvent(AssetCreated)
    ED->>AEH: CanHandle(event)?
    AEH-->>ED: true
    ED->>AEH: HandleAsync(event)
    AEH->>AEH: HandleAssetCreate(event)
    AEH->>AEH: GenerateAuditMessage()
    AEH->>AUDITDB: LogAudit(assetCreated, details)
    AUDITDB-->>AEH: Audit Logged
    AEH->>ED: Complete
```

---

## 2. Asset Update with Event Dispatch

```mermaid
sequenceDiagram
    actor User
    participant UI as Asset Master UI
    participant API as Asset API
    participant AB as Asset Business Service
    participant AES as AssetEventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant NH as NotificationHandler
    participant SIGNALR as SignalR Hub
    participant ASSETDB as Asset Master DB
    participant AUDITDB as Audit Log DB

    User->>UI: Update Asset
    UI->>API: PUT /assets/{id}
    API->>AB: ValidateAndUpdate(assetData)
    AB->>ASSETDB: Save Updated Asset
    ASSETDB-->>AB: Asset Updated
    AB->>AES: EmitEvent(AssetUpdated)
    AES-->>API: Event Stored
    API-->>UI: Asset Updated Response

    AES->>ED: PublishEvent(AssetUpdated)
    ED->>AEH: DispatchAsync(event)
    AEH->>AEH: ExtractModifiedFields(before, after)
    AEH->>AUDITDB: LogAudit(fieldChanges)
    AUDITDB-->>AEH: Audit Logged
    AEH->>ED: NotifyOnCompletion()
    
    ED->>NH: DispatchAsync(event)
    NH->>NH: ResolveChannels(recipients)
    NH->>SIGNALR: SendToUIAsync(notification)
    SIGNALR-->>NH: Sent
    NH->>AUDITDB: LogNotification()
    AUDITDB-->>NH: Logged
    
    SIGNALR->>UI: Push Notification
    UI-->>User: Display Update
```

---

## 3. Document Upload & Scan Flow

```mermaid
sequenceDiagram
    actor User
    participant UI as Asset Master UI
    participant API as Asset API
    participant DH as DocumentHandler
    participant VOTIRO as Votiro CDR API
    participant ATTACHDB as Attachment DB
    participant SCANDB as DocumentScan DB
    participant WEBHOOK as Event Bus
    participant ED as EventDispatcher
    participant NH as NotificationHandler

    User->>UI: Upload Document
    UI->>API: POST /assets/{assetId}/attachments
    API->>DH: UploadAndScanAsync(file, metadata)
    
    DH->>VOTIRO: SubmitFileAsync(file, metadata)
    VOTIRO-->>DH: VotiroResponse {correlationId}
    
    DH->>ATTACHDB: CreateAttachment(fileName, status=Pending)
    ATTACHDB-->>DH: Attachment Created
    
    DH->>SCANDB: CreateDocumentScan(correlationId, status=Submitted)
    SCANDB-->>DH: DocumentScan Created
    
    DH-->>API: Submitted for Scan
    API-->>UI: Return correlationId
    UI-->>User: Scanning...
```

---

## 4. Document Scan Result Polling & Completion

```mermaid
sequenceDiagram
    participant DH as DocumentHandler
    participant SCANDB as DocumentScan DB
    participant VOTIRO as Votiro CDR API
    participant ATTACHDB as Attachment DB
    participant AUDITDB as Audit Log DB
    participant WEBHOOK as Event Bus
    participant ED as EventDispatcher
    participant NH as NotificationHandler
    participant SIGNALR as SignalR Hub
    participant UI as Asset Master UI

    loop Poll Every N Seconds
        DH->>SCANDB: GetDocumentScan(correlationId)
        SCANDB-->>DH: DocumentScan {status, pollingAttempts}
        
        alt Status Not Complete
            DH->>VOTIRO: GetScanResultAsync(correlationId)
            VOTIRO-->>DH: ScanResult
            
            alt Threat Detected
                DH->>ATTACHDB: UpdateAttachment(status=ThreatDetected)
                DH->>SCANDB: MarkThreatDetected(threatName)
                DH->>AUDITDB: LogScan(threatDetected)
            else Clean
                DH->>ATTACHDB: UpdateAttachment(status=Clean)
                DH->>SCANDB: MarkClean()
                DH->>AUDITDB: LogScan(clean)
            else Failed
                DH->>SCANDB: MarkFailed(reason)
                DH->>AUDITDB: LogScan(failed)
            end
            
            DH->>WEBHOOK: PublishEvent(ScanCompleted)
        else Max Retries Exceeded
            DH->>SCANDB: MarkFailed(maxRetriesExceeded)
            break Stop Polling
        end
    end

    WEBHOOK->>ED: DispatchEvent(ScanCompleted)
    ED->>NH: DispatchAsync(event)
    NH->>NH: ResolveChannels(assetOwner)
    NH->>SIGNALR: SendToUIAsync(scanResult)
    SIGNALR-->>UI: Push Result
    NH->>AUDITDB: LogNotification()
    UI-->>User: Display Scan Result
```

---

## 5. EventDispatcher Coordination Flow

```mermaid
sequenceDiagram
    participant AES as AssetEventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant NH as NotificationHandler
    participant DH as DocumentHandler
    participant AUDITDB as Audit Log DB

    AES->>ED: PublishEvent(event)
    ED->>ED: ResolveHandlers(event)
    Note over ED: Check CanHandle() for each handler

    par Parallel Handler Execution
        ED->>AEH: ExecuteHandlerAsync(event)
        AEH->>AEH: HandleAsync(event)
        AEH->>AUDITDB: LogAudit()
        AUDITDB-->>AEH: Done
        AEH-->>ED: Success
        ED->>AUDITDB: LogDispatchAudit(handler=AEH, status=Success)
    and
        ED->>NH: ExecuteHandlerAsync(event)
        NH->>NH: HandleAsync(event)
        NH->>AUDITDB: LogNotification()
        AUDITDB-->>NH: Done
        NH-->>ED: Success
        ED->>AUDITDB: LogDispatchAudit(handler=NH, status=Success)
    and
        ED->>DH: ExecuteHandlerAsync(event)
        DH->>DH: HandleAsync(event)
        DH->>AUDITDB: LogScan()
        AUDITDB-->>DH: Done
        DH-->>ED: Success
        ED->>AUDITDB: LogDispatchAudit(handler=DH, status=Success)
    end

    ED-->>AES: Dispatch Complete
    Note over AUDITDB: All dispatch activities logged
```

---

## 6. Notification Dispatch to Multiple Channels

```mermaid
sequenceDiagram
    participant ED as EventDispatcher
    participant NH as NotificationHandler
    participant CHANNELS as INotificationChannel
    participant SIGNALR as UINotificationChannel
    participant EMAIL as EmailNotificationChannel
    participant NOTIFDB as NotificationLog DB
    participant SIGNALRHUB as SignalR Hub
    participant EMAILSVC as Email Service

    ED->>NH: HandleAsync(event)
    NH->>NH: ResolveChannels(recipients)
    Note over NH: Determine notification channels<br/>(UI, Email)

    par Send to All Channels
        NH->>SIGNALR: SendAsync(message, recipient)
        SIGNALR->>SIGNALRHUB: SendAsync(connectionId, message)
        SIGNALRHUB-->>SIGNALR: Sent
        SIGNALR->>NOTIFDB: LogSuccess(Channel=UI)
        NOTIFDB-->>SIGNALR: Logged
        SIGNALR-->>NH: true
    and
        NH->>EMAIL: SendAsync(message, recipient)
        EMAIL->>EMAILSVC: SendEmailAsync(template, recipient)
        EMAILSVC-->>EMAIL: Sent
        EMAIL->>NOTIFDB: LogSuccess(Channel=Email)
        NOTIFDB-->>EMAIL: Logged
        EMAIL-->>NH: true
    end

    NH-->>ED: Dispatch Complete
```

---

## 7. Asset Deletion with Cascading Events

```mermaid
sequenceDiagram
    actor User
    participant UI as Asset Master UI
    participant API as Asset API
    participant AB as Asset Business Service
    participant AES as AssetEventStore
    participant ED as EventDispatcher
    participant AEH as AssetEventHandler
    participant NH as NotificationHandler
    participant ASSETDB as Asset Master DB
    participant AUDITDB as Audit Log DB

    User->>UI: Delete Asset
    UI->>API: DELETE /assets/{id}
    API->>AB: ValidateAndDelete(assetId)
    AB->>ASSETDB: Delete Asset (Soft Delete)
    ASSETDB-->>AB: Asset Deleted
    AB->>AES: EmitEvent(AssetDeleted)
    AES-->>API: Event Stored
    API-->>UI: Asset Deleted

    AES->>ED: PublishEvent(AssetDeleted)
    
    par Handler Execution
        ED->>AEH: DispatchAsync(event)
        AEH->>AEH: HandleAssetDelete(event)
        AEH->>AUDITDB: LogAudit(assetDeleted)
        AUDITDB-->>AEH: Logged
        AEH-->>ED: Complete
    and
        ED->>NH: DispatchAsync(event)
        NH->>NH: ResolveChannels(admins, assetOwner)
        NH->>AUDITDB: LogNotification()
        AUDITDB-->>NH: Logged
        NH-->>ED: Complete
    end

    UI-->>User: Asset Removed
```

---

## 8. Error Handling in Document Scan

```mermaid
sequenceDiagram
    participant DH as DocumentHandler
    participant VOTIRO as Votiro CDR API
    participant SCANDB as DocumentScan DB
    participant AUDITDB as Audit Log DB
    participant ED as EventDispatcher
    participant NH as NotificationHandler

    DH->>VOTIRO: SubmitFileAsync(file)
    
    alt Votiro Service Available
        VOTIRO-->>DH: {correlationId}
        DH->>SCANDB: CreateDocumentScan(correlationId)
    else Votiro Service Error
        VOTIRO-->>DH: Error
        DH->>SCANDB: CreateDocumentScan(status=Failed)
        DH->>AUDITDB: LogScan(error=VotiroServiceError)
        DH->>ED: PublishEvent(ScanFailed)
        ED->>NH: DispatchAsync(event)
        NH->>NH: NotifyError(assetOwner, errorMsg)
        NH-->>DH: Notified
    end

    loop Polling
        DH->>VOTIRO: GetScanResultAsync(correlationId)
        
        alt Polling Successful
            VOTIRO-->>DH: ScanResult
            DH->>SCANDB: UpdateStatus(result)
        else Polling Failed - Retry
            VOTIRO-->>DH: Error
            DH->>SCANDB: IncrementPollingAttempt()
            
            alt Max Retries Not Exceeded
                Note over DH: Continue polling in next cycle
            else Max Retries Exceeded
                DH->>SCANDB: MarkFailed(maxRetriesExceeded)
                DH->>AUDITDB: LogScan(failed)
                DH->>ED: PublishEvent(ScanFailed)
                break Stop Polling
            end
        end
    end
```
