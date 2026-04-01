<!--
tags: agents/tool-loop, architecture/streaming, architecture/query-engine, context-management/token-counting, llm-application/core-engine
keywords: query-engine, tool-loop, streaming, turn-management, token-budget, agentic-loop, tool-use, message-normalization, conversation-management
related_files: 03_prompt_engineering.md, 04_tool_system.md, 05_context_management.md, 07_retry_resilience.md, 08_streaming_concurrency.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 02. Query Engine Core

> Claude Code의 심장부 — QueryEngine.ts (~46K lines). LLM API 호출, streaming 처리, tool execution loop, token 관리를 모두 담당하는 핵심 엔진.

## Keywords
query-engine, tool-loop, agentic-loop, streaming-response, turn-management, token-budget, message-normalization, tool-use-block, content-block, context-window, conversation-history

## Tool Loop: 에이전트의 본질

모든 에이전트 앱의 핵심은 **tool loop**다. Claude Code의 구현은 이렇다:

```
query() 호출
  ↓
messages 배열 구성 (conversation history + new user message)
  ↓
while (true) {
  API 호출 (streaming) → response 수신

  response에서 tool_use block 발견?
    YES → tool 실행 → 결과를 messages에 추가 → continue
    NO  → 최종 text 응답 → break
}
```

**핵심 설계**: 이 루프는 LLM이 스스로 "도구가 더 필요 없다"고 판단할 때만 종료된다. 선택적으로 `maxTurns`와 `maxBudgetUsd`로 제한할 수 있고, `taskBudget`으로 sub-task별 token 소비를 제한할 수도 있다.

### QueryEngine의 핵심 메서드: `submitMessage()`

`async *submitMessage(prompt, options?)` — **AsyncGenerator<SDKMessage>**:

```typescript
// QueryEngineConfig의 핵심 필드들
{
  tools: Tools,                    // 사용 가능한 tool 목록
  commands: Command[],             // slash commands
  mcpClients: MCPServerConnection[], // MCP 서버 연결
  canUseTool: CanUseToolFn,        // permission 체크 함수
  initialMessages?: Message[],     // 이전 대화 히스토리
  thinkingConfig?: ThinkingConfig, // extended thinking 설정
  maxTurns?: number,               // 최대 턴 수 제한
  maxBudgetUsd?: number,           // 비용 제한
  taskBudget?: { total: number },  // sub-task 토큰 예산
  jsonSchema?: Record<string, unknown>, // structured output 스키마
  fallbackModel?: string,          // Opus → Sonnet fallback
}
```

**비동기 제너레이터 패턴**: `submitMessage()`는 일반 함수가 아니라 `AsyncGenerator`다. 각 메시지(assistant response, tool result, compact boundary)가 생성될 때마다 `yield`하여, 호출자가 실시간으로 진행 상황을 받아볼 수 있다.

`query.ts` 파일이 진입점이고, 실제 로직은 `QueryEngine.ts`에 있다. 이 분리는 initialization context(어떤 tool이 사용 가능한지, 어떤 model을 쓰는지)와 execution logic을 분리하기 위함이다.

### Tool Loop의 세부 흐름

1. **Tool Use Detection**: Streaming response에서 `content_block_start` 이벤트의 `type: "tool_use"`를 감지
2. **Input Assembly**: `content_block_delta`로 tool 입력 JSON이 조각조각 들어옴 → 버퍼에 축적
3. **content_block_stop**: 완성된 tool 입력으로 실행 시작
4. **Permission Check**: `checkPermissions(input, context)` 호출 — 위험한 작업이면 사용자 승인 요청
5. **Execution**: Tool의 `call()` 메서드 실행
6. **Result Formatting**: 실행 결과를 `tool_result` content block으로 포맷
7. **Re-submission**: tool_result를 messages에 추가하고 다시 API 호출

**LLM 엔지니어링 인사이트**: Tool 입력이 streaming으로 조각조각 도착하는 것을 처리하는 것이 생각보다 복잡하다. JSON이 완성되기 전에 실행을 시작할 수 없으므로 반드시 buffering이 필요하다. Claude Code는 `content_block_stop` 이벤트를 기다린 후에만 실행을 시작한다.

---

## 직접 만들어보기: Tool Loop 완전 가이드

이 섹션은 Claude Code의 `query.ts`에서 추출한 **실제 state machine**을 language-agnostic pseudocode로 재구성한 것이다. 이것만 이해하면 Python이든 TypeScript든 production-grade tool loop를 구현할 수 있다.

### State Machine: 5단계 루프

Claude Code의 tool loop는 단순한 `while(tool_use)` 가 아니다. 실제로는 5개의 명확한 phase를 가진 **state machine**이다:

```
┌─────────────────────────────────────────────────────────────────┐
│                         LOOP START                              │
│  state = { messages, turnCount, recoveryCount, ... }            │
└─────────────────┬───────────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  [1] MESSAGE_PREP                                               │
│  - context compaction (auto/reactive/snip)                      │
│  - token budget check (blocking limit?)                         │
│  - message normalization for API                                │
│  - system prompt assembly                                       │
└─────────────────┬───────────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  [2] MODEL_CALL                                                 │
│  - streaming API call (with fallback model retry)               │
│  - collect assistant messages + tool_use blocks                 │
│  - streaming tool execution (start tools before stream ends)    │
│  - withhold recoverable errors (PTL, max_output_tokens)         │
└─────────────────┬───────────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  [3] POST_STREAM                                                │
│  - abort check → yield interruption, return                     │
│  - if NO tool_use detected → go to EXIT_DECISION               │
│  - if tool_use detected → go to TOOL_EXECUTION                  │
└─────────┬───────────────────────────────────┬───────────────────┘
          ▼                                   ▼
┌───────────────────────┐   ┌─────────────────────────────────────┐
│  EXIT_DECISION        │   │  [4] TOOL_EXECUTION                 │
│                       │   │  - run tools (parallel or streaming) │
│  Recovery checks:     │   │  - yield results to caller           │
│  - PTL → reactive     │   │  - abort check                      │
│    compact → retry    │   │  - permission hook check             │
│  - max_output_tokens  │   └─────────────────┬───────────────────┘
│    → escalate (8k→64k)│                     ▼
│    → recovery message │   ┌─────────────────────────────────────┐
│    → or surface error │   │  [5] LOOP_CONTINUATION              │
│  - stop hooks         │   │  - gather attachments (memory, etc) │
│  - token budget check │   │  - maxTurns check                   │
│  - return 'completed' │   │  - state = { messages + results }   │
└───────────────────────┘   │  - continue → back to [1]           │
                            └─────────────────────────────────────┘
```

