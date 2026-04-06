---
name: create-query
description: Creates Obsidian Tasks plugin query blocks with filters, sorting, grouping, and layout options. Use when the user asks to build a task dashboard, show overdue tasks, list tasks by priority, create a task view or report in a note, write a tasks code block, group tasks by project, filter tasks by folder or date, make a weekly review page, or anything involving a tasks query block in markdown.
compatibility: Designed for Obsidian vaults with the Tasks plugin installed.
---

# Creating Task Queries

Write `tasks` code blocks in markdown notes. The Tasks plugin renders them as live, filtered views of tasks across the vault.

## Query Block Syntax

````markdown
```tasks
not done
due before tomorrow
sort by priority
```
````

Each line is a separate instruction. All filter lines must match (implicit AND). Queries are case-insensitive except boolean operators (`AND`, `OR`, `NOT` must be UPPERCASE — lowercase is silently ignored). Add `explain` to any query to debug how Tasks interprets it.

## Common Patterns

**Overdue tasks:**
````markdown
```tasks
not done
due before today
sort by due
```
````

**Due this week, grouped by file:**
````markdown
```tasks
not done
due this week
group by filename
sort by due
```
````

**High priority:**
````markdown
```tasks
not done
(priority is highest) OR (priority is high)
sort by due
```
````

**Exclude folders:**
````markdown
```tasks
not done
(folder does not include Templates) AND (folder does not include Archive)
sort by due
```
````

**Backlog (no dates):**
````markdown
```tasks
not done
no due date
no scheduled date
sort by path
limit 20
```
````

These cover the most common cases. For anything more complex — date ranges, custom scripting, regex filters, layout options — read the reference files below.

## Boolean Filters

Combine multiple conditions on a single line with operators:

```
(filter 1) AND (filter 2)
(filter 1) OR (filter 2)
NOT (filter 1)
(filter 1) AND NOT (filter 2)
```

Operators must be UPPERCASE — this is a common mistake. `and`/`or` in lowercase are silently ignored, making the query behave unexpectedly. Delimiters: `()`, `[]`, `{}`, `""` (don't mix types on one line).

## Reference Documentation

Read these when the inline patterns above aren't enough:

- [Query Reference](references/QUERY_REFERENCE.md) — complete filter, sorting, grouping, layout options, boolean combinations, and custom scripting. Read when you need a specific filter syntax or want to check available sort/group fields.
- [Examples](references/EXAMPLES.md) — more query recipes including dependencies, compact views, task trees, and date category grouping. Read when building a complex dashboard or looking for inspiration.
