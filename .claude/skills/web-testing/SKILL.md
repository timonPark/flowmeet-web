---
name: web-testing
description: flowmeet-web 테스트 코드 작성·리뷰 가드 — Vitest 유닛/컴포넌트, MSW 네트워크 모킹, RHF/Zustand/RQ 테스트 패턴, Playwright E2E + Next 16 testmode. 테스트 파일을 쓰거나 구현 모듈과 함께 테스트 전략을 결정할 때 로드.
---

# Web Testing — flowmeet-web

[#6 test-strategy](../../../docs/axis-decisions/test-strategy.md) 결정을 **코드로 적용할 때** 쓰는 가드. "뭘 쓸지" 는 결정 축이, "그래서 테스트 코드를 어떻게 쓰나" 는 이 스킬이.

> **핵심 원칙** (#6 재언급): 행동을 테스트한다. `getByTestId` 는 최후수단. 커버리지 % 하드 게이트 없음. 플레이크는 bug — 재시도로 숨기지 않는다. 코로케이션.

---

## 1. 디렉터리 (고정)

```
src/
├─ features/
│  └─ meeting/
│     ├─ CreateMeetingForm.tsx
│     ├─ CreateMeetingForm.test.tsx       # 코로케이트
│     ├─ createMeetingLogic.ts            # pure helper (Server Action 비즈니스 로직)
│     ├─ createMeetingLogic.test.ts       # Server Action 테스트 = helper 테스트
│     ├─ actions.ts                       # 얇은 Server Action wrapper
│     ├─ queries.ts                       # 클라 RQ queries
│     └─ queries.test.ts                  # queryKey·invalidation 로직 테스트
├─ lib/
│  ├─ validation/schemas/meeting.ts
│  ├─ validation/schemas/meeting.test.ts  # zod 스키마 유닛
│  ├─ errors/messages.ts
│  ├─ errors/messages.test.ts
│  └─ ...
└─ test/
   ├─ mocks/
   │  ├─ handlers.ts                      # MSW handlers (유닛+E2E 공용)
   │  ├─ server.ts                        # Vitest 전용 setupServer
   │  └─ domain/                          # 도메인별 handler 분리 (meeting/dashboard/auth…)
   │     ├─ meeting.ts
   │     └─ ...
   ├─ factories/                          # 테스트 데이터 생성
   │  ├─ meeting.ts                       # makeMeeting(overrides?)
   │  └─ user.ts
   ├─ renderWithProviders.tsx             # RTL render 래퍼 (QueryClient + AuthProvider)
   └─ resetStores.ts                      # Zustand 모든 slice 초기화

e2e/
├─ login.spec.ts
├─ meeting-create.spec.ts
├─ dashboard-delayed.spec.ts
├─ calendar-nav.spec.ts
├─ recording-smoke.spec.ts
├─ fixtures.ts                            # Playwright fixture + a11y 유틸
└─ msw-worker.ts                          # Playwright 서비스 워커 설정
```

---

## 2. Vitest 설정

### 2.1. `vitest.config.ts` 템플릿

```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') },
  },
  test: {
    environment: 'jsdom',
    setupFiles: ['./vitest.setup.ts'],
    globals: true,          // describe/it/expect 전역 — 코로케이션 import 지루함 완화
    css: false,             // CSS 파싱 생략 (속도)
    exclude: ['e2e/**', 'node_modules/**'],  // Playwright 는 분리
  },
});
```

### 2.2. `vitest.setup.ts`

```ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach, beforeAll, afterAll } from 'vitest';
import { server } from './src/test/mocks/server';
import { resetStores } from './src/test/resetStores';

// MSW: 핸들러 없는 요청은 실패 (실수 외부 요청 차단)
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));

afterEach(() => {
  cleanup();              // RTL — 렌더된 DOM 제거
  server.resetHandlers(); // 테스트별 핸들러 override 격리
  resetStores();          // Zustand 초기화
});

afterAll(() => server.close());
```

### 2.3. `package.json` 스크립트

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test"
  }
}
```

- `tsc --noEmit` 스크립트도 같이 정비 (#6 CI 요구):
  ```json
  "typecheck": "tsc --noEmit"
  ```

---

## 3. MSW 핸들러 — 유닛·E2E 공용

### 3.1. 핸들러 구조

```ts
// src/test/mocks/handlers.ts
import { meetingHandlers } from './domain/meeting';
import { authHandlers } from './domain/auth';
import { dashboardHandlers } from './domain/dashboard';

