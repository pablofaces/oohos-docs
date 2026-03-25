# Assets Module — Deep Code + Schema Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (AI — autonomous development agent)
**Source:** Live codebase at `/workspace/group/everboarding` + production DB REST introspection
**Predecessor:** v1.0 audit by Cascade (2026-03-23)

---

## 1. Scope of This Audit

This audit extends v1.0 with direct code reading, live DB introspection, and cross-module
boundary analysis. It covers:

- Schema location reality (public vs ooh_assets)
- TypeScript module completeness vs v1.0 claims
- DB table existence vs what the repository queries
- Sales↔Assets EventBus wiring status
- CMMS overlap and boundary violations
- UI completeness
- Production readiness score

---

## 2. What Changed Since v1.0

### 2.1 New findings not in v1.0

v1.0 described the Assets module as "IAssetPort interface + 1 service" (CLAUDE.md §3 wording at
time of that audit). The actual state is substantially more complete:

| Item | v1.0 said | v2.0 found |
|------|-----------|------------|
| TypeScript files | 1 service stub | 11 files — full port/service/repo/types layer |
| IAssetPort | Referenced as pending | Fully implemented: 14 methods, all tenanted |
| ISalesAssetPort | Not mentioned | Defined and implemented (subset of 7 methods) |
| AssetManagementAdapter | Not mentioned | CMMS adapter fully wiring IAssetPort |
| SalesAssetAdapter | Not mentioned | Sales adapter calling edge function |
| AssetStatus | Fragmented conflict noted | Canonical 11-value enum + legacy map + transition graph |
| Unit tests | Not mentioned | Test file present at `__tests__/AssetManagementService.test.ts` |
| Standalone /assets route | Not mentioned | `/assets/*` route with 4-tab UI shell live in AnimatedRoutes |
| Edge functions | Not enumerated | 2 deployed: `cmms-asset-availability-check`, `cmms-update-asset-status` |

### 2.2 v1.0 claims confirmed accurate

- Schema extraction migration (`migration_extract_assets.sql`) exists but is entirely commented
  out — all ALTER TABLE statements are no-ops. All tables remain in `public` schema.
- GPS/map view and bulk CSV import remain missing.
- Photo gallery still deferred.

### 2.3 v1.0 claims that require correction

v1.0 stated "the activation cascade — structure → frame → screen — is fully designed but awaits
UI completion." The activation cascade is *not* implemented as EventBus emissions. See §5 below.

---

## 3. Schema Location — public vs ooh_assets

**Finding: All asset tables remain in the `public` schema. The `ooh_assets` schema has never been
activated in production.**

Evidence:
- `migration_extract_assets.sql` creates the `ooh_assets` schema and grants permissions, but
  every `ALTER TABLE public.* SET SCHEMA ooh_assets` line is commented out.
- Live DB introspection (Supabase REST `/rest/v1/`) confirms the following tables exist in
  `public` (not `ooh_assets`):
  - `assets` (primary table)
  - `asset_component_faces`
  - `asset_financial_records`
  - `asset_performance_metrics`
  - `asset_risk_scores`
  - `asset_status_history`
  - `asset_components`, `asset_images`, `asset_types`, `asset_structures`, `asset_locations`,
    `asset_panels`, `asset_faces`, `asset_sites`, `asset_subtypes`, `asset_utilization`,
    `asset_inspections`, `asset_inspection_schedules`, `asset_license_records`,
    `asset_import_batches`, `asset_risk_scores`, `asset_criticality_ratings`,
    `asset_decommission_records`, `asset_revenue_tracking`, `asset_saved_searches`,
    `asset_permits`, `asset_products`, `asset_relationships`, `asset_correspondence`,
    `asset_performance_profiles`
  - Total: ~30 asset_* tables in public

- `asset_audience_metrics` does **not** exist. The DB has `face_audience_metrics` instead
  (confirmed in `20260310110217_remote_schema.sql`). The `AssetRepository.getAudienceMetrics()`
  method queries `asset_audience_metrics` — this query will silently return null on every call.

