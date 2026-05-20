# 01. 설계 원칙 (P1 ~ P16)

> 본 하네스의 **non-negotiable** 원칙. 모든 워커·룰·워크플로가 이 원칙들에 정렬되어야 한다.

| # | 원칙 | 이유 |
|---|---|---|
| P1 | **Plan-first.** 모든 작업은 계획 산출 → 검토 → 승인 이후에만 코드를 친다. | 컨텍스트 낭비 + 잘못된 방향 조기 차단 |
| P2 | **Role-based agents.** 실제 SaaS 개발팀 직책 그대로 매핑. | 책임 경계가 명확 → 프롬프트 응집도 ↑ |
| P3 | **Context isolation by hierarchy.** 상위 에이전트는 하위의 **요약만** 본다. | 컨텍스트 폭증 방지 |
| P4 | **Worktree parallelism by default.** DAG에서 독립 노드는 무조건 별도 worktree에서 병렬 실행. | 실 개발팀과 동일한 동시성 모델 |
| P5 | **Always-on quality supervision.** 매 커밋마다 Sentinel 자동 실행. | 과설계/데드코드/컨벤션 위반 누적 방지 |
| P6 | **Ralph Loop until satisfied.** 목표 만족 검증 통과까지 정지하지 않음. | 부분 완료/거짓 완료 차단 |
| P7 | **Human-in-the-loop only at decision forks.** 실행 단계는 자율. | 사용자 인지 부담 최소화 |
| P8 | **No silent expansion.** 에이전트는 위임받은 스코프 밖으로 절대 확장하지 않는다. | 과설계 방지 (Sentinel과 짝) |
| P9 | **TDD-first.** 코드 워커는 RED(실패 테스트) → GREEN(통과 최소 코드) → REFACTOR 무조건. 테스트 없는 코드 머지 X. | 회귀 보호 + 명세 강제 + AI hallucination 차단 |
| P10 | **Compound every cycle.** 워커 task / Phase / Ralph 사이클이 끝날 때마다 lessons 캡처. 같은 실수 반복 금지. | 컴파운딩 학습 — 다음 사이클의 입력으로 사용 |
| P11 | **Self-evolving harness.** 하네스는 자기 사용 데이터로 자기 자신을 진화시킨다. Agent-Architect가 skill/hook/agent 개선 제안. | 사용할수록 정합도 ↑ |
| P12 | **Maestro is the only user surface.** 사용자는 Maestro와만 대화한다. Vision/Roadmap/Milestone/Backlog/Task 6계층 CRUD + 의존성·우선순위. 모든 sub-agent 입출력은 Maestro가 변환. | 사용자 인지부담 일원화 + 컨텍스트 격리 + 영구 프로젝트 메모리 |
| P13 | **Backlog is the single source of truth.** 모든 할 일(기능/버그/기술부채/리서치/spike)은 backlog에 등록되어야 한다. 다른 곳 todo 저장 금지. 코드 주석 TODO 단독 불가. Sentinel `backlog_singularity` 강제. | 누락·중복 todo 제거, 우선순위의 단일 진실 |
| P14 | **4-step cycle enforced.** Refinement → Planning → Execution → Review를 milestone마다 직선으로 통과. 단계 스킵 금지. Sentinel `workflow_phase_skip` 룰. | 무계획 실행 / 무검토 종결 차단 |
| P15 | **Definition of Done mandatory.** 모든 milestone은 측정 가능한 DoD 없이 생성 불가. DoD 모두 PASS 없이 done 불가. Sentinel `dod_required` 룰. | "만들었다"를 "검증되었다"로 강제 |
| P16 | **Flexibility with traceability.** Roadmap/Milestone은 변경 가능. 변경 시 영향받는 backlog item priority/milestone link가 자동 재정렬. 변경 이력은 CONTEXT.md append-only. | 유연성과 가시성 동시 확보 |

---

## 원칙들 간의 관계

```
P1 (Plan-first)  +  P14 (4-step) +  P15 (DoD)
   └─ Maestro의 Planning 단계에서 한꺼번에 강제

P3 (Context isolation) + P12 (Single surface)
   └─ Maestro만 user-facing, 워커는 task() 격리

P5 (Sentinel) + P8 (No silent expansion) + P9 (TDD) + P10 (Compound) + P13 (Backlog SSOT)
   └─ Sentinel의 룰 카테고리 = 위 원칙들의 자동 강제

P11 (Self-evolving) + P10 (Compound)
   └─ 데이터 → lessons → proposals → 하네스 진화

P16 (Flexibility)
   └─ P15(DoD)의 엄격함을 운영적으로 보완 (변경 가능 + 추적 가능)
```

## 원칙 위반 시 동작

각 원칙은 **자동 강제 메커니즘**을 가진다:

| 원칙 | 강제 메커니즘 |
|---|---|
| P1 | Maestro가 plan 없이 dispatch 거부 |
| P3 | task() 별도 세션, Foreman은 워커 출력 요약만 본다 |
| P5 | post-commit hook이 Sentinel 자동 트리거 |
| P1 | Sentinel `plan_placeholder` 룰 (TBD/추후/모호 표현 BLOCK) |
| P5 | Sentinel `analysis_paralysis` 룰 (5회 read-only 연속 시 자백 또는 BLOCKED) |
| P5 | Sentinel `slopcheck` 룰 (패키지 등급 [SLOP] BLOCK, [ASSUMED] HITL) |
| P5 | Sentinel `self_check_required` 룰 (워커 commit 시 Self-Check 블록 의무) |
| P5 | Sentinel `4_level_verifier` 방법론 (EXISTS→SUBSTANTIVE→WIRED→DATA FLOW) |
| P8 | Sentinel `scope_creep` 룰 (`files_changed not in task.plan.touched_files`) |
| P9 | Sentinel `tdd_violation` 룰 (P0) |
| P10 | Sentinel `compound_required` 룰 (P0) |
| P12 | Foreman/Sentinel/L2 워커는 user-facing 출력 채널 없음 (시스템 차원) |
| P13 | Sentinel `backlog_singularity` + `discovered_not_logged` 룰 (P0) |
| P14 | Sentinel `workflow_phase_skip` 룰 (STATE.md phase 마커 검증) |
| P15 | Sentinel `dod_required` 룰 (P0) |
| P16 | Sentinel `flexibility_traceability` 룰 (CONTEXT.md 로그 의무) |
