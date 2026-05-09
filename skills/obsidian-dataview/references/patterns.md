# Patterns, integration, performance

Cookbook of common shapes, plugin-developer integration, performance tuning, and the full caveats list. Use this when the user asks "how do I…?", "why is my query empty?", "can another plugin call Dataview?", or "is this going to be slow?".

## 1. Cookbook

### 1.1 Recently modified pages

```dataview
LIST
WHERE file.mtime >= date(today) - dur(7 days)
SORT file.mtime DESC
```

### 1.2 All open tasks across the vault

```dataview
TASK WHERE !completed
```

DataviewJS, ungrouped:

```dataviewjs
const open = dv.pages().file.tasks.where(t => !t.completed);
dv.taskList(open, false);
```

### 1.3 Open tasks grouped by source file

```dataview
TASK WHERE !completed GROUP BY file.link
```

### 1.4 Overdue tasks

```dataview
TASK WHERE !completed AND due < date(today) AND typeof(due) = "date"
SORT due ASC
```

### 1.5 Top-N by metric

```dataview
TABLE rating, author FROM #book SORT rating DESC LIMIT 10
```

### 1.6 Group with per-group table (DataviewJS)

```dataviewjs
for (const g of dv.pages('"Books"').groupBy(p => p.genre)) {
  dv.header(3, g.key);
  dv.table(
    ["Title", "Rating"],
    g.rows.sort(p => p.rating, "desc").map(p => [p.file.link, p.rating])
  );
}
```

### 1.7 Tag breakdown with counts

```dataview
TABLE length(rows) AS Count
FROM ""
GROUP BY file.etags AS Tag
SORT Count DESC
```

### 1.8 Authors expanded — one row each

```dataview
TABLE authors FROM #LiteratureNote FLATTEN authors
```

### 1.9 Daily-note metric series

```dataview
TABLE WITHOUT ID file.link AS Day, mood, energy
FROM "Daily"
WHERE typeof(mood) = "number"
SORT file.day DESC
LIMIT 14
```

### 1.10 Aggregate metric over a range (DataviewJS)

```dataviewjs
const start = dv.date("2026-05-01");
const end   = dv.date("2026-05-08");
const days  = dv.pages('"Daily"')
  .where(p => p.file.day >= start && p.file.day <= end);

const moods = days.array().map(p => p.mood).filter(v => typeof v === "number");
const mean  = moods.length ? moods.reduce((a, b) => a + b, 0) / moods.length : null;
dv.paragraph(`Avg mood: ${mean?.toFixed(2) ?? "—"} (n=${moods.length})`);
```

### 1.11 Linked-mention "back-links to this note"

```dataview
LIST FROM [[]]
WHERE !contains(file.outlinks, this.file.link)
```
"Pages that link into this file but this file does not link back to."

### 1.12 Calendar of deadlines

```dataview
CALENDAR due WHERE typeof(due) = "date"
```

### 1.13 Recursive task expansion

```dataviewjs
const all = dv.current().file.tasks.expand("children");
dv.taskList(all.where(t => !t.completed), false);
```

### 1.14 DFS over the link graph

```dataviewjs
const start = dv.current().file.path;
const visited = new Set();
const stack = [start];

while (stack.length) {
  const path = stack.pop();
  const meta = dv.page(path);
  if (!meta) continue;

  for (const link of meta.file.inlinks.concat(meta.file.outlinks).array()) {
    if (visited.has(link.path)) continue;
    visited.add(link.path);
    stack.push(link.path);
  }
}
dv.list(dv.array(Array.from(visited)).map(p => dv.page(p).file.link));
```

### 1.15 CSV ingest

```dataviewjs
const rows = await dv.io.csv("data/imports.csv");
dv.table(["Title", "Year"], rows.map(r => [r.title, r.year]));
```

### 1.16 Custom view with arguments

```dataviewjs
await dv.view("Scripts/leaderboard", { source: '"Books"', limit: 10 });
```

```js
// Scripts/leaderboard/view.js
module.exports = async ({ dv, input }) => {
  const rows = dv.pages(input.source)
    .where(p => p.score != null)
    .sort(p => p.score, "desc")
    .limit(input.limit ?? 10);
  dv.table(["Player", "Score"], rows.map(p => [p.file.link, p.score]));
};
```

