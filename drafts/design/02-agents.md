# 02. 에이전트 로스터 & 워커 내부 구조

> 본 하네스의 모든 에이전트 정의, 계층 관계, 워커 내부 step 흐름, 모델 할당을 다룬다.

---

## 2.1 계층 구조

```
                    ╔═════════════════════════════════════════════╗
   사용자 ◄════════►║  Maestro Core (단일 창구 + 최종 결정권자)       ║
                    ║                                              ║
                    ║  소유: 사용자 대화 · HITL · 실행 승인 · 최종 판단 ║
                    ║  위임: PM CRUD · 계획 초안 · spec · report 요약  ║
                    ╚═══════════╤════════════╤════════════════════╝
                                │            │ private PM sub-agents
                                │            ▼
                                │  ┌─────────────────────────────┐
                                │  │ Board Clerk / Milestone     │
                                │  │ Planner / Spec Writer /     │
                                │  │ Report Editor / Librarian   │
                                │  └─────────────────────────────┘
                                     │  Task Spec (명세서)
                                     ▼
                       ┌─────────────────────────────┐
                       │  Foreman  (현장 십장)         │   user-facing 아님
                       │  DAG 빌드, worktree 분배       │
                       │  Ralph 루프 / Wave 머지        │
                       └──┬───────────────────────┬──┘
                          │                       │
          ┌───────────────┼────────────────┐      │
          │               │                │      ▼
       ┌──▼──┐         ┌──▼──┐         ┌──▼──┐  ┌──────────────┐
       │ L2  │   ...   │ L2  │   ...   │ L2  │  │  Sentinel     │
       │Worker│        │Worker│        │Worker│  │  (감독관)      │
       └──┬──┘         └──┬──┘         └──┬──┘  └──────────────┘
          │ 7/5/6-step    │               │           ▲
          ▼               ▼               ▼           │ 매 commit
       commit ────────────────────────────────────────┘
          │
          ▼
      compound ──► .harness/lessons/  ──► Librarian이 다음 task 시작 시 inject
          │
          ▼
      Foreman Report ──► Report Editor ──► Maestro Core가 사용자에게 surface
```

**핵심 컨벤션**:
- 사용자 화살표는 오직 Maestro Core에서 시작/종료
- Maestro Core는 단일 사용자 창구와 최종 결정권자이며, PM 반복 노동은 private PM sub-agent에 위임
- private PM sub-agent와 Foreman은 Maestro Core의 sub-agent (사용자가 직접 호출 불가)
- L2 워커 출력 → Foreman → Report Editor → Maestro Core로 요약 흐름. 사용자가 본 모든 메시지는 Maestro Core가 작성한 것.

---

## 2.2 에이전트 명세 (코어 + 진화형 worker profiles)

- `TDD` 컬럼: ✅ = RED→GREEN→REFACTOR 의무, ⚙️ = skill-TDD (산출물 baseline-fail 검증), ❌ = 적용 안 함 (prose/spec)
- `Compound` 컬럼: 매 task 종료 시 lessons 캡처 기여도. **Lead** = lesson 작성 주체, Source = lesson 원천.

### 2.2.1 코어 에이전트 (항상 존재)

