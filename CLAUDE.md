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
- 비-폼 Server Action 은 `ActionResult<T>` (ok discriminated union). 폼 `FormActionResult` 와 shape 통일. **throw 금지**.
- 토스트는 `uiStore.toast.{error,success,...}` 추상 API. 중복 deduping은 `uiStore` 가 보장.
- 로깅 훅: **`reportError(err, ctx)`** 추상. 지금은 `console.error`, #8 후 Sentry 로 **구현만 교체**.

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
