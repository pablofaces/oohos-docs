# DMS Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Core Classification Engine Live — AI Intelligence Layer Built  
**Source paths audited:** `supabase/functions/dms-*/`, `supabase/functions/download-supplier-document/`, `supabase/functions/upload-supplier-document/`, `supabase/functions/enforce-retention-policies/`, `CLAUDE.md §§18.1 DMS`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | DMS (Document Management System) |
| DB schema | `public` (`dms_documents`, `document_approval_workflow`, related tables) |
| Frontend path | No dedicated app — UI via Portal + Admin views |
| Backend path | `supabase/functions/dms-*/` + shared upload/download functions |
| Owner | Horizontal |
| Phase | Core live — Intelligence Layer (AI classification) built |

---

## 2. Current State Summary

The DMS is the document repository and intelligence layer for OOH OS. Every module-generated PDF (proposals, booking orders, invoices, proof of play reports) and every inbound document (POs, agency documents) flows through the DMS. It stores document references (URLs), not binary files — actual storage is in Supabase Storage.

**What is built:**
- Document upload and download (`upload-supplier-document`, `download-supplier-document`)
- AI classification engine (`dms-intelligence-processor`) — classifies inbound documents by type, extracts PO numbers via OCR + regex
- Dataset export for retraining (`dms-export-classification-dataset`)
- Retention policy enforcement (`enforce-retention-policies`) — scheduled, tenant-configurable
- Document approval workflow (`document_approval_workflow` table — canonical name, not `document_approvals`)
- 5 retention classes seeded
- Integration with Sales (PO classification → `dms.purchase_order.classified` event)
- Integration with DMS-backed Portal Financials page (document type display with tenant terminology)

**Key architectural note:** The table is named `document_approval_workflow` — **not** `document_approvals`. All code and migrations use this exact name.

---

## 3. Document Types

| Document type key | Display label (tenant terminology) | Direction | Linked entity | Stored by |
|---|---|---|---|---|
| `proposal` | Proposal | Outbound | `proposal_id` | Sales on send |
| `booking_order` | Booking Order | Outbound | `booking_order_id` | Sales on generate |
| `purchase_order` | PO / IO (tenant term via `useTenantTerminology()`) | Inbound | `booking_id` | Rep upload or Portal FM upload |
| `order_confirmation` | Order Confirmation | Outbound | `order_confirmation_id` + `booking_id` | Sales on issue |
| `invoice` | Invoice | Outbound | `invoice_id` | Sales on generate |
| `credit_note` | Credit Note | Outbound | `invoice_id` | Sales on generate |
| `proof_of_play` | Proof of Play Report | Outbound | `booking_id` | Sales on campaign complete |
| `delivery_report` | Campaign Report | Outbound | `booking_id` | Sales on campaign complete |

**Terminology rule:** The document type label for `purchase_order` must render via `t('purchase_order_label')` from `useTenantTerminology()` — never hardcoded as "Purchase Order" or "IO". This is enforced on the Portal Financials page.

---

## 4. AI Classification (IngestionAgent)

`dms-intelligence-processor` Edge Function processes inbound documents:

1. Document uploaded → type unknown
2. OCR + NLP → document type classified (8 types)
3. For `purchase_order` type: regex extracts PO number from document text
4. Compare extracted PO number vs `booking.po_number` (if already set)
5. If match → `dms.purchase_order.classified` event with `matches_booking: true`
6. If mismatch or no `booking.po_number` → event with `matches_booking: false` → flagged for rep review
7. Sales module receives event → updates `booking.po_document_url` with DMS URL

`dms-export-classification-dataset` — exports classified documents for model retraining. Enables continuous improvement of the classification model.

---

