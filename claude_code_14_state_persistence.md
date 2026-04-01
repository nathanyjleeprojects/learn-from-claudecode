<!--
tags: architecture/state-management, context-management/session-persistence, context-management/memory-system, llm-application/persistence
keywords: state-management, AppStateStore, session-persistence, JSONL, stats-cache, memory-system, CLAUDE-md, session-storage, atomic-writes, change-observers, file-history
related_files: 02_query_engine_core.md, 05_context_management.md, 15_configuration_system.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 14. State & Persistence

> Claude Code의 상태 관리와 영속성: AppStateStore, session JSONL, stats cache, memory 계층.

## Keywords
state-management, AppStateStore, session-persistence, JSONL-transcript, stats-cache, memory-system, CLAUDE-md, atomic-writes, change-observers, memoization, session-storage, file-history, session-recovery

## AppStateStore: Global Mutable State

`src/state/AppStateStore.ts`가 전역 상태를 관리한다.

### State 구조

```typescript
type State = {
  // 세션 정보
  originalCwd: string          // 최초 작업 디렉토리
  projectRoot: string          // 프로젝트 루트
  sessionId: SessionId         // 고유 세션 ID
  isInteractive: boolean       // CLI vs programmatic

  // 비용/성능 추적
  totalCostUSD: number         // 누적 비용
  totalAPIDuration: number     // 총 API 호출 시간
  totalToolDuration: number    // 총 tool 실행 시간
  modelUsage: Record<string, ModelUsage>  // 모델별 토큰 사용량

  // 인증
  authToken?: string           // OAuth 토큰
  apiKey?: string              // API 키

  // Feature flags
  featureFlags: Record<string, boolean>

  // ... 50+ 추가 필드
}
```

### Change Observers

`src/state/onChangeAppState.ts`가 상태 변경 시 side-effect를 실행한다:

```typescript
// 상태 변경 → observer 호출 → side-effect 실행
appState.totalCostUSD가 변경됨
  → observer: 비용 표시 업데이트
  → observer: 예산 초과 경고 체크
```

이 패턴은 React의 `useEffect`와 유사하지만, React 외부에서도 동작한다.

### Selectors

`src/state/`에 derived state 함수들:
- `getModelUsageSummary(state)`: 모델별 사용량 요약
- `getSessionDuration(state)`: 세션 경과 시간
- `isOverBudget(state)`: 예산 초과 여부

## Session Persistence: JSONL

`src/utils/sessionStorage.ts`가 세션을 JSONL(JSON Lines)로 저장한다.

### 저장 위치

```
~/.claude/sessions/{projectRoot}/{sessionId}/
├── transcript.jsonl    # 대화 기록 (메시지당 한 줄)
├── file-history.json   # 수정된 파일 히스토리
└── attribution.json    # 누가 어떤 tool을 실행했는지
```

### JSONL 형식의 장점

| | JSON | JSONL |
|---|------|-------|
| 쓰기 | 전체 파일 덮어쓰기 | 한 줄 append |
| 읽기 | 전체 파싱 필요 | 줄 단위 스트리밍 |
| 크래시 안정성 | 불완전한 JSON → 전체 손실 | 마지막 줄만 손실 |
| 디버깅 | 구조 파악 필요 | 줄 단위 확인 가능 |

**LLM 엔지니어링 인사이트**: 대화 기록은 JSONL이 JSON보다 월등히 적합하다. Append-only 특성이 크래시 시 데이터 손실을 최소화하고, 긴 대화에서도 전체 파일을 메모리에 올릴 필요가 없다.

### Session Recovery

`/resume` 명령으로 이전 세션을 복원한다:

```
1. ~/.claude/sessions/ 탐색 → 최근 세션 목록
2. 사용자가 세션 선택
3. transcript.jsonl 로드 → conversation history 복원
4. 마지막 상태에서 이어서 대화
```

## Stats Cache

`src/utils/statsCache.ts`가 통계 데이터를 디스크에 캐시한다.

### 캐시 구조

```typescript
// ~/.claude/stats-cache.json
{
  version: 3,  // 마이그레이션 지원
  dailyActivity: [...],          // 일별 활동 (일수로 제한)
  dailyModelTokenUsage: [...],   // 일별 모델별 토큰 (일수로 제한)
  modelUsageAggregated: {...},   // 모델별 누적 (모델 수로 제한)
  sessionAggregates: {...},      // 세션 누적 통계
  hourCounts: [...],             // 시간대별 사용량 (24개)
  speculationTimeSaved: 0,       // 추론 시간 절약
}
```

### Atomic Writes

```typescript
// 데이터 손실 방지를 위한 atomic write
writeStatsCache(data) {
  const tempFile = path + '.tmp'
  writeFileSync(tempFile, JSON.stringify(data))
  renameSync(tempFile, path)  // atomic rename
}
```

