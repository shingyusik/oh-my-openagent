# CHANGELOG — 하네스 설계 변경 이력

> 이 파일은 `my-harness-design.md` 및 `design/*.md`의 **변경 이력과 결정 사항 누적**을 보관한다.
> 본 설계 문서들은 "현재 상태"만 기술하며 과거 버전 정보는 여기에 있다.

---

## v0.5.3 (2026-05-20) - 일반 코딩 하네스화

### 변경 요약
- **SaaS 전용 프레이밍 제거**: 현재 설계 제목과 빠른 시작을 "코딩 하네스" / "새 프로젝트" 기준으로 변경.
- **코어/프로젝트 레이어 분리**: TDD, Maestro Core 단일 surface, Backlog SSOT, DoD, 4-step cycle, Sentinel, Compound는 고정 코어로 유지.
- **Worker profile 진화 모델 도입**: Frontend/Backend/DB/DevOps 같은 고정 직책 대신 Planning/Design/Implementation/Data/Security/Quality/Ops/Documentation seed profile을 두고 프로젝트별로 채택·분리·병합·폐기.
- **스택 하드코딩 제거**: `saas-stack`, `monorepo-layout`, Supabase/Cloudflare/Tailwind/shadcn 등 고정 스택 규칙을 `project-profile` + `convention-registry`로 대체.
- **Sentinel 프로필 기반화**: 정적 도구와 architecture/convention rule을 `.harness/PROJECT_PROFILE.md`와 `.harness/CONVENTIONS.md`에서 읽어 적용.

### 신규 결정 사항

| # | 항목 | 결정 |
|---|---|---|
| 47 | 일반 코딩 하네스 | 특정 제품 유형(SaaS 등)이나 스택을 기본값으로 삼지 않는다 |
| 48 | Project profile | Phase 0에서 project_type, languages, runtimes, commands, layout, worker profiles를 기록 |
| 49 | Convention registry | 코드 컨벤션과 관리 규칙은 core/project/worker rule로 분리 |
| 50 | Worker profile evolution | worker agents는 seed에서 시작하고 lessons/proposals로 진화한다 |
| 51 | Sentinel profile slots | lint/typecheck/test/security/dependency 도구는 profile slot으로 선언한다 |

### 영향 받은 파일
- `my-harness-design.md` - 제목과 빠른 시작에서 SaaS 제거
- `design/01-principles.md` - P2를 project-adaptive agents로 재정의
- `design/02-agents.md` - 고정 L2 직책을 진화형 worker profiles로 전환
- `design/05-sentinel.md` - 스택 종속 룰과 도구 매핑을 프로젝트 프로필 기반으로 전환
- `design/07-inventory.md` - 고정 SaaS/모노레포 스택 대신 PROJECT_PROFILE/CONVENTIONS 추가
- `design/08-implementation.md` - `project-profile`, `convention-registry`, 일반 코딩 하네스 기본 결정 추가

---

## v0.5.2 (2026-05-20) - Maestro 책임 분해

### 변경 요약
- **Maestro 재정의**: 단일 사용자 창구 + PM 엔티티 매니저에서 **단일 사용자 창구 + 최종 결정권자**로 축소.
- **Private PM sub-agent 5종 추가**: Board Clerk, Milestone Planner, Spec Writer, Report Editor, Context Librarian.
- **PM 원장 쓰기 권한 분리**: `board/*` 직접 쓰기는 Board Clerk 전담. Maestro Core는 승인/거부와 사용자 surface만 담당.
- **컨텍스트 압축 기본값 확정**: Maestro Core는 `_index.md`, active item, 최근 summary만 상시 보유하고 본문은 on-demand read.
- **구현 순서 조정**: Maestro prompt를 얇게 만들고 `pm-delegation` skill과 private PM prompt를 선행 작성하도록 변경.

### 신규 결정 사항

| # | 항목 | 결정 |
|---|---|---|
| 40 | Maestro Core 책임 | 사용자 대화, HITL 질문, 실행 승인, 최종 판단, sub-agent 산출물 surface |
| 41 | Board Clerk | Vision/Roadmap/Milestone/Backlog/Task CRUD와 `_index.md` 정합성 전담 |
| 42 | Milestone Planner | Refinement/Planning proposal, 우선순위, 의존성, DoD 초안 담당 |
| 43 | Spec Writer | selected backlog/task를 Foreman Task Spec으로 변환 |
| 44 | Report Editor | Foreman/Sentinel raw report를 사용자용 summary로 변환 |
| 45 | Context Librarian | board 본문, lessons, prior decisions를 on-demand 검색 요약 |
| 46 | PM delegation schema | private PM sub-agent 출력은 schema 응답만 허용. 자유 형식 긴 보고서 금지 |

