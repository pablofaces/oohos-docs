# CMMS Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (AI — Claude Sonnet 4.6)
**Supersedes:** cmms-audit-v1.0 (2026-03-23, Cascade)
**Source paths audited:**
- `/workspace/group/everboarding/src/cmms/`
- `/workspace/group/everboarding/apps/cmms/`
- `/workspace/group/everboarding/supabase/functions/`
- `/workspace/group/everboarding/src/sales/src/` (cross-module)
- Live DB via Supabase REST API introspection

---

## 1. Executive Summary

The CMMS module has advanced materially since the v1.0 audit. Several items flagged as stubs or incomplete are now live. However, the v1.0 audit contained inaccuracies in agent naming, lifecycle config counts, adapter completion status, and — critically — the Sales stub status. This audit corrects the record.

**Production Readiness Score: 6.5 / 10**

---

## 2. ADR Count — Verified

| v1.0 Claim | v2.0 Finding | Status |
|---|---|---|
| 7 ADRs accepted | 7 ADRs present in ARCHITECTURE.md | CONFIRMED |

All 7 ADRs (ADR-001 through ADR-007) are present in `/workspace/group/everboarding/src/cmms/ARCHITECTURE.md`. The file is 393 lines.

Note: The v1.0 audit cited the file as 24.3KB. Current measured size: ~12KB (393 lines). The file may have been trimmed or the earlier measurement was incorrect.

---

## 3. Edge Function Count — DISCREPANCY

| v1.0 Claim | v2.0 Finding | Status |
|---|---|---|
| Not explicitly claimed | 19 cmms-* edge functions found | NEW DATA |

The v1.0 audit mentioned edge functions in passing (referencing `cmms-asset-availability-check` as the Sales integration point) but gave no total count. Actual count: **19 deployed edge functions** with the `cmms-` prefix.

Full list:
```
cmms-allocate-cost
cmms-asset-availability-check
cmms-check-compliance-expiry
cmms-check-warranty-expiry
cmms-close-work-order
cmms-create-work-order
cmms-deliver-scheduled-report
cmms-escalate-work-order
cmms-evaluate-meter-thresholds
cmms-generate-compliance-report
cmms-generate-pm-schedules
cmms-ingest-device-telemetry
cmms-process-inventory-movement
cmms-receive-goods
cmms-reconcile-energy
cmms-reserve-inventory-for-wo
cmms-submit-purchase-requisition
cmms-update-asset-status
cmms-update-exchange-rates
```

Total edge functions in repo: 174. CMMS claims 19 (10.9% of platform surface).

---

## 4. Services Count — Verified

| v1.0 Claim | v2.0 Finding | Status |
|---|---|---|
| Not counted | 54 service files (non-test) | NEW DATA |

`/workspace/group/everboarding/src/cmms/src/services/` contains 54 `.ts` service files excluding tests. Notable services confirmed present beyond what v1.0 listed:
- `CampaignDeadlineService.ts` — live SLA enforcement for campaign go-live deadlines
- `WeatherEventService.ts` — emergency cascade WO generation from weather events
- `DispatchReadinessService.ts` — field crew readiness checks
- `ReliabilityMetricsService.ts` — MTBF/MTTR computation
- `LOTOService.ts` — Lockout/Tagout safety compliance

---

## 5. IoT / Telemetry Adapter Status — PARTIAL DISCREPANCY

### v1.0 Claim: All 3 adapters built

| Adapter | v1.0 Status | v2.0 Finding |
|---|---|---|
| DynascanAdapter | Built | CONFIRMED — full `DevicePort` implementation with HTTP auth, retry logic, tenant config lookup |
| UniluminAdapter | Built | PARTIAL — class exists and implements `DevicePort` interface but all methods throw `not yet implemented` |
| NovastarAdapter | Built | CONFIRMED — full implementation with VNNOX Cloud API auth and device command surface |
| NovastarTelemetryAdapter | Not mentioned separately | NEW — separate `TelemetryPort` adapter with test coverage (1 test file) |

