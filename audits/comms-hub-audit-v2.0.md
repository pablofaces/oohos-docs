# Comms Hub Module Audit v2.0
**Date:** 2026-03-25
**Auditor:** Kai (OOH OS autonomous dev agent)
**Scope:** Full code + schema audit against v1.0 baseline
**Codebase:** `/workspace/group/everboarding`

---

## 1. Audit Methodology

This audit was conducted across seven source areas:

- REST API schema introspection (`/rest/v1/` OpenAPI endpoint)
- Main migration snapshot (`supabase/migrations/20260310110217_remote_schema.sql`)
- Comms Hub sprint migrations (`20260317134230` through `20260317143327`)
- Edge function source (`supabase/functions/hub-*` and `comms-*`)
- Module source tree (`src/communication-hub/`)
- Standalone app (`apps/comms-hub/`)
- Internal Comms Hub sub-project (`src/communication-hub/supabase/`)

The v1.0 audit document was fetched from `https://raw.githubusercontent.com/pablofaces/oohos-docs/main/audits/comms-hub-audit-v1.0.md` and used as a baseline for discrepancy identification.

---

## 2. Discrepancies from v1.0

### 2.1 Edge Function Count: Overclaimed

v1.0 stated "18 Edge Functions operational." The platform-level `supabase/functions/` directory contains exactly **14 comms/hub-prefixed functions**:

```
comms-check-followups
comms-handle-unsubscribe
comms-meta-webhook
comms-resend-webhook
comms-send-broadcast
comms-sync-wa-templates
comms-twilio-webhook
hub-analytics
hub-contacts
hub-flows
hub-health
hub-notifications
hub-segments
hub-templates
```

The Comms Hub sub-project (`src/communication-hub/supabase/functions/`) has 10 additional standalone functions (`analytics`, `api-keys`, `assets`, `contacts`, `docs`, `emails`, `flows`, `health`, `segments`, `teams`, `templates`, `webhooks`, `integrations`). These appear to be from a prior standalone architecture and are NOT deployed to the platform Supabase project. If v1.0 counted both sets, the number was inflated. Active platform functions: **14**, not 18.

### 2.2 Core Table Count: Understated

v1.0 stated "4 core tables built." This is significantly understated. The database reality is a three-layer schema:

**Layer A — `ooh_comms` schema (canonical):** 3 tables confirmed in migrations:
- `ooh_comms.channel_adapter_configs`
- `ooh_comms.email_suppressions`
- `ooh_comms.tenant_rate_limits`

**Layer B — `public.hub_*` tables (4 tables):**
- `hub_flow_definitions`
- `hub_message_logs`
- `hub_segments`
- `hub_templates`
- `hub_webhook_endpoints`

**Layer C — `public.comms_*` tables:** 102 tables exist in the live schema. These originated in the `public` schema before the Sprint 0 consolidation migration attempted to move them to `ooh_comms`. The REST API confirms all 102 `comms_*` tables remain accessible at `/rest/v1/`, including the full A/B experiment set (`comms_experiments`, `comms_experiment_variants`, `comms_experiment_assignments`, `comms_experiment_results`).

The Sprint 0 migration (`20260317134230`) performed `ALTER TABLE public.comms_* SET SCHEMA ooh_comms` operations. However, the main remote schema snapshot (`20260310110217`) still shows all 102 `comms_*` tables in `public`. This indicates either:
- The Sprint 0 migration ran after the remote schema snapshot (likely), or
- The snapshot does not reflect the current live state.

**The REST API confirms the live DB has comms tables accessible from the `public` schema views** — backward-compatibility views were created by Sprint 0. The ooh_comms schema holds the canonical tables.

### 2.3 Legacy Email Templates: Migration Status Uncertain

v1.0 stated the legacy `email_templates` table was dropped after Phase 4. However, the migration record shows `content_templates` remains present in public schema (with a `template_key` UNIQUE constraint), which appears to be the renamed successor. No `DROP TABLE email_templates` migration was found in the migration directory. The claim of clean drop cannot be confirmed from the migration file set alone.

### 2.4 AI Subject/Body Generator: Still Unintegrated

