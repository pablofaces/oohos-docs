# Partners Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Agreement Architecture Complete — Core Services Built  
**Source paths audited:** `src/partners/`, `CLAUDE.md §§7.9a, 18.2`, `supabase/migrations/20260316150300_agreement_architecture_schema.sql`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Partners |
| DB schema | `public` (partners table + agreement tables in shared schema) |
| Frontend path | `src/partners/src/` (minimal — primary UI via portal and site-dev) |
| Owner | Vertical |
| Phase | Agreement architecture complete; commission UI pending |

---

## 2. Current State Summary

The Partners module manages the relationships between OOH operators and their commercial partners — primarily agencies and media buying intermediaries that introduce clients and earn commission, and site/infrastructure partners (councils, landowners) who govern where structures are placed under framework agreements.

Two distinct partner relationship types:
1. **Commercial partners (agencies)** — introduce clients, earn commission rates on bookings
2. **Infrastructure partners (councils/landowners)** — govern site placement under programme agreements

**What is built:**
- `partner_agreements` table (37 columns) with `agreement_location_instances` for per-location governance
- `CoverageResolutionService` — 4-tier cascade (explicit_list → locality → region → all_partners)
- `ContentRestrictionsService` — prohibited categories, proximity rules, exclusivity, scheduling
- `LocationInstanceService` — creates instances on permit/commissioning triggers
- `RenewalService` — automatic/mutually_agreed renewals with escalation rules
- Edge Function `partners-activate-agreement` — atomic conflict check + activation
- Ever-boarding integration for partner onboarding journey
- Commission rate lookup API (used by Sales during proposal pricing)

**What is pending:**
- Partner management UI (CRUD for agency partner records)
- Commission statement/reporting UI
- Agency portal multi-client view (planned in Portal module)
- `partner_programmes` UI builder (operator-defined programme configs)

---

## 3. Client Journey

### Agency Partner
1. Agency onboarded via Ever-boarding → `partners.partner.onboarded` emitted
2. Sales creates `sales_agency_relationship` record with commission rate
3. Agency introduces clients → `sales.account.agency_introduced` event
4. Commission calculated on each confirmed booking → `sales.commission.calculated` emitted to Partners
5. Commission statements generated periodically (UI: roadmap)

### Infrastructure Partner (Council/Landowner)
1. Partner agreement created and activated via `partners-activate-agreement` Edge Function
2. `CoverageResolutionService` resolves which partner governs each site (4-tier cascade)
3. Agreement governs content restrictions per location (via `ContentRestrictionsService`)
4. On planning approval (`PLANNING_APPROVED`): `LocationInstanceService` creates governing instance
5. On installation (`INSTALLED`): governing instance linked via `site_requests.governing_instance_id`
6. Agreement renewal tracked by `RenewalService` (index-linked, fixed_increment, or mutually_agreed)

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `partner_agreements` schema (37 cols) | ✅ Built | Migration `20260316150300_agreement_architecture_schema.sql` |
| `agreement_location_instances` | ✅ Built | Immutable after creation — resolved config locked at instance creation |
| `CoverageResolutionService` | ✅ Built | 4-tier cascade: explicit_list → locality → region → all_partners |
| `ContentRestrictionsService` | ✅ Built | Read-only; consumed by Sales + Compliance |
| `LocationInstanceService` | ✅ Built | Creates instances on lifecycle trigger |
| `RenewalService` | ✅ Built | Auto/mutual renewals, escalation rules, lapse handling |
| Edge Function `partners-activate-agreement` | ✅ Deployed | Atomic conflict check + JWT auth |
| Commission rate lookup API | ✅ Built | Used synchronously by Sales during proposal pricing |
| Ever-boarding integration | ✅ Built | Partner onboarding journey |
| 5 retention classes | ✅ Seeded | Via migration |
| 15 RLS policies (3 per table) | ✅ Built | `get_user_platform_tenant_id()` |
| Partner management UI | ❌ Missing | CRUD for agency partner records |
| Commission statement / reporting UI | ❌ Missing | Schema exists, no UI |
| `partner_programmes` UI builder | ❌ Missing | Operator-defined programme configs — no UI yet |
| Agency portal (multi-client view) | ❌ Missing | Planned in Portal module |

