# Partners Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (AI Development Agent)
**Prior audit:** partners-audit-v1.0.md (2026-03-23, auditor: Cascade)
**Source audited:** `/workspace/group/everboarding/src/partners/`, `supabase/migrations/`, `supabase/functions/`, `src/portal/councils/`
**DB verified via:** Supabase REST API (live schema interrogation)

---

## 1. Audit Scope and Method

This audit directly reads source files, migration SQL, edge function code, portal pages, and live DB tables to verify claims made in v1.0. Every finding below is grounded in actual file content, not documentation.

---

## 2. Schema: Migration File vs Live DB

### Migration filename discrepancy

v1.0 cited `20260316150300_agreement_architecture_schema.sql`. The actual file in the repo is:

```
supabase/migrations/20260316150802_agreement_architecture_schema.sql
```

The timestamp suffix differs (150300 vs 150802). The v1.0 audit referenced a filename that does not exist. The correct migration is `20260316150802_agreement_architecture_schema.sql`.

### Live DB table confirmation (REST API)

All five core agreement tables confirmed present and queryable in production:

| Table | Live in DB | REST path confirmed |
|---|---|---|
| `partner_programmes` | Yes | `/partner_programmes` |
| `partner_agreements` | Yes | `/partner_agreements` |
| `agreement_coverage_members` | Yes | `/agreement_coverage_members` |
| `agreement_location_instances` | Yes | `/agreement_location_instances` |
| `agreement_renewals` | Yes | `/agreement_renewals` |

Also confirmed live: `partners`, `v_partners_from_clients`, `partner_contracts`, `lease_agreements`, `lease_agreement_assets`.

### Schema column discrepancy: `localities` vs `coverage_locality`

The `FrameworkAgreementsPage.tsx` portal page selects and inserts a `localities` column (array type) on `partner_agreements`:

```typescript
.select('id, agreement_ref, partner_id, status, localities, effective_from, effective_until, ...')
.insert({ ..., localities: localityArray, ... })
```

The actual migration defines `coverage_locality TEXT` (scalar, not array) and `coverage_scope TEXT` to indicate the type. There is no `localities` array column in the migration. The portal page is querying a column that either (a) does not exist or (b) was added by an earlier pre-agreement-architecture migration without appearing in the audited file.

**This is an unresolved schema discrepancy.** The portal `FrameworkAgreementsPage` was built before the agreement architecture migration and operates against an older `partner_agreements` schema shape. It is not wired to `coverage_scope`/`coverage_locality`/`coverage_region` at all.

### `agreement_type` column

The migration defines `agreement_type TEXT CHECK (agreement_type IN ('framework', 'site_installation'))`. The portal page correctly uses `agreement_type: 'framework'`. This column is consistent between migration and portal.

### `partner_id` type discrepancy (confirmed from v1.0)

Migration line 63: `partner_id TEXT NOT NULL`. The `agreement_coverage_members` table uses `partner_id UUID`. These are inconsistent — `partner_agreements.partner_id` is TEXT, not UUID. The v1.0 audit flagged this as a pre-existing schema inconsistency on `site_requests.partner_id`. The same issue exists on `partner_agreements`.

---

## 3. Services: Code Audit vs v1.0 Claims

All four services exist and are fully implemented (not stubs):

### CoverageResolutionService

**Status: Fully implemented with one deferred dependency.**

- 4-tier cascade implemented correctly: `checkExplicitListCoverage` → `checkLocalityCoverage` → `checkRegionCoverage` → `checkAllPartnersCoverage`
- `activateAgreement()` method exists on `CoverageResolutionService` (not only on the edge function). This creates a dual activation path: the edge function and the service. They have the same conflict check logic. The service path has no JWT guard or rate limiting.
- `createOnboardingCandidates()` is a private stub. It contains a comment acknowledging the `onboarding_candidates` table does not yet exist (deferred to "Sprint 4, S4-T1"). For `explicit_list` scope it counts partner IDs but writes nothing to DB. For all other scopes it returns 0 immediately. **This is a silent no-op, not a real implementation.**
- The method `resolveForSiteRequest` uses `programmeTypeId` as a required parameter. Callers (LocationInstanceService) must supply this. If the programme type is unknown at trigger time, resolution will return 'none'.
- Column mismatch risk: `checkExplicitListCoverage` joins `agreement_coverage_members` using `.or('location_id.eq.${locationId},partner_id.eq.${partnerId}')`. The `agreement_coverage_members.partner_id` is UUID, but `partner_agreements.partner_id` is TEXT. Queries joining these via the inner join on `partner_agreements` may behave unexpectedly.

