# OOH OS — Master Implementation Plan
**Version:** 1.0
**Date:** 2026-03-26
**Author:** Kai (Autonomous Development Agent)
**Scope:** All 20 modules, post v2.0 audit findings
**Objective:** Single-operator production go-live (Malta/Faces) → International multi-operator platform

---

## 1. Platform Snapshot

| Metric | Value |
|---|---|
| Total source files | 5,729 |
| Edge Functions deployed | 174 |
| DB schemas | 8 (public + 7 ooh_*) |
| DB tables (public) | 555 |
| DB tables (ooh_sales) | 57 |
| DB tables (ooh_cms) | 13 |
| Agents registered | 28 platform + 7 CMMS-local |
| Migrations committed | 74 |
| Modules audited | 20 |

**Composite platform readiness: 5.8 / 10**

---

## 2. P0 Bug Registry — Fix Before Any Go-Live

These are confirmed production-breaking defects. Nothing ships until all P0s are resolved.

### P0-001 — Sales Pricing UI Completely Broken
- **File:** `src/sales/src/hooks/usePricing.ts`
- **Bug:** Queries `supabase.from('sales_rate_card')` without `.schema('ooh_sales')` — hits public schema where Sales tables don't exist. Returns empty data on every call.
- **Secondary:** Queries `sales_pricing_rule` (table name is `sales_pricing_rules` — plural).
- **Impact:** `/sales/pricing` shows no data. PricingRulesEngine has no rate cards, no corridors, no pricing rules. Every booking priced via the UI gets no pricing data.
- **Fix:** Rewrite `usePricing` to use `supabase.schema('ooh_sales').from(...)` on all queries, OR refactor to use `PricingRepository` (already correct). Fix table name to `sales_pricing_rules`.
- **Effort:** 1 hour

### P0-002 — DMS New Module Approval Operations Broken
- **File:** `src/dms/src/repositories/DocumentRepository.ts` lines 143, 155, 166
- **Bug:** Queries `.from('document_approvals')` — table does not exist. Canonical table is `document_approval_workflow`.
- **Impact:** All approval operations through the new DMS standalone module throw PostgREST errors at runtime.
- **Fix:** 3-line change: `'document_approvals'` → `'document_approval_workflow'`.
- **Effort:** 15 minutes

### P0-003 — GDPR Erasure Inverted Retention Check
- **File:** `src/compliance/gdpr/gdprService.ts` ~line 417
- **Bug:** `gte('created_at', sevenYearsAgo)` selects records NEWER than 7 years — it should be `lte` to select records OLDER than 7 years. The financial retention blocker is inverted: it blocks erasure for NEW records and allows it for old records.
- **Impact:** Regulatory exposure. The platform is erasing data it should retain and retaining data it should erase.
- **Fix:** Change `gte` to `lte`.
- **Effort:** 5 minutes

### P0-004 — GDPR DSR Processing Split — Two Tables, No Bridge
- **Files:** `src/compliance/gdpr/gdprService.ts` vs `supabase/functions/submit-dsr-request/`
- **Bug:** `submit-dsr-request` edge function writes to `gdpr_data_subject_requests`. `GdprService` processes from `gdpr_requests`. Admin UI reads `gdpr_data_subject_requests`. These are two different tables with no synchronisation. DSRs submitted via portal never reach the processing code.
- **Impact:** GDPR compliance failure — submitted DSRs appear in admin queue but are never processed.
- **Fix:** Migrate `GdprService.processAccessRequest()`, `processErasureRequest()`, etc. to use `gdpr_data_subject_requests`. Drop or archive `gdpr_requests`. Add the bridge field (`processing_status`) to `gdpr_data_subject_requests` if needed.
- **Effort:** 1 day

### P0-005 — ProgrammaticGateway: All State In-Memory
- **File:** `src/cms/programmatic/ProgrammaticGateway.ts`
- **Bug:** All SSP registrations, programmatic wins, reconciliation results, OpenDirect orders are stored in private arrays. Every container restart loses all state. `cms_programmatic_win` and `cms_ssp_impression_reconciliation` tables exist but are never written.
- **Edge Function:** `action=bid-request` and `action=reconcile` are explicit stubs returning hardcoded responses.
- **Impact:** Zero durable programmatic revenue tracking. Yield analysis operates on empty arrays.
- **Fix:** Add Supabase client to `ProgrammaticGateway` constructor. Replace in-memory stores with DB reads/writes. Replace Edge Function stubs with real method calls.
- **Effort:** 3 days

