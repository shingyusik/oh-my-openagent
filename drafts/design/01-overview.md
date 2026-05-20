# 01. Overview - Concept, Core Principles, Architecture Map

> This is the top-level map. It explains the harness concept, the non-negotiable core, and the architecture layers without diving into operational detail.

---

## 1.1 Concept

VibeForge Harness is a self-evolving harness for disciplined vibe coding. It keeps a small set of workflow laws fixed and lets project-specific workers, conventions, repository layout, and management rules evolve from use.

One sentence:

> User talks only to Maestro Core. Maestro Core owns decisions and delegates PM labor to private sub-agents. Foreman turns approved Task Specs into parallel work. Worker profiles execute in isolated worktrees. Sentinel guards quality and workflow rules. Lessons compound into future behavior, and Agent Architect proposes harness evolution.

---

## 1.2 Fixed Core vs Adaptive Layer

| Layer | Status | Examples |
|---|---|---|
| **Fixed core** | Always enforced | TDD for code work, Maestro Core as single user surface, Backlog SSOT, measurable DoD, 4-step milestone cycle, Sentinel, Compound, Phase 8 evolution |
| **Project profile** | Created at Phase 0, evolves over time | languages, runtimes, package managers, test/build commands, repository layout, active worker profiles |
| **Convention registry** | Starts small, grows from repeated findings | naming, dependency direction, formatting, test pyramid, release rules |
| **Worker profiles** | Seeded, then split/merge/deprecate by evidence | implementation, data, quality, ops, documentation, project-specific specialists |

The core protects correctness. The adaptive layer lets each project develop its own execution shape.

---

## 1.3 Architecture Layers

```text
User
  <-> L0 Maestro Core
        -> L0-private PM sub-agents
             Board Clerk / Milestone Planner / Spec Writer / Report Editor / Context Librarian
        -> L1 Foreman
             DAG builder / worktree orchestration / merge serialisation / Task Report
        -> L1 Sentinel
             workflow + quality + convention gates
        -> L2 Worker Profiles
             Planning / Design / Implementation / Data / Security / Quality / Ops / Documentation / evolved profiles
        -> Meta Agent Architect
             proposals for workers, rules, skills, hooks, and process changes
```

Important ownership boundaries:

| Boundary | Rule |
|---|---|
| User surface | Only Maestro Core speaks to the user |
| PM write path | Board Clerk writes `board/*`; Maestro Core approves |
| Execution | Foreman owns DAG/worktree/merge orchestration |
| Code work | Worker profiles own their task worktree and step flow |
| Quality gates | Sentinel can fix, ask, or block |
| Evolution | Agent Architect proposes; Maestro Core and user decide |

---

## 1.4 Non-Negotiable Principles

| ID | Principle | Meaning |
|---|---|---|
| P1 | **Plan-first** | No code work before a concrete plan, review, and approval path |
| P2 | **Project-adaptive agents** | Core agents are fixed; worker profiles and conventions evolve by project evidence |
| P3 | **Context isolation by hierarchy** | Higher layers see structured summaries, not raw lower-layer context |
| P4 | **Worktree parallelism by default** | Independent DAG units run in separate worktrees |
| P5 | **Always-on quality supervision** | Sentinel runs at commit, phase, and merge gates |
| P6 | **Ralph loop until satisfied** | The harness keeps cycling until acceptance and DoD evidence exist |
| P7 | **HITL only at decision forks** | The user is asked for real decisions, not routine execution details |
| P8 | **No silent expansion** | Agents cannot expand scope without backlog/plan traceability |
| P9 | **TDD-first for code work** | RED -> GREEN -> REFACTOR is mandatory unless approved exception applies |
| P10 | **Compound every cycle** | Task, phase, and milestone learning is captured and reused |
| P11 | **Self-evolving harness** | Usage data drives proposed improvements to the harness itself |
| P12 | **Maestro Core is the only user surface and decision owner** | Sub-agents never speak directly to the user |
| P13 | **Backlog is the single source of truth** | All todo/work items live in backlog, not scattered notes |
| P14 | **4-step cycle enforced** | Refinement -> Planning -> Execution -> Review cannot be skipped |
| P15 | **Definition of Done mandatory** | Milestones need measurable DoD and evidence before completion |
| P16 | **Flexibility with traceability** | Plans may change, but changes update links and append decision history |

---

## 1.5 Document Map

Read in order:

| # | File | Scope |
|---|---|---|
| 01 | `01-overview.md` | Concept, fixed core, adaptive layer, architecture map |
| 02 | `02-l0-maestro-pm.md` | Maestro Core, private PM sub-agents, PM entity model |
| 03 | `03-l1-foreman-execution.md` | Foreman execution layer, DAG, worktrees, Task Spec/Report |
| 04 | `04-l2-worker-profiles.md` | Worker profile model and worker internal step flows |
| 05 | `05-l1-sentinel-quality.md` | Sentinel quality/workflow gates and verifier |
| 06 | `06-cross-layer-workflows.md` | Phase 0, 4-step cycle, Compound, Phase 8, Ralph loop |
| 07 | `07-runtime-inventory.md` | Runtime files, project profile, conventions, commands |
| 08 | `08-implementation-roadmap.md` | OpenCode mapping, implementation order, open decisions |
