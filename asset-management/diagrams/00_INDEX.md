# Asset Master — Diagrams Index

> **Module:** Asset Master System | **Version:** 1.0 | **Date:** May 3, 2026

---

## Documents

| # | File | Contents |
|---|------|----------|
| 1 | [01_ER_Diagrams.md](01_ER_Diagrams.md) | All entity-relationship diagrams |
| 2 | [02_Sequence_Diagrams.md](02_Sequence_Diagrams.md) | All sequence / flow diagrams |
| 3 | [03_Component_Class_Diagrams.md](03_Component_Class_Diagrams.md) | System architecture, class diagrams |

---

## Quick Map

### ER Diagrams (`01_ER_Diagrams.md`)
| Section | Covers |
|---------|--------|
| 1. Complete ER | All entities in one view |
| 2. Core Asset Domain | `ASSET_TEMPLATE` ↔ `ASSET_MASTER` |
| 3. Attributes Domain | `ATTRIBUTE_DEF` → `TEMPLATE_ATTRIBUTE` → `MASTER_ATTRIBUTE` |
| 4. Components & Attachments | `COMPONENT` / `ATTACHMENT` → template & master links |
| 5. Event & Audit Domain | `ASSET_EVENT_STORE` → `ASSET_AUDIT` → `NOTIFICATION_LOG` |
| 6. Document Scan Domain | `ATTACHMENT` ↔ `DOCUMENT_SCAN` |

### Sequence Diagrams (`02_Sequence_Diagrams.md`)
| Section | Covers |
|---------|--------|
| 1. Asset Create | Full create flow — UI → API → DB → Event → Audit → Push |
| 2. Asset Update | Full update flow |
| 3. Audit Handler Flow | Event consumed → Audit record → Notification dispatched |
| 4. Notification Handler Flow | Multi-channel dispatch (UI Push + Email) |
| 5. Document Upload + Votiro Scan | Upload → Quarantine → Scan → CLEAN / INFECTED result |

### Component & Class Diagrams (`03_Component_Class_Diagrams.md`)
| Section | Covers |
|---------|--------|
| 1. System Architecture | End-to-end component graph |
| 2. AssetEventHandler | Class structure and relationships |
| 3. NotificationHandler | Channel pattern and log structure |
| 4. DocumentHandler | Votiro integration and file storage |
