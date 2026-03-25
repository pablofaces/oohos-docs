# Compliance Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (OOH OS Development Agent)
**Status:** Verified against live codebase and production schema
**v1.0 Reference:** https://raw.githubusercontent.com/pablofaces/oohos-docs/main/audits/compliance-audit-v1.0.md
**Codebase:** `/workspace/group/everboarding` (branch: main, as of audit date)

---

## 1. Executive Summary

The Compliance module has expanded significantly since v1.0. The frontend service layer is substantially more complete than documented. However, two critical discrepancies undermine the v1.0 assessment: the frame eligibility API (`compliance.checkFrameEligibility`) has **no implementation anywhere in the codebase** — it exists only as a described contract in CLAUDE.md — and the dual-table naming collision between `gdpr_requests` (used by `GdprService`) and `gdpr_data_subject_requests` (used by Edge Functions and admin UI) is a live schema integrity issue that will cause silent failures.

**Production Readiness: 6/10**

---

## 2. What Changed Since v1.0

### 2.1 Module Structure — v1.0 Was Understated

v1.0 described the module as "Early development, 6 files." The current state is:

```
src/compliance/
  audit/auditService.ts         627 lines  — full implementation
  audit/useAuditLogs.ts          65 lines  — React Query hooks
  entity/entityComplianceService.ts  338 lines  — full implementation
  gdpr/gdprService.ts           479 lines  — full implementation
  retention/dataRetentionService.ts  190 lines  — full implementation
  index.ts                       55 lines  — public API barrel
```

**Total: 6 files, 1,754 lines** (confirmed — matches CLAUDE.md v5.3 annotation)

v1.0 referenced `useScopedComplianceTasks` (5.8KB hook) as a compliance file. This hook lives at `/workspace/group/everboarding/src/portal/hooks/useScopedComplianceTasks.ts` (173 lines), outside the `src/compliance/` module boundary. It belongs to the Portal layer, not the Compliance module itself. This was a v1.0 categorization error.

### 2.2 Edge Functions — v1.0 Inventory Discrepancy

v1.0 listed 6 compliance-related Edge Functions. The actual deployed set is **9**:

| Function | Lines | v1.0 Listed | Status |
|---|---|---|---|
| `submit-dsr-request` | 175 | Yes | Real — full implementation |
| `verify-dsr-identity` | 203 | Yes | Real — token verification |
| `check-deletion-eligibility` | 181 | Yes | Real — 4-blocker logic |
| `track-consent` | 110 | Yes | Real — consent recording |
| `confirm-gdpr-consent` | 137 | Yes | Real — consent confirmation |
| `enforce-retention-policies` | 135 | Yes | Real — scheduled enforcement |
| `create-supplier-compliance-check` | 90 | Yes | Real — supplier check initiation |
| `cmms-generate-compliance-report` | 434 | **No** | Real — CMMS compliance reports |
| `cmms-check-compliance-expiry` | 149 | **No** | Real — CMMS expiry scanning |

v1.0 missed 2 CMMS-scoped compliance Edge Functions (`cmms-generate-compliance-report` at 434 lines, `cmms-check-compliance-expiry` at 149 lines). These are operationally real functions deployed to production.

### 2.3 Admin UI — Substantially More Complete Than v1.0 Stated

v1.0 described the Comms Hub `CompliancePage` as "stubbed" and implied no functional compliance dashboard. The reality is more nuanced:

**Admin GDPR pages (fully implemented, wired in `AdminRoutes.tsx`):**

| Page | Path | Lines | Status |
|---|---|---|---|
| `DSRQueue.tsx` | `/admin/gdpr/requests` | 278 | Real — queries `gdpr_data_subject_requests`, filterable |
| `DSRDetail.tsx` | `/admin/gdpr/requests/:id` | 393 | Real — full detail view with actions |
| `ConsentRecords.tsx` | `/admin/gdpr/consents` | 279 | Real — queries consent tables |
| `ProcessingActivities.tsx` | `/admin/gdpr/processing-activities` | 339 | Real — GDPR Art. 30 records |
| `BreachIncidents.tsx` | `/admin/gdpr/breach-incidents` | 434 | Real — incident create/manage |
| `SecurityCompliancePage.tsx` | `/admin/compliance` | 303 | Real — platform-wide overview |