**Impact:** The `ooh_assets` schema extraction (sprint GC-2, referenced in CLAUDE.md §3) is
planned but unstarted. The module is functional against `public` but violates the schema-per-module
isolation principle. This is a known debt item, not a regression.

**Discrepancy from v1.0:** v1.0 did not explicitly state the schema was still `public`. The
CLAUDE.md §3 entry says "Real asset implementation lives inside `src/cmms/` pending Sprint 3 GC-2
extraction to `ooh_assets` schema" — this is now outdated. The TypeScript layer has been
extracted to `src/assets/`; only the DB schema extraction remains.

---

## 4. TypeScript Module — Completeness Assessment

### 4.1 File inventory

```
src/assets/
├── README.md
├── react.svg                                  (stray file — should not be here)
├── schema/
│   └── migration_extract_assets.sql           (all commented out — see §3)
└── src/
    ├── __tests__/
    │   └── AssetManagementService.test.ts     (exists, content not verified for pass rate)
    ├── index.ts                               (module init — initAssets/getAssetService)
    ├── ports/
    │   ├── IAssetPort.ts                      (14 methods, fully tenanted, stable)
    │   └── ISalesAssetPort.ts                 (7-method Sales subset, stable)
    ├── repositories/
    │   └── AssetRepository.ts                 (full Supabase implementation, ~313 lines)
    ├── services/
    │   └── AssetService.ts                    (AssetManagementService implementing IAssetPort)
    └── types/
        ├── Asset.ts                           (9 interfaces: Asset, AssetFace, ...)
        └── AssetStatus.ts                     (canonical 11-value enum + transitions + normaliser)
```

### 4.2 Service layer completeness

`AssetManagementService` fully implements all 14 `IAssetPort` methods. The service adds one
meaningful layer of logic: status transition validation via `isValidAssetTransition()` before
delegating to the repository. All other methods are clean pass-throughs.

### 4.3 Repository: bugs and gaps found

**Bug 1 — `asset_audience_metrics` table does not exist.**
`AssetRepository.getAudienceMetrics()` queries `.from('asset_audience_metrics')`. The actual DB
table is `face_audience_metrics`. This will return null on every call in production.
`IAssetPort.getAudienceMetrics()` throws when null is returned from the repo (at the service
layer), so callers will get an exception rather than silent empty data.

**Bug 2 — Secondary table queries are not tenant-scoped.**
The following repository methods query their respective tables without a `platform_tenant_id` filter:
- `getStatusHistory()` — queries `asset_status_history` with only `asset_id`
- `getFaces()` — queries `asset_component_faces` with only `asset_component_id`
- `getFaceIds()` — same
- `getFinancialRecord()` — queries `asset_financial_records` with only `asset_id`
- `getPerformanceMetrics()` — queries `asset_performance_metrics` with only `asset_id`
- `getRiskScore()` — queries `asset_risk_scores` with only `asset_id`
- `getAudienceMetrics()` — same (plus the table name bug above)

Only `findById`, `findAll`, `findByFaceIds`, `findAvailable`, `checkAvailability`,
`updateStatus`, and `getPricingAttributes` are properly tenant-scoped. If RLS policies exist on
these secondary tables, they provide the safety net. If not, cross-tenant data leakage is possible.

**Gap 1 — `findAvailable()` does not check bookings.**
The implementation returns all `operational` assets. It does not query `sales_digital_availability`
or `sales_classic_availability` to exclude frames with active bookings. This is effectively a stub
and is labeled as a comment in the file: "Assets with operational status are considered available
unless they have a booking conflict in the given date range." The conflict check is absent.

**Gap 2 — `checkAvailability()` likewise has no booking conflict check.**
Returns `available: true` for any asset with `operational_status = 'operational'` and
`conflicts: []` always. The `conflicts` array in `AssetAvailabilityResult` will always be empty.

