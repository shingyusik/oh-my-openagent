# Survey: gsd-build/get-shit-done

> Source: https://github.com/gsd-build/get-shit-done
> Type: Runtime-agnostic agent harness ("harness on top of harnesses")
> 분석 시점: 2026-05-20

---

## 1. 개요

GSD (Get Shit Done)는 **컨텍스트 rot 방지**를 최상위 목표로 하는 메타-프롬프팅 하네스. **15개의 다른 코딩 런타임**(Claude Code, OpenCode, Codex, Cursor 등)에 동시 install 되는 단일 source.

핵심:
- **Thin Orchestrator** (15% 컨텍스트만 유지) + **File-State Memory** (.planning/)
- **Wave Execution Model**로 병렬화
- **Decision Coverage Gates**로 결정 미커버 차단
- **Adversarial Goal-Backward Verification** (목표가 충족되지 않았다는 가정으로 시작)
- **Runtime Abstraction Layer**로 매핑

구성:
- **agent 33개**
- **command 67개**
- **hook 11개 + lib**
- **SDK** (`gsd-sdk query <namespace>` JSON 반환)

---

## 2. 철학 / 핵심 원칙

| 원칙 | 의미 |
|---|---|
| **Thin orchestrator** | 오케스트레이터는 컨텍스트의 15%만 점유. 나머지는 .planning/ 파일에 저장. 어느 시점에 컨텍스트가 reset되어도 회복 가능. |
| **File-state memory** | 모든 영구 상태는 사람이 읽을 수 있는 markdown으로 .planning/에 저장. 사람이 직접 편집 가능. |
| **Wave execution** | DAG의 의존성 wave로 그룹화. 같은 wave 내는 병렬, wave 간은 직렬. 오케스트레이터는 wave 끝에서만 pre-commit hook 1회 실행 (per-task 호출 X). |
| **Adversarial verification** | 검증자는 "목표가 안 됐다"를 가정. 코드베이스 증거로 반증되어야 통과. |
| **Decision coverage** | 모든 결정은 REQ-ID 태그. 실행되지 않은 결정 = blocking error. |
| **Runtime abstraction** | 단일 source → 15 런타임. install 시 tool name / hook signature / agent format normalize. |

---

## 3. 아키텍처 / 구조

```
gsd-build/get-shit-done/
├── agents/                    (33개)
│   ├── researchers/           (5: ai/domain/ui/project/phase)
│   ├── synthesizers/
│   ├── planners/              (4: planner/roadmapper/plan-checker/framework-selector)
│   ├── execution/             (executor + code-fixer)
│   ├── verification/          (verifier + integration/doc/ui-checker)
│   ├── audit/                 (security/ui/eval/nyquist/assumptions)
│   ├── debug/                 (debugger + session-manager)
│   ├── mapping/               (codebase-mapper + pattern-mapper)
│   ├── doc/                   (writer + classifier + synthesizer + verifier)
│   ├── eval/
│   ├── profile/               (user-profiler)
│   ├── intel/                 (updater)
│   └── advisor/               (researcher)
├── commands/                  (67개)
│   ├── core/                  (new-project/discuss-phase/plan-phase/execute-phase/verify-work/ship)
│   ├── progression/           (progress/autonomous/phase/resume-work/pause-work)
│   ├── audit/                 (audit-fix/audit-milestone/audit-uat/forensics)
│   ├── quick/                 (fast/quick/spike/sketch)
│   ├── inspection/            (graphify/health/stats/surface/explore)
│   ├── specialty-phases/      (ui-phase/secure-phase/validate-phase/mvp-phase/ai-integration-phase/ultraplan-phase/spec-phase)
│   └── ns-*                   (namespace cluster)
├── hooks/                     (11 + lib)
│   ├── gsd-context-monitor.js
│   ├── gsd-workflow-guard.js
│   ├── gsd-prompt-guard.js
│   ├── gsd-read-injection-scanner.js
│   ├── gsd-validate-commit.sh
│   ├── gsd-phase-boundary.sh
│   ├── gsd-session-state.sh
│   ├── gsd-statusline.js
│   ├── gsd-check-update.js
│   ├── gsd-check-update-worker.js
│   └── gsd-update-banner.js
├── sdk/
│   └── gsd-sdk                (CLI returning structured JSON)
├── runtimes/                  (15 런타임 adapter)
└── .planning/                 (런타임에 생성됨)
    ├── PROJECT.md
    ├── REQUIREMENTS.md
    ├── ROADMAP.md
    ├── STATE.md               (O_EXCL lock 사용)
    ├── CONTEXT.md
    └── phases/
        └── <N>/
            ├── PLAN.md
            ├── SUMMARY.md
            └── VERIFY.md
```

