# Asset Master — Data Organization Example with Template Versions

> **Purpose:** Demonstrates how data flows through the normalized ER schema when templates are versioned and assets are reused across versions instead of duplicated. 

---

## Scenario Overview

- **GOS Register:** RTG (Rubber-tired Gantry) at port AG.02.131.100
- **OEM Templates:** Konecranes RTG Electric with 2 versions (V1 → V2)
- **Assets:** Create RTG-100 from V1 in 2022, upgrade to V2 in 2023 without duplicating the asset row

---

## Sample Data Tables

### GOS_REGISTER (Immutable GOS Hierarchy)

**Purpose:** Single source of truth for GOS object classifications. Loaded once from GOS CSV, never duplicated.

| id | parent_id | gos_object_type | gos_object_id | gos_hierarchy_path | display_name | status |
|----|-----------|-----------------|---------------|--------------------|--------------|--------|
| g1 | null | AG | AG | AG | All Gantries | active |
| g2 | g1 | AG | AG.02 | AG.02 | Port 02 Terminal | active |
| g3 | g2 | AG | AG.02.131.100 | AG.02.131.100 | RTG Electric Slot 100 | active |

---

### OEM_TEMPLATE (Manufacturer/Model Catalog)

**Purpose:** Links a GOS entry to an OEM manufacturer and model. One catalog entry per OEM+Model combination.

| id | gos_register_id | template_code | template_name | oem_manufacturer | oem_model | status |
|----|-----------------|---------------|---------------|------------------|-----------|--------|
| t1 | g3 | AG-KONE-RTG | Konecranes RTG Electric | Konecranes | RTG-E | active |

---

### OEM_TEMPLATE_VERSION (Versioned Specifications & Components)

**Purpose:** Capture versioned specs, components, attachments. This is what template inheritance pulls from.

| id | oem_template_id | parent_version_id | version_code | version_number | is_latest_version | effective_from | effective_to | status | specifications |
|----|-----------------|-------------------|--------------|--------|---------|----|----|---------|-----------|
| v1 | t1 | null | V1 | 1 | false | 2022-01-01 | 2023-03-01 | deprecated | `{"firmware":"1.0","cdr_version":"v1","max_payload":"50t"}` |
| v2 | t1 | v1 | V2 | 2 | true | 2023-03-02 | null | active | `{"firmware":"2.0","cdr_version":"v2","max_payload":"55t"}` |

**Timeline:**
- 2022-01-01: V1 published (firmware 1.0, 50t payload)
- 2023-03-01: V1 deprecated (end of support)
- 2023-03-02: V2 released (firmware 2.0, 55t payload, backward compatible)

---

### ASSET_MASTER (Physical Asset Instances)

**Purpose:** Single record per managed asset. Can be bound to multiple template versions over time (reuse without duplication).

| id | gos_register_id | oem_template_version_id | parent_asset_id | asset_code | asset_name | serial_number | creation_mode | status | installation_date |
|----|-----------------|-------------------------|-----------------|------------|------------|---------------|---------------|--------|-------------------|
| a1 | g3 | **v1** (original) | null | AG02-RTG-100 | RTG-100 | KR-RTG-E-12345 | TEMPLATE_BASED | active | 2022-01-15 |

**Key Point:** Asset `a1` row is created **once** from v1 in Jan 2022. When v2 releases in March 2023, we don't create a new asset row; instead, we create a new binding (see next table).

---

### ASSET_TEMPLATE_BINDING (Tracks Which Version an Asset Uses Over Time)

**Purpose:** Junction table that records which template version an asset is bound to and when the binding is active.

| id | asset_master_id | oem_template_version_id | effective_from | effective_to | role | notes |
|----|-----------------|-------------------------|----------------|--------------|------|-------|
| b1 | a1 | v1 | 2022-01-15 | 2023-03-01 | primary | Initial binding on V1 |
| b2 | a1 | v2 | 2023-03-02 | null | primary | Upgraded to V2 on release |

**Timeline:**
- **2022-01-15:** RTG-100 created and bound to v1 (binding b1 active).
- **2023-03-01:** V1 reaches end-of-support; b1 marked `effective_to = 2023-03-01`.
- **2023-03-02:** Auto-rebind job executes; v2 binding b2 created with `effective_from = 2023-03-02`.
- **Today:** RTG-100 (`a1`) is bound to v2 (`b2`, no end date).

---

## How Reuse Works (Step-by-Step)

### Timeline: Asset Creation (2022-01-15)

**User Action:** Create RTG-100 using Konecranes V1 template

