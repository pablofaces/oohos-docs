# Help Desk Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (AI — Claude Sonnet 4.6)
**Audit scope:** Full code + schema inspection against v1.0 baseline
**Source paths audited:**
- `src/helpdesk/` (103 files)
- `supabase/functions/api-helpdesk/`, `supabase/functions/onboarding-ticket-triggers/`
- DB schema via Supabase REST API (live)
- `src/components/layout/AnimatedRoutes.tsx`
- `src/pages/admin/AdminRoutes.tsx`

---

## 1. Executive Summary

The Help Desk module has advanced significantly beyond what the v1.0 audit described. The v1.0 audit was based on inferred state from `CLAUDE.md` and the `api-helpdesk` Edge Function; the actual codebase contains a fully-built, multi-feature support platform. Most items marked "Missing" in v1.0 are now built. The module is architecturally sound but has two production-blocking gaps: admin routes are registered in `AdminRoutes.tsx` (not `AnimatedRoutes.tsx`, which the v1.0 auditor checked), and Sales entity linking does not cover `booking_id` / `invoice_id` / `proposal_id` as v1.0 claimed.

**Production readiness: 6.5 / 10**

---

## 2. v1.0 Discrepancies

### Items v1.0 marked Built — status confirmed correct
| Feature | v1.0 claim | v2.0 finding |
|---|---|---|
| `api-helpdesk` Edge Function | Built | Confirmed — `supabase/functions/api-helpdesk/` exists |
| Admin ticket queue | Built | Confirmed — `HelpdeskDashboard.tsx`, `TicketQueue.tsx`, `TicketDetail.tsx` |
| AI triage | Built | Confirmed — but see §6 below; agents exist as classes, wire-up to live dispatch is unverified |

### Items v1.0 marked Missing — now built
| Feature | v1.0 claim | v2.0 finding |
|---|---|---|
| Satisfaction surveys (CSAT) | Missing | Built — `CSATWidget.tsx`, `useCSAT.ts`, `helpdesk_ticket_satisfaction` table in DB |
| SLA tracking | Missing | Built — `sla_configurations`, `sla_tracking`, `agent_mode2_sla_tracking` tables in DB; `SlaService` via `types/sla.ts`, `types/tieredSla.ts` |
| Escalation workflow | Missing | Built — `EscalationPredictionAgent.ts`, `escalation_configs`, `escalation_queue`, `ticket_escalations` tables in DB |
| Knowledge base | Missing | Built — full KB stack (see §4) |
| Ticket merge / dedup | Missing | Built — `mergeService.ts`, `useMergeTickets.ts`, `MergeTicketDialog.tsx`, `helpdesk_ticket_merge_history` table |

### Items v1.0 described incorrectly
| Field | v1.0 description | v2.0 finding |
|---|---|---|
| Main table name | `helpdesk_tickets` | Actual table is `support_tickets` |
| Message thread table | `helpdesk_ticket_messages` | Actual table is `ticket_comments` |
| Audit log table | `helpdesk_ticket_events` | No such table found; there is `canned_response_usage` as the closest audit log for template tracking |
| SLA config table | `helpdesk_sla_config` | Actual tables are `sla_configurations` and `sla_tracking` |
| Portal frontend path | `src/portal/pages/SupportPage.tsx` | Actual paths are `src/helpdesk/pages/portal/MyTicketsPage.tsx`, `TicketDetailPage.tsx`, `KnowledgeBasePortalPage.tsx` |
| Admin route check | v1.0 checked `AnimatedRoutes.tsx` only | Admin routes live in `src/pages/admin/AdminRoutes.tsx` under `/admin/helpdesk/*` |

### Items v1.0 described as Built — over-stated
| Feature | v1.0 claim | v2.0 finding |
|---|---|---|
| Sales entity context linking | "Every ticket carries Sales entity reference (booking_id / invoice_id / proposal_id)" | `LinkedEntityType` in `ticket.ts` does NOT include `booking`, `invoice`, or `proposal`. Supported entity types are: `onboarding_flow`, `invoice` (✓), `cmms_work_order`, `site_request`, `communication_thread`, `audit_log`. `booking_id` and `proposal_id` linkage are absent. |
| Comms Hub integration | "Message threads linked to tickets" | No Comms Hub EventBus wiring found in `src/helpdesk/`. The `commentService.ts` handles in-platform comments directly. Cross-channel message threading is not implemented. |

