<!--
tags: architecture/streaming, agents/concurrent-execution, architecture/event-driven, architecture/batching, llm-application/streaming
keywords: streaming, concurrent-tool-execution, StreamingToolExecutor, event-driven, progress-messages, SerialBatchEventUploader, HybridTransport, backpressure, content-block-delta
related_files: 02_query_engine_core.md, 04_tool_system.md, 07_retry_resilience.md, 13_ui_terminal.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 08. Streaming & Concurrency

> 실시간 LLM 응답 처리와 concurrent tool execution: StreamingToolExecutor, event-driven architecture, batching, backpressure.

## Keywords
streaming, concurrent-tool-execution, StreamingToolExecutor, event-driven, content-block-delta, progress-messages, backpressure, SerialBatchEventUploader, HybridTransport, stream-buffering, parallel-tools

## Streaming Architecture: 왜 Streaming이 필수인가

LLM 응답은 수 초에서 수십 초가 걸린다. Streaming 없이 전체 응답을 기다리면:
- 사용자는 "멈춘 건가?" 불안감을 느낌
- 에러 발생 시 이미 소모한 token을 되돌릴 수 없음
- Tool call이 포함된 긴 응답에서 전체 완료 전에 tool 실행을 시작할 수 없음

Claude Code는 **모든 API 호출을 streaming**으로 처리한다.

### Stream Event Processing

Anthropic SDK의 `Stream<BetaRawMessageStreamEvent>` 타입을 사용:

```
message_start
  ↓
content_block_start (type: "text")
  → content_block_delta × N (text 조각)
  → content_block_stop
  ↓
content_block_start (type: "tool_use")
  → content_block_delta × N (tool input JSON 조각)
  → content_block_stop
  ↓
content_block_start (type: "thinking")
  → content_block_delta × N (thinking 내용)
  → content_block_stop
  ↓
message_delta (stop_reason, usage)
  ↓
message_stop
```

### Text vs Tool Use 처리 차이

| 이벤트 | 처리 |
|--------|------|
| **Text delta** | 즉시 터미널에 렌더링 (character by character) |
| **Tool use delta** | JSON 버퍼에 축적, content_block_stop 후 실행 |
| **Thinking delta** | 설정에 따라 표시 또는 숨김, budget 추적 |

**LLM 엔지니어링 인사이트**: Tool input JSON이 streaming으로 조각조각 도착하는 것이 streaming + tool use의 가장 까다로운 부분이다. `{"path": "/Us`... `ers/file.ts"}` 이런 식으로 나뉘어 도착하므로, 완성된 JSON을 파싱하기 전까지 tool 실행을 시작할 수 없다. **반드시 content_block_stop을 기다려야** 한다.

## StreamingToolExecutor: Concurrent Tool Execution

`src/services/tools/StreamingToolExecutor.ts`가 streaming 중 tool 실행을 관리한다.

### 핵심 알고리즘

```
LLM 응답에서 tool_use block 1개 완성됨
  → StreamingToolExecutor.addTool(toolUseBlock) 호출
  ↓
isConcurrencySafe(input) 확인
  → true: 기존 실행 중인 tool과 병렬로 즉시 시작
  → false: 다른 모든 tool 완료 후 exclusive하게 실행
  ↓
실행 완료 → 결과 버퍼에 저장
  ↓
모든 tool 완료 → 결과를 호출 순서대로 반환
```

### canExecuteTool 실제 구현 로직

`canExecuteTool()`은 tool 실행 가능 여부를 판단하는 핵심 게이트다. 실제 코드에서 추출한 정확한 3가지 조건:

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||                                    // 조건 1
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))  // 조건 2
  )
}
```

| 조건 | 의미 |
|------|------|
| **`executingTools.length === 0`** | 실행 중인 tool이 하나도 없으면 무조건 실행 가능 |
| **`isConcurrencySafe`** | 새로 실행하려는 tool 자체가 concurrent-safe |
| **`executingTools.every(t => t.isConcurrencySafe)`** | 현재 실행 중인 모든 tool도 concurrent-safe |

세 조건이 AND로 결합 (조건 2, 3): **새 tool도 safe, 기존 tool들도 모두 safe일 때만 병렬 실행**. 하나라도 exclusive tool이 실행 중이면 새 tool은 대기한다.

**상태 전이 시나리오**:

```
시나리오 1: [Read A 실행중] + Read B 도착
  → A는 safe, B도 safe → 병렬 실행 ✓

