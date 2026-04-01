<!--
tags: architecture/command-system, workflow/slash-commands, architecture/extensibility, llm-application/cli-design
keywords: command-system, slash-commands, PromptCommand, LocalCommand, LocalJSXCommand, command-registry, argument-parsing, command-routing, command-vs-tool
related_files: 04_tool_system.md, 03_prompt_engineering.md, 13_ui_terminal.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 12. Command System

> 87개 slash command의 설계: PromptCommand vs LocalCommand, command routing, tool과의 구분.

## Keywords
command-system, slash-commands, PromptCommand, LocalCommand, LocalJSXCommand, command-registry, command-routing, argument-parsing, command-vs-tool, getPromptForCommand

## Command vs Tool: 근본적 차이

| | Command | Tool |
|---|---------|------|
| **호출자** | 사용자 (`/commit`) | LLM (`tool_use` block) |
| **실행 시점** | 사용자 입력 즉시 | LLM 응답 도중 |
| **목적** | 사용자 워크플로우 | LLM 에이전트 행동 |
| **예시** | `/commit`, `/review`, `/cost` | BashTool, FileReadTool |

일부 command는 내부적으로 LLM에게 특화된 프롬프트를 보내서 tool loop를 트리거한다. 이 경우 command는 "프롬프트 + tool set을 미리 설정한 shortcut"이다.

## Command Types

### PromptCommand: LLM에게 전문화된 프롬프트 전송

```typescript
// 예: /commit
{
  type: 'prompt',
  getPromptForCommand(args) {
    return {
      prompt: "Git 상태를 확인하고 적절한 커밋 메시지를 생성해주세요...",
      allowedTools: ['BashTool', 'FileReadTool', 'GrepTool'],
      systemPromptAppend: "커밋 메시지 작성 가이드라인..."
    }
  }
}
```

PromptCommand의 핵심: **프롬프트 + 허용 tool set + 추가 시스템 프롬프트**를 하나로 패키징.

대표적 PromptCommand:
- `/commit` — git commit 워크플로우
- `/review` — 코드 리뷰
- `/pr_comments` — PR 코멘트 처리
- `/simplify` — 코드 단순화

### LocalCommand: 프로세스 내 실행, 텍스트 반환

```typescript
// 예: /cost
{
  type: 'local',
  execute() {
    return `Total cost: $${getTotalCost()}`
  }
}
```

LLM 호출 없이 즉시 실행. 대표적:
- `/cost` — 비용 현황
- `/version` — 버전 정보
- `/clear` — 대화 초기화
- `/compact` — context 압축

### LocalJSXCommand: 프로세스 내 실행, React JSX 반환

```typescript
// 예: /doctor
{
  type: 'local-jsx',
  execute() {
    return <DoctorScreen />
  }
}
```

복잡한 UI가 필요한 명령. 대표적:
- `/doctor` — 환경 진단 화면
- `/install` — 설치 가이드
- `/config` — 설정 관리 UI

## Command Registry

`src/commands.ts` (~25K lines)에 모든 command가 등록된다. Feature flag와 환경에 따라 조건부로 로드:

```typescript
// 항상 사용 가능
const commitCommand = require('./commands/commit').default
const reviewCommand = require('./commands/review').default

// Feature flag로 gating
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice').default
  : null

// 환경 조건
const bridgeCommand = feature('BRIDGE_MODE')
  ? require('./commands/bridge').default
  : null
```

## Command Processing Flow

```
사용자 입력: "/commit -m 'Fix bug'"
  ↓
processUserInput() → slash command 감지
  ↓
processSlashCommand():
  1. Command name 파싱: "commit"
  2. Arguments 파싱: ["-m", "Fix bug"]
  3. Command registry에서 찾기
  4. Command type에 따라 분기
  ↓
PromptCommand?
  → getPromptForCommand(args) 호출
  → 반환된 prompt를 LLM에게 전송
  → allowedTools로 tool set 제한
  → 결과를 사용자에게 표시

LocalCommand?
  → execute(args) 즉시 호출
  → 반환된 텍스트 표시

LocalJSXCommand?
  → execute(args) 호출
  → 반환된 JSX를 Ink로 렌더링
```

## 87개 Command 카테고리

