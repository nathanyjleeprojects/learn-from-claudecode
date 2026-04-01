<!--
tags: architecture/retry-strategy, architecture/model-fallback, architecture/rate-limit-handling, agents/resilience, llm-application/error-handling
keywords: retry-logic, exponential-backoff, model-fallback, rate-limit, overload-detection, persistent-retry, withRetry, 429-error, 529-error, unattended-mode, heartbeat, fast-mode-fallback
related_files: 02_query_engine_core.md, 06_api_layer_providers.md, 08_streaming_concurrency.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 07. Retry & Resilience

> LLM API의 독특한 failure mode에 대응하는 Claude Code의 retry 전략: `withRetry.ts` (824 lines) 심층 분석. Exponential backoff, model fallback, persistent retry, rate limit handling. 이 장은 **LLM harness를 직접 설계하는 엔지니어**를 위한 실전 레퍼런스다.

## Keywords
retry-logic, exponential-backoff, model-fallback, rate-limit-handling, 429-error, 529-error, overload-detection, persistent-retry, unattended-mode, heartbeat, fast-mode-fallback, connection-error, token-refresh, withRetry

---

## LLM API Failure는 일반 API Failure와 다르다

일반 REST API의 retry는 비교적 단순하다: 500 에러가 나면 재시도한다. 하지만 LLM API는 독특한 failure mode들이 있다:

1. **Rate limit (429)**: Token budget 소진 -- 시간이 지나면 자동 해결
2. **Overload (529)**: 서버 과부하 -- 일시적이지만 지속 시간 예측 불가
3. **Context overflow**: 요청 자체가 너무 커서 실패 -- 재시도가 아닌 context 축소 필요
4. **Streaming 중단**: 응답 도중 connection 끊김 -- 부분 응답 recovery 필요
5. **Auth 만료**: OAuth token 만료 -- token refresh 후 재시도
6. **Model 불가**: 특정 모델이 일시적으로 사용 불가 -- 다른 모델로 fallback

Claude Code의 `src/services/api/withRetry.ts` (824 lines)는 이 모든 경우를 처리한다.

### 대안 비교: Simple Exponential Backoff vs LLM-Aware Retry

일반적인 HTTP retry 라이브러리(axios-retry, got 등)가 LLM API에 왜 부족한지 구체적 시나리오로 비교한다.

| 시나리오 | Simple Exponential Backoff | LLM-Aware Retry (Claude Code) |
|---------|---------------------------|-------------------------------|
| **429 rate limit, retry-after: 5s** | Backoff 계산으로 대기 (실제 대기 시간 불일치) | `retry-after` header를 정확히 파싱해서 5초 대기 |
| **529 on background summarizer** | 동일하게 retry -- cascade 증폭 | Background source 감지, **즉시 bail** -- cascade 차단 |
| **529 on foreground, 3회 연속** | 계속 같은 모델로 retry | Model fallback 트리거 (Opus -> Sonnet) |
| **Context overflow (400)** | 동일 요청으로 재시도 -- 영원히 실패 | Error message 파싱 -> `max_tokens` 자동 조정 후 재시도 |
| **401 OAuth expired** | 실패 반환 또는 무의미한 재시도 | Token refresh -> 새 client 생성 -> 재시도 |
| **Fast mode 429, retry-after: 60s** | 60초 대기 후 같은 모드로 retry | Standard mode로 fallback, 30분 cooldown 설정 |
| **ECONNRESET** | 같은 connection pool로 재시도 -- 반복 실패 | Keep-alive 비활성화 -> 새 connection으로 재시도 |

**핵심 인사이트**: Simple backoff는 "같은 요청을 나중에 다시 보내면 성공할 것"이라는 가정에 기반한다. LLM API에서는 이 가정이 자주 깨진다. Context overflow는 시간이 지나도 해결되지 않고, background 529 retry는 서버를 더 압박하며, fast mode rate limit은 모드 전환 없이는 풀리지 않는다.

---

## Retry 구조: Context와 Strategy

### Retry Context

```typescript
// 각 retry 시도마다 변할 수 있는 context
export interface RetryContext {
  maxTokensOverride?: number           // Context overflow 시 조정
  model: string                        // Fallback 시 변경 가능
  thinkingConfig: ThinkingConfig       // Retry 시 thinking 비활성화 가능
  fastMode?: boolean                   // Fast mode fallback 시 변경
}
```

**설계 포인트**: Retry context가 mutable인 이유는, 각 시도마다 **요청 자체를 수정**해야 할 수 있기 때문이다. 일반 HTTP retry는 동일한 요청을 반복하지만, LLM retry는 model, max_tokens, fast mode 등을 attempt마다 조정한다.

### 기본 설정 (구체적 수치)