시나리오 2: [Read A 실행중] + Write C 도착
  → C는 NOT safe → 대기 (A 완료 후 실행)

시나리오 3: [Write C 실행중] + Read D 도착
  → executingTools에 NOT safe인 C가 있음 → D는 대기

시나리오 4: [] + Write C 도착
  → executingTools.length === 0 → 무조건 실행 ✓
```

### Concurrency Safety Examples

```typescript
// 동시 실행 가능 — 읽기 전용, 서로 독립
FileReadTool.isConcurrencySafe()  → true   // 파일 읽기
GlobTool.isConcurrencySafe()      → true   // 파일 검색
GrepTool.isConcurrencySafe()      → true   // 내용 검색

// 조건부 — 입력에 따라 다름
BashTool.isConcurrencySafe(input) → isReadOnlyCommand(input.command)
  // "git log" → true (읽기 전용)
  // "npm install" → false (시스템 변경)

// 항상 exclusive — 파일 시스템 변경
FileWriteTool.isConcurrencySafe() → false
FileEditTool.isConcurrencySafe()  → false
```

### 결과 순서 보장

병렬 실행해도 **결과는 호출 순서대로** LLM에게 반환된다. 이유: LLM은 tool call의 순서에 의미를 부여할 수 있으므로, 결과 순서가 바뀌면 혼란을 일으킬 수 있다.

```
Tool calls: [read A, read B, read C]  (병렬 실행)
완료 순서:  [B완료, C완료, A완료]      (실제)
반환 순서:  [A결과, B결과, C결과]      (호출 순서대로)
```

**LLM 엔지니어링 인사이트**: "병렬 실행 + 순서 보장"은 LLM tool 실행에서 흔히 놓치는 요구사항이다. 구현은 간단하다: 결과를 Map에 저장하고, 모든 tool 완료 후 호출 순서대로 꺼내면 된다.

### Error Cascading: Bash만 특별하다

흥미로운 설계 결정: **Bash tool의 에러만** sibling tool들을 취소한다. FileRead, WebFetch 등 다른 tool의 에러는 cascade하지 않는다. 이유: bash 명령어는 종종 암묵적 의존 관계가 있지만 (예: `mkdir dir && cd dir`), 파일 읽기나 웹 검색은 서로 독립적이다.

실제 구현에서는 `siblingAbortController`라는 별도의 abort controller가 이를 관리한다. Bash error 발생 시 이 controller를 abort하면, 같은 controller를 공유하는 sibling tool들의 subprocess가 즉시 종료된다. 중요한 점: 이 abort는 **parent query controller에는 전파되지 않는다** — 즉, turn 자체는 끝나지 않고, 에러 결과만 LLM에게 돌아간다.

### Interrupt Behavior

각 tool은 `interruptBehavior()`를 선언한다: `'cancel'` 또는 `'block'` (기본값). 사용자가 새 메시지를 입력하면 `'cancel'` tool만 중단되고, `'block'` tool은 완료될 때까지 기다린다.

---

## 직접 만들어보기: Streaming Tool Executor Pseudocode

아래는 Claude Code의 StreamingToolExecutor 핵심 동작을 재현하는 완전한 pseudocode다. concurrent-safe/exclusive 구분, 결과 순서 보장, abort cascading을 모두 포함한다.

```python
class StreamingToolExecutor:
    """LLM streaming 중 tool을 즉시 실행하되, concurrency safety를 보장한다."""

    def __init__(self, tool_definitions, abort_controller):
        self.tools: list[TrackedTool] = []         # 등록 순서 유지
        self.tool_definitions = tool_definitions
        self.parent_abort = abort_controller
        self.sibling_abort = ChildAbortController(parent=abort_controller)
        self.has_errored = False

    # ── Step 1: Tool 등록 (content_block_stop 시 호출) ──
    def add_tool(self, tool_use_block):
        definition = find_tool(self.tool_definitions, tool_use_block.name)
        if definition is None:
            # 알 수 없는 tool → 즉시 에러 결과로 완료 처리
            self.tools.append(TrackedTool(
                block=tool_use_block,
                status='completed',
                is_concurrent_safe=True,  # 다른 tool blocking 방지
                results=[error_message(f"No such tool: {tool_use_block.name}")]
            ))
            return

        # Input validation → concurrency safety 결정
        parsed = definition.input_schema.safe_parse(tool_use_block.input)
        is_safe = (
            parsed.success and
            definition.is_concurrency_safe(parsed.data)
        )  # 실패 시 False (안전 쪽으로 default)

        self.tools.append(TrackedTool(
            block=tool_use_block,
            status='queued',
            is_concurrent_safe=is_safe,
            results=[]
        ))
        self.process_queue()  # 즉시 실행 시도

    # ── Step 2: 실행 가능 여부 판단 (핵심 게이트) ──
    def can_execute(self, is_concurrent_safe: bool) -> bool:
        executing = [t for t in self.tools if t.status == 'executing']
        return (
            len(executing) == 0                                     # 아무것도 실행 중이 아님
            or (
                is_concurrent_safe                                  # 새 tool이 safe
                and all(t.is_concurrent_safe for t in executing)    # 기존 tool들도 모두 safe
            )
        )

    # ── Step 3: Queue 처리 ──
    def process_queue(self):
        for tool in self.tools:
            if tool.status != 'queued':
                continue
            if self.can_execute(tool.is_concurrent_safe):
                self.execute_tool(tool)            # 비동기 실행 시작 (await 안함)
            elif not tool.is_concurrent_safe:
                break  # exclusive tool이 대기 중이면, 뒤의 tool도 실행 불가

    # ── Step 4: Tool 실행 + 에러 cascading ──
    async def execute_tool(self, tool):
        tool.status = 'executing'
        tool_abort = ChildAbortController(parent=self.sibling_abort)

        try:
            # abort 이미 발생했으면 synthetic error
            if self.has_errored or self.parent_abort.is_aborted:
                tool.results = [synthetic_error(tool, reason='sibling_error')]
                tool.status = 'completed'
                return

            async for update in run_tool(tool.block, abort=tool_abort):
                # 실행 중 abort 체크
                if self.has_errored:
                    tool.results.append(synthetic_error(tool, 'sibling_error'))
                    break

                if update.is_error and tool.block.name == 'Bash':
                    # ★ Bash만 sibling cascade
                    self.has_errored = True
                    self.sibling_abort.abort('sibling_error')

                tool.results.append(update.message)

        finally:
            tool.status = 'completed'
            self.process_queue()  # ★ 완료 후 대기 중인 tool 깨우기

    # ── Step 5: 결과 반환 (호출 순서 보장) ──
    async def get_ordered_results(self) -> list[Message]:
        """모든 tool 완료 후, 등록 순서대로 결과 반환."""
        await asyncio.gather(*[t.promise for t in self.tools if t.promise])

        ordered = []
        for tool in self.tools:          # self.tools는 등록 순서
            ordered.extend(tool.results)  # 각 tool의 결과를 순서대로 이어붙임
        return ordered
