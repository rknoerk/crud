# Form / Edit Pattern

Form view for creating and editing entity records. Uses React Hook Form + Zod validation, shadcn/ui widgets, European input formats.

## Layout

```
Header: <- Abbrechen                    [Speichern]
Fields (in schema order):
  Label *
  [Input/Select/Datepicker/...]

  Label
  [Input/Select/Datepicker/...]

Sub-lists (if 1:n relations exist):
  Sub-Entity (count)              [+ Neu]
  Entry 1
  Entry 2

Footer:
  [Loeschen]
```

- Header is sticky at top
- "Abbrechen" navigates back (with unsaved changes guard if dirty)
- "Speichern" submits the form, disabled while submitting
- "Loeschen" — bottom left, below all fields and sub-lists. Destructive button (red/outline, text only, no icon). Only shown for existing records, not for new ones. Opens ConfirmDialog: "Eintrag loeschen? Diese Aktion kann nicht rueckgaengig gemacht werden." On confirm: hard delete via Supabase, invalidate queries, toast, navigate back to list.
- Sub-lists for 1:n relations appear below the form fields (read-only list with count, "+ Neu" button, "Alle anzeigen" link)
- Required fields show `*` after the label
- Fields render in schema order, full width, stacked vertically
- Validation errors appear inline below each field in red

## Widget Mapping

| Schema Type | Component | Notes |
|-------------|-----------|-------|
| `text` | `<Input />` | Standard text input |
| `textarea` | `<Textarea />` | Auto-grows, preserves line breaks |
| `enum` | `<Select>` | Options from `values` array |
| `number` | `<Input type="text" inputMode="decimal" />` | European formatting (comma decimal), parsed on blur |
| `date` | `<DatePicker />` | DD.MM.YYYY format |
| `relation` | `<Combobox />` | Searchable, loads options from target table |
| `boolean` | `<Switch />` | With inline label |

## Zod Schema Generation

Generate a Zod schema from YAML field definitions. One file per entity in `schemas/`.

### `generateZodSchema`

```typescript
import { z, ZodTypeAny } from 'zod';

interface FieldDef {
  name: string;
  type: string;
  required?: boolean;
  values?: string[];
  target?: string;
}

export function generateZodSchema(fields: FieldDef[]) {
  const shape: Record<string, ZodTypeAny> = {};

  for (const field of fields) {
    let schema: ZodTypeAny;

    switch (field.type) {
      case 'text':
        schema = z.string().min(1, 'Pflichtfeld');
        break;
      case 'textarea':
        schema = z.string().min(1, 'Pflichtfeld');
        break;
      case 'enum':
        schema = z.enum(field.values as [string, ...string[]]);
        break;
      case 'number':
        schema = z.number({ invalid_type_error: 'Bitte eine Zahl eingeben' });
        break;
      case 'date':
        schema = z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Ungueltiges Datum');
        break;
      case 'relation':
        schema = z.string().uuid('Bitte einen Eintrag waehlen');
        break;
      case 'boolean':
        schema = z.boolean();
        break;
      default:
        schema = z.string();
    }

    if (!field.required) {
      schema = schema.optional().nullable();
    }

    shape[field.name] = schema;
  }

  return z.object(shape);
}
```

### Example: `projekt` Entity Schema

Generated from the `projekt` YAML definition:

```typescript
// schemas/projekt.ts
import { z } from 'zod';

export const projektSchema = z.object({
  titel: z.string().min(1, 'Pflichtfeld'),
  status: z.enum(['entwurf', 'aktiv', 'abgeschlossen', 'archiviert']),
  kunde: z.string().uuid('Bitte einen Eintrag waehlen').optional().nullable(),
  budget: z.number({ invalid_type_error: 'Bitte eine Zahl eingeben' }).optional().nullable(),
  notizen: z.string().optional().nullable(),
  startdatum: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Ungueltiges Datum').optional().nullable(),
});

export type ProjektFormValues = z.infer<typeof projektSchema>;
```

## React Hook Form Integration

### Form Setup

