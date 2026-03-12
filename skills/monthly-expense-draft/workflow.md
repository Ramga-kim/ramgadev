# 경비청구서 기안 — 워크플로우 (벤더 무관)

이 문서는 AI 코딩 에이전트 종류와 무관하게 적용되는 **비즈니스 로직, 데이터 흐름, DOM 구조**를 정의한다.
벤더별 실행 전략(서브에이전트, 병렬화, 모델 선택 등)은 각 어댑터 문서를 참조한다.

---

## 0. 설정 파일

- 선택적 설정 파일: `skill-config.yaml`
- 기본 템플릿: `skill-config.example.yaml`
- 우선순위: `skill-config.yaml` 값 -> 이 문서의 기본값 -> 사용자에게 1회 질문
- 영속 설정이 필요하면 `workflow.md`를 수정하지 말고 `skill-config.yaml`에만 기록한다.
- 모델명/서브에이전트명은 이 문서에서 직접 고정하지 말고, 설정 파일의 역할명(`orchestrator`, `receipt_worker`, `browser_worker`)으로만 다룬다.

---

## 1. 개요

영수증 이미지에서 데이터를 자동 추출하고, Goworks 경비청구서 기안 폼을 자동으로 채운다.

### 전체 흐름

```
[영수증 폴더 확인] → [파일명 정규화] → [이미지 OCR] → [Goworks 폼 로드]
                                          ↓                    ↓
                                    [데이터 JSON]      [폼 ready]
                                          ↓                    ↓
                                    [폼 일괄 입력 + zip 첨부] → [사용자 검토 → 상신]
```

---

## 2. 영수증 폴더 관리

### 루트 경로 및 컨벤션

- 실제 경로 : `D:\_User\_Desktop\_Document And Project\_영수증`
- 폴더 컨벤션: `{루트}/{연도}/{N}월 경비청구/`
  - 예: `D:\_User\_Desktop\_Document And Project\_영수증\2026\2월 경비청구\`
- `skill-config.yaml`에 `business.receipt_root`가 있으면 우선 사용
- 루트 경로가 비어 있으면 사용자에게 1회 질문하고, 영속 저장이 필요할 때만 `skill-config.yaml`에 기록

### 대상 월 결정

- 기안 대상 = 전월 (현재 3월이면 2월)
- 1월 기안 시 전년도 12월

### 파일명 정규화 규칙

| 현재 파일명 | 처리 |
|------------|------|
| `yyyyMMdd.jpg` 또는 `yyyyMMdd_NN.jpg` | 리네임 불필요 |
| 그 외 | 이미지를 읽어 결제 날짜 추출 → `yyyyMMdd.jpg`로 리네임 |
| 동일 날짜 복수 | `yyyyMMdd_01.jpg`, `yyyyMMdd_02.jpg`, ... |

---

## 3. 영수증 OCR 데이터 추출

### 추출 항목

각 영수증 이미지에서 다음 4개 필드를 추출한다:

| 필드 | 설명 | 예시 |
|------|------|------|
| `date` | 사용일 (파일명에서 파싱) | `"2026-02-03"` |
| `amount` | 결제금액 (숫자만, 콤마 제거) | `"14400"` |
| `vendor` | 가맹점/상호명 | `"브라더김치찜 본점"` |
| `category` | 용도 분류 (아래 규칙) | `"야근주말식대"` |

### 용도(category) 분류 규칙

| category | 대상 |
|----------|------|
| **야근주말식대** | 식당, 카페, 편의점 식품, 배달음식 등 식사 관련 |
| **교통비** | 버스, 기차, 택시, 주유소/기름, 하이패스, 주차비 등 교통 관련 |
| **운송비** | 택배, 화물운송 등 배송/화물 관련 |
| **기타** | 위에 해당하지 않는 비용 |

### 출력 포맷

날짜 오름차순 JSON 배열:
```json
[
  { "date": "2026-02-03", "amount": "14400", "vendor": "브라더김치찜 본점", "category": "야근주말식대" },
  { "date": "2026-02-10", "amount": "14100", "vendor": "도리분식&정우의도시락", "category": "야근주말식대" },
  { "date": "2026-02-15", "amount": "50000", "vendor": "SK에너지 수원IC", "category": "교통비" }
]
```

---

## 4. Goworks 경비청구서 폼 구조

### 접속 정보

- URL: `https://www.goworks.co.kr/Approval/NewForm?templateid=2679`
- 첫 접속 시 홈(`/Home/Index`)으로 리다이렉트되는 경우가 있음 -> 같은 URL로 재시도
- `Loading...` 텍스트가 사라질 때까지 대기 필요
- `skill-config.yaml`의 `goworks.form_url`가 있으면 그 값을 우선 사용

### 로그인

- Microsoft SSO 기반. 이미 로그인 상태면 `경비청구서` heading이 보임
- 로그인 필요 시 사용자에게 수동 로그인 안내
- 저장된 계정 정보는 `skill-config.yaml`의 `goworks.email`, `goworks.password`에서만 읽는다.
- 값이 비어 있거나 저장을 원하지 않으면 수동 로그인으로 진행한다.

### 양식 기본값 (서버 자동 설정)

- 결재선: 신현규 팀장 -> 최민석 이사 -> 이해철 사장
- 참조: 마하은
- 작성자: 김가람/AI연구소/팀원
- 보안구분: 비공개 / 보존연한: 5년

### 제목 규칙

- 기본값: `[경비청구서] {N}월 개인경비` (예: `[경비청구서] 2월 개인경비`)
- `skill-config.yaml`의 `business.title_template`가 있으면 그 값을 우선 사용

### 내역 테이블 (Table3) DOM 구조