```
Input:
  - gos_register_id: g3 (AG.02.131.100)
  - oem_template_version_id: v1
  - asset_code: "AG02-RTG-100"
  - asset_name: "RTG-100"
  - reuse_if_exists: true (policy flag)

Transaction (create-or-reuse):
  1. BEGIN
  2. SELECT id FROM ASSET_MASTER 
     WHERE gos_register_id = 'g3' AND status <> 'retired' 
     LIMIT 1 FOR UPDATE
  3. Found: None (first asset for this GOS)
  4. INSERT ASSET_MASTER(a1, gos_register_id='g3', ...)
  5. INSERT ASSET_TEMPLATE_BINDING(b1, asset_master_id='a1', oem_template_version_id='v1', ...)
  6. COMMIT
  
Result: New asset a1 created with binding b1 to v1
```

---

### Timeline: Template Upgrade (2023-03-02)

**System Action:** V2 published; auto-rebind job runs

```
Input:
  - oem_template_id: t1 (Konecranes RTG Electric)
  - new_version: v2
  - policy: auto_upgrade_all

Transaction (rebind):
  FOR EACH asset WITH gos_register_id = 'g3' AND status <> 'retired':
    1. BEGIN
    2. SELECT binding.id FROM ASSET_TEMPLATE_BINDING binding
       WHERE binding.asset_master_id = 'a1' AND binding.effective_to IS NULL
       FOR UPDATE
    3. Found: b1
    4. UPDATE ASSET_TEMPLATE_BINDING SET effective_to = '2023-03-01'
       WHERE id = 'b1'
    5. INSERT ASSET_TEMPLATE_BINDING(b2, asset_master_id='a1', 
         oem_template_version_id='v2', effective_from='2023-03-02', ...)
    6. COMMIT
    7. LOG event: AssetTemplateUpgraded(asset=a1, from=v1, to=v2)
  END FOR

Result: Asset a1 now bound to v2 without being duplicated
```

---

## Query Examples

### 1. Get Current Template Version for an Asset

```sql
SELECT otv.version_code, otv.specifications, b.effective_from
FROM ASSET_TEMPLATE_BINDING b
JOIN OEM_TEMPLATE_VERSION otv ON b.oem_template_version_id = otv.id
WHERE b.asset_master_id = 'a1' AND b.effective_to IS NULL;

-- Result:
-- version_code: V2
-- specifications: {"firmware":"2.0","cdr_version":"v2","max_payload":"55t"}
-- effective_from: 2023-03-02
```

---

### 2. Get All Versions an Asset Has Used (History)

```sql
SELECT otv.version_code, otv.version_number, b.effective_from, b.effective_to
FROM ASSET_TEMPLATE_BINDING b
JOIN OEM_TEMPLATE_VERSION otv ON b.oem_template_version_id = otv.id
WHERE b.asset_master_id = 'a1'
ORDER BY b.effective_from DESC;

-- Result:
-- V2, 2, 2023-03-02, null
-- V1, 1, 2022-01-15, 2023-03-01
```

---

### 3. Find Asset by GOS Entry (Reuse if Exists)

```sql
-- Check if an asset already exists for GOS g3
SELECT id FROM ASSET_MASTER 
WHERE gos_register_id = 'g3' AND status <> 'retired' 
LIMIT 1;

-- Result: a1 (asset RTG-100 already exists)
-- Action: Create binding to new version instead of creating duplicate asset
```

---

### 4. Bulk Rebind: Upgrade All Assets to Latest Template Version

```sql
BEGIN;

-- Find all assets that should be upgraded
SELECT a.id FROM ASSET_MASTER a
WHERE a.gos_register_id IN (
  SELECT id FROM GOS_REGISTER WHERE gos_hierarchy_path LIKE 'AG.02%'
) AND a.status <> 'retired'
FOR UPDATE;

-- For each asset, close current binding and create new binding to v2:
UPDATE ASSET_TEMPLATE_BINDING 
SET effective_to = CURRENT_DATE
WHERE asset_master_id = 'a1' AND effective_to IS NULL;

INSERT INTO ASSET_TEMPLATE_BINDING(
  asset_master_id, oem_template_version_id, 
  effective_from, role, created_by, created_at
) VALUES(
  'a1', 'v2', CURRENT_DATE + INTERVAL '1 day', 'primary', 'admin', CURRENT_TIMESTAMP
);

COMMIT;
```

---

## Multi-Asset Scenario

If you have **multiple RTGs** for the same GOS entry (e.g., 2 RTGs in slot 100):

### ASSET_MASTER (Multiple Assets, Single GOS Entry)

| id | gos_register_id | oem_template_version_id | asset_code | asset_name | creation_mode |
|----|-----------------|-------------------------|------------|------------|---------------|
| a1 | g3 | v1 | AG02-RTG-100-A | RTG-100-A | TEMPLATE_BASED |
| a2 | g3 | v1 | AG02-RTG-100-B | RTG-100-B | TEMPLATE_BASED |

### ASSET_TEMPLATE_BINDING (Each Asset Has Its Own Timeline)