```typescript
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

export function EntityForm({ schema, record }: EntityFormProps) {
  const { id } = useParams<{ id: string }>();
  const goBack = useNavigateBack();

  const form = useForm<FormValues>({
    resolver: zodResolver(entityZodSchema),
    defaultValues: record ?? {},  // existing record for edit, empty for create (defaults in DB)
  });

  const { control, handleSubmit, formState: { errors, isDirty, isSubmitting } } = form;

  const updateMutation = useUpdateEntity({ table: schema.table, id: id! });

  const onSubmit = (data: FormValues) => {
    updateMutation.mutate(data);
  };

  return (
    <>
      <UnsavedChangesGuard isDirty={isDirty} />
      <form onSubmit={handleSubmit(onSubmit)} className="mx-auto max-w-2xl space-y-8 p-6">
        {/* Header */}
        <div className="flex items-center justify-between">
          <Button type="button" variant="ghost" size="sm" onClick={goBack}>
            <ArrowLeft className="mr-2 h-4 w-4" />
            Abbrechen
          </Button>
          <Button type="submit" size="sm" disabled={isSubmitting}>
            {isSubmitting ? <Loader2 className="mr-2 h-4 w-4 animate-spin" /> : null}
            Speichern
          </Button>
        </div>

        {/* Fields in schema order */}
        <div className="space-y-6">
          {schema.fields.map((field) => (
            <FormFieldWidget
              key={field.name}
              field={field}
              control={control}
              errors={errors}
            />
          ))}
        </div>
      </form>
    </>
  );
}
```

### `FormFieldWidget` Component

Renders the correct widget for each field type with label, error message, and Controller wrapper.

```tsx
import { Controller, Control, FieldErrors } from 'react-hook-form';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Switch } from '@/components/ui/switch';
import { Label } from '@/components/ui/label';
import { DatePicker } from '@/components/ui/date-picker';
import { RelationCombobox } from '@/components/RelationCombobox';
import { EuropeanNumberInput } from '@/components/EuropeanNumberInput';

interface FormFieldWidgetProps {
  field: FieldDef;
  control: Control<any>;
  errors: FieldErrors;
}

export function FormFieldWidget({ field, control, errors }: FormFieldWidgetProps) {
  const error = errors[field.name];
  const isRequired = field.required === true;

  return (
    <div className="space-y-2">
      <Label htmlFor={field.name}>
        {field.label}
        {isRequired && <span className="text-destructive ml-1">*</span>}
      </Label>

      <Controller
        name={field.name}
        control={control}
        render={({ field: rhfField }) => {
          switch (field.type) {
            case 'text':
              return (
                <Input
                  id={field.name}
                  {...rhfField}
                  value={rhfField.value ?? ''}
                />
              );

            case 'textarea':
              return (
                <Textarea
                  id={field.name}
                  {...rhfField}
                  value={rhfField.value ?? ''}
                  rows={4}
                />
              );

            case 'enum':
              return (
                <Select
                  value={rhfField.value ?? ''}
                  onValueChange={rhfField.onChange}
                >
                  <SelectTrigger id={field.name}>
                    <SelectValue placeholder="Bitte waehlen..." />
                  </SelectTrigger>
                  <SelectContent>
                    {field.values!.map((v) => (
                      <SelectItem key={v} value={v}>
                        {v}
                      </SelectItem>
                    ))}
                  </SelectContent>
                </Select>
              );

            case 'number':
              return (
                <EuropeanNumberInput
                  id={field.name}
                  value={rhfField.value}
                  onChange={rhfField.onChange}
                  format={field.format}
                />
              );

            case 'date':
              return (
                <DatePicker
                  id={field.name}
                  value={rhfField.value}
                  onChange={rhfField.onChange}
                />
              );

            case 'relation':
              return (
                <RelationCombobox
                  id={field.name}
                  targetTable={field.target!}
                  value={rhfField.value}
                  onChange={rhfField.onChange}
                />
              );

            case 'boolean':
              return (
                <div className="flex items-center gap-2">
                  <Switch
                    id={field.name}
                    checked={rhfField.value ?? false}
                    onCheckedChange={rhfField.onChange}
                  />
                </div>
              );

            default:
              return <Input id={field.name} {...rhfField} value={rhfField.value ?? ''} />;
          }
        }}
      />

      {error && (
        <p className="text-sm text-destructive">{error.message as string}</p>
      )}
    </div>
  );
}
```

