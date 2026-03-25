# Analytics Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Vision Only — Not Yet Built  
**Source paths audited:** `CLAUDE.md §§18.1 Analytics`, codebase search (no dedicated analytics module found)

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Analytics |
| DB schema | No dedicated schema — reads from `ooh_sales`, `ooh_cms`, `ooh_audience` |
| Frontend path | Not yet built |
| Owner | Horizontal |
| Phase | Vision defined — not yet built |

---

## 2. Current State Summary

The Analytics module is planned as a read-only intelligence layer that aggregates data across Sales, CMS, and Audience for operator reporting. It is not yet built as a dedicated module.

**Current state:** Analytics data is currently surfaced inline within other modules:
- Sales pipeline metrics visible in Sales UI
- Delivery summaries visible in Sales booking detail and Portal
- Audience impression data in Audience module

**Planned capability:** A dedicated Analytics module that provides cross-module reporting without modules needing to build their own reporting layers.

---

## 3. Planned Data Sources

Analytics never writes to Sales schema. Read-only access enforced at DB level.

| Data | Source | Mechanism |
|---|---|---|
| Revenue by market/format/account | `sales_booking`, `sales_proposal` | DB read (Analytics reads `ooh_sales` schema directly — read-only) |
| Fill rate by screen/market | `sales_digital_availability`, `sales_delivery_summary` | DB read |
| Pipeline value | Active proposals × AI-estimated win probability | DB read |
| Win/loss rates | `sales_proposal` status history | DB read |
| Agency performance | Bookings + revenue by `sales_account where account_type = agency` | DB read |
| Commission reports | `sales_commission` table | DB read |
| Real-time booking events | All `sales.*` EventBus events | EventBus consumer |
| Programmatic revenue | `sales_programmatic_transaction` aggregations | DB read |
| Campaign delivery | `cms_delivery_summary` | DB read |
| Audience benchmarks | `audience_frame_profiles` | DB read |

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| Dedicated Analytics module | ❌ Missing | No `src/analytics/` or `apps/analytics/` directory |
| Cross-module reporting dashboard | ❌ Missing | |
| Revenue by market/format/account | ❌ Missing | Data exists in Sales schema but no reporting UI |
| Fill rate heatmap | ❌ Missing | |
| Pipeline value visualisation | ❌ Missing | |
| Win/loss funnel | ❌ Missing | |
| Real-time EventBus consumption | ❌ Missing | Events defined but no consumer built |
| Cross-operator benchmarking | ❌ Missing | Phase 3d (Marketplace) |
| Export to Excel / PDF | ❌ Missing | |
| Inline Sales metrics (in Sales UI) | ✅ Exists | Pipeline, proposal stats surfaced within Sales module |
| Inline delivery metrics (Portal) | ✅ Exists | Proof of play, delivery summaries in Portal and Sales |

---

## 5. EventBus Consumer (Planned)

Analytics is intended to consume all `sales.*` events for real-time dashboards. This is defined in the integration map but not yet implemented.

| Event source | Events consumed (planned) |
|---|---|
| Sales | `sales.booking.confirmed`, `sales.booking.live`, `sales.booking.completed`, `sales.delivery.updated`, `sales.programmatic.win` |
| CMS | `cms.campaign.completed`, `cms.ssp.impression_discrepancy` |

---

## 6. Integration Map (Planned)

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| Sales | Analytics ← | DB read (read-only) + EventBus | ❌ Not built |
| CMS | Analytics ← | DB read + EventBus | ❌ Not built |
| Audience | Analytics ← | DB read | ❌ Not built |

---

## 7. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| Module not yet built | High | All analytics surfaced ad-hoc within individual modules |
| No cross-module reporting | High | Operators cannot compare revenue vs delivery vs audience in one view |
| DB read access to `ooh_sales` requires careful RLS setup | Medium | Analytics read-only access must be enforced at DB level — no write path |
| EventBus consumer not built | Medium | Real-time dashboard requires consuming all `sales.*` events |
| Cross-operator benchmarking blocked until Marketplace Phase 3d | Low | Planned feature, not yet in scope |

---

*End of Analytics Module Audit v1.0*
