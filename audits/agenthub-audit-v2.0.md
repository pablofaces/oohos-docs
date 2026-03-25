# AgentHub + Operational Memory Audit v2.0

Date: 2026-03-25
Auditor: Kai (AI)
Source: Live codebase at `/workspace/group/everboarding/`, live DB via REST API, v1.0 audit comparison
Reference: `agenthub-audit-v1.0.md` (dated 2026-03-23, auditor: Cascade)

---

## 1. Methodology

This audit cross-references five data sources:

1. Filesystem walk of `src/agents/` (100+ files mapped)
2. Supabase REST API introspection — full OpenAPI spec (9.8 MB)
3. Migration SQL scan (`supabase/migrations/*.sql`)
4. Key file reads: AgentHub.ts, LearningEngine.ts, CircuitBreakerRegistry.ts, Mode2FailureHandler.ts, AgentProfile.ts, AgentRegistry.ts
5. CLAUDE.md §7 (canonical spec)

---

## 2. Discrepancies from v1.0

### 2.1 OpMemory Schema — WRONG in v1.0

v1.0 claimed: `sales_operational_memory` table owned by Sales schema. "AgentHub routes all writes via Sales API."

Reality: The canonical table is `ooh_agents.operational_memory` in a dedicated `ooh_agents` schema. Migration `20260316100859_create_ooh_agents_operational_memory.sql` confirms this. All writes use `writeOpMemoryEvent()` from `src/lib/agents/opMemory.ts` — no Sales API indirection. This is a significant factual error in v1.0.

The v1.0 "technical debt" item noting the Sales schema ownership was never accurate — the correct schema was already in place when v1.0 was written.

### 2.2 Autonomy Tier Labels — WRONG in v1.0

v1.0 used: `L1 recommends`, `L2 acts_and_notifies`, `L3 acts_silently`, `L4 autonomous`, `L5 approval_required`

Reality (from `src/agents/shared/types.ts`):

```
'disabled' | 'approval_required' | 'supervised' | 'silent' | 'full'
```

The v1.0 label system (L1–L5 with distinct names) does not exist in the codebase. The actual type uses five string literal values. The ordering is: `disabled → approval_required → supervised → silent → full`, confirmed by `AIEnhancedAgent.ts`:

```typescript
const tierOrder = ['disabled', 'approval_required', 'supervised', 'silent', 'full'];
```

CLAUDE.md §6.2 labels these L1–L4 (skipping `disabled`) but the DB and code use the string values, not numeric labels. v1.0's `L5 approval_required` treatment as an override level is incorrect — `approval_required` is simply the lowest active tier (L1), not an override flag.

### 2.3 Edge Functions — PARTIALLY WRONG in v1.0

v1.0 claimed these edge functions exist: `ai-gateway`, `ai-autopilot`, `ai-execution-pipeline`, `onboarding-intelligence`, `auto-resolve-exceptions`

Reality (from `ls supabase/functions/ | grep -iE "agent|autonomy|circuit|mode2|sla"`):

```
check-agent-advancement
check-mode2-slas
process-agent-events
process-agent-tasks
send-agent-onboarding-notification
```

The functions `ai-gateway`, `ai-autopilot`, `ai-execution-pipeline`, `onboarding-intelligence`, and `auto-resolve-exceptions` do not exist. The actual edge functions are fewer and named differently. v1.0 appears to have conflated planned architecture with deployed reality.

### 2.4 Agent Count — INFLATED in v1.0

v1.0 claimed "25+ agents registered" and listed 18 Sales agents including `OnboardingAnalysisAgent` and `ReminderTimingAgent` as Sales agents.

Reality:
- `OnboardingAnalysisAgent` and `ReminderTimingAgent` are in `src/agents/specialists/` as standalone `.ts` files (not in a subdirectory with prompts), and are classified under the `lifecycle` module in CLAUDE.md, not Sales.
- The BUILT_IN_AGENTS array in `AgentProfile.ts` defines 14 built-in agents across: helpdesk (3), portal (4), onboarding (1), signatures (1), communications (1), dms (4). These are the registry-seeded agents.
- Specialist agents in `src/agents/specialists/` number 16 named directories/files (14 subdirectory agents + OnboardingAnalysisAgent + ReminderTimingAgent).
- CLAUDE.md §7.3 gives the authoritative count: 28 platform AgentHub agents.

