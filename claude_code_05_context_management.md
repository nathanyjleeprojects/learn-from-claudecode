<!--
tags: context-management/token-counting, context-management/compression, context-management/conversation-history, context-management/memory-system, context-management/disk-offload, context-management/file-cache
keywords: context-management, token-counting, compact-mode, microcompact, conversation-history, context-window, token-estimation, cache-aware-counting, CLAUDE-md, memory-system, context-overflow, tool-result-persistence, file-state-cache, analysis-drafting
related_files: 02_query_engine_core.md, 03_prompt_engineering.md, 06_api_layer_providers.md, 14_state_persistence.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 05. Context Management

> LLM 앱에서 가장 과소평가되는 문제 — context window 관리. Token counting, tool result 디스크 오프로드, compact/microcompact 압축, FileStateCache, conversation history 관리, memory 시스템.

## Keywords
context-management, token-counting, compact-mode, microcompact, context-window-overflow, token-estimation, cache-aware-counting, conversation-history, context-compression, memory-system, CLAUDE-md, session-memory, tool-result-persistence, disk-offload, file-state-cache, analysis-drafting

## Context Budget Problem: 왜 이게 어려운가

에이전트 앱은 context window를 예측하기 어렵다:

- **System prompt**: ~11K tokens (tool descriptions 포함)
- **Conversation history**: 대화가 길어질수록 증가
- **Tool results**: 파일 읽기, grep 결과 등 **예측 불가능한 크기**
- **Thinking blocks**: Extended thinking이 추가 token 소모

일반 챗봇과 달리, 에이전트는 **tool result의 크기를 제어할 수 없다**. `grep`으로 1000줄이 나올 수도 있고, 파일 하나가 10K tokens일 수도 있다. 따라서 context overflow는 "언젠가 일어날 문제"가 아니라 **매 세션마다 대비해야 하는 현실**이다.

더 나쁜 시나리오: **N개의 병렬 tool 호출이 동시에 큰 결과를 반환하는 경우**. 예를 들어 10개의 파일을 동시에 읽으면 10 x 40K = 400K 문자가 한 턴에 쏟아진다. 이것이 단순 truncation이 아닌 **다층 방어 전략**이 필요한 이유다.

**LLM 엔지니어링 인사이트**: "Context window가 1M이니까 충분하다"는 프로토타입에서는 맞지만, 프로덕션에서는 틀리다. Token 비용, 응답 속도, 그리고 LLM의 "lost in the middle" 문제(긴 context의 중간 부분을 놓치는 현상) 때문에, 작은 context가 항상 더 낫다.

## Token Counting: 정확 vs 추정

Claude Code는 두 가지 token counting 전략을 사용한다:

### 1. API 응답 기반 (정확)

`src/utils/tokens.ts`의 `getTokenUsage(message)`:

```typescript
// API response의 usage 필드에서 정확한 수치 추출
{
  input_tokens: 15234,
  cache_creation_input_tokens: 8000,   // 새로 캐시에 저장된 토큰
  cache_read_input_tokens: 3000,       // 캐시에서 읽은 토큰
  output_tokens: 2456,
}
```

`getTokenCountFromUsage()`는 이 값들을 합산한다:
- **총 비용 토큰** = input + cache_creation + cache_read + output
- Cache-aware counting이 정확한 비용 추적을 가능하게 함

### 2. 사전 추정 (빠른)

`src/services/tokenEstimation.ts`의 `roughTokenCountEstimationForMessages()`:

- API 호출 전에 대략적 크기를 추정하여 overflow를 미리 방지
- 문자열 길이 기반 heuristic (영어 약 4자/token, 한국어 약 2자/token)
- Tool schema, system prompt 크기를 고정 상수로 추가
- 정확도보다 **속도와 보수적 추정**(실제보다 크게 추정)이 목표

**LLM 엔지니어링 인사이트**: Token counting은 "API 호출 후 정확한 값"과 "API 호출 전 빠른 추정" 두 가지가 모두 필요하다. 추정만으로는 부정확하고, 정확한 값만으로는 overflow를 사전에 방지할 수 없다.

## Tool Result 디스크 오프로드

Context overflow의 가장 빈번한 원인은 **거대한 tool result**다. Claude Code는 tool result가 일정 크기를 초과하면 전체 내용을 디스크에 저장하고, context에는 preview만 남기는 **디스크 오프로드** 전략을 사용한다.

### 세 가지 크기 임계값

`src/constants/toolLimits.ts`에 정의된 핵심 상수:

