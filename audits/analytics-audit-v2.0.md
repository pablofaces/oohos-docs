# Analytics Module Audit v2.0
**Date:** 2026-03-25
**Auditor:** Kai (AI — OOH OS embedded dev agent)
**Based on:** v1.0 audit (2026-03-23, Cascade), full codebase + schema scan

---

## Executive Summary

The v1.0 audit declared the Analytics module "Vision Only — Not Yet Built." That finding was partially accurate but substantially incomplete. Since v1.0, significant analytics scaffolding has been laid — primarily inside the CMMS sub-module, the communication hub, and the admin layer. However, the _platform-level_ Analytics module described in the product vision (cross-module, read-only, `ooh_analytics`-schema-backed, EventBus-driven) remains unbuilt. What exists is a collection of siloed, module-local analytics surfaces with no unifying data layer.

**Production Readiness: 3 / 10**

---

## What v1.0 Got Right

- `ooh_analytics` schema exists in the DB but contains **zero tables** — only the schema shell, ownership, and grants were committed (migration `20260310110217_remote_schema.sql`, lines 34–40). The comment reads: `'Analytics module — aggregated events, report definitions'`. This confirms deliberate intent with no follow-through.
- No EventBus consumers for real-time metric streaming exist anywhere in the codebase.
- No cross-module reporting dashboard (revenue vs. delivery vs. audience in one view) has been built.
- Export functionality is absent from all analytics surfaces examined.

---

## What v1.0 Missed — Fragments That Exist

### 1. Admin Analytics (Platform Layer)

**Files:**
- `src/pages/admin/AnalyticsPage.tsx` — 3-tab shell (Overview, Suppliers, Operations). Functional routing with lazy-loaded children.
- `src/pages/admin/UnifiedAnalytics.tsx` — Queries `onboarding_flows` directly via Supabase client. Calculates completion rates, avg time. Several data fetch calls are commented out with TODO stubs (cross-system queries disabled). Hardcoded fallback values present (e.g., `operationalEfficiency = 85` when no real data).
- `src/pages/admin/PlatformAnalytics.tsx` — Queries `platform_tenants`, `profiles`, `onboarding_flows`, `email_logs`. Fully wired hook. Renders KPI cards. Orphaned — not linked from `AnalyticsPage.tsx`.
- `src/pages/admin/SupplierAnalytics.tsx` — Calls `supabase.rpc('get_supplier_analytics_metrics')` and `supabase.rpc('get_supplier_type_breakdown')`. Depends on RPCs whose existence in the DB was not verified.
- `src/pages/admin/PortalAccessAnalytics.tsx` — Reads `audit_logs` for `client_portal_view` events. Uses `usePortalAccessAnalytics` hook. Renders Recharts line/pie charts.
- `src/components/admin/analytics/OperationsDashboard.tsx` — Delegates to `AnalyticsService`, `SlaMonitoringService`, `EscalationService`. Most complete service-backed dashboard found.

**Hook:** `src/hooks/repositories/usePlatformAnalytics.ts` — Live, fully implemented. Queries 4 tables in parallel. Used by `PlatformAnalytics.tsx`.

**Platform `AnalyticsService`:** `src/services/analyticsService.ts` — Full class with `getDashboardData()`, `getOnboardingMetrics()`, `getCreditMetrics()`, `getSlaMetrics()`, `getActivityTimeSeries()`, `topPerformingBrands()`. Backed by real Supabase queries against `onboarding_flows`, `credit_applications`, `sla_instances`.

### 2. CMMS Analytics (Sub-module Layer)

57 analytics-related source files discovered. Key components:

- `src/cmms/src/components/analytics/AnalyticsRouter.tsx` — Routes between `AnalyticsDashboardSimple` and `AssetPerformanceDetail`.
- `src/cmms/src/components/analytics/AnalyticsDashboard.tsx` — Full Recharts dashboard: bar charts for asset types, maintenance type trends, top-performing assets. Backed by `useAnalytics` hook.
- `src/cmms/src/components/analytics/UnifiedAnalyticsDashboard.tsx`, `PostingKPIDashboard.tsx`, `CleaningKPIDashboard.tsx`, `FlagTrendsChart.tsx`, `PoPReconciliationDashboard.tsx` — All present.
- `src/cmms/src/components/inventory/analytics/` — 4 components: `EnhancedInventoryAnalyticsDashboard`, `InventoryKPIDashboard`, `AdvancedReportBuilder`, `BusinessIntelligenceDashboard`.
- `src/cmms/src/components/assets/analytics/` — 6 components covering heatmap, flag trends, cost trends, location insights, forecast accuracy, risk section, AI suggestions.
- `src/cmms/src/hooks/analytics/useUnifiedAnalytics.ts` — Core data calls are **commented out**. Returns hardcoded/derived values with zero real data. The cross-system service calls (`services.analytics.getDashboardStats`, `services.tasks.getTasks`, `services.documents.getDocuments`) are stubbed.
- `src/cmms/src/adapters/analytics/CMMSAnalyticsAdapter.ts` — Port/Adapter pattern defined. `track()` falls back to `AuditService.log()`. `query()` is a complete stub returning empty arrays and logs to console only.
- `src/cmms/src/services/AnalyticsService.ts` — Large class but business logic delegates to `BaseRepository` stubs. Anomaly detection, rule conflict detection, batch result analysis are structurally present but unimplemented.

