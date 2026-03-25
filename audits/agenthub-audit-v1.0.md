# AgentHub + Operational Memory Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** v4.0 — Production Ready (all modules wired)  
**Source paths audited:** `src/agents/`, `src/agents/hub/`, `src/agents/specialists/`, `supabase/functions/ai-*/`, `supabase/functions/process-agent-*/`, `CLAUDE.md §§7, 7.3, 18.1 AI Agents`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | AgentHub + Operational Memory |
| DB schema | `ooh_sales` (operational_memory table owned by Sales schema) |
| Backend path | `src/agents/` |
| Edge Functions | `ai-gateway`, `ai-autopilot`, `ai-execution-pipeline`, `process-agent-events`, `process-agent-tasks`, `check-agent-advancement`, `onboarding-intelligence`, `auto-resolve-exceptions` |
| Owner | Horizontal (AI orchestration layer) |
| Phase | v4.0 — 5-layer architecture complete, 25+ agents registered |

---

## 2. Current State Summary

AgentHub is the sole AI orchestration layer for OOH OS. **No module calls the Claude API directly.** All AI functionality is dispatched through `agentHub.dispatch({task, module, context, mode})`. This enforces:
- Unified agent governance (autonomy ladder, circuit breakers, policy engine)
- Operational memory — every agent action is logged with reasoning + confidence
- Trust progression — agents earn autonomy through track record, not configuration

**Architecture (5 layers):**
1. **Dispatch layer** — `AgentHub.ts` (20.4KB) — routes tasks to specialist agents
2. **Policy layer** — `PolicyEngine.ts` (11.7KB) — evaluates autonomy level, circuit breakers, mode overrides
3. **Execution layer** — `ai-execution-pipeline` Edge Function — calls Claude API, applies prompt templates
4. **Memory layer** — `OpMemoryQueryService.ts` (6.7KB) — reads/writes `sales_operational_memory`
5. **Learning layer** — `LearningEngine.ts` (6.3KB) — processes outcome signals, updates agent models

**Core services in `src/agents/hub/`:**
- `AgentHub.ts` (20.4KB) — main dispatch hub
- `PolicyEngine.ts` (11.7KB) — autonomy level + circuit breaker evaluation
- `LearningEngine.ts` (6.3KB) — outcome signal processing
- `Mode2FailureHandler.ts` (8.2KB) — autonomous failure recovery (booking cancellation, refunds)
- `EntityLockManager.ts` (5.6KB) — preemption + resume for concurrent agent tasks
- `CircuitBreakerRegistry.ts` (6.6KB) — per-agent circuit breakers
- `DependencyGraphEngine.ts` (5.4KB) — multi-agent task dependency resolution
- `AutonomyAdvancementChecker.ts` (6.9KB) — checks 180-day track record for L → L+1 promotions
- `TaskQueue.ts` (7.8KB) — async task queue with priority
- `EventBus.ts` (8.2KB) — internal EventBus (NOT same as platform EventBus)
- `OpMemoryQueryService.ts` (6.7KB) — Operational Memory read/write
- `AdapterRegistry.ts` (6.2KB) — module adapter registration

---

## 3. Autonomy Ladder (5 tiers)

| Level | Name | Behaviour | Human gate |
|---|---|---|---|
| L1 | `recommends` | Returns recommendation only — human must act | Always |
| L2 | `acts_and_notifies` | Takes action + notifies human immediately | No (but notifies) |
| L3 | `acts_silently` | Takes action with no notification | No |
| L4 | `autonomous` | Full autonomous operation, no human loop | No |
| L5 | `approval_required` | Override — forces approval regardless of agent level | Always |

**Default:** All new agents start at L1 (`recommends`). Promotion to L2+ requires:
- 180+ days continuous track record
- `AutonomyAdvancementChecker` validates: win rate, override rate, confidence calibration, no major errors
- Proposed via `agenthub.autonomy_advancement_proposed` event → human manager reviews

