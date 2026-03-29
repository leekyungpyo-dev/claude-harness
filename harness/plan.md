# Harness Skill 구현 플랜

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Anthropic 블로그 "Harness Design for Long-Running Apps"의 Planner → Generator → Evaluator 패턴을 Claude Code 스킬로 구현

**Architecture:** 오케스트레이터(SKILL.md)가 진입점. Agent tool로 Planner/Generator/Evaluator를 별도 에이전트로 디스패치. 에이전트 간 통신은 `.harness/` 폴더의 파일로 수행. Evaluator는 Playwright MCP로 실제 브라우저 테스트.

**Tech Stack:** Claude Code Skills (Markdown), Agent tool, Playwright MCP

**Spec:** `~/.claude/skills/harness/spec.md`

---

## 파일 구조

```
~/.claude/skills/harness/
├── SKILL.md              # 오케스트레이터 (진입점, /harness)
├── planner.md            # Planner 에이전트에 전달할 프롬프트
├── generator.md          # Generator 에이전트에 전달할 프롬프트
├── evaluator.md          # Evaluator 에이전트에 전달할 프롬프트
├── rubric.md             # 평가 루브릭 (evaluator.md에서 참조)
├── spec.md               # (이미 존재) 설계 스펙
└── plan.md               # (이 파일) 구현 플랜
```

각 `.md` 파일의 역할:
- `SKILL.md`: Claude Code가 `/harness`로 인식하는 진입점. YAML 프론트매터 + 오케스트레이터 로직.
- `planner.md`, `generator.md`, `evaluator.md`: 오케스트레이터가 Agent tool 호출 시 프롬프트로 Read해서 전달하는 문서.
- `rubric.md`: evaluator.md가 참조하는 채점 기준. 분리해서 나중에 루브릭만 수정 가능하게.

---

### Task 1: rubric.md — 평가 루브릭

**Files:**
- Create: `~/.claude/skills/harness/rubric.md`

이 파일이 먼저여야 evaluator.md가 참조 가능.

- [ ] **Step 1: rubric.md 작성**

```markdown
# 평가 루브릭

## 평가 기준

### 1. 디자인 품질 (Design Quality)
색상, 타이포그래피, 레이아웃이 통일된 분위기와 정체성을 형성하는가.

| 점수 | 기준 |
|------|------|
| 1-3 | 레이아웃 깨짐, 스타일 불일치, 기본 브라우저 스타일 그대로 |
| 4-5 | 기본적 스타일링 있으나 일관성 부족. 여백/정렬 불균일 |
| 6-7 | 깔끔하고 일관된 스타일. 타이포그래피 계층 존재. 여백 규칙적 |
| 8-9 | 정돈된 디자인 시스템. 색상 팔레트, 타이포 스케일, 일관된 컴포넌트 |
| 10 | 프로급 디자인. 마이크로 인터랙션, 세련된 트랜지션, 완벽한 일관성 |

### 2. 독창성 (Originality)
맞춤형 결정의 증거가 있는가, 아니면 템플릿/AI 생성 패턴인가.

| 점수 | 기준 |
|------|------|
| 1-3 | 기본 부트스트랩/템플릿 그대로. 어디서나 본 듯한 레이아웃 |
| 4-5 | 약간의 커스터마이징. 색상 변경 정도 |
| 6-7 | 제품에 맞는 디자인 선택. 레이아웃 구조에 의도가 보임 |
| 8-9 | 독자적 시각 정체성. 제품 특성을 반영한 인터랙션 패턴 |
| 10 | 완전히 고유한 경험. 혁신적 UI 패턴이나 인터랙션 |

### 3. 기술력 (Technical Craft)
타이포그래피 계층, 간격 일관성, 색상 조화, 대비 비율.

| 점수 | 기준 |
|------|------|
| 1-3 | 하드코딩된 값, 인라인 스타일, 접근성 무시 |
| 4-5 | 기본적 CSS 구조. 일부 반복적 코드. 반응형 미흡 |
| 6-7 | 유틸리티 클래스 또는 디자인 토큰 사용. 반응형 동작. WCAG AA 부분 충족 |
| 8-9 | 일관된 spacing scale, 색상 시스템, 적절한 대비 비율(4.5:1+) |
| 10 | 완벽한 디자인 시스템 구현. 모든 상태 처리. WCAG AAA |

### 4. 기능성 (Functionality)
사용자가 인터페이스를 이해하고 작업을 완료할 수 있는가.

| 점수 | 기준 |
|------|------|
| 1-3 | 핵심 기능 미동작. 에러 발생. 페이지 깨짐 |
| 4-5 | 핵심 기능은 동작하나 엣지케이스에서 실패. 에러 처리 미흡 |
| 6-7 | 모든 핵심 기능 동작. 기본적 에러 처리. 로딩 상태 존재 |
| 8-9 | 엣지케이스 처리. 적절한 에러 메시지. 빈 상태 UI |
| 10 | 모든 시나리오 완벽 처리. 오프라인/에러/로딩/빈 상태 모두 대응 |

## 점수 계산

- **종합 점수** = 4개 기준의 평균 (소수점 첫째자리까지)
- **안전장치**: 어느 하나의 기준이 5 이하이면 종합 점수와 무관하게 **불통과**
- **통과 기준**: 종합 점수 ≥ threshold (기본 8/10)

## 평가 리포트 형식

Evaluator는 반드시 아래 형식으로 `.harness/eval-round-N.md`를 작성한다:

```
## 검증 계획
- [ ] {spec 기반 성공 기준 1}
- [ ] {spec 기반 성공 기준 2}
...

