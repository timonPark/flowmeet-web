---
title: 에러 처리(error-ux) 전략
date: 2026-10-16
author: kindcorder@gmail.com
status: DECIDED
issue: "#5"
scope: flowmeet-web 에러 경계·토스트/인라인 UX·ApiErrorCode 매핑·RQ 연동
depends_on:
  - "#2 데이터 페칭/캐시 전략"
  - "#3 상태 관리 전략"
  - "#4 폼/검증 전략"
---

# 에러 처리(error-ux) 전략

[#2 데이터 페칭](./data-fetching.md) 에서 fetch wrapper 가 `ApiError` 를 던지도록, [#3 상태 관리](./state-management.md) 에서 토스트 큐가 `uiStore` 에 자리잡도록, [#4 폼/검증](./form-validation.md) 에서 폼 서버 에러는 `FormActionResult` 로 **throw 없이** 돌려주도록 결정됐다. 이 축은 **"폼 외의 모든 에러 UX"** 와 **중앙 `ApiErrorCode` → 사용자 메시지 매핑**, **RQ·Server Action·경계 간 연동 규칙** 을 확정한다.

> 이 문서는 `error-ux` 스킬([#16](https://github.com/timonPark/flowmeet-web/issues/16)) 의 입력 — 특히 §B 경계·§C 매핑·§E RQ 기본 옵션.

---

## 1. 결정 요약

| 차원 | 결정 |
|---|---|
| 경계 레이어링 | 하이브리드: `global-error.tsx` + 섹션별 `error.tsx` + 선별적 `<ErrorBoundary>` |
| ErrorBoundary 라이브러리 | **`react-error-boundary` v4** |
| ApiError 홈 | **`src/lib/errors/`** |
| ApiErrorCode | **문자열 union**, 밖의 코드는 `UNKNOWN` fallback |
| 사용자 메시지 매핑 | **`src/lib/errors/messages.ts`** 중앙 테이블 (i18n-ready 구조) |
| 폼 매핑과의 관계 | `src/lib/validation/errors.ts` (#4) 가 `codeToMessage` 를 **import 재사용** |
| RQ 쿼리 기본 | **`throwOnError: true`** → 섹션 ErrorBoundary 로 bubbling |
| RQ 뮤테이션 기본 | **전역 `onError` → 토스트** (`uiStore.toast.error`) |
| 비-폼 Server Action | **`ActionResult<T>`** (폼의 `FormActionResult` 와 shape 통일) |
| 로깅 훅 | **`reportError(err, context)`** 추상 — Sentry 교체는 #8 |

**원칙**: *경계는 "실패 시 폴백이 더 유용한 지점" 에만 둔다. 토스트는 비동기 결과 알림, 인라인은 사용자 액션에 묶인 문맥, 풀페이지는 섹션/루트 복구 실패.*

---

## 2. 결정 상세

### A. 에러 범주 지도

| 범주 | 발생 지점 | 처리 계층 |
|---|---|---|
| RSC/클라 **렌더 에러** | 컴포넌트 트리 throw | `error.tsx` / `ErrorBoundary` |
| **ApiError** (fetch wrapper throw) | `serverClient` / `browserClient` | 호출자(RQ / Server Action / RSC) |
| **RQ query 에러** | `useQuery` 등 | `throwOnError: true` → 섹션 ErrorBoundary |
| **RQ mutation 에러** | `useMutation` | 전역 `onError` → 토스트 |
| **Server Action — 폼** | 폼 제출 | `FormActionResult` (#4) |
| **Server Action — 비-폼** | 토글·간단 CRUD | `ActionResult<T>` (shape 통일) |
| **401 세션 만료** | fetch wrapper | `redirect('/login')` (#2) |
| **403 forbidden** | fetch wrapper | 호출자 context — 가드 리다이렉트 or 토스트 |
| **네트워크·500+** | fetch wrapper | 토스트 (fallback 메시지) + `reportError` |
| **예상 못한 에러 (uncaught)** | 어디서든 | `global-error.tsx` + `reportError` |

### B. 경계 레이어링

**(A3) 하이브리드**:

- **`src/app/global-error.tsx`** — 최종 안전망. 루트 레이아웃까지 터질 때. 미니멀 UI + `reportError` + "새로고침" 버튼.
- **섹션별 `src/app/<group>/error.tsx`** — 그룹 라우트 (예: `(dashboard)`, `(meeting)`) 단위로 하나씩. 섹션 내 서버 데이터 로드 실패 → 섹션만 폴백 UI 로 교체 + 재시도.
- **컴포넌트 `<ErrorBoundary>`** — `react-error-boundary` 사용. **일부가 터져도 나머지는 살아야 하는 위젯** 에만 (예: 대시보드 카드 1개 실패 시 나머지 카드 유지).

**원칙**:
- "경계는 폴백이 더 유용한 곳" 에만. 모든 컴포넌트에 ErrorBoundary 두는 것 금지.
- 경계는 **UI 복구 + `reportError` 호출** 둘만 책임. 비즈니스 로직 금지.
- 경계 폴백은 **재시도 버튼** 과 **"처음으로"** 를 기본 제공.

### C. `ApiError` 와 코드 매핑

디렉터리:

```
src/lib/errors/
├─ ApiError.ts          # class ApiError extends Error { code, status, cause, ... }
├─ codes.ts             # type ApiErrorCode = 'AUTH_INVALID' | 'MEETING_NOT_FOUND' | ... | 'UNKNOWN'
├─ messages.ts          # codeToMessage(code, fallback): string
├─ handle.ts            # handleApiError(err, { onToast, onInline, onRedirect })
├─ reportError.ts       # 추상 로깅 훅 (#8 까지 console.error, 이후 Sentry)
└─ ErrorBoundary.tsx    # react-error-boundary 래퍼 (reportError 호출 포함)
```

#### `ApiError.ts`

```ts
export class ApiError extends Error {
  readonly code: string;        // 서버가 준 원문 코드 — union 밖이어도 수용
  readonly status: number;      // HTTP status
  readonly fieldErrors?: Record<string, string[]>;  // 422 응답일 때
  readonly cause?: unknown;

  constructor(args: { code: string; status: number; message: string; fieldErrors?: Record<string, string[]>; cause?: unknown }) {
    super(args.message);
    this.name = 'ApiError';
    this.code = args.code;
    this.status = args.status;
    this.fieldErrors = args.fieldErrors;
    this.cause = args.cause;
  }
}
```

#### `codes.ts`

```ts
export type ApiErrorCode =
  | 'AUTH_INVALID'
  | 'AUTH_SESSION_EXPIRED'
  | 'FORBIDDEN'
  | 'MEETING_NOT_FOUND'
  // ... 초기엔 작게. 쓰이는 대로 PR 로 확장.
  | 'UNKNOWN';

export function toApiErrorCode(code: string): ApiErrorCode {
  // union 밖은 UNKNOWN 으로 수렴. union 확장은 PR 필요.
  return (KNOWN_CODES as readonly string[]).includes(code)
    ? (code as ApiErrorCode)
    : 'UNKNOWN';
}
```

- api 리포 `ErrorCode` enum 과 1:1 미러. 서버가 코드 추가하면 **문자열로 받되** `UNKNOWN` 처리, 프런트에서 명시 처리하려면 union 확장 PR.
- 초기 리스트는 **실제 쓰이는 지점이 생겼을 때** 만든다 (지금은 뼈대만).

#### `messages.ts`

```ts
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

- i18n (#7) 적용 시 `MESSAGES` 를 번역 리소스로 치환. **함수 시그니처 유지** → 호출부 변경 없음.

#### `#4` 와 겹치지 않게

- `src/lib/validation/errors.ts` (#4) 는 **폼 전용 변환**만:
  - `ApiError.fieldErrors` → RHF `setError` 포맷으로 배열 평탄화
  - 비필드 에러는 `codeToMessage(err.code)` 를 **import 재사용**
- 메시지 테이블은 `src/lib/errors/messages.ts` **단일 소스**. 중복 금지.

### D. 토스트/인라인/풀페이지 규칙

| 상황 | UX |
|---|---|
| 뮤테이션 성공/실패 (저장·삭제·토글) | **토스트** |
| 폼 필드 에러 | **인라인 아래** (#4) |
| 폼 비필드 에러 | **폼 상단 배너** or **토스트** (호출자 선택) |
| 섹션 서버 데이터 로드 실패 | **섹션 `error.tsx`** 폴백 + 재시도 |
| 위젯 렌더 실패 (대시보드 카드 등) | **`<ErrorBoundary>`** 폴백 + 재시도 |
| 세션 만료 (401) | **리다이렉트** → 로그인 페이지에서 토스트 |
| 권한 없음 (403) | 라우트 가드 **리다이렉트**, 액션 중이면 **토스트** |
| 완전 예상 못한 에러 | **`global-error.tsx`** |

- 토스트 severity: `error` / `warning` / `info` / `success` (`uiStore` API).
- 중복 토스트 deduping: 동일 code+message 가 5초 내 연속이면 1개로 압축 (`uiStore` 가 보장).
- 토스트 UI 라이브러리는 열어둠 (§4) — `uiStore` 가 중립.

### E. React Query 연동

`QueryClient` 기본 옵션:

```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      throwOnError: true,       // 섹션 ErrorBoundary 로 bubbling (의도적)
      retry: 1,
      staleTime: 30_000,
      gcTime: 5 * 60 * 1000,
      refetchOnWindowFocus: false,
    },
    mutations: {
      throwOnError: false,
      onError: (err) => {
        reportError(err, { source: 'rq-mutation' });
        toast.error(codeToMessage((err as ApiError).code ?? 'UNKNOWN'));
      },
    },
  },
});
```

- `throwOnError: true` → 쿼리 에러가 섹션 ErrorBoundary 에 잡힘 (일관된 폴백).
- 호출부가 inline 처리하고 싶으면 `useQuery({ throwOnError: false })` 로 **의식적 opt-out**.
- 뮤테이션 전역 `onError` 를 각 뮤테이션이 override 가능 (예: 낙관 롤백 전용 처리).

### F. 비-폼 Server Action: `ActionResult<T>`

```ts
export type ActionResult<T = unknown> =
  | { ok: true; data: T }
  | { ok: false; code: string; message: string; cause?: unknown };
```

- `FormActionResult` (#4) 와 **shape 통일** (ok discriminated union).
- 폼은 `fieldErrors` 를 추가로 가짐 — 폼 shape 는 `ActionResult` 를 **확장**.
- Server Action 은 **throw 하지 않는다** — 예외는 세션 만료 `redirect('/login')` 뿐.

### G. 로깅 훅 (#8 교체 예정)

```ts
// src/lib/errors/reportError.ts
type Ctx = { source: string; extra?: Record<string, unknown> };

export function reportError(err: unknown, ctx: Ctx): void {
  // 지금: console.error
  // #8 결정 후 Sentry.captureException(err, { tags: { source: ctx.source }, extra: ctx.extra }) 로 교체
  console.error('[reportError]', ctx.source, err, ctx.extra);
}
```

- **인터페이스 선 확정, 구현만 교체** — 호출부(경계·RQ·fetch wrapper) 변경 없음.
- 민감정보 (토큰·개인정보) 마스킹 규칙은 #19 `web-security` 에서 확정.

---

## 3. 다른 축에 미치는 영향

| 이슈 | 내용 |
|---|---|
| [#2](https://github.com/timonPark/flowmeet-web/issues/2) 데이터 페칭 | fetch wrapper 가 던지는 `ApiError` shape 가 이 축의 §C 와 정합해야 함. 변경 없음. |
| [#3](https://github.com/timonPark/flowmeet-web/issues/3) 상태 관리 | `uiStore.toast` API (severity, deduping) 를 명시적으로 요구. |
| [#4](https://github.com/timonPark/flowmeet-web/issues/4) 폼/검증 | `src/lib/validation/errors.ts` 가 `codeToMessage` import. 중복 메시지 테이블 금지. |
| [#8](https://github.com/timonPark/flowmeet-web/issues/8) 로깅/모니터링 | `reportError` 구현만 Sentry 로 교체. 인터페이스 유지. |
| [#7](https://github.com/timonPark/flowmeet-web/issues/7) i18n | `MESSAGES` 테이블을 번역 리소스로 치환. 함수 시그니처 유지. |
| [#13](https://github.com/timonPark/flowmeet-web/issues/13) `nextjs16-reality-check` | Next 16 에서 `error.tsx` 는 "use client" 필수 — 가드에 포함. |
| [#15](https://github.com/timonPark/flowmeet-web/issues/15) `app-router-layering` | `src/lib/errors/` 디렉터리 책임 포함. |
| [#16](https://github.com/timonPark/flowmeet-web/issues/16) `error-ux` 스킬 | 이 문서가 입력. 스킬 체크 포인트: 경계 남용 금지, `codeToMessage` 재사용, 메시지 테이블 중복 금지, `throwOnError` 기본 유지. |
| [#19](https://github.com/timonPark/flowmeet-web/issues/19) `web-security` | `reportError` 가 토큰/개인정보 전송 시 마스킹 규칙 필요. |

---

## 4. 열어둔 포인트 (defer)

| 항목 | 사유 | 어디로 |
|---|---|---|
| 토스트 UI 라이브러리 (Sonner / 자체 구현) | UI 디자인 확정 때 재검토. 지금은 `uiStore.toast` 추상 API 만. | UI 구현 단계 |
| `ApiErrorCode` union 초기 리스트 | 실제 쓰이는 지점이 생길 때 추가. 지금은 뼈대만. | 각 피처 이슈 |
| Sentry/모니터링 통합 | `reportError` 구현 교체만 | #8 |
| 에러 메시지 i18n | `MESSAGES` 테이블을 번역 리소스로 치환 | #7 |
| 섹션 그룹 라우트 구조 | `(dashboard)` / `(meeting)` 등 그룹 라우트 설계 — `app-router-layering` 과 함께 | #15 |

---

## 5. 완료 조건 체크리스트

- [x] 경계 레이어링 전략
- [x] `ApiError` / `codes` / `messages` 분리
- [x] #4 와의 중복 해소
- [x] 토스트/인라인/풀페이지 UX 매핑
- [x] RQ 기본 옵션 (`throwOnError`, `onError`)
- [x] 비-폼 Server Action `ActionResult<T>`
- [x] `reportError` 추상 훅
- [x] 다른 축 영향 매핑
- [x] 열어둔 포인트 이관처
- [ ] CLAUDE.md 축약 반영 + 이 문서 링크 (이 PR)
- [ ] 실제 구현 (이 PR 범위 밖)

## 6. 참고

- [`docs/axis-decisions/data-fetching.md`](./data-fetching.md)
- [`docs/axis-decisions/state-management.md`](./state-management.md)
- [`docs/axis-decisions/form-validation.md`](./form-validation.md)
- [`docs/harness-bootstrap-survey.md`](../harness-bootstrap-survey.md) §2, §4 매핑
- `react-error-boundary` v4 공식 가이드
- Next 16 `error.tsx` / `global-error.tsx` 규약
