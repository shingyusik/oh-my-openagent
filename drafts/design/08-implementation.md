# 08. 구현 매핑 + 다음 액션 + 열린 사항

> 본 하네스를 `oh-my-openagent` 위에 어떻게 얹을 것인가의 구현 매핑, 우선순위, 열린 결정 항목.

---

## 8.1 oh-my-openagent 매핑

| 본 설계 | OmO 메커니즘 | 구현 방법 |
|---|---|---|
| **Maestro Core** | Prometheus 확장 + `pm-delegation` skill | `.opencode/agents/maestro.md` + `agents.prometheus.prompt_append` |
| **Private PM sub-agents** | Custom Agent + `pm-board` / `pm-delegation` skills | `.opencode/agents/board-clerk.md`, `milestone-planner.md`, `spec-writer.md`, `report-editor.md`, `context-librarian.md` |
| **Foreman** | Atlas 확장 또는 신규 primary | `.opencode/agents/foreman.md` (Atlas 패턴 차용) |
| **Sentinel** | 신규 + post-commit hook | `.opencode/agents/sentinel.md` + `.opencode/hooks/post-commit-*.sh` |
| **Worker profiles** | Custom Agent + Category + project profile | `.opencode/agents/worker-profiles/*.md` + `categories` 항목 |
| **워커 내부 7/5/6-step** | OmO `task()` 도구 별도 세션 | 워커 prompt가 task()로 자기 sub-step 호출 |
| **Worktree 병렬** | git worktree + OmO background task | Foreman이 bash로 worktree 생성, 각 worktree에서 OmO 세션 |
| **Ralph 루프** | 기존 `/ulw-loop` 패턴 | Maestro Core가 milestone 단위 자체 루프 |
| **자료조사** | Librarian + Explore + Oracle | 그대로 |
| **컨텍스트 격리** | task() 별도 세션 + 구조화 출력 | 프롬프트로 강제 |
| **코드 품질 룰** | Sentinel 프롬프트 + Skill | `.opencode/skills/sentinel-rules/SKILL.md` |
| **PM 6엔티티** | `pm-board` skill + `board/` 마크다운 | Board Clerk이 쓰고 Maestro Core가 승인 |
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
| A1 | Plan placeholder 금지 validator | `05-sentinel.md` §5.3 `plan_placeholder` 룰 + `02-agents.md` Planning Worker 책무 |
| A2 | 3-failure 아키텍처 escalation | Ralph 루프 circuit-breaker |
| A3 | Brainstorming 9-step | Maestro Core Phase 0 |
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
3. **Maestro Core 시스템 프롬프트 v0** - 얇은 orchestration prompt. 다음만 통합:
   - 단일 사용자 창구 책무
   - 최종 결정권자 / HITL 게이트
   - 4-step 사이클 driver
   - private PM sub-agent delegation
   - HITL 모드 인식
4. **Private PM sub-agent prompt v0 5종** - Board Clerk / Milestone Planner / Spec Writer / Report Editor / Context Librarian
5. **`pm-delegation` skill 본문 작성** - Maestro Core ↔ private PM sub-agent request/response schema
6. **`pm-board` skill 본문 작성** - Board Clerk용 6엔티티 CRUD + 우선순위 매트릭스 + 자동 트리거 + phase 전이 + 중복 감지
7. **Task Spec / Task Report / Report Summary 스키마 동결** - YAML + 마크다운 본문, JSON Schema 또는 zod로 검증
8. **`tdd-discipline` skill 본문** — RED→GREEN→REFACTOR + PROJECT_PROFILE의 test command 슬롯 연동
9. **`compound-cycle` skill 본문** — Tier 1/2/3 + fingerprint-merge + lesson 템플릿
10. **`harness-evolution` skill 본문** — Phase 8 + proposal 템플릿 + 메타-회귀 가드
11. **`lesson-format` / `proposal-format` / `entity-format` 스키마 확정**
12. **Sentinel 룰 정적 도구 매핑 구체화** — `tdd_violation`, `compound_required`, `backlog_singularity`, `workflow_phase_skip`, `dod_required` 룰을 실제 명령으로
13. **`project-profile` + `convention-registry` skill 본문**
14. **seed worker profile 마크다운 초안** — Planning / Design / Implementation / Data / Security / Quality / Ops / Documentation
15. **Foreman DAG 빌더 알고리즘 의사코드** — Task Spec → U-ID DAG → wave grouping
16. **PM 명령군 본문** — `/board`, `/roadmap`, `/milestone`, `/task`, `/backlog`, `/dod`, `/next` 등
17. **`/start` + `/evolve` + `/proposals` 커맨드 본문**
18. **End-to-end 드라이런** — 작은 일반 코딩 프로젝트, `--mode=dry-run`. 검증:
    - 사용자가 Maestro Core와만 대화하는가
    - Maestro Core가 board 본문 전체를 직접 읽지 않는가
    - Board Clerk / Milestone Planner / Spec Writer / Report Editor delegation schema가 지켜지는가
    - board/에 V→R→M→B→T 엔티티가 자동 생성되는가
    - DoD 게이트가 동작하는가
    - 4-step 사이클이 직선으로 진행되는가
    - Sentinel이 backlog_singularity / discovered_not_logged를 실제로 잡는가
    - Task Spec ↔ Report 인터페이스가 동작하는가
    - Foreman이 user-facing 출력 안 하는가
    - Phase 8이 첫 proposal을 생성하는가 (자기 진화 자기 검증)
    - 반복 finding을 바탕으로 worker profile 또는 convention proposal이 생성되는가