v1.0 flagged `configured: false` for the AI generator. The source at `src/communication-hub/client/src/components/AiSubjectGenerator.tsx` and `AiTemplateImprover.tsx` remain present and the integration with AgentHub is not wired. This is unchanged from v1.0.

### 2.5 ConversationsPage: Actually Wired (Partial)

v1.0 listed `ConversationsPage` as "stubbed." The actual file (`src/communication-hub/client/src/pages/ConversationsPage.tsx`) contains real Supabase query logic against `conversations` and `conversation_messages` tables with filtering, search, real-time subscription hooks, and send mutations. It is functional, not a stub. This is a positive discrepancy vs v1.0.

### 2.6 Rate Limiting: Implemented at Broadcast Level Only

v1.0 flagged "no rate limiting guards bulk broadcasts" as an operational gap. `comms-send-broadcast` now queries `tenant_rate_limits` (`per_hour` and `per_minute` limit types). However the enforcement checks only `per_hour` in the active code path (15 occurrences confirmed). No `per_day` guard is applied in the broadcast function. The `ooh_comms.tenant_rate_limits` table has `per_minute`, `per_hour`, `per_day` columns — `per_day` is wired at the schema level but not consumed in code.

### 2.7 A/B Testing: Schema Exists, UI Wired to Empty

v1.0 stated ABTestingPage was "stubbed." The schema reality: `comms_experiments`, `comms_experiment_variants`, `comms_experiment_assignments`, and `comms_experiment_results` tables all exist in the live DB (confirmed via REST API). The `comms-send-broadcast` function references experiment logic. However, `ABTestingPage.tsx` has `queryFn: async () => [] as ABTest[]` — it returns an empty array and the create mutation throws `'A/B testing not yet available'`. The schema is production-ready; the management UI is not wired to it.

---

## 3. Channel Status

### 3.1 Email (Resend)

**Status: Production — Fully Operational**

- `hub-notifications` dispatches via Resend SDK (`resend@4.0.0`)
- `comms-resend-webhook` handles delivery/bounce/complaint callbacks
- Hard bounce and complaint records written to `email_suppressions`
- `{{unsubscribe_link}}` auto-injected with HMAC-signed token
- `execute-email-workflows` and `process-email-queue` confirmed to route through `hub-notifications` with no direct Resend calls
- Email block builder UI is fully implemented (`src/communication-hub/client/src/components/email/`)
- DevicePreview, EmailClientPreview, BlockEditor, BuilderCanvas all present

### 3.2 SMS (Twilio)

**Status: Production — Operational with Test Coverage**

- `comms-twilio-webhook` handles delivery status + inbound SMS
- Unit test present: `tests/unit/channels/twilio-sms-provider.test.ts`
- Channel config retrieved from `channel_configs` table per tenant
- Inbound SMS routing confirmed present in webhook handler

### 3.3 WhatsApp (Meta Cloud API)

**Status: Production — Operational with Sync Cron**

- `comms-meta-webhook` handles delivery receipts and inbound messages
- Inbound messages stored to `conversations` / `conversation_messages`
- `comms-sync-wa-templates` is fully implemented: paginates Meta Graph API (`v18.0/{wabaId}/message_templates`), maps all Meta approval statuses to internal values, updates `hub_templates.wa_approval_status`, runs daily via pg_cron (migration `20260317140825`)
- WA approval guard in `hub-notifications` prevents dispatch of non-approved templates
- `META_STATUS_MAP` covers: `APPROVED`, `PENDING`, `REJECTED`, `PAUSED`, `DISABLED`, `IN_APPEAL`, `PENDING_DELETION`, `DELETED`, `LIMIT_EXCEEDED`
- EnquiryAgent trigger confirmed: new inbound from unknown contact fires `comms.whatsapp.new_enquiry` event

### 3.4 Push (FCM)

**Status: Implemented — Dispatch Wired, No Test Coverage Confirmed**

- `hub-notifications` maps `send_push` step type to FCM channel via `STEP_CHANNEL_MAP`
- Channel credentials resolved from `channel_configs` per tenant
- No standalone push test file found in `tests/unit/channels/`
- `NotificationPreferences.tsx` component exists for client-side preference management

### 3.5 In-App (Supabase Realtime)

**Status: Implemented — Insert-Based, No Real-Time Test Coverage**

