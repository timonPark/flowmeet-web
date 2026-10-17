---
name: issue-kickoff
description: GitHub Issue #N 과 사전 수집된 baseBranch·branchType을 받아 (1) 이슈 조회 (2) slug 추출 (3) 브랜치 분기 를 수행하는 수행 에이전트. work-manager _issue-flow Step 1에서 호출된다. Agent 툴 호출 금지 — Claude Code가 Read 후 직접 실행.
metadata:
  allowed-tools:
    - Read
    - Bash(git fetch:*)
    - Bash(git checkout:*)
    - Bash(git branch:*)
    - Bash(git status:*)
    - Bash(gh issue view:*)
    - Bash(date:*)
---

# issue-kickoff — 이슈 조회 + 브랜치 분기

work-manager `_issue-flow.md` Step 1에서 호출. **Agent 툴로 호출하지 않는다** — Claude Code가 이 파일을 Read 후 **직접 실행**한다. (1-depth 유지)

## 입력

| 변수 | 설명 |
|---|---|
| `{N}` | GitHub Issue 번호 (정수) |
| `{baseBranch}` | `main` / `develop` / `release` 중 하나 (Step 0에서 수집) |
| `{branchType}` | `feature` / `fix` / `chore` / `refactor` / `docs` / `test` (Step 0에서 수집) |

## 수행

### 1. 이슈 조회

```bash
gh issue view {N} --repo timonPark/flowmeet-web \
  --json number,title,body,labels,state,url
```

- 404 → "이슈 #{N}이 존재하지 않습니다. 번호를 다시 확인해 주세요." 로 되묻고 중단.
- `state == CLOSED` → "#{N}은 이미 종료된 이슈입니다. 작업을 계속할까요?"로 확인.
- 결과에서 `title`, `labels[].name`, `state`, `url` 보존.

### 2. slug 추출 `[결정적 알고리즘]`

제목에서 **소문자·하이픈·30자 이내** slug 조립. 아래 순서대로 적용 — 여러 호출 간 동일 결과가 나와야 한다.

1. **대괄호 접두어 제거**: 제목 앞의 `[...]` 블록 모두 제거 (`[P2-2]`, `[P1-1]`, `[fix]` 등). 안에 있는 내용은 slug에 들어가지 않는다.
2. **ASCII 외 문자 처리**: 한글 등은 **영문 요약으로 교체**. 교체가 애매하면 그 토큰을 **제거**(음차 금지). 결과가 비면 사용자에게 slug 입력을 요청.
3. 괄호·특수문자(`().,!?/\&`)·공백 → **하이픈**.
4. 소문자화.
5. 연속 하이픈은 1개로 축약.
6. 양 끝 하이픈 제거.
7. 30자 초과 시 **뒤쪽 토큰부터** 하이픈 경계로 잘라낸다 (중간이 아니라 토큰 단위).

예시 (두 문서가 동일 결과를 내야 함):
- `[P2-2] GitHub Issue flow (상태 라벨 전환, 브랜치 생성 통합)`
  → 접두어 제거: `GitHub Issue flow (상태 라벨 전환, 브랜치 생성 통합)`
  → 한글 토큰 제거 + 괄호·공백 → 하이픈 + 소문자 + 축약: `github-issue-flow`
- `[fix] 로그인 500 에러`
  → 접두어 제거: `로그인 500 에러`
  → 한글 요약 교체: `login error`
  → 하이픈화·축약: `login-500-error`

→ 결과를 `{slug}`에 저장. 결과가 비면 사용자에게 수동 slug를 요청.

### 3. 브랜치 조립 + 분기

```bash
BRANCH="{branchType}/{N}-{slug}"

# 3-1. dirty worktree 보호 — 변경사항이 있으면 중단하고 사용자에게 확인
if [ -n "$(git status --porcelain)" ]; then
  echo "작업 중 변경사항이 있습니다. stash/commit 후 재시도하거나 사용자 확인이 필요합니다." >&2
  git status --short >&2
  exit 1
fi

# 3-2. 원격 최신화 (로컬 base 브랜치는 체크아웃하지 않음 — origin/{baseBranch}에서 바로 분기)
git fetch origin "{baseBranch}"

# 3-3. 분기 또는 전환
if git show-ref --verify --quiet "refs/heads/${BRANCH}"; then
  git checkout "${BRANCH}"
else
  git checkout -b "${BRANCH}" "origin/{baseBranch}"
fi

# 3-4. 결과 확인
HEAD_SHA=$(git rev-parse --short HEAD)
echo "Branch: ${BRANCH}, HEAD: ${HEAD_SHA}, Base: {baseBranch} (origin/{baseBranch})"
```

> 로컬 `main`·`develop` 등을 전환·pull 하지 않는다 — `origin/{baseBranch}`를 fetch 후 바로 거기서 새 브랜치를 만든다. 사용자가 선택한 base가 정확히 반영된다.

### 4. 결과 보고 (투명성 포맷)

```
▶ [issue-kickoff] 이슈 #{N} 조회 및 브랜치 분기
✅ [issue-kickoff] 브랜치: {branch}, HEAD: {sha}, base: {baseBranch}
   · 제목: {title — 이슈 title 그대로. 민감 내용이 섞여 보이면 요약으로 교체}
   · 라벨: {labels 배열}
   · state: {OPEN|CLOSED}
```

## 호출자 (work-manager) 책임

issue-kickoff 반환 후 호출자가 수행:

1. **status 라벨 전환** (`_issue-flow.md` Step 2 — `status:todo` → `status:in-progress`, 리포에 라벨 세트가 설정돼 있을 때)
2. brainstormer → plan-writer → 전문 에이전트 핸드오프 (복잡도별 skip 가능)

> **work_history 파일 생성 책임은 없음** — flowmeet-web은 `.claude/work_history/` 를 두지 않는다 ([operations.md §2.3](../../docs/axis-decisions/operations.md)). 상태는 라벨 + PR body + 커밋 메시지로 추적.

## 금지

- ❌ Agent 툴로 호출 (반드시 Claude Code가 직접 Read·실행).
- ❌ 브랜치 전환 전에 `git stash` 없이 작업 중인 변경 유실.
- ❌ 이슈 본문(`body`)을 로그·보고 메시지에 그대로 노출 — 요약하거나 생략.
- ❌ 사용자 확인 없이 CLOSED 이슈에서 작업 재개.

## 관련

- [work-manager/_issue-flow.md](../skills/work-manager/_issue-flow.md) — 호출자
- [work-manager/_REFERENCE.md](../skills/work-manager/_REFERENCE.md) — 명명 규약
- [web-security](../skills/web-security/SKILL.md) — `maskSensitive()` 사용 기준
