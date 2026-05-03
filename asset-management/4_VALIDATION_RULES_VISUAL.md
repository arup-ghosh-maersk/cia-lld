# Validation Rules — Visual Guide

> **Focus:** Rule types, evaluation flow diagrams | **Minimal code samples**

---

## 1. Validation Rules Overview

```
VALIDATION RULES ENGINE
═════════════════════════════════════════════════════════════

Purpose: Ensure data quality by checking attributes against
         defined constraints before saving to database.

WHEN TRIGGERED:
├─ User creates new asset
├─ User updates asset values
├─ System imports batch data
└─ Scheduled background validation

PROCESS:
Asset input → Validation engine → Check each rule → 
PASS: Save | FAIL: Return error list
```

---

## 2. Rule Type Inventory

```
┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #1: REQUIRED                                       │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  Field must have a value (not null/empty)           │
│ Example:  VIN is REQUIRED for all vehicles                   │
│ Triggers: When user tries to save without VIN                │
│ Error:    "VIN is required"                                  │
│                                                              │
│ Usage:    ├─ Mandatory fields (Make, Model)                  │
│           ├─ Primary identifiers (Serial Number)             │
│           └─ Critical business data (Part Number)            │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #2: MIN_VALUE                                      │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  Numeric value must be >= threshold                 │
│ Example:  Year >= 1900                                       │
│ Triggers: User enters Year = 1850                            │
│ Error:    "Year must be >= 1900"                             │
│                                                              │
│ Usage:    ├─ Reasonable minimums (Year, Age)                 │
│           ├─ Positive values only (Quantity, Power)          │
│           └─ Zero-or-greater (Rating, Count)                 │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #3: MAX_VALUE                                      │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  Numeric value must be <= threshold                 │
│ Example:  Temperature <= 200°C                               │
│ Triggers: User enters Temp = 250                             │
│ Error:    "Temperature must be <= 200"                       │
│                                                              │
│ Usage:    ├─ Realistic limits (Temp, Pressure)               │
│           ├─ Year not in future (Current Year)               │
│           └─ Percentage bounds (0-100%)                      │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #4: MIN_LENGTH                                     │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  String length must be >= characters                │
│ Example:  Password >= 8 characters                           │
│ Triggers: User enters 5-char password                        │
│ Error:    "Password must be at least 8 characters"           │
│                                                              │
│ Usage:    ├─ Passwords (min 8-12 chars)                      │
│           ├─ Codes (VIN is 17 chars)                         │
│           └─ Valid identifiers (SKU min length)              │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #5: MAX_LENGTH                                     │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  String length must be <= characters                │
│ Example:  Description <= 500 characters                      │
│ Triggers: User enters 600-char description                   │
│ Error:    "Description must not exceed 500 characters"       │
│                                                              │
│ Usage:    ├─ Text fields (Description, Notes)                │
│           ├─ API payloads (field size limits)                │
│           └─ Database field constraints                      │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #6: REGEX                                          │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  String must match pattern                          │
│ Example:  Email matches ^[a-zA-Z0-9._%+-]+@[a-z]+\.[a-z]+$  │
│ Triggers: User enters "invalid-email"                        │
│ Error:    "Email format is invalid"                          │
│                                                              │
│ Usage:    ├─ Email validation                                │
│           ├─ VIN format (17 alphanumeric)                    │
│           ├─ Phone numbers                                   │
│           └─ License plate format                            │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #7: ENUM                                           │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  Value must be in allowed list                      │
│ Example:  Status in [Active, Inactive, Archived]             │
│ Triggers: User enters Status = "Deleted"                     │
│ Error:    "Status must be: Active, Inactive, or Archived"    │
│                                                              │
│ Usage:    ├─ Status fields (Active, Pending, Closed)         │
│           ├─ Category selections (Type, Priority)            │
│           └─ Fixed options (Fuel Type, Transmission)         │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ RULE TYPE #8: DATE_RANGE                                     │
├──────────────────────────────────────────────────────────────┤
│ Purpose:  Date must be between start_date and end_date       │
│ Example:  Inspection date within last 12 months              │
│ Triggers: User enters date = 3 years ago                     │
│ Error:    "Inspection must be within last 12 months"         │
│                                                              │
│ Usage:    ├─ Recency checks (Last inspection, Audit)         │
│           ├─ Future dates (Warranty expiration)              │
│           └─ History range (Birth year)                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Validation Pipeline

```
┌────────────────────────────────────────────────────────────┐
│ USER SUBMITS ASSET DATA                                    │
└─────────────────┬────────────────────────────────────────┘
                  ↓
        ┌────────────────────────┐
        │ ATTRIBUTE SCHEMA CHECK │
        ├────────────────────────┤
        │ • Asset has template?  │
        │ • Required fields?     │
        │ • Data types match?    │
        └────────────┬───────────┘
                     ↓
        ┌────────────────────────┐
        │ LOAD ATTRIBUTE DEFS    │
        │ (fetch from cache/DB)  │
        └────────────┬───────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ FOR EACH MASTER_ATTRIBUTE:                                 │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 1. FETCH RULES from ATTRIBUTE_DEF                       │ │
