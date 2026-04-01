<!--
tags: architecture/design-patterns, agents/defensive-programming, architecture/graceful-degradation, prompt-engineering/context-budget, llm-application/lessons-learned, agents/multi-agent, security/classification, agents/verification
keywords: design-patterns, lessons-learned, anti-patterns, defensive-llm-programming, graceful-degradation, context-budget, schema-single-source-of-truth, self-contained-modules, parallel-by-default, permission-as-ux, adversarial-verification, lazy-delegation, two-stage-classifier, synthesize-never-delegate, actionable-checklist
related_files: 00_index.md, 02_query_engine_core.md, 03_prompt_engineering.md, 04_tool_system.md, 05_context_management.md, 07_retry_resilience.md, 09_security_model.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 18. Patterns & Lessons

> 512K 줄의 프로덕션 LLM 코드에서 추출한 cross-cutting 설계 패턴, anti-patterns, 그리고 "내가 내일 LLM 앱을 만든다면" actionable checklist. Claude Code 전체 분석의 최종 종합.

## Keywords
design-patterns, lessons-learned, anti-patterns, defensive-llm-programming, graceful-degradation, context-budget-first-class, schema-single-source-of-truth, self-contained-modules, parallel-by-default, permission-as-ux, adversarial-verification, synthesize-never-delegate, two-stage-security-classifier, lazy-delegation, llm-engineering-checklist

---

## Pattern 1: Self-Contained Modules with Shared Contracts

**관찰**: Claude Code의 42개 tool은 각각 독립된 디렉토리에 schema, permission, execution, UI, prompt를 모두 포함한다. `buildTool()` factory가 공통 계약을 강제한다.

**원칙**: 모듈은 "내가 필요한 모든 것을 나 안에 갖고 있다" + "외부와의 상호작용은 표준 인터페이스로만 한다."

**적용하기**:
- 새 기능(tool, command, plugin)을 추가할 때, 기존 파일을 수정하지 않고 새 디렉토리만 추가하면 되는 구조인가?
- Schema, validation, implementation, test가 한 곳에 있는가?
- 모듈 간 의존성이 interface(contract)를 통해서만 존재하는가?

**왜 LLM 앱에서 특히 중요한가**: Tool이 자주 추가/제거/수정된다. LLM의 행동이 변하면 tool의 prompt를 바꿔야 하고, tool의 기능이 변하면 schema를 바꿔야 한다. 이 모든 것이 한 디렉토리에 있으면 변경의 blast radius가 최소화된다.

---

## Pattern 2: Defensive LLM Programming

**관찰**: Claude Code는 LLM이 "틀릴 수 있다"는 전제 하에 모든 것을 설계한다.

**구체적 방어 기법**:

1. **Tool 에러를 LLM에게 반환**: Tool이 실패하면 앱이 크래시하는 대신, 에러 메시지를 tool_result로 LLM에게 돌려보낸다. LLM은 에러를 읽고 다른 접근법을 시도할 수 있다.

2. **Permission 거부를 LLM에게 알림**: 사용자가 tool 사용을 거부하면, "사용자가 이 작업을 거부했습니다"라는 정보가 LLM에게 전달된다. LLM은 대안을 찾는다.

3. **부정 지시의 구체성**: "Don't add features beyond what was asked", "Three similar lines is better than a premature abstraction" 같은 구체적 부정 지시로 LLM의 "과도한 도움" 경향을 억제한다.

4. **Input validation at every boundary**: Zod schema로 tool 입력을 검증하고, Unicode sanitization으로 텍스트를 정화하며, 23개 보안 체크로 shell 명령을 검증한다.

**적용하기**: "LLM이 이 부분에서 실수하면 어떻게 되는가?"를 매 코드 경로마다 물어라. 대부분의 경우, 실수를 LLM에게 피드백하여 self-correction하게 하는 것이 최선이다.

---

## Pattern 3: Graceful Degradation Chains

**관찰**: Claude Code의 모든 외부 서비스는 실패해도 전체 앱이 동작한다.

**Degradation chain 예시**:

```
Opus → 529 overload → Sonnet fallback → 성공
Fast mode → 429 rate limit → Standard mode → 성공
OAuth token → 만료 → Token refresh → 성공 → API key fallback
Prompt cache → cache break → 재생성 → 성공 (비용 증가)
MCP 서버 → 연결 실패 → MCP 도구 비활성화 → 기본 도구로 동작
GrowthBook → 초기화 실패 → 모든 flag = false → 기본 기능만 사용
```

**핵심 질문**: "이 서비스가 완전히 죽으면 앱은 어떻게 되는가?"

| 서비스 | 실패 시 동작 | 사용자 영향 |
|--------|------------|-----------|
| API (Direct) | **Fatal** | 앱 기능 불가 |
| MCP 서버 | MCP 도구만 비활성 | 기본 도구로 작업 가능 |
| GrowthBook | 모든 flag = false | 실험 기능 비활성 |
| Policy limits | 제한 없이 동작 | 비용 통제 불가 |
| Keychain | API key 수동 입력 | 약간 불편 |

**적용하기**: Hard dependency는 API client 하나뿐이어야 한다. 나머지는 모두 graceful하게 실패해야 한다.

---

## Pattern 4: Context Budget as First-Class Concern

