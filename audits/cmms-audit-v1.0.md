# CMMS Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Architecture Complete — Ongoing Build  
**Source paths audited:** `src/cmms/`, `src/cmms/ARCHITECTURE.md`, `src/cmms/README.md`, `CLAUDE.md §§7.3, 18.2`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | CMMS (Computerised Maintenance Management System) |
| DB schema | `ooh_cmms` (inferred from prefix conventions) |
| Frontend path | `apps/cmms/src/` |
| Backend path | `src/cmms/src/` |
| Owner | Vertical |
| Phase | Architecture complete (7 ADRs accepted); build in progress |

---

## 2. Current State Summary

The CMMS module is OOH OS's asset operations and maintenance management engine. It manages the full work order lifecycle for physical OOH assets (panels, screens, structures), integrates with IoT device telemetry, and coordinates field crews via AI agents.

The architecture is formally documented via 7 Architecture Decision Records (ADRs) in `ARCHITECTURE.md` (24.3KB), covering task taxonomy, lifecycle state machines, AI agent integration, IoT hardware abstraction, and cross-platform orchestration.

**What is architecturally decided (ADRs Accepted):**
- ADR-001: Unified task taxonomy — 7 maintenance strategies aligned with EN 13306 / ISO 14224
- ADR-002: Emergency is a flag (`is_emergency: boolean`), not a strategy
- ADR-003: Lifecycle phases (commissioning, testing) excluded from maintenance taxonomy
- ADR-004: Two-layer navigation shell (`InnerTabNavigation` + `SubNavigation`)
- ADR-005: Generic lifecycle state machine engine governing 4 entities (Asset, Product, Work Order, Contract)
- ADR-006: Strategy-lifecycle integration (strategy-conditional guards + effects)
- ADR-007: Intelligent Work Order Engine — IoT, AI agents, cross-platform orchestration

**What is built:**
- Lifecycle engine core (`src/cmms/src/lib/lifecycle/engine.ts`)
- 4 entity lifecycle configs (Asset, Product, Work Order, Contract)
- DevicePort hardware abstraction interface
- Manufacturer adapters: Dynascan, Unilumin, Novastar
- TelemetryPort pipeline (ingestion → threshold eval → anomaly detection → degradation modelling)
- PriorityScoringEngine (15% business impact weight fed by campaign revenue-at-risk)
- OpenAPI spec (20.7KB) documenting all CMMS endpoints
- `apps/cmms/` — standalone Vite frontend app for the CMMS UI

**What is stubbed / incomplete:**
- Sales integration (stub returns no active maintenance for any screen per `cmms.stub.ts` in Sales)
- Weather intelligence integration (`WeatherPort` — designed, not wired to live provider)
- Campaign revenue awareness (`CampaignPort` — designed, uses Sales/Salesforce data)

---

## 3. Asset Hierarchy

The CMMS module uses a 4-level physical asset hierarchy:

```
Location → Structure → Panel → Face
```

Each `Face` maps to:
- A `sales_frame` (commercial inventory record owned by Sales)
- Zero or more IoT device bindings via `asset_components.telemetry_endpoint`

**Data ownership rule:** Assets module owns physical asset records. CMMS owns operational maintenance records. Sales owns commercial inventory records. None modifies the other's data directly.

---

## 4. Lifecycle State Machines

Managed by the generic engine in `src/cmms/src/lib/lifecycle/engine.ts`:

### Work Order States
`draft → pending → assigned → in_progress → on_hold → completed → cancelled → closed`

### Asset States
`planning → procurement → installation → commissioning → operational → maintenance → suspended → decommissioning → retired`

### Product States
`draft → review → approved → commissioned → deprecated → retired`

### Contract States
`draft → pending_approval → active → renewal_pending → expired → terminated`

**Engine features:** Guard conditions, approval gates, side effects, `lifecycle_transitions` audit trail, `getAvailableTransitions()` for AI agent reasoning.

---

## 5. Task Taxonomy (ADR-001 + ADR-002)

| Strategy | Philosophy | AI Agent use |
|---|---|---|
| `preventive` | Scheduled — prevent failure | PM schedule generation |
| `corrective` | Repair after failure/defect | FMEA failure mode logging |
| `condition_based` | Triggered by sensor/telemetry data | Anomaly detection → auto-WO |
| `routine` | Standard recurring operational | Recurrence auto-create on complete |
| `improvement` | Upgrade / life extension | CapEx approval gate |
| `inspection` | Formal assessment for compliance | Structured inspection report required |
| `other` | Catch-all | Ad-hoc requests |

**Emergency:** `is_emergency: boolean` flag — overrides `sla_priority_tier` to 1, triggers immediate dispatch. Not a strategy.

---

## 6. AI Agent Ecosystem (ADR-007 §7.5)

7 specialist agents registered in AgentHub:

