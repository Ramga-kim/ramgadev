# GPT Codex adapter

## Discovery and invocation

- Install under `.agents/skills/monthly-expense-draft/` or `~/.agents/skills/monthly-expense-draft/`.
- Prefer explicit invocation with `$monthly-expense-draft`.
- `agents/openai.yaml` disables implicit invocation for this side-effectful workflow.

## Capability mapping

- `orchestrator`: primary Codex thread on the main reasoning model.
- `receipt-worker`: faster secondary thread or cheaper model for repeated OCR work.
- `browser-worker`: tool-stable thread with MCP or browser access.

## Recommended orchestration

1. Read `workflow.md`, config, and this adapter.
2. Resolve month, folder, and filename normalization in the primary thread.
3. If Codex multi-agent is enabled, parallelize receipt OCR and browser preload; otherwise run serially.
4. Merge sorted receipt JSON.
5. Create the zip locally.
6. Use MCP or browser tools to fill the Goworks form and upload the zip.
7. Return the final snapshot, upload verification, and manual-submit handoff.

## Codex-specific notes

- Use `/mcp` to inspect available external tools before starting browser work.
- Keep Codex-specific model choices in `skill-config.yaml` or Codex config, not in the shared skill text.
- Never rely on implicit skill activation for this workflow; explicit `$monthly-expense-draft` is safer.
