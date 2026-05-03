# Attribute Management System

> **Date:** May 3, 2026 | **Version:** 1.0 | **Status:** Production Ready (Phase 2)

## Table of Contents
1. [Overview](#overview)
2. [Data Model](#data-model)
3. [Attribute Definition](#attribute-definition)
4. [Attribute Inheritance](#attribute-inheritance)
5. [Attribute Scope](#attribute-scope)
6. [Key Operations](#key-operations)
7. [Code Examples](#code-examples)

---

## Overview

The **Attribute Management System** enables dynamic, reusable attributes across the Asset Master hierarchy. Instead of hard-coding fields, users define attributes once and apply them to templates and assets.

### Core Concepts
- **ATTRIBUTE_DEF:** Global attribute definitions (reusable templates)
- **TEMPLATE_ATTRIBUTE:** Attributes assigned to templates (inheritance point)
- **MASTER_ATTRIBUTE:** Attribute values on individual assets
- **Inheritance:** Assets inherit attributes from their templates
- **Scope Management:** Attributes can be scoped to object types, levels, categories

### Benefits
- ✅ **Flexibility:** Add attributes without code changes
- ✅ **Reusability:** One definition, multiple templates
- ✅ **Consistency:** Enforced rules and validation
- ✅ **Performance:** Indexed lookups, cached definitions

---

## Data Model

### ATTRIBUTE_DEF Table
Global attribute definition registry.

```sql
CREATE TABLE ATTRIBUTE_DEF (
    id UUID PRIMARY KEY,
    attribute_key VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    description TEXT,
    data_type VARCHAR(20) NOT NULL,  -- STRING, NUMBER, DATE, BOOLEAN, ENUM, JSON
    unit VARCHAR(50),
    group_name VARCHAR(100),
    default_value VARCHAR(1000),
    valid_values JSONB,
    sort_order INT DEFAULT 0,
    is_required BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    validation_rules JSONB NOT NULL DEFAULT '[]'::JSONB,
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_attribute_key ON ATTRIBUTE_DEF(attribute_key);
CREATE INDEX idx_group_name ON ATTRIBUTE_DEF(group_name);
CREATE INDEX idx_data_type ON ATTRIBUTE_DEF(data_type);
```

**Key Fields:**
- `attribute_key` — Unique identifier (snake_case, e.g., `max_operating_temp`)
- `data_type` — STRING, NUMBER, DATE, BOOLEAN, ENUM, JSON
- `validation_rules` — JSONB array of rule objects (see [4_VALIDATION_RULES.md](./4_VALIDATION_RULES.md))
- `valid_values` — For ENUM type: `["LOW", "MEDIUM", "HIGH"]`
- `unit` — Optional unit (e.g., "°C", "kW", "hours")

### TEMPLATE_ATTRIBUTE Table
Attributes assigned to templates.

```sql
CREATE TABLE TEMPLATE_ATTRIBUTE (
    id UUID PRIMARY KEY,
    asset_template_id UUID NOT NULL,
    attribute_def_id UUID NOT NULL,
    is_mandatory BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (asset_template_id) REFERENCES ASSET_TEMPLATE(id),
    FOREIGN KEY (attribute_def_id) REFERENCES ATTRIBUTE_DEF(id),
    UNIQUE (asset_template_id, attribute_def_id)
);

CREATE INDEX idx_asset_template_id ON TEMPLATE_ATTRIBUTE(asset_template_id);
CREATE INDEX idx_attribute_def_id ON TEMPLATE_ATTRIBUTE(attribute_def_id);
```

### MASTER_ATTRIBUTE Table
Attribute values on individual assets.

```sql
CREATE TABLE MASTER_ATTRIBUTE (
    id UUID PRIMARY KEY,
    asset_master_id UUID NOT NULL,
    attribute_def_id UUID NOT NULL,
    template_attribute_id UUID,
    attribute_value VARCHAR(4000),
    is_inherited BOOLEAN DEFAULT FALSE,
    source VARCHAR(50),  -- TEMPLATE, USER_INPUT, SYSTEM
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (asset_master_id) REFERENCES ASSET_MASTER(id),
    FOREIGN KEY (attribute_def_id) REFERENCES ATTRIBUTE_DEF(id),
    FOREIGN KEY (template_attribute_id) REFERENCES TEMPLATE_ATTRIBUTE(id),
    UNIQUE (asset_master_id, attribute_def_id)
);

CREATE INDEX idx_asset_master_id ON MASTER_ATTRIBUTE(asset_master_id);
CREATE INDEX idx_attribute_def_id ON MASTER_ATTRIBUTE(attribute_def_id);
CREATE INDEX idx_is_inherited ON MASTER_ATTRIBUTE(is_inherited);
```

---

## Attribute Definition

### Data Types Supported

| Type | Example | Default Rule | Notes |
|------|---------|---------------|-------|
| **STRING** | "Pump Model XYZ" | MAX_LENGTH: 255 | Text input, searchable |
| **NUMBER** | 1500 | MIN/MAX_VALUE | Integer or decimal |
| **DATE** | 2026-05-03 | DATE_RANGE | YYYY-MM-DD format |
| **BOOLEAN** | true | ENUM: [true, false] | Toggle/checkbox |
| **ENUM** | "HIGH" | ENUM: valid_values | Dropdown list |
| **JSON** | `{"temp": 80, "pressure": 120}` | JSON_SCHEMA | Complex nested data |

### Attribute States

```
┌─────────────────────────────────────────────────────┐
│  State Diagram: Attribute Lifecycle                 │
└─────────────────────────────────────────────────────┘

CREATED (Draft)
    ↓
ACTIVE (Ready for use)
    ↓
DEPRECATED (Plan removal)
    ↓
INACTIVE (Archived)
```

### Example: Attribute Definition for Motor Speed

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "attribute_key": "motor_speed_rpm",
  "display_name": "Motor Speed (RPM)",
  "description": "Rated speed of the electric motor",
  "data_type": "NUMBER",
  "unit": "RPM",
  "group_name": "Motor Specifications",
  "default_value": "1500",
  "valid_values": null,
  "sort_order": 1,
  "is_required": true,
  "is_active": true,
  "validation_rules": [
    {
      "rule_type": "REQUIRED",
      "error_message": "Motor speed is required",
      "severity": "ERROR",
      "evaluation_order": 1
    },
    {
      "rule_type": "MIN_VALUE",
      "rule_value": "100",
      "error_message": "Motor speed must be at least 100 RPM",
      "severity": "ERROR",
      "evaluation_order": 2
    },
    {
      "rule_type": "MAX_VALUE",
      "rule_value": "10000",
      "error_message": "Motor speed cannot exceed 10,000 RPM",
      "severity": "ERROR",
      "evaluation_order": 3
    }
  ],
  "notes": "Standard IEC motor speeds: 750, 1000, 1500, 3000 RPM"
}
```

---

## Attribute Inheritance

### Inheritance Flow

```
ATTRIBUTE_DEF (Global)
    ↓
    └─→ TEMPLATE_ATTRIBUTE (Template-level)
            ↓
            └─→ MASTER_ATTRIBUTE (Asset-level)
```

### Rules
1. **Templates define scope:** If attribute added to template, only assets of that template can have it
2. **Assets can override:** Child assets can override inherited values
3. **Track source:** `source` field shows: TEMPLATE, USER_INPUT, SYSTEM
4. **Inheritance flag:** `is_inherited` = true if value came from template
5. **Path tracking:** `template_attribute_id` links asset attribute back to template

### Inheritance Example

```sql
-- 1. Define attribute globally
INSERT INTO ATTRIBUTE_DEF (
  attribute_key, display_name, data_type, is_required, validation_rules
) VALUES (
  'installation_date', 'Installation Date', 'DATE', true, 
  '[{"rule_type": "REQUIRED", ...}]'::JSONB
);

-- 2. Assign to template
INSERT INTO TEMPLATE_ATTRIBUTE (
  asset_template_id, attribute_def_id, is_mandatory
) VALUES (
  'template-pump-001', 'attr-installation-date', true
);

-- 3. Asset inherits from template
INSERT INTO MASTER_ATTRIBUTE (
  asset_master_id, attribute_def_id, template_attribute_id,
  attribute_value, is_inherited, source
) VALUES (
  'asset-pump-123', 'attr-installation-date', 'template-attr-001',
  '2024-01-15', true, 'TEMPLATE'
);

-- 4. Override inheritance
UPDATE MASTER_ATTRIBUTE SET
  attribute_value = '2024-01-20',
  is_inherited = false,
  source = 'USER_INPUT'
WHERE asset_master_id = 'asset-pump-123'
  AND attribute_def_id = 'attr-installation-date';
```

---

## Attribute Scope

### Scope Mechanism
Attributes can be restricted to specific contexts:

```sql
CREATE TABLE ATTRIBUTE_SCOPE (
    id UUID PRIMARY KEY,
    attribute_def_id UUID NOT NULL,
    object_type VARCHAR(50),        -- NULL = all types
    object_level INT,               -- NULL = all levels
    category VARCHAR(100),          -- NULL = all categories
    functional_type VARCHAR(100),   -- NULL = all types
    FOREIGN KEY (attribute_def_id) REFERENCES ATTRIBUTE_DEF(id),
    UNIQUE (attribute_def_id, object_type, object_level, category, functional_type)
);
```

### Example Scoping
```
Attribute: "blade_count"
Scopes:
  - object_type = "EQUIPMENT" AND functional_type = "FAN"
  - object_type = "EQUIPMENT" AND functional_type = "COMPRESSOR"

Result: "blade_count" only appears on fans and compressors
```

---

## Key Operations

### Query: Get All Attributes for Asset
```sql
-- Get all attribute definitions applicable to an asset
SELECT DISTINCT ad.*, ma.attribute_value, ma.is_inherited
FROM ATTRIBUTE_DEF ad
JOIN TEMPLATE_ATTRIBUTE ta ON ad.id = ta.attribute_def_id
JOIN ASSET_MASTER am ON am.asset_template_id = ta.asset_template_id
LEFT JOIN MASTER_ATTRIBUTE ma ON ad.id = ma.attribute_def_id 
  AND am.id = ma.asset_master_id
WHERE am.id = $1
ORDER BY ad.group_name, ad.sort_order;
```

### Query: Inherited vs Overridden Attributes
```sql
-- Find attributes that were inherited but are now overridden
SELECT ma.*, ad.display_name
FROM MASTER_ATTRIBUTE ma
JOIN ATTRIBUTE_DEF ad ON ma.attribute_def_id = ad.id
WHERE ma.asset_master_id = $1
  AND ma.is_inherited = false
  AND ma.source = 'TEMPLATE'
  AND ma.attribute_value != (
    SELECT ma2.attribute_value 
    FROM MASTER_ATTRIBUTE ma2 
    WHERE ma2.asset_master_id = ma.asset_master_id
      AND ma2.attribute_def_id = ma.attribute_def_id
  );
```

### Operation: Create Attribute with Validation Rules
```sql
BEGIN TRANSACTION;

-- 1. Create attribute definition
INSERT INTO ATTRIBUTE_DEF (
  attribute_key, display_name, data_type, unit, 
  group_name, default_value, is_required, validation_rules
) VALUES (
  'operating_temperature', 'Operating Temperature (°C)', 'NUMBER', '°C',
  'Operating Parameters', '25', true,
  '[
    {"rule_type": "REQUIRED", "severity": "ERROR", "evaluation_order": 1},
    {"rule_type": "MIN_VALUE", "rule_value": "-40", "severity": "ERROR", "evaluation_order": 2},
    {"rule_type": "MAX_VALUE", "rule_value": "150", "severity": "ERROR", "evaluation_order": 3}
  ]'::JSONB
) RETURNING id;

-- 2. Assign to template (result: template-pump-001)
INSERT INTO TEMPLATE_ATTRIBUTE (asset_template_id, attribute_def_id, is_mandatory)
SELECT 'template-pump-001', id FROM ATTRIBUTE_DEF WHERE attribute_key = 'operating_temperature';

COMMIT;
```

---

## Code Examples

### C# — Retrieve Asset Attributes
```csharp
public class AttributeService
{
    public async Task<List<AttributeDto>> GetAssetAttributesAsync(Guid assetId)
    {
        var sql = @"
            SELECT ad.*, ma.attribute_value, ma.is_inherited
            FROM ATTRIBUTE_DEF ad
            JOIN TEMPLATE_ATTRIBUTE ta ON ad.id = ta.attribute_def_id
            JOIN ASSET_MASTER am ON am.asset_template_id = ta.asset_template_id
            LEFT JOIN MASTER_ATTRIBUTE ma ON ad.id = ma.attribute_def_id 
              AND am.id = ma.asset_master_id
            WHERE am.id = @AssetId
            ORDER BY ad.group_name, ad.sort_order";

        using var connection = new NpgsqlConnection(_connectionString);
        var attributes = await connection.QueryAsync<AttributeDto>(sql, 
            new { AssetId = assetId });
        
        return attributes.ToList();
    }
}

public class AttributeDto
{
    public Guid Id { get; set; }
    public string DisplayName { get; set; }
    public string DataType { get; set; }
    public string AttributeValue { get; set; }
    public bool IsInherited { get; set; }
    public string Unit { get; set; }
    public List<ValidationRule> ValidationRules { get; set; }
}
```

### C# — Validate Attribute Value
```csharp
public class AttributeValidator
{
    public async Task<ValidationResult> ValidateAttributeAsync(
        Guid attributeDefId, string value)
    {
        var attrDef = await _repo.GetAttributeDefAsync(attributeDefId);
        if (attrDef == null)
            return ValidationResult.Failure("Attribute not found");

        var errors = new List<string>();
        var warnings = new List<string>();

        foreach (var rule in attrDef.ValidationRules.OrderBy(r => r.EvaluationOrder))
        {
            var result = rule.RuleType switch
            {
                "REQUIRED" => ValidateRequired(value, rule),
                "MIN_VALUE" => ValidateMinValue(value, rule),
                "MAX_VALUE" => ValidateMaxValue(value, rule),
                "MIN_LENGTH" => ValidateMinLength(value, rule),
                "MAX_LENGTH" => ValidateMaxLength(value, rule),
                "REGEX" => ValidateRegex(value, rule),
                "ENUM" => ValidateEnum(value, rule),
                "DATE_RANGE" => ValidateDateRange(value, rule),
                _ => ValidationRuleResult.Success()
            };

            if (!result.IsValid)
            {
                if (rule.Severity == "ERROR")
                    errors.Add(result.ErrorMessage);
                else
                    warnings.Add(result.ErrorMessage);
            }
        }

        return errors.Any() 
            ? ValidationResult.Failure(errors) 
            : new ValidationResult(warnings);
    }
}
```

### Query: Attributes by Group
```sql
SELECT 
  group_name,
  attribute_key,
  display_name,
  data_type,
  unit,
  is_required,
  COUNT(*) OVER (PARTITION BY group_name) as group_size
FROM ATTRIBUTE_DEF
WHERE is_active = true
ORDER BY group_name, sort_order;
```

---

## Related Documents
- [1_ARCHITECTURE.md](./1_ARCHITECTURE.md) — Data model overview
- [4_VALIDATION_RULES.md](./4_VALIDATION_RULES.md) — Validation rule engine details
- [ATTRIBUTE_INHERITANCE_GUIDE.md](./ATTRIBUTE_INHERITANCE_GUIDE.md) — Deep dive on inheritance
- [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) — Quick attribute lookup

---

**Last Updated:** May 3, 2026 | **Next Review:** Phase 2 completion
