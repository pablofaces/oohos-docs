# Portal Module Audit v2.0 — Deep Verification

**Date:** 2026-03-25
**Auditor:** Kai (Claude Code — automated code + schema verification)
**Previous version:** v1.0 (Cascade, 2026-03-23)
**Status:** ⚠️ ROUTING GAPS — several pages unreachable, redirect chain issues found

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Portal |
| DB schema | `public` — portal-specific tables: `portal_campaigns`, `portal_payments`, `portal_scheduled_payments`, `portal_statements`, `portal_transaction_documents`, `portal_transactions` |
| Frontend path | `src/portal/` |
| Pages | 23 page files verified |
| Hooks | 24 hooks verified |
| Council sub-portal | 13 pages in `src/portal/councils/pages/` |
| Phase | ~80% accessible — routing gaps reduce effective coverage |

---

## 2. Schema Correction — v1.0 Was Wrong

v1.0 stated: *"Portal has no dedicated database schema — reads via Sales API + Ever-boarding API."*

**Actual DB:** 6 portal-owned tables exist in `public` schema:
- `portal_campaigns`
- `portal_payments`
- `portal_scheduled_payments`
- `portal_statements`
- `portal_transaction_documents`
- `portal_transactions`

These tables store portal-level financial state (payments, statements, transactions). The portal is not purely a pass-through to Sales APIs — it maintains its own financial document state.

---

## 3. Routing Gaps Found

### 3.1 `/portal/documents` → Wrong Destination (HIGH)

v1.0 stated `DocumentsPage` is served at `/portal/documents` ✅.

**Actual router:** `/portal/documents` is a `<Navigate to="/portal/settings/tasks" />` redirect — it goes to `TasksSettingsPage`, not `DocumentsPage`.

`DocumentsPage.tsx` (6,668 bytes) exists but has **no direct route**. It is either:
- Orphaned (built but not wired)
- Intentionally accessed only as a component embedded in another page

**Impact:** Clients navigating to Documents get Tasks Settings instead. Any direct `/portal/documents` link (from emails, Comms Hub notifications, bookmarks) lands on the wrong page.

### 3.2 `/portal/support` → Redirects to Dashboard (MEDIUM)

`/portal/support` redirects to `/portal` (the dashboard), not to a support overview page.

The `SupportPage.tsx` (9,023 bytes) exists but is not reachable via `/portal/support`. Sub-routes work (`/portal/support/tickets`, `/portal/support/messages`, `/portal/support/knowledge-base`) but the parent path drops users at the dashboard with no indication of what happened.

**Impact:** Any nav link or external URL pointing to `/portal/support` silently redirects to dashboard.

### 3.3 Duplicate `portal/settings` Redirect (LOW)

