<!--
tags: agents/tool-system, agents/tool-schema-design, agents/permission-model, agents/concurrent-execution, llm-application/tool-design, agents/sandbox, agents/subagent
keywords: tool-system, buildTool, tool-schema, zod-validation, permission-model, concurrent-execution, tool-registry, tool-presets, tool-use-context, self-contained-module, sandbox, fork-subagent, edit-after-read
related_files: 02_query_engine_core.md, 03_prompt_engineering.md, 08_streaming_concurrency.md, 09_security_model.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 04. Tool System

> Claude Code의 42개 tool 시스템: buildTool() 팩토리, Zod schema → JSON Schema 변환, permission 모델, concurrent execution, self-contained module 설계. 실제 구현 코드와 함께 "나만의 tool system 만들기"까지.

## Keywords
tool-system, buildTool, tool-factory, tool-schema, zod-to-json-schema, permission-model, concurrent-execution, tool-registry, feature-gating, tool-presets, tool-use-context, isConcurrencySafe, sandbox, fork-subagent, edit-after-read, fail-closed

---

## buildTool() Factory: 모든 Tool의 공통 계약

`src/Tool.ts` (~29K lines)이 모든 tool의 base type과 `buildTool()` factory를 정의한다. 모든 tool은 이 패턴을 따른다:

```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  aliases: ['my_tool'],
  description: 'What this tool does',

  inputSchema: z.object({
    param: z.string().describe('Parameter description'),
    optional_param: z.number().optional(),
  }),

  async call(args, context, canUseTool, parentMessage, onProgress) {
    // 실행 로직
    return { data: result, newMessages?: [...] }
  },

  async checkPermissions(input, context) {
    // Permission 체크 → { granted: boolean, reason?, prompt? }
  },

  isConcurrencySafe(input) {
    // 다른 tool과 병렬 실행 가능한가?
    return true
  },

  isReadOnly(input) {
    // 시스템 상태를 변경하지 않는가?
    return false
  },

  prompt(options) {
    // 시스템 프롬프트에 주입할 tool 사용 가이드
    return 'Use this tool when...'
  },

  renderToolUseMessage(input, options) {
    // Tool 호출 시 터미널에 표시할 UI
  },

  renderToolResultMessage(content, progressMessages, options) {
    // Tool 결과를 터미널에 표시할 UI
  },
})
```

**LLM 엔지니어링 인사이트**: `buildTool()` factory 패턴의 핵심은 **하나의 계약으로 모든 concern을 통합**한다는 것이다. Schema, permission, execution, UI가 한 곳에 정의되므로, 새 tool을 추가할 때 "다른 파일도 수정해야 하나?"라는 질문이 필요 없다.

### buildTool() 실제 구현: 3-Line Spread + Fail-Closed Defaults

실제 `buildTool()` 함수의 런타임 코드는 놀랍도록 간결하다 -- 단 3줄의 spread 패턴이다:

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,        // 1. 안전한 기본값으로 시작
    userFacingName: () => def.name,  // 2. name에서 파생
    ...def,                  // 3. 사용자 정의가 덮어씀
  } as BuiltTool<D>
}
```

핵심은 `TOOL_DEFAULTS`다. **Fail-closed philosophy** -- 명시적으로 override하지 않으면 가장 보수적인 동작을 한다:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,   // ← 병렬 실행 불가 (안전)
  isReadOnly: (_input?: unknown) => false,           // ← 쓰기 가능으로 간주 (안전)
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (
    input: { [key: string]: unknown },
    _ctx?: ToolUseContext,
  ): Promise<PermissionResult> =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',   // ← classifier 미사용 (안전)
  userFacingName: (_input?: unknown) => '',
}
```

**왜 이 디자인이 중요한가?**