### 2.5 CMMS Agent Names — WRONG in v1.0

v1.0 listed CMMS agents as: `TriageAgent`, `DispatchAgent`, `PredictiveMaintenanceAgent`, `WeatherRiskAgent`, `CampaignImpactAgent`, `VerificationAgent`, `ReportingAgent`

CLAUDE.md §7.3 canonical names: `MaintenanceAgent`, `TelemetryAgent`, `ImpactAssessmentAgent`, `SLAAgent`, `PredictiveAgent`, `PartInventoryAgent`, `RouteAgent`

These are entirely different names. v1.0 used invented names not present in any file.

### 2.6 Learning Engine Integration Method — IMPRECISE in v1.0

v1.0 described the learning engine as subscribing to `agenthub.outcome_signal` EventBus events. Reality: `LearningEngine.ts` exposes a `receiveOutcomeSignal()` method that accepts an `OutcomeSignal` object directly — it is called programmatically, not via an EventBus subscription. The in-process buffer-and-batch pattern (batch size 20, flush every 30 seconds) is not mentioned in v1.0.

---

## 3. Actual Agent Count

### Platform AgentHub (src/agents/specialists/)

Specialist agents with full subdirectory structure (TypeScript file + prompts):

| Agent | Module | Prompts |
|---|---|---|
| AccountHealthAgent | sales | system.v1.0 + task.v1.0 |
| AttributionAgent | sales | system.v1.0 + task.v1.0 |
| AvailabilityForecastAgent | sales | system.v1.0 + task.v1.0 |
| CreativeAdvisorAgent | sales | system.v1.0 + task.v1.0 |
| DeliveryOptimisationAgent | sales | system.v1.0 + task.v1.0 |
| DiscountApprovalAgent | sales | system.v1.0 + task.v1.0 |
| EnquiryAgent | sales | system.v1.0 + task.v1.0 |
| FollowUpAgent | sales | system.v1.0 + task.v1.0 |
| FrameSubstituteAgent | sales | system.v1.0 + task.v1.0 |
| LeadScoringAgent | sales | system.v1.0 + task.v1.0 |
| NegotiationAgent | sales | system.v1.0 + task.v1.0 |
| ProgrammaticAgent | sales | system.v1.0 + system.v2.0 + task.v1.0 |
| ProposalAgent | sales | system.v1.0 + system.v2.0 + task.v1.0 + task.v2.0 + context_window.v1.0 |
| RenewalAgent | sales | system.v1.0 + task.v1.0 |
| WinLossAgent | sales | system.v1.0 + task.v1.0 |
| YieldAgent | sales | system.v1.0 + system.v2.0 + task.v1.0 |
| OnboardingAnalysisAgent | lifecycle | no prompt subdirectory |
| ReminderTimingAgent | lifecycle | no prompt subdirectory |

Specialist count: 18 files in specialists directory (16 with full prompt structure, 2 stub-style flat files)

### BUILT_IN_AGENTS (AgentProfile.ts — registry-seeded)

14 agents: helpdesk.triage, helpdesk.response, helpdesk.escalation, portal.support, portal.billing, portal.notifications, portal.documents, onboarding.analysis, signatures.reminder, communications.scheduler, dms.classification, dms.anomaly_detection, dms.approval_learning, dms.completeness_prediction

These are the agents that seed `agent_profiles` table on initialization. They are distinct from the specialist sales agents — the two sets do not overlap.

### CLAUDE.md §7.3 Authoritative Count: 28 platform agents

