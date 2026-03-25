# DMS Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (AI)
**Based on:** v1.0 audit (2026-03-23) + live codebase scan + production DB introspection
**Source paths audited:**
- `/workspace/group/everboarding/src/dms/` (full tree)
- `/workspace/group/everboarding/supabase/functions/` (DMS-related)
- `/workspace/group/everboarding/supabase/migrations/` (DMS-related)
- Production REST API (`gwtzedkvcnnnlbeidzar.supabase.co`)

---

## 1. Executive Summary

Since v1.0 (2026-03-23), the DMS has grown substantially. The intelligence layer (v1.5) has been built in `ooh_dms` schema. A full schema split migration (`20260419000000_extract_dms_to_dms_schema.sql`) has been authored but **not yet applied to production** — all document tables currently live in `public`. A new standalone DMS module exists at `src/dms/` with its own hexagonal architecture (ports/adapters/repositories) running alongside the legacy `src/dms/services/` layer, which is now marked `@deprecated`. There is one confirmed live naming bug: `DocumentRepository` (the new standalone module) queries `document_approvals` — a table that does not exist. The canonical name is `document_approval_workflow`.

**Production readiness: 5.5 / 10**

---

## 2. Schema State — Three-Schema Reality

The v1.0 audit described a two-schema world (`public` + `ooh_dms`). There is now a three-schema target state:

| Schema | Purpose | Current State |
|---|---|---|
| `public` | All core document tables — `documents`, `document_versions`, `document_approval_workflow`, `document_retention_policies`, 30+ others | **Live in production** — verified via REST API |
| `ooh_dms` | Intelligence / AI pipeline tables — `document_classifications`, `classification_corrections`, `document_anomalies`, `approval_outcomes`, `document_prescores`, `entity_completeness_predictions`, `memory_events`, `rejection_reason_codes` | **Live in production** — created by `20260318130000_dms_v1_5_intelligence_layer.sql` |
| `dms` | Target schema for all core tables (migration planned) | **Not yet created in production** — migration authored but unapplied |

### Schema Migration Status

`20260419000000_extract_dms_to_dms_schema.sql` is a complete, well-structured migration that:
1. Creates `dms` schema
2. Moves 30 tables from `public` to `dms` via `ALTER TABLE SET SCHEMA`
3. Creates backward-compatibility `SELECT` views in `public`
4. Grants appropriate permissions

It has been committed to `supabase/migrations/` with timestamp `20260419` (future-dated: April 19, 2026). It is **not applied**. This migration is blocked because:
- Backward-compat views only cover `SELECT` — INSERT/UPDATE/DELETE through the public views will fail unless they are replaced with updatable views or the calling code is migrated simultaneously
- PostgREST requires `search_path` update (commented out in the migration: `ALTER DATABASE postgres SET search_path TO public, dms, ooh_dms, ooh_everboarding`) — this needs to be uncommented before applying
- The migration moves `document_approval_workflow` to `dms.document_approval_workflow`, which resolves the naming bug in new code but exposes the naming bug in `DocumentRepository` more sharply

A second, earlier draft exists at `src/dms/schema/migration_extract_dms.sql` — this is a **planning artifact** (all ALTER statements are commented out). It references `document_approvals` (wrong name) and `document_access` (not in the final migration). It should be deleted or clearly marked as superseded to avoid confusion.

---

## 3. Module Architecture — New Standalone DMS

v1.0 described the DMS as "no dedicated app — UI via Portal + Admin views" with backend in `supabase/functions/dms-*/`. Since then, a full standalone TypeScript module has been built at `src/dms/`:

```
src/dms/
├── agents/                    # IngestionAgent, ComplianceAgent, ExpiryAgent
│   └── prompts/              # System + task prompts per agent
├── schema/
│   └── migration_extract_dms.sql   # Planning draft (commented-out)
├── services/                  # @deprecated legacy services (18 files)
└── src/                       # New hexagonal module
    ├── adapters/DMSAdapter.ts
    ├── lib/index.ts
    ├── ports/IDMSPort.ts
    └── repositories/DocumentRepository.ts
    └── services/DocumentService.ts
```

