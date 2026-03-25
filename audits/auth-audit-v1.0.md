# Auth Module Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Production Live — Supabase Auth + RLS + Magic Links  
**Source paths audited:** `packages/auth/src/`, `supabase/functions/` (auth-*), `CLAUDE.md §§18.1 AUTH`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Auth |
| DB schema | Supabase Auth (`auth.users`) + `public` (magic_link_tokens, app_roles) |
| Package path | `packages/auth/src/` |
| Backend path | `supabase/functions/request-magic-link/`, `validate-magic-token/`, `validate-signature-token/` |
| Owner | Horizontal (foundational dependency) |
| Phase | Production live |

---

## 2. Current State Summary

Auth is the foundational identity and access layer for all OOH OS modules. It provides JWT validation, role-based access control, tenant scoping, and multi-channel magic link authentication.

**Architecture rule:** No module implements its own auth logic. All identity and permission decisions go through the Auth module.

**What is built:**
- Supabase Auth integration — JWT-based sessions, email/password, magic links
- Row-Level Security (RLS) on all tables — tenant scoping via `auth.uid()` + `get_user_platform_tenant_id()`
- `app_role` enum — all platform roles defined as DB-level enum
- `request-magic-link` Edge Function — generates typed tokens (`inv`, `del`, `sig`)
- `validate-magic-token` — validates and consumes magic link tokens
- `validate-signature-token` — validates signing-specific tokens
- `packages/auth/src/` — auth package with hooks, adapters, constants, utilities
- Legacy server/Express/Passport stack removed (Phase 6 of legacy removal)
- `STUB_[MODULE]=true` environment variable pattern for test bypasses

---

## 3. Auth Architecture

### JWT Flow
1. User authenticates (email+password or magic link)
2. Supabase Auth issues JWT with `sub` = user UUID, `app_metadata.role`
3. All API requests carry JWT in `Authorization: Bearer` header
4. Supabase validates JWT automatically on all Edge Function invocations and DB queries
5. RLS policies evaluate `auth.uid()` + `get_user_platform_tenant_id()` for tenant scoping

### Magic Link Flow
1. `request-magic-link` generates token with type (`inv`/`del`/`sig`) and stores in `magic_link_tokens`
2. Comms Hub sends link to recipient
3. Recipient clicks → `validate-magic-token` validates token + expiry + type
4. On valid: token consumed (one-time use), session established or action completed
5. `process-expiring-tokens` — scheduled cleanup of expired tokens

### Tenant Scoping
- `get_user_platform_tenant_id()` — DB function returning current user's tenant ID
- Called in every RLS policy — ensures cross-tenant data leakage is impossible
- `api-tenants` Edge Function — tenant management operations
- `manage-tenant-hierarchy` Edge Function — multi-level tenant hierarchy management

---

## 4. Role Model (app_role enum)

| Role | Scope |
|---|---|
| `platform_admin` | Full platform access (operator admin) |
| `sales_admin` | Full Sales module access |
| `sales_manager` | Approve discounts, confirm bookings |
| `sales_rep` | Assigned accounts, create proposals |
| `finance` | Invoices, commissions, reconciliation |
| `portal_client` | Own account data only |
| `portal_admin` | Full company view in Portal |
| `portal_contact` | Limited portal access |
| `agency_user` | Introduced clients' data only |
| `council_admin` | Council portal routes |
| `service_provider` | Provider portal routes |
| `partner` | Partner-specific flows |

**Role assignment:** Via Ever-boarding flow + `create-role-assignments` Edge Function. Roles are stored in `auth.users.app_metadata` (Supabase Auth) and in `public.user_roles` table for RLS queries.

---

## 5. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| Supabase Auth JWT | ✅ Live | All sessions JWT-based |
| RLS on all tables | ✅ Live | 579 tables (baseline migration), 1,154 RLS policies |
| `request-magic-link` | ✅ Live | Types: `inv`, `del`, `sig` |
| `validate-magic-token` | ✅ Live | One-time use, type-safe |
| `validate-signature-token` | ✅ Live | Signing-specific token validation |
| `process-expiring-tokens` | ✅ Live | Scheduled token cleanup |
| `get_user_platform_tenant_id()` | ✅ Live | Core tenant scoping function |
| `api-keys` Edge Function | ✅ Built | API key management |
| `manage-tenant-hierarchy` | ✅ Built | Multi-level tenant hierarchy |
| `api-tenants` | ✅ Built | Tenant CRUD |
| `packages/auth/src/` package | ✅ Built | Hooks, adapters, constants, utils, testing helpers |
| OAuth (Google, Microsoft) | ❌ Missing | SSO not yet configured |
| MFA enforcement | ❌ Missing | Multi-factor auth not yet enforced (Supabase Auth supports it) |
| Session management UI | ❌ Missing | Admins cannot view/revoke active sessions |
| API key rotation UI | ❌ Missing | `api-keys` function exists but no UI |

---

## 6. Schema Detail

| Table | Purpose |
|---|---|
| `auth.users` | Supabase Auth — user records, `app_metadata.role` |
| `magic_link_tokens` | Token records — `type` (`inv`/`del`/`sig`), `expires_at`, `used_at` |
| `user_roles` | RLS-queryable role assignments per user per tenant |
| `platform_tenants` | Tenant records — `role_config JSONB`, `id` used in all tenant scoping |
| `tenant_feature_flags` | Per-tenant feature flag registry |
| `tenant_business_rules` | Per-tenant configurable rules (hold durations, terminology, SLA, etc.) |

---

## 7. Key Auth Rules

- **Never implement auth logic in modules** — all identity/permission decisions go through Auth
- **RLS enforced on every table** — no table is accessible without RLS
- **Magic link tokens are one-time use** — `validate-magic-token` marks token as used on first access
- **`STUB_[MODULE]=true`** — environment variable to bypass module auth in tests only (never in production)
- **Supabase project:** `gwtzedkvcnnnlbeidzar` (staging only — production is separate, no AI access)

---

## 8. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| No OAuth / SSO | Medium | Enterprise clients expect SSO (Google Workspace, Microsoft 365) |
| MFA not enforced | Medium | Supabase Auth supports TOTP but not yet configured as required for admin roles |
| Session revocation UI missing | Medium | Admins cannot see or revoke active user sessions |
| `app_role` enum changes require migration | Low | Adding/removing roles requires a new migration — cannot be done at runtime |
| `user_roles` table may drift from `auth.users.app_metadata` | Low | Two sources of truth for roles — sync mechanism needed |
| API key rotation has no UI | Low | `api-keys` function exists but ops must use DB or Edge Function directly |
| Magic link `process-expiring-tokens` run frequency not documented | Low | How often does cleanup run? |

---

*End of Auth Module Audit v1.0*
