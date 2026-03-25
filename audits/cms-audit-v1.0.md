# CMS Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Sprint 5 Complete — Programmatic Gateway Live  
**Source paths audited:** `src/cms/`, `CLAUDE.md §§1, 6.14, 16B`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | CMS (Content Management System) |
| DB schema | `ooh_cms` |
| Frontend path | None — headless backend service module |
| Backend path | `src/cms/` |
| Owner | Vertical |
| Phase | Sprint 5 of 5 complete — production-ready |

---

## 2. Current State Summary

The CMS module is the digital delivery engine for OOH OS. It sits between confirmed bookings and physical screens, managing creative scheduling, proof of play, programmatic slot management, and content rebalancing.

The module has completed 7 development sprints (Sprint 0 Foundation → Sprint 5 Programmatic Gateway) as of late 2025 / early 2026. It is the most technically complete vertical module in the codebase with 155 unit tests.

**What works today:**
- Full scheduling engine (`SchedulingEngine.ts`, 17.7KB) with 6 delivery handlers
- Rebalancing engine (`RebalancingEngine.ts`, 15.8KB) for SOV correction
- Programmatic gateway (`ProgrammaticGateway.ts`, 19.8KB) — OpenRTB 2.6 DOOH bid handling
- IPlayerPort hardware abstraction layer (Broadsign, Xibo, Outsmart adapters)
- Creative Library service (`CreativeLibraryService.ts`) with transcoding job queue
- Proof of Play ingestion + aggregation (delivery summaries)
- Dynamic Screen Selection engine
- Static operations (posting confirmation for classic)
- 4 Edge Functions deployed

**Sprints completed:**
- Sprint 0: Foundation (schema, RLS, basic types)
- Sprint 1: Creative Library (upload, transcoding, approval)
- Sprint 2: SchedulingEngine (6 handlers + IPlayerPort)
- Sprint 2b: RebalancingEngine (saturation correction, priority queue)
- Sprint 3: Proof of Play (play log ingestion, delivery summaries)
- Sprint 4: Phase 2 Adapters (Xibo live, Outsmart adapter)
- Sprint 5: Programmatic Gateway (OpenRTB 2.6 DOOH, SSP reconciliation)

---

## 3. Client Journey

### Operator Flow
1. Booking confirmed in Sales → `sales.booking.confirmed` event → CMS receives screen_ids + creative_id
2. `SchedulingEngine` selects delivery handler based on `booking.delivery_commitment_type`
3. Schedule pushed to device via IPlayerPort adapter (Broadsign/Xibo/Outsmart)
4. Devices play content → play logs ingested → `cms_play_log` populated
5. `cms_delivery_summary` aggregated daily per booking line per device
6. Proof of play report generated → `cms.campaign.completed` event → Sales surfaces to Portal
7. If delivery falls short → `RebalancingEngine` reallocates SOV across screen pool
8. Static classic campaigns → `cms_posting_confirmation` photo + GPS proof captured

### Programmatic Flow
1. SSP sends OpenRTB bid request → `ProgrammaticGateway` validates against floor price
2. Win recorded → `cms_programmatic_win` → creative scheduled in next available slot
3. `cms_ssp_impression_reconciliation` — nightly reconciliation vs SSP-reported impressions
4. Discrepancy > threshold → alert via EventBus

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| SchedulingEngine (6 handlers) | ✅ Built | `src/cms/scheduling/SchedulingEngine.ts` (17.7KB) |
| RebalancingEngine | ✅ Built | `src/cms/rebalancing/RebalancingEngine.ts` (15.8KB) |
| ProgrammaticGateway (OpenRTB 2.6) | ✅ Built | `src/cms/programmatic/ProgrammaticGateway.ts` (19.8KB) |
| IPlayerPort (hardware abstraction) | ✅ Built | `src/cms/ports/` — Broadsign, Xibo, Outsmart |
| Creative Library Service | ✅ Built | `src/cms/services/CreativeLibraryService.ts` (7.1KB) |
| Proof of Play ingestion | ✅ Built | `src/cms/proof-of-play/` |
| Static Ops (posting confirmation) | ✅ Built | `src/cms/static-ops/` |
| Edge Function: cms-upload-creative | ✅ Deployed | `supabase/functions/cms-upload-creative/` |
| Edge Function: cms-manage-creative | ✅ Deployed | `supabase/functions/cms-manage-creative/` |
| Edge Function: cms-process-transcoding | ✅ Deployed | `supabase/functions/cms-process-transcoding/` |
| Edge Function: cms-push-schedules | ✅ Deployed | `supabase/functions/cms-push-schedules/` |
| Dynamic Screen Selection engine | ✅ Built | Pool assignment in `cms_pool_daily_assignment` |
| SSP impression reconciliation | ✅ Built | `cms_ssp_impression_reconciliation` table |
| Operator UI for schedule management | ❌ Missing | No ops-facing UI — operator manages via Sales module |
| Creative review UI (operator) | ❌ Missing | Approval workflow is API-only currently |
| CMS-owned Frontend module | ❌ Not planned | Intentional — CMS is headless; UI via Sales + Portal |

