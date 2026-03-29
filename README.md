# claude-harness

Anthropic 블로그 [Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)의 Generator-Evaluator 패턴을 Claude Code 스킬로 구현한 하네스.

## 핵심 아이디어

GAN에서 영감을 받아 **코드 생성(Generator)과 평가(Evaluator)를 별도 에이전트로 분리**합니다. 자기가 만든 코드를 자기가 평가하는 편향을 제거하고, Playwright MCP로 실제 브라우저에서 앱을 테스트합니다.

```
Planner → Generator → Evaluator → (피드백) → Generator → Evaluator → ... → 통과
```

## 스킬

### `/harness` — 신규 앱 생성

짧은 프롬프트에서 완전한 웹 앱을 생성합니다.

```
/harness "할일 관리 앱 만들어줘"
/harness "포트폴리오 사이트" --threshold 7 --max-rounds 3
/harness "SaaS 대시보드" --stack next
```

**흐름:**
1. **Planner** — 프롬프트를 제품 스펙으로 확장
2. **유저 승인** — 스펙 확인
3. **Generator → Evaluator 루프** — 코드 생성, Playwright 테스트, 피드백, 반복
4. **종료** — 점수 threshold 도달 또는 max-rounds 소진

### `/harness-improve` — 기존 앱 개선

이미 존재하는 프로젝트의 디자인/코드 품질을 개선합니다.

```
/harness-improve "디자인 다듬어줘"
/harness-improve                          # 프롬프트 없이도 가능 (baseline 기반 자동 판단)
/harness-improve --threshold 9 --max-rounds 3
```

**흐름:**
1. **Analyzer** — 기존 코드 분석, 현재 상태 스펙 생성
2. **Baseline 측정** — Evaluator가 현재 점수 측정
3. **Generator → Evaluator 루프** — 기존 코드 수정, 재평가, 반복
4. **종료** — baseline 대비 개선률 리포트

## 평가 기준

4가지 기준으로 10점 만점 채점:

| 기준 | 평가 내용 |
|------|-----------|
| **디자인 품질** | 색상, 타이포그래피, 레이아웃의 통일성 |
| **독창성** | 템플릿이 아닌 맞춤형 디자인 결정의 증거 |
| **기술력** | 간격 일관성, 색상 조화, 대비 비율, 반응형 |
| **기능성** | 사용자가 작업을 완료할 수 있는가 |

- **종합 점수** = 4개 평균
- **안전장치** = 어느 하나가 5 이하면 무조건 불통과
- **기본 threshold** = 8/10

## 옵션

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `--threshold` | 8 | 통과 기준 점수 (1-10) |
| `--max-rounds` | 5 | 최대 반복 횟수 |
| `--stack` | react-vite-fastapi | 기술 스택 (`/harness`만 해당) |

## 필수 환경

- [Claude Code](https://claude.ai/code)
- Playwright MCP 서버 설정:

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

## 설치

스킬 파일을 `~/.claude/skills/`에 복사합니다:

```bash
# 클론
git clone git@github.com:leekyungpyo-dev/claude-harness.git

# 스킬 디렉토리에 복사
cp -r claude-harness/harness ~/.claude/skills/harness
cp -r claude-harness/harness-improve ~/.claude/skills/harness-improve
```

Claude Code를 재시작하면 `/harness`와 `/harness-improve`가 스킬 목록에 나타납니다.

## 권장 워크플로우

```
1. (선택) brainstorming 스킬로 아이디어 정리
2. /harness "정리된 아이디어" → 자동 생성 + 평가 루프
3. /harness-improve → 생성된 앱 추가 개선
4. .harness/summary.md로 결과 확인
```

## 산출물

하네스 실행 시 프로젝트 루트에 `.harness/` 폴더가 생성됩니다:

```
.harness/
├── spec.md           # 제품 스펙
├── server-info.json  # dev 서버 접속 정보
├── eval-round-0.md   # baseline (harness-improve만)
├── eval-round-1.md   # 1회차 평가 리포트
├── eval-round-N.md   # N회차 평가 리포트
└── summary.md        # 최종 요약 (점수 이력, 결과)
```

## 참고

- [Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps) — Anthropic Engineering Blog
