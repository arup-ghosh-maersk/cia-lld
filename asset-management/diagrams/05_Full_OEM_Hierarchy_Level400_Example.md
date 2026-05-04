# Full OEM Hierarchy with Level 400 Modification Example

> **Scenario:** Konecranes RTG (Rubber-Tired Gantry) — Version V1 → V2 with Level 400 component hierarchy change

---

## 1. Complete GOS Hierarchy — RTG Example

### GOS_REGISTER Hierarchy Tree

```
GOS_REGISTER (Level 1 = 200)
├── gos_object_type: AG
├── gos_lv1_code: AG
├── gos_object_level: 200
├── gos_hierarchy_level: 1
├── display_name: "Automated Guided Vehicles & Equipment"
├── status: active
│
└── CHILD: GOS_REGISTER (Level 2 = 300)
    ├── parent_id: AG (Level 1)
    ├── gos_object_type: AG
    ├── gos_lv1_code: AG
    ├── gos_lv2_code: 02
    ├── gos_object_level: 300
    ├── gos_hierarchy_level: 2
    ├── display_name: "Automated Equipment — Material Handling"
    │
    └── CHILD: GOS_REGISTER (Level 3 = 400)
        ├── parent_id: AG.02 (Level 2)
        ├── gos_object_type: AG
        ├── gos_lv1_code: AG
        ├── gos_lv2_code: 02
        ├── gos_lv3_code: 131
        ├── gos_object_level: 400
        ├── gos_hierarchy_level: 3
        ├── display_name: "RTG — Rubber-Tired Gantry Cranes"
        ├── gos_category: EQ
        ├── gos_functional_type: Functional
        ├── gos_equipment_type: Rotary
        │
        ├── CHILD: GOS_REGISTER (Level 4 = 500) — MOTOR/ENGINE
        │   ├── parent_id: AG.02.131 (Level 3)
        │   ├── gos_lv4_code: 100
        │   ├── gos_object_level: 500
        │   ├── gos_hierarchy_level: 4
        │   ├── display_name: "Electrical Motor — Main Drive"
        │   ├── gos_category: EQ
        │   ├── gos_functional_type: Functional
        │   ├── gos_equipment_type: Electrical
        │   ├── gos_is_tool: false
        │   │
        │   └── CHILD: GOS_REGISTER (Level 5 = 600) — MOTOR BEARING
        │       ├── parent_id: AG.02.131.100 (Level 4)
        │       ├── gos_lv5_code: P001
        │       ├── gos_object_level: 600
        │       ├── gos_hierarchy_level: 5
        │       ├── display_name: "Bearing Assembly — Main Motor"
        │       └── gos_is_tool: false
        │
        ├── CHILD: GOS_REGISTER (Level 4 = 500) — HYDRAULIC SYSTEM
        │   ├── parent_id: AG.02.131 (Level 3)
        │   ├── gos_lv4_code: 200
        │   ├── gos_object_level: 500
        │   ├── gos_hierarchy_level: 4
        │   ├── display_name: "Hydraulic System — Hoist Control"
        │   ├── gos_equipment_type: Hydraulic
        │   │
        │   └── CHILD: GOS_REGISTER (Level 5 = 600) — HYDRAULIC PUMP
        │       ├── parent_id: AG.02.131.200 (Level 4)
        │       ├── gos_lv5_code: P002
        │       ├── gos_object_level: 600
        │       └── display_name: "Hydraulic Pump Assembly"
        │
        └── CHILD: GOS_REGISTER (Level 4 = 500) — CONTROL SYSTEM
            ├── parent_id: AG.02.131 (Level 3)
            ├── gos_lv4_code: 230
            ├── gos_object_level: 500
            ├── gos_hierarchy_level: 4
            ├── display_name: "PLC Control System — RTG Management"
            ├── gos_equipment_type: Electrical
            │
            └── CHILD: GOS_REGISTER (Level 5 = 600) — CONTROL MODULE
                ├── parent_id: AG.02.131.230 (Level 4)
                ├── gos_lv5_code: P003
                ├── gos_object_level: 600
                └── display_name: "PLC Module — Safety & Logic"
```

---

## 2. OEM_TEMPLATE Hierarchy

