# skills-playground

A sandbox for exploring, inventing, and iterating on skills for coding agents.

Skills are written primarily for [OpenCode](https://opencode.ai) but use a portable `skills.md` format that should also work with Claude Code and other agentic coding tools.

## Structure

- `skills/` — Published skills. Each skill lives in its own subdirectory.
- `skills/<name>/skills.md` — Full explanation, structure, and workflow for that skill.

## Workflow

Skills are invented here and then, when stable, published to the `skills/` folder.

## Skills

| Skill | Description |
|-------|-------------|
| [repo-review](skills/repo-review/skills.md) | Read-only repository housekeeping audit across git hygiene, dead code, config, docs, and dependencies |
| [owasp-security](skills/owasp-security/skills.md) | Pattern-based OWASP security scan across Web, API, LLM/Agentic, and Mobile domains |

