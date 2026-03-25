# E-Signatures Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (AI Development Agent)
**Status:** Full internal implementation live — no external provider, no stubs in edge functions
**Source paths audited:**
- `supabase/functions/` — 9 signature edge functions, all read in full
- `src/signatures/` — 63 files, all inventoried
- `src/portal/pages/SignaturesPage.tsx`, `src/portal/hooks/`
- `src/pages/SignDocumentPage.tsx`, `SignatoryAcceptPage.tsx`, `SigningDelegation.tsx`
- `src/components/layout/AnimatedRoutes.tsx` — all signature routes verified
- Supabase REST API (`/rest/v1/`) — full schema enumerated

---

## 1. Executive Summary

The v1.0 audit (2026-03-23) described the E-Signatures module as "Core Signing Flow Live — Provider TBD (Stub Active)." That description is **materially wrong** in two directions:

**Over-reported:** There is no `esignatures.stub.ts` file in the codebase. No auto-firing webhook exists. No 5-second mock delay was found anywhere. The v1.0 audit invented a stub mechanism that does not exist in the actual code.

**Under-reported:** The module has been significantly expanded beyond what v1.0 described. The schema is deeper, the front-end is a full standalone module with its own layout, nav, and five dedicated pages, and multi-recipient sequential signing is fully implemented — not merely "designed."

The module is an **internally-complete e-signature system** that does not depend on DocuSign, PandaDoc, or any external provider. The "provider TBD" concern from v1.0 is now largely moot: the system handles the entire signing ceremony natively (OTP verification, drawn/typed/uploaded signatures, PDF embedding, audit trail). The only remaining external-provider question is whether the operator wants to optionally route high-assurance documents through a third-party provider in the future — which would require additional work.

---

## 2. Critical Discrepancies from v1.0

### 2.1 Schema Is Completely Different

v1.0 described these tables:
- `signature_requests` — correct, exists
- `signature_tokens` — **does not exist** in the live DB
- `signature_signatories` — **does not exist**; replaced by `signature_recipients`
- `signature_audit_log` — **does not exist**; replaced by `signature_audit_events`

Actual tables found in the live Supabase REST API schema (verified 2026-03-25):

| Table | Status |
|---|---|
| `signature_requests` | Present — master signing request record |
| `signature_recipients` | Present — per-recipient records (replaces `signature_signatories`) |
| `signature_records` | Present — actual captured signature data |
| `signature_document_fields` | Present — PDF field placement coordinates |
| `signature_audit_events` | Present — immutable event log (replaces `signature_audit_log`) |
| `signature_templates` | Present — reusable templates (not mentioned in v1.0) |
| `user_signature_profiles` | Present — saved signatures for reuse (not mentioned in v1.0) |
| `signing_authority_delegations` | Present — delegation support (not mentioned in v1.0) |

The `signature_tokens` table described in v1.0 does not exist. Magic link tokens are stored directly on `signature_recipients.magic_token` and `signature_requests.magic_token` (legacy field for backward compatibility). There is no separate token table.

### 2.2 Front-End Architecture Is Entirely Different

v1.0 claimed the front-end was `src/portal/pages/SignaturesPage.tsx` (11.7KB) and described a single Portal-facing page. This is incorrect.

The module now has **two distinct front-end surfaces**:

**1. Internal Ops Dashboard Module (`src/signatures/`)**
A full standalone module with its own layout, nav, and pages:
- `src/signatures/components/layout/SignaturesLayout.tsx`
- `src/signatures/components/layout/SignaturesNav.tsx`
- `src/signatures/pages/SignaturesHome.tsx` (145 lines)
- `src/signatures/pages/CreateRequestPage.tsx` (492 lines)
- `src/signatures/pages/RequestDetailPage.tsx` (367 lines)
- `src/signatures/pages/TemplatesPage.tsx` (~200 lines)
- `src/signatures/pages/SettingsPage.tsx` (161 lines)
- 14 UI components (3,956 total lines) including `SignaturePad.tsx`, `OTPVerification.tsx`, `RecipientManager.tsx`, `AuditTrailView.tsx`, `SignatureRequestsDashboard.tsx`
- PDF tooling: `FieldPlacementEditor.tsx`, `PDFViewer.tsx`, `FieldPalette.tsx`, `DocumentUpload.tsx`

