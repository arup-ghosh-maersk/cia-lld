# Components & Attachments — Visual Overview

> **Focus:** Component registry, file pipelines | **Diagrams & flows**

---

## 1. Component System Overview

```
┌────────────────────────────────────────────────────────┐
│           COMPONENT REGISTRY & SYSTEM                 │
├────────────────────────────────────────────────────────┤
│                                                        │
│ Components are reusable, typed sub-parts that can be  │
│ attached to assets to build complex structures.       │
│                                                        │
│ Examples:                                             │
│ ├─ Engine (has power, fuel type, displacement)       │
│ ├─ Transmission (has gears, type, efficiency)        │
│ ├─ Battery (has voltage, capacity, chemistry)        │
│ ├─ Pump (has flow rate, pressure, material)          │
│ └─ Wheel (has size, material, tread pattern)         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 2. Component Registry Model

```
COMPONENT (Master Definition)
═════════════════════════════════════════════════════════

ID: engine-001
Name: "Engine"
Type: "Engine"
Category: "Power Unit"

Attributes:
├─ power_output (NUMBER, kW)
├─ fuel_type (ENUM: Diesel, Petrol, Electric)
├─ displacement (NUMBER, cc)
├─ cylinders (NUMBER, 1-12)
├─ max_rpm (NUMBER, 0-10000)
└─ efficiency (NUMBER, %, 0-100)

Status: ACTIVE
Created: 2024-01-15
Version: 1.2


COMPONENT (Master Definition)
═════════════════════════════════════════════════════════

ID: transmission-001
Name: "Transmission"
Type: "Transmission"
Category: "Drive System"

Attributes:
├─ num_gears (NUMBER, 1-12)
├─ type (ENUM: Manual, Automatic, CVT)
├─ efficiency (NUMBER, %)
├─ oil_capacity (NUMBER, liters)
└─ torque_limit (NUMBER, Nm)

Status: ACTIVE
Created: 2024-01-15
Version: 1.0


COMPONENT (Master Definition)
═════════════════════════════════════════════════════════

ID: battery-001
Name: "Battery"
Type: "Battery"
Category: "Power Storage"

Attributes:
├─ voltage (NUMBER, V: 12, 24, 48)
├─ capacity (NUMBER, Ah)
├─ chemistry (ENUM: Lead-acid, Lithium, Nickel)
├─ weight (NUMBER, kg)
└─ lifecycle (NUMBER, cycles)

Status: ACTIVE
Created: 2024-02-01
Version: 2.0
```

---

## 3. Template-to-Asset Component Flow

```
┌──────────────────────────────────┐
│   ASSET TEMPLATE                 │
│   (e.g., "Vehicle")              │
│                                  │
│ Defines available components:    │
│ • Can have Engine (optional)     │
│ • Can have Transmission (opt)    │
│ • Can have Battery (optional)    │
│ • Must have Wheels (required)    │
└─────────────────┬────────────────┘
                  ↓
        ┌──────────────────────────┐
        │  TEMPLATE_COMPONENT      │
        │  (from COMPONENT registry)│
        │  Specifies:              │
        │  • Which components      │
        │  • Required/optional     │
        │  • Sort order            │
        └──────────┬───────────────┘
                   ↓
┌──────────────────────────────────────────┐
│  NEW ASSET CREATED FROM TEMPLATE         │
│  (Assets inherit component definitions)  │
│                                          │
│  Asset: "My Vehicle"                     │
│  ├─ Template: Vehicle                    │
│  │  └─ Inherit: Engine, Transmission, ✓  │
│  │     Wheels                            │
│  │                                       │
│  └─ MASTER_COMPONENT records:            │
│     ├─ Component: Engine-001             │
│     │  └─ Status: ATTACHED               │
│     │  └─ Instances: 1                   │
│     │                                    │
│     ├─ Component: Transmission-001       │
│     │  └─ Status: ATTACHED               │
│     │  └─ Instances: 1                   │
│     │                                    │
│     └─ Component: Wheels                 │
│        └─ Status: ATTACHED               │
│        └─ Instances: 4                   │
│                                          │
└──────────────────────────────────────────┘
```

---

## 4. File Attachment System

```
FILE ATTACHMENT LIFECYCLE
═════════════════════════════════════════════════════════

┌────────────────────────────────┐
│ USER UPLOADS FILE              │
│ (Asset detail page)            │
│ • Select file (< 500 MB)       │
│ • Auto-detect file type        │
│ • Optional description         │
└──────────────┬─────────────────┘
               ↓
