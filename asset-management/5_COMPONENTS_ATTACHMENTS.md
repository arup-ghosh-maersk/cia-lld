# Components & File Attachments

> **Date:** May 3, 2026 | **Version:** 1.0 | **Status:** Production Ready (Phase 2)

## Table of Contents
1. [Overview](#overview)
2. [Data Model](#data-model)
3. [Components System](#components-system)
4. [Attachments & Documents](#attachments--documents)
5. [Votiro CDR Integration](#votiro-cdr-integration)
6. [File Security](#file-security)
7. [Operations](#operations)
8. [Code Examples](#code-examples)

---

## Overview

The **Components & Attachments** subsystem manages:
- **Components:** Reusable equipment parts (motors, pumps, etc.) linked to assets
- **Attachments:** Documents (PDFs, images) associated with assets
- **CDR Scanning:** Content Disarm & Reconstruct with Votiro for malware protection

### Business Context
- Assets can have many components (e.g., motor + pump + bearings)
- Users upload technical documents, manuals, certificates
- All files are scanned for malware before storage/download
- Components and attachments are tracked in the audit log

### Key Features
- ✅ **Component Registry:** Reusable component definitions
- ✅ **Bulk Association:** Add components to assets via templates
- ✅ **File Upload:** Multi-file support with progress tracking
- ✅ **CDR Scanning:** Automatic malware detection and remediation
- ✅ **Scan Status:** Real-time status and download availability
- ✅ **Versioning:** Track multiple versions of documents

---

## Data Model

### COMPONENT Table
Component definition registry.

```sql
CREATE TABLE COMPONENT (
    id UUID PRIMARY KEY,
    component_code VARCHAR(100) UNIQUE NOT NULL,
    component_name VARCHAR(255) NOT NULL,
    description TEXT,
    component_type VARCHAR(100),        -- MOTOR, PUMP, BEARING, VALVE, etc.
    manufacturer VARCHAR(255),
    model_number VARCHAR(100),
    specs JSONB,                        -- Power, voltage, efficiency, etc.
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_component_code ON COMPONENT(component_code);
CREATE INDEX idx_component_type ON COMPONENT(component_type);
```

### TEMPLATE_COMPONENT Table
Components assigned to templates.

```sql
CREATE TABLE TEMPLATE_COMPONENT (
    id UUID PRIMARY KEY,
    asset_template_id UUID NOT NULL,
    component_id UUID NOT NULL,
    quantity INT DEFAULT 1,
    is_mandatory BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (asset_template_id) REFERENCES ASSET_TEMPLATE(id),
    FOREIGN KEY (component_id) REFERENCES COMPONENT(id),
    UNIQUE (asset_template_id, component_id)
);

CREATE INDEX idx_asset_template_id ON TEMPLATE_COMPONENT(asset_template_id);
CREATE INDEX idx_component_id ON TEMPLATE_COMPONENT(component_id);
```

### MASTER_COMPONENT Table
Component instances on assets.

```sql
CREATE TABLE MASTER_COMPONENT (
    id UUID PRIMARY KEY,
    asset_master_id UUID NOT NULL,
    component_id UUID NOT NULL,
    template_component_id UUID,
    serial_number VARCHAR(255),        -- Component's own serial number
    quantity INT DEFAULT 1,
    installation_date DATE,
    last_maintenance_date DATE,
    is_inherited BOOLEAN DEFAULT FALSE,
    source VARCHAR(50),                -- TEMPLATE, USER_INPUT, SYSTEM
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (asset_master_id) REFERENCES ASSET_MASTER(id),
    FOREIGN KEY (component_id) REFERENCES COMPONENT(id),
    FOREIGN KEY (template_component_id) REFERENCES TEMPLATE_COMPONENT(id)
);

CREATE INDEX idx_asset_master_id ON MASTER_COMPONENT(asset_master_id);
CREATE INDEX idx_component_id ON MASTER_COMPONENT(component_id);
```

### ATTACHMENT Table
File metadata and CDR status.

```sql
CREATE TABLE ATTACHMENT (
    id UUID PRIMARY KEY,
    asset_master_id UUID NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    mime_type VARCHAR(100),            -- application/pdf, image/png, etc.
    file_path VARCHAR(1024),           -- Azure Blob storage path
    file_hash VARCHAR(64),             -- SHA-256 for integrity
    upload_status VARCHAR(50),         -- PENDING, UPLOADED, FAILED
    uploaded_at TIMESTAMP,
    uploaded_by UUID,                  -- User ID
    document_type VARCHAR(100),        -- MANUAL, CERTIFICATE, DRAWING, etc.
    document_category VARCHAR(100),    -- TECHNICAL, LEGAL, MAINTENANCE
    is_confidential BOOLEAN DEFAULT FALSE,
    retention_days INT,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (asset_master_id) REFERENCES ASSET_MASTER(id),
    UNIQUE (asset_master_id, file_name)
);

CREATE INDEX idx_asset_master_id ON ATTACHMENT(asset_master_id);
CREATE INDEX idx_upload_status ON ATTACHMENT(upload_status);
CREATE INDEX idx_created_at ON ATTACHMENT(created_at);
```

### DOCUMENT_SCAN Table
CDR scan results and status.

```sql
CREATE TABLE DOCUMENT_SCAN (
    id UUID PRIMARY KEY,
    attachment_id UUID NOT NULL,
    scan_status VARCHAR(50) NOT NULL, -- PENDING, SCANNING, CLEAN, INFECTED, FAILED
    scan_started_at TIMESTAMP,
    scan_completed_at TIMESTAMP,
    risk_level VARCHAR(50),            -- CLEAN, LOW, MEDIUM, HIGH
    threats_found TEXT,                -- JSON array of detected threats
    remediation_actions TEXT,          -- Actions taken (disarm, remove, etc.)
    clean_file_path VARCHAR(1024),     -- Path to remediated file
    clean_file_size_bytes BIGINT,
    download_allowed BOOLEAN DEFAULT FALSE,
    download_reason VARCHAR(255),      -- Why file can/cannot be downloaded
    scan_engine_version VARCHAR(100),
    votiro_job_id VARCHAR(255),        -- Votiro API reference
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (attachment_id) REFERENCES ATTACHMENT(id),
    UNIQUE (attachment_id)
);

CREATE INDEX idx_attachment_id ON DOCUMENT_SCAN(attachment_id);
CREATE INDEX idx_scan_status ON DOCUMENT_SCAN(scan_status);
CREATE INDEX idx_risk_level ON DOCUMENT_SCAN(risk_level);
```

---

## Components System

### Component Lifecycle

```
┌─────────────────────────────────────┐
│ COMPONENT REGISTRY                  │
│ (Global definition)                 │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ TEMPLATE_COMPONENT                  │
│ (Assign to template)                │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ MASTER_COMPONENT                    │
│ (Instance on asset)                 │
│                                     │
│ Can override: serial, dates,        │
│ quantity, notes                     │
└─────────────────────────────────────┘
```

### Component Definition Example

```json
{
  "id": "comp-123",
  "component_code": "MTR-IE3-11KW",
  "component_name": "IE3 Electric Motor 11kW",
  "component_type": "MOTOR",
  "manufacturer": "Siemens",
  "model_number": "1LA7 163-4AA61-Z A55",
  "description": "Three-phase asynchronous motor, IE3 efficiency class",
  "specs": {
    "power_kw": 11,
    "voltage_v": 400,
    "frequency_hz": 50,
    "poles": 4,
    "rated_speed_rpm": 1500,
    "efficiency_class": "IE3",
    "protection_class": "IP55",
    "frame_size": "160M"
  },
  "is_active": true
}
```

### Query: Get All Components for Asset

```sql
SELECT 
  c.component_code,
  c.component_name,
  c.component_type,
  c.manufacturer,
  mc.serial_number,
  mc.quantity,
  mc.installation_date,
  mc.last_maintenance_date,
  mc.is_inherited,
  c.specs
FROM MASTER_COMPONENT mc
JOIN COMPONENT c ON mc.component_id = c.id
WHERE mc.asset_master_id = $1
ORDER BY c.component_type, c.component_name;
```

---

## Attachments & Documents

### File Upload Workflow

```
┌──────────────────┐
│ User selects     │
│ file to upload   │
└────────┬─────────┘
         ↓
┌──────────────────────────────────────┐
│ Validate:                            │
│ - File size (max 100MB)              │
│ - MIME type (PDF, PNG, DOCX, etc.)  │
│ - File name (no special chars)       │
└────────┬─────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│ Create ATTACHMENT record             │
│ upload_status = PENDING              │
│ Calculate SHA-256 hash               │
└────────┬─────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│ Upload to Azure Blob Storage         │
└────────┬─────────────────────────────┘
         ↓ (on success)
┌──────────────────────────────────────┐
│ Update ATTACHMENT:                   │
│ upload_status = UPLOADED             │
│ uploaded_at = now()                  │
└────────┬─────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│ Send to Votiro CDR for scanning      │
│ Create DOCUMENT_SCAN record          │
│ scan_status = PENDING                │
└────────┬─────────────────────────────┘
         ↓ (async webhook from Votiro)
┌──────────────────────────────────────┐
│ Receive scan results                 │
│ Update DOCUMENT_SCAN:                │
│ - scan_status = CLEAN/INFECTED/etc. │
│ - risk_level = assessment            │
│ - download_allowed = decision        │
└──────────────────────────────────────┘
```

### File Size & Type Limits

| Category | Limit | Allowed Types |
|----------|-------|---------------|
| **Single File** | 100 MB | PDF, PNG, JPEG, DOCX, XLSX, TXT |
| **Total per Asset** | 1 GB | (same types) |
| **Retention** | 7 years | (configurable per document type) |

### Document Types

```
┌─ TECHNICAL
│  ├─ MANUAL (User guides, operation manuals)
│  ├─ DRAWING (Blueprints, schematics)
│  ├─ SPECIFICATION (Data sheets, tech specs)
│  └─ MAINTENANCE_LOG (Service records)
│
├─ LEGAL
│  ├─ WARRANTY (Warranty certificates)
│  ├─ CERTIFICATE (Conformity, compliance)
│  └─ LICENSE (Software licenses)
│
└─ OTHER
   ├─ INSPECTION_REPORT
   ├─ CALIBRATION_REPORT
   └─ CUSTOM_DOCUMENT
```

---

## Votiro CDR Integration

### What is CDR?
**Content Disarm & Reconstruct** removes malicious code from documents while preserving content.

### Supported Formats
- ✅ PDF, Microsoft Office (DOCX, XLSX, PPTX)
- ✅ Images (PNG, JPEG, GIF, BMP)
- ✅ Archives (ZIP, RAR)
- ✅ Plain text files

### Scan States

```
PENDING   → Waiting in queue
   ↓
SCANNING  → Votiro processing
   ↓
CLEAN     → No threats, file safe
   ↓ OR ↓
INFECTED  → Threats found & removed (if remediation possible)
   ↓ OR ↓
FAILED    → Scan error, manual review needed
```

### Risk Levels

| Risk | Status | Action | Remediation |
|------|--------|--------|-------------|
| **CLEAN** | ✅ Safe | Allow download | No action |
| **LOW** | ⚠️ Minor issues | Allow download | Cosmetic removal |
| **MEDIUM** | ⚠️ Suspicious | Allow (warning) | Content extraction |
| **HIGH** | ❌ Dangerous | Block download | Request review |

### Votiro Configuration (C#)

```csharp
public class VotiroSettings
{
    public string ApiKey { get; set; }
    public string ApiUrl { get; set; } = "https://api.votiro.com/v1";
    public TimeSpan ScanTimeout { get; set; } = TimeSpan.FromMinutes(10);
    public TimeSpan WebhookTimeout { get; set; } = TimeSpan.FromMinutes(30);
    public string WebhookCallbackUrl { get; set; }
}

public interface IVotiroService
{
    Task<string> SendForScanAsync(Stream fileStream, string fileName);
    Task<ScanResult> GetScanResultAsync(string votiroJobId);
    Task<Stream> DownloadCleanFileAsync(string votiroJobId);
}
```

### Votiro API Integration (C#)

```csharp
public class VotiroService : IVotiroService
{
    private readonly HttpClient _client;
    private readonly ILogger<VotiroService> _logger;
    
    public async Task<string> SendForScanAsync(Stream fileStream, string fileName)
    {
        var content = new MultipartFormDataContent();
        content.Add(new StreamContent(fileStream), "file", fileName);
        
        var response = await _client.PostAsync(
            "/scan", content);
        
        response.EnsureSuccessStatusCode();
        
        var result = await response.Content.ReadAsAsync<VotiroSubmitResponse>();
        _logger.LogInformation($"File {fileName} sent to Votiro: {result.JobId}");
        
        return result.JobId;
    }
    
    public async Task<ScanResult> GetScanResultAsync(string votiroJobId)
    {
        var response = await _client.GetAsync($"/status/{votiroJobId}");
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadAsAsync<ScanResult>();
    }
}
```

### Webhook Handler (Scan Complete)

```csharp
[ApiController]
[Route("api/v1/webhooks")]
public class VotiroWebhookController : ControllerBase
{
    private readonly IDocumentScanService _scanService;
    
    [HttpPost("votiro/scan-complete")]
    public async Task<IActionResult> OnScanComplete(
        [FromBody] VotiroWebhookPayload payload)
    {
        _logger.LogInformation(
            $"Scan complete webhook: JobId={payload.JobId}, Status={payload.Status}");
        
        var scanResult = new ScanResult
        {
            VotiroJobId = payload.JobId,
            ScanStatus = payload.Status,  // CLEAN, INFECTED, etc.
            RiskLevel = payload.RiskLevel,
            ThreatsFound = payload.Threats,
            CleanFileSize = payload.CleanFileSize,
            CompletedAt = DateTime.UtcNow
        };
        
        await _scanService.UpdateScanResultAsync(scanResult);
        return Ok();
    }
}
```

---

## File Security

### Security Controls

#### 1. Upload Validation
```csharp
public class FileValidationService
{
    private const long MaxFileSize = 100 * 1024 * 1024; // 100 MB
    private static readonly string[] AllowedMimeTypes = 
    {
        "application/pdf",
        "image/png", "image/jpeg",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    };
    
    public async Task<ValidationResult> ValidateFileAsync(IFormFile file)
    {
        // Check size
        if (file.Length > MaxFileSize)
            return ValidationResult.Failure($"File exceeds {MaxFileSize / 1024 / 1024}MB");
        
        // Check MIME type
        if (!AllowedMimeTypes.Contains(file.ContentType))
            return ValidationResult.Failure($"File type {file.ContentType} not allowed");
        
        // Check for suspicious signatures
        var header = new byte[4];
        await file.OpenReadStream().ReadAsync(header, 0, 4);
        
        return IsValidFileSignature(header)
            ? ValidationResult.Success()
            : ValidationResult.Failure("File signature invalid or corrupted");
    }
}
```

#### 2. Encrypted Storage
```
Azure Blob Storage Configuration:
- All files encrypted at rest (256-bit AES)
- Encrypted in transit (TLS 1.2+)
- Blobs stored in private containers
- No public URLs generated
- Access via SAS tokens (time-limited, single-use)
```

#### 3. Access Control
```sql
-- Only asset owners/admins can download
CREATE TABLE ATTACHMENT_ACCESS (
    id UUID PRIMARY KEY,
    attachment_id UUID NOT NULL,
    user_id UUID NOT NULL,
    access_granted_at TIMESTAMP,
    accessed_at TIMESTAMP,
    FOREIGN KEY (attachment_id) REFERENCES ATTACHMENT(id),
    UNIQUE (attachment_id, user_id)
);
```

#### 4. Audit Logging
```
Every file access logged:
- Who downloaded
- When (timestamp)
- From where (IP address)
- User agent
- Duration of access
```

---

## Operations

### Query: Attachments by Asset
```sql
SELECT 
  a.id, a.file_name, a.file_size_bytes,
  a.document_type, a.document_category,
  ds.scan_status, ds.risk_level,
  ds.download_allowed,
  a.uploaded_at
FROM ATTACHMENT a
LEFT JOIN DOCUMENT_SCAN ds ON a.id = ds.attachment_id
WHERE a.asset_master_id = $1
ORDER BY a.created_at DESC;
```

### Query: Pending Scans
```sql
SELECT 
  a.id, a.file_name,
  ds.scan_status, ds.scan_started_at,
  EXTRACT(EPOCH FROM (NOW() - ds.scan_started_at)) as scan_duration_seconds
FROM DOCUMENT_SCAN ds
JOIN ATTACHMENT a ON ds.attachment_id = a.id
WHERE ds.scan_status IN ('PENDING', 'SCANNING')
ORDER BY ds.scan_started_at ASC;
```

---

## Code Examples

### C# — Upload File and Initiate Scan

```csharp
[HttpPost("api/v1/assets/{assetId}/attachments")]
public async Task<IActionResult> UploadAttachment(
    Guid assetId, IFormFile file)
{
    // Validate
    var validation = await _fileValidator.ValidateFileAsync(file);
    if (!validation.IsValid)
        return BadRequest(validation.Error);
    
    // Create attachment record
    var attachment = new Attachment
    {
        Id = Guid.NewGuid(),
        AssetMasterId = assetId,
        FileName = file.FileName,
        FileSizeBytes = file.Length,
        MimeType = file.ContentType,
        FileHash = await CalculateSha256Async(file),
        UploadStatus = "PENDING"
    };
    
    await _repo.CreateAttachmentAsync(attachment);
    
    // Upload to Azure Blob
    var blobPath = await _storageService.UploadAsync(
        file, assetId, attachment.Id);
    
    attachment.FilePath = blobPath;
    attachment.UploadStatus = "UPLOADED";
    attachment.UploadedAt = DateTime.UtcNow;
    await _repo.UpdateAttachmentAsync(attachment);
    
    // Send to Votiro
    var votiroJobId = await _votiroService.SendForScanAsync(
        file.OpenReadStream(), file.FileName);
    
    var scan = new DocumentScan
    {
        Id = Guid.NewGuid(),
        AttachmentId = attachment.Id,
        ScanStatus = "PENDING",
        VotiroJobId = votiroJobId
    };
    
    await _repo.CreateScanAsync(scan);
    
    return Accepted(new { attachment.Id, votiroJobId });
}
```

### C# — Download File with Scan Check

```csharp
[HttpGet("api/v1/attachments/{attachmentId}/download")]
public async Task<IActionResult> DownloadAttachment(Guid attachmentId)
{
    var attachment = await _repo.GetAttachmentAsync(attachmentId);
    if (attachment == null)
        return NotFound();
    
    var scan = await _repo.GetScanAsync(attachment.Id);
    if (scan == null || scan.ScanStatus != "CLEAN")
    {
        if (scan?.ScanStatus == "PENDING" || scan?.ScanStatus == "SCANNING")
            return Accepted(new { status = "SCANNING", scan.ScanStartedAt });
        
        if (scan?.RiskLevel == "HIGH")
            return Forbid("File blocked due to security scan results");
        
        return BadRequest("File scan incomplete");
    }
    
    // Log access
    await _auditService.LogFileAccessAsync(
        attachmentId, User.GetId(), HttpContext.Connection.RemoteIpAddress);
    
    // Generate SAS token
    var sasUri = await _storageService.GenerateDownloadUrlAsync(
        attachment.FilePath, TimeSpan.FromMinutes(15));
    
    return Ok(new { downloadUrl = sasUri, fileName = attachment.FileName });
}
```

---

## Related Documents
- [1_ARCHITECTURE.md](./1_ARCHITECTURE.md) — Data model overview
- [7_HANDLERS.md](./7_HANDLERS.md) — DocumentHandler for CDR processing
- [9_API_ENDPOINTS.md](./9_API_ENDPOINTS.md) — Upload/download endpoints
- [GLOSSARY.md](./GLOSSARY.md) — File & security terms

---

**Last Updated:** May 3, 2026 | **Next Review:** Phase 2 completion
