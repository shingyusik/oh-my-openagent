# 05. Sentinel — 품질 감독관

> 매 commit / phase 경계 / merge gate에서 자동 실행되는 항상-감시. 룰 위반은 `AUTO-FIX` / `ASK` / `BLOCK`으로 분기.

---

## 5.1 Sentinel 트리거

- **Git post-commit hook** (worktree 내부) — 매 워커 commit마다
- **Phase 경계 hook** — Refinement/Planning/Execution/Review 진입/종료마다
- **Merge gate** — Foreman이 worktree → dev 머지 직전
- **명시 호출** — `/sentinel-check [scope]`, `/dod verify <M-id>`

---

## 5.2 룰 우선순위 (P0 / P1 / P2)

- **P0**: 매 commit 무조건 검사. BLOCK 권한.
- **P1**: 매 commit 검사하지만 위반 즉시 BLOCK 아님 (배치 처리). 머지 게이트에서 강제.
- **P2**: 머지 게이트에서만 검사.

`severity`:
- `warn` → 로그만, 진행
- `fix` → Sentinel이 직접 fix diff 생성, worker가 적용
- `block` → 커밋 revert, worker에게 재계획 요구

---

## 5.3 룰 카탈로그

```yaml
sentinel_rules:

  # ==== P0: 최우선 (Plan-first, No silent expansion 원칙) ====
  over_engineering:                           # P0
    - rule: "interface/abstract with single implementer"
      severity: block
    - rule: "DI container / IoC for ≤3 services"
      severity: block
    - rule: "config flag for unimplemented feature"
      severity: block
    - rule: "premature generics (T used only once in same module)"
      severity: fix
    - rule: "factory/builder for ≤2 instantiation sites"
      severity: fix
    - rule: "wrapper class that only forwards calls"
      severity: block
    - rule: "design pattern (strategy/observer/visitor) with single concrete"
      severity: block
    - rule: "speculative future-proof params/options (never read)"
      severity: block

  dead_code:                                  # P0
    - rule: "exported but never imported"
      severity: fix
    - rule: "function/method defined but never called"
      severity: fix
    - rule: "unreachable branch"
      severity: fix
    - rule: "commented-out code blocks > 3 lines"
      severity: fix
    - rule: "unused import / unused parameter (non-_prefixed)"
      severity: fix
    - rule: "TODO/FIXME without owner+date older than this PR"
      severity: warn

  # ==== P1: 아키텍처 위반 ====
  architecture_violation:                     # P1
    - rule: "frontend imports backend internals (cross-app boundary)"
      severity: block
    - rule: "backend reads from frontend assets"
      severity: block
    - rule: "domain layer imports infrastructure (Supabase client, fetch 등)"
      severity: block
    - rule: "circular dependency between packages"
      severity: block
    - rule: "raw SQL outside db package"
      severity: block
    - rule: "auth check missing on protected route"
      severity: block
    - rule: "shadcn 컴포넌트 직접 수정 (eject 없이 인라인 수정)"
      severity: fix
    - rule: "Tailwind 클래스 충돌 / 임의 magic value"
      severity: fix
    - rule: "Cloudflare Workers에서 node-only API 사용"
      severity: block
    - rule: "Supabase RLS 정책 누락 (테이블 추가 시)"
      severity: block
    - rule: "API 핸들러가 도메인 모델 직접 변형 (서비스 레이어 우회)"
      severity: block
    - rule: "monorepo 의존 방향 위반 (apps → packages 만 허용)"
      severity: block

  # ==== P1: 스코프 일탈 ====
  scope_creep:                                # P1
    - rule: "files changed not in task.plan.touched_files"
      severity: block
    - rule: "현재 task의 role과 무관한 파일 수정 (예: backend worker가 frontend 수정)"
      severity: block

  # ==== P1: 컨벤션 ====
  conventions:                                # P1
    - rule: "lint/format errors (eslint + prettier + ruff + black)"
      severity: fix
    - rule: "naming convention violation (project AGENTS.md 규약 기준)"
      severity: fix
    - rule: "타입 미정의 (any/unknown 남발, Python untyped)"
      severity: fix
    - rule: "magic number/string (3회 이상 반복)"
      severity: fix

  # ==== P2: 머지 게이트 ====
  ai_slop:                                    # P2
    - rule: "comments explaining WHAT (not WHY)"
      severity: fix
    - rule: "redundant try/catch wrapping infallible code"
      severity: fix
    - rule: "logging that just restates parameters"
      severity: fix
    - rule: "JSDoc/docstring that just rewords function name"
      severity: fix

  test_coverage:                              # P2
    - rule: "changed lines coverage < 70%"
      severity: warn
    - rule: "변경된 public function에 테스트 없음"
      severity: fix

  security_lite:                              # P0 (Security 워커 풀감사 외 상시)
    - rule: "literal secret/key in code"
      severity: block
    - rule: "eval / exec / raw shell with interpolation"
      severity: block
    - rule: "missing input validation on API boundary"
      severity: fix

  # ==== P0: TDD 강제 (P9) ====
  tdd_violation:                              # P0
    - rule: "코드 변경 commit에 대응하는 신규/수정 테스트 부재"
      severity: block
      # 단, plan에 tdd_exception 명시되고 Maestro 승인 마커 있으면 통과
    - rule: "테스트가 모든 가능 입력에 PASS 반환 (assertion-free placeholder test)"
      severity: block
    - rule: "step:red 단계에서 새 테스트가 FAIL 증거 없음 (테스트 실행 로그에 PASS만)"
      severity: block
    - rule: "기존 테스트 삭제 + 같은 step에서 코드 변경 (안 보이는 회귀)"
      severity: block
    - rule: "test coverage 변화 -5% 초과 감소"
      severity: warn
    - rule: "skip된 테스트 / `it.skip` / `@pytest.mark.skip` 증가"
      severity: fix
    - rule: "마이그레이션 SQL이 dry-run 증거 없이 commit"
      severity: block

  # ==== P0: Compound 강제 (P10) ====
  compound_required:                          # P0
    - rule: "워커 task 완료 보고에 compound_emitted 필드 없음"
      severity: block
    - rule: "Phase 종료에 .harness/lessons/_pending/* 합쳐서 .harness/lessons/<cat>/L-*.md 산출 없음"
      severity: block
    - rule: "lesson 마크다운에 YAML frontmatter (track/category/problem_type/module/tags) 누락"
      severity: fix
    - rule: "같은 root_cause의 lesson이 3+ 사이클에서 재등장하고 ETHOS/proposal 미반영"
      severity: warn

  # ==== P0: PM 워크플로 강제 (P13) ====
  backlog_singularity:                        # P0
    - rule: "코드 주석에 TODO/FIXME 추가하고 backlog/B-* 미생성"
      severity: block
    - rule: ".harness/ 외부에 todo list / planning markdown 생성 시도"
      severity: block
    - rule: "워커 보고에 새 발견 lesson 있으나 backlog item 생성 안됨"
      severity: block
    - rule: "백로그 외 위치에 'tasks' 'todos' 'planning' 류 키워드 파일 생성"
      severity: fix
    - rule: ".harness/board/backlog/_index.md priority 순 미정렬"
      severity: fix

  workflow_phase_skip:                        # P0 (P14)
    - rule: "STATE.md의 phase가 refinement인데 task dispatch 시도"
      severity: block
    - rule: "STATE.md의 phase가 planning인데 worker 7-step 진입 시도"
      severity: block
    - rule: "이전 phase의 phase_completed_at == null인데 다음 phase 진입"
      severity: block
    - rule: "phase_entered_at 없이 phase 동작 수행"
      severity: block
    - rule: "Review 거치지 않고 다음 milestone Refinement 진입"
      severity: block

  dod_required:                               # P0 (P15)
    - rule: "milestone 생성에 dod 필드 없거나 비어있음"
      severity: block
    - rule: "DoD 항목에 verify_cmd 없음"
      severity: block
    - rule: "DoD criterion에 측정 불가 표현 ('대충 잘', '잘 되어야', '괜찮으면')"
      severity: block
    - rule: "milestone.status = done 시도인데 DoD 중 state != passed 존재"
      severity: block
    - rule: "DoD verify_cmd 실행 결과 0 exit code가 아닌데 state=passed 표시"
      severity: block
    - rule: "DoD 변경 시 변경 이력 CONTEXT.md 미기록"
      severity: fix

  discovered_not_logged:                      # P0 (P13 보강)
    - rule: "워커가 'we should also...', 'TODO:', 'FIXME:', '추후', '나중에' 언급 후 backlog 미등록"
      severity: block
    - rule: "Sentinel 자체가 BLOCK 5회 같은 룰 누적했는데 backlog/B-*(type=tech_debt or bug) 미생성"
      severity: fix  # Sentinel 자기 자신이 자동 생성

  flexibility_traceability:                   # P1 (P16)
    - rule: "roadmap/milestone 변경 시 영향받는 backlog item priority 재정렬 없음"
      severity: fix
    - rule: "milestone 삭제/취소 시 종속 backlog item 처리 (close/reassign) 없음"
      severity: block
    - rule: "vision/roadmap 변경이 CONTEXT.md append-only 로그에 없음"
      severity: fix

  # ==== P0: Analysis-Paralysis Guard ====
  # AI가 끝없이 탐색만 하고 작업을 안 하는 패턴 차단
  analysis_paralysis:                         # P0
    - rule: "연속 5회 read-only 도구 호출 (Read/Grep/Glob/lsp_*) 후 Write/Edit/Bash 없음"
      severity: block
      action: "워커는 1문장으로 왜 안 썼는지 자백 또는 BLOCKED 보고 후 다음 1턴 안에 Write 실행"
    - rule: "같은 파일을 5회 이상 Read (변경 없이)"
      severity: warn
    - rule: "10턴 누적 read-only ratio > 90%"
      severity: block

  # ==== P0: Slopcheck (패키지 정합성) ====
  # AI hallucination 패키지 차단 ("slopsquatting" 표적 방지)
  slopcheck:                                  # P0  [NEVER_GATE]
    - rule: "package.json / requirements.txt / pyproject.toml에 새 의존성 추가 후 slopcheck 미실행"
      severity: block
    - rule: "패키지 등급 [SLOP] (npm/pypi 존재 X 또는 hallucinated 이름)"
      severity: block
      action: "BLOCK. backlog에 등록하지도 말 것"
    - rule: "패키지 등급 [ASSUMED] (존재하나 신규/저트래픽)"
      severity: block
      action: "Maestro HITL gate 발동. 사용자 컨펌 후만 통과"
    - rule: "Cloudflare Workers/Pages 환경에 node-only 의존성 추가"
      severity: block
  # 등급 판정 휴리스틱:
  #   [VERIFIED] - npm/pypi 존재 ≥6mo + ≥10k weekly downloads + active repo + 이름 정확
  #   [ASSUMED]  - 존재하나 신규/저트래픽
  #   [SLOP]     - 존재하지 않거나 이름 유사 (오탈자 가능성)

  # ==== P0: Self-Check Before Completion ====
  # "should work" 거짓완료 차단
  self_check_required:                        # P0
    - rule: "워커 task 종료 보고 (SUMMARY.md)에 '## Self-Check' 블록 없음"
      severity: block
    - rule: "Self-Check에 claimed file 중 디스크에 존재하지 않는 파일 있음"
      severity: block
    - rule: "Self-Check에 claimed commit hash 중 git log에 없는 것 있음"
      severity: block
    - rule: "Self-Check 결과 'PASSED'인데 실제 검증 명령 실행 로그 없음"
      severity: block
    - rule: "워커 응답에 'should work', 'probably', 'might work', 'I think it's done' 류 hedge 단어 + 실행 증거 없음"
      severity: block

  # ==== P0: Plan Placeholder Validator ====
  # Strategist/Maestro plan 산출물의 모호 표현 차단
  plan_placeholder:                           # P0
    - rule: "plan 본문에 'TBD', 'TODO', 'XXX', '나중에', '추후', 'somehow' 표현"
      severity: block
    - rule: "task에 'add validation', 'wire it up', 'handle errors' 류 추상 동작만"
      severity: fix
      action: "구체 파일 경로 + literal code 또는 명령 요구"
    - rule: "task에 file path 명시 없음"
      severity: block
    - rule: "task에 acceptance criteria (AC) 명시 없음"
      severity: block
    - rule: "milestone DoD에 측정 가능한 verify_cmd 없는 항목"
      severity: block  # dod_required와 중복이지만 placeholder 관점에서 재확인
```

