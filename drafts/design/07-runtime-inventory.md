# 07. 런타임 인벤토리, 상태, 프로필, 명령

> 이 문서는 런타임 파일, 영속 상태, 프로젝트 프로필/컨벤션, 사용자 명령 surface를 정리한다.

---

## 7.1 디렉터리 인벤토리

```text
project/
  .opencode/
    oh-my-openagent.jsonc
    agents/
      maestro.md
      board-clerk.md
      milestone-planner.md
      spec-writer.md
      report-editor.md
      context-librarian.md
      foreman.md
      sentinel.md
      agent-architect.md
      worker-profiles/
        planning.md
        design.md
        implementation.md
        data.md
        security.md
        quality.md
        ops.md
        documentation.md
    skills/
      pm-board/SKILL.md
      pm-delegation/SKILL.md
      project-profile/SKILL.md
      convention-registry/SKILL.md
      sentinel-rules/SKILL.md
      worktree-orchestrator/SKILL.md
      dag-builder/SKILL.md
      tdd-discipline/SKILL.md
      compound-cycle/SKILL.md
      harness-evolution/SKILL.md
      lesson-format/SKILL.md
      proposal-format/SKILL.md
    command/
      start.md
      board.md
      vision.md
      roadmap.md
      milestone.md
      dod.md
      backlog.md
      task.md
      refinement.md
      planning.md
      execution.md
      review.md
      report.md
      mode.md
      evolve.md
      lessons.md
      proposals.md
      tdd-exception.md
    hooks/
      worktree-guard.sh
      state-lock.sh
      pre-commit-tdd-check.sh
      post-commit-compound-emit.sh
      phase-boundary-state-update.sh
      pre-tool-backlog-singularity.sh

  .harness/
    goal.md
    ETHOS.md
    PROJECT_PROFILE.md
    CONVENTIONS.md
    STATE.md
    PROJECT.md
    REQUIREMENTS.md
    ROADMAP.md
    CONTEXT.md
    prd.md
    design-spec.md
    progress.jsonl
    sentinel-log.jsonl
    board/
      vision.md
      roadmap.md
      milestones/_index.md
      milestones/M-*.md
      backlog/_index.md
      backlog/_types.md
      backlog/B-*.md
      tasks/_index.md
      tasks/T-*.md
    phases/
      <N>/PLAN.md
      <N>/SUMMARY.md
      <N>/VERIFY.md
      <N>/specs/
      <N>/reports/
    lessons/
      _pending/
      architecture/
      bugs/
      perf/
      ops/
      process/
      prompts/
    proposals/
      _pending/
      accepted/
      rejected/
      superseded/

  .worktrees/
    <worker-profile>-<T-id>-<U-id>/

  <프로젝트 본체>
```

---

## 7.2 프로젝트 프로필

`.harness/PROJECT_PROFILE.md`는 프로젝트 고유 계층을 기록한다:

```yaml
project_profile:
  project_type: library | app | cli | service | research | mixed
  languages: []
  runtimes: []
  package_managers: []
  test_commands: []
  build_commands: []
  repository_layout:
    source_roots: []
    test_roots: []
    docs_roots: []
  active_worker_profiles:
    - implementation
    - quality
    - documentation
  architecture_rules: []
  code_conventions: []
  management_rules: []
  tool_slots:
    format_commands: []
    lint_commands: []
    typecheck_commands: []
    test_run_commands: []
    coverage_commands: []
    security_scan_commands: []
```

Phase 0가 생성한다. Phase 8이 진화시킨다.

---

## 7.3 컨벤션 레지스트리

`.harness/CONVENTIONS.md`는 다음을 분리한다:

| 계층 | 예시 | 변경 경로 |
|---|---|---|
| 코어 룰 | TDD, Backlog SSOT, 4-step 사이클 | 명시적 주요 결정만 |
| 프로젝트 룰 | 네이밍, 레이아웃, 의존성 방향, 테스트 스타일 | Phase 0 + proposal |
| 워커 룰 | review lens, 허용 경로, verify 명령 | 워커 프로필 proposal |

Sentinel은 프로젝트 고유 점검을 위해 이 레지스트리를 읽는다.

---

## 7.4 STATE.md

`.harness/STATE.md`는 배타 lock으로 보호된다.

```yaml
current_cycle:
  vision: V-001
  roadmap: R-001
  milestone: M-003
  phase: execution
  mode: gated
active_workers:
  - profile: implementation
    task: T-101
    status: in_progress
completed:
  - T-099
blocked: []
last_sentinel_run:
  pass: 12
  fix: 3
  block: 0
recent_events: []
```

쓰기 주체:

| Writer | 범위 |
|---|---|
| Maestro Core | 페이즈/모드 승인 |
| Board Clerk | board 기반 상태 요약 |
| Foreman | active/completed/blocked 워커 |
| Sentinel | 마지막 실행과 발견 |
| Worker | 자신의 task에 대한 최근 이벤트만 |

---

## 7.5 컨텍스트 격리 인벤토리

| 계층 | 보유 내용 |
|---|---|
| Maestro Core | 목표, 인덱스, 활성 항목 요약, 보고서 요약, HITL 질의 |
| Private PM sub-agent | 위임된 PM task 한 건 |
| Foreman | Task Spec, DAG, 워커 요약 |
| 워커 프로필 | 자신의 Task Spec 슬라이스와 worktree |
| 워커 step | 직전 step 산출물만 |

raw diff, raw Sentinel 로그, 하위 계층 출력은 사용자 대면이 아니다.

---

## 7.6 사용자 명령

| 명령 | 목적 |
|---|---|
| `/start "<goal>"` | Phase 0 부트스트랩 |
| `/mode auto|gated|plan-only|dry-run` | run mode |
| `/board` | PM board 요약 |
| `/vision`, `/roadmap`, `/milestone` | PM 엔티티 관리 |
| `/backlog`, `/issue`, `/task` | 작업 항목 관리 |
| `/dod` | DoD 관리와 검증 |
| `/refinement`, `/planning`, `/execution`, `/review` | 페이즈 제어 |
| `/next` | 다음 권장 액션 |
| `/report` | task/마일스톤 보고서 |
| `/sentinel-check` | 수동 품질 점검 |
| `/lessons`, `/compound-now` | 학습 |
| `/evolve`, `/proposals` | 하네스 진화 |
| `/tdd-exception` | 승인된 예외 요청 |
| `/pause`, `/resume`, `/stop` | 라이프사이클 제어 |

사용자는 Maestro Core를 통해 명령을 호출한다.
