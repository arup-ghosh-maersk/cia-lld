# Events & Audit Trail — Visual Guide

> **Focus:** Event sourcing, immutable audit log, event types | **Flows & diagrams**

---

## 1. Event Sourcing Architecture

```
TRADITIONAL APPROACH (State Only)
─────────────────────────────────

Asset state: {Make: Honda, Model: CR-V, Year: 2023}
         ↓ [User updates Year: 2023 → 2024]
Asset state: {Make: Honda, Model: CR-V, Year: 2024}
         ↓ [Old state lost]
         No audit trail available


EVENT SOURCING APPROACH (State + History)
─────────────────────────────────────────

ASSET_EVENT_STORE (Immutable Log):
┌────────────────────────────────────────────────────────┐
│ [1] ASSET_CREATED @ 2024-01-15 09:00                  │
│     {asset_id, template_id, Make: Honda, Model: CR-V} │
├────────────────────────────────────────────────────────┤
│ [2] ASSET_UPDATED @ 2024-01-16 10:30                  │
│     {asset_id, Year: 2023 → 2024}                     │
├────────────────────────────────────────────────────────┤
│ [3] COMPONENT_ATTACHED @ 2024-01-17 14:15             │
│     {asset_id, component_id, Engine-001}              │
├────────────────────────────────────────────────────────┤
│ [4] ATTRIBUTE_CHANGED @ 2024-01-18 09:45              │
│     {asset_id, power: 150 kW, old: 140 kW}            │
└────────────────────────────────────────────────────────┘
              ↓
Current Asset State = Replay all events
   {Make: Honda, Model: CR-V, Year: 2024, Power: 150 kW}
              ↓
Complete Audit Trail = All 4 events with timestamps


BENEFITS:
✓ Every change recorded (immutable)
✓ Who changed what, when
✓ Can replay to any point in time
✓ Compliance & audit requirements
✓ No data loss
```

---

## 2. Event Store Table Structure

```
ASSET_EVENT_STORE
═════════════════════════════════════════════════════════

Column             Type        Purpose
─────────────────────────────────────────────────────────
id                 UUID        Event ID (unique)
event_id           UUID        Idempotency key
sequence_number    BIGINT      Order (1, 2, 3, ...)
asset_master_id    UUID        Which asset?
event_type         VARCHAR     Event classification
event_timestamp    TIMESTAMP   When event occurred
user_id            UUID        Who triggered it?
event_payload      JSONB       Event data
event_metadata     JSONB       System metadata
correlation_id     UUID        Related events
causation_id       UUID        Parent event
version            INT         Aggregate version
created_at         TIMESTAMP   Record creation
─────────────────────────────────────────────────────────

Key Characteristics:
├─ IMMUTABLE (never update, only append)
├─ ORDERED (sequence_number ensures order)
├─ INDEXED on (asset_master_id, sequence_number)
└─ PARTITIONED by year (for performance)

Typical Row:

id: 550e8400-e29b-41d4-a716-446655440000
sequence_number: 3
asset_master_id: 770e8400-e29b-41d4-a716-446655440111
event_type: ASSET_UPDATED
event_timestamp: 2024-01-16 10:30:45.123 UTC
user_id: user-789
event_payload: {
  "changes": {
    "year": {"old": 2023, "new": 2024}
  },
  "updated_fields": ["year"],
  "validation_passed": true
}
event_metadata: {
  "ip_address": "192.168.1.100",
  "user_agent": "Mozilla/5.0...",
  "request_id": "req-456"
}
```

---

## 3. Complete Event Lifecycle

```
┌─────────────────────────────────┐
│ USER TRIGGERS ACTION            │
│ (Create/Update/Delete Asset)    │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 1. API RECEIVES REQUEST         │
│    • Validate token             │
│    • Check permissions          │
│    • Parse body                 │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 2. BUSINESS LOGIC               │
│    • Validate rules             │
│    • Check constraints          │
│    • Prepare changes            │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 3. PERSISTENCE                  │
│    • Save ASSET_MASTER (state)  │
│    • Save MASTER_ATTRIBUTE vals │
│    • Commit transaction         │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 4. EVENT CREATION               │
│    • Construct event object     │
│    • Include all change details │
│    • Set metadata (user, time)  │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 5. EVENT STORAGE                │
│    • Append to ASSET_EVENT_STORE│
│    • Generate sequence number   │
│    • Assign timestamp           │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 6. EVENT PUBLICATION            │
│    • Publish to message bus     │
│    • Set correlation ID         │
│    • Enable async handlers      │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 7. ASYNC HANDLERS (Parallel)    │
│    • AssetEventHandler          │
│    • NotificationHandler        │
│    • DocumentHandler            │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│ 8. RESPONSE TO USER             │
│    • 200 OK / 201 Created       │
│    • Return updated asset       │
│    • Include new ID if created  │
└─────────────────────────────────┘
```

