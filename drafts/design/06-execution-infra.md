# 06. 실행 인프라 — Worktree · 컨텍스트 격리 · 상태 메모리

> 워크트리 병렬 실행, 컨텍스트 격리 메커니즘, `.harness/STATE.md` 상태 보호.

---

## 6.1 컨텍스트 격리 메커니즘

3-레벨 격리:

| 레벨 | 보유 컨텍스트 | 격리 수단 |
|---|---|---|
| **Maestro** | `goal.md`, `board/*`, Phase별 요약(JSON), 사용자 발화 | Foreman 결과는 ≤500자 요약만 받음. 코드/diff/raw sentinel 출력 안 받음. |
| **Foreman** | `dag.json`, 각 worker의 step-result 요약, Task Spec | 각 worker의 코드/diff는 절대 안 봄 |
| **Worker(L2)** | 자기 task spec, 자기 worktree 파일들 | step별로 별도 task() 호출로 fresh context |
| **Worker step** | 직전 step의 산출물만 (plan.md, diff 등) | 입력 명세에 의해 강제 |

기술적 구현:
- OmO `task()` tool로 sub-step을 호출하면 **별도 세션**으로 실행됨
- 각 step의 출력은 구조화된 JSON으로 받음 (`{plan_path, summary}` 등)
- 상위 에이전트는 그 JSON만 보관

**P12 강제**: 사용자 화면에는 항상 Maestro의 1차 가공된 메시지만 나옴. Foreman/Sentinel/L2 워커의 stdout은 시스템 차원에서 user-facing 채널 X.

---

## 6.2 Worktree 병렬화

### 6.2.1 디렉토리 컨벤션

```
project/
├─ .git/
├─ .harness/                  ← 런타임 상태
└─ .worktrees/                ← 병렬 작업장 (gitignore)
   ├─ backend-T-101-U-001/
   ├─ frontend-T-102-U-001/
   ├─ db-T-103-U-001/
   └─ ...
```

worktree 경로: `.worktrees/<role>-<T-id>-<U-id>` (예: `.worktrees/backend-T-101-U-001`)

- `<role>`: 워커 종류 (backend/frontend/db/security/devops/qa)
- `<T-id>`: 부모 Task ID
- `<U-id>`: Foreman 내부 unit ID

### 6.2.2 머지 전략 (Foreman 단독 책임)

- 각 worker는 자기 worktree에서만 작업
- Foreman은 **task 완료 순서대로** `dev`에 머지 시도
- Wave 머지: 같은 wave 워커들이 끝나면 Foreman이 **pre-commit hook을 1회 일괄 실행** (per-task 호출 X → 호출 횟수 N배 절감)
- 충돌 발생 시:
  - 자동 머지 가능 → 진행
  - 충돌 → `git-master` skill의 Rebase Surgeon에 위임
  - 본질적 충돌(같은 파일 같은 함수) → DAG 재설계 (Planning 단계로 회귀)

### 6.2.3 Worktree Prohibition Layer

Foreman의 worktree dispatcher PreToolUse hook으로 다음 명령 차단:

```
금지 명령 (post-startup):
- git stash           ← worktree 간 공유되어 다른 worktree의 WIP를 끌어옴
- git reset --hard
- git clean -fd
- git update-ref refs/heads/main
- git push --force-with-lease to protected refs
- git push -f

모든 commit 직전 어서션:
- assert: HEAD == expected_head
- assert: cwd == expected_worktree_path
- assert: git status 의 untracked가 plan에 명시됨
```

`git stash`는 worktree 간 silent하게 leak되므로 명시 차단 필수.

### 6.2.4 Patch Harvest (SHA256 dedup)

여러 worktree에서 같은 패치를 중복 생성하는 것 방지:

```
on worker step:commit:
  patch_hash = sha256(diff)
  dedup_index = .harness/_worktree-dedup.json
  if dedup_index[patch_hash]:
    skip commit (이미 다른 worktree에서 적용됨)
  else:
    commit + record_hash
```

### 6.2.5 DAG 메타룰 (Foreman의 휴리스틱)

- DB 스키마 변경은 항상 다른 모든 코드보다 **선행**
- Security task는 해당 도메인 task 완료 **후행** (실코드 보고 감사)
- Design spec은 Frontend task의 **선행 조건**
- 같은 파일을 수정하는 두 unit은 같은 wave에 둘 수 없음 (직렬화 강제)

→ Foreman 프롬프트에 이 메타룰을 하드코딩.

---

## 6.3 STATE.md (atomic locking)

### 6.3.1 위치 및 보호

`.harness/STATE.md` — 단일 파일, **`O_EXCL` lock** 사용.

