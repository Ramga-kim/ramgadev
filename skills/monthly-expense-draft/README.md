# monthly-expense-draft

Goworks 월간 경비청구서 기안을 자동화하는 workflow skill.

## What it does

- 전월 영수증 폴더를 찾고 이미지 또는 PDF 영수증 정보를 읽는다
- 사용일, 금액, 상호명을 추출해 내역 테이블을 채운다
- 모든 첨부용 영수증 파일을 `yyyyMMdd[_NN].pdf` 형식으로 정리한 뒤 PDF 묶음 zip을 만들어 첨부한다
- 최종 제출 직전 상태까지만 자동화한다

## Prerequisites

- 브라우저 자동화가 가능한 에이전트 환경
- Windows 기반 사내 환경
- Goworks 경비청구서 접근 권한
- 월별로 정리된 로컬 영수증 폴더
- OpenCode, Claude Code, Gemini CLI, GPT Codex 중 어느 환경이든 가능하지만, 해당 환경에 브라우저 도구 또는 Playwright MCP가 연결되어 있어야 한다

## Browser setup

- `Claude Code`: `/plugin`에서 `playwright` 설치 후 `/reload-plugins`
- `OpenCode`: `~/.config/opencode/opencode.json`에 `playwright` MCP 및 권한 추가

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": { "*": "allow" },
    "write": { "*": "allow" },
    "read": { "*": "allow" },
    "external_directory": { "*": "allow" }
  },
  "mcp": {
    "playwright": {
      "type": "local",
      "command": ["npx", "@playwright/mcp@latest"],
      "enabled": true
    }
  }
}
```

## Safety boundaries

- 최종 상신 버튼은 절대 누르지 않는다
- 계정, 비밀번호, 개인 경로는 로컬에서만 채운다
- 임시 업로드 파일은 상신 후 첨부 확인 전까지 삭제하지 않는다

## Files

- `SKILL.md`: 기본 진입점이자 실제 스킬 본문
- `skill.md`: 구형 호환용 엔트리
- `workflow.md`: 공통 업무 흐름
- `skill-config.example.yaml`: 로컬 설정 예시
- `reference/goworks-upload.md`: Goworks DOM 및 업로드 참고 메모
