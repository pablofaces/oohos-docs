# Audience Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Phase 1 Complete (Seeded Data) — Phase 2 Architecture Defined  
**Source paths audited:** `packages/audience/src/`, `CLAUDE.md §§8, 8.1–8.7, 18.1 Audience`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Audience |
| DB schema | `ooh_audience` |
| Package path | `packages/audience/src/` |
| Owner | Horizontal |
| Phase | Phase 1 live (seeded Malta data); Phase 2 architecture complete |

---

## 2. Current State Summary

The Audience module calculates and serves audience profiles for OOH frames and screens. Its core output is the `AudienceProfile` — a structured data object containing impression counts, demographic breakdowns, and OpenRTB-compatible audience descriptors. This data feeds Sales proposal building, CPM pricing (step 4 of PricingRulesEngine), and the Marketplace InventoryListingAPI.

**Phase 1 (live):** Seeded static impression data for Malta (in `supabase/seeds/audience_impressions_malta.sql`). The Audience API is fully wired but returns pre-seeded data only.

**Phase 2 (architecture defined, not yet built):** Live gravity model combining census data, footfall sensors, mobile signal density, and OpenOOH venue taxonomy scoring.

**The module is a package** (`packages/audience/`) — it exposes a public API consumed by Sales, CMS (programmatic), and Marketplace. It never calls modules back.

---

## 3. Core Output: AudienceProfile

```typescript
interface AudienceProfile {
  frame_id: string
  weekly_impressions: number
  daily_average: number
  hourly_breakdown: Record<string, number>  // by daypart
  demographic_breakdown: {
    age_groups: Record<string, number>
    gender: { male: number, female: number }
    income_quintile: Record<string, number>
  }
  venue_type: string              // OpenOOH taxonomy string
  venue_enum_id: number           // OpenOOH enum ID
  confidence_score: number        // 0–1 data quality indicator
  data_source: string             // 'seeded' | 'census' | 'gravity_model' | 'sensor'
  last_refreshed_at: string
}
```

---

## 4. Public API Methods

All consumed via `agentHub.dispatch()` pattern or direct package import by Sales:

| Method | Used by | Description |
|---|---|---|
| `audience.getProfile({ frame_id, date_range, dayparts })` | Sales (proposal build), ProposalAgent | Single frame profile |
| `audience.getBulkProfiles({ frame_ids, date_range })` | Sales (proposal send to Portal) | Batch profiles |
| `audience.estimateReach({ frame_ids, date_range, frequency })` | ProposalAgent, Portal | Unique reach + total impressions |
| `audience.getOpenRTBAudienceDescriptor({ screen_id })` | ProgrammaticAgent, InventoryListingAPI | OpenRTB DOOH audience object |
| `audience.getBulkProfiles({ all_active_frame_ids })` | Daily cache refresh job | All-frame batch for performance |

**Phase 1 contract:** All methods return stub/seeded data until the live data pipeline is active. Sales codes against the interface, not the implementation.

---

## 5. Data Source Pyramid (Phase 2)

Layers stack — lower layers enrich higher layers:

| Layer | Source | Status |
|---|---|---|
| L1 — Census + static | National census data, council footfall counts | Phase 2 planned |
| L2 — POI density | Google Places / HERE POI database — proximity scoring | Phase 2 planned |
| L3 — Gravity model | Distance-decay population gravity model | Phase 2 planned |
| L4 — Mobile signal | Anonymised mobile device density (telco data) | Phase 3 planned |
| L5 — Sensor fusion | Footfall sensors (IR/camera), WiFi probes | Phase 3 planned |

**Phase 1 substitute:** Malta seed data in `supabase/seeds/audience_impressions_malta.sql` fills all layers with static pre-calculated values.

---

## 6. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `AudienceProfile` type definition | ✅ Built | `packages/audience/src/types/` |
| Public API interface (`IAudiencePort`) | ✅ Built | `packages/audience/src/ports/` |
| Phase 1 seeded data (Malta) | ✅ Live | `supabase/seeds/audience_impressions_malta.sql` |
| `SalesAudienceAdapter` in Sales module | ✅ Built | `src/sales/src/adapters/SalesAudienceAdapter` — bridges port |
| `audience.getProfile()` (seeded) | ✅ Live | Returns Malta static data |
| `audience.getBulkProfiles()` (seeded) | ✅ Live | Batch return |
| `audience.estimateReach()` (seeded) | ✅ Live | Static calculation |
| `audience.getOpenRTBAudienceDescriptor()` | ✅ Built | OpenRTB DOOH format |
| POI database schema | ✅ Designed | `CLAUDE.md §8.5` — schema defined |
| OpenOOH venue taxonomy | ✅ Designed | Full taxonomy in `ooh_audience` schema |
| Daypart definitions (6 standard) | ✅ Built | Morning / Midday / Afternoon / Evening / Night / Late Night |
| Intelligence ingestion adapters | 🟡 Designed | Adapter interfaces for census, POI, mobile, sensor |
| Gravity model engine | ❌ Missing | Phase 2 — not yet built |
| Live census data pipeline | ❌ Missing | Phase 2 |
| Mobile signal density integration | ❌ Missing | Phase 3 |
| Footfall sensor integration | ❌ Missing | Phase 3 |
| Cross-operator audience benchmarking | ❌ Missing | Phase 3d (Marketplace) |
| Audience segment builder (UI) | ❌ Missing | No UI — API-only currently |