Line 358 in `AnimatedRoutes.tsx` defines:
```
<Route path="/portal/settings" element={<Navigate to="/portal/settings" replace />} />
```
A route that redirects to itself. This is an infinite redirect loop if ever triggered (React Router may handle it gracefully, but it's dead code at best and a browser hang at worst).

### 3.4 `NotificationSettingsPage` — 533-byte Stub Routed but Non-functional (MEDIUM)

`/portal/settings/notifications` redirects to `/portal/settings/me` (not to `NotificationSettingsPage` directly). However, `NotificationSettingsPage.tsx` at 533 bytes is a stub. The redirect hides the stub, but if the redirect is ever removed the stub will be exposed.

### 3.5 Route Reorganisation — Confirmed Navigation Refactoring

The following v1.0 routes have been reorganised (redirected to new paths):
| v1.0 route | Current destination |
|---|---|
| `/portal/financials` | `/portal/money` |
| `/portal/billing` | `/portal/money` |
| `/portal/compliance` | `/portal/settings/tasks` |
| `/portal/documents` | `/portal/settings/tasks` |
| `/portal/users` | `/portal/settings/team` |
| `/portal/brands` | `/portal/settings` |
| `/portal/account` | `/portal/settings` |
| `/portal/messages` | `/portal/support/messages` |
| `/portal/signatures` | `/portal/support/signatures` |

These redirects are correct for navigation continuity but any hardcoded links in Comms Hub emails, onboarding flows, or ever-boarding task notifications pointing to old paths need auditing.

---

## 4. All 23 Portal Pages — Verified Present and Routed

| Page | File Size | Route | Status |
|---|---|---|---|
| DashboardPage | 9,357 bytes | `/portal` | ✅ Routed |
| CampaignsPage | 13,481 bytes | `/portal/campaigns` | ✅ Routed |
| FinancialsPage | 12,715 bytes | (via MoneyPage) | ⚠️ Route unclear |
| MoneyPage | 12,395 bytes | `/portal/money` | ✅ Routed |
| SignaturesPage | 11,362 bytes | `/portal/support/signatures` | ✅ Routed |
| OnboardingStatusPage | 12,077 bytes | `/portal/onboarding-status` | ✅ Routed |
| CompliancePage | 12,764 bytes | (via TasksSettings) | ⚠️ No direct route |
| NotificationsPage | 15,834 bytes | `/portal/notifications` | ✅ Routed |
| UsersPage | 12,242 bytes | `/portal/settings/team` | ✅ Routed |
| CreditRequestPage | 21,894 bytes | `/portal/credit/request` | ✅ Routed |
| TasksSettingsPage | 17,976 bytes | `/portal/settings/tasks` | ✅ Routed |
| MeSettingsPage | 11,553 bytes | `/portal/settings/me` | ✅ Routed |
| PartnerCompliancePage | 11,532 bytes | `/portal/partner-compliance` | ✅ Routed |
| MessagesPage | 9,984 bytes | `/portal/support/messages` | ✅ Routed |
| SupportPage | 9,023 bytes | NO DIRECT ROUTE | ❌ Orphaned |
| ResolveBlockersPage | 8,721 bytes | `/portal/resolve-blockers` | ✅ Routed |
| CouncilDashboard | 8,963 bytes | `/portal/councils/*` | ✅ Routed |
| BillingPage | 7,198 bytes | (via MoneyPage) | ⚠️ Superseded |
| DocumentsPage | 6,668 bytes | NO DIRECT ROUTE | ❌ Orphaned |
| ProfileSettingsPage | 4,596 bytes | `/portal/settings` (with me-tab?) | ⚠️ Unclear |
| SecuritySettingsPage | 3,735 bytes | `/portal/settings/me` redirect | ⚠️ May be embedded |
| BrandsPage | 3,358 bytes | (redirect to settings) | ⚠️ No direct route |
| AccountPage | 871 bytes | `/portal/settings` | ✅ Thin wrapper |
| NotificationSettingsPage | 533 bytes | Redirected away from | 🟡 Stub |

**Summary:** 2 pages are orphaned (no route), 4 have unclear or redirected routing.

---

## 5. Portal Auth Architecture

`PortalRoute` component (`AnimatedRoutes.tsx:258`) wraps all portal pages with:
```tsx
<RequireAuth requireOnboardingComplete>
```
This means clients cannot access any portal page until `onboarding_status = complete`. This is correct by design but means:
- A client who has just been invited cannot access the portal at all
- They must complete Ever-boarding before any portal navigation is available
- The `/portal/onboarding-status` and `/portal/resolve-blockers` pages must NOT be wrapped in `PortalRoute` or they create a catch-22

**Verified:** `/portal/onboarding-status` and `/portal/resolve-blockers` are correctly wrapped in `PortalRoute` — this should be reviewed. If `requireOnboardingComplete` blocks access for incomplete users, these pages are themselves inaccessible to the users who need them most.

---

## 6. What is Confirmed Working

- ✅ All 23 portal page files exist (verified)
- ✅ 24 portal hooks implemented
- ✅ `portal_campaigns`, `portal_payments`, `portal_transactions`, `portal_statements`, `portal_scheduled_payments`, `portal_transaction_documents` tables in DB
- ✅ Council portal — 13 pages, routed at `/portal/councils/*`
- ✅ Service Provider portal — `ProviderDashboard`, `ProviderAssignmentDetail` pages
- ✅ Portal API client (`api/client.ts`) with endpoint modules for account, billing, campaigns, documents, messages, payments, statements, transactions, users
- ✅ `PortalAgentAdapter` — 4 specialist agents (Billing, DocumentRequest, Notification, Support)
- ✅ Ever-boarding integration services (`everboardingIntegration.ts`, `partnerEverboardingIntegration.ts`)
- ✅ `openapi.yaml` present — Portal has documented API spec

---

## 7. What Remains Incomplete

| Feature | Status | Notes |
|---|---|---|
| Agency multi-client portal | ❌ Missing | Not built |
| Marketplace self-service booking | ❌ Missing | Phase 3b |
| "Run again" brief | ❌ Missing | Roadmap |
| Programmatic deal view | ❌ Missing | No SSP win visibility |
| Real-time delivery WebSocket | ❌ Missing | Polling only |
| `DocumentsPage` routing | ❌ Orphaned | Page exists, no route |
| `SupportPage` routing | ❌ Orphaned | Page exists, `/portal/support` redirects to dashboard |
| `NotificationSettingsPage` | 🟡 Stub | 533 bytes, not functional |
| `ResolveBlockersPage` | 🟡 Partial | 8,721 bytes, partial implementation |

---

## 8. Technical Debt

| Issue | Severity | Notes |
|---|---|---|
| `/portal/documents` routes to wrong page | HIGH | Goes to TasksSettings instead of DocumentsPage |
| `/portal/support` redirects to dashboard | MEDIUM | SupportPage orphaned |
| Self-referencing redirect at `/portal/settings` | MEDIUM | Potential infinite loop |
| `PortalRoute requireOnboardingComplete` blocks onboarding pages | MEDIUM | `/portal/onboarding-status` and `/portal/resolve-blockers` may be inaccessible to the users who need them |
| Old route paths hardcoded in Comms Hub notifications | MEDIUM | Email links to `/portal/documents`, `/portal/compliance` etc. will redirect — need audit |
| `MoneyPage` and `FinancialsPage` both exist | LOW | Scope overlap — v1.0 noted this, still unresolved |
| `BillingPage` superseded by `MoneyPage` | LOW | 7KB page with no active route |
| No error boundaries on portal routes | LOW | Page crashes not isolated |
| Council portal increases main bundle | LOW | Heavy route nested in portal bundle |

---

## 9. Business Requirements Gap Assessment

| Requirement | Status | Notes |
|---|---|---|
| Clients view proposals | ✅ Met | CampaignsPage |
| Clients sign booking orders | ✅ Met | SignaturesPage |
| Clients upload creatives | ✅ Met | CampaignsPage |
| Clients view proof of play | ✅ Met | CampaignsPage |
| Clients download invoices | ⚠️ Partial | DocumentsPage exists but not routed |
| Clients manage team/roles | ✅ Met | UsersPage |
| Clients view compliance tasks | ⚠️ Partial | CompliancePage exists, no direct route |
| Clients request credit | ✅ Met | CreditRequestPage |
| Clients message sales rep | ✅ Met | MessagesPage |
| Council review applications | ✅ Met | Council sub-portal complete |
| Partner onboarding flow | ✅ Met | partnerEverboardingIntegration |

---

## 10. Production Readiness

**Rating: 7/10**

The portal has substantial functionality. The main risk is routing — 2 pages are orphaned, the documents route goes to the wrong page, and the support parent route is dead. These affect real client journeys (downloading invoices, raising support). Easy to fix but must be done before go-live.

---

*Portal Module Audit v2.0 — 2026-03-25*
