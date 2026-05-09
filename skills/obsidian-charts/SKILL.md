---
name: obsidian-charts
description: Authoring charts in Obsidian via the Charts plugin (`obsidian-charts` by phibr0) — covers the ` ```chart ` YAML codeblock (all types: bar, line, pie, doughnut, radar, polarArea, scatter, bubble, sankey), the ` ```advanced-chart ` JSON escape hatch, table-to-chart conversion (block-id linking via `id`/`file`/`layout`/`select`), and the `window.renderChart(data, container)` integration with DataviewJS (raw Chart.js v3 config). Use this skill whenever the user asks to draw a chart, plot, graph, time series, bar chart, pie/doughnut, radar/spider, scatter, bubble, or sankey diagram inside an Obsidian note, mentions ` ```chart `, ` ```advanced-chart `, `window.renderChart`, the Charts plugin, Chart.js inside Obsidian, asks to visualize a markdown table or daily-note metric, or wants to render any quantitative data over notes (mood/energy series, tag counts, books-by-rating, tasks-by-status). Trigger even when "Charts plugin" is not named — any request to visualise vault data inline goes here. Pair with the `obsidian-dataview` skill when data comes from `dv.pages()` and the chart is rendered from DataviewJS.
---

# Obsidian Charts

Reference for the **Charts** plugin (`obsidian-charts` by phibr0). Wraps **Chart.js v3** with an Obsidian-friendly YAML codeblock, a DataviewJS integration, and a table-to-chart command.

## What the plugin is

A community plugin that registers two markdown codeblock processors and one global function:

- ` ```chart ` — YAML config (everyday API).
- ` ```advanced-chart ` — raw JSON Chart.js config (escape hatch).
- `window.renderChart(data, container)` — programmatic rendering (use from `dataviewjs`).

Plus four editor commands:

- **Insert new Chart** — opens a graphical creator modal.
- **Create Chart from Table (Column oriented Layout)** — replaces a selected markdown table with a `chart` codeblock.
- **Create Chart from Table (Row oriented Layout)** — same, transposed.
- **Create Image from Chart** — exports a rendered chart as PNG/JPEG/WebP.

There is **no inline `` `chart: …` ``** form. Charts only render inside fenced blocks or via `window.renderChart`.

Chart.js version is pinned to **v3.9.1**. Most options follow Chart.js v3 syntax (notably `legend` lives under `options.plugins.legend`, not at root).

## Decision tree — which surface to use

| Need | Pick |
|---|---|
| Static chart from hard-coded data | ` ```chart ` (YAML). Simplest. |
| Chart that mirrors a table elsewhere in the vault | ` ```chart ` with `id:` / `file:` / `layout:` / `select:` (block-id linked, auto-refreshes). |
| Chart from `dv.pages()` / inline-fields / metadata | ` ```dataviewjs ` + `window.renderChart(rawConfig, this.container)`. |
| Chart.js features YAML doesn't expose (annotations, mixed types beyond per-series override, custom tooltip callbacks) | ` ```advanced-chart ` (JSON) — or the DataviewJS path. |
| One-off animation, scatter, bubble, sankey | ` ```chart ` works for all of these (sankey/scatter/bubble undocumented but supported), or use DataviewJS. |
| Save the rendered chart as an image | Select the ` ```chart ` block → run **Create Image from Chart**. |

## YAML quickstart — `chart` block

```chart
type: bar
labels: [Mon, Tue, Wed, Thu, Fri]
series:
  - title: Steps
    data: [8200, 9100, 7400, 10200, 6800]
beginAtZero: true
yTitle: Steps
```

Required: `type`, `labels`, `series` (each entry needs `data`; `title` is optional).

Full field list, every chart type's example, all color/legend/axis/time/mixed-type options → **`references/codeblock.md`**.

## DataviewJS quickstart — `window.renderChart`

```dataviewjs
const pages = dv.pages('"Daily"')
  .where(p => typeof p.mood === "number")
  .sort(p => p.file.day, "asc")
  .limit(30);

window.renderChart({
  type: "line",
  data: {
    labels: pages.array().map(p => p.file.name),
    datasets: [{
      label: "Mood",
      data: pages.array().map(p => p.mood),
      borderColor: "rgba(54, 162, 235, 1)",
      backgroundColor: "rgba(54, 162, 235, 0.2)",
      fill: true,
      tension: 0.3
    }]
  },
  options: { scales: { y: { min: 1, max: 7 } } }
}, this.container);
```

Pass any **raw Chart.js v3 ChartConfiguration**. The function returns the `Chart` instance.

Theme integration, async/sync semantics, common patterns (time series, tag pies, inline-field bars), Chart.js shape reference → **`references/dataviewjs.md`**.

## Tables → charts

Two flows:

