# Harness-Improve: 기존 앱 품질 개선 하네스

> 기존 하네스(`/harness`)의 Evaluator/Generator/Rubric을 재사용하여
> 이미 존재하는 앱의 디자인/코드 품질을 반복적으로 개선하는 스킬.

## 핵심 사상

1. **Evaluator 먼저** — 신규 하네스는 Generator 먼저지만, 여기서는 baseline 측정이 선행
2. **기존 구조 존중** — 프로젝트 구조를 변경하지 않음. 있는 그대로의 코드를 수정
3. **기존 하네스 자산 재사용** — evaluator.md, generator.md, rubric.md를 Read 참조
4. **dev 서버 자동 감지** — package.json, Makefile 등을 분석해서 서버 실행. 실패 시 유저에게 질문

## 아키텍처

### 파일 구조

```
~/.claude/skills/harness-improve/
├── SKILL.md              # 오케스트레이터 (진입점, /harness-improve)
├── analyzer.md           # 기존 코드 분석 → 스펙 생성 (Planner 대체)
├── spec.md               # (이 파일) 설계 스펙
```

### 재사용 파일 (기존 하네스에서 Read 참조)

```
~/.claude/skills/harness/
├── generator.md          # Generator 에이전트 프롬프트
├── evaluator.md          # Evaluator 에이전트 프롬프트
├── rubric.md             # 평가 루브릭
```

### 프로젝트 내 생성 파일

```
<project-root>/
├── (기존 코드 그대로)
├── .harness/
│   ├── spec.md           # Analyzer 산출물 (현재 상태 + 개선 목표)
│   ├── server-info.json  # dev 서버 접속 정보
│   ├── eval-round-0.md   # baseline 평가 (개선 전 상태)
│   ├── eval-round-1.md   # 1회차 개선 후 평가
│   ├── eval-round-N.md   # N회차 개선 후 평가
│   └── summary.md        # 최종 요약 (baseline → 최종 비교)
```

## 실행 흐름

```
유저: /harness-improve "디자인 다듬어줘"
유저: /harness-improve  (프롬프트 없이도 가능)
유저: /harness-improve --threshold 9 --max-rounds 3
  │
  ▼
[오케스트레이터]
  │
  ├─1. 초기화
  │    - .harness/ 디렉토리 생성
  │    - 인자 파싱 (threshold 기본 8, max-rounds 기본 5)
  │    - dev 서버 자동 감지 + 실행 → server-info.json 기록
  │    - 감지 실패 시 → AskUserQuestion으로 실행 명령어 질문
  │
  ├─2. Analyzer 에이전트
  │    - 기존 코드 구조 분석 (파일 트리, 기술 스택, 라우트 구조)
  │    - .harness/spec.md 생성
  │    - 유저 프롬프트가 있으면 "## 개선 목표" 섹션 포함
  │    - 유저에게 spec 요약 + 승인 요청
  │
  ├─3. Baseline 측정 (Evaluator 에이전트)
  │    - 현재 앱을 Playwright MCP로 테스트
  │    - .harness/eval-round-0.md 생성 (baseline 점수)
  │    - 유저에게 baseline 보여줌: "현재 점수: 디자인 6, 독창성 5, 기술력 7, 기능성 8 → 종합 6.5/10"
  │    - 프롬프트 없었으면: 낮은 기준을 자동 개선 대상으로 설정하고 spec의 "## 개선 목표"에 추가
  │
  ├─3.5. Baseline 체크
  │    - baseline 종합 점수 >= threshold이면:
  │      AskUserQuestion: "현재 점수가 이미 {N}/10으로 threshold({THRESHOLD})를 충족합니다.
  │      A) threshold를 올려서 더 개선 (예: 9/10)
  │      B) 그대로 종료"
  │    - B 선택 시 → Phase 5 종료로
  │
  ├─4. Improve 루프 (max N회, threshold M/10)
  │    │
  │    │  [Generator 에이전트]
  │    │   ← reads: spec.md + eval-round-N.md
  │    │   → 기존 코드 수정 (기존 구조 존중, ❌ 항목만 처리)
  │    │   → 추가 지시 (오케스트레이터가 프롬프트에 삽입):
  │    │     "이것은 기존 앱 개선이다. 새 파일/폴더 구조를 생성하지 마라.
  │    │      eval의 ❌ 항목만 처리하라.
  │    │      기존 기능이 깨지지 않도록 최소한의 변경만 하라."
  │    │   → dev 서버 핫 리로드 확인 / 재시작
  │    │
  │    │  [Evaluator 에이전트]
  │    │   Phase A: 검증 계획 (spec + 이전 eval 기반)
  │    │   Phase B: Playwright 테스트 + 디자인 평가 + 코드 리뷰
  │    │   Phase C: eval-round-N+1.md 작성
  │    │   → 종합 점수 >= threshold? → BREAK
  │    │
  │    │  오케스트레이터: "Round N: 7.5/10 (baseline 6.5 대비 +1.0)"
  │    │
  │
  └─5. 종료
       - dev 서버 정리
       - .harness/summary.md 생성
       - baseline 대비 개선률 포함
```

