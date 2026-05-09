---
name: obsidian-tasks-syntax
description: Creates and edits Obsidian Tasks plugin task lines with emoji-based dates, priorities, recurrence, dependencies, and statuses. Use when the user asks to add a task, create a todo, set a due date, change priority, add recurrence, make a recurring reminder, edit a checkbox line, or anything involving task emoji signifiers (📅⏳🛫🔁). Also use when the user says "remind me to", "add a todo", "new checkbox", or wants to understand the task format.
compatibility: Designed for Obsidian vaults with the Tasks plugin installed.
---

# Creating Tasks

## Task Line Format

```markdown
- [ ] Task description 🔼 🔁 every week 🛫 2025-01-01 ⏳ 2025-01-05 📅 2025-01-10
```

Order matters: description first, then priority, then recurrence, then dates. The Tasks plugin parses emoji signifiers by position — putting them out of order can cause misreads. Dates are always `YYYY-MM-DD` — natural language like "tomorrow" is not recognized on task lines (it only works in query blocks).

## Dates

| Emoji | Name | When to use |
|-------|------|-------------|
| 📅 | Due date | The deadline — when the task must be finished |
| ⏳ | Scheduled date | When you plan to work on it (hides task from views until this date) |
| 🛫 | Start date | Earliest date the task becomes actionable (blocks it before then) |
| ➕ | Created date | Auto-set by Tasks plugin, or add manually for tracking |
| ✅ | Done date | Auto-set when completing a task |
| ❌ | Cancelled date | Auto-set when cancelling |

**Choosing between dates:** Use 📅 due for deadlines. Use ⏳ scheduled to defer a task you don't want cluttering today's view. Use 🛫 start for tasks that literally can't begin before a date (e.g., waiting for access). Most tasks only need 📅 due.

## Priority (decreasing)

🔺 Highest · ⏫ High · 🔼 Medium · _(none = Normal)_ · 🔽 Low · ⏬ Lowest

Normal sits between Low and Medium — tasks without a priority emoji are higher than Low. Only add priority when it meaningfully differs from normal.

## Recurrence

🔁 followed by a rule starting with `every`. Requires at least one date.

```markdown
- [ ] Weekly review 🔁 every Friday 📅 2025-01-10
- [ ] Pay rent 🔁 every month on the 1st 📅 2025-02-01
- [ ] Water plants 🔁 every 3 days when done 📅 2025-01-10
```

Patterns: `every day`, `every weekday`, `every week on Monday, Wednesday`, `every 2 weeks`, `every month on the 1st`, `every month on the last Friday`, `every year`, `every January on the 15th`.

**`when done`** — add this to base the next occurrence on the completion date instead of the original date. Use it for flexible-interval tasks (like watering plants) where the gap matters more than a fixed schedule.

## Dependencies

| Emoji | Field |
|-------|-------|
| 🆔 | id — identifies this task |
| ⛔ | dependsOn — blocks until the referenced id is done |

```markdown
- [ ] Write draft 🆔 draft1
- [ ] Review draft ⛔ draft1
```

A blocked task (⛔) won't appear in "not blocked" queries until its dependency is completed.

## Statuses

| Checkbox | Type | Query match |
|----------|------|-------------|
| `[ ]` | TODO | `not done` |
| `[/]` | IN_PROGRESS | `not done` |
| `[x]` | DONE | `done` |
| `[-]` | CANCELLED | `done` |

Custom statuses (ON_HOLD, NON_TASK, etc.) depend on theme/CSS configuration.

## Examples

```markdown
- [ ] Buy groceries 📅 2025-04-10
- [ ] Weekly review 🔼 🔁 every Friday 📅 2025-04-11
- [ ] Start writing article 🛫 2025-04-07 📅 2025-04-14
- [ ] Research topic ⏳ 2025-04-08 📅 2025-04-12
- [/] Working on presentation 📅 2025-04-10
- [ ] Prepare slides 🆔 slides1 📅 2025-04-09
- [ ] Send presentation ⛔ slides1 📅 2025-04-10
- [ ] Water plants 🔁 every 3 days when done 📅 2025-04-08
- [ ] Pay rent 🔺 🔁 every month on the 1st 📅 2025-05-01
```