### 영향 받은 파일
- `my-harness-design.md` - 한 줄 요약과 계층도에서 Maestro Core + private PM sub-agent 구조 반영
- `design/01-principles.md` - P12를 단일 surface + decision owner로 재정의
- `design/02-agents.md` - 에이전트 수 18 + helpers로 갱신, private PM sub-agent roster 추가
- `design/03-pm-model.md` - 쓰기 권한, 상태 전이, Task Spec/Report 인터페이스를 위임 구조로 갱신
- `design/04-workflow.md` - Phase 0과 4-step cycle을 Maestro Core + private PM sub-agent 흐름으로 갱신
- `design/06-execution-infra.md` - 4-level context isolation과 Maestro Core 컨텍스트 상한 추가
- `design/07-inventory.md` - agent/skill inventory에 private PM sub-agent와 `pm-delegation` 추가
- `design/08-implementation.md` - 구현 매핑, 우선순위, v0.6 기본 결정 추가

---

## application-plan v0.3 (2026-05-20) — stale 정리 + split 구조 반영

> 본 항목은 `application-plan.md` 자체의 변경 이력. design 본체와는 분리.

### 변경 요약
- **참조 갱신**: 모든 `§2.2`/`§2.3`/`§3 Phase` 류 stale 섹션 번호 → `design/0X-*.md` 파일 + 섹션 참조로 교체
- **Status 컬럼 추가**: S/A/B-tier 매트릭스 모든 행에 ✅ / ⚙️ / ⏳ / ⛔ 적용 상태 표기
- **v0.4/v0.5 변경 반영**: PM 6-tier, 4-step 사이클, DoD 의무화, Vision entity 등 누락된 진화 흡수
- **§4 obsolete 제거**: 기존 "v0.3 메인 설계 문서 갱신 plan"은 갱신이 완료되어 의미 없음. "구현 페이징"으로 재작성
- **§7 흡수**: "v0.3 추가 요구 — TDD/Compound/자기 진화" 별도 섹션 → §3 "사용자 추가 요구의 외부 패턴 매핑"으로 정상화
- **충돌 표 보강**: v0.4/v0.5 추가 충돌 (GSD discuss-plan-execute-verify vs worktree 병렬, compound STRATEGY vs Vision entity, GSD adversarial verifier 등) 4건 추가
- **구현 페이즈 신설**: design 단계는 완료(v0.5.1)되었으므로 impl v0.1 → v0.9 로드맵으로 갱신

### 적용 status 요약 (application-plan v0.3 시점)
- **S-tier (13개)**: 모두 ✅ 적용 완료
- **A-tier (13개)**: 11개 ✅, 1개 ⚙️ (A12 fingerprint 알고리즘 명시 보강 필요), 1개 ⏳ (A10 컨벤션 skill 작성 시 적용)
- **B-tier (8개)**: 3개 ✅, 2개 ⚙️ (B3 patch harvest, B6 REQ 커버리지), 3개 ⏳ (B1/B2 + 구현 단계 결정)

---

## v0.5.1 (2026-05-20) — application-plan 누락 패턴 적용

### 변경 요약
- application-plan.md의 S-tier에서 design에 누락되었던 5개 패턴 적용 완료.

### 신규 결정 사항

| # | 항목 | 결정 |
|---|---|---|
| 35 | `analysis_paralysis` Sentinel 룰 | P0. 연속 5회 read-only 도구 호출 후 Write 없으면 BLOCK 또는 자백. 같은 파일 5회 Read, 10턴 read-only ratio >90% 등 |
| 36 | `slopcheck` Sentinel 룰 | P0 [NEVER_GATE]. 패키지 등급 [VERIFIED/ASSUMED/SLOP]. ASSUMED 이상 Maestro HITL. CF 환경에 node-only 의존성 추가 BLOCK |
| 37 | `self_check_required` Sentinel 룰 | P0. 워커 commit 시 SUMMARY.md에 `## Self-Check` 블록 의무. files-exist + commit hash 존재 + verify exit code 검증 |
| 38 | `plan_placeholder` Sentinel 룰 | P0. plan 본문에 TBD/추후/모호 표현 BLOCK. Strategist 자기 출력 검증 의무 |
| 39 | 4-level Verifier 방법론 표준화 | EXISTS → SUBSTANTIVE → WIRED → REAL DATA FLOW. DoD verify + worker spec-review + merge gate에서 사용. adversarial 기본 가정 ("통과 증거 없으면 실패") |

