---
name: monthly-expense-draft
description: 월간 경비청구 기안. Goworks 경비청구서 작성, 전자결재 경비 기안 요청 시 사용.
allowed-tools: mcp__plugin_playwright_playwright__browser_navigate, mcp__plugin_playwright_playwright__browser_snapshot, mcp__plugin_playwright_playwright__browser_click, mcp__plugin_playwright_playwright__browser_fill_form, mcp__plugin_playwright_playwright__browser_wait_for, mcp__plugin_playwright_playwright__browser_select_option, mcp__plugin_playwright_playwright__browser_take_screenshot, mcp__plugin_playwright_playwright__browser_run_code, mcp__plugin_playwright_playwright__browser_file_upload, mcp__plugin_playwright_playwright__browser_network_requests, mcp__plugin_playwright_playwright__browser_tabs
compatibility: claude-code, gemini-cli, opencode, gpt-codex
metadata:
  category: finance-ops
  requires_browser_automation: "true"
  requires_receipt_files: "true"
---

# Goworks 경비청구서 기안

영수증 파일에서 데이터를 추출하고 Goworks 경비청구서 기안 폼을 자동으로 채운다.

## Preconditions

- 브라우저 자동화 도구가 필요하다.
- 도구가 없으면 아래 에이전트별 브라우저 설정 안내를 먼저 따른다.
- 브라우저 자동화가 준비되지 않으면 폼 자동 입력 단계로 진행하지 않는다.
- 이 스킬은 사내 Windows 환경을 전제로 한다. Unix/Linux 명령 전제나 POSIX 도구 사용을 허용하지 않는다.

### Agent browser setup

- `Claude Code`: `/plugin`에서 공식 `playwright` 플러그인을 설치하고 `/reload-plugins`를 실행한다.
- `OpenCode`: `~/.config/opencode/opencode.json`에 `playwright` MCP와 `bash`/`write`/`read`/`external_directory` 권한을 추가한다.
- 다른 에이전트도 동일하게 Playwright MCP 또는 동등한 브라우저 자동화 도구가 노출되어 있어야 한다.

## First read

1. `workflow.md`
2. `skill-config.yaml` 또는 `skill-config.example.yaml`
3. `reference/goworks-upload.md`

## Inputs

- 영수증 루트 경로는 최초 1회만 확인한다.
- 개인 설정값은 로컬에서만 채운다.
  - 이메일: `{YOUR_EMAIL}`
  - 패스워드: `{YOUR_PASSWORD}`
  - 영수증 루트 경로: `{YOUR_RECEIPT_ROOT_PATH}`

## 접속 정보

- 경비청구서 기안 URL: `https://www.goworks.co.kr/Approval/NewForm?templateid=2679`

### 로그인 처리

- 폼 제목이 보이면 로그인된 상태다.
- Microsoft 로그인으로 넘어가면 저장된 계정 정보로 로그인하거나 사용자에게 수동 로그인 안내를 한다.

## Workflow

### Phase 1: 입력과 폴더 준비

1. `receipt_root`가 없으면 한 번만 질문하고, 가능하면 `skill-config.yaml`에 저장한다.
2. 브라우저 워커는 답변을 기다리지 말고 먼저 Goworks 폼 접속과 로그인 상태 확인을 진행한다.
3. 전월 기준 대상 폴더를 정하고, 없으면 생성한다.
4. 영수증 입력은 이미지 파일(`jpg`, `jpeg`, `png`)과 PDF 파일(`pdf`)을 대상으로 한다. `zip`, `7z`, `rar`, `xlsx`는 무시한다.
5. PDF 하나 안에 여러 영수증이 있을 수 있으므로, 파일 단위가 아니라 영수증 건 단위로 추출한다.
6. 영수증 추출과 파일 읽기는 메인 세션이 직접 처리하지 말고 반드시 별도 워커 또는 서브에이전트에 위임한다.
7. 추출 결과는 `{ date, amount, vendor, category }` 형식의 날짜 오름차순 JSON 배열로 받는다.

### Phase 2: 폼 입력과 첨부

1. 제목은 `[경비청구서] {전월}월 개인경비` 기본값을 사용한다.
2. 행 추가와 값 입력은 가능한 한 `browser_run_code` 한 번으로 일괄 처리한다.
3. 계정과목은 기본 `야근주말식대`, 지출구분은 기본 `개인`을 사용한다.
4. 이미지 영수증은 건별 PDF로 변환하고, 원래 PDF인 영수증도 최종 첨부용 PDF 이름을 날짜 기준으로 정규화한다.
5. 최종 첨부 zip 내부 PDF 파일명은 반드시 `yyyyMMdd.pdf` 또는 `yyyyMMdd_NN.pdf` 형식을 따른다.
6. 업로드 확인과 DOM 세부 구조는 `reference/goworks-upload.md`를 따른다.

## Validation

- 폼 제목이 전월 기준으로 입력되어 있어야 한다.
- 내역 행 수가 영수증 건수와 일치해야 한다.
- 첨부 목록에 zip 파일명과 용량이 보여야 한다.
- 클라이언트 측 업로드 목록에 `File` 인스턴스가 존재해야 한다.
- zip 내부 PDF 파일명이 모두 `yyyyMMdd[_NN].pdf` 형식이어야 한다.

## 기본값

- 계정과목: `야근주말식대`
- 지출구분: `개인`
- 결재선, 참조, 보안구분, 보존연한은 서버 기본값을 따른다.

## Constraints

- 사용자에게 묻는 것은 루트 경로처럼 처음 한 번 필요한 값만 허용한다.
- 업무내용과 사업부는 사용자가 직접 보충하거나 미리 알려준 경우만 자동 입력한다.
- 상신 버튼은 절대 클릭하지 않는다.
- 상신 후 확인 단계 전에는 임시 zip 복사본을 삭제하지 않는다.
- 파일 탐색, 리네임, 복사, 압축은 Windows 기준으로만 수행하고 `ls`, `find`, `mv`, `cp` 같은 Unix식 명령을 사용하지 않는다.
- PowerShell 인코딩이나 quoting 문제를 우회하기 위해 `.ps1` 임시 파일을 생성하거나 실행하는 전략을 기본 또는 fallback으로 사용하지 않는다.
- file chooser modal이 열려 있는 동안에는 `browser_file_upload` 외 다른 browser tool을 호출하지 않는다.
- `browser_run_code`는 `page.evaluate(...)` 또는 검증된 Playwright 배치 상호작용이 포함된 경우에만 사용하고, DOM 전역 객체를 직접 참조하는 코드나 단일 필드 입력 용도로 사용하지 않는다.

## References

- `workflow.md`
- `reference/goworks-upload.md`
