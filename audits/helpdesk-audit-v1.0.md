# Help Desk Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Core Ticketing Live — AI Triage Wired  
**Source paths audited:** `supabase/functions/api-helpdesk/`, `CLAUDE.md §§7.3 Help Desk agents, 18.1 Help Desk`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Help Desk |
| DB schema | `public` (helpdesk_tickets, related tables) |
| Backend path | `supabase/functions/api-helpdesk/` |
| Frontend path | `src/portal/pages/SupportPage.tsx` (client-facing), Admin help desk view |
| Owner | Horizontal |
| Phase | Core ticketing live; AI triage built |

---

## 2. Current State Summary

The Help Desk module is the unified support layer for OOH OS. It handles tickets raised by clients via the Portal and by Sales reps on behalf of clients, with direct context linking to Sales entities (booking_id, invoice_id, proposal_id).

Every ticket carries a Sales entity reference so support agents have immediate context — no separate context lookup needed.

**What is built:**
- `api-helpdesk` Edge Function — ticket CRUD + assignment
- Client-facing `SupportPage.tsx` in Portal — ticket creation + messaging thread
- Admin help desk view — ticket queue + assignment
- Sales EventBus integration (disputes, availability errors, invoice discrepancies, delivery shortfalls)
- AI triage via AgentHub (help desk agent classification + routing)

---

## 3. Ticket Lifecycle

1. Client raises ticket via Portal `SupportPage` (or Sales rep raises on their behalf)
2. Ticket created with entity reference (`booking_id` / `invoice_id` / `proposal_id`)
3. AgentHub triage: auto-classify ticket type, suggest priority, route to correct team
4. Ticket assigned to support rep
5. Rep + client communicate via threaded messages (Comms Hub message thread linked to ticket)
6. Resolution recorded — entity reference updated (e.g., delivery shortfall → `sales_make_good` created)
7. Ticket closed + satisfaction survey (planned)

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| `api-helpdesk` Edge Function | ✅ Built | Ticket CRUD |
| `SupportPage.tsx` in Portal | ✅ Built | 9.3KB — client-facing ticket creation + thread |
| Admin ticket queue | ✅ Built | Assignment + status management |
| Sales entity context linking | ✅ Built | Every ticket carries Sales entity reference |
| AI triage classification | ✅ Built | AgentHub routes and classifies |
| Comms Hub integration | ✅ Built | Message threads linked to tickets |
| Satisfaction surveys | ❌ Missing | Post-resolution CSAT not built |
| SLA tracking dashboard | ❌ Missing | No SLA breach monitoring UI |
| Escalation workflow | ❌ Missing | No formal escalation path (rep → manager) |
| Knowledge base | ❌ Missing | No self-service article library |
| Ticket merge / dedup | ❌ Missing | Duplicate ticket detection not built |

---

## 5. Schema Detail

**Multi-tenancy:** All tables include `tenant_id` + RLS enforced

| Table | Purpose |
|---|---|
| `helpdesk_tickets` | Master ticket record — `id, tenant_id, subject, status, priority, entity_type, entity_id, account_id, raised_by, assigned_to` |
| `helpdesk_ticket_messages` | Threaded messages per ticket |
| `helpdesk_ticket_events` | Audit log of ticket state changes |
| `helpdesk_sla_config` | SLA definitions per ticket priority (tenant-configurable) |

---

## 6. EventBus Surface

### Inbound Events (Sales → Help Desk)

| Event | Source | Help Desk action |
|---|---|---|
| `sales.booking.disputed` | Sales | Creates dispute ticket with `booking_id` context |
| `sales.availability.error` | Sales | Creates availability error ticket with `frame_id` context |
| `sales.invoice.disputed` | Sales | Creates invoice discrepancy ticket |
| `sales.delivery.shortfall` | Sales | Creates delivery shortfall ticket — triggers make-good assessment |

All inbound events include the relevant Sales entity reference — tickets are pre-populated with context.

---

## 7. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| No SLA tracking dashboard | Medium | Operators cannot monitor response time SLAs |
| No escalation path | Medium | If rep cannot resolve, no formal escalation workflow |
| Satisfaction surveys missing | Low | No CSAT data to measure support quality |
| Knowledge base missing | Low | Clients must raise tickets for issues that could be self-served |
| Ticket deduplication missing | Low | Duplicate tickets from same client for same issue not detected |

---

*End of Help Desk Module Audit v1.0*
