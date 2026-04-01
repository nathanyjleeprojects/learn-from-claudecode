<!--
tags: llm-application/testing, llm-application/evaluation, architecture/quality-assurance
keywords: llm-testing, evaluation, mocking, snapshot-testing, deterministic-boundary, eval-framework, confidence-scoring, regression-testing
related_files: 04_tool_system.md, 07_retry_resilience.md, 18_patterns_and_lessons.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 24. Testing & Evaluation Patterns for LLM Applications

> LLM 애플리케이션을 어떻게 테스트하는가: 결정론적 경계 설계, LLM 응답 mocking, tool execution testing, evaluation framework, confidence-based 품질 관리.


## LLM 테스팅의 근본적 어려움

### 비결정성(Non-determinism)
- 동일 입력에 다른 출력 → 기존 assertion 방식 부적합
- Temperature > 0이면 매번 다른 응답
- 모델 업데이트로 기존 통과 테스트가 실패할 수 있음

### 무엇을 테스트할 것인가
```
LLM Application의 테스트 레이어:

1. Deterministic Layer (확실히 테스트 가능)
   ├── Input validation (Zod schema)
   ├── Tool parameter parsing
   ├── Security checks (Unicode sanitization, bash checks)
   ├── Token counting logic
   ├── Cache break detection
   └── Config loading / migration

2. Integration Layer (조건부 테스트 가능)
   ├── Tool execution (file I/O, bash)
   ├── API client (retry logic, error handling)
   ├── Streaming event processing
   └── Permission system

3. LLM Behavior Layer (eval로 측정)
   ├── 정확한 tool 선택
   ├── 올바른 tool parameter 생성
   ├── Multi-step 추론 품질
   └── Prompt 변경의 영향
```


## Pattern 1: Deterministic Boundary 설계

### 원칙: LLM과 비-LLM 코드를 명확히 분리

```typescript
// ✅ 테스트 가능: 순수 함수
function validateToolInput(input: unknown, schema: ZodSchema): Result {
  return schema.safeParse(input)
}

// ✅ 테스트 가능: 결정론적 변환
function normalizeUnicode(text: string): string {
  return text.normalize('NFKC').replace(/[\p{Cf}\p{Co}\p{Cn}]/gu, '')
}

// ❌ 직접 테스트 어려움: LLM 호출
async function generateCode(prompt: string): Promise<string> {
  return await callClaude(prompt) // 비결정적
}
```

### Claude Code의 실천
- **buildTool()**: schema validation과 execution을 분리. Schema 검증은 deterministic → 100% 테스트 가능
- **Security checks**: 23개 bash security check는 모두 deterministic → 각각 unit test 가능
- **Token estimation**: 수식 기반 → deterministic
- **Cache break detection**: hash 비교 → deterministic


## Pattern 2: LLM 응답 Mocking

### Mock 전략
```typescript
// API client를 mock하여 고정된 응답 반환
const mockAPIResponse = {
  content: [
    { type: "text", text: "I'll read the file." },
    { type: "tool_use", id: "toolu_123", name: "Read",
      input: { file_path: "/src/main.ts" } }
  ],
  stop_reason: "tool_use",
  usage: { input_tokens: 100, output_tokens: 50 }
}

// Tool loop 테스트: mock response → tool execution → result 검증
test('tool loop processes Read tool correctly', async () => {
  const api = mockAPI([mockAPIResponse, endTurnResponse])
  const result = await runToolLoop(api, "Read main.ts")
  expect(result.toolCalls).toHaveLength(1)
  expect(result.toolCalls[0].name).toBe('Read')
})
```

### 무엇을 Mock하는가
| Layer | Mock 여부 | 이유 |
|-------|----------|------|
| LLM API | ✅ Mock | 비용, 속도, 결정성 |
| File system | 상황에 따라 | 가능하면 real (integration test) |
| Bash execution | ✅ Mock | 부작용 방지 |
| Network | ✅ Mock | 외부 의존 제거 |
| Token counting | ❌ Real | Deterministic, 빠름 |

### Snapshot Testing
```typescript
// LLM의 system prompt 변경 감지
test('system prompt snapshot', () => {
  const prompt = buildSystemPrompt(defaultConfig)
  expect(prompt).toMatchSnapshot()
})
// → prompt 변경 시 snapshot 깨짐 → 의도적 변경인지 확인 강제
```


## Pattern 3: Tool Execution Testing