```
OEM_TEMPLATE (Top-Level RTG)
├── id: oem_tmp_001
├── gos_register_id: AG.02.131 (Level 3)
├── template_code: AG-KONECRANES-RTG-001
├── template_name: "Konecranes RTG Electric — Standard Model"
├── oem_manufacturer: "Konecranes"
├── oem_model: "RTG-E Series"
├── status: active
│
└── OEM_TEMPLATE_VERSION (V1)
    ├── id: oem_tmpl_v_001
    ├── version_code: V1
    ├── version_number: 1
    ├── is_latest_version: false
    ├── effective_from: 2024-01-15
    ├── effective_to: 2025-06-30
    ├── status: active → deprecated
    │
    ├── TEMPLATE_COMPONENT (Motor — V1)
    │   ├── gos_register_id: AG.02.131.100 (Level 4 — Electrical Motor)
    │   ├── component_name: "Main Motor — ABB IE3 75kW"
    │   ├── component_model: "ABB M3BP250S-4"
    │   ├── quantity: 1
    │   ├── is_mandatory: true
    │   │
    │   └── SPECIFICATION (V1)
    │       ├── power_rating: "75 kW"
    │       ├── voltage: "400V 3-phase"
    │       ├── speed: "1500 rpm"
    │       ├── efficiency: "IE3 (95.0%)"
    │       ├── bearing_type: "Deep groove ball bearing (OLD SPEC)"
    │       └── notes: "Standard industrial motor"
    │
    ├── TEMPLATE_COMPONENT (Hydraulic System — V1)
    │   ├── gos_register_id: AG.02.131.200 (Level 4 — Hydraulic System)
    │   ├── component_name: "Hydraulic Pump — Parker"
    │   ├── component_model: "Parker PV063"
    │   ├── quantity: 1
    │   ├── is_mandatory: true
    │   │
    │   └── SPECIFICATION (V1)
    │       ├── pump_type: "Variable Displacement Piston Pump"
    │       ├── displacement: "63 cc/rev"
    │       ├── max_pressure: "280 bar"
    │       └── fluid_type: "ISO VG 46 Mineral Oil"
    │
    └── TEMPLATE_COMPONENT (Control System — V1)
        ├── gos_register_id: AG.02.131.230 (Level 4 — Control System)
        ├── component_name: "PLC Controller — Siemens S7-1200"
        ├── component_model: "Siemens 6ES7214-1AG40-0XB0"
        ├── quantity: 1
        ├── is_mandatory: true
        │
        └── SPECIFICATION (V1)
            ├── processor: "1.5 MHz"
            ├── memory: "50 KB program + 50 KB data"
            ├── io_modules: "2 (8DI, 6DO)"
            └── safety_rating: "Cat 2 PLd"

└── OEM_TEMPLATE_VERSION (V2) — ⭐ LEVEL 400 MODIFICATION
    ├── id: oem_tmpl_v_002
    ├── parent_version_id: oem_tmpl_v_001
    ├── version_code: V2
    ├── version_number: 2
    ├── is_latest_version: true
    ├── effective_from: 2025-07-01
    ├── effective_to: null
    ├── status: active
    ├── change_summary: "Major upgrade: Motor bearing specification, Hydraulic pump efficiency, Control system capacity"
    │
    ├── TEMPLATE_COMPONENT (Motor — V2) ⭐ CHANGED
    │   ├── gos_register_id: AG.02.131.100 (Level 4 — Same motor, different specs!)
    │   ├── component_name: "Main Motor — ABB IE3 75kW + Ceramic Bearing"
    │   ├── component_model: "ABB M3BP250S-4" (same model code)
    │   ├── quantity: 1
    │   ├── is_mandatory: true
    │   ├── change_indicator: "🔄 UPGRADED"
    │   │
    │   └── SPECIFICATION (V2) ⭐ KEY CHANGE
    │       ├── power_rating: "75 kW" (same)
    │       ├── voltage: "400V 3-phase" (same)
    │       ├── speed: "1500 rpm" (same)
    │       ├── efficiency: "IE3 (95.0%)" (same)
    │       ├── bearing_type: "HYBRID CERAMIC BALL BEARING (NEW!)" ⭐
    │       │   ├── old_spec: "Deep groove ball bearing"
    │       │   ├── new_spec: "Si₃N₄ ceramic rolling elements"
    │       │   ├── benefit: "30% longer service life, lower friction"
    │       │   └── cost_delta: "+€1,200"
    │       └── notes: "Enhanced durability for high-cycle operations"
    │
    ├── TEMPLATE_COMPONENT (Hydraulic System — V2) ⭐ CHANGED
    │   ├── gos_register_id: AG.02.131.200 (Level 4 — Same, different specs)
    │   ├── component_name: "Hydraulic Pump — Parker + EcoMode"
    │   ├── component_model: "Parker PV063 e-Plus" ⭐ NEW MODEL
    │   ├── quantity: 1
    │   ├── is_mandatory: true
    │   ├── change_indicator: "🔄 UPGRADED"
    │   │
    │   └── SPECIFICATION (V2) ⭐ KEY CHANGE
    │       ├── pump_type: "Variable Displacement Piston Pump + Electronic Pressure Compensator"
    │       ├── displacement: "63 cc/rev" (same)
    │       ├── max_pressure: "280 bar" (same)
    │       ├── fluid_type: "ISO VG 46 Mineral Oil OR Biodegradable HETG" ⭐
    │       ├── efficiency_gain: "12% improvement in full-load cycle"
    │       ├── eco_mode: "NEW: Reduces standby pressure to 20 bar"
    │       └── cost_delta: "+€3,500"
    │
    └── TEMPLATE_COMPONENT (Control System — V2) ⭐ UPGRADED
        ├── gos_register_id: AG.02.131.230 (Level 4 — Same component family, newer generation)
        ├── component_name: "PLC Controller — Siemens S7-1200F (Safety-Rated)"
        ├── component_model: "Siemens 6ES7214-1AG40-0XB0-SAFE" ⭐ NEW MODEL CODE
        ├── quantity: 1
        ├── is_mandatory: true
        ├── change_indicator: "🔄 UPGRADED"
        │
        └── SPECIFICATION (V2) ⭐ KEY CHANGE
            ├── processor: "2.2 MHz" ⭐ (faster)
            ├── memory: "100 KB program + 100 KB data" ⭐ (doubled)
            ├── io_modules: "4 (16DI, 12DO)" ⭐ (expanded)
            ├── safety_rating: "Cat 3 PLe" ⭐ (improved from Cat 2 PLd)
            └── cost_delta: "+€2,800"
```