**Discrepancy:** The v1.0 audit reported `UniluminAdapter` as `Built`. It is a skeleton class: instantiates, registers in `DeviceAdapterRegistry`, but all API calls return errors. It is effectively a stub. Calling any `sendCommand()` or `readTelemetry()` on a Unilumin device in production will fail with `UniluminAdapter not yet implemented`.

The `DeviceAdapterRegistry` (singleton) auto-registers all three adapters at startup. Unilumin devices would silently get a non-functional adapter instance.

**WeatherPort — No concrete implementation exists.** The interface is defined in `ports/WeatherPort.ts`. There is a UI settings component (`OpenWeatherIntegration.tsx`) that lets admins enter an OpenWeather API key and test the connection, but no class implementing `WeatherPort` exists in the codebase. The `useWeatherData` hook returns all-null static state (`temperature: null`, `humidity: null`, etc.) — it is a hardcoded stub with no API call.

**Impact:** Any lifecycle guard or agent logic that depends on live weather data will receive null/undefined values. `weather_dependent: true` work orders cannot enforce safety guards in production.

---

## 6. Sales Stub Replacement Status — MAJOR DISCREPANCY

### v1.0 Claim: Sales integration is a stub returning no active maintenance

**This is no longer accurate.** The Sales stub described in v1.0 has been replaced by a live architecture:

1. `SalesAssetAdapter` (`src/sales/src/adapters/SalesAssetAdapter.ts`) implements `ISalesAssetPort` and calls the live `cmms-asset-availability-check` edge function via `supabase.functions.invoke()`.

2. `AvailabilityService` (`src/sales/src/services/AvailabilityService.ts`) merges CMMS maintenance state with Sales booking state. A frame is only available if **both** CMMS and Sales availability checks pass.

3. The v1.0 audit referenced `cmms.stub.ts` in the Sales module — **this file does not exist** at any path under `/workspace/group/everboarding/src/`. It has been removed entirely.

**Remaining caveat:** `CampaignIntegrationService` (`src/cmms/src/services/integration/CampaignIntegrationService.ts`) is self-described as a "shadow-mode implementation." It returns mocked data (`name: 'Mock Campaign'`, empty booking windows) and logs payloads with `console.debug('[Shadow]')`. This affects the `CampaignImpactAgent` and the 15% business-impact weight in `PriorityScoringEngine`. Revenue-at-risk calculations are currently based on mock data, not live Salesforce/Sales data.

---

## 7. Agent Ecosystem — NAMING DISCREPANCY

### v1.0 Claim: 7 agents with specific names

| v1.0 Agent Name | v2.0 Actual Name | Status |
|---|---|---|
| TriageAgent | MaintenanceAgent | RENAMED — same role: IoT anomaly analysis, WO creation from threshold breaches |
| DispatchAgent | RouteAgent | RENAMED — same role: crew assignment, route optimisation |
| PredictiveMaintenanceAgent | PredictiveAgent | RENAMED |
| WeatherRiskAgent | NOT PRESENT | MISSING — no dedicated weather risk agent exists |
| CampaignImpactAgent | ImpactAssessmentAgent | RENAMED — handles sales integration + revenue risk |
| VerificationAgent | NOT PRESENT | MISSING — no photo verification agent in codebase |
| ReportingAgent | NOT PRESENT | MISSING — no standalone reporting agent |

**Actual agents (7 total, matching count):**
1. MaintenanceAgent
2. TelemetryAgent
3. ImpactAssessmentAgent
4. SLAAgent
5. PredictiveAgent
6. PartInventoryAgent
7. RouteAgent

The agent count (7) matches v1.0, but the names and roles differ. Three agents from v1.0 (`WeatherRiskAgent`, `VerificationAgent`, `ReportingAgent`) do not exist. Weather risk handling is embedded in `WeatherEventService`. There is no photo verification agent — this capability may be in a different module or planned. Reporting likely uses `ReliabilityMetricsService` directly.

---

## 8. Lifecycle Engine — EXPANDED SCOPE DISCREPANCY

### v1.0 Claim: 4 entity lifecycle configs (Asset, Product, Work Order, Contract)

