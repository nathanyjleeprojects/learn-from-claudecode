<!--
tags: agents/multi-agent, agents/sub-agent-spawning, agents/coordinator-pattern, agents/inter-agent-communication, llm-application/orchestration
keywords: multi-agent, sub-agent, AgentTool, TeamCreateTool, SendMessageTool, coordinator-mode, agent-spawning, worktree-isolation, resource-isolation, parallel-agents
related_files: 02_query_engine_core.md, 04_tool_system.md, 08_streaming_concurrency.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 10. Multi-Agent Orchestration

> Claude Code의 다중 에이전트 시스템: sub-agent spawning, team creation, inter-agent communication, coordinator mode, resource isolation.

## Keywords
multi-agent, sub-agent, AgentTool, TeamCreateTool, SendMessageTool, coordinator-mode, agent-spawning, worktree-isolation, resource-isolation, parallel-agents, InProcessTeammateTask, agent-lifecycle

## 왜 Multi-Agent가 필요한가

단일 에이전트의 한계:
- **Context window 공유**: 하나의 context에 모든 작업 정보를 넣으면 overflow 위험
- **직렬 실행**: 독립적인 작업도 순차적으로 처리
- **전문성 분산**: 하나의 에이전트가 모든 도메인을 처리하면 품질 저하
- **블로킹**: 한 작업이 오래 걸리면 다른 작업이 대기

Claude Code는 이를 **sub-agent spawning**과 **coordinator pattern**으로 해결한다.

## AgentTool: Sub-Agent Spawning

`src/tools/AgentTool/`이 새로운 에이전트를 생성한다.

### 동작 원리

```
Parent agent → AgentTool 호출 (prompt + 설정)
  ↓
새로운 agent 프로세스 생성:
  - 독립된 context window (conversation history 없음)
  - 사용 가능한 tool set 상속 (또는 커스텀)
  - 독립된 token budget
  ↓
Sub-agent가 작업 수행 (tool loop)
  ↓
결과를 parent agent에게 반환
  ↓
Parent agent가 결과를 자신의 context에 통합
```

### Agent 타입

시스템 프롬프트에 정의된 agent types:

| Type | 용도 | 특징 |
|------|------|------|
| `general-purpose` | 복잡한 멀티스텝 작업 | 모든 tool 사용 가능 |
| `Explore` | 코드베이스 탐색 | Read-only tools만, 빠름 |
| `Plan` | 구현 계획 설계 | Read-only + 계획 작성 |
| `statusline-setup` | 상태줄 설정 | Read + Edit만 |
| `claude-code-guide` | Claude Code 사용 가이드 | Read + Web tools |

### Worktree Isolation

Git worktree를 사용한 파일 시스템 격리:

```typescript
// isolation: "worktree" 옵션으로 활성화
{
  isolation: "worktree"
  // → 임시 git worktree 생성
  // → Agent가 격리된 파일 시스템 복사본에서 작업
  // → 변경 없으면 자동 정리
  // → 변경 있으면 worktree 경로 + 브랜치명 반환
}
```

**LLM 엔지니어링 인사이트**: Sub-agent가 파일을 수정할 때, parent agent의 작업 디렉토리와 충돌할 수 있다. Git worktree isolation은 이 문제를 우아하게 해결한다. Agent가 수정한 내용은 별도 브랜치에 있으므로, parent가 검토 후 merge할 수 있다.

## TeamCreateTool: 팀 에이전트

`src/tools/TeamCreateTool/`이 병렬로 작동하는 에이전트 팀을 생성한다.

### Team vs Individual Agent

| | AgentTool | TeamCreateTool |
|---|-----------|----------------|
| 실행 | 순차 (하나씩) | 병렬 (동시에) |
| 용도 | 단일 복잡 작업 | 다수의 독립 작업 |
| 결과 | 완료 시 반환 | 각 member 완료 시 개별 반환 |
| 관리 | 자동 lifecycle | TeamDeleteTool로 수동 관리 |

### Team 패턴 예시