### 각 Tool의 독립 테스트
```typescript
// Read tool 테스트
test('Read tool returns file content', async () => {
  const result = await ReadTool.call(
    { file_path: '/tmp/test.txt' },
    mockContext
  )
  expect(result.data).toContain('file content')
})

// Permission 테스트
test('Read tool grants permission for allowed paths', async () => {
  const perm = await ReadTool.checkPermissions(
    { file_path: '/project/src/main.ts' },
    contextWithRules([{ allow: 'Read(/project/*)' }])
  )
  expect(perm.granted).toBe(true)
})
```

### buildTool()이 테스트를 쉽게 만드는 이유
- 모든 tool이 같은 인터페이스: `call(args, context) → { data, newMessages }`
- Permission check가 분리: `checkPermissions(input, context) → { granted }`
- Schema validation이 자동: Zod schema가 잘못된 입력을 거부


## Pattern 4: Evaluation Framework

### 3-Tier Evaluation (Hamel Husain 모델 적용)

```
Tier 1: Unit Tests (자동, 빠름)
  - Deterministic 코드의 기존 unit test
  - Schema validation, security checks
  - 실행: 매 commit

Tier 2: Model Evaluation (준자동, 중간)
  - LLM을 evaluator로 사용
  - 예: "이 tool 선택이 적절한가?" → LLM이 판정
  - Confidence scoring (high/medium/low)
  - 실행: 매 PR 또는 prompt 변경 시

Tier 3: Human Evaluation (수동, 느림)
  - 실제 사용자의 만족도
  - Edge case 발견
  - 실행: 주기적 또는 major 변경 시
```

### Confidence-Based Quality Control (code-review 플러그인 패턴)
```typescript
// 4개 parallel agent가 각각 findings 반환
const findings = await Promise.all([
  agent1.review(code),
  agent2.review(code),
  agent3.review(code),
  agent4.review(code),
])

// Confidence filtering
const highConfidence = findings
  .flat()
  .filter(f => f.confidence === 'high')

// High confidence만 최종 리포트에 포함
return formatReport(highConfidence)
```

**왜 이 패턴이 강력한가**:
- LLM은 가끔 false positive를 만든다
- 여러 agent의 합의(consensus) → false positive 감소
- Confidence score로 필터링 → precision 향상

### Eval-Driven Skill Development (skill-creator 패턴)
```
1. Skill 작성
2. Test case 정의 (input → expected behavior)
3. Skill 실행 → 결과 수집
4. 결과 평가 (자동 + 수동)
5. Skill 개선
6. 반복
```


## Pattern 5: Regression Testing for Prompts

### 문제: Prompt 변경의 나비 효과
- System prompt의 작은 변경이 예상치 못한 행동 변화 야기
- "Do not add comments" 추가 → 기존에 잘 되던 기능이 깨질 수 있음

### 대응 전략
1. **Prompt snapshot testing**: 변경 감지
2. **Golden test suite**: 핵심 시나리오의 기대 행동 정의
3. **A/B comparison**: 변경 전/후 동일 입력으로 비교
4. **Feature flags**: 새 prompt를 flag 뒤에 숨기고 점진적 배포

```typescript
// Golden test: 이 시나리오에서 LLM이 Read tool을 선택해야 한다
const goldenTests = [
  {
    input: "What's in main.ts?",
    expectedTool: "Read",
    expectedArgs: { file_path: expect.stringContaining("main.ts") }
  },
  {
    input: "Find all TODO comments",
    expectedTool: "Grep",
    expectedArgs: { pattern: expect.stringContaining("TODO") }
  }
]
```


## LLM 엔지니어링 인사이트

1. **Deterministic boundary가 테스트 전략의 핵심**: LLM 호출을 최소한의 접점으로 격리하면, 나머지 코드는 전통적 방법으로 100% 테스트 가능
2. **Mock은 LLM API에만**: File system, DB는 가능하면 real로 (Report 07의 교훈 — mock과 prod의 괴리)
3. **Confidence scoring으로 LLM의 불확실성을 관리**: "맞을 수도 있고 틀릴 수도 있다" 대신 "이건 확신도 높음 / 낮음"
4. **Prompt 변경 = 코드 변경**: Prompt도 버전 관리, 테스트, 점진적 배포가 필요
5. **Eval은 TDD**: "먼저 eval을 정의하고, eval을 통과하도록 prompt를 개선"하는 workflow가 가장 효과적
6. **비용 인식 테스팅**: LLM 호출 테스트는 비용이 발생. Mock으로 일상 테스트, real API로 주기적 eval


## 적용하기: 테스트 레이어 설계

