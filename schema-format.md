# YAML Schema Format Reference

This file defines the YAML schema format for entity definitions used to generate CRUD apps.

## Entity-Level Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `entity` | string | yes | Entity name in singular, lowercase, English (e.g. `projekt`) |
| `table` | string | yes | Supabase table name (e.g. `projekte`) |
| `label.singular` | string | yes | German UI label, singular |
| `label.plural` | string | yes | German UI label, plural |
| `default_sort` | object | no | `{ field, direction: asc\|desc }` |

## Field Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | string | *required* | Field name, English, snake_case |
| `type` | string | *required* | One of: `text`, `textarea`, `enum`, `number`, `date`, `time`, `datetime`, `relation`, `boolean` |
| `label` | string | *required* | German UI label |
| `required` | boolean | `false` | Field is required |
| `default` | any | — | Default value |
| `list` | boolean | `false` | Show in list view |
| `sortable` | boolean | `false` | Sortable column in list |
| `filterable` | boolean | `false` | Filter option in list |
| `searchable` | boolean | `false` | Included in text search |
| `format` | string | — | Display format: `currency`, `percent`, `date` |

**Type-specific properties:**

- `values` (array) — required for `enum`, defines allowed values
- `target` (string) — required for `relation`, references the target table name

## Type Mapping

| Schema Type | DB Type | UI Widget | Zod |
|-------------|---------|-----------|-----|
| text | varchar | Input | z.string() |
| textarea | text | Textarea | z.string() |
| enum | Postgres ENUM | Select | z.enum([...]) |
| number | numeric | Number Input | z.number() |
| date | date | Masked date input (DD.MM.YYYY) | z.string().date() |
| time | time | Masked time input (HH:MM) | z.string().regex(/^\d{2}:\d{2}$/) |
| datetime | timestamptz | Date mask + time mask | z.string().datetime() |
| relation | uuid REFERENCES | Combobox with search | z.string().uuid() |
| boolean | boolean | Toggle/Switch | z.boolean() |

## Auto-Generated Fields

These fields are created automatically and must NOT appear in the schema:

- `id` — uuid, primary key
- `created_at` — timestamptz, set on insert
- `updated_at` — timestamptz, updated via trigger

## Complete Example

```yaml
entity: projekt
table: projekte
label:
  singular: Projekt
  plural: Projekte

fields:
  - name: titel
    type: text
    label: Titel
    required: true
    list: true
    sortable: true
    searchable: true

  - name: status
    type: enum
    label: Status
    values: [entwurf, aktiv, abgeschlossen, archiviert]
    default: entwurf
    list: true
    sortable: true
    filterable: true

  - name: kunde
    type: relation
    target: kontakte
    label: Kunde
    list: true

  - name: budget
    type: number
    label: Budget (EUR)
    format: currency
    list: true
    sortable: true

  - name: notizen
    type: textarea
    label: Notizen

  - name: startdatum
    type: date
    label: Startdatum
    list: true
    sortable: true

default_sort: { field: startdatum, direction: desc }
```
