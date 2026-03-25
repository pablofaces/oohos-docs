# Programmatic Gateway Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Sprint 5 Complete — Multi-SSP + OpenDirect 2.1 Live  
**Source paths audited:** `src/cms/programmatic/`, `supabase/functions/cms-programmatic-gateway/`, `CLAUDE.md §§1 Sprint 5, 16B`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Programmatic Gateway |
| Parent module | CMS (delivered as CMS Sprint 5) |
| DB schema | `ooh_cms` (programmatic tables) |
| Backend path | `src/cms/programmatic/ProgrammaticGateway.ts` (19.8KB) |
| Edge Function | `supabase/functions/cms-programmatic-gateway/` |
| Owner | Vertical (CMS) |
| Phase | Sprint 5 complete — production-ready |

---

## 2. Current State Summary

The Programmatic Gateway is the real-time bidding and programmatic deal management layer within the CMS module. It handles OpenRTB 2.6 DOOH bid requests from multiple SSPs, OpenDirect 2.1 programmatic guaranteed deals, SSP impression reconciliation, and yield analysis.

**What is built:**
- Multi-SSP passthrough: Hivestack, Perion, Place Exchange, Broadsign Reach
- OpenRTB 2.6 DOOH bid request handling (`ProgrammaticGateway.ts`)
- OpenDirect 2.1 (OOHbjects): orders, lines, creative, delivery
- SSP impression reconciliation — discrepancy threshold, make-good trigger
- Yield analysis: floor price recommendations, high-value screen identification, fill rate reports
- `cms_programmatic_win` table — per-win records
- `cms_ssp_impression_reconciliation` table — nightly reconciliation
- Edge Function `cms-programmatic-gateway` — entry point for SSP bid requests
- `ProgrammaticAgent` in AgentHub — evaluates bids, checks availability, responds

**27 unit tests** (part of 181 total CMS tests)

---

## 3. Programmatic Flow

### OpenRTB (Open Auction)
1. SSP sends OpenRTB 2.6 DOOH bid request → `cms-programmatic-gateway` Edge Function
2. `ProgrammaticGateway` validates: screen availability, floor price, audience match, compliance category
3. `ProgrammaticAgent` (AgentHub) evaluates bid against `sales_screen.floor_price_cpm`
4. Bid response returned (win/no-bid/error) within SSP timeout window
5. On win: `cms_programmatic_win` record created → creative scheduled in next available slot
6. `sales.programmatic.win` emitted → Availability surface updated + Analytics

### OpenDirect 2.1 (Programmatic Guaranteed)
1. Buyer submits OpenDirect order via `InventoryListingAPI.submitBid()` (PG path)
2. `ProgrammaticGateway` creates deal record with committed inventory
3. OOHbjects metadata attached (venue type, location, audience)
4. Creative submitted + transcoded via `cms-upload-creative`
5. Schedule committed → `cms.campaign.scheduled` emitted to Sales

### Impression Reconciliation (Nightly)
1. `cms_ssp_impression_reconciliation` records: OOH OS play logs vs SSP-reported impressions
2. Discrepancy > configurable threshold → `cms.ssp.impression_discrepancy` event
3. If material shortfall → `make-good trigger` → Sales creates `sales_make_good` record

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| OpenRTB 2.6 DOOH bid handling | ✅ Built | `ProgrammaticGateway.ts` |
| Multi-SSP support (4 SSPs) | ✅ Built | Hivestack, Perion, Place Exchange, Broadsign |
| OpenDirect 2.1 (OOHbjects) | ✅ Built | Orders, lines, creative, delivery |
| `cms_programmatic_win` table | ✅ Built | Per-win records |
| `cms_ssp_impression_reconciliation` table | ✅ Built | Nightly reconciliation |
| Floor price check on bid | ✅ Built | vs `sales_screen.floor_price_cpm` |
| Yield analysis | ✅ Built | Floor price recommendations, fill rate reports |
| Edge Function `cms-programmatic-gateway` | ✅ Deployed | SSP entry point |
| `ProgrammaticAgent` in AgentHub | ✅ Built | Full autonomy (L4) — evaluates + responds |
| SSP named credentials per tenant | 🟡 Designed | Credentials stored per tenant — provider setup required per operator |
| OpenRTB audience descriptor | 🟡 Phase 1 | From Audience module — seeded Malta data in Phase 1 |
| DSP connectivity (open auction) | 🟡 Partial | 4 SSPs integrated; new SSP = new adapter |
| Cross-operator programmatic (Marketplace) | ❌ Missing | Phase 3c |
| Private marketplace deal UI (operator) | ❌ Missing | Deal management is API-only |
| Programmatic campaign view in Portal | ❌ Missing | Clients cannot see SSP win records |

