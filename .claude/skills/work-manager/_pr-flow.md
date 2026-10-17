# PR 경로 — GitHub PR 처리 flow

> GitHub PR URL 입력 시 실행. Issue URL이면 `_issue-flow.md` 로 이동.

PR URL을 받아 **메타 조회 → 유형 판정 → codex 리뷰 (필수) → 리뷰 체크 → 사용자 명시 승인 후 머지 → 완료 처리**를 수행한다. flowmeet-web 은 `.claude/work_history/` 를 두지 않으므로 ([operations.md §2.3](../../../docs/axis-decisions/operations.md)) 상태 추적은 **라벨 + PR body + 커밋 메시지** 로 한정한다.

---

## 입력 검증

| 조건 | 처리 |
|---|---|
| 번호가 정수가 아님 | 사용자에게 되묻기 (추측 금지) |
| `gh pr view` 404 | 번호 재확인 |
| PR 상태 `MERGED` / `CLOSED` | "이미 종료된 PR입니다. 어떤 작업을 할까요?" 확인 — 리뷰 조회·회고만 허용 |
| 호스트가 다른 저장소 | "이 리포(`timonPark/flowmeet-web`)의 번호만 처리합니다" |

---

## 전체 흐름 요약

```
[입력] PR URL or PR #N
  │
  ▼ Step 1 — pr-review-check 호출 (gh pr view 로 메타 수집)
  ▼ Step 2 — 유형 판정 (라벨/제목 접두사)
  ▼ Step 3 — codex-review-triage 호출 [필수 — operations.md §2.2]
  ▼ Step 4 — 리뷰 상태 점검 (codex 산출물 / 사람 승인 / 미답변 코멘트 / mergeState)
  ▼ Step 5 — 머지 승인 대기 → 사용자 명시 승인 받은 후 squash merge
  ▼ Step 6 — 완료 처리 (라벨 전환 / 로컬 브랜치 정리)
```

> api 리포 flow 의 **Step 3 (work_history 플레이스홀더)** 는 web 에서 제거. 상태는 라벨 + PR body + 커밋 메시지로 추적. Step 번호는 api 와 어긋남 — cross-reference 시 주의.

---

## Step 1 — pr-review-check 호출 `[Agent 툴 금지]`

**`.claude/agents/pr-review-check.md` Read 후 Claude Code가 직접 실행**한다. (1-depth 유지)

수행 요약:

1. `gh pr view {N} --repo timonPark/flowmeet-web --json ...` 로 메타 수집.
2. 수집 필드: `number`, `title`, `body`, `labels`, `headRefName`, `baseRefName`, `state`, `mergeStateStatus`, `mergeable`, `isDraft`, `reviews`, `url`, `author`, `statusCheckRollup`.
3. 결과를 다음 Step에 전달.

### 투명성 포맷

```
▶ [Step 1] PR #{N} 메타 조회 (gh pr view --json ...)
✅ [Step 1] title=..., state=OPEN, head=..., base=main, mergeState=CLEAN, isDraft=false
```

---

## Step 2 — 유형 판정 `[결정적, 우선순위 명시]`

### 판정 우선순위

1. **PR title 접두사 우선** (가장 결정적인 신호). `<type>(scope): ...` 패턴에서 `type` 추출.
2. title에 접두사가 없거나 모호(`WIP:` 등)하면 **변경 범위 기반**.
3. 둘 다 모호하면 `DEPLOY-REVIEW` 로 분류하고 사용자에게 확인.

### 접두사 매트릭스 (1순위)

| title 접두사 | 판정 `type` |
|---|---|
| `feat` | `FEAT` |
| `fix`, `hotfix` | `FIX` |
| `refactor` | `REFACTOR` |
| `perf` | `PERF` |
| `docs` | `DOC` |
| `test` | `TEST` |
| `chore` | `CHORE` |

### 변경 범위 기반 (2순위)

| 변경 범위 | 판정 `type` |
|---|---|
| `.md` + `.claude/` + `docs/` 만 | `DOC` |
| `src/**/*.test.ts`·`src/**/*.test.tsx`·`e2e/` 만 | `TEST` |
| `.github/`·`package.json`·`tsconfig.json`·설정 파일만 | `CHORE` |
| 그 외 (`src/` 실행 코드 변경 포함) | `DEPLOY-REVIEW` 로 두고 사용자 확인 |

### 충돌 처리