| 상수 | 값 | 역할 |
|------|------|------|
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 (50K) | **개별 tool result** 최대 크기. 초과 시 디스크에 persist |
| `MAX_TOOL_RESULT_TOKENS` | 100,000 (100K) | Token 기준 상한. ~400KB 텍스트 |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 (200K) | **한 턴의 병렬 tool result 합산** 최대 크기 |

세 번째 임계값이 핵심이다. 개별 tool result가 각각 50K 이하여도, 병렬 호출 결과의 **합**이 200K를 초과하면 가장 큰 것부터 디스크로 내린다.

> **WHY**: 10개의 Read tool이 병렬로 실행되어 각각 40K를 반환하면, 한 턴에 400K 문자가 context에 들어간다. 개별 50K 체크만으로는 이 상황을 방어할 수 없다.

### 오프로드 알고리즘

`src/utils/toolResultStorage.ts`의 `maybePersistLargeToolResult()`:

```
function maybePersistLargeToolResult(toolResult, toolName, threshold):
    content = toolResult.content

    // 1. 빈 결과 처리 — 일부 모델이 빈 tool_result에서 오작동
    if isEmpty(content):
        return "(toolName completed with no output)"

    // 2. 이미지 블록은 skip — 그대로 전송해야 함
    if hasImageBlock(content):
        return toolResult

    // 3. 크기 체크 — 대부분의 결과는 여기서 바로 통과
    size = contentSize(content)
    if size <= threshold:
        return toolResult    // 작은 결과는 그대로 유지

    // 4. 디스크에 persist
    filepath = persistToolResult(content, toolUseId)
    // 'wx' 플래그로 원자적 쓰기 — 같은 ID는 한 번만 저장

    // 5. Preview 생성 (첫 2000바이트, 줄바꿈 경계에서 자름)
    preview = generatePreview(content, 2000)

    // 6. <persisted-output> 태그로 래핑
    return buildLargeToolResultMessage(filepath, preview)
```

### Preview 생성의 디테일

```typescript
// generatePreview: 2000바이트에서 가장 가까운 줄바꿈 경계에서 자름
const truncated = content.slice(0, maxBytes)
const lastNewline = truncated.lastIndexOf('\n')
// 줄바꿈이 50% 이상 지점에 있으면 거기서 자르고, 아니면 정확히 maxBytes에서 자름
const cutPoint = lastNewline > maxBytes * 0.5 ? lastNewline : maxBytes
```

모델이 보는 최종 형태:

```xml
<persisted-output>
Output too large (42.3 KB). Full output saved to: /path/to/session/tool-results/tu_abc123.txt

Preview (first 2.0 KB):
[첫 2000바이트의 내용, 줄바꿈 경계에서 잘림]
...
</persisted-output>
```

### 메시지 단위 합산 예산 (Per-Message Budget)

`enforceToolResultBudget()`는 한 턴의 모든 tool result를 합산하여 200K 제한을 적용한다:

1. **같은 assistant response에 속하는 tool result들을 그룹으로 묶음** (normalize 후 와이어에서 하나의 user message가 되므로)
2. 그룹의 총 크기가 200K 초과 시, **가장 큰 것부터** 디스크 오프로드
3. `ContentReplacementState`로 교체 결정을 기록 — prompt cache를 보존하기 위해 **같은 결정을 반복**해야 함. API의 prompt caching은 이전 턴의 content가 동일해야 cache hit이 된다. 따라서 한 번 오프로드 결정을 내리면, 이후 턴에서도 동일한 오프로드를 적용해야 캐시가 유지된다. ContentReplacementState가 이 결정을 세션 내내 추적한다.
4. Subagent fork 시 `cloneContentReplacementState()`로 부모의 교체 상태를 복제

**LLM 엔지니어링 인사이트**: 디스크 오프로드는 단순히 "큰 결과를 잘라내는 것"이 아니다. 모델은 `<persisted-output>` 태그를 보고 필요하면 `Read` tool로 해당 파일을 다시 읽을 수 있다. 즉, **정보를 버리는 것이 아니라 lazy-loading으로 전환**하는 것이다.

## FileStateCache: 파일 읽기 중복 방지

에이전트는 같은 파일을 반복해서 읽는 경향이 있다. 분석하고, 수정하고, 다시 확인하고... 매번 파일 전체를 context에 넣으면 token이 낭비된다. `src/utils/fileStateCache.ts`의 `FileStateCache`가 이 문제를 해결한다.

### 캐시 구성

```typescript
// LRU 캐시: 최대 100개 항목, 25MB 메모리 상한
READ_FILE_STATE_CACHE_SIZE = 100
DEFAULT_MAX_CACHE_SIZE_BYTES = 25 * 1024 * 1024  // 25MB

// LRUCache의 sizeCalculation으로 각 항목의 실제 메모리 사용량을 추적
sizeCalculation: value => Math.max(1, Buffer.byteLength(value.content))
```

