# 종합 평가: 코딩 하네스 설계 (v0.5.1)

> 대상: `drafts/my-harness-design.md` + `drafts/design/01~08-*.md` + `drafts/CHANGELOG.md` 전체
> 일자: 2026-05-20

## 한 줄 평
**야심차고 내부 정합성이 높지만, 복잡도 / 비용 / 부트스트랩 부담이 큰 설계.** S-tier 패턴은 잘 흡수했고, 자기진화 메커니즘까지 갖췄지만 **개방 결정 ~20건이 잠긴 다음에야 구현으로 들어가야 안전.**

---

## 강점 (Strengths)

### 1. **4-level Verifier (§5.9)** — 가장 차별적
EXISTS → SUBSTANTIVE → WIRED → REAL DATA FLOW. 대부분 하네스가 Level 1~2에서 끝나는데, 이 설계는 Level 3 (마운트/라우터 등재)과 Level 4 (실데이터 흐름)까지 강제. **"통과 증거 없으면 실패"** 라는 adversarial 기본가정도 정확.

### 2. **Backlog SSOT (P5)** — AI 에이전트 고질병 정통 타격
코드 주석 TODO 단독 금지 + 모든 todo는 `B-*.md`로. `discovered_not_logged` 룰까지 P0로. AI가 "추후", "TODO:" 남기고 빠지는 패턴을 시스템 차원에서 차단하는 게 정확한 접근.

### 3. **DoD 의무 + verify_cmd (P7)**
`measurable: true` 강제 + 측정 불가 표현 ("대충", "잘 되어야") BLOCK. "만들었다 ≠ 검증되었다"를 시스템 차원에서 강제.

### 4. **Maestro 단일 창구 (P2)**
사용자 인지부담 일원화. Foreman/Sentinel/L2의 stdout이 시스템 차원에서 user-facing 채널 X로 격리. 컨텍스트 격리 3-레벨 (Maestro/Foreman/Worker step) 설계도 깔끔.

### 5. **Worktree prohibition + STATE.md O_EXCL lock**
GSD에서 잘 가져옴. `git stash`가 worktree 간 silent leak되는 함정을 명시적으로 차단한 건 실무 경험 기반의 깊이 있는 디테일.

### 6. **Phase 8 진화 + 메타-회귀 방지 (§4.6.5)**
`evolved: true` 태그 + 3-cycle 효과 측정 후 영구화 + Oracle peer-review. 자기진화 하네스의 *자기드리프트* 위험을 인지하고 가드 둔 점이 성숙함.

### 7. **2-tier Sentinel 비용 제어**
1차 cheap (gpt-5.4-mini, 정적도구) + 2차 ultrabrain (의심 시만). 매 커밋 감사가 경제적으로 작동 가능한 구조.

### 8. **Compound 3-tier + fingerprint-merge**
Tier 1(task) / Tier 2(phase, Tech-Writer Lead) / Tier 3(cycle, cross-cutting pattern). lessons가 생성만 되는 게 아니라 **Librarian이 inject로 소비**하는 closed-loop도 갖춤.

---

## 약점 / 리스크

### 🔴 Critical — 진행 전에 풀어야 할 것

**1. Maestro가 단일 병목 + 과부하 (P2의 그늘)**
- 사용자 대화 + 6-tier CRUD + 4-step driver + DoD 인터뷰 + Task Spec 작성 + 보고서 가공
- 프롬프트 응집도 ↑ 라기엔 *Maestro 한 프롬프트에 5개 책무*. §8.4 "Maestro 자체 컨텍스트 압축" 으로 자인.
- **권고**: `_index.md`만 컨텍스트에 두고 본문은 on-demand read 강제하는 룰을 P2 강제 메커니즘으로 추가. Maestro의 task() sub-call로 PRD 작성/보고서 가공 등을 분리하는 게 안전.

**2. 개방 결정 ~20건이 implementation-blocking (§8.4)**
- DoD verify_cmd 표준 셰이프, DoD 항목 수 권장, 다중 milestone 동시 진행, Vision 변경 정책, ETHOS 영구화 게이트, lesson fingerprint 알고리즘 등 *critical*.
- 이걸 안 잠그고 구현 들어가면 각 skill 작성자가 자기 멋대로 해석 → 분기 발생.
- **권고**: §8.4 모든 항목에 default 결정 추가 후 v0.6으로 동결.

**3. Sentinel 룰 60+ 개의 정책 충돌**
- 같은 commit에 `plan_placeholder` + `discovered_not_logged` + `tdd_violation` + `backlog_singularity` 동시 BLOCK 가능 → 어디부터 풀지 모르는 deadlock.
- fingerprint-merge는 *동일 finding* dedup만 함. 정책 우선순위 (어떤 룰 먼저 해결?) 미정의.
- **권고**: BLOCK이 multi-rule일 때 *해결 순서 메타룰* 추가 (예: TDD → backlog → plan_placeholder → scope_creep).

### 🟡 Mid — 구현하면서 잡아도 되지만 인지 필요

