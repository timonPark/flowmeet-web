---
name: app-router-layering
description: flowmeet-web 의 디렉터리 책임·레이어 의존 방향·Server/Client 컴포넌트 경계·데이터 페칭 레이어 규칙. 새 라우트·피처·공용 모듈을 만들거나 리뷰할 때 로드. api 리포의 `spring-layering` 과 같은 자리를 web 에서 담당.
---

# App Router Layering — flowmeet-web

Next.js 16 App Router + React 19 환경에서 **디렉터리별 책임**, **레이어 간 의존 방향**, **Server/Client 컴포넌트 경계**, **데이터 페칭 레이어 분리** 를 결정한다. api 리포의 `spring-layering` (controller/application/domain/infrastructure) 과 동일한 자리에서 web 측 구조 룰을 정의.

> 이 스킬은 **구조 가드** — 각 영역의 세부 패턴은 Phase 1 결정 축 문서 (`docs/axis-decisions/*.md`) 가 소스. 중복 서술 없이 참조로 연결.

---

## 1. 디렉터리 책임

```
src/
├─ app/                    # ① Routes — Next 가 정한 파일 (page/layout/error/route/proxy 등)
│  ├─ (dashboard)/
│  │  ├─ layout.tsx
│  │  ├─ error.tsx
│  │  └─ page.tsx
│  ├─ layout.tsx
│  ├─ providers.tsx        # Provider 트리 (QueryClient, AuthProvider) — client component
│  └─ global-error.tsx
├─ features/               # ② Feature slices — 피처별 서버/클라 컴포넌트, Server Actions, 피처 전용 훅
│  └─ meeting/
│     ├─ CreateMeetingForm.tsx
│     ├─ CreateMeetingForm.test.tsx
│     ├─ actions.ts        # Server Actions
│     ├─ queries.ts        # 피처 전용 RQ 쿼리 함수
│     └─ useMeetingDraft.ts
├─ components/             # ③ Shared UI — 피처 간 재사용되는 범용 UI (피처 전용은 features/ 안)
│  ├─ Button.tsx
│  └─ DataTable.tsx
├─ lib/                    # ④ Platform — 프레임워크·앱 비의존 유틸·클라이언트·스토어
│  ├─ api/
│  │  ├─ _core.ts          # URL 조립·에러 매핑·응답 파싱·ApiError (중립)
│  │  ├─ serverClient.ts   # cookies/headers 로 token 주입, 401 refresh retry → redirect
│  │  └─ browserClient.ts  # AuthContext/memory token, 401 → Server Action refresh 위임
│  ├─ stores/
│  │  ├─ uiStore.ts        # 사이드바·모달·토스트 큐
│  │  ├─ recordingStore.ts # 녹음 세션 메타 (chunk 는 전용 IndexedDB 레이어)
│  │  └─ speakerMatchStore.ts
│  ├─ validation/
│  │  ├─ schemas/<domain>.ts   # zod 스키마 (클라·서버 공용)
│  │  └─ errors.ts         # 폼 매핑 (codeToMessage import 재사용)
│  ├─ errors/
│  │  ├─ ApiError.ts
│  │  ├─ codes.ts
│  │  ├─ messages.ts       # 중앙 매핑 (추후 next-intl 로 치환)
│  │  ├─ handle.ts
│  │  ├─ reportError.ts    # 구조화 JSON console.error (외부 벤더 미도입)
│  │  ├─ maskSensitive.ts
│  │  └─ ErrorBoundary.tsx # react-error-boundary 래퍼
│  ├─ forms/               # RHF Controller 래퍼: TextField / Select / DatePicker ...
│  ├─ ui/
│  │  └─ variants.ts       # CVA button/badge/... variants
│  └─ i18n/
│     └─ format.ts         # 날짜·숫자 포맷 유틸
├─ messages/
│  └─ ko.json              # next-intl 메시지 (common/nav/auth/meeting/dashboard/errors/validation...)
├─ test/                   # 테스트 공용 — mocks, factories, renderWithProviders, resetStores
└─ (optional) features-only utilities stay inside features/<domain>/
```

**레이어 번호**는 §2 의존 방향에 쓰임.

### 1.1. 어디에 두는지 결정 트리