### P0-006 — Compliance Gate Bypassed for All Bookings
- **Files:** Missing — `compliance.checkFrameEligibility()` does not exist anywhere
- **Bug:** The Sales compliance gate described in CLAUDE.md §18.1 ("No booking can be confirmed if Compliance returns a block for any line item") has no implementation. The function is documented as a contract but there is no TypeScript file, no edge function, no stub.
- **Impact:** All bookings pass compliance unchecked. Restricted inventory can be booked for any advertiser category.
- **Fix (Phase 1 stub):** Create `src/sales/src/stubs/compliance.stub.ts` that returns all frames eligible. Wire it into `BookingService.confirmBooking()`. The stub pattern allows a real rules engine later.
- **Effort:** 2 hours for stub, 2+ weeks for full rules engine

### P0-007 — Portal Documents Page Unreachable
- **File:** `src/components/layout/AnimatedRoutes.tsx`
- **Bug:** `/portal/documents` is defined as `<Navigate to="/portal/settings/tasks" />` — routes to the wrong page. `DocumentsPage.tsx` (6,668 bytes) exists and is built but has no route.
- **Impact:** Clients navigating to Documents (via email links, nav bar) land on Task Settings. Invoice download, document management inaccessible via direct URL.
- **Fix:** Change `/portal/documents` to render `DocumentsPage` instead of redirect.
- **Effort:** 10 minutes

### P0-008 — Portal Support Page Unreachable
- **File:** `src/components/layout/AnimatedRoutes.tsx`
- **Bug:** `/portal/support` redirects to `/portal` (dashboard). `SupportPage.tsx` (9,023 bytes) exists but is orphaned.
- **Impact:** Any nav link or email pointing to `/portal/support` silently drops clients at the dashboard.
- **Fix:** Change `/portal/support` to render `SupportPage`.
- **Effort:** 10 minutes

### P0-009 — Ever-boarding: Writes to onboarding_flows VIEW Fail
- **File:** DB schema — `public.onboarding_flows` is a READ-ONLY VIEW over `ooh_everboarding.onboarding_flows`
- **Bug:** Any stored procedure or new migration that does `UPDATE public.onboarding_flows SET ...` will fail at runtime. No INSTEAD OF trigger exists. This is an active developer footgun — the natural thing to do fails silently or throws.
- **Impact:** Every new migration that touches `onboarding_flows` via the public path is a latent data corruption risk.
- **Fix (immediate):** Add a comment on the view: `COMMENT ON VIEW public.onboarding_flows IS 'READ-ONLY — writes must target ooh_everboarding.onboarding_flows'`. Add an INSTEAD OF UPDATE/INSERT/DELETE trigger in a migration.
- **Effort:** 2 hours for trigger

### P0-010 — Assets Module: Wrong Table Name in getAudienceMetrics
- **File:** `src/assets/src/repositories/AssetRepository.ts`
- **Bug:** `getAudienceMetrics()` queries `.from('asset_audience_metrics')` — table does not exist. The DB table is `face_audience_metrics`. This throws an exception on every call.
- **Impact:** Any caller of `IAssetPort.getAudienceMetrics()` gets an exception rather than data.
- **Fix:** Change table name in the query.
- **Effort:** 5 minutes

### P0-011 — Assets: 7 Secondary Queries Not Tenant-Scoped
- **File:** `src/assets/src/repositories/AssetRepository.ts`
- **Bug:** `getStatusHistory`, `getFaces`, `getFaceIds`, `getFinancialRecord`, `getPerformanceMetrics`, `getRiskScore`, `getAudienceMetrics` all query without `platform_tenant_id` filter.
- **Impact:** Cross-tenant data leak for any multi-tenant deployment.
- **Fix:** Add `.eq('platform_tenant_id', tenantId)` to all 7 methods.
- **Effort:** 30 minutes

### P0-012 — CMS Init: Hardcoded Europe/Malta Timezone
- **File:** `src/cms/init.ts` line 24
- **Bug:** `defaultTimezone: 'Europe/Malta'` — single-tenant bias in the scheduling engine initialisation.
- **Impact:** All CMS scheduling for any non-Malta tenant uses the wrong timezone.
- **Fix:** Resolve from `tenant_business_rules` at startup.
- **Effort:** 30 minutes

### P0-013 — Signature Status Inconsistency: voided vs cancelled
- **File:** `supabase/functions/void-signature-request/index.ts` sets `status: "cancelled"`. `src/signatures/services/SignatureService.ts` sets `status: 'voided'`.
- **Bug:** Two different status values for the same state. Dashboard filters will render inconsistently.
- **Fix:** Standardise on one value (recommend `voided`). Update edge function to use `voided`.
- **Effort:** 30 minutes

