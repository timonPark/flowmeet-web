---
title: 테스트 전략
date: 2026-10-16
author: kindcorder@gmail.com
status: DECIDED
issue: "#6"
scope: flowmeet-web 유닛/통합/E2E 러너·네트워크 모킹·범위·운영 원칙
depends_on:
  - "#2 데이터 페칭/캐시 전략"
  - "#3 상태 관리 전략"
  - "#4 폼/검증 전략"
  - "#5 에러 처리(error-ux) 전략"
---

# 테스트 전략

flowmeet-web 은 Next.js 16 (`cacheComponents` + `partialPrefetching`) + React 19 환경이다. Next 16 는 `next/server/testing` (라우팅/미들웨어 설정 테스트) 과 `next/experimental/testmode` (Playwright 에서 **서버 랜더 중 fetch 가로채기**) 를 제공한다. 이 축은 **"어떤 러너·어떤 범위·어떤 모킹"** 을 쓸지와, **커버리지 하드 게이트 없이 변경 지향 테스트** 원칙을 확정한다.

> 이 문서는 `web-testing` 스킬([#18](https://github.com/timonPark/flowmeet-web/issues/18)) 의 입력. 공용 유틸(`renderWithProviders`, `resetStores`, MSW handlers) 네이밍이 스킬 작성 시 기준이 된다.

---

## 1. 결정 요약

| 레벨 | 도구 |
|---|---|
| 유닛 / 통합 | **Vitest** + `@vitejs/plugin-react` |
| 컴포넌트 | **@testing-library/react** + **@testing-library/user-event** |
| 네트워크 모킹 | **MSW v2** (유닛·E2E 공용 handlers) |
| E2E | **Playwright** + Next 16 `experimental/testmode` (서버 fetch 가로채기) |
| 코로케이션 | 유닛·컴포넌트는 **코로케이트** (`foo.ts` + `foo.test.ts`) |
| E2E 위치 | 루트 **`e2e/`** |
| 커버리지 게이트 | **없음** (informational). "변경 모듈에 테스트 있는지" PR 리뷰로 관리 |

**원칙**: *행동을 테스트한다. 구현 세부·내부 state inspection·shallow 금지. 플레이크는 bug — 재시도로 숨기지 않는다.*

---

## 2. 결정 상세

### 2.1. 러너: Vitest

- 선정 사유:
  - Next 16 + Turbopack 시대 설정과 궁합. Jest 는 ESM/SWC/transform 조합이 Next 16 레시피와 흔들림.
  - API 가 Jest 와 95%+ 호환 — 전환 비용 낮음.
  - 병렬 실행·HMR·speed 이점 체감.
- 설치/설정 (`vitest.config.ts`): Next 16 와 공존하도록 `plugins: [react()]` + `test.environment: 'jsdom'` + alias `@/* → src/*` + setup `vitest.setup.ts` (RTL cleanup, MSW `server.listen`/`close`).
- Jest 금지. 혼용 금지.

### 2.2. 컴포넌트: @testing-library/react + user-event

- 쿼리 우선순위: `getByRole` > `getByLabelText` > `getByPlaceholderText` > `getByText`. `getByTestId` 는 **최후 수단**, PR 리뷰에서 지양.
- 사용자 상호작용은 `userEvent` 로. `fireEvent` 는 특수 상황 (focus 이벤트 등) 에만.
- 비동기 어서션: `findBy*` 또는 `waitFor(() => expect(...))` — `waitForElementToBeRemoved` 등 명확한 util 사용.

### 2.3. 네트워크 모킹: MSW v2

- Handlers 는 유닛(Node interceptor) 과 E2E(service worker) **같은 소스 공유** — drift 방지.
- 디렉터리: `src/test/mocks/{handlers.ts, server.ts}`.
- E2E 는 Next 16 `experimental/testmode` 로 서버 fetch 를 reader 에 꽂아 가로챔 (RSC 서버 로딩도 모킹 가능).
- 함수-레벨 fetch mock (`vi.spyOn(global, 'fetch')`) 지양 — MSW 로 통일.

### 2.4. E2E: Playwright

- 선정 사유: Trace viewer, 병렬, cross-browser, Next 16 testmode 공식 연계.
- Cypress 미채택: Next 16 서버 사이드 가로채기 궁합 열세, CI 비용·러닝커브 큼.
- 초기 스코프 (3-5개 스모크):
  - `login.spec.ts` — 이메일/비번 로그인 → 대시보드
  - `meeting-create.spec.ts` — 회의 생성 폼 → 성공 플로우
  - `dashboard-delayed.spec.ts` — 팀 대시보드 지연업무 리스트 (TRD §8)
  - `calendar-nav.spec.ts` — 캘린더 일/주 전환 (TRD §4.2)
  - `recording-smoke.spec.ts` — 녹음 세션 시작/종료 UI (오디오 자체는 mock, TRD §4.1)
- 피처 추가될 때 함께 테스트 추가. 처음부터 커버리지 숫자 쫓지 않음.

### 2.5. 디렉터리 구조

```
src/
├─ app/
├─ lib/
│   ├─ validation/schemas/meeting.ts
│   ├─ validation/schemas/meeting.test.ts        # 코로케이트
│   └─ errors/messages.ts
│       └─ messages.test.ts
├─ features/
│   └─ meeting/
│       ├─ CreateMeetingForm.tsx
│       └─ CreateMeetingForm.test.tsx
└─ test/
    ├─ mocks/
    │   ├─ handlers.ts         # MSW handlers (유닛·E2E 공용)
    │   └─ server.ts           # setupServer(...)  (Vitest 전용)
    ├─ factories/              # test data factories (meeting, user, directive, ...)
    ├─ renderWithProviders.tsx # QueryClient + AuthProvider 등 래퍼
    └─ resetStores.ts          # Zustand 전체 초기화

e2e/
├─ login.spec.ts
├─ meeting-create.spec.ts
├─ dashboard-delayed.spec.ts
├─ calendar-nav.spec.ts
├─ recording-smoke.spec.ts
└─ fixtures.ts                 # Playwright + testmode 설정, MSW handlers import
```

### 2.6. 공용 래퍼 / 리셋 유틸

- **`renderWithProviders(ui, { queryClient?, authState?, ... })`** — RTL 표준 래퍼. **테스트마다 fresh `QueryClient`** 생성 (shared cache 유출 금지).
- **`resetStores()`** — Zustand 모든 slice 초기 상태로. `beforeEach(resetStores)` 로 매 테스트 격리.
- MSW 보일러:
  ```ts
  // vitest.setup.ts
  import { server } from './src/test/mocks/server';
  beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
  afterEach(() => server.resetHandlers());
  afterAll(() => server.close());
  ```
- `onUnhandledRequest: 'error'` — 핸들러 없는 요청은 테스트 실패로. (실수로 외부 서비스 때리는 것 방지)

### 2.7. 테스트 범위 매트릭스

| 레벨 | 무엇을 | 어디서 | 예시 |
|---|---|---|---|
| **유닛 (Vitest)** | 순수 함수·zod 스키마·store slice 액션·에러 매핑·fetch wrapper util | `src/lib/**/*.test.ts` | `codeToMessage.test.ts`, `createMeetingSchema.test.ts`, `recordingStore.test.ts` |
| **훅 (Vitest + RTL)** | custom hooks, RQ hook wrapper | 코로케이트 `*.test.tsx` | `useAuth.test.tsx`, `useRecordingSession.test.tsx` |
| **컴포넌트 (Vitest + RTL + MSW)** | 핵심 폼·에러 바운더리·대시보드 카드·화자매칭 UI | 코로케이트 `*.test.tsx` | `CreateMeetingForm.test.tsx`, `SpeakerMatchPanel.test.tsx` |
| **E2E (Playwright + MSW + testmode)** | 사용자 플로우 스모크 | `e2e/*.spec.ts` | 위 §2.4 5개 |

**지금 당장 테스트 하지 않는 것**:
- RSC 서버 컴포넌트 **직접 유닛 테스트** (프레임워크 한계) → E2E 로 커버
- Next 프레임워크/HOC 자체
- 트리비얼 pass-through 컴포넌트 (`<Button>` 등)
- 외부 라이브러리 재래핑 수준

### 2.8. Server Action / RSC 전략

- **Server Action**: 비즈니스 로직을 **pure helper 로 추출** → helper 를 Vitest 로. Server Action 함수 자체는 helper + `serverClient` 조립이라 **얇게 유지**.
- **RSC**: 유닛 테스트 포기. **Playwright + testmode** 로 서버 fetch 가로채며 전체 플로우 검증.
- **fetch wrapper** (`serverClient`/`browserClient`): 401 refresh 재시도·에러 매핑·headers 조립 모두 유닛 (fetch 자체를 mock, MSW).

### 2.9. 데이터 팩토리

- `src/test/factories/*.ts` **수제 함수** (`makeMeeting(overrides)`, `makeUser(overrides)` 등).
- 외부 라이브러리 (fishery 등) 초기 도입 안 함. 간단히 시작, 복잡해지면 재검토.

### 2.10. 작성 원칙

1. **행동 테스트** — "`setXxx` 가 호출됐는가" 가 아니라 "사용자가 보는 결과가 바뀌는가". `screen.getByRole('button', { name: '저장' })` 등 **의미 중심 쿼리**.
2. **구현 세부 금지** — Enzyme-식 shallow 금지, 내부 state inspection 금지, 라이브러리 내부 함수 spy 금지.
3. **네트워크는 MSW** — 함수-레벨 fetch mock 지양. 변경 영향 범위 최소화.
4. **플레이크는 bug** — 비결정적 테스트는 수정 대상. 재시도 설정 (`retry: 1`) 로 숨기지 않는다. E2E Playwright `retries: 0` (local) / `retries: 1` (CI) 상한.
5. **독립성** — `beforeEach(resetStores + resetHandlers)` 로 상태 공유 금지. 순서 의존 테스트 금지.
6. **왜 테스트 하는지 명확히** — 안 쓰는 테스트는 삭제. "일단 넣어둬" 금지.

### 2.11. CI 운영

- **PR 당 유닛 + 컴포넌트** 전부, **E2E 는 스모크만**.
- 커버리지 **하드 게이트 없음** — "변경된 모듈에 테스트 있는가" PR 리뷰 체크로 관리. 리포트는 informational.
- E2E 는 Playwright `trace: 'retain-on-failure'` 로 실패시 trace 보존.
- 실제 `.github/workflows/*.yml` 설정은 **운영 규정 (#9)** 결정 후 별도 chore PR.

---

## 3. 다른 축에 미치는 영향

| 이슈 | 내용 |
|---|---|
| [#2](https://github.com/timonPark/flowmeet-web/issues/2) 데이터 페칭 | `serverClient`/`browserClient` 유닛 테스트 레시피 (401 refresh·에러 매핑) 가 이 축에서 정립. |
| [#3](https://github.com/timonPark/flowmeet-web/issues/3) 상태 관리 | `resetStores()` 유틸 요구. Zustand slice 액션 테스트는 유닛. |
| [#4](https://github.com/timonPark/flowmeet-web/issues/4) 폼/검증 | zod 스키마는 유닛, 폼 컴포넌트는 RTL. Server Action 은 pure helper 추출 후 유닛. |
| [#5](https://github.com/timonPark/flowmeet-web/issues/5) 에러 처리 | `codeToMessage`·`handleApiError`·`ErrorBoundary` 모두 유닛/컴포넌트. `reportError` 는 spy. |
| [#9](https://github.com/timonPark/flowmeet-web/issues/9) 운영 규정 | PR 리뷰 체크 ("변경 모듈 테스트 있는지") 를 체크리스트에 포함. CI workflow 는 여기서 합의. |
| [#13](https://github.com/timonPark/flowmeet-web/issues/13) `nextjs16-reality-check` | `experimental/testmode` 를 Playwright 와 연계하는 패턴을 가드에 포함. |
| [#15](https://github.com/timonPark/flowmeet-web/issues/15) `app-router-layering` | `src/test/` 디렉터리 책임, `e2e/` 위치, 테스트 네이밍 포함. |
| [#18](https://github.com/timonPark/flowmeet-web/issues/18) `web-testing` 스킬 | 이 문서가 주 입력. 스킬 체크 포인트: `getByRole` 우선, MSW 통일, `renderWithProviders` 사용, 플레이크 금지, 코로케이션 지키기. |

---

## 4. 열어둔 포인트 (defer)

| 항목 | 사유 | 어디로 |
|---|---|---|
| CI workflow (`.github/workflows/`) | 운영 규정 (#9) 과 묶는 게 깔끔 | #9 / 별도 chore PR |
| Visual regression (Playwright screenshot) | UI 안정화 전까지 보류 | 추후 신규 이슈 |
| Mutation / property testing | 지금 범위 밖. zod 스키마 property 테스트는 후보. | 피처 안정화 후 재검토 |
| 커버리지 리포트 호스팅 (codecov 등) | 하드 게이트 없으니 지금 불필요 | 추후 운영 피드백 보고 |
| Server Action E2E 와 unit 중복 룰 | pure helper 추출 전략이 1차 완충. 실제 중복 생기면 보강. | 구현 단계 피드백 |

---

## 5. 완료 조건 체크리스트

- [x] 러너·컴포넌트·네트워크·E2E 도구 결정
- [x] 범위 매트릭스 (유닛/훅/컴포넌트/E2E)
- [x] 디렉터리·네이밍·코로케이션 규칙
- [x] 공용 래퍼·리셋 유틸 명시
- [x] Server Action / RSC 전략
- [x] 작성 원칙 (행동 테스트·플레이크 금지 등)
- [x] CI 운영 원칙 (하드 게이트 없음)
- [x] 다른 축 영향 매핑
- [x] 열어둔 포인트 이관처
- [ ] CLAUDE.md 축약 반영 + 이 문서 링크 (이 PR)
- [ ] 실제 설정·테스트 작성 (이 PR 범위 밖)

## 6. 참고

- [`docs/axis-decisions/data-fetching.md`](./data-fetching.md)
- [`docs/axis-decisions/state-management.md`](./state-management.md)
- [`docs/axis-decisions/form-validation.md`](./form-validation.md)
- [`docs/axis-decisions/error-ux.md`](./error-ux.md)
- Next 16 testing primitives: `node_modules/next/dist/experimental/testing/server/index.d.ts`, `node_modules/next/dist/experimental/testmode/fetch.d.ts`
- Vitest, Testing Library, MSW v2, Playwright 공식 가이드