### 캐시 항목 구조

```typescript
type FileState = {
  content: string          // 파일 내용 (원본 디스크 바이트)
  timestamp: number        // 최종 읽기 시점 — 변경 감지에 사용
  offset: number | undefined   // 부분 읽기 시 시작 위치
  limit: number | undefined    // 부분 읽기 시 줄 수 제한
  isPartialView?: boolean  // CLAUDE.md 자동 주입 등 가공된 내용인 경우 true
}
```

### Path Normalization: 캐시 미스 방지

모든 캐시 접근에서 `normalize(key)`를 호출한다:

```typescript
// 이 두 경로가 같은 캐시 항목을 가리킴
cache.get("./src/utils/foo.ts")
cache.get("/absolute/path/src/utils/foo.ts")
// normalize()가 /foo/../bar → /bar, ./ → absolute path 등을 처리
```

이것이 없으면 tool이 상대경로로 접근하고, 다른 tool이 절대경로로 접근할 때 **같은 파일에 대해 두 번 캐시 미스**가 발생한다.

### Fork/Subagent에서의 캐시 관리

```typescript
// 서브에이전트 생성 시 부모 캐시를 깊은 복사
function cloneFileStateCache(cache: FileStateCache): FileStateCache {
  const cloned = createFileStateCacheWithSizeLimit(cache.max, cache.maxSize)
  cloned.load(cache.dump())   // LRUCache의 dump/load로 전체 상태 복제
  return cloned
}

// 서브에이전트 완료 후, 더 최신 항목만 병합
function mergeFileStateCaches(parent, child): FileStateCache {
  const merged = clone(parent)
  for (const [path, state] of child.entries()) {
    const existing = merged.get(path)
    if (!existing || state.timestamp > existing.timestamp) {
      merged.set(path, state)  // 더 최신 정보가 이김
    }
  }
  return merged
}
```

**LLM 엔지니어링 인사이트**: 파일 캐시는 token 절약뿐 아니라 **Edit/Write 안전성**에도 기여한다. `isPartialView` 플래그는 CLAUDE.md 자동 주입처럼 가공된 내용이 캐시에 있을 때, Edit/Write가 이를 기반으로 잘못된 수정을 하지 않도록 "명시적 Read 필수"를 강제한다.

## Compact Mode: 대화 히스토리 압축

Context가 한계에 근접하면 `src/services/compact/`가 대화 히스토리를 압축한다.

### Compact의 동작 원리와 구체적 수치

```typescript
POST_COMPACT_MAX_FILES_TO_RESTORE = 5       // 압축 후 복원할 최대 파일 수
POST_COMPACT_TOKEN_BUDGET = 50_000          // 압축 후 총 토큰 예산
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000    // 파일당 최대 토큰
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000   // 스킬당 최대 토큰
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000   // 스킬 총 토큰 예산
MAX_COMPACT_STREAMING_RETRIES = 2           // 스트리밍 재시도
MAX_PTL_RETRIES = 3                         // prompt-too-long 재시도
```

1. **Trigger**: Token 사용량이 임계값 초과 시 자동 또는 `/compact` 명령으로 수동
2. **Pre-compact hooks 실행**: 사용자 정의 hook이 있으면 먼저 실행
3. **대상 선택**: 오래된 대화 턴을 API-round 그룹 단위로 선택
4. **요약 생성**: Streaming으로 요약 생성 + PTL(Prompt Too Long) 발생 시 가장 오래된 그룹부터 drop하며 최대 3회 재시도
5. **교체**: 원본 턴들을 요약으로 교체
6. **경계 마커**: `CompactBoundaryMessage` 삽입
7. **Post-compact 복원**: 최근 파일 내용, 스킬 정보 등을 budget 내에서 재주입

### Compact의 `<analysis>` 드래프팅 기법

Compact의 품질을 결정하는 핵심 기법은 **분석-후-요약(analysis-then-summary)** 패턴이다. 이것은 chain-of-thought의 실전 응용이다.

#### 작동 원리

LLM에게 두 단계로 응답하게 한다:

```
Step 1: <analysis> 태그에 상세 분석을 작성
        - 대화의 모든 메시지를 시간순으로 분석
        - 사용자 요청, 기술적 결정, 파일/코드 변경 등을 꼼꼼히 정리
        - 이것은 "생각하는 과정"의 scratchpad

Step 2: <summary> 태그에 깔끔한 요약을 작성
        - analysis를 바탕으로 정리된 최종 요약
        - 구조화된 9개 섹션으로 작성
```

