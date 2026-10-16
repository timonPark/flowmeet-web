# CLAUDE.md

조직 지시 및 업무 흐름 관리 플랫폼 웹 프론트엔드 리포 (`flowmeet-web`). API 서버는 별도 리포 (`../flowmeet-api`).

## 작업 시작 규칙 (필수)

**어떤 작업이든 반드시 전용 브랜치를 만들어 진행한다. `main`에 직접 커밋 금지.**

- 작업 착수 시 가장 먼저 `main`에서 새 브랜치를 분기한다.
- 브랜치 명명 규약: `<type>/<issue-number>-<slug>`
  - `type`: `feature` / `fix` / `refactor` / `chore` / `docs` / `test`
  - 예: `feature/5-meeting-list`, `fix/12-recording-button`, `docs/3-readme`
- 이슈 번호가 없는 작업은 `<type>/<slug>` (예: `chore/update-tailwind`).
- 작업 완료 후 PR을 생성해 `main`으로 머지 — 직접 푸시 금지.

```bash
# 올바른 착수 흐름
git checkout main && git pull
git checkout -b feature/<issue>-<slug>
# ... 작업 ...
git push -u origin feature/<issue>-<slug>
gh pr create --fill
```

## PR·머지 가드레일

> 상세: [`docs/axis-decisions/operations.md`](docs/axis-decisions/operations.md) (이슈 #9)

- **PR 자동 머지 금지** — GitHub `Auto-merge` / "Merge when ready" 체크 금지. **사람이 눌러야 머지**. (api 2026-10-14 사고 조항 그대로)
- **머지 방식: squash 고정** — "Create a merge commit" / "Rebase and merge" 금지.
- **`main` 브랜치 보호** — 직접 push 금지, PR 필수. CI 통과 필수 조건은 CI workflow 생긴 뒤 활성화.
- **PR 템플릿**: `.github/pull_request_template.md` — 요약·변경·테스트·codex 리뷰 상태·의존 이슈 포함.
- **codex 외부 리뷰 필수**: 모든 PR 머지 전에 `codex-review-triage` 흐름 (6단계) 완료. 산출물 `codereview/<YYYYMMDD>/<slug>.md` (로컬 전용, `.gitignore`). 유효 지적만 반영, Claude triage 생략 금지. **사람 리뷰 대체 아님**.
- **CI (예정)**: GitHub Actions. PR 당 `lint` + `tsc --noEmit` + `npm test` + 조건부 Playwright E2E. 실패 시 머지 블로킹. 커버리지 하드 게이트 없음.
- **PR 리뷰 체크리스트**: 별도 문서 없음 — **이 CLAUDE.md 의 각 섹션 (API 통신·상태 관리·폼·에러·테스트·a11y·로깅 등) 이 그대로 체크리스트**. PR 리뷰 시 scan.
- **작업 로그**: `.claude/work_history/` 미도입. 결정은 `docs/axis-decisions/`, 변경 이유는 커밋/PR, 구조는 CLAUDE.md. 중복 소스 금지.
- **투명성 원칙**: 작업 중 상태 보고에 `▶` (진행 중) / `✅` (완료) / `🚧` (블로커·대기) 를 **의도적으로 사용**. 사용자가 흐름을 끊지 않고도 어디까지 왔는지 알 수 있도록.

## 하네스 스킬 (`.claude/skills/`)

- Next.js 코드·설정·캐시·라우팅·Server Actions·RSC·Tailwind 테마를 쓰거나 리뷰하기 전에 **[`nextjs16-reality-check`](.claude/skills/nextjs16-reality-check/SKILL.md)** 로드 필수. Opus/Sonnet 학습 지식이 Next 15 전제라 틀어지는 지점을 catch.
- 다른 스킬은 Phase 3 에서 순차 추가 — `ts-style` (#14) / `app-router-layering` (#15) / `error-ux` (#16) / `data-fetching-guard` (#17) / `web-testing` (#18) / `web-security` (#19) / 신규 축 묶음 (#20).
- codex 외부 리뷰는 Phase 2 포팅 대상 — api 리포의 `codex-review-triage` 를 그대로 복제 (#9, #11).

## 기술 스택

- Next.js 16 (App Router) + React 19 + TypeScript 5
- Tailwind CSS 4 (`@tailwindcss/turbopack`)
- ESLint 9 (`eslint-config-next`)
- Node.js 20.9+ (권장: 22 LTS, nvm 사용)

## 디렉터리 구조

```text
flowmeet-web/
├─ src/app/              # App Router 페이지 및 레이아웃
│  ├─ layout.tsx
│  ├─ page.tsx
│  └─ globals.css
├─ public/               # 정적 자산
├─ eslint.config.mjs
├─ next.config.ts
├─ tsconfig.json         # @/* → src/* alias
└─ package.json
```

## 자주 쓰는 명령

```bash
# 최초 1회 (nvm 사용 시)
nvm use 22
npm install
cp .env.local.example .env.local

# 개발 서버 (http://localhost:3000)
npm run dev

# 프로덕션 빌드
npm run build
npm start

# 린트
npm run lint
```

## 환경 변수

| 변수 | 설명 |
| --- | --- |
| `NEXT_PUBLIC_API_BASE_URL` | `flowmeet-api` 베이스 URL (기본: `http://localhost:8080`) |

`NEXT_PUBLIC_` 접두사가 없는 값은 서버 전용. 비밀값은 반드시 서버 컴포넌트/Route Handler에서만 사용.

## 설계 원칙 (TRD 발췌)

- **인증**: 이메일·비밀번호 로그인, JWT Access Token + Refresh Token. 최초 로그인 시 임시 비밀번호 변경. (TRD §6)
- **권한 검사는 백엔드가 수행**. 프론트는 UI 가드 용도로만 역할을 체크하고, 보안 경계는 서버에 위임.
- **녹음**: 브라우저 MediaRecorder API로 WebM/Opus 또는 브라우저 원본 형식 생성 → IndexedDB에 임시 저장 → 회의 종료 후 서버 업로드. (TRD §4.1)
- **화자 매칭 UI**: 화자 라벨별 매칭 상태와 전체 STT utterance 목록을 함께 받아 즉시 로컬 반영. (TRD §4.3)
- **회의실 캘린더**: 웹은 일·주간 캘린더, 모바일은 날짜별 예약 목록. (TRD §4.2)
- **대시보드**: `dashboardType=MEETING | TEAM` 분기. 팀 대시보드는 지연 업무(업무명·담당자·기한·지연일수)를 표시. (TRD §8)

## API 통신 / 데이터 페칭

> 상세: [`docs/axis-decisions/data-fetching.md`](docs/axis-decisions/data-fetching.md) (이슈 #2)

- `flowmeet-api` 베이스 URL: `process.env.NEXT_PUBLIC_API_BASE_URL`
- 인증: `Authorization: Bearer <accessToken>` — fetch wrapper 가 자동 주입
- Refresh Token: HttpOnly 쿠키 (세부 정책은 #19 `web-security` 에서 확정)
- fetch wrapper 는 **서버/클라 분리**: `src/lib/api/serverClient.ts`, `src/lib/api/browserClient.ts`, 공통 `_core.ts`. 한 함수에 환경 분기 금지.
- 서버 데이터는 **기본 dynamic**, 캐시는 `'use cache'` + `cacheLife` + `cacheTag` 로 명시 opt-in (Next 16 `cacheComponents` 모델). "`fetch` 는 알아서 캐시" 전제는 **틀림**.
- **`'use cache'` 함수 안에서 `cookies()`/`headers()` 접근 금지** — 사용자 간 캐시 오염 위험. 사용자별 데이터는 `userId` 를 명시 인수로 받고 `cacheTag('<domain>:user:${id}')` 로 격리하거나, 캐시 함수를 공용 데이터만 다루고 개인화는 캐시 밖에서 수행.
- 클라이언트 캐시: **React Query v5**. 낙관 업데이트가 필요한 흐름(화자 매칭·녹음 상태·토글) 은 RQ mutation, 폼 제출은 Server Actions + `revalidateTag` 혼용.
- Server Action 내부에서도 **반드시 `serverClient`** 사용 (직접 `fetch` 금지).

## 상태 관리

> 상세: [`docs/axis-decisions/state-management.md`](docs/axis-decisions/state-management.md) (이슈 #3)

- **서버 데이터**: React Query (#2). "RQ 가 가지고 있으면 Zustand 에 넣지 않는다".
- **인증 세션** (user, role, 메모리 accessToken): React Context `AuthProvider` — `src/app/providers.tsx`.
- **UI 상태 / 피처 로컬 상태**: Zustand slice — `src/lib/stores/{uiStore,recordingStore,speakerMatchStore,...}.ts`. Provider 없음, 셀렉터 패턴 필수.
- **폼 상태**: 폼 라이브러리가 담당 (#4). 폼 중간값을 Zustand/Context 로 복제 금지.
- Store 안에서 직접 `fetch` 호출 금지. 서버 통신은 RQ `queryFn`·Server Action 경유.
- 녹음 chunk 등 바이너리 영속화는 Zustand `persist` 가 아닌 전용 IndexedDB 레이어로 분리. Zustand 는 메타데이터만.

## 폼 / 검증

> 상세: [`docs/axis-decisions/form-validation.md`](docs/axis-decisions/form-validation.md) (이슈 #4)

- 폼 라이브러리: **react-hook-form v7** + **@hookform/resolvers/zod**.
- 스키마: **zod**. `src/lib/validation/schemas/<domain>.ts` 에 한 번만 선언 → RHF resolver (클라) 와 Server Action 입력 파싱 (서버) **양쪽 공용**.
- Server Action 은 throw 하지 않고 `FormActionResult<T>` (`ok` discriminated union) 를 반환. `fieldErrors` 는 RHF `setError`, `formError` 는 폼 상단/토스트.
- 서버가 **진실의 근원**. 클라 선검증은 UX 목적만, 서버 결과로 덮어쓴다. Server Action 상단에서 `schema.safeParse` 재검증 필수.
- 비-폼 상호작용 (토글·화자매칭 라벨 등) 은 RHF 쓰지 않음 → RQ mutation 직결 (#2).
- 서버→폼 에러 매핑 함수는 `src/lib/validation/errors.ts` 한 곳에만. 폼 쪽에서 재구현 금지.

## 에러 처리 (error-ux)

> 상세: [`docs/axis-decisions/error-ux.md`](docs/axis-decisions/error-ux.md) (이슈 #5)

- 경계: **하이브리드** — 루트 `global-error.tsx` + 섹션 그룹 `error.tsx` + 선별적 `<ErrorBoundary>` (`react-error-boundary` v4). "폴백이 더 유용한 지점" 에만 둔다.
- `ApiError` / `ApiErrorCode` / `codeToMessage` 는 **`src/lib/errors/`** 에 집중. api 리포 `ErrorCode` 와 1:1 미러, union 밖은 `UNKNOWN`.
- **중앙 메시지 테이블 1건** (`src/lib/errors/messages.ts`). 폼 쪽은 import 재사용, 중복 금지.
- RQ 기본: `queries.throwOnError: true` (섹션 ErrorBoundary 로 bubbling), `mutations.onError: 전역 토스트`. 호출부가 override 가능.
- 비-폼 Server Action 은 `ActionResult<T>` (ok discriminated union). 폼 `FormActionResult` 와 shape 통일.
- Server Action throw 정책 — **오류 성격에 따라 다름**: 예상 가능한 도메인/검증 오류 → `ActionResult` 반환. 세션 만료 → `redirect('/login')`. **예상 못한 예외 → `reportError` 로깅 후 throw** (에러 바운더리로 bubble). 모든 예외를 조용히 변환하면 관측 누락. 상세는 error-ux.md §F.
- 토스트는 `uiStore.toast.{error,success,...}` 추상 API. 중복 deduping은 `uiStore` 가 보장.
- 로깅 훅: **`reportError(err, ctx)`** 추상. **외부 SaaS 벤더 미도입** (#8). 구현은 `src/lib/errors/reportError.ts` 가 **구조화 JSON `console.error`**. 추후 자체 백엔드 수집 엔드포인트 생기면 함수 내부에서 조건부 fetch 추가 — 호출부 변경 없음.

## 테스트

> 상세: [`docs/axis-decisions/test-strategy.md`](docs/axis-decisions/test-strategy.md) (이슈 #6)

- 유닛/통합: **Vitest** + `@vitejs/plugin-react` + jsdom
- 컴포넌트: **@testing-library/react** + **@testing-library/user-event** (`getByRole` 우선, `getByTestId` 는 최후수단)
- 네트워크: **MSW v2** — handlers 는 유닛/E2E 공용 (`src/test/mocks/`). 함수-레벨 fetch mock 금지. `onUnhandledRequest: 'error'`.
- E2E: **Playwright** + Next 16 `experimental/testmode` (서버 fetch 가로채기). 루트 `e2e/` 디렉터리. 초기 스모크 3-5개.
- **코로케이션**: `foo.ts` + `foo.test.ts` 나란히.
- 공용 래퍼: `src/test/renderWithProviders.tsx` (매 테스트 fresh `QueryClient`), `src/test/resetStores.ts` (`beforeEach` 격리).
- **원칙**: 행동 테스트. 구현 세부·shallow·내부 state inspection 금지. 플레이크는 bug — 재시도로 숨기지 않음.
- **RSC 유닛 테스트 금지** (프레임워크 한계) → E2E 로 커버. Server Action 은 pure helper 추출해 helper 를 유닛.
- 커버리지 **하드 게이트 없음** — "변경 모듈에 테스트 있는가" PR 리뷰로 관리. CI workflow 는 #9 와 묶음.

## 접근성 · i18n · 디자인 토큰

> 상세: [`docs/axis-decisions/a11y-i18n-tokens.md`](docs/axis-decisions/a11y-i18n-tokens.md) (이슈 #7)

### 접근성 (a11y)
- `eslint-plugin-jsx-a11y/recommended` 유지.
- 접근성 복잡 컴포넌트는 **Radix UI primitives** (Dialog/Dropdown/Popover/Tabs/Tooltip 등) 선택 도입. 토스트는 **Sonner**. 수제 Modal/Dropdown 금지.
- Variants 관리: **CVA (`class-variance-authority`)** — Tailwind 클래스를 의미 variant 로 추상화.
- 런타임 검사: **`@axe-core/playwright`** 를 E2E 각 스펙 `afterEach`. **critical/serious → fail**, moderate/minor → warning.
- `focus-visible:` 유틸로 포커스 가시화 필수. 색만으로 상태 전달 금지. 아이콘 전용 버튼은 `aria-label` 필수.

### i18n
- **next-intl v3** 도입. 초기 로케일 **`ko` 1개**, 비-라우트 모드.
- `<html lang="ko">` **(현재 `"en"` 교정 대상)**.
- 메시지: `src/messages/ko.json`, 네임스페이스 분리 (`common`, `nav`, `auth`, `meeting`, `dashboard`, `recording`, `speakerMatch`, `errors`, `validation`).
- 날짜/숫자 포맷은 `src/lib/i18n/format.ts` 공용 유틸만. 직접 `toLocaleString` 호출 지양.
- #5 `errors/messages.ts` 와 #4 zod 메시지는 **점진 전환** — 함수 시그니처 유지, 내부에서 `useTranslations` 호출로 교체.

### 디자인 토큰 & 테마
- **Tailwind 4 `@theme` + CSS variables**. 색상은 **`oklch`** + **의미 토큰** (`bg` / `fg` / `primary` / `muted` / `border` / `danger` / `success` / `warning` + `-fg` variant). 9단계 scale 금지.
- 다크모드: 토큰 구조만 지원 (`data-theme="dark"`). 토글 UI 는 추후.
- **금지**: 임의 hex inline, `bg-red-500` 등 Tailwind 기본 색상 scale 직접 참조, `dark:` 유틸, `tailwind.config.ts` 색상 선언, 임의 spacing.
- 폰트: **Pretendard** (`next/font/local` self-host) + `system-ui` 폴백.
- 반응형 breakpoint 는 Tailwind 기본 유지.

## 로깅 · 에러 모니터링

> 상세: [`docs/axis-decisions/logging-monitoring.md`](docs/axis-decisions/logging-monitoring.md) (이슈 #8)

- **외부 벤더 미도입** — Sentry·Datadog·LogRocket·GlitchTip 등 전부 X. 사용자가 명시 요청하기 전까지 재제안 금지.
- `reportError(err, ctx)` 구현: **구조화 JSON** 을 `console.error` 로 출력. `{ ts, level, source, userId, message, name, stack, extra }` shape.
- 사용자 식별: **opaque `userId`** 만. 이메일/이름/전화 등 PII 금지.
- 민감정보 마스킹: `maskSensitive(obj)` 가 `password|token|secret|authorization|cookie|email|phone` 등 알려진 키를 재귀로 `'***'` 치환. 상세 정책은 #19 `web-security`.
- **클라→서버 수집 엔드포인트**는 열어둠 — flowmeet-api 설계 완료 시 `reportError` 내부에서 조건부 fetch 로 전송. 호출부 변경 없음.
- 서버 일반 로그는 지금 `console.*` 만. 구조화 로깅·수집 (`pino` 등) 은 **배포 플랫폼 결정 후**.
- 클라이언트 일반 로그는 **수집하지 않는다**. 모든 통찰은 에러 payload 로.

## 참고 문서

- TRD: `../outputs/TRD_조직지시_업무흐름관리플랫폼.md`
- PRD: `../outputs/PRD_조직지시_업무흐름관리플랫폼.md`
- UI PRD (웹): `../outputs/STITCH_UI_PRD_WEB_웹용.md`
- UI PRD (모바일): `../outputs/STITCH_UI_PRD_MOBILE_모바일용.md`
- 통합 UI PRD: `../outputs/STITCH_UI_PRD_조직업무흐름플랫폼.md`

## 코드 규칙

- App Router 우선 (`pages/` 사용 금지).
- 서버 컴포넌트가 기본, 클라이언트 상태가 필요한 경우에만 `"use client"`.
- Import alias는 `@/*` (= `src/*`).
- 타입은 `type` 선호, 확장이 필요한 경우에만 `interface`.
- 스타일은 Tailwind 유틸리티 우선. 반복 패턴은 컴포넌트로 추출.
- 폼 검증은 백엔드 응답을 신뢰하되 UX를 위한 선검증은 클라이언트에서 수행.