| 설정 | 값 | 환경변수 |
|------|-----|---------|
| Max retries | `DEFAULT_MAX_RETRIES = 10` | `CLAUDE_CODE_MAX_RETRIES` |
| Base delay | `BASE_DELAY_MS = 500ms` | - |
| Max delay | `32,000ms` (32초) | - |
| Max 529 retries | `MAX_529_RETRIES = 3` | - |
| Context overflow floor | `FLOOR_OUTPUT_TOKENS = 3,000` | - |
| Persistent max backoff | `5분` | - |
| Persistent reset cap | `6시간` | - |
| Heartbeat interval | `30초` | - |
| Fast mode fallback hold | `30분` | - |
| Fast mode short retry threshold | `20초` | - |
| Fast mode min cooldown | `10분` | - |

### Backoff 공식

```
delayMs = min(BASE_DELAY_MS * 2^(attempt-1), maxDelayMs) + jitter
jitter  = random() * 0.25 * baseDelay
```

실제 코드:

```typescript
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  // Server directive takes priority
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) {
      return seconds * 1000
    }
  }

  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),
    maxDelayMs,
  )
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

구체적 backoff 진행:

| Attempt | Base Delay | + Jitter (최대) | 실제 범위 |
|---------|-----------|----------------|----------|
| 1 | 500ms | +125ms | 500-625ms |
| 2 | 1,000ms | +250ms | 1,000-1,250ms |
| 3 | 2,000ms | +500ms | 2,000-2,500ms |
| 4 | 4,000ms | +1,000ms | 4,000-5,000ms |
| 5 | 8,000ms | +2,000ms | 8,000-10,000ms |
| 6 | 16,000ms | +4,000ms | 16,000-20,000ms |
| 7+ | 32,000ms | +8,000ms | 32,000-40,000ms (cap) |

---

## 실제 코드 발췌: shouldRetry() 핵심 분기

`shouldRetry()`는 에러를 받아 재시도 여부를 결정하는 핵심 함수다. 분기 순서가 중요하다 -- 우선순위가 높은 조건이 먼저 평가된다.

```typescript
function shouldRetry(error: APIError): boolean {
  // 1. Mock errors (테스트용) -- 절대 retry하지 않음
  if (isMockRateLimitError(error)) {
    return false
  }

  // 2. Persistent mode: 429/529는 무조건 retry
  //    subscriber 제한과 x-should-retry header를 무시
  if (isPersistentRetryEnabled() && isTransientCapacityError(error)) {
    return true
  }

  // 3. Remote mode (CCR): 401/403은 일시적 인프라 이슈
  //    x-should-retry:false를 무시 -- 서버는 bad key라고 판단하지만
  //    실제로는 인프라 JWT가 유효함
  if (
    isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
    (error.status === 401 || error.status === 403)
  ) {
    return true
  }

  // 4. SDK가 529 status를 제대로 전달 못하는 streaming 버그 대응
  //    error message에서 직접 overloaded_error 감지
  if (error.message?.includes('"type":"overloaded_error"')) {
    return true
  }

  // 5. Context overflow -- retry 가능 (max_tokens 조정 후)
  if (parseMaxTokensContextOverflowError(error)) {
    return true
  }

  // 6. x-should-retry header 존중 (비표준 header)
  //    단, Max/Pro subscriber에게는 true여도 무시 -- 수시간 뒤 retry 가능하다는 뜻이므로
  //    Enterprise는 PAYG이므로 존중
  const shouldRetryHeader = error.headers?.get('x-should-retry')
  if (
    shouldRetryHeader === 'true' &&
    (!isClaudeAISubscriber() || isEnterpriseSubscriber())
  ) {
    return true
  }

  // 7. x-should-retry: false 존중
  //    단, Anthropic 내부 사용자(ant)는 5xx에 한해 무시 가능
  if (shouldRetryHeader === 'false') {
    const is5xxError = error.status !== undefined && error.status >= 500
    if (!(process.env.USER_TYPE === 'ant' && is5xxError)) {
      return false
    }
  }

  // 8. Connection error -- 항상 retry
  if (error instanceof APIConnectionError) {
    return true
  }

  if (!error.status) return false

  // 9. Status code별 처리
  if (error.status === 408) return true  // Request timeout
  if (error.status === 409) return true  // Lock timeout

  // 10. 429: ClaudeAI subscriber는 retry 안 함 (Enterprise 제외)
  if (error.status === 429) {
    return !isClaudeAISubscriber() || isEnterpriseSubscriber()
  }

  // 11. 401: API key cache 초기화 후 retry
  if (error.status === 401) {
    clearApiKeyHelperCache()
    return true
  }

  // 12. 403 "token revoked" -- OAuth refresh 후 retry
  if (isOAuthTokenRevokedError(error)) {
    return true
  }

  // 13. 5xx server error -- 항상 retry
  if (error.status && error.status >= 500) return true

  return false
}
```

**주목할 설계 결정들**:

1. **`x-should-retry` header를 존중하되 override하는 경우가 있다**: Persistent mode에서는 완전히 무시하고, subscriber 유형에 따라 `true`의 의미가 달라진다. "수시간 후 retry 가능"이라는 `true`는 Max/Pro 사용자에게 실용적이지 않다.

2. **SDK 버그를 error message parsing으로 우회**: Streaming 중 529가 발생하면 SDK가 status code를 제대로 전달하지 못하는 경우가 있어, `"type":"overloaded_error"` 문자열을 직접 검사한다.

3. **401 처리 시 side effect**: `clearApiKeyHelperCache()`를 호출한 후 retry한다. 이는 shouldRetry가 순수 함수가 아닌 이유 -- retry 판단과 동시에 상태를 정리한다.

---

## Error 유형별 처리 전략

### 429 (Rate Limit)

```
429 발생 -> retry-after header 확인
  있으면 -> 해당 시간만큼 대기 후 재시도
  없으면 -> exponential backoff으로 재시도

