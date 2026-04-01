<!--
tags: llm-application/extended-thinking, context-management/thinking-budget, api-layer/beta-headers
keywords: extended-thinking, thinking-budget, interleaved-thinking, thinking-redaction, thinking-signature, chain-of-thought, output-budget, ThinkingConfig
related_files: 02_query_engine_core.md, 05_context_management.md, 06_api_layer_providers.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 20. Extended Thinking & Chain-of-Thought Management

> Claude Code가 모델의 추론 품질을 극대화하는 방법: thinking budget 관리, interleaved thinking, thinking redaction, 그리고 비용-품질 트레이드오프.

---

## Extended Thinking이란

Extended thinking은 Claude의 내부 추론 과정(chain-of-thought)을 명시적으로 활성화하는 기능이다. 일반 응답에서는 모델이 곧바로 결론을 출력하지만, extended thinking을 켜면 모델이 답변 전에 충분히 "생각"할 수 있는 공간을 확보한다.

- **일반 응답보다 더 깊은 사고가 가능하다.** 수학 문제 풀이, 복잡한 코드 분석, 다단계 계획 수립 등에서 품질이 크게 향상된다.
- **API에서 beta header로 활성화한다.** 표준 API endpoint가 아닌, beta feature flag를 통해 접근한다.
- **비용과 직결된다.** Thinking tokens은 output token 가격으로 과금되므로, 무제한 사용은 비현실적이다.

Claude Code는 이 기능을 단순히 "켜는" 수준이 아니라, context 상황에 따라 thinking budget을 동적으로 관리하는 정교한 전략을 구현한다.


---

## ThinkingConfig 구조

Extended thinking의 활성화 여부와 규모는 `ThinkingConfig` 객체로 제어한다.

```typescript
// thinking 활성화 설정
{
  type: "enabled",
  budget_tokens: number  // thinking에 할당할 최대 토큰
}
```

핵심 파라미터:

- **`budget_tokens`**: Thinking에 사용할 최대 토큰 수. 이 값이 클수록 모델이 더 오래 생각할 수 있지만, 비용과 latency가 증가한다.
- **`output_tokens`와는 별도로 관리한다.** Thinking budget과 output budget은 독립적이지만, 전체 context window 안에서 공존해야 하므로 상호 영향을 준다. Thinking이 output을 잠식하지 않도록 설계해야 한다.
- **Claude Code는 context 상황에 따라 동적으로 budget을 조정한다.** 대화 초반(context 여유)에는 넉넉하게, context 압박이 심할 때는 축소한다.


---

## Beta Headers

Extended thinking 관련 기능은 beta header를 통해 활성화된다. 실제 telemetry에서 관찰된 beta headers는 다음과 같다:

```
betas: "interleaved-thinking-2025-05-14,redact-thinking-2026-02-12,context-management-2025-06-27,prompt-caching-scope-2026-01-05"
```

각 header의 역할을 살펴본다.

### interleaved-thinking-2025-05-14

기존 extended thinking은 **single-shot** 방식이었다:

1. Thinking (한 번) → Tool calls → 결과 반환

Interleaved thinking은 이를 확장한다:

1. Thinking → Tool call → Tool result → **다시 Thinking** → Tool call → ...

이 차이는 결정적이다. Multi-step 추론에서 각 단계별로 tool result를 반영하여 재사고할 수 있기 때문에, 단일 thinking보다 훨씬 정확한 의사결정이 가능하다. 예를 들어, 파일을 읽은 후 그 내용을 기반으로 다음 행동을 생각하는 것과, 파일을 읽기 전에 모든 계획을 세우는 것은 품질 차이가 크다.

### redact-thinking-2026-02-12

Thinking 내용을 API 응답에서 제거(redact)하는 기능이다.

- Thinking 텍스트 대신 **signature 필드만 남긴다.**
- **보안 목적**: Thinking에는 민감한 추론 과정이 포함될 수 있다. 내부적으로 어떤 가정을 했는지, 어떤 코드 패턴을 분석했는지가 노출되면 보안 위험이 될 수 있다.
- **토큰 절약**: Redacted thinking은 다음 턴에서 원문 대신 signature만 전송하므로, context 공간을 절약한다.


---

## Thinking Block의 구조

### API 응답에서의 thinking block

API가 반환하는 thinking block의 형태:

```json
{
  "type": "thinking",
  "thinking": "Let me analyze this code...\n\nThe function has a bug because...",
  "signature": "Ev8ECkYICx..."
}
```

- `type`: `"thinking"`으로 일반 text block과 구분된다.
- `thinking`: 모델의 실제 추론 텍스트. Redaction이 활성화되면 이 필드가 비어 있다.
- `signature`: Redacted thinking의 무결성 검증용. 변조 방지 역할을 한다.

### JSONL 세션에서의 저장

Claude Code가 세션을 JSONL로 저장할 때, thinking block은 다음과 같이 기록된다:

