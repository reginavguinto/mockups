# EC Line Scanner — Admin Panel Developer Spec
**Prepared for:** Peter  
**Reference mockup:** `line-scanner-admin-v2.html`  
**Last updated:** May 14, 2026

---

## Overview

The admin panel is a tablet-first UI (15" landscape, max-width 1280px) used by line supervisors to manage worker clock events, view audit history, and assign training types. It is a single-page app with screen-based navigation (no routing). All screens share a common header shell.

Each line scanner device is associated with a specific line at the facility. Throughout this document, **{LINE}** refers to the line number/name the device is registered to, **{SHIFT}** refers to the currently active shift, and **{DATE}** refers to today's date — all resolved at runtime from device/session context.

---

## Device Context

Every screen in the admin panel is scoped to a device context that must be resolved on login:

| Field | Description | Example |
|---|---|---|
| `currentLine` | The line this scanner is physically assigned to | `5` |
| `currentShift` | The shift currently active based on the clock | `2` |
| `currentDate` | Today's date | `2026-05-14` |

These values are treated as **read-only context**, not user preferences. They are never persisted to session storage — always re-derived from the server/device on login.

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
- Each screen is a `div.screen` hidden by default; the active screen gets class `.active`
- Navigation sets the active screen and calls the appropriate render function
- All screens share the same fixed header structure

### Header

- Height: 72px, blue background (`--blue`)
- Left: back `←` button (44px tap target) + screen title
- Right: two-line context block, right-aligned, always visible:
  - Line 1: `"{LINE_NAME} · {DATE}"` — 14px, 600 weight, white
  - Line 2: `"{CURRENT_TIME} · Shift {SHIFT}"` — 12px, white 75% opacity

The context block reflects the device's current line, active shift, and today's date. It does not update to reflect filter selections — it always shows where the scanner physically is right now.

---

## Screen 1 — Admin Menu

Entry point after login. Lists all available actions.

### Menu Items

| Label | Subtitle | Destination |
|---|---|---|
| Manage Worker Times ({LINE_NAME}) | View and edit worker check-in and check-out times for this line | Manage Workers screen |
| Manage Worker Training Types ({LINE_NAME}) | View and edit training types for workers checked in to this line | Training Types screen |
| Audit Log | View badge scans, check-in and check-out history | Audit Log screen |
| System Settings | — | Coming Soon screen |
| Check Out All Workers | Red-accented row, triggers confirmation modal | Modal |

The `{LINE_NAME}` in the label parentheses is dynamically populated from `currentLine` (e.g. `"Line 5"`, `"Line 12"`). It communicates to the supervisor which line they are scoped to by default on those screens.

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

**Defaults** (resolved at runtime from device context):
```js
{
  lines:       [currentLine],  // the device's assigned line — always an array
  shift:       currentShift,   // the currently active shift as a string e.g. '2'
  date:        'today',
  customStart: null,           // { y, m, d } or null
  customEnd:   null
}
```

### Filter Persistence

Filters are split into two categories with different persistence rules:

**Never persisted — always reset to device context on login:**
| Field | Reason |
|---|---|
| `lines` | The scanner is physically on a specific line; defaulting to another line would be disorienting |
| `date` | "Today" on a new login is always the current date; yesterday's date filter would be stale and confusing |

**Persisted to `sessionStorage` for the duration of the browser session:**
| Field | Reason |
|---|---|
| `shift` | An admin investigating a prior shift mid-day should have that context preserved across quick re-logins |
| `customStart` / `customEnd` | If a custom date range was set, retain it within the session |

On login: read `shift` (and custom dates if applicable) from `sessionStorage`. If no stored value, use the device default. On logout or new calendar day: clear session storage.

**Detecting a new calendar day:** compare stored `sessionDate` (set at login) against `new Date().toDateString()`. If different, treat as a fresh session and discard all stored filter state.

### Filter Modal

Slides in from the right (420px wide, full height). Semi-transparent overlay behind it.

Sections (top to bottom):

**Line** — 6-column grid
- "All Lines" pill (selects empty array — shows all lines)
- Pills for each available line (multi-select; the number of lines is facility-configured)

**Shift** — 3-column grid
- All Shifts, Shift 1, Shift 2, Shift 3 (single-select; expand if facility has more shifts)

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

Chips are shown only for filters that differ from the current defaults. A chip for `lines` only appears if a line *other than* the device's assigned line is selected (or if "All Lines" is selected).

| Filter changed | Chip label |
|---|---|
| Lines | `"Line {N}"` or `"Line {N}, {M}"` (comma-separated) or `"All Lines"` |
| Shift | `"Shift {N}"` or `"All Shifts"` |
| Date | `"Yesterday"` / `"Past 7 Days"` / `"Past 30 Days"` |
| Custom date | `"{start} → {end}"` or `"{start}"` (if only start set) |

Max 2 chips visible. If 3+, show `+N more` (gray pill, opens modal). Always show `✕` after chips.

### Worker Data Model

```js
{
  id:       string,   // internal identifier
  initials: string,   // 2-char display initials for avatar
  name:     string,   // full name
  agencyId: string,   // agency worker ID e.g. 'SAA-1042'
  inTime:   string,   // '12:30 PM' — empty string if not clocked in
  outTime:  string,   // '2:45 PM' — empty string if not clocked out
  shift:    number,   // 1 | 2 | 3 (or however many shifts are configured)
  line:     number,   // which line this record belongs to
  date:     string,   // display date e.g. 'May 14'
  past:     boolean   // false = current shift, true = historical record
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
- Context line (past workers only): 12px, blue, 600 weight — `"Line {N} · Shift {N} · {Date}"`

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
   Section header: `"Current Shift — {LINE_NAME} · {DATE} · Shift {SHIFT}"`
   Sorted by `inTime` descending (latest check-in first)

2. **Past workers** — `past: true` workers, grouped by unique `(date, line, shift)` combination
   Section header per group: `"{date} · Line {N} · Shift {N}"`
   Within each group: sorted by `inTime` descending

Section headers are a gray band: 11px, 700 weight, uppercase, `--gray-text`, `--page-bg` background.

---

## Screen 3 — Audit Log

### Purpose

Read-only chronological log of all badge scan and clock events. Supervisors can search and filter but cannot edit anything here.

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

**Defaults** (resolved at runtime from device context):
```js
{
  lines:  [currentLine],  // device's assigned line
  shift:  'all',          // Audit Log defaults to all shifts (broader view)
  date:   'today',
  type:   'all',          // all event types
  agency: 'all'           // all agencies
}
```

### Filter Persistence

Same rules as Manage Workers, with two additional fields:

**Never persisted:**
| Field | Reason |
|---|---|
| `lines` | Always defaults to the device's line |
| `date` | Always defaults to today |

**Persisted to `sessionStorage`:**
| Field | Reason |
|---|---|
| `shift` | Admin may be reviewing a specific shift throughout their session |
| `type` | Admin investigating errors or a specific event type should retain that view |
| `agency` | Admin scoped to a specific agency workforce should retain that context |

Same new-calendar-day detection applies: if `sessionDate` doesn't match today, clear all stored filter state.

### Filter Modal Sections

**Line** — 6-column grid (all available lines + "All Lines")
**Shift** — 3-column grid (All Shifts, Shift 1, Shift 2, Shift 3)
**Date** — vertical stacked pills (Today, Yesterday, Past 7 Days, Past 30 Days) — no custom calendar
**Event Type** — 3-column grid:
- All Events, Check-In, Check-Out (row 1)
- Abandoned, Errors (row 2)

**Agency** — 3-column grid (All Agencies + one pill per configured agency)

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

Source: `datetime` string in format `"MM/DD H:MM AM/PM"`. Split on first space to separate date and time; append the 4-digit year to the date portion for display.

**Worker (360px)**
- Name: 15px, 700 weight
- ID: 12px, `--text-sub`, monospace, margin-top 2px
- Note: 12px, `--red-text`, italic, margin-top 2px (only if `note` field is present)

For unknown badge rows (UUID-only, no resolved worker):
- Name field: show the raw `uuid` value
- ID field: show `"Unknown badge"`
- Note field: show `note` if present (e.g. `"Invalid badge"`, `"Abandoned after 3s"`)

**Line · Shift (flex)**
- Line: 15px, 700 weight (e.g. `"Line 5"`)
- Shift: 12px, `--text-sub`, margin-top 2px (e.g. `"Shift 2"`)

### Audit Event Data Model

```js
{
  type:     string,        // 'checkin' | 'checkout' | 'abandoned' | 'error'
  name:     string | null, // Resolved worker full name, or null for unknown badge
  agencyId: string | null, // Agency worker ID e.g. 'SAA-0831', or null
  uuid:     string | null, // Raw badge UUID, or null for known workers
  line:     number,        // Which line the event occurred on
  shift:    number,        // Which shift was active
  agency:   string | null, // Agency code e.g. 'SAA', 'MBT', or null for errors
  datetime: string,        // 'MM/DD H:MM AM/PM' e.g. '05/14 4:09 PM'
  note:     string | null  // Optional detail e.g. 'Invalid badge', 'Abandoned after 6s'
}
```

### Sort

Always sort descending by `datetime` (newest first) after filtering. Parse the `datetime` string:

```js
const [md, time, ampm] = dt.split(' ');
const [month, day] = md.split('/');
let [h, min] = time.split(':');
h = parseInt(h);
if (ampm === 'PM' && h !== 12) h += 12;
if (ampm === 'AM' && h === 12) h = 0;
return new Date(year, parseInt(month) - 1, parseInt(day), h, parseInt(min));
// where `year` is the current year from device context
```

---

## Screen 4 — Manage Worker Training Types

### Purpose

Supervisors assign one or more training types to each worker clocked in for the current shift. This is a bulk-entry screen — all workers are shown at once, and changes are saved together.

### Training Types

The list of training types is facility-configured. The mockup uses:
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

- Confirmation modal: `"This will check out all {N} workers currently on {LINE_NAME}."`
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

1. **Device context is not user preference** — `currentLine`, `currentShift`, and `currentDate` are resolved from the server/device on login, never from stored state. Treat them as constants for the session.

2. **Pending vs applied filter state** — Always maintain two filter objects (applied and pending). The UI reflects applied state; the modal edits pending state. "Apply" copies pending → applied. "Reset" restores pending to defaults without closing the modal.

3. **Filter persistence via `sessionStorage`** — Only persist `shift`, `type`, and `agency`. Never persist `lines` or `date`. On each login, check if `sessionDate` matches today — if not, discard all stored filter state and start fresh.

4. **Line selection is multi-select** — `lines` is always an array. Empty array means "All Lines." The default is `[currentLine]` (a single-element array).

5. **The number of lines and shifts is facility-configured** — do not hardcode 18 lines or 3 shifts. Render line and shift pills dynamically from the facility configuration.

6. **Active filter chips only show non-default values** — a chip for `lines` only appears when a line other than `currentLine` is selected (or "All Lines" is selected). If the user picks their own line back, the chip disappears.

7. **Sort is always applied after filter** — never store pre-sorted data; sort at render time.

8. **Time parsing** — `inTime`/`outTime` are plain strings like `"12:30 PM"`. Parse to minutes-since-midnight for sort comparison: `h*60 + min`, adjusting for AM/PM.

9. **Fixed column widths with `flex-shrink: 0`** — critical for both Manage Workers and Audit Log rows so content never shifts based on data length. Use `gap` on the row container rather than padding on individual cells for consistent inter-column spacing.

10. **Search is live** — fire on every `input` event. No debounce needed at this data scale, but add one (~150ms) if querying a backend.

11. **The "past" worker concept** — a worker record is "past" if it belongs to a prior shift, line, or date relative to the current filters. Past rows render with a slightly gray background (`#fafbfc`), dimmed time values, and a context line showing `Line · Shift · Date`.

12. **Tablet-first, not mobile** — minimum touch target is 56px (buttons) or 44px (chips/icons). The layout assumes 1280px landscape. No hamburger menu or bottom nav needed.