**관찰**: Claude Code의 거의 모든 설계 결정에 "context window를 얼마나 차지하는가?"라는 질문이 따라붙는다.

**구체적 사례**:

- **Tool description caching**: ~11K tokens을 매번 재생성하지 않고 캐시
- **Deferred tools**: MCP tool schema를 필요할 때만 로드하여 기본 prompt 크기 억제
- **Compact/Microcompact**: Context가 차면 능동적으로 압축
- **String replacement**: 파일 편집에 전체 파일이 아닌 diff만 전송
- **Progress message 삭제**: Compact 시 중간 상태는 제거하고 최종 결과만 보존
- **Memory relevance filtering**: 모든 memory가 아닌 관련 memory만 주입

**적용하기**: 모든 기능을 추가할 때 "이것이 context에 추가하는 token은 몇 개이고, 그만한 가치가 있는가?"를 질문하라.

---

## Pattern 5: Permission as UX, Not Just Security

**관찰**: Claude Code의 permission 시스템은 보안 게이트가 아니라 **사용자와의 투명한 소통 채널**이다.

**왜 이것이 패턴인가**:
- Permission 요청 = "이 작업을 하려고 합니다, 이유는 X입니다"
- 사용자가 LLM의 계획을 이해하고 교정할 기회 제공
- 거부 시 LLM에게 피드백 → 대안 탐색
- Wildcard rule로 반복적 승인 불편 해소

**적용하기**: 위험한 작업에 대한 사용자 확인을 "보안 기능"이 아닌 "사용자 교육 + 신뢰 구축"의 기회로 설계하라. "이 명령을 실행해도 될까요?"보다 "이 명령은 X를 Y로 변경합니다. 실행하시겠습니까?"가 더 좋다.

---

## Pattern 6: Parallel-By-Default, Sequential-When-Needed

**관찰**: Claude Code는 가능한 모든 곳에서 병렬 실행을 기본으로 한다.

**병렬화 사례**:
- Startup: MDM + Keychain + API preconnect 병렬
- Tool execution: `isConcurrencySafe()` = true인 tool들 병렬
- Sub-agent: Team agent들 병렬 실행
- Event upload: SerialBatchEventUploader로 배치 병렬화

**순차 실행이 필요한 경우**:
- Tool이 `isConcurrencySafe()` = false (파일 쓰기 등)
- 의존성이 있는 작업 (파일 읽기 → 분석 → 편집)
- Permission 승인 대기

**적용하기**: "이 작업들은 독립적인가?"를 먼저 물어라. 독립적이면 병렬, 의존적이면 순차. 각 작업이 자신의 독립성을 self-declare하는 패턴이 깔끔하다 (`isConcurrencySafe()`).

---

## Pattern 7: Schema as Single Source of Truth

**관찰**: Claude Code에서 Zod schema는 5가지 목적을 동시에 수행한다.

```
Zod Schema (하나의 정의)
  → TypeScript 타입 (z.infer<>)
  → JSON Schema (API tools 파라미터)
  → LLM 프롬프트 (tool description)
  → Runtime validation (사용자/LLM 입력 검증)
  → Config documentation (어떤 설정이 가능한가)
```

**적용하기**: Schema를 한 곳에서 정의하고 여러 목적으로 활용하라. Schema가 중복되면 불일치가 발생하고, 불일치는 LLM이 잘못된 tool 입력을 생성하는 원인이 된다.

---

## Pattern 8: Adversarial Verification — "Your Job Is to Try to Break It"

**관찰**: Claude Code의 검증 에이전트는 "구현이 동작하는지 확인"하는 것이 아니라 **"구현을 깨뜨리는 것"**이 임무다. 이것은 LLM 에이전트 시스템에서 가장 과소평가된 패턴이다.

### 핵심 원칙: 검증은 실행이다

```
"A check without a command run is not a PASS — it's a skip."
```

모든 검증은 반드시 다음 구조를 따라야 한다:

```
### Check: [무엇을 검증하는가]
**Command run:**
  [실제 실행한 명령]
**Output observed:**
  [실제 터미널 출력 — copy-paste, 요약 아님]
**Result: PASS** (또는 FAIL — Expected vs Actual 포함)
```

코드를 "읽고" PASS를 부여하는 것은 검증이 아니다. **실행**해야 검증이다.

### Self-Rationalization Detection

LLM 검증 에이전트의 가장 큰 위험은 **자기 합리화**다. Claude Code는 검증 에이전트가 빠지기 쉬운 자기 합리화 패턴을 명시적으로 나열하고, "이런 생각이 들면 반대로 행동하라"고 지시한다:

| 자기 합리화 (이런 생각이 들면...) | 올바른 반응 |
|---|---|
| "The code looks correct based on my reading" | Reading is not verification. **Run it.** |
| "The implementer's tests already pass" | The implementer is an LLM. **Verify independently.** |
| "This is probably fine" | Probably is not verified. **Run it.** |
| "I don't have a browser" | Did you check for mcp__playwright__*? **Check your tools.** |
| "This would take too long" | **Not your call.** |

**왜 이것이 중요한가**: LLM은 본질적으로 "그럴듯한 설명"을 생성하도록 훈련되어 있다. 검증 에이전트가 이 성질에 빠지면, "코드가 맞아 보인다"는 자기 합리화로 실제 버그를 놓치게 된다. **명시적인 자기 합리화 목록**은 이 문제를 구조적으로 방지한다.