---

## 4. Event Types Catalog

```
┌──────────────────────────────────────────────────────────┐
│ EVENT #1: ASSET_CREATED                                 │
├──────────────────────────────────────────────────────────┤
│ When:     New asset added to system                      │
│ Triggers: 1) AssetEventHandler → Generate audit msg     │
│           2) NotificationHandler → UI notification      │
│           3) DocumentHandler → Check for files          │
│                                                          │
│ Payload:                                                │
│ {                                                        │
│   "event_type": "ASSET_CREATED",                        │
│   "asset_id": "550e8400-e29b-41d4-a716-...",          │
│   "template_id": "660e8400-e29b-41d4-a716-...",        │
│   "asset_name": "Equipment-001",                        │
│   "attributes": {                                        │
│     "make": "Honda",                                     │
│     "model": "CR-V",                                     │
│     "year": 2023                                         │
│   },                                                     │
│   "created_by": "user-123",                             │
│   "created_at": "2024-01-15T09:00:00Z"                  │
│ }                                                        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ EVENT #2: ASSET_UPDATED                                 │
├──────────────────────────────────────────────────────────┤
│ When:     Asset properties changed                       │
│ Triggers: 1) AssetEventHandler → Audit message          │
│           2) NotificationHandler → Alert subscribers    │
│                                                          │
│ Payload:                                                │
│ {                                                        │
│   "event_type": "ASSET_UPDATED",                        │
│   "asset_id": "550e8400-e29b-41d4-a716-...",          │
│   "changes": {                                           │
│     "year": { "old": 2023, "new": 2024 },              │
│     "status": { "old": "DRAFT", "new": "ACTIVE" }       │
│   },                                                     │
│   "updated_fields": ["year", "status"],                 │
│   "updated_by": "user-456",                             │
│   "updated_at": "2024-01-16T10:30:00Z",                │
│   "reason": "Customer requested year correction"        │
│ }                                                        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ EVENT #3: ATTRIBUTE_CHANGED                             │
├──────────────────────────────────────────────────────────┤
│ When:     Single attribute value updated                 │
│ Triggers: 1) Validation check                           │
│           2) AssetEventHandler → Audit                  │
│                                                          │
│ Payload:                                                │
│ {                                                        │
│   "event_type": "ATTRIBUTE_CHANGED",                    │
│   "asset_id": "550e8400-e29b-41d4-a716-...",          │
│   "attribute_key": "max_operating_temp",                │
│   "old_value": 150,                                      │
│   "new_value": 160,                                      │
│   "unit": "°C",                                          │
│   "validation_passed": true,                            │
│   "changed_by": "user-789",                             │
│   "changed_at": "2024-01-17T14:15:00Z"                  │
│ }                                                        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ EVENT #4: COMPONENT_ATTACHED                            │
├──────────────────────────────────────────────────────────┤
│ When:     Component added to asset                       │
│ Triggers: 1) Component attributes inherit               │
│           2) NotificationHandler → Notification         │
│                                                          │
│ Payload:                                                │
│ {                                                        │
│   "event_type": "COMPONENT_ATTACHED",                   │
│   "asset_id": "550e8400-e29b-41d4-a716-...",          │
│   "component_id": "880e8400-e29b-41d4-a716-...",       │
│   "component_name": "Engine-001",                       │
│   "component_type": "Engine",                           │
│   "attached_by": "user-123",                            │
│   "attached_at": "2024-01-17T14:15:00Z",               │
│   "inherited_attributes": [                             │
│     "power_output",                                      │
│     "fuel_type",                                         │
│     "displacement"                                       │
│   ]                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ EVENT #5: FILE_UPLOADED                                 │
├──────────────────────────────────────────────────────────┤
│ When:     File attached to asset                         │
│ Triggers: DocumentHandler → Votiro scan                 │
│                                                          │
│ Payload:                                                │
│ {                                                        │
│   "event_type": "FILE_UPLOADED",                        │
│   "asset_id": "550e8400-e29b-41d4-a716-...",          │
│   "attachment_id": "990e8400-e29b-41d4-a716-...",      │
│   "file_name": "specification.pdf",                     │
│   "file_size": 2048000,                                  │
│   "content_type": "application/pdf",                    │
│   "file_hash": "abc123def456...",                       │
│   "uploaded_by": "user-456",                            │
│   "uploaded_at": "2024-01-18T09:45:00Z"                 │
│ }                                                        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ EVENT #6: FILE_SCANNED                                  │
├──────────────────────────────────────────────────────────┤
│ When:     File malware scan completed                    │
│ Triggers: 1) Update file status (CLEAN/QUARANTINE)      │
│           2) NotificationHandler → Alert if threat      │
│                                                          │
│ Payload:                                                │
│ {                                                        │
│   "event_type": "FILE_SCANNED",                         │
│   "attachment_id": "990e8400-e29b-41d4-a716-...",      │
│   "scan_status": "CLEAN",                               │
│   "threat_score": 0,                                     │
│   "scan_details": {                                      │
│     "malware_detected": false,                          │
│     "exploits": [],                                      │
│     "macros": "none",                                    │
│     "scan_engine": "Votiro CDR 5.2"                     │
│   },                                                     │
│   "scanned_at": "2024-01-18T10:00:00Z",                │
│   "scan_duration_ms": 15234                             │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
```

