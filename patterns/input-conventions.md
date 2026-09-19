# Input Conventions

Rules for form inputs: visual masks, smart parsing, and placeholder policy.

## Input Height

All interactive form elements must match the shadcn/ui default height of `h-9` (36px). This includes custom components like masked time/date inputs.

| Component | Height | Class |
|-----------|--------|-------|
| Input | 36px | `h-9` (shadcn default) |
| Select trigger | 36px | `h-9` (shadcn default) |
| Custom masked input (time, date) | 36px | `h-9` on outer container |
| Button (sm) | 36px | `h-9` (shadcn default) |

Custom input containers that wrap multiple segments (e.g. `[HH]:[MM]`) must set `h-9` on the outer `div`, not on the inner segment inputs. Inner inputs should be borderless and fill the container height.

**Anti-pattern:** `h-10`, `h-12` on input containers — causes baseline misalignment in grid rows.

## Global: Hide Number Spinners

Browsers (especially Safari) add increment/decrement spinners to `<input type="number">`. Remove them globally in `styles.css` / `globals.css`:

```css
/* Remove number input spinners globally */
input[type="number"]::-webkit-inner-spin-button,
input[type="number"]::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
input[type="number"] {
  -moz-appearance: textfield;
}
```

Alternatively, use `<input type="text" inputMode="numeric">` for number fields — this shows the numeric keyboard on mobile without triggering browser-native number behavior (spinners, scroll-to-change). This is the preferred approach for currency and formatted number fields (see European Number Input below).

## Placeholder Policy

**No text placeholders in text inputs.** They look like real values and are redundant when a label is present. "Vorname" as label does not need "Max" as placeholder.

| Widget | Placeholder | Reason |
|--------|------------|--------|
| Input (text, number) | None | Label is sufficient |
| Textarea | None | Label is sufficient |
| Select | "Bitte waehlen..." | State hint, not example data |
| Combobox/Relation | "Suchen..." | Indicates search behavior |
| Date | None | Select-all on focus; smart parsing on blur |
| Time | Visual mask `[  :  ]` | Shows expected structure |
| Toggle/Switch | None | Visual state is self-evident |

## Field Sizing

Fields must never be wider than the maximum possible input. Oversized fields waste space and obscure what input is expected.

| Type | Max Content | Width |
|------|-------------|-------|
| year | `YYYY` (4 chars) | `6ch` + padding |
| time | `HH:MM` (5 chars) | `5ch` + padding |
| duration | `MMM:SS` (6 chars) | `6ch` + padding (`px-2`) |
| date | `DD.MM.YY` (8 chars) | `9ch` + padding |
| datetime | `DD.MM.YY HH:MM` (14 chars) | `14ch` + padding |
| number | varies | set `max-width` based on expected range |
| number (currency) | `999.999,99 €` (~13 chars) | `13ch` + padding |
| enum (select) | longest option | auto (fits content) |
| text | free text | `100%` (full width, default) |
| textarea | free text | `100%` (full width) |

Use `ch` units for structured fields — `1ch` equals the width of one character in the current font. Add ~`1.5rem` padding (left + right) for comfortable spacing.

**Important:** These `max-width` constraints apply EVEN inside 2-column grids. A year field in a `grid-cols-2` cell must still be `max-w-[calc(6ch+1.5rem)]`, not stretch to 50% of the form.

```css
.input-year     { max-width: calc(6ch + 1.5rem); }
.input-time     { max-width: calc(5ch + 1.5rem); }
.input-duration { max-width: calc(6ch + 1rem); }
.input-date     { max-width: calc(9ch + 2rem); }
.input-datetime { max-width: calc(14ch + 2rem); }
.input-currency { max-width: calc(13ch + 1.5rem); }
```

### Year Fields

Year inputs (Jahrgang, Produktionsjahr, etc.) use `type="text" inputMode="numeric"` — NEVER `type="number"` (shows spinners, allows scroll-to-change, no max-width hint). Constrain to 4 digits with `maxLength` or onChange filter.

