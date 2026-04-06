Agent Skills for use with the [Obsidian Tasks](https://github.com/obsidian-tasks-group/obsidian-tasks) plugin.

These skills follow the [Agent Skills specification](https://agentskills.io/specification) so they can be used by any skills-compatible agent, including Claude Code and Codex CLI.

## Installation

### Marketplace

```
/plugin marketplace add lexbritvin/obsidian-tasks-skills
/plugin install obsidian-tasks@obsidian-tasks-skills
```

### Manually

#### Claude Code

Add the contents of this repo to a `/.claude` folder in the root of your Obsidian vault (or whichever folder you're using with Claude Code). See more in the [official Claude Skills documentation](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview).

#### Codex CLI

Copy the `skills/` directory into your Codex skills path (typically `~/.codex/skills`). See the [Agent Skills specification](https://agentskills.io/specification) for the standard skill format.

#### OpenCode

Clone the entire repo into the OpenCode skills directory (`~/.opencode/skills/`):

```
git clone https://github.com/lexbritvin/obsidian-tasks-skills.git ~/.opencode/skills/obsidian-tasks-skills
```

Do not copy only the inner `skills/` folder — clone the full repo so the directory structure is `~/.opencode/skills/obsidian-tasks-skills/skills/<skill-name>/SKILL.md`.

OpenCode auto-discovers all `SKILL.md` files under `~/.opencode/skills/`. No changes to `opencode.json` or any config file are needed. Skills become available after restarting OpenCode.

## Requirements

- [Obsidian Tasks plugin](https://github.com/obsidian-tasks-group/obsidian-tasks) installed in your vault
- **manage-tasks only:** [Obsidian CLI](https://help.obsidian.md/cli) (Obsidian 1.12+, Settings > General > CLI) and Obsidian running

## Skills

| Skill | Description |
| --- | --- |
| [create-task](https://github.com/lexbritvin/obsidian-tasks-skills/blob/main/skills/create-task) | Create and edit task lines with emoji-based dates, priorities, recurrence, dependencies, and statuses |
| [create-query](https://github.com/lexbritvin/obsidian-tasks-skills/blob/main/skills/create-query) | Build `tasks` query blocks with filters, sorting, grouping, and layout options |
| [manage-tasks](https://github.com/lexbritvin/obsidian-tasks-skills/blob/main/skills/manage-tasks) | Read, filter, and update tasks via the [Obsidian CLI](https://help.obsidian.md/cli) and Tasks plugin API |
