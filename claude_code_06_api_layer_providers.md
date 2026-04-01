<!--
tags: architecture/provider-abstraction, architecture/multi-provider, workflow/prompt-caching, architecture/oauth, llm-application/api-layer
keywords: multi-provider, provider-abstraction, anthropic-sdk, bedrock, vertex-ai, foundry, prompt-caching, cache-control, oauth, api-client, cost-tracking, beta-headers
related_files: 02_query_engine_core.md, 05_context_management.md, 07_retry_resilience.md
source: https://github.com/nirholas/claude-code
-->

# 06. API Layer & Multi-Provider Support

> 4개의 LLM provider를 하나의 인터페이스로 통합: Direct API, AWS Bedrock, Google Vertex AI, Azure Foundry. Prompt caching, OAuth, cost tracking.

## Keywords
multi-provider, provider-abstraction, anthropic-sdk, aws-bedrock, google-vertex-ai, azure-foundry, prompt-caching, cache-control, cache-scope, oauth, api-client, cost-tracking, beta-headers, token-pricing

## Provider Abstraction: 4개의 백엔드, 하나의 인터페이스

`src/services/api/client.ts`가 환경 변수에 따라 올바른 provider client를 생성한다:

```typescript
// Provider 선택 우선순위
if (process.env.CLAUDE_CODE_USE_BEDROCK)   → AWS Bedrock client
if (process.env.CLAUDE_CODE_USE_VERTEX)    → Google Vertex AI client
if (process.env.CLAUDE_CODE_USE_FOUNDRY)   → Azure Foundry client
else                                        → Direct Anthropic API client
```

### 각 Provider의 특징

| Provider | 인증 | 특이사항 |
|----------|------|----------|
| **Direct API** | API key or OAuth | 가장 완전한 기능 지원, prompt caching 완전 지원 |
| **AWS Bedrock** | AWS credentials | AWS IAM 기반 인증, credential refresh 필요 |
| **Google Vertex** | GCP service account | GCP 인증, 별도의 endpoint 형식 |
| **Azure Foundry** | Azure credentials | Azure AD 인증 |

### Client 생성 패턴

```typescript
// 공통 인터페이스
const client = createAnthropicClient({
  apiKey: resolvedApiKey,
  baseURL: resolvedBaseURL,
  defaultHeaders: {
    'x-app': 'claude-code',
    'User-Agent': `claude-code/${version}`,
    'X-Claude-Code-Session-Id': sessionId,
    ...customHeaders,
  },
})
```

**LLM 엔지니어링 인사이트**: Multi-provider 지원은 처음에는 "나중에 하면 되지"라고 생각하지만, 엔터프라이즈 고객은 AWS/GCP/Azure에서만 LLM을 사용할 수 있는 경우가 많다. Client 추상화를 처음부터 설계하면 나중에 provider 추가가 수월하다.

## Authentication Flows

### 1. API Key (가장 단순)
```
ANTHROPIC_API_KEY=sk-ant-... → 직접 전달
```

### 2. OAuth 2.0 (Claude.ai 로그인)
```
OAuth flow → Token 저장 (~/.claude/.credentials.json) → 자동 refresh
```

OAuth token refresh는 retry logic과 통합되어 있다:
- 401/403 에러 시 자동으로 token refresh 시도
- Refresh 실패 시 re-login 유도

### 3. Cloud Provider Credentials
```
AWS: IAM credentials → STS AssumeRole → Bedrock client
GCP: Service account JSON → Access token → Vertex client
Azure: Azure AD → Bearer token → Foundry client
```

각 provider의 credential refresh는 API 호출 전에 자동으로 수행된다.

### 4. API Key Helper Command
외부 프로그램으로 API key를 가져오는 패턴:
```
CLAUDE_CODE_API_KEY_HELPER=my-vault-cli → 실행하여 key 획득
```

### 5. File Descriptor Passing
관리 환경에서 file descriptor로 key를 전달:
```
CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR=3 → fd 3에서 key 읽기
```

