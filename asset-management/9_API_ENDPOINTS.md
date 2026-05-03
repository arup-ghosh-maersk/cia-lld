# 9. API ENDPOINTS — Asset Master System

> **Purpose:** Complete REST API specification with request/response examples

---

## API Overview

```
BASE URL: http://localhost:5000/api/v1
AUTHENTICATION: Bearer token (JWT)
CONTENT-TYPE: application/json
```

---

## Asset Management

### 1. Create Asset

**Endpoint:** `POST /api/v1/assets`

**Description:** Create new asset from template. Triggers ASSET_EVENT_STORE event.

**Request:**
```json
{
  "assetTemplateId": "550e8400-e29b-41d4-a716-446655440000",
  "assetCode": "ASSET001",
  "name": "Motor #123",
  "serialNumber": "SN-2025-001",
  "installationDate": "2025-05-03",
  "components": [
    {
      "componentId": "550e8400-e29b-41d4-a716-446655440100",
      "quantity": 2
    }
  ],
  "attachments": [
    {
      "attachmentId": "550e8400-e29b-41d4-a716-446655440200"
    }
  ]
}
```

**Response (201 Created):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "assetCode": "ASSET001",
  "name": "Motor #123",
  "status": "ACTIVE",
  "createdAt": "2025-05-03T10:30:00Z",
  "eventId": "550e8400-e29b-41d4-a716-446655440010"
}
```

**Errors:**
- `400 Bad Request` — Invalid template or missing required fields
- `409 Conflict` — Asset code already exists
- `500 Internal Server Error` — Database or event bus failure

---

### 2. Update Asset

**Endpoint:** `PUT /api/v1/assets/{assetId}`

**Description:** Update asset details. Triggers ASSET_EVENT_STORE event.

**Request:**
```json
{
  "name": "Motor #123 (Updated)",
  "serialNumber": "SN-2025-001-REV2",
  "status": "INACTIVE"
}
```

**Response (200 OK):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "assetCode": "ASSET001",
  "name": "Motor #123 (Updated)",
  "status": "INACTIVE",
  "updatedAt": "2025-05-03T11:00:00Z",
  "eventId": "550e8400-e29b-41d4-a716-446655440011"
}
```

---

### 3. Get Asset

**Endpoint:** `GET /api/v1/assets/{assetId}`

**Description:** Retrieve asset details with all attributes, components, attachments.

