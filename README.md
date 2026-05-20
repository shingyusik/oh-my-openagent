# VibeForge Harness

> Status: design in progress.
> Base runtime: `oh-my-openagent`, an OpenCode plugin architecture.

VibeForge Harness is a self-evolving harness for disciplined vibe coding.

The harness keeps a small set of core workflow laws fixed, then lets project-specific workers, code conventions, repository rules, and management practices evolve from actual use.

## One Sentence

User talks only to Maestro Core. Maestro Core owns decisions and delegates PM labor to private PM sub-agents. Foreman turns approved Task Specs into parallel work. Worker profiles execute in isolated worktrees. Sentinel guards quality and workflow rules. Lessons compound into future behavior, and Agent Architect proposes harness evolution.

## What Stays Fixed

The harness always enforces these rules:

| Core | Meaning |
|---|---|
| Maestro Core | The only user-facing surface and final decision owner |
| Backlog SSOT | All work items live in the backlog, not scattered notes |
| 4-step cycle | Refinement -> Planning -> Execution -> Review |
| DoD mandatory | Every milestone needs measurable Definition of Done |
| TDD-first | Code work follows RED -> GREEN -> REFACTOR unless explicitly exempted |
| Worktree parallelism | Independent units run in isolated worktrees |
| Sentinel gates | Quality, workflow, convention, and DoD checks run at commit/phase/merge gates |
| Compound learning | Lessons from tasks, phases, and milestones are captured and reused |

Project-specific details evolve inside those rules.

## What Evolves By Project

The project shapes the harness over time:

| Evolving Area | Evolves From |
|---|---|
| Project profile | Languages, runtimes, package managers, commands, repository layout |
| Convention registry | Naming, dependency direction, formatting, test pyramid, release rules |
| Worker profiles | Seed profiles split, merge, specialize, or disappear based on evidence |
| Management rules | Backlog, milestone, review, and reporting habits refined from use |
| Harness architecture | Agent Architect proposes new hooks, skills, workers, and rules |

The point is to enforce discipline while letting each project grow its own execution shape.

## Architecture

```text
User
  <-> L0 Maestro Core
        -> private PM sub-agents
             Board Clerk / Milestone Planner / Spec Writer / Report Editor / Context Librarian
        -> L1 Foreman
             DAG builder / worktree orchestration / merge serialization / Task Report
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

## Operating Loop

Each milestone runs through one enforced cycle:

1. Refinement: backlog is clarified, deduplicated, prioritized, and connected to roadmap.
2. Planning: the current milestone is selected, measurable DoD is written, and Task Specs are prepared.
3. Execution: Foreman dispatches Task Specs into worker worktrees and serializes merge flow.
4. Review: Sentinel verifies DoD evidence, lessons are captured, and the next cycle is adjusted.

The harness keeps cycling until the current Vision is satisfied.

## Repository Map

| Path | Role |
|---|---|
| [`drafts/my-harness-design.md`](drafts/my-harness-design.md) | Current design entry point |
| [`drafts/design/`](drafts/design/) | Top-down harness architecture and workflow design |
| [`drafts/review-v0.5.1.md`](drafts/review-v0.5.1.md) | Design review notes |
| [`drafts/surveys/`](drafts/surveys/) | External harness survey material |
| [`drafts/CHANGELOG.md`](drafts/CHANGELOG.md) | Design change history |
| [`docs/`](docs/) | Original OMO definition and user documentation |
| [`src/`](src/) | Existing OpenCode plugin implementation base |

## Design Documents

Read these in order:

| # | File | Scope |
|---|---|---|
| 01 | [`Overview`](drafts/design/01-overview.md) | Concept, fixed rules, evolving project areas, architecture map |
| 02 | [`L0 Maestro + PM`](drafts/design/02-l0-maestro-pm.md) | Maestro Core, private PM sub-agents, PM entity model |
| 03 | [`L1 Foreman`](drafts/design/03-l1-foreman-execution.md) | Task Spec, DAG, worktree orchestration, Task Report |
| 04 | [`L2 Worker Profiles`](drafts/design/04-l2-worker-profiles.md) | Worker profile model and internal step flows |
| 05 | [`L1 Sentinel`](drafts/design/05-l1-sentinel-quality.md) | Quality/workflow gates and verifier |
| 06 | [`Cross-Layer Workflows`](drafts/design/06-cross-layer-workflows.md) | Phase 0, 4-step cycle, Compound, Phase 8, Ralph loop |
| 07 | [`Runtime Inventory`](drafts/design/07-runtime-inventory.md) | Runtime files, project profile, conventions, commands |
| 08 | [`Implementation Roadmap`](drafts/design/08-implementation-roadmap.md) | OpenCode mapping, implementation order, open decisions |

## Current Direction

The immediate design goal for VibeForge is to keep the core small and strict:

- one user surface
- one backlog source of truth
- one milestone cycle
- measurable DoD
- TDD-first code work
- quality gates that can block
- lessons that compound into future behavior

Inside that frame, the harness should adapt to the user, the project, and the evidence produced by repeated work.
