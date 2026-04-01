<!--
tags: prompt-engineering/system-prompt, prompt-engineering/hierarchical-assembly, prompt-engineering/feature-gating, context-management/prompt-optimization, prompt-engineering/memory-selection, prompt-engineering/scratchpad
keywords: system-prompt, prompt-engineering, hierarchical-prompt, feature-gated-sections, tool-description, memory-injection, prompt-ordering, negative-instructions, prompt-sections, scratchpad, edit-after-read, internal-external-split, memory-selection
related_files: 02_query_engine_core.md, 04_tool_system.md, 05_context_management.md, 14_state_persistence.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 03. Prompt Engineering

> Claude Code가 시스템 프롬프트를 구축하는 방법: 계층적 조립, feature-gated sections, tool description 생성, memory injection, scratchpad, memory selection. 프로덕션 LLM harness의 프롬프트 엔지니어링 실전 교과서.

## Keywords
system-prompt, prompt-engineering, hierarchical-assembly, feature-gating, tool-description-generation, memory-injection, prompt-ordering, negative-instructions, prompt-sections, modular-prompt, cache-control, scratchpad, edit-after-read, internal-external-split, memory-selection

---

## 계층적 프롬프트 조립: 하드코딩된 문자열이 아니다

Claude Code의 시스템 프롬프트는 단일 문자열이 아니라 **모듈화된 sections의 조합**이다. `src/utils/systemPrompt.ts`의 `fetchSystemPromptParts()`가 다양한 소스에서 prompt sections를 수집하고, 우선순위에 따라 결합한다.

### 프롬프트 소스 우선순위 (높은 순)

1. **Override system prompt** -- 디버깅/테스트용 완전 오버라이드
2. **Coordinator system prompt** -- Multi-agent coordinator 모드일 때
3. **Agent system prompt** -- Custom agent 정의가 있을 때
4. **Custom system prompt** -- 사용자가 설정에서 지정한 프롬프트
5. **Default system prompt** -- `src/constants/prompts.ts`의 기본 프롬프트
6. **Append system prompt** -- 조건부 추가 (feature flags, MCP 서버 등)

**LLM 엔지니어링 인사이트**: 시스템 프롬프트를 계층으로 구성하면, 각 모드(coordinator, agent, user-custom)마다 별도의 거대한 프롬프트를 관리할 필요 없이, 공통 기반 위에 차이점만 overlay할 수 있다.

---

## System Prompt의 구조

`src/constants/prompts.ts` (~900 lines)와 `src/constants/systemPromptSections.ts` (70 lines)가 실제 프롬프트 내용을 정의한다.

### Prompt Section Caching: `systemPromptSection()` vs `DANGEROUS_uncachedSystemPromptSection()`

```typescript
// 캐시됨 -- 한 번 계산, /clear나 /compact까지 유지
systemPromptSection('env_info', () => computeEnvInfo())

// 캐시 안 됨 -- 매 턴마다 재계산 (prompt cache도 깨짐)
DANGEROUS_uncachedSystemPromptSection('mcp_instructions', () => getMcpPrompt(),
  'MCP 서버가 턴 사이에 연결/해제될 수 있으므로')
```

`DANGEROUS_` 접두사는 **의도적 naming convention**이다. 이 함수를 쓰면 prompt cache가 깨지므로, 사용을 자제하게 만든다. 이름 자체가 코드 리뷰에서 경고등 역할을 한다.

### System Prompt의 실제 구성 순서

`getSystemPrompt()`가 반환하는 sections (소스코드 직접 확인):

```
[Globally Cacheable -- 변하지 않는 부분]
1. getSimpleIntroSection()        -- 역할 정의
2. getSimpleSystemSection()       -- 시스템 지시
3. getSimpleDoingTasksSection()   -- 작업 수행 가이드
4. getActionsSection()            -- 행동 지침 (위험 작업 확인)
5. getUsingYourToolsSection()     -- Tool 사용법
6. getSimpleToneAndStyleSection() -- 톤과 스타일
7. getOutputEfficiencySection()   -- 출력 효율성

[SYSTEM_PROMPT_DYNAMIC_BOUNDARY -- cache scope 전환점]

[Dynamic Sections -- 턴마다 변할 수 있는 부분]
8. session_guidance              -- 세션 가이드
9. memory                        -- CLAUDE.md 내용
10. ant_model_override           -- Anthropic 내부 모델 오버라이드
11. env_info_simple              -- OS, shell, git 정보
12. language                     -- 언어 설정
13. output_style                 -- 출력 스타일 (Learning, Explanatory 등)
14. mcp_instructions (UNCACHED!) -- MCP 서버 관련 (cache break)
15. scratchpad                   -- Scratchpad 디렉터리 지침
16. frc                          -- Function Result Clearing
17. summarize_tool_results       -- 도구 결과 요약 지시
18. numeric_length_anchors       -- 숫자 길이 앵커 (ant only)
19. token_budget                 -- 토큰 예산 정보 (feature-flagged)
20. brief                        -- Brief 모드 지시 (KAIROS)
```