This dual-layer structure is intentional: legacy `services/` files are individually marked `@deprecated Sprint 3 — migrate callers to src/dms`. Migration is in progress.

---

## 4. AI Classification — Verified Working (with Caveats)

**Status: Live and operational, but scope differs from v1.0 description.**

### What v1.0 said
- `dms-intelligence-processor` classifies inbound OOH documents (proposals, POs, invoices, etc.)
- OCR + NLP → 8 document types
- PO number extraction via regex → `dms.purchase_order.classified` event

### What the code actually does
The `dms-intelligence-processor` Edge Function supports 4 actions:
- `classify` — zero-shot LLM classification via `google/gemini-3-flash-preview` (Lovable AI gateway)
- `anomaly_check` — LLM-based structure/content verification (AN1)
- `pre_score` — pre-scoring with historical approval context (L1)
- `readiness` — entity readiness scoring against next gate (L2)

**Key discrepancy:** The classifier is built specifically for **Maltese bus shelter planning applications**, not for generic OOH operator documents. The system prompt reads: `"You are a document classifier for Maltese bus shelter planning applications."` The `typeLabels` parameter is caller-supplied, so the classification framework is flexible, but the LLM prompt context is hardcoded to this specific use case. If used for general OOH document types (proposals, invoices, POs), the prompt context is wrong and will degrade accuracy.

The `LOVABLE_API_KEY` dependency is a single point of failure. No fallback or circuit-breaker exists.

**OCR + PO number extraction** described in v1.0 is **not present** in the current Edge Function. The function receives `contentText` (pre-extracted) as input — the OCR step is assumed to happen upstream (not audited as part of DMS).

The new `IngestionAgent` (`src/dms/agents/IngestionAgent.ts`) uses `AIEnhancedAgent` infrastructure and delegates to `aiAnalyze()` for document validation on upload. This is a separate AI pathway from `dms-intelligence-processor`. The two AI pathways are not formally connected.

`dms-export-classification-dataset` — present as an Edge Function, consistent with v1.0.

---

## 5. `document_approvals` Naming Bug — Confirmed, Impact Assessed

**Status: Active bug in production-bound code.**

### The split

| Layer | Table name used | Correct? |
|---|---|---|
| `DocumentApprovalService` (legacy, `src/dms/services/`) | `document_approval_workflow` | Yes |
| `DocumentRepository` (new standalone module, `src/dms/src/repositories/`) | `document_approvals` | **No — table does not exist** |
| `src/dms/schema/migration_extract_dms.sql` (planning draft) | `document_approvals` | **No — carried over wrong name** |

### Affected code in `DocumentRepository`

Three methods query `document_approvals`:
- `findApproval()` — line 143
- `insertApproval()` — line 155
- `updateApproval()` — line 166

All three will return a PostgREST 404/error at runtime since `document_approvals` does not exist in any schema. The new `DMSAdapter` delegates to `DocumentService` which uses `DocumentRepository` — meaning approval operations through the new module are broken end-to-end.

### v1.0 claim status
v1.0 stated "all code and migrations use `document_approval_workflow` (canonical name)." This was accurate at audit time for the services layer but is now false for the new `src/dms/src/` module. The bug was introduced during the Sprint 3 standalone module build.

### Fix required
```typescript
// In /workspace/group/everboarding/src/dms/src/repositories/DocumentRepository.ts
// Lines 143, 155, 166: change .from('document_approvals') to .from('document_approval_workflow')
```

---

## 6. Retention Enforcement — Scope Mismatch

**Status: Retention function is live but enforces client GDPR retention, not document TTL.**

The `enforce-retention-policies` Edge Function calls `check_retention_compliance()` RPC which returns clients eligible for data deletion. The enforcement actions are:
- Nulling marketing fields (website, phone) on `clients` table
- Anonymising `clients` records after 7-year financial retention

This is **GDPR client data retention** — not document-level retention based on `document_retention_policies` or `document_retention_classes`. The DMS-level document TTL enforcement (archiving/deleting documents based on their retention class) is **not implemented** in this function.

