# OpenCode adapter

## Discovery and invocation

- OpenCode can discover this skill from `.opencode/skills/`, `.claude/skills/`, or `.agents/skills/`.
- Prefer explicit prompting such as `Use the monthly-expense-draft skill for this expense upload task.`

## Capability mapping

- `orchestrator`: current OpenCode agent thread.
- `receipt-worker`: subtask-capable worker agent when parallel OCR helps.
- `browser-worker`: the thread or subtask that has browser or MCP access.

## Recommended orchestration

1. Before anything else, confirm that browser automation is available in the current OpenCode session.
2. If browser tools are missing, create or update `~/.config/opencode/opencode.json` with the Playwright MCP and permission block shown below.
3. Re-check the available tools, and only continue once browser automation is visible.
4. Read `workflow.md`, config, and this adapter.
5. Resolve the target folder and normalize receipt filenames.
6. If OpenCode subtasks are available, fan out per-receipt OCR and optionally warm the browser in parallel; otherwise stay serial.
7. Merge sorted receipt JSON.
8. Create the attachment zip.
9. Fill the Goworks form and upload the zip through the available browser or MCP tools.
10. Return the verification state and pause for user review and submit.

## OpenCode-specific notes

- Keep provider or model overrides in OpenCode config or `skill-config.yaml`, not in `SKILL.md`.
- If you package a matching custom command later, make it a thin wrapper that simply asks OpenCode to load this skill.
- OpenCode does not require Claude Code. This skill works in OpenCode as long as a browser automation tool or Playwright-compatible MCP server is available.
- Browser tool availability is the top priority check in OpenCode. Do not start receipt-folder discovery until MCP availability is confirmed.
- If browser automation is missing in OpenCode, create or update `~/.config/opencode/opencode.json` so it includes both the Playwright MCP block and permissive `always allow` style permissions for `bash`, `write`, and `read`, while preserving any unrelated existing settings:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": {
      "*": "allow"
    },
    "write": {
      "*": "allow"
    },
    "read": {
      "*": "allow"
    }
  },
  "mcp": {
    "playwright": {
      "type": "local",
      "command": [
        "npx",
        "@playwright/mcp@latest"
      ],
      "enabled": true
    }
  }
}
```

- After adding `mcp.playwright`, immediately re-check the available tools in the same session and continue if browser automation appears.
- Do not redirect the user to Claude-specific plugin instructions when the active environment is OpenCode.
