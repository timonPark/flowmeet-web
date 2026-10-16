---
title: 접근성·i18n·디자인 토큰 전략
date: 2026-10-16
author: kindcorder@gmail.com
status: DECIDED
issue: "#7"
scope: a11y 린트/런타임 검사·헤드리스 접근성 라이브러리·next-intl 도입·Tailwind 4 토큰
depends_on:
  - "#4 폼/검증 전략"
  - "#5 에러 처리(error-ux) 전략"
  - "#6 테스트 전략"
---

# 접근성·i18n·디자인 토큰 전략

세 영역을 하나의 축에 묶는 이유: **디자인 토큰** 이 **다크모드/대비** 로 접근성에 영향을 주고, **메시지 i18n** 이 접근성 (스크린리더 언어 선언) 과 폼 에러 UX (#4·#5) 양쪽을 건드리기 때문이다. 세 영역 모두 **"지금 구조만 심어두고, 완성은 UI 안정화 후"** 라는 시점을 공유한다.

> 이 문서는 신규 축 묶음 스킬([#20](https://github.com/timonPark/flowmeet-web/issues/20)) 의 입력 — `a11y` / `i18n` / `design-tokens` 세 서브-스킬이 각각 §A·§B·§C 를 소스로 삼는다.

---

## 1. 결정 요약

| 영역 | 결정 |
|---|---|
| **a11y lint** | `eslint-plugin-jsx-a11y/recommended` 유지 (`eslint-config-next` 기본) |
| **a11y 헤드리스 라이브러리** | **Radix UI primitives** (Dialog / Dropdown Menu / Tooltip / Toast / Tabs / Popover 등 선택 도입) |
| **a11y 런타임 검사** | **`@axe-core/playwright`** E2E `afterEach` 로 각 페이지 자동 스캔 |
| **variants 관리** | **CVA (`class-variance-authority`)** |
| **i18n 라이브러리** | **next-intl v3**, 로케일 1개 (`ko`) 로 시작 |
| **`<html lang>`** | **`"ko"` 교정** (현재 `"en"` 은 버그) |
| **메시지 조직** | `src/messages/ko.json`, 네임스페이스 (`common`, `meeting`, `auth`, `dashboard`, `errors`, `validation`) |
| **디자인 토큰** | **Tailwind 4 `@theme` + CSS variables**, **의미 토큰** (shadcn 식) |
| **다크모드** | **토큰 구조 지원** (`data-theme="dark"`), **토글 UI 는 추후** |
| **폰트** | **Pretendard** + `system-ui` 폴백 (`next/font/local`) |
| **색 대비 기준** | WCAG **AA** (4.5:1) — 토큰 설계 시 확인 |

**원칙**: *지금 구조·규약·토큰 이름만 정한다. 실제 UI·토글·다국어 추가는 UI 구현 단계. "미래 비용이 큰 결정" 만 지금 못 박음.*

---

## 2. 결정 상세

### A. 접근성 (a11y)

#### A.1 Lint 레벨

- **`jsx-a11y/recommended` 유지** — `eslint-config-next` 가 이미 포함. strict/all 은 UI 안정화 후 재검토.
- 개별 rule 상향 대신 **"린트가 잡지 못하는 접근성 문제는 런타임 axe 가 잡는다"** 로 역할 분리.

#### A.2 Radix UI primitives

- **접근성이 복잡한 컴포넌트만** Radix 사용. 수제 구현하면 포커스 트랩·`aria-*`·키보드 네비·focus restoration 등 함정 매우 많음.
- **도입 대상 (초기)**:
  - `@radix-ui/react-dialog` — Dialog, AlertDialog
  - `@radix-ui/react-dropdown-menu` — Dropdown
  - `@radix-ui/react-popover` — Popover
  - `@radix-ui/react-tabs` — Tabs
  - `@radix-ui/react-tooltip` — Tooltip
  - `sonner` — Toast (Radix 가 아니지만 접근성 양호, `uiStore` 큐의 실제 렌더)
- **도입 안 함**:
  - Radix 가 아닌 자체 수제 Modal/Dropdown/Tooltip 금지
  - shadcn/ui 전체 CLI 복제는 **일단 안 함** — 피처별로 Radix + CVA 로 조립. 생산성 이슈 생기면 shadcn 선택 복사 하이브리드 재검토 (§4 열어둠).

#### A.3 CVA (class-variance-authority)

- 컴포넌트 variants 관리: `<Button variant="primary" size="sm">` 식.
- Tailwind 클래스를 **의미 variant 로 추상화** → 호출부가 유틸 클래스 조립하지 않도록.
- 위치: `src/lib/ui/variants.ts` (또는 컴포넌트 파일 안의 `const buttonVariants = cva(...)`).

#### A.4 런타임 접근성 검사

- **`@axe-core/playwright`** 를 E2E 스펙 `afterEach` 로.
  ```ts
  // e2e/fixtures.ts
  import AxeBuilder from '@axe-core/playwright';
  export async function expectNoA11yViolations(page) {
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa'])
      .analyze();
    const serious = results.violations.filter(v => ['critical', 'serious'].includes(v.impact ?? ''));
    expect(serious, violationsSummary(results)).toHaveLength(0);
  }
  ```
- **임계값**: `critical` + `serious` → **fail**. `moderate` / `minor` → **warning (리포트만, 통과)**.
- 규제 상향은 UI 안정화 후.

#### A.5 공용 규약

- **Interactive 요소**: `focus-visible:` 유틸로 포커스 가시화 필수.
- **색만으로 상태 전달 금지** — 아이콘/텍스트 보조.
- **라벨링**: 아이콘 전용 버튼은 `aria-label` 필수. ESLint 가 상당수 잡아줌.
- **키보드 네비**: 모든 상호작용은 키보드만으로 가능. 커스텀 스크립트로 포커스 조작은 Radix 에 위임.
- **모션**: `prefers-reduced-motion` 미디어 쿼리 존중.

---

### B. 국제화 (i18n)

#### B.1 라이브러리

- **next-intl v3**. App Router + Server Components + Server Actions 공식 지원.
- `next-i18next` 미채택: Pages Router 중심, App Router 지원 약함.
- 하드코딩 유지 미채택: 메시지 하드코딩 후 i18n 적용은 현실적으로 **전면 리팩터**. 지금 키-값 분리 비용은 미미.

#### B.2 로케일

- **초기 `ko` 1개**.
- `defaultLocale: 'ko'`, 추가 로케일 없음.
- 추후 로케일 추가 시 라우팅 전략 (`/[locale]/...` vs 쿠키) 결정 — **지금은 비-라우트 모드** (`locale` 로 분기 없이 단일 locale 하드 설정).

#### B.3 메시지 조직

```
src/messages/
├─ ko.json
```

네임스페이스 (최상위 키):

- `common` — 버튼 레이블, 공용 액션 ("저장", "취소", "삭제", "확인", ...)
- `nav` — 네비게이션 ("대시보드", "회의", "지시", "설정", ...)
- `auth` — 로그인·세션 관련 ("이메일", "비밀번호", "로그인", "로그아웃", ...)
- `meeting` — 회의 관련 (생성 폼 레이블, 상태 등)
- `dashboard` — 대시보드 ("지연 업무", "담당자", "기한", ...)
- `recording` — 녹음 UI ("녹음 시작", "일시정지", "종료", ...)
- `speakerMatch` — 화자 매칭
- `errors` — ApiErrorCode → 사용자 메시지 (**#5 `MESSAGES` 테이블 이관**)
- `validation` — zod 커스텀 메시지 (**#4 폼 검증 메시지**)

#### B.4 `<html lang>` 교정

- `src/app/layout.tsx` 의 `<html lang="en">` → **`<html lang="ko">`** 로 교정.
- 접근성 관점: 스크린리더가 한국어 발음 규칙 적용.

#### B.5 날짜/숫자/통화 포맷

- next-intl `useFormatter` → `Intl.DateTimeFormat` / `Intl.NumberFormat` 래퍼.
- 하드코딩 금지 (`new Date().toLocaleString('ko-KR')` 같은 직접 호출도 가급적 유틸로).
- 공용 유틸: `src/lib/i18n/format.ts` (`formatDate`, `formatTime`, `formatNumber`, `formatRelativeTime`).

#### B.6 다른 축과의 연결

- **#5 `errors/messages.ts` 점진 전환**:
  1. 지금: `MESSAGES: Record<ApiErrorCode, string>` 한글 하드코딩
  2. next-intl 도입 후: `MESSAGES` 를 `src/messages/ko.json#errors.<CODE>` 로 이관. `codeToMessage(code)` 는 내부에서 `useTranslations('errors')(code)` 호출.
  3. **함수 시그니처 유지** → 호출부 변경 없음.
- **#4 zod 커스텀 메시지**:
  - `createMeetingSchema` 등에서 메시지는 `t('validation.meeting.titleRequired')` 식 참조. zod `errorMap` 로 기본 i18n 적용.
  - 당장은 `src/messages/ko.json#validation` 에 필요한 키만 추가.

---

### C. 디자인 토큰 & 테마

#### C.1 토큰 체계 — Tailwind 4 `@theme` + CSS variables

- Tailwind 4 는 `@theme` 디렉티브로 CSS variable 토큰을 **공식 지원**. Tailwind 3 의 `tailwind.config.ts` 와 다름.
- `src/app/globals.css` 에 토큰 선언:

```css
@import "tailwindcss";

@theme {
  /* 색상 — 의미 토큰 (shadcn 식) */
  --color-bg: oklch(1 0 0);
  --color-fg: oklch(0.15 0 0);
  --color-muted: oklch(0.96 0 0);
  --color-muted-fg: oklch(0.45 0 0);
  --color-border: oklch(0.9 0 0);
  --color-primary: oklch(0.6 0.2 260);
  --color-primary-fg: oklch(0.99 0 0);
  --color-danger: oklch(0.6 0.22 25);
  --color-danger-fg: oklch(0.99 0 0);
  --color-success: oklch(0.6 0.18 150);
  --color-warning: oklch(0.75 0.17 85);

  /* 반경 */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-xl: 1rem;

  /* 그림자 (2-3 레벨) */
  --shadow-sm: 0 1px 2px rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);

  /* z-index 체계 */
  --z-dropdown: 10;
  --z-sticky: 20;
  --z-modal: 50;
  --z-popover: 60;
  --z-toast: 100;
  --z-tooltip: 110;

  /* 폰트 */
  --font-sans: 'Pretendard', system-ui, -apple-system, sans-serif;
}

/* 다크모드 — 구조만. 토글 UI 는 추후 */
[data-theme="dark"] {
  --color-bg: oklch(0.15 0 0);
  --color-fg: oklch(0.95 0 0);
  --color-muted: oklch(0.22 0 0);
  --color-muted-fg: oklch(0.65 0 0);
  --color-border: oklch(0.3 0 0);
  --color-primary: oklch(0.7 0.2 260);
  /* ... */
}
```

- **`oklch`** 사용: 지각 균일한 색 공간, 다크모드 변주 쉬움. (Tailwind 4·modern browsers 지원.)
- **의미 토큰** 만 노출: `bg` / `fg` / `primary` / `muted` / `border` / `danger` / `success` / `warning` (+ foreground variant). **9단계 scale (`primary-50`~`900`) 금지** — 의미 추상화 유지, 세분화 필요하면 variant 로.

#### C.2 다크모드

- **토큰 구조만 지원**. HTML 에 `data-theme="dark"` 를 세팅하는 토글 UI 는 **UI 안정화 후** 추가.
- 서버에서 `data-theme` 결정 (쿠키 기반) → FOUC 방지.

#### C.3 CVA (class-variance-authority)

- 컴포넌트 variants 선언:
  ```ts
  import { cva, type VariantProps } from 'class-variance-authority';
  const buttonVariants = cva(
    'inline-flex items-center justify-center rounded-md font-medium transition focus-visible:ring-2 focus-visible:ring-primary',
    {
      variants: {
        variant: {
          primary: 'bg-primary text-primary-fg hover:opacity-90',
          secondary: 'bg-muted text-fg hover:bg-border',
          danger: 'bg-danger text-danger-fg hover:opacity-90',
          ghost: 'bg-transparent hover:bg-muted',
        },
        size: {
          sm: 'h-8 px-3 text-sm',
          md: 'h-10 px-4',
          lg: 'h-12 px-6 text-lg',
        },
      },
      defaultVariants: { variant: 'primary', size: 'md' },
    }
  );
  ```
- 호출부: `<button className={buttonVariants({ variant: 'danger' })}>삭제</button>`
- Tailwind 클래스를 **의미 variant 로 숨김** → 호출부가 유틸 조립하지 않도록.

#### C.4 폰트

- **Pretendard** (한국어 가독성 표준, OFL).
- `next/font/local` 로 self-host (CDN 의존성 제거, FOUC 방지, 성능 유리).
- `src/app/fonts/pretendard/*.woff2` + `src/app/layout.tsx` 에서 `localFont` 로드.
- 폴백: `system-ui`, `-apple-system`, `sans-serif`.

#### C.5 반응형 breakpoint

- Tailwind 4 기본 (`sm` 640 / `md` 768 / `lg` 1024 / `xl` 1280 / `2xl` 1536) **유지**.
- 커스텀 breakpoint 추가 금지 (UI 디자인 확정 전까지).

#### C.6 금지 패턴

- 임의 hex/rgb inline (`style={{ color: '#ff0000' }}`) — **의미 토큰만 사용**
- 컴포넌트별 색상 하드코딩 (`bg-red-500` 등 Tailwind 기본 scale 직접 참조도 지양) — `bg-danger` 등 의미 토큰 사용
- 임의 spacing (`style={{ margin: '13px' }}`) — Tailwind scale 유지
- `tailwind.config.ts` 에 색상 직접 선언 (Tailwind 4 는 `@theme` 가 공식 경로)
- 다크모드 변주를 유틸로 (`dark:bg-black` 식) — **토큰 변주로 자동**. `dark:` 유틸 사용 지양.

---

## 3. 다른 축에 미치는 영향

| 이슈 | 내용 |
|---|---|
| [#4](https://github.com/timonPark/flowmeet-web/issues/4) 폼/검증 | zod `errorMap` + `src/messages/ko.json#validation` 로 메시지 i18n. 공용 폼 컴포넌트에 `aria-invalid`/`aria-describedby` 기본 제공 (이미 §4 결정에 반영). |
| [#5](https://github.com/timonPark/flowmeet-web/issues/5) 에러 처리 | `errors/messages.ts` 가 `useTranslations('errors')` 호출로 교체. 함수 시그니처 유지. |
| [#6](https://github.com/timonPark/flowmeet-web/issues/6) 테스트 | `@axe-core/playwright` 가 E2E 에 추가. `expectNoA11yViolations` fixture 공용 유틸. |
| [#8](https://github.com/timonPark/flowmeet-web/issues/8) 로깅/모니터링 | 외부 벤더 미도입. 접근성 위반은 `@axe-core/playwright` E2E 로만 잡고, production 리포트 수집 레이어는 추후. |
| [#9](https://github.com/timonPark/flowmeet-web/issues/9) 운영 규정 | PR 리뷰 체크에 "의미 토큰 사용·`dark:` 유틸 지양·`<html lang="ko">` 유지" 포함. |
| [#13](https://github.com/timonPark/flowmeet-web/issues/13) `nextjs16-reality-check` | Tailwind 4 `@theme` 는 Tailwind 3 와 다른 경로 — 가드에 포함. Opus/Sonnet 가 `tailwind.config.ts` 로 색상 선언하려 들면 틀림. |
| [#15](https://github.com/timonPark/flowmeet-web/issues/15) `app-router-layering` | `src/messages/`, `src/lib/i18n/`, `src/lib/ui/variants.ts`, `src/app/fonts/` 디렉터리 책임 포함. |
| [#20](https://github.com/timonPark/flowmeet-web/issues/20) 신규 축 묶음 스킬 | `a11y` / `i18n` / `design-tokens` 세 서브-스킬이 이 문서를 입력으로 사용. |

---

## 4. 열어둔 포인트 (defer)

| 항목 | 사유 | 어디로 |
|---|---|---|
| shadcn/ui 전체 CLI 도입 여부 | 지금은 Radix primitives + CVA 로 수제. 생산성 이슈 생기면 shadcn 선택 복사 하이브리드. | UI 구현 피드백 후 |
| 다국어 라우팅 전략 (`/[locale]/...` vs 쿠키) | 두 번째 로케일 추가 시 결정 | 로케일 확장 시 |
| axe 임계값 상향 (moderate 도 fail?) | UI 안정화 후 재검토 | 추후 운영 피드백 |
| 폰트 CDN vs self-host 성능 벤치 | 지금은 self-host 전제. 번들/LCP 측정 후 조정. | 성능 측정 단계 |
| 아이콘 세트 (lucide-react / heroicons) | UI 구현 단계 | UI 구현 피드백 |
| 다크모드 토글 UI | 토큰 구조만 지금. 토글은 UI 안정화 후. | UI 구현 피드백 |
| 색 대비 자동 검증 도구 | axe 가 상당수 커버. 추가 도구는 운영 피드백 후. | 추후 |

---

## 5. 완료 조건 체크리스트

- [x] a11y lint 레벨·헤드리스 라이브러리·런타임 검사 결정
- [x] i18n 라이브러리·로케일·메시지 네임스페이스 결정
- [x] `<html lang>` 교정 명시
- [x] 디자인 토큰 체계 (`@theme` + CSS variables + 의미 토큰) 결정
- [x] 다크모드 구조 수준 지원 명시
- [x] CVA 도입 결정
- [x] 폰트 (Pretendard) 결정
- [x] 금지 패턴 명시
- [x] 다른 축 영향 매핑
- [x] 열어둔 포인트 이관처
- [ ] CLAUDE.md 축약 반영 + 이 문서 링크 (이 PR)
- [ ] 실제 구현 (이 PR 범위 밖)

## 6. 참고

- [`docs/axis-decisions/form-validation.md`](./form-validation.md)
- [`docs/axis-decisions/error-ux.md`](./error-ux.md)
- [`docs/axis-decisions/test-strategy.md`](./test-strategy.md)
- Tailwind CSS 4 `@theme` 공식 가이드
- next-intl v3 공식 가이드 (App Router)
- Radix UI primitives 공식
- WCAG 2.1 AA 기준
- Pretendard (SIL OFL 1.1)
