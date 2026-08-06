# Detail View Pattern

Read-only view for a single entity record with field display, relation links, sub-lists, and delete action.

## Layout

```
Header: <- Zurueck              [Bearbeiten] [Trash]
Title:  [First text+required field value]

Fields:
  Feld-Label    Wert
  Feld-Label    Wert (clickable link for relations)
  Feld-Label    Wert

Sub-lists (if 1:n relations exist):
  Sub-Entity (count)              [+ Neu]
  ----------------------------------------
  Entry 1
  Entry 2
  "Alle anzeigen"
```

- Title is the value of the first field with `type: text` AND `required: true`
- Fields render in schema order
- Sub-lists appear below fields, one section per 1:n relation pointing TO this entity

## Field Rendering by Type

| Schema Type | Display |
|-------------|---------|
| `text` | Plain text |
| `textarea` | Plain text (preserves line breaks via `whitespace-pre-wrap`) |
| `enum` | Translated label if mapping exists, otherwise raw value |
| `number` | Formatted by `format` property (see European Formatting) |
| `date` | `DD.MM.YYYY` |
| `relation` | Linked name of related record, click navigates to related detail |
| `boolean` | Badge: "Ja" (green) / "Nein" (gray) |

## European Formatting Rules

All numbers and dates use German/European locale conventions:

| Format | Example | Rule |
|--------|---------|------|
| Date | `15.03.2026` | `DD.MM.YYYY` |
| Currency | `50.000 EUR` | Dot thousands, no decimals for whole numbers, space + EUR |
| Currency (cents) | `1.234,50 EUR` | Comma decimal when fractional |
| Number | `1.234.567` | Dot thousands separator |
| Decimal | `12,5` | Comma decimal separator |
| Percent | `12,5 %` | Value + space + % |

## Sub-Lists

Derived from 1:n relations: any entity whose schema has a `relation` field with `target` pointing to this entity's table.

- Header: `{SubEntity.label.plural} ({count})` + `[+ Neu]` button
- Compact list: 2-3 columns (fields where `list: true`), max 5 rows
- "Alle anzeigen" link navigates to `/entity/:id/sub-entity`
- "+ Neu" creates sub-entity with parent FK pre-filled via URL state

## Delete

- Trash icon button in header
- Opens `ConfirmDialog` with: "Eintrag loeschen? Diese Aktion kann nicht rueckgaengig gemacht werden."
- On confirm: delete via Supabase, invalidate queries, navigate to list
- Success: Toast "Eintrag geloescht"
- Error: Toast with error message

---

## Code

### `formatValue` Utility

European number/date formatting for display.

```typescript
import { format as formatDate, parseISO } from 'date-fns';

interface FieldDef {
  type: string;
  format?: 'currency' | 'percent';
}

export function formatValue(value: unknown, field: FieldDef): string {
  if (value === null || value === undefined) return '—';

  switch (field.type) {
    case 'date':
      return formatDate(parseISO(value as string), 'dd.MM.yyyy');

    case 'number': {
      const num = Number(value);
      if (field.format === 'currency') {
        const formatted = Number.isInteger(num)
          ? num.toLocaleString('de-DE', { minimumFractionDigits: 0 })
          : num.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
        return `${formatted} EUR`;
      }
      if (field.format === 'percent') {
        return `${num.toLocaleString('de-DE', { minimumFractionDigits: 0, maximumFractionDigits: 1 })} %`;
      }
      return num.toLocaleString('de-DE');
    }

    case 'boolean':
      return value ? 'Ja' : 'Nein';

    default:
      return String(value);
  }
}
```

### `useEntity` Hook

Fetches a single record by ID. Optionally fetches related data for sub-lists.

```typescript
import { useQuery } from '@tanstack/react-query';
import { supabase } from '@/lib/supabase';

interface UseEntityOptions {
  table: string;
  id: string;
  select?: string;            // e.g. '*, kontakte(name)' for relation joins
  subLists?: SubListDef[];    // 1:n relations to fetch counts + preview rows
}

interface SubListDef {
  table: string;
  fk: string;                 // foreign key column pointing to parent
  select: string;             // columns for preview rows
  limit?: number;             // default 5
}

export function useEntity({ table, id, select = '*', subLists = [] }: UseEntityOptions) {
  const entityQuery = useQuery({
    queryKey: [table, id],
    queryFn: async () => {
      const { data, error } = await supabase
        .from(table)
        .select(select)
        .eq('id', id)
        .single();
      if (error) throw error;
      return data;
    },
    enabled: !!id,
  });

  const subListQueries = subLists.map((sub) =>
    useQuery({
      queryKey: [sub.table, { [sub.fk]: id }],
      queryFn: async () => {
        const [countResult, rowsResult] = await Promise.all([
          supabase.from(sub.table).select('*', { count: 'exact', head: true }).eq(sub.fk, id),
          supabase.from(sub.table).select(sub.select).eq(sub.fk, id).limit(sub.limit ?? 5),
        ]);
        if (countResult.error) throw countResult.error;
        if (rowsResult.error) throw rowsResult.error;
        return { count: countResult.count ?? 0, rows: rowsResult.data };
      },
      enabled: !!id,
    })
  );

  return { entityQuery, subListQueries };
}
```

