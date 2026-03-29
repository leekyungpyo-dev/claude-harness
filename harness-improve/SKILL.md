---
name: harness-improve
description: |
  기존 앱 품질 개선 하네스. 이미 존재하는 프로젝트의 디자인/코드 품질을
  Evaluator → Generator 반복 루프로 개선한다. Baseline 측정 후 자동 개선.
  /harness의 evaluator, generator, rubric을 재사용.
  사용: /harness-improve "디자인 다듬어줘"
  프롬프트 없이도 가능: /harness-improve (baseline 기반 자동 개선)
  옵션: --threshold N (기본 8), --max-rounds N (기본 5)
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

# Harness-Improve: 기존 앱 품질 개선 하네스

> 기존 하네스(`/harness`)의 Evaluator/Generator/Rubric을 재사용하여
> 이미 존재하는 앱의 디자인/코드 품질을 반복적으로 개선합니다.

## 인자 파싱

유저 입력에서 다음을 추출한다:
- `USER_PROMPT`: 개선 요청 (선택. 없으면 baseline 기반 자동 판단)
- `THRESHOLD`: `--threshold` 값 (기본: 8)
- `MAX_ROUNDS`: `--max-rounds` 값 (기본: 5)

## Phase 1: 초기화

1. 프로젝트 루트에 `.harness/` 디렉토리를 생성한다

```bash
mkdir -p .harness
```

2. dev 서버 자동 감지 + 실행:
   - `package.json`의 `scripts.dev` 또는 `scripts.start` 확인
   - `Makefile`의 `dev` 또는 `run` 타겟 확인
   - `docker-compose.yml` 확인
   - `manage.py` (Django), `main.py` + uvicorn (FastAPI) 등 프레임워크별 패턴
   - **모두 실패 시**: AskUserQuestion으로 "dev 서버 실행 명령어를 알려주세요" 질문
3. 서버 실행 후 `.harness/server-info.json` 기록

## Phase 2: Analyzer

1. `~/.claude/skills/harness-improve/analyzer.md`를 Read한다
2. Agent tool로 Analyzer 에이전트를 실행한다:
   - description: "Analyze existing codebase"
   - prompt 구성:
     - analyzer.md의 전체 내용
     - `{USER_PROMPT}`를 유저가 입력한 실제 프롬프트로 치환 (없으면 "없음"으로)
3. Analyzer가 `.harness/spec.md`를 생성하면:
   - spec 내용을 유저에게 요약해서 보여준다
   - AskUserQuestion으로 승인을 요청한다:
     - "승인" → Phase 3 진행
     - "수정 필요" → 유저 피드백을 반영하여 Analyzer 재실행

## Phase 3: Baseline 측정

1. `~/.claude/skills/harness/evaluator.md`를 Read한다
2. `~/.claude/skills/harness/rubric.md`를 Read한다
3. `.harness/spec.md`를 Read한다
4. `.harness/server-info.json`을 Read한다
5. Agent tool로 Evaluator 에이전트를 실행한다:
   - description: "Measure baseline"
   - prompt 구성:
     - evaluator.md의 전체 내용
     - rubric.md의 전체 내용을 "## 루브릭" 섹션으로 포함
     - `{ROUND}`를 `0`으로 치환
     - `{THRESHOLD}`를 실제 값으로 치환
     - spec.md의 전체 내용을 "## 스펙 내용" 섹션으로 포함
     - server-info.json의 내용을 "## 서버 정보" 섹션으로 포함
6. Evaluator가 `.harness/eval-round-0.md`를 생성하면:
   - 유저에게 baseline 보여줌: "현재 점수: 디자인 N, 독창성 N, 기술력 N, 기능성 N → 종합 N/10"

### Phase 3.5: Baseline 체크

- baseline 종합 점수 >= THRESHOLD이면:
  - AskUserQuestion: "현재 점수가 이미 {N}/10으로 threshold({THRESHOLD})를 충족합니다."
    - A) threshold를 올려서 더 개선 (예: 9/10)
    - B) 그대로 종료
  - B 선택 시 → Phase 5 종료로
  - A 선택 시 → THRESHOLD 업데이트 후 계속

- USER_PROMPT가 없었으면:
  - eval-round-0.md에서 낮은 기준을 자동 개선 대상으로 설정
  - `.harness/spec.md`의 "## 개선 목표" 섹션을 업데이트

## Phase 4: Improve 루프

```
ROUND = 1
while ROUND <= MAX_ROUNDS:
```

### 4-1. Generator 실행