---

## 5. Schema Detail

**Schema:** `ooh_cms`  
**Table count:** 13  
**Multi-tenancy:** All tables include `tenant_id uuid NOT NULL` + RLS enforced (4 policies per table = 48 total + 7 on Sprint 1 additions)  
**Migrations:** `20260318150000_cms_sprint0_schema.sql` + `20260318160000_cms_sprint1_creative_library.sql`

| Table | Purpose | Key columns |
|---|---|---|
| `cms_format_spec` | Accepted creative specs per screen type | `width`, `height`, `duration_seconds`, `file_types[]`, `max_file_size_bytes`, `codec` |
| `cms_creative` | Creative assets uploaded by clients/ops | `status` (pending_review/approved/rejected/paused/archived), `file_path`, `sha256_hash`, `format_spec_id` |
| `cms_play_schedule` | Schedule per device per date range | `device_id`, `valid_from`, `valid_to`, `timezone`, `loop_duration_seconds`, `status` (draft/pending_push/pushed/failed) |
| `cms_schedule_slot` | Individual slot within a schedule | `schedule_id`, `creative_id`, `booking_line_id`, `type` (discriminated union), `priority`, `is_soft_hold` |
| `cms_play_log` | Raw play log entries from device adapters | `device_id`, `creative_id`, `played_at`, `duration_ms`, `source_adapter` |
| `cms_delivery_summary` | Aggregated delivery per booking line per device per day | `booking_line_id`, `device_id`, `summary_date`, `plays_delivered`, `impressions_delivered`, `delivery_pct` |
| `cms_posting_confirmation` | Static posting proof (photo, GPS, timestamp) | `booking_line_id`, `frame_id`, `photo_path`, `gps_lat`, `gps_lng`, `confirmed_by` |
| `cms_pool_daily_assignment` | Daily screen assignments for pool/dynamic campaigns | `pool_id`, `device_id`, `assignment_date`, `booking_line_id` |
| `cms_rebalancing_queue` | Priority queue for saturation correction | `booking_line_id`, `priority`, `reason`, `queued_at`, `status` |
| `cms_rebalancing_audit_log` | Immutable record of rebalancing decisions | `booking_line_id`, `device_id`, `old_sov_pct`, `new_sov_pct`, `reason`, `decided_at` |
| `cms_programmatic_win` | SSP win records for programmatic slots | `device_id`, `ssp_name`, `win_id`, `cpm_bid`, `creative_url`, `played_at` |
| `cms_ssp_impression_reconciliation` | Reconciliation between OOH OS and SSP impressions | `device_id`, `ssp_name`, `reconciliation_date`, `ooh_os_impressions`, `ssp_impressions`, `discrepancy_pct` |
| `cms_transcoding_job` | Creative transcoding job queue | `creative_id`, `status` (pending/processing/completed/failed), `output_path`, `attempts`, `max_attempts` |

**Storage:** `cms-creatives` bucket (private, 200MB limit, tenant-isolated by folder prefix, 3 storage RLS policies)

### Sales schema additions (owned by CMS Sprint 0 migration)
| Table | Purpose |
|---|---|
| `sales_screen_pool` | Named pool for Dynamic Screen Selection |
| `sales_screen_pool_member` | Pool membership |

