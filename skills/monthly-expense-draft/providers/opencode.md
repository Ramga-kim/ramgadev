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
5. Start browser prewarm immediately: open the Goworks form, wait for load, and confirm login state even if the receipt root path is still unknown.
6. In parallel, resolve the target receipt folder and normalize filenames.
7. If OpenCode subtasks are available, fan out per-receipt OCR while the browser worker keeps the form ready; otherwise stay serial.
8. Merge sorted receipt JSON.
9. Create the attachment zip and copy it into an allowed upload path before the first upload attempt.
10. Fill the Goworks form and upload the zip through the already prepared browser or MCP tools, using lightweight checks instead of repeated full snapshots where possible.
11. Return the verification state and pause for user review and submit.

## OpenCode-specific notes

- Keep provider or model overrides in OpenCode config or `skill-config.yaml`, not in `SKILL.md`.
- If you package a matching custom command later, make it a thin wrapper that simply asks OpenCode to load this skill.
- OpenCode does not require Claude Code. This skill works in OpenCode as long as a browser automation tool or Playwright-compatible MCP server is available.
- Browser tool availability is the top priority check in OpenCode. Do not start receipt-folder discovery until MCP availability is confirmed.
- Once MCP availability is confirmed, do not leave the browser worker idle while waiting for the receipt root answer. Preload the Goworks form first and overlap that work with the folder question.
- If browser automation is missing in OpenCode, create or update `~/.config/opencode/opencode.json` so it includes both the Playwright MCP block and permissive `always allow` style permissions for `bash`, `write`, `read`, and `external_directory`, while preserving any unrelated existing settings:

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
    },
    "external_directory": {
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