```

**구현 핵심 포인트**:
- `process_queue()`는 addTool 시점과 executeTool 완료 시점 **양쪽에서** 호출된다. 이로써 exclusive tool 완료 후 대기 중이던 tool이 자동으로 시작된다.
- `sibling_abort`는 parent abort의 **child**다. Bash error로 sibling을 죽여도 parent(query loop)는 영향받지 않는다.
- Queue iteration에서 exclusive tool을 만나면 `break` — 그 뒤의 concurrent-safe tool도 대기시킨다 (순서 보장).

---

## 대안 비교: Sequential-Only vs Concurrent with Declaration

### 방식 1: Sequential-Only (가장 단순)

```python
for tool_call in llm_response.tool_calls:
    result = await execute_tool(tool_call)    # 하나씩 순차 실행
    results.append(result)
```

**장점**: 구현 극도로 단순, 상태 관리 불필요, race condition 없음
**단점**: 독립적인 읽기 작업도 직렬화

### 방식 2: Concurrent with Declaration (Claude Code 방식)

```python
for tool_call in llm_response.tool_calls:
    if can_execute(tool_call.is_concurrent_safe):
        start_async(tool_call)   # 즉시 병렬 실행
    else:
        await all_executing()    # 기존 완료 대기
        start_async(tool_call)   # 단독 실행
results = await get_ordered_results()
```

**장점**: 읽기 작업 병렬화, 쓰기 작업은 안전하게 직렬화
**단점**: 구현 복잡도 증가, 결과 순서 보장 필요

### 방식 3: Full Concurrent (위험)

```python
results = await asyncio.gather(*[execute_tool(t) for t in tool_calls])
```

**장점**: 최대 병렬성
**단점**: Write 후 Read 순서가 보장되지 않아 **데이터 불일치 발생 가능**

### 구체적 시간 비교

**시나리오**: LLM이 3개 파일을 동시에 읽으라고 요청 (각 300ms)

```
방식 1 (Sequential):
  Read A ─────── Read B ─────── Read C ───────
  [0ms          300ms          600ms          900ms]
  총 소요: 900ms