---

## 3. Ticketing System Status

**Status: Production-ready for core flow. SLA and multi-channel gaps remain.**

### What is fully built
- `support_tickets` table with `platform_tenant_id`, `client_id`, `ticket_number`, `status`, `priority`, `category`, `source`, `assigned_to_user_id`, `assigned_to_team`, `sla_config_id`, `due_at`, `first_response_at`, `resolved_at`, `linked_entity_type`, `linked_entity_id`
- Full CRUD via `ticketService.ts` — list with filters/pagination, get, create, update, assign, stats
- `ticket_comments` table — threaded reply system
- `ticket_assignments` — assignment audit trail with assigned_from / assigned_to / assigned_by
- `helpdesk_ticket_presence` — real-time agent presence indicators
- `helpdesk_ticket_merge_history` — merge/dedup tracking
- `helpdesk_ticket_satisfaction` — CSAT scores post-resolution
- Smart routing: `useSmartRouting.ts`, `SmartRoutingSuggestion.tsx`
- Canned responses: `canned_responses`, `canned_response_categories`, `canned_response_usage` tables + full service/UI
- Macros: `helpdesk_macros` + `macroService.ts`, `useMacros.ts`, `MacroQuickBar.tsx`
- Onboarding ticket integration: `onboardingTicketService.ts`, `siteRequestTicketService.ts` — tickets can be raised from onboarding and site request flows
- Ticket sources: `portal`, `email`, `phone`, `admin_created`, `api`

### Gaps and issues
- `ticket_comments` has `source_channel` but the `TicketSource` type in code (`src/helpdesk/types/ticket.ts`) does not include `live_chat` as a source. The `liveChatService.ts` `convertToTicket()` function inserts `source: 'portal'` for chat-originated tickets — meaning chat-to-ticket conversions are misclassified at DB level.
- No EventBus wiring for Sales disputes inbound events (`sales.booking.disputed`, `sales.invoice.disputed`, `sales.delivery.shortfall`, `sales.availability.error`) — these are listed as planned in `CLAUDE.md §18.1` but no handler exists in `src/helpdesk/`.
- `supabase` TypeScript types have not been regenerated. All DB access uses `(supabase as any).from(tableName)` type assertions across every service file. This is a technical debt item noted in the service files themselves.

---

## 4. Knowledge Base Status

**Status: Fully built — portal and admin surfaces both live.**

### DB tables (confirmed via REST API)
- `kb_articles` — articles with content, status, metadata
- `kb_categories` — hierarchical categories (parent_id support)
- `kb_article_feedback` — thumbs up/down per article
- `helpdesk_kb_deflection_events` — tracks when KB deflected a ticket before creation

### Services and hooks
- `knowledgeBaseService.ts` — full CRUD: categories, articles, search, feedback submission, stats
- `onboardingKBService.ts` — KB articles specifically for onboarding context
- `useKnowledgeBase.ts`, `useKBDeflection.ts` — hooks for portal and admin consumption
- `deflectionService.ts` — pre-ticket-submission deflection logic
- `DeflectionMetricsCard.tsx` — admin analytics on KB deflection rate

### UI
- Admin: `KnowledgeBasePage.tsx` + `KnowledgeBaseManager.tsx` — full article/category authoring
- Portal: `KnowledgeBasePortal.tsx`, `KnowledgeBasePortalPage.tsx` — client self-service search
- Shared: `KBDeflectionPanel.tsx` — shown in ticket creation flow to deflect with articles
- `OnboardingKBHelpCard.tsx` — contextual KB in onboarding panel

### Routes
- Admin: `/admin/helpdesk/knowledge-base` (AdminRoutes.tsx line 162)
- Portal: `/portal/support/knowledge-base` (AnimatedRoutes.tsx line 350)

### Gaps
- No versioning of KB articles (publish/draft only, no version history)
- No AI-suggested articles during ticket triage (TicketTriageAgent does not call KB search)

---

## 5. Live Chat Status

**Status: Fully built at service + DB + admin UI level. Widget and portal-side flow need verification.**

### DB tables (confirmed via REST API)
- `live_chat_sessions` — session lifecycle (waiting → active → ended/abandoned)
- `live_chat_messages` — per-session message thread
- `live_chat_typing` — real-time typing indicators via Supabase Realtime upsert

