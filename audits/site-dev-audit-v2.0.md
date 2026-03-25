# Site Development Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai
**Basis:** Live codebase at `/workspace/group/everboarding`, live DB schema (project `gwtzedkvcnnnlbeidzar`), v1.0 audit (2026-03-23)
**Scope:** Full code + schema delta from v1.0

---

## 1. Executive Summary

The Site Development module is structurally sound and substantially built. The Programme Engine (Stage 4) is the core innovation and is correctly implemented end-to-end in service layer and UI. However three infrastructure gaps — missing DB tables required by the overdue-detection edge function, a non-transactional builder save flow, and a feature flag that gates the engine OFF by default — mean the module is not yet production-complete. The council portal and service provider portal are both fully wired into the route tree. The v1.0 audit contained two inaccuracies which are corrected below.

---

## 2. V1.0 Audit Discrepancies

### 2a. Status value naming error (schema vs audit)

The v1.0 audit stated `programme_types.status` takes values `(draft/published/archived)`. This is incorrect.

- **Migration** (`20260316135222_programme_engine_data_model.sql` line 52): `CHECK (status IN ('draft', 'active', 'archived'))`
- **`ProgrammeTypeConfigService.ts`**: queries with `.eq('status', 'active')` throughout
- **`ProgrammeTypesPage.tsx`**: publishes with `.update({ status: 'active' })` and renders badge for `case 'active'`

The value is `active`, not `published`. The audit's type definition `status: 'draft' | 'active' | 'archived'` in `ProgrammeTypeConfigService.ts` is correct and matches the DB. The v1.0 description prose was wrong.

### 2b. Route inventory error for /site-dev/cycles

The v1.0 audit listed `/site-dev/cycles` as a standalone route with a `CycleManager` component. This route does not exist.

- `SiteDevModule.tsx` registers: `index`, `requests`, `assessments`, `planning`, `programmes`, `programmes/:id`
- `SiteDevNavigation.tsx` has five nav items — no "Cycles" tab
- `CycleManager` is a **tab** inside `ProgrammeTypeBuilderPage` (`TabsContent value="cycles"`), accessed at `/site-dev/programmes/:id`

There is no standalone cycles management route. Cycle CRUD is scoped per programme type, inside the builder.

### 2c. Service count discrepancy

v1.0 claimed "16 services in `src/site-dev/services/` (15 domain services + `index.ts`)". Actual file count is **17**: 15 domain service files + `index.ts` + `SurveyIntelligence.ts` (which the audit also listed separately but did not include in its count). Count is 16 unique services: `ApplicationClosureService`, `ArchitectWarrantValidator`, `BatchApplicationService`, `DesignReviewConsentService`, `DocumentProvenanceService`, `FrameworkAgreementService`, `OverdueAssignmentService`, `PlanningRevisionService`, `PostCompletionService`, `PreScreeningService`, `ProgrammeAnalyticsService`, `ProgrammeCycleService`, `ProgrammeTypeConfigService`, `SiteRequestDocumentService`, `SurveyIntelligence`, `SurveySubmissionValidator`. The `index.ts` is a barrel, not a service.

---

## 3. Verified Working — Code + Schema Confirmed

### 3a. DB Schema (all confirmed live in production DB)

| Table | Status |
|---|---|
| `programme_types` | EXISTS |
| `programme_type_phases` | EXISTS |
| `programme_type_documents` | EXISTS |
| `programme_type_notifications` | EXISTS |
| `programme_cycles` | EXISTS |
| `site_requests` | EXISTS (with `programme_type_id`, `programme_cycle_id`, `current_phase`, `phase_status` columns per migration) |
| `service_providers` | EXISTS |
| `service_assignments` | EXISTS |
| `document_provenance` | EXISTS |
| `planning_revision_requests` | EXISTS |
| `architect_warrant_register` | EXISTS |
| `programme_application_groups` | EXISTS |
| `jurisdictions` | EXISTS |
| `partner_programmes` | EXISTS |
| `partner_agreements` | EXISTS |
| `agreement_coverage_members` | EXISTS |
| `agreement_location_instances` | EXISTS |
| `agreement_renewals` | EXISTS |
| `lifecycle_transitions` | EXISTS |
| `site_assessments` | EXISTS |
| `site_access_rules` | EXISTS |
| `site_stakeholders` | EXISTS |
| `survey_form_submissions` | EXISTS |
| `survey_photo_submissions` | EXISTS |
| `transport_authorities` | EXISTS |
| `transport_correspondence` | EXISTS |
| `transport_permit_documents` | EXISTS |
| `planning_permits` | EXISTS |
| `asset_permits` | EXISTS |

