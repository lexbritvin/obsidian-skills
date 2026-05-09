# Metadata model — what Dataview indexes

Every page Dataview returns (`dv.page`, `dv.pages`, `dv.current`, plus rows in DQL queries) is the merge of four data sources, all flattened into one record with implicit `file.*` fields and user-defined fields.

## 1. Where metadata comes from

### 1.1 YAML frontmatter

```yaml
---
alias: "document"
last-reviewed: 2026-05-08
rating: 5
tags: [book, fiction]
thoughts:
  rating: 8
  reviewable: false
class: "[[Math]]"      # links MUST be quoted in YAML
deadline: 2026-12-31
---
```

- Page-level only — no frontmatter on list items / tasks.
- Full type support: text, number, boolean, date, list, object (via nesting), link.
- Visibility: hidden in Reader mode by default (Obsidian setting).

### 1.2 Inline field — line form `Key:: value`

```markdown
Basic Field:: Some random value
**Bold Field**:: Nice!
```

- The line must start with `Key::` (allowing leading bold/italic markers). Anything before `Key::` (other than markup) prevents parsing.
- The value runs from after `::` to the next newline.
- Visibility: visible in Reader mode (literal `Key:: value` text shows).

### 1.3 Inline field — bracket form `[Key:: value]`

```markdown
I would rate this a [rating:: 9]! It was [mood:: acceptable].
- [ ] Send mail [due:: 2026-04-05].
```

- Embeds anywhere in a line. Multiple per line allowed.
- Works on list items / tasks: scope is the list item, not the page.
- Visibility: in Reader mode renders as `key value` (key visible, brackets gone).

### 1.4 Inline field — parenthesis form `(Key:: value)`

```markdown
This will not show the (longKeyIDontNeedWhenReading:: key).
```

- Same semantics as bracket form, but the **key is hidden** in Reader mode (only the value renders).
- Use when the field name is noisy in prose.

### 1.5 Key normalization

Dataview canonicalizes inline-field keys before indexing:

- Spaces → dashes.
- All letters → lowercase.
- Bold/italic markdown stripped.
- Emoji and non-Latin keys are supported, but **must** use bracket or parenthesis form (line form rejects them).

`**Last Reviewed**::` → queryable as `last-reviewed`.
`[🎅:: ho]` → queryable as `🎅`.

YAML frontmatter keys are **not** normalized the same way — match the literal key.

### 1.6 Duplicates

If the same key appears in multiple sources (frontmatter + inline, or repeated inlines), the values **merge into a list automatically**.

### 1.7 Multiline values

- Inline values end at the next newline. No multiline support for inline fields.
- For multiline strings use YAML pipe operator `|`.

## 2. Field types

| Type | Detected from | Example |
|---|---|---|
| `text` | Default fallback | `Example:: This is some normal text.` |
| `number` | Unquoted numerics | `rating: 5`, `Example:: -80`, `Example:: 2.4` |
| `boolean` | `true` / `false` | `Example:: true` |
| `date` | ISO-8601 | `2026-04`, `2026-04-18`, `2026-04-18T04:19:35.000+06:30` |
| `duration` | `<n> <unit>[, …]` | `7 hours`, `9 years, 8 months, 4 days` |
| `link` | `[[Page]]` (quoted in YAML) | `class: "[[Math]]"` |
| `list` | YAML array, comma-separated inline | `tags: [a, b]`, `Example:: "yes", "or", "no"` |
| `object` | Nested YAML | `thoughts: { rating: 8, reviewable: false }` |
| `null` | Missing field | `default(field, fallback)` to guard |
| `html` | API helpers | `Values.isHtml(v)` to detect |

`typeof(x)` returns one of: `"number"`, `"string"`, `"boolean"`, `"date"`, `"duration"`, `"link"`, `"array"`, `"object"`, `"null"`.

### Date components

A date value exposes Luxon-backed component access:

- `.year`, `.month`, `.day`, `.hour`, `.minute`, `.second`, `.millisecond`
- `.weekday` (1 = Mon, 7 = Sun in Luxon)
- `.weekNumber`, `.ordinal`

```dataview
TABLE file.day.year AS Y, file.day.weekday AS WD FROM "Daily"
```

### Duration components

`.years`, `.months`, `.days`, `.hours`, `.minutes`, `.seconds`, `.milliseconds`.

## 3. Implicit page-level fields — `file.*`