Breakdown per §7.3:
- 16 Sales agents
- 1 Lifecycle (ReminderTimingAgent; OnboardingAnalysisAgent also lifecycle but counted separately)
- 5 CMS (CreativeApprovalAgent, ScheduleOptimisationAgent, ConflictResolutionAgent, UnderDeliveryAgent, ProgrammaticYieldAgent)
- 3 DMS (IngestionAgent, ComplianceAgent, ExpiryAgent)
- 3 Ever-boarding (OnboardingAgent, LifecycleAgent, ContractAgent)

Module-local (not in platform AgentHub): 7 CMMS agents (use outcomeSignalWriter), 4 DMS intelligence agents (use dmsEventBus).

Note: The 5 CMS agents, 3 DMS platform agents, and 3 Ever-boarding agents appear in CLAUDE.md as "planned" but are listed as fully registered at MVP. No TypeScript files for these agents were found in `src/agents/specialists/` — they are registered in `agent_module_registry` DB table but lack codebase implementations outside the Sales and Lifecycle specialists.

---

## 4. Autonomy System Status

### What is implemented

The autonomy tier type is defined and used throughout. `AIEnhancedAgent.ts` resolves the current tier at execution time by querying `agent_policies` table, constructing the `tierOrder` array, and selecting prompts based on position. Tier-aware prompt selection is operational.

`AutonomyAdvancementChecker.ts` (referenced from LearningEngine) evaluates advancement eligibility and can both propose (writes pending OpMemory event for human approval) and auto-apply (rollbacks only — always auto-applied for safety).

Circuit breaker is fully implemented (closed → open → half_open) with DB persistence via `agent_circuit_breaker` table. Default config: failure threshold 5, success threshold 2, timeout 60 seconds.

### What is not yet operational

All agents start at `approval_required` (L1). No agent has a promotion track record because the system has zero production history. The 180-day advancement requirement means no agent can reach `supervised` (L2) until at minimum mid-September 2026 from a cold start — assuming continuous positive track record from launch.

The `LearningEngine.checkAdvancement()` method uses a `now % 10 !== 0` modulo gate — approximately 90% of outcome signals skip the advancement check. This is a throttle, not a bug, but it means advancement checks are probabilistic rather than guaranteed on every qualifying signal.

No circuit breakers have ever been tripped (system is pre-production). The Mode 2 autonomous marketplace itself is not yet live — `booking_mode = 'marketplace_autonomous'` is a DB field that exists but has no live traffic.

---

## 5. OpMemory Schema (ooh_agents)

Schema: `ooh_agents` (dedicated, not Sales schema — v1.0 was wrong)
Table: `ooh_agents.operational_memory`
Migration: `20260316100859_create_ooh_agents_operational_memory.sql`

Key columns (confirmed from migration):

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| tenant_id | UUID | required |
| correlation_id | UUID | links events in a workflow |
| parent_action_id | UUID | FK → self (immediate predecessor) |
| root_action_id | UUID | FK → self (originating action) |
| module | varchar | e.g. 'sales', 'cms', 'dms' |
| entity_type | varchar | e.g. 'booking', 'proposal' |
| entity_id | UUID | |
| actor_type | varchar | 'human' / 'ai_agent' / 'system' / 'external' |
| actor_id | varchar | |
| event_type | varchar | e.g. 'mode2.failure.detected' |
| agent_name | varchar | null for human/system actors |
| agent_version | varchar | |
| reasoning | text | agent's stated reasoning |
| context_snapshot | jsonb | state snapshot at event time |
| ai_confidence | float | 0–1 |
| confidence_breakdown | jsonb | |
| model_version | varchar | |
| human_override | bool | was this action overridden |
| human_override_by | UUID | |
| human_override_reason | text | |
| override_delta | jsonb | what changed |
| booking_mode | varchar | 'direct' / 'marketplace_autonomous' |
| autonomy_level_at_action | int | tier index at time of action |
| prompt_template_id | UUID | |
| prompt_template_version | varchar | |
| outcome | varchar | |
| outcome_positive | bool | the learning signal |
| outcome_detail | jsonb | |
| outcome_recorded_at | timestamptz | |
| created_at | timestamptz | |

