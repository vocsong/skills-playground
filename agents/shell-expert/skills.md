---
name: shell-expert
description: Delegates all shell/terminal command execution to the shell-expert sub-agent backed by a stronger model. Use when any shell command, CLI tool, or terminal task is needed — from simple ls/pwd to complex multi-step pipelines.
---

## What I Do

I intercept all shell/terminal command requests and delegate them to the `shell-expert` OpenCode sub-agent. The primary model never uses the bash tool directly — every terminal task goes through the stronger model-backed sub-agent.

## When to Use Me

Load this skill when the user asks for **any** shell, terminal, or command execution task. This includes:

- CLI tool invocations: `npm`, `pip`, `git`, `docker`, `curl`, `wget`, `make`, `cargo`, `go`, `kubectl`, `helm`, `brew`, `apt`, `yarn`, `pnpm`, `poetry`
- Action words: `run`, `execute`, `install`, `build`, `test`, `deploy`, `shell`, `bash`, `terminal`, `command`, `cmd`, `cli`, `script`
- Simple commands: `ls`, `pwd`, `git status`, `echo`, `cat`, `date`
- Complex workflows: multi-step pipelines, chained commands with `&&`, error recovery loops

## Workflow

### Step 1: Verify Configuration

Check that `opencode.json` has the `shell-expert` agent defined. If missing, instruct the user to add:

```json
{
  "agent": {
    "shell-expert": {
      "model": "openai/gpt-5.5",
      "description": "Handles all shell command generation and execution. Primary model must delegate all terminal tasks to this sub-agent.",
      "mode": "subagent",
      "temperature": 0.2,
      "steps": 20,
      "permission": {
        "bash": "allow"
      }
    }
  }
}
```

### Step 2: Delegate EVERY Shell Task

**Critical rule: Never use the bash tool directly. Always delegate.**

```
Task(description="<what needs to happen>", prompt="<full instructions>", subagent_type="shell-expert")
```

This applies to every shell invocation — simple or complex, read-only or mutating.

### Step 3: Integrate the Result

Read the shell expert's structured summary and integrate findings into the main task.

## Hard Rules

- **Never generate shell commands directly.** If about to use bash, stop and delegate.
- **All shell tasks qualify.** `ls`, `pwd`, `git status`, `npm install`, `docker build` — everything goes through shell-expert.
- **File read/search/edit tools are exempt.** Only terminal/shell operations go through shell-expert.