Also confirmed in live DB: RPC `save_programme_type_config` exists (visible in Supabase OpenAPI spec at `/rpc/save_programme_type_config`). This is the transactional save path identified as a known issue in v1.0 — the RPC is in the DB but **the frontend `ProgrammeTypeBuilderPage.tsx` is not calling it**. See Section 4 below.

### 3b. Source Files

All 16 services verified present in `/workspace/group/everboarding/src/site-dev/services/`. File sizes (line counts) are consistent with v1.0 claims.

All 4 builder components verified present:
- `/workspace/group/everboarding/src/site-dev/components/builder/PhaseSequencer.tsx`
- `/workspace/group/everboarding/src/site-dev/components/builder/DocumentRulesMatrix.tsx`
- `/workspace/group/everboarding/src/site-dev/components/builder/NotificationConfigEditor.tsx`
- `/workspace/group/everboarding/src/site-dev/components/builder/CycleManager.tsx`

### 3c. Routing (verified in AnimatedRoutes.tsx)

| Route | Component | Guard | Status |
|---|---|---|---|
| `/site-dev/*` | `SiteDevModule` | `RequireAuth` | Wired |
| `/provider/*` | `ServiceProviderRoutes` | `RequireAuth` + `allowedRoles: ['service_provider', 'super_admin', 'platform_admin']` | Wired |
| `/portal/councils/*` | `CouncilPortalRoutes` | `RequireAuth` | Wired |

### 3d. Programme Engine — Config-Driven Transitions

`useApplicationTransitions` lives in `/workspace/group/everboarding/src/portal/councils/hooks/useApplicationTransitions.ts` and is consumed by `ApplicationTransitionActions.tsx`. It correctly:
1. Queries `ProgrammeTypeConfigService.resolveForSiteRequest()` for programme config
2. If config transitions exist, uses them (config-driven path)
3. Falls back to `useSiteRequestLifecycle().getAvailableTransitions()` (legacy hardcoded path)
4. Filters `FACES_ONLY_TRANSITIONS` from council-visible actions

The feature flag `FEATURE_LIFECYCLE_ENGINE_SITE_REQUEST` defaults **OFF** (`false`). This means programme_type_id-bearing site requests currently **block all transitions** (the guard in `useSiteRequestLifecycle.canTransition()` returns `false` with a console warning if `programmeTypeId` is set but the engine is disabled). This is intentional during soak period, but means the full engine is not yet operationally live.

### 3e. Service Provider Portal

`ServiceProviderRoutes.tsx` is a clean lazy-loaded route tree:
- `/provider` → `ProviderDashboard`
- `/provider/assignments/:assignmentId` → `ProviderAssignmentDetail`

Both pages exist in `/workspace/group/everboarding/src/portal/providers/pages/`. Role gate in AnimatedRoutes.tsx enforces `service_provider | super_admin | platform_admin`.

### 3f. Council Portal

`CouncilPortalRoutes` (`/portal/councils/*`) has 12 pages fully registered. All pages exist in `/workspace/group/everboarding/src/portal/councils/pages/`. `ApplicationTransitionActions.tsx` and `AssignServiceProviderSection.tsx` integrate with the programme engine (reads `current_phase` column from `site_requests`). `SignaturePendingBanner`, `AgreementSection`, `PlanningRevisionSection`, and `ClosureReasonSection` all present as named components.

---

## 4. Known Issues (Confirmed by Code Inspection)

### 4a. Non-Transactional Programme Type Builder Save — NOT YET FIXED

v1.0 flagged this as a known issue. As of this audit it remains unfixed.

`ProgrammeTypeBuilderPage.tsx` (lines 206–275) uses a sequential delete-then-insert pattern across three separate Supabase calls (phases, doc rules, notifications). The RPC `save_programme_type_config` exists in the live DB but is not called by the frontend. A partial failure mid-save leaves the DB in a partial state. The hydration guard (`useRef`) only prevents React Query from overwriting unsaved UI state — it does not prevent DB corruption.

**Impact:** Data integrity risk on every save of a programme type. Low probability in practice (network partition mid-sequence), but unacceptable for a published production type managing live applications.

**Fix path:** Wire `ProgrammeTypeBuilderPage` to call `supabase.rpc('save_programme_type_config', {...})` instead of the three-step delete-insert. The RPC is already deployed.

### 4b. 104 TypeScript `as any` Casts in Services

`grep -rn "as any"` across `src/site-dev/services/` returns 104 hits. All programme engine tables (`programme_cycles`, `service_assignments`, `architect_warrant_register`, etc.) are cast via `(supabase as any).from('table_name' as any)`. This indicates the Supabase generated TypeScript client types (`supabase/types.ts`) have not been regenerated since the programme engine migration was applied. Type safety for all programme engine DB interactions is bypassed.

