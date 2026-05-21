# 01. Overview — 컨셉, 핵심 원칙, 아키텍처 지도

> 이 문서는 최상위 지도다. 운영 디테일까지 들어가지 않고, 하네스의 컨셉, 협상 불가능한 코어, 아키텍처 계층만 설명한다.

---

## 1.1 컨셉

VibeForge Harness는 **규율 있는 바이브코딩**을 위한 자기 진화형 하네스다. 소수의 워크플로 법칙은 고정으로 유지하고, 프로젝트별 워커·컨벤션·레포 구조·관리 규칙은 사용 경험을 통해 진화하도록 한다.

한 줄 요약:

> 사용자는 Maestro Core와만 대화한다. Maestro Core는 결정권을 보유하고 PM 노동은 private sub-agent에 위임한다. Foreman은 승인된 Task Spec을 병렬 작업으로 풀어낸다. 워커 프로필은 격리된 worktree에서 실행된다. Sentinel은 품질과 워크플로 규칙을 지킨다. 교훈은 다음 행동으로 compound되고, Agent-Architect는 하네스 자체의 진화를 제안한다.

---

## 1.2 무엇이 고정이고 무엇이 진화하는가

이 절은 **정책 경계**를 정의한다. VibeForge의 어느 부분이 계약 수준의 규칙이고, 어느 부분이 그 규칙 안에서 자라는 프로젝트별 재료인지 구분한다.

| 영역 | 고정/진화 | 동작 방식 | 예시 |
|---|---|---|---|
| **하네스 계약** | 고정 | Maestro Core, Foreman, Sentinel, 런타임 파일에 의해 항상 강제 | 코드 작업의 TDD, 단일 사용자 창구 Maestro Core, Backlog SSOT, 측정 가능한 DoD, 4-step 마일스톤 사이클, Compound, Phase 8 진화 |
| **프로젝트 프로필** | 진화 | Phase 0에서 작성하고, 레포에서 더 나은 사실이 드러나면 갱신 | 언어, 런타임, 패키지 매니저, 테스트/빌드 명령, 레포 레이아웃, 활성 워커 프로필 |
| **컨벤션 레지스트리** | 진화 | 작게 시작하고, 반복 발견으로 룰이 정당화될 때만 엄격해짐 | 네이밍, 의존성 방향, 포매팅, 테스트 피라미드, 릴리스 규칙 |
| **워커 프로필** | 진화 | seed 프로필에서 시작해 증거를 기반으로 분리/병합/특화/폐기 | implementation, data, quality, ops, documentation, 프로젝트 특화 스페셜리스트 |
| **관리 관행** | 진화 | 고정 PM 모델을 이 프로젝트에 맞춰 어떻게 운용할지 다듬음 | 우선순위 결정 습관, 보고서 형식, 마일스톤 사이징, 리뷰 주기 |

고정 계약은 정확성을 보호한다. 진화 영역은 각 프로젝트가 자신만의 실행 형태를 발전시키도록 한다. 다음 절의 L0/L1/L2 아키텍처가 이 둘을 모두 구현한다.

---

## 1.3 아키텍처 계층

```text
사용자
  <-> L0 Maestro Core
        -> L0-private PM sub-agent
             Board Clerk / Milestone Planner / Spec Writer / Report Editor / Context Librarian
        -> L1 Foreman
             DAG 빌더 / worktree 오케스트레이션 / 머지 직렬화 / Task Report
        -> L1 Sentinel
             워크플로 + 품질 + 컨벤션 게이트
        -> L2 워커 프로필
             Planning / Design / Implementation / Data / Security / Quality / Ops / Documentation / 진화된 프로필
        -> Meta Agent-Architect
             워커·룰·skill·hook·프로세스 변경 제안
```

아키텍처 컴포넌트별 소유 범위:

| 아키텍처 컴포넌트 | 소유 | 경계 |
|---|---|---|
| 사용자 | 목표, 승인, 결정 입력 | Maestro Core 하고만 대화 |
| L0 Maestro Core | 사용자 대화, HITL 질문, 최종 승인, 가시 상태 | PM 장부 작업이나 코드 작업을 직접 수행하지 않음 |
| L0-private PM sub-agent | board 갱신, 마일스톤 플래닝, Task Spec 초안, 보고서 편집, 컨텍스트 조회 | Maestro Core 뒤에서 동작하며 사용자에게 직접 말하지 않음 |
| L1 Foreman | DAG, worktree 디스패치, 머지 오케스트레이션, Task Report | 승인된 Task Spec을 받음, 사용자 raw 요청은 받지 않음 |
| L1 Sentinel | 워크플로, 품질, 컨벤션, DoD 게이트 | 수정/질의/차단은 가능하나 제품 결정은 소유하지 않음 |
| L2 워커 프로필 | task worktree 실행과 step flow | 자신에게 할당된 task 범위만 소유 |
| Meta Agent-Architect | 하네스 개선 제안 | 변경 제안만 하고 채택은 Maestro Core와 사용자가 결정 |

