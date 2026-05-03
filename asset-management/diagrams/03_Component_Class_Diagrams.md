# Asset Master — Component & Class Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Overall System Architecture

```mermaid
graph LR
    subgraph "Asset Master System"
        UI[Asset Master UI]
        API[Asset API Layer]

        subgraph "Core Services"
            AES[AssetEventStore]
            AB[AssetBusiness Service]
            ED[EventDispatcher]
        end

        subgraph "Handler Services"
            AEH[AssetEventHandler]
            NH[NotificationHandler]
            DH[DocumentHandler]
        end

        subgraph "External Services"
            VOTIRO[Votiro CDR API]
            EMAIL[Email Service]
        end

        subgraph "Databases"
            ASSETDB[(Asset Master DB)]
            AUDITDB[(Audit Log DB)]
        end

        subgraph "Communication"
            SIGNALR[SignalR Hub]
            WEBHOOK[Event Bus]
        end
    end

    UI -->|Create / Update Asset| API
    API -->|Validate & Persist| AB
    AB -->|Emit Event| AES

    AES -->|Retrieve Events| ED
    ED -->|Dispatch to Handler| AEH
    ED -->|Dispatch to Handler| NH
    ED -->|Dispatch to Handler| DH
    
    AEH -->|Log Audit| AUDITDB
    AEH -->|Notify via| NH

    NH -->|Push| SIGNALR
    NH -->|Send| EMAIL
    NH -->|Log| AUDITDB
    SIGNALR -->|Display| UI

    UI -->|Upload File| API
    API -->|Upload & Scan| DH
    DH -->|Direct Upload + Get CorrelationId| VOTIRO
    DH -->|Poll for Result| VOTIRO
    DH -->|Webhook Callback| WEBHOOK
    WEBHOOK -->|Dispatch| ED
    DH -->|Log Scan| AUDITDB
    DH -->|Notify via| NH

    AB -->|Query| ASSETDB
    AUDITDB -->|Store Logs| ASSETDB

    style ED fill:#c8e6c9
    style AEH fill:#e1f5ff
    style NH fill:#f3e5f5
    style DH fill:#fff3e0
    style API fill:#e8f5e9
    style SIGNALR fill:#fce4ec
    style VOTIRO fill:#ffe0b2
```

---

## 2. Handler Base Architecture

```mermaid
classDiagram
    class BaseHandler {
        <<abstract>>
        -eventBus: IEventBus
        -logger: ILogger
        -auditRepository: IAuditRepository
        #HandleAsync(event) Task
        #LogAudit(handlerName, message, detail) Task
        #PublishEvent(event) Task
        #NotifyOnCompletion(notification) Task
    }

    class AssetEventHandler {
        -notificationHandler: INotificationHandler
        +HandleAsync(event) Task
        -HandleAssetCreate(event) Task
        -HandleAssetUpdate(event) Task
        -HandleAssetDelete(event) Task
        -ExtractModifiedFields(before, after) List~string~
    }

    class NotificationHandler {
        -channels: Dictionary~Channel, INotificationChannel~
        -notificationLogRepository: IRepository
        +HandleAsync(request) Task
        -ResolveChannels(recipients) List~Channel~
        -DispatchToChannels(msg, recipients) Task
    }

    class DocumentHandler {
        -votiroService: IVotiroService
        -attachmentRepository: IRepository
        -documentScanRepository: IRepository
        +HandleAsync(file, metadata) Task
        -SubmitForScanAsync(file) Task~string~
        -PollScanResultAsync(correlationId) Task
        -HandleScanCompletion(result) Task
    }

    BaseHandler <|-- AssetEventHandler
    BaseHandler <|-- NotificationHandler
    BaseHandler <|-- DocumentHandler

    class AssetEvent {
        -eventId: Guid
        -eventType: EventType
        -eventDetails: JsonObject
        -createdAt: DateTime
        -createdBy: string
    }

    class AuditRecord {
        -auditId: Guid
        -handlerName: string
        -referenceId: Guid
        -auditMessage: string
        -detail: JsonObject
        -status: AuditStatus
        -createdAt: DateTime
    }    AssetEventHandler --> AssetEvent
    AssetEventHandler --> AuditRecord
    NotificationHandler --> AuditRecord
    DocumentHandler --> AuditRecord
```

---

## 3. EventDispatcher & Handler Coordination

