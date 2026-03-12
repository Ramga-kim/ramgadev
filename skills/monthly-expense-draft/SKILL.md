---
name: monthly-expense-draft
description: 월간 경비청구 기안. Goworks 경비청구서 작성, 전자결재 경비 기안 요청 시 사용.
allowed-tools: mcp__plugin_playwright_playwright__browser_navigate, mcp__plugin_playwright_playwright__browser_snapshot, mcp__plugin_playwright_playwright__browser_click, mcp__plugin_playwright_playwright__browser_fill_form, mcp__plugin_playwright_playwright__browser_wait_for, mcp__plugin_playwright_playwright__browser_select_option, mcp__plugin_playwright_playwright__browser_take_screenshot, mcp__plugin_playwright_playwright__browser_run_code, mcp__plugin_playwright_playwright__browser_file_upload, mcp__plugin_playwright_playwright__browser_network_requests, mcp__plugin_playwright_playwright__browser_tabs
compatibility: claude-code, gemini-cli, opencode, gpt-codex
metadata:
  category: finance-ops
  requires_browser_automation: "true"
  requires_receipt_images: "true"
disable-model-invocation: true
---

# Goworks 경비청구서 기안

영수증 이미지에서 데이터를 추출하고 Goworks 경비청구서 기안 폼을 자동으로 채운다.

## Preconditions

- 이 스킬은 Playwright 브라우저 도구를 사용한다.
- `mcp__plugin_playwright_playwright__browser_navigate` 같은 도구가 보이지 않으면 Playwright 플러그인이 설치되지 않은 것이다.
- 이 경우 사용자에게 다음을 안내한다: `Claude Code`에서 `/plugins` 명령으로 `playwright@claude-plugins-official` 플러그인을 설치 및 활성화해주세요.

## Inputs

- 최초 1회만 영수증 월간 관리 루트 경로와 폴더 컨벤션을 확인한다.
- 사용자별 설정은 아래 placeholder를 로컬에서 채울 수 있다. 비어 있으면 수동 로그인 또는 질문으로 처리한다.
  - 이메일: `{YOUR_EMAIL}`
  - 패스워드: `{YOUR_PASSWORD}`
  - 영수증 루트 경로: `{YOUR_RECEIPT_ROOT_PATH}`

## 접속 정보

- 경비청구서 기안 URL: `https://www.goworks.co.kr/Approval/NewForm?templateid=2679`

### 로그인 처리

1. `browser_navigate`로 기안 URL에 접속한 뒤 `browser_snapshot`으로 로그인 상태를 확인한다.
2. 페이지에 `경비청구서` 제목이 보이면 로그인된 상태로 보고 그대로 진행한다.
3. Microsoft 로그인 페이지로 리다이렉트되면 로그인 필요 상태다.
4. 저장된 계정 정보가 비어 있지 않으면 자동 로그인한다.
5. 비어 있으면 사용자에게 직접 로그인을 안내하고, 로그인 완료 후 `browser_snapshot`으로 재확인한다.

## Workflow

### Phase 1: 영수증 폴더 확인 및 이미지 전처리

#### 1-1. 루트 경로 및 컨벤션 확인

1. 사용자에게 영수증 월간 관리 루트 경로를 묻는다.
2. 예시 경로: `D:\_User\_Desktop\_Document And Project\_영수증`
3. 설정값이 비어 있으면 질문받은 값을 로컬 설정으로 반영한 뒤 계속 진행한다.
4. 루트 하위 폴더 구조를 탐색해서 연도별 폴더 여부와 월별 네이밍 패턴을 파악한다.
5. 컨벤션이 불명확한 경우에만 추가 질문한다.
6. 그 외 단계는 자동 수행한다.

#### 1-2. 대상 폴더 자동 결정

- 기안 대상은 전월 기준이다.
- 파악한 컨벤션에 따라 대상 영수증 폴더를 자동 결정한다.
- 예시 패턴: `{루트}/{연도}/{N}월 경비청구/`
- 폴더가 없으면 자동 생성한다.

#### 1-3. 영수증 가공을 서브에이전트에 위임

대상 폴더가 정해지면 서브에이전트에 아래 작업을 위임한다. 프롬프트에는 대상 폴더 절대경로를 반드시 포함한다.

중요 원칙:

- 메인 에이전트는 서브에이전트를 띄운 직후 Phase 2의 Goworks 접속과 폼 로드까지 먼저 진행한다.
- 서브에이전트 완료를 기다리며 멈추지 않는다.
- 서브에이전트 반환값은 날짜 오름차순 JSON 배열이어야 한다.

서브에이전트 지시사항:

> 주어진 폴더의 영수증 이미지(`jpg`, `jpeg`, `png`)를 가공하라.
>
> Step 1. 파일명 정규화
> 1. 이미 `yyyyMMdd` 또는 `yyyyMMdd_NN` 형식인 파일은 건너뛴다.
> 2. 그 외 파일은 이미지를 읽어 실제 결제 날짜를 추출한다.
> 3. 파일명을 `yyyyMMdd.jpg` 형식으로 변경한다.
> 4. 동일 날짜가 여러 장이면 `yyyyMMdd_01.jpg`, `yyyyMMdd_02.jpg`처럼 접미사를 붙인다.
>
> Step 2. 데이터 추출
> - 사용일: 파일명에서 `yyyy-MM-dd` 형식으로 파싱
> - 금액: 합계, 총액, 결제금액을 읽어 숫자만 추출
> - 상호명: 가맹점 또는 업체명 추출
>
> 최종 출력:
> ```json
> [
>   { "date": "2026-02-03", "amount": "14400", "vendor": "브라더김치찜 본점" }
> ]
> ```