anthropic-ratelimit-unified-reset header?
  있으면 -> reset 시간까지 대기 (window-based rate limit)

Overage 감지?
  있으면 -> overage 상태를 latched로 기록 (cache break 방지)
```

### 529 (Overload) -- Foreground vs Background 구분

```
529 발생 -> 요청 source 확인
  Foreground (사용자 대기 중)?
    -> Exponential backoff으로 재시도
    -> 3회 실패 후 model fallback (Opus -> Sonnet)
  Background (요약, 제안 등)?
    -> 즉시 bail (사용자가 기다리지 않으므로)
```

#### Foreground vs Background: FOREGROUND_529_RETRY_SOURCES

Claude Code는 모든 API 요청에 `QuerySource` 태그를 붙인다. 529가 발생하면 이 태그로 foreground/background를 구분한다.

```typescript
// Foreground: 사용자가 결과를 기다리고 있는 요청 -- 529 시 retry
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',                        // 메인 대화
  'repl_main_thread:outputStyle:custom',     // 커스텀 출력 스타일
  'repl_main_thread:outputStyle:Explanatory',
  'repl_main_thread:outputStyle:Learning',
  'sdk',                                     // SDK 호출
  'agent:custom',                            // 에이전트 실행
  'agent:default',
  'agent:builtin',
  'compact',                                 // Context 압축
  'hook_agent',                              // Hook 실행
  'hook_prompt',
  'verification_agent',                      // 검증 에이전트
  'side_question',                           // 사이드 질문

  // Security classifiers -- auto-mode 정확성에 필수
  'auto_mode',                               // YOLO/auto-mode 분류기
  ...(feature('BASH_CLASSIFIER') ? (['bash_classifier'] as const) : []),
])
```

**Background 요청 (retry 안 함)**: title 생성, 자동 요약, 코드 제안, 일반 classifier 등. 이 요청들이 실패해도 사용자 경험에 직접적 영향이 없다.

**왜 background를 bail하는가?**

```
서버 과부하 상황에서의 cascade 시나리오:

[시간 0s] 서버 529 반환 시작
[시간 1s] Foreground retry x3 (사용자가 기다리는 중 -- 필요)
          + Background retry x3 (요약 생성 -- 불필요)
          + Background retry x3 (제안 생성 -- 불필요)
          = 총 9건의 retry

각 retry는 gateway에서 3-10x 증폭됨 (load balancer retry, connection retry 등)
-> 불필요한 background retry가 서버 부하를 2-3배 증가시킴
-> 서버 복구 시간 지연 -> 모든 사용자에게 영향
```

**Auto-mode classifier가 foreground인 이유**: YOLO(auto-mode) 분류기는 tool 실행의 보안 게이트다. 이 classifier가 실패하면 tool이 실행되지 않아 에이전트 전체가 멈춘다. 따라서 529에서도 retry해야 한다. (`'auto_mode'` source로 태그됨)

### 401/403 (Auth Failure)

```
401/403 발생 -> OAuth token refresh 시도
  성공 -> 새 token으로 재시도
  실패 -> 사용자에게 re-login 요청

Remote mode (CCR)?
  -> x-should-retry:false를 무시하고 retry
  -> 인프라 JWT는 유효하므로 일시적 blip으로 판단
```

### Connection Errors (ECONNRESET, EPIPE)

```
Connection error -> keepalive disable 후 재시도
  이유: HTTP keepalive connection이 서버에서 닫혔을 때 발생
  해결: 새 connection으로 재시도
```

```typescript
function isStaleConnectionError(error: unknown): boolean {
  if (!(error instanceof APIConnectionError)) return false
  const details = extractConnectionErrorDetails(error)
  return details?.code === 'ECONNRESET' || details?.code === 'EPIPE'
}
```

Streaming 중단의 구체적 recovery는 [08_streaming_concurrency.md]에서 다룬다. 요약: connection이 끊기면 마지막 완성된 content block까지의 부분 응답을 보존하고, 새 API 호출로 이어서 진행한다.

### Context Overflow (Max Tokens Exceeded)

```
Max tokens exceeded -> error message에서 수치 파싱
  "input length and `max_tokens` exceed context limit: 188059 + 20000 > 200000"
  -> inputTokens=188059, contextLimit=200000
  -> availableContext = 200000 - 188059 - 1000(safety) = 10941
  -> availableContext >= FLOOR_OUTPUT_TOKENS(3000)? -> maxTokensOverride = 10941
  -> availableContext < 3000? -> CannotRetryError throw
