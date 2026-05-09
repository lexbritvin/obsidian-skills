<div align="center">

# Obsidian Skills Pack

**Make your agent fluent in Obsidian.**

Nine skills covering plugin syntax, graph view tuning, vault auditing, and heatmap visualizations. Compatible with Claude Code, Codex CLI, OpenCode, and any [skills-compatible agent](https://agentskills.io/specification).

[![License](https://img.shields.io/github/license/lexbritvin/obsidian-skills-pack)](LICENSE)
![Skills](https://img.shields.io/badge/skills-9-blue)
[![Agent Skills spec](https://img.shields.io/badge/agentskills.io-compatible-7c3aed)](https://agentskills.io/specification)

</div>

---

## What's inside

### Plugin syntax — author plugin features inside your notes

- **[`obsidian-dataview`](skills/obsidian-dataview)** — for [Dataview](https://github.com/blacksmithgu/obsidian-dataview). Queries and dashboards over your notes.
- **[`obsidian-charts`](skills/obsidian-charts)** — for [Charts](https://github.com/phibr0/obsidian-charts). Bar, line, pie, radar, scatter; pairs with Dataview for live data.
- **[`obsidian-meta-bind`](skills/obsidian-meta-bind)** — for [Meta Bind](https://github.com/mProjectsCode/obsidian-meta-bind-plugin). Buttons, input/view fields, embeds.
- **[`obsidian-tasks-syntax`](skills/obsidian-tasks-syntax)** — for [Tasks](https://github.com/obsidian-tasks-group/obsidian-tasks). Task lines with due dates, priorities, recurrence, dependencies.
- **[`obsidian-tasks-query`](skills/obsidian-tasks-query)** — for Tasks. Query blocks to surface tasks across the vault.
- **[`obsidian-tasks-cli`](skills/obsidian-tasks-cli)** — for Tasks. Read and update tasks from the [Obsidian CLI](https://help.obsidian.md/cli).
- **[`obsidian-heatmap`](skills/obsidian-heatmap)** — for [Heatmap Calendar](https://github.com/Richardsl/heatmap-calendar-obsidian) plus custom DataviewJS. GitHub-style year heatmaps and ad-hoc layouts.

### Vault management — tune and audit the vault itself

- **[`obsidian-graph`](skills/obsidian-graph)** — Graph view tuning: color groups, filters, forces, palette. Full `graph.json` schema and pitfalls.
- **[`obsidian-graph-audit`](skills/obsidian-graph-audit)** — diagnostic patterns for the graph: orphans, hubs, name collisions, hidden cluster patterns.

Each skill ships with detailed reference files — open the linked folder for full coverage.

## Install

In Claude Code, add the marketplace and pick one skill to start:

```
/plugin marketplace add lexbritvin/obsidian-skills-pack
/plugin install obsidian-dataview@obsidian-skills-pack
```

Repeat the second line for each skill you want.

<details>
<summary>Install all nine skills</summary>

```
/plugin install obsidian-dataview@obsidian-skills-pack
/plugin install obsidian-charts@obsidian-skills-pack
/plugin install obsidian-meta-bind@obsidian-skills-pack
/plugin install obsidian-tasks-syntax@obsidian-skills-pack
/plugin install obsidian-tasks-query@obsidian-skills-pack
/plugin install obsidian-tasks-cli@obsidian-skills-pack
/plugin install obsidian-heatmap@obsidian-skills-pack
/plugin install obsidian-graph@obsidian-skills-pack
/plugin install obsidian-graph-audit@obsidian-skills-pack
```

</details>

### Other agents

- **Codex CLI** — copy `skills/<skill-name>/` into `~/.codex/skills/`.
- **OpenCode** — `git clone https://github.com/lexbritvin/obsidian-skills-pack.git ~/.opencode/skills/obsidian-skills-pack`
- **Manually in a vault** — copy `skills/<skill-name>/` into `<vault>/.claude/skills/`.

## License

[MIT](LICENSE)
