# Audience Module Audit v2.0
**Date:** 2026-03-25
**Auditor:** Kai (autonomous dev agent)
**Scope:** packages/audience — Phase 1 seed data, DB schema, IAudiencePort implementations, Sales integration, production readiness

---

## 1. Source Map

**Package path:** `/workspace/group/everboarding/packages/audience/`

```
packages/audience/
  README.md
  package.json
  tsconfig.json
  src/
    index.ts                            — barrel re-export
    ports/
      IAudiencePricingPort.ts           — 4-method interface contract
    adapters/
      AudiencePricingAdapter.ts         — public entry point, wraps service
    services/
      AudienceService.ts                — business logic + fallback handling
      JurisdictionConfigResolver.ts     — 5-level inheritance resolver
    repositories/
      AudienceRepository.ts             — Supabase query layer + 5-min cache
    types/
      index.ts                          — all TS types (12 interfaces/types)
    __tests__/
      AudienceService.test.ts           — service unit tests (vitest)
      JurisdictionConfigResolver.test.ts — resolver unit tests (vitest)
```

Total: 12 files, ~834 lines. Clean hexagonal structure (port → adapter → service → repository).

---

## 2. DB Schema — Live Production Verification

**Migration file:** `supabase/migrations/20260318140000_create_audience_module.sql`
**Status:** Applied. All Phase 1 tables confirmed live via REST API introspection.

### Tables confirmed in production (via Supabase REST API swagger):

| Table | In Migration | In DB | Notes |
|---|---|---|---|
| `audience_impression_data` | Yes | Yes | Phase 1 core table |
| `audience_jurisdiction_config` | Yes | Yes | 15 rows seeded |
| `audience_locality_hierarchy` | Yes | Yes | 68 Malta localities |
| `audience_event_uplifts` | Yes | Yes | Table present, no seed rows |
| `audience_frame_profiles` | No | Yes | SCHEMA DRIFT — not in Phase 1 migration |
| `audience_frame_proximity` | No | Yes | SCHEMA DRIFT — from CLAUDE.md §8.9 |
| `audience_poi` | No | Yes | SCHEMA DRIFT — from CLAUDE.md §8.9 |
| `audience_jurisdiction_datasource` | No | Yes | SCHEMA DRIFT — undocumented |
| `audience_refresh_log` | No | Yes | SCHEMA DRIFT — undocumented |
| `audience_config` | No | Yes | SCHEMA DRIFT — undocumented |
| `audience_segment` | No | Yes | SCHEMA DRIFT — undocumented |
| `audience_type` | No | Yes | SCHEMA DRIFT — undocumented |

**Finding:** 8 tables exist in the live DB that are not in the committed Phase 1 migration. The Phase 1 migration creates 4 tables; the live DB has 12 audience tables. This is significant schema drift. The additional tables (`audience_poi`, `audience_frame_proximity`, `audience_frame_profiles`) correspond to Phase 2b/2a components described in CLAUDE.md §8.9, but their migration SQL is not committed to the repo. They were likely applied manually or via an uncommitted migration.

### Schema quality (Phase 1 tables):
- RLS enabled on all 4 Phase 1 tables with tenant isolation + service_role bypass
- Correct indexes: `(frame_id, effective_date)`, `(tenant_id, effective_date)`, `(tenant_id, audience_segment)`
- `UNIQUE(tenant_id, frame_id, effective_date)` prevents duplicate seeding
- `data_source` CHECK constraint enforces `'seeded' | 'computed' | 'estimated'`
- `audience_segment` CHECK constraint with 8 valid values
- Tables remain in `public` schema (Phase 2 note in migration: "move to audience schema after Python engine built")

---

## 3. Phase 1 Seed Data Status

**Seed files present:**
- `supabase/seeds/audience_impressions_malta.sql` — 12 frames × 12 months = 144 rows
- `supabase/seeds/audience_jurisdiction_malta.sql` — 15 jurisdiction config rows
- `supabase/seeds/audience_locality_hierarchy_malta.sql` — 68 Malta localities

