# VibeForge Harness — Light Design

> 이전 설계(`design/` 8개 파일)는 세부까지 결정해뒀지만 시작 부담이 컸음.
> 이 문서는 **주요 흐름과 구조만** 남긴 경량 버전. 세부는 사용하면서 채운다.

---

## 한 줄

> 사용자는 **Maestro 한 명**과만 대화한다. Maestro는 목표를 **milestone + DoD**로 쪼개고, 각 task를 **Foreman → worker → worktree** 경로로 격리 실행한다. **Sentinel**이 커밋/머지 시점에 가벼운 가드를 걸고, 사이클 끝에 배운 것을 적어 다음 사이클에 주입한다.

---

## 구조 한눈에

```
User ⇄ Maestro  (단일 창구 + 최종 결정자)
          │
          │   필요할 때만 내부 위임:
          │     · 보드 정리 / milestone 계획 / task spec 작성 / 보고 요약
          │
          ├─► Foreman   (task → DAG → worktree 분배)
          │      └─► Workers (역할 프로필; 코드면 TDD)
          │
          └─► Sentinel  (커밋/머지 시 게이트)
```

3개 층:

| 층 | 누구 | 역할 |
|---|---|---|
| L0 | Maestro | 사용자 대화, 결정, PM 잡일 위임 |
| L1 | Foreman / Sentinel | 실행 조율 / 품질 가드 |
| L2 | Workers | worktree에서 task 실행 |

---

## 핵심 원칙 (5)

| # | 원칙 | 의미 |
|---|---|---|
| 1 | **단일 창구** | 사용자는 Maestro와만 대화. 다른 에이전트는 사용자에게 직접 말하지 않음. |
| 2 | **Plan-first / DoD 필수** | 측정 가능한 DoD 없이 milestone 시작/완료 불가. |
| 3 | **Worktree 격리 + TDD** | 동시 작업은 분리된 worktree에서. 코드 작업은 RED→GREEN→REFACTOR. |
| 4 | **Sentinel 게이트** | 매 커밋/머지에 가벼운 룰 검사. PASS / FIX / ASK / BLOCK. |
| 5 | **Compound** | task/사이클 끝에 배운 것 한 줄 캡처 → 다음 사이클에 주입. |

(이 5개는 고정. 그 외는 사용하면서 진화시킴.)

---

## 엔티티 모델 (3단)

```
Goal (Vision)
  └─► Milestone (+ measurable DoD)
        └─► Task  (Foreman으로 dispatch)
```

- **Backlog**는 별도 엔티티가 아니라 *"아직 milestone에 넣지 않은 항목들"* 의 집합.
- 더 자세한 분류(feature / bug / tech_debt / research / spike)는 필요해질 때 추가.

---

## 사이클 (milestone 단위)

```
  ┌──────────────┐
  │ Plan         │ ← milestone 선정 + DoD + task 분해
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Execute      │ ← Foreman → workers → worktree → commit
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Review       │ ← DoD 검증, 실패는 backlog로
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Learn        │ ← lessons 캡처 → 다음 Plan에 주입
  └──────┬───────┘
         │
         └──► 다음 milestone Plan
```

> 기존 4-step(Refinement / Planning / Execution / Review) 중 Refinement는 Plan에 흡수,
> Compound는 Learn으로 분리. 필요해지면 다시 쪼개도 됨.

---

## 워커 흐름 (얇게)

| 작업 종류 | 흐름 |
|---|---|
| **코드** | plan → red → green → refactor → commit → lesson |
| **문서/스펙** | draft → review → revise → lesson |

세부 step(spec-review, quality-review, self-check block 등)은 필요해진 시점에 worker profile에 추가.

---

## Sentinel (가벼운 가드)

작동 시점:
- 매 커밋 후
- 머지 직전
- milestone Review (DoD 검증 시)

기본 룰셋 (시작용):

| 룰 | 무엇을 막나 |
|---|---|
| `tdd` | 코드 변경에 테스트가 없음 |
| `scope` | task plan에 없는 파일/경로 변경 |
| `backlog` | 코드 주석 TODO / 흩어진 todo 노트 |
| `dod` | 측정 불가능한 DoD |
| `lesson` | 사이클 끝에 lesson 없음 |

결과: `PASS` / `FIX`(자동 수정) / `ASK`(Maestro에게 묻기) / `BLOCK`(중단).

> 14개 룰, 4-level verifier, 2-tier 비용 분리 등은 *"사용하다가 정말 필요해지면"* 추가.

---

## 진화

- 누적 데이터(반복되는 Sentinel 발견, 반복되는 lesson, 사용자 선호)는 **Agent Architect**가 제안으로 변환.
- 사용자 수락 → 워커 프로필 / 컨벤션 / 룰 추가·수정.
- **5개 핵심 원칙은 고정**, 나머지는 진화.

---

## 런타임 핵심 파일

```
.harness/
  STATE.md         # 현재 milestone / phase / 활성 worker
  PROJECT.md       # 언어, 빌드/테스트 명령, repo 레이아웃
  CONVENTIONS.md   # 누적 컨벤션
  board/
    goal.md
    milestones/M-*.md
    tasks/T-*.md
  lessons/         # 누적 학습
.worktrees/        # 실행 격리
```

> 기존 설계의 `phases/`, `proposals/_pending/`, `lessons/{architecture,bugs,perf,…}` 등 하위 분류는 모두 생략.
> 폴더가 커지면 그때 쪼갠다.

---

## 사용자 명령 (최소)

```
/start "<goal>"            # 부트스트랩 (PROJECT.md, goal, 첫 milestone)
/plan | /exec | /review    # 사이클 단계 수동 진행
/board | /next             # 현재 상태 / 다음 권장 행동
/lesson                    # 수동 lesson 캡처
/evolve                    # 누적 제안 보기/수락
```

> `/refinement /planning /execution /review /dod /vision /roadmap /milestone /backlog /issue /task /report /sentinel-check /tdd-exception /compound-now /lessons /proposals /mode /pause /resume /stop` 등 20+ 명령은 *필요해질 때만* 추가.

---

## 다음에 결정할 것 (보류)

지금은 결정하지 않고 사용하면서 정한다:

- run mode 종류 (auto / gated / plan-only / dry-run)
- worker profile seed 8종 분할 여부
- DoD verify 명령 schema
- worktree 동시 실행 상한
- 보드 git 커밋 정책
- Sentinel 룰 hit-rate 기반 자동 demote

---

## 참고

이전의 무거운 버전은 `drafts/design/` 에 그대로 보존되어 있고, 더 깊은 결정이 필요해진 항목은 거기서 가져와 채워넣으면 된다.