### Pseudocode: 완전한 Tool Loop

아래는 `query.ts`의 ~1700줄을 핵심 로직만 추출한 pseudocode다. 실제 구현 시 이 구조를 따르면 된다:

```python
# Language-agnostic pseudocode — Python 스타일로 작성
# 실제 Claude Code의 query.ts에서 추출

async def query_loop(params):
    """Production-grade agentic tool loop."""

    # ── Immutable params (루프 전체에서 변경되지 않음) ──
    system_prompt = params.system_prompt
    tools = params.tools
    can_use_tool = params.can_use_tool
    max_turns = params.max_turns
    fallback_model = params.fallback_model

    # ── Mutable state (매 iteration마다 갱신) ──
    state = {
        "messages": params.messages,          # 대화 히스토리 (mutable array)
        "turn_count": 1,
        "max_output_recovery_count": 0,       # max_output_tokens 복구 시도 횟수
        "has_attempted_reactive_compact": False,
        "max_output_override": None,          # 8k→64k 에스컬레이션
        "transition": None,                   # 이전 iteration의 continue 사유
    }

    while True:
        messages = state["messages"]
        turn_count = state["turn_count"]

        # ═══════════════════════════════════════════════
        # [1] MESSAGE_PREP: Context 관리 + 메시지 준비
        # ═══════════════════════════════════════════════

        messages_for_query = get_messages_after_compact_boundary(messages)

        # Tool result size 제한 (큰 결과를 truncate)
        messages_for_query = apply_tool_result_budget(messages_for_query)

        # Snip compaction (오래된 중간 부분 제거)
        messages_for_query = snip_compact_if_needed(messages_for_query)

        # Microcompact (서버 캐시 기반 압축)
        messages_for_query = microcompact(messages_for_query)

        # Auto-compact (threshold 초과 시 요약으로 압축)
        compact_result = auto_compact(messages_for_query, system_prompt)
        if compact_result:
            messages_for_query = compact_result.post_compact_messages
            for msg in messages_for_query:
                yield msg  # caller에게 compact 결과 전달

        # Blocking limit 체크 (overflow 시 즉시 종료)
        if is_at_blocking_limit(messages_for_query):
            yield error_message("Prompt too long")
            return {"reason": "blocking_limit"}

        # ═══════════════════════════════════════════════
        # [2] MODEL_CALL: Streaming API 호출
        # ═══════════════════════════════════════════════

        assistant_messages = []
        tool_results = []
        tool_use_blocks = []
        needs_follow_up = False

        # Fallback retry loop (Opus → Sonnet)
        attempt_with_fallback = True
        while attempt_with_fallback:
            attempt_with_fallback = False
            try:
                async for message in call_model_streaming(
                    messages=messages_for_query,
                    system_prompt=system_prompt,
                    tools=tools,
                    model=current_model,
                    max_output_tokens=state["max_output_override"],
                ):
                    # 복구 가능한 에러는 withhold (즉시 yield하지 않음)
                    withheld = False
                    if is_prompt_too_long(message):
                        withheld = True
                    if is_max_output_tokens(message):
                        withheld = True

                    if not withheld:
                        yield message  # caller에게 실시간 전달

                    if message.type == "assistant":
                        assistant_messages.append(message)
                        # tool_use block 수집
                        for block in message.content:
                            if block.type == "tool_use":
                                tool_use_blocks.append(block)
                                needs_follow_up = True

                                # [핵심] Streaming tool execution:
                                # stream이 끝나기 전에 tool 실행 시작
                                if streaming_executor:
                                    streaming_executor.add_tool(block)

                    # Streaming 중 완료된 tool 결과 수확
                    if streaming_executor:
                        for result in streaming_executor.get_completed():
                            yield result
                            tool_results.append(result)

            except FallbackTriggeredError:
                if fallback_model:
                    current_model = fallback_model
                    attempt_with_fallback = True
                    # 이전 결과 전부 폐기
                    assistant_messages.clear()
                    tool_results.clear()
                    tool_use_blocks.clear()
                    yield system_message(f"Switched to {fallback_model}")
                    continue
                raise

        # ═══════════════════════════════════════════════
        # [3] POST_STREAM: Abort 체크 + 분기
        # ═══════════════════════════════════════════════

        if abort_signal.aborted:
            yield interruption_message()
            return {"reason": "aborted"}

        # ═══════════════════════════════════════════════
        # EXIT_DECISION: Tool 없으면 종료 판단
        # ═══════════════════════════════════════════════

        if not needs_follow_up:
            last_message = assistant_messages[-1] if assistant_messages else None

            # ── Recovery Path 1: Prompt Too Long (PTL) ──
            if is_prompt_too_long(last_message):
                # 먼저 context collapse drain 시도
                if state["transition"] != "collapse_drain_retry":
                    drained = recover_from_overflow(messages_for_query)
                    if drained.committed > 0:
                        state = {**state,
                            "messages": drained.messages,
                            "transition": "collapse_drain_retry",
                        }
                        continue  # [1]로 돌아감

                # 그 다음 reactive compact 시도
                if not state["has_attempted_reactive_compact"]:
                    compacted = try_reactive_compact(messages_for_query)
                    if compacted:
                        state = {**state,
                            "messages": compacted.post_compact_messages,
                            "has_attempted_reactive_compact": True,
                            "transition": "reactive_compact_retry",
                        }
                        continue  # [1]로 돌아감

                # 복구 실패 → 에러 표시
                yield last_message
                return {"reason": "prompt_too_long"}

            # ── Recovery Path 2: Max Output Tokens ──
            if is_max_output_tokens(last_message):
                # 2a: 에스컬레이션 (8k → 64k, 1회만)
                if state["max_output_override"] is None:
                    state = {**state,
                        "max_output_override": 64000,  # ESCALATED_MAX_TOKENS
                        "transition": "max_output_tokens_escalate",
                    }
                    continue  # 같은 요청을 더 큰 limit으로 재시도

                # 2b: Multi-turn recovery (최대 3회)
                if state["max_output_recovery_count"] < 3:
                    recovery_msg = create_user_message(
                        "Output token limit hit. Resume directly — "
                        "no apology, no recap. Pick up mid-thought."
                    )
                    state = {**state,
                        "messages": [*messages_for_query,
                                     *assistant_messages, recovery_msg],
                        "max_output_recovery_count":
                            state["max_output_recovery_count"] + 1,
                        "max_output_override": None,
                        "transition": "max_output_tokens_recovery",
                    }
                    continue  # [1]로 돌아감

                # 복구 소진 → 에러 표시
                yield last_message

            # Stop hooks 실행
            stop_result = handle_stop_hooks(messages_for_query, assistant_messages)
            if stop_result.blocking_errors:
                state = {**state,
                    "messages": [*messages_for_query,
                                 *assistant_messages,
                                 *stop_result.blocking_errors],
                    "transition": "stop_hook_blocking",
                }
                continue  # hook이 에러 주입 → 모델이 수정하도록 재시도

            # Token budget continuation 체크
            if should_continue_for_budget():
                state = {**state,
                    "messages": [*messages_for_query,
                                 *assistant_messages, nudge_message],
                    "transition": "token_budget_continuation",
                }
                continue

            return {"reason": "completed"}

        # ═══════════════════════════════════════════════
        # [4] TOOL_EXECUTION: Tool 실행
        # ═══════════════════════════════════════════════

        if streaming_executor:
            # Streaming executor에서 나머지 결과 수확
            tool_updates = streaming_executor.get_remaining_results()
        else:
            # 일반 실행 (병렬)
            tool_updates = run_tools(tool_use_blocks, can_use_tool)

        for update in tool_updates:
            if update.message:
                yield update.message
                tool_results.append(update.message)
            if update.should_prevent_continuation:
                return {"reason": "hook_stopped"}

        if abort_signal.aborted:
            yield interruption_message()
            return {"reason": "aborted_tools"}

        # ═══════════════════════════════════════════════
        # [5] LOOP_CONTINUATION: 다음 턴 준비
        # ═══════════════════════════════════════════════

        # Attachment 수집 (memory, file changes, queued commands)
        for attachment in get_attachment_messages(messages_for_query):
            yield attachment
            tool_results.append(attachment)

        # MaxTurns 체크
        next_turn = turn_count + 1
        if max_turns and next_turn > max_turns:
            yield max_turns_reached_message()
            return {"reason": "max_turns"}

        # State 갱신 → while(true) 상단으로
        state = {
            "messages": [*messages_for_query,
                         *assistant_messages, *tool_results],
            "turn_count": next_turn,
            "max_output_recovery_count": 0,
            "has_attempted_reactive_compact": False,
            "max_output_override": None,
            "transition": "next_turn",
        }
        # continue → [1] MESSAGE_PREP로 돌아감
```

