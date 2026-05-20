# 04. L2 - Worker Profiles and Internal Step Flows

> Worker profiles are project-adaptive execution roles. The harness ships seed profiles, but the active set evolves through project evidence.

---

## 4.1 Worker Profile Model

Worker profiles are not permanent product roles. They are files under:

```text
.opencode/agents/worker-profiles/*.md
```

Each profile defines:

| Field | Meaning |
|---|---|
| `id` | stable profile name |
| `use_when` | when Foreman should allocate it |
| `allowed_paths` | file/path boundaries, if any |
| `step_flow` | code 7-step, prose 5-step, or custom evolved flow |
| `verify_commands` | profile-specific checks |
| `review_lens` | what this profile is best at catching |
| `model_category` | preferred category and fallback |
| `evolution_notes` | changes accepted/rejected over time |

---

## 4.2 Seed Profiles

| Profile | Use When | Step Flow | TDD |
|---|---|---|---|
| Planning | requirements, scope, AC, priority | prose/spec 5-step | no |
| Design | UX, system design, interface design, architecture sketch | prose/spec 5-step | no |
| Implementation | code changes in the main source tree | code 7-step | yes |
| Data | schema, migrations, pipelines, storage model | code 7-step + dry-run/rollback plug | yes |
| Security | auth, authorization, input validation, secrets, threat model | code 7-step + audit plug | yes |
| Quality | tests, regression suite, coverage, E2E/integration verification | code 7-step | yes |
| Ops | CI/CD, release, runtime, observability, operational automation | code 7-step + smoke/rollback plug | yes |
| Documentation | README, ADR, changelog, user docs, lessons | prose/spec 5-step | no |

Examples of project evolution:

- web project splits Implementation into `ui-implementation` and `api-implementation`
- library project adds `public-api` and `compatibility`
- performance-heavy project adds `benchmark`
- tiny CLI project merges Data/Ops into Implementation

---

## 4.3 Code Worker 7-Step

Applies to code-affecting profiles.

```text
Worker
  step:plan
  step:red
  step:green
  step:refactor
  step:spec-review
  step:quality-review
  step:commit
  step:compound
```

Gate details:

| Step | Required Evidence |
|---|---|
| plan | files, AC, constraints, no placeholders |
| red | failing test or approved TDD exception |
| green | minimal implementation and passing new test |
| refactor | all tests still pass |
| spec-review | AC/spec satisfied, separate task context |
| quality-review | simplicity, duplication, style, boundary check |
| commit | atomic commit plus Self-Check block |
| compound | lesson candidate or explicit "no lesson" with reason |

Self-Check block:

```markdown
## Self-Check
- Files claimed: <list> -> EXISTS verified
- Commits claimed: <hashes> -> git log verified
- Tests added/modified: <count>
- Verify commands run: <cmd> -> exit code
- Result: PASSED|FAILED
```

TDD exception requires:

- `tdd_exception: "<reason>"` in plan
- Maestro Core approval
- follow-up regression or equivalent evidence

Allowed categories include spike/POC, hot-fix with follow-up regression, config-only change, and data/schema migration where dry-run replaces RED.

---

## 4.4 Prose/Spec Worker 5-Step

Applies to Planning, Design, Documentation, and similar profiles.

```text
Worker
  step:research
  step:draft
  step:peer-review
  step:revise
  step:compound
```

Rules:

- `step:draft` cannot contain TBD/TODO/placeholder wording
- acceptance criteria must be concrete when a plan/spec is produced
- peer review is a separate task context
- lessons are emitted when the prose/spec changed future execution behavior

---

## 4.5 Meta Worker: Agent Architect

Agent Architect is a core meta-worker. It proposes changes; it does not silently mutate the harness.

```text
Agent Architect
  step:diagnose
  step:baseline
  step:design
  step:propose
  step:peer-review
  step:compound
```

Inputs:

- `sentinel-log.jsonl`
- lessons
- accepted/rejected proposals
- repeated worker failures
- user preferences

Outputs:

- proposal to add/split/merge/deprecate worker profile
- proposal to add/update Sentinel rule
- proposal to update project conventions
- proposal to change skill/hook/prompt

Accepted changes are tagged with `evolved: true` and monitored for effect.

---

## 4.6 Context Isolation

Worker steps run in fresh task contexts. Each step receives only declared inputs from the previous step.

The parent layer receives summary JSON, not full context:

```json
{
  "u_id": "U-007",
  "worker_profile": "implementation",
  "status": "done",
  "commit": "abc123",
  "tests_added": 4,
  "files_touched": 3,
  "sentinel": "pass",
  "compound_emitted": "L-2026-05-20-003"
}
```

---

## 4.7 Model Allocation

Default model choices are tunable and project-adaptive:

| Profile | Default Category | Reason |
|---|---|---|
| Maestro Core | high-reasoning conversational | user interaction and final judgment |
| Board Clerk | writing/structured | schema-safe PM writes |
| Milestone Planner | writing/reasoning | priority and dependency reasoning |
| Spec Writer | writing/structured | precise task specs |
| Report Editor | quick/writing | concise summaries |
| Context Librarian | quick/search | retrieval summaries |
| Foreman | medium reasoning | DAG and merge coordination |
| Sentinel 1st pass | quick | cheap frequent checks |
| Sentinel 2nd pass | high reasoning | semantic judgment |
| Agent Architect | high reasoning | meta-design |
| Worker profiles | from `PROJECT_PROFILE` | tuned by hit-rate and quality |