| id | asset_master_id | oem_template_version_id | effective_from | effective_to |
|----|-----------------|-------------------------|----------------|--------------|
| b1 | a1 | v1 | 2022-01-15 | 2023-03-01 |
| b2 | a1 | v2 | 2023-03-02 | null |
| b3 | a2 | v1 | 2022-02-20 | null |

**Interpretation:**
- **RTG-100-A:** Upgraded from v1 → v2 on 2023-03-02 (has 2 bindings).
- **RTG-100-B:** Still on v1, not upgraded yet (has 1 binding).

You can **selectively upgrade** individual assets or enforce **bulk upgrades** via policy.

---

## Creation Modes

| creation_mode | oem_template_version_id | gos_register_id | Use Case |
|---------------|-------------------------|-----------------|----------|
| TEMPLATE_BASED | **not null** | **not null** | Asset from OEM template, tied to GOS, reusable |
| TEMPLATE_BASED | **not null** | null | Asset from template but ad-hoc, not GOS-backed |
| AD_HOC | null | null | Manually created, no template inheritance |
| AD_HOC | null | **not null** | Ad-hoc asset linked to GOS entry for classification |

---

## Policy Choices

### 1. Unique Asset per GOS Entry?

**Option A: Enforce Unique (Recommended for most cases)**
- Add DB constraint: `UNIQUE(gos_register_id)` where `gos_register_id IS NOT NULL`
- Pro: Simple, enforces reuse, prevents duplicate managed assets
- Con: Multiple RTGs in slot 100 must have different GOS IDs or leave `gos_register_id` null

**Option B: Allow Multiple Assets per GOS**
- No unique constraint; allow 2+ assets to reference same GOS
- Pro: Flexible; multiple RTGs can share same GOS entry
- Con: Reuse policy enforced in application logic, not DB

### 2. Rebinding Strategy

**Option A: Auto-rebind on Version Release**
- Background job automatically upgrades all eligible assets when new version published
- Pro: Fast, consistent
- Con: Less control; may not suit all organizations

**Option B: Manual Rebind**
- Users rebind individual assets via UI
- Pro: Full control, allows phased rollouts
- Con: Operational overhead

**Option C: Approval-based Auto-rebind**
- Auto-rebind scheduled but requires approval before execution
- Pro: Balanced control
- Con: Adds workflow complexity

---

## Audit & Event Trail

When an asset is rebound, capture the change in the event store:

```sql
INSERT INTO ASSET_EVENT_STORE(
  event_id, event_type, source_entity_id, source_entity_type, 
  event_details, created_by, created_at
) VALUES(
  gen_random_uuid(),
  'AssetTemplateUpgraded',
  'a1',
  'ASSET_MASTER',
  jsonb_build_object(
    'from_version', 'v1',
    'to_version', 'v2',
    'binding_id_closed', 'b1',
    'binding_id_opened', 'b2',
    'reason', 'Auto-upgrade on V2 release'
  ),
  'admin',
  CURRENT_TIMESTAMP
);
```

This event is then audited and can trigger notifications to asset owners.

---

## Implementation Checklist

- [ ] Create `GOS_REGISTER`, `OEM_TEMPLATE`, `OEM_TEMPLATE_VERSION` tables
- [ ] Create `ASSET_MASTER` with optional `gos_register_id` FK
- [ ] Create `ASSET_TEMPLATE_BINDING` junction table
- [ ] Add DB unique constraint on `gos_register_id` (if enforcing one asset per GOS)
  ```sql
  ALTER TABLE ASSET_MASTER ADD CONSTRAINT uk_gos_register_id 
    UNIQUE(gos_register_id) WHERE gos_register_id IS NOT NULL;
  ```
- [ ] Add indexes for performance:
  ```sql
  CREATE INDEX idx_asset_master_gos_register_id ON ASSET_MASTER(gos_register_id);
  CREATE INDEX idx_asset_template_binding_asset_effective_to 
    ON ASSET_TEMPLATE_BINDING(asset_master_id, effective_to);
  ```
- [ ] Implement create-or-reuse transaction logic (SELECT ... FOR UPDATE)
- [ ] Implement rebinding logic (close old binding, insert new binding)
- [ ] Add audit event capture on every binding change
- [ ] Test bulk rebind with multiple assets and versions
- [ ] Document reuse policy (unique vs. multiple, auto vs. manual rebind)
- [ ] Create admin dashboard to view asset binding timeline
- [ ] Add approval workflow for manual rebinds (optional)

---

## Summary

**Key Takeaway:** By separating asset creation from template binding, you achieve:
1. **No duplication:** One asset row, many template versions over time
2. **Clear history:** Binding table shows exactly when/how versions changed
3. **Flexible reuse:** Same asset can be upgraded, downgraded, or bound to multiple versions
4. **Scalability:** Bulk upgrades and selective rebinds based on policy
5. **Auditability:** Every binding change logged and traceable
