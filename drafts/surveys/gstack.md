# Survey: garrytan/gstack

> Source: https://github.com/garrytan/gstack
> Author: Garry Tan
> Type: Skill library + multi-host adapter layer
> 분석 시점: 2026-05-20

---

## 1. 개요

`gstack`은 **자율 오케스트레이터가 아니라 사람이 호출하는 skill pack**. ~50개 skill을 통해 CEO / Engineering Manager / Designer / QA / Release Manager 역할을 시뮬레이션하고, **11개 host adapter**(Claude Code, OpenCode, Cursor, Codex, Factory Droid 등)를 통해 단일 source를 여러 런타임에 컴파일.

핵심 특징:
- **Skill = directory** (`SKILL.md` generated + `SKILL.md.tmpl` source-of-truth)
- **Host adapter pattern**으로 multi-runtime 지원
- **영구 Chromium daemon** (browse) for real-browser QA
- **Conductor 통합** (별도 paid 도구와 협업)
- ETHOS 파일이 모든 skill의 preamble에 symlink로 주입

---

## 2. 철학 / 핵심 원칙

`ETHOS.md`로 명시된 룰:

| 원칙 | 의미 |
|---|---|
| **Boil the Lake** | small-iter scope. 한 번에 전체를 바꾸지 말고 핵심 1개만 끓이기. |
| **Search Before Building** | 새로 만들기 전에 기존 코드/패턴/라이브러리 검색 의무. |
| **User Sovereignty** | 사용자가 결정한다. agent는 옵션 제시, 사용자가 선택. |

운영 원칙:
- **Adaptive specialist gating**: hit-rate가 0인 specialist는 자동 비활성. security/data-migration은 `[NEVER_GATE]` (절대 비활성 X).
- **Fix-First Review**: 모든 finding을 `[AUTO-FIX]` vs `[ASK]`로 분류. 기계적 fix는 즉시 적용.
- **Confidence-scored findings**: 1-10 점수. <7은 caveat 처리, 3-4는 appendix-only.

---

## 3. 아키텍처 / 구조

```
gstack/
├── ETHOS.md                       (모든 skill preamble에 symlink로 주입)
├── ARCHITECTURE.md
├── SKILL.md                       (메타 skill 작성 가이드)
├── SKILL.md.tmpl                  (위의 source)
├── package.json                   (Bun runtime)
├── bin/                           (글로벌 install 스크립트)
├── lib/
│   ├── worktree.ts                (detached worktree + SHA256 dedup)
│   ├── slop-scan.ts               (패키지 정합성)
│   └── ...
├── hosts/                         (11개 host adapter, 각 ~30 LOC)
│   ├── claude-code.ts
│   ├── opencode.ts
│   ├── cursor.ts
│   ├── codex.ts
│   ├── factory.ts                 (Factory Droid)
│   ├── continue.ts
│   ├── aider.ts
│   └── ...
├── skills/                        (~50 skills)
│   ├── office-hours/SKILL.md.tmpl
│   ├── autoplan/SKILL.md.tmpl
│   ├── review/
│   │   ├── SKILL.md.tmpl
│   │   ├── checklist.md
│   │   └── specialists/
│   │       ├── testing.md
│   │       ├── maintainability.md
│   │       ├── security.md
│   │       ├── performance.md
│   │       ├── data-migration.md
│   │       ├── api-contract.md
│   │       └── red-team.md
│   ├── ship/SKILL.md.tmpl
│   ├── learn/SKILL.md.tmpl
│   ├── browse/                   (Chromium 데몬 binary 포함)
│   └── ...
├── tools/
│   └── gen-skills.ts             (host별 컴파일러)
├── slop-scan.config.json
└── conductor.json                (Conductor 병렬 sprint 통합)
```

`SKILL.md`는 generated이므로 **직접 수정 금지**. `SKILL.md.tmpl`만 편집 후 `bun run gen:skill-docs` 실행.

