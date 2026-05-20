# 01. Overview - Concept, Core Principles, Architecture Map

> This is the top-level map. It explains the harness concept, the non-negotiable core, and the architecture layers without diving into operational detail.

---

## 1.1 Concept

VibeForge Harness is a self-evolving harness for disciplined vibe coding. It keeps a small set of workflow laws fixed and lets project-specific workers, conventions, repository layout, and management rules evolve from use.

One sentence:

> User talks only to Maestro Core. Maestro Core owns decisions and delegates PM labor to private sub-agents. Foreman turns approved Task Specs into parallel work. Worker profiles execute in isolated worktrees. Sentinel guards quality and workflow rules. Lessons compound into future behavior, and Agent Architect proposes harness evolution.

---

## 1.2 What Stays Fixed vs What Evolves

This section defines the policy boundary: which parts of VibeForge are contract-level rules and which parts are project-specific material that grows inside those rules.

| Part | Fixed or Evolving | How It Behaves | Examples |
|---|---|---|---|
| **Harness contract** | Fixed | Always enforced by Maestro Core, Foreman, Sentinel, and the runtime files | TDD for code work, Maestro Core as single user surface, Backlog SSOT, measurable DoD, 4-step milestone cycle, Compound, Phase 8 evolution |
| **Project profile** | Evolving | Created during Phase 0, then updated when the repository reveals better facts | languages, runtimes, package managers, test/build commands, repository layout, active worker profiles |
| **Convention registry** | Evolving | Starts small and becomes stricter only when repeated findings justify a rule | naming, dependency direction, formatting, test pyramid, release rules |
| **Worker profiles** | Evolving | Start from seed profiles, then split, merge, specialize, or retire by evidence | implementation, data, quality, ops, documentation, project-specific specialists |
| **Management practice** | Evolving | Refines how the fixed PM model is operated for this project | prioritization habits, report shape, milestone sizing, review cadence |

The fixed contract protects correctness. The evolving parts let each project develop its own execution shape. The L0/L1/L2 architecture in the next section implements both.

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

Ownership by architecture component:

| Architecture Component | Owns | Boundary |
|---|---|---|
| User | Goals, approvals, and decision input | Talks only with Maestro Core |
| L0 Maestro Core | User conversation, HITL questions, final approval, visible status | Does not perform PM bookkeeping or code work directly |
| L0-private PM sub-agents | Board updates, milestone planning, Task Spec drafting, report editing, context lookup | Operate behind Maestro Core and never speak directly to the user |
| L1 Foreman | DAG, worktree dispatch, merge orchestration, Task Report | Receives approved Task Specs, not raw user requests |
| L1 Sentinel | Workflow, quality, convention, and DoD gates | Can fix, ask, or block, but does not own product decisions |
| L2 Worker Profiles | Task worktree execution and step flow | Own only their assigned task scope |
| Meta Agent Architect | Harness improvement proposals | Proposes changes; Maestro Core and user decide |

---

## 1.4 Non-Negotiable Principles

The principles are grouped by the kind of discipline they protect. Principle IDs follow this grouped order and are used for cross-document references.

### Single Surface, Real Decisions

| ID | Principle | Meaning |
|---|---|---|
| P1 | **HITL only at decision forks** | The user is asked for real decisions, not routine execution details |
| P2 | **Maestro Core is the only user surface and decision owner** | Sub-agents never speak directly to the user |

### One Planning Spine

| ID | Principle | Meaning |
|---|---|---|
| P3 | **Plan-first** | No code work before a concrete plan, review, and approval path |
| P4 | **No silent expansion** | Agents cannot expand scope without backlog/plan traceability |
| P5 | **Backlog is the single source of truth** | All todo/work items live in backlog, not scattered notes |
| P6 | **4-step cycle enforced** | Refinement -> Planning -> Execution -> Review cannot be skipped |
| P7 | **Definition of Done mandatory** | Milestones need measurable DoD and evidence before completion |
| P8 | **Flexibility with traceability** | Plans may change, but changes update links and append decision history |

### Parallel Execution, TDD Discipline

| ID | Principle | Meaning |
|---|---|---|
| P9 | **Worktree parallelism by default** | Independent DAG units run in separate worktrees |
| P10 | **Ralph loop until satisfied** | The harness keeps cycling until acceptance and DoD evidence exist |
| P11 | **TDD-first for code work** | RED -> GREEN -> REFACTOR is mandatory unless approved exception applies |

### Isolated Context, Project Adaptation

| ID | Principle | Meaning |
|---|---|---|
| P12 | **Project-adaptive agents** | Core agents are fixed; worker profiles and conventions evolve by project evidence |
| P13 | **Context isolation by hierarchy** | Higher layers see structured summaries, not raw lower-layer context |

### Gated Quality, Compounding Learning

| ID | Principle | Meaning |
|---|---|---|
| P14 | **Always-on quality supervision** | Sentinel runs at commit, phase, and merge gates |
| P15 | **Compound every cycle** | Task, phase, and milestone learning is captured and reused |
| P16 | **Self-evolving harness** | Usage data drives proposed improvements to the harness itself |

---

## 1.5 Document Map

Read in order:

| # | File | Scope |
|---|---|---|
| 01 | `01-overview.md` | Concept, fixed rules, evolving project areas, architecture map |
| 02 | `02-l0-maestro-pm.md` | Maestro Core, private PM sub-agents, PM entity model |
| 03 | `03-l1-foreman-execution.md` | Foreman execution layer, DAG, worktrees, Task Spec/Report |
| 04 | `04-l2-worker-profiles.md` | Worker profile model and worker internal step flows |
| 05 | `05-l1-sentinel-quality.md` | Sentinel quality/workflow gates and verifier |
| 06 | `06-cross-layer-workflows.md` | Phase 0, 4-step cycle, Compound, Phase 8, Ralph loop |
| 07 | `07-runtime-inventory.md` | Runtime files, project profile, conventions, commands |
| 08 | `08-implementation-roadmap.md` | OpenCode mapping, implementation order, open decisions |
