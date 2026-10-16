---
title: 운영 규정 (PR·브랜치·codex·work_history·CI)
date: 2026-10-16
author: kindcorder@gmail.com
status: DECIDED
issue: "#9"
scope: PR 리뷰/머지 규약·codex 외부 리뷰 도입·work_history 조직·CI 운영 원칙
depends_on:
  - "#6 테스트 전략"
---

# 운영 규정

Phase 1 (축 합의) 의 마지막 축. 피처 구현 (Phase 2+) 으로 넘어가기 전에 **"어떻게 작업을 돌리고 머지할지"** 의 운영 규약을 확정한다. 지금까지 CLAUDE.md 상단에 "전용 브랜치 필수·main 직접 커밋 금지" 만 있었던 것을 **PR 머지 가드레일·외부 리뷰·작업 로그·CI 운영 원칙** 까지 확장한다.

> 이 문서는 `work-manager` 스킬([#10](https://github.com/timonPark/web/issues/10)) 과 공통 에이전트 복제 ([#11](https://github.com/timonPark/flowmeet-web/issues/11)) 작업의 입력. api 리포 패턴을 그대로 복제·포팅하는 범위를 여기서 못 박음.

---

## 1. 결정 요약

| 영역 | 결정 |
|---|---|
| **PR 자동 머지** | **금지** (사람이 눌러 머지, api 조항 그대로) |
| **머지 방식** | **squash** 고정 |
| **브랜치 보호** | `main` 직접 push 금지, PR 필수 (GitHub 설정) |
| **PR 템플릿** | `.github/pull_request_template.md` 추가 (이 PR 포함) |
| **PR 리뷰 체크리스트** | 축별 룰 모음 — **`CLAUDE.md` 섹션** 으로 유지, 축 완성될수록 자람 |
| **codex 외부 리뷰** | **도입** — api 리포 `codex-review-triage` 스킬 그대로 포팅 |
| **codex 트리거** | **모든 PR 의 머지 전 흐름의 일부** (`work-manager _pr-flow` 가 호출) |
| **codex 산출물** | `codereview/<YYYYMMDD>/<slug>.md`, **`.gitignore` 로컬 전용** |
| **work_history 폴더** | **미도입** — `docs/axis-decisions/` + PR 본문 + 커밋 메시지로 커버 |
| **장기 피처 노트** | 선택적 `docs/features/<name>/notes.md` (피처 PR 시작 때 결정) |
| **CI** | **GitHub Actions**, PR 당 lint + typecheck + test + 조건부 e2e. 커버리지 하드 게이트 없음 (#6 유지) |
| **실제 CI YAML / 브랜치 보호 설정** | **이 PR 범위 밖**, 별도 chore PR + GitHub 설정 |

**원칙**: *사람이 머지한다. 외부 모델 2차 눈으로 저자-자신 리뷰의 blind spot 을 보완한다. 중복 로그 소스 없이 결정은 `docs/axis-decisions/`, 변경 이유는 커밋/PR, 구조·규약은 `CLAUDE.md` 한 곳에만.*

---

## 2. 결정 상세

### 2.1. PR·브랜치 규약

#### 자동 머지 금지

- api 리포 2026-10-14 사고 이후 명문화된 조항을 그대로 가져옴.
- 이유: "CI 그린 자동 머지" 는 **메타 검증 간과를 유발** — 테스트는 통과하지만 결정 축과 충돌하는 변경 등. 사람 승인이 **최종 gate**.
- GitHub `Auto-merge` 기능 사용 금지. "Merge when ready" / "Auto-merge" 체크 금지.

#### 머지 방식 — squash

- 지금까지의 패턴 유지. feature 브랜치의 중간 커밋을 main 에 남기지 않음.
- GitHub PR 머지 시 **Squash and merge** 만 사용. "Create a merge commit" / "Rebase and merge" 금지.
- 리포 설정에서 다른 방식을 **비활성화** (GitHub 리포 Settings).

#### 브랜치 명명 규약 (CLAUDE.md 유지)

`<type>/<issue-number>-<slug>` 또는 이슈 없는 작업은 `<type>/<slug>`.
- type: `feature` / `fix` / `refactor` / `chore` / `docs` / `test`

#### 브랜치 보호 룰 (GitHub 설정)

- `main` **직접 push 금지**
- PR 필수
- **"CI 통과 필수"** 는 CI workflow 생긴 뒤 활성화 (지금 비활성)
- Draft PR 머지 차단
- stale review 자동 dismiss 없음 (리뷰어 1명 체제라 무의미)

→ **실제 GitHub 설정은 이 PR 머지 후 사용자가 수동 적용**. 결정 문서에만 명시.

#### PR 템플릿

`.github/pull_request_template.md` 를 이 PR 에 **동봉**. 포함 항목:
- 요약
- 변경 요지
- 테스트 (lint/typecheck/unit/수동 확인 체크박스)
- 의존 이슈·축 결정 영향
- codex 리뷰 상태 (수행 여부·triage 결과 요약 링크)
- Closes #N

### 2.2. codex 외부 리뷰 — **도입**

#### 포팅 대상

api 리포의 `../flowmeet-api/.claude/skills/codex-review-triage/SKILL.md` 를 **web 리포로 그대로 복제**. 교체 포인트:
- 스택 참조: Kotlin + Spring Boot → **TS + React 19 + Next 16**
- 서버 검증 레벨:
  - **L0** — 하네스/문서만 (`.claude/`·`CLAUDE.md`·`docs/`): 검증 생략 가능, 커밋 메시지에 `Verified: L0 skipped — doc-only change`
  - **L1** — `src/` 하위 코드 변경: `npm run lint` + `tsc --noEmit` + `npm test` (Vitest 유닛·컴포넌트)
  - **L2** — 라우팅·보안·API 클라이언트·토큰 흐름 변경: L1 + `npm run test:e2e` (Playwright 스모크)
- 상위 규칙 참조: api 의 `kotlin-style`·`spring-layering` → web 의 `ts-style`(#14)·`app-router-layering`(#15)·`data-fetching-guard`(#17)·`error-ux`(#16) 등
- 산출물 경로·`-s read-only` 샌드박스·억제 지시 (스킬 로드 금지·파일 쓰기 금지·Jira 참조 금지)·triage 축 (유효/부분유효/반려) 는 **동일**

#### 트리거

- **모든 PR 의 머지 전 흐름에 포함**. 선택적 아님.
- 호출 지점: 포팅될 `work-manager` 스킬 (#10) 의 `_pr-flow` 가 **PR 생성 직후 자동 로드**.
- 실행 순서 (api 패턴 그대로):
  1. codex exec 로 리뷰 생성 (`-s read-only`)
  2. 산출물 `codereview/<YYYYMMDD>/<slug>.md` 저장
  3. Claude triage — 각 지적을 원문 대조·유효/부분유효/반려 분류
  4. 유효 항목만 반영
  5. 서버 검증 (L0/L1/L2 중 선택, 커밋 메시지에 `Verified:` 라인)
  6. (선택) 재리뷰 — regression 체크

#### 산출물 저장

- 경로: `codereview/<YYYYMMDD>/<slug>.md` (리포 루트)
- `.gitignore` 에 `codereview/` 추가 — **원격 push 안 됨, 로컬·개인 아카이브**
- frontmatter 필수 필드: `pr`, `branch`, `base`, `reviewer`, `generated_at`

#### 금지 사항 (api 와 동일)

- codex 결과를 **triage 없이** 그대로 수용·커밋 금지
- `-s workspace-write` 이상 샌드박스 금지
- 거대 PR 리뷰 금지 — 수백 줄 넘기면 PR 쪼갬
- Jira 키·외부 티켓 참조 금지
- 사람 리뷰 대체 금지 — 보조 레인

#### 전제조건

- 로컬 `codex` CLI 설치 필요. 사용자 환경에 설치 안 돼 있으면 `work-manager _pr-flow` 가 **codex 단계를 skip 하지 않고 사용자에게 설치 안내**.

### 2.3. work_history

#### 결정

- **`.claude/work_history/` 전용 폴더 미도입**.
- 이유: `docs/axis-decisions/*.md` 가 **결정의 근거·대안·열어둔 포인트** 를 이미 담고 있고, **PR 본문·커밋 메시지** 가 "뭘 바꿨는지" 를 담는다. 중복 소스를 두면 drift 가 불가피.

#### 피처 구현 단계의 작업 노트

- 기본: **PR 본문 + 커밋 메시지로 충분**.
- 장기·복잡 피처 (녹음 세션·화자 매칭·캘린더 등) 에서 **PR 이 여러 번 쪼개져 이어질 때만** 선택적으로 `docs/features/<feature>/notes.md` 생성.
  - 피처 PR 시작 때 결정, 모든 피처에 자동 적용 아님
  - 포맷: `# <피처명>` + `## 배경` + `## 결정 이력` (ADR 미니) + `## 미결·열린 질문`

#### 중복 로그 소스 금지 원칙

- 결정 → `docs/axis-decisions/`
- 변경 이유 → 커밋/PR
- 구조·규약 → `CLAUDE.md`
- 어디에도 안 맞는 로그가 필요할 때 **그때 새 장소를 만든다** — 미리 폴더 파놓지 않음.

### 2.4. CI 운영 (#6 에서 이관)

#### 프로바이더

- **GitHub Actions**. 배포 플랫폼 (Vercel/AWS 등) 비의존.

#### 돌릴 Job (PR 트리거)

| Job | 명령 | 조건 |
|---|---|---|
| `lint` | `npm run lint` | 항상 |
| `typecheck` | `npm run tsc --noEmit` (스크립트 추가 필요) | 항상 |
| `test` | `npm test` (Vitest 유닛 + 컴포넌트) | 항상 |
| `e2e` | `npm run test:e2e` (Playwright 스모크) | `src/`·`e2e/`·`next.config.ts`·`package.json` 변경 PR 에만 |

- **실패 시**: 머지 블로킹 (브랜치 보호 "CI 통과 필수" 와 결합)
- **커버리지 하드 게이트**: **없음** (#6 결정 유지)
- **캐시**: `node_modules` + Playwright browsers
- **E2E Playwright `trace: 'retain-on-failure'`** — 실패 시 trace 보존

#### 실제 YAML 작성 — **별도 chore PR**

- `.github/workflows/ci.yml` 작성·검증은 **이 PR 범위 밖**.
- 이유: CI 설정은 반복 iteration 이 필요 — 결정 문서 PR 과 분리해야 리뷰·롤백이 깔끔.
- 별도 chore PR 번호는 CI YAML 작성 시 발행.

### 2.5. PR 리뷰 체크리스트 — **CLAUDE.md 섹션으로 유지**

#### 결정

- **별도 `docs/pr-review-checklist.md` 신설 안 함**.
- 축이 완성될수록 **CLAUDE.md 의 각 섹션 (API 통신·상태 관리·폼·에러·테스트·a11y·로깅·운영) 이 그대로 리뷰 체크리스트** 역할.
- PR 리뷰 시 CLAUDE.md 를 scan → 각 섹션 룰과 변경이 충돌 없는지 확인.

#### 축 완성 후 재검토

- Phase 2·3 완료 시 PR 리뷰 체크리스트를 **별도 문서로 분리할지** 재검토. 지금은 CLAUDE.md 섹션 중복을 피함.

---

## 3. 다른 축에 미치는 영향

| 이슈 | 내용 |
|---|---|
| [#6](https://github.com/timonPark/flowmeet-web/issues/6) 테스트 | CI 트리거 조건·L1/L2 서버 검증 명령이 테스트 전략의 상위 운영. 변경 없음. |
| [#10](https://github.com/timonPark/flowmeet-web/issues/10) `work-manager` 포팅 | api `work-manager` 포팅 시 **codex-review-triage 호출 지점 동일 유지**. L0/L1/L2 명령만 web 용으로 교체. |
| [#11](https://github.com/timonPark/flowmeet-web/issues/11) 공통 에이전트 복제 | 포팅 범위에 `codex-review-triage` 스킬 **포함**. `pr-review-check` 에이전트도 함께. |
| [#12](https://github.com/timonPark/flowmeet-web/issues/12) CLAUDE.md 운영 규정 추가 | 이 결정 문서의 §2.1·§2.5 를 CLAUDE.md 로 반영 (이 PR 에 포함) |

---

## 4. 열어둔 포인트 (defer)

| 항목 | 사유 | 어디로 |
|---|---|---|
| `.github/workflows/ci.yml` 실제 작성 | 반복 iteration 필요, 결정 PR 과 분리 | 별도 chore PR (예정) |
| GitHub 리포 설정 (브랜치 보호, auto-merge 비활성, squash-only) | GitHub Web UI 설정, 코드 아님 | PR 머지 후 사용자 수동 적용 |
| `codex` CLI 설치 가이드 | 사용자 환경별 다름 | work-manager 포팅 PR 에 안내 섹션 |
| PR 리뷰 체크리스트 별도 문서화 | 축이 완성될 때 재검토 | Phase 3 완료 후 |
| 장기 피처 `docs/features/<name>/notes.md` 포맷 세부 | 첫 피처 PR 시 결정 | 녹음 세션·화자 매칭 구현 시 |
| 릴리스 전략·태깅 | 첫 프로덕션 배포 전 | 배포 플랫폼 결정 시 |

---

## 5. 완료 조건 체크리스트

- [x] PR 자동 머지 금지·squash·브랜치 보호 결정
- [x] PR 템플릿 포맷 정의 (실제 파일 추가는 이 PR 에)
- [x] codex 외부 리뷰 **도입** 결정 + 포팅 대상 명시
- [x] codex 트리거·산출물·금지사항·전제조건
- [x] work_history 폴더 **미도입** 결정 + 피처 노트 선택적 룰
- [x] CI 운영 원칙 (뭘 돌릴지·언제·실패시) — 실제 YAML 은 defer
- [x] PR 리뷰 체크리스트 = CLAUDE.md 섹션 재사용 결정
- [x] 다른 축 영향 매핑
- [x] 열어둔 포인트 이관처
- [ ] CLAUDE.md 축약 반영 + 이 문서 링크 (이 PR)
- [ ] `.github/pull_request_template.md` 추가 (이 PR)
- [ ] `.gitignore` 에 `codereview/` 추가 (이 PR)

## 6. 참고

- [`docs/axis-decisions/test-strategy.md`](./test-strategy.md) — CI 돌릴 테스트 종류 결정 원천
- [`docs/harness-bootstrap-survey.md`](../harness-bootstrap-survey.md) §4 재사용 가능 — codex 흐름 복제 전제
- api 리포 `../flowmeet-api/.claude/skills/codex-review-triage/SKILL.md` — 포팅 대상 소스
- api 리포 `../flowmeet-api/.claude/agents/pr-review-check.md` — 함께 포팅
- api 리포 CLAUDE.md "자동 머지 금지" 조항 — 그대로 가져옴