### Change-Type-Specific Verification Strategies

Claude Code는 변경 유형별로 검증 전략을 다르게 지시한다:

| 변경 유형 | 검증 전략 |
|---|---|
| **Frontend** | Dev server 시작 → 브라우저 자동화(Playwright)로 실제 클릭/스크린샷 → subresource curl → 프론트엔드 테스트 |
| **Backend/API** | Server 시작 → endpoint curl → response shape 검증 → error handling → edge cases |
| **Bug fix** | 원래 버그 재현 → 수정 확인 → regression 테스트 → 관련 기능 side effect 확인 |
| **Refactoring** | 기존 테스트 변경 없이 통과 → public API surface diff → 동일 input → 동일 output 확인 |
| **DB migration** | Migration up → schema 검증 → migration down(가역성) → 기존 데이터 대상 테스트 |
| **Infra/config** | Syntax validate → dry-run (terraform plan, kubectl --dry-run) → env var 실제 참조 확인 |

### Adversarial Probes — "First 80% Is the Easy Part"

```
"You see a polished UI or a passing test suite and feel inclined to pass it,
not noticing half the buttons do nothing, the state vanishes on refresh,
or the backend crashes on bad input. The first 80% is the easy part.
Your entire value is in finding the last 20%."
```

필수 adversarial probe 유형:
- **Concurrency**: 병렬 요청 → duplicate 생성? lost write?
- **Boundary values**: 0, -1, empty string, MAX_INT, unicode
- **Idempotency**: 동일 mutating request 2회 → duplicate? error? correct no-op?
- **Orphan operations**: 존재하지 않는 ID에 대한 delete/reference

**적용하기**: 검증 에이전트를 만들 때, (1) 파일 수정 권한을 제거하고, (2) 자기 합리화 목록을 프롬프트에 명시하고, (3) 모든 체크에 실행 증거를 요구하라. 검증의 가치는 "PASS를 내는 것"이 아니라 **"FAIL을 찾아내는 것"**이다.

---

## Pattern 9: "Synthesize, Never Delegate Lazily"

**관찰**: Claude Code의 coordinator(조정자)는 multi-agent 오케스트레이션에서 가장 중요한 역할이 **"종합(synthesis)"**이라고 명시한다. Worker에게 작업을 넘기기 전에, coordinator가 반드시 조사 결과를 이해하고 구체적 명세로 변환해야 한다.

### 핵심 원칙: Worker는 Coordinator 대화를 볼 수 없다

```
"Workers can't see your conversation. Every prompt must be self-contained."
```

이것이 multi-agent 시스템의 가장 흔한 실패 원인이다. Coordinator가 "based on your findings, fix the bug"라고 보내면, worker는 어떤 findings인지, 어떤 bug인지 모른다. **모든 명세에는 exact file paths, line numbers, 정확히 무엇을 바꿔야 하는지가 포함되어야 한다.**

### Anti-pattern vs Pattern: Lazy Delegation vs Synthesized Spec

```python
# BAD — lazy delegation
agent.spawn(prompt="Based on your findings, fix the auth bug")
agent.spawn(prompt="The worker found an issue in the auth module. Please fix it.")
agent.spawn(prompt="Something went wrong with the tests, can you look?")

# GOOD — synthesized spec
agent.spawn(prompt="""
Fix the null pointer in src/auth/validate.ts:42.
The `user` field on Session (src/auth/types.ts:15) is undefined when
sessions expire but the token remains cached.
Add a null check before user.id access — if null, return 401 with
'Session expired'.
Run tests and type checker. Commit and report the hash.
""")
```

### Continue vs Spawn 의사결정 프레임워크

종합이 끝나면, worker의 기존 context가 다음 작업에 도움이 되는지 판단한다:

| 상황 | 방식 | 이유 |
|---|---|---|
| 조사 범위 = 수정 범위 | **Continue** (SendMessage) | Worker가 이미 해당 파일 context를 보유 |
| 조사는 넓었지만 구현은 좁음 | **Spawn fresh** (Agent) | 탐색 잡음을 끌고 가지 않기 위해 |
| 실패 수정 / 직전 작업 연장 | **Continue** | Worker가 오류 context를 보유 |
| 다른 worker가 작성한 코드 검증 | **Spawn fresh** | 구현 가정 없이 새 시각으로 |
| 첫 시도가 완전히 잘못된 접근 | **Spawn fresh** | 잘못된 접근의 context가 재시도를 오염 |
| 완전히 무관한 작업 | **Spawn fresh** | 재사용할 context가 없음 |

**기본값은 없다.** Context 겹침이 크면 continue, 작으면 spawn.

### 4-Phase Workflow: Research → Synthesis → Implementation → Verification

| Phase | 담당 | 핵심 원칙 |
|---|---|---|
| **Research** | Workers (병렬) | 여러 각도를 동시에 조사. 읽기 전용. |
| **Synthesis** | **Coordinator** | 조사 결과를 이해하고 구체적 명세로 변환. **이 단계를 건너뛰면 모든 것이 무너진다.** |
| **Implementation** | Workers | 명세에 따라 구현. 파일 집합별로 하나씩. 커밋 전 자체 검증. |
| **Verification** | Workers (독립) | 구현 worker와 별개의 worker가 검증. 검증 = 실행 (Pattern 8 참조). |

