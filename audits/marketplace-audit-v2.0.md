# Marketplace Module Audit v2.0
**OOH OS — Kai Autonomous Agent**
**Date:** 2026-03-25
**Codebase:** `/workspace/group/everboarding`
**Sources:** v1.0 audit doc, live codebase scan, migration SQL, Supabase types, CLAUDE.md

---

## 1. Scope of This Audit

This audit cross-references the v1.0 design document against what has actually been built, identifies schema gaps and mismatches, assesses the Phase 3 readiness blockers, and rates production readiness.

---

## 2. What the Design Says vs What Is Built

### 2.1 Marketplace as a Separate Application

**Designed:** The Marketplace is architecturally separate from the operator Sales module — its own app (`/apps/marketplace`), its own package (`/packages/marketplace-core`), its own database scope, its own buyer user types, and its own revenue streams (listing fees, GMV commission, data products).

**Built:** Nothing. There is no `/apps/marketplace`, no `/packages/marketplace-core`, no marketplace buyer schema, no marketplace UI routes, no `MarketplaceBuyer`, `MarketplaceOperator`, `MarketplacePlan`, `MarketplaceRFP`, or `MarketplaceCampaign` tables anywhere in the codebase or migrations. The CLAUDE.md §3 module inventory explicitly records status as "Planned — Phase 3".

**Verdict:** Zero application-layer code built for the true Marketplace module.

---

### 2.2 Settlement Tables (ooh_sales Schema)

**Designed (CLAUDE.md §6.13c):**
```
marketplace_transaction {
  client_account_id, operator_tenant_id,
  gross_booking_value, commission_pct, commission_amount, net_to_operator,
  payment_status (pending|charged|settled|refunded|disputed),
  payment_provider_ref, charged_at, settled_at, settlement_batch_id,
  disputed_at, dispute_reason, dispute_outcome
}
marketplace_settlement_batch {
  operator_tenant_id, period_start, period_end,
  transaction_count, gross_total, commission_total, net_total,
  remittance_ref, settled_at
}
```

**Built (migration `20260316161739_create_ooh_sales_schema_part8_creative_delivery_finance.sql`, confirmed in Supabase types):**

`ooh_sales.marketplace_transaction`:
- `id`, `tenant_id`, `booking_id` (FK → sales_booking), `gross_value`, `platform_commission_pct`, `platform_commission_amount`, `operator_net_value`, `settlement_batch_id`, `created_at`

`ooh_sales.marketplace_settlement_batch`:
- `id`, `tenant_id`, `batch_reference`, `period_start`, `period_end`, `total_transactions`, `total_gross_value`, `total_operator_payout`, `status` (pending|processing|completed|failed), `remittance_url`, `created_at`

**Schema Gap Analysis:**

| Field | Designed | Built |
|---|---|---|
| `client_account_id` | Yes | No — missing cross-tenant buyer identity |
| `operator_tenant_id` | Yes (separate column) | Covered by `tenant_id` only |
| `payment_status` (full enum) | pending/charged/settled/refunded/disputed | Not present — no payment lifecycle |
| `payment_provider_ref` | Yes (Stripe ref) | No |
| `charged_at` / `settled_at` | Yes | No |
| `disputed_at` / `dispute_reason` / `dispute_outcome` | Yes | No |
| `commission_total` / `net_total` (batch) | Yes | Approximated by `total_operator_payout` (no commission breakdown) |
| `remittance_ref` | Yes | Replaced by `remittance_url` (weaker) |

**Notable gap:** The built `marketplace_transaction` table uses `buyer_name TEXT` in `ProgrammaticService.ts` but this column does not appear in the migration or Supabase types — it was written into the service (`ProgrammaticService.recordTransaction`) but maps to columns that don't fully exist as designed. The service inserts `buyer_name` and `seller_tenant_id`, `status = 'pending_settlement'`, `net_amount` — none of which appear in the migration schema. This means **ProgrammaticService.ts is broken against the actual DB schema**.

**Verdict:** Settlement tables exist as stubs. They capture basic financial math but are missing the full payment lifecycle (Stripe integration, dispute handling, buyer identity). The `ProgrammaticService.ts` service has column-name mismatches against the actual migration DDL.

---

### 2.3 Mode 2 (marketplace_autonomous) Wiring in Sales Module

**Designed:** `booking_mode` column on `sales_booking` and `sales_proposal`. YieldAgent corridors required on every `marketplace_eligible` frame. Mode 2 agents run at L4 autonomy. PricingRulesEngine applies corridor bounds.

