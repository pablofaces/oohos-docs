# Sales Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** MVP-Complete (March 23 2026)  
**Source paths audited:** `src/sales/`, `supabase/migrations/`, `CLAUDE.md §§6, 13b, 16, 16C, 18`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Sales |
| DB schema | `ooh_sales` |
| Frontend path | `src/sales/src/` |
| Package path | `packages/sales/` (planned — not yet extracted) |
| Owner | Vertical |
| Phase | MVP — Layer 1 complete |

---

## 2. Current State Summary

The Sales module is the commercial engine of OOH OS. As of the March 23 2026 MVP, a single tenant (Faces Displays Limited, Tenant 0) can operate a 1,000-screen network end-to-end: sell, schedule, deliver, prove, invoice, and notify.

**What works today:**
- Full pipeline UI (Pipeline → Proposal Builder → Booking Management → Pricing & Rate Cards)
- 12 backend services covering the full commercial lifecycle
- 11-step PricingRulesEngine with full derivation trace
- 16 registered AgentHub agents (all extending `AIEnhancedAgent`)
- Audience Phase 1 (seeded Malta impression data feeding CPM pricing)
- EventBus wired to CMS (booking.confirmed → SchedulingEngine) and Comms Hub (12 event handlers)
- Dev seed data: 21 entity groups, fixed UUIDs, Malta-specific

**What is stubbed:**
- CMMS integration (stub returns no active maintenance)
- Compliance integration (stub returns all frames eligible)
- E-signatures (stub returns mock signing URL, auto-fires webhook after 5s)
- Marketplace/InventoryListingAPI (designed, not wired to live Marketplace)

**What is not yet built:**
- Mode 2 autonomous marketplace flow (schema designed, agents not at L4)
- Audience Phase 2+ (live census/gravity model data)
- Commission module (schema exists, UI not built)
- Rate card impact analysis UI
- Flighted campaigns UI

---

## 3. Client Journey

### Sales Rep (Mode 1 — Direct Sales)
1. Lead arrives via WhatsApp/email → `EnquiryAgent` creates `sales_lead`
2. Rep opens Pipeline view → sees lead scored by `LeadScoringAgent`
3. Rep opens Proposal Builder → selects frames on map, `ProposalAgent` generates draft with reach estimates from Audience
4. Pricing auto-calculated by `PricingRulesEngine` (11 steps, full trace shown)
5. Proposal PDF generated → sent to client via Comms Hub (email + portal link)
6. Client views proposal in Portal → `sales.proposal.viewed` fires → rep notified
7. Client accepts → `sales.proposals.accept()` → booking order generated → e-signature initiated
8. Booking order signed → `sales.booking.confirmed` emitted → CMS schedules, CMMS flags screens
9. Creative uploaded by client in Portal → `CreativeAdvisorAgent` validates quality
10. Campaign goes live → proof of play ingested → delivery summary updated → Portal shows client
11. Campaign completes → Invoice generated → Comms Hub sends → payment reconciled

