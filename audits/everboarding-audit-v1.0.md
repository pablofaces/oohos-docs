# Ever-boarding Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Core Live — Dynamic Roles + Delegation Complete  
**Source paths audited:** `packages/everboarding/src/`, `apps/everboarding-form/src/`, `supabase/functions/` (onboarding-*), `CLAUDE.md §§6.7, 18.2`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Ever-boarding |
| DB schema | `ooh_everboarding` (actual table: `ooh_everboarding.onboarding_flows`) |
| Package path | `packages/everboarding/src/` |
| App path | `apps/everboarding-form/src/` (external-facing FM completion form) |
| Frontend path | `src/` root (embedded in main app) |
| Owner | Horizontal (platform-wide) |
| Phase | Dynamic roles + delegation complete; AI health layer active |

---

## 2. Current State Summary

Ever-boarding is OOH OS's client onboarding orchestration engine. It manages the full journey from first invitation through to commercial activation, coordinating role delegation chains, document collection, GDPR consent, credit checks, and the booking gate.

**Critical dependency for Sales:** A booking **cannot** be confirmed until `onboarding_status = complete` for the account (`Confirm_Order_With_Onboarding_Check`). This was proven in the V2 Salesforce flow and is enforced as a hard gate.

**What is built:**
- Full onboarding flow lifecycle (create → delegate → complete → activate)
- Role delegation chains (FM delegates to CM, AM delegates FM roles)
- Magic link-based invitation system with token types (`inv`, `del`, `sig`)
- Dynamic role selection with org-type-specific role requirements
- `delegation-health-check` — detects broken delegation chains
- `process-role-reassignment` — handles role reassignment on delegation failure
- Partner and supplier-specific onboarding flows
- `OnboardingAnalysisAgent` — AI health scoring of onboarding journeys
- `ReminderTimingAgent` — AI-optimal reminder scheduling
- `apps/everboarding-form/` — standalone Nginx-served form for external-facing FM completion
- Ever-boarding integration in Portal (`everboardingIntegration.ts`, `partnerEverboardingIntegration.ts`)

**Key bugs fixed:**
- `platformTenantId` always empty → fixed via `onboarding.flowId` mapping
- `RoleDelegationStep` useEffect race condition → fixed with `!rolesLoading` guard
- `ManageDelegationsPanel` empty `client_id` on reassign → fixed
- BU creation race condition → `23505` handler + DB unique constraints

---

## 3. Client Journey

### New Client (FM - Finance Manager)
1. Sales creates account → `sales.account.created` emitted → Ever-boarding begins journey
2. Invitation sent (magic link) → FM receives email → lands on `apps/everboarding-form/`
3. FM completes steps: KYB documents, company details, GDPR consent, contact verification
4. FM delegates roles: assigns CM (Commercial Manager), may delegate additional sub-roles
5. Delegated contacts receive magic link invitations, complete their steps
6. All required roles completed → `onboarding_status = complete` → `everboarding.onboarding.complete` emitted
7. Sales booking gate unlocked — bookings can now be confirmed

### Partner/Agency Onboarding
- `create-partner-onboarding` Edge Function — partner-specific flow
- `partnerEverboardingIntegration.ts` in Portal — partner journey management