---

## 3. ASSET_MASTER Instance Data — RTG #001

### RTG #001 — Created with V1 Template (2024-02-20)

```sql
INSERT INTO ASSET_MASTER (
    id, oem_template_version_id, parent_asset_id, asset_hierarchy_level,
    asset_code, asset_name, asset_type, serial_number, manufacturer, model,
    location, owner_user_id, status, creation_mode, installation_date,
    commissioning_date, acquisition_cost, acquisition_currency, criticality_level
) VALUES (
    'asset_001', 'oem_tmpl_v_001', NULL, 1,
    'RTG-MAASMECH-001', 'RTG Electric #001 — Maas Mechanical Terminal',
    'Rubber-Tired Gantry', 'SN-ABB-M3BP-001', 'Konecranes', 'RTG-E Series',
    'Port of Rotterdam — Bay 5', 'ops_manager_001', 'active',
    'TEMPLATE_BASED', '2024-02-20', '2024-03-15', 425000.00, 'EUR', 'critical'
);
```

### RTG #001 — Component Hierarchy (V1 @ Creation)

```
ASSET_MASTER: RTG-MAASMECH-001 (Level 1 — Top-level equipment)
asset_id: asset_001
hierarchy_level: 1
created_from_template: oem_tmpl_v_001 (V1)

├── MASTER_COMPONENT: Main Motor (Level 2)
│   ├── asset_master_id: asset_001
│   ├── component_id: comp_motor_abb_m3bp_v1
│   ├── template_component_id: temp_comp_motor_v1
│   ├── source: "template"
│   ├── status: "active"
│   │
│   └── MASTER_ATTRIBUTE: Motor Bearing Specification (INHERITED FROM V1)
│       ├── value: "Deep groove ball bearing"
│       ├── source: "template"
│       ├── serial_number: "BRG-SKF-6308-22K"
│       └── installation_date: "2024-02-20"
│
├── MASTER_COMPONENT: Hydraulic System (Level 2)
│   ├── asset_master_id: asset_001
│   ├── component_id: comp_hydraulic_parker_pv063_v1
│   ├── template_component_id: temp_comp_hydraulic_v1
│   ├── source: "template"
│   ├── status: "active"
│   │
│   └── MASTER_ATTRIBUTE: Hydraulic Pump Type (INHERITED FROM V1)
│       ├── value: "Parker PV063 — Variable Displacement Piston Pump"
│       ├── source: "template"
│       └── fluid_type: "ISO VG 46 Mineral Oil"
│
└── MASTER_COMPONENT: Control System (Level 2)
    ├── asset_master_id: asset_001
    ├── component_id: comp_plc_siemens_s71200_v1
    ├── template_component_id: temp_comp_control_v1
    ├── source: "template"
    ├── status: "active"
    │
    └── MASTER_ATTRIBUTE: PLC Controller Type (INHERITED FROM V1)
        ├── value: "Siemens S7-1200 — 6ES7214-1AG40-0XB0"
        ├── memory: "50 KB program + 50 KB data"
        └── safety_rating: "Cat 2 PLd"
```

