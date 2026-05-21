# 06. 계층 횡단 워크플로

> 이 문서는 L0, L1, L2, Sentinel, 진화 계층을 가로지르는 워크플로를 설명한다. 계층 내부 디테일은 02~05 문서에 있다.

---

## 6.1 Run Mode

| 모드 | 동작 | HITL |
|---|---|---|
| `auto` | 사용자 인터뷰 후 자율 사이클 진행 | 결정 분기만 |
| `gated` | 사용자가 각 페이즈 경계에서 확인 | 모든 페이즈 |
| `plan-only` | Refinement + Planning만, 코드 작업 없음 | 플래닝 완료 시 |
| `dry-run` | 페이즈와 게이트 시뮬레이션, 머지/배포 없음 | 결정 분기만 |

모드 무관 HITL:

- 비가역 작업
- 높은 보안 리스크
- 동일 Sentinel BLOCK 3회 반복
- 불확실한 외부 의존성
- Vision/DoD 변경
- 예산 임계 초과

---

## 6.2 Phase 0: 프로젝트 부트스트랩

```text
사용자 목표
  -> Maestro Core 인터뷰
  -> Context Librarian / Explore / Oracle 리서치
  -> PROJECT_PROFILE.md 초안
  -> CONVENTIONS.md 초안
  -> Vision + Roadmap + 첫 Milestone + DoD
  -> 초기 Backlog
  -> 사용자 승인
  -> 첫 Refinement
```

Phase 0 산출물:

- `.harness/PROJECT_PROFILE.md`
- `.harness/CONVENTIONS.md`
- `board/vision.md`
- `board/roadmap.md`
- 첫 `board/milestones/M-*.md`
- `board/backlog/_index.md`
- 초기 backlog item

---

## 6.3 마일스톤 4-Step 사이클

```text
Refinement -> Planning -> Execution -> Review -> 다음 Refinement
```

### Refinement

목적: backlog를 단일 진실 저장소로 유지한다.

흐름:

1. Context Librarian이 활성 마일스톤과 backlog brief
2. Milestone Planner가 triage, 우선순위, dedup, 링크 제안
3. Maestro Core 승인 또는 사용자 질의
4. Board Clerk 변경 작성
5. Sentinel이 Backlog SSOT와 페이즈 마커 점검

### Planning

목적: 마일스톤 작업을 선택하고 검증 가능하게 만든다.

흐름:

1. Milestone Planner가 마일스톤 범위와 DoD 제안
2. 필요 시 Planning/Design 워커가 요구사항/spec 보강
3. Spec Writer가 task 분해 제안
4. Maestro Core 승인
5. Board Clerk가 M/B/T 링크 작성
6. Sentinel이 DoD, 페이즈 순서, placeholder plan 점검

### Execution

목적: Foreman을 통해 승인된 task 실행.

흐름:

1. Maestro Core가 준비된 task 선택
2. Spec Writer가 Task Spec 초안 작성
3. Foreman이 DAG와 worktree 구성
4. 워커 프로필이 step flow 실행
5. Sentinel이 커밋·머지 게이트에서 동작
6. Foreman이 Task Report 반환
7. Report Editor가 요약
8. Maestro Core가 결과를 사용자에게 surface

### Review

목적: 마일스톤 완료를 증명하고 학습한다.

흐름:

1. DoD verify 명령 실행
2. Sentinel 4-level verifier가 증거 평가
3. 실패한 DoD는 backlog 작업 생성
4. Tier 3 compound가 횡단 lesson 추출
5. 다음 마일스톤/backlog 우선순위 조정
6. Maestro Core가 보고 후 진행

---

## 6.4 Compound 학습

| Tier | 시점 | 출력 |
|---|---|---|
| Tier 1 | 워커 task 종료 | lesson 후보 |
| Tier 2 | 페이즈 경계 | dedup된 lesson markdown |
| Tier 3 | 마일스톤 Review | 시스템 수준 패턴과 ETHOS 후보 |

Lesson 모양:

```yaml
symptom:
root_cause:
what_worked:
prevent_next_time:
related_u_ids:
category:
```

Lesson은 이후 task에서 Context Librarian과 워커 프로필 프롬프트가 소비한다.

---

## 6.5 Phase 8 하네스 진화

트리거:

- 동일 Sentinel 발견 반복
- 사이클을 가로지르는 동일 lesson 재발
- 동일 사용자 선호 반복
- 워커 프로필이 회피 가능한 실패를 만들어냄
- proposal 채택/거부 패턴에서 drift 신호

Agent-Architect 흐름:

```text
diagnose -> baseline -> design -> propose -> peer-review -> compound
```

Proposal 대상:

- 워커 프로필 추가/분리/병합/폐기
- Sentinel 룰 변경
- 컨벤션 레지스트리 갱신
- 프로젝트 프로필 갱신
- skill/hook/prompt 개선
- ETHOS 후보

채택:

- Maestro Core가 proposal을 surface
- 사용자가 수락/거부/보류
- 채택된 변경은 `evolved: true` 태그
- 향후 폐기 가능성을 위해 효과 측정

---

## 6.6 Ralph 루프 종료

다음 조건이 모두 충족될 때만 종료한다:

- 현재 범위의 모든 P0/P1 task가 done 또는 의도적 보류
- 모든 마일스톤 DoD 점검 통과
- Sentinel에 미해결 BLOCK 없음
- 통합/보안/품질 게이트가 프로젝트 프로필에 따라 통과
- 열린 P0/P1 backlog item이 0 또는 명시적 보류
- Tier 3 compound 작성됨
- 사용자가 완료를 거부하지 않음

미완료 시:

- 미완 task -> Foreman 재투입
- 실패한 DoD -> backlog item/task
- 누락된 룰/프로필/컨벤션 -> Phase 8 proposal
- 모호한 결정 -> Maestro Core HITL

---

## 6.7 횡단 불변식

| 불변식 | 강제 주체 |
|---|---|
| Maestro Core만 사용자 대면 | P2 + 런타임 채널 격리 |
| 페이즈 건너뛰기 금지 | Sentinel `workflow_phase_skip` |
| 숨은 todo 금지 | Sentinel `backlog_singularity` / `discovered_not_logged` |
| 증거 없는 코드 금지 | TDD + Self-Check + 4-level verifier |
| 암묵적 범위 확장 금지 | Task Spec 제약 + Sentinel `scope_creep` |
| 학습은 의무 | Compound 게이트 |
| 진화는 명시적 | proposal 워크플로 |