### 1.17 Section / block links from JS

```dataviewjs
dv.paragraph(dv.sectionLink("Project Notes", "Action Items"));
dv.paragraph(dv.blockLink("Project Notes", "abc123", false, "see here"));
```

### 1.18 Tasks-plugin emoji translation in DQL

```dataview
TASK WHERE due
SORT due ASC
```

`due` here is the field promoted from `📅 2026-05-12`.

### 1.19 Conditional rendering in DataviewJS

```dataviewjs
const open = dv.current().file.tasks.where(t => !t.completed);
if (open.length === 0) {
  dv.paragraph("All clear ✅");
} else {
  dv.taskList(open, false);
}
```

### 1.20 Multi-key sort with custom comparator

```dataviewjs
dv.pages("#book")
  .sort(b => [b.genre, -b.rating], "asc",
        (a, b) => dv.compare(a[0], b[0]) || dv.compare(a[1], b[1]));
```

## 2. Plugin developer integration

### 2.1 Acquiring the API

```ts
import { getAPI, isPluginEnabled, Values } from "obsidian-dataview";

if (isPluginEnabled(this.app)) {
  const api = getAPI(this.app);
  // api: DataviewApi | undefined  (undefined while Dataview is still initializing)
}
```

Without the npm package: `app.plugins.plugins.dataview.api`. Beware: Dataview is late-loaded relative to many plugins — guard with `isPluginEnabled` or wait on `dataview:index-ready`.

### 2.2 The `DataviewApi` surface (excerpt from `src/api/plugin-api.ts`)

```ts
class DataviewApi {
  app: App;
  index: FullIndex;
  settings: DataviewSettings;
  evaluationContext: Context;
  io: DataviewIOApi;
  func: Record<string, BoundFunctionImpl>;
  value: Values;
  widget: Widgets;
  luxon: Luxon;

  // Queries (async)
  query(source, originFile?, settings?): Promise<Result<QueryResult, string>>;
  tryQuery(source, originFile?, settings?): Promise<QueryResult>;
  queryMarkdown(source, originFile?, settings?): Promise<Result<string, string>>;
  tryQueryMarkdown(source, originFile?, settings?): Promise<string>;

  // Pages (sync)
  page(path, originFile?): Page | undefined;
  pages(query?, originFile?): DataArray<Page>;
  pagePaths(query?, originFile?): DataArray<string>;

  // Expressions (sync)
  evaluate(expression, context?, originFile?): Result<Literal, string>;
  tryEvaluate(expression, context?, originFile?): Literal;
  evaluateInline(expression, origin): Result<Literal, string>;

  // Embedding views into your own UI (async)
  execute(source, container, component, filePath): Promise<void>;
  executeJs(code, container, component, filePath): Promise<void>;
}
```

### 2.3 Events on `metadataCache`

```ts
plugin.registerEvent(
  plugin.app.metadataCache.on("dataview:index-ready", () => {
    // Initial index pass complete. Safe to call api.pages(), api.query(), etc.
  })
);

plugin.registerEvent(
  plugin.app.metadataCache.on("dataview:metadata-change",
    (type, file, oldPath?) => {
      // type: "update" | "rename" | "delete"
    })
);
```

A third event the plugin recognises is `dataview:refresh-views`. External code can trigger it to force every visible Dataview block to re-render:

```ts
plugin.app.metadataCache.trigger("dataview:refresh-views");
```

### 2.4 Type guards via `Values`

```ts
import { Values } from "obsidian-dataview";

if      (Values.isHtml(v))     { /* rendered HTML */ }
else if (Values.isLink(v))     { /* Link object */ }
else if (Values.isDate(v))     { /* DateTime */ }
else if (Values.isDuration(v)) { /* Duration */ }
else if (Values.isArray(v))    { /* DataArray or array */ }
else if (Values.isObject(v))   { /* plain object */ }
else if (Values.isNumber(v))   { /* number */ }
else if (Values.isBoolean(v))  { /* boolean */ }
else if (Values.isString(v))   { /* string */ }
else if (Values.isFunction(v)) { /* function */ }
else if (Values.isNull(v))     { /* null */ }
```