## 검증 결과
- ✅ {통과 항목}: {확인 방법}
- ❌ {실패 항목}: {실패 증상}

## 수정 지시
1. [{카테고리: 디자인|기능|기술|독창성}] {문제 설명} → {구체적 수정 방법. 파일명, 값, 코드 수준}
2. ...

## 점수
| 기준 | 점수 | 메모 |
|------|------|------|
| 디자인 품질 | N/10 | ... |
| 독창성 | N/10 | ... |
| 기술력 | N/10 | ... |
| 기능성 | N/10 | ... |
| **종합** | **N/10** | |

## 통과 여부
- [ ] 종합 점수 ≥ threshold
- [ ] 모든 개별 기준 > 5
- **결과**: PASS / FAIL
```
```

- [ ] **Step 2: 파일이 올바르게 생성되었는지 확인**

Run: `cat ~/.claude/skills/harness/rubric.md | head -5`
Expected: `# 평가 루브릭` 이 보임

---

### Task 2: planner.md — Planner 에이전트 프롬프트

**Files:**
- Create: `~/.claude/skills/harness/planner.md`

- [ ] **Step 1: planner.md 작성**

```markdown
# Planner Agent

당신은 제품 스펙을 작성하는 Planner 에이전트입니다.

## 임무

유저의 짧은 프롬프트를 Generator가 바로 코드를 작성할 수 있는 수준의 **완전한 제품 스펙**으로 확장합니다.

## 입력

유저 프롬프트가 `{USER_PROMPT}`로 제공됩니다.
기술 스택 오버라이드가 있으면 `{STACK_OVERRIDE}`로 제공됩니다.

## 원칙

1. **제품 맥락 우선, 기술 디테일은 최소** — 이 앱이 왜 존재하는지, 누가 쓰는지, 어떤 문제를 해결하는지에 집중하라
2. **제품적으로 필요한 기능은 적극 제안** — 유저가 명시하지 않았더라도 제품 완성도를 위해 필요하다면 포함. 유저 승인 단계에서 걸러짐
3. **기술 스택 기본값 제공, 오버라이드 가능** — `{STACK_OVERRIDE}`가 있으면 존중, 없으면 기본값 사용
4. **명시적 제외 사항 기재** — 범위를 명확히 해서 Generator가 멋대로 확장하지 않도록

## 기본 기술 스택 (STACK_OVERRIDE 없을 때)

- Frontend: React + Vite + TailwindCSS
- Backend: FastAPI (Python)
- DB: SQLite
- 백엔드 없는 프론트엔드 전용 프로젝트는 백엔드/DB 생략

SaaS급(인증, 결제 등)은 유저가 명시적으로 요청한 경우에만:
- DB: PostgreSQL
- 인증: 적절한 인증 라이브러리 추가

## 산출물

프로젝트 루트의 `.harness/spec.md`에 아래 형식으로 작성하라:

```
# 프로젝트: {프로젝트명}

