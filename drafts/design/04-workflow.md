# 04. 워크플로 — Phase 0 + 4-step 사이클 + Compound + Evolution + Ralph

> 본 하네스의 시간축. 진입(Phase 0) → milestone 마다 4-step 사이클 반복 → 사이클 끝마다 Compound → 누적되면 Phase 8 Evolution.

---

## 4.1 실행 모드

사용자가 `/start` 시점에 모드 선택. 기본값 `auto`. 도중에 `/mode <name>`으로 변경 가능.

| 모드 | 동작 | HITL 게이트 | 코드 작성 | 사용 사례 |
|---|---|---|---|---|
| **`auto`** *(default)* | Maestro Core 인터뷰만 받고 이후 완전 자율. Ralph 루프 종료까지 무인. | Phase 0 + 결정사항 발생 시점만 | 한다 | 본 작업 모드 |
| **`gated`** | 각 phase 종료마다 사용자 컨펌 게이트. | Refinement/Planning/Execution/Review 종료마다 | 한다 | 중요 프로젝트 |
| **`plan-only`** | Refinement + Planning까지만. DoD/backlog 산출 후 종료. | Phase 0 + Planning 끝 | **안 한다** | 견적·범위 산정 |
| **`dry-run`** | 전 phase 실행하지만 코드/머지/배포 안 함. Sentinel 룰만 평가. | auto와 동일 | 시뮬레이션만 | 룰셋·프롬프트 디버깅 |

### 모드 무관 결정 게이트

다음은 **모드 무관**하게 사용자에게 묻는다:

- 비가역 작업 직전 (프로덕션 DB 마이그레이션, 외부 API 결제, public push, DNS 변경)
- 보안 워커가 **HIGH** 등급 위협 발견
- Sentinel `BLOCK` 누적 같은 룰 3회 → 재계획 승인
- 외부 의존성 추가 (slopcheck `[ASSUMED]` 이상)
- Vision 변경 (영향 범위가 큼)
- Milestone DoD 변경 시
- 비용 임계 초과 (MVP 이후 적용)

---

## 4.2 Phase 0 — Vision & PM 부트스트랩 (일회성)

**시점**: `/start` 직후. 프로젝트 시작 시 한 번.

**Maestro Core + private PM sub-agent 행동**:
1. 사용자 인터뷰 (brainstorming 9-step):
   - 타겟 유저 / 핵심 가치 / End State
   - 기술 제약, 도메인 제약
   - Non-Goals (명시 제외)
2. 자료조사 위임: Librarian / Explore / Oracle 병렬
3. Context Librarian이 관련 lessons/prior decisions brief 생성
4. Milestone Planner가 초기 PM 구조 proposal 생성
5. **Board Clerk이 PM 엔티티 초기 생성** (6-tier 모두 부트스트랩, Maestro Core 승인 후):
   1. `board/vision.md` — **V-001 작성 → 사용자 명시 승인 필수 (가장 중요한 게이트)**
   2. `board/roadmap.md` — R-001/R-002… (Vision을 시간 분할)
   3. `board/milestones/M-001.md` — 첫 milestone **DoD 포함** (DoD 없으면 생성 거부)
   4. `board/backlog/_index.md` + 초기 BacklogItem 다수 (feature/bug/tech_debt 후보 모두)
   5. `board/tasks/` — 비어있음 (Planning에서 채워짐)
6. 사용자 승인 게이트: Vision + Roadmap + 첫 Milestone DoD를 사용자에게 보여주고 컨펌
7. 첫 milestone의 **Refinement** 자동 진입

---

## 4.3 4-step 사이클 (milestone마다 반복)

각 milestone은 **Refinement → Planning → Execution → Review** 4단계를 직선으로 통과. 단계 스킵은 Sentinel `workflow_phase_skip` 룰로 차단.

```
                    ┌───── 다음 milestone 활성 ─────┐
                    │                              │
                    ▼                              │
              [Refinement]                         │
                    │                              │
                    ▼                              │
              [Planning]                           │
                    │                              │
                    ▼                              │
              [Execution]  ◄── Sentinel always-on  │
                    │                              │
                    ▼                              │
              [Review]                             │
                    │                              │
                    └──────────────────────────────┘
```

### 4.3.1 Refinement (백로그 정리)

**시점**: milestone 시작 직전. 또는 backlog에 변화가 누적되었을 때 명시적 트리거.

