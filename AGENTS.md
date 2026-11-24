# Editing opencode configuration

## Files

- **`opencode.jsonc`** — Main configuration: provider settings (models, thinking budgets), MCP servers. Schema: <https://opencode.ai/config.json>
- **`tui.jsonc`** — TUI settings (attention, notifications, sound, volume). Schema: <https://opencode.ai/tui.jsonc>
- **`agents/`** — Custom subagent definitions (YAML frontmatter + instructions). Each `.md` file defines one agent with `mode: subagent`, `tools`, `permission`, `temperature`.
- **`skills/`** — Skill definitions. Each subdirectory has a `SKILL.md` with shared instructions.
- **`commands/`** — Custom `/command` handlers. Each `.md` file maps a command name to instructions for the primary agent.

## Conventions

### Adding a new agent
1. Create `agents/<name>.md` with YAML frontmatter:
   - `description` (shown in tool lists)
   - `mode: subagent`
   - `temperature` (0.1 for deterministic critic, 0.2 for creative actor, higher for open-ended)
   - `tools:` map (read/write/edit/bash/grep/glob/task — all `true`/`false`)
   - `permission:` map (`edit: deny` or `allow`, `bash: allow` for read-only agents)
2. Write the agent's instructions in the body.

### Adding a new skill
1. Create `skills/<name>/SKILL.md`.
2. List it in the `available_skills` system prompt of the model (this controls when it auto-loads).
3. In `SKILL.md`, include a `name` and `description` in YAML frontmatter, then the instructions.

### Adding a new command
1. Create `commands/<name>.md` with YAML frontmatter: `description`.
2. The body is a prompt that receives `$ARGUMENTS`.
3. Register in `opencode.jsonc` under the `commands` key if needed.

### Permission model
- Read-only agents: `edit: deny`, `write: false`, `task: false`, `bash: allow` (safe because edit/write are blocked).
- Writing agents: `edit: true`, `write: true`, `bash: allow`, `task: true` (can spawn sub-agents).
- Never give an agent `bash: allow` without `edit: deny` if it should be read-only.

## Package dependencies
- Only `@opencode-ai/plugin` (in `package.json`).
- `node_modules/` is gitignored.

## Tips
- Use `$schema` URLs for IDE autocomplete in JSONC files.
- The `permission` map overrides the `tools` map for bash/edit — `edit: deny` blocks all writes even if `tools.edit: true`.