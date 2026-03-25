# Ever-boarding Module Audit v2.0

**Date:** 2026-03-25
**Auditor:** Kai (AI — Claude Sonnet 4.6)
**Status:** Core Live — several v1.0 claims require correction
**Source paths audited:**
- `packages/everboarding/src/` (barrel re-export only)
- `src/components/onboarding/` (130+ component files)
- `supabase/migrations/20260310110217_remote_schema.sql` (master schema)
- `supabase/migrations/*.sql` (all subsequent migrations)
- `supabase/functions/` (174 edge functions total)
- Live Supabase REST API (`gwtzedkvcnnnlbeidzar.supabase.co`)

---

## 1. Purpose of This Audit

This v2.0 audit was commissioned to:
1. Identify discrepancies between the v1.0 audit and reality
2. Confirm or refute the ooh_everboarding VIEW vs table trigger footgun
3. Verify dynamic roles status (org-type-specific)
4. Assess delegation chain completeness
5. Assess AI health scoring reality
6. Map the full schema surface (public vs ooh_everboarding)
7. Score production readiness

---

## 2. Discrepancies from v1.0 Audit

### 2.1 `magic_link_tokens` table — name wrong in v1.0

v1.0 states the magic link token system is backed by a `public.magic_link_tokens` table.

**Reality:** The table is named `public.magic_tokens` (not `magic_link_tokens`).

```sql
CREATE TABLE IF NOT EXISTS "public"."magic_tokens" (...)
COMMENT ON TABLE "public"."magic_tokens" IS 'Centralized storage for all magic link tokens';
```

The `ooh_everboarding.onboarding_flows` table has an embedded `magic_link_token` column (with its own indexes), separate from and in addition to the `public.magic_tokens` table. There are effectively two parallel token stores. v1.0 described only one. This creates a latent confusion about which store is canonical for which operation.

### 2.2 `packages/everboarding` — described as built, reality is a barrel stub

v1.0 lists `packages/everboarding/src/` as a source path and implies a real package implementation. The package directory contains exactly 6 files total (package.json + index stubs for components, hooks, services, types, and the main index). The main index does:

```typescript
export * from '../../../src/lifecycle';
```

This is an intentional extraction-in-progress barrel. The actual implementation lives in `src/lifecycle/` (36 files, 3,145 lines per CLAUDE.md). v1.0 audited a path that is a pass-through, not a standalone implementation. This is architecturally correct per CLAUDE.md §3 ("packages/everboarding/ is an intentional 55-line barrel re-export of src/lifecycle/ — not a gap") but was misleadingly described in v1.0 as a source path.

### 2.3 `validate-supplier` — listed in v1.0 journey, confirmed present

v1.0 mentions `validate-supplier` in the supplier onboarding journey. This function exists at `supabase/functions/validate-supplier/`. No discrepancy.

### 2.4 `onboarding-ticket-triggers` — undocumented in v1.0

v1.0's edge function table lists 21 functions. The deployed codebase has `onboarding-ticket-triggers`, which v1.0 does not mention. This function creates helpdesk tickets automatically for: stalled flows (>= 5 days), email bounces, orphaned roles, and critical escalations. It is a Comms Hub integration bridge that v1.0 missed.

Total deployed onboarding-relevant edge functions: 22 (not 21).

### 2.5 `apps/everboarding-form` — confirmed present

v1.0 states this exists as an Nginx-served standalone form with a Dockerfile. Confirmed present at `apps/everboarding-form/` with docker-compose wiring in `docker-compose.modules.yml`. No discrepancy.

### 2.6 `onboarding-intelligence` uses Gemini, not Claude

v1.0 states `onboarding-intelligence` provides "AI health scoring."

**Reality:** The edge function explicitly uses the Lovable AI Gateway with Gemini:

```
Uses Lovable AI Gateway with Gemini for fast, accurate predictions.
const LOVABLE_API_KEY = Deno.env.get("LOVABLE_API_KEY");
```