---

## 4. 컴포넌트 카탈로그

### 4.1 Agents (33개, 카테고리별)

#### Researchers (5)
| Agent | 책무 |
|---|---|
| **gsd-ai-researcher** | AI/ML 관련 자료조사. |
| **gsd-domain-researcher** | 비즈니스 도메인 분석. |
| **gsd-ui-researcher** | UX/UI 패턴 리서치. |
| **gsd-project-researcher** | 기존 코드베이스 매핑. |
| **gsd-phase-researcher** | 현재 phase의 사전조사 전담. |

#### Synthesizers (2)
| Agent | 책무 |
|---|---|
| **gsd-research-synthesizer** | 여러 researcher 결과를 1개 문서로. |
| **gsd-doc-synthesizer** | 여러 doc source를 통합. |

#### Planners (4)
| Agent | 책무 |
|---|---|
| **gsd-planner** | phase PLAN.md 작성. |
| **gsd-roadmapper** | ROADMAP.md 작성/갱신. |
| **gsd-plan-checker** | plan에 placeholder/누락/모호 표현 검출. |
| **gsd-framework-selector** | 라이브러리/프레임워크 선택 가이드. |

#### Execution (2)
| Agent | 책무 |
|---|---|
| **gsd-executor** | PLAN.md를 실행. 모든 위험한 git 명령 차단. analysis-paralysis guard 내장. |
| **gsd-code-fixer** | verify 실패 시 fix만 담당. |

#### Verification (4)
| Agent | 책무 |
|---|---|
| **gsd-verifier** | **goal-backward 4-level**: exists → substantive → wired → real data flow. |
| **gsd-integration-checker** | 모듈 통합 검증. |
| **gsd-doc-verifier** | 문서가 실제 코드와 일치하는지. |
| **gsd-ui-checker** | UI rendering 검증. |

#### Audit (5)
| Agent | 책무 |
|---|---|
| **gsd-security-auditor** | OWASP + 시크릿/eval/raw SQL 등. |
| **gsd-ui-auditor** | 디자인 일관성, a11y. |
| **gsd-eval-auditor** | AI eval pipeline 감사. |
| **gsd-nyquist-auditor** | data sampling/signal processing 감사 (niche). |
| **gsd-assumptions-analyzer** | 코드 내 implicit assumption 추출 → CONTEXT.md 등재. |

#### Debug (2)
| Agent | 책무 |
|---|---|
| **gsd-debugger** | 디버깅 세션 driver. |
| **gsd-debug-session-manager** | debug 세션 상태 유지. |

#### Mapping (2)
| Agent | 책무 |
|---|---|
| **gsd-codebase-mapper** | 디렉토리 트리 + entrypoint + 의존 그래프. last_mapped_commit 추적. |
| **gsd-pattern-mapper** | 코드 패턴 추출 (factory, repository 등). |

#### Doc (2)
| Agent | 책무 |
|---|---|
| **gsd-doc-writer** | README/CHANGELOG/ADR 작성. |
| **gsd-doc-classifier** | 기존 doc을 카테고리 분류. |

#### 기타 (5)
- **gsd-eval-planner** — eval test plan 설계.
- **gsd-user-profiler** — 사용자 프로필 추출 (페르소나).
- **gsd-intel-updater** — 외부 변화 (라이브러리 업데이트 등) 모니터링.
- **gsd-advisor-researcher** — 결정 시 advisor 입장 리서치.

### 4.2 Commands (67개, 카테고리별)

#### Core Loop (6) — 가장 중요
- **new-project** — 프로젝트 부트스트랩 (`.planning/` 생성).
- **discuss-phase** — phase 시작 전 사용자와 논의.
- **plan-phase** — PLAN.md 생성.
- **execute-phase** — plan 실행.
- **verify-work** — goal-backward verify.
- **ship** — release.

#### Progression (5)
- **progress** — 현재 진행상태 출력.
- **autonomous** — ROADMAP을 1회 직진. HITL 최소화.
- **phase** — 다음 phase로 이동.
- **resume-work** — 중단된 phase 재개.
- **pause-work** — 현재 phase 일시중지.

