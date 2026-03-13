# 경비청구서 기안 — 워크플로우 (벤더 무관)

이 문서는 AI 코딩 에이전트 종류와 무관하게 적용되는 핵심 비즈니스 로직만 정리한다.
DOM/업로드 세부는 `goworks-upload.md`를 참조한다.

---

## 0. 설정 파일

- 선택적 설정 파일: `skill-config.yaml`
- 기본 템플릿: `skill-config.example.yaml`
- 우선순위: `skill-config.yaml` 값 -> 이 문서의 기본값 -> 사용자에게 1회 질문
- 영속 설정이 필요하면 `workflow.md`를 수정하지 말고 `skill-config.yaml`에만 기록한다.
- 모델명/서브에이전트명은 이 문서에서 직접 고정하지 말고, 설정 파일의 역할명(`orchestrator`, `receipt_worker`, `browser_worker`)으로만 다룬다.

---

## 1. 개요

영수증 파일에서 데이터를 자동 추출하고, Goworks 경비청구서 기안 폼을 자동으로 채운다.

### 전체 흐름

```
[영수증 폴더 확인] → [파일명 정규화] → [이미지 OCR] → [Goworks 폼 로드]
                                          ↓                    ↓
                                    [데이터 JSON]      [폼 ready]
                                          ↓                    ↓
                                    [폼 일괄 입력 + zip 첨부] → [사용자 검토 → 상신]
```

### 병렬 처리 원칙

- 브라우저 작업은 영수증 루트 경로 응답을 기다릴 필요가 없다.
- 사용자에게 영수증 루트 경로를 묻는 동안에도 브라우저 워커는 Goworks 기안 URL 접속, 로그인 상태 확인, 폼 로드까지 먼저 진행한다.
- 영수증 루트 경로가 필요한 작업은 폴더 탐색, 파일명 정규화, OCR, zip 생성부터다.
- 따라서 브라우저 자동화가 가능하면 입력 대기 중에도 브라우저 워커를 idle 상태로 두지 말고, 먼저 할 수 있는 준비 작업을 끝내둔다.
- 전체 페이지 스냅샷은 꼭 필요할 때만 사용하고, 가능하면 특정 필드값이나 브라우저 내 검증 코드로 대체한다.

---

## 2. 영수증 폴더 관리

### 루트 경로 및 컨벤션