---

## 5. Schema Detail (Programmatic-specific tables in `ooh_cms`)

| Table | Purpose | Key columns |
|---|---|---|
| `cms_programmatic_win` | SSP win records | `device_id`, `ssp_name`, `win_id`, `cpm_bid`, `creative_url`, `played_at` |
| `cms_ssp_impression_reconciliation` | Reconciliation records | `device_id`, `ssp_name`, `reconciliation_date`, `ooh_os_impressions`, `ssp_impressions`, `discrepancy_pct` |

### Sales schema fields for programmatic (on `sales_screen`)
- `openrtb_w`, `openrtb_h` — OpenRTB dimensions
- `openrtb_mime_types text[]` — accepted creative formats
- `floor_price_cpm` — minimum bid price
- `floor_price_currency` — from `sales_market.currency_code`
- `ssp_screen_ids jsonb` — per-SSP screen IDs (Hivestack ID, Perion ID, etc.)
- `programmatic_enabled boolean`

### Sales schema fields for programmatic (on `sales_frame`)
- `dpaa_venue_type`, `dpaa_venue_sub_type` — DPAA venue taxonomy
- `programmatic_eligible boolean`
- `ssp_ids text[]` — SSP identifiers

---

## 6. EventBus Surface

### Outbound Events

| Event | Trigger | Consumer(s) |
|---|---|---|
| `cms.programmatic.win_received` | SSP bid won | Availability surface update |
| `cms.programmatic.make_good_triggered` | Reconciliation discrepancy > threshold | Sales (`sales_make_good`) |
| `cms.ssp.impression_discrepancy` | Discrepancy above threshold | Analytics, Sales |
| `sales.programmatic.win` | (emitted by Sales on CMS win) | Availability surface, Analytics |

### Inbound Events

| Event | Source | Gateway action |
|---|---|---|
| `sales.booking.confirmed` (PG path) | Sales | Commit programmatic guaranteed slots |
| `sales.booking.cancelled` | Sales | Release PG slots |

---

## 7. External Integrations

| SSP | Protocol | Status |
|---|---|---|
| Hivestack | OpenRTB 2.6 DOOH | ✅ Built |
| Perion (formerly Undertone) | OpenRTB 2.6 DOOH | ✅ Built |
| Place Exchange | OpenRTB 2.6 DOOH | ✅ Built |
| Broadsign Reach | OpenRTB 2.6 DOOH | ✅ Built |

All SSPs use the same `ISSPAdapter` interface — adding a new SSP requires only a new adapter.

---

## 8. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| SSP credentials require per-operator setup | Medium | Each operator must configure named credentials in `channel_configs` or equivalent — no self-service UI |
| OpenRTB audience descriptor uses Phase 1 seeded data | Medium | Bid responses will include Malta impression data for all markets in Phase 1 |
| Private marketplace deal management has no operator UI | Medium | Deal setup is API/DB-only |
| Programmatic campaign not visible in Portal | Medium | Clients with programmatic bookings cannot track SSP wins |
| Reconciliation discrepancy threshold is not tenant-configurable | Low | Should live in `tenant_business_rules` |
| `cms_programmatic_win` has no TTL / archival | Low | Win history grows unboundedly |
| Bid response latency not monitored | Low | No alerting if gateway response time approaches SSP timeout |
| Floor price currency consistency | Low | `floor_price_currency` must match `sales_market.currency_code` — no DB constraint enforces this |

---

*End of Programmatic Gateway Audit v1.0*
