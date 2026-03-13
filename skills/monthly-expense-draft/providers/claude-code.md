# Claude Code adapter

## Discovery and invocation

- Install under `.claude/skills/monthly-expense-draft/`.
- Explicit invocation with `/monthly-expense-draft` is still supported, but automatic skill invocation should also be allowed.
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
5. Create the zip in the primary session and copy it into an allowed upload path before the first upload attempt.
6. Fill the Goworks form, prefer targeted verification over repeated full-page snapshots, then capture only the final user-facing confirmation snapshot.
7. Hand off to the user for manual review and submit.

## PowerShell encoding on Windows

Claude Code의 Write 도구는 UTF-8 without BOM으로 파일을 저장한다.
Windows PowerShell 5.x는 BOM 없는 UTF-8 `.ps1` 파일을 시스템 로캘로 읽을 수 있어 한글 경로와 파일명이 깨질 수 있다.

### Avoid

```text
Write -> .ps1 저장 -> powershell -File script.ps1
bash heredoc -> powershell -Command "$var = ..."
```

### Prefer

1. 인라인 `powershell -NoProfile -Command` + `chcp 65001`
2. 필요 시 `-EncodedCommand` 또는 BOM 재인코딩 후 실행
3. 한글 경로는 환경변수로 전달하고 PowerShell 내부에서 `$env:`로 참조

예시:

```bash
powershell -NoProfile -Command "chcp 65001 | Out-Null; [Console]::OutputEncoding = [System.Text.Encoding]::UTF8; <command>"
```

- `$` 변수는 bash 치환을 피하도록 이스케이프하거나 인용부호를 주의한다.
- Node.js, Python 등 추가 런타임이 설치되어 있다고 가정하지 않는다.
- Windows 파일 작업은 기본 내장 PowerShell만으로 해결하는 것을 우선한다.

## Claude-specific notes

- Treat Opus/Sonnet selection as an optimization, not a requirement.
- Do not hardcode Claude-only tool IDs in the shared workflow.
- If Playwright is exposed through a plugin or MCP server, use the available browser tool names in the current environment.
- If browser automation is missing, first guide the user to `/plugin` -> `playwright`, then `/reload-plugins`, before falling back to broader MCP setup guidance.
