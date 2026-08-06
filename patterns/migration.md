# Migration Pattern

## File Naming

```
supabase/migrations/YYYYMMDDHHMMSS_create_{table}.sql
```

Timestamp format matches Supabase CLI convention (`supabase migration new` generates this).

## Type Mapping

| Schema Type | SQL Type | Constraints |
|-------------|----------|-------------|
| text | varchar | NOT NULL if required |
| textarea | text | NOT NULL if required |
| enum | custom ENUM type | NOT NULL if required, DEFAULT if specified |
| number | numeric | NOT NULL if required |
| date | date | NOT NULL if required |
| relation | uuid | REFERENCES target(id) ON DELETE SET NULL, NOT NULL if required |
| boolean | boolean | DEFAULT false, NOT NULL if required |

## Auto-Generated Columns

Every table gets these columns automatically — they must NOT appear in the YAML schema:

```sql
id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
created_at timestamptz DEFAULT now() NOT NULL,
updated_at timestamptz DEFAULT now() NOT NULL
```

The `updated_at` trigger function (create once, reuse across tables):

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Attach per table:

```sql
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON {table}
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();
```

## Enum Creation

Create the enum type before the table. Naming convention: `{table}_{field}`.

```sql
CREATE TYPE projekte_status AS ENUM ('entwurf', 'aktiv', 'abgeschlossen', 'archiviert');
```

If a table has multiple enum fields, create one type per field.

## Indexes

Create indexes on:

- All fields with `sortable: true`
- All fields with `filterable: true`
- All `relation` fields (foreign key lookups)

Format:

```sql
CREATE INDEX idx_{table}_{field} ON {table}({field});
```

A field that is both `sortable` and `filterable` gets one index (no duplicates).

## Complete Example

Full migration for the `projekt` entity (table `projekte`) from the design doc.

**File:** `supabase/migrations/20260806120000_create_projekte.sql`

```sql
-- Enum types
CREATE TYPE projekte_status AS ENUM ('entwurf', 'aktiv', 'abgeschlossen', 'archiviert');

-- Table
CREATE TABLE projekte (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  titel varchar NOT NULL,
  status projekte_status NOT NULL DEFAULT 'entwurf',
  kunde uuid REFERENCES kontakte(id) ON DELETE SET NULL,
  budget numeric,
  notizen text,
  startdatum date,
  created_at timestamptz DEFAULT now() NOT NULL,
  updated_at timestamptz DEFAULT now() NOT NULL
);

-- Updated-at trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON projekte
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();

-- Indexes: sortable, filterable, and relation fields
CREATE INDEX idx_projekte_titel ON projekte(titel);
CREATE INDEX idx_projekte_status ON projekte(status);
CREATE INDEX idx_projekte_kunde ON projekte(kunde);
CREATE INDEX idx_projekte_budget ON projekte(budget);
CREATE INDEX idx_projekte_startdatum ON projekte(startdatum);
```

### Index justification

| Field | sortable | filterable | relation | Indexed |
|-------|----------|------------|----------|---------|
| titel | yes | — | — | yes |
| status | yes | yes | — | yes (one index) |
| kunde | — | — | yes | yes |
| budget | yes | — | — | yes |
| startdatum | yes | — | — | yes |
| notizen | — | — | — | no |