```bash
exec {LOCK_FD}>.harness/.state.lock
flock -n $LOCK_FD || (sleep 0.5 && retry)
# critical section: read/modify/write STATE.md
exec {LOCK_FD}>&-
```

- 10초 timeout
- stale lock 감지 (lock holder PID 사망 확인 후 재시도)
- 병렬 worktree에서 같은 STATE.md를 안전하게 갱신

### 6.3.2 STATE.md 본문

```markdown
# State

## Current Cycle
- vision: V-001
- roadmap: R-001
- milestone: M-003
- phase: execution
- phase_entered_at: 2026-05-25T09:00:00Z
- phase_completed_at: null
- mode: gated

## Active Workers
- backend (T-101): in_progress (worktree: backend-T-101-U-001)
- frontend (T-102): in_progress (worktree: frontend-T-102-U-001)
- qa (T-103): pending

## Completed (this milestone)
- T-099, T-100

## Blocked
- none

## Last Sentinel Run
- 2026-05-26T11:23:04Z
- pass: 12, fix: 3 (applied), block: 0

## Recent Events
- 2026-05-26T11:00:00Z [maestro] dispatched T-101 to Foreman
- 2026-05-26T11:23:04Z [sentinel] T-101 commit abc123 PASS
- 2026-05-26T11:45:00Z [foreman] T-101 done, merged to dev
```

### 6.3.3 STATE.md 갱신 권한

| 에이전트 | 갱신 권한 |
|---|---|
| Maestro | phase, mode, Current Cycle 전체 |
| Foreman | Active Workers, Completed, Blocked, Recent Events |
| Sentinel | Last Sentinel Run, Recent Events |
| L2 워커 | Recent Events만 (자기 task 관련) |

모든 갱신은 lock 안에서만. append-only가 자연스러운 필드(Recent Events)는 append만.

---

## 6.4 `.harness/` 영구 상태 디렉토리

```
.harness/
├─ goal.md                   ← Phase 0에서 사용자 합의된 골 (불변)
├─ prd.md                    ← Strategist 산출 (milestone 단위 갱신)
├─ design-spec.md            ← Designer 산출
├─ ETHOS.md                  ← 모든 워커 preamble에 inject되는 원칙 문서
├─ STATE.md                  ← 현재 phase + active workers (O_EXCL lock)
├─ PROJECT.md                ← 프로젝트 메타 (이름, 목적, stakeholder)
├─ REQUIREMENTS.md           ← REQ-ID로 분해된 요구사항
├─ ROADMAP.md                ← roadmap 진행 상황 (board/roadmap.md의 운영 뷰)
├─ CONTEXT.md                ← 모든 결정 + REQ-ID 태그 (append-only)
├─ progress.jsonl            ← append-only 진행 로그
├─ sentinel-log.jsonl        ← append-only Sentinel finding 로그
├─ board/                    ← PM 엔티티 (03-pm-model.md 참조)
│   ├─ vision.md / roadmap.md
│   ├─ milestones/M-*.md
│   ├─ backlog/B-*.md
│   └─ tasks/T-*.md
├─ phases/                   ← 각 phase 산출 (Spec/Report/DoD verify 결과)
│   └─ <N>/
│       ├─ PLAN.md
│       ├─ SUMMARY.md
│       └─ VERIFY.md
├─ lessons/                  ← 3-tier compound 영구 저장
│   ├─ _pending/             ← Tier 1 후보 (워커 emit, 임시)
│   ├─ architecture/
│   ├─ bugs/
│   ├─ perf/
│   ├─ ops/
│   ├─ process/
│   └─ prompts/
└─ proposals/                ← Phase 8 진화 제안
    ├─ _pending/
    ├─ accepted/             (commit ID 메타 포함)
    ├─ rejected/             (사유 메모)
    └─ superseded/
```

### git 커밋 정책

- `board/`, `vision.md`, `goal.md`, `prd.md`, `design-spec.md`, `ETHOS.md`, `lessons/`, `proposals/accepted/` → **git commit 권장**
- `STATE.md`, `progress.jsonl`, `sentinel-log.jsonl`, `phases/`, `_pending/` → gitignore
- `.worktrees/` → gitignore

---

## 6.5 STATE.md ↔ board 동기화

board 엔티티 변경 시 STATE.md의 cross-reference 자동 갱신:

```
사건                            STATE.md 갱신
─────────────────────────────  ──────────────────────
Milestone M-003 active        Current Cycle.milestone = M-003
DoD verify 실행 결과 변경       Last DoD Verify 추가
Task T-101 dispatched          Active Workers 추가
Task T-101 done                Active Workers 제거 + Completed 추가
phase 전이                     phase + phase_entered_at/completed_at
```

이 동기화는 Maestro/Foreman의 책임. Sentinel은 동기화 누락을 감시.
