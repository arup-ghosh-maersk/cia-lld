# Asset Master — Component & Class Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Core Domain Models

```mermaid
classDiagram
    class Asset {
        +UUID assetId
        +String assetName
        +String assetType
        +String description
        +String location
        +String ownerUserId
        +DateTime createdAt
        +DateTime updatedAt
        +AssetStatus status
        +Validate() bool
        +ToDTO() AssetDTO
    }

    class AssetStatus {
        <<enumeration>>
        ACTIVE
        INACTIVE
        ARCHIVED
        RETIRED
    }

    class Attachment {
        +UUID attachmentId
        +UUID assetId
        +String fileName
        +String filePath
        +String contentType
        +Long fileSize
        +AttachmentStatus status
        +DateTime uploadedAt
        +String uploadedBy
    }

    class AttachmentStatus {
        <<enumeration>>
        PENDING
        CLEAN
        THREAT_DETECTED
        FAILED
        QUARANTINED
    }

    class DocumentScan {
        +UUID scanId
        +UUID attachmentId
        +String correlationId
        +DateTime scanStartTime
        +DateTime scanEndTime
        +ScanStatus status
        +String threatName
        +String scanResult
        +Int32 pollingAttempts
    }

    class ScanStatus {
        <<enumeration>>
        SUBMITTED
        IN_PROGRESS
        COMPLETED
        THREAT_DETECTED
        FAILED
        TIMEOUT
    }

    Asset "1" -- "*" Attachment : contains
    Attachment "1" -- "*" DocumentScan : scanned_by
    Attachment --> AttachmentStatus
    DocumentScan --> ScanStatus
    Asset --> AssetStatus
```

---

## 2. Event & Handler Architecture

```mermaid
classDiagram
    class Event {
        <<abstract>>
        +UUID eventId
        +String eventType
        +String correlationId
        +DateTime timestamp
        +String sourceId
        +Dict~String,Object~ metadata
        +Serialize() String
    }

    class AssetEvent {
        <<abstract>>
        +UUID assetId
    }

    class AssetCreatedEvent {
        +Asset assetData
        +String createdBy
    }

    class AssetUpdatedEvent {
        +UUID assetId
        +Dict~String,Object~ changes
        +String updatedBy
    }

    class AssetDeletedEvent {
        +UUID assetId
        +String deletedBy
    }

    class DocumentScanStartedEvent {
        +UUID attachmentId
        +String correlationId
    }

    class DocumentScanCompletedEvent {
        +UUID scanId
        +String scanResult
        +String threatName
    }

    class EventHandler {
        <<abstract>>
        #String handlerName
        +CanHandle(event Event) bool*
        +HandleAsync(event Event) Task
    }

    class AssetEventHandler {
        -IAssetRepository repository
        -IAuditStore auditStore
        +HandleAssetCreated(event AssetCreatedEvent) Task
        +HandleAssetUpdated(event AssetUpdatedEvent) Task
        +HandleAssetDeleted(event AssetDeletedEvent) Task
    }

    class DocumentEventHandler {
        -IDocumentScanRepository scanRepo
        -IAuditStore auditStore
        +HandleScanStarted(event DocumentScanStartedEvent) Task
        +HandleScanCompleted(event DocumentScanCompletedEvent) Task
    }

    class NotificationHandler {
        -INotificationService notificationSvc
        -IAuditStore auditStore
        +HandleAsync(event Event) Task
        -ResolveChannels(event Event) List~IChannel~
        -SendToChannels(channels List~IChannel~, message Message) Task
    }

    Event <|-- AssetEvent
    AssetEvent <|-- AssetCreatedEvent
    AssetEvent <|-- AssetUpdatedEvent
    AssetEvent <|-- AssetDeletedEvent
    Event <|-- DocumentScanStartedEvent
    Event <|-- DocumentScanCompletedEvent
    EventHandler <|-- AssetEventHandler
    EventHandler <|-- DocumentEventHandler
    EventHandler <|-- NotificationHandler
```

---

## 3. Event Dispatcher & Registry

```mermaid
classDiagram
    class EventDispatcher {
        -Dictionary~String,List~EventHandler~~ handlers
        -IAuditStore auditStore
        +RegisterHandler(eventType String, handler EventHandler) void
        +PublishEvent(event Event) Task
        +ResolveHandlers(event Event) List~EventHandler~
        -ExecuteHandlerAsync(handler EventHandler, event Event) Task
        -LogDispatchAudit(handler String, event Event, result HandlerResult) Task
    }

    class HandlerRegistry {
        -Dictionary~String,HandlerConfig~ registry
        +Register(eventType String, handler EventHandler, priority Int32) void
        +Unregister(eventType String, handler EventHandler) void
        +GetHandlers(eventType String) List~EventHandler~
        +GetByPriority(eventType String) List~EventHandler~
    }

    class HandlerConfig {
        +String handlerName
        +Type handlerType
        +Int32 priority
        +Bool async
        +TimeSpan timeout
    }

    class HandlerResult {
        <<enumeration>>
        SUCCESS
        FAILURE
        TIMEOUT
        SKIPPED
    }

    EventDispatcher --> HandlerRegistry
    HandlerRegistry --> HandlerConfig
    EventDispatcher --> HandlerResult
```

