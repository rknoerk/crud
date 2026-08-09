---
name: crud
description: Use when building a CRUD web app from a YAML entity schema, scaffolding Supabase tables, React components, routes, and forms for list/detail/edit views.
---

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

1. **Schema lesen & validieren** — Parse YAML, validate required fields, resolve relations. Report errors early. → See `schema-format.md`
2. **Supabase-Migration generieren** — CREATE TABLE with correct types, auto id/timestamps, indexes. → See `patterns/migration.md`
3. **TypeScript-Typen generieren** — Zod schema from YAML, infer TS interface. One file per entity in `schemas/`. → See `patterns/form.md` (Zod section)
4. **Routes anlegen** — List, detail, edit per entity. Sub-routes for hierarchical relations. Responsive master-detail. → See `patterns/navigation.md`
5. **Komponenten generieren** — EntityList, EntityDetail, EntityForm + React Query hooks. → See `patterns/list.md`, `patterns/detail.md`, `patterns/form.md`
6. **Qualitaets-Check** — Run through this checklist before presenting to user:
   - [ ] All routes reachable (list, detail, edit per entity)
   - [ ] Zod schema matches DB schema (types, required, enums)
   - [ ] All `list: true` fields appear in DataTable columns
   - [ ] All `sortable` fields have sort toggle + DB index
   - [ ] All `filterable` fields have filter UI + DB index
   - [ ] All `searchable` fields included in search query
   - [ ] Relations render as links in detail, combobox in form
   - [ ] European formats applied (DD.MM.YYYY, dot thousands, comma decimal)
   - [ ] Design system tokens used, no hardcoded colors/sizes
   - [ ] Empty states and error toasts present
   - [ ] Unsaved changes guard on edit form
   - [ ] Delete with confirmation dialog
   - [ ] Breadcrumbs generated from URL hierarchy
   - [ ] Filter/sort persisted in URL search params
   - [ ] German UI labels, English code

## Key Conventions
- German UI labels, English code
- European formats: dates DD.MM.YYYY, numbers with dot thousands separator, comma decimal
- All components use CSS variables from design system tokens, no hardcoded values → See `design-system.md`
- Filter/sort state in URL search params (shareable, preserved on back-navigation)
- Explicit save (no autosave), unsaved changes warning on navigate away
- Hard delete with confirmation dialog
- Toasts for success/error, validation errors inline at field
