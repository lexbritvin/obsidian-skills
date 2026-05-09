---
name: obsidian-tasks-cli
description: Reads, filters, and updates Obsidian tasks using the CLI. Lists tasks, marks them done, toggles status, counts tasks, and runs advanced queries via the Tasks plugin API. Use when the user asks to see their tasks, check what's overdue, mark something done, analyze task progress, get a task summary, count tasks, or retrieve task data programmatically — even if they don't mention CLI or API.
compatibility: Requires Obsidian 1.12+ with CLI enabled (Settings > General > CLI) and Obsidian running.
---

# Managing Tasks via CLI

Use the `obsidian` CLI to read and update tasks. Obsidian must be running.

Start with simple CLI commands. Only use `obsidian eval` when you need filtering that CLI flags can't express (by priority, dates, tags, or custom logic).

## Listing Tasks

```bash
obsidian tasks todo                    # all incomplete tasks
obsidian tasks done                    # completed tasks
obsidian tasks todo format=json        # JSON output for parsing
obsidian tasks verbose                 # grouped by file
obsidian tasks todo total              # count only
obsidian tasks todo path="Projects"    # filter by folder
obsidian tasks todo file="Shopping"    # filter by file name
obsidian tasks daily todo              # tasks from daily note
```

JSON output shape:

```json
[
  {
    "status": " ",
    "text": "- [ ] Buy groceries 📅 2025-04-10",
    "file": "Projects/Shopping.md",
    "line": "12"
  }
]
```

Status characters: `" "` = TODO, `"/"` = IN_PROGRESS, `"x"` = DONE, `"-"` = CANCELLED.

## Updating Tasks

```bash
obsidian task path="note.md" line=12 toggle      # toggle status
obsidian task path="note.md" line=12 done         # mark done
obsidian task path="note.md" line=12 todo         # mark todo
obsidian task path="note.md" line=12 status="/"   # set in-progress
```

To add a new task to a file:

```bash
obsidian append path="note.md" content="- [ ] New task 📅 2025-04-10"
```

## Advanced Queries via Plugin API

For filtering by priority, dates, tags, or any combination — use `obsidian eval` to access the Tasks plugin cache. The cache holds all parsed task objects with rich queryable properties.

Wrap eval code in an async IIFE so you can call `loadData()` (used to read settings like `globalQuery`) and because `obsidian eval` doesn't support top-level return:

```bash
obsidian eval code="(async()=>{
  const plugin = app.plugins.plugins['obsidian-tasks-plugin'];
  const tasks = plugin.cache.getTasks();
  // filter, map, return
})()"
```

## Respecting globalQuery

The Tasks plugin has a `globalQuery` setting (Settings → Tasks → "Global query") that applies to every rendered query block. The cache returned by `getTasks()` does **not** automatically apply it — you'll see all tasks, including ones the user has globally excluded.

To make eval recipes consistent with what the user sees in their query blocks, read `globalQuery` and translate the filters to JS:

```javascript
const plugin = app.plugins.plugins['obsidian-tasks-plugin'];
const data = await plugin.loadData();
const globalQuery = data.globalQuery || '';
```

Then translate each line into a JS predicate. The Tasks query DSL is rich, but the common cases map cleanly:

| Tasks DSL line | JS predicate on `t` |
|----------------|---------------------|
| `folder includes X` | `t.taskLocation.path.startsWith('X')` |
| `folder does not include X` | `!t.taskLocation.path.startsWith('X')` |
| `path includes X` | `t.taskLocation.path.includes('X')` |
| `path does not include X` | `!t.taskLocation.path.includes('X')` |
| `filename includes X` | `t.taskLocation.path.split('/').pop().includes('X')` |
| `tag includes #X` / `tags include #X` | `t.tags.includes('#X')` |
| `tag does not include #X` | `!t.tags.includes('#X')` |
| `filter by function (task.file.tags ?? []).includes('#X')` (file frontmatter tag) | `(t.taskLocation._tasksFile?.tags ?? []).includes('#X')` |
| `filter by function ! (task.file.tags ?? []).includes('#X')` | `!(t.taskLocation._tasksFile?.tags ?? []).includes('#X')` |
| `not done` | `!t.isDone` |
| `done` | `t.isDone` |
| `is recurring` | `t.isRecurring` |
| `is blocked` / `is not blocked` | `t.isBlocked` / `!t.isBlocked` |
| `priority is highest/high/medium/low/lowest` | `t.priorityName === 'Highest'` etc. |
| `priority is above medium` | `['Highest','High'].includes(t.priorityName)` |
| `has due date` / `no due date` | `t.due.moment !== null` / `t.due.moment === null` |
| `due before today` | `t.due.moment && t.due.moment.isBefore(window.moment().startOf('day'))` |
| `due this week` | `t.due.moment && t.due.moment.isSame(window.moment(), 'week')` |