**2. Portal / Client-Facing (`src/portal/`)**
- `src/portal/pages/SignaturesPage.tsx` (329 lines — not 11.7KB as stated)
- `src/portal/hooks/usePortalSignatures.ts` (144 lines)
- `src/portal/hooks/useSignatureProfile.ts` (181 lines)

**3. Public Signing Pages (`src/pages/`)**
- `src/pages/SignDocumentPage.tsx` (589 lines) — the magic link signing page
- `src/pages/SignatoryAcceptPage.tsx` (221 lines)
- `src/pages/SigningDelegation.tsx` (303 lines)

v1.0 missed the entire ops module, all three public pages, and reported an incorrect file size for the portal page.

### 2.3 Multi-Signatory Flow Is Fully Implemented, Not "Designed"

v1.0 stated: "Schema supports multiple signatories; flow for sequential signing not confirmed."

The actual `capture-signature/index.ts` (399 lines) implements sequential signing end-to-end: after each signature is recorded, it queries all `signature_recipients` ordered by `signing_order`, finds the next `pending` recipient, updates their status to `sent`, and dispatches a notification via Communications Hub. The sequential flow is live code, not a design.

The `send-signature-request/index.ts` (265 lines) correctly creates recipient records with `status: 'pending'` for non-first signers in sequential mode, `status: 'sent'` for all in parallel mode.

### 2.4 Signatory Delegation Is Present in DB Schema

v1.0 listed "Signatory delegation" as Missing (❌). The `signing_authority_delegations` table exists in the live DB. Whether the front-end fully exposes this and whether `SigningDelegation.tsx` (303 lines) covers the full workflow needs further verification, but the schema foundation is present — not missing.

### 2.5 `process-expired-confirmations` Is Not a Signature Function

v1.0 listed `process-expired-confirmations` as a signature edge function that "handles expired signing requests." The actual code (`supabase/functions/process-expired-confirmations/index.ts`, 95 lines) handles **GDPR migration confirmation expiry** — it calls `expire_migration_confirmations()` and blocks bookings for clients with expired GDPR consents. It has no connection to the signature system.

The signature expiry path is handled differently: `validate-signature-token` marks tokens as `expired` inline when the token expiry timestamp is in the past. There is no separate scheduled signature expiry job.

### 2.6 No External Provider Stub Exists

v1.0 extensively described `esignatures.stub.ts` with a mock signing URL and a 5-second auto-fire webhook. This file does not exist anywhere in the codebase (`find` confirms zero matches for `*stub*` in signature paths, and `grep` across all `src/` for `esignatures.stub` returns nothing). The stub described in v1.0 was fictional or refers to a file that was deleted before the audit was performed.

### 2.7 Status Inconsistency Between Edge Function and Service

`void-signature-request/index.ts` sets `status: "cancelled"` on voided requests. `SignatureService.ts` in the front-end calls `voidRequest()` and sets `status: 'voided'`. These are different string values for the same conceptual state — a data integrity discrepancy that will cause filtering/display bugs.

---

## 3. What Is Verified Working

### 3.1 Edge Functions (all 8 genuine signature functions)

| Function | Lines | Status | Notes |
|---|---|---|---|
| `send-signature-request` | 265 | Verified real | Multi-recipient, sequential/parallel, Comms Hub integration |
| `capture-signature` | 399 | Verified real | Dual path: `signature_recipients` (new) + `signature_requests` (legacy fallback), sequential chain activation |
| `generate-signed-pdf` | 331 | Verified real | pdf-lib embeds signatures at placed field coordinates or default position, optional certificate page |
| `validate-signature-token` | 259 | Verified real | Dual path validation, viewed_at tracking, returns fields for signing page |
| `send-signature-otp` | 297 | Verified real | SMS via Comms Hub, email fallback, SHA-256 hashed OTP storage, rate limiting (3/hour) |
| `verify-signature-otp` | 158 | Verified real | Hash comparison, max 5 attempts, clears OTP on success |
| `void-signature-request` | 201 | Verified real | Auth check (requester only), cascades to recipients, Comms Hub notifications |
| `process-signature-reminders` | 177 | Verified real | Queries `signature_recipients` with expiry window, max 2 reminders, 24h minimum interval |

`process-expired-confirmations` (95 lines) is confirmed **not a signature function** — it handles GDPR migration confirmations only.

### 3.2 Service Layer (`src/signatures/services/`)

