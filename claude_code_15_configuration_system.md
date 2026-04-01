<!--
tags: architecture/configuration, architecture/feature-flags, workflow/enterprise-policies, llm-application/config-management
keywords: configuration, zod-schemas, config-migration, feature-flags, bun-bundle, environment-variables, enterprise-policies, MDM, settings-cache, dead-code-elimination
related_files: 01_architecture_overview.md, 14_state_persistence.md, 16_performance_optimization.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 15. Configuration System

> Zod schemas, config migrations, feature flags (build-time dead code elimination), environment variable 우선순위, enterprise MDM policies.

## Keywords
configuration, zod-schemas, config-migration, feature-flags, bun-bundle, dead-code-elimination, environment-variables, enterprise-policies, MDM, settings-cache, config-validation, settings-json

## Configuration Loading

`src/utils/config.ts`의 `getConfig()`:

```typescript
const getConfig = memoize(async () => {
  // Re-entrancy guard (logEvent → getConfig 순환 방지)
  if (isLoading) return defaultConfig

  // Legacy path: ~/.claude/.config.json
  // Current path: ~/.claude/.claude.json
  const configPath = getConfigPath()

  // JSON 파싱 + Zod 검증
  const raw = JSON.parse(readFileSync(configPath))
  return configSchema.parse(raw)
})
```

### 주목할 패턴

- **Memoization**: `lodash.memoize()`로 한 번만 로드
- **Re-entrancy guard**: Config 로드 중 logging → logging이 config 접근 → 무한 루프 방지
- **Legacy migration**: 이전 경로(`~/.claude/.config.json`)에서 현재 경로(`~/.claude/.claude.json`)로 자동 마이그레이션
- **Atomic write**: Temp file + rename 패턴

## Zod Schema Validation

`src/schemas/`에 모든 설정의 schema가 Zod v4로 정의되어 있다:

```typescript
const userSettingsSchema = z.object({
  theme: z.enum(['dark', 'light', 'system']).default('system'),
  model: z.string().optional(),
  maxTokens: z.number().min(1).max(100000).optional(),
  permissionMode: z.enum(['default', 'plan', 'bypassPermissions', 'auto']),
  allowedTools: z.array(z.string()).optional(),
  mcpServers: z.record(mcpServerConfigSchema).optional(),
  // ...
})
```

### Schema의 역할

1. **Runtime validation**: 잘못된 설정값을 로드 시점에 감지
2. **TypeScript 타입 추론**: `z.infer<typeof schema>`로 타입 자동 생성
3. **Default 값**: `.default()`로 누락된 필드에 기본값 제공
4. **Documentation**: Schema 자체가 "어떤 설정이 가능한가"의 문서

## Config Migration System

`src/migrations/`가 설정 형식 변경을 처리한다:

```
v1: { apiKey: "sk-ant-..." }
  ↓ migration
v2: { auth: { type: "apiKey", key: "sk-ant-..." } }
  ↓ migration
v3: { auth: { type: "apiKey", key: "sk-ant-...", provider: "direct" } }
```

각 마이그레이션은:
- **Version check**: 현재 config version 확인
- **Transform**: 이전 형식 → 현재 형식 변환
- **Validate**: Zod schema로 결과 검증
- **Save**: 마이그레이션된 config 저장

## Feature Flags: Build-Time Dead Code Elimination

`bun:bundle`의 `feature()` 함수로 build-time feature flags:

```typescript
import { feature } from 'bun:bundle'

if (feature('VOICE_MODE')) {
  // VOICE_MODE가 off면 이 코드 블록이 빌드에서 완전히 제거됨
  const voiceModule = require('./voice')
}
```

### Feature Flag 목록

| Flag | Feature | 상태 |
|------|---------|------|
| `PROACTIVE` | 자율 에이전트 모드 | Internal |
| `KAIROS` | Assistant mode | Internal |
| `BRIDGE_MODE` | IDE 통합 | 활성 |
| `DAEMON` | 백그라운드 데몬 | Internal |
| `VOICE_MODE` | 음성 입출력 | Internal |
| `AGENT_TRIGGERS` | 트리거 기반 에이전트 | Internal |
| `MONITOR_TOOL` | 모니터링 도구 | Internal |
| `COORDINATOR_MODE` | 다중 에이전트 | Internal |
| `WORKFLOW_SCRIPTS` | 워크플로우 자동화 | Internal |

### Runtime Flag vs Build-Time Flag

| | Runtime (env var) | Build-time (bun:bundle) |
|---|-------------------|------------------------|
| 코드 존재 | 항상 번들에 포함 | 비활성 시 제거 |
| 전환 | 재시작 시 변경 가능 | 재빌드 필요 |
| 성능 | 런타임 분기 비용 | 분기 자체가 없음 |
| 용도 | A/B 테스트, 점진적 롤아웃 | 미출시 기능, 내부 기능 |

**LLM 엔지니어링 인사이트**: Voice mode, coordinator mode 같은 대규모 기능은 수백 KB의 코드를 동반한다. Build-time flag로 제거하면 배포 바이너리가 가벼워지고, 시작 시간이 단축되며, 코드 경로가 단순해진다.

## Environment Variables

`.env.example`에 문서화된 환경 변수들의 우선순위:

```
1. 명시적 환경 변수 (ANTHROPIC_API_KEY=...)
2. Settings file (~/.claude/.claude.json)
3. Project settings (CLAUDE.md의 설정)
4. MDM policy (기업 관리 정책)
5. Default values (코드 내 기본값)
```