| Agent | Default Autonomy | Role |
|---|---|---|
| `TriageAgent` | L2 | Evaluates predictive flags, creates WOs from threshold breaches |
| `DispatchAgent` | L2 | Assigns field crews based on skills, location, availability |
| `PredictiveMaintenanceAgent` | L1 | RUL estimation, failure prediction, PM schedule optimisation |
| `WeatherRiskAgent` | L2 | Assesses weather safety, reschedules WOs |
| `CampaignImpactAgent` | L1 | Revenue-at-risk calculation, SLA deadline pressure |
| `VerificationAgent` | L1 | Photo verification of completed work (AI image analysis) |
| `ReportingAgent` | L1 | MTBF/MTTR analytics, SLA compliance reports |

---

## 7. IoT Hardware Abstraction (ADR-007 §7.1)

**`DevicePort` interface** — contract for all manufacturer adapters:
- Connection management (connect/disconnect/health)
- Bidirectional telemetry (read metrics + send commands)
- Firmware management (version check, update initiation)
- Device provisioning/decommissioning

**Adapters built:**
- `DynascanAdapter` — Dynascan Cloud API (high-brightness outdoor)
- `UniluminAdapter` — Unilumin Cloud Platform (LED display systems)
- `NovastarAdapter` — VNNOX Cloud (sending/receiving card systems)

**Planned adapters:** BrightSign, Samsung, LG, Daktronics

**Telemetry pipeline:**
1. Ingestion (`TelemetryPort`)
2. Threshold evaluation (configurable per component type, with tenant overrides)
3. Anomaly detection (statistical deviation, trend analysis, pattern break)
4. Degradation modelling (RUL estimation, health scoring per component × manufacturer)
5. Aggregation (time-bucketed min/max/avg/median/stddev)

---

## 8. EventBus Surface

### Outbound Events (CMMS → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `cmms.maintenance.scheduled` | Maintenance window booked | Sales (blocks `sales_digital_availability`) |
| `cmms.maintenance.completed` | Work order closed | Sales (releases blocked availability) |
| `cmms.screen.fault` | Critical fault detected | Sales (marks frame unavailable), Comms Hub |
| `cmms.screen.restored` | Screen repaired + verified | Sales (restores availability) |
| `cmms.playback.logged` | Device play log received | Sales (`sales_proof_of_play`) |
| `cmms.sensor.threshold_breach` | IoT threshold exceeded | Comms Hub (SMS/Push to field crew) |
| `cmms.weather.alert` | Severe weather detected | Comms Hub (field crew notification) |
| `cmms.work_order.created` | Work order created | Comms Hub, DMS |
| `cmms.work_order.completed` | Work order closed | Analytics, Assets |

### Inbound Events (Platform → CMMS)

| Event | Source | CMMS Action |
|---|---|---|
| `sales.booking.confirmed` | Sales | Flag screens as commercially committed |
| `sales.booking.cancelled` | Sales | Release commercial flag |
| `assets.frame.activated` | Assets | Begin PM schedule initialisation |
| `assets.frame.decommissioned` | Assets | Cancel future WOs |

---

## 9. External Integrations

| Integration | Type | Status | Notes |
|---|---|---|---|
| Dynascan Cloud API | DevicePort adapter | ✅ Built | High-brightness outdoor displays |
| Unilumin Cloud Platform | DevicePort adapter | ✅ Built | LED display systems |
| VNNOX Cloud (Novastar) | DevicePort adapter | ✅ Built | Sending/receiving card systems |
| Weather API (provider TBD) | WeatherPort | 🟡 Designed | Not wired to live provider |
| Sales Campaign data | CampaignPort | 🟡 Stub | Revenue-at-risk feeds PriorityScoringEngine |
| Comms Hub | EventBus | ✅ Designed | Threshold breach + weather alerts |
| DMS | EventBus | ✅ Designed | Work order documents |
| AgentHub | API call | ✅ Designed | 7 agents registered |

---

## 10. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| `apps/cmms/` is a standalone Vite app separate from main repo frontend | Medium | Separate deployment concern — needs CI/CD wiring |
| Sales stub returns no active maintenance (inverted from reality) | Medium | `cmms.stub.ts` always returns clean — real CMMS integration needed for Sales to block availability |
| WeatherPort provider not selected | Medium | Provider TBD — WOs with `weather_dependent: true` can't enforce guards without live weather data |
| `lifecycle_transitions` table size management | Low | Append-only audit trail with no archival policy — will grow with scale |
| No UI-level access to DevicePort adapters (no debug console) | Low | Field engineers cannot inspect IoT device state from CMMS UI |
| MTBF/MTTR analytics must exclude lifecycle-phase tasks (ADR-003) | Low | Enforcement relies on correct strategy assignment — no hard DB constraint prevents misclassification |
| `condition_based` strategy requires sensor data reference | Low | Guard condition requires `cmms.sensor.threshold_breach` linkage — not enforced in Phase 1 |
| `improvement` strategy requires cost approval gate | Low | CapEx threshold is configurable per tenant but not yet in `tenant_business_rules` |
| OpenAPI spec (20.7KB) not yet validated against running implementation | Low | Spec may diverge from actual routes as module is built |

---

*End of CMMS Module Audit v1.0*