export const handlers = [
  ...meetingHandlers,
  ...authHandlers,
  ...dashboardHandlers,
];
```

```ts
// src/test/mocks/domain/meeting.ts
import { http, HttpResponse } from 'msw';
import { makeMeeting } from '@/test/factories/meeting';
import type { Meeting } from '@/lib/types/meeting';

const BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL ?? 'http://localhost:8080';

export const meetingHandlers = [
  http.get(`${BASE_URL}/meetings/:id`, ({ params }) =>
    HttpResponse.json(makeMeeting({ id: params.id as string })),
  ),

  http.post(`${BASE_URL}/meetings`, async ({ request }) => {
    const body = (await request.json()) as Partial<Meeting>;
    // 테스트 fixture 라 과한 검증은 불필요 — 다만 명시 타입은 유지
    return HttpResponse.json(makeMeeting(body), { status: 201 });
  }),
];
```

- `Meeting` 타입 import — 호출자가 handler 를 재사용할 때 타입 정합성 유지.
- 엄격한 request body 검증은 fixture 책임 밖. 테스트가 잘못된 body 를 보내는지는 **테스트 본문의 assertion** 이 커버.

### 3.2. Vitest `server.ts`

```ts
// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### 3.3. 테스트별 핸들러 override

```ts
import { http, HttpResponse } from 'msw';
import { server } from '@/test/mocks/server';

it('shows error toast on 500', async () => {
  server.use(
    http.post(`${BASE_URL}/meetings`, () =>
      HttpResponse.json({ code: 'INTERNAL', message: '서버 오류' }, { status: 500 })
    ),
  );
  // ... 테스트 ...
});
```

- `afterEach(server.resetHandlers)` 가 다음 테스트로의 유출 방지.

### 3.4. 금지

- 함수-레벨 `vi.spyOn(global, 'fetch')` → MSW 로 통일
- 유닛과 E2E 의 handler **중복 작성** → `src/test/mocks/` 한 소스에서 공유
- handler 안에 테스트 로직 (`expect(...)`) 섞음 → fixture 로 분리

---

## 4. 공용 유틸

### 4.1. `renderWithProviders`

```tsx
// src/test/renderWithProviders.tsx
import { render, type RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { AuthProvider, type AuthState } from '@/features/auth/AuthContext';

type Options = Omit<RenderOptions, 'wrapper'> & {
  authState?: Partial<AuthState>;
  queryClient?: QueryClient;
};

export function renderWithProviders(ui: React.ReactElement, opts: Options = {}) {
  // 매 테스트 fresh QueryClient — 캐시 유출 금지
  const client = opts.queryClient ?? new QueryClient({
    defaultOptions: {
      queries: { retry: false, throwOnError: false },  // 테스트는 재시도·bubbling 끔
      mutations: { retry: false },
    },
  });
  return render(
    <QueryClientProvider client={client}>
      <AuthProvider initialState={opts.authState}>
        {ui}
      </AuthProvider>
    </QueryClientProvider>,
    opts,
  );
}
```

### 4.2. `resetStores`

```ts
// src/test/resetStores.ts
import { useUIStore } from '@/lib/stores/uiStore';
import { useRecordingStore } from '@/lib/stores/recordingStore';
import { useSpeakerMatchStore } from '@/lib/stores/speakerMatchStore';

export function resetStores() {
  useUIStore.setState(useUIStore.getInitialState(), true);  // true = replace
  useRecordingStore.setState(useRecordingStore.getInitialState(), true);
  useSpeakerMatchStore.setState(useSpeakerMatchStore.getInitialState(), true);
}
```

- **`getInitialState()` 는 Zustand v5 vanilla/hook store API 에 **원래 존재** — `create()` 로 만든 store 는 자동으로 노출**. 별도 설정 필요 없음.
- **단, slice composition pattern** (여러 slice 를 합쳐 하나의 store 를 만드는 패턴) 을 쓰면 `getInitialState()` 가 **composition 결과의 초기 상태 전체** 를 돌려주지만, 피처별 slice 를 섞은 뒤 초기화하면 다른 slice state 까지 함께 초기화됨 — slice 격리 테스트 시 **각 slice 가 자체 `reset` 액션 export** 하는 패턴이 더 안전.
- `useRecordingStore.setState(useRecordingStore.getInitialState(), true)` 패턴은 **단일 store** (slice composition 아님) 에서 안전.