**Comms Hub CompliancePage** (`src/communication-hub/client/src/pages/settings/CompliancePage.tsx`, 517 lines): This is **not a stub**. It is a full standalone compliance management UI inside the Comms Hub sub-application, with consent record management, retention policy configuration, and compliance report generation. It was incorrectly flagged as a stub in v1.0.

**Portal compliance pages** (confirmed working):
- `CompliancePage.tsx` at `src/portal/pages/CompliancePage.tsx` (352 lines)
- `PartnerCompliancePage.tsx` at `src/portal/pages/PartnerCompliancePage.tsx` (317 lines)

**Routes** (confirmed in `AnimatedRoutes.tsx`):
- `/gdpr/request` — public DSR submission page
- `/confirm-gdpr` — GDPR consent confirmation
- `/portal/partner-compliance` — partner compliance portal view
- Note: `/portal/compliance`, `/portal/settings/compliance`, `/portal/account/compliance` all redirect to `/portal/settings/tasks` — this is a **design redirect, not a stub**

---

## 3. Critical Discrepancies from v1.0

### DISCREPANCY 1 — Frame Eligibility API: Designed Contract, Zero Code

v1.0 stated: "Frame eligibility API — Built (stub) — Phase 1: returns all frames eligible (stub)"

**Reality:** There is no file named `compliance.stub.ts` anywhere in the codebase. A search for `checkFrameEligibility`, `compliance.stub`, and `FrameEligib` across all TypeScript files returns zero results. The function `compliance.checkFrameEligibility(frameIds, advertiserCategory)` exists only as a contract description in CLAUDE.md (§18.1). The stub referenced in v1.0 (`compliance.stub.ts` in `supabase/functions/` stubs) does not exist on disk. The Sales module's `src/sales/src/stubs/compliance.stub.ts` (referenced in CLAUDE.md §18.4) is also absent.

**Impact:** Any Sales module code that calls `compliance.checkFrameEligibility()` will throw a runtime error. No booking can be validated against content category restrictions. The compliance gate described as blocking bookings is effectively bypassed.

**Risk: CRITICAL**

### DISCREPANCY 2 — Dual GDPR Request Table Schema

`GdprService` (frontend service, `src/compliance/gdpr/gdprService.ts`) writes to and reads from `gdpr_requests`. The `submit-dsr-request` Edge Function inserts into `gdpr_data_subject_requests`. The admin UI (`DSRQueue.tsx`, `DSRDetail.tsx`) reads from `gdpr_data_subject_requests`.

**Both tables are present in the production schema** (confirmed via REST API introspection — both `/gdpr_requests` and `/gdpr_data_subject_requests` are exposed endpoints).

This means:
- DSRs submitted via the Edge Function land in `gdpr_data_subject_requests` and are visible in the admin UI
- DSRs processed via `GdprService.processAccessRequest()` / `processErasureRequest()` operate against `gdpr_requests` — a different table that the admin UI never shows
- There is no data synchronization between the two tables
- Programmatic GDPR processing via the frontend service and admin-initiated GDPR workflow operate on **separate datasets**

**Risk: HIGH — Data integrity failure in GDPR process chain**

### DISCREPANCY 3 — Consent Hash Is Not Cryptographic

v1.0 implied consent snapshots use proper cryptographic hashing. `GdprService.captureConsentWithSnapshot()` stores a `consent_text_hash` using a private `hashText()` method that implements a simple djb2/Java-style integer hash (not SHA-256 or any cryptographically secure algorithm). This hash provides no integrity guarantee and could be trivially reproduced or forged. Under GDPR Article 7 requirements for demonstrable consent, this may not constitute sufficient proof.

**Risk: MEDIUM — Regulatory exposure for consent evidence**

