# Site Development Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Stage 4 Complete — Programme Engine Live  
**Source paths audited:** `src/site-dev/`, `CLAUDE.md §19`, `supabase/migrations/20260316150200_programme_types_config_model.sql`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Site Development |
| DB schema | `public` (programme_types, programme_cycles, site_requests — shared schema) |
| Frontend path | `src/site-dev/` (+ `src/portal/councils/` for council-facing routes) |
| Owner | Vertical |
| Phase | Stage 4 complete (Programme Type Builder + Service Provider Portal) |

---

## 2. Current State Summary

The Site Development module manages the full application lifecycle from intake to installation-complete for any programme type (bus shelter rollouts, billboard negotiations, etc.). It is the operational backbone for OOH operators growing their estate.

The **Programme Engine** (implemented Stage 4) is the centrepiece — a config-driven, data-first lifecycle engine that replaces hardcoded phase flows with tenant-defined programme type configurations. This makes the module jurisdiction-agnostic: Malta's 27-status programme type, UK planning permission flows, and any future jurisdiction are all tenant configuration, not platform code.

**What is built:**
- `ProgrammeTypeConfigService` — reads programme type config from DB
- `useApplicationTransitions` hook — config-driven transitions (falls back to legacy hardcoded)
- 6 lifecycle effects auto-firing on transitions
- 16 services in `src/site-dev/services/` (15 domain services + `index.ts`)
- Service Provider Portal (`/provider/*`) with `ProviderDashboard` and `ProviderAssignmentDetail`
- Programme Type Builder UI (Stage 4): `ProgrammeTypesPage` + `ProgrammeTypeBuilderPage`
- 4-tab builder: Phases & Transitions, Document Rules, Notifications, Cycles
- Council portal integration for site request review + pending signatures

**What is pending (Open Decisions):**
1. Transactional save for Programme Type Builder (Supabase RPC vs edge function)
2. Stage 5 validation — second programme type (billboard_negotiated) to validate engine flexibility
3. Drag-and-drop phase reordering (currently button-based; `dnd-kit` planned)
4. Programme type versioning strategy (immutable published vs mutable draft with diff tracking)

---

## 3. Client Journey

### Operator / Site Development Manager
1. Creates Programme Type in Builder UI (`/site-dev/programmes`) — defines phases, transitions, document requirements, notification rules
2. Creates Programme Cycle (capacity window: target_installations, survey/installation date windows)
3. Opens Intake: applications classified against active cycle
4. Applications flow through configured phases: intake_classification → survey_path → planning_application → design_review → installation → complete

### Site Request Applicant (partner/council)
1. Submits site request via portal
2. `PreScreeningService` checks location viability (planning refusal history, technical rejection flags)
3. `ProgrammeCycleService.classifyForCycle()` auto-assigns to first open cycle on `ELIGIBILITY_SUBMITTED`
4. Survey path: service provider assigned → `SurveySubmissionValidator` checks deliverables
5. Technical review: `ArchitectWarrantValidator` validates architect warrant (3-tier: regex → register → blocking check)
6. Design consent: `DesignReviewConsentService` captures formal checklist per programme type
7. Planning: `PlanningRevisionService` manages PA revision loop (7 categories, create/resolve/escalate/withdraw)
8. Installation: `PostCompletionService` runs 6-item checklist (QC, council acceptance, warranty, revenue share, CMS, cycle counter)
9. Completion: `incrementCycleCounter` lifecycle effect increments `programme_cycles.applications_classified`

