---
name: work-manager
description: GitHub Issue URL/번호 또는 GitHub PR URL을 받아 작업을 분석하고 적절한 전문 에이전트로 라우팅하는 오케스트레이터. 상태는 GitHub 라벨·PR body·커밋 메시지로 추적한다 (operations.md §2.3 — work_history 미도입). 사용자가 flowmeet-web 작업을 Issue URL, #N, PR URL 로 시작할 때 로드.
metadata:
  allowed-tools:
    - AskUserQuestion
    - Agent
    - Skill
    - Read
    - Write
    - Edit
    - Bash(git config:*)
    - Bash(git fetch:*)
    - Bash(git checkout:*)
    - Bash(git branch:*)
    - Bash(git status:*)
    - Bash(gh issue view:*)
    - Bash(gh issue edit:*)
    - Bash(gh pr view:*)
    - Bash(gh pr create:*)
    - Bash(gh pr merge:*)
    - Bash(gh api:*)
    - Bash(find:*)
    - Bash(ls:*)
    - Bash(date:*)
---

# work-manager — flowmeet-web

GitHub Issue 또는 PR URL을 받아 **유형 판정 → 하위 flow 라우팅 → 전문 에이전트 호출**을 조율한다. 모든 단계는 **투명성 원칙**을 따른다. Jira는 쓰지 않는다 (GitHub Issues/Projects로 대체).

> **work_history 미도입**: api 리포와 달리 flowmeet-web은 작업 로그 폴더(`.claude/work_history/`)를 두지 않는다. 결정은 [`docs/axis-decisions/`](../../../docs/axis-decisions/), 변경 이유는 커밋/PR, 구조·규약은 [CLAUDE.md](../../../CLAUDE.md)로 분산. (사유: [operations.md §2.3](../../../docs/axis-decisions/operations.md))

## 인수

```
https://github.com/timonPark/flowmeet-web/issues/123
#123
https://github.com/timonPark/flowmeet-web/pull/456
```

## 투명성 원칙 `[전 단계 필수]`

work-manager 및 이후 호출되는 **모든 전문 에이전트**는 아래 규칙을 반드시 따른다.

### 단계 시작 시 출력

```
▶ [단계명] {하려는 작업 한 줄 설명}
```

예시: `▶ [이슈 조회] GitHub #42 본문·라벨 수집 중`

### 단계 완료 시 출력

```
✅ [단계명] {완료 결과 한 줄 요약}
```

예시: `✅ [이슈 조회] 유형=FEAT, phase-1 라벨, 레이어링 규칙 추가 요청`

### 병목 발생 시 즉시 보고

```
🚧 [병목] {상황 설명} — 어떻게 진행할까요?
```

> 침묵 금지. "뭔가 돌고 있다"보다 "지금 X 단계에서 Y 때문에 멈춤"이 항상 우선.

### 출력 민감정보 금지 `[web-security §3]`

