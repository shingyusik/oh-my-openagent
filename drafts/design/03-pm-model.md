# 03. PM 데이터 모델 + Maestro ↔ Foreman 인터페이스

> Maestro가 관리하는 6-tier 엔티티 모델, 상태 전이, 우선순위, 그리고 Foreman에 dispatch할 때의 Spec/Report 표준 스키마.

---

## 3.1 6-tier PM 계층

```
Vision (V-001)                  ← 비전: 프로젝트의 최종 모습 (변경 드묾)
   │
   ▼
Roadmap (R-*)                   ← 분기/시즌 전략 테마
   │
   ▼
Milestone (M-*)                 ← 검증 가능한 종착점, DoD 필수
   │
   │ (Planning 단계에서 backlog item 선정 → milestone.selected_backlog_items)
   ▼
[Backlog (단일 SSOT)]            ← 모든 todo 저장
   │
   ▼
BacklogItem (B-*)               ← 타입: feature/bug/tech_debt/research/spike
   │
   │ (실행 가능 단위로 쪼개면)
   ▼
Task (T-*)                      ← Foreman에 dispatch되는 최소 단위
```

---

## 3.2 엔티티 정의

| # | 엔티티 | ID prefix | 단일성 | 의미 | 라이프사이클 |
|---|---|---|---|---|---|
| 1 | **Vision** | `V-` | 보통 1개 | 프로젝트의 최종 모습. 가장 추상적, 가장 안정적. | `draft` → `active` → `superseded` |
| 2 | **Roadmap** | `R-` | 1~N개 | 비전 달성을 위한 시간대별 전략 테마 (분기/시즌). | `draft` → `active` → `archived` |
| 3 | **Milestone** | `M-` | 다수 | 로드맵의 주요 지점. 측정 가능한 성과. **DoD 필수**. | `proposed` → `defined` (DoD 완비) → `planned` → `in_progress` → `in_review` → `done` / `cancelled` |
| 4 | **Backlog** | (컨테이너) | **1개 강제** | 프로젝트 모든 할 일 중앙 저장소. SSOT. | (상태 없음, 컨테이너) |
| 5 | **BacklogItem** | `B-` | 다수 | 타입: `feature` / `bug` / `tech_debt` / `research` / `spike`. | `idea` → `triaged` (priority 부여) → `selected` (milestone에 pull) → `in_progress` → `done` → `closed` / `dropped` |
| 6 | **Task** | `T-` | 다수 | BacklogItem의 실행 단위 분해. 단순한 backlog item은 task 1개. | `pending` → `ready` → `dispatched` → `in_progress` → `done` / `blocked` / `cancelled` |

**Issue는 별도 엔티티가 아니다.** `BacklogItem(type=bug)`의 별칭. `/issue add "..."`는 syntactic sugar.

---

## 3.3 엔티티 관계 (ER)

```
Vision (V-001)
   │ guided_by
   ▼
Roadmap (R-001) ── contains ──► Roadmap (R-002)
   │ contains
   ▼
Milestone (M-003) ── depends_on ──► Milestone (M-001)
   │ requires (DoD)
   │ pulls (during Planning)
   ▼
[Backlog (SSOT)]
   │ contains
   ▼
BacklogItem (B-042, type: feature)
   │ broken_into (1+)
   ▼
Task (T-101) ── blocked_by ──► Task (T-099)
                 blocks ──► Task (T-105)


# 실행 중 발견된 버그:
Worker(during T-101) ── auto_creates ──► BacklogItem (B-099, type=bug)
                                          │ Sentinel rule로 priority 자동
                                          │ if severity=high → P0/P1
                                          ▼
                                          backlog _index에 자동 정렬 삽입

# 변경의 추적성 (P16):
Roadmap 변경 → 영향 milestone 자동 재평가 → 영향 backlog item priority 재정렬
   └─ 모든 변경 CONTEXT.md에 append-only 로그
```

---