### Seed data design:
- All three seeds resolve tenant ID dynamically: queries `platform_tenants WHERE name ILIKE '%Faces%'`, falls back to `'00000000-0000-0000-0000-000000000001'`
- Seed uses `ON CONFLICT DO NOTHING` for idempotency
- `audience_impressions_malta.sql` loops through `ooh_sales.sales_frame` joined to `ooh_sales.sales_location` to derive frame-to-locality mapping — meaning it depends on Sales seed data being applied first (implicit ordering dependency, not enforced)
- Seasonal curves applied per audience segment (e.g. tourist_hotspot peaks in summer months, commercial_core flatter curve)
- Jurisdiction hierarchy: 1 tenant-level row, 1 country (MT), 5 regional rows (Northern Harbour, Northern, Southern Harbour, Western, South Eastern), locality-level rows for key areas, sub-district rows for Paceville/Valletta

### What is missing from seed data:
- `audience_event_uplifts` table has no seed rows. `getEventImpressionsUplift()` will always return `null` in dev/staging unless events are seeded separately.
- No seed data for the 8 undocumented tables (`audience_poi`, `audience_frame_proximity`, etc.)
- Seed commentary hardcodes Faces Displays Limited as the reference tenant — confirmed single-tenant bias in the seed file name and tenant resolution query, which is acceptable per CLAUDE.md §1.1 (seed files are explicitly operator-specific, not platform code)

### Phase 1 seed data verdict: COMPLETE for stated scope (144 impression rows, 68 localities, 15 jurisdiction configs). Event uplifts are empty.

---

## 4. Phase 2 Gravity Model Status

**Status: NOT IMPLEMENTED**

The README states: "Phase 2 (future) — Python gravity model computation engine. Live computed data replaces seeded data. TypeScript layer unchanged — only data source changes."

CLAUDE.md §8.11 describes Phase 2a as "NSO census adapter: demographic profile per locality. Gravity model: population draw from census, distance decay."

### What does and does not exist:
- No Python files anywhere in the repo (confirmed via source map)
- No `census-data.adapter.ts`, `traffic-counts.adapter.ts`, or any other adapter listed in CLAUDE.md §8.5
- No `impressions.ts`, `demographic-index.ts`, or `confidence.ts` calculator files
- No `/packages/audience/adapters/` directory — the only adapter is `AudiencePricingAdapter.ts` (the Sales-facing one)
- The `JurisdictionConfigResolver` does hold the `distanceDecayBeta` and `catchmentRadiusM` parameters, which are Phase 2 inputs — but they are stored as config only; no gravity computation uses them
- The `data_source` field supports `'computed'` as a value, and the DB schema includes a `methodology_version` column — both Phase 2 hooks are present in the schema
- `audience_frame_profiles` and `audience_frame_proximity` tables exist in the live DB (schema drift mentioned above), suggesting some Phase 2 groundwork was applied to production but no corresponding TypeScript implementation exists

### Phase 2 gravity model verdict: POST-MVP. Zero implementation. TypeScript interface is forward-compatible. Schema has some Phase 2 tables applied but no code consumes them.

---

## 5. IAudiencePort Implementations

Two port interfaces exist with different scopes:

### 5a. `IAudiencePricingPort` (packages/audience)
**Path:** `/workspace/group/everboarding/packages/audience/src/ports/IAudiencePricingPort.ts`

4 methods:
```
getFrameImpressions(frameId, startDate, endDate, tenantId, daypart?) → FrameImpressionsResult
getImpliedCPM(frameId, rateCardPrice, flightWeeks, tenantId) → ImpliedCPM
getEventImpressionsUplift(frameId, eventId, tenantId) → EventUplift | null
getJurisdictionConfig(localityName, tenantId) → ResolvedJurisdictionConfig | null
```

**Implementations:**
- `AudiencePricingAdapter` (packages/audience) — full implementation, passes to `AudienceService`
- No stub implementation exists. No `audience.stub.ts` in `/packages/sales/src/stubs/` or anywhere else

### 5b. `IAudiencePort` (src/sales)
**Path:** `/workspace/group/everboarding/src/sales/src/ports/IAudiencePort.ts`

3 methods (subset of IAudiencePricingPort — excludes `getJurisdictionConfig`):
```
getFrameImpressions(frameId, startDate, endDate, tenantId) → FrameImpressionsResult
getImpliedCPM(frameId, rateCardPrice, flightWeeks, tenantId) → ImpliedCPM
getEventImpressionsUplift(frameId, eventId, tenantId) → EventUplift | null
```

**Note:** `IAudiencePort` does not include the `daypart?` parameter that `IAudiencePricingPort` supports. This is a divergence — `SalesAudienceAdapter` wraps the full adapter but exposes a narrower interface.

**Implementations:**
- `SalesAudienceAdapter` — the bridge from Sales to the audience package

