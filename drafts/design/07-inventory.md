# 07. 파일·디렉토리 인벤토리 + 사용자 명령

> 본 하네스가 실제로 디스크에 남기는 모든 산출물 + 사용자가 Maestro Core에 보내는 모든 명령.

---

## 7.1 디렉토리 인벤토리 (전체)

```
project/
├─ .git/
├─ .opencode/                          ← 하네스 정의 (git commit)
│   ├─ oh-my-openagent.jsonc           ← 카테고리/에이전트 오버라이드 + 모드 기본값
│   ├─ agents/                          ← 코어 에이전트 + seed worker profiles
│   │   ├─ maestro.md                   L0 (단일 사용자 창구 + 최종 결정권자)
│   │   ├─ board-clerk.md               L0-private PM 원장 CRUD
│   │   ├─ milestone-planner.md         L0-private Refinement/Planning proposal
│   │   ├─ spec-writer.md               L0-private Task Spec 작성
│   │   ├─ report-editor.md             L0-private 사용자용 report summary
│   │   ├─ context-librarian.md         L0-private board/lessons 검색 요약
│   │   ├─ foreman.md                   L1
│   │   ├─ sentinel.md                  L1
│   │   ├─ agent-architect.md           L2 (메타, 진화 사이클)
│   │   └─ worker-profiles/             프로젝트별 활성 worker profile
│   │       ├─ planning.md              seed
│   │       ├─ design.md                seed
│   │       ├─ implementation.md        seed
│   │       ├─ data.md                  seed
│   │       ├─ security.md              seed
│   │       ├─ quality.md               seed
│   │       ├─ ops.md                   seed
│   │       └─ documentation.md         seed
│   ├─ skills/
│   │   ├─ sentinel-rules/SKILL.md      Sentinel 룰셋 + 정적 도구 호출 레시피
│   │   ├─ pm-board/SKILL.md            ★ Board Clerk용 6엔티티 CRUD + 4-step 상태 전이
│   │   ├─ pm-delegation/SKILL.md       Maestro Core ↔ private PM sub-agent schema
│   │   ├─ worktree-orchestrator/SKILL.md
│   │   ├─ dag-builder/SKILL.md
│   │   ├─ project-profile/SKILL.md     프로젝트 스택/구조/워커 프로필 기록
│   │   ├─ convention-registry/SKILL.md 코드 컨벤션/관리 규칙 registry
│   │   ├─ run-mode/SKILL.md            모드 동작
│   │   ├─ tdd-discipline/SKILL.md      RED→GREEN→REFACTOR + 도구별 명령
│   │   ├─ compound-cycle/SKILL.md      3-tier compound + fingerprint-merge
│   │   ├─ harness-evolution/SKILL.md   Phase 8 진화 사이클
│   │   ├─ lesson-format/SKILL.md       lesson markdown 스키마
│   │   └─ proposal-format/SKILL.md     proposal 스키마
│   ├─ command/
│   │   ├─ start.md                     Phase 0 진입 + 모드 선택
│   │   ├─ board.md                     전체 보드 보기
│   │   ├─ vision.md / roadmap.md / milestone.md / dod.md
│   │   ├─ backlog.md / issue.md (alias) / task.md
│   │   ├─ refinement.md / planning.md / execution.md / review.md
│   │   ├─ next.md                      다음 해야 할 일 추천
│   │   ├─ report.md                    Task/milestone 보고서 raw 보기
│   │   ├─ mode.md                      모드 전환
│   │   ├─ pause.md / resume.md / stop.md
│   │   ├─ revise.md                    Vision 또는 goal 수정
│   │   ├─ sentinel-check.md            수동 감사
│   │   ├─ cleanup-worktrees.md
│   │   ├─ evolve.md                    Phase 8 수동 트리거
│   │   ├─ lessons.md / proposals.md
│   │   ├─ tdd-exception.md             TDD 예외 등록
│   │   └─ compound-now.md              강제 Tier 2 compound 실행
│   └─ hooks/
│       ├─ pre-commit-tdd-check.sh
│       ├─ post-commit-compound-emit.sh
│       ├─ phase-boundary-compound.sh
│       ├─ phase-boundary-state-update.sh
│       ├─ worktree-guard.sh            git stash/reset-hard/clean 차단
│       └─ pre-tool-backlog-singularity.sh
│
├─ .harness/                          ← 런타임 상태 (일부 git commit)
│   ├─ goal.md
│   ├─ ETHOS.md                      ← P1~P16 본문화, 모든 워커 preamble
│   ├─ STATE.md                      ← O_EXCL lock
│   ├─ PROJECT_PROFILE.md            ← 프로젝트 타입/스택/레이아웃/worker profile
│   ├─ CONVENTIONS.md                ← 코드 컨벤션 + 관리 규칙 registry
│   ├─ PROJECT.md / REQUIREMENTS.md / ROADMAP.md / CONTEXT.md
│   ├─ prd.md / design-spec.md
│   ├─ progress.jsonl                ← append-only
│   ├─ sentinel-log.jsonl            ← append-only
│   ├─ board/                        ← PM 엔티티 (git commit)
│   │   ├─ vision.md
│   │   ├─ roadmap.md
│   │   ├─ milestones/_index.md, M-*.md
│   │   ├─ backlog/_index.md, _types.md, B-*.md
│   │   └─ tasks/_index.md, T-*.md
│   ├─ phases/<N>/PLAN.md, SUMMARY.md, VERIFY.md
│   ├─ lessons/                      ← 3-tier compound (git commit)
│   │   ├─ _pending/
│   │   └─ architecture/ bugs/ perf/ ops/ process/ prompts/
│   └─ proposals/                    ← Phase 8 진화 제안
│       ├─ _pending/ accepted/ rejected/ superseded/
│
├─ .worktrees/                        ← 병렬 작업장 (gitignore)
│   └─ <role>-<T-id>-<U-id>/
│
└─ <프로젝트 본체>
    └─ 구조는 `.harness/PROJECT_PROFILE.md`가 선언한 layout을 따른다
```