- 접두사와 변경 범위가 상반(`chore(harness): ...` 인데 `src/` 코드 변경 포함) → **title 접두사 우선 + PR body "충돌" 섹션에 한 줄 기록**.
- `title` 접두사 자체가 없으면 2순위로 이동.

> type ↔ 브랜치 type 매핑은 [`_REFERENCE.md`](./_REFERENCE.md#type--브랜치-type-매핑) 참조. 판정 근거(접두사·변경 범위·충돌 여부)는 **PR body 에 한 줄로 남긴다** (`gh pr edit --body-file -`).

### 투명성 포맷

```
▶ [Step 2] 유형 판정
✅ [Step 2] type=FEAT (title "feat(harness):" 접두사 매칭)
```

---

## Step 3 — codex-review-triage 호출 `[필수 — operations.md §2.2]`

[operations.md §2.2](../../../docs/axis-decisions/operations.md) 결정으로 **모든 PR 의 머지 전 흐름의 일부**. 선택적 아님.

### 수행 요약

**`.claude/skills/codex-review-triage/SKILL.md` Read 후 Claude Code가 직접 실행** (1-depth 유지). 포팅은 #11 범위.

1. `codex` CLI 설치 확인 — 없으면 skip 하지 않고 **사용자에게 설치 안내** 후 `🚧` 보고.
2. codex exec (`-s read-only` 샌드박스) 로 리뷰 생성.
3. 산출물 저장: `codereview/<YYYYMMDD>/<branch-slug>.md`.
   - `branch-slug`: 브랜치명의 `/` 를 `-` 로 치환.
   - `.gitignore` 로 로컬 전용 — 원격 push 안 됨.
4. **triage 분류** — 각 지적을 원문 대조로 **유효 / 부분유효 / 반려** 로 분류 (Claude 가 수행, codex 결과 무비판 수용 금지).
5. 유효 항목은 커밋으로 반영.
6. 서버 검증 — L0/L1/L2 선택 후 명령 실행:
   - **L0**: `.claude/`·`CLAUDE.md`·`docs/` 만 변경 → `Verified: L0 skipped — doc-only change`
   - **L1**: `src/` 하위 코드 변경 → `npm run lint && npx tsc --noEmit && npm test`
   - **L2**: 라우팅·보안·API 클라이언트·토큰 흐름 변경 → L1 + `npm run test:e2e` (Playwright 스모크)
7. (선택) 재리뷰 — regression 체크.

### 산출물 frontmatter 필수 필드

```yaml
---
pr: "{PR URL}"
branch: "{headRefName}"
base: "{baseRefName}"
reviewer: codex
generated_at: "{YYYY-MM-DD HH:mm}"
---
```

### 투명성 포맷

```
▶ [Step 3] codex-review-triage — codex exec + triage
🚧 [Step 3] codex CLI 미설치 — 설치 안내 필요
✅ [Step 3] codereview/20261017/docs-10-work-manager-port.md (유효 2 / 부분유효 1 / 반려 3)
```

### 금지 사항 (operations.md §2.2)

- codex 결과를 **triage 없이** 그대로 수용·커밋 금지.
- `-s workspace-write` 이상 샌드박스 금지.
- 거대 PR 리뷰 금지 — 수백 줄 넘기면 PR 쪼갬.
- Jira 키·외부 티켓 참조 금지.
- 사람 리뷰 대체 금지 — 보조 레인.

---

## Step 4 — 리뷰 상태 점검 `[사용자 보고용]`

머지 **직전** 반드시 수행. 하나라도 미해결이면 사용자에게 보고하고 머지를 보류한다.

### 4-1. 미답변 인라인 코멘트

```bash
gh api repos/timonPark/flowmeet-web/pulls/{N}/comments \
  --jq '.[] | select(.in_reply_to_id == null) | {id, path, user: .user.login, body: (.body[:120])}'
```

- 답글이 없는 인라인 코멘트가 있으면 "반영 또는 미반영 사유 댓글이 먼저 필요합니다"로 알림.

### 4-2. 사람 리뷰 승인 상태

```bash
gh pr view {N} --repo timonPark/flowmeet-web --json reviews \
  --jq '[.reviews[] | select(.state == "APPROVED")] | length'
```

- 조직 정책에 따라 1건 이상 요구 — 현재 flowmeet-web은 **최소 1명 승인 권장**(강제는 아님).

### 4-3. codex 산출물 존재 확인 `[Step 3 결과 재확인]`

경로 규약은 `codereview/<YYYYMMDD>/<branch-slug>.md`:

```bash
BRANCH="{headRefName}"
SLUG=$(echo "$BRANCH" | tr '/' '-')
find codereview -mindepth 2 -type f -name "${SLUG}.md" 2>/dev/null
```

- 산출물이 **없으면 Step 3 재실행** (operations.md §2.2 로 필수).
- 여러 날짜 폴더가 있으면 최신 날짜의 파일을 선택.

### 4-4. mergeable·CI 상태 `[Step 1 메타 재사용]`

**Step 1 (pr-review-check)이 이미 수집한 값을 재사용**한다. `gh pr view` 를 다시 호출하지 않는다 — 중복 호출은 호출 간 상태 불일치 위험만 늘린다.

- `mergeStateStatus == CLEAN` 그리고 `mergeable == MERGEABLE` 이어야 머지 가능.
- `statusCheckRollup`에 FAILURE가 있으면 머지 보류.
- 수 분 이상 지나서 상태가 바뀔 가능성이 크면 Step 1부터 재실행.

### 투명성 포맷 예시

```
▶ [Step 4] 리뷰 상태 점검
✅ [Step 4]
  · 미답변 인라인 코멘트: 0
  · 사람 승인: 1
  · codex 산출물: codereview/20261017/docs-10-work-manager-port.md
  · mergeState: CLEAN / mergeable
```

---

## Step 5 — 머지 승인 대기 `[CLAUDE.md 핵심 규칙]`

**사용자가 해당 PR에 대해 명시적으로 "머지해줘" 라고 지시한 경우에만** 머지 실행. 다음은 머지 조건이 **아니다**:

- codex 리뷰 통과만으로
- `npm run lint` + `tsc --noEmit` + `npm test` 통과만으로
- CI 그린만으로
- 이전에 다른 PR에서 받았던 머지 승인

### 모호할 때

"어떤 PR을 머지할까요?"로 되묻기 (여러 PR이 열려 있거나 범위 불명확한 경우).

### 승인 받은 뒤 — 머지 실행

**머지 방식은 squash 고정** ([operations.md §2.1](../../../docs/axis-decisions/operations.md)). `--merge` / `--rebase` 금지.

```bash
# 1. 원격 squash merge + 원격 브랜치 삭제
gh pr merge {N} --repo timonPark/flowmeet-web --squash --delete-branch

# 2. 로컬 main 최신화
git checkout main && git pull

# 3. 로컬 head 브랜치 삭제 (존재할 때만, -D 강제 삭제 금지)
HEAD_BRANCH={headRefName}
if git show-ref --verify --quiet "refs/heads/${HEAD_BRANCH}"; then
  git branch -d "${HEAD_BRANCH}" || {
    echo "로컬 ${HEAD_BRANCH} 삭제 실패 — squash merge 특성상 발생 가능. 수동 확인 필요." >&2
  }
fi
```

- `--delete-branch`는 **원격만** 삭제. 로컬은 별도.
- 삭제 전 반드시 `main` 체크아웃. 현재 브랜치는 삭제 못 함.
- `git branch -d` 실패 시 `-D`로 강제 삭제 **금지** — 미머지 커밋 유실 위험.

---

## Step 6 — 완료 처리

### 6-1. 연결된 Issue 라벨 전환 (`in-review` → `done`)

PR body에 `Closes #{M}` / `Fixes #{M}` 패턴이 있으면 해당 이슈 번호 추출:

```bash
# PR body에서 "Closes #숫자" 패턴 추출
ISSUE_NUMS=$(gh pr view {N} --repo timonPark/flowmeet-web --json body --jq '.body' \
  | grep -oiE '(close[sd]?|fix(e[sd])?|resolve[sd]?) #[0-9]+' \
  | grep -oE '#[0-9]+' | tr -d '#' | sort -u)

for I in $ISSUE_NUMS; do
  # 머지되면 이슈는 GitHub가 자동 close하지만, 라벨은 수동 전환
  gh issue edit "$I" --repo timonPark/flowmeet-web --remove-label "status:in-review"
  gh issue edit "$I" --repo timonPark/flowmeet-web --add-label "status:done"
  # 검증 — 집합 비교 (GitHub API 라벨 반환 순서 비의존)
  gh issue view "$I" --repo timonPark/flowmeet-web --json labels \
    --jq '[.labels[].name] | map(select(startswith("status:"))) | sort' \
    | jq -e '. == ["status:done"]' > /dev/null || {
    echo "이슈 #${I} 라벨 전환 검증 실패 (현재 status 라벨이 ['status:done']과 불일치)" >&2
  }
done
```

### 6-2. 에픽(Epic) 이슈 체크박스 업데이트

연결 이슈가 에픽의 하위 이슈라면 에픽 body의 체크박스를 `- [x]`로 바꾼다. 수동 또는 별도 스킬 범위.

### 6-3. 완료 보고

- **work_history 미도입** 이므로 별도 파일 기록 없음.
- 완료 사실은 **머지 커밋 메시지 + GitHub status:done 라벨 + Closes 로 인한 이슈 자동 close** 로 완결.
- 투명성 포맷으로 사용자에게 보고.

### 투명성 포맷

```
▶ [Step 6] 완료 처리 — 라벨 전환·로컬 정리
✅ [Step 6]
  · 머지 커밋: e79f83e
  · 이슈 #10 라벨: in-review → done
  · 로컬 docs/10-work-manager-port 삭제 완료
```

---

## 블로킹·중단 처리

작업이 중단되거나 리뷰에서 blocker가 나오면:

- GitHub 라벨은 **`status:in-review` 유지** (down-grade 금지).
- 투명성 포맷으로 `🚧 블로킹 — {사유}` 보고.
- 작업 재개 시 blocker 해소 후 Step 4 재검증 → Step 5 승인 대기.

---

## 보안 가드 — PR body·커밋 메시지·codex 산출물

- `gh pr view` body에 PII·토큰·음성 원문이 섞여 있을 수 있다. **직접 복사 금지**.
- codex 산출물은 `.gitignore` 로 로컬 전용이지만, triage 결과를 커밋 메시지·PR body 에 반영할 때는 식별자 중심 요약만.
- 투명성 포맷(`▶`/`✅`/`🚧`)에 title·body·코멘트 발췌가 포함될 때 — 출력 전 [`maskSensitive()`](../web-security/SKILL.md#3-pii-마스킹-상세-8-완결) 통과.
- 식별자(number·sha·refName·state·labels[].name)는 mask 생략.
- 금지 필드 전체는 [`_REFERENCE.md` 금지 필드·금지 출력](./_REFERENCE.md#금지-필드금지-출력-web-security-3) 참조.

---

## 금지 사항

- ❌ **사용자 명시 승인 없이** `gh pr merge` 실행 — 자동 머지는 어떤 조건에서도 금지.
- ❌ GitHub `Auto-merge` / "Merge when ready" 활용.
- ❌ Squash 외 머지 방식 (`--merge`·`--rebase` 금지, operations.md §2.1).
- ❌ "앞서 한 번 머지 승인받았으니 다음 PR도 자동" — 승인은 PR별로 1회성.
- ❌ 사람 리뷰 생략 — codex 리뷰는 보조 레인이지 대체재 아님.
- ❌ codex-review-triage skip (operations.md §2.2 필수).
- ❌ `git branch -D`로 로컬 브랜치 강제 삭제 (미머지 커밋 유실 위험).
- ❌ 미답변 인라인 코멘트가 남은 상태에서 머지.
- ❌ `gh pr view` body를 커밋 메시지·PR 보강 body에 그대로 복사 (PII·토큰·회의 내용 유출).
- ❌ 라벨 두 개가 공존하는 상태 (반드시 remove → add 세트 + 검증).
- ❌ `.claude/work_history/` 파일 생성 (operations.md §2.3 미도입).

---

## 전체 투명성 포맷 예시

```
▶ [Step 1] PR #24 메타 조회
✅ [Step 1] title=feat(harness): ..., state=OPEN, head=docs/10-work-manager-port, base=main, mergeState=CLEAN
▶ [Step 2] 유형 판정
✅ [Step 2] type=DOC (title "docs(skill):" 접두사 매칭)
▶ [Step 3] codex-review-triage
✅ [Step 3] codereview/20261017/docs-10-work-manager-port.md (유효 2 / 부분유효 1 / 반려 3)
▶ [Step 4] 리뷰 상태 점검
✅ [Step 4] 미답변 코멘트 0 / 승인 1 / codex 산출물 OK / mergeState CLEAN
🚧 [Step 5] 머지 승인 대기 — 사용자가 "머지해줘" 지시 전까지 대기
```

사용자 승인 후:

```
▶ [Step 5] gh pr merge --squash --delete-branch 실행
✅ [Step 5] 머지 완료 (merge commit: e79f83e)
▶ [Step 6] 완료 처리
✅ [Step 6] 이슈 #10 라벨 전환·로컬 브랜치 정리
```