| Default 값 | 의미 | 안전한 이유 |
|-------------|------|------------|
| `isConcurrencySafe → false` | 새 tool은 기본적으로 직렬 실행 | Race condition 방지. 검증 후에만 `true`로 override |
| `isReadOnly → false` | 새 tool은 기본적으로 write 가능으로 분류 | Permission 체크에서 더 엄격하게 처리됨 |
| `toAutoClassifierInput → ''` | Auto-approval classifier에 보내지 않음 | 보안 관련 tool은 반드시 명시적으로 override해야 함 |

**LLM 엔지니어링 인사이트**: Tool system을 설계할 때 default를 "최대 기능"이 아니라 **"최소 권한"**으로 설정하라. 개발자가 `isConcurrencySafe: true`를 깜빡 잊으면 성능만 약간 떨어지지만, `isConcurrencySafe: false`를 깜빡 잊으면 (즉 default가 true라면) race condition이 발생한다. **"느리지만 안전한 default > 빠르지만 위험한 default"**.

---

## Edit-after-Read 제약: Prompt + Code 이중 방어

Claude Code에서 가장 교묘한 안전장치 중 하나: **파일을 Read하지 않으면 Edit할 수 없다**.

### Prompt 레벨 강제

Edit tool의 description에 명시적으로 기재된다:

```
You must use your `Read` tool at least once in the conversation before editing.
This tool will error if you attempt an edit without reading the file.
```

### Code 레벨 강제

프롬프트만으로는 부족하다 -- LLM은 프롬프트를 "무시"할 수 있다. 따라서 **코드에서도 강제**한다. `readFileTimestamps` (ToolUseContext에 포함)를 체크하여, 해당 파일이 Read된 적 없으면 Edit을 거부한다.

```
Prompt 강제: "Read 먼저 해야 합니다"
    ↓
LLM이 프롬프트 무시 가능
    ↓
Code 강제: readFileTimestamps에 기록 없으면 → 에러 반환
    ↓
LLM에게 에러 메시지: "이 파일을 먼저 Read해야 합니다"
    ↓
LLM이 Read 호출 → 다시 Edit 시도 → 성공
```

### 왜 이 제약이 필요한가?

LLM은 **파일 내용을 "추측"하는 경향**이 강하다. 특히:

1. **Training data에서 본 코드**: "이 파일은 아마 이런 구조일 것이다"라고 가정하고 Edit을 시도
2. **이전 대화에서 본 내용이 오래된 경우**: 대화 초반에 Read한 파일이 이후 변경되었을 수 있음
3. **old_string 불일치**: 추측한 내용으로 string replacement를 시도하면, `old_string`이 실제 파일에 없어서 실패

**이중 방어 (Dual Enforcement) 패턴**은 LLM tool system의 일반 원칙이다:

| 강제 방법 | 장점 | 한계 |
|-----------|------|------|
| Prompt 강제 | LLM이 "왜" 해야 하는지 이해 | 무시 가능 |
| Code 강제 | 물리적으로 우회 불가 | LLM이 에러에서 학습해야 함 |
| **둘 다** | LLM이 이해하고 + 우회 불가 | 구현 복잡도 증가 |

**LLM 엔지니어링 인사이트**: 중요한 safety constraint는 항상 이중 방어로 구현하라. Prompt-only constraint는 LLM이 무시할 수 있고, code-only constraint는 LLM이 왜 실패했는지 이해하지 못해 반복 시도한다. 둘을 결합하면 LLM이 제약을 이해하면서도 물리적으로 우회할 수 없다.

---

## Fork Subagent Rules: "Don't Peek, Don't Race"

Agent tool은 fork subagent를 생성하여 독립적인 작업을 수행시킨다. 이때 핵심 규칙 두 가지가 있다:

### Rule 1: "Don't Peek" -- Fork 출력 파일을 절대 읽지 마라

```
**Don't peek.** Do not Read or tail the output_file unless the user explicitly asks.
```

Fork는 결과를 `output_file`에 기록하지만, parent agent가 이 파일을 직접 읽으면 안 된다. 이유:

- **Context pollution**: Fork의 전체 출력(수천 줄)을 parent context에 넣으면 context window가 낭비됨
- **설계 의도**: Fork는 `output_file`이 아닌 **요약 메시지**를 반환하도록 설계됨
- **사용자 투명성**: Agent 출력은 사용자에게 보이지 않으므로 → 반드시 명시적 summary를 텍스트로 전달해야 함

### Rule 2: "Don't Race" -- Fork 결과를 절대 예측하지 마라

```
**Don't race.** Never fabricate or predict fork results.
```

LLM은 fork가 무엇을 하는지 알고 있으므로, 결과를 "예측"하고 싶어 한다. 하지만:

- Fork가 **실패**했을 수 있음 (네트워크 에러, 파일 없음 등)
- Fork가 **다른 결과**를 냈을 수 있음 (코드 변경, 외부 상태 변경)
- 예측이 틀리면 **이후 작업이 모두 잘못됨** (cascading error)

### Fork 활용 가이드

```
Fork yourself (omit `subagent_type`) when the intermediate tool output
isn't worth keeping in your context.
- Research: fork open-ended questions. Launch parallel forks in one message.
- Implementation: prefer to fork implementation work that requires
  more than a couple of edits.

Forks are cheap because they share your prompt cache. Don't set `model`.
```

**LLM 엔지니어링 인사이트**: Subagent 패턴에서 가장 흔한 실수는 "parent가 child의 작업을 예측하는 것"이다. Prompt cache sharing 덕분에 fork는 저렴하므로, 의심스러우면 fork하고 결과를 기다리는 것이 항상 더 안전하다. "Agent output invisible to user → explicit summary required" 원칙도 중요 -- LLM의 내부 통신은 사용자에게 보이지 않으므로, 최종 결과는 반드시 명시적으로 전달해야 한다.

---

## Bash Tool Sandbox System: 파일시스템 + 네트워크 격리

Bash tool은 가장 위험한 tool이다 -- 임의의 시스템 명령을 실행할 수 있다. Claude Code는 sandbox 시스템으로 이를 제어한다.

### 파일시스템 제한

| 접근 유형 | 방식 | 예시 |
|-----------|------|------|
| **읽기** | Deny-list (차단 목록) | `~/.ssh/`, `~/.aws/credentials` 등 민감 경로 차단 |
| **쓰기** | Allow-list (허용 목록) | 프로젝트 디렉토리, `$TMPDIR`만 쓰기 허용 |

**읽기는 deny-list, 쓰기는 allow-list** -- 이 비대칭이 핵심이다:

- **읽기 deny-list**: 대부분의 파일은 읽어도 안전하므로, 위험한 것만 차단. 새로운 파일 경로가 추가되어도 기본적으로 읽기 가능.
- **쓰기 allow-list**: 임의의 경로에 쓰기는 위험하므로, 명시적으로 허용한 곳만 가능. 새로운 경로는 기본적으로 쓰기 불가.

### 네트워크 제한

```
네트워크 접근:
- Allow hosts: 명시적으로 허용된 호스트만 접근 가능
- Deny hosts: 특정 호스트 차단 (내부 네트워크 등)
```

### $TMPDIR 요구사항

```
임시 파일은 /tmp가 아니라 $TMPDIR를 사용해야 합니다.
```

왜 `/tmp`가 아닌 `$TMPDIR`인가?

- **sandbox 격리**: `$TMPDIR`은 sandbox가 관리하는 경로를 가리킴. `/tmp`를 직접 사용하면 sandbox를 우회할 수 있음.
- **macOS 호환성**: macOS에서 `/tmp`는 `/private/tmp`의 심볼릭 링크이고, sandbox profile이 이를 다르게 처리할 수 있음.
- **세션 격리**: `$TMPDIR`은 세션별로 다른 경로를 가리킬 수 있어, 여러 Claude Code 인스턴스가 충돌하지 않음.

### Sandbox 아키텍처

