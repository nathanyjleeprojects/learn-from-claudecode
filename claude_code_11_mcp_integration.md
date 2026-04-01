<!--
tags: agents/mcp-protocol, agents/tool-discovery, architecture/mcp-client, architecture/mcp-server, llm-application/protocol-integration
keywords: mcp, model-context-protocol, mcp-client, mcp-server, tool-discovery, resource-browsing, mcp-transport, stdio, sse, http, mcp-auth, deferred-tools, ToolSearchTool
related_files: 04_tool_system.md, 10_multi_agent.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 11. MCP Integration

> Model Context Protocol (MCP) 구현: Claude Code가 MCP client이자 MCP server로 동작하는 방식. Tool discovery, resource browsing, transport handling.

## Keywords
mcp, model-context-protocol, mcp-client, mcp-server, tool-discovery, resource-browsing, mcp-transport, stdio-transport, sse-transport, http-transport, mcp-auth, deferred-tools, ToolSearchTool, McpAuthTool, dynamic-tool-loading

## MCP란 무엇인가

Model Context Protocol (MCP)은 Anthropic이 제안한 오픈 표준으로, AI 에이전트가 외부 도구와 리소스에 접근하는 방식을 표준화한다. REST API가 웹 서비스의 표준이 된 것처럼, MCP는 **AI 에이전트의 도구 접근 표준**이 되려 한다.

Claude Code는 MCP를 **양방향**으로 지원한다:
- **MCP Client**: 외부 MCP 서버의 도구/리소스를 사용
- **MCP Server**: 자신의 도구를 다른 AI 에이전트에게 노출

## MCP Client Implementation

`src/services/mcp/client.ts`가 MCP 클라이언트를 구현한다.

### 연결 흐름

```
MCP 서버 설정 (설정 파일 또는 /mcp 명령)
  ↓
Transport 선택 (stdio / HTTP / SSE / WebSocket)
  ↓
Connection 수립 + Handshake
  ↓
Tool Discovery (사용 가능한 도구 목록 획득)
  ↓
Resource Discovery (사용 가능한 리소스 목록 획득)
  ↓
Tool/Resource를 Claude Code의 tool registry에 등록
```

### Transport Types

| Transport | 사용 사례 | 특징 |
|-----------|----------|------|
| **stdio** | 로컬 프로세스 | 가장 단순, 별도 프로세스 실행 |
| **http** | 원격 서버 | Streamable HTTP, POST/GET |
| **sse** | Legacy 원격 | Server-Sent Events, 단방향 스트림 |
| **ws** | 실시간 양방향 | WebSocket persistent connection |
| **sse-ide** / **ws-ide** | IDE 연동 | Bridge 전용 transport |
| **claudeai-proxy** | Claude.ai | OAuth bearer token 자동 부착 |
| **sdk** | In-process | 프로세스 내 직접 연결 |

### 구체적 수치

```typescript
DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000  // ~27.8시간 (사실상 무한)
MAX_MCP_DESCRIPTION_LENGTH = 2048           // Tool description 최대 길이
MCP_REQUEST_TIMEOUT_MS = 60_000             // 개별 요청 timeout
MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000     // Auth 캐시 15분
CONNECTION_TIMEOUT_MS = 30_000              // 연결 timeout (MCP_TIMEOUT env로 오버라이드)
BATCH_SIZE = 3 (local) / 20 (remote)       // 동시 연결 수
STDERR_CAP = 64MB                           // stderr 축적 상한
```

### Tool Discovery

MCP 서버에 연결하면 사용 가능한 도구 목록을 자동으로 가져온다:

```
MCP Server → tools/list → [
  { name: "query_db", description: "SQL 쿼리 실행", inputSchema: {...} },
  { name: "send_email", description: "이메일 전송", inputSchema: {...} },
]
```

이 도구들은 Claude Code의 built-in tools와 동일하게 취급된다: LLM이 호출할 수 있고, permission 체크를 거치며, 결과가 tool_result로 반환된다.

