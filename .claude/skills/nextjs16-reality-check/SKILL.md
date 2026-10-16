---
name: nextjs16-reality-check
description: flowmeet-web은 Next.js 16.4 (`cacheComponents: true` + `partialPrefetching: true`) + React 19.3 + Tailwind 4 환경이다. Opus/Sonnet의 학습 데이터는 Next 15 이하 전제가 섞여 있어 **그대로 적용하면 깨진다**. Next.js 코드·설정·캐시·라우팅·Server Actions·RSC·Tailwind 테마를 쓰거나 리뷰하기 전에 로드한다.
---

# Next.js 16 Reality Check — flowmeet-web

**이 리포는 당신이 아는 Next.js 가 아닐 수 있다.** Next 16 은 캐시 모델·설정 경로·테스트 primitive·심지어 Tailwind 토큰 선언 위치까지 Next 15 시대와 다르다. `AGENTS.md` 가 자동 생성해 둔 경고 블록이 이 사실을 명시한다.

Opus/Sonnet 학습 데이터로 Next 15 식 코드를 쓰면 **빌드는 통과해도 런타임 동작이 틀어진다** (예: 캐시가 안 먹거나, 오히려 과하게 먹거나). 이 스킬은 **코드 쓰기 전에** 로드해 가장 자주 틀어지는 지점을 검증한다.

---

## 1. 최우선 — 쓰기 전에 소스 확인

Next.js 설치가 되어 있다면 (`node_modules/next` 가 존재), **학습 지식보다 설치된 버전의 파일이 우선**이다.

```bash
# 설치 버전 확인
cat node_modules/next/package.json | grep '"version"'

# 공개 API (가장 신뢰) — next/cache, next/server, next/navigation 등
ls node_modules/next/*.d.ts

# 설정 defaults — cacheComponents, cacheLife, partialPrefetching 포함
sed -n '85,170p' node_modules/next/dist/esm/server/config-shared.js

# 테스트 primitives (Next 16 신규)
ls node_modules/next/dist/experimental/testing/server/
ls node_modules/next/dist/experimental/testmode/
```

**불확실하면 추측하지 말고 Read 로 파일을 열어 확인한다.**

---

## 2. Next 16 이 Next 15 와 다른 지점 (실측)

### 2.1. 캐시 모델 — `cacheComponents: true`

Next 15 전제: `fetch()` 는 **기본 캐시**. 캐시 해제는 `{ cache: 'no-store' }` 또는 `dynamic = 'force-dynamic'`.

**Next 16 (`cacheComponents: true` 전제) 전제: 기본 DYNAMIC. 캐시는 `'use cache'` 디렉티브로 명시 opt-in.**

```ts
// ❌ Next 15 식 (틀림, Next 16 에선 기본 캐시 안 됨)
async function getDashboard() {
  const res = await fetch(API);  // 호출마다 재요청
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

관련 공개 API (`next/cache`):

```
unstable_cache                 (레거시, 가급적 쓰지 않음)
revalidatePath / revalidateTag / updateTag / refresh
cacheTag       (= unstable_cacheTag)
cacheLife      (= unstable_cacheLife)  — profile 7종: default/seconds/minutes/hours/days/weeks/max 또는 커스텀
unstable_noStore
io
navigation / prefetch
```

세부 패턴은 [`docs/axis-decisions/data-fetching.md`](../../../docs/axis-decisions/data-fetching.md) 참조.

### 2.2. `partialPrefetching: true`

- `cacheComponents` 가 켜져 있지 않으면 **빌드 에러** (`node_modules/next/dist/esm/server/config.js:375-383`).
- 네비게이션 시 캐시된 shell 만 prefetch, dynamic 부분은 요청 시 스트림.
- 라우트 디자인 시 "RSC 가 dynamic / 섹션이 cached" 경계가 명확해야 효과.

### 2.3. `io()` 프리미티브 (Next 16 신규)

- `next/dist/server/request/io` — 명시적 I/O 호출용.
- 세부 사용법은 공식 docs 확인 (학습 지식에 없을 가능성 큼).

### 2.4. 테스트 primitives (Next 16 신규)

```ts
// 라우팅/미들웨어 설정 테스트
import { unstable_getResponseFromNextConfig, unstable_doesMiddlewareMatch }
  from 'next/dist/experimental/testing/server';

// Playwright 에서 서버 fetch 가로채기
import { interceptFetch, handleFetch, reader }
  from 'next/dist/experimental/testmode/fetch';
```

[`docs/axis-decisions/test-strategy.md`](../../../docs/axis-decisions/test-strategy.md) 가 Playwright + testmode 로 RSC fetch 모킹한다.

### 2.5. `error.tsx` / `global-error.tsx` — "use client" 필수

- App Router 의 에러 경계는 **client component 로만** 선언 가능.
- 상단에 `"use client"` 가 없으면 Next 가 거부.

### 2.6. Tailwind **4** — 토큰은 `@theme`

Next 15 시대 많은 레시피: `tailwind.config.ts` 의 `theme.extend.colors.primary = ...`.

**Tailwind 4 는 `tailwind.config.ts` 가 아니라 `@theme` 디렉티브로 선언**:

```css
/* src/app/globals.css */
@import "tailwindcss";