```
Bash 명령 실행 요청
    ↓
bashSecurity.ts (23개 보안 체크)
    ↓ PASS
Sandbox profile 적용
    ├── 읽기: deny-list 경로 차단
    ├── 쓰기: allow-list 경로만 허용
    └── 네트워크: allow/deny host 적용
    ↓
명령 실행 (격리된 환경)
    ↓
결과 반환 (stdout/stderr + exit code)
```

**LLM 엔지니어링 인사이트**: Shell tool의 sandbox는 "LLM이 실수해도 시스템이 안전한" 환경을 만든다. 핵심 원칙은 **읽기는 관대하게 (deny-list), 쓰기는 엄격하게 (allow-list)**. 이 비대칭은 LLM의 작업 패턴과도 일치한다 -- 탐색(읽기)은 자유롭게, 변경(쓰기)은 신중하게.

---

## ToolUseContext: Tool이 알아야 할 모든 것

Tool의 `call()` 메서드에 전달되는 `context` 객체는 tool이 필요로 하는 모든 런타임 정보를 포함한다:

| Field | 용도 |
|-------|------|
| `appState` | Global app state (conversation history, settings) |
| `sessionId` | 현재 세션 ID |
| `cwd` | Current working directory |
| `tools` | 사용 가능한 모든 tool 목록 |
| `abortSignal` | 취소 신호 (사용자가 Ctrl+C) |
| `readFileTimestamps` | 파일 읽기 타임스탬프 (Edit-after-Read 강제 + race condition 방지) |
| `options` | Tool-specific 옵션 |

이 context 패턴은 tool 간 암묵적 의존성을 방지한다. Tool은 global state에 직접 접근하지 않고, context를 통해서만 정보를 얻는다.

---

## Tool 디렉토리 구조: Self-Contained Modules

각 tool은 독립된 디렉토리를 가진다:

```
src/tools/BashTool/
├── BashTool.ts          # 핵심 실행 로직
├── UI.tsx               # Ink 기반 터미널 렌더링
├── prompt.ts            # 시스템 프롬프트 기여
├── bashSecurity.ts      # Bash-specific 보안 검증 (23개 체크)
├── utils.ts             # Tool-specific 유틸리티
└── index.ts             # Re-export
```

**왜 이 구조인가?**

- **Prompt는 tool과 함께 산다**: Tool의 기능이 바뀌면 prompt도 같이 업데이트해야 하므로, 같은 디렉토리에 둔다.
- **UI도 tool과 함께 산다**: Tool 결과의 렌더링 방식은 tool이 반환하는 데이터 구조에 의존하므로, 같은 디렉토리에 둔다.
- **Security check도 tool과 함께 산다**: BashTool의 보안 검증은 bash 명령어 구조를 이해해야 하므로, 범용 보안 모듈이 아닌 tool-specific 모듈이다.

---

## Tool Registry: Feature-Gated Discovery

`src/tools.ts`가 모든 tool을 등록한다. 핵심은 **feature flag와 환경 변수로 tool을 조건부 로드**한다는 점이다:

```typescript
// 항상 사용 가능
const FileReadTool = require('./tools/FileReadTool').FileReadTool
const BashTool = require('./tools/BashTool').BashTool

// Feature flag로 gating
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool').SleepTool
  : null

const MonitorTool = feature('MONITOR_TOOL')
  ? require('./tools/MonitorTool').MonitorTool
  : null

// Anthropic 내부 전용
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool').REPLTool
  : null
```

null check로 gating된 tool은 registry에 포함되지 않으므로, LLM이 해당 tool을 호출할 수 없다.

### Tool 카테고리