│ │    rules = [REQUIRED, MIN_VALUE(0), MAX_VALUE(200), ...]│ │
│ └─────────┬───────────────────────────────────────────────┘ │
│           ↓                                                  │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 2. RUN EACH RULE (in order)                             │ │
│ │    For rule in rules:                                   │ │
│ │        result = rule.evaluate(value)                    │ │
│ │        if result == FAIL:                               │ │
│ │            errors.add(rule.error_message)               │ │
│ │            break (stop checking this attribute)         │ │
│ └─────────┬───────────────────────────────────────────────┘ │
│           ↓                                                  │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 3. COLLECT ERRORS                                        │ │
│ │    if errors.count > 0:                                 │ │
│ │        validation_status = FAIL                         │ │
│ │    else:                                                │ │
│ │        validation_status = PASS                         │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                     ↓
        ┌────────────────────────────┐
        │ ALL ATTRIBUTES VALIDATED?  │
        └────────┬───────────────────┘
                 ↓
        ╔════════════════════════╗
        ║ VALIDATION RESULT      ║
        ╠════════════════════════╣
        ║ PASS: Save to DB       ║
        ║ FAIL: Return errors    ║
        ╚════════════════════════╝
```

---

## 4. Rule Evaluation Examples

```
EXAMPLE 1: REQUIRED Rule
────────────────────────
Attribute: VIN (Vehicle Identification Number)
Rule: REQUIRED

Input: VIN = null
├─ Evaluate: Is null?
├─ Result: FAIL ✗
└─ Error: "VIN is required"

Input: VIN = "2HRCF4K7XCH651042"
├─ Evaluate: Is null?
├─ Result: PASS ✓
└─ Continue to next rule


EXAMPLE 2: MIN_VALUE + MAX_VALUE (Range)
──────────────────────────────────────────
Attribute: Temperature (°C)
Rules: MIN_VALUE(0), MAX_VALUE(200)

Input: Temperature = -10
├─ Rule 1 - MIN_VALUE(0): -10 >= 0? NO → FAIL ✗
└─ Error: "Temperature must be >= 0°C"

Input: Temperature = 150
├─ Rule 1 - MIN_VALUE(0): 150 >= 0? YES ✓
├─ Rule 2 - MAX_VALUE(200): 150 <= 200? YES ✓
└─ Result: PASS → Continue


EXAMPLE 3: REGEX (Pattern Matching)
────────────────────────────────────
Attribute: Email
Rule: REGEX(^[a-zA-Z0-9._%+-]+@[a-z]+\.[a-z]+$)

Input: Email = "john@example.com"
├─ Pattern match? YES ✓
└─ Result: PASS

Input: Email = "john@invalid"
├─ Pattern match? NO ✗
└─ Error: "Email format is invalid"


EXAMPLE 4: ENUM (Dropdown List)
────────────────────────────────
Attribute: Status
Rule: ENUM(["Active", "Inactive", "Archived"])

Input: Status = "Active"
├─ Is in list? YES ✓
└─ Result: PASS

Input: Status = "Deleted"
├─ Is in list? NO ✗
└─ Error: "Status must be: Active, Inactive, or Archived"


EXAMPLE 5: DATE_RANGE (Recency Check)
──────────────────────────────────────
Attribute: Last Inspection Date
Rule: DATE_RANGE(start: now - 12 months, end: now)

Input: Last Inspection = "2022-01-15"
├─ Is after: 2023-01-15? NO ✗
└─ Error: "Last inspection must be within 12 months"