For more complex DSL (boolean combinations, custom scripting, regex), translate inline as needed or warn the user that the filter is too complex to auto-apply.

**Practical pattern** — read globalQuery once, parse the common exclusion lines, then filter:

```javascript
const data = await plugin.loadData();
const lines = (data.globalQuery || '').split('\n');
const excludedFolders = lines
  .map(l => l.match(/^\s*folder does not include\s+(.+?)\s*$/i))
  .filter(Boolean).map(m => m[1]);
const excludedFileTags = lines
  .map(l => l.match(/^\s*filter by function\s*!\s*\(task\.file\.tags\s*\?\?\s*\[\]\)\.includes\(["'](#?[^"']+)["']\)/))
  .filter(Boolean).map(m => m[1].startsWith('#') ? m[1] : '#' + m[1]);

const tasks = plugin.cache.getTasks().filter(t => {
  if (excludedFolders.some(f => t.taskLocation.path.startsWith(f))) return false;
  const fileTags = t.taskLocation._tasksFile?.tags ?? [];
  if (excludedFileTags.some(tag => fileTags.includes(tag))) return false;
  return true;
});
```

This covers folder exclusions and file-frontmatter tag exclusions — the two most common globalQuery patterns. For other DSL filters, extend the parsing or write a tailored predicate.

**Note:** changing `globalQuery` via `saveData()` does not refresh rendered query blocks until the plugin reloads (`app.plugins.disablePlugin('obsidian-tasks-plugin'); app.plugins.enablePlugin('obsidian-tasks-plugin')`). The cache itself never applies globalQuery — that's why these recipes filter manually.

## Task Object Properties

| Property | Type | Description |
|----------|------|-------------|
| `description` | string | Task text without emoji signifiers |
| `status.symbol` | string | ` `, `x`, `/`, `-` |
| `isDone` | boolean | Whether task is complete |
| `isBlocked` | boolean | Blocked by a dependency |
| `isBlocking` | boolean | Other tasks depend on this |
| `priorityName` | string | `Highest`, `High`, `Medium`, `Normal`, `Low`, `Lowest` |
| `urgency` | number | Computed urgency score |
| `due` | TasksDate | Due date (check `due.moment` for presence) |
| `scheduled` | TasksDate | Scheduled date |
| `start` | TasksDate | Start date |
| `created` | TasksDate | Created date |
| `done` | TasksDate | Completion date |
| `cancelled` | TasksDate | Cancellation date |
| `tags` | string[] | Tags on the task |
| `recurrenceRule` | string | Recurrence pattern |
| `isRecurring` | boolean | Has recurrence |
| `id` | string | Task ID (for dependencies) |
| `taskLocation.path` | string | File path from vault root |
| `taskLocation.lineNumber` | number | Line number in file |
| `originalMarkdown` | string | Raw markdown line |

## TasksDate

A TasksDate object is always present, but the underlying date may be empty. Check `task.due.moment` — it's `null` when no date is set:

```javascript
task.due.moment                          // Moment.js object or null
task.due.moment.format('YYYY-MM-DD')     // "2025-04-10"
task.due.moment.isBefore(window.moment()) // overdue check
task.due.formatAsDate()                  // formatted string
task.due.category.groupText              // "Overdue", "Today", "Future", "Undated"
```

## Recipes

Each recipe uses an async IIFE, reads `globalQuery` for folder exclusions, and applies them before filtering. Add more globalQuery translations from the table above when relevant.

