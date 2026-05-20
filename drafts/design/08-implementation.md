# 08. 구현 매핑 + 다음 액션 + 열린 사항

> 본 하네스를 `oh-my-openagent` 위에 어떻게 얹을 것인가의 구현 매핑, 우선순위, 열린 결정 항목.

---

## 8.1 oh-my-openagent 매핑

| 본 설계 | OmO 메커니즘 | 구현 방법 |
|---|---|---|
| **Maestro** | Prometheus 확장 + `pm-board` skill | `.opencode/agents/maestro.md` + `agents.prometheus.prompt_append` |
| **Foreman** | Atlas 확장 또는 신규 primary | `.opencode/agents/foreman.md` (Atlas 패턴 차용) |
| **Sentinel** | 신규 + post-commit hook | `.opencode/agents/sentinel.md` + `.opencode/hooks/post-commit-*.sh` |
| **L2 워커** (10직책) | Custom Agent + Category | `.opencode/agents/<role>.md` + `categories` 항목 |
| **워커 내부 7/5/6-step** | OmO `task()` 도구 별도 세션 | 워커 prompt가 task()로 자기 sub-step 호출 |
| **Worktree 병렬** | git worktree + OmO background task | Foreman이 bash로 worktree 생성, 각 worktree에서 OmO 세션 |
| **Ralph 루프** | 기존 `/ulw-loop` 패턴 | Maestro가 milestone 단위 자체 루프 |
| **자료조사** | Librarian + Explore + Oracle | 그대로 |
| **컨텍스트 격리** | task() 별도 세션 + 구조화 출력 | 프롬프트로 강제 |
| **코드 품질 룰** | Sentinel 프롬프트 + Skill | `.opencode/skills/sentinel-rules/SKILL.md` |
| **PM 6엔티티** | `pm-board` skill + `board/` 마크다운 | Maestro가 직접 읽고 씀 |
| **DoD 검증** | `pm-board` skill의 verify_cmd 실행 | bash 호출 + exit code 해석 |
| **Compound 3-tier** | `compound-cycle` skill + 별도 task() 분리 | Tier 1: 워커 step, Tier 2: Tech-Writer task(), Tier 3: Tech-Writer + Agent-Architect 합작 |
| **Phase 8 진화** | `harness-evolution` skill | Agent-Architect의 별도 task() 세션 |
| **STATE.md lock** | flock 기반 bash | `.opencode/hooks/state-lock.sh` 유틸리티 |
| **worktree prohibition** | PreToolUse hook | `.opencode/hooks/worktree-guard.sh` |

---

## 8.2 외부 하네스 패턴 통합

[`application-plan.md`](../application-plan.md) 참조. 핵심 통합 사항:

### S-tier (반드시 이식)

| ID | 패턴 | 출처 | 본 설계 위치 |
|---|---|---|---|
| S1 | Stable U-ID (DAG 노드 추적) | compound-engineering | Foreman DAG |
| S2 | 2-stage 리뷰 (spec/quality 별도 task()) | superpowers | 워커 7-step |
| S3 | Worktree prohibition layer | GSD | Foreman dispatcher hook |
| S4 | STATE.md O_EXCL lock | GSD | `.harness/STATE.md` |
| S5 | Analysis-Paralysis Guard | GSD | `05-sentinel.md` §5.3 `analysis_paralysis` 룰 (P0) |
| S6 | 4-level verifier (exists→substantive→wired→data flow) | GSD | `05-sentinel.md` §5.9 (DoD verify + worker spec-review에서 사용) |
| S7 | Slopcheck (패키지 정합성) | GSD | `05-sentinel.md` §5.3 `slopcheck` 룰 (P0 [NEVER_GATE]) |
| S8 | Self-Check before completion | GSD | `05-sentinel.md` §5.3 `self_check_required` + `02-agents.md` §2.3.1 commit step |
| S9 | Knowledge compounding (`docs/solutions/`) | compound-engineering | Tier 2/3 compound |
| S10 | Fingerprint-merge + confidence anchor | compound-engineering | Sentinel 룰 통합 |
| S11 | TDD-mandatory RED→GREEN→REFACTOR | superpowers | 워커 7-step |
| S12 | 3-tier Compound | compound + gstack | Tier 1/2/3 |
| S13 | Harness Evolution (Phase 8) | superpowers + compound + gstack | Agent-Architect |

