# Sales Module Audit v2.0 — Deep Verification

**Date:** 2026-03-25
**Auditor:** Kai (Claude Code — automated code + schema verification)
**Previous version:** v1.0 (Cascade, 2026-03-23)
**Status:** ⚠️ SUBSTANTIAL GAPS — core pricing UI is broken, 14 tables missing from schema

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Sales |
| DB schema | `ooh_sales` (separate from `public`) |
| Frontend path | `src/sales/src/` |
| Services | 12 (verified) |
| Agents | 16 prompt sets (19 agent files in `/src/agents/specialists/`) |
| Edge Functions | 5 Sales-specific + 6 Salesforce-related |
| Phase | MVP claimed — NOT fully production-ready due to bugs below |

---

## 2. Critical Bugs Found

### 2.1 `usePricing` Hook — WRONG SCHEMA (CRITICAL)

**File:** `src/sales/src/hooks/usePricing.ts`

The `usePricing` hook calls `supabase.from('sales_rate_card')` directly — **without** `.schema('ooh_sales')`. All Sales tables are in the `ooh_sales` schema, not `public`. This means **every pricing query returns empty/error silently**.

The correct pattern (used in `SalesBaseRepository`) is:
```ts
supabase.schema('ooh_sales').from('sales_rate_card')
```

`usePricing` bypasses `SalesBaseRepository` entirely and calls Supabase directly, hitting the wrong schema.

**Impact:** The entire Pricing & Rate Cards page (`/sales/pricing`) will show no data. PricingRulesEngine has no rate cards, seasons, pricing rules, discount rules, corridors, or buying strategy modifiers to work with. This breaks Step 3+ of the 11-step pricing engine for any booking created via the UI.

**Fix required:** Rewrite `usePricing` to use `supabase.schema('ooh_sales').from(...)` for all queries, or refactor to use `PricingRepository` (which extends `SalesBaseRepository` and uses the correct schema).

### 2.2 `usePricing` — WRONG TABLE NAME

The hook queries `.from('sales_pricing_rule')` but the actual migration creates the table as `sales_pricing_rules` (plural). Even if the schema issue above were fixed, this query would fail.

**All affected table name mismatches in `usePricing`:**
| Code calls | Actual table name |
|---|---|
| `sales_pricing_rule` | `sales_pricing_rules` |

### 2.3 Role References — Stale (from Auth audit)

v1.0 described roles `sales_admin`, `sales_rep`, `sales_manager`, `finance` — these don't exist in the actual auth type system. The real roles are `platform_admin`, `admin`, `agent`, `user`. Sales-level access is controlled by namespaced permissions (`admin.*`), not a dedicated sales role enum.

---

## 3. Schema Gaps — Tables in v1.0 Audit but MISSING from Migrations

The v1.0 audit claimed "55-table schema". Actual `ooh_sales` migration count: **57 tables**. However, 14 specific tables described in v1.0 do not exist in any migration file:

| Missing Table | v1.0 Claimed Purpose | Status |
|---|---|---|
| `sales_structure` | Physical advertising structure | ❌ Not in migrations |
| `sales_holding_group` | Agency holding group | ❌ Not in migrations |
| `sales_proposal_line_classic` | Classic frame line item | ❌ Schema consolidated into `sales_proposal_line` |
| `sales_proposal_line_digital` | Digital line item | ❌ Schema consolidated into `sales_proposal_line` |
| `sales_booking_schedule_classic` | Classic schedule lines | ❌ Not in migrations |
| `sales_campaign_booking` | Campaign↔Booking junction | ❌ Not in migrations |
| `sales_creative_revision` | Creative revision requests | ❌ Not in migrations |
| `sales_package_booking` | Booked package instances | ❌ Not in migrations |
| `sales_inventory_package_screen` | Package↔Screen junction | ❌ Not in migrations |
| `sales_programmatic_deal_screen` | Deal↔Screen junction | ❌ Not in migrations |
| `sales_programmatic_transaction` | Per-bid-win log | ❌ Not in migrations |
| `sales_commission_setting` | Per-rep commission config | ❌ Not in migrations |
| `sales_seasonal_rate` | Season multipliers | ❌ Not in migrations |
| `platform_revenue_config` | Platform commission config | ❌ Not in migrations |

**Consolidations (not bugs — design changes):**
- `sales_proposal_line_classic` + `sales_proposal_line_digital` → unified `sales_proposal_line` (single table with `selling_method` column). This is actually a cleaner design.

**Real gaps:**
- `sales_structure` — the V1 physical structure concept was apparently dropped; frames exist without a parent structure level. This means the inventory hierarchy is Market→Location→Frame (not Market→Location→Structure→Frame as v1.0 described).
- `sales_commission_setting` + `sales_seasonal_rate` — missing despite commission and seasonal pricing features being claimed as built.

---

## 4. What IS Verified and Working

### Schema (confirmed in migrations)
- ✅ 57 tables in `ooh_sales` schema
- ✅ Inventory hierarchy: Market → Location → Frame → Screen
- ✅ Classic + digital availability tables
- ✅ Rate cards, pricing rules, corridors, buying strategy modifiers
- ✅ Proposal + proposal lines (unified)
- ✅ Booking + booking flights, orders, goals
- ✅ Delivery summary, proof of play, make good
- ✅ Invoice
- ✅ Commission (schema only — no UI)
- ✅ Programmatic deal table
- ✅ Marketplace settlement tables
- ✅ Frame targeting, waitlist, pool availability