Input: Last Inspection = "2024-04-20"
├─ Is within range? YES ✓
└─ Result: PASS
```

---

## 5. Rule Composition (Multiple Rules)

```
┌────────────────────────────────────────────────────────┐
│ COMPLEX ATTRIBUTE WITH MULTIPLE RULES                 │
├────────────────────────────────────────────────────────┤
│                                                        │
│ Attribute: "Engine Power Output"                      │
│ Type: NUMBER                                          │
│ Unit: kW                                              │
│                                                        │
│ Rules (applied in order):                             │
│ ┌─────────────────────────────────────────────────┐   │
│ │ 1. REQUIRED                                     │   │
│ │    └─ Value cannot be null/empty               │   │
│ │       Error: "Power output is required"        │   │
│ │                                                 │   │
│ │ 2. MIN_VALUE(0)                                │   │
│ │    └─ Power must be positive                   │   │
│ │       Error: "Power must be >= 0 kW"           │   │
│ │                                                 │   │
│ │ 3. MAX_VALUE(1000)                             │   │
│ │    └─ Power realistic maximum                  │   │
│ │       Error: "Power must be <= 1000 kW"        │   │
│ │                                                 │   │
│ │ 4. REGEX (optional)                            │   │
│ │    └─ Format check (2 decimals max)            │   │
│ │       Pattern: ^[0-9]+(\.[0-9]{1,2})?$         │   │
│ │       Error: "Power format invalid"            │   │
│ └─────────────────────────────────────────────────┘   │
│                                                        │
│ VALIDATION FLOW:                                      │
│                                                        │
│ Input: power_output = 250.5                           │
│ ├─ REQUIRED? null? NO ✓                               │
│ ├─ MIN_VALUE? 250.5 >= 0? YES ✓                       │
│ ├─ MAX_VALUE? 250.5 <= 1000? YES ✓                    │
│ ├─ REGEX? Matches pattern? YES ✓                      │
│ └─ Result: ✓ VALID                                    │
│                                                        │
│ Input: power_output = 1500.25                         │
│ ├─ REQUIRED? null? NO ✓                               │
│ ├─ MIN_VALUE? 1500.25 >= 0? YES ✓                     │
│ ├─ MAX_VALUE? 1500.25 <= 1000? NO ✗                   │
│ └─ Error: "Power must be <= 1000 kW" [STOP]           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 6. Validation Scenarios

```
SCENARIO 1: Create New Asset (All Validations)
───────────────────────────────────────────────
User creates asset from "Vehicle" template.

Template has attributes:
├─ Make (REQUIRED, STRING)
├─ Model (REQUIRED, STRING)
├─ Year (REQUIRED, NUMBER, MIN=1900, MAX=2099)
├─ VIN (REQUIRED, REGEX pattern)
└─ Color (OPTIONAL, ENUM=[Red, Blue, Green])

User input:
├─ Make: "Honda" ✓
├─ Model: "CR-V" ✓
├─ Year: 2023 ✓
├─ VIN: "2HRCF4K7XCH651042" ✓
└─ Color: "Silver" ✗ (not in enum)

Result: FAIL with error:
└─ "Color must be: Red, Blue, or Green"

User corrects:
└─ Color: "Blue" ✓

Result: ALL PASS → Asset created & saved


SCENARIO 2: Batch Import (Partial Failures)
────────────────────────────────────────────
Admin imports 1000 vehicles from CSV.

System validates each row:
├─ Row 1: ✓ Valid → Created
├─ Row 2: ✓ Valid → Created
├─ Row 3: ✗ Invalid (Year=1850, < 1900) → Rejected
├─ Row 4: ✗ Invalid (VIN missing) → Rejected
├─ Row 5: ✓ Valid → Created
└─ ...

Result: Import summary
├─ Total: 1000
├─ Success: 998
├─ Failed: 2
└─ Errors: [Log with details]

Admin can:
├─ Download error list
├─ Fix & re-import
└─ Or skip failed rows


SCENARIO 3: Update Validation (Only Changed Fields)
────────────────────────────────────────────────────
User updates existing vehicle:

Before:
├─ Make: "Honda"
├─ Model: "CR-V"
├─ Year: 2023
├─ VIN: "2HRCF4K7XCH651042"
└─ Color: "Blue"

User changes only:
└─ Year: 2023 → 2024

Validation runs for:
├─ Year: 2024
│  ├─ REQUIRED? NO (has value) ✓
│  ├─ MIN_VALUE(1900)? 2024 >= 1900? YES ✓
│  ├─ MAX_VALUE(2099)? 2024 <= 2099? YES ✓
│  └─ Result: ✓ VALID
└─ Other fields: NOT validated (unchanged)

Result: Update saved successfully


SCENARIO 4: Conditional Validation (Template Rules)
────────────────────────────────────────────────────
Asset template "Industrial Equipment" has rule:
  "max_operating_temp is REQUIRED for power >= 100 kW"

Case A: power_output = 50 kW
├─ max_operating_temp = null
├─ Condition: power >= 100? NO
└─ Rule applies? NO → OK to save ✓

Case B: power_output = 150 kW
├─ max_operating_temp = null
├─ Condition: power >= 100? YES
└─ Rule applies: REQUIRED → Error ✗
```

---

## 7. Error Handling & User Feedback