**`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`**: 이 마커 위의 부분은 global cache scope로 모든 사용자가 공유할 수 있고, 아래 부분은 사용자/세션별로 다르다. 실제 코드에서 `shouldUseGlobalCacheScope()`가 true일 때만 이 boundary가 삽입된다.

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`는 실제 API 호출에서 `cache_control` 파라미터로 매핑된다:

```json
{
  "system": [
    {"type": "text", "text": "...static sections..."},
    {"type": "text", "text": "...last static section...",
     "cache_control": {"type": "ephemeral"}},
    {"type": "text", "text": "...dynamic sections..."}
  ]
}
```

`cache_control: {"type": "ephemeral"}`이 DYNAMIC_BOUNDARY 직전의 static section에 붙어서, 그 위의 모든 content가 prompt cache에 저장된다. Dynamic sections는 매 턴 바뀔 수 있으므로 cache scope에 포함되지 않는다.

---

## 실제 시스템 프롬프트 텍스트: Anti-Gold-Plating 지시

Claude Code의 프롬프트에서 가장 주목할 만한 부분은 LLM의 "과도한 도움" 경향을 억제하는 구체적인 지시들이다. `getSimpleDoingTasksSection()` 함수에서 직접 확인한 원문:

### "Don't" 3연타: 과도한 기능 추가 방지

```
Don't add features, refactor code, or make "improvements" beyond what was asked.
A bug fix doesn't need surrounding code cleaned up.
A simple feature doesn't need extra configurability.
Don't add docstrings, comments, or type annotations to code you didn't change.
Only add comments where the logic isn't self-evident.
```

### "Three similar lines" 원칙: 조기 추상화 방지

```
Don't create helpers, utilities, or abstractions for one-time operations.
Don't design for hypothetical future requirements.
The right amount of complexity is what the task actually requires—no speculative
abstractions, but no half-finished implementations either.
Three similar lines of code is better than a premature abstraction.
```

이것이 왜 중요한가: LLM은 "좋은 코드 = DRY 코드"라는 학습 데이터의 편향이 있다. 그래서 3줄 중복 코드를 발견하면 본능적으로 헬퍼 함수를 만들려 한다. 하지만 프로덕션에서 사용자가 요청하지 않은 리팩터링은 review burden만 증가시키고, 의도치 않은 regression을 초래한다. "Three similar lines is better than a premature abstraction"은 이 편향에 대한 정확한 카운터다.

### 에러 핸들링 과잉 방지

```
Don't add error handling, fallbacks, or validation for scenarios that can't happen.
Trust internal code and framework guarantees.
Only validate at system boundaries (user input, external APIs).
Don't use feature flags or backwards-compatibility shims when you can just change the code.
```

**LLM 엔지니어링 인사이트**: 이런 부정 지시들은 **구체적인 예시**와 함께 제공될 때 효과적이다. "불필요한 변경을 하지 마라"보다 "A bug fix doesn't need surrounding code cleaned up"이 훨씬 잘 따라진다. 추상적 규칙보다 concrete scenario가 LLM의 행동을 더 정확하게 제어한다.

### "Reversibility and Blast Radius" 프레임워크

`getActionsSection()` 함수는 위험한 액션에 대한 판단 프레임워크를 제공한다. 실제 원문:

```
Carefully consider the reversibility and blast radius of actions. Generally you
can freely take local, reversible actions like editing files or running tests.
But for actions that are hard to reverse, affect shared systems beyond your local
environment, or could otherwise be risky or destructive, check with the user
before proceeding. The cost of pausing to confirm is low, while the cost of an
unwanted action (lost work, unintended messages sent, deleted branches) can be
very high.
```

이 프레임워크의 핵심은 **이진 분류가 아니라 스펙트럼**으로 위험을 판단하게 만드는 것이다:

| 차원 | 낮은 위험 | 높은 위험 |
|------|-----------|-----------|
| Reversibility | 파일 편집, 테스트 실행 | force-push, rm -rf, DB drop |
| Blast radius | 로컬 파일만 영향 | PR/Issue 생성, Slack 메시지, CI 수정 |
| Visibility | 자기 환경 내 | 다른 사람이 볼 수 있는 곳 |

실제 프롬프트에 나열된 위험 액션 예시:

```
- Destructive operations: deleting files/branches, dropping database tables,
  killing processes, rm -rf, overwriting uncommitted changes
- Hard-to-reverse operations: force-pushing, git reset --hard, amending
  published commits, removing or downgrading packages/dependencies
- Actions visible to others: pushing code, creating/closing/commenting on
  PRs or issues, sending messages (Slack, email, GitHub)
```

그리고 장애물 대처에 대한 명시적 지시:

```
When you encounter an obstacle, do not use destructive actions as a shortcut to
simply make it go away. For instance, try to identify root causes and fix
underlying issues rather than bypassing safety checks (e.g. --no-verify).
```

**LLM 엔지니어링 인사이트**: "measure twice, cut once"라는 마무리 문구처럼, 이 프레임워크는 LLM에게 **판단의 원칙**을 제공한다. 모든 시나리오를 열거하는 것은 불가능하므로, 원칙(reversibility, blast radius)을 가르치고 구체적 예시로 보강하는 패턴이 효과적이다.

---

## Internal vs External 프롬프트 차이: Collaborator vs Output Efficiency

Claude Code는 `process.env.USER_TYPE === 'ant'` 분기를 통해 Anthropic 내부 사용자와 외부 사용자에게 **다른 프롬프트**를 제공한다. 이 차이는 단순한 분량 차이가 아니라 **근본적으로 다른 interaction 철학**을 반영한다.

### 외부 사용자: Output Efficiency 모드

```
# Output efficiency

