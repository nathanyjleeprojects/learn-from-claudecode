<!--
tags: architecture/terminal-ui, architecture/react-ink, workflow/vim-mode, llm-application/cli-ux
keywords: react-ink, terminal-ui, ink-components, react-hooks, vim-mode, useTextInput, useVimInput, usePasteHandler, chalk, streaming-render, concurrent-rendering
related_files: 01_architecture_overview.md, 02_query_engine_core.md, 08_streaming_concurrency.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 13. Terminal UI (React + Ink)

> React 19 + Ink로 구축된 터미널 UI: 140개 컴포넌트, 80개 hooks, vim mode. 웹 앱과 동일한 React 패턴을 터미널에서.

## Keywords
react-ink, terminal-ui, ink-components, react-hooks, vim-mode, streaming-render, concurrent-rendering, chalk, useTextInput, useVimInput, usePasteHandler, design-system, REPL-screen

## 왜 React + Ink인가

터미널 UI 프레임워크 선택지:
- **readline / blessed**: 전통적, 명령형
- **React + Ink**: 선언적, 컴포넌트 기반
- **직접 ANSI escape**: 최대 제어, 최대 복잡도

Claude Code가 React를 선택한 이유:

1. **Streaming 중 UI 업데이트**: 텍스트가 흘러나오면서 tool 상태가 바뀌고, permission 다이얼로그가 뜨는 **동시 업데이트**를 선언적으로 처리
2. **140개 컴포넌트 규모**: 이 규모에서 명령형 UI는 관리 불가능
3. **React Compiler**: 자동 최적화된 re-render로 터미널 깜빡임 방지
4. **React 생태계**: hooks, context, concurrent rendering 모두 활용

**LLM 엔지니어링 인사이트**: LLM 응답의 streaming은 본질적으로 **연속적 상태 변화**다. React의 선언적 렌더링 모델이 이를 가장 자연스럽게 표현한다. `<StreamingText content={partialResponse} />` 하나로 끝.

## Component Architecture

### 디렉토리 구조

```
src/components/
├── design-system/      # Box, Text, Spinner 등 기본 요소
├── messages/            # 메시지 렌더링 (user, assistant, tool, error)
├── permissions/         # Permission request 다이얼로그
├── input/              # 텍스트 입력, 자동완성
├── status/             # 상태바, progress indicator
└── [기능별 컴포넌트]    # ~140개
```

### 주요 컴포넌트

| Component | 역할 |
|-----------|------|
| `REPL.tsx` | 메인 화면 — 입력, 메시지 목록, 상태 |
| `Message.tsx` | 메시지 렌더링 (syntax highlighting 포함) |
| `ToolUseBlock.tsx` | Tool 호출/결과 표시 |
| `PermissionRequest.tsx` | "이 작업을 허용하시겠습니까?" 다이얼로그 |
| `ThinkingIndicator.tsx` | Extended thinking 진행 표시 |
| `Spinner.tsx` | 로딩 표시 |

## Hooks: 80개의 Custom Hook

### Input Handling

```typescript
useTextInput()       // 기본 텍스트 입력 + 히스토리 탐색
useVimInput()        // Vim 키바인딩 (normal/insert/visual mode)
useInputBuffer()     // 입력 버퍼 관리 (멀티라인)
usePasteHandler()    // 붙여넣기 (이미지 포함) 처리
```

### Permission & Tool

```typescript
useCanUseTool()              // Tool permission 체크
useIDEIntegration()          // IDE 연결 상태
useDiffInIDE()               // IDE에서 diff 표시
useMcpConnectivityStatus()   // MCP 서버 상태
```

### Session & State

```typescript
useSessionBackgrounding()    // 세션 백그라운드/포그라운드 전환
useRemoteSession()           // Bridge/remote 세션 관리
useAssistantHistory()        // 대화 히스토리 탐색
useSkillsChange()            // Skill 변경 감지
useManagePlugins()           // 플러그인 lifecycle 관리
```

### Notification

```typescript
useToast()                   // Toast 알림
useRateLimitNotice()         // Rate limit 경고
useDeprecationWarning()      // 기능 폐지 알림
```

## Vim Mode

`src/vim/`이 완전한 vim 키바인딩을 구현한다:

```
Normal mode:
  h/j/k/l — 커서 이동
  i/a/o — Insert mode 전환
  dd — 줄 삭제
  yy — 줄 복사
  p — 붙여넣기
  / — 검색
  : — 명령 모드

Insert mode:
  일반 텍스트 입력
  ESC — Normal mode 복귀

Visual mode:
  v — 선택 시작
  y/d — 선택 영역 복사/삭제
```

`/vim` 명령으로 활성화/비활성화. 활성화 시 `useVimInput()` hook이 `useTextInput()`을 대체한다.

