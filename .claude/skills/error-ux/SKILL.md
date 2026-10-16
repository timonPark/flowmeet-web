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
import { reportError } from '@/lib/errors/reportError';

export default function GlobalError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  // global-error 는 루트 레이아웃을 대체 — 자체 <html>/<body> 필수 (#13 §2.5)
  reportError(error, { source: 'global-error', extra: { digest: error.digest } });
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
- `reportError` 호출 누락 → 관측 공백

### 2.2. 섹션 `src/app/(group)/error.tsx`

```tsx
'use client';  // 필수
import { reportError } from '@/lib/errors/reportError';

export default function Error({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  reportError(error, { source: 'section-error:(dashboard)', extra: { digest: error.digest } });
  return (
    <div role="alert">
      <h2>이 섹션을 불러오지 못했습니다</h2>
      <button onClick={() => reset()}>다시 시도</button>
    </div>
  );
}
```

- 섹션 폴백 UI 는 **재시도 버튼 + "처음으로"** 를 기본 제공 (#5).
- 섹션 안에서만 교체 — 상위 레이아웃·네비게이션은 유지.

### 2.3. 컴포넌트 `ErrorBoundary` (선별적)

```tsx
// src/lib/errors/ErrorBoundary.tsx
'use client';
import { ErrorBoundary as ReactErrorBoundary, type FallbackProps } from 'react-error-boundary';
import { reportError } from './reportError';

type Props = {
  children: React.ReactNode;
  fallback: (props: FallbackProps) => React.ReactNode;
  source: string;
};

export function ErrorBoundary({ children, fallback, source }: Props) {
  return (
    <ReactErrorBoundary
      FallbackComponent={({ error, resetErrorBoundary }) => fallback({ error, resetErrorBoundary })}
      onError={(error) => reportError(error, { source })}
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
  if (res.ok) return res.json();

  const body = await res.json().catch(() => ({}));
  throw new ApiError({
    code: body.code ?? 'UNKNOWN',
    status: res.status,
    message: body.message ?? '요청을 처리하지 못했습니다',
    fieldErrors: body.fieldErrors,
    cause: body,
  });
}
```

- `code` 가 서버에서 안 왔으면 `'UNKNOWN'`.
- `message` 는 서버가 준 메시지 그대로 저장. **사용자에게 보여줄 때는 반드시 `codeToMessage(err.code)` 통과** — 서버 메시지가 영어거나 민감정보 섞여 있을 수 있음.

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
    // 2) pure helper 호출 (helper 가 ActionResult 로 반환하도록)
    return await createMeetingLogic(parsed.data, serverClient);
  } catch (err) {
    // 3) 예상 가능한 도메인 오류 (예: 중복) → 반환
    if (err instanceof ApiError && isDomainErrorCode(err.code)) {
      return { ok: false, code: err.code, message: codeToMessage(err.code) };
    }
    // 4) 예상 못한 예외 → 로깅 후 throw
    reportError(err, { source: 'server-action:createMeeting' });
    throw err;
  }
}
```

### 7.3. 호출부 (클라)

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

### 7.4. 금지 패턴

- **모든 예외를 `ActionResult` 로 변환** → 관측 공백, 에러 바운더리 bypass
- 예상 가능 오류를 throw → UX 거침, 사용자가 "오류가 발생했습니다" 만 보게 됨
- `reportError` 호출 없이 throw → 관측 공백
- Server Action 안에서 직접 `fetch` → `serverClient` 경유 (#2)
- Server Action 이 거대한 비즈니스 로직 → pure helper 분리 (#6)

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

- [ ] `global-error.tsx` 가 `"use client"` + 자체 `<html>`/`<body>` + `reportError` 호출을 포함하는가?
- [ ] 섹션 `error.tsx` 가 `"use client"` + `reportError` 호출을 포함하는가?
- [ ] `ErrorBoundary` 가 "일부 터져도 나머지 살아야" 하는 지점에만 쓰이는가 (남용 금지)?
- [ ] `ApiError` 를 던질 때 `code`·`status`·`message`·`fieldErrors?` 가 올바르게 채워지는가?
- [ ] 사용자 메시지는 **`codeToMessage` 통과** 인가? `err.message` 를 UI 에 그대로 노출하지 않는가?
- [ ] 토스트 메시지가 하드코딩이 아니라 `codeToMessage` 통과인가?
- [ ] RQ `defaultOptions.queries.throwOnError: true` + `mutations.onError` 전역 토스트가 설정됐는가?
- [ ] 특정 쿼리가 inline 처리면 명시적 `throwOnError: false` 로 opt-out 했는가?
- [ ] Server Action 이 **예상 가능 오류** 는 `ActionResult` 반환, **예상 못한 예외** 는 `reportError` + throw 하는가?
- [ ] Server Action 호출부 (특히 클라 핸들러) 가 throw 경로를 명시 처리하거나 ErrorBoundary 로 감싸는가?
- [ ] `reportError` 의 `source` 가 `<layer>:<subject>` 형식인가? `extra` 에 민감정보가 없는가?
- [ ] 폼 에러 매핑이 `codeToMessage` 를 import 재사용 (중복 테이블 없음)?

---

## 10. 참고

- 결정 축: [#5 error-ux](../../../docs/axis-decisions/error-ux.md), [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md), [#4 form-validation](../../../docs/axis-decisions/form-validation.md), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md)
- 관련 스킬: [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md) §2.5 (`error.tsx` / `global-error.tsx` 구조), [`app-router-layering`](../app-router-layering/SKILL.md) §5 (Server Action throw 정책), [`ts-style`](../ts-style/SKILL.md)