---

## 5.4 비용 제어 (2-tier)

Sentinel은 매 commit마다 돌므로 1차/2차로 분리:

### 1차 패스 (cheap, 매 commit)
- 모델: `quick` 카테고리 (gpt-5.4-mini)
- 정적 도구 위주: ts-prune, knip, eslint, ruff, ts-unused-exports, depcruise
- P0 룰 + 단순 패턴 매칭
- 빠른 결과 (수 초 이내)

### 2차 패스 (expensive)
- **트리거**: 1차에서 의심 발견 시 + 머지 게이트 + DoD verify
- 모델: `ultrabrain` 카테고리 (gpt-5.4 xhigh)
- 아키텍처 위반의 의미론적 판정 (Cloudflare 호환성, RLS 정책 누락 등)
- 과설계 판정의 회색 영역
- 길어질 수 있는 추론 (수 십 초)

---

## 5.5 정적 도구 매핑 (스택 종속)

| 룰 카테고리 | TypeScript 도구 | Python 도구 |
|---|---|---|
| dead_code | `ts-prune`, `knip`, `eslint no-unused-*` | `vulture`, `ruff F401/F841` |
| over_engineering | `eslint-plugin-sonarjs` (cognitive-complexity), custom ast-grep | `radon`, `mccabe` |
| architecture_violation | `dependency-cruiser`, `eslint-plugin-boundaries` | `import-linter` |
| conventions | `eslint` + `prettier` + tsc strict | `ruff` + `black` + `mypy` |
| security_lite | `eslint-plugin-security`, `semgrep` (JS rules) | `bandit`, `semgrep` (py rules) |
| test_coverage | `vitest --coverage`, `c8` | `pytest-cov` |
| tdd_violation | git diff + `vitest list`/`pytest --collect-only` | (위) |

