# 08. 구현 로드맵과 열린 결정

> 이 문서는 아키텍처를 oh-my-openagent/OpenCode 구현 작업으로 매핑하고, 남은 결정사항을 추적한다.

---

## 8.1 OpenCode 매핑

| 하네스 컨셉 | 메커니즘 | 구현 |
|---|---|---|
| Maestro Core | Prometheus 확장 + delegation skill | `.opencode/agents/maestro.md` |
| Private PM sub-agent | 커스텀 에이전트 + PM skill | `board-clerk.md`, `milestone-planner.md`, `spec-writer.md`, `report-editor.md`, `context-librarian.md` |
| Foreman | Atlas류 실행 에이전트 | `.opencode/agents/foreman.md` |
| Sentinel | agent + hook + rule skill | `.opencode/agents/sentinel.md`, hooks, `sentinel-rules` |
| 워커 프로필 | 프로젝트 적응형 agent profile | `.opencode/agents/worker-profiles/*.md` |
| 프로젝트 프로필 | skill + 런타임 파일 | `project-profile/SKILL.md`, `.harness/PROJECT_PROFILE.md` |
| 컨벤션 레지스트리 | skill + 런타임 파일 | `convention-registry/SKILL.md`, `.harness/CONVENTIONS.md` |
| Worktree 실행 | git worktree + 가드된 hook | `worktree-orchestrator`, `worktree-guard.sh` |
| DAG 빌더 | skill/Foreman 프롬프트 | `dag-builder/SKILL.md` |
| TDD 규율 | 워커 skill | `tdd-discipline/SKILL.md` |
| Compound | skill + hook | `compound-cycle/SKILL.md`, phase hook |
| Phase 8 진화 | Agent-Architect skill | `harness-evolution/SKILL.md` |
| STATE lock | shell 유틸 | `state-lock.sh` |

---

## 8.2 구현 순서

1. 고정 코어로부터 `.harness/ETHOS.md` v0 작성
2. Maestro Core 프롬프트 v0를 얇은 오케스트레이터로 작성
3. private PM sub-agent 프롬프트 작성
4. `pm-delegation` schema 구현
5. `pm-board` 엔티티 CRUD와 인덱스 정합성 구현
6. Task Spec, Task Report, Report Summary schema 정의
7. `project-profile`과 `convention-registry` 구현
8. seed 워커 프로필 구현
9. 프로필 선언 test 명령에 대한 `tdd-discipline` 구현
10. Sentinel 코어 룰과 프로필 슬롯 읽기 구현
11. Foreman DAG 빌더와 worktree 가드 구현
12. Compound lesson 포맷과 phase hook 구현
13. Phase 8 proposal 포맷과 Agent-Architect flow 구현
14. 사용자 명령 파일 구현
15. 소규모 범용 코딩 dry-run 실행
16. dry-run 발견을 v0.6 결정에 반영

---

## 8.3 Dry-Run 수용 기준

코어 하네스를 충분히 운동시킬 만큼의 파일·테스트·워크플로 결정을 가진 소규모 범용 코딩 프로젝트를 사용한다.

Dry-run이 다음을 증명해야 한다:

- 사용자는 Maestro Core하고만 대화
- Phase 0이 `PROJECT_PROFILE.md`와 `CONVENTIONS.md`를 생성
- board 엔티티가 생성되고 링크됨
- DoD가 측정 가능
- Refinement → Planning → Execution → Review가 강제됨
- Foreman이 사용자 대면 없이 DAG와 Task Report를 산출
- 워커 프로필 할당이 프로젝트 프로필을 사용
- Sentinel이 Backlog SSOT와 placeholder 위반을 잡아냄
- Task Spec/Report/Report Summary schema가 동작
- Phase 8이 관측 데이터에서 최소 하나의 proposal을 생성

---

## 8.4 v0.6 잠금 기본값

### Maestro Core 분리

| 결정 | 기본값 |
|---|---|
| 사용자 대면 채널 | Maestro Core 전용 |
| PM 쓰기 | 승인 후 Board Clerk 전용 |
| 플래닝 | Milestone Planner proposal + Maestro Core 승인 |
| task spec | Spec Writer 초안 + Maestro Core 승인 |
| 보고서 surface | Report Editor 요약 + Maestro Core 최종 표현 |
| 컨텍스트 | 인덱스 + 활성 요약, 본문은 on-demand 읽기 |

### VibeForge Harness

| 결정 | 기본값 |
|---|---|
| 고정 코어 | TDD, Maestro Core, Backlog SSOT, DoD, 4-step 사이클, Sentinel, Compound, Phase 8 |
| 진화 프로젝트 영역 | 워커 프로필, 컨벤션, 레포 레이아웃, 명령, 릴리스 규칙 |
| 프로필 파일 | `.harness/PROJECT_PROFILE.md`, `.harness/CONVENTIONS.md` |
| Sentinel 프로젝트 룰 | 하드코딩된 스택 가정이 아니라 프로필/레지스트리에서 읽음 |
| 진화 | 반복 발견과 사용자 선호가 proposal로 |

---

## 8.5 열린 결정

### PM 모델

- board git 정책: Board Clerk 쓰기마다 커밋할 것인가, 페이즈 단위로 묶을 것인가?
- 엔티티 그래프 렌더링: list, ASCII tree, Mermaid, 또는 모드 의존?
- stale 엔티티 정책: 휴면 P3/보류 작업을 언제 archive할 것인가?
- 다중 활성 마일스톤: 기본은 단일, 그러나 P0 hotfix가 어떻게 끼어드는가?

### DoD

- verify 명령 형식: inline shell, script 경로, 또는 command/env/cwd schema?
- 마일스톤당 권장 DoD 개수
- Level 4 real data flow가 의무인 시점과 불가능한 시점

### 워커 프로필

- 모든 seed 프로필로 시작할 것인가, 프로젝트별 최소 부분집합으로 시작할 것인가?
- 프로필을 두 스페셜리스트로 분리하는 임계
- 저가치 프로필을 병합하는 임계
- 워커 프로필 변경 버전 관리 방식

### 컨벤션 진화

- 사용자 선호가 프로젝트 컨벤션이 되는 시점
- 룰 proposal까지의 반복 발견 횟수
- 소음이 큰 프로젝트 룰의 폐기 방식

### 운영

- 기본 `max_concurrent_worktrees`
- worktree 정리 정책
- 예산 게이트 형식: 토큰, 달러, 시간 또는 결합
- 프로젝트 유형별 배포/릴리스 게이트

### Sentinel

- 다중 BLOCK 해결 순서
- 룰 confidence/fingerprint 알고리즘
- 프로필 슬롯 자동 감지 vs 명시적 사용자 확인
