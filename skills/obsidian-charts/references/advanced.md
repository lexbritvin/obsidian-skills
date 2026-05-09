# Advanced — `advanced-chart`, image export, settings, full caveats

## 1. `advanced-chart` codeblock — full Chart.js escape hatch

The plugin registers a second processor:

```ts
this.registerMarkdownCodeBlockProcessor("advanced-chart", async (data, el) =>
  this.renderer.renderRaw(await JSON.parse(data), el)
);
```

The body is **raw JSON** (not YAML), parsed with `JSON.parse` and passed straight to Chart.js. Use this when you need:

- Annotations (`chartjs-plugin-annotation` is registered globally).
- Mixed types beyond per-series override.
- Static tooltip / scale config beyond the YAML path.
- Sankey with full controller options.

```advanced-chart
{
  "type": "line",
  "data": {
    "labels": ["A", "B", "C", "D"],
    "datasets": [{
      "label": "Demo",
      "data": [1, 3, 2, 5],
      "borderColor": "rgba(54, 162, 235, 1)",
      "backgroundColor": "rgba(54, 162, 235, 0.2)",
      "fill": true,
      "tension": 0.3
    }]
  },
  "options": {
    "scales": {
      "y": { "beginAtZero": true, "title": { "display": true, "text": "Score" } }
    },
    "plugins": {
      "legend": { "position": "bottom" }
    }
  }
}
```

### What you lose

- **Functions can't survive `JSON.parse`.** Tooltip callbacks, scriptable colors, dynamic labels — none work in `advanced-chart`. For function-valued config, use `dataviewjs` + `window.renderChart`.
- **No theme defaults injected.** Same as `window.renderChart` — the YAML codeblock path is the only one that maps `--text-muted` / `--background-modifier-border` automatically.
- **No animation override built in.** Chart.js default is animated; pass `"animation": { "duration": 0 }` to disable.

## 2. Annotation plugin (registered globally)

`chartjs-plugin-annotation@^2.2.1` is registered on plugin load. Use via `options.plugins.annotation`:

```advanced-chart
{
  "type": "line",
  "data": {
    "labels": ["Mon","Tue","Wed","Thu","Fri"],
    "datasets": [{ "label":"Mood","data":[5,6,5,6,7] }]
  },
  "options": {
    "plugins": {
      "annotation": {
        "annotations": {
          "goal": {
            "type": "line",
            "yMin": 6, "yMax": 6,
            "borderColor": "rgba(255,99,132,1)",
            "borderWidth": 1,
            "borderDash": [4,4],
            "label": { "display": true, "content": "Goal", "position": "end" }
          }
        }
      }
    }
  }
}
```

Annotation types: `line`, `box`, `point`, `polygon`, `ellipse`, `label`. See `chartjs-plugin-annotation` docs.

## 3. Mixed chart types (YAML path)

Per-series `type:` override works in plain YAML because each series entry is spread into the Chart.js dataset:

```chart
type: bar
labels: [Q1, Q2, Q3, Q4]
series:
  - title: Revenue
    data: [10, 14, 12, 18]
  - title: Trend
    data: [11, 13, 14, 16]
    type: line
    fill: false
    borderColor: "rgba(255,99,132,1)"
    borderWidth: 2
```

For a horizontal mixed chart, set `indexAxis: y` at the top level.

## 4. Save chart as image

Editor command **Create Image from Chart**.

1. Select the entire ` ```chart … ``` ` block (including fences).
2. Run the command.
3. The chart is rendered off-screen, exported via `canvas.toDataURL(format, quality)`, saved into the attachments folder as `Chart <date>.<ext>`, and the selection is replaced with a markdown image link.

Format and quality come from settings:

- **Image Format**: `image/png` (default), `image/jpeg`, `image/webp`.
- **Image Quality**: 0.01 – 1.00 (default `0.92`). Effective only for lossy formats.

## 5. Plugin settings — full panel

| Setting | Type | Default | Effect |
|---|---|---|---|
| **Show Button in Context Menu** | toggle | `true` | Adds an "Insert Chart" item to the editor right-click menu. |
| **Enable Theme Colors** | toggle | `false` | Renderer reads CSS vars `--chart-color-1`, `--chart-color-2`, … in order until first unset. |
| **Color #1..N** | color picker | 6 defaults (red/blue/yellow/teal/purple/orange) | Border palette. Inner fill is the same color at reduced alpha. Add/remove rows from the panel; minimum one. |
| **Image Format** | dropdown | `image/png` | For Create Image from Chart. |
| **Image Quality** | slider 0.01–1, step 0.01 | `0.92` | Lossy-format quality. |

### Theme colors via CSS

Add a snippet (`Settings → Appearance → CSS snippets`):

```css
:root {
  --chart-color-1: var(--text-accent);
  --chart-color-2: rgb(0, 200, 100);
  --chart-color-3: #ff00ff;
  --chart-color-4: orange;
}
```

Charts pick these up only when **Enable Theme Colors** is toggled on. The renderer reads in order until the first unset var — don't skip numbers.

`Chart.defaults.color` defaults to `var(--text-muted)`, grid color to `var(--background-modifier-border)`, font family to `var(--mermaid-font)`. Charts re-skin automatically on theme change.

## 6. Public API surface

Beyond the codeblock processors, the plugin exposes exactly one global:

```js
window.renderChart(data, el): Chart | null
```

