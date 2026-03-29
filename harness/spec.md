# Harness: Generator-Evaluator 앱 생성 하네스

> Anthropic 블로그 "Harness Design for Long-Running Apps"의 GAN 영감 패턴을
> Claude Code 스킬로 구현한 하네스.

## 핵심 사상

1. **생성과 평가의 컨텍스트 분리** — Generator와 Evaluator는 별도 에이전트로 실행. 자기평가 편향 제거.
2. **파일 기반 통신** — 에이전트 간 `.harness/` 폴더의 파일로 소통. 매 라운드 깨끗한 컨텍스트.
3. **실행 가능한 피드백** — "개선 필요" (X) → "h1을 2.5rem으로 변경하라" (O)
4. **Playwright 기반 실제 검증** — 코드 리뷰가 아닌 실제 브라우저에서 앱을 사용자처럼 테스트.

## 아키텍처

### 파일 구조

```
~/.claude/skills/harness/
├── SKILL.md              # 오케스트레이터 (진입점, /harness)
├── planner.md            # Planner 에이전트 프롬프트
├── generator.md          # Generator 에이전트 프롬프트
├── evaluator.md          # Evaluator 에이전트 프롬프트
└── rubric.md             # 평가 루브릭 (4가지 기준 + 점수 체계)
```

### 프로젝트 내 생성 파일

```
<project-root>/
├── frontend/             # React + Vite + TailwindCSS
├── backend/              # FastAPI
├── .harness/
│   ├── spec.md           # Planner 산출물
│   ├── server-info.json  # dev 서버 접속 정보
│   ├── eval-round-1.md   # 1회차 평가 리포트
│   ├── eval-round-N.md   # N회차 평가 리포트
│   └── summary.md        # 최종 요약
```

### 실행 흐름

```
유저: /harness "할일 관리 앱 만들어줘"
  │
  ▼
[오케스트레이터]
  │
  ├─1. 초기화
  │    - .harness/ 디렉토리 생성
  │    - 인자 파싱 (threshold 기본 8, max-rounds 기본 5)
  │    - .harness/spec.md 이미 존재 시 → "기존 스펙 사용?" 확인
  │    - brainstorming 안내: "더 정교한 스펙이 필요하면 /brainstorm을 먼저 사용해보세요"
  │
  ├─2. Planner 에이전트 호출
  │    → .harness/spec.md 생성
  │    → 유저에게 spec 요약 + 승인 요청
  │    → 거부 시 수정 반영 후 재실행
  │
  ├─3. Build-Eval 루프 (max N회, threshold M/10)
  │    │
  │    │  [Generator 에이전트]
  │    │   ← reads: spec.md + eval-round-N.md (있으면)
  │    │   → 코드 생성/수정
  │    │   → dev 서버 실행
  │    │   → .harness/server-info.json 기록
  │    │
  │    │  [Evaluator 에이전트]
  │    │   Phase A: 검증 계획 수립
  │    │    ← reads: spec.md
  │    │    → 이번 라운드 성공 기준 목록 작성
  │    │
  │    │   Phase B: 실제 검증
  │    │    → Playwright MCP로 앱 접속
  │    │    → 네비게이션, 클릭, 폼 입력 테스트
  │    │    → 스크린샷 캡처 → 디자인 품질 평가
  │    │    → 코드 리뷰 (구조, 보안, 패턴)
  │    │
  │    │   Phase C: 실행 가능한 피드백 작성
  │    │    → .harness/eval-round-N.md
  │    │    → ✅/❌ 검증 결과 + 구체적 수정 지시 + 4기준 점수
  │    │    → 종합 점수 ≥ threshold → BREAK
  │    │
  │    │  오케스트레이터: "Round N: 6.5/10 — 수정 지시 2건. 계속 진행합니다."
  │    │
  │
  └─4. 종료
       → dev 서버 정리
       → .harness/summary.md 생성 (점수 이력, 최종 상태)
       → max-rounds 도달 시 최종 점수 + 잔여 이슈 안내
```

## Planner 상세

### 역할

유저의 짧은 프롬프트(1~4문장)를 Generator가 바로 코드를 쓸 수 있는 수준의 제품 스펙으로 확장.

### 원칙

1. **제품 맥락 우선, 기술 디테일은 최소** — 블로그 원문: "세부 기술 사항보다는 제품 맥락과 고수준 설계에 집중"
2. **제품적으로 필요한 기능은 적극 제안** — 유저 승인 단계에서 걸러짐
3. **기술 스택 기본값 제공, 오버라이드 가능** — 유저가 지정하면 존중, 아니면 기본값
4. **명시적 제외 사항 기재** — 범위를 명확히 해서 Generator가 멋대로 확장하지 않도록

### 기본 기술 스택 (유저 미지정 시)

| 레이어 | 기본값 | 이유 |
|--------|--------|------|
| Frontend | React + Vite + TailwindCSS | 빠른 부트스트랩, Claude가 가장 잘 아는 조합 |
| Backend | FastAPI | 가볍고 빠름, Python 생태계 |
| DB | SQLite | 설치 불필요, 프로토타입에 최적 |
| SaaS급 | PostgreSQL + 인증 추가 | 유저가 명시적으로 요청 시 |

### 산출물 형식: `.harness/spec.md`

```markdown
# 프로젝트: {프로젝트명}

## 제품 개요
- 목적, 타겟 사용자

## 기술 스택
- Frontend, Backend, DB 등

## 페이지 구조
- 각 페이지의 경로와 역할

## 데이터 모델
- 주요 엔티티와 필드

## 핵심 기능
- 번호 매긴 기능 목록

## 디자인 방향
- 전반적 톤, 스타일 가이드

## 명시적 제외 사항
- 이 프로젝트에서 만들지 않는 것
```