#### `formatCompactSummary()`: analysis 제거

```typescript
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary

  // analysis 블록 제거 — 품질 향상에 기여하지만 context에 남길 이유 없음
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, ''
  )

  // summary 태그를 읽기 쉬운 형식으로 변환
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${summaryMatch[1].trim()}`
    )
  }
  return formattedSummary.trim()
}
```

**핵심 통찰**: `<analysis>` 블록은 요약의 **품질을 높이지만**, 생성 후 context에서 **제거**된다. 이것은 extended thinking과 유사한 패턴이지만, 프롬프트 레벨에서 직접 구현한 것이다. Reasoning은 비용을 들여 수행하되, 그 결과물만 보존한다.

#### Anti-Tool-Call Preamble

Compact fork는 부모의 전체 tool 세트를 상속하므로(cache-key 일치를 위해), 모델이 tool을 호출하려 할 수 있다. 이를 방지하는 강력한 preamble:

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

`maxTurns: 1`로 설정되어 있어서, tool 호출이 거부되면 텍스트 출력 없이 턴이 끝남 = 요약 실패. 이 preamble은 **Sonnet 4.6+에서 2.79%이던 tool 호출 시도율을 사실상 0으로 줄인다**.

#### 9개 필수 요약 섹션

Compact prompt가 요구하는 구조화된 섹션:

| # | 섹션명 | 보존하는 정보 |
|---|--------|------------|
| 1 | **Primary Request and Intent** | 사용자의 모든 명시적 요청과 의도 |
| 2 | **Key Technical Concepts** | 논의된 기술, 프레임워크, 패턴 |
| 3 | **Files and Code Sections** | 검토/수정/생성한 파일, 전체 코드 스니펫, 변경 이유 |
| 4 | **Errors and Fixes** | 발생한 에러와 해결 과정 (사용자 피드백 포함) |
| 5 | **Problem Solving** | 해결된 문제와 진행 중인 트러블슈팅 |
| 6 | **All User Messages** | Tool result가 아닌 모든 사용자 메시지 (의도 변화 추적) |
| 7 | **Pending Tasks** | 명시적으로 요청받은 미완료 작업 |
| 8 | **Current Work** | 요약 직전 작업의 정확한 상태 (파일명, 코드 포함) |
| 9 | **Optional Next Step** | 최근 요청과 직접 관련된 다음 단계만 (탈선 방지) |

**섹션 6이 독특하다**: "모든 사용자 메시지"를 원문 그대로 보존하라는 요구. Tool result에 파묻힌 사용자의 피드백과 의도 변화가 compact 과정에서 사라지는 것을 방지한다.

**섹션 9의 탈선 방지 장치**: "IMPORTANT: ensure that this step is DIRECTLY in line with the user's most recent explicit requests." Compact 후 모델이 오래된 미완료 작업에 되돌아가는 것(drift)을 방지한다. 직전 작업에서 직접 인용(verbatim)까지 요구한다.

### 세 가지 Compact 변형

| 변형 | 대상 | 용도 |
|------|------|------|
| **Full compact** | 전체 대화 | context가 크게 초과할 때 |
| **Partial compact (recent)** | 최근 메시지만 | 이전 context 유지 + 최근만 압축 |
| **Partial compact (up-to)** | 이전 메시지만 | 최근 작업 유지 + 오래된 부분 압축 |

Partial up-to 변형에서는 섹션 8이 "Work Completed"로, 섹션 9가 "Context for Continuing Work"로 바뀐다.

### Grouped Tool Uses

연속된 tool 호출들을 그룹으로 묶어서 한번에 요약한다:

```
Before compact:
  [tool_use: read file A] → [tool_result: 200 lines] →
  [tool_use: read file B] → [tool_result: 300 lines] →
  [tool_use: edit file A] → [tool_result: success]

After compact:
  "파일 A와 B를 읽은 후, 파일 A의 X 함수를 수정하여 Y 기능을 추가함"
