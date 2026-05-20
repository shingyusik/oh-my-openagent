# Survey: EveryInc/compound-engineering-plugin

> Source: https://github.com/EveryInc/compound-engineering-plugin
> Author: Every Inc / Kieran Klaassen
> Type: Claude Code plugin (멀티 런타임 컨버터 포함)
> 분석 시점: 2026-05-20

---

## 1. 개요

`compound-engineering-plugin`은 Every Inc가 popularize한 "compound engineering" 방법론을 Claude Code 위에 운영화한 대형 플러그인. **51개 agent + 37개 skill**로 구성되며, 핵심 사상은:

> "모든 작업 단위는 다음 단위를 더 쉽게 만들어야 한다."

이를 위해 매 사이클의 학습을 `docs/solutions/` 디렉토리에 쿼리 가능한 형태로 codify하고, 이후 작업이 자동으로 이를 검색하도록 한다.

5단계 컴파운딩 루프:
```
brainstorm  →  plan  →  work  →  review  →  compound
   ↑                                            ↓
   └────────── outputs become next inputs ──────┘
```

---

## 2. 철학 / 핵심 원칙

| 원칙 | 의미 |
|---|---|
| **Compounding** | 한 사이클의 출력(requirements, plan unit, commits, findings)이 다음 사이클의 구조화된 입력이 된다. 매번 처음부터 시작하지 않는다. |
| **80/20 inversion** | 80% planning + review, 20% execution. 실행은 자동, 사람과 모델의 노력은 fork 시점에 집중. |
| **Stable Implementation Units (U-ID)** | plan에서 부여된 `U1, U2, ...` ID는 절대 renumber되지 않으며 task tracker, commit message, reviewer finding, PR title을 관통한다. |
| **Bounded parallel reviewers** | 여러 리뷰어를 병렬 실행하되 fingerprint 기준으로 dedup + confidence 승격. 무한 reviewer는 X. |
| **Knowledge as a filesystem** | 학습된 패턴/해결책은 `docs/solutions/<category>/<slug>-<date>.md`로 영구 저장. YAML frontmatter로 쿼리 가능. |
| **Discoverability check** | 새 솔루션이 작성되면 AGENTS.md/CLAUDE.md가 이를 surface하는지 검증, 안 되면 자동 패치. |

---

## 3. 아키텍처 / 구조

```
plugins/compound-engineering/
├── plugin.json                  (메타데이터; hook 디렉토리 없음)
├── AGENTS.md                    (38KB; skill 작성 원칙 + load-bearing 룰)
├── skills/                      (37개 ce-* 스킬)
│   ├── ce-strategy/SKILL.md
│   ├── ce-brainstorm/SKILL.md
│   ├── ce-plan/SKILL.md
│   ├── ce-work/SKILL.md         (27KB; 4-phase executor)
│   ├── ce-code-review/SKILL.md  (89KB; 멀티 리뷰어 통합)
│   ├── ce-compound/SKILL.md     (37KB; 학습 코디파이어)
│   ├── ce-compound-refresh/SKILL.md
│   ├── ce-worktree/SKILL.md
│   ├── ce-commit, ce-commit-push-pr, ce-resolve-pr-feedback, ce-clean-gone-branches
│   ├── ce-debug, ce-simplify-code, ce-optimize, ce-proof, ce-doc-review
│   ├── ce-frontend-design, ce-sessions, ce-setup, ce-update, ...
├── agents/                      (51개; 카테고리별 그룹)
│   ├── always-on/
│   ├── cross-cutting/
│   ├── personas/                (DHH, Kieran, Julik 등 named lens)
│   ├── sentinels/
│   ├── researchers/
│   └── lenses/
├── docs/                        (예시 솔루션 + 가이드)
└── .cursor-plugin/, .codex-plugin/, src/ (멀티 플랫폼 변환기)
```

- **hook 없음** (manifest only). 자동 트리거는 모두 skill description 매칭으로 처리.
- **command 따로 없음**. Skill에 `ce-` prefix → Claude Code가 `/ce-*`로 surface.
- prefix 선택 이유: 내장 `/plan`, `/review`와 충돌 회피.

---

## 4. 컴포넌트 카탈로그

### 4.1 Skills (37개, 카테고리별)

