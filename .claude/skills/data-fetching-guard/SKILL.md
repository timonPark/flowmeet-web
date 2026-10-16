---
name: data-fetching-guard
description: flowmeet-web 데이터 페칭 코드 작성·리뷰 가드 — fetch wrapper (`serverClient`/`browserClient`/`publicApiClient`), `'use cache'` + `cacheLife`/`cacheTag`, React Query 패턴, 401 refresh 흐름, 페이지네이션·재시도·취소. 네트워크/캐시 관련 코드 쓸 때 로드.
---

# Data Fetching Guard — flowmeet-web

[#2 data-fetching](../../../docs/axis-decisions/data-fetching.md) 결정 + [§A1-보강 (`'use cache'` 인증 경계)](../../../docs/axis-decisions/data-fetching.md#a1-보강-use-cache--인증-경계-사용자-간-캐시-오염-방지) 을 **코드로 적용할 때** 쓰는 가드. "클라이언트 어디로 import 할지, 캐시는 어떻게 쓸지, 토큰은 어디서 주입할지, 401 은 어떻게 돌릴지" 를 코드 수준으로 catch.

> 결정 축이 "무엇을" 정했다면 이 스킬은 "그래서 코드를 어떻게 쓰나" 중심.

---

## 1. fetch wrapper 세 종류 — 분리 사용

```
src/lib/api/
├─ _core.ts            # 중립 — URL 조립, parseResponse, ApiError 생성, 공통 헤더
├─ serverClient.ts     # 서버 전용 — cookies() 로 token 자동 주입, 401 refresh retry
├─ browserClient.ts    # 클라 전용 — AuthContext memory token, 401 → Server Action refresh
└─ publicApiClient.ts  # 서버 전용 — 인증 토큰 주입 안 함. 'use cache' 함수 안에서만 사용
```

### 1.1. 사용 매트릭스

| 호출 지점 | 사용할 클라이언트 |
|---|---|
| RSC (서버 컴포넌트, 비-캐시) | `serverClient` |
| **`'use cache'` 함수 안의 서버 호출** | **`publicApiClient` 또는 명시 익명 옵션** (인증 토큰 금지) |
| Server Action | `serverClient` |
| Route Handler (`route.ts`) | `serverClient` |
| React Query `queryFn` (클라) | `browserClient` |
| React Query `mutationFn` (클라) | `browserClient` |
| 폼 `onSubmit` 핸들러 | **Server Action 경유** (직접 API 호출 금지) |
| 녹음 chunk 업로드·외부 WebRTC 등 | 전용 클라이언트 (범위 밖) |

### 1.2. 금지 패턴

- **직접 `fetch()` 호출** (`features/*`, `components/*`, `lib/stores/*` 에서) → `lib/api/*` 경유 필수
- **한 함수 안에서 환경 분기** (`if (typeof window === 'undefined') ...`) → 파일 두 개로 분리
- `"use client"` 안에서 `serverClient` import → 서버 전용 (cookies 접근) 이라 번들링 깨짐
- `'use cache'` 함수 안에서 `serverClient` 호출 → 암묵 토큰 주입 → 사용자 간 캐시 오염

---

## 2. `_core.ts` — 공통 중립

```ts
// src/lib/api/_core.ts
import { ApiError } from '@/lib/errors/ApiError';

export const BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL ?? 'http://localhost:8080';

export type RequestOptions = {
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  body?: unknown;
  headers?: Record<string, string>;
  signal?: AbortSignal;
};

export function buildUrl(path: string, params?: Record<string, string | number | undefined>): string {
  const url = new URL(path.startsWith('http') ? path : `${BASE_URL}${path}`);
  if (params) {
    for (const [k, v] of Object.entries(params)) {
      if (v !== undefined) url.searchParams.set(k, String(v));
    }
  }
  return url.toString();
}

export async function parseResponse<T>(res: Response): Promise<T> {
  if (res.ok) {
    if (res.status === 204) return undefined as T;
    const text = await res.text();
    if (!text) return undefined as T;
    return JSON.parse(text) as T;
  }

  let body: Record<string, unknown> = {};
  try {
    const text = await res.text();
    if (text) body = JSON.parse(text);
  } catch {
    // non-JSON — body stays empty
  }

  const code = typeof body.code === 'string' ? body.code : 'UNKNOWN';
  const message = typeof body.message === 'string' ? body.message : '요청을 처리하지 못했습니다';
  const fieldErrors = isFieldErrorsShape(body.fieldErrors) ? body.fieldErrors : undefined;

  throw new ApiError({ code, status: res.status, message, fieldErrors, cause: body });
}

function isFieldErrorsShape(v: unknown): v is Record<string, string[]> {
  // 간단 가드. 상세는 피처 요구시 확장.
  return v !== null && typeof v === 'object' && !Array.isArray(v);
}
```

- `BASE_URL` 은 `NEXT_PUBLIC_API_BASE_URL` 환경변수. 미설정 시 localhost 폴백 (dev 편의).
- `parseResponse` 는 **204 / 비-JSON 안전 처리** ([#16 error-ux](../error-ux/SKILL.md) §3 참조).
- `ApiError` 생성 책임은 _core 에만.

---

## 3. `serverClient.ts` — 서버 전용, 토큰 자동 주입

```ts
// src/lib/api/serverClient.ts
import 'server-only';  // 클라 import 시 번들 에러 유도
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { buildUrl, parseResponse, type RequestOptions } from './_core';
import { refreshTokenAction } from '@/features/auth/actions';  // 서버 액션으로 refresh

async function execute<T>(path: string, opts: RequestOptions, retriedWithFreshToken = false): Promise<T> {
  const cookieStore = await cookies();
  const accessToken = cookieStore.get('access_token')?.value;

  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    ...(opts.headers ?? {}),
  };
  if (accessToken) headers.Authorization = `Bearer ${accessToken}`;

  const res = await fetch(buildUrl(path), {
    method: opts.method ?? 'GET',
    headers,
    body: opts.body !== undefined ? JSON.stringify(opts.body) : undefined,
    signal: opts.signal,
    // Next 16 cacheComponents 환경 — fetch 는 캐시 안 됨 (상위 'use cache' 가 결정)
  });

  if (res.status === 401 && !retriedWithFreshToken) {
    const refreshed = await refreshTokenAction();  // 서버에서 refresh token 쿠키로 재발급
    if (refreshed.ok) return execute<T>(path, opts, true);
    redirect('/login');  // 특수 예외 — Next 가 처리
  }

  return parseResponse<T>(res);
}

export const serverClient = {
  get: <T>(path: string, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'GET' }),
  post: <T>(path: string, body?: unknown, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'POST', body }),
  put: <T>(path: string, body?: unknown, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'PUT', body }),
  patch: <T>(path: string, body?: unknown, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'PATCH', body }),
  delete: <T>(path: string, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'DELETE' }),
};
```

**핵심 규칙**:
- 최상단 `import 'server-only'` — 클라 번들링 시 빌드 에러로 **방지 가드**.
- 401 refresh **1회만** 재시도 — 무한 루프 방지.
- Refresh 실패 시 `redirect('/login')` — **특수 예외** 라서 호출자 catch 가 이를 재throw 해야 함 ([#16 §7.2](../error-ux/SKILL.md)).
- Access Token 은 HttpOnly 쿠키에서 (XSS 방어). 상세는 #19 `web-security`.
- `fetch` 에 캐시 옵션 명시하지 않음 — `cacheComponents: true` 하에서 상위 `'use cache'` 가 결정.

---

## 4. `browserClient.ts` — 클라 전용

```ts
// src/lib/api/browserClient.ts
import { getAccessToken, setAccessToken, logout } from '@/features/auth/AuthContext';
import { refreshTokenViaAction } from '@/features/auth/refresh';
import { buildUrl, parseResponse, type RequestOptions } from './_core';

async function execute<T>(path: string, opts: RequestOptions, retriedWithFreshToken = false): Promise<T> {
  const token = getAccessToken();
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    ...(opts.headers ?? {}),
  };
  if (token) headers.Authorization = `Bearer ${token}`;

  const res = await fetch(buildUrl(path), {
    method: opts.method ?? 'GET',
    headers,
    body: opts.body !== undefined ? JSON.stringify(opts.body) : undefined,
    signal: opts.signal,
  });

  if (res.status === 401 && !retriedWithFreshToken) {
    try {
      const newToken = await refreshTokenViaAction();  // Server Action 호출 — 쿠키 접근은 서버에서만
      setAccessToken(newToken);
      return execute<T>(path, opts, true);
    } catch {
      logout();
      // 호출자(RQ onError 또는 호출부)가 UX 처리. 여기선 던짐.
      throw new Error('SESSION_EXPIRED');
    }
  }

  return parseResponse<T>(res);
}

export const browserClient = {
  get: <T>(path: string, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'GET' }),
  post: <T>(path: string, body?: unknown, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'POST', body }),
  put: <T>(path: string, body?: unknown, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'PUT', body }),
  patch: <T>(path: string, body?: unknown, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'PATCH', body }),
  delete: <T>(path: string, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'DELETE' }),
};
```

**핵심 규칙**:
- Access Token 은 **메모리/`AuthContext`** 에서. `localStorage` 금지 (XSS 리스크, #19).
- Refresh 는 **Server Action 호출** 로 위임 (HttpOnly refresh cookie 접근은 서버에서만).
- 401 refresh 1회만, 실패 시 `logout()` + throw.
- `AbortSignal` 통해 **요청 취소** 지원 (RQ 가 unmount 시 자동 전달).

---

## 5. `publicApiClient.ts` — `'use cache'` 안에서 쓰는 익명 클라이언트

```ts
// src/lib/api/publicApiClient.ts
import 'server-only';
import { buildUrl, parseResponse, type RequestOptions } from './_core';

async function execute<T>(path: string, opts: RequestOptions): Promise<T> {
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    ...(opts.headers ?? {}),
  };
  // 토큰 주입 없음 — cookies/headers 접근하지 않음

  const res = await fetch(buildUrl(path), {
    method: opts.method ?? 'GET',
    headers,
    body: opts.body !== undefined ? JSON.stringify(opts.body) : undefined,
    signal: opts.signal,
  });

  return parseResponse<T>(res);
}

export const publicApiClient = {
  get: <T>(path: string, opts?: RequestOptions) => execute<T>(path, { ...opts, method: 'GET' }),
  // 공용 엔드포인트만 — mutation 메서드는 필요 생기면 추가 (지금은 GET 우선)
};
```

**핵심 규칙**:
- `cookies()`/`headers()` 접근 **없음** — `'use cache'` 함수 안에서 사용 안전.
- **공용 엔드포인트 전용**. 백엔드가 anonymous OK 또는 API 키 인증 (헤더 명시) 한 엔드포인트만.
- GET 우선. mutation 은 Server Action 경유라 publicApiClient 에 거의 필요 없음.

---

## 6. `'use cache'` + `cacheLife` + `cacheTag` 사용 패턴

> **중요 (#33 보강)**: 캐시 키는 **함수 인수** 가 결정. `cacheTag` 는 **무효화 라벨** 이지 격리 수단이 아니다.

### 6.1. 캐시 범위 결정 트리

```
새 서버 데이터 로드 코드 작성:

1. 사용자별 필터링·개인 데이터인가?
   YES → 'use cache' 쓸거면 userId 를 함수 인수로 받고 publicApiClient 사용
         (serverClient 금지)
         OR 캐시 안 함 (serverClient 로 직접 호출)
   NO  → 2

2. 변화 빈도는?
   매 요청마다 바뀌면 → 캐시 안 함 (serverClient)
   분 단위 → 'use cache' + cacheLife('minutes')
   시간 단위 → 'use cache' + cacheLife('hours')
   일 단위 → 'use cache' + cacheLife('days')
   거의 안 바뀜 → 'use cache' + cacheLife('max')

3. 명시적 무효화 필요?
   YES → cacheTag('<domain>:<key>') 추가 → 무효화 시 revalidateTag(...)
   NO  → cacheTag 생략 가능 (expire 로만 자연 무효화)
```

### 6.2. 공용 데이터 캐시 — 패턴 A

```ts
// src/features/dashboard/queries.ts
import 'server-only';
import { cacheLife, cacheTag } from 'next/cache';
import { publicApiClient } from '@/lib/api/publicApiClient';

export async function getPublicDashboardTemplate(teamId: string) {
  'use cache';
  cacheLife('hours');
  cacheTag(`dashboard:template:${teamId}`);  // 무효화 라벨
  return publicApiClient.get<DashboardTemplate>(`/teams/${teamId}/dashboard-template`);
}
```

- `teamId` 가 **함수 인수** → 팀별 캐시 키 자동 격리.
- 호출자 (RSC) 가 세션에서 teamId 를 얻어 전달.

### 6.3. 사용자별 데이터 캐시 — 패턴 B

```ts
// src/features/meeting/queries.ts
import 'server-only';
import { cacheLife, cacheTag } from 'next/cache';
import { publicApiClient } from '@/lib/api/publicApiClient';

export async function getUserMeetings(userId: string) {
  'use cache';
  cacheLife('minutes');
  cacheTag(`meetings:user:${userId}`);  // 라벨
  // 캐시 키는 userId 인수가 담당 (cacheTag 는 무효화 전용)
  return publicApiClient.get<Meeting[]>(`/users/${userId}/meetings`);
}

// 호출자 (RSC)
export default async function Page() {
  const userId = await getUserIdFromCookies();  // 세션 조회 (캐시 밖)
  const meetings = await getUserMeetings(userId);
  return <MeetingList items={meetings} />;
}
```

### 6.4. 캐시 안 함 — 민감 사용자별 데이터

```ts
// 'use cache' 안 씀 — 요청마다 serverClient 로 최신 조회
export async function getMyActiveSession() {
  return serverClient.get<Session>('/me/active-session');
}
```

### 6.5. 무효화

```ts
// Server Action 내 쓰기 후
import { revalidateTag } from 'next/cache';

export async function createMeetingAction(input: unknown) {
  // ... 성공 처리 후
  revalidateTag(`meetings:user:${userId}`);
  revalidateTag(`dashboard:team:${teamId}`);
  return { ok: true, data: result };
}
```

### 6.6. 금지 패턴

- `'use cache'` 함수 안에서 `cookies()`/`headers()`/`draftMode()` 호출
- `'use cache'` 함수 안에서 `serverClient.*` 호출 (암묵 토큰 주입)
- `cacheTag` 를 **캐시 키 격리 수단** 으로 쓰는 서술 — 격리는 함수 인수가 담당
- 사용자별 캐시 함수에 **인수 없이** `cookies()` 로 userId 를 얻는 패턴 — 캐시 키 격리 깨짐
- 캐시 가능 데이터인데 `cacheLife` 없이 `'use cache'` 선언 (기본 profile 적용되지만 명시가 유지보수성 좋음)

---

## 7. React Query 패턴

### 7.1. Query Key 네이밍

```ts
// src/features/meeting/queryKeys.ts
export const meetingKeys = {
  all: ['meeting'] as const,
  lists: () => [...meetingKeys.all, 'list'] as const,
  list: (filters: ListFilters) => [...meetingKeys.lists(), filters] as const,
  details: () => [...meetingKeys.all, 'detail'] as const,
  detail: (id: string) => [...meetingKeys.details(), id] as const,
};
```

- **피처별 네임스페이스**, 계층 구조로 **부분 invalidation** 쉽게.
- 서버 캐시의 `cacheTag` 와 **다른 이름 공간** — 혼동 금지. (RQ 키는 클라 캐시 식별자, cacheTag 는 서버 캐시 라벨.)

### 7.2. Query 함수

```ts
// src/features/meeting/queries.ts (클라 측)
import { queryOptions } from '@tanstack/react-query';
import { browserClient } from '@/lib/api/browserClient';
import { meetingKeys } from './queryKeys';

export function meetingDetailQuery(id: string) {
  return queryOptions({
    queryKey: meetingKeys.detail(id),
    queryFn: ({ signal }) => browserClient.get<Meeting>(`/meetings/${id}`, { signal }),
  });
}

// 사용
const { data } = useQuery(meetingDetailQuery(id));
```

- **`queryOptions`** 로 공유 — 호출부가 `useQuery(query)` / `useSuspenseQuery(query)` / `queryClient.prefetchQuery(query)` 로 재사용.
- **`signal` 전달 필수** — 컴포넌트 unmount 시 자동 취소.

### 7.3. Mutation 패턴

```ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { browserClient } from '@/lib/api/browserClient';
import { meetingKeys } from './queryKeys';

export function useUpdateMeeting(id: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (patch: Partial<Meeting>) =>
      browserClient.patch<Meeting>(`/meetings/${id}`, patch),
    onSuccess: (updated) => {
      qc.setQueryData(meetingKeys.detail(id), updated);  // 상세는 즉시 교체
      qc.invalidateQueries({ queryKey: meetingKeys.lists() });  // 리스트는 invalidate
    },
    // 전역 onError 가 토스트 (#5 결정)
  });
}
```

- **낙관 업데이트** 필요하면 `onMutate` 로 `setQueryData` 하고, `onError` 에서 롤백.
- 전역 토스트와 **중복 금지** — 특수 UX 가 필요한 뮤테이션만 `onError` override.

### 7.4. 서버 → 클라 Hydration

```tsx
// 서버에서 prefetch
import { QueryClient, dehydrate, HydrationBoundary } from '@tanstack/react-query';
import { meetingDetailQuery } from '@/features/meeting/queries';

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const qc = new QueryClient();                   // 요청별 격리 (서버)
  await qc.prefetchQuery(meetingDetailQuery(id));  // browserClient 가 아니라 서버 queryFn 이 필요하면 별도 쿼리로
  return (
    <HydrationBoundary state={dehydrate(qc)}>
      <MeetingDetail id={id} />
    </HydrationBoundary>
  );
}
```

- **주의**: `prefetchQuery` 에 쓰는 `queryFn` 은 서버 전용. 보통 **서버용 queries 파일** (`features/<domain>/serverQueries.ts`) 과 **클라용 queries 파일** 을 분리. 또는 queryFn 을 환경 비의존 (`publicApiClient`) 으로 작성 가능하면 공용.
- 모든 페이지에 강제 적용 금지 — "서버에서 1차 랜더 → 클라에서 조작" 흐름이 명확할 때만.

---

## 8. 페이지네이션

### 8.1. 커서 기반 (권장)

```ts
export function meetingListQuery(filters: ListFilters) {
  return infiniteQueryOptions({
    queryKey: meetingKeys.list(filters),
    queryFn: ({ pageParam, signal }) =>
      browserClient.get<MeetingPage>(
        buildUrl('/meetings', { ...filters, cursor: pageParam }),
        { signal },
      ),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (last) => last.nextCursor ?? undefined,
  });
}
```

- 백엔드가 `{ items, nextCursor }` shape 반환 가정.
- offset/limit 방식보다 **중복/누락 없음**.

### 8.2. 금지

- 수동으로 `page = 1, 2, 3...` 상태를 Zustand 에 저장 → RQ 가 담당 (`useInfiniteQuery`)
- 서버가 totalCount 를 매번 full-scan 하는 API 를 호출 — 커서로 교체 요청

---

## 9. 재시도·백오프·취소

### 9.1. 재시도 정책

- **클라 서버 네트워크 오류**: fetch wrapper 가 재시도하지 않음 (`retry` 는 호출자 선택).
- **401 refresh**: wrapper 가 **1회만** 재시도.
- **RQ queries**: `retry: 1` ([#5](../../../docs/axis-decisions/error-ux.md) §E 기본 옵션).
- **RQ mutations**: `retry: 0` (사용자 action 이라 자동 재시도 위험).

### 9.2. AbortSignal

- 모든 wrapper 가 `opts.signal` 받아 `fetch` 에 전달.
- RQ 가 `queryFn({ signal })` 로 자동 전달 — 컴포넌트 unmount 시 자동 취소.
- 수동 호출 (예: Server Action 안의 serverClient) 은 signal 전달 생략 가능 (대부분 끝까지 기다림).

### 9.3. 금지

- 재시도 로직을 **각 호출자가 재구현** → wrapper 또는 RQ 에만.
- `AbortController` 를 wrapper 안에서 만들어 호출자가 취소 못 하게 숨김 → signal 은 **호출자가 전달**.

---

## 10. 체크리스트 — 데이터 페칭 코드 작성·리뷰 시

- [ ] 직접 `fetch()` 호출 없이 `serverClient` / `browserClient` / `publicApiClient` 경유인가?
- [ ] 호출 지점의 **환경** (서버/클라) 과 import 한 클라이언트가 **일치**하는가? (serverClient 를 `"use client"` 에서 import 금지)
- [ ] **`'use cache'` 안에서 `cookies()`/`headers()`/`serverClient` 사용 없는가**? publicApiClient 또는 명시 익명 호출인가?
- [ ] 사용자별 캐시가 **`userId` 를 함수 인수로** 받아 캐시 키 자동 격리하는가? `cacheTag` 는 **무효화용**으로만 쓰는가?
- [ ] `cacheLife` 가 데이터 성격에 맞게 설정됐는가 (`seconds`/`minutes`/`hours`/`days`/`weeks`/`max`)?
- [ ] 쓰기 Server Action 이 **관련 cacheTag 를 `revalidateTag`** 로 무효화하는가?
- [ ] 401 refresh 가 wrapper 안에서 **1회만** 재시도하는가 (호출자 중복 금지)?
- [ ] `parseResponse` 가 **204 + 비-JSON 안전** 처리인가?
- [ ] RQ queryKey 가 **피처 네임스페이스** 로 계층 구조인가? `cacheTag` 와 혼동 금지?
- [ ] RQ `queryFn({ signal })` 로 **AbortSignal 전달** 하는가?
- [ ] 페이지네이션이 **커서 기반** 인가? Zustand 로 page 상태 저장 금지?
- [ ] 뮤테이션이 전역 `onError` 와 **중복 토스트** 없이 override 하는가?
- [ ] `prefetchQuery` + `HydrationBoundary` 는 "서버 1차 랜더 → 클라 조작" 흐름일 때만 쓰는가?

---

## 11. 참고

- 결정 축: [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md), [#5 error-ux](../../../docs/axis-decisions/error-ux.md), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md)
- 관련 스킬: [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md) §2.1 (캐시 모델), [`app-router-layering`](../app-router-layering/SKILL.md) §4 (데이터 페칭 레이어·`'use cache'`+인증), [`error-ux`](../error-ux/SKILL.md) §3 (parseResponse) · §7 (Server Action throw), [`ts-style`](../ts-style/SKILL.md)
- Next 16 공개 API 선언: `node_modules/next/cache.d.ts` (`cacheLife`/`cacheTag`/`revalidateTag`/`io`/`navigation`/`prefetch`)
