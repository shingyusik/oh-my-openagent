# 외부 하네스 패턴 — 내 코딩 하네스 적용 계획

> 상태: **v0.3** (2026-05-20)
> Companion: [`surveys/superpowers.md`](./surveys/superpowers.md) · [`surveys/compound-engineering.md`](./surveys/compound-engineering.md) · [`surveys/gstack.md`](./surveys/gstack.md) · [`surveys/get-shit-done.md`](./surveys/get-shit-done.md)
> Target: [`my-harness-design.md`](./my-harness-design.md) + [`design/*.md`](./design/) (현재 v0.5.1, split 구조)
>
> 본 문서는 **surveys/에서 발굴한 외부 패턴들이 현재 design/*.md의 어디에 어떻게 매핑되었는지의 결정 기록**이다. 각 패턴의 동작 원리는 surveys/에서 확인.
>
> **v0.3 변경**: design이 v0.4 PM 6-tier, v0.5 4-step cycle/DoD, v0.5.1 누락 5룰 적용으로 진화한 것을 반영. 모든 섹션 참조를 split 구조에 맞춰 `design/0X-*.md`로 갱신. S-tier 13개 전부 + A-tier 대부분 적용 완료 status 표기.

---

## 0. 한 줄 결론

> 4개 외부 하네스에서 **S-tier 13개 + A-tier 13개 + B-tier 8개 = 총 34개 패턴**을 발굴. **S-tier 13개 전부 design에 적용 완료** (v0.5.1 시점). A-tier는 대부분 적용, B-tier는 옵션으로 보류. 가장 위험한 발견은 GSD의 `git stash` worktree-leak — `design/03-l1-foreman-execution.md` §6.2.3에 prohibition layer로 명시 적용.

---

## 1. 적용 우선순위 매트릭스 + 적용 status

평가축: **Impact** × **Fit** × **Effort**. **Status**: ✅ 적용 완료 / ⚙️ 부분 적용 / ⏳ pending / ⛔ skip

### S-tier — 반드시 이식 (13개, 모두 적용 완료) ✅

| # | 패턴 | 출처 | Impact | Fit | Status | design 위치 |
|---|---|---|---|---|---|---|
| S1 | **Stable U-ID** (plan→commit→리뷰→PR 관통) | compound | ★★★ | ★★★ | ✅ | `04-l2-worker-profiles.md` worker step + `03-l1-foreman-execution.md` §6.2.1 worktree 경로 컨벤션 |
| S2 | **2-stage 리뷰 분리** (spec / quality 별도 task()) | superpowers | ★★★ | ★★★ | ✅ | `04-l2-worker-profiles.md` §2.3.1 7-step (spec-review + quality-review) |
| S3 | **Worktree prohibition layer** (git stash/reset-hard 금지 + HEAD 검증) | GSD | ★★★ | ★★★ | ✅ | `03-l1-foreman-execution.md` §6.2.3 |
| S4 | **STATE.md O_EXCL 잠금** | GSD | ★★★ | ★★★ | ✅ | `03-l1-foreman-execution.md` §6.3 |
| S5 | **Analysis-Paralysis Guard** (5 read-only → 자백) | GSD | ★★ | ★★★ | ✅ | `05-l1-sentinel-quality.md` §5.3 `analysis_paralysis` 룰 (P0) |
| S6 | **4-level verifier** (exists→substantive→wired→data flow) | GSD | ★★★ | ★★★ | ✅ | `05-l1-sentinel-quality.md` §5.9 별도 방법론 + DoD verify + spec-review + merge gate에서 사용 |
| S7 | **Slopcheck** (패키지 정합성 [VERIFIED/ASSUMED/SLOP]) | GSD | ★★★ | ★★★ | ✅ | `05-l1-sentinel-quality.md` §5.3 `slopcheck` 룰 (P0 [NEVER_GATE]) |
| S8 | **Self-Check before completion** | GSD | ★★★ | ★★★ | ✅ | `05-l1-sentinel-quality.md` §5.3 `self_check_required` + `04-l2-worker-profiles.md` §2.3.1 commit step Self-Check 블록 의무 |
| S9 | **Knowledge compounding** (`docs/solutions/<cat>/<slug>.md`) | compound | ★★★ | ★★ | ✅ | `06-cross-layer-workflows.md` §4.5 Tier 2 + `.harness/lessons/<category>/` |
| S10 | **Fingerprint-merge + confidence anchor** | compound | ★★★ | ★★★ | ✅ | `05-l1-sentinel-quality.md` §5.6 |
| S11 | **TDD-mandatory RED→GREEN→REFACTOR** | superpowers + GSD | ★★★ | ★★★ | ✅ | P9 (`01-overview.md`) + `04-l2-worker-profiles.md` §2.3.1 7-step + `05-l1-sentinel-quality.md` `tdd_violation` 룰 |
| S12 | **3-tier Compound** | compound + gstack | ★★★ | ★★★ | ✅ | P10 + `06-cross-layer-workflows.md` §4.5 Tier 1/2/3 + `05-l1-sentinel-quality.md` `compound_required` 룰 |
| S13 | **Harness Evolution (Phase 8)** | superpowers + compound + gstack | ★★★ | ★★★ | ✅ | P11 + `06-cross-layer-workflows.md` §4.6 + Agent-Architect 워커 6-step |

### A-tier — 강력 추천 (13개)

| # | 패턴 | 출처 | Status | design 위치 |
|---|---|---|---|---|
| A1 | **Plan placeholder 금지 validator** | superpowers | ✅ | `05-l1-sentinel-quality.md` §5.3 `plan_placeholder` 룰 + `04-l2-worker-profiles.md` Planning Worker 자기 출력 검증 책무 |
| A2 | **3-failure 아키텍처 escalation** | superpowers | ✅ | `06-cross-layer-workflows.md` §4.7.3 Ralph 루프 circuit-breaker |
| A3 | **Brainstorming 9-step 게이트** | superpowers | ✅ | `06-cross-layer-workflows.md` §4.2 Phase 0 인터뷰 통합 |
| A4 | **Adaptive specialist gating** (hit-rate 0 → auto-disable) | gstack | ✅ | `05-l1-sentinel-quality.md` §5.7 |
| A5 | **AUTO-FIX vs ASK 분류** | gstack | ✅ | `05-l1-sentinel-quality.md` §5.8 |
| A6 | **`.planning/` 파일 메모리 트리** | GSD | ✅ | `03-l1-foreman-execution.md` §6.4 `.harness/` 디렉토리 |
| A7 | **Phase 2.5 discoverability** (AGENTS.md surface) | compound | ✅ | `06-cross-layer-workflows.md` §4.5 Tier 2의 Agent-Architect 책무 |
| A8 | **ETHOS.md preamble injection** | gstack | ✅ | `.harness/ETHOS.md` (`03-l1-foreman-execution.md` §6.4) — 모든 워커 preamble |
| A9 | **`/learn` typed memory** (patterns/pitfalls/preferences/architecture) | gstack | ✅ | `.harness/lessons/<category>/` 6개 카테고리 (architecture/bugs/perf/ops/process/prompts) |
| A10 | **"Use when ..." description 컨벤션** | superpowers | ⏳ | skill 본문 작성 시 적용 예정 (현재 design에 컨벤션 명시는 누락) |
| A11 | **TDD 예외 게이트** (`/tdd-exception`) | superpowers | ✅ | `04-l2-worker-profiles.md` §2.3.1 TDD 예외 + `07-runtime-inventory.md` §7.4.5 명령 |
| A12 | **Lesson fingerprint-merge** (Tier 2/3 dedup) | compound | ⚙️ | Sentinel S10에 사용되지만 Compound Tier 2/3에는 명시적 알고리즘 미작성. 구현 시 보강 필요 |
| A13 | **Proposal 메타-회귀 deprecate** | gstack | ✅ | `06-cross-layer-workflows.md` §4.6.5 메타-회귀 방지 |

### B-tier — 가능하면 (8개)

| # | 패턴 | 출처 | Status | 메모 |
|---|---|---|---|---|
| B1 | **SKILL.md.tmpl + host adapter** | gstack | ⏳ | 멀티 런타임 지원 시 검토. 현재 OmO 단일. |
| B2 | **Persona reviewer를 워커 lens로** | compound | ⏳ | 각 L2 워커 안에 2-3 persona. 구현 단계에서 결정 |
| B3 | **`lib/worktree.ts` patch harvest** (SHA256 dedup) | gstack | ⚙️ | `03-l1-foreman-execution.md` §6.2.4에 dedup 룰 명시. 코드 구현 시 적용 |
| B4 | **Wave execution** (pre-commit 1회/wave) | GSD | ✅ | `03-l1-foreman-execution.md` §6.2.2 wave 머지 패턴 |
| B5 | **Skill-TDD** (baseline 실패 보고) | superpowers | ✅ | Agent-Architect의 `step:baseline` 의무 (`04-l2-worker-profiles.md` §2.3.3) |
| B6 | **Decision Coverage Gates** (REQ-ID 미커버 차단) | GSD | ⚙️ | `03-l1-foreman-execution.md` §6.4의 REQUIREMENTS.md/CONTEXT.md에 토대. Maestro 종료 조건에 미완전 통합 |
| B7 | **Lesson 자동 inject (Librarian)** | gstack | ✅ | `04-l2-worker-profiles.md` Librarian Helper 책무 + `06-cross-layer-workflows.md` §4.5 재사용 메커니즘 |
| B8 | **System-level lesson → ETHOS 승급** | gstack | ✅ | `06-cross-layer-workflows.md` §4.5 Tier 3 |

### Skip — 이식 X (정리)

| 출처 | 항목 | 사유 |
|---|---|---|
| superpowers | `using-git-worktrees` skill | Foreman이 이미 처리 |
| superpowers | `finishing-a-development-branch` 4-옵션 | 개인 dev 플로우, 자동 merge에 부적합 |
| superpowers | `dispatching-parallel-agents` skill | Foreman 책임 |
| superpowers | SessionStart 폴리글랏 hook | 워커별 prompt에 직접 inject |
| superpowers | TDD-as-absolute-law (global) | Design/Documentation/Planning Worker 제외 후 선택 적용 |
| compound | 51 agent 평면 구조 | seed worker profiles + 내부 lens로 통합 |
| compound | Rails 전용 reviewer (DHH/Kieran-rails/swift-ios) | 스택 불일치 |
| compound | 멀티 플랫폼 변환기 (.cursor-plugin/.codex-plugin) | OmO + Claude Code만 사용 |
| compound | `ce-strategy` STRATEGY.md | 본 Vision (V-*) entity가 대체 |
| compound | `ce-proof` | upstream bug |
| compound | Every Inc 전용 (figma/demo-reel/riffrec) | 비범용 |
| gstack | `browse` 데몬 (58MB binary, mac-arm64) | Playwright MCP 사용 |
| gstack | GBrain (telemetry-backed memory) | 자체 메모리 레이어 구축 (`.harness/lessons/`) |
| gstack | `/codex` cross-model | OpenAI 의존, Sentinel로 대체 |
| gstack | `conductor.json` 통합 | 별도 paid 도구 |
| gstack | Garry 개인 톤 (`/office-hours`, `/retro`) | 톤 강함 |
| gstack | `bin/` 글로벌 install | 프로젝트 workspace로 통합 |
| GSD | 67개 평면 command | Maestro entry로 통합 (P12) |
| GSD | 15-runtime 변환 레이어 | OmO 1 런타임만 |
| GSD | "autonomous" 1회 직진 | Ralph 4모드가 더 풍부 |
| GSD | `gsd-sdk query` CLI | in-process TypeScript 호출 |
| GSD | 4개로 쪼개진 doc agent | Tech-Writer 1개로 통합 |
| GSD | `nyquist/eval/user-profiler/eval-planner` | niche, Planning+Quality Worker로 흡수 |

---

## 2. 충돌 해결 (소스 간 모순)

| 충돌 | 결정 | 사유 | design 위치 |
|---|---|---|---|
| superpowers "1 task = fresh subagent" vs compound "사이클별 outputs 연속성" | **둘 다 채택**: 워커 *내부*는 fresh subagent (S2), 사이클 *간*은 `.harness/lessons/` + `STATE.md`로 연속성 | 격리와 학습은 다른 layer | `04-l2-worker-profiles.md` §2.3 + `06-cross-layer-workflows.md` §4.5 |
| GSD 67 command vs Maestro entry | **Maestro 유지** (P12) | 단일 entry로 사용자 인지 부담 ↓ | `01-overview.md` P12 |
| compound 51 agent vs worker profiles | **seed worker profiles 유지**, persona는 worker 내부 lens로 흡수 (B2) | 평면 51개는 유지보수 부담 | `04-l2-worker-profiles.md` |
| superpowers TDD-absolute-law vs prose/documentation workers | **선택 적용**: 코드 워커만 ✅, prose 워커 ❌, Agent-Architect ⚙️ skill-TDD | 산출물에 맞는 baseline-fail 검증 | `04-l2-worker-profiles.md` §2.2 TDD 컬럼 |
| GSD "discuss→plan→execute→verify 순차" vs worktree 병렬 | **둘 다 채택**: phase는 직선 통과 (P14), 같은 phase 내 독립 노드는 worktree 병렬 | DAG가 의존성 표현 | `06-cross-layer-workflows.md` §4.3 |
| compound STRATEGY.md (영구 product anchor) vs Vision (V-*) entity | **Vision entity 채택, STRATEGY.md 미채택** | 본 Vision이 더 작은 단위 | `02-l0-maestro-pm.md` §3.2 |
| gstack adaptive gating의 자동 비활성 vs Phase 8 deprecate | **레이어 분리**: gating(Sentinel 룰 내부) vs deprecation(Phase 8 메타) | 같은 발상 다른 시간축 | `05-l1-sentinel-quality.md` §5.7 + `06-cross-layer-workflows.md` §4.6.5 |
| compound `ce-compound`의 자동 트리거 어구 vs 의무화 | **의무화 채택** (P10) | 매 task/phase/cycle 강제 → 우연 트리거 신뢰 X | `01-overview.md` P10 + `05-l1-sentinel-quality.md` `compound_required` |
| GSD adversarial verifier 기본 가정 vs 일반 verifier | **adversarial 채택** | "통과 증거 없으면 실패" | `05-l1-sentinel-quality.md` §5.9 4-level verifier |

---

## 3. 사용자 추가 요구의 외부 패턴 매핑

본 설계의 핵심 원칙 중 일부는 사용자 발화에서 출발했다. 각각이 외부 4 하네스의 어떤 패턴들을 융합한 결과인지 기록.

### 3.1 TDD 강제 (P9)

| 출처 패턴 | 활용 |
|---|---|
| superpowers `test-driven-development` skill | RED-GREEN-REFACTOR 본문 → `tdd-discipline` skill에 차용 |
| superpowers의 "delete pre-test code" 룰 | Sentinel `tdd_violation`의 "기존 테스트 삭제 + 같은 step에서 코드 변경" 룰로 변환 |
| GSD `gsd-verifier` 4-level | TDD의 GREEN 단계 검증에 Level 2(substantive) + Level 3(wired) 결합 |
| gstack `/qa-only` skill | QA 워커가 다른 워커와 동일 7-step, 코드 = 테스트로 해석 |

**TDD 적용 매트릭스**: `04-l2-worker-profiles.md` §2.2 TDD 컬럼 참조.

### 3.2 Compound 3-tier (P10)

| Tier | 시점 | 출처 융합 |
|---|---|---|
| Tier 1 | 워커 task 종료 | superpowers `verification-before-completion` + compound `ce-compound` lightweight |
| Tier 2 | Phase 종료 | compound `ce-compound` Full + `ce-compound-refresh` dedup |
| Tier 3 | Ralph 사이클 종료 | compound system-level + ETHOS 승급 후보 + gstack `/learn` 영구화 |

**재사용 루프**: Librarian 자동 inject = gstack `/learn` + GSD `.planning/` 파일 메모리 결합.

**YAML 스키마**: compound-engineering의 frontmatter 그대로 차용 (`track / category / problem_type / module / tags`).

**자동 트리거 어구 매칭**: compound가 "that worked" 등으로 자동 트리거하지만, **본 설계는 의무화로 대체** (우연 신뢰 X).

### 3.3 하네스 자기 진화 (P11)

| 출처 패턴 | 활용 |
|---|---|
| superpowers `writing-skills` (Skill-TDD: baseline-fail 후 작성) | Agent-Architect `step:baseline` 단계 정의 |
| compound Phase 2.5 discoverability | proposals 채택 후 surface 절차 |
| gstack adaptive specialist gating | 메타-회귀 방지: 채택된 룰이 N 사이클 효과 0이면 deprecate 후보 |
| GSD `gsd-assumptions-analyzer` | implicit assumption 추출 → 진화 trigger 신호 |
| GSD `gsd-codebase-mapper` drift detector | codebase 변화 감지 → 새 Sentinel 룰 후보 신호 |

**진화 사이클**: `06-cross-layer-workflows.md` §4.6.2 6-step (diagnose → baseline → design → propose → peer-review → compound).

**자기 참조**: 6-step 패턴은 워커 내부 step 패턴(`04-l2-worker-profiles.md` §2.3) 그 자체. 하네스가 자기 자신의 패턴을 자기 진화에도 사용.

### 3.4 PM 6-tier + 4-step 사이클 + DoD (P12~P16)

본 추가 요구는 외부 4 하네스에 직접적 출처가 없는 **사용자 자체 입력**. 다만 일부 인스피레이션:

| 본 설계 | 일부 영감 |
|---|---|
| Vision entity | (외부 직접 X — 제품 전략 일반론) |
| Roadmap → Milestone → Backlog 계층 | (Linear/Jira/agile 일반론) |
| 4-step Refinement/Planning/Execution/Review | (agile/scrum 일반론) |
| Definition of Done 의무화 | (agile 일반론) + GSD의 결정 커버리지 게이트 발상 |
| Backlog SSOT (P13) + discovered_not_logged | GSD의 .planning/REQUIREMENTS.md SSOT 패턴 응용 |
| 단일 사용자 창구 (P12) | GSD의 thin orchestrator 패턴 응용 |

→ 즉 P12~P16은 외부 패턴의 직접 이식이 아니라 **사용자 의도의 1차 결정**. application-plan의 §1 tier 매트릭스에 들어가지 않음.

---

## 4. 구현 페이징 (design 완료 → 구현 단계)

design 단계가 v0.5.1으로 마무리되었으므로 이제 **구현 페이즈**로 이동.

### 4.1 design 단계 (완료)

| 페이즈 | 산출물 | 상태 |
|---|---|---|
| design v0.1~v0.3 | 초안 + 원칙 + TDD/Compound/Evolution | ✅ |
| design v0.4 | Maestro 단일 창구 + PM 5엔티티 | ✅ |
| design v0.5 | Vision + 4-step + DoD | ✅ |
| design v0.5.1 | application-plan 누락 5룰 적용 | ✅ |
| application-plan v0.3 | 본 문서 stale 정리 | ✅ (현재) |

### 4.2 구현 단계 (다음)

| 페이즈 | 범위 | 산출물 |
|---|---|---|
| **impl v0.1 (foundation)** | `ETHOS.md` v0 + Maestro 시스템 프롬프트 v0 + `pm-board` skill 본문 + Spec/Report 스키마 | `.harness/ETHOS.md` + `.opencode/agents/maestro.md` + `.opencode/skills/pm-board/SKILL.md` |
| **impl v0.2 (sentinel core)** | Sentinel 룰 → 정적 도구 매핑 + `sentinel-rules` skill 본문 + 4-level verifier 구현 | `.opencode/skills/sentinel-rules/SKILL.md` + 룰별 hook 스크립트 |
| **impl v0.3 (workers)** | 각 L2 에이전트 마크다운 초안 + 워커 7/5/6-step prompt | `.opencode/agents/*.md` 12개 |
| **impl v0.4 (workflow skills)** | `tdd-discipline` + `compound-cycle` + `harness-evolution` + `worktree-orchestrator` + `dag-builder` skill | `.opencode/skills/*/SKILL.md` |
| **impl v0.5 (commands)** | `/board`, `/roadmap`, `/milestone`, `/task`, `/backlog`, `/dod`, `/next`, `/refinement`, `/planning`, `/review`, `/evolve` 등 | `.opencode/command/*.md` |
| **impl v0.6 (profile skills)** | `project-profile` + `convention-registry` + `run-mode` + `lesson-format` + `proposal-format` | `.opencode/skills/*/SKILL.md` |
| **impl v0.7 (hooks)** | `pre-commit-tdd-check`, `post-commit-compound-emit`, `phase-boundary-*`, `worktree-guard`, `pre-tool-backlog-singularity` | `.opencode/hooks/*.sh` |
| **impl v0.8 (e2e dryrun)** | 작은 일반 코딩 프로젝트로 end-to-end dry-run | 검증: 사용자가 Maestro Core와만 대화 / 4-step 직선 통과 / Sentinel 룰 동작 / Phase 8 첫 proposal 생성 |
| **impl v0.9 (B-tier)** | B1 host adapter / B2 persona lens / A10 "Use when ..." 컨벤션 / A12 lesson fingerprint 알고리즘 | 옵션 |

각 impl 페이즈는 **독립 worktree에서 병렬 가능** — 본 하네스를 자기 자신에게 적용해 자기를 개발.

---

## 5. 즉시 다음 액션

순서대로:

1. **본 application-plan v0.3 사용자 컨펌** ← 현재
2. **impl v0.1 진입**: `.harness/ETHOS.md` v0 작성 (P1~P16 본문화)
3. **Maestro 시스템 프롬프트 v0** — 본 하네스의 가장 큰 프롬프트 (단일 창구 + PM 6엔티티 + 4-step driver + DoD 인터뷰어 + 보고서 변환기)
4. **`pm-board` skill 본문** — 6엔티티 CRUD + 우선순위 매트릭스 + 4-step 전이 + DoD 인터뷰 + 중복 감지
5. **Task Spec / Task Report 스키마 동결** — JSON Schema 또는 zod로 검증
6. **Sentinel 룰셋 → 정적 도구 매핑 구체화** — 각 룰을 실제 명령으로 (`05-l1-sentinel-quality.md` §5.5 표를 실행 명령으로)
7. **Foreman의 worktree prohibition hook 의사코드** (`design/03-l1-foreman-execution.md` §6.2.3 실행 코드)
8. **작은 일반 코딩 프로젝트로 end-to-end dry-run** → 회고 → 다음 버전 튜닝
