# Survey: obra/superpowers

> Source: https://github.com/obra/superpowers
> Author: Jesse Vincent (obra)
> Type: Claude Code plugin
> 분석 시점: 2026-05-20

---

## 1. 개요

`superpowers`는 **agent도 command도 없이 오직 14개의 skill + 1개의 hook**만으로 동작하는 Claude Code 플러그인. 임의의 코딩 에이전트(Claude Code, Codex, Gemini, OpenCode, Cursor, Copilot 등)에 일관된 **7단계 SDLC 방법론을 hard-gate로 강제**하는 것이 목적.

핵심 단계 순서:
```
brainstorm → worktree → plan → subagent-driven implement → TDD → review → finish
```

각 단계 사이는 **권고가 아닌 강제 게이트**. 예: "설계가 승인되기 전까지 구현 스킬은 호출될 수 없다", "근본 원인 조사 없이 fix 불가", "fresh verification evidence 없이 완료 선언 불가".

---

## 2. 철학 / 핵심 원칙

| 원칙 | 의미 |
|---|---|
| **Trigger-based loading** | skill은 활성 조건 ("Use when ...")으로 로드되지, 작업 요약으로 로드되지 않는다. 후자는 본문이 무시되는 경향이 있음. |
| **Hard gates between phases** | 단계 간 이동은 명시적 사전조건 충족 필요. 'phase가 advisory가 아니다'를 반복 강조. |
| **Subagent isolation** | 1개의 task는 1개의 fresh subagent (2~5분 분량)로 처리. 절대 같은 subagent에서 implement+review 동시 수행 금지. |
| **Skill-TDD** | skill 자체도 RED-GREEN-REFACTOR로 작성. "스킬을 빼고도 에이전트가 잘 하면 그 스킬은 잘못 쓰여진 것" |
| **No "should work"** | verification 증거 없이 완료 주장 금지. 'should/probably/done' 같은 hedge 단어 차단. |
| **Anti-sycophancy** | 리뷰 피드백 받을 때 "You're absolutely right!" 류 응답 금지. 기술적 재서술 강제. |

---

## 3. 아키텍처 / 구조

```
superpowers/
├── hooks/
│   ├── hooks.json
│   └── run-hook.cmd       (polyglot: Windows batch + bash 한 파일)
└── skills/
    ├── using-superpowers/SKILL.md       (메타 부트스트랩)
    ├── brainstorming/SKILL.md
    ├── writing-plans/SKILL.md
    ├── executing-plans/SKILL.md
    ├── subagent-driven-development/SKILL.md
    ├── dispatching-parallel-agents/SKILL.md
    ├── using-git-worktrees/SKILL.md
    ├── test-driven-development/SKILL.md
    ├── systematic-debugging/SKILL.md
    ├── verification-before-completion/SKILL.md
    ├── requesting-code-review/
    │   ├── SKILL.md
    │   └── code-reviewer.md            (리뷰어 페르소나 템플릿)
    ├── receiving-code-review/SKILL.md
    ├── finishing-a-development-branch/SKILL.md
    └── writing-skills/SKILL.md         (메타: skill 작성법)
```

- agent 디렉토리 없음
- command 디렉토리 없음
- 모든 행동은 skill의 description 매칭으로 트리거

---

## 4. 컴포넌트 카탈로그

### 4.1 Hook (1개)

| Hook | 트리거 | 동작 |
|---|---|---|
| **SessionStart** | session start / clear / compact | `using-superpowers` skill 본문을 매 새 컨텍스트 윈도우에 주입. 폴리글랏 `run-hook.cmd`가 OS 자동 분기. |

### 4.2 Skills (14개)

#### 메타/부트스트랩
| Skill | 책무 |
|---|---|
| **using-superpowers** | 플러그인 사용 진입점. SessionStart로 모든 세션에 주입되어 다른 skill을 어떻게 trigger할지 가이드. |
| **writing-skills** | skill 작성 방법론. description은 무조건 "Use when ..."으로 시작해야 한다는 룰의 본진. |