**Built:**
- `booking_mode TEXT CHECK IN ('direct','marketplace_autonomous')` is present in migrations for both `sales_proposal` and `sales_booking` — confirmed.
- `marketplace_eligible BOOLEAN DEFAULT false` is on `sales_frame` — confirmed in migration `part1`.
- Index `idx_sales_frame_marketplace` on `sales_frame(tenant_id) WHERE marketplace_eligible = true` — present.
- `PricingService.ts` checks `bookingMode === 'marketplace_autonomous'` and applies corridor logic.
- `ProposalService.ts` carries the `booking_mode` field.
- `BookingService.ts` carries `booking_mode`.
- `PricingCorridors.tsx` UI component correctly labels corridors as "Mode 2 autonomous marketplace only".

**Not built (Mode 2 gaps):**
- `marketplace_corridor_id`, `marketplace_allocation_pct`, `marketplace_floor_price` columns on `sales_frame` — in the design (§6.13a), not in any migration.
- `platform_revenue_config` table — designed in §6.13e, not in any migration.
- Self-qualification flow (KYB, credit check, T&Cs click-through) — no code exists.
- Client-facing Mode 2 portal or buyer accounts — none.
- `AgentHub.ts` checks `bookingMode === 'marketplace_autonomous'` (line 335) — wired for Mode 2 dispatch routing.
- `Mode2FailureHandler.ts` — fully implemented: SLA tracking, human queue, safe default, escalation, OpMemory writes. Functional but untested end-to-end because no Mode 2 bookings can actually be created without the buyer portal.

**Verdict:** Mode 2 database columns present. Pricing and agent infrastructure partially wired. Frame-level Mode 2 designation columns missing. No buyer-side anything built.

---

### 2.4 InventoryListingAPI

**Designed (§5.2, §17.6):** The Sales module must expose a standardised REST API from day one — `searchFrames`, `getFrame`, `checkAvailability`, `getOpenRTBDescriptor`, `submitRFP`, `submitBid`. This is the interface the Marketplace consumes.

**Built:** Zero. No `InventoryListingAPI` implementation, no endpoint, no stub, no interface TypeScript file found anywhere in the codebase. The CLAUDE.md §17.8 pre-launch requirement explicitly lists this as a blocker: "InventoryListingAPI implemented and tested in Sales module."

**Verdict:** Critical blocker — not started.

---

### 2.5 CMMS "Marketplace" (Module Marketplace / API Marketplace)

**Note on naming collision:** The two files found in `src/cmms/src/lib/repositories/` (`ModuleMarketplaceRepository.ts` and `APIMarketplaceRepository.ts`) are entirely unrelated to the OOH inventory Marketplace. They are the CMMS internal module/integration catalog — a store for installable modules and third-party API integrations within CMMS tenants. These tables (`module_marketplace_catalog`, `api_marketplace_catalog`, `tenant_integration_installations`) serve a separate purpose. No confusion with the OOH inventory Marketplace should be drawn.

---

### 2.6 ProgrammaticService Dead Code Status

CLAUDE.md v5.13 flagged `ProgrammaticService.ts` in Sales as "never imported — dead code." This audit confirms:

- It is exported from `src/sales/src/services/index.ts`
- It is **not** imported or instantiated anywhere else in the codebase
- The service queries `marketplace_transaction` and `marketplace_settlement_batch` using column names (`buyer_name`, `seller_tenant_id`, `status`, `net_amount`) that differ from the actual migration DDL
- The deal creation method queries `sales_programmatic_deal` which is in the migration but with different column constraints (migration uses `preferred_deal`, service uses `preferred` — mismatch)

**Verdict:** Dead code with schema mismatches. Cannot be activated without a fix pass.

---

## 3. Operator Count Requirement

**Designed:** CLAUDE.md §17.8 states: "At least 2 operator tenants live on OOH OS with real inventory" as a hard pre-launch requirement for the Marketplace.

**Current reality:** One operator tenant exists — Faces Displays Limited (Tenant 0, the initial MVP operator). No second tenant has been onboarded. The post-MVP roadmap in CLAUDE.md explicitly calls out "Second tenant onboarding" as a post-MVP item.

**Implication:** Even if the InventoryListingAPI were built and the Marketplace application scaffolded today, it could not meaningfully launch because cross-operator discovery with a single operator is not a marketplace — it is just a branded front-end for one operator's Sales module.