## Mutations

### `useUpdateEntity`

Updates an existing record, invalidates queries, shows toast, navigates to detail.

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useNavigate } from 'react-router-dom';
import { supabase } from '@/lib/supabase';
import { toast } from 'sonner';

interface UseUpdateEntityOptions {
  table: string;
  id: string;
}

export function useUpdateEntity({ table, id }: UseUpdateEntityOptions) {
  const queryClient = useQueryClient();
  const navigate = useNavigate();

  return useMutation({
    mutationFn: async (data: Record<string, unknown>) => {
      const { error } = await supabase
        .from(table)
        .update(data)
        .eq('id', id);
      if (error) throw error;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: [table, id] });
      queryClient.invalidateQueries({ queryKey: [table, 'list'] });
      toast.success('Gespeichert');
      navigate(`/${table}/${id}`);
    },
    onError: (error: Error) => {
      toast.error(`Fehler beim Speichern: ${error.message}`);
    },
  });
}
```

### `useCreateEntity`

Inserts a new record with schema defaults, returns new ID for navigation to edit.

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useNavigate, useLocation } from 'react-router-dom';
import { supabase } from '@/lib/supabase';
import { toast } from 'sonner';

interface UseCreateEntityOptions {
  table: string;
  defaults?: Record<string, unknown>;  // from schema `default` values
}

export function useCreateEntity({ table, defaults = {} }: UseCreateEntityOptions) {
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  const location = useLocation();

  return useMutation({
    mutationFn: async (extraData?: Record<string, unknown>) => {
      const { data, error } = await supabase
        .from(table)
        .insert({ ...defaults, ...extraData })
        .select('id')
        .single();
      if (error) throw error;
      return data;
    },
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: [table, 'list'] });
      navigate(`/${table}/${data.id}/edit`, {
        state: { from: location.pathname },
      });
    },
    onError: (error: Error) => {
      toast.error(`Fehler beim Erstellen: ${error.message}`);
    },
  });
}
```

### `useDeleteEntity`

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

Usage with `ConfirmDialog` (placed at the bottom of the form, only for existing records):

```tsx
const { requestDelete, cancelDelete, confirmDelete, showConfirm, isDeleting } = useDeleteEntity({
  table: 'projekte',
  id: params.id,
  listPath: '/projekte',
});

// In the form footer (below fields and sub-lists), only for existing records:
{id && (
  <Button type="button" variant="outline" className="text-destructive" onClick={requestDelete}>
    Loeschen
  </Button>
)}

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

## Unsaved Changes Guard

See `patterns/navigation.md` for the `UnsavedChangesGuard` component.

## European Input Formats

### `EuropeanNumberInput`

Displays and accepts numbers in European format (comma decimal, dot thousands). Stores the raw numeric value internally.

```tsx
import { useState, useCallback } from 'react';
import { Input } from '@/components/ui/input';

interface EuropeanNumberInputProps {
  id?: string;
  value: number | null | undefined;
  onChange: (value: number | null) => void;
  format?: 'currency' | 'percent';
}

/**
 * Parses a European-formatted string "1.234,56" into a number 1234.56.
 */
function parseEuropeanNumber(input: string): number | null {
  const cleaned = input
    .replace(/\s/g, '')       // remove whitespace
    .replace(/\./g, '')       // remove dot (thousands separator)
    .replace(',', '.');       // comma -> decimal point
  const num = Number(cleaned);
  return Number.isNaN(num) ? null : num;
}

/**
 * Formats a number into European display format.
 */
function formatEuropeanNumber(value: number | null | undefined, format?: string): string {
  if (value === null || value === undefined) return '';
  const options: Intl.NumberFormatOptions =
    format === 'currency'
      ? { minimumFractionDigits: 2, maximumFractionDigits: 2 }
      : {};
  return value.toLocaleString('de-DE', options);
}