### 영향 받은 파일
- `design/05-sentinel.md` — 룰 4종 + §5.9 4-level verifier 방법론 신규 섹션
- `design/02-agents.md` — 7-step에 self-check 출력 의무 명시, 5-step에 plan_placeholder 게이트 추가, Strategist 책무 보강
- `design/04-workflow.md` — Review 단계 DoD 검증에 4-level verifier 명시
- `design/01-principles.md` — P1/P5 강제 메커니즘 표에 신규 4룰 + 방법론 매핑 추가
- `design/08-implementation.md` — S5/S6/S7/S8/A1의 본 설계 위치 정확화

---

## v0.5 (2026-05-20)

### 변경 요약
- **PM 계층 재정렬**: `Vision → Roadmap → Milestone → Backlog → BacklogItem → Task` 6-tier (Vision 신규 최상단).
- **Backlog = 단일 진실 저장소 (SSOT)**: 기능/버그/기술부채/리서치 모든 것이 backlog로. 다른 todo store 금지 (P13 신규).
- **4-step 운영 사이클 강제**: Refinement → Planning → Execution → Review. Maestro와 Sentinel이 단계 스킵 차단 (P14 신규).
- **Definition of Done (DoD) 의무화**: 모든 milestone은 측정 가능한 DoD 없이 생성 불가. 모든 DoD PASS 없이 done 불가 (P15 신규).
- **Issue 엔티티 흡수**: Issue는 `BacklogItem(type=bug)`의 별칭. 별도 entity X.
- **Flexibility 룰**: roadmap/milestone은 변경 가능하지만 변경 시 영향받는 backlog item 재정렬 자동 (P16 신규).
- 신규 Sentinel 룰 5종: `backlog_singularity`, `dod_required`, `workflow_phase_skip`, `discovered_not_logged`, `flexibility_traceability`.
- PM 명령군 재편: `/vision`, `/refinement`, `/planning`, `/review`, `/dod` 신규. `/issue` → `/backlog add bug` 별칭.

### 신규 결정 사항

| # | 항목 | 결정 |
|---|---|---|
| 24 | 6-tier PM 계층 | Vision → Roadmap → Milestone → Backlog → BacklogItem → Task. 단방향. |
| 25 | Backlog = 단일 SSOT (P13) | 모든 todo는 backlog로. 다른 곳 todo 저장 금지 (코드 주석 TODO 포함). Sentinel `backlog_singularity`. |
| 26 | Issue 엔티티 흡수 | Issue 별도 엔티티 X. `BacklogItem(type=bug)`로 통합. `/issue add` = `/backlog add bug` 별칭. |
| 27 | 4-step 사이클 강제 (P14) | Refinement → Planning → Execution → Review. STATE.md phase 마커로 강제. 스킵 시 Sentinel `workflow_phase_skip` BLOCK. |
| 28 | DoD 의무화 (P15) | 모든 milestone은 측정 가능한 DoD 없이 생성 불가. verify_cmd 필수. 모두 passed 없이 done 불가. |
| 29 | Flexibility 룰 (P16) | roadmap/milestone 변경 시 영향 backlog item 자동 재정렬 + CONTEXT.md append-only 로그. |
| 30 | DoD 측정 가능성 자동 검증 | DoD criterion에 "대충", "잘", "괜찮으면" 등 비측정 표현 발견 시 BLOCK. verify_cmd 빈 항목 BLOCK. |
| 31 | Discovered-not-logged 강제 | 워커가 새 발견 언급 후 backlog item 미생성 → BLOCK. 코드 주석 TODO 단독 금지. |
| 32 | BacklogItem 타입 카탈로그 | `feature` / `bug` / `tech_debt` / `research` / `spike` 5종. 다른 type 거부. |
| 33 | 가치×시급성 priority 매트릭스 | P0~P3 자동 계산 + 수동 override 가능. milestone에 selected된 후 P2/P3는 경고. |
| 34 | Phase 마커 STATE.md 위치 | `.harness/STATE.md`에 milestone + phase + phase_entered_at + phase_completed_at 기록. O_EXCL lock 유지. |

---

## v0.4 (2026-05-20)

