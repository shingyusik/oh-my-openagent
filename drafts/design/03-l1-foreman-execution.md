# 03. L1 — Foreman 실행 계층

> Foreman은 승인된 Task Spec을 병렬 작업, worktree 격리, 머지 순서, Task Report로 풀어낸다. 사용자 대면 컴포넌트가 아니다.

---

## 3.1 역할

Foreman이 소유하는 것:

- Task Spec 수신
- DAG 구성
- 워커 프로필 할당
- worktree 생성과 정리
- wave 실행과 머지 순서
- 커밋/머지 주변의 Sentinel 호출
- Task Report 생성

Foreman이 하지 않는 것:

- 사용자와의 대화
- backlog 직접 재작성
- 제품 우선순위 결정
- 구조화된 에스컬레이션 필요가 없는 한 워커 컨텍스트 전체 열람

---

## 3.2 입력: Task Spec

Spec Writer가 Task Spec을 초안하고, Maestro Core가 승인한 뒤 Foreman이 받는다.

필수 필드:

| 섹션 | 필수 내용 |
|---|---|
| Identity | `T-id`, backlog item, milestone, priority, mode |
| Goal | task 설명과 수용 기준 |
| Constraints | 허용 경로, 금지 경로, 프로젝트 프로필 제약, TDD flag |
| Dependencies | blocked-by / blocks 관계 |
| Worker allocation hints | 권장 워커 프로필 |
| Lessons to consult | 관련 lesson ID |
| Report back | 기대 Task Report schema와 에스컬레이션 규칙 |

워커 할당 예시는 고정 제품 직책이 아니라 프로필명으로 표기한다:

```yaml
suggested_worker_allocation:
  - implementation
  - data
  - quality
  - documentation
```

---

## 3.3 DAG 빌더

Foreman은 Task Spec을 안정 unit ID로 분해한다:

```text
T-101
  U-001 data 경계 또는 셋업
  U-002 implementation
  U-003 품질 검증
  U-004 문서화
```

DAG 규칙:

| 규칙 | 이유 |
|---|---|
| 유닛당 안정된 U-ID | plan → commit → report → lesson 추적 |
| 같은 파일/함수는 같은 wave에서 동작 불가 | 머지 충돌 churn 방지 |
| Data/schema 경계 변경은 의존 implementation보다 선행 | false green 방지 |
| Design/spec 작업은 의존 implementation보다 선행 | 누락 spec을 상대로 build 방지 |
| 보안 민감 리뷰는 implementation 이후 | 추측이 아니라 실코드를 리뷰 |
| 품질 검증은 implementation 이후. 단, RED step이 필요하면 테스트가 먼저 작성될 수 있음 | TDD 유지 |

---

## 3.4 Worktree 실행

```text
project/
  .worktrees/
    implementation-T-101-U-001/
    data-T-101-U-002/
    quality-T-101-U-003/
```

경로 포맷:

```text
.worktrees/<worker-profile>-<T-id>-<U-id>
```

모든 worktree는 커밋 전에 다음을 검증한다:

- `HEAD == expected_head`
- `cwd == expected_worktree_path`
- 변경 파일은 plan에 기록된 touched files 내부
- untracked 파일은 plan에 명시되어 있음

worktree 시작 후 금지 명령:

- `git stash`
- `git reset --hard`
- `git clean -fd`
- 보호된 브랜치 ref 재작성
- 보호된 ref로 force push

`git stash`는 worktree 경계를 조용히 넘어가기 때문에 금지한다.

---

## 3.5 워커 핸드오프

Foreman은 각 워커에게 다음만 전달한다:

- unit ID
- 승인된 Task Spec의 슬라이스
- 관련 프로젝트 프로필/컨벤션 발췌
- 필요한 lesson
- 기대 step 출력

Foreman은 raw chain-of-thought이나 작업 컨텍스트 전체가 아니라 구조화된 요약만 받는다.

워커 요약 모양:

```json
{
  "u_id": "U-003",
  "worker_profile": "quality",
  "status": "done",
  "commit": "abc123",
  "tests_added": 4,
  "files_touched": 3,
  "sentinel": "pass",
  "compound_emitted": "L-2026-05-20-003"
}
```

---

## 3.6 머지와 Sentinel 게이트

Foreman은 wave 단위로 완료된 작업을 머지한다:

1. 완료된 유닛 요약 수집
2. 각 커밋/worktree 결과에 Sentinel 실행
3. 허용된 곳에서 AUTO-FIX 패치 적용
4. 필요 시 ASK 항목 차단 또는 에스컬레이션
5. 대상 브랜치로 wave 머지
6. merge-gate Sentinel과 DoD 관련 체크 실행
7. worktree 정리 또는 정책에 따라 보존

패치 dedup:

```text
patch_hash = sha256(diff)
이미 적용된 해시면:
  중복 커밋 skip
아니면:
  커밋하고 해시 기록
```

---

## 3.7 출력: Task Report

Foreman은 Report Editor와 Maestro Core에 raw Task Report를 반환한다:

| 섹션 | 내용 |
|---|---|
| Outcome | status, 커밋, worktree, 대상 브랜치, 소요 시간/공수 |
| AC verification | 수용 기준과 verifier 증거 |
| New backlog items | 자동 생성된 발견 |
| Lessons captured | task 수준 lesson ID |
| Sentinel summary | PASS/FIX/BLOCK/ASK |
| TDD compliance | 추가/수정된 테스트, 예외 |
| Compound summary | 발생/접힌 lesson |
| Suggested next actions | 해제된 task, 리스크, 질의 |

Maestro Core로 에스컬레이션하는 시점:

- 동일 Sentinel BLOCK이 3회 반복
- 신규 P0/P1 backlog item 발생
- 공수가 견적 임계를 초과
- 비가역 작업이 필요
- Task Spec이 실질적으로 잘못된 경우
