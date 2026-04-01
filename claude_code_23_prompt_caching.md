<!--
tags: api-layer/prompt-caching, performance/cost-optimization, context-management/cache-strategy
keywords: prompt-caching, cache-control, ephemeral, cache-break, cache-hit-rate, cache-aware-ordering, latched-states, cost-optimization, breakpoint-placement
related_files: 05_context_management.md, 06_api_layer_providers.md, 16_performance_optimization.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 23. Prompt Caching Deep-Dive

> Prompt caching의 메커니즘, cache-aware 메시지 순서 설계, breakpoint 배치 전략, 비용 수학, 그리고 99%+ cache hit rate를 달성하는 실전 기법.

---

## Prompt Caching이란

- API 호출 시 이전 요청과 동일한 prefix를 재사용하는 기능
- 동일한 prefix → 재계산 없이 캐시에서 로드 → 비용/지연시간 절감
- Anthropic API의 `cache_control` 파라미터로 제어

---

## 비용 구조: 왜 중요한가

| Token Type | Cost (Opus 기준) | 일반 대비 |
|------------|------------------|----------|
| Input (일반) | $15/1M | 100% |
| Cache creation | $18.75/1M | 125% |
| Cache read | $1.50/1M | 10% |
| Output | $75/1M | - |

**핵심 수학**:
- Cache read는 일반 input의 **10%** 비용
- Cache creation은 **125%** (처음 한 번만)
- 두 번째 요청부터 cache hit → 90% 절감
- Break-even: 2회 이상 호출 시 이득

**실제 관찰 데이터** (telemetry):
```json
{
  "cache_hits": 12242,
  "cache_misses": 8,
  "hit_rate": 0.9993  // 99.93%
}
```

---

## Cache Control 메커니즘

### cache_control 파라미터
```json
{
  "type": "text",
  "text": "System prompt content here...",
  "cache_control": { "type": "ephemeral" }
}
```

### Cache Scope Options
| Scope | TTL | 용도 |
|-------|-----|------|
| `ephemeral` (기본) | 5분 | 일반 대화 |
| `ephemeral_1h` | 1시간 | 긴 세션 |
| Global | Organization-wide | 팀 공유 prefix |

- Beta header: `prompt-caching-scope-2026-01-05`로 scope 선택

### 실제 JSONL에서의 cache 토큰
```json
{
  "usage": {
    "input_tokens": 15234,
    "cache_creation_input_tokens": 8000,
    "cache_read_input_tokens": 3000,
    "cache_creation": {
      "ephemeral_5m_input_tokens": 8000,
      "ephemeral_1h_input_tokens": 0
    }
  }
}
```

---

## Cache-Aware 메시지 순서 설계

### 핵심 원칙: "가장 안정적인 것을 먼저"

```
API 요청 구조 (위에서 아래로):
┌────────────────────────────────┐
│ System Prompt (하드코딩)        │ ← 가장 안정 (변하지 않음)
│ cache_control: ephemeral       │
├────────────────────────────────┤
│ Tool Schemas (~11K tokens)     │ ← 세션 중 거의 불변
│ cache_control: ephemeral       │
├────────────────────────────────┤
│ CLAUDE.md content              │ ← 세션 중 불변
├────────────────────────────────┤
│ Memory content                 │ ← 세션 중 거의 불변
├────────────────────────────────┤
│ Conversation History           │ ← 매 턴마다 변경
│ (이전 턴까지는 캐시 가능)       │
├────────────────────────────────┤
│ Current user message           │ ← 항상 새로움
└────────────────────────────────┘
```

- 위쪽이 안정적 → 캐시 prefix가 길어짐 → 더 많은 절약
- Conversation history도 이전 턴까지는 캐시 가능 (append-only)

### 왜 이 순서가 최적인가
```
Turn 1: [System + Tools + CLAUDE.md + Memory + User1] → cache creation
Turn 2: [System + Tools + CLAUDE.md + Memory + User1 + Asst1 + User2]
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
         이 부분은 Turn 1과 동일 → cache hit!
```
- 매 턴마다 이전 대화까지의 prefix가 캐시됨
- 새로 계산하는 건 마지막 user message + 약간의 context만

---

## Cache Break: 언제 캐시가 깨지는가

### Cache Break 원인
1. **System prompt 변경** — Tool 추가/제거, feature flag 변경
2. **Tool schema 변경** — MCP server 연결/해제
3. **Beta headers 변경** — 기능 활성화/비활성화
4. **Model 변경** — Opus → Sonnet fallback
5. **Cache scope/TTL 변경**
6. **CLAUDE.md 편집** — 프로젝트 설정 변경

### Cache Break Detection
Claude Code는 이전 요청과 현재 요청의 hash를 비교:
```
hash = hash(system_prompt + tool_schemas + beta_headers + model + cache_scope)
if (hash !== previous_hash) → cache break detected → 진단 로깅
```

---

## Latched States: Cache 보호 메커니즘

```
한 번 활성화되면 세션 중 비활성화하지 않는 상태들:
- Auto mode: ON → 계속 ON (OFF로 돌아가면 cache break)
- Overage state: detected → 계속 detected
- Cache editing mode: ON → 계속 ON
```

