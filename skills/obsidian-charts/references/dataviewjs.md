# `window.renderChart` — DataviewJS integration

The plugin exposes one global at load time:

```js
window.renderChart = renderer.renderRaw;
```

Signature:

```ts
window.renderChart(data: any, el: HTMLElement): Chart | null
```

Returns the Chart.js `Chart` instance (or `null` on render error — an inline error pane is drawn). Synchronous at the call site; the chart starts animating immediately.

## 1. Accepted data shapes

The function does a duck-type check on `data.chartOptions`:

- **If `data.chartOptions` is present** — treats `data` as the plugin's internal shape (output of `datasetPrep`). Sets the parent width and renders.
- **Otherwise** — passes `data` straight to `new Chart(ctx, data)`. So **anything Chart.js accepts as a config object** works.

In practice you always use shape #2 — a raw Chart.js v3 `ChartConfiguration`:

```js
const config = {
  type: "bar",
  data: {
    labels: [...],
    datasets: [{
      label: "...",
      data: [...],
      borderColor: "...",
      backgroundColor: "...",
      borderWidth: 1
    }]
  },
  options: { /* full Chart.js v3 options */ }
};
window.renderChart(config, this.container);
```

## 2. Element argument

In a `dataviewjs` block, `this.container` is the live container; pass it directly. The function appends a `<canvas>` to it (`destination = el.createEl("canvas")`).

```js
window.renderChart(config, this.container);
// or equivalently:
window.renderChart(config, dv.container);
```

If you want the chart inside a custom wrapper (e.g. with a header above), create the wrapper first and pass it:

```js
dv.header(3, "Mood — last 30 days");
const wrapper = dv.container.createEl("div", { cls: "my-chart" });
window.renderChart(config, wrapper);
```

## 3. Theme awareness

`window.renderChart` does **not** apply the Obsidian-theme defaults that the YAML codeblock path does. If you want theme-aware text/grid colors via JS, set them yourself:

```js
const muted = getComputedStyle(document.body).getPropertyValue("--text-muted").trim();
const grid  = getComputedStyle(document.body).getPropertyValue("--background-modifier-border").trim();

config.options = {
  ...config.options,
  scales: {
    x: { grid: { color: grid }, ticks: { color: muted } },
    y: { grid: { color: grid }, ticks: { color: muted } },
  },
  plugins: { legend: { labels: { color: muted } } },
};
```

## 4. Animation control

The YAML codeblock path force-disables animations. `window.renderChart` does **not** — animations follow whatever `options.animation` you pass (Chart.js default is enabled).

```js
config.options = { ...config.options, animation: { duration: 0 } };  // disable
config.options = { ...config.options, animation: { duration: 800 } }; // 0.8s
```

## 5. Reactivity

A single `dataviewjs` block re-runs entirely on Dataview's metadata-change refresh (debounced by Dataview's "Refresh Interval" setting). Each refresh creates a new `<canvas>` and a new chart — there's no stable identity to call `chart.update()` on across refreshes inside the dataviewjs path.

For "live" charts that update on file changes without losing identity, use the table+`id` flow (see `tables.md`).

## 6. Common patterns

### 6.1 Time series of a daily-note field

```dataviewjs
const pages = dv.pages('"Daily"')
  .where(p => p.mood != null)
  .sort(p => p.file.day, "asc")
  .limit(30);

const labels = pages.array().map(p => p.file.name);
const moods  = pages.array().map(p => p.mood);

window.renderChart({
  type: "line",
  data: {
    labels,
    datasets: [{
      label: "Mood",
      data: moods,
      borderColor: "rgba(54, 162, 235, 1)",
      backgroundColor: "rgba(54, 162, 235, 0.2)",
      borderWidth: 1,
      tension: 0.3,
      fill: true,
    }]
  },
  options: {
    scales: { y: { min: 1, max: 7, title: { display: true, text: "Score" } } }
  }
}, this.container);
```