### 목적 문장 (Purpose Statement)

Worker가 깊이와 강조점을 조절할 수 있도록 짧은 목적 문장을 포함:

```
"이 조사는 PR 설명 작성에 쓰일 예정입니다. 사용자에게 보이는 변경에 집중하세요."
"구현 계획을 세우는 데 필요합니다. 파일 경로, 줄 번호, 타입 시그니처를 보고하세요."
"merge 전 빠른 점검입니다. happy path만 검증하세요."
```

**적용하기**: Multi-agent 시스템을 만들 때, coordinator의 가장 중요한 역할은 "작업 분배"가 아니라 **"지식 종합"**이다. Worker에게 보내는 모든 프롬프트가 self-contained인지 검증하라. "based on your findings"라는 문구가 있으면, 그것은 lazy delegation의 신호다.

---

## Pattern 10: Two-Stage Security Classification

**관찰**: Claude Code의 auto-mode(이전 명칭 "YOLO mode")는 **2-stage 보안 분류기**로 tool 호출을 자동 승인/차단한다. 이것은 LLM 에이전트의 자율성과 안전성 사이 균형을 잡는 정교한 메커니즘이다.

### Architecture: Safe Bypass → Stage 1 (Fast) → Stage 2 (Extended Reasoning)

```
Tool 호출 발생
  │
  ├─ Safe tool? (읽기 전용: Read, Glob, Grep) ──→ 즉시 승인 (classifier 건너뜀)
  │
  └─ 위험 가능성 있는 tool ──→ Stage 1: Fast Classification
                                  │
                                  ├─ 명확히 안전 ──→ 승인
                                  ├─ 명확히 위험 ──→ 차단
                                  └─ 불확실 ──→ Stage 2: Extended Reasoning
                                                  │
                                                  ├─ 승인 (with reasoning)
                                                  └─ 차단 (with reason)
```

### Self-Manipulation Prevention: Assistant Text Exclusion

분류기가 받는 대화 기록(transcript)에서 **assistant의 텍스트 블록은 의도적으로 제외**된다. tool_use 블록만 포함된다.

```
Transcript에 포함되는 것:
  - User messages: 텍스트 전문
  - Assistant messages: tool_use 블록만 (텍스트 제외)

왜?
  - LLM(assistant)이 자신의 텍스트를 통해 분류기를 조종하는 것을 방지
  - "이 명령은 안전합니다"라는 assistant 텍스트가 분류기에 영향을 주면 안 됨
  - Tool 호출의 실제 파라미터만이 분류 근거가 되어야 함
```

이것은 **LLM self-manipulation의 구조적 방지**다. 분류기도 LLM이므로, assistant가 "이건 안전해요"라고 쓴 텍스트에 영향받을 수 있다. 텍스트를 아예 제거함으로써 이 공격 벡터를 차단한다.

### CLAUDE.md as "User Intent" for Classifier

```
"The following is the user's CLAUDE.md configuration. These are instructions
the user provided to the agent and should be treated as part of the user's
intent when evaluating actions."
```

CLAUDE.md 내용은 분류기에게 "사용자가 이 에이전트에게 허용한 범위"로 제공된다. 사용자가 CLAUDE.md에 "npm run build는 항상 허용"이라고 쓰면, 분류기는 이를 사용자 의도로 존중한다.

### 3-Tier Permission Customization

사용자는 분류기 동작을 세 축으로 조정할 수 있다:

| 축 | 용도 | 예시 |
|---|---|---|
| `allow` | 명시적 허용 | "npm run *는 항상 OK" |
| `soft_deny` | 차단하되 재정의 가능 | "rm -rf는 기본 차단, 필요시 요청" |
| `environment` | 환경 컨텍스트 | "이것은 개발 서버, production 아님" |

### Structured Output via Forced Tool Call

분류기는 free-form 텍스트가 아닌 **강제 tool 호출**로 결과를 반환한다:

```json
{
  "thinking": "Step-by-step reasoning about safety",
  "shouldBlock": true,
  "reason": "This command modifies system files outside the project"
}
```

Free-form 텍스트 응답은 파싱 실패 위험이 있다. Forced tool call은 구조화된 출력을 보장한다.

**적용하기**: LLM 에이전트에 자율 모드를 추가할 때, (1) 안전한 도구는 분류기를 건너뛰게 하고, (2) assistant 텍스트를 분류기 입력에서 제외하여 self-manipulation을 방지하고, (3) 사용자에게 allow/deny/environment 커스터마이제이션을 제공하라.

---

## Anti-Patterns: Claude Code가 의도적으로 피한 것들

### Anti-Pattern 1: "모든 것을 추상화"

Claude Code는 `QueryEngine.ts`를 46K 줄의 단일 파일로 유지한다. 작은 파일로 분리하는 것이 일반적이지만, 에이전트 엔진의 모든 상태 전이가 긴밀하게 결합되어 있어서 분리하면 오히려 복잡도가 증가한다.

