<!--
tags: architecture/ide-integration, workflow/vscode-extension, architecture/bidirectional-ipc, llm-application/ide-bridge
keywords: ide-bridge, vscode, jetbrains, jwt-auth, bidirectional-messaging, session-management, permission-proxy, bridge-protocol, trusted-device, workspace-secret
related_files: 01_architecture_overview.md, 04_tool_system.md, 09_security_model.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 17. IDE Bridge

> VS Code / JetBrains와의 양방향 통합: JWT 인증, session management, permission proxy, bidirectional messaging.

## Keywords
ide-bridge, vscode-extension, jetbrains-plugin, jwt-auth, bidirectional-messaging, session-management, permission-proxy, bridge-protocol, trusted-device, workspace-secret, bridgeMain, replBridge

## Bridge Architecture

```
┌──────────────────┐         ┌──────────────────────┐
│   IDE Extension  │◄───────►│   Bridge Layer       │
│  (VS Code, JB)   │  JWT    │  (src/bridge/)       │
│                  │  Auth   │                      │
│  - UI rendering  │         │  - Session mgmt      │
│  - File watching │         │  - Message routing    │
│  - Diff display  │         │  - Permission proxy   │
└──────────────────┘         └──────────┬───────────┘
                                        │
                                        ▼
                              ┌──────────────────────┐
                              │   Claude Code Core   │
                              │  (QueryEngine, Tools) │
                              └──────────────────────┘
```

IDE extension이 "프론트엔드", Bridge가 "미들웨어", Claude Code Core가 "백엔드" 역할을 한다.

## Key Files

| File | 역할 |
|------|------|
| `bridgeMain.ts` | 메인 브릿지 루프 — 양방향 채널 시작 |
| `bridgeMessaging.ts` | 메시지 프로토콜 (직렬화/역직렬화) |
| `bridgePermissionCallbacks.ts` | Permission 요청을 IDE로 라우팅 |
| `bridgeApi.ts` | IDE에 노출되는 API surface |
| `bridgeConfig.ts` | Bridge 설정 |
| `replBridge.ts` | REPL 세션과 bridge 연결 |
| `jwtUtils.ts` | JWT 기반 인증 |
| `sessionRunner.ts` | Bridge session 실행 관리 |
| `createSession.ts` | 새 bridge session 생성 |
| `trustedDevice.ts` | 디바이스 신뢰 검증 |
| `workSecret.ts` | 워크스페이스 스코프 시크릿 |

## JWT Authentication

CLI와 IDE 간 통신은 JWT로 인증된다:

```
1. CLI가 JWT 토큰 생성 (로컬 secret key 사용)
2. IDE extension이 CLI에 연결 요청 + JWT
3. CLI가 JWT 검증
4. 검증 성공 → 양방향 채널 수립
5. 이후 모든 메시지에 JWT 포함
```

### Trusted Device

`trustedDevice.ts`가 디바이스 신뢰를 검증한다:
- 처음 연결 시 디바이스 fingerprint 저장
- 이후 연결 시 fingerprint 비교
- 불일치 시 사용자에게 알림

### Workspace Secret

`workSecret.ts`가 워크스페이스 단위의 secret을 관리:
- 워크스페이스별 고유 secret 생성
- IDE extension과 CLI가 동일한 secret을 공유해야 통신 가능
- Secret은 로컬 파일 시스템에 안전하게 저장

## Message Protocol

`bridgeMessaging.ts`가 메시지 형식을 정의한다:

### IDE → CLI 메시지

```typescript
// 사용자 입력
{ type: 'user_input', content: "코드를 리팩토링해줘" }

// Permission 응답
{ type: 'permission_response', granted: true, toolName: 'FileEdit' }

// 파일 변경 알림
{ type: 'file_changed', path: '/src/app.ts', content: "..." }
```

### CLI → IDE 메시지

```typescript
// LLM 응답 (streaming)
{ type: 'assistant_message', content: "리팩토링을 시작하겠습니다." }

// Tool 실행 알림
{ type: 'tool_use', tool: 'FileEdit', input: { file: "...", edit: "..." } }

// Permission 요청
{ type: 'permission_request', tool: 'BashTool', command: "npm test" }

// Diff 표시 요청
{ type: 'show_diff', file: '/src/app.ts', before: "...", after: "..." }
```

## Permission Proxy

`bridgePermissionCallbacks.ts`가 permission 요청을 IDE로 라우팅:

```
Tool 실행 → Permission check → 사용자 승인 필요
  ↓
CLI에서:
  터미널 모드? → 터미널에 permission dialog 표시
  Bridge 모드? → IDE로 permission 요청 전송
  ↓
IDE에서:
  VS Code → 알림 팝업 또는 인라인 UI로 승인 요청
  JetBrains → 유사한 UI로 승인 요청
  ↓
사용자 응답 → IDE → CLI로 전달
```