┌──────────────────────────────────┐
│ VALIDATE FILE                    │
├──────────────────────────────────┤
│ • Size check (< 500 MB)          │
│ • File type allowed?             │
│ • Generate file hash (MD5/SHA256)│
│ • Scan for obvious issues        │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ CREATE MASTER_ATTACHMENT RECORD  │
├──────────────────────────────────┤
│ Table: MASTER_ATTACHMENT         │
│ ├─ asset_master_id               │
│ ├─ file_name                     │
│ ├─ file_size                     │
│ ├─ file_hash                     │
│ ├─ content_type (MIME)           │
│ ├─ status: PENDING               │
│ ├─ uploaded_at: NOW              │
│ └─ uploaded_by: user_id          │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ UPLOAD TO AZURE BLOB STORAGE     │
├──────────────────────────────────┤
│ • Create blob container          │
│ • Upload file bytes              │
│ • Set expiration policy          │
│ • Store blob URI in DB           │
│ • Set backup/replication         │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ PUBLISH FILE_UPLOADED EVENT      │
├──────────────────────────────────┤
│ Event: FILE_UPLOADED             │
│ ├─ asset_id                      │
│ ├─ attachment_id                 │
│ ├─ file_name                     │
│ ├─ timestamp                     │
│ └─ user_id                       │
│                                  │
│ → DocumentHandler consumes event │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ DOCUMENTHANDLER PROCESSES        │
│ (Async, background)              │
├──────────────────────────────────┤
│ 1. Retrieve file from blob       │
│ 2. Call Votiro CDR API           │
│ 3. Submit for malware scan       │
│ 4. Poll for result (timeout 5min)│
│ 5. Handle timeout (retry logic)  │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ VOTIRO CDR SCANS FILE            │
├──────────────────────────────────┤
│ Analysis includes:               │
│ • Malware/threat detection       │
│ • Vulnerable code patterns       │
│ • Archive bomb detection         │
│ • Exploit detection              │
│ • Macro/script analysis          │
│                                  │
│ Result:                          │
│ ├─ Status: CLEAN                 │
│ ├─ Threat Score: 0-100           │
│ ├─ Details: {...}                │
│ └─ Timestamp: ISO 8601           │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ UPDATE MASTER_ATTACHMENT         │
├──────────────────────────────────┤
│ • status: CLEAN or QUARANTINE    │
│ • threat_score: (0-100)          │
│ • scan_result: {...}             │
│ • scanned_at: timestamp          │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ PUBLISH FILE_SCANNED EVENT       │
├──────────────────────────────────┤
│ Event: FILE_SCANNED              │
│ ├─ attachment_id                 │
│ ├─ status (CLEAN/QUARANTINE)     │
│ ├─ threat_score                  │
│ ├─ timestamp                     │
│ └─ scan_details                  │
│                                  │
│ → Handlers process (notifications)
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│ SEND NOTIFICATIONS               │
├──────────────────────────────────┤
│ If CLEAN:                        │
│ • UI notification: File safe ✓   │
│ • User can download file         │
│                                  │
│ If QUARANTINE:                   │
│ • UI alert: File quarantined ⚠️  │
│ • Email alert to admin           │
│ • Webhook to security system     │
│ • Block file access              │
└──────────────────────────────────┘

USER SEES RESULT
```

---

## 5. Attachment Data Model

```
MASTER_ATTACHMENT Table
═════════════════════════════════════════════════════════

Column                  Type        Purpose
─────────────────────────────────────────────────────────
id                      UUID        Primary key
asset_master_id         UUID        Which asset?
file_name               VARCHAR     Original filename
file_size               INT         Size in bytes
file_hash               VARCHAR     MD5/SHA256 checksum
content_type            VARCHAR     MIME type (application/pdf)
blob_uri                VARCHAR     Azure Blob path
uploaded_at             TIMESTAMP   When uploaded
uploaded_by             UUID        User who uploaded
status                  VARCHAR     PENDING/CLEAN/QUARANTINE
threat_score            INT         0-100 (from Votiro)
scan_result             JSONB       Detailed scan data
scanned_at              TIMESTAMP   When scan completed
expires_at              TIMESTAMP   Deletion date (retention)
notes                   TEXT        Admin notes
created_at              TIMESTAMP   Record creation
updated_at              TIMESTAMP   Last modification
─────────────────────────────────────────────────────────

Status Values:
├─ PENDING       File uploaded, awaiting scan
├─ CLEAN         Scanned, no threats found
├─ QUARANTINE    Threats detected, quarantined
└─ DELETED       File expired or removed


Retention Policy:
├─ CLEAN files: Keep 5 years
├─ QUARANTINE files: Keep 2 years
└─ PENDING files: Auto-purge after 30 days if not scanned
```

---

## 6. File Security Pipeline

```
SECURITY LAYERS
═════════════════════════════════════════════════════════

