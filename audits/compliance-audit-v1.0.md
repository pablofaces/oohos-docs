# Compliance Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** GDPR + Frame Eligibility Live — Regulatory Framework Built  
**Source paths audited:** `supabase/functions/` (compliance-*), `CLAUDE.md §§18.1 Compliance, GDPR sections`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Compliance |
| DB schema | `public` (compliance_tasks, comms_consent, GDPR tables) |
| Backend path | `supabase/functions/` (compliance-related) |
| Frontend path | Embedded in Portal (`CompliancePage`) and Admin views |
| Owner | Horizontal |
| Phase | GDPR compliance framework complete; content category restrictions active |

---

## 2. Current State Summary

The Compliance module has two distinct responsibilities:

1. **GDPR / data privacy compliance** — consent management, data subject requests (DSRs), retention enforcement, right to erasure, data portability
2. **Advertising content compliance** — frame eligibility checks against advertiser category restrictions, proximity rules, regulatory permits

Both are operationally critical: a booking cannot be confirmed if Compliance returns a `block` for any line item, and GDPR consent is checked on every outbound Comms Hub message.

**What is built:**
- `create-supplier-compliance-check` — supplier compliance check initiation
- `submit-dsr-request` + `verify-dsr-identity` — GDPR DSR workflow
- `track-consent` — consent recording with timestamp
- `check-deletion-eligibility` — pre-deletion eligibility check
- `enforce-retention-policies` — scheduled data retention (shared with DMS)
- `comms_consent` table — per-contact per-channel consent records
- `CompliancePage` in Portal — scoped compliance task list per user role
- `PartnerCompliancePage` in Portal — partner-specific compliance tasks
- `useScopedComplianceTasks` hook — role-scoped task list
- Frame eligibility API (`compliance.checkFrameEligibility`) — synchronous, consumed by Sales
- `compliance.frame.restricted` / `compliance.frame.restriction_lifted` events

---

## 3. GDPR Framework

### Data Subject Rights (DSR) Workflow
1. DSR submitted via `submit-dsr-request` Edge Function
2. Identity verified via `verify-dsr-identity` (multi-factor verification for sensitive requests)
3. DSR type determined: access / rectification / erasure / portability / objection
4. For erasure: `check-deletion-eligibility` runs (checks for active bookings, legal holds, retention class minimums)
5. If eligible: data deleted / anonymised per retention class rules
6. Confirmation sent via Comms Hub

### Consent Management
- `track-consent` — records consent per contact per channel (email, SMS, WhatsApp, push)
- `comms_consent` table — source of truth for all outbound channel sends
- Comms Hub checks `comms_consent` before every dispatch — `status = revoked` blocks send
- Unsubscribe events auto-update `comms_consent` and add to `email_suppressions`
- `confirm-gdpr-consent` Edge Function — explicit consent confirmation flow

### Retention
- 5 retention classes (seeded) — each with TTL and GDPR legal basis
- `enforce-retention-policies` — scheduled enforcement (shared with DMS)
- `check-document-expiration` — pre-expiry warnings (shared with DMS)

---

## 4. Advertising Content Compliance

### Frame Eligibility API
Synchronous API consumed by Sales during proposal building:

```typescript
compliance.checkFrameEligibility(frameIds, advertiserCategory)
// Returns: { frame_id, eligible: 'eligible' | 'restricted' | 'blocked' }[]
```

- `eligible` — no restrictions apply
- `restricted` — restrictions apply but waivable
- `blocked` — hard block for this category at this frame

**Rule:** No booking can be confirmed if Compliance returns `block` for any line item. `ProposalAgent` must filter restricted frames before including in proposals.

### EventBus integration
- `compliance.frame.restricted` → Sales blocks future availability for category
- `compliance.frame.restriction_lifted` → Sales restores availability

### Restriction types
- **Category restrictions** — advertiser category (alcohol, gambling, etc.) blocked at specific frames (e.g. near schools)
- **Proximity rules** — minimum distance from sensitive venues (set per jurisdiction)
- **Regulatory permits** — frame's planning permit includes content restrictions
- **Partner content restrictions** — `ContentRestrictionsService` in Partners module feeds permitted categories per agreement

---