### P0-014 — DMS Classifier Hardcoded to Maltese Bus Shelters
- **File:** `supabase/functions/dms-intelligence-processor/index.ts`
- **Bug:** The LLM system prompt reads: "You are a document classifier for Maltese bus shelter planning applications." All document classification through DMS Intelligence Processor uses this wrong domain context.
- **Impact:** Classification accuracy for any non-Malta or non-planning-application document will be degraded.
- **Fix:** Remove or parametrise the hardcoded domain context. Make it a caller-supplied parameter.
- **Effort:** 1 hour

### P0-015 — CMMS UniluminAdapter: Registered but Non-Functional
- **File:** `src/cmms/src/adapters/devices/UniluminAdapter.ts`
- **Bug:** All methods in `UniluminAdapter` throw `not yet implemented`. The adapter is registered in `DeviceAdapterRegistry` at startup, so Unilumin devices silently get a non-functional adapter.
- **Impact:** Any production deployment with Unilumin screens fails all device commands.
- **Fix (Option A):** Complete the `UniluminAdapter` implementation. **Fix (Option B):** Remove from `DeviceAdapterRegistry` until implemented — throw explicitly if someone tries to use it.
- **Effort:** Option B = 15 minutes; Option A = 3-5 days

---

## 3. P1 Defects — Production Quality (Sprint 1-2)

### AUTH
- P1-A01: Activate MFA enforcement for `platform_admin` and above
- P1-A02: Wire SSO config UI to `tenant_sso_configs` table
- P1-A03: Build session revocation admin UI (`user_sessions` table exists)
- P1-A04: Audit all code referencing old role names (`sales_admin`, `portal_client`, etc.)

### SALES
- P1-S01: Audit all Sales hooks (`usePipeline`, `useProposal`, `useBookings`) for same schema bypass pattern as `usePricing`
- P1-S02: Write migration for `sales_commission_setting` and `sales_seasonal_rate` (in v1.0 docs, not in migrations)
- P1-S03: Add PostGIS `geography(POINT)` to `sales_location` for spatial queries
- P1-S04: Wrap bulk booking operations in DB transactions
- P1-S05: Build Commission UI (service + schema exist, no UI)
- P1-S06: Fix `ProgrammaticService.ts` column name mismatches vs migration DDL (8 fields wrong)