**Gap 3 — `performance_metrics` query is wrong.**
`getPerformanceMetrics()` filters by `created_at` between `period.from` and `period.to`.
The `asset_performance_metrics` table likely has a `metric_date` column (confirmed by the index
`idx_asset_performance_metrics_date` in the remote schema migration). Filtering on `created_at`
will not return the correct period data.

### 4.4 AssetStatus — canonical enum

This is the strongest part of the module. `AssetStatus.ts` defines:
- 11-value canonical enum replacing 4 conflicting legacy status sets
- `LEGACY_STATUS_MAP` covering ~20 legacy values (lifecycle engine, validation service, Zod
  schema, face-level, UI/import)
- `ASSET_STATUS_TRANSITIONS` as a full directed graph (only `DECOMMISSIONED` is terminal)
- `normalizeAssetStatus()` with fallback to `PLANNED` for unknown values
- `isValidAssetTransition()` used by the service layer

This is production-quality and correct.

### 4.5 ISalesAssetPort — divergence between assets and sales modules

There are **two different `ISalesAssetPort` definitions** in the codebase:

| Location | Interface |
|----------|-----------|
| `src/assets/src/ports/ISalesAssetPort.ts` | 7 methods: findAll, findByFaceIds, findAvailable, checkAvailability, getPricingAttributes, getAudienceMetrics, getFaceIds |
| `src/sales/src/ports/ISalesAssetPort.ts` | 3 methods: checkAvailability, getPricingAttributes, getAudienceMetrics |

The Sales module uses its own 3-method `ISalesAssetPort` (pointing to the edge function). The
Assets module carries a 7-method `ISalesAssetPort` that is not consumed by anything. These have
diverged without a shared source of truth.

---

## 5. Sales↔Assets EventBus Wiring

**Finding: The EventBus events documented in CLAUDE.md §18.2 (Assets→Sales) are NOT emitted
anywhere in the codebase.**

CLAUDE.md §18.2 specifies these events should flow from Assets to Sales:
- `assets.structure.activated` — triggers Sales to create `sales_structure` record
- `assets.frame.activated` — triggers Sales to create `sales_frame` + availability records
- `assets.screen.registered` — triggers Sales to create `sales_screen` record
- `assets.frame.decommissioned` — triggers Sales to deactivate frame
- `assets.frame.attributes_updated` — triggers Sales to update frame + Audience refresh

**Search result:** Zero occurrences of `assets.frame.activated`, `assets.screen.registered`, or
`assets.structure.activated` in `src/`. The activation cascade from Assets to Sales exists only
in documentation.

**What actually exists:** The `cmms-update-asset-status` edge function emits a status-change
event by inserting into `hub_message_logs` with `eventKey: "cmms.asset.status_changed"`. This is
not an EventBus emission — it is a log entry that nothing subscribes to. It also emits
`cmms.asset.operational` toward Everboarding via the same `hub_message_logs` pattern when a
transition to `operational` occurs. These are informal notifications, not the typed EventBus
events the architecture requires.

**Implication:** When a new asset/frame/screen is activated via the Assets module or the
`cmms-update-asset-status` edge function, Sales does **not** automatically create the
corresponding `sales_frame` or `sales_screen` records. The inventory gap between physical assets
and the commercial Sales inventory must currently be bridged manually or via the CMMS UI.

**Root cause:** The Assets module's TypeScript service layer (`AssetService.ts`,
`AssetRepository.ts`) has no EventBus import or usage at all. EventBus wiring must be added to
`AssetService.updateStatus()` and a new `activateFrame()` / `registerScreen()` method set.

---

## 6. CMMS Overlap and Boundary Violations

### 6.1 Architectural intent

The intended boundary:
- `src/assets/` owns asset CRUD, status, risk, financial records, performance, faces
- CMMS consumes `IAssetPort` via `AssetManagementAdapter`
- CMMS never queries asset tables directly

### 6.2 Actual state — 57 direct `assets` table violations in CMMS

A direct count of `.from('assets')` calls in `src/cmms/` finds **57 occurrences** bypassing
`IAssetPort`. Affected files include:

