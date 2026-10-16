---
name: web-security
description: flowmeet-web 보안 가드 — 토큰 저장·전달, XSS/CSRF 방어, PII 마스킹 상세 정책, 민감정보 로깅 금지, CSP·헤더, 입력 검증, 비밀값 관리, 녹음 데이터 보안. 인증·토큰·로깅·외부 리소스 처리 코드 쓸 때 로드. #8 로깅 결정의 PII 마스킹 상세 완결.
---

# Web Security — flowmeet-web

[#2 data-fetching](../../../docs/axis-decisions/data-fetching.md), [#5 error-ux](../../../docs/axis-decisions/error-ux.md), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md), [#17 data-fetching-guard](../data-fetching-guard/SKILL.md) 에서 흩어져 있던 보안 지침을 모으고, 특히 **#8 가 defer 한 PII 마스킹 상세 정책** 을 완결한다. api 리포 `security-logging` 과 같은 자리.

> 보안은 **깊이 방어**. 한 레이어 가 뚫려도 다음 레이어가 막는 구성이어야 한다. 아래는 각 레이어의 실무 규칙.

---

## 1. 토큰 저장·전달 (확정 — #2 결정)

| 토큰 | 저장 위치 | 수명 | 접근 가능 지점 |
|---|---|---|---|
| **Access Token** | **클라이언트 메모리 (AuthContext)** | 짧게 (예: 15분) | `"use client"` 안에서 `getAccessToken()`, 서버는 쿠키 (아래 참조) |
| **Refresh Token** | **HttpOnly + Secure + SameSite=Lax Cookie** | 긺 (예: 7-30일), 회전 (rotation) | **서버만** (`cookies()` 또는 fetch 시 자동 전송) |
| **CSRF Token** (필요 시) | 쿠키 + 응답 헤더 double-submit | 요청별 | 상태 변경 요청에 명시 포함 |

### 1.1. 왜 이 조합인가

- **Access Token 을 localStorage 에 저장하면 XSS 로 즉시 탈취** — 모든 서드파티 스크립트가 읽을 수 있음.
- **메모리 저장은 XSS 리스크 완화** — 탈취하려면 JS 실행 중에 메모리 접근 필요 (여전히 가능하지만 시간창 짧음, 페이지 리로드로 초기화).
- **Refresh Token 을 HttpOnly 쿠키에 두면 JS 가 못 읽음** — XSS 로 탈취 불가. 다만 **CSRF 리스크** 는 생기므로 아래 §2 CSRF 방어 필수.

### 1.2. Access Token 서버 저장 — 쿠키 사용

- `serverClient` 가 `cookies()` 로 **access_token 도 쿠키에서** 읽음 ([#17](../data-fetching-guard/SKILL.md) §3). → 서버에선 쿠키, 클라에선 메모리.
- 쿠키 설정 공통:
  - `HttpOnly: true` — JS 접근 불가
  - `Secure: true` — HTTPS 전용 (production)
  - `SameSite: 'lax'` — CSRF 완화 (strict 는 외부 리다이렉트 로그인 UX 깨짐)
  - `Path: '/'`
  - access_token 수명 짧게, refresh_token 수명 길게
- 쿠키 설정 자체는 **Server Action 또는 Route Handler** 에서 (`cookies().set(...)`) — RSC 에선 쓰기 불가.

### 1.3. 금지 패턴

- `localStorage.setItem('access_token', ...)` / `sessionStorage` — **금지**
- `document.cookie` 로 토큰 설정 — HttpOnly 없는 쿠키라 XSS 노출. 모든 쿠키는 **서버가 Set-Cookie** 로.
- URL query string 에 토큰 — 리퍼러·히스토리·로그 노출
- 토큰을 React Query 캐시에 저장 — RQ dev-tools 가 노출
- Access Token 을 로그에 남김 — `reportError` 가 `maskSensitive` 로 자동 제거하되 **애당초 extra 에 넣지 말 것**

---

## 2. XSS / CSRF 방어

### 2.1. XSS 방어

React 는 기본 escape 하므로 기본적으로 안전. 다만 아래가 **깨뜨리는 지점**:

- **`dangerouslySetInnerHTML`** — 사용 시 **반드시 DOMPurify** 로 sanitize. 사용 지점은 PR 리뷰 체크.
- **`href={userUrl}`** — `javascript:` URL 가능. **URL 화이트리스트** (`https:` / `mailto:` / 상대경로) 로 가드.
- **`<iframe src={userUrl}>`** — 동일 위험. 외부 임베드는 `sandbox` 속성 + CSP `frame-src` 제한.
- **스타일 injection** — inline `style={{...}}` 는 안전, `innerHTML` 로 `<style>` 금지.
- **MSW·Vitest 환경에서는 dev 전용** — production 번들에 포함 금지 (`import 'msw'` 가 production code 에 들어가면 bundle inflate + 보안 평가 영역 확장).

### 2.2. CSRF 방어

- **`SameSite: 'lax'`** 가 **기본 방어** — 대부분의 CSRF 공격 차단.
- **상태 변경 (POST/PUT/PATCH/DELETE) 요청에 추가 가드** 필요한 경우:
  - **Origin 헤더 검증** — 서버가 요청 Origin 이 자기 도메인인지 확인 (API 서버 책임, 프런트는 요청 시 자동 포함)
  - **CSRF 토큰 (double-submit cookie)** — 로그인 플로우가 cross-site redirect 를 쓰면 추가 필요. 지금은 보류, 요구 생기면 도입.
- **Server Actions 는 Next 가 자동 CSRF 가드** — 요청에 `Next-Action` 헤더 포함, 같은 origin 만 허용 (`output: 'standalone'` + `allowedOrigins` 설정 가능).

### 2.3. 금지 패턴

- `dangerouslySetInnerHTML={{ __html: userContent }}` 를 sanitize 없이 — DOMPurify 필수
- `<a href={userUrl}>` 를 scheme 검증 없이
- `window.open(userUrl)` — scheme 검증 + `noopener,noreferrer`
- `postMessage` 수신 시 `event.origin` 검증 생략
- CORS `Access-Control-Allow-Origin: *` + credentials 사용 시도 (브라우저가 거부하지만 설계 자체가 틀림)

---

## 3. PII 마스킹 상세 (#8 완결)

[#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md) §2.4 가 **최소 안전망** 수준의 `maskSensitive` 를 정의하고 **상세는 #19 로 이관**. 이 섹션이 그 상세.

### 3.1. 자동 마스킹 키 리스트 (확장)

```ts
// src/lib/errors/maskSensitive.ts (확장판)
const SENSITIVE_KEYS = [
  // 인증
  /^(password|passwd|pwd)$/i,
  /^(token|access_?token|refresh_?token|id_?token|bearer)$/i,
  /^(secret|api_?key|apikey|private_?key)$/i,
  /^(authorization|auth|cookie|set-cookie)$/i,
  /^(session|session_?id|sid)$/i,

  // 개인정보 (PII)
  /^(email|e_?mail|user_?email)$/i,
  /^(phone|mobile|tel|contact_?number)$/i,
  /^(name|first_?name|last_?name|full_?name|real_?name)$/i,
  /^(address|street|city|zip|postal)$/i,
  /^(ssn|social_?security|rrn|resident_?number)$/i,       // 주민번호
  /^(dob|birth|birthday|birth_?date)$/i,
  /^(credit_?card|cc_?number|card_?number|cvv|cvc)$/i,

  // 결제·금융
  /^(account_?number|iban|bic|swift|routing)$/i,
];

export function maskSensitive(obj: unknown, depth = 0): unknown {
  if (depth > 10) return '[DEPTH_EXCEEDED]';  // 재귀 폭발 방지
  if (obj === null || obj === undefined) return obj;
  if (typeof obj !== 'object') return obj;
  if (Array.isArray(obj)) return obj.map((v) => maskSensitive(v, depth + 1));

  const out: Record<string, unknown> = {};
  for (const [key, val] of Object.entries(obj as Record<string, unknown>)) {
    if (SENSITIVE_KEYS.some((re) => re.test(key))) {
      out[key] = '***';
    } else {
      out[key] = maskSensitive(val, depth + 1);
    }
  }
  return out;
}
```

### 3.2. 값 패턴 마스킹 (키 이름이 평범해도 값이 PII 인 경우)

```ts
// src/lib/errors/maskSensitive.ts (값 패턴)
const VALUE_PATTERNS: Array<[RegExp, string]> = [
  [/eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+/g, '[JWT]'],       // JWT
  [/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b/g, '[EMAIL]'],    // email
  [/\b\d{3}-\d{4}-\d{4}\b/g, '[PHONE]'],                                 // 국내 휴대폰 010-xxxx-xxxx
  [/\b\d{6}-[1-4]\d{6}\b/g, '[RRN]'],                                    // 주민번호 (대략적)
  [/\b(?:\d[ -]*?){13,19}\b/g, '[CARD]'],                                // 카드번호 (범위 검출)
];

function maskStringValue(s: string): string {
  let out = s;
  for (const [re, replacement] of VALUE_PATTERNS) {
    out = out.replace(re, replacement);
  }
  return out;
}
```

- **문자열 필드** 를 maskSensitive 가 통과시킬 때도 **값 패턴 마스킹** 을 적용 — 백엔드가 error message 에 이메일 끼워 보내는 등 edge case 방어.
- 지금 적용 범위: `extra` 와 `stack` trace. 성능 영향 최소화.

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

- **모든 외부 입력 (사용자, API 응답, URL params, cookies 값) 은 zod `safeParse`** 로 통과시킴.
- Server Action **상단 재검증** 필수 ([#4](../../../docs/axis-decisions/form-validation.md) §2.3) — 클라를 신뢰하지 않는다.
- URL params (`params`, `searchParams`) 도 zod 로 가드:
  ```ts
  const paramsSchema = z.object({ id: z.string().uuid() });
  const parsed = paramsSchema.safeParse(await params);
  if (!parsed.success) notFound();
  ```
- API 응답 역시 가드하면 안전 — 다만 전부 가드는 과함. **보안 결정적 필드** (userId, role, teamId, 결제금액 등) 만.

---

## 5. CSP (Content Security Policy)

### 5.1. 기본 정책 (권장 시작점)

- **Next 16** 에서는 `next.config.ts` 의 `headers()` 로 설정, 또는 `src/proxy.ts` 에서 응답 헤더 조작.
- 초기 권장:
  ```
  default-src 'self';
  script-src 'self' 'strict-dynamic' 'nonce-{generated}';
  style-src 'self' 'unsafe-inline';   // Tailwind 는 inline style 사용
  img-src 'self' data: https:;
  font-src 'self' data:;
  connect-src 'self' ${NEXT_PUBLIC_API_BASE_URL};
  frame-ancestors 'none';              // clickjacking 방어
  base-uri 'self';
  form-action 'self';
  ```
- nonce 는 Next 16 가 자동 — 수동 구현 전에 공식 문서 확인.
- 초기엔 `Content-Security-Policy-Report-Only` 로 **report 모드** 로 시작 → 실제 영향 확인 후 enforce.

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

- **MediaRecorder 접근 전 `getUserMedia` 권한** 명시적으로 사용자에게 요청.
- 녹음 데이터는 **IndexedDB** 에 임시 저장 — localStorage 금지 (크기 제한 + 보안 경계 약함).
- 서버 업로드는 **HTTPS 전용**. 업로드 완료 확인 후 IndexedDB 비우기.
- 녹음 중 **다른 탭·배경 녹음 가능성** 명시적으로 UI 로 알림 (사용자 동의).
- 업로드 URL 은 서버에서 **presigned URL** 발급 받아 사용 (백엔드와 조율).

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
- [ ] 쿠키에 `HttpOnly`·`Secure`·`SameSite: lax`·`Path: /` 세팅됐는가?
- [ ] 사용자 입력을 `dangerouslySetInnerHTML` 로 쓸 때 **DOMPurify sanitize** 됐는가?
- [ ] `href={userUrl}` / `window.open(userUrl)` 가 **scheme 화이트리스트** 로 가드되는가?
- [ ] `postMessage` 수신이 `event.origin` 화이트리스트 체크하는가?
- [ ] 로그에 **토큰·이메일·전화·주민번호 등** 가 들어가지 않는가? `reportError` 가 `maskSensitive` 통과시키는가?
- [ ] `reportError` 의 `extra` 에 **서버 응답 전체** 를 넣지 않았는가? 필요한 field 만 추출?
- [ ] 녹음 chunk 등 **binary 데이터를 로그에 넣지 않았는가**?
- [ ] zod `safeParse` 가 **외부 입력** (사용자, API 응답 보안 필드, URL params) 에 적용되는가?
- [ ] 환경변수 네이밍이 **`NEXT_PUBLIC_` 접두어의 의미** 와 일치하는가? 비밀값에 접두어 금지?
- [ ] 서버 전용 모듈이 **`import 'server-only'`** 를 선언했는가?
- [ ] iframe 에 `sandbox` 속성이 있는가? 외부 이미지가 `remotePatterns` 화이트리스트인가?
- [ ] 녹음 플로우가 **업로드 성공 확인 후 IndexedDB 비움** 하는가?
- [ ] `package-lock.json` 커밋됐는가? CI 가 `npm ci` 사용?
- [ ] Server Action 이 **세션 확인** 후 로직 실행하는가?

---

## 12. 참고

- 결정 축: [#2 data-fetching](../../../docs/axis-decisions/data-fetching.md) (토큰 저장), [#5 error-ux](../../../docs/axis-decisions/error-ux.md) (ApiError·reportError), [#8 logging-monitoring](../../../docs/axis-decisions/logging-monitoring.md) (maskSensitive 최소안전망)
- 관련 스킬: [`data-fetching-guard`](../data-fetching-guard/SKILL.md) §3·§4 (serverClient·browserClient 토큰 흐름 + single-flight refresh), [`error-ux`](../error-ux/SKILL.md) §8 (reportError 호출 규약), [`app-router-layering`](../app-router-layering/SKILL.md) §4.3 ('use cache' 인증 경계)
- TRD §6 (인증), §11 (보안), §4.1 (녹음)
- CLAUDE.md "설계 원칙 (TRD 발췌)" — 권한 검사는 백엔드가 수행
- OWASP Top 10 2024, OWASP Secure Headers Project