```json
{
  "message": {
    "content": [
      {
        "type": "thinking",
        "thinking": "...",
        "signature": "Ev8ECkYICx..."
      },
      {
        "type": "tool_use",
        "name": "Read",
        "input": {"file_path": "/src/main.ts"}
      }
    ]
  }
}
```

주목할 점:

- **Thinking과 tool_use가 같은 content 배열에 공존한다.** 하나의 assistant 메시지 안에 여러 content block이 들어간다.
- **Interleaved thinking 패턴**: `thinking → tool_use → thinking → tool_use`가 하나의 content 배열 안에서 반복된다. 각 thinking block은 직전 tool result를 반영한 새로운 추론이다.


---

## Thinking Budget vs Output Budget

### 문제: Budget 경쟁

Context window는 유한하다. 그 안에서 여러 요소가 공간을 차지한다:

```
Context Window (200K tokens)
├── System Prompt (~11K)
├── Conversation History (변동)
├── Tool Results (변동, 예측 불가)
├── Thinking Budget (할당)        ← 여기서 경쟁
└── Output Budget (할당)          ← 여기서 경쟁
```

Thinking budget과 output budget은 "남은 공간"을 나눠 가져야 한다. Thinking에 너무 많이 할당하면 output이 부족하고, 반대로 thinking을 줄이면 추론 품질이 떨어진다.

### Claude Code의 전략

Claude Code는 고정 budget이 아닌, 상황 기반 동적 조정 전략을 사용한다:

1. **Context 여유 → Thinking budget 증가**: 대화 초반, context가 넉넉할 때는 모델이 충분히 생각할 수 있도록 높은 thinking budget을 할당한다.
2. **Context 부족 → Thinking budget 축소**: Context 사용률이 높아지면 thinking을 줄여서 output token 공간을 확보한다. 응답을 못 하는 것보다 생각을 덜 하는 게 낫다.
3. **Compact 후 → Budget 재조정**: Context compaction이 발생하면 freed tokens을 thinking에 재배분한다.
4. **Tool result 크기 → 동적 조정**: 대형 tool result(예: 수천 줄 파일 읽기)가 들어온 후에는 thinking budget을 축소하여 전체 token budget을 관리한다.

### 비용 구조

Thinking tokens의 가격은 output tokens과 동일하다:

- **Opus에서 thinking: $75 / 1M tokens** (output 가격과 동일)
- Thinking 1,000 tokens = 일반 output 1,000 tokens과 동일 비용
- 따라서 **thinking budget은 직접적으로 비용에 영향을 미친다.** Budget 설계 없이 무제한 thinking을 허용하면 비용이 급증한다.

이것이 Claude Code가 동적 budget 관리를 구현한 핵심 이유다. "더 잘 생각하게 하자"는 단순한 접근이 아니라, "이 상황에서 얼마나 생각하는 게 비용 대비 최적인가"를 계산한다.


---

## Thinking의 UI 처리

### 터미널에서의 표시

Claude Code의 터미널 UI는 thinking을 다음과 같이 처리한다:

- **Thinking 진행 중**: 스피너 또는 로딩 인디케이터를 표시하여 모델이 "생각 중"임을 사용자에게 알린다.
- **Thinking 완료**: 접을 수 있는(collapsible) 블록으로 표시한다. 기본적으로 접혀 있어 UI를 깔끔하게 유지하면서, 필요시 펼쳐서 모델의 추론 과정을 확인할 수 있다.
- **Redacted thinking**: `"[Thinking redacted]"` 또는 아예 표시하지 않는다.

### Streaming에서의 처리

Streaming SSE에서 thinking block은 다음 순서로 전달된다:

```
content_block_start (type: "thinking")
  → content_block_delta (thinking text 스트리밍)
  → content_block_stop
content_block_start (type: "text" 또는 "tool_use")
  → ...
```

- **Thinking block이 먼저 스트리밍되고, 이후 실제 응답이나 tool call이 온다.** 이 순서는 보장된다.
- Interleaved thinking에서는 `thinking → tool_use → thinking → text` 같은 복합 패턴이 발생하며, 각 block은 `content_block_start`/`content_block_stop`으로 구분된다.


---

## LLM 엔지니어링 인사이트

### 1. Thinking은 "공짜"가 아니다

Output 토큰과 같은 가격이다. Budget 설계가 비용에 직접 영향을 미친다. "많이 생각하면 좋지 않을까?"라는 접근은 비용 폭탄으로 이어진다. 상황별로 적절한 thinking budget을 배분하는 것이 엔지니어링의 핵심이다.

### 2. Interleaved > Single-shot

Multi-step 작업에서 각 단계별 재사고가 훨씬 정확하다. 단일 thinking에서 모든 계획을 세우는 것보다, tool result를 받은 후 재사고하는 것이 더 나은 결과를 낳는다. 이는 인간이 "일단 해보고 판단한다"는 것과 같은 원리다.

### 3. Budget 동적 조정이 핵심이다

