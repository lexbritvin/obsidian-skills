# `chart` codeblock — full YAML reference

The everyday API. Renders inside ` ```chart ` fences. Strict YAML — author in source mode.

## 1. Minimum shape

````chart
type: bar
labels: [Mon, Tue, Wed, Thu, Fri]
series:
  - title: Series 1
    data: [1, 2, 3, 4, 5]
````

The renderer rejects the block with `"Missing type, labels or series"` if any of `type`, `labels`, or `series` is absent — **unless** `id:` is set, in which case the table-driven path is used and these can be omitted (see `tables.md`).

## 2. Supported chart types

Renderer accepts:

| `type` | Documented? | Notes |
|---|---|---|
| `bar` | yes | Default cartesian. |
| `line` | yes | Smoothing, fill, best-fit, time-axis. |
| `pie` | yes | `width:` recommended (otherwise expands). |
| `doughnut` | yes | Same. |
| `radar` | yes | Uses radial `r` axis. |
| `polarArea` | yes | Same. |
| `scatter` | undocumented but supported | `data:` is array of `{x, y}` objects. |
| `bubble` | undocumented but supported | `data:` is array of `{x, y, r}` objects. |
| `sankey` | docs only in old Docusaurus build | Series uses `[from, flow, to]` triples. |

The graphical creator modal exposes only bar/line/pie/doughnut/radar/polarArea. Scatter / bubble / sankey require hand-written codeblocks.

## 3. Series structure

```yaml
series:
  - title: Series A      # optional, becomes dataset.label
    data: [1, 2, 3]      # required (or {x,y} / {x,y,r} / [from,flow,to])
  - title: Series B
    data: [4, 5, 6]
    # any other key is spread into the Chart.js dataset:
    # borderColor, backgroundColor, borderDash, pointRadius,
    # yAxisID, fill, tension, borderWidth, hidden, stack, type,
    # colorFrom, colorTo, priority (sankey), ...
