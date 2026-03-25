# Programmatic Gateway Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai
**Previous audit:** v1.0 (2026-03-23, Cascade AI)
**Source paths audited:**
- `src/cms/programmatic/` (3 files)
- `supabase/functions/cms-programmatic-gateway/index.ts`
- `src/cms/agents/ProgrammaticYieldAgent.ts`
- `src/cms/agents/prompts/programmatic-yield/`
- `supabase/migrations/20260318150000_cms_sprint0_schema.sql`
- `src/integrations/supabase/types.ts`

---

## Executive Summary

The Programmatic Gateway module has a critical architectural gap that v1.0 under-reported: **all core business state lives exclusively in-process memory**. The DB tables exist and the Edge Function exists, but neither the gateway's `processWin()` path, nor its screen registration, OpenDirect orders, or reconciliation results ever write to those tables in production code. The Edge Function itself contains two explicit stub comments acknowledging this. The module is test-complete and logic-complete, but is **not production-ready** as a persistent service. Everything else (OpenDirect completeness, SSP adapter scope, yield agent) has specific gaps detailed below.

**Production readiness score: 3 / 10**

---

## 1. Critical Finding: In-Memory Persistence (P0)

### What the code actually does

`ProgrammaticGateway.ts` declares six private in-process stores:

```typescript
private sspConfigs: Map<SSPName, SSPConfig> = new Map();   // line 39
private registrations: ScreenRegistration[] = [];           // line 42
private wins: ProgrammaticWin[] = [];                       // line 43
private reconciliations: ImpressionReconciliation[] = [];   // line 44
private openDirectOrders: OpenDirectOrder[] = [];           // line 45
private oohbjects: OOHbject[] = [];                        // line 46
```

Every method (`registerScreen`, `processWin`, `reconcileImpressions`, `createOrder`, `registerOOHbject`) reads and writes only these arrays. There are no Supabase client calls, no `INSERT` statements, no DB references anywhere in the class. The comment on line 41 reads: `// In-memory stores (production: DB)`. This was intentional scaffolding, not an oversight — but it was never replaced.

### DB tables exist but are never written by the gateway

The migration `20260318150000_cms_sprint0_schema.sql` created `ooh_cms.cms_programmatic_win` and `ooh_cms.cms_ssp_impression_reconciliation` with correct schemas, RLS, and indexes. The Supabase TypeScript types are generated for both tables. But a grep of all `src/` confirms zero `INSERT` or `upsert` calls targeting either table from any TypeScript service layer. The tables are empty in production.

### Edge Function stubs confirm the gap

The Edge Function (`supabase/functions/cms-programmatic-gateway/index.ts`) exposes three routes. Two contain explicit stub comments:

- `action=bid-request` (line ~95): `// Stub: In production, ProgrammaticGateway.generateBidRequest() builds OpenRTB 2.6 payload` — returns a hardcoded `{ status: "generated" }` JSON without actually calling `ProgrammaticGateway`.
- `action=reconcile` (line ~130): `// Stub: In production, ProgrammaticGateway.reconcileImpressions() runs reconciliation` — writes an OpMemory entry then returns `{ status: "completed" }` without running any reconciliation logic.

Only `action=win` writes anything to Supabase — but it writes to `ooh_agents.operational_memory`, not to `cms_programmatic_win`. The win payload is logged as an audit entry, not as a billable win record.

### Impact

- Win records are lost on every container restart or function cold start
- Screen registrations are lost between calls (each Edge Function invocation creates a fresh `ProgrammaticGateway` instance with empty state — the constructor at line 49 shows no DB hydration)
- Reconciliation results are never persisted — the nightly reconciliation job produces no durable output
- OpenDirect orders and OOHbjects exist only for the lifetime of a single in-process instance
- Yield analysis (`getFloorPriceRecommendations`, `getHighValueScreens`) operates on the in-memory `this.wins` array, which is always empty in production

### What must be built

1. DB write in `processWin()` — `INSERT INTO ooh_cms.cms_programmatic_win`
2. DB write in `reconcileImpressions()` — `INSERT INTO ooh_cms.cms_ssp_impression_reconciliation`
3. DB-backed screen registration table (currently no migration exists for this)
4. Constructor hydration or stateless query pattern for `sspConfigs` and `registrations`
5. Replace Edge Function stubs with real `ProgrammaticGateway` method calls
6. OpenDirect order persistence (no table exists for this — see Section 3)

---

