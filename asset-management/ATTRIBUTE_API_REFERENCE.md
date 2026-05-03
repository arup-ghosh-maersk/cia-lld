# Attribute Management API — Reference Card

> **Date:** May 3, 2026  
> **Version:** 1.0  
> **Purpose:** Quick API reference for developers  
> **Phases:** Phase 1 (Current) + Phase 2 (Future)

---

## Asset Attribute Endpoints (Phase 1 — Current)

### 1. GET All Attributes (Template-Inherited + Custom)

**Endpoint:** `GET /api/v1/assets/{assetId}/attribute-defs`

**Query Parameters:**
```
scope=TEMPLATE        (optional) - Filter to inherited only
scope=ASSET          (optional) - Filter to custom only
sort=sort_order      (optional) - Sort key
```

**Response: 200 OK**
```json
[
  {
    "id": "DEF-001",
    "attribute_key": "rated_capacity",
    "display_name": "Rated Capacity",
    "data_type": "NUMBER",
    "unit": "kW",
    "scope": "TEMPLATE",
    "owner_id": "TEMPLATE-MOTOR-ID",
    "is_mandatory": true,
    "default_value": "300",
    "sort_order": 1,
    "validation_rules": [
      { "rule_type": "REQUIRED", "severity": "ERROR" },
      { "rule_type": "MIN_VALUE", "rule_value": "10" },
      { "rule_type": "MAX_VALUE", "rule_value": "500" }
    ]
  },
  {
    "id": "DEF-002",
    "attribute_key": "custom_mount_height",
    "display_name": "Custom Mount Height",
    "scope": "ASSET",
    "owner_id": "ASSET-123",
    "source": "NEW",
    "data_type": "NUMBER",
    "unit": "mm"
  }
]
```

---

### 2. POST Create Custom Attribute (Asset-Only)

**Endpoint:** `POST /api/v1/assets/{assetId}/attribute-defs`

**Request:**
```json
{
  "display_name": "Custom Field Name",
  "data_type": "STRING|NUMBER|DATE|BOOLEAN|ENUM|JSON",
  "unit": "optional - e.g., kW, kg, mm",
  "group_name": "optional - UI section",
  "is_required": true,
  "default_value": "optional",
  "valid_values": ["required for ENUM type"],
  "sort_order": 10,
  "validation_rules": [
    {
      "rule_type": "MIN_VALUE|MAX_VALUE|MIN_LENGTH|MAX_LENGTH|REGEX|ENUM|DATE_RANGE|REQUIRED",
      "rule_value": "optional - rule parameter",
      "error_message": "User-friendly error text",
      "severity": "ERROR|WARNING",
      "is_active": true
    }
  ]
}
```

**Response: 201 Created**
```json
{
  "id": "DEF-NEW-001",
  "owner_id": "ASSET-123",
  "scope": "ASSET",
  "source": "NEW",
  "attribute_key": "custom_field_name",
  "display_name": "Custom Field Name",
  "data_type": "STRING"
}
```

**Error: 400 Bad Request**
```json
{
  "error": "Validation failed",
  "details": ["display_name is required", "data_type must be a valid type"]
}
```

---

### 3. PUT Update Attribute & Rules (Asset-Custom Only)

**Endpoint:** `PUT /api/v1/assets/{assetId}/attribute-defs/{defId}`

**Request:**
```json
{
  "display_name": "Updated Name",
  "unit": "new unit",
  "validation_rules": [
    {
      "rule_type": "MIN_VALUE",
      "rule_value": "5",
      "error_message": "Must be at least 5",
      "severity": "ERROR",
      "is_active": true
    }
  ]
}
```

**Response: 200 OK**
```json
{
  "id": "DEF-002",
  "display_name": "Updated Name",
  "validation_rules": [...]
}
```

**Error: 403 Forbidden** (if trying to update TEMPLATE-scoped attribute)
```json
{
  "error": "Cannot update template-scoped attributes from asset endpoint",
  "hint": "Use /api/v1/templates/{templateId}/attribute-defs in Phase 2"
}
```

---

### 4. POST Add Rule to Attribute

**Endpoint:** `POST /api/v1/assets/{assetId}/attribute-defs/{defId}/rules`

**Request:**
```json
{
  "rule_type": "REGEX",
  "rule_value": "^[A-Z]{2}[0-9]{4}$",
  "error_message": "Must be 2 uppercase letters + 4 digits",
  "severity": "ERROR",
  "is_active": true
}
```

**Response: 201 Created**
```json
{
  "rule_type": "REGEX",
  "rule_value": "^[A-Z]{2}[0-9]{4}$",
  "error_message": "Must be 2 uppercase letters + 4 digits",
  "severity": "ERROR",
  "is_active": true,
  "evaluation_order": 3
}
```

