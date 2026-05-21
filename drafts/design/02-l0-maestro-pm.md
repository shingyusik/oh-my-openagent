# 02. L0 — Maestro Core와 PM 계층

> 이 문서는 사용자 대면 계층과 PM 메모리 모델을 소유한다. 워커 내부나 Sentinel 룰의 상세는 다루지 않는다.

---

## 2.1 L0 책임

Maestro Core는 의도적으로 좁다:

| 책임 | Maestro Core 소유? | 비고 |
|---|---:|---|
| 사용자 대화 | 예 | 사용자는 Maestro Core만 본다 |
| 최종 결정 | 예 | 승인, 거부, 보류, 사용자 질의 |
| HITL 게이트 | 예 | 비가역 작업, 큰 범위 변경, 모호한 트레이드오프 |
| PM 파일 쓰기 | 아니오 | 승인 후 Board Clerk에 위임 |
| 플래닝 분석 | 아니오 | Milestone Planner에 위임 |
| Task Spec 초안 | 아니오 | Spec Writer에 위임 |
| 보고서 재작성 | 아니오 | Report Editor에 위임 |
| 컨텍스트 검색 | 아니오 | Context Librarian에 위임 |

핵심 원칙:

> Maestro Core는 대화와 결정을 소유한다. PM 잡일은 소유하지 않는다.

---

## 2.2 Private PM Sub-Agent

private PM sub-agent는 사용자에게 직접 말하지 않는다. schema 형태의 산출물만 Maestro Core에 반환한다.

| 에이전트 | 역할 | 쓰기 권한 |
|---|---|---|
| **Board Clerk** | Vision/Roadmap/Milestone/Backlog/Task CRUD, `_index.md`, 상호 링크, 상태 전이 | `board/*`, 승인된 변경만 |
| **Milestone Planner** | Refinement, 우선순위, 의존성 분석, milestone/DoD 제안 | 없음, 제안만 |
| **Spec Writer** | 선택된 backlog/task를 Foreman Task Spec으로 변환 | 없음, spec 초안만 |
| **Report Editor** | Foreman/Sentinel raw 보고서를 사용자용 요약으로 변환 | 없음, 요약만 |
| **Context Librarian** | 관련된 board 본문, lesson, 과거 결정을 on-demand 검색 | 읽기 전용 |

표준 위임 계약:

| 요청 | 처리자 | 산출 | 승인 |
|---|---|---|---|
| `context_brief` | Context Librarian | 소스 id + 짧은 brief | 없음 |
| `refinement_proposal` | Milestone Planner | triage, 우선순위, 병합 후보 | Maestro Core |
| `board_patch_request` | Board Clerk | 적용된 변경 + 정합성 검증 | 쓰기 전 Maestro Core |
| `task_spec_draft` | Spec Writer | Task Spec markdown/schema | 디스패치 전 Maestro Core |
| `report_summary` | Report Editor | 사용자용 요약 + 질의 | surface 전 Maestro Core |

이 계층에서는 자유 형식 장문 보고서를 금지한다. 모든 응답은 구조화되어야 한다.

---

## 2.3 PM 엔티티 모델

```text
Vision (V-001)
  -> Roadmap (R-*)
       -> Milestone (M-*) with measurable DoD
            -> Backlog (단일 진실 저장소)
                 -> BacklogItem (B-*)
                      -> Task (T-*) → Foreman으로 디스패치
```

| 엔티티 | 의미 | 소유자 |
|---|---|---|
| Vision | 도달하려는 최종 상태와 non-goal | Maestro Core 승인, Board Clerk 작성 |
| Roadmap | Vision으로 향하는 전략 슬라이스 | Board Clerk |
| Milestone | 검증 가능한 전달 체크포인트 | Milestone Planner 제안, Board Clerk 작성 |
| Backlog | 모든 향후 작업의 단일 출처 | Board Clerk |
| BacklogItem | feature, bug, tech_debt, research, spike | Board Clerk, Sentinel/워커는 아이디어 자동 생성 가능 |
| Task | 디스패치 가능한 실행 단위 | Spec Writer 초안, Foreman 실행 |

Backlog item 타입:

| 타입 | 의미 |
|---|---|
| `feature` | 사용자 또는 제품 capability |
| `bug` | 잘못된 동작 또는 DoD 실패 |
| `tech_debt` | 향후 작업을 늦추는 구조적 이슈 |
| `research` | 구현 전에 해결해야 하는 미지의 문제 |
| `spike` | 시간 박스 실험 |

---

## 2.4 상태 전이

```text
Refinement
  외부 입력 / 워커 발견 -> B-* status: idea
  Milestone Planner가 triage 제안
  Maestro Core 승인
  Board Clerk가 status: triaged 갱신

Planning
  Milestone Planner가 마일스톤 + 선택된 backlog item 제안
  Spec Writer가 task 분해 제안
  Maestro Core 승인
  Board Clerk가 M-*/B-*/T-* 링크 작성

Execution
  Maestro Core가 준비된 T-* 선택
  Spec Writer가 Task Spec 초안 작성
  Maestro Core가 디스패치 승인
  Foreman 실행
  Board Clerk가 task 상태 변화 반영

Review
  DoD verify 결과가 마일스톤 상태 갱신
  실패한 DoD는 backlog 생성/갱신
  Review 요약이 다음 Refinement에 반영
```

---

## 2.5 Maestro Core 컨텍스트 예산

Maestro Core는 다음만 상시 보유한다:

- 사용자 목표와 현재 run mode
- `board/*/_index.md`
- 활성 Vision/Milestone/Task 요약
- 최근 Report Editor 요약
- 대기 중인 HITL 질의

board 본문 전체, raw diff, Sentinel 로그, 워커 출력은 컨텍스트에 두지 않는다. 이들은 Context Librarian이나 Report Editor를 통해 on-demand로 요청한다.

---

## 2.6 L0이 소유하는 사용자 명령

명령은 사용자 대면이지만, 구현은 내부적으로 위임될 수 있다.

| 명령군 | 의미 |
|---|---|
| `/start`, `/mode`, `/pause`, `/resume`, `/stop` | 라이프사이클과 run mode |
| `/board`, `/vision`, `/roadmap`, `/milestone` | PM 엔티티 조회/변경 |
| `/backlog`, `/issue`, `/task` | 작업 항목 관리 |
| `/dod` | DoD 작성 및 검증 |
| `/next`, `/refinement`, `/planning`, `/execution`, `/review` | 페이즈 제어 |
| `/report` | 명시적으로 요청된 경우의 요약과 raw 보고서 접근 |
| `/lessons`, `/proposals`, `/evolve` | 학습과 진화 |

다른 에이전트가 작업하더라도 출력은 항상 Maestro Core에서 나온다.