**LLM 엔지니어링 인사이트**: MCP의 핵심 가치는 **tool을 코드 수정 없이 추가/제거**할 수 있다는 것이다. 새 데이터베이스 연결이 필요하면 MCP 서버를 설정하기만 하면 된다. Tool의 hot-plug.

### Resource Browsing

도구 외에도 MCP는 **리소스**(정적 데이터)를 노출한다:

```
MCP Server → resources/list → [
  { uri: "db://schema", name: "Database Schema" },
  { uri: "docs://api-reference", name: "API Reference" },
]
```

리소스는 LLM의 context에 주입되어 참고 자료로 활용된다.

## Deferred Tools: Lazy Loading for Tools

42개 built-in tools + N개 MCP tools를 모두 시스템 프롬프트에 포함하면 token이 폭발한다. `ToolSearchTool`이 이를 해결한다:

### Deferred Tool 패턴

```
시작 시:
  - Built-in tools (42개) → 시스템 프롬프트에 포함
  - MCP tools → 이름만 system-reminder에 나열

LLM이 MCP tool 사용 필요:
  → ToolSearchTool 호출 (query: "database", max_results: 5)
  → 매칭되는 MCP tool의 full schema 반환
  → 이제 LLM이 해당 tool 호출 가능
```

**LLM 엔지니어링 인사이트**: "Deferred tools"는 tool이 많을 때 **시스템 프롬프트를 작게 유지하면서도 많은 tool을 제공하는** 우아한 패턴이다. Tool 이름만 알려주고, 실제 schema는 필요할 때만 로드한다. API 세계의 "lazy loading"과 동일한 개념이다.

## MCP Server Mode

`src/entrypoints/mcp.ts`에서 Claude Code 자체를 MCP 서버로 실행할 수 있다. 이 모드에서는 다른 AI 에이전트가 Claude Code의 도구를 사용할 수 있다.

### 노출되는 기능

| 카테고리 | 내용 |
|----------|------|
| **Tools** | Claude Code의 built-in tools 하위 집합 |
| **Resources** | 프로젝트 구조, 아키텍처 정보 |
| **Prompts** | 코드 분석, 설명 등의 프롬프트 템플릿 |

### MCP 서버 패키지

이 repo에는 별도의 `mcp-server/` 디렉토리에 독립적인 MCP 서버가 포함되어 있다:

```
mcp-server/
├── src/              # MCP 서버 소스
├── package.json      # 독립 패키지
└── README.md         # 설정 가이드
```

**노출하는 Tools (8개)**:
- `list_tools`, `list_commands` — 도구/명령 목록
- `get_tool_source`, `get_command_source` — 소스 코드
- `read_source_file` — 파일 읽기
- `search_source` — 소스 검색
- `list_directory` — 디렉토리 탐색
- `get_architecture` — 아키텍처 정보

**Prompts (5개)**:
- `explain_tool`, `explain_command` — 도구/명령 설명
- `architecture_overview` — 아키텍처 개요
- `how_does_it_work` — 특정 기능 설명
- `compare_tools` — 도구 비교

## MCP Auth

`src/tools/McpAuthTool/`이 MCP 서버 인증을 처리한다:

- OAuth 2.0 flow 지원
- API key 기반 인증 지원
- 토큰 저장 및 자동 refresh
- Server approval flow (`src/services/mcpServerApproval.tsx`)

### Server Approval

새 MCP 서버 연결 시 사용자 승인을 요구한다:
- 서버 URL과 노출되는 도구 목록을 표시
- 사용자가 명시적으로 승인해야 연결
- 승인된 서버 목록은 설정에 저장

## MCP 설정

```json
// .claude/settings.json 또는 /mcp 명령으로 설정
{
  "mcpServers": {
    "my-database": {
      "command": "npx",
      "args": ["my-db-mcp-server"],
      "env": { "DATABASE_URL": "..." }
    },
    "remote-api": {
      "url": "https://api.example.com/mcp",
      "transport": "http"
    }
  }
}
```