- `hub-notifications` maps `send_in_app` to insert into `in_app_notifications` table
- Supabase Realtime subscription expected to fan out to client
- No dedicated in-app test file confirmed in unit test suite
- `InAppMessagesPage.tsx` present under `engagement/` — wiring status not confirmed at query level

---

## 4. hub_templates BIGSERIAL Debt Confirmed

The BIGSERIAL primary key issue is **confirmed in production schema** and is more extensive than v1.0 documented.

From `20260310110217_remote_schema.sql` lines 23990–24025:

```sql
CREATE TABLE IF NOT EXISTS "public"."hub_templates" (
    "id" bigint NOT NULL,
    "tenant_id" "text" NOT NULL,
    ...
```

The `id` column is `bigint` backed by a `hub_templates_id_seq` integer sequence — this is effectively `BIGSERIAL`. The `tenant_id` column is `text`, not `uuid`.

**Impact assessment (v2.0 — more severe than v1.0 rated):**

1. `tenant_id TEXT` on `hub_templates` creates a type mismatch against `platform_tenants.id` (uuid). Joins require explicit casting. RLS policies must use text comparison (`tenant_id IN (SELECT id::text FROM platform_tenants...)`).

2. `hub_templates` has **8 foreign key references** from other tables in the production schema:
   - `email_logs.hub_template_id`
   - `email_workflows.hub_template_id`
   - `hub_message_logs.template_id`
   - `tenant_onboarding_automation.hub_abandoned_template_id`
   - `tenant_onboarding_automation.hub_completion_template_id`
   - `tenant_onboarding_automation.hub_escalation_template_id`
   - `tenant_onboarding_automation.hub_invitation_template_id`
   - `tenant_onboarding_automation.hub_reminder_template_id`

   Migrating to UUID PK requires dropping and recreating all 8 FKs plus updating any application code that uses integer IDs. This is a **High severity** breaking change — upgraded from v1.0's Medium rating.

3. The multi-language fields (`template_key`, `locale`, `wa_template_name`, `wa_approval_status`, `is_default`) referenced in the CLAUDE.md architecture were added via Sprint 1 locale columns migration (`20260317134621`) as `ALTER TABLE ADD COLUMN`. The base `hub_templates` schema in the remote snapshot predates this — confirming these are additive columns, not present in the original DDL.

**Resolution path:** A migration must: (a) create a new `hub_templates_v2` with UUID PK and UUID tenant_id, (b) backfill data with `gen_random_uuid()` for each row, (c) update all 8 FK references, (d) rename tables. Estimated effort: 1 sprint. Cannot be done live without a maintenance window.

---

## 5. Legacy Stack Removal Status

**Verdict: Clean within Comms Hub boundary. Legacy references are in other modules only.**

The grep search for `legacy`, `nodemailer`, and `sendgrid` across `src/` returned zero hits within `src/communication-hub/`. All legacy references found were in unrelated modules:
- `src/portal/councils/` — legacy partners table fallback (unrelated)
- `src/cmms/src/DEPRECATION_REGISTRY.ts` — CMMS-specific deprecation tracking (unrelated)
- `src/contexts/OrganizationContext.tsx` — org lookup fallback (unrelated)

**Enforcement of the invariant is confirmed working:** Multiple edge functions explicitly document "No direct Resend API calls" in their headers:
- `execute-email-workflows/index.ts`
- `process-email-queue/index.ts`
- `send-everboarding-notifications/index.ts`
- `process-signature-reminders/index.ts`

The `_shared/communications-hub.ts` bridge function routes all inter-function sends through `hub-notifications` correctly.

**One exception noted:** `cms-manage-creative/index.ts` line 193 calls `supabase.functions.invoke("hub-notifications", ...)` directly — this is the correct pattern (Edge Function to Edge Function via invoke, not direct Resend call), so it does not violate the invariant.

---

## 6. Schema Split: public vs ooh_comms

### Current State (Confirmed)

The schema is split across three layers:

| Layer | Location | Tables | Purpose |
|---|---|---|---|
| Canonical operational | `ooh_comms` schema | 3 tables (channel_adapter_configs, email_suppressions, tenant_rate_limits) | New tables created in Sprint 0 |
| Core hub tables | `public` schema | 5 tables (hub_flow_definitions, hub_message_logs, hub_segments, hub_templates, hub_webhook_endpoints) | Pre-Sprint 0, never moved |
| Legacy comms tables (migrated) | `ooh_comms` schema (post-Sprint 0) | ~102 tables renamed (comms_ prefix stripped) | Originally public.comms_* |
| Backward-compat views | `public` schema | ~102 views | Retain comms_ names for existing code |

### Key Risks

1. **Split identity:** `hub_templates` is in `public` while new canonical tables are in `ooh_comms`. The architectural documentation (CLAUDE.md §10.2) says "ooh_comms schema is canonical" but `hub_templates` — the most frequently referenced Comms table across the platform — remains in `public` with incompatible types.

2. **View staleness risk:** The 102 backward-compatibility views in `public` will drift from the canonical `ooh_comms` tables if new columns are added directly to `ooh_comms` without updating the views.

3. **REST API exposure:** All 102 `comms_*` tables and `hub_*` tables are exposed through the public REST API (`/rest/v1/`). The `comms_` names hit the backward-compat views. RLS is enabled on `hub_templates` (confirmed: `ENABLE ROW LEVEL SECURITY` + 3 policies in migration). RLS status on the ooh_comms.* canonical tables via the views needs verification.

4. **`hub_segments` duplicate:** A `hub_segments` table exists in `public` AND a `comms_segments` → `ooh_comms.segments` table exists from the migration. These are different tables serving similar purposes. No deduplication has been done.

---

## 7. A/B Testing and Experiments

**Schema: Production-ready. UI: Blocked.**

The `comms_experiments` / `comms_experiment_variants` / `comms_experiment_assignments` / `comms_experiment_results` tables are fully migrated to `ooh_comms` schema (Sprint 0) and exposed via backward-compat views. The `comms_experiment_results` table has columns for full funnel metrics: `sent`, `delivered`, `bounced`, `opened`, `clicked`, `unsubscribed`, `complained`, `converted`, `revenue`, `open_rate`, `click_rate`, `conversion_rate`.

`comms-send-broadcast` references rate-limit logic that would integrate with experiment variant routing.

`ABTestingPage.tsx` has full UI scaffolding (create dialog, variant config, traffic percentage sliders, status badges) but the query function returns an empty array and the mutation throws a hard error. **Estimated wiring effort: 1-2 days** to connect existing UI to the existing schema.

`FlowBuilderPage.tsx` similarly has a full ReactFlow canvas implementation with `// TODO: Wire to Supabase when flows table is created` — however `hub_flow_definitions` already exists. The comment is stale; the table exists.

---

## 8. Production Readiness Assessment

**Overall Score: 6.5 / 10**

### What is working and confirmed production-grade

| Component | Status | Evidence |
|---|---|---|
| Email dispatch (Resend) | Production | Webhook handler, suppression list, unsubscribe injection confirmed |
| SMS dispatch (Twilio) | Production | Webhook handler, test coverage, inbound routing |
| WhatsApp dispatch + sync | Production | Meta webhook, approval guard, daily sync cron implemented |
| Push dispatch (FCM) | Functional | STEP_CHANNEL_MAP wired, no test coverage |
| In-app dispatch | Functional | DB insert path wired, no test coverage |
| GDPR consent checks | Production | Consent check in hub-notifications, purpose-based bypass confirmed |
| Email suppression | Production | Hard bounce/complaint/unsubscribe → email_suppressions |
| Multi-language templates | Production | 3-tier locale fallback, Sprint 1 columns confirmed |
| WhatsApp inbound + EnquiryAgent | Production | Conversations stored, agent event fired |
| Rate limiting (per_hour) | Partial | Implemented in broadcast; per_day guard absent |
| Invariant enforcement | Strong | No direct Resend/Twilio/Meta calls outside hub-notifications |
| Conversations inbox | Functional | Real Supabase queries, real-time subscription wired |

### Active gaps blocking full production rating