### A-tier (강력 추천)

| ID | 패턴 | 본 설계 위치 |
|---|---|---|
| A1 | Plan placeholder 금지 validator | `05-sentinel.md` §5.3 `plan_placeholder` 룰 + `02-agents.md` Strategist 책무 |
| A2 | 3-failure 아키텍처 escalation | Ralph 루프 circuit-breaker |
| A3 | Brainstorming 9-step | Maestro Phase 0 |
| A4 | Adaptive specialist gating | Sentinel hit-rate 추적 |
| A5 | AUTO-FIX vs ASK | Sentinel 출력 분류 |
| A6 | `.planning/` 파일 메모리 | `.harness/` 구조 |
| A7 | Phase 2.5 discoverability | Agent-Architect 정기 작업 |
| A8 | ETHOS preamble injection | `.harness/ETHOS.md` |
| A9 | `/learn` typed memory | `.harness/lessons/<category>/` |
| A10 | "Use when ..." convention | 모든 skill description |

---

## 8.3 다음 액션 (구현 우선순위)

1. **본 설계 문서 사용자 최종 컨펌** ← 현재
2. **`.harness/ETHOS.md` v0 작성** — P1~P16 본문화
3. **Maestro 시스템 프롬프트 v0** — 가장 큰 프롬프트. 다음을 통합:
   - 단일 사용자 창구 책무
   - PM 6엔티티 매니저
   - 4-step 사이클 driver
   - DoD 인터뷰어
   - Task Spec 작성자
   - 보고서 변환기 (Foreman raw → 사용자 surface)
   - HITL 모드 인식
4. **`pm-board` skill 본문 작성** — 6엔티티 CRUD + 우선순위 매트릭스 + 자동 트리거 + phase 전이 + DoD 인터뷰 + 중복 감지
5. **Task Spec / Task Report 스키마 동결** — YAML + 마크다운 본문, JSON Schema 또는 zod로 검증
6. **`tdd-discipline` skill 본문** — RED→GREEN→REFACTOR + 도구별 명령 (vitest/pytest/playwright/supabase migration dry-run)
7. **`compound-cycle` skill 본문** — Tier 1/2/3 + fingerprint-merge + lesson 템플릿
8. **`harness-evolution` skill 본문** — Phase 8 + proposal 템플릿 + 메타-회귀 가드
9. **`lesson-format` / `proposal-format` / `entity-format` 스키마 확정**
10. **Sentinel 룰 정적 도구 매핑 구체화** — `tdd_violation`, `compound_required`, `backlog_singularity`, `workflow_phase_skip`, `dod_required` 룰을 실제 명령으로
11. **`saas-stack` + `monorepo-layout` skill 본문**
12. **각 L2 에이전트 마크다운 초안** — 7/5/6-step + ETHOS preamble inject
13. **Foreman DAG 빌더 알고리즘 의사코드** — Task Spec → U-ID DAG → wave grouping
14. **PM 명령군 본문** — `/board`, `/roadmap`, `/milestone`, `/task`, `/backlog`, `/dod`, `/next` 등
15. **`/start` + `/evolve` + `/proposals` 커맨드 본문**
16. **End-to-end 드라이런** — URL 단축 SaaS, `--mode=dry-run`. 검증:
    - 사용자가 Maestro와만 대화하는가
    - board/에 V→R→M→B→T 엔티티가 자동 생성되는가
    - DoD 게이트가 동작하는가
    - 4-step 사이클이 직선으로 진행되는가
    - Sentinel이 backlog_singularity / discovered_not_logged를 실제로 잡는가
    - Task Spec ↔ Report 인터페이스가 동작하는가
    - Foreman이 user-facing 출력 안 하는가
    - Phase 8이 첫 proposal을 생성하는가 (자기 진화 자기 검증)