export function EuropeanNumberInput({ id, value, onChange, format }: EuropeanNumberInputProps) {
  const [displayValue, setDisplayValue] = useState(() => formatEuropeanNumber(value, format));
  const [isFocused, setIsFocused] = useState(false);

  const handleFocus = useCallback(() => {
    setIsFocused(true);
    // Show raw value without thousands separators for easier editing
    if (value !== null && value !== undefined) {
      setDisplayValue(String(value).replace('.', ','));
    }
  }, [value]);

  const handleBlur = useCallback(() => {
    setIsFocused(false);
    const parsed = parseEuropeanNumber(displayValue);
    onChange(parsed);
    setDisplayValue(formatEuropeanNumber(parsed, format));
  }, [displayValue, onChange, format]);

  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    setDisplayValue(e.target.value);
  }, []);

  return (
    <div className="relative">
      <Input
        id={id}
        type="text"
        inputMode="decimal"
        value={displayValue}
        onChange={handleChange}
        onFocus={handleFocus}
        onBlur={handleBlur}
        className={format === 'currency' ? 'pr-10' : undefined}
      />
      {format === 'currency' && (
        <span className="pointer-events-none absolute right-3 top-1/2 -translate-y-1/2 text-muted-foreground">
          EUR
        </span>
      )}
    </div>
  );
}
```

### `DatePicker`

Date picker using DD.MM.YYYY format. Stores ISO date strings (YYYY-MM-DD) internally.

```tsx
import { useState, useCallback } from 'react';
import { format as formatDate, parse, isValid } from 'date-fns';
import { de } from 'date-fns/locale';
import { CalendarIcon } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Calendar } from '@/components/ui/calendar';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { Input } from '@/components/ui/input';
import { cn } from '@/lib/utils';

interface DatePickerProps {
  id?: string;
  value: string | null | undefined;   // ISO date string YYYY-MM-DD
  onChange: (value: string | null) => void;
}

export function DatePicker({ id, value, onChange }: DatePickerProps) {
  const [inputValue, setInputValue] = useState(() =>
    value ? formatDate(new Date(value), 'dd.MM.yyyy') : ''
  );
  const [open, setOpen] = useState(false);

  const selectedDate = value ? new Date(value) : undefined;

  const handleInputBlur = useCallback(() => {
    if (!inputValue.trim()) {
      onChange(null);
      return;
    }
    const parsed = parse(inputValue, 'dd.MM.yyyy', new Date());
    if (isValid(parsed)) {
      onChange(formatDate(parsed, 'yyyy-MM-dd'));
      setInputValue(formatDate(parsed, 'dd.MM.yyyy'));
    } else {
      // Reset to previous valid value
      setInputValue(value ? formatDate(new Date(value), 'dd.MM.yyyy') : '');
    }
  }, [inputValue, value, onChange]);

  const handleCalendarSelect = useCallback(
    (date: Date | undefined) => {
      if (date) {
        onChange(formatDate(date, 'yyyy-MM-dd'));
        setInputValue(formatDate(date, 'dd.MM.yyyy'));
      } else {
        onChange(null);
        setInputValue('');
      }
      setOpen(false);
    },
    [onChange],
  );

  return (
    <div className="flex gap-2">
      <Input
        id={id}
        placeholder="TT.MM.JJJJ"
        value={inputValue}
        onChange={(e) => setInputValue(e.target.value)}
        onBlur={handleInputBlur}
        className="flex-1"
      />
      <Popover open={open} onOpenChange={setOpen}>
        <PopoverTrigger asChild>
          <Button type="button" variant="outline" size="icon" className="shrink-0">
            <CalendarIcon className="h-4 w-4" />
          </Button>
        </PopoverTrigger>
        <PopoverContent className="w-auto p-0" align="end">
          <Calendar
            mode="single"
            selected={selectedDate}
            onSelect={handleCalendarSelect}
            locale={de}
            initialFocus
          />
        </PopoverContent>
      </Popover>
    </div>
  );
}
```

### `RelationCombobox`

Searchable combobox that loads options from the target table. Uses debounced search against Supabase.

```tsx
import { useState, useMemo } from 'react';
import { useQuery } from '@tanstack/react-query';
import { Check, ChevronsUpDown } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Command, CommandEmpty, CommandGroup, CommandInput, CommandItem, CommandList } from '@/components/ui/command';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { supabase } from '@/lib/supabase';
import { cn } from '@/lib/utils';
import { useDebounce } from '@/hooks/useDebounce';