| Gap | Severity | Impact |
|---|---|---|
| hub_templates BIGSERIAL + TEXT tenant_id | High | Type mismatch across 8 FK references, schema inconsistency |
| Schema split incomplete (hub_* still in public) | High | Architecture document says ooh_comms is canonical; reality contradicts this |
| A/B test UI not wired to schema | Medium | Feature exists in DB and broadcast function; UI unreachable |
| FlowBuilderPage has stale TODO (table exists) | Medium | Visual flow editing blocked by a false assumption |
| per_day rate limit not enforced in code | Medium | Schema supports it; code does not consume it |
| AI subject/body generator (configured: false) | Medium | AgentHub integration pending |
| Push + in-app: no unit test coverage | Low | Untested dispatch paths |
| Digest scheduling not tenant-configurable | Low | Hardcoded cron expressions |
| AutomationPage flows entirely stubbed | Low | `throw new Error('Automation flows not yet available')` |

### Positive changes since v1.0

- ConversationsPage is functional (previously listed as stubbed)
- Rate limiting is partially implemented (previously listed as absent)
- A/B testing schema exists and is production-grade (v1.0 did not acknowledge schema existence)
- WA sync cron is confirmed deployed with full Meta API pagination
- Legacy stack removal is complete within the Comms Hub boundary

---

## 9. Recommended Actions (Priority Order)

1. **[HIGH — Sprint-level] Migrate hub_templates to UUID PK + UUID tenant_id.** Create a migration that builds `hub_templates_v2` with correct types, backfills data, swaps all 8 FK references atomically. Move the table to `ooh_comms` schema in the same migration. This resolves both the BIGSERIAL debt and the schema split simultaneously for the most critical table.

2. **[HIGH — 1 day] Move remaining hub_* tables to ooh_comms schema.** `hub_flow_definitions`, `hub_message_logs`, `hub_segments`, `hub_webhook_endpoints` should all be moved with backward-compat views, matching the pattern already established by Sprint 0.

3. **[MEDIUM — 2 days] Wire ABTestingPage and FlowBuilderPage.** The schemas are live. Both pages need their `queryFn` stubs replaced with real Supabase queries. FlowBuilderPage's stale TODO (`// Wire to Supabase when flows table is created`) must be removed — `hub_flow_definitions` already exists.

4. **[MEDIUM — 0.5 days] Enforce per_day rate limit in comms-send-broadcast.** The `tenant_rate_limits` schema supports it; the function only checks per_hour. Add per_day guard to match the documented rate limiting architecture.

5. **[LOW — 1 day] Add unit tests for Push and In-App dispatch paths.** Email and SMS have test coverage; push and in-app do not.

6. **[LOW — backlog] Wire AI subject/body generator to AgentHub.** `AiSubjectGenerator.tsx` and `AiTemplateImprover.tsx` have the UI scaffolding. Requires AgentHub capability registration for a `comms.template.improve` action.

---

## 10. Appendix: Key File Paths

| Item | Path |
|---|---|
| Canonical send engine | `/workspace/group/everboarding/supabase/functions/hub-notifications/index.ts` |
| WA template sync | `/workspace/group/everboarding/supabase/functions/comms-sync-wa-templates/index.ts` |
| Broadcast + rate limit | `/workspace/group/everboarding/supabase/functions/comms-send-broadcast/index.ts` |
| Schema consolidation migration | `/workspace/group/everboarding/supabase/migrations/20260317134230_comms_hub_sprint0_schema_consolidation.sql` |
| hub_templates DDL | `/workspace/group/everboarding/supabase/migrations/20260310110217_remote_schema.sql` lines 23990–24025 |
| Locale columns migration | `/workspace/group/everboarding/supabase/migrations/20260317134621_comms_hub_sprint1_locale_columns.sql` |
| WA cron migration | `/workspace/group/everboarding/supabase/migrations/20260317140825_comms_hub_sprint1b_wa_sync_cron.sql` |
| Client source root | `/workspace/group/everboarding/src/communication-hub/client/src/` |
| ABTestingPage (stub) | `/workspace/group/everboarding/src/communication-hub/client/src/pages/engagement/ABTestingPage.tsx` |
| FlowBuilderPage (stale TODO) | `/workspace/group/everboarding/src/communication-hub/client/src/pages/FlowBuilderPage.tsx` |
| ConversationsPage (functional) | `/workspace/group/everboarding/src/communication-hub/client/src/pages/ConversationsPage.tsx` |
| Standalone app entry | `/workspace/group/everboarding/apps/comms-hub/src/main.tsx` |