| Category | Tools | 특징 |
|----------|-------|------|
| **File I/O** | FileRead, FileWrite, FileEdit, Glob, Grep, NotebookEdit | 대부분 read-only, 높은 concurrency safety |
| **Shell** | Bash, PowerShell, REPL | Permission 필수, 보안 검증 복잡 |
| **Agent** | Agent, SendMessage, TeamCreate, TeamDelete | Sub-agent lifecycle 관리 |
| **Task** | TaskCreate, TaskUpdate, TaskGet, TaskList, TaskOutput, TaskStop | Background work 관리 |
| **Web** | WebFetch, WebSearch | External I/O, read-only |
| **MCP** | MCPTool, ListMcpResources, ReadMcpResource, McpAuth, ToolSearch | Protocol integration |
| **State** | EnterPlanMode, ExitPlanMode, EnterWorktree, ExitWorktree | Mode switching |
| **Scheduling** | ScheduleCron, RemoteTrigger | Async execution |
| **Utility** | AskUserQuestion, Brief, Config, Skill | Misc |

---

## Permission Model: UX로서의 보안

Tool permission은 단순한 보안 게이트가 아니라 **사용자 교육의 기회**다.

### Permission Flow

```
Tool 호출 → checkPermissions() → granted?
  YES → 즉시 실행
  NO  → 사용자에게 승인 요청 (이유와 함께)
    APPROVE → 실행 + 규칙에 기억 가능
    DENY    → LLM에게 거부 사실 반환 → 대안 시도
```

### Permission Modes

| Mode | 동작 | Use Case |
|------|------|----------|
| `default` | 위험한 작업마다 개별 승인 | 일반 사용 |
| `plan` | 전체 실행 계획을 보여주고 일괄 승인 | 복잡한 작업 |
| `bypassPermissions` | 모든 작업 자동 승인 | 신뢰 환경, CI/CD |
| `auto` | ML-based classifier가 자동 판단 | 실험적 |

### Permission Rules (Wildcard Pattern)

```
Bash(git *)           # git 명령어는 모두 허용
Bash(npm test)        # 특정 명령만 허용
FileEdit(/src/*)      # 특정 경로만 편집 허용
FileRead(*)           # 모든 파일 읽기 허용
```

**LLM 엔지니어링 인사이트**: Permission을 "security feature"로만 보면 UX가 나빠진다. Claude Code는 permission 요청을 "이 작업을 하려고 합니다, 괜찮으세요?"라는 **투명한 커뮤니케이션**으로 설계한다. 사용자가 거부하면 그 정보가 LLM에게 전달되어 대안을 찾는다.

---

## Concurrent Tool Execution

각 tool은 `isConcurrencySafe(input)` 메서드로 병렬 실행 가능 여부를 선언한다:

```typescript
// FileReadTool — 파일 읽기는 안전
isConcurrencySafe() { return true }

// BashTool — 명령어에 따라 다름
isConcurrencySafe(input) {
  return isReadOnlyCommand(input.command)
}

// FileWriteTool — 파일 쓰기는 불안전
isConcurrencySafe() { return false }
```

`StreamingToolExecutor`가 이 정보를 사용하여:
- **Concurrent-safe tools**: 병렬 실행
- **Non-concurrent tools**: Exclusive access (다른 tool 완료 후 실행)
- **결과 순서 보장**: 병렬 실행해도 결과는 호출 순서대로 반환

**LLM 엔지니어링 인사이트**: LLM이 한 번의 응답에서 여러 tool을 호출할 수 있다 (예: 3개 파일을 동시에 읽기). 이때 각 tool의 concurrency safety를 고려한 실행 전략이 필요하다. 무조건 직렬이면 느리고, 무조건 병렬이면 race condition이 발생한다.

---

## Zod Schema → JSON Schema → Prompt

Tool의 입력 정의가 3가지 목적으로 활용되는 "Schema as Single Source of Truth" 패턴:

```
Zod Schema (코드 작성 시)
    ↓
JSON Schema (API 전송 시 — tools 파라미터)
    ↓
Prompt Text (시스템 프롬프트에서 tool 설명)
```

1. **Zod**: TypeScript 타입 안전성 + runtime validation
2. **JSON Schema**: `toolToAPISchema()`가 Zod → JSON Schema 변환. API의 `tools` 파라미터로 전달
3. **Prompt**: JSON Schema가 LLM에게 tool의 입력 형식을 알려줌

