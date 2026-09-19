# Navigation & Routing Patterns

## Route Structure per Entity

```
/entity                              -> List view
/entity/$id                          -> Form view (edit)
/entity/$id/sub-entity               -> Sub-list (hierarchical 1:n)
/entity/$id/sub-entity/$subId        -> Sub-form (edit)
```

**No separate detail/read-only view.** Clicking a row in the list opens the edit form directly. This eliminates an unnecessary click and a second view to maintain per entity.

## TanStack Start File-Based Routing (Default)

TanStack Start uses file-based routing under `src/routes/`. File names map to URL paths.

### File Naming Convention

**IMPORTANT: Use the `_` (pathless layout) prefix** to make `$id` routes standalone siblings of the list route, NOT children:

```
src/routes/
  __root.tsx              -> Root layout (sidebar, Outlet)
  index.tsx               -> /
  projekte.tsx            -> /projekte          (list)
  projekte_.$id.tsx       -> /projekte/$id      (form — standalone, not nested under projekte.tsx)
  projekte_.$id.szenen.tsx         -> /projekte/$id/szenen          (sub-list)
  projekte_.$id.szenen.$szeneId.tsx -> /projekte/$id/szenen/$szeneId (sub-form)
```

The `_` after `projekte` in `projekte_.$id.tsx` means: "share the `/projekte` URL prefix but do NOT render inside `projekte.tsx`". Without `_`, TanStack Router treats `projekte.$id.tsx` as a child of `projekte.tsx` and requires `<Outlet />` in the parent — which breaks the list/form switching pattern.

**Rule: Always use `entity_.$id.tsx` (with underscore) unless you specifically want a master-detail layout with `<Outlet />` in the list.**

### Route Definition

```tsx
// src/routes/projekte.tsx (list)
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/projekte')({
  component: ProjekteList,
})

function ProjekteList() {
  // ...
}
```

```tsx
// src/routes/projekte_.$id.tsx (form — note the path does NOT include the underscore)
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/projekte/$id')({
  component: ProjektForm,
})

function ProjektForm() {
  const { id } = Route.useParams()
  // ...
}
```

### Navigation

```tsx
import { useNavigate, Link } from '@tanstack/react-router'

// Programmatic navigation
const navigate = useNavigate()
navigate({ to: '/projekte/$id', params: { id: projekt.id } })

// Link component
<Link to="/projekte/$id" params={{ id: projekt.id }}>
  {projekt.titel}
</Link>

// Navigate back to list
navigate({ to: '/projekte' })
```

### Getting Route Params

```tsx
// Inside a route component — use the Route object
const { id } = Route.useParams()

// Or with the generic hook
import { useParams } from '@tanstack/react-router'
const { id } = useParams({ from: '/projekte/$id' })
```

## URL Hierarchy = Navigation Stack (Breadcrumbs)

Each URL segment maps to a breadcrumb entry. Labels come from an entity registry: `label.plural` for list segments, the record's title field for detail segments.

Example: `/projekte/123/szenen/456`
-> `Projekte > Film A > Szenen > Szene 1`

### Entity Registry

```tsx
// lib/entity-registry.ts
export interface EntityMeta {
  /** URL path segment (plural, lowercase) */
  path: string;
  /** German labels */
  label: { singular: string; plural: string };
  /** Field used as display title in breadcrumbs and detail headers */
  titleField: string;
  /** Supabase table name */
  table: string;
  /** Parent entity path, if hierarchical */
  parent?: string;
}

export const entityRegistry: Record<string, EntityMeta> = {
  projekte: {
    path: "projekte",
    label: { singular: "Projekt", plural: "Projekte" },
    titleField: "titel",
    table: "projekte",
  },
  szenen: {
    path: "szenen",
    label: { singular: "Szene", plural: "Szenen" },
    titleField: "titel",
    table: "szenen",
    parent: "projekte",
  },
};
```

### Breadcrumbs Component