- 기본 예시 경로 : `C:\Users\<USER>\Desktop\영수증`
- 폴더 컨벤션: `{루트}/{연도}/{N}월 경비청구/`
  - 예: `C:\Users\<USER>\Desktop\영수증\2026\2월 경비청구\`
- `skill-config.yaml`에 `business.receipt_root`가 있으면 우선 사용
- 루트 경로가 비어 있으면 사용자에게 1회 질문하고, 받은 값을 현재 실행에 즉시 사용한다.
- `skill-config.yaml`이 있으면 `business.receipt_root`에 저장해 다음 실행에서도 재사용한다.
- `skill-config.yaml`이 없고 로컬 파일 저장이 가능하면 새로 만들거나 업데이트해서 같은 경로를 기억한다.
- 저장이 불가능한 환경이면 현재 실행 동안 메모리에 유지하고, 같은 실행에서는 다시 묻지 않는다.

### 대상 월 결정

- 기안 대상 = 전월 (현재 3월이면 2월)
- 1월 기안 시 전년도 12월

### 파일 처리 규칙

- 이 워크플로우는 Windows 환경만 지원한다.
- 입력 대상은 이미지 파일(`jpg`, `jpeg`, `png`)과 PDF 파일(`pdf`)이다.
- `zip`, `7z`, `rar`, `xlsx` 등 비대상 파일은 무시한다.
- Windows 환경에서는 파일 확인, 리네임, 복사, 압축 작업에 PowerShell을 우선 사용한다.
- `ls`, `find`, `mv`, `cp` 등 Unix식 파일 명령을 기본 전략으로 사용하지 않는다. 파일 탐색은 Glob 또는 PowerShell, 파일 조작은 PowerShell로만 처리한다.
- 한글 경로 처리 시 인코딩 주의가 필요하다. 벤더별 허용 패턴은 각 provider 문서를 따른다.
- Node.js, Python 등 Windows 기본 제공이 아닌 런타임에 의존하지 않는다.
- PowerShell quoting이나 인코딩 문제를 피하기 위한 `.ps1` 임시 파일 생성/실행을 기본 전략이나 fallback으로 사용하지 않는다.
- 이미 `yyyyMMdd` 또는 `yyyyMMdd_NN` 형식인 파일은 유지한다.
- 그 외 파일은 실제 결제 날짜를 읽어 `yyyyMMdd.<ext>` 형식으로 정규화한다.
- 동일 날짜 복수 파일은 `yyyyMMdd_01.<ext>`, `yyyyMMdd_02.<ext>`처럼 접미사를 붙인다.
- PowerShell 인라인 명령에서 bash와 `$` 이스케이프 충돌이 날 수 있으므로, provider 문서의 검증된 quoting 패턴을 우선 사용한다.
- bash 안에서 PowerShell을 호출할 때 `$`가 들어가면 single-quoted outer command 또는 `$env:` 전달 패턴을 반드시 사용한다.
- PDF 하나 안에 여러 영수증이 있을 수 있으므로, 파일 수와 영수증 건수를 같다고 가정하지 않는다.

---

## 3. 영수증 OCR 데이터 추출

### 추출 항목

- 영수증 이미지 읽기와 PDF 추출은 메인 세션이 직접 처리하지 않는다.
- 영수증 파일마다 별도 `receipt_worker` 서브에이전트 1개를 반드시 fan-out한다. 폴더 전체를 단일 worker 하나가 순차 처리하는 방식을 기본 또는 fallback으로 사용해서는 안 된다.
- 메인 세션은 OCR 결과 JSON만 받아 이후 폼 입력을 진행한다.
- `receipt_worker`는 이 스킬을 다시 로드하거나 전체 경비청구 workflow를 재실행해서는 안 된다. 자신의 역할은 영수증 추출과 첨부 산출물 준비로만 제한한다.
- `receipt_worker`는 멀티모달 LLM으로 이미지/PDF 내용을 읽어 추출해야 한다. Python, Node.js, ImageMagick, Ghostscript, 커스텀 OCR 스크립트 탐색을 시도해서는 안 된다.
- vendor는 원문에 보이는 상호명을 보수적으로 추출해야 하며, `본점`을 `분점`으로 바꾸는 식의 추정이나 의역을 해서는 안 된다.
- 중복 판정은 승인번호를 우선 사용해야 하며, 승인번호가 없을 때만 날짜+금액+상호 조합으로 수행해야 한다.
- 최종 반환의 `json:` 값은 반드시 전체 JSON 배열이어야 하며, 객체 나열이나 느슨한 포맷을 사용해서는 안 된다.
- 각 영수증 건에서 `date`, `amount`, `vendor`, `category`를 추출한다.
- `amount`는 숫자만 유지한다.
- `category`는 `야근주말식대`, `교통비`, `운송비`, `기타` 중 하나로 정규화한다.
- 식사류는 `야근주말식대`, 교통/주유/주차는 `교통비`, 택배/화물은 `운송비`, 그 외는 `기타`를 사용한다.
- 출력은 날짜 오름차순 JSON 배열로 통일한다.

---

## 4. Goworks 경비청구서 폼

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
- SSO 환경에서는 navigate 직후 바로 클릭하지 말고, 현재 URL 또는 화면 상태를 먼저 확인한 뒤 로그인 클릭 필요 여부를 판단한다.
- navigate 직후 최종 URL이 이미 Goworks 도메인으로 돌아왔거나 홈 화면 요소가 보이면 로그인 클릭을 시도하지 않는다.

### 제목 규칙

- 기본값: `[경비청구서] {N}월 개인경비` (예: `[경비청구서] 2월 개인경비`)
- `skill-config.yaml`의 `business.title_template`가 있으면 그 값을 우선 사용

- 행 추가는 반드시 한 번의 배치 브라우저 실행 또는 동일 실행 컨텍스트 내부 루프로 묶어 처리한다. 개별 `browser_click` 반복을 기본 전략으로 사용하지 않는다.
- 이 전자결재 양식의 상세 테이블 구조는 고정이며, 컬럼 의미를 매 실행마다 다시 탐색하지 않는다.
- DOM 구조와 고정 셀 인덱스는 `goworks-upload.md`를 그대로 따른다.
- 행 추가 버튼의 Playwright locator가 바로 잡히지 않으면, 같은 배치 실행 안에서 검증된 버튼 탐색 fallback (`button`, `input[type="button"]`, known onclick 패턴)으로 전환한다.
- 기본값은 계정과목 `야근주말식대`, 지출구분 `개인`이다.
- 서버 자동 계산 또는 readonly 필드는 직접 입력하지 않고, 자동 반영 결과만 확인한다.
- Goworks 테이블 입력은 다음 고정 전략을 반드시 따른다: 배치 행 추가 -> 배치 텍스트 입력 -> 배치 select 처리 -> blur 또는 경량 검증.
- `browser_select_option` 개별 반복은 fallback일 뿐이며, 기본 전략으로 사용하지 않는다.
- select 필드는 먼저 배치 처리 전략을 시도하고, Goworks 이벤트 반영이 실제로 실패한 경우에만 `browser_select_option` 개별 호출로 내려간다.
- 단일 필드 제목 입력이나 단순 텍스트 입력은 `browser_fill_form` 또는 직접 ref 기반 입력을 우선 사용하고, 복잡한 locator를 새로 구성하는 `browser_run_code`를 기본 전략으로 쓰지 않는다.
- `browser_run_code` 안에서 DOM 접근이 필요하면 반드시 `page.evaluate(() => ...)` 내부에서 처리한다. `document`나 `window`를 top-level code에서 직접 참조하지 않는다.
- 테이블 구조 재탐색은 기본 단계가 아니며, 고정 구조 기반 입력이 실제로 실패했을 때만 예외적으로 허용한다.

---

## 5. zip 첨부

- 입력이 이미지면 영수증 1건당 PDF 1개로 변환한다.
- 입력이 이미 PDF여도 최종 첨부용 PDF 파일명은 반드시 날짜 기준으로 정규화한다.
- 최종 첨부 대상은 `yyyyMMdd.pdf` 또는 `yyyyMMdd_NN.pdf` 형식으로 정규화된 PDF 파일들만 포함한 zip이다.
- 파일명과 생성 명령은 `skill-config.yaml` 기본값 또는 로컬 설정을 따른다.
- `skill-config.yaml`의 `business.upload_staging_path`를 allowed root 기본값으로 사용한다.
- 원본 zip은 대상 폴더에 보존한다.
- 업로드 전에 Playwright 허용 경로 안의 임시 경로를 먼저 준비하고, 첫 업로드부터 그 경로만 사용한다.
- 실패 후 다른 경로를 다시 시도하지 말고, 허용 경로 여부를 먼저 확인한다.
- zip 생성 직후 allowed root 내부 경로로 복사 또는 생성해서, 업로드 단계에서 경로 전환이 발생하지 않게 한다.
- staging 경로는 디렉토리 자체가 아니라 파일 경로까지 포함한 full path로 다루며, staging 폴더 자체를 삭제 대상으로 취급하지 않는다.
- 원본 PDF 이름(`afsbb.pdf` 같은 이름)을 zip 내부에 그대로 남기면 안 된다. 최종 첨부 산출물 이름은 모두 날짜 기준으로 재명명되어야 한다.

- 업로드 검증은 첨부 목록 표시와 `insertFileList`의 `File` 존재 여부로 확인한다.
- 임시 zip 파일은 상신 후 첨부 확인 전까지 삭제하지 않는다.
- file chooser modal이 열려 있으면 `browser_file_upload`만 호출한다. modal이 열린 상태에서 `browser_click`, `browser_run_code`, 추가 탐색 tool을 호출하지 않는다.
- 필요하면 zip 생성 직후 압축을 풀지 않고도 zip 내부 파일명 목록을 확인해서 `yyyyMMdd[_NN].pdf` 규칙을 검증한다.

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

## 8. 오케스트레이션 규칙

- 영수증 루트 경로가 아직 없어도 브라우저 워커는 즉시 시작할 수 있다.
- 권장 순서: 브라우저 확인 -> 폼 prewarm -> 루트 경로 확인 -> OCR -> 일괄 입력
- 브라우저 워커는 영수증 루트 경로 응답 전이라도 가능한 준비 작업을 모두 완료한 뒤 대기한다.
- 폼 확인은 전체 스냅샷 반복보다 경량 검증을 우선 사용한다.
- 로그인 직후나 페이지 전환 직후에는 stale ref를 가정하고, 이전 element ref 재사용보다 새 locator 또는 새 확인 절차를 우선 사용한다.
- snapshot의 `ref` 값은 Playwright 도구용 참조일 뿐이며, `browser_run_code` 내부에서 DOM selector로 사용하지 않는다.
- 제목, 합계, 입력 행 수처럼 다음 단계 진행에 필요한 조건이 충족되면 추가 탐색이나 재판단 없이 즉시 다음 tool call을 실행한다.
- 같은 세션에서 이미 성공한 입력 전략이 있으면 다른 selector 전략이나 구조 탐색을 다시 하지 않는다.
- 첨부 단계에서는 `business.upload_staging_path`만 사용하며, 다른 경로를 다시 비교하거나 탐색하지 않는다.
- modal state가 보이면 현재 modal을 먼저 해소하고, 그 modal을 처리하는 tool 외 다른 tool은 사용하지 않는다.