---

## 7.2 프로젝트 프로필

하네스 코어는 특정 스택, 레포 구조, 도메인을 강제하지 않는다. `/start`의 Phase 0에서 Maestro Core가 사용자의 목표와 기존 repo를 읽고 `.harness/PROJECT_PROFILE.md`를 만든다.

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
```

프로필은 시작점일 뿐이다. 사용 중 반복되는 실패, Sentinel finding, lessons가 쌓이면 Agent-Architect가 profile 변경 proposal을 낸다.

---

## 7.3 진화형 컨벤션과 관리 규칙

`convention-registry`는 다음 세 층을 구분한다:

| 층 | 예 | 변경 방식 |
|---|---|---|
| **Core rules** | TDD, Backlog SSOT, DoD, 4-step cycle, Maestro 단일 surface | 변경 불가 또는 사용자 명시 승인 |
| **Project rules** | repo layout, dependency direction, naming, formatter, test pyramid | Phase 0에서 시작, Phase 8 proposal로 진화 |
| **Worker rules** | 특정 worker가 보는 파일, verify command, review lens | worker profile별로 진화 |

Sentinel은 Core rules를 항상 강제하고, Project/Worker rules는 `.harness/PROJECT_PROFILE.md`와 `.harness/CONVENTIONS.md`에서 읽어 적용한다. 특정 스택 규칙은 하네스에 하드코딩하지 않는다.

---

## 7.4 사용자 명령 인터페이스

> 사용자는 Maestro Core와만 대화. 모든 명령은 Maestro Core가 해석하고, PM 원장 변경은 private PM sub-agent가 구조화 요청으로 실행한다.

### 7.4.1 진입 & 기본

```bash
/start "<자연어 목표>"                    # Phase 0 진입
/start --mode=gated "<목표>"
/start --mode=plan-only "<목표>"
/start --mode=dry-run "<목표>"

/mode auto|gated|plan-only|dry-run       # 모드 전환

/pause / /resume / /stop                  # 사이클 제어
/status                                   # 현재 상태 요약
```

### 7.4.2 PM 보드 (6 entity CRUD)

```bash
/board
  → 전체 PM 보드 요약: vision → 활성 roadmap → 현재 milestone (phase + DoD%)
    → backlog top 10 (priority) → 활성 task

