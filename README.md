# Learn from Claude Code

> Deep analysis of Anthropic's **Claude Code** CLI source (~1,900 TS files, 512K+ lines), extracting production-grade LLM engineering patterns across 26 in-depth articles.

**[ENG](#eng) | [KR](#kr)**

---

<a id="eng"></a>

## ENG

### Claude Code Plugin: what-would-cc-do

This repo doubles as a **Claude Code plugin**. Assess your code and LLM practices against Claude Code's production engineering patterns, then apply improvements directly.

#### Installation

```
/plugin marketplace add nathanyjleeprojects/learn-from-claudecode
/plugin install what-would-cc-do@learn-from-claudecode
```

#### Commands

| Command | Description |
|---------|-------------|
| `/what-would-cc-do:assess` | Analyze your code/practices against Claude Code patterns |
| `/what-would-cc-do:claudecodefy` | Apply improvements based on assess results |

#### Workflow

1. Run `:assess` — select scope and what to analyze
2. Review gap analysis — each gap shows CC's approach, your approach, pros/cons
3. Choose which gaps to pursue (all / high-impact only / specific items / none)
4. Run `:claudecodefy` — apply selected improvements to your code

No external DB or API keys required. The 27 markdown files in this repo are the knowledge base.

### What's in the Knowledge Base

A curated curriculum for becoming a better LLM engineer, organized in 4 phases:

#### Phase 1: Core Architecture (Essential)

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 1 | [00_index](claude_code_00_index.md) | Master Index | Full structure, navigation |
| 2 | [01_architecture_overview](claude_code_01_architecture_overview.md) | System Architecture | Module structure, request lifecycle, pipeline model |
| 3 | [02_query_engine_core](claude_code_02_query_engine_core.md) | LLM Engine Core | Agent loop, tool execution cycle |
| 4 | [03_prompt_engineering](claude_code_03_prompt_engineering.md) | Prompt Construction | System prompt assembly, hierarchical prompt design |
| 5 | [04_tool_system](claude_code_04_tool_system.md) | Tool System | Design patterns for 42 agent tools |

#### Phase 2: Production Infrastructure

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 6 | [05_context_management](claude_code_05_context_management.md) | Context Management | Token counting, context compression, window management |
| 7 | [07_retry_resilience](claude_code_07_retry_resilience.md) | Retry & Resilience | LLM API stability strategies |
| 8 | [06_api_layer_providers](claude_code_06_api_layer_providers.md) | Multi-Provider API | LLM provider abstraction |
| 9 | [08_streaming_concurrency](claude_code_08_streaming_concurrency.md) | Streaming & Concurrency | Real-time response, parallel execution |
| 10 | [09_security_model](claude_code_09_security_model.md) | Security Model | Prompt injection defense, LLM-specific security |

#### Phase 3: Advanced Patterns

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 11 | [10_multi_agent](claude_code_10_multi_agent.md) | Multi-Agent | Agent orchestration patterns |
| 12 | [11_mcp_integration](claude_code_11_mcp_integration.md) | MCP Protocol | MCP integration and tool discovery |
| 13 | [19_hooks_system](claude_code_19_hooks_system.md) | Hooks System | Lifecycle event-based workflow control |
| 14 | [20_extended_thinking](claude_code_20_extended_thinking.md) | Extended Thinking | Reasoning budget, thinking token management |
| 15 | [23_prompt_caching](claude_code_23_prompt_caching.md) | Prompt Caching | Cost optimization, cache strategies |

#### Phase 4: System Completion

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 16 | [12_command_system](claude_code_12_command_system.md) | Slash Commands | Extensible design for 87 commands |
| 17 | [13_ui_terminal](claude_code_13_ui_terminal.md) | Terminal UI | React + Ink terminal interface |
| 18 | [14_state_persistence](claude_code_14_state_persistence.md) | State Persistence | Session persistence, state management |
| 19 | [15_configuration_system](claude_code_15_configuration_system.md) | Configuration | Config layers and feature flags |
| 20 | [16_performance_optimization](claude_code_16_performance_optimization.md) | Performance | Lazy loading, bundle optimization |
| 21 | [17_ide_bridge](claude_code_17_ide_bridge.md) | IDE Integration | VS Code extension bridge |
| 22 | [21_plugins_skills](claude_code_21_plugins_skills.md) | Plugins & Skills | Plugin system architecture |
| 23 | [22_claude_md_convention](claude_code_22_claude_md_convention.md) | CLAUDE.md Convention | Project config patterns, memory hierarchy |
| 24 | [24_testing_evaluation](claude_code_24_testing_evaluation.md) | Testing & Evaluation | LLM app testing frameworks |
| 25 | [25_supplementary_patterns](claude_code_25_supplementary_patterns.md) | Supplementary Patterns | Telemetry, session formatting |

#### Wrap-up

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 26 | [18_patterns_and_lessons](claude_code_18_patterns_and_lessons.md) | Patterns & Lessons | Comprehensive synthesis, actionable checklist |

### Quick Lookup

| Question | File |
|----------|------|
| How to handle LLM API failures? | [07_retry_resilience](claude_code_07_retry_resilience.md) |
| How to structure system prompts? | [03_prompt_engineering](claude_code_03_prompt_engineering.md) |
| What happens when context window fills up? | [05_context_management](claude_code_05_context_management.md) |
| How to design agent tools? | [04_tool_system](claude_code_04_tool_system.md) |
| How to run multiple tools in parallel? | [08_streaming_concurrency](claude_code_08_streaming_concurrency.md) |
| How to prevent prompt injection? | [09_security_model](claude_code_09_security_model.md) |
| How to support multiple LLM providers? | [06_api_layer_providers](claude_code_06_api_layer_providers.md) |
| How to manage sub-agents? | [10_multi_agent](claude_code_10_multi_agent.md) |
| How is MCP protocol implemented? | [11_mcp_integration](claude_code_11_mcp_integration.md) |
| How to control workflows with hooks? | [19_hooks_system](claude_code_19_hooks_system.md) |
| How to manage extended thinking budget? | [20_extended_thinking](claude_code_20_extended_thinking.md) |
| How to optimize prompt cache costs? | [23_prompt_caching](claude_code_23_prompt_caching.md) |
| How to test LLM apps? | [24_testing_evaluation](claude_code_24_testing_evaluation.md) |
| Common patterns in production LLM apps? | [18_patterns_and_lessons](claude_code_18_patterns_and_lessons.md) |

### Tech Stack Analyzed

| Category | Technology |
|----------|------------|
| Runtime | Bun v1.1.0+ |
| Language | TypeScript (strict mode) |
| Terminal UI | React 19 + Ink |
| CLI Parser | Commander.js |
| Validation | Zod v3.24 |
| LLM SDK | Anthropic SDK v0.39.0 |
| Protocols | MCP SDK v1.12.1, LSP |
| Auth | OAuth 2.0, JWT |

### Codebase Scale

- **1,916** TypeScript files
- **512,000+** lines of code
- **42** agent tools
- **87** slash commands

---

<a id="kr"></a>

## KR

### Claude Code Plugin: what-would-cc-do

이 리포는 **Claude Code 플러그인**으로도 사용할 수 있습니다. 자신의 코드와 LLM practice를 Claude Code의 패턴과 비교 분석하고, 개선점을 적용할 수 있습니다.

#### 설치

```
/plugin marketplace add nathanyjleeprojects/learn-from-claudecode
/plugin install what-would-cc-do@learn-from-claudecode
```

#### 사용법

| 명령어 | 설명 |
|--------|------|
| `/what-would-cc-do:assess` | 내 코드/practice를 Claude Code 패턴과 비교 분석 |
| `/what-would-cc-do:claudecodefy` | 분석 결과를 바탕으로 개선 적용 |

#### 워크플로우

1. `:assess` 실행 — 분석 대상과 범위 선택
2. Gap 분석 결과 확인 — 각 gap별 CC 방식, 현재 방식, pros/cons 제시
3. 개선할 gap 선택 (전체 / 고영향만 / 특정 항목 / 없음)
4. `:claudecodefy` 실행 — 선택한 개선점을 코드에 적용

외부 DB나 API 키가 필요 없습니다. 이 리포의 27개 MD 파일이 곧 knowledge base입니다.

### 이 리포의 목적

**이 리포를 통해 더 나은 LLM 엔지니어가 될 수 있도록 커리큘럼화 하는 데 집중했습니다.**

단순한 코드 문서화가 아닙니다. 실제 프로덕션에서 작동하는 LLM 애플리케이션이 어떤 설계 결정을 내렸는지, 왜 그렇게 만들었는지, 그리고 그 패턴을 자신의 프로젝트에 어떻게 적용할 수 있는지를 체계적으로 학습할 수 있도록 구성했습니다.

#### 대상 독자

- LLM/AI 엔지니어로 성장하고 싶은 개발자
- 에이전틱 시스템을 직접 구축하려는 실무자
- 프로덕션급 AI 애플리케이션의 내부 구조가 궁금한 분

### 커리큘럼 학습 순서

LLM 엔지니어링 관점에서 가치가 높은 순서로 정리했습니다. 순서대로 읽으면 전체 시스템을 체계적으로 이해할 수 있습니다.

#### Phase 1: 핵심 아키텍처 (필수)

| 순서 | 파일 | 주제 | 배울 수 있는 것 |
|------|------|------|-----------------|
| 1 | [00_index](claude_code_00_index.md) | Master Index | 전체 구조 파악, 네비게이션 |
| 2 | [01_architecture_overview](claude_code_01_architecture_overview.md) | 시스템 아키텍처 | 모듈 구조, 요청 라이프사이클, 파이프라인 모델 |
| 3 | [02_query_engine_core](claude_code_02_query_engine_core.md) | LLM 엔진 코어 | 에이전트 루프, 도구 실행 사이클의 심장부 |
| 4 | [03_prompt_engineering](claude_code_03_prompt_engineering.md) | 프롬프트 구축 | 시스템 프롬프트 조립, 계층적 프롬프트 설계 |
| 5 | [04_tool_system](claude_code_04_tool_system.md) | 도구 시스템 | 42개 에이전트 도구의 설계 패턴과 스키마 |

#### Phase 2: 프로덕션 인프라

| 순서 | 파일 | 주제 | 배울 수 있는 것 |
|------|------|------|-----------------|
| 6 | [05_context_management](claude_code_05_context_management.md) | 컨텍스트 관리 | 토큰 카운팅, 컨텍스트 압축, 윈도우 관리 |
| 7 | [07_retry_resilience](claude_code_07_retry_resilience.md) | 재시도와 복원력 | LLM API 안정성 확보 전략 |
| 8 | [06_api_layer_providers](claude_code_06_api_layer_providers.md) | 멀티 프로바이더 API | 여러 LLM 프로바이더 추상화 |
| 9 | [08_streaming_concurrency](claude_code_08_streaming_concurrency.md) | 스트리밍과 동시성 | 실시간 응답 처리, 병렬 실행 |
| 10 | [09_security_model](claude_code_09_security_model.md) | 보안 모델 | 프롬프트 인젝션 방어, LLM 특화 보안 |

#### Phase 3: 고급 패턴

| 순서 | 파일 | 주제 | 배울 수 있는 것 |
|------|------|------|-----------------|
| 11 | [10_multi_agent](claude_code_10_multi_agent.md) | 멀티 에이전트 | 다중 에이전트 오케스트레이션 |
| 12 | [11_mcp_integration](claude_code_11_mcp_integration.md) | MCP 프로토콜 | MCP 통합과 도구 발견 |
| 13 | [19_hooks_system](claude_code_19_hooks_system.md) | 훅 시스템 | 라이프사이클 이벤트 기반 워크플로우 제어 |
| 14 | [20_extended_thinking](claude_code_20_extended_thinking.md) | 확장 사고 | 추론 예산 관리, thinking 토큰 |
| 15 | [23_prompt_caching](claude_code_23_prompt_caching.md) | 프롬프트 캐싱 | 비용 최적화, 캐시 전략 |

#### Phase 4: 시스템 완성

| 순서 | 파일 | 주제 | 배울 수 있는 것 |
|------|------|------|-----------------|
| 16 | [12_command_system](claude_code_12_command_system.md) | 슬래시 명령어 | 87개 명령어의 확장 가능한 설계 |
| 17 | [13_ui_terminal](claude_code_13_ui_terminal.md) | 터미널 UI | React + Ink 기반 터미널 인터페이스 |
| 18 | [14_state_persistence](claude_code_14_state_persistence.md) | 상태 저장 | 세션 영속성, 상태 관리 |
| 19 | [15_configuration_system](claude_code_15_configuration_system.md) | 설정 시스템 | 설정 계층과 피처 플래그 |
| 20 | [16_performance_optimization](claude_code_16_performance_optimization.md) | 성능 최적화 | 지연 로딩, 번들 최적화 |
| 21 | [17_ide_bridge](claude_code_17_ide_bridge.md) | IDE 통합 | VS Code 확장 브릿지 구조 |
| 22 | [21_plugins_skills](claude_code_21_plugins_skills.md) | 플러그인과 스킬 | 플러그인 시스템 설계 |
| 23 | [22_claude_md_convention](claude_code_22_claude_md_convention.md) | CLAUDE.md 컨벤션 | 프로젝트 설정 패턴, 메모리 계층 |
| 24 | [24_testing_evaluation](claude_code_24_testing_evaluation.md) | 테스트와 평가 | LLM 앱 테스트 프레임워크 |
| 25 | [25_supplementary_patterns](claude_code_25_supplementary_patterns.md) | 보충 패턴 | 텔레메트리, 세션 포맷 |

#### 마무리

| 순서 | 파일 | 주제 | 배울 수 있는 것 |
|------|------|------|-----------------|
| 26 | [18_patterns_and_lessons](claude_code_18_patterns_and_lessons.md) | 종합 패턴과 교훈 | 전체 분석의 종합, actionable checklist |

### 질문으로 찾기

| 궁금한 것 | 파일 |
|-----------|------|
| LLM API 호출이 실패하면 어떻게 처리하지? | [07_retry_resilience](claude_code_07_retry_resilience.md) |
| 시스템 프롬프트를 어떻게 구성하지? | [03_prompt_engineering](claude_code_03_prompt_engineering.md) |
| 컨텍스트 윈도우가 꽉 차면? | [05_context_management](claude_code_05_context_management.md) |
| 에이전트 도구를 어떻게 설계하지? | [04_tool_system](claude_code_04_tool_system.md) |
| 여러 LLM 도구를 동시에 실행하려면? | [08_streaming_concurrency](claude_code_08_streaming_concurrency.md) |
| 프롬프트 인젝션을 어떻게 막지? | [09_security_model](claude_code_09_security_model.md) |
| 여러 LLM 프로바이더를 지원하려면? | [06_api_layer_providers](claude_code_06_api_layer_providers.md) |
| sub-agent를 어떻게 관리하지? | [10_multi_agent](claude_code_10_multi_agent.md) |
| MCP 프로토콜은 어떻게 구현되어 있지? | [11_mcp_integration](claude_code_11_mcp_integration.md) |
| Hook으로 LLM 워크플로우를 제어하려면? | [19_hooks_system](claude_code_19_hooks_system.md) |
| Extended thinking budget을 어떻게 관리하지? | [20_extended_thinking](claude_code_20_extended_thinking.md) |
| 프롬프트 캐시 비용을 최적화하려면? | [23_prompt_caching](claude_code_23_prompt_caching.md) |
| LLM 앱을 어떻게 테스트하지? | [24_testing_evaluation](claude_code_24_testing_evaluation.md) |
| 프로덕션 LLM 앱의 공통 패턴은? | [18_patterns_and_lessons](claude_code_18_patterns_and_lessons.md) |

### 분석 대상 Tech Stack

| 카테고리 | 기술 |
|----------|------|
| Runtime | Bun v1.1.0+ |
| Language | TypeScript (strict mode) |
| Terminal UI | React 19 + Ink |
| CLI Parser | Commander.js |
| Validation | Zod v3.24 |
| LLM SDK | Anthropic SDK v0.39.0 |
| Protocols | MCP SDK v1.12.1, LSP |
| Auth | OAuth 2.0, JWT |

### 코드베이스 규모

- **1,916** TypeScript 파일
- **512,000+** 줄의 코드
- **42** 에이전트 도구
- **87** 슬래시 명령어

---

## 출처 / Source

이 분석은 [Claude Code npm 패키지의 공개된 sourcemap](https://github.com/nirholas/claude-code)을 기반으로 합니다.

This analysis is based on the [publicly available sourcemap from the Claude Code npm package](https://github.com/nirholas/claude-code).