```

Behaviour:

- `title` → `dataset.label`. Omit for an unlabeled trace.
- Everything else on the series entry is **spread into the Chart.js dataset object**. Per-series Chart.js options work directly: `borderDash`, `pointRadius`, `yAxisID`, `type:` for mixed bar+line charts, etc.
- `borderColor` / `backgroundColor` filled from the palette per-series unless overridden.
- `borderWidth` defaults to `1`. Override per-series with `borderWidth: 2`.
- `tension` and `fill` apply to every series from the top-level options (line type).

## 4. Top-level options — every field

### 4.1 General / styling

| Field | Type | Default | Applies to | Notes |
|---|---|---:|---|---|
| `width` | CSS length string | `100%` | all | Sets parent element's `style.width`. Use `"60%"`, `"400px"`. Bare numbers do nothing. |
| `tension` | number 0..1 | `0` | line | Bezier smoothing. `0` = polyline, `1` = max. |
| `fill` | boolean | `false` | line | Fill area under the trace. With `stacked: true`, dataset 0 fills to origin and others fill to previous (`fill: '-1'`). |
| `labelColors` | boolean | `false` | typically pie/doughnut/polarArea | When `true`, every datapoint within a series gets a different color (one-color-per-label). When `false`, one color per series. |
| `transparency` | number 0..1 | `0.25` | all | Inner fill alpha. |
| `legend` | boolean | `true` | all | Show / hide the legend. |
| `legendPosition` | `top` \| `bottom` \| `left` \| `right` | `top` | all | Legend position. |
| `padding` | number \| object | unset | all | Undocumented. Passes to `Chart.defaults.layout.padding`. Accepts a Chart.js padding object. |
| `textColor` | CSS color | `var(--text-muted)` | all | Undocumented. Overrides `Chart.defaults.color` for axis labels and legend text. |
| `time` | string | unset | bar/line | Treat the X axis as time. Value is the Chart.js time `unit`: `millisecond`, `second`, `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`. Labels become dates parsed by the bundled moment adapter. |
| `spanGaps` | boolean | `false` | line (also passes through to default branch) | Connect across `null` data points. |

### 4.2 Bar/line-only

| Field | Type | Default | Notes |
|---|---|---:|---|
| `indexAxis` | `x` \| `y` | `x` | Set to `y` for horizontal bars / horizontal lines. |
| `stacked` | boolean | `false` | Stacks both X and Y scales. |

### 4.3 Best fit (line only)

| Field | Type | Default | Notes |
|---|---|---:|---|
| `bestFit` | boolean | `false` | Adds an extra series that is the linear regression of one of the existing series. |
| `bestFitTitle` | string | `Line of Best Fit` | Legend label for the regression line. |
| `bestFitNumber` | integer | `0` | Index of the source series. |

> **Beware:** the regression uses `labels` as the dependent variable. Non-numeric labels produce `NaN`s. Use only when labels are numbers.

### 4.4 Cartesian axes (bar, line) — `x` / `y` prefix

Replace `x` / `y` with the axis name. All optional.

| Field | Type | Default | Notes |
|---|---|---:|---|
| `xMin` / `yMin` | number | unset | Axis minimum. `Min` overrides `beginAtZero`. |
| `xMax` / `yMax` | number | unset | Axis maximum. |
| `xReverse` / `yReverse` | boolean | `false` | Reverse the axis. |
| `xTitle` / `yTitle` | string | unset | Axis title text **and** display flag. Set to a non-empty string to show. |
| `xDisplay` / `yDisplay` | boolean | `true` | Whether the axis is rendered. |
| `xTickDisplay` / `yTickDisplay` | boolean | `true` | Whether axis ticks are rendered. |
| `xTickPadding` / `yTickPadding` | number | unset | Pixel padding for ticks. Undocumented but read by the renderer. |
| `beginAtZero` | boolean | `false` | Forces Y axis (and `r` axis on radar/polarArea) to start at 0. |

### 4.5 Radial axis (radar, polarArea) — `r` prefix

| Field | Type | Default | Notes |
|---|---|---:|---|
| `rMin` | number | unset | Inner radius value. |
| `rMax` | number | unset | Outer radius value (use this for radar/polarArea, not `yMax`). |
| `beginAtZero` | boolean | `false` | Same as cartesian. |

### 4.6 Table-driven mode (alternative to `labels`/`series`)

| Field | Type | Default | Notes |
|---|---|---:|---|
| `id` | string | — | Block id (`^name`) of a table elsewhere in the vault. When set, `labels` / `series` are not required and are ignored. |
| `file` | string | current note | Filename (basename or path) where the table lives. |
| `layout` | `rows` \| `columns` | `columns` | Whether each row or each column becomes a series. |
| `select` | array of strings | unset | Subset of row/column titles to include. |

See `tables.md` for the full table-linked workflow.

## 5. Color customization

Three layers, in priority order:

1. **CSS variables** (when "Enable Theme Colors" is on, or `themeColors: true` is passed at render time):

   ```css
   :root {
     --chart-color-1: #ff00ff;
     --chart-color-2: rgb(0, 200, 100);
     --chart-color-3: var(--text-accent);
   }
   ```

   The renderer reads `--chart-color-1`, `--chart-color-2`, … in order until the first unset one. Don't skip numbers.
2. **Plugin settings palette** (default). Six colors out of the box; add/remove rows in the panel.
3. **Per-series override**: pass `borderColor` / `backgroundColor` directly on the series entry.

`Chart.defaults.color` defaults to `var(--text-muted)`, grid color to `var(--background-modifier-border)`, font family to `var(--mermaid-font)`. Charts re-skin automatically on theme change.

## 6. Time-axis

Set `time: <unit>` on a bar/line chart. The plugin bundles `chartjs-adapter-moment`, so labels can be ISO strings, `YYYY-MM-DD`, or any moment-parseable date.

```chart
type: line
labels: ["2026-01-01", "2026-02-01", "2026-03-01", "2026-04-01"]
series:
  - title: Mood
    data: [4, 5, 6, 5]
time: month
yMin: 1
yMax: 7
```

Units: `millisecond`, `second`, `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`.

## 7. Mixed chart types

Per-series `type:` override works because the entry is spread into the Chart.js dataset:

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
```

## 8. Per-type examples

### 8.1 Bar

```chart
type: bar
labels: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
series:
  - title: Steps
    data: [8200, 9100, 7400, 10200, 6800, 4200, 3100]
  - title: Goal
    data: [8000, 8000, 8000, 8000, 8000, 8000, 8000]
beginAtZero: true
yTitle: Steps
```

### 8.2 Line — smoothing, fill, time axis

```chart
type: line
labels: ["2026-01-01", "2026-02-01", "2026-03-01", "2026-04-01", "2026-05-01"]
series:
  - title: Mood
    data: [5, 6, 5, 6, 7]
  - title: Energy
    data: [4, 5, 6, 6, 7]
time: month
tension: 0.4
fill: true
yMin: 1
yMax: 7
yTitle: Score
```

### 8.3 Pie

```chart
type: pie
labels: [Sleep, Work, Sport, Reading, Other]
series:
  - title: Hours
    data: [8, 8, 1, 2, 5]
width: 40%
labelColors: true
```

### 8.4 Doughnut

```chart
type: doughnut
labels: [Done, In progress, Backlog, Blocked]
series:
  - title: Tasks
    data: [12, 5, 18, 2]
width: 40%
labelColors: true
```

