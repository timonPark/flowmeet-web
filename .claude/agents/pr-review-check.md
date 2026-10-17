---
name: pr-review-check
description: GitHub PR #N 메타(title/labels/head/base/state/mergeState/reviews)를 수집하고 리뷰 점검 체크리스트(미답변 코멘트·사람 승인·codex 산출물·CI 상태)를 생성하는 수행 에이전트. work-manager _pr-flow Step 1·4에서 호출된다. Agent 툴 호출 금지 — Claude Code가 Read 후 직접 실행.
metadata:
  allowed-tools:
    - Read
    - Bash(gh pr view:*)
    - Bash(gh api:*)
    - Bash(gh issue view:*)
    - Bash(git branch:*)
    - Bash(git show-ref:*)
    - Bash(ls:*)
    - Bash(find:*)
    - Bash(date:*)
---

# pr-review-check — PR 메타 수집 + 리뷰 체크리스트

work-manager `_pr-flow.md` Step 1·4 에서 호출. **Agent 툴로 호출하지 않는다** — Claude Code가 Read 후 직접 실행한다. (1-depth 유지)

## 입력

| 변수 | 설명 |
|---|---|
| `{N}` | GitHub PR 번호 (정수) |

## 수행

### 1. 메타 수집 `[단일 호출]`

```bash
gh pr view {N} --repo timonPark/flowmeet-web --json \
  number,title,body,labels,headRefName,baseRefName,state,mergeStateStatus,mergeable,isDraft,reviews,url,author,statusCheckRollup
```

- 404 → "PR #{N}이 존재하지 않습니다. 번호를 다시 확인해 주세요."
- `state == MERGED` / `CLOSED` → 사용자에게 어떤 작업을 할지 확인 (회고·리뷰 조회만 허용).
- `isDraft == true` → "Draft PR입니다. 머지 전 Ready for review 전환 필요합니다." 로 안내.

### 2. 미답변 인라인 코멘트 조회 `[두 단계 로직]`

최상위 코멘트가 "답글 없음"인지 정확히 판정하려면 **모든 답글의 `in_reply_to_id` 집합**을 먼저 모아, 그 집합에 포함되지 않은 최상위 코멘트만 미답변으로 분류한다.

```bash
# 1) 모든 코멘트 수집
ALL=$(gh api repos/timonPark/flowmeet-web/pulls/{N}/comments)

# 2) 답글이 가리키는 "부모 id" 집합을 수집
REPLIED_PARENTS=$(echo "$ALL" \
  | jq '[.[] | select(.in_reply_to_id != null) | .in_reply_to_id] | unique')

# 3) 최상위 코멘트 중 "답글 받지 못한" 것만 추출 → 미답변 목록
echo "$ALL" | jq --argjson replied "$REPLIED_PARENTS" '
  [.[] | select(.in_reply_to_id == null)
       | select((.id | IN($replied[])) | not)
       | {id, path, line, user: .user.login, body}]
'
```