| Field | Type | Meaning |
|---|---|---|
| `file.name` | string | File name shown in the sidebar (no extension). |
| `file.folder` | string | Folder path containing the file. |
| `file.path` | string | Full vault-relative path, including filename. |
| `file.ext` | string | Extension (typically `md`). |
| `file.link` | Link | Link object pointing at the file. |
| `file.size` | number | File size in bytes. |
| `file.ctime` | DateTime | Creation timestamp (date+time). |
| `file.cday` | Date | Creation date (date-only). |
| `file.mtime` | DateTime | Last-modified timestamp. |
| `file.mday` | Date | Last-modified date. |
| `file.tags` | list<string> | All tags **with subtags expanded** — `#a/b/c` adds `#a`, `#a/b`, `#a/b/c`. |
| `file.etags` | list<string> | Explicit tags only (no subtag expansion). |
| `file.inlinks` | list<Link> | Pages linking **into** this file. |
| `file.outlinks` | list<Link> | Pages this file links to. |
| `file.aliases` | list<string> | Aliases declared in frontmatter. |
| `file.tasks` | list<Task> | Every task in the file. |
| `file.lists` | list<ListItem> | Every list item, tasks included. |
| `file.frontmatter` | object | Raw frontmatter (untyped, before Dataview parsing). |
| `file.day` | Date \| null | Date inferred from filename pattern (`yyyy-mm-dd`, `yyyymmdd`) or a `Date::` field. Null if neither. |
| `file.starred` | boolean | True if file is bookmarked via Obsidian's Bookmarks core plugin. |

User-defined frontmatter and inline fields appear directly on the page object — `dv.current().rating`, `dv.current()["last-reviewed"]`. Top-level YAML keys are exposed both as `page.<key>` (Dataview-typed) and `page.file.frontmatter.<key>` (raw value).

## 4. Tasks and list items

Detection: any line matching `- [<char>]` is a task. Anything else `- foo` is a plain list item. Both are included in `file.lists`; only tasks are in `file.tasks`.

Custom statuses are accepted: `- [/]`, `- [-]`, `- [>]`, `- [!]`, etc. The character inside the brackets becomes `task.status`.

### 4.1 Auto-extracted fields

| Field | Type | Notes |
|---|---|---|
| `status` | string | Character inside `[ ]` (`" "`, `"x"`, `"/"`, …). |
| `checked` | boolean | True for any non-blank status. |
| `completed` | boolean | True only for `[x]`. |
| `fullyCompleted` | boolean | True iff this task and **all subtasks** are completed. |
| `text` | string | Raw markdown text including inline-field annotations. |
| `visual` | string | Rendered text — overridable in DataviewJS by mutating `task.visual`. |
| `line` | number | Zero-indexed line in the file. |
| `lineCount` | number | How many markdown lines the task spans. |
| `path` | string | Vault-relative path of the source file. |
| `section` | Link | Link to the containing section. |
| `header` | Link | Synonymous with `section`. |
| `link` | Link | Closest linkable block (block-id link if present, else section). |
| `tags` | list<string> | Tags inside the task text. |
| `outlinks` | list<Link> | Wikilinks inside the task. |
| `children` | list<Task\|ListItem> | Nested subtasks / sublist items. |
| `task` | boolean | True for tasks (`file.lists` mixes tasks and plain items). |
| `real` | boolean | True for items derived from real list lines (vs synthesized). |
| `list` | number | Line number of the parent list block. |
| `parent` | number \| null | Line number of the parent task / list (null at top level). |
| `annotated` | boolean | True if the task carries `field::` annotations. |
| `blockId` | string \| null | Block ID (`^abc123`) if present. |

### 4.2 Date fields

Set via inline-field syntax or Tasks-plugin emoji shorthands.

| Key | Inline form | Tasks-plugin emoji |
|---|---|---|
| `due` | `[due:: 2026-05-10]` | 📅 (also 🗓️) |
| `completion` | `[completion:: 2026-05-08]` | ✅ |
| `created` | `[created:: 2026-05-01]` | ➕ |
| `start` | `[start:: 2026-05-08]` | 🛫 |
| `scheduled` | `[scheduled:: 2026-05-09]` | ⏳ |

Example:

```markdown
- [x] Sent the email ✅ 2026-05-08
- [ ] Finish report 📅 2026-05-12 ⏳ 2026-05-09
```

