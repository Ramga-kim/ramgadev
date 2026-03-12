# OpenCode adapter

## Discovery and invocation

- OpenCode can discover this skill from `.opencode/skills/`, `.claude/skills/`, or `.agents/skills/`.
- Prefer explicit prompting such as `Use the monthly-expense-draft skill for this expense upload task.`

## Capability mapping

- `orchestrator`: current OpenCode agent thread.
- `receipt-worker`: subtask-capable worker agent when parallel OCR helps.
- `browser-worker`: the thread or subtask that has browser or MCP access.

## Recommended orchestration

1. Read `workflow.md`, config, and this adapter.
2. Resolve the target folder and normalize receipt filenames.
3. If OpenCode subtasks are available, fan out per-receipt OCR and optionally warm the browser in parallel; otherwise stay serial.
4. Merge sorted receipt JSON.
5. Create the attachment zip.
6. Fill the Goworks form and upload the zip through the available browser or MCP tools.
7. Return the verification state and pause for user review and submit.

## OpenCode-specific notes

- Keep provider or model overrides in OpenCode config or `skill-config.yaml`, not in `SKILL.md`.
- If you package a matching custom command later, make it a thin wrapper that simply asks OpenCode to load this skill.
- If the active agent lacks browser automation, stop and report the missing MCP or browser capability.