- 최상위 코멘트라도 **다른 코멘트가 답글로 가리키고 있으면 처리된 것**으로 간주.
- 결과를 Step 4-1 보고에 사용.
- **민감정보 보호**: `body` 필드를 그대로 보고 메시지에 넣지 말 것. 요약하거나 [`maskSensitive()`](../skills/web-security/SKILL.md#3-pii-마스킹-상세-8-완결) 통과.

### 3. 사람 리뷰 승인 수

```bash
APPROVED=$(gh pr view {N} --repo timonPark/flowmeet-web --json reviews \
  --jq '[.reviews[] | select(.state == "APPROVED")] | length')
echo "approved_count=${APPROVED}"
```

### 4. codex 2차 리뷰 산출물 확인 `[필수 — operations.md §2.2]`

산출물 경로 규약은 `codereview/<YYYYMMDD>/<branch-slug>.md` ([codex-review-triage §2](../skills/codex-review-triage/SKILL.md)). 날짜 폴더가 여러 개일 수 있으므로 **glob로 날짜 폴더를 포함해 탐색**한다.

```bash
BRANCH="{headRefName}"
SLUG=$(echo "$BRANCH" | tr '/' '-')
find codereview -mindepth 2 -type f -name "${SLUG}.md" 2>/dev/null
```

- 없으면 **`codex-review-triage` 흐름 수행이 필수** ([operations.md §2.2](../../docs/axis-decisions/operations.md)). 호출자에게 "codex 산출물 누락 — 머지 보류" 로 보고.
- 여러 날짜 폴더에 존재하면 최신 날짜(가장 큰 YYYYMMDD)를 선택.

### 5. mergeable·CI 상태

Step 1에서 이미 받은 `mergeStateStatus`·`mergeable`·`statusCheckRollup` 재사용. 추가 호출 불필요.

## 반환 (호출자에게 전달하는 요약)

**민감정보 처리**: 아래 자유 서술 필드는 [`maskSensitive()`](../skills/web-security/SKILL.md#3-pii-마스킹-상세-8-완결) 통과 후 반환한다.

- `title`, `body` (PR 본문), 코멘트 `body`, `author.login` (이메일 치환 시).
- 식별자(number·sha·refName·state·labels[].name)는 mask 생략.

```
PR #{N}
  title: {mask(title)}
  state: {OPEN|MERGED|CLOSED}
  head → base: {headRefName} → {baseRefName}
  mergeState: {CLEAN|DIRTY|BLOCKED|UNKNOWN}
  mergeable: {MERGEABLE|CONFLICTING|UNKNOWN}
  isDraft: {true|false}
  labels: [{labels[].name}]

리뷰 점검:
  미답변 인라인 코멘트: {count} ({파일: 사용자 간단 목록 — body는 mask(body)[:80]로 축약})
  사람 승인: {APPROVED count}
  codex 산출물: {경로 또는 "없음 — triage 필수"}
  CI 상태: {statusCheckRollup 요약 — SUCCESS|FAILURE|PENDING|NONE}
```

**Step 1에서 수집한 메타는 전체 보고에서 재사용**한다. 호출자가 Step 4에서 `gh pr view`를 다시 호출하지 않도록 핵심 상태(mergeStateStatus·mergeable·statusCheckRollup 요약)를 반환 포함.

## 호출자 (work-manager) 책임

pr-review-check 반환 후 호출자가 수행:

1. **유형 판정** (`_pr-flow.md` Step 2 — title 접두사/변경 범위 기준).
2. **codex-review-triage 호출** (`_pr-flow.md` Step 3 — operations.md §2.2 필수).
3. **사용자에게 리뷰 점검 결과 보고** (`_pr-flow.md` Step 4 — 투명성 포맷).
4. **머지 승인 대기** (`_pr-flow.md` Step 5 — 사용자 명시 "머지해줘" 전까지 대기).

> **work_history 플레이스홀더 작성 책임은 없음** — flowmeet-web은 `.claude/work_history/` 를 두지 않는다 ([operations.md §2.3](../../docs/axis-decisions/operations.md)).

## 금지

- ❌ Agent 툴로 호출 (반드시 Claude Code가 직접 Read·실행).
- ❌ PR body(`body`)를 로그·보고 메시지에 그대로 노출 — 요약 또는 생략.
- ❌ `gh pr merge` 를 이 에이전트에서 실행 — 머지는 `_pr-flow.md` Step 5 책임이고 **반드시 사용자 명시 승인 후**.
- ❌ CLOSED/MERGED PR에서 사용자 확인 없이 작업 재개.
- ❌ codex 산출물 없이 머지 가능 보고 (operations.md §2.2 필수).

## 관련

- [work-manager/_pr-flow.md](../skills/work-manager/_pr-flow.md) — 호출자
- [codex-review-triage](../skills/codex-review-triage/SKILL.md) — codex 산출물 생성 흐름
- [web-security](../skills/web-security/SKILL.md) — `maskSensitive()` 사용 기준
- CLAUDE.md "PR·머지 가드레일" — 사용자 명시 승인 원칙