### 3. Communication Hub Analytics

**Files:**
- `src/communication-hub/client/src/pages/analytics/ChannelPerformancePage.tsx` — Has `@ts-nocheck` at top. Comment reads: `// TODO: Wire to Supabase when analytics tables are created`. All three query functions (`emailStats`, `smsStats`, `pushStats`) return hardcoded zero objects. Zero real data.
- `src/communication-hub/client/src/pages/analytics/ConversionsPage.tsx` — Not inspected, presumed similar.

**Edge Function:** `supabase/functions/hub-analytics/index.ts` — The most production-ready analytics code in the codebase. Serverless Deno function routing `overview` and `timeseries` sub-paths. Queries `hub_message_logs` with real tenant filtering. Proper CORS, auth via JWT claims, service-role DB client. This is functional.

### 4. Scattered Module-Local Tables (public schema)

Six analytics-related tables live in `public`, not `ooh_analytics`:
- `cleaning_performance_analytics`
- `comms_analytics_snapshots`
- `comms_email_analytics_summary`
- `document_utilization_analytics`
- `message_send_analytics`
- `onboarding_analytics_summary`

These are siloed snapshot/summary tables. No cross-module views or materialized views exist. No foreign keys link them to a central analytics fact table.

### 5. Routing

Route `/analytics/*` is registered in `AnimatedRoutes.tsx` (lines 192–427), lazy-loading `AnalyticsEntry` from `src/cmms/src/AnalyticsEntry`. The CMMS analytics router is the live entry point. The admin-layer pages are accessible via `/admin/analytics` but are separate from the main `/analytics` route.

---

## Critical Gaps (Updated from v1.0)

| Gap | v1.0 Status | v2.0 Status |
|-----|-------------|-------------|
| `ooh_analytics` schema — no tables | Confirmed | Confirmed — schema shell only |
| Cross-module dashboard (sales/CMS/audience) | Not built | Not built |
| EventBus consumers for real-time metrics | Not built | Not built |
| Export functionality | Not built | Not built |
| `useUnifiedAnalytics` — real data | N/A | Hook exists but all data calls commented out |
| `CMMSAnalyticsAdapter.query()` | N/A | Stub — returns empty array |
| `ChannelPerformancePage` Supabase wiring | N/A | TODO comment, hardcoded zeros |
| `SupplierAnalytics` RPCs | N/A | Calls RPCs whose DB presence is unverified |
| `PlatformAnalytics.tsx` routing | N/A | Page exists but orphaned from nav |
| Hardcoded metric fallbacks | N/A | Present in `UnifiedAnalytics.tsx` (e.g. `= 85`) |

---

## What Would Be Required to Build the Vision

### Phase 1 — Schema (1–2 days)
1. Create tables inside `ooh_analytics` schema: `metric_events`, `report_definitions`, `dashboard_snapshots`, `aggregated_kpis`.
2. Create cross-schema views reading from `ooh_sales`, `ooh_cms`, `ooh_audience` (these schemas need to exist).
3. Verify or create RPCs: `get_supplier_analytics_metrics`, `get_supplier_type_breakdown`.
4. Move the 6 orphaned `public.*_analytics` tables into `ooh_analytics` or build views over them.

### Phase 2 — Data Layer (2–3 days)
1. Implement `CMMSAnalyticsAdapter.query()` against real tables.
2. Uncomment and fix `useUnifiedAnalytics` service calls.
3. Wire `ChannelPerformancePage` to `hub-analytics` edge function (already built).
4. Remove hardcoded fallback values in `UnifiedAnalytics.tsx`.

### Phase 3 — Cross-Module Dashboard (3–5 days)
1. Build an `ooh_analytics` read-only API layer (edge function or DB functions).
2. Build a unified React dashboard consuming it — revenue, delivery, audience in one view.
3. Link `PlatformAnalytics.tsx` into the admin nav.

### Phase 4 — Real-time + Export (2–3 days)
1. EventBus consumers (Supabase Realtime subscriptions) feeding `ooh_analytics.metric_events`.
2. CSV/PDF export added to key dashboards.
3. Enforce read-only DB role for `ooh_analytics` schema (remove the current blanket `GRANT ALL` on tables — replace with `GRANT SELECT` for `authenticated`).

---

## Security Note

The current schema grants in migrations give `authenticated` role `ALL` privileges on `ooh_analytics` tables (line 60312–60314). The product vision specifies read-only access. This must be corrected before any tables are added to the schema.

---

## Production Readiness: 3 / 10

**Scoring rationale:**

- +1: Route exists, lazy-loaded, accessible in app.
- +1: `hub-analytics` edge function is genuinely functional (real queries, auth, CORS).
- +1: Platform `AnalyticsService` + `usePlatformAnalytics` hook are fully implemented for the onboarding/credit/SLA domain.
- -7: `ooh_analytics` schema is empty. The intended central data layer does not exist. Most dashboards return zeroes, stubs, or hardcoded values. No cross-module reporting. No real-time. No export. One page has `@ts-nocheck` with all queries hardcoded to zero. The CMMS adapter's `query()` method logs to console and returns nothing. The unified analytics hook has its core data calls commented out.

The module is best described as: **scaffolding with one working edge, six orphaned snapshot tables, and a schema placeholder waiting for a data model.**
