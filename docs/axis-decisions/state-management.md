---
title: 상태 관리 전략
date: 2026-10-16
author: m05214@idstrust.com
status: DECIDED
issue: "#3"
scope: flowmeet-web 클라이언트 상태의 범주별 담당 계층 결정 (서버/폼 상태 제외)
depends_on: ["#2 데이터 페칭/캐시 전략"]
---

# 상태 관리 전략

[#2 데이터 페칭](./data-fetching.md) 결정으로 **서버 상태는 React Query v5** 가 전담하게 되어, 이 축에서 다루는 범위는 **"서버 상태·폼 상태를 제외한 클라이언트 상태"** 로 좁혀진다. 특히 TRD §4.1 녹음 세션·§4.3 화자 매칭처럼 **수명이 길고 여러 컴포넌트가 공유하며 자주 변하는** 피처 로컬 상태의 담당자를 정하는 것이 핵심이다.

> 이 문서는 `app-router-layering` 스킬([#15](https://github.com/timonPark/flowmeet-web/issues/15)) 의 입력 중 하나다.

---

## 1. 결정 요약

| 상태 범주 | 담당 계층 |
|---|---|
| 서버 데이터 | **React Query v5** (#2 결정) |
| 폼 상태 | **react-hook-form 유력** (#4 에서 확정) |
| 인증 세션 (user, role, accessToken) | **React Context** (`AuthProvider`) |
| UI 상태 (모달·토스트·사이드바) | **Zustand slice** (`uiStore`) |
| 피처 로컬 상태 (녹음·화자매칭 등) | **Zustand slice** (피처별 분리) |
| 서버→클라 하이드레이션 | **RQ `HydrationBoundary`** (#2 결정) |

**원칙**: *"RQ 가 가지고 있으면 Zustand 에 넣지 않는다"* — 서버 데이터 중복 저장 금지.

---

## 2. 결정 상세

### 2.1. 라이브러리 선택: Context + Zustand 하이브리드

- **Context** — `AuthProvider` 하나만. 로그인 후 거의 안 바뀌므로 리렌더 비용 무시 가능. `src/app/providers.tsx` (client component) 에 모든 Provider 집계.
- **Zustand (v5)** — 피처별·관심사별 slice store. Provider 불필요 → Next 16 `cacheComponents` 환경에서 서버/클라 경계 사고 최소화. 셀렉터 패턴으로 리렌더 범위 제한.
- **채택하지 않은 것**:
  - Jotai: 아톰 하이드레이션(`useHydrateAtoms`) 의 Next 16 성숙도 미달
  - Redux Toolkit: 이 앱 규모에서 번들·러닝커브 과함
  - "Context 만": 녹음·화자매칭처럼 자주 바뀌는 상태에서 전체 Provider 리렌더 비용

### 2.2. Store 구성

디렉터리: `src/lib/stores/`

| 파일 | 책임 | 참고 |
|---|---|---|
| `uiStore.ts` | 사이드바 토글, 모달 열림/닫힘, 토스트 큐 | 전역 UI 크로스컷 |
| `recordingStore.ts` | MediaRecorder 상태(idle/recording/paused), 세션 id, 업로드 진행률, 에러 — **메타데이터만** | TRD §4.1 — chunk 바이너리는 전용 IndexedDB 레이어 담당, Zustand 는 참조 id 만 |
| `speakerMatchStore.ts` | 화자 라벨↔사용자 매핑 드래프트, 선택된 utterance id 집합 | TRD §4.3 "즉시 로컬 반영" |

> 새 피처가 "크로스 컴포넌트 + 자주 변함 + 서버 데이터 아님" 세 조건을 모두 만족하면 slice 추가. 아니면 로컬 `useState` 로 시작.

### 2.3. Zustand 사용 규칙

- **셀렉터 필수**: `const sessionId = useRecordingStore(s => s.sessionId)` 형태. 전체 상태를 구독하는 `useRecordingStore()` 호출 금지.
- **얕은 비교 필요한 경우** `shallow` 적용:
  ```ts
  import { useShallow } from 'zustand/react/shallow';
  const { a, b } = useStore(useShallow(s => ({ a: s.a, b: s.b })));
  ```
- **actions 는 state 와 함께 정의**: `set` / `get` 을 쓰는 action 을 store 안에 두고, 호출부는 `const start = useRecordingStore(s => s.start)` 로 꺼내 쓴다.
- **DevTools**: `zustand/middleware` 의 `devtools` 를 **개발 빌드에서만** 적용.
- **금지**:
  - 서버 데이터를 저장 (RQ 영역 침범)
  - 모놀리식 글로벌 store (피처별 분리 유지)
  - store 안에서 직접 `fetch` 호출 (서버 통신은 RQ `queryFn`·Server Action 경유)

### 2.4. 인증 Context

- 범위: `user`, `role`, 메모리 보관 `accessToken` (Refresh Token 은 HttpOnly 쿠키 — #19 에서 확정)
- 공급 위치: `src/app/providers.tsx` 안의 `AuthProvider` (client component)
- 서버 측 세션 조회는 Server Action 또는 RSC 에서 쿠키 직접 판독 — Context 를 거치지 않음.

### 2.5. 폼 상태 (경계)

- 이 축에서는 결정하지 않는다 (#4 에서 확정).
- 다만 **원칙**: 폼 내부 상태는 폼 라이브러리가 보유, Zustand/Context 에 폼 중간값을 복제하지 않는다. 필요 시 폼 `onSubmit` 결과만 상위 store 로 올린다.

---

## 3. Provider 트리 (요약)

```tsx
// src/app/providers.tsx (client component)
'use client';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProviderWrapper>
      <AuthProvider>
        {children}
      </AuthProvider>
    </QueryClientProviderWrapper>
  );
}
```

- Zustand store 는 Provider 가 **없다** — 각 store 모듈이 싱글턴. 테스트 격리는 store `setState` 리셋 유틸로 처리.
- `QueryClientProviderWrapper` 안에서 `QueryClient` 는 **요청마다 새 인스턴스** 로 생성해 서버 렌더 간 유출 방지 (`cacheComponents` 하에서 중요).

---

## 4. 다른 축에 미치는 영향

| 이슈 | 내용 |
|---|---|
| [#2](https://github.com/timonPark/flowmeet-web/issues/2) 데이터 페칭 | 이 결정의 전제. 변경 불가 전제. |
| [#4](https://github.com/timonPark/flowmeet-web/issues/4) 폼/검증 | 폼 라이브러리는 폼 내부 상태 담당. 상위 공유가 필요한 값만 Zustand/Context 로. |
| [#5](https://github.com/timonPark/flowmeet-web/issues/5) 에러 처리 | 토스트 큐는 `uiStore` 에 위치. `error-ux` 가 토스트 발행 API 를 정의. |
| [#15](https://github.com/timonPark/flowmeet-web/issues/15) `app-router-layering` | `src/lib/stores/` 와 `src/lib/api/` 의 책임 분리 규칙 포함 (store 가 fetch 금지 등). |
| [#19](https://github.com/timonPark/flowmeet-web/issues/19) `web-security` | `AuthProvider` 에 보관하는 accessToken 의 **메모리 전용 보관** 원칙 반영. |

---

## 5. 열어둔 포인트 (defer)

| 항목 | 사유 | 어디로 |
|---|---|---|
| 녹음 chunk 영속화 어댑터 (Zustand `persist` vs 전용 IndexedDB 레이어) | 녹음 플로우 구현 시 재검토. 지금은 "Zustand 는 메타데이터만, 실데이터는 전용 레이어" 전제. | 녹음 피처 구현 이슈 |
| Zustand store 테스트 유틸 (`resetAll` 등) | 테스트 전략(#6) 확정 후 함께. | #6 / #18 |
| Devtools 설정 세부 (각 store 네임) | 첫 store 작성 시 함께 결정. | 구현 단계 |

---

## 6. 완료 조건 체크리스트

- [x] 범주별 담당 계층 매핑
- [x] 라이브러리 선택 + 선정 사유
- [x] Zustand 사용 규칙 (셀렉터·actions·금지 패턴)
- [x] Provider 트리 스케치
- [x] 다른 축 영향 매핑
- [x] 열어둔 포인트 이관처 명시
- [ ] CLAUDE.md 축약 반영 + 이 문서 링크 (이 PR 에 포함)
- [ ] 실제 구현 (이 PR 범위 밖)

## 7. 참고

- [`docs/axis-decisions/data-fetching.md`](./data-fetching.md) — 서버 상태 담당 결정
- [`docs/harness-bootstrap-survey.md`](../harness-bootstrap-survey.md) §2
- TRD §4.1 녹음, §4.3 화자 매칭
- Zustand v5 공식 가이드 (최신 셀렉터/`useShallow` 패턴)