**Mode 2 rule:** Agents operating in `mode_2` (fully autonomous bookings) must be at L4. No self-escalation.

---

## 4. Registered Agents

### Sales Module (18 agents)

| Agent | Default Level | Key task |
|---|---|---|
| `ProposalAgent` | L1 | Generate draft proposal from brief, reach estimates, pricing |
| `LeadScoringAgent` | L2 | Score + qualify inbound leads |
| `YieldAgent` | L2→full | Floor price adjustment recommendations |
| `FrameSubstituteAgent` | L1 | Score substitute frames (location 30, format 25, market 20, price 15) |
| `CreativeAdvisorAgent` | L2 | Creative quality + compliance validation |
| `DeliveryOptimisationAgent` | L2 | Under-delivery prevention, make-good scheduling |
| `DiscountApprovalAgent` | L1 | Discount justification + approval routing |
| `AccountHealthAgent` | L1 | Account health scoring + risk flags |
| `AvailabilityForecastAgent` | L1 | Fill rate forecasting per frame/market |
| `ProgrammaticAgent` | full | Evaluate SSP bids, check availability, respond |
| `EnquiryAgent` | L2 | Inbound WhatsApp → create `sales_lead` |
| `FollowUpAgent` | L1 | Draft follow-up on stale proposals |
| `NegotiationAgent` | L1 | Negotiation guidance within commercial guardrails |
| `RenewalAgent` | L1 | Assess account, draft renewal brief + outreach |
| `AttributionAgent` | L2 | Post-campaign attribution import + analysis |
| `WinLossAgent` | L1 | Win/loss pattern analysis + coaching |
| `OnboardingAnalysisAgent` | L2 | `OnboardingAnalysisAgent.ts` (12.5KB) — onboarding health |
| `ReminderTimingAgent` | L2 | `ReminderTimingAgent.ts` (10.7KB) — optimal reminder scheduling |

### CMMS Module (7 agents)
`TriageAgent`, `DispatchAgent`, `PredictiveMaintenanceAgent`, `WeatherRiskAgent`, `CampaignImpactAgent`, `VerificationAgent`, `ReportingAgent` — see CMMS audit for detail.

### DMS Module
`IngestionAgent` — document classification, PO number extraction

### Comms Hub Module
`EnquiryAgent` (shared) — WhatsApp inbound → Sales lead creation

### Portal Module
Portal-specific agents via `PortalAgentAdapter.ts` — specialists in `src/portal/agents/specialists/`

---

## 5. Operational Memory

**Table:** `sales_operational_memory` (owned by Sales schema, written by AgentHub via Sales API)  
**Rules:** Append-only — never update, never delete. Every agent action creates a new row.

| Column | Purpose |
|---|---|
| `id` | UUID PK |
| `tenant_id` | Tenant scoping |
| `agent_name` | Which agent wrote this entry |
| `task_type` | Task classification |
| `entity_type` | What entity was acted on (proposal, booking, lead, etc.) |
| `entity_id` | UUID of the entity |
| `action_taken` | What the agent did |
| `reasoning` | Agent's reasoning (from LLM response) |
| `confidence` | 0–1 confidence score |
| `outcome` | Result of the action |
| `human_override` | Was this overridden by a human? |
| `override_reason` | Why human overrode (feeds `LearningEngine`) |
| `correlation_id` | Links related entries across a multi-step task |
| `created_at` | Timestamp |

**`OpMemoryQueryService`** provides:
- `getRecentActionsForEntity(entityId)` — context for follow-on agent tasks
- `getAgentTrackRecord(agentName, days)` — feeds `AutonomyAdvancementChecker`
- `getOverridePatterns(agentName)` — identifies systematic biases for learning

**AgentHub data ownership rule:** AgentHub never has direct DB access to the Sales schema. All writes to `sales_operational_memory` go through the Sales API endpoint.

---