- `src/cmms/src/components/work-orders/WeatherAlertBanner.tsx`
- `src/cmms/src/components/lease/LeaseAgreementForm.tsx`
- `src/cmms/src/components/maintenance/RegionCoordinationDashboard.tsx` (2 calls)
- `src/cmms/src/components/maintenance/EnvironmentalMonitoring.tsx` (2 calls)
- `src/cmms/src/components/maintenance/ComplianceScheduleManager.tsx` (2 calls)
- `src/cmms/src/components/maintenance/RevenueEmergencyResponse.tsx`
- `src/cmms/src/components/maintenance/PredictiveMaintenanceEngine.tsx`
- `src/cmms/src/components/assets/BulkTerritoryAssignment.tsx`
- `src/cmms/src/components/assets/AssetTerritorySelector.tsx` (2 calls)
- `src/cmms/src/hooks/useLeaseAgreementAssets.ts`
- `src/cmms/src/hooks/repositories/useSearchRepository.ts`

The `AssetManagementAdapter` exists and is correctly wired, but the majority of CMMS components
still use direct Supabase queries against `public.assets`. This is an active boundary violation
that makes the `ooh_assets` schema extraction harder — any move of the `assets` table to
`ooh_assets` would break all 57 of these callsites.

### 6.3 CMMS UI for assets

The CMMS contains a rich asset management UI that predates the `src/assets/` module extraction:

| CMMS Page | Route | Notes |
|-----------|-------|-------|
| `Assets.tsx` | via CMMS routes | List with filters, pagination, bulk territory assignment |
| `AdminAssetList.tsx` | CMMS admin | Admin-specific asset list |
| `AssetDetail.tsx` | CMMS detail | Asset detail view |
| `AssetsByPartner.tsx` | CMMS | Partner-filtered view |
| `AssetsModule.tsx` | CMMS | Module wrapper |
| `AssetComponentConfigurator.tsx` | `/assets/:id/components` | Component-level configurator |

There is also a **standalone `/assets/*` route** in `AnimatedRoutes.tsx` that renders
`AssetManagementEntry` (from `src/cmms/`). This entry point provides a 4-tab shell:
`Assets | Partners | Products | Permits`. This is the module's primary production UI and it is
live and routed. It renders `AssetsPage` (the CMMS Assets.tsx) as the index tab.

So the UI is not missing — it is the CMMS UI re-exported as a standalone module entry point.
What is missing is a UI built on top of the new `src/assets/` TypeScript layer (IAssetPort,
AssetManagementService). The UI talks directly to Supabase, not through the port.

---

## 7. Production DB — Key Table Status

| Table | In DB? | Schema | Notes |
|-------|--------|--------|-------|
| `assets` | Yes | public | Primary asset table |
| `asset_component_faces` | Yes | public | Status check uses legacy values (operational/degraded/failed/maintenance/decommissioned) — not the 11-value canonical enum |
| `asset_financial_records` | Yes | public | |
| `asset_performance_metrics` | Yes | public | Has `metric_date` column, not `created_at` |
| `asset_risk_scores` | Yes | public | |
| `asset_status_history` | Yes | public | |
| `asset_audience_metrics` | **No** | — | Does not exist. `face_audience_metrics` exists instead |
| `ooh_assets` schema | **No** | — | `GRANT` statements ran but no tables moved |

The `asset_component_faces` table has a status CHECK constraint allowing only
`operational | degraded | failed | maintenance | decommissioned`. The canonical `AssetStatus` enum
uses `under_maintenance` (not `maintenance`) and `fault` (not `failed`). The
`normalizeAssetStatus()` function handles these mappings — but the DB constraint will reject any
`UPDATE` using the canonical values directly. The `AssetRepository.updateStatus()` method writes
the canonical status value (`under_maintenance`, `fault`, etc.) to the `assets` table, which has
no such constraint. If anyone tries to update `asset_component_faces.operational_status` via the
repo, it will fail the DB constraint.

---

## 8. Edge Functions

