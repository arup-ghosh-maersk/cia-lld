# Asset Master — Data Organization Example with Template Versions

> **Purpose:** Demonstrates how data flows through the normalized ER schema when templates are versioned and assets are reused across versions instead of duplicated. 

---

## Scenario Overview

- **GOS Register:** RTG (Rubber-tired Gantry) at port AG.02.131.100 — single source of truth, no duplication
- **OEM Templates:** Konecranes RTG Electric with 2 versions (V1 → V2)
- **V1 Scope:** GOS hierarchy up to Level 300 only
- **V2 Scope:** GOS hierarchy extended to Level 400 (new node added)
- **Assets:** RTG-100 from V1 sees only Levels 200–300; RTG-200 from V2 sees Levels 200–400
- **Key:** `TEMPLATE_VERSION_GOS` junction table controls which GOS nodes each version exposes — no GOS rows duplicated

---

## Sample Data Tables

### GOS_REGISTER (Single Source of Truth — Never Duplicated)

> All GOS levels exist here once. No row is added or removed when a new template version is created.

| id  | parent_id | gos_object_id   | gos_hierarchy_path | display_name          | gos_object_level | status |
|-----|-----------|-----------------|--------------------|-----------------------|------------------|--------|
| g1  | null      | AG              | AG                 | All Gantries          | 200              | active |
| g2  | g1        | AG.02           | AG.02              | Port 02 Terminal      | 300              | active |
| g3  | g2        | AG.02.131       | AG.02.131          | RTG Electric Slot     | 400              | active |

> `g3` (Level 400) was **added to GOS_REGISTER when the new level was defined globally**. It exists in the registry for both V1 and V2 timeframes — but only V2 references it via `TEMPLATE_VERSION_GOS`.

---

### OEM_TEMPLATE

| id  | gos_register_id | template_code   | oem_manufacturer | oem_model | status |
|-----|-----------------|----------------|------------------|-----------|--------|
| t1  | g1              | AG-KONE-RTG    | Konecranes       | RTG-E     | active |

> Template anchors to Level 200 (root GOS node). Versions control deeper scope.

---

### OEM_TEMPLATE_VERSION

| id  | oem_template_id | parent_version_id | version_code | is_latest_version | effective_from | effective_to | status     |
|-----|-----------------|-------------------|--------------|-------------------|----------------|--------------|------------|
| v1  | t1              | null              | V1           | false             | 2022-01-01     | 2023-03-01   | deprecated |
| v2  | t1              | v1                | V2           | true              | 2023-03-02     | null         | active     |

---

### TEMPLATE_VERSION_GOS (Junction — Controls Which GOS Nodes Each Version Exposes)

> This is the key table. V1 only maps to g1, g2 (Levels 200 & 300). V2 additionally maps to g3 (Level 400). GOS_REGISTER itself is unchanged.

| id    | oem_template_version_id | gos_register_id | is_root_node | is_visible | inclusion_reason |
|-------|------------------------|-----------------|--------------|------------|------------------|
| tvg-1 | v1                     | g1              | true         | true       | inherited        |
| tvg-2 | v1                     | g2              | false        | true       | inherited        |
| tvg-3 | v2                     | g1              | true         | true       | inherited        |
| tvg-4 | v2                     | g2              | false        | true       | inherited        |
| tvg-5 | v2                     | g3              | false        | true       | added            |

**What this means:**
- **V1** → only `g1` (Level 200) and `g2` (Level 300) are in scope. Level 400 (`g3`) is NOT visible.
- **V2** → `g1`, `g2`, and `g3` (Level 400) are all in scope. Level 400 IS visible.
- `g3` is never duplicated — it exists once in `GOS_REGISTER`, but only `v2` references it in `TEMPLATE_VERSION_GOS`.

---

### ASSET_MASTER

| id  | gos_register_id | oem_template_version_id | asset_code | asset_name | creation_mode  | status |
|-----|-----------------|------------------------|------------|------------|----------------|--------|
| a1  | g1              | v1                     | RTG-100    | RTG Unit 1 | TEMPLATE_BASED | active |
| a2  | g1              | v2                     | RTG-200    | RTG Unit 2 | TEMPLATE_BASED | active |

---

### What Each Asset Sees in the UI (GOS Hierarchy Traversal)

| Asset | Template Version | GOS Levels Visible | Level 400 Visible? |
|-------|-----------------|--------------------|--------------------|
| RTG-100 (a1) | V1 | 200 → 300 | No |
| RTG-200 (a2) | V2 | 200 → 300 → 400 | Yes |

> The UI queries `TEMPLATE_VERSION_GOS` filtered by the asset's `oem_template_version_id` to build the hierarchy tree. This ensures each asset only sees the GOS levels its template version declared.

---

### SQL: Get GOS Hierarchy for a Specific Asset

```sql
-- Get all GOS nodes visible for asset RTG-100 (bound to V1)
SELECT gr.gos_object_id, gr.display_name, gr.gos_object_level, tvg.is_visible
FROM TEMPLATE_VERSION_GOS tvg
JOIN GOS_REGISTER gr ON tvg.gos_register_id = gr.id
JOIN ASSET_MASTER am ON am.oem_template_version_id = tvg.oem_template_version_id
WHERE am.id = 'a1'
  AND tvg.is_visible = true
ORDER BY gr.gos_object_level;

-- Result for RTG-100 (V1):
-- AG      | All Gantries       | 200
-- AG.02   | Port 02 Terminal   | 300
-- (AG.02.131 / Level 400 NOT returned)

-- Result for RTG-200 (V2):
-- AG          | All Gantries       | 200
-- AG.02       | Port 02 Terminal   | 300
-- AG.02.131   | RTG Electric Slot  | 400
```

---

### SQL: Create V2 — Copy V1 GOS Scope + Add New Level 400 Node

```sql
BEGIN;

-- 1. Create the new template version
INSERT INTO OEM_TEMPLATE_VERSION (id, oem_template_id, parent_version_id, version_code, version_number, is_latest_version, effective_from, status)
VALUES ('v2', 't1', 'v1', 'V2', 2, true, '2023-03-02', 'active');

-- 2. Deprecate V1
UPDATE OEM_TEMPLATE_VERSION SET is_latest_version = false, effective_to = '2023-03-01', status = 'deprecated'
WHERE id = 'v1';

-- 3. Copy all V1 GOS scope into V2 (inherit existing nodes)
INSERT INTO TEMPLATE_VERSION_GOS (id, oem_template_version_id, gos_register_id, is_root_node, is_visible, inclusion_reason)
SELECT gen_random_uuid(), 'v2', gos_register_id, is_root_node, is_visible, 'inherited'
FROM TEMPLATE_VERSION_GOS
WHERE oem_template_version_id = 'v1';

-- 4. Add the new Level 400 GOS node to V2 only
INSERT INTO TEMPLATE_VERSION_GOS (id, oem_template_version_id, gos_register_id, is_root_node, is_visible, inclusion_reason)
VALUES (gen_random_uuid(), 'v2', 'g3', false, true, 'added');

COMMIT;
```



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