## 5. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| Document upload | ✅ Built | `upload-supplier-document` Edge Function |
| Document download | ✅ Built | `download-supplier-document` Edge Function |
| AI classification engine | ✅ Built | `dms-intelligence-processor` |
| Classification dataset export | ✅ Built | `dms-export-classification-dataset` |
| Retention policy enforcement | ✅ Built | `enforce-retention-policies` (scheduled) |
| Document approval workflow table | ✅ Built | `document_approval_workflow` (canonical name) |
| 5 retention classes | ✅ Seeded | Via migration |
| PO number extraction + matching | ✅ Built | OCR + regex → `dms.purchase_order.classified` event |
| Signed PDF generation | ✅ Built | `generate-signed-pdf` Edge Function |
| Document expiry checking | ✅ Built | `check-document-expiration` Edge Function |
| Dedicated DMS admin UI | ❌ Missing | Document management UI is embedded in Portal + Admin views, not a standalone app |
| Bulk document re-classification | ❌ Missing | Ad-hoc; no batch re-run UI |
| Document version history UI | ❌ Missing | Schema may support versions; no UI |
| Cross-module document search | ❌ Missing | No global document search across all document types |

---

## 6. Schema Detail

**Multi-tenancy:** All tables include `tenant_id` + RLS enforced

| Table | Purpose |
|---|---|
| `dms_documents` | Master document registry — `id, tenant_id, document_type, entity_type, entity_id, file_path, file_name, sha256_hash, created_at, expires_at, retention_class_id` |
| `document_approval_workflow` | Approval workflow state — `document_id, tenant_id, status, approver_id, approved_at, rejection_reason` |
| `document_retention_classes` | 5 seeded retention classes with TTL and GDPR basis |
| `document_versions` | Version history per document (planned) |

**Storage:** Supabase Storage — DMS stores URL references only, not binary files. Files are tenant-isolated by folder prefix.

---

## 7. EventBus Surface

### Outbound Events (DMS → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `dms.document.classified` | Inbound document type identified | Sales (links to booking), Ever-boarding |
| `dms.purchase_order.classified` | PO number extracted + matched | Sales (updates `booking.po_document_url`) |
| `dms.document.expiry_warning` | Document approaching expiry | Sales (e.g. rate card expiring), Compliance |

### Inbound Events (Platform → DMS)

| Event | Source | DMS Action |
|---|---|---|
| `sales.proposal.sent` | Sales | Store proposal PDF reference |
| `sales.booking_order.generated` | Sales | Store booking order reference |
| `sales.booking_order.signed` | Sales | Store signed document reference |
| `sales.order_confirmation.issued` | Sales | Store order confirmation reference |
| `sales.rate_card.updated` | Sales | Store rate card PDF reference |
| `sales.delivery.report_ready` | Sales | Store proof of play / campaign report |

---

## 8. External Integrations

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| Supabase Storage | Read/Write | Direct SDK | ✅ Live — all file storage |
| Sales | Bidirectional | EventBus | ✅ Live |
| Compliance | DMS → Compliance | EventBus (`dms.document.expiry_warning`) | ✅ Designed |
| Ever-boarding | DMS ← Ever-boarding | EventBus | ✅ Live — document uploads for FM steps |
| E-signatures | DMS ← E-sig | Signed PDF stored post-signing | ✅ Live |
| AgentHub (IngestionAgent) | DMS → AgentHub | Via `dms-intelligence-processor` | ✅ Live |

---

## 9. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| Table name `document_approval_workflow` not `document_approvals` | Medium | Confusing name — all code must use exact canonical name. Known footgun. |
| No dedicated DMS admin UI | Medium | Operators manage documents via Portal + Admin views only — no standalone DMS dashboard |
| PO number extraction relies on OCR quality | Medium | Poor scan quality will produce false `matches_booking: false` flags — rep review burden |
| `generate-signed-pdf` tightly coupled to E-signatures module | Low | Changing e-sig provider may require refactoring this function |
| No bulk re-classification | Low | If classification model improves, no way to re-classify existing documents in bulk |
| Retention policy scheduling frequency not tenant-configurable | Low | Should live in `tenant_business_rules` |
| Document version history not fully implemented | Low | Schema supports it but no version-aware UI |
| `check-document-expiration` expiry warning threshold not tenant-configurable | Low | Hardcoded warning window — should be configurable per document type |

---

*End of DMS Module Audit v1.0*