IMPORTANT: Go straight to the point. Try the simplest approach first without
going in circles. Do not overdo it. Be extra concise.

Keep your text output brief and direct. Lead with the answer or action, not the
reasoning. Skip filler words, preamble, and unnecessary transitions. Do not
restate what the user said -- just do it.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan

If you can say it in one sentence, don't use three.
```

핵심 키워드: **concise, direct, brief**. 외부 사용자에게는 "결과물 효율성"이 최우선이다.

### 내부 사용자: Communicating with the User (Collaborator) 모드

```
# Communicating with the user
When sending user-facing text, you're writing for a person, not logging to a
console. Assume users can't see most tool calls or thinking - only your text
output. Before your first tool call, briefly state what you're about to do.
While working, give short updates at key moments: when you find something
load-bearing (a bug, a root cause), when changing direction, when you've made
progress without an update.

When making updates, assume the person has stepped away and lost the thread.
They don't know codenames, abbreviations, or shorthand you created along the
way, and didn't track your process. Write so they can pick back up cold: use
complete, grammatically correct sentences without unexplained jargon.
```

핵심 키워드: **clarity, context, flowing prose**. 내부 사용자에게는 "이해 가능성"이 최우선이다.

### Collaborator 전용 추가 지시

내부 빌드에서만 활성화되는 지시들이 여럿 있다:

**1. 적극적 버그 알림**:
```
If you notice the user's request is based on a misconception, or spot a bug
adjacent to what they asked about, say so. You're a collaborator, not just an
executor--users benefit from your judgment, not just your compliance.
```

**2. 코멘트 철학**:
```
Default to writing no comments. Only add one when the WHY is non-obvious: a
hidden constraint, a subtle invariant, a workaround for a specific bug, behavior
that would surprise a reader.
```

**3. 완료 검증 의무**:
```
Before reporting a task complete, verify it actually works: run the test, execute
the script, check the output. Minimum complexity means no gold-plating, not
skipping the finish line.
```

**4. 결과 보고의 정직성**:
```
Report outcomes faithfully: if tests fail, say so with the relevant output; if
you did not run a verification step, say that rather than implying it succeeded.
Never claim "all tests pass" when output shows failures, never suppress or
simplify failing checks to manufacture a green result. Equally, when a check did
pass or a task is complete, state it plainly -- do not hedge confirmed results
with unnecessary disclaimers.
```

소스코드에서 흥미로운 주석도 확인할 수 있다:

```typescript
// @[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302) -- un-gate once validated on external via A/B
// @[MODEL LAUNCH]: capy v8 assertiveness counterweight (PR #24302) -- un-gate once validated on external via A/B
// @[MODEL LAUNCH]: False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)
```

이 주석들은 이 지시들이 특정 모델(Capybara v8)의 행동 특성에 대한 counterweight임을 보여준다. 내부에서 A/B 테스트로 검증한 후 외부로 확대하는 전략이다.

### 이 차이가 의미하는 것

| 관점 | 외부 (Output Efficiency) | 내부 (Collaborator) |
|------|--------------------------|---------------------|
| 커뮤니케이션 목표 | 최소한의 텍스트로 결과 전달 | 맥락을 잃어도 이해 가능한 보고 |
| 코드 변경 범위 | 요청한 것만 수행 | 인접 버그 발견 시 알림 |
| 코멘트 정책 | 로직이 자명하지 않을 때만 | 기본적으로 코멘트 없음 (더 엄격) |
| 완료 보고 | 암묵적 (작업 완료 후 요약) | 명시적 검증 필수 + 결과 정직 보고 |
| 숫자 길이 앵커 | 없음 | tool call 사이 25단어, 최종 응답 100단어 |

**LLM 엔지니어링 인사이트**: 같은 LLM 제품이라도 사용자 세그먼트에 따라 프롬프트를 분화하는 것이 효과적이다. Anthropic 내부 사용자는 LLM 행동의 미묘한 차이를 감지하고 피드백할 수 있으므로, 더 실험적인 지시(collaborator 모드, 결과 검증 의무)를 먼저 적용하고 A/B 테스트 후 외부로 확대한다. `process.env.USER_TYPE` 분기는 build-time dead code elimination으로 외부 빌드에서는 아예 코드가 제거된다.

---

## Scratchpad 시스템

Claude Code는 세션별 임시 디렉터리인 **Scratchpad**를 제공한다. `getScratchpadInstructions()` 함수의 실제 프롬프트:

```
# Scratchpad Directory

IMPORTANT: Always use this scratchpad directory for temporary files instead of
`/tmp` or other system temp directories:
`/path/to/scratchpad`

Use this directory for ALL temporary file needs:
- Storing intermediate results or data during multi-step tasks
- Writing temporary scripts or configuration files
- Saving outputs that don't belong in the user's project
- Creating working files during analysis or processing
- Any file that would otherwise go to `/tmp`

Only use `/tmp` if the user explicitly requests it.

