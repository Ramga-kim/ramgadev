---
name: monthly-expense-draft
description: Manual skill for preparing the previous month's personal expense draft from receipt images and uploading the draft package to Goworks/NADA with browser automation. Use only when the user explicitly asks to draft, upload, or automate the expense form.
compatibility: claude-code, gemini-cli, opencode, gpt-codex
metadata:
  category: finance-ops
  requires_browser_automation: "true"
  requires_receipt_images: "true"
disable-model-invocation: true
---

# Monthly Expense Draft

This is the compatibility entrypoint kept for older Claude-only setups that still reference `skill.md` directly.
The canonical cross-provider entrypoint is `SKILL.md`, and both files intentionally carry the same instructions.

Use this skill only when the user explicitly asks to prepare or upload a monthly expense draft.

## First read

1. Read `workflow.md` for the provider-neutral business logic.
2. Read `skill-config.yaml` if it exists; otherwise read `skill-config.example.yaml` to understand defaults and optional overrides.
3. Read the provider adapter that matches the current environment:
   - `providers/claude-code.md`
   - `providers/gemini-cli.md`
   - `providers/opencode.md`
   - `providers/gpt-codex.md`
4. If the environment is unknown, follow `workflow.md` serially with the tools you actually have.

## Required capabilities

- Local file access to the receipt folder
- Receipt image understanding or OCR
- Browser automation for Goworks/NADA
- Ability to upload a local zip file from the active machine

If any required capability is missing, stop early and tell the user exactly what is missing and which adapter/setup file to check.

## Universal rules

- Never click the final submit or approval button; the user must do that.
- Never delete the generated zip or any temporary copy before post-submit verification succeeds.
- Ask only for missing business inputs that cannot be inferred from config, receipts, or the current page.
- Prefer provider-native subagents or parallelism when available, but keep the same business workflow when running serially.
- Keep the user updated with concrete progress: folder resolved, OCR done, form filled, upload verified, waiting for manual submit.

## Execution roles

- `orchestrator`: resolves month and paths, coordinates workers, merges results, and reports progress.
- `receipt-worker`: normalizes one receipt filename and extracts `{date, amount, vendor, category}`.
- `browser-worker`: loads the Goworks form, fills rows, uploads the zip, and captures a final snapshot.

Treat these as logical roles. Map them to environment-specific models, subagents, or sessions using `skill-config.yaml` and the provider adapter instead of hardcoding vendor names here.

## Output contract

Return all of the following:

1. Target month and resolved receipt folder
2. Date-sorted JSON array for all receipts
3. Attachment path and upload verification result
4. Final handoff telling the user what to review or click next