**Maestro Core + private PM sub-agent 행동**:
1. Maestro Core가 Context Librarian에 active milestone/backlog brief 요청
2. Milestone Planner가 status=idea 항목들 triage proposal 작성:
   - type 결정 (feature/bug/tech_debt/research/spike)
   - 가치 × 시급성 매트릭스로 priority 부여
   - 중복 감지 (similar title fingerprint) → merge 제안
3. Milestone Planner가 기존 triaged 항목의 priority 재평가
4. Maestro Core가 proposal 승인 또는 사용자에게 결정 요청
5. Board Clerk이 `_index.md`를 priority 순으로 재정렬하고 cross-link 정합성 검사
6. Maestro Core가 backlog 상태 사용자에게 surface (대시보드 형식: top N + 분포 통계)

**산출**: 정돈된 backlog. 사용자 컨펌(mode=gated인 경우만).

**Sentinel 게이트**:
- backlog item이 다른 곳(임시 메모, 코드 주석 TODO 등)에 산재 → `backlog_singularity` BLOCK
- priority 미부여 status=idea가 > 0 → 다음 단계 진입 거부

### 4.3.2 Planning (계획 수립)

**시점**: Refinement 통과 후. 또는 새 milestone 시작 시.

**Maestro Core + private PM sub-agent 행동**:
1. Milestone Planner가 **Milestone 확인/생성 proposal** 작성:
   - 기존 milestone에 작업 추가: existing M-*.md 보강
   - 새 milestone 생성: title + roadmap link + **DoD 작성 게이트**
   - DoD는 측정 가능해야 함 (verify_cmd 필수). 측정 불가하면 거부
2. Planning Worker + Design Worker **병렬 dispatch** (필요 시):
   - Planning Worker: PRD/요구사항 보강 (현재 milestone 범위)
   - Design Worker: design-spec/구조 보강
3. Milestone Planner가 결과 수신 → **backlog item 선정 proposal (selected_backlog_items)**:
   - milestone의 DoD 충족에 필요한 B-* 항목들을 backlog에서 골라 milestone.selected_backlog_items에 추가
   - 각 B-*.status: triaged → selected
4. Spec Writer가 **Task 분해 초안** 작성 (필요 시):
   - 복잡한 B-*는 Foreman 호출 전 단계에서 task 1+ 개로 분해
   - 단순한 B-*는 task가 자기 자신 1개
5. Maestro Core가 승인 → Board Clerk이 M-*/B-*/T-* 변경 반영
6. 사용자 컨펌 게이트 (mode=gated): "이번 milestone에 이 항목들 진행할까요?"

**산출**: milestone.selected_backlog_items + tasks/T-*.md (ready 상태)

**Sentinel 게이트**:
- milestone에 DoD 없음 → `dod_required` BLOCK
- DoD 항목 중 verify_cmd 빈 것 → BLOCK
- Refinement 거치지 않고 Planning 진입 → `workflow_phase_skip` BLOCK

### 4.3.3 Execution (수행)

**시점**: Planning 통과 후.

**Maestro Core + private PM sub-agent 행동**:
1. ready 상태 task 중 P0/P1 우선 선택
2. Spec Writer가 Task Spec 초안 작성
3. Maestro Core가 Task Spec 승인 → Foreman dispatch
4. Foreman의 DAG 빌드 + worktree 병렬 + 워커 7-step 실행
5. Sentinel always-on 감사 (TDD, compound, 룰 전체)
6. **실행 중 발견 → 즉시 backlog 등록** (Sentinel `discovered_not_logged` 강제):
   - 워커가 buggy edge case 발견 → 자동 `B-* (type=bug)` 생성
   - 새 todo 떠올림 → `B-* (type=feature/research)` 생성
   - 절대 코드 주석 TODO만 남기고 끝내지 X (Sentinel BLOCK)
7. Foreman의 Task Report 수신 → Report Editor가 사용자용 summary 작성 → Maestro Core가 surface
8. milestone의 모든 selected task가 done이 될 때까지 반복

**Sentinel 게이트**:
- TDD/compound/architecture violation 전체 (`05-sentinel.md` 참조)
- 워커가 task 종료 보고에 새 발견 lesson이 있는데 backlog 등록 없음 → BLOCK
- Planning 거치지 않고 Execution → `workflow_phase_skip` BLOCK

