# Comms Hub Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Phase 5 Complete — Multi-Channel Live  
**Source paths audited:** `apps/comms-hub/`, `supabase/functions/hub-*/`, `supabase/functions/comms-*/`, `CLAUDE.md §§9.3, 18.1`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Comms Hub |
| DB schema | `public` (hub_templates, hub_message_logs, channel_configs, in_app_notifications) |
| Frontend path | `apps/comms-hub/src/` |
| Backend path | `supabase/functions/hub-*/` + `supabase/functions/comms-*/` |
| Owner | Horizontal |
| Phase | Phase 5 complete — 5 channel types live, legacy stack removed |

---

## 2. Current State Summary

The Comms Hub is the single outbound communications gateway for all OOH OS modules. No module constructs message content or calls channel APIs directly — all communications flow through the Hub's event-driven dispatch pattern.

**Architecture pattern:** Modules emit events → `hub_flow_definitions` maps event to flow → `hub-notifications` Edge Function dispatches via configured channels.

**What is built (5 completed phases):**
- Phase 0: Template consolidation — all 12 legacy `email_templates` rows migrated to `hub_templates`
- Phase 1: All `email_templates` reads eliminated — zero `.from('email_templates')` in codebase
- Phase 2: FK redirect — all references point to `hub_templates`
- Phase 3: UI unification — `EmailManagement.tsx` redirects to `/comms/messaging/templates`
- Phase 4: Legacy table drop — `email_templates` removed
- Phase 5A: Multi-channel dispatch — `channel_configs`, `in_app_notifications` tables; 5 channel types
- Phase 5B: In-app notification system — Supabase Realtime subscription, dual data sources
- Phase 5C: Admin channel config UI — live CRUD on `channel_configs`
- Phase 5D: TypeScript types regenerated with new tables
- Phase 5E: Channel selector in template builder

**Single Resend caller:** Only `hub-notifications/index.ts` calls the Resend API — no module bypasses this.

---

## 3. Client Journey

### Operator / Admin
1. Configures channels at `/comms/channels` (ChannelsPage) — provider credentials per channel (Resend, Twilio, Meta, FCM)
2. Creates templates at `/comms/messaging/templates` — multi-channel, with AI subject/body suggestions
3. Creates automation flows — event → template mapping via `hub_flow_definitions`
4. Monitors delivery via health metrics from `hub_message_logs` (24h sent/failed/delivery rate)
5. Views conversation threads and broadcast campaigns

### Message Delivery Flow
1. Module emits event (e.g. `sales.proposal.sent`)
2. `hub_flow_definitions` resolves template + recipient + channel for that event
3. `hub-notifications` Edge Function dispatches: email (Resend), SMS (Twilio), WhatsApp (Meta), Push (FCM), or in-app
4. Delivery status webhook received (comms-resend-webhook, comms-twilio-webhook, comms-meta-webhook)
5. `hub_message_logs` updated with sent/delivered/bounced/complained status
6. Bounces + complaints trigger `email_suppressions` / `comms_consent` revocation

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `hub-notifications` Edge Function | ✅ Live | Multi-channel dispatch — sole Resend caller |
| `hub-templates` Edge Function | ✅ Live | Template CRUD |
| `hub-flows` Edge Function | ✅ Live | Flow definition management |
| `hub-analytics` Edge Function | ✅ Live | Delivery analytics |
| `hub-contacts` Edge Function | ✅ Live | Contact management |
| `hub-segments` Edge Function | ✅ Live | Contact segmentation |
| `hub-health` Edge Function | ✅ Live | Health check |
| `comms-resend-webhook` | ✅ Live | Resend delivery webhooks |
| `comms-twilio-webhook` | ✅ Live | Twilio SMS webhooks |
| `comms-meta-webhook` | ✅ Live | Meta WhatsApp webhooks |
| `comms-handle-unsubscribe` | ✅ Live | GDPR unsubscribe handling |
| `comms-check-followups` | ✅ Live | Stale proposal follow-up detection |
| `comms-send-broadcast` | ✅ Live | Bulk broadcast campaigns |
| `comms-sync-wa-templates` | ✅ Live | WhatsApp template sync with Meta |
| `process-email-queue` | ✅ Live | Email queue processor |
| `process-notification-queue` | ✅ Live | Notification queue processor |
| `retry-failed-emails` | ✅ Live | Retry scheduler |
| `send-daily-digest` | ✅ Live | Daily digest scheduler |
| `render-template` | ✅ Live | Template rendering service |
| `channel_configs` table | ✅ Built | Per-tenant provider credentials, RLS |
| `in_app_notifications` table | ✅ Built | Hub-dispatched in-app messages, Realtime |
| `hub_message_logs` table | ✅ Built | Delivery audit trail (all channels) |
| `hub_templates` table | ✅ Built | BIGSERIAL PK, tenant_id TEXT, channel TEXT |
| Admin channel config UI | ✅ Built | Live CRUD on `channel_configs` |
| In-app notification Realtime | ✅ Built | Supabase Realtime subscription |
| ConversationsPage | 🟡 Stub | No backing table — returns empty |
| FeatureFlagsPage | 🟡 Stub | Stubbed |
| ABTestingPage | 🟡 Stub | Stubbed |
| AutomationPage | 🟡 Stub | Stubbed |
| FlowBuilderPage | 🟡 Stub | Stubbed |
| NotificationsPage (Comms Hub admin) | 🟡 Stub | Stubbed |
| WebhooksPage | 🟡 Stub | Stubbed |
| SmsPage, ProvidersPage | 🟡 Stub | Stubbed |
| AuditLogsPage | 🟡 Stub | Stubbed |
| CompliancePage (Comms Hub) | 🟡 Stub | Stubbed |
| AI subject/body generator | 🟡 Stub | Returns `configured: false` |

