<!--
tags: architecture/system-design, architecture/module-organization, architecture/request-lifecycle, architecture/startup-optimization
keywords: claude-code, architecture, module-map, request-lifecycle, startup-sequence, dependency-graph, bun-runtime, react-ink, pipeline-model
related_files: 00_index.md, 02_query_engine_core.md, 04_tool_system.md, 16_performance_optimization.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 01. Architecture Overview

> Claude Code의 전체 시스템 아키텍처: 모듈 구조, 요청 라이프사이클, 스타트업 시퀀스, 핵심 설계 결정.

## Keywords
architecture, module-organization, request-lifecycle, startup-sequence, pipeline-model, bun, react-ink, commander-js, tool-loop, dependency-graph

## 파이프라인 모델: 모든 것은 하나의 흐름

Claude Code의 전체 아키텍처는 단일 파이프라인으로 설명된다:

```
User Input → CLI Parser → Query Engine → LLM API → Tool Execution Loop → Terminal UI
```

이 단순한 흐름이 512K 줄의 코드로 확장되는 이유는, 각 단계에 **프로덕션 요구사항**(retry, security, permissions, caching, multi-provider, streaming)이 겹겹이 쌓이기 때문이다. 프로토타입에서 프로덕션으로 가는 과정의 현실을 보여주는 좋은 사례다.

**LLM 엔지니어링 인사이트**: 에이전트 앱의 핵심은 `while(LLM이 도구를 호출하는 동안) { 실행 → 결과 반환 }` 루프다. 나머지 모든 코드는 이 루프를 안정적으로 만들기 위한 인프라다.

## 모듈 구조: 6개 핵심 레이어

### Layer 1: Entry & CLI (`src/main.tsx`, `src/entrypoints/`)

Commander.js로 CLI 인자를 파싱하고, React/Ink 렌더러를 초기화한다. `main.tsx`가 4,684줄인 이유는 단순한 파싱이 아니라 **parallel prefetch**(MDM 설정, Keychain, API preconnect)를 import 전에 시작하기 때문이다.

```
src/main.tsx          → CLI parser + parallel prefetch
src/entrypoints/
  cli.tsx             → CLI session orchestration
  init.ts             → Config, telemetry, OAuth, MDM policy
  mcp.ts              → MCP server mode entry
  sdk/                → Agent SDK (programmatic API)
```

### Layer 2: Query Engine (`src/QueryEngine.ts`, ~46K lines)

시스템의 심장. 하나의 거대한 파일에 모든 핵심 로직이 집중되어 있다: streaming response 처리, tool loop, retry, token counting, context management. [상세 분석 → 02_query_engine_core.md]

### Layer 3: Tool & Command System

```
src/Tool.ts           → Tool 타입 정의 + buildTool() factory (~29K lines)
src/tools.ts          → Tool registry (feature flag gating)
src/tools/            → 42개 개별 tool 구현
src/commands.ts       → Command registry (~25K lines)
src/commands/         → 87개 slash command 구현
```

Tool과 Command의 차이: **Tool은 LLM이 호출**하고, **Command는 사용자가 호출**한다. Command 중 `PromptCommand` 타입은 내부적으로 LLM에게 특화된 프롬프트를 보내므로 간접적으로 tool loop를 트리거한다.

### Layer 4: Services (`src/services/`)

외부 시스템과의 통합. 모든 서비스는 graceful degradation을 지원한다 — 하나가 실패해도 전체 앱은 동작한다.

| Service | 역할 | Failure Mode |
|---------|------|-------------|
| `api/` | Anthropic SDK client, multi-provider | Fatal (앱 기능 불가) |
| `mcp/` | MCP protocol client | Graceful (MCP 도구만 비활성) |
| `oauth/` | OAuth 2.0 인증 | Fallback to API key |
| `compact/` | Context compression | Graceful (압축 없이 진행) |
| `analytics/` | GrowthBook feature flags | Graceful (모든 flag → false) |
| `policyLimits/` | 조직 rate limit | Graceful (제한 없이 진행) |
| `plugins/` | Plugin loader | Graceful (플러그인 없이 진행) |

### Layer 5: UI (`src/screens/`, `src/components/`, `src/hooks/`)

React + Ink로 구축된 터미널 UI. 웹 앱과 동일한 패턴(components, hooks, state)을 터미널에서 사용한다. React Compiler가 활성화되어 최적화된 re-render를 제공한다.

### Layer 6: State & Persistence

```
src/state/AppStateStore.ts  → Global mutable state
src/utils/sessionStorage.ts → Session JSONL persistence
src/memdir/                 → Persistent memory (CLAUDE.md)
```

## Request Lifecycle: 사용자 입력에서 응답까지

