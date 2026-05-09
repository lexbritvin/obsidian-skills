---
name: obsidian-heatmap
description: Build activity heatmaps in Obsidian — GitHub-style year overview via the Heatmap Calendar plugin (Richardsl), and custom DataviewJS heatmaps (weeks × days, rows × cols matrices, project activity grids) when the plugin is not enough. Use whenever the user asks about heatmaps, calendar visualizations, streaks, "did I do X every day", year-in-pixels, contribution graphs, GitHub-style activity views, daily-log visualizations, or wants to render any time-based intensity grid in a note.
---

# Obsidian heatmaps

Two complementary tools cover almost every heatmap need in Obsidian:

1. **Heatmap Calendar plugin** (Richardsl) — GitHub-style **year overview**, one call from a `dataviewjs` block. Handles 365-day grid, month labels, weekday rows, today border, color gradients, click/hover. **Use this first** when the question is "did I do X on each day of the year".

2. **Custom DataviewJS heatmaps** — anything the plugin cannot draw: weeks × 7 days for the last N weeks, rows × cols matrix (e.g. metrics × weeks), rows × N days (projects × dates), custom score functions, multi-metric tooltips, streaks. The plugin draws **a year**; custom DOM draws **whatever shape you need**.

Pick the plugin first; drop to custom only when the year frame is wrong or you need a non-calendar shape.

## Decision: which heatmap

| Need | Tool |
|---|---|
| One year, one metric, "did I do X today" | Plugin |
| One year, two-three metrics overlaid by color | Plugin (multi-color entries) |
| Last N weeks (e.g. 14), GitHub-contribution shape | Custom DataviewJS — weeks × 7 days |
| Rows × cols matrix (metrics × weeks, projects × months) | Custom DataviewJS — grid |
| Rows × days (projects × dates with sticky row labels) | Custom DataviewJS — activity grid |
| Cell holds an emoji / number / link, not just intensity | Plugin (`content` field) or custom |
| Multiple years on one screen | Two plugin blocks, one per `year` |
| "Cell is a task you can click" | Custom DOM with click handlers |

If the user shows a GitHub contributions screenshot — that is the **last 53 weeks** scrolling shape, not a calendar year. The plugin always renders Jan 1 → Dec 31 of `year`. For a rolling-N-weeks contribution graph use the custom pattern.

## Part A — Heatmap Calendar plugin (Richardsl)

Repo: `Richardsl/heatmap-calendar-obsidian`. Install via Community plugins in Obsidian. Exposes a single global function on plugin load.

### API surface

```javascript
renderHeatmapCalendar(container, calendarData)
```

`container` is the DOM element to render into — from a `dataviewjs` block use `this.container`. The function is on `window`, so just call `renderHeatmapCalendar(...)`.

### `calendarData` fields

| Field | Type | Default | Notes |
|---|---|---|---|
| `entries` | array | required | One object per day with data; missing days render as empty |
| `year` | number | current | Plugin filters entries to this year (`new Date(date).getFullYear() === year`) |
| `colors` | string \| object | `"default"` | Either name a palette saved in plugin settings, or pass an inline object |
| `showCurrentDayBorder` | boolean | true | Outlines today's cell |
| `defaultEntryIntensity` | number | 4 | Used for entries without `intensity` |
| `intensityScaleStart` | number | min(intensities) | Lower bound mapped to color level 1 |
| `intensityScaleEnd` | number | max(intensities) | Upper bound mapped to top color level |
| `separateMonths` | boolean | false | Adds spacing between months for per-month styling |

### Entry object

```javascript
{
  date: "YYYY-MM-DD",   // required, must be ISO; daily-note filenames work directly
  intensity: 3,         // optional, falls back to defaultEntryIntensity
  content: "🏋️",       // optional, rendered inside the cell (emoji, text, dv.span(...))
  color: "green"        // optional, key into colors object; defaults to first key
}
```

### Color palettes

Each palette is an array of HEX or `rgba()` strings — one per intensity level. **More entries = finer gradient.**

```javascript
colors: {
  green: ["#c6e48b", "#7bc96f", "#49af5d", "#2e8840", "#196127"],   // GitHub default
  blue:  ["#8cb9ff", "#69a3ff", "#428bff", "#1872ff", "#0058e2"],
  red:   ["#ffb3b3", "#ff7373", "#ff3333", "#cc0000", "#800000"],
}
```

