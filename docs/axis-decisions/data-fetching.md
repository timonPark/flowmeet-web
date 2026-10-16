---
title: 데이터 페칭/캐시 전략
date: 2026-10-16
author: kindcorder@gmail.com
status: DECIDED
issue: "#2"
scope: flowmeet-web 서버/클라이언트 데이터 로딩·캐싱·뮤테이션 기본 패턴
supersedes: null
---

# 데이터 페칭/캐시 전략

flowmeet-web은 Next.js 16.4 (`cacheComponents: true` + `partialPrefetching: true`) 를 전제로 한다. Next 15 식 "fetch 쓰면 알아서 캐시"가 폐기되고 **"기본 dynamic, `'use cache'` 디렉티브로 명시 opt-in"** 모델로 바뀐 환경이다. 이 축에서는 네 가지 차원 (A) 서버 로딩 베이스, (B) 클라이언트 캐시, (C) 뮤테이션, (D) fetch wrapper 위치 를 결정한다.

> 이 문서는 `data-fetching-guard` 스킬([#17](https://github.com/timonPark/flowmeet-web/issues/17)) 의 입력이다. 후속 스킬 작성 시 "왜 이렇게 정했는지"를 다시 설명하지 말고 이 문서를 링크한다.

---

## 1. 결정 요약

| 차원 | 결정 | 축약 |
|---|---|---|
| A. 서버 로딩 베이스 | **RSC `fetch` + `'use cache'` 선택 적용** | A1 |
| B. 클라이언트 캐시 | **React Query v5 (TanStack Query)** | B3 |
| C. 뮤테이션 | **Server Actions + React Query mutation 혼용** | C3 |
| D. fetch wrapper 위치 | **서버용·클라이언트용 분리, 공통 유틸만 중립** | D2 |

> 추가: 서버에서 미리 받은 데이터는 React Query `HydrationBoundary` 로 **SSR → 클라 캐시 매끄럽게 연결**한다.

---

## 2. 결정 상세

### A1. 서버 로딩 베이스 — RSC `fetch` + `'use cache'` 선택 적용

- 서버 컴포넌트의 `fetch()` 는 Next 16 cache components 모델상 **기본 dynamic**. 명시 캐시가 필요한 곳만 `'use cache'` 함수로 감싼다.
- 캐시 적용 범위:
  - **대시보드 집계** (TRD §8, `dashboardType=TEAM` 지연 업무 등 — 조직 단위 집계) → `'use cache'` + `cacheLife('minutes')` + `cacheTag('dashboard:{teamId}')`
  - **회의실 캘린더 뷰** (TRD §4.2 — 일/주간) → `'use cache'` + `cacheLife('minutes')` + `cacheTag('calendar:{roomId}:{yyyy-ww}')`
  - **마스터 데이터** (팀 목록, 회의실 목록 등) → `'use cache'` + `cacheLife('hours')`
- 캐시 미적용 (dynamic 유지):
  - **사용자별 리스트** (내 회의, 내 지시) — 세션 쿠키 기반, 캐시 공유 위험
  - **회의 상세·녹음·화자매칭 등 상호작용 리소스** — 변화 빈도 높음, 아래 B3 에서 클라 캐시로 처리
- 무효화: 쓰기 Server Action 또는 Route Handler 후 `revalidateTag(...)` / `updateTag(...)` 호출.

### B3. 클라이언트 캐시 — React Query v5

- 선정 사유: TRD §4.3 "**화자 매칭 UI 즉시 로컬 반영**", §4.1 녹음 중 상태, 지연 업무 폴링 등 **클라이언트 상호작용 중심 흐름**이 많다. React Query 의 낙관 업데이트·롤백·`useInfiniteQuery`·stale/refetch 전략이 성숙하다.
- SWR 미채택: 뮤테이션 생태(낙관 업데이트, 롤백, invalidation 그래프)가 상대적으로 얇다.
- 설치: `@tanstack/react-query` (버전은 설치 시점 최신 v5).
- `QueryClient` 는 **루트 `app/layout.tsx` 에서 공급하지 말고, 클라이언트 전용 Provider 컴포넌트**(`src/lib/query/QueryProvider.tsx` 등) 로 감싸 서버 렌더 시 생성/주입. (`cacheComponents` 환경에서 요청마다 QueryClient 가 유출되지 않도록)
- 기본 옵션:
  - `staleTime: 30_000` (불필요한 재요청 억제)
  - `gcTime: 5 * 60 * 1000`
  - `retry: 1` (서버 측 재시도는 fetch wrapper 가 담당, 더 반복하지 않음)
  - `refetchOnWindowFocus: false` (대시보드 폴링은 명시적 `refetchInterval` 로)

### C3. 뮤테이션 — Server Actions + React Query mutation 혼용

| 흐름 | 선택 | 근거 |
|---|---|---|
| 폼 제출 (회의 생성, 지시 생성, 일정 등록) | **Server Actions** + `revalidateTag` | 서버-서버 왕복 1회, `useActionState` 로 폼 UX 깔끔 |
| 화자 매칭 라벨 변경 (TRD §4.3) | **React Query mutation + 낙관 업데이트** | 즉시 로컬 반영 요구 |
| 녹음 중 상태 (업로드 진행률 등) | **React Query mutation** | 클라이언트 상태와 결합 |
| 간단 토글 (즐겨찾기 등) | **React Query mutation + 낙관 업데이트** | 서버 액션은 과한 왕복 |

- Server Action 내부에서도 반드시 **D2 서버용 fetch wrapper**를 사용해 api 호출 (직접 `fetch` 금지).

### D2. fetch wrapper — 서버·클라이언트 분리

- 공통 중립 유틸: `src/lib/api/_core.ts` (URL 조립, 에러 매핑, 응답 파싱, `ApiError` 타입)
- 서버 전용: `src/lib/api/serverClient.ts`
  - `cookies()` / `headers()` 로 access token·요청 메타 추출
  - 401 응답 시 **Refresh Token 회전** (쿠키의 HttpOnly refresh token 사용) — 1회만 재시도, 실패 시 `redirect('/login')`
  - Server Action / RSC / Route Handler 에서 사용
- 클라이언트 전용: `src/lib/api/browserClient.ts`
  - access token은 메모리/`AuthContext` 에서 조회 (localStorage 사용 금지 — XSS 리스크)
  - 401 응답 시 Server Action 호출로 refresh 위임 (쿠키 접근은 서버에서만)
  - React Query 의 `queryFn` 에서 사용

- 서버·클라 공용 함수는 금지. 둘 중 하나에서 쓰이면 그 쪽 wrapper 를 명시적으로 import. (한 함수에 환경 분기 넣지 않음)

### 추가: React Query SSR Hydration

- 서버에서 RSC `fetch` 로 받은 데이터를 React Query 로 승계할 흐름이 생기면 (예: 회의 상세 페이지의 초기 데이터 → 클라에서 이후 업데이트 조작), **`HydrationBoundary` + `prefetchQuery` 패턴**을 사용한다.
- 모든 페이지에 강제 적용하지는 않는다. "서버에서 1차 랜더 → 클라에서 조작" 흐름이 명확한 경우에만.

---

## 3. 다른 축에 미치는 영향

| 영향 받는 축 / 이슈 | 내용 |
|---|---|
| [#5](https://github.com/timonPark/flowmeet-web/issues/5) 에러 처리 | fetch wrapper 가 던지는 `ApiError` 가 `error-ux` 입력. api `ErrorCode` → 사용자 메시지 매핑 테이블 필요. |
| [#8](https://github.com/timonPark/flowmeet-web/issues/8) 로깅/모니터링 | 서버/클라 wrapper 양쪽에 Sentry 등 훅 포인트. |
| [#9](https://github.com/timonPark/flowmeet-web/issues/9) 운영 규정 | 캐시 태그 네이밍 규약 ("`<domain>:<id>`") 을 PR 체크리스트에 포함. |
| [#13](https://github.com/timonPark/flowmeet-web/issues/13) `nextjs16-reality-check` | Opus/Sonnet 가 "`fetch` 는 기본 캐시" 라고 전제하는 코드를 쓰면 틀림. 가드 포인트로 명시. |
| [#15](https://github.com/timonPark/flowmeet-web/issues/15) `app-router-layering` | `src/lib/api/{serverClient,browserClient,_core}.ts` 경계를 레이어링 룰에 포함. |
| [#17](https://github.com/timonPark/flowmeet-web/issues/17) `data-fetching-guard` | 이 문서가 입력. 스킬에서 체크해야 할 것: `'use cache'` 사용처, `cacheTag` 네이밍, 서버/클라 wrapper 혼용 금지, 401 refresh 중복 호출 금지. |
| [#19](https://github.com/timonPark/flowmeet-web/issues/19) `web-security` | Refresh Token 저장 (HttpOnly 쿠키) 결정은 이쪽에서 최종 확정. |

---

## 4. 열어둔 포인트 (defer)

| 항목 | 결정 보류 사유 | 어디로 |
|---|---|---|
| Refresh Token 저장 방식 (HttpOnly 쿠키 전제이나 세부 TTL·회전 정책) | 보안 축에서 묶어 보는 게 깔끔 | [#8](https://github.com/timonPark/flowmeet-web/issues/8) / [#19](https://github.com/timonPark/flowmeet-web/issues/19) |
| Realtime (WebSocket / SSE) 필요 여부 | TRD 상 명시 없음. 당분간 **polling 전제** (React Query `refetchInterval`). 요구 생기면 재합의. | 추후 신규 이슈 |
| 외부 HTTP 라이브러리 (`ky`, `axios`) 도입 | 네이티브 `fetch` + 얇은 wrapper 로 시작. 재시도·인터셉터 복잡해지면 재검토. | `data-fetching-guard` 작성 시 재평가 |

---

## 5. 완료 조건 체크리스트

- [x] A·B·C·D 네 차원 결정
- [x] 각 결정의 트레이드오프와 선정 사유 명시
- [x] 다른 축/이슈에 미치는 영향 매핑
- [x] 열어둔 포인트의 이관처 명시
- [ ] CLAUDE.md 에 축약 반영 + 이 문서 링크 (이 PR 에 포함)
- [ ] 실제 구현 (이 PR 범위 밖, 후속 이슈)

## 6. 참고

- [`docs/harness-bootstrap-survey.md`](../harness-bootstrap-survey.md) §2, §3, §4
- Next 16 `next/cache` 공개 API: `node_modules/next/cache.d.ts`
- Next 16 config 기본값: `node_modules/next/dist/esm/server/config-shared.js:85-170`
- `partialPrefetching` 요구사항: `node_modules/next/dist/esm/server/config.js:375-383`
- TRD §4.1 녹음, §4.2 회의실 캘린더, §4.3 화자 매칭, §8 대시보드
