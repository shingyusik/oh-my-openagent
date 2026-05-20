# 05. L1 - Sentinel Quality and Workflow Gates

> Sentinel is the always-on guard layer. It enforces core rules, reads project/worker rules from profile files, and returns PASS/FIX/ASK/BLOCK.

---

## 5.1 Trigger Points

| Trigger | When |
|---|---|
| post-commit | every worker commit |
| phase boundary | entering/leaving Refinement, Planning, Execution, Review |
| merge gate | before Foreman merges a worktree/wave |
| DoD verify | milestone Review |
| explicit call | `/sentinel-check [scope]` |

---

## 5.2 Result Types

| Result | Meaning |
|---|---|
| PASS | proceed |
| FIX | Sentinel can produce a mechanical patch; worker applies and reruns |
| ASK | judgment needed; batch to Maestro Core |
| BLOCK | stop current path, require fix/replan |

Severity levels:

| Level | Default Behavior |
|---|---|
| P0 | always checked, can block immediately |
| P1 | checked often, enforced at merge/phase gates |
| P2 | advisory or merge-gate only |

---

## 5.3 Core Rules

These are stack-independent and always active:

| Rule | Purpose |
|---|---|
| `over_engineering` | prevent premature abstractions and speculative architecture |
| `dead_code` | remove unused, unreachable, or commented-out code |
| `scope_creep` | changed files must match task plan and worker profile |
| `tdd_violation` | no code merge without RED/GREEN/REFACTOR evidence or approved exception |
| `compound_required` | task/phase/cycle learning must be emitted |
| `backlog_singularity` | todo/work items must live in backlog |
| `discovered_not_logged` | new discoveries must become backlog items |
| `workflow_phase_skip` | Refinement -> Planning -> Execution -> Review cannot be skipped |
| `dod_required` | measurable DoD required before milestone creation/completion |
| `analysis_paralysis` | repeated read-only work without progress must stop or confess blocked state |
| `slopcheck` | hallucinated/unsafe dependencies blocked |
| `self_check_required` | completion claims need file/commit/verify evidence |
| `plan_placeholder` | TBD/TODO/hand-wavy plans blocked |
| `flexibility_traceability` | plan changes update links and append decision context |

`[NEVER_GATE]` rules are never auto-disabled even if hit-rate is low:

- security-sensitive checks
- architecture boundaries
- TDD
- DoD
- Backlog SSOT
- phase order
- slopcheck
- self-check
- plan placeholder

---

## 5.4 Project and Worker Rules

Sentinel does not hardcode a product type, stack, or repository layout.

It reads:

- `.harness/PROJECT_PROFILE.md`
- `.harness/CONVENTIONS.md`
- active `worker-profiles/*.md`

Examples:

| Profile Slot | Used For |
|---|---|
| `architecture_rules[]` | layer boundaries, dependency direction |
| `runtime_constraints[]` | forbidden APIs/dependencies for declared runtime |
| `format_commands[]` | formatting enforcement |
| `lint_commands[]` | lint enforcement |
| `typecheck_commands[]` | type/contract enforcement |
| `test_run_commands[]` | TDD and DoD verification |
| `security_scan_commands[]` | project-specific security tools |
| `coverage_commands[]` | optional coverage gates |

If a needed slot is missing, Sentinel emits an ASK or Phase 8 proposal instead of inventing a hidden default.

---

## 5.5 Two-Tier Cost Control

| Tier | Runs | Mechanism |
|---|---|---|
| Tier 1 cheap | every commit | profile-declared static tools + P0 pattern checks |
| Tier 2 expensive | suspicious findings, merge gate, DoD verify | semantic review with stronger model |

Tier 2 is for questions like:

- is this boundary violation meaningful?
- is this abstraction justified?
- does this satisfy real data flow?
- is this dependency safe enough to accept?

---

## 5.6 AUTO-FIX vs ASK

AUTO-FIX examples:

- formatter/linter mechanical fixes
- unused imports
- obvious dead code
- import order
- naming case when convention is explicit

ASK examples:

- ambiguous naming tradeoff
- architecture alternatives
- business/domain meaning
- dependency risk requiring user tolerance
- rule violation that may be intentional

ASKs are batched to Maestro Core.

---

## 5.7 4-Level Goal-Backward Verifier

Used at DoD verify, worker spec-review, and merge gates.

```text
Level 1: EXISTS
  files, imports, commands, commits exist

Level 2: SUBSTANTIVE
  no stubs, dummy returns, fake tests, empty implementations

Level 3: WIRED
  entrypoints, routes, exports, callers, config, docs are connected

Level 4: REAL DATA FLOW
  real data or real execution path proves the goal works
```

Default stance:

> Lack of passing evidence is failure.

Output shape:

```text
Level 1 EXISTS: PASS
Level 2 SUBSTANTIVE: PASS
Level 3 WIRED: FAIL - <dangling entrypoint>
Level 4 REAL DATA FLOW: SKIPPED
```

---

## 5.8 Adaptive Gating

Sentinel tracks hit-rate per rule.

Rules with no findings for a configured window may be demoted unless tagged `[NEVER_GATE]`. Demotion is itself visible and can be reviewed during Phase 8.

Repeated blocks can create:

- backlog item
- convention update proposal
- worker profile split/merge proposal
- Sentinel rule refinement proposal