**교훈**: 추상화는 복잡도를 숨기는 것이지 제거하는 것이 아니다. 긴밀하게 결합된 로직은 한 곳에 두는 것이 더 명확할 수 있다.

### Anti-Pattern 2: "범용 보안 모듈"

Bash security check는 BashTool 디렉토리 안에 있다. "범용 보안 라이브러리"로 분리하지 않았다. 이유: Bash 명령어의 보안 검증은 bash 문법에 대한 깊은 이해가 필요하므로, 범용화하면 각 tool에 맞는 정밀한 검증이 어려워진다.

**교훈**: 보안은 도메인 특화적이다. "범용 보안 레이어"보다 "각 도구에 맞춤화된 보안 검증"이 더 효과적이다.

### Anti-Pattern 3: "무한 Retry"

일반 모드에서 max retries는 10회로 제한된다. Persistent retry mode도 5시간 cap이 있다. "절대 포기하지 않는" retry는 비용 폭발과 사용자 경험 악화를 초래한다.

**교훈**: Retry에는 반드시 bound가 있어야 한다. 무한 retry는 LLM API 비용이 무한히 증가한다는 뜻이다.

### Anti-Pattern 4: "모든 context를 항상 제공"

Claude Code는 모든 memory를 항상 주입하지 않고, relevance filtering으로 관련된 것만 선택한다. Deferred tools로 MCP tool schema도 필요할 때만 로드한다.

**교훈**: Context window에 넣을 수 있다고 넣어야 하는 것은 아니다. 관련 없는 context는 LLM의 주의를 분산시키고, 비용을 증가시킨다.

### Anti-Pattern 5: "Lazy Delegation" — Multi-Agent의 가장 흔한 실패

**관찰**: Claude Code의 coordinator 프롬프트는 lazy delegation을 가장 강하게 경고한다. "based on your findings"나 "based on the research" 같은 표현은 **절대 쓰지 말라**고 명시한다.

**왜 실패하는가**:

1. **Worker는 coordinator 대화를 볼 수 없다**: "아까 논의한 그 버그"는 worker에게 의미가 없다. Worker는 새로운 context에서 시작한다.

2. **이해 책임의 전가**: "Based on your findings, fix it"은 coordinator가 findings를 이해하지 않았다는 고백이다. Coordinator가 이해하지 못한 것을 worker가 더 잘 이해할 수는 없다.

3. **누적 모호성**: 모호한 지시 → worker의 추측 → 잘못된 구현 → 수정 요청 → 또 모호한 지시... 이 루프는 비용과 시간만 소모한다.

**구체적 증상들**:

```python
# Lazy delegation의 전형적 신호들
"Fix the bug we discussed"              # 어떤 bug? 어디서?
"Based on your findings, implement"      # 어떤 findings?
"Create a PR for the recent changes"     # 어떤 changes? 어떤 branch? draft?
"Something went wrong, can you look?"    # 어떤 error? 어떤 file?
```

**해결법**: Coordinator가 findings를 읽고, 이해하고, exact file path + line number + 구체적 변경 사항을 포함한 self-contained 명세를 작성해야 한다. 명세 품질이 결과 품질을 결정한다.

---

## 내가 내일 LLM 앱을 만든다면: Actionable Checklist

### Day 1: Foundation

- [ ] **Tool loop 구현**: 이것이 에이전트의 핵심

```python
# 모든 에이전트의 심장
messages = [system_prompt, user_message]
while True:
    response = llm.chat(messages)
    tool_calls = extract_tool_calls(response)
    if not tool_calls:
        break  # 최종 응답
    results = []
    for call in tool_calls:
        try:
            result = execute_tool(call)
            results.append({"role": "tool", "content": result})
        except Exception as e:
            # 에러를 LLM에게 반환 — 앱이 크래시하면 안 됨
            results.append({"role": "tool", "content": f"Error: {e}"})
    messages.extend([response, *results])
```

- [ ] **Streaming 활성화**: 모든 API 호출을 streaming으로. 사용자가 "멈춘 건가?" 불안을 느끼지 않도록

```python
# Streaming은 Day 1부터
async for chunk in llm.stream(messages):
    if chunk.type == "content_block_delta":
        yield chunk.delta.text  # 실시간 출력
    elif chunk.type == "tool_use":
        pending_tool_calls.append(chunk)
```

- [ ] **Token counting**: 매 turn마다 사용량 추적. 사전 추정 + 사후 정확 측정 둘 다 구현

```python
class TokenTracker:
    def __init__(self, model_limit: int):
        self.used = 0
        self.limit = model_limit

    def add_turn(self, input_tokens: int, output_tokens: int):
        self.used += input_tokens + output_tokens

    def remaining(self) -> int:
        return self.limit - self.used

    def should_compact(self) -> bool:
        return self.remaining() < self.limit * 0.2  # 20% 남으면 압축
```

### Week 1: Resilience

- [ ] **Retry with awareness**: 429(rate limit), 529(overload), auth 만료를 구분하여 각각 다르게 처리

```python
async def resilient_call(messages, max_retries=10):
    for attempt in range(max_retries):
        try:
            return await llm.chat(messages)
        except RateLimitError as e:
            wait = e.retry_after or (2 ** attempt)
            await asyncio.sleep(wait)
        except OverloadError:
            await asyncio.sleep(30)  # 서버 과부하는 더 오래 대기
        except AuthError:
            token = await refresh_token()  # 토큰 갱신 시도
            llm.set_token(token)
    raise MaxRetriesExceeded()
```