---

## 4. Version Upgrade Scenario: V1 → V2 (Level 400 Modifications)

### Timeline

| Date | Event | Action |
|------|-------|--------|
| 2024-02-20 | RTG #001 created | Binds to `oem_tmpl_v_001` |
| 2025-06-30 | V1 effective_to expires | V1 no longer active for **new** assets |
| 2025-07-01 | V2 becomes effective | `is_latest_version = true` |
| 2025-07-15 | Upgrade decision made | RTG #001 scheduled for upgrade |
| 2025-08-01 | **Manual Upgrade Executed** | Components physically replaced & DB updated |

---

### Step 1: V2 Released (2025-07-01)

**New assets created after this date automatically use V2 specs.**

```sql
-- RTG #002 created with V2 (automatically gets new motor bearing, pump model, PLC)
INSERT INTO ASSET_MASTER (
    id, oem_template_version_id, parent_asset_id, asset_hierarchy_level,
    asset_code, asset_name, status, creation_mode, installation_date
) VALUES (
    'asset_002', 'oem_tmpl_v_002', NULL, 1,
    'RTG-ANTWERPPORT-002', 'RTG Electric #002 — Antwerp Port',
    'active', 'TEMPLATE_BASED', '2025-07-15'
    -- Automatically inherits Level 400 changes: ceramic bearing, EcoMode pump, Cat 3 PLC
);
```

---

### Step 2: Upgrade Decision for RTG #001 (2025-07-15)

**Existing asset RTG #001 still bound to V1. Manual policy decision:**

**Option A: SELECTIVE COMPONENT UPGRADE**
```
🔄 Upgrade ONLY Level 400 components that have breaking changes
├── Motor bearing: UPGRADE (ceramic bearing, better longevity)
├── Hydraulic pump: UPGRADE (EcoMode efficiency)
└── Control system: REVIEW (new safety features, but works with old system)
```

**Option B: FULL VERSION BINDING**
```
🔄 Rebind entire asset to V2, triggering complete component replacement
```

### Step 3: Selective Component Upgrade (CHOSEN)

**RTG #001 gets Level 400 motor bearing upgrade (physical replacement on 2025-08-01)**

```sql
-- BEFORE: V1 Motor Bearing (still active)
SELECT * FROM MASTER_ATTRIBUTE
WHERE asset_master_id = 'asset_001'
  AND attribute_key = 'motor_bearing_type'
  AND status = 'active';

-- Result:
-- | id | asset_id | value | source | status | installed_at |
-- | ma_001 | asset_001 | Deep groove ball bearing | template | active | 2024-02-20 |


-- Step 3A: Archive old V1 bearing specification
UPDATE MASTER_ATTRIBUTE
SET status = 'archived',
    removed_date = '2025-08-01',
    removal_reason = 'Upgraded to ceramic hybrid bearing per V2 spec'
WHERE id = 'ma_001';

-- Step 3B: Create new V2 bearing specification
INSERT INTO MASTER_ATTRIBUTE (
    id, asset_master_id, attribute_def_id, template_attribute_id,
    source, value_string, status, installed_at, installed_by, notes
) VALUES (
    'ma_002', 'asset_001', 'attr_motor_bearing', 'temp_attr_bearing_v2',
    'template_v2_upgrade', 'Hybrid Ceramic Ball Bearing (Si₃N₄)', 'active',
    '2025-08-01', 'maint_tech_042', 'Upgraded from deep groove to ceramic bearing'
);

-- Step 3C: Update MASTER_COMPONENT to reference V2 spec
UPDATE MASTER_COMPONENT
SET status = 'replaced',
    removal_date = '2025-08-01',
    removal_reason = 'Replaced with upgraded bearing assembly'
WHERE asset_master_id = 'asset_001'
  AND component_id = 'comp_motor_bearing';

INSERT INTO MASTER_COMPONENT (
    id, asset_master_id, component_id, template_component_id,
    source, status, installation_date, serial_number, configuration_applied
) VALUES (
    'master_comp_motor_bearing_v2', 'asset_001', 'comp_motor_bearing_ceramic_v2',
    'temp_comp_motor_bearing_v2', 'template_v2_upgrade', 'active',
    '2025-08-01', 'BRG-CERAMIC-SKF-001', '{"bearing_type": "Si3N4 Hybrid", "lubrication": "synthetic"}'
);
```