Two ways to pass:
- **Inline** — pass the object directly in `calendarData.colors`. Self-contained, lives in the note.
- **Named** — define the palette once in Settings → Heatmap Calendar, then pass `colors: "myPalette"` (string). Reusable across notes; survives moving the block.

How intensity → color level works: plugin scales each entry's `intensity` from `[intensityScaleStart, intensityScaleEnd]` to `[1, palette.length]`, rounds, and picks the corresponding HEX. So a palette of 5 colors gives 5 visible levels regardless of raw values.

### CSS hooks

The plugin renders a known DOM tree you can style via Settings → Appearance → CSS snippets:

| Selector | What |
|---|---|
| `.heatmap-calendar-graph` | The whole container |
| `.heatmap-calendar-year` | Two-digit year label (top-left) |
| `.heatmap-calendar-months` | Month axis (Jan…Dec) |
| `.heatmap-calendar-days` | Weekday axis (Mon…Sun) |
| `.heatmap-calendar-boxes` | The 7 × 53 cell grid |
| `.heatmap-calendar-boxes li.today` | Current day's cell |
| `.heatmap-calendar-boxes li.hasData` | Cells with entries |
| `.heatmap-calendar-boxes li.isEmpty` | Cells without entries |
| `.heatmap-calendar-boxes li.month-jan` … `.month-dec` | Per-month styling |
| `.heatmap-calendar-content` | The `content` text/emoji span inside a cell |
| `[data-date="YYYY-MM-DD"]` | Pick a specific cell by date |

Cell `background-color` is inlined — to override, use `!important` in your snippet.

### Minimal example

```dataviewjs
const data = {
  year: 2026,
  entries: dv.pages('"Daily"')
    .where(p => p.exercise)
    .map(p => ({
      date: p.file.name,           // daily-note name = "YYYY-MM-DD"
      intensity: p.exercise,
      content: "🏋️",
    }))
    .array(),
};
renderHeatmapCalendar(this.container, data);
```

### Multi-color overlay

Two metrics on the same year — entries with the same date can both render (last one wins per cell, so prefer disjoint days or accept overwrite):

```dataviewjs
const data = {
  year: 2026,
  colors: {
    green: ["#c6e48b", "#7bc96f", "#49af5d", "#2e8840", "#196127"],
    blue:  ["#8cb9ff", "#69a3ff", "#428bff", "#1872ff", "#0058e2"],
  },
  intensityScaleStart: 1,
  intensityScaleEnd: 60,
  entries: [],
};

for (const p of dv.pages('"Daily"').where(p => p.exercise)) {
  data.entries.push({ date: p.file.name, intensity: p.exercise, content: "🏋️", color: "green" });
}
for (const p of dv.pages('"Daily"').where(p => p.meditation)) {
  data.entries.push({ date: p.file.name, intensity: p.meditation, content: "🧘", color: "blue" });
}

renderHeatmapCalendar(this.container, data);
```

When two metrics matter equally per day, **don't** overlay — split into two heatmap blocks (one per metric) stacked vertically. Overlay is for highlighting outliers, not comparison.

### Pick a data source first

The plugin doesn't care where data comes from — but the choice of source shapes everything else (tracking friction, edit ergonomics, what queries are cheap). Pick deliberately:

| Data you have | Best source | Reasoning |
|---|---|---|
| One scalar metric per day, you keep daily notes anyway | frontmatter on daily note (`workout: 3`) | Cell click jumps to the note; rich context per day |
| Multiple metrics per day, no daily note overhead | inline-fields in monthly log (`- YYYY-MM-DD [pages:: 32] [minutes:: 45]`) | One file per month, all metrics on one row, low-friction edit |
| Already tracking via Tasks plugin | task `✅ YYYY-MM-DD` markers | Reuses existing task discipline; intensity = count of completions |
| Pure binary habit ("did X today?") | tag presence on daily note (`#workout` in `2026-04-25.md`) | Lightweight; one tag per day per habit; no field schema |
| "How many things of kind X happened on day Y?" | count of pages tagged `#X` with `cday` on Y | Works without daily notes; aggregates any tagged content |

These are not exclusive — you can run two heatmap blocks side by side, one driven by frontmatter and another by tags. **Don't pick more than one source for the same metric** in the same heatmap; mixed sources produce inconsistent results when the same date is recorded twice.

### Pattern — entries from frontmatter

Daily-note frontmatter is the cleanest source: one note per day, named `YYYY-MM-DD.md`, frontmatter has `exercise: 3` etc.