## 5. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `submit-dsr-request` | ✅ Built | GDPR DSR submission |
| `verify-dsr-identity` | ✅ Built | Multi-factor identity verification |
| `track-consent` | ✅ Built | Consent recording |
| `confirm-gdpr-consent` | ✅ Built | Explicit consent confirmation |
| `check-deletion-eligibility` | ✅ Built | Pre-deletion eligibility check |
| `enforce-retention-policies` | ✅ Built | Scheduled enforcement |
| `create-supplier-compliance-check` | ✅ Built | Supplier compliance initiation |
| `comms_consent` table | ✅ Built | Per-contact per-channel consent |
| `CompliancePage` in Portal | ✅ Built | Scoped task list |
| `PartnerCompliancePage` in Portal | ✅ Built | Partner compliance tasks |
| `useScopedComplianceTasks` hook | ✅ Built | Role-scoped tasks (5.8KB) |
| Frame eligibility API | ✅ Built (stub) | Phase 1: returns all frames eligible (stub) |
| Frame restriction EventBus events | ✅ Designed | `compliance.frame.restricted` / `.restriction_lifted` |
| Compliance admin UI | 🟡 Stub | `CompliancePage` in Comms Hub app is stubbed |
| Automated restriction scanning | ❌ Missing | No automated scan of frame locations vs restriction zones |
| ICO registration / regulatory filings | ❌ Missing | Out of scope for platform — operator responsibility |
| Consent audit export | ❌ Missing | No bulk consent audit export for regulator requests |
| Cross-border data transfer controls | ❌ Missing | Standard Contractual Clauses management not built |

---

## 6. Schema Detail

**Multi-tenancy:** All tables include `tenant_id` + RLS enforced

| Table | Purpose |
|---|---|
| `compliance_tasks` | Task list per account per role — surfaced in Portal |
| `comms_consent` | Per-contact per-channel consent records |
| `email_suppressions` | Suppressed email addresses (bounce/complaint) |
| `gdpr_consent_records` | Full consent audit trail |
| `dsr_requests` | Data subject request records |
| `document_retention_classes` | Retention class definitions (5 seeded) |
| `content_restrictions` | Frame-level content restrictions per category |

---

## 7. EventBus Surface

### Outbound Events (Compliance → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `compliance.frame.restricted` | Frame restriction added | Sales (blocks future availability) |
| `compliance.frame.restriction_lifted` | Restriction removed | Sales (restores availability) |

### Inbound Events (Platform → Compliance)

| Event | Source | Effect |
|---|---|---|
| `sales.booking.confirmed` | Sales | Compliance verifies content category before CMS scheduling |
| `sitedevelopment.permit.approved` | Site Dev | Compliance updates frame compliance status |
| `dms.document.expiry_warning` | DMS | Compliance tasks updated |

---

## 8. External Integrations

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| Sales | Bidirectional | API call (sync) + EventBus | ✅ Live (stub Phase 1) |
| Comms Hub | Compliance → Comms Hub | `comms_consent` table read | ✅ Live |
| DMS | Bidirectional | EventBus | ✅ Live — retention + document expiry |
| Site Development | Site Dev → Compliance | EventBus | ✅ Designed — permit approval |
| Partners (`ContentRestrictionsService`) | Partners → Compliance | Service read | ✅ Live — per-agreement restrictions |

---

## 9. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| Frame eligibility API is a stub in Phase 1 | High | `compliance.stub.ts` returns all frames eligible — real compliance checks not active |
| No automated restriction zone scanning | High | Operators must manually flag frames — no automated geofencing vs schools/hospitals |
| Compliance admin UI in Comms Hub app is stubbed | Medium | `CompliancePage` in Comms Hub returns empty — no functional compliance dashboard |
| Consent audit export not built | Medium | Regulators may require bulk consent export — not currently possible via UI |
| DSR identity verification strength not configurable | Low | Verification method (email/phone/ID) should be configurable per jurisdiction |
| `content_restrictions` table not yet UI-managed | Low | Operators manage restrictions via DB — no UI for restriction CRUD |
| Cross-border data transfer controls missing | Low | SCCs / adequacy decisions — operator responsibility but platform offers no tooling |

---

*End of Compliance Module Audit v1.0*