```tsx
// components/Breadcrumbs.tsx
import { Link, useRouterState } from "@tanstack/react-router";
import { useQuery } from "@tanstack/react-query";
import { supabase } from "@/lib/supabase";
import { entityRegistry } from "@/lib/entity-registry";

interface Crumb {
  label: string;
  href: string;
}

function useRecordTitle(table: string, id: string | undefined, titleField: string) {
  return useQuery({
    queryKey: [table, id, "title"],
    queryFn: async () => {
      const { data } = await supabase
        .from(table)
        .select(titleField)
        .eq("id", id!)
        .single();
      return data?.[titleField] ?? id;
    },
    enabled: !!id,
    staleTime: 5 * 60 * 1000,
  });
}

export function Breadcrumbs() {
  const routerState = useRouterState();
  const pathname = routerState.location.pathname;

  // Split path into segments, build crumbs from entity registry
  const segments = pathname.split("/").filter(Boolean);
  const crumbs: Crumb[] = [];
  let href = "";

  for (let i = 0; i < segments.length; i++) {
    const segment = segments[i];
    href += `/${segment}`;

    const entity = entityRegistry[segment];
    if (entity) {
      crumbs.push({ label: entity.label.plural, href });
    } else {
      // ID segment -> resolved to title via query (see RecordCrumb)
      crumbs.push({ label: segment, href });
    }
  }

  return (
    <nav aria-label="Breadcrumb">
      <ol className="flex items-center gap-1.5 text-sm text-muted-foreground">
        {crumbs.map((crumb, i) => (
          <li key={crumb.href} className="flex items-center gap-1.5">
            {i > 0 && <span>/</span>}
            {i < crumbs.length - 1 ? (
              <Link to={crumb.href} className="hover:text-foreground transition-colors">
                <RecordCrumb segments={segments} index={i} fallback={crumb.label} />
              </Link>
            ) : (
              <span className="text-foreground font-medium">
                <RecordCrumb segments={segments} index={i} fallback={crumb.label} />
              </span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}

/** Resolves an ID segment to a record title, or renders the static label. */
function RecordCrumb({ segments, index, fallback }: {
  segments: string[];
  index: number;
  fallback: string;
}) {
  const segment = segments[index];
  const entity = entityRegistry[segment];

  if (entity) {
    return <>{fallback}</>;
  }

  // ID segment: look up the preceding entity segment
  const parentSegment = segments[index - 1];
  const parentEntity = entityRegistry[parentSegment];
  if (!parentEntity) return <>{fallback}</>;

  const { data: title } = useRecordTitle(parentEntity.table, segment, parentEntity.titleField);
  return <>{title ?? fallback}</>;
}
```

## Filter & Sort in URL Search Params

Format: `/projekte?status=aktiv&sort=-startdatum`

- Prefix `-` on sort value = descending, no prefix = ascending
- Multiple filters: `/projekte?status=aktiv&kunde=abc-123`
- React Query key includes these params -> auto-refetch on param change
- URLs are shareable and params are preserved on back-navigation

### useUrlParams Hook

```tsx
// hooks/useUrlParams.ts
import { useNavigate, useRouterState } from "@tanstack/react-router";
import { useMemo, useCallback } from "react";

export interface SortParam {
  field: string;
  direction: "asc" | "desc";
}

export interface UrlParamsResult {
  filters: Record<string, string>;
  sort: SortParam | null;
  search: string;
  setFilter: (key: string, value: string | null) => void;
  setSort: (field: string) => void;  // toggles direction if same field
  setSearch: (value: string) => void;
  /** All params as a stable object for React Query keys */
  queryParams: Record<string, string>;
}

export function useUrlParams(): UrlParamsResult {
  const routerState = useRouterState();
  const navigate = useNavigate();
  const searchParams = routerState.location.search as Record<string, string>;

  const queryParams = useMemo(() => {
    return { ...searchParams };
  }, [searchParams]);

  const sort = useMemo((): SortParam | null => {
    const raw = searchParams.sort;
    if (!raw) return null;
    if (raw.startsWith("-")) {
      return { field: raw.slice(1), direction: "desc" };
    }
    return { field: raw, direction: "asc" };
  }, [searchParams]);

  const search = searchParams.q ?? "";

  const filters = useMemo(() => {
    const f: Record<string, string> = {};
    for (const [key, value] of Object.entries(searchParams)) {
      if (key !== "sort" && key !== "q") {
        f[key] = value;
      }
    }
    return f;
  }, [searchParams]);

  const updateSearch = useCallback(
    (updater: (prev: Record<string, string>) => Record<string, string>) => {
      navigate({
        search: (prev: Record<string, string>) => updater(prev),
        replace: true,
      });
    },
    [navigate],
  );

  const setFilter = useCallback(
    (key: string, value: string | null) => {
      updateSearch((prev) => {
        const next = { ...prev };
        if (value === null) {
          delete next[key];
        } else {
          next[key] = value;
        }
        return next;
      });
    },
    [updateSearch],
  );

  const setSort = useCallback(
    (field: string) => {
      updateSearch((prev) => {
        const next = { ...prev };
        if (prev.sort === field) {
          next.sort = `-${field}`; // toggle to desc
        } else if (prev.sort === `-${field}`) {
          delete next.sort; // toggle off
        } else {
          next.sort = field; // new field, asc
        }
        return next;
      });
    },
    [updateSearch],
  );

  const setSearch = useCallback(
    (value: string) => {
      updateSearch((prev) => {
        const next = { ...prev };
        if (value) {
          next.q = value;
        } else {
          delete next.q;
        }
        return next;
      });
    },
    [updateSearch],
  );

  return { filters, sort, search, setFilter, setSort, setSearch, queryParams };
}
```

