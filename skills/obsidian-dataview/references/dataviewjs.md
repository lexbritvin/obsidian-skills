# DataviewJS — the `dv` API and DataArray

JavaScript surface of Dataview. Lives inside ` ```dataviewjs ` blocks (full block) or `` `$= …` `` (inline). Both require the **"Enable JavaScript Queries"** setting.

Top-level `await` is allowed in `dataviewjs` blocks. Methods marked **⌛** below are async — they need `await`.

## 1. Entry points and scope

### 1.1 Full block

````markdown
```dataviewjs
const pages = dv.pages('"Books"').where(p => p.rating >= 7);
dv.table(["Title", "Rating"], pages.map(p => [p.file.link, p.rating]));
```
````

In scope:

- `dv` — the API (alias `dataview`).
- `this` — the post-processor context. `this.container` ≡ `dv.container`.
- Top-level `await`.

### 1.2 Inline JS — `` `$= expr` ``

```markdown
Last edited: `$= dv.current().file.mtime`
```

Single-value expression. Full `dv` available. Default prefix `$=` (configurable).

### 1.3 External plugin / console

```js
const api = app.plugins.plugins.dataview.api;
// or, if depending on the npm package:
import { getAPI, isPluginEnabled } from "obsidian-dataview";
const api = getAPI(this.app);
```

The plugin API is essentially the same surface, but **without** an implicit "current file" — methods that depend on it (e.g. `dv.current()`, `dv.io.load(path)` without `originFile`) need an explicit `originFile` arg.

## 2. The `dv` object — catalog

### 2.1 Context

| Member | Type | Notes |
|---|---|---|
| `dv.current()` | Page record | The page the block lives in. |
| `dv.container` | HTMLElement | Where rendering injects DOM. Mutate freely. |
| `dv.component` | Component | The Obsidian Component owning the rendered subtree (use as parent for `MarkdownRenderer.renderMarkdown`). |
| `dv.app` | App | The Obsidian app — `app.vault`, `app.workspace`, `app.metadataCache`. |
| `dv.luxon` | Luxon module | `dv.luxon.DateTime.now()`, `dv.luxon.Duration.fromISO("PT1H30M")`. |

### 2.2 Page / source access (sync)

```ts
dv.pages(source?: string): DataArray<Page>
dv.pagePaths(source?: string): DataArray<string>
dv.page(path: string | Link, originFile?: string): Page | undefined
```

The `source` string uses the same syntax as DQL `FROM`. Folders need **double quotes inside the string**.

```js
dv.pages()                       // every markdown page
dv.pages("#books")               // tag
dv.pages('"Books"')              // folder
dv.pages('"Books" and -#draft')  // boolean source
dv.pages("[[Index]]")            // pages linking into Index
dv.pages("outgoing([[Index]])")  // pages Index links to
dv.page("Books/The Raisin")
dv.page(dv.current().file.outlinks[0])
```

### 2.3 Rendering — DOM (sync)

All rendering helpers append to `dv.container`.

```ts
dv.el(tag: string, text: any, opts?: { cls?: string; attr?: Record<string,string>; container?: HTMLElement }): HTMLElement
dv.header(level: 1|2|3|4|5|6, text: string): HTMLElement
dv.paragraph(text: any): HTMLElement
dv.span(text: any): HTMLElement
dv.list(values: any[] | DataArray<any>): void
dv.taskList(tasks: Task[] | DataArray<Task>, groupByFile?: boolean = true): void
dv.table(headers: string[], rows: any[][] | DataArray<any[]>): void
dv.execute(source: string): void           // embed a DQL query result
dv.executeJs(code: string): void           // embed a JS snippet's output
```

```js
dv.el("b", "bold", { cls: "my-class", attr: { alt: "tooltip" }});
dv.header(2, "Section");
dv.paragraph("Inline text.");
dv.list(dv.pages("#book").file.link);
dv.taskList(dv.pages("#project").file.tasks.where(t => !t.completed), false);
dv.table(["Title", "Rating"], dv.pages("#book").map(b => [b.file.link, b.rating]));
dv.execute("LIST FROM #tag");
dv.executeJs("dv.list([1, 2, 3])");
```

#### `dv.taskList` — special behaviours

- `groupByFile = true` (default) groups tasks under their source file as a header.
- `groupByFile = false` renders one flat interactive list.
- Each task carries a writable `visual` field — assign a string before passing to `taskList` to override the displayed text without mutating `text`.

```js
const tasks = dv.pages("#project").file.tasks.where(t => !t.completed);
tasks.forEach(t => { t.visual = `[${t.path}] ${t.text}`; });
dv.taskList(tasks, false);
```

### 2.4 Rendering — Markdown strings (sync)

These return strings instead of rendering. Pass through `dv.paragraph(...)` to display, or post-process.

```ts
dv.markdownTable(headers: string[], rows: any[][] | DataArray<any[]>): string
dv.markdownList(values: any[] | DataArray<any>): string
dv.markdownTaskList(tasks: Task[] | DataArray<Task>): string
```

### 2.5 Custom views — `dv.view` ⌛

```ts
await dv.view(path: string, input?: any): Promise<void>
```

Loads `path/view.js` (and optionally `path/view.css`) and executes it. `path` is **vault-rooted** (no leading `/`). Cannot read dot-prefixed folders (`.scripts/` throws "custom view not found"). If `path` resolves directly to a `.js` file, only that file is loaded.

```js
// In any note:
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