→ 위 도구들의 출력을 Sentinel이 통합 해석. `.opencode/skills/sentinel-rules/SKILL.md`에 도구 호출 레시피 작성.

---

## 5.6 출력 통합 (Fingerprint-merge)

여러 룰/렌즈가 같은 finding을 중복 발견하므로 dedup:

```
finding_fingerprint = (
  normalize(file_path),
  bucket(line_number, ±3),      // 인접 3줄은 같은 finding
  normalize(title)               // 공백/언어 변형 무시
)
```

같은 fingerprint를 가진 finding이 N개 룰에서 나오면:
| N | confidence |
|---|---|
| 1 | 50 |
| 2 | 75 |
| 3+ | 100 |

추가 룰:
- **P0 + 2+ rule agreement**: confidence demotion에서 면제
- **mode 인식 demotion**: headless/dry-run에서는 P2 advisory는 demote, P0+는 유지

---

## 5.7 Adaptive Gating

각 룰의 hit-rate를 영구 추적:

```
for each rule:
  if rule.last_N_runs.findings_count == 0 (N=10):
    if rule.tag != "[NEVER_GATE]":
      auto_disable
```

**`[NEVER_GATE]` 태그** (절대 비활성 X):
- `security_lite`
- `architecture_violation`
- `tdd_violation`
- `dod_required`
- `backlog_singularity`
- `workflow_phase_skip`
- `slopcheck`
- `self_check_required`
- `plan_placeholder`