**LLM 엔지니어링 인사이트**: Schema를 한 곳에서 정의하고 여러 곳에서 사용하면 불일치를 방지할 수 있다. Zod → JSON Schema → API 전달은 이미 검증된 패턴이다. Schema에 `.describe()`를 사용하면 LLM이 각 파라미터의 의미를 더 잘 이해한다.

---

## Tool Presets

`src/tools.ts`에서 tool들을 용도별로 그룹화한다:

- **Full toolset**: 모든 42개 tool (일반 사용)
- **Read-only tools**: FileRead, Glob, Grep 등 (code review, plan mode)
- **Minimal tools**: AskUserQuestion + Brief (제한된 환경)

이를 통해 **모드에 따라 LLM이 사용할 수 있는 tool을 제한**한다. Plan mode에서는 write tool이 비활성화되고, code review에서는 read-only tool만 활성화된다.

---

## Notable Tool Implementations

### FileEditTool: String Replacement 전략

파일 편집을 "전체 파일 덮어쓰기"가 아닌 **string replacement**로 구현한다:

```
old_string: "function hello() {"
new_string: "function hello(name: string) {"
```

이유: LLM이 전체 파일을 출력하면 token이 낭비되고, 실수로 다른 부분을 변경할 위험이 있다. String replacement는 최소한의 변경만 전달하므로 **token 효율적이고 안전하다**.

### AgentTool: Sub-Agent Spawning

새로운 agent를 생성하여 독립적인 작업을 수행시킨다. Agent는 자체 context window, tool set, conversation history를 가진다. Git worktree isolation으로 파일 시스템 충돌도 방지한다.

### ToolSearchTool: Deferred Tool Discovery

모든 tool을 미리 로드하면 시스템 프롬프트가 너무 커진다. `ToolSearchTool`은 MCP 서버의 tool을 on-demand로 검색하여, 필요할 때만 사용 가능하게 만든다. "Lazy loading for tools."

---

## 직접 만들어보기: Tool Factory

Claude Code의 tool system을 참고하여 유사한 시스템을 구축하는 pseudocode.

### Step 1: Tool 인터페이스 정의

```typescript
interface ToolDefinition<TInput, TOutput> {
  // === 필수: 정체성 ===
  name: string
  description: string
  inputSchema: ZodSchema<TInput>

  // === 필수: 실행 ===
  call(input: TInput, context: ToolContext): Promise<TOutput>

  // === 선택: Safety (fail-closed defaults) ===
  isConcurrencySafe?(input: TInput): boolean     // default: false
  isReadOnly?(input: TInput): boolean             // default: false
  checkPermissions?(input: TInput, ctx: ToolContext): Promise<PermissionResult>

  // === 선택: Prompt ===
  prompt?(options: PromptOptions): string
}
```

### Step 2: Fail-Closed Default가 적용되는 buildTool()

```typescript
const SAFE_DEFAULTS = {
  isConcurrencySafe: () => false,   // 안전: 직렬 실행
  isReadOnly: () => false,          // 안전: write로 간주 → 엄격한 permission
  checkPermissions: async (input) => ({ allowed: true, input }),
  prompt: () => '',
}

function buildTool<TInput, TOutput>(
  def: ToolDefinition<TInput, TOutput>
): CompleteTool<TInput, TOutput> {
  return {
    ...SAFE_DEFAULTS,               // 1. 안전한 기본값
    ...def,                         // 2. 사용자 정의가 덮어씀
  }
}
```

### Step 3: Schema Validation + Permission 파이프라인

```typescript
async function executeTool(
  tool: CompleteTool,
  rawInput: unknown,
  context: ToolContext
): Promise<ToolResult> {

  // 1. Schema validation (Zod)
  const parseResult = tool.inputSchema.safeParse(rawInput)
  if (!parseResult.success) {
    return { error: `Invalid input: ${parseResult.error.message}` }
  }
  const input = parseResult.data

  // 2. Permission check
  const permission = await tool.checkPermissions(input, context)
  if (!permission.allowed) {
    return { error: `Permission denied: ${permission.reason}` }
  }

  // 3. Execute
  try {
    const result = await tool.call(permission.input, context)
    return { data: result }
  } catch (e) {
    return { error: e.message }
  }
}
```