## 3.4 파일 구조

```
.harness/board/
├── vision.md                        ← V-* (보통 1개, append-only)
├── roadmap.md                       ← R-* 인덱스 + 본문 (단일 파일)
├── milestones/
│   ├── _index.md                    ← M-* 리스트 (priority/status/DoD 충족률)
│   └── M-*.md                       ← 각 milestone (DoD 본문 포함)
├── backlog/
│   ├── _index.md                    ← 모든 B-* priority 정렬 (가시성)
│   ├── _types.md                    ← type별 통계 (feature/bug/tech_debt/research/spike)
│   └── B-*.md                       ← 각 backlog item
└── tasks/
    ├── _index.md                    ← T-* 활성 리스트
    └── T-*.md                       ← 각 task
```

**`issues/` 디렉토리는 없다.** 모든 bug는 `backlog/B-*.md`에 `type: bug`로 저장.

---

## 3.5 엔티티 마크다운 스키마

### Vision (`vision.md`)
```markdown
---
id: V-001
type: vision
status: active
created: 2026-05-20
last_revised: 2026-05-22
revision_reason: "initial draft"
---

# Vision: <한 문장>

## Why
<왜 이 비전인가>

## End State
<이뤘을 때의 모습 — 구체적으로 묘사>

## Non-Goals
<명시적으로 안 할 것 — 스코프 가드>
```

### Milestone with DoD (`M-003.md`)
```markdown
---
id: M-003
type: milestone
status: in_progress
roadmap: R-001
priority: P1
target_date: 2026-07-01
created: 2026-05-20
updated: 2026-05-25
selected_backlog_items: [B-042, B-043, B-044, B-099]
depends_on: [M-001]
dod:
  - id: DoD-1
    criterion: "OAuth 로그인 e2e 테스트 5종 PASS"
    verify_cmd: "pnpm test:e2e apps/web/tests/auth/"
    measurable: true
    state: passed         # pending | in_progress | passed | failed
  - id: DoD-2
    criterion: "Sentinel 4-level verifier 통과 (B-042, B-043, B-044)"
    verify_cmd: "sentinel verify --items B-042,B-043,B-044"
    measurable: true
    state: pending
  - id: DoD-3
    criterion: "Security OWASP 리뷰 PASS"
    verify_cmd: "security audit --milestone M-003"
    measurable: true
    state: pending
  - id: DoD-4
    criterion: "OAuth 사용자 가이드 (docs/auth/oauth.md)"
    verify_cmd: "test -f docs/auth/oauth.md"
    measurable: true
    state: in_progress
---

# Milestone: OAuth 로그인 출시 (M-003)

## Goal
사용자가 Google / GitHub로 가입·로그인 가능하게.

## Scope
<범위 명시>

## Out of Scope
<명시 제외>
```

### BacklogItem (`B-042.md`)
```markdown
---
id: B-042
type: bug               # feature | bug | tech_debt | research | spike
status: selected        # idea | triaged | selected | in_progress | done | closed | dropped
priority: P1            # P0 | P1 | P2 | P3
value: high             # high | medium | low
urgency: medium
milestone: M-003        # null이면 backlog 자체에 머무름
tasks: [T-101, T-102]
discovered_during: T-088
created: 2026-05-22
updated: 2026-05-25
related_lessons: [L-2026-04-10-002]
tags: [auth, oauth, supabase]
---

# Add OAuth login (Google + GitHub)

## Context
<왜 이 일이 필요한가>

## Acceptance Criteria
- AC-1: ...
- AC-2: ...

## Refinement Notes
<우선순위·범위·트레이드오프 메모 — Refinement 단계에서 Maestro 작성>
```