이유: low-frequency-but-catastrophic 룰.

비활성된 룰은 Phase 8 Evolution에서 재평가. 새 룰 추가 시 자동 재활성 검토.

---

## 5.8 AUTO-FIX vs ASK 분류

모든 finding은 다음 두 분류:

### AUTO-FIX (Sentinel이 직접 패치)
- prettier/ESLint/ruff 위반
- unused import 제거
- 명백한 dead code 삭제
- import 순서, naming case 통일

→ Sentinel이 patch 생성, worker가 적용, commit에 포함.

### ASK (Maestro에 escalate)
- 명명 모호성 (alternative 명 제시)
- 추상화 수준 (A vs B trade-off)
- 비즈니스 로직 의미 모호
- 룰 위반이지만 의도적일 수 있음

→ Maestro가 사용자에게 batch로 surface ("3개 ASK 처리 필요").

---

## 5.9 4-Level 검증 방법론 (Goal-Backward Verifier)

DoD verify_cmd 실행, merge gate, milestone Review 단계에서 사용되는 **4단계 검증 사다리**. **"통과 못한 증거가 있어야 통과"가 아니라 "통과 증거가 없으면 실패"** (adversarial).

각 단계는 이전 단계 통과 후에만 진입. 한 단계라도 실패하면 즉시 BLOCK.