---

## 5. Audit Trail Reconstruction

```
HOW TO REPLAY ASSET HISTORY
═════════════════════════════════════════════════════════

Asset ID: 550e8400-e29b-41d4-a716-446655440000

Query:
  SELECT * FROM ASSET_EVENT_STORE
  WHERE asset_master_id = '550e8400-e29b-41d4-a716-446655440000'
  ORDER BY sequence_number ASC

Result:

┌─────────────────────────────────────────────────────────┐
│ [Seq 1] ASSET_CREATED @ 2024-01-15 09:00               │
│ State:  {id: 550e..., name: Equipment-001,             │
│         template: Vehicle, created_by: user-123}       │
├─────────────────────────────────────────────────────────┤
│ [Seq 2] ASSET_UPDATED @ 2024-01-16 10:30               │
│ Change: Year: 2023 → 2024                              │
│ State:  {..., year: 2024, ...}                         │
├─────────────────────────────────────────────────────────┤
│ [Seq 3] ATTRIBUTE_CHANGED @ 2024-01-17 09:45           │
│ Change: Power: 140 → 150 kW                            │
│ State:  {..., power_output: 150, ...}                  │
├─────────────────────────────────────────────────────────┤
│ [Seq 4] COMPONENT_ATTACHED @ 2024-01-17 14:15          │
│ Change: Engine-001 attached                            │
│ State:  {..., components: [Engine-001], ...}           │
├─────────────────────────────────────────────────────────┤
│ [Seq 5] FILE_UPLOADED @ 2024-01-18 09:45               │
│ Change: spec.pdf uploaded                              │
│ State:  {..., attachments: [spec.pdf], ...}            │
└─────────────────────────────────────────────────────────┘

CURRENT STATE (Replayed):
  ├─ ID: 550e8400-e29b-41d4-a716-446655440000
  ├─ Name: Equipment-001
  ├─ Template: Vehicle
  ├─ Year: 2024 (not 2023)
  ├─ Power: 150 kW (not 140)
  ├─ Components: [Engine-001]
  ├─ Attachments: [spec.pdf]
  └─ Created: 2024-01-15 09:00

POINT-IN-TIME QUERY (What was state on 2024-01-16 10:00?):
  Only events up to that time → Ignore events after
  State would be: {Year: 2023, Power: 140 kW, no components}

AUDIT REPORT (Who changed what?):
  ├─ 2024-01-15 09:00: user-123 created asset
  ├─ 2024-01-16 10:30: user-456 changed year (2023→2024)
  ├─ 2024-01-17 09:45: user-789 changed power (140→150)
  ├─ 2024-01-17 14:15: user-123 attached component
  └─ 2024-01-18 09:45: user-456 uploaded file
```

