# 02. 에이전트 로스터 & 워커 내부 구조

> 본 하네스의 모든 에이전트 정의, 계층 관계, 워커 내부 step 흐름, 모델 할당을 다룬다.

---

## 2.1 계층 구조

```
                    ╔═════════════════════════════════════════════╗
   사용자 ◄════════►║  Maestro  (단일 창구 + PM 엔티티 매니저)         ║
                    ║                                              ║
                    ║  관리:  Vision / Roadmap / Milestone /        ║
                    ║         Backlog / BacklogItem / Task         ║
                    ║  동작:  CRUD · 의존성 · 우선순위 · DoD          ║
                    ║         + 4-step 사이클 driver                ║
                    ╚════════════════╤═════════════════════════════╝
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
      Foreman Report ──► Maestro가 사용자에게 요약/번역하여 surface
```

**핵심 컨벤션**:
- 사용자 화살표는 오직 Maestro에서 시작/종료
- Foreman은 Maestro의 sub-agent (사용자가 직접 호출 불가)
- L2 워커 출력 → Foreman → Maestro로 요약 흐름. 사용자가 본 모든 메시지는 Maestro가 작성한 것.

---

## 2.2 에이전트 명세 (총 13)

- `TDD` 컬럼: ✅ = RED→GREEN→REFACTOR 의무, ⚙️ = skill-TDD (산출물 baseline-fail 검증), ❌ = 적용 안 함 (prose/spec)
- `Compound` 컬럼: 매 task 종료 시 lessons 캡처 기여도. **Lead** = lesson 작성 주체, Source = lesson 원천.

