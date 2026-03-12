# Monthly Expense Draft

Cross-provider agent skill for preparing last month's personal expense draft from receipt images and uploading the draft package to Goworks/NADA.

## What changed

- `workflow.md` now holds the provider-neutral business logic.
- `SKILL.md` is the canonical open-standard entrypoint.
- `providers/` contains orchestration adapters for Claude Code, Gemini CLI, OpenCode, and GPT Codex.
- `skill-config.example.yaml` provides a provider-neutral place for local defaults and optional model-role mapping.
- `agents/openai.yaml` adds Codex-specific presentation and explicit-invocation policy.

## Recommended install paths

- Claude Code: `~/.claude/skills/monthly-expense-draft/SKILL.md`
- Gemini CLI: `~/.gemini/skills/monthly-expense-draft/SKILL.md` or `~/.agents/skills/monthly-expense-draft/SKILL.md`
- OpenCode: `~/.config/opencode/skills/monthly-expense-draft/SKILL.md`, `~/.claude/skills/...`, or `~/.agents/skills/...`
- GPT Codex: `~/.agents/skills/monthly-expense-draft/SKILL.md`

## Explicit invocation hints

- Claude Code: `/monthly-expense-draft`
- Gemini CLI: ask to use the `monthly-expense-draft` skill, then approve activation
- OpenCode: ask to use or load the `monthly-expense-draft` skill
- GPT Codex: `$monthly-expense-draft`

## Minimal setup

1. Install the skill in the path your agent discovers.
2. Copy `skill-config.example.yaml` to `skill-config.yaml` if you want persistent local defaults.
3. Ensure browser automation is available in that client (for example Playwright via MCP/plugin).
4. Invoke the skill explicitly for expense drafting tasks.

## Files to read first

- `SKILL.md`
- `workflow.md`
- `providers/claude-code.md`
- `providers/gemini-cli.md`
- `providers/opencode.md`
- `providers/gpt-codex.md`
- `skill-config.example.yaml`

## Compatibility note

The lowercase `skill.md` file is retained as a legacy entrypoint for older Claude-only setups that already reference it directly. New installs should use `SKILL.md`.