### 8.5 Radar

```chart
type: radar
labels: [Fitness, Sleep, Mood, Focus, Social, Learning]
series:
  - title: This week
    data: [4, 6, 5, 6, 4, 7]
  - title: Last week
    data: [3, 5, 4, 5, 5, 6]
width: 50%
rMin: 0
rMax: 7
```

### 8.6 Polar Area

```chart
type: polarArea
labels: [Mon, Tue, Wed, Thu, Fri]
series:
  - title: Pomodoros
    data: [4, 6, 3, 5, 7]
width: 40%
labelColors: true
```

### 8.7 Scatter

```chart
type: scatter
labels: []
series:
  - title: Sample A
    data:
      - { x: 1, y: 2 }
      - { x: 2, y: 4 }
      - { x: 3, y: 3 }
      - { x: 5, y: 7 }
  - title: Sample B
    data:
      - { x: 1.5, y: 1 }
      - { x: 2.5, y: 5 }
      - { x: 4, y: 6 }
width: 60%
```

### 8.8 Bubble

```chart
type: bubble
labels: []
series:
  - title: Risk vs reward
    data:
      - { x: 10, y: 20, r: 5 }
      - { x: 15, y: 10, r: 12 }
      - { x: 20, y: 30, r: 8 }
width: 60%
```

### 8.9 Sankey

```chart
type: sankey
labels: [Oil, "Natural Gas", Coal, "Fossil Fuels", Electricity, Energy]
series:
  - data:
      - [Oil, 15, "Fossil Fuels"]
      - ["Natural Gas", 20, "Fossil Fuels"]
      - [Coal, 25, "Fossil Fuels"]
      - [Coal, 25, Electricity]
      - ["Fossil Fuels", 60, Energy]
      - [Electricity, 25, Energy]
    priority:
      Oil: 1
      "Natural Gas": 2
      Coal: 3
      "Fossil Fuels": 1
      Electricity: 2
      Energy: 1
    colorFrom:
      Oil: black
      Coal: gray
      "Fossil Fuels": slategray
      Electricity: blue
      Energy: orange
    colorTo:
      Oil: black
      Coal: gray
      "Fossil Fuels": slategray
      Electricity: blue
      Energy: orange
```

The renderer rewrites `[from, flow, to]` triples into `{from, flow, to}` Chart.js objects, and rewires `colorFrom`/`colorTo` from object maps into the lookup functions the controller wants.

## 9. Exhaustive YAML field list — quick reference

```yaml
# Required (unless `id` is set)
type: bar | line | pie | doughnut | radar | polarArea | scatter | bubble | sankey
labels: [..., ...]
series:
  - title: <string?>          # optional, becomes dataset.label
    data: [...] | [{x,y}] | [{x,y,r}] | [[from,flow,to]]
    # any other key is spread into the Chart.js dataset:
    # type, borderColor, backgroundColor, borderDash, pointRadius,
    # yAxisID, fill, tension, borderWidth, hidden, stack,
    # colorFrom, colorTo, priority (sankey), ...

# Identity / table-driven (alternative to labels+series)
id: <block-id>
file: <basename or path>
layout: rows | columns        # default: columns
select: [<row-or-column-title>, ...]

# General
width: <CSS length>           # default: 100%
labelColors: <bool>           # default: false
transparency: <0..1>          # default: 0.25
legend: <bool>                # default: true
legendPosition: top | bottom | left | right   # default: top
padding: <number | object>    # undocumented; passes to Chart.defaults.layout.padding
textColor: <CSS color>        # undocumented; default --text-muted
time: millisecond | second | minute | hour | day | week | month | quarter | year
spanGaps: <bool>              # default: false

# Line-only
tension: <0..1>               # default: 0
fill: <bool>                  # default: false
bestFit: <bool>               # default: false
bestFitTitle: <string>        # default: "Line of Best Fit"
bestFitNumber: <int>          # default: 0

# Bar/line-only
indexAxis: x | y              # default: x
stacked: <bool>               # default: false

# Cartesian axes (bar, line) — replace `x`/`y` as needed
xMin: <number>
xMax: <number>
yMin: <number>
yMax: <number>
xReverse: <bool>              # default: false
yReverse: <bool>              # default: false
xTitle: <string>              # default: unset (also controls display)
yTitle: <string>              # default: unset
xDisplay: <bool>              # default: true
yDisplay: <bool>              # default: true
xTickDisplay: <bool>          # default: true
yTickDisplay: <bool>          # default: true
xTickPadding: <number>        # undocumented
yTickPadding: <number>        # undocumented
beginAtZero: <bool>           # default: false (overridden by Min)

# Radial axis (radar, polarArea)
rMin: <number>
rMax: <number>
```
