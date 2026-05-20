# 08. Implementation Roadmap and Open Decisions

> This document maps the architecture to oh-my-openagent/OpenCode implementation work and tracks remaining decisions.

---

## 8.1 OpenCode Mapping

| Harness Concept | Mechanism | Implementation |
|---|---|---|
| Maestro Core | Prometheus extension + delegation skill | `.opencode/agents/maestro.md` |
| Private PM sub-agents | custom agents + PM skills | `board-clerk.md`, `milestone-planner.md`, `spec-writer.md`, `report-editor.md`, `context-librarian.md` |
| Foreman | Atlas-like execution agent | `.opencode/agents/foreman.md` |
| Sentinel | agent + hooks + rule skill | `.opencode/agents/sentinel.md`, hooks, `sentinel-rules` |
| Worker profiles | project-adaptive agent profiles | `.opencode/agents/worker-profiles/*.md` |
| Project profile | skill + runtime file | `project-profile/SKILL.md`, `.harness/PROJECT_PROFILE.md` |
| Convention registry | skill + runtime file | `convention-registry/SKILL.md`, `.harness/CONVENTIONS.md` |
| Worktree execution | git worktree + guarded hooks | `worktree-orchestrator`, `worktree-guard.sh` |
| DAG builder | skill/Foreman prompt | `dag-builder/SKILL.md` |
| TDD discipline | worker skill | `tdd-discipline/SKILL.md` |
| Compound | skill + hooks | `compound-cycle/SKILL.md`, phase hooks |
| Phase 8 evolution | Agent Architect skill | `harness-evolution/SKILL.md` |
| STATE lock | shell utility | `state-lock.sh` |

---

## 8.2 Implementation Order

1. Write `.harness/ETHOS.md` v0 from the fixed core.
2. Write Maestro Core prompt v0 as a thin orchestrator.
3. Write private PM sub-agent prompts.
4. Implement `pm-delegation` schemas.
5. Implement `pm-board` entity CRUD and index integrity.
6. Define Task Spec, Task Report, Report Summary schemas.
7. Implement `project-profile` and `convention-registry`.
8. Implement seed worker profiles.
9. Implement `tdd-discipline` against profile-declared test commands.
10. Implement Sentinel core rules and profile slot reading.
11. Implement Foreman DAG builder and worktree guard.
12. Implement Compound lesson format and phase hooks.
13. Implement Phase 8 proposal format and Agent Architect flow.
14. Implement user command files.
15. Run a small general coding dry-run.
16. Fold dry-run findings into v0.6 decisions.

---

## 8.3 Dry-Run Acceptance

Use a small general coding project, not a product-type-specific app.

Dry-run must prove:

- user only talks to Maestro Core
- Phase 0 creates `PROJECT_PROFILE.md` and `CONVENTIONS.md`
- board entities are created and linked
- DoD is measurable
- Refinement -> Planning -> Execution -> Review is enforced
- Foreman produces DAG and Task Report without being user-facing
- worker profile allocation uses project profile
- Sentinel catches Backlog SSOT and placeholder violations
- Task Spec/Report/Report Summary schemas work
- Phase 8 creates at least one proposal from observed data

---

## 8.4 v0.6 Locked Defaults

### Maestro Core Split

| Decision | Default |
|---|---|
| user-facing channel | Maestro Core only |
| PM writes | Board Clerk only, after approval |
| planning | Milestone Planner proposal + Maestro Core approval |
| task spec | Spec Writer draft + Maestro Core approval |
| report surface | Report Editor summary + Maestro Core final wording |
| context | indexes + active summaries; body reads on demand |

### General Coding Harness

| Decision | Default |
|---|---|
| fixed core | TDD, Maestro Core, Backlog SSOT, DoD, 4-step cycle, Sentinel, Compound, Phase 8 |
| adaptive layer | worker profiles, conventions, repo layout, commands, release rules |
| profile files | `.harness/PROJECT_PROFILE.md`, `.harness/CONVENTIONS.md` |
| Sentinel project rules | read from profile/registry, not hardcoded stack assumptions |
| evolution | repeated findings and user preferences become proposals |

---

## 8.5 Open Decisions

### PM Model

- board git policy: commit every Board Clerk write or batch per phase?
- entity graph rendering: list, ASCII tree, Mermaid, or mode-dependent?
- stale entity policy: when to archive dormant P3/parked work?
- multiple active milestones: only one by default, but how does P0 hotfix interrupt?

### DoD

- verify command shape: inline shell, script path, or schema with command/env/cwd?
- recommended DoD count per milestone
- when Level 4 real data flow is mandatory vs impossible

### Worker Profiles

- start with all seed profiles or minimal subset per project?
- threshold for splitting a profile into two specialists
- threshold for merging low-value profiles
- how to version worker profile changes

### Convention Evolution

- when user preference becomes project convention
- how many repeated findings before rule proposal
- how to deprecate noisy project rules

### Operations

- default `max_concurrent_worktrees`
- worktree cleanup policy
- budget gate shape: tokens, dollars, time, or combined
- deploy/release gates per project type

### Sentinel

- multi-BLOCK resolution ordering
- rule confidence/fingerprint algorithm
- profile slot auto-detection vs explicit user confirmation
