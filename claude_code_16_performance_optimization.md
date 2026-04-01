<!--
tags: architecture/performance, architecture/lazy-loading, workflow/startup-optimization, llm-application/performance
keywords: performance, parallel-prefetch, lazy-loading, tool-schema-cache, memoization, dead-code-elimination, startup-time, bundle-size, dynamic-import, esbuild
related_files: 01_architecture_overview.md, 06_api_layer_providers.md, 15_configuration_system.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 16. Performance Optimization

> CLI 시작 시간이 UX를 결정한다: parallel prefetch, lazy loading, tool schema caching, memoization, dead code elimination.

## Keywords
performance, parallel-prefetch, lazy-loading, dynamic-import, tool-schema-cache, memoization, dead-code-elimination, startup-time, bundle-size, esbuild, bun-bundle, preconnect

## Startup Optimization: 첫 인상이 중요하다

CLI 도구의 시작 시간은 사용자 만족도에 직접 영향한다. 2초 이상 걸리면 "느리다"는 인식이 생긴다.

### Parallel Prefetch Pattern

`src/main.tsx`의 최상단 — **import 문 이전에** 실행:

```typescript
// 이 코드들은 어떤 모듈보다 먼저 실행됨
prefetchMDMSettings()     // MDM 정책 서버에 비동기 요청
prefetchKeychainRead()    // macOS Keychain에서 API key 비동기 읽기
preconnectAPI()           // Anthropic API 서버에 TCP handshake 시작
```

**왜 import 전인가**: TypeScript/JavaScript에서 `import` 문은 모듈을 로드하고 초기화한다. 무거운 모듈(React, Ink, Commander 등)의 초기화에 수백 ms가 걸린다. 이 시간 동안 네트워크 요청을 병렬로 시작하면, 모듈 로드 완료 시점에 네트워크 응답이 이미 도착해 있다.

```
시간 ──────────────────────────────────────►

Without prefetch:
  [Module loading...300ms][MDM fetch...200ms][Keychain...100ms][API connect...150ms]
  Total: ~750ms

With prefetch:
  [Module loading...300ms]
  [MDM fetch........200ms]   ← 병렬 시작
  [Keychain....100ms]        ← 병렬 시작
  [API connect..150ms]       ← 병렬 시작
  Total: ~300ms (가장 오래 걸리는 작업 = module loading)
```

**LLM 엔지니어링 인사이트**: 이 패턴은 모든 CLI 앱에 적용 가능하다. `import` 전에 네트워크 I/O를 시작하는 것은 "무료 최적화"에 가깝다.

## Lazy Loading: 무거운 모듈 지연 로드

```typescript
// OpenTelemetry (~400KB) — 텔레메트리 필요할 때만 로드
const { OTel } = await import('./services/analytics/otel.js')

// gRPC (~700KB) — gRPC 연결 필요할 때만 로드
const { grpc } = await import('@grpc/grpc-js')

// Heavy tools — 사용될 때만 로드
const { AgentTool } = await import('./tools/AgentTool/index.js')
```

### Lazy Loading 효과

| Module | Size | 언제 필요 | Load 방식 |
|--------|------|----------|-----------|
| OpenTelemetry | ~400KB | 텔레메트리 전송 시 | Dynamic import |
| gRPC | ~700KB | gRPC 연결 시 | Dynamic import |
| Voice modules | ~200KB | 음성 기능 사용 시 | Feature flag + dynamic import |
| Bridge modules | ~150KB | IDE 연동 시 | Feature flag + dynamic import |

## Tool Schema Cache

`src/utils/toolSchemaCache.ts`가 tool schema 렌더링 결과를 캐시한다.

### 문제

42개 tool의 schema를 JSON Schema로 변환하고 시스템 프롬프트에 포함하는 작업이 매 API 호출마다 발생. 결과는 ~11K tokens.

### 해결

```typescript
// 세션 시작 시 한 번 렌더링
const cachedSchemas = renderToolSchemas(tools)
schemaCache.set(toolsHash, cachedSchemas)

// 이후 API 호출마다 캐시 사용
const schemas = schemaCache.get(toolsHash) || renderToolSchemas(tools)
```

### Cache Invalidation Triggers

| Trigger | 이유 |
|---------|------|
| OAuth token refresh | Tool 접근 권한 변경 가능 |
| Tool 추가/제거 | Schema 목록 변경 |
| MCP 서버 연결/해제 | Dynamic tool 목록 변경 |
| Feature flag 변경 | 사용 가능 tool 변경 |

### Prompt Caching과의 연계

Tool schema가 바뀌면 prompt cache도 깨진다. Schema cache가 안정적이면 prompt cache hit rate가 높아지고, 비용이 절감된다.

## Memoization Patterns

`lodash.memoize()`를 사용한 패턴들:

```typescript
// 환경 감지 — 세션당 한 번
const getGlobalClaudeFile = memoize(() => {
  // ~/.claude/CLAUDE.md 경로 확인 (legacy path 포함)
})

const detectPackageManagers = memoize(() => {
  // npm, yarn, pnpm, bun 감지
})

const detectRuntimes = memoize(() => {
  // Node.js, Bun, Deno 감지
})
```

