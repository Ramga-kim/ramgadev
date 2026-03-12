# monthly-expense-skills

Multi-agent distribution repository for expense-draft skills.

## Goal

Package the `monthly-expense-draft` workflow as a shareable repository so different users can install the same skill set into Claude Code, Gemini CLI, OpenCode, or GPT Codex.

## Repository structure

```text
skills/
  monthly-expense-draft/
    SKILL.md
    skill.md
    workflow.md
    skill-config.example.yaml
    README.md
    providers/
      claude-code.md
      gemini-cli.md
      opencode.md
      gpt-codex.md
    agents/
      openai.yaml
```

## Install

Install the repository with your internal skill installer, or copy `skills/monthly-expense-draft/` into the target agent's skills directory.

Example repository install after upload:

```bash
npx skills add <org>/<repo> --agent opencode --yes
npx skills add <org>/<repo> --agent claude-code --yes
npx skills add <org>/<repo> --agent gemini-cli --yes
npx skills add <org>/<repo> --agent gpt-codex --yes
```

Recommended targets:

- Claude Code: `.claude/skills/monthly-expense-draft/`
- Gemini CLI: `.gemini/skills/monthly-expense-draft/` or `.agents/skills/monthly-expense-draft/`
- OpenCode: `.opencode/skills/monthly-expense-draft/` or `.agents/skills/monthly-expense-draft/`
- GPT Codex: `.agents/skills/monthly-expense-draft/`

## Included skills

| Skill | Type | Description |
|------|------|------|
| `monthly-expense-draft` | Workflow | Prepare the previous month's expense draft from receipts and upload the attachment package to Goworks/NADA. |