**Impact:** No runtime risk, but type errors are silenced. Future regressions will not be caught by the compiler.

**Fix path:** Run `supabase gen types typescript --project-id gwtzedkvcnnnlbeidzar > src/integrations/supabase/types.ts` (from host machine) to regenerate types.

### 4c. Feature Flag Defaults OFF — Engine Not Operationally Active

`FEATURE_LIFECYCLE_ENGINE_SITE_REQUEST` defaults `false`. Site requests with `programme_type_id` set will have transitions blocked entirely. The flag is stored in the DB (`feature_flags` table, seeded in migration) but must be enabled per-tenant before the programme engine goes live.

**Impact:** The Programme Engine is deployed but not active. Any site request assigned to a programme type will be stuck — no transitions possible.

**Action required:** Once soak period is complete, enable the flag via the feature flags admin UI or a direct DB update for the Malta tenant.

---

## 5. Missing Items

### 5a. DB Tables Missing — Edge Function Will Fail

The `detect-site-request-overdue` edge function references three tables that do not exist in the live DB:

| Table Referenced | Status | Effect |
|---|---|---|
| `jurisdiction_configs` | MISSING | Edge function query at line 107 will return empty/error — falls back to `DEFAULT_THRESHOLDS` (workable fallback) |
| `business_calendars` | MISSING | Line 124 query will error — working-day calculation breaks, falls back to calendar-day counting |
| `site_request_overdue_log` | MISSING | Line ~150 insert will throw — the entire detection run will fail silently (error caught per-row) |

The migration `20260311161000_site_request_overdue_detection.sql` creates `site_request_overdue_log` but it has not been applied. `jurisdiction_configs` and `business_calendars` appear to be planned but no migration file exists for them in `/workspace/group/everboarding/supabase/migrations/`.

**Impact:** The cron-scheduled overdue detection (daily at 08:00 UTC) is silently failing or producing degraded results. Overdue site requests are not being flagged or notified.

### 5b. Billboard Negotiated Programme Type (Stage 5)

No `billboard_negotiated` programme type seed data exists. This was flagged in v1.0 as Stage 5 validation work. No progress made. The engine cannot be considered validated for multi-jurisdiction use until a second distinct programme type is configured and exercised.

### 5c. Drag-and-Drop Phase Reordering

`PhaseSequencer.tsx` uses `ChevronUp`/`ChevronDown` button-based reordering. The `GripVertical` icon is imported and rendered (visible in the UI as a visual affordance) but there is no `dnd-kit` integration. Users can reorder phases but it is clunky for long phase sequences.

### 5d. Programme Type Versioning / Diff Tracking

No immutable-published / mutable-draft version diffing is implemented. The `version` column exists on `programme_types` but is a static string (defaults `'1.0.0'`), not auto-incremented on publish. Editing and republishing a programme type overwrites the previous published version without creating a new immutable record. Live applications that pin `programme_type_version` on `site_requests` could reference a config that has been mutated.

**Impact:** Active applications could silently shift to a modified config on next load. This is a data integrity risk for long-running programme types.

---

## 6. Council Portal Status

| Area | Status | Notes |
|---|---|---|
| Route tree | Complete | 12 routes, all pages present, role guards applied |
| `ApplicationTransitionActions` | Working | Uses config-driven transitions with legacy fallback |
| `AssignServiceProviderSection` | Working | Reads `current_phase` from `site_requests` correctly |
| `SurveyReviewPage` | Present | Role-gated to `partner_technical_officer` |
| `FrameworkAgreementsPage` | Present | Admin-only (`faces_admin`) |
| `PrechecksPage` | Present | Role-gated to `partner_permit_officer` |
| `AppealPage` | Present | Role-gated to `partner_signatory` |
| `FinancialsPage` | Present | Role-gated to `partner_finance_officer | partner_signatory` |
| `SignaturePendingBanner` | Present as component | Not verified if it surfaces in ApplicationDetailPage |
| `AgreementSection` | Present as component | Renders framework agreement per application |
| Pending signatures view | Exists via `SignaturePendingBanner` component | No dedicated `/pending-signatures` route |

**Gap:** The v1.0 audit mentioned "council portal — site request review + pending signatures" under Programme Type Builder integration. There is no dedicated pending-signatures page/route. The `SignaturePendingBanner` is a component rendered within `ApplicationDetailPage` context, not a queue/list view. Councils cannot see all pending-signature items across applications in one place.

---

## 7. Programme Engine Completeness

