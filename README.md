# Symphony Skill

AI agent skill for [Symphony](https://symphony.fr), a personalized meal delivery service in Paris.

## Install

**Claude Code:**
```
mkdir -p .claude/skills/symphony && curl -o .claude/skills/symphony/SKILL.md https://raw.githubusercontent.com/symphony-fr/symphony-skill/main/SKILL.md
```

**OpenClaw:**
```
openclaw skills install git:symphony-fr/symphony-skill
```

**Any agent:** point it at `https://raw.githubusercontent.com/symphony-fr/symphony-skill/main/SKILL.md`

## API Key

Get your API key at https://symphony.fr/en/my-account/account/api-keys

## Format

SKILL.md follows the [Agent Skills](https://agentskills.io/) open standard — compatible with Claude Code, OpenClaw, Codex CLI, Cursor, Gemini CLI, and 20+ agents.
