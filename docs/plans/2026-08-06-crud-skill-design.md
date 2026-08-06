# CRUD Skill — Design Document

> Hybrid aus Pattern-Rezeptbuch und Schema-getriebenem Workflow, der eine YAML-Entity-Definition als Input nimmt und daraus Supabase-Migrationen, TypeScript-Typen, Routes und UI-Komponenten generiert.

## V1 Scope

- Schema-Format (YAML mit DB + UI-Hints)
- Patterns (Navigation, Listen, Detail, Edit, Loeschen)
- Workflow (Schema -> Migration -> Routes -> Komponenten)
- Design-System-Struktur (Token-Konventionen, Komponentenkatalog-Format)

## Tech Stack

- React + TypeScript + Vite
- Supabase (PostgreSQL, Auth)
- TanStack React Query v5
- React Hook Form + Zod
- Tailwind CSS + shadcn/ui
- Lovable (Deployment)
- Router: noch offen (TanStack Start vs. React Router)

---

## 1. Schema-Format

Jede Entity wird in einer YAML-Datei beschrieben:

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

### Prinzipien

- `type` deckt DB-Typ und UI-Widget ab:
  - `text` -> `varchar` + Input
  - `textarea` -> `text` + Textarea
  - `enum` -> Postgres-Enum + Select
  - `number` -> `numeric` + Number-Input
  - `date` -> `date` + Datepicker
  - `relation` -> `uuid REFERENCES` + Combobox mit Suche
  - `boolean` -> `boolean` + Toggle/Switch
- UI-Hints (`list`, `sortable`, `filterable`, `searchable`) steuern Listen-Verhalten
- `format` fuer Darstellung (`currency`, `date`, `percent`)
- `id`, `created_at`, `updated_at` werden automatisch generiert, nicht im Schema

---

## 2. Navigation & Routing

### Routen-Struktur pro Entity

```
/entity                    -> Liste
/entity/:id                -> Detail (Leseansicht)
/entity/:id/edit           -> Formular (Bearbeiten)
/entity/:id/sub-entity     -> Sub-Liste (hierarchisch)
/entity/:id/sub-entity/:id -> Sub-Detail
```

### Regeln

- **URL-Hierarchie = Navigation-Stack.** Breadcrumbs werden aus URL-Segmenten generiert. Jedes Segment hat ein Label (aus Schema: `label.plural` fuer Listen, Titel-Feld fuer Detail).
- **Filter & Sort in Search-Params:** `/projekte?status=aktiv&sort=-startdatum`
- **Querverweise** nutzen `navigate(url, { state: { from: currentPath } })`. Zurueck: State vorhanden -> dorthin, sonst -> kanonische Eltern-Route.
- **Responsive Master-Detail:**
  - Desktop: Liste + Detail nebeneinander (Nested Route/Outlet), Liste bleibt gemounted
  - Mobile: Vollbild-Wechsel, Scroll-Position in `sessionStorage` gespeichert

### Formular-Navigation

- "Bearbeiten" -> navigiert zu `/edit`
- "Neuer Eintrag" in der Liste -> erstellt Datensatz mit Defaults via Supabase, navigiert zu `/:id/edit`
- Speichern/Abbrechen -> navigiert zurueck zur Detail-Ansicht
- Ungespeicherte Aenderungen -> Route-Guard mit Warnung

---

## 3. Listen-Pattern

### Layout

```
+------------------------------------------+
| [Entity-Plural]              [+ Neu]     |  Header + Create-Button
+------------------------------------------+
| Suche...    [Filter1] [Filter2]          |  searchable/filterable Felder
+------------------------------------------+
| Spalte1 v   Spalte2   Spalte3   Spalte4  |  list + sortable Felder
| ---------------------------------------- |
| Wert        Wert      Wert      Wert     |
| Wert        Wert      Wert      Wert     |
+------------------------------------------+
```

### Verhalten

- Alle Daten initial laden. Paginierung ab Schwellwert (~200 Eintraege) via Supabase `.range()`.
- Filter, Sort und Suche als URL-Search-Params. React Query Key enthaelt diese Params -> automatisches Refetching.
- Klick auf Zeile -> navigiert zu `/:id`
- Leerer Zustand: freundliche Nachricht + Create-Button
- Loeschen aus Liste moeglich (Swipe Mobile, Context-Menu Desktop), immer mit Bestaetigungsdialog

### React Query Pattern

```typescript
const { data, isLoading } = useQuery({
  queryKey: ['projekte', { status, sort, search }],
  queryFn: () => supabase
    .from('projekte')
    .select('*')
    .match(filters)
    .order(sort.field, { ascending: sort.direction === 'asc' })
})
```

---

## 4. Detail-Ansicht

### Layout

```
+------------------------------------------+
| <- Zurueck          [Bearbeiten] [Del]   |  Breadcrumb/Back + Aktionen
+------------------------------------------+
| Titel des Eintrags                       |  Erstes text-required-Feld
|                                          |
| Feld-Label    Wert                       |
| Feld-Label    Wert (Link bei Relation)   |
| Feld-Label    Wert                       |
+------------------------------------------+
| Sub-Entity (Anzahl)            [+ Neu]   |  Sub-Listen (1:n Relations)
| ---------------------------------------- |
| Eintrag 1                                |
| Eintrag 2                                |
+------------------------------------------+
```

### Regeln