#### 전략·계획 (5)
| Skill | 책무 |
|---|---|
| **ce-strategy** | `STRATEGY.md`를 product anchor로 유지. 전반 의사결정의 ground truth. |
| **ce-brainstorm** | 인터랙티브 요구사항 수집. R/A/F/AE ID 부여 (Requirements/Assumptions/Features/Acceptance Evidence). |
| **ce-plan** | `docs/plans/YYYY-MM-DD-NNN-<type>-<name>-plan.md` 생성. U-ID 부여, depth-stratified (light/standard/deep). |
| **ce-roadmap** | 멀티 plan을 roadmap에 묶음. |
| **ce-sessions** | 과거 세션 retrieval. |

#### 실행 (3)
| Skill | 책무 |
|---|---|
| **ce-work** | 4-phase executor: dispatch / monitor / collect / synthesize. inline / serial / parallel worktree 모드 자동 선택. |
| **ce-debug** | 디버깅 사이클. ce-compound와 연계되어 디버그 학습 자동 저장. |
| **ce-simplify-code** | 단순화 패스. 사용처 1곳뿐인 추상화 제거 등. |

#### 리뷰 (1 + 다수 agent)
| Skill | 책무 |
|---|---|
| **ce-code-review** (89KB) | **이 플러그인의 marquee**. 다중 리뷰 agent를 병렬 dispatch + fingerprint-merge + confidence 승격 + 모드별 demotion. |

#### 학습 코디피케이션 (2)
| Skill | 책무 |
|---|---|
| **ce-compound** (37KB) | 해결된 문제를 `docs/solutions/<category>/<slug>-<date>.md`로 저장. **Full vs Lightweight** 모드. "that worked", "it's fixed" 등 자동 트리거 어구 매칭. **Phase 2.5 discoverability**: AGENTS.md가 새 솔루션을 가리키는지 확인 후 패치. |
| **ce-compound-refresh** | 기존 solution docs를 dedup/merge. |

#### Git 워크플로 (5)
| Skill | 책무 |
|---|---|
| **ce-commit** | 단일 commit 생성. 컨벤셔널 commit 강제. |
| **ce-commit-push-pr** | commit + push + PR open. |
| **ce-resolve-pr-feedback** | 리뷰 피드백 받아 자동 처리. |
| **ce-clean-gone-branches** | 머지된 원격 브랜치 로컬 정리. |
| **ce-worktree** | `.worktrees/<branch>` 생성 + env 복사. branch-aware trust: trusted base branches는 direnv 자동 승인, feature branch는 prompt. |

#### 기타
- **ce-optimize**, **ce-proof**(upstream bug 존재), **ce-doc-review**, **ce-frontend-design**, **ce-setup**, **ce-update** 등.
- 일부 Every Inc 전용: `ce-figma-design-sync`, `ce-demo-reel`, `ce-gemini-imagegen`, `ce-riffrec-feedback-analysis`.

### 4.2 Agents (51개)

#### Always-on reviewers (6)
정상 동작에서 거의 항상 실행됨:
- `correctness-reviewer`
- `testing-reviewer`
- `maintainability-reviewer`
- `project-standards-reviewer`
- `agent-native-reviewer` (LLM 친화도)
- `learnings-researcher` (compound 학습 활용 점검)

#### Cross-cutting (조건부, 8)
변경 종류에 따라 선택적으로 dispatch:
- `security-reviewer`
- `performance-reviewer`
- `api-contract-reviewer`
- `data-migrations-reviewer`
- `reliability-reviewer`
- `adversarial-reviewer` (red team)
- `previous-comments-reviewer` (과거 리뷰 일관성)
- `scope-guardian` (PR scope creep 감지)

#### Stack personas (named lens, 5)
- `dhh-rails-reviewer` (DHH 스타일 Rails)
- `kieran-rails`, `kieran-python`, `kieran-typescript`
- `julik-frontend-races` (frontend race condition)
- `swift-ios`

#### Sentinels (5)
지속적 모니터링:
- `security-sentinel`
- `data-integrity-guardian`
- `schema-drift-detector`
- `deployment-verification`
- `performance-oracle`