Surfaces as `task.completion = 2026-05-08`, `task.completed = true`, etc.

### 4.3 Tasks-plugin compatibility — partial

Dataview promotes only the **date emojis** above into fields. The following Tasks-plugin shortcuts are **not** auto-promoted:

- `🔁` recurrence
- `🔼 ⏫ 🔽 ⏬ 🔺` priority
- `⛔ 🆔` dependencies

To filter on those in Dataview, encode them as plain inline fields:

```markdown
- [ ] Weekly review [recurrence:: every week] [priority:: high]
```

### 4.4 Subtask hierarchy

- `task.children` returns nested Task/ListItem objects.
- `fullyCompleted` cascades — a parent is fully complete only when itself and every descendant has `completed === true`.
- Walk all tasks recursively in DataviewJS:

```js
const all = dv.current().file.tasks.expand("children");
```

- In `TASK` queries, **subtasks under matching parents are kept** even if their own predicates fail.

### 4.5 Inheritance

Tasks inherit all parent-page fields. So `task.someFrontmatterKey` works:

```dataview
TASK WHERE !completed AND project = "X"
// `project` here is read from the parent page's frontmatter.
```

### 4.6 Mutation — the only writable surface

Clicking a checkbox in a rendered TASK block toggles the underlying file. DataviewJS itself cannot write to files — the click is a user interaction routed through Obsidian.

## 5. The four codeblock types

| Surface | Fence / inline | Read JS? | When to use |
|---|---|---|---|
| DQL block | ` ```dataview ` | No | Static views: TABLE/LIST/TASK/CALENDAR. |
| DataviewJS block | ` ```dataviewjs ` | Yes (toggle) | Loops, multi-step, await, custom DOM, `dv.view`. |
| Inline DQL | `` `= expr` `` | No | Single-value expression mid-paragraph. |
| Inline JS | `` `$= expr` `` | Yes (toggle) | Same as inline DQL but with full `dv` API. |

Both JS surfaces are gated by **"Enable JavaScript Queries"**. Inline JS has its own toggle in addition. Inline DQL has a toggle too.

Default prefixes — both configurable in plugin settings:

- Inline DQL prefix: `=`
- Inline JS prefix: `$=`

## 6. Plugin settings that affect output

- **Enable JavaScript Queries** — master JS toggle (off by default after the security review).
- **Enable Inline JavaScript Queries** — separate toggle for `$=` even when full JS blocks are off.
- **Enable Inline Queries** — toggle for inline DQL `=`.
- **Inline Query Prefix** / **Inline JavaScript Query Prefix** — string prefixes inside backticks.
- **Date Format** (Luxon, default `MMMM dd, yyyy`) — how date values render.
- **Date+Time Format** (Luxon, default `h:mm a — MMMM dd, yyyy`) — how datetime values render.
- **Default Link Format** — short vs full path for Link rendering in tables/lists.
- **Pretty Rendering / "Render dataview tasks as nice rows"** — enriched task rendering.
- **Render Null As** — string substitute for nulls in tables (default `\-`).
- **Refresh Interval (ms)** — debounce between re-renders after metadata changes.
- **Warn on Empty Result** — placeholder for empty queries.
- **Task Link Location** — where the source-file link sits relative to the task.
- **Automatic View Refreshing** — toggle live updates when files change.
- **Maximum Recursive Render Depth** — bounds nested-list rendering.

## 7. Limitations

- **Read-only.** Dataview cannot create, edit, or delete files or fields. The TASK block is the only interactive write path, and only via user click.
- **No frontmatter on list items / tasks.** Page-level only. Per-task fields use inline syntax.
- **Inline values are single-line.** Use frontmatter with YAML pipe `|` for multi-line.
- **Single vault scope.** No cross-vault queries.
- **Reactivity bounded by Refresh Interval.** Inline edits surface after the configured debounce, not instantly.
- **Tasks-plugin integration is partial.** Only date emojis. Recurrence/priority/dependency need plain inline fields.
- **Frontmatter links must be quoted.** Bare `[[…]]` in YAML parses as a flow array.
- **YAML frontmatter keys are not normalized** the way inline keys are. `Last Reviewed:` is queryable only as `Last Reviewed`. For consistency, prefer dash-separated lowercase keys in frontmatter too (`last-reviewed:`).
- **DataviewJS is untrusted code** — full vault read access. Treat third-party blocks as code.
