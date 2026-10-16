---
title: 폼/검증 전략
date: 2026-10-16
author: kindcorder@gmail.com
status: DECIDED
issue: "#4"
scope: flowmeet-web 폼 라이브러리·스키마 검증·서버 응답 매핑 규칙
depends_on: ["#2 데이터 페칭/캐시 전략", "#3 상태 관리 전략"]
---

# 폼/검증 전략

[#2 데이터 페칭](./data-fetching.md) 에서 **폼 제출 = Server Actions + `revalidateTag`**, [#3 상태 관리](./state-management.md) 에서 **폼 중간값을 Zustand/Context 로 복제 금지** 가 확정됐다. 이 축은 **"Server Actions가 뒤를 맡는 상황에서 클라이언트 폼 UX·검증을 뭐로 할지"** 를 결정한다.

> 이 문서는 `error-ux` 스킬([#16](https://github.com/timonPark/flowmeet-web/issues/16)) 의 입력 중 하나다 — 특히 §3 서버 응답 매핑 규칙.

---

## 1. 결정 요약

| 차원 | 결정 |
|---|---|
| 폼 라이브러리 | **react-hook-form v7** |
| 리졸버 | **@hookform/resolvers/zod** |
| 스키마 검증 | **zod** |
| 스키마 위치 | **`src/lib/validation/schemas/<domain>.ts` — 클라·서버 공용** |
| Server Action 반환 shape | **`FormActionResult<T>` 표준 (`ok` discriminated union)** |
| 서버 에러 → 폼 반영 | **`422` → `fieldErrors` → RHF `setError`, 그 외는 `formError`/토스트** |
| 비-폼 상호작용 (토글·화자매칭) | **RHF 쓰지 않음, RQ mutation 직결** (#2 결정) |

**원칙**: *서버가 진실의 근원. 클라 선검증은 UX 목적일 뿐, 서버가 돌려준 에러로 덮어쓴다.*

---

## 2. 결정 상세

### 2.1. 라이브러리 선택

- **react-hook-form v7** — uncontrolled 베이스로 빠르고, 중첩/배열/동적 필드 레시피가 풍부. 산업 표준.
- **zod** — RHF 공식 리졸버, 백엔드 `ErrorCode` 매핑·공유 스키마·향후 i18n 과의 조합 폭이 가장 넓음.

채택하지 않은 것:

| 후보 | 미채택 사유 |
|---|---|
| Conform | progressive enhancement (JS 없는 환경 지원) 이점이 핵심인데, 이 앱은 전부 인증 월드라 체감 X. 낙관 업데이트·복잡 상호작용에서 손이 더 감. |
| React 19 `useActionState` 베어 | 중첩/배열/동적 필드에서 수제 코드 폭증. 1-2 필드 단순 폼엔 가능하지만 전용 라이브러리 혼용 금지. |
| TanStack Form | 아직 레시피 부족, 번들 큼. 재검토는 1년 뒤. |
| valibot | ~11kb 번들 이득이 어드민 성격 앱에서 체감 X. 생태계 열세. |
| TS 타입만 | **금지** — 서버 응답/사용자 입력 런타임 검증 필수. |

### 2.2. 스키마 조직

디렉터리:

```
src/lib/
├─ validation/
│  ├─ schemas/
│  │   ├─ meeting.ts      # createMeetingSchema, updateMeetingSchema, ...
│  │   ├─ directive.ts
│  │   └─ user.ts
│  └─ errors.ts           # mapServerErrorToFormErrors(), ApiErrorCode → form field 매핑
└─ forms/                 # 폼 공용 컴포넌트 (TextField, Select, DatePicker 래퍼 등)
```

- 한 스키마가 **클라 선검증 (RHF resolver)** 과 **Server Action 입력 파싱** 양쪽에서 공용 — 소스 1건, drift 원천 차단.
- Server Action 상단에서 `schema.safeParse(input)` 로 서버 측도 반드시 재검증 (클라를 신뢰하지 않는다).
- 스키마 네이밍 규약: `<verb><Entity>Schema` (예: `createMeetingSchema`). 타입은 `z.infer<typeof createMeetingSchema>` 로 바로 끌어 씀.

### 2.3. Server Action 반환 shape

```ts
// src/lib/validation/formAction.ts (예정)
export type FormActionResult<T = unknown> =
  | { ok: true; data: T }
  | {
      ok: false;
      fieldErrors?: Record<string, string[]>;
      formError?: string;
      code?: ApiErrorCode;
    };
```

- Server Action 은 **throw 하지 않고 이 shape 를 반환**. (throw 하면 Server Action 자동 에러 경계로 가 UX 가 거침)
- 예외: 401/403 등 세션 만료는 Server Action 내부에서 `redirect('/login')` — 반환 shape 이전에 처리.

### 2.4. 서버 에러 → 폼 매핑 규칙

| HTTP | 서버 응답 | 처리 |
|---|---|---|
| 422 | `{ fieldErrors: {...} }` | RHF `setError(field, { message })` 로 각 필드에 꽂음 |
| 400 / 409 / 기타 비필드 | `{ formError, code }` | 폼 상단 inline + 선택적 토스트 (`uiStore`) |
| 401 | - | Server Action 내 `redirect('/login')` |
| 403 | - | 상위 가드가 처리 (라우트 가드 또는 토스트) |
| 500+ | `{ formError: '잠시 후 다시 시도해 주세요.' }` + `reportError` (#8) | 기본 메시지 표시, 상세는 로깅으로 |

**매핑 함수는 `src/lib/validation/errors.ts` 에 한 곳에만 둔다** — 각 폼에서 재구현 금지. 폼 측은 결과 shape 만 소비.

### 2.5. 제출 플로우 (레시피)

```tsx
// src/features/meeting/CreateMeetingForm.tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createMeetingSchema } from '@/lib/validation/schemas/meeting';
import { createMeetingAction } from './actions';
import { toast } from '@/lib/stores/uiStore';

type FormValues = z.infer<typeof createMeetingSchema>;

export function CreateMeetingForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(createMeetingSchema),
  });

  const onSubmit = form.handleSubmit(async (values) => {
    const res = await createMeetingAction(values);
    if (!res.ok) {
      if (res.fieldErrors) {
        Object.entries(res.fieldErrors).forEach(([field, messages]) => {
          form.setError(field as keyof FormValues, { message: messages.join(', ') });
        });
      } else if (res.formError) {
        toast.error(res.formError);
      }
      return;
    }
    // 성공 처리
  });

  return <form onSubmit={onSubmit}>{/* ... */}</form>;
}
```

- 제출 중 표시: `form.formState.isSubmitting` 사용. React 19 `useFormStatus` 는 Conform/베어 플로우용이라 **혼용하지 않음**.
- Server Action 내부도 반드시 `src/lib/api/serverClient.ts` 를 경유 (#2 결정) — 직접 `fetch` 금지.

### 2.6. 비-폼 상호작용

- 낙관 업데이트가 필요한 토글·화자매칭 라벨 저장 등은 **RHF 를 쓰지 않는다** — RQ mutation 으로 직결 (#2, #3 결정).
- "폼" 의 정의: 사용자가 **여러 필드를 채우고 제출 버튼으로 commit** 하는 상호작용. 그 외 (버튼 1개, 토글 1개) 는 폼이 아니다.

### 2.7. 공용 폼 컴포넌트

- `src/lib/forms/` 에 `TextField`, `Select`, `DatePicker`, `Checkbox` 등 RHF `Controller` 래퍼를 둔다.
- 호출부는 `name`·`label`·`description` 등 **의미 중심 props** 만 받고, RHF 내부 디테일은 숨긴다.
- a11y: `aria-invalid`, `aria-describedby` 를 공용 컴포넌트에서 보장 (#7 과 연계).

---

## 3. 다른 축에 미치는 영향

| 이슈 | 내용 |
|---|---|
| [#2](https://github.com/timonPark/flowmeet-web/issues/2) 데이터 페칭 | 변경 없음 — 폼 제출 = Server Action 전제 유지. Server Action 내부 `serverClient` 사용 재확인. |
| [#3](https://github.com/timonPark/flowmeet-web/issues/3) 상태 관리 | 폼 중간값은 RHF 가 보유, Zustand/Context 로 복제 금지 원칙 유지. 토스트는 `uiStore` 경유. |
| [#5](https://github.com/timonPark/flowmeet-web/issues/5) 에러 처리 | §2.4 매핑 규칙이 `error-ux` 의 "폼 에러 UI" 부분 입력. `errors.ts` 매핑 함수가 공통 포인트. |
| [#7](https://github.com/timonPark/flowmeet-web/issues/7) a11y / i18n | `aria-*` 는 공용 폼 컴포넌트에서 보장. zod 에러 메시지 i18n 은 §4 열어둔 포인트. |
| [#15](https://github.com/timonPark/flowmeet-web/issues/15) `app-router-layering` | `src/lib/validation/`, `src/lib/forms/` 디렉터리 책임을 레이어링 룰에 포함. |
| [#16](https://github.com/timonPark/flowmeet-web/issues/16) `error-ux` | `FormActionResult<T>` 와 `fieldErrors`/`formError` 네이밍, `uiStore` 토스트 호출 규약을 공유. |

---

## 4. 열어둔 포인트 (defer)

| 항목 | 사유 | 어디로 |
|---|---|---|
| zod 에러 메시지 i18n | 다국어 전략 확정 후 (#7) 재검토. 지금은 inline 커스텀 메시지. | #7 |
| 파일 업로드 레시피 | 녹음 chunk 는 전용 플로우(폼 아님). 아바타 등 일반 파일 업로드가 생기면 추가. | 해당 피처 이슈 |
| 서버 응답 DTO ↔ 스키마 중복 방지 | base schema + extend 패턴. 레시피는 `error-ux` / `data-fetching-guard` 스킬 작성 시. | #16 / #17 |
| 폼 자동 저장 (draft) | TRD 상 명시 요구 없음. 요구 생기면 feature flag + `localStorage`/IndexedDB 어댑터로 추가. | 추후 신규 이슈 |

---

## 5. 완료 조건 체크리스트

- [x] 라이브러리·스키마·반환 shape·에러 매핑 규칙 결정
- [x] 디렉터리 조직 (`validation/` + `forms/`)
- [x] 제출 플로우 레시피
- [x] 비-폼 상호작용과의 경계 명시
- [x] 다른 축 영향 매핑
- [x] 열어둔 포인트 이관처
- [ ] CLAUDE.md 축약 반영 + 이 문서 링크 (이 PR 에 포함)
- [ ] 실제 구현 (이 PR 범위 밖)

## 6. 참고

- [`docs/axis-decisions/data-fetching.md`](./data-fetching.md)
- [`docs/axis-decisions/state-management.md`](./state-management.md)
- [`docs/harness-bootstrap-survey.md`](../harness-bootstrap-survey.md) §2
- CLAUDE.md 설계 원칙 "폼 검증은 백엔드 응답을 신뢰하되 UX 선검증은 클라이언트"
- react-hook-form v7 + @hookform/resolvers/zod 공식 가이드