```mermaid
classDiagram
    class IEventHandler {
        <<interface>>
        +CanHandle(event) bool
        +HandleAsync(event) Task
        +GetHandlerType() HandlerType
    }

    class EventDispatcher {
        -handlers: List~IEventHandler~
        -logger: ILogger
        -auditRepository: IAuditRepository
        +RegisterHandler(handler) void
        +DispatchAsync(event) Task
        -ResolveHandlers(event) List~IEventHandler~
        -ExecuteHandlerAsync(handler, event) Task
        -LogDispatchAudit(event, handler, result) Task
    }

    class AssetEventHandler {
        +CanHandle(event) bool
        +HandleAsync(event) Task
        +GetHandlerType() HandlerType
        -HandleAssetCreate(event) Task
        -HandleAssetUpdate(event) Task
        -HandleAssetDelete(event) Task
    }

    class NotificationHandler {
        +CanHandle(event) bool
        +HandleAsync(event) Task
        +GetHandlerType() HandlerType
        -channels: Dictionary~Channel, INotificationChannel~
    }

    class DocumentHandler {
        +CanHandle(event) bool
        +HandleAsync(event) Task
        +GetHandlerType() HandlerType
        -votiroService: IVotiroService
    }

    class AssetEvent {
        -eventId: Guid
        -eventType: EventType
        -eventDetails: JsonObject
        -createdAt: DateTime
        -createdBy: string
        +GetEventType() EventType
    }

    class DispatchAudit {
        -id: Guid
        -eventId: Guid
        -handlerType: HandlerType
        -dispatchStatus: DispatchStatus
        -handlerResult: string
        -executionTime: TimeSpan
        -dispatchedAt: DateTime
        -completedAt: DateTime
    }

    EventDispatcher --> IEventHandler
    IEventHandler <|-- AssetEventHandler
    IEventHandler <|-- NotificationHandler
    IEventHandler <|-- DocumentHandler
    EventDispatcher --> AssetEvent
    EventDispatcher --> DispatchAudit
```

---

## 4. NotificationHandler Class Diagram

```mermaid
classDiagram
    class NotificationHandler {
        -channels: Dictionary~Channel, INotificationChannel~
        -notificationLogRepository: IRepository
        +HandleAsync(request) Task
        -ResolveChannels(recipients) List~Channel~
        -SendToUIAsync(msg, recipient) Task
        -SendEmailAsync(msg, recipient) Task
    }

    class INotificationChannel {
        <<interface>>
        +SendAsync(message, recipient) Task~bool~
        +GetChannelType() Channel
    }

    class UINotificationChannel {
        -signalRHub: IHubContext
        +SendAsync(message, recipient) Task~bool~
        +GetChannelType() Channel
    }

    class EmailNotificationChannel {
        -emailService: IEmailService
        -templateEngine: ITemplateEngine
        +SendAsync(message, recipient) Task~bool~
        +GetChannelType() Channel
    }

    class NotificationLog {
        -id: Guid
        -notificationId: Guid
        -channel: Channel
        -recipient: string
        -status: NotificationStatus
        -retryCount: int
        -errorMessage: string
        -sentAt: DateTime
        +LogSuccess(sentAt) void
        +LogFailure(error) void
        +IncrementRetry() void
    }---

## 5. DocumentHandler & Votiro Integration

```mermaid
classDiagram
    class DocumentHandler {
        -votiroService: IVotiroService
        -attachmentRepository: IRepository
        -documentScanRepository: IRepository
        -notificationHandler: INotificationHandler
        +UploadAndScanAsync(file, metadata) Task~Attachment~
        -SubmitForScanAsync(file, metadata) Task~string~
        -PollScanStatusAsync(correlationId) Task~ScanResult~
        -HandleScanCompletion(result) Task
        -NotifyScanResult(attachment, result) Task
    }

    class IVotiroService {
        <<interface>>
        +SubmitFileAsync(file, metadata) Task~VotiroResponse~
        +GetScanResultAsync(correlationId) Task~ScanResult~
    }

    class VotiroService {
        <<implements IVotiroService>>
        -apiClient: HttpClient
        -apiKey: string
        -baseUrl: string
        +SubmitFileAsync(file, metadata) Task~VotiroResponse~
        +GetScanResultAsync(correlationId) Task~ScanResult~
        -BuildAuthHeaders() Dictionary~string, string~
    }

    class VotiroResponse {
        -correlationId: string
        -status: string
        -message: string
        -timestamp: DateTime
    }

    class ScanResult {
        -correlationId: string
        -status: ScanStatus
        -threatDetected: bool
        -threatName: string
        -scanDuration: TimeSpan
        -processedAt: DateTime
    }

    class Attachment {
        -id: Guid
        -fileName: string
        -mimeType: string
        -fileSize: long
        -correlationId: string
        -scanStatus: ScanStatus
        -scanResult: string
        -assetId: Guid
        +UpdateScanStatus(status) void
        +SetCorrelationId(id) void
        +MarkScanned(result) void
    }

    class DocumentScan {
        -id: Guid
        -attachmentId: Guid
        -correlationId: string
        -scanStatus: ScanStatus
        -scanResultDetail: string
        -threatDetected: bool
        -threatName: string
        -pollingAttempts: int
        -lastPolledAt: DateTime
        +MarkClean() void
        +MarkThreatDetected(threat) void
        +MarkFailed(reason) void
        +IncrementPollingAttempt() void
    }

    DocumentHandler --> IVotiroService
    DocumentHandler --> Attachment
    DocumentHandler --> DocumentScan
    DocumentHandler --> VotiroResponse
    IVotiroService --> VotiroResponse
    IVotiroService --> ScanResult
    DocumentScan --> ScanResult
```