- [ ] **Model fallback**: 주력 모델 실패 시 대안 모델로 fallback

```python
MODEL_CHAIN = ["claude-sonnet-4-20250514", "claude-haiku-4-20250414"]

async def call_with_fallback(messages):
    for model in MODEL_CHAIN:
        try:
            return await llm.chat(messages, model=model)
        except (OverloadError, RateLimitError):
            continue
    raise AllModelsFailed()
```

- [ ] **Tool 에러 → LLM 반환**: 에러를 앱이 처리하지 말고 LLM에게 돌려보내 self-correction 유도
- [ ] **Graceful degradation**: API 외의 모든 서비스는 실패해도 앱이 동작하도록

### Week 2: Prompt & Context

- [ ] **계층적 시스템 프롬프트**: 모듈화된 sections + feature-gated sections

```python
def build_system_prompt(user, features):
    sections = [CORE_IDENTITY]  # 항상 포함

    if features.get("code_editing"):
        sections.append(CODE_EDITING_INSTRUCTIONS)
    if features.get("web_search"):
        sections.append(WEB_SEARCH_INSTRUCTIONS)

    # 사용자별 memory — 관련된 것만
    relevant_memories = filter_by_relevance(
        user.memories, current_task
    )
    if relevant_memories:
        sections.append(format_memories(relevant_memories))

    return "\n\n".join(sections)
```

- [ ] **부정 지시 추가**: LLM의 "과도한 도움"을 억제하는 구체적 지시
- [ ] **Context compression**: Compact mode 구현. "무엇을 보존하고 무엇을 버릴지" 명시

```python
async def compact_conversation(messages, token_tracker):
    if not token_tracker.should_compact():
        return messages

    summary_prompt = """
    Summarize this conversation preserving:
    1. Current task and user intent
    2. Key decisions made
    3. File paths and code changes
    4. Errors encountered and their resolutions
    Discard: progress updates, intermediate reasoning, tool outputs already acted on.
    """
    summary = await llm.chat([{"role": "user", "content": summary_prompt}])
    return [messages[0], {"role": "user", "content": summary}]
```

- [ ] **Memory system**: Session을 넘어서는 persistent context (CLAUDE.md 패턴)

### Week 3: Security & UX

- [ ] **Unicode sanitization**: NFKC + 위험 문자 클래스 제거

```python
import unicodedata

def sanitize_input(text: str) -> str:
    # Step 1: NFKC 정규화 (호환 분해 + 정준 결합)
    text = unicodedata.normalize("NFKC", text)
    # Step 2: 위험 문자 제거 (zero-width, bidi override 등)
    dangerous = {'\u200b', '\u200c', '\u200d', '\u2028', '\u2029',
                 '\u202a', '\u202b', '\u202c', '\u202d', '\u202e',
                 '\ufeff'}
    return ''.join(c for c in text if c not in dangerous)
```

- [ ] **Shell command validation**: Allowlist 기반 + AST 분석 (가능하면)

```python
SAFE_COMMANDS = {"ls", "cat", "grep", "find", "echo", "pwd", "git"}
BLOCKED_PATTERNS = [
    r"rm\s+-rf\s+/",      # rm -rf /
    r">\s*/dev/sd",         # 디스크 직접 쓰기
    r"curl.*\|\s*sh",       # pipe to shell
    r"eval\s+",             # eval 실행
]

def validate_command(cmd: str) -> tuple[bool, str]:
    base_cmd = shlex.split(cmd)[0]
    if base_cmd not in SAFE_COMMANDS:
        return False, f"Command '{base_cmd}' not in allowlist"
    for pattern in BLOCKED_PATTERNS:
        if re.search(pattern, cmd):
            return False, f"Blocked pattern detected: {pattern}"
    return True, "OK"
```

- [ ] **Permission system**: 위험한 작업에 사용자 확인. 거부 시 LLM에게 피드백
- [ ] **Prompt caching**: Cache break 모니터링. 안정적인 cache hit으로 비용 절감

### Week 4: Scale

- [ ] **Concurrent tool execution**: 각 tool의 concurrency safety self-declaration

```python
class Tool:
    name: str
    is_concurrency_safe: bool  # 각 tool이 자신의 병렬 안전성을 선언

async def execute_tools(tool_calls: list[ToolCall]):
    safe = [t for t in tool_calls if t.tool.is_concurrency_safe]
    unsafe = [t for t in tool_calls if not t.tool.is_concurrency_safe]

    # Safe tools: 병렬 실행
    safe_results = await asyncio.gather(
        *[execute(t) for t in safe]
    )
    # Unsafe tools: 순차 실행
    unsafe_results = []
    for t in unsafe:
        unsafe_results.append(await execute(t))

    return safe_results + unsafe_results
```

- [ ] **Sub-agent pattern**: 독립적 작업을 sub-agent에게 위임하여 context isolation