### `cmms-asset-availability-check`
- Deployed and functional
- Queries `assets` table for maintenance windows and returns `pricingAttributes` and
  `audienceMetrics` (hard-coded stub values — `baseRate: 0`, `currency: 'EUR'`, etc.)
- `pricingAttributes` and `audienceMetrics` in the response are typed differently from the
  `AssetPricingAttributes` and `FaceAudienceMetrics` interfaces in `src/assets/src/types/Asset.ts`
- The Sales `SalesAssetAdapter` calls this edge function — **this is the live path Sales uses for
  availability, pricing, and audience data**
- The `AssetManagementService.checkAvailability()` and `getPricingAttributes()` methods are NOT
  used by Sales — the edge function is the actual integration point

### `cmms-update-asset-status`
- Deployed and functional
- Queries `assets.status` column (not `operational_status`) — there may be a column name
  discrepancy with the TypeScript `Asset` interface which uses `operational_status`
- Emits informal events via `hub_message_logs` insertion (not typed EventBus)
- Fires Everboarding hook on transition to `operational` via same informal mechanism
- Does NOT validate status transitions against `ASSET_STATUS_TRANSITIONS`

---

## 9. UI Completeness

| Feature | Status | Notes |
|---------|--------|-------|
| Asset list with filters | Live | CMMS Assets.tsx, direct Supabase queries |
| Asset detail view | Live | ResponsiveAssetDetail via CMMS |
| Asset component configurator | Live | AssetComponentConfigurator.tsx |
| Partner-filtered asset view | Live | AssetsByPartner.tsx |
| Admin asset list | Live | AdminAssetList.tsx |
| Bulk territory assignment | Live | BulkTerritoryAssignment.tsx |
| Status lifecycle management (UI) | Partial | cmms-update-asset-status edge function, no transition guard in UI |
| Financial records view | Missing | IAssetPort.getFinancialRecord() defined, no UI |
| Performance metrics view | Missing | IAssetPort.getPerformanceMetrics() defined, no UI |
| Risk score view | Missing | IAssetPort.getRiskScore() defined, no UI |
| Status history timeline | Missing | IAssetPort.getStatusHistory() defined, no UI |
| Bulk CSV import | Missing | Confirmed missing in v1.0, still missing |
| Photo gallery | Missing | Confirmed missing in v1.0, still missing |
| GPS map view | Missing | Confirmed missing in v1.0, still missing |
| Face-level management | Missing | AssetFace types defined, no UI component |
| Audience metrics view | Missing | Also blocked by wrong table name (§4.3 Bug 1) |
| Activation wizard (structure→frame→screen) | Missing | No UI, no EventBus wiring |

---

## 10. Discrepancies from v1.0

| # | v1.0 Claim | v2.0 Reality |
|---|-----------|--------------|
| D1 | Module described as "boundary stub — IAssetPort interface + 1 service" | Module has 11 files: full types, ports, repository, service, index, tests |
| D2 | "Activation cascade awaits UI completion" | Activation cascade has no EventBus implementation at all — architectural gap, not UI gap |
| D3 | "Six critical EventBus signals to Sales, CMMS, and Audience" | Zero EventBus signals emitted from any asset code path |
| D4 | Schema described as pending `ooh_assets` extraction | All ~30 tables confirmed in `public`; migration SQL exists but is entirely commented out |
| D5 | ISalesAssetPort described as single interface | Two diverged versions exist — 7-method in assets module, 3-method in sales module |
| D6 | "Asset management UI build in progress" | Substantial CMMS-originated UI exists and is live at `/assets/*`; it bypasses IAssetPort |
| D7 | Missing items limited to "bulk import, photo gallery, GPS map" | Also missing: financial records UI, performance metrics UI, risk score UI, status history UI, face management UI, activation wizard |
| D8 | `asset_audience_metrics` implied to exist | Table does not exist in DB; `face_audience_metrics` exists instead; `getAudienceMetrics()` broken |

---

## 11. Production Readiness Score

