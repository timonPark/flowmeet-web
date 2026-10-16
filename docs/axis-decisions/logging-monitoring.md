---
title: 로깅·에러 모니터링 전략
date: 2026-10-16
author: kindcorder@gmail.com
status: DECIDED
issue: "#8"
scope: 에러 모니터링 벤더·클라/서버 로그 수집·`reportError` 구현·마스킹 최소 안전망
depends_on:
  - "#5 에러 처리(error-ux) 전략"
---

# 로깅·에러 모니터링 전략

[#5 에러 처리](./error-ux.md) 가 `reportError(err, ctx)` 를 **추상 훅**으로 심어 두었다. 이 축은 그 **구현을 뭘로 채울지** 와 **클라/서버 로그 수집 범위** 를 확정한다. 결론은 **외부 SaaS 벤더 미도입** — `reportError` 는 **구조화 JSON `console.error`** 로 유지하고, 필요 시 **자체 백엔드 수집 엔드포인트** 로 확장할 수 있도록 인터페이스만 다져 둔다.

> 이 문서는 `web-security` 스킬([#19](https://github.com/timonPark/flowmeet-web/issues/19)) 과 `web-testing` 스킬([#18](https://github.com/timonPark/flowmeet-web/issues/18)) 의 입력. `reportError` 가 테스트에서 spy 되는 지점 역시 유지.

---

## 1. 결정 요약

| 차원 | 결정 |
|---|---|
| 외부 모니터링 벤더 | **미도입** (Sentry / Datadog / GlitchTip / LogRocket 등 전부 X) |
| `reportError` 구현 | **구조화 JSON** `console.error` 유지 |
| 클라이언트 → 서버 수집 엔드포인트 | **열어둠** — flowmeet-api 설계 완료 시 `reportError` 내부에서 조건부 fetch |
| 서버 로그 | `console.*` 만. 구조화 로그·수집은 **배포 플랫폼 결정 후** |
| 사용자 식별 | **opaque userId** 만. 이메일/이름/전화 등 PII 금지 |
| 민감정보 마스킹 | **최소 안전망** — `password|token|secret|authorization|cookie` 등 알려진 키 자동 치환. 상세는 #19 |
| 재평가 트리거 | production blind 가 운영 문제가 될 때 (사용자 리포트 반복, 트래픽 급증) |

**원칙**: *외부로 데이터를 내보내지 않는다. 추후 자체 수집 레이어든 다른 선택이든, 호출부가 변경될 필요 없는 추상만 지금 다져 둔다.*

---

## 2. 결정 상세

### 2.1. 외부 벤더 미도입

- **Sentry, Datadog, LogRocket, GlitchTip, Rollbar 등 외부 SaaS 또는 자가 호스팅 OSS 수집 서비스 모두 지금 도입하지 않는다.**
- 사용자 결정 (2026-10-16).
- 재제안 금지: 사용자가 **명시적으로** 다른 벤더를 요청하기 전까지, 축의 추천·열어둔 포인트 어디에도 "Sentry 를 쓰자" 는 복귀 제안을 넣지 않는다.

### 2.2. `reportError` 구현 — 구조화 JSON `console.error`

```ts
// src/lib/errors/reportError.ts
import { maskSensitive } from './maskSensitive';

type Ctx = {
  source: string;              // 'rsc-boundary' | 'rq-mutation' | 'server-action' | ...
  extra?: Record<string, unknown>;
  userId?: string;             // opaque id only
};

type LogPayload = {
  ts: string;
  level: 'error';
  source: string;
  userId?: string;
  message: string;
  name?: string;
  stack?: string;
  extra?: Record<string, unknown>;
};

export function reportError(err: unknown, ctx: Ctx): void {
  const payload: LogPayload = {
    ts: new Date().toISOString(),
    level: 'error',
    source: ctx.source,
    userId: ctx.userId,
    message: err instanceof Error ? err.message : String(err),
    name: err instanceof Error ? err.name : undefined,
    stack: err instanceof Error ? err.stack : undefined,
    extra: maskSensitive(ctx.extra),
  };

  // 지금: console.error. 추후 자체 수집 엔드포인트 생기면 조건부 fetch 를 이 함수 안에서만 추가.
  // 호출부(#5 경계·RQ·fetch wrapper) 는 변경 없음.
  console.error(JSON.stringify(payload));
}
```

- **구조화 JSON** → 배포 플랫폼 log drain, 자체 백엔드 엔드포인트, 그리고 나중에 (혹시) 외부 벤더로 바꾸더라도 같은 shape 로 쉽게 매핑.
- `maskSensitive` 가 **최소 안전망** (아래 §2.4).

### 2.3. 자체 백엔드 수집 엔드포인트 (defer)

- 향후 **flowmeet-api 측에** `POST /api/client-errors` 같은 엔드포인트가 생기면, `reportError` 내부에서 **조건부 fetch** 로 전송.
  - 전제: Rate limit, 자체 샘플링, 크기 제한, auth-less (CSRF 보호).
  - 사용자가 로그인한 상태면 `Authorization` 헤더 주입 X — 로그 수집에 토큰 노출 금지. 쿠키 세션 기반이면 자동.
- 지금은 **엔드포인트 미설계** — 설계 시점을 **flowmeet-api 측 이슈** 로 열어 둠 (별도 리포).

### 2.4. 민감정보 마스킹 — 최소 안전망

```ts
// src/lib/errors/maskSensitive.ts
const SENSITIVE_KEYS = /^(password|token|access_?token|refresh_?token|secret|authorization|cookie|email|phone)$/i;

export function maskSensitive(obj: Record<string, unknown> | undefined): Record<string, unknown> | undefined {
  if (!obj) return undefined;
  const out: Record<string, unknown> = {};
  for (const [key, val] of Object.entries(obj)) {
    if (SENSITIVE_KEYS.test(key)) {
      out[key] = '***';
    } else if (val && typeof val === 'object' && !Array.isArray(val)) {
      out[key] = maskSensitive(val as Record<string, unknown>);
    } else {
      out[key] = val;
    }
  }
  return out;
}
```

- "알려진 공통 키" 수준의 **최소 안전망**. 완벽한 PII 분류가 아님.
- **상세 정책 (어떤 필드가 민감한지, 로그 보존 기간, 서버→클라 응답 중 어떤 걸 재로그하면 안 되는지 등)** 은 #19 `web-security` 에서 완결.
- 지금은 로그가 **외부로 나가지 않으므로** 마스킹 미비로 인한 실질 피해 가능성 낮음. 그러나 **지금부터 습관을 들이는** 것이 추후 전송 레이어 추가 시 사고 방지.

### 2.5. 사용자 식별

- 로그 payload 의 `userId` 는 **opaque identifier** 만.
- `AuthProvider` (#3) 가 로그인 시 세션의 `userId` 를 설정, 로그아웃 시 undefined.
- **이메일·이름·전화·역할 등 PII 를 직접 payload 에 넣지 않는다.** 디버깅에 필요하면 `userId` 로 서버 조회.

### 2.6. 서버 로그 (일반 로그)

- **지금**: `console.*` 만. 구조화 로깅 라이브러리 (`pino` 등) 도입하지 않음.
- **다음**: 배포 플랫폼 (Vercel vs AWS vs 자체) 결정 시 재검토.
  - Vercel: 플랫폼 log drain + 구조화 JSON stdout
  - AWS/자체: `pino` + stdout → CloudWatch / ELK 등
- 클라이언트 사이드 **일반 로그 수집은 하지 않는다**. 모든 통찰은 에러 payload 로.

### 2.7. 테스트 (#6 과 연계)

- `reportError` 는 테스트에서 **spy** (Vitest `vi.spyOn(console, 'error')` 또는 wrapper 자체 mock).
- 추상 유지 → 추후 자체 수집 fetch 추가해도 테스트 어떻게 변경? **구현 교체 시 mock 포인트만** 업데이트. 호출부 테스트는 변경 없음.

### 2.8. 재평가 트리거

- **"production blind"** 가 다음 중 하나로 실질 운영 문제가 될 때 축 재검토:
  - 원인 모를 사용자 리포트가 반복
  - 트래픽 급증으로 에러 발생 통계 자체가 필요해짐
  - 성능/UX 저하가 눈에 띄는데 원인 추적 불가
- 재검토 시 선택지:
  1. 자체 백엔드 에러 수집 엔드포인트 (내부)
  2. 자가 호스팅 OSS (GlitchTip 등)
  3. 외부 SaaS (사용자 재확인 필수)

---

## 3. 다른 축에 미치는 영향

| 이슈 | 영향 |
|---|---|
| [#5](https://github.com/timonPark/flowmeet-web/issues/5) 에러 처리 | `reportError` 추상 유지. **구현 교체 전제 삭제 (Sentry 교체 아님)**. 다른 축 결정 문서들의 Sentry 언급은 이 PR 에서 정정. |
| [#6](https://github.com/timonPark/flowmeet-web/issues/6) 테스트 | `reportError` 는 spy. 구현 교체되어도 호출부 테스트 변경 없음. |
| [#9](https://github.com/timonPark/flowmeet-web/issues/9) 운영 규정 | PR 리뷰 체크: "토큰·이메일·개인정보 직접 로깅 금지", "모든 에러 로깅은 `reportError` 경유". |
| [#15](https://github.com/timonPark/flowmeet-web/issues/15) `app-router-layering` | `src/lib/errors/` 디렉터리 책임 포함 (reportError, maskSensitive). |
| [#16](https://github.com/timonPark/flowmeet-web/issues/16) `error-ux` 스킬 | **구현 교체 전제 수정** — Sentry 아닌 "자체 수집 또는 미도입" 전제. |
| [#19](https://github.com/timonPark/flowmeet-web/issues/19) `web-security` | PII 마스킹 상세 정책 완결. `maskSensitive` 의 **키 리스트 확장·중첩 규칙·서버 응답 재로그 금지 패턴** 등. |

---

## 4. 열어둔 포인트 (defer)

| 항목 | 사유 | 어디로 |
|---|---|---|
| 자체 백엔드 에러 수집 엔드포인트 설계·도입 | flowmeet-api 측 설계 필요 | 별도 리포 이슈 |
| 서버 로그 수집·구조화 로깅 (`pino` 등) | 배포 플랫폼 결정 후 | 배포 플랫폼 결정 이슈 |
| RUM / APM | 지금 범위 밖 | 추후 운영 피드백 |
| 상세 PII 마스킹 정책 | 보안 축에서 완결 | #19 |
| 외부 벤더 재평가 | 재평가 트리거 (§2.8) 발동 시, **사용자 명시 요청** 필요 | 사용자 재검토 요청 |

---

## 5. 완료 조건 체크리스트

- [x] 외부 벤더 미도입 명시 (재제안 금지 포함)
- [x] `reportError` 구조화 JSON 구현
- [x] `maskSensitive` 최소 안전망
- [x] 사용자 식별 규칙 (opaque userId)
- [x] 서버 로그·클라 일반 로그 수집 방침
- [x] 테스트 전략 접점 명시
- [x] 재평가 트리거 명시
- [x] 다른 축 영향 매핑
- [x] 열어둔 포인트 이관처
- [ ] CLAUDE.md 축약 반영 + 이 문서 링크 (이 PR)
- [ ] 다른 축 결정 문서의 Sentry 언급 정정 (이 PR)
- [ ] 실제 구현 (이 PR 범위 밖)

## 6. 참고

- [`docs/axis-decisions/error-ux.md`](./error-ux.md) — `reportError` 추상 훅 정의
- [`docs/harness-bootstrap-survey.md`](../harness-bootstrap-survey.md) §2
