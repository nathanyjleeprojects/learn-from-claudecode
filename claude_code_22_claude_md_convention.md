<!--
tags: prompt-engineering/project-config, prompt-engineering/hierarchical-loading, architecture/context-engineering
keywords: CLAUDE-md, hierarchical-config, project-configuration, local-md, context-engineering, system-prompt-injection, prompt-assembly
related_files: 03_prompt_engineering.md, 05_context_management.md, 15_configuration_system.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 22. CLAUDE.md Convention & Hierarchical Project Configuration

> 프로젝트 레벨 context engineering의 핵심: CLAUDE.md의 계층적 로딩, .local.md 규약, 시스템 프롬프트 내 위치, 실전 best practices.

---

## CLAUDE.md란 무엇인가

- 프로젝트별 "헌법" — 모든 세션에서 자동 로드되는 설정 파일
- LLM에게 프로젝트의 컨텍스트, 규칙, 관습을 전달
- Markdown 형식 → 사람도 읽을 수 있고, LLM도 이해

---

## 계층적 로딩 순서

```
우선순위 (높은 → 낮은):
1. ~/.claude/CLAUDE.md          — Global (모든 프로젝트)
2. {project-root}/CLAUDE.md     — Project root
3. {project-root}/.claude/CLAUDE.md — Project .claude dir
4. {subdirectory}/CLAUDE.md     — Subdirectory level
```

### 각 레벨의 용도

| Level | Location | 용도 | 예시 |
|-------|----------|------|------|
| Global | `~/.claude/CLAUDE.md` | 개인 선호, 모든 프로젝트 공통 | "응답은 한국어로", "commit message는 영어로" |
| Project | `./CLAUDE.md` | 프로젝트 아키텍처, 팀 규칙 | "이 프로젝트는 Next.js 16 사용", "테스트는 vitest" |
| .claude dir | `./.claude/CLAUDE.md` | 프로젝트 설정 (git tracked) | CI/CD 관련 규칙, 배포 절차 |
| Subdirectory | `src/CLAUDE.md` | 디렉토리별 규칙 | "이 디렉토리의 파일은 ESM only" |

### .local.md Convention

- `CLAUDE.local.md` — gitignored된 개인 설정
- 팀 공유 CLAUDE.md + 개인 CLAUDE.local.md 조합
- 예: API key 위치, 개인 테스트 환경, IDE 설정

```
./CLAUDE.md            ← git tracked, 팀 공유
./CLAUDE.local.md      ← gitignored, 개인 설정
./.claude/CLAUDE.md    ← git tracked
./.claude/CLAUDE.local.md ← gitignored
```

---

## System Prompt Assembly 내 위치

Report 03에서 다룬 프롬프트 조립 파이프라인에서 CLAUDE.md의 위치:

```
System Prompt 조립 순서:
1. Identity & Capabilities (하드코딩)
2. Tool Descriptions (~11K tokens)
3. Environment Info (OS, shell, etc.)
4. ▶ CLAUDE.md content (계층 병합 결과) ◀
5. Memory (auto-memory)
6. Feature-gated sections
7. MCP server instructions
```

- CLAUDE.md는 tool descriptions 뒤, memory 앞에 위치
- 이 위치가 중요한 이유: tool 사용법 다음에 "이 프로젝트에서는 이렇게 써라"가 오는 자연스러운 흐름

### Prompt Caching과의 관계

- CLAUDE.md 내용이 바뀌면 → 해당 지점 이후 prompt cache가 깨짐
- 따라서 자주 바뀌는 내용은 CLAUDE.md보다 대화 중 지시가 나음
- 안정적인 프로젝트 규칙만 CLAUDE.md에 넣어야 cache 효율 유지

---

## Size 제한과 Token Budget

```typescript
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000  // CLAUDE.md도 이 제한 적용
```

- CLAUDE.md가 너무 크면 compact 시 잘림
- 권장: 2000 tokens 이하 (약 1500-2000 words)
- 대형 프로젝트 문서는 CLAUDE.md에 넣지 말고 별도 파일 + Read tool로

---

## 실전 Best Practices

### 좋은 CLAUDE.md 예시

```markdown
# Project: MyApp

## Architecture
- Next.js 16 App Router
- Database: PostgreSQL via Prisma
- Auth: Better Auth with session strategy

## Conventions
- TypeScript strict mode
- Tests: vitest + playwright
- Commits: conventional commits (feat:, fix:, etc.)

## Commands
- `npm run dev` — development server
- `npm test` — run tests
- `npm run lint` — lint check

## Important
- Never modify migration files directly
- API routes require auth middleware
- Environment variables are in .env.local (gitignored)
```

### 안티패턴

```markdown
❌ 너무 긴 CLAUDE.md (5000+ words)
❌ 자주 바뀌는 내용 (현재 sprint 목표 등) → cache break
❌ 코드 스니펫 대량 포함 → Read tool이 더 효율적
❌ "모든 파일에 주석을 달아라" 같은 과도한 지시
❌ 민감한 정보 (API keys, passwords)
```

---

## Memory System과의 관계

CLAUDE.md vs Memory:

| | CLAUDE.md | Memory |
|--|-----------|--------|
| 범위 | 프로젝트 전체 | 세션/사용자별 |
| 변경 주기 | 드물게 (프로젝트 설정) | 자주 (학습된 선호) |
| 관리 | 수동 (사용자 편집) | 자동 (auto-memory) |
| Cache 영향 | 변경 시 cache break | 별도 섹션, 영향 적음 |
| 팀 공유 | 가능 (git tracked) | 불가 (개인) |

---

## LLM 엔지니어링 인사이트

1. **CLAUDE.md는 "project-level system prompt"**: 코드를 변경하지 않고 LLM 행동을 프로젝트별로 커스터마이즈
2. **계층적 설계의 위력**: Global → Project → Directory로 점점 구체적인 규칙. CSS의 cascade와 같은 원리
3. **Cache 친화적 설계**: 안정적 내용만 → prompt cache 효율. 불안정한 내용은 대화 중 system-reminder로
4. **.local.md 패턴**: .gitignore + .local 접미사로 팀 설정과 개인 설정의 깔끔한 분리
5. **Size = Cost**: CLAUDE.md의 모든 토큰은 매 API 호출마다 전송. 간결함이 비용 효율

---

## 적용하기: CLAUDE.md 작성 체크리스트

- [ ] 500 words 이내로 유지 (compact 안전 + cache 효율)
- [ ] 팀 공유 내용은 CLAUDE.md, 개인 설정은 CLAUDE.local.md
- [ ] 프로젝트 아키텍처 + 규칙 + 주요 명령어 포함
- [ ] 코드는 넣지 말고 파일 경로만 언급
- [ ] 민감한 정보 제외
- [ ] 자주 바뀌는 내용은 포함하지 않음

---

## 대안 비교

| 접근 방식 | 사람 가독성 | LLM 이해도 | Cache 영향 | 계층적 구성 | 적합한 상황 |
|----------|-----------|-----------|-----------|-----------|-----------|
| **Markdown Config (CLAUDE.md)** | ✅ 최고 | ✅ LLM training data에 풍부 | 변경 시 break | ✅ 디렉토리 계층 | LLM 에이전트의 프로젝트별 규칙 |
| **YAML Config** | 좋음 | 좋음 | 변경 시 break | ✅ 파일 단위 | 구조화된 설정이 필요할 때 |
| **JSON Config** | 중간 (주석 불가) | 좋음 | 변경 시 break | ❌ 단일 파일 중심 | API 스펙, machine-readable 설정 |
| **환경 변수** | 낮음 | 낮음 | 영향 없음 | ❌ flat key-value | 시크릿, 런타임 설정 |
| **코드 내 하드코딩** | 중간 | N/A | 영향 없음 | ❌ | 변경 불가 고정 규칙 |

**핵심 인사이트**: LLM에게 프로젝트 컨텍스트를 전달하는 용도라면 Markdown이 최적이다. LLM은 Markdown을 자연어처럼 이해하며, 사람도 쉽게 읽고 편집할 수 있다. YAML/JSON은 구조화된 설정에는 강하지만, "이 프로젝트에서는 migration 파일을 직접 수정하지 마세요"와 같은 자연어 규칙 전달에는 Markdown이 우월하다.

---

## 내 프로젝트에 적용하기

### Step 1: 프로젝트 Config 파일 규약 만들기

프로젝트 루트에 LLM 에이전트를 위한 config 파일을 만든다. 파일명은 `CLAUDE.md`, `AI_CONTEXT.md` 등 프로젝트에 맞게 정한다. 핵심 섹션 3가지:
1. **Architecture**: 기술 스택, 주요 디렉토리 구조, 데이터 흐름
2. **Conventions**: 코드 스타일, 테스트 전략, commit 규칙, 금지 사항
3. **Commands**: 자주 쓰는 CLI 명령어 (`npm run dev`, `npm test` 등)

이 파일은 git tracked하여 팀 전체가 공유한다. 개인 설정은 `.local.md` 접미사로 분리하고 `.gitignore`에 추가한다.

### Step 2: 계층적 로딩 구현

Config를 계층적으로 로드하여 점점 구체적인 규칙이 적용되도록 한다:
```
1. ~/.config/ai/global.md      — 모든 프로젝트 공통 (언어 선호, 응답 스타일)
2. {project-root}/CLAUDE.md    — 프로젝트 전체 규칙 (아키텍처, 규칙)
3. {subdirectory}/CLAUDE.md    — 디렉토리 특화 규칙 ("이 디렉토리는 ESM only")
```
로딩 순서는 **일반 → 구체** 방향으로, 하위 레벨이 상위를 override한다. 병합된 결과를 system prompt에 주입한다. CSS의 cascade 원리와 동일하다.

### Step 3: 500 Words 이내 유지 + Cache 효율

CLAUDE.md의 모든 토큰은 **매 API 호출마다 전송**된다. 따라서:
- **500 words(~700 tokens) 이내**로 유지한다. 이 크기면 compact 시 잘리지 않고, cache 효율도 유지된다
- 코드 스니펫은 넣지 않는다 → Read tool로 필요시 로드가 더 효율적
- **자주 바뀌는 내용은 제외**한다 (현재 sprint 목표, 임시 규칙 등) → 변경마다 prompt cache가 깨짐
- 동적 정보는 대신 `system-reminder`로 대화 중 주입한다 → cache에 영향 없음