The exported entry receives a single object `{ dv, input }`. Inside the file, `dv` is the same API; `input` is whatever the caller passed.

### 2.6 Query evaluation ⌛

```ts
await dv.query(source: string, originFile?: string, settings?: QueryApiSettings):
  Promise<Result<QueryResult, string>>
await dv.tryQuery(source, originFile?, settings?): Promise<QueryResult>
await dv.queryMarkdown(source, originFile?, settings?): Promise<Result<string, string>>
await dv.tryQueryMarkdown(source, originFile?, settings?): Promise<string>
```

`Result<T, string>` shape: `{ successful: true; value: T } | { successful: false; error: string }`.

`QueryResult` shape varies by query type:

```js
// LIST
{ type: "list", values: [link1, link2, ...] }

// TABLE
{ type: "table", headers: ["file.name", "rating"], values: [["A", 8], ["B", 6]] }

// TASK
{ type: "task", values: [task1, task2, ...] }

// CALENDAR
{ type: "calendar", values: [{ date, link, value }, ...] }
```

```js
const r = await dv.query(`TABLE WITHOUT ID file.name, rating FROM #book`);
if (r.successful) dv.table(r.value.headers, r.value.values);
else dv.paragraph(`Query failed: ${r.error}`);
```

Use `tryQuery` when you want exceptions instead of `Result` branching.

### 2.7 Expression evaluation (sync)

```ts
dv.evaluate(expression: string, context?: Record<string, any>):
  Result<any, string>