```tsx
<Input
  type="text"
  inputMode="numeric"
  className="max-w-[calc(6ch+1.5rem)] tabular-nums"
  {...field}
  onChange={(e) => {
    const v = e.target.value.replace(/\D/g, '').slice(0, 4)
    field.onChange(v ? Number(v) : '')
  }}
/>
```

## Monospace for Structured Data

Structured inputs (time, date, currency, numbers) use a monospace font. This ensures:
- Digits align vertically in lists and forms
- Separators (`:`, `.`, `,`) stay at fixed positions
- The mask structure is visually stable during typing

```css
.input-structured {
  font-family: var(--font-mono, ui-monospace, 'SF Mono', 'Cascadia Code', monospace);
  font-variant-numeric: tabular-nums;
}
```

Apply `.input-structured` to: `time`, `date`, `datetime`, `number`, `currency` fields.

Do NOT apply to: `text`, `textarea`, `enum` (select), `relation` (combobox), `boolean` (toggle). These use the regular body font (`--font-body`).

The design system token `--font-mono` should be defined per project. If not set, the fallback chain provides sensible defaults.

## Visual Masks

Time inputs use a visual mask pattern with visible separators. Date inputs do NOT use a mask — they use a plain `<Input>` with smart parsing on blur (see "Date Input" section below).

### Time Mask

Display: segments with colon as fixed separator.

```
[  :  ]
 HH  MM
```

Same approach — colon always visible.

## Masked Input Behavior (Time only)

Time inputs use a segment-based approach: each part (HH, MM) is a separate logical segment. This solves the backspace/delete problem. Date inputs do NOT use segments — see "Date Input" below.

### Recommended Implementation: Segment-Based Input

Use multiple small inputs (one per segment) with fixed separators rendered between them. This gives native cursor and selection behavior per segment.

```
Time:     [ HH ] : [ MM ]
```

Each segment is its own `<input>`:
- `type="text"`, `inputMode="numeric"` (shows numeric keyboard on mobile)
- `maxLength` per segment (2 for HH/MM)
- Monospace font, right-aligned text within segment

### Keyboard Behavior

| Key | Behavior |
|-----|----------|
| Digit | Enters digit in current segment. When segment is full (maxLength reached), auto-advance to next segment. |
| Backspace | Deletes last digit in current segment. If segment is empty, move focus to previous segment. |
| Delete | Clears current segment. |
| Tab / Arrow Right | Move to next segment. |
| Shift+Tab / Arrow Left | Move to previous segment. |
| `:` (separator key) | Move to next segment (acts like Tab). Allows natural typing of `14:30`. |

### Auto-Advance Rules

- When a segment reaches `maxLength`, focus moves to the next segment automatically.
- Exception: If the first digit makes only one valid completion possible, do NOT auto-advance yet. E.g. in hours: typing `2` could be `20`, `21`, `22`, `23` — wait for second digit. But typing `3` can only be `03` — zero-pad and auto-advance.

### Auto-Pad on Blur (Left-Pad)

When a segment loses focus with a single digit, **left-pad with zero** (prepend):
- Hours: `9` → `09`
- Minutes: `5` → `05`

Do NOT right-pad (e.g. `3` → `30`), even though `10:30` is more common than `10:03` as a work time. Right-padding has no established convention and breaks user expectations in non-work contexts.

To enter `10:30`, the user types 4 digits: `1`, `0` (auto-advance to MM), `3`, `0`. The select-all-on-focus behavior (see below) ensures this is fast — no manual deletion needed.

### Select-All on Focus

When a segment receives focus, select all content in that segment. This way typing immediately replaces the old value — no need to manually delete first.

### Example Component Structure