The scratchpad directory is session-specific, isolated from the user's project,
and can be used freely without permission prompts.
```

### Scratchpad의 설계 의도

1. **Permission bypass**: 사용자 프로젝트 외부이므로 파일 쓰기 권한 프롬프트가 불필요하다. Sandbox 환경에서도 자유롭게 사용 가능.
2. **Session isolation**: 세션별로 독립된 디렉터리이므로 다른 세션의 데이터와 충돌하지 않는다.
3. **Coordinator 간 공유**: Multi-agent (coordinator/subagent) 환경에서 같은 세션의 에이전트들이 scratchpad를 통해 중간 결과를 공유할 수 있다.
4. **프로젝트 오염 방지**: 분석용 임시 파일, 스크립트, 중간 데이터가 사용자의 프로젝트 디렉터리에 남지 않는다.

소스코드에서 확인할 수 있는 등록 방식:

```typescript
// Dynamic section으로 등록 -- 캐시됨 (세션 내 scratchpad 경로 불변)
systemPromptSection('scratchpad', () => getScratchpadInstructions()),
```

`isScratchpadEnabled()`가 false이면 null을 반환하므로 프롬프트에서 완전히 제외된다.

**LLM 엔지니어링 인사이트**: LLM 에이전트에게 "안전한 작업 공간"을 명시적으로 제공하는 것은 간과되기 쉽지만 중요한 패턴이다. `/tmp`를 사용하면 다른 프로세스와 충돌할 수 있고, 사용자 프로젝트에 쓰면 오염된다. 전용 scratchpad는 이 두 문제를 동시에 해결한다.

---

## Memory Selection 메커니즘

Claude Code의 memory 시스템에서 가장 정교한 부분은 **어떤 memory를 선택할 것인가**를 결정하는 메커니즘이다. 이를 위해 별도의 Sonnet 호출을 사용한다.

### 선택 프롬프트 원문

```
You are selecting memories that will be useful to Claude Code as it processes a
user's query. You will be given the user's query and a list of available memory
files with their filenames and descriptions.

Return a list of filenames for the memories that will clearly be useful to
Claude Code as it processes the user's query (up to 5). Only include memories
that you are certain will be helpful based on their name and description.
- If you are unsure if a memory will be useful in processing the user's query,
  then do not include it in your list. Be selective and discerning.
- If there are no memories in the list that would clearly be useful, feel free
  to return an empty list.
- If a list of recently-used tools is provided, do not select memories that are
  usage reference or API documentation for those tools (Claude Code is already
  exercising them). DO still select memories containing warnings, gotchas, or
  known issues about those tools -- active use is exactly when those matter.
```

### 기술적 세부사항

| 속성 | 값 |
|------|-----|
| **Model** | Sonnet (`getDefaultSonnetModel()`) |
| **Max Tokens** | 256 |
| **Output Format** | 구조화된 JSON `{"selected_memories": ["file1.md", ...]}` |
| **Max Results** | 최대 5개 메모리 파일 |
| **제외 대상** | `MEMORY.md` (이미 시스템 프롬프트에 포함), 이전 턴에서 노출된 파일 |

### "API docs는 제외, warnings는 포함" 패턴

이 규칙이 흥미로운 이유: LLM이 이미 특정 tool을 사용 중이면, 그 tool의 기본 사용법 문서는 redundant하다 (이미 tool description에 포함됨). 하지만 **warnings, gotchas, known issues**는 정반대다 -- 실제로 tool을 사용하는 순간이야말로 그 주의사항이 가장 필요한 때다.

```
DO still select memories containing warnings, gotchas, or known issues about
those tools -- active use is exactly when those matter.
```

### 선택 파이프라인

```
User query arrives
    |
    +-- Scan memory directory for .md files with frontmatter
    |   +-- Extract filename + description from YAML headers
    |
    +-- Filter out already-surfaced memories (from prior turns)
    |
    +-- Format manifest: "filename.md -- description"
    |
    +-- Include recently-used tools list (to avoid redundant API docs)
    |
    +-- Sonnet selects up to 5 -> {"selected_memories": ["file1.md", ...]}
        +-- Validated against actual filenames on disk
```

**LLM 엔지니어링 인사이트**: Memory selection에 별도의 LLM(Sonnet)을 사용하는 것은 비용 대비 효과가 높은 전략이다. 키워드 매칭이나 임베딩 유사도보다 semantic 이해가 정확하고, 256 tokens의 짧은 출력이므로 비용이 낮다. 핵심은 **"확실할 때만 포함"**이라는 보수적 기준과, "recently-used tool의 API docs 제외, warnings 포함"이라는 세밀한 규칙이다.

---

## Memory Injection: CLAUDE.md 시스템

Claude Code의 memory 시스템은 시스템 프롬프트에 persistent context를 주입한다:

### Memory 계층

1. **Global memory**: `~/.claude/CLAUDE.md` -- 사용자 선호, 크로스 프로젝트 설정
2. **Project memory**: `CLAUDE.md` (프로젝트 루트) -- 프로젝트 컨벤션, 아키텍처
3. **Session memory**: 대화 중 추출된 기억 (`src/services/extractMemories/`)
4. **Team memory**: 팀 공유 지식 (`src/services/teamMemorySync/`)

### 주입 위치

Memory는 시스템 프롬프트의 **Dynamic section 영역** (SYSTEM_PROMPT_DYNAMIC_BOUNDARY 이후)에 삽입된다. 이는 의도적이다:
- 앞부분의 핵심 지시사항이 memory 내용에 의해 override되지 않도록
- Memory가 길어져도 핵심 행동 지시에 영향을 주지 않도록
- Prompt caching에서 앞부분의 stable sections이 cache hit되도록

**LLM 엔지니어링 인사이트**: Memory를 시스템 프롬프트에 주입할 때 **위치가 중요하다**. 핵심 지시사항은 앞에, 변동이 큰 context(memory, file contents)는 뒤에 배치하면 prompt caching 효율과 instruction following 모두를 최적화할 수 있다.

---

## Edit-after-Read 제약: "Understand Before Modifying" 원칙

Claude Code의 Edit tool 프롬프트에는 강력한 제약이 있다:

```
You must use your `Read` tool at least once in the conversation before editing.
This tool will error if you attempt an edit without reading the file.
```

이것은 단순한 가이드라인이 아니라 **tool-level enforcement**다. Edit tool의 실행 로직 자체가 "이 파일을 이전에 Read한 적이 있는가?"를 확인하고, 없으면 에러를 반환한다.

### 왜 이것이 중요한가

LLM은 파일의 내용을 "추측"하여 편집하려는 경향이 있다. 학습 데이터에서 본 유사한 코드 패턴을 기반으로 `old_string`을 생성하면:
- 실제 파일에 해당 문자열이 없어서 편집 실패
- 비슷하지만 다른 부분을 매칭하여 의도치 않은 변경
- 들여쓰기, 공백 등의 미세한 차이로 인한 오류

시스템 프롬프트에서도 이 원칙을 강화한다:

```
In general, do not propose changes to code you haven't read. If a user asks
about or wants you to modify a file, read it first. Understand existing code
before suggesting modifications.
```

### Edit tool의 추가적 안전장치

```
- When editing text from Read tool output, ensure you preserve the exact
  indentation (tabs/spaces).