---

## 4. Notification System

```mermaid
classDiagram
    class INotificationChannel {
        <<interface>>
        +SendAsync(message Message, recipient String) Task~bool~
    }

    class UINotificationChannel {
        -ISignalRHubContext hubContext
        +SendAsync(message Message, recipient String) Task~bool~
        -ResolveConnectionIds(userId String) List~String~
    }

    class EmailNotificationChannel {
        -IEmailService emailSvc
        -ITemplateEngine templateEngine
        +SendAsync(message Message, recipient String) Task~bool~
        -BuildEmailContent(template String, data Object) String
    }

    class SMSNotificationChannel {
        -ISMSService smsSvc
        +SendAsync(message Message, recipient String) Task~bool~
    }

    class Message {
        +String title
        +String body
        +Dict~String,Object~ data
        +String templateId
        +DateTime createdAt
    }

    class NotificationLog {
        +UUID logId
        +String channelType
        +String recipient
        +NotificationStatus status
        +DateTime sentAt
        +String failureReason
    }

    class NotificationStatus {
        <<enumeration>>
        PENDING
        SENT
        DELIVERED
        FAILED
        BOUNCED
    }

    INotificationChannel <|-- UINotificationChannel
    INotificationChannel <|-- EmailNotificationChannel
    INotificationChannel <|-- SMSNotificationChannel
    NotificationLog --> NotificationStatus
```

---

## 5. Document Scanning Integration

```mermaid
classDiagram
    class IDocumentScanService {
        <<interface>>
        +SubmitFileAsync(file IFormFile, metadata Dict~String,Object~) Task~ScanResponse~
        +GetScanResultAsync(correlationId String) Task~ScanResult~
        +CancelScanAsync(correlationId String) Task~bool~
    }

    class VotiroCDRService {
        -HttpClient httpClient
        -String apiBaseUrl
        -String apiKey
        +SubmitFileAsync(file IFormFile, metadata Dict) Task~ScanResponse~
        +GetScanResultAsync(correlationId String) Task~ScanResult~
        -BuildRequest(file IFormFile) HttpRequestMessage
        -ParseResponse(response HttpResponseMessage) ScanResult
    }

    class ScanResponse {
        +String correlationId
        +DateTime submittedAt
        +String statusUrl
    }

    class ScanResult {
        +String correlationId
        +ScanStatus status
        +String threatName
        +String threatLevel
        +DateTime completedAt
        +Dict~String,Object~ metadata
    }

    class DocumentScanPoller {
        -IDocumentScanService scanSvc
        -IDocumentScanRepository scanRepo
        -IEventStore eventStore
        -TimeSpan pollInterval
        -Int32 maxRetries
        +StartPollingAsync(correlationId String, attachmentId UUID) Task
        -PollOnceAsync(correlationId String) Task
        -HandleScanCompletion(result ScanResult) Task
        -HandleScanFailure(correlationId String, reason String) Task
    }

    class IDocumentScanRepository {
        <<interface>>
        +CreateAsync(scan DocumentScan) Task
        +UpdateAsync(scan DocumentScan) Task
        +GetByCorrelationIdAsync(correlationId String) Task~DocumentScan~
        +GetByAttachmentIdAsync(attachmentId UUID) Task~DocumentScan~
    }

    IDocumentScanService <|-- VotiroCDRService
    VotiroCDRService --> ScanResponse
    VotiroCDRService --> ScanResult
    DocumentScanPoller --> IDocumentScanService
    DocumentScanPoller --> IDocumentScanRepository
    ScanResult --> ScanStatus
```

---

## 6. Repository & Data Access