### 2.5 Programmatic query from a plugin

```ts
const result = await api.query(
  `TABLE rating FROM #books WHERE rating >= 7 SORT rating DESC`,
  this.app.workspace.getActiveFile()?.path
);
if (result.successful) {
  const { type, headers, values } = result.value as any;
  // type === "table" — render however you want
} else {
  console.error(result.error);
}
```

### 2.6 Embedding a Dataview view into your plugin's UI

```ts
await api.execute(
  `TABLE rating FROM #books`,
  containerEl,
  this,             // a Component (your plugin / view)
  this.app.workspace.getActiveFile()?.path ?? ""
);
```

`api.executeJs(code, containerEl, this, filePath)` does the same for DataviewJS.

## 3. Performance

- The Dataview index is a single in-memory structure. Size scales with vault size, total inline fields, and tasks. Initial indexing happens once at plugin load; subsequent updates are incremental.
- **Refresh Interval** (plugin settings, ms) debounces re-renders after metadata changes. Too low → noticeable churn in busy vaults. Default is fine for most.
- **DQL is cheap to re-run** — parsed once, executed against in-memory data.
- **DataviewJS re-runs the entire script on every refresh.** Heavy work (large folder scans, regex over every page body) belongs in:
  - A `dv.view` file (still re-runs but separable / cacheable).
  - An external plugin (a custom toolkit plugin, if the vault has one).
  - Or a memoization layer on `window.<myCacheKey>` (works but feels hacky).
- **Tighten `FROM` first.** Fewer initial pages → fewer rows everywhere downstream.
- **Avoid `dv.pages()` without a source** if you can. Vault-wide scans are the typical hot spot.
- **Keep `dataviewjs` blocks short.** Obsidian lazy-renders codeblocks past a certain height — large blocks leave a visible gap until scrolled into view. Target ≤70 lines per block; extract helpers if you exceed.

## 4. Security

- DataviewJS executes JavaScript with full vault read access via Obsidian's adapter. Treat third-party `dataviewjs` blocks as untrusted code.
- The "Enable JavaScript Queries" setting is off by default after the plugin's security review. Keep it off in shared/sync vaults unless you trust every contributor.
- `dv.io.load(path)` can read **any file in the vault**, including hidden folders (with the exception that `dv.view` blocks dot-prefixed folders).

## 5. Caveats — full list

### DQL syntax

- **`FROM` operators** are `and`/`or`/`!`/`-`. **`WHERE` operators** are `&&`/`||`/`!` (or `and`/`or`/`not`). Don't mix the two positions.
- **`LIMIT` truncates whatever is current.** Place after `SORT` for top-N.
- **Two `SORT` clauses do not chain.** Use comma-separated keys for tie-breaking.
- **After `GROUP BY`, original-row fields disappear.** Use `rows.<field>`.
- **`CALENDAR` ignores `SORT` and `GROUP BY`.**
- **Subtasks under a matching parent are kept** in `TASK` queries.

### Metadata

- **Inline keys are normalized** (lowercase + dashes); YAML frontmatter keys are **not**.
- **Frontmatter links must be quoted** (`class: "[[Math]]"`).
- **No frontmatter on list items / tasks** — page-level only.
- **Inline values are single-line** — multiline only via YAML pipe `|`.
- **`completed` ≠ `checked`** — `completed` is true only for `[x]`; `checked` is true for any non-blank status.
- **Tasks-plugin recurrence/priority/dependency emojis are NOT auto-promoted** to fields (only date emojis are).

### DataviewJS

- **Folders need quoting inside `dv.pages('"folder"')`** — never `dv.pages("folder")`.
- **`dv.view` blocks dot-prefixed folders.**
- **`dv.view` paths are vault-rooted** (no leading `/`); pass `originFile` to `dv.io.*` to switch resolution.
- **DataArray operations are immutable** — only `mutate()` and `forEach()` are exceptions.
- **`task.visual` is writable** — assign before passing to `taskList` to override displayed text.
- **`dv.query` return type depends on query type** — branch on `result.value.type`.
- **`dv.evaluate` evaluates an expression**, not a full query.
- **Inline DQL** ` `= …` ` does not support query types or data commands.
- **Plugin API has no implicit current file** — pass `originFile` to `dv.query`, `dv.io.*`, `dv.view`.
- **DataviewJS re-runs on every refresh** — cache or extract long work.

### Type system

- **Type coercion is silent** — comparisons across mismatched types may yield surprising results. Guard with `typeof(x) = "..."` for date / duration / link comparisons.
- **`null` is falsy.** A missing field is `null`. Use `default(field, fallback)` to substitute.
- **`first()` / `last()` on empty arrays** return `undefined`. Aggregates on empty arrays may return `null` or `NaN` — guard with `length > 0`.
- **`flat` default depth is 1.** For deep flattening, pass an explicit depth or use `expand(...)` in DataviewJS.

### Misc

- **Heading characters `( ) / # [ ]`** in `[[file#Heading]]` break parsing. Keep headings to plain text + dash separators.
- **`file.day` is best-effort** — set when the filename matches a date pattern or a `Date::` field exists; null otherwise.
- **`file.tags` vs `file.etags`**: `#a/b` adds `#a` and `#a/b` to `file.tags`; `file.etags` keeps only `#a/b`.
- **Reserved field names** (`where`, `limit`, etc.) need bracket access: `row["where"]`.