## Generator 상세

### 역할

spec.md를 읽고 실제 코드를 생성/수정하고, dev 서버를 띄운다.

### 입력 (매 라운드)

- `.harness/spec.md` — 항상
- `.harness/eval-round-N.md` — 2라운드부터 (이전 평가 피드백)

### 행동 규칙

1. **1라운드**: spec 기반으로 전체 앱 생성
2. **2라운드~**: eval 피드백의 ❌ 수정 지시 목록만 처리. ✅ 항목은 건드리지 않음
3. 코드 생성 후 dev 서버 실행 → `server-info.json` 기록. 2라운드~: 기존 dev 서버가 실행 중이면 코드 수정 후 핫 리로드 확인. 서버가 죽었으면 재시작
4. 프로젝트 구조 컨벤션 준수 (`frontend/`, `backend/`, `.harness/`). 백엔드 없는 프론트엔드 전용 프로젝트는 `frontend/` 단일 구조

### 서버 실행 규칙

1. 백엔드 먼저 실행
2. 프론트엔드 실행
3. `.harness/server-info.json`에 기록:

```json
{
  "frontend": "http://localhost:{port}",
  "backend": "http://localhost:{port}",
  "start_command": "실행 명령어",
  "stop_command": "종료 명령어"
}
```

프론트엔드 전용 프로젝트는 `backend` 필드를 `null`로 기록.

### 금지 사항

- spec에 없는 기능을 추가하지 않는다
- 평가에서 지적하지 않은 부분을 "개선"하지 않는다

## Evaluator 상세

### 역할

실행 중인 앱을 사용자 관점에서 검증하고, Generator가 바로 행동할 수 있는 구체적 피드백을 작성한다.

### 입력

- `.harness/spec.md` — 검증 기준의 원천
- `.harness/server-info.json` — 접속 정보
- `.harness/eval-round-{N-1}.md` — 이전 평가 리포트 (2라운드부터, 이전 ❌ 항목 재검증용)
- 프로젝트 코드 — 코드 리뷰용

### 3단계 실행

#### Phase A: 검증 계획

spec.md를 읽고 이번 라운드에서 검증할 성공 기준을 체크리스트로 작성.
2라운드 이후에는 이전 eval의 ❌ 항목을 우선 재검증 대상으로 포함.

#### Phase B: 실제 검증

1. `server-info.json`에서 URL 읽기
2. Playwright MCP로 브라우저 접속
3. 검증 계획의 각 항목을 실제로 테스트 (네비게이션, 클릭, 폼 입력, 제출)
4. 각 페이지에서 스크린샷 캡처
5. 스크린샷 기반 디자인 품질 평가
6. 코드 리뷰 (구조, 보안, 패턴)

#### Phase C: 피드백 작성

### 평가 기준 (4가지)

| 기준 | 평가 내용 |
|------|-----------|
| 디자인 품질 | 색상, 타이포그래피, 레이아웃이 통일된 분위기와 정체성을 형성하는가 |
| 독창성 | 맞춤형 결정의 증거가 있는가, 아니면 템플릿/AI 생성 패턴인가 |
| 기술력 | 타이포그래피 계층, 간격 일관성, 색상 조화, 대비 비율 |
| 기능성 | 사용자가 인터페이스를 이해하고 작업을 완료할 수 있는가 |

### 평가 리포트 형식: `.harness/eval-round-N.md`

```markdown
## 검증 결과
- ✅ {통과 항목}
- ❌ {실패 항목}

## 수정 지시
1. [{카테고리}] {문제 설명} → {구체적 수정 방법}
2. [{카테고리}] {문제 설명} → {구체적 수정 방법}

## 점수
| 기준 | 점수 | 메모 |
|------|------|------|
| 디자인 품질 | N/10 | ... |
| 독창성 | N/10 | ... |
| 기술력 | N/10 | ... |
| 기능성 | N/10 | ... |
| **종합** | **N/10** | |
```

### Evaluator 원칙

1. **✅도 명시** — Generator가 잘 되는 부분을 건드리지 않게
2. **수정 지시는 구체적으로** — "디자인 개선 필요" (X) → "h1을 2.5rem으로 변경하라" (O)
3. **spec에 없는 기준으로 채점하지 않는다** — spec이 계약
4. **종합 점수 = 4개 기준의 평균, 단 어느 하나가 5 이하면 무조건 불통과** — 평균으로 전반적 품질을 반영하되, 치명적 결함은 안전장치로 잡는다

## 오케스트레이터 상세

### 호출 방식

```
/harness "프롬프트"
/harness "프롬프트" --threshold 7 --max-rounds 3
/harness "프롬프트" --stack next
```

### 기본값

| 파라미터 | 기본값 |
|----------|--------|
| threshold | 8/10 |
| max-rounds | 5 |
| stack | react-vite-fastapi |

### 오케스트레이터가 직접 하지 않는 것

- 코드 생성/수정 — Generator의 몫
- 평가/채점 — Evaluator의 몫
- 스펙 작성 — Planner의 몫

오케스트레이터는 **루프 관리 + 파일 전달 + 유저 소통**만 한다.

## 워크플로우 가이드

권장 사용 흐름:

```
1. (선택) /brainstorm → 아이디어 정리
2. /harness "정리된 아이디어" → 자동 생성 + 평가 루프
3. .harness/summary.md로 결과 확인
```

이 흐름은 ~/.claude/CLAUDE.md에도 기록하여 매 세션에서 참조 가능하게 한다.