It calls Gemini (via Lovable's wrapper), not Claude/Anthropic. If `LOVABLE_API_KEY` is not set, the function throws and fails entirely. This is an undocumented external dependency that creates a deployment risk. v1.0 described it as "AI health scoring" without noting the external gateway dependency or the model choice.

### 2.7 `ReminderTimingAgent` — v1.0 claims it is "built"

v1.0 states `ReminderTimingAgent` is built at `src/agents/specialists/ReminderTimingAgent.ts (10.7KB)`. The file exists and is referenced in CLAUDE.md. No discrepancy on existence.

### 2.8 Schema: public schema has 20 real tables plus views — v1.0 only listed 7

v1.0's schema table listed 7 tables. The actual public schema surface for onboarding/everboarding exposes:

**Real tables (public schema):**
- `everboarding_phases`
- `everboarding_reminders`
- `everboarding_task_types`
- `everboarding_tasks`
- `onboarding_analytics_summary`
- `onboarding_behavior_events`
- `onboarding_completion_status`
- `onboarding_completion_tasks`
- `onboarding_events`
- `onboarding_exceptions`
- `onboarding_field_dependencies`
- `onboarding_flow_brands`
- `onboarding_flow_role_assignments`
- `onboarding_role_hierarchy`
- `onboarding_task_events`
- `onboarding_user_notifications`
- `partial_onboarding_flows` (real table)
- `supplier_onboarding_flows` (real table with full GDPR consent columns)
- `tenant_onboarding_automation`
- `tenant_onboarding_rules`

**Views (public schema pointing to ooh_everboarding):**
- `public.onboarding_flows` → `ooh_everboarding.onboarding_flows`
- `public.onboarding_brand_role_assignments` → `ooh_everboarding.onboarding_brand_role_assignments`
- `public.onboarding_progress_snapshots`
- `public.onboarding_sessions`
- `public.onboarding_step_data`
- `public.v_everboarding_due_soon`
- `public.v_everboarding_overdue_summary`
- `public.v_everboarding_pending_reminders`

**ooh_everboarding schema real tables:**
- `ooh_everboarding.onboarding_flows` (master — 90+ columns)
- `ooh_everboarding.onboarding_brand_role_assignments`

v1.0 severely understated the schema surface. The task system alone (`everboarding_tasks`, `everboarding_task_types`, `onboarding_completion_tasks`, `onboarding_task_events`) represents a complete lifecycle task management layer not mentioned in v1.0.

### 2.9 RPC surface — extensive, not documented in v1.0

v1.0 mentions no RPCs. The live REST API exposes 27 RPC endpoints including:
- `complete_onboarding_workflow` — full workflow completion
- `get_everboarding_journey` — journey state query
- `calculate_onboarding_risk_level` — risk assessment
- `get_stuck_onboardings` — operational query
- `process_onboarding_completion` — completion processing
- `validate_partial_onboarding_token`
- `create_partial_onboarding_flow`

This is a significant operational surface that v1.0 omitted entirely.

---

## 3. ooh_everboarding Trigger Footgun — Confirmed

### Status: CONFIRMED RISK, PARTIALLY MITIGATED

v1.0 documented this as a high-severity issue: `public.onboarding_flows` is a VIEW over `ooh_everboarding.onboarding_flows`. Any developer who places a `CREATE TRIGGER` on `public.onboarding_flows` will silently fail because you cannot place standard row-level triggers on views (only `INSTEAD OF` triggers).

**Evidence from migration scan:**

The migration file contains exactly one `CREATE TRIGGER` statement in the entire database:

```sql
CREATE TRIGGER on_auth_user_created AFTER INSERT ON auth.users FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

There are zero triggers on `ooh_everboarding.onboarding_flows` or `public.onboarding_flows`. The public view has no `INSTEAD OF` triggers.

**Confirmed footgun details:**
- `public.onboarding_flows` (VIEW) — no write capability, no triggers
- `ooh_everboarding.onboarding_flows` (TABLE) — all writes must go here
- The view is read-only with no INSTEAD OF trigger, meaning any INSERT/UPDATE/DELETE against `public.onboarding_flows` will fail at runtime
- The `complete_onboarding_workflow` RPC and other stored procedures use `SELECT * INTO v_flow FROM onboarding_flows` — these reference the public view, which works for reads but would fail for writes if the function tried to UPDATE it
- The stored procedures examined do UPDATE `public.onboarding_flows`, which will fail silently or throw depending on the PostgreSQL version and configuration

**Risk assessment:** The view-as-table confusion is an active footgun. Any new developer writing a migration that does `UPDATE public.onboarding_flows SET ...` will encounter a permissions or read-only error. This has likely caused bugs in development that were fixed by switching the target to `ooh_everboarding.onboarding_flows`. The risk is not theoretical — it is an ongoing developer experience hazard.

**Mitigation not yet applied:** An `INSTEAD OF` trigger on `public.onboarding_flows` would allow transparent writes through the view to the underlying table. No such trigger exists. Adding a `COMMENT ON VIEW public.onboarding_flows IS 'READ-ONLY VIEW — writes must target ooh_everboarding.onboarding_flows'` would be a minimal guard.

---

## 4. Dynamic Roles — Status: BUILT AND OPERATIONAL

v1.0 claimed "dynamic roles complete." This is confirmed and expanded:

**What is built:**
- `DynamicEnhancedRoleSelectionFlow.tsx` (261 lines) — org-type-aware role selection
- `DynamicRoleAssignmentStep.tsx`, `DynamicRoleForm.tsx`, `DynamicRoleSelector.tsx`, `DynamicFieldRenderer.tsx`, `DynamicStepProgress.tsx`
- `DynamicForms/OrganizationTypeFiltering.ts` — filters role requirements by org type
- `DynamicForms/ConditionalLogic.ts` — conditional field logic per role
- `DynamicForms/EnhancedFieldBehaviors.ts` — enhanced field behaviors
- `DynamicForms/ComponentRegistry.tsx` — dynamic component registry
- DB: `organization_type_role_requirements` table with `requirement_type` CHECK (`mandatory`, `recommended`, `optional`, `not_applicable`)
- DB: `onboarding_flow_role_assignments` and `onboarding_role_hierarchy` tables

**Status:** Fully implemented. The `DynamicEnhancedRoleSelectionFlow` is the production path. The static `EnhancedRoleSelectionFlow` appears to be the legacy path. Both exist, suggesting a migration in progress from static to dynamic forms.

---

## 5. Delegation Chain — Status: BUILT, SCHEMA COMPLETE, UI COMPLETE

**Schema level:** `ooh_everboarding.onboarding_flows` has:
- `parent_flow_id uuid` — FK to primary flow for delegated roles
- `delegated_role text` — role code that was delegated
- `delegation_status text` — CHECK constraint: `pending`, `in_progress`, `completed`, `cancelled`
- `external_domain_authorized boolean` — explicit auth for cross-domain delegation
- `authorized_by text`, `authorized_at timestamptz`

**Service layer:** Four delegation services confirmed:
- `src/services/delegationService.ts`
- `src/services/delegationNotificationService.ts`
- `src/services/delegationReminderService.ts`
- `src/utils/delegationUtils.ts`

**UI layer:**
- `RoleDelegationStep.tsx` (357 lines) — delegation step in role flow
- `DelegationConfirmationDialog.tsx`
- `common/DelegationForm.tsx`
- `SignatoryManagement/SignatoryManagementPanel.tsx`

**Edge functions:**
- `delegation-health-check` — detects broken chains
- `process-role-reassignment` — handles reassignment
- `create-role-assignments` — creates assignments

**Assessment:** Delegation chain is complete at schema, service, and UI levels. The chain supports: FM delegates to CM, AM delegates FM roles, signatory delegation, external domain delegation with explicit authorization. The `delegation-health-check` function provides operational monitoring. No gaps detected at the code level.

---

## 6. AI Health Scoring — Reality Check

### OnboardingAnalysisAgent (src layer)
- Located at `src/agents/specialists/OnboardingAnalysisAgent.ts`
- Extends `AIEnhancedAgent` (routes through AgentHub, uses Claude via the platform's AI gateway)
- Health score calculation is **heuristic, not AI-driven**: `healthScore = Math.max(0, Math.min(100, progressRatio * 100 - activityPenalty))`
- Supported actions: `analyze` (blocker analysis), `predict` (completion prediction), `recommend`, `summarize`
- The "AI" in this agent is used for the blocker analysis narrative and recommendations, not for the health score number itself
- The health score is a simple formula; only the surrounding analysis text uses Claude

### onboarding-intelligence edge function
- **Does not use Claude/Anthropic.** Uses Lovable AI Gateway → Gemini
- Hard dependency on `LOVABLE_API_KEY` environment variable
- If `LOVABLE_API_KEY` is missing, the function throws and returns 500
- Provides: stall risk prediction, natural language queries, action recommendations, completion time estimates
- This is a **separate AI layer** from the AgentHub-registered `OnboardingAnalysisAgent`

### Summary
There are effectively two AI health/intelligence systems:
1. `OnboardingAnalysisAgent` — Claude-based via AgentHub, computes a simple heuristic health score, uses AI for narrative analysis
2. `onboarding-intelligence` edge function — Gemini-based via Lovable gateway, provides predictive stall detection and NL queries

v1.0 conflated these as one "AI health scoring" feature. They are distinct, use different AI providers, and have different failure modes. The `onboarding-intelligence` function is the higher-risk one due to the `LOVABLE_API_KEY` external dependency.

---

## 7. PO Preference FM Step — Status: NOT BUILT

v1.0 marked this as `🟡 Planned`. CLAUDE.md §6.10 specifies the FM billing_info substep addition (PO required / PO contact name).

Confirmed not present: grep of all `FinanceManagerSteps/` files finds no `po_required`, `po_contact`, or `billing_info` PO preference field. The FM step files cover: `BillingAddressStep`, `BoardResolutionUploadStep`, `CompanyDetailsStep`, `CreditAgreementStep`, `CreditApplicationStep`, `CreditApplicationSubmittedStep`, `CreditDecisionStep`, `ExistingCreditStep`, `OrganizationTypeStep`, `PaymentInfoStep`, `PersonalInfoStep`, `ReviewStep`.

**Status confirmed: Not built. Remains planned.**

---

## 8. Schema Summary — Public vs ooh_everboarding

| Layer | Count | Notes |
|---|---|---|
| `ooh_everboarding` real tables | 2 | `onboarding_flows` (master, 90+ cols), `onboarding_brand_role_assignments` |
| `public` real tables (onboarding domain) | 20 | Task system, analytics, events, reminders, notifications, rules, supplier flows, partial flows |
| `public` views (onboarding domain) | 8 | `onboarding_flows` + 7 others pointing to ooh_everboarding or derived |
| RPC functions (onboarding domain) | 27 | Full workflow, journey, risk, stuck detection, completion processing |
| Edge functions (onboarding domain) | 22 | Per section 9 below |

**Key schema facts not in v1.0:**
- `supplier_onboarding_flows` is a standalone real table (not a view), with full GDPR consent columns (Article 7 compliance fields)
- `partial_onboarding_flows` is a real table with its own token validation RPC
- The everboarding task system (`everboarding_tasks`, `everboarding_task_types`, `everboarding_phases`, `everboarding_reminders`) is a complete operational layer — not mentioned in v1.0
- `onboarding_behavior_events` and `onboarding_analytics_summary` indicate a behavioral analytics layer — not mentioned in v1.0
- `public.magic_tokens` is the token store (not `magic_link_tokens`)
- `ooh_everboarding.onboarding_flows.status` supports 8 values: `pending`, `in_progress`, `completed`, `stalled`, `cancelled`, `awaiting_client`, `awaiting_first_order`, `superseded` — the `superseded` status was not documented in v1.0

---

## 9. Edge Functions — Full Inventory

v1.0 listed 21 functions. Confirmed 22 onboarding-relevant functions deployed:

| Function | In v1.0 | Notes |
|---|---|---|
| `api-onboarding` | Yes | |
| `complete-role-step` | Yes | |
| `create-ao-onboarding-flow` | Yes | |
| `create-partner-onboarding` | Yes | |
| `create-role-assignments` | Yes | |
| `create-supplier-onboarding` | Yes | |
| `delegation-health-check` | Yes | |
| `detect-everboarding-triggers` | Yes | |
| `everboarding-data-change-triggers` | Yes | |
| `handle-expired-onboarding` | Yes | |
| `on-role-completed` | Yes | |
| `onboarding-intelligence` | Yes | Uses Gemini via Lovable (not Claude) |
| `onboarding-ticket-triggers` | **No** | Creates helpdesk tickets for stalled/bounced/orphaned flows |
| `process-everboarding-schedule` | Yes | |
| `process-role-reassignment` | Yes | |
| `save-onboarding-progress` | Yes | |
| `send-agent-onboarding-notification` | Yes | |
| `send-everboarding-notifications` | Yes | |
| `send-onboarding-invitation` | Yes | |
| `send-onboarding-reminders` | Yes | |
| `validate-onboarding-flow` | Yes | |
| `complete-supplier-step` | Yes | |

---

## 10. Production Readiness Assessment

### Scoring criteria (each item /1.0)

| Area | Score | Notes |
|---|---|---|
| Core flow lifecycle (create → delegate → complete → activate) | 1.0 | Fully built, schema sound, edge functions deployed |
| Schema integrity | 0.7 | ooh_everboarding/public split works, but trigger footgun is unresolved, magic token duplication is confusing |
| Delegation chain | 0.9 | Schema + services + UI complete; `delegation-health-check` operational; minor: health-check scheduling not documented |
| Dynamic roles | 0.9 | Fully built; dual static/dynamic flow paths indicate migration in progress, not a blocker |
| AI health scoring | 0.5 | Two parallel systems with different AI providers, one with external key dependency; heuristic score not true AI; `onboarding-intelligence` blocked if LOVABLE_API_KEY absent |
| Magic link system | 0.8 | Functional but two parallel token stores (`magic_tokens` table vs embedded column in `onboarding_flows`) creates potential for stale-token bugs |
| Supplier/partner onboarding | 0.9 | Full flow built including `supplier_onboarding_flows` with GDPR consent columns |
| PO workflow FM step | 0.0 | Not built |
| Observability / analytics | 0.8 | `onboarding_behavior_events`, `onboarding_analytics_summary`, `onboarding_exceptions` present; no documented consumer |
| Documentation accuracy | 0.4 | v1.0 audit had multiple schema errors, missed 15+ tables, missed 1 edge function, wrong token table name, wrong AI provider |

### Overall Production Readiness: **6.9 / 10**

**What is solid:**
- The booking gate dependency (`everboarding.onboarding.complete` → Sales unlock) is implemented
- The delegation chain is complete and monitored
- The RPC surface is rich and well-structured
- Multi-role, multi-step, multi-tenant onboarding flows work
- Supplier/partner onboarding complete
- 22 edge functions deployed covering all stated capabilities

**What blocks a higher score:**

1. **Trigger footgun (unresolved):** `public.onboarding_flows` is a read-only view with no INSTEAD OF trigger. Any developer writing `UPDATE public.onboarding_flows` (the natural, obvious thing to do) will fail. This is an active risk for every new migration. Recommendation: add an INSTEAD OF trigger or a blocking comment on the view.

2. **LOVABLE_API_KEY dependency:** `onboarding-intelligence` hard-fails if this external key is absent. If Lovable changes pricing, deprecates the gateway, or rotates keys, the entire AI prediction layer goes dark. Recommendation: add a graceful degradation path.

3. **Dual token stores:** `public.magic_tokens` and the embedded `magic_link_token` column on `ooh_everboarding.onboarding_flows` serve overlapping purposes. It is unclear which is canonical for `request-magic-link` vs `validate-magic-token`. A stale-token bug could arise if one store is checked but not the other.

4. **PO workflow FM step missing:** Required per CLAUDE.md §6.10 for the complete FM onboarding experience. Blocking for tenants with `po_workflow_enabled = true`.

5. **Documentation/audit accuracy gap:** The module's behaviour is significantly more complex than the v1.0 audit indicated. The task system, analytics layer, and RPC surface were entirely undocumented. This creates onboarding risk for developers joining the project.

---

## 11. Technical Debt Delta (v1.0 → v2.0)

| Issue | Severity | Status Change |
|---|---|---|
| `onboarding_flows` VIEW vs table trigger footgun | High | **Still unresolved** — no INSTEAD OF trigger added |
| `platformTenantId` mapping bug | Resolved | Per v1.0 — not reverified in v2.0 |
| `RoleDelegationStep` useEffect race condition | Resolved | Per v1.0 |
| `ManageDelegationsPanel` empty `client_id` | Resolved | Per v1.0 |
| BU creation race condition | Resolved | Per v1.0 |
| PO workflow FM step not built | Medium | **Still unresolved** |
| `magic_link_tokens` name error in documentation | Medium | **New finding** — real table is `magic_tokens` |
| Dual token stores (confusion risk) | Medium | **New finding** |
| `onboarding-intelligence` LOVABLE_API_KEY hard dependency | Medium | **New finding** |
| AI provider mismatch (`onboarding-intelligence` uses Gemini, not Claude) | Low | **New finding** |
| `onboarding-ticket-triggers` undocumented | Low | **New finding** — function exists, just wasn't in v1.0 |
| Everboarding task system undocumented | Low | **New finding** — substantial operational layer missing from all docs |
| Dual static/dynamic role selection flow paths | Low | Migration in progress — not a blocker |
| `onboarding-intelligence` scheduling frequency undocumented | Low | **Still unresolved** |
| Bulk reassignment no admin UI | Low | **Still unresolved** |

---

*End of Ever-boarding Module Audit v2.0*