| v1.0 Claim | v2.0 Finding | Status |
|---|---|---|
| 4 entity configs | 8 entity lifecycle configs | UNDERSTATED |

Actual lifecycle configs in `/workspace/group/everboarding/src/cmms/src/lib/lifecycle/configs/`:
1. `asset.ts`
2. `contract.ts`
3. `device.ts` (new)
4. `product.ts`
5. `siteRequest.ts` (new)
6. `tool.ts` (new)
7. `vehicle.ts` (new)
8. `workOrder.ts`

The engine also ships strategy-specific guard layers for `deviceStrategy`, `workOrderStrategy`, and `assetStrategy`. Total lifecycle lib files: 31.

---

## 9. Database Schema — Verified

CMMS-relevant tables confirmed live in production DB (via REST introspection):

**Work Orders:** `work_orders`, `work_order_assignments`, `work_order_checklist`, `work_order_comments`, `work_order_materials`, `work_order_parts`, `work_order_status_history`, `work_order_templates`, `work_order_template_equipment`, `work_order_template_skills`, `work_order_template_steps`, `mobile_work_orders`

**Assets:** `assets`, `asset_components`, `asset_component_faces`, `asset_faces`, `asset_panels`, `asset_structures`, `asset_sites`, `asset_locations`, `asset_hierarchy_view`, `asset_inspections`, `asset_inspection_schedules`, `asset_risk_scores`, `asset_performance_metrics`, `asset_financial_records`, `asset_criticality_ratings`, `asset_decommission_records`, `asset_status_history`, `asset_revenue_tracking`, `asset_utilization`, `asset_types`, `asset_subtypes`, `asset_relationships`, `asset_products`, `asset_permits`, `asset_license_records`, `asset_images`, `asset_import_batches`, `asset_saved_searches`, `asset_correspondence`

**Technicians:** `technician_profiles`, `technician_skills`, `technician_certifications`, `technician_schedules`, `technician_shifts`, `technician_availability`, `technician_locations`, `technician_roles`, `technician_role_assignments`, `technician_expertise_groups`, `technician_group_assignments`, `technician_leave`, `technician_inventory`, `technician_stock_movements`, `technician_task_summary`

**Inventory:** `inventory_items`, `inventory_categories`, `inventory_locations`, `inventory_stock_levels`, `inventory_movements`, `inventory_transactions`, `inventory_alerts`, `inventory_reorder_rules`, `spare_parts_catalog`, `maintenance_spare_parts_map`

**Telemetry/IoT:** `telemetry_readings`, `telemetry_threshold_rules`, `threshold_rules`, `threshold_breaches`, `cmms_devices`, `cmms_tasks`

**Maintenance:** `maintenance_tasks`, `maintenance_profiles`, `maintenance_programs`, `maintenance_windows`, `maintenance_vendors`, `maintenance_region_coordination`, `maintenance_predictive_models`, `maintenance_tasks`

**Cleaning:** `automated_cleaning_schedules`, `cleaning_consumables_inventory`, `cleaning_performance_analytics`, `cleaning_quality_checkpoints`, `cleaning_route_optimization`, `cleaning_vehicles`

**Vehicles/Territories:** `vehicles`, `vehicle_equipment`, `territory_assets`, `territory_technicians`

**Products:** `product_asset_types`, `product_cleaning_templates`, `asset_performance_profiles`

The v1.0 audit did not enumerate the DB schema. The actual table surface is substantially larger than the architecture description suggests (~90+ CMMS-relevant tables confirmed in production).

---

## 10. OpenAPI Spec

| v1.0 Claim | v2.0 Finding | Status |
|---|---|---|
| 20.7KB spec | 820-line YAML | ROUGHLY CONSISTENT |

The spec exists at `/workspace/group/everboarding/src/cmms/openapi.yaml` (820 lines). File size check was not re-measured in bytes, but line count is consistent with a ~20KB YAML document.

---

## 11. apps/cmms Frontend