### Service Provider
1. Assigned to site request by council admin
2. Logs in to `/provider/*` — sees `ProviderDashboard` (work queue)
3. Opens `ProviderAssignmentDetail` — uploads survey deliverables, updates status
4. `OverdueAssignmentService` detects overdue assignments → notifies council via Comms Hub

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `ProgrammeTypeConfigService` | ✅ Built | `src/site-dev/services/ProgrammeTypeConfigService.ts` (6.2KB) |
| `useApplicationTransitions` hook | ✅ Built | Config-driven + legacy fallback |
| 6 lifecycle effects in `siteRequest.ts` | ✅ Built | Auto-fire on transitions |
| `SurveySubmissionValidator` | ✅ Built | 13.0KB — two-tier validation |
| `DesignReviewConsentService` | ✅ Built | 7.0KB — structured consent workflow |
| `PlanningRevisionService` | ✅ Built | 7.4KB — PA revision loop (7 categories) |
| `ProgrammeCycleService` | ✅ Built | 7.0KB — intake classification + capacity |
| `ApplicationClosureService` | ✅ Built | 8.8KB — 7 structured closure reasons |
| `DocumentProvenanceService` | ✅ Built | 7.5KB — SHA-256 chain of custody |
| `ArchitectWarrantValidator` | ✅ Built | 6.0KB — 3-tier warrant validation |
| `PostCompletionService` | ✅ Built | 8.0KB — 6-item post-completion checklist |
| `BatchApplicationService` | ✅ Built | 11.9KB — application groups, shared docs |
| `FrameworkAgreementService` | ✅ Built | 5.8KB — base + supplement term resolution |
| `ProgrammeAnalyticsService` | ✅ Built | 8.2KB — phase timings (mean+P90), bottleneck index |
| `PreScreeningService` | ✅ Built | 5.3KB — location viability check |
| `SiteRequestDocumentService` | ✅ Built | 12.8KB — document management per request |
| `SurveyIntelligence` | ✅ Built | 11.9KB — AI-assisted survey analysis |
| `OverdueAssignmentService` | ✅ Built | 4.2KB — detects overdue, notifies via Comms Hub |
| Programme Type Builder UI | ✅ Built | 4-tab builder, clone/publish/archive |
| Service Provider Portal | ✅ Built | `/provider/*` with work queue + detail |
| Malta 27-status programme type | ✅ Seeded | Tenant config data via migration |
| Transactional save (Programme Type Builder) | 🟡 Known issue | Non-transactional delete-then-insert; future: Supabase RPC |
| Billboard_negotiated programme type | ❌ Missing | Stage 5 validation — not yet created |
| Drag-and-drop phase reordering | ❌ Missing | Planned: `dnd-kit` |
| Programme type version diffing | ❌ Missing | Roadmap: immutable published + diff tracking |
| Agreement architecture integration | ✅ Built | `agreement_architecture_schema` migration applied; `FrameworkAgreementService` updated |

---

## 5. Schema Detail

**Migration:** `20260316150200_programme_types_config_model.sql`  
**Multi-tenancy:** All tables filter by `tenant_id` / `platform_tenant_id` + RLS enforced

### New Tables

| Table | Purpose |
|---|---|
| `programme_types` | Versioned programme type config — `id, tenant_id, code, name, version, status (draft/published/archived), config JSONB` |
| `programme_type_phases` | Named phases per programme type — `phase_key, label, sort_order, is_terminal, badge_variant` |
| `programme_type_documents` | Document rules per phase — `document_category, is_required, max_file_size_mb, allowed_formats` |
| `programme_type_notifications` | Per-transition notification rules — `from_phase, to_phase, event_key, recipient_roles, channels, template_id` |
| `programme_cycles` | Capacity-tracked intake windows — `target_installations, applications_classified, survey_window_start/end, installation_window_start/end, cycle_status` |

### Columns Added to Existing Tables

**`site_requests`:** `programme_type_id`, `programme_type_version`, `programme_cycle_id`, `intake_priority_group`, `current_phase`, `phase_status`

### Agreement Architecture (separate migration)

| Table | Purpose |
|---|---|
| `partner_programmes` | Tenant-defined programme configs (no platform defaults) |
| `partner_agreements` | Framework agreements (37 columns) |
| `agreement_coverage_members` | Coverage scope members |
| `agreement_location_instances` | Per-location agreement instances (immutable after creation) |
| `agreement_renewals` | Renewal tracking |

**`site_requests.governing_instance_id`** — FK to `agreement_location_instances`

---

## 6. Role Model

| Role | Access |
|---|---|
| `site_dev_admin` | Full access — programme type management, all applications, analytics |
| `site_dev_manager` | Application review, cycle management, SP assignment |
| `council_admin` | Council portal — site request review, pending signatures, SP assignment |
| `service_provider` | Provider portal — assigned work queue, deliverable uploads |
| `partner` | Submit applications, track own applications |

