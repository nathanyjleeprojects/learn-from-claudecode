<!--
tags: llm-application/telemetry, context-management/session-format, prompt-engineering/system-reminder, context-management/memory-system, performance/cost-optimization
keywords: telemetry, session-jsonl, system-reminder-injection, memory-system, MEMORY-md, cost-optimization, model-tiering, GrowthBook, deferred-tool-loading, auto-memory
related_files: 02_query_engine_core.md, 05_context_management.md, 06_api_layer_providers.md, 14_state_persistence.md, 15_configuration_system.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 25. Supplementary Patterns

> 기존 리포트에서 다루지 못한 중요 패턴 5가지: Telemetry & Privacy, Session JSONL Format, System-Reminder Injection, Persistent Memory System, Cost Optimization 통합 전략.

---

## 25.1 Telemetry & Privacy

### 수집 구조

OpenTelemetry 기반, gRPC pipeline으로 telemetry 데이터를 수집한다. 다음은 실제 전송되는 이벤트의 구조이다:

```json
{
  "event_type": "ClaudeCodeInternalEvent",
  "event_data": {
    "event_name": "tengu_config_cache_stats",
    "client_timestamp": "2026-03-22T13:08:28.033Z",
    "model": "claude-opus-4-6[1m]",
    "session_id": "d583e66b-...",
    "betas": "interleaved-thinking-2025-05-14,redact-thinking-2026-02-12,...",
    "process": "{\"uptime\":711779,\"rss\":254099456,\"heapTotal\":63216640,...}",
    "additional_metadata": "{\"cache_hits\":12242,\"cache_misses\":8,\"hit_rate\":0.9993}"
  },
  "env": {
    "platform": "darwin",
    "terminal": "tmux",
    "is_ci": false,
    "is_claude_ai_auth": true,
    "version": "2.1.76",
    "arch": "arm64"
  }
}
```

### 수집 항목

| 카테고리 | 수집 데이터 | 목적 |
|---------|-----------|------|
| Process | uptime, RSS, heap, CPU usage | 성능 모니터링 |
| Cache | hits, misses, hit_rate | 캐시 효율 측정 |
| Session | model, betas, entrypoint | 사용 패턴 분석 |
| Environment | OS, terminal, version | 호환성 추적 |
| Auth | organization_uuid, account_uuid | 라이선스 관리 |

### Privacy 모델

- **device_id**: SHA-256 해시 (원본 복원 불가)
- **email**: 인증 목적으로만 수집
- **코드 내용**: 수집하지 않음
- **Failed events**: 로컬에 저장 후 재전송 (`1p_failed_events.*.json`)
- **Opt-out**: 환경 변수로 비활성화 가능

### GrowthBook A/B Testing

- 세션 시작 시 feature flags 평가
- 평가 결과는 세션 중 고정 (latch) → cache break 방지
- 내부 실험 기능: PROACTIVE, KAIROS, VOICE_MODE 등
- Flag 초기화 실패 시 모든 flag = false (graceful degradation)

### LLM 엔지니어링 인사이트

- **Cache hit rate가 가장 중요한 metric**: 99%+ hit rate = 비용 90% 절감
- **Process metrics로 메모리 누수 감지**: 장시간 세션에서 RSS 증가 추적
- **Failed events retry**: 네트워크 불안정 시 데이터 손실 방지

---

## 25.2 Session JSONL Format

### 왜 JSONL인가

- **Append-only**: 크래시 안전 (마지막 줄만 손상 가능)
- **Streaming 친화**: 줄 단위 읽기/쓰기
- **크기 무제한**: JSON 배열처럼 전체를 메모리에 올릴 필요 없음

### Record Types

**Session Metadata** (`sessions/{pid}.json`):

```json
{
  "pid": 52491,
  "sessionId": "d1098664-...",
  "cwd": "/Users/username/my-project",
  "startedAt": 1774976104102,
  "kind": "interactive",
  "entrypoint": "cli"
}
```

**User Message**:

```json
{
  "parentUuid": null,
  "isSidechain": false,
  "type": "user",
  "message": { "role": "user", "content": "..." },
  "uuid": "abc-123",
  "timestamp": "2026-03-31T...",
  "permissionMode": "bypassPermissions",
  "cwd": "/project",
  "gitBranch": "main",
  "version": "2.1.83"
}
```

