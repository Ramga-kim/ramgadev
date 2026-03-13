# GPT Codex adapter

## Discovery and invocation

- Install under `.agents/skills/monthly-expense-draft/` or `~/.agents/skills/monthly-expense-draft/`.
- Explicit invocation with `$monthly-expense-draft` is supported, and automatic invocation may also occur when the request clearly matches this workflow.

## Capability mapping

- `orchestrator`: primary Codex thread on the main reasoning model.
- `receipt-worker`: faster secondary thread or cheaper model for repeated OCR work.
- `browser-worker`: tool-stable thread with MCP or browser access.

## Recommended orchestration

1. Read `workflow.md`, config, and this adapter.
2. Resolve month, folder, and filename normalization in the primary thread.
3. If Codex multi-agent is enabled, parallelize receipt OCR and browser preload; otherwise run serially.
4. Merge sorted receipt JSON.
5. Create the zip locally and move or copy it into an allowed upload path before the first upload attempt.
6. Use MCP or browser tools to fill the Goworks form and upload the zip, preferring targeted checks over repeated full-page snapshots.
7. Return the final snapshot, upload verification, and manual-submit handoff.

## Codex-specific notes

- Use `/mcp` to inspect available external tools before starting browser work.
- Keep Codex-specific model choices in `skill-config.yaml` or Codex config, not in the shared skill text.