### 6.2 Pie of tag counts

```dataviewjs
const counts = {};
for (const p of dv.pages()) {
  for (const tag of p.file.etags) counts[tag] = (counts[tag] ?? 0) + 1;
}
const top = Object.entries(counts)
  .sort((a, b) => b[1] - a[1])
  .slice(0, 8);

window.renderChart({
  type: "doughnut",
  data: {
    labels: top.map(([t]) => t),
    datasets: [{
      label: "Notes",
      data: top.map(([, n]) => n),
      borderWidth: 1,
    }]
  }
}, this.container);
```

### 6.3 Horizontal bar of inline-field totals

```dataviewjs
const pages = dv.pages("#book").where(p => typeof p.pages === "number");
const labels  = pages.array().map(p => p.file.name);
const lengths = pages.array().map(p => p.pages);

window.renderChart({
  type: "bar",
  data: {
    labels,
    datasets: [{
      label: "Pages",
      data: lengths,
      backgroundColor: "rgba(75, 192, 192, 0.2)",
      borderColor:     "rgba(75, 192, 192, 1)",
      borderWidth: 1,
    }]
  },
  options: {
    indexAxis: "y",
    scales: { x: { beginAtZero: true } }
  }
}, this.container);
```

### 6.4 Multi-series stacked bar from groupBy

```dataviewjs
const pages = dv.pages("#task").where(p => p.status && p.project);

const projects = pages.groupBy(p => p.project);
const statuses = ["open", "in-progress", "done", "blocked"];

const labels = projects.array().map(g => g.key);
const datasets = statuses.map((s, i) => ({
  label: s,
  data: projects.array().map(g => g.rows.where(p => p.status === s).length),
  borderWidth: 1,
}));

window.renderChart({
  type: "bar",
  data: { labels, datasets },
  options: {
    scales: {
      x: { stacked: true },
      y: { stacked: true, beginAtZero: true }
    }
  }
}, this.container);
```

### 6.5 Radar of self-tracked metrics

```dataviewjs
const today = dv.pages('"Daily"')
  .where(p => p.file.day && p.file.day.toISODate() === dv.luxon.DateTime.now().toISODate())
  .first();
if (!today) { dv.paragraph("No daily note today."); return; }

const axes = ["mood", "energy", "focus", "social", "fitness", "sleep"];

window.renderChart({
  type: "radar",
  data: {
    labels: axes.map(a => a[0].toUpperCase() + a.slice(1)),
    datasets: [{
      label: "Today",
      data: axes.map(a => today[a] ?? 0),
      borderWidth: 1,
    }]
  },
  options: {
    scales: { r: { min: 0, max: 7 } }
  }
}, this.container);
```

### 6.6 Build the YAML, let the codeblock processor render it

If you want the YAML-path defaults (Obsidian-theme colors, no animations, the palette), build a string and emit it as a fenced block via `dv.paragraph`:

````dataviewjs
const data = dv.current();
dv.paragraph(`\`\`\`chart
type: bar
labels: [${data.test}]
series:
  - title: Grades
    data: [${data.mark}]
\`\`\``);
````

Use this only when the YAML path is enough — it loses the per-call control of `window.renderChart`.

### 6.7 Mixed bar + line

```dataviewjs
const labels = ["Q1", "Q2", "Q3", "Q4"];

window.renderChart({
  type: "bar",
  data: {
    labels,
    datasets: [
      { label: "Revenue", data: [10, 14, 12, 18], borderWidth: 1 },
      { label: "Trend",   data: [11, 13, 14, 16], type: "line", fill: false, borderColor: "red", borderWidth: 2 },
    ]
  }
}, this.container);
```

### 6.8 Scatter from two-dimensional data