The standalone Vite app at `/workspace/group/everboarding/apps/cmms/` is minimal: `index.html`, `main.tsx`, `routes.tsx`, `Login.tsx`, `sw.ts`, and config files. It is a thin shell — the actual 918 component files live in `/workspace/group/everboarding/src/cmms/src/components/` and are imported as a package dependency. The `vercel.json` suggests this app is independently deployed to Vercel but CI/CD wiring status is unverified.

---

## 12. Known Issues — Status Update vs v1.0

| v1.0 Issue | v2.0 Status |
|---|---|
| Sales stub returns no active maintenance | RESOLVED — live `cmms-asset-availability-check` edge function wired via `SalesAssetAdapter` |
| WeatherPort provider not selected | PARTIALLY RESOLVED — OpenWeather selected as provider, UI settings component built, but `WeatherPort` interface has no concrete implementation class and `useWeatherData` hook returns null values |
| `apps/cmms/` separate deployment | STILL OPEN — Vercel config present, CI/CD wiring unverified |
| `lifecycle_transitions` table archival | STILL OPEN — no archival policy observed |
| No DevicePort debug console for field engineers | STILL OPEN |
| MTBF/MTTR misclassification risk | STILL OPEN |
| `condition_based` sensor reference not enforced | STILL OPEN |
| CapEx approval gate not in `tenant_business_rules` | STILL OPEN |

**New issues identified in v2.0:**

| Issue | Severity | Notes |
|---|---|---|
| `UniluminAdapter` registered but non-functional | High | All methods throw — Unilumin devices in production will silently fail adapter calls |
| `WeatherPort` has no implementation class | High | All weather-dependent guards are effectively bypassed; `useWeatherData` returns all nulls |
| `CampaignIntegrationService` in shadow mode | Medium | Revenue-at-risk in `PriorityScoringEngine` uses mock campaign data — SLA priority scores are not accurate for campaign-affected assets |
| Agent name drift from ADR-007 spec | Low | 3 agents named in ADR-007 (WeatherRisk, Verification, Reporting) have no implementation; 3 others were silently renamed. ADR-007 should be updated to reflect actual agent manifest |
| Lifecycle config expanded to 8 entities undocumented | Low | `device.ts`, `siteRequest.ts`, `tool.ts`, `vehicle.ts` configs exist with no ADR coverage |

---

## 13. Production Readiness Score: 6.5 / 10

| Dimension | Score | Rationale |
|---|---|---|
| Core work order lifecycle | 9/10 | Engine solid, 8 entity configs, guards + effects in place, full audit trail design |
| DB schema completeness | 8/10 | ~90 production tables covering all CMMS domains including cleaning, vehicles, territories |
| Edge function coverage | 7/10 | 19 edge functions covering key operations (create, close, escalate, telemetry ingest, PM schedules, compliance) |
| IoT adapter readiness | 5/10 | Dynascan and Novastar full; Unilumin is a non-functional stub registered in the live registry |
| Weather intelligence | 2/10 | Interface designed, provider selected (OpenWeather), settings UI built — but zero live data flowing; hook returns nulls |
| Sales integration | 7/10 | Availability blocking is live end-to-end; campaign revenue-at-risk is still shadow-mode |
| Agent ecosystem | 6/10 | 7 agents implemented and exported; WeatherRisk, Verification, Reporting capabilities absent |
| Frontend (apps/cmms) | 5/10 | 918 component files, functional UI shell, but deployment pipeline unverified |
| Test coverage | 4/10 | Only 1 test file found for IoT adapters (NovastarTelemetryAdapter); no evidence of service-level tests |
| Documentation accuracy | 4/10 | v1.0 ADRs exist and are solid; agent names, lifecycle scope, and adapter status all misrepresented |

**Overall: 6.5 / 10** — The CMMS is a credible module with a strong data model and a working work-order pipeline. Two hard blockers before claiming production-grade: (1) `UniluminAdapter` must be completed or removed from the registry, and (2) `WeatherPort` needs a concrete implementation connected to the OpenWeather API key infrastructure already built in settings. The shadow-mode `CampaignIntegrationService` is a medium-priority fix once Salesforce integration is scheduled.