### Portal Client
- Views proposals, signs booking orders, uploads creatives, views delivery reports, downloads invoices
- Self-service brief via "run again" (roadmap)

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| Inventory hierarchy (Market→Location→Structure→Frame→Screen) | ✅ Built | 55-table schema, fully normalised |
| Classic availability (per period) | ✅ Built | `sales_classic_availability` |
| Digital availability (per date/daypart) | ✅ Built | `sales_digital_availability`, SELECT FOR UPDATE |
| Rate cards + PricingRulesEngine (11 steps) | ✅ Built | Full derivation trace returned |
| Proposal builder (map + list) | ✅ Built | `ProposalService`, UI complete |
| Booking state machine (18 statuses) | ✅ Built | `BookingService` |
| Invoice generation | ✅ Built | `InvoiceService` |
| 16 AgentHub agents | ✅ Built | All registered, L1 default |
| Audience CPM integration (Phase 1) | ✅ Built | Seeded static data, Step 4 of pricing |
| EventBus (20+ events) | ✅ Built | CMS + Comms Hub wired |
| Dev seed data (21 groups) | ✅ Built | Malta-specific, local only |
| CMMS integration | 🟡 Stub | `cmms.stub.ts` |
| Compliance integration | 🟡 Stub | `compliance.stub.ts` |
| E-signatures integration | 🟡 Stub | `esignatures.stub.ts` |
| InventoryListingAPI | 🟡 Designed | Not wired to live Marketplace |
| Commission UI | ❌ Missing | Schema exists, UI not built |
| Mode 2 autonomous marketplace flow | ❌ Missing | Schema + agents designed |
| Audience Phase 2 (census/gravity) | ❌ Missing | Phase 1 stub active |
| Rate card impact analysis UI | ❌ Missing | Schema designed |
| Flighted campaigns UI | ❌ Missing | Schema exists |
| Shared Goals UI | ❌ Missing | Schema exists |
| Dynamic Screen Selection UI | ❌ Missing | Schema exists |
| DCO creative variants UI | ❌ Missing | Schema exists |
| Frame waitlist UI | ❌ Missing | Schema exists |
| Bulk operations API endpoints | ❌ Missing | Designed in §13b.5 |
| Payment reconciliation | ❌ Missing | Designed in §13b.8 |

---

## 5. Schema Detail

**Schema:** `ooh_sales`  
**Table count:** 55+ tables  
**Multi-tenancy:** All tables include `tenant_id uuid NOT NULL` + RLS enforced  
**Currency:** Sourced from `sales_market.currency_code` — never hardcoded

### Inventory Hierarchy
| Table | Purpose |
|---|---|
| `sales_market` | Top-level geographic unit, has `currency_code`, `country_code` |
| `sales_location` | Physical site with `latitude`, `longitude`, planned `geography(POINT)` |
| `sales_structure` | Physical advertising structure (merges V1 Asset + Panel) |
| `sales_frame` | Bookable unit for classic inventory. Has AI fields + OpenOOH taxonomy |
| `sales_screen` | Digital display child of Frame. Has SSP fields, loop config |
| `sales_format` | Media type definition |
| `sales_network` | Named group of frames (replaces V1 Phase) |
| `sales_network_frame` | Network↔Frame junction |
| `sales_frame_targeting` | Multi-dimensional targeting tags |
| `sales_pool_availability` | Pool-level availability for Dynamic Screen Selection |
| `sales_screen_pool` | Named pool for dynamic campaigns |
| `sales_screen_pool_member` | Pool membership |

### Availability
| Table | Purpose |
|---|---|
| `sales_classic_availability` | Per-frame per-period: `available|penciled|booked|maintenance|blocked` |
| `sales_digital_availability` | Per-screen per-date per-daypart: spots tracking, SELECT FOR UPDATE |
| `sales_frame_waitlist` | Waitlist when frame is penciled by another proposal |
| `sales_availability_check_log` | Audit trail of availability checks per proposal |

### Pricing
| Table | Purpose |
|---|---|
| `sales_rate_card` | Named price list with version, approval status |
| `sales_rate` | Per-frame per-period list price |
| `sales_digital_rate` | Per-screen per-daypart CPM rate |
| `sales_season` | Named season definitions |
| `sales_seasonal_rate` | Season multipliers (stacking multiplicatively) |
| `sales_pricing_rules` | Rules-as-data (9 rule types, 3 action types) |
| `sales_pricing_corridor` | YieldAgent min/max % bounds |
| `sales_buying_strategy_modifier` | Area domination, RON, proximity premiums |
| `sales_commercial_agreement` | Annual commitments + volume bonus tiers |
| `sales_event` | Event overlays with distance-decay pricing |

### Accounts & Pipeline
| Table | Purpose |
|---|---|
| `sales_account` | Advertiser account (direct/agency/trading_desk) |
| `sales_contact` | Contact at an account |
| `sales_agency_relationship` | Agency↔client commission relationships |
| `sales_holding_group` | Agency holding group |
| `sales_lead` | Lead with AI fields, source tracking |
| `sales_proposal` | Quote with 18-status lifecycle, AI fields |
| `sales_proposal_template` | Visual/content template for proposal PDF |
| `sales_proposal_line_classic` | Classic frame line item |
| `sales_proposal_line_digital` | Digital screen line item with 9 selling methods |