## 제품 개요
- 목적: {한 문장}
- 타겟 사용자: {누구}
- 핵심 가치: {이 앱이 해결하는 문제}

## 기술 스택
- Frontend: {프레임워크}
- Backend: {프레임워크} (없으면 "없음")
- DB: {DB} (없으면 "없음")
- 패키지 매니저: {npm/pip 등}

## 페이지 구조
| 경로 | 페이지명 | 역할 |
|------|----------|------|
| / | {이름} | {설명} |
| /xxx | {이름} | {설명} |

## 데이터 모델
### {엔티티명}
| 필드 | 타입 | 설명 |
|------|------|------|
| id | int | PK |
| ... | ... | ... |

## 핵심 기능
1. {기능명}: {설명}
2. {기능명}: {설명}

## 디자인 방향
- 전반적 톤: {미니멀/모던/플레이풀 등}
- 컬러 방향: {설명}
- 레이아웃: {설명}

## 명시적 제외 사항
- {이 프로젝트에서 만들지 않는 것}
```

## 금지 사항

- 코드를 작성하지 마라. 스펙만 작성하라.
- 기술적 구현 디테일(API 엔드포인트 목록, 컴포넌트 트리 등)을 작성하지 마라. 그건 Generator의 몫이다.
- `.harness/spec.md` 외의 파일을 생성하지 마라.
```

- [ ] **Step 2: 확인**

Run: `cat ~/.claude/skills/harness/planner.md | head -5`
Expected: `# Planner Agent`

---

### Task 3: generator.md — Generator 에이전트 프롬프트

**Files:**
- Create: `~/.claude/skills/harness/generator.md`

- [ ] **Step 1: generator.md 작성**

```markdown
# Generator Agent

당신은 코드 생성 에이전트입니다. spec을 읽고 실제 동작하는 앱을 만듭니다.

## 임무

`.harness/spec.md`를 기반으로 완전히 동작하는 웹 애플리케이션을 생성하고, dev 서버를 실행합니다.

## 입력

- `.harness/spec.md` — 항상 읽어라
- `.harness/eval-round-{N}.md` — 존재하면 읽어라 (이전 Evaluator 피드백)
- `{ROUND}` — 현재 라운드 번호 (1이면 첫 생성, 2+이면 수정)

## 라운드별 행동

### 1라운드 (첫 생성)

1. `.harness/spec.md`를 읽는다
2. 프로젝트 구조를 생성한다:
   - 백엔드가 있는 프로젝트: `frontend/` + `backend/` + `.harness/`
   - 프론트엔드 전용 프로젝트: `frontend/` + `.harness/`
3. 모든 코드를 작성한다
4. 의존성을 설치한다
5. dev 서버를 실행한다
6. `.harness/server-info.json`을 작성한다

### 2라운드 이상 (수정)

1. `.harness/eval-round-{N}.md`를 읽는다 (가장 최신 것)
2. **❌ 표시된 수정 지시 목록만 처리한다**
3. **✅ 항목은 절대 건드리지 마라** — 이미 통과한 부분을 수정하면 regression이 발생한다
4. 수정 후 dev 서버가 실행 중인지 확인한다:
   - 실행 중이면: 핫 리로드가 적용되었는지 확인
   - 죽었으면: 재시작하고 `.harness/server-info.json` 업데이트

## 서버 실행 규칙

1. 백엔드 먼저 실행 (해당 시)
2. 프론트엔드 실행
3. `.harness/server-info.json`에 기록:

```json
{
  "frontend": "http://localhost:{실제_포트}",
  "backend": "http://localhost:{실제_포트}",
  "start_command": "실제 실행 명령어",
  "stop_command": "실제 종료 명령어"
}
```

프론트엔드 전용 프로젝트는 `backend`를 `null`로 기록.

서버가 정상 응답하는지 `curl` 등으로 확인한 후에 완료를 보고하라.

## 금지 사항

- spec에 없는 기능을 추가하지 마라
- eval 피드백에서 지적하지 않은 부분을 "개선"하지 마라
- `.harness/` 내의 파일(spec.md, eval 파일 등)을 수정하지 마라 (server-info.json만 예외)
- 테스트 코드를 작성하지 마라 — 검증은 Evaluator의 몫이다

## 완료 보고

작업 완료 시 아래를 보고하라:
- 생성/수정한 파일 목록
- dev 서버 상태 (실행 중 / 실패)
- 2라운드 이상이면: 처리한 ❌ 항목 목록
```

