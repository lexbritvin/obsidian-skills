---
name: manage-tasks
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
    "file": "10 - Projects/Shopping.md",
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

All eval code must be wrapped in an IIFE because `obsidian eval` doesn't support top-level return:

```bash
obsidian eval code="(()=>{
  const tasks = app.plugins.plugins['obsidian-tasks-plugin'].cache.getTasks();
  // filter, map, return
})()"
```

### Task Object Properties

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

### TasksDate

A TasksDate object is always present, but the underlying date may be empty. Check `task.due.moment` — it's `null` when no date is set:

```javascript
task.due.moment                          // Moment.js object or null
task.due.moment.format('YYYY-MM-DD')     // "2025-04-10"
task.due.moment.isBefore(window.moment()) // overdue check
task.due.formatAsDate()                  // formatted string
task.due.category.groupText              // "Overdue", "Today", "Future", "Undated"
```

### Recipes

**Task count by status:**

```bash
obsidian eval code="(()=>{
  const tasks = app.plugins.plugins['obsidian-tasks-plugin'].cache.getTasks();
  const counts = {};
  tasks.forEach(t => { const s = t.isDone ? 'done' : 'todo'; counts[s] = (counts[s]||0)+1; });
  return JSON.stringify(counts);
})()"
```

**Overdue tasks:**

```bash
obsidian eval code="(()=>{
  const tasks = app.plugins.plugins['obsidian-tasks-plugin'].cache.getTasks();
  const today = window.moment().startOf('day');
  return tasks
    .filter(t => !t.isDone && t.due.moment && t.due.moment.isBefore(today))
    .map(t => t.description + ' 📅 ' + t.due.formatAsDate() + ' (' + t.taskLocation.path + ')')
    .join('\n') || 'No overdue tasks';
})()"
```

**High priority tasks:**

```bash
obsidian eval code="(()=>{
  const tasks = app.plugins.plugins['obsidian-tasks-plugin'].cache.getTasks();
  return tasks
    .filter(t => !t.isDone && ['Highest','High'].includes(t.priorityName))
    .map(t => t.priorityName + ': ' + t.description)
    .join('\n') || 'No high priority tasks';
})()"
```

**Tasks by folder:**

```bash
obsidian eval code="(()=>{
  const tasks = app.plugins.plugins['obsidian-tasks-plugin'].cache.getTasks();
  const groups = {};
  tasks.filter(t => !t.isDone).forEach(t => {
    const folder = t.taskLocation.path.split('/').slice(0,-1).join('/') || '/';
    groups[folder] = (groups[folder]||0)+1;
  });
  return Object.entries(groups).sort((a,b)=>b[1]-a[1]).map(([f,c])=>c+' '+f).join('\n');
})()"
```

**Tasks due this week:**

```bash
obsidian eval code="(()=>{
  const tasks = app.plugins.plugins['obsidian-tasks-plugin'].cache.getTasks();
  const start = window.moment().startOf('week');
  const end = window.moment().endOf('week');
  return tasks
    .filter(t => !t.isDone && t.due.moment && t.due.moment.isBetween(start, end, null, '[]'))
    .sort((a,b) => a.due.moment.valueOf() - b.due.moment.valueOf())
    .map(t => t.due.formatAsDate() + ' ' + t.description)
    .join('\n') || 'No tasks due this week';
})()"
```