---

## 6. Event Sourcing vs Traditional Approach

```
┌────────────────────────────────────────────────────────┐
│           TRADITIONAL APPROACH                        │
├────────────────────────────────────────────────────────┤
│ UPDATE ASSET SET year=2024 WHERE id=550e...          │
│           ↓                                            │
│ BEFORE: {year: 2023}                                  │
│ AFTER:  {year: 2024}                                  │
│           ↓ [Old value lost]                           │
│ No audit trail, "who changed it" unclear              │
│           ↓                                            │
│ PROBLEM: Data loss, no compliance trail               │
└────────────────────────────────────────────────────────┘

vs

┌────────────────────────────────────────────────────────┐
│        EVENT SOURCING APPROACH                        │
├────────────────────────────────────────────────────────┤
│ INSERT into ASSET_EVENT_STORE                         │
│   (event_type=ASSET_UPDATED,                          │
│    changes={year: 2023→2024},                         │
│    user_id=user-456,                                  │
│    timestamp=2024-01-16 10:30)                        │
│           ↓                                            │
│ RESULT: Complete history preserved                    │
│  • What changed: year field                           │
│  • Old value: 2023                                    │
│  • New value: 2024                                    │
│  • Who: user-456                                      │
│  • When: 2024-01-16 10:30                             │
│           ↓                                            │
│ BENEFIT: Full audit trail, compliance ready           │
└────────────────────────────────────────────────────────┘
```

---

## 7. Retention & Compliance

```
RETENTION POLICIES
═════════════════════════════════════════════════════════

ASSET_EVENT_STORE Records:
├─ Keep all events indefinitely (immutable)
├─ Indexed by asset_id, date, event_type
├─ Partition by year (for query performance)
└─ Archive to cold storage after 5 years

AUDIT REQUIREMENTS:
├─ SOX (Sarbanes-Oxley): 7 years
├─ HIPAA: 6 years
├─ GDPR: Up to individual (but immutable logs help)
└─ ISO 27001: Demonstrate audit trail

EVENT RETENTION DECISION MATRIX:
┌──────────────┬──────────────┬──────────────┐
│ Event Type   │ Retention    │ Storage      │
├──────────────┼──────────────┼──────────────┤
│ ASSET_*      │ Indefinite   │ Active (1yr) │
│              │              │ + Cold (5yr) │
├──────────────┼──────────────┼──────────────┤
│ FILE_*       │ Per policy   │ Same as file │
│              │ (usually 5yr)│ (auto-delete)│
├──────────────┼──────────────┼──────────────┤
│ DELETE_*     │ Indefinite   │ Active (1yr) │
│ (important)  │              │ + Cold (5yr) │
└──────────────┴──────────────┴──────────────┘

EXAMPLE: "Show me all changes to Asset-001 by user X in 2023"
Query:
  SELECT * FROM ASSET_EVENT_STORE
  WHERE asset_master_id = 'Asset-001'
    AND user_id = 'user-X'
    AND YEAR(event_timestamp) = 2023
  ORDER BY sequence_number ASC

Result: 23 changes found (CREATE, 15 UPDATES, 7 ATTRIBUTE_CHANGED)
```

---

## 8. Event Correlation & Causation