**Task count by status:**

```bash
obsidian eval code="(async()=>{
  const plugin = app.plugins.plugins['obsidian-tasks-plugin'];
  const data = await plugin.loadData();
  const excluded = (data.globalQuery||'').split('\n').map(l=>l.match(/^\s*folder does not include\s+(.+?)\s*$/i)).filter(Boolean).map(m=>m[1]);
  const tasks = plugin.cache.getTasks().filter(t => !excluded.some(f => t.taskLocation.path.startsWith(f)));
  const counts = {};
  tasks.forEach(t => { const s = t.isDone ? 'done' : 'todo'; counts[s] = (counts[s]||0)+1; });
  return JSON.stringify(counts);
})()"
```

**Overdue tasks:**

```bash
obsidian eval code="(async()=>{
  const plugin = app.plugins.plugins['obsidian-tasks-plugin'];
  const data = await plugin.loadData();
  const excluded = (data.globalQuery||'').split('\n').map(l=>l.match(/^\s*folder does not include\s+(.+?)\s*$/i)).filter(Boolean).map(m=>m[1]);
  const today = window.moment().startOf('day');
  return plugin.cache.getTasks()
    .filter(t => !excluded.some(f => t.taskLocation.path.startsWith(f)))
    .filter(t => !t.isDone && t.due.moment && t.due.moment.isBefore(today))
    .map(t => t.description + ' 📅 ' + t.due.formatAsDate() + ' (' + t.taskLocation.path + ')')
    .join('\n') || 'No overdue tasks';
})()"
```

**High priority tasks:**

```bash
obsidian eval code="(async()=>{
  const plugin = app.plugins.plugins['obsidian-tasks-plugin'];
  const data = await plugin.loadData();
  const excluded = (data.globalQuery||'').split('\n').map(l=>l.match(/^\s*folder does not include\s+(.+?)\s*$/i)).filter(Boolean).map(m=>m[1]);
  return plugin.cache.getTasks()
    .filter(t => !excluded.some(f => t.taskLocation.path.startsWith(f)))
    .filter(t => !t.isDone && ['Highest','High'].includes(t.priorityName))
    .map(t => t.priorityName + ': ' + t.description)
    .join('\n') || 'No high priority tasks';
})()"
```

**Tasks by folder:**

```bash
obsidian eval code="(async()=>{
  const plugin = app.plugins.plugins['obsidian-tasks-plugin'];
  const data = await plugin.loadData();
  const excluded = (data.globalQuery||'').split('\n').map(l=>l.match(/^\s*folder does not include\s+(.+?)\s*$/i)).filter(Boolean).map(m=>m[1]);
  const groups = {};
  plugin.cache.getTasks()
    .filter(t => !excluded.some(f => t.taskLocation.path.startsWith(f)))
    .filter(t => !t.isDone)
    .forEach(t => {
      const folder = t.taskLocation.path.split('/').slice(0,-1).join('/') || '/';
      groups[folder] = (groups[folder]||0)+1;
    });
  return Object.entries(groups).sort((a,b)=>b[1]-a[1]).map(([f,c])=>c+' '+f).join('\n');
})()"
```

**Tasks due this week:**

```bash
obsidian eval code="(async()=>{
  const plugin = app.plugins.plugins['obsidian-tasks-plugin'];
  const data = await plugin.loadData();
  const excluded = (data.globalQuery||'').split('\n').map(l=>l.match(/^\s*folder does not include\s+(.+?)\s*$/i)).filter(Boolean).map(m=>m[1]);
  const start = window.moment().startOf('week');
  const end = window.moment().endOf('week');
  return plugin.cache.getTasks()
    .filter(t => !excluded.some(f => t.taskLocation.path.startsWith(f)))
    .filter(t => !t.isDone && t.due.moment && t.due.moment.isBetween(start, end, null, '[]'))
    .sort((a,b) => a.due.moment.valueOf() - b.due.moment.valueOf())
    .map(t => t.due.formatAsDate() + ' ' + t.description)
    .join('\n') || 'No tasks due this week';
})()"
```