---

### 5. PATCH Toggle Rule Active/Inactive

**Endpoint:** `PATCH /api/v1/assets/{assetId}/attribute-defs/{defId}/rules/{evaluationOrder}`

**Request:**
```json
{
  "is_active": false
}
```

**Response: 200 OK**
```json
{
  "rule_type": "MAX_VALUE",
  "rule_value": "500",
  "is_active": false
}
```

---

### 6. PUT Save Attribute Value (Triggers Validation)

**Endpoint:** `PUT /api/v1/assets/{assetId}/attribute-values`

**Request:**
```json
{
  "attribute_def_id": "DEF-001",
  "value": "350"
}
```

**Response: 200 OK** (All validations passed)
```json
{
  "id": "VALUE-001",
  "asset_master_id": "ASSET-123",
  "attribute_def_id": "DEF-001",
  "attribute_key": "rated_capacity",
  "value_string": "350",
  "validation_status": "VALID",
  "validation_errors": [],
  "is_overridden": true
}
```

**Response: 200 OK** (With warnings)
```json
{
  "id": "VALUE-002",
  "asset_master_id": "ASSET-123",
  "attribute_def_id": "DEF-002",
  "value_string": "750",
  "validation_status": "WARNING",
  "validation_errors": [
    {
      "rule_type": "MAX_VALUE",
      "message": "Exceeds 500 kW (warning only)",
      "severity": "WARNING"
    }
  ],
  "is_overridden": true
}
```

**Response: 422 Unprocessable Entity** (Validation failed)
```json
{
  "saved": false,
  "validation_status": "INVALID",
  "validation_errors": [
    {
      "rule_type": "REQUIRED",
      "message": "Rated Capacity is required",
      "severity": "ERROR"
    },
    {
      "rule_type": "MIN_VALUE",
      "message": "Must be ≥ 10 kW",
      "severity": "ERROR"
    }
  ]
}
```

---

### 7. GET Attribute Value (with validation status)

**Endpoint:** `GET /api/v1/assets/{assetId}/attribute-values/{attributeDefId}`

**Response: 200 OK**
```json
{
  "id": "VALUE-001",
  "attribute_def_id": "DEF-001",
  "attribute_key": "rated_capacity",
  "value_string": "350",
  "validation_status": "VALID",
  "validation_errors": [],
  "is_overridden": true,
  "updated_at": "2026-05-03T14:30:00Z",
  "updated_by": "user@example.com"
}
```

---

## Template Attribute Endpoints (Phase 2 — Future)

### 1. POST Create Template Attribute

**Endpoint:** `POST /api/v1/templates/{templateId}/attribute-defs`

**Request:**
```json
{
  "display_name": "Rated Capacity",
  "data_type": "NUMBER",
  "unit": "kW",
  "is_mandatory": true,
  "default_value": "300",
  "validation_rules": [...]
}
```

**Response: 201 Created**
```json
{
  "id": "DEF-100",
  "owner_id": "TEMPLATE-MOTOR-ID",
  "scope": "TEMPLATE",
  "attribute_key": "rated_capacity",
  "is_mandatory": true,
  "default_value": "300"
}
```

**Effect:** All NEW assets created from this template will inherit this definition.

---

### 2. GET Template Attributes

**Endpoint:** `GET /api/v1/templates/{templateId}/attribute-defs`

**Response: 200 OK**
```json
[
  {
    "id": "DEF-100",
    "attribute_key": "rated_capacity",
    "scope": "TEMPLATE",
    "is_mandatory": true,
    "default_value": "300"
  },
  {
    "id": "DEF-101",
    "attribute_key": "rpm_limit",
    "scope": "TEMPLATE",
    "is_mandatory": false
  }
]
```

---

### 3. PUT Update Template Attribute

**Endpoint:** `PUT /api/v1/templates/{templateId}/attribute-defs/{defId}`

**Request:**
```json
{
  "validation_rules": [
    { "rule_type": "MIN_VALUE", "rule_value": "15" }
  ]
}
```

**Response: 200 OK**
```json
{
  "id": "DEF-100",
  "validation_rules": [...]
}
```

**Effect:** Changes apply to all assets inheriting this definition.

---

## Rule Type Reference

| Rule Type | Data Type(s) | Parameter | Example |
|-----------|--------------|-----------|---------|
| `REQUIRED` | All | None | Must not be null |
| `MIN_VALUE` | NUMBER | `rule_value: "10"` | value ≥ 10 |
| `MAX_VALUE` | NUMBER | `rule_value: "500"` | value ≤ 500 |
| `MIN_LENGTH` | STRING | `rule_value: "5"` | length ≥ 5 |
| `MAX_LENGTH` | STRING | `rule_value: "50"` | length ≤ 50 |
| `REGEX` | STRING | `rule_value: "^[A-Z]+$"` | Pattern match |
| `ENUM` | ENUM | Uses `valid_values` | Must be in list |
| `DATE_RANGE` | DATE | `rule_value: "2020-01-01,2030-12-31"` | Date within range |