### Step 4: Concurrency-Aware Executor

```typescript
async function executeToolBatch(
  calls: ToolCall[],
  tools: Map<string, CompleteTool>,
  context: ToolContext
): Promise<ToolResult[]> {
  const results: ToolResult[] = new Array(calls.length)

  // 분류: concurrent-safe vs exclusive
  const concurrent: number[] = []
  const exclusive: number[] = []

  for (let i = 0; i < calls.length; i++) {
    const tool = tools.get(calls[i].name)!
    if (tool.isConcurrencySafe(calls[i].input)) {
      concurrent.push(i)
    } else {
      exclusive.push(i)
    }
  }

  // Concurrent-safe → 병렬 실행
  await Promise.all(
    concurrent.map(async (i) => {
      results[i] = await executeTool(
        tools.get(calls[i].name)!, calls[i].input, context
      )
    })
  )

  // Exclusive → 직렬 실행
  for (const i of exclusive) {
    results[i] = await executeTool(
      tools.get(calls[i].name)!, calls[i].input, context
    )
  }

  return results  // 원래 순서 유지
}
```

### Step 5: Registry + Feature Gating

```typescript
class ToolRegistry {
  private tools = new Map<string, CompleteTool>()

  register(tool: CompleteTool, gate?: () => boolean) {
    // Feature gate: gate()가 false면 등록하지 않음
    if (gate && !gate()) return
    this.tools.set(tool.name, tool)
  }

  // LLM에게 전달할 tool 목록 생성
  toAPITools(): APIToolSchema[] {
    return [...this.tools.values()]
      .filter(t => t.isEnabled())
      .map(t => ({
        name: t.name,
        description: t.description,
        input_schema: zodToJsonSchema(t.inputSchema),
      }))
  }

  // 모드별 subset
  getReadOnlyTools(): CompleteTool[] {
    return [...this.tools.values()].filter(t => t.isReadOnly())
  }
}
```

**LLM 엔지니어링 인사이트**: 이 5단계가 tool system의 최소 완성형이다. 핵심 원칙: (1) Fail-closed defaults, (2) Schema = single source of truth, (3) Permission은 tool과 executor 양쪽에서, (4) Concurrency는 tool이 self-declare, (5) Feature gating으로 동적 제어.

---

## 대안 비교: Centralized Registry vs Self-Contained Modules

Claude Code는 **self-contained module** 패턴을 선택했다. 이를 **centralized registry** 패턴과 비교한다.

### Centralized Registry 패턴

```
src/
├── tools/
│   ├── read.ts          # 실행 로직만
│   ├── write.ts
│   └── bash.ts
├── prompts/
│   ├── read.prompt.ts   # 프롬프트 별도
│   ├── write.prompt.ts
│   └── bash.prompt.ts
├── permissions/
│   ├── read.perm.ts     # 퍼미션 별도
│   ├── write.perm.ts
│   └── bash.perm.ts
├── schemas/
│   ├── read.schema.ts   # 스키마 별도
│   └── ...
└── registry.ts          # 중앙 등록
```

### Self-Contained Module 패턴 (Claude Code 선택)

```
src/tools/
├── FileReadTool/
│   ├── FileReadTool.ts  # 실행 + 스키마 + 퍼미션
│   ├── prompt.ts        # 프롬프트
│   ├── UI.tsx           # 렌더링
│   └── index.ts
├── BashTool/
│   ├── BashTool.ts
│   ├── prompt.ts
│   ├── bashSecurity.ts  # Tool-specific 보안
│   ├── UI.tsx
│   └── index.ts
└── ...
```

### 비교 분석