```dataviewjs
renderHeatmapCalendar(this.container, {
  year: dv.date("today").year,
  entries: dv.pages('"Daily"')
    .where(p => p.workout != null)
    .map(p => ({ date: p.file.name, intensity: p.workout, content: "🏋️" }))
    .array(),
});
```

### Pattern — entries from inline-fields in monthly logs

When you don't keep a daily note per day but log inside a monthly file using flat bullets — for example, a reading log:

```
- 2026-04-25 [pages:: 32] [minutes:: 45]
- 2026-04-26 [pages:: 18] [minutes:: 25]
- 2026-04-27 [pages:: 60] [minutes:: 90]
```

Dataview parses these into list items with date in `text` and inline-fields as keys. Score them and render:

```dataviewjs
// Pages is the goal; minutes adds a small bonus so a long careful read still registers
const score = e => (e.pages ?? 0) + (e.minutes ?? 0) * 0.3;

const entries = dv.pages('"Logs/Monthly"')
  .file.lists
  .where(l => /^\d{4}-\d{2}-\d{2}/.test(l.text))
  .map(l => {
    const date = l.text.match(/^\d{4}-\d{2}-\d{2}/)[0];
    return { date, intensity: score(l), content: "📖" };
  })
  .array();

renderHeatmapCalendar(this.container, { year: dv.date("today").year, entries });
```

Adapt the schema to whatever you track — running (`km`, `minutes`), writing (`words`, `sessions`), language study (`new_words`, `minutes`). The score function is where you decide what the cell intensity *means*: pure output, time-weighted, or composite.

### Pattern — entries from task counts

Cell intensity = number of tasks completed on that day, from `✅ YYYY-MM-DD` markers in any task:

```dataviewjs
const byDate = new Map();
for (const t of dv.pages().file.tasks.where(t => t.completed && t.completion)) {
  const d = t.completion.toString().slice(0, 10);
  byDate.set(d, (byDate.get(d) ?? 0) + 1);
}
const entries = Array.from(byDate, ([date, n]) => ({ date, intensity: n, content: "" }));
renderHeatmapCalendar(this.container, { year: dv.date("today").year, entries });
```

### Pattern — entries from tag presence

Two flavors, depending on what "presence" means.

**Binary habit on daily notes** — daily note named `YYYY-MM-DD.md` has tag `#workout` → that day is "done":

```dataviewjs
const TAG = "#workout";
const entries = dv.pages('"Daily"')
  .where(p => p.file.tags.includes(TAG))
  .map(p => ({ date: p.file.name, intensity: 1, content: "🏋️" }))
  .array();
renderHeatmapCalendar(this.container, {
  year: dv.date("today").year,
  entries,
  intensityScaleStart: 0,
  intensityScaleEnd: 1,    // binary — every "done" day uses the top color
});
```

`p.file.tags` includes nested tags too (`#workout/morning` matches `#workout`). For exact-match only, compare manually: `p.file.tags.some(t => t === TAG)`.

**Count of tagged pages per day** — intensity = number of pages with `#read` whose creation date falls on day Y. Works without per-day notes:

```dataviewjs
const TAG = "#read";
const byDate = new Map();
for (const p of dv.pages(TAG)) {
  const d = p.file.cday.toFormat("yyyy-MM-dd");
  byDate.set(d, (byDate.get(d) ?? 0) + 1);
}
const entries = Array.from(byDate, ([date, n]) => ({ date, intensity: n, content: "📖" }));
renderHeatmapCalendar(this.container, { year: dv.date("today").year, entries });
```

`p.file.cday` is creation day; `p.file.mday` is last-modified day. For "what did I read today" use `cday`; for "what did I touch today" use `mday`. If your note has an explicit `date:` frontmatter, prefer that — file timestamps reflect filesystem operations, not the event the note describes.

**Multi-tag overlay** — one heatmap, many tags, color per tag:

```dataviewjs
const TAGS = [
  { tag: "#workout",    color: "orange", emoji: "🏋️" },
  { tag: "#meditation", color: "blue",   emoji: "🧘" },
  { tag: "#read",       color: "green",  emoji: "📖" },
];
const entries = [];
for (const { tag, color, emoji } of TAGS) {
  for (const p of dv.pages('"Daily"').where(p => p.file.tags.includes(tag))) {
    entries.push({ date: p.file.name, intensity: 1, content: emoji, color });
  }
}
renderHeatmapCalendar(this.container, {
  year: dv.date("today").year,
  colors: {
    orange: ["#ffa244", "#fd7f00"],
    blue:   ["#8cb9ff", "#428bff"],
    green:  ["#7bc96f", "#196127"],
  },
  entries,
});
```