### 왜 이 구조가 중요한가

위 pseudocode에서 주목할 점들:

1. **`state` 객체를 통한 루프 제어**: `continue`할 때마다 `state`를 새로 할당한다. `transition` 필드가 "왜 다시 돌아왔는지"를 기록하므로, 무한 루프 디버깅이 쉬워진다. 실제 코드에서도 `state.transition.reason`으로 recovery path를 추적한다.

2. **Error를 withhold하는 패턴**: prompt-too-long이나 max_output_tokens 에러를 즉시 yield하지 않고 **보류**한다. 복구에 성공하면 에러를 삼키고, 실패하면 그때 yield한다. 이것이 없으면 SDK consumer(Desktop 앱 등)가 중간 에러를 보고 세션을 종료해버린다.

3. **Multi-phase recovery**: PTL 에러 하나에도 3단계 복구가 있다 — context collapse drain → reactive compact → error surface. 각 단계는 독립적이고 순서대로 시도된다.

---

## 실제 코드 발췌: query.ts 핵심 구조

아래는 `query.ts`의 실제 구조를 simplified하되 정확하게 보여주는 TypeScript 발췌다:

```typescript
// query.ts — 실제 코드에서 핵심만 추출 (간소화했으나 구조는 정확)

// Mutable state carried between loop iterations
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  turnCount: number
  transition: Continue | undefined  // 왜 이전 iteration이 continue했는지
}

async function* queryLoop(params: QueryParams): AsyncGenerator<...> {
  // Immutable params — 루프 중 절대 변경 안 됨
  const { systemPrompt, canUseTool, fallbackModel, maxTurns } = params

  // Mutable state — 매 continue마다 새 객체로 교체
  let state: State = {
    messages: params.messages,
    turnCount: 1,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    // ...
    transition: undefined,
  }

  while (true) {
    // 매 iteration 시작: state에서 destructure
    const { messages, turnCount, maxOutputTokensRecoveryCount } = state

    // [1] MESSAGE_PREP
    let messagesForQuery = getMessagesAfterCompactBoundary(messages)
    messagesForQuery = await applyToolResultBudget(messagesForQuery, ...)
    // snip → microcompact → autocompact 순서로 적용
    const { compactionResult } = await deps.autocompact(messagesForQuery, ...)
    if (compactionResult) {
      messagesForQuery = buildPostCompactMessages(compactionResult)
      for (const msg of messagesForQuery) yield msg
    }

    // [2] MODEL_CALL
    const assistantMessages: AssistantMessage[] = []
    const toolUseBlocks: ToolUseBlock[] = []
    let needsFollowUp = false

    for await (const message of deps.callModel({ messages: messagesForQuery, ... })) {
      // withheld 에러 체크 (PTL, max_output_tokens)
      let withheld = false
      if (isPromptTooLongMessage(message)) withheld = true
      if (isWithheldMaxOutputTokens(message)) withheld = true
      if (!withheld) yield message

      if (message.type === 'assistant') {
        assistantMessages.push(message)
        const blocks = message.message.content.filter(c => c.type === 'tool_use')
        if (blocks.length > 0) {
          toolUseBlocks.push(...blocks)
          needsFollowUp = true
        }
      }
    }

    // [3] POST_STREAM
    if (toolUseContext.abortController.signal.aborted) {
      yield createUserInterruptionMessage()
      return { reason: 'aborted_streaming' }
    }

    if (!needsFollowUp) {
      // EXIT_DECISION: recovery 경로들...
      const lastMessage = assistantMessages.at(-1)

      // PTL recovery → reactive compact
      if (isWithheld413) {
        const compacted = await reactiveCompact.tryReactiveCompact(...)
        if (compacted) {
          state = { ...state, messages: compacted, transition: { reason: 'reactive_compact_retry' } }
          continue  // ← 루프 상단으로
        }
        yield lastMessage  // 복구 실패, 에러 표시
        return { reason: 'prompt_too_long' }
      }

      // Max output tokens recovery
      if (isWithheldMaxOutputTokens(lastMessage)) {
        if (maxOutputTokensRecoveryCount < 3) {
          state = {
            ...state,
            messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
            maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
            transition: { reason: 'max_output_tokens_recovery' },
          }
          continue
        }
        yield lastMessage
      }

      return { reason: 'completed' }
    }

    // [4] TOOL_EXECUTION
    for await (const update of runTools(toolUseBlocks, ...)) {
      if (update.message) {
        yield update.message
        toolResults.push(update.message)
      }
    }

    // [5] LOOP_CONTINUATION
    if (maxTurns && turnCount + 1 > maxTurns) {
      return { reason: 'max_turns' }
    }

    state = {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      turnCount: turnCount + 1,
      maxOutputTokensRecoveryCount: 0,
      hasAttemptedReactiveCompact: false,
      transition: { reason: 'next_turn' },
    }
  } // while (true)
}
```

