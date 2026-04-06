# Tasks Query Reference

## Contents
- Filters (status, dates, dependencies, priority, recurrence, description/tags, file properties)
- Combining Filters (boolean AND, OR, NOT, XOR)
- Sorting
- Grouping
- Layout (hide/show elements, display modes)
- Other Instructions (explain, comments, limit)
- Custom Scripting (filter/sort/group by function)

## Filters

### Status Filters

```
done
not done
status.name (includes|does not include) <string>
status.name (regex matches|regex does not match) /regex/i
status.type (is|is not) (TODO|DONE|IN_PROGRESS|ON_HOLD|CANCELLED|NON_TASK)
```

### Date Filters

Available for: `due`, `scheduled`, `starts`, `created`, `done`, `cancelled`, `happens`

```
<date-field> (on|before|after|on or before|on or after) <date>
<date-field> (in|before|after|in or before|in or after) <date-range>
has <date-field> date
no <date-field> date
<date-field> date is invalid
```

`happens` = earliest of start, scheduled, and due dates.

#### Date Values

Absolute: `2025-01-15`

Relative: `today`, `tomorrow`, `yesterday`, `last/this/next monday` (any day of week), `in two weeks`

#### Date Ranges

```
YYYY-MM-DD YYYY-MM-DD          (two absolute dates)
(last|this|next) week           
(last|this|next) month          
(last|this|next) quarter        
(last|this|next) year           
YYYY-Www                        (week number, e.g. 2025-W03)
YYYY-mm                         (month, e.g. 2025-01)
YYYY-Qq                         (quarter, e.g. 2025-Q1)
YYYY                            (year)
```

### Dependency Filters

```
is blocking / is not blocking
is blocked / is not blocked
has id / no id
id (includes|does not include) <string>
has depends on / no depends on
```

### Priority Filter

```
priority is (above|below|not)? (lowest|low|none|medium|high|highest)
```

### Recurrence Filters

```
is recurring / is not recurring
recurrence (includes|does not include) <string>
```

### Description & Tags

```
description (includes|does not include) <string>
description (regex matches|regex does not match) /regex/i
has tags / no tags
tag (includes|does not include) <tag>
tags (include|do not include) <tag>
tag (regex matches|regex does not match) /regex/i
```

### File Property Filters

```
path (includes|does not include) <path>
root (includes|does not include) <root>
folder (includes|does not include) <folder>
filename (includes|does not include) <filename>
heading (includes|does not include) <string>
```

All text filters also support `regex matches|regex does not match`.

Template variables: `{{query.file.path}}`, `{{query.file.folder}}`, `{{query.file.filename}}`

### Other Filters

```
exclude sub-items
limit to <number> tasks    / limit <number>
limit groups to <number> tasks / limit groups <number>
```

## Combining Filters (Boolean)

Within a single line, wrap filters in delimiters and join with operators:

```
(filter 1) AND (filter 2)
(filter 1) OR (filter 2)
NOT (filter 1)
(filter 1) XOR (filter 2)
(filter 1) AND NOT (filter 2)
(filter 1) OR NOT (filter 2)
```

**Operators must be UPPERCASE.** Delimiters: `()`, `[]`, `{}`, `""` (don't mix types on one line).

Evaluation order: NOT > XOR > AND > OR. Use parentheses for clarity.

## Sorting

Default sort (always appended): `status.type`, `urgency`, `due`, `priority`, `path`

```
sort by status              sort by status.name         sort by status.type
sort by due                 sort by scheduled           sort by start
sort by created             sort by done                sort by cancelled
sort by happens             sort by priority            sort by urgency
sort by description         sort by recurring           sort by tag [N]
sort by path                sort by filename            sort by heading
sort by id                  sort by random
sort by function <expression>
```

Add `reverse` after any sort: `sort by due reverse`

Multiple `sort by` lines: first has highest priority.

## Grouping

```
group by status             group by status.name        group by status.type
group by due                group by scheduled           group by start
group by created            group by done                group by cancelled
group by happens            group by priority            group by urgency
group by recurring          group by recurrence          group by tags
group by path               group by root                group by folder
group by filename           group by heading             group by backlink
group by id
group by function <expression>
```

Multiple `group by` lines create nested groups.

## Layout

### Hide/Show Task Elements

```
hide priority               hide created date           hide start date
hide scheduled date         hide due date               hide done date
hide cancelled date         hide recurrence rule        hide on completion
hide tags                   hide id                     hide depends on
```

### Hide/Show Query Elements

```
hide edit button            hide postpone button        hide backlink
hide task count             hide toolbar
show urgency                show tree
```

### Display Modes

```
short mode          (emojis only, hover for details)
full mode           (default, shows everything)
```

## Other Instructions

```
explain             (shows how query is interpreted — great for debugging)
# comment           (lines starting with # are ignored)
ignore global query
```

## Custom Scripting

For advanced logic, use JavaScript expressions:

```
filter by function <expression>
sort by function <expression>
group by function <expression>
```

Available objects: `task` (with properties like `.due`, `.description`, `.priority`, `.status`, `.tags`, `.file`), `query` (with `.file` properties).

TasksDate properties: `.format('YYYY-MM-DD')`, `.category.groupText`, `.fromNow.groupText`, `.moment`