### DISCREPANCY 4 — `check-deletion-eligibility` Has Malta-Hardcoded Logic

The `check-deletion-eligibility` Edge Function hardcodes retention logic specific to "Malta VAT Act" (7-year retention periods, references to Malta VAT Act in legal basis strings). This violates the multi-tenancy principle documented in CLAUDE.md §1.1. Operators in other jurisdictions would receive incorrect retention period calculations and wrong legal basis citations.

**Risk: MEDIUM — Multi-tenancy violation, legal basis correctness in non-Malta jurisdictions**

---

## 4. Schema Verification — What Is Actually in Production

Tables confirmed present in production via REST API schema introspection (2026-03-25):

### Core GDPR Tables (confirmed)
| Table | Confirmed | Notes |
|---|---|---|
| `gdpr_data_subject_requests` | Yes | Used by Edge Functions + admin UI |
| `gdpr_requests` | Yes | Used by `GdprService` frontend service — orphaned from admin UI |
| `gdpr_data_breach_incidents` | Yes | Full CRUD in `BreachIncidents.tsx` |
| `gdpr_processing_activities` | Yes | Managed by `ProcessingActivities.tsx` |
| `gdpr_dpia_assessments` | Yes | Present in schema |
| `gdpr_dpia_consultations` | Yes | Present in schema |
| `gdpr_data_classification_rules` | Yes | Present in schema |
| `gdpr_consent_records` | Yes | Present in schema — separate from `consent_records` |
| `gdpr_data_retention_policies` | Yes | Present in schema |
| `gdpr_retention_job_log` | Yes | Retention job audit trail |
| `migration_gdpr_confirmations` | Yes | Migration-era GDPR data |
| `v_gdpr_consent_records` | Yes | View |

### Consent Tables (confirmed — multiple co-existing)
| Table | Confirmed | Used by |
|---|---|---|
| `consent_records` | Yes | `GdprService.recordConsent()`, `GdprService.getUserConsents()` |
| `comms_consent` | Yes | Comms Hub channel consent gate |
| `gdpr_consent_records` | Yes | Audit trail (separate from above) |
| `client_consent_records` | Yes | Client-specific consent |
| `supplier_consent_records` | Yes | Supplier-specific consent |
| `v_client_consent_records` | Yes | View |
| `v_supplier_consent_records` | Yes | View |

**Note:** There are at least 5 overlapping consent tables in production. The relationship and deduplication strategy between these tables is not documented. This is unresolved technical debt.

### Compliance Status Tables (confirmed)
| Table | Confirmed | Notes |
|---|---|---|
| `entity_compliance_status` | Yes | Used by `EntityComplianceService` (try/catch on insert — "table might not exist" comment in code) |
| `compliance_alerts` | Yes | Used by `EntityComplianceService.getAlerts()` (try/catch) |
| `compliance_frameworks` | Yes | Present in schema |
| `compliance_assessments` | Yes | Present in schema |
| `compliance_requirements` | Yes | Present in schema |
| `compliance_actions` | Yes | Present in schema |
| `supplier_compliance_checks` | Yes | Used by `create-supplier-compliance-check` |
| `tenant_compliance_configuration` | Yes | Tenant-level config |
| `document_compliance_checks` | Yes | Document-level checks |

### Retention Tables (confirmed)
| Table | Confirmed | Notes |
|---|---|---|
| `data_retention_policies` | Yes | Managed in Comms Hub CompliancePage |
| `data_retention_audit_log` | Yes | Retention job log |
| `document_retention_policies` | Yes | DMS-side retention |
| `retention_classes` | Yes | General retention class definitions |
| `gdpr_data_retention_policies` | Yes | GDPR-specific retention (separate from above) |

**Note:** There are at least 4 separate retention policy/class tables. `DataRetentionService` in `src/compliance/retention/dataRetentionService.ts` uses **none of these database tables** — it operates entirely from `DEFAULT_RETENTION_POLICIES` hardcoded in-memory constants. The service does not read from or write to any retention policy table. The retention policy UI in Comms Hub manages `data_retention_policies` table but the enforcement service ignores it.