- The edit will FAIL if `old_string` is not unique in the file. Either provide
  a larger string with more surrounding context to make it unique or use
  `replace_all` to change every instance.
- ALWAYS prefer editing existing files in the codebase. NEVER write new files
  unless explicitly required.
```

**LLM 엔지니어링 인사이트**: Tool-level enforcement는 prompt-level instruction보다 강력하다. "파일을 먼저 읽어라"라는 지시는 무시될 수 있지만, Read 없이 Edit을 호출하면 에러가 발생하는 것은 무시할 수 없다. 이처럼 중요한 제약은 프롬프트와 코드 양쪽에서 이중으로 강제하는 것이 효과적이다.

---

## Feature-Gated Prompt Sections

시스템 프롬프트의 일부는 feature flags에 따라 조건부로 포함된다:

```typescript
// src/constants/systemPromptSections.ts 패턴
if (feature('PROACTIVE')) {
  sections.push(proactivePromptSection)
}
if (feature('COORDINATOR_MODE')) {
  sections.push(coordinatorPromptSection)
}
if (feature('VOICE_MODE')) {
  sections.push(voicePromptSection)
}
```

**Feature-gated sections 목록**:

| Flag | Prompt Section | 내용 |
|------|---------------|------|
| `PROACTIVE` | Proactive mode | 자율적으로 파일 모니터링, 제안 생성 |
| `COORDINATOR_MODE` | Coordinator | 다중 에이전트 팀 관리 지시 |
| `VOICE_MODE` | Voice | 음성 입력 처리 가이드라인 |
| `KAIROS` | Kairos | Assistant mode 특화 지시 |
| `BRIDGE_MODE` | Bridge | IDE 연동 시 행동 변화 |
| `TOKEN_BUDGET` | Token budget | 사용자 지정 토큰 목표 추적 |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill discovery | 스킬 자동 서피싱 + 검색 |
| `VERIFICATION_AGENT` | Verification | 비자명 구현 후 독립적 검증 의무 |

**LLM 엔지니어링 인사이트**: Feature flag로 프롬프트 sections를 제어하면, 새 기능을 추가할 때 기존 프롬프트를 수정하지 않고 section을 추가하기만 하면 된다. 또한 build-time dead code elimination으로 비활성 feature의 프롬프트 코드 자체가 번들에서 제거된다. 이는 A/B 테스트, 점진적 롤아웃, 내부 우선 실험 등을 가능하게 한다.

---

## Tool Description Generation

42개 tool의 description이 시스템 프롬프트에 포함되는 방식:

1. 각 tool의 `prompt.ts`에서 tool-specific 설명 로드
2. Zod schema -> JSON Schema 변환 (`toolToAPISchema()`)
3. Schema + description을 API의 `tools` 파라미터로 전달
4. 추가로, 시스템 프롬프트에도 사용법 가이드라인 삽입

### Tool Schema Caching

Tool description 생성은 비싸다 (~11K tokens). `src/utils/toolSchemaCache.ts`가 세션 내에서 캐싱한다:

- 각 tool의 rendered schema를 해시로 추적
- Dynamic tools (AgentTool, SkillTool)의 embedded list 변경 감지
- Cache invalidation 트리거: OAuth refresh, tool 변경, MCP 서버 변경
- Mid-session GrowthBook flag 변경에 의한 unexpected cache bust 방지

### Tool 사용 우선순위 지시

시스템 프롬프트(`getUsingYourToolsSection`)에서 Bash 대신 전용 tool을 사용하도록 강제한다. 실제 원문:

```
Do NOT use the Bash to run commands when a relevant dedicated tool is provided.
Using dedicated tools allows the user to better understand and review your work.
This is CRITICAL to assisting the user:
  - To read files use Read instead of cat, head, tail, or sed
  - To edit files use Edit instead of sed or awk
  - To create files use Write instead of cat with heredoc or echo redirection
  - To search for files use Glob instead of find or ls
  - To search the content of files, use Grep instead of grep or rg
  - Reserve using the Bash exclusively for system commands and terminal
    operations that require shell execution.