#### Audit (4)
- **audit-fix** — Sentinel 감사 + 자동 fix.
- **audit-milestone** — 마일스톤 단위 감사.
- **audit-uat** — UAT 시뮬레이션.
- **forensics** — git 이력 분석.

#### Quick Paths (4)
- **fast** — 빠른 실행 (verify skip 가능).
- **quick** — 1 unit 작업.
- **spike** — POC 모드.
- **sketch** — design 단계만.

#### Inspection (5)
- **graphify** — 의존 그래프 시각화.
- **health** — 코드베이스 health 체크.
- **stats** — 통계.
- **surface** — 노출된 API.
- **explore** — 자유 탐색.

#### Specialty Phases (7)
- **ui-phase**, **secure-phase**, **validate-phase**, **mvp-phase**, **ai-integration-phase**, **ultraplan-phase**, **spec-phase**

#### Namespace Cluster (`ns-*`)
- `ns-init`, `ns-link`, `ns-publish` 등. workspace/namespace 관리 명령군.

### 4.3 Hooks (11)

| Hook | 종류 | 동작 |
|---|---|---|
| **gsd-context-monitor.js** | PostToolUse | 컨텍스트 사용량 모니터. 35%에서 "wrap-up" 경고, 25% 남으면 "stop" 경고. debounced (5-tool 단위). 임계 초과는 즉시 escalate (debounce 무시). breadcrumb로 .planning/STATE.md에 표시. |
| **gsd-workflow-guard.js** | PreToolUse | .planning/ 외부에 대한 write/edit이 sub-agent 안에서가 아니면 advisory. |
| **gsd-prompt-guard.js** | PreToolUse | .planning/ 파일에 쓸 때 13개 injection regex + invisible Unicode 스캔. log-only. |
| **gsd-read-injection-scanner.js** | PostToolUse | Read 출력이 untrusted source일 때 inject 시도 스캔. |
| **gsd-validate-commit.sh** | PreToolUse | conventional commit 메시지 강제. opt-in (`hooks.community: true`). |
| **gsd-phase-boundary.sh** | PreToolUse | phase 경계에서 SUMMARY.md/VERIFY.md 작성 강제. |
| **gsd-session-state.sh** | SessionStart | STATE.md를 읽어 현재 phase 정보 주입. |
| **gsd-statusline.js** | UI hook | 상태바 표시. |
| **gsd-check-update*.js** | startup | gsd 자체 업데이트 확인. |
| **gsd-update-banner.js** | UI hook | 업데이트 가능 시 배너. |

### 4.4 SDK

`gsd-sdk query <namespace>` 형태의 CLI. JSON 반환.

namespace 예:
- `init.<workflow>` — 워크플로 부트스트랩 설정
- `resolve-model` — 모델 선택 알고리즘
- `state.current` — 현재 state
- `state.set` — state 변경
- `roadmap.next` — 다음 phase
- `requirements.list`
- `commit-to-subrepo` — 모노레포 sub-repo 분리 커밋
- `task.is-behavior-adding` — task가 새 행동 추가인지 판정

agent와 command 모두 이 SDK를 호출. agent prompt 안에서 SDK 호출을 강제하여 결정적 동작 확보.

---

## 5. 핵심 메커니즘 상세

### 5.1 File-State Memory: `.planning/`

모든 영구 상태:

```
.planning/
├── PROJECT.md           — 프로젝트 메타 (이름, 목적, stakeholder)
├── REQUIREMENTS.md      — REQ-ID로 분해된 요구사항
├── ROADMAP.md           — phase 순서
├── STATE.md             — 현재 phase, wave, 진행 노드 (O_EXCL lock)
├── CONTEXT.md           — 모든 결정 + REQ-ID 태그
└── phases/<N>/
    ├── PLAN.md          — 이 phase의 unit 분해
    ├── SUMMARY.md       — 완료된 작업 (각 unit의 self-check 결과 포함)
    └── VERIFY.md        — verifier 출력
```

- 사람이 직접 편집 가능 (markdown)
- 컨텍스트 reset되어도 회복 (오케스트레이터가 STATE.md 읽고 재시작)
- `gated` / `pause-work` / `resume-work`가 자연스럽게 동작

### 5.2 STATE.md Atomic Locking

```bash
exec {LOCK_FD}>.planning/.state.lock
flock -n $LOCK_FD || (sleep 0.5 && retry)
# critical section: read/modify/write STATE.md
exec {LOCK_FD}>&-
```

- `O_EXCL` 시맨틱
- 10초 timeout
- stale lock 감지 (lock holder PID 사망 확인)
- 병렬 worktree에서 같은 STATE.md를 안전하게 갱신