| Category | Examples | Count |
|----------|---------|-------|
| **Git & Code Review** | /commit, /review, /pr_comments, /diff, /branch | ~15 |
| **Settings & Config** | /config, /doctor, /theme, /keybindings | ~10 |
| **Authentication** | /login, /logout, /oauth-refresh | ~5 |
| **Workflow** | /memory, /skills, /tasks, /resume, /share | ~15 |
| **Development** | /mcp, /plugin, /agent, /bridge | ~10 |
| **Navigation** | /compact, /clear, /cost, /version | ~10 |
| **Mode** | /vim, /fast, /plan | ~5 |
| **Internal/Debug** | 다양한 내부 도구 | ~17 |

## LLM 엔지니어링 인사이트

**PromptCommand 패턴의 가치**: `/commit`이라는 단일 명령 뒤에는 "git status 확인 → 변경사항 분석 → 적절한 커밋 메시지 생성 → 커밋 실행"이라는 복잡한 워크플로우가 숨어 있다. 이를 사용자에게 매번 프롬프트로 설명하게 하는 대신, **자주 쓰는 워크플로우를 command로 패키징**하면 UX가 크게 개선된다.

**Command = 도메인 전문가의 프롬프트**: `/review`는 코드 리뷰 전문가가 작성한 프롬프트와 적절한 tool set을 패키징한 것이다. 사용자가 "코드 리뷰해줘"라고 말하는 것보다, `/review`가 더 일관되고 높은 품질의 결과를 제공한다.

## 내 프로젝트에 적용하기

### PromptCommand 패턴 적용하기

**Step 1: 반복되는 워크플로우 식별**

팀에서 자주 쓰는 프롬프트 패턴을 목록화한다:
- "이 PR의 변경사항을 리뷰해줘" → `/review`
- "이 코드의 테스트를 작성해줘" → `/test`
- "이 에러를 디버깅해줘" → `/debug`

**Step 2: PromptCommand 구조 만들기**

```typescript
interface PromptCommand {
  name: string
  description: string
  getPrompt(args: string[]): {
    prompt: string              // LLM에게 보낼 메시지
    allowedTools: string[]      // 사용 가능한 tool 제한
    systemPromptAppend?: string // 추가 시스템 프롬프트 (도메인 전문성)
  }
}

// 예시: /review command
const reviewCommand: PromptCommand = {
  name: 'review',
  description: '현재 브랜치의 변경사항을 코드 리뷰',
  getPrompt(args) {
    return {
      prompt: 'git diff main...HEAD를 확인하고 코드 리뷰를 수행하라.',
      allowedTools: ['BashTool', 'FileReadTool', 'GrepTool'],
      systemPromptAppend: '보안 취약점, 성능 문제, 타입 안전성에 집중하라.'
    }
  }
}
```

**Step 3: Command registry 구현**

```typescript
const commands = new Map<string, Command>()
commands.set('review', reviewCommand)
commands.set('cost', costCommand)  // LocalCommand

// 입력 처리
if (input.startsWith('/')) {
  const [name, ...args] = input.slice(1).split(' ')
  const cmd = commands.get(name)
  if (cmd.type === 'prompt') return await sendToLLM(cmd.getPrompt(args))
  if (cmd.type === 'local') return cmd.execute(args)
}
```

**Step 4: 도메인 전문가의 프롬프트를 command로 패키징**

핵심 가치: `/review`가 "코드 리뷰해줘"보다 일관되게 좋은 결과를 내는 이유는, `systemPromptAppend`에 도메인 전문가가 작성한 가이드라인이 포함되기 때문이다. 팀의 코딩 컨벤션, 리뷰 체크리스트 등을 command에 내장하라.

## Key Takeaways

- **Command는 "pre-configured prompt + tool set" shortcut**: 자주 쓰는 워크플로우를 패키징하여 일관된 품질과 편의성 제공.
- **3가지 command type**: PromptCommand(LLM에게 전송), LocalCommand(즉시 실행), LocalJSXCommand(UI 렌더링).
- **Feature-gated registry**: 새 command를 flag로 제어하여 안전하게 롤아웃.
- **Command vs Tool 분리**: 사용자 워크플로우(command)와 LLM 행동(tool)을 명확히 구분.
- **실전 적용 시**: 팀에서 반복되는 프롬프트 패턴을 PromptCommand로 패키징하라. `prompt + allowedTools + systemPromptAppend` 세 요소가 command의 핵심이다. 도메인 전문가의 지식을 command에 내장하면 팀 전체의 LLM 활용 품질이 올라간다.
