<!--
tags: architecture/system-design, architecture/module-organization, llm-application/production-architecture
keywords: claude-code, anthropic, cli, llm-application, agentic-coding, typescript, react-ink, production-architecture, index
related_files: 01_architecture_overview.md, 18_patterns_and_lessons.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# Claude Code Deep Analysis - Master Index

> Anthropic의 공식 CLI 툴 소스코드(~1900 TS 파일, 512K+ lines)를 분석하여 프로덕션급 LLM 애플리케이션의 핵심 엔지니어링 패턴을 추출한 심층 분석 시리즈.

## Keywords
claude-code, anthropic, agentic-cli, llm-engineering, tool-use, prompt-engineering, streaming, context-management, multi-agent, mcp-protocol, typescript, react-ink, production-llm-app

## 이 분석이 무엇인가

Claude Code는 Anthropic이 만든 터미널 기반 AI 코딩 어시스턴트다. 2026년 3월 31일 npm 패키지의 sourcemap 파일을 통해 소스코드가 유출되었고, 이를 통해 프로덕션급 LLM 애플리케이션이 실제로 어떻게 구축되는지를 직접 들여다볼 수 있게 되었다.

이 분석 시리즈는 단순한 코드 문서화가 아니다. **LLM/AI 엔지니어로서 실제 프로덕션에서 작동하는 패턴을 배우고, 자신의 프로젝트에 적용할 수 있는 인사이트를 추출하는 것**이 목표다.

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User Input (Terminal / IDE)                  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CLI Parser (Commander.js)  →  REPL Screen (React + Ink)           │
│  src/main.tsx                  src/screens/REPL.tsx                 │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
          ┌─────────────────┐      ┌──────────────────┐
          │  Slash Commands  │      │  Query Engine     │
          │  src/commands/   │      │  src/QueryEngine.ts│
          │  (87 commands)   │      │  (~46K lines)     │
          └─────────────────┘      └────────┬─────────┘
                                            │
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                    ┌──────────────┐ ┌───────────┐ ┌──────────────┐
                    │ System Prompt│ │  API Layer │ │  Tool System │
                    │ Assembly     │ │  (Multi-   │ │  (42 tools)  │
                    │ src/constants│ │  Provider) │ │  src/tools/  │
                    │ /prompts.ts  │ │  +Retry    │ │  + Tool.ts   │
                    └──────────────┘ └───────────┘ └──────┬───────┘
                                                          │
                              ┌────────────┬──────────────┼──────────┐
                              ▼            ▼              ▼          ▼
                        ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐
                        │File I/O  │ │Shell Exec│ │Sub-Agent │ │MCP/LSP │
                        │Read/Write│ │Bash/PS   │ │Spawning  │ │Protocol│
                        │Edit/Glob │ │Security  │ │Teams     │ │Client  │
                        └──────────┘ └──────────┘ └──────────┘ └────────┘
                              │            │              │          │
                              ▼            ▼              ▼          ▼
                    ┌─────────────────────────────────────────────────────┐
                    │              Services Layer                         │
                    │  OAuth | MCP | LSP | Analytics | Compact | Policy  │
                    └─────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────────────────────────────────────────┐
                    │              State & Persistence                    │
                    │  AppState | Session JSONL | Stats Cache | Memory   │
                    └─────────────────────────────────────────────────────┘