### 4.3. 팩토리 패턴

```ts
// src/test/factories/meeting.ts
import type { Meeting } from '@/lib/types/meeting';

export function makeMeeting(overrides: Partial<Meeting> = {}): Meeting {
  return {
    id: 'm-1',
    title: '기본 회의',
    startAt: '2026-10-16T10:00:00Z',
    endAt: '2026-10-16T11:00:00Z',
    status: 'draft',
    createdBy: 'u-1',
    ...overrides,
  };
}
```

- 외부 라이브러리 (fishery 등) 미도입 — 함수로 시작, 복잡해지면 재검토 (#6).
- counter 가 필요한 경우 모듈 수준 변수 또는 `crypto.randomUUID()`.

---

## 5. 유닛 테스트 — zod 스키마

```ts
// src/lib/validation/schemas/meeting.test.ts
import { createMeetingSchema } from './meeting';

describe('createMeetingSchema', () => {
  it('accepts valid input', () => {
    const result = createMeetingSchema.safeParse({
      title: '팀 미팅',
      startAt: '2026-10-16T10:00:00Z',
      roomId: 'r-1',
    });
    expect(result.success).toBe(true);
  });

  it('rejects empty title with specific error', () => {
    const result = createMeetingSchema.safeParse({ title: '', startAt: '...', roomId: 'r-1' });
    expect(result.success).toBe(false);
    if (!result.success) {
      expect(result.error.issues[0].path).toEqual(['title']);
    }
  });
});
```

- 긍정 경로 + **대표적인 거부 경로** 몇 개만. 모든 edge case 를 테스트할 필요 없음.
- zod 자체는 테스트하지 않음 — 우리의 **비즈니스 규칙** (문자열 길이·날짜 범위 등) 만.

---

## 6. 유닛 테스트 — pure helper (Server Action 비즈니스 로직)

```ts
// src/features/meeting/createMeetingLogic.test.ts
import { createMeetingLogic } from './createMeetingLogic';
import { ApiError } from '@/lib/errors/ApiError';

describe('createMeetingLogic', () => {
  it('returns ok with created meeting', async () => {
    const fakeClient = { post: vi.fn().mockResolvedValue(makeMeeting({ id: 'm-99' })) };
    const result = await createMeetingLogic(validInput, fakeClient as any);
    expect(result).toEqual({ ok: true, data: { id: 'm-99' } });
  });

  it('returns ActionResult for duplicate ApiError', async () => {
    const fakeClient = { post: vi.fn().mockRejectedValue(
      new ApiError({ code: 'MEETING_DUPLICATE', status: 409, message: '중복' })
    )};
    const result = await createMeetingLogic(validInput, fakeClient as any);
    expect(result).toEqual({ ok: false, code: 'MEETING_DUPLICATE', message: expect.any(String) });
  });

  it('rethrows unexpected errors', async () => {
    const fakeClient = { post: vi.fn().mockRejectedValue(new Error('DB down')) };
    await expect(createMeetingLogic(validInput, fakeClient as any))
      .rejects.toThrow('DB down');
  });
});
```

- **client 를 주입** 받는 함수로 helper 설계 → mock 쉬움, MSW 불필요.
- Server Action (`actions.ts`) 자체는 helper + schema 조립이라 **대개 생략 가능** — 다만 schema 검증·redirect 분기·반환 shape 등 **통합 경계 동작** 은 중요하므로 **얇은 wrapper 테스트 1개** 로 "실제로 helper 가 호출되고 결과가 전파된다" 를 확인하는 게 안전. 전부 무조건 생략은 회귀 놓침.
- `redirect()` 등 Next 특수 예외가 섞이면 **Vitest `vi.mock('next/navigation', () => ({ redirect: vi.fn() }))`** 로 모킹.

---

## 7. 컴포넌트 테스트 — 폼 (RHF + zod)

```tsx
// src/features/meeting/CreateMeetingForm.test.tsx
import { CreateMeetingForm } from './CreateMeetingForm';
import { renderWithProviders } from '@/test/renderWithProviders';
import { screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { http, HttpResponse } from 'msw';
import { server } from '@/test/mocks/server';

describe('CreateMeetingForm', () => {
  it('submits and shows success', async () => {
    const user = userEvent.setup();
    renderWithProviders(<CreateMeetingForm />);
    await user.type(screen.getByLabelText('회의 제목'), '팀 미팅');
    await user.click(screen.getByRole('button', { name: '생성' }));
    expect(await screen.findByText(/생성됐습니다/)).toBeInTheDocument();
  });

  it('maps 422 fieldErrors to form fields', async () => {
    server.use(
      http.post('*/meetings', () =>
        HttpResponse.json({ code: 'VALIDATION', fieldErrors: { title: ['이미 사용 중'] } }, { status: 422 })
      ),
    );
    const user = userEvent.setup();
    renderWithProviders(<CreateMeetingForm />);
    await user.type(screen.getByLabelText('회의 제목'), '중복');
    await user.click(screen.getByRole('button', { name: '생성' }));
    expect(await screen.findByText('이미 사용 중')).toBeInTheDocument();
  });

  it('is accessible: button and inputs have accessible names', () => {
    renderWithProviders(<CreateMeetingForm />);
    expect(screen.getByRole('button', { name: '생성' })).toBeInTheDocument();
    expect(screen.getByLabelText('회의 제목')).toBeInTheDocument();
  });
});
```

**원칙**:
- 쿼리 우선순위: `getByRole` > `getByLabelText` > `getByText` > `getByTestId` (최후).
- `userEvent.setup()` 로 realistic 상호작용. `fireEvent` 는 특수 상황만.
- 비동기는 `findBy*` 또는 `waitFor`. `await` 생략 금지 (unhandled promise warning).
- **구현 세부 (RHF 내부 state inspection) 금지** — 사용자 보이는 결과만.

---

## 8. Zustand slice 테스트

```ts
// src/lib/stores/recordingStore.test.ts
import { useRecordingStore } from './recordingStore';

describe('recordingStore', () => {
  // beforeEach(resetStores) 가 vitest.setup.ts 에서 돌아감

  it('starts a session', () => {
    useRecordingStore.getState().start('session-1');
    expect(useRecordingStore.getState().status).toBe('recording');
    expect(useRecordingStore.getState().sessionId).toBe('session-1');
  });

  it('pauses then resumes', () => {
    const { start, pause, resume } = useRecordingStore.getState();
    start('s-1');
    pause();
    expect(useRecordingStore.getState().status).toBe('paused');
    resume();
    expect(useRecordingStore.getState().status).toBe('recording');
  });
});
```

- store 를 **훅으로 쓰지 않고 `.getState()` / `.setState()`** 로 직접 조작 (유닛).
- 컴포넌트 안에서 쓰이는 셀렉터는 **컴포넌트 테스트** 로 커버.

---

## 9. 에러 바운더리 테스트

```tsx
// src/lib/errors/ErrorBoundary.test.tsx
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { screen } from '@testing-library/react';
import { ErrorBoundary } from './ErrorBoundary';
import * as reportModule from './reportError';
import { renderWithProviders } from '@/test/renderWithProviders';

function Boom(): never { throw new Error('kaboom'); }

describe('ErrorBoundary', () => {
  let reportSpy: ReturnType<typeof vi.spyOn>;
  let consoleSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    // reportError 모듈을 스파이 — 실제 에러 보고 경계 검증
    reportSpy = vi.spyOn(reportModule, 'reportError').mockImplementation(() => {});
    // React dev-mode 가 에러 바운더리 작동 시 console.error 를 출력 — 테스트 노이즈 억제
    consoleSpy = vi.spyOn(console, 'error').mockImplementation(() => {});
  });

  afterEach(() => {
    // mockRestore 는 assertion 실패해도 실행돼야 함 → afterEach 에서
    reportSpy.mockRestore();
    consoleSpy.mockRestore();
  });

  it('shows fallback and reports', () => {
    renderWithProviders(
      <ErrorBoundary source="test" Fallback={({ error }) => <div>FAIL: {error.message}</div>}>
        <Boom />
      </ErrorBoundary>
    );
    expect(screen.getByText(/FAIL: kaboom/)).toBeInTheDocument();
    expect(reportSpy).toHaveBeenCalledWith(
      expect.any(Error),
      expect.objectContaining({ source: 'test' })
    );
  });
});
```

- **`reportError` 자체를 spy** — `console.error` 가 아니라 **실제 보고 경계** 를 검증.
- `mockRestore()` 를 `afterEach` 에 둠 — assertion 실패 시에도 다른 테스트에 영향 없음.
- React dev-mode 가 ErrorBoundary 발생 시 자동으로 콘솔에 찍는 노이즈를 `consoleSpy` 로 억제.

---

## 10. React Query 테스트

```tsx
// src/features/meeting/queries.test.tsx  (JSX 쓰므로 .tsx)
import type { ReactNode } from 'react';
import { useQuery, QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { renderHook, waitFor } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { meetingDetailQuery } from './queries';

describe('meetingDetailQuery', () => {
  it('fetches a meeting by id', async () => {
    const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });
    const wrapper = ({ children }: { children: ReactNode }) => (
      <QueryClientProvider client={qc}>{children}</QueryClientProvider>
    );
    const { result } = renderHook(() => useQuery(meetingDetailQuery('m-1')), { wrapper });
    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data).toEqual(expect.objectContaining({ id: 'm-1' }));
  });
});
```

- 매 테스트 fresh `QueryClient`.
- MSW 가 `GET /meetings/m-1` 응답 — handler 가 default.
- 에러 시나리오는 `server.use(http.get(..., () => HttpResponse.json(..., { status: 500 })))`.
- **파일 확장자**: JSX 쓰는 테스트는 `.test.tsx`.
- 복사-가능 템플릿 — import 완비, `tsc --noEmit` 통과 전제.

---

## 11. Playwright E2E

### 11.1. 네트워크 모킹 범위 — 중요

Playwright E2E 에서 모킹 가능한 네트워크 호출은 **발생 위치별로 다르다**:

| 호출 위치 | 모킹 방법 |
|---|---|
| 브라우저 코드 (`"use client"` 안, React Query, `browserClient`) | **Playwright `page.route()`** 로 네이티브 가로채기 (권장, 간단) <br/> 또는 MSW browser worker (`public/mockServiceWorker.js` + `setupWorker(...).start()`) — 설치 복잡 |
| RSC 서버 컴포넌트 fetch, Server Action, Route Handler | **직접 가로채기 어려움**. 두 선택지: <br/> (a) **백엔드 자체를 모킹** (별도 모킹 서버 띄우고 `NEXT_PUBLIC_API_BASE_URL` 교체) <br/> (b) Next 16 `experimental/testmode` 사용 — **실험적 단계, 공식 recipe 미성숙**. 지금은 활성화 안 함 |

**현재 결정** (실용 중심):
- E2E 는 **`page.route()` + 백엔드 모킹 서버** 조합 — handlers 는 `src/test/mocks/` 와 **같은 domain 로직** 을 재사용 (완전 공유는 어려움, 어댑터 함수로 변환).
- `experimental/testmode` 는 **"탐색 과제"** 로 열어 둠. 공식 Playwright 통합 recipe 가 나오면 재검토 (현재 미도입).
- 백엔드 모킹 서버 실제 구성은 **첫 E2E 스펙 작성 시** 정립 (`e2e/mock-server/` 디렉터리 또는 MSW `setupServer` + Node 서버 wrapper).

### 11.2. 기본 설정 (`playwright.config.ts` 핵심)

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 1 : 0,  // local 은 재시도 없음 (플레이크 숨김 방지)
  reporter: 'html',
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL ?? 'http://localhost:3000',
    trace: 'retain-on-failure',  // 실패시만 trace 보존
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
  webServer: {
    command: 'npm run build && npm start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    env: {
      // 백엔드 모킹 서버로 교체 (현재는 placeholder, 첫 E2E 작성 시 정립)
      NEXT_PUBLIC_API_BASE_URL: process.env.E2E_API_BASE_URL ?? 'http://localhost:8080',
    },
  },
});
```

### 11.3. Fixtures — Playwright `page.route()` + a11y

```ts
// e2e/fixtures.ts
import { test as base, expect, type Route, type Request } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

