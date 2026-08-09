# List Pattern

## Layout

```
Header: [Entity label.plural]              [+ Neu]
Search: [Suche...]  [Filter1 v] [Filter2 v]
Table:  Spalte1 v  Spalte2  Spalte3  Spalte4
        Wert       Wert     Wert     Wert
        Wert       Wert     Wert     Wert
```

- **Columns** from schema fields with `list: true`, in schema order
- **Sort indicators** on fields with `sortable: true` (arrow up/down showing direction)
- **Filter dropdowns** for fields with `filterable: true` — enum `values` as options, plus "Alle" reset option
- **Search input** rendered only if any field has `searchable: true`
- **Row click** navigates to `/:id` (detail view)

## Data Loading

- Load all data initially (no pagination)
- When `count > 200`: switch to server-side pagination using Supabase `.range(from, to)`, show page controls
- React Query with filter/sort/search params in the query key — changing params triggers automatic refetch

## Filter, Sort, Search as URL Params

Format:

```
?status=aktiv&sort=-startdatum&q=suchbegriff
```

- Prefix `-` on sort means descending, no prefix means ascending
- `q` is the full-text search term
- All other params are filters (key = field name, value = filter value)
- On change: update URL search params, React Query refetches automatically via query key

See `patterns/navigation.md` for the `useUrlParams` hook.

## Entity List Hook

One hook per entity. Applies filters, sort, and search from URL params to the Supabase query.

```typescript
function useProjekte() {
  const { filters, sort, search } = useUrlParams()

  return useQuery({
    queryKey: ['projekte', { filters, sort, search }],
    queryFn: async () => {
      let query = supabase
        .from('projekte')
        .select('*', { count: 'exact' })

      // Search across all searchable fields
      if (search) {
        query = query.or(`titel.ilike.%${search}%`)
      }

      // Apply filters (one .eq per active filter)
      if (filters.status) query = query.eq('status', filters.status)

      // Apply sort (fall back to schema default_sort)
      if (sort) {
        query = query.order(sort.field, { ascending: sort.direction === 'asc' })
      } else {
        query = query.order('startdatum', { ascending: false }) // default_sort from schema
      }

      const { data, error, count } = await query
      if (error) throw error
      return { data, count }
    },
  })
}
```

When count exceeds 200, add pagination:

```typescript
const PAGE_SIZE = 50

function useProjektePaginated() {
  const { filters, sort, search } = useUrlParams()
  const [page, setPage] = useState(0)

  return useQuery({
    queryKey: ['projekte', { filters, sort, search, page }],
    queryFn: async () => {
      let query = supabase
        .from('projekte')
        .select('*', { count: 'exact' })
        .range(page * PAGE_SIZE, (page + 1) * PAGE_SIZE - 1)

      // ... same filter/sort/search logic as above

      const { data, error, count } = await query
      if (error) throw error
      return { data, count, page, pageCount: Math.ceil((count ?? 0) / PAGE_SIZE) }
    },
  })
}
```

## Scroll Position Preservation

- **Desktop (master-detail layout):** List stays mounted in the left panel while detail renders in the outlet. No special handling needed.
- **Mobile (full-screen navigation):** Save and restore scroll position via `sessionStorage`.

See `patterns/navigation.md` for the `useScrollRestore` hook.

Call `saveScroll()` on row click before navigating to edit.

## Empty State

When no data matches (or table is empty):

```tsx
<EmptyState
  icon={FolderOpen}
  title="Keine Projekte vorhanden"
  description="Erstellen Sie Ihr erstes Projekt, um loszulegen."
  action={<Button onClick={handleCreate}>+ Neues Projekt</Button>}
/>
```

## Delete from List

- **Desktop:** Right-click context menu on row with "Loeschen" option
- **Mobile:** Swipe-left gesture reveals delete action
- Both trigger `ConfirmDialog` before executing

```tsx
<ConfirmDialog
  open={deleteTarget !== null}
  title="Projekt loeschen?"
  description="Dieser Vorgang kann nicht rueckgaengig gemacht werden."
  onConfirm={() => deleteMutation.mutate(deleteTarget!)}
  onCancel={() => setDeleteTarget(null)}
/>
```

Delete mutation invalidates the list query:

```typescript
const useDeleteProjekt = () =>
  useMutation({
    mutationFn: async (id: string) => {
      const { error } = await supabase.from('projekte').delete().eq('id', id)
      if (error) throw error
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['projekte'] })
      toast.success('Projekt geloescht')
    },
    onError: () => {
      toast.error('Fehler beim Loeschen')
    },
  })
```

## Component Structure

```tsx
function ProjektList() {
  const { filters, sort, search, setFilter, setSort, setSearch } = useUrlParams()
  const { data, isLoading } = useProjekte()
  const navigate = useNavigate()
  const { listRef, saveScroll } = useScrollRestore('/projekte')
  const [deleteTarget, setDeleteTarget] = useState<string | null>(null)
  const deleteMutation = useDeleteProjekt()

  const handleRowClick = (id: string) => {
    saveScroll()
    navigate(`/projekte/${id}`)
  }

  if (isLoading) return <ListSkeleton columns={4} rows={8} />

  return (
    <div className="flex flex-col gap-4">
      {/* Header */}
      <PageHeader title="Projekte">
        <Button onClick={() => handleCreate(navigate)}>+ Neu</Button>
      </PageHeader>

      {/* Search + Filters */}
      <div className="flex items-center gap-2">
        <SearchInput value={search} onChange={setSearch} placeholder="Suche..." />
        <FilterSelect
          label="Status"
          value={filters.status}
          options={['entwurf', 'aktiv', 'abgeschlossen', 'archiviert']}
          onChange={(v) => setFilter('status', v)}
        />
      </div>

      {/* Table or Empty State */}
      {data?.data?.length === 0 ? (
        <EmptyState
          title="Keine Projekte vorhanden"
          description="Erstellen Sie Ihr erstes Projekt, um loszulegen."
          action={<Button onClick={() => handleCreate(navigate)}>+ Neues Projekt</Button>}
        />
      ) : (
        <div ref={listRef} className="overflow-auto">
          <DataTable
            columns={columns}
            data={data?.data ?? []}
            sort={sort}
            onSort={setSort}
            onRowClick={handleRowClick}
            onDelete={(id) => setDeleteTarget(id)}
          />
        </div>
      )}

      {/* Delete Confirmation */}
      <ConfirmDialog
        open={deleteTarget !== null}
        title="Projekt loeschen?"
        description="Dieser Vorgang kann nicht rueckgaengig gemacht werden."
        onConfirm={() => deleteMutation.mutate(deleteTarget!)}
        onCancel={() => setDeleteTarget(null)}
      />
    </div>
  )
}
```

## Column Definition

Columns are derived from schema fields where `list: true`. Format values according to `format` property using European conventions.

```typescript
const columns: ColumnDef<Projekt>[] = [
  { accessorKey: 'titel', header: 'Titel', sortable: true },
  { accessorKey: 'status', header: 'Status', sortable: true,
    cell: ({ value }) => <StatusBadge value={value} /> },
  { accessorKey: 'budget', header: 'Budget (EUR)', sortable: true,
    cell: ({ value }) => formatCurrency(value) },       // "50.000,00 EUR"
  { accessorKey: 'startdatum', header: 'Startdatum', sortable: true,
    cell: ({ value }) => formatDate(value) },            // "15.03.2026"
]
```

---

## See also

- `patterns/navigation.md` — useUrlParams, useScrollRestore
- `patterns/formatting.md` — formatValue for column rendering
- `patterns/navigation-layout.md` — AppShell, sidebar, bottom tabs
