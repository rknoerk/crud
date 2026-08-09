# Navigation & Routing Patterns

## Route Structure per Entity

```
/entity                              -> List view
/entity/:id                          -> Form view (edit)
/entity/:id/sub-entity               -> Sub-list (hierarchical 1:n)
/entity/:id/sub-entity/:subId        -> Sub-form (edit)
```

**No separate detail/read-only view.** Clicking a row in the list opens the edit form directly. This eliminates an unnecessary click and a second view to maintain per entity.

Route definition example (React Router v6):

```tsx
// routes.tsx
import { createBrowserRouter } from "react-router-dom";
import { AppLayout } from "@/components/AppLayout";

export const router = createBrowserRouter([
  {
    path: "/",
    element: <AppLayout />,
    children: [
      {
        path: "projekte",
        element: <ProjekteLayout />,  // desktop: master-detail via <Outlet />
        children: [
          { index: true, element: <ProjektList /> },
          { path: ":id", element: <ProjektForm /> },
          {
            path: ":id/szenen",
            children: [
              { index: true, element: <SzeneList /> },
              { path: ":szeneId", element: <SzeneForm /> },
            ],
          },
        ],
      },
    ],
  },
]);
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
import { Link, useMatches, useParams } from "react-router-dom";
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
  const params = useParams();
  const pathname = location.pathname;

  // Split path into segments, build crumbs from entity registry
  const segments = pathname.split("/").filter(Boolean);
  const crumbs: Crumb[] = [];
  let href = "";

  for (let i = 0; i < segments.length; i++) {
    const segment = segments[i];
    href += `/${segment}`;

    const entity = entityRegistry[segment];
    if (entity) {
      // This is an entity list segment
      crumbs.push({ label: entity.label.plural, href });
    } else if (segment === "edit") {
      crumbs.push({ label: "Bearbeiten", href });
    } else {
      // This is an ID segment -> resolved to title via query (see RecordCrumb)
      crumbs.push({ label: segment, href }); // placeholder, replaced by RecordCrumb
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

  if (entity || segment === "edit") {
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
import { useSearchParams } from "react-router-dom";
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
  const [searchParams, setSearchParams] = useSearchParams();

  const queryParams = useMemo(() => {
    const obj: Record<string, string> = {};
    searchParams.forEach((value, key) => {
      obj[key] = value;
    });
    return obj;
  }, [searchParams]);

  const sort = useMemo((): SortParam | null => {
    const raw = searchParams.get("sort");
    if (!raw) return null;
    if (raw.startsWith("-")) {
      return { field: raw.slice(1), direction: "desc" };
    }
    return { field: raw, direction: "asc" };
  }, [searchParams]);

  const search = searchParams.get("q") ?? "";

  const filters = useMemo(() => {
    const f: Record<string, string> = {};
    searchParams.forEach((value, key) => {
      if (key !== "sort" && key !== "q") {
        f[key] = value;
      }
    });
    return f;
  }, [searchParams]);

  const setFilter = useCallback(
    (key: string, value: string | null) => {
      setSearchParams((prev) => {
        const next = new URLSearchParams(prev);
        if (value === null) {
          next.delete(key);
        } else {
          next.set(key, value);
        }
        return next;
      }, { replace: true });
    },
    [setSearchParams],
  );

  const setSort = useCallback(
    (field: string) => {
      setSearchParams((prev) => {
        const next = new URLSearchParams(prev);
        const current = next.get("sort");
        if (current === field) {
          next.set("sort", `-${field}`); // toggle to desc
        } else if (current === `-${field}`) {
          next.delete("sort"); // toggle off
        } else {
          next.set("sort", field); // new field, asc
        }
        return next;
      }, { replace: true });
    },
    [setSearchParams],
  );

  const setSearch = useCallback(
    (value: string) => {
      setSearchParams((prev) => {
        const next = new URLSearchParams(prev);
        if (value) {
          next.set("q", value);
        } else {
          next.delete("q");
        }
        return next;
      }, { replace: true });
    },
    [setSearchParams],
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

When navigating to a related record from a detail view (e.g., clicking a Kunde link from a Projekt), pass the origin path so the back button returns to the referrer, not the canonical parent.

```tsx
// Navigating to a cross-reference
navigate(`/kontakte/${kundeId}`, {
  state: { from: location.pathname },
});
```

Back navigation: if `state.from` exists, go there. Otherwise, go one URL segment up (canonical parent).

### useNavigateBack Hook

```tsx
// hooks/useNavigateBack.ts
import { useNavigate, useLocation } from "react-router-dom";
import { useCallback } from "react";

/**
 * Returns a navigate-back function that:
 * 1. Uses location.state.from if set (cross-reference return)
 * 2. Falls back to the canonical parent route (one segment up)
 */