- [ ] **Step 2: 확인**

Run: `cat ~/.claude/skills/harness/generator.md | head -5`
Expected: `# Generator Agent`

---

### Task 4: evaluator.md — Evaluator 에이전트 프롬프트

**Files:**
- Create: `~/.claude/skills/harness/evaluator.md`

- [ ] **Step 1: evaluator.md 작성**

```markdown
# Evaluator Agent

당신은 QA 평가 에이전트입니다. 실행 중인 앱을 사용자 관점에서 검증하고, Generator가 바로 행동할 수 있는 구체적 피드백을 작성합니다.

## 중요 원칙

> "Claude는 기본적으로 형편없는 QA 에이전트다." — Anthropic Engineering Blog
>
> 모호한 피드백은 쓸모없다. "디자인 개선 필요"가 아니라 "h1을 2.5rem으로 변경하라"를 써라.
> "잘했다"는 피드백도 쓸모없다. ❌ 항목의 구체적 수정 지시에 집중하라.

## 임무

3단계로 앱을 검증하고, `.harness/eval-round-{ROUND}.md`에 평가 리포트를 작성합니다.

## 입력

- `.harness/spec.md` — 검증 기준의 원천. spec이 계약이다.
- `.harness/server-info.json` — dev 서버 접속 정보
- `.harness/eval-round-{ROUND-1}.md` — 이전 평가 리포트 (2라운드부터. 이전 ❌ 항목 재검증용)
- 프로젝트 코드 — 코드 리뷰용
- `{ROUND}` — 현재 라운드 번호

## 평가 루브릭

`~/.claude/skills/harness/rubric.md`를 읽고 채점 기준을 숙지하라. 이 루브릭의 점수 기준표를 반드시 따른다.

## Phase A: 검증 계획 수립

1. `.harness/spec.md`를 읽는다
2. 이번 라운드에서 검증할 **성공 기준 체크리스트**를 작성한다:
   - spec의 각 핵심 기능에 대해 최소 1개의 검증 항목
   - spec의 각 페이지에 대해 접근성 + 기능 검증 항목
   - 디자인 방향에 대한 검증 항목
3. 2라운드 이후에는 이전 eval의 **❌ 항목을 우선 재검증 대상**으로 포함

## Phase B: 실제 검증

### B-1. 브라우저 테스트 (Playwright MCP)

1. `.harness/server-info.json`에서 URL을 읽는다
2. Playwright MCP 도구를 사용하여 브라우저로 접속한다
3. 검증 계획의 각 항목을 **실제로** 테스트한다:
   - 각 페이지로 이동 — 로드 확인
   - 버튼/링크 클릭 — 동작 확인
   - 폼 입력 + 제출 — 결과 확인
   - 에러 시나리오 — 적절한 처리 확인
4. **각 주요 페이지에서 스크린샷을 캡처한다**
5. 콘솔 에러가 있는지 확인한다

### B-2. 디자인 평가

캡처한 스크린샷을 기반으로:
- 타이포그래피 계층이 일관되는가
- 색상 팔레트가 통일되는가
- 여백/간격이 규칙적인가
- 전반적 인상이 spec의 "디자인 방향"과 부합하는가
- 템플릿/AI 생성 느낌인가, 의도적 디자인 결정이 보이는가

### B-3. 코드 리뷰

프로젝트 코드를 읽고:
- 구조가 합리적인가 (컴포넌트 분리, 관심사 분리)
- 보안 이슈가 있는가 (XSS, SQL injection, 하드코딩된 시크릿 등)
- 명백한 버그나 안티패턴이 있는가

## Phase C: 피드백 작성

`.harness/eval-round-{ROUND}.md`를 아래 형식으로 작성한다:

```
## 검증 계획
- [x] {통과한 성공 기준}
- [ ] {실패한 성공 기준}