**코드에서 배울 점**: `state` 객체를 매번 새로 생성(`state = { ... }`)하는 패턴에 주목하라. 9개의 개별 변수를 각각 재할당하는 대신 하나의 객체로 묶으면, 실수로 하나를 빠뜨려 stale state로 돌아가는 버그를 방지할 수 있다. 이것은 실제 코드 주석에도 명시되어 있다: *"Continue sites write `state = { ... }` instead of 9 separate assignments."*

---

## Streaming Architecture: 이벤트 기반 응답 처리

Claude Code는 Anthropic SDK의 `Stream<BetaRawMessageStreamEvent>`를 사용한다. 주요 이벤트 타입:

| Event | 의미 | 처리 |
|-------|------|------|
| `message_start` | 새 메시지 시작 | Usage 정보 초기화 |
| `content_block_start` | 새 content block (text/tool_use/thinking) | Block 타입 식별 |
| `content_block_delta` | Content 조각 도착 | 실시간 렌더링 또는 버퍼링 |
| `content_block_stop` | Block 완성 | Tool이면 실행 시작 |
| `message_delta` | Message-level 업데이트 | stop_reason, usage 추적 |
| `message_stop` | 메시지 완료 | Turn 종료 |

### Thinking Blocks 처리

Extended thinking은 `BetaThinkingBlock`으로 도착한다:

- `ThinkingConfig`로 활성화/비활성화 + budget (token 단위) 설정
- Thinking content는 사용자에게 표시하거나 숨길 수 있음
- Redacted thinking 지원 (Anthropic이 내부적으로 thinking 내용을 redact하는 경우)
- Thinking budget은 요청 간에 추적됨

**Thinking block의 3가지 규칙** (실제 코드 주석에서 발견):

1. Thinking 또는 redacted_thinking block을 포함한 메시지는 `max_thinking_length > 0`인 쿼리에 속해야 한다
2. Thinking block이 메시지의 마지막 block이 되어서는 안 된다
3. Thinking block은 assistant trajectory(하나의 turn, 또는 tool_use를 포함하면 후속 tool_result와 다음 assistant message까지) 동안 보존되어야 한다

> *"Heed these rules well, young wizard. For they are the rules of thinking, and the rules of thinking are the rules of the universe. If ye does not heed these rules, ye will be punished with an entire day of debugging and hair pulling."* — 실제 query.ts 주석

**LLM 엔지니어링 인사이트**: Thinking blocks는 tool 호출 결정의 질을 높이지만, token budget을 소모한다. Claude Code는 thinking budget을 별도로 관리하여 실제 작업에 쓸 수 있는 output token이 thinking에 의해 고갈되지 않도록 한다.

---

## 대안 비교: 왜 이 아키텍처인가?

### Streaming + Tool Loop vs. Batch API Calls

| 관점 | Streaming + Tool Loop | Batch (동기식) |
|------|----------------------|----------------|
| **UX 응답성** | 첫 토큰이 수백ms 내에 표시 | 전체 응답 완료까지 아무것도 표시 안 됨 |
| **Tool 실행 타이밍** | Stream 도중에 tool 실행 시작 가능 (`StreamingToolExecutor`) | 응답 완료 후에만 실행 가능 |
| **에러 복구** | Stream 중간에 fallback 모델 전환 가능 | 전체 요청 재시도 필요 |
| **메모리** | 메시지를 조각조각 처리 → 피크 메모리 낮음 | 전체 응답을 메모리에 보관 |
| **복잡도** | 높음 (event parsing, buffering, state machine) | 낮음 (request-response) |
| **Abort** | 즉각적인 중단 가능 (`AbortController`) | 요청 완료까지 대기 필요 |

