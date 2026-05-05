# Agent Instructions

## Skill creation workflow

When the user asks to create a skill, follow this requirements gathering workflow before writing anything:

1. **Goal** — What problem does the skill solve? Who is it for?
2. **Inputs** — What does the user provide? What context does the skill need?
3. **Outputs** — What does the skill produce? What does success look like?
4. **Steps** — Walk through the skill step-by-step. What tools/commands are needed at each step?
5. **Edge cases** — What can go wrong? What are the boundaries?

Ask clarifying questions iteratively until all details are clear. Only then write the skill.

## Agent creation workflow

When the user asks to create an agent, follow this requirements gathering workflow before writing anything:

1. **Goal** — What problem does the agent solve? Who is it for?
2. **Inputs** — What does the user provide? What context does the agent need?
3. **Outputs** — What does the agent produce? What does success look like?
4. **Steps** — Walk through the agent step-by-step. What tools are needed at each step?
5. **Edge cases** — What can go wrong? What are the boundaries?

Ask clarifying questions iteratively until all details are clear. Only then write the agent.

## Plugin creation workflow

When the user asks to create a plugin, follow this requirements gathering workflow before writing anything:

1. **Goal** — What problem does the plugin solve? Who is it for?
2. **Inputs** — What does the user provide? What context does the plugin need?
3. **Outputs** — What does the plugin produce? What does success look like?
4. **Steps** — Walk through the plugin step-by-step. What tools are needed at each step?
5. **Edge cases** — What can go wrong? What are the boundaries?

Ask clarifying questions iteratively until all details are clear. Only then write the plugin.

## Publishing

### Skills
- Published to `skills/<name>/`
- Each skill must include `skills.md` with the full explanation, structure, and workflow
- After publishing, update the Skills table in `README.md` with a link to the new `skills.md`

### Agents
- Published to `agents/<name>/`
- Each agent must include `agents.md` with the full explanation, structure, and workflow
- After publishing, update the Agents table in `README.md` with a link to the new `agents.md`

### Plugins
- Published to `plugins/<name>/`
- Each plugin must include `plugins.md` with the full explanation, structure, and workflow
- After publishing, update the Plugins table in `README.md` with a link to the new `plugins.md`

Do not invent placeholder items — only create skills, agents, or plugins when the requirements are fully understood.