**LLM 엔지니어링 인사이트**: 인증은 "API key 하나면 된다"에서 시작하지만, 프로덕션에서는 OAuth refresh, cloud IAM, vault integration, managed environments까지 필요해진다. 처음부터 인증을 추상화 레이어로 분리하는 것이 현명하다.

## Prompt Caching: 비용과 속도 최적화

Claude Code는 Anthropic의 prompt caching을 적극 활용하여 비용과 latency를 줄인다.

### Cache Control 설정

```typescript
// System prompt에 cache control 추가
{
  type: "text",
  text: systemPromptContent,
  cache_control: { type: "ephemeral" }  // 이 부분을 캐시
}
```

### Cache Scope

| Scope | TTL | Use Case |
|-------|-----|----------|
| Default (5min) | 5분 | 일반 대화 |
| 1 Hour | 1시간 | 긴 작업 세션 |
| Global | 전역 | 조직 전체 공유 |

`PROMPT_CACHING_SCOPE_BETA_HEADER`로 scope를 제어한다.

### Cache Break Detection

`src/services/api/promptCacheBreakDetection.ts`가 cache가 깨지는 시점을 추적한다:

**Cache가 깨지는 원인들**:
- System prompt 내용 변경
- Tool schema 변경 (tool 추가/제거)
- Beta headers 변경
- Model 변경
- Cache control scope/TTL 변경

**Cache break 추적 방법**:
- 각 요소의 hash를 저장
- 이전 요청의 hash와 비교
- 변경 시 diff를 기록 (어떤 요소가 cache를 깬 건지 진단용)

### Latched States (Cache 안정성)

한번 활성화되면 세션 내에서 되돌리지 않는 상태들:
- **Auto mode**: 활성화 후 비활성화하면 cache가 깨지므로 유지
- **Overage usage**: 감지 후 reset하면 cache가 깨지므로 유지
- **Cache editing**: 활성화 후 비활성화하면 cache가 깨지므로 유지

**LLM 엔지니어링 인사이트**: Prompt caching은 "설정하면 끝"이 아니다. Cache가 예상치 못하게 깨지면 비용이 급증한다. Claude Code처럼 **cache break를 모니터링하고 의도치 않은 break를 방지**하는 메커니즘이 필요하다.

## Beta Headers: 고급 기능 활성화

Claude Code가 사용하는 beta headers:

| Header | 기능 |
|--------|------|
| `AFK_MODE_BETA_HEADER` | Away mode (백그라운드 작업) |
| `CONTEXT_1M_BETA_HEADER` | 1M token context window |
| `CONTEXT_MANAGEMENT_BETA_HEADER` | Microcompact (server-side context management) |
| `EFFORT_BETA_HEADER` | Effort/quality control |
| `FAST_MODE_BETA_HEADER` | Fast inference mode |
| `PROMPT_CACHING_SCOPE_BETA_HEADER` | Cache scope 설정 |
| `REDACT_THINKING_BETA_HEADER` | Thinking content redaction |
| `STRUCTURED_OUTPUTS_BETA_HEADER` | JSON output validation |
| `TASK_BUDGETS_BETA_HEADER` | Task spending limits |

**LLM 엔지니어링 인사이트**: Beta headers는 "있으면 좋은 것"이 아니라 프로덕션 기능의 핵심이다. Fast mode는 UX를, prompt caching은 비용을, structured outputs는 신뢰성을 직접적으로 개선한다.

## Cost Tracking

`src/cost-tracker.ts`가 매 API 호출의 비용을 추적한다:

```typescript
// 모델별 usage 추적
modelUsage: {
  "claude-opus-4-6": {
    inputTokens: 150000,
    outputTokens: 25000,
    cacheCreationTokens: 80000,
    cacheReadTokens: 50000,
    estimatedCostUSD: 2.45,
  }
}
```

- Per-model 사용량 분리 추적 (Opus, Sonnet, Haiku 각각)
- Cache-aware 비용 계산 (cache read는 cache creation보다 저렴)
- `/cost` 명령어로 현재 세션의 누적 비용 확인

## Request/Response Normalization

