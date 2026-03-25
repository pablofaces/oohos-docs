# Portal Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** ~85% Complete — Active Development  
**Source paths audited:** `src/portal/`, `src/portal/README.md`, `CLAUDE.md §9`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Portal |
| DB schema | No dedicated schema — reads via Sales API + Ever-boarding API |
| Frontend path | `src/portal/` |
| Package path | `packages/portal/` (future extraction planned) |
| Owner | Horizontal (client-facing wrapper) |
| Phase | ~85% complete — some pages active, others stubs |

---

## 2. Current State Summary

The Portal is the primary client-facing interface of OOH OS. It serves both:
1. **Onboarded clients** — view proposals, sign booking orders, upload creatives, track delivery, download invoices
2. **Council users** — manage site request approvals (delegated via `apps/council-portal/` fork)

The Portal is **not** a standalone backend — it reads all commercial data through the Sales API and delegates onboarding progression to the Ever-boarding module. It has no dedicated `ooh_portal` schema; portal-state is stored in `ever_boarding` and `ooh_sales` tables.

**What works today:**
- Dashboard with onboarding progress, proposal status, campaign status tiles
- Campaign list and detail (booking status, creative upload, proof of play)
- Documents hub (proposals, booking orders, invoices, creatives)
- Financials (invoice list, payment status, DMS document display)
- Signatures (DocuSign/PandaDoc embedded signing flow)
- Support (ticket creation, messaging thread)
- User/team management (invite, role assignment, sub-role management)
- Compliance tasks (scoped per user role)
- Notifications hub (real-time + digest preferences)
- Brands management (logo, brand guidelines upload)
- Credit request workflow (CreditRequestPage — 22KB, most complex page)
- Ever-boarding integration (everboardingIntegration.ts, partnerEverboardingIntegration.ts)
- Council portal routes (`/portal/councils/*`)

**What is stubbed / incomplete:**
- Proposal viewer (interactive map) — partially implemented
- Real-time delivery report (polling only, no WebSocket)
- ResolveBlockersPage — exists, partial implementation
- MoneyPage — separate from FinancialsPage, scope overlap under review