```

이 경우는 단순 재시도가 아니라 **요청 자체를 수정**해야 한다. `retryContext.maxTokensOverride`를 설정하면 다음 attempt에서 자동으로 줄어든 max_tokens로 요청한다.

---

## Model Fallback: Opus -> Sonnet

Claude Code의 가장 독특한 resilience 패턴:

```
Opus 요청 -> 529 (overload) -> 3회 재시도 실패
  -> FallbackTriggeredError 발생
  -> Retry context의 model을 Sonnet으로 변경
  -> Sonnet으로 재시도
  -> 성공하면 사용자에게 "Sonnet으로 fallback했습니다" 알림
```

실제 코드의 트리거 조건:

```typescript
if (is529Error(error) &&
    (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
     (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
  }
}
```

**주목**: `initialConsecutive529Errors` 옵션은 streaming 중 529가 발생한 경우를 처리한다. Streaming 요청이 529로 실패하고 non-streaming fallback으로 전환될 때, 이전 529 카운트를 이어받아 총 MAX_529_RETRIES를 일관되게 유지한다.

**왜 이게 유용한가**: Opus가 overload일 때 Sonnet은 정상일 수 있다. 품질이 약간 낮아지더라도 **작업이 멈추지 않는 것**이 사용자에게 더 가치있다.

**LLM 엔지니어링 인사이트**: Model fallback chain을 설계할 때, 단순히 "가장 좋은 모델 -> 차선 모델"이 아니라, **어떤 failure mode에서 어떤 모델로 fallback할지**를 명확히 해야 한다. 모든 실패에 fallback하면 불필요하게 품질이 낮아진다.

---

## Fast Mode Fallback Mechanics

Fast mode (더 빠른 추론)가 활성화된 상태에서의 상세 fallback 로직:

### 결정 트리

```
Fast mode 요청 -> 429/529 실패
  |
  +-- Overage disabled reason header 있음?
  |     -> Fast mode 영구 비활성화 (overage 불가)
  |     -> 즉시 standard mode로 retry
  |
  +-- retry-after < 20초 (SHORT_RETRY_THRESHOLD_MS)?
  |     -> retry-after만큼 대기
  |     -> Fast mode 유지한 채 retry
  |     -> 이유: prompt cache 보존 (같은 model name 유지)
  |
  +-- retry-after >= 20초 또는 retry-after 없음?
        -> Cooldown 진입
        -> cooldownMs = max(retryAfterMs ?? 30분, 10분)
        -> Standard mode로 전환
        -> cooldown 끝날 때까지 fast mode 비활성
```

### 실제 코드

```typescript
const DEFAULT_FAST_MODE_FALLBACK_HOLD_MS = 30 * 60 * 1000 // 30 minutes
const SHORT_RETRY_THRESHOLD_MS = 20 * 1000                // 20 seconds
const MIN_COOLDOWN_MS = 10 * 60 * 1000                    // 10 minutes

// Fast mode fallback 분기 (withRetry 내부)
if (wasFastModeActive && !isPersistentRetryEnabled() &&
    error instanceof APIError &&
    (error.status === 429 || is529Error(error))) {

  // Overage disabled -> 영구 비활성화
  const overageReason = error.headers?.get(
    'anthropic-ratelimit-unified-overage-disabled-reason',
  )
  if (overageReason !== null && overageReason !== undefined) {
    handleFastModeOverageRejection(overageReason)
    retryContext.fastMode = false
    continue
  }

  const retryAfterMs = getRetryAfterMs(error)
  if (retryAfterMs !== null && retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
    // Short wait: fast mode 유지 -> prompt cache 보존
    await sleep(retryAfterMs, options.signal, { abortError })
    continue
  }

  // Long wait: cooldown 진입 -> standard mode 전환
  const cooldownMs = Math.max(
    retryAfterMs ?? DEFAULT_FAST_MODE_FALLBACK_HOLD_MS,
    MIN_COOLDOWN_MS,
  )
  const cooldownReason: CooldownReason = is529Error(error)
    ? 'overloaded' : 'rate_limit'
  triggerFastModeCooldown(Date.now() + cooldownMs, cooldownReason)
  retryContext.fastMode = false
  continue
}
```

### 왜 20초가 threshold인가?

- **Prompt cache 보존**: Fast mode와 standard mode는 다른 model name을 사용한다. 모드를 전환하면 prompt cache가 깨진다 (cache key에 model name이 포함됨).
- **20초 이하**: 짧은 대기 후 같은 모드로 retry하면 cache를 보존할 수 있다. 사용자 체감 대기 시간도 수용 가능.
- **20초 초과**: Cache를 보존하기 위해 오래 기다리는 것보다, standard mode로 전환해서 즉시 요청하는 게 낫다.
- **MIN_COOLDOWN_MS = 10분**: Mode를 너무 자주 전환하면 flip-flopping이 발생한다. 최소 10분 cooldown으로 안정성 확보.

---

## Persistent Retry Mode (Unattended)

장시간 무인 작업을 위한 특수 모드. `CLAUDE_CODE_UNATTENDED_RETRY` 환경변수로 활성화:

```
특징:
- 최대 5분 backoff 증가, 전체 6시간 cap
- 30초마다 heartbeat 메시지 yield (connection timeout 방지)
- SystemAPIErrorMessage로 상태 알림
- 429/529를 무한 retry (subscriber 제한 무시, x-should-retry 무시)
```

### Heartbeat 메커니즘

```typescript
// Persistent mode: 긴 sleep을 chunk로 나눠서 heartbeat 유지
let remaining = delayMs
while (remaining > 0) {
  if (options.signal?.aborted) throw new APIUserAbortError()
  if (error instanceof APIError) {
    yield createSystemAPIErrorMessage(error, remaining, reportedAttempt, maxRetries)
  }
  const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)  // 30초
  await sleep(chunk, options.signal, { abortError })
  remaining -= chunk
}
// for-loop가 종료되지 않도록 attempt를 clamp
if (attempt >= maxRetries) attempt = maxRetries
```

**왜 heartbeat이 필요한가**: Bridge mode에서 IDE가 "응답 없음"으로 판단하거나, remote session에서 SSH/WebSocket timeout이 발생하는 것을 방지한다. 30초마다 `{type:'system', subtype:'api_retry'}` 메시지를 stdout으로 emit한다.

### Window-based Rate Limit 처리

Persistent mode에서 429를 받으면, 단순 backoff 대신 `anthropic-ratelimit-unified-reset` header를 확인한다:

```typescript
function getRateLimitResetDelayMs(error: APIError): number | null {
  const resetHeader = error.headers?.get?.('anthropic-ratelimit-unified-reset')
  if (!resetHeader) return null
  const resetUnixSec = Number(resetHeader)
  if (!Number.isFinite(resetUnixSec)) return null
  const delayMs = resetUnixSec * 1000 - Date.now()
  if (delayMs <= 0) return null
  return Math.min(delayMs, PERSISTENT_RESET_CAP_MS)  // 6시간 cap
}
```

5시간 Max/Pro rate limit window가 있다면, 매 5분마다 무의미하게 polling하는 대신 reset 시간까지 정확히 대기한다.

**LLM 엔지니어링 인사이트**: 무인 에이전트(CI/CD, scheduled tasks)에서는 "포기하지 않는 retry"가 필요하다. 하지만 무한 retry는 비용 폭발로 이어질 수 있으므로, **시간 기반 cap**과 **heartbeat**으로 제어해야 한다.

---

## Rate Limit Parsing의 세밀함

Claude Code는 여러 rate limit header를 파싱한다:

### 1. `retry-after` (초 단위)
```
retry-after: 30 -> 30초 후 재시도
```

### 2. `anthropic-ratelimit-unified-reset` (Window-based)
```
anthropic-ratelimit-unified-reset: 1743523800
-> Unix timestamp -> Date.now()와의 차이만큼 대기
-> 6시간 cap 적용
```

### 3. Overage Detection
Rate limit 초과 시 "overage" 상태를 감지하여:
- 경고 메시지 표시
- Latched state로 기록 (cache break 방지)
- 추가 요청 억제

---

## Error Types

`src/services/api/errors.ts`가 정의하는 에러 타입:

| Type | 의미 |
|------|------|
| `CannotRetryError` | 원본 에러 + retry context를 래핑 -- 더 이상 retry 불가 |
| `FallbackTriggeredError` | Model fallback이 발생했음을 알림 |
| `APIError` | SDK의 일반 API 에러 |
| `APIConnectionError` | 네트워크 연결 에러 |
| `APIUserAbortError` | 사용자가 취소 (Ctrl+C) |

---

## 직접 만들어보기: LLM-Aware Retry State Machine

Claude Code의 retry 로직을 참고한 **최소 구현 가능한 LLM retry state machine** pseudocode. 실제 프로덕션에서 사용할 수 있는 수준의 설계다.

```
=== LLM-Aware Retry State Machine ===

