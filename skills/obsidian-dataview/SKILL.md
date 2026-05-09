---
name: obsidian-dataview
description: Authoring queries, dashboards, and scripts for the Obsidian Dataview plugin — covers DQL (TABLE/LIST/TASK/CALENDAR), DataviewJS (`dv.*` API, DataArray, `dv.io`, `dv.view`, `dv.query`), inline DQL/JS, the metadata model (frontmatter, `Field:: value`, `[k:: v]`, `(k:: v)`, every `file.*` and task field), all DQL functions, and external plugin integration via `getAPI(app)`. Use this skill whenever the user mentions Dataview, DQL, DataviewJS, `dv.pages`, `dv.table`, `dv.taskList`, `dv.view`, `dv.io`, `file.tasks`, `file.outlinks`, `file.frontmatter`, asks to build a dashboard or query over notes ("list every note that…", "table of pages where…", "tasks due before…", "group books by genre"), writes ` ```dataview ` or ` ```dataviewjs ` blocks, uses inline `` `= …` `` or `` `$= …` ``, edits `.base` views or `dv.view` scripts, integrates Dataview from another plugin, or asks why a query returns wrong/empty results. Trigger even when "Dataview" is not named explicitly — any natural-language request to query Obsidian metadata is in scope.
---

# Dataview

Reference for authoring queries and scripts against the Obsidian **Dataview** plugin. Dataview indexes vault metadata (frontmatter, inline fields, tasks, links) and exposes it through four codeblock surfaces.

This file is the index. For deep coverage, consult the reference files (loaded only when relevant).

## What Dataview is

A read-only metadata index. It scans the vault, builds a typed in-memory database from frontmatter + inline fields + tasks + the Obsidian link graph, and renders results back into notes through codeblocks. It does **not** modify files (the one exception: clicking a checkbox in a rendered TASK block toggles the underlying source, but DataviewJS code itself cannot write).

## The four codeblock surfaces

| Surface | Fence | Use it for |
|---|---|---|
| **DQL query** | ` ```dataview ` | Static, declarative views. Safer, faster to render, no JS required. |
| **DataviewJS** | ` ```dataviewjs ` | Anything DQL can't express: control flow, multi-step computation, custom DOM, awaiting `dv.io`/`dv.query`, custom views via `dv.view`. |
| **Inline DQL** | `` `= expr` `` (default prefix `=`) | Single-value expression rendered inline mid-paragraph. |
| **Inline JS DQL** | `` `$= expr` `` (default prefix `$=`) | Same idea, full `dv` API available. |

JS surfaces are gated by the plugin's "Enable JavaScript Queries" toggle. Treat third-party DataviewJS as untrusted code — it has full vault read access.

## Decision tree — DQL vs DataviewJS

Pick **DQL** unless one of these is true:

- The output needs `await` (`dv.query`, `dv.io.csv`, `dv.io.load`, `dv.view`).
- You want loops, mutable state, conditionals over the rendered output (e.g. one header per group, then a per-group table).
- You want to render a custom DOM structure or multiple sub-views in one block.
- The transformation needs intermediate variables that DQL can't express.
- You need to import a `dv.view("Scripts/foo", input)` helper.

Otherwise DQL wins on conciseness, performance, and reactivity.

## DQL — minimum viable shape

```
<QUERY-TYPE> <fields>
FROM <source>
<DATA-COMMAND> <expression>
...
```

- Query type is mandatory and first: `TABLE`, `LIST`, `TASK`, `CALENDAR` (with `WITHOUT ID` variants for `TABLE`/`LIST`).
- `FROM` is optional but, if present, must come right after the query type (max one).
- `WHERE`, `SORT`, `GROUP BY`, `FLATTEN`, `LIMIT` are repeatable and **execute in the order written** — order matters (e.g. `LIMIT` before `SORT` truncates an unsorted set).

```dataview
TABLE rating, file.mtime AS Modified
FROM #book
WHERE rating >= 7
SORT rating DESC
LIMIT 10
```