## 2. SSP Adapter Status

### Named providers and adapter architecture

The `SSPName` type (`types.ts` line 12) defines: `'hivestack' | 'perion' | 'place_exchange' | 'broadsign' | string`. These four are the only SSPs with named support. There are no named references to Vistar, VIOOH, Lamar, Clear Channel, Outfront, Moving Walls, or Displayce anywhere in `src/cms/`.

The CLAUDE.md `§5.6` lists Vistar Media, Hivestack, Displayce, and Broadsign Reach as priority SSPs. Vistar and Displayce are absent from the implementation. The v1.0 audit claimed "4 SSPs integrated" — this is accurate in name but not in practice.

### There are no SSP adapters

Despite the module being described as "Multi-SSP passthrough", there are no adapter files. The entire SSP layer is a generic `SSPConfig` object with `apiBaseUrl` and `apiKey` fields. There is no HTTP client, no SSP-specific protocol handling, no bid request serialisation, no win notification deserialisation, and no per-SSP authentication scheme.

What exists is a configuration container. An operator can `registerSSP({ sspName: 'hivestack', apiBaseUrl: '...', apiKey: '...' })` and the gateway will accept win notifications tagged with that SSP name. But it does not call out to the SSP. It does not send bid responses in OpenRTB format to the SSP. It does not handle the inbound webhook format variations between Hivestack, Perion, Place Exchange, and Broadsign.

The term "passthrough" in the module comment is misleading. A passthrough implies the gateway forwards bid requests to SSPs and returns their responses. No such forwarding logic exists. The gateway is a win-receiver and a configuration store.

### `externalScreenId` is never populated

`ScreenRegistration.externalScreenId` (`types.ts` line 31) is typed as `string | undefined` with no documentation. The `registerScreen()` method (line 80) does not accept or set this field. This means no SSP-assigned screen identifier is ever stored, making it impossible to correlate OOH OS screens with their representations in SSP inventory systems.

### Summary

| SSP | Named in type | Protocol handling | Auth implementation | Screen reg |
|---|---|---|---|---|
| Hivestack | Yes | None | None | None |
| Perion | Yes | None | None | None |
| Place Exchange | Yes | None | None | None |
| Broadsign | Yes | None | None | None |
| Vistar | No | None | None | None |
| Displayce | No | None | None | None |

---

## 3. OpenDirect 2.1 Completeness

### What is implemented

The `ProgrammaticGateway` implements the following OpenDirect 2.1 operations as TypeScript methods:

- `registerOOHbject()` — product registration with location, venue type, audience estimate, pricing
- `getProducts()` — filtered product listing (type, availability, venueType)
- `createOrder()` — order creation (draft status)
- `addOrderLine()` — line item creation with quantity, dates, cost, targeting
- `assignCreative()` — creative assignment to a line
- `updateLineDelivery()` — delivery stats update
- `getOrders()` / `getOrder()` — order retrieval

The `OOHbject` type covers venue taxonomy, audience estimates, and rate types (CPM/flat/SOV). The `OpenDirectOrder` type covers the buyer organisation, status lifecycle (draft/pending/approved/rejected/completed/cancelled), and line items.

### What is missing

**Order status workflow.** There is no `approveOrder()`, `rejectOrder()`, `cancelOrder()` method, or any method that transitions order status. An order is created in `draft` and has no path to any other status within the class.

**No inventory lock on order creation.** When a buyer creates an OpenDirect order for a product, the product's `availability` field on the `OOHbject` is not updated. Two buyers can create orders for the same product simultaneously with no conflict detection.

**No HTTP API surface.** OpenDirect 2.1 defines a REST API (`GET /products`, `POST /orders`, `POST /orders/{id}/lines`, etc.). These methods exist in the TypeScript class but are not exposed via any Edge Function. The `cms-programmatic-gateway` Edge Function has no routes for OpenDirect endpoints.

**No DB persistence.** As covered in Section 1, all orders and OOHbjects are in-memory only.

**No OOHbjects 1.1 metadata.** The IAB OpenDirect 2.1 for OOH (OOHbjects) standard requires DPAA venue taxonomy fields and specific audience descriptor formats for SSP compatibility. The `OOHbject` type uses a simplified `location.venueType` string rather than the structured `dpaa_venue_type` / `dpaa_venue_sub_type` fields defined in `sales_frame` (confirmed in CLAUDE.md §16C).