`src/services/api/claude.ts` (~5000 lines)가 API와의 통신을 표준화한다:

- `BetaMessageStreamParams`: 요청 파라미터 타입
- `BetaRawMessageStreamEvent`: 스트리밍 이벤트 타입
- `BetaMessage`: 완성된 응답 타입
- `BetaToolUnion`: Tool 정의 타입

Provider별로 약간씩 다른 응답 형식을 통일된 내부 형식으로 변환한다.

## 내 프로젝트에 적용하기

### 멀티 프로바이더 지원 구현하기

**Step 1: Provider interface 정의**

처음부터 provider를 추상화한다. 나중에 추가하면 리팩토링 비용이 크다:

```typescript
interface LLMProvider {
  createMessage(params: CreateMessageParams): AsyncIterable<StreamEvent>
  countTokens(messages: Message[]): Promise<number>
  readonly name: string
  readonly supportsPromptCaching: boolean
}

// 환경 변수로 provider 선택
function createProvider(): LLMProvider {
  if (process.env.USE_BEDROCK) return new BedrockProvider()
  if (process.env.USE_VERTEX) return new VertexProvider()
  return new AnthropicDirectProvider()  // default
}
```

**Step 2: Cache break monitoring 기본 구현**

Prompt caching을 쓴다면 cache가 깨지는 시점을 반드시 추적한다:

```typescript
function detectCacheBreak(prev: RequestParams, curr: RequestParams): string[] {
  const breaks: string[] = []
  if (hash(prev.systemPrompt) !== hash(curr.systemPrompt)) breaks.push('system_prompt')
  if (hash(prev.tools) !== hash(curr.tools)) breaks.push('tools')
  if (prev.model !== curr.model) breaks.push('model')
  if (breaks.length > 0) logger.warn('Cache break detected:', breaks)
  return breaks
}
```

**Step 3: Auth 추상화 레이어**

최소한 API key + OAuth 두 가지를 지원하는 auth layer를 만든다:

```typescript
interface AuthProvider {
  getCredentials(): Promise<{ headers: Record<string, string> }>
  refresh?(): Promise<void>
}
```

**Step 4: Cost tracking 추가**

API 호출마다 usage를 추적한다. Cache read/write를 구분해야 정확한 비용 계산이 가능:

```typescript
type Usage = { inputTokens: number; outputTokens: number; cacheReadTokens?: number; cacheWriteTokens?: number }
```

## 대안 비교

| 접근 방식 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **직접 SDK 래핑** (Claude Code 방식) | 완전한 제어, 최적화 가능 | 각 provider별 코드 필요 | 프로덕션 앱, 세밀한 비용 제어 |
| **LiteLLM** | 100+ provider 즉시 지원 | 추상화 비용, 디버깅 어려움 | 빠른 프로토타입, 다양한 모델 실험 |
| **OpenAI-compatible API** | 대부분 provider가 지원 | Anthropic 고유 기능 못 씀 (cache, thinking) | OpenAI 중심 앱 |
| **Gateway (Portkey, Helicone)** | 로깅/비용 추적 내장 | 외부 의존성, latency 추가 | 팀 단위 사용, 거버넌스 필요 |

## Key Takeaways

- **Provider abstraction**: 환경 변수 하나로 provider를 전환할 수 있는 구조. 엔터프라이즈 요구사항 대응에 필수.
- **Auth 추상화**: API key, OAuth, cloud IAM, vault integration을 하나의 레이어로 통합.
- **Prompt cache monitoring**: Cache break를 추적하고 의도치 않은 break를 latched states로 방지.
- **Beta headers 활용**: 비용, 속도, 신뢰성 개선을 위한 API provider의 고급 기능을 적극 활용.
- **Cache-aware cost tracking**: 단순 token count가 아닌, cache 상태를 고려한 정확한 비용 추적.
- **실전 적용 시**: Provider interface를 먼저 정의하고, cache break detection을 초기에 넣어라. "나중에 하면 되지"가 아니라 비용 급증 사고를 예방하는 필수 장치다.