Temp file에 먼저 쓰고 rename하는 패턴. 쓰는 도중 크래시가 나도 원본 파일은 안전하다.

### Lock-Protected Access

```typescript
withStatsCacheLock(async () => {
  const cache = readStatsCache()
  cache.totalSessions++
  writeStatsCache(cache)
})
```

동시 접근(여러 Claude Code 인스턴스)에서 데이터 corruption을 방지한다.

## Memory System: CLAUDE.md

`src/memdir/`가 영구 메모리를 관리한다.

### Memory 계층

| Level | Location | 범위 | 용도 |
|-------|----------|------|------|
| **Global** | `~/.claude/CLAUDE.md` | 모든 프로젝트 | 사용자 선호, 글로벌 설정 |
| **Project** | `CLAUDE.md` (프로젝트 루트) | 해당 프로젝트 | 컨벤션, 아키텍처 결정 |
| **User memories** | `~/.claude/projects/{id}/memory/*.md` | 프로젝트별 사용자 | 학습된 선호, 피드백 |
| **Team** | `src/services/teamMemorySync/` | 팀 전체 | 팀 컨벤션, 공유 지식 |
| **Session** | In-memory | 현재 대화 | 대화 중 추출된 기억 |

### Memory Frontmatter Format

```markdown
---
name: user_role
description: 사용자의 역할과 전문성
type: user
---

사용자는 시니어 프론트엔드 엔지니어. React + TypeScript 10년 경력.
백엔드는 Go를 주로 사용하지만 이 프로젝트에서는 처음.
```

### Memory Auto-Extraction

`src/services/extractMemories/`가 대화에서 자동으로 유용한 정보를 추출하여 memory로 저장한다:

- 사용자 수정/피드백 → feedback memory
- 프로젝트 결정사항 → project memory
- 외부 리소스 참조 → reference memory

## 내 프로젝트에 적용하기

### 세션 영속성 구현하기

**Step 1: JSONL append 패턴으로 대화 저장**

대화 기록은 JSON이 아니라 JSONL로 저장한다. 핵심 이유: 크래시 시 마지막 한 줄만 손실:

```typescript
import { appendFileSync } from 'fs'

function saveMessage(sessionPath: string, message: Message) {
  const line = JSON.stringify({
    timestamp: Date.now(),
    role: message.role,
    content: message.content,
    ...(message.toolUse && { toolUse: message.toolUse })
  })
  appendFileSync(sessionPath, line + '\n')
}

function loadSession(sessionPath: string): Message[] {
  return readFileSync(sessionPath, 'utf-8')
    .split('\n')
    .filter(Boolean)
    .map(line => JSON.parse(line))
}
```

**Step 2: Atomic writes for 설정/캐시**

설정 파일처럼 전체 덮어쓰기가 필요한 경우, temp + rename 패턴:

```typescript
function atomicWrite(filePath: string, data: string) {
  const tmpPath = filePath + '.tmp'
  writeFileSync(tmpPath, data)
  renameSync(tmpPath, filePath)  // OS 수준의 atomic operation
}
```

**Step 3: Memory 계층 설계**

LLM에게 전달하는 context를 계층화한다:

```
Global memory  → 모든 프로젝트 공통 (사용자 선호, 코딩 스타일)
Project memory → 프로젝트별 (아키텍처 결정, 컨벤션)
Session memory → 현재 대화 (대화 중 발견한 사실)
```

계층이 올라갈수록 변경 빈도가 낮고, 적용 범위가 넓다. 시스템 프롬프트에 계층별로 구분해서 주입한다.

**Step 4: Version 필드로 마이그레이션 대비**

영속 데이터에는 반드시 version 필드를 넣는다:

```typescript
interface PersistedData {
  version: number  // 형식이 바뀔 때마다 increment
  // ... fields
}
```

## Key Takeaways

- **JSONL for conversation**: Append-only, crash-safe, streaming-friendly. 대화 기록 저장에 최적.
- **Atomic writes**: Temp file + rename 패턴으로 데이터 손실 방지. 통계 캐시, 설정 파일 모두에 적용.
- **Memory 계층**: Global → Project → User → Session. 범위별로 memory를 분리하여 관련 context만 LLM에게 전달.
- **Stats cache versioning**: Version 필드로 마이그레이션 지원. 캐시 형식이 바뀌어도 기존 데이터 유지.
- **Change observers**: React 외부에서도 상태 변경에 반응하는 observer 패턴.
- **실전 적용 시**: 대화 기록은 JSONL, 설정은 atomic write, 모든 영속 데이터에 version 필드. 이 세 가지만 지키면 데이터 손실 사고를 대부분 예방할 수 있다.