### Bookings & Campaigns
| Table | Purpose |
|---|---|
| `sales_booking` | Confirmed booking with state machine, AI fields, rebalancing fields |
| `sales_booking_schedule_classic` | Per-period classic schedule lines |
| `sales_booking_schedule_digital` | Per-date/daypart digital schedule lines |
| `sales_booking_flight` | Flighted campaign periods |
| `sales_booking_goal` | Shared delivery goal |
| `sales_booking_goal_member` | Bookings contributing to shared goal |
| `sales_campaign` | Campaign container |
| `sales_campaign_booking` | Campaign↔Booking junction |
| `sales_creative` | Creative asset with approval workflow + DCO fields |
| `sales_creative_revision` | Revision requests |
| `sales_creative_dco_variant` | Dynamic Creative Optimisation variants |
| `sales_screen_creative_spec` | Accepted creative specs per screen |
| `sales_package_booking` | Booked package instances |
| `sales_inventory_package` | Pre-priced frame bundles |
| `sales_inventory_package_frame` | Package↔Frame junction |
| `sales_inventory_package_screen` | Package↔Screen junction |

### Delivery & Finance
| Table | Purpose |
|---|---|
| `sales_proof_of_play` | Verified digital delivery records |
| `sales_delivery_summary` | Aggregated delivery per booking per screen per day |
| `sales_make_good` | Under-delivery remedy management |
| `sales_commission_setting` | Per-rep commission configuration |
| `sales_commission` | Commission records per booking |
| `sales_programmatic_deal` | SSP deal records |
| `sales_programmatic_deal_screen` | Deal↔Screen junction |
| `sales_programmatic_transaction` | Per-bid-win log |

### Marketplace (Mode 2)
| Table | Purpose |
|---|---|
| `marketplace_transaction` | Autonomous booking transaction records |
| `marketplace_settlement_batch` | Weekly/monthly settlement batches |
| `platform_revenue_config` | Platform-level commission config (above tenant) |

---

## 6. Role Model

| Role | Access |
|---|---|
| `sales_admin` | Full access to all accounts, proposals, bookings, rate cards, commissions |
| `sales_manager` | Approve discounts, confirm bookings, view all accounts |
| `sales_rep` | Assigned accounts only, create proposals, cannot confirm bookings |
| `finance` | Read/write invoices, commissions, payment reconciliation |
| `portal_client` | Own account data only: proposals, bookings, invoices, creatives |
| `agency_user` | Introduced clients' data only |

All access gated by Supabase Auth JWT + RLS. Sales never implements its own auth logic.

---

## 7. UI Screens

| Screen | Path | Status |
|---|---|---|
| Pipeline Dashboard | `/sales` | ✅ Built — lead/proposal pipeline with funnel view |
| Proposal Builder | `/sales/proposals/new` | ✅ Built — map-based frame selector, AI draft generation |
| Proposal View / Edit | `/sales/proposals/:id` | ✅ Built — line items, pricing trace, approval flow |
| Booking Management | `/sales/bookings` | ✅ Built — booking list with state badges, delivery view |
| Booking Detail | `/sales/bookings/:id` | ✅ Built — schedule, creatives, proof of play, invoices |
| Pricing & Rate Cards | `/sales/pricing` | ✅ Built — rate card CRUD, season management |
| Account List | `/sales/accounts` | ✅ Built — account/contact management |
| Account Detail | `/sales/accounts/:id` | ✅ Built — proposals, bookings, health score |
| Commission Admin | `/sales/commissions` | ❌ Missing |
| Rate Card Impact Analysis | `/sales/pricing/impact` | ❌ Missing |

**Key UI components:**
- `SalesModule.tsx` — root module (5,073 bytes)
- Map component (Mapbox/Leaflet with OOH OS design system)
- Proposal viewer with interactive inventory map (shared with Portal)
- Pricing derivation trace display (rep-facing transparency)

**Hooks (src/sales/src/hooks/):** All data fetching via TanStack Query v5