---

## 4. 컴포넌트 카탈로그

### 4.1 Host Adapter Pattern

`hosts/*.ts`는 단일 `HostConfig` shape를 export:

```typescript
export const config: HostConfig = {
  pathRewrites: {
    "~/.claude/skills": "~/.config/opencode/skills",
  },
  toolRewrites: {
    "use the Bash tool": "run this command",
    "Read tool": "read",
  },
  frontmatter: {
    allow: ["name", "description", "model", "tools"],
    deny: ["allowed-tools"],
  },
  suppressedResolvers: ["windows-only-helper"],
  runtimeRoot: {
    globalSymlinks: ["~/.gstack-dev/ETHOS.md → ./ETHOS.md"],
  },
};
```

`gen-skills.ts`가 모든 `SKILL.md.tmpl`을 각 host config에 따라 변환 → host별 `SKILL.md` 생성.

### 4.2 Skills (~50개, 카테고리별)

#### 계획 (6)
| Skill | 책무 |
|---|---|
| **/office-hours** | 6개 forcing question으로 아이디어 압박 (Garry의 YC톤 그대로). |
| **/autoplan** | CEO → design → eng → DX 4단계 자동 파이프라인. |
| **/plan-ceo-review** | CEO 관점 리뷰 (비전·범위·우선순위). |
| **/plan-eng-review** | Engineering 관점 (구현 난이도, 기술 부채). |
| **/plan-design-review** | Design 관점 (UX, 일관성). |
| **/plan-devex-review** | Developer experience (테스트, 디버깅 용이성). |
| **/plan-tune** | 위 4개 리뷰 결과 종합해 plan 조정. |

#### 리뷰 (3 + 7 specialists)
| Skill | 책무 |
|---|---|
| **/review** | 메인 리뷰 dispatcher. specialist sub-agent들을 적응형으로 dispatch. |
| **/cso** | OWASP Top 10 + STRIDE 위협 모델 통합 보안 리뷰. |
| **/codex** | OpenAI Codex로 cross-model second opinion. |
| **(specialists 7)** | testing / maintainability / security / performance / data-migration / api-contract / red-team. 각각 review/specialists/*.md. |

#### 마감/배포 (3)
| Skill | 책무 |
|---|---|
| **/ship** | 자동 VERSION bump + git diff → changelog 생성 + bisectable commit 분할. |
| **/land-and-deploy** | merge + deploy 파이프라인. |
| **/document-release** | release note + changelog markdown. |

#### QA (3)
| Skill | 책무 |
|---|---|
| **/qa** | 일반 QA dispatch. |
| **/qa-only** | QA만, 다른 단계 skip. |
| **/browse** | 영구 Chromium daemon에 접속. ARIA `@e1` 등 reference로 element 식별. ring-buffered 로그. |

#### 안전장치 (4)
| Skill | 책무 |
|---|---|
| **/careful** | 다음 호출에 conservative 모드 적용 (delete/migration/prod-critical 조심). |
| **/freeze** | 변경 동결 (release candidate 등). |
| **/unfreeze** | 동결 해제. |
| **/guard** | 특정 파일/디렉토리 보호 룰 등록. |

#### 메모리 (3)
| Skill | 책무 |
|---|---|
| **/learn** | **typed memory** 저장. 카테고리: patterns / pitfalls / preferences / architecture. 각 entry에 confidence + dedup. prune/stats 명령 포함. |
| **/context-save** | 현재 컨텍스트 요약을 디스크에 저장. |
| **/context-restore** | 저장된 컨텍스트 재주입. |
| **/retro** | 회고 작성. Garry의 weekly metrics 포맷. |

#### 인프라 (3)
| Skill | 책무 |
|---|---|
| **GBrain** | Supabase + PGLite로 knowledge base. remote MCP로 접근. |
| **slop-scan** | npm/pypi 패키지 정합성 검사 (config: slop-scan.config.json). |
| **conductor** | Conductor 도구와 통합 (병렬 sprint 환경). |

