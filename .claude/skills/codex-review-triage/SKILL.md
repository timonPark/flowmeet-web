---
name: codex-review-triage
description: PR을 codex CLI로 2차 리뷰받고, Claude가 지적사항의 타당성을 평가(triage)해 유효한 항목만 반영하는 흐름. PR 생성 후 머지 전에 로드. operations.md §2.2 로 모든 PR 머지 전 필수화. 사람 리뷰를 대체하지 않는 보조 레인.
---

# Codex Review Triage

PR을 외부 리뷰어(**codex CLI**)에게 2차 리뷰받고, Claude가 각 지적을 원문 대조해 **유효/부분유효/반려**로 분류한 뒤 유효한 것만 반영한다. 사람 리뷰와 병행하는 보조 레인.

> **flowmeet-web 에서는 선택이 아니라 필수** — `work-manager/_pr-flow.md` Step 3 가 자동 로드. 근거: [operations.md §2.2](../../../docs/axis-decisions/operations.md).

## 전제조건

- 로컬에 codex CLI 설치 (`which codex`). 미설치 시 **skip 하지 않고** 사용자에게 설치 안내 (operations.md §2.2 "전제조건").
- 현재 브랜치에 PR이 열려 있고 base 브랜치가 결정돼 있음(보통 `main`).
- `codereview/` 디렉터리는 **로컬 전용 산출물** — `.gitignore`에 포함돼 있어 git 추적 대상이 아니다(원격에 push되지 않음).

## 전체 흐름 (6단계)

```
1. 리뷰 생성 (codex exec)
2. 산출물 저장 (codereview/<YYYYMMDD>/<slug>.md, 로컬 전용)
3. Triage (Claude가 각 지적을 원문 대조·평가)
4. 선택적 반영 (유효 판정 항목만)
5. 서버 검증 (반영 후 필수 — 레벨 결정 후 실행)
6. 재리뷰 (regress 체크, 선택)
```

## 1. 리뷰 생성 — codex exec

### diff 준비

```bash
BASE=main
git diff ${BASE}...HEAD > /tmp/pr.diff
```

### 프롬프트에 반드시 포함할 억제 지시

사용자 환경의 로컬 codex 스킬이 자동 로드돼 산출물을 엉뚱한 경로(예: `code_review/<JIRA-KEY>/`)에 쓰려 한다. **프롬프트에서 명시적으로 억제**한다.

- `다른 스킬·로컬 workflow 로드 금지 (예: branch-code-review-report)`
- `파일 쓰기·git 수정·shell 실행 금지` — stdout의 Markdown 본문만 응답
- `Jira 등 외부 티켓 시스템 참조 금지`
- 응답은 **Markdown 본문만** (서두·메타 주석 없음)
- 지적사항에는 **`[severity: blocker|high|medium|low]`** 태그 필수

### 실행