## Analyzer 상세

### 역할

기존 코드를 분석하여 현재 상태를 스펙으로 문서화. Planner가 "무에서 유"라면 Analyzer는 "유에서 유".

### 분석 대상

1. **프로젝트 루트** — 파일/폴더 구조 탐색
2. **기술 스택 감지**:
   - `package.json` → Node.js 프로젝트, 프레임워크 판별
   - `requirements.txt` / `pyproject.toml` → Python 프로젝트
   - `Cargo.toml`, `go.mod` 등 → 기타 언어
3. **라우트/페이지 구조** — 프레임워크별 라우팅 파일 분석
4. **주요 컴포넌트/모듈** — 코드를 읽어서 핵심 기능 파악

### 산출물: `.harness/spec.md`

```markdown
# 프로젝트: {프로젝트명}

## 현재 상태
- 기술 스택: {감지된 스택}
- 프로젝트 구조: {간략 요약}
- 주요 기능: {현재 동작하는 기능 목록}

## 페이지 구조
| 경로 | 역할 |
|------|------|
| / | {설명} |
| /xxx | {설명} |

## 디자인 현황
- 현재 스타일링 방식: {CSS/Tailwind/styled-components 등}
- 전반적 톤: {관찰된 디자인 방향}

## 개선 목표
{유저 프롬프트 기반 또는 baseline 점수 기반}
- {개선 목표 1}
- {개선 목표 2}

## 제약 사항
- 기존 프로젝트 구조를 변경하지 않는다
- 기존 기능을 깨뜨리지 않는다
```

### Analyzer 원칙

1. **현재 상태를 있는 그대로 기술** — 판단하지 말고 사실만 기록
2. **기존 구조를 변경 제안하지 않는다** — 구조 개선은 범위 밖
3. **코드를 수정하지 않는다** — 분석만

## Dev 서버 자동 감지

### 감지 순서

1. `package.json`의 `scripts.dev` 또는 `scripts.start` 확인
2. `Makefile`의 `dev` 또는 `run` 타겟 확인
3. `docker-compose.yml` 확인
4. `manage.py` (Django), `main.py` + uvicorn (FastAPI) 등 프레임워크별 패턴
5. **모두 실패 시**: AskUserQuestion으로 "dev 서버 실행 명령어를 알려주세요" 질문

### server-info.json 기록

감지 또는 유저 답변 기반으로 서버를 실행하고 기록:

```json
{
  "frontend": "http://localhost:{감지된_포트}",
  "backend": "http://localhost:{감지된_포트}",
  "start_command": "감지된 실행 명령어",
  "stop_command": "감지된 종료 명령어"
}
```

## Generator 사용 시 차이점

기존 하네스의 generator.md를 그대로 사용하되, 오케스트레이터가 프롬프트에 아래 추가 지시를 삽입:

```
## 추가 지시 (기존 앱 개선 모드)
- 이것은 기존 앱 개선이다. 새 파일/폴더 구조를 생성하지 마라.
- 1라운드부터 수정 모드다. eval-round-0.md의 ❌ 항목만 처리하라.
- 기존 기능이 깨지지 않도록 최소한의 변경만 하라.
- frontend/, backend/ 디렉토리를 새로 만들지 마라. 기존 구조를 그대로 사용하라.
- 기존에 없던 의존성을 추가하지 마라 (스타일링 라이브러리 변경 등 금지).
```

## 오케스트레이터 상세

### 호출 방식

```
/harness-improve
/harness-improve "디자인 다듬어줘"
/harness-improve --threshold 9 --max-rounds 3
```

### 기본값

| 파라미터 | 기본값 |
|----------|--------|
| threshold | 8/10 |
| max-rounds | 5 |

### summary.md 형식

```markdown
# 하네스 개선 요약

- **프로젝트**: {프로젝트명}
- **총 라운드**: {ROUND}/{MAX_ROUNDS}
- **통과 기준**: {THRESHOLD}/10
- **최종 상태**: {PASS 또는 FAIL}

## Baseline → 최종 비교

| 기준 | Baseline | 최종 | 변화 |
|------|----------|------|------|
| 디자인 품질 | 6 | 8 | +2 |
| 독창성 | 5 | 7 | +2 |
| 기술력 | 7 | 8 | +1 |
| 기능성 | 8 | 9 | +1 |
| **종합** | **6.5** | **8.0** | **+1.5** |

## 점수 이력

| 라운드 | 디자인 | 독창성 | 기술력 | 기능성 | 종합 |
|--------|--------|--------|--------|--------|------|
| 0 (baseline) | 6 | 5 | 7 | 8 | 6.5 |
| 1 | 7 | 6 | 7 | 8 | 7.0 |
| 2 | 8 | 7 | 8 | 9 | 8.0 |
```