투명성 출력(`▶`/`✅`/`🚧`)과 PR body·커밋 메시지에 **PII·토큰·음성 원문·회의 세부 내용**이 섞여서는 안 된다. 특히 `gh issue view`·`gh pr view` 결과를 그대로 로그에 넣지 말 것. 식별자(이슈 번호·사용자 id·회의 id) 중심으로 표현하고, free-form 발췌가 불가피하면 [`maskSensitive()`](../web-security/SKILL.md#3-pii-마스킹-상세-8-완결) 통과 후 사용.

## 사전 준비

- `git config user.name` 실행 값을 `{assignee}`로 사용한다.
- `GH_REPO=timonPark/flowmeet-web` (고정). PR·이슈 명령에 `--repo` 플래그 전달 또는 `gh` 기본 remote가 origin.

## 입력 유형 감지 `[최우선 — 이후 파일 로드 결정]`

### 입력 정규화 (적용 순서)

1. 양 끝 공백 제거.
2. 끝의 `/`, `#`, `?...`, `#...` 제거 (trailing slash·query·fragment).
3. 호스트명·경로는 **대소문자 무시**로 비교. 번호는 정수로 파싱(`parseInt`), 실패하면 "판정 불가".

### 패턴 매칭

| 입력 패턴 (정규식) | 판단 | 다음 액션 |
|---|---|---|
| `(?i)github\.com/timonPark/flowmeet-web/pull/(\d+)` | GitHub PR URL | `.claude/skills/work-manager/_pr-flow.md` 를 Read → 절차 실행 |
| `(?i)github\.com/timonPark/flowmeet-web/issues/(\d+)` 또는 `^#(\d+)$` | GitHub Issue URL/번호 | `.claude/skills/work-manager/_issue-flow.md` 를 Read → 절차 실행 |
| 그 외 (저장소 불일치·숫자 아님·빈 입력) | 판정 불가 | 사용자에게 어떤 URL/번호인지 되묻는다 (추측 금지) |

### 추가 검증 (Issue·PR 공통)

- 번호가 정수가 아니면 즉시 되묻기.
- `gh issue view` 또는 `gh pr view` 호출 결과 404면 "번호를 다시 확인해 주세요".
- 호스트가 다른 저장소면 "이 리포(`timonPark/flowmeet-web`)의 번호만 처리합니다"로 거절.

> 라우팅 매트릭스는 `.claude/skills/work-manager/_REFERENCE.md` 참조.

## 라우팅 목적지 (전문 에이전트)

flow 파일이 판단한 **작업 유형**에 따라 `.claude/agents/<name>.md` 에 정의된 전문 에이전트 중 하나를 Claude Code가 **Read 후 직접 실행**(Agent 툴 호출 금지, 1-depth 유지)한다.

| 단계 | 에이전트 | 호출 조건 |
|---|---|---|
| 공통 선행 | [`issue-kickoff`](../../agents/issue-kickoff.md) | 모든 Issue 경로 |
| 접근 방식 탐색 | [`brainstormer`](../../agents/brainstormer.md) | 복잡 FEAT·REFACTOR·PERF (복잡도 필터 통과 시) |
| 플랜 작성 | [`plan-writer`](../../agents/plan-writer.md) | 대부분 작업 (DOC/간단 CHORE 제외) |
| 신규 기능 / 테스트 | [`feat-developer`](../../agents/feat-developer.md) | FEAT, TEST |
| 버그 수정 | [`fix-developer`](../../agents/fix-developer.md) | FIX |
| 리팩토링·성능 | [`refactor-developer`](../../agents/refactor-developer.md) | REFACTOR, PERF |
| 문서·하네스 | [`doc-updater`](../../agents/doc-updater.md) | DOC, CHORE(하네스), ANALYSIS(문서) |
| PR 리뷰 체크 | [`pr-review-check`](../../agents/pr-review-check.md) | PR URL 입력 (DEPLOY-REVIEW) |

> 흐름: `issue-kickoff → brainstormer → plan-writer → 전문 에이전트 → PR → codex-review-triage → 사용자 승인 → merge` (복잡도 필터 통과 지점에서 brainstormer/plan-writer skip 가능).
>
> 상세 매핑과 skip 기준은 [`_REFERENCE.md`의 type ↔ 전문 에이전트 라우팅](./_REFERENCE.md#type--전문-에이전트-라우팅) 및 `_issue-flow.md` Step 3.3/3.5 참조.

## 작업 흐름 요약

```
입력 (URL / #N / PR URL)
  │
  ▼ 유형 감지
  ├─ Issue → _issue-flow.md (브랜치 생성·라벨 전환·전문 에이전트 호출·PR 생성)
  └─ PR    → _pr-flow.md    (PR 분석·codex 리뷰 필수·사용자 승인 후 머지)
  │
  ▼ 모든 단계 투명성 원칙 적용
  ▼ 상태는 GitHub 라벨 + PR body + 커밋 메시지로 추적 (work_history 미도입)
  ▼ 검증
```

## codex 외부 리뷰 `[PR 머지 전 필수]`

[operations.md §2.2](../../../docs/axis-decisions/operations.md) 결정으로 **모든 PR 의 머지 전 흐름의 일부**. 선택적 아님.

- PR 생성 직후 `_pr-flow.md` 가 [`codex-review-triage`](../codex-review-triage/SKILL.md) 를 자동 로드.
- `codex` CLI 미설치 시 skip 하지 않고 사용자에게 설치 안내.
- 산출물: `codereview/<YYYYMMDD>/<slug>.md` — `.gitignore` 로 로컬 전용.

## PR 머지 규칙 (상위 CLAUDE.md 재강조)

`_pr-flow.md`가 PR 머지 단계에 도달해도 **사용자의 명시적 승인 없이 `gh pr merge` 를 실행하지 않는다** (CLAUDE.md "PR·머지 가드레일" 참조). codex 리뷰 통과·`npm run lint` + `tsc --noEmit` + `npm test` 통과·CI 그린만으로는 머지 조건 성립 아님. **머지 방식은 squash 고정**.

## 검증

라우팅을 마친 뒤 아래를 확인한다. 실행 모드별로 요구 수준이 다르다.

| # | 조건 | 적용 범위 |
|---|---|---|
| 1 | 선택한 전문 에이전트가 `.claude/agents/` 에 존재하거나, 미등록 상태를 **투명성 포맷으로 "manual" 분기 선언** | 모든 라우팅 |
| 2 | GitHub 라벨 전환이 **집합 비교로 검증** 되었음 (`status:*` 라벨이 정확히 한 개) | Issue·PR 머지 모두 |
| 3 | 이슈/PR 유형 판정 근거(라벨·본문 키워드·제목)를 **PR body 또는 커밋 메시지에 한 줄로 남겼는가** | 모든 라우팅 |
| 4 | PR 머지 지점이면 사용자 명시 승인이 대화에 남았는가 | PR 머지 단계 |
| 5 | 투명성 출력과 커밋/PR 본문에 민감정보가 섞이지 않았는가(필요 시 `maskSensitive()` 통과) | 모든 라우팅 |
| 6 | PR 머지 직전 codex-review-triage 산출물이 존재하는가 (`codereview/<YYYYMMDD>/<slug>.md`) | PR 머지 단계 |

하나라도 실패하면 고치고 다시 검사한다. 전부 통과하면 라우팅 결과와 근거를 사용자에게 보고한다.

> 상세 flow(#11) 포팅 전 **stub 수동 분기 모드**는 1번을 "manual 선언"으로 완화한다 — 자동 라우팅은 에이전트 복제 완료 후.

## 금지 사항

- ❌ Jira·외부 티켓 시스템 참조. 이 리포는 GitHub Issues/Projects만 사용.
- ❌ 유형을 모호한 상태로 라우팅 (추측 금지 — 사용자에게 되묻는다).
- ❌ 투명성 포맷 생략 — "⋯ 작업 중" 같은 모호한 메시지로 침묵하지 않는다.
- ❌ PR 머지를 사용자 명시 승인 없이 실행.
- ❌ `.claude/work_history/` 폴더 생성 — operations.md §2.3 가 명시적으로 미도입.
- ❌ codex-review-triage skip 하고 머지 (operations.md §2.2 로 필수화).
- ❌ GitHub `Auto-merge` / "Merge when ready" 체크.
- ❌ Squash 외 머지 방식 (merge commit·rebase 금지).

## 관련

- `_REFERENCE.md` — type ↔ 브랜치 매핑, 라우팅 매트릭스, 명명 규약
- `_issue-flow.md` — GitHub Issue 입력 처리
- `_pr-flow.md` — GitHub PR 입력 처리 (codex 필수)
- 상위 규칙: [ts-style](../ts-style/SKILL.md), [app-router-layering](../app-router-layering/SKILL.md), [nextjs16-reality-check](../nextjs16-reality-check/SKILL.md), [error-ux](../error-ux/SKILL.md), [data-fetching-guard](../data-fetching-guard/SKILL.md), [web-testing](../web-testing/SKILL.md), [web-security](../web-security/SKILL.md)
- 신규 축: [a11y](../a11y/SKILL.md), [i18n](../i18n/SKILL.md), [design-tokens](../design-tokens/SKILL.md), [perf-budget](../perf-budget/SKILL.md)
- 외부 코드 리뷰 흐름: [codex-review-triage](../codex-review-triage/SKILL.md) (포팅 예정 — #11 범위)
- 운영 규정 출처: [operations.md](../../../docs/axis-decisions/operations.md)
