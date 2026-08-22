# CRUD Skill

Claude Code Skill for generating CRUD web apps from YAML entity schemas.

## What it does

Takes a YAML entity definition as input and generates:
- Supabase migrations (tables, enums, indexes, triggers)
- TypeScript/Zod validation schemas
- TanStack Start routes (list + edit per entity)
- React components (DataTable, forms, filters, search)

## Tech Stack

- TanStack Start (React + TypeScript + Vite)
- Supabase (PostgreSQL, Auth, Storage)
- TanStack React Query v5
- React Hook Form + Zod
- Tailwind CSS + shadcn/ui

## Usage

Invoke via `/crud` in Claude Code, then provide a YAML entity schema. The skill walks through 6 steps with a halt between each for feedback:

1. Schema lesen & validieren
2. Supabase-Migration generieren
3. TypeScript-Typen generieren
4. Routes anlegen
5. Komponenten generieren
6. Qualitaets-Check

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill definition, workflow, quality checklist |
| `schema-format.md` | YAML schema format reference |
| `design-system.md` | CSS variable tokens, component catalog |
| `patterns/migration.md` | YAML to SQL migration pattern |
| `patterns/form.md` | React Hook Form + Zod, widgets, mutations |
| `patterns/list.md` | DataTable, filter/sort/search, pagination |
| `patterns/navigation.md` | Routing, breadcrumbs, URL params, guards |
| `patterns/navigation-layout.md` | Sidebar, navigation rail, bottom tabs |
| `patterns/formatting.md` | Value display, European formats |
| `patterns/input-conventions.md` | Input height, masks, parsing, field sizing |
| `patterns/images.md` | Upload, gallery, drag-reorder, focal point |

## Versioning

Uses semver. The version is stored in `SKILL.md` frontmatter and displayed on first invocation.

- **Major** (2.0.0): Breaking changes to schema format, workflow steps, or pattern structure
- **Minor** (1.1.0): New patterns, new checklist items, new field types
- **Patch** (1.1.1): Typo fixes, clarifications, example corrections

## Changelog

### 1.1.0 (2026-08-22)
- Added Visual Consistency checklist (8 code-checkable rules: button placement, input height, primary count, placeholders, ConfirmDialog, UnsavedChangesGuard)
- Added Input Height rule (all inputs must match shadcn `h-9`)
- Added Button Placement principle (Save in header, Delete in footer)
- Language flexibility: English UI labels allowed when explicitly requested
- Added README with versioning policy

### 1.0.0 (2026-08-06)
- Initial release: schema format, 6-step workflow, all patterns (migration, form, list, navigation, navigation-layout, formatting, input-conventions, images, design-system)
