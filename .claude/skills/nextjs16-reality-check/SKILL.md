---
name: nextjs16-reality-check
description: flowmeet-web은 Next.js 16.4 (`cacheComponents: true` + `partialPrefetching: true`) + React 19.3 + Tailwind 4 환경이다. Opus/Sonnet의 학습 데이터는 Next 15 이하 전제가 섞여 있어 **그대로 적용하면 깨진다**. Next.js 코드·설정·캐시·라우팅·Server Actions·RSC·Tailwind 테마를 쓰거나 리뷰하기 전에 로드한다.
---

# Next.js 16 Reality Check — flowmeet-web

**이 리포는 당신이 아는 Next.js 가 아닐 수 있다.** Next 16 은 캐시 모델·설정 경로·테스트 primitive·심지어 Tailwind 토큰 선언 위치까지 Next 15 시대와 다르다. `AGENTS.md` 가 자동 생성해 둔 경고 블록이 이 사실을 명시한다.

Opus/Sonnet 학습 데이터로 Next 15 식 코드를 쓰면 **런타임 동작이 틀어질 수 있다** (예: 캐시 모델이 바뀌어 의도한 캐시가 안 먹거나 반대로 과하게 먹음). 이 스킬은 **코드 쓰기 전에** 로드해 가장 자주 틀어지는 지점을 검증한다.

### 두 성격의 규칙을 섞지 말 것

- **Next.js 런타임 요구사항** — Next 가 강제. 안 지키면 빌드/런타임 실패.
- **flowmeet-web 프로젝트 규칙** — Phase 1 축 결정 (`docs/axis-decisions/*.md`). 다른 리포에선 다를 수 있음.

이 스킬은 두 성격을 분리해 표기한다. §2 는 **Next 런타임**, §3 은 **프로젝트 규칙**.

---

## 1. 최우선 — 쓰기 전에 소스 확인

Next.js 설치가 되어 있다면 (`node_modules/next` 가 존재), **학습 지식보다 설치된 버전의 파일이 우선**이다. 주 수단은 **Read** 로 파일을 여는 것 — 아래 쉘 명령은 보조적 조회용.

- 설치 버전: `node_modules/next/package.json` 의 `"version"` 필드 Read.
- 공개 API 선언 (가장 신뢰): `node_modules/next/*.d.ts` — 특히 `cache.d.ts`, `navigation.d.ts`, `headers.d.ts`, `server.d.ts`.
- 설정 defaults: `node_modules/next/dist/esm/server/config-shared.js` 를 열어 `cacheComponents` / `cacheLife` / `partialPrefetching` 선언 확인.
- 테스트 primitives (Next 16 신규): `node_modules/next/dist/experimental/testing/server/`, `node_modules/next/dist/experimental/testmode/`.

**불확실하면 추측하지 말고 Read 로 파일을 열어 확인한다.** 아래는 보조 명령 예시 (환경에 따라 Read 가 더 확실):

```bash
cat node_modules/next/package.json | grep '"version"'
ls node_modules/next/*.d.ts
ls node_modules/next/dist/experimental/testing/server/
ls node_modules/next/dist/experimental/testmode/
```

---

## 2. Next 16 런타임 — Next 15 와 다른 지점

이 섹션의 규칙은 **Next.js 자체가 강제**. 지키지 않으면 빌드/런타임 거부.

### 2.1. 캐시 모델 — `cacheComponents: true`

Next 15 전제: `fetch()` 는 **기본 캐시**. 캐시 해제는 `{ cache: 'no-store' }` 또는 `dynamic = 'force-dynamic'`.

**Next 16 (`cacheComponents: true` 전제): `'use cache'` 로 감싸지 않은 함수는 캐시 대상이 아니며 요청 시 평가된다.** 결과적으로 "자동 캐시" 가정이 깨짐. 실제 결과 캐시 여부는 `'use cache'` 적용 + 요청-동적 입력 (`cookies()`, `headers()`, dynamic params 등) 여부에 따라 결정 — 단순히 "모든 fetch 가 호출마다 재요청" 은 아니다.

