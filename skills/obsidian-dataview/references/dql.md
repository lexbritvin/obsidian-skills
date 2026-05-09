# DQL — Dataview Query Language

The declarative query language used inside ` ```dataview ` codeblocks and (for single expressions) inline `` `= …` ``. SQL-like, read-only, executes against the Dataview index.

## 1. Query shape

```
<QUERY-TYPE> <fields>
FROM <source>
<DATA-COMMAND> <expression>
<DATA-COMMAND> <expression>
...
```

- The **query type** is mandatory and first.
- `FROM` is optional, max one, must follow the query type immediately.
- Other data commands (`WHERE`, `SORT`, `GROUP BY`, `FLATTEN`, `LIMIT`) are repeatable in any order, executed top-to-bottom.
- Comments: `// like this`.

## 2. Query types

### 2.1 LIST

Bullet-point list. Default content is the file link; an optional extra expression appears after each link.

```dataview
LIST
```

```dataview
LIST file.folder
```

```dataview
LIST "Created " + file.cday FROM "Daily"
```

### 2.2 LIST WITHOUT ID

Drops the leading file link, leaving only the expression.

```dataview
LIST WITHOUT ID file.folder
```

```dataview
LIST WITHOUT ID length(rows) + " items in " + key GROUP BY type
```

### 2.3 TABLE

Multi-column table. The implicit first column is `file.link`; `WITHOUT ID` removes it. Use `AS "Header"` to rename a column.

```dataview
TABLE rating, file.mtime AS Modified FROM #book
```

```dataview
TABLE WITHOUT ID file.link AS "Game", file.etags AS "Tags" FROM #games
```

Computed columns are full expressions:

```dataview
TABLE default(finished, date(today)) - started AS "Played for" FROM #games
```

### 2.4 TASK

Interactive checkbox list. Operates at task granularity, not page; clicking a box toggles the underlying source file. All task-level fields (§ `metadata.md`) are available.

```dataview
TASK
```

```dataview
TASK WHERE !completed AND contains(tags, "#shopping")
```

```dataview
TASK WHERE !completed GROUP BY file.link
```

A subtask matches if its parent matches — children of a matching parent are kept even if their own predicates fail.

### 2.5 CALENDAR

Calendar grid. Requires a date-typed expression as additional info; each match is a dot. `SORT` and `GROUP BY` have no effect.

```dataview
CALENDAR file.ctime
```

```dataview
CALENDAR due WHERE typeof(due) = "date"
```

There are no `WITHOUT ID` variants for `TASK` or `CALENDAR`.

## 3. Data commands

### 3.1 FROM

Sets the initial page set from a **source** expression. Source-level booleans: `and`, `or`, `!` (or `-`). Parenthesise to group.

```
FROM #projects
FROM "Daily" AND #review
FROM (#book OR #podcast) AND -"Archive"
FROM [[]]
FROM outgoing([[Dashboard]])
```

See § 4 for source syntax.

### 3.2 WHERE

Predicate filter. Predicate-level booleans: `&&`, `||`, `!`, plus `and`, `or`, `not`. **Do not** confuse with the source-level booleans — those only apply inside `FROM`.

```dataview
LIST WHERE file.mtime >= date(today) - dur(7 days)
```

```dataview
LIST FROM #projects
WHERE !completed AND file.ctime <= date(today) - dur(1 month)
```

Multiple `WHERE` clauses are equivalent to chaining with `AND`.

### 3.3 SORT

Multi-key in a single clause via comma; ties on the first key break by the next.

```
SORT field [ASC|ASCENDING|DESC|DESCENDING]
SORT field1 [ASC|DESC], field2 [ASC|DESC], ...
```

```dataview
TABLE rating FROM #book SORT rating DESC, file.name ASC
```

A second `SORT` clause **overwrites** the first (it re-sorts the already-sorted list); use comma-separated keys for tie-breaking.

### 3.4 GROUP BY

Collapses rows into groups keyed by the expression. Each group exposes:

- `key` — the grouping value.
- `rows` — DataArray of the original rows.

After `GROUP BY`, only `key` and `rows` are addressable. Original-row fields are reached via `rows.<field>` (returns a list) or `rows[0].<field>`.

```dataview
TABLE rows.file.link AS Pages, length(rows) AS Count
FROM #book
GROUP BY genre
```

Group expression with a custom name:

```dataview
LIST rows.file.link
GROUP BY (file.folder + "/" + file.name) AS path
```

### 3.5 FLATTEN

Explodes a list-valued field into one row per element. Useful for `file.tasks`, `file.lists`, `file.outlinks`, multi-author fields. Often the inverse of `GROUP BY`.