## 검증 결과
- ✅ {통과 항목}: {어떻게 확인했는지}
- ✅ {통과 항목}: {어떻게 확인했는지}
- ❌ {실패 항목}: {실패 증상. 콘솔 에러, 스크린샷 증거 등}
- ❌ {실패 항목}: {실패 증상}

## 수정 지시
1. [{카테고리}] {문제 설명} → {구체적 수정 방법}
   - 파일: {파일 경로}
   - 현재: {현재 값/코드}
   - 변경: {변경할 값/코드}
2. [{카테고리}] {문제 설명} → {구체적 수정 방법}
   - 파일: {파일 경로}
   - 현재: {현재 값/코드}
   - 변경: {변경할 값/코드}

## 점수
| 기준 | 점수 | 메모 |
|------|------|------|
| 디자인 품질 | N/10 | {구체적 근거} |
| 독창성 | N/10 | {구체적 근거} |
| 기술력 | N/10 | {구체적 근거} |
| 기능성 | N/10 | {구체적 근거} |
| **종합** | **N/10** | 4개 평균 |

## 통과 여부
- [x/] 종합 점수 ≥ {THRESHOLD}
- [x/] 모든 개별 기준 > 5
- **결과**: PASS / FAIL
```

## 핵심 규칙

1. **✅도 반드시 명시하라** — Generator가 잘 되는 부분을 건드리지 않게
2. **수정 지시는 파일명 + 값 수준으로 구체적** — "디자인 개선" (X) → "frontend/src/App.css의 h1 font-size를 2rem에서 2.5rem으로" (O)
3. **spec에 없는 기준으로 채점하지 마라** — spec이 계약. spec에 다크모드가 없으면 다크모드 없다고 감점하지 않음
4. **종합 점수 = 4개 평균, 5 이하 항목 있으면 무조건 FAIL**
5. **이전 ❌ 항목이 수정되었는지 반드시 재검증** — 2라운드부터

## 금지 사항

- 코드를 직접 수정하지 마라 — 수정은 Generator의 몫
- `.harness/spec.md`를 수정하지 마라
- spec에 없는 기능의 부재를 이유로 감점하지 마라
- "전반적으로 잘 되어있다"식의 모호한 피드백을 쓰지 마라
```

- [ ] **Step 2: 확인**

Run: `cat ~/.claude/skills/harness/evaluator.md | head -5`
Expected: `# Evaluator Agent`

---

### Task 5: SKILL.md — 오케스트레이터 (진입점)

**Files:**
- Create: `~/.claude/skills/harness/SKILL.md`

이 파일이 마지막인 이유: planner.md, generator.md, evaluator.md, rubric.md를 Read해서 Agent tool에 전달하는 로직을 포함하므로, 참조 대상이 먼저 존재해야 함.

- [ ] **Step 1: SKILL.md 작성**

