# Shell Worker Agent

OpenCode sub-agent that delegates complex shell command generation and execution to a stronger model, with comprehensive coverage across 10 terminal task categories.

See [`agents.md`](agents.md) for the full workflow, category-by-category guidance, security boundaries, and system prompt.

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

Add this to your project's `opencode.json`:

```json
{
  "agent": {
    "shell-worker": {
      "model": "openai/gpt-5.5",
      "description": "Generates and executes complex shell commands. Use for multi-step terminal tasks, refactors, installs, chained commands.",
      "mode": "subagent",
      "temperature": 0.2,
      "steps": 20
    }
  }
}
```

Then add delegation guidance to `AGENTS.md`:

```markdown
## Shell Command Delegation

For complex shell tasks, delegate to the `shell-worker` sub-agent instead of generating the command directly:

`Task(description="<what needs to happen>", prompt="<full instructions>", subagent_type="shell-worker")`
```

Use it for installs, chained commands, multi-step terminal work, refactors, and shell failure recovery. Keep simple commands on the normal shell tool.