#### 계획 단계
| Skill | 책무 |
|---|---|
| **brainstorming** | 9단계 설계 게이트. 1 질문 → 답변 → 2-3 alternatives + trade-offs → 명시 승인. 비공식 응답 차단. |
| **writing-plans** | 플랜 파일 작성: `docs/superpowers/plans/YYYY-MM-DD-*.md`. 본질 룰 → ① 2-5분 분량 task로 분해 ② 각 task에 literal code block + file path 포함 ③ placeholder ("TBD", "add validation") 금지 ④ task끼리 코드 의도적 중복 OK (각 task가 fresh subagent로 자족). |
| **executing-plans** | 플랜을 따라 순차 실행. 이탈 시 즉시 plan 갱신 후 진행. |

#### 실행 단계
| Skill | 책무 |
|---|---|
| **subagent-driven-development** | 핵심 패턴. 1 task = 1 fresh subagent. **2단계 리뷰 분리**: spec-compliance review subagent + code-quality review subagent. 절대 한 subagent에서 둘 다 X. 절대 implementer 병렬 X (병렬은 도메인 단위). |
| **dispatching-parallel-agents** | 도메인 독립일 때 1 도메인 = 1 에이전트로 분기. |
| **using-git-worktrees** | git worktree로 분기 환경 격리. 플랜의 어느 단계에 진입하면 자동으로 worktree 생성. |
| **test-driven-development** | RED-GREEN-REFACTOR 강제. RED 단계에서 실패하는 테스트 작성 → 통과시키는 최소 코드 → 리팩터. 기존 코드를 테스트 없이 만들면 안되고, 테스트 없이 만들어진 코드는 삭제 후 재시작. |

#### 디버깅/검증
| Skill | 책무 |
|---|---|
| **systematic-debugging** | 디버깅 절차. 3회 연속 실패 시 즉시 아키텍처 escalation (개별 fix 시도 중단, 상위 설계 의심). |
| **verification-before-completion** | 완료 선언 firewall. "should work", "probably fixed", "done" 류 hedge 단어 사용 시 차단. 실제 verify 명령(테스트/실행/확인 step) + exit code 증거 필수. |

#### 리뷰
| Skill | 책무 |
|---|---|
| **requesting-code-review** | 코드 리뷰 요청. `code-reviewer.md` 페르소나로 별도 subagent dispatch. 출력 포맷: Critical / Important / Minor + Yes/No/With-fixes 판정. |
| **receiving-code-review** | 받은 리뷰 처리. anti-sycophancy: "You're right" 등 즉답 금지. 기술적 재서술 → 동의/반박 명시 → 액션 결정. |

#### 마감
| Skill | 책무 |
|---|---|
| **finishing-a-development-branch** | 브랜치 마감 4-옵션 메뉴: ① merge to main ② squash merge ③ rebase + merge ④ keep open. 사용자 선택 받아 실행. |

---

## 5. 핵심 메커니즘 상세

### 5.1 "Use when ..." 트리거 컨벤션

모든 skill의 YAML frontmatter `description` 필드는 **무조건 "Use when ..."으로 시작**. 예:

```yaml
---
description: Use when starting a new feature and need to design before writing code
---
```

이유: Claude는 description이 작업 요약형이면 skill 본문을 스킵하는 경향이 있음. trigger 조건이 명시되어야 본문을 읽게 됨. `writing-skills` skill이 이 룰의 본진이며 다른 모든 skill이 이를 준수.

### 5.2 2-stage Review (Spec / Quality 분리)

`subagent-driven-development`의 핵심:

```
implementation subagent  →  spec-compliance review subagent  →  code-quality review subagent
                            (변경이 plan/AC를 충족?)              (스타일/복잡도/중복?)
```

- 절대 같은 subagent에서 둘 다 하지 않음
- spec-review subagent는 스타일을 보지 않음 (눈이 흐려짐)
- quality-review subagent는 스펙 충족을 가정하고 시작
- 둘 다 PASS여야 commit 가능

### 5.3 3-failure Architectural Escalation