#### Researchers (8)
정보 수집:
- `web-researcher`, `slack-researcher`, `framework-docs-researcher`
- `repo-research`, `best-practices-researcher`
- `git-history-analyzer`, `pattern-recognition-researcher`, `issue-intelligence-researcher`

#### Lens reviewers (6)
관점별 리뷰:
- `design-lens`, `product-lens`, `security-lens`, `coherence-lens`, `feasibility-lens`, `code-simplicity-lens`

#### 기타 (~13개)
- 모델/플랫폼별 특화 페르소나
- `update-orchestrator`, `migration-runner` 류

### 4.3 Hooks

**plugin manifest에 hooks 디렉토리 없음.** 자동 트리거는 100% skill description 매칭으로 처리. 이는 Claude Code의 trigger-based skill loading에 의존.

### 4.4 Commands

별도 command 디렉토리 없음. Skill의 `ce-` prefix가 Claude Code의 슬래시 surface로 그대로 노출.

---

## 5. 핵심 메커니즘 상세

### 5.1 Stable U-ID (Implementation Unit ID)

`ce-plan`이 부여:
```markdown
## U1: Add user authentication
**Acceptance**: R1, R3
**Estimated**: 4h

## U2: Wire login form to U1
**Depends on**: U1
**Acceptance**: R2

## U3: ...
```

이 ID는:
- **renumber 절대 X** (U2 삭제되어도 U3는 U3 유지)
- task tracker entry에 그대로 사용
- commit message prefix: `[U1] add JWT validation`
- reviewer finding 메타에 포함: `finding.unit = "U1"`
- PR title 사용: `[U1, U2] Add authentication`

결과: 모든 산출물이 단일 spine으로 cross-reference 가능.

### 5.2 Fingerprint-Merge + Confidence Anchor

`ce-code-review`의 핵심 알고리즘. 여러 리뷰어 agent를 병렬 실행하면 finding이 폭증하므로 dedup 필요:

```
finding_fingerprint = (
  normalize(file_path),
  bucket(line_number, ±3),      // 인접 3줄은 같은 finding으로 본다
  normalize(title)               // 공백/언어변형 무시
)
```

같은 fingerprint를 가진 finding이 N개 리뷰어에서 나오면:
| N | confidence |
|---|---|
| 1 | 50 |
| 2 | 75 |
| 3+ | 100 |

추가 룰:
- **P0(severity critical) + 2-reviewer agreement**: confidence demotion에서 면제
- **모드 인식 demotion**: headless mode에서는 P2/P3 advisory는 demote, P0+는 유지

### 5.3 ce-compound: Knowledge Compounding

5단계 컴파운딩 루프의 마지막 단계. 자동 트리거 어구 ("that worked", "it's fixed", "good") 감지 시 발동.

산출물 형식:
```markdown
---
track: <project|cross-project>
category: <bugs|architecture|perf|ops|patterns>
problem_type: <symptom 슬러그>
module: <영향 모듈>
tags: [tag1, tag2]
related_units: [U3, U7]
created: 2026-05-20
---

# <Problem Title>

## Symptom
<observed behavior>

## Root cause
<analysis>

## Solution
<what worked, with code>

## Why this matters
<future prevention angle>
```

저장 위치: `docs/solutions/<category>/<slug>-<date>.md`

**Full vs Lightweight 모드**:
- **Full**: 발견된 문제가 cross-cutting (다른 module에서 재현 가능) → 전체 schema
- **Lightweight**: localized → title + 1-paragraph note만

### 5.4 Phase 2.5: Discoverability Check

`ce-compound`의 sub-phase. 솔루션을 작성한 후:
1. 프로젝트의 AGENTS.md/CLAUDE.md를 읽음
2. 새 솔루션 카테고리가 인덱싱되어 있는지 확인
3. 없으면 자동으로 인덱스 라인 추가:
   ```markdown
   ## Solutions
   - Bugs: see docs/solutions/bugs/
   - Architecture: see docs/solutions/architecture/
   - ...
   ```

이유: "솔루션은 작성되었지만 미래의 agent가 못 찾는" 문제 해결.

### 5.5 ce-work: 4-Phase Executor

```
phase 1: dispatch   — plan을 읽어 unit별 작업 분배 (inline/serial/parallel worktree 결정)
phase 2: monitor    — 각 unit 진행 추적
phase 3: collect    — 결과 회수
phase 4: synthesize — 최종 PR/요약 생성
```

