# 04. L2 — 워커 프로필과 내부 step flow

> 워커 프로필은 프로젝트 적응형 실행 직책이다. 하네스는 seed 프로필을 함께 제공하지만, 실제 활성 집합은 프로젝트 증거를 통해 진화한다.

---

## 4.1 워커 프로필 모델

워커 프로필은 영구적 제품 직책이 아니다. 다음 위치의 파일이다:

```text
.opencode/agents/worker-profiles/*.md
```

각 프로필이 정의하는 것:

| 필드 | 의미 |
|---|---|
| `id` | 안정된 프로필 이름 |
| `use_when` | Foreman이 언제 할당해야 하는가 |
| `allowed_paths` | 파일/경로 경계(있는 경우) |
| `step_flow` | 코드 7-step, 산문 5-step, 또는 진화된 커스텀 flow |
| `verify_commands` | 프로필 고유 검증 |
| `review_lens` | 이 프로필이 가장 잘 잡아내는 것 |
| `model_category` | 선호 모델 카테고리와 fallback |
| `evolution_notes` | 시간에 걸친 변경 채택/거부 이력 |

---

## 4.2 Seed 프로필

| 프로필 | 사용 시점 | Step Flow | TDD |
|---|---|---|---|
| Planning | 요구사항, 범위, AC, 우선순위 | 산문/spec 5-step | 아니오 |
| Design | UX, 시스템 설계, 인터페이스 설계, 아키텍처 스케치 | 산문/spec 5-step | 아니오 |
| Implementation | 메인 소스 트리의 코드 변경 | 코드 7-step | 예 |
| Data | 스키마, 마이그레이션, 파이프라인, 저장 모델 | 코드 7-step + dry-run/rollback plug | 예 |
| Security | auth, authorization, 입력 검증, 시크릿, 위협 모델 | 코드 7-step + audit plug | 예 |
| Quality | 테스트, 회귀 스위트, 커버리지, E2E/통합 검증 | 코드 7-step | 예 |
| Ops | CI/CD, 릴리스, 런타임, 옵저버빌리티, 운영 자동화 | 코드 7-step + smoke/rollback plug | 예 |
| Documentation | README, ADR, changelog, 사용자 문서, lesson | 산문/spec 5-step | 아니오 |

프로젝트 진화 예시:

- 웹 프로젝트는 Implementation을 `ui-implementation`과 `api-implementation`으로 분리
- 라이브러리 프로젝트는 `public-api`와 `compatibility` 추가
- 성능 중요 프로젝트는 `benchmark` 추가
- 작은 CLI 프로젝트는 Data/Ops를 Implementation으로 병합

---

## 4.3 코드 워커 7-Step

코드를 다루는 프로필에 적용된다.

```text
Worker
  step:plan
  step:red
  step:green
  step:refactor
  step:spec-review
  step:quality-review
  step:commit
  step:compound
```

게이트 상세:

| Step | 필수 증거 |
|---|---|
| plan | 파일, AC, 제약, placeholder 없음 |
| red | 실패하는 테스트 또는 승인된 TDD 예외 |
| green | 최소 구현과 새 테스트 통과 |
| refactor | 모든 테스트 여전히 통과 |
| spec-review | AC/spec 충족, 별도 task 컨텍스트 |
| quality-review | 단순성, 중복, 스타일, 경계 점검 |
| commit | atomic 커밋 + Self-Check 블록 |
| compound | lesson 후보 또는 사유와 함께 "lesson 없음" 명시 |

Self-Check 블록:

```markdown
## Self-Check
- Files claimed: <list> -> EXISTS verified
- Commits claimed: <hashes> -> git log verified
- Tests added/modified: <count>
- Verify commands run: <cmd> -> exit code
- Result: PASSED|FAILED
```

TDD 예외 요건:

- plan 안에 `tdd_exception: "<reason>"`
- Maestro Core 승인
- 후속 회귀 또는 동등 증거

허용 카테고리: spike/POC, 후속 회귀 동반 hot-fix, config-only 변경, RED를 dry-run으로 대체하는 데이터/스키마 마이그레이션.

---

## 4.4 산문/spec 워커 5-Step

Planning, Design, Documentation 등 산문 작업 프로필에 적용된다.

```text
Worker
  step:research
  step:draft
  step:peer-review
  step:revise
  step:compound
```

룰:

- `step:draft`에는 TBD/TODO/placeholder 표현 금지
- 계획/spec을 산출할 때 수용 기준은 구체적이어야 함
- peer review는 별도 task 컨텍스트
- 산문/spec이 향후 실행 행동을 변경시킨 경우에만 lesson 발행

---

## 4.5 메타 워커: Agent-Architect

Agent-Architect는 코어 메타 워커다. 변경을 **제안**할 뿐, 하네스를 조용히 변경하지 않는다.

```text
Agent-Architect
  step:diagnose
  step:baseline
  step:design
  step:propose
  step:peer-review
  step:compound
```

입력:

- `sentinel-log.jsonl`
- lessons
- 채택/거부된 proposal
- 반복되는 워커 실패
- 사용자 선호

출력:

- 워커 프로필 추가/분리/병합/폐기 제안
- Sentinel 룰 추가/갱신 제안
- 프로젝트 컨벤션 갱신 제안
- skill/hook/prompt 변경 제안

채택된 변경은 `evolved: true`로 태그되고 효과가 모니터링된다.

---

## 4.6 컨텍스트 격리

워커 step은 별도 task 컨텍스트에서 실행된다. 각 step은 이전 step에서 선언된 입력만 받는다.

상위 계층은 컨텍스트 전체가 아니라 요약 JSON만 받는다:

```json
{
  "u_id": "U-007",
  "worker_profile": "implementation",
  "status": "done",
  "commit": "abc123",
  "tests_added": 4,
  "files_touched": 3,
  "sentinel": "pass",
  "compound_emitted": "L-2026-05-20-003"
}
```

---

## 4.7 모델 할당

기본 모델 선택은 튜닝 가능하고 프로젝트 적응적이다:

| 프로필 | 기본 카테고리 | 이유 |
|---|---|---|
| Maestro Core | 고추론 대화 | 사용자 인터랙션과 최종 판단 |
| Board Clerk | 작성/구조화 | schema-safe PM 쓰기 |
| Milestone Planner | 작성/추론 | 우선순위와 의존성 추론 |
| Spec Writer | 작성/구조화 | 정밀한 task spec |
| Report Editor | 빠름/작성 | 간결한 요약 |
| Context Librarian | 빠름/검색 | 검색 요약 |
| Foreman | 중간 추론 | DAG와 머지 조율 |
| Sentinel 1차 패스 | 빠름 | 잦은 저비용 점검 |
| Sentinel 2차 패스 | 고추론 | 의미 판단 |
| Agent-Architect | 고추론 | 메타 설계 |
| 워커 프로필 | `PROJECT_PROFILE`에서 | hit-rate와 품질로 튜닝 |