### `FieldDisplay` Component

Renders a single field value based on type and format.

```typescript
import { Badge } from '@/components/ui/badge';
import { formatValue } from '@/lib/format';
import { RelationLink } from './RelationLink';

interface FieldDisplayProps {
  field: FieldDef;
  value: unknown;
  record: Record<string, unknown>;   // full record for relation lookups
}

export function FieldDisplay({ field, value, record }: FieldDisplayProps) {
  if (value === null || value === undefined) {
    return <span className="text-muted-foreground">—</span>;
  }

  if (field.type === 'relation') {
    return (
      <RelationLink
        targetTable={field.target!}
        targetId={value as string}
      />
    );
  }

  if (field.type === 'boolean') {
    return (
      <Badge variant={value ? 'default' : 'secondary'}>
        {value ? 'Ja' : 'Nein'}
      </Badge>
    );
  }

  if (field.type === 'textarea') {
    return <span className="whitespace-pre-wrap">{formatValue(value, field)}</span>;
  }

  return <span>{formatValue(value, field)}</span>;
}
```

Usage in the detail view:

```tsx
<dl className="grid grid-cols-[auto_1fr] gap-x-6 gap-y-4">
  {fields.map((field) => (
    <Fragment key={field.name}>
      <dt className="text-sm font-medium text-muted-foreground">{field.label}</dt>
      <dd className="text-sm">
        <FieldDisplay field={field} value={record[field.name]} record={record} />
      </dd>
    </Fragment>
  ))}
</dl>
```

### `RelationLink` Component

Fetches the display name of a related record and renders a navigable link.

```typescript
import { useQuery } from '@tanstack/react-query';
import { Link, useLocation } from 'react-router-dom';
import { supabase } from '@/lib/supabase';
import { Skeleton } from '@/components/ui/skeleton';

interface RelationLinkProps {
  targetTable: string;
  targetId: string;
  displayField?: string;   // defaults to first text+required field of target schema
}

export function RelationLink({ targetTable, targetId, displayField = 'name' }: RelationLinkProps) {
  const location = useLocation();

  const { data, isLoading } = useQuery({
    queryKey: [targetTable, targetId, 'title'],
    queryFn: async () => {
      const { data, error } = await supabase
        .from(targetTable)
        .select(displayField)
        .eq('id', targetId)
        .single();
      if (error) throw error;
      return data;
    },
    enabled: !!targetId,
    staleTime: 5 * 60 * 1000,   // relation titles rarely change
  });

  if (isLoading) return <Skeleton className="h-4 w-24" />;

  const title = data?.[displayField] ?? targetId;

  return (
    <Link
      to={`/${targetTable}/${targetId}`}
      state={{ from: location.pathname }}
      className="text-primary underline-offset-4 hover:underline"
    >
      {title}
    </Link>
  );
}
```

### `useDeleteEntity` Mutation

Delete with confirmation flow, query invalidation, and navigation.

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useNavigate, useLocation } from 'react-router-dom';
import { supabase } from '@/lib/supabase';
import { toast } from 'sonner';
import { useState } from 'react';

interface UseDeleteEntityOptions {
  table: string;
  id: string;
  listPath: string;          // e.g. '/projekte'
}

export function useDeleteEntity({ table, id, listPath }: UseDeleteEntityOptions) {
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  const location = useLocation();
  const [showConfirm, setShowConfirm] = useState(false);

  const mutation = useMutation({
    mutationFn: async () => {
      const { error } = await supabase.from(table).delete().eq('id', id);
      if (error) throw error;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: [table] });
      toast.success('Eintrag geloescht');
      const backPath = location.state?.from ?? listPath;
      navigate(backPath, { replace: true });
    },
    onError: (error: Error) => {
      toast.error(`Fehler beim Loeschen: ${error.message}`);
    },
  });

  const requestDelete = () => setShowConfirm(true);
  const cancelDelete = () => setShowConfirm(false);
  const confirmDelete = () => {
    setShowConfirm(false);
    mutation.mutate();
  };

  return { requestDelete, cancelDelete, confirmDelete, showConfirm, isDeleting: mutation.isPending };
}
```

Usage with `ConfirmDialog`:

```tsx
const { requestDelete, cancelDelete, confirmDelete, showConfirm, isDeleting } = useDeleteEntity({
  table: 'projekte',
  id: params.id,
  listPath: '/projekte',
});