The second tenant gap is not just a business constraint. It is an architectural validation gap: the InventoryListingAPI contract, cross-tenant settlement, and Marketplace DB isolation design have never been stress-tested with real multi-operator data. Schema assumptions (e.g. `operator_tenant_id` in settlement) have not been validated against a real second tenant's data model.

---

## 4. Pre-Phase 3 Blockers (Full Inventory)

These are the six blockers from the v1.0 audit, now assessed against the codebase:

| Blocker | v1.0 Status | v2.0 Reality |
|---|---|---|
| Multiple operator tenants with live inventory | Unbuilt | Still 1 operator. Second tenant is post-MVP roadmap item. |
| `InventoryListingAPI` in Sales module | Not started | Not started. No file, no stub, no interface. |
| Credible audience profiles (Audience module) | Phase 1 only (seeded Malta data) | Phase 1 delivered. Confidence ~0.30 from seeded data. Phase 2 (gravity model) not built. Sufficient for Phase 3a demo; borderline for credible buyer-facing profiles. |
| OpenOOH taxonomy tags on inventory | Partially done | `openooh_parent`, `openooh_child`, `openooh_grandchild`, `openooh_enum_id` on `sales_frame` — in migration, in dev seed. Populated for 12 dev seed frames. Not validated at scale. |
| Legal settlement framework | Not built | No `platform_revenue_config` table, no operator agreement templates, no buyer T&Cs. |
| Operator and buyer agreement templates | Not built | DMS stores operator docs but no marketplace-specific agreement templates exist. |

Additional blockers discovered in this audit:

| Blocker | Detail |
|---|---|
| `ProgrammaticService.ts` schema mismatches | Must be fixed before any Mode 2 transaction can be recorded. Column names differ from migration DDL. |
| `marketplace_transaction` missing payment lifecycle | `payment_status`, `payment_provider_ref`, `charged_at`, `settled_at`, `disputed_at` all absent from migration. Critical for Phase 3b. |
| `sales_frame` Mode 2 columns missing | `marketplace_corridor_id`, `marketplace_allocation_pct`, `marketplace_floor_price` not in any migration. Needed for operator inventory designation (§6.13a). |
| `platform_revenue_config` table absent | Commission model, default rates, settlement frequency — none of this is in the DB. |
| Marketplace buyer schema absent | No buyer accounts, no buyer portal, no RFP storage, no cross-tenant campaign aggregation. |

---

## 5. What Is Actually Ready for Phase 3

### Ready (can be used as foundation):

- `booking_mode` columns on `sales_proposal` and `sales_booking` with correct CHECK constraints
- `marketplace_eligible` flag on `sales_frame` with index
- `marketplace_transaction` and `marketplace_settlement_batch` tables as financial stubs
- `Mode2FailureHandler.ts` — SLA tracking, human queue, escalation fully implemented
- `AgentHub.ts` Mode 2 routing hook (line 335)
- `PricingService.ts` corridor enforcement for `marketplace_autonomous`
- `PricingCorridors.tsx` UI component
- `sales_pricing_corridor` table in migrations
- Settlement batch creation logic in `ProgrammaticService.ts` (logic is correct, column names need fixing)
- Audience Phase 1 data (seeded, low confidence but present)
- OpenOOH tags on `sales_frame` schema
- `Mode2FailureHandler` SLA breach detection

### Not ready (must be built for Phase 3a):

- InventoryListingAPI (entire interface)
- Second operator tenant
- Marketplace application (`/apps/marketplace`)
- Marketplace buyer accounts and portal
- RFP workflow (Phase 3a core feature)
- Operator listing profile management
- Legal and agreement framework

### Not ready (must be built for Phase 3b):

- Full payment lifecycle on `marketplace_transaction`
- Stripe/payment provider integration
- KYB self-qualification flow
- `platform_revenue_config` table and configuration
- `marketplace_corridor_id` / `marketplace_allocation_pct` / `marketplace_floor_price` on `sales_frame`
- `ProgrammaticService.ts` schema fix (column name alignment)
- CreativeAdvisorAgent at L4 for auto-approval
- LeadScoringAgent at L4 for Mode 2 self-qualification

---

## 6. Schema Mismatch Detail: ProgrammaticService vs Migration DDL

`ProgrammaticService.ts` (src/sales/src/services/ProgrammaticService.ts):