**Response (200 OK):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "assetCode": "ASSET001",
  "name": "Motor #123",
  "serialNumber": "SN-2025-001",
  "status": "ACTIVE",
  "installationDate": "2025-05-03",
  "attributes": [
    {
      "attributeId": "550e8400-e29b-41d4-a716-446655440301",
      "key": "rated_capacity",
      "displayName": "Rated Capacity",
      "value": "250",
      "unit": "kW",
      "validationStatus": "VALID"
    }
  ],
  "components": [
    {
      "componentId": "550e8400-e29b-41d4-a716-446655440100",
      "name": "Bearing Assembly",
      "quantity": 2,
      "status": "ACTIVE"
    }
  ],
  "attachments": [
    {
      "attachmentId": "550e8400-e29b-41d4-a716-446655440200",
      "fileName": "Manual.pdf",
      "category": "MANUAL",
      "scanStatus": "CLEAN",
      "uploadedAt": "2025-05-03T10:30:00Z"
    }
  ],
  "createdAt": "2025-05-03T10:30:00Z",
  "updatedAt": "2025-05-03T11:00:00Z"
}
```

---

### 4. Get Audit Trail

**Endpoint:** `GET /api/v1/assets/{assetId}/audit`

**Query Parameters:**
- `page` (default: 1)
- `pageSize` (default: 50, max: 500)
- `eventType` (optional: 1=Create, 2=Update, 3=Delete)

**Response (200 OK):**
```json
{
  "assetId": "550e8400-e29b-41d4-a716-446655440000",
  "total": 5,
  "page": 1,
  "pageSize": 50,
  "items": [
    {
      "notificationId": "550e8400-e29b-41d4-a716-446655440020",
      "eventId": "550e8400-e29b-41d4-a716-446655440011",
      "eventType": 2,
      "auditMessage": "Updated Asset ASSET001, Modified Fields: name, status",
      "detail": { /* full event details */ },
      "createdAt": "2025-05-03T11:00:00Z",
      "createdBy": "user@company.com"
    },
    {
      "notificationId": "550e8400-e29b-41d4-a716-446655440021",
      "eventId": "550e8400-e29b-41d4-a716-446655440010",
      "eventType": 1,
      "auditMessage": "Created Asset ASSET001, Added Components (2), Attachments (1)",
      "detail": { /* full event details */ },
      "createdAt": "2025-05-03T10:30:00Z",
      "createdBy": "user@company.com"
    }
  ]
}
```

---

### 5. Delete Asset

**Endpoint:** `DELETE /api/v1/assets/{assetId}`

**Description:** Soft delete asset (status → DECOMMISSIONED, triggers event).

**Response (204 No Content)**

---

## Attribute Management

### 6. Save Attribute Value (with Validation)

**Endpoint:** `PUT /api/v1/assets/{assetId}/attribute-values`

**Description:** Save/update attribute value with automatic validation against rules.

**Request:**
```json
{
  "attributeDefId": "550e8400-e29b-41d4-a716-446655440301",
  "value": "250"
}
```

**Response (200 OK):**
```json
{
  "attributeId": "550e8400-e29b-41d4-a716-446655440401",
  "attributeDefId": "550e8400-e29b-41d4-a716-446655440301",
  "key": "rated_capacity",
  "value": "250",
  "validationStatus": "VALID",
  "validationErrors": []
}
```

**Response (400 Bad Request) — Validation Failed:**
```json
{
  "attributeId": "550e8400-e29b-41d4-a716-446655440401",
  "validationStatus": "INVALID",
  "validationErrors": [
    {
      "rule": "MAX_VALUE",
      "message": "Cannot exceed 500 kW",
      "severity": "ERROR"
    }
  ]
}
```

---

### 7. Get Attribute Definitions

**Endpoint:** `GET /api/v1/assets/{assetId}/attribute-defs`

**Query Parameters:**
- `scope` (optional: TEMPLATE, ASSET, or ALL)
- `groupName` (optional: filter by group)

**Response (200 OK):**
```json
{
  "assetId": "550e8400-e29b-41d4-a716-446655440000",
  "total": 3,
  "items": [
    {
      "attributeDefId": "550e8400-e29b-41d4-a716-446655440301",
      "attributeKey": "rated_capacity",
      "displayName": "Rated Capacity",
      "dataType": "NUMBER",
      "defaultValue": "100",
      "unit": "kW",
      "groupName": "Electrical",
      "sortOrder": 1,
      "isRequired": true,
      "scope": "TEMPLATE",
      "source": "FROM_TEMPLATE",
      "validationRules": [
        {
          "type": "REQUIRED",
          "severity": "ERROR",
          "active": true
        },
        {
          "type": "MIN_VALUE",
          "value": "10",
          "severity": "ERROR",
          "active": true
        }
      ]
    }
  ]
}
```

---

## Document Management

### 8. Upload Document (with Votiro Scan)

**Endpoint:** `POST /api/v1/assets/{assetId}/attachments`

**Description:** Upload file → quarantine → submit Votiro CDR scan (async)

**Request:**
```
Content-Type: multipart/form-data

Parameters:
  - file: (required) Binary file stream (PDF, image, Office)
  - category: (required) MANUAL|DRAWING|CERTIFICATE|IMAGE|OTHER
  - description: (optional) "Operating Manual for Motor #123"
```

**Response (202 Accepted):**
```json
{
  "attachmentId": "550e8400-e29b-41d4-a716-446655440200",
  "fileName": "Manual.pdf",
  "category": "MANUAL",
  "fileSize": 2097152,
  "scanStatus": "PENDING",
  "uploadedAt": "2025-05-03T14:30:00Z",
  "uploadedBy": "user@company.com",
  "message": "File uploaded successfully. Scanning in progress..."
}
```

**Errors:**
- `400 Bad Request` — Missing file or invalid category
- `413 Payload Too Large` — File exceeds 50 MB
- `415 Unsupported Media Type` — File type not allowed (.exe, .zip)
- `500 Internal Server Error` — Storage or scan submission failure

**cURL Example:**
```bash
curl -X POST http://localhost:5000/api/v1/assets/550e8400-e29b-41d4-a716-446655440000/attachments \
  -H "Authorization: Bearer {token}" \
  -F "file=@Manual.pdf" \
  -F "category=MANUAL" \
  -F "description=Operating Manual"