고정 budget은 두 가지 문제 중 하나를 야기한다:
- Budget이 너무 크면 → Context overflow 위험
- Budget이 너무 작으면 → Thinking 부족으로 품질 저하

Claude Code처럼 context 상황에 따라 조정해야 한다. 이것은 단순한 최적화가 아니라, 시스템 안정성의 문제다.

### 4. Redaction의 이중 목적

- **보안**: 민감한 추론 과정이 외부에 노출되지 않도록 한다.
- **성능**: Redacted thinking은 다음 턴에서 원문 대신 signature만 전송하므로 context 공간을 절약하고, prompt caching에서 cache hit 가능성을 높인다.

### 5. Signature 필드

Redacted thinking의 무결성을 검증한다. 클라이언트가 thinking을 임의로 변조하거나 삽입하는 것을 방지한다. API 보안의 한 계층이다.

### 6. Thinking은 디버깅 도구이기도 하다

개발 중에는 thinking을 보면서 모델의 추론 과정을 이해하고, 프롬프트가 의도대로 작동하는지 검증한다. 프로덕션에서는 redact하여 보안과 비용을 관리한다.


---

## 적용하기

| 상황 | 권장 설정 |
|------|----------|
| 복잡한 코드 분석 | Thinking 활성화, budget 높게 |
| 단순 파일 읽기/수정 | Thinking 비활성 또는 최소 budget |
| Multi-step 에이전트 | Interleaved thinking 필수 |
| 비용 민감한 환경 | Budget cap 설정 + 동적 조정 |
| 보안 민감 환경 | Redaction 활성화 |

핵심 원칙: Extended thinking은 "켜고 끄는" binary 선택이 아니라, context와 비용과 품질 사이에서 **동적으로 최적점을 찾는** continuous 조정이다. Claude Code의 구현이 이 원칙을 잘 보여준다.

---

## 대안 비교

| 접근 방식 | 추론 품질 | 비용 | Latency | 구현 복잡도 | 적합한 상황 |
|----------|---------|------|---------|-----------|-----------|
| **Extended Thinking (budget 기반)** | 최고 | 높음 (output 가격) | 높음 | 중간 (budget 관리 필요) | 복잡한 코드 분석, 아키텍처 설계, 수학적 추론 |
| **Chain-of-Thought Prompting** | 중상 | 낮음 (output만) | 중간 | 낮음 ("step by step" 추가) | 단순한 multi-step 추론, 비용 민감 환경 |
| **Multi-step Reasoning (tool loop)** | 중상 | 중간 (여러 API 호출) | 높음 | 중간 | Tool 결과에 의존하는 반복적 판단 |
| **별도 Planning Call** | 상 | 높음 (2회 호출) | 높음 | 높음 | 계획과 실행을 완전히 분리해야 할 때 |
| **Thinking 없이 직접 응답** | 중하 | 최저 | 최저 | 최저 | 단순 질의, 코드 포매팅, 반복 작업 |

**핵심 트레이드오프**: Extended thinking은 "더 깊이 생각하는 비용"을 명시적으로 제어할 수 있다는 것이 최대 장점이다. CoT prompting은 무료지만 thinking 깊이를 제어할 수 없고, 별도 planning call은 비용이 2배지만 계획과 실행의 완전한 분리가 가능하다.

---

## 내 프로젝트에 적용하기

### Step 1: Thinking 활성화 및 Budget 설정

API 호출 시 `thinking` 파라미터를 추가한다:
```json
{ "type": "enabled", "budget_tokens": 10000 }
```
초기 budget은 **10,000 tokens**으로 시작하여 작업 유형별로 조정한다. 복잡한 아키텍처 분석은 20,000+, 단순 코드 수정은 5,000 이하. Beta header(`interleaved-thinking`)를 포함하여 multi-step 작업에서 각 tool result 후 재사고가 가능하도록 한다.

### Step 2: Context 사용률 기반 동적 Budget 조정

고정 budget은 context overflow 또는 thinking 부족을 야기한다. 다음 공식으로 동적 조정한다:
```
remaining = context_window - system_prompt - conversation_history - output_reserve
thinking_budget = min(max_thinking, remaining * 0.4)
```
- Context 사용률 < 50%: thinking budget을 넉넉하게 (remaining의 40%)
- Context 사용률 > 75%: thinking budget을 축소 (remaining의 20%)
- Compact 발생 후: freed tokens의 일부를 thinking에 재배분

이 조정은 매 API 호출 전에 자동으로 수행되어야 한다.

### Step 3: 비용 모니터링 설정

Thinking tokens은 output 가격($75/1M for Opus)으로 과금되므로 비용 추적이 필수다:
- 매 응답의 `usage` 필드에서 thinking tokens를 별도 집계
- 일별/세션별 thinking 비용 = `thinking_tokens × output_price`
- **Alert 기준**: 세션당 thinking 비용이 전체 비용의 40%를 초과하면 budget cap 재검토
- Redaction 활성화로 다음 턴에서 thinking 원문 대신 signature만 전송하여 context 절약 + cache hit 향상