### ContentRestrictionsService

**Status: Fully implemented. No defects found.**

- `getForLocation()`, `filterFramesByCategory()`, `validateCreativeForLocation()` all implemented.
- The `has_restrictions` logic has a minor inconsistency: it checks `Object.keys(instance.content_restrictions || {}).length > 0 && JSON.stringify(instance.content_restrictions) !== '{}'`. Both conditions check the same thing. Harmless but redundant.
- `filterFramesByCategory()` queries `asset_locations` to validate frame IDs but does not use the result — it loops `frameIds` directly. The validation step is therefore cosmetic (returns early only if `frames` is empty, but proceeds with all frameIds regardless). This is a logic gap: if a frameId does not exist in `asset_locations`, it will be treated as unrestricted rather than excluded.
- Proximity rules are checked at `filterFramesByCategory` level by matching the IAB category against `prohibited_categories` on each scheduling restriction. However, proximity rules also have a `radius_metres` field that is never evaluated in `filterFramesByCategory` — the spatial check is absent. This is a content compliance gap.

### LocationInstanceService

**Status: Fully implemented.**

- Idempotency guard works: if `coverage.instanceId` already exists, it links and returns without inserting a duplicate.
- Term end date calculation: `termEnd.setFullYear(termEnd.getFullYear() + (agreement.term_years || 3))`. Defaults to 3 years if not set.
- `contentRestrictions` parameter is optional. If not provided at trigger time, the instance is created with empty `{}` restrictions. The caller (lifecycle effects in `siteRequest.ts`) must supply this if content restrictions are needed at snapshot time. If the caller omits it, the snapshot will be empty regardless of what the agreement has.
- `linkToSiteRequest` uses a raw `.update()` with no error check — if it fails silently, the site_request will lack a `governing_instance_id` even though the instance was created successfully.

### RenewalService

**Status: Fully implemented with one structural concern.**

- `processUpcomingRenewals()` and `processExpiredRenewalNotices()` are both implemented and follow the agreed design.
- `applyEscalationRule()` correctly handles `fixed_increment` and `index_linked` types.
- The `automatic` renewal path in `initiateRenewal()` sets `trigger_event: 'commissioning_date'` on all renewed instances regardless of the original trigger. This may be incorrect for instances originally created on `permit_date`.
- The `RenewalService` is designed to be called by a "daily scheduled Edge Function" — but no such scheduled edge function exists in `supabase/functions/`. There is no `partners-process-renewals` or equivalent. The service exists but has no scheduler to invoke it. Renewals will not be processed automatically unless manually triggered.

---

## 4. Edge Function: `partners-activate-agreement`

**Status: Complete. Not a stub.**

The function at `supabase/functions/partners-activate-agreement/index.ts` (155 lines) is a complete, production-quality implementation:

- POST only, 405 on other methods
- JWT auth required (extracts user from Authorization header)
- Uses service role for the activation transaction (bypasses RLS correctly)
- Status guard: rejects activation if already `active` or not in `draft`/`pending_signatures`
- Conflict check via `scopesOverlap()` (same logic as `CoverageResolutionService.checkConflicts`) — duplicate logic but consistent
- Atomically sets `status='active'`, `activated_at`, `activated_by`
- Returns `coverageCount` for `explicit_list` scope

**Gaps vs v1.0 claims:**
- v1.0 claimed "Atomic conflict check + activation" via a Postgres transaction. The implementation is NOT atomic at the DB level — it is two sequential API calls (conflict check read, then update). A race condition exists: two simultaneous activation calls for conflicting agreements could both pass the conflict check before either writes the status. For production volumes this is low risk but the atomicity claim is overstated.
- No rate limiting (confirmed from v1.0).
- The `scopesOverlap` implementation in the edge function is slightly stricter than in `CoverageResolutionService`: the service has a comment noting "explicit_list conflicts with region/locality (conservative)" and returns `true`. The edge function does the same. They are consistent but both take a conservative stance that may block legitimate combinations (e.g. an explicit_list agreement covering specific bus stops in a locality where a separate locality-scope agreement also exists).

---

## 5. Coverage Resolution: Verified Working

4-tier cascade is correctly implemented. Specific findings:

| Tier | Status | Notes |
|---|---|---|
| `explicit_list` | Working | Matches on `location_id` OR `partner_id` from `agreement_coverage_members` |
| `locality` | Working | Looks up `asset_locations.locality`, matches `partner_agreements.coverage_locality` |
| `region` | Working | Looks up `asset_locations.region`, matches `partner_agreements.coverage_region` |
| `all_partners` | Working | Simple scope match, returns first active all_partners agreement |