### 5.3 Wave Execution Model

```
plan.units → topological sort → wave grouping

wave 1: [U1, U2]     (depends on: nothing)
wave 2: [U3, U4, U5] (depends on: U1)
wave 3: [U6]         (depends on: U3, U4)

for wave in waves:
  parallel-dispatch(wave.units)  → 각 unit이 sub-agent
  wait_all()
  orchestrator.run_pre_commit_hooks_once()   # wave 끝에 1회
  merge(wave)
```

**parallel commit safety**:
- sub-agent들은 `--no-verify`로 빠르게 commit
- pre-commit hook은 wave 끝에 orchestrator가 1회 일괄 실행
- 결과: hook 실행시간 N배 절감 + lock contention 없음

### 5.4 Goal-Backward 4-Level Verifier

`gsd-verifier`의 단계:

```
Level 1: EXISTS
  - 모든 변경된 entrypoint 파일이 디스크에 존재?
  - import 경로가 resolve됨?

Level 2: SUBSTANTIVE
  - 함수 body가 stub("throw notImplemented") 아님?
  - 변수가 hardcoded null/empty 아님?

Level 3: WIRED
  - 새 컴포넌트가 부모에 mount됨?
  - 새 route가 router에 등록됨?
  - 새 API endpoint가 export됨?

Level 4: REAL DATA FLOW
  - 실제 데이터 (DB/API) 흘려보면 결과가 나옴?
  - mock/fixture가 아닌 실데이터로 1회 실행 확인

→ Level 통과해야 다음 Level 진입.
→ adversarial: "통과 안 됐다"가 기본 가정.
```

기존 대부분 하네스가 Level 1-2만 검사하고 끝남. Wired/Data flow가 GSD의 차별점.

### 5.5 Analysis-Paralysis Guard

`gsd-executor` 내부 룰:

```
counter = 0
for each tool call:
  if tool in {Read, Grep, Glob, lsp_*}:
    counter += 1
  else:
    counter = 0

  if counter >= 5:
    halt
    require: "다음 1문장으로 왜 아무것도 쓰지 않았는지 설명. 직후에 Write/Edit/Bash 또는 'BLOCKED' 보고."
```

이유: AI가 무한히 탐색만 하다 작업을 안 하는 패턴 차단.

### 5.6 Worktree Prohibition Layer (gsd-executor)

`gsd-executor`가 시작될 때 self-imposing rules:

```
포워딩 금지 명령:
- git stash (worktree 간 공유되어 다른 worktree의 WIP를 끌어옴)
- git reset --hard (post-startup; 시작 시점 이전은 OK)
- git clean -fd
- git update-ref refs/heads/main
- git push --force-with-lease to protected refs
- git push -f

모든 commit 직전:
- assert: HEAD == expected_head
- assert: cwd == expected_worktree_path
- assert: git status에 untracked가 plan에 명시됨
```

이유: 병렬 worktree에서 git stash leak은 silent하고 치명적. assertion으로 가시화.

### 5.7 Slopcheck (Package Legitimacy)

새 패키지 install 전:

```
for each new dependency:
  fetch_metadata(package)
  classify:
    [VERIFIED] - existing in npm/pypi >6mo + >10k weekly downloads + active repo
    [ASSUMED]  - exists but new/low-traffic
    [SLOP]     - 존재하지 않거나 hallucinated 이름
```

- `[SLOP]`: 즉시 block, plan 수정 요청
- `[ASSUMED]`: HITL gate ("이 패키지 정말 맞나요?")
- `[VERIFIED]`: 통과

이유: AI가 `react-supabase-toolkit` 같은 존재하지 않는 패키지를 자신감 있게 install하는 패턴 차단 ("slopsquatting" 표적 방지).

### 5.8 Self-Check Before Completion

`gsd-executor`의 완료 시 SUMMARY.md 마지막:

```markdown
## Self-Check
- Files claimed: src/foo.ts, src/bar.ts, tests/foo.test.ts
  - src/foo.ts: EXISTS
  - src/bar.ts: EXISTS
  - tests/foo.test.ts: EXISTS
- Commits claimed: abc123, def456
  - abc123: FOUND in git log
  - def456: FOUND in git log
- Result: PASSED
```

PASSED가 아니면 verifier에서 BLOCKED.

### 5.9 Adaptive Context Enrichment

agent dispatch 시:

```
if model.context_window >= 500_000:
  inject prior phases' SUMMARY.md (cross-plan awareness)
else:
  truncate to cache-friendly head/tail; STATE.md만 inject
```

모델 capability에 따라 컨텍스트 정책 자동 분기.

### 5.10 Decision Coverage Gates

CONTEXT.md의 각 decision은:

```yaml
- id: D-007
  req: [R-002, R-005]
  decided: 2026-05-18
  text: "auth provider는 Supabase Auth 사용 (Clerk 거부)"
  reason: "RLS 통합 우선"
  executed_by: [U2, U4]
  evidence: phases/03/SUMMARY.md#U2
```

- `executed_by`가 비어있고 phase가 진행됨 → **blocking error**
- decision이 코드로 옮겨지지 않은 채 끝나는 것 차단

### 5.11 Codebase Drift Detector (#2003)

`gsd-codebase-mapper`가 매번 실행 시 `last_mapped_commit` 추적:

```
diff = git log last_mapped_commit..HEAD
if diff has:
  - new directories
  - new barrel exports
  - new migrations
  - new routes:
    if count > threshold:
      auto_remap()
    else:
      warn
```

오래된 codemap이 silently rot하는 것 방지.

---

## 6. 출력 포맷 디테일

### 6.1 STATE.md

```markdown
# State
- Phase: 03 (Auth implementation)
- Wave: 2
- Active units: U3 (executor), U4 (executor)
- Completed units: U1, U2
- Blocked: none
- Mode: autonomous
- Last update: 2026-05-20T14:23:01Z by gsd-executor[U3]
```

### 6.2 CONTEXT.md

```markdown
# Context

## Decisions
- D-001 (REQ R-001): Stack = Next.js + Supabase
  - Reason: speed + RLS
  - Executed by: U1
- D-002 (REQ R-003): Use Stripe (not Lemon Squeezy)
  - ...

## Assumptions
- A-001: All users have email (not phone-only)
- ...
```

### 6.3 Verify Output (VERIFY.md)

```markdown
# Verify: phase 03
Generated: 2026-05-20

## Level 1: EXISTS
- src/auth/jwt.ts: PASS
- src/auth/middleware.ts: PASS

## Level 2: SUBSTANTIVE
- src/auth/jwt.ts: PASS (60 lines, no stub)
- src/auth/middleware.ts: FAIL (body is `throw new Error('TODO')`)
  → blocking

## Level 3: WIRED
(skipped — Level 2 failed)

## Level 4: REAL DATA FLOW
(skipped)

## Verdict: FAIL (U4 middleware is stub)
```

---

## 7. 주목할 디테일

- **67개 command의 평면 surface**: 사용자가 진입점을 어느 명령으로 잡아야 할지 불확실. 학습곡선이 큼.
- **agent + command + hook + SDK 4축**: 컴포넌트 수가 매우 많음 (총 110+). 풀스택을 다 이해 못해도 일부만 활용 가능하도록 설계.
- **SDK 의존**: agent prompt가 자유롭게 코드 짜는 게 아니라 `gsd-sdk query`로 결정을 deterministic하게 가져옴. 일관성 ↑ but 유연성 ↓.
- **`.planning/` markdown의 인간 편집성**: gated 모드/디버깅에 매우 유용. 사용자가 직접 STATE.md를 수정해서 재개 가능.
- **Wave 머지 패턴**: pre-commit hook 비용 절감과 lock contention 회피의 절묘한 절충.
- **adversarial verifier**: 대부분 verifier는 "통과 못한 증거를 찾으면 실패"인데, GSD는 "통과 증거가 없으면 실패". 디폴트가 다름.
- **15-런타임 매트릭스**: install 머신이 복잡 (tool name, hook signature, agent format 모두 normalize). 단일 런타임 사용자에게는 오버헤드.

---

## 8. 참조 URL

- README: https://github.com/gsd-build/get-shit-done
- 핵심 파일:
  - `hooks/gsd-context-monitor.js` (35%/25% 임계, debounced)
  - `hooks/gsd-workflow-guard.js`
  - `hooks/gsd-prompt-guard.js` (13 injection regex)
  - `agents/execution/gsd-executor.md` (worktree prohibition + analysis-paralysis)
  - `agents/verification/gsd-verifier.md` (4-level)
  - `agents/audit/gsd-assumptions-analyzer.md`
  - `agents/mapping/gsd-codebase-mapper.md` (drift detector)
- SDK: `sdk/gsd-sdk`