### 4.3.4 Review (검토 및 회고)

**시점**: milestone의 모든 selected task가 done 또는 cancelled.

**Maestro Core + private PM sub-agent 행동**:
1. **DoD 검증 (필수)** — Sentinel 4-level verifier 사용 (`05-sentinel.md` §5.9):
   - milestone.dod[*].verify_cmd 1개씩 실행
   - 각 verify_cmd는 Level 1 (EXISTS) → Level 2 (SUBSTANTIVE) → Level 3 (WIRED) → Level 4 (REAL DATA FLOW) 순서로 검증
   - 한 단계 실패 시 즉시 BLOCK (다음 단계 skip)
   - 결과 state = passed / failed / in_progress
   - 모든 passed가 아니면 milestone done 거부
   - 실패한 DoD → 새 task로 backlog에 추가 (P0/P1)
2. **회고 (Tier 3 compound, §4.5 참조)**:
   - Tech-Writer + Agent-Architect 합작
   - 이번 milestone의 전 Tier 1/2 lessons를 가로질러 system-level pattern 추출
   - ETHOS 승급 후보 검토
   - `harness/lessons/_milestone-retro/<M-id>.md` 생성
3. Milestone Planner가 **다음 milestone을 위한 backlog 우선순위 재조정 proposal** 작성:
   - 변경된 컨텍스트(이번에 배운 것) 반영
   - 새로 발견된 B-* 정렬
   - 다음 milestone의 selected_backlog_items 후보 prep
4. Milestone.status: in_review → done
5. **사용자 보고**:
   - milestone summary (DoD 충족표, 완료된 backlog item, 소요 시간, 비용)
   - 다음 milestone 제안 (사용자 컨펌으로 활성화)
6. 다음 milestone 활성화 → **다음 Refinement 트리거** (사이클 재진입)

**Sentinel 게이트**:
- DoD 1개라도 fail → milestone done 거부 (`dod_required`)
- 회고 lesson 없음 → `compound_required` BLOCK
- Execution 거치지 않고 Review → `workflow_phase_skip` BLOCK

### 4.3.5 단계 진입 가드

`.harness/STATE.md`에 현재 milestone의 phase 마커 기록:
```
milestone: M-003
phase: refinement | planning | execution | review
phase_entered_at: 2026-05-25T09:00:00Z
phase_completed_at: null
```

각 단계는 이전 단계의 `phase_completed_at`이 있어야 진입 가능. Sentinel이 매 turn 검증. STATE.md는 `O_EXCL` lock으로 보호.

---

## 4.4 Worker Loop (Execution 내부)

각 워커는 자기 worktree에서 다음을 수행:

```
loop:
  1. plan            — 자기 task 세분화 → checklist
  2. red             — 실패 테스트 작성 (TDD)
  3. green           — 통과 최소 코드
  4. refactor        — 정리
  5. spec-review     — plan/AC 충족 (별도 task() 세션)
  6. quality-review  — 스타일/복잡도 (별도 task() 세션)
  7. commit          — git-master로 atomic commit
  8. sentinel-check  — Foreman을 통해 Sentinel 호출 (PASS/FIX/BLOCK)
     - PASS  → 다음 checklist 항목
     - FIX   → fix 후 다시 3번부터
     - BLOCK → Foreman에게 escalate
  9. compound        — task-level lesson emit
  checklist 다 끝나면 → emit <role-done> → Foreman에 결과 요약 전송
```

(상세 step 정의는 `02-agents.md` §2.3 참조)

---

## 4.5 Compound (3-tier 학습 코디피케이션)

**3-tier compound**:

### Tier 1 — Task-level (각 워커 step:compound)
- 매 task 종료 시 워커가 1-paragraph **lesson 후보** emit
- 형식: `{ symptom, root_cause, what_worked, prevent_next_time, related_u_ids }`
- 출처: 자기 task의 sentinel-log + fix 시도 + 테스트 실패 이력
- 임시 저장: `.harness/lessons/_pending/L-<u_id>-<seq>.json`