Full grammar, every data command, multi-clause behavior, source syntax, expression operators, literals, and gotchas → **`references/dql.md`**.

## DataviewJS — the canonical shape

```dataviewjs
const pages = dv.pages('"Books"').where(p => p.rating >= 7);
for (const g of pages.groupBy(p => p.genre)) {
    dv.header(3, g.key);
    dv.table(
      ["Title", "Rating"],
      g.rows.sort(p => p.rating, "desc").map(p => [p.file.link, p.rating])
    );
}
```

Three things are always in scope:

- `dv` — the API object (also aliased `dataview`).
- `this` — the post-processor context (rarely needed directly).
- Top-level `await` — yes, it's allowed inside `dataviewjs`.

Full `dv.*` catalog (rendering, page access, queries, I/O, views, links, dates, evaluation), the DataArray method list, sync vs async cheatsheet, and end-to-end examples → **`references/dataviewjs.md`**.

## What Dataview indexes

Every page exposes:

- **Implicit `file.*` fields** — `file.name`, `file.path`, `file.folder`, `file.link`, `file.size`, `file.ctime`/`cday`, `file.mtime`/`mday`, `file.tags` (subtag-expanded), `file.etags` (no subtag expansion), `file.inlinks`, `file.outlinks`, `file.aliases`, `file.tasks`, `file.lists`, `file.frontmatter`, `file.day`, `file.starred`.
- **YAML frontmatter** — every key is a field, types coerced (number, boolean, date, list, object, link).
- **Inline fields** — three syntaxes:
  - `Key:: value` (line form, key visible in Reader mode)
  - `[Key:: value]` (bracketed, embedable mid-line, key visible)
  - `(Key:: value)` (parenthesized, **key hidden** in Reader mode)
  - Keys are normalized to lowercase + dashes (`Last Reviewed::` → `last-reviewed`).
- **Tasks and list items** — every `- [<char>]` line, with auto-extracted fields (`status`, `completed`, `fullyCompleted`, `text`, `due`, `scheduled`, `start`, `completion`, `created`, `children`, `link`, `section`, `tags`, `outlinks`, …) and Tasks-plugin emoji shorthands (📅 ✅ ➕ 🛫 ⏳).

Full field type system, every `file.*` field, every task field, frontmatter vs inline rules, normalization edge cases → **`references/metadata.md`**.

## Functions

Dataview exposes a rich function library, available in DQL directly and in DataviewJS via `dv.func.<name>`. Categories: numeric (`round`, `min`, `sum`, `reduce`, `minby`, …), string (`lower`, `regexreplace`, `containsword`, `split`, `truncate`, …), list/object (`length`, `contains`, `filter`, `map`, `unique`, `flat`, `extract`, …), date (`date`, `dateformat`, `striptime`, `dur`, `durationformat`), constructors (`object`, `list`, `link`, `embed`, `elink`), utility (`default`, `choice`, `typeof`, `meta`, `localtime`, `currencyformat`, `hash`).

Every function with signature and one-line example → **`references/functions.md`**.

## Common patterns and integration

Cookbook (open tasks across the vault, daily-note metric aggregation, leaderboards, custom `dv.view` helpers, link/section/block links, recursive task expansion, group-by then per-group table), settings that affect rendering, plugin-developer integration (`getAPI(app)`, `dataview:index-ready`, `dataview:metadata-change`, `dataview:refresh-views`), performance notes, and the full caveats list → **`references/patterns.md`**.

## Order of operations cheatsheet — DQL

1. **Query type** picks what gets rendered (TABLE/LIST/TASK/CALENDAR).
2. **`FROM`** picks the initial page set (single position, source-level booleans `and`/`or`/`!`).
3. Each subsequent clause runs on the previous output:
   - `WHERE pred` — filter (predicate-level booleans `&&`/`||`/`!`).
   - `SORT key [ASC|DESC], key2 [ASC|DESC]` — multi-key only via comma; a second `SORT` clause overwrites, doesn't tie-break.
   - `GROUP BY key` — collapses rows into groups; thereafter only `key` and `rows` are addressable.
   - `FLATTEN field` — explodes a list-valued field; reverses GROUP BY.
   - `LIMIT n` — truncate; put it **after** `SORT` for top-N.

