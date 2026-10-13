# CLAUDE.md

조직 지시 및 업무 흐름 관리 플랫폼 웹 프론트엔드 리포 (`flowmeet-web`). API 서버는 별도 리포 (`../flowmeet-api`).

## 작업 시작 규칙 (필수)

**어떤 작업이든 반드시 전용 브랜치를 만들어 진행한다. `main`에 직접 커밋 금지.**

- 작업 착수 시 가장 먼저 `main`에서 새 브랜치를 분기한다.
- 브랜치 명명 규약: `<type>/<issue-number>-<slug>`
  - `type`: `feature` / `fix` / `refactor` / `chore` / `docs` / `test`
  - 예: `feature/5-meeting-list`, `fix/12-recording-button`, `docs/3-readme`
- 이슈 번호가 없는 작업은 `<type>/<slug>` (예: `chore/update-tailwind`).
- 작업 완료 후 PR을 생성해 `main`으로 머지 — 직접 푸시 금지.

```bash
# 올바른 착수 흐름
git checkout main && git pull
git checkout -b feature/<issue>-<slug>
# ... 작업 ...
git push -u origin feature/<issue>-<slug>
gh pr create --fill
```

## 기술 스택

- Next.js 16 (App Router) + React 19 + TypeScript 5
- Tailwind CSS 4 (`@tailwindcss/turbopack`)
- ESLint 9 (`eslint-config-next`)
- Node.js 20.9+ (권장: 22 LTS, nvm 사용)

## 디렉터리 구조

```text
flowmeet-web/
├─ src/app/              # App Router 페이지 및 레이아웃
│  ├─ layout.tsx
│  ├─ page.tsx
│  └─ globals.css
├─ public/               # 정적 자산
├─ eslint.config.mjs
├─ next.config.ts
├─ tsconfig.json         # @/* → src/* alias
└─ package.json
```

## 자주 쓰는 명령

```bash
# 최초 1회 (nvm 사용 시)
nvm use 22
npm install
cp .env.local.example .env.local

# 개발 서버 (http://localhost:3000)
npm run dev

# 프로덕션 빌드
npm run build
npm start

# 린트
npm run lint
```

## 환경 변수

| 변수 | 설명 |
| --- | --- |
| `NEXT_PUBLIC_API_BASE_URL` | `flowmeet-api` 베이스 URL (기본: `http://localhost:8080`) |

`NEXT_PUBLIC_` 접두사가 없는 값은 서버 전용. 비밀값은 반드시 서버 컴포넌트/Route Handler에서만 사용.

## 설계 원칙 (TRD 발췌)

- **인증**: 이메일·비밀번호 로그인, JWT Access Token + Refresh Token. 최초 로그인 시 임시 비밀번호 변경. (TRD §6)
- **권한 검사는 백엔드가 수행**. 프론트는 UI 가드 용도로만 역할을 체크하고, 보안 경계는 서버에 위임.
- **녹음**: 브라우저 MediaRecorder API로 WebM/Opus 또는 브라우저 원본 형식 생성 → IndexedDB에 임시 저장 → 회의 종료 후 서버 업로드. (TRD §4.1)
- **화자 매칭 UI**: 화자 라벨별 매칭 상태와 전체 STT utterance 목록을 함께 받아 즉시 로컬 반영. (TRD §4.3)
- **회의실 캘린더**: 웹은 일·주간 캘린더, 모바일은 날짜별 예약 목록. (TRD §4.2)
- **대시보드**: `dashboardType=MEETING | TEAM` 분기. 팀 대시보드는 지연 업무(업무명·담당자·기한·지연일수)를 표시. (TRD §8)

## API 통신

- `flowmeet-api` 베이스 URL: `process.env.NEXT_PUBLIC_API_BASE_URL`
- 인증: `Authorization: Bearer <accessToken>`
- Refresh Token은 쿠키 또는 안전한 스토리지에 보관 (XSS 리스크 고려)
- API 클라이언트는 `src/lib/api/`에 레이어로 분리 예정

## 참고 문서

- TRD: `../outputs/TRD_조직지시_업무흐름관리플랫폼.md`
- PRD: `../outputs/PRD_조직지시_업무흐름관리플랫폼.md`
- UI PRD (웹): `../outputs/STITCH_UI_PRD_WEB_웹용.md`
- UI PRD (모바일): `../outputs/STITCH_UI_PRD_MOBILE_모바일용.md`
- 통합 UI PRD: `../outputs/STITCH_UI_PRD_조직업무흐름플랫폼.md`

## 코드 규칙

- App Router 우선 (`pages/` 사용 금지).
- 서버 컴포넌트가 기본, 클라이언트 상태가 필요한 경우에만 `"use client"`.
- Import alias는 `@/*` (= `src/*`).
- 타입은 `type` 선호, 확장이 필요한 경우에만 `interface`.
- 스타일은 Tailwind 유틸리티 우선. 반복 패턴은 컴포넌트로 추출.
- 폼 검증은 백엔드 응답을 신뢰하되 UX를 위한 선검증은 클라이언트에서 수행.