### 4.3 lib/worktree.ts

핵심 라이브러리:
```typescript
// detached worktree 생성
await createDetachedWorktree(branch);

// SHA256 patch dedup
const patchHash = sha256(patch);
const dedupIndex = "~/.gstack-dev/harvests/dedup.json";
if (dedupIndex[patchHash]) skip();

// stale worktree pruning
await pruneStaleWorktrees({ olderThan: "7d" });

// exit handler cleanup
process.on("exit", () => cleanupOurWorktrees());
```

특징:
- **path allowlist**: 자기가 만든 worktree만 자동 cleanup (다른 도구의 worktree 건드리지 X)
- **patch harvest**: 여러 worktree에서 생성된 diff를 dedup하여 중복 작업 방지
- **detached worktree** (no branch initially) → 작업 완료 후 branch 명명

### 4.4 ETHOS preamble injection

```
모든 skill SKILL.md의 첫 줄:
<!-- ETHOS_INJECTED_HERE -->
```

빌드 시 `gen-skills.ts`가 이 마커를 ETHOS.md 내용으로 치환. 모든 skill이 같은 원칙을 preamble로 가짐.

---

## 5. 핵심 메커니즘 상세

### 5.1 Adaptive Specialist Gating

`/review`가 specialist를 dispatch할 때:

```
for each specialist:
  if specialist.history.last_10_runs.findings_count == 0:
    if specialist.tag != "[NEVER_GATE]":
      skip
```

이력은 영구 저장 (디스크). 시간이 지나면서 codebase 특성에 맞춰 specialist 풀이 좁아짐.

`[NEVER_GATE]` 태그를 가진 specialist:
- `security`
- `data-migration`

이유: low-frequency-but-catastrophic 문제는 0건 기간에도 무조건 검사.

### 5.2 Fix-First Review Pipeline (AUTO-FIX vs ASK)

`review/checklist.md`의 Pass-1/Pass-2 구조:

**Pass 1 (Mechanical fixes — AUTO-FIX)**:
- import 순서, unused import, prettier 위반
- typo, naming convention
- 명백한 dead code 제거

→ Sentinel이 직접 patch, 출력만 보고.

**Pass 2 (Judgment calls — ASK)**:
- 명명 모호성 (alternative 명 제시)
- 추상화 수준 (A vs B trade-off)
- 비즈니스 로직 의미 모호

→ Maestro(혹은 사용자)에게 escalate. 모든 ASK를 한 번에 batch.

### 5.3 Confidence-Scored Findings

모든 finding은 1-10 점수:
| 점수 | 처리 |
|---|---|
| 9-10 | 본문에 그대로 |
| 7-8 | 본문에 표시, but "verify" 권고 |
| 5-6 | caveat 처리 ("possible issue") |
| 3-4 | appendix만 |
| 1-2 | 출력 X (드롭) |

### 5.4 SKILL.md.tmpl → 다중 host 컴파일

```
gstack/skills/review/SKILL.md.tmpl
      ↓ gen-skills.ts + hosts/claude-code.ts
~/.claude/skills/review/SKILL.md

gstack/skills/review/SKILL.md.tmpl
      ↓ gen-skills.ts + hosts/opencode.ts
~/.config/opencode/skills/review/SKILL.md
```

각 host config의:
- `toolRewrites`: "Bash tool" → "run shell"
- `pathRewrites`: 디렉토리 경로 변환
- `frontmatter.allow/deny`: 지원되지 않는 YAML 필드 제거

### 5.5 Persistent Browser Daemon (`browse`)

