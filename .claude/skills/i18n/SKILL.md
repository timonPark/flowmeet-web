---
name: i18n
description: flowmeet-web 국제화 가드 — next-intl v3, ko 로케일, 메시지 네임스페이스, 포맷 유틸, zod/에러 메시지 점진 전환. 사용자 가시 문자열·날짜/숫자 포맷 코드 쓸 때 로드. #7 결정의 i18n 축 실무 적용.
---

# i18n — flowmeet-web

[#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) §B 결정을 코드로 적용. 초기 로케일 1개 (`ko`) 지만 **키-값 분리는 지금** — 미래 다국어 비용 10배 절감.

---

## 1. 라이브러리 — next-intl v3

- App Router + Server Components + Server Actions 공식 지원.
- 설치: `npm install next-intl`.
- 비-라우트 모드 (초기 로케일 1개라 `/[locale]/...` 미사용).

---

## 2. `<html lang="ko">` — 버그 교정 필수

`src/app/layout.tsx` 상단:

```tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

- 현재 `"en"` 인 것은 **접근성·SEO 버그** — 스크린리더가 영어 발음 규칙 적용. 교정 필수.

---

## 3. 메시지 조직

### 3.1. 파일

```
src/messages/
└─ ko.json
```

### 3.2. 네임스페이스 (최상위 키)

```json
{
  "common": {
    "save": "저장", "cancel": "취소", "delete": "삭제", "confirm": "확인", "edit": "편집"
  },
  "nav": {
    "dashboard": "대시보드", "meeting": "회의", "directive": "지시", "settings": "설정"
  },
  "auth": {
    "email": "이메일", "password": "비밀번호", "login": "로그인", "logout": "로그아웃", "changePassword": "비밀번호 변경"
  },
  "meeting": {
    "create": {
      "title": "회의 생성",
      "titleField": "회의 제목",
      "startAt": "시작 시각",
      "room": "회의실"
    },
    "status": { "draft": "준비", "ongoing": "진행 중", "ended": "종료" }
  },
  "dashboard": {
    "delayedTasks": "지연 업무",
    "assignee": "담당자",
    "dueDate": "기한",
    "overdueDays": "지연 일수"
  },
  "recording": {
    "start": "녹음 시작", "pause": "일시정지", "resume": "다시 시작", "stop": "종료",
    "uploadProgress": "{percent}% 업로드 중"
  },
  "speakerMatch": {
    "unassigned": "미지정", "matchTo": "{name} 로 매칭"
  },
  "errors": {
    "AUTH_INVALID": "로그인 정보가 올바르지 않습니다.",
    "AUTH_SESSION_EXPIRED": "로그인이 만료되었습니다. 다시 로그인해 주세요.",
    "FORBIDDEN": "접근 권한이 없습니다.",
    "MEETING_NOT_FOUND": "존재하지 않는 회의입니다.",
    "UNKNOWN": "오류가 발생했습니다. 잠시 후 다시 시도해 주세요."
  },
  "validation": {
    "required": "필수 항목입니다",
    "email": "올바른 이메일 형식이 아닙니다",
    "minLength": "{min}자 이상 입력해 주세요"
  }
}
```

### 3.3. 네이밍 규약

- 최상위 네임스페이스는 **도메인** (`meeting`, `dashboard`) 또는 **공통 영역** (`common`, `nav`, `errors`, `validation`).
- 중첩은 **의미 중심** (`meeting.create.title`). 화면/컴포넌트 이름 기반 (`meetingCreatePage.saveButton`) 지양 — UI 리팩터시 깨짐.
- 변수는 **`{name}`** 식 ICU placeholder.
- 복수형은 ICU `plural` (예: `"{count, plural, one {# 명} other {# 명}}"` — 한국어는 복수형 구분 거의 없어 생략 가능).

---

## 4. 사용 패턴

### 4.1. Server Component

```tsx
import { getTranslations } from 'next-intl/server';

export default async function MeetingCreatePage() {
  const t = await getTranslations('meeting.create');
  return <h1>{t('title')}</h1>;
}
```

### 4.2. Client Component

```tsx
'use client';
import { useTranslations } from 'next-intl';

export function CreateMeetingForm() {
  const t = useTranslations('meeting.create');
  return <button>{t('title')}</button>;
}
```

### 4.3. ICU 변수

```tsx
const t = useTranslations('recording');
return <span>{t('uploadProgress', { percent: 42 })}</span>;
// → "42% 업로드 중"
```

---

## 5. 날짜·숫자·통화 포맷

### 5.1. 공용 유틸

```ts
// src/lib/i18n/format.ts
export function formatDate(date: Date | string, opts?: Intl.DateTimeFormatOptions): string {
  const d = typeof date === 'string' ? new Date(date) : date;
  return new Intl.DateTimeFormat('ko-KR', opts ?? { dateStyle: 'medium' }).format(d);
}

export function formatTime(date: Date | string): string {
  return formatDate(date, { timeStyle: 'short' });
}

export function formatDateTime(date: Date | string): string {
  return formatDate(date, { dateStyle: 'medium', timeStyle: 'short' });
}

export function formatNumber(n: number, opts?: Intl.NumberFormatOptions): string {
  return new Intl.NumberFormat('ko-KR', opts).format(n);
}

export function formatRelativeTime(date: Date | string): string {
  const d = typeof date === 'string' ? new Date(date) : date;
  const diffSec = (d.getTime() - Date.now()) / 1000;
  const rtf = new Intl.RelativeTimeFormat('ko-KR', { numeric: 'auto' });
  const abs = Math.abs(diffSec);
  if (abs < 60) return rtf.format(Math.round(diffSec), 'second');
  if (abs < 3600) return rtf.format(Math.round(diffSec / 60), 'minute');
  if (abs < 86400) return rtf.format(Math.round(diffSec / 3600), 'hour');
  return rtf.format(Math.round(diffSec / 86400), 'day');
}
```

- next-intl `useFormatter` 를 쓸 수도 있으나 **얇은 Intl.* wrapper 가 더 테스트 쉽고 종속성 작음**. 복잡해지면 next-intl `useFormatter` 로 전환.

### 5.2. 금지

- 컴포넌트에서 **직접 `toLocaleString('ko-KR')` 호출** — 유틸 경유. 로케일 바꾸려 할 때 전수 변경 지옥.
- 하드코딩 포맷 (`${year}-${month}-${day}`) — Intl 가 자동. 수제 포맷은 edge case (타임존·윤년) 실패.

---

## 6. #5 에러 메시지 점진 전환

[#5 error-ux](../../../docs/axis-decisions/error-ux.md) 의 `src/lib/errors/messages.ts` 는 지금 하드코딩. **다국어 적용 시 함수 시그니처 유지, 내부만 교체**:

```ts
// 지금 (하드코딩)
export function codeToMessage(code: string, fallback = MESSAGES.UNKNOWN): string {
  return MESSAGES[toApiErrorCode(code)] ?? fallback;
}

// next-intl 적용 후 (내부만 교체)
import { useTranslations } from 'next-intl';

export function useCodeToMessage() {
  const t = useTranslations('errors');
  return (code: string, fallback?: string) => {
    const key = toApiErrorCode(code);
    try {
      return t(key);
    } catch {
      return fallback ?? t('UNKNOWN');
    }
  };
}
```

- 서버에서는 `getTranslations('errors')` 로 동일 처리.
- **호출부 변경 없음** — `codeToMessage(err.code)` 는 그대로.
- 전환 시점: 다국어 요구 생길 때 또는 i18n infrastructure 구축 완료 시.

---

## 7. #4 zod 메시지 점진 전환

zod 에러 메시지는 **스키마 선언 시점에 하드코딩** 되기 쉬움. 다국어 원할 때:

### 7.1. 전역 errorMap

```ts
// src/lib/validation/zodErrorMap.ts
import { z } from 'zod';
import { useTranslations } from 'next-intl';

export function createZodErrorMap(t: ReturnType<typeof useTranslations<'validation'>>): z.ZodErrorMap {
  return (issue, ctx) => {
    switch (issue.code) {
      case 'invalid_type': return { message: t('required') };
      case 'too_small': return { message: t('minLength', { min: issue.minimum as number }) };
      // ...
    }
    return { message: ctx.defaultError };
  };
}

// 사용
const t = useTranslations('validation');
z.setErrorMap(createZodErrorMap(t));
```

- 전역 `z.setErrorMap` 는 **client/server 각 context 안에서** 설정 (hook 사용 context 필요).
- 스키마별 커스텀 메시지는 `.min(1, { message: t('fieldSpecific') })` 로.

### 7.2. 지금은 하드코딩 inline — 이관 준비만

```ts
// 현재 (하드코딩 inline)
export const createMeetingSchema = z.object({
  title: z.string().min(1, { message: '회의 제목을 입력해 주세요' }),
});

// next-intl 적용 후 (스키마 생성을 함수로)
export function makeCreateMeetingSchema(t: ReturnType<typeof useTranslations>) {
  return z.object({
    title: z.string().min(1, { message: t('validation.meeting.titleRequired') }),
  });
}
```

- 전환 시점에 **스키마 생성을 함수로** 리팩터. 지금은 하드코딩 OK.

---

## 8. SSR / Hydration

- next-intl v3 는 **서버에서 메시지 pre-load → 클라에 전달** 하는 provider 자동 처리.
- `src/app/providers.tsx` 또는 layout 에서 `NextIntlClientProvider` 설정:

```tsx
import { NextIntlClientProvider } from 'next-intl';
import { getMessages } from 'next-intl/server';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const messages = await getMessages();
  return (
    <html lang="ko">
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

- 로케일 1개라 messages 가 자동 로드. 다국어 추가 시 `next-intl` 설정 (`i18n.ts` config) 추가 필요.

---

## 9. 금지 패턴

- 컴포넌트에 **하드코딩 사용자 문자열** (`<button>저장</button>`) — 사소해 보여도 축적되면 i18n 적용 비용 폭발
- `toLocaleString('ko-KR')` 직접 호출 — `formatDate` 등 유틸 경유
- 네임스페이스를 **UI 컴포넌트 이름** 으로 (`createMeetingPage.form.titleInput`) — 리팩터 시 모든 키 깨짐
- 영어 메시지를 하드코딩하고 "나중에 번역" — 처음부터 한국어 `ko.json` 에서 시작
- messages 파일을 **매 요청 fetch** — next-intl 가 compile-time 번들링 (JSON import)

---

## 10. 체크리스트

- [ ] 모든 사용자 가시 문자열이 `t('...')` 로 처리됐는가? 하드코딩 금지?
- [ ] 네임스페이스가 **도메인/공통 영역** 기반인가? UI 컴포넌트 이름 아님?
- [ ] 날짜/숫자 포맷이 `src/lib/i18n/format.ts` 유틸 경유인가? `toLocaleString` 직접 호출 없음?
- [ ] `<html lang="ko">` 설정됐는가? (현재 `"en"` 교정)
- [ ] `NextIntlClientProvider` 가 root layout 에 설정됐는가?
- [ ] 에러·zod 메시지 전환 시 함수 시그니처 유지 (호출부 변경 없음)?
- [ ] ICU 변수가 `{name}` 식인가? 하드 문자열 연결 (`"안녕 " + name`) 금지?

---

## 11. 참고

- 결정 축: [#7 a11y-i18n-tokens](../../../docs/axis-decisions/a11y-i18n-tokens.md) §B
- 관련 스킬: [`error-ux`](../error-ux/SKILL.md) §4 (messages 테이블), [`ts-style`](../ts-style/SKILL.md)
- next-intl v3 공식 docs (App Router), MDN Intl.* API