| Component | Built | Operational | Notes |
|---|---|---|---|
| `ProgrammeTypeConfigService` | Yes | Yes | 16 services, all present |
| Config-driven transitions hook | Yes | Blocked by feature flag | Will activate when flag enabled |
| Legacy fallback transitions | Yes | Active | Working for flag-OFF state |
| Lifecycle effects (6 effects) | Yes | Blocked by flag | In `useSiteRequestLifecycle` new-engine path |
| `lifecycle_transitions` audit writes | Yes | Active in legacy path | Both paths write audit rows |
| `OverdueAssignmentService` | Yes | Degraded | Edge function missing 3 DB tables |
| Builder UI (4-tab) | Yes | Yes | Save non-transactional (known issue) |
| `save_programme_type_config` RPC | Deployed in DB | Not called | Frontend wiring missing |
| Malta programme type seed | Yes | Depends on flag | Config data present; engine activation pending |
| Billboard programme type | No | No | Stage 5 not started |
| Phase reorder (dnd-kit) | No | Partial (buttons work) | Visual affordance without implementation |
| Version diff / immutable publish | No | No | Data integrity risk for active applications |
| `jurisdiction_configs` table | No | No | Planned, no migration exists |
| `business_calendars` table | No | No | Planned, no migration exists |

---

## 8. Production Readiness

**Score: 6 / 10**

| Dimension | Score | Rationale |
|---|---|---|
| Service layer completeness | 9/10 | 16 services, all domain logic implemented, well-structured |
| DB schema | 8/10 | All core tables live; 3 supporting tables missing |
| UI completeness | 7/10 | All primary routes present; cycles route non-existent (embedded); no DnD; pending-signatures list absent |
| Type safety | 4/10 | 104 `as any` casts — generated types out of date with schema |
| Data integrity | 5/10 | Non-transactional builder save + no programme type versioning |
| Operational readiness | 4/10 | Feature flag OFF blocks engine; overdue detection silently failing |
| Council portal | 8/10 | Fully wired, role-gated, 12 pages; missing dedicated pending-signatures queue |
| Service provider portal | 9/10 | Clean, role-gated, both routes wired |
| Test coverage | Unknown | No test files found under `src/site-dev/` during audit |
| EventBus / integrations | 7/10 | Events designed; Sales/Assets/Compliance consumers confirmed designed but not verified deployed |

**Blockers before production:**
1. Apply migration `20260311161000_site_request_overdue_detection.sql` (creates `site_request_overdue_log`)
2. Create migrations for `jurisdiction_configs` and `business_calendars`
3. Wire `ProgrammeTypeBuilderPage` to `save_programme_type_config` RPC
4. Regenerate Supabase TypeScript types
5. Enable `FEATURE_LIFECYCLE_ENGINE_SITE_REQUEST` flag per tenant after soak confirmation

**Not blocking but required before GA:**
- Billboard negotiated programme type (Stage 5 validation)
- Programme type version immutability strategy
- DnD phase reorder
- Pending-signatures queue in council portal

---

## 9. File Reference Map

| Area | Path |
|---|---|
| Services | `/workspace/group/everboarding/src/site-dev/services/` |
| Builder pages | `/workspace/group/everboarding/src/site-dev/pages/ProgrammeTypesPage.tsx`, `ProgrammeTypeBuilderPage.tsx` |
| Builder components | `/workspace/group/everboarding/src/site-dev/components/builder/` |
| Module root | `/workspace/group/everboarding/src/site-dev/pages/SiteDevModule.tsx` |
| Lifecycle hook | `/workspace/group/everboarding/src/site-dev/hooks/useSiteRequestLifecycle.ts` |
| Transitions hook | `/workspace/group/everboarding/src/portal/councils/hooks/useApplicationTransitions.ts` |
| Council portal routes | `/workspace/group/everboarding/src/portal/councils/routes.tsx` |
| Council portal pages | `/workspace/group/everboarding/src/portal/councils/pages/` |
| Provider portal routes | `/workspace/group/everboarding/src/portal/providers/ServiceProviderRoutes.tsx` |
| Provider portal pages | `/workspace/group/everboarding/src/portal/providers/pages/` |
| Main route tree | `/workspace/group/everboarding/src/components/layout/AnimatedRoutes.tsx` |
| Edge function (overdue) | `/workspace/group/everboarding/supabase/functions/detect-site-request-overdue/index.ts` |
| Migrations | `/workspace/group/everboarding/supabase/migrations/20260316135222_programme_engine_data_model.sql` |
| | `/workspace/group/everboarding/supabase/migrations/20260316150802_agreement_architecture_schema.sql` |
| | `/workspace/group/everboarding/supabase/migrations/20260311161000_site_request_overdue_detection.sql` (not yet applied) |