### Task (`T-101.md`)
```markdown
---
id: T-101
type: task
status: dispatched
backlog_item: B-042
estimated_effort: medium
worker_hint: backend            # Foreman이 무시 가능
blocked_by: [T-099]
blocks: [T-105]
worktree: backend-T-101-U
spec_file: phases/<N>/specs/T-101.spec.md
report_file: phases/<N>/reports/T-101.report.md
created: 2026-05-25
updated: 2026-05-26
---

# Task: Implement OAuth provider abstraction layer

(본문은 Maestro가 작성, Foreman에 dispatch될 spec의 source)
```

---

## 3.6 우선순위 매트릭스

가치(Value) × 시급성(Urgency):

| | Urgency: high | medium | low |
|---|---|---|---|
| Value: high | **P0** | P1 | P2 |
| medium | P1 | P2 | P3 |
| low | P2 | P3 | P3 |

**자동 트리거**:
- `BacklogItem(type=bug, severity=high)` 생성 → 자동 P0/P1 (Sentinel 신호 기반)
- `BacklogItem`이 milestone에 `selected`된 후 P2/P3 → 경고 ("정말 이번에 해야 하나요?")
- Sentinel BLOCK 같은 룰 ≥5회 → 자동 `B-* (type=bug)` 생성, P1
- 같은 lesson 3+ 사이클 재등장 → 자동 `B-* (type=tech_debt)` + Phase 8 proposal

---

## 3.7 엔티티 쓰기 권한

| 에이전트 | 쓰기 권한 |
|---|---|
| Maestro | `board/*` 전체 |
| Foreman | `tasks/T-*.md` status 필드만 (`dispatched`→`in_progress`→`done` 등) |
| Sentinel | `backlog/B-*.md` 자동 생성 (type=bug, severity=high BLOCK 시) |
| L2 워커 | `backlog/B-*.md` 자동 생성 (workflow 중 발견 — `discovered_during` 자동), `tasks/T-*.md` status 필드 |
| Agent-Architect | `proposals/*` (board에는 직접 안 씀; 채택 시 Maestro 경유) |

워커가 만든 backlog item은 `status: idea`로 시작. Maestro Refinement에서 triage.

---

## 3.8 4-step 사이클의 엔티티 상태 전이

```
[Refinement 단계]
  - 외부 입력 / 워커 발견 → backlog/B-* 생성 (status: idea)
  - Maestro가 priority 부여 → status: triaged
  - 우선순위 정렬, _index.md 갱신

[Planning 단계]
  - 신규 milestone 생성: status: proposed → DoD 완성 → defined → planned
  - Triaged backlog items 중에서 selected_backlog_items에 link
  - 해당 item.status: triaged → selected
  - Task 분해 (필요 시): backlog item → tasks/T-*.md 생성

[Execution 단계]
  - Maestro가 ready 상태 T-*를 골라 Foreman dispatch
  - T-*.status: dispatched → in_progress → done
  - 부모 backlog item의 task가 모두 done → B-*.status: done

[Review 단계]
  - Milestone.dod[*].verify_cmd 실행 → state: pending → in_progress → passed/failed
  - 모든 passed → milestone.status: in_review → done
  - 종속된 B-* 들의 status: done → closed (선택, milestone done 시 일괄)
  - 회고 lesson 작성 (Tier 3 compound)
  - 다음 milestone을 active로 → 다음 Refinement
```

---

## 3.9 Maestro ↔ Foreman 인터페이스

Maestro가 Foreman에 일을 시킬 때 표준 **Task Spec**을 넘긴다. Foreman은 작업 후 표준 **Task Report**를 돌려준다.

### 3.9.1 Task Spec (Maestro → Foreman)

`task()` 호출 시 prompt에 다음 마크다운을 포함:

```markdown
# Task Spec: T-101 (backlog item B-042)

## Identity
- id: T-101
- backlog_item: B-042 (OAuth 로그인)
- priority: P1
- milestone: M-003
- mode: gated      ← auto/gated/plan-only/dry-run

## Goal
[task의 description + acceptance criteria]

## Acceptance Criteria
- AC-1: ...
- AC-2: ...

## Constraints
- 스택: Supabase Auth만 사용 (Custom OAuth 금지)
- 영향 모듈: apps/web/src/auth/, packages/db/migrations/
- 절대 변경 금지: apps/workers/billing/
- TDD: required (예외 없음)

## Dependencies
- Blocked by: T-099 (done 확인됨)
- Blocks: T-105

## Suggested Worker Allocation
- db-architect (스키마)
- backend (auth API)
- frontend (login UI)
- security (위협 모델)
- qa (e2e 1개 이상)

## Lessons to Consult
- L-2026-04-10-002: Supabase RLS 패턴
- L-2026-04-22-007: Cloudflare Workers OAuth 함정

## References
- 관련 backlog item: B-042
- 외부: <Supabase OAuth docs URL>

## Report Back
- Format: 아래 Task Report 스키마
- Time budget: <Maestro 추정치>
- Escalate to Maestro if:
  - Sentinel BLOCK 3회 같은 룰
  - 새 backlog item이 P0 severity
  - 예상 effort 2배 초과
```

### 3.9.2 Task Report (Foreman → Maestro)

Foreman의 task() 반환값:

```markdown
# Task Report: T-101

## Outcome
- status: done | partial | blocked | cancelled
- commits: [abc123, def456, ghi789]
- worktrees: cleaned
- merged_into: dev
- time_elapsed: 47min
- effort_actual_vs_estimate: 1.1x

## AC Verification (4-level verifier 결과)
- AC-1: PASS (test: e2e/login.spec.ts:23)
- AC-2: PASS
- AC-3: PASS
- AC-4: PARTIAL — UI 완료, edge case <설명>
- AC-5: PASS

## New Backlog Items Discovered
- B-099 (auto-created, type=bug): Refresh token expiry on CF cron

## Lessons Captured (this task)
- L-2026-05-21-001 (architecture): Supabase Auth + CF edge runtime 호환 ...

## Sentinel Summary
- AUTO-FIXED: 12 findings
- ASK pending: 1
  - [src/auth/session.ts:45] naming: `sessionToken` vs `accessToken`?
  - recommend: sessionToken
  - escalate: false

## TDD Compliance
- tests_added: 11
- tests_modified: 3
- coverage_delta: +4.2%
- exceptions_used: 0

## Compound Summary
- Tier 1 lessons emitted: 5
- Tier 2 lessons folded into: L-2026-05-21-001

## Suggested Next Actions (for Maestro to surface)
- T-105 unblocked (depends on T-101 done)
- B-099 should be triaged before M-003 close
- Sentinel ASK 1건 — 사용자 컨펌 권장
```

### 3.9.3 Maestro의 보고서 가공 (사용자 surface)

Maestro는 raw report를 받아 사용자에게:

```markdown
**T-101 (OAuth Provider 추상화) 완료**

✅ 5개 AC 중 4개 PASS, 1개 PARTIAL (UI edge case)
✅ 47분 소요 (예상 대비 1.1x)
✅ 테스트 11개 추가, 커버리지 +4.2%

발견된 새 일:
- 🐛 B-099 (P1): CF cron에서 refresh token 만료 미감지
- ❓ Sentinel: `sessionToken` 명명 OK?

다음 권장:
- T-105 (Profile UI) 시작 가능 (블로커 해제)
- M-003 종료 전 B-099 해결 권장

진행할까요? [Yes / 조정 / 보류]
```

- 사용자는 코드/diff/sentinel raw 출력 안 봄
- 사용자가 자세히 보고 싶을 때만 `/task show T-101` 등으로 깊이 들어감

### 3.9.4 컨텍스트 격리 보장

- Maestro 세션은 board/* + report 요약만 보유 (diff/코드/sentinel raw 출력 X)
- Foreman은 task spec만 받음. 다른 task의 컨텍스트 X.
- task() 분리로 각 호출이 fresh
- 사용자 화면에는 항상 Maestro의 1차 가공된 메시지만 나옴
