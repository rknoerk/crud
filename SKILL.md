---
name: crud
description: Use when building a CRUD web app from a YAML entity schema, scaffolding Supabase tables, React components, routes, and forms for list/edit views.
version: 1.3.1
---

**CRUD Skill v1.3.1 loaded.**

# CRUD Generator

## Overview
- Hybrid pattern-book + schema-driven workflow
- Input: YAML entity schemas defining DB structure + UI hints
- Output: Supabase migrations, TypeScript/Zod types, routes, React components
- Tech stack: TanStack Start (React + TypeScript + Vite), Supabase, TanStack React Query v5, React Hook Form + Zod, Tailwind + shadcn/ui

## Router & Framework

- **Default: TanStack Start** — file-based routing, server functions with middleware, SSR via Nitro. This is the Lovable default since May 2026.
- **Legacy: React Router v6/v7** — aeltere Lovable-Projekte (vor Mai 2026) nutzen noch React Router mit Client-side Routing. Die Patterns in diesem Skill sind primaer fuer TanStack Start geschrieben. Bei Legacy-Projekten: Route-Definitionen und Navigation-Hooks anpassen (z.B. `useRouter` → `useNavigate`, file-based routes → manuelle Route-Config). Die Kernlogik (React Query, Zod, Supabase, Komponenten) bleibt identisch.
- **Erkennung:** TanStack Start hat `@tanstack/react-start` in package.json. Legacy-Projekte haben `react-router-dom`.

## When to Use
- Building a new CRUD app or adding entities to an existing one
- User provides or you help create a YAML entity schema
- NOT for: non-CRUD features, dashboards, auth setup, export functionality

## Workflow

6 steps, halt between each to show user and get feedback:

1. **Schema lesen & validieren** → See `schema-format.md`
2. **Supabase-Migration generieren** → See `patterns/migration.md`
3. **TypeScript-Typen generieren** → See `patterns/form.md` (Zod section)
4. **Routes anlegen** → See `patterns/navigation.md` (routing, breadcrumbs, URL params, guards), `patterns/navigation-layout.md` (sidebar, tabs, AppShell)
5. **Komponenten generieren** → See `patterns/list.md`, `patterns/form.md`, `patterns/formatting.md` (value display), `patterns/input-conventions.md` (masks, parsing, field sizing), `patterns/images.md` (upload, gallery, focal point)
6. **Qualitaets-Check** — **HALT: Do NOT present code to user until EVERY item below is verified. Go through each item, check the generated code, and fix violations before proceeding. This is not optional.**

   **Functional completeness:**
   - [ ] All routes reachable (list + edit per entity)
   - [ ] Single shared Zod schema per entity — used by both form (client) and server function (server). No duplicate schemas.
   - [ ] When adding a field: schema file, form UI, DB column, and types file all updated
   - [ ] All `list: true` fields appear in DataTable columns
   - [ ] All `sortable` fields have sort toggle + DB index
   - [ ] All `filterable` fields have filter UI + DB index
   - [ ] All `searchable` fields included in search query
   - [ ] Relations render as links in list, combobox in form
   - [ ] European formats applied (DD.MM.YYYY, dot thousands, comma decimal)
   - [ ] Design system tokens used, no hardcoded colors/sizes
   - [ ] Empty states and error toasts present
   - [ ] Delete with confirmation dialog
   - [ ] Breadcrumbs generated from URL hierarchy
   - [ ] Filter/sort persisted in URL search params
   - [ ] UI labels in German (or English if explicitly requested by user), code in English

   **Unsaved changes guard (MANDATORY — grep for `window.confirm` to catch violations):**
   - [ ] `UnsavedChangesGuard` component present in every form with editable state
   - [ ] `window.confirm()` NEVER used anywhere — always use `UnsavedChangesGuard` which uses `beforeunload` + router blocker
   - [ ] The `handleCancel` function must NOT contain `window.confirm` — instead rely on `UnsavedChangesGuard` to intercept navigation

   Correct pattern:
   ```tsx
   // UnsavedChangesGuard handles both browser close AND in-app navigation
   <UnsavedChangesGuard isDirty={form.formState.isDirty} />

   // handleCancel just navigates — the guard intercepts if dirty
   function handleCancel() {
     navigate({ to: '/list' })
   }
   ```

   Wrong pattern (NEVER do this):
   ```tsx
   // ❌ window.confirm is ugly, inconsistent, and blocks the thread
   function handleCancel() {
     if (form.formState.isDirty) {
       if (!window.confirm('Ungespeicherte Änderungen...')) return
     }
     navigate({ to: '/list' })
   }
   ```

   **Visual Consistency (code-prüfbar):**
   - [ ] Save button in header (right), Delete button in footer (left) — never in same row
   - [ ] Delete button: `variant="outline"` + `className="text-destructive"`, text only, no icon, only for existing records
   - [ ] Max 1 `variant="default"` (primary) button per view — all others `outline` or `ghost`
   - [ ] Custom input components match shadcn height (`h-9`) — flag `h-10`, `h-11`, `h-12` in input/container elements
   - [ ] All elements in a grid row same height — no mixed `h-9`/`h-12` causing baseline misalignment
   - [ ] No `placeholder` prop on `<Input>` or `<Textarea>` — only on `<Select>` ("Bitte wählen...") and `<Combobox>` ("Suchen...")
   - [ ] `window.confirm()` not used — see "Unsaved changes guard" section above

## Key Conventions
- UI labels default to German. English labels are allowed when the user explicitly requests it. Code is always English.
- European formats: dates DD.MM.YYYY, numbers with dot thousands separator, comma decimal
- All components use CSS variables from design system tokens, no hardcoded values → See `design-system.md`
- Filter/sort state in URL search params (shareable, preserved on back-navigation)
- Explicit save (no autosave), unsaved changes warning on navigate away
- Hard delete with confirmation dialog
- Toasts for success/error, validation errors inline at field