**결론**: Production agent에서 streaming은 선택이 아니라 필수다. 사용자가 10초 이상 빈 화면을 볼 수 있는 시스템은 실전에서 사용되지 않는다. Claude Code의 `StreamingToolExecutor`는 여기서 한 발 더 나아가, **stream이 끝나기도 전에 이미 완성된 tool block의 실행을 시작**한다.

### AsyncGenerator vs. Callback

| 관점 | AsyncGenerator (`yield`) | Callback (`onMessage(msg)`) |
|------|--------------------------|----------------------------|
| **제어 흐름** | 호출자가 `for await`로 pull | Producer가 push |
| **Backpressure** | 자연스럽게 내장 (yield가 consumer를 기다림) | 별도 구현 필요 (queue, buffering) |
| **취소** | `.return()`으로 generator 종료 | Callback 해제 로직 필요 |
| **합성** | `yield*`로 sub-generator 위임 | Callback 중첩 → callback hell |
| **에러 전파** | `throw`가 generator를 통해 전파 | Try-catch가 안 먹힘 |
| **테스트** | `const results = []; for await (const r of gen) results.push(r)` | Mock callback 세팅 필요 |

**결론**: Claude Code가 `async *submitMessage()`, `async *queryLoop()`를 모두 AsyncGenerator로 구현한 이유가 여기 있다. `yield*`를 통한 위임(`yield* queryLoop(params)`)으로 `query()` → `queryLoop()`의 계층 구조가 깔끔해진다. Callback 패턴이었다면 이 수준의 합성은 불가능에 가깝다.

### Mutable Messages Array vs. Immutable

```
// Claude Code의 실제 패턴: Mutable array + 새 State 객체
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  // ↑ 매 continue마다 새 array 생성
  transition: { reason: 'next_turn' },
}
```

| 관점 | Mutable Array (Claude Code) | Immutable (매번 복사) |
|------|---------------------------|---------------------|
| **성능** | O(1) push, 전체 history를 매번 복사하지 않음 | O(n) 매 turn마다 전체 복사 |
| **메모리** | History가 수천 개 메시지일 때 효율적 | 긴 대화에서 GC 압박 |
| **안전성** | 실수로 중간에 mutation하면 버그 | 참조 투명성 보장 |
| **디버깅** | State snapshot이 없어 추적 어려움 | 매 시점의 state 확인 가능 |

**Claude Code의 타협**: Messages array 자체는 매 continue에서 spread(`[...old, ...new]`)로 새로 만들지만, 개별 message 객체는 mutation하지 않는다. 이것은 **shallow copy** 패턴으로, 성능과 안전성의 균형점이다. 완전한 deep copy는 수천 개 메시지에서 비현실적이고, 완전한 mutation은 prompt caching을 깨뜨린다 (바이트가 달라지면 캐시 miss).

---

## Turn Management: 대화 턴 구조

Claude Code의 대화는 명확한 턴 구조를 갖는다:

```
Turn 1: [user message] → [assistant response (text + tool calls)]
Turn 2: [tool results] → [assistant response (text + more tool calls)]
Turn 3: [tool results] → [assistant response (final text)]
```

**Message normalization** (`src/utils/messages.ts`)이 핵심 역할을 한다:

- `normalizeMessagesForAPI()`: 내부 메시지 형식을 API 형식으로 변환
- `normalizeContentFromAPI()`: API 응답을 내부 형식으로 변환
- Advisor blocks, tool reference blocks를 API 전송 시 제거
- System reminder tags를 적절한 위치에 주입

### Message 타입의 다양성

단순한 user/assistant 외에도 다양한 내부 메시지 타입이 있다:

| Type | 용도 |
|------|------|
| `UserMessage` | 사용자 입력 |
| `AssistantMessage` | LLM 응답 |
| `ToolUseMessage` | Tool 호출 요청 |
| `ToolResultMessage` | Tool 실행 결과 |
| `ProgressMessage` | 장시간 작업 중간 상태 |
| `CompactBoundaryMessage` | Context 압축 경계 마커 |
| `SystemAPIErrorMessage` | API 오류 (retry 중 heartbeat) |

## Token Budget Management

QueryEngine은 매 turn마다 token 사용량을 추적한다:

### 토큰 카운팅 전략

1. **API 응답 기반 (정확)**: `getTokenUsage(message)`로 API response의 `usage` 필드에서 정확한 수치 추출
   - `input_tokens`: 입력 토큰
   - `cache_creation_input_tokens`: 캐시 생성에 쓴 토큰
   - `cache_read_input_tokens`: 캐시에서 읽은 토큰
   - `output_tokens`: 출력 토큰

2. **사전 추정 (빠른)**: `roughTokenCountEstimationForMessages()`로 API 호출 전에 대략적 추정
   - 문자열 길이 기반 heuristic
   - Tool schema 크기 고려
   - 시스템 프롬프트 크기 고려

### Context Window Overflow 처리

Token이 context window 한계에 도달하면:

1. **Compact mode** 트리거 → 이전 대화를 요약으로 압축 [상세 → 05_context_management.md]
2. **Microcompact**: API-side context management with `cache_edits`
3. 최악의 경우: `max tokens exceeded` 에러 → retry logic에서 context 축소 후 재시도

**LLM 엔지니어링 인사이트**: Token counting은 LLM 앱에서 가장 과소평가되는 기능이다. "동작은 하는데 가끔 실패한다"의 대부분은 context overflow가 원인이다. Claude Code처럼 **매 turn마다** 사용량을 추적하고, overflow 전에 proactive하게 압축하는 것이 핵심이다.

## Retry Logic Within the Engine

QueryEngine 내부의 retry는 API-level retry (`withRetry.ts`)와 별개로 동작한다:

- **Tool 실행 실패**: 에러 메시지를 tool_result로 포맷하여 LLM에게 반환 → LLM이 다른 접근법 시도
- **Streaming 중단**: Connection 재설정 후 마지막 완성된 message부터 재시도
- **Context overflow**: Compact 실행 후 재시도
- **Model fallback**: 특정 조건에서 Opus → Sonnet으로 자동 전환 [상세 → 07_retry_resilience.md]

**핵심 패턴**: Tool 실행 에러를 앱 수준에서 처리하지 않고 **LLM에게 돌려보내는 것**이 에이전트 아키텍처의 핵심이다. LLM은 에러 메시지를 읽고 다른 전략을 시도할 수 있다.

## API 호출 구성

QueryEngine이 API에 보내는 요청의 구조:

```typescript
{
  model: "claude-opus-4-6",           // 또는 환경변수로 오버라이드
  // (모델 ID는 시점에 따라 다를 수 있음 — API 문서에서 최신 ID 확인)
  max_tokens: 16384,                   // 모델별 기본값
  system: [...systemPromptBlocks],     // 계층적으로 조립된 시스템 프롬프트
  messages: [...conversationHistory],  // 정규화된 대화 히스토리
  tools: [...toolSchemas],             // 42개 tool의 JSON Schema
  stream: true,                        // 항상 streaming
  metadata: { user_id: "..." },        // 사용자 식별
  // Beta headers:
  betas: [
    "context-management",              // Microcompact
    "fast-mode",                       // Fast inference
    "prompt-caching-scope",            // Cache scope
    "structured-outputs",              // JSON validation
    "task-budgets",                    // Task spending limits
  ]
}
```

## Server-Side Tool Loops

Claude Code는 **server-side tool loops**도 지원한다. 이는 일부 tool 실행이 API 서버에서 직접 이루어지는 패턴이다. `finalContextTokensFromLastResponse()`는 서버에서 실행된 tool loop의 token 사용량까지 정확하게 추적한다.

---

## Crash Resilience: Transcript 선 저장과 --resume

### 설계 결정

QueryEngine은 **API 호출 전에** transcript를 디스크에 저장한다:

```typescript
// API 호출 전에 transcript 저장
// 이유: 프로세스가 mid-request에서 죽어도 --resume으로 복구 가능
await persistTranscript(messages)
```

### 왜 "전"에 저장해야 하는가: 실패 시나리오

이 패턴의 중요성을 이해하려면 실패 시나리오를 생각해보자:

```
시나리오: API 호출 "후"에 저장하는 경우

1. User: "이 프로젝트의 모든 테스트를 수정해줘"
2. Agent: ReadFile → 10개 파일 읽기 (Turn 1-10)
3. Agent: EditFile → 5개 파일 수정 완료 (Turn 11-15)
4. Agent: API 호출 시작 (Turn 16) ← streaming 응답 수신 중...
5. 💥 프로세스 crash (OOM, 전원 차단, Ctrl+C 등)
6. Transcript: Turn 15까지만 저장됨

결과: --resume 시 Turn 16의 API 호출 결과를 잃어버림.
      하지만 15개 turn의 작업은 보존됨. 수용 가능.
```

```
시나리오: API 호출 "후"에도 저장을 안 하는 경우 (naive 구현)

1. 동일한 1-15 turn 실행
2. 💥 프로세스 crash
3. Transcript: 아예 없거나 세션 시작 시점만 저장

결과: 15개 turn의 모든 작업 기록 소실. --resume 불가능.
      사용자가 처음부터 다시 시작해야 함.
```

**핵심 원리**: 이것은 database의 **Write-Ahead Logging (WAL)** 패턴과 동일하다. "작업을 하기 전에 의도를 기록한다." API 호출은 수십 초가 걸릴 수 있고, 그 사이에 프로세스가 죽을 확률이 가장 높다. 따라서 호출 전에 현재까지의 상태를 저장하면, 최악의 경우에도 마지막 API 호출만 재실행하면 된다.

**구현 팁**: `recordTranscript()`는 `sessionStorage.ts`에서 세션 파일에 기록한다. Agent ID가 있으면 sidechain 파일에, 없으면 메인 세션 파일에 저장한다. 이 분리가 sub-agent의 독립적 resume을 가능하게 한다.

---

## Parallel Tool Call의 Message Split 패턴

### 문제

LLM이 하나의 응답에서 여러 tool을 동시에 호출할 때 (예: "3개 파일을 동시에 읽어라"), API 응답은 하나의 assistant message에 3개의 `tool_use` block이 들어온다. 하지만 각 tool의 결과는 별도의 `tool_result` message로 돌려야 한다.

### Claude Code의 해결: Sibling Message ID 패턴

Streaming 코드가 **하나의 assistant message를 여러 개의 sibling record로 분리**한다:

```
API 응답 (하나의 assistant message):
  assistant(id=A) {
    tool_use_1: ReadFile("src/a.ts")
    tool_use_2: ReadFile("src/b.ts")
    tool_use_3: ReadFile("src/c.ts")
  }

내부 messages 배열 (분리됨):
  [...,
    assistant(id=A, content=[tool_use_1]),
    user(tool_result_1),                    // ← interleaved
    assistant(id=A, content=[tool_use_2]),
    user(tool_result_2),
    assistant(id=A, content=[tool_use_3]),
    user(tool_result_3)
  ]
```

같은 `message.id = A`를 공유하는 이 "sibling" records가 핵심이다.

### Token Counting의 특별 처리

이 분리 때문에 token counting이 복잡해진다:

```
문제: assistant(id=A, tool_use_2)의 token 사용량은 얼마인가?

naive 접근: 이 record만의 input_tokens를 읽는다
           → 잘못됨. 이 record에는 tool_use_2만 있지만,
             실제 API 호출에는 tool_use_1의 결과도 포함되어 있었다.

Claude Code 접근: tokenCountWithEstimation()이
  같은 id의 첫 sibling까지 거슬러 올라가서
  모든 interleaved tool_results를 포함하여 계산한다.
```