```dataview
TABLE authors FROM #LiteratureNote FLATTEN authors
```

```dataview
TASK FROM "Projects"
FLATTEN file.tasks
WHERE !file.tasks.completed
```

Computed flatten:

```dataview
TABLE name FROM #book FLATTEN (split(authors, ",")) AS name
```

### 3.6 LIMIT

Caps the row count. Place **after** `SORT` for top-N.

```dataview
TABLE rating FROM #book SORT rating DESC LIMIT 10
```

## 4. Sources (FROM)

### 4.1 Tag

```
#tag
#parent/child
```

`#parent` matches any subtag (`#parent/child`).

### 4.2 Folder (and specific file)

```
"folder"
"folder/sub"
"folder/File"
"folder/File.md"
```

Path is vault-rooted, no leading slash. Folders match files in subfolders too. If a folder and file share a name, Dataview prefers the folder unless `.md` is given.

### 4.3 Link

- `[[Note]]` — pages that link **into** `Note`.
- `outgoing([[Note]])` — pages that `Note` links to.
- `[[]]` — current file.

```
LIST FROM [[]]
LIST FROM outgoing([[Dashboard]])
```

### 4.4 Combination

```
#a and "folder"
[[Food]] or [[Exercise]]
#food and !#fastfood
(#a or #b) and (#c or #d)
```

Source-level operators are `and`/`or`/`!`/`-`. They only apply inside `FROM`.

## 5. Literals

### 5.1 Primitives

- Numbers: `0`, `1337`, `-200`, `3.14`.
- Booleans: `true`, `false`.
- Strings: `"text"` (double quotes; escape backslashes for regex).
- Null: `null`.

### 5.2 Dates

`date(<iso>)` — ISO-8601, with optional time:

- `date(2021-11-11)`
- `date(2021-09-20T20:17)`

Special tokens:

| Token | Meaning |
|---|---|
| `date(today)` | Today, no time. |
| `date(now)` | Now, with time. |
| `date(tomorrow)` | Next day. |
| `date(yesterday)` | Previous day. |
| `date(sow)` | Start of week. |
| `date(eow)` | End of week. |
| `date(som)` | Start of month. |
| `date(eom)` | End of month. |
| `date(soy)` | Start of year. |
| `date(eoy)` | End of year. |

Component access on a date: `.year`, `.month`, `.day`, `.hour`, `.minute`, `.second`, `.weekday`, etc. (Luxon-backed.)

### 5.3 Durations

`dur(<n> <unit>[, <n> <unit>...])`:

| Unit | Aliases |
|---|---|
| second | `s`, `sec`, `second`, `seconds` |
| minute | `m`, `min`, `minute`, `minutes` |
| hour | `h`, `hr`, `hour`, `hours` |
| day | `d`, `day`, `days` |
| week | `w`, `wk`, `week`, `weeks` |
| month | `mo`, `month`, `months` |
| year | `yr`, `year`, `years` |

Examples: `dur(1 s)`, `dur(3 wk)`, `dur(1s 2m 3h)`, `dur(1 second 2 minutes 3 hours)`, `dur(9 years, 8 months, 4 days)`.

Date − Date → Duration. Date ± Duration → Date.

### 5.4 Links

- `[[Page]]`, `[[]]` (current file).
- `[[Page#Heading]]` — heading link.
- `[[Page^block-id]]` — block link.
- `[[Page|Display]]` — display alias.

In **frontmatter**, links must be quoted: `class: "[[Math]]"` — bare `[[…]]` is interpreted as a YAML flow array.

### 5.5 Lists and objects

- List: `[1, 2, 3]`, `[[1, 2], [3, 4]]`.
- Object: `{ a: 1, b: 2 }`. (Inline objects only via DataviewJS / nested YAML; DQL accepts object literals in expressions but rarely needs them.)

## 6. Expressions

### 6.1 Operators

Arithmetic / concatenation (works on numbers, strings, lists):

`+ - * / %`

Comparison (return booleans):

`= != < <= > >=`

Logical:

`&& || !` (also `and`, `or`, `not`)

Aggregation operands for `reduce()`:

`+ - * / & |`

Type coercion across mismatched types is implicit but unreliable — guard with `typeof(x) = "..."` when comparing.

### 6.2 Indexing and dotted access

- List: `mylist[0]`, zero-based.
- Object/page: `obj.field` or `obj["field"]`.
- Reserved field name: bracket form — `row["where"]`, `row["limit"]`.
- Through a link to another page: `[[Assignment]].duedate` reads the field on the linked page.
- If a field's value **is** a link, dot through the field: `Class.timetable` (where `Class:: [[Math]]`), not `[[Class]].timetable`.

