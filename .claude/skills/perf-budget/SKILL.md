---
name: perf-budget
description: flowmeet-web 번들·성능 가드 — Next 16 `cacheComponents` 를 전제로 번들 사이즈·Core Web Vitals·클라이언트 번들 최소화 규칙. 새 라이브러리 도입·대형 컴포넌트 추가·성능 체감 저하 시 로드.
---

# Performance Budget — flowmeet-web

flowmeet-web 은 Next 16 `cacheComponents: true` + `partialPrefetching: true` 환경이라 **서버 랜더가 디폴트** 다. 따라서 성능의 핵심은:
1. **서버 랜더링 최대화** — "use client" 를 꼭 필요한 곳만
2. **클라이언트 번들 사이즈 통제** — 라이브러리 선택이 번들에 큰 영향
3. **캐시 적극 활용** — `'use cache'` 로 비싼 서버 fetch 재사용 (단, 인증 경계 주의)
4. **Core Web Vitals** — LCP, INP, CLS 모니터링

> **원칙**: 완벽한 "성능 숫자" 추구 아님. **사용자 체감** + **눈에 띄는 회귀 방지** 가 목표.

---

## 1. 번들 사이즈 예산

### 1.1. 라우트별 First Load JS 목표 (가이드, 하드 게이트 아님)

**측정 기준**: Next.js `npm run build` 출력의 "First Load JS" 는 **일반적으로 gzip 전 크기** (compressed size 가 아니라 bundled JS 크기 기준). gzip 측정은 별도 (예: `gzip -c <file> | wc -c` 또는 `@next/bundle-analyzer`).

| 라우트 성격 | First Load JS 목표 (gzip 전, Next build 출력) | gzip 환산 (추정 30-35%) |
|---|---|---|
| 공개 (로그인 등) | **≤ 300 KB** | ~100 KB |
| 인증 월드 (대시보드 등) | **≤ 500 KB** | ~170 KB |
| 녹음·화자매칭 (상호작용 집약) | **≤ 800 KB** | ~280 KB |

**중요 — 숫자는 baseline 생기면 재조정**:
- 위 목표는 **초기 가이드**. 실제 피처가 쌓이면 **baseline (최초 안정 버전)** 을 기준으로 **회귀 폭 ≤ 20%** 를 더 중요한 기준으로.
- **측정 환경 명시 필요**: 모바일 CPU slowdown 4x, Fast 3G, Lighthouse 등. 로컬 Chrome DevTools 결과만으론 production 체감 못 함.
- 하드 게이트 아님 — PR 리뷰에서 사유 설명 가능하면 통과.

### 1.2. 전역 shared chunk

- Next 가 자동 split. 특별 설정 불필요.
- 다만 **모든 라우트가 끌어쓰는 거대 라이브러리** 추가는 PR 리뷰에서 심의. 예: moment.js (`~70KB`), lodash 전체 (`~70KB`).

### 1.3. 측정 도구

```bash
# 빌드 분석
npm run build  # Next 가 route 별 First Load JS 출력
ANALYZE=true npm run build  # @next/bundle-analyzer (별도 설치)
```

- `@next/bundle-analyzer` 는 **필요할 때만** 설치. 상시 설치는 과함.

---

## 2. "use client" 최소화 — 서버 랜더 디폴트

