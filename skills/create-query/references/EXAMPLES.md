# Tasks Query Examples

## Basic Queries

### Due today

````markdown
```tasks
not done
due today
```
````

### Due this week

````markdown
```tasks
not done
due this week
sort by due
```
````

### Overdue tasks

````markdown
```tasks
not done
due before today
sort by due
```
````

### All open tasks sorted by priority

````markdown
```tasks
not done
sort by priority
sort by due
```
````

## Filtering by Location

### Tasks from a specific folder

````markdown
```tasks
not done
folder includes 10 - Projects
sort by due
```
````

### Tasks from the current file

````markdown
```tasks
not done
path includes {{query.file.path}}
```
````

### Tasks under a specific heading

````markdown
```tasks
not done
heading includes TODO
```
````

## Grouping

### Group by priority

````markdown
```tasks
not done
group by priority
sort by due
```
````

### Group by file

````markdown
```tasks
not done
due this week
group by filename
sort by due
```
````

### Group by due date category (overdue/today/future/undated)

````markdown
```tasks
not done
group by function task.due.category.groupText
group by due
```
````

## Boolean Filters

### Tasks due soon OR high priority

````markdown
```tasks
not done
(due before in one week) OR (priority is above medium)
sort by urgency
```
````

### Exclude specific folders

````markdown
```tasks
not done
(folder does not include Templates) AND (folder does not include Attachments)
sort by due
```
````

### In-progress or on-hold tasks

````markdown
```tasks
(status.type is IN_PROGRESS) OR (status.type is ON_HOLD)
sort by status.type
sort by due
```
````

## Dates & Scheduling

### Upcoming week with no start date blocking

````markdown
```tasks
not done
due after yesterday
due before in 8 days
(no start date) OR (starts before tomorrow)
sort by due
```
````

### Tasks with no due date (backlog review)

````markdown
```tasks
not done
no due date
no scheduled date
sort by path
limit 20
```
````

### Completed this week

````markdown
```tasks
done
done this week
sort by done reverse
```
````

## Recurrence

### All recurring tasks

````markdown
```tasks
not done
is recurring
sort by due
```
````

## Dependencies

### Actionable tasks (not blocked)

````markdown
```tasks
not done
is not blocked
sort by urgency
```
````

### Blocking tasks (do these first)

````markdown
```tasks
not done
is blocking
sort by due
```
````

## Layout & Display

### Compact view

````markdown
```tasks
not done
due this week
short mode
hide backlink
hide task count
hide edit button
```
````

### Show task tree (parent/child)

````markdown
```tasks
not done
folder includes 10 - Projects
show tree
hide backlink
```
````

### Debug a query

Add `explain` to any query to see how Tasks interprets it:

````markdown
```tasks
not done
due this week
(priority is high) OR (priority is highest)
explain
```
````

## Task Line Examples

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