```

---

### 9. Check Scan Status

**Endpoint:** `GET /api/v1/attachments/{attachmentId}/scan-status`

**Description:** Poll for Votiro scan result.

**Response (200 OK) — CLEAN:**
```json
{
  "attachmentId": "550e8400-e29b-41d4-a716-446655440200",
  "fileName": "Manual.pdf",
  "scanStatus": "CLEAN",
  "scanEngine": "VOTIRO",
  "fileSize": 2097152,
  "scannedAt": "2025-05-03T14:31:30Z",
  "downloadUrl": "/api/v1/attachments/550e8400-e29b-41d4-a716-446655440200/download",
  "message": "Document scanned and cleared for access"
}
```

**Response (200 OK) — INFECTED:**
```json
{
  "attachmentId": "550e8400-e29b-41d4-a716-446655440201",
  "fileName": "Report.xlsx",
  "scanStatus": "INFECTED",
  "scanEngine": "VOTIRO",
  "scannedAt": "2025-05-03T14:32:00Z",
  "threatName": "Trojan.Generic.XLS",
  "message": "File blocked due to security threat. Contact IT Support."
}
```

**Response (200 OK) — PENDING:**
```json
{
  "attachmentId": "550e8400-e29b-41d4-a716-446655440202",
  "fileName": "Document.pdf",
  "scanStatus": "PENDING",
  "scanEngine": "VOTIRO",
  "message": "Scan in progress. Please check again in 5 seconds."
}
```

---

### 10. Download Document

**Endpoint:** `GET /api/v1/attachments/{attachmentId}/download`

**Description:** Download scanned document (only if scan_status=CLEAN).

**Response (200 OK):**
```
Content-Type: application/pdf (or detected MIME type)
Content-Disposition: attachment; filename="Manual.pdf"
Content-Length: 2097152

[binary file content]
```

**Errors:**
- `404 Not Found` — Attachment not found
- `410 Gone` — Attachment was deleted
- `423 Locked` — Scan still in progress (return after 2 seconds)
- `451 Unavailable For Legal Reasons` — File blocked due to threat

---

### 11. List Attachments for Asset

**Endpoint:** `GET /api/v1/assets/{assetId}/attachments`

**Query Parameters:**
- `category` (optional: filter by category)
- `scanStatus` (optional: PENDING|CLEAN|INFECTED|FAILED)

**Response (200 OK):**
```json
{
  "assetId": "550e8400-e29b-41d4-a716-446655440000",
  "total": 2,
  "items": [
    {
      "attachmentId": "550e8400-e29b-41d4-a716-446655440200",
      "fileName": "Manual.pdf",
      "category": "MANUAL",
      "fileSize": 2097152,
      "scanStatus": "CLEAN",
      "uploadedAt": "2025-05-03T14:30:00Z",
      "downloadUrl": "/api/v1/attachments/550e8400-e29b-41d4-a716-446655440200/download"
    },
    {
      "attachmentId": "550e8400-e29b-41d4-a716-446655440201",
      "fileName": "Report.xlsx",
      "category": "DRAWING",
      "fileSize": 1048576,
      "scanStatus": "INFECTED",
      "uploadedAt": "2025-05-03T14:35:00Z",
      "message": "File blocked - security threat detected"
    }
  ]
}
```

---

### 12. Delete Attachment

**Endpoint:** `DELETE /api/v1/attachments/{attachmentId}`

**Response (204 No Content)**

---

## Error Response Format

All errors follow consistent format:

```json
{
  "errorCode": "VALIDATION_FAILED",
  "message": "User-friendly error message",
  "statusCode": 400,
  "details": {
    "field": "assetCode",
    "issue": "Already exists"
  },
  "timestamp": "2025-05-03T14:30:00Z",
  "requestId": "550e8400-e29b-41d4-a716-446655440999"
}
```

---

## Common Error Codes

| Code | HTTP | Meaning | Solution |
|------|------|---------|----------|
| VALIDATION_FAILED | 400 | Input validation error | Check request format and values |
| DUPLICATE_ASSET_CODE | 409 | Asset code already exists | Use unique code |
| ASSET_NOT_FOUND | 404 | Asset doesn't exist | Verify asset ID |
| TEMPLATE_NOT_FOUND | 404 | Template doesn't exist | Verify template ID |
| FILE_TOO_LARGE | 413 | File exceeds 50 MB | Compress or split file |
| UNSUPPORTED_FILE_TYPE | 415 | File type not allowed | Use PDF/image/Office format |
| SCAN_IN_PROGRESS | 423 | File scan still running | Retry after 2–5 seconds |
| FILE_INFECTED | 451 | File blocked by security | Delete and upload clean file |
| INTERNAL_ERROR | 500 | Server error | Check logs, contact support |

---

## Rate Limiting

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1620059160
```

**Limits:**
- 1000 requests per hour per user
- 10 concurrent uploads per user
- File size: 50 MB max

---

## Pagination

**Query Parameters:**
- `page` (default: 1, min: 1)
- `pageSize` (default: 50, min: 10, max: 500)

**Response Header:**
```json
{
  "total": 256,
  "page": 1,
  "pageSize": 50,
  "totalPages": 6,
  "items": [...]
}
```

---

## Next Steps

1. **Implementation:** See [8_HANDLERS_IMPLEMENTATION.md](./8_HANDLERS_IMPLEMENTATION.md)
2. **Integration:** See [10_INTEGRATION_GUIDE.md](./10_INTEGRATION_GUIDE.md)
3. **Testing:** See [HANDLERS_TESTING_GUIDE.md](./HANDLERS_TESTING_GUIDE.md)

---

**File:** 9_API_ENDPOINTS.md | **Lines:** ~300
