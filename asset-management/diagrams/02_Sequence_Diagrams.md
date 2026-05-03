# Asset Master — Sequence Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Asset Create

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Asset UI
    participant API as Asset API
    participant DB as Database
    participant ES as EventStore
    participant Handler as EventHandler
    participant Audit as AuditStore
    participant Push as Notification

    User->>UI: Fill form & Submit
    UI->>API: POST /api/v1/assets
    API->>DB: INSERT Asset, Components, Attachments
    DB-->>API: AssetId: ASSET001

    API->>ES: INSERT Event {Type: AssetCreation, Details: {...}}
    ES-->>API: EventId (uuid)
    API-->>UI: 201 Created

    Note over ES,Push: Async Processing
    Handler->>ES: Consume AssetCreation Event
    ES-->>Handler: Event Payload
    Handler->>Handler: GenerateAuditMessage()
    Handler->>Audit: INSERT AuditRecord
    Audit-->>Handler: NotificationId
    Handler->>Push: Dispatch Notification
    Push-->>UI: Push — "Asset ASSET001 Created"
```

---

## 2. Asset Update

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Asset UI
    participant API as Asset API
    participant DB as Database
    participant ES as EventStore
    participant Handler as EventHandler
    participant Audit as AuditStore
    participant Push as Notification

    User->>UI: Modify Asset & Submit
    UI->>API: PUT /api/v1/assets/ASSET001
    API->>DB: UPDATE Asset Record
    DB-->>API: Updated

    API->>ES: INSERT Event {Type: AssetUpdate, Details: {...}}
    ES-->>API: EventId (uuid)
    API-->>UI: 200 OK

    Note over ES,Push: Async Processing
    Handler->>ES: Consume AssetUpdate Event
    ES-->>Handler: Event Payload
    Handler->>Handler: GenerateAuditMessage()
    Handler->>Audit: INSERT AuditRecord
    Audit-->>Handler: NotificationId
    Handler->>Push: Dispatch Notification
    Push-->>UI: Push — "Asset ASSET001 Updated"
```

---

## 3. Audit Handler Flow

```mermaid
sequenceDiagram
    autonumber
    participant API as Asset API
    participant ES as AssetEventStore
    participant AEH as AssetEventHandler
    participant AA as AssetAudit
    participant NH as NotificationHandler

    API->>ES: INSERT Event {Type, EventDetails}
    Note over ES,AEH: Async — event consumed by handler
    AEH->>ES: Consume(event)
    ES-->>AEH: Event Payload

    AEH->>AEH: Identify EventType (Create / Update / Delete)
    AEH->>AEH: GenerateAuditMessage()

    AEH->>AA: INSERT AssetAudit {event_id, AuditMessage, status=PENDING}
    AA-->>AEH: NotificationId

    AEH->>NH: Dispatch(NotificationId, message)
    NH-->>AEH: Dispatched OK
    AEH->>AA: UPDATE status = DISPATCHED
```

---

## 4. Notification Handler Flow

```mermaid
sequenceDiagram
    autonumber
    participant AEH as AssetEventHandler
    participant NH as NotificationHandler
    participant NL as NotificationLog
    participant UI as Asset Master UI
    participant EMAIL as Email Service

    AEH->>NH: Dispatch(NotificationId, message, recipients)
    NH->>NH: Resolve channels per recipient (UI_PUSH, EMAIL, WEBHOOK)

    par UI Push
        NH->>UI: SignalR Push — "Asset ASSET001 Created"
        UI-->>NH: Acknowledged
        NH->>NL: INSERT {channel=UI_PUSH, status=SENT}
    and Email
        NH->>EMAIL: Send Email Notification
        EMAIL-->>NH: Delivered
        NH->>NL: INSERT {channel=EMAIL, status=SENT}
    end

    NH-->>AEH: All channels dispatched
```

---

## 5. Document Upload with Votiro Scan

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Asset Master UI
    participant API as Asset API
    participant DH as DocumentHandler
    participant VOTIRO as Votiro API
    participant STORE as File Storage
    participant DB as Database

    User->>UI: Upload file (e.g. Manual.pdf)
    UI->>API: POST /api/v1/attachments {file, category, assetId}

    API->>STORE: Store in quarantine bucket
    STORE-->>API: temp_file_path

    API->>DB: INSERT ATTACHMENT {file_name, scan_status=PENDING}
    DB-->>API: attachment_id

    API->>DH: SubmitForScan(attachment_id, temp_file_path)
    DH->>VOTIRO: POST /api/scan {file_stream, reference_id}
    VOTIRO-->>DH: {scan_reference_id, status=PENDING}

    DH->>DB: INSERT DOCUMENT_SCAN {attachment_id, scan_engine=VOTIRO, status=PENDING}
    API-->>UI: 202 Accepted {attachment_id, scan_status=PENDING}
    UI-->>User: ⏳ "File uploaded, scanning in progress..."

    Note over DH,VOTIRO: Async — Votiro CDR processing

    VOTIRO-->>DH: Webhook {scan_reference_id, status, sanitized_file_path}

    alt CLEAN
        DH->>STORE: Move sanitized file to /documents/{assetId}/
        DH->>DB: UPDATE ATTACHMENT {file_path, scan_status=CLEAN}
        DH->>DB: UPDATE DOCUMENT_SCAN {scan_status=CLEAN, sanitized_file_path}
        DH-->>UI: Push ✅ "Manual.pdf is ready"
    else INFECTED
        DH->>STORE: Delete quarantine file
        DH->>DB: UPDATE ATTACHMENT {scan_status=INFECTED}
        DH->>DB: UPDATE DOCUMENT_SCAN {scan_status=INFECTED, threat_name}
        DH-->>UI: Push ❌ "Manual.pdf failed security scan"
    end
```