Same caveat as multi-color overlay: when two tags hit the same day, the **last entry in the array wins** the cell. Order `TAGS` so the more important tag goes last, or split into separate heatmaps.

### Caveats with the plugin

- **One year per call.** `year` filters entries; cross-year data renders as empty in the wrong year. Stack two blocks for two years.
- **`renderHeatmapCalendar` may be undefined on first paint** if the plugin loads after the dataview block. Guard with `if (typeof renderHeatmapCalendar === 'function') {...}` and reload the note. Persistent issue → set "Show ribbon icon" off and on, or restart Obsidian.
- **Daily-note name must be `YYYY-MM-DD`.** If your daily notes use a different format, derive the date string explicitly (`p.date` from frontmatter, or `dv.func.dateformat(p.file.day, "yyyy-MM-dd")`).
- **Entries with the same date overwrite.** Last entry in array wins. If you build entries in a loop, latest insert is what renders.
- **`content` is a string, not HTML.** For interactive content (links to the daily note), pass `content: await dv.span(\`[[\${p.file.name}]]\`)` and the dataview span renders inline.
- **Hover preview needs Page Preview enabled.** Core plugin → Page Preview → toggle "Source mode" / "Reading view" depending on where the heatmap renders.
- **Color array length determines resolution.** 3 colors → 3 visible levels; 5 → 5; 10 → smooth gradient. Don't pass 100 — eyes can't tell.
- **`intensityScaleStart === intensityScaleEnd`** (or all entries have the same intensity) makes every cell the top color. Set explicit `start`/`end` if your data is bimodal.
- **`separateMonths: true`** breaks the visual rhythm of the GitHub layout — only enable if you want per-month dividers for styling.

### When to override colors via CSS instead of `colors` field

Use `colors` field when each block has its own palette. Use a CSS snippet when:
- You want the same palette across the entire vault and tire of repeating it.
- You want CSS variables driven by the active theme (light/dark).
- You want to tweak `.isEmpty` background (the inline `background-color` on `.hasData` overrides palette, but `.isEmpty` has only the CSS class, easily themable).

```css
/* CSS snippet — empty cells barely visible in dark theme */
.theme-dark .heatmap-calendar-boxes li.isEmpty {
  background-color: rgba(148, 163, 184, 0.06);
}
.heatmap-calendar-boxes li.today {
  outline: 1.5px solid var(--interactive-accent) !important;
}
```

## Part B — Custom DataviewJS heatmaps

When the year-frame doesn't fit. Three reusable shapes cover most needs.

### Pattern B1 — weeks × 7 days, last N weeks (GitHub-contribution shape)

This is what most "GitHub-style" requests actually want: rolling N weeks, scroll cuts off naturally, today on the right.