```
RELATED EVENTS & CAUSATION CHAIN
═════════════════════════════════════════════════════════

Single Event Example:
┌─────────────────────────────────┐
│ Event: ASSET_CREATED            │
│ correlation_id: abc123          │
│ causation_id: null              │
└─────────────────────────────────┘
     ↓ [Triggers handlers]
  ┌──────────────────┬──────────────────┬──────────────────┐
  ↓                  ↓                  ↓                  ↓
AssetEventHandler   NotificationHandler DocumentHandler  (others)
     ↓                  ↓                  ↓
  Audit Msg        Send Email         Check Files
     ↓                  ↓                  ↓
[New Event]       [New Event]         [New Event]
correlation_id:   correlation_id:     correlation_id:
abc123            abc123              abc123

All related events linked via correlation_id


Complex Scenario Example:
┌────────────────────────────────────────────────────────┐
│ 1. User uploads file (MASTER event)                   │
│    Event: FILE_UPLOADED                               │
│    correlation_id: xyz789                             │
│    causation_id: null (root event)                    │
└────────────────────────────────────────────────────────┘
         ↓ [Triggers DocumentHandler]
┌────────────────────────────────────────────────────────┐
│ 2. DocumentHandler calls Votiro                       │
│    Event: VOTIRO_SCAN_INITIATED                       │
│    correlation_id: xyz789                             │
│    causation_id: FILE_UPLOADED event id               │
└────────────────────────────────────────────────────────┘
         ↓ [Votiro returns result]
┌────────────────────────────────────────────────────────┐
│ 3. Scan complete, file status updated                 │
│    Event: FILE_SCANNED                                │
│    correlation_id: xyz789                             │
│    causation_id: VOTIRO_SCAN_INITIATED event id       │
└────────────────────────────────────────────────────────┘
         ↓ [Triggers notification]
┌────────────────────────────────────────────────────────┐
│ 4. User notified of result                            │
│    Event: NOTIFICATION_SENT                           │
│    correlation_id: xyz789                             │
│    causation_id: FILE_SCANNED event id                │
└────────────────────────────────────────────────────────┘

ENTIRE CAUSATION CHAIN:
FILE_UPLOADED
  → VOTIRO_SCAN_INITIATED
    → FILE_SCANNED
      → NOTIFICATION_SENT

All events have correlation_id=xyz789 (trace entire flow)
```

---

## 9. Using Event Store for Debugging

```
SCENARIO: "File shows CLEAN but user says it was QUARANTINE earlier"

Investigation Steps:

1. Find all events for this attachment:
   SELECT * FROM ASSET_EVENT_STORE
   WHERE event_payload->>'attachment_id' = '990e8400...'
   ORDER BY sequence_number ASC

   Results:
   ├─ [Seq 10] FILE_UPLOADED @ 09:45
   ├─ [Seq 11] FILE_SCANNED @ 10:00 (QUARANTINE, score=92)
   ├─ [Seq 12] FILE_SCANNED @ 10:15 (CLEAN, score=0) ← Re-scan?
   └─ [Seq 13] NOTIFICATION_SENT @ 10:16

2. What happened?
   ├─ First scan (10:00): Detected malware → QUARANTINE
   ├─ Second scan (10:15): No threats → CLEAN
   └─ User was notified of both? Need to check notifications

3. Why two scans?
   ├─ Check causation chain
   ├─ Admin re-submitted? User complained?
   └─ Auto-retry on file update?

4. Look at MASTER_ATTACHMENT updates:
   SELECT * FROM MASTER_ATTACHMENT
   WHERE id = '990e8400...'

   Current status: CLEAN
   Updated: 2024-01-18 10:15:00
   Threat_score: 0

5. Root cause found:
   └─ Admin manually re-submitted file for scan
   └─ Second scan found no threats
   └─ Event log proves both scans occurred

CONCLUSION: No data inconsistency, events explain state
```

---

## Quick Reference

| Concept | Definition | Purpose |
|---------|-----------|---------|
| **Event Store** | ASSET_EVENT_STORE table | Immutable log of all changes |
| **Event Type** | ASSET_CREATED, FILE_UPLOADED, etc. | Categorizes changes |
| **Event Payload** | JSON with change details | What changed & why |
| **Correlation ID** | Links related events | Trace entire flow |
| **Causation ID** | Links parent event | Event dependency |
| **Sequence Num** | Order of events | Replay correctly |
| **Audit Trail** | All events for asset | Compliance & debugging |

---

## Related Documents

- **Architecture** → [MASTER_VISUAL_GUIDE.md](./MASTER_VISUAL_GUIDE.md#7-event-types--audit-trail)
- **Event Handlers** → [8_HANDLERS_IMPLEMENTATION.md](./8_HANDLERS_IMPLEMENTATION.md)
- **API Reference** → [9_API_ENDPOINTS.md](./9_API_ENDPOINTS.md)

---

**Last Updated:** 2026-05-03 | **Version:** 2.0