Constants:
  BASE_DELAY_MS     = 500
  MAX_DELAY_MS      = 32000
  MAX_RETRIES       = 10
  MAX_529_RETRIES   = 3
  FLOOR_OUTPUT_TOKENS = 3000

State:
  attempt           = 0
  consecutive529    = 0
  currentModel      = primaryModel
  maxTokensOverride = null
  fastMode          = initialFastMode

function retryLoop(request):
  while attempt <= MAX_RETRIES:
    attempt++

    if signal.aborted:
      throw AbortError

    try:
      response = callAPI(request, {
        model: currentModel,
        maxTokens: maxTokensOverride ?? defaultMaxTokens,
        fastMode: fastMode,
      })
      return response

    catch error:
      // ---- Phase 1: Classify error ----

      match error:

        case UserAbortError:
          throw error  // Never retry user cancellation

        case ContextOverflow(inputTokens, contextLimit):
          available = contextLimit - inputTokens - 1000  // safety buffer
          if available < FLOOR_OUTPUT_TOKENS:
            throw CannotRetryError(error)  // Too little space
          maxTokensOverride = max(FLOOR_OUTPUT_TOKENS, available)
          continue  // Retry with reduced max_tokens (no delay)

        case AuthError(401, 403):
          refreshed = tryRefreshToken()
          if not refreshed:
            throw CannotRetryError(error)
          recreateClient()
          continue  // Retry with new token (no delay)

        case ConnectionError(ECONNRESET, EPIPE):
          disableKeepAlive()
          recreateClient()
          continue  // Retry with new connection (no delay)

        case FastModeError(429 | 529) when fastMode:
          retryAfterMs = parseRetryAfter(error)
          if retryAfterMs != null and retryAfterMs < 20_000:
            sleep(retryAfterMs)
            continue  // Keep fast mode, preserve cache
          else:
            cooldown = max(retryAfterMs ?? 30min, 10min)
            enterFastModeCooldown(cooldown)
            fastMode = false
            continue  // Switch to standard mode

        case Overload(529) when not isForegroundRequest:
          throw CannotRetryError(error)  // Bail immediately

        case Overload(529) when isForegroundRequest:
          consecutive529++
          if consecutive529 >= MAX_529_RETRIES and fallbackModel:
            throw FallbackTriggeredError(currentModel, fallbackModel)
          // Fall through to normal backoff

        case RateLimit(429):
          // Fall through to normal backoff

        case ServerError(5xx):
          // Fall through to normal backoff

        default:
          throw CannotRetryError(error)  // Unknown error, don't retry

      // ---- Phase 2: Compute delay ----

      retryAfter = parseRetryAfterHeader(error)
      delayMs = computeBackoff(attempt, retryAfter)

      // ---- Phase 3: Wait with heartbeat ----

      if persistentMode:
        waitWithHeartbeat(delayMs, interval=30s)
        clampAttempt()  // Never exceed maxRetries
      else:
        emitRetryStatus(error, delayMs, attempt)
        sleep(delayMs)

  throw CannotRetryError(lastError)