### Service capabilities (`liveChatService.ts`)
- Session lifecycle: `startSession`, `acceptSession`, `endSession`, `transferSession`
- Messaging: `sendMessage`, `getMessages`, `markMessagesRead`
- Typing indicators: `updateTyping` with upsert to `live_chat_typing`
- Real-time: `subscribeToSessions`, `subscribeToMessages`, `subscribeToTyping` — all wired via Supabase Realtime `postgres_changes`
- Conversion: `convertToTicket` — creates `support_tickets` record with chat transcript in description, links `ticket_id` back to session

### Admin UI
- `LiveChatDashboard.tsx` — session management for agents
- `LiveChatPage.tsx` — admin page at `/admin/helpdesk/live`
- `AgentAvailabilitySelector.tsx` — agent online/offline state
- `helpdesk_agent_availability` and `helpdesk_agent_skills` tables in DB

### Widget
- `ChatWidget.tsx` in `src/helpdesk/components/widget/` — embeddable chat widget
- `useLiveChat.ts` — hook for widget/portal consumption

### Known issues
- `convertToTicket` inserts `source: 'portal'` rather than a distinct `live_chat` source, as noted in §3
- No Edge Function for server-side chat operations — all chat logic runs client-side via Supabase Realtime, which means no server-enforced queue prioritisation or SLA timers at session start
- `helpdesk_agent_availability` table exists in DB but no background job found that auto-sets agents to `offline` on session expiry

---

## 6. AI Triage Status

**Status: Classes built and well-structured. Runtime dispatch path unverified.**

### Agents built
All three agents extend `AIEnhancedAgent` from `src/agents/shared/AIEnhancedAgent`:

| Agent | File | Capabilities |
|---|---|---|
| `TicketTriageAgent` | `agents/specialists/TicketTriageAgent.ts` | Priority classification (4 levels), category classification (6 types), sentiment detection (4 states), tag extraction, VIP client priority boost, confidence-weighted output |
| `ResponseSuggestionAgent` | `agents/specialists/ResponseSuggestionAgent.ts` | Draft reply suggestions based on ticket content and history |
| `EscalationPredictionAgent` | `agents/specialists/EscalationPredictionAgent.ts` | SLA breach risk scoring, escalation path recommendations, structured escalation reasons |

### Hooks and UI
- `useTicketAI.ts` — React hook that surfaces triage results in UI
- `TicketAIInsights.tsx` — displays AI suggestions in admin ticket detail view
- `HelpdeskAgentAdapter.ts` — adapts helpdesk context for AgentHub dispatch pattern

### Gap: No verified AgentHub registration
The three helpdesk agents are listed in `CLAUDE.md §7.3` as "confirmed from codebase audit" with default tiers (TicketTriage: `approval_required`, ResponseSuggestion: `supervised`, EscalationPrediction: `supervised`). However, no `agent_module_registry` row insertion or `AgentHub.register()` call was found in the helpdesk codebase. The agents are instantiated directly (Pattern A — direct instantiation) but there is no evidence they write to `ooh_agents.operational_memory` in the live flow. This means autonomy advancement tracking and LearningEngine feedback loops are not active for helpdesk agents.

---

## 7. Sales Entity Linking Status

**Status: Partially built — Invoice linkage only. Booking and proposal linkage absent.**

The `LinkedEntityType` enum in `src/helpdesk/types/ticket.ts` (lines 83–89) defines:
```
'onboarding_flow' | 'invoice' | 'cmms_work_order' | 'site_request' | 'communication_thread' | 'audit_log'
```

`invoice` is present. `booking` and `proposal` are **absent**.

The `CLAUDE.md §18.1` specification states four Sales EventBus events should create tickets with Sales entity context:
- `sales.booking.disputed` → `booking_id` context
- `sales.availability.error` → `frame_id` context
- `sales.invoice.disputed` → `invoice_id` context
- `sales.delivery.shortfall` → `booking_id` context

None of these event handlers exist in `src/helpdesk/`. The integration described in `CLAUDE.md §18.1` is architectural intent, not implemented code.

The `support_tickets` table also has `linked_entity_type` and `linked_entity_id` generic columns (confirmed via DB REST API and `ticketService.ts` mapping), so the schema could accommodate these entity types — they simply need to be added to the TypeScript enum and the EventBus handlers need to be written.

---

## 8. Admin vs Portal Routes