interface RelationComboboxProps {
  id?: string;
  targetTable: string;
  value: string | null | undefined;
  onChange: (value: string | null) => void;
  displayField?: string;    // defaults to first text+required field of target
}

export function RelationCombobox({
  id,
  targetTable,
  value,
  onChange,
  displayField = 'name',
}: RelationComboboxProps) {
  const [open, setOpen] = useState(false);
  const [search, setSearch] = useState('');
  const debouncedSearch = useDebounce(search, 300);

  // Fetch options based on search term
  const { data: options = [] } = useQuery({
    queryKey: [targetTable, 'combobox', debouncedSearch],
    queryFn: async () => {
      let query = supabase
        .from(targetTable)
        .select(`id, ${displayField}`)
        .order(displayField)
        .limit(50);

      if (debouncedSearch) {
        query = query.ilike(displayField, `%${debouncedSearch}%`);
      }

      const { data, error } = await query;
      if (error) throw error;
      return data as Array<{ id: string; [key: string]: string }>;
    },
    staleTime: 30_000,
  });

  // Fetch the display label for the currently selected value
  const { data: selectedRecord } = useQuery({
    queryKey: [targetTable, value, 'label'],
    queryFn: async () => {
      const { data, error } = await supabase
        .from(targetTable)
        .select(`id, ${displayField}`)
        .eq('id', value!)
        .single();
      if (error) throw error;
      return data;
    },
    enabled: !!value,
    staleTime: 5 * 60_000,
  });

  const selectedLabel = selectedRecord?.[displayField] ?? '';

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button
          id={id}
          type="button"
          variant="outline"
          role="combobox"
          aria-expanded={open}
          className="w-full justify-between font-normal"
        >
          {selectedLabel || <span className="text-muted-foreground">Bitte waehlen...</span>}
          <ChevronsUpDown className="ml-2 h-4 w-4 shrink-0 opacity-50" />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[--radix-popover-trigger-width] p-0" align="start">
        <Command shouldFilter={false}>
          <CommandInput
            placeholder="Suchen..."
            value={search}
            onValueChange={setSearch}
          />
          <CommandList>
            <CommandEmpty>Keine Ergebnisse.</CommandEmpty>
            <CommandGroup>
              {options.map((option) => (
                <CommandItem
                  key={option.id}
                  value={option.id}
                  onSelect={(selectedId) => {
                    onChange(selectedId === value ? null : selectedId);
                    setOpen(false);
                  }}
                >
                  <Check
                    className={cn('mr-2 h-4 w-4', value === option.id ? 'opacity-100' : 'opacity-0')}
                  />
                  {option[displayField]}
                </CommandItem>
              ))}
            </CommandGroup>
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  );
}
```

## Sub-Lists (1:n Relations below the Form)

When an entity has 1:n relations (other entities with a foreign key pointing to this entity), render them as read-only sub-lists below the form fields. This replaces the former detail view's sub-list section.

### `SubListDef` Interface

```typescript
interface SubListDef {
  table: string;
  fk: string;                 // foreign key column pointing to parent
  select: string;             // columns for preview rows
  label: string;              // section header label (plural)
  fields: FieldDef[];         // field definitions for rendering preview columns
  limit?: number;             // max preview rows, default 5
}
```

### `SubList` Component

```tsx
import { useQuery } from '@tanstack/react-query';
import { Link, useNavigate } from 'react-router-dom';
import { supabase } from '@/lib/supabase';
import { Separator } from '@/components/ui/separator';
import { Skeleton } from '@/components/ui/skeleton';
import { Button } from '@/components/ui/button';

interface SubListProps {
  parentTable: string;
  parentId: string;
  def: SubListDef;
}