### Supplier/Agent Onboarding
- `create-supplier-onboarding` Edge Function — supplier-specific flow
- `create-ao-onboarding-flow` — Account Owner onboarding flow
- `complete-supplier-step`, `validate-supplier` Edge Functions

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| Onboarding flow creation (client) | ✅ Built | `api-onboarding`, `create-ao-onboarding-flow` |
| Partner onboarding | ✅ Built | `create-partner-onboarding` |
| Supplier onboarding | ✅ Built | `create-supplier-onboarding`, `complete-supplier-step` |
| Role delegation (FM → CM, sub-roles) | ✅ Built | `create-role-assignments`, `complete-role-step` |
| Magic link token system | ✅ Built | `request-magic-link`, `validate-magic-token`; types: `inv`, `del`, `sig` |
| Invitation emails | ✅ Built | `send-onboarding-invitation`, `send-onboarding-reminders` |
| Agent onboarding notifications | ✅ Built | `send-agent-onboarding-notification` |
| `delegation-health-check` | ✅ Built | Detects broken delegation chains |
| `process-role-reassignment` | ✅ Built | Handles reassignment on delegation failure |
| `handle-expired-onboarding` | ✅ Built | Cleans up expired flows |
| `detect-everboarding-triggers` | ✅ Built | Event-driven onboarding trigger detection |
| `everboarding-data-change-triggers` | ✅ Built | DB change listeners for onboarding state |
| `process-everboarding-schedule` | ✅ Built | Scheduled processing (reminders, expiry) |
| `on-role-completed` | ✅ Built | Post-role-completion orchestration |
| `save-onboarding-progress` | ✅ Built | Incremental progress saving |
| `validate-onboarding-flow` | ✅ Built | Flow validation before activation |
| `OnboardingAnalysisAgent` | ✅ Built | `src/agents/specialists/OnboardingAnalysisAgent.ts` (12.5KB) |
| `ReminderTimingAgent` | ✅ Built | `src/agents/specialists/ReminderTimingAgent.ts` (10.7KB) |
| `apps/everboarding-form/` | ✅ Built | Nginx-served standalone form (Dockerfile present) |
| Dynamic roles (org-type-specific requirements) | ✅ Built | `role_templates`, `organization_type_role_requirements` |
| Supervisory tiers config | ✅ Built | Stored in `platform_tenants.role_config` JSONB |
| `onboarding-intelligence` Edge Function | ✅ Built | AI health scoring |
| BU + Brand dedup constraints | ✅ Built | DB unique constraints via migration `20260306150000` |
| Portal onboarding status page | ✅ Built | `OnboardingStatusPage.tsx` in Portal |
| PO workflow FM step (capture preferences) | 🟡 Planned | §6.10 FM step addition — not confirmed built |
| Bulk onboarding for marketplace clients | ❌ Missing | Phase 3b — not yet designed |
| Self-service onboarding without invitation | ❌ Missing | Clients must receive magic link invitation |

---

## 5. Schema Detail

**Key architectural note:** The `onboarding_flows` table exposed in `public` schema is a **VIEW**. The actual table is `ooh_everboarding.onboarding_flows`. All triggers must be placed on `ooh_everboarding.onboarding_flows`.

**Multi-tenancy:** All tables include `tenant_id` (as `platform_tenant_id` on some tables) + RLS enforced

| Table | Schema | Purpose |
|---|---|---|
| `ooh_everboarding.onboarding_flows` | `ooh_everboarding` | Master flow record — parent_flow_id for delegation chains |
| `role_templates` | `public` | Role definitions with category CHECK constraint (9 categories) |
| `organization_type_role_requirements` | `public` | Per-org-type role requirements (mandatory/recommended/optional/not_applicable) |
| `platform_tenants` | `public` | Tenant records — `role_config JSONB` stores supervisory tiers config |
| `clients` | `public` | Client records — `primary_contact_email`, `mobile_phone` |
| `magic_link_tokens` | `public` | Magic link tokens with type (`inv`/`del`/`sig`) and expiry |
| `business_units` | `public` | Business unit records with FM/AO contact FKs |

### Role Template Categories (CHECK constraint)
`executive`, `finance`, `operations`, `legal`, `compliance`, `creative`, `technical`, `governance`, `administration`, `planning`

### Org Type Role Requirement Types (CHECK constraint)
`mandatory`, `recommended`, `optional`, `not_applicable`

### Key field mappings (discovered during migrations)
- `clients.primary_contact_email` (not `contact_email`)
- `clients.mobile_phone` (not `phone`)
- `organization_type_role_requirements.organization_type_id` (not `org_type_id`)
- `organization_type_role_requirements.requirement_type` CHECK uses `mandatory` (not `required`)

---

## 6. Role Model