```
새 모듈을 어디에 둘지?

1. Next 가 정한 파일? (page/layout/error/route/proxy/instrumentation/default)
   → src/app/<path>/<name>.tsx

2. 특정 도메인에만 쓰이는 UI·훅·액션?
   → src/features/<domain>/<Name>.tsx

3. 두 피처 이상이 공유하는 범용 UI?
   → src/components/<Name>.tsx  (스토리북 성격)

4. 네트워크·에러·스토어·검증·토큰·토스트 큐 등 플랫폼 유틸?
   → src/lib/<subarea>/<name>.ts

5. 번역 가능한 사용자 문자열?
   → src/messages/ko.json 네임스페이스 추가

6. 테스트 공용 유틸?
   → src/test/<name>.ts(x)
```

**결정 못 하면 features/<domain>/ 에 두는 걸 디폴트** — 나중에 두 피처가 공유하면 그때 승격 (`features/<a>` + `features/<b>` → `components/` 또는 `lib/`).

---

## 2. 레이어 의존 방향

**단방향 그래프**. 역방향 import 는 PR 리뷰에서 반려.

```
┌─────────────────────────┐
│  ① app/   (route edge)  │
└─────────────┬───────────┘
              ↓
┌─────────────────────────┐
│  ② features/<domain>/   │
└─────────┬───────────┬───┘
          ↓           ↓
┌──────────────────┐ ┌─────────────────┐
│ ③ components/    │ │ ④ lib/          │
└────────┬─────────┘ └────────┬────────┘
         └──────→ ④ lib/ ─────┘
```

### 금지 패턴

- **역방향**: `lib/` → `features/`, `components/` → `features/`, `features/<a>` → `features/<b>`.
  - 두 피처가 공유하는 코드는 `components/` 또는 `lib/` 로 **올려야** 함.
- **features 간 상호 의존**: `features/meeting` 과 `features/task` 가 서로 import 금지. 공유 지점을 `lib/` 또는 `components/` 로.
- **app 을 다른 레이어가 import**: `app/` 은 "edge". 다른 레이어가 route 를 참조하지 않는다.
- **순환 import** (`lib/a` ↔ `lib/b`): 공통 추출로 끊음.

### 측정·강제

- 지금은 **PR 리뷰 체크** 로 강제. ESLint 플러그인 (`eslint-plugin-boundaries` 등) 도입은 추후 `ts-style` 와 함께 재검토.

---

## 3. Server / Client 컴포넌트 경계

### 3.1. 기본값은 Server Component

- `"use client"` 디렉티브가 **없으면 Server Component**. 이게 기본.
- "use client" 는 **상태·이벤트·브라우저 API·훅이 필요한 트리의 꼭대기만**. 자식은 자동으로 클라이언트 번들.
- 꼭대기 설정이 중요 — 아래에 있는 Server-side 로 가능한 로직을 괜히 클라이언트로 끌어내지 말 것.

### 3.2. 배치 원칙

```
page.tsx (Server)
  ↓
  ├─ 서버 데이터 로드 ('use cache' 또는 Server Action)
  ├─ 서버 전용 변환 (date formatting, i18n via server)
  └─ 클라이언트 상호작용 섹션 → <ClientSection> ("use client")
       └─ onClick, useState, useQuery, 녹음, 폼 등
```

- 패턴: **서버에서 데이터를 받아 props 로 클라이언트 컴포넌트에 넘긴다**. "use client" 가 fetch 하지 않도록.

### 3.3. 환경별 전용 API

| API | 어디서 |
|---|---|
| `cookies()`, `headers()`, `redirect()`, `revalidateTag()` | Server Component / Server Action / Route Handler (`route.ts`) |
| `useState`, `useEffect`, `useQuery`, `useForm`, `onClick` | `"use client"` 안 |
| `localStorage`, `window`, DOM API | `"use client"` + `useEffect` (SSR 중엔 접근 X) |
| `fs/promises`, `crypto` (Node) | Server 전용 — `"use client"` 에서 import 금지 |

### 3.4. 서버·클라 공용 모듈

