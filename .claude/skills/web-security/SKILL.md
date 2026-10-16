---
name: web-security
description: flowmeet-web 보안 가드 — 토큰 저장·전달, XSS/CSRF 방어, PII 마스킹 상세 정책, 민감정보 로깅 금지, CSP·헤더, 입력 검증, 비밀값 관리, 녹음 데이터 보안. 인증·토큰·로깅·외부 리소스 처리 코드 쓸 때 로드. #8 로깅 결정의 PII 마스킹 상세 완결.
---

# Web Security — flowmeet-web

[#2 data-fetching](../../../docs/axis-decisions/data-fetching.md), [#5 error-ux](../../../docs/axis-decisions/error-ux.md), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md), [#17 data-fetching-guard](../data-fetching-guard/SKILL.md) 에서 흩어져 있던 보안 지침을 모으고, 특히 **#8 가 defer 한 PII 마스킹 상세 정책** 을 완결한다. api 리포 `security-logging` 과 같은 자리.

> 보안은 **깊이 방어**. 한 레이어 가 뚫려도 다음 레이어가 막는 구성이어야 한다. 아래는 각 레이어의 실무 규칙.

---

## 1. 토큰 저장·전달 (확정 — #2 결정)

**핵심 모델**: 브라우저 경로와 서버 경로가 **서로 다른 토큰 소스**를 사용한다.

| 토큰 | 저장 위치 | 수명 | 전달 방식 |
|---|---|---|---|
| **Access Token (브라우저용)** | **메모리 (AuthContext)** | 짧게 (예: 15분) | `browserClient` 가 **`Authorization: Bearer` 헤더**로 전달. 쿠키 자동 전송 **금지** (`credentials: 'omit'`). |
| **Access Token (서버용)** | **HttpOnly 쿠키 `access_token`** | 짧게 | `serverClient` 가 `await cookies()` 로 꺼내 **Authorization 헤더** 로 전달. 백엔드엔 쿠키 자체를 보내지 않음. |
| **Refresh Token** | **`__Host-refresh_token` HttpOnly + Secure + SameSite=Lax + Path=/** | 긺 (7-30일), **rotation** | **서버만** 접근. refresh Server Action/Route Handler 가 cookie 를 교체. |

### 1.1. 왜 이 모델인가

- **브라우저 요청 (RQ, `browserClient`)** 는 **Authorization 헤더** 로만 토큰 전달. **쿠키 자동 전송 안 함** → 쿠키 기반 CSRF 벡터 차단. 메모리 저장 유지 이점 (XSS 리스크 완화) 보존.
- **서버 요청 (RSC·Server Action·Route Handler)** 는 쿠키에서 토큰을 읽어 **동일하게 Authorization 헤더로 전달**. 백엔드는 두 경로 모두 **헤더만 신뢰** — 쿠키 기반 자동 인증 금지.
- **CSRF 안전성**: 상태 변경 API 가 쿠키가 아닌 **Authorization 헤더** 로 인증하므로 cross-site POST 가 자동으로 인증되지 않음. SameSite=lax 는 refresh token 쿠키의 추가 방어막.
- **Access Token 을 `localStorage` 에 저장하면 XSS 로 즉시 탈취** — 금지.
- **Refresh Token 은 HttpOnly 쿠키** — JS 접근 불가, XSS 로 탈취 불가.

### 1.2. 쿠키 설정 (`access_token` + `__Host-refresh_token`)

- 공통 속성:
  - `HttpOnly: true` — JS 접근 불가
  - `Secure: true` — HTTPS 전용 (production)
  - `SameSite: 'lax'` — CSRF 완화 (strict 는 외부 리다이렉트 로그인 UX 깨짐)
  - `Path: '/'`
- Refresh token 쿠키는 **`__Host-` 접두어** 사용:
  - 쿠키 이름 `__Host-refresh_token` → 브라우저가 `Domain` 미설정 + `Path=/` + `Secure` 를 **강제 검증**. 서브도메인 쿠키 주입 공격 방어.
- 수명:
  - `access_token`: 짧게 (15분 ~ 1시간)
  - `__Host-refresh_token`: 7-30일, **rotation 때마다 갱신**
- **Rotation reuse detection**: 백엔드가 이전 refresh token 의 재사용 시도를 탐지하면 **해당 세션 전체 폐기** — 토큰 탈취 시 완화.
- 쿠키 **쓰기**는 **Server Action / Route Handler 에서만** (`const cookieStore = await cookies(); cookieStore.set(...)`). RSC 는 **읽기 전용**.

### 1.3. Next 16 `cookies()` 는 비동기

```ts
// ✅ Next 16
import { cookies } from 'next/headers';

export async function getAccessToken() {
  const cookieStore = await cookies();   // ← await 필수
  return cookieStore.get('access_token')?.value;
}

// Server Action 안에서 쿠키 쓰기
export async function setAccessTokenAction(token: string) {
  const cookieStore = await cookies();
  cookieStore.set('access_token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 60 * 15,  // 15분
  });
}
```

- Next 14 이하 syntax (`cookies().get(...)` 직접) 는 **Next 16 에서 틀림**.
- 쿠키 쓰기는 **응답 헤더 조작** 가능한 context (Action/Route Handler) 에서만. **RSC 안에서 cookies().set() 호출 금지** — 런타임 에러.

### 1.4. 금지 패턴

- `localStorage.setItem('access_token', ...)` / `sessionStorage` — **금지**
- `document.cookie` 로 토큰 설정 — HttpOnly 없는 쿠키라 XSS 노출. 모든 쿠키는 **서버가 Set-Cookie** 로.
- URL query string 에 토큰 — 리퍼러·히스토리·로그 노출
- 토큰을 React Query 캐시에 저장 — RQ dev-tools 가 노출
- Access Token 을 로그에 남김 — `reportError` 가 `maskSensitive` 로 자동 제거하되 **애당초 extra 에 넣지 말 것**
- `browserClient` 가 `fetch(url, { credentials: 'include' })` 로 쿠키 전송 — Authorization 헤더만 사용 (`credentials: 'omit'` 명시)
- `cookies().get(...)` 를 await 없이 — Next 14 syntax, Next 16 에서 틀림
- RSC 안에서 `cookies().set(...)` — 런타임 에러

---

## 2. XSS / CSRF 방어

### 2.1. XSS 방어

React 는 기본 escape 하므로 기본적으로 안전. 다만 아래가 **깨뜨리는 지점**:

- **`dangerouslySetInnerHTML`** — 사용 시 **반드시 DOMPurify** 로 sanitize. 사용 지점은 PR 리뷰 체크.
- **`href={userUrl}`** — `javascript:` URL 가능. **URL 화이트리스트** (`https:` / `mailto:` / 상대경로) 로 가드.
- **`<iframe src={userUrl}>`** — 동일 위험. 외부 임베드는 `sandbox` 속성 + CSP `frame-src` 제한.
- **스타일 injection** — inline `style={{...}}` 는 안전, `innerHTML` 로 `<style>` 금지.
- **MSW·Vitest 환경에서는 dev 전용** — production 번들에 포함 금지 (`import 'msw'` 가 production code 에 들어가면 bundle inflate + 보안 평가 영역 확장).

### 2.2. CSRF 방어 (레이어별)

**이 프로젝트의 1차 방어는 "쿠키 자동 인증 안 함"**. §1 모델이 CSRF 벡터 자체를 좁힌다. 그 위에:

| 레이어 | 방어 |
|---|---|
| 백엔드 API 호출 (`browserClient`·`serverClient`) | **Authorization 헤더 인증만**. 쿠키 자동 전송 금지 → cross-site POST 가 자동 인증 안 됨. |
| Refresh token 쿠키 | `SameSite: 'lax'` + `__Host-` 접두어 → cross-site 전송 거의 차단 |
| **Server Actions** | Next 가 **`Next-Action` 헤더 + Origin 검증 + `allowedOrigins`** 설정으로 가드. **단, 완전 방어 아님** — 프록시 환경·구조에 따라 Origin 검증이 우회되거나 미작동 가능. `next.config.ts` `experimental.serverActions.allowedOrigins` 명시 권장. |
| 상태 변경 Route Handler (`route.ts` POST/PUT/DELETE) | **Server Action 보다 가드 약함**. 수동으로 Origin 헤더 검증·CSRF 토큰 필요 시 추가. **가능하면 Server Action 쪽으로 통일**. |

**도입 보류 (요구 생기면)**:
- CSRF 토큰 (double-submit cookie) — 쿠키 자동 인증을 쓰지 않으면 대부분 불필요. Route Handler 가 쿠키 인증을 쓰면 추가 검토.

**금지**:
- Route Handler 가 쿠키만으로 상태 변경 — Authorization 헤더 검증 또는 Origin 체크 필수
- `experimental.serverActions.allowedOrigins` 미설정 상태로 production (dev 는 자동 완화됨)

### 2.3. 금지 패턴

- `dangerouslySetInnerHTML={{ __html: userContent }}` 를 sanitize 없이 — DOMPurify 필수
- `<a href={userUrl}>` 를 scheme 검증 없이
- `window.open(userUrl)` — scheme 검증 + `noopener,noreferrer`
- `postMessage` 수신 시 `event.origin` 검증 생략
- CORS `Access-Control-Allow-Origin: *` + credentials 사용 시도 (브라우저가 거부하지만 설계 자체가 틀림)

---

## 3. PII 마스킹 상세 (#8 완결)

[#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md) §2.4 가 **최소 안전망** 수준의 `maskSensitive` 를 정의하고 **상세는 #19 로 이관**. 이 섹션이 그 상세.

### 3.1. 통합 `maskSensitive` 구현

```ts
// src/lib/errors/maskSensitive.ts
const SENSITIVE_KEYS = [
  // 인증
  /password|passwd|pwd/i,
  /token|bearer/i,                              // access_?token, refresh_?token, id_?token, tokenValue, accessTokenValue 등 변형 포함
  /secret|apikey|api_?key|private_?key/i,
  /authorization|cookie|set-?cookie/i,
  /session_?id|sid/i,
  // 개인정보 (PII) — 변형 포괄 (userEmail, phoneNumber, billingAddress 등)
  /e_?mail/i,
  /phone|mobile|tel\b|contact_?number/i,
  /ssn|social_?security|rrn|resident_?number/i,
  /dob|birth_?date|birthday/i,
  /credit_?card|cc_?number|card_?number|cvv|cvc/i,
  /account_?number|iban|bic|swift|routing/i,
];

// 변형이 다수인 "일반 이름" (name·address 등) 은 blocklist 로 일괄 마스킹하면 디버깅 손실 큼.
// → PII 필드가 로그에 들어가지 않도록 **extra 작성 측** 에서 애당초 제외 (allowlist 성향).
// 자동 마스킹은 "탈취 시 즉시 피해 큰" 값만 커버.

const VALUE_PATTERNS: Array<[RegExp, string]> = [
  [/eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+/g, '[JWT]'],
  [/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b/g, '[EMAIL]'],
  [/\b01[0-9]-?\d{3,4}-?\d{4}\b/g, '[PHONE]'],
  [/\b\d{6}-[1-4]\d{6}\b/g, '[RRN]'],
];
// 카드번호: 오탐 심해 (주문번호·타임스탬프) → 값 패턴 미적용. 키 매칭 (card_number 등) 에 의존.

const MAX_DEPTH = 10;
const MAX_STRING = 2000;
const MAX_ARRAY = 100;

function maskString(s: string): string {
  const trimmed = s.length > MAX_STRING ? s.slice(0, MAX_STRING) + '…[truncated]' : s;
  let out = trimmed;
  for (const [re, repl] of VALUE_PATTERNS) out = out.replace(re, repl);
  return out;
}

export function maskSensitive(obj: unknown, seen = new WeakSet<object>(), depth = 0): unknown {
  if (obj === null || obj === undefined) return obj;
  if (typeof obj === 'string') return maskString(obj);
  if (typeof obj !== 'object') return obj;
  if (depth > MAX_DEPTH) return '[DEPTH_EXCEEDED]';

  // 순환 참조 가드
  if (seen.has(obj as object)) return '[CIRCULAR]';
  seen.add(obj as object);

  // Error 는 message/name/stack/cause 만 선별
  if (obj instanceof Error) {
    return {
      name: obj.name,
      message: maskString(obj.message),
      stack: obj.stack ? maskString(obj.stack) : undefined,
      cause: obj.cause !== undefined ? maskSensitive(obj.cause, seen, depth + 1) : undefined,
    };
  }

  // Date / Map / Set 은 toString 또는 생략
  if (obj instanceof Date) return obj.toISOString();
  if (obj instanceof Map || obj instanceof Set) return `[${obj.constructor.name} size=${obj.size}]`;

  if (Array.isArray(obj)) {
    const slice = obj.length > MAX_ARRAY ? obj.slice(0, MAX_ARRAY) : obj;
    const mapped = slice.map((v) => maskSensitive(v, seen, depth + 1));
    return obj.length > MAX_ARRAY ? [...mapped, `[...+${obj.length - MAX_ARRAY} more]`] : mapped;
  }

  const out: Record<string, unknown> = {};
  // getter 평가 중 throw 가드
  for (const key of Object.keys(obj as Record<string, unknown>)) {
    let val: unknown;
    try {
      val = (obj as Record<string, unknown>)[key];
    } catch {
      out[key] = '[GETTER_THREW]';
      continue;
    }
    if (SENSITIVE_KEYS.some((re) => re.test(key))) {
      out[key] = '***';
    } else {
      out[key] = maskSensitive(val, seen, depth + 1);
    }
  }
  return out;
}
```

**핵심 개선** (vs 초기 안):
- **문자열 값 패턴을 실제 호출** — JWT·이메일·전화번호·주민번호 자동 마스킹.
- **순환 참조 (WeakSet) 가드** — `[CIRCULAR]`.
- **`Error` 처리** — `message`/`name`/`stack`/`cause` 선별, 다른 enumerable 속성 drop.
- **Date / Map / Set** 별도 처리 — 재귀 폭주 방지.
- **getter throw 가드** — `[GETTER_THREW]`.
- **크기 제한** — 깊이 `MAX_DEPTH` / 배열 길이 `MAX_ARRAY` / 문자열 길이 `MAX_STRING`.
- **카드번호 값 패턴 제거** — Luhn 검증 없는 범위 매칭은 오탐 심함 (주문번호·타임스탬프 손상). 키 매칭에만 의존.
- **일반 이름 필드 (`name`·`address`·`city`) 제외** — blocklist 로 일괄 마스킹하면 디버깅 손실 큼. 작성 측 (extra 넣는 곳) 에서 애당초 PII 제외 책임.

**적용 범위**: `reportError(err, { extra })` 호출의 `extra` 와 `err` 자체. 모든 호출이 `maskSensitive` 를 통과하도록 `reportError.ts` 안에서 1회 적용 ([#8](../../../docs/axis-decisions/logging-monitoring.md) §2.2 참조).

### 3.2. 테스트 요구사항 (`maskSensitive.test.ts` 에서 커버)

- 문자열 안 JWT·이메일 치환 확인
- 순환 참조 → `[CIRCULAR]`
- Error.cause → 재귀 마스킹
- Date → ISO string
- 큰 배열 → `[...+N more]`
- 깊이 폭발 → `[DEPTH_EXCEEDED]`
- getter throw → `[GETTER_THREW]`

### 3.3. 서버 응답을 로그에 재덤프 금지

- `reportError(err, { extra: { response: await res.json() } })` 같이 **서버 응답 전체** 를 extra 에 넣는 패턴 **금지**.
- 서버 응답은 **PII 가 섞여 있을 가능성** 이 매우 높음. 디버깅에 필요한 **최소 정보** 만 (status code, error code, requestId 등) 추출.
- `ApiError.cause` 에는 서버 bodyJSON 전체가 들어 있을 수 있음 — `reportError` 에 `err` 자체를 넘길 때는 **cause 를 함께 로깅하지 말고** 명시 field 만.

### 3.4. 녹음·업로드 파일 (TRD §4.1)

- 녹음 chunk 는 **원 사용자 음성** — 민감 데이터. 로그에 **binary body 넣기 금지**.
- 업로드 재시도 시도 횟수·실패 코드 등 **메타만** 로그.
- IndexedDB 저장은 **사용자 디바이스 로컬** → 디바이스 공격 시 유출 가능. 녹음 종료 시 서버 업로드 완료 확인 후 **IndexedDB 비움** (업로드 성공 전까진 유지, 성공 즉시 삭제).

### 3.5. 의도적 로깅 가능 필드 (allowlist 대안 — 보류)

> 현재 전략은 **blocklist** (SENSITIVE_KEYS). 미래에 PII 엄격 필요 시 **allowlist** 로 전환 — "명시 허용된 필드만 로깅, 나머지는 전부 `[REDACTED]`". 지금은 blocklist 가 실용적.

---

## 4. 입력 검증 (zod — #4 결정과 정합)

**우선순위** (전부 균등하게 검증하면 실무 적용 떨어짐 + "safeParse 는 호출됐지만 결과를 쓰지 않는" 형식적 준수 유도):

| 우선순위 | 대상 | 이유 |
|---|---|---|
| **필수** | **상태 변경 Server Action 입력** | 사용자 입력, 공격 벡터 1차 |
| **필수** | **보안 결정 필드** (userId, role, teamId, 결제금액, 권한 플래그 등) 서버→클라 응답 | 신뢰 경계 통과 지점 |
| **권장** | **인증 경계 입력** (login, refresh 요청) | 공격 대상 |
| **권장** | **URL params** 중 라우팅 결과를 좌우하는 값 (id, slug 등) | 조작 쉬움 |
| 선택 | 일반 조회 응답 전체 | zod 가 TS 타입 보장 수단으로 유용하면 적용. 전수 검증은 과함. |

**예시**:
```ts
// Server Action 상단 — 필수 (#4)
const parsed = createMeetingSchema.safeParse(input);
if (!parsed.success) return mapZodErrorToFormResult(parsed.error);

// URL params — 권장
const paramsSchema = z.object({ id: z.string().uuid() });
const p = paramsSchema.safeParse(await params);
if (!p.success) notFound();

// 보안 결정 필드만 — 필수
const sessionSchema = z.object({ userId: z.string(), role: z.enum(['admin', 'member']) });
const session = sessionSchema.parse(await fetchMySession());  // 보안 결정이므로 throw
```

**금지**:
- `safeParse` 호출만 하고 **결과 `parsed.data` 를 쓰지 않는** 형식적 준수
- 모든 응답에 대규모 스키마 적용 (유지보수 폭탄)

---

## 5. CSP (Content Security Policy)

### 5.1. 기본 정책 (방향, 실제 설정은 검증 후)

**중요**: CSP nonce 는 **요청별 생성 + HTML inline script 주입** 이 필요. `next.config.ts` 의 정적 `headers()` 에 nonce 문자열을 하드코딩하는 건 **방어 효과 없음** (틀린 설정). Next 16 에서 공식 nonce 패턴은 `src/proxy.ts` 에서 `crypto.randomUUID()` 로 nonce 생성 → `response.headers.set('x-nonce', nonce)` + CSP 헤더 조립 → React 가 nonce 를 inline script 에 주입. 단, 동적 nonce 는 **PPR·캐싱과 영향** 이 있어 공식 가이드 확인 후 설정.

- **초기 권장 방향** (실제 설정은 Next 16 공식 docs 확인 후 별도 chore PR):
  ```
  default-src 'self';
  script-src 'self' 'strict-dynamic' 'nonce-{per-request}';
  style-src 'self';                    // Tailwind 는 클래스 기반 — 'unsafe-inline' 불필요
  img-src 'self' data: https:;
  font-src 'self' data:;
  connect-src 'self' ${NEXT_PUBLIC_API_BASE_URL};
  frame-ancestors 'none';              // clickjacking 방어
  base-uri 'self';
  form-action 'self';
  ```
- **`style-src 'unsafe-inline'` 디폴트 금지** — Tailwind 유틸은 클래스로 처리되고 inline style 을 요구하지 않음. 특정 서드파티·Next 런타임이 inline style 을 쓰면 그때 **구체적 nonce 또는 hash 로 좁혀서 허용**.
- 초기엔 `Content-Security-Policy-Report-Only` 로 **report 모드** 로 시작 → 실제 영향 확인 후 enforce.
- Report endpoint 자체도 **PII 유출 금지** — Next 16 가 자동 report 하는 규격 외 데이터는 받지 않음.

### 5.2. 추가 보안 헤더

- `X-Frame-Options: DENY` (중복이지만 old browser 보강)
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy` — camera/microphone/geolocation 중 **필요한 것만 허용** (녹음용 microphone 은 self).

### 5.3. 금지

- `script-src 'unsafe-eval'` — 성능 저하 + XSS 위험 증가
- `frame-ancestors *` — clickjacking 허용
- CSP 를 전혀 안 거는 상태 운영 (production)

---

## 6. 비밀값 관리

### 6.1. 환경 변수 네이밍 (Next 규약)

- **`NEXT_PUBLIC_*`** — 브라우저 번들에 포함. **공개 가능한 값만** (예: `NEXT_PUBLIC_API_BASE_URL`).
- **`NEXT_PUBLIC_` 없는** 변수 — 서버 전용. 비밀값 (API 키, 서버 간 secret) 은 이쪽.
- **금지**: `NEXT_PUBLIC_API_SECRET`, `NEXT_PUBLIC_JWT_SECRET` 같은 네이밍 — 번들에 포함되어 **공개 노출**.

### 6.2. `.env.local` / `.env.production`

- `.env*` 는 **`.gitignore`** (현재 이미 처리).
- `.env.local.example` 로 **키 이름만** 공유. 값 공유는 1Password/Vault 등 외부.
- 로컬 개발 시 production 비밀 대신 **스테이징 또는 dev 전용** 값 사용.

### 6.3. 런타임 접근 가드

```ts
// src/lib/env.ts
import 'server-only';

function requireEnv(name: string): string {
  const v = process.env[name];
  if (!v) throw new Error(`Missing env: ${name}`);
  return v;
}

export const SERVER_ENV = {
  JWT_SECRET: requireEnv('JWT_SECRET'),
  // ...
};
```

- 서버 전용 비밀은 `server-only` import 로 **클라 번들링 방지**.
- 누락 시 **런타임 명시 에러** — 조용히 undefined 로 흘러가지 않게.

---

## 7. 외부 리소스·iframe·postMessage

### 7.1. 외부 이미지 (`<Image>`)

- `next/image` 는 `remotePatterns` 에 선언한 도메인만 허용.
- 사용자 컨텐츠 이미지는 **백엔드가 proxy** 하거나 **CDN 화이트리스트** 로.

### 7.2. iframe

- 외부 임베드가 있으면 **`sandbox` 속성** 필수:
  ```tsx
  <iframe src={externalUrl} sandbox="allow-scripts allow-same-origin" />
  ```
- `sandbox` 없이는 iframe 안 스크립트가 부모 페이지 쿠키·스토리지 접근 가능 (same-origin 인 경우).

### 7.3. `postMessage`

- 수신 시 `event.origin` 반드시 **화이트리스트 검증**:
  ```ts
  window.addEventListener('message', (e) => {
    if (e.origin !== 'https://trusted.example.com') return;
    // ...
  });
  ```

---

## 8. 녹음·업로드 보안 (TRD §4.1)

### 8.1. 권한·저장

- **MediaRecorder 접근 전 `getUserMedia` 권한** 명시적으로 사용자에게 요청.
- 녹음 데이터는 **IndexedDB** 에 임시 저장 — localStorage 금지 (크기 제한 + 보안 경계 약함).
- 녹음 중 **다른 탭·배경 녹음 가능성** 명시적으로 UI 로 알림 (사용자 동의).

### 8.2. 업로드 흐름 안전성

- **chunk 식별자 + 멱등성 키** — 각 chunk 에 `chunkId` (UUID) 와 `idempotencyKey` 를 포함. 중복 전송·재시도가 **같은 chunk 를 두 번 저장하지 않도록** 백엔드가 보장.
- **Presigned URL 조건**:
  - 짧은 만료 시간 (예: 5분)
  - `Content-Type` 제한 (`audio/webm` 등 명시)
  - `Content-Length` 상한 (chunk 크기 상한)
  - 특정 사용자·세션 범위
- **업로드 완료 기준**: **단순 PUT 2xx 가 아니라 백엔드가 "최종 조립·검증 완료" 신호** (예: `/sessions/:id/finalize` 성공 응답) 를 받은 뒤 IndexedDB 비움.
- **부분 실패 처리**: chunk 1~N 중 일부만 성공한 상태에서 세션 중단 시 재개 가능하도록 chunk 상태 저장 (`uploaded` / `pending` / `failed`).
- **IndexedDB 삭제 실패** 는 다음 세션 시작 시 **오래된 chunk gc** 로 처리 (예: 7일 지난 chunk 는 자동 삭제).

### 8.3. 네트워크

- 서버 업로드는 **HTTPS 전용**.
- Presigned URL 호출은 `credentials: 'omit'` — 외부 S3 등으로 가는 요청에 쿠키 노출 금지.

### 8.4. 금지

- 녹음 chunk 를 로그에 (binary 든 base64 든) 포함
- chunk id 없이 "순서 번호" 만으로 중복 가드 — 재시도 시 번호 중복 가능
- Presigned URL 만료 체크 없이 재시도 (만료 URL 로 403 반복)

---

## 9. 의존성 보안

### 9.1. Lockfile 커밋

- `package-lock.json` 커밋 — 재현 가능 빌드. **매 환경에서 같은 버전** 설치.
- `npm ci` 로 CI 설치 (lockfile 강제).

### 9.2. 감사

- 정기적으로 `npm audit` 실행 — CI 에 포함 고려 (현재는 로컬).
- `npm audit fix --force` 는 **destructive** — breaking changes 발생 가능. 수동 리뷰 후 적용.
- 치명적 CVE 발견 시 **긴급 패치 PR**.

### 9.3. 디펜던시 추가 기준

- 번들 사이즈 영향 확인 (bundlephobia).
- 메인테이너·마지막 release 날짜 확인 (abandoned 라이브러리 주의).
- 라이선스 (MIT / Apache 2.0 / ISC 가 안전, GPL 류는 재검토).

---

## 10. Server Action 보안

### 10.1. 권한 검사는 **서버에서**

- CLAUDE.md 설계 원칙: "권한 검사는 백엔드가 수행". 프런트는 UI 가드 용도.
- Server Action 은 **백엔드 호출을 조립** 하는 게 주 역할. **독자 권한 로직 금지** — 모두 백엔드에 위임.
- Server Action 자체도 세션 확인 필수:
  ```ts
  export async function createMeetingAction(input: unknown) {
    const session = await getServerSession();  // 세션 확인
    if (!session) redirect('/login');
    // 이후 schema 검증 + 백엔드 호출
  }
  ```

### 10.2. 반환 데이터 노출 제어

- Server Action 반환 shape (`ActionResult<T>`) 에 **민감정보 포함 금지**.
- 백엔드가 `password_hash` 등을 응답에 섞어 보내도 **프런트가 선별 반환**.

---

## 11. 체크리스트 — 보안 관련 코드 작성·리뷰 시

- [ ] Access/Refresh Token 저장이 **메모리/HttpOnly 쿠키** 분리 규약 지키는가? localStorage 금지?
- [ ] 브라우저 요청이 **Authorization 헤더만** 사용하는가 (`credentials: 'omit'`)? 쿠키 자동 전송 금지?
- [ ] 쿠키에 `HttpOnly`·`Secure`·`SameSite: lax`·`Path: /` 세팅됐는가? Refresh token 에 `__Host-` 접두어?
- [ ] `cookies()` 를 **`await cookies()`** 로 사용 (Next 16 비동기 API)?
- [ ] `cookies().set(...)` 을 **Server Action/Route Handler 에서만** (RSC 아님)?
- [ ] Server Actions 의 `experimental.serverActions.allowedOrigins` 설정됐는가?
- [ ] 사용자 입력을 `dangerouslySetInnerHTML` 로 쓸 때 **DOMPurify sanitize** 됐는가?
- [ ] `href={userUrl}` / `window.open(userUrl)` 가 **scheme 화이트리스트** 로 가드되는가?
- [ ] `postMessage` 수신이 `event.origin` 화이트리스트 체크하는가?
- [ ] 로그에 **토큰·이메일·전화·주민번호 등** 가 들어가지 않는가? `reportError` 가 `maskSensitive` 통과시키는가?
- [ ] `maskSensitive` 가 **문자열 값 패턴 (JWT·이메일·전화) 을 실제로 적용** 하는가? 순환 참조 가드? Error.cause 처리?
- [ ] `reportError` 의 `extra` 에 **서버 응답 전체** 를 넣지 않았는가? 필요한 field 만 추출?
- [ ] 녹음 chunk 등 **binary 데이터를 로그에 넣지 않았는가**? chunk id 와 멱등 키로 중복 가드?
- [ ] zod `safeParse` 가 **외부 입력** (사용자, API 응답 보안 필드, URL params) 에 적용되는가?
- [ ] 환경변수 네이밍이 **`NEXT_PUBLIC_` 접두어의 의미** 와 일치하는가? 비밀값에 접두어 금지?
- [ ] 서버 전용 모듈이 **`import 'server-only'`** 를 선언했는가?
- [ ] iframe 에 `sandbox` 속성이 있는가? 외부 이미지가 `remotePatterns` 화이트리스트인가?
- [ ] 녹음 플로우가 **백엔드 finalize 신호 받은 뒤 IndexedDB 비움** 하는가 (PUT 2xx 만으로는 부족)? presigned URL 만료·Content-Type·Content-Length 제한?
- [ ] `package-lock.json` 커밋됐는가? CI 가 `npm ci` 사용?
- [ ] Server Action 이 **세션 확인** 후 로직 실행하는가?

---

## 12. 참고

- 결정 축: [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md) (토큰 저장), [#5 error-ux](../../../docs/axis-decisions/error-ux.md) (ApiError·reportError), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md) (maskSensitive 최소안전망)
- 관련 스킬: [`data-fetching-guard`](../data-fetching-guard/SKILL.md) §3·§4 (serverClient·browserClient 토큰 흐름 + single-flight refresh), [`error-ux`](../error-ux/SKILL.md) §8 (reportError 호출 규약), [`app-router-layering`](../app-router-layering/SKILL.md) §4.3 ('use cache' 인증 경계)
- TRD §6 (인증), §11 (보안), §4.1 (녹음)
- CLAUDE.md "설계 원칙 (TRD 발췌)" — 권한 검사는 백엔드가 수행
- OWASP Top 10 2024, OWASP Secure Headers Project