**No buyer authentication.** OpenDirect buyers authenticate via OAuth or API keys. No buyer credential management exists.

**OpenDirect completeness score: 4 / 10.** The data model is a reasonable approximation of the standard's object model, but missing order lifecycle, inventory locking, HTTP endpoints, DB persistence, and buyer auth.

---

## 4. ProgrammaticYieldAgent Status

### Implementation

`ProgrammaticYieldAgent.ts` is a complete AgentHub-registered agent extending `AIEnhancedAgent`. It:

- Accepts `fillRates` array as input (deviceId, sspName, fillRatePct, avgCpm, currentFloor, winCount)
- Calls `this.aiAnalyze()` with a structured prompt requesting price adjustments, high-value screens, and zero-fill screens
- Writes an outcome signal via `writeOutcomeSignal()`
- Emits `cms.programmatic.yield_report_ready`
- Has versioned prompts at `src/cms/agents/prompts/programmatic-yield/system.v1.0.md`

The system prompt correctly scopes to any OOH operator, advises conservative ±20% adjustment cycles, and requires human review at `approval_required` tier.

### Data feed gap

The agent receives its `fillRates` input from the caller. It does not query the DB itself. Since `cms_programmatic_win` is never written (Section 1), any invocation of this agent will receive an empty or externally fabricated `fillRates` array. The agent is logically complete but is operating without real signal data in production.

**Yield agent completeness:** Architecture and AI integration are correct. Blocked by in-memory persistence of the underlying data source.

---

## 5. Cross-Operator Programmatic Layer

The cross-operator programmatic layer (Phase 3c in CLAUDE.md §17) does not exist anywhere in the codebase. No marketplace-level SSP aggregation, no cross-tenant OpenRTB bid routing, and no private marketplace deal management across operators is built. This is correctly flagged in the v1.0 audit as "Phase 3c — Missing."

The `InventoryListingAPI.submitBid()` endpoint defined in CLAUDE.md §17.6 has no implementation. The Marketplace module itself is marked "Planned — Phase 3." Nothing in `src/` or `supabase/functions/` serves a cross-operator programmatic function.

This is an expected gap for the current MVP phase, not a defect in the Sprint 5 deliverable.

---

## 6. Schema vs Implementation Delta

| Table | Migration exists | RLS applied | TypeScript types generated | Written by app code |
|---|---|---|---|---|
| `ooh_cms.cms_programmatic_win` | Yes | Yes | Yes | No (Edge Function writes OpMemory only) |
| `ooh_cms.cms_ssp_impression_reconciliation` | Yes | Yes | Yes | No |
| `ooh_sales.sales_programmatic_deal` | Yes | Via ooh_sales RLS | Yes | ProgrammaticService (Sales) |

Note: `sales_programmatic_deal` is owned by the Sales module's `ProgrammaticService.ts`. CLAUDE.md v5.13 flagged `ProgrammaticService.ts` in Sales as "dead code — never imported." That file is separate from the CMS `ProgrammaticGateway`. The CMS gateway has no access to or dependency on the Sales programmatic tables.

The `cms_programmatic_win` table has a `schedule_id` FK to `cms_play_schedule` and a `creative_id` FK to `cms_creative`. These FK relationships are correct and required for the win-to-slot injection flow. However because the table is never written, the FK constraints have never been exercised.

---

## 7. Edge Function Architecture Assessment

The `cms-programmatic-gateway` Edge Function has three routes:

| Route | Auth | DB writes | Status |
|---|---|---|---|
| `?action=win` | API key (`CMS_PROGRAMMATIC_API_KEY`) | OpMemory only | Accepts wins but does not persist them to `cms_programmatic_win` |
| `?action=bid-request` | JWT (user token) | None | Explicit stub — returns fake OpenRTB metadata |
| `?action=reconcile` | Service role | OpMemory only | Explicit stub — returns fake reconciliation result |

The `CMS_PROGRAMMATIC_API_KEY` env var used for SSP win authentication is not documented in CLAUDE.md's credentials table (§ Credentials Available). Its provisioning and rotation process is undefined.

The win route accepts `sspProvider` as an optional field with no validation against registered SSPs. A win notification for a non-existent SSP will be logged to OpMemory without error.

There is no rate limiting on the win endpoint. A malformed or malicious SSP could flood the function.

---

## 8. Test Coverage Assessment

The v1.0 audit states 27 unit tests for `ProgrammaticGateway`. These tests validate the in-memory logic and pass correctly — they test the class in isolation, not against the DB. Given the persistence gap, the tests are accurate for what exists but provide no coverage of:

- DB write paths (none exist to test)
- Edge Function integration (stubs bypass the class entirely)
- Multi-instance state consistency (each test creates a fresh instance)
- SSP protocol handling (no adapters to test)
- OpenDirect order lifecycle transitions

---

## 9. Issues Not Found in v1.0 Audit

The v1.0 audit (2026-03-23, Cascade) reached a "production-ready" conclusion and gave the following as "Built" without qualification:

| v1.0 claim | v2.0 finding |
|---|---|
| `cms_programmatic_win` table — Per-win records | Table exists, never written by gateway code |
| SSP impression reconciliation — nightly | Edge Function reconcile route is an explicit stub |
| Edge Function `cms-programmatic-gateway` — SSP entry point | Deployed but bid-request and reconcile routes are stubs |
| OpenDirect 2.1 (OOHbjects): orders, lines, creative, delivery | Implemented in-memory only; no HTTP endpoints; no order lifecycle transitions |
| Multi-SSP passthrough: Hivestack, Perion, Place Exchange, Broadsign | Named in type union only; no HTTP clients, no protocol handling |

The v1.0 audit appears to have assessed the TypeScript class surface and migration files without tracing the full write path from class method to DB row, and without reading the Edge Function stub comments.

---

## 10. Production Readiness Score: 3 / 10

| Dimension | Score | Rationale |
|---|---|---|
| DB persistence | 0/2 | Win records and reconciliation results never written to DB |
| SSP protocol handling | 0/2 | No HTTP clients, no per-SSP adapters, no bid request/response handling |
| OpenDirect 2.1 | 1/2 | Data model present; no HTTP surface, no lifecycle, no inventory lock |
| Edge Function | 1/2 | Win route accepts and logs inbound; bid-request and reconcile are stubs |
| Yield agent | 1/1 | Agent is correctly built; blocked by data feed gap |
| Cross-operator layer | 0/1 | Phase 3 — not expected at MVP |

**Total: 3/10**

The module is well-architected and the TypeScript logic is sound. It passes 27 unit tests. But it cannot sustain a production programmatic operation because wins evaporate on restart, the Edge Function bid route is a stub, reconciliation produces no durable output, and no SSP integration exists beyond accepting inbound win webhooks with a generic API key.

---

## 11. Recommended Remediation (Priority Order)

**P0 — Win persistence (blocks all revenue tracking)**
Add Supabase client to `ProgrammaticGateway` constructor. In `processWin()`, after building the `programmaticWin` object, `INSERT` into `ooh_cms.cms_programmatic_win`. In the Edge Function `action=win` route, call `ProgrammaticGateway.processWin()` rather than logging to OpMemory only.

**P0 — Reconciliation persistence**
Replace the Edge Function reconcile stub with a call to `ProgrammaticGateway.reconcileImpressions()`. Add DB write in `reconcileImpressions()` targeting `ooh_cms.cms_ssp_impression_reconciliation`.

**P1 — Screen registration persistence**
Create migration for `ooh_cms.cms_programmatic_screen_registration`. Hydrate `sspConfigs` and `registrations` from DB on each stateless invocation (or add a separate registration Edge Function that writes to DB).

**P1 — Bid-request route implementation**
Replace the Edge Function bid-request stub with an actual OpenRTB 2.6 DOOH payload builder. This requires querying `sales_screen` for `openrtb_w`, `openrtb_h`, `openrtb_mime_types`, `floor_price_cpm`, and the Audience module for the audience descriptor.

**P2 — SSP adapters**
Implement per-SSP HTTP adapters (at minimum Hivestack and Broadsign Reach, per CLAUDE.md §16A). Each adapter handles: outbound bid response delivery, inbound win notification format parsing, screen registration API call, and authentication scheme.

**P2 — OpenDirect HTTP surface**
Expose OpenDirect endpoints via a new Edge Function or extend the gateway. Minimum viable: `GET /products`, `POST /orders`, `POST /orders/{id}/lines`, `PATCH /orders/{id}` for status transitions.

**P3 — OpenDirect inventory locking**
On order creation, deduct from `sales_digital_availability` or mark OOHbject availability as `limited`. Implement conflict detection using `SELECT FOR UPDATE`.

---

*Audit produced by Kai — OOH OS development agent. Written to `/workspace/group/programmatic-gateway-audit-v2.0.md`.*