---

## 1.4 협상 불가능한 원칙

원칙은 보호하는 규율 성격별로 묶었다. 원칙 ID는 묶음 순서를 따르며, 다른 문서에서 교차 참조용으로 사용한다.

### 단일 창구, 진짜 결정

| ID | 원칙 | 의미 |
|---|---|---|
| P1 | **HITL은 결정 분기에서만** | 사용자에게는 진짜 결정만 묻는다. 실행 디테일을 매번 묻지 않는다 |
| P2 | **Maestro Core가 유일한 사용자 창구이자 결정권자** | sub-agent는 사용자에게 직접 말하지 않는다 |

### 단일 플래닝 척추

| ID | 원칙 | 의미 |
|---|---|---|
| P3 | **Plan-first** | 구체 계획·리뷰·승인 경로 없이는 코드 작업 진입 금지 |
| P4 | **암묵적 확장 금지** | 에이전트는 backlog/plan 추적 없이 범위를 확장할 수 없다 |
| P5 | **Backlog가 단일 진실 저장소** | 모든 todo/작업은 backlog에 있다. 흩어진 메모 금지 |
| P6 | **4-step 사이클 강제** | Refinement → Planning → Execution → Review를 건너뛸 수 없다 |
| P7 | **Definition of Done 의무화** | 마일스톤은 측정 가능한 DoD와 증거 없이는 완료 불가 |
| P8 | **유연성과 추적성** | 계획은 바뀔 수 있지만, 변경 시 링크를 갱신하고 결정 이력을 append |

### 병렬 실행, TDD 규율

| ID | 원칙 | 의미 |
|---|---|---|
| P9 | **기본은 worktree 병렬** | 독립적인 DAG 단위는 별도 worktree에서 실행 |
| P10 | **만족할 때까지 Ralph 루프** | 수용 + DoD 증거가 갖춰질 때까지 사이클을 반복 |
| P11 | **코드 작업은 TDD-first** | 승인된 예외가 적용되지 않는 한 RED → GREEN → REFACTOR 의무 |

### 컨텍스트 격리, 프로젝트 적응

| ID | 원칙 | 의미 |
|---|---|---|
| P12 | **프로젝트 적응형 에이전트** | 코어 에이전트는 고정. 워커 프로필과 컨벤션은 프로젝트 증거로 진화 |
| P13 | **계층 기반 컨텍스트 격리** | 상위 계층은 구조화된 요약만 본다. 하위 계층의 raw 컨텍스트는 보지 않는다 |

### 게이트 품질, 복리 학습

| ID | 원칙 | 의미 |
|---|---|---|
| P14 | **상시 품질 감시** | Sentinel은 커밋·페이즈·머지 게이트에서 항상 동작 |
| P15 | **매 사이클 Compound** | task·페이즈·마일스톤 단위의 학습을 포착하고 재사용 |
| P16 | **자기 진화 하네스** | 사용 데이터가 하네스 자체의 개선 제안을 만들어낸다 |

---

## 1.5 문서 맵

읽는 순서:

| # | 파일 | 범위 |
|---|---|---|
| 01 | `01-overview.md` | 컨셉, 고정 룰, 진화 영역, 아키텍처 지도 |
| 02 | `02-l0-maestro-pm.md` | Maestro Core, private PM sub-agent, PM 엔티티 모델 |
| 03 | `03-l1-foreman-execution.md` | Foreman 실행 계층, DAG, worktree, Task Spec/Report |
| 04 | `04-l2-worker-profiles.md` | 워커 프로필 모델과 워커 내부 step flow |
| 05 | `05-l1-sentinel-quality.md` | Sentinel 품질/워크플로 게이트와 verifier |
| 06 | `06-cross-layer-workflows.md` | Phase 0, 4-step 사이클, Compound, Phase 8, Ralph 루프 |
| 07 | `07-runtime-inventory.md` | 런타임 파일, 프로젝트 프로필, 컨벤션, 명령 |
| 08 | `08-implementation-roadmap.md` | OpenCode 매핑, 구현 순서, 열린 결정 |
