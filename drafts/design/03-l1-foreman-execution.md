# 03. L1 - Foreman Execution Layer

> Foreman turns an approved Task Spec into parallel work, worktree isolation, merge sequencing, and a Task Report. It is not user-facing.

---

## 3.1 Role

Foreman owns:

- Task Spec intake
- DAG construction
- worker profile allocation
- worktree creation and cleanup
- wave execution and merge order
- Sentinel invocation around commits/merges
- Task Report generation

Foreman does not:

- talk to the user
- rewrite the backlog directly
- decide product priorities
- inspect full worker context unless a structured escalation requires it

---

## 3.2 Input: Task Spec

Spec Writer drafts the Task Spec and Maestro Core approves it before Foreman receives it.

Required fields:

| Section | Required Content |
|---|---|
| Identity | `T-id`, backlog item, milestone, priority, mode |
| Goal | task description and acceptance criteria |
| Constraints | allowed paths, forbidden paths, project profile constraints, TDD flag |
| Dependencies | blocked-by / blocks relationships |
| Worker allocation hints | suggested worker profiles |
| Lessons to consult | relevant lesson IDs |
| Report back | expected Task Report schema and escalation rules |

Example worker allocation uses profile names, not fixed product roles:

```yaml
suggested_worker_allocation:
  - implementation
  - data
  - quality
  - documentation
```

---

## 3.3 DAG Builder

Foreman decomposes the Task Spec into stable unit IDs:

```text
T-101
  U-001 data boundary or setup
  U-002 implementation
  U-003 quality verification
  U-004 documentation
```

DAG rules:

| Rule | Reason |
|---|---|
| Stable U-ID per unit | trace plan -> commit -> report -> lesson |
| Same file/function cannot run in same wave | avoid merge conflict churn |
| Data/schema boundary changes precede dependent implementation | avoid false green |
| Design/spec work precedes dependent implementation | avoid building against missing spec |
| Security-sensitive review follows implementation | review real code, not guesses |
| Quality verification follows implementation but may write tests earlier if RED step requires it | keep TDD intact |

---

## 3.4 Worktree Execution

```text
project/
  .worktrees/
    implementation-T-101-U-001/
    data-T-101-U-002/
    quality-T-101-U-003/
```

Path format:

```text
.worktrees/<worker-profile>-<T-id>-<U-id>
```

Every worktree asserts before commit:

- `HEAD == expected_head`
- `cwd == expected_worktree_path`
- changed files are within planned touched files
- untracked files are explicitly listed in plan

Forbidden commands after worktree start:

- `git stash`
- `git reset --hard`
- `git clean -fd`
- protected branch ref rewrites
- force push to protected refs

`git stash` is prohibited because it silently crosses worktree boundaries.

---

## 3.5 Worker Handoff

Foreman sends each worker only:

- its unit ID
- the approved slice of the Task Spec
- relevant project profile/convention extracts
- required lessons
- expected step outputs

Foreman receives structured summaries, not raw chain-of-thought or entire working context.

Worker summary shape:

```json
{
  "u_id": "U-003",
  "worker_profile": "quality",
  "status": "done",
  "commit": "abc123",
  "tests_added": 4,
  "files_touched": 3,
  "sentinel": "pass",
  "compound_emitted": "L-2026-05-20-003"
}
```

---

## 3.6 Merge and Sentinel Gates

Foreman merges completed work by wave:

1. collect completed unit summaries
2. run Sentinel on each commit/worktree result
3. apply AUTO-FIX patches where allowed
4. block or escalate ASK items when required
5. merge wave into target branch
6. run merge-gate Sentinel and DoD-relevant checks
7. clean worktrees or preserve if policy requires

Patch dedup:

```text
patch_hash = sha256(diff)
if hash already applied:
  skip duplicate commit
else:
  commit and record hash
```

---

## 3.7 Output: Task Report

Foreman returns a raw Task Report to Report Editor and Maestro Core:

| Section | Content |
|---|---|
| Outcome | status, commits, worktrees, target branch, time/effort |
| AC verification | acceptance criteria and verifier evidence |
| New backlog items | auto-created discoveries |
| Lessons captured | task-level lesson IDs |
| Sentinel summary | PASS/FIX/BLOCK/ASK |
| TDD compliance | tests added/modified, exceptions |
| Compound summary | emitted/folded lessons |
| Suggested next actions | unblocked tasks, risks, asks |

Escalate to Maestro Core when:

- same Sentinel BLOCK repeats 3 times
- new P0/P1 backlog item appears
- effort exceeds estimate threshold
- irreversible action is needed
- Task Spec is materially wrong