Layer 1: UPLOAD VALIDATION
├─ File size limit: 500 MB
├─ Allowed types: .pdf, .docx, .xlsx, .jpg, .png, .zip
├─ File name sanitization (no path traversal)
└─ Check: Is user authorized to attach files?

         ↓ [PASS]

Layer 2: AZURE BLOB STORAGE
├─ Encryption at rest (AES-256)
├─ HTTPS transport encryption
├─ Access Control:
│  └─ Only authenticated users can access
├─ Backup & replication
└─ Audit logging (who accessed, when)

         ↓ [PASS]

Layer 3: VOTIRO CDR SCAN
├─ Malware detection (signatures + heuristics)
├─ Vulnerability patterns
├─ Archive bomb detection
├─ Macro/code analysis
└─ Threat scoring (0-100)

         ↓ [PASS or QUARANTINE]

Layer 4: ACCESS CONTROL
├─ Only users with "View Assets" permission
├─ Asset owner can download CLEAN files
├─ Admin review QUARANTINE files
├─ Audit log: Who downloaded, when
└─ File auto-deletes per retention policy

RESULT:
Multiple security layers ensure uploaded files are safe
before they can be downloaded by users.
```

---

## 7. File Scanning States

```
FILE STATES & TRANSITIONS
═════════════════════════════════════════════════════════

                    ┌──────────────┐
                    │   UPLOADED   │ (User selected file)
                    └──────┬───────┘
                           ↓
                   ┌────────────────┐
                   │    PENDING     │ (Awaiting scan)
                   └──────┬─────────┘
                          ↓
                   [VOTIRO SCAN]
                          ↓
            ┌─────────────┴──────────────┐
            ↓                            ↓
      ┌──────────┐                ┌─────────────┐
      │  CLEAN   │                │ QUARANTINE  │
      │ (Safe)   │                │ (Threat!)   │
      └──────────┘                └─────────────┘
            ↓                            ↓
    [User can download]        [Admin review only]
            ↓                            ↓
            └──────────┬─────────────────┘
                       ↓
            [Retention period ends]
                       ↓
                ┌──────────────┐
                │   DELETED    │ (Purged from system)
                └──────────────┘

Retention Times:
├─ CLEAN: 5 years
├─ QUARANTINE: 2 years
└─ PENDING: 30 days (then auto-delete)
```

---

## 8. Attachment Events

```
ATTACHMENT-RELATED EVENTS
═════════════════════════════════════════════════════════

Event: FILE_UPLOADED
│
├─ When: User uploads file
├─ Triggers: DocumentHandler starts scan
└─ Payload:
   {
     "asset_id": "550e8400-...",
     "attachment_id": "660e8400-...",
     "file_name": "specification.pdf",
     "file_size": 2048000,
     "content_type": "application/pdf",
     "uploaded_by": "user-123",
     "uploaded_at": "2024-05-03T14:30:00Z"
   }


Event: FILE_SCANNING
│
├─ When: DocumentHandler sends to Votiro
├─ Triggers: Polling for result
└─ Payload:
   {
     "attachment_id": "660e8400-...",
     "status": "SCANNING",
     "votiro_job_id": "job-456",
     "started_at": "2024-05-03T14:31:00Z"
   }


Event: FILE_SCANNED
│
├─ When: Votiro returns result
├─ Triggers: Update MASTER_ATTACHMENT, send notifications
└─ Payload:
   {
     "attachment_id": "660e8400-...",
     "status": "CLEAN",
     "threat_score": 0,
     "scan_details": {
       "malware_detected": false,
       "exploits": [],
       "suspicious_macros": false
     },
     "scanned_at": "2024-05-03T14:32:00Z"
   }


Event: FILE_QUARANTINE
│
├─ When: File fails scan (threats found)
├─ Triggers: Email alert, webhook, UI warning
└─ Payload:
   {
     "attachment_id": "660e8400-...",
     "status": "QUARANTINE",
     "threat_score": 85,
     "threats": ["Trojan.Win32.Generic", "Suspicious.Macro"],
     "scanned_at": "2024-05-03T14:32:00Z"
   }
```

---

## 9. Component Attachment to Asset

```
ATTACHING COMPONENTS TO ASSET
═════════════════════════════════════════════════════════

User Action:
[Asset Detail Page]
  → Click "Add Component"
  → Select component type (Engine)
  → Confirm → Event published

