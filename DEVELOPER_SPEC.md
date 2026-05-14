# EC Line Scanner — Admin Panel Developer Spec
**Prepared for:** Peter  
**Reference mockup:** `line-scanner-admin-v2.html`  
**Last updated:** May 14, 2026

---

## Overview

The admin panel is a tablet-first UI (15" landscape, max-width 1280px) used by line supervisors to manage worker clock events, view audit history, and assign training types. It is a single-page app with screen-based navigation (no routing). All screens share a common header shell.

---

## Design Tokens

Define these as CSS custom properties on `:root`:

```css
--blue:        #2980b9;
--blue-dark:   #1f6391;
--blue-light:  #d6eaf8;
--teal:        #1abc9c;
--green-text:  #27ae60;
--green-bg:    #d5f5e3;
--red-text:    #c0392b;
--red-bg:      #fdecea;
--orange-text: #e67e22;
--orange-bg:   #fef3e2;
--gray-text:   #7f8c8d;
--gray-bg:     #ecf0f1;
--gray-border: #bdc3c7;
--text:        #2c3e50;
--text-sub:    #7f8c8d;
--surface:     #ffffff;
--page-bg:     #f4f6f8;
--header-h:    72px;
--row-min:     72px;
--touch-min:   56px;
```

---

## App Shell

- Max-width: **1280px**, centered on screen
- Each screen is a `div.screen` that is hidden by default; the active screen gets class `.active`
- Navigation sets the active screen and calls the appropriate render function
- All screens share the same fixed header structure

### Header

- Height: 72px, blue background (`--blue`)
- Left: back `←` button (44px tap target) + screen title
- Right: two-line context block, right-aligned:
  - Line 1: `"Line 5 · May 14, 2026"` — 14px, 600 weight, white
  - Line 2: `"12:30 PM · Shift 2"` — 12px, white 75% opacity
- The context block is always visible on every screen and reflects the supervisor's current active line/shift/date

---

## Screen 1 — Admin Menu

Entry point after login. Lists all available actions.

### Menu Items

| Label | Subtitle | Destination |
|---|---|---|
| Manage Worker Times (Line 5) | View and edit worker check-in and check-out times for this line | Manage Workers screen |
| Manage Worker Training Types (Line 5) | View and edit training types for workers checked in to this line | Training Types screen |
| Audit Log | View badge scans, check-in and check-out history | Audit Log screen |
| System Settings | — | Coming Soon screen |
| Check Out All Workers | Red-accented row, triggers confirmation modal | Modal |

### Row layout

Each row is a horizontal flex container:
- **Icon tile** — 40×40px square, light blue background (`--blue-light`), SVG icon centered
- **Text block** — label (17px, 700 weight) + subtitle (13px, `--text-sub`)
- **Chevron** — `›` character, right-aligned

---

## Screen 2 — Manage Workers

### Purpose

Supervisors view and edit check-in / check-out times for workers on their line. Supports browsing across shifts, lines, and dates via filters.

### Layout (top to bottom, all fixed/non-scrolling except list body)

1. Header
2. Blue alert banner: `"Edit check-in and check-out times. Use filters to view workers from other shifts or lines."`
3. Search + filter bar
4. Scrollable list body

### Search + Filter Bar

```
[ Search input (flex: 1) ]  [ chips ][ ✕ ][ Filters btn ]
```

- Search input: 48px tall, searches worker `name` and `agencyId` live on every keystroke
- Chips + Filters button are wrapped together in a right-anchored group (`margin-left: auto`)
- **Filters button**: 48px tall, blue border; turns solid blue when any filter departs from defaults
- **Active filter chips**: max 2 visible; extras collapse to `+N more` pill (tapping it opens filter modal)
- **✕ clear button**: 44×44px circle, resets all filters to defaults immediately

### Filter State

Use two objects — **applied** (`mwFilters`) and **pending** (`mwPending`). Pending state is only copied to applied when the user taps "Apply Filters". Reset restores pending to defaults without closing the modal.

**Defaults:**
```js
{
  lines: [5],        // array — which lines to show; empty = all lines
  shift: '2',        // 'all' | '1' | '2' | '3'
  date: 'today',     // 'today' | 'yesterday' | 'week' | 'month' | 'custom'
  customStart: null, // { y, m, d } object or null
  customEnd: null    // { y, m, d } object or null
}
```

### Filter Modal

Slides in from the right (420px wide, full height). Semi-transparent overlay behind it.

Sections (top to bottom):

**Line** — 6-column grid  
- "All Lines" pill (full width, selects empty array)  
- Pills for lines 1–18 (individual, multi-select)

**Shift** — 3-column grid  
- All Shifts, Shift 1, Shift 2, Shift 3 (single-select)

**Date** — vertical stacked pills (full width each)  
- Today, Yesterday, Past 7 Days, Past 30 Days  
- Custom range… (toggle; reveals inline calendar when selected)

**Custom Calendar** (shown only when Custom range is selected):
- Month/year header with prev/next nav
- 7-column day grid
- Tap once = range start, tap again = range end; third tap resets
- Visual states: `.today` (bold blue), `.selected-start` / `.selected-end` (blue fill), `.in-range` (light blue fill)
- Range label below grid shows selected dates in plain text

**Footer**: Reset (secondary) | Apply Filters (primary blue)

### Active Filter Chips

Generated from the diff between `mwFilters` and defaults:

| Filter changed | Chip label |
|---|---|
| Lines | `"Line 5"` or `"Line 5, 14"` |
| Shift | `"Shift 1"` / `"Shift 2"` / `"Shift 3"` |
| Date | `"Yesterday"` / `"Past 7 Days"` / `"Past 30 Days"` |
| Custom date | `"May 13 → May 14"` or `"May 13"` (if only start set) |

Max 2 chips visible. If 3+, show `+N more` (gray pill, opens modal). Always show `✕` after chips.

### Worker Data Model

```js
{
  id:       string,   // internal key e.g. 'jl', 'em'
  initials: string,   // 'JL', 'EM' — displayed in avatar circle
  name:     string,   // full name
  agencyId: string,   // 'SAA-1042', 'MBT-3312'
  inTime:   string,   // '12:30 PM' — empty string if not clocked in
  outTime:  string,   // '2:45 PM' — empty string if not clocked out
  shift:    number,   // 1 | 2 | 3
  line:     number,   // e.g. 5, 14
  date:     string,   // 'May 14', 'May 13'
  past:     boolean   // false = current shift, true = historical
}
```

### Row Layout

Each row is a flex container, `min-height: 72px`, `padding: 14px 20px`.

```
[ Avatar ][ Worker meta (210px) ][ CHECK-IN col (110px) ][ CHECK-OUT col (110px) ][ Buttons → ]
```

**Avatar** — 46px circle, blue background, white initials (15px, 700 weight)

**Worker meta** — fixed 210px, `flex-shrink: 0`:
- Name: 17px, 700 weight
- Agency ID: 13px, `--text-sub`
- Context line (past workers only): 12px, blue, 600 weight — `"Line 5 · Shift 2 · May 13"`

**CHECK-IN column** — fixed 110px, `flex-shrink: 0`:
- Label: `"CHECK-IN"` — 11px, 700 weight, uppercase, green (`--green-text`), letter-spacing 0.06em
- Time: 20px, 700 weight, `white-space: nowrap` (past workers: 18px, `--text-sub`)

**CHECK-OUT column** — fixed 110px, `flex-shrink: 0` (hidden if no outTime):
- Label: `"CHECK-OUT"` — 11px, 700 weight, uppercase, red (`--red-text`)
- Time: 20px, 700 weight (past: 18px, gray)

**Buttons** — `margin-left: auto`, flex row, gap 8px:
- If checked in only: `[Edit Check-In]` (blue filled) + `[Check Out]` (red outline)
- If checked in + out: `[Edit Check-In]` (blue filled) + `[Edit Check-Out]` (red outline)
- Both buttons: 56px tall min, 110px wide min, pencil icon prefix

**Past worker rows**: background `#fafbfc`

### Section Grouping & Sort

Workers are split into two groups:

1. **Current shift** — `past: false` workers  
   Section header: `"Current Shift — Line 5 · May 14, 2026 · Shift 2"`  
   Sorted by `inTime` descending (latest check-in first)

2. **Past workers** — `past: true` workers, grouped by unique `(date, line, shift)` combination  
   Section header per group: `"{date} · Line {line} · Shift {shift}"`  
   Within each group: sorted by `inTime` descending

Section headers are a gray band: 11px, 700 weight, uppercase, `--gray-text`, `--page-bg` background.

---

## Screen 3 — Audit Log

### Purpose

Read-only chronological log of all badge scan and clock events on the line. Supervisors can search and filter but cannot edit anything here.

### Layout (top to bottom)

1. Header
2. Blue alert banner: `"Read-only record of all badge scans and clock events on this line. Use filters to search across lines, shifts, agencies, and date ranges."`
3. Search + filter bar
4. Column header band (fixed, gray)
5. Scrollable list body (endless scroll)

### Search + Filter Bar

Same pattern as Manage Workers:
```
[ Search input (flex: 1) ]  [ chips ][ ✕ ][ Filters btn ]
```
- Search: matches `name`, `agencyId`, or `uuid` (case-insensitive, live)

### Filter State

**Defaults:**
```js
{
  lines:  [5],     // array; empty = all lines
  shift:  'all',   // 'all' | '1' | '2' | '3'
  date:   'today', // 'today' | 'yesterday' | 'week' | 'month'
  type:   'all',   // 'all' | 'checkin' | 'checkout' | 'abandoned' | 'error'
  agency: 'all'    // 'all' | 'saa' | 'mbt'
}
```

Same two-object pattern (applied vs pending), same chip/overflow behavior as Manage Workers.

### Filter Modal Sections

**Line** — 6-column grid (lines 1–18 + All Lines)  
**Shift** — 3-column grid (All Shifts, Shift 1, Shift 2, Shift 3)  
**Date** — vertical stacked pills (Today, Yesterday, Past 7 Days, Past 30 Days) — no custom calendar  
**Event Type** — 3-column grid:  
- All Events, Check-In, Check-Out (row 1)  
- Abandoned, Errors (row 2)  

**Agency** — 3-column grid (All Agencies, SAA, MBT)

### Column Layout

Column headers sit in a fixed light-gray band (`#f0f3f6`) between the search bar and first data row.

| Column | Width | Header label |
|---|---|---|
| Event | 160px | EVENT |
| Time | 180px | TIME |
| Worker | 360px | WORKER |
| Line · Shift | flex (remaining) | LINE · SHIFT |

Row container: `display: flex`, `gap: 24px`, `padding: 12px 20px`, `min-height: 72px`. Column headers use the same `gap: 24px` so labels align perfectly with data.

### Column Content

**Event (160px)**  
Badge pill only — colored dot (10px SVG circle) + label text:

| Type | Dot color | Label | Badge class |
|---|---|---|---|
| checkin | `#27ae60` | CHECK-IN | `.badge-green` |
| checkout | `#c0392b` | CHECK-OUT | `.badge-red` |
| abandoned | `#e67e22` | ABANDONED | `.badge-orange` |
| error | `#c0392b` | ERROR | `.badge-red` |

**Time (180px)**  
- Time value: 19px, 700 weight (e.g. `"4:09 PM"`)
- Date below: 12px, `--text-sub` (e.g. `"05/14/2026"`)

Source field: `datetime` string in format `"MM/DD H:MM AM/PM"` — split on first space to get date, remainder is time; append `/2026` to date for display.

**Worker (360px)**  
- Name: 15px, 700 weight
- ID: 12px, `--text-sub`, monospace, margin-top 2px
- Note: 12px, `--red-text`, italic, margin-top 2px (only if `note` field exists)

For unknown badge rows (no `name`/`agencyId`, UUID only):
- Name field: show the `uuid` value
- ID field: show `"Unknown badge"`
- Note field: show `note` if present (e.g. `"Invalid badge"`, `"Abandoned after 3s"`)

**Line · Shift (flex)**  
- Line: 15px, 700 weight (e.g. `"Line 5"`)
- Shift: 12px, `--text-sub`, margin-top 2px (e.g. `"Shift 2"`)

### Audit Event Data Model

```js
{
  type:     string,        // 'checkin' | 'checkout' | 'abandoned' | 'error'
  name:     string | null, // Worker full name, or null for unknown badge
  agencyId: string | null, // e.g. 'SAA-0831', or null
  uuid:     string | null, // Raw badge UUID, or null for known workers
  line:     number,        // e.g. 5, 14
  shift:    number,        // 1 | 2 | 3
  agency:   string | null, // 'SAA' | 'MBT' | null (null for errors)
  datetime: string,        // 'MM/DD H:MM AM/PM' e.g. '05/14 4:09 PM'
  note:     string | null  // e.g. 'Invalid badge', 'Abandoned after 6s'
}
```

### Sort

Always sort descending by `datetime` (newest first) after filtering. Parse `datetime` string as:

```js
// "05/14 4:09 PM" → Date object
const [md, time, ampm] = dt.split(' ');
const [month, day] = md.split('/');
let [h, min] = time.split(':');
h = parseInt(h);
if (ampm === 'PM' && h !== 12) h += 12;
if (ampm === 'AM' && h === 12) h = 0;
return new Date(2026, parseInt(month) - 1, parseInt(day), h, parseInt(min));
```

---

## Screen 4 — Manage Worker Training Types

### Purpose

Supervisors assign one or more training types to each worker clocked in for the shift. This is a bulk-entry screen — all workers are shown at once, and changes are saved together.

### Training Types

`['Stocking', 'Packing', 'Folding', 'Receiving', 'Labeling', 'Other', 'N/A']`

### Worker Card Layout

Each worker gets a card with:
- **Header row**: Avatar + Name (17px, 700) + Agency ID (13px, gray) + warning `"⚠ No types assigned"` (13px, red, only if nothing selected)
- **Chip row**: One pill per training type, flex-wrap
  - Unselected: transparent bg, gray border
  - Selected: blue bg, white text
  - N/A: mutually exclusive with all other types (selecting N/A clears selections; selecting any type clears N/A)
  - "Other": reveals a text input for a free-form note when selected

### State per worker

```js
{
  selected:  string[],  // e.g. ['Stocking', 'Packing']
  na:        boolean,   // true if N/A is selected
  otherNote: string     // free text when 'Other' is selected
}
```

### Alert Banner States

- **Empty** (no assignments yet): blue banner — `"Select the type of work each worker performed during this shift."`
- **In progress** (some assigned, some not): amber banner — `"⚠ Action required: N worker(s) are missing assigned training types."` + `"Show missing only"` toggle button

### Footer

Fixed to bottom of screen: full-width `"Save all"` primary button.

---

## Modals

All modals use a full-screen dark overlay (rgba 0,0,0,0.45) and a centered white card (max-width 520px, max-height 90vh).

### Edit Check-In Time / Edit Check-Out Time

**Triggered by**: Edit Check-In / Edit Check-Out buttons on Manage Workers rows.

**Contents:**
- Title + worker name + agency ID as subtitle
- Current recorded time display
- Time entry: two text inputs (HH, MM) + AM/PM toggle buttons
- 3×4 numpad (digits 1–9, backspace, 0)
  - Numpad appends to whichever field (HH or MM) is active
  - Auto-advances from HH to MM after 2 digits
- Footer: **Save** (primary) + **Cancel** (secondary)
- On save: show toast `"Check-in time updated"` / `"Check-out time updated"`

### Check Out All Workers

**Triggered by**: "Check Out All Workers" row on admin menu.

- Confirmation modal: counts active workers on current line
- Buttons: **Check Out All** (red/danger) + **Cancel**

### Log Out

- Confirmation modal
- Buttons: **Log Out** (red/danger) + **Cancel**

---

## Toast Notifications

- Position: bottom-center, fixed
- Style: dark green background (`#1e7e3e`), white text, 14px
- Behavior: fade in on show, auto-dismiss after 2.8s

---

## Shared UI Patterns

### Badge Pills

```
● LABEL
```

Small colored dot (SVG circle, 10px) + uppercase label text. Used in Audit Log rows and the admin menu "Check Out All" row.

| Class | Background | Text color |
|---|---|---|
| `.badge-green` | `--green-bg` | `--green-text` |
| `.badge-red` | `--red-bg` | `--red-text` |
| `.badge-orange` | `--orange-bg` | `--orange-text` |
| `.badge-gray` | `--gray-bg` | `--gray-text` |
| `.badge-blue` | `--blue-light` | `--blue` |

### Buttons

| Class | Style |
|---|---|
| `.btn-primary` | Blue bg, white text |
| `.btn-secondary` | Transparent, gray border + text |
| `.btn-danger` | Red bg (`--red-text`), white text |
| `.btn-outline-red` | Transparent, red border + text |

All interactive buttons: `min-height: var(--touch-min)` (56px) for tablet tap targets. ✕ clear chip: 44×44px.

### Alert Banners

Two variants: `.alert-banner.blue` and `.alert-banner.amber`.  
Both: light tinted background, matching-color bottom border, 14px body text.

### Section Headers (Manage Workers)

Gray band between groups of rows:  
- Background: `--page-bg`
- Text: 11px, 700 weight, uppercase, `--gray-text`, letter-spacing 0.07em
- Border-bottom: 1px `#e8ecef`

---

## Key Implementation Notes

1. **Pending vs applied filter state** — Always maintain two filter objects (applied and pending). The UI reflects applied state; the modal edits pending state. "Apply" copies pending → applied. "Reset" restores pending to defaults without closing the modal.

2. **Line selection is multi-select** — `lines` is always an array. Empty array means "All Lines." Single line default is `[5]`.

3. **Shift and date are single-select strings** — not arrays.

4. **Sort is always applied after filter** — never store pre-sorted data; sort at render time.

5. **Time parsing** — `inTime`/`outTime` are plain strings like `"12:30 PM"`. Parse to minutes-since-midnight for comparison: `h*60 + min`, adjusting for AM/PM.

6. **Fixed column widths with `flex-shrink: 0`** — critical for both MW and Audit Log rows so content never shifts based on data length. Use `gap` on the row container rather than padding on individual cells for consistent inter-column spacing.

7. **Search is live** — fire on every `input` event, no debounce needed at this data scale.

8. **All chip and filter logic is client-side** — the mockup has no API calls. Wire these up to your backend filter/search endpoints as needed.

9. **The "past" worker concept** — a worker is "past" if their record belongs to a prior shift, line, or date. Past rows render with a slightly gray background and dimmed time values, plus a context line showing their Line · Shift · Date.

10. **Tablet-first, not mobile** — minimum touch target is 56px (buttons) or 44px (chips/icons). The layout assumes 1280px landscape. No hamburger menu or bottom nav needed.