### 5c. UI-side audience consumers (not port-based):
- `/workspace/group/everboarding/src/cmms/src/components/assets/FaceAudienceMetricsPanel.tsx`
- `/workspace/group/everboarding/src/cmms/src/hooks/useFaceAudienceMetrics.ts`
- `/workspace/group/everboarding/src/communication-hub/client/src/components/AudienceSelector.tsx`
- `/workspace/group/everboarding/src/communication-hub/client/src/pages/audience/` (directory)

These UI components are separate consumers, not IAudiencePort implementations.

### Port gap: `IAudiencePort` in Sales (3 methods) is a subset of `IAudiencePricingPort` in the package (4 methods). The `getJurisdictionConfig` method is not exposed to Sales. This is likely intentional (Sales doesn't need raw jurisdiction config) but is undocumented. The `daypart` parameter is also dropped in the Sales port — Sales callers cannot request daypart-weighted CPMs.

---

## 6. Sales Integration Reality

### 6a. SalesAudienceAdapter
**Path:** `/workspace/group/everboarding/src/sales/src/adapters/SalesAudienceAdapter.ts`

Clean thin adapter. Constructor takes `SupabaseClient`, instantiates `AudiencePricingAdapter` directly. All three IAudiencePort methods delegate straight through. No transformation, no error swallowing beyond what the underlying adapter provides.

```typescript
export class SalesAudienceAdapter implements IAudiencePort {
  private audienceAdapter: AudiencePricingAdapter;
  constructor(supabase: SupabaseClient) {
    this.audienceAdapter = new AudiencePricingAdapter(supabase);
  }
  // 3 pass-through methods
}
```

### 6b. PricingService Step 4 (sales-price-proposal Edge Function)
**Integration confirmed** in `supabase/functions/sales-price-proposal/index.ts`:

Step 4 queries `audience_impression_data` directly via the Supabase client (not via SalesAudienceAdapter/IAudiencePort). This is an architectural inconsistency: the Sales package defines an IAudiencePort abstraction, but the Edge Function bypasses it and queries the DB table directly.

The edge function implements the same fallback logic as the TypeScript service (exact date range → nearest month → skip gracefully), which means the fallback behaviour is duplicated in two places.

Step 4 is correctly marked as "informational" — it attaches `audience_impressions`, `audience_cpm`, and `audience_segment` to the pricing derivation trace but does not modify the price.

### 6c. PricingService constructor injection
Per CLAUDE.md §8.12: "PricingService constructor: `PricingService(repo?, audiencePort?)` — audiencePort is optional, Step 4 gracefully degrades when no adapter is configured."

This matches the edge function behaviour: the audience lookup is wrapped in try/catch with a graceful degradation path.

### 6d. Missing wiring in PricingService (TypeScript service layer)
The SalesAudienceAdapter and IAudiencePort exist as TypeScript constructs but there is no evidence of `PricingService` in TypeScript injecting a `SalesAudienceAdapter`. The audience integration lives in the Edge Function only. The TypeScript service layer does not call the audience port.

---

## 7. Test Coverage

### packages/audience tests:
- `AudienceService.test.ts` — vitest, mocks AudienceRepository at method level. Tests: getFrameImpressions (success, no_data fallback, confidence threshold), getImpliedCPM (above_market, below_market, competitive), getEventImpressionsUplift (null case, uplift case).
- `JurisdictionConfigResolver.test.ts` — vitest, tests 5-level inheritance chain. Covers: tenant fallback, country override, region override, locality override, sub_district override, null-field inheritance (most specific non-null wins).

Estimated: 20+ passing tests (CLAUDE.md states "Unit tests: 20+ passing").

### Gaps:
- No integration tests against real DB
- No test for AudiencePricingAdapter (only AudienceService tested)
- No test for SalesAudienceAdapter
- No test for the Edge Function's Step 4 audience path

---

## 8. Identified Issues and Gaps

### Critical
1. **Schema drift**: 8 audience tables in the live DB have no corresponding migration in the repo (`audience_poi`, `audience_frame_proximity`, `audience_frame_profiles`, `audience_jurisdiction_datasource`, `audience_refresh_log`, `audience_config`, `audience_segment`, `audience_type`). These cannot be reproduced in a fresh environment. Any new developer onboarding or staging reset will be missing these tables.