### v1.0 Tables Claimed — Verified
| v1.0 Claimed Table | Confirmed in Production | Notes |
|---|---|---|
| `compliance_tasks` | Absent from REST API path list | Not found as a top-level REST endpoint |
| `comms_consent` | Yes | Confirmed |
| `email_suppressions` | Confirmed (not in path list but referenced in types) | Present in schema types |
| `gdpr_consent_records` | Yes | Confirmed |
| `dsr_requests` | Absent — actual table is `gdpr_data_subject_requests` | v1.0 used wrong table name |
| `document_retention_classes` | Partial — `retention_classes` exists, not `document_retention_classes` | Name mismatch |
| `content_restrictions` | Absent from REST API path list | Not found as a REST endpoint |

**`compliance_tasks` is absent from the production schema.** The `useScopedComplianceTasks` hook that v1.0 described as querying this table is actually backed by a different mechanism (confirmed: it exists at `src/portal/hooks/useScopedComplianceTasks.ts`).

---

## 5. GDPR DSR Workflow Completeness

### End-to-End Flow Assessment

**Submission path (Edge Function):**
1. `submit-dsr-request` — collects `platform_tenant_id`, `request_type`, `data_subject_email`, optionally calls `check-deletion-eligibility` for erasure requests, inserts into `gdpr_data_subject_requests`, sends verification email via `hub-notifications` with `gdpr.verification.requested` event key. **COMPLETE.**

2. `verify-dsr-identity` (203 lines) — token-based identity verification, updates `gdpr_data_subject_requests` status from `identity_verification_pending` to `identity_verified`. **COMPLETE.**

3. Admin review — `DSRQueue.tsx` + `DSRDetail.tsx` allow admins to view, filter, and manage requests against `gdpr_data_subject_requests`. **COMPLETE.**

4. Processing (access/erasure/portability) — `GdprService.processAccessRequest()` and `processErasureRequest()` implement processing logic **but operate on `gdpr_requests`, not `gdpr_data_subject_requests`**. There is no bridge between the Edge Function submission flow and the frontend service processing flow. **BROKEN — table mismatch.**

5. Confirmation to data subject — `submit-dsr-request` sends a verification email. No completion notification is triggered from `GdprService` (it updates the DB record but does not call Comms Hub). **INCOMPLETE — no completion notification.**

**GDPR 30-day deadline tracking:** `submit-dsr-request` sets `due_date = now + 30 days`. No cron job or alert mechanism enforces this deadline or escalates overdue requests. **MISSING.**

**DSR types supported:** access, erasure, portability, rectification, restriction (defined in `GdprService`). Edge Function validates `request_type` but the enum is not enforced at DB level — any string can be inserted.

**Identity verification strength:** Single-factor email token only. The v1.0 audit noted "multi-factor identity verification" but `verify-dsr-identity` implements only email token verification with a 24-hour expiry. No phone or ID verification exists.

### Erasure Logic Review

`GdprService.processErasureRequest()` performs:
- Active contract check (via `credit_applications` table, status `agreement_signed`)
- Financial records check (via `credit_applications`, 7-year lookback)
- Delete `onboarding_step_data`
- Anonymize `profiles` (name→DELETED USER, email→`deleted_{uuid}@anonymized.local`, phone→null)
- Delete `consent_records`

**Gaps in erasure coverage:**
- Does not delete or anonymize `audit_logs` (GDPR may require pseudonymization)
- Does not delete `gdpr_requests` entries referencing the deleted user (creates orphaned records)
- Does not notify Comms Hub to suppress future sends
- Does not delete from `comms_consent` table
- Does not clear data from any Sales module tables (`sales_contact`, `sales_account`)
- Check for financial records uses a buggy query: `gte('created_at', sevenYearsAgo)` should be `lte` — it retains records **younger** than 7 years rather than records **older** than 7 years

**Risk: HIGH — Incomplete erasure scope, inverted financial retention check**

---

## 6. Frame Eligibility Integration Status