```dataviewjs
const pages = dv.pages("#book").where(p => typeof p.rating === "number" && typeof p.pages === "number");

window.renderChart({
  type: "scatter",
  data: {
    datasets: [{
      label: "Rating vs length",
      data: pages.array().map(p => ({ x: p.pages, y: p.rating })),
      borderWidth: 1,
    }]
  },
  options: {
    scales: {
      x: { title: { display: true, text: "Pages" } },
      y: { title: { display: true, text: "Rating" }, min: 1, max: 7 }
    }
  }
}, this.container);
```

## 7. Chart.js v3 cheatsheet — the parts that bite

The plugin is pinned to **Chart.js 3.9.1**. Key shape:

```js
{
  type: "bar" | "line" | "pie" | "doughnut" | "radar" | "polarArea" | "scatter" | "bubble" | "sankey",
  data: {
    labels: [...],
    datasets: [
      {
        label: string,
        data: number[] | { x: number, y: number }[] | { x, y, r }[],
        borderColor?: string | string[],
        backgroundColor?: string | string[],
        borderWidth?: number,
        borderDash?: number[],
        fill?: boolean | "origin" | "end" | "-1" | number,
        tension?: number,
        pointRadius?: number,
        pointHoverRadius?: number,
        yAxisID?: string,
        type?: string,            // for mixed charts
        stack?: string,           // for grouped stacks
        hidden?: boolean,
      }
    ]
  },
  options: {
    indexAxis: "x" | "y",
    responsive: true,
    maintainAspectRatio: true,
    aspectRatio: 2,
    animation: { duration: number, ... },
    scales: {
      x: { type, min, max, beginAtZero, stacked, reverse, title: {display,text}, ticks, grid, time: {unit} },
      y: { ... },
      r: { ... }   // radar / polarArea
    },
    plugins: {
      legend:  { display: true, position: "top"|"bottom"|"left"|"right", labels: {color, font} },
      title:   { display: true, text: "..." },
      tooltip: { enabled: true, mode: "index"|"point"|... },
      annotation: { /* via chartjs-plugin-annotation, registered globally */ }
    }
  }
}
```

Notes that trip people:

- In v3, `legend`, `title`, `tooltip` live under `options.plugins.*`, not at root.
- Scales are objects keyed by axis id (`x`, `y`, `r`), not arrays of `{ id }`.
- For dates, use `options.scales.x = { type: "time", time: { unit: "day" } }`. The plugin already registers the moment adapter.
- `chartjs-plugin-annotation` and `chartjs-chart-sankey` are **registered globally** by the plugin on load. You can reference them via `options.plugins.annotation` and `type: "sankey"` without registering yourself.

## 8. Pitfalls

- **`window.renderChart` is global** — only available after the plugin has loaded. If the user opens a note before plugins are ready, the function may be undefined for a moment. Defensive guard: `if (typeof window.renderChart !== "function") { dv.paragraph("Charts plugin not loaded yet"); return; }`.
- **The function appends to the element** — don't pre-fill it with content unless you wrap. Each Dataview refresh empties and re-renders the container, so there's no stale-canvas risk inside `dataviewjs`.
- **Theme defaults are not applied via this path.** Read `--text-muted` / `--background-modifier-border` yourself if you want the chart to match the theme.
- **No options spread on YAML path** — if you tried YAML and hit a wall, this is your fallback.
- **`dv.pages(...).map(...)` returns a DataArray, not a plain array.** Call `.array()` before passing to Chart.js, or rely on `.values` (DataArray exposes both).
- **Tasks-plugin emojis are not auto-promoted** to fields by Dataview. If you're charting `due` dates from `📅 …` task lines, that's surfaced; for `🔁` recurrence / priority emojis, write `[priority:: high]` and chart `priority`.
- **Chart.js v3 vs v4 syntax** — examples on Chart.js's site default to v4 these days. If a config doesn't render, check whether the option moved out of `options.plugins`.
- **DataviewJS re-runs on every refresh** — heavy data prep belongs in a custom helper plugin (or a memoization layer), not inline.