### Memoization 기준

- **변하지 않는 것**: OS, shell, 설치된 패키지 관리자 → Session memoize
- **가끔 변하는 것**: Git status, 파일 목록 → 짧은 TTL 또는 invalidation trigger
- **자주 변하는 것**: Conversation state, token count → Memoize 안 함

## Dead Code Elimination

`bun:bundle`의 `feature()` 함수가 build-time에 코드를 제거:

```typescript
import { feature } from 'bun:bundle'

// feature('VOICE_MODE')가 false면, 이 블록 전체가 빌드에서 제거
if (feature('VOICE_MODE')) {
  import('./voice/voiceEngine.js')  // ~200KB 모듈
  import('./services/voiceStreamSTT.js')
  import('./hooks/useVoice.js')
}
```

### 제거되는 코드량

| Feature | 대략적 코드량 | 빌드 포함 여부 |
|---------|------------|--------------|
| VOICE_MODE | ~200KB | External build만 |
| COORDINATOR_MODE | ~150KB | External build만 |
| DAEMON | ~100KB | Internal build만 |
| PROACTIVE | ~100KB | Internal build만 |
| BRIDGE_MODE | ~150KB | IDE build만 |

외부 배포 빌드에서 internal 기능이 제거되면 **번들 크기가 ~500KB 이상 줄어든다**.

## Build System

`scripts/build-bundle.ts`가 esbuild로 번들링:

```typescript
// 단일 파일 번들 생성
esbuild.build({
  entryPoints: ['src/main.tsx'],
  bundle: true,
  outfile: 'dist/cli.mjs',
  platform: 'node',
  target: 'esnext',
  minify: isProduction,
  // Feature flags injection
  define: {
    'feature("VOICE_MODE")': 'false',
    'feature("COORDINATOR_MODE")': 'false',
    // ...
  }
})
```

결과: **단일 `dist/cli.mjs` 파일**로 배포. Dependencies가 번들에 포함되어 `node_modules` 없이 실행 가능.

## 내 프로젝트에 적용하기

### LLM 앱 최적화 체크리스트

**Step 1: Import 전 prefetch (가장 높은 ROI)**

이것 하나만 해도 시작 시간이 30-50% 줄어든다:

```typescript
// main.ts 최상단 — 첫 번째 import보다 위에
const configP = fetch(CONFIG_URL).then(r => r.json()).catch(() => DEFAULT_CONFIG)
const authP = readKeychain().catch(() => null)

// 그 다음에 무거운 import
import { App } from './app.js'          // 200ms+
import { ReactInk } from 'ink'           // 100ms+

// import 완료 시점에 config/auth가 이미 도착해 있음
const [config, auth] = await Promise.all([configP, authP])
```

**Step 2: Lazy loading 대상 식별**

모든 모듈의 크기를 측정하고, 시작 시 불필요한 것을 dynamic import로 전환:

```typescript
// 측정: bundlesize, source-map-explorer, 또는 단순 timing
console.time('import')
import('./heavy-module.js')
console.timeEnd('import')

// 기준: 100KB 이상이면서 시작 시 불필요 → lazy loading
// 예: telemetry, gRPC, voice, analytics
```

**Step 3: Schema/prompt caching**

매 API 호출마다 tool schema를 재생성하지 말고 캐시:

```typescript
let cachedToolPrompt: string | null = null
let cachedToolsHash: string | null = null

function getToolPrompt(tools: Tool[]): string {
  const hash = computeHash(tools)
  if (hash === cachedToolsHash) return cachedToolPrompt!
  cachedToolPrompt = renderToolSchemas(tools)
  cachedToolsHash = hash
  return cachedToolPrompt
}
```

이것이 prompt caching과 연계되면 비용 절감 효과가 배가된다: schema가 안정적 → prompt cache hit → cache read 비용만 지불.

**Step 4: Memoization 기준 정하기**

| 변경 빈도 | 예시 | 전략 |
|---|---|---|
| 앱 lifetime 동안 불변 | OS, shell, 설치된 도구 | Session memoize |
| 가끔 변경 | Git branch, 파일 목록 | TTL 기반 캐시 (30s~5min) |
| 매 호출마다 변경 | Token count, cost | Memoize 안 함 |

## Key Takeaways

- **Import 전 네트워크 prefetch**: 모듈 로딩과 네트워크 I/O를 병렬화하면 시작 시간을 극적으로 줄일 수 있다.
- **Lazy loading for heavy modules**: 모든 모듈을 시작 시 로드하지 않는다. 필요할 때만 `import()`.
- **Tool schema caching**: ~11K tokens의 schema를 매번 재생성하지 않고 캐시. Prompt cache hit rate 향상.
- **Build-time dead code elimination**: 미사용 기능을 런타임 분기가 아닌 빌드 시 완전 제거. 번들 크기 500KB+ 절감.
- **단일 파일 번들**: esbuild로 모든 dependency를 하나의 파일에 번들. 배포와 실행이 단순해진다.
- **실전 적용 시**: 최적화 우선순위는 (1) import 전 prefetch, (2) lazy loading, (3) schema caching, (4) memoization 순이다. (1)이 가장 ROI가 높고 구현도 가장 간단하다.