dispatch 결정 휴리스틱:
- file 수 ≤2 + 같은 모듈 → **inline** (현재 컨텍스트에서)
- depends on chain 존재 → **serial** (순서대로)
- 독립 unit ≥2 → **parallel worktree** (각 unit 1 worktree)

### 5.6 Branch-Aware Worktree Trust

`ce-worktree`의 특징:
```
if base_branch in TRUSTED_BASES:
  auto_approve_direnv()
else:
  prompt_user_for_direnv()
```

이유: feature branch는 untrusted code 포함 가능 → direnv 자동 승인 시 위험. main/dev 등 trusted base는 자동.

### 5.7 Persona Reviewers

named opinion vectors. 단순 "reviewer 1, reviewer 2" 대신 캐릭터를 부여:

- **dhh-rails-reviewer**: DHH 톤 — convention over configuration, fat models thin controllers
- **kieran-typescript**: Kieran 톤 — strict types, narrow returns, no any
- **julik-frontend-races**: Julik 톤 — race condition / concurrent UI 관점 집착

이유: generic "code reviewer"보다 narrowed lens가 더 specific finding 생산. 같은 코드를 다른 페르소나가 보면 다른 종류의 issue가 잡힘.

---

## 6. 출력 포맷 디테일

### 6.1 Plan File (ce-plan)

```markdown
# Plan: <feature>

## Metadata
- Date: 2026-05-20
- Number: 042
- Type: feature
- Depth: standard

## Requirements
- R1: ...
- R2: ...

## Implementation Units

### U1: <title>
- Depends on: -
- Acceptance: R1
- Files: [src/foo.ts, src/bar.ts]
- Code:
  ```ts
  // literal
  ```

### U2: <title>
- Depends on: U1
- ...
```

### 6.2 Review Output (ce-code-review)

```
## Findings (merged)

### High (confidence 100)
- [U2] [src/auth.ts:42] missing input validation
  Sources: correctness, security, kieran-typescript

### Medium (confidence 75)
- [U1] [src/foo.ts:18] unclear naming
  Sources: maintainability, dhh-rails

### Low (confidence 50)
- ...

## Verdict
- Spec compliance: PASS (with fixes on U2)
- Quality: PASS
- Ship: With Fixes
```

---

## 7. 주목할 디테일

- **51개 agent의 평면 디렉토리 구조**: 스케일 시 검색/유지보수 부담. 카테고리 폴더로 정리되지만 여전히 큰 표면.
- **89KB짜리 단일 SKILL.md (ce-code-review)**: skill을 매우 깊게 작성. Claude Code skill loading의 context impact가 큼.
- **자동 트리거 어구 매칭**: "that worked" 등 사용자 발화를 패턴 매칭해서 ce-compound 자동 실행. 일반적인 skill 트리거(intent matching)와 별개로 phrase 매칭 사용.
- **멀티 런타임 변환기 (`.cursor-plugin/`, `.codex-plugin/`)**: src/에 TypeScript 트랜스파일러. 단일 source를 여러 런타임에 호환되게 변환.
- **AGENTS.md (38KB)** 자체가 skill 작성 가이드를 담음. "calibrate prescription to failure mode", "extract load-bearing rules to top of SKILL.md" 같은 메타-skill design 원칙.
- **Rails 편향**: persona reviewer 다수가 Rails/Ruby. Every Inc 스택이 Rails라서.

---

## 8. 참조 URL

- README: https://github.com/EveryInc/compound-engineering-plugin
- 핵심 스킬:
  - `plugins/compound-engineering/skills/ce-compound/SKILL.md` (37KB)
  - `plugins/compound-engineering/skills/ce-code-review/SKILL.md` (89KB)
  - `plugins/compound-engineering/skills/ce-plan/SKILL.md`
  - `plugins/compound-engineering/skills/ce-work/SKILL.md` (27KB)
- 메타: `plugins/compound-engineering/AGENTS.md` (38KB; skill design 원칙)
- 외부 글:
  - Every Inc 블로그에서 "Compound Engineering" 키워드 검색 (Kieran Klaassen 저자)