```

## Reading Guide

### Linear Reading (전체 시스템 학습)

LLM 엔지니어링의 가치가 높은 순서:

1. **01** Architecture Overview → 전체 구조 파악
2. **02** Query Engine → LLM 앱의 심장부
3. **03** Prompt Engineering → 시스템 프롬프트 구축 기법
4. **04** Tool System → 에이전트 도구 설계
5. **05** Context Management → 컨텍스트 윈도우 관리
6. **07** Retry & Resilience → LLM API 안정성 확보
7. **06** API Layer → 멀티 프로바이더 추상화
8. **08** Streaming & Concurrency → 실시간 응답 처리
9. **09** Security → LLM 특화 보안
10. **10** Multi-Agent → 다중 에이전트 오케스트레이션
11. **18** Patterns & Lessons → 종합 교훈 (마지막에 읽기)

### Question-Based Navigation (질문으로 찾기)

| 질문 | 파일 |
|------|------|
| "LLM API 호출이 실패하면 어떻게 처리하지?" | [07_retry_resilience.md](07_retry_resilience.md) |
| "시스템 프롬프트를 어떻게 구성하지?" | [03_prompt_engineering.md](03_prompt_engineering.md) |
| "컨텍스트 윈도우가 꽉 차면?" | [05_context_management.md](05_context_management.md) |
| "에이전트 도구를 어떻게 설계하지?" | [04_tool_system.md](04_tool_system.md) |
| "여러 LLM 도구를 동시에 실행하려면?" | [08_streaming_concurrency.md](08_streaming_concurrency.md) |
| "프롬프트 인젝션을 어떻게 막지?" | [09_security_model.md](09_security_model.md) |
| "여러 LLM 프로바이더를 지원하려면?" | [06_api_layer_providers.md](06_api_layer_providers.md) |
| "sub-agent를 어떻게 관리하지?" | [10_multi_agent.md](10_multi_agent.md) |
| "MCP 프로토콜은 어떻게 구현되어 있지?" | [11_mcp_integration.md](11_mcp_integration.md) |
| "터미널 UI를 React로 만들 수 있어?" | [13_ui_terminal.md](13_ui_terminal.md) |
| "세션 상태를 어떻게 유지하지?" | [14_state_persistence.md](14_state_persistence.md) |
| "프로덕션 LLM 앱의 공통 패턴은?" | [18_patterns_and_lessons.md](18_patterns_and_lessons.md) |
| "Hook으로 LLM 워크플로우를 제어하려면?" | [19_hooks_system.md](19_hooks_system.md) |
| "Extended thinking budget을 어떻게 관리하지?" | [20_extended_thinking.md](20_extended_thinking.md) |
| "플러그인/스킬 시스템을 만들려면?" | [21_plugins_skills.md](21_plugins_skills.md) |
| "CLAUDE.md 같은 프로젝트 설정 패턴?" | [22_claude_md_convention.md](22_claude_md_convention.md) |
| "프롬프트 캐시 비용을 최적화하려면?" | [23_prompt_caching.md](23_prompt_caching.md) |
| "LLM 앱을 어떻게 테스트하지?" | [24_testing_evaluation.md](24_testing_evaluation.md) |

## File Manifest

| # | File | Topic | Tags |
|---|------|-------|------|
| 00 | [00_index.md](00_index.md) | Master Index | `architecture/system-design` |
| 01 | [01_architecture_overview.md](01_architecture_overview.md) | System Architecture | `architecture/module-organization`, `architecture/request-lifecycle` |
| 02 | [02_query_engine_core.md](02_query_engine_core.md) | LLM Engine Core | `agents/tool-loop`, `architecture/streaming` |
| 03 | [03_prompt_engineering.md](03_prompt_engineering.md) | Prompt Construction | `prompt-engineering/system-prompt`, `prompt-engineering/hierarchical-assembly` |
| 04 | [04_tool_system.md](04_tool_system.md) | Tool Abstraction | `agents/tool-system`, `agents/tool-schema-design` |
| 05 | [05_context_management.md](05_context_management.md) | Context & Tokens | `context-management/token-counting`, `context-management/compression` |
| 06 | [06_api_layer_providers.md](06_api_layer_providers.md) | Multi-Provider API | `architecture/provider-abstraction`, `workflow/prompt-caching` |
| 07 | [07_retry_resilience.md](07_retry_resilience.md) | Retry & Fallback | `architecture/retry-strategy`, `architecture/model-fallback` |
| 08 | [08_streaming_concurrency.md](08_streaming_concurrency.md) | Streaming & Parallel | `architecture/streaming`, `agents/concurrent-execution` |
| 09 | [09_security_model.md](09_security_model.md) | LLM Security | `architecture/security`, `agents/prompt-injection-prevention` |
| 10 | [10_multi_agent.md](10_multi_agent.md) | Multi-Agent | `agents/multi-agent`, `agents/coordinator-pattern` |
| 11 | [11_mcp_integration.md](11_mcp_integration.md) | MCP Protocol | `agents/mcp-protocol`, `agents/tool-discovery` |
| 12 | [12_command_system.md](12_command_system.md) | Slash Commands | `architecture/command-system`, `architecture/extensibility` |
| 13 | [13_ui_terminal.md](13_ui_terminal.md) | Terminal UI | `architecture/terminal-ui`, `architecture/react-ink` |
| 14 | [14_state_persistence.md](14_state_persistence.md) | State & Sessions | `architecture/state-management`, `context-management/session-persistence` |
| 15 | [15_configuration_system.md](15_configuration_system.md) | Config & Flags | `architecture/configuration`, `architecture/feature-flags` |
| 16 | [16_performance_optimization.md](16_performance_optimization.md) | Performance | `architecture/performance`, `architecture/lazy-loading` |
| 17 | [17_ide_bridge.md](17_ide_bridge.md) | IDE Integration | `architecture/ide-integration`, `workflow/vscode-extension` |
| 18 | [18_patterns_and_lessons.md](18_patterns_and_lessons.md) | Patterns & Lessons | `architecture/design-patterns`, `agents/defensive-programming` |
| 19 | [19_hooks_system.md](19_hooks_system.md) | Hooks System | `architecture/hooks`, `workflow/lifecycle-events` |
| 20 | [20_extended_thinking.md](20_extended_thinking.md) | Extended Thinking | `agents/thinking-budget`, `architecture/reasoning` |
| 21 | [21_plugins_skills.md](21_plugins_skills.md) | Plugins & Skills | `architecture/plugin-system`, `workflow/skill-design` |
| 22 | [22_claude_md_convention.md](22_claude_md_convention.md) | CLAUDE.md Convention | `context-management/memory-hierarchy`, `workflow/project-config` |
| 23 | [23_prompt_caching.md](23_prompt_caching.md) | Prompt Caching Deep Dive | `workflow/prompt-caching`, `architecture/cost-optimization` |
| 24 | [24_testing_evaluation.md](24_testing_evaluation.md) | Testing & Evaluation | `workflow/testing`, `agents/evaluation-framework` |
| 25 | [25_supplementary_patterns.md](25_supplementary_patterns.md) | Supplementary Patterns | `architecture/telemetry`, `context-management/session-format` |

## Tech Stack 요약

| Category | Technology |
|----------|------------|
| Runtime | Bun v1.1.0+ |
| Language | TypeScript (strict mode) |
| Terminal UI | React 19 + Ink |
| CLI Parser | Commander.js |
| Validation | Zod v3.24 |
| Code Search | ripgrep |
| LLM SDK | Anthropic SDK v0.39.0 |
| Protocols | MCP SDK v1.12.1, LSP |
| Telemetry | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook v1.4.0 |
| Auth | OAuth 2.0, JWT, macOS Keychain |
| Web UI | Next.js 14.2, Zustand, Radix UI, Tailwind CSS |

## Codebase 규모

- **1,916** TypeScript files
- **512,000+** lines of code
- **42** agent tools
- **87** slash commands
- **~140** UI components
- **~80** React hooks