## Common pitfalls

- **Folders need double-quoting** in `dv.pages` and `FROM`: `dv.pages('"Books"')`, `FROM "Books"`. Never bare.
- **`FROM` boolean syntax** uses `and`/`or`/`!`/`-`. **`WHERE` boolean syntax** uses `&&`/`||`/`!`. Don't mix the two positions.
- **`LIMIT` before `SORT`** truncates an unsorted set — `SORT … LIMIT n` is the top-N idiom.
- **Two `SORT` clauses do not chain.** The second resorts the already-sorted list, breaking nothing. Use `SORT a ASC, b DESC` for tie-breaking.
- **After `GROUP BY`, original-row fields disappear.** Reach them via `rows.<field>` (a list) or `rows[0].<field>`.
- **Reserved names need bracket access** — `row["where"]`, `row["limit"]` if a field name collides with a keyword.
- **`completed` ≠ `checked`.** `completed` is true only for `[x]`. `checked` is true for any non-blank status (`/`, `-`, `>`, etc.).
- **Tasks-plugin recurrence/priority/dependency emojis are NOT auto-promoted** to fields. Only the date emojis (📅 ✅ ➕ 🛫 ⏳) are. To filter on `🔁` / priority / `⛔` use plain `[recurrence:: weekly]`-style annotations.
- **Frontmatter links must be quoted**: `class: "[[Math]]"`, not bare `[[Math]]`.
- **Header chars `( ) / # [ ]`** in `[[file#Heading]]` break parsing — keep headings to plain text + dash separators.
- **Inline field keys are normalized**: `**Bold Field**::` → `bold-field`. Query the normalized form.
- **`dv.view` cannot read dot-prefixed folders** (`.scripts/`). Throws "custom view not found".
- **Paths in `dv.io` and `dv.view`** are vault-rooted; pass `originFile` to switch resolution.
- **`dv.query` returns `Result<QueryResult>`** — branch on `.successful` and `.value.type` (`"list"`/`"table"`/`"task"`/`"calendar"`). `dv.tryQuery` throws instead.
- **DataviewJS re-runs the whole script** on every refresh. Hot expensive work belongs in `dv.view` files or external plugins, not inline.

## Custom helper plugins

If the vault has a companion Obsidian plugin owning shared DataviewJS helpers (a common pattern — sometimes called a "vault toolkit"), prefer the toolkit accessor over duplicating logic across notes:

```js
const tk = app.plugins.plugins["vault-toolkit"].api.dataview;
const rows = tk.data.loadByStatus(dv, '"Some/Folder"', { status: "active" });
tk.ui.progressBars(this.container, { bars: [...] });
```

The vault's own `CLAUDE.md` should document its toolkit conventions. The typical shape: inside a `dataviewjs` block keep three hardcoded values — the toolkit accessor, the source string (folder/tag), and any domain constants — and route everything else through the toolkit.

For one-off views inside a single note, plain `dataviewjs` is fine — keep blocks under ~70 lines or Obsidian's lazy-render leaves a gap until you scroll the block into view.

## How to use this skill

1. Identify the surface: DQL block, DataviewJS, inline DQL, inline JS, or external-plugin integration.
2. For DQL: pick query type, build the source, append clauses in the right order. Consult `references/dql.md` for syntax, `references/functions.md` for the function vocabulary, `references/metadata.md` for available fields.
3. For DataviewJS: lean on `references/dataviewjs.md` for the API + DataArray methods. If the user wants a recurring helper, propose a `dv.view` file or — if the vault has a custom toolkit plugin — a module on that.
4. When something doesn't render or returns empty, walk the "Common pitfalls" list above before spelunking — most issues are one of those.
5. Keep blocks focused. If a single block grows past ~70 lines, suggest extracting helpers.

Source of truth for everything in this skill: <https://blacksmithgu.github.io/obsidian-dataview/>.
