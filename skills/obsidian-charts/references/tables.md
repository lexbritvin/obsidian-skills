# Table → chart

Two flows for turning a markdown table into a chart. Both rely on `markdown-tables-to-json` for parsing.

## 1. Replace selection with a chart codeblock (one-shot)

Select an **entire** markdown table in the editor and run one of:

- **Create Chart from Table (Column oriented Layout)**
- **Create Chart from Table (Row oriented Layout)**

The selection must contain at least 3 lines and at least 2 pipes. The selection is replaced with a generated `chart` codeblock you then edit by hand.

### Example

Source table:

```md
|       | Q1  | Q2  | Q3  | Q4  |
| ----- | --- | --- | --- | --- |
| Sales | 10  | 14  | 12  | 18  |
| Costs | 8   | 9   | 11  | 13  |
```

After running the **column-oriented** command (each row → series):

````chart
type: bar
labels: [Q1, Q2, Q3, Q4]
series:
  - title: Sales
    data: [10, 14, 12, 18]
  - title: Costs
    data: [8, 9, 11, 13]
width: 80%
beginAtZero: true
````

After **row-oriented** (each column → series), labels and series swap accordingly.

You then change `type:`, add axis options, etc.

## 2. Block-id linked table (live, recommended)

Tag the table with a block id and reference it from a `chart` codeblock. The chart re-renders on metadata-cache changes for the source file.

### 2.1 Tag the table

```md
|       | Test1 | Test2 | Test3 |
| ----- | ----- | ----- | ----- |
| Data1 | 1     | 2     | 3.33  |
| Data2 | 3     | 2     | 1     |
| Data3 | 6.7   | 4     | 2     |
^table
```

The `^table` line is Obsidian's standard block-id syntax — applies to the preceding block.

### 2.2 Reference the block from a chart

````chart
type: bar
id: table
layout: rows
width: 80%
beginAtZero: true
````

`labels` and `series` are **not** required when `id:` is set — they are pulled from the parsed table.

### 2.3 Cross-file reference

If the table lives in another note:

````chart
type: line
id: table
file: Daily Metrics
layout: columns
````

`file` accepts a basename (`Daily Metrics`) or a full path (`Tracking/Daily Metrics.md`).

### 2.4 Filter rows / columns

Pick a subset of row or column titles:

````chart
type: bar
id: table
layout: rows
select: [Data2, Data3]
````

When `layout: rows`, `select` filters by **row labels**. When `layout: columns`, by **column headers**.

## 3. Layout: rows vs columns

| `layout` | What becomes a series | What becomes labels |
|---|---|---|
| `columns` (default) | Each **column** of the table → one series. | Row headers (first column). |
| `rows` | Each **row** of the table → one series. | Column headers (first row). |

If your data is naturally "metrics × time", choose:

- Columns are time → `layout: columns`. Series = metrics, labels = time.
- Rows are time → `layout: rows`. Series = metrics, labels = time.

## 4. Reactivity

- A chart linked via `id:` re-renders on:
  - Metadata-cache changes for the source file (file save / inline-field edit).
  - Theme switch (`workspace.on("css-change")`).
- A chart with inline `labels:` / `series:` only re-renders on theme switch — it has no live data binding.

For "live" charts driven by frontmatter / inline fields without a table, write a `dataviewjs` block (see `dataviewjs.md`).

## 5. Pitfalls

- **The table must parse cleanly** with `markdown-tables-to-json`. Common breakage:
  - Merged cells.
  - Leading / trailing spaces in headers.
  - Alignment colons (`:---:`) sometimes confuse it.
  - Inconsistent column counts row-to-row.
  - If you see the error `"Table malformed"`, that's why.
- **Block ids are case-sensitive** — `id: table` references `^table`, not `^Table`.
- **`select` filters by display title**, not by index. Match exactly (case- and whitespace-sensitive).
- **`id:` overrides `labels:`/`series:`.** Don't write both — the inline arrays are silently ignored when `id` is set.
- **Generated codeblocks have hardcoded `width: 80%` and `beginAtZero: true`** — edit them out if you want different defaults. The defaults are baked into the conversion, not into the renderer.
- **Cross-file `file:` resolution** — if multiple notes share a basename, the conversion may pick the wrong one. Use a full path to disambiguate.
