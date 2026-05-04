# Asset Master — Component & Class Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Core Domain Models

```mermaid
classDiagram
    class GosRegister {
        +UUID id
        +String gosObjectType
        +String gosObjectId
        +String gosLv1Code
        +String gosLv2Code
        +String gosLv3Code
        +String gosLv4Code
        +String gosLv5Code
        +Int gosObjectLevel
        +String gosCategory
        +String gosFunctionalType
        +String gosEquipmentType
        +String gosHierarchyPath
        +Bool gosIsTool
        +String displayName
        +String status
        +IsRoot() bool
        +GetChildren() List~GosRegister~
        +GetAncestors() List~GosRegister~
    }

    class GosCategory {
        <<enumeration>>
        EQ
        TOOL
        CIV
        FUEL
    }

    class OemTemplate {
        +UUID id
        +UUID gosRegisterId
        +String templateCode
        +String templateName
        +String oemManufacturer
        +String oemModel
        +String status
        +GetLatestVersion() OemTemplateVersion
        +GetVersion(versionNumber Int) OemTemplateVersion
        +CreateNewVersion() OemTemplateVersion
    }

    class OemTemplateVersion {
        +UUID id
        +UUID oemTemplateId
        +UUID parentVersionId
        +String versionCode
        +Int versionNumber
        +Bool isLatestVersion
        +Date effectiveFrom
        +Date effectiveTo
        +String status
        +Dict specifications
        +GetPreviousVersion() OemTemplateVersion
        +CloneAsNewVersion() OemTemplateVersion
        +IsActive() bool
    }

    class VersionStatus {
        <<enumeration>>
        DRAFT
        ACTIVE
        DEPRECATED
        RETIRED
    }

    class Asset {
        +UUID assetId
        +UUID templateVersionId
        +UUID parentAssetId
        +Int hierarchyLevel
        +String assetCode
        +String assetName
        +String description
        +String assetType
        +String serialNumber
        +String manufacturer
        +String model
        +String location
        +String ownerUserId
        +DateTime installationDate
        +DateTime commissioningDate
        +Decimal acquisitionCost
        +AssetStatus status
        +CreationMode creationMode
        +Boolean isActive
        +Validate() bool
        +ToDTO() AssetDTO
        +CanCreateChildAsset() bool
    }

    class AssetStatus {
        <<enumeration>>
        ACTIVE
        INACTIVE
        RETIRED
        MAINTENANCE
    }

    class CreationMode {
        <<enumeration>>
        TEMPLATE_BASED
        AD_HOC
    }

    class Attachment {
        +UUID attachmentId
        +UUID assetId
        +String fileName
        +String filePath
        +String mimeType
        +Long fileSize
        +String fileHash
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
        +String scanEngine
        +DateTime scanStartTime
        +DateTime scanEndTime
        +ScanStatus status
        +String threatName
        +String scanResult
        +Int32 pollingAttempts
        +Int32 maxPollingAttempts
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

    GosRegister --> GosCategory
    GosRegister "1" -- "*" GosRegister : parent_of
    GosRegister "1" -- "*" OemTemplate : extended_by
    OemTemplate "1" -- "*" OemTemplateVersion : has_versions
    OemTemplateVersion --> VersionStatus
    OemTemplateVersion "1" -- "*" OemTemplateVersion : versioned_as
    OemTemplateVersion "0..1" -- "*" Asset : may_instantiate
    Asset --> AssetStatus
    Asset --> CreationMode
    Asset "1" -- "*" Asset : parent_of
    Asset "1" -- "*" Attachment : contains
    Attachment "1" -- "*" DocumentScan : scanned_by
    Attachment --> AttachmentStatus
    DocumentScan --> ScanStatus
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