### Phase 2: Goworks 경비청구서 기안 폼 자동 입력

1. `browser_navigate`로 기안 URL에 접속한다.
2. `browser_wait_for`로 로딩이 끝날 때까지 기다린다.
3. 로그인 필요 여부를 다시 확인한다.
4. 제목을 `[경비청구서] {전월}월 개인경비` 형식으로 입력한다.
5. Phase 1에서 받은 JSON 배열로 내역 테이블을 일괄 입력한다.
6. 영수증 이미지를 zip으로 압축해 첨부한다.
7. 마지막 상태를 `browser_snapshot`으로 사용자에게 보여준다.

### 폼 일괄 입력 절차

날짜 선택 달력을 반복해서 조작하지 말고 `browser_run_code`로 행 추가와 값 입력을 한 번에 처리한다.

예시 패턴:

```javascript
async (page) => {
  const data = [
    { date: "2026-02-03", amount: "14400", vendor: "김치찌개" },
    { date: "2026-02-10", amount: "14100", vendor: "된장찌개" }
  ];

  const addBtn = page.getByRole('button', { name: '행 추가' });
  for (let i = 1; i < data.length; i++) {
    await addBtn.click();
    await page.waitForTimeout(300);
  }

  await page.evaluate((data) => {
    const table = document.getElementById('Table3');
    const rows = table.querySelectorAll('tbody > tr');
    for (let i = 0; i < data.length; i++) {
      const row = rows[i + 1];
      if (!row) continue;
      const dateInput = row.querySelector('[id$="_1"] input[type="text"]');
      const amountInput = row.querySelector('[id$="_2"] input[type="text"]');
      const vendorInput = row.querySelector('[id$="_3"] input[type="text"]');
      if (dateInput) {
        dateInput.value = data[i].date;
        dateInput.dispatchEvent(new Event('change', { bubbles: true }));
      }
      if (amountInput) {
        amountInput.value = data[i].amount;
        amountInput.dispatchEvent(new Event('change', { bubbles: true }));
      }
      if (vendorInput) {
        vendorInput.value = data[i].vendor;
        vendorInput.dispatchEvent(new Event('change', { bubbles: true }));
      }
    }
  }, data);

  for (let i = 0; i < data.length; i++) {
    const rowIdx = i + 2;
    await page.locator(`#Table3 > tbody > tr:nth-child(${rowIdx}) > [id$="_5"] > select`).selectOption('야근주말식대');
    await page.locator(`#Table3 > tbody > tr:nth-child(${rowIdx}) > [id$="_8"] > select`).selectOption('개인');
  }

  for (let i = 0; i < data.length; i++) {
    const rowIdx = i + 2;
    const amountInput = page.locator(`#Table3 > tbody > tr:nth-child(${rowIdx}) [id$="_2"] input[type="text"]`);
    await amountInput.click();
    await amountInput.dispatchEvent('blur');
  }

  await page.waitForTimeout(500);
  return 'Done';
}
```

DOM과 첨부 동작 참고는 `reference/goworks-upload.md`를 본다.

### 첨부파일 절차

1. 대상 폴더의 영수증 이미지 파일만 압축한다.
2. zip 파일명은 `{N}월_경비청구서_{이름}.zip` 형식을 사용한다.
3. 원본 zip은 대상 폴더에 보존한다.
4. Playwright 업로드 허용 경로 밖에 있으면 작업 디렉토리로 복사한 임시본을 사용한다.
5. 임시본은 사용자가 상신 완료를 알리고 진행함에서 첨부파일 존재를 확인한 뒤에만 삭제한다.

예시 명령:

```bash
powershell -Command "Compress-Archive -Path '{대상폴더}\*.jpg','{대상폴더}\*.jpeg','{대상폴더}\*.png' -DestinationPath '{대상폴더}\{N}월_경비청구서_{이름}.zip' -Force"
```

## Validation

- 폼 제목이 전월 기준으로 입력되어 있어야 한다.
- 내역 행 수가 영수증 건수와 일치해야 한다.
- 첨부 목록에 zip 파일명과 용량이 보여야 한다.
- 클라이언트 측 업로드 목록에 `File` 인스턴스가 존재해야 한다.

## 기본값

- 계정과목: `야근주말식대`
- 지출구분: `개인`
- 결재선, 참조, 보안구분, 보존연한은 서버 기본값을 따른다.

## Constraints

- 사용자에게 묻는 것은 루트 경로와 컨벤션처럼 처음 한 번 필요한 값만 허용한다.
- 업무내용과 사업부는 사용자가 직접 보충하거나 미리 알려준 경우만 자동 입력한다.
- 상신 버튼은 절대 클릭하지 않는다.
- 상신 후 확인 단계 전에는 임시 zip 복사본을 삭제하지 않는다.

## References

- `reference/goworks-upload.md`
