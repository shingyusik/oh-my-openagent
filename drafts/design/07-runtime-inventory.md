# 07. Runtime Inventory, State, Profiles, and Commands

> This document lists runtime files, persistent state, project profile/conventions, and user command surface.

---

## 7.1 Directory Inventory

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

  <project body>
```

---

## 7.2 Project Profile

`.harness/PROJECT_PROFILE.md` records the project-specific layer:

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

Phase 0 creates it. Phase 8 evolves it.

---

## 7.3 Convention Registry

`.harness/CONVENTIONS.md` separates:

| Layer | Examples | Change Path |
|---|---|---|
| Core rules | TDD, Backlog SSOT, 4-step cycle | only explicit major decision |
| Project rules | naming, layout, dependency direction, test style | Phase 0 + proposals |
| Worker rules | review lens, allowed paths, verify commands | worker profile proposals |

Sentinel reads this registry for project-specific checks.

---

## 7.4 STATE.md

`.harness/STATE.md` is protected by an exclusive lock.

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

Writers:

| Writer | Scope |
|---|---|
| Maestro Core | phase/mode approval |
| Board Clerk | board-derived status summary |
| Foreman | active/completed/blocked workers |
| Sentinel | last run and findings |
| Worker | recent event about own task only |

---

## 7.5 Context Isolation Inventory

| Layer | Holds |
|---|---|
| Maestro Core | goal, indexes, active item summaries, report summaries, HITL asks |
| Private PM sub-agent | one delegated PM task |
| Foreman | Task Spec, DAG, worker summaries |
| Worker profile | own Task Spec slice and worktree |
| Worker step | prior step artifact only |

Raw diffs, raw Sentinel logs, and lower-layer outputs are not user-facing.

---

## 7.6 User Commands

| Command | Purpose |
|---|---|
| `/start "<goal>"` | Phase 0 bootstrap |
| `/mode auto|gated|plan-only|dry-run` | run mode |
| `/board` | PM board summary |
| `/vision`, `/roadmap`, `/milestone` | PM entity management |
| `/backlog`, `/issue`, `/task` | work item management |
| `/dod` | DoD management and verification |
| `/refinement`, `/planning`, `/execution`, `/review` | phase controls |
| `/next` | next recommended action |
| `/report` | task/milestone reports |
| `/sentinel-check` | manual quality check |
| `/lessons`, `/compound-now` | learning |
| `/evolve`, `/proposals` | harness evolution |
| `/tdd-exception` | approved exception request |
| `/pause`, `/resume`, `/stop` | lifecycle control |

The user invokes commands through Maestro Core.
