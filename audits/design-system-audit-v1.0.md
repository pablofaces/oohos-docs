# Design System Audit v1.0

**Date:** 2026-03-23  
**Auditor:** Cascade (AI)  
**Status:** Package Live — Consumed by All Modules  
**Source paths audited:** `packages/design-system/src/`, `docs/design-system/`, `.storybook/`

---

## 1. Module Identity

| Field | Value |
|---|---|
| Module name | Design System |
| Package path | `packages/design-system/src/` |
| Import path | `@oohos/design-system` |
| Owner | Horizontal (platform-wide) |
| Phase | Live — consumed by all modules |

---

## 2. Current State Summary

The Design System is the shared UI component library and design token system for OOH OS. All module frontends consume it via `@oohos/design-system`. No module imports external UI libraries directly — all UI primitives come from the design system.

**What is built:**
- Design tokens (`packages/design-system/src/tokens/`) — colours, typography, spacing, status colours
- Shared components (`packages/design-system/src/components/`) — forms, tables, maps, modals, cards, status badges
- Storybook integration (`.storybook/`) — component documentation + visual testing
- Tailwind preset (`packages/design-system/tailwind.preset.ts`) — shared Tailwind config
- Map component — Mapbox/Leaflet wrapper with OOH OS styling
- Status colour tokens — availability (green/amber/red), proposal stages, booking states
- Typography — DM Sans, minimalist hierarchy, no custom fonts in module UI

---

## 3. Design Principles

- **DM Sans** — primary typeface, minimalist hierarchy
- **No custom fonts** — no module adds its own typeface
- **No external UI libraries in modules** — TailwindCSS + shadcn/ui + design system tokens only
- **Status colours** — semantic colour tokens for OOH-specific states (availability, booking lifecycle, compliance status)
- **Map component** — single shared map wrapper; all modules use the same map with OOH OS styling overlay

---

## 4. Built vs Stub vs Missing

| Feature | Status | Notes |
|---|---|---|
| Design tokens (colours, typography, spacing) | ✅ Built | `packages/design-system/src/tokens/` |
| Shared components (forms, tables, modals, cards, badges) | ✅ Built | `packages/design-system/src/components/` |
| Tailwind preset | ✅ Built | `tailwind.preset.ts` — shared config |
| Storybook setup | ✅ Built | `.storybook/` — `main.ts` + `preview.tsx` |
| Map component | ✅ Built | Mapbox/Leaflet with OOH OS styling |
| Status colour tokens | ✅ Built | Availability, proposal stages, booking states |
| Linting rules (`src/lint/`) | ✅ Built | Design system lint enforcements |
| Dark mode | ❌ Missing | Not built — light mode only |
| Accessibility audit | ❌ Missing | No formal WCAG audit documented |
| Component playground / live docs | 🟡 Partial | Storybook exists but story coverage unknown |
| Icon library | ✅ Built | Lucide icons (standard across platform) |

---

## 5. Package Structure

```
packages/design-system/src/
├── components/     — shared React components
├── tokens/         — design tokens (colours, typography, spacing)
├── lib/            — utility functions
├── lint/           — ESLint rules for design system compliance
└── index.ts        — public API (5,459 bytes)
```

`tailwind.preset.ts` — shared Tailwind config consumed by all app `tailwind.config.ts` files.

**Per-app Tailwind configs** reference the preset:
- `apps/cmms/tailwind.config.ts`
- `apps/comms-hub/tailwind.config.ts`
- `apps/council-portal/tailwind.config.ts`
- `apps/everboarding-form/tailwind.config.ts`

---

## 6. Documentation

`docs/design-system/` contains:
- `README.md` — overview and usage
- `SETUP.md` — integration guide for new modules
- `CHANGELOG.md` — version history
- `components/` — per-component documentation
- `guidelines/` — usage guidelines
- `patterns/` — design patterns

---

## 7. Storybook

`.storybook/main.ts` + `.storybook/preview.tsx` — Storybook v7+ configuration.

Used for: component documentation, visual regression testing, design review.

---

## 8. Technical Debt and Known Issues

| Issue | Severity | Notes |
|---|---|---|
| No dark mode | Medium | Light mode only — enterprise clients may expect dark mode option |
| No formal WCAG audit | Medium | Accessibility compliance not verified against WCAG 2.1 AA |
| Storybook story coverage unknown | Low | Stories may not cover all components |
| No visual regression CI | Low | No automated screenshot comparison in CI/CD pipeline |
| `src/lint/` enforcement coverage unknown | Low | Design system lint rules may not cover all violation patterns |

---

*End of Design System Audit v1.0*