19. **회고 → 다음 버전 룰셋/프롬프트 튜닝**

---

## 8.4 열린 결정 사항 (다음 버전으로)

### PM 모델 관련
- **board의 git 버전 관리 정책**: board/* 는 git commit 권장이지만 자동 vs 수동? Board Clerk의 승인된 CRUD를 자동 commit? lessons/는?
- **엔티티 cross-link UI 표현**: `/board` 출력 시 의존성 그래프를 ASCII 트리로 보여줄지, 단순 리스트로 보여줄지. mermaid 다이어그램 생성?
- **stale 엔티티 정리 정책**: 14일 이상 update 없는 P3 task / parked backlog 자동 archived 후보로 표시할지.
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
- **Refinement 자동 vs 수동**: 사용자가 매번 `/refinement` 쳐야 하는가, 아니면 backlog에 변화 누적 시 Maestro Core가 자동으로 Refinement 권유?
- **인터뷰 스크립트 디테일**: Maestro Core Phase 0의 표준 질문지 (10~15개). 프로젝트 타입/언어/런타임/레포 구조/검증 명령 중심.
- **DAG 빌더 휴리스틱**: task 단위 입도(granularity) 결정 룰.
- **Worktree 정리 정책**: 머지 후 즉시 삭제 vs 보존. CI 캐시·디스크 비용.
- **배포/릴리스 단계**: Ops Worker가 자동 배포/릴리스까지 갈지, dry-run/staging 까지만 갈지.
- **데이터 변경 안전망**: Data Worker의 dry-run/rollback/backup 게이트를 프로젝트별로 어디까지 강제할지.
- **테스트 계층 기본값**: Quality Worker의 default test pyramid 비율 (unit/integration/e2e/property/benchmark 등).

### 진화형 프로필 관련
- **seed worker profile 최소 세트**: 모든 프로젝트에 8개 seed를 둘지, Phase 0에서 3~4개만 시작할지.
- **프로필 분리 기준**: 같은 Implementation Worker가 반복적으로 다른 실패를 낼 때 새 worker로 분리하는 hit-rate 임계.
- **컨벤션 승격 기준**: 한 번의 사용자 선호를 project rule로 둘지, 3회 반복 후 rule로 승격할지.

---

## 8.5 v0.6 기본 결정: Maestro Core 분해

v0.6부터 Maestro Core는 단일 사용자 창구와 최종 결정권자 역할만 가진다. PM 반복 노동은 private PM sub-agent가 맡는다.

| 결정 | 기본값 |
|---|---|
| Maestro Core 컨텍스트 | `board/*/_index.md`, active milestone/task, 최근 summary만 상시 보유 |
| board 본문 접근 | Context Librarian을 통한 on-demand read |
| board 쓰기 | Board Clerk 전담. Maestro Core 승인 없는 직접 CRUD 금지 |
| Refinement/Planning | Milestone Planner가 proposal 생성, Maestro Core가 승인 |
| Task Spec | Spec Writer가 초안 생성, Maestro Core 승인 후 Foreman dispatch |
| Report surface | Report Editor가 사용자용 summary 생성, Maestro Core가 최종 문구 결정 |
| sub-agent 출력 | schema 응답만 허용. 자유 형식 긴 보고서 금지 |
| user-facing 채널 | Maestro Core만 허용 |

---

## 8.6 v0.6 기본 결정: 일반 코딩 하네스

본 하네스는 특정 제품 유형이나 스택을 기본값으로 삼지 않는다. 코어는 강제하고, 세부 운영은 프로젝트 프로필과 진화 사이클이 만든다.

| 구분 | 결정 |
|---|---|
| 고정 코어 | TDD, Maestro Core 단일 surface, Backlog SSOT, DoD, 4-step cycle, Sentinel, Compound, Phase 8 |
| 비고정 영역 | worker profiles, 코드 컨벤션, repo layout, test/build command, 배포/릴리스 규칙 |
| 초기화 | Phase 0에서 `.harness/PROJECT_PROFILE.md`와 `.harness/CONVENTIONS.md`를 생성 |
| 진화 | 반복 finding/lesson/user preference를 Agent-Architect가 proposal로 승격 |
| Sentinel 적용 | Core rules는 항상 강제, Project/Worker rules는 profile/registry에서 읽어 적용 |
