# Shell Expert Agent

OpenCode sub-agent that handles **all** shell command generation and execution through a stronger model. The primary model never uses the bash tool directly — every terminal task is delegated.

See [`agents.md`](agents.md) for the full workflow, 10 category-by-category guidance, security boundaries, and system prompt.

## Terminal Task Categories Covered

| Category | Examples |
|----------|----------|
| File Operations | cp, mv, rm, mkdir, chmod, symlinks |
| Text Processing | grep, sed, awk, jq, CSV, JSON |
| Package Management | npm, pip, apt, cargo, go mod |
| Git Workflows | merge, rebase, push, conflict resolution |
| System Diagnostics | ps, df, free, netstat, systemctl |
| Environment Setup | PATH, venvs, nvm, shell rc files |
| Build & Compilation | make, cmake, gcc, tsc, cargo |
| Container Operations | docker, podman, compose |
| Network Operations | curl, wget, APIs, DNS, SSL |
| Data Pipelines | pipes, xargs, parallel, tee |

## Quick Install

**Two pieces are required** — the agent definition in `opencode.json` and the companion skill in `.opencode/skills/`.

### 1. Install the companion skill

Copy the skill content from [`skills.md`](skills.md) to `.opencode/skills/shell-expert/SKILL.md` in your project. This skill auto-matches on shell/terminal prompts and instructs the primary model to delegate to the `shell-expert` sub-agent. **Without this skill, the primary model won't know the agent exists.**

```bash
mkdir -p .opencode/skills/shell-expert
cp skills.md .opencode/skills/shell-expert/SKILL.md
```

### 2. Add the agent definition

Add this to your project's `opencode.json`:

```json
{
  "agent": {
    "shell-expert": {
      "model": "openai/gpt-5.5",
      "description": "Handles all shell command generation and execution. Primary model must delegate all terminal tasks to this sub-agent.",
      "mode": "subagent",
      "temperature": 0.2,
      "steps": 20
    }
  }
}
```

### 3. Add delegation instructions to `AGENTS.md`

```markdown
## Shell Command Delegation

**ALL shell commands must be delegated to the `shell-expert` sub-agent.** Never use the bash tool directly.

`Task(description="<what needs to happen>", prompt="<full instructions>", subagent_type="shell-expert")`

This applies to every shell invocation — simple or complex, read-only or mutating.
```