Immutability: A `BEFORE UPDATE` trigger (`ooh_agents.enforce_opmem_immutability`) blocks all column updates except outcome fields. Append-only is enforced at DB level, not just application level.

RLS: 6 policies — tenant read (own), service role full, authenticated insert, authenticated outcome-only update.

Additional DB tables confirmed in migrations and REST API:

| Table | Purpose |
|---|---|
| agent_circuit_breaker | Per-agent circuit state (closed/open/half_open) |
| agent_entity_lock | Exclusive/shared locks with TTL |
| agent_autonomy_advancement | Audit trail of tier changes |
| agent_mode2_sla_tracking | Mode 2 deadline + breach tracking |
| agenthub_agent_dependency | Topological sort inputs |
| agent_prompt_template | Versioned prompts per agent per tier |
| agent_module_registry | Capabilities + mode2_safe_default |
| agent_policies | Per-tenant autonomy level overrides |
| agent_profiles | Registry-seeded agent identities |
| agent_statistics | Aggregated performance metrics |
| agent_task_queue | Async task queue |
| agent_events | Outcome signal log |
| agent_human_overrides | Human approve/reject records |

Older `ai_*` tables (`ai_agent_actions`, `ai_agent_learning_data`, `ai_agent_predictions`, `ai_context_memory`, `ai_conversations`, `ai_messages`, `ai_tool_executions`) also exist in the public schema — these predate the AgentHub architecture and appear to be legacy tables from the pre-v4 ever-boarding AI system. They are not referenced in the current `src/agents/` code.

---

## 6. Learning Engine Reality vs Claim

### CLAUDE.md §6.2 claim

"LearningEngine subscribes to agenthub.outcome_signal events cross-module."

### v1.0 claim

Same — EventBus subscription.

### Reality

`LearningEngine.ts` does not subscribe to an EventBus. It exposes `receiveOutcomeSignal(signal: OutcomeSignal)` which is called directly. The engine buffers signals in memory (`outcomeBuffer: OutcomeSignal[]`), flushes in batches of 20, on a 30-second interval timer.

Signal processing does two things only:
1. If `signal.agent_name` is set, check advancement eligibility (throttled to ~10% of signals via modulo gate).
2. If `outcome_positive === false`, call `detectOverridePattern()` which queries `OpMemoryQueryService.getOverridePatterns()` over 7 days.

There is a separate `src/agents/learning/LearningEngine.ts` (different from `src/agents/hub/LearningEngine.ts`). The learning directory also contains `AutomationLearningAdapter.ts`. This creates a dual-implementation situation: two files named LearningEngine.ts at different paths. The hub version is the one wired into AgentHub; the learning/ version's wiring status is unclear without deeper inspection.

The `ai_agent_learning_data` table in the DB is not referenced by the hub LearningEngine — it may belong to the older `src/agents/learning/` implementation.

Prompt template performance tracking (described in CLAUDE.md §6.3 as `getPromptTemplatePerformance()`) is defined in `OpMemoryQueryService` but the LearningEngine does not currently call it. This capability exists but is not wired into the learning loop.

---

## 7. Mode 2 Readiness

### Infrastructure: Present

- `agent_mode2_sla_tracking` table exists with `deadline_at`, `breach_action`, `met_at`, `breached` columns
- `Mode2FailureHandler.ts` is fully implemented: handles circuit_open, retryable failures (up to 2 retries), safe_default lookup from `agent_module_registry.mode2_safe_default`, human queue fallback, and escalation to sales_ops
- `check-mode2-slas` edge function exists (scanned in functions list)
- `booking_mode` column on `sales_booking` exists with `'direct' | 'marketplace_autonomous'` values
- `marketplace_eligible`, `marketplace_corridor_id`, `marketplace_floor_price` columns on `sales_frame` are defined in CLAUDE.md §6.13a
- All Mode 2 agent autonomy checks are in place in `AIEnhancedAgent.ts`