type Fixtures = {
  expectNoA11yViolations: (opts?: { tags?: string[] }) => Promise<void>;
  mockApi: (path: string | RegExp, handler: (route: Route, request: Request) => Promise<void> | void) => Promise<void>;
};

export const test = base.extend<Fixtures>({
  expectNoA11yViolations: async ({ page }, use) => {
    await use(async (opts) => {
      const results = await new AxeBuilder({ page })
        .withTags(opts?.tags ?? ['wcag2a', 'wcag2aa'])
        .analyze();
      const serious = results.violations.filter(v => ['critical', 'serious'].includes(v.impact ?? ''));
      const warn = results.violations.filter(v => ['moderate', 'minor'].includes(v.impact ?? ''));
      if (warn.length > 0) {
        // moderate/minor 는 fail 아닌 warning — HTML reporter attach + stdout 기록
        console.warn('[a11y warning]', JSON.stringify(warn.map(v => ({ id: v.id, impact: v.impact, help: v.help })), null, 2));
      }
      expect(serious, JSON.stringify(serious, null, 2)).toHaveLength(0);
    });
  },

  mockApi: async ({ page }, use) => {
    await use(async (path, handler) => {
      await page.route(path, handler);
    });
  },
});

export { expect };
```

### 11.4. 스모크 스펙 템플릿

```ts
// e2e/meeting-create.spec.ts
import { test, expect } from './fixtures';