### 변경 요약
- **Maestro = 단일 사용자 창구 + PM 엔티티 매니저** (P12 신규). 5종 엔티티 관리: roadmap / milestone / task / issue / backlog.
- Maestro의 CRUD 책무 명시 (등록/삭제/수정 + 상태 전이).
- **Foreman은 user-facing 제거**: Maestro로부터 "현재 해야 할 일 + 명세서"를 받는 sub-agent로 좌천. 사용자 직접 호출 X.
- **Spec 스키마 / Report 스키마** 표준화 (Maestro ↔ Foreman 인터페이스).
- PM 명령군 신규: `/roadmap`, `/milestone`, `/task`, `/issue`, `/backlog`, `/next`, `/board`.
- 기존 컨텍스트 격리 원칙 강화: 사용자는 Maestro만 본다, Foreman/Sentinel/L2의 출력은 Maestro가 요약·번역해서 surface.

### 결정 사항

| # | 항목 | 결정 |
|---|---|---|
| 16 | Maestro = 단일 사용자 창구 | 모든 사용자 입출력은 Maestro 경유. Foreman/Sentinel/L2 워커 직접 노출 금지. |
| 17 | PM 모델 | v0.4: 5엔티티 (roadmap/milestone/task/issue/backlog) → **v0.5에서 Vision 신규 + Issue 흡수로 6-tier로 변경.** |
| 18 | 엔티티 ID 컨벤션 | `V-` (v0.5) / `R-` / `M-` / `T-` / `B-` prefix. 영구 ID, renumber 금지. (I-는 v0.5에서 폐기) |
| 19 | Task Spec / Task Report 표준화 | Maestro → Foreman 명세서 + 보고서 스키마. 사용자에게는 Maestro 가공본만 surface. |
| 20 | 엔티티 쓰기 권한 분리 | board/* 본문 쓰기 = Maestro 전용. Foreman은 status 필드만. Sentinel/워커는 backlog item 자동 생성. |
| 21 | 우선순위 자동 트리거 | high severity bug 자동 P0/P1, BLOCK ≥5회 자동 tech_debt item 생성, 재발 lesson 자동 proposal. |
| 22 | Phase 1의 Foreman 제거 | PRD/design-spec 수신 주체 Maestro. (v0.5에서 Planning 단계로 흡수) |
| 23 | milestone 단위 종료 조건 | Ralph 종료는 milestone 단위. (v0.5에서 4-step Review 단계로 재해석) |

---

## v0.3 (2026-05-20)

### 변경 요약
- **TDD 강제** (P9 신규): 코드 작업 워커는 RED→GREEN→REFACTOR 사이클 무조건. 워커 내부 step 5→7로 확장. Sentinel `tdd_violation` 룰 P0. Designer/Strategist/Tech-Writer 등 prose 작업은 명시적 제외.
- **Compound 단계 신규** (P10): 모든 워커 task 끝, 모든 Phase 끝, 모든 Ralph 사이클 끝에 **3-tier compound** 단계 추가. 발생한 실수·교훈을 `.harness/lessons/`로 영구화하여 다음 사이클에 재사용.
- **하네스 자기 진화** (P11 신규 + Phase 8 신규): Agent-Architect 워커가 `sentinel-log` + `lessons` + `proposals`를 주기 분석하여 신규 skill/hook/agent/룰을 **제안 문서**로 작성.
- 워커 추천 빈도 차이 반영 (코드 vs prose 작업의 step 차등).

### 결정 사항

| # | 항목 | 결정 |
|---|---|---|
| 9 | TDD 강제 | 코드 워커는 RED→GREEN→REFACTOR 의무. Sentinel `tdd_violation` P0. Prose 워커 명시적 제외. spike/hotfix는 `/tdd-exception` 게이트. |
| 10 | Compound 단계 추가 | 모든 워커 task / Phase / Ralph 사이클 끝에 3-tier compound. lessons는 `.harness/lessons/<cat>/L-*.md` 영구화. Tech-Writer Lead. |
| 11 | 하네스 자기 진화 | Agent-Architect가 sentinel-log + lessons 분석 → `.harness/proposals/*.md` 작성. Maestro HITL 채택. 채택 시 실 파일 수정. |
| 12 | 워커별 step 차등 | 코드 워커 7-step, prose 워커 5-step, 메타 워커 6-step. 균일 구조 강제 X. |
| 13 | TDD 예외 마커 경로 | plan에 `tdd_exception: "<사유>"` + Maestro 승인 마커. 예외 사용 시 사후 회귀 테스트 추가 의무. |
| 14 | Lesson 재사용 자동 inject | Librarian이 task 시작 시 `.harness/lessons/`를 모듈/카테고리/태그로 grep → 컨텍스트 주입. ETHOS 승급 임계: 5회 hit. |
| 15 | 진화 제안 메타-회귀 방지 | 채택된 룰은 N 사이클 효과 추적 → 0이면 deprecate 후보. ETHOS 추가는 3 사이클 검증. |

---

## v0.2 (2026-05-19)

### 변경 요약
- 결정사항 8개 모두 확정 (12절 → "Resolved Decisions"로 갱신)
- L2 워커에 **DevOps / QA / Tech Writer** 3종 추가 (총 10직책)
- Sentinel에 **아키텍처 위반 점검** 추가, over_engineering & dead_code를 최우선 룰로 승격
- **모드 선택** 기능 신규 (`auto` / `gated` / `plan-only` / `dry-run`)
- 타겟 스택 픽스: TypeScript · Tailwind · shadcn/ui · Python · Supabase · Postgres · PocketBase · Cloudflare
- **모노레포** 전제 명시, 디렉터리 컨벤션 보강

### 결정 사항

| # | 항목 | 결정 |
|---|---|---|
| 1 | 네이밍 | **Maestro / Foreman / Sentinel** 그대로 유지 |
| 2 | 타겟 스택 | TypeScript · Tailwind · shadcn · Python · Supabase · Postgres · PocketBase · Cloudflare |
| 3 | 레포 구조 | 모노레포 (`apps/`, `packages/`, `infra/`) |
| 4 | Sentinel 룰 우선순위 | **over_engineering & dead_code = P0**, **아키텍처 위반 신규 추가 (P1)** |
| 5 | HITL 게이트 | 결정사항 발생 시점만, DAG/실행은 자동, **모드 선택 기능 신규** |
| 6 | 비용 상한 | MVP 이후로 보류 |
| 7 | L2 워커 추가 | **DevOps + QA + Tech Writer 3종 추가** (총 10직책) |
| 8 | Worker 내부 3-step | `task()` 별도 세션으로 분리 (컨텍스트 격리 강제) |

---

## v0.1 (2026-05-19)

### 변경 요약
- 초안. 3계층 (Maestro / Foreman / Sentinel) + L2 워커 7직책 + Phase 0~7 워크플로 + 기본 Sentinel 룰셋.

---

## 의사결정 인덱스 (누적 34건, 최신순)

| # | 핵심 | 도입 버전 |
|---|---|---|
| 34 | Phase 마커 STATE.md 위치 | v0.5 |
| 33 | 가치×시급성 priority 매트릭스 | v0.5 |
| 32 | BacklogItem 타입 카탈로그 | v0.5 |
| 31 | Discovered-not-logged 강제 | v0.5 |
| 30 | DoD 측정 가능성 자동 검증 | v0.5 |
| 29 | Flexibility 룰 (P16) | v0.5 |
| 28 | DoD 의무화 (P15) | v0.5 |
| 27 | 4-step 사이클 강제 (P14) | v0.5 |
| 26 | Issue 엔티티 흡수 | v0.5 |
| 25 | Backlog = SSOT (P13) | v0.5 |
| 24 | 6-tier PM 계층 | v0.5 |
| 23 | milestone 단위 종료 조건 | v0.4 → v0.5 재해석 |
| 22 | Phase 1의 Foreman 제거 | v0.4 → v0.5 재해석 |
| 21 | 우선순위 자동 트리거 | v0.4 |
| 20 | 엔티티 쓰기 권한 분리 | v0.4 |
| 19 | Task Spec / Report 표준화 | v0.4 |
| 18 | 엔티티 ID 컨벤션 | v0.4 → v0.5 변경 |
| 17 | PM 모델 | v0.4 → v0.5 재정렬 |
| 16 | Maestro = 단일 사용자 창구 | v0.4 |
| 15 | 진화 제안 메타-회귀 방지 | v0.3 |
| 14 | Lesson 재사용 자동 inject | v0.3 |
| 13 | TDD 예외 마커 경로 | v0.3 |
| 12 | 워커별 step 차등 | v0.3 |
| 11 | 하네스 자기 진화 | v0.3 |
| 10 | Compound 단계 추가 | v0.3 |
| 9 | TDD 강제 | v0.3 |
| 8 | Worker 내부 task() 분리 | v0.2 |
| 7 | L2 워커 DevOps + QA + Tech Writer 추가 | v0.2 |
| 6 | 비용 상한 MVP 이후 | v0.2 |
| 5 | HITL 게이트 + 모드 선택 | v0.2 |
| 4 | Sentinel 룰 우선순위 | v0.2 |
| 3 | 모노레포 | v0.2 |
| 2 | 타겟 스택 확정 | v0.2 |
| 1 | 네이밍 Maestro/Foreman/Sentinel | v0.2 |
