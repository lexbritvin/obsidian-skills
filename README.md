# Obsidian Skills for Claude

Agent Skills that teach Claude to write [Obsidian](https://obsidian.md) plugin syntax fluently — queries, tasks, and charts directly inside your notes. Compatible with Claude Code, Codex CLI, OpenCode, and any other [skills-compatible agent](https://agentskills.io/specification).

| Skill | For plugin | What it does |
| --- | --- | --- |
| [`obsidian-dataview`](skills/obsidian-dataview) | [Dataview](https://github.com/blacksmithgu/obsidian-dataview) | Queries and dashboards over your notes — find files by tag, list overdue tasks, build tables from frontmatter. |
| [`obsidian-charts`](skills/obsidian-charts) | [Charts](https://github.com/phibr0/obsidian-charts) | Bar, line, pie, radar, and scatter charts inside notes. Pairs with Dataview for live data. |
| [`obsidian-meta-bind`](skills/obsidian-meta-bind) | [Meta Bind](https://github.com/mProjectsCode/obsidian-meta-bind-plugin) | Buttons, input fields, view fields, and embeds. Schema, JS Engine integration, validation pitfalls. |
| [`obsidian-tasks-syntax`](skills/obsidian-tasks-syntax) | [Tasks](https://github.com/obsidian-tasks-group/obsidian-tasks) | Task lines with due dates, priorities, recurrence, and dependencies. |
| [`obsidian-tasks-query`](skills/obsidian-tasks-query) | [Tasks](https://github.com/obsidian-tasks-group/obsidian-tasks) | Query blocks to surface tasks across your vault — overdue, by project, by tag. |
| [`obsidian-tasks-cli`](skills/obsidian-tasks-cli) | [Tasks](https://github.com/obsidian-tasks-group/obsidian-tasks) | Read and update tasks from the command line via the [Obsidian CLI](https://help.obsidian.md/cli). |

Each skill ships with detailed reference files — open the linked folder for full coverage.

## Install

### Claude Code

```
/plugin marketplace add lexbritvin/obsidian-skills
/plugin install obsidian-dataview@lexbritvin-obsidian-skills
/plugin install obsidian-charts@lexbritvin-obsidian-skills
/plugin install obsidian-meta-bind@lexbritvin-obsidian-skills
/plugin install obsidian-tasks-syntax@lexbritvin-obsidian-skills
/plugin install obsidian-tasks-query@lexbritvin-obsidian-skills
/plugin install obsidian-tasks-cli@lexbritvin-obsidian-skills
```

Each line installs one skill. Pick whichever you need.

### Codex CLI

Copy the relevant `skills/<skill-name>/` directories into `~/.codex/skills/`.

### OpenCode

```
git clone https://github.com/lexbritvin/obsidian-skills.git ~/.opencode/skills/obsidian-skills
```

### Manually in a vault

Copy the relevant `skills/<skill-name>/` directories into `<vault>/.claude/skills/`.

## Versioning

[Semver](https://semver.org/) at two levels — marketplace `metadata.version` and per-plugin `version`.

- **MAJOR** — plugin renamed, removed, or install path broken (existing installs fail after `/plugin update`).
- **MINOR** — new plugin or skill added; existing installs keep working.
- **PATCH** — content fixes inside an existing skill; no structural change.

## License

[MIT](LICENSE)