- Felder in Schema-Reihenfolge
- Relations als klickbare Links (Querverweis mit State)
- `format` bestimmt Darstellung (`currency` -> `50.000 EUR`, `date` -> `15.03.2026`)
- Loeschen: Bestaetigungsdialog, danach zurueck zur Liste
- Sub-Listen aus 1:n-Relations abgeleitet, mit Link zur vollen Sub-Liste

---

## 5. Formular (Edit)

### Widget-Mapping

| Schema-Type | Widget | DB-Typ |
|-------------|--------|--------|
| `text` | Input | `varchar` |
| `textarea` | Textarea | `text` |
| `enum` | Select | Postgres-Enum |
| `number` | Number-Input | `numeric` |
| `date` | Datepicker | `date` |
| `relation` | Combobox mit Suche | `uuid REFERENCES` |
| `boolean` | Toggle/Switch | `boolean` |

### Regeln

- Felder in Schema-Reihenfolge
- `required: true` -> Pflichtfeld-Markierung (*), Zod-Validierung
- Validierungsfehler inline unter dem Feld (React Hook Form + Zod)
- Speichern: Toast bei Erfolg/Fehler, navigiert zurueck zu Detail
- Abbrechen: Route-Guard warnt bei ungespeicherten Aenderungen
- Europaeische Formate: Datum `DD.MM.YYYY`, Zahlen mit Punkt als Tausender, Komma als Dezimal

### Validierung

- Client-side: Zod-Schema wird aus YAML generiert, sofortiges Feedback
- DB-Constraints: CHECK, NOT NULL als Sicherheitsnetz
- Kein separater Server-Layer noetig

### Error Handling

- Validierungsfehler: inline am Feld
- Speichern/Loeschen Erfolg: Toast
- Server-Fehler: Toast mit Fehlermeldung

---

## 6. Design-System-Struktur

### Token-Hierarchie

```
tokens/
  colors.css      -> --color-primary, --color-surface, --color-border, ...
  typography.css   -> --font-body, --font-heading, --text-sm/md/lg/xl
  spacing.css      -> --space-1 bis --space-12
  radius.css       -> --radius-sm/md/lg
  shadows.css      -> --shadow-sm/md/lg
```

Pro Projekt wird ein Theme definiert das diese Variablen belegt. Der Skill liefert nur die Variablen-Namen und deren Verwendung.

### Komponenten-Katalog (shadcn/ui-basiert)

| Pattern | Komponenten |
|---------|-------------|
| Liste | DataTable, SearchInput, FilterSelect, Pagination |
| Detail | FieldDisplay, RelationLink, SubList |
| Formular | FormField, FormSelect, FormCombobox, FormDatepicker, FormToggle |
| Layout | AppShell, MasterDetail, PageHeader, Breadcrumbs |
| Feedback | Toast (Sonner), ConfirmDialog, EmptyState |
| Navigation | Sidebar, MobileNav |

### Regeln

- Alle Komponenten nutzen ausschliesslich CSS-Variablen, keine hardcoded Farben/Groessen
- shadcn/ui als Basis, erweitert wo noetig
- Jede Komponente dokumentiert welche Tokens sie nutzt

---

## 7. Workflow

Wenn der Skill mit einem Schema aufgerufen wird:

### Schritt 1: Schema lesen & validieren
- YAML parsen, Pflichtfelder pruefen, Relations aufloesen
- Fehler frueh melden ("Entity 'kontakte' in Relation referenziert, aber kein Schema vorhanden")

### Schritt 2: Supabase-Migration generieren
- `CREATE TABLE` mit korrekten Typen
- `id uuid DEFAULT gen_random_uuid()`, `created_at`, `updated_at` automatisch
- Indizes auf `sortable` und `filterable` Felder

### Schritt 3: TypeScript-Typen generieren
- Zod-Schema aus YAML ableiten
- TypeScript-Interface aus Zod inferieren
- Ein File pro Entity: `schemas/entity.ts`

### Schritt 4: Routes anlegen
- Liste, Detail, Edit pro Entity
- Sub-Routen fuer hierarchische Relations
- Responsive MasterDetail-Layout einbinden

### Schritt 5: Komponenten generieren
- `EntityList` — DataTable mit Filtern/Sort aus Schema
- `EntityDetail` — Feld-Anzeige + Sub-Listen
- `EntityForm` — React Hook Form + Zod, Widgets aus `type`
- React Query Hooks: `useEntities`, `useEntity`, `useCreateEntity`, `useUpdateEntity`, `useDeleteEntity`

### Schritt 6: Qualitaets-Check
- Alle Routen erreichbar?
- Zod-Schema und DB-Schema konsistent?
- Design-System-Tokens referenziert, keine hardcoded Werte?
- Leere Zustaende und Fehlerfaelle abgedeckt?

**Zwischen jedem Schritt:** Ergebnis dem User zeigen, Feedback einholen, dann weiter.

---

## 8. Offene Punkte

- **Auth & RLS** — muss nochmal analysiert werden, was bisher problematisch war
- **State Management** — Entscheidung anhand konkreter Faelle, nicht abstrakt
- **Router** — TanStack Start vs. React Router, noch keine Festlegung

## V2 Roadmap

- Spalten ein-/ausblenden mit User-Praeferenz (localStorage)
- Multi-Entity-Relationen (n:m) in der UI
- Dashboard/Uebersichtsseiten
- Export (PDF, Excel)
- Bulk-Aktionen in Listen
- Audit-Log
