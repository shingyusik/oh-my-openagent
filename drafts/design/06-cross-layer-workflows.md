# 06. Cross-Layer Workflows

> This document explains workflows that cross L0, L1, L2, Sentinel, and evolution. Layer-local details live in documents 02-05.

---

## 6.1 Run Modes

| Mode | Behavior | HITL |
|---|---|---|
| `auto` | user interview, then autonomous cycles | decision forks only |
| `gated` | user confirms each phase boundary | every phase |
| `plan-only` | Refinement + Planning, no code work | planning complete |
| `dry-run` | simulate phases and gates, no merge/deploy | decision forks only |

Mode-independent HITL:

- irreversible work
- high security risk
- same Sentinel BLOCK repeats 3 times
- uncertain external dependency
- Vision/DoD change
- budget threshold exceeded

---

## 6.2 Phase 0: Project Bootstrap

```text
User goal
  -> Maestro Core interview
  -> Context Librarian / Explore / Oracle research
  -> PROJECT_PROFILE.md draft
  -> CONVENTIONS.md draft
  -> Vision + Roadmap + first Milestone + DoD
  -> initial Backlog
  -> user approval
  -> first Refinement
```

Phase 0 output:

- `.harness/PROJECT_PROFILE.md`
- `.harness/CONVENTIONS.md`
- `board/vision.md`
- `board/roadmap.md`
- first `board/milestones/M-*.md`
- `board/backlog/_index.md`
- initial backlog items

---

## 6.3 Milestone 4-Step Cycle

```text
Refinement -> Planning -> Execution -> Review -> next Refinement
```

### Refinement

Purpose: keep backlog as the single source of truth.

Flow:

1. Context Librarian briefs active milestone and backlog
2. Milestone Planner proposes triage, priority, dedup, links
3. Maestro Core approves or asks user
4. Board Clerk writes changes
5. Sentinel checks Backlog SSOT and phase marker

### Planning

Purpose: select milestone work and make it verifiable.

Flow:

1. Milestone Planner proposes milestone scope and DoD
2. Planning/Design workers enrich requirements/specs if needed
3. Spec Writer proposes task breakdown
4. Maestro Core approves
5. Board Clerk writes M/B/T links
6. Sentinel checks DoD, phase order, placeholder plans

### Execution

Purpose: execute approved tasks through Foreman.

Flow:

1. Maestro Core selects ready task
2. Spec Writer drafts Task Spec
3. Foreman builds DAG and worktrees
4. Worker profiles execute step flows
5. Sentinel runs at commit and merge gates
6. Foreman returns Task Report
7. Report Editor summarizes
8. Maestro Core surfaces result

### Review

Purpose: prove milestone completion and learn.

Flow:

1. DoD verify commands run
2. Sentinel 4-level verifier evaluates evidence
3. failed DoD creates backlog work
4. Tier 3 compound extracts cross-cutting lessons
5. next milestone/backlog priority is adjusted
6. Maestro Core reports and proceeds

---

## 6.4 Compound Learning

| Tier | When | Output |
|---|---|---|
| Tier 1 | worker task end | lesson candidate |
| Tier 2 | phase boundary | deduped lesson markdown |
| Tier 3 | milestone Review | system-level pattern and ETHOS candidates |

Lesson shape:

```yaml
symptom:
root_cause:
what_worked:
prevent_next_time:
related_u_ids:
category:
```

Lessons are consumed by Context Librarian and worker profile prompts in later tasks.

---

## 6.5 Phase 8 Harness Evolution

Triggers:

- same Sentinel finding repeats
- same lesson recurs across cycles
- user preference repeats
- worker profile produces avoidable failures
- proposal acceptance/rejection pattern reveals drift

Agent Architect flow:

```text
diagnose -> baseline -> design -> propose -> peer-review -> compound
```

Proposal targets:

- worker profile add/split/merge/deprecate
- Sentinel rule change
- convention registry update
- project profile update
- skill/hook/prompt improvement
- ETHOS candidate

Acceptance:

- Maestro Core presents proposal
- user accepts/rejects/defers
- accepted changes are tagged `evolved: true`
- effect is measured for future deprecation

---

## 6.6 Ralph Loop Termination

The harness is done only when:

- all P0/P1 tasks in current scope are done or intentionally parked
- all milestone DoD checks pass
- Sentinel has no unresolved BLOCK
- integration/security/quality gates pass according to project profile
- open P0/P1 backlog items are zero or explicitly deferred
- Tier 3 compound is written
- user has not rejected completion

If not done:

- unfinished task -> Foreman reinject
- failed DoD -> backlog item/task
- missing rule/profile/convention -> Phase 8 proposal
- ambiguous decision -> Maestro Core HITL

---

## 6.7 Cross-Layer Invariants

| Invariant | Enforced By |
|---|---|
| only Maestro Core is user-facing | P2 + runtime channel isolation |
| no phase skipping | Sentinel `workflow_phase_skip` |
| no hidden todo | Sentinel `backlog_singularity` / `discovered_not_logged` |
| no code without evidence | TDD + Self-Check + 4-level verifier |
| no silent scope growth | Task Spec constraints + Sentinel `scope_creep` |
| learning is mandatory | Compound gates |
| evolution is explicit | proposal workflow |