**Gap:** The `locality` and `region` lookups query `asset_locations`, not `sales_location`. The agreement architecture was designed for the Partners/Site-Dev domain which uses `asset_locations`. If the Sales module's `sales_location` table is the canonical location store, these queries may be targeting the wrong table. This needs verification against the actual table that stores `locality` and `region` columns in the live DB.

---

## 6. Portal Partner Pages: Verified State

Portal pages in `src/portal/councils/` provide coverage of the partner journey:

| Page | File | Status |
|---|---|---|
| Framework Agreements list/create/detail | `FrameworkAgreementsPage.tsx` | Built, but uses old schema shape (`localities` array, not `coverage_scope`) |
| Council Dashboard | `DashboardPage.tsx` | Exists |
| Partner Onboarding Status | `PartnerOnboardingStatusPage.tsx` | Exists |
| Application List | `ApplicationListPage.tsx` | Exists |
| Application Detail | `ApplicationDetailPage.tsx` | Exists |
| Financials | `FinancialsPage.tsx` | Exists |

**Critical finding on FrameworkAgreementsPage:**
This is the primary UI for managing `partner_agreements`, but it predates the agreement architecture sprint. It:
- Queries/inserts `localities` (array) — not a migration-defined column on the new schema
- Does not set `coverage_scope`, `coverage_region`, or `coverage_locality`
- Does not set `programme_type_id`
- Inserts with `status: 'pending_signatures'` — the `partners-activate-agreement` edge function then handles activation, but the UI has no "Activate" button calling that function
- Terminates agreements directly via `.update({ status: 'terminated' })` — bypasses any conflict check logic
- Creates agreements without `programme_config` JSONB (required by schema)

The portal UI for framework agreements is **not aligned with the agreement architecture schema** introduced in sprint `20260316150802`. The two were built independently and not reconciled.

---

## 7. Discrepancies vs v1.0 Audit

| Claim in v1.0 | v2.0 Finding |
|---|---|
| Migration ref `20260316150300_agreement_architecture_schema.sql` | Actual file is `20260316150802_agreement_architecture_schema.sql` — filename in v1.0 was wrong |
| "`partners-activate-agreement` — Atomic conflict check + activation" | Not atomic at DB level. Two sequential API calls. Race condition exists under concurrent requests. |
| "15 RLS policies (3 per table)" | Migration shows 3 per table (SELECT/INSERT/UPDATE or SELECT/INSERT/DELETE depending on table). Confirmed 3 per table = 15 total for 5 tables. Consistent. |
| "5 retention classes seeded via migration" | Migration file contains a comment about `retention_classes` but no INSERT statements for it. The seeded data claim cannot be confirmed from the migration file alone. |
| "CoverageResolutionService — 4-tier cascade: explicit_list → locality → region → all_partners" | Confirmed correct. |
| "createOnboardingCandidates" creates candidate records | It is a no-op for all non-explicit_list scopes and a count-only for explicit_list. No records are written to any table. |
| "Partner management UI — Missing" | Partially false. `FrameworkAgreementsPage.tsx` exists and provides create/list/terminate for `agreement_type='framework'`. However it is misaligned with the new schema. |
| "`RenewalService` — daily scheduled Edge Function" | No scheduled edge function exists. The service has no automated invoker. |
| "`partners.framework_agreement_id` is plain UUID (no FK constraint)" | Confirmed as known issue in migration (backward compat comment present). |
| "`site_requests.partner_id` is TEXT not UUID" | Confirmed — `partner_agreements.partner_id` is also TEXT, compounding the inconsistency. |

---

## 8. Agreement Activation Status

The activation path is **functional but fragmented**:

1. `partners-activate-agreement` edge function — complete, JWT-protected, has conflict check. Called from… nothing in the portal currently. The `FrameworkAgreementsPage` creates agreements with `status: 'pending_signatures'` but has no button to invoke this function.

2. `CoverageResolutionService.activateAgreement()` — implements the same logic server-side but is not guarded by JWT and has no rate limiting. Not called from any portal UI.

3. Lifecycle effects (`siteRequest.ts`) call `LocationInstanceService.createInstanceForTriggerEvent()` on `PLANNING_APPROVED` and `INSTALLED` transitions — this does invoke `CoverageResolutionService.resolveForSiteRequest()` but this is the resolution path, not the activation path. Agreements must be active before resolution works.

