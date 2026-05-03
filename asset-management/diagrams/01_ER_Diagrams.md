# Asset Master — ER Diagrams

> **Module:** Asset Master System | **Version:** 1.0

---

## 1. Complete ER Diagram

```mermaid
erDiagram
    ASSET_TEMPLATE ||--o{ ASSET_TEMPLATE : "parent_of"
    ASSET_TEMPLATE ||--o{ ASSET_MASTER : "instantiated_as"
    ASSET_TEMPLATE ||--o{ TEMPLATE_COMPONENT : "defines"
    ASSET_TEMPLATE ||--o{ TEMPLATE_ATTACHMENT : "defines"
    ASSET_TEMPLATE ||--o{ TEMPLATE_ATTRIBUTE : "defines"
    ASSET_MASTER ||--o{ ASSET_MASTER : "parent_of"
    ASSET_MASTER ||--o{ MASTER_COMPONENT : "has"
    ASSET_MASTER ||--o{ MASTER_ATTACHMENT : "has"
    ASSET_MASTER ||--o{ MASTER_ATTRIBUTE : "has"
    ASSET_MASTER ||--o{ ASSET_EVENT_STORE : "generates"
    ATTRIBUTE_DEF ||--o{ TEMPLATE_ATTRIBUTE : "referenced_by"
    ATTRIBUTE_DEF ||--o{ MASTER_ATTRIBUTE : "referenced_by"
    TEMPLATE_ATTRIBUTE ||--o{ MASTER_ATTRIBUTE : "sourced_from"
    COMPONENT ||--o{ TEMPLATE_COMPONENT : "referenced_by"
    COMPONENT ||--o{ MASTER_COMPONENT : "referenced_by"
    ATTACHMENT ||--o{ TEMPLATE_ATTACHMENT : "referenced_by"
    ATTACHMENT ||--o{ MASTER_ATTACHMENT : "references"
    ATTACHMENT ||--o{ DOCUMENT_SCAN : "scanned_by"
    TEMPLATE_COMPONENT ||--o{ MASTER_COMPONENT : "sourced_from"
    TEMPLATE_ATTACHMENT ||--o{ MASTER_ATTACHMENT : "sourced_from"
    ASSET_EVENT_STORE ||--o{ ASSET_AUDIT : "produces"
    ASSET_AUDIT ||--o{ NOTIFICATION_LOG : "dispatched_as"
```

---

## 2. Core Asset Domain

```mermaid
erDiagram
    ASSET_TEMPLATE ||--o{ ASSET_TEMPLATE : "parent_of"
    ASSET_TEMPLATE ||--o{ ASSET_MASTER : "instantiated_as"
    ASSET_MASTER ||--o{ ASSET_MASTER : "parent_of"

    ASSET_TEMPLATE {
        uuid id PK
        uuid parent_template_id FK
        string object_id UK
        string object_type
        int object_level
        string category
        string functional_type
        string hierarchy_path
    }

    ASSET_MASTER {
        uuid id PK
        uuid asset_template_id FK
        uuid parent_asset_id FK
        string asset_code UK
        string name
        string serial_number
        string status
        date installation_date
    }
```

---

## 3. Attributes Domain

```mermaid
erDiagram
    ATTRIBUTE_DEF ||--o{ TEMPLATE_ATTRIBUTE : "referenced_by"
    ATTRIBUTE_DEF ||--o{ MASTER_ATTRIBUTE : "referenced_by"
    ASSET_TEMPLATE ||--o{ TEMPLATE_ATTRIBUTE : "defines"
    ASSET_MASTER ||--o{ MASTER_ATTRIBUTE : "has"
    TEMPLATE_ATTRIBUTE ||--o{ MASTER_ATTRIBUTE : "sourced_from"

    ATTRIBUTE_DEF {
        uuid id PK
        string attribute_key UK
        string display_name
        string data_type
        string default_value
        jsonb valid_values
        string unit
        string group_name
        int sort_order
        boolean is_required
        jsonb validation_rules
        string notes
    }

    TEMPLATE_ATTRIBUTE {
        uuid id PK
        uuid asset_template_id FK
        uuid attribute_def_id FK
        boolean is_mandatory
        int sort_order
        string notes
    }

    MASTER_ATTRIBUTE {
        uuid id PK
        uuid asset_master_id FK
        uuid attribute_def_id FK
        uuid template_attribute_id FK
        string source
        string value_string
        decimal value_number
        date value_date
        boolean value_boolean
        jsonb value_json
        string validation_status
        jsonb validation_errors
        boolean is_overridden
    }
```