@theme {
  --color-primary: oklch(0.6 0.2 260);
  --color-primary-fg: oklch(0.99 0 0);
  /* ... */
}
```

세부는 [`docs/axis-decisions/a11y-i18n-tokens.md`](../../../docs/axis-decisions/a11y-i18n-tokens.md) §C.

### 2.7. React 19 신규 — `useFormStatus` 신중히

- RHF (#4 결정) 를 쓰는 폼은 `form.formState.isSubmitting` 사용, `useFormStatus` 는 **쓰지 않음** (혼용 금지).
- `useFormStatus` 는 Conform/베어 Server Action 플로우용.

### 2.8. Node 20.9+

- `engines` 명시 없어도 Next 16 은 20.9 미만에서 설치/실행 실패.
- nvm 사용 (`nvm use 22` 권장, CLAUDE.md 명시).

### 2.9. `AGENTS.md` 자동 생성 블록

- `next dev` 가 `AGENTS.md` 를 **쓰고 복원**한다. 삭제해도 다시 나타남.
- 블록 소스: `node_modules/next/dist/server/lib/generate-agent-files.js`.
- diff 에서 제거하지 말 것 — 커밋 트리에 포함해 두는 게 깨끗.

---

## 3. 자주 틀어지는 패턴 (금지)

| 패턴 | 왜 틀렸는가 | 대안 |
|---|---|---|
| `const res = await fetch(url)` 로 서버 데이터 로드 후 "캐시 걸릴 거야" 가정 | Next 16 `cacheComponents` 하에서 **기본 dynamic**. | `'use cache'` + `cacheLife` + `cacheTag`. 세부는 #2 |
| `tailwind.config.ts` 에 `colors: { primary: '#...' }` 선언 | Tailwind 4 는 `@theme` 가 공식 경로 | `src/app/globals.css` 의 `@theme { --color-primary: oklch(...) }` |
| `<div className="dark:bg-black">` 로 다크모드 변주 | 토큰 체계가 자동 변주 (`[data-theme="dark"]`) — `dark:` 유틸은 drift 소스 | 의미 토큰 사용. `dark:` 유틸 지양 |
| `pages/` 디렉터리 생성 | App Router 전용 (`pages/` 금지) | `src/app/` 사용 |
| RSC 안에서 `useState` / `useEffect` | 서버 컴포넌트에선 React 훅 X | `"use client"` 분리 |
| `error.tsx` 에 `"use client"` 생략 | 빌드/런타임 거부 | 최상단에 `"use client"` |
| `useRouter` from `next/router` | Pages Router 전용 (삭제됨) | `useRouter` from `next/navigation` |
| metadata 를 `<Head>` 로 설정 | App Router 는 `export const metadata` | `metadata` export 또는 `generateMetadata()` |
| `getServerSideProps` / `getStaticProps` | Pages Router 전용 | RSC 에서 직접 `await` |
| cookies/headers 를 `req.cookies` 로 | App Router 는 `cookies()` from `next/headers` | `import { cookies, headers } from 'next/headers'` |
| `useFormStatus` 와 RHF 혼용 | 상태 소스 두 곳으로 분산 | #4 결정 — RHF `formState.isSubmitting` 만 사용 |
| 서버 액션에서 throw | #5 결정 — Server Action 은 `FormActionResult`/`ActionResult` 반환 | 세션 만료 `redirect('/login')` 외 throw 금지 |

---

## 4. 결정 축과의 연결

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

## 5. 체크리스트 — 코드 작성·리뷰 시

- [ ] Next.js 버전을 **학습 지식이 아니라 `node_modules/next/package.json`** 으로 확인했는가?
- [ ] 서버 데이터 로드 코드에 **캐시 전략 (`'use cache'` 또는 명시적 dynamic)** 이 적용돼 있는가?
- [ ] Tailwind 색상 선언이 **`@theme`** 경로인가 (`tailwind.config.ts` 아님)?
- [ ] `dark:` 유틸 대신 **의미 토큰 + `data-theme`** 를 쓰는가?
- [ ] `error.tsx` 상단에 **`"use client"`** 가 있는가?
- [ ] `useRouter` 를 `next/navigation` 에서 import 했는가 (`next/router` 아님)?
- [ ] RSC 안에서 React 훅을 쓰지 않는가 (훅 쓰면 `"use client"` 분리)?
- [ ] Server Action 이 throw 하지 않는가 (`FormActionResult`/`ActionResult` 반환 — #4·#5)?
- [ ] 폼에서 **RHF `formState.isSubmitting`** 사용인가 (`useFormStatus` 혼용 금지)?
- [ ] 불확실한 지점을 **node_modules 소스로 검증** 했는가?

---

## 6. 참고 (실제 Next 16 소스 포인터)

- 공개 API 타입 선언:
  - `node_modules/next/cache.d.ts` — `cacheLife`, `cacheTag`, `revalidate*`, `io`, `navigation`, `prefetch`
  - `node_modules/next/navigation.d.ts`
  - `node_modules/next/headers.d.ts`
  - `node_modules/next/server.d.ts`
- 설정 defaults: `node_modules/next/dist/esm/server/config-shared.js:85-170`
- `partialPrefetching` 검증: `node_modules/next/dist/esm/server/config.js:375-383`
- 테스트 primitives: `node_modules/next/dist/experimental/testing/server/`, `node_modules/next/dist/experimental/testmode/`
- 에이전트 블록 생성기: `node_modules/next/dist/server/lib/generate-agent-files.js`
- AGENTS.md (프로젝트 루트) — Next 가 쓰고 복원하는 경고 블록
