---
name: design-tokens
description: flowmeet-web 디자인 토큰 가드 — Tailwind 4 `@theme` + CSS variables, oklch 의미 토큰, 다크모드 구조, CVA variants, Pretendard 폰트. 스타일 코드·새 컴포넌트·테마 변경 시 로드. #7 결정의 디자인 토큰 축 실무 적용.
---

# Design Tokens — flowmeet-web

[#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) §C 결정을 코드로. **Tailwind 4 는 `@theme` 가 공식 토큰 경로** — Tailwind 3 의 `tailwind.config.ts` 와 다름.

---

## 1. `@theme` 선언 — `src/app/globals.css`

```css
@import "tailwindcss";

@theme {
  /* 색상 — 의미 토큰 (shadcn 식). oklch 사용 (지각 균일) */
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

  /* 그림자 (3 레벨) */
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

/* 다크모드 — 토큰 구조만. 토글 UI 는 추후. */
[data-theme="dark"] {
  --color-bg: oklch(0.15 0 0);
  --color-fg: oklch(0.95 0 0);
  --color-muted: oklch(0.22 0 0);
  --color-muted-fg: oklch(0.65 0 0);
  --color-border: oklch(0.3 0 0);
  --color-primary: oklch(0.7 0.2 260);
  --color-primary-fg: oklch(0.1 0 0);
  --color-danger: oklch(0.65 0.22 25);
  --color-danger-fg: oklch(0.1 0 0);
  /* success/warning 등도 유사 패턴 */
}
```

### 1.1. 왜 oklch?

- 지각 균일 (perceptually uniform) 색 공간 — 밝기 조정이 자연스러움.
- 다크모드 변주가 깔끔 (lightness 조정만으로 톤 유지).
- modern browser 지원 양호 (Safari 15.4+, Chrome 111+).

### 1.2. 왜 의미 토큰?

- `primary`, `danger`, `muted` 등 **의미 중심** — 디자인 의도가 변해도 토큰만 바꾸면 됨.
- `blue-500`, `red-600` 식 색상-이름 토큰 금지 — 컴포넌트가 **"파란색"** 에 묶여 재브랜딩 시 전수 변경.

---

## 2. 사용 패턴

### 2.1. Tailwind 유틸로 참조

```tsx
<button className="bg-primary text-primary-fg rounded-md px-4 py-2 shadow-md">
  저장
</button>
```

- `bg-primary` → `background-color: var(--color-primary)` 자동.
- Tailwind 4 가 `@theme` 선언을 자동 utility 로 변환.

### 2.2. CVA variants (의미 추상화)

```ts
// src/lib/ui/variants.ts (a11y 스킬 §3 중복)
export const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md font-medium focus-visible:ring-2 focus-visible:ring-primary',
  {
    variants: {
      variant: {
        primary: 'bg-primary text-primary-fg',
        secondary: 'bg-muted text-fg',
        danger: 'bg-danger text-danger-fg',
      },
    },
  }
);
```

- 호출부가 `className="bg-primary ..."` 를 직접 조합하지 않음.

### 2.3. 다크모드

- `document.documentElement.setAttribute('data-theme', 'dark')` 로 전환 (토글 UI 는 추후).
- 컴포넌트 코드는 변경 없음 — 토큰이 자동 변주.

---

## 3. 폰트 — Pretendard

### 3.1. self-host (`next/font/local`)

```ts
// src/app/fonts/pretendard.ts
import localFont from 'next/font/local';

export const pretendard = localFont({
  src: [
    { path: './pretendard/Pretendard-Regular.woff2', weight: '400', style: 'normal' },
    { path: './pretendard/Pretendard-Medium.woff2', weight: '500', style: 'normal' },
    { path: './pretendard/Pretendard-SemiBold.woff2', weight: '600', style: 'normal' },
    { path: './pretendard/Pretendard-Bold.woff2', weight: '700', style: 'normal' },
  ],
  display: 'swap',
  variable: '--font-pretendard',
});
```

### 3.2. layout 적용

```tsx
// src/app/layout.tsx
import { pretendard } from './fonts/pretendard';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko" className={pretendard.variable}>
      <body style={{ fontFamily: 'var(--font-pretendard), system-ui, sans-serif' }}>
        {children}
      </body>
    </html>
  );
}
```

- `next/font/local` 가 build time 최적화 (subset, preload).
- CDN 의존 제거 → FOUC 방지, 성능 유리.
- 폴백: `system-ui`, `-apple-system`, `sans-serif`.

### 3.3. 라이선스

- Pretendard 는 **SIL OFL 1.1** — 자유 재배포 가능.
- `src/app/fonts/pretendard/LICENSE` 에 라이선스 파일 포함.

---

## 4. 금지 패턴

- **임의 hex / rgb inline** — `style={{ color: '#ff0000' }}`. 의미 토큰 (`text-danger`) 만.
- **Tailwind 기본 색상 scale 직접 참조** — `bg-red-500`, `text-blue-600` 등. 의미 토큰으로.
- **`dark:` 유틸로 다크모드 변주** — `dark:bg-black`. 토큰 자동 변주이므로 불필요. drift 소스.
- **`tailwind.config.ts` 에 색상 선언** — Tailwind 4 는 `@theme` 가 공식 경로. config 파일의 colors는 **쓰지 않음** (레거시 호환은 되지만 drift).
- **임의 spacing** — `style={{ margin: '13px' }}`. Tailwind scale (`m-3` = 12px, `m-4` = 16px 등) 유지.
- **9단계 색상 scale** (`primary-50` ~ `primary-900`) — 의미 추상화 깨짐. variant 로.
- **CSS-in-JS (styled-components, emotion)** — Tailwind 와 중복 + 번들 비용. 지금 미도입.

---

## 5. 반응형

- Tailwind 4 기본 breakpoint **유지**: `sm` 640 / `md` 768 / `lg` 1024 / `xl` 1280 / `2xl` 1536.
- 커스텀 breakpoint **추가 금지** (UI 디자인 확정 전까지).
- 모바일 우선 (`<md:...` 먼저, `md:...` 뒤).

---

## 6. 아이콘·이미지

- 아이콘 세트는 **UI 구현 단계에서 결정** ([#7](../../../docs/axis-decisions/a11y-i18n-tokens.md) §4 열어둠). 현재 lucide-react / heroicons 등 후보.
- 결정 전까지는 **인라인 SVG** 사용 가능. 결정 후 교체.
- `next/image` 로 이미지 최적화. `remotePatterns` 화이트리스트 ([#19 web-security](../web-security/SKILL.md) §7.1).

---

## 7. 체크리스트

- [ ] 색상이 **의미 토큰** (`bg-primary`, `text-danger`) 인가? Tailwind 기본 scale (`bg-red-500`) 직접 참조 없음?
- [ ] 임의 hex / rgb inline 없는가? `style={{ color: '#...' }}` 금지?
- [ ] `dark:` 유틸 사용 없는가? 다크모드는 토큰 자동 변주?
- [ ] `tailwind.config.ts` 에 색상 선언 없는가? `@theme` 가 유일 경로?
- [ ] spacing 이 Tailwind scale 유지 (`m-3`, `p-4` 등)인가? 임의 px 없음?
- [ ] 폰트가 Pretendard self-host (`next/font/local`) 인가? CDN 의존 없음?
- [ ] 반응형이 Tailwind 기본 breakpoint 유지인가? 커스텀 추가 없음?

---

## 8. 참고

- 결정 축: [#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) §C
- 관련 스킬: [`a11y`](../a11y/SKILL.md) §3 (CVA variants), [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md) §2.6 (Tailwind 4 `@theme`)
- Tailwind CSS 4 공식 docs, Pretendard (SIL OFL 1.1), oklch (CSS Color Module Level 4)