test('user can create a meeting', async ({ page, mockApi, expectNoA11yViolations }) => {
  // 브라우저 요청 모킹 — page.route 로 간단히
  await mockApi(/\/meetings$/, async (route, req) => {
    if (req.method() === 'POST') {
      await route.fulfill({ status: 201, body: JSON.stringify({ id: 'm-99' }) });
    } else {
      await route.continue();
    }
  });

  await page.goto('/login');
  await page.getByLabel('이메일').fill('user@example.com');
  await page.getByLabel('비밀번호').fill('password');
  await page.getByRole('button', { name: '로그인' }).click();

  await page.goto('/meeting/new');
  await page.getByLabel('회의 제목').fill('팀 스탠드업');
  await page.getByRole('button', { name: '생성' }).click();

  await expect(page.getByText('생성됐습니다')).toBeVisible();
  await expectNoA11yViolations();  // critical/serious fail / moderate/minor warn
});
```

> **주**: 서버 Action 이 호출하는 백엔드 요청은 `page.route()` 로 못 가로챈다 (서버에서 발생). 이런 호출은 **백엔드 모킹 서버가 담당** — 첫 Server Action E2E 작성 시 모킹 서버 설정 추가.

### 11.3. 스모크 스펙 템플릿

```ts
// e2e/meeting-create.spec.ts
import { test, expect } from './fixtures';