### Tier 2 — Phase-level (각 phase 종료 시)
- Tech-Writer 워커가 **Lead**로서:
  1. 해당 phase의 모든 `_pending/L-*.json` 수집
  2. 카테고리별 그룹핑 (architecture/bugs/perf/ops/process/prompts)
  3. dedup + merge (fingerprint-merge 알고리즘: 같은 root_cause 묶기)
  4. 최종 lesson markdown 작성 → `.harness/lessons/<category>/L-<slug>-<date>.md`
  5. YAML frontmatter: `track / category / problem_type / module / tags / req_ids / cycle_phase`
- Agent-Architect가 **discoverability patch**: 새 lesson이 AGENTS.md / `.harness/ETHOS.md`에서 검색 가능하도록 인덱스 라인 추가

### Tier 3 — Cycle-level (Review 단계의 일부)
- Tech-Writer + Agent-Architect 합작:
  1. 전 phase의 lessons를 가로질러 **cross-cutting pattern** 발견
  2. 같은 root_cause가 ≥3 다른 워커에서 발생 → "system-level lesson"으로 승격
  3. system-level lesson은 ETHOS.md에 추가 후보로 표시
  4. 다음 milestone의 모든 워커 preamble에 자동 inject

**Lesson 재사용 메커니즘**:
- Librarian 워커는 모든 task 시작 시 `.harness/lessons/`를 grep해 관련 lesson을 컨텍스트로 inject
- 매칭 키: 영향 모듈 경로 + 카테고리 + tags
- 같은 lesson이 다른 워커에서 5회 hit → "high-leverage" 태그 부여, ETHOS 승급 후보

**Compound 게이트 (Sentinel)**:
- 워커 task 완료에 `compound_emitted: null`이면 BLOCK
- Phase 종료에 lesson 산출 없으면 BLOCK
- "no learnings" 사유 명시한 빈 lesson은 valid (반복 시 자기교정 신호)

---

## 4.6 Phase 8 — Harness Evolution Review

Ralph 사이클이 DONE되거나 사용자가 `/evolve`로 수동 트리거하면 실행.

### 4.6.1 트리거
- 자동: 매 milestone Review 직후 1회
- 자동: 누적 Sentinel BLOCK이 같은 룰에서 ≥5회 → 즉시 트리거
- 자동: 같은 lesson이 ≥3 사이클에서 재등장 → 트리거
- 수동: `/evolve` 명령

### 4.6.2 Agent-Architect의 진화 사이클 (워커 6-step)

```
1. diagnose
   ├─ scan .harness/sentinel-log.jsonl (최근 N 사이클)
   ├─ scan .harness/lessons/* (특히 high-leverage 태그)
   ├─ scan .harness/proposals/* (과거 제안의 채택/기각 이력)
   └─ 누적 패턴 도출:
       - 어떤 룰이 가장 자주 BLOCK 했나
       - 어떤 lesson이 가장 자주 재등장하나
       - 어떤 워커가 가장 자주 escalate 했나
       - 어떤 task 유형에서 step:red 실패 빈도가 높았나

2. baseline
   └─ 발견된 실패 패턴 N개 중 가장 빈도 높은 1~3개를 실험 시나리오로 재현
       (이 패턴이 막혀야 한다는 RED 증거 확보)

3. design
   └─ 개입 옵션 카탈로그:
       a. 신규 Sentinel 룰 추가
       b. 기존 룰의 severity 변경 (warn→fix→block)
       c. 신규 hook (PreCommit/PostCommit/PreToolUse 등)
       d. 신규 skill
       e. 기존 워커 prompt 수정 (특정 step에 가드 prose 추가)
       f. 신규 워커 추가
       g. DAG 빌더 휴리스틱 추가 (Foreman 메타룰)
       h. 카테고리 신규 (예: `migration-safety` 카테고리)
       i. ETHOS 항목 추가
       각 옵션의 비용/효과/롤백 시나리오 평가

4. propose
   └─ .harness/proposals/<YYYY-MM-DD>-<slug>.md 작성

5. peer-review
   └─ Oracle에 의뢰: "이 제안이 과설계인가? 더 단순한 대안은?"

6. compound
   └─ 진화 제안 자체의 lesson emit (어떤 제안이 채택/기각되었나 추적)
```

### 4.6.3 Proposal 문서 형식