방식 2 (Concurrent w/ Declaration):
  Read A ───────┐
  Read B ───────┤ (모두 concurrent-safe → 병렬)
  Read C ───────┘
  [0ms          300ms]
  총 소요: 300ms  ← 3배 빠름

혼합 시나리오: [Read A, Write B, Read C]
  방식 1: 900ms (300 + 300 + 300)
  방식 2: 900ms (Read A 300ms → Write B 300ms → Read C 300ms)
    → Write가 끼면 이점이 사라짐. 하지만 실전에서 LLM은 읽기를 몰아서 요청하는 경향이 있다.

실전 시나리오: 코드베이스 탐색 (5개 파일 동시 읽기, 각 200ms)
  방식 1: 1000ms
  방식 2: 200ms  ← 5배 빠름
```

**결론**: LLM agent에서 tool call의 70% 이상이 읽기 작업이므로, concurrent declaration 방식의 실질적 효과가 크다.

---

## 내 프로젝트에 적용하기: Sequential에서 Concurrent로 3단계 전환

### Step 1: Tool에 concurrency safety 선언 추가

기존 tool 정의에 `is_concurrency_safe` 메서드를 추가한다. **보수적으로 시작**: 확실한 읽기 전용만 True.

```python
class Tool(ABC):
    @abstractmethod
    def execute(self, input): ...

    def is_concurrency_safe(self, input) -> bool:
        return False  # 기본값: exclusive (안전 쪽 default)

class FileReadTool(Tool):
    def is_concurrency_safe(self, input) -> bool:
        return True   # 파일 읽기는 항상 safe

class BashTool(Tool):
    def is_concurrency_safe(self, input) -> bool:
        return is_read_only_command(input['command'])  # 조건부

class FileWriteTool(Tool):
    def is_concurrency_safe(self, input) -> bool:
        return False  # 쓰기는 항상 exclusive
```

**핵심**: `is_concurrency_safe`의 기본값은 **반드시 False**여야 한다. 새 tool을 추가할 때 선언을 빠뜨리면 자동으로 exclusive 실행되어 안전하다.

### Step 2: Tool executor에 concurrency gate 추가

기존 sequential executor를 수정:

```python
# Before: 단순 sequential
for tool in tool_calls:
    result = await execute(tool)

# After: concurrency gate 추가
executor = StreamingToolExecutor(tool_definitions)
for tool in tool_calls:
    executor.add_tool(tool)       # 내부에서 can_execute 판단 후 즉시/대기

results = await executor.get_ordered_results()
```

### Step 3: Error cascading과 abort 연결

마지막으로 Bash-only error cascading을 추가한다. 이 단계는 선택적이지만, 없으면 실패한 `mkdir` 뒤에 관련 없는 `cd`가 실행되는 문제가 생긴다.

```python
# Error cascading 규칙 정의
CASCADING_TOOLS = {'Bash'}  # Bash만 sibling cancel

# execute_tool 내부에서:
if is_error and tool.name in CASCADING_TOOLS:
    sibling_abort.abort()  # 같은 batch의 다른 tool 취소
```

**단계별 적용 팁**:
- Step 1만으로도 안전하게 배포 가능 (선언만 추가, 실행은 그대로 sequential)
- Step 2에서 실제 병렬화 시작 — 성능 개선 즉시 체감
- Step 3은 Bash/shell tool이 있는 경우에만 필요

---

## Progress Messages: 긴 작업의 사용자 경험

Tool 실행이 오래 걸릴 때 `onProgress` 콜백으로 중간 상태를 알린다:

```typescript
async call(args, context, canUseTool, parentMessage, onProgress) {
  onProgress({ type: "progress", message: "파일 검색 중..." })
  // ... 실행 ...
  onProgress({ type: "progress", message: "500개 파일 검색 완료, 결과 정리 중..." })
  // ... 정리 ...
  return { data: result }
}
```

Progress messages는:
- **즉시 yield**: 터미널에 바로 표시
- **Non-blocking**: Tool 실행을 중단하지 않음
- **Compact에서 제거**: Context 압축 시 progress messages는 삭제 (최종 결과만 보존)

## SerialBatchEventUploader: 대량 이벤트 효율적 전송

`src/cli/transports/SerialBatchEventUploader.ts`가 대량의 이벤트를 효율적으로 업로드한다:

### 설정

```typescript
{
  maxBatchSize: 500,       // 배치당 최대 항목 수
  maxBatchBytes: undefined, // 배치당 최대 바이트 (선택)
  maxQueueSize: 1000,      // 큐 최대 크기 (backpressure)
  baseDelayMs: 500,        // 재시도 기본 지연
  maxDelayMs: 8000,        // 재시도 최대 지연
  jitterMs: 1000,          // Thundering herd 방지 jitter
  maxConsecutiveFailures: 5, // 연속 실패 시 배치 드롭
}
```

### 동작 원리

```
이벤트 발생 → 큐에 추가
  ↓