---

## 7. Schema Detail

**Schema:** `ooh_audience`  
**Multi-tenancy:** All tables include `tenant_id` + RLS enforced

| Table | Purpose |
|---|---|
| `audience_frame_profiles` | Cached `AudienceProfile` per frame — refreshed daily |
| `audience_daypart_impressions` | Hourly impression breakdown per frame per daypart |
| `audience_demographic_breakdown` | Demographic segments per frame |
| `audience_poi_database` | POI records with lat/lng, category, OpenOOH venue type |
| `audience_locality_hierarchy` | Malta locality hierarchy (seeded) |
| `audience_jurisdiction` | Jurisdiction reference (Malta seeded) |
| `audience_impressions` | Raw impression records (Phase 1: Malta seed data) |

**Seed files:**
- `supabase/seeds/audience_impressions_malta.sql`
- `supabase/seeds/audience_jurisdiction_malta.sql`
- `supabase/seeds/audience_locality_hierarchy_malta.sql`

---

## 8. OpenOOH Venue Taxonomy

The Audience module implements the full OpenOOH venue taxonomy for all frames:

| Top-level category | Examples |
|---|---|
| `transit` | Bus shelters, train stations, airports |
| `retail` | Shopping centres, supermarkets |
| `outdoor` | Roadside billboards, street furniture |
| `leisure` | Cinemas, gyms, stadiums |
| `point_of_care` | Hospitals, pharmacies |
| `education` | Universities, schools |
| `office` | Office buildings, business parks |

Stored as `openooh_venue_type` (string) and `openooh_enum_id` (integer) on `sales_frame` and in `AudienceProfile`. Required for InventoryListingAPI and OpenRTB bid requests.

---

## 9. Daypart Definitions (6 standard)

| Daypart | Time window | OOH significance |
|---|---|---|
| Morning | 06:00–10:00 | Commuter peak |
| Midday | 10:00–14:00 | Lunchtime retail |
| Afternoon | 14:00–18:00 | School run + afternoon commute |
| Evening | 18:00–22:00 | Leisure + dining |
| Night | 22:00–01:00 | Entertainment venues |
| Late Night | 01:00–06:00 | Low traffic / illuminated only |

---

## 10. External Integrations

| Integration | Direction | Status | Notes |
|---|---|---|---|
| Sales (`SalesAudienceAdapter`) | Audience → Sales | ✅ Live | Adapter pattern bridges IAudiencePort |
| CMS (`ProgrammaticGateway`) | Audience → CMS | ✅ Designed | OpenRTB descriptor for bid responses |
| Marketplace (InventoryListingAPI) | Audience → Marketplace | ✅ Designed | Weekly impressions + AudienceProfile in FrameListing |
| Google Places API | Audience ← | 🟡 Designed | POI database enrichment (Phase 2b) |
| Census data APIs | Audience ← | ❌ Missing | National statistics sources (Phase 2) |
| Mobile signal telco | Audience ← | ❌ Missing | Phase 3 — anonymised device density |

---

## 11. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| Phase 1 seeded data only covers Malta | High | Any non-Malta market will receive Malta impression data — confidence_score will be misleading |
| No circuit breaker on Sales → Audience API call | Medium | PricingService calls Audience synchronously — slow Audience API blocks proposal pricing |
| `data_source = 'seeded'` not surfaced in Sales UI | Medium | Sales reps cannot see whether impressions are real or seeded — transparency issue |
| Daily cache refresh job not documented | Low | `audience.getBulkProfiles({ all_active_frame_ids })` — schedule not confirmed wired |
| Confidence score calibration not defined | Low | How is `confidence_score` calculated in Phase 1? Likely hardcoded. |
| OpenOOH venue taxonomy tags not validated on all frames | Low | Tags required for InventoryListingAPI and programmatic — gap until all frames are tagged |
| No UI for audience data management | Low | Audience module is API-only — no operator-facing UI for data quality review |

---

*End of Audience Module Audit v1.0*