export function useNavigateBack() {
  const navigate = useNavigate();
  const location = useLocation();

  const goBack = useCallback(() => {
    const from = (location.state as { from?: string })?.from;
    if (from) {
      navigate(from);
      return;
    }

    // Canonical parent: strip last path segment
    const parentPath = location.pathname.replace(/\/[^/]+\/?$/, "") || "/";
    navigate(parentPath);
  }, [navigate, location]);

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

## Responsive Master-Detail

### Desktop (>=1024px): Side-by-Side

List and detail render simultaneously via nested route with `<Outlet />`. The list stays mounted, so scroll position is preserved automatically.

```tsx
// components/ProjekteLayout.tsx
import { Outlet, useParams } from "react-router-dom";
import { useMediaQuery } from "@/hooks/useMediaQuery";

export function ProjekteLayout() {
  const isDesktop = useMediaQuery("(min-width: 1024px)");
  const params = useParams();
  const hasSelection = !!params.id;

  if (!isDesktop) {
    // Mobile: render either list OR detail, not both
    return <Outlet />;
  }

  return (
    <div className="flex h-full">
      <div className="w-[400px] shrink-0 border-r overflow-y-auto">
        <ProjektList />
      </div>
      <div className="flex-1 overflow-y-auto">
        {hasSelection ? <Outlet /> : <EmptySelection entity="Projekt" />}
      </div>
    </div>
  );
}
```

### Mobile (<1024px): Full-Page Transitions

List and detail are full-page views. Scroll position is saved to `sessionStorage` keyed by route path, and restored on return.

```tsx
// hooks/useScrollRestore.ts
import { useEffect } from "react";
import { useLocation } from "react-router-dom";

const SCROLL_KEY_PREFIX = "scroll:";

export function useScrollRestore(containerRef: React.RefObject<HTMLElement | null>) {
  const location = useLocation();
  const key = `${SCROLL_KEY_PREFIX}${location.pathname}`;

  // Restore scroll position on mount
  useEffect(() => {
    const saved = sessionStorage.getItem(key);
    if (saved && containerRef.current) {
      containerRef.current.scrollTop = parseInt(saved, 10);
    }
  }, [key, containerRef]);

  // Save scroll position on unmount
  useEffect(() => {
    const el = containerRef.current;
    return () => {
      if (el) {
        sessionStorage.setItem(key, String(el.scrollTop));
      }
    };
  }, [key, containerRef]);
}
```

## Form Navigation

### Edit Flow

"Bearbeiten" button on detail view navigates to the edit route:

```tsx
<Button onClick={() => navigate(`/projekte/${id}/edit`)}>
  Bearbeiten
</Button>
```

### Create Flow

"Neuer Eintrag" inserts a record with defaults via Supabase, then navigates to edit:

```tsx
const createAndEdit = useMutation({
  mutationFn: async () => {
    const { data, error } = await supabase
      .from("projekte")
      .insert({ status: "entwurf" })  // schema defaults
      .select("id")
      .single();
    if (error) throw error;
    return data;
  },
  onSuccess: (data) => {
    navigate(`/projekte/${data.id}/edit`, {
      state: { from: location.pathname },
    });
  },
});
```

### Save Flow

Save the form, show a toast, navigate back to detail:

```tsx
const updateMutation = useMutation({
  mutationFn: async (values: ProjektFormValues) => {
    const { error } = await supabase
      .from("projekte")
      .update(values)
      .eq("id", id);
    if (error) throw error;
  },
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["projekte", id] });
    toast({ title: "Gespeichert", description: "Aenderungen wurden gespeichert." });
    navigate(`/projekte/${id}`);
  },
});
```

### Cancel + Unsaved Changes Guard

Cancel checks for dirty state before navigating away. A route guard covers both in-app navigation and browser back/close.

```tsx
// components/UnsavedChangesGuard.tsx
import { useEffect } from "react";
import { useBlocker } from "react-router-dom";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";

interface Props {
  isDirty: boolean;
}

export function UnsavedChangesGuard({ isDirty }: Props) {
  // Block in-app navigation via React Router
  const blocker = useBlocker(isDirty);

  // Block browser close / refresh
  useEffect(() => {
    if (!isDirty) return;

    const handler = (e: BeforeUnloadEvent) => {
      e.preventDefault();
      // Legacy browsers need returnValue
      e.returnValue = "";
    };

    window.addEventListener("beforeunload", handler);
    return () => window.removeEventListener("beforeunload", handler);
  }, [isDirty]);

  if (blocker.state !== "blocked") return null;

  return (
    <AlertDialog open>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Ungespeicherte Aenderungen</AlertDialogTitle>
          <AlertDialogDescription>
            Es gibt ungespeicherte Aenderungen. Moechtest du die Seite wirklich verlassen?
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel onClick={() => blocker.reset()}>
            Abbrechen
          </AlertDialogCancel>
          <AlertDialogAction onClick={() => blocker.proceed()}>
            Verwerfen
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

Usage in a form component:

```tsx
export function ProjektForm() {
  const { id } = useParams();
  const goBack = useNavigateBack();
  const form = useForm<ProjektFormValues>({ resolver: zodResolver(projektSchema) });
  const { isDirty } = form.formState;

  return (
    <>
      <UnsavedChangesGuard isDirty={isDirty} />
      <form onSubmit={form.handleSubmit(onSave)}>
        {/* fields */}
        <div className="flex gap-2">
          <Button type="button" variant="outline" onClick={goBack}>
            Abbrechen
          </Button>
          <Button type="submit">Speichern</Button>
        </div>
      </form>
    </>
  );
}
```