**Assistant Message** (with tool calls + thinking):

```json
{
  "parentUuid": "abc-123",
  "type": "assistant",
  "message": {
    "model": "claude-opus-4-6",
    "content": [
      { "type": "thinking", "thinking": "...", "signature": "Ev8EC..." },
      { "type": "tool_use", "id": "toolu_...", "name": "Read", "input": {} }
    ],
    "stop_reason": "tool_use",
    "usage": {
      "input_tokens": 15234,
      "output_tokens": 2456,
      "cache_creation_input_tokens": 8000,
      "cache_read_input_tokens": 3000
    }
  },
  "uuid": "def-456"
}
```

**Progress Message** (sub-agent):

```json
{
  "type": "progress",
  "data": {
    "type": "agent_progress",
    "prompt": "Explore the codebase...",
    "agentId": "abc123"
  },
  "toolUseID": "toolu_...",
  "parentToolUseID": "toolu_..."
}
```

**File History Snapshot**:

```json
{
  "type": "file-history-snapshot",
  "messageId": "uuid",
  "snapshot": {
    "trackedFileBackups": {},
    "timestamp": "ISO8601"
  }
}
```

### Conversation Threading

- `uuid` + `parentUuid`로 대화 트리 구성
- `isSidechain: true` → 브랜치 대화 (main thread에서 분기)
- `/resume`이 JSONL을 파싱하여 대화 상태 복원

### LLM 엔지니어링 인사이트

- **JSONL > JSON**: 에이전트 세션은 길고 예측 불가. Append-only가 유일하게 안전한 선택
- **모든 메타데이터 저장**: version, cwd, gitBranch — 디버깅과 재현에 필수
- **Atomic write**: temp file → rename 패턴으로 크래시 안전 보장

---

## 25.3 System-Reminder Injection

### 무엇인가

대화 중간에 `<system-reminder>` 태그로 context를 주입하는 패턴이다:

```xml
<system-reminder>
Plan mode is active. You MUST NOT make any edits...
</system-reminder>
```

### 주입 유형

| 유형 | 목적 | 예시 |
|------|------|------|
| Plan mode 강제 | 행동 제한 | "edits 금지, read-only만" |
| Tool availability | 사용 가능 tool 알림 | "deferred tools 로드됨" |
| Task reminders | 진행 상황 관리 | "task tools 사용 권장" |
| Skill advertising | 사용 가능 skill 목록 | "available skills: commit, review-pr" |
| Mode enforcement | 모드별 행동 규칙 | "fast mode 활성, 간결하게" |

### System Prompt vs System-Reminder

| | System Prompt | System-Reminder |
|--|--------------|----------------|
| 위치 | 대화 시작 | 대화 중간 |
| 변경 | 세션 중 드물게 | 매 턴마다 가능 |
| Cache 영향 | 변경 시 break | 영향 없음 (conversation history 일부) |
| 용도 | 정적 규칙 | 동적 상태 전달 |

### Deferred Tool Loading 패턴

```xml
<system-reminder>
The following deferred tools are now available via ToolSearch:
AskUserQuestion, CronCreate, NotebookEdit, WebFetch, WebSearch...
</system-reminder>
```

- 모든 tool을 처음부터 로드하면 system prompt가 비대해짐
- Deferred: 이름만 알려주고, 필요 시 `ToolSearch`로 schema 로드
- 효과: system prompt ~11K tokens 안정 유지

### normalizeMessagesForAPI()

- System-reminder 태그는 API 전송 전에 정리/변환
- 일부는 user message에 삽입, 일부는 제거
- LLM은 이를 "시스템이 주입한 정보"로 인식

### LLM 엔지니어링 인사이트

- **동적 context는 reminder로**: System prompt 변경은 cache break. Reminder는 conversation history의 일부 → cache 안전
- **Deferred loading은 필수**: Tool이 많아질수록 이 패턴 없이는 context 폭발
- **Tag convention**: `<system-reminder>`처럼 명확한 태그로 LLM이 "이건 시스템 메시지"임을 인식

---

## 25.4 Persistent Memory System

### Memory 디렉토리 구조