```ts
// ❌ Next 15 식 가정 (Next 16 에선 암묵적 캐시 없음)
async function getDashboard() {
  const res = await fetch(API);  // 캐시 자동 적용 가정 → 틀림
  return res.json();
}

// ✅ Next 16 — 명시 opt-in
async function getDashboard() {
  'use cache';
  cacheLife('minutes');
  cacheTag('dashboard');
  const res = await fetch(API);
  return res.json();
}
```

**`next/cache` 공개 export (`node_modules/next/cache.d.ts` 실측)**:

```
unstable_cache                                   (레거시, 가급적 쓰지 않음)
revalidatePath / revalidateTag / updateTag / refresh
unstable_noStore
io
cacheTag       (별칭: unstable_cacheTag)
cacheLife      (별칭: unstable_cacheLife) — profile 7종: default / seconds / minutes / hours / days / weeks / max, 또는 커스텀
navigation / prefetch                            (from `next/dist/server/request/cache-stages`, re-export)
```

세부 패턴은 [`docs/axis-decisions/data-fetching.md`](../../../docs/axis-decisions/data-fetching.md) 참조.

### 2.2. `partialPrefetching: true`

- `cacheComponents` 가 켜져 있지 않으면 **빌드 에러** — 현재 16.4.0 설치본 기준: `node_modules/next/dist/esm/server/config.js` 의 `partialPrefetching` 검증 로직 (라인 번호는 패치 버전마다 바뀔 수 있음, 안정 참조 아님).
- 네비게이션 시 캐시된 shell 만 prefetch, dynamic 부분은 요청 시 스트림.
- 라우트 디자인 시 "RSC 가 dynamic / 섹션이 cached" 경계가 명확해야 효과.

### 2.3. `io()` 프리미티브 (Next 16 신규)

- `next/cache` 에서 공개 export (실측). 명시적 I/O 호출용.
- 세부 사용법은 Next 16 공식 docs 또는 공개 타입 선언 확인 (학습 지식에 없을 가능성 큼).

### 2.4. 테스트 primitives (Next 16 신규)

- 현재 설치본 기준 포인터 (경로는 패치에서 바뀔 수 있음):
  - 라우팅/미들웨어 설정 테스트: `next/dist/experimental/testing/server` → `unstable_getResponseFromNextConfig`, `unstable_doesMiddlewareMatch`
  - Playwright 서버 fetch 가로채기: `next/dist/experimental/testmode/fetch` → `interceptFetch`, `handleFetch`, `reader`

[`docs/axis-decisions/test-strategy.md`](../../../docs/axis-decisions/test-strategy.md) 가 Playwright + testmode 로 RSC fetch 모킹한다.

### 2.5. `error.tsx` / `global-error.tsx` — "use client" 필수 + 구조 요구 분리

- 두 파일 **모두 `"use client"` 필수**. 상단 없으면 Next 가 거부.
- **차이**:
  - `error.tsx` — 섹션 레이아웃 안에서 fallback. 해당 섹션 UI 만 교체.
  - `global-error.tsx` — **루트 레이아웃을 대체**하므로 **자체 `<html>`·`<body>` 를 포함해야 함**. 일반 폴백처럼 `<div>` 만 돌려주면 안 됨.
- 세부 UX 는 [`docs/axis-decisions/error-ux.md`](../../../docs/axis-decisions/error-ux.md).

### 2.6. React 19 — `useFormStatus` 는 Server-Action-only

- `useFormStatus` 는 **동봉된 네이티브 `<form action={serverAction}>`** 내에서 pending 을 읽는 훅. RHF 같은 폼 라이브러리를 쓰면 RHF 가 자체 `isSubmitting` 을 제공하므로 **같은 폼 안에서 두 소스 혼용 금지** (상태 정합성 깨짐).
- 이 구분은 Next 16 자체가 강제하지 않지만 React 19 의 API 설계상 **동시에 쓸 이유가 없다** — 프로젝트 규칙으로 §3 에서 다시 명시.