| 계층 | 에이전트 | 역할 | 권한 | TDD | Compound | OmO 매핑 |
|---|---|---|---|---|---|---|
| **L0** | **Maestro Core** | **단일 사용자 창구 + 최종 결정권자.** 사용자 대화, HITL 질문, 실행 승인, 4-step 사이클 orchestration, sub-agent 산출물 검토와 최종 surface. PM 노동은 직접 수행하지 않고 private PM sub-agent에 위임. | read(_index + active items), ask user, delegate(task), approve/reject | ❌ | Source (사용자 피드백 lesson) | Prometheus 확장 |
| **L0-private** | **Board Clerk** | Vision/Roadmap/Milestone/Backlog/BacklogItem/Task CRUD, `_index.md` 갱신, cross-link 정합성, 상태 전이 적용. Maestro Core가 승인한 patch request만 반영. | write(.harness/board/*), read board/* | ❌ | Source (PM drift lesson) | 신규 (pm-board) |
| **L0-private** | **Milestone Planner** | Refinement, backlog triage, milestone 후보, 우선순위/의존성 분석, DoD 초안. 실행 결정은 Maestro Core에 제안만 한다. | read board/*, write structured proposal only | ❌ | Source (planning lesson) | 신규 (planning) |
| **L0-private** | **Spec Writer** | selected backlog item과 task를 Foreman Task Spec으로 변환. AC, constraints, worker allocation, escalation 조건을 schema에 맞춘다. | read board active items, write structured spec only | ❌ | Source (spec gap lesson) | 신규 (writing) |
| **L0-private** | **Report Editor** | Foreman/Sentinel raw report를 사용자용 요약으로 변환. 코드/diff/raw 로그를 사용자에게 직접 노출하지 않는다. | read report, write structured summary only | ❌ | Source (report clarity lesson) | 신규 (writing) |
| **L0-private** | **Context Librarian** | Maestro Core가 요청한 결정에 필요한 board 본문, lessons, prior decisions만 검색해 500자 내외 요약으로 반환. | read-only | ❌ | Source (retrieval lesson) | Librarian 확장 |
| **L1** | **Foreman** | Maestro Core의 sub-agent (user-facing 아님). Spec을 받아 DAG 작성, worktree 스폰, 머지 직렬화, Ralph 루프 감시. 결과는 Report Editor와 Maestro Core에 반환. | bash(git), task, write(DAG file only) | ❌ | Source (워커 간 충돌/머지 lesson) | Atlas 확장 |
| **L1** | **Sentinel** | 코드 품질 감사 (과설계/데드코드/컨벤션/TDD위반/아키위반/PM워크플로 위반). 거부 권한. | read, grep, lsp, ast-grep, task(spawn fixer only) | ⚙️ (룰 자체의 baseline) | Source (누적 finding이 가장 큰 lesson 원천) | 신규 |
| **L2** | **Agent Architect** | 메타. 새 워크플로우/에이전트/스킬 정의 + **하네스 자기 진화 제안**. | read, edit(.opencode/*), write(.harness/proposals/*) | ⚙️ skill-TDD | **Lead** (전체 lessons 통합 + 진화 제안 작성) | 신규 |
| **Helpers** | Oracle, Librarian, Explore, Multimodal-Looker | 기존 OmO 그대로 활용. | (기본값) | ❌ | Source | 그대로 |

### 2.2.2 Worker profiles (프로젝트별 활성화)

Worker profile은 하네스의 고정 원칙이 아니라 **프로젝트 프로필**이다. `/start` 시점에는 최소 seed만 제안하고, 실제 역할은 backlog/lessons/Sentinel hit-rate를 통해 채택·분리·병합·폐기된다.

| 프로필 유형 | Use when | 기본 step | TDD | 생성/변경 경로 |
|---|---|---|---|---|
| **Planning Worker** | 요구사항, 범위, 우선순위, AC를 명확히 해야 할 때 | 5-step prose/spec | ❌ | seed 또는 Phase 8 proposal |
| **Design Worker** | 사용자 경험, 시스템 설계, 정보구조, 인터페이스 설계가 필요할 때 | 5-step prose/spec | ❌ | seed 또는 Phase 8 proposal |
| **Implementation Worker** | 코드 변경이 필요한 일반 작업 | 7-step code | ✅ | project profile에서 언어/스택별로 specialization |
| **Data Worker** | 스키마, 마이그레이션, 데이터 파이프라인, 저장소 모델 변경 | 7-step code + dry-run/rollback plug | ✅ | 필요 시 specialization |
| **Security Worker** | 권한, 인증, 입력 검증, 비밀, 위협 모델 관련 작업 | 7-step code + audit plug | ✅ | 필요 시 specialization |
| **Quality Worker** | 테스트 전략, 회귀 테스트, 커버리지, E2E/통합 검증 | 7-step code | ✅ | 필요 시 specialization |
| **Ops Worker** | 배포, CI/CD, 런타임, 관측성, 운영 자동화 | 7-step code + smoke/rollback plug | ✅ | 필요 시 specialization |
| **Documentation Worker** | README, ADR, changelog, 사용자 문서, lessons 본문 | 5-step prose/spec | ❌ | seed 또는 Phase 8 proposal |

예: 웹앱 프로젝트는 `ui-implementation` / `api-implementation`으로 나눌 수 있고, 라이브러리 프로젝트는 `public-api` / `compatibility` / `benchmark` worker로 진화할 수 있다. 하네스는 이름을 고정하지 않고 `worker-profiles/*.md`를 읽어 Foreman의 worker allocation에 반영한다.

**Helpers 역할 요약**:
- **Oracle**: 아키텍처 컨설팅 (읽기전용, 깊은 추론)
- **Librarian**: 문서/OSS 검색 + `.harness/lessons/` 자동 inject 담당
- **Explore**: 빠른 codebase grep
- **Multimodal-Looker**: PDF/이미지/스크린샷 분석

---

## 2.3 워커 내부 구조 (컨텍스트 격리 패턴)

각 워커의 step 흐름은 **워커 유형에 따라 다르다**. 각 step은 **별도 task() 세션**으로 실행되어 컨텍스트가 누적되지 않는다.

### 2.3.1 코드 워커 (Implementation / Data / Security / Quality / Ops 등) — **7-step**

```
Worker
  ├─ step:plan              → task 분해, 영향 file, AC 명시
  │                           (Sentinel plan_placeholder 검증: TBD/추후/모호 표현 금지)
  ├─ step:red               → 실패하는 테스트 작성
  │                           (lsp_diagnostics 컴파일 OK + 테스트 실행 시 FAIL 확인)
  ├─ step:green             → 테스트 통과시키는 최소 코드
  ├─ step:refactor          → 통과 유지하며 정리 (중복 제거, 명명, 단순화)
  ├─ step:spec-review       → 변경이 plan/AC를 충족하는가만 판정 (별도 task())
  │                           (Sentinel 4-level verifier Level 1~3 적용)
  ├─ step:quality-review    → 스타일·복잡도·중복만 판정 (별도 task())
  ├─ step:commit            → git-master로 atomic commit
  │                           + step:self-check 산출: SUMMARY.md에 다음 블록 의무
  │                             ## Self-Check
  │                             - Files claimed: <list> → 각각 EXISTS 검증
  │                             - Commits claimed: <hashes> → git log 존재 검증
  │                             - Tests added/modified: <count>
  │                             - Verify commands run: <cmd> → exit code
  │                             - Result: PASSED|FAILED
  │                           (Sentinel self_check_required 강제)
  └─ step:compound          → 이 task 동안의 실수·교훈을 1-paragraph lessons 후보 emit
```

**게이트**:
- **plan 게이트**: Sentinel `plan_placeholder` — TBD/추후/모호 표현 발견 시 BLOCK
- **red 게이트**: 작성한 테스트가 실제로 FAIL 함을 증명하지 못하면 green 진입 차단. Sentinel `tdd_violation` 자동 감시.
- **refactor 게이트**: 모든 기존 + 신규 테스트 통과 유지.
- **commit 게이트**: Self-Check 블록 누락 또는 검증 실패 시 BLOCK. Sentinel `self_check_required`.
- **모든 단계**: Sentinel `analysis_paralysis` — 연속 5회 read-only 후 자백 또는 BLOCKED 보고 의무.

**TDD 예외** (drop의 경우만):
- plan에 `tdd_exception: "<사유>"` 명시 + Maestro Core HITL 승인
- 허용 카테고리: 1회성 spike/POC (`.spikes/` 디렉토리), hot-fix(사후 회귀 테스트 의무), 설정 변경, 데이터/스키마 마이그레이션(dry-run 의무로 대체)

### 2.3.2 Prose/Spec 워커 (Planning / Design / Documentation 등) — **5-step**

```
Worker
  ├─ step:research          → 자료조사, 기존 산출물 검토
  ├─ step:draft             → 초안 작성
  │                           (Sentinel plan_placeholder 검증:
  │                            Planning Worker의 spec/task plan 산출에 TBD/모호 표현 차단)
  ├─ step:peer-review       → 다른 prose 워커 또는 Oracle에 리뷰 의뢰 (별도 task())
  ├─ step:revise            → 피드백 반영
  └─ step:compound          → lessons 후보 emit
```

### 2.3.3 메타 워커 (Agent-Architect) — **6-step + skill-TDD**

```
Worker
  ├─ step:diagnose          → sentinel-log + lessons + proposals 스캔
  ├─ step:baseline          → 제안하려는 skill/hook이 막아야 할 실패 시나리오를 baseline으로 재현
  ├─ step:design            → 신규 skill/hook/agent 설계 (SKILL.md.tmpl, hook.json 등)
  ├─ step:propose           → .harness/proposals/<date>-<topic>.md로 출력 (구현은 X)
  ├─ step:peer-review       → Oracle에 의뢰
  └─ step:compound          → 진화 메커니즘 자체의 lesson emit
```

### 2.3.4 공통 격리 원칙

- 각 step은 직전 step의 **명시된 산출물만** 입력으로 받는다 (plan.md, diff, test output 등)
- 상위(Foreman)는 step 결과의 **요약 JSON만** 받는다:
  ```json
  {
    "u_id": "U-007",
    "worker": "implementation",
    "status": "done",
    "commit": "abc123",
    "tests_added": 4,
    "files_touched": 3,
    "sentinel": "pass",
    "compound_emitted": "L-2026-05-20-003"
  }
  ```
- 상위는 코드/테스트 본문을 절대 보지 않는다. lesson ID로 필요 시 lookup.

---

## 2.4 모델 할당 (가변, 운영하면서 튜닝)

| 에이전트 | 1순위 | Fallback | 카테고리 | 이유 |
|---|---|---|---|---|
| Maestro Core | claude-opus-4-7 max | kimi-k2.5 | — | 인터뷰·대화 품질, 최종 판단 |
| Board Clerk | gpt-5.4 medium | claude-sonnet-4-6 | writing | schema 정합성, 원장 수정 |
| Milestone Planner | claude-opus-4-7 | gpt-5.4 | writing | 우선순위·의존성 추론 |
| Spec Writer | gpt-5.4 medium | claude-sonnet-4-6 | writing | 명세 구조화 |
| Report Editor | gpt-5.4-mini | claude-haiku-4-5 | quick | 보고서 요약 |
| Context Librarian | gpt-5.4-mini | claude-haiku-4-5 | quick | 검색 요약 |
| Foreman | claude-sonnet-4-6 | gpt-5.4 medium | — | DAG 추론, 비용 효율 |
| Sentinel (1차) | gpt-5.4-mini | claude-haiku-4-5 | quick | 매 커밋이라 cheap |
| Sentinel (2차) | gpt-5.4 xhigh | claude-opus-4-7 max | ultrabrain | 깊은 판정 |
| Agent Architect | claude-opus-4-7 max | gpt-5.4 xhigh | ultrabrain | 메타 설계 |
| Planning Worker | claude-opus-4-7 | kimi-k2.5 | writing | 요구사항·범위 추론 |
| Design Worker | gemini-3.1-pro | claude-opus-4-7 | artistry | 설계·UX·시각 추론 |
| Implementation Worker | gpt-5.4 medium | claude-opus-4-7 | deep | 자율 코드 변경 |
| Data Worker | gpt-5.4 xhigh | claude-opus-4-7 max | ultrabrain | 데이터 무결성 |
| Security Worker | gpt-5.4 xhigh | claude-opus-4-7 max | ultrabrain | 보안 추론 |
| Quality Worker | gpt-5.4 medium | gemini-3.1-pro | deep | 테스트 시나리오 생성 |
| Ops Worker | gpt-5.4 medium | claude-sonnet-4-6 | deep | 운영 자동화 |
| Documentation Worker | gemini-3.1-pro | claude-opus-4-7 | writing | 산문/문서 품질 |