### Portal routes (AnimatedRoutes.tsx — confirmed)
| Route | Component | Status |
|---|---|---|
| `/portal/support/tickets` | `MyTicketsPage` | Registered and lazy-loaded |
| `/portal/support/tickets/:ticketId` | `PortalTicketDetailPage` | Registered and lazy-loaded |
| `/portal/support/knowledge-base` | `KnowledgeBasePortalPage` | Registered and lazy-loaded |

**Note:** There is no `/portal/support/new` or ticket creation route visible in AnimatedRoutes. Ticket creation likely happens from within `MyTicketsPage` via a modal (`CreateTicketForm.tsx` component exists in `src/helpdesk/components/portal/`).

### Admin routes (AdminRoutes.tsx — confirmed)
| Route | Component | Status |
|---|---|---|
| `/admin/helpdesk` | `HelpdeskDashboard` | Registered |
| `/admin/helpdesk/tickets/:ticketId` | `AdminTicketDetailPage` | Registered |
| `/admin/helpdesk/knowledge-base` | `KnowledgeBasePage` | Registered |
| `/admin/helpdesk/templates` | `CannedResponsesPage` | Registered |
| `/admin/helpdesk/live` | `LiveChatPage` | Registered |
| `/admin/helpdesk/reports` | `HelpdeskReportsPage` | Registered |

**Key finding for v1.0 auditor:** Admin routes are in `src/pages/admin/AdminRoutes.tsx`, not in `AnimatedRoutes.tsx`. The v1.0 audit only checked `AnimatedRoutes.tsx` and found only portal routes, which led to an incomplete picture of admin coverage.

**Gap:** There is no `/admin/helpdesk/escalations` route despite `escalation_configs` and `escalation_queue` tables existing in the DB. Escalation management has no dedicated admin UI page.

---

## 9. Edge Functions

Only two Edge Functions are relevant to helpdesk:
- `api-helpdesk` — confirmed in `supabase/functions/`
- `onboarding-ticket-triggers` — confirmed; handles ticket creation from onboarding flow events

No helpdesk-specific Edge Functions for:
- KB article semantic search (deflection is client-side keyword search)
- SLA breach monitoring (no cron function found — if SLA timers fire they depend on client-side polling)
- Chat queue management (all real-time, client-side)

---

## 10. Production Readiness Assessment

**Score: 6.5 / 10**

| Area | Score | Rationale |
|---|---|---|
| Core ticketing | 8/10 | Full CRUD, pagination, filters, assignment, CSAT, merge — solid |
| Knowledge base | 8/10 | Complete stack portal + admin; no article versioning |
| Live chat | 7/10 | Service + DB + admin + widget complete; server-side queue/SLA logic absent |
| AI triage | 5/10 | Classes complete and well-designed; AgentHub registration + OpMemory write path unverified |
| Sales entity linking | 3/10 | Only `invoice` linked; `booking` and `proposal` absent; EventBus handlers not built |
| Escalation management | 5/10 | DB + agent built; no admin UI for escalation config; no inbound Sales EventBus handlers |
| SLA monitoring | 6/10 | Config tables and tracking tables exist; no server-side breach detection cron |
| Route coverage | 7/10 | All 6 admin pages routed; no escalation admin page; portal ticket creation route unclear |
| Type safety | 4/10 | All Supabase access uses `(supabase as any).from()` — types not regenerated after schema additions |
| Multi-tenancy | 9/10 | All tables use `platform_tenant_id` + RLS; ticket number generation via RPC |

### Blocking items before production sign-off
1. Supabase TypeScript types must be regenerated (`supabase gen types typescript`) — currently all DB calls bypass TypeScript type safety
2. `TicketSource` enum should include `live_chat` to correctly classify chat-originated tickets
3. Sales EventBus inbound handlers must be built for the 4 dispute/shortfall events listed in `CLAUDE.md §18.1`
4. AgentHub registration for the 3 helpdesk agents must be confirmed or added; OpMemory write must be verified
5. Server-side SLA breach detection (Edge Function cron or database trigger) is needed for SLA SLAs to be enforceable in production

### Non-blocking but recommended
- Add `booking` and `proposal` to `LinkedEntityType`
- Add `/admin/helpdesk/escalations` route and admin page
- Add KB article version history
- TicketTriageAgent should call `deflectionService` to suggest KB articles during triage

---

*End of Help Desk Module Audit v2.0*
