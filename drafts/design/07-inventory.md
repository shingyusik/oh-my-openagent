# 07. 파일·디렉토리 인벤토리 + 사용자 명령

> 본 하네스가 실제로 디스크에 남기는 모든 산출물 + 사용자가 Maestro에 보내는 모든 명령.

---

## 7.1 디렉토리 인벤토리 (전체)

```
project/
├─ .git/
├─ .opencode/                          ← 하네스 정의 (git commit)
│   ├─ oh-my-openagent.jsonc           ← 카테고리/에이전트 오버라이드 + 모드 기본값
│   ├─ agents/                          ← 마크다운 에이전트 13개
│   │   ├─ maestro.md                   L0 (단일 사용자 창구 + PM)
│   │   ├─ foreman.md                   L1
│   │   ├─ sentinel.md                  L1
│   │   ├─ strategist.md                L2 기획
│   │   ├─ designer.md                  L2 디자인
│   │   ├─ frontend.md                  L2
│   │   ├─ backend.md                   L2
│   │   ├─ db-architect.md              L2
│   │   ├─ security.md                  L2
│   │   ├─ devops.md                    L2
│   │   ├─ qa.md                        L2
│   │   ├─ tech-writer.md               L2 (lessons 작성 Lead)
│   │   └─ agent-architect.md           L2 (메타, 진화 사이클)
│   ├─ skills/
│   │   ├─ sentinel-rules/SKILL.md      Sentinel 룰셋 + 정적 도구 호출 레시피
│   │   ├─ pm-board/SKILL.md            ★ 6엔티티 CRUD + 4-step + DoD 인터뷰
│   │   ├─ worktree-orchestrator/SKILL.md
│   │   ├─ dag-builder/SKILL.md
│   │   ├─ saas-stack/SKILL.md          스택 컨벤션 (TS/Tailwind/shadcn/Python/Supabase/CF/PocketBase)
│   │   ├─ monorepo-layout/SKILL.md     모노레포 구조 강제
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
└─ <SaaS 프로젝트 모노레포 본체>
    ├─ apps/web/                     ← Next.js (TS, Tailwind, shadcn)
    ├─ apps/workers/                 ← Cloudflare Workers (edge runtime)
    ├─ apps/pyservices/              ← Python 서비스 (FastAPI 등)
    ├─ packages/ui/                  ← shadcn 컴포넌트 eject 위치
    ├─ packages/db/                  ← Supabase 마이그레이션 + 타입
    ├─ packages/shared/              ← 공유 타입/유틸 (TS only)
    ├─ packages/py-shared/           ← Python 공유 (pydantic 모델 등)
    ├─ infra/cloudflare/             ← wrangler.toml, IaC
    ├─ infra/supabase/               ← config + migrations
    ├─ tests/e2e/                    ← Playwright 통합 테스트
    └─ docs/                         ← Tech Writer 산출물
```

---

## 7.2 타겟 스택

| 레이어 | 기술 | 비고 |
|---|---|---|
| Lang (FE) | **TypeScript** (strict) | tsc strict, no implicit any |
| Lang (BE/Tools) | **Python** | ruff + black + mypy |
| UI 스타일 | **Tailwind CSS** | shadcn 토큰 시스템과 정렬 |
| UI 컴포넌트 | **shadcn/ui** | 컴포넌트는 `packages/ui/`에 eject |
| BaaS | **Supabase** | auth, realtime, storage, RLS |
| DB (primary) | **Postgres** (Supabase 호스팅) | RLS 정책 필수 |
| DB (보조) | **PocketBase** | local-first / 임베디드 시나리오 |
| 인프라 | **Cloudflare** | Workers (edge API), Pages (FE), R2 (asset), D1 (옵션), KV |

`saas-stack` skill이 각 항목의 베스트 프랙티스 + 흔한 함정을 룰화. 예:
- "Cloudflare Workers에서 `fs`, `crypto.randomBytes`, 동기 fs API 사용 금지"
- "Supabase 테이블 추가 시 RLS 정책 동시 작성 필수"
- "shadcn 컴포넌트는 `packages/ui/`에 eject 후 수정, 직접 인라인 수정 금지"

---

## 7.3 모노레포 의존 방향 룰

- `apps/*` → `packages/*` ✅
- `packages/*` → `packages/*` ✅ (순환 금지)
- `packages/*` → `apps/*` ❌
- TS 코드 → Python 코드 ❌ (반대도)
- 통신은 HTTP/RPC만

**모노레포 도구**: pnpm workspaces + turborepo (TS 측), uv (Python 측). 첫 부트스트랩에서 확정.

→ Sentinel `architecture_violation`이 이 룰을 강제.

---

## 7.4 사용자 명령 인터페이스

> 사용자는 Maestro와만 대화. 모든 명령은 Maestro가 해석·실행.

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

/tdd-exception <task-id> "<사유>"          # TDD 예외 등록 (Maestro 승인 필요)

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

→ Maestro가 영향 범위 분석 → 영향 backlog item 자동 재정렬 → 변경 이력 CONTEXT.md append → 사용자 컨펌.

---

## 7.5 사용자가 보지 못하는 명령들

다음은 시스템 내부 명령. 사용자가 직접 호출할 수 없다 (P12 강제):

- Foreman의 worktree spawn / merge
- Sentinel 룰 실행
- L2 워커의 `task()` 호출
- Agent-Architect의 proposal 작성
- compound emit

이 명령들의 결과는 항상 Maestro를 거쳐 가공된 메시지로만 surface.