## 6. Prompt Template Architecture

Templates stored in `src/agents/templates/`:
- Tenant-level overrides via `tenant_prompt_overrides` table
- `ContextBuilder` assembles: tenant context + entity data + operational memory entries → prompt
- Base template + tenant overlay → merged prompt sent to Claude API
- Response parsed via structured output schemas

**AI models:**
- Standard: `claude-sonnet-4-6` — used for all L1/L2 tasks
- Complex: `claude-opus` — used for multi-step L3/L4 tasks (e.g., full autonomous booking flow)

---

## 7. Circuit Breakers

`CircuitBreakerRegistry.ts` — per-agent circuit breakers:

| State | Condition | Behaviour |
|---|---|---|
| `closed` | Normal operation | Requests pass through |
| `open` | >threshold failures in window | All requests blocked → `Mode2FailureHandler` invoked |
| `half-open` | After cooldown period | Single test request allowed |

On circuit open: `Mode2FailureHandler` triggers:
- `agenthub.booking_cancellation_required` → Sales + Marketplace
- `agenthub.refund_required` → Marketplace
- `agenthub.operator_queue_item_created` → Comms Hub (human queue)

---

## 8. Edge Functions

| Function | Purpose |
|---|---|
| `ai-gateway` | Entry point for all AI requests from frontend — routes to correct agent |
| `ai-autopilot` | Mode 2 autonomous execution pipeline |
| `ai-execution-pipeline` | Core Claude API call wrapper — applies prompt templates, logs to OpMemory |
| `process-agent-events` | EventBus consumer for agent-triggering events |
| `process-agent-tasks` | Async task queue processor |
| `check-agent-advancement` | Scheduled — evaluates agents for autonomy advancement |
| `onboarding-intelligence` | Onboarding-specific AI analysis |
| `auto-resolve-exceptions` | Attempts autonomous resolution of Mode 2 failures before human queue |

---

## 9. EventBus Surface (Platform-Level)

### Outbound Events (AgentHub → Platform)

| Event | Trigger | Consumer(s) |
|---|---|---|
| `agenthub.outcome_signal` | Human approve/reject/override | LearningEngine |
| `agenthub.booking_cancellation_required` | Mode 2 circuit open failure | Sales, Marketplace |
| `agenthub.refund_required` | Mode 2 circuit open failure | Marketplace |
| `agenthub.operator_queue_item_created` | Mode 2 failure → human queue | Comms Hub |
| `agenthub.agent_preempted` | EntityLockManager preempts agent | Affected agent |
| `agenthub.agent_resume` | EntityLockManager resumes agent | Affected agent |
| `agenthub.autonomy_advancement_proposed` | AutonomyAdvancementChecker approves | Manager UI |

---

## 10. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| `sales_operational_memory` lives in Sales schema | Medium | AgentHub must route all writes via Sales API — creates indirect coupling. Future: dedicated `agenthub_operational_memory` schema |
| All agents default to L1 | Medium | No agent has yet earned L2+ promotion in production — 180-day track record requirement means L1 default is correct but limits AI utility |
| `DependencyGraphEngine` not yet tested for multi-agent deadlock | Medium | Circular dependencies between agents with mutual entity locks could deadlock |
| Prompt templates stored in filesystem (`src/agents/templates/`) | Low | Templates should be in DB for per-tenant override without code deployment |
| `ai-execution-pipeline` has no per-tenant rate limiting | Low | All tenants share Claude API quota — high-volume tenant could throttle others |
| Circuit breaker thresholds are not tenant-configurable | Low | Should live in `tenant_business_rules` |
| `Mode2FailureHandler` cancellation events have no idempotency guard | Low | If `booking_cancellation_required` is re-emitted, Sales must deduplicate |
| Claude model selection (sonnet vs opus) is hardcoded per task type | Low | Should be configurable per agent + tenant |

---

*End of AgentHub + Operational Memory Audit v1.0*