```

## Microcompact: API-Side Context Management

Compact보다 더 적극적인 압축 방식. `cache_edits` beta feature를 사용한다:

- **Client-side compact**: 클라이언트가 요약을 생성하여 메시지 교체
- **Microcompact**: API 서버가 직접 context를 관리하여 delta만 전송

Microcompact는 `CONTEXT_MANAGEMENT_BETA_HEADER`로 활성화되며, prompt caching과 연동하여 이전 대화의 캐시된 부분은 유지하면서 새로운 부분만 추가한다.

이 기능은 Anthropic API의 beta feature로, 공개 문서가 제한적이다. 핵심 원리: server가 이전 턴의 캐시된 prefix를 유지하면서 새 메시지만 delta로 전송받는 방식. 직접 구현보다는 API provider가 지원할 때 활용하는 것이 현실적이다.

**LLM 엔지니어링 인사이트**: API provider가 server-side context management를 지원한다면 적극 활용하라. 클라이언트가 요약을 생성하는 것보다 정보 손실이 적고, 캐시 효율도 높다.

## Conversation History Management

### 히스토리 구조

```typescript
messages: [
  { role: "user", content: "파일을 분석해줘" },
  { role: "assistant", content: [
    { type: "text", text: "네, 분석하겠습니다." },
    { type: "tool_use", id: "tu_1", name: "FileRead", input: { path: "..." } },
  ]},
  { role: "user", content: [
    { type: "tool_result", tool_use_id: "tu_1", content: "파일 내용..." },
  ]},
  { role: "assistant", content: "분석 결과입니다..." },
]
```

### Normalization Rules

`normalizeMessagesForAPI()`가 내부 표현을 API 형식으로 변환:

- `tool_result`는 반드시 `user` role
- 연속된 같은 role의 메시지는 하나로 합침
- Advisor blocks, reference blocks 제거
- System reminder tags 제거 또는 유지 (위치에 따라)
- Empty content blocks 제거

### What Gets Preserved vs Dropped

| Preserved | Dropped |
|-----------|---------|
| Tool call + result pairs | 중간 progress messages |
| 사용자 지시사항 | Advisor blocks |
| 에러 메시지와 recovery | 중복된 파일 내용 |
| 파일 경로, 함수명 | 인사말, 확인 응답 |

## Memory System: 세션을 넘어서는 Context

`src/memdir/`가 persistent memory를 관리한다. [상세 -> 14_state_persistence.md]

### Memory 계층과 로딩

| Level | File | 로딩 시점 | Cache |
|-------|------|----------|-------|
| Global | `~/.claude/CLAUDE.md` | 앱 시작 시 | Memoized |
| Project | `CLAUDE.md` (프로젝트 루트) | 앱 시작 시 | Memoized |
| User memories | `~/.claude/projects/{id}/memory/*.md` | 쿼리 시 | On-demand |
| Session | 대화 중 추출 | 실시간 | In-memory |

### Relevant Memory Retrieval

`findRelevantMemories()`가 현재 쿼리에 관련된 memory만 선택하여 context에 주입한다. 모든 memory를 항상 주입하면 context를 낭비하므로, **relevance filtering**이 필수적이다.

## Context Size Planning

`src/utils/analyzeContext.ts`와 `src/utils/queryContext.ts`가 API 호출 전에 context 크기를 계획한다:

1. **System prompt parts 수집**: `fetchSystemPromptParts()`
2. **Attachment 관리**: Memory, code snippets 등
3. **Unused token 감지**: 할당되었지만 사용되지 않는 token 공간
4. **Overflow 예측**: 현재 메시지 + tool results 추정 -> 잔여 공간 계산
5. **Compact 필요 판단**: 잔여 공간이 부족하면 compact 트리거

## Context Window 크기별 전략

| Context Size | 전략 |
|-------------|------|
| < 50% 사용 | 정상 운영 |
| 50-80% | Token 추정 강화, tool result truncation 시작 |
| 80-95% | Compact mode 트리거, 오래된 턴 압축 |
| > 95% | Microcompact, 가장 오래된 non-essential 메시지 제거 |
| Overflow | `max tokens exceeded` -> retry with context reduction |

## 대안 비교: 무한 Context 신뢰 vs 능동 압축

"Context window가 1M tokens이면 관리할 필요 없지 않나?" 이 질문은 프로덕션 LLM 앱을 만들 때 반드시 부딪히는 문제다.

### 무한 Context 신뢰 (Naive Approach)

**전략**: 모든 것을 그대로 넣고, window가 차면 오래된 것부터 자른다.

| 장점 | 단점 |
|------|------|
| 구현 단순 | **Lost-in-the-middle**: 100K+ tokens에서 중간 내용의 recall이 급격히 떨어짐 |
| 정보 손실 없음 (window 내) | **비용 폭발**: 200K input tokens = ~$0.60/request (Claude Sonnet 기준). 10턴이면 $6 |
| | **속도 저하**: Input tokens 비례하여 TTFT(Time to First Token) 증가 |
| | **캐시 비효율**: 매 턴마다 대부분의 tokens를 다시 처리 |

### 능동 압축 (Claude Code Approach)

**전략**: 다층 방어 — 디스크 오프로드 -> 캐시 -> 압축 -> 메모리

| 장점 | 단점 |
|------|------|
| 비용 예측 가능 (compact 후 ~50K 토큰 유지) | 구현 복잡도 높음 |
| TTFT 일정 | 압축 과정에서 정보 손실 가능 |
| Lost-in-the-middle 회피 | Compact 자체가 추가 API 호출 (비용) |
| 캐시 친화적 (접두사 안정) | 압축 프롬프트 설계에 경험 필요 |

### 실제 수치로 보는 차이

```
시나리오: 20턴 코딩 세션, 각 턴 평균 5K tokens의 tool result

Naive approach:
  턴 20의 input = ~100K tokens
  비용: ~$0.30/request, 누적 ~$3.00
  TTFT: ~3-5초

능동 압축 (compact at 80%):
  턴 20의 input = ~50K tokens (compact 후)
  비용: ~$0.15/request + compact 비용 $0.10 x 2회 = 누적 ~$1.70
  TTFT: ~1-2초
```

**LLM 엔지니어링 인사이트**: 능동 압축은 단순히 비용 절약이 아니다. **LLM의 인지 품질을 높인다**. 200K tokens 중간에 파묻힌 중요한 사용자 지시사항보다, 50K tokens의 잘 정리된 요약 속의 지시사항이 모델에게 훨씬 잘 전달된다.

## 직접 만들어보기: Context Management Pipeline Pseudocode

아래는 Claude Code의 context management를 참고하여, 자체 LLM 에이전트에 적용할 수 있는 전체 파이프라인의 pseudocode다.

```python
# === 1. 상수 정의 ===
MAX_CONTEXT_TOKENS = 200_000         # 모델의 context window
COMPACT_TRIGGER_RATIO = 0.80         # 80%에서 compact 시작
PER_TOOL_RESULT_LIMIT = 50_000       # 개별 tool result 문자 상한
PER_MESSAGE_BUDGET = 200_000         # 한 턴의 tool result 합산 상한
PREVIEW_SIZE = 2000                  # 디스크 오프로드 시 preview 크기
BYTES_PER_TOKEN = 4                  # 토큰 추정용 (영어 기준)
# 한국어가 많으면 BYTES_PER_TOKEN = 2로 조정. CJK 비율에 따라 동적 조정 가능.

# === 2. Token 추정 ===
def estimate_tokens(messages, system_prompt):
    """API 호출 전 빠른 추정. 보수적으로 크게 잡는다."""
    total = len(system_prompt) // BYTES_PER_TOKEN
    for msg in messages:
        if isinstance(msg.content, str):
            total += len(msg.content) // BYTES_PER_TOKEN
        elif isinstance(msg.content, list):
            for block in msg.content:
                total += len(str(block)) // BYTES_PER_TOKEN
    return int(total * 1.1)  # 10% safety margin

# === 3. Tool Result 디스크 오프로드 ===
def maybe_offload_tool_result(result, tool_name, threshold=PER_TOOL_RESULT_LIMIT):
    """개별 tool result가 threshold 초과 시 디스크에 저장하고 preview 반환"""
    content = result.content
    if len(content) <= threshold:
        return result  # 작은 결과는 그대로

    # 디스크에 저장
    filepath = f"session/{session_id}/tool-results/{result.id}.txt"
    write_file(filepath, content)

    # Preview 생성: 2000바이트, 줄바꿈 경계에서 자름
    preview = content[:PREVIEW_SIZE]
    last_nl = preview.rfind('\n')
    if last_nl > PREVIEW_SIZE * 0.5:
        preview = preview[:last_nl]

    return ToolResult(
        id=result.id,
        content=f"<persisted-output>\n"
                f"Output too large ({len(content)} chars). "
                f"Saved to: {filepath}\n\n"
                f"Preview:\n{preview}\n...\n"
                f"</persisted-output>"
    )

def enforce_message_budget(tool_results, budget=PER_MESSAGE_BUDGET):
    """한 턴의 병렬 tool results 합산이 budget 초과 시 큰 것부터 오프로드"""
    total = sum(len(r.content) for r in tool_results)
    if total <= budget:
        return tool_results

    # 크기 내림차순 정렬, 큰 것부터 오프로드
    sorted_results = sorted(tool_results, key=lambda r: len(r.content), reverse=True)
    for i, result in enumerate(sorted_results):
        if total <= budget:
            break
        offloaded = maybe_offload_tool_result(result, result.tool_name, threshold=0)
        total -= len(result.content) - len(offloaded.content)
        sorted_results[i] = offloaded
    return sorted_results

# === 4. 파일 캐시 ===
class FileStateCache:
    def __init__(self, max_entries=100, max_bytes=25*1024*1024):
        self.cache = LRUCache(max_entries, max_bytes)

    def get(self, path):
        return self.cache.get(os.path.normpath(path))

    def set(self, path, content, timestamp):
        self.cache.set(os.path.normpath(path), {
            'content': content,
            'timestamp': timestamp
        })

    def clone(self):
        """서브에이전트 생성 시 상태 복제"""
        new_cache = FileStateCache()
        new_cache.cache = self.cache.copy()
        return new_cache

# === 5. Compact 트리거 판단 ===
def should_compact(messages, system_prompt):
    estimated = estimate_tokens(messages, system_prompt)
    return estimated > MAX_CONTEXT_TOKENS * COMPACT_TRIGGER_RATIO

# === 6. Compact 실행 ===
def run_compact(messages, system_prompt, custom_instructions=None):
    """대화 히스토리를 압축하여 token 사용량을 줄인다."""
    if not should_compact(messages, system_prompt):
        return messages

    # 오래된 턴 선택 (최근 2턴은 보존)
    old_turns = messages[:-4]  # assistant+user 쌍 기준
    recent_turns = messages[-4:]

    # compact prompt 구성
    compact_prompt = f"""
    {NO_TOOLS_PREAMBLE}
    Your task is to create a detailed summary of the conversation so far.

    Before your final summary, write <analysis> tags to organize your thoughts.
    Then write your summary in <summary> tags with these sections:
    1. Primary Request and Intent
    2. Key Technical Concepts
    3. Files and Code Sections
    4. Errors and Fixes
    5. Problem Solving
    6. All User Messages
    7. Pending Tasks
    8. Current Work
    9. Optional Next Step
    {custom_instructions or ''}
    """

    # LLM 호출로 요약 생성
    summary = llm_call(
        messages=old_turns,
        system=compact_prompt,
        max_turns=1  # tool 호출 방지
    )

    # <analysis> 블록 제거 — reasoning은 품질 향상용, context에는 불필요
    summary = re.sub(r'<analysis>.*?</analysis>', '', summary, flags=re.DOTALL)
    summary_match = re.search(r'<summary>(.*?)</summary>', summary, re.DOTALL)
    if summary_match:
        summary = f"Summary:\n{summary_match.group(1).strip()}"

    # 요약 메시지 구성
    summary_messages = [
        {"role": "user", "content": f"[Previous conversation summary]\n{summary}"},
        {"role": "assistant", "content": "Understood. I have the context from our previous conversation."},
    ]

    # Post-compact: 최근 수정 파일 복원
    recent_files = get_recently_modified_files(limit=POST_COMPACT_MAX_FILES_TO_RESTORE)
    restored_tokens = 0
    for file_path in recent_files:
        content = read_file(file_path)
        file_tokens = estimate_tokens(content)
        if restored_tokens + file_tokens > POST_COMPACT_TOKEN_BUDGET:
            content = truncate_to_tokens(content, POST_COMPACT_MAX_TOKENS_PER_FILE)
        summary_messages.append(make_file_context(file_path, content))
        restored_tokens += estimate_tokens(content)

    # 교체: 오래된 턴 -> 요약 메시지 + 복원된 파일 + 경계 마커 + 최근 턴
    return [*summary_messages, *recent_turns]

# === 7. 메인 루프 통합 ===
def agent_loop(user_message):
    messages.append({"role": "user", "content": user_message})

    while True:
        # Compact 체크
        if should_compact(messages, system_prompt):
            messages = run_compact(messages, system_prompt)

        # LLM 호출
        response = llm_call(messages, system_prompt)

        # Tool 호출 처리
        if has_tool_calls(response):
            tool_results = execute_tools_parallel(response.tool_calls)
            # 개별 오프로드
            tool_results = [maybe_offload_tool_result(r, r.tool_name) for r in tool_results]
            # 메시지 단위 예산 적용
            tool_results = enforce_message_budget(tool_results)
            messages.append(response)
            messages.append({"role": "user", "content": tool_results})
        else:
            messages.append(response)
            return response.text
```

## 내 프로젝트에 적용하기

위의 파이프라인을 실제 프로젝트에 적용하기 위한 구체적 단계:

### Step 1: Token 추정기 추가 (난이도: 낮음)

```python
# 가장 먼저 구현할 것: 현재 context 크기를 항상 파악
def add_token_tracking(messages):
    estimated = estimate_tokens(messages, system_prompt)
    ratio = estimated / MAX_CONTEXT_TOKENS
    logger.info(f"Context usage: {ratio:.1%} ({estimated}/{MAX_CONTEXT_TOKENS})")
    return ratio
```

매 턴마다 이 비율을 로깅하는 것만으로도 "언제 문제가 터지는지" 가시성을 확보할 수 있다.

### Step 2: Tool Result 크기 제한 (난이도: 낮음)

개별 tool의 결과에 `max_chars` 파라미터를 추가하고, 초과 시 truncation 또는 디스크 오프로드를 적용한다. 가장 자주 큰 결과를 내는 tool부터:

- **파일 읽기**: 전체 파일 대신 관련 부분만 (offset/limit 파라미터)
- **Shell 실행**: stdout/stderr 합산 크기 제한
- **검색 결과**: 매칭 라인 수 제한

### Step 3: 디스크 오프로드 구현 (난이도: 중간)

위의 `maybe_offload_tool_result()` 패턴을 적용한다. 핵심은:

1. **Preview는 줄바꿈 경계에서 자를 것** — 잘린 줄은 모델에게 혼란을 준다
2. **`<persisted-output>` 같은 명확한 태그** 사용 — 모델이 이것이 전체 내용이 아님을 인식해야 함
3. **원본 파일 경로를 제공** — 모델이 필요하면 Read tool로 다시 읽을 수 있도록

### Step 4: Compact 구현 (난이도: 높음)

Compact는 가장 복잡하지만 가장 큰 효과를 준다:

1. **Compact prompt를 구체적으로 설계**: "무엇을 보존하고 무엇을 버릴지"를 섹션별로 명시
2. **`<analysis>` -> `<summary>` 패턴 사용**: LLM이 먼저 분석하고 요약하게 하되, analysis는 제거
3. **Anti-tool-call 방어**: Compact fork가 tool을 호출하지 못하게 강력한 preamble 추가
4. **PTL 재시도 로직**: Compact 대상이 너무 크면, 가장 오래된 그룹부터 drop하고 재시도
5. **Post-compact 파일 복원**: 최근 작업한 파일 내용을 budget 내에서 다시 주입

### Step 5: 파일 캐시 추가 (난이도: 중간)

파일을 자주 읽는 에이전트라면:

1. **LRU 캐시**: 100개 항목, 25MB 상한이 좋은 시작점
2. **Path normalization 필수**: 상대/절대 경로 혼용에 대비
3. **Timestamp 기반 staleness 체크**: 캐시된 내용이 디스크와 다를 수 있음
4. **Subagent 생성 시 clone, 완료 시 merge**: 병렬 작업에서의 캐시 일관성

### 우선순위 권장

```
[Week 1] Token 추정 + 로깅        → 문제의 규모를 파악
[Week 2] Tool result 크기 제한     → 가장 흔한 overflow 원인 차단
[Week 3] 디스크 오프로드            → 큰 결과를 안전하게 처리
[Week 4+] Compact                  → 장기 세션 지원
```

## Key Takeaways

- **Context budget은 first-class concern**: "충분히 크다"가 아니라 "항상 관리해야 한다". Tool result의 크기가 예측 불가능하므로.
- **다층 방어 전략**: 개별 tool result 제한 (50K) -> 메시지 단위 합산 제한 (200K) -> 디스크 오프로드 -> Compact. 한 겹만으로는 부족하다.
- **이중 counting 전략**: 사전 추정(빠르지만 부정확) + 사후 정확 측정. 둘 다 필요하다.
- **`<analysis>` 드래프팅 기법**: LLM에게 먼저 분석을 쓰게 하고, 결과물만 보존하고 분석 과정은 제거. Chain-of-thought의 실전 응용.
- **Compact prompt 설계가 핵심**: "무엇을 보존하고 무엇을 버릴지"를 9개 섹션으로 구체적 지시. "모든 사용자 메시지 보존"과 "다음 단계 탈선 방지"가 특히 중요.
- **디스크 오프로드 = lazy loading**: 정보를 버리는 것이 아니라, 필요할 때 다시 읽을 수 있도록 전환. `<persisted-output>` 태그가 모델에게 힌트를 제공.
- **FileStateCache**: Path normalization, LRU eviction, timestamp staleness, fork clone/merge까지 — 단순한 캐시가 아닌 에이전트 인프라.
- **능동 압축의 경제학**: 200K input으로 $0.60 쓰는 것보다, compact 비용 $0.10을 써서 50K로 줄이면 비용, 속도, 품질 모두 이득.
- **Server-side context management**: 가능하면 API provider의 cache_edits 같은 기능을 활용하여 정보 손실을 최소화.
- **Memory relevance filtering**: 모든 memory를 항상 주입하지 말고, 현재 쿼리에 관련된 것만 선택적으로 주입.