# Vision
/vision                                   # V-001 본문
/vision revise "<변경>"                    # 비전 수정 (드물게, CONTEXT.md 로그 자동)

# Roadmap
/roadmap                                  # R-* 본문
/roadmap add "<title>"
/roadmap edit R-001 "<변경>"
/roadmap archive R-001

# Milestone (+ DoD)
/milestone                                # 리스트 + 각 milestone의 DoD 충족률
/milestone show M-003
/milestone add "<title>" [--roadmap=R-001]   # DoD 인터뷰 게이트 자동 발동
/milestone edit M-003 "<변경>"
/milestone done M-003                     # 모든 DoD passed 시만 통과

/dod show M-003
/dod add M-003 "<criterion>" --verify "<cmd>"
/dod edit M-003 DoD-1 "<변경>"
/dod verify M-003                         # 모든 DoD verify_cmd 실행

# Backlog (단일 SSOT)
/backlog                                  # 전체 priority 정렬
/backlog show B-042
/backlog add <type> "<title>" [--priority=P1]
                                          # type: feature|bug|tech_debt|research|spike
/backlog edit B-042 "<변경>"
/backlog triage B-042 --priority P1       # idea → triaged
/backlog select B-042 --milestone M-003   # triaged → selected
/backlog deselect B-042                   # 다시 backlog로
/backlog drop B-042 "<사유>"               # dropped

# Issue (alias for bug)
/issue add "<title>" [--severity=high]    # = /backlog add bug
/issue show B-099

# Task
/task                                     # 활성 task 리스트
/task show T-101
/task add "<title>" --backlog B-042       # 대부분 Planning에서 자동
/task edit T-101 "<변경>"
/task block T-101 --by T-099
/task unblock T-101 --from T-099
/task status T-101 cancelled|blocked
```

### 7.4.3 4-step 사이클 제어

```bash
/refinement                               # Refinement 단계 진입 (백로그 정리)
/planning                                 # Planning 단계 (milestone + DoD 게이트)
/execution                                # Execution 명시 진입 (auto/gated 자동)
/review                                   # Review 단계 (DoD verify + Tier 3 compound)

/next                                     # 현재 phase에서 다음 해야 할 일 추천
/next dispatch                            # 위 결정 후 Foreman dispatch
```

### 7.4.4 보고서 & 디버그

```bash
/report T-101                             # Foreman의 raw Task Report
/report milestone M-003                   # milestone 요약

/sentinel-check [경로]                    # 수동 Sentinel 감사
/cleanup-worktrees                        # stale worktree 정리
```

### 7.4.5 Compound & Evolution

```bash
/lessons [--category=bugs|arch|...] [--cycle=N] [--high-leverage]
/lessons show <L-id>

/tdd-exception <task-id> "<사유>"          # TDD 예외 등록 (Maestro Core 승인 필요)

/evolve                                   # Phase 8 수동 트리거
/proposals                                # _pending 목록 + 요약
/proposals show <P-id>
/proposals accept|reject|defer <P-id> [메모]

/compound-now                             # 강제 Tier 2 compound 실행 (디버그)
```

### 7.4.6 골 수정

```bash
/revise "<수정 사항>"                       # Vision/Roadmap/Milestone 영향 큰 변경
```

→ Milestone Planner가 영향 범위 분석 → Board Clerk이 영향 backlog item 자동 재정렬 초안 + 변경 이력 CONTEXT.md append 초안 작성 → Maestro Core 사용자 컨펌.

---

## 7.5 사용자가 보지 못하는 명령들

다음은 시스템 내부 명령. 사용자가 직접 호출할 수 없다 (P12 강제):

- Foreman의 worktree spawn / merge
- Sentinel 룰 실행
- private PM sub-agent의 board patch / planning proposal / report summary 생성
- worker profile의 `task()` 호출
- Agent-Architect의 proposal 작성
- compound emit

이 명령들의 결과는 항상 Maestro Core를 거쳐 가공된 메시지로만 surface.