---

## 5. Schema Detail

**Multi-tenancy:** All tables include `tenant_id` + RLS enforced

| Table | Purpose | Key columns |
|---|---|---|
| `hub_templates` | Master template table (replaces `email_templates`) | `id` (BIGSERIAL), `tenant_id` TEXT, `channel` TEXT, `name`, `subject`, `body_html`, `body_text` |
| `hub_message_logs` | Full delivery audit trail | `id`, `tenant_id`, `template_id`, `channel`, `recipient`, `status` (sent/delivered/bounced/complained/failed), `sent_at` |
| `channel_configs` | Per-tenant channel provider credentials | `tenant_id`, `channel`, `provider`, `credentials JSONB`, `is_active` |
| `in_app_notifications` | Hub-dispatched in-app notifications | `id`, `tenant_id`, `user_id`, `title`, `body`, `read_at`, `created_at` |
| `hub_flow_definitions` | Event → template → channel mapping | `event_key`, `tenant_id`, `template_id`, `channels[]`, `recipient_resolver` |
| `email_suppressions` | Suppressed email addresses | `email`, `tenant_id`, `reason` (bounce/complaint), `suppressed_at` |
| `comms_consent` | Consent records per contact per channel | `contact_id`, `channel`, `status`, `updated_at` |

---

## 6. Channel Support

| Channel | Provider | Status | Webhook |
|---|---|---|---|
| Email | Resend | ✅ Live | `comms-resend-webhook` |
| SMS | Twilio | ✅ Live | `comms-twilio-webhook` |
| WhatsApp | Meta (WhatsApp Business API) | ✅ Live | `comms-meta-webhook` |
| Push | FCM (Firebase Cloud Messaging) | ✅ Live | — |
| In-App | Supabase Realtime (`in_app_notifications`) | ✅ Live | — |

---

## 7. Modules That Call Comms Hub

Pattern: All via `hub_flow_definitions` event mapping — no module calls channel APIs directly.

| Module | Events routed via Comms Hub |
|---|---|
| Sales | proposal.sent, booking.confirmed, invoice.issued, payment reminders, campaign notifications, creative approval |
| CMMS | work_order.created, screen.fault, maintenance.scheduled |
| Ever-boarding | invitation.sent, delegation.email, onboarding.completed, reminders, token.expiry |
| Partners | renewal.initiated, agreement.lapsed, credit.status, signing.requests |
| Portal | self-service action events consumed by hub_flow_definitions |
| Site Development | overdue assignment notifications |

---

## 8. EventBus Surface

### Outbound Events (Comms Hub → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `comms.whatsapp.new_enquiry` | Inbound WhatsApp from unknown contact | AgentHub (EnquiryAgent) |
| `comms.followup.proposal_stale` | Proposal sent >5 days without response | AgentHub (FollowUpAgent) |
| `comms.followup.wa_no_response` | WhatsApp conversation >24h without agent reply | AgentHub (FollowUpAgent) |
| `comms.message.delivered` | Delivery confirmed by provider webhook | Analytics, `hub_message_logs` |
| `comms.message.bounced` | Hard/soft bounce received | `email_suppressions`, `comms_consent` revocation |
| `comms.message.complained` | Spam complaint received | `email_suppressions`, `comms_consent` revocation |
| `comms.consent.revoked` | Contact unsubscribed | Future sends blocked by consent check |

---

## 9. GDPR / Consent Rules

- Every outbound channel send checks `comms_consent` before dispatch — rejected if `status = revoked`
- Bounces and spam complaints auto-revoke consent and add to `email_suppressions`
- `comms-handle-unsubscribe` — processes unsubscribe links (one-click unsubscribe per RFC 8058)
- `track-consent` Edge Function — records consent changes with timestamp
- `submit-dsr-request` + `verify-dsr-identity` — GDPR data subject request handling

---

## 10. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| `hub_templates` uses BIGSERIAL (not UUID) PK | Medium | Legacy pattern — inconsistent with rest of schema. Cannot change without breaking all FK references |
| `hub_templates.tenant_id` is TEXT not UUID | Medium | All other tables use UUID — type mismatch is a latent bug risk in cross-table joins |
| ~14 admin UI pages are stubs | Medium | ConversationsPage, FlowBuilderPage, ABTestingPage etc. — navigation exists but no functionality |
| AI subject/body generator returns `configured: false` | Low | AgentHub integration for template AI not yet wired |
| `comms-sync-wa-templates` requires Meta Business Verification | Low | WhatsApp template messaging requires approved business — operator-level setup required |
| No rate limiting on `comms-send-broadcast` | Low | Bulk sends could breach Resend/Twilio rate limits — needs per-tenant send throttling |
| `hub_flow_definitions` mapping not yet UI-manageable | Low | Flow definitions are currently DB-managed — FlowBuilderPage stub is the planned UI |
| `send-daily-digest` schedule not configurable per tenant | Low | Digest timing should live in `tenant_business_rules` |

---

*End of Comms Hub Module Audit v1.0*