### 2.7. Node 20.9+

- Next 16 은 20.9 미만에서 설치/실행 실패 (`engines` 명시 유무와 무관).
- nvm 사용 (`nvm use 22` 권장, CLAUDE.md 명시).

### 2.8. `AGENTS.md` 자동 생성 블록

- `next dev` 가 `AGENTS.md` 를 **쓰고 복원**한다. 삭제해도 다시 나타남.
- 블록 소스: `node_modules/next/dist/server/lib/generate-agent-files.js` (현재 설치본 기준 포인터).
- diff 에서 제거하지 말 것 — 커밋 트리에 포함해 두는 게 깨끗.

---

## 3. flowmeet-web 프로젝트 규칙

이 섹션의 규칙은 **이 프로젝트의 결정**이지 Next.js 자체 요구가 아니다. 다른 Next.js 프로젝트에선 다를 수 있다.

| 패턴 | 왜 틀렸는가 (이 프로젝트에서) | 대안 |
|---|---|---|
| 서버 데이터 로드 시 `'use cache'` 없이 "암묵 캐시" 가정 | #2 결정상 캐시는 명시 opt-in 만. Next 16 런타임 동작과도 정합. | `'use cache'` + `cacheLife` + `cacheTag`. 세부 #2 |
| `tailwind.config.ts` 에 색상 선언 | #7 결정 — Tailwind 4 자체는 레거시 config 지원하나, 이 프로젝트는 `@theme` 를 공식 경로로 고정 (의미 토큰 drift 방지) | `src/app/globals.css` 의 `@theme { --color-primary: oklch(...) }` |
| `<div className="dark:bg-black">` 로 다크모드 변주 | #7 결정 — Tailwind `dark:` 유틸은 지원되지만 토큰 자동 변주 (`[data-theme="dark"]`) 와 중복 drift. | 의미 토큰 사용. `dark:` 유틸 지양 |
| Server Action 에서 `throw` | #5 결정 — Server Action 은 `FormActionResult`/`ActionResult` 반환 (throw 자체는 Next 가 허용) | 세션 만료 `redirect('/login')` 외 throw 금지 |
| `useFormStatus` 와 RHF 혼용 (§2.6 참고) | #4 결정 — 상태 소스 두 곳으로 분산 | RHF `formState.isSubmitting` 만 사용 |

Next.js 자체가 삭제했거나 금지한 패턴 (프로젝트 규칙이 아니라 **런타임 요구**) 은 아래 §4 참조.

---

## 4. Next.js 런타임이 금지한 패턴

이 섹션은 Next.js **자체가 강제**. flowmeet-web 결정과 무관하게 Next 를 쓰면 지켜야 함.

| 패턴 | 왜 안 되는가 | 대안 |
|---|---|---|
| App Router 리포에 `pages/` 디렉터리 혼용 | Next 16 은 App Router 기본, 두 라우터 혼용은 라우팅 충돌 소지 | `src/app/` 만 사용 |
| RSC 안에서 `useState` / `useEffect` 등 React 훅 | 서버 컴포넌트는 훅 비지원 | 상태 쓰는 하위 컴포넌트를 `"use client"` 로 분리 |
| `useRouter` from `next/router` | Pages Router 전용 — 삭제됨 | `useRouter` from `next/navigation` |
| metadata 를 `<Head>` 로 설정 | App Router 는 metadata API | `export const metadata` 또는 `generateMetadata()` |
| `getServerSideProps` / `getStaticProps` | Pages Router 전용 | RSC 에서 직접 `await` |
| cookies/headers 를 `req.cookies` 로 | App Router 는 non-request API | `import { cookies, headers } from 'next/headers'` |
| `error.tsx` / `global-error.tsx` 에 `"use client"` 생략 | Next 가 거부 | 최상단 `"use client"` |
| `global-error.tsx` 가 `<html>`/`<body>` 생략 | 루트 레이아웃 대체이므로 HTML 구조 필요 | 자체 `<html><body>` 반환 |