- Server Component 가 기본. `"use client"` 는 **상태·이벤트·브라우저 API·훅이 필요한 트리의 꼭대기만** ([#15 app-router-layering](../app-router-layering/SKILL.md) §3).
- 꼭대기에 두어 **자식 Server Component 는 서버 랜더 유지** (`children` prop 패턴).

### 금지

- 트리 전체를 `"use client"` 로 감싸기 — 모든 자식이 클라 번들에 포함.
- 불필요한 `"use client"` (훅 없는데 추가) — 의미 없이 클라 번들 증가.

---

## 3. 라이브러리 선택 가이드

### 3.1. 번들 크기 체크 (`bundlephobia.com`)

PR 에 신규 디펜던시 추가 시:
- gzipped size 를 PR 본문에 명시
- 유사 기능 대체 라이브러리와 비교

### 3.2. Phase 1 결정 라이브러리 기준치 (대략)

| 라이브러리 | gzip (약) | 비고 |
|---|---|---|
| `react` + `react-dom` 19 | ~45 KB | 필수 |
| `next` 16 runtime | ~30 KB (route 공유) | 필수 |
| `@tanstack/react-query` v5 | ~13 KB | #2 결정 |
| `zustand` v5 | ~1 KB | #3 결정 |
| `react-hook-form` v7 | ~9 KB | #4 결정 |
| `zod` | ~13 KB | #4 결정 |
| `react-error-boundary` v4 | ~3 KB | #5 결정 |
| `@axe-core/playwright` | dev-only | 성능 영향 없음 |
| `next-intl` v3 | ~10 KB + messages 번들 | #7 결정 |
| `@radix-ui/react-*` (각 primitive) | 5-15 KB 씩 | 쓰는 primitive 만 import |
| `sonner` (토스트) | ~5 KB | #5 |
| `class-variance-authority` | ~2 KB | #7 |
| **합계 (추정)** | **~130-180 KB** | 라우트별 상이, Radix 는 사용분만 |

### 3.3. 금지 라이브러리 (대안 있음)

- `moment.js` → `Intl.DateTimeFormat` (native, 0 KB) 또는 `date-fns` (tree-shakable)
- `lodash` 전체 → ES built-in (`Array.prototype.*`, `Object.*`) 또는 `lodash-es` 개별 import (`import debounce from 'lodash-es/debounce'`)
- `jQuery` → React native API
- `axios` → native `fetch` + 자체 wrapper (#17 `data-fetching-guard` 가 담당)

### 3.4. 신규 라이브러리 심의 기준

1. **용도** — 꼭 라이브러리여야 하나? 자체 구현 50줄로 되나?
2. **번들 크기** — gzipped 10KB 넘으면 심의.
3. **메인테이너·마지막 release** — abandoned 금지.
4. **라이선스** — MIT / Apache 2.0 / ISC 안전.
5. **Tree-shaking 지원** — ESM 지원, side-effect-free.

---

## 4. 코드 분할 (`next/dynamic`)

**중요**: `next/dynamic({ ssr: false })` 는 **Client Component 안에서만** 호출 가능. Server Component 에서 호출하면 런타임 에러.

```tsx
// src/features/recording/RecordingSessionLoader.tsx
'use client';  // 필수 — dynamic({ ssr: false }) 쓰려면
import dynamic from 'next/dynamic';

const RecordingSession = dynamic(() => import('./RecordingSession'), {
  loading: () => <div>준비 중...</div>,
  ssr: false,  // MediaRecorder 등 브라우저 API 전용
});

export default RecordingSession;
```

```tsx
// 호출 (Server Component 또는 Client Component)
import RecordingSessionLoader from '@/features/recording/RecordingSessionLoader';

export default function Page() {
  return <RecordingSessionLoader />;
}
```

- Server Component 에서 바로 `dynamic({ ssr: false })` 호출 **금지** → `"use client"` 모듈로 wrap.
- `ssr: true` (기본) 는 어디서나 가능.
- 적용 대상:
  - MediaRecorder·IndexedDB 로직 (녹음) — `ssr: false`
  - 차트 라이브러리 (대시보드) — 서버 렌더 가능하면 `ssr: true`
  - Rich text editor (지시 작성, 생기면) — `ssr: false`
- 무조건 적용 아님 — **번들 실측 후 큰 chunk 만**.

---

## 5. 이미지 최적화

- `next/image` 사용 — `remotePatterns` 화이트리스트 ([#19 web-security](../web-security/SKILL.md)).
- `priority` 는 **above-the-fold 중요 이미지만** — 모든 이미지에 쓰면 LCP 오히려 악화.
- 적절한 `sizes` 지정 — 반응형 이미지 sauropod.
- 외부 이미지는 **백엔드가 썸네일 생성** → 작은 사이즈 전달 (풀 해상도 받아 CSS 리사이즈 금지).

---

## 6. 캐시 전략 (성능 관점)

- **`'use cache'` + `cacheLife('minutes')` / `('hours')`** 로 비싼 집계 쿼리 재사용 ([#17 data-fetching-guard](../data-fetching-guard/SKILL.md) §6).
- 단, 인증 경계 주의 (사용자 간 캐시 오염 방지).
- **React Query `staleTime: 30_000`** 기본 — 불필요한 재요청 억제 (#5).
- **`refetchOnWindowFocus: false`** 기본 — 폴링은 명시적 `refetchInterval`.

---

## 7. Core Web Vitals

### 7.1. 지표 목표

| 지표 | 목표 | 비고 |
|---|---|---|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | 메인 콘텐츠 렌더까지 |
| **INP** (Interaction to Next Paint) | ≤ 200ms | 상호작용 반응 |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | 레이아웃 안정성 |

### 7.2. LCP 개선

- 이미지 preload (`priority`)
- 폰트 preload (self-host + `display: swap`)
- Server Component 로 핵심 콘텐츠 서버 랜더

### 7.3. INP 개선

- 큰 리스트 가상화 (예: `@tanstack/react-virtual`)
- 무거운 이벤트 핸들러를 `startTransition` 로 감쌈 (React 19)
- 디바운스 (`useDeferredValue`, debounce)
- `"use client"` 트리 최소화 (JS 실행량 줄임)

### 7.4. CLS 개선

- 이미지에 `width`/`height` 지정 — reserve space
- 폰트 fallback 로드 시 **layout shift 방지** (`font-display: swap` + 유사 metric 폴백)
- Skeleton loader 사용 (고정 크기)
- 광고·동적 삽입 영역에 min-height 미리 지정

---

## 8. 측정·모니터링

### 8.1. 로컬 측정

```bash
npm run build          # route 별 First Load JS
npm start              # production 모드 실행
# Chrome DevTools → Lighthouse 또는 Performance 탭
```

### 8.2. 프로덕션 모니터링

- **Vercel Analytics** 또는 **web-vitals 라이브러리** → 자체 백엔드 수집 (#8 과 정합, 외부 벤더 미도입)
- `reportWebVitals` 콜백 ([Next 공식](https://nextjs.org/docs)) → `reportError` 와 같이 자체 수집 레이어 생기면 전송
- 지금 당장은 **로컬 측정만**, production 모니터링은 배포 플랫폼 결정 후.

---

## 9. 금지 패턴

- 측정 없이 "성능 걱정돼서" 코드 복잡화 (premature optimization)
- 커버리지처럼 **번들 사이즈 하드 게이트** — 상황에 따라 유연, PR 리뷰 가이드
- 모든 컴포넌트에 `React.memo` / `useMemo` — overhead 가 더 큼. 실측 후 bottleneck 만.
- 모든 이미지에 `priority` — LCP 악화
- `'use cache'` 를 **아무 함수에나** — 인증 경계 사고 가능 (#17)
- `useEffect` 로 데이터 fetch — RQ 또는 RSC fetch 사용

---

## 10. 체크리스트

- [ ] 새 라이브러리 추가 시 **gzipped 크기** 가 PR 본문에 명시됐는가? 유사 대안과 비교?
- [ ] 라우트 First Load JS 가 **목표치 (§1.1)** 안인가? 초과 시 사유 명시?
- [ ] `"use client"` 가 **꼭대기만** 쓰이는가? 불필요한 "use client" 없음?
- [ ] 큰 상호작용 컴포넌트 (녹음·차트·editor) 가 `next/dynamic` 으로 분리됐는가?
- [ ] 이미지에 `width`/`height` 지정 (CLS 방지)? above-the-fold 만 `priority`?
- [ ] 비싼 서버 쿼리에 `'use cache'` + `cacheLife` 적용됐는가? 인증 경계 안전?
- [ ] `moment.js` / `lodash 전체` / `axios` 등 **금지 라이브러리** 사용 없음?
- [ ] 성능 최적화 전에 **실측 (Lighthouse / DevTools)** 했는가? premature optimization 없음?

---

## 11. 참고

- 결정 축: [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md) (캐시 전략), [#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) §4 (성능 예산 열어둠), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md) (외부 모니터링 미도입)
- 관련 스킬: [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md), [`app-router-layering`](../app-router-layering/SKILL.md) §3 (Server/Client 경계), [`data-fetching-guard`](../data-fetching-guard/SKILL.md) §6 (`'use cache'`), [`design-tokens`](../design-tokens/SKILL.md)
- Core Web Vitals (web.dev), bundlephobia.com, Next.js build output