**Adapters (src/sales/src/adapters/):**
- `SalesAudienceAdapter` — bridges `IAudiencePort` to Audience module Phase 1

**Ports (src/sales/src/ports/):**
- `IAudiencePort` — contract for audience data (Phase 1: stub/seeded, Phase 2: live)

---

## 8. EventBus Surface

### Outbound Events (Sales → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `sales.lead.created` | New lead created | Ever-boarding, Comms Hub, LeadScoringAgent |
| `sales.account.created` | Lead won, account created | Ever-boarding |
| `sales.proposal.sent` | Proposal sent to client | Comms Hub, DMS, Portal |
| `sales.proposal.viewed` | Client opens proposal in Portal | AgentHub (RenewalAgent) |
| `sales.proposal.accepted` | Client accepts | Booking creation, Ever-boarding |
| `sales.booking.confirmed` | Booking confirmed | CMS, CMMS, Portal, Assets, Comms Hub, Ever-boarding |
| `sales.booking.live` | Campaign start date reached | Portal, Analytics, Comms Hub |
| `sales.booking.completed` | Campaign end date reached | RenewalAgent, Analytics |
| `sales.booking.cancelled` | Booking cancelled | CMS, CMMS, Marketplace |
| `sales.booking.amended` | Booking dates amended | CMS |
| `sales.booking.displaced` | Displaced by takeover | CMS (rebalancing) |
| `sales.booking.po_required` | PO flag set on booking | Internal notification to rep |
| `sales.booking.po_received` | Agency PO received | Triggers Order Confirmation generation |
| `sales.pencil.created` | Inventory penciled | Availability surface |
| `sales.pencil.expired` | Pencil auto-released | Availability surface + waitlist |
| `sales.proposal.availability_stale` | Live check finds unavailable frames | Comms Hub, Portal |
| `sales.proposal.substitutes_offered` | FrameSubstituteAgent returns recommendations | Portal, Comms Hub |
| `sales.waitlist.frame_available` | Waited-on frame becomes available | Comms Hub (2h window) |
| `sales.waitlist.claim_expired` | Rep didn't claim | Next waitlisted rep |
| `sales.invoice.issued` | Invoice generated | Comms Hub |
| `sales.invoice.overdue` | Payment overdue | Comms Hub |
| `sales.payment.received` | Payment confirmed | Comms Hub |
| `sales.delivery.updated` | Proof of play data received | Portal, Analytics |
| `sales.delivery.report_ready` | Campaign report built | DMS, Comms Hub, Portal, Marketplace |
| `sales.booking.renewal_due` | Renewal approaching | Comms Hub (RenewalAgent-drafted) |
| `sales.creative.pending_approval` | Creative needs review | Comms Hub, Portal |
| `sales.creative.approved` | Creative approved | CMS (schedule content) |
| `sales.campaign.attribution_ready` | Attribution data imported | Portal |
| `sales.rate_card.updated` | Rate card activated | DMS |
| `sales.order_confirmation.issued` | Order Confirmation generated | Comms Hub, DMS |
| `sales.booking_order.generated` | Booking order created | DMS |
| `sales.booking_order.signed` | Contract signed | DMS |
| `sales.commission.calculated` | Commission calculated | Partners |
| `sales.programmatic.win` | SSP bid won | Availability surface, Analytics |

### Inbound Events (Platform → Sales)