**Net result:** Agreements created via the portal remain in `pending_signatures` forever unless a developer manually calls the edge function or uses the service layer directly. There is no UI path to activate an agreement.

---

## 9. Coverage Resolution Status

The resolution service is **ready but untriggered in practice** because:

1. No active `partner_agreements` records exist in the live DB (confirmed: REST API returns empty array `[]`).
2. The portal UI cannot create schema-compliant agreements (missing `coverage_scope`, `programme_type_id`, `programme_config`).
3. Even if agreements existed, the resolution service requires `programmeTypeId` as a parameter — callers must know the programme type UUID at trigger time.
4. The `asset_locations` table dependency requires verification that `locality` and `region` columns are populated.

---

## 10. Missing Items (v2.0 confirmed)

In addition to items listed in v1.0, the following were confirmed or newly identified:

| Item | Severity | Detail |
|---|---|---|
| `FrameworkAgreementsPage` schema mismatch | High | Portal inserts against old schema shape. Will fail on `programme_config NOT NULL` constraint or silently use wrong columns. |
| No UI path to activate agreements | High | Edge function exists but is never called from portal. Activation is unreachable from the UI. |
| `RenewalService` has no scheduler | High | Service works but has nothing to invoke it. Renewals will not process. |
| `createOnboardingCandidates` is a no-op | Medium | Acknowledged in code comments. Deferred to Sprint 4. |
| `linkToSiteRequest` error not handled | Medium | Silent failure possible. Site request could lose its `governing_instance_id` link. |
| `filterFramesByCategory` proximity radius not evaluated | Medium | Proximity rules are pattern-matched on category but spatial distance is ignored. |
| `filterFramesByCategory` validates frames then ignores result | Low | Cosmetic logic gap. Non-existent frames are treated as unrestricted. |
| Duplicate activation logic (edge fn + service) | Low | Two activation paths with identical logic. Creates maintenance surface. |
| `automatic` renewal reuses `commissioning_date` trigger type | Low | May not reflect original trigger event accurately. |
| No `onboarding_candidates` table | Low | Deferred. Code comments acknowledge this. |

---

## 11. Verified Working

Items confirmed working from source and schema:

- All 5 agreement tables deployed and live in production DB
- RLS enforced via `get_user_platform_tenant_id()` on all 5 tables
- `partners-activate-agreement` edge function deployed and complete
- `CoverageResolutionService` 4-tier cascade logic correct
- `ContentRestrictionsService` category/exclusivity/scheduling checks working
- `LocationInstanceService` idempotent instance creation working
- `RenewalService` renewal logic (automatic + mutually_agreed + lapse) working
- `partner_agreements.governing_instance_id` FK on `site_requests` added by migration
- `agreement_location_instances.resolved_programme_config` immutability documented in migration comment
- Ever-boarding integration: `partnerEverboardingIntegration.ts` wired to `everboarding-data-change-triggers` edge function
- Council portal pages exist with onboarding status, application list, financials

---

## 12. Production Readiness Score

**4 / 10**

### Breakdown

| Dimension | Score | Rationale |
|---|---|---|
| Schema completeness | 8/10 | All 5 tables live. Minor column inconsistencies (partner_id TEXT, localities mismatch). |
| Service layer quality | 7/10 | All 4 services implemented, non-trivial logic correct. Minor gaps (error handling, proximity radius). |
| Edge function | 6/10 | Activation function complete but not truly atomic. Race condition at scale. |
| Portal UI | 2/10 | FrameworkAgreementsPage is misaligned with new schema. No activation UI. |
| End-to-end flow | 2/10 | Cannot create a schema-compliant agreement via the portal. Activation is unreachable from UI. |
| Automated processes | 3/10 | RenewalService exists but has no scheduler. LocationInstanceService has no automated invoker from lifecycle events verified. |
| Test coverage | 1/10 | No unit tests for any partners service (no test files found under `src/partners/`). |

### What must be resolved before calling this production-ready

1. Reconcile `FrameworkAgreementsPage.tsx` with the new agreement architecture schema (coverage_scope, programme_type_id, programme_config).
2. Add an "Activate" button in the portal that calls `partners-activate-agreement`.
3. Create a scheduled edge function (or cron) that calls `RenewalService.processUpcomingRenewals()` and `processExpiredRenewalNotices()`.
4. Add error handling for `linkToSiteRequest` in `LocationInstanceService`.
5. Verify `asset_locations` table has `locality` and `region` columns populated for all active sites.
6. Write integration tests for the coverage resolution cascade.

---

*End of Partners Module Audit v2.0*