```
사용자: "이 3개 파일을 각각 리팩토링해줘"
  ↓
Coordinator가 TeamCreateTool로 3개 agent 생성:
  Agent A: "file1.ts 리팩토링" (worktree isolation)
  Agent B: "file2.ts 리팩토링" (worktree isolation)
  Agent C: "file3.ts 리팩토링" (worktree isolation)
  ↓
3개 agent가 병렬로 작업
  ↓
각 agent 완료 시 결과를 coordinator에게 반환
  ↓
Coordinator가 결과를 종합하여 사용자에게 보고
```

## SendMessageTool: Inter-Agent Communication

`src/tools/SendMessageTool/`로 에이전트 간 메시지를 교환한다:

```
Parent → SendMessage(to: agentId, message: "진행 상황 알려줘")
  ↓
Sub-agent가 메시지를 수신하고 context에 추가
  ↓
Sub-agent가 응답 생성
  ↓
응답이 parent에게 반환
```

### 사용 사례

- **진행 상황 확인**: Parent가 오래 걸리는 sub-agent에게 상태 질문
- **추가 지시**: 작업 도중 방향 수정
- **Context 공유**: 한 agent의 발견을 다른 agent에게 전달
- **Agent 재활용**: 이전에 spawned된 agent에게 추가 작업 위임

## Coordinator Mode

`src/coordinator/coordinatorMode.ts`가 전체 에이전트 팀을 관리한다. Feature flag `COORDINATOR_MODE`로 gating.

### Coordinator의 역할

```
사용자 요청 수신
  ↓
Coordinator가 작업 분석:
  - 독립적인 sub-task 식별
  - 의존 관계 파악
  - 적절한 agent type 선택
  ↓
Team 또는 Individual agent 생성
  ↓
진행 모니터링 (SendMessage)
  ↓
결과 종합 및 사용자 보고
```

### Coordinator System Prompt

Coordinator mode일 때 특화된 시스템 프롬프트가 적용된다:
- "작업을 분해하여 적절한 agent에게 위임하라"
- "독립적인 작업은 병렬로 실행하라"
- "의존적인 작업은 순서를 지켜라"
- "각 agent의 결과를 종합하여 사용자에게 보고하라"

## Task System과의 연동

`src/tasks/`의 task 타입들이 agent lifecycle을 관리한다:

| Task Type | 위치 | 용도 |
|-----------|------|------|
| `LocalAgentTask` | `LocalAgentTask/` | 로컬 sub-agent |
| `RemoteAgentTask` | `RemoteAgentTask/` | 원격 agent |
| `InProcessTeammateTask` | `InProcessTeammateTask/` | 병렬 teammate |
| `DreamTask` | `DreamTask/` | 백그라운드 "dreaming" |

### Agent as Background Task

Sub-agent를 background로 실행할 수 있다:

```
AgentTool(run_in_background: true)
  → Task로 등록
  → Parent agent가 다른 작업 진행
  → 완료 시 자동 알림
  → TaskOutput으로 결과 확인
```

## Resource Isolation 전략

| 리소스 | 격리 방법 |
|--------|----------|
| **Context window** | 각 agent가 독립된 context |
| **파일 시스템** | Git worktree isolation |
| **Token budget** | Task-level budget 설정 가능 |
| **Tool set** | Agent 생성 시 사용 가능 tool 제한 |
| **Permission** | Parent의 permission 상속 또는 커스텀 |

## Multi-Agent 실전 패턴

### Pattern 1: Explore → Plan → Execute

```
1. Explore agent (read-only) → 코드베이스 탐색
2. Plan agent (read-only) → 구현 계획 수립
3. Execute agent (full tools) → 코드 작성
```

각 단계가 독립된 context에서 동작하므로, 탐색 결과가 실행 context를 오염시키지 않는다.

### Pattern 2: Parallel Research

```
Agent A: "인증 시스템 코드 분석"
Agent B: "데이터베이스 스키마 분석"
Agent C: "테스트 패턴 분석"
→ 3개 결과를 parent가 종합하여 리팩토링 계획 수립
```