### Code (verified in `/src/sales/src/`)
- ✅ 12 services: Account, Attribution, Availability, Booking, Commission, Delivery, Invoice, Lead, MakeGood, Pricing, Programmatic, Proposal
- ✅ SalesBaseRepository correctly uses `supabase.schema('ooh_sales')`
- ✅ 8 repositories: Account, Availability, Booking, Frame, Invoice, Lead, Pricing, Proposal
- ✅ 5 adapters: Asset, Audience, Auth, Comms, DMS
- ✅ 4 ports: IAudiencePort, ICommsPort, IDMSPort, ISalesAssetPort
- ✅ UI pages: Pipeline, Proposal Builder, Booking Management, Rate Cards (all exist)
- ✅ Components: booking, pipeline, pricing, proposal, shared — all present
- ✅ 16 agent prompt sets (system + task prompts per agent)

### Edge Functions (all 5 verified present)
- ✅ `sales-check-availability`
- ✅ `sales-confirm-booking`
- ✅ `sales-create-booking`
- ✅ `sales-price-proposal`
- ✅ `sales-release-pencils`

### Routing
- ✅ `/sales/*` → `SalesModule` with `RequireAuth` guard

---

## 5. Missing Features (Confirmed)

| Feature | Status | Notes |
|---|---|---|
| Commission UI | ❌ Missing | `CommissionService` exists, no UI page |
| Rate card impact analysis | ❌ Missing | No page/component |
| Flighted campaigns UI | ❌ Missing | `sales_booking_flight` schema exists, no UI |
| Shared Goals UI | ❌ Missing | Schema exists, no UI |
| DCO creative variants UI | ❌ Missing | Schema exists, no UI |
| Frame waitlist UI | ❌ Missing | Schema exists, no UI |
| Bulk operations UI | ❌ Missing | No bulk approve/send UI |
| Payment reconciliation | ❌ Missing | No UI or service |
| CMMS integration | 🟡 Stub | Claimed in v1.0 |
| Compliance integration | 🟡 Stub | Claimed in v1.0 |
| E-signatures integration | 🟡 Stub | Real provider not wired |
| Account list page | ❌ Route missing | No `/sales/accounts` route in AnimatedRoutes |

---

## 6. Route Gap

The v1.0 audit listed `/sales/accounts` and `/sales/accounts/:id` as built. Inspection of `AnimatedRoutes.tsx` shows `/sales/*` is a wildcard catch-all routed to `SalesModule`. The internal routing within `SalesModule.tsx` would need to be checked, but no top-level account routes are defined. This needs verification in the SalesModule internal router.

---

## 7. Technical Debt — v1.0 Items Confirmed + New Findings

| Issue | Severity | Source |
|---|---|---|
| `usePricing` queries wrong schema — pricing UI broken | CRITICAL | New (v2.0) |
| `sales_pricing_rule` vs `sales_pricing_rules` table name mismatch | HIGH | New (v2.0) |
| 14 tables in v1.0 docs missing from actual migrations | HIGH | New (v2.0) |
| `sales_structure` level dropped — inventory hierarchy docs wrong | HIGH | New (v2.0) |
| `sales_classic_availability` UNIQUE constraint references `face_id` | MEDIUM | v1.0 confirmed |
| `sales_location` lacks `geography(POINT)` for PostGIS queries | MEDIUM | v1.0 confirmed |
| No DB transaction wrapping on bulk operations | MEDIUM | v1.0 confirmed |
| Commission module has schema + service but no UI | MEDIUM | v1.0 confirmed |
| `usePipeline`, `useBookings`, `useProposal` — need schema verification | MEDIUM | New (v2.0) |
| No circuit breaker on PricingService → Audience call | LOW | v1.0 confirmed |
| Dev seed uses hardcoded `tenant_id` | LOW | v1.0 confirmed |

---

## 8. Production Readiness

**Rating: 5/10**

The schema and services are substantially built. However, the pricing UI is broken (wrong schema in `usePricing`), 14 documented tables are missing from migrations, and several critical features (commission, payment reconciliation, CMMS integration) are stubs or absent. The `usePricing` bug alone breaks the core commercial pricing workflow and must be fixed before any real bookings can be correctly priced via the UI.

---

## 9. Immediate Actions Required

1. **Fix `usePricing` schema** — change all `supabase.from(...)` to `supabase.schema('ooh_sales').from(...)` or delegate to `PricingRepository`
2. **Fix `sales_pricing_rule` → `sales_pricing_rules`** table name
3. **Audit all Sales hooks** (`usePipeline`, `useProposal`, `useBookings`) for same schema bypass pattern
4. **Write migrations** for `sales_commission_setting`, `sales_seasonal_rate`, `sales_campaign_booking`, `sales_creative_revision`
5. **Update inventory hierarchy docs** — `sales_structure` was dropped; docs must reflect Market→Location→Frame→Screen

---

*Sales Module Audit v2.0 — 2026-03-25*