export function SubList({ parentTable, parentId, def }: SubListProps) {
  const navigate = useNavigate();

  const { data, isLoading } = useQuery({
    queryKey: [def.table, { [def.fk]: parentId }],
    queryFn: async () => {
      const [countResult, rowsResult] = await Promise.all([
        supabase.from(def.table).select('*', { count: 'exact', head: true }).eq(def.fk, parentId),
        supabase.from(def.table).select(def.select).eq(def.fk, parentId).limit(def.limit ?? 5),
      ]);
      if (countResult.error) throw countResult.error;
      if (rowsResult.error) throw rowsResult.error;
      return { count: countResult.count ?? 0, rows: rowsResult.data };
    },
    enabled: !!parentId,
  });

  return (
    <section className="space-y-3">
      <div className="flex items-center justify-between">
        <h2 className="text-lg font-medium">
          {def.label} {data ? `(${data.count})` : ''}
        </h2>
        <Button
          type="button"
          variant="outline"
          size="sm"
          onClick={() => navigate(`/${def.table}/new`, { state: { [def.fk]: parentId } })}
        >
          + Neu
        </Button>
      </div>
      <Separator />
      {isLoading && <Skeleton className="h-20 w-full" />}
      {data?.rows.map((row) => (
        <SubListRow key={row.id} row={row} table={def.table} fields={def.fields} />
      ))}
      {(data?.count ?? 0) > (def.limit ?? 5) && (
        <Link
          to={`/${parentTable}/${parentId}/${def.table}`}
          className="text-sm text-primary hover:underline"
        >
          Alle anzeigen
        </Link>
      )}
    </section>
  );
}
```

Place sub-lists below the form fields and above the delete button.

## Full Page Structure (Projekt Example)

```tsx
import { useParams } from 'react-router-dom';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { ArrowLeft, Loader2 } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { useEntity } from '@/hooks/useEntity';
import { useUpdateEntity } from '@/hooks/useUpdateEntity';
import { useNavigateBack } from '@/hooks/useNavigateBack';
import { UnsavedChangesGuard } from '@/components/UnsavedChangesGuard';
import { FormFieldWidget } from '@/components/FormFieldWidget';
import { projektSchema, ProjektFormValues } from '@/schemas/projekt';
import { projektFields } from '@/lib/entity-registry';

export function ProjektForm() {
  const { id } = useParams<{ id: string }>();
  const goBack = useNavigateBack();

  const { entityQuery } = useEntity({ table: 'projekte', id: id! });
  const updateMutation = useUpdateEntity({ table: 'projekte', id: id! });

  const form = useForm<ProjektFormValues>({
    resolver: zodResolver(projektSchema),
    defaultValues: entityQuery.data ?? {},
  });

  const { control, handleSubmit, formState: { errors, isDirty, isSubmitting } } = form;

  if (entityQuery.isLoading) return <FormSkeleton fields={projektFields.length} />;
  if (entityQuery.isError) return <ErrorState error={entityQuery.error} />;

  return (
    <>
      <UnsavedChangesGuard isDirty={isDirty} />
      <form
        onSubmit={handleSubmit((data) => updateMutation.mutate(data))}
        className="mx-auto max-w-2xl space-y-8 p-6"
      >
        {/* Header */}
        <div className="flex items-center justify-between">
          <Button type="button" variant="ghost" size="sm" onClick={goBack}>
            <ArrowLeft className="mr-2 h-4 w-4" />
            Abbrechen
          </Button>
          <Button type="submit" size="sm" disabled={isSubmitting}>
            {isSubmitting && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
            Speichern
          </Button>
        </div>

        {/* Fields */}
        <div className="space-y-6">
          {projektFields.map((field) => (
            <FormFieldWidget
              key={field.name}
              field={field}
              control={control}
              errors={errors}
            />
          ))}
        </div>
      </form>
    </>
  );
}
```

---

## See also

- `patterns/navigation.md` — UnsavedChangesGuard, useNavigateBack
- `patterns/formatting.md` — FieldDisplay for read-only relation display, formatValue
- `patterns/input-conventions.md` — masks, parsing, field sizing