큐에 배치 크기만큼 쌓이면 → POST 전송
  ↓
성공 → 다음 배치
실패 → exponential backoff + jitter → 재시도
  → 연속 실패 N회 → 배치 드롭 (데이터 손실 감수)
  ↓
큐가 maxQueueSize 도달 → backpressure (enqueue 블록)
```

**핵심 패턴**:
- **Serial ordered**: 한 번에 하나의 POST만 in-flight
- **Thundering herd 방지**: Random jitter로 여러 클라이언트가 동시에 재시도하는 것 방지
- **Graceful degradation**: 연속 실패 시 데이터를 드롭하지만 앱은 계속 동작

## HybridTransport: 읽기/쓰기 분리

`src/cli/transports/HybridTransport.ts`는 WebSocket과 HTTP를 조합한다:

```
읽기 (서버 → 클라이언트): WebSocket
  → 실시간 stream event 수신
  → 연결 유지, latency 최소

쓰기 (클라이언트 → 서버): HTTP POST (SerialBatchEventUploader)
  → 대량 이벤트 배치 전송
  → 100ms 배치 (content delta를 묶어서)
  → POST 수 감소
```

**왜 이 조합인가**: WebSocket은 서버→클라이언트 실시간 전송에 최적이지만, 클라이언트→서버 대량 전송에서는 HTTP POST의 배치 처리가 더 효율적이다.

### Stream Event Buffering

Content delta 이벤트는 **100ms 단위로 배치**된다:

```
delta 1 (0ms) → 버퍼에
delta 2 (20ms) → 버퍼에
delta 3 (50ms) → 버퍼에
flush (100ms) → [delta 1, 2, 3]을 한 번의 POST로 전송
```

이렇게 하면 개별 delta마다 POST하는 것보다 네트워크 효율이 크게 향상된다.

## Abort Handling: 사용자 취소

사용자가 Ctrl+C를 누르면:

1. `AbortSignal`이 모든 실행 중인 tool에 전파
2. Tool의 `call()` 메서드가 signal을 확인하고 조기 종료
3. Streaming connection 닫기
4. 부분 결과가 있으면 저장 (세션 복구용)
5. UI에 취소 상태 표시

```typescript
// Tool 내부에서 abort 확인
async call(args, context) {
  for (const file of files) {
    if (context.abortSignal.aborted) break
    // ... 처리 ...
  }
}
```

## Key Takeaways

- **모든 API 호출은 Streaming**: 사용자 경험과 tool 실행 속도 모두를 위해 필수.
- **Tool input buffering**: JSON이 완성될 때까지(content_block_stop) tool 실행을 시작하지 않는다.
- **canExecuteTool 3조건**: (1) 실행 중 tool 없음, OR (2) 새 tool이 safe AND 기존 모두 safe.
- **Concurrency declaration**: 각 tool이 `isConcurrencySafe()`를 선언하여 실행 엔진이 안전하게 병렬화. 기본값은 반드시 False.
- **결과 순서 보장**: 병렬 실행해도 LLM에게 반환하는 결과는 호출 순서대로.
- **Bash-only error cascade**: `siblingAbortController`로 Bash 에러만 sibling을 취소. Parent query는 영향 없음.
- **Progress messages**: 긴 작업의 중간 상태를 즉시 표시하되, context 압축 시 제거.
- **읽기/쓰기 분리**: WebSocket(실시간 읽기) + HTTP POST(배치 쓰기)의 하이브리드 전송.
- **Sequential→Concurrent 전환**: (1) safety 선언 추가 → (2) executor gate → (3) error cascade. 단계별로 안전하게 적용 가능.