### 6.3 Function calls

`functionName(arg1, arg2, ...)`. The function vocabulary lives in `functions.md`.

### 6.4 Lambdas

`(arg1, arg2, ...) => <expression>`. Used by `filter`, `map`, `all`, `any`, `none`, `minby`, `maxby`.

```dataview
LIST FROM #project WHERE all(file.tasks, (t) => t.completed)
```

### 6.5 Vectorisation

Many operators and functions auto-broadcast: applied element-wise to a list and return a list of equal length.

## 7. Special symbols

- `this` — the current page object (e.g. `WHERE contains(file.outlinks, this.file.link)`).
- `[[]]` — current file as a link literal.
- `key`, `rows` — only after `GROUP BY`. Keys for grouping value, rows for the original DataArray.
- `row[...]` — fallback indexing for fields whose names collide with reserved keywords.

## 8. Implicit fields available in DQL

Page-level (`file.*`) and task-level fields are catalogued in `metadata.md`. Use them anywhere a field can appear (output column, `WHERE`, `SORT`, `GROUP BY`, `FLATTEN`, function arg).

## 9. Inline DQL — `` `= expr` ``

Single-value expressions only. `this.` resolves to the current page; `[[Page]].field` resolves to another page's field.

```
Today is `= date(today)`.
File: `= this.file.name`.
Days to deadline: `= [[exams]].deadline - date(today)`.
```

No query types and no data commands. The default prefix `=` is configurable.

## 10. Common DQL patterns

### Recently modified pages
```dataview
LIST WHERE file.mtime >= date(today) - dur(7 days) SORT file.mtime DESC
```

### Top-N rated books
```dataview
TABLE rating, author FROM #book SORT rating DESC LIMIT 10
```

### Open tasks grouped by source file
```dataview
TASK WHERE !completed GROUP BY file.link
```

### Authors expanded — one per row
```dataview
TABLE authors FROM #LiteratureNote FLATTEN authors
```

### Overdue items
```dataview
TABLE due FROM "" WHERE typeof(due) = "date" AND due < date(today)
```

### Calendar of deadlines
```dataview
CALENDAR due WHERE typeof(due) = "date"
```

### Tag breakdown with counts
```dataview
TABLE length(rows) AS Count
GROUP BY file.etags AS Tag
SORT Count DESC
```

### Linked-mention dashboard
```dataview
LIST FROM [[]] WHERE !contains(file.outlinks, this.file.link)
```
("Pages that link into this file but this file does not link back to.")

### Daily-note metric series
```dataview
TABLE WITHOUT ID file.link AS Day, mood, energy
FROM "Daily"
WHERE typeof(mood) = "number"
SORT file.day DESC
LIMIT 14
```

## 11. Caveats

- **`FROM` syntax vs `WHERE` syntax.** Source booleans inside `FROM` are `and`/`or`/`!`/`-`. Predicate booleans inside `WHERE` are `&&`/`||`/`!` (or `and`/`or`/`not`). Don't mix.
- **`LIMIT` truncates whatever is current.** Place after `SORT` for top-N.
- **Two `SORT`s do not chain.** Use comma-separated keys for tie-breaking; a second `SORT` clause re-sorts.
- **After `GROUP BY`, original-row fields disappear.** Use `rows.<field>`.
- **Frontmatter links must be quoted.**
- **Reserved field names need bracket access:** `row["where"]`.
- **Inline keys are normalised** (lowercase + dashes). Query `last-reviewed`, not `Last Reviewed`, for cross-note reliability.
- **`completed` is true only for `[x]`.** `checked` is true for any non-blank status.
- **Subtasks under matching parents are kept** in `TASK` queries.
- **`CALENDAR` ignores `SORT` and `GROUP BY`.**
- **Heading characters `( ) / # [ ]`** in `[[file#Heading]]` break parsing.
- **Type coercion is silent.** Mismatched types in comparisons can yield surprising results — `typeof(x)` guards are recommended for date / duration / link arithmetic.
- **Null vs missing.** A missing field is `null`. Use `default(field, fallback)` or `nonnull(list)` to guard. `null` is falsy.

## 12. Performance

- DQL parses once per render, then runs against the in-memory index. Cheap.
- Tighten `FROM` first (fewer pages → fewer rows everywhere downstream).
- `WHERE` on a primitive field is fast; calling functions over `file.tasks` for every page is the typical hot spot in big vaults.
- The "Refresh Interval (ms)" setting governs how aggressively blocks re-render after metadata changes.