Process:
┌──────────────────────────────┐
│ 1. Validate                  │
│    • Asset exists?           │
│    • Component type allowed? │
│    • Permission check?       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Create MASTER_COMPONENT   │
│    • asset_master_id         │
│    • component_id            │
│    • status: ACTIVE          │
│    • attached_at: NOW        │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Publish COMPONENT_ATTACHED│
│    Event → Handlers process  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Component attributes      │
│    inherited by asset        │
│    (power_output, fuel_type) │
└──────────────────────────────┘

Component Attributes Become Asset Attributes:
├─ component.power_output (150 kW)
│  ↓ Creates MASTER_ATTRIBUTE
│  └─ asset.component_power_output = 150
│
├─ component.fuel_type (Diesel)
│  ↓ Creates MASTER_ATTRIBUTE
│  └─ asset.component_fuel_type = Diesel
│
└─ [All inherited, validated against rules]
```

---

## 10. Common Operations

```
USE CASE 1: Upload & Scan Asset Document
──────────────────────────────────────────
1. User opens asset "Equipment-001"
2. Clicks "Add Attachment"
3. Selects specification sheet (specification.pdf)
4. File uploaded → MASTER_ATTACHMENT created (PENDING)
5. DocumentHandler polls Votiro CDR
6. Votiro returns: CLEAN (threat_score=0)
7. Status updated to CLEAN
8. User notified: "File ready to download"
9. User can now download file

Timeline: ~15 seconds (including Votiro scan)


USE CASE 2: Quarantine Suspicious File
────────────────────────────────────────
1. User uploads manual.pdf
2. Votiro scan completes
3. Result: QUARANTINE (threat_score=92, malware detected)
4. Status updated to QUARANTINE
5. Email alert sent to admin:
   "File quarantined: manual.pdf (equipment-001)"
6. Webhook sent to security system
7. File access blocked for regular users
8. Admin can review threat details
9. After 2 years, file auto-deleted per retention

Timeline: ~20 seconds (scan + notifications)


USE CASE 3: Attach Component to Asset
───────────────────────────────────────
1. User opens asset "Vehicle-001"
2. Clicks "Add Component"
3. Selects "Engine" from registry
4. Component attached (Engine-001)
5. System creates MASTER_COMPONENT record
6. Engine attributes inherited by asset:
   ├─ engine.power_output → asset attribute
   ├─ engine.fuel_type → asset attribute
   └─ engine.displacement → asset attribute
7. Engine appears in asset detail
8. Can now fill engine attributes

Timeline: Instant (no external calls)
```

---

## 11. Performance Notes

```
OPTIMIZATION TIPS
═════════════════════════════════════════════════════════

FILE UPLOADS:
├─ Use chunked upload for large files (> 10 MB)
├─ Parallel chunk processing
├─ Resumable uploads (keep hash of uploaded chunks)
└─ Bandwidth: ~5 MB/sec typical for 500 MB file

VOTIRO SCANNING:
├─ Timeout: 5 minutes (reasonable for most files)
├─ Retry on timeout: 3 attempts
├─ Queue scanning if load high (async)
├─ Parallel scans: 10 concurrent (Votiro limit)
└─ Expected time: Small files (~1 MB): 10-20 sec
                   Large files (~100 MB): 60-120 sec

DATABASE:
├─ Index on (asset_master_id) for fast lookup
├─ Index on (status) for filtering
├─ Archive old DELETED records after 5 years
└─ File metadata cached (1 hour TTL)

BLOB STORAGE:
├─ Use blob tiers: Hot (recent) → Cool (old)
├─ GRS replication for critical files
├─ Compression for text files (saves ~40%)
└─ CDN caching for frequent access
```

---

## Quick Reference

| Concept | Definition | Table |
|---------|-----------|-------|
| **Component** | Reusable sub-part (Engine, Battery) | COMPONENT |
| **Master Component** | Component attached to asset | MASTER_COMPONENT |
| **Attachment** | Uploaded file | MASTER_ATTACHMENT |
| **Votiro CDR** | Malware scanning service | (external) |
| **File Status** | PENDING, CLEAN, QUARANTINE, DELETED | status field |
| **Threat Score** | 0-100 risk level (Votiro) | threat_score |

---

## Related Documents

- **Architecture** → [MASTER_VISUAL_GUIDE.md](./MASTER_VISUAL_GUIDE.md)
- **API Reference** → [9_API_ENDPOINTS.md](./9_API_ENDPOINTS.md)
- **Integration** → [10_INTEGRATION_GUIDE.md](./10_INTEGRATION_GUIDE.md)
- **Events** → [6_EVENTS_AUDIT.md](./6_EVENTS_AUDIT.md)

---

**Last Updated:** 2026-05-03 | **Version:** 2.0