| Field used in service | Field in migration DDL | Match? |
|---|---|---|
| `buyer_name` | Not in DDL | No |
| `seller_tenant_id` | Not in DDL (only `tenant_id`) | No |
| `gross_amount` | `gross_value` in DDL | No |
| `platform_fee` | `platform_commission_amount` in DDL | No |
| `net_amount` | `operator_net_value` in DDL | No |
| `status: 'pending_settlement'` | No `status` column in DDL | No |
| `settlement_batch_id` | Present in DDL | Yes |
| deal_type `'preferred'` | Migration CHECK: `'preferred_deal'` | No |

The service also uses `.schema('ooh_sales')` correctly.

**Fix required before any Mode 2 transaction:** Either rewrite the service column names to match the migration, or add the missing columns (`buyer_name`, `seller_tenant_id`, `status`) via a new migration.

---

## 7. Production Readiness Score

**Score: 2 / 10**

| Dimension | Score | Rationale |
|---|---|---|
| Schema foundation | 4/10 | Settlement tables exist but stripped-down; Mode 2 columns on booking/frame present; frame-level Mode 2 designation columns missing; payment lifecycle absent |
| Application code | 1/10 | Zero Marketplace application code. Two CMMS files named "Marketplace" are unrelated. |
| InventoryListingAPI | 0/10 | Not started |
| Mode 2 agent wiring | 4/10 | AgentHub routing, Mode2FailureHandler, PricingService corridors built; ProgrammaticService broken; no end-to-end path |
| Multi-operator readiness | 0/10 | Single tenant only |
| Legal/compliance framework | 0/10 | No templates, no agreement framework for marketplace, no T&Cs |
| Buyer experience | 0/10 | No buyer portal, no buyer accounts |
| Settlement/payments | 2/10 | Tables exist as stubs; no payment provider, no dispute handling |
| **Composite** | **2/10** | |

A score of 2 reflects that the architectural thinking is sound, the internal Mode 2 wiring has meaningful foundations, and the settlement tables are in the right schema — but the actual Marketplace product does not exist as a buildable, runnable thing.

---

## 8. Recommended Build Sequence for Phase 3a

1. **Fix `ProgrammaticService.ts`** — align column names to migration DDL or write a migration to add missing columns. Low-effort, unlocks Mode 2 transaction recording.

2. **Add missing `sales_frame` Mode 2 columns** — `marketplace_corridor_id`, `marketplace_allocation_pct`, `marketplace_floor_price` via new migration. Required for operator inventory designation UI.

3. **Add `platform_revenue_config` table** — commission model, default rate, settlement frequency. Single migration, no UI needed initially (seed with defaults).

4. **Extend `marketplace_transaction`** — add `payment_status`, `buyer_name`, `seller_tenant_id`, `payment_provider_ref`, `charged_at`, `settled_at`, dispute columns.

5. **Implement `InventoryListingAPI` interface** — TypeScript interface file + stub implementation in Sales module. This is the critical path item for Phase 3a.

6. **Onboard second operator tenant** — business/operational task, but required before any meaningful testing.

7. **Scaffold Marketplace application** — buyer auth, operator listing profile pages, map-based frame discovery consuming InventoryListingAPI, basic RFP workflow.

Steps 1–4 are pure code/migration work estimable at 1–2 days. Steps 5–7 are Phase 3a sprint work.

---

## 9. Summary of Designed vs Built

| Component | Designed | Built | Gap |
|---|---|---|---|
| Marketplace app (`/apps/marketplace`) | Yes | No | 100% |
| `marketplace-core` package | Yes | No | 100% |
| Buyer accounts / portal | Yes | No | 100% |
| RFP workflow | Yes | No | 100% |
| InventoryListingAPI | Yes | No | 100% |
| `marketplace_transaction` table | Yes | Partial stub | ~40% |
| `marketplace_settlement_batch` table | Yes | Partial stub | ~50% |
| Mode 2 booking_mode columns | Yes | Yes | 0% |
| `marketplace_eligible` on frame | Yes | Yes | 0% |
| Frame Mode 2 corridor/allocation columns | Yes | No | 100% |
| `platform_revenue_config` | Yes | No | 100% |
| Mode 2 agent routing (AgentHub) | Yes | Partial | ~60% |
| Mode2FailureHandler | Yes | Yes (functional) | 0% |
| PricingService corridor enforcement | Yes | Yes | 0% |
| Settlement batch logic (service layer) | Yes | Yes (broken schema) | Schema fix needed |
| CMMS Module/API Marketplace repos | N/A (unrelated) | Yes | N/A |

---

*Audit conducted by Kai — OOH OS Autonomous Development Agent*
*Sources: codebase at `/workspace/group/everboarding`, migrations, Supabase types, CLAUDE.md v6.0, v1.0 audit document*