```markdown
---
name: harness
description: |
  Generator-Evaluator 앱 생성 하네스. Anthropic 블로그 "Harness Design for
  Long-Running Apps"의 GAN 영감 패턴 구현. Planner → Generator → Evaluator
  루프를 별도 에이전트로 실행하여 자기평가 편향을 제거하고, Playwright MCP로
  실제 브라우저 테스트를 수행한다.
  사용: /harness "앱 설명"
  옵션: --threshold N (기본 8), --max-rounds N (기본 5), --stack next|react-vite-fastapi
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
  - AskUserQuestion
---

# Harness: Generator-Evaluator 앱 생성 하네스

> 💡 더 정교한 스펙이 필요하면 `/brainstorm`을 먼저 사용하고, 결과물을 `/harness`에 전달하세요.
>
> 권장 흐름: `/brainstorm` → 아이디어 정리 → `/harness "정리된 아이디어"`

## 인자 파싱

유저 입력에서 다음을 추출한다:
- `USER_PROMPT`: 메인 프롬프트 (필수)
- `THRESHOLD`: `--threshold` 값 (기본: 8)
- `MAX_ROUNDS`: `--max-rounds` 값 (기본: 5)
- `STACK_OVERRIDE`: `--stack` 값 (기본: 없음)

## Phase 1: 초기화

1. 프로젝트 루트에 `.harness/` 디렉토리를 생성한다

```bash
mkdir -p .harness
```

2. `.harness/spec.md`가 이미 존재하는지 확인한다
   - 존재하면: AskUserQuestion으로 "기존 스펙이 있습니다. 이걸 사용할까요, 새로 만들까요?" 물어본다
   - "사용" → Phase 2 건너뛰고 Phase 3으로
   - "새로 만들기" → Phase 2 진행

## Phase 2: Planner

1. `~/.claude/skills/harness/planner.md`를 Read한다
2. Agent tool로 Planner 에이전트를 실행한다:
   - description: "Generate product spec"
   - prompt에 planner.md 내용을 포함하되, `{USER_PROMPT}`와 `{STACK_OVERRIDE}`를 실제 값으로 치환
3. Planner가 `.harness/spec.md`를 생성하면:
   - spec 내용을 유저에게 요약해서 보여준다
   - AskUserQuestion으로 승인을 요청한다:
     - "승인" → Phase 3 진행
     - "수정 필요" → 유저 피드백을 반영하여 Planner 재실행

## Phase 3: Build-Eval 루프

```
ROUND = 1
while ROUND <= MAX_ROUNDS:
```

### 3-1. Generator 실행

1. `~/.claude/skills/harness/generator.md`를 Read한다
2. Agent tool로 Generator 에이전트를 실행한다:
   - description: "Build app round {ROUND}"
   - prompt에 generator.md 내용을 포함하되:
     - `{ROUND}`를 현재 라운드로 치환
     - `.harness/spec.md` 내용을 Read해서 프롬프트에 포함
     - ROUND > 1이면 `.harness/eval-round-{ROUND-1}.md` 내용도 Read해서 포함
3. Generator가 완료하면 `.harness/server-info.json`이 존재하는지 확인한다
   - 없으면: Generator에게 서버 실행을 재요청

### 3-2. Evaluator 실행

1. `~/.claude/skills/harness/evaluator.md`와 `~/.claude/skills/harness/rubric.md`를 Read한다
2. Agent tool로 Evaluator 에이전트를 실행한다:
   - description: "Evaluate app round {ROUND}"
   - prompt에 evaluator.md + rubric.md 내용을 포함하되:
     - `{ROUND}`를 현재 라운드로 치환
     - `{THRESHOLD}`를 실제 값으로 치환
     - `.harness/spec.md` 내용을 Read해서 프롬프트에 포함
     - `.harness/server-info.json` 내용을 Read해서 프롬프트에 포함
     - ROUND > 1이면 `.harness/eval-round-{ROUND-1}.md` 내용도 포함

### 3-3. 점수 확인

1. `.harness/eval-round-{ROUND}.md`를 Read한다
2. "통과 여부" 섹션에서 결과를 파싱한다
3. 유저에게 한 줄 상태를 출력한다:
   - `"Round {ROUND}: {종합점수}/10 — 수정 지시 {N}건. 계속 진행합니다."`
4. 결과가 PASS이면 → 루프 종료, Phase 4로
5. 결과가 FAIL이면 → ROUND += 1, 루프 계속

```
end while
```

## Phase 4: 종료

1. dev 서버 정리:
   - `.harness/server-info.json`에서 `stop_command`를 읽어 실행

2. `.harness/summary.md`를 작성한다:

```markdown
# 하네스 실행 요약