```dataviewjs
const WEEKS = 14;
const MAX = 60;                          // 60 pages saturates the color
const score = e => e.pages ?? 0;

// Source: flat bullets `- YYYY-MM-DD [pages:: N] [minutes:: M]`
const entries = [];
for (const l of dv.pages('"Logs/Monthly"').file.lists) {
  const m = l.text.match(/^(\d{4}-\d{2}-\d{2})/);
  if (!m) continue;
  entries.push({ date: m[1], pages: l.pages ?? 0, minutes: l.minutes ?? 0 });
}
const byDate = new Map(entries.map(e => [e.date, e]));

const todayDt = dv.date("today");
const start = todayDt.startOf("week").minus({ weeks: WEEKS - 1 });
const wrap = this.container.createEl("div", { attr: { style: "overflow-x: auto; padding: 8px 0;" } });
const flex = wrap.createEl("div", { attr: { style: "display: flex; gap: 3px; align-items: flex-start;" } });

// Day-of-week label column (Mon, Wed, Fri)
const dayCol = flex.createEl("div", { attr: { style: "display: flex; flex-direction: column; gap: 3px; padding-top: 16px;" } });
for (const dl of ["Mon", "", "Wed", "", "Fri", "", "Sun"]) {
  dayCol.createEl("div", { text: dl, attr: { style: "height: 16px; font-size: 9px; color: #64748b; line-height: 16px;" } });
}

// Week columns
for (let w = 0; w < WEEKS; w++) {
  const col = flex.createEl("div", { attr: { style: "display: flex; flex-direction: column; gap: 3px;" } });
  const weekStart = start.plus({ weeks: w });
  // Show month label on the column when first day of month falls in this week
  const monLabel = weekStart.day <= 7 ? weekStart.toFormat("LLL") : "";
  col.createEl("div", { text: monLabel, attr: { style: "height: 13px; font-size: 9px; color: #94a3b8; text-align: center;" } });

  for (let d = 0; d < 7; d++) {
    const cellDate = weekStart.plus({ days: d });
    const ds = cellDate.toFormat("yyyy-MM-dd");
    const data = byDate.get(ds);
    const isFuture = cellDate > todayDt;
    const isToday = ds === todayDt.toFormat("yyyy-MM-dd");

    let bg, title;
    if (isFuture) {
      bg = "rgba(148,163,184,0.04)";
      title = `${ds}: future`;
    } else if (data) {
      const sc = score(data);
      const intensity = Math.min(1, sc / MAX);
      const light = 25 + Math.round(intensity * 30);
      bg = `hsl(120, 65%, ${light}%)`;
      title = `${ds}: score ${sc}`;
    } else {
      bg = "rgba(239,68,68,0.10)";
      title = `${ds}: missed`;
    }
    const border = isToday ? "1.5px solid #3b82f6" : "1px solid transparent";
    col.createEl("div", { attr: { style: `width: 16px; height: 16px; background: ${bg}; border: ${border}; border-radius: 3px;`, title } });
  }
}
```

Knobs to tune: `WEEKS`, `MAX`, the `score` function, the empty/missed colors. Future cells stay barely visible; today gets a blue outline; missed cells use a soft red so gaps are visible at a glance.

### Pattern B2 — rows × cols matrix (metrics × weeks)

For "how does each component look across the last 8 weeks". Cell text shows the value; color encodes ratio to row-specific max.

```dataviewjs
// data shape:
//   rows = [{ key, label, max }]
//   cols = [{ key, label }]
//   value(rowKey, colKey) → number | null

// Habit tracker: rows = habits (each with its own weekly target), cols = recent weeks.
// Cell value = number of times that habit happened in that week.
const rows = [
  { key: "exercise",   label: "Exercise",   max: 5 },   // up to 5 sessions/wk (cap to avoid overtraining)
  { key: "reading",    label: "Reading",    max: 7 },   // daily target
  { key: "journaling", label: "Journaling", max: 7 },   // daily target
  { key: "language",   label: "Language",   max: 7 },   // daily target
];
const cols = [
  { key: "2026-W18", label: "W18" },
  { key: "2026-W17", label: "W17" },
  { key: "2026-W16", label: "W16" },
];
const data = {
  "2026-W18": { exercise: 4, reading: 6, journaling: 5, language: 7 },
  "2026-W17": { exercise: 5, reading: 7, journaling: 4, language: 6 },
  "2026-W16": { exercise: 3, reading: 5, journaling: 6, language: 4 },
};
const value = (rk, ck) => data[ck]?.[rk] ?? null;

const wrap = this.container.createEl("div", { attr: { style: "overflow-x: auto; padding: 8px 0;" } });
const grid = wrap.createEl("div", {
  attr: { style: `display: grid; grid-template-columns: 90px repeat(${cols.length}, minmax(56px, 1fr)); gap: 3px; font-size: 12px;` }
});

// header
grid.createEl("div", { text: "" });
for (const c of cols) {
  grid.createEl("div", { text: c.label, attr: { style: "color: #94a3b8; text-align: center; font-size: 10px; align-self: end;" } });
}

// rows
for (const r of rows) {
  grid.createEl("div", { text: r.label, attr: { style: "color: #cbd5e1; font-weight: 500; padding: 6px 4px; align-self: center;" } });
  for (const c of cols) {
    const v = value(r.key, c.key);
    const ratio = v == null ? null : Math.max(0, Math.min(1, v / r.max));
    let bg = "rgba(148,163,184,0.1)";
    let color = "#64748b";
    if (ratio != null) {
      const hue = Math.round(ratio * 120);          // 0 = red, 120 = green
      const light = 28 + Math.round(ratio * 22);    // 28% → 50%
      bg = `hsl(${hue}, 65%, ${light}%)`;
      color = ratio > 0.4 ? "white" : "#e2e8f0";
    }
    const cell = grid.createEl("div", {
      attr: {
        style: `background: ${bg}; border-radius: 4px; padding: 10px 4px; text-align: center; color: ${color}; font-weight: 600;`,
        title: `${r.label} ${c.label}: ${v ?? "—"}/${r.max}`,
      },
    });
    cell.textContent = v == null ? "—" : (Number.isInteger(v) ? v : v.toFixed(1));
  }
}
```

