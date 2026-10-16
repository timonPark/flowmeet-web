---
name: ts-style
description: flowmeet-web 의 TypeScript·React·파일 네이밍·import·주석 규약. TypeScript/TSX 파일을 쓰거나 리뷰할 때 로드. ESLint 가 자동 강제하는 영역은 생략하고, 사람 판단이 필요한 영역만 명시.
---

# TypeScript Style — flowmeet-web

Next.js 16 App Router + React 19 + TypeScript 5 + ESLint 9 (`eslint-config-next` core-web-vitals + typescript) 기반. **ESLint 가 자동 강제** 하는 포맷/일부 안전 규칙은 아래에서 반복 설명하지 않고, 사람 판단이 필요한 **설계·네이밍·구조·주석** 지침만 명시한다.

> 포맷은 `npm run lint -- --fix` 로 ESLint 가 처리 가능한 범위만 자동 교정. 포매터 (Prettier/dprint) 는 현재 미도입 — [§8 참조](#8-포매터-현재-미도입).

---

## 1. 파일·디렉터리 네이밍

### 1.1. 라우트 (`src/app/`)

Next App Router 규칙을 따르므로 **Next 가 정한 파일명**은 소문자/고정. 현재 Next 16.4 기준:

- 라우트 세그먼트: `page.tsx`, `layout.tsx`, `template.tsx`, `error.tsx`, `global-error.tsx`, `loading.tsx`, `not-found.tsx`, `default.tsx` (parallel route 슬롯 기본값)
- API: `route.ts`
- **프로젝트 레벨 특수 파일** (리포 루트 또는 `src/`):
  - **`proxy.ts`** — Next 16 신규. 과거 `middleware.ts` 자리. **두 파일이 동시에 있으면 Next 가 거부** (constants: `PROXY_FILENAME = 'proxy'`, `MIDDLEWARE_FILENAME = 'middleware'`). 신규 작성은 **`proxy.ts`** 로.
  - `instrumentation.ts` — Server lifecycle 훅.

폴더명은 **kebab-case**:

- `src/app/meeting-rooms/page.tsx`
- `src/app/(dashboard)/team/page.tsx`
- 그룹 라우트: `(group)` — [#5 결정](../../../docs/axis-decisions/error-ux.md) 참고.
- 동적 세그먼트: `[id]`, `[...slug]`, `[[...optional]]`.

### 1.2. 컴포넌트 파일 (`src/components/`, `src/features/*/`, `src/lib/forms/` 등)

- **PascalCase**, 한 파일 한 컴포넌트.
- 파일명 = 컴포넌트명.
  - ✅ `CreateMeetingForm.tsx` → `export function CreateMeetingForm() {}`
  - ❌ `create-meeting-form.tsx`, ❌ `createMeetingForm.tsx`
- 테스트는 코로케이트: `CreateMeetingForm.test.tsx` ([#6 결정](../../../docs/axis-decisions/test-strategy.md)).

### 1.3. 유틸·훅·스키마 (`src/lib/`)

- 함수/상수 중심 파일은 **camelCase**: `fetchClient.ts`, `codeToMessage.ts`, `maskSensitive.ts`.
- 훅은 `use` 접두 유지: `useRecordingSession.ts`.
- 스키마: `src/lib/validation/schemas/meeting.ts` (BC 명)([#4](../../../docs/axis-decisions/form-validation.md)).
- Store slice: `src/lib/stores/<domain>Store.ts` — `uiStore.ts`, `recordingStore.ts` ([#3](../../../docs/axis-decisions/state-management.md)).

### 1.4. 디렉터리 책임 (요약, 상세는 `app-router-layering` 스킬 #15)

- `src/app/` — 라우트, Server Components, `page.tsx` / `layout.tsx` / `error.tsx` / `route.ts` 등 Next 고정 파일
- `src/features/<domain>/` — 피처별 서버·클라이언트 컴포넌트, Server Actions, 피처 전용 훅/유틸
- `src/components/` — 피처 간 재사용되는 범용 UI (피처별 전용은 `features/` 안)
- `src/lib/` — api wrapper, 에러, 폼, 스토어, 유틸 (프레임워크·앱 비의존)
- `src/messages/` — i18n 메시지 ([#7](../../../docs/axis-decisions/a11y-i18n-tokens.md))
- `src/test/` — 테스트 공용 유틸 ([#6](../../../docs/axis-decisions/test-strategy.md))

---

## 2. 네이밍 (심볼)

| 대상 | 규칙 | 예 |
|---|---|---|
| 변수·함수 | camelCase | `fetchDashboard`, `userId` |
| React 컴포넌트 | PascalCase | `CreateMeetingForm`, `DashboardCard` |
| 훅 | `use` 접두 + camelCase | `useRecordingSession` |
| 타입·인터페이스·enum | PascalCase | `type MeetingStatus`, `interface AuthContextValue` |
| 상수 (모듈 수준 불변 값) | SCREAMING_SNAKE_CASE | `const MAX_RETRY = 3` |
| boolean | `is` / `has` / `can` / `should` 접두 | `isSubmitting`, `hasError`, `canEdit` |
| 이벤트 핸들러 props | `on<Event>` | `onClick`, `onSubmit`, `onSelectRow` |
| 이벤트 핸들러 함수 | `handle<Event>` | `handleSubmit`, `handleRowSelect` |
| private 아님 marker `_` | **금지** (JS 는 접근 제어 없음, 혼동 소지) | - |
| 접미사 `I` 인터페이스 (`IFoo`) | 금지 | `AuthContextValue` ✓, `IAuthContextValue` ✗ |

**예외**: Next 가 고정한 식별자는 그대로 (`metadata`, `generateMetadata`, `generateStaticParams`, `revalidate`, `dynamic` 등).

---

## 3. `type` vs `interface`

**디폴트: `type`.** 그 이유로 선호한다:
- union / intersection / mapped types 등 모든 조합 가능
- **선언 병합 (declaration merging)** — 같은 이름의 `interface` 선언이 **자동으로 합쳐지는** TypeScript 동작 — 이 애플리케이션 코드에선 **의도치 않은 확장 소스**가 됨 (서드파티가 같은 이름으로 선언을 끼워 넣을 수도 있음). `type` 은 같은 이름 재선언이 **에러**라 이 위험이 없다.
- extensibility 가 꼭 필요하면 `&` intersection 으로 명시 가능

**`interface` 를 쓰는 경우** (실제 사례):
- 라이브러리의 interface 를 확장 (`declare module 'next' { interface ... }` 안)
- 공개 확장 지점을 제공하는 라이브러리 코드 — 외부 소비자가 선언 병합으로 끼워 넣는 걸 **의도** 할 때 (예: `next-auth` session 확장)
- 그 외 애플리케이션 내부 타입은 `type` 유지

### 선언 패턴

```ts
// ✅ 선호
type MeetingStatus = 'draft' | 'ongoing' | 'ended';

type CreateMeetingInput = {
  title: string;
  startAt: string;
  roomId: string;
};

type Props = {
  meetings: Meeting[];
  onSelect: (id: string) => void;
};

// z.infer 는 사실상 type-alias — 그대로 OK
type FormValues = z.infer<typeof createMeetingSchema>;
```

- `Props` 는 **컴포넌트 모듈 안의 로컬** 로. 외부에서 import 할 일 거의 없음.
- 여러 모듈이 공유하는 타입은 **의미 이름** (`Meeting`, `MeetingSummary`) 으로. `Props`·`State` 같은 보통명사 export 금지.

---

## 4. Import 규칙

### 4.1. Alias `@/*` 필수

`tsconfig.json` 의 `"paths": { "@/*": ["./src/*"] }` 사용. **상대경로 지양** — 다음과 같은 경우 외 금지:

- 같은 디렉터리 내부: `./CreateMeetingForm` (허용)
- 같은 피처 내 하위: `./actions`, `./schema` (허용)

**금지**:
- `../../../lib/api/serverClient` — alias 로 `@/lib/api/serverClient`
- `../../features/meeting/CreateMeetingForm` — alias 또는 재구성

### 4.2. Import 그룹 순서

ESLint 가 완전 자동정렬하진 않으므로 **사람 수동 유지**:

```ts
// 1) React / Next 프레임워크
import { useState } from 'react';
import Link from 'next/link';

// 2) 외부 라이브러리
import { useForm } from 'react-hook-form';
import { z } from 'zod';

// 3) alias (@/*)  — 가까운 레이어에서 먼 레이어 순서는 자유
import { apiError } from '@/lib/errors';
import { createMeetingSchema } from '@/lib/validation/schemas/meeting';

// 4) 상대 경로
import { CreateMeetingFormFields } from './CreateMeetingFormFields';
import { createMeetingAction } from './actions';

// 5) 타입-only
import type { Meeting } from '@/lib/types/meeting';
```

- 타입-only import 는 **별도 그룹 끝** (`import type { ... }`).
- 각 그룹 사이 **빈 줄 1개**.
- **현재 ESLint 가 이 그룹 순서를 자동 강제하지는 않는다** — 로컬 컨벤션으로 PR 리뷰에서 유지. 추후 `@typescript-eslint/consistent-type-imports` + `import/order` 도입 시 자동화 재검토.

### 4.3. Export 패턴

- **named export 선호** (flowmeet-web 결정 — Next 자체는 일반 컴포넌트의 default export 를 금지하지 않음).
- **default export 가 Next 측에서 요구되는 파일**: `page.tsx`, `layout.tsx`, `template.tsx`, `error.tsx`, `global-error.tsx`, `loading.tsx`, `not-found.tsx`, `default.tsx`, `proxy.ts`, `instrumentation.ts`. 이들만 default export 사용.
- 그 외는 **named export 만** — rename-via-import 로 네이밍 drift 발생 방지 (프로젝트 결정).

```ts
// ✅ 피처 컴포넌트
export function CreateMeetingForm() {}

// ✅ Next 라우트
export default function Page() {}
export const metadata: Metadata = { title: '...' };
```

- **Barrel export (`index.ts` 로 전체 재export) 지양** (프로젝트 결정): 번들러의 tree-shaking 은 보통 잘 동작하지만 — 1) **순환 import 유발 가능**, 2) **import 추적성 저하** (어느 파일에서 왔는지 안 보임), 3) **모듈 초기화 부작용 전파** (barrel import 가 쓰이지 않는 하위 모듈까지 평가 트리거 가능). 명시 import 가 안전.

---

## 5. 컴포넌트 작성

### 5.1. 함수 컴포넌트, 선언문

```ts
// ✅ 선호 — function 선언
export function CreateMeetingForm({ meetings }: Props) {
  return <form>...</form>;
}

// 가능 — const + arrow (팀 선호 따름, 혼용 금지)
export const CreateMeetingForm = ({ meetings }: Props) => {
  return <form>...</form>;
};
```

- 이 리포는 **`function` 선언문 통일**. `const + arrow` 금지 — hoisting 과 스택 트레이스 구분 명확성.

### 5.2. Props 타입

- 모듈 로컬 `type Props`.
- Props 안의 `children` 은 `React.ReactNode`. `JSX.Element` 는 더 좁아져서 유연성 떨어짐.
- `className?: string` 은 재사용 UI 에서만 받음 (피처 컴포넌트는 받지 말 것, 스타일 drift 소스).

```ts
type Props = {
  meetings: Meeting[];
  onSelect: (id: string) => void;
  children?: React.ReactNode;
};
```

### 5.3. `"use client"` 는 트리 **꼭대기만**

- 상태/이벤트가 필요한 컴포넌트에만 `"use client"`. 부모가 `"use client"` 면 자식은 자동으로 클라이언트 번들.
- `"use client"` 안에서 서버 전용 함수 import 금지 (`cookies()`, `headers()`, `fs/promises` 등).

### 5.4. React 19 / Next 16 패턴

- **`forwardRef` — 내부 컴포넌트는 지양**: React 19 에서 함수 컴포넌트는 `ref` 를 일반 prop 처럼 받을 수 있다 (`ref: React.Ref<HTMLInputElement>`). 내부 피처 컴포넌트는 새 패턴 사용.
  - **예외**: 외부 라이브러리에 export 하는 공개 컴포넌트, 또는 기존 React 18 타입 소비자가 많은 경우엔 `forwardRef` 유지 가능. 타입 호환성 확인 후 결정.
- `useFormStatus` 는 **RHF 와 혼용 금지** ([#4](../../../docs/axis-decisions/form-validation.md) · [#13 nextjs16-reality-check](../nextjs16-reality-check/SKILL.md)).
- Server Component 는 async 가능, Client Component 는 async 불가 (Suspense 로 감쌈).

---

## 6. 타입 안전성

### 6.1. `any` 금지 → `unknown` 사용

- `any` 는 **전염** 되어 타입 추적 끊김. **금지**.
- 외부 입력·`JSON.parse` 결과 등 **진짜 모를 때** `unknown` 사용 후 좁히기 (`typeof`·`in`·zod `safeParse`).
- 라이브러리 d.ts 가 `any` 를 쓰면 wrapper 에서 `unknown` 으로 좁혀 밖에 노출.

### 6.2. `as` 캐스트 신중히

- 다른 수단 (type guard, `zod.parse`, generics) 이 가능하면 그걸로.
- `as` 가 필요하면 **왜** 를 1줄 주석으로 (예: `// 서버 schema 가 string literal union 이지만 d.ts 는 string 반환`).
- `as any` **절대 금지**.

### 6.3. non-null 단언 `!` 지양

- `foo!.bar` 식 non-null 단언은 **런타임 가정** — 틀리면 조용히 undefined 참조.
- 조건부 체이닝 (`foo?.bar`) 또는 **narrowing (조기 return)** 선호.
- 테스트 코드는 예외 — 테스트 setup 이 보장하면 `!` 사용 가능.

### 6.4. `satisfies` 활용

```ts
// 값은 좁은 타입 추론 유지, 넓은 타입 만족 검증
const QUERY_KEYS = {
  meetings: ['meetings'],
  meeting: (id: string) => ['meeting', id],
} satisfies Record<string, unknown>;
```

### 6.5. Nullable 표현

- **내부 애플리케이션 타입**은 `undefined` 로 통일. React 폼의 미입력 필드, 옵셔널 필드 등은 `undefined`.
- **API/DB 계약이 `null` 과 `undefined` 를 의미적으로 구분**할 때 (예: PATCH 요청에서 `null` = "지우기" vs `undefined` = "변경 없음") 는 **계약을 우선 존중**. 경계에서 함부로 변환하지 않는다.
- 의미 구분이 없는 외부 경계에서는 **경계에서 `?? undefined`** 로 변환해 내부는 `undefined` 통일.
- `type Foo | null | undefined` 를 애플리케이션 내부 타입에서 **지양** — 둘 중 하나로 통일. 외부 계약 타입은 그대로 유지 가능.

---

## 7. 주석 (CLAUDE.md 와 정합)

- **디폴트: 주석 없음**. 식별자 이름이 의미를 전달하면 주석 중복.
- **쓰는 경우**: *왜*. 숨은 제약·미묘한 invariant·workaround·놀라운 동작.
- **쓰지 않는 경우**:
  - *무엇을* 하는지 설명 (이름이 할 일)
  - PR 설명을 코드에 재탕 ("added for X ticket")
  - history 적기 ("previously was Y")
- JSDoc 은 **공개 API 로 쓰이는 유틸** 에만. 내부 함수에 JSDoc 과잉 금지.

```ts
// ❌ 과잉
// Returns the user's full name.
function getFullName(user: User) {
  return `${user.firstName} ${user.lastName}`;
}

// ✅ 왜가 명확한 주석
// Deepgram STT 가 10kB 미만 chunk 를 리젝트. 녹음 시작 직후 짧은 chunk 는 모아 보낸다.
if (chunk.byteLength < MIN_UPLOAD_CHUNK) {
  pending.push(chunk);
  return;
}
```

---

## 8. 포매터 (현재 미도입)

- **Prettier/dprint 미설치**. 현재 `eslint-config-next` 가 로드하는 규칙만 적용 중 — 포맷 규칙이 포함되는지는 설정에 따라 다르므로 **"ESLint 가 포맷 전체를 보장한다" 고 가정하지 않는다**.
- **권장 (추후 도입)**: **Prettier** — Tailwind·Next·RHF 커뮤니티 사실상 표준. 도입 전까지는 수작업 포맷 drift 는 PR 리뷰에서 지적.
- 포매터 도입은 **별도 chore PR** 로 (논의 분리). 이 스킬은 **포맷이 아니라 설계·구조·네이밍** 에 집중.

---

## 9. ESLint 규칙 커스터마이즈

`eslint.config.mjs` 는 `eslint-config-next/core-web-vitals + typescript` 를 그대로 사용.

**지금 상태 유지** (추가 규칙 최소). 다음은 **추후 추가 고려** 레벨:

- `@typescript-eslint/no-explicit-any`: `error` (현재 warn 가능)
- `@typescript-eslint/consistent-type-imports`: `error` + `prefer: 'type-imports'`
- `import/order`: 그룹 자동 정렬

**추가는 별도 chore PR**. 추가 시 **기존 코드가 깨지지 않는지** 확인 (`npm run lint`).

---

## 10. 체크리스트 — 코드 작성·리뷰 시

- [ ] 파일명이 역할에 맞는가 (컴포넌트 PascalCase.tsx / 유틸 camelCase.ts / 라우트 Next 고정)?
- [ ] boolean 변수에 `is`/`has`/`can`/`should` 접두가 있는가?
- [ ] 이벤트 handler prop 이 `on<Event>`, 함수가 `handle<Event>` 인가?
- [ ] 타입이 `type` 우선인가? 라이브러리 확장이 아닌데 `interface` 를 쓰는 이유가 있는가?
- [ ] import 가 `@/*` alias 를 썼는가? 상대경로가 불필요하게 깊지 않은가?
- [ ] default export 가 **Next 라우트 파일에만** 쓰이는가?
- [ ] `any` 가 없는가? 있다면 `unknown` 으로 좁힐 수 있는가?
- [ ] `as` 캐스트가 있다면 **왜** 1줄 주석이 있는가?
- [ ] non-null `!` 가 있다면 narrowing 으로 대체 가능한가 (테스트 제외)?
- [ ] 주석이 **왜** 를 설명하는가, **무엇을** 중복 설명하지는 않는가?
- [ ] `className` prop 을 피처 컴포넌트가 받지 않는가 (재사용 UI 만)?
- [ ] `"use client"` 가 트리 꼭대기에만 있는가 (불필요하게 아래쪽에도 흩뿌리지 않는가)?

---

## 11. 참고

- `node_modules/eslint-config-next/*.d.ts` — 현재 로드되는 ESLint 규칙 셋
- `tsconfig.json` — `strict: true`, `moduleResolution: 'bundler'`, `@/*` alias
- `eslint.config.mjs` — 추가 커스터마이즈 자리
- 결정 축: [#2](../../../docs/axis-decisions/data-fetching.md), [#3](../../../docs/axis-decisions/state-management.md), [#4](../../../docs/axis-decisions/form-validation.md), [#5](../../../docs/axis-decisions/error-ux.md), [#7](../../../docs/axis-decisions/a11y-i18n-tokens.md)
- 관련 스킬: [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md)