dv.tryEvaluate(expression: string, context?: Record<string, any>): any
```

Evaluates a DQL expression (not a full query — no `FROM`/`WHERE`). `this` is implicit.

```js
dv.evaluate("2 + 2")                              // { successful: true, value: 4 }
dv.evaluate("x + 2", { x: 3 })                    // { successful: true, value: 5 }
dv.tryEvaluate("length(this.file.tasks)")         // task count for current file
dv.tryEvaluate('link("Books/The Raisin")')        // Link object
```

### 2.8 Value coercion / construction (sync)

```ts
dv.array(value: any[] | DataArray<any>): DataArray<any>
dv.isArray(value: any): boolean
dv.fileLink(path: string, embed?: boolean, displayName?: string): Link
dv.sectionLink(path: string, section: string, embed?: boolean, display?: string): Link
dv.blockLink(path: string, blockId: string, embed?: boolean, display?: string): Link
dv.date(text: string | Link | DateTime): DateTime | null
dv.duration(text: string): Duration | null
dv.parse(value: string): any   // returns Link | DateTime | Duration depending on input
dv.clone<T>(value: T): T       // deep-clone respecting DV types
dv.compare(a: any, b: any): -1 | 0 | 1
dv.equal(a: any, b: any): boolean
```

```js
dv.fileLink("2026-05-08")                       // [[2026-05-08]]
dv.fileLink("Books/The Raisin", true)           // ![[Books/The Raisin]]
dv.fileLink("Test", false, "Test File")         // [[Test|Test File]]
dv.sectionLink("Index", "Books")                // [[Index#Books]]
dv.blockLink("Notes", "abc123")                 // [[Notes#^abc123]]
dv.date("2021-08-08")                           // DateTime
dv.duration("9 hours, 2 minutes")               // Duration
dv.parse("[[A]]")                               // Link { path: "A" }
dv.parse("2020-08-14")                          // DateTime
dv.parse("9 seconds")                           // Duration
dv.compare(1, 2)                                // -1
```

### 2.9 File I/O — `dv.io.*`

`csv` and `load` are async; `normalize` is sync. `originFile` defaults to the current file (relative paths resolve from there).

```ts
await dv.io.csv(path: string | Link, originFile?: string):
  Promise<DataArray<Record<string, string>> | undefined>
await dv.io.load(path: string | Link, originFile?: string):
  Promise<string | undefined>
dv.io.normalize(path: string | Link, originFile?: string): string
```

```js
const rows = await dv.io.csv("data/imports.csv");
const text = await dv.io.load("Notes/Inbox.md");
dv.io.normalize("Test", "dataview/test2/Index.md");
// → "dataview/test2/Test.md"
```

### 2.10 Function and value namespaces

| Namespace | Purpose |
|---|---|
| `dv.func` | Every DQL function bound for direct JS use — `dv.func.length(arr)`, `dv.func.dateformat(d, "yyyy-MM-dd")`, `dv.func.contains(arr, x)`, etc. |
| `dv.value` | `Values` type-predicate utility — `Values.isHtml`, `isLink`, `isDate`, `isDuration`, `isArray`, `isObject`, `isNumber`, `isBoolean`, `isString`, `isFunction`, `isNull`. |
| `dv.widget` | Widget constructors used internally for pretty rendering. |
| `dv.luxon` | Re-exported Luxon — `DateTime`, `Duration`, `Interval`. |

```js
import { Values } from "obsidian-dataview"; // external use; in-block use dv.value
if (dv.value.isLink(x)) { /* ... */ }
```

## 3. DataArray — the core collection

`dv.pages(...)` and friends return `DataArray<T>` — a Proxy-wrapped array with extra operators. **Operations return new arrays** (immutable) — except `mutate()` and `forEach()`.

### 3.1 Swizzling

Indexing a DataArray with a **field name** maps each element to that field, flattening if the field is itself an array.

```js
dv.pages().file.name             // DataArray<string>
dv.pages("#book").genres         // flattened list of all genres
dv.pages().file.tasks            // flattened list of every task
```

Equivalent to `.to("name")`.

### 3.2 Methods — full list

```ts
type ArrayFunc<T, O> = (elem: T, index: number, arr: T[]) => O;
type ArrayComparator<T> = (a: T, b: T) => number;
```

#### Filter / map

```ts
where(pred: ArrayFunc<T, boolean>): DataArray<T>     // alias filter
filter(pred: ArrayFunc<T, boolean>): DataArray<T>
map<U>(fn: ArrayFunc<T, U>): DataArray<U>
flatMap<U>(fn: ArrayFunc<T, U[]>): DataArray<U>
mutate(fn: ArrayFunc<T, any>): DataArray<any>        // mutates in place
```

#### Slice

```ts
limit(n: number): DataArray<T>
slice(start?: number, end?: number): DataArray<T>
concat(other: Iterable<T>): DataArray<T>
```

#### Search / predicates

```ts
indexOf(el: T, fromIndex?: number): number
find(pred: ArrayFunc<T, boolean>): T | undefined
findIndex(pred: ArrayFunc<T, boolean>, fromIndex?: number): number
includes(el: T): boolean
every(pred: ArrayFunc<T, boolean>): boolean
some(pred: ArrayFunc<T, boolean>): boolean
none(pred: ArrayFunc<T, boolean>): boolean
```

#### Aggregate

```ts
join(sep?: string): string  // sep defaults to ", "
sum(): number
avg(): number
min(): number
max(): number
first(): T   // undefined if empty
last(): T    // undefined if empty
length: number
```

#### Sort / group / dedup

```ts
sort<U>(
  key: ArrayFunc<T, U>,
  direction?: "asc" | "desc",
  comparator?: ArrayComparator<U>
): DataArray<T>