test('user can create a meeting', async ({ page, expectNoA11yViolations }) => {
  await page.goto('/login');
  await page.getByLabel('이메일').fill('user@example.com');
  await page.getByLabel('비밀번호').fill('password');
  await page.getByRole('button', { name: '로그인' }).click();

  await page.goto('/meeting/new');
  await page.getByLabel('회의 제목').fill('팀 스탠드업');
  await page.getByRole('button', { name: '생성' }).click();

  await expect(page.getByText('생성됐습니다')).toBeVisible();
  await expectNoA11yViolations();  // #7 결정 — critical/serious fail
});
```

### 11.4. E2E 원칙

- 스펙당 **1 유저 플로우**. 하나로 모든 걸 검증 금지.
- `page.getByRole` 우선. `page.locator('.css-selector')` 지양 (내부 구조 결합).
- 네트워크 모킹은 **MSW handlers 재사용** — fixture 가 설치.
- `await expect(locator).toBeVisible()` 등 **auto-wait** 어서션. `page.waitForTimeout` 금지.
- 플레이크 보이면 **원인 수정**. retries 로 숨기지 않음 (`retries: 0` local).

---

## 12. 접근성 테스트 (#7 연계)

- E2E 각 스펙에서 `expectNoA11yViolations()` 호출 — critical/serious 는 fail, moderate/minor 는 warning (axe `tags` 로 레벨 선택).
- 컴포넌트 유닛에서도 간단 체크 가능 — `getByRole`/`getByLabelText` 가 안 보이면 접근 가능한 이름이 없다는 뜻 → 자연스레 접근성 가드.
- `jest-axe` 는 **당장 도입 안 함** — Playwright 의 axe 가 통합 상황에서 더 정확. 유닛 레벨 a11y 는 RTL 쿼리로 커버.

---

## 13. 커버리지

- **하드 게이트 없음** (#6 결정).
- 로컬 리포트: `vitest run --coverage` (v8 provider 기본).
- **"변경된 모듈에 테스트 있는가"** PR 리뷰 체크로 관리.
- 커버리지 낮은 영역이 쓸모없는 테스트로 채워지지 않도록 — 오히려 **무엇을 안 테스트하는지 명시** (주석 또는 섹션) 하는 게 건강.

---

## 14. 금지 패턴 (요약)

- **RSC 서버 컴포넌트의 Next 프레임워크 통합 동작 (렌더링·캐시·prefetch) 을 직접 유닛 테스트** → E2E 로 (#6). 단 서버 컴포넌트가 쓰는 **순수 helper/formatter/변환 함수** 는 추출해서 유닛 가능.
- **Server Action 함수 자체를 직접 테스트** → pure helper 추출해 helper 를 유닛
- **구현 세부 테스트** (hook internal state, class private, 외부 라이브러리 내부 함수 spy)
- **함수 레벨 fetch mock** (`vi.spyOn(global, 'fetch')`) → MSW
- `page.waitForTimeout(1000)` — 명확한 어서션 (`toBeVisible` 등) 사용
- 플레이크 재시도로 숨기기 (local `retries: 0`, CI `retries: 1` 상한)
- 테스트에서 production store 직접 import 후 조작 — `resetStores()` 유틸 + `.setState` 패턴
- Fake timers 로 production 코드를 쪼개기 — 짧은 setTimeout 는 real-time 로 테스트, 긴 interval 만 `vi.useFakeTimers()`
- E2E 가 real backend 호출 — MSW 통일. 만약 integration 필요하면 **별도 project** 로 분리

---

## 15. 체크리스트 — 테스트 작성·리뷰 시

- [ ] 테스트 파일이 **코로케이트** (`foo.ts` + `foo.test.ts`)?
- [ ] `getByRole` / `getByLabelText` 우선 사용? `getByTestId` 를 **최후 수단**으로만?
- [ ] 네트워크 호출이 **MSW handlers** 를 통해 모킹? 함수 레벨 fetch mock 없음?
- [ ] `renderWithProviders` 사용 (매 테스트 fresh `QueryClient`)?
- [ ] `afterEach` 에 `cleanup` + `server.resetHandlers` + `resetStores` 가 돌아가는가 (setup 자동)?
- [ ] 비동기 어서션이 `findBy*` 또는 `waitFor`? `waitForTimeout` 없는가?
- [ ] Server Action 테스트가 **pure helper 유닛** 인가? action.ts 자체 테스트 지양?
- [ ] RSC 를 유닛으로 테스트하려 시도하지 않았는가? → E2E 로.
- [ ] Zustand store 유닛이 `.getState()`/`.setState()` 로 조작 (훅 아닌)?
- [ ] 에러 바운더리 테스트가 **`reportError` 모듈 자체** 를 spy (`console.error` 아님)? `mockRestore` 가 `afterEach` 에?
- [ ] E2E 가 1 스펙 1 유저 플로우? `page.waitForTimeout` 없는가?
- [ ] E2E 네트워크 모킹 범위를 알고 있는가? **브라우저 요청 → `page.route`**, **서버 요청 (RSC/Server Action) → 백엔드 모킹 서버 또는 testmode (현재 미도입)**.
- [ ] E2E 스펙에 `expectNoA11yViolations()` 가 포함? moderate/minor 는 console.warn 으로 기록되는가?
- [ ] 플레이크 발견 시 **원인 수정** (재시도로 숨김 금지)?
- [ ] 테스트가 **행동** 을 검증 (구현 세부·내부 state inspection 금지)?

---

## 16. 참고

- 결정 축: [#6 test-strategy](../../../docs/axis-decisions/test-strategy.md), [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md), [#3 state-management](../../../docs/axis-decisions/state-management.md), [#4 form-validation](../../../docs/axis-decisions/form-validation.md), [#5 error-ux](../../../docs/axis-decisions/error-ux.md), [#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md)
- 관련 스킬: [`nextjs16-reality-check`](../nextjs16-reality-check/SKILL.md) §2.4 (testmode primitives), [`app-router-layering`](../app-router-layering/SKILL.md), [`error-ux`](../error-ux/SKILL.md), [`data-fetching-guard`](../data-fetching-guard/SKILL.md), [`ts-style`](../ts-style/SKILL.md)
- Next 16 testing primitives: `node_modules/next/dist/experimental/testing/server/`, `node_modules/next/dist/experimental/testmode/`
- Vitest / Testing Library / MSW v2 / Playwright 공식 docs