```
~/.claude/projects/{project-id}/memory/
├── MEMORY.md              # Index (항상 context에 로드)
├── user_role.md           # 사용자 정보
├── project_analysis.md    # 프로젝트 컨텍스트
├── reference_tools.md     # 외부 참조
└── feedback_testing.md    # 피드백/선호
```

### Memory Types

| Type | 저장 내용 | 예시 |
|------|----------|------|
| `user` | 역할, 선호, 지식 수준 | "시니어 백엔드 엔지니어, Go 10년" |
| `feedback` | 교정/확인 사항 | "mock DB 쓰지 마 — 작년 사고" |
| `project` | 진행 중인 작업, 결정 | "3/5까지 merge freeze" |
| `reference` | 외부 시스템 위치 | "bugs는 Linear INGEST 프로젝트에서" |

### Memory File Format

```markdown
---
name: user_role
description: User is an LLM engineer building knowledge systems
type: user
---

Senior AI/LLM engineer focused on production systems.
Deep experience with TypeScript and Python.
```

### MEMORY.md: Index Pattern

```markdown
- [User Role](user_role.md) — senior LLM engineer, Korean
- [Project Analysis](project_analysis.md) — 18-file claude-code analysis
- [Reference: Stimpack](reference_stimpack.md) — NeonDB pipeline, model strategy
```

- 항상 context에 로드 (~200줄 제한)
- 각 항목 1줄, 150자 이내
- 실제 내용은 개별 파일에

### Auto-Memory 결정 로직

Claude Code가 자동으로 memory를 저장하는 기준:

1. 사용자가 명시적으로 "remember this" → 즉시 저장
2. 사용자가 접근법을 교정 → feedback memory
3. 비자명한 접근법을 확인 → feedback memory (긍정적)
4. 프로젝트 결정/마감 언급 → project memory
5. 외부 시스템 참조 → reference memory

### 저장하지 않는 것

- 코드 패턴, 아키텍처 → 코드에서 직접 확인 가능
- Git history → `git log`으로 확인
- 디버깅 솔루션 → 코드에 이미 반영됨
- CLAUDE.md에 이미 있는 내용 → 중복

### LLM 엔지니어링 인사이트

- **Index + Detail 분리**: MEMORY.md(항상 로드) + 개별 파일(필요시 로드) = context 효율
- **Memory는 cache 후에 위치**: System prompt → Tools → CLAUDE.md → Memory 순서 → memory 변경이 cache에 미치는 영향 최소
- **Memory decay**: 오래된 memory는 실제와 다를 수 있음 → "memory says X, verify before acting"

---

## 25.5 Cost Optimization 통합 전략

### 비용 구성 요소

```
총 비용 = Σ (input_tokens × input_price
          + cache_creation_tokens × 1.25 × input_price
          + cache_read_tokens × 0.10 × input_price
          + output_tokens × output_price
          + thinking_tokens × output_price)
```

### 최적화 레버

#### 1. Model Tiering (가장 큰 절감)

| 작업 유형 | 권장 모델 | 비용 (per 1M output) |
|----------|----------|---------------------|
| 복잡한 아키텍처 | Opus | $75 |
| 일반 코딩 | Sonnet | $15 |
| 단순 탐색 | Haiku | $4 |

Claude Code의 실천:

- Explore agent → `model: sonnet` (read-only, 빠름)
- Code architect → `model: opus` (복잡한 설계)
- Code reviewer → `model: sonnet` (패턴 매칭)
- Sub-agent는 `ANTHROPIC_SMALL_FAST_MODEL` 사용

#### 2. Prompt Caching (Report 23 참조)

- 99%+ hit rate로 input 비용 90% 절감
- 가장 ROI가 높은 최적화

#### 3. Context Compression (Report 05 참조)

- Compact mode: 오래된 대화 요약 → token 절감
- Microcompact: 더 공격적인 압축
- 효과: 긴 세션에서 context overflow 방지 + 비용 절감

#### 4. Deferred Tool Loading

- 42개 tool 전체 schema: ~11K tokens
- 사용하지 않는 MCP tool은 deferred → 필요시만 로드
- 효과: system prompt 크기 안정화

#### 5. Token Budget Allocation