function computeBackoff(attempt, retryAfter, maxDelay=MAX_DELAY_MS):
  if retryAfter:
    return retryAfter  // Server directive takes priority
  baseDelay = min(BASE_DELAY_MS * 2^(attempt-1), maxDelay)
  jitter = random() * 0.25 * baseDelay
  return baseDelay + jitter
```

### State Machine 다이어그램

```
                    +--------+
                    | START  |
                    +---+----+
                        |
                        v
                 +------+-------+
            +--->| CALL API     |<-----------+
            |    +------+-------+            |
            |           |                    |
            |      +----v----+               |
            |      | SUCCESS |               |
            |      +---------+               |
            |           |                    |
            |       (return)                 |
            |                                |
            |    +----------+                |
            +----|  RETRY   |<-----+         |
            |    +-----+----+      |         |
            |          |           |         |
            |    +-----v--------+  |         |
            |    | CLASSIFY     |  |         |
            |    | ERROR        |  |         |
            |    +-----+--------+  |         |
            |          |           |         |
            |    +-----v--------+  |   +-----+------+
            |    | Context      +--+-->| Adjust     |
            |    | Overflow?    |      | maxTokens  +---+
            |    +--------------+      +------------+   |
            |          |                                |
            |    +-----v--------+      +------------+   |
            |    | Auth Error?  +----->| Refresh    +---+
            |    +--------------+      | Token      |
            |          |               +------------+
            |    +-----v--------+      +------------+
            |    | Connection   +----->| New        +---+
            |    | Error?       |      | Connection |   |
            |    +--------------+      +------------+   |
            |          |                                |
            |    +-----v--------+      +------------+   |
            |    | 529 +        +----->| BAIL       |   |
            |    | Background?  |      | (no retry) |   |
            |    +--------------+      +------------+   |
            |          |                                |
            |    +-----v--------+      +------------+   |
            |    | 529 x3 +     +----->| MODEL      |   |
            |    | Foreground?  |      | FALLBACK   +---+
            |    +--------------+      +------------+
            |          |
            |    +-----v--------+
            |    | COMPUTE      |
            |    | BACKOFF      |
            |    +-----+--------+
            |          |
            |    +-----v--------+
            +----| SLEEP +      |
                 | HEARTBEAT    |
                 +--------------+
```

---

## Retry 전체 흐름 요약

```
API 호출 시도
  |
  v
성공? -> return
  |
  v