---

### Step 4: Create ASSET_TEMPLATE_BINDING Records

**Track which asset uses which template version over time:**

```sql
-- V1 Binding (original)
INSERT INTO ASSET_TEMPLATE_BINDING (
    id, asset_master_id, oem_template_version_id,
    binding_type, effective_from, effective_to,
    binding_reason, notes
) VALUES (
    'atb_001', 'asset_001', 'oem_tmpl_v_001',
    'ORIGINAL', '2024-02-20', '2025-07-31',
    'Initial template-based creation', 'RTG created under V1 specs'
);

-- V2 Partial Binding (selective components upgraded)
INSERT INTO ASSET_TEMPLATE_BINDING (
    id, asset_master_id, oem_template_version_id,
    binding_type, effective_from, effective_to,
    binding_reason, notes, upgrade_scope
) VALUES (
    'atb_002', 'asset_001', 'oem_tmpl_v_002',
    'PARTIAL_UPGRADE', '2025-08-01', NULL,
    'Selective L400 component upgrade', 
    'Motor bearing + hydraulic pump upgraded. Control system deferred.',
    '{"upgraded_components": ["motor_bearing", "hydraulic_pump"], "deferred": ["control_system"]}'
);
```

---

### Step 5: Asset Hierarchy State Comparison

#### RTG #001 BEFORE Upgrade (2025-07-31)
```
ASSET_MASTER: RTG-MAASMECH-001
├── MASTER_COMPONENT: Motor (Level 2)
│   ├── Bearing Type: "Deep groove ball bearing" [V1] ← OLD
│   └── Serial: BRG-SKF-6308-22K
│
├── MASTER_COMPONENT: Hydraulic System (Level 2)
│   ├── Pump Model: "Parker PV063" [V1] ← OLD
│   └── Fluid: ISO VG 46 Mineral Oil
│
└── MASTER_COMPONENT: Control System (Level 2)
    ├── PLC: "Siemens S7-1200" [V1] ← OLD
    └── Safety Rating: Cat 2 PLd
```

#### RTG #001 AFTER Selective Upgrade (2025-08-01)
```
ASSET_MASTER: RTG-MAASMECH-001
├── MASTER_COMPONENT: Motor (Level 2)
│   ├── Bearing Type: "Hybrid Ceramic Ball Bearing (Si₃N₄)" [V2] ✅ UPGRADED
│   └── Serial: BRG-CERAMIC-SKF-001
│   └── Notes: "Replaced 2025-08-01 for durability improvement"
│
├── MASTER_COMPONENT: Hydraulic System (Level 2)
│   ├── Pump Model: "Parker PV063 e-Plus" [V2] ✅ UPGRADED
│   └── Fluid: ISO VG 46 Mineral Oil OR Biodegradable HETG
│   └── EcoMode: ENABLED (new feature)
│
└── MASTER_COMPONENT: Control System (Level 2)
    ├── PLC: "Siemens S7-1200" [V1] ⏸️  DEFERRED
    └── Safety Rating: Cat 2 PLd (will upgrade in 2026)
```

---

## 5. Query Examples: Version History & Hierarchy Changes

### Query 1: Show All Templates Versions for RTG

```sql
SELECT 
    v.version_code,
    v.version_number,
    v.effective_from,
    v.effective_to,
    v.status,
    v.change_summary
FROM oem_template_version v
JOIN oem_template t ON v.oem_template_id = t.id
WHERE t.template_code = 'AG-KONECRANES-RTG-001'
ORDER BY v.version_number ASC;

-- Result:
-- | version_code | version_number | effective_from | effective_to | status     | change_summary                           |
-- | V1           | 1              | 2024-01-15     | 2025-06-30   | deprecated | Initial production release               |
-- | V2           | 2              | 2025-07-01     | null         | active     | Major upgrade: Motor bearing, Pump, PLC  |
```

### Query 2: Show All Assets Bound to Each Version