Use this when each row has its own max — habits with different weekly ceilings (exercise capped at 5/wk, reading at 7/wk), or score components that each cap at a different number. A single global max forces low-cap rows to always look saturated and high-cap rows to always look pale, hiding the signal.

### Pattern B3 — rows × N days (project activity, sticky labels)

Many rows (10+ projects), many columns (60 days), labels must stay visible while scrolling sideways.

```dataviewjs
const days = 60;
const cellSize = 11;
const today = dv.date("today").startOf("day");

// Source: rows = [{ name, files: [{ mtime }] }] — files anywhere with luxon mtime
const rows = [
  { name: "Project A", files: dv.pages('"Project A"').array().map(p => ({ mtime: p.file.mtime })) },
  { name: "Project B", files: dv.pages('"Project B"').array().map(p => ({ mtime: p.file.mtime })) },
].filter(r => r.files.length);

const stickyBg = "var(--background-primary)";
const labelStyle = `font-weight: 600; color: var(--text-normal); font-size: 11px; white-space: nowrap; position: sticky; left: 0; background: ${stickyBg}; z-index: 1; padding: 2px 6px 2px 0;`;
const cornerStyle = `position: sticky; left: 0; background: ${stickyBg}; z-index: 1;`;

const scroller = this.container.createEl("div", { attr: { style: "overflow-x: auto; padding-bottom: 4px;" } });
const grid = scroller.createEl("div", {
  attr: { style: `display: grid; grid-template-columns: 130px repeat(${days}, ${cellSize}px); gap: 2px; align-items: center;` }
});

// Header — month labels on the 1st of each month
grid.createEl("div", { attr: { style: cornerStyle } });
for (let d = 0; d < days; d++) {
  const date = today.minus({ days: days - 1 - d });
  const showLabel = date.day === 1 || d === 0;
  grid.createEl("div", { text: showLabel ? date.toFormat("LLL") : "", attr: { style: "font-size: 9px; color: var(--text-muted);" } });
}

// Data rows — bucket files into days, intensity = count
for (const r of rows) {
  const buckets = new Array(days).fill(0);
  for (const f of r.files) {
    const fday = dv.date(f.mtime).startOf("day");
    const diff = Math.floor(today.diff(fday, "days").days);
    if (diff >= 0 && diff < days) buckets[days - 1 - diff] += 1;
  }
  const max = Math.max(...buckets, 1);

  grid.createEl("div", { text: r.name, attr: { style: labelStyle } });
  for (let d = 0; d < days; d++) {
    const v = buckets[d];
    const intensity = v === 0 ? 0 : 0.25 + (v / max) * 0.75;
    const bg = v === 0 ? "rgba(148,163,184,0.07)" : `rgba(34,197,94,${intensity})`;
    grid.createEl("div", {
      attr: { style: `width: ${cellSize}px; height: ${cellSize}px; background: ${bg}; border-radius: 2px;`, title: v ? `${v} files` : "" }
    });
  }
}

// Auto-scroll to today on the right edge
requestAnimationFrame(() => { scroller.scrollLeft = scroller.scrollWidth; });
```

The `position: sticky` on row labels keeps them visible while the cells scroll horizontally — without it, labels disappear and you can't tell which row is which past day 30.

## Aux helpers

### Current streak

Days-in-a-row done, counted from today (or yesterday if today not yet recorded) backwards.

```javascript
const currentStreak = (entries) => {
  const todayDt = dv.date("today");
  const todayDs = todayDt.toFormat("yyyy-MM-dd");
  const lookup = new Map(entries.map(e => [e.date, e]));
  let cursor = lookup.has(todayDs) ? todayDt : todayDt.minus({ days: 1 });
  let streak = 0;
  while (true) {
    const ds = cursor.toFormat("yyyy-MM-dd");
    const data = lookup.get(ds);
    if (data && data.done) { streak++; cursor = cursor.minus({ days: 1 }); }
    else break;
  }
  return streak;
};
```

