# claude-skills

A Claude Code plugin marketplace. Each plugin contains one skill.

## Install a plugin

1. Add the marketplace in Claude Code:

   ```
   /plugin marketplace add pparitoshh/claude-skills
   ```

2. Install a plugin:

   ```
   /plugin install brain-worker@claude-skills
   ```

## Plugins

| Plugin | Skill | Description |
|---|---|---|
| `brain-worker` | `/brain-worker:brain-worker` | Set up a Herdr brain + worker pair. |

## Add a new skill

1. Create `plugins/<name>/skills/<name>/SKILL.md`.
2. Create `plugins/<name>/.claude-plugin/plugin.json`.
3. Add the plugin to `.claude-plugin/marketplace.json`.
4. Run `claude plugin validate .` to check the repo.