**Columns added to `sales_booking`:** `rebalancing_enabled`, `rebalance_threshold`, `saturation_freeze_from`, `delivery_goal_type`, `delivery_goal_value`, `delivery_goal_progress`, `delivery_goal_currency`

---

## 6. Delivery Handler Architecture

The `SchedulingEngine` uses a discriminated union pattern with 6 delivery handlers:

| Handler | Delivery Type | Description |
|---|---|---|
| `FixedSlotHandler` | `fixed_slot` | Specific daypart + SOV% on named screens |
| `SharedGoalHandler` | `shared_goal` | Multiple bookings contribute to a delivery goal |
| `PoolDynamicHandler` | `pool_dynamic` | Dynamic Screen Selection from named pool |
| `ProgrammaticGuaranteedHandler` | `programmatic_guaranteed` | OpenDirect-style committed programmatic |
| `OpenMarketplaceHandler` | `open_marketplace` | OpenRTB open auction slots |
| `StaticPostingHandler` | `static_posting` | Classic/static billboard posting confirmation |

IPlayerPort adapters: `BroadsignAdapter`, `XiboAdapter` (live Sprint 4), `OutsmartAdapter` (Sprint 4)

---

## 7. EventBus Surface

### Outbound Events (CMS → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `cms.campaign.scheduled` | Schedule pushed to device | Sales (stores `cms_campaign_id`) |
| `cms.campaign.completed` | Campaign end date + final delivery | Sales (marks booking Completed, triggers PoP report) |
| `cms.playback.failed` | Content failed to play | Sales (triggers make-good) |
| `cms.booking.rebalanced` | SOV reallocated across screen pool | Sales (records new assignments) |
| `cms.creative.transcoding_complete` | Transcoding job finished | Sales (updates creative status) |
| `cms.creative.transcoding_failed` | Transcoding failed | Sales (notifies creative owner) |
| `cms.ssp.impression_discrepancy` | Reconciliation discrepancy > threshold | Analytics, Sales |

### Inbound Events (Platform → CMS)

| Event | Source | CMS Action |
|---|---|---|
| `sales.booking.confirmed` | Sales | Receive screen_ids + creative_id → begin scheduling |
| `sales.booking.cancelled` | Sales | Remove campaign from schedule |
| `sales.booking.amended` | Sales | Update schedule dates |
| `sales.creative.approved` | Sales | Trigger schedule slot assignment |

---

## 8. External Integrations

| Integration | Type | Status | Notes |
|---|---|---|---|
| Broadsign | IPlayerPort adapter | ✅ Built | `BroadsignAdapter` in `src/cms/adapters/` |
| Xibo | IPlayerPort adapter | ✅ Live | `XiboAdapter` — Sprint 4 live integration |
| Outsmart | IPlayerPort adapter | ✅ Built | `OutsmartAdapter` — Sprint 4 |
| Broadsign Reach (SSP) | ProgrammaticGateway | ✅ Built | OpenRTB 2.6 bid handling |
| Supabase Storage | Creative file storage | ✅ Live | `cms-creatives` bucket |
| Supabase Edge Functions | Transcoding + push | ✅ Deployed | 4 functions live |

---

## 9. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| No operator UI for schedule management | Medium | CMS is fully headless — operators cannot inspect schedules without DB access |
| Creative approval workflow is API-only | Medium | No UI surface for ops to review/approve creatives directly in CMS context |
| `cms_rebalancing_audit_log` is immutable but has no archival policy | Low | Will grow unboundedly — needs a retention/archival strategy |
| Transcoding `max_attempts` is hardcoded | Low | Should be configurable per tenant business rules |
| `cms_programmatic_win` has no TTL / cleanup | Low | Historical win data will accumulate — archival policy needed |
| SSP reconciliation threshold is not tenant-configurable | Low | Discrepancy alert threshold should live in `tenant_business_rules` |
| No circuit breaker on IPlayerPort adapter calls | Low | If Xibo/Broadsign API is slow, schedule push will block |
| `cms-push-schedules` Edge Function retry strategy not documented | Low | Should align with EventBus retry pattern (30s→2m→10m→1h→DLQ) |
| `cms_play_schedule.status = 'failed'` has no auto-retry | Low | Failed push requires manual re-trigger |

---

*End of CMS Module Audit v1.0*