groupBy<U>(
  key: ArrayFunc<T, U>,
  comparator?: ArrayComparator<U>
): DataArray<{ key: U; rows: DataArray<T> }>

distinct<U>(
  key?: ArrayFunc<T, U>,
  comparator?: ArrayComparator<U>
): DataArray<T>
```

#### Field navigation

```ts
to(key: string): DataArray<any>      // explicit form of swizzle
expand(key: string): DataArray<any>  // recursive flatten by following `key`
```

`expand("children")` is the standard recipe for walking nested tasks/lists.

#### Iteration / conversion

```ts
forEach(fn: ArrayFunc<T, void>): void
array(): T[]                          // → vanilla JS array
[Symbol.iterator](): Iterator<T>      // for-of
[index]                               // numeric indexing
[fieldName]                           // swizzling
```

### 3.3 Patterns

```js
// All open tasks in the vault, ungrouped
const open = dv.pages().file.tasks.where(t => !t.completed);
dv.taskList(open, false);

// Books grouped by genre, sorted by rating per group
for (const g of dv.pages('"Books"').groupBy(p => p.genre)) {
  dv.header(3, g.key);
  dv.table(
    ["Title", "Rating"],
    g.rows.sort(p => p.rating, "desc").map(p => [p.file.link, p.rating])
  );
}

// Recursive task expansion (incl. nested subtasks)
const all = dv.current().file.tasks.expand("children");
dv.taskList(all.where(t => !t.completed), false);

// Custom multi-key comparator via dv.compare
dv.pages("#book")
  .sort(b => [b.genre, -b.rating], "asc",
        (a, b) => dv.compare(a[0], b[0]) || dv.compare(a[1], b[1]));

// Concat across multiple sources
const both = dv.pages("#a").concat(dv.pages("#b"));
```

## 4. Sync vs async cheatsheet

| Sync | Async (need `await`) |
|---|---|
| `dv.current`, `dv.page`, `dv.pages`, `dv.pagePaths` | `dv.view` |
| `dv.array`, `dv.isArray` | `dv.io.csv`, `dv.io.load` |
| `dv.list`, `dv.table`, `dv.taskList` | `dv.query`, `dv.tryQuery` |
| `dv.markdownTable`, `dv.markdownList`, `dv.markdownTaskList` | `dv.queryMarkdown`, `dv.tryQueryMarkdown` |
| `dv.el`, `dv.header`, `dv.paragraph`, `dv.span` | |
| `dv.execute`, `dv.executeJs` | |
| `dv.fileLink`, `dv.sectionLink`, `dv.blockLink` | |
| `dv.date`, `dv.duration`, `dv.parse` | |
| `dv.compare`, `dv.equal`, `dv.clone` | |
| `dv.evaluate`, `dv.tryEvaluate` | |
| `dv.io.normalize` | |

## 5. End-to-end examples

### 5.1 Books grouped by genre, sorted by rating

```dataviewjs
for (const g of dv.pages("#book").groupBy(p => p.genre)) {
  dv.header(3, g.key);
  dv.table(
    ["Title", "Time Read", "Rating"],
    g.rows.sort(k => k.rating, "desc").map(k => [k.file.link, k["time-read"], k.rating])
  );
}
```

### 5.2 DFS over the link graph from current note

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

### 5.3 Aggregate metric over a date range

```dataviewjs
const start = dv.date("2026-05-01");
const end   = dv.date("2026-05-08");
const days  = dv.pages('"Daily"')
  .where(p => p.file.day >= start && p.file.day <= end);