| Event | Source | Effect |
|---|---|---|
| `everboarding.onboarding.complete` | Ever-boarding | Unlocks booking confirmation for account |
| `everboarding.onboarding.stalled` | Ever-boarding | Blocks booking confirmation |
| `everboarding.account.health_updated` | Ever-boarding | Visible in Sales account view |
| `cmms.maintenance.scheduled` | CMMS | Blocks `sales_digital_availability` slots |
| `cmms.maintenance.completed` | CMMS | Releases blocked availability |
| `cmms.screen.fault` | CMMS | Critical: marks frame unavailable |
| `cmms.screen.restored` | CMMS | Restores frame availability |
| `cmms.playback.logged` | CMMS | Inserts `sales_proof_of_play` |
| `assets.frame.activated` | Assets | Creates `sales_frame` + availability records |
| `assets.screen.registered` | Assets | Creates `sales_screen` |
| `assets.frame.decommissioned` | Assets | Deactivates frame, cancels availability |
| `compliance.frame.restricted` | Compliance | Blocks future availability for category |
| `compliance.frame.restriction_lifted` | Compliance | Restores availability |
| `dms.document.classified` | DMS | Links classified PO to booking |
| `dms.purchase_order.classified` | DMS | Updates `booking.po_document_url` |
| `partners.partner.onboarded` | Partners | Creates agency_relationship record |
| `cms.campaign.scheduled` | CMS | Stores `cms_campaign_id` on booking |
| `cms.booking.rebalanced` | CMS | Records new screen assignments |
| `cms.playback.failed` | CMS | Triggers make-good process |
| `cms.campaign.completed` | CMS | Marks booking Completed, triggers PoP report |
| `sitedevelopment.site.approved` | Site Dev | Pre-creates location record |
| `comms.whatsapp.new_enquiry` | Comms Hub | Creates `sales_lead` via EnquiryAgent |

---

## 9. External Integrations

| Integration | Type | Status | Notes |
|---|---|---|---|
| Supabase Auth | Auth middleware | ✅ Live | JWT validation, RLS, tenant scoping |
| Xibo CMS (via EventBus) | EventBus → CMS stub | 🟡 Stub | Phase 1: stubbed, EventBus abstraction ready |
| Audience module | API call | 🟡 Phase 1 | Seeded Malta data; Phase 2 = live gravity model |
| DocuSign / PandaDoc | E-signatures webhook | 🟡 Stub | Credential: named per tenant, provider TBD |
| DMS | EventBus + API | ✅ Designed | Document storage for all sales PDFs |
| Comms Hub | EventBus | ✅ Live | 12 event handlers wired |
| Ever-boarding | EventBus | ✅ Live | Onboarding gate on booking confirmation |
| AgentHub | API call | ✅ Live | 16 agents registered, all L1 default |
| Broadsign Reach (SSP) | OpenRTB (planned) | ❌ Not built | Named credential stub exists |
| Google Places | POI API | 🟡 Adapters only | Phase 2b |

---

## 10. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| `sales_structure` has duplicate `structure_type` column | Medium | Schema bug — two `structure_type` declarations in CLAUDE.md |
| `sales_classic_availability` UNIQUE constraint references `face_id` (V1 name) | Medium | Should reference `frame_id` — migration needed |
| `sales_booking_schedule_digital` columns misaligned | Medium | `discount_pct`, `is_penciled`, `pencil_expires_at` listed twice across split blocks |
| No database transaction wrapping on bulk operations | Medium | Bulk approve/send could leave partial state on failure |
| `sales_location` lacks `geography(POINT)` column | Medium | Required for PostGIS proximity queries (planned in Phase 2a migration) |
| Occupancy-tier pricing fields not yet added to availability tables | Medium | `occupancy_pct`, `occupancy_tier` columns designed but not yet in migration |
| Commission module schema exists but no UI | Low | `sales_commission_setting`, `sales_commission` tables unserviced |
| Audience Phase 1 uses hardcoded Malta seed data | Low | Confidence scores will be misleading for non-Malta markets |
| `sales_screen.ssp_platform` enum underdefined | Low | Should be a FK to a config table, not a hardcoded enum |
| PO workflow gated behind `po_workflow_enabled` flag but FM onboarding step not yet updated | Low | §6.10 FM step addition not confirmed as built |
| No circuit breaker on PricingService→Audience API call | Low | If Audience API is slow, pricing requests will timeout |
| Dev seed uses `tenant_id = '00000000-0000-0000-0000-000000000001'` hardcoded | Low | Correct for dev but must never reach staging/production |
| `sales_make_good` and `sales_booking_goal` tables interleaved in schema definition | Low | Cosmetic: CLAUDE.md shows table definitions out of order |
| Mode 2 `booking_mode = 'marketplace_autonomous'` relies on agents at L4 | Future | Requires 180+ days track record per agent trust progression |

---

*End of Sales Module Audit v1.0*
