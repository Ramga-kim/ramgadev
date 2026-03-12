# Gemini CLI adapter

## Discovery and invocation

- Install under `.gemini/skills/monthly-expense-draft/` or `.agents/skills/monthly-expense-draft/`.
- After updating files, run `/skills reload` or restart Gemini CLI.
- Prefer explicit prompts such as: `Use the monthly-expense-draft skill to prepare last month's personal expense draft.`

## Capability mapping

- `orchestrator`: main Gemini session, usually a stronger reasoning model.
- `receipt-worker`: faster or cheaper worker model for repeated receipt OCR.
- `browser-worker`: tool-stable model with browser or MCP access.

## Recommended orchestration

1. Activate the skill and read `workflow.md`, config, and this adapter.
2. Resolve the target month and normalize filenames.
3. If Gemini subagents or remote agents are enabled, parallelize receipt OCR and browser prewarm; otherwise run serially.
4. Build the sorted JSON result.
5. Create the zip locally.
6. Use the configured browser automation or Playwright MCP server to fill the form and upload the zip.
7. Return upload verification plus final manual-submit instructions.

## Gemini-specific notes

- Use Gemini-native skills and MCP features instead of Claude-style subagent assumptions.
- If browser automation comes from MCP, verify that server before starting form work.
- Keep model names in `skill-config.yaml` or local Gemini settings, not in the shared workflow text.