v1.0 marked retention enforcement as "✅ Built" — this is partially correct (GDPR client retention is enforced) but the document-level retention described in the schema (`document_retention_policies` table, `retention_class_id` on `dms_documents`) has no corresponding enforcement logic in any Edge Function.

The `document_retention_policies` table exists in production (confirmed via REST API). The `check-document-expiration` Edge Function handles expiry warnings (consistent with v1.0). But no function deletes or archives documents when their retention class TTL expires.

---

## 7. Production Tables — Full Inventory (REST API Verified)

Tables confirmed in `public` schema via live REST API:

**Core DMS tables:**
- `documents`
- `document_versions`
- `document_version_changes`
- `document_categories`
- `document_types`
- `document_metadata_fields`
- `document_approval_workflow` (canonical name — confirmed present)
- `document_comments`
- `document_cross_references`
- `document_processing_queue`
- `document_search_index`
- `document_system_settings`
- `document_retention_policies`
- `document_audit_trail`
- `document_access_log`
- `document_families`
- `document_family_members`
- `document_shares`
- `document_labels`
- `document_label_assignments`
- `document_workflows`
- `document_workflow_steps`
- `document_templates`
- `document_compliance_checks`
- `document_expiry_notifications`
- `document_quality_scores`
- `document_ai_analysis`
- `document_ai_feedback`
- `document_analysis_results`
- `document_archival_suggestions`
- `document_utilization_analytics`

**Notable: `document_approvals` does NOT appear** — confirming the naming bug.

**`ooh_dms` schema tables (from migrations):**
- `rejection_reason_codes` (seeded — 15 codes)
- `document_classifications`
- `classification_corrections`
- `document_anomalies`
- `approval_outcomes`
- `document_prescores`
- `entity_completeness_predictions`
- `memory_events`

**Not in v1.0:** The v1.0 audit listed only 4 tables (`dms_documents`, `document_approval_workflow`, `document_retention_classes`, `document_versions`). The production DB has 30+ document-prefixed tables in `public` plus 8 in `ooh_dms`. The module is significantly larger than v1.0 represented.

---

## 8. Discrepancies vs v1.0

| v1.0 Claim | v2.0 Finding | Severity |
|---|---|---|
| "All code uses `document_approval_workflow` (canonical name)" | `DocumentRepository` uses `document_approvals` (broken) | High |
| "No dedicated DMS admin UI" | Still true — but standalone `src/dms/` module now exists | Low (clarification) |
| "AI classification: OCR + regex extracts PO numbers" | Edge function has no OCR; receives pre-extracted text; classifier prompt targets Maltese planning apps | Medium |
| "Retention enforcement ✅ Built" | `enforce-retention-policies` enforces client GDPR retention, not document TTL | Medium |
| "DB schema: 4 tables" | 30+ tables in `public`, 8 in `ooh_dms` | Low (undercounted) |
| "dms-intelligence-processor classifies OOH document types" | Classifier is context-specific to Maltese bus shelter applications | Medium |
| "`dms` schema" as target | Three-schema reality: `public` (live), `ooh_dms` (live), `dms` (future, migration unapplied) | Medium |
| "PO number extraction via regex → event" | Not found in Edge Function; likely external to DMS boundary | Medium |

---

## 9. Verified Working

| Component | Status | Evidence |
|---|---|---|
| Document upload/download Edge Functions | ✅ Live | `upload-supplier-document`, `download-supplier-document` deployed |
| AI classification pipeline (4-action) | ✅ Live | `dms-intelligence-processor` — LLM via Lovable gateway |
| Classification dataset export | ✅ Live | `dms-export-classification-dataset` deployed |
| GDPR client retention enforcement | ✅ Live | `enforce-retention-policies` (scope: client data) |
| Document expiry checking | ✅ Live | `check-document-expiration` deployed |
| Signed PDF generation | ✅ Live | `generate-signed-pdf` deployed |
| `document_approval_workflow` table | ✅ Live in `public` | Confirmed via REST API |
| ooh_dms intelligence tables | ✅ Live | Created by `20260318130000_dms_v1_5_intelligence_layer.sql` |
| Rejection reason codes (15 seeded) | ✅ Live | Seeded in v1.5 migration |
| Legacy `services/` layer | ✅ Functional | Uses correct table names; `@deprecated` but not removed |
| New `src/dms/src/` approval operations | ❌ Broken | `document_approvals` table does not exist |
| Document-level TTL enforcement | ❌ Missing | No function archives/deletes on retention class expiry |
| `dms` schema migration | ⏳ Pending | Migration authored, not applied |