```sql
SELECT 
    ab.binding_type,
    COUNT(DISTINCT ab.asset_master_id) as asset_count,
    ab.oem_template_version_id,
    tv.version_code,
    ab.effective_from,
    ab.effective_to
FROM asset_template_binding ab
JOIN oem_template_version tv ON ab.oem_template_version_id = tv.id
WHERE tv.oem_template_id = (
    SELECT id FROM oem_template WHERE template_code = 'AG-KONECRANES-RTG-001'
)
GROUP BY ab.oem_template_version_id, ab.binding_type, ab.effective_from, ab.effective_to, tv.version_code
ORDER BY ab.effective_from ASC;

-- Result:
-- | binding_type       | asset_count | version_code | effective_from | effective_to |
-- | ORIGINAL           | 5           | V1           | 2024-02-20     | 2025-07-31   |
-- | ORIGINAL           | 3           | V1           | 2024-06-10     | 2025-07-31   |
-- | ORIGINAL           | 2           | V1           | 2025-01-05     | 2025-07-31   |
-- | FULL_UPGRADE       | 2           | V2           | 2025-08-01     | null         |
-- | PARTIAL_UPGRADE    | 5           | V2           | 2025-08-01     | null         |
-- | ORIGINAL           | 3           | V2           | 2025-07-15     | null         |
```

### Query 3: Level 400 Component Changes from V1 → V2

```sql
SELECT 
    c.component_name,
    c.component_model as v1_model,
    c2.component_model as v2_model,
    tc_v1.quantity as v1_qty,
    tc_v2.quantity as v2_qty,
    tv_v2.change_summary
FROM template_component tc_v1
JOIN component c ON tc_v1.component_id = c.id
JOIN oem_template_version tv_v1 ON tc_v1.oem_template_version_id = tv_v1.id
LEFT JOIN template_component tc_v2 ON c.component_name = (
    SELECT DISTINCT component_name 
    FROM component WHERE id = tc_v2.component_id
) AND tc_v2.oem_template_version_id = (
    SELECT id FROM oem_template_version WHERE version_number = 2
)
LEFT JOIN component c2 ON tc_v2.component_id = c2.id
LEFT JOIN oem_template_version tv_v2 ON tc_v2.oem_template_version_id = tv_v2.id
WHERE tv_v1.version_number = 1
  AND tv_v1.oem_template_id = (
    SELECT id FROM oem_template WHERE template_code = 'AG-KONECRANES-RTG-001'
  )
ORDER BY c.component_name ASC;

-- Result:
-- | component_name                       | v1_model              | v2_model                   | v1_qty | v2_qty | change_summary      |
-- | Main Motor — ABB IE3 75kW             | ABB M3BP250S-4        | ABB M3BP250S-4             | 1      | 1      | Bearing type changed |
-- | Hydraulic System — Hoist Control     | Parker PV063          | Parker PV063 e-Plus ✅     | 1      | 1      | Pump model upgraded  |
-- | PLC Control System — RTG Management  | Siemens 6ES7214-...   | Siemens 6ES7214-...-SAFE ✅ | 1      | 1      | Safety-rated version |
```

### Query 4: RTG #001 Component History (Before/After Upgrade)

```sql
-- Show all component versions for asset RTG-MAASMECH-001
SELECT 
    mc.id as master_component_id,
    c.component_name,
    c.component_model,
    mc.status,
    mc.installation_date,
    mc.removal_date,
    CASE 
        WHEN mc.removal_date IS NULL THEN 'ACTIVE'
        WHEN mc.removal_date >= '2025-08-01' THEN 'REPLACED (V2 Upgrade)'
        ELSE 'ARCHIVED'
    END as component_status_label
FROM master_component mc
JOIN component c ON mc.component_id = c.id
WHERE mc.asset_master_id = 'asset_001' -- RTG #001
ORDER BY mc.installation_date ASC;

-- Result:
-- | master_component_id | component_name                    | component_model       | status   | installation_date | removal_date | component_status_label     |
-- | mcomp_001           | Main Motor — ABB IE3 75kW        | ABB M3BP250S-4        | replaced | 2024-02-20        | 2025-08-01   | REPLACED (V2 Upgrade)      |
-- | mcomp_bearing_v2    | Motor Bearing (Ceramic Hybrid)   | Si3N4 Bearing Assy    | active   | 2025-08-01        | null         | ACTIVE                     |
-- | mcomp_002           | Hydraulic System                 | Parker PV063          | replaced | 2024-02-20        | 2025-08-01   | REPLACED (V2 Upgrade)      |
-- | mcomp_hydraulic_v2  | Hydraulic Pump (e-Plus)          | Parker PV063 e-Plus   | active   | 2025-08-01        | null         | ACTIVE                     |
-- | mcomp_003           | PLC Control System               | Siemens 6ES7214-...   | active   | 2024-02-20        | null         | ACTIVE                     |
```