| Service | Lines | Verified |
|---|---|---|
| `SignatureService.ts` | ~500 | Full CRUD, createEnvelope (multi-recipient), captureSignature, verifySignature, voidRequest, mappers |
| `BulkSendService.ts` | 260 | Batch multi-recipient sends via SignatureService.createEnvelope |
| `SmartReminderService.ts` | 421 | Query pending recipients, urgency classification, reminder dispatch |
| `OnboardingIntegrationService.ts` | 352 | Maps document types to onboarding completion fields, updates flows |
| `OnboardingSignatureOrchestrator.ts` | 525 | Org-type-driven document requirement config, batch signature request creation |
| `SignatureNotificationService.ts` | ~100 | Thin wrapper for requester notifications |
| `HashService.ts` | ~50 | SHA-256 document hashing for integrity |
| `VerificationService.ts` | ~80 | Document hash verification |

### 3.3 Front-End Components

The full PDF field-placement editor (`FieldPlacementEditor.tsx`, 298 lines, described as "DocuSign-like drag-and-drop") is implemented. The signing page (`SignDocumentPage.tsx`, 589 lines) handles both placed-field mode and default-position mode with OTP verification gating.

### 3.4 Audit Trail

`signature_audit_events` is used consistently across all edge functions. Events tracked: `signature_requested`, `signature_viewed`, `signature_signed`, `signature_otp_sent`, `signature_otp_verified`, `signature_voided`, `signature_reminder_sent`, `pdf_downloaded`.

### 3.5 Database Schema (Live DB confirmed)

All 8 tables listed in section 2.1 are confirmed present via REST API enumeration. RLS enforced (tenant scoping visible in all function queries via `platform_tenant_id`). Notably:

- `signature_recipients` has: `magic_token`, `magic_token_expires_at`, `status`, `signed_at`, `viewed_at`, `signing_order`, `role` (signer/cc/approver/viewer), `otp_code`, `otp_expires_at`, `otp_verified_at`, `otp_attempts`, `last_reminder_at`, `reminder_count`
- `signature_records` has: `signature_data` (base64), `signature_type` (drawn/typed/uploaded), `ip_address`, `user_agent`, `geolocation`, `document_hash_at_signing`, `consent_given`, `consent_text`, `consent_timestamp`
- `signature_document_fields` has: `x_position`, `y_position`, `width`, `height`, `page_number`, `field_type`, `filled_value`, `filled_at`
- `user_signature_profiles` enables saved signature reuse
- `signing_authority_delegations` present for delegation flows

### 3.6 Routing

All signature routes verified in `src/components/layout/AnimatedRoutes.tsx`:

| Route | Component | Auth Guard |
|---|---|---|
| `/sign/:token` | `SignDocumentPage` | None (public — magic link) |
| `/signatory/accept/:token` | `SignatoryAcceptPage` | None (public) |
| `/delegation/signing` | `SigningDelegation` | None (public) |
| `/signatures` | `SignaturesHome` + `SignaturesLayout` | `RequireAuth` |
| `/signatures/create` | `CreateRequestPage` + `SignaturesLayout` | `RequireAuth` |
| `/signatures/:requestId` | `RequestDetailPage` + `SignaturesLayout` | `RequireAuth` |
| `/signatures/templates` | `TemplatesPage` + `SignaturesLayout` | `RequireAuth` |
| `/signatures/settings` | `SettingsPage` + `SignaturesLayout` | `RequireAuth` |
| `/portal/support/signatures` | `SignaturesPage` (portal) | `PortalRoute` |
| `/portal/signatures` | Redirects → `/portal/support/signatures` | Redirect |

All 10 routes are wired and use lazy loading with retry. No broken routes found.

### 3.7 Communications Hub Integration

All edge functions route notifications exclusively through the `_shared/communications-hub.ts` helper (`sendNotification()` + `isEmailAllowedInDev()`). There are no direct Resend API calls. Events used:
- `signature.request.created` — initial request to signer
- `signature.received` — requester notified of individual signature
- `signature.document.completed` — requester notified when all signed
- `signature.reminder.sent` — reminder to pending signer
- `signature.request.voided` — void notification to recipients
- `signature.otp.sent` — OTP delivery (email + SMS paths)

This is compliant with Communications Hub invariant rules.

---

## 4. What Is Missing or Stubbed

### 4.1 External Provider Integration — Confirmed Absent