`systematic-debugging` 룰: 연속 3회 fix 시도가 실패하면 **즉시 멈추고 아키텍처 의심**. 4번째 시도를 하지 않음. 사용자(또는 상위 agent)에게 "이건 아키텍처 문제 같음, 다음 옵션 A/B/C 중 선택" 보고.

근거: AI는 같은 wrong fork에서 무한 시도하는 경향. 명시적 threshold가 없으면 깊이 들어감.

### 5.4 Plan Task Granularity Rule

`writing-plans` 본질:
- 1 task ≈ 2~5분 작업
- 각 task에는 **literal code block** + 정확한 file path
- placeholder/TBD/모호한 표현 ("add validation", "wire it up", "handle errors") **금지**
- task끼리 코드가 중복되어도 OK — 각 task가 fresh subagent에서 자족 가능해야 함

이유: subagent-driven-development에서 plan을 받는 subagent는 prior context가 없음. self-contained 한 단위로 쪼개야 함.

### 5.5 Anti-Sycophancy in Review Reception

`receiving-code-review`는 다음을 명시 금지:
- "You're absolutely right!"
- "Great catch!"
- "Sorry, I'll fix that right away"

대신 강제하는 응답 형태:
```
"Reviewer claim: [기술적 재서술]
My assessment: [agree/disagree, 이유]
Action: [fix / discuss / dismiss]"
```

### 5.6 Skill-TDD

`writing-skills`의 메타 룰: 새 skill을 만들기 전에
1. 해당 skill 없이 agent가 실패하는 baseline 시나리오 실행
2. 실패 양상 기록 (RED)
3. 그 실패만 막을 수 있는 최소 skill 작성 (GREEN)
4. 사용 후 loophole 발견 → skill 보강 (REFACTOR)

skill이 막을 게 없으면 작성하지 않음.

### 5.7 SessionStart Hook 폴리글랏

`run-hook.cmd`는 Windows batch와 bash가 같은 파일에서 양쪽 다 valid한 syntax로 작성됨. OS가 자동으로 자기 인터프리터로 해석. 이로써 cross-platform 단일 파일 hook.

---

## 6. 출력 포맷 디테일

### 6.1 Code Review Output (`code-reviewer.md`)

```
## Critical
- [file:line] [issue]
- [file:line] [issue]

## Important
- ...

## Minor
- ...

## Verdict
- [ ] Yes (ship as-is)
- [ ] No (block)
- [ ] With fixes (Critical/Important addressed first)
```

### 6.2 Plan File

```markdown
# Plan: <feature>

## Task 1: <imperative title>
- File: src/foo/bar.ts
- Action: <verb>
- Code:
  ```ts
  // literal block, complete
  export function ...
  ```

## Task 2: ...
```

---

## 7. 주목할 디테일

- **agent/command 없는 단순성**: 14 skill + 1 hook만으로 7단계 SDLC 강제. 복잡도 vs 효과 비율이 매우 높음.
- **모든 강제는 skill 본문의 prose 룰**: 코드 레벨 차단이 아니라 prompt 룰. 따라서 host에이전트가 룰을 무시할 여지는 있음. trigger 컨벤션과 hard gate prose로 가능한 한 제약.
- **TDD-as-absolute-law 한계**: Designer / Tech-Writer 같은 prose-출력 작업에는 어울리지 않음. 코드 작업 전용.
- **placeholder ban이 가장 효과적**: plan에 "TBD"가 있으면 subagent가 그걸 자기 식으로 채워서 산출물이 발산. literal code block 강제가 이를 근본 차단.

---

## 8. 참조 URL

- README: https://github.com/obra/superpowers
- 핵심 스킬:
  - https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md
  - https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md
  - https://github.com/obra/superpowers/blob/main/skills/verification-before-completion/SKILL.md
  - https://github.com/obra/superpowers/blob/main/skills/systematic-debugging/SKILL.md
  - https://github.com/obra/superpowers/blob/main/skills/requesting-code-review/code-reviewer.md
  - https://github.com/obra/superpowers/blob/main/skills/writing-skills/SKILL.md
- 후크: https://github.com/obra/superpowers/blob/main/hooks/hooks.json