### 주요 환경 변수 그룹

| Group | Variables | 용도 |
|-------|-----------|------|
| **Auth** | `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` | 인증 |
| **Provider** | `CLAUDE_CODE_USE_BEDROCK/VERTEX/FOUNDRY` | 프로바이더 선택 |
| **Model** | `ANTHROPIC_MODEL`, `ANTHROPIC_SMALL_FAST_MODEL` | 모델 오버라이드 |
| **Retry** | `CLAUDE_CODE_MAX_RETRIES`, `CLAUDE_CODE_UNATTENDED_RETRY` | Retry 설정 |
| **URL** | `ANTHROPIC_BASE_URL` | API endpoint |
| **Headers** | `ANTHROPIC_CUSTOM_HEADERS` | 추가 헤더 |

## Enterprise MDM Policies

`src/services/remoteManagedSettings/`가 기업 관리 정책을 처리한다:

- **MDM (Mobile Device Management)**: 기업이 사원의 Claude Code 설정을 원격으로 관리
- **Policy limits**: 일일 토큰 한도, 허용 모델 목록, 허용 도구 목록
- **Remote settings sync**: 서버에서 push하는 설정을 로컬에 적용

## Settings Cache

`src/utils/settings/settingsCache.ts`:

- File watcher(`chokidar`)로 설정 파일 변경 감지
- 변경 시 캐시 무효화 → 다음 접근 시 재로드
- 다른 프로세스(예: IDE extension)가 설정을 변경해도 감지

## 내 프로젝트에 적용하기

### 설정 시스템 설계하기

**Step 1: Zod schema로 설정 정의**

설정 파일을 "just JSON"으로 읽지 말고, Zod schema로 정의한다. Runtime validation + TypeScript 타입 + 기본값을 한 번에 해결:

```typescript
import { z } from 'zod'

const configSchema = z.object({
  model: z.string().default('claude-sonnet-4-20250514'),
  maxTokens: z.number().min(1).max(100000).default(4096),
  temperature: z.number().min(0).max(1).default(0),
  apiProvider: z.enum(['direct', 'bedrock', 'vertex']).default('direct'),
  tools: z.object({
    enabled: z.array(z.string()).default([]),
    timeout: z.number().default(30000),
  }).default({}),
})

type Config = z.infer<typeof configSchema>  // TypeScript 타입 자동 생성

// 사용
const config = configSchema.parse(JSON.parse(rawJson))
// 잘못된 값이 있으면 여기서 즉시 에러 + 어떤 필드가 잘못됐는지 메시지
```

**Step 2: Config migration strategy**

설정 형식이 바뀔 때를 대비한 migration 체계:

```typescript
const migrations: Record<number, (config: any) => any> = {
  1: (c) => ({ ...c, version: 2, auth: { type: 'apiKey', key: c.apiKey } }),
  2: (c) => ({ ...c, version: 3, auth: { ...c.auth, provider: 'direct' } }),
}

function migrateConfig(config: any): Config {
  let current = config
  while (current.version < CURRENT_VERSION) {
    current = migrations[current.version](current)
  }
  return configSchema.parse(current)
}
```

**Step 3: 환경 변수 우선순위 chain**

```
ENV var > CLI flag > settings.json > project config > defaults
```

이 우선순위를 코드에 명시적으로 구현한다. `??` (nullish coalescing)을 chain으로 사용하면 깔끔:

```typescript
const model = process.env.MODEL ?? cliArgs.model ?? settings.model ?? projectConfig.model ?? 'claude-sonnet-4-20250514'
```

**Step 4: Re-entrancy guard**

Config 로딩 중 logger가 config를 참조하면 무한 루프가 된다. 간단한 guard:

```typescript
let isLoading = false
const getConfig = memoize(async () => {
  if (isLoading) return DEFAULT_CONFIG  // 순환 참조 방지
  isLoading = true
  try { /* load */ } finally { isLoading = false }
})
```

## 대안 비교

| 접근 방식 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **Zod schema** (Claude Code) | Validation + types + defaults 통합 | 의존성 추가 | TypeScript 프로젝트 |
| **JSON Schema + ajv** | 언어 무관, 표준 | TypeScript 타입 별도 관리 | Multi-language 환경 |
| **YAML + manual validation** | 읽기 쉬움, 주석 가능 | Validation 직접 구현 | DevOps 도구 |
| **TOML (pyproject.toml 스타일)** | 구조적, 읽기 쉬움 | 중첩 구조 불편 | Python 생태계 |

## Key Takeaways

- **Zod schemas = validation + types + docs**: 하나의 schema 정의에서 런타임 검증, TypeScript 타입, 문서 모두 파생.
- **Build-time feature flags**: 미출시 기능의 코드를 바이너리에서 완전히 제거. 번들 크기와 시작 시간 최적화.
- **Config migration**: 설정 형식 변경 시 자동 마이그레이션. 사용자가 수동으로 설정을 업데이트할 필요 없음.
- **Environment variable 우선순위**: 명확한 우선순위 chain으로 다양한 배포 환경(개인, 팀, 기업)을 지원.
- **Re-entrancy guard**: Config 로드 중 logging 시스템이 config를 접근하는 순환 참조 방지.
- **실전 적용 시**: 첫날부터 Zod schema + version 필드 + migration 함수를 넣어라. 나중에 설정 형식이 바뀔 때 (반드시 바뀐다) 사용자에게 수동 마이그레이션을 요구하지 않을 수 있다.