```bash
(cat <<'PROMPT'
너는 숙련된 TypeScript + React 19 + Next.js 16 (App Router) 코드 리뷰어다. 아래 diff는 flowmeet-web PR #<N> (브랜치 `<branch>`, base `<base>`) 변경이다.

중요 지침:
- **다른 스킬·로컬 workflow 로드 금지** (예: branch-code-review-report).
- **파일을 쓰거나 git을 수정하지 말 것**. shell 명령 실행 금지.
- **Jira·외부 티켓 시스템 참조·경로 금지**. 리포 로컬 산출물 경로만 상정.
- 응답은 **Markdown 본문만** 출력. 메타 주석·인삿말·서두 없음.
- Next.js 16 `cacheComponents: true` + `partialPrefetching: true` 환경 — Next 15 이하 전제의 fetch/cache 조언 금지.
- React 19.3 전제 — React 18 이하 전제의 Suspense/transition 조언 금지.

응답 섹션 (한국어, 이 순서 그대로):

# Code Review — PR #<N> <branch>

## 요약
(3~5문장: 변경 의도 + 리뷰 총평)

## 강점
- bullet

## 지적사항
각 항목에 **[severity: blocker|high|medium|low]** 태그 필수. 파일/섹션 명시, 근거 포함.
- [severity: ...] <제목>: <설명>

리뷰 축 (해당 항목만 평가):
- TypeScript 타입 안전성 (any/unknown 사용, discriminated union, zod schema 재사용)
- React 19.3 패턴 (use(), Server Actions, Suspense, transition)
- Next.js 16 App Router (Server/Client 경계, cacheComponents, 'use cache' + cacheLife/cacheTag, Server Actions)
- 데이터 페칭 (serverClient/browserClient/publicApiClient 사용, React Query v5 패턴, 401 refresh)
- 상태 관리 (RQ vs Zustand 경계, 폼 상태 중복 금지)
- 폼/검증 (react-hook-form + zod, FormActionResult 패턴)
- 에러 처리 (ApiError·codeToMessage·boundary 배치·reportError)
- 접근성 (Radix primitives 활용, aria-label, focus-visible)
- i18n (next-intl 네임스페이스, 하드코딩 한국어)
- 디자인 토큰 (@theme CSS variables, oklch 의미 토큰 — Tailwind 기본 scale/dark: 금지)
- 번들/성능 (클라이언트 번들, Core Web Vitals)
- 테스트 (Vitest/MSW/Playwright 패턴, RSC 유닛 금지)
- 보안 (토큰 저장·전달, PII 마스킹, 민감정보 로깅 금지)

## 질문·제안
- bullet

## 체크리스트
- [ ] 후속 확인 항목

컨텍스트:
- 리포 스택: Next.js 16.4 (App Router, cacheComponents) + React 19.3 + TypeScript 5 + Tailwind 4
- Node.js 20.9+, 권장 22 LTS
- PR 변경 범위: <한 줄 요약>
- 상위 규칙 문서:
  - `.claude/skills/nextjs16-reality-check/SKILL.md`
  - `.claude/skills/ts-style/SKILL.md`
  - `.claude/skills/app-router-layering/SKILL.md`
  - `.claude/skills/data-fetching-guard/SKILL.md`
  - `.claude/skills/error-ux/SKILL.md`
  - `.claude/skills/web-testing/SKILL.md`
  - `.claude/skills/web-security/SKILL.md`
  - `.claude/skills/a11y/SKILL.md`
  - `.claude/skills/i18n/SKILL.md`
  - `.claude/skills/design-tokens/SKILL.md`
  - `.claude/skills/perf-budget/SKILL.md`

---DIFF---
PROMPT
cat /tmp/pr.diff
) | codex exec -s read-only - 2>&1 | tee /tmp/codex-review.log
```

- `-s read-only` **고정** — codex가 로컬 파일을 쓰지 못하게 함.
- stdin으로 `---DIFF---` 섹션 포함해 넘기면 codex가 diff를 그대로 리뷰 대상으로 삼는다.

## 2. 산출물 저장

### 경로 규약

```text
codereview/<YYYYMMDD>/<slug>.md
```

- `YYYYMMDD`: 리뷰 실행일(로컬 기준).
- `slug`: 브랜치명에서 `/`를 `-`로 치환. 예: `docs/11-common-agents-codex` → `docs-11-common-agents-codex`.
- 같은 브랜치를 같은 날 재리뷰하면 **덮어쓴다**. 다른 날 리뷰는 날짜 폴더가 달라 자연히 분기.
- **Jira 키·외부 경로 사용 금지** — 리포 로컬 경로만 허용.
- **git 추적 금지**: `codereview/`는 `.gitignore`에 포함. 원격에 push하지 않는다(개인/팀 로컬 아카이브).

### 본문 추출

codex exec 로그는 헤더·tokens 요약 등이 섞인다. `# Code Review`부터 `tokens used` 전까지만 추출.

```bash
DATE=$(date +%Y%m%d)
SLUG=$(git branch --show-current | tr '/' '-')
OUT="codereview/${DATE}/${SLUG}.md"
mkdir -p "$(dirname "$OUT")"

# 리뷰 본문 추출 (첫 '# Code Review' 블록 ~ 'tokens used' 직전)
BODY=$(sed -n '/^# Code Review/,/^tokens used$/p' /tmp/codex-review.log | sed '$d')

{
  echo "---"
  echo "pr: <N>"
  echo "branch: $(git branch --show-current)"
  echo "base: ${BASE}"
  echo "reviewer: $(codex --version 2>&1 | head -1)"
  echo "generated_at: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
  echo "---"
  echo
  echo "$BODY"
} > "$OUT"
```

### Frontmatter 필수 필드

| 필드 | 설명 |
| --- | --- |
| `pr` | GitHub PR 번호 |
| `branch` | 리뷰 대상 브랜치 |
| `base` | diff 기준 브랜치 |
| `reviewer` | codex 버전 문자열 |
| `generated_at` | ISO8601 UTC |