### Infrastructure: Missing or Not Ready

- No live Mode 2 traffic. All agents are at `approval_required` — Mode 2 requires agents at L4 (`full`). The minimum advancement timeline is 180+ days. Mode 2 cannot be enabled until at least one qualifying agent earns `full` tier through production track record.
- The client self-qualification flow (LeadScoringAgent L4, KYB integration, T&Cs, payment processing) is architecturally defined in §6.13b but no implementation files for it were found in `src/agents/`.
- `marketplace_transaction` and `marketplace_settlement_batch` tables are defined in §6.13c but were not found in the REST API table list or migration scan. The settlement layer is not yet built.
- `PricingCorridors` UI exists in the Sales rate cards screen but corridor enforcement at the Mode 2 execution layer is not confirmed in the codebase.
- The `ai-autopilot` and `ai-execution-pipeline` edge functions described in v1.0 as Mode 2 execution infrastructure do not exist.

Mode 2 readiness verdict: Infrastructure scaffolding is present. End-to-end autonomous transaction capability is not present. Mode 2 requires: agent advancement to L4 (180+ days minimum), settlement layer build, KYB integration, and payment provider integration. Realistic timeline from cold start: 9–12 months.

---

## 8. Production Readiness Score: 6/10

### Scoring rationale

| Dimension | Score | Notes |
|---|---|---|
| Architecture soundness | 8/10 | Five-layer design is coherent. Schema isolation (ooh_agents) is correct. Circuit breaker, entity lock, dependency graph all implemented. |
| Specialist agent coverage | 7/10 | 16 Sales specialists with versioned prompts are production-quality. 5 CMS + 3 DMS + 3 Ever-boarding agents are registered in DB but lack TypeScript implementations. |
| OpMemory integrity | 9/10 | DB-level immutability trigger, RLS, causal chain model — strongest part of the system. |
| Learning loop | 4/10 | Real-time signal buffering works. Advancement checker works. EventBus wiring claimed in docs does not match direct-call reality. Dual LearningEngine files create ambiguity. Prompt template performance loop not wired. |
| Autonomy progression | 3/10 | All agents at L1. 180-day requirement correct but means zero agents advance until late 2026 from production launch. No live outcomes to learn from yet. |
| Mode 2 | 2/10 | Scaffolding present. Settlement, KYB, payment, L4 advancement all absent. Not launchable. |
| Edge function coverage | 5/10 | `process-agent-events`, `process-agent-tasks`, `check-agent-advancement`, `check-mode2-slas` confirmed. Several functions described in v1.0 do not exist. |
| Legacy table hygiene | 4/10 | Seven `ai_*` tables from pre-v4 system remain in public schema. Risk of confusion and orphaned writes. |

Overall: 6/10

The system is production-ready for Mode 1 (human-in-loop) agent workflows with the 16 Sales specialists. It is not production-ready for autonomous operation (Mode 2), and the learning loop has documentation/reality gaps that need resolving before the autonomy advancement system can be trusted.

---

## 9. Recommended Actions (Priority Order)

1. Correct CLAUDE.md §7.4 — the claim that LearningEngine subscribes via EventBus is incorrect. Document the direct-call pattern and clarify which LearningEngine file is canonical (hub vs learning/).
2. Audit and deprecate `ai_*` legacy tables — confirm none are written to by active code, then drop or archive.
3. Wire prompt template performance tracking into LearningEngine — `getPromptTemplatePerformance()` exists in OpMemoryQueryService but is unused.
4. Implement the 5 CMS + 3 DMS + 3 Ever-boarding agent TypeScript files — they are registered in the DB with no backing code.
5. Build settlement layer (`marketplace_transaction`, `marketplace_settlement_batch`) as a prerequisite for Mode 2 planning.
6. Add idempotency guard on Mode2FailureHandler cancellation events (flagged in v1.0 technical debt, still open).

---

*Audit completed 2026-03-25 by Kai. Codebase ref: `/workspace/group/everboarding/`. Do not push to GitHub.*
