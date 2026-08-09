# Formatting & Value Display

Single source of truth for displaying formatted values in lists, forms (read-only relation display), and any other read-only context.

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

## Field Rendering by Type

| Schema Type | Display |
|-------------|---------|
| `text` | Plain text |
| `textarea` | Plain text (preserves line breaks via `whitespace-pre-wrap`) |
| `enum` | Translated label if mapping exists, otherwise raw value |
| `number` | Formatted by `format` property (see European Formatting) |
| `date` | `DD.MM.YYYY` |
| `relation` | Linked name of related record (via `RelationLink`) |
| `boolean` | Badge: "Ja" (green) / "Nein" (gray) |

## `formatValue` Utility

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

## `FieldDisplay` Component

Renders a single field value based on type and format. Useful in lists (formatted columns) and forms (read-only display of relation values).

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

Usage example (e.g. in a detail-like layout or list cell):

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

## `RelationLink` Component

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

---

## See also

- `patterns/input-conventions.md` — input-side formatting (masks, parsing, field sizing)
- `schema-format.md` — field type definitions that drive formatting decisions