```mermaid
classDiagram
    class IRepository {
        <<interface>>
        +CreateAsync(entity T) Task~Guid~
        +UpdateAsync(entity T) Task~bool~
        +DeleteAsync(id Guid) Task~bool~
        +GetByIdAsync(id Guid) Task~T~
        +GetAllAsync() Task~List~T~~
    }

    class IAssetRepository {
        <<interface>>
        +GetByIdAsync(id UUID) Task~Asset~
        +GetByOwnerAsync(ownerId String) Task~List~Asset~~
        +GetByStatusAsync(status AssetStatus) Task~List~Asset~~
        +SearchAsync(criteria SearchCriteria) Task~List~Asset~~
    }

    class IAttachmentRepository {
        <<interface>>
        +GetByAssetIdAsync(assetId UUID) Task~List~Attachment~~
        +GetByStatusAsync(status AttachmentStatus) Task~List~Attachment~~
    }

    class IAuditStore {
        <<interface>>
        +LogAuditAsync(audit AuditRecord) Task
        +LogDispatchAsync(dispatch DispatchAudit) Task
        +GetAuditsByAssetAsync(assetId UUID) Task~List~AuditRecord~~
        +GetDispatchLogsAsync(eventId UUID) Task~List~DispatchAudit~~
    }

    class IEventStore {
        <<interface>>
        +StoreEventAsync(event Event) Task~Guid~
        +GetEventsByCorrelationAsync(correlationId String) Task~List~Event~~
        +GetEventsByTypeAsync(eventType String) Task~List~Event~~
    }

    class AuditRecord {
        +UUID auditId
        +String entityType
        +UUID entityId
        +String action
        +String userId
        +DateTime timestamp
        +String oldValues
        +String newValues
        +String details
    }

    class DispatchAudit {
        +UUID dispatchId
        +UUID eventId
        +String handlerName
        +String status
        +DateTime startTime
        +DateTime endTime
        +String errorMessage
    }

    IRepository <|-- IAssetRepository
    IRepository <|-- IAttachmentRepository
```

---

## 7. Service Layer

```mermaid
classDiagram
    class IAssetService {
        <<interface>>
        +CreateAssetAsync(dto CreateAssetDTO) Task~AssetDTO~
        +UpdateAssetAsync(id UUID, dto UpdateAssetDTO) Task~AssetDTO~
        +DeleteAssetAsync(id UUID) Task~bool~
        +GetAssetAsync(id UUID) Task~AssetDTO~
        +ListAssetsAsync(filter AssetFilter) Task~PagedResult~AssetDTO~~
    }

    class AssetService {
        -IAssetRepository assetRepo
        -IEventStore eventStore
        -IValidator validator
        +CreateAssetAsync(dto CreateAssetDTO) Task~AssetDTO~
        +UpdateAssetAsync(id UUID, dto UpdateAssetDTO) Task~AssetDTO~
        +DeleteAssetAsync(id UUID) Task~bool~
        -PublishEventAsync(event Event) Task
        -ValidateAsset(asset Asset) ValidationResult
    }

    class IAttachmentService {
        <<interface>>
        +UploadAndScanAsync(assetId UUID, file IFormFile) Task~AttachmentDTO~
        +GetAttachmentsAsync(assetId UUID) Task~List~AttachmentDTO~~
        +DeleteAttachmentAsync(attachmentId UUID) Task~bool~
        +GetScanStatusAsync(attachmentId UUID) Task~DocumentScanDTO~
    }

    class AttachmentService {
        -IAttachmentRepository attachmentRepo
        -IDocumentScanService scanSvc
        -IDocumentScanPoller poller
        -IEventStore eventStore
        +UploadAndScanAsync(assetId UUID, file IFormFile) Task~AttachmentDTO~
        -ValidateFile(file IFormFile) ValidationResult
        -SubmitToScanAsync(attachment Attachment) Task
    }

    IAssetService <|-- AssetService
    IAttachmentService <|-- AttachmentService
```

---

## 8. Dependency Injection & Configuration

```mermaid
classDiagram
    class ServiceRegistration {
        <<static>>
        +RegisterCoreServices(services IServiceCollection) IServiceCollection$
        +RegisterRepositories(services IServiceCollection) IServiceCollection$
        +RegisterEventHandling(services IServiceCollection) IServiceCollection$
        +RegisterNotificationChannels(services IServiceCollection) IServiceCollection$
    }

    class EventHandlerConfiguration {
        <<static>>
        +ConfigureHandlers(dispatcher EventDispatcher) void$
        +RegisterAssetHandlers(dispatcher EventDispatcher) void$
        +RegisterDocumentHandlers(dispatcher EventDispatcher) void$
        +RegisterNotificationHandlers(dispatcher EventDispatcher) void$
    }

    class PollConfiguration {
        +TimeSpan PollInterval
        +Int32 MaxRetries
        +Int32 TimeoutSeconds
        +Bool EnableBackoffStrategy
    }

    class NotificationConfiguration {
        +List~String~ EnabledChannels
        +Dict~String,String~ ChannelConfig
        +Int32 RetryCount
        +TimeSpan RetryDelay
    }

    class ScanServiceConfiguration {
        +String VotiroApiUrl
        +String VotiroApiKey
        +TimeSpan RequestTimeout
        +Bool VerifySSL
    }

    ServiceRegistration --> EventHandlerConfiguration
    ServiceRegistration --> PollConfiguration
    ServiceRegistration --> NotificationConfiguration
    ServiceRegistration --> ScanServiceConfiguration
```