### PORTAL
- P1-P01: Fix self-referencing redirect at `/portal/settings` (infinite redirect loop risk)
- P1-P02: Move `RequireAuth requireOnboardingComplete` off `/portal/onboarding-status` and `/portal/resolve-blockers` (catch-22: those pages must be accessible to users who haven't completed onboarding)
- P1-P03: Audit all Comms Hub email notification links for old portal route paths

### E-SIGNATURES
- P1-E01: Make reminder schedule (max 2, 24h gap) tenant-configurable via `tenant_business_rules`
- P1-E02: Make OTP expiry (hardcoded 10 min) tenant-configurable
- P1-E03: Add scheduled batch expiry for `signature_recipients`
- P1-E04: Complete template field placement persistence
- P1-E05: Register `SignatureAnalysisAgent` in AgentHub

### COMPLIANCE
- P1-C01: Remove Malta-specific hardcoding from `check-deletion-eligibility` — move to `tenant_business_rules`
- P1-C02: Replace djb2 consent hash with SHA-256 in `GdprService.captureConsentWithSnapshot()`
- P1-C03: Implement DSR 30-day deadline cron/alerting
- P1-C04: Extend GDPR erasure to cover `comms_consent`, `sales_contact`, `sales_account`
- P1-C05: Wire `DataRetentionService` to read policies from `data_retention_policies` DB table

### CMS
- P1-CMS01: Wire 7 unwired inbound EventBus subscriptions in `src/cms/init.ts`
- P1-CMS02: Persist ProgrammaticGateway state to DB (part of P0-005)
- P1-CMS03: Fix `StaticOpsService` to write posting confirmations to `cms_posting_confirmation`
- P1-CMS04: Make SSP reconciliation threshold tenant-configurable
- P1-CMS05: Implement BrightSign RSA/ECDSA signature verification (currently presence-check only)

### CMMS
- P1-CMMS01: Implement `WeatherPort` concrete class (OpenWeather API key infra already built)
- P1-CMMS02: Take `CampaignIntegrationService` out of shadow mode — wire to live Sales data
- P1-CMMS03: Update ARCHITECTURE.md agent registry to match actual agent names
- P1-CMMS04: Complete or remove `UniluminAdapter` (P0-015)

### DMS
- P1-D01: Implement document-level TTL enforcement in `enforce-retention-policies` or new function
- P1-D02: Uncomment `search_path` ALTER in `20260419000000_extract_dms_to_dms_schema.sql`
- P1-D03: Replace read-only views with updatable views before applying schema migration
- P1-D04: Delete or archive planning draft `src/dms/schema/migration_extract_dms.sql`

### AGENTHUB
- P1-AG01: Resolve dual `LearningEngine.ts` — clarify which is canonical (hub vs learning/)
- P1-AG02: Audit and deprecate 7 legacy `ai_*` tables from pre-v4 architecture
- P1-AG03: Wire prompt template performance tracking into LearningEngine
- P1-AG04: Implement TypeScript files for 5 CMS + 3 DMS + 3 Ever-boarding agents (currently DB-registered but no code)
- P1-AG05: Add idempotency guard on Mode2FailureHandler cancellation events

### COMMS HUB
- P1-CH01: Migrate `hub_templates` to UUID PK + UUID tenant_id (affects 8 FK references)
- P1-CH02: Move remaining `hub_*` tables to `ooh_comms` schema
- P1-CH03: Wire `ABTestingPage` to existing schema (1-2 day effort)
- P1-CH04: Fix stale TODO in `FlowBuilderPage` — wire to `hub_flow_definitions` (table exists)
- P1-CH05: Enforce `per_day` rate limit in `comms-send-broadcast`

### HELPDESK
- P1-HD01: Regenerate Supabase TypeScript types — all DB calls currently use `(supabase as any)`
- P1-HD02: Add `live_chat` to `TicketSource` enum
- P1-HD03: Build Sales EventBus inbound handlers for 4 dispute/shortfall events
- P1-HD04: Confirm AgentHub registration for 3 helpdesk agents + OpMemory write path
- P1-HD05: Add server-side SLA breach detection (Edge Function cron or DB trigger)

### EVERBOARDING
- P1-EB01: Add INSTEAD OF trigger on `public.onboarding_flows` view (or add view comment as interim)
- P1-EB02: Resolve dual token stores (`magic_tokens` table vs embedded column on `onboarding_flows`)
- P1-EB03: Add graceful degradation if `LOVABLE_API_KEY` is absent in `onboarding-intelligence`
- P1-EB04: Build PO workflow FM step (CLAUDE.md §6.10)

### ASSETS
- P1-AS01: Implement EventBus emissions: `assets.frame.activated`, `assets.screen.registered`, `assets.frame.decommissioned`, `assets.frame.attributes_updated`
- P1-AS02: Fix `findAvailable()` and `checkAvailability()` to check actual bookings (currently status-only filter)
- P1-AS03: Fix `getPerformanceMetrics()` date column (`metric_date` not `created_at`)
- P1-AS04: Unify the two `ISalesAssetPort` definitions (7-method in assets vs 3-method in sales)
- P1-AS05: Migrate CMMS's 57 direct `assets` table queries to use `AssetManagementAdapter`

### AUDIENCE
- P1-AU01: Write and commit migration for 8 undocumented audience tables in live DB
- P1-AU02: Refactor `sales-price-proposal` Step 4 to use `SalesAudienceAdapter` instead of direct table query
- P1-AU03: Create `audience.stub.ts` for isolated Sales development

### SITE DEV
- P1-SD01: Apply migration `20260311161000_site_request_overdue_detection.sql` (creates `site_request_overdue_log`)
- P1-SD02: Create migrations for `jurisdiction_configs` and `business_calendars`
- P1-SD03: Wire `ProgrammeTypeBuilderPage` to call `save_programme_type_config` RPC instead of 3-step delete-insert
- P1-SD04: Regenerate Supabase TypeScript types (104 `as any` casts)
- P1-SD05: Enable `FEATURE_LIFECYCLE_ENGINE_SITE_REQUEST` flag after soak confirmation

### PARTNERS
- P1-PA01: Reconcile `FrameworkAgreementsPage` with new agreement architecture schema
- P1-PA02: Add "Activate" button calling `partners-activate-agreement` edge function
- P1-PA03: Create scheduled edge function for `RenewalService.processUpcomingRenewals()`
- P1-PA04: Handle `linkToSiteRequest` error in `LocationInstanceService`

### ANALYTICS
- P1-AN01: Create tables in `ooh_analytics` schema: `metric_events`, `report_definitions`, `dashboard_snapshots`, `aggregated_kpis`
- P1-AN02: Wire `ChannelPerformancePage` to `hub-analytics` edge function (already built)
- P1-AN03: Uncomment `useUnifiedAnalytics` data calls and fix them
- P1-AN04: Remove hardcoded fallback values from `UnifiedAnalytics.tsx`
- P1-AN05: Verify/create `get_supplier_analytics_metrics` and `get_supplier_type_breakdown` RPCs

---

## 4. P2 Enhancements — Competitive Edge (Sprint 3-6)

| ID | Enhancement | Module |
|---|---|---|
| P2-001 | Audience Phase 2 gravity model (NSO census adapter + Python engine) | Audience |
| P2-002 | SSP HTTP adapters: Hivestack + Broadsign Reach (outbound bid response + win parsing) | Programmatic |
| P2-003 | OpenDirect 2.1 HTTP endpoint surface | Programmatic |
| P2-004 | Cross-operator programmatic layer (Phase 3c) | Marketplace |
| P2-005 | Auth0 SSO activation for enterprise tenants | Auth |
| P2-006 | Agent autonomy advancement (180-day track record required — start clock now) | AgentHub |
| P2-007 | Real-time delivery WebSocket for Portal (polling → realtime) | Portal |
| P2-008 | eIDAS qualified signature support (DocuSign/PandaDoc adapter for EU QES) | E-Signatures |
| P2-009 | Content restrictions spatial check (proximity radius in filterFramesByCategory) | Partners/Compliance |
| P2-010 | Programme type version immutability / diff tracking | Site Dev |
| P2-011 | A/B testing UI fully wired to existing schema | Comms Hub |
| P2-012 | Agency multi-client portal | Portal |
| P2-013 | Commission UI + payment reconciliation | Sales |
| P2-014 | Flighted campaigns UI + shared goals | Sales |
| P2-015 | Knowledge base article version history | Helpdesk |
| P2-016 | GPS map view for assets | Assets |
| P2-017 | Bulk CSV import for assets | Assets |
| P2-018 | Analytics cross-module dashboard (revenue + delivery + audience) | Analytics |
| P2-019 | Ooh_assets schema extraction (57 CMMS violations must be fixed first) | Assets |
| P2-020 | `check-mode2-slas` cron wired + Mode2 KYB self-qualification flow | AgentHub/Marketplace |

---

## 5. Sprint Plan — Track 1: Malta Single-Operator Go-Live

**Target: Production-ready for Faces Displays Limited**

### Sprint 1 — Bug Blitz (2 weeks)
**Goal:** All P0s resolved. Core commercial flows unblocked.

| Task | Owner | P0 Ref |
|---|---|---|
| Fix `usePricing` schema + table name | Dev | P0-001 |
| Fix `DocumentRepository` table name | Dev | P0-002 |
| Fix GDPR erasure `gte` → `lte` | Dev | P0-003 |
| Fix GDPR DSR split (migrate to single table) | Dev | P0-004 |
| Fix portal documents route | Dev | P0-007 |
| Fix portal support route | Dev | P0-008 |
| Fix signature `voided` vs `cancelled` status | Dev | P0-013 |
| Fix DMS classifier system prompt | Dev | P0-014 |
| Remove `UniluminAdapter` from live registry OR implement it | Dev | P0-015 |
| Fix Assets `getAudienceMetrics` table name | Dev | P0-010 |
| Add tenant scoping to 7 Assets secondary queries | Dev | P0-011 |
| Create compliance stub (`checkFrameEligibility`) | Dev | P0-006 |
| Fix CMS init timezone | Dev | P0-012 |
| Add INSTEAD OF trigger on `onboarding_flows` VIEW | Dev | P0-009 |

**Sprint 1 Deliverable:** All P0 bugs patched. Push PRs to main. Request user run `supabase db push` for schema changes.

---

### Sprint 2 — Auth, Sales, Portal Hardening (2 weeks)

**Goal:** Commercial flows end-to-end validated. Portal client experience solid.

| Task | Ref |
|---|---|
| Regenerate Supabase TypeScript types (Site Dev + Helpdesk `as any` casts) | P1-SD04, P1-HD01 |
| Build Commission UI | P1-S05 |
| Fix ProgrammaticService column mismatches | P1-S06 |
| Audit Sales hooks for schema bypass (usePipeline, useProposal) | P1-S01 |
| Wire `save_programme_type_config` RPC in Programme Builder | P1-SD03 |
| Apply `site_request_overdue_detection.sql` migration | P1-SD01 |
| Fix `portal/settings` self-referencing redirect | P1-P01 |
| Fix PortalRoute `requireOnboardingComplete` catch-22 | P1-P02 |
| Wire 7 CMS EventBus subscriptions | P1-CMS01 |
| Implement WeatherPort concrete class | P1-CMMS01 |
| Complete GDPR DSR chain (completion notification, deadline enforcement) | P1-C03, P1-C04 |
| Remove Malta hardcoding from `check-deletion-eligibility` | P1-C01 |
| Resolve dual `LearningEngine` files | P1-AG01 |
| Deprecate legacy `ai_*` tables | P1-AG02 |

---

### Sprint 3 — Comms, Helpdesk, Signatures Hardening (2 weeks)

**Goal:** Client communication channels and support flows production-quality.

| Task | Ref |
|---|---|
| Migrate `hub_templates` to UUID PK + UUID tenant_id | P1-CH01 |
| Wire `ABTestingPage` to schema | P1-CH03 |
| Fix `FlowBuilderPage` stale TODO | P1-CH04 |
| Enforce per_day rate limit in broadcast | P1-CH05 |
| Build helpdesk Sales EventBus handlers | P1-HD03 |
| Add SLA breach detection cron | P1-HD05 |
| Register helpdesk agents in AgentHub + verify OpMemory writes | P1-HD04 |
| Make signature reminder schedule + OTP expiry tenant-configurable | P1-E01, P1-E02 |
| Register `SignatureAnalysisAgent` in AgentHub | P1-E05 |
| Complete signature template field placement | P1-E04 |
| Replace consent hash with SHA-256 | P1-C02 |
| Build PO workflow FM step | P1-EB04 |

---

### Sprint 4 — Programmatic Gateway + DMS (2 weeks)

**Goal:** Programmatic revenue tracking durable. DMS production-safe.

| Task | Ref |
|---|---|
| Implement ProgrammaticGateway DB persistence (wins + reconciliation) | P0-005 |
| Replace Edge Function stubs with real calls | P0-005 |
| Add screen registration DB table and migration | P0-005 |
| Apply `dms` schema migration (with search_path fix + updatable views) | P1-D01-D04 |
| Implement document-level TTL enforcement | P1-D01 |
| Wire `DataRetentionService` to DB policy tables | P1-C05 |
| Write Audience uncommitted migration | P1-AU01 |
| Fix `sales-price-proposal` Step 4 to use adapter | P1-AU02 |

---

### Sprint 5 — Assets, Partners, Site Dev (2 weeks)

**Goal:** Physical inventory flow from asset activation to Sales frame creation. Partners agreement management end-to-end.

| Task | Ref |
|---|---|
| Implement Assets EventBus emissions | P1-AS01 |
| Fix `findAvailable` / `checkAvailability` to check bookings | P1-AS02 |
| Reconcile `FrameworkAgreementsPage` with new schema | P1-PA01 |
| Add agreement activation button | P1-PA02 |
| Create RenewalService scheduler | P1-PA03 |
| Create `jurisdiction_configs` + `business_calendars` migrations | P1-SD02 |
| Enable `FEATURE_LIFECYCLE_ENGINE_SITE_REQUEST` flag | P1-SD05 |

---

### Sprint 6 — Analytics + Agent Foundation (2 weeks)

**Goal:** Analytics data model live. Agent advancement clock starts.

| Task | Ref |
|---|---|
| Create `ooh_analytics` schema tables | P1-AN01 |
| Wire `ChannelPerformancePage` | P1-AN02 |
| Uncomment `useUnifiedAnalytics` data calls | P1-AN03 |
| Verify/create supplier analytics RPCs | P1-AN05 |
| Implement 5 CMS + 3 DMS + 3 Ever-boarding agent TypeScript files | P1-AG04 |
| Wire prompt template performance tracking into LearningEngine | P1-AG03 |

**MALTA GO-LIVE GATE:** After Sprint 6, the platform is production-ready for single-operator deployment.

---

## 6. Go-Live Checklist — Faces Displays Malta

Before switching Faces Displays to production traffic:

**Platform:**
- [ ] All P0 bugs resolved and deployed
- [ ] All P1 Sprints 1-4 complete
- [ ] `FEATURE_LIFECYCLE_ENGINE_SITE_REQUEST` enabled for tenant
- [ ] Supabase TypeScript types regenerated and committed
- [ ] `supabase db push` run for all pending migrations
- [ ] All Edge Functions deployed (`supabase functions deploy`)

**Data:**
- [ ] Audience seed data applied (`audience_impressions_malta.sql`, `audience_jurisdiction_malta.sql`, `audience_locality_hierarchy_malta.sql`)
- [ ] Dev seed data replaced with production data
- [ ] `platform_revenue_config` table seeded with commission defaults
- [ ] Malta programme type activated (`FEATURE_LIFECYCLE_ENGINE_SITE_REQUEST` = true)

**Auth:**
- [ ] MFA enabled for all `platform_admin` accounts
- [ ] Old role name references (`sales_admin`, `portal_client`) audited and purged
- [ ] API key rotation UI accessible to admin

**Compliance:**
- [ ] DSR workflow end-to-end tested (submit → verify identity → admin review → process)
- [ ] Data retention policies seeded in DB
- [ ] `check-deletion-eligibility` tested with Malta retention rules (from `tenant_business_rules`, not hardcoded)

**Sales:**
- [ ] Rate cards seeded in `ooh_sales.sales_rate_card`
- [ ] Pricing corridors configured
- [ ] `usePricing` fix verified — pricing UI showing data
- [ ] At least one proposal created + signed + booked end-to-end

**Operations:**
- [ ] At least one frame activated via Assets module → EventBus → Sales frame created
- [ ] CMMS work order lifecycle tested end-to-end
- [ ] CMS scheduling engine tested with at least one digital screen

---

## 7. Track 2 — International Multi-Operator Expansion

**Milestone:** 2nd operator tenant onboarded. Cross-operator inventory visible.

### Pre-Phase 3 Requirements (before any Marketplace work)

| Requirement | Current State | Gap |
|---|---|---|
| ≥2 operator tenants live | 1 (Faces Malta) | Onboard 2nd operator |
| `InventoryListingAPI` in Sales module | Not started | Full build required |
| Audience confidence >0.5 | ~0.30 (seeded data) | Gravity model Phase 2 needed |
| OpenOOH taxonomy on all frames | Schema present, seed only | Validate at scale |
| Legal settlement framework | Not built | `platform_revenue_config` + operator T&Cs |
| Marketplace buyer schema | Not built | Full build required |

### Phase 3a — Marketplace MVP (8–12 weeks post Malta go-live)

1. Fix `ProgrammaticService.ts` column names (P1-S06)
2. Add missing `sales_frame` Mode 2 columns: `marketplace_corridor_id`, `marketplace_allocation_pct`, `marketplace_floor_price`
3. Add `platform_revenue_config` table
4. Extend `marketplace_transaction` with payment lifecycle fields
5. Implement `InventoryListingAPI` interface: `searchFrames`, `getFrame`, `checkAvailability`, `getOpenRTBDescriptor`, `submitRFP`, `submitBid`
6. Onboard second operator tenant
7. Scaffold Marketplace app (`/apps/marketplace`)
8. Buyer accounts + portal: registration, KYC, RFP workflow

### Phase 3b — Marketplace Commercial (12–20 weeks)

1. Stripe / payment provider integration
2. Full KYB self-qualification flow
3. Cross-operator OpenRTB bid routing
4. Private marketplace deal management
5. `platform_revenue_config` admin UI
6. Automated settlement batches + remittance

### Phase 3c — Global Scale (20+ weeks)

1. SSP HTTP adapters: Hivestack, Broadsign Reach, Vistar, Displayce (P2-002)
2. OpenDirect 2.1 HTTP surface (P2-003)
3. Audience gravity model: NSO census + Python engine (P2-001)
4. Agent autonomy advancement: first agents reach `supervised` tier (180+ days track record)
5. Mode 2 autonomous marketplace (requires agents at `full` tier — ~12+ months from cold start)
6. Multi-jurisdiction compliance engine (content restrictions + planning rules per country)

---

## 8. Technical Debt Priority Matrix

### Must resolve before international launch

| Debt | Severity | Modules Affected |
|---|---|---|
| `hub_templates` BIGSERIAL + TEXT tenant_id | High | Comms Hub + 8 dependent tables |
| 57 CMMS direct `assets` table violations | High | CMMS + Assets (blocks schema extraction) |
| Dual GDPR tables (`gdpr_requests` vs `gdpr_data_subject_requests`) | High | Compliance |
| 5 overlapping consent tables undocumented | High | Compliance |
| 4 overlapping retention policy tables undocumented | High | Compliance |
| Old `ai_*` legacy tables in public schema | Medium | AgentHub |
| Dual `LearningEngine.ts` files | Medium | AgentHub |
| Audience schema drift (8 undocumented tables) | Medium | Audience |
| DMS dual service layer migration in progress | Medium | DMS |
| `ooh_auth` schema declared but empty | Low | Auth |

---

## 9. Module Readiness Heatmap

| Module | Score | Go-Live Blocker? |
|---|---|---|
| Auth | 7/10 | No (core auth solid) |
| Sales | 5/10 | YES — usePricing P0 |
| Portal | 7/10 | YES — docs/support routes P0 |
| E-Signatures | 7/10 | No (status bug P0 is minor) |
| Compliance | 6/10 | YES — GDPR DSR split, erasure logic inverted |
| CMS | 7/10 | No (timezone P0 + EventBus gaps P1) |
| CMMS | 6.5/10 | No (Unilumin P0 minor) |
| Site Dev | 6/10 | YES — overdue detection failing, builder non-transactional |
| Partners | 4/10 | YES — portal misaligned with schema, no activation UI |
| DMS | 5.5/10 | YES — approval table name P0, classifier wrong domain |
| AgentHub | 6/10 | No (Mode 1 solid; Mode 2 future) |
| Comms Hub | 6.5/10 | No (hub_templates debt is P1) |
| Helpdesk | 6.5/10 | No (types not regenerated is P1) |
| Everboarding | 6.9/10 | YES — VIEW footgun P0 |
| Assets | 4/10 | YES — audience table P0, tenant scoping P0 |
| Audience | 6/10 | No (informational, not gating) |
| Analytics | 3/10 | No (not blocking commercial ops) |
| Marketplace | 2/10 | N/A (Phase 3) |
| Programmatic | 3/10 | YES — in-memory state P0 |
| Site Dev (partners) | 4/10 | YES — (see Partners above) |

---

## 10. Dependency Map

```
Assets EventBus emissions (P1-AS01)
  └─► Sales frame auto-creation on activation
  └─► CMS screen registration

Compliance stub (P0-006)
  └─► Sales booking confirmation gate unblocked
  └─► Frame eligibility enforcement (future rules engine)

GDPR DSR fix (P0-004)
  └─► Compliance production certification

Programmatic Gateway DB (P0-005)
  └─► ProgrammaticYieldAgent gets real data
  └─► SSP revenue tracking starts
  └─► InventoryListingAPI (Phase 3 prerequisite)

hub_templates UUID migration (P1-CH01)
  └─► All 8 FK dependencies unblocked
  └─► ooh_comms schema canonicalisation

Agent autonomy clock (starts at Malta go-live)
  └─► 180 days minimum to reach supervised tier
  └─► Mode 2 requires full tier (360+ days minimum)
  └─► Mode 2 marketplace is a 2027 feature at earliest from cold start

Malta go-live
  └─► Second tenant onboarding possible
  └─► Phase 3a Marketplace work can begin
  └─► Agent track record accumulation starts
```

---

## 11. Resource Estimates

**Track 1 (Malta Go-Live):**
- Sprint 1 (P0 Bug Blitz): 2 weeks, 1 full-stack dev
- Sprint 2-6 (P1 Hardening): 10 weeks, 1-2 devs
- Total: 12 weeks to Malta production go-live from today

**Track 2 (International):**
- Phase 3a (Marketplace MVP): 8–12 weeks post Malta go-live, 2-3 devs
- Phase 3b (Commercial): 12–20 weeks, 3-4 devs
- Phase 3c (Global scale): 20+ weeks, 4-6 devs

**Agent Autonomy:**
- Mode 1 (human-in-loop): Available from go-live day 1
- Mode 2 prerequisites: 180 days minimum track record + settlement layer + KYB
- Mode 2 realistic launch: 12+ months from Malta go-live

---

## 12. Immediate Next Actions (Week 1)

1. **P0-001:** Fix `usePricing.ts` — 1 hour — branch `fix/sales-pricing-schema`
2. **P0-002:** Fix `DocumentRepository` — 15 min — add to same branch
3. **P0-003:** Fix GDPR erasure `gte` → `lte` — 5 min
4. **P0-007/008:** Fix portal routes — 20 min
5. **P0-013:** Fix signature status — 30 min
6. **P0-010/011:** Fix Assets table name + tenant scoping — 45 min
7. **P0-006:** Create compliance stub — 2 hours
8. **P0-012:** Fix CMS timezone — 30 min
9. Request K run `supabase db push` for any schema migrations
10. Push all fixes to `main` via PR, tag `sprint-1-p0-bugfix`

---

*Master Implementation Plan v1.0 — Kai, OOH OS Autonomous Development Agent — 2026-03-26*
*Based on independent v2.0 code + schema audit of all 20 modules*
*Next review: After Sprint 1 completion*