에러 분석:
  429 (rate limit) -> retry-after 대기 -> retry
  529 (overload) + foreground -> backoff retry -> 3회 후 model fallback
  529 (overload) + background -> bail immediately
  401/403 -> token refresh -> retry
  ECONNRESET -> disable keepalive -> retry
  Max tokens exceeded -> maxTokens 조정 -> retry
  Fast mode + 429/529 -> short wait 또는 cooldown 전환
  기타 -> CannotRetryError throw
  |
  v
Max retries 초과?
  Unattended mode -> persistent retry (6hr cap, 30s heartbeat)
  Normal mode -> CannotRetryError throw
```

---

## 내 프로젝트에 적용하기: 단계별 가이드

### Level 1: Minimum Viable Retry (모든 LLM 프로젝트에 필수)

가장 기본적이지만 대부분의 프로젝트에서 빠뜨리는 것: **에러 유형 구분**.

```python
# Python pseudocode - 최소 구현
class LLMRetryError(Exception):
    """더 이상 retry 불가능한 에러"""
    pass

def call_with_retry(fn, max_retries=5, base_delay=0.5):
    for attempt in range(1, max_retries + 1):
        try:
            return fn()
        except RateLimitError as e:        # 429
            delay = parse_retry_after(e) or backoff(attempt, base_delay)
            log(f"Rate limited, waiting {delay}s")
            sleep(delay)
        except OverloadError as e:          # 529
            delay = backoff(attempt, base_delay)
            log(f"Server overloaded, waiting {delay}s")
            sleep(delay)
        except AuthError as e:              # 401/403
            if refresh_token():
                continue  # No delay needed
            raise LLMRetryError(e)
        except ContextOverflowError as e:   # 400
            # 재시도해봤자 같은 에러 발생
            raise LLMRetryError(e)  # 또는 compact 후 재시도
        except Exception as e:
            raise LLMRetryError(e)

    raise LLMRetryError(f"Max retries ({max_retries}) exceeded")

def backoff(attempt, base_delay, max_delay=32.0):
    base = min(base_delay * (2 ** (attempt - 1)), max_delay)
    jitter = random() * 0.25 * base
    return base + jitter
```

**절대 하지 말아야 할 것**: 모든 에러를 동일하게 retry하는 것. Context overflow를 retry하면 영원히 실패하고, auth error를 backoff으로 retry하면 불필요하게 시간을 낭비한다.

### Level 2: Model Fallback 추가 (프로덕션 서비스)

사용자가 응답을 기다리는 서비스라면, 특정 모델의 장애가 전체 서비스 중단으로 이어지면 안 된다.

```python
# Level 2: Model fallback 추가
# (모델 ID는 예시이며, 실제 사용 시 API 문서에서 최신 ID를 확인)
FALLBACK_CHAIN = {
    "claude-opus-4-6": "claude-sonnet-4-20250514",
    "claude-sonnet-4-20250514": "claude-haiku-3-5-20241022",
}
MAX_529_BEFORE_FALLBACK = 3

def call_with_fallback(fn, model, max_retries=10):
    consecutive_529 = 0

    for attempt in range(1, max_retries + 1):
        try:
            return fn(model=model)
        except OverloadError:
            consecutive_529 += 1
            if consecutive_529 >= MAX_529_BEFORE_FALLBACK:
                fallback = FALLBACK_CHAIN.get(model)
                if fallback:
                    log(f"Falling back: {model} -> {fallback}")
                    model = fallback
                    consecutive_529 = 0
                    continue
            sleep(backoff(attempt))
        except RateLimitError as e:
            consecutive_529 = 0  # Reset 529 counter
            sleep(parse_retry_after(e) or backoff(attempt))
        except Exception as e:
            raise
```

**언제 model fallback을 추가하는가**:
- 사용자가 실시간으로 응답을 기다리는 서비스
- SLA가 있는 프로덕션 API
- "품질 저하보다 서비스 중단이 더 나쁜" 경우

**언제 model fallback을 추가하지 않는가**:
- Batch processing (기다리면 됨)
- 품질이 결과의 핵심인 경우 (코드 리뷰, 의학 분석 등)
- 단일 모델만 지원하는 경우

### Level 3: Persistent Retry Mode 추가 (무인 에이전트)

CI/CD, cron job, 자동화 파이프라인에서 실행되는 에이전트:

```python
# Level 3: Persistent retry
PERSISTENT_MAX_BACKOFF = 300   # 5분
PERSISTENT_CAP = 21600         # 6시간
HEARTBEAT_INTERVAL = 30        # 30초

def call_persistent(fn, model, heartbeat_fn=None):
    start_time = time.time()

    for attempt in count(1):
        elapsed = time.time() - start_time
        if elapsed > PERSISTENT_CAP:
            raise TimeoutError(f"Persistent retry cap ({PERSISTENT_CAP}s) exceeded")

        try:
            return fn(model=model)
        except (RateLimitError, OverloadError) as e:
            delay = min(backoff(attempt, max_delay=PERSISTENT_MAX_BACKOFF),
                       PERSISTENT_CAP - elapsed)

            # Heartbeat: 긴 대기를 chunk로 나눔
            remaining = delay
            while remaining > 0:
                chunk = min(remaining, HEARTBEAT_INTERVAL)
                if heartbeat_fn:
                    heartbeat_fn(remaining=remaining, attempt=attempt)
                sleep(chunk)
                remaining -= chunk