| Role | Ever-boarding access |
|---|---|
| `platform_admin` | Full access to all onboarding flows |
| `account_owner` (`ao`) | Manage own client's onboarding journey |
| `finance_manager` (`fm`) | Complete FM-specific steps, delegate sub-roles |
| `commercial_manager` (`cm`) | Complete CM-specific steps |
| `partner` | Partner-specific onboarding flow |
| `supplier` | Supplier onboarding flow |
| `service_provider` | Service provider onboarding (Site Dev) |

---

## 7. EventBus Surface

### Outbound Events (Ever-boarding → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `everboarding.onboarding.complete` | All required roles complete | Sales (unlocks booking gate), Portal |
| `everboarding.onboarding.stalled` | Required step overdue / blocked | Sales (blocks booking), Portal (shows blocker), Comms Hub |
| `everboarding.account.health_updated` | AI health score recalculated | Sales (visible in account view), `LifecycleAgent` |

### Inbound Events (Platform → Ever-boarding)

| Event | Source | Effect |
|---|---|---|
| `sales.account.created` | Sales | Begins client onboarding journey |
| `sales.booking.confirmed` | Sales | Records commercial milestone |
| `sales.account.churned` | Sales | Closes lifecycle journey |

---

## 8. Edge Functions

| Function | Purpose |
|---|---|
| `api-onboarding` | REST API for onboarding operations |
| `create-ao-onboarding-flow` | Create Account Owner flow |
| `create-partner-onboarding` | Create partner-specific flow |
| `create-role-assignments` | Assign roles within a flow |
| `create-supplier-onboarding` | Create supplier-specific flow |
| `complete-role-step` | Mark a role step complete |
| `complete-supplier-step` | Mark supplier step complete |
| `detect-everboarding-triggers` | Event-driven trigger detection |
| `everboarding-data-change-triggers` | DB change listeners |
| `handle-expired-onboarding` | Expiry cleanup |
| `on-role-completed` | Post-role orchestration |
| `process-everboarding-schedule` | Scheduled processing |
| `process-role-reassignment` | Role reassignment handling |
| `save-onboarding-progress` | Incremental progress save |
| `validate-onboarding-flow` | Flow validation |
| `send-onboarding-invitation` | Magic link invitation dispatch |
| `send-onboarding-reminders` | Reminder scheduling |
| `send-everboarding-notifications` | General notifications |
| `send-agent-onboarding-notification` | Agent-specific notifications |
| `delegation-health-check` | Detects broken delegation chains |
| `onboarding-intelligence` | AI health scoring |

---

## 9. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| `onboarding_flows` VIEW vs actual table confusion | High | Public schema exposes a VIEW; all triggers on actual `ooh_everboarding.onboarding_flows` — easy to place trigger on wrong table |
| `platformTenantId` mapping bug (fixed) | Resolved | `flow.platform_tenant_id` was mapped to `flowId` — fix applied in `onboardingService.ts` |
| `RoleDelegationStep` useEffect race condition (fixed) | Resolved | `!rolesLoading` guard added |
| `ManageDelegationsPanel` empty `client_id` (fixed) | Resolved | Resolves `client_id` from parent flow before delegating |
| BU creation race condition (fixed) | Resolved | `23505` handler + unique constraints via `20260306150000` migration |
| PO workflow FM step not confirmed built | Medium | §6.10 specifies adding FM step for PO preference capture — status unclear |
| No self-service onboarding | Medium | All clients must receive invitation — limits marketplace self-signup (Phase 3b blocker) |
| `apps/everboarding-form/` is a separate Docker app | Low | Separate deployment concern — needs CI/CD integration |
| Magic link token expiry not per-tenant-configurable | Low | Should live in `tenant_business_rules` |
| `delegation-health-check` scheduling frequency not documented | Low | How often does it run? Should be configurable. |
| Bulk reassignment not UI-supported | Low | `process-role-reassignment` is function-only, no admin UI for bulk operations |

---

*End of Ever-boarding Module Audit v1.0*
