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
6. In parallel, resolve the target receipt folder using Windows-safe tools only. Do not use Unix-style file commands.
7. Receipt extraction must be delegated to a receipt worker or subtask. Do not try Python/Node-based parsing or custom OCR scripts in the main session.
8. Merge sorted receipt JSON and stop re-evaluating duplicates once the receipt worker returns a validated result.
9. Create the attachment zip in `business.upload_staging_path` semantics before the first upload attempt, and keep all attachment work inside the allowed root.
10. Fill the Goworks form using the fixed table structure and batched browser interactions by default. Do not rediscover the table structure, inspect internal calculation functions, or probe hidden metadata on normal runs.
11. Do not manually set readonly or auto-calculated summary fields. Verify them only after the main row inputs are complete.
12. Use per-field fallbacks only after a concrete failure in the batched strategy.
13. Return the verification state and pause for user review and submit.

## OpenCode-specific notes

- Keep provider or model overrides in OpenCode config or `skill-config.yaml`, not in `SKILL.md`.
- If you package a matching custom command later, make it a thin wrapper that simply asks OpenCode to load this skill.
- OpenCode does not require Claude Code. This skill works in OpenCode as long as a browser automation tool or Playwright-compatible MCP server is available.
- Browser tool availability is the top priority check in OpenCode. Do not start receipt-folder discovery until MCP availability is confirmed.
- Once MCP availability is confirmed, do not leave the browser worker idle while waiting for the receipt root answer. Preload the Goworks form first and overlap that work with the folder question.
- This workflow is Windows-only. Do not use `ls`, `find`, `mv`, `cp`, or other Unix-style file commands as the default strategy.
- In OpenCode, PowerShell inside bash must start with the validated single-quoted outer command pattern. Do not experiment with alternate quoting first.
- Receipt extraction is a multimodal LLM task handled by the receipt worker. Do not introduce Python, Node.js, or custom script-based parsing as a workaround.
- Do not rediscover the fixed Goworks detail-table structure on normal runs. Use the documented fixed structure and only fall back to discovery after a concrete failure.
- Do not inspect `fnCalculation`, `fnTotalCalculation`, `fnSetCalColumn`, `range`, or similar internal calculation mechanics on normal runs.
- Do not manually set 지급금액, 합계, 또는 다른 readonly/calculation 필드. They are verification targets only.
- In OpenCode, if the current step has already met its success condition, immediately execute the next tool call instead of continuing to analyze or re-check the same state.
- If a file chooser modal is open, only `browser_file_upload` is allowed until that modal is resolved.
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