**What is not yet built:**
- Agency portal (separate user type — agency_user reads multiple clients' data)
- Programmatic campaign view
- Marketplace self-service booking flow (Phase 3b)
- "Run again" brief self-service

---

## 3. Client Journey

### New Onboarding Client
1. Receives magic link invitation (from Ever-boarding) → lands on `/portal`
2. Sees `OnboardingStatusPage` with FM checklist progress
3. Completes required steps (KYB docs, GDPR consent, contact verification)
4. `ResolveBlockersPage` surfaces any gated items (credit, admin blocks)
5. Once `onboarding_status = complete`, commercial activity unlocked

### Active Client
1. Dashboard → sees open proposals, live campaigns, pending actions
2. Campaigns → reviews booking detail, upload creative per format spec
3. Documents → downloads proposals, booking orders, invoices
4. Financials → views invoice + payment status, uploads PO/IO
5. Signatures → embedded signing for booking orders (e-sig widget)
6. Support → raises ticket or messages sales rep via Comms Hub
7. Settings → manages team members, roles, notifications

### Partner/Agency Client
- `partnerEverboardingIntegration.ts` handles partner-specific onboarding flow
- `PartnerCompliancePage` — compliance tasks scoped for partner role
- Agency view (multi-client) — planned but not yet built

### Council User
- `/portal/councils/*` routes via `CouncilDashboard`
- Views site requests, pending signatures, application approvals
- Separate fork available: `apps/council-portal/`

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| Dashboard (`DashboardPage`) | ✅ Built | 9.6KB — onboarding tiles, campaign summary, proposal status |
| Campaign list + detail (`CampaignsPage`) | ✅ Built | 13.9KB — creative upload, proof of play view |
| Documents hub (`DocumentsPage`) | ✅ Built | 6.9KB — proposal/invoice/creative downloads |
| Financials (`FinancialsPage`) | ✅ Built | 13.1KB — invoice table, PO/IO upload, DMS-backed |
| Signatures (`SignaturesPage`) | ✅ Built | 11.7KB — embedded e-sig flow |
| Support (`SupportPage`) | ✅ Built | 9.3KB — ticket + messaging |
| User/team management (`UsersPage`) | ✅ Built | 12.6KB — invite, roles, sub-roles |
| Onboarding status (`OnboardingStatusPage`) | ✅ Built | 12.4KB — FM checklist, progress tracking |
| Compliance tasks (`CompliancePage`) | ✅ Built | 13.1KB — scoped per role |
| Notifications hub (`NotificationsPage`) | ✅ Built | 16.2KB — real-time + digest |
| Brands management (`BrandsPage`) | ✅ Built | 3.5KB — logo + brand guidelines |
| Credit request (`CreditRequestPage`) | ✅ Built | 22.4KB — most complex page |
| Profile settings (`ProfileSettingsPage`) | ✅ Built | 4.7KB |
| Security settings (`SecuritySettingsPage`) | ✅ Built | 3.8KB |
| Tasks settings (`TasksSettingsPage`) | ✅ Built | 18.4KB |
| Account page (`AccountPage`) | ✅ Built | 904 bytes — thin wrapper |
| Messages (`MessagesPage`) | ✅ Built | 10.3KB — Comms Hub thread view |
| Partner compliance (`PartnerCompliancePage`) | ✅ Built | 11.8KB |
| Council dashboard (`CouncilDashboard`) | ✅ Built | 9.2KB |
| Billing (`BillingPage`) | ✅ Built | 7.4KB |
| Me/settings (`MeSettingsPage`) | ✅ Built | 11.9KB |
| Resolve blockers (`ResolveBlockersPage`) | 🟡 Partial | 9.0KB — partial implementation |
| Money page (`MoneyPage`) | 🟡 Partial | 12.8KB — overlap with FinancialsPage under review |
| Notification settings (`NotificationSettingsPage`) | 🟡 Stub | 554 bytes — thin stub |
| Proposal viewer (interactive) | 🟡 Partial | Map view partial; full interactive not complete |
| Agency portal (multi-client view) | ❌ Missing | Designed, not built |
| Marketplace self-service booking | ❌ Missing | Phase 3b |
| "Run again" brief | ❌ Missing | Roadmap |
| Programmatic campaign view | ❌ Missing | No SSP win/bid view in portal yet |
| Real-time delivery WebSocket | ❌ Missing | Polling only |

---

## 5. Schema Detail

The Portal has **no dedicated database schema**. All data is sourced via:

### Primary data sources
| Source | Mechanism | Data fetched |
|---|---|---|
| Sales API | REST API calls via `api/client.ts` | Proposals, bookings, invoices, delivery summaries, creatives |
| Ever-boarding API | `everboardingIntegration.ts` | Onboarding status, FM checklist, blocker items |
| DMS API | Via Sales events | Document URLs for proposals, invoices, PoP reports |
| Comms Hub API | Via `useMessaging` hook | Message threads |
| Audience API | Via Sales API (embedded in proposal data) | Reach estimates on proposal viewer |

### Portal-relevant tables (owned by other modules)
| Table | Owner | Used by Portal for |
|---|---|---|
| `sales_proposal` | Sales | Proposal list, status, PDF link |
| `sales_booking` | Sales | Campaign list, status badges |
| `sales_delivery_summary` | Sales | Proof of play dashboard |
| `sales_invoice` | Sales | Invoice table in Financials |
| `ever_boarding_progress` | Ever-boarding | OnboardingStatusPage FM steps |
| `hub_message_logs` | Comms Hub | Messages page threads |
| `compliance_tasks` | Compliance | CompliancePage task list |
| `dms_documents` | DMS | Document downloads |

### Portal-maintained state
The Portal writes via API calls — it never writes directly to other module schemas. Key write operations:
- `sales.proposals.recordView(proposal_id)` → updates `portal_viewed_at`
- `sales.proposals.accept()` / `sales.proposals.reject()` → proposal state changes
- `sales.creative.upload()` → creative upload + compliance trigger
- `sales.bookings.requestChange()` → change request creation
- `everboarding.uploadDocument()` → FM document upload

---

## 6. Role Model

| Role | Portal Access |
|---|---|
| `portal_client` | Own account: proposals, bookings, invoices, creatives, support |
| `portal_admin` | Full company view: all users, billing, credit, all documents |
| `portal_contact` | Limited: view proposals + documents assigned to them |
| `agency_user` | Introduced clients' data only (planned — not fully built) |
| `council_admin` | Council portal routes: site request review, pending signatures |
| `service_provider` | Provider portal: assigned surveys/deliverables |

Role management implemented in `usePortalRoles.ts` (14.8KB — most complex hook).  
Sub-roles handled via `useSubRoleMembers.ts` (7.0KB).

---

## 7. UI Screens

### Core Navigation (5 pillars)

| Pillar | Route | Primary Page | Status |
|---|---|---|---|
| Dashboard | `/portal` | `DashboardPage` | ✅ |
| Campaigns | `/portal/campaigns` | `CampaignsPage` | ✅ |
| Documents | `/portal/documents` | `DocumentsPage` | ✅ |
| Financials | `/portal/financials` | `FinancialsPage` | ✅ |
| Support | `/portal/support` | `SupportPage` | ✅ |

### Secondary Routes

| Route | Page | Status |
|---|---|---|
| `/portal/onboarding` | `OnboardingStatusPage` | ✅ |
| `/portal/signatures` | `SignaturesPage` | ✅ |
| `/portal/users` | `UsersPage` | ✅ |
| `/portal/compliance` | `CompliancePage` | ✅ |
| `/portal/notifications` | `NotificationsPage` | ✅ |
| `/portal/brands` | `BrandsPage` | ✅ |
| `/portal/credit` | `CreditRequestPage` | ✅ |
| `/portal/billing` | `BillingPage` | ✅ |
| `/portal/messages` | `MessagesPage` | ✅ |
| `/portal/settings/me` | `MeSettingsPage` | ✅ |
| `/portal/settings/profile` | `ProfileSettingsPage` | ✅ |
| `/portal/settings/security` | `SecuritySettingsPage` | ✅ |
| `/portal/settings/tasks` | `TasksSettingsPage` | ✅ |
| `/portal/settings/notifications` | `NotificationSettingsPage` | 🟡 Stub |
| `/portal/resolve` | `ResolveBlockersPage` | 🟡 Partial |
| `/portal/money` | `MoneyPage` | 🟡 Partial |
| `/portal/partner-compliance` | `PartnerCompliancePage` | ✅ |
| `/portal/councils/*` | `CouncilDashboard` | ✅ |
| `/portal/account` | `AccountPage` | ✅ |

### Provider tree
`PortalProviders.tsx` (757 bytes) — wraps all portal routes with context providers.  
`providers/` subdirectory — additional scoped providers.  
`contexts/` subdirectory — portal-level React contexts.

### Components structure
16 component groups under `src/portal/components/`:
`account/`, `billing/`, `blockers/`, `brands/`, `campaigns/`, `dashboard/`, `documents/`, `financials/`, `help/`, `layout/`, `messaging/`, `profile/`, `settings/`, `support/`, `ui/`, `users/`

---

## 8. Hooks Reference

| Hook | Size | Purpose |
|---|---|---|
| `usePortalRoles.ts` | 14.8KB | Role resolution, permission checks, sub-role management |
| `usePortalTeam.ts` | 17.4KB | Team members, invitations, role assignments |
| `useMessaging.ts` | 18.1KB | Comms Hub message thread management |
| `useNotificationPreferences.ts` | 8.1KB | Notification channel + digest preferences |
| `useFinancialDocumentNotifications.ts` | 8.6KB | Real-time financial document status |
| `usePortalProfile.ts` | 8.0KB | User profile data + updates |
| `usePortalBrands.ts` | 7.3KB | Brand asset management |
| `useSubRoleMembers.ts` | 7.0KB | Sub-role membership management |
| `usePortalOnboardingStatus.ts` | 5.5KB | FM onboarding progress |
| `useScopedComplianceTasks.ts` | 5.8KB | Compliance tasks per role scope |
| `useDelegationTracking.ts` | 4.5KB | Delegation chain (Ever-boarding) |
| `useSignatureProfile.ts` | 5.3KB | Signatory profile for e-sig flow |
| `usePortalSignatures.ts` | 4.2KB | Signature request list + status |
| `useFinancialDocuments.ts` | 4.5KB | DMS financial document list |
| `useAccountInfo.ts` | 5.3KB | Account header data |
| `useBlockerResolution.ts` | 4.6KB | Blocker item resolution workflow |
| `useAccountCreditContext.ts` | 3.0KB | Credit status for gating |
| `useCreditStatus.ts` | 2.7KB | Credit limit / balance |
| `usePortalBookingStatus.ts` | 2.4KB | Booking status tiles for dashboard |
| `useOnboardingProgress.ts` | 6.7KB | FM checklist progress |
| `usePortalUsers.ts` | 5.0KB | User list for admin |
| `useCouncilSiteRequests.ts` | 2.9KB | Council site request list |
| `useCouncilPendingSignatures.ts` | 2.5KB | Council pending signature queue |
| `useRoleRequirement.ts` | 3.1KB | Route-level role requirement checks |

---

## 9. EventBus Surface

The Portal does not emit EventBus events directly. All writes are via Sales API calls which trigger EventBus events server-side.

### Events consumed by Portal (via Sales API updates / polling)

| Event | Source | Portal response |
|---|---|---|
| `sales.proposal.sent` | Sales | Surfaces proposal in Dashboard + Documents |
| `sales.booking.confirmed` | Sales | Updates Campaign status badge |
| `sales.booking.live` | Sales | Updates Campaign to "Live" |
| `sales.booking.completed` | Sales | Updates Campaign to "Completed", shows PoP link |
| `sales.invoice.issued` | Sales | Surfaces invoice in Financials |
| `sales.delivery.report_ready` | Sales | Surfaces delivery report download |
| `sales.proposal.availability_stale` | Sales | Highlights unavailable frames on proposal viewer |
| `sales.proposal.substitutes_offered` | Sales | Shows alternative frame suggestions |
| `sales.campaign.attribution_ready` | Sales | Shows post-campaign attribution report |
| `everboarding.onboarding.complete` | Ever-boarding | Unlocks commercial navigation items |
| `everboarding.onboarding.stalled` | Ever-boarding | Shows blocker banner |

**Agent in Portal:** `PortalAgentAdapter.ts` (8.4KB) — bridges portal actions to AgentHub specialists. Located in `src/portal/agents/`.

---

## 10. External Integrations

| Integration | Direction | Mechanism | Status |
|---|---|---|---|
| Sales module | Bidirectional | REST API (`api/client.ts`) | ✅ Live |
| Ever-boarding | Read + write | `everboardingIntegration.ts` service | ✅ Live |
| Partners ever-boarding | Read + write | `partnerEverboardingIntegration.ts` | ✅ Live |
| Comms Hub | Read | `useMessaging` hook | ✅ Live |
| DMS | Read | Via Sales API document URLs | ✅ Live |
| E-signatures | Embed | Widget in `SignaturesPage` | ✅ Live (stub provider) |
| Compliance | Read | `useScopedComplianceTasks` | ✅ Live |
| AgentHub | Via `PortalAgentAdapter` | `specialists/` agents | ✅ Live |
| Council portal fork | Shared routes | `/portal/councils/*` | ✅ Live |
| `apps/council-portal/` | Fork | Separate Vite app | ✅ Deployable |

---

## 11. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| `MoneyPage` and `FinancialsPage` scope overlap | High | Two pages covering financial data — consolidation needed. `MoneyPage` (12.8KB) responsibility unclear. |
| `NotificationSettingsPage` is a 554-byte stub | Medium | Non-functional notification settings exposed in nav |
| `ResolveBlockersPage` partially implemented | Medium | Blocker resolution UX incomplete |
| No WebSocket/Realtime on delivery reporting | Medium | Portal polls for proof of play updates — should use Supabase Realtime |
| Agency multi-client view not built | Medium | `agency_user` role defined but UI not implemented |
| Portal has no dedicated audit service integration | Medium | `auditService.ts` exists (5.0KB) but coverage unclear |
| `PortalProviders.tsx` is only 757 bytes | Low | May need expansion as portal state grows |
| `AccountPage.tsx` is 904 bytes (thin wrapper) | Low | May be redundant with Dashboard tile data |
| No error boundaries on portal routes | Low | Individual page failures could crash entire portal shell |
| E-signature provider not confirmed (DocuSign vs PandaDoc) | Low | Stub in place — tenant-level config needed before go-live |
| Council portal routes live inside main portal bundle | Low | `/portal/councils/*` increases bundle size for non-council clients |
| `TasksSettingsPage` is 18.4KB | Low | Largest settings page — may need splitting |
| Interactive proposal map on Portal viewer not complete | Low | Client sees proposal without full frame map experience |
| Programmatic deal view not surfaced | Low | No SSP win visibility for clients in programmatic campaigns |

---

*End of Portal Module Audit v1.0*