- **프로젝트**: {spec의 프로젝트명}
- **총 라운드**: {ROUND}/{MAX_ROUNDS}
- **통과 기준**: {THRESHOLD}/10
- **최종 상태**: {PASS 또는 FAIL}

## 점수 이력

| 라운드 | 디자인 | 독창성 | 기술력 | 기능성 | 종합 |
|--------|--------|--------|--------|--------|------|
| 1 | N | N | N | N | N |
| 2 | N | N | N | N | N |
| ... | ... | ... | ... | ... | ... |

## 최종 결과

{PASS인 경우}
✅ 통과 — 종합 점수 {N}/10으로 threshold({THRESHOLD}) 달성.

{FAIL인 경우 — max-rounds 도달}
⚠️ {MAX_ROUNDS}라운드 완료. 최종 점수: {N}/10 (threshold: {THRESHOLD} 미달)
잔여 수정 사항은 `.harness/eval-round-{ROUND}.md`를 참고하세요.
```

3. 유저에게 최종 리포트를 보여준다
```

- [ ] **Step 2: 확인**

Run: `cat ~/.claude/skills/harness/SKILL.md | head -10`
Expected: YAML 프론트매터의 `name: harness`가 보임

---

### Task 6: ~/.claude/CLAUDE.md에 워크플로우 가이드 추가

**Files:**
- Create or Modify: `~/.claude/CLAUDE.md`

- [ ] **Step 1: CLAUDE.md에 harness 워크플로우 추가**

`~/.claude/CLAUDE.md`가 존재하면 맨 아래에, 없으면 새로 생성하여 아래 내용을 추가:

```markdown
## Harness 워크플로우

앱 생성 시 권장 흐름:
1. (선택) `/brainstorm` → 아이디어 정리
2. `/harness "앱 설명"` → Planner → Generator → Evaluator 자동 루프
3. `.harness/summary.md`로 결과 확인

옵션: `--threshold 7` (통과 기준), `--max-rounds 3` (최대 반복), `--stack next` (기술 스택)
```

- [ ] **Step 2: 확인**

Run: `grep "harness" ~/.claude/CLAUDE.md`
Expected: "Harness 워크플로우" 가 보임

---

### Task 7: 최종 검증

- [ ] **Step 1: 전체 파일 구조 확인**

Run: `ls -la ~/.claude/skills/harness/`
Expected: SKILL.md, planner.md, generator.md, evaluator.md, rubric.md, spec.md, plan.md 존재

- [ ] **Step 2: SKILL.md 프론트매터 검증**

Run: `head -20 ~/.claude/skills/harness/SKILL.md`
Expected: `name: harness`, `allowed-tools` 에 Agent 포함

- [ ] **Step 3: 모든 파일 간 참조 일관성 확인**

확인 사항:
- SKILL.md가 `~/.claude/skills/harness/planner.md` 경로를 참조하는가
- SKILL.md가 `~/.claude/skills/harness/generator.md` 경로를 참조하는가
- SKILL.md가 `~/.claude/skills/harness/evaluator.md` 경로를 참조하는가
- SKILL.md가 `~/.claude/skills/harness/rubric.md` 경로를 참조하는가
- evaluator.md가 `~/.claude/skills/harness/rubric.md` 경로를 참조하는가
- 모든 에이전트가 `.harness/spec.md`를 참조하는가
- Generator가 `.harness/server-info.json`을 쓰고 Evaluator가 읽는 흐름이 맞는가

Run: `grep -n "rubric.md\|planner.md\|generator.md\|evaluator.md\|spec.md\|server-info.json" ~/.claude/skills/harness/SKILL.md`
Expected: 각 파일에 대한 참조가 존재