---

## Data Types Supported

| Type | Storage | Example Value | Use Case |
|------|---------|---------------|----------|
| `STRING` | text | "Silver" | Categorical text |
| `NUMBER` | decimal | 350.5 | Measurements, counts |
| `DATE` | date | 2026-05-03 | Dates, timestamps |
| `BOOLEAN` | bool | true | Yes/no flags |
| `ENUM` | enum | "Good" | Dropdown selection |
| `JSON` | jsonb | `{"key":"val"}` | Complex structures |

---

## Response Status Codes

| Code | Meaning |
|------|---------|
| `200` | OK - Request succeeded |
| `201` | Created - Resource created |
| `400` | Bad Request - Invalid input |
| `403` | Forbidden - Not allowed (e.g., updating template attr from asset endpoint) |
| `404` | Not Found - Resource doesn't exist |
| `409` | Conflict - Duplicate key or constraint violation |
| `422` | Unprocessable Entity - Validation failed |
| `500` | Internal Server Error |

---

## Common Patterns

### Pattern 1: Create and Set Value

```bash
# 1. Create attribute
curl -X POST /api/v1/assets/ASSET-123/attribute-defs \
  -d '{ "display_name": "Capacity", "data_type": "NUMBER" }'
# → Response: { "id": "DEF-NEW" }

# 2. Set value on newly created attribute
curl -X PUT /api/v1/assets/ASSET-123/attribute-values \
  -d '{ "attribute_def_id": "DEF-NEW", "value": "250" }'
# → Response: { "validation_status": "VALID" }
```

### Pattern 2: Override Inherited Template Value

```bash
# Get all attributes (see DEF-001 from template)
curl -X GET /api/v1/assets/ASSET-123/attribute-defs

# Override the inherited value
curl -X PUT /api/v1/assets/ASSET-123/attribute-values \
  -d '{ "attribute_def_id": "DEF-001", "value": "500" }'
# → Response: { "is_overridden": true }
```

### Pattern 3: Update Rules on Custom Attribute

```bash
# Get attribute details
curl -X GET /api/v1/assets/ASSET-123/attribute-defs?scope=ASSET

# Update rules
curl -X PUT /api/v1/assets/ASSET-123/attribute-defs/DEF-NEW \
  -d '{ "validation_rules": [ { "rule_type": "MIN_VALUE", "rule_value": "100" } ] }'
```

### Pattern 4: Fetch Only Inherited Attributes

```bash
# Filter by scope
curl -X GET '/api/v1/assets/ASSET-123/attribute-defs?scope=TEMPLATE'
```

### Pattern 5: Fetch Only Custom Attributes

```bash
# Filter by scope
curl -X GET '/api/v1/assets/ASSET-123/attribute-defs?scope=ASSET'
```

---

## Scope & Source Combinations

| Scenario | scope | source | owner_id | parent_attr_def | Phase |
|----------|-------|--------|----------|-----------------|-------|
| Inherited template attr | TEMPLATE | - | TEMPLATE_ID | NULL | P1/P2 |
| Asset custom attribute | ASSET | NEW | ASSET_ID | NULL | P1/P2 |
| Asset sourced from template | ASSET | FROM_TEMPLATE | ASSET_ID | TEMPLATE_DEF_ID | P2 |

---

## Error Handling Examples

### Invalid Data Type

```json
{
  "error": "Validation Error",
  "field": "data_type",
  "message": "Invalid data type. Must be one of: STRING, NUMBER, DATE, BOOLEAN, ENUM, JSON"
}
```

### Duplicate Attribute Key

```json
{
  "error": "Conflict",
  "message": "Attribute with key 'rated_capacity' already exists for this asset"
}
```

### Validation Failure

```json
{
  "error": "Unprocessable Entity",
  "validation_status": "INVALID",
  "failed_rules": [
    { "rule_type": "REQUIRED", "message": "Value is required" }
  ]
}
```

---

## Authentication

All endpoints require Bearer token authentication:

```bash
curl -X GET /api/v1/assets/ASSET-123/attribute-defs \
  -H "Authorization: Bearer {JWT_TOKEN}"
```

---

## Rate Limiting

- **Limit:** 1000 requests per minute per user
- **Header:** `X-RateLimit-Remaining: 999`

---

## Changelog

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2026-05-03 | Phase 1 endpoints + Phase 2 roadmap |

---

*End of API Reference*