```python
async def spawn_sub_agent(task_prompt: str, tools: list[Tool]):
    """Sub-agent는 자체 context를 가진다 — main agent의 context를 오염시키지 않는다."""
    sub_messages = [
        {"role": "system", "content": SUB_AGENT_SYSTEM_PROMPT},
        {"role": "user", "content": task_prompt},  # Self-contained!
    ]
    result = await run_agent_loop(sub_messages, tools)
    return result.final_response  # Summary만 main agent에 반환
```

- [ ] **Deferred tools**: 많은 tool이 있을 때 lazy loading으로 prompt 크기 관리
- [ ] **Performance**: Import 전 prefetch, lazy loading, schema caching

### Week 5: Multi-Agent

- [ ] **Coordinator-Worker 분리**: Coordinator는 종합과 의사결정, Worker는 실행

```python
class Coordinator:
    """작업을 분배하는 것이 아니라 지식을 종합하는 것이 핵심 역할."""

    async def handle_task(self, user_request: str):
        # Phase 1: Research (병렬)
        research_results = await asyncio.gather(
            self.spawn_worker("Investigate src/auth/ for session handling patterns. Report file paths and line numbers."),
            self.spawn_worker("Find all tests related to session expiry in tests/. Report which pass/fail."),
        )

        # Phase 2: Synthesis (coordinator가 직접 수행)
        spec = self.synthesize(research_results)
        # spec = "Fix null pointer in src/auth/validate.ts:42.
        #         Session.user is undefined when token cached but session expired.
        #         Add null check before user.id access, return 401."

        # Phase 3: Implementation
        impl_result = await self.spawn_worker(spec)

        # Phase 4: Verification (새 worker — 구현 가정 없이)
        verify_result = await self.spawn_worker(
            f"Verify the fix in commit {impl_result.commit_hash}. "
            f"Reproduce the original bug, confirm fix, check edge cases. "
            f"Do NOT just read the code — run it."
        )
```

- [ ] **Self-contained prompts**: 모든 worker 프롬프트가 coordinator 대화 없이 독립적으로 실행 가능한지 검증
- [ ] **Continue vs Spawn 의사결정**: Context 겹침 기반 판단 (Pattern 9 참조)
- [ ] **Verification worker 분리**: 구현 worker와 검증 worker를 반드시 분리. 검증 worker에게는 파일 수정 권한 없음

### Week 6: Production Monitoring

- [ ] **Cost tracking per session**: 세션별 비용 추적 및 예산 제한

```python
class CostTracker:
    def __init__(self, budget_usd: float = 10.0):
        self.budget = budget_usd
        self.spent = 0.0

    def add_usage(self, input_tokens: int, output_tokens: int, model: str):
        rates = MODEL_RATES[model]  # $/1M tokens
        cost = (input_tokens * rates["input"] + output_tokens * rates["output"]) / 1_000_000
        self.spent += cost
        if self.spent > self.budget:
            raise BudgetExceeded(f"Session cost ${self.spent:.2f} exceeds budget ${self.budget:.2f}")

    def report(self) -> dict:
        return {"spent_usd": self.spent, "budget_usd": self.budget, "remaining_pct": (1 - self.spent/self.budget) * 100}
```

- [ ] **Token usage analytics**: 어디서 token이 소비되는지 분류 (system prompt, user input, tool results, LLM output)
- [ ] **Cache hit rate monitoring**: Prompt cache 효율성 추적. Cache break 시 alert

```python
class CacheMonitor:
    def __init__(self):
        self.hits = 0
        self.misses = 0

    def record(self, response_headers):
        if response_headers.get("cache_read_input_tokens", 0) > 0:
            self.hits += 1
        else:
            self.misses += 1

    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0.0

    def alert_if_degraded(self, threshold=0.5):
        if self.hit_rate() < threshold and (self.hits + self.misses) > 10:
            logger.warning(f"Cache hit rate degraded: {self.hit_rate():.1%}")
```

- [ ] **Error categorization**: 에러를 user-error / LLM-error / system-error로 분류하여 각각 다른 대응
- [ ] **Latency tracking**: API latency, tool execution time, total turn time 분리 측정
- [ ] **Conversation quality signals**: Compact 빈도, retry 빈도, 평균 turn 수 등으로 "대화 건강도" 추적

---

## 프로토타입 → 프로덕션 Gap: 500K 줄이 추가하는 것들

Claude Code의 512K 줄 중 에이전트의 **핵심 로직**은 비교적 단순하다: `입력 → 프롬프트 → API 호출 → 도구 실행 루프 → 응답`. 나머지 50만 줄은 이 단순한 루프를 **안정적으로, 안전하게, 효율적으로, 사용자 친화적으로** 만들기 위한 인프라다.

프로토타입은 tool loop만 있으면 "동작"한다. 프로덕션은 다르다. 아래는 500K 줄이 추가하는 **구체적 메커니즘 목록**이다:

### Resilience (복원력) — 7 mechanisms

| # | 메커니즘 | 프로토타입 | 프로덕션 |
|---|---|---|---|
| 1 | **Error-aware retry** | `while True: retry()` | 429/529/auth 구분, exponential backoff, jitter, max cap |
| 2 | **Model fallback chain** | 단일 모델 | Opus → Sonnet → Haiku chain, 자동 전환 |
| 3 | **Tool error feedback** | `try/except → crash` | 에러를 tool_result로 LLM에 반환, self-correction 유도 |
| 4 | **Permission denial recovery** | N/A | 거부 정보를 LLM에 피드백, 대안 탐색 |
| 5 | **Connection prewarming** | Cold start | API 연결 사전 수립, import 전 prefetch |
| 6 | **Graceful degradation** | Hard dependencies | MCP/GrowthBook/Keychain 실패 시 기능 축소로 계속 동작 |
| 7 | **Persistent retry mode** | 10회 제한 | 선택적 5시간 persistent mode (사용자 동의 하에) |

