# work-manager 참조 문서

실행 로직과 무관한 참조용 정의. Claude 실행 시 자동 로드되지 않는다 (언더스코어 접두 파일).

> **work_history 미도입**: api 리포 `_REFERENCE.md` 가 담고 있던 frontmatter 필드 정의·메트릭 추출·`wh-metrics.py` 섹션은 **web 리포에선 적용하지 않는다** ([operations.md §2.3](../../../docs/axis-decisions/operations.md)). 결정은 `docs/axis-decisions/`, 변경 이유는 커밋/PR, 구조·규약은 CLAUDE.md 로 분산.

---

## type ↔ 브랜치 type 매핑

CLAUDE.md "작업 시작 규칙"의 브랜치 명명과 유형 판정을 1:1로 맞춘다.

| `type` (판정) | 브랜치 `type` | 비고 |
|---|---|---|
| `FEAT` | `feature` | 신규 기능 |
| `FIX` | `fix` | 버그 수정 |
| `PERF` | `refactor` | 성능 개선 — 외형 변경 작으면 refactor로 분류 |
| `REFACTOR` | `refactor` | 구조 개선 |
| `DOC` | `docs` | 문서·하네스 |
| `CHORE` | `chore` | 설정·의존성 |
| `TEST` | `test` | 테스트 추가·보강 |
| `DEPLOY-REVIEW` | — (브랜치 없음) | PR URL 입력 — 브랜치 불필요 |
| `ANALYSIS` | `chore` | 조사·분석 결과물. 코드 변경 작으면 chore |

---

## type ↔ 전문 에이전트 라우팅

work-manager는 유형 판정 결과를 받아 아래 에이전트를 호출한다. 에이전트 정의는 `.claude/agents/*.md` (포팅 예정 — #11 범위). 호출 금지(1-depth): Claude Code가 Read 후 직접 실행.

### 흐름 전체

```
issue-kickoff (공통) → [brainstormer] → [plan-writer] → 전문 에이전트 → PR → codex-review-triage → merge
                      └── 복잡도별 skip ──┘
```

### type별 매핑

| `type` | brainstormer | plan-writer | 전문 에이전트 | 비고 |
|---|---|---|---|---|
| `FEAT` (복잡: 영향 파일 3개↑ 또는 보안/외부 연동/DB 스키마) | ✅ | ✅ | [feat-developer](../../agents/feat-developer.md) | TDD 사이클 |
| `FEAT` (작음: 단일 파일·레이어) | ⬜ | ✅ (간소) | [feat-developer](../../agents/feat-developer.md) | |
| `FIX` (재현 조건 명확 + 수정 범위 좁음) | ⬜ | ✅ (간소) | [fix-developer](../../agents/fix-developer.md) | 재현 테스트 먼저 |
| `FIX` (영향 범위 넓음) | ✅ | ✅ | [fix-developer](../../agents/fix-developer.md) | |
| `REFACTOR` · `PERF` | ✅ | ✅ | [refactor-developer](../../agents/refactor-developer.md) | 외부 계약 유지 |
| `TEST` | ⬜ | ✅ (간소) | [feat-developer](../../agents/feat-developer.md) | 테스트도 TDD |
| `DOC` · `CHORE`(하네스) | ⬜ | ⬜ | [doc-updater](../../agents/doc-updater.md) | L0 |
| `ANALYSIS` (문서 산출물) | ⬜ | ✅ (간소) | [doc-updater](../../agents/doc-updater.md) | |
| `ANALYSIS` (코드 변경 포함) | ✅ | ✅ | feat-developer 또는 refactor-developer | 샘플 코드 추가 등 |
| `DEPLOY-REVIEW` | — | — | [pr-review-check](../../agents/pr-review-check.md) | PR URL 입력 전용 |

> `brainstormer`·`plan-writer` 상세는 [`brainstormer.md`](../../agents/brainstormer.md) · [`plan-writer.md`](../../agents/plan-writer.md). skip 결정 기준은 `_issue-flow.md` Step 3.3/3.5 참조.

> issue 수신 직후의 **브랜치 분기**는 공통으로 [`issue-kickoff`](../../agents/issue-kickoff.md)가 수행. 그 뒤 brainstormer → plan-writer → 전문 에이전트 순으로 핸드오프.

### 금지 필드·금지 출력 (web-security §3)

투명성 포맷·PR body·커밋 메시지 **어디에도 쓰지 않는다**:

- 비밀번호·토큰·API 키·세션 쿠키
- 음성 원문·transcript 본문
- 회의 참가자 이메일·전화번호 원본 (식별자만 — `meeting_id`, `user_id`)

> free-form 메모가 필요하면 식별자 중심으로 쓴다. 이메일/토큰이 섞여 들어가면 출력 직전 [`maskSensitive()`](../web-security/SKILL.md#3-pii-마스킹-상세-8-완결) 를 반드시 통과시킨다. `gh issue view`·`gh pr view` 결과를 그대로 저장·출력하지 말 것 — body 발췌가 필요하면 mask 후.

---

## 명명 규약

### 브랜치명 (CLAUDE.md "작업 시작 규칙" 재인용)

```
<type>/<issue-number>-<slug>
```

- `type`: `feature` / `fix` / `refactor` / `chore` / `docs` / `test`
- 이슈 번호가 없는 작업은 `<type>/<slug>` (예: `chore/update-tailwind`)

### 커밋 메시지 (관례)

Conventional Commits 변형:

```
<type>(<scope>): <summary> (#<issue>)

<body>

Verified: L<N> — <command> (passed, <n> tests)
Refs: #<issue>
```

L0/L1/L2 레벨은 [codex-review-triage §5](../codex-review-triage/SKILL.md) 참조. web 기준:

- **L0** — `.claude/`·`CLAUDE.md`·`docs/` 등 문서/하네스만 변경: 검증 생략 가능, 커밋 메시지에 `Verified: L0 skipped — doc-only change`
- **L1** — `src/` 하위 코드 변경: `npm run lint` + `tsc --noEmit` + `npm test`
- **L2** — 라우팅·보안·API 클라이언트·토큰 흐름 변경: L1 + `npm run test:e2e` (Playwright 스모크)

> 출처: [operations.md §2.2](../../../docs/axis-decisions/operations.md).

### PR 제목·body

- 제목: `<type>(<scope>): <summary> (#<issue>)` — Conventional Commits 와 동일.
- body: `.github/pull_request_template.md` 템플릿 사용 — 요약·변경·테스트·codex 리뷰 상태·의존 이슈·`Closes #N` 포함.
- `Closes #{N}` / `Fixes #{N}` 패턴으로 이슈 자동 close 트리거.

### codex 산출물

```
codereview/<YYYYMMDD>/<branch-slug>.md
```

- `branch-slug`: 브랜치명의 `/` 를 `-` 로 치환 (예: `docs/10-work-manager-port` → `docs-10-work-manager-port`).
- `.gitignore` 로 로컬 전용 — 원격 push 안 됨.