## Connectivity Monitoring

`useMcpConnectivityStatus` hook이 MCP 서버 연결 상태를 실시간으로 추적한다:

- 연결 끊김 감지 → 자동 재연결 시도
- 서버 응답 없음 → timeout 후 비활성화
- 재연결 성공 → tool 목록 갱신

## 내 프로젝트에 적용하기

### MCP 서버 만들기 시작점

**Step 1: Minimal MCP server (stdio transport)**

가장 단순한 MCP 서버부터 시작한다. `@modelcontextprotocol/sdk`를 사용:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { z } from 'zod'

const server = new McpServer({ name: 'my-tools', version: '1.0.0' })

// Tool 등록
server.tool('query_db', { sql: z.string() }, async ({ sql }) => {
  const result = await db.query(sql)
  return { content: [{ type: 'text', text: JSON.stringify(result) }] }
})

// stdio transport로 실행
const transport = new StdioServerTransport()
await server.connect(transport)
```

**Step 2: Claude Code에 연결**

```json
// .claude/settings.json
{
  "mcpServers": {
    "my-tools": {
      "command": "node",
      "args": ["./my-mcp-server.js"]
    }
  }
}
```

**Step 3: Deferred tools 패턴 이해하고 활용**

Tool이 10개 이상이면 deferred tools를 고려한다. 핵심 아이디어:
- 시스템 프롬프트에는 tool **이름만** 나열
- LLM이 tool을 사용하려 할 때 `ToolSearchTool`로 full schema를 fetch
- 시스템 프롬프트 token 수를 수천 개 절약

직접 구현할 때: tool registry를 "summary" (이름 + 한 줄 설명)와 "full" (complete JSON schema) 두 단계로 나누면 된다.

**Step 4: Resource도 노출하기**

Tool 외에 정적 context를 MCP resource로 제공하면 LLM이 더 정확한 답을 낸다:

```typescript
server.resource('schema', 'db://schema', async () => ({
  contents: [{ uri: 'db://schema', text: await getDbSchema(), mimeType: 'text/plain' }]
}))
```

## 대안 비교

| 접근 방식 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **MCP Server** | 표준 프로토콜, 다양한 client 지원 | 학습 곡선, 아직 초기 생태계 | 여러 AI agent에서 도구 공유 |
| **직접 Tool 구현** | 완전한 제어, 단순 | 앱에 종속, 재사용 어려움 | 단일 앱 내 도구 |
| **REST API + function calling** | 익숙한 패턴, 기존 인프라 활용 | Tool schema 수동 관리 | 기존 API가 있을 때 |
| **Plugin system** | 앱 내 확장성 | 표준 없음, 앱별 구현 | 앱 특화 확장 |

## Key Takeaways

- **MCP = Tool의 Hot-Plug**: 코드 수정 없이 외부 도구를 추가/제거. REST API가 서비스 연결의 표준인 것처럼, MCP는 AI 도구 연결의 표준이 되려 한다.
- **Deferred tools 패턴**: Tool 이름만 프롬프트에 포함하고, full schema는 필요할 때만 로드. 시스템 프롬프트 크기를 관리하는 핵심 기법.
- **양방향 MCP**: Client로서 외부 도구를 사용하면서, Server로서 자신의 도구를 노출. 에이전트 생태계의 상호운용성.
- **Multi-transport**: stdio(로컬), HTTP(원격), SSE(레거시), WebSocket(실시간) — 다양한 환경을 지원하는 유연한 transport 레이어.
- **Server approval**: 새 MCP 서버 연결에 사용자 승인 필수. 보안과 투명성의 균형.
- **실전 적용 시**: stdio transport MCP 서버부터 시작하라. 10줄 미만의 코드로 첫 tool을 노출할 수 있다. Tool이 많아지면 deferred tools 패턴으로 프롬프트 크기를 관리한다.