const moods = days.array()
  .map(p => p.mood)
  .filter(v => typeof v === "number");

const mean = moods.reduce((a, b) => a + b, 0) / Math.max(moods.length, 1);
dv.paragraph(`Avg mood last 7 days: ${mean.toFixed(2)} (n=${moods.length})`);
```

### 5.4 CSV + table render

```dataviewjs
const rows = await dv.io.csv("data/imports.csv");
dv.table(["Title", "Year"], rows.map(r => [r.title, r.year]));
```

### 5.5 Custom view with arguments

```dataviewjs
await dv.view("Scripts/dashboard", { sources: ["#a", "#b"], limit: 5 });
```

```js
// Scripts/dashboard/view.js
module.exports = async ({ dv, input }) => {
  for (const src of input.sources) {
    dv.header(3, src);
    dv.list(dv.pages(src).file.link.limit(input.limit));
  }
};
```

### 5.6 dv.query for structured rendering

```dataviewjs
const r = await dv.query(`TABLE WITHOUT ID file.name, rating FROM #book`);
if (r.successful) dv.table(r.value.headers, r.value.values);
else dv.paragraph(`Query failed: ${r.error}`);
```

### 5.7 Markdown table embedded in a paragraph

```dataviewjs
const md = dv.markdownTable(
  ["File", "Genre", "Rating"],
  dv.pages("#book").sort(b => b.rating, "desc").map(b => [b.file.link, b.genre, b.rating])
);
dv.paragraph(md);
```

## 6. Pitfalls

- **Folders need quoting inside `dv.pages()`**: `dv.pages('"Books"')`, never `dv.pages("Books")`.
- **`dv.view` blocks dot-prefixed folders.** `Scripts/...` works; `.scripts/...` throws.
- **`view.js` + `view.css`**: pass a folder path; both are loaded.
- **Path resolution** for `dv.io.*` and `dv.view` is vault-rooted unless you pass `originFile`.
- **DataArray immutability** — every operator returns a new array; only `mutate()` and `forEach()` are exceptions.
- **`task.visual`** is writable — assign before `taskList` to override displayed text.
- **`dv.query` return type depends on the query type** — branch on `result.value.type`.
- **`dv.evaluate` evaluates an expression**, not a full query (no `FROM`/`WHERE`).
- **Inline DQL** ` `= …` ` does **not** support query types or data commands. Inline JS `` `$= …` `` has the full `dv` API.
- **Plugin-API caveat**: outside a `dataviewjs` block, no implicit current file — pass `originFile` to `dv.io.*`, `dv.view`, `dv.query`.
- **DataviewJS re-runs the whole script on every refresh.** Keep blocks short (~70 lines max — Obsidian's lazy-render leaves a gap on bigger blocks). Move expensive work into `dv.view` files or external plugins.
- **Frontmatter access**: top-level keys appear both as `page.<key>` (Dataview-typed) and `page.file.frontmatter.<key>` (raw value).
- **`file.tags` vs `file.etags`**: `#a/b` adds `#a` and `#a/b` to `file.tags`; `file.etags` keeps only `#a/b`.
- **`first()` / `last()`** on empty arrays return `undefined`. `min/max/sum/avg` on empty arrays may return `null` or `NaN` depending on element types — guard with `length > 0`.