## 3. Triage — Claude가 각 지적을 평가

**Claude는 codex 결과를 그대로 신뢰하지 않는다.** 각 항목을 원문과 대조한다.

### 평가 축

| 판정 | 기준 |
| --- | --- |
| **유효 ✓** | 지적이 원문/코드에서 사실로 확인되고, 수정이 필요 |
| **부분유효 △** | 지적은 사실이지만 상위 규칙·다른 문서가 이미 처리 중이거나 범위가 과장됨 |
| **반려 ✗** | 환각(실제 코드에 없는 라인/API를 지적), 사실 오인, heuristic 과잉 적용 — 특히 Next 15 이하 전제의 조언은 반려 (nextjs16-reality-check 참조) |

### 체크해야 할 항목

- **원문 대조**: 지적된 파일·섹션·라인을 Read로 확인. 인용된 심볼이 실제로 존재하는가?
- **상호 참조**: 상위 규칙(ts-style·app-router-layering 등)이 이미 커버하는지 — 중복 지적인지 확인.
- **Next 16 전제 체크**: codex가 Next 15 이하 전제로 조언하면 반려 또는 재해석.
- **severity 재분류**: codex가 과대·과소 평가하는 경우가 있다. Claude가 재조정.
- **놓친 지점**: codex가 발견 못한 중대 이슈(보안·토큰·접근성)가 있는지 사람 리뷰어 관점으로 보강.

### 보고 포맷

사용자에게 제시할 Triage 요약:

```markdown
## 전체 평가: ★★★★☆ (예시)

### 항목별 평가

#### 1. [codex severity] 제목 — **유효 ✓**
- 근거: `<파일>:<섹션>`에서 실제 확인
- 권장 액션: ...

#### 2. [codex severity] 제목 — **부분유효 △**
- 근거: ...
- 미반영 사유: 상위 규칙 X가 이미 커버

#### 3. [codex severity] 제목 — **반려 ✗**
- 근거: ...
- 사유: 환각 / 사실 오인 / Next 15 이하 전제
```

## 4. 선택적 반영

- **유효 판정 항목만 수정**. 부분유효·반려는 미반영.
- 수정 작업은 **TaskCreate로 추적** (지적 1건 = 태스크 1건 권장).
- 미반영 항목은 사유를 보고에 명시 — 사용자가 재확인·오버라이드할 수 있도록.
- 커밋은 **Step 5 서버 검증 통과 후**에만 push. 커밋 메시지에 Verified 라인을 포함(아래 §5 참조).

## 5. 서버 검증 (필수)

리뷰 지적을 반영한 뒤 **반드시 서버 측 검증**을 돌린다. 반영이 회귀를 유발하지 않았는지 확인하는 단계. 변경 범위에 따라 레벨을 고른다.

### 레벨 선택 기준 (operations.md §2.2 web 기준)

| 레벨 | 트리거 | 명령 | 시간 |
| --- | --- | --- | --- |
| **L0 — 문서/하네스** | `.claude/`·`CLAUDE.md`·`docs/` 만 변경 | 생략 가능 (`Verified: L0 skipped — doc-only change`) | — |
| **L1 — 표준** | `src/` 하위 코드 변경 | `npm run lint && npx tsc --noEmit && npm test` | ~30s–2min |
| **L2 — 전체** | 라우팅·보안·API 클라이언트·토큰 흐름 변경 | L1 + `npm run test:e2e` (Playwright 스모크) | ~2–5min |

### 명령 템플릿

```bash
# L0 — 문서/하네스만 바뀐 PR
# 검증 생략 — 커밋 메시지에 "Verified: L0 skipped — doc-only change" 명시

# L1 — 일반적인 코드 변경
npm run lint && npx tsc --noEmit && npm test

# L2 — 라우팅/보안/API 클라이언트/토큰 흐름
npm run lint && npx tsc --noEmit && npm test && npm run test:e2e
```

### 실패 처리

- **lint 실패** → `npm run lint -- --fix` 로 자동 수정 가능한 항목부터 처리, 수동 수정은 지적을 재검토.
- **typecheck 실패** → 새로 추가한 타입이 상위 모듈과 불일치. 반영을 롤백하거나 상위 타입을 함께 수정.
- **test 실패** → 반영 커밋을 push하지 **않는다**. 원인을 수정하거나 반영을 롤백.
- **test:e2e 실패** → Playwright trace (`playwright-report/`) 확인. flaky 재시도로 숨기지 않음. 환경 이슈면 재시도, 코드 문제면 반영 롤백.
- **원인이 반영 그 자체이면 Triage 결과를 재평가** — "유효" 판정이 과했을 수 있다.