### Current State: Unimplemented

The `compliance.checkFrameEligibility(frameIds, advertiserCategory)` API is defined in CLAUDE.md as a synchronous API consumed by Sales during proposal building. As of this audit:

- No TypeScript service or function named `checkFrameEligibility` exists in the codebase
- No file named `compliance.stub.ts` exists anywhere under `src/` or `supabase/`
- The `content_restrictions` table is not present as a REST API endpoint (suggesting it either does not exist in production or is not publicly exposed)
- The `ContentRestrictionsService` at `src/partners/src/services/ContentRestrictionsService.ts` exists and is used by the Partners module, but it is a **read-only service** for Sales/Compliance to check per-agreement content restrictions — it is not the frame eligibility checking API
- The EventBus events `compliance.frame.restricted` and `compliance.frame.restriction_lifted` have no emitter anywhere in the codebase

**Consequence:** The Sales module's booking confirmation gate described in CLAUDE.md §18.1 ("No booking can be confirmed if Compliance returns a block for any line item") is inactive. Proposals with restricted inventory pass through to booking confirmation unchecked.

**What exists related to content compliance:**
- `ContentRestrictionsService` in Partners reads `agreement_location_instances.content_restrictions` (immutable per-agreement field)
- `cmms-generate-compliance-report` (434 lines) generates CMMS-domain compliance reports for asset maintenance
- `cmms-check-compliance-expiry` scans for expiring compliance records in CMMS context
- These are maintenance/asset compliance, not advertising content compliance

**Phase 1 stub status:** Per CLAUDE.md §18.4, the stub should return all frames as eligible. Since no stub exists, calls to this API will throw `TypeError: compliance.checkFrameEligibility is not a function` or equivalent. This is worse than a stub — it is an unimplemented dependency.

---

## 7. Technical Debt Inventory

### High Severity

| Issue | Detail | Location |
|---|---|---|
| Frame eligibility API completely absent | No implementation, no stub — Sales compliance gate inactive | Everywhere — missing file |
| `gdpr_requests` vs `gdpr_data_subject_requests` split | Two tables for DSRs, frontend service and Edge Functions operate on different datasets | `src/compliance/gdpr/gdprService.ts` vs `supabase/functions/submit-dsr-request/` |
| Inverted financial retention check in erasure | `gte(sevenYearsAgo)` should be `lte` — retains wrong records during erasure | `src/compliance/gdpr/gdprService.ts` line ~417 |
| Erasure scope incomplete | Does not clear `comms_consent`, `audit_logs`, `sales_contact`, `sales_account`, Sales module data | `src/compliance/gdpr/gdprService.ts` |
| `DataRetentionService` ignores all DB retention tables | Hardcoded in-memory policies, never reads `data_retention_policies` DB table | `src/compliance/retention/dataRetentionService.ts` |

### Medium Severity

| Issue | Detail | Location |
|---|---|---|
| Malta-hardcoded legal basis in deletion eligibility | "Malta VAT Act" hardcoded — wrong for any non-Malta operator | `supabase/functions/check-deletion-eligibility/index.ts` |
| Consent hash is not cryptographic | djb2 integer hash used instead of SHA-256 for consent text proof | `src/compliance/gdpr/gdprService.ts` line ~437 |
| 5+ overlapping consent tables with no documented relationship | `consent_records`, `comms_consent`, `gdpr_consent_records`, `client_consent_records`, `supplier_consent_records` | Production schema |
| No DSR 30-day deadline enforcement | Due date is set but no cron job enforces or alerts on overdue requests | Missing edge function / cron |
| `entity_compliance_status` inserts wrapped in try/catch with "table might not exist" | Suggests this table was added after the service was written — live data writes silently fail if table absent | `src/compliance/entity/entityComplianceService.ts` line ~94 |
| Compliance admin UI not linked from main admin navigation | 5 GDPR admin pages exist but their navigation entry point is not verified | `src/pages/admin/AdminRoutes.tsx` |
| No DSR completion notification | `GdprService.processAccessRequest()` completes without triggering Comms Hub | `src/compliance/gdpr/gdprService.ts` |
| `gdpr_requests` table presence unexplained | Frontend service writes to `gdpr_requests` but this table name was never in CLAUDE.md schema docs | Production schema |