### Pattern 3: Divide & Conquer

```
"10개 파일을 수정해야 할 때"
→ 3-4개 agent에게 분배
→ 각각 worktree isolation으로 충돌 없이 작업
→ 결과를 순차적으로 main branch에 merge
```

## 내 프로젝트에 적용하기

### 내 프로젝트에 Multi-Agent 도입하기

**Step 1: 정말 multi-agent가 필요한지 판단하기**

Multi-agent는 overhead가 크다. 다음 조건을 **모두** 만족할 때만 도입:

- 작업이 **진정으로 독립적** (파일, context, 결과가 겹치지 않음)
- 단일 context window로는 **부족**한 규모의 작업
- 병렬 실행으로 **체감 시간이 의미 있게 줄어듦**

대부분의 경우, 단일 agent + 잘 설계된 prompt가 multi-agent보다 결과가 좋다.

**Step 2: "Synthesize, never delegate lazily" 원칙**

Coordinator에서 worker로 작업을 위임할 때 가장 흔한 실수: 모호한 지시를 보내는 것. Worker는 coordinator의 conversation history를 볼 수 없다. 따라서:

```
// BAD: Worker가 context를 모름
"파일을 리팩토링해줘"

// GOOD: Self-contained spec with exact file paths
"다음 파일을 리팩토링하라:
 - /src/utils/auth.ts: 함수 validateToken()을 pure function으로 변경
 - 변경 이유: side effect 제거로 테스트 용이성 확보
 - 제약: 외부 인터페이스(export signature) 변경 금지
 - 완료 기준: 기존 테스트 통과 + 새 unit test 1개 추가"
```

**Step 3: Worktree isolation 패턴 적용**

여러 agent가 파일을 수정한다면 반드시 격리:

```bash
# Agent별 git worktree 생성
git worktree add /tmp/agent-a-work -b agent-a-branch
git worktree add /tmp/agent-b-work -b agent-b-branch

# 각 agent는 자신의 worktree에서만 작업
# 완료 후 coordinator가 검토 → merge
```

**Step 4: 결과 종합은 coordinator가 직접**

Worker 결과를 단순 concat하지 말고, coordinator가 결과를 읽고 종합하여 최종 응답을 생성한다. 이 종합 단계가 multi-agent의 품질을 결정한다.

## 대안 비교

| 패턴 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **단일 Agent** | 단순, 디버깅 쉬움, context 일관성 | Context window 한계 | 대부분의 작업 |
| **Sub-agent spawning** (Claude Code) | Context isolation, 병렬 가능 | Coordinator 품질에 의존 | 독립적 sub-task가 명확할 때 |
| **Crew/Team 패턴** (CrewAI) | Role 기반 분업, 직관적 | Role 정의가 모호하면 품질 저하 | Role이 명확한 워크플로우 |
| **Graph 기반** (LangGraph) | 복잡한 분기/합류, 시각화 | 오버엔지니어링 위험 | 복잡한 조건 분기가 많을 때 |

## Key Takeaways

- **Sub-agent로 context isolation**: 각 agent가 독립된 context window를 가지므로, 한 작업의 정보가 다른 작업의 context를 오염시키지 않는다.
- **Git worktree로 파일 격리**: 여러 agent가 동시에 파일을 수정해도 충돌하지 않는다.
- **Agent type 분리**: Read-only agent(Explore)와 full-power agent(general-purpose)를 구분하면 안전성과 효율성이 높아진다.
- **Background agent + Task system**: Agent를 background로 실행하고 TaskOutput으로 결과를 확인하는 비동기 패턴.
- **Coordinator pattern**: 복잡한 작업을 자동으로 분해하고 적절한 agent에게 위임. 단, 이 패턴은 overhead가 있으므로 단순한 작업에는 과잉이다.
- **"Synthesize, never delegate lazily"**: Worker에게 보내는 spec은 self-contained이어야 한다. Exact file paths, 변경 이유, 완료 기준을 명시하라. Worker는 coordinator의 대화를 볼 수 없다.
