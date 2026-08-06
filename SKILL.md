---
name: crud
description: Use when building a CRUD web app from a YAML entity schema, scaffolding Supabase tables, React components, routes, and forms for list/detail/edit views.
---

# CRUD Generator

## Overview
- Hybrid pattern-book + schema-driven workflow
- Input: YAML entity schemas defining DB structure + UI hints
- Output: Supabase migrations, TypeScript/Zod types, routes, React components
- Tech stack: React + TypeScript + Vite, Supabase, TanStack React Query v5, React Hook Form + Zod, Tailwind + shadcn/ui

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
6. **Qualitaets-Check** — Checklist (to be added later)

## Key Conventions
- German UI labels, English code
- European formats: dates DD.MM.YYYY, numbers with dot thousands separator, comma decimal
- All components use CSS variables from design system tokens, no hardcoded values → See `design-system.md`
- Filter/sort state in URL search params (shareable, preserved on back-navigation)
- Explicit save (no autosave), unsaved changes warning on navigate away
- Hard delete with confirmation dialog
- Toasts for success/error, validation errors inline at field