1. `~/.claude/skills/harness/generator.md`를 Read한다
2. `.harness/spec.md`를 Read한다
3. `.harness/eval-round-{ROUND-1}.md`를 Read한다 (ROUND 1이면 eval-round-0.md)
4. Agent tool로 Generator 에이전트를 실행한다:
   - description: "Improve app round {ROUND}"
   - prompt 구성:
     - generator.md의 전체 내용
     - **추가 지시 삽입:**
       ```
       ## 추가 지시 (기존 앱 개선 모드)
       - 이것은 기존 앱 개선이다. 새 파일/폴더 구조를 생성하지 마라.
       - 1라운드부터 수정 모드다. eval의 ❌ 항목만 처리하라.
       - 기존 기능이 깨지지 않도록 최소한의 변경만 하라.
       - frontend/, backend/ 디렉토리를 새로 만들지 마라. 기존 구조를 그대로 사용하라.
       - 기존에 없던 의존성을 추가하지 마라 (스타일링 라이브러리 변경 등 금지).
       ```
     - `{ROUND}`를 현재 라운드 번호로 치환
     - spec.md의 전체 내용을 "## 스펙 내용" 섹션으로 포함
     - eval 파일의 전체 내용을 "## 이전 평가 피드백" 섹션으로 포함

### 4-2. Evaluator 실행

1. `~/.claude/skills/harness/evaluator.md`를 Read한다
2. `~/.claude/skills/harness/rubric.md`를 Read한다
3. `.harness/spec.md`를 Read한다
4. `.harness/server-info.json`을 Read한다
5. `.harness/eval-round-{ROUND-1}.md`를 Read한다
6. Agent tool로 Evaluator 에이전트를 실행한다:
   - description: "Evaluate improvement round {ROUND}"
   - prompt 구성:
     - evaluator.md의 전체 내용
     - rubric.md의 전체 내용을 "## 루브릭" 섹션으로 포함
     - `{ROUND}`를 현재 라운드 번호로 치환
     - `{THRESHOLD}`를 실제 값으로 치환
     - spec.md의 전체 내용을 "## 스펙 내용" 섹션으로 포함
     - server-info.json의 내용을 "## 서버 정보" 섹션으로 포함
     - 이전 eval 파일의 전체 내용을 "## 이전 평가" 섹션으로 포함

### 4-3. 점수 확인

1. `.harness/eval-round-{ROUND}.md`를 Read한다
2. "통과 여부" 섹션에서 결과를 파싱한다
3. eval-round-0.md에서 baseline 점수를 참조하여 유저에게 한 줄 상태를 출력한다:
   - `"Round {ROUND}: {종합점수}/10 (baseline {BASELINE} 대비 +{DIFF}) — 수정 지시 {N}건"`
4. 결과가 PASS이면 → 루프 종료, Phase 5로
5. 결과가 FAIL이면 → ROUND += 1, 루프 계속

```
end while
```

## Phase 5: 종료

1. dev 서버 정리:
   - `.harness/server-info.json`에서 `stop_command`를 읽어 실행한다

2. `.harness/summary.md`를 작성한다:

```markdown
# 하네스 개선 요약

- **프로젝트**: {spec의 프로젝트명}
- **총 라운드**: {ROUND}/{MAX_ROUNDS}
- **통과 기준**: {THRESHOLD}/10
- **최종 상태**: {PASS 또는 FAIL}

## Baseline → 최종 비교

| 기준 | Baseline | 최종 | 변화 |
|------|----------|------|------|
| 디자인 품질 | N | N | +N |
| 독창성 | N | N | +N |
| 기술력 | N | N | +N |
| 기능성 | N | N | +N |
| **종합** | **N** | **N** | **+N** |

## 점수 이력

| 라운드 | 디자인 | 독창성 | 기술력 | 기능성 | 종합 |
|--------|--------|--------|--------|--------|------|
| 0 (baseline) | N | N | N | N | N |
| 1 | N | N | N | N | N |
| ... | ... | ... | ... | ... | ... |
```

3. 유저에게 최종 리포트를 보여준다:
   - PASS: "통과 — baseline {BASELINE}/10 → 최종 {FINAL}/10 (+{DIFF}). Threshold {THRESHOLD} 달성."
   - FAIL: "{MAX_ROUNDS}라운드 완료. baseline {BASELINE}/10 → 최종 {FINAL}/10 (+{DIFF}). Threshold {THRESHOLD} 미달. 잔여 수정 사항은 `.harness/eval-round-{ROUND}.md`를 참고하세요."
