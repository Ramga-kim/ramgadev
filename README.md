# nada-skills

나다 사내 업무 자동화용 AI agent skills 모음.

## Install

### 1. Configure access (one time)

`nada-dl` org 저장소는 비공개이므로 GitHub PAT가 필요하다.

```bash
# git - 스킬 설치 및 npm git 의존성 접근용
git config --global url."https://<PAT>@github.com/nada-dl/".insteadOf "https://github.com/nada-dl/"

# npm - GitHub Packages (@nada-dl 스코프)
npm config set @nada-dl:registry https://npm.pkg.github.com
npm config set //npm.pkg.github.com/:_authToken <PAT>
```

### 2. 스킬 설치

```bash
npx skills add Ramga-kim/ramgadev --agent opencode --yes
```

필요하면 다른 에이전트에서도 같은 저장소 주소를 사용할 수 있다.

```bash
npx skills add Ramga-kim/ramgadev --agent claude-code --yes
npx skills add Ramga-kim/ramgadev --agent gemini-cli --yes
npx skills add Ramga-kim/ramgadev --agent gpt-codex --yes
```

## Repository layout

```text
nada-skills/
|- README.md
`- skills/
   `- monthly-expense-draft/
      |- SKILL.md
      |- README.md
      |- workflow.md
      |- skill.md
      |- skill-config.example.yaml
      |- providers/
      |- agents/
      `- reference/
         `- goworks-upload.md
```

## Skill catalog

### Workflow

| Skill | Depends on | Description |
|------|------|------|
| `monthly-expense-draft` | Playwright plugin | Goworks 월간 경비청구서 기안 자동화 |

### Tool Reference

아직 없음.

## Conventions

- 각 스킬은 `skills/<skill-name>/SKILL.md`에 둔다
- 스킬 이름은 kebab-case를 사용하고 폴더명과 동일하게 맞춘다
- 사람용 설명은 `skills/<skill-name>/README.md`에 둔다
- 길어지는 도메인 규칙이나 셀렉터 메모는 `skills/<skill-name>/reference/`로 분리한다
- 저장소는 단순하게 유지하고, 공용 문서는 스킬이 늘어날 때만 추가한다

## 새 스킬 추가

1. `skills/<skill-name>/SKILL.md` 생성
2. `skills/<skill-name>/README.md` 추가
3. 긴 참고 내용은 `skills/<skill-name>/reference/`에 정리
4. 이 `README.md`의 스킬 목록에 등록
5. 계정, PAT, 개인 경로 같은 값은 커밋하지 않기