| 계층 | 에이전트 | 역할 | 권한 | TDD | Compound | OmO 매핑 |
|---|---|---|---|---|---|---|
| **L0** | **Maestro** | **단일 사용자 창구.** 6종 엔티티 (vision/roadmap/milestone/backlog/backlog-item/task) CRUD + 의존성 + 우선순위. 4-step 사이클 driver. 현재 task spec을 Foreman에 dispatch. 보고서 변환. HITL 인터뷰. 자료조사 위임. | read, ask user, delegate(task), write(.harness/board/*) | ❌ | Source (사용자 피드백 lesson) | Prometheus 확장 |
| **L1** | **Foreman** | Maestro의 sub-agent (user-facing 아님). Spec을 받아 DAG 작성, worktree 스폰, 머지 직렬화, Ralph 루프 감시. 결과는 보고서로 Maestro에 반환. | bash(git), task, write(DAG file only) | ❌ | Source (워커 간 충돌/머지 lesson) | Atlas 확장 |
| **L1** | **Sentinel** | 코드 품질 감사 (과설계/데드코드/컨벤션/TDD위반/아키위반/PM워크플로 위반). 거부 권한. | read, grep, lsp, ast-grep, task(spawn fixer only) | ⚙️ (룰 자체의 baseline) | Source (누적 finding이 가장 큰 lesson 원천) | 신규 |
| **L2** | **Strategist** | 기획. PRD/유저스토리/AC 작성, 우선순위, 스코프 컷. **자기 출력에 plan_placeholder validator 적용** (TBD/모호 표현 자체 거부 후 재작성). | read, web, write(docs only) | ❌ | Source (PRD-실코드 갭 lesson) | 신규 (writing) |
| **L2** | **Designer** | UX/UI. 와이어/디자인시스템 토큰/컴포넌트 사양. | read, write(design specs), multimodal-look | ❌ | Source | 신규 (artistry) |
| **L2** | **Frontend** | UI 구현. | full code | **✅** | Source | 신규 (visual-engineering) |
| **L2** | **Backend** | API/도메인 로직. | full code | **✅** | Source | 신규 (deep) |
| **L2** | **DB Architect** | 스키마/마이그레이션/인덱스/쿼리 최적화. | full code, bash(migrations) | **✅** (마이그 dry-run + rollback 테스트) | Source | 신규 (ultrabrain) |
| **L2** | **Security** | 인증/인가/시크릿/OWASP, 위협 모델링. | read, edit(security-related), audit | **✅** (보안 회귀 테스트) | Source | 신규 (ultrabrain) |
| **L2** | **Agent Architect** | 메타. 새 워크플로우/에이전트/스킬 정의 + **하네스 자기 진화 제안**. | read, edit(.opencode/*), write(.harness/proposals/*) | ⚙️ skill-TDD | **Lead** (전체 lessons 통합 + 진화 제안 작성) | 신규 |
| **L2** | **DevOps** | CI/CD, Cloudflare Workers/Pages/R2 배포, 환경변수, 시크릿, 로깅/모니터링, IaC. | full code, bash(deploy/wrangler/docker) | **✅** (deploy smoke test, rollback test) | Source | 신규 (deep) |
| **L2** | **QA** | 테스트 전략, E2E/Integration/Unit 계층 설계, Playwright/Vitest/Pytest 작성, 회귀 풀, 커버리지 게이트. | full code (tests only), playwright skill | **✅** (테스트의 테스트) | **Lead** (test-failure lesson 정리) | 신규 (deep) |
| **L2** | **Tech Writer** | API 레퍼런스, README, ADR, 사용자 가이드, 변경 로그, 데모 시나리오 + **lessons 본문 작성**. | read, write(docs only, .harness/lessons/*) | ❌ | **Lead** (lessons 마크다운 작성 책임) | 신규 (writing) |
| **Helpers** | Oracle, Librarian, Explore, Multimodal-Looker | 기존 OmO 그대로 활용. | (기본값) | ❌ | Source | 그대로 |

**Helpers 역할 요약**:
- **Oracle**: 아키텍처 컨설팅 (읽기전용, 깊은 추론)
- **Librarian**: 문서/OSS 검색 + `.harness/lessons/` 자동 inject 담당
- **Explore**: 빠른 codebase grep
- **Multimodal-Looker**: PDF/이미지/스크린샷 분석

---

## 2.3 워커 내부 구조 (컨텍스트 격리 패턴)

각 워커의 step 흐름은 **워커 유형에 따라 다르다**. 각 step은 **별도 task() 세션**으로 실행되어 컨텍스트가 누적되지 않는다.

### 2.3.1 코드 워커 (Backend / Frontend / DB / Security / DevOps / QA) — **7-step**

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
- plan에 `tdd_exception: "<사유>"` 명시 + Maestro HITL 승인
- 허용 카테고리: 1회성 spike/POC (`.spikes/` 디렉토리), hot-fix(사후 회귀 테스트 의무), 설정 변경, 마이그레이션 SQL(dry-run 의무로 대체)

### 2.3.2 Prose/Spec 워커 (Strategist / Designer / Tech Writer) — **5-step**

```
Worker
  ├─ step:research          → 자료조사, 기존 산출물 검토
  ├─ step:draft             → 초안 작성
  │                           (Sentinel plan_placeholder 검증:
  │                            Strategist의 PRD/task plan 산출에 TBD/모호 표현 차단)
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
    "worker": "backend",
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
| Maestro | claude-opus-4-7 max | kimi-k2.5 | — | 인터뷰·대화 품질, PM 추론 |
| Foreman | claude-sonnet-4-6 | gpt-5.4 medium | — | DAG 추론, 비용 효율 |
| Sentinel (1차) | gpt-5.4-mini | claude-haiku-4-5 | quick | 매 커밋이라 cheap |
| Sentinel (2차) | gpt-5.4 xhigh | claude-opus-4-7 max | ultrabrain | 깊은 판정 |
| Strategist | claude-opus-4-7 | kimi-k2.5 | writing | 글·스펙 작성 |
| Designer | gemini-3.1-pro | claude-opus-4-7 | artistry | 시각/창의 |
| Frontend | gemini-3.1-pro | gpt-5.4 | visual-engineering | UI 강점 |
| Backend | gpt-5.4 medium | claude-opus-4-7 | deep | 자율 실행 |
| DB Architect | gpt-5.4 xhigh | claude-opus-4-7 max | ultrabrain | 정확성 |
| Security | gpt-5.4 xhigh | claude-opus-4-7 max | ultrabrain | 추론 깊이 |
| Agent Architect | claude-opus-4-7 max | gpt-5.4 xhigh | ultrabrain | 메타 설계 |
| DevOps | gpt-5.4 medium | claude-sonnet-4-6 | deep | 자율 배포 스크립트 |
| QA | gpt-5.4 medium | gemini-3.1-pro | deep | 테스트 시나리오 생성 |
| Tech Writer | gemini-3.1-pro | claude-opus-4-7 | writing | 산문/문서 품질 |
