# Claude Code adapter

## Discovery and invocation

- Install under `.claude/skills/monthly-expense-draft/`.
- Prefer explicit invocation with `/monthly-expense-draft`.
- `disable-model-invocation: true` in `SKILL.md` keeps this side-effectful workflow manual by default.
- If Playwright browser tools are missing, use `/plugin` to install the official `playwright` plugin in Claude Code.
- After plugin install or update, run `/reload-plugins` so the current session can use the new browser tools without restarting Claude Code.

## Capability mapping

- `orchestrator`: primary Claude session, usually the strongest reasoning model available.
- `receipt-worker`: fast worker subagents when parallel receipt OCR is worth the overhead.
- `browser-worker`: a tool-enabled worker or the primary session if browser work must stay inline.

## Recommended orchestration

1. Read `workflow.md`, config, and this adapter.
2. Resolve target month and receipt folder in the primary session.
3. Normalize filenames in the primary session.
4. If subagents are available, parallelize per-receipt OCR and optionally pre-load the browser in parallel; otherwise run the same steps serially.
5. Create the zip in the primary session.
6. Fill the Goworks form, upload the zip, and capture a final snapshot.
7. Hand off to the user for manual review and submit.

## Claude-specific notes

- Treat Opus/Sonnet selection as an optimization, not a requirement.
- Do not hardcode Claude-only tool IDs in the shared workflow.
- If Playwright is exposed through a plugin or MCP server, use the available browser tool names in the current environment.
- If browser automation is missing, first guide the user to `/plugin` -> `playwright`, then `/reload-plugins`, before falling back to broader MCP setup guidance.
