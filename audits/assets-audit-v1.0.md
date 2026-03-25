# Assets Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Schema + Service Layer Defined — Build In Progress  
**Source paths audited:** `src/assets/`, `CLAUDE.md §§18.2 Assets`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Assets |
| DB schema | `ooh_assets` (inferred from prefix conventions) |
| Backend path | `src/assets/src/` |
| Owner | Vertical |
| Phase | Schema + ports defined; UI and full service layer in progress |

---

## 2. Current State Summary

The Assets module is the physical asset register for OOH OS. It owns the authoritative records for structures, panels, faces, and screens as physical objects — their installation status, maintenance history, utilisation, and commissioning lifecycle. This is distinct from Sales (which owns commercial inventory records) and CMMS (which owns maintenance operations).

**Data ownership rule:** Assets module owns physical asset records. Sales module owns commercial inventory records. They reference the same physical entities but neither module modifies the other's data directly.

**Integration pattern:** Assets is upstream of Sales. When a new structure/frame/screen is activated in Assets, it emits EventBus events that trigger Sales to create the corresponding commercial inventory records.

---

## 3. Asset Hierarchy

```
Location → Structure → Frame (classic) / Screen (digital)
```

Mirrors the Sales inventory hierarchy but from the physical asset perspective:
- `assets_location` = physical site
- `assets_structure` = advertising structure at a site
- `assets_frame` = printable face on a structure
- `assets_screen` = digital display on a structure

Each physical asset has:
- Lifecycle state (planning → operational → maintenance → retired)
- Maintenance history (via CMMS integration)
- Commercial utilisation record (via Sales integration)
- Physical attributes (dimensions, illumination type, material, GPS coordinates)

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `src/assets/src/ports/` — port interfaces | ✅ Built | `IAudiencePort` equivalent for assets |
| `src/assets/src/repositories/` — data layer | ✅ Built | Repository pattern |
| `src/assets/src/services/` — business logic | ✅ Built | Asset lifecycle services |
| `src/assets/src/types/` — type definitions | ✅ Built | Physical asset types |
| Asset activation flow (structure → frame → screen) | ✅ Designed | EventBus cascade: `assets.structure.activated` → `assets.frame.activated` → `assets.screen.registered` |
| Asset decommissioning flow | ✅ Designed | `assets.frame.decommissioned` → Sales deactivates, cancels availability |
| Physical attribute updates | ✅ Designed | `assets.frame.attributes_updated` → Sales updates frame, triggers Audience refresh |
| Asset utilisation reporting | ✅ Designed | Commercial utilisation via Sales integration |
| Asset management UI | 🟡 Partial | Build in progress |
| Bulk asset import | ❌ Missing | CSV import for existing estate |
| Asset photo gallery | ❌ Missing | Photo evidence per asset |
| GPS-based asset map view | ❌ Missing | Map view of full estate |

---

## 5. Schema Detail

**Schema:** `ooh_assets`  
**Multi-tenancy:** All tables include `tenant_id uuid NOT NULL` + RLS enforced

| Table | Purpose |
|---|---|
| `assets_location` | Physical site record — lat/lng, address, council reference |
| `assets_structure` | Advertising structure — type, dimensions, planning permission ref |
| `assets_frame` | Classic printable face — illumination type, material, face dimensions |
| `assets_screen` | Digital display — resolution, brightness, player hardware ref |
| `assets_component` | Sub-components (LED panels, power supply, lighting) — telemetry endpoint |
| `assets_utilisation` | Commercial utilisation records (FK to Sales booking) |
| `assets_photo` | Proof-of-installation / inspection photos |
| `assets_location_history` | Address / GPS change history |

---

## 6. EventBus Surface

### Outbound Events (Assets → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `assets.structure.activated` | Structure installed + active | Sales (creates `sales_structure`) |
| `assets.frame.activated` | Frame activated on structure | Sales (creates `sales_frame` + `sales_classic_availability`) |
| `assets.screen.registered` | Digital screen registered | Sales (creates `sales_screen`) |
| `assets.frame.decommissioned` | Frame taken out of service | Sales (deactivates frame, cancels availability, notifies affected bookings) |
| `assets.screen.decommissioned` | Screen removed | Sales (deactivates screen, releases digital availability) |
| `assets.frame.attributes_updated` | Physical attributes changed | Sales (updates frame record), Audience (triggers profile refresh) |

### Inbound Events (Platform → Assets)

| Event | Source | Assets action |
|---|---|---|
| `sales.booking.confirmed` | Sales | Marks frames as sold for period (utilisation reporting) |
| `cmms.maintenance.scheduled` | CMMS | Flags frame as under maintenance |
| `cmms.maintenance.completed` | CMMS | Restores frame to active |
| `sitedevelopment.site.activated` | Site Dev | Triggers Assets activation cascade |

---

## 7. External Integrations

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| Sales | Assets → Sales | EventBus | ✅ Designed — activation cascade |
| CMMS | Bidirectional | EventBus | ✅ Designed — maintenance state |
| Site Development | Site Dev → Assets | EventBus | ✅ Designed — new site triggers cascade |
| Audience | Assets → Audience | Via `assets.frame.attributes_updated` | ✅ Designed — profile refresh |
| GPS / mapping | Assets ← | API (planned) | ❌ Missing — PostGIS geography columns planned |

---

## 8. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| No asset management UI | High | Cannot manage physical assets without DB access |
| No bulk import | High | Operators with existing estates have no migration path |
| No GPS / PostGIS geography column | Medium | `assets_location` lacks `geography(POINT)` — planned for Phase 2a migration |
| Asset photo gallery missing | Medium | No visual evidence for proof of installation |
| No asset map view | Medium | Operators cannot see their full estate on a map |
| Utilisation reporting not UI-surfaced | Low | `assets_utilisation` records exist but no report UI |
| `assets_component.telemetry_endpoint` is unstructured JSONB | Low | Each manufacturer adapter uses different keys — no schema enforcement |

---

*End of Assets Module Audit v1.0*