```
1. 사용자가 터미널에 입력
   ↓
2. processUserInput() → Unicode 정규화, slash command 파싱, 이미지 첨부 처리
   ↓
3. SystemPrompt 조립:
   - Static instructions (src/constants/prompts.ts)
   - Dynamic context (OS, shell, git, working directory)
   - Tool descriptions (42개 tool schema → prompt text)
   - Memory (CLAUDE.md, .claude/ files)
   - Feature-gated sections (voice, coordinator, proactive 등)
   ↓
4. QueryEngine.query() 호출
   ↓
5. Message normalization → API format으로 변환
   ↓
6. API 호출 (streaming):
   - Provider 선택 (Direct / Bedrock / Vertex / Foundry)
   - Beta headers 설정 (thinking, fast mode, 1M context 등)
   - Prompt cache control 설정
   ↓
7. Streaming response 처리:
   - text blocks → 즉시 터미널에 렌더링
   - thinking blocks → budget 관리하며 처리
   - tool_use blocks → Tool execution으로 분기
   ↓
8. Tool Execution Loop:
   a. tool_use block 감지
   b. Permission check (사용자 승인 필요 여부)
   c. Tool 실행 (concurrent-safe면 병렬, 아니면 직렬)
   d. 결과를 tool_result로 포맷
   e. 다시 API에 전송 → 7번으로 돌아감
   ↓
9. LLM이 tool 호출 없이 text만 반환하면 루프 종료
   ↓
10. 최종 응답 렌더링 + 세션 저장 + 비용 추적
```

**핵심 관찰**: 7-8번 사이의 루프가 에이전트의 본질이다. LLM이 "더 이상 도구가 필요 없다"고 판단할 때까지 계속 반복된다. 이 루프의 안정성이 곧 제품의 안정성이다.

## Startup Sequence: 속도를 위한 병렬화

Claude Code의 시작 시퀀스는 **극도로 최적화**되어 있다. `main.tsx`에서 무거운 모듈을 import하기 전에 먼저 side-effect를 fire한다:

```typescript
// main.tsx 최상단 — import 전에 실행
prefetchMDMSettings()      // MDM 정책 비동기 로드
prefetchKeychainRead()     // macOS Keychain 비동기 읽기
preconnectAPI()            // API 서버에 TCP connection 미리 열기
```

이후 초기화 순서:

1. **Config load** (memoized, `~/.claude/.claude.json`)
2. **Auth check** (API key or OAuth token)
3. **Feature flags** (GrowthBook, fail → all false)
4. **Policy limits** (fail → no limits)
5. **MCP servers** (async, fail → no MCP tools)
6. **Plugin load** (async, fail → no plugins)

**LLM 엔지니어링 인사이트**: LLM CLI의 시작 시간은 UX에 직접적 영향을 준다. `import()` 전에 네트워크 요청을 시작하는 패턴은 Node/Bun 기반 CLI에서 광범위하게 적용 가능하다.

## 핵심 설계 결정과 그 이유

### 결정 1: 단일 거대 파일 (QueryEngine.ts, 46K lines)

일반적인 "작은 파일" 원칙에 반하지만, LLM 엔진의 모든 상태 전이(streaming → tool execution → retry → response)가 **긴밀하게 결합**되어 있어서 분리하면 오히려 복잡도가 증가한다. 이는 의도적인 결정이며, 컴파일러/인터프리터와 유사한 패턴이다.

### 결정 2: React + Ink for Terminal UI

터미널 UI에 React를 사용하는 것은 과하게 보일 수 있지만, 140개의 컴포넌트와 80개의 hook이 필요한 규모에서는 React의 컴포넌트 모델이 정당화된다. 특히 **streaming 중 UI 업데이트**(텍스트가 흘러나오면서 동시에 tool 상태가 바뀌는 것)를 선언적으로 처리할 수 있다.

### 결정 3: Bun Runtime

Node.js 대신 Bun을 선택한 이유:
- Native JSX/TSX 지원 (별도 transpilation 불필요)
- `bun:bundle` feature flags로 build-time dead code elimination
- 빠른 시작 시간
- ES modules + `.js` extension 컨벤션

### 결정 4: Feature Flags via Dead Code Elimination

```typescript
import { feature } from 'bun:bundle'

if (feature('VOICE_MODE')) {
  // 이 코드 블록은 VOICE_MODE가 off일 때 빌드에서 완전히 제거됨
  const voiceCommand = require('./commands/voice/index.js').default
}
```

Runtime flag 대신 build-time flag를 사용하여 **번들 크기와 시작 시간 모두 최적화**. 활성화되지 않는 기능(voice, coordinator, daemon 등)은 바이너리에 존재하지 않는다.

### 결정 5: Self-Contained Tool Modules

각 tool은 독립된 디렉토리에 모든 것을 포함한다:

```
src/tools/BashTool/
  BashTool.ts      → 실행 로직
  UI.tsx           → 터미널 렌더링
  prompt.ts        → 시스템 프롬프트 기여
  utils.ts         → Tool-specific 헬퍼
```

새 tool을 추가할 때 다른 tool의 코드를 건드릴 필요가 없다. Schema, permission, execution, UI가 모두 한 곳에 있다.

## 내 프로젝트에 적용하기

### 내 LLM 앱의 아키텍처를 설계할 때

**Step 1: 파이프라인 골격 먼저 설계**

프로토타입 단계에서 가장 먼저 할 일은 파이프라인의 각 단계를 명확한 함수/모듈로 분리하는 것이다:

```typescript
// 이 5개 함수가 앱의 뼈대. 여기서 시작하라.
async function processInput(raw: string): ProcessedInput { ... }
async function buildPrompt(input: ProcessedInput, context: Context): Message[] { ... }
async function callLLM(messages: Message[]): AsyncIterable<StreamEvent> { ... }
async function executeTools(toolCalls: ToolCall[]): ToolResult[] { ... }
async function renderResponse(stream: AsyncIterable<StreamEvent>): void { ... }
```

**Step 2: Hard dependency vs Graceful degradation 분류**

모든 외부 서비스를 두 카테고리로 분류한다:

| Hard Dependency (실패 시 앱 불가) | Graceful Degradation (실패 시 기능 축소) |
|---|---|
| LLM API client | 캐시 서버 |
| 인증 서비스 (최소 1개) | 텔레메트리/분석 |
| | 플러그인/확장 |
| | Feature flag 서버 |

원칙: Hard dependency는 **최소화**한다. 가능하면 fallback을 둔다 (예: OAuth 실패 → API key fallback).

**Step 3: Startup 병렬화 적용**

`import` 전에 네트워크 I/O를 시작하는 패턴을 적용한다. Node.js/Bun CLI라면 `main.ts` 최상단에 `fetch()`, `dns.lookup()` 등을 먼저 fire:

```typescript
// main.ts 최상단 — 어떤 import보다 먼저
const configPromise = fetch('https://config.example.com/flags')
const authPromise = readKeychain()

// 이 아래에서 무거운 import 시작
import { App } from './app.js'
```

**Step 4: Tool/Command 모듈 구조 결정**

새 기능을 추가할 때 기존 코드를 건드리지 않는 구조를 목표로 한다. Self-contained 디렉토리 패턴:

```
src/tools/MyNewTool/
  index.ts       → schema + execution
  ui.tsx         → 렌더링 (있다면)
  prompt.ts      → 시스템 프롬프트 기여분
```

## 대안 비교

| 아키텍처 패턴 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **단일 파이프라인** (Claude Code) | 단순, 디버깅 쉬움 | 거대 파일 발생 가능 | 대부분의 LLM 앱 |
| **Event-driven / Actor** (AutoGen) | 유연한 agent 통신 | 디버깅 어려움, 복잡 | 복잡한 multi-agent |
| **Graph-based** (LangGraph) | 시각화 가능, 분기 명확 | 오버헤드, 학습 곡선 | 복잡한 조건 분기 워크플로우 |
| **Framework 기반** (LangChain) | 빠른 프로토타입 | 추상화 비용, 디버깅 어려움 | 빠른 PoC |

실무 권장: 프로토타입은 단일 파이프라인으로 시작하고, 필요에 따라 복잡도를 추가한다. 처음부터 framework에 의존하면 디버깅이 어려워진다.

## Key Takeaways

- **파이프라인 모델**: 에이전트 앱의 핵심은 `입력 → 프롬프트 조립 → API 호출 → 도구 실행 루프 → 응답`이라는 단순한 파이프라인이다. 복잡도는 각 단계의 edge case 처리에서 발생한다.
- **Graceful Degradation**: 모든 외부 서비스는 실패해도 앱이 동작한다. API client만이 유일한 hard dependency다.
- **Parallel Prefetch**: CLI 시작 시 `import()` 전에 네트워크 요청을 시작하는 패턴은 체감 속도를 크게 개선한다.
- **Build-time Feature Flags**: Runtime flag 대신 dead code elimination으로 사용하지 않는 기능을 바이너리에서 완전히 제거한다.
- **Self-Contained Modules**: Tool/Command를 독립 모듈로 구성하면 500+ 파일 규모에서도 개별 기능을 독립적으로 개발·테스트할 수 있다.
- **실전 적용 시**: 파이프라인 골격 → dependency 분류 → startup 병렬화 → 모듈 구조 순으로 설계하라. 각 단계를 명확한 인터페이스로 분리해두면 나중에 provider 교체, retry 추가, caching 추가가 수월하다.