**LLM 엔지니어링 인사이트**: CLI 도구의 타겟 사용자가 개발자라면, vim 키바인딩 지원이 채택률에 큰 영향을 줄 수 있다. "개발자의 근육 기억"을 존중하는 것이 UX의 핵심이다.

## Streaming Render Performance

LLM 응답이 streaming으로 도착하면 매 delta마다 re-render가 발생한다. 최적화 전략:

1. **React Compiler**: 자동으로 불필요한 re-render를 제거
2. **Content batching**: 빠르게 도착하는 delta를 묶어서 한 번에 렌더링
3. **Selective re-render**: 변경된 컴포넌트만 re-render (React의 기본 동작)
4. **ANSI escape 최적화**: Ink가 변경된 부분만 터미널에 쓰기

### 터미널 렌더링 vs 웹 렌더링

| | 웹 (DOM) | 터미널 (Ink) |
|---|----------|-------------|
| 렌더 타겟 | DOM elements | ANSI escape sequences |
| 레이아웃 | CSS (Flexbox, Grid) | Flexbox (Yoga layout) |
| 색상 | CSS colors | Chalk (256 color + TrueColor) |
| 이벤트 | DOM events | stdin keypress |
| 스크롤 | 브라우저 기본 | 수동 구현 |

## Design System

`src/components/design-system/`에 터미널 전용 디자인 시스템:

```typescript
<Box flexDirection="column" padding={1} borderStyle="round">
  <Text bold color="green">Success</Text>
  <Text dimColor>Details here</Text>
</Box>
```

Ink의 `Box`와 `Text`가 터미널에서 CSS Flexbox와 유사한 레이아웃을 제공한다.

## 내 프로젝트에 적용하기

### LLM 앱의 UI 설계 원칙

**Step 1: Streaming-first UI 설계**

LLM 응답은 본질적으로 streaming이다. UI를 "완성된 응답을 표시"가 아니라 "점진적으로 도착하는 데이터를 표시"로 설계한다:

```typescript
// BAD: 응답 완료까지 대기
const response = await llm.complete(prompt)
render(response)

// GOOD: Streaming으로 점진적 렌더링
for await (const chunk of llm.stream(prompt)) {
  appendToDisplay(chunk)  // 도착하는 즉시 표시
}
```

**Step 2: Progress indicator는 필수**

LLM 호출은 수 초~수십 초 걸린다. 사용자가 "동작 중인지" 알 수 있어야 한다:

- **Thinking indicator**: Extended thinking 중일 때 표시 (예: "Thinking..." + elapsed time)
- **Tool execution status**: 어떤 tool이 실행 중인지 표시 (예: "Running: npm test")
- **Token counter**: 현재까지 사용된 token 수 (비용 인식)

**Step 3: Permission dialog 패턴**

Tool 실행 전 사용자 승인이 필요한 경우, 명확한 permission dialog:

```
┌─ Permission Request ──────────────────────┐
│ BashTool wants to run:                    │
│   npm install lodash                      │
│                                           │
│ [y] Allow  [n] Deny  [a] Always allow     │
└───────────────────────────────────────────┘
```

핵심: **무엇을 할 것인지** + **선택지** + **영구 허용 옵션**. "Always allow" 옵션이 없으면 매번 승인하는 피로감이 쌓인다.

**Step 4: 프레임워크 선택**

- **10개 미만 컴포넌트**: readline + chalk로 충분
- **10-50개 컴포넌트**: Ink (React for terminal) 고려
- **50개 이상 또는 복잡한 상태**: Ink + React hooks 필수

## 대안 비교

| UI 프레임워크 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **React + Ink** (Claude Code) | 선언적, 컴포넌트 재사용, streaming 자연스러움 | 번들 크기 증가, 학습 곡선 | 복잡한 터미널 UI |
| **blessed / blessed-contrib** | 풍부한 위젯, 대시보드 | 유지보수 중단, 무거움 | 대시보드형 UI |
| **readline + chalk** | 가볍고 단순 | 복잡한 상태 관리 어려움 | 간단한 CLI |
| **Web UI (Next.js 등)** | 가장 풍부한 UI 가능 | 브라우저 필요, 배포 복잡 | 웹 앱, 비개발자 대상 |

## Key Takeaways

- **React for terminal**: 140+ 컴포넌트 규모에서 선언적 UI 모델이 필수. Streaming 중 동시 업데이트를 자연스럽게 처리.
- **Hook 패턴의 재사용**: 80개 hook이 permission, session, input, notification을 관리. 웹 React와 동일한 composition 패턴.
- **Vim mode = 개발자 UX**: 타겟 사용자의 근육 기억을 존중하는 것이 채택률에 영향.
- **Streaming render 최적화**: React Compiler + content batching + selective re-render로 터미널 깜빡임 방지.
- **실전 적용 시**: Streaming-first로 설계하고, progress indicator와 permission dialog를 초기에 넣어라. 이 두 가지가 LLM 앱 UX의 80%를 결정한다.