```

**LLM 엔지니어링 인사이트**: Tool description은 매 API 호출마다 전송되므로, 변경이 없으면 캐시하여 불필요한 재생성을 방지해야 한다. 특히 prompt caching을 사용할 때, tool description이 바뀌면 전체 cache가 무효화되므로 안정성이 중요하다.

---

## Prompt Caching 최적화

Claude Code는 prompt caching을 적극 활용한다. `src/services/api/promptCacheBreakDetection.ts`가 cache break를 감지한다:

### Cache Break Detection

다음 요소들의 해시를 추적하여 cache가 깨지는 시점을 감지한다:
- System prompt hash
- Tool schemas hash
- Cache control hash (scope, TTL 변경)
- Beta headers
- Model 및 fast mode 상태

### Latched States (Sticky-on)

특정 상태는 한번 활성화되면 세션 내에서 되돌아가지 않는다:
- Auto mode 활성화
- Overage usage 감지
- Cache editing 상태

이는 accidental cache break를 방지한다. 예: auto mode를 켰다가 끄면 cache가 깨지므로, 한번 켜진 상태는 유지한다.

### Token Budget 캐싱의 교훈

소스코드 주석에서 발견한 실제 최적화 사례:

```typescript
// Cached unconditionally -- the "When the user specifies..." phrasing
// makes it a no-op with no budget active. Was DANGEROUS_uncached
// (toggled on getCurrentTurnTokenBudget()), busting ~20K tokens per
// budget flip.
```

원래 token budget section은 활성 여부에 따라 UNCACHED로 토글되었는데, 이것이 매번 ~20K tokens의 cache bust를 유발했다. 해결책은 프롬프트 텍스트를 "When the user specifies..."로 시작하게 만들어, budget이 비활성일 때도 프롬프트를 포함하되 자연스럽게 no-op이 되도록 한 것이다. 이렇게 하면 텍스트가 불변이므로 안전하게 캐시할 수 있다.

---

## System Reminder Injection

대화 중간에 동적으로 `<system-reminder>` 태그를 주입하는 패턴:

```xml
<system-reminder>
The following deferred tools are now available via ToolSearch: ...
</system-reminder>
```

System reminders는:
- User message나 tool result에 인라인으로 삽입됨
- 별도의 system prompt 업데이트 없이 실시간으로 context 제공
- Plan mode, permission mode 같은 상태 변화를 LLM에게 알림
- Deferred tools가 로드되었을 때 사용 가능 알림

시스템 프롬프트에서 이 태그의 존재를 설명한다:

```
Tool results and user messages may include <system-reminder> or other tags.
Tags contain information from the system. They bear no direct relation to the
specific tool results or user messages in which they appear.
```

**LLM 엔지니어링 인사이트**: 시스템 프롬프트를 매번 다시 보내는 것보다, 대화 중간에 `<system-reminder>` 같은 마커로 동적 context를 주입하는 것이 더 효율적이다. Prompt cache를 깨지 않으면서도 LLM에게 새로운 정보를 전달할 수 있다.

---

## 대안 비교: 단일 거대 프롬프트 vs 모듈화된 Sections

Claude Code가 채택한 모듈화된 section 방식과, 대안인 단일 거대 프롬프트 방식을 비교한다:

### 접근법 비교

| 관점 | 단일 거대 프롬프트 | 모듈화된 Sections (Claude Code) |
|------|-------------------|-------------------------------|
| **구현 복잡성** | 낮음 -- 문자열 하나 | 높음 -- section registry, cache 관리 |
| **유지보수** | 나쁨 -- 한 줄 변경이 전체 영향 | 좋음 -- 독립적 section 수정 |
| **Prompt cache 효율** | 한 글자라도 바뀌면 전체 miss | Static/Dynamic 분리로 부분 hit |
| **Feature flag 연동** | 조건부 삽입이 string manipulation | Section 등록/해제가 자연스럽다 |
| **A/B 테스트** | 전체 프롬프트를 두 벌 관리 | 특정 section만 variant 생성 |
| **Multi-agent 지원** | 별도 프롬프트 관리 필요 | 공통 기반 + coordinator overlay |
| **디버깅** | 어디서 문제인지 추적 어려움 | Section 이름으로 즉시 식별 |

### Cache 효율의 구체적 차이

단일 프롬프트에서 MCP 서버가 연결/해제되면:
```
[전체 프롬프트 ~15K tokens] -> FULL CACHE MISS
```

모듈화된 sections에서 같은 이벤트:
```
[Static sections ~8K tokens] -> CACHE HIT (global scope)
[Dynamic boundary]
[Dynamic sections ~7K tokens] -> 일부 miss (mcp_instructions만)
```

### Feature Flag + A/B 테스트

소스코드에서 확인한 실제 패턴:

```typescript
// @[MODEL LAUNCH]: capy v8 assertiveness counterweight (PR #24302)
// -- un-gate once validated on external via A/B
...(process.env.USER_TYPE === 'ant'
  ? [`If you notice the user's request is based on a misconception...`]
  : []),