| 기준 | Centralized Registry | Self-Contained Module |
|------|---------------------|----------------------|
| **새 tool 추가** | 4-5개 파일 수정 (tool, prompt, permission, schema, registry) | 1개 디렉토리 추가 |
| **한 tool 수정** | 관련 파일 찾기 필요 | 디렉토리 하나만 보면 됨 |
| **Cross-cutting concern** | 쉬움 (한 곳에 모여 있으므로) | 어려움 (각 디렉토리를 순회해야) |
| **코드 재사용** | 공통 패턴이 자연스럽게 공유됨 | 공유 로직은 별도 모듈로 추출 필요 |
| **500+ 파일 규모** | 각 concern 디렉토리가 비대해짐. `prompts/` 안에 500개 파일 | tool별 디렉토리가 500개이지만 각각은 작음 |
| **삭제 시** | 여러 디렉토리에서 관련 파일 제거 필요 | 디렉토리 하나 삭제 |
| **Prompt-코드 동기화** | Prompt와 코드가 멀리 떨어져 있어 불일치 위험 | 같은 디렉토리에 있어 동기화 용이 |

### 스케일에서의 판단

**10개 이하의 tool**: 어느 패턴이든 상관없다.

**50개 이상의 tool**: Self-contained module이 유리해진다. 이유:
- **Locality of change**: 한 tool 수정 시 한 디렉토리만 건드림
- **Deletion safety**: 디렉토리 삭제 = tool 완전 제거 (dangling reference 없음)
- **Onboarding**: 새 개발자가 "BashTool을 이해하려면 BashTool/ 디렉토리만 보면 됨"

**Cross-cutting concern이 많은 경우**: Centralized가 유리할 수 있다. 예를 들어 "모든 tool의 permission 로직을 일괄 변경"은 centralized에서 더 쉽다. Claude Code는 이를 `buildTool()` factory의 `TOOL_DEFAULTS`로 해결한다 -- cross-cutting default를 한 곳에서 관리.

**LLM 엔지니어링 인사이트**: LLM tool system은 **prompt와 코드의 동기화**가 생명이다. Tool의 동작이 바뀌었는데 prompt가 안 바뀌면 LLM이 잘못된 방식으로 tool을 사용한다. Self-contained module은 이 동기화를 디렉토리 구조로 강제하므로, LLM 특화 시스템에서는 거의 항상 더 나은 선택이다.

---

## Key Takeaways

- **buildTool() factory**: 3-line spread pattern으로 구현. TOOL_DEFAULTS가 fail-closed philosophy를 적용하여 새 tool은 기본적으로 가장 안전한 동작.
- **Edit-after-Read 이중 방어**: Prompt + Code 양쪽에서 강제. LLM이 파일 내용을 추측하는 것을 방지. 중요한 constraint는 항상 dual enforcement.
- **Fork "Don't Peek, Don't Race"**: Subagent 결과를 예측하거나 output file을 직접 읽지 않음. Agent output은 사용자에게 보이지 않으므로 explicit summary 필수.
- **Bash Sandbox**: 읽기는 deny-list (관대), 쓰기는 allow-list (엄격). $TMPDIR로 세션 격리.
- **Self-contained modules**: Tool = 디렉토리. Prompt, UI, security가 모두 tool과 함께 산다. 500+ 규모에서도 locality of change 유지.
- **Feature-gated registry**: Feature flag + env var로 tool 가용성 제어. Null check로 안전하게 비활성화.
- **Permission as UX**: 보안 게이트가 아니라 투명한 커뮤니케이션. 거부 시 LLM에게 반환하여 대안 탐색.
- **Concurrency declaration**: 각 tool이 병렬 실행 가능 여부를 self-declare. Default가 false이므로 "깜빡 잊어도 안전".
- **Schema as Single Source of Truth**: Zod → JSON Schema → Prompt. 하나의 정의에서 validation, API, LLM 설명 모두 파생.
- **Tool Factory 구축**: 5단계 (인터페이스 → buildTool → validation pipeline → concurrency executor → registry)로 유사 시스템 구축 가능.
