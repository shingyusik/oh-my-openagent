# 02. L0 - Maestro Core and PM Layer

> This document owns the user-facing layer and PM memory model. It does not describe worker internals or Sentinel rule detail.

---

## 2.1 L0 Responsibilities

Maestro Core is deliberately narrow:

| Responsibility | Maestro Core owns? | Notes |
|---|---:|---|
| User conversation | yes | The user only sees Maestro Core |
| Final decisions | yes | Approve, reject, defer, ask user |
| HITL gates | yes | Irreversible work, large scope changes, ambiguous tradeoffs |
| PM file writes | no | Delegated to Board Clerk after approval |
| Planning analysis | no | Delegated to Milestone Planner |
| Task Spec drafting | no | Delegated to Spec Writer |
| Report rewriting | no | Delegated to Report Editor |
| Context search | no | Delegated to Context Librarian |

Core rule:

> Maestro Core owns conversation and decisions, not PM busywork.

---

## 2.2 Private PM Sub-Agents

Private PM sub-agents never speak directly to the user. They return schema-shaped artifacts to Maestro Core.

| Agent | Role | Write Access |
|---|---|---|
| **Board Clerk** | CRUD for Vision/Roadmap/Milestone/Backlog/Task, `_index.md`, cross-links, state transitions | `board/*`, approved changes only |
| **Milestone Planner** | Refinement, prioritization, dependency analysis, milestone/DoD proposals | none, proposal only |
| **Spec Writer** | Converts selected backlog/task into Foreman Task Spec | none, spec draft only |
| **Report Editor** | Converts raw Foreman/Sentinel report into user-facing summary | none, summary only |
| **Context Librarian** | Retrieves relevant board bodies, lessons, and prior decisions on demand | read-only |

Standard delegation contract:

| Request | Handler | Output | Approval |
|---|---|---|---|
| `context_brief` | Context Librarian | source ids + short brief | none |
| `refinement_proposal` | Milestone Planner | triage, priority, merge candidates | Maestro Core |
| `board_patch_request` | Board Clerk | applied changes + integrity check | Maestro Core before write |
| `task_spec_draft` | Spec Writer | Task Spec markdown/schema | Maestro Core before dispatch |
| `report_summary` | Report Editor | user-facing summary + asks | Maestro Core before surface |

Long free-form reports are banned at this layer. Every response must be structured.

---

## 2.3 PM Entity Model

```text
Vision (V-001)
  -> Roadmap (R-*)
       -> Milestone (M-*) with measurable DoD
            -> Backlog (single source of truth)
                 -> BacklogItem (B-*)
                      -> Task (T-*) dispatched to Foreman
```

| Entity | Meaning | Owner |
|---|---|---|
| Vision | Desired end state and non-goals | Maestro Core approves, Board Clerk writes |
| Roadmap | Strategic slices toward Vision | Board Clerk |
| Milestone | A verifiable delivery checkpoint | Milestone Planner proposes, Board Clerk writes |
| Backlog | Single source of all future work | Board Clerk |
| BacklogItem | Feature, bug, tech debt, research, spike | Board Clerk, Sentinel/worker can auto-create ideas |
| Task | Dispatchable execution unit | Spec Writer drafts, Foreman executes |

Backlog item types:

| Type | Meaning |
|---|---|
| `feature` | User or product capability |
| `bug` | Incorrect behavior or failed DoD |
| `tech_debt` | Structural issue that slows future work |
| `research` | Unknown that must be resolved before implementation |
| `spike` | Time-boxed experiment |

---

## 2.4 State Transitions

```text
Refinement
  external input / worker finding -> B-* status: idea
  Milestone Planner proposes triage
  Maestro Core approves
  Board Clerk updates status: triaged

Planning
  Milestone Planner proposes milestone + selected backlog items
  Spec Writer proposes task breakdown
  Maestro Core approves
  Board Clerk writes M-*/B-*/T-* links

Execution
  Maestro Core picks ready T-*
  Spec Writer drafts Task Spec
  Maestro Core approves dispatch
  Foreman executes
  Board Clerk reflects task status changes

Review
  DoD verify results update milestone state
  failed DoD creates/updates backlog
  Review summary informs next Refinement
```

---

## 2.5 Maestro Core Context Budget

Maestro Core keeps only:

- user goal and current run mode
- `board/*/_index.md`
- active Vision/Milestone/Task summaries
- recent Report Editor summaries
- pending HITL asks

It does not keep full board bodies, raw diffs, Sentinel logs, or worker output in context. Those are requested on demand through Context Librarian or Report Editor.

---

## 2.6 User Commands Owned by L0

Commands are user-facing, but implementation may delegate internally.

| Command Family | Meaning |
|---|---|
| `/start`, `/mode`, `/pause`, `/resume`, `/stop` | lifecycle and run mode |
| `/board`, `/vision`, `/roadmap`, `/milestone` | PM entity views/changes |
| `/backlog`, `/issue`, `/task` | work item management |
| `/dod` | DoD authoring and verification |
| `/next`, `/refinement`, `/planning`, `/execution`, `/review` | phase control |
| `/report` | summaries and raw report access when explicitly requested |
| `/lessons`, `/proposals`, `/evolve` | learning and evolution |

The output still comes from Maestro Core, even when other agents do the work.
