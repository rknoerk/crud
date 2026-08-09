# Input Conventions

Rules for form inputs: visual masks, smart parsing, and placeholder policy.

## Placeholder Policy

**No text placeholders in text inputs.** They look like real values and are redundant when a label is present. "Vorname" as label does not need "Max" as placeholder.

| Widget | Placeholder | Reason |
|--------|------------|--------|
| Input (text, number) | None | Label is sufficient |
| Textarea | None | Label is sufficient |
| Select | "Bitte waehlen..." | State hint, not example data |
| Combobox/Relation | "Suchen..." | Indicates search behavior |
| Date | Visual mask `[  .  .  ]` | Shows expected structure |
| Time | Visual mask `[  :  ]` | Shows expected structure |
| Toggle/Switch | None | Visual state is self-evident |

## Field Sizing

Fields must never be wider than the maximum possible input. Oversized fields waste space and obscure what input is expected.

| Type | Max Content | Width |
|------|-------------|-------|
| time | `HH:MM` (5 chars) | `5ch` + padding |
| date | `DD.MM.YYYY` (10 chars) | `10ch` + padding |
| datetime | `DD.MM.YYYY HH:MM` (16 chars) | `16ch` + padding |
| number | varies | set `max-width` based on expected range |
| number (currency) | `999.999,99 €` (~13 chars) | `13ch` + padding |
| enum (select) | longest option | auto (fits content) |
| text | free text | `100%` (full width, default) |
| textarea | free text | `100%` (full width) |

Use `ch` units for structured fields — `1ch` equals the width of one character in the current font. Add ~`1.5rem` padding (left + right) for comfortable spacing.

```css
.input-time     { max-width: calc(5ch + 1.5rem); }
.input-date     { max-width: calc(10ch + 1.5rem); }
.input-datetime { max-width: calc(16ch + 1.5rem); }
.input-currency { max-width: calc(13ch + 1.5rem); }
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

Structured inputs (date, time) use visual mask patterns instead of text placeholders. The mask shows the expected structure with visible separators rendered as part of the input UI, not as placeholder text.

### Date Mask

Display: segments with dots as fixed separators, rendered visually (not as placeholder text).

```
[  .  .    ]
 DD  MM  YYYY
```

Implementation approach: Either a masked input library or three adjacent input segments with dot separators rendered between them. The dots are always visible, not dependent on focus state.

### Time Mask

Display: segments with colon as fixed separator.

```
[  :  ]
 HH  MM
```

Same approach — colon always visible.

## Masked Input Behavior

Masked inputs (date, time) use a segment-based approach: each part (HH, MM, DD, MM, YYYY) is a separate logical segment. This solves the backspace/delete problem.

### Recommended Implementation: Segment-Based Input

Instead of a single `<input>` with a mask library, use multiple small inputs (one per segment) with fixed separators rendered between them. This gives native cursor and selection behavior per segment.

```
Time:     [ HH ] : [ MM ]
Date:     [ DD ] . [ MM ] . [ YYYY ]
Datetime: [ DD ] . [ MM ] . [ YYYY ]  [ HH ] : [ MM ]
```

Each segment is its own `<input>`:
- `type="text"`, `inputMode="numeric"` (shows numeric keyboard on mobile)
- `maxLength` per segment (2 for HH/MM/DD/MM, 4 for YYYY)
- Monospace font, right-aligned text within segment

### Keyboard Behavior

| Key | Behavior |
|-----|----------|
| Digit | Enters digit in current segment. When segment is full (maxLength reached), auto-advance to next segment. |
| Backspace | Deletes last digit in current segment. If segment is empty, move focus to previous segment. |
| Delete | Clears current segment. |
| Tab / Arrow Right | Move to next segment. |
| Shift+Tab / Arrow Left | Move to previous segment. |
| `:` or `.` (separator key) | Move to next segment (acts like Tab). Allows natural typing of `14:30` or `1.2.26`. |

### Auto-Advance Rules

- When a segment reaches `maxLength`, focus moves to the next segment automatically.
- Exception: If the first digit makes only one valid completion possible, do NOT auto-advance yet. E.g. in hours: typing `2` could be `20`, `21`, `22`, `23` — wait for second digit. But typing `3` can only be `03` — zero-pad and auto-advance.

### Auto-Pad on Blur

When a segment loses focus with a single digit:
- Hours: `9` → `09`
- Minutes: `5` → `05`
- Day: `1` → `01`
- Month: `2` → `02`

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

Format: DD.MM.YYYY (European). Current year is used as default when year is omitted.

| Input | Result (assuming 2026) | Rule |
|-------|----------------------|------|
| `1.2.` | `01.02.2026` | Day.Month. → current year |
| `1.2` | `01.02.2026` | Day.Month without trailing dot → current year |
| `1.2.25` | `01.02.2025` | Two-digit year → 20XX |
| `1.2.2025` | `01.02.2025` | Full date |
| `01.02.2025` | `01.02.2025` | Already formatted |
| `31.2.` | invalid | Day out of range for month |

```typescript
function parseDateInput(raw: string): string | null {
  const cleaned = raw.trim()
  const parts = cleaned.split('.').filter(p => p !== '')

  if (parts.length < 2) return null

  const day = parseInt(parts[0], 10)
  const month = parseInt(parts[1], 10)

  let year: number
  if (parts.length === 2 || parts[2] === '') {
    year = new Date().getFullYear()               // current year
  } else if (parts[2].length <= 2) {
    year = 2000 + parseInt(parts[2], 10)           // "25" → 2025
  } else {
    year = parseInt(parts[2], 10)                  // "2025" → 2025
  }

  // Validate
  const date = new Date(year, month - 1, day)
  if (date.getFullYear() !== year || date.getMonth() !== month - 1 || date.getDate() !== day) {
    return null  // invalid date (e.g. 31.02)
  }

  return `${String(day).padStart(2, '0')}.${String(month).padStart(2, '0')}.${year}`
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

## Schema Types for Time

Two new types to add to schema-format.md:

| Schema Type | DB Type | UI Widget | Zod |
|-------------|---------|-----------|-----|
| `time` | `time` | Masked time input with smart parsing | `z.string().regex(/^\d{2}:\d{2}$/)` |
| `datetime` | `timestamptz` | Date mask + Time mask side by side | `z.string().datetime()` |

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