- `lib/validation/schemas/*.ts` (zod) — 양쪽 공용 (#4)
- `lib/errors/codes.ts`, `messages.ts`, `maskSensitive.ts` — 양쪽 공용 (단, 메시지 i18n 함수는 이후 context 에 따라 분기)
- `lib/api/_core.ts` — 중립 유틸
- `lib/api/serverClient.ts` — **서버 전용** (`"use client"` 에서 import 금지)
- `lib/api/browserClient.ts` — **클라이언트 전용** (서버에서 import 금지)

### 3.5. 금지 패턴

- `"use client"` 안에서 `cookies()`·`headers()`·`fs/*` import
- 한 함수가 "서버/클라 공통" 이라고 선언해 두고 환경 분기 (`if (typeof window === 'undefined')`) — **금지**. 두 파일로 분리 (`serverClient.ts` / `browserClient.ts`).
- `"use client"` 를 트리 밑바닥에 끼워 넣는 식의 "미세 use client" — 상위를 서버로 유지하기 위해 꼭대기 조정.

---

## 4. 데이터 페칭 레이어

### 4.1. 레이어 책임 (요약, 상세는 [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md))

| 레이어 | 책임 |
|---|---|
| `lib/api/_core.ts` | URL 조립, 응답 파싱, `ApiError` 생성, 공통 헤더 |
| `lib/api/serverClient.ts` | 서버 전용 — `cookies()` 로 token 추출, 401 → Refresh Token 회전 → 실패 시 `redirect('/login')` |
| `lib/api/browserClient.ts` | 클라 전용 — `AuthContext` memory token, 401 → Server Action refresh 위임 |
| `features/<domain>/queries.ts` | RQ `queryFn` / query key 선언. 각 쿼리는 **반드시** `browserClient` 경유 |
| `features/<domain>/actions.ts` | Server Actions. **반드시** `serverClient` 경유. 반환은 `FormActionResult`/`ActionResult` ([#5](../../../docs/axis-decisions/error-ux.md)) |
| RSC fetch (서버 컴포넌트 안의 직접 `await`) | `serverClient` 경유. 캐시는 `'use cache'` + `cacheLife` + `cacheTag` 로 명시 opt-in |

### 4.2. 금지 패턴

- `features/*` 또는 `components/*` 가 **직접 `fetch()` 호출** → `lib/api/*` 경유 필수
- `lib/stores/*` 가 **직접 `fetch()` 호출** → store 안 네트워크 금지 ([#3](../../../docs/axis-decisions/state-management.md))
- 한 함수에서 `serverClient` 와 `browserClient` 를 분기해 호출 → 호출 지점이 서버냐 클라냐에 따라 **파일 분리**
- 서버 데이터를 `lib/stores/*` (Zustand) 에 저장 → RQ 가 가진 데이터는 Zustand 에 넣지 않는다 ([#3](../../../docs/axis-decisions/state-management.md))

---

## 5. Server Action 레이어

### 5.1. 위치

- 각 피처의 `features/<domain>/actions.ts` 에 집합. 파일 상단 `"use server"`.
- 간단한 액션 (단일 모듈) 은 Server Action 을 호출하는 Client Component 와 같은 디렉터리.

### 5.2. 구조

```ts
// features/meeting/actions.ts
'use server';

import { serverClient } from '@/lib/api/serverClient';
import { createMeetingSchema } from '@/lib/validation/schemas/meeting';
import { createMeetingLogic } from './createMeetingLogic'; // pure helper
import type { ActionResult } from '@/lib/errors/ActionResult';

export async function createMeetingAction(input: unknown): Promise<ActionResult<{ id: string }>> {
  const parsed = createMeetingSchema.safeParse(input);
  if (!parsed.success) return mapZodErrorToActionResult(parsed.error);
  return createMeetingLogic(parsed.data, serverClient);
}
```

- **비즈니스 로직은 pure helper (`createMeetingLogic`) 로 추출** → Vitest 로 유닛 테스트 ([#6](../../../docs/axis-decisions/test-strategy.md)).
- Server Action 함수는 **얇게** — schema 검증 + helper 호출 + 결과 매핑.
- **throw 금지** — `ActionResult`/`FormActionResult` 로 반환. 예외: 세션 만료 `redirect('/login')`.

### 5.3. 금지 패턴

- Server Action 안에서 직접 `fetch` → `serverClient` 경유
- Server Action 이 거대한 비즈니스 로직 포함 → pure helper 로 분리
- Server Action 이 throw → `ActionResult` 반환
- Server Action 호출 결과를 Zustand 에 중복 저장 → RQ 가 담당

---

## 6. 라우팅 구조

### 6.1. Route groups

- `src/app/(dashboard)/` — 레이아웃·에러 바운더리 공유. URL 에는 안 나타남.
- 섹션별 그룹: `(dashboard)`, `(meeting)`, `(auth)`, `(settings)` — 섹션 레이아웃 + `error.tsx` 선택적 배치.
- 세부는 [#5 error-ux](../../../docs/axis-decisions/error-ux.md) §B.

### 6.2. 라우트 파일은 얇게

- `page.tsx` 는 **데이터 로드 + 레이아웃 조립** 수준. 복잡 로직은 `features/<domain>/` 로 위임.
- 불가피하게 긴 로직은 **pure helper** (`features/<domain>/<name>Logic.ts`) 로 분리하여 테스트 가능하게.

### 6.3. 특수 파일

- `error.tsx` (섹션) · `global-error.tsx` (루트) — 둘 다 `"use client"` 필수. `global-error.tsx` 는 자체 `<html>`/`<body>` 요구 ([#13 nextjs16-reality-check](../nextjs16-reality-check/SKILL.md) §2.5).
- `loading.tsx` — Suspense fallback.
- `not-found.tsx` — 404.
- `default.tsx` — parallel route slot 기본값.
- `proxy.ts` (리포 레벨, Next 16) — 미들웨어 교체.

---

## 7. Providers 트리

```tsx
// src/app/providers.tsx (client component)
'use client';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProviderWrapper>
      <AuthProvider>
        {children}
      </AuthProvider>
    </QueryClientProviderWrapper>
  );
}

// src/app/layout.tsx (server component)
import { Providers } from './providers';
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

- `QueryClient` 는 **매 요청 fresh 인스턴스** (`cacheComponents` 하에서 중요, 세션 간 유출 방지) ([#2](../../../docs/axis-decisions/data-fetching.md)).
- `Zustand` 는 **Provider 없음** — slice 모듈이 싱글턴 ([#3](../../../docs/axis-decisions/state-management.md)).
- `<html lang="ko">` 는 i18n·a11y 결정 ([#7](../../../docs/axis-decisions/a11y-i18n-tokens.md)).

---

## 8. 테스트 코로케이션 (요약, 상세 #6)

- 유닛/컴포넌트 테스트는 **같은 디렉터리에 `*.test.ts(x)`** 로 코로케이트.
- E2E 는 루트 `e2e/`.
- `src/test/` 는 공용 유틸 (`renderWithProviders`, `resetStores`, MSW handlers, factories).
- 레이어링 룰 자체도 테스트로 보강: `src/test/architecture.test.ts` 에서 `lib/*` 가 `features/*` 를 import 하지 않는지 등 (추후 도입 — 지금은 PR 리뷰).

---

## 9. 체크리스트 — 새 모듈 작성·리뷰 시

- [ ] 모듈이 **디렉터리 책임** (§1) 과 일치하는가? (피처 전용인데 `components/` 에 두지 않았는가 등)
- [ ] **의존 방향** (§2) 을 지키는가? 역방향 / 피처 간 교차 import 없는가?
- [ ] Server/Client **경계** (§3) 가 명확한가? `"use client"` 가 꼭대기만 쓰이는가?
- [ ] fetch 가 **`lib/api/*` 경유** 인가? (직접 `fetch` 금지)
- [ ] 서버/클라 wrapper 를 **분리** 사용하는가? (`serverClient` 가 `"use client"` 에서 import 되지 않는가)
- [ ] Server Action 이 `"use server"` 상단 + **얇은 조립** 인가? 비즈니스 로직은 pure helper?
- [ ] Server Action 이 `ActionResult`/`FormActionResult` 반환하는가 (throw 금지)?
- [ ] Store 가 **네트워크 호출 안** 하는가? 서버 데이터를 복제 저장 안 하는가?
- [ ] 라우트 파일이 **얇은가**? 복잡 로직을 `features/<domain>/` 로 위임?
- [ ] `global-error.tsx` 가 자체 `<html>`/`<body>` 를 포함하는가?

---

## 10. 참고

- 결정 축: [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md), [#3 state-management](../../../docs/axis-decisions/state-management.md), [#4 form-validation](../../../docs/axis-decisions/form-validation.md), [#5 error-ux](../../../docs/axis-decisions/error-ux.md), [#6 test-strategy](../../../docs/axis-decisions/test-strategy.md), [#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md)
- 관련 스킬: [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md), [`ts-style`](../ts-style/SKILL.md)
- api 리포 대응 스킬: `../../flowmeet-api/.claude/skills/spring-layering/SKILL.md` — 동일 자리의 백엔드 버전 (참고용, 구조만)
