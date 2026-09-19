# Navigation Layout

App-wide navigation structure. Sidebar and bottom tabs share the same navigation model — items are defined once, rendered differently per breakpoint.

## Responsive Breakpoints

| Breakpoint | Pattern | Details |
|---|---|---|
| Desktop (>=1024px) | Sidebar | Persistent, expanded with icons + labels. Collapsible sections for grouping. |
| Tablet (768-1023px) | Navigation Rail | Narrow vertical bar, icons only. Tooltip on hover for labels. |
| Mobile (<768px) | Bottom Tab Bar | 4-5 items max. Icons + short labels below. |

## Navigation Model

Define navigation items once, render per breakpoint:

```typescript
interface NavItem {
  id: string
  label: string           // German UI label
  icon: LucideIcon
  path: string            // route path, e.g. '/projekte'
  primary: boolean        // true = visible in bottom tabs, false = only in sidebar/"Mehr"
}

// Example for a film production app:
const navItems: NavItem[] = [
  { id: 'projekte', label: 'Projekte', icon: Film, path: '/projekte', primary: true },
  { id: 'kontakte', label: 'Kontakte', icon: Users, path: '/kontakte', primary: true },
  { id: 'termine', label: 'Termine', icon: Calendar, path: '/termine', primary: true },
  { id: 'dokumente', label: 'Dokumente', icon: FileText, path: '/dokumente', primary: true },
  { id: 'einstellungen', label: 'Einstellungen', icon: Settings, path: '/einstellungen', primary: false },
  { id: 'hilfe', label: 'Hilfe', icon: HelpCircle, path: '/hilfe', primary: false },
]
```

## Bottom Tab Bar (Mobile)

Rules:
- Max 5 tabs. If more than 5 primary items, last tab becomes "Mehr" (MoreHorizontal icon) opening a list of remaining items.
- Each tab: icon (24px) + label below (text-xs)
- Active tab: highlighted with `--color-primary`
- Min tap target: 44x44px
- Fixed at bottom, above safe area (iOS notch)
- Current route determines active tab

```tsx
function BottomTabBar({ items }: { items: NavItem[] }) {
  const location = useLocation()
  const primaryItems = items.filter(i => i.primary).slice(0, 4)
  const secondaryItems = items.filter(i => !i.primary || items.filter(i => i.primary).indexOf(i) >= 4)
  const showMore = secondaryItems.length > 0

  return (
    <nav className="fixed bottom-0 inset-x-0 z-50 border-t bg-[var(--color-background)] pb-[env(safe-area-inset-bottom)] md:hidden">
      <div className="flex items-center justify-around h-16">
        {primaryItems.map(item => (
          <TabItem
            key={item.id}
            item={item}
            active={location.pathname.startsWith(item.path)}
          />
        ))}
        {showMore && (
          <MoreTab items={secondaryItems} />
        )}
      </div>
    </nav>
  )
}

function TabItem({ item, active }: { item: NavItem; active: boolean }) {
  return (
    <Link
      to={item.path}
      className={cn(
        "flex flex-col items-center justify-center gap-1 min-w-[44px] min-h-[44px] text-xs",
        active ? "text-[var(--color-primary)]" : "text-[var(--color-muted-foreground)]"
      )}
    >
      <item.icon className="h-6 w-6" />
      <span>{item.label}</span>
    </Link>
  )
}
```

## "Mehr" Tab

When >5 sections exist, the 5th tab opens a bottom sheet or full-screen list with remaining items:

```tsx
function MoreTab({ items }: { items: NavItem[] }) {
  const [open, setOpen] = useState(false)

  return (
    <>
      <button
        onClick={() => setOpen(true)}
        className="flex flex-col items-center justify-center gap-1 min-w-[44px] min-h-[44px] text-xs text-[var(--color-muted-foreground)]"
      >
        <MoreHorizontal className="h-6 w-6" />
        <span>Mehr</span>
      </button>
      <Sheet open={open} onOpenChange={setOpen}>
        <SheetContent side="bottom">
          <SheetHeader>
            <SheetTitle>Navigation</SheetTitle>
          </SheetHeader>
          <nav className="grid gap-2 py-4">
            {items.map(item => (
              <Link
                key={item.id}
                to={item.path}
                onClick={() => setOpen(false)}
                className="flex items-center gap-3 px-4 py-3 rounded-md hover:bg-[var(--color-accent)]"
              >
                <item.icon className="h-5 w-5" />
                <span>{item.label}</span>
              </Link>
            ))}
          </nav>
        </SheetContent>
      </Sheet>
    </>
  )
}
```

## Sidebar (Desktop)

**Dark sidebar is the default.** This is the 2024-2026 standard (Linear, Notion, Vercel, Supabase). It solves contrast problems, creates natural visual hierarchy, and eliminates the need to fine-tune gray-on-gray color relationships between sidebar, page background, and cards.

Three zones: fixed header, scrollable nav, fixed footer (user/logout). The sidebar stays viewport-height while main content scrolls independently.

### Color scheme

| Element | Class | Notes |
|---|---|---|
| Sidebar background | `bg-slate-900` | Dark, high contrast with main content |
| Borders | `border-slate-800` | Subtle on dark |
| Nav text (default) | `text-slate-400` | Muted but readable |
| Nav text (hover) | `hover:bg-slate-800 hover:text-white` | Lighten on hover |
| Nav text (active) | `bg-primary font-medium text-white` | Brand/accent color fill — NOT a transparent tint |
| User/logout footer | Same as nav default | `text-slate-400 hover:bg-slate-800 hover:text-white` |
| App name | `text-white` | High contrast |