```markdown
---
proposal_id: P-2026-05-20-001
status: pending | accepted | rejected | superseded
type: new_rule | new_hook | new_skill | new_agent | prompt_revision | category | ethos
target: sentinel | foreman | <worker-name> | global
evidence_lessons: [L-001, L-007, L-012]
evidence_block_count: 8
expected_impact: high | medium | low
expected_effort: hours | days | week
rollback: "<롤백 절차>"
---

# Proposal: <짧은 제목>

## 문제 (Diagnose 요약)
<누적 데이터로 본 문제>

## 시도해 본 베이스라인
<P9 P10이 못 잡은 이유>

## 제안 개입
<구체 변경: 파일 경로, 추가될 룰/프롬프트/훅 본문>

## 비용/효과
- 비용: <구현 + 운영 비용>
- 효과: <어떤 lesson/block을 막을 것으로 예상>
- 위험: <과설계 가능성, false-positive, 다른 워커 충돌>

## 대안
<더 단순한 대안 또는 안 하는 옵션>

## 롤백
<채택 후 안 좋으면 되돌리는 절차>
```

### 4.6.4 Maestro Core의 채택 결정 (HITL)

- Maestro Core가 `proposals/_pending/`에서 새 제안을 사용자에게 1건씩 또는 batch로 제시
- 사용자 결정:
  - **accept** → Agent-Architect가 직접 실 파일 수정 (.opencode/, .harness/) + 변경 commit
  - **reject** → `status: rejected` + 사유 메모
  - **defer** → 다음 사이클 재평가
  - **supersede** → 더 나은 제안 작성

### 4.6.5 메타-회귀 방지

- 진화로 추가된 룰/skill에는 `evolved: true` + `source_proposal_id` 메타데이터
- 채택 후 N 사이클 동안 효과 측정: 의도한 lesson이 실제로 줄어들었나?
- 효과가 0이면 자동 deprecate 후보로 다음 evolution 사이클에서 검토 (룰 과부하 방지)
- ETHOS 항목 추가는 `accepted` 후 최소 3 사이클 효과 검증 거쳐야 영구화

### 4.6.6 컨텍스트 분리

- Phase 8은 Maestro Core 컨텍스트에 들어오지 않음
- Agent-Architect의 별도 task() 세션에서 진행
- 결과는 `proposals/*.md` 파일만 Maestro Core에 통보 (요약 1줄)

---

## 4.7 Ralph Loop 종료 조건

### 4.7.1 종료 신호 매트릭스

| 레벨 | 종료 신호 | 종료 조건 | 감시자 |
|---|---|---|---|
| Worker step | `<step-done>` | checklist 1항목 완료 + lsp 진단 깨끗 | Worker 자기 자신 |
| Worker (전체 task) | `<role-done>` | 모든 checklist + Sentinel PASS + compound emit | Foreman |
| Worktree merge | (자동) | Worker done + merge clean | Foreman |
| Task | (자동) | 모든 unit done | Foreman → Report Editor → Maestro Core |
| Milestone | (자동) | 모든 DoD passed + 회고 작성 완료 | Maestro Core |
| Global | `<promise>DONE</promise>` | 모든 active milestone done | Maestro Core |

각 레벨은 **상위 레벨의 done을 기다리지 않는다**. 자기 조건만 만족하면 자기 신호 emit. 상위가 폴링.

### 4.7.2 milestone 단위 종료

```
DONE = (모든 P0/P1 task = done)
     ∧ (현재 milestone의 모든 DoD verify_cmd PASS)
     ∧ (Sentinel 누적 BLOCK == 0)
     ∧ (Integration tests pass)
     ∧ (Security audit pass)
     ∧ (open backlog item 중 P0/P1 == 0)
     ∧ (Tier 3 compound 회고 작성 완료)
     ∧ (사용자 명시 거부 없음)

if DONE:
  → milestone status = done
  → 다음 milestone이 있으면 그것을 active로 → Refinement 재진입
  → 없으면 emit <promise>DONE</promise>, 사용자 보고

else:
  → 미충족 항목을 Milestone Planner가 분석하고 Maestro Core가 승인:
      - 미완 task → Foreman에 reinject
      - 새 backlog item (type=bug) → 자동 등록 + priority 부여
      - 누락 DoD → 새 task 자동 생성 → board에 추가
  → 적절한 phase로 재진입
```

### 4.7.3 상한

- `max_global_iterations`(기본 50)
- `/stop` 즉시 중단
- idle 감지(3사이클 무진전)
- 같은 룰 BLOCK 누적 5회 → Phase 8 evolution trigger + Maestro Core escalate
