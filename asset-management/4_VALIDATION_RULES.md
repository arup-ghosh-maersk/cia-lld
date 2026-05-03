# Validation Rules Engine

> **Date:** May 3, 2026 | **Version:** 1.0 | **Status:** Production Ready (Phase 2)

## Table of Contents
1. [Overview](#overview)
2. [Rule Types](#rule-types)
3. [Rule Format](#rule-format)
4. [Evaluation Pipeline](#evaluation-pipeline)
5. [Configuration](#configuration)
6. [Examples](#examples)
7. [Best Practices](#best-practices)

---

## Overview

The **Validation Rules Engine** allows dynamic validation rules on attributes without code changes. Rules are stored as JSONB on `ATTRIBUTE_DEF.validation_rules` and evaluated at attribute assignment time.

### Key Features
- ✅ **8 Rule Types:** REQUIRED, MIN/MAX_VALUE, MIN/MAX_LENGTH, REGEX, ENUM, DATE_RANGE
- ✅ **Severity Levels:** ERROR (blocks save), WARNING (allows with alert)
- ✅ **Ordering:** Evaluation order configurable (short-circuit on first error)
- ✅ **Extensible:** Easy to add new rule types
- ✅ **Performance:** Rules cached in memory with invalidation

### Why Not Hard-Code?
- **Speed:** Define rules in UI without code deploy
- **Consistency:** Same rules across all systems
- **Change Management:** Audit trail of rule changes
- **Flexibility:** Different rules per template, asset type

---

## Rule Types

### 1. REQUIRED
**Purpose:** Attribute must have a value.

```json
{
  "rule_type": "REQUIRED",
  "error_message": "This field is required",
  "severity": "ERROR",
  "evaluation_order": 1
}
```

**Evaluation:**
```
value IS NOT NULL AND value.Trim().Length > 0
```

**Example Failures:**
- `null` → ❌ FAIL
- `""` → ❌ FAIL
- `"   "` → ❌ FAIL
- `"valid"` → ✅ PASS

---

### 2. MIN_VALUE
**Purpose:** Numeric value must be ≥ minimum.

```json
{
  "rule_type": "MIN_VALUE",
  "rule_value": "100",
  "error_message": "Must be at least 100",
  "severity": "ERROR",
  "evaluation_order": 2
}
```

**Evaluation:**
```
decimal.Parse(value) >= decimal.Parse(rule_value)
```

**Example Failures (rule_value = 100):**
- `99` → ❌ FAIL
- `100` → ✅ PASS
- `100.01` → ✅ PASS

---

### 3. MAX_VALUE
**Purpose:** Numeric value must be ≤ maximum.

```json
{
  "rule_type": "MAX_VALUE",
  "rule_value": "500",
  "error_message": "Cannot exceed 500",
  "severity": "WARNING",
  "evaluation_order": 3
}
```

**Evaluation:**
```
decimal.Parse(value) <= decimal.Parse(rule_value)
```

**Example Failures (rule_value = 500):**
- `501` → ❌ FAIL
- `500` → ✅ PASS
- `499.99` → ✅ PASS

---

### 4. MIN_LENGTH
**Purpose:** String length must be ≥ minimum.

```json
{
  "rule_type": "MIN_LENGTH",
  "rule_value": "3",
  "error_message": "At least 3 characters required",
  "severity": "ERROR",
  "evaluation_order": 4
}
```

**Evaluation:**
```
value.Length >= int.Parse(rule_value)
```

**Example Failures (rule_value = 3):**
- `"ab"` → ❌ FAIL
- `"abc"` → ✅ PASS
- `"abcd"` → ✅ PASS

---

### 5. MAX_LENGTH
**Purpose:** String length must be ≤ maximum.

```json
{
  "rule_type": "MAX_LENGTH",
  "rule_value": "255",
  "error_message": "Maximum 255 characters allowed",
  "severity": "ERROR",
  "evaluation_order": 5
}
```

**Evaluation:**
```
value.Length <= int.Parse(rule_value)
```

**Example Failures (rule_value = 255):**
- `"x" * 256` → ❌ FAIL
- `"x" * 255` → ✅ PASS

---

### 6. REGEX
**Purpose:** String must match regex pattern.

```json
{
  "rule_type": "REGEX",
  "rule_value": "^[A-Z]{2}-[0-9]{4}$",
  "error_message": "Format must be XX-0000 (e.g., AB-1234)",
  "severity": "ERROR",
  "evaluation_order": 6
}
```

**Evaluation:**
```
Regex.IsMatch(value, rule_value)
```

**Example Failures (pattern = `^[A-Z]{2}-[0-9]{4}$`):**
- `"AB-1234"` → ✅ PASS
- `"ab-1234"` → ❌ FAIL (lowercase)
- `"ABC-123"` → ❌ FAIL (wrong length)
- `"AB-123A"` → ❌ FAIL (letter in number)

**Common Patterns:**
```
UUID:                  ^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$
Email:                 ^[^\s@]+@[^\s@]+\.[^\s@]+$
Phone (US):            ^\\(?\\d{3}\\)?[\\s.-]?\\d{3}[\\s.-]?\\d{4}$
ISO Date (YYYY-MM-DD): ^\\d{4}-\\d{2}-\\d{2}$
Serial (no spaces):    ^[A-Z0-9]{5,20}$
```

---

### 7. ENUM
**Purpose:** Value must be in allowed list.

```json
{
  "rule_type": "ENUM",
  "rule_value": "[\"LOW\", \"MEDIUM\", \"HIGH\", \"CRITICAL\"]",
  "error_message": "Must be LOW, MEDIUM, HIGH, or CRITICAL",
  "severity": "ERROR",
  "evaluation_order": 7
}
```

**Evaluation:**
```
allowedValues.Contains(value, StringComparer.OrdinalIgnoreCase)
```

**Example Failures (allowed = LOW, MEDIUM, HIGH):**
- `"LOW"` → ✅ PASS
- `"Medium"` → ✅ PASS (case-insensitive)
- `"INVALID"` → ❌ FAIL

---

### 8. DATE_RANGE
**Purpose:** Date must be within range.

```json
{
  "rule_type": "DATE_RANGE",
  "rule_value": "{\"from\": \"2024-01-01\", \"to\": \"2026-12-31\"}",
  "error_message": "Date must be between 2024-01-01 and 2026-12-31",
  "severity": "ERROR",
  "evaluation_order": 8
}
```

**Evaluation:**
```
DateTime.ParseExact(value, "yyyy-MM-dd", CultureInfo.InvariantCulture) 
  >= DateTime.Parse(from) 
  AND <= DateTime.Parse(to)
```

**Example Failures (range = 2024-01-01 to 2026-12-31):**
- `"2024-01-01"` → ✅ PASS
- `"2025-06-15"` → ✅ PASS
- `"2026-12-31"` → ✅ PASS
- `"2023-12-31"` → ❌ FAIL (before start)
- `"2027-01-01"` → ❌ FAIL (after end)

---

## Rule Format

### Complete Rule Object

```json
{
  "rule_type": "MIN_VALUE",
  "rule_value": "100",
  "error_message": "Value must be at least 100",
  "severity": "ERROR",
  "evaluation_order": 1,
  "is_active": true,
  "created_at": "2026-05-03T10:00:00Z",
  "created_by": "system"
}
```

### Field Descriptions

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `rule_type` | string | ✅ | One of 8 types above |
| `rule_value` | string | ✅ | The constraint value(s) |
| `error_message` | string | ✅ | User-facing error text |
| `severity` | string | ✅ | ERROR or WARNING |
| `evaluation_order` | int | ✅ | 1-99, defines execution order |
| `is_active` | boolean | optional | Default: true |
| `created_at` | timestamp | optional | Audit trail |
| `created_by` | string | optional | Who created the rule |

### Severity Levels

| Severity | Behavior | Use Case |
|----------|----------|----------|
| **ERROR** | Blocks attribute save, returned to UI | Mandatory constraints |
| **WARNING** | Allows save with warning message | Soft constraints, recommendations |

---

## Evaluation Pipeline

### Step-by-Step Flow

```
┌──────────────────────────────────────────────────────────┐
│ 1. ATTRIBUTE CHANGE TRIGGERED                            │
│    (User updates attribute value)                         │
└────────────────┬─────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────┐
│ 2. LOAD ATTRIBUTE DEFINITION                             │
│    (Fetch from ATTRIBUTE_DEF table)                       │
└────────────────┬─────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────┐
│ 3. SORT RULES BY EVALUATION_ORDER                        │
│    (Order 1, 2, 3, ...)                                  │
└────────────────┬─────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────┐
│ 4. ITERATE THROUGH RULES (In Order)                      │
│                                                          │
│   FOR EACH rule WHERE is_active = true:                 │
│     → Evaluate rule against value                        │
│     → If FAIL:                                           │
│       - If severity = ERROR: Collect error               │
│       - If severity = WARNING: Collect warning           │
│     → Continue to next rule                              │
└────────────────┬─────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────┐
│ 5. AGGREGATE RESULTS                                     │
│    - List of errors (blocking)                           │
│    - List of warnings (non-blocking)                     │
└────────────────┬─────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────┐
│ 6. RETURN VALIDATION RESULT                              │
│                                                          │
│   IF errors.Any(): Return FAILURE with errors           │
│   ELSE: Return SUCCESS (with warnings if any)           │
└──────────────────────────────────────────────────────────┘
```

### Example Evaluation

```
Rule Set for "operating_temperature":
  [
    { order: 1, type: REQUIRED, severity: ERROR },
    { order: 2, type: MIN_VALUE, value: "-40", severity: ERROR },
    { order: 3, type: MAX_VALUE, value: "150", severity: ERROR }
  ]

User Input: "-50"

Evaluation:
  Rule 1 (REQUIRED): "-50" is not empty ✅ PASS
  Rule 2 (MIN_VALUE): -50 >= -40? ❌ FAIL → Add error "Must be >= -40"
  Rule 3 (MAX_VALUE): Skip (has error, could still check)
  
Result: FAILURE ["Must be >= -40"]
```

---

## Configuration

### Lazy Loading & Caching

```csharp
public class ValidationRuleCache
{
    private readonly IDistributedCache _cache;
    private readonly IRepository _repo;
    
    public async Task<ValidationRuleSet> GetRulesAsync(Guid attributeDefId)
    {
        string cacheKey = $"attr_rules_{attributeDefId}";
        
        // Try cache first
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
            return JsonSerializer.Deserialize<ValidationRuleSet>(cached);
        
        // Cache miss, load from DB
        var rules = await _repo.GetValidationRulesAsync(attributeDefId);
        
        // Cache for 1 hour
        await _cache.SetStringAsync(cacheKey, 
            JsonSerializer.Serialize(rules),
            new DistributedCacheEntryOptions 
            { 
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            });
        
        return rules;
    }
}
```

### Rule Priority (Recommended Order)

```
1. REQUIRED — Fail fast if empty
2. MIN_VALUE / MIN_LENGTH — Lower bounds
3. MAX_VALUE / MAX_LENGTH — Upper bounds
4. ENUM — Value must be in list
5. REGEX — Pattern matching
6. DATE_RANGE — Date constraints
```

---

## Examples

### Example 1: Motor RPM Attribute
```json
{
  "attribute_key": "motor_speed_rpm",
  "display_name": "Motor Speed",
  "data_type": "NUMBER",
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
      "error_message": "Minimum 100 RPM",
      "severity": "ERROR",
      "evaluation_order": 2
    },
    {
      "rule_type": "MAX_VALUE",
      "rule_value": "10000",
      "error_message": "Maximum 10,000 RPM",
      "severity": "ERROR",
      "evaluation_order": 3
    }
  ]
}
```

### Example 2: Asset Serial Number
```json
{
  "attribute_key": "serial_number",
  "display_name": "Serial Number",
  "data_type": "STRING",
  "validation_rules": [
    {
      "rule_type": "REQUIRED",
      "error_message": "Serial number required",
      "severity": "ERROR",
      "evaluation_order": 1
    },
    {
      "rule_type": "MIN_LENGTH",
      "rule_value": "5",
      "error_message": "At least 5 characters",
      "severity": "ERROR",
      "evaluation_order": 2
    },
    {
      "rule_type": "MAX_LENGTH",
      "rule_value": "50",
      "error_message": "Maximum 50 characters",
      "severity": "ERROR",
      "evaluation_order": 3
    },
    {
      "rule_type": "REGEX",
      "rule_value": "^[A-Z0-9-]+$",
      "error_message": "Only uppercase letters, numbers, and hyphens",
      "severity": "ERROR",
      "evaluation_order": 4
    }
  ]
}
```

### Example 3: Priority Level (Enum)
```json
{
  "attribute_key": "priority",
  "display_name": "Priority",
  "data_type": "ENUM",
  "valid_values": ["LOW", "MEDIUM", "HIGH", "CRITICAL"],
  "validation_rules": [
    {
      "rule_type": "REQUIRED",
      "error_message": "Priority must be specified",
      "severity": "ERROR",
      "evaluation_order": 1
    },
    {
      "rule_type": "ENUM",
      "rule_value": "[\"LOW\", \"MEDIUM\", \"HIGH\", \"CRITICAL\"]",
      "error_message": "Invalid priority level",
      "severity": "ERROR",
      "evaluation_order": 2
    }
  ]
}
```

---

## Best Practices

### ✅ DO
- **Order rules logically:** REQUIRED → range checks → format checks
- **Clear messages:** "Must be between 100 and 500" not "range_error_1"
- **Use appropriate severity:** ERROR for critical, WARNING for soft constraints
- **Cache rules:** Load once, use many times
- **Document patterns:** Comment REGEX patterns in code
- **Version rules:** Track changes, maintain audit trail
- **Test edge cases:** Boundaries (0, max, min, null)

### ❌ DON'T
- **Complex business logic in rules:** Keep rules simple, use code for complex validation
- **Overlapping rules:** Avoid conflicting MIN and MAX values
- **Excessive warnings:** Too many warnings = ignored alerts
- **Performance-heavy regex:** Test for performance impact
- **Hard-code constants:** Use ATTRIBUTE_DEF defaults instead
- **Assume rule order:** Always specify evaluation_order explicitly

### Performance Tips
- Rules evaluated in memory (cached)
- Short-circuit on first ERROR (no subsequent checks)
- Index on attribute_def_id for lookups
- Batch rule updates to invalidate cache once
- Monitor rule evaluation time (should be <10ms per attribute)

---

## Related Documents
- [3_ATTRIBUTES.md](./3_ATTRIBUTES.md) — Attribute system overview
- [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) — Rule type quick lookup
- [FAQ.md](./FAQ.md) — Common validation questions

---

**Last Updated:** May 3, 2026 | **Next Review:** Phase 2 completion