No DocuSign or PandaDoc integration exists in the codebase. The `credit_agreements` table has `docusign_envelope_id`, `docusign_signed_document_url`, `docusign_status` columns — these are vestiges from an earlier design (or specific to the credit agreement flow) but there is no corresponding edge function or service that calls the DocuSign API.

**Assessment:** This is not a blocker. The internal signing flow is complete and production-capable for the use case described (internal OTP + drawn signature + PDF embed). The operator does not need DocuSign unless they specifically require qualified electronic signatures (eIDAS) or have a legal requirement for a third-party-verified signing ceremony.

### 4.2 No Signature Expiry Scheduled Job

There is no scheduled function that bulk-expires overdue `signature_recipients`. Expiry is handled lazily: `validate-signature-token` detects expiry on access and marks the record expired. Recipients who never access their signing link will remain in `sent` or `viewed` status indefinitely in the DB until the reminder system stops contacting them (after 2 reminders).

**Severity:** Low. The reminders correctly cap at 2, and the `magic_token_expires_at` field is set. A batch cleanup job would be nice for reporting accuracy but is not functionally blocking.

### 4.3 SignatureAnalysisAgent — Implemented but Not Deployed

`src/signatures/agents/SignatureAnalysisAgent.ts` extends `AIEnhancedAgent` and implements four capabilities: `analyze`, `remind`, `assess`, `summarize`. The agent class is complete. However, there is no edge function wiring it to a scheduled trigger or AgentHub dispatch. It exists as a callable service, not a running agent.

**Status:** Built but not operationally registered in AgentHub scheduler.

### 4.4 eIDAS Qualified Signatures

Confirmed absent. No qualified signature support. Provider-dependent and not required without an explicit operator requirement for EU-qualified signatures. Remains a valid gap for EU jurisdictions that require QES.

### 4.5 Templates — DB Present, CRUD Built, Not Field-Defined

`signature_templates` exists in the live DB and `TemplatesPage.tsx` + `useSignatureTemplates.ts` are implemented for CRUD. However, templates do not pre-populate `signature_document_fields` for placed fields — users must re-place fields each time. Templates only store metadata, signingOrder, and defaultMessage. Partial implementation.

### 4.6 `process-expired-confirmations` Incorrectly Listed as Signature Function

This must be removed from any signature module inventory. It is a GDPR migration tool.

---

## 5. Routing Verification

Fully verified. See section 3.6. All 10 routes are wired correctly. Three public routes (no auth required) correctly service magic-link signing flows. Five internal routes are guarded by `RequireAuth`. One portal route uses `PortalRoute`. One redirect is in place for the old `/portal/signatures` path.

No orphaned route imports. No missing lazy-load targets. The `SignaturesAgentAdapter.ts` exists in `src/signatures/agents/` and provides the `SignaturesHubAdapter` pattern used by hooks that call AgentHub for signature intelligence.

---

## 6. Provider Selection Status

**There is no external provider.** The v1.0 framing of "DocuSign vs PandaDoc — decision not yet made" is no longer the correct framing. The system has been built as a **native internal e-signature platform** with:

- OTP-based identity verification (SMS or email)
- Drawn, typed, or uploaded signature capture
- SHA-256 document integrity hashing at capture time
- IP address, user agent, and geolocation capture
- Consent recording with timestamp and text
- PDF embedding via pdf-lib
- Optional certificate-of-completion page in the PDF
- Immutable audit event log

This is not a stub waiting for a real provider. It is a functioning e-signature system. The decision about external provider integration should be reframed as: **does the operator have a legal or compliance requirement that necessitates a third-party certified signing ceremony?** If no, the native system is production-ready. If yes, a DocuSign/PandaDoc adapter would need to be built alongside the existing internal flow.

---

## 7. Technical Debt