**구현 시 주의점**: Parallel tool call을 지원하려면, messages array에서 같은 `message.id`를 가진 모든 record를 "하나의 논리적 turn"으로 취급하는 로직이 필요하다. 이것을 빠뜨리면 token counting이 실제보다 적게 나와서 context overflow가 발생한다.

---

## 내 프로젝트에 적용하기: Step-by-Step 구현 가이드

이 섹션은 위의 분석을 바탕으로, production-grade tool loop를 **처음부터** 만드는 단계별 가이드다.

### Step 1: 최소한의 Tool Loop (MVP)

먼저 동작하는 최소 구현부터 시작하라. 모든 고급 기능은 나중에 추가한다.

```python
# Step 1: 가장 단순한 tool loop (30줄)

import anthropic

client = anthropic.Anthropic()

def run_agent(user_message: str, tools: list[dict], tool_handlers: dict):
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        # Assistant 응답을 history에 추가
        messages.append({"role": "assistant", "content": response.content})

        # Tool use가 없으면 종료
        tool_blocks = [b for b in response.content if b.type == "tool_use"]
        if not tool_blocks:
            return response.content  # 최종 텍스트 응답

        # Tool 실행 + 결과 추가
        tool_results = []
        for block in tool_blocks:
            try:
                result = tool_handlers[block.name](block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result),
                })
            except Exception as e:
                # [핵심] 에러를 LLM에게 돌려보내기
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": f"Error: {e}",
                    "is_error": True,
                })

        messages.append({"role": "user", "content": tool_results})
```

이것만으로도 동작하는 agent다. 하지만 production에서는 부족하다.

### Step 2: Streaming 추가

```python
# Step 2: Streaming으로 전환

async def run_agent_streaming(user_message, tools, tool_handlers):
    messages = [{"role": "user", "content": user_message}]

    while True:
        # Streaming API 호출
        assistant_content = []
        tool_blocks = []

        # 참고: 아래는 간소화된 pseudocode. 실제 async 환경에서는
        # `async with client.messages.stream()` 사용.
        with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        ) as stream:
            current_tool_buffer = None  # tool input 버퍼링용

            for event in stream:
                if event.type == "content_block_start":
                    if event.content_block.type == "text":
                        pass  # text block 시작
                    elif event.content_block.type == "tool_use":
                        # tool block 시작 — input 버퍼링 초기화
                        current_tool_buffer = {
                            "id": event.content_block.id,
                            "name": event.content_block.name,
                            "input_json": "",  # 조각을 여기에 축적
                        }

                elif event.type == "content_block_delta":
                    if hasattr(event.delta, "text"):
                        print(event.delta.text, end="", flush=True)
                        # ↑ 실시간 텍스트 출력
                    elif hasattr(event.delta, "partial_json"):
                        # tool input JSON 조각 도착 → 버퍼에 축적
                        if current_tool_buffer:
                            current_tool_buffer["input_json"] += event.delta.partial_json

                elif event.type == "content_block_stop":
                    # block 완성 — tool이면 JSON 파싱 후 tool_use block 생성
                    if current_tool_buffer:
                        import json
                        parsed_input = json.loads(current_tool_buffer["input_json"])
                        tool_blocks.append({
                            "type": "tool_use",
                            "id": current_tool_buffer["id"],
                            "name": current_tool_buffer["name"],
                            "input": parsed_input,
                        })
                        current_tool_buffer = None

            # Stream 완료 후 최종 메시지 수집
            final_message = stream.get_final_message()

        messages.append({"role": "assistant", "content": final_message.content})

        tool_blocks = [b for b in final_message.content if b.type == "tool_use"]
        if not tool_blocks:
            return final_message.content

        # Tool 실행 (Step 1과 동일)
        tool_results = execute_tools(tool_blocks, tool_handlers)
        messages.append({"role": "user", "content": tool_results})
```

### Step 3: Token Tracking + Context Overflow 방지

```python
# Step 3: Token tracking 추가

MAX_CONTEXT_TOKENS = 200_000  # 모델별 한계
COMPACT_THRESHOLD = 0.8       # 80%에서 압축 시작

class TokenTracker:
    def __init__(self):
        self.total_input = 0
        self.total_output = 0

    def update(self, usage):
        self.total_input = usage.input_tokens  # 누적이 아닌 최신값
        self.total_output += usage.output_tokens

    def needs_compact(self) -> bool:
        return self.total_input > MAX_CONTEXT_TOKENS * COMPACT_THRESHOLD

    def is_at_limit(self) -> bool:
        return self.total_input > MAX_CONTEXT_TOKENS * 0.95

async def run_agent_with_tracking(user_message, tools, tool_handlers):
    messages = [{"role": "user", "content": user_message}]
    tracker = TokenTracker()

    while True:
        # Context overflow 방지
        if tracker.is_at_limit():
            messages = compact_messages(messages)  # 요약으로 압축

        elif tracker.needs_compact():
            messages = compact_messages(messages)

        response = call_model(messages, tools)
        tracker.update(response.usage)

        # ... (나머지 tool loop 동일)

def compact_messages(messages):
    """대화 히스토리를 요약으로 압축. 05_context_management.md에 전체 구현."""
    # Call LLM with summarization prompt → replace old messages with summary
    # → insert CompactBoundary marker. See 05_context_management.md for full implementation.
    summary = llm_call(messages[:-4], system=COMPACT_PROMPT, max_turns=1)
    summary = strip_analysis_block(summary)  # <analysis> 제거, <summary>만 보존
    return [make_summary_message(summary), COMPACT_BOUNDARY, *messages[-4:]]
```

### Step 4: Error Recovery (PTL + Max Output Tokens)