## 6. Debugging recipes

### "Query returns nothing — why?"

1. Run the same `FROM` source alone: `LIST FROM <source>`. If empty, the source is the problem (often: folder not double-quoted, or tag with subtag that doesn't exist).
2. Strip the `WHERE` clause. If the rows reappear, the predicate is the problem (often: type mismatch between field and literal — `WHERE rating >= 7` fails when `rating` is the string `"7"`).
3. Drop `GROUP BY`. If you started seeing rows, you're trying to access an original-row field after grouping — switch to `rows.<field>`.

### "Field is missing on a page that obviously has it"

1. Check key normalization — inline `**Last Reviewed**::` becomes `last-reviewed`, but YAML `Last Reviewed:` stays `Last Reviewed`. Querying as `Last Reviewed` works for YAML, fails for inline; querying as `last-reviewed` works for inline, fails for YAML. Standardise on dash-separated lowercase in both.
2. Check for accidental linebreaks — inline values end at the next newline. A wrapped-over-two-lines value parses as just the first line.
3. Check the file is in the index — `dv.page("path/to/file")` returns `undefined` for unindexed files (Obsidian must have scanned them at least once).

### "Inline `$=` doesn't render — only shows the literal"

- Inline JS Queries setting is disabled. Enable both "Enable JavaScript Queries" and "Enable Inline JavaScript Queries".

### "Task box is interactive but doesn't toggle the source file"

- The note must be openable for write (not in a read-only sync state). Some setups (mobile + iCloud sync) sometimes lag the writeback.

### "TASK query shows an unexpected child task"

- Subtasks inherit a matching parent's match state. To filter strictly, run a DataviewJS query and use `expand("children")` after the predicate.

## 7. Custom helper plugin (if the vault has one)

A common vault pattern: a companion Obsidian plugin owns shared DataviewJS helpers, so individual `dataviewjs` blocks stay thin. Idiomatic shape:

```dataviewjs
const tk = app.plugins.plugins["vault-toolkit"].api.dataview;
const rows = tk.data.loadByStatus(dv, '"Some/Folder"', { status: "active" });
tk.ui.progressBars(this.container, { bars: [/* … */] });
```

Conventions worth honouring (the vault's own `CLAUDE.md` should spell them out):

- **Three hardcoded values per block**: the toolkit accessor, the source string, any domain constants. Everything else goes via the toolkit.
- **No domain business logic inside the plugin** — only generic data-loading and UI primitives. Domain knowledge lives in the calling block.
- **No hardcoded vault paths inside plugin functions** — pass them in as arguments.

If the vault has a toolkit plugin, read its own `CLAUDE.md` before adding new shared modules.