1. **One-shot**: select a markdown table, run **Create Chart from Table (Column/Row oriented Layout)**. The selection is replaced with a generated `chart` codeblock you can edit by hand.
2. **Live link**: tag a table with `^block-id`, reference it from a `chart` block via `id: block-id` (plus optional `file:`, `layout: rows|columns`, `select: [...]`). The chart re-renders on metadata-cache changes for the source file.

Detailed flow, parsing rules, multi-file references → **`references/tables.md`**.

## Advanced — `advanced-chart`, image export, settings

` ```advanced-chart ` accepts a raw JSON Chart.js config — the full v3 surface, including `chartjs-plugin-annotation` and `chartjs-chart-sankey` (registered globally on plugin load).

```advanced-chart
{ "type": "line",
  "data": { "labels": ["A","B","C"], "datasets": [{"label":"x","data":[1,3,2]}] },
  "options": { "plugins": { "legend": { "position": "bottom" } } }
}
```

Settings panel options (theme colors via `--chart-color-N`, palette overrides, image format/quality), the **Create Image from Chart** flow, mixed chart types via per-series `type:` override, undocumented options (`textColor`, `padding`, per-series spread into Chart.js dataset), and the full pitfalls list → **`references/advanced.md`**.

## Common pitfalls

- **Animations are off** on YAML-rendered charts (`animation: { duration: 0 }` is hard-coded). To get animation, render via `window.renderChart` and pass your own `options.animation`.
- **YAML indentation is strict.** Author in source mode — Obsidian's live preview can mangle list-item indentation in `series:` blocks. The plugin pre-converts tabs to 4 spaces but the parser is still YAML-strict.
- **`bestFit` treats `labels` as numeric** — non-numeric labels produce `NaN`. Use only when labels are numbers.
- **`labelColors` semantics flip with chart type.** For pie/doughnut/polarArea you usually want `true` (one colour per slice). For bar/line/radar usually `false` (one colour per series). Default is `false`.
- **`width` is a CSS string on the parent container**, not a number. Bare numbers are silently ignored. Use `"60%"`, `"400px"`, etc.
- **`time:` only formats labels as dates** if the labels parse as dates. Use ISO `YYYY-MM-DD` (or any moment-parseable string — the plugin bundles `chartjs-adapter-moment`).
- **No `options:` passthrough on YAML blocks.** The YAML path does not spread an `options` block into Chart.js. For annotations, dual axes, custom callbacks → switch to `advanced-chart` or `window.renderChart`.
- **Per-series properties are spread into the Chart.js dataset** (undocumented but useful) — `borderDash`, `pointRadius`, `yAxisID`, even `type:` for mixed bar+line charts. Anything you'd put on a Chart.js dataset works on a series entry.
- **Chart.js v3 syntax**, not v4. Examples written for v4 may need to be down-leveled (`legend` is at `plugins.legend`, scales are `x`/`y` objects, not arrays).
- **Inline `` `chart: …` ``** — does not exist. Use a fenced block or `window.renderChart`.
- **`window.renderChart` skips the YAML-path theme defaults.** If you want theme-aware text/grid colors via JS, set them yourself from `--text-muted` / `--background-modifier-border` (see `references/dataviewjs.md`).
- **Sankey series uses `[from, flow, to]` triples** — the renderer rewrites them into `{from, flow, to}` Chart.js objects. `colorFrom` / `colorTo` are object maps in YAML and become functions inside Chart.js.
- **Block-id-linked charts re-render on metadata changes**, but inline-data charts only re-render on theme change. To make an inline chart reactive to a metric, use the table+`id` flow or a `dataviewjs` block.
- **Print width is forced to 500px** by the plugin's CSS — a workaround for a print-rendering bug.

## Pairing with other skills

Pair this skill with the **`obsidian-dataview`** skill whenever data comes from `dv.pages()` — that one covers the query/data side; this one covers the rendering.

For recurring dashboards, push data-loading into a custom toolkit plugin (if the vault has one) and keep the `dataviewjs` block thin: load → shape → `window.renderChart`.

## How to use this skill

1. Identify the surface: YAML `chart`, JSON `advanced-chart`, `dataviewjs` + `window.renderChart`, or table-driven via `id:`.
2. For YAML: start with the minimal shape (`type`, `labels`, `series`), then layer options. Consult `references/codeblock.md` for the per-type examples and full option list.
3. For DataviewJS: build a raw Chart.js v3 config (`{ type, data: { labels, datasets }, options }`) and pass it to `window.renderChart(config, this.container)`. Use `references/dataviewjs.md` patterns.
4. For "I have a table already": prefer the `id:` linked-table flow over the one-shot conversion if the data still changes.
5. When something doesn't render or looks wrong, walk the "Common pitfalls" list before spelunking — most issues are listed there.
6. Keep `dataviewjs` blocks ≤ ~70 lines (Obsidian lazy-renders past that). Extract data-loading into helpers.

Source of truth: <https://charts.phib.ro/Meta/Charts/Charts+Documentation>. Code: <https://github.com/phibr0/obsidian-charts>.