**Score: 4 / 10**

Rationale by dimension:

| Dimension | Score | Reason |
|-----------|-------|--------|
| Schema design | 6/10 | Tables exist and are functional; wrong schema location; `asset_component_faces` constraint diverges from canonical enum |
| TypeScript layer | 6/10 | Solid types, port, service; repo has 3 bugs (audience table, no tenant scoping on secondaries, wrong date column) |
| EventBus wiring | 1/10 | Zero emissions from the module; activation cascade entirely absent; cmms-update-asset-status uses informal log-insertion, not EventBus |
| Sales↔Assets integration | 4/10 | SalesAssetAdapter→edge function path works but returns stub data; ISalesAssetPort has diverged into two definitions |
| CMMS boundary enforcement | 2/10 | 57 direct `.from('assets')` violations; AssetManagementAdapter exists but is not used by most CMMS components |
| UI completeness | 5/10 | List, detail, configurator, partner view all live; financial/performance/risk/status-history/faces views missing; direct Supabase, not IAssetPort |
| Test coverage | 3/10 | One test file exists; pass rate unknown; no tests for edge functions; no tests for EventBus emissions (which don't exist) |
| Multi-tenancy | 5/10 | Primary queries tenant-scoped; 7 secondary table queries are not |
| Data integrity | 4/10 | `findAvailable` / `checkAvailability` stub logic produces incorrect results; `asset_audience_metrics` broken; `performance_metrics` date filter wrong |

**Summary:** The Assets module has a well-designed TypeScript layer and a working UI, but it
is not production-safe for the following reasons: (1) the EventBus activation cascade that creates
Sales inventory from physical asset events does not exist; (2) availability checking is a stub that
ignores actual bookings; (3) one broken table reference means audience metrics always throw; (4)
57 CMMS components bypass the module boundary making schema migration impossible without a major
refactor; (5) the schema extraction to `ooh_assets` has not been run.

---

## 12. Priority Remediation Items

Ordered by impact:

1. **(P0) Fix `getAudienceMetrics()` table name** — change `asset_audience_metrics` to
   `face_audience_metrics` in `AssetRepository.ts`.

2. **(P0) Add tenant scoping to all secondary table queries** — `getStatusHistory`,
   `getFaces`, `getFaceIds`, `getFinancialRecord`, `getPerformanceMetrics`, `getRiskScore`,
   `getAudienceMetrics` must all include `.eq('platform_tenant_id', tenantId)`.

3. **(P1) Implement EventBus emissions for asset lifecycle events** — `AssetService` needs to
   emit `assets.frame.activated`, `assets.screen.registered`, `assets.frame.decommissioned`,
   `assets.frame.attributes_updated` so Sales can maintain its inventory surface automatically.

4. **(P1) Fix `findAvailable()` / `checkAvailability()` to query bookings** — these methods
   must join against sales availability tables (or call the CMMS edge function) to return real
   conflicts, not a status-only filter.

5. **(P1) Fix `getPerformanceMetrics()` date column** — filter on `metric_date`, not
   `created_at`.

6. **(P1) Reconcile the two `ISalesAssetPort` definitions** — the assets-module version (7
   methods) and the sales-module version (3 methods) must be unified. The Sales module's edge
   function adapter is the active path; the assets-module version is unused. Either delete the
   assets-module version or make it the canonical source imported by Sales.

7. **(P2) Fix `asset_component_faces` status values** — the DB CHECK constraint accepts
   `failed | maintenance`; the canonical enum uses `fault | under_maintenance`. Migration
   needed to widen the constraint, or the normaliser must be applied before any write.

8. **(P2) Migrate CMMS direct `assets` queries to use `AssetManagementAdapter`** — 57
   violations. Required prerequisite for `ooh_assets` schema extraction.

9. **(P2) Run `ooh_assets` schema extraction** — uncomment and execute
   `migration_extract_assets.sql` in a branch database first, then production.

10. **(P3) Delete `react.svg`** from `src/assets/` — stray file with no purpose.
