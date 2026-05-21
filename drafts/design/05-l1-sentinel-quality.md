# 05. L1 — Sentinel 품질 및 워크플로 게이트

> Sentinel은 상시 가드 계층이다. 코어 룰을 강제하고, 프로필 파일에서 프로젝트/워커 룰을 읽어와 PASS/FIX/ASK/BLOCK을 반환한다.

---

## 5.1 트리거 지점

| 트리거 | 시점 |
|---|---|
| post-commit | 모든 워커 커밋 |
| phase boundary | Refinement, Planning, Execution, Review 진입/이탈 |
| merge gate | Foreman이 worktree/wave를 머지하기 전 |
| DoD verify | 마일스톤 Review |
| explicit call | `/sentinel-check [scope]` |

---

## 5.2 결과 타입

| 결과 | 의미 |
|---|---|
| PASS | 진행 |
| FIX | Sentinel이 기계적 패치를 생성. 워커가 적용 후 재실행 |
| ASK | 판단 필요. Maestro Core로 batch |
| BLOCK | 현재 경로 중단, 수정/재계획 요구 |

심각도 레벨:

| 레벨 | 기본 동작 |
|---|---|
| P0 | 항상 점검, 즉시 차단 가능 |
| P1 | 자주 점검, 머지/페이즈 게이트에서 강제 |
| P2 | 권고 또는 머지 게이트 전용 |

---

## 5.3 코어 룰

다음은 스택 독립이며 항상 활성이다:

| 룰 | 목적 |
|---|---|
| `over_engineering` | 시기상조 추상화와 투기적 아키텍처 방지 |
| `dead_code` | 미사용·도달 불가·주석 처리 코드 제거 |
| `scope_creep` | 변경 파일은 task plan과 워커 프로필에 부합해야 함 |
| `tdd_violation` | RED/GREEN/REFACTOR 증거 또는 승인된 예외 없이 코드 머지 금지 |
| `compound_required` | task/페이즈/사이클 학습은 반드시 발행 |
| `backlog_singularity` | todo/작업 항목은 backlog에 있어야 함 |
| `discovered_not_logged` | 새 발견은 backlog item이 되어야 함 |
| `workflow_phase_skip` | Refinement → Planning → Execution → Review는 건너뛸 수 없음 |
| `dod_required` | 마일스톤 생성/완료 전 측정 가능한 DoD 필수 |
| `analysis_paralysis` | 진척 없이 read-only 반복 시 중단하거나 차단 상태를 자백해야 함 |
| `slopcheck` | 환각/안전하지 않은 의존성 차단 |
| `self_check_required` | 완료 주장은 파일/커밋/verify 증거 필요 |
| `plan_placeholder` | TBD/TODO/모호한 계획 차단 |
| `flexibility_traceability` | 계획 변경 시 링크 갱신과 결정 컨텍스트 append |

`[NEVER_GATE]` 룰은 hit-rate가 낮더라도 자동 비활성화되지 않는다:

- 보안 민감 점검
- 아키텍처 경계
- TDD
- DoD
- Backlog SSOT
- phase 순서
- slopcheck
- self-check
- plan placeholder

---

## 5.4 프로젝트 및 워커 룰

Sentinel은 제품 유형, 스택, 레포 레이아웃을 하드코딩하지 않는다.

다음을 읽는다:

- `.harness/PROJECT_PROFILE.md`
- `.harness/CONVENTIONS.md`
- 활성 `worker-profiles/*.md`

예시:

| 프로필 슬롯 | 용도 |
|---|---|
| `architecture_rules[]` | 계층 경계, 의존성 방향 |
| `runtime_constraints[]` | 선언된 런타임에 대한 금지 API/의존성 |
| `format_commands[]` | 포매팅 강제 |
| `lint_commands[]` | lint 강제 |
| `typecheck_commands[]` | 타입/계약 강제 |
| `test_run_commands[]` | TDD와 DoD 검증 |
| `security_scan_commands[]` | 프로젝트 고유 보안 도구 |
| `coverage_commands[]` | 선택적 커버리지 게이트 |

필요한 슬롯이 비어 있으면, Sentinel은 숨겨진 기본값을 만들어내지 않고 ASK 또는 Phase 8 proposal을 발행한다.

---

## 5.5 2-Tier 비용 통제

| 티어 | 실행 시점 | 메커니즘 |
|---|---|---|
| Tier 1 cheap | 매 커밋 | 프로필 선언 정적 도구 + P0 패턴 점검 |
| Tier 2 expensive | 의심스러운 발견, 머지 게이트, DoD verify | 더 강한 모델로 의미 리뷰 |

Tier 2는 다음 같은 질문을 위한 것이다:

- 이 경계 위반이 의미가 있는가?
- 이 추상화가 정당한가?
- 실데이터 흐름을 만족하는가?
- 이 의존성을 수용할 만큼 안전한가?

---

## 5.6 AUTO-FIX 대 ASK

AUTO-FIX 예시:

- formatter/linter 기계 수정
- 사용 안 하는 import
- 명백한 dead code
- import 순서
- 컨벤션이 명확할 때의 네이밍 케이스

ASK 예시:

- 모호한 네이밍 트레이드오프
- 아키텍처 대안
- 비즈니스/도메인 의미
- 사용자 허용도가 필요한 의존성 리스크
- 의도된 것일 수 있는 룰 위반

ASK는 Maestro Core로 batch된다.

---

## 5.7 4-Level Goal-Backward Verifier

DoD verify, 워커 spec-review, 머지 게이트에서 사용된다.

```text
Level 1: EXISTS
  파일, import, 명령, 커밋이 존재함

Level 2: SUBSTANTIVE
  stub, dummy return, fake test, 빈 구현 없음

Level 3: WIRED
  엔트리포인트, 라우트, export, 호출자, config, 문서가 연결됨

Level 4: REAL DATA FLOW
  실데이터 또는 실행 경로가 목표 동작을 증명함
```

기본 입장:

> 통과 증거가 없는 것은 실패다.

출력 모양:

```text
Level 1 EXISTS: PASS
Level 2 SUBSTANTIVE: PASS
Level 3 WIRED: FAIL - <연결되지 않은 엔트리포인트>
Level 4 REAL DATA FLOW: SKIPPED
```

---

## 5.8 적응형 게이팅

Sentinel은 룰별 hit-rate를 추적한다.

구성된 윈도우 동안 발견이 없는 룰은 `[NEVER_GATE]` 태그가 없는 한 강등될 수 있다. 강등 자체도 가시화되고, Phase 8에서 리뷰될 수 있다.

반복되는 BLOCK은 다음을 만들어낼 수 있다:

- backlog item
- 컨벤션 갱신 proposal
- 워커 프로필 분리/병합 proposal
- Sentinel 룰 개선 proposal