**왜 latch하는가?**
```
일반적 접근:
Turn 5: auto_mode=true  → system prompt에 auto mode 지시 추가
Turn 6: auto_mode=false → system prompt에서 auto mode 지시 제거 → CACHE BREAK!
Turn 7: auto_mode=true  → 다시 추가 → CACHE BREAK!

Latched 접근:
Turn 5: auto_mode=true → 추가 → cache break (1회)
Turn 6-∞: 계속 true → cache hit
```
- 상태 변경 횟수를 최소화하여 cache break 방지
- trade-off: 약간의 불필요한 prompt tokens vs 대규모 cache 절약

---

## 99%+ Hit Rate를 달성하는 전략

1. **Stable prefix first**: System prompt, tools, CLAUDE.md를 앞에
2. **Latched states**: 한 번 변경된 상태는 되돌리지 않음
3. **Tool description caching**: ~11K tokens의 tool descriptions를 매번 재생성하지 않고 캐시
4. **Deferred tool loading**: MCP tool schema를 처음부터 로드하지 않음 → system prompt 크기 안정화
5. **Compact의 cache 인식**: 압축 후에도 system prompt prefix는 보존
6. **DANGEROUS_uncachedSystemPromptSection**: 이 함수를 최소한으로 사용 (cache를 깨뜨리므로)
7. **Feature flag 안정화**: GrowthBook flag를 세션 시작 시 한 번만 평가, 이후 고정

---

## LLM 엔지니어링 인사이트

1. **Prompt caching은 cost optimization의 80%**: 다른 최적화보다 ROI가 압도적
2. **순서가 곧 돈**: 동일한 내용이라도 순서를 바꾸면 cache hit → miss
3. **Latched state는 반직관적이지만 경제적**: "정확한 상태 반영"보다 "cache 유지"가 비용 면에서 우월
4. **Break-even은 2회**: cache creation의 125% 비용은 2회차에 회수
5. **Monitoring 필수**: cache hit/miss ratio를 추적하지 않으면 최적화 불가능
6. **CLAUDE.md 편집 = cache break**: 프로젝트 설정 변경의 숨겨진 비용

---

## 적용하기

| 상황 | 전략 |
|------|------|
| System prompt 설계 | 안정적 부분을 앞에, 변동 부분을 뒤에 |
| Feature flags | 세션 시작 시 한 번 평가, latch |
| MCP tools | Deferred loading으로 prefix 안정화 |
| CLAUDE.md | 자주 수정하지 않음, 안정적 내용만 |
| 모니터링 | cache_hits / (cache_hits + cache_misses) 추적 |

---

## 내 프로젝트에 적용하기

### Step 1: Prompt 섹션을 안정성 순서로 배치

Prompt caching의 핵심은 **"가장 안정적인 것을 먼저"** 배치하는 것이다. API 요청 구조를 다음 순서로 설계한다:
```
1. System prompt (하드코딩, 거의 불변)        ← cache prefix 시작
2. Tool schemas (~11K tokens, 세션 중 불변)
3. 프로젝트 config (CLAUDE.md, 세션 중 불변)
4. Memory / 사용자 설정 (거의 불변)
5. Conversation history (append-only)        ← 이전 턴까지 cache 가능
6. Current user message (항상 새로움)         ← cache 불가
```
위쪽 섹션이 안정적일수록 cache prefix가 길어지고, 더 많은 토큰이 cache read(10% 비용)로 처리된다. 순서를 바꾸면 동일한 내용이라도 cache miss가 발생한다.

### Step 2: cache_control 설정 추가

System prompt와 tool schema 끝에 `cache_control` breakpoint를 배치한다:
```json
{ "type": "text", "text": "System prompt...", "cache_control": { "type": "ephemeral" } }
```
- **짧은 세션** (5분 이내): `ephemeral` (5분 TTL) — 기본값, 대부분의 대화에 적합
- **긴 세션** (코드 리뷰, 분석): `ephemeral_1h` (1시간 TTL) — beta header 필요
- **Latched state 패턴 적용**: auto mode, feature flag 등 한 번 활성화된 상태는 세션 중 되돌리지 않는다. 상태 토글이 cache break의 주요 원인이므로, 한 번 변경하면 latch하여 cache를 보호한다.

### Step 3: Cache Hit Rate 모니터링

최적화 없이는 최적화할 수 없다. 매 API 응답의 `usage` 필드에서 cache 관련 metrics를 추적한다:
```
hit_rate = cache_read_input_tokens / (cache_read_input_tokens + cache_creation_input_tokens + input_tokens)
```
- **목표: 95%+ hit rate**. Claude Code는 99.93%를 달성한다
- **Hit rate 급락 시 진단**: system prompt 변경, tool schema 변경, CLAUDE.md 편집, model 변경, beta header 변경이 주요 원인
- **비용 절감 계산**: `saved = (cache_read_tokens × 0.9 × input_price)`. Cache read는 일반 input의 10%이므로, hit되는 모든 토큰에서 90% 절감
- 일별 리포트로 cache 효율과 비용 추이를 추적하여, cache break를 유발하는 패턴을 식별하고 제거한다