---

## 4. Components & Attachments Domain

```mermaid
erDiagram
    ASSET_TEMPLATE ||--o{ TEMPLATE_COMPONENT : "defines"
    ASSET_TEMPLATE ||--o{ TEMPLATE_ATTACHMENT : "defines"
    ASSET_MASTER ||--o{ MASTER_COMPONENT : "has"
    ASSET_MASTER ||--o{ MASTER_ATTACHMENT : "has"
    COMPONENT ||--o{ TEMPLATE_COMPONENT : "referenced_by"
    COMPONENT ||--o{ MASTER_COMPONENT : "referenced_by"
    ATTACHMENT ||--o{ TEMPLATE_ATTACHMENT : "referenced_by"
    ATTACHMENT ||--o{ MASTER_ATTACHMENT : "referenced_by"
    TEMPLATE_COMPONENT ||--o{ MASTER_COMPONENT : "sourced_from"
    TEMPLATE_ATTACHMENT ||--o{ MASTER_ATTACHMENT : "sourced_from"

    COMPONENT {
        uuid id PK
        string component_code UK
        string name
        jsonb specifications
    }

    ATTACHMENT {
        uuid id PK
        string file_name
        string file_type
        string category
        bigint file_size
        string scan_status
        timestamp uploaded_at
    }

    TEMPLATE_COMPONENT {
        uuid id PK
        uuid asset_template_id FK
        uuid component_id FK
        int quantity
        boolean is_mandatory
    }

    TEMPLATE_ATTACHMENT {
        uuid id PK
        uuid asset_template_id FK
        uuid attachment_id FK
        boolean is_mandatory
    }

    MASTER_COMPONENT {
        uuid id PK
        uuid asset_master_id FK
        uuid component_id FK
        uuid template_component_id FK
        string source
        string status
    }

    MASTER_ATTACHMENT {
        uuid id PK
        uuid asset_master_id FK
        uuid attachment_id FK
        uuid template_attachment_id FK
        string source
        string status
    }
```

---

## 5. Event & Audit Domain

```mermaid
erDiagram
    ASSET_MASTER ||--o{ ASSET_EVENT_STORE : "generates"
    ASSET_EVENT_STORE ||--o{ ASSET_AUDIT : "produces"
    ASSET_AUDIT ||--o{ NOTIFICATION_LOG : "dispatched_as"

    ASSET_EVENT_STORE {
        uuid EventId PK
        uint EventType
        jsonb EventDetails
        timestamp created_at
        string created_by
    }

    ASSET_AUDIT {
        uuid NotificationId PK
        uuid event_id FK
        string AuditMessage
        jsonb Detail
        string status
        timestamp created_at
        string created_by
    }

    NOTIFICATION_LOG {
        uuid id PK
        uuid notification_id FK
        string channel
        string recipient
        string status
        int retry_count
        string error_message
        timestamp sent_at
    }
```

---

## 6. Document Scan Domain

```mermaid
erDiagram
    ATTACHMENT ||--o{ DOCUMENT_SCAN : "scanned_by"
    MASTER_ATTACHMENT ||--|| ATTACHMENT : "references"
    TEMPLATE_ATTACHMENT ||--|| ATTACHMENT : "references"

    ATTACHMENT {
        uuid id PK
        string file_name
        string file_path
        string mime_type
        string category
        bigint file_size
        string scan_status
        timestamp uploaded_at
        uuid uploaded_by FK
    }

    DOCUMENT_SCAN {
        uuid id PK
        uuid attachment_id FK
        string scan_engine
        string scan_status
        string scan_result_detail
        string sanitized_file_path
        timestamp scanned_at
        string scan_reference_id
    }
```