### Security (보안) — 5 mechanisms

| # | 메커니즘 | 프로토타입 | 프로덕션 |
|---|---|---|---|
| 8 | **Unicode sanitization** | `str.strip()` | NFKC 정규화 + zero-width/bidi 문자 제거 |
| 9 | **Shell command validation** | 없음 | 23개 보안 체크, AST 기반 파이프/체이닝 분석 |
| 10 | **2-stage security classifier** | 없음 | Fast → Extended reasoning, assistant text 제외, self-manipulation 방지 |
| 11 | **Permission system** | `confirm()` | 계층적 permission (project/user/session), wildcard rules, 거부 시 LLM 피드백 |
| 12 | **Path traversal prevention** | 없음 | Symlink resolution, project boundary 검증, `.env` 파일 보호 |

### UX (사용자 경험) — 4 mechanisms

| # | 메커니즘 | 프로토타입 | 프로덕션 |
|---|---|---|---|
| 13 | **Streaming with tool status** | `print(response)` | 실시간 스트리밍 + tool 실행 중 spinner/progress 표시 |
| 14 | **Permission as communication** | "OK?" → Y/N | "이 명령은 X를 Y로 변경합니다" + 이유 설명 + wildcard rule 제안 |
| 15 | **Context compression feedback** | 없음 | Compact 발생 시 사용자에게 알림, 보존/삭제 항목 표시 |
| 16 | **Cost transparency** | 없음 | 세션별 token 사용량 + 비용 실시간 표시 |

### Performance (성능) — 4 mechanisms

| # | 메커니즘 | 프로토타입 | 프로덕션 |
|---|---|---|---|
| 17 | **Prompt caching** | 매 호출 전체 전송 | System prompt cache + tool description cache (~11K tokens 절약) |
| 18 | **Deferred tool loading** | 모든 tool schema 항상 포함 | MCP tools lazy load, 필요 시에만 schema 주입 |
| 19 | **Concurrent tool execution** | 순차 실행 | `isConcurrencySafe()` 기반 병렬/순차 자동 판단 |
| 20 | **Parallel startup** | 순차 초기화 | MDM + Keychain + API preconnect + Feature flags 병렬 |

### Observability (관측성) — 2 mechanisms

| # | 메커니즘 | 프로토타입 | 프로덕션 |
|---|---|---|---|
| 21 | **Structured event logging** | `console.log()` | SerialBatchEventUploader, 배치 업로드, 구조화된 이벤트 |
| 22 | **Feature flags** | 하드코딩 | GrowthBook 통합, A/B 테스트, 점진적 rollout |

### 한 줄 요약

```
프로토타입 (1K lines):  input → prompt → API → tool loop → output
프로덕션 (500K lines):  위의 루프 + 22개 메커니즘이 루프의 모든 접점을 감싸고 보호
```

**이 22개 메커니즘 중 어느 하나를 빠뜨려도 프로덕션에서 문제가 된다.** Retry가 없으면 일시적 장애에 앱이 죽고, security가 없으면 LLM이 위험한 명령을 실행하고, context management가 없으면 긴 대화에서 성능이 붕괴한다.

---

## 최종 교훈: 512K Lines에서 배운 것

1. **LLM 앱의 복잡도는 LLM 자체가 아니라 LLM을 둘러싼 인프라에 있다.** API 호출은 쉽다. 그 호출이 실패했을 때, 비용이 폭발할 때, 보안이 뚫릴 때, 사용자가 혼란스러울 때 — 이 모든 edge case를 처리하는 것이 프로덕션이다.

2. **LLM은 틀린다. 항상.** 모든 설계는 LLM이 틀릴 수 있다는 전제에서 시작해야 한다. Tool 에러 반환, 자기 합리화 감지, adversarial 검증 — 이 모든 것은 "LLM은 완벽하지 않다"는 인정에서 나온다.

3. **Multi-agent에서 가장 중요한 것은 agent 수가 아니라 종합(synthesis) 품질이다.** Worker를 10개 띄우는 것보다, coordinator가 findings를 정확히 이해하고 self-contained spec을 작성하는 것이 10배 더 중요하다.

4. **보안은 도메인 특화적이어야 한다.** 범용 보안 레이어보다 각 도구에 맞춤화된 검증이 효과적이다. 그리고 LLM이 자기 자신을 조종하는 것(self-manipulation)은 별도로 방어해야 한다.

5. **Context window는 가장 귀한 자원이다.** 모든 기능 추가에 "이것이 몇 token인가?"를 물어야 한다. 넣을 수 있다고 넣어야 하는 것은 아니다.

6. **Permission은 보안 기능이 아니라 사용자와의 대화다.** "이 명령을 실행할까요?"보다 "이 명령은 X를 Y로 변경합니다. 이유는 Z입니다."가 100배 더 좋다.