```tsx
function MaskedTimeInput({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  const [hours, setHours] = useState(value?.slice(0, 2) ?? '')
  const [minutes, setMinutes] = useState(value?.slice(3, 5) ?? '')
  const minutesRef = useRef<HTMLInputElement>(null)
  const hoursRef = useRef<HTMLInputElement>(null)

  const handleHoursChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const v = e.target.value.replace(/\D/g, '').slice(0, 2)
    setHours(v)
    if (v.length === 2) minutesRef.current?.focus()
  }

  const handleHoursKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === ':') {
      e.preventDefault()
      minutesRef.current?.focus()
    }
  }

  const handleMinutesKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Backspace' && minutes === '') {
      e.preventDefault()
      hoursRef.current?.focus()
    }
  }

  const handleBlur = () => {
    const h = hours.padStart(hours.length > 0 ? 2 : 0, '0')
    const m = minutes.padStart(minutes.length > 0 ? 2 : 0, '0')
    setHours(h)
    setMinutes(m)
    if (h && m) onChange(`${h}:${m}`)
  }

  return (
    <div className="inline-flex items-center gap-0.5 input-structured">
      <input
        ref={hoursRef}
        value={hours}
        onChange={handleHoursChange}
        onKeyDown={handleHoursKeyDown}
        onBlur={handleBlur}
        onFocus={e => e.target.select()}
        inputMode="numeric"
        maxLength={2}
        className="w-[2ch] text-center bg-transparent outline-none"
      />
      <span className="text-[var(--color-muted-foreground)]">:</span>
      <input
        ref={minutesRef}
        value={minutes}
        onChange={e => setMinutes(e.target.value.replace(/\D/g, '').slice(0, 2))}
        onKeyDown={handleMinutesKeyDown}
        onBlur={handleBlur}
        onFocus={e => e.target.select()}
        inputMode="numeric"
        maxLength={2}
        className="w-[2ch] text-center bg-transparent outline-none"
      />
    </div>
  )
}
```

The outer container gets the border, sizing (`max-width: calc(5ch + 1.5rem)`), and `.input-structured` class. The inner inputs are borderless and transparent.

## Date Input

Date inputs use a single `<Input type="text">` with smart parsing on blur — NOT the segment-based approach used for time inputs. This is simpler, less code, and works well because date separators (dots) are part of the typed string.

### Key behaviors

- **Select-all on focus** — typing immediately replaces the old value
- **Smart parsing on blur** — flexible input normalized to `DD.MM.YY`
- **Invalid input reverts** — if parsing fails, revert to the last valid value (no inline error needed)
- **No placeholder, no mask** — the label is sufficient context
- **`maxLength={8}`** — limits input to `DD.MM.YY` length

### Two-Layer Architecture

**`SmartDateInput`** — for string-valued form fields. Accepts and emits `DD.MM.YY` strings.

```tsx
function SmartDateInput({ value = '', onChange, size = 'default', className, disabled }: SmartDateInputProps) {
  const [displayValue, setDisplayValue] = useState(value)

  useEffect(() => { setDisplayValue(value) }, [value])

  return (
    <Input
      type="text"
      className={cn(size === 'table' ? 'h-8' : 'h-10', 'font-mono tabular-nums', className)}
      style={{ maxWidth: 'calc(9ch + 2rem)' }}
      value={displayValue}
      onChange={(e) => setDisplayValue(e.target.value)}
      onFocus={(e) => { const el = e.target; setTimeout(() => el.select(), 0) }}
      onBlur={() => {
        const parsed = smartParseDateYY(displayValue)
        if (parsed) {
          setDisplayValue(parsed)
          onChange?.(parsed)
        } else if (displayValue === '') {
          onChange?.('')
        } else {
          setDisplayValue(value) // revert invalid input
        }
      }}
      disabled={disabled}
      maxLength={8}
    />
  )
}
```

**`DateInput`** — wrapper for `Date`-valued form fields. Converts between `Date` objects and `DD.MM.YY` strings.

```tsx
function DateInput({ value, onChange, disabled }: { value: Date | undefined; onChange: (date: Date | undefined) => void; disabled?: boolean }) {
  return (
    <SmartDateInput
      value={dateToYY(value)}
      onChange={(formatted) => {
        if (formatted === '') {
          onChange(undefined)
        } else {
          const iso = smartParseDateISO(formatted)
          if (iso) onChange(new Date(iso + 'T00:00:00'))
        }
      }}
      disabled={disabled}
    />
  )
}
```

### Usage with react-hook-form