```

이 패턴은:
1. 내부 사용자에게 먼저 배포 (USER_TYPE=ant)
2. A/B 테스트로 효과 검증
3. 검증 후 외부로 확대 (un-gate)
4. Build-time DCE로 외부 빌드에서 코드 자체 제거

단일 프롬프트라면 이런 점진적 롤아웃이 훨씬 복잡해진다.

### 결론

프로덕션 LLM 애플리케이션에서 프롬프트가 한 페이지 이상이 되는 순간, 모듈화는 선택이 아니라 필수다. Claude Code의 `systemPromptSection()` / `DANGEROUS_uncachedSystemPromptSection()` 이원 체계는 유지보수성, cache 효율, 실험 가능성을 모두 만족하는 설계다.

---

## 직접 만들어보기: 시스템 프롬프트 조립기 Pseudocode

Claude Code의 패턴을 참고하여 자신만의 시스템 프롬프트 조립기를 만드는 방법:

```python
from dataclasses import dataclass, field
from typing import Callable, Optional
import hashlib

@dataclass
class PromptSection:
    """시스템 프롬프트의 한 섹션."""
    name: str
    compute: Callable[[], Optional[str]]
    cached: bool = True  # False = DANGEROUS_uncached
    reason: str = ""     # uncached인 경우 이유
    required_feature: str = None  # feature flag 이름. 설정 시 해당 flag가 활성일 때만 포함

class SystemPromptAssembler:
    """
    Claude Code 스타일의 시스템 프롬프트 조립기.

    핵심 개념:
    - Section registry: 이름으로 관리되는 섹션들
    - Memoization: cached=True인 섹션은 한 번만 계산
    - Dynamic boundary: 여기를 기준으로 cache scope 전환
    - Memory injection: 관련 memory를 동적으로 선택/주입
    """

    def __init__(self):
        self._static_sections: list[PromptSection] = []
        self._dynamic_sections: list[PromptSection] = []
        self._memo_cache: dict[str, Optional[str]] = {}
        self._section_hashes: dict[str, str] = {}

    # --- Section 등록 ---

    def register_static(self, name: str, compute: Callable[[], Optional[str]]):
        """Cache-safe section. 변하지 않는 부분."""
        self._static_sections.append(
            PromptSection(name=name, compute=compute, cached=True)
        )

    def register_dynamic(self, name: str, compute: Callable[[], Optional[str]],
                         cached: bool = True, reason: str = ""):
        """
        Dynamic section. SYSTEM_PROMPT_DYNAMIC_BOUNDARY 이후에 배치.

        cached=False로 등록하면 매 턴마다 재계산 -- prompt cache를 깬다.
        Claude Code의 DANGEROUS_ 접두사처럼, 이유를 명시하도록 강제.
        """
        if not cached and not reason:
            raise ValueError(
                f"Uncached section '{name}'은 reason이 필수입니다. "
                "이 section이 매 턴마다 재계산되어야 하는 이유를 명시하세요."
            )
        self._dynamic_sections.append(
            PromptSection(name=name, compute=compute, cached=cached, reason=reason)
        )

    # --- Memoization ---

    def _resolve_section(self, section: PromptSection) -> Optional[str]:
        """섹션을 resolve. cached=True이면 memoize."""
        if section.cached and section.name in self._memo_cache:
            return self._memo_cache[section.name]

        result = section.compute()

        if section.cached:
            self._memo_cache[section.name] = result

        return result

    def invalidate(self, *section_names: str):
        """특정 섹션의 memo cache 무효화. /clear, /compact 시 호출."""
        for name in section_names:
            self._memo_cache.pop(name, None)

    def invalidate_all(self):
        """전체 memo cache 무효화."""
        self._memo_cache.clear()

    # --- Cache break detection ---

    def _compute_hash(self, text: str) -> str:
        return hashlib.blake2b(text.encode(), digest_size=16).hexdigest()

    def detect_cache_break(self, assembled: list[str]) -> bool:
        """
        조립된 프롬프트의 해시를 이전과 비교하여 cache break 감지.
        실제 Claude Code는 system prompt hash, tool schema hash,
        cache control hash 등을 별도로 추적한다.
        """
        full_hash = self._compute_hash("\n".join(assembled))
        changed = self._section_hashes.get("__full__") != full_hash
        self._section_hashes["__full__"] = full_hash
        return changed

    # --- 조립 ---

    def assemble(self, features: dict[str, bool] = None) -> list[str]:
        """
        전체 시스템 프롬프트 조립.

        반환: [static_part1, static_part2, ..., BOUNDARY, dynamic_part1, ...]
        """
        features = features or {}
        parts: list[str] = []

        # 1. Static sections (globally cacheable)
        for section in self._static_sections:
            result = self._resolve_section(section)
            if result is not None:
                parts.append(result)

        # 2. Dynamic boundary marker
        parts.append("__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__")

        # 3. Dynamic sections (session-specific, with feature flag check)
        for section in self._dynamic_sections:
            if section.required_feature and not features.get(section.required_feature, False):
                continue  # feature flag가 비활성이면 skip
            result = self._resolve_section(section)
            if result is not None:
                parts.append(result)

        return parts

# --- Memory injection with relevance filtering ---

