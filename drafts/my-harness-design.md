# VibeForge Harness 설계

> 베이스: `oh-my-openagent` (OpenCode 플러그인 아키텍처) 위에 얹는 커스텀 레이어.
> 바이브코딩 기법을 구조화하고, 프로젝트 경험으로 계속 진화하는 하네스.
> 변경 이력은 [`CHANGELOG.md`](./CHANGELOG.md) 참조. 본 문서들은 **현재 상태만** 기술한다.

---

## 한 줄 요약

> **사용자는 Maestro Core와만 대화 → Maestro Core는 단일 사용자 창구와 최종 결정권을 유지하되 PM 반복 노동은 private PM sub-agent(Board Clerk, Milestone Planner, Spec Writer, Report Editor, Context Librarian)에 위임 → Vision → Roadmap → Milestone → Backlog → BacklogItem → Task 6계층을 구조화 → Refinement(백로그 정리) / Planning(이번 milestone 선정 + DoD 정의) / Execution(현재 task의 spec을 Foreman에 dispatch → 워커들이 worktree에서 7-step 병렬) / Review(DoD 검증 + 회고 + 우선순위 재정렬) 4-step 사이클을 milestone마다 반복 → Sentinel이 매 커밋마다 품질 + 워크플로 순서 + DoD 충족 감시 → Report Editor가 요약하고 Maestro Core가 사용자에게 surface → 다음 milestone으로 사이클 재진입, Agent-Architect는 누적 데이터로 하네스 자체를 진화.**

---

## 계층 구조 (한눈에)

```
                    ╔═════════════════════════════════════════════╗
   사용자 ◄════════►║  Maestro Core (단일 창구 + 최종 결정권자)         ║   P12
                    ║                                              ║
                    ║  위임: Board Clerk / Milestone Planner /      ║
                    ║        Spec Writer / Report Editor / Librarian║
                    ║  사이클: Refinement → Planning → Execution    ║
                    ║         → Review (per milestone, P14)        ║
                    ╚════════════════╤═════════════════════════════╝
                                     │  Task Spec (명세서)
                                     ▼
                       ┌─────────────────────────────┐
                       │  Foreman  (현장 십장)         │   user-facing 아님
                       │  DAG 빌드, worktree 분배       │
                       │  Ralph 루프 / Wave 머지        │
                       └──┬───────────────────────┬──┘
                          │                       │
          ┌───────────────┼────────────────┐      │
          │               │                │      ▼
       ┌──▼──┐         ┌──▼──┐         ┌──▼──┐  ┌──────────────┐
       │ L2  │   ...   │ L2  │   ...   │ L2  │  │  Sentinel     │
       │Worker│        │Worker│        │Worker│  │  (감독관)      │
       └──┬──┘         └──┬──┘         └──┬──┘  └──────────────┘
          │ 7/5/6-step    │               │           ▲
          ▼               ▼               ▼           │ 매 commit
       commit ────────────────────────────────────────┘
          │
          ▼
      compound ──► .harness/lessons/  ──► Librarian이 다음 task 시작 시 inject
          │
          ▼
      Foreman Report ──► Report Editor ──► Maestro Core가 사용자에게 surface
```

---

## 문서 구조 (섹션별 분리)

상세 설계는 [`design/`](./design/) 디렉토리의 8개 파일로 분리되어 있다.

| # | 파일 | 내용 |
|---|---|---|
| 01 | [**Overview**](./design/01-overview.md) | 전체 컨셉, 고정 코어, 적응형 레이어, 아키텍처 계층 지도 |
| 02 | [**L0 Maestro + PM**](./design/02-l0-maestro-pm.md) | Maestro Core, private PM sub-agent, 6-tier PM 엔티티 |
| 03 | [**L1 Foreman**](./design/03-l1-foreman-execution.md) | Task Spec, DAG, worktree, merge, Task Report |
| 04 | [**L2 Worker Profiles**](./design/04-l2-worker-profiles.md) | 진화형 worker profile과 7/5/6-step 내부 흐름 |
| 05 | [**L1 Sentinel**](./design/05-l1-sentinel-quality.md) | 품질/워크플로 게이트, 2-tier 비용 제어, 4-level verifier |
| 06 | [**Cross-Layer Workflows**](./design/06-cross-layer-workflows.md) | Phase 0, 4-step cycle, Compound, Phase 8, Ralph loop |
| 07 | [**Runtime Inventory**](./design/07-runtime-inventory.md) | 파일 구조, PROJECT_PROFILE, CONVENTIONS, STATE, 사용자 명령 |
| 08 | [**Implementation Roadmap**](./design/08-implementation-roadmap.md) | OpenCode 매핑, 구현 순서, v0.6 기본 결정, 열린 사항 |

---

## Companion Documents

### 설계 참고 자료
- [`review-v0.5.1.md`](./review-v0.5.1.md) — v0.5.1 설계 리뷰와 보강 제안.
- [`surveys/superpowers.md`](./surveys/superpowers.md) — obra/superpowers
- [`surveys/compound-engineering.md`](./surveys/compound-engineering.md) — Every Inc 컴파운드 엔지니어링
- [`surveys/gstack.md`](./surveys/gstack.md) — Garry Tan gstack
- [`surveys/get-shit-done.md`](./surveys/get-shit-done.md) — GSD 하네스

### 변경 이력
- [`CHANGELOG.md`](./CHANGELOG.md) — v0.1 ~ v0.5.5 모든 버전 변경 요약 + 결정 사항 누적.

---

## 빠른 시작 (사용자 관점)

```bash
# 1. 새 프로젝트 부트스트랩
/start "<자연어 목표>"
  → Maestro Core 인터뷰
  → Vision + Roadmap + 첫 Milestone (DoD 포함) + 초기 Backlog 자동 생성
  → 사용자 컨펌 후 자동 진행

# 2. 진행 상황 확인
/board                          # 전체 보드
/status                         # 현재 phase + 활성 worker

# 3. 사이클은 자동 진행 (mode=auto 기본)
#    Refinement → Planning → Execution → Review → 다음 milestone

# 4. 사용자 개입은 게이트에서만
#    - 비가역 작업 직전
#    - Sentinel BLOCK 3회 같은 룰
#    - 외부 의존성 추가 (slopcheck)
#    - DoD 변경

# 5. 끝까지
/status                          # 마지막 milestone done 확인
# Vision 달성 → emit <promise>DONE</promise>
```

상세 명령은 [`design/07-runtime-inventory.md`](./design/07-runtime-inventory.md) §7.6 참조.

---

## 핵심 강제 메커니즘 요약

| 원칙 | Sentinel 룰 | 효과 |
|---|---|---|
| P9 TDD-first | `tdd_violation` | 테스트 없는 코드 머지 차단 |
| P10 Compound | `compound_required` | 매 task/phase/cycle 끝에 lessons 캡처 의무 |
| P13 Backlog SSOT | `backlog_singularity` + `discovered_not_logged` | 코드 주석 TODO 단독 금지, 모든 todo는 backlog로 |
| P14 4-step cycle | `workflow_phase_skip` | STATE.md phase 마커 검증, 단계 스킵 차단 |
| P15 DoD mandatory | `dod_required` | 측정 가능한 DoD 없이 milestone 생성/완료 불가 |
| P16 Flexibility traceability | `flexibility_traceability` | 변경 시 영향 backlog 재정렬 + CONTEXT.md 로그 |

(전체 룰: [`design/05-l1-sentinel-quality.md`](./design/05-l1-sentinel-quality.md))