## Session Management

```
IDE 연결 → createSession() → 새 bridge session 생성
  ↓
sessionRunner.ts가 session 실행 관리:
  - 연결 상태 모니터링
  - 메시지 라우팅
  - 에러 복구
  ↓
연결 끊김 → session 일시 정지 → 재연결 시 복구
```

### Feature Flag

Bridge는 `BRIDGE_MODE` feature flag로 gating. IDE build에서만 포함되고, standalone CLI build에서는 완전히 제거된다 (dead code elimination).

## VS Code vs JetBrains 차이

| | VS Code | JetBrains |
|---|---------|-----------|
| Extension language | TypeScript | Kotlin/Java |
| 통신 방식 | stdio / WebSocket | stdio / WebSocket |
| Diff 표시 | VS Code Diff Editor | JetBrains Diff Viewer |
| Permission UI | VS Code Notification | JetBrains Dialog |
| File watching | VS Code FileSystemWatcher | JetBrains VFS |

핵심 프로토콜은 동일하지만, UI 통합 부분이 IDE마다 다르다.

## 내 프로젝트에 적용하기

### IDE 통합 설계 원칙

**Step 1: CLI-as-backend 패턴 채택**

LLM 로직을 IDE extension에 넣지 말고, CLI/서버에 집중한다. IDE extension은 thin client:

```
// Architecture
CLI (backend)          ←→  IDE Extension (frontend)
- LLM 호출                  - UI 렌더링
- Tool 실행                  - Permission dialog
- Session 관리               - Diff 표시
- Cost tracking              - File watching
```

이 패턴의 핵심 이점: CLI가 독립적으로 동작하므로 터미널 사용자와 IDE 사용자 모두 지원. IDE extension은 CLI의 기능을 그대로 활용.

**Step 2: Protocol-first design**

IDE별 UI 구현보다 메시지 프로토콜을 먼저 정의한다:

```typescript
// 메시지 프로토콜 정의 (IDE 무관)
type ClientToServer =
  | { type: 'user_input'; content: string }
  | { type: 'permission_response'; granted: boolean; toolName: string }
  | { type: 'file_changed'; path: string }

type ServerToClient =
  | { type: 'assistant_message'; content: string; streaming: boolean }
  | { type: 'permission_request'; tool: string; description: string }
  | { type: 'show_diff'; file: string; before: string; after: string }
```

프로토콜이 안정적이면 VS Code, JetBrains, Vim plugin 등을 독립적으로 개발할 수 있다.

**Step 3: Local auth (JWT)**

로컬 프로세스 간 통신이라도 인증을 넣는다. 악의적 프로세스가 CLI에 명령을 보내는 것을 방지:

```typescript
// CLI 시작 시 random secret 생성 → JWT signing
const secret = crypto.randomBytes(32)
const token = jwt.sign({ pid: process.pid, ts: Date.now() }, secret)
// IDE extension에게 token 전달 (파일 또는 환경 변수)
```

**Step 4: Feature flag로 bridge 코드 분리**

IDE 통합 코드가 standalone CLI 빌드에 포함되지 않도록 build-time flag로 제거한다. 이는 번들 크기와 시작 시간 모두에 영향.

## 대안 비교

| 통합 패턴 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **CLI-as-backend** (Claude Code) | CLI 독립 동작, 프로토콜 통일 | IPC 구현 필요 | CLI + IDE 모두 지원 |
| **Extension-native** (Copilot) | IDE 깊은 통합, 빠른 응답 | IDE별 전체 구현, CLI 없음 | IDE 전용 도구 |
| **Language Server Protocol** | 표준, 많은 IDE 지원 | LLM 앱과 모델 불일치 | 코드 분석/완성 도구 |
| **Web UI + localhost** | 가장 풍부한 UI | 브라우저 필요, 보안 고려 | 복잡한 UI 필요 시 |

## Key Takeaways

- **CLI as Backend**: IDE extension은 프론트엔드, CLI는 백엔드. LLM 로직은 CLI에 집중되고, IDE는 UI만 담당.
- **JWT for local auth**: 로컬 프로세스 간 통신에도 인증이 필요. JWT가 간단하면서도 안전.
- **Permission proxy**: Tool permission을 IDE UI로 라우팅하면 사용자가 터미널과 IDE를 오가지 않아도 된다.
- **Feature flag로 분리**: Bridge 코드가 standalone CLI에 포함되지 않도록 build-time 제거.
- **Protocol 통일, UI 분리**: 메시지 프로토콜은 VS Code/JetBrains 공통, UI 통합만 IDE별로 구현.
- **실전 적용 시**: Protocol을 먼저 정의하고, CLI가 독립적으로 동작하게 만든 후, IDE extension을 thin client로 구현하라. 이 순서가 가장 안전하고 유지보수가 쉽다.
