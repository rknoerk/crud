# CRUD Skill Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a Claude Code skill that generates CRUD web apps from YAML entity schemas, following documented patterns for Supabase + React + shadcn/ui.

**Architecture:** Single skill (`~/.claude/skills/crud/SKILL.md`) with supporting reference files for patterns, schema format, and design tokens. The skill defines a 6-step workflow that reads YAML schemas and guides generation of migrations, types, routes, and components.

**Tech Stack:** Claude Code skill (Markdown), YAML schema format, reference code in TypeScript/SQL/CSS.

---

### Task 1: Create skill skeleton with frontmatter and workflow overview

**Files:**
- Create: `~/.claude/skills/crud/SKILL.md`

**Step 1: Create the skill directory**

```bash
mkdir -p ~/.claude/skills/crud
```

**Step 2: Write SKILL.md with frontmatter, overview, and workflow**

The skill file needs:
- Frontmatter: `name: crud`, `description: Use when building a CRUD web app from a YAML entity schema, scaffolding Supabase tables, React components, routes, and forms for list/detail/edit views.`
- Overview: what the skill does (hybrid pattern-book + schema-driven workflow)
- When to use / when not to use
- The 6-step workflow as a numbered process with halt points between steps

Content should be concise (<500 words for this section). Reference supporting files for details.

**Step 3: Commit**

```bash
cd ~/projects/crud && git add -A && git commit -m "feat: add skill skeleton with workflow overview"
```

---

### Task 2: Write the schema format reference

**Files:**
- Create: `~/.claude/skills/crud/schema-format.md`

**Step 1: Write schema-format.md**

Document the YAML schema format with:
- All supported field types (`text`, `textarea`, `enum`, `number`, `date`, `relation`, `boolean`)
- All UI hints (`list`, `sortable`, `filterable`, `searchable`, `format`, `default`, `required`)
- Entity-level properties (`entity`, `table`, `label`, `default_sort`)
- Auto-generated fields (`id`, `created_at`, `updated_at`)
- A complete example (the `projekt` entity from the design doc)
- Type-to-DB and type-to-widget mapping table

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: add schema format reference"
```

---

### Task 3: Write the navigation & routing patterns

**Files:**
- Create: `~/.claude/skills/crud/patterns/navigation.md`

**Step 1: Create patterns directory and write navigation.md**

Document:
- Route structure per entity (`/entity`, `/:id`, `/:id/edit`, sub-routes)
- URL-hierarchy = navigation stack, breadcrumb generation
- Filter & sort in search params
- Cross-reference navigation with state + fallback
- Responsive master-detail (desktop: nested outlet, mobile: full-page + sessionStorage scroll)
- Formular navigation (edit, create, save, cancel, unsaved changes guard)

Include code snippets for:
- Route definition example
- Breadcrumb component logic
- useNavigateBack hook (state-aware back navigation)

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: add navigation and routing patterns"
```

---

### Task 4: Write the list pattern

**Files:**
- Create: `~/.claude/skills/crud/patterns/list.md`

**Step 1: Write list.md**

Document:
- List layout (header, search, filters, sortable columns, rows)
- Data loading strategy (all initially, paginate at ~200)
- Filter/sort/search as URL search params
- React Query pattern with params in query key
- Empty state
- Delete from list with confirmation
- Scroll position preservation

Include code snippets for:
- useEntityList hook (React Query + Supabase + URL params)
- useUrlParams hook (read/write search params for filters/sort)
- EntityList component structure

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: add list pattern"
```

---

### Task 5: Write the detail pattern

**Files:**
- Create: `~/.claude/skills/crud/patterns/detail.md`

**Step 1: Write detail.md**

Document:
- Detail layout (back/breadcrumb, title, fields, sub-lists)
- Field rendering by type (text, date, currency, relation as link)
- European formatting rules
- Sub-lists from 1:n relations
- Delete with confirmation dialog

Include code snippets for:
- useEntity hook (single record fetch)
- FieldDisplay component (type-aware rendering)
- RelationLink component
- SubList component

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: add detail pattern"
```

---

### Task 6: Write the form/edit pattern

**Files:**
- Create: `~/.claude/skills/crud/patterns/form.md`

**Step 1: Write form.md**

Document:
- Form layout (cancel, save, fields by schema order)
- Widget mapping (type -> component)
- Zod schema generation from YAML
- React Hook Form integration
- Validation (inline errors, required fields)
- Save flow (mutation, toast, navigate back)
- Unsaved changes guard (route blocker)
- European input formats

Include code snippets for:
- generateZodSchema utility (YAML field -> Zod)
- EntityForm component structure
- useCreateEntity / useUpdateEntity / useDeleteEntity mutations
- UnsavedChangesGuard component

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: add form/edit pattern"
```

---

### Task 7: Write the design system structure

**Files:**
- Create: `~/.claude/skills/crud/design-system.md`

**Step 1: Write design-system.md**

Document:
- Token hierarchy (colors, typography, spacing, radius, shadows)
- Complete list of CSS variable names and their purpose
- Component catalog table (pattern -> components -> tokens used)
- Rules: no hardcoded values, shadcn/ui as base, document token usage
- How to create a project-specific theme (which variables to set)

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: add design system structure"
```

---

### Task 8: Write the migration generation pattern

**Files:**
- Create: `~/.claude/skills/crud/patterns/migration.md`

**Step 1: Write migration.md**

Document:
- YAML type -> SQL type mapping
- Auto-generated columns (id, created_at, updated_at with trigger)
- Enum creation
- Foreign key constraints for relations
- Indexes on sortable/filterable fields
- Complete example: `projekt` entity -> full SQL migration

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: add migration generation pattern"
```

---

### Task 9: Wire up SKILL.md to reference all patterns

**Files:**
- Modify: `~/.claude/skills/crud/SKILL.md`

**Step 1: Update SKILL.md**

Add to each workflow step a reference to the corresponding pattern file:
- Step 1 (Schema) -> `schema-format.md`
- Step 2 (Migration) -> `patterns/migration.md`
- Step 3 (Types) -> embedded in form pattern (Zod generation)
- Step 4 (Routes) -> `patterns/navigation.md`
- Step 5 (Components) -> `patterns/list.md`, `patterns/detail.md`, `patterns/form.md`
- Step 6 (Quality check) -> checklist inline

Add the quality check checklist inline in SKILL.md.

**Step 2: Commit**

```bash
git add -A && git commit -m "feat: wire up skill to all pattern references"
```

---

### Task 10: Symlink skill and test with a dry run

**Files:**
- Create symlink: `~/.agents/skills/crud` -> `~/.claude/skills/crud`

**Step 1: Create symlink for codex compatibility**

```bash
ln -sf ~/.claude/skills/crud ~/.agents/skills/crud
```

**Step 2: Test skill discovery**

Open a new Claude Code session and verify the skill appears in the available skills list.

**Step 3: Dry run**

In a test project directory, invoke `/crud` and provide the `projekt` example schema from the design doc. Verify the skill walks through all 6 steps correctly.

**Step 4: Commit any fixes**

```bash
git add -A && git commit -m "feat: finalize skill and add symlink"
```

---

### Task 11: Push to GitHub

**Step 1: Set remote and push**

```bash
cd ~/projects/crud
git remote add origin https://github.com/rknoerk/crud.git 2>/dev/null || true
git push -u origin main
```
