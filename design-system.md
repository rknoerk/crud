# Design System Token Structure

Defines the token structure (CSS variable names) and component catalog used by all CRUD apps. Each project defines its own theme by setting these CSS variables.

## Token Hierarchy

### Colors

```css
--color-primary, --color-primary-foreground
--color-secondary, --color-secondary-foreground
--color-accent, --color-accent-foreground
--color-destructive, --color-destructive-foreground
--color-muted, --color-muted-foreground
--color-background, --color-foreground
--color-card, --color-card-foreground
--color-popover, --color-popover-foreground
--color-border
--color-input
--color-ring
```

### Typography

```css
--font-body, --font-heading
--text-xs, --text-sm, --text-base, --text-lg, --text-xl, --text-2xl
--leading-tight, --leading-normal, --leading-relaxed
--font-weight-normal, --font-weight-medium, --font-weight-semibold, --font-weight-bold
```

### Spacing

```css
--space-0.5, --space-1, --space-1.5, --space-2, --space-2.5, --space-3,
--space-4, --space-5, --space-6, --space-8, --space-10, --space-12, --space-16
```

Follows the Tailwind spacing scale.

### Radius

```css
--radius-sm, --radius-md, --radius-lg, --radius-full
```

### Shadows

```css
--shadow-sm, --shadow-md, --shadow-lg
```

## Component Catalog

| Pattern | Components | Key Tokens Used |
|---------|-----------|-----------------|
| Liste | DataTable, SearchInput, FilterSelect, Pagination | `--color-border`, `--color-muted`, `--text-sm` |
| Detail | FieldDisplay, RelationLink, SubList | `--color-muted-foreground`, `--text-base` |
| Formular | FormField, FormSelect, FormCombobox, FormDatepicker, FormToggle | `--color-input`, `--color-ring`, `--color-destructive` |
| Layout | AppShell, MasterDetail, PageHeader, Breadcrumbs | `--color-background`, `--color-card`, `--shadow-sm` |
| Feedback | Toast (Sonner), ConfirmDialog, EmptyState | `--color-primary`, `--color-destructive` |
| Navigation | Sidebar, MobileNav | `--color-accent`, `--color-border` |

## Rules

1. All components use CSS variables exclusively -- no hardcoded colors or sizes
2. shadcn/ui as base component library, extended where needed
3. Each component documents which tokens it consumes
4. shadcn/ui already uses CSS variables internally -- these token names align with shadcn conventions

## Creating a Project Theme

Define all variables in `app/globals.css` under `:root` (light) and `.dark` (dark mode):

```css
@layer base {
  :root {
    --color-background: /* light background */;
    --color-foreground: /* light foreground */;

    --color-primary: /* ... */;
    --color-primary-foreground: /* ... */;
    --color-secondary: /* ... */;
    --color-secondary-foreground: /* ... */;
    --color-accent: /* ... */;
    --color-accent-foreground: /* ... */;
    --color-destructive: /* ... */;
    --color-destructive-foreground: /* ... */;
    --color-muted: /* ... */;
    --color-muted-foreground: /* ... */;
    --color-card: /* ... */;
    --color-card-foreground: /* ... */;
    --color-popover: /* ... */;
    --color-popover-foreground: /* ... */;

    --color-border: /* ... */;
    --color-input: /* ... */;
    --color-ring: /* ... */;

    --font-body: /* e.g. "Inter", sans-serif */;
    --font-heading: /* e.g. "Inter", sans-serif */;

    --radius-sm: /* e.g. 0.25rem */;
    --radius-md: /* e.g. 0.375rem */;
    --radius-lg: /* e.g. 0.5rem */;
    --radius-full: /* 9999px */;

    --shadow-sm: /* ... */;
    --shadow-md: /* ... */;
    --shadow-lg: /* ... */;
  }

  .dark {
    --color-background: /* dark background */;
    --color-foreground: /* dark foreground */;
    /* ... override all color tokens for dark mode ... */
  }
}
```

Spacing and typography scale tokens use Tailwind defaults and only need overriding if the project deviates from standard Tailwind values.