class MemorySelector:
    """
    Sonnet을 사용한 memory selection.
    Claude Code 패턴: 최대 5개, 256 tokens, 구조화된 JSON 출력.
    """

    def __init__(self, llm_client, memory_dir: str):
        self.llm = llm_client
        self.memory_dir = memory_dir
        self._surfaced: set[str] = set()  # 이미 노출된 파일 추적

    def select(self, query: str, recently_used_tools: list[str] = None
               ) -> list[str]:
        """관련 memory 최대 5개 선택."""
        manifest = self._build_manifest()
        if not manifest:
            return []

        # 이미 노출된 파일 제외
        manifest = [m for m in manifest if m["filename"] not in self._surfaced]

        response = self.llm.create(
            model="sonnet",  # 저비용 semantic matching
            max_tokens=256,
            system=MEMORY_SELECTION_PROMPT,
            messages=[{
                "role": "user",
                "content": self._format_query(query, manifest, recently_used_tools)
            }],
            # 참고: Anthropic API에서는 tool_use 기반 structured output 사용.
            # 여기서는 추상화된 pseudocode
            response_format={"type": "json_schema", "schema": MEMORY_SCHEMA}
        )

        selected = response["selected_memories"]

        # 디스크에 실제 존재하는 파일만 반환 (hallucination 방지)
        validated = [f for f in selected if self._exists(f)]
        self._surfaced.update(validated)

        return validated[:5]

    def _build_manifest(self) -> list[dict]:
        """memory 디렉터리의 .md 파일에서 YAML frontmatter 추출."""
        import yaml
        manifest = []
        for filepath in Path(self.memory_dir).glob("*.md"):
            text = filepath.read_text()
            # --- markers 사이의 YAML frontmatter 파싱
            if text.startswith("---"):
                end = text.find("---", 3)
                if end != -1:
                    frontmatter = yaml.safe_load(text[3:end])
                    manifest.append({
                        "filename": filepath.name,
                        "description": frontmatter.get("description", filepath.stem),
                        "name": frontmatter.get("name", filepath.stem),
                    })
        return manifest

    def _format_query(self, query, manifest, tools) -> str:
        parts = [f"Query: {query}"]
        parts.append("Available memories:")
        for m in manifest:
            parts.append(f"  {m['filename']} -- {m['description']}")
        if tools:
            parts.append(f"Recently used tools: {', '.join(tools)}")
        return "\n".join(parts)

# --- 사용 예시 ---

def build_my_harness():
    assembler = SystemPromptAssembler()

    # Static sections (cache-safe)
    assembler.register_static("identity", lambda:
        "You are MyAssistant, a coding helper.")
    assembler.register_static("doing_tasks", lambda:
        ANTI_GOLD_PLATING_INSTRUCTIONS)
    assembler.register_static("actions", lambda:
        REVERSIBILITY_FRAMEWORK)
    assembler.register_static("tools", lambda:
        TOOL_USAGE_GUIDELINES)

    # Dynamic sections
    assembler.register_dynamic("memory", lambda:
        load_relevant_memories())
    assembler.register_dynamic("env_info", lambda:
        compute_env_info())

    # DANGEROUS uncached -- 이유 필수!
    assembler.register_dynamic("mcp", lambda:
        get_mcp_instructions(),
        cached=False,
        reason="MCP servers connect/disconnect between turns")

    return assembler.assemble()
```

### 이 pseudocode에서 배울 핵심 패턴

1. **이원 cache 체계**: `register_static` vs `register_dynamic(cached=False)`. 후자는 이유를 강제한다.
2. **Memoization with selective invalidation**: 전체 초기화(`/clear`) vs 부분 초기화(특정 section만).
3. **Dynamic boundary marker**: 이 위치를 기준으로 prompt caching의 scope가 나뉜다.
4. **Memory selection의 보수적 기준**: "확실할 때만 포함", "API docs는 제외하되 warnings는 포함".
5. **Disk validation**: LLM이 선택한 파일명이 실제로 존재하는지 확인 (hallucination 방지).

---

## Key Takeaways

- **실제 프롬프트 텍스트의 구체성**: "Three similar lines of code is better than a premature abstraction" 같은 구체적 시나리오가 추상적 규칙보다 LLM 행동 제어에 효과적이다.
- **Internal/External 프롬프트 분화**: 같은 제품이라도 사용자 세그먼트별로 다른 프롬프트를 제공하고, 내부에서 A/B 테스트 후 외부로 확대하는 전략이 유효하다.
- **Scratchpad 시스템**: 세션별 임시 디렉터리는 permission bypass, session isolation, 프로젝트 오염 방지를 동시에 해결한다.
- **Memory Selection**: 별도 LLM(Sonnet)으로 semantic matching, 최대 5개/256 tokens의 저비용 전략. "API docs 제외, warnings 포함" 같은 세밀한 규칙이 품질을 결정한다.
- **Edit-after-Read 제약**: 프롬프트 지시와 tool-level enforcement의 이중 강제가 가장 강력한 행동 제어 방법이다.
- **계층적 프롬프트 조립**: 모듈화된 sections, feature-gated 포함, cache-safe/uncached 분리는 프로덕션 LLM 앱의 프롬프트가 한 페이지를 넘는 순간 필수가 된다.
- **Reversibility and Blast Radius**: 이진 분류가 아닌 스펙트럼으로 위험을 판단하게 만드는 프레임워크. 모든 시나리오 열거 대신 원칙 + 예시 조합이 효과적이다.
- **Cache 최적화 기법**: "When the user specifies..." 패턴처럼, 프롬프트 텍스트를 불변으로 유지하면서 런타임에 자연스럽게 no-op이 되게 만드는 기법이 prompt cache 효율을 극대화한다.