Usage in React Query:

```tsx
const { queryParams } = useUrlParams();

const { data } = useQuery({
  queryKey: ["projekte", "list", queryParams],
  queryFn: () => fetchProjekte(queryParams),
});
```

## Cross-Reference Navigation (Querverweise)

When navigating to a related record from a detail view (e.g., clicking a Kunde link from a Projekt), pass search params so the back button can return to the referrer.

```tsx
// Navigating to a cross-reference
navigate({
  to: '/kontakte/$id',
  params: { id: kundeId },
  search: { from: location.pathname },
})
```

### useNavigateBack Hook

```tsx
// hooks/useNavigateBack.ts
import { useNavigate, useRouterState } from "@tanstack/react-router";
import { useCallback } from "react";

/**
 * Returns a navigate-back function that:
 * 1. Uses search.from if set (cross-reference return)
 * 2. Falls back to the canonical parent route (one segment up)
 */
export function useNavigateBack() {
  const navigate = useNavigate();
  const routerState = useRouterState();
  const pathname = routerState.location.pathname;
  const search = routerState.location.search as Record<string, string>;

  const goBack = useCallback(() => {
    if (search.from) {
      navigate({ to: search.from });
      return;
    }

    // Canonical parent: strip last path segment
    const parentPath = pathname.replace(/\/[^/]+\/?$/, "") || "/";
    navigate({ to: parentPath });
  }, [navigate, pathname, search]);

  return goBack;
}
```

Usage:

```tsx
const goBack = useNavigateBack();

<Button variant="ghost" onClick={goBack}>
  <ArrowLeft className="mr-2 h-4 w-4" />
  Zurueck
</Button>
```

## Unsaved Changes Guard

TanStack Router has `useBlocker` with built-in `enableBeforeUnload`. Use `withResolver: true` to show an AlertDialog instead of `window.confirm`.

**NEVER use `window.confirm()` for unsaved changes.** Always use this component.

```tsx
// components/UnsavedChangesGuard.tsx
import { useBlocker } from '@tanstack/react-router'
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from '@/components/ui/alert-dialog'

interface Props {
  isDirty: boolean
}

export function UnsavedChangesGuard({ isDirty }: Props) {
  const { proceed, reset, status } = useBlocker({
    shouldBlockFn: () => isDirty,
    enableBeforeUnload: isDirty,
    withResolver: true,
  })

  return (
    <AlertDialog open={status === 'blocked'}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Ungespeicherte Änderungen</AlertDialogTitle>
          <AlertDialogDescription>
            Es gibt ungespeicherte Änderungen. Trotzdem verlassen?
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel onClick={reset}>Abbrechen</AlertDialogCancel>
          <AlertDialogAction onClick={proceed}>Verlassen</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  )
}
```

This handles both:
- **Browser close/refresh:** native `beforeunload` dialog
- **In-app navigation:** styled AlertDialog via `useBlocker` resolver

Cancel button — just navigate, the guard intercepts if dirty:

```tsx
function handleCancel() {
  navigate({ to: '/list' })
}

<Button variant="ghost" onClick={handleCancel}>
  Abbrechen
</Button>
```

## Legacy: React Router v6/v7

For projects using React Router (pre-May 2026 Lovable projects), use the standard React Router APIs:

- `createBrowserRouter` instead of file-based routing
- `useSearchParams` instead of `useRouterState().location.search`
- `useParams` from `react-router-dom`
- `useNavigate` from `react-router-dom`
- `useBlocker` for unsaved changes guard (available in React Router v6.4+)

The component logic (React Query, Supabase, forms) stays the same — only the routing layer differs.

---

## See also

- `patterns/navigation-layout.md` — AppShell, sidebar, bottom tabs
- `patterns/list.md` — list view (uses useUrlParams, useScrollRestore)
- `patterns/form.md` — form view (uses UnsavedChangesGuard, useNavigateBack)