```
Context Budget 분배 전략:
├── System Prompt: ~11K (고정)
├── CLAUDE.md: ~2K (고정)
├── Memory: ~1K (고정)
├── Conversation: 동적 (compact으로 관리)
├── Thinking: 동적 (context에 따라 조정)
└── Output: 동적 (thinking과 경쟁)
```

#### 6. Enterprise Cost Controls

- 일일 토큰 한도 설정 가능
- Policy limits 초과 시 경고/차단
- `dailyModelTokenUsage`로 사용량 추적

### 비용 추적 구조

```typescript
// Stats cache
{
  "dailyModelTokenUsage": {
    "2026-03-31": {
      "claude-opus-4-6": { input: 500000, output: 100000 },
      "claude-sonnet-4-6": { input: 200000, output: 50000 }
    }
  },
  "modelUsageAggregated": {
    "claude-opus-4-6": { totalCost: 12.50, sessions: 45 }
  }
}
```

### LLM 엔지니어링 인사이트

1. **Model tiering이 첫 번째**: 같은 작업을 Opus 대신 Sonnet으로 하면 5배 절감
2. **Prompt caching이 두 번째**: 90% input 비용 절감
3. **Compact가 세 번째**: 긴 세션의 비용 폭발 방지
4. **모니터링 없이 최적화 없다**: 모델별, 세션별, 일별 비용 추적 필수
5. **비용 = 품질의 함수**: 과도한 절감은 품질 저하. Tiering으로 적재적소에 투자

---

## 내 프로젝트에 적용하기

### Step 1: JSONL 세션 포맷 도입

에이전트 세션을 JSON 배열이 아닌 **JSONL(줄 단위 JSON)**으로 저장한다:
- **Append-only**: 크래시 시 마지막 줄만 손상. JSON 배열은 닫는 `]`가 없으면 전체 파손
- **Streaming 친화**: 줄 단위 읽기/쓰기로 실시간 모니터링 가능
- **메모리 효율**: 전체를 메모리에 올릴 필요 없이 줄 단위 처리

각 record에 `uuid` + `parentUuid`를 포함하여 대화 트리를 구성하고, `type` 필드로 user/assistant/progress/file-history를 구분한다. 메타데이터(version, cwd, gitBranch)를 모든 메시지에 포함하면 디버깅과 세션 재현이 크게 쉬워진다.

### Step 2: System-Reminder Injection으로 동적 Context 관리

System prompt 변경은 **prompt cache break**를 유발한다. 동적으로 변하는 context는 대신 `<system-reminder>` 태그로 conversation history에 주입한다:
```xml
<system-reminder>
Plan mode is active. You MUST NOT make any edits.
Available deferred tools: WebFetch, WebSearch, NotebookEdit
</system-reminder>
```
- **System prompt**: 정적 규칙만 (identity, tools, 프로젝트 config) → cache 유지
- **System-reminder**: 동적 상태 (모드 변경, deferred tool 로드, skill 광고) → cache 무영향
- Deferred tool loading 패턴: 42개 tool 전체 schema(~11K tokens) 대신, 이름만 알려주고 필요 시 `ToolSearch`로 schema 로드

이 분리로 system prompt를 안정적으로 유지하면서도 매 턴마다 context를 업데이트할 수 있다.

### Step 3: 비용 추적 공식 구현

모든 API 응답의 `usage` 필드에서 비용을 계산하고 누적한다:
```
턴 비용 = input_tokens × input_price
        + cache_creation_tokens × 1.25 × input_price
        + cache_read_tokens × 0.10 × input_price
        + output_tokens × output_price
        + thinking_tokens × output_price
```
- **일별/모델별 집계**: `{ "2026-04-01": { "opus": { input: 500K, output: 100K, cost: $12.50 } } }`
- **Model tiering 효과 측정**: 같은 작업을 Opus($75/1M output) 대신 Sonnet($15/1M)으로 처리하면 5배 절감
- **Alert 설정**: 일일 비용 한도를 초과하면 경고. Enterprise 환경에서는 `dailyModelTokenUsage`로 정책 기반 차단
- **비용 최적화 우선순위**: (1) Model tiering → (2) Prompt caching → (3) Context compression → (4) Deferred tool loading