<ConfirmDialog
  open={showConfirm}
  onCancel={cancelDelete}
  onConfirm={confirmDelete}
  title="Eintrag loeschen"
  description="Eintrag loeschen? Diese Aktion kann nicht rueckgaengig gemacht werden."
  confirmLabel="Loeschen"
  loading={isDeleting}
  variant="destructive"
/>
```

### Full Page Structure

```tsx
import { useParams, useNavigate, useLocation } from 'react-router-dom';
import { ArrowLeft, Pencil, Trash2 } from 'lucide-react';
import { Button } from '@/components/ui/button';

export function EntityDetail({ schema, subListDefs }: EntityDetailProps) {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();
  const location = useLocation();

  const { entityQuery, subListQueries } = useEntity({
    table: schema.table,
    id: id!,
    subLists: subListDefs,
  });

  const { requestDelete, cancelDelete, confirmDelete, showConfirm, isDeleting } = useDeleteEntity({
    table: schema.table,
    id: id!,
    listPath: `/${schema.table}`,
  });

  if (entityQuery.isLoading) return <DetailSkeleton />;
  if (entityQuery.isError) return <ErrorState error={entityQuery.error} />;

  const record = entityQuery.data;
  const titleField = schema.fields.find((f) => f.type === 'text' && f.required);
  const title = titleField ? record[titleField.name] : record.id;

  const goBack = () => {
    const from = location.state?.from;
    navigate(from ?? `/${schema.table}`);
  };

  return (
    <div className="mx-auto max-w-2xl space-y-8 p-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <Button variant="ghost" size="sm" onClick={goBack}>
          <ArrowLeft className="mr-2 h-4 w-4" />
          Zurueck
        </Button>
        <div className="flex gap-2">
          <Button variant="outline" size="sm" onClick={() => navigate('edit')}>
            <Pencil className="mr-2 h-4 w-4" />
            Bearbeiten
          </Button>
          <Button variant="ghost" size="icon" onClick={requestDelete}>
            <Trash2 className="h-4 w-4" />
          </Button>
        </div>
      </div>

      {/* Title */}
      <h1 className="text-2xl font-semibold">{title}</h1>

      {/* Fields */}
      <dl className="grid grid-cols-[auto_1fr] gap-x-6 gap-y-4">
        {schema.fields.map((field) => (
          <Fragment key={field.name}>
            <dt className="text-sm font-medium text-muted-foreground">{field.label}</dt>
            <dd className="text-sm">
              <FieldDisplay field={field} value={record[field.name]} record={record} />
            </dd>
          </Fragment>
        ))}
      </dl>

      {/* Sub-lists */}
      {subListDefs.map((sub, i) => {
        const query = subListQueries[i];
        return (
          <section key={sub.table} className="space-y-3">
            <div className="flex items-center justify-between">
              <h2 className="text-lg font-medium">
                {sub.label} {query.data ? `(${query.data.count})` : ''}
              </h2>
              <Button
                variant="outline"
                size="sm"
                onClick={() => navigate(`${sub.table}/new`, { state: { [sub.fk]: id } })}
              >
                + Neu
              </Button>
            </div>
            <Separator />
            {query.isLoading && <Skeleton className="h-20 w-full" />}
            {query.data?.rows.map((row) => (
              <SubListRow key={row.id} row={row} table={sub.table} fields={sub.fields} />
            ))}
            {(query.data?.count ?? 0) > (sub.limit ?? 5) && (
              <Link
                to={`/${schema.table}/${id}/${sub.table}`}
                className="text-sm text-primary hover:underline"
              >
                Alle anzeigen
              </Link>
            )}
          </section>
        );
      })}

      <ConfirmDialog
        open={showConfirm}
        onCancel={cancelDelete}
        onConfirm={confirmDelete}
        title="Eintrag loeschen"
        description="Eintrag loeschen? Diese Aktion kann nicht rueckgaengig gemacht werden."
        confirmLabel="Loeschen"
        loading={isDeleting}
        variant="destructive"
      />
    </div>
  );
}
```