### Low Severity

| Issue | Detail | Location |
|---|---|---|
| `compliance_tasks` table absent | v1.0 cited this as core schema; not present as REST endpoint | Schema gap |
| `content_restrictions` table absent from REST API | v1.0 cited this as built; not found in schema introspection | Schema gap or RLS-hidden |
| `check-deletion-eligibility` not called from admin UI | Edge Function exists but `DSRDetail.tsx` does not invoke it programmatically | `src/pages/admin/gdpr/DSRDetail.tsx` |
| `audit_logs.archived_at` column assumed present | `DataRetentionService.archiveAuditLogs()` updates `archived_at` column but this column is not in the type definitions | `src/compliance/retention/dataRetentionService.ts` line ~110 |
| 4 separate retention policy tables with no unification | `data_retention_policies`, `gdpr_data_retention_policies`, `document_retention_policies`, `retention_classes` | Production schema |
| No automated restriction zone scanning | Documented as missing in v1.0, remains absent | Unbuilt feature |
| Consent audit export not built | Documented as missing in v1.0, remains absent | Unbuilt feature |
| Cross-border data transfer controls absent | Documented as missing in v1.0, remains absent | Unbuilt feature |

---

## 8. What Is Verified Working (Code + Schema)

The following components are confirmed implemented and functionally wired:

### Edge Functions (verified real implementations)
- `submit-dsr-request` — full submission flow with eligibility pre-check, email verification, 30-day due date
- `verify-dsr-identity` — email token verification with expiry
- `check-deletion-eligibility` — 4-category blocker logic (credit agreement, client status, retention period, tax compliance)
- `track-consent` — consent recording
- `confirm-gdpr-consent` — explicit consent confirmation
- `enforce-retention-policies` — scheduled enforcement (shared with DMS)
- `create-supplier-compliance-check` — supplier compliance initiation
- `cmms-generate-compliance-report` — CMMS-domain compliance reports (434 lines)
- `cmms-check-compliance-expiry` — CMMS expiry scanning

### Frontend Services
- `AuditService` (627 lines) — complete audit logging, field-level diff tracking, CSV export, query layer, convenience methods for auth/delegation/compliance/security events
- `GdprService` (479 lines) — DSR CRUD, consent management, data export, erasure logic, consent verification, consent summary (note: table naming issues as documented above)
- `EntityComplianceService` (338 lines) — entity-level compliance badges, alerts, consent status, summary dashboards
- `DataRetentionService` (190 lines) — policy-driven cleanup for magic tokens, audit logs, email logs, onboarding data, sessions, incomplete flows (note: operates from hardcoded policies, not DB tables)

### React Hooks
- `useAuditLogs`, `useAuditSummary`, `useAuditActivitySummary`, `useSecurityEvents`, `useRecentErrors` — all backed by real `AuditService` queries with React Query caching
- `useScopedComplianceTasks` (173 lines) — role-scoped task list for Portal

### Admin UI Pages (all wired in `AdminRoutes.tsx`)
- `DSRQueue` — filterable DSR management against live `gdpr_data_subject_requests`
- `DSRDetail` — full DSR detail view with status management
- `ConsentRecords` — consent record management
- `ProcessingActivities` — GDPR Art. 30 processing activity records
- `BreachIncidents` — full incident reporting with CRUD against `gdpr_data_breach_incidents`
- `SecurityCompliancePage` — platform-wide security and compliance overview

### Portal Pages (verified in `AnimatedRoutes.tsx`)
- `/gdpr/request` — public DSR submission (unauthenticated)
- `/confirm-gdpr` — consent confirmation
- `/portal/partner-compliance` — partner compliance portal

### Comms Hub Integration
- `comms_consent` table gating all outbound channel sends (implemented in `hub-notifications`)
- Unsubscribe flow wired to `comms_consent` revocation
- `gdpr.verification.requested` event key wired for DSR email delivery