**4. 7/5/6-step 워커 비대칭**
- 코드(7) / Prose(5) / Agent-Architect(6) 셋을 따로 유지. 장기 유지보수 비용.
- **권고**: 5-step universal (research-or-plan / draft / peer-review / commit-or-revise / compound) + TDD/baseline을 *플러그*로 표현. 비즈니스 의미는 보존.

**5. 비용·예산 모델 부재**
- 매 commit Sentinel 2-tier + 고비용 모델 다수(Maestro Core/Planning/Data/Security/Agent-Architect) + 병렬 worktree 5+.
- 한 milestone당 토큰/$ 예상치 없음. `max_global_iterations: 50`만 있고 비용 게이트 X.
- **권고**: cycle-level budget 룰을 Sentinel `[NEVER_GATE]` 옆에 추가. "milestone당 X 토큰 초과 시 HITL escalate".

**6. 동시성 상한 없음**
- DAG 빌더가 wide면 worktree 5~20개 가능. 디스크/CI/머지 충돌 비용 폭증.
- **권고**: `max_concurrent_worktrees` 명시 (예: 4). DAG가 wider면 wave 직렬화.

**7. Tier 2 compound 시점 모호**
- "phase 종료 시"인데 phase는 milestone 단위 4-step. workers가 parallel하게 commit하면 "어디가 phase 종료?"가 ambiguous.
- **권고**: Tier 2를 "milestone Execution 종료"로 명시 정상화 (사실상 Tier 2 = milestone-mid, Tier 3 = milestone-end).

**8. backlog_singularity가 너무 strict**
- 코드 주석 TODO 단독 금지지만, 정당한 use case 있음 (해당 라인에서만 의미 있는 알림). 모두 B-* 만들면 backlog 노이즈 폭발.
- **권고**: `TODO(B-099): ...` 패턴은 허용 (backlog 참조 있으면 OK). Sentinel 정규식 한 줄 추가로 해결.

**9. UI 계열 worker의 TDD 부적합**
- Visual exploration은 RED→GREEN보다 "show then test interactions"이 더 자연스러움. `tdd_exception`은 무거움.
- **권고**: UI 계열 worker는 *visual snapshot test*를 default RED로 인정 (Playwright screenshot diff 등).

### 🟢 Minor

- **worker profile 과다**: Implementation/Data/Ops 분리 기준이 흐리면 skill switching 비용이 커짐. 실제 hit-rate를 보고 병합/분리 권장.
- **ASK 배치 protocol 미정의**: per-task / per-phase 어느 단위? stale ASK 방지책 없음.
- **Observability 부재**: `progress.jsonl` / `sentinel-log.jsonl`만으로 운영 통찰 부족. 룰 hit-rate 대시보드 필요.
- **Issue 흡수 UX 비용**: severity/reproduction은 `tags`로 표현하지만 bug-specific 필드 schema 권장.
- **Maestro 자체 hallucination**: 존재 안 하는 B-ID/T-ID 참조 시 fallback 미정의. Sentinel이 board 정합성 룰 추가 필요.

---

## 외부 패턴 흡수도

| 패턴 | 흡수 정도 |
|---|---|
| superpowers (TDD, 2-stage review, brainstorming-9, harness evolution) | ✅ 충실 |
| GSD (4-level verifier, slopcheck, self-check, analysis-paralysis, STATE lock, prohibition layer) | ✅ 충실 (v0.5.1에서 보강) |
| compound-engineering (3-tier compound, U-ID DAG, fingerprint-merge, AUTO-FIX/ASK) | ✅ 충실 |
| gstack (ETHOS preamble, 카테고리 기반 lesson) | ✅ 흡수 |

**S-tier 13개 전부 매핑 완료**된 건 강력. 다만 *흡수 vs 통합*은 다름 — 다음 단계에서 같은 finding을 다른 패턴이 다른 액션으로 처리할 때 우선순위 결정 필요.

---

## 권고 우선순위

**v0.6에서 반드시:**
1. §8.4 개방결정 ~20건에 default 박기
2. Sentinel 다중-BLOCK 시 해결 순서 메타룰 추가
3. Maestro 컨텍스트 압축 룰 (`_index.md` only) 강제화
4. Budget 게이트 (token/$/cycle) 추가
5. `max_concurrent_worktrees` 명시

**v0.7+에서:**
6. 7/5/6-step → 5-step universal + plug 통합
7. L2 워커 13 → 8~9 슬림화
8. UI 계열 visual-snapshot TDD 인정
9. `TODO(B-id)` 허용 패턴 + Sentinel 룰 완화
10. Observability layer (cycle 메트릭 마크다운 자동생성)

**실험으로 검증할 것:**
- §8.3 dry-run — 이걸 안 돌려보고 implementation 들어가면 진짜 큰 위험. **여기가 진실의 순간**. 작은 일반 코딩 프로젝트의 첫 3 milestone을 dry-run에서 돌려보고 Maestro 컨텍스트 폭증 / Sentinel BLOCK 충돌 / DoD verify 비용 등 실측 후 v0.6 결정 잠그기를 강력 권고.