---

## 10. Technical Debt — Updated

| Issue | Severity | Status vs v1.0 |
|---|---|---|
| `DocumentRepository` queries non-existent `document_approvals` | **High** | New — introduced in Sprint 3 |
| `dms-intelligence-processor` classifier prompt hardcoded to Maltese planning context | **High** | New — not in v1.0 |
| Document-level TTL retention enforcement unimplemented | **High** | v1.0 marked as fixed — actually missing |
| `src/dms/schema/migration_extract_dms.sql` planning draft uses wrong table name `document_approvals` | Medium | New |
| `dms` schema migration unapplied — code and schema diverging | Medium | New |
| Backward-compat views in migration are read-only (`SELECT *`) — INSERT/UPDATE/DELETE will fail during transition | Medium | New |
| `search_path` update commented out in `20260419000000_extract_dms_to_dms_schema.sql` — PostgREST will not resolve `dms.*` tables without it | Medium | New |
| `LOVABLE_API_KEY` single point of failure — no fallback | Medium | New |
| Two parallel AI pathways (IngestionAgent + dms-intelligence-processor) not formally connected | Medium | New |
| Dual service layers (`services/` deprecated + `src/dms/src/`) — migration in progress but incomplete | Medium | New |
| No bulk re-classification | Low | Unchanged from v1.0 |
| Retention scheduling frequency not tenant-configurable | Low | Unchanged from v1.0 |
| Expiry warning threshold hardcoded | Low | Unchanged from v1.0 |

---

## 11. Immediate Actions Required

**Priority 1 — Fix before any production use of new module:**

1. Fix `DocumentRepository` table name:
   - File: `/workspace/group/everboarding/src/dms/src/repositories/DocumentRepository.ts`
   - Lines 143, 155, 166: `'document_approvals'` → `'document_approval_workflow'`

2. Fix `dms-intelligence-processor` classifier system prompt:
   - Remove Maltese bus shelter context from the `handleClassify` function
   - Make the domain context caller-supplied or generic OOH

**Priority 2 — Before applying schema migration:**

3. Uncomment `search_path` ALTER in `20260419000000_extract_dms_to_dms_schema.sql`
4. Replace read-only views with updatable views (or migrate all INSERT/UPDATE/DELETE callers first)
5. Delete or archive `src/dms/schema/migration_extract_dms.sql` (planning artifact with wrong table name)

**Priority 3 — Functionality gap:**

6. Implement document-level TTL enforcement in `enforce-retention-policies` or a new function — iterate `document_retention_policies`, compare `expires_at` on `documents`, archive/soft-delete expired records

---

## 12. Production Readiness Score

**5.5 / 10**

| Dimension | Score | Notes |
|---|---|---|
| Core storage (upload/download) | 9/10 | Solid, deployed, working |
| AI classification | 5/10 | LLM call works but prompt is wrong domain; no OCR; single point of failure |
| Approval workflow (legacy layer) | 8/10 | `DocumentApprovalService` correct and feature-complete |
| Approval workflow (new module) | 1/10 | `DocumentRepository` broken — queries non-existent table |
| Schema clarity | 4/10 | Three-schema state partially built; migration unapplied; planning artifact with wrong names |
| Retention enforcement | 3/10 | GDPR client retention works; document TTL enforcement absent |
| Architecture / code structure | 6/10 | Hexagonal design sound; dual-layer in-flight migration acceptable if completed |
| Test coverage | 2/10 | No test files found in `src/dms/` |

*Score penalised primarily by: broken approval operations in new module, mismatched AI prompt context, and absent document TTL enforcement despite being marked complete.*

---

*End of DMS Module Audit v2.0*