---

## 7. UI Screens

| Route | Screen | Status |
|---|---|---|
| `/site-dev/programmes` | `ProgrammeTypesPage` — list, create, clone, publish, archive | ✅ Built |
| `/site-dev/programmes/:id` | `ProgrammeTypeBuilderPage` — 4-tab builder | ✅ Built |
| `/site-dev/applications` | Application list | ✅ Built |
| `/site-dev/applications/:id` | Application detail + transitions | ✅ Built |
| `/site-dev/cycles` | `CycleManager` component — cycle CRUD + capacity | ✅ Built |
| `/provider` | `ProviderDashboard` — assigned work queue | ✅ Built |
| `/provider/:id` | `ProviderAssignmentDetail` — upload deliverables, update status | ✅ Built |
| `/portal/councils/*` | `CouncilDashboard` — site requests, pending signatures | ✅ Built |

**Known issue (Programme Type Builder):** Save is non-transactional (delete-then-insert per step). On error, UI re-hydrates from surviving DB state. Hydration guard (`useRef`) prevents React Query background refetches from clobbering unsaved edits.

---

## 8. EventBus Surface

### Outbound Events (Site Development → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `site_dev.application.phase_changed` | Phase transition completed | Analytics, Comms Hub |
| `site_dev.application.intake_classified` | Application assigned to cycle | Analytics |
| `site_dev.survey.submitted` | Survey deliverables uploaded by SP | Council notification |
| `site_dev.document.gate_passed` | All required docs for phase confirmed | Application workflow |
| `site_dev.application.closed` | Application closed (7 structured reasons) | Comms Hub, Assets |
| `site_dev.cycle.capacity_warning` | Cycle approaching/at capacity | Comms Hub (operator alert) |

### Inbound Events (Platform → Site Development)

| Event | Source | Effect |
|---|---|---|
| `sitedevelopment.site.approved` | Internal (Sales consumes) | Sales pre-creates location record |
| `sitedevelopment.site.activated` | Internal (Assets consumes) | Triggers Assets activation cascade → Sales frame creation |
| `sitedevelopment.permit.approved` | Internal (Compliance consumes) | Compliance + Sales frame compliance status updated |

---

## 9. External Integrations

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| Sales | Site Dev → Sales | EventBus | ✅ Designed — `sitedevelopment.site.approved` → Sales pre-creates location |
| Assets | Site Dev → Assets | EventBus | ✅ Designed — `sitedevelopment.site.activated` triggers frame cascade |
| Compliance | Site Dev → Compliance | EventBus | ✅ Designed — permit approval updates compliance status |
| Comms Hub | Site Dev → Comms Hub | EventBus | ✅ Live — overdue assignment notification |
| DMS | Site Dev → DMS | Via `SiteRequestDocumentService` | ✅ Live — document provenance + SHA-256 hashing |
| AgentHub | Via `SurveyIntelligence` | API call | ✅ Live — AI-assisted survey analysis |

---

## 10. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| Programme Type Builder save is non-transactional | High | Delete-then-insert across phases/documents/notifications — partial failure leaves inconsistent state. Future: Supabase RPC wrapping in `BEGIN/COMMIT` |
| Only one programme type validated (Malta) | Medium | Stage 5 (billboard_negotiated) not yet created — engine flexibility not yet proven for second type |
| `agreement_location_instances.resolved_programme_config` is immutable after creation | Medium | Correct by design but requires care: changes to `partner_agreements` don't retroactively affect issued instances |
| No drag-and-drop phase reordering | Low | Button-based only; `dnd-kit` planned |
| Programme type versioning strategy unresolved | Low | ADR needed: immutable published versions vs mutable draft with diff tracking |
| `programme_type_notifications` uses `hub_templates` FK (BIGINT) | Low | Works correctly but BIGINT PK on `hub_templates` is legacy pattern — new tables use UUID |
| `site_requests.partner_id` is TEXT not UUID | Low | Backward compatibility maintained but inconsistent with rest of schema |
| `can_read_via_hierarchy()` function signature must be exact | Low | `(uuid, uuid, text)` — deviations in RLS policies will silently deny access |

---

*End of Site Development Module Audit v1.0*