```
┌─────────────────────────────────────────────┐
│ CI Pipeline (매 commit)                      │
│  └── Unit tests (deterministic code)         │
│  └── Snapshot tests (prompt 변경 감지)        │
│  └── Tool execution tests (mocked API)       │
├─────────────────────────────────────────────┤
│ PR Pipeline (매 PR)                          │
│  └── Integration tests (tool + mock LLM)     │
│  └── Golden tests (핵심 시나리오)              │
├─────────────────────────────────────────────┤
│ Weekly/Release Pipeline                      │
│  └── Model evaluation (real API)             │
│  └── Regression suite (prompt 변경 영향)      │
│  └── Cost analysis (토큰 사용량 추적)         │
├─────────────────────────────────────────────┤
│ Periodic (manual)                            │
│  └── Human evaluation                        │
│  └── Edge case exploration                   │
└─────────────────────────────────────────────┘

---

## 대안 비교

| 접근 방식 | 비용 | 속도 | 커버리지 | 결정성 | 적합한 단계 |
|----------|------|------|---------|--------|-----------|
| **Manual Testing** | 높음 (인건비) | 느림 | 낮음 (사람의 한계) | ❌ | 초기 프로토타입, edge case 발굴 |
| **Automated Evals (LLM-as-judge)** | 중간 (API 비용) | 중간 | 높음 | ❌ (LLM도 확률적) | PR 단위 품질 검증, prompt 변경 영향 측정 |
| **A/B Testing** | 높음 (2배 트래픽) | 느림 (통계적 유의성) | 높음 | ✅ (통계적) | 프로덕션 prompt 변경, 모델 교체 |
| **Shadow Mode (prod 병행)** | 중간 | 실시간 | 최고 | ✅ | 모델 업그레이드 전 검증, 새 prompt 배포 전 |
| **Deterministic Unit Tests** | 최저 | 최고 | 제한적 (비-LLM만) | ✅ 100% | 매 commit, CI 파이프라인 |

**권장 조합**: Deterministic unit tests(매 commit) + Automated evals(매 PR) + Shadow mode(major 변경). Manual testing은 새로운 기능 탐색에만 사용하고, A/B testing은 비용이 정당화되는 핵심 기능 변경에만 적용한다.

---

## 내 프로젝트에 적용하기

### Step 1: Deterministic Tests 먼저 구축 (Security, Schema)

LLM 호출을 포함하지 않는 **결정론적 코드를 먼저 100% 테스트**한다:
- **보안 검증**: Unicode sanitization, bash injection check, API key 패턴 탐지 → 각각 regex 기반이므로 100% deterministic
- **Schema validation**: Zod/Pydantic으로 tool input 검증 → `safeParse(input)` 결과를 assertion
- **Token counting**: 수식 기반 추정 → deterministic
- **Config loading**: 계층적 config 병합 로직 → deterministic

이 레이어만으로도 전체 코드의 60-70%를 테스트할 수 있다. CI에서 매 commit마다 실행한다.

### Step 2: LLM 응답 Mocking으로 Tool Loop 테스트

LLM API를 mock하여 **tool 선택 → 실행 → 결과 처리 루프**를 테스트한다:
```typescript
const mockResponse = { content: [{ type: "tool_use", name: "Read", input: { file_path: "/src/main.ts" } }] }
const api = mockAPI([mockResponse, endTurnResponse])
const result = await runToolLoop(api, "Read main.ts")
expect(result.toolCalls[0].name).toBe('Read')
```
Mock하는 것: LLM API, Bash execution, Network calls. Mock하지 않는 것: File system(가능하면 real), Token counting(deterministic). 이렇게 하면 비용 없이 tool loop의 정확성을 빠르게 검증할 수 있다.

### Step 3: Golden Test Suite로 Prompt Regression 방지

핵심 시나리오의 **기대 행동을 정의한 golden test suite**를 구축한다:
```typescript
const goldenTests = [
  { input: "What's in main.ts?", expectedTool: "Read" },
  { input: "Find all TODO comments", expectedTool: "Grep" },
  { input: "Create a new component", expectedTool: "Write" }
]
```
- System prompt 변경 시 golden tests를 real API로 실행하여 regression 감지
- Prompt snapshot testing으로 의도하지 않은 prompt 변경을 CI에서 차단
- Feature flag 뒤에 새 prompt를 숨기고, golden test 통과 후 점진적 배포
- 주기적(weekly) eval로 모델 업데이트에 의한 행동 변화도 추적
```
