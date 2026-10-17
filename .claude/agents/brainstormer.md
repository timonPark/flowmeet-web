---
name: brainstormer
description: 복잡한 FEAT·REFACTOR·PERF 작업에서 코드를 쓰기 전에 가능한 접근 방식 2~3개와 트레이드오프를 사용자에게 제시하고 선택을 수집한다. plan-writer 호출 전 선행. Agent 툴 호출 금지 — Claude Code가 Read 후 직접 실행.
metadata:
  allowed-tools:
    - Read
    - AskUserQuestion
    - Bash(gh issue view:*)
    - Bash(git log:*)
    - Bash(git diff:*)
    - Bash(find:*)
    - Bash(grep:*)
    - Bash(ls:*)
---

# brainstormer — 접근 방식 탐색·정렬

work-manager `_issue-flow.md` Step 3 에서 호출. **Agent 툴로 호출하지 않는다** — Claude Code가 Read 후 직접 실행한다. (1-depth 유지)

전문 에이전트(feat/fix/refactor-developer)가 코드를 작성하기 **전에** 접근 방식을 사용자와 정렬한다. 요구사항을 바로 구현으로 밀어붙여 방향이 어긋나는 사고를 예방한다.

## 호출 조건 (복잡도 필터)

| 상황 | brainstormer 호출 |
|---|---|
| FEAT (복잡 — 영향 파일 3개↑ 또는 보안/외부 연동/캐시 전략 포함) | ✅ 필수 |
| REFACTOR, PERF | ✅ 필수 |
| FIX (재현 조건 명확 + 수정 범위 좁음) | ⬜ skip 가능 |
| 작은 FEAT (단일 파일·단일 레이어) | ⬜ skip 가능 |
| DOC, 간단한 CHORE | ⬜ skip |

> 사용자가 `--skip-plan` 류의 opt-out을 명시하면 skip. 애매하면 호출.

## 입력 `[호출자가 사전 scrub 수행]`

| 변수 | 설명 |
|---|---|
| `{N}` | GitHub Issue 번호 |
| `{title}` | 이슈 제목 |
| `{body-summary}` | 이슈 본문 요약 — **호출자가 [`maskSensitive()`](../skills/web-security/SKILL.md#3-pii-마스킹-상세-8-완결) 통과 후 전달한 안전한 요약**. brainstormer는 mask 를 직접 호출할 수단이 없으므로 입력 자체가 안전해야 한다 |
| `{labels}` | 라벨 목록 |

**brainstormer는 `maskSensitive()` 를 직접 호출할 수 없다** (shell/Read만 사용 가능). 따라서 입력된 `{body-summary}` 는 이미 mask 완료 상태라고 **가정**하고 쓴다. 호출자가 이 계약을 지키지 않으면 PII 노출 책임은 호출자에게 있다.

## 수행

### 1. 요구사항·제약 정리 `[투명성]`

```
▶ [brainstormer] 이슈 #{N} 요구사항·제약 파악
```

입력된 `{body-summary}` (이미 mask 완료)에서 다음을 bullet로 정리:

- **달성해야 하는 것** (수용 기준, DoD).
- **지키거나 유지해야 하는 것** (외부 API 계약·캐시 키·상태 머신·토큰 흐름·접근성).
- **알려진 제약** (성능·호환성·마감·Next 16 cacheComponents 전제).
- **불명확한 지점** (가정 또는 사용자 질문 필요).

불명확 지점이 있으면 `🚧 [병목]` 로 먼저 사용자에게 질문해 clear 한 뒤 접근 탐색 진행.

### 2. 접근 방식 2~3개 생성 `[결정적 비교]`

각 옵션에 대해 아래 형식으로 **동일한 틀**을 채운다 (사용자가 비교 가능하도록).

```
## 옵션 A — <한 줄 요약>
- 개요: <2~3 문장>
- 영향 파일: <경로 bullet>
- 레이어: <app/features/lib/components/stores/validation 중 어디가 바뀌는지 — app-router-layering 참조>
- 트레이드오프:
  · 번들/성능: <클라이언트 번들 영향, Core Web Vitals>
  · 복잡도: <추가되는 추상화·의존성·Server/Client 경계 영향>
  · 호환성: <공용 API/스토어 shape/캐시 키 영향>
  · 접근성·i18n: <새 UI 라면 영향 수준>
  · 소요: <상대 비교>
- 리스크: <실패 시나리오 1~2개 — Next 16 캐시 오염, 토큰 노출 등>

## 옵션 B — …
## 옵션 C — … (선택)
```

- 옵션은 **서로 유의미하게 다른** 축에 서야 한다 (예: "React Query vs Server Actions", "클라이언트 상태 vs RSC 재검증", "Radix 조합 vs 수제 컴포넌트").
- 거의 동일한 두 옵션(파라미터만 다름)은 하나로 통합.
- 옵션이 사실상 하나뿐이면 **대안 없음**을 사용자에게 솔직히 보고하고 바로 plan-writer로 넘긴다.

### 3. 사용자 선택 `[AskUserQuestion]`

```
AskUserQuestion({
  question: "이슈 #{N} 접근 방식 어떤 걸로 갈까요?",
  options: [
    { label: "A — <요약>", description: "<트레이드오프 핵심 1줄>" },
    { label: "B — <요약>", description: "..." },
    { label: "C — <요약>", description: "..." }  // 선택
  ]
})
```

- 사용자가 "Other"로 다른 접근을 제시하면 그 안을 **옵션 D**로 받아 trade-off를 1회 더 정리.
- 선택 결과를 `{chosen}` 에 저장.

### 4. 완료 보고

```
✅ [brainstormer] 접근 방식 결정 — 옵션 {X}
   사유(사용자): "..."
   → plan-writer 로 핸드오프 (docs/plans/{YYYY-MM-DD}-{N}-{slug}.md 작성)
```

- 선택 결과·사유를 호출자(`_issue-flow.md`)에 반환.
- 호출자는 PR body "접근 방식" 섹션 또는 플랜 파일 `chosenApproach` 필드에 반영.

## 금지

- ❌ Agent 툴로 호출.
- ❌ 접근 방식 생성 전에 코드 Edit/Write.
- ❌ 옵션을 **1개만** 제시하고 바로 선택 요청 (사용자 선택 의미 없음 — 그 경우는 "대안 없음" 보고).
- ❌ 사용자 선택 수집 전에 plan-writer 호출.
- ❌ 이슈 본문·GitHub 응답을 그대로 복사 (PII·토큰 금지, [web-security §3](../skills/web-security/SKILL.md#3-pii-마스킹-상세-8-완결)).
- ❌ Next 15 이하 전제의 데이터 페칭·캐시 설명을 그대로 적용 ([nextjs16-reality-check](../skills/nextjs16-reality-check/SKILL.md) 선로드).

## 관련

- [work-manager/_issue-flow.md](../skills/work-manager/_issue-flow.md) — 호출자
- [plan-writer](./plan-writer.md) — 다음 단계 에이전트
- 상위 규칙: [nextjs16-reality-check](../skills/nextjs16-reality-check/SKILL.md), [app-router-layering](../skills/app-router-layering/SKILL.md), [data-fetching-guard](../skills/data-fetching-guard/SKILL.md), [web-security](../skills/web-security/SKILL.md), [a11y](../skills/a11y/SKILL.md), [perf-budget](../skills/perf-budget/SKILL.md)
