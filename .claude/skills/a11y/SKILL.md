---
name: a11y
description: flowmeet-web 접근성 가드 — jsx-a11y lint, Radix UI primitives 사용, CVA variants, @axe-core/playwright 자동 검사, 공용 폼 컴포넌트 ARIA 보장. 컴포넌트·UI·폼 코드 쓸 때 로드. #7 결정의 a11y 축 실무 적용.
---

# a11y — flowmeet-web

[#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) §A 결정을 코드로 적용. "무엇을" 은 결정 축, "그래서 코드를 어떻게" 는 이 스킬.

---

## 1. Lint — `jsx-a11y/recommended` 유지

- `eslint-config-next` 가 자동 포함. 추가 설정 불필요.
- `strict` / `all` 로 상향은 UI 안정화 후 재검토 (현재 불필요).

---

## 2. 접근성 복잡 컴포넌트 — Radix UI primitives

### 2.1. 도입 대상 (초기)

| 컴포넌트 | Radix 패키지 | 이유 |
|---|---|---|
| Dialog | `@radix-ui/react-dialog` | 포커스 트랩·focus restoration·`aria-modal`·`esc` 처리 자동 |
| AlertDialog | `@radix-ui/react-alert-dialog` | **별도 패키지**. 파괴 작업 확인 다이얼로그 |
| Dropdown Menu | `@radix-ui/react-dropdown-menu` | 키보드 네비·`role="menu"`·`aria-expanded` |
| Popover | `@radix-ui/react-popover` | 포커스 관리·`aria-haspopup` |
| Tabs | `@radix-ui/react-tabs` | `role="tablist"`·`role="tabpanel"`·화살표 키 |
| Tooltip | `@radix-ui/react-tooltip` | hover + focus 동시 지원, `aria-describedby` |
| Toast | **Sonner** (Radix 아님) | 접근성 양호, `uiStore` 큐의 실제 렌더 |

### 2.2. 금지

- 수제 Modal / Dropdown / Popover / Tooltip — 포커스 트랩, focus restoration, `aria-*`, `esc` 키, 키보드 네비게이션의 함정이 매우 많음. **반드시 Radix** 또는 Radix 기반 wrapper 사용.
- shadcn/ui 전체 CLI 복제는 **현재 안 함** — 생산성 이슈 체감 시 선택 복사 검토 ([#7](../../../docs/axis-decisions/a11y-i18n-tokens.md) §4 열어둠).

---

## 3. CVA variants — **단일 소유자는 `design-tokens`**

- Button / Badge / Input 등 공용 variant 정의는 **`src/lib/ui/variants.ts`** 한 곳. 소유 스킬은 [`design-tokens`](../design-tokens/SKILL.md) §2.2.
- `a11y` 스킬은 **a11y 관점 요구만 명시**:
  - 모든 variant 의 base 에 **`focus-visible:ring-2 ring-primary ring-offset-2`** 포함 (포커스 가시화)
  - `disabled:opacity-50 disabled:pointer-events-none` (비활성 접근성)
  - interactive 요소에 `cursor-pointer` 는 피함 (버튼은 기본 pointer, 필요 시 shape 로 전달)
- a11y 스킬이 variant 코드를 **중복 선언 금지** — drift 방지.

---

## 4. 공용 폼 컴포넌트 — ARIA 기본 제공

```tsx
// src/lib/forms/TextField.tsx
'use client';
import { useId } from 'react';
import { useController, type FieldValues, type UseControllerProps } from 'react-hook-form';

type Props<T extends FieldValues> = UseControllerProps<T> & {
  label: string;
  description?: string;
  type?: 'text' | 'email' | 'password';
};

export function TextField<T extends FieldValues>({ label, description, type = 'text', ...control }: Props<T>) {
  const id = useId();
  const descId = description ? `${id}-desc` : undefined;
  const { field, fieldState } = useController(control);
  const errorId = fieldState.error ? `${id}-err` : undefined;

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      {description && <p id={descId} className="text-sm text-muted-fg">{description}</p>}
      <input
        {...field}
        id={id}
        type={type}
        aria-invalid={!!fieldState.error}
        aria-describedby={[descId, errorId].filter(Boolean).join(' ') || undefined}
        aria-required={control.rules?.required !== undefined}
      />
      {fieldState.error && <p id={errorId} role="alert">{fieldState.error.message}</p>}
    </div>
  );
}
```

**필수 ARIA**:
- `<label htmlFor>` — 입력에 접근 가능한 이름.
- `aria-invalid` — 에러 상태.
- `aria-describedby` — description + error 를 입력에 연결.
- `role="alert"` — 에러가 실시간으로 스크린리더에 알려짐.
- `useId()` — SSR hydration safe 한 유니크 ID.

---

## 5. 런타임 접근성 검사 — `@axe-core/playwright`

E2E 각 스펙 fixture 에서 자동 실행 ([`web-testing`](../web-testing/SKILL.md) §11.3 참조).

- **critical / serious → fail**
- **moderate / minor → warning** (console.warn + reporter attach, 통과)

---

## 6. 공용 규약

- **키보드 접근**: 모든 interactive 요소는 Tab 으로 접근 가능, Enter/Space 로 활성화. `onClick` 만 있는 `<div>` 금지 → `<button>` 또는 role + tabIndex + keydown 처리.
- **포커스 가시화**: `focus-visible:` 유틸 필수. `outline: none` 금지 (대체 포커스 ring 없이).
- **색만으로 상태 전달 금지**: 에러는 색 + 아이콘/텍스트, 성공은 색 + 체크 등.
- **아이콘 전용 버튼**: `aria-label="..."` 필수 (ESLint 가 상당수 catch).
- **`alt` 속성**: 의미 있는 이미지는 설명, 장식용은 `alt=""`.
- **헤딩 계층**: `<h1>` 하나, `<h2>`·`<h3>` 순차. 건너뛰기 금지.
- **Landmark**: `<header>`·`<nav>`·`<main>`·`<footer>` 등 명시.
- **모션**: `@media (prefers-reduced-motion: reduce)` 존중. 큰 animation 은 조건부 비활성화.

---

## 7. 금지 패턴

- 수제 Modal / Dropdown / Popover / Tooltip
- `onClick` 만 있는 `<div>` / `<span>` interactive
- `outline: none` 를 포커스 대체 없이
- 색만으로 에러·성공 구분
- 아이콘 전용 버튼에 `aria-label` 없음
- `tabIndex={-1}` 를 포커스 받아야 할 요소에
- 모달 열려있을 때 배경 스크롤 가능 (Radix 가 처리하는 영역, 수제 금지)

---

## 8. 체크리스트

- [ ] 복잡 컴포넌트가 Radix (또는 Sonner) 기반인가? 수제 아닌가?
- [ ] `focus-visible:` 포커스 ring 이 모든 interactive 에 있는가?
- [ ] 아이콘 전용 버튼에 `aria-label` 이 있는가?
- [ ] 폼 입력이 `<label htmlFor>` + `aria-invalid` + `aria-describedby` 구성인가?
- [ ] 에러 메시지가 `role="alert"` 로 발표되는가?
- [ ] 색만으로 상태 전달하지 않는가 (아이콘·텍스트 보조)?
- [ ] 키보드만으로 모든 상호작용 가능한가?
- [ ] 헤딩 계층이 순차적인가? landmark (`<nav>`·`<main>` 등) 사용?
- [ ] E2E 스펙에 `expectNoA11yViolations()` 가 포함?

---

## 9. 참고

- 결정 축: [#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) §A
- 관련 스킬: [`ts-style`](../ts-style/SKILL.md), [`error-ux`](../error-ux/SKILL.md) §5 (토스트), [`web-testing`](../web-testing/SKILL.md) §11.3·§12 (axe E2E)
- Radix UI 공식 docs, WCAG 2.1 AA