---

## 5. Schema Detail

**Multi-tenancy:** All tables filter by `tenant_id` + RLS enforced

### Core Tables

| Table | Purpose | Key columns |
|---|---|---|
| `partner_programmes` | Operator-defined programme types — no platform defaults | `id, tenant_id, programme_code, name, coverage_scope, content_restrictions JSONB` |
| `partner_agreements` | Framework agreements (37 columns) | `id, tenant_id, partner_id, programme_id, status, coverage_scope, coverage_members, content_restrictions, renewal_config, resolved_programme_config JSONB` |
| `agreement_coverage_members` | Coverage scope members for explicit_list agreements | `agreement_id, member_type, member_id` |
| `agreement_location_instances` | Per-location immutable snapshots (created on permit/commissioning) | `id, agreement_id, location_id, governing_instance_id, resolved_programme_config JSONB, content_restrictions JSONB` |
| `agreement_renewals` | Renewal tracking | `agreement_id, renewal_type, renewal_date, status, escalation_reason` |

### Sales-owned tables referenced by Partners
| Table | Owner | Used by Partners for |
|---|---|---|
| `sales_agency_relationship` | Sales | Commission rate per agency per client |
| `sales_commission_setting` | Sales | Per-rep commission configuration |
| `sales_commission` | Sales | Commission records per booking |

---

## 6. Coverage Resolution (4-tier Cascade)

`CoverageResolutionService.resolveForLocation(locationId)`:

1. **`explicit_list`** — Agreement lists specific locations/structures explicitly → highest priority
2. **`locality`** — Agreement covers all locations in a named locality
3. **`region`** — Agreement covers all locations in a named region
4. **`all_partners`** — Catch-all agreement covering all unmatched locations

Two `explicit_list` agreements for different locations do **not** conflict (intentional design).

---

## 7. EventBus Surface

### Outbound Events (Partners → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `partners.partner.onboarded` | Partner onboarding complete | Sales (creates `sales_agency_relationship`) |
| `partners.agreement.updated` | Commission agreement updated | Sales (updates commission calculation) |

### Inbound Events (Platform → Partners)

| Event | Source | Partners Action |
|---|---|---|
| `sales.commission.calculated` | Sales | Records commission payable |
| `sales.account.agency_introduced` | Sales | Attribution tracking |
| `sitedevelopment.permit.approved` | Site Dev | Triggers `LocationInstanceService.createOnPermit()` |
| `sitedevelopment.site.activated` | Site Dev | Triggers `LocationInstanceService.createOnCommissioning()` |

---

## 8. External Integrations

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| Sales | Bidirectional | EventBus + API | ✅ Live — commission rate lookup (sync) |
| Ever-boarding | Partners ← | EventBus | ✅ Live — partner onboarding journey |
| Site Development | Site Dev → Partners | Lifecycle effects | ✅ Live — instance creation on transitions |
| Compliance | Partners → Compliance | Via `ContentRestrictionsService` | ✅ Live — restrictions read by Compliance |
| Comms Hub | Partners → Comms Hub | EventBus | ✅ Designed — renewal notices, lapse alerts |

---

## 9. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| `agreement_location_instances.resolved_programme_config` is immutable | Medium | Correct by design but changes to `partner_agreements` don't retroactively affect existing instances — operators must be aware |
| No partner management UI | Medium | Partners module is schema + service layer only — no UI for managing agency partner records |
| Commission UI not built | Medium | `sales_commission` and `sales_commission_setting` tables exist but no UI surface |
| `partner_programmes` has no platform defaults | Low | Intentional multi-tenancy design but requires operator onboarding effort to configure |
| Renewal candidates for non-explicit scope are deferred | Low | Mass-inviting renewal candidates not supported by design — operator action required |
| `partners.framework_agreement_id` is plain UUID (no FK constraint) | Low | Discovered during migration — backward compatibility maintained but no referential integrity |
| `site_requests.partner_id` is TEXT not UUID | Low | Pre-existing schema inconsistency |
| Edge Function `partners-activate-agreement` has no rate limiting | Low | Atomic conflict check prevents duplicate activations but no rate limit on endpoint |

---

*End of Partners Module Audit v1.0*
