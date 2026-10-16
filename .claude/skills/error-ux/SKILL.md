---
name: error-ux
description: flowmeet-web 의 에러 처리 가드 — 경계·`ApiError`·매핑·토스트/인라인/풀페이지·RQ 연동·Server Action throw·reportError 를 코드로 적용할 때 로드. 에러 바운더리, Server Action 오류 처리, API 에러 매핑 코드를 쓰거나 리뷰할 때 호출.
---

# Error UX — flowmeet-web

[#5 error-ux](../../../docs/axis-decisions/error-ux.md) 결정 + [#33 리뷰 보강](../../../docs/axis-decisions/error-ux.md#f-보강-throw-정책--오류의-성격에-따라-다르다) 을 **코드로 적용할 때** 쓰는 가드. "경계는 어디에, 토스트는 언제, Server Action throw 는 어떻게" 를 구체 코드 수준으로 catch.

> 결정 축이 "무엇을" 정했다면 이 스킬은 "그래서 코드를 어떻게 쓰나" 를 다룬다. 세부 정책의 소스는 결정 문서 유지, 이 스킬은 **구현 체크리스트 + 금지 패턴**.

---

## 1. 디렉터리 (고정)

```
src/lib/errors/
├─ ApiError.ts          # class ApiError { code, status, fieldErrors?, cause? }
├─ codes.ts             # type ApiErrorCode union + toApiErrorCode(string)
├─ messages.ts          # codeToMessage(code, fallback): string  (단일 소스)
├─ handle.ts            # handleApiError(err, { onToast, onInline, onRedirect })
├─ reportError.ts       # 구조화 JSON console.error (외부 벤더 미도입 #8)
├─ maskSensitive.ts     # 민감 키 재귀 치환 (상세 #19)
├─ ErrorBoundary.tsx    # react-error-boundary 래퍼 + reportError 훅
└─ ActionResult.ts      # type ActionResult<T> (ok discriminated union)
```

- `src/lib/validation/errors.ts` (폼 전용 매핑) 는 **`codeToMessage` 를 import 재사용**. 자체 메시지 테이블 금지 (#4).

---

## 2. 경계 레이어링 — 코드 패턴

### 2.1. `src/app/global-error.tsx`

```tsx
'use client';  // 필수 (Next 16)
import { useEffect } from 'react';
import { reportError } from '@/lib/errors/reportError';

export default function GlobalError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  // 렌더 본문에서 호출 금지 — React 재랜더/Strict Mode 로 중복 보고
  useEffect(() => {
    reportError(error, { source: 'global-error', extra: { digest: error.digest } });
  }, [error]);

  // global-error 는 루트 레이아웃을 대체 — 자체 <html>/<body> 필수 (#13 §2.5)
  return (
    <html lang="ko">
      <body>
        <div>
          <h1>오류가 발생했습니다</h1>
          <button onClick={() => reset()}>다시 시도</button>
        </div>
      </body>
    </html>
  );
}
```

**금지**:
- `"use client"` 누락 → Next 가 거부
- `<html>`/`<body>` 생략 → 루트 레이아웃이 사라져 깨짐
- **렌더 본문에서 `reportError` 직접 호출** → 재랜더 중복 보고. `useEffect(..., [error])` 로 이동
- `reportError` 호출 누락 → 관측 공백

### 2.2. 섹션 `src/app/(group)/error.tsx`

```tsx
'use client';  // 필수
import { useEffect } from 'react';
import Link from 'next/link';
import { reportError } from '@/lib/errors/reportError';

export default function Error({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => {
    reportError(error, { source: 'section-error:(dashboard)', extra: { digest: error.digest } });
  }, [error]);

  return (
    <div role="alert">
      <h2>이 섹션을 불러오지 못했습니다</h2>
      <div>
        <button onClick={() => reset()}>다시 시도</button>
        <Link href="/">처음으로</Link>
      </div>
    </div>
  );
}
```

- 섹션 폴백 UI 는 **재시도 버튼 + "처음으로"** 를 기본 제공 (#5 결정). 둘 다 포함.
- 섹션 안에서만 교체 — 상위 레이아웃·네비게이션은 유지.
- `reportError` 는 **`useEffect` 안에서** (렌더 본문 호출 금지).

### 2.3. 컴포넌트 `ErrorBoundary` (선별적)

```tsx
// src/lib/errors/ErrorBoundary.tsx
'use client';
import { ErrorBoundary as ReactErrorBoundary, type FallbackProps } from 'react-error-boundary';
import { reportError } from './reportError';

type Props = {
  children: React.ReactNode;
  Fallback: React.ComponentType<FallbackProps>;  // 전체 FallbackProps 전달 (componentStack 유지)
  source: string;
};

export function ErrorBoundary({ children, Fallback, source }: Props) {
  return (
    <ReactErrorBoundary
      FallbackComponent={Fallback}
      onError={(error, info) => reportError(error, { source, extra: { componentStack: info.componentStack } })}
    >
      {children}
    </ReactErrorBoundary>
  );
}
```

**사용 원칙**:
- "일부가 터져도 나머지는 살아야 하는 위젯" 에만. 대시보드 카드 1개 실패 시 나머지 카드 유지 등.
- 모든 컴포넌트에 두는 것 금지.
- `source` 는 reportError 추적용 라벨 (예: `'dashboard:delayed-tasks-card'`).

---

## 3. `ApiError` 생성 — fetch wrapper 쪽

```ts
// src/lib/api/_core.ts (예시)
export async function parseResponse<T>(res: Response): Promise<T> {
  if (res.ok) {
    if (res.status === 204) return undefined as T;  // No Content — body 없음
    const text = await res.text();
    if (!text) return undefined as T;
    return JSON.parse(text) as T;
  }

  // 오류 응답 바디 — JSON 이 아닐 수도 있음, 안전 파싱
  let body: Record<string, unknown> = {};
  try {
    const text = await res.text();
    if (text) body = JSON.parse(text);
  } catch {
    // non-JSON 응답은 그대로 UNKNOWN 처리
  }

  const code = typeof body.code === 'string' ? body.code : 'UNKNOWN';
  const message = typeof body.message === 'string' ? body.message : '요청을 처리하지 못했습니다';
  const fieldErrors = isFieldErrorsShape(body.fieldErrors) ? body.fieldErrors : undefined;

  throw new ApiError({ code, status: res.status, message, fieldErrors, cause: body });
}
```

- **`204 No Content` 처리 필수** — `res.json()` 직접 호출하면 실패.
- 오류 바디가 **JSON 이 아니거나 `code`/`message` 가 문자열이 아닐 수** 있음. 런타임 가드 통과 후 `ApiError` 생성.
- `code` 가 서버에서 안 왔거나 비정상이면 `'UNKNOWN'`.
- `message` 는 서버 원문 저장. **사용자에게 보여줄 때는 반드시 `codeToMessage(err.code)` 통과** — 서버 메시지가 영어거나 민감정보 섞여 있을 수 있음.

---

## 4. `codeToMessage` 는 **단일 매핑 소스**

```ts
// src/lib/errors/messages.ts
import { toApiErrorCode, type ApiErrorCode } from './codes';

const MESSAGES: Record<ApiErrorCode, string> = {
  AUTH_INVALID: '로그인 정보가 올바르지 않습니다.',
  AUTH_SESSION_EXPIRED: '로그인이 만료되었습니다. 다시 로그인해 주세요.',
  FORBIDDEN: '접근 권한이 없습니다.',
  MEETING_NOT_FOUND: '존재하지 않는 회의입니다.',
  UNKNOWN: '오류가 발생했습니다. 잠시 후 다시 시도해 주세요.',
};

export function codeToMessage(code: string, fallback = MESSAGES.UNKNOWN): string {
  const key = toApiErrorCode(code);
  return MESSAGES[key] ?? fallback;
}
```

**금지**:
- 폼 쪽에서 `codeToMessage` 를 **다시 구현** (`src/lib/validation/errors.ts` 는 import 재사용)
- 피처 컴포넌트에서 `err.message` 를 **그대로 사용** (서버 원문 노출)
- 컴포넌트별 하드코딩 메시지 ("회의 없음" 같은 비즈니스 메시지를 UI 안에 묻는 것)

---

## 5. 토스트/인라인/풀페이지 선택

| 상황 | 호출 패턴 |
|---|---|
| 뮤테이션 성공 | `toast.success('저장했습니다')` |
| 뮤테이션 실패 | 전역 RQ `onError` 가 처리 — 직접 토스트 호출 **지양** |
| 폼 필드 에러 | RHF `setError(field, { message })` (#4) |
| 폼 비필드 에러 | `toast.error(codeToMessage(err.code))` 또는 인라인 배너 |
| 섹션 데이터 로드 실패 | 섹션 `error.tsx` 가 자동 |
| 위젯 렌더 실패 | `<ErrorBoundary source="...">` 가 자동 |
| 세션 만료 | fetch wrapper 또는 Server Action 이 `redirect('/login')` |

**금지**:
- `toast.error('저장 실패')` 식의 **하드코딩 메시지** — `codeToMessage` 통과
- 동일 에러에 토스트 + 인라인 **중복 표시**
- 토스트 severity 를 직접 색상으로 구현 (`uiStore.toast.error/warning/info/success` API 만)

---

## 6. React Query 연동

### 6.1. QueryClient 기본 설정 (요약, 상세 #5·#2)

```ts
// src/app/providers.tsx (일부)
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';
import { reportError } from '@/lib/errors/reportError';
import { codeToMessage } from '@/lib/errors/messages';
import { ApiError } from '@/lib/errors/ApiError';
import { toast } from '@/lib/stores/uiStore';

export function QueryClientProviderWrapper({ children }: { children: React.ReactNode }) {
  // 브라우저: 안정 단일 인스턴스 (매 랜더마다 새로 만들지 않음)
  const [client] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        throwOnError: true,                   // 섹션 ErrorBoundary 로 bubbling (#5)
        retry: 1,
        staleTime: 30_000,
        gcTime: 5 * 60 * 1000,
        refetchOnWindowFocus: false,
      },
      mutations: {
        throwOnError: false,
        onError: (err) => {
          reportError(err, { source: 'rq-mutation' });
          const code = err instanceof ApiError ? err.code : 'UNKNOWN';
          toast.error(codeToMessage(code));
        },
      },
    },
  }));
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}
```

### 6.2. 쿼리 특수 처리

```ts
// 특정 쿼리가 inline 에러 처리를 원할 때 — 의식적 opt-out
const { data, error } = useQuery({
  queryKey: ['meeting', id],
  queryFn: ({ signal }) => browserClient.get(`/meetings/${id}`, { signal }),
  throwOnError: false,  // 명시적 opt-out
});

if (error) {
  // inline 처리
}
```

### 6.3. 뮤테이션 override

```ts
// 낙관 업데이트 롤백 등 전역 처리와 다를 때
const mutation = useMutation({
  mutationFn: toggleFavorite,
  onMutate: async (newValue) => { /* 낙관 반영 */ },
  onError: (err, newValue, context) => {
    // 1) 전역 toast 는 이미 발생 — 중복 호출 금지
    // 2) 낙관 롤백만 수행
    rollback(context);
  },
});
```

---

## 7. Server Action 오류 처리 — **throw 정책 분기**

> **중요**: 모든 예외를 `ActionResult` 로 조용히 변환하면 **관측 누락 + 에러 바운더리 bypass**. 오류 성격에 따라 처리를 분기한다 (#5 §F-보강).

### 7.1. 분기 테이블

| 오류 성격 | 처리 |
|---|---|
| 예상 가능 (zod 실패·중복·권한 거부·비즈니스 룰) | `ActionResult`/`FormActionResult` 반환 |
| 세션 만료 (401) | `redirect('/login')` — 반환 shape 이전에 |
| 예상 못한 예외 (DB 다운·외부 API 실패·버그) | `reportError` 후 throw |

### 7.2. 템플릿

```ts
'use server';
import { isRedirectError } from 'next/dist/client/components/redirect';  // 또는 next 공개 API 로 대체 (버전 확인)
import { createMeetingSchema } from '@/lib/validation/schemas/meeting';
import { serverClient } from '@/lib/api/serverClient';
import { createMeetingLogic } from './createMeetingLogic';  // pure helper (#6)
import { ApiError } from '@/lib/errors/ApiError';
import { codeToMessage } from '@/lib/errors/messages';
import { reportError } from '@/lib/errors/reportError';
import type { FormActionResult } from '@/lib/validation/formAction';
import { mapZodErrorToFormResult } from '@/lib/validation/errors';
import { isDomainErrorCode } from '@/lib/errors/codes';

export async function createMeetingAction(input: unknown): Promise<FormActionResult<{ id: string }>> {
  // 1) zod 선검증 — 예상 가능
  const parsed = createMeetingSchema.safeParse(input);
  if (!parsed.success) return mapZodErrorToFormResult(parsed.error);

  try {
    // 2) pure helper 호출. 세션 만료 (401) 는 serverClient 내부에서 `redirect('/login')` 호출.
    //    redirect() 는 특수 예외를 던지므로 아래 catch 가 반드시 재throw 해야 함.
    return await createMeetingLogic(parsed.data, serverClient);
  } catch (err) {
    // 3) Next 의 redirect/notFound 특수 예외는 **반드시 재throw** — Next 가 처리해야 함
    if (isRedirectError(err) || isNotFoundError(err)) {
      throw err;
    }
    // 4) 예상 가능한 도메인 오류 (예: 중복, 권한 거부) → 반환
    if (err instanceof ApiError && isDomainErrorCode(err.code)) {
      return { ok: false, code: err.code, message: codeToMessage(err.code) };
    }
    // 5) 예상 못한 예외 → 로깅 후 throw
    reportError(err, { source: 'server-action:createMeeting' });
    throw err;
  }
}
```

> **중요**: `redirect()` / `notFound()` 는 Next 가 내부적으로 **특수 예외** (`NEXT_REDIRECT`, `NEXT_NOT_FOUND`) 를 throw 하는 방식으로 동작한다. 일반 `catch` 가 이들을 잡아 `ActionResult` 로 변환하거나 `reportError` 로 로깅하면 **리다이렉트/404 처리가 실패**한다. 반드시 **먼저 검사해 재throw**. (`isRedirectError`/`isNotFoundError` 는 Next 공개 API 를 사용 — 버전별 import 경로는 `next/dist/*` 가 아닐 수 있으므로 설치본 확인.)

### 7.3. 도메인 오류 판별 보수적으로

- `err instanceof ApiError` 체크는 **같은 모듈 그래프 안에서 생성된 ApiError 만** 매칭. serverClient 가 다른 번들러 context 를 거치거나 wrap 하면 miss 가능.
- 대안:
  - `err && typeof err === 'object' && 'code' in err` 형태로 **duck-typing** 보강
  - 또는 `err.name === 'ApiError'` 체크
- 통합 테스트로 "serverClient 의 모든 오류가 ApiError 로 호출부에 도달하는지" 보장.

### 7.4. 호출부 (클라)

```tsx
// RHF 폼 제출 (#4)
const onSubmit = form.handleSubmit(async (values) => {
  try {
    const res = await createMeetingAction(values);
    if (!res.ok) {
      // 예상 가능 — fieldErrors / formError 처리
      if (res.fieldErrors) {
        Object.entries(res.fieldErrors).forEach(([f, msgs]) =>
          form.setError(f as keyof FormValues, { message: msgs.join(', ') })
        );
      } else {
        toast.error(res.message);
      }
      return;
    }
    // 성공
  } catch (err) {
    // 예상 못한 예외 — Server Action 이 throw 한 것. 호출 경로에 따라 처리 다름.
    // 폼 onSubmit 안에서는 react-error-boundary 로 자동 bubble 안 됨 → 명시 처리.
    reportError(err, { source: 'form-submit:createMeeting' });
    toast.error(codeToMessage('UNKNOWN'));
  }
});
```

> **주**: Server Action throw 가 자동으로 섹션 `error.tsx` 로 가지 않는 호출 경로 (클라 핸들러 등) 는 **호출부가 명시 catch** 하거나 **react-error-boundary** 로 감싸야 한다 (#5 §F-주). 통합 테스트로 확인.

### 7.5. 금지 패턴

- **모든 예외를 `ActionResult` 로 변환** → 관측 공백, 에러 바운더리 bypass
- **`redirect()`/`notFound()` 특수 예외를 `catch` 가 삼킴** → 리다이렉트/404 실패. 반드시 재throw.
- 예상 가능 오류를 throw → UX 거침, 사용자가 "오류가 발생했습니다" 만 보게 됨
- `reportError` 호출 없이 throw → 관측 공백
- Server Action 안에서 직접 `fetch` → `serverClient` 경유 (#2)
- Server Action 이 거대한 비즈니스 로직 → pure helper 분리 (#6)

### 7.6. 중복 로깅 방지 — 레이어별 책임

하나의 장애가 Server Action + 클라 호출부 + ErrorBoundary 세 곳에서 각각 `reportError` 호출될 수 있다. **최종 관측 지점을 하나로** 정한다.

| 발생 | 1차 로깅 책임 | 재로깅 금지 |
|---|---|---|
| Server Action 안의 예상 못한 예외 | Server Action (`source: 'server-action:xxx'`) | 호출부 catch, ErrorBoundary 는 **같은 에러면 중복 금지** |
| 클라 호출부가 자체 발생시킨 에러 (RHF 등) | 호출부 (`source: 'form-submit:xxx'`) | - |
| ErrorBoundary 가 받은 렌더 에러 | ErrorBoundary (`source: 'error-boundary:xxx'`) | - |

- **룰**: 로깅은 **에러가 발생한 레이어** 에서 1회만. 상위 레이어가 받으면 **UX 처리만** (토스트 등). 재로깅 금지.
- 호출부 catch 가 반드시 로깅해야 할 경우 (예: Server Action 이 로깅 안 했을 가능성) 라면 `source` 를 **`<layer>:catch` 로 접미** 해 중복인지 구분 가능하게.

---

## 8. `reportError` 호출 규약

```ts
reportError(err, { source: '<layer>:<subject>', extra: { ... }, userId: '...' });
```

- `source` 네이밍: `<layer>:<subject>` 형식 — 예:
  - `rsc-boundary:(dashboard)`
  - `rq-mutation:createMeeting`
  - `server-action:toggleFavorite`
  - `form-submit:createMeeting`
  - `error-boundary:dashboard-card:delayed-tasks`
- `extra` 는 **디버그 context** — 민감정보 넣지 말 것 (`maskSensitive` 가 1차 안전망이지만 애당초 안 넣는 게 베스트).
- `userId` 는 **opaque id 만** — 이메일·이름·전화 금지 (#8).

**금지**:
- `console.error` 를 직접 호출 (`reportError` 경유)
- `reportError` 안에서 다시 throw (로깅은 side-effect 로만)
- extra 에 `password`/`token`/`cookies` 등 민감 키 넣기

---

## 9. 체크리스트 — 에러 처리 코드 작성·리뷰 시

- [ ] `global-error.tsx` 가 `"use client"` + 자체 `<html>`/`<body>` + **`useEffect` 안의** `reportError` 호출을 포함하는가?
- [ ] 섹션 `error.tsx` 가 `"use client"` + **`useEffect` 안의** `reportError` + **재시도 + "처음으로"** 둘 다 포함하는가?
- [ ] `ErrorBoundary` 가 "일부 터져도 나머지 살아야" 하는 지점에만 쓰이는가 (남용 금지)? Fallback 이 `componentStack` 받을 수 있는 구조인가?
- [ ] `ApiError` 를 던질 때 `code`·`status`·`message`·`fieldErrors?` 가 올바르게 채워지는가?
- [ ] `parseResponse` 가 `204 No Content` + 비-JSON 오류 바디를 안전하게 처리하는가?
- [ ] 사용자 메시지는 **`codeToMessage` 통과** 인가? `err.message` 를 UI 에 그대로 노출하지 않는가?
- [ ] 토스트 메시지가 하드코딩이 아니라 `codeToMessage` 통과인가?
- [ ] RQ `defaultOptions.queries.throwOnError: true` + `mutations.onError` 전역 토스트가 설정됐는가?
- [ ] 특정 쿼리가 inline 처리면 명시적 `throwOnError: false` 로 opt-out 했는가?
- [ ] Server Action 이 **예상 가능 오류** 는 `ActionResult` 반환, **예상 못한 예외** 는 `reportError` + throw 하는가?
- [ ] Server Action 의 `catch` 가 **`redirect()`/`notFound()` 특수 예외를 먼저 재throw** 하는가?
- [ ] Server Action 호출부 (특히 클라 핸들러) 가 throw 경로를 명시 처리하거나 ErrorBoundary 로 감싸는가?
- [ ] `reportError` 의 `source` 가 `<layer>:<subject>` 형식인가? `extra` 에 민감정보가 없는가?
- [ ] **중복 로깅 금지** — 상위 레이어가 catch 하면 **UX 처리만**, `reportError` 는 1차 레이어에서 1회?
- [ ] 폼 에러 매핑이 `codeToMessage` 를 import 재사용 (중복 테이블 없음)?

---

## 10. 참고

- 결정 축: [#5 error-ux](../../../docs/axis-decisions/error-ux.md), [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md), [#4 form-validation](../../../docs/axis-decisions/form-validation.md), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md)
- 관련 스킬: [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md) §2.5 (`error.tsx` / `global-error.tsx` 구조), [`app-router-layering`](../app-router-layering/SKILL.md) §5 (Server Action throw 정책), [`ts-style`](../ts-style/SKILL.md)