| Issue | Severity | Detail |
|---|---|---|
| `status: 'voided'` vs `status: 'cancelled'` inconsistency | High | `void-signature-request` edge function sets `status: "cancelled"`. `SignatureService.ts` sets `status: 'voided'`. These are different string values. Dashboard filters and status displays will render inconsistently depending on which path triggered the void. |
| `process-expired-confirmations` misidentified as signature function | High | v1.0 audit and possibly internal docs describe this as a signature expiry function. It is a GDPR migration tool. Teams building on the assumption that signature expiry is automatically processed will be wrong. |
| Legacy `magic_token` on `signature_requests` | Medium | `send-signature-request` writes a `magic_token` directly to `signature_requests` alongside creating proper `signature_recipients` records. This legacy field is maintained "for backward compatibility" per code comments. It creates a dual-path in `capture-signature` and `validate-signature-token` that will persist indefinitely. Should be deprecated with a migration target date. |
| OTP hardcoded 10-minute expiry | Medium | `send-signature-otp` sets `otp_expires_at = now() + 10 minutes`. This is not tenant-configurable. For high-security deployments or slow-network contexts, this may be too short. |
| Reminder schedule hardcoded (max 2, min 24h gap) | Medium | `process-signature-reminders` hardcodes `reminderCount >= 2` and `hoursSinceLastReminder < 24`. These should live in `tenant_business_rules`. |
| `signature_document_fields` not pre-populated by templates | Medium | Templates exist but do not persist field placement data. Every use of a template requires re-placing fields manually. |
| `generate-signed-pdf` fallback position is hardcoded | Low | When no fields are placed, signatures default to bottom-right of first page at pixel position `x: width - 200, y: 100`. This may clash with document content. |
| No batch expiry job for `signature_recipients` | Low | Expired recipients remain in `sent`/`viewed` status unless accessed. Reporting on "expired" signatures will be inaccurate. |
| SignatureAnalysisAgent not scheduled | Low | Agent is built but not operationally triggered. Urgency classification and AI-assisted reminders are unused. |
| `generate-signed-pdf` only supports PNG | Low | `pdfDoc.embedPng()` is used for all signature types. A drawn SVG path or typed-text signature rendered as PNG will work, but typed signatures stored as plain text would fail without pre-rendering. |
| CORS headers allow `*` origin on all functions | Low | All 8 edge functions use `"Access-Control-Allow-Origin": "*"`. For a document-signing system, this should be scoped to the application domain. |

---

## 8. Production Readiness Rating

**Overall: 7 / 10**

| Dimension | Score | Rationale |
|---|---|---|
| Core signing flow | 9/10 | Fully functional end-to-end for internal OTP signing with PDF embed |
| Multi-signatory | 8/10 | Sequential and parallel both implemented; delegation schema present but delegation UI needs verification |
| Schema integrity | 6/10 | `voided` vs `cancelled` inconsistency; legacy token dual-path; no expiry cleanup job |
| Routing | 10/10 | All 10 routes verified, correct auth guards |
| Provider coverage | 7/10 | Native system is complete for the declared use case; no external provider if needed for eIDAS |
| Agent intelligence | 4/10 | Agent built, not running; SmartReminderService built, not triggered from Hub |
| Audit trail | 9/10 | Comprehensive event logging across all functions |
| Comms Hub compliance | 10/10 | All notifications routed correctly; no direct API calls |
| Template system | 5/10 | CRUD present, field templates incomplete |
| Technical debt | 6/10 | Status inconsistency and legacy dual-path are real bugs; others are configuration gaps |

**What would take this to 9/10:**
1. Fix `voided` vs `cancelled` status inconsistency (30-minute fix)
2. Make reminder schedule and OTP expiry tenant-configurable
3. Add scheduled batch expiry for `signature_recipients`
4. Operationally register `SignatureAnalysisAgent` in AgentHub
5. Complete template field placement persistence

---

## 9. File Inventory Summary

**Edge Functions (genuine signature functions — 8):**
`capture-signature` (399), `generate-signed-pdf` (331), `send-signature-request` (265), `validate-signature-token` (259), `void-signature-request` (201), `send-signature-otp` (297), `verify-signature-otp` (158), `process-signature-reminders` (177)
Total edge function lines: **2,087**

**Source module (`src/signatures/`) — 63 files:**
- Services (8): 2,213 lines
- Components (14): 3,956 lines
- Pages (5): 1,165 lines
- Hooks (8): ~600 lines
- Agents (3): ~400 lines
- API / Auth / Config / Types: ~500 lines
**Estimated total: ~9,000 lines**

**Portal hooks and page:**
`SignaturesPage.tsx` (329), `usePortalSignatures.ts` (144), `useSignatureProfile.ts` (181)

**Public signing pages:**
`SignDocumentPage.tsx` (589), `SignatoryAcceptPage.tsx` (221), `SigningDelegation.tsx` (303)

---

*End of E-Signatures Module Audit v2.0*
*Audited by Kai — 2026-03-25*