```
VALIDATION ERROR RESPONSE
═════════════════════════════════════════════════════════════

HTTP 400 Bad Request
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "vin",
      "value": "invalid-vin",
      "rule": "REGEX",
      "message": "VIN must be 17 alphanumeric characters"
    },
    {
      "field": "year",
      "value": 1850,
      "rule": "MIN_VALUE",
      "message": "Year must be >= 1900"
    },
    {
      "field": "temperature",
      "value": null,
      "rule": "REQUIRED",
      "message": "Temperature is required for equipment"
    }
  ]
}

USER SEES IN UI:
┌──────────────────────────────────┐
│ ❌ Validation Failed (3 errors)  │
├──────────────────────────────────┤
│                                  │
│ VIN                              │
│ [invalid-vin           ] ✗        │
│ ⚠️ VIN must be 17 alphanumeric   │
│                                  │
│ Year                             │
│ [1850                  ] ✗        │
│ ⚠️ Year must be >= 1900          │
│                                  │
│ Temperature                      │
│ [                      ] ✗        │
│ ⚠️ Temperature is required        │
│                                  │
│           [← Back]  [Fix]        │
└──────────────────────────────────┘
```

---

## 8. Performance Considerations

```
OPTIMIZATION STRATEGIES
═════════════════════════════════════════════════════════════

1. RULE CACHING
   ├─ Cache ATTRIBUTE_DEF with rules in memory
   ├─ TTL: 1 hour (or manual invalidation)
   └─ Reduces DB queries for validation

2. SHORT-CIRCUIT EVALUATION
   ├─ Stop checking rules on first failure
   ├─ Don't waste time on invalid attributes
   └─ Return error immediately

3. ASYNC VALIDATION (Optional)
   ├─ For heavy/slow rules (e.g., DB lookup)
   ├─ Run in background after initial save
   └─ Notify user asynchronously of results

4. BATCH VALIDATION
   ├─ For bulk imports, validate in parallel
   ├─ Distribute across multiple threads
   └─ Faster processing of 1000s of records

5. RULE DEPENDENCY MANAGEMENT
   ├─ Don't apply REGEX if REQUIRED fails
   ├─ Skip MAX_VALUE if MIN_VALUE fails
   └─ Avoid redundant checks

TYPICAL PERFORMANCE:
└─ Single attribute validation: < 1ms
└─ Full asset (20 attributes): < 20ms
└─ Batch 1000 assets: < 20 seconds
```

---

## 9. Custom Rules (Advanced)

```
EXTENSIBLE RULE SYSTEM
═════════════════════════════════════════════════════════════

Beyond the 8 standard rules, custom rules can be added:

EXAMPLE: Business Logic Rule
─────────────────────────────
Rule: "If asset_type=Vehicle, then maintenance_schedule required"

Implementation:
├─ Rule name: CONDITIONAL_REQUIRED
├─ Condition: asset_type == "Vehicle"
├─ Action: Require maintenance_schedule field
└─ Error: "Maintenance schedule required for vehicles"

EXAMPLE: Cross-Field Validation
────────────────────────────────
Rule: "End date must be after start date"

Implementation:
├─ Rule name: CROSS_FIELD_COMPARE
├─ Field 1: start_date
├─ Field 2: end_date
├─ Logic: end_date > start_date
└─ Error: "End date must be after start date"

EXAMPLE: External Service Validation
─────────────────────────────────────
Rule: "VIN must exist in external vehicle database"

Implementation:
├─ Rule name: EXTERNAL_LOOKUP
├─ Service: VehicleDatabase API
├─ Lookup: Query by VIN
├─ Timeout: 5 seconds
└─ Fallback: Warn, allow to proceed
```

---

## Quick Reference

| Rule Type | Check | Example |
|-----------|-------|---------|
| **REQUIRED** | Not null/empty | Email field |
| **MIN_VALUE** | Value >= limit | Year >= 1900 |
| **MAX_VALUE** | Value <= limit | Temp <= 200 |
| **MIN_LENGTH** | String >= chars | Password >= 8 |
| **MAX_LENGTH** | String <= chars | Description <= 500 |
| **REGEX** | Pattern match | VIN format check |
| **ENUM** | In allowed list | Status: Active/Inactive |
| **DATE_RANGE** | Between dates | Inspection within 12mo |

---

## Related Documents

- **Architecture** → [1_ARCHITECTURE.md](./1_ARCHITECTURE.md)
- **Attributes** → [3_ATTRIBUTES_VISUAL.md](./3_ATTRIBUTES_VISUAL.md)
- **API Reference** → [9_API_ENDPOINTS.md](./9_API_ENDPOINTS.md)
- **Master Visual Guide** → [MASTER_VISUAL_GUIDE.md](./MASTER_VISUAL_GUIDE.md)

---

**Last Updated:** 2026-05-03 | **Version:** 2.0