### High
2. **Edge Function bypasses port abstraction**: `sales-price-proposal` queries `audience_impression_data` directly instead of through `SalesAudienceAdapter`/`IAudiencePort`. This duplicates fallback logic and breaks the abstraction that was specifically designed so "only the data source changes" in Phase 2.
3. **No audience stub**: CLAUDE.md §18.4 requires `audience.stub.ts` in the Sales stubs. It does not exist. PricingService graceful degradation works, but a proper stub would enable isolated Sales development without a seeded DB.
4. **`audience_event_uplifts` has zero seed rows**: `getEventImpressionsUplift()` always returns `null` in any non-production environment. Event uplift functionality is untestable end-to-end.

### Medium
5. **`IAudiencePort` drops `daypart` parameter**: The Sales-facing port is a subset of `IAudiencePricingPort` and does not expose daypart-weighted pricing. This limits Sales module callers from ever requesting morning/afternoon/evening CPM differentiation without changing the interface.
6. **Impression seed depends on Sales seed ordering**: `audience_impressions_malta.sql` JOINs `ooh_sales.sales_frame` to iterate frames. If Sales seed is not applied first, the audience impression seed produces zero rows silently (the `FOR` loop simply doesn't execute — no error).
7. **`getFrameLocality` parses `jurisdiction_path` string**: The repository extracts locality by splitting a text path (`'MT/Northern/Sliema'` → `'Sliema'`). This is brittle — it will break if the path format changes, and it does a second DB query to do something that could be stored as a direct FK.

### Low
8. **`distanceDecayBeta` and `catchmentRadiusM` in config but never used**: Phase 2 gravity model parameters are stored and resolved correctly but no code in Phase 1 uses them. This is expected (Phase 2 post-MVP) but should be documented as intentionally dormant.
9. **Hardcoded EUR in JurisdictionConfigResolver fallback**: Line 70 `currencyCode: 'EUR'` is a hardcoded default. For a tenant not using EUR and with no currency configured at any scope level, the fallback would be wrong. The config system supports `currencyCode` at every scope level, so this is only a risk if tenant-level config is missing entirely.

---

## 9. Production Readiness Score: 6/10

| Dimension | Score | Notes |
|---|---|---|
| Architecture | 8/10 | Clean hexagonal design, well-typed, good port abstraction |
| DB schema | 5/10 | Phase 1 tables correct; 8 undocumented tables in production = schema drift |
| Data layer | 6/10 | 144 impression rows functional; event uplifts empty; dependency on Sales seed ordering |
| Code quality | 8/10 | Service layer clean, repository pattern solid, 5-min cache correct |
| Test coverage | 6/10 | 20+ unit tests passing, but no integration tests, adapter untested |
| Sales integration | 5/10 | Adapter wired correctly; Edge Function bypasses abstraction; no stub |
| Phase 2 readiness | 4/10 | Interface forward-compatible; schema hooks present; no Python engine; partial schema in DB |
| Multi-tenancy | 7/10 | RLS correct; seed tenant resolution correct; EUR hardcoded fallback is minor risk |

**Overall: 6/10**

The Audience module is functional as a Phase 1 informational input to Sales pricing. It correctly serves seeded Malta impression data, the jurisdiction resolution chain works, and the Sales adapter wires cleanly. The module is not production-critical in Phase 1 (it enriches but does not gate pricing), which reduces the severity of its gaps.

The main blockers to higher confidence are: undocumented schema drift in the live DB, the Edge Function bypassing the port abstraction (defeating Phase 2 upgrade path), and the absence of a Sales stub.

---

## 10. Recommended Actions (Priority Order)

1. **Write and commit migration for the 8 undocumented tables** — run `supabase db diff --linked` from a machine with Docker to generate the missing migration SQL. Commit as `20260325000000_audience_phase2_schema.sql`.
2. **Refactor sales-price-proposal Step 4** to call `SalesAudienceAdapter.getFrameImpressions()` rather than querying `audience_impression_data` directly. One code path, one upgrade point for Phase 2.
3. **Seed `audience_event_uplifts`** with at least 2-3 sample uplift rows tied to dev fixture events so the code path is testable.
4. **Create `audience.stub.ts`** at `src/sales/src/stubs/audience.stub.ts` returning mock `FrameImpressionsResult` with realistic Malta values.
5. **Add `daypart` parameter to `IAudiencePort`** to match `IAudiencePricingPort` — low effort, unblocks future daypart pricing in Sales.
6. **Document seed ordering dependency** in `audience_impressions_malta.sql` header and in the main seed README if one exists.
