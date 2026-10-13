# flowmeet-frontend

조직 지시 및 업무 흐름 관리 플랫폼 프론트엔드.

- Next.js 16 (App Router) + TypeScript
- Tailwind CSS 4
- ESLint 9

백엔드 리포: `../backend` (별도 리포로 분리됨)

## 사전 요구사항

- Node.js 20.9+ (권장: 22 LTS)
- npm 10+

## 최초 세팅

```bash
# 의존성 설치
npm install

# 환경 변수 파일 생성
cp .env.local.example .env.local

# 개발 서버
npm run dev
```

브라우저에서 http://localhost:3000 접속.

## 환경 변수

| 변수명 | 설명 |
| --- | --- |
| `NEXT_PUBLIC_API_BASE_URL` | 백엔드 API 베이스 URL (기본: `http://localhost:8080`) |

## 디렉터리 구조

```text
frontend/
├─ src/app/           # App Router 페이지
├─ public/            # 정적 자산
├─ next.config.ts
├─ tailwind.config.* / eslint.config.mjs
└─ package.json
```

TRD는 상위 레포 `outputs/TRD_조직지시_업무흐름관리플랫폼.md` 참고.
