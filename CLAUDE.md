# ai-marketplace

Personal Claude Code plugin marketplace.

## Adding a new skill to an existing plugin

1. Create `plugins/<plugin>/skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`) and instructions.

## Adding a new plugin

1. Create `plugins/<plugin>/.claude-plugin/plugin.json` with `name`, `description`, `version`, `author`.
2. Create at least one skill under `plugins/<plugin>/skills/`.
3. Add an entry to `.claude-plugin/marketplace.json` → `plugins[]`.