### 검증 결과 기록

반영 커밋 메시지에 검증 레벨과 결과를 포함한다.

```text
chore(review): address codex findings (<severity-count>)

- <항목 1> (§<섹션>)
- <항목 2> (§<섹션>)

Verified: L1 — npm run lint && tsc --noEmit && npm test (passed)
Refs codex review: codereview/<YYYYMMDD>/<slug>.md
```

> 검증을 **생략했다면 그 사유도 커밋 메시지에 명시**한다 (예: `Verified: L0 skipped — doc-only change`).

## 6. 재리뷰 (선택)

- blocker/high를 반영하고 서버 검증을 통과한 뒤 regression 체크용으로 Step 1~2 재실행.
- 같은 날이면 산출물이 **덮어쓰기**된다 — 이전 리뷰를 보존하고 싶으면 복사해 둔다(원격 push 안 됨).

## 금지 사항

- ❌ **Jira 키·외부 티켓 참조**: codex에게 Jira 키·외부 URL을 넘기지 않는다. 산출물 경로에도 티켓 키를 쓰지 않는다. (이전 프로젝트의 `branch-code-review-report` 스킬이 Jira 기반 경로를 요구하지만, flowmeet-web에서는 **그 스킬 자체를 억제**한다.)
- ❌ **`-s workspace-write` 이상의 샌드박스**: codex가 파일을 쓰거나 git을 조작할 가능성을 차단해야 한다. 반드시 `-s read-only`.
- ❌ **거대 PR 리뷰**: diff 수백 줄을 넘기면 리뷰 품질이 급락한다. PR을 쪼갠다.
- ❌ **codex 결과를 그대로 수용·커밋**: Claude의 triage를 생략하지 않는다. 과장·환각이 섞여 있을 수 있다.
- ❌ **사람 리뷰 대체**: 이 흐름은 보조 레인. 프로덕션 코드는 사람 승인 필수.
- ❌ **codex 흐름 skip**: operations.md §2.2 로 필수. CLI 미설치 시 skip 하지 않고 설치 안내.

## 리뷰 체크리스트

- [ ] codex 호출에 `-s read-only` 적용했는가?
- [ ] 프롬프트에 "스킬 로드 금지, 파일 쓰기 금지, Jira 참조 금지" 명시했는가?
- [ ] 프롬프트에 Next 16 + React 19 전제 명시했는가?
- [ ] 산출물에 frontmatter(pr·branch·base·reviewer·generated_at) 포함했는가?
- [ ] 모든 지적을 원문 대조 후 평가했는가 (유효/부분유효/반려)?
- [ ] Next 15 이하 전제의 지적은 반려 또는 재해석했는가?
- [ ] 미반영 항목에 사유를 보고했는가?
- [ ] 반영 후 blocker·high가 전원 해소되었는가?
- [ ] **서버 검증(L0/L1/L2)을 실행하고 통과했는가?** 생략했다면 사유를 커밋 메시지에 명시했는가?
- [ ] 커밋 메시지에 `Verified:` 라인이 포함되었는가?
- [ ] `codereview/`가 로컬 전용으로 유지되는가 (`.gitignore` 반영, push 안 됨)?

## 관련 스킬·문서

- [nextjs16-reality-check](../nextjs16-reality-check/SKILL.md) — Next 16 전제 (리뷰 시 상호 참조 핵심)
- [ts-style](../ts-style/SKILL.md) — TypeScript 규칙
- [app-router-layering](../app-router-layering/SKILL.md) — App Router 레이어링
- [data-fetching-guard](../data-fetching-guard/SKILL.md) — fetch wrapper·캐시·RQ
- [error-ux](../error-ux/SKILL.md) — 에러 처리
- [web-testing](../web-testing/SKILL.md) — 테스트 패턴
- [web-security](../web-security/SKILL.md) — 토큰·PII 마스킹
- [a11y](../a11y/SKILL.md) · [i18n](../i18n/SKILL.md) · [design-tokens](../design-tokens/SKILL.md) · [perf-budget](../perf-budget/SKILL.md)
- `../../../CLAUDE.md` — 외부 코드 리뷰 섹션 (이 스킬의 포인터)
- [operations.md §2.2](../../../docs/axis-decisions/operations.md) — codex 필수화 결정
