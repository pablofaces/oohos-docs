# Marketplace Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Vision + Architecture Complete — Not Yet Built  
**Source paths audited:** `CLAUDE.md §§17, 17.1–17.9, 18.3`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Marketplace |
| DB schema | Separate Marketplace DB (not shared with operator tenants) |
| App path | `/apps/marketplace` (not yet created) |
| Package path | `/packages/marketplace-core` (not yet created) |
| Owner | Platform (above tenant layer) |
| Phase | Architecture designed — not yet built (Phase 3a+) |

---

## 2. Current State Summary

The Marketplace is a separate platform product that sits **above** the operator layer. It transforms OOH OS from a vertical SaaS into a platform business — connecting buyers (agencies, brands, trading desks) to inventory across multiple OOH OS operator tenants.

**Critical distinction:** The Marketplace is NOT a feature of the Sales module. It is a separate application with its own database, user types, and revenue model.

**What is designed (not yet built):**
- Full data model: MarketplaceBuyer, MarketplacePlan, MarketplaceRFP, MarketplaceCampaign, MarketplaceSettlement
- `InventoryListingAPI` contract — every operator's Sales module must expose this interface
- 4-phase roadmap: Discovery & Planning → Autonomous Transactions → Cross-Operator Programmatic → Intelligence & Data Products
- Revenue model: listing fee + transaction commission + data products
- `MarketplaceOperator` profile (operator's listing on the Marketplace)

**Pre-requisites before launch:**
1. ≥2 operator tenants live on OOH OS with real inventory
2. `InventoryListingAPI` implemented + tested in Sales module
3. Audience module producing credible profiles for listed frames
4. OpenOOH venue taxonomy tags on all listed frames
5. Legal framework for cross-operator transaction settlement
6. Marketplace operator + buyer agreements

---

## 3. Architecture

```
BUYER LAYER
Agency / Brand / Trading Desk / DSP

        ↓
MARKETPLACE LAYER (/apps/marketplace + /packages/marketplace-core)
Cross-operator inventory discovery
Unified audience data layer
Reach & frequency planning tools
Proposal and transaction engine
Settlement and billing

        ↓  InventoryListingAPI  ↓

OPERATOR LAYER (each tenant's Sales module)
┌────────────┬────────────┬────────────┐
│ Operator A │ Operator B │ Operator C │
└────────────┴────────────┴────────────┘
```

---

## 4. User Types

| User type | Role |
|---|---|
| **Buyer (Agency/Brand)** | Discovers cross-operator inventory, builds plans, requests proposals, manages creatives, tracks delivery, receives consolidated reporting |
| **Media Planner** | Advanced reach/frequency tools, audience index queries, format/venue filtering, budget allocation modelling, export to planning tools |
| **Trading Desk** | Programmatic deal management, private marketplace, bid strategy, performance reporting |
| **Operator (on Marketplace)** | Manages inventory listing profile, sets floor prices + blocked categories, reviews/accepts RFPs, views marketplace revenue alongside direct |

---

## 5. Phase Roadmap

### Phase 3a — Discovery & Planning (MVP)
- Operator inventory via `InventoryListingAPI`
- Map-based frame discovery across operators
- Audience data overlay per operator's Audience module
- Basic reach + frequency estimation
- RFP submission → operator sales team responds (Mode 1)
- Platform earns: listing fee per operator

### Phase 3b — Autonomous Transactions (Mode 2)
- Client self-qualifies (LeadScoringAgent L4, KYB, credit check, T&Cs)
- `PricingRulesEngine` calculates price within YieldAgent corridors
- Client books directly — no operator sales team involved
- `CreativeAdvisorAgent` validates quality + compliance automatically
- CMS schedules autonomously
- Platform charges client, deducts commission, remits net to operator
- Settlement: weekly or monthly per operator
- Platform earns: commission % of GMV (suggested 8–15%)

### Phase 3c — Cross-Operator Programmatic
- Private marketplace deal management across operators
- Cross-operator programmatic guaranteed (OpenDirect)
- Open auction aggregation (OpenRTB)
- DSP connectivity via marketplace SSP layer

### Phase 3d — Intelligence & Data Products
- Cross-operator audience benchmarking (anonymised CPM benchmarks per market)
- Market-level inventory availability forecasts
- Campaign performance attribution across operators
- Audience data products sold to agencies and researchers
- `MarketIntelligenceAgent` — feeds cross-operator benchmarks back to each tenant's YieldAgent

---

## 6. Data Model (Planned)

Marketplace has its own database — does NOT share schemas with operator tenants.

| Entity | Key fields |
|---|---|
| `MarketplaceBuyer` | `id, company_name, buyer_type (agency/brand/trading_desk), holding_group, country, currency_preference` |
| `MarketplaceBuyerUser` | `id, buyer_id, role (planner/buyer/finance/admin), email, name` |
| `MarketplaceOperator` | `id, tenant_id (FK to OOH OS tenant), display_name, listing_status, floor_price_override_enabled, blocked_categories[]` |
| `MarketplacePlan` | `id, buyer_id, name, target_markets[], target_formats[], date_range, budget, status` |
| `MarketplacePlanFrame` | `plan_id, operator_id, frame_id, dayparts[], estimated_impressions, indicative_price, status` |
| `MarketplaceRFP` | `id, plan_id, operator_id, status, operator_proposal_ref, agreed_value` |
| `MarketplaceCampaign` | `id, buyer_id, rfp_ids[], total_value, start_date, end_date, status` |
| `MarketplaceDeliveryReport` | `campaign_id, operator_id, frame_id, report_date, spots_booked, spots_delivered, impressions_delivered` |
| `MarketplaceSettlement` | `id, campaign_id, operator_id, gross_value, marketplace_fee_pct, operator_net_value, status` |

---

## 7. InventoryListingAPI Contract

Every operator's Sales module must expose this interface. It is the sole data bridge between the Marketplace and operator inventory:

```typescript
interface InventoryListingAPI {
  searchFrames(params): FrameListing[]
  getFrame(frameId): FrameDetail
  checkAvailability(params): AvailabilityResult[]
  getOpenRTBDescriptor(screenId): OpenRTBDoohDescriptor
  submitRFP(rfp): { proposal_id, estimated_response_time }
  submitBid(bidRequest): OpenRTBBidResponse
}
```

`estimated_response_time` sourced from `tenant_business_rules.sla_rules.rfp_response_hours`.

---

## 8. Revenue Model

| Stream | Phase | Description |
|---|---|---|
| Operator listing fee | 3a | Monthly SaaS fee to list inventory |
| Transaction fee | 3b | % of marketplace-transacted GMV (suggested 2–5%) |
| Premium listing | 3b | Promoted placement in discovery results |
| Data products | 3d | Aggregated audience + market intelligence reports |
| API access | 3c | Agency DSPs paying for `InventoryListingAPI` access |

---

## 9. What Marketplace is NOT

- Not a replacement for direct sales — operators keep 100% of direct revenue
- Not an SSP — does not participate in real-time bidding (that's the SSP layer)
- Not a DSP — buyers still use their own planning tools
- Not a CMS — creative management stays in each operator's CMS module
- Not multi-tenant Salesforce — purpose-built for OOH, not generic

---

## 10. Current Gaps (Pre-Launch Blockers)

| Gap | Blocker for |
|---|---|
| `InventoryListingAPI` not yet wired in Sales | Phase 3a |
| Only 1 operator tenant (Faces) on OOH OS | Phase 3a — need ≥2 |
| Audience Phase 2 (live data) not built | Phase 3a — credible profiles required |
| OpenOOH tags not on all frames | Phase 3a |
| Legal framework for settlement | Phase 3a |
| Mode 2 agents at L4 (180-day track record) | Phase 3b |
| `/apps/marketplace` app not yet created | Phase 3a |
| `MarketplaceOperator` agreement template | Phase 3a |

---

*End of Marketplace Module Audit v1.0*