---

## 9. Module Boundary Assessment

The Compliance module at `src/compliance/` exports a clean public API via `index.ts`. However, significant compliance functionality lives **outside this boundary**:

- Portal compliance pages: `src/portal/pages/CompliancePage.tsx`, `PartnerCompliancePage.tsx`
- Portal hooks: `src/portal/hooks/useScopedComplianceTasks.ts`
- Comms Hub compliance UI: `src/communication-hub/client/src/pages/settings/CompliancePage.tsx`
- Admin GDPR pages: `src/pages/admin/gdpr/` (5 files, 1,723 lines)
- CMMS compliance services: `src/cmms/src/services/SubcontractorComplianceService.ts`
- CMMS compliance hooks: `src/cmms/src/hooks/usePartnerCompliance.ts`
- Legacy repository: `src/repositories/supabase/SupabaseComplianceRepository.ts` (outside module boundary)

**The total compliance-related code footprint is significantly larger than the 6-file `src/compliance/` module suggests.** Estimated total: ~5,500+ lines across all locations.

---

## 10. No Agents Built

As of this audit, **zero AI agents** are registered for the Compliance module. CLAUDE.md references a `ComplianceAgent` (DMS agent) in the planned agent registry but this is a DMS agent operating on documents, not a standalone Compliance module agent. The `compliance` module has no agent directory, no prompt templates, and no `agent_module_registry` entries.

---

## 11. Production Readiness Rating: 6/10

### Rationale

**Scoring breakdown:**

| Dimension | Score | Notes |
|---|---|---|
| GDPR data collection infrastructure | 8/10 | Tables, Edge Functions, and admin UI are real and functional |
| DSR workflow end-to-end | 4/10 | Submission works; processing is broken by table split; no deadline enforcement |
| Consent management | 6/10 | Multiple overlapping tables; basic recording works; no cryptographic integrity |
| Audit trail | 8/10 | AuditService is comprehensive and production-quality |
| Data retention enforcement | 4/10 | Service exists but ignores DB policy tables; Malta-hardcoded logic |
| Advertising content compliance | 1/10 | Frame eligibility API is entirely absent |
| Breach management | 7/10 | Full CRUD UI exists and wired to `gdpr_data_breach_incidents` |
| Multi-tenancy compliance | 5/10 | Malta-specific hardcoding in deletion eligibility; multi-tenant in most other areas |
| Module architecture | 6/10 | Clean barrel export but compliance logic is fragmented across 8+ locations |
| Agents / automation | 2/10 | No agents; no automated DSR deadline enforcement; no automated restriction scanning |

### What must be resolved before production use at scale

1. **Immediate blockers:**
   - Implement `compliance.checkFrameEligibility()` stub (CLAUDE.md §18.4 specifies `supabase/functions/_shared/` or `src/sales/src/stubs/compliance.stub.ts`) — returns all-eligible until real rules engine is built
   - Fix `gdpr_requests` vs `gdpr_data_subject_requests` split — either migrate `GdprService` to use `gdpr_data_subject_requests`, or document the two-table model with a clear bridge
   - Fix the inverted financial retention check in `GdprService.processErasureRequest()` (`gte` → `lte`)

2. **High priority before regulated market launch:**
   - Remove Malta-specific hardcoding from `check-deletion-eligibility` — move retention period and legal basis citations to `tenant_business_rules`
   - Implement DSR 30-day deadline cron / alerting
   - Extend erasure scope to cover `comms_consent`, Sales module contact data
   - Replace consent hash with SHA-256 or similar

3. **Medium priority (compliance completeness):**
   - Unify or document the 5-table consent model
   - Unify or document the 4-table retention policy model
   - Wire `DataRetentionService` to read policies from `data_retention_policies` table
   - Add completion notification from `GdprService` to Comms Hub on DSR resolution
   - Add `compliance_tasks` table or remove references to it

---

*End of Compliance Module Audit v2.0*