`done` can be derived: `data.done = (data.score > 0)` if the user doesn't track an explicit done flag.

### Missed days

Days with no entry between first recorded date and yesterday.

```javascript
const missedDays = (entries) => {
  if (!entries.length) return 0;
  const todayDs = dv.date("today").toFormat("yyyy-MM-dd");
  const lookup = new Set(entries.map(e => e.date));
  let missed = 0;
  let cursor = dv.date(entries[0].date);
  while (cursor.toFormat("yyyy-MM-dd") < todayDs) {
    if (!lookup.has(cursor.toFormat("yyyy-MM-dd"))) missed++;
    cursor = cursor.plus({ days: 1 });
  }
  return missed;
};
```

## Best practices

1. **Default to the plugin.** Year overview, the most common request, is one block of code with the plugin and 60 lines without. Drop to custom only when the shape genuinely doesn't fit.

2. **One metric per block.** Tempting to overlay everything; in practice readers can't decode multi-color cells. Stack two heatmaps over each other instead.

3. **Show empty days deliberately.** GitHub uses a barely-visible gray for empty-but-tracked days. Use a softer gray for "before tracking started" and a soft red for "missed" — the contrast between empty and missed is what makes streaks legible.

4. **Today border, not today fill.** If you fill today with the strongest color, the user can't read its intensity. Outline or border instead.

5. **Cell size matters at scale.** 11px cells give 53 weeks × 7 days a width of ~650px. 16px gives a friendlier shape but only ~30 weeks fit before scroll. Match cell size to viewport and how often the heatmap is reviewed.

6. **`title` attribute is your tooltip.** No need for a tooltip library — `title="..."` on each cell gives free hover text in any browser. Keep titles compact: `2026-04-25: 32 pages, 45 min` not multi-line.

7. **Color hue from data, lightness from intensity.** `hsl(${hue}, 65%, ${light}%)` lets you compare cells across rows with different scales. Pure red→green is intuitive for "bad → good"; if direction is ambiguous, stick to a single hue and vary lightness only.

8. **Future cells barely visible.** Don't render them as "missed" — they haven't happened yet. Use `rgba(148,163,184,0.04)` (almost transparent) so the grid shape stays but the user's eye doesn't read failure.

9. **Auto-scroll right** on rolling-window heatmaps so today is visible on first render. `requestAnimationFrame(() => { scroller.scrollLeft = scroller.scrollWidth; })` is the one-line fix.

10. **Sticky labels for rows × N days.** Without `position: sticky` the row name disappears as soon as you scroll horizontally. Always pair sticky-left labels with a sticky-left "corner" cell on the header row.

## When NOT to use a heatmap

- **You have ≤ 7 data points.** A bar chart or sparkline shows them better.
- **You care about absolute values, not patterns.** A table with sortable columns is more honest.
- **The metric is monotonically increasing.** Cumulative-progress charts (line, area) communicate growth; heatmap won't.
- **Most days are zeros.** Heatmaps are pattern-detectors; if 95% of cells are empty, the eye sees noise.
- **You want to compare two people / two metrics on equal footing.** Side-by-side bars or a scatter plot answers "are they correlated" better than two stacked heatmaps.

## Common gotchas

- **Plugin shows nothing** — likely the daily-note name isn't `YYYY-MM-DD`, or `entries` array is empty after filtering by year, or `renderHeatmapCalendar` isn't loaded yet (reload the note).
- **All cells same color** — `intensity` is the same value for every entry, or all entries fall outside `[intensityScaleStart, intensityScaleEnd]` and clamp to one bound.
- **Today's cell on the wrong day** — Obsidian's `dv.date("today")` uses local time; the plugin uses UTC for cross-year filter and local for the today-border. In timezones far from UTC the off-by-one shows around midnight; rarely matters in practice.
- **CSS snippet doesn't apply** — plugin sets `style="background-color: ..."` inline on `.hasData` cells; use `!important` to override, or target `.isEmpty` only (no inline style there).
- **Custom heatmap renders empty** — `dv.pages(...)` returned a Proxy, not an array. Call `.array()` before `.map()` on Dataview Proxies if you intend to use plain array methods.
- **Block flickers on scroll** — if your DataviewJS heatmap is far down a long note, Obsidian re-renders it lazily on scroll. Keep heavy work outside the render loop, or move large logic into a plugin / shared script and call a thin wrapper from the block.