---

## 5. 결정 축과의 연결

이 스킬은 **단독 가드** 가 아니라 Phase 1 결정 축들의 **상위 reality check**. 특정 영역의 세부 규칙은 각 축 결정 문서 참조:

| 영역 | 참조 |
|---|---|
| 서버/클라이언트 fetch wrapper, `'use cache'` 전략 | [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md) |
| 상태 관리 (Zustand + Context) | [#3 state-management](../../../docs/axis-decisions/state-management.md) |
| 폼 (RHF + zod + Server Actions) | [#4 form-validation](../../../docs/axis-decisions/form-validation.md) |
| 에러 경계 (`error.tsx`·`global-error.tsx`·`ErrorBoundary`) | [#5 error-ux](../../../docs/axis-decisions/error-ux.md) |
| 테스트 (`experimental/testmode` 포함) | [#6 test-strategy](../../../docs/axis-decisions/test-strategy.md) |
| Tailwind 4 `@theme` 토큰, next-intl v3 | [#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) |
| `reportError` 추상 (외부 벤더 미도입) | [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md) |

---

## 6. 체크리스트 — 코드 작성·리뷰 시

- [ ] Next.js 버전을 **학습 지식이 아니라 `node_modules/next/package.json`** 에서 확인했는가?
- [ ] 서버 데이터 로드 코드에 **명시 캐시 전략 (`'use cache'` 또는 요청-동적)** 이 적용돼 있는가?
- [ ] Tailwind 색상 선언이 **`@theme`** 경로인가 (이 프로젝트 정책 — #7)?
- [ ] `dark:` 유틸 대신 **의미 토큰 + `data-theme`** 를 쓰는가 (#7)?
- [ ] `error.tsx` 와 `global-error.tsx` 상단에 **`"use client"`** 가 있는가? `global-error.tsx` 는 **자체 `<html>`/`<body>`** 를 포함하는가?
- [ ] `useRouter` 를 `next/navigation` 에서 import 했는가 (`next/router` 삭제됨)?
- [ ] RSC 안에서 React 훅을 쓰지 않는가 (훅 쓰면 `"use client"` 분리)?
- [ ] Server Action 이 throw 하지 않는가 (`FormActionResult`/`ActionResult` 반환 — #4·#5)?
- [ ] 폼에서 **RHF `formState.isSubmitting`** 사용인가 (`useFormStatus` 혼용 금지 — #4)?
- [ ] 불확실한 지점을 **node_modules 소스 Read 로 검증** 했는가?

---

## 7. 참고 (실제 Next 16 소스 포인터)

> **주의**: 아래 `dist/*` 경로와 라인 번호는 **현재 16.4.0 설치본 기준** 진단 포인터. 패치/마이너 업데이트에서 바뀔 수 있고 **공개 API 가 아니다**. 가능하면 **공개 `*.d.ts`** 와 공식 docs 를 우선 참조.

- **공개 API 타입 선언** (안정 참조):
  - `node_modules/next/cache.d.ts` — `cacheLife`, `cacheTag`, `revalidate*`, `io`, `navigation`, `prefetch`, `unstable_*`
  - `node_modules/next/navigation.d.ts`
  - `node_modules/next/headers.d.ts`
  - `node_modules/next/server.d.ts`
- **설치본 진단 포인터** (현재 16.4.0 기준, 변동 가능):
  - `node_modules/next/dist/esm/server/config-shared.js` — `cacheComponents` / `cacheLife` 기본 선언
  - `node_modules/next/dist/esm/server/config.js` — `partialPrefetching` 유효성 검증
  - `node_modules/next/dist/experimental/testing/server/`
  - `node_modules/next/dist/experimental/testmode/`
  - `node_modules/next/dist/server/lib/generate-agent-files.js` — AGENTS.md 자동 복원 로직
- `AGENTS.md` (프로젝트 루트) — Next 가 쓰고 복원하는 경고 블록
