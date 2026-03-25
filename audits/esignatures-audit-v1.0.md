# E-Signatures Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Core Signing Flow Live — Provider TBD (Stub Active)  
**Source paths audited:** `supabase/functions/` (signature-*), `CLAUDE.md §§18.1 E-Signatures`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | E-Signatures |
| DB schema | `public` (signature_requests, signature_tokens, related tables) |
| Backend path | `supabase/functions/` (signing-related) |
| Frontend path | `src/portal/pages/SignaturesPage.tsx` (11.7KB) |
| Owner | Horizontal |
| Phase | Core flow live with stub provider; production provider TBD (DocuSign or PandaDoc) |

---

## 2. Current State Summary

The E-Signatures module manages the digital signing lifecycle for booking orders and other contractual documents. It is a blocking integration for Sales — a booking order cannot proceed to `signed` status without a valid signing completion webhook from the e-sig provider.

**Phase 1 stub:** The stub in Sales (`esignatures.stub.ts`) returns a mock `signing_url` and auto-fires the signed webhook after 5 seconds. This allows end-to-end development without a live provider.

**What is built:**
- `capture-signature` — captures OTP-verified signature for internal signing flows
- `generate-signed-pdf` — generates signed PDF after completion
- `send-signature-request` — initiates a signing request (external provider or internal OTP)
- `void-signature-request` — cancels a pending signing request
- `send-signature-otp` — sends OTP for internal signature verification
- `verify-signature-otp` — verifies OTP, completes signing
- `process-signature-reminders` — scheduled reminders for unsigned documents
- `process-expired-confirmations` — handles expired signing requests
- `validate-signature-token` — validates signing link tokens
- `SignaturesPage.tsx` in Portal — client-facing signing interface (11.7KB)
- `usePortalSignatures.ts` hook — signature request list + status

**Provider:** DocuSign or PandaDoc — named credential configured per tenant. Decision not yet made.

---

## 3. Signing Flow

### External Provider (DocuSign / PandaDoc)
1. Booking order generated in Sales
2. Sales calls `esignatures.createSigningRequest({ document_url, signatories, booking_order_id })`
3. Returns `{ external_signing_id, signing_url }`
4. `signing_url` sent to client via Comms Hub → client clicks link
5. Client signs in external provider UI
6. Provider fires webhook → `POST /api/sales/webhooks/esignature`
7. Webhook payload: `{ external_signing_id, signed_document_url, signed_at, signatory }`
8. Sales updates `booking_order.status = signed`, stores `signed_document_url`
9. `sales.booking_order.signed` emitted → DMS stores signed document

### Internal OTP Flow (lightweight alternative)
1. `send-signature-request` initiates internal OTP flow
2. `send-signature-otp` sends OTP to signatory's email/phone
3. Signatory enters OTP → `verify-signature-otp` validates
4. `capture-signature` captures drawn/typed signature
5. `generate-signed-pdf` creates signed PDF with audit trail
6. Signing record created → Sales webhook callback invoked

### Cancellation
- `void-signature-request` — cancels before signing; Sales resets `booking_order.status`
- `process-expired-confirmations` — auto-voids expired requests; Sales resets status

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `send-signature-request` | ✅ Built | Initiates signing request |
| `void-signature-request` | ✅ Built | Cancels pending request |
| `capture-signature` | ✅ Built | OTP-verified internal signature |
| `generate-signed-pdf` | ✅ Built | Signed PDF with audit trail |
| `send-signature-otp` | ✅ Built | OTP dispatch |
| `verify-signature-otp` | ✅ Built | OTP validation |
| `process-signature-reminders` | ✅ Built | Scheduled reminder dispatch |
| `process-expired-confirmations` | ✅ Built | Expiry handling |
| `validate-signature-token` | ✅ Built | Token validation |
| `SignaturesPage.tsx` in Portal | ✅ Built | Client-facing UI (11.7KB) |
| `usePortalSignatures.ts` hook | ✅ Built | Signature request list + status |
| `useSignatureProfile.ts` hook | ✅ Built | Signatory profile (5.3KB) |
| Sales webhook endpoint | ✅ Built | `POST /api/sales/webhooks/esignature` |
| External provider (DocuSign/PandaDoc) | 🟡 Stub | `esignatures.stub.ts` — mock URL + auto-fires webhook |
| Provider selection | ❌ Pending | DocuSign vs PandaDoc decision not made |
| Multi-signatory workflows | 🟡 Designed | Schema supports multiple signatories; flow for sequential signing not confirmed |
| Signatory delegation | ❌ Missing | What if the primary signatory is unavailable? |
| Signing ceremony audit log | ✅ Built | Via `generate-signed-pdf` audit trail |
| eIDAS qualified signature support | ❌ Missing | EU qualified electronic signatures — provider-dependent |

---

## 5. Schema Detail

**Multi-tenancy:** All tables include `tenant_id` + RLS enforced

| Table | Purpose |
|---|---|
| `signature_requests` | Master signing request record — `booking_order_id`, `external_signing_id`, `status`, `provider`, `signing_url`, `signed_at`, `signed_document_url` |
| `signature_tokens` | Magic link tokens for signing links — `type: 'sig'`, expiry |
| `signature_signatories` | Per-signatory records within a request — `contact_id`, `role`, `signed_at`, `otp_verified` |
| `signature_audit_log` | Immutable event log for each signing ceremony |

---

## 6. EventBus Surface

### Inbound Events / Webhook (External → E-Signatures)

| Event / Webhook | Source | Effect |
|---|---|---|
| `POST /api/sales/webhooks/esignature` | External provider | Updates `signature_requests.status`, fires `sales.booking_order.signed` |
| `sales.booking_order.generated` | Sales | Triggers signing request creation |

### Outbound (E-Signatures → Platform)
E-signatures does not emit platform EventBus events directly — it calls back to Sales via the webhook pattern, which then emits `sales.booking_order.signed` to DMS and Comms Hub.

---

## 7. External Integrations

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| DocuSign / PandaDoc | E-sig → Provider | REST API + webhook | 🟡 Stub (provider TBD) |
| Sales | Bidirectional | API call + webhook | ✅ Live (stub) |
| DMS | E-sig → DMS (via Sales) | EventBus | ✅ Designed — signed PDF stored |
| Comms Hub | E-sig → Comms Hub (via Sales) | EventBus | ✅ Designed — signed confirmation to client |
| Portal | E-sig → Portal | `SignaturesPage` + `usePortalSignatures` | ✅ Live |

---

## 8. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| Provider not selected (DocuSign vs PandaDoc) | High | Blocking for production go-live — named credential config needed per tenant |
| Stub auto-fires webhook after 5 seconds | Medium | Tests must account for 5s delay; production webhook timing is external provider-dependent |
| Multi-signatory sequential flow not confirmed | Medium | Schema supports multiple signatories but sequential signing order logic unclear |
| No signatory delegation workflow | Medium | If FM is unavailable, there is no flow to reassign signing authority |
| eIDAS qualified signature not supported | Low | Required for some EU jurisdictions — provider-dependent capability |
| Reminder schedule not tenant-configurable | Low | Should live in `tenant_business_rules` |
| OTP expiry duration hardcoded | Low | Should be configurable per tenant |
| `process-expired-confirmations` does not notify client | Low | Client receives no communication when signing link expires |

---

*End of E-Signatures Module Audit v1.0*