```
#Table3 > tbody > tr
  |- tr[0] = 헤더 (사용일, 금액, 상호, 업무내용, 용도, 프로젝트, 사업부, 지출구분)
  |- tr[1] = 데이터행 1 (첫 행은 이미 존재)
  |- tr[2] = 데이터행 2 ("행 추가" 버튼으로 생성)
  |- ...
  |- tr[last] = 합계행
```

각 행의 셀 ID 패턴: `{행번호}_{열번호}`
| 열번호 | 필드 | 타입 |
|--------|------|------|
| `_1` | 사용일 | text input |
| `_2` | 금액 | text input |
| `_3` | 상호 | text input |
| `_4` | 업무내용 | text input |
| `_5` | 용도 | select (야근주말식대/교통비/운송비/기타) |
| `_6` | 프로젝트 | text input |
| `_7` | 사업부 | select |
| `_8` | 지출구분 | select (법인카드/개인) |

### 폼 일괄 입력 JavaScript

```javascript
async (page) => {
  const data = [/* OCR 결과 JSON 배열 */];

  // 1) 행 추가 (첫 행은 이미 존재, 클릭 사이 300ms 대기 필수)
  const addBtn = page.getByRole('button', { name: '행 추가' });
  for (let i = 1; i < data.length; i++) {
    await addBtn.click();
    await page.waitForTimeout(300);
  }

  // 2) 텍스트 필드 일괄 입력
  await page.evaluate((data) => {
    const table = document.getElementById('Table3');
    const rows = table.querySelectorAll('tbody > tr');
    for (let i = 0; i < data.length; i++) {
      const row = rows[i + 1];
      if (!row) continue;
      const dateInput = row.querySelector('[id$="_1"] input[type="text"]');
      const amountInput = row.querySelector('[id$="_2"] input[type="text"]');
      const vendorInput = row.querySelector('[id$="_3"] input[type="text"]');
      if (dateInput) { dateInput.value = data[i].date; dateInput.dispatchEvent(new Event('change', {bubbles:true})); }
      if (amountInput) { amountInput.value = data[i].amount; amountInput.dispatchEvent(new Event('change', {bubbles:true})); }
      if (vendorInput) { vendorInput.value = data[i].vendor; vendorInput.dispatchEvent(new Event('change', {bubbles:true})); }
    }
  }, data);

  // 3) select 필드
  for (let i = 0; i < data.length; i++) {
    const rowIdx = i + 2;
    await page.locator(`#Table3 > tbody > tr:nth-child(${rowIdx}) > [id$="_5"] > select`).selectOption(data[i].category);
    await page.locator(`#Table3 > tbody > tr:nth-child(${rowIdx}) > [id$="_8"] > select`).selectOption('개인');
  }

  // 4) 합계 자동 계산 트리거
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

### 계정과목 및 지출구분

- 계정과목: 야근주말식대, 교통비, 운송비, 기타
- 지출구분: 법인카드 / 개인 (기본: 개인)
- 사업부 옵션: 나다, 경영지원팀, 기술영업부, 영업팀, 엔지니어링팀, 해외영업팀, 연구소, AI센터, 시스템개발팀

---

## 5. zip 첨부

### zip 생성

- 대상: 영수증 이미지 파일(jpg, jpeg, png)만 포함 (숨김폴더 제외)
- 기본 파일명: `{N}월_경비청구서_김가람.zip`
- `skill-config.yaml`의 `business.attachment_name_template`가 있으면 그 값을 우선 사용
- 생성 위치: 영수증 대상 폴더 (보존)
- Windows: `powershell -Command "Compress-Archive -Path '{폴더}\*.jpg','{폴더}\*.jpeg','{폴더}\*.png' -DestinationPath '{폴더}\{N}월_경비청구서_김가람.zip' -Force"`

### 업로드

1. `첨부파일 찾아보기` 버튼 클릭 -> file chooser -> zip 경로 전달
2. 업로드 확인: `insertFileList`에 `hasFile: true` 인 `File` 인스턴스 존재 확인
   ```javascript
   async (page) => {
     const result = await page.evaluate(() => {
       return insertFileList.map(item => ({
         fileName: item.file?.name, fileSize: item.file?.size, hasFile: item.file instanceof File
       }));
     });
     return JSON.stringify(result);
   }
   ```

### Goworks 업로드 아키텍처 주의

- 파일 선택 시 서버에 즉시 업로드하지 않음 -> `insertFileList`에 클라이언트 보관
- **상신 시** `PerformUpload`로 서버에 POST
- 따라서 **임시 zip 파일을 상신 완료 전에 삭제하면 안 됨** (lazy 참조 깨짐 -> `ERR_FILE_NOT_FOUND`)

---

## 6. 상신 후 확인

- **상신 버튼은 에이전트가 절대 클릭하지 않는다** - 사용자가 직접 수행
- 상신 완료 후 확인 절차:
  1. 진행함 페이지: `https://www.goworks.co.kr/Approval/DocumentBox?documentStatus=AprIng`
  2. 목록 최상단에 기안 확인 -> 클릭 -> 첨부파일 섹션에 zip 존재 확인
  3. 확인 완료 후 임시 zip 복사본 삭제 가능

---

## 7. 사용자 입력이 필요한 항목

| 항목 | 시점 | 비고 |
|------|------|------|
| 영수증 루트 경로 | 최초 1회 | 필요하면 `skill-config.yaml`에 저장 |
| 업무내용 | 폼 완성 후 | 사용자가 직접 입력 |
| 사업부 | 폼 완성 후 | 사용자가 직접 선택 |
| 상신 | 검토 후 | 사용자가 직접 클릭 |