17. **회고 → 다음 버전 룰셋/프롬프트 튜닝**

---

## 8.4 열린 결정 사항 (다음 버전으로)

### PM 모델 관련
- **board의 git 버전 관리 정책**: board/* 는 git commit 권장이지만 자동 vs 수동? Maestro의 매 CRUD를 자동 commit? lessons/는?
- **엔티티 cross-link UI 표현**: `/board` 출력 시 의존성 그래프를 ASCII 트리로 보여줄지, 단순 리스트로 보여줄지. mermaid 다이어그램 생성?
- **stale 엔티티 정리 정책**: 14일 이상 update 없는 P3 task / parked backlog 자동 archived 후보로 표시할지.
- **Maestro 자체의 컨텍스트 압축**: board가 커지면 Maestro 컨텍스트도 커짐. `_index.md`만 컨텍스트에 두고 본문은 on-demand read로 강제하는 룰 필요.
- **여러 milestone 동시 진행 허용?**: 현재는 milestone 1개씩 active 가정. P1 milestone 진행 중에 P0 hotfix가 들어오면 어떻게 인터럽트?

### DoD 관련
- **DoD verify_cmd 표준**: 어떤 명령 셰이프를 표준으로 받을 것인가? bash 한 줄? script 파일 경로? 모든 verify_cmd가 exit code 의미를 같게 가져야 함.
- **DoD 항목 수 권장**: 너무 많으면 검증 부담, 너무 적으면 검증력 부족. 권장 임계?

### Compound 관련
- **BacklogItem 중복 감지 알고리즘**: Refinement에서 유사 title의 dedup. 임베딩 vs fuzzy. Phase 8에서 진화할 것인가?
- **Lesson fingerprint 알고리즘**: 같은 root_cause로 묶을 기준 (file path bucket, error class, stack frame normalize 등). 너무 좁으면 dedup 실패, 너무 넓으면 false merge.
- **ETHOS 영구화 가드**: 어떤 lesson은 ETHOS로 승격? 사이클 임계(3) + 사용자 명시 승인을 함께 게이트로.

### Vision 관련
- **Vision 변경 정책**: vision은 거의 안 바뀌지만 바뀔 때 영향 범위가 큼. 변경 시 모든 active roadmap/milestone 자동 재평가 트리거?
- **Vision의 ID 컨벤션**: 새 vision은 V-002로 진화? supersede? 1개만 active?

### TDD 관련
- **TDD 예외 카탈로그 확정**: 어떤 task 유형까지 `/tdd-exception`을 받을 것인가? hotfix 정의의 경계? spike → 본실로 승격 시 회귀 테스트 추가 절차?

### Phase 8 관련
- **Proposal 채택률 모니터링**: Agent-Architect가 매번 같은 종류 제안만 내고 다 reject되면 → Agent-Architect 자체의 학습 필요. 메타-메타 진화 정의?

### 운영
- **Refinement 자동 vs 수동**: 사용자가 매번 `/refinement` 쳐야 하는가, 아니면 backlog에 변화 누적 시 Maestro가 자동으로 Refinement 권유?
- **인터뷰 스크립트 디테일**: Maestro Phase 0의 표준 질문지 (10~15개). SaaS 도메인 일반 + 본 스택(Supabase/Cloudflare) 특화.
- **DAG 빌더 휴리스틱**: task 단위 입도(granularity) 결정 룰.
- **Worktree 정리 정책**: 머지 후 즉시 삭제 vs 보존. CI 캐시·디스크 비용.
- **Cloudflare 배포 단계**: DevOps 워커가 자동 배포까지 갈지, staging 까지만 갈지.
- **Supabase 마이그레이션 안전망**: dev/staging/prod 단계별 게이트.
- **테스트 계층 기본값**: QA 워커의 default test pyramid 비율 (unit/integration/e2e).