There's no exported `app.plugins.plugins["obsidian-charts"].api`. There's no event bus and no hook registry. For metadata-driven reactivity, use the table+`id` flow.

The plugin **registers globally** on load (other code in the same Obsidian process sees these without registering them again):

- All Chart.js v3 registerables.
- `chartjs-plugin-annotation`.
- `SankeyController` + `Flow` from `chartjs-chart-sankey`.
- `chartjs-adapter-moment` (date axis adapter).

## 7. `bestFit` regression — read before trusting

The line-of-best-fit feature uses `labels` as the dependent variable:

```ts
// Pseudo-code from src/main.ts
out_y[i] = out_y[i] + labels[i];
```

Implications:

- Non-numeric labels produce `NaN`s — the line won't draw.
- Even with numeric labels, the regression treats labels as Y, which is unusual. Verify against an external regression before publishing.
- Use only when both axes are numeric and you understand the math; otherwise compute the trend yourself in DataviewJS and pass it as a normal series.

## 8. Full caveats list

### Codeblock parsing
- **YAML indentation is strict.** Author in source mode. Plugin pre-converts tabs to 4 spaces but the parser is still YAML-strict.
- **Live preview can mangle list-item indentation** in `series:` blocks during paste.

### Chart options
- **`width` is a CSS string on the parent container.** Bare numbers do nothing; use `"60%"`, `"400px"`.
- **`labelColors` semantics flip with type** — `true` for pie/doughnut/polarArea, `false` for bar/line/radar.
- **`time:` only formats labels as dates** if labels parse as dates. Use ISO `YYYY-MM-DD`.
- **`indexAxis` and `stacked` apply to bar/line only.** Ignored on pie/doughnut/radar/polarArea/scatter/bubble/sankey.
- **`spanGaps` is set on the chart**, not per-dataset. For per-series, pass `spanGaps: true` inside the series entry.

### Animation
- **Animations are off** on YAML-rendered charts (`animation: { duration: 0 }` is hardcoded).
- **`window.renderChart` and `advanced-chart` keep Chart.js default animations** — pass `options.animation` to override.

### Reactivity
- **Block-id-linked charts auto-reload** on metadata-cache changes for the source file and on `css-change`.
- **Inline-data charts re-render only on theme change.**
- **DataviewJS charts re-run the entire script on every Dataview refresh.**

### Print
- **Print width is forced to 500px** (`.print .block-language-chart` style) — workaround for a print-rendering bug.

### Mobile
- Plugin is desktop-non-only (`isDesktopOnly: false`).
- Helper modal and color picker are clunky on phones.

### YAML escape hatches
- **No `options:` passthrough on YAML blocks.** For Chart.js options the YAML path doesn't expose, switch to `advanced-chart` or `window.renderChart`.
- **Per-series properties are spread into the Chart.js dataset** (undocumented but useful) — `borderDash`, `pointRadius`, `yAxisID`, `type:` for mixed charts, etc.
- **Top-level `padding` and `textColor`** are read by the renderer but not documented.

### Data sources
- **No CSV import.** Sources: inline YAML, linked markdown table, or programmatic via DataviewJS.
- **`--chart-color-N` parsing stops at the first missing index** — don't skip numbers.

### Sankey specifics
- **Series uses `[from, flow, to]` triples** in YAML. The renderer rewrites them into `{from, flow, to}` Chart.js objects.
- **`colorFrom` / `colorTo` are object maps** in YAML and become functions inside Chart.js (the controller wants functions).

### Chart.js version
- **Pinned to 3.x (`^3.9.1`)**, not 4.x. Examples written for v4 may need adjustment:
  - `legend`, `title`, `tooltip` live under `options.plugins.*`.
  - Scales are objects keyed by axis id (`x`, `y`, `r`), not arrays.

## 9. Debug recipes

### "Chart shows nothing / shows the codeblock literally"
- Plugin not loaded or disabled — Settings → Community plugins → enable Charts.
- For DataviewJS path: `window.renderChart` is undefined — plugin loaded after this script ran. Refresh the note or add a guard:
  ```js
  if (typeof window.renderChart !== "function") {
    dv.paragraph("Charts plugin not loaded yet.");
    return;
  }
  ```

### "Missing type, labels or series"
- One of the three required fields is absent (YAML path). Or you set `id:` but the table reference is wrong (the renderer falls back to required-field check).

### "Table malformed"
- `markdown-tables-to-json` couldn't parse the linked table. Check for merged cells, alignment colons, leading/trailing spaces in headers, inconsistent column counts.

### Chart renders but data looks wrong
- Check `layout:` — column-oriented may swap labels and series vs row-oriented.
- Check `select:` — case- and whitespace-sensitive.
- DataviewJS: verify `pages.array().map(...)` (not raw `.map` on a DataArray which can return another DataArray Chart.js doesn't unwrap).

### Theme colors not applied
- "Enable Theme Colors" must be on in plugin settings.
- CSS vars must be set on `:root` or `body` — not on `.workspace-leaf`.
- Don't skip indices — `--chart-color-1`, `--chart-color-2`, `--chart-color-3`. Skipping `-2` stops parsing at `-1`.

### Animation is missing
- YAML codeblock path force-disables animation. Switch to `window.renderChart` or `advanced-chart` and pass `options.animation`.

### Annotation plugin not working
- Only works in `advanced-chart` and `window.renderChart`. The YAML path doesn't accept an `options:` block.
- In `advanced-chart`, the body must be valid JSON — no comments, double-quoted keys.
