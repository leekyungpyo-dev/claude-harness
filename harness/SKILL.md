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

> 더 정교한 스펙이 필요하면 brainstorming 스킬을 먼저 사용하고, 결과물을 `/harness`에 전달하세요.
>
> 권장 흐름: brainstorming → 아이디어 정리 → `/harness "정리된 아이디어"`

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
   - prompt 구성:
     - planner.md의 전체 내용
     - `{USER_PROMPT}`를 유저가 입력한 실제 프롬프트로 치환
     - `{STACK_OVERRIDE}`를 실제 값으로 치환 (없으면 "없음"으로)
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
2. `.harness/spec.md`를 Read한다
3. ROUND > 1이면 `.harness/eval-round-{ROUND-1}.md`를 Read한다
4. Agent tool로 Generator 에이전트를 실행한다:
   - description: "Build app round {ROUND}"
   - prompt 구성:
     - generator.md의 전체 내용
     - `{ROUND}`를 현재 라운드 번호로 치환
     - spec.md의 전체 내용을 "## 스펙 내용" 섹션으로 포함
     - (ROUND > 1이면) eval 파일의 전체 내용을 "## 이전 평가 피드백" 섹션으로 포함
5. Generator가 완료하면 `.harness/server-info.json`이 존재하는지 확인한다
   - 없으면: Generator에게 서버 실행을 재요청

### 3-2. Evaluator 실행

1. `~/.claude/skills/harness/evaluator.md`를 Read한다
2. `~/.claude/skills/harness/rubric.md`를 Read한다
3. `.harness/spec.md`를 Read한다
4. `.harness/server-info.json`을 Read한다
5. ROUND > 1이면 `.harness/eval-round-{ROUND-1}.md`를 Read한다
6. Agent tool로 Evaluator 에이전트를 실행한다:
   - description: "Evaluate app round {ROUND}"
   - prompt 구성:
     - evaluator.md의 전체 내용
     - rubric.md의 전체 내용을 "## 루브릭" 섹션으로 포함
     - `{ROUND}`를 현재 라운드 번호로 치환
     - `{THRESHOLD}`를 실제 값으로 치환
     - spec.md의 전체 내용을 "## 스펙 내용" 섹션으로 포함
     - server-info.json의 내용을 "## 서버 정보" 섹션으로 포함
     - (ROUND > 1이면) 이전 eval 파일의 전체 내용을 "## 이전 평가" 섹션으로 포함

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
   - `.harness/server-info.json`에서 `stop_command`를 읽어 실행한다

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
| ... | ... | ... | ... | ... | ... |

## 최종 결과

{결과 메시지}
```

3. 유저에게 최종 리포트를 보여준다:
   - PASS: "통과 — 종합 점수 {N}/10으로 threshold({THRESHOLD}) 달성."
   - FAIL (max-rounds 도달): "{MAX_ROUNDS}라운드 완료. 최종 점수: {N}/10 (threshold: {THRESHOLD} 미달). 잔여 수정 사항은 `.harness/eval-round-{ROUND}.md`를 참고하세요."
