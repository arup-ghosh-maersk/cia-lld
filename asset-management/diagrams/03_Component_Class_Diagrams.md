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
        end

        subgraph "Handler Services"
            AEH[AssetEventHandler]
            NH[NotificationHandler]
            DH[DocumentHandler]
        end

        subgraph "External Services"
            VOTIRO[Votiro CDR API]
            EMAIL[Email Service]
            STORAGE[File Storage]
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

    AES -->|Event Stream| AEH
    AEH -->|Log Audit| AUDITDB
    AEH -->|Notify| NH

    NH -->|Push| SIGNALR
    NH -->|Send| EMAIL
    NH -->|Log| AUDITDB
    SIGNALR -->|Display| UI

    UI -->|Upload File| API
    API -->|Store & Scan| DH
    DH -->|Quarantine| STORAGE
    DH -->|Submit| VOTIRO
    DH -->|Webhook Callback| WEBHOOK
    WEBHOOK -->|Dispatch| NH
    DH -->|Log Scan| AUDITDB
    VOTIRO -->|Result| DH
    DH -->|Move to Secure| STORAGE

    AB -->|Query| ASSETDB
    AUDITDB -->|Store Logs| ASSETDB

    style AEH fill:#e1f5ff
    style NH fill:#f3e5f5
    style DH fill:#fff3e0
    style API fill:#e8f5e9
    style SIGNALR fill:#fce4ec
    style VOTIRO fill:#ffe0b2
```

---

## 2. AssetEventHandler Class Diagram

```mermaid
classDiagram
    class EventHandler {
        <<abstract>>
        -eventBus: IEventBus
        -logger: ILogger
        -database: IRepository
        +HandleAsync(event) Task
        #LogEvent(msg) void
    }

    class AssetEventHandler {
        -auditRepository: IAuditRepository
        -notificationHandler: INotificationHandler
        +HandleAsync(event) Task
        -GenerateAuditMessage(event) string
        -HandleAssetCreate(event) Task
        -HandleAssetUpdate(event) Task
        -HandleAssetDelete(event) Task
        -ExtractModifiedFields(before, after) List~string~
    }

    class AssetEvent {
        -eventId: Guid
        -eventType: EventType
        -eventDetails: JsonObject
        -createdAt: DateTime
        -createdBy: string
        +GetAssetId() Guid
        +GetEventType() EventType
    }

    class AuditRecord {
        -notificationId: Guid
        -eventId: Guid
        -auditMessage: string
        -detail: JsonObject
        -status: AuditStatus
        -createdAt: DateTime
        +MarkDispatched() void
        +MarkFailed(reason) void
    }

    class NotificationRequest {
        -notificationId: Guid
        -message: string
        -recipients: List~string~
        -priority: Priority
    }

    EventHandler <|-- AssetEventHandler
    AssetEventHandler --> AssetEvent
    AssetEventHandler --> AuditRecord
    AssetEventHandler --> NotificationRequest
```

---

## 3. NotificationHandler Class Diagram

```mermaid
classDiagram
    class NotificationHandler {
        -channels: Dictionary~Channel, INotificationChannel~
        -notificationLogRepository: IRepository
        +DispatchAsync(request) Task
        -ResolveChannels(recipients) List~Channel~
        -SendToUIAsync(msg, recipient) Task
        -SendEmailAsync(msg, recipient) Task
        -SendWebhookAsync(msg, endpoint) Task
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

    class WebhookNotificationChannel {
        -httpClient: HttpClient
        -retryPolicy: IRetryPolicy
        +SendAsync(message, endpoint) Task~bool~
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
    }

    NotificationHandler --> INotificationChannel
    INotificationChannel <|-- UINotificationChannel
    INotificationChannel <|-- EmailNotificationChannel
    INotificationChannel <|-- WebhookNotificationChannel
    NotificationHandler --> NotificationLog
```

---

## 4. DocumentHandler Class Diagram

```mermaid
classDiagram
    class DocumentHandler {
        -fileStorage: IFileStorageService
        -scanService: IScanService
        -attachmentRepository: IRepository
        -documentScanRepository: IRepository
        -notificationHandler: INotificationHandler
        +UploadAndScanAsync(file, metadata) Task~Attachment~
        -StoreQuarantineFile(file) Task~string~
        -SubmitForScanAsync(attachmentId, filePath) Task
        -HandleScanResultAsync(scanResult) Task
        -MoveToSecureStorageAsync(filePath, assetId) Task~string~
    }

    class VotiroScanService {
        <<implements IScanService>>
        -apiClient: HttpClient
        -apiKey: string
        +SubmitScanAsync(filePath, referenceId) Task~string~
        +GetScanResultAsync(scanReferenceId) Task~ScanResult~
        +RegisterWebhookAsync(url) Task
    }

    class FileStorageService {
        <<implements IFileStorageService>>
        -quarantinePath: string
        -documentPath: string
        +SaveQuarantineAsync(file, tempId) Task~string~
        +MoveFromQuarantineAsync(tempPath, targetPath) Task~string~
        +DeleteAsync(filePath) Task
        +GetFileStreamAsync(filePath) Task~Stream~
    }

    class Attachment {
        -id: Guid
        -fileName: string
        -filePath: string
        -mimeType: string
        -category: AttachmentCategory
        -fileSize: long
        -scanStatus: ScanStatus
        +UpdateScanStatus(status) void
        +SetFilePath(path) void
    }

    class DocumentScan {
        -id: Guid
        -attachmentId: Guid
        -scanEngine: string
        -scanStatus: ScanStatus
        -scanResultDetail: string
        -sanitizedFilePath: string
        -scanReferenceId: string
        +MarkClean(sanitizedPath) void
        +MarkInfected(threatName) void
        +MarkFailed(reason) void
    }

    class ScanResult {
        -referenceId: string
        -status: ScanStatus
        -threatName: string
        -sanitizedFilePath: string
        -scanDuration: TimeSpan
    }

    DocumentHandler --> VotiroScanService
    DocumentHandler --> FileStorageService
    DocumentHandler --> Attachment
    DocumentHandler --> DocumentScan
    VotiroScanService --> ScanResult
```