### Query 5: Audit Trail — RTG #001 Upgrade Events

```sql
SELECT 
    ae.event_type,
    ae.event_id,
    ae.source_entity_type,
    aa.action,
    aa.old_values,
    aa.new_values,
    aa.user_id,
    aa.created_at
FROM asset_event_store ae
LEFT JOIN asset_audit aa ON ae.event_id = aa.event_id
WHERE ae.source_entity_id = 'asset_001'
  AND ae.created_at >= '2025-08-01'
ORDER BY ae.created_at ASC;

-- Result:
-- | event_type              | source_entity_type | action | old_values                        | new_values                              | user_id        | created_at          |
-- | ComponentReplaced       | MASTER_COMPONENT   | UPDATE | status: active                    | status: replaced, removal_date: 2025-08-01 | maint_tech_042 | 2025-08-01 09:15:00 |
-- | ComponentInstalled      | MASTER_COMPONENT   | CREATE | null                              | component_id: ceramic_bearing_v2, status: active | maint_tech_042 | 2025-08-01 09:30:00 |
-- | AttributeUpdated        | MASTER_ATTRIBUTE   | UPDATE | value: "Deep groove", status: active | value: "Ceramic Hybrid", status: archived | maint_tech_042 | 2025-08-01 09:30:00 |
-- | VersionBindingCreated   | ASSET_TEMPLATE_BINDING | CREATE | null                              | binding_type: PARTIAL_UPGRADE, effective_from: 2025-08-01 | maint_tech_042 | 2025-08-01 10:00:00 |
```

---

## 6. Visual Summary: Asset Lifecycle Through Version Changes

```
Timeline: RTG #001 Lifecycle

2024-01-15: V1 Released
           └─→ OEM_TEMPLATE_VERSION = oem_tmpl_v_001 (effective_from: 2024-01-15)

2024-02-20: RTG #001 Created
           └─→ ASSET_MASTER.oem_template_version_id = oem_tmpl_v_001
           └─→ ASSET_TEMPLATE_BINDING: ORIGINAL binding (V1)
           └─→ Components: Motor (SKF 6308-22K), Pump (PV063), PLC (S7-1200 Cat 2)

2025-06-30: V1 Effective_to Expires
           └─→ V1 no longer used for new asset creation
           └─→ Existing RTG #001 still operates on V1 specs

2025-07-01: V2 Released ⭐
           └─→ OEM_TEMPLATE_VERSION = oem_tmpl_v_002 (effective_from: 2025-07-01)
           └─→ LEVEL 400 CHANGES:
               • Motor: Deep groove bearing → Ceramic hybrid bearing (+€1,200)
               • Pump: PV063 → PV063 e-Plus with EcoMode (+€3,500)
               • Control: S7-1200 → S7-1200F Safety-rated (+€2,800)

2025-07-15: New RTG #002 Created with V2
           └─→ Automatically gets ceramic bearing, EcoMode pump, safety-rated PLC

2025-08-01: RTG #001 Selective Upgrade ⭐⭐ (MANUAL POLICY DECISION)
           └─→ MASTER_COMPONENT replacements:
               ✅ Motor bearing: Deep groove → Ceramic hybrid (installed)
               ✅ Hydraulic pump: PV063 → PV063 e-Plus (installed)
               ⏸️  Control system: Keep S7-1200 (deferred to 2026)
           └─→ ASSET_TEMPLATE_BINDING: PARTIAL_UPGRADE binding (V2)
           └─→ ASSET_AUDIT: 3 component changes, 1 attribute update, 1 binding creation

2026-Q1: RTG #001 Control System Upgrade (Planned)
           └─→ Replace S7-1200 with S7-1200F Safety-rated
           └─→ Update ASSET_TEMPLATE_BINDING to FULL_UPGRADE

Queries:
────────
Current State:        SELECT * FROM MASTER_COMPONENT WHERE asset_master_id = 'asset_001'
                     → Shows ceramic bearing + EcoMode pump + old PLC

Version History:      SELECT * FROM ASSET_TEMPLATE_BINDING WHERE asset_master_id = 'asset_001'
                     → Shows V1 (original), V2 (partial upgrade from 2025-08-01)

Changes 2025-08-01:   SELECT * FROM ASSET_AUDIT WHERE entity_id = 'asset_001' AND created_at >= '2025-08-01'
                     → Shows component replacements, attribute updates, binding creation
```

---

## 7. Implementation Checklist for Level 400 Modifications