### Level 1 — EXISTS
- 변경된 모든 entrypoint 파일이 디스크에 존재하는가?
- 모든 import 경로가 resolve되는가?
- 명시된 commit hash가 git log에 존재하는가?
- 실패 시: 누락 파일/import/hash 리스트 반환 → BLOCK

### Level 2 — SUBSTANTIVE
- 함수 body가 stub이 아닌가? (`throw new Error("notImplemented")`, `pass`, `return null` 단독 X)
- 변수가 hardcoded 더미 값 아닌가? (`return []`, `return {}`, `const data = null` 단독 X)
- 테스트가 assertion 1개 이상 갖는가?
- 실패 시: stub/dummy 위치 리스트 → fix 요청 또는 BLOCK

### Level 3 — WIRED
- 새 컴포넌트가 부모에 mount 되었는가?
- 새 route가 router에 등록되었는가?
- 새 API endpoint가 export + 라우터에 연결되었는가?
- 새 마이그레이션이 supabase migration list에 등재되었는가?
- 실패 시: dangling 모듈 리스트 → BLOCK

### Level 4 — REAL DATA FLOW
- 실제 데이터(DB row / API response)를 흘려보면 결과가 나오는가?
- mock/fixture 아닌 실 데이터 1회 통과 증거가 있는가?
- e2e 테스트가 실제 시스템 통과 (mock 외부 호출 아님)?
- 실패 시: data flow 단절 지점 → fix 또는 BLOCK

### 사용처
- **DoD verify** (Milestone Review 단계): 각 DoD 항목의 verify_cmd가 4-level 중 어디까지 검증하는지 명시. 권장: Level 4까지.
- **Worker step:commit 직전**: 해당 변경의 entrypoint에 대해 1~4 일괄
- **Merge gate**: worktree → dev 머지 직전 Foreman이 호출

### 출력
```
Level 1 (EXISTS):         PASS (3 files, 2 commits 확인)
Level 2 (SUBSTANTIVE):    PASS
Level 3 (WIRED):          FAIL — apps/web/src/routes/login.tsx not mounted in apps/web/src/app.tsx
Level 4 (REAL DATA FLOW): SKIPPED (Level 3 미통과)

Verdict: BLOCK
```

대부분 하네스가 Level 1~2만 검사하고 끝남. Level 3 + 4가 본 하네스의 차별 검증력.

---

## 5.10 출력 포맷

```markdown
## Pass 1 (AUTO-FIXED, 적용 완료)
- [src/foo.ts:12] unused import → removed
- [src/bar.ts:45] prettier → applied

## Pass 2 (ASK, Maestro escalate)
- [src/baz.ts:30] naming: `data` is generic
  Options: `userPayload`, `apiResponse`, `inputDoc`
  Recommend: userPayload (matches existing pattern)

## Findings (merged, confidence 정렬)

### High (confidence 100)
- [U2] [src/auth.ts:42] missing input validation
  Sources: security_lite, conventions, architecture_violation

### Medium (confidence 75)
- [U1] [src/foo.ts:18] unclear naming
  Sources: ai_slop, conventions

## TDD Compliance
- tests_added: 4
- tests_modified: 0
- coverage_delta: +2.1%
- exceptions_used: 0

## PM Workflow Check
- phase: execution
- backlog_singularity: PASS
- discovered_not_logged: 0 violations

## Verdict
- Pass 1 fixes: 5 applied
- Pass 2 asks: 1 pending Maestro
- BLOCK: 0
- Verdict: PASS / FIX / BLOCK
```