### Logo handling

If the logo PNG has a non-transparent background (common with institutional logos), wrap it in a white pill:

```tsx
<img src="/logo.png" alt="App" className="h-9 rounded p-0.5 bg-white" />
```

This makes the white background intentional rather than looking like a bug. `rounded` for soft corners, `p-0.5` for breathing room, `bg-white` for the pill.

### Code

```tsx
function Sidebar({ items, user }: { items: NavItem[]; user: { name: string } }) {
  const location = useLocation()

  return (
    <aside className="hidden lg:flex lg:flex-col lg:w-64 border-r border-slate-800 bg-slate-900 h-screen sticky top-0">
      <div className="shrink-0 flex items-center gap-2 h-14 border-b border-slate-800 px-5">
        <img src="/logo.png" alt="App" className="h-9 rounded p-0.5 bg-white" />
        <span className="text-sm font-semibold tracking-tight text-white">App Name</span>
      </div>
      <nav className="flex-1 overflow-y-auto p-3">
        {items.map(item => (
          <Link
            key={item.id}
            to={item.path}
            className={cn(
              "flex items-center gap-2 rounded-md px-3 py-1.5 text-sm",
              location.pathname.startsWith(item.path)
                ? "bg-primary font-medium text-white"
                : "text-slate-400 hover:bg-slate-800 hover:text-white"
            )}
          >
            <item.icon className="h-4 w-4" />
            <span>{item.label}</span>
          </Link>
        ))}
      </nav>
      <div className="shrink-0 border-t border-slate-800 p-3">
        <Link
          to="/profil"
          className="flex items-center gap-2 rounded-md px-3 py-1.5 text-sm text-slate-400 hover:bg-slate-800 hover:text-white"
        >
          <span className="truncate">{user.name}</span>
        </Link>
        <button
          type="button"
          onClick={handleLogout}
          className="mt-0.5 flex w-full items-center gap-2 rounded-md px-3 py-1.5 text-sm text-slate-400 hover:bg-slate-800 hover:text-white"
        >
          <LogOut className="h-4 w-4" />
          Abmelden
        </button>
      </div>
    </aside>
  )
}
```

**Layout principle:** `h-screen sticky top-0` on aside, `shrink-0` on header and footer, `flex-1 overflow-y-auto` on nav. This keeps header and user footer always visible while nav scrolls if items overflow.

### Why not gray sidebar?

Gray sidebars (`bg-secondary`, `bg-muted`) create cascading contrast problems:
- Logo contrast on gray is poor (especially institutional logos with white backgrounds)
- Active nav item tints (`bg-primary/10`) are barely visible on gray
- Page background must be carefully tuned to differentiate from sidebar AND from cards
- Results in 3+ iterations of gray-on-gray adjustments

Dark sidebar eliminates all of these. One decision, zero follow-up issues.

## Navigation Rail (Tablet)

Icons only, narrow bar, tooltip labels:

```tsx
function NavigationRail({ items }: { items: NavItem[] }) {
  const location = useLocation()

  return (
    <aside className="hidden md:flex lg:hidden flex-col items-center w-16 border-r bg-[var(--color-card)] h-screen sticky top-0 py-4 gap-2">
      {items.map(item => (
        <Tooltip key={item.id}>
          <TooltipTrigger asChild>
            <Link
              to={item.path}
              className={cn(
                "flex items-center justify-center w-10 h-10 rounded-md",
                location.pathname.startsWith(item.path)
                  ? "bg-[var(--color-accent)] text-[var(--color-accent-foreground)]"
                  : "text-[var(--color-muted-foreground)] hover:bg-[var(--color-accent)]"
              )}
            >
              <item.icon className="h-5 w-5" />
            </Link>
          </TooltipTrigger>
          <TooltipContent side="right">{item.label}</TooltipContent>
        </Tooltip>
      ))}
    </aside>
  )
}
```

## AppShell

Combines all navigation patterns into a single responsive layout:

```tsx
function AppShell({ children, navItems }: { children: React.ReactNode; navItems: NavItem[] }) {
  return (
    <div className="flex min-h-screen">
      <Sidebar items={navItems} />
      <NavigationRail items={navItems} />
      <main className="flex-1 pb-16 md:pb-0">
        {children}
      </main>
      <BottomTabBar items={navItems} />
    </div>
  )
}
```

Key: `pb-16 md:pb-0` on main gives space for bottom tabs on mobile, removed on tablet/desktop.

## Rules

- Navigation items defined once, filtered/rendered per breakpoint
- All components use CSS variables from design system tokens exclusively
- Active state uses `--color-primary` (tabs) or `--color-accent` (sidebar/rail)
- Bottom tabs: fixed position, respects safe-area-inset-bottom for iOS
- Sidebar: dark (`bg-slate-900`), sticky, full height, scrollable nav area
- Active nav item: `bg-primary text-white` (brand accent), NOT transparent tint
- Max 5 bottom tabs including "Mehr" — if project has <=5 entities, no "Mehr" needed
- Hamburger menu is NOT used as primary navigation. Only acceptable for authenticated user menu (profile, logout) if needed.
- Page background should be slightly tinted (`oklch(0.985)`) so white cards pop — NOT pure white

---

## See also

- `patterns/navigation.md` — routing, breadcrumbs, URL params, guards
- `design-system.md` — CSS variable tokens used by all navigation components