```python
# Step 4: Recovery 경로 추가

MAX_OUTPUT_RECOVERY = 3

async def run_agent_resilient(user_message, tools, tool_handlers):
    messages = [{"role": "user", "content": user_message}]
    output_recovery_count = 0
    max_output_override = None

    while True:
        try:
            response = call_model(
                messages, tools,
                max_tokens=max_output_override or 4096,
            )
        except PromptTooLongError:
            # Recovery: 대화 압축 후 재시도
            messages = compact_messages(messages)
            continue

        # Max output tokens 처리
        if response.stop_reason == "max_tokens":
            # 1단계: 에스컬레이션 (4k → 64k)
            if max_output_override is None:
                max_output_override = 64000
                continue  # 같은 요청 재시도

            # 2단계: Multi-turn recovery
            if output_recovery_count < MAX_OUTPUT_RECOVERY:
                messages.append({
                    "role": "assistant", "content": response.content
                })
                messages.append({
                    "role": "user",
                    "content": "Output limit hit. Resume directly, "
                               "no recap. Pick up mid-thought."
                })
                output_recovery_count += 1
                max_output_override = None
                continue

        # 정상 종료 또는 tool 실행
        output_recovery_count = 0
        max_output_override = None
        # ... tool execution logic
```

### Step 5: Crash Resilience (Transcript 저장)

```python
# Step 5: --resume을 위한 transcript 선 저장

import json
from pathlib import Path

SESSION_FILE = Path("~/.my-agent/sessions/").expanduser()

def save_transcript(session_id: str, messages: list):
    """API 호출 전에 항상 호출한다."""
    path = SESSION_FILE / f"{session_id}.json"
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(messages, default=str))

def load_transcript(session_id: str) -> list | None:
    """--resume 시 호출."""
    path = SESSION_FILE / f"{session_id}.json"
    if path.exists():
        return json.loads(path.read_text())
    return None

async def run_agent_persistent(user_message, tools, session_id):
    # Resume 지원
    existing = load_transcript(session_id)
    if existing:
        messages = existing
    else:
        messages = [{"role": "user", "content": user_message}]

    while True:
        # [핵심] API 호출 전에 저장
        save_transcript(session_id, messages)

        response = call_model(messages, tools)
        # ... tool loop
```

### Step 6: AsyncGenerator로 전환

```python
# Step 6: AsyncGenerator 패턴 (Claude Code 방식)

from typing import AsyncGenerator

async def query(
    messages: list,
    tools: list,
    max_turns: int = 50,
) -> AsyncGenerator[dict, None]:
    """
    AsyncGenerator로 구현하면:
    1. 호출자가 실시간으로 각 메시지를 받아볼 수 있다
    2. 호출자가 언제든 중단할 수 있다 (.return())
    3. Sub-generator와 합성할 수 있다 (yield from)
    """
    turn = 0

    while True:
        turn += 1
        if turn > max_turns:
            yield {"type": "max_turns_reached", "turns": turn}
            return

        # API 호출 + streaming
        async for chunk in call_model_streaming(messages, tools):
            yield {"type": "stream_chunk", "data": chunk}
            # ↑ 호출자에게 실시간 전달

        final = collect_final_message()
        yield {"type": "assistant_message", "data": final}

        tool_blocks = extract_tool_uses(final)
        if not tool_blocks:
            return  # 정상 종료

        for block in tool_blocks:
            result = await execute_tool(block)
            yield {"type": "tool_result", "data": result}

        # messages 갱신 후 continue


# 호출자 코드
async def main():
    async for event in query(messages, tools):
        if event["type"] == "stream_chunk":
            print(event["data"], end="")
        elif event["type"] == "tool_result":
            print(f"\n[Tool] {event['data']}")
        elif event["type"] == "max_turns_reached":
            print(f"\n[Warning] {event['turns']} turns reached")
```

### 구현 순서 요약

| 단계 | 추가 기능 | 코드량 추가 | 영향 |
|------|----------|-----------|------|
| 1 | 기본 tool loop | ~30줄 | "동작은 함" |
| 2 | Streaming | +50줄 | UX 개선 (실시간 출력) |
| 3 | Token tracking | +40줄 | 긴 대화에서 crash 방지 |
| 4 | Error recovery | +60줄 | PTL/max_tokens에서 자동 복구 |
| 5 | Transcript 저장 | +30줄 | Crash 후 resume 가능 |
| 6 | AsyncGenerator | +40줄 (리팩토링) | 합성 가능한 아키텍처 |

**각 단계는 독립적으로 적용 가능**하다. Step 1만으로도 동작하는 agent이고, production 요구사항에 따라 Step 2-6을 점진적으로 추가하면 된다. Claude Code는 이 모든 단계를 (그리고 그 이상을) 구현한 결과물이다.

---

## Key Takeaways

- **Tool loop는 에이전트의 본질**: `while(tool_use) { execute → re-submit }` 패턴은 모든 에이전트 앱의 핵심이다. Claude Code는 이를 streaming과 결합하여 실시간 UX를 제공한다.
- **Tool 에러는 LLM에게 반환**: 에러를 앱이 처리하지 않고 LLM에게 tool_result로 돌려보내면, LLM이 자체적으로 recovery 전략을 수행할 수 있다.
- **Token tracking은 선택이 아닌 필수**: 매 turn마다 사용량을 추적하고, overflow 전에 proactive하게 압축하는 것이 안정적인 에이전트의 전제조건이다.
- **Streaming + Tool execution의 조합**: Tool 입력이 streaming으로 도착하므로, JSON buffering + content_block_stop 대기가 필수적이다.
- **Message normalization**: 내부 표현과 API 표현이 다르므로 양방향 변환 레이어가 반드시 필요하다.
- **State machine으로서의 tool loop**: 단순한 while 루프가 아니라 5개의 명확한 phase와 다수의 recovery path를 가진 state machine이다. `transition` 필드로 "왜 돌아왔는지"를 추적하는 것이 디버깅의 핵심이다.
- **Transcript 선 저장 (WAL 패턴)**: API 호출 전에 저장해야 crash resilience를 확보할 수 있다. "작업을 하기 전에 의도를 기록한다."
- **Sibling message split**: Parallel tool call 지원 시 같은 message ID를 공유하는 분리된 record를 하나의 논리적 turn으로 취급해야 token counting이 정확해진다.