```

### Level 4: Budget Guard -- Retry Storm 방지

**가장 간과되는 문제**: Retry 자체가 비용을 발생시킨다.

```python
# Level 4: Budget guard
class BudgetGuard:
    def __init__(self, max_retry_cost_usd=1.0, window_seconds=300):
        self.max_cost = max_retry_cost_usd
        self.window = window_seconds
        self.retry_costs = []  # (timestamp, estimated_cost)

    # 비용 추정 (모델 ID는 예시이며, 실제 사용 시 API 문서에서 최신 ID를 확인)
    COST_PER_1M = {
        "claude-opus-4-6": {"input": 15.0, "output": 75.0},
        "claude-sonnet-4-20250514": {"input": 3.0, "output": 15.0},
    }

    @staticmethod
    def estimate_cost(input_tokens, output_tokens_est, model):
        rates = BudgetGuard.COST_PER_1M.get(model, {"input": 15.0, "output": 75.0})
        return (input_tokens * rates["input"] + output_tokens_est * rates["output"]) / 1_000_000

    def check_budget(self, estimated_input_tokens, model):
        now = time.time()
        # 윈도우 밖의 기록 제거
        self.retry_costs = [(t, c) for t, c in self.retry_costs
                            if now - t < self.window]
        # 현재 윈도우의 총 retry 비용
        total = sum(c for _, c in self.retry_costs)
        estimated_cost = estimate_cost(estimated_input_tokens, model)

        if total + estimated_cost > self.max_cost:
            raise BudgetExceededError(
                f"Retry budget exceeded: ${total:.2f} + ${estimated_cost:.2f} "
                f"> ${self.max_cost:.2f} in {self.window}s window"
            )

        self.retry_costs.append((now, estimated_cost))

budget = BudgetGuard(max_retry_cost_usd=2.0, window_seconds=300)

def call_with_budget(fn, model, input_tokens):
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            budget.check_budget(input_tokens, model)
            return fn(model=model)
        except BudgetExceededError:
            # Fallback to cheaper model or bail
            cheaper = FALLBACK_CHAIN.get(model)
            if cheaper:
                model = cheaper
                continue
            raise
        except (RateLimitError, OverloadError):
            sleep(backoff(attempt))
```

**Retry storm 시나리오**: 긴 context (100K tokens)를 가진 요청이 429로 실패하면, 매 retry마다 동일한 100K tokens가 전송된다. 10회 retry = 1M tokens = $15+ (Opus 기준). Budget guard 없이는 이런 비용 폭발을 감지할 수 없다.

### 적용 우선순위 요약

| 프로젝트 유형 | 최소 필요 Level | 이유 |
|-------------|---------------|------|
| 개인 프로젝트/PoC | Level 1 | 에러 구분만 해도 대부분 해결 |
| 프로덕션 API | Level 2 | Model 장애 시 서비스 중단 방지 |
| CI/CD 에이전트 | Level 3 | 일시적 장애에 포기하면 안 됨 |
| 고비용 워크로드 | Level 1 + Level 4 | Retry storm으로 예산 초과 방지 |
| 고가용성 + 고비용 | Level 2 + Level 4 | 모든 것을 커버 |

---

## Key Takeaways

- **LLM API failure는 특별하다**: Rate limit, overload, context overflow 등 일반 API에는 없는 failure mode가 있다. 각각에 맞는 처리가 필요하다.
- **Foreground vs Background retry**: 사용자가 기다리는 요청과 백그라운드 요청의 retry 전략을 구분한다. Background 529 retry는 cascade를 증폭시키므로 즉시 bail한다.
- **Model fallback chain**: 고성능 모델이 실패하면 저성능 모델로 fallback. "멈추지 않는 것"이 "최적 품질"보다 중요할 때. 단, **529 overload에서만** 트리거 -- 모든 에러에 fallback하면 불필요하게 품질이 낮아진다.
- **Fast mode fallback은 cache-aware**: 20초 이하 대기는 fast mode를 유지해서 prompt cache를 보존한다. 20초 초과면 standard로 전환하되 최소 10분 cooldown으로 flip-flopping을 방지한다.
- **`x-should-retry` header는 존중하되 맹신하지 않는다**: Persistent mode, remote mode, subscriber 유형에 따라 override한다. Server의 판단과 client의 context가 다를 수 있다.
- **Persistent retry + heartbeat**: 무인 작업에서는 시간 기반 cap(6시간)과 heartbeat(30초)으로 제어된 "포기하지 않는 retry".
- **Cache break 방지**: Rate limit 관련 상태 변화가 prompt cache를 깨지 않도록 latched states 사용.
- **Budget guard**: Retry가 무료가 아님을 기억하라. 긴 context의 반복 전송은 비용 폭발로 이어질 수 있다.