`/browse` skill의 백엔드:
- **Bun + Playwright + SQLite cookie storage** 영구 데몬
- 시작: `gstack browse start` (port 9222 + auth port 9223)
- 클라이언트 접속: `gstack browse connect`
- 보안 레이어:
  - dual-listener port-separation (control vs data)
  - WebSocket auth token
  - BERT-small prompt-injection classifier (받은 명령을 분류)
  - canary tokens (응답에 캐너리 → 누수 감지)
- 출력: ARIA tree + `@e1, @e2, ...` reference로 element 지정
- 로그: ring-buffer (메모리 사용량 cap)

### 5.6 /learn Typed Memory

저장 카테고리:
| 카테고리 | 예시 |
|---|---|
| **patterns** | "이 codebase는 Result<T, E> 모나드 사용" |
| **pitfalls** | "useState in Server Component → silent failure" |
| **preferences** | "tailwind 클래스 정렬은 Headwind 순서로" |
| **architecture** | "domain 레이어는 infrastructure 모름" |

각 entry:
```yaml
category: pattern
confidence: 0.85
content: "..."
created: 2026-05-20
dedup_key: "tailwind-class-order"
```

`/learn prune`: dedup + confidence 낮은 것 제거
`/learn stats`: 카테고리별 entry 수

### 5.7 Plan-Completion Audit

`/ship` 직전, plan에 적힌 각 unit을 다음 5단계로 분류:
- **DONE** (구현 + 테스트 + 머지)
- **PARTIAL** (일부만 구현)
- **NOT-DONE** (착수 X)
- **CHANGED** (계획 변경됨, 사유 명시)
- **UNVERIFIABLE** (증거 부족)

UNVERIFIABLE은 무조건 fail. 사용자가 manual verify 후 진행.

---

## 6. 출력 포맷 디테일

### 6.1 Review Output

```
## Pass 1 (AUTO-FIXED)
- [src/foo.ts:12] unused import → removed
- [src/bar.ts:45] prettier → applied

## Pass 2 (ASK)
- [src/baz.ts:30] naming: `data` is generic
  Options: `userPayload`, `apiResponse`, `inputDoc`
  Recommend: userPayload (matches existing pattern)

## Specialist findings

### security (confidence 8)
- [src/auth.ts:80] potential JWT replay window

### maintainability (confidence 6, caveat)
- possible duplication with src/utils/format.ts

## Verdict
- Pass 1 fixes: 3 applied
- Pass 2 asks: 1 pending user
- Block: No
```

---

## 7. 주목할 디테일

- **SKILL.md generated** + **SKILL.md.tmpl source**: source-of-truth 분리. 사람이 generated를 직접 고치는 사고 방지.
- **bun 의존**: Node가 아닌 Bun runtime 가정. 설치 진입장벽.
- **mac-arm64 only binary** (browse daemon): cross-platform 부족.
- **~/.gstack-dev/**: 글로벌 dotfile에 상태 저장 (worktree dedup, learn memory 등). 다른 도구와 충돌 가능성.
- **CSO/codex skill의 외부 의존**: OWASP DB, OpenAI Codex API. 비용 + 네트워크.
- **Adaptive gating의 영속성**: hit-rate 추적이 디스크에 남아서 codebase가 바뀌면 outdated될 위험. 주기 prune 필요.
- **Conductor 통합**: gstack 단독 사용 시에는 conductor.json 무시 가능. 통합 시 별도 sprint 환경 필요.

---

## 8. 참조 URL

- README: https://github.com/garrytan/gstack
- 핵심 파일:
  - `ETHOS.md`
  - `ARCHITECTURE.md`
  - `lib/worktree.ts`
  - `hosts/opencode.ts` (~30 LOC, host adapter 예시)
  - `hosts/factory.ts` (Factory Droid 변환)
  - `skills/review/checklist.md` (Pass-1/Pass-2)
  - `skills/review/specialists/*.md` (7 specialist 정의)
  - `skills/learn/SKILL.md` (typed memory)
  - `slop-scan.config.json`
- 컴파일러: `tools/gen-skills.ts`