```
Pre-Upgrade Planning:
  ☐ Identify all assets currently bound to V1 (oem_tmpl_v_001)
  ☐ Assess impact of each Level 400 change (bearing, pump, control)
  ☐ Determine upgrade scope: FULL vs. PARTIAL vs. DEFERRED
  ☐ Schedule physical replacement: Check asset availability, maintenance windows
  ☐ Budget review: Calculate cost_delta for each component upgrade
  ☐ Notify asset owners: Send notifications for scheduled upgrades

Database Preparation:
  ☐ Verify OEM_TEMPLATE_VERSION V2 data is complete (all specs)
  ☐ Verify TEMPLATE_COMPONENT records exist for V2 (3 components: motor, pump, control)
  ☐ Verify COMPONENT records exist for new/updated models (ceramic bearing, eplus pump, safety PLC)
  ☐ Create MASTER_COMPONENT records for new versions
  ☐ Prepare ASSET_TEMPLATE_BINDING records (PARTIAL_UPGRADE type)
  ☐ Set up ASSET_EVENT_STORE event templates for upgrade events

Physical Replacement (Maintenance Window):
  ☐ Remove old bearing assembly (status = 'replaced', removal_date = 2025-08-01)
  ☐ Install new ceramic hybrid bearing (status = 'active', installation_date = 2025-08-01)
  ☐ Remove old hydraulic pump (status = 'replaced', removal_date = 2025-08-01)
  ☐ Install new e-Plus pump with EcoMode config (status = 'active', installation_date = 2025-08-01)
  ☐ Test systems: bearing load test, pump pressure test, control logic verification
  ☐ Sign-off by maintenance technician & operations manager

Database Updates (Coordinated with Physical Replacement):
  ☐ UPDATE MASTER_COMPONENT: Set old components to 'replaced'
  ☐ INSERT MASTER_COMPONENT: Create new components with V2 specs
  ☐ UPDATE MASTER_ATTRIBUTE: Archive old bearing spec (status = 'archived')
  ☐ INSERT MASTER_ATTRIBUTE: Create new ceramic bearing spec (source = 'template_v2_upgrade')
  ☐ INSERT ASSET_TEMPLATE_BINDING: Record PARTIAL_UPGRADE binding
  ☐ INSERT ASSET_EVENT_STORE: Log ComponentReplaced, ComponentInstalled events
  ☐ INSERT ASSET_AUDIT: Log action = 'UPDATE' for each component change

Post-Upgrade Verification:
  ☐ Query MASTER_COMPONENT: Confirm ceramic bearing + e-Plus pump + old PLC
  ☐ Query ASSET_TEMPLATE_BINDING: Confirm PARTIAL_UPGRADE binding exists
  ☐ Query ASSET_AUDIT: Confirm 3 component changes logged
  ☐ Visual inspection: Verify physical components match database state
  ☐ Operational test: Run asset through typical usage cycles, monitor sensor data
  ☐ Close maintenance ticket, update asset documentation

Deferred Items (Control System — Planned 2026):
  ☐ Schedule: Q1 2026 upgrade window
  ☐ Plan: Replace S7-1200 with S7-1200F Safety-rated
  ☐ Update: Create new ASSET_TEMPLATE_BINDING record (FULL_UPGRADE type)
  ☐ Communicate: Send 90-day advance notice to asset owner
```

---

## 8. Key Takeaways

| Aspect | Detail |
|--------|--------|
| **GOS Hierarchy Depth** | 5 levels (Lv1=200, Lv2=300, Lv3=400, Lv4=500, Lv5=600) |
| **Level 400 Example** | RTG (Rubber-Tired Gantry) — Parent for 3 L400 components |
| **Level 400 Components** | Motor (100), Hydraulic (200), Control (230) — 3 component types |
| **Version Change Type** | SPEC CHANGE within same Level 400 component (not hierarchy restructure) |
| **V1 → V2 Changes** | Bearing material, pump model, PLC generation — all at Level 400 |
| **Asset Binding** | RTG #001 tracks V1 (original) → V2 (partial upgrade) via ASSET_TEMPLATE_BINDING |
| **Selective Upgrade** | Motor bearing + pump upgraded (2025-08-01), control system deferred (Q1 2026) |
| **No Duplication** | Same asset_id (asset_001) used across versions, only components replaced |
| **Audit Trail** | ASSET_AUDIT records all 3 component changes + binding creation |
| **Future Query** | Show RTG #001 history: ORIGINAL (V1) → PARTIAL_UPGRADE (V2) → FULL_UPGRADE (V2 complete) |