For ISO string fields (`z.string()`):
```tsx
<SmartDateInput
  value={field.value ? formatDateYY(field.value) : ''}
  onChange={(formatted) => field.onChange(smartParseDateISO(formatted))}
/>
```

For Date fields (`z.date()`):
```tsx
<DateInput value={field.value} onChange={(date) => field.onChange(date)} />
```

## Smart Parsing

Inputs accept abbreviated values and auto-complete them. Parsing happens on blur (when the user leaves the field).

### Time Parsing Rules

| Input | Result | Rule |
|-------|--------|------|
| `9` | `09:00` | Single digit → zero-pad, assume :00 |
| `14` | `14:00` | Two digits ≤23 → assume :00 |
| `815` | `08:15` | Three digits → first digit is hour, rest is minutes |
| `1430` | `14:30` | Four digits → HHMM |
| `9:5` | `09:05` | Colon present → split, zero-pad both |
| `14:3` | `14:03` | Colon present → split, zero-pad both |
| `25` | invalid | Hours >23 → validation error |
| `1465` | invalid | Minutes >59 → validation error |

```typescript
function parseTimeInput(raw: string): string | null {
  const cleaned = raw.trim().replace(/[^0-9:]/g, '')
  
  if (cleaned.includes(':')) {
    const [h, m] = cleaned.split(':')
    const hours = parseInt(h, 10)
    const minutes = parseInt(m || '0', 10)
    if (hours > 23 || minutes > 59) return null
    return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}`
  }

  const digits = cleaned.replace(/\D/g, '')
  if (digits.length === 0) return null

  let hours: number
  let minutes: number

  switch (digits.length) {
    case 1:                              // "9" → 09:00
      hours = parseInt(digits, 10)
      minutes = 0
      break
    case 2:                              // "14" → 14:00
      hours = parseInt(digits, 10)
      minutes = 0
      break
    case 3:                              // "815" → 08:15
      hours = parseInt(digits[0], 10)
      minutes = parseInt(digits.slice(1), 10)
      break
    case 4:                              // "1430" → 14:30
      hours = parseInt(digits.slice(0, 2), 10)
      minutes = parseInt(digits.slice(2), 10)
      break
    default:
      return null
  }

  if (hours > 23 || minutes > 59) return null
  return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}`
}
```

### Date Parsing Rules

Format: DD.MM.YY (European, 2-digit year). Current year is used as default when year is omitted. Output is always `DD.MM.YY` (zero-padded, 2-digit year).

| Input | Result (assuming 2026) | Rule |
|-------|----------------------|------|
| `1.2` | `01.02.26` | Day.Month → current year |
| `1.2.` | `01.02.26` | Day.Month. → current year |
| `1.2.25` | `01.02.25` | Two-digit year → 20XX |
| `1.2.2025` | `01.02.25` | Full year → truncated to YY |
| `01.02.26` | `01.02.26` | Already formatted |
| `31.2.` | invalid | Day out of range for month → revert to previous value |

```typescript
function smartParseDateYY(raw: string): string | null {
  const parts = raw.trim().split('.').filter(p => p !== '')
  if (parts.length < 2) return null

  const d = parseInt(parts[0], 10)
  const m = parseInt(parts[1], 10)

  let y: number
  if (!parts[2] || parts[2] === '') {
    y = new Date().getFullYear()               // current year
  } else if (parts[2].length <= 2) {
    y = 2000 + parseInt(parts[2], 10)           // "25" → 2025
  } else {
    y = parseInt(parts[2], 10)                  // "2025" → 2025
  }

  // Validate
  const date = new Date(y, m - 1, d)
  if (date.getFullYear() !== y || date.getMonth() !== m - 1 || date.getDate() !== d) {
    return null  // invalid date (e.g. 31.02)
  }

  return `${String(d).padStart(2, '0')}.${String(m).padStart(2, '0')}.${String(y).slice(-2)}`
}
```

A second helper converts to ISO for DB storage:

```typescript
function smartParseDateISO(raw: string): string | null {
  // Same parsing logic as smartParseDateYY, but returns "YYYY-MM-DD"
  // ...
  return `${String(y).padStart(4, '0')}-${String(m).padStart(2, '0')}-${String(d).padStart(2, '0')}`
}
```

### Number Parsing

European format: comma as decimal, dot as thousands. Parsing accepts flexible input.

| Input | Result | Rule |
|-------|--------|------|
| `1000` | `1.000` | Auto-format thousands |
| `1.000` | `1.000` | Already formatted |
| `1000,5` | `1.000,5` | Comma = decimal |
| `1,5` | `1,5` | Small decimal number |

Formatting happens on blur. During typing, the raw input is shown.

## Duration Input (MM:SS)

For fields representing a duration (e.g. film runtime), use a segment-based `DurationInput` component. Similar to `MaskedTimeInput` (HH:MM) but stores `MM:SS` — minutes can exceed 59 (e.g. `120:00` for a 2-hour film).

### Key differences from Time (HH:MM)

| | Time (HH:MM) | Duration (MM:SS) |
|---|---|---|
| Minutes segment | max 2 digits, 0-23 range | up to 3 digits, no upper limit |
| Seconds segment | max 2 digits, 0-59 range | max 2 digits, 0-59 range |
| Stored format | `"HH:MM"` | `"MM:SS"` or `"MMM:SS"` |
| DB type | `time` | `text` (flexible length) |
| Use case | Clock times (Drehbeginn) | Durations (Laufzeit) |

### Smart Parsing Rules

| Input | Result | Rule |
|-------|--------|------|
| `5` | `05:00` | Single digit → zero-pad, assume :00 |
| `12` | `12:00` | Two digits → assume :00 |
| `130` | `01:30` | Three digits → first digit is minutes, rest is seconds |
| `1230` | `12:30` | Four digits → MM:SS |
| `5:3` | `05:03` | Colon present → split, zero-pad both |
| `12:30` | `12:30` | Already formatted |
| `120:00` | `120:00` | Long duration (>99 min) is valid |
| `12:65` | invalid | Seconds >59 → validation error |

### Component Structure

```tsx
<DurationInput value={field.value} onChange={field.onChange} />
```

The component renders as two segment inputs with `:` separator, sized at `max-width: calc(7ch + 1.5rem)`. Minutes segment allows up to 3 digits. Supports paste (smart-parsed), Tab/arrow between segments, select-all-on-focus.

### Extending to HH:MM:SS

When longer durations are needed, extend the component with a third segment:
```
[ HH ] : [ MM ] : [ SS ]
```
Same principles apply — each segment is its own `<input>`, auto-advance on `maxLength`, left-pad on blur.

## Schema Types for Time and Duration

Types to add to schema-format.md:

| Schema Type | DB Type | UI Widget | Zod |
|-------------|---------|-----------|-----|
| `date` | `date` | SmartDateInput (single text input, smart parsing on blur) | `z.string().regex(/^\d{2}\.\d{2}\.\d{2}$/)` |
| `time` | `time` | Masked time input (segment-based) with smart parsing | `z.string().regex(/^\d{2}:\d{2}$/)` |
| `duration` | `text` | DurationInput (MM:SS segment input) | `z.string().regex(/^\d{2,3}:\d{2}$/)` |
| `datetime` | `timestamptz` | SmartDateInput + Time mask side by side | `z.string().datetime()` |

### Time field example in YAML:
```yaml
- name: drehbeginn
  type: time
  label: Drehbeginn
  required: true
  list: true
  sortable: true
```

### Datetime field example in YAML:
```yaml
- name: deadline
  type: datetime
  label: Deadline
  required: true
  list: true
  sortable: true
```

## Integration with Form Pattern

All smart parsing runs on blur. During active typing, show raw input. On blur:
1. Run parser (parseTimeInput / parseDateInput)
2. If valid → replace raw input with formatted value, update form state
3. If invalid → show inline validation error, keep raw input so user can fix it

---

## See also

- `patterns/formatting.md` — output-side formatting (display values)
- `patterns/form.md` — form widgets that consume these input conventions
- `schema-format.md` — field type definitions (time, datetime)
