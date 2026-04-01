<!--
tags: architecture/plugin-system, architecture/extensibility, agents/skill-definition, agents/agent-definition
keywords: plugin-system, skills, SKILL-md, plugin-json, marketplace, agent-definition, command-definition, hooks-integration, extensibility, plugin-dev
related_files: 04_tool_system.md, 10_multi_agent.md, 12_command_system.md, 19_hooks_system.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 21. Plugin & Skills Architecture

> Claude Code의 확장 모델: Plugin manifest, Skill 정의, Agent markdown, Command 확장, Marketplace 생태계. 32개 공식 플러그인의 설계 패턴 분석.


## 21.1 Plugin System 개요

Claude Code는 monolithic 구조가 아니라 **plugin-based 확장 모델**을 채택하고 있다. 이는 core 기능을 최소화하고, 대부분의 도메인 특화 기능을 plugin으로 분리하여 독립적으로 개발·배포·버전관리할 수 있게 하는 아키텍처적 결정이다.

핵심 설계 원칙:

- **32개 공식 플러그인**이 marketplace를 통해 배포된다
- **Plugin = commands + agents + skills + hooks + MCP servers의 번들**이다. 즉 plugin은 단일 기능이 아니라 관련 있는 여러 확장 요소를 하나의 배포 단위로 묶은 것이다
- Plugin은 Claude Code의 기존 extension point들(command system, agent system, hook system, MCP integration)을 조합하여 새로운 기능을 제공한다

이 아키텍처가 중요한 이유는, LLM 기반 도구에서 "기능 추가"가 코드 변경이 아니라 **Markdown 문서 추가**로 이루어진다는 패러다임 전환을 보여주기 때문이다. Agent 정의, Skill 정의, Command 정의 모두 Markdown + YAML frontmatter로 작성되며, 이는 개발자가 아닌 도메인 전문가도 LLM의 행동을 확장할 수 있게 한다.


## 21.2 Plugin Manifest

모든 plugin의 진입점은 `.claude-plugin/plugin.json` 파일이다. 이 manifest 파일이 plugin의 메타데이터와 구성 요소 위치를 선언한다.

### 21.2.1 Manifest 구조

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "What this plugin does",
  "author": { "name": "Author", "email": "...", "url": "..." },
  "keywords": ["testing", "automation"],
  "commands": "./custom-commands",
  "agents": ["./agents", "./specialized-agents"],
  "hooks": "./config/hooks.json",
  "mcpServers": "./.mcp.json"
}
```

### 21.2.2 필드별 상세 설명

**name** (필수)
- kebab-case 형식 (예: `my-plugin`, `code-review-pro`)
- Marketplace 전체에서 unique해야 한다
- Plugin 식별자이자 `/plugin-name` 형태의 command namespace가 된다

**version** (필수)
- Semantic versioning (major.minor.patch)
- Marketplace 업데이트 판단 기준

**description** (필수)
- Plugin의 용도를 설명하는 텍스트
- Marketplace 검색과 LLM의 plugin 선택에 사용된다

**author** (필수)
- name, email, url 중 하나 이상 포함

**keywords** (선택)
- Marketplace 검색 태그
- 카테고리 분류에 사용

**commands** (선택)
- Slash command 정의 파일(.md)이 있는 디렉토리 경로
- `./` 접두사, plugin 루트 기준 상대 경로
- 해당 디렉토리의 `.md` 파일이 각각 하나의 command가 된다

**agents** (선택)
- Sub-agent 정의 파일(.md)이 있는 디렉토리 경로
- 배열로 여러 디렉토리 지정 가능: `["./agents", "./specialized-agents"]`
- Auto-discovery: 지정된 디렉토리 내 `.md` 파일을 자동 탐색

**hooks** (선택)
- Hook 설정 파일 경로
- Claude Code의 hooks system과 통합된다 (19_hooks_system.md 참조)

**mcpServers** (선택)
- MCP 서버 설정 파일 경로
- Claude Code의 MCP integration과 통합된다 (11_mcp_integration.md 참조)

### 21.2.3 Auto-discovery 메커니즘

Plugin은 명시적 경로 지정 외에도 convention-based auto-discovery를 지원한다. Plugin 루트에 다음 디렉토리가 있으면 자동 탐색된다:

- `commands/` — Slash command 정의
- `agents/` — Sub-agent 정의
- `skills/` — Skill 정의
- `hooks/` — Hook 스크립트

이는 "Convention over Configuration" 원칙의 적용이다. Manifest에 명시적으로 경로를 지정하면 convention을 override할 수 있다.


## 21.3 Plugin 디렉토리 구조

표준적인 plugin 디렉토리 레이아웃:

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # 필수: 매니페스트
├── commands/                 # Slash commands (.md)
│   ├── review.md
│   └── analyze.md
├── agents/                   # Sub-agent 정의 (.md)
│   ├── code-explorer.md
│   └── code-reviewer.md
├── skills/
│   └── skill-name/
│       ├── SKILL.md         # 필수: Skill 정의
│       ├── scripts/         # 실행 코드
│       ├── references/      # 참조 문서
│       └── examples/        # 예제 파일
├── hooks/
│   ├── hooks.json           # Hook 설정
│   ├── pre-commit.py        # Python hooks
│   └── validate.sh          # Bash hooks
├── .mcp.json                # MCP 서버 정의
├── scripts/                 # 유틸리티 스크립트
├── README.md
└── LICENSE
```

각 디렉토리의 역할:

- **`.claude-plugin/`**: Plugin identity. `plugin.json`이 이 디렉토리 안에 있어야 Claude Code가 plugin으로 인식한다
- **`commands/`**: 사용자가 `/command-name`으로 호출하는 slash command 정의. 각 `.md` 파일이 하나의 command
- **`agents/`**: Claude Code가 task 수행 시 위임할 수 있는 sub-agent 정의. Model, tools, 행동 지침을 포함
- **`skills/`**: LLM이 context에 따라 자동 로드하거나 사용자가 명시적으로 호출하는 지식 모듈. 가장 풍부한 구조를 가짐
- **`hooks/`**: Claude Code의 lifecycle event에 반응하는 스크립트. Pre/post processing, validation 등
- **`.mcp.json`**: Plugin이 제공하는 MCP 서버 설정. External tool 통합
- **`scripts/`**: Plugin 내부에서 사용하는 유틸리티 스크립트 (validation, generation 등)


## 21.4 Skills: LLM이 호출하는 지식 모듈

Skill은 Claude Code plugin system에서 가장 독특하고 중요한 개념이다. 전통적인 plugin이 "코드를 실행"하는 것이라면, Skill은 **"LLM에게 지식과 지시사항을 주입"**하는 것이다.

### 21.4.1 SKILL.md Format

모든 skill의 핵심은 `SKILL.md` 파일이다:

```markdown
---
name: skill-identifier
description: >
  When to trigger: user asks to "create a hookify rule",
  "write a hook rule", "configure hookify"
version: 0.1.0
compatibility: ["Read", "Write", "Bash"]
---

# Skill Title

실제 지시사항 (Claude가 읽고 따르는 내용)

## Step 1: 분석
...

## Step 2: 구현
...
```

**Frontmatter 필드:**

| Field | Required | Purpose |
|-------|----------|---------|
| name | ✓ | Skill 식별자, kebab-case |
| description | ✓ | LLM이 언제 이 skill을 사용할지 판단하는 기준 |
| version | ✓ | Semantic versioning |
| compatibility | ✗ | 이 skill이 사용하는 tools 목록 |

**description의 중요성**: 이 필드는 단순한 설명이 아니라, **LLM의 routing 판단 기준**이다. "When to trigger" 형태로 구체적인 trigger 조건을 명시하는 것이 best practice이다. Claude는 사용자의 요청과 이 description을 매칭하여 해당 skill을 로드할지 결정한다.

**compatibility**: Skill이 사용하는 tool 목록. Skill 로드 시 해당 tool이 사용 가능한지 확인하는 데 쓰인다. 예를 들어 `["Read", "Write", "Bash"]`는 이 skill이 파일 읽기, 쓰기, 명령 실행을 필요로 함을 의미한다.

### 21.4.2 Model-Invoked vs User-Invoked

Skill에는 두 가지 호출 방식이 있다:

**Model-invoked (자동 호출)**
- Claude가 사용자 요청의 context를 분석하여, skill의 description과 매칭되면 자동으로 로드한다
- 예: 사용자가 "hookify rule을 만들어줘"라고 하면, description에 "create a hookify rule"이 포함된 skill이 자동 trigger된다
- 사용자는 skill의 존재를 알 필요가 없다
- Skill Tool을 통해 호출된다

**User-invoked (명시적 호출)**
- 사용자가 `/skill-name`으로 직접 호출한다
- 예: `/code-review`, `/create-skill`
- 사용자가 의도적으로 특정 workflow를 실행하고 싶을 때 사용한다

이 이중 호출 모델은 skill의 접근성을 극대화한다. 자주 사용하는 skill은 model-invoked로 자연스럽게 활성화되고, 특수한 workflow는 user-invoked로 명시적 제어가 가능하다.

### 21.4.3 Context 전략: Progressive Disclosure

Skill system의 핵심 설계 패턴은 **Progressive Disclosure**이다. 이는 LLM의 context window를 효율적으로 사용하기 위한 3-tier 로딩 전략이다:

```
Tier 1 — 항상 로드: skill 이름 + description (~100 tokens)
Tier 2 — Trigger 시: SKILL.md body (~500 lines)
Tier 3 — 필요 시: references/ 디렉토리 (무제한)
```

**Tier 1: Metadata (항상 로드)**
- Skill의 name과 description만 context에 포함
- ~100 tokens 정도의 아주 작은 비용
- 10개의 skill이 있어도 ~1,000 tokens만 소비
- Claude가 "이 skill이 필요한가?"를 판단하는 데 사용

**Tier 2: SKILL.md Body (Trigger 시 로드)**
- Skill이 trigger되면 SKILL.md의 전체 body를 context에 로드
- 500 lines 이하 권장 (너무 길면 context 낭비)
- Claude가 "어떻게 이 skill을 실행할 것인가?"를 결정하는 데 사용
- 구체적인 step-by-step 지시사항, 규칙, 패턴 포함

**Tier 3: References (필요 시 로드)**
- `references/` 디렉토리의 파일은 Claude가 Read tool로 on-demand 로드
- 용량 제한 없음 (context window 한도 내에서)
- API 문서, 상세 스펙, 대규모 예제 등

이 패턴의 장점: **Skill 수에 비례하여 context가 선형 증가하지 않는다**. 100개의 skill을 설치해도 ~10,000 tokens만 항상 소비하고, 실제 사용되는 1-2개의 skill만 full context를 차지한다.

### 21.4.4 Skill 디렉토리 구조의 의미

```
skills/
└── my-skill/
    ├── SKILL.md         # 필수: 정의 + 지시사항
    ├── scripts/         # 실행 코드 (Claude가 Bash tool로 실행)
    ├── references/      # 참조 문서 (Claude가 Read tool로 로드)
    └── examples/        # 예제 (Claude가 참고하여 output 생성)
```

- **`scripts/`**: Skill 실행 중 Claude가 Bash tool로 호출하는 스크립트. Validation, generation, transformation 등의 작업 수행
- **`references/`**: Skill이 참조하는 문서. API 스펙, 스타일 가이드, 프로토콜 정의 등. SKILL.md에서 "필요 시 references/api-spec.md를 읽어라"와 같이 지시
- **`examples/`**: 예제 입력/출력. Claude가 few-shot learning으로 활용. "examples/ 디렉토리의 파일을 참고하여 동일한 패턴으로 생성하라"


## 21.5 Agent Definitions: Markdown으로 정의하는 Sub-Agent

Agent definition은 Claude Code의 multi-agent architecture(10_multi_agent.md)에서 사용되는 sub-agent를 Markdown으로 정의하는 방식이다.

### 21.5.1 Agent Definition Format

```markdown
---
name: code-reviewer
description: >
  Use this agent when reviewing code changes.

  <example>
  Context: User asks to review a PR
  user: "Review my changes"
  assistant: "[launches code-reviewer agent]"
  </example>

model: sonnet
color: red
tools: ["Read", "Grep", "Glob"]
---

You are a code reviewer. Your responsibilities:
1. Check for bugs and security issues
2. Evaluate code quality and maintainability
3. Suggest improvements with specific code examples
4. Rate confidence for each finding (high/medium/low)

## Review Process
1. First, understand the overall change scope using Glob and Grep
2. Read each changed file carefully
3. Check for common anti-patterns
4. Produce a structured review report
```

### 21.5.2 Frontmatter 필드 상세

| Field | Required | Purpose | 값 예시 |
|-------|----------|---------|---------|
| name | ✓ | Agent 식별자, kebab-case, 3-50 chars | `code-reviewer` |
| description | ✓ | Trigger 조건 + examples | 아래 설명 |
| model | ✓ | 사용할 LLM 모델 | `inherit`, `sonnet`, `opus`, `haiku` |
| color | ✓ | UI에서의 식별색 | `red`, `blue`, `green` |
| tools | ✗ | 사용 가능한 tool 목록 | `["Read", "Grep", "Glob"]` |

**name**: Kebab-case, 3-50 글자. Main agent가 이 이름으로 sub-agent를 호출한다.

**description**: 가장 중요한 필드. Main agent(orchestrator)가 이 description을 보고 해당 sub-agent에게 task를 위임할지 결정한다. `<example>` 블록을 포함하는 것이 강력히 권장된다. Example은 LLM이 "이 상황에서 이 agent를 사용해야 한다"를 판단하는 데 description 텍스트보다 훨씬 효과적이다.

**model**: Sub-agent가 사용할 LLM 모델.
- `inherit`: Parent agent와 동일한 모델 사용
- `sonnet`: Claude Sonnet (빠르고 저렴, 탐색/검증 작업에 적합)
- `opus`: Claude Opus (느리고 비싸지만 높은 추론 능력, 설계/복잡한 판단에 적합)
- `haiku`: Claude Haiku (가장 빠르고 저렴, 단순 작업에 적합)

**color**: Terminal UI에서 agent의 출력을 시각적으로 구분하기 위한 색상. 여러 agent가 동시에 실행될 때 어떤 agent의 출력인지 구분 가능.

**tools**: Agent가 사용할 수 있는 tool의 allowlist. 지정하지 않으면 모든 tool 사용 가능. 보안과 행동 제어를 위해 제한하는 것이 권장된다. 예를 들어 "탐색 전용" agent는 `["Read", "Grep", "Glob"]`만 허용하여 파일 수정을 원천 차단한다.

### 21.5.3 Agent Body: System Prompt

Frontmatter 아래의 Markdown body는 sub-agent의 system prompt가 된다. 이 부분에서 agent의:
- 역할과 책임 정의
- 작업 프로세스 (step-by-step)
- 출력 형식 요구사항
- 제약 조건

을 명시한다. Claude는 이 body를 system prompt로 받아 해당 역할을 수행한다.

### 21.5.4 LLM 엔지니어링 인사이트

**Model 선택이 비용을 결정한다**: 실제 plugin에서는 작업 특성에 따라 model을 세분화한다.
- `code-explorer` (탐색): sonnet — 빠르게 코드를 읽고 구조를 파악
- `code-architect` (설계): opus — 복잡한 아키텍처 결정
- `code-reviewer` (검증): sonnet — 패턴 매칭 기반 리뷰

이런 model tiering은 비용을 50-70% 절감할 수 있다 (모든 작업에 opus를 쓰는 것 대비).

**Tools 제한으로 행동 범위를 강제한다**: Explore agent에 Write/Edit tool을 주지 않으면, 아무리 "이 파일을 수정하면 좋겠다"고 판단해도 실행할 수 없다. 이는 multi-agent system에서 각 agent의 책임 분리를 enforcing하는 메커니즘이다.

**`<example>` 블록의 위력**: LLM이 agent를 선택할 때, abstract한 description보다 concrete한 example이 더 정확한 trigger 역할을 한다. 이는 LLM의 in-context learning 특성을 활용한 것이다.


## 21.6 Command Definitions

Command는 사용자가 `/command-name`으로 호출하는 slash command를 정의한다. Agent definition과 유사하지만, 목적이 다르다: Agent는 "Claude가 위임하는 sub-task 수행자"이고, Command는 "사용자가 명시적으로 실행하는 workflow"이다.

### 21.6.1 Command Definition Format

```markdown
---
description: Review code for bugs, security issues, and quality
argument-hint: "PROMPT [--max-iterations N]"
allowed-tools: ["Read", "Write", "Edit", "Bash(git:*)"]
model: sonnet
disable-model-invocation: false
---

# /code-review Implementation

When this command is invoked:
1. Analyze the changed files using git diff
2. Run parallel review agents for each category
3. Aggregate findings with confidence scores
4. Present a structured report

## Review Categories
- Security vulnerabilities
- Performance issues
- Code quality and maintainability
- Test coverage gaps

## Output Format
...
```

### 21.6.2 Command Frontmatter 필드

**description** (필수): Command의 용도. `/help`에서 표시되고, model-invoked일 때 trigger 판단에 사용.

**argument-hint** (선택): 사용자에게 보여주는 인자 힌트. 예: `"PROMPT [--max-iterations N]"`

**allowed-tools** (선택): Command 실행 중 사용 가능한 tool. 특히 `Bash(git:*)`와 같은 문법으로 특정 subcommand만 허용할 수 있다:
- `Bash(git:*)` — git으로 시작하는 모든 bash 명령 허용
- `Bash(npm:test)` — `npm test`만 허용
- 이는 command의 보안 범위를 세밀하게 제어한다

**model** (선택): Command 실행 시 사용할 모델.

**disable-model-invocation** (선택): `true`로 설정하면 model-invoked 호출을 비활성화. 사용자의 명시적 `/command` 호출만 허용.

### 21.6.3 핵심 개념: Commands는 Claude에게 보내는 지시

Command body는 **사용자에게 보여주는 documentation이 아니라, Claude에게 보내는 지시사항**이다. 사용자가 `/code-review`를 입력하면, command body 전체가 Claude의 prompt에 inject된다. 따라서 body는 Claude가 이해하고 따를 수 있는 명확한 지시 형태로 작성해야 한다.

이는 12_command_system.md에서 다룬 command system과 동일한 메커니즘이지만, plugin context에서는 외부 배포가 가능하다는 차이가 있다.


## 21.7 Marketplace & Policy

Plugin ecosystem의 건전성을 유지하기 위한 정책 시스템이 존재한다.

### 21.7.1 known_marketplaces.json

공식 인증된 marketplace 목록:

```json
["https://github.com/anthropics/claude-plugins-official"]
```

Claude Code는 이 목록에 있는 marketplace에서만 plugin을 검색하고 설치한다. 이는 supply chain attack을 방지하기 위한 첫 번째 방어선이다.

### 21.7.2 blocklist.json

보안 위반, 악성 행위 등으로 차단된 plugin 목록. Claude Code는 설치 시 이 목록을 확인하고, 차단된 plugin은 설치를 거부한다. 이미 설치된 plugin이 나중에 차단 목록에 추가되면, 다음 실행 시 경고를 표시한다.

### 21.7.3 skills_policy.json: 설치 게이트

Plugin 설치 시 최소 요구사항을 정의하는 정책:

```json
{
  "minStars": 15,
  "minDownloads": 2000,
  "minInstallsCurrent": 5,
  "allowSuspicious": false
}
```

- **minStars**: GitHub 최소 star 수. Community trust의 proxy
- **minDownloads**: 총 다운로드 수. 충분히 검증된 plugin만 허용
- **minInstallsCurrent**: 현재 활성 설치 수. 실제 사용 중인 plugin만 허용
- **allowSuspicious**: Heuristic 기반 의심 플래그가 있는 plugin 허용 여부

이 정책은 사용자가 조정할 수 있으며, 엄격한 기업 환경에서는 높은 threshold를, 실험적 환경에서는 낮은 threshold를 설정할 수 있다.


## 21.8 실전 패턴 분석

32개 공식 plugin에서 반복적으로 나타나는 설계 패턴들을 분석한다.

### 21.8.1 Pattern 1: Multi-Agent Orchestration (code-review plugin)

```
[Main Agent (Orchestrator)]
        │
        ├── [Security Reviewer] ──→ findings + confidence
        ├── [Performance Reviewer] ──→ findings + confidence
        ├── [Quality Reviewer] ──→ findings + confidence
        └── [Test Coverage Reviewer] ──→ findings + confidence
                │
        [Aggregator: high confidence만 필터링]
                │
        [Final Report]
```

핵심 설계:
- **4개 parallel agent 실행**: 각 agent가 독립적으로 코드를 분석. Parallel 실행으로 latency 최소화
- **Confidence score 반환**: 각 agent가 발견한 이슈에 high/medium/low confidence를 부여
- **High confidence 이슈만 최종 리포트에 포함**: False positive를 줄여 사용자 신뢰 확보
- **Model 선택**: Review agent들은 모두 sonnet 사용. 패턴 매칭 기반 리뷰에 opus는 과잉

이 패턴은 "여러 관점에서 동시에 분석하고, 결과를 종합"하는 모든 task에 적용 가능하다.

### 21.8.2 Pattern 2: Phased Workflow (feature-dev plugin)

7-phase 순차 실행 구조:

```
Phase 1: Understand (요구사항 분석)
    ↓
Phase 2: Explore (코드베이스 탐색) — agent: code-explorer (sonnet)
    ↓
Phase 3: Plan (설계) — agent: code-architect (opus)
    ↓
Phase 4: Implement (구현)
    ↓
Phase 5: Test (테스트 작성/실행)
    ↓
Phase 6: Review (자동 코드 리뷰) — agent: code-reviewer (sonnet)
    ↓
Phase 7: Complete (정리/보고)
```

핵심 설계:
- **각 phase에 전용 agent 배정**: Phase 2는 code-explorer(sonnet, Read-only tools), Phase 3는 code-architect(opus, 모든 tools)
- **Phase 간 결과물이 다음 phase의 input**: Phase 2의 탐색 결과가 Phase 3의 설계 context가 된다
- **Model tiering 적용**: 탐색(sonnet) → 설계(opus) → 검증(sonnet). 가장 비싼 opus는 가장 중요한 설계 phase에만 사용

이 패턴은 소프트웨어 개발의 전체 lifecycle을 자동화하는 데 적합하다.

### 21.8.3 Pattern 3: Meta-Plugin (plugin-dev plugin)

"Plugin을 만드는 plugin"이라는 재귀적 구조:

```
plugin-dev/
├── skills/
│   ├── create-plugin/       # 새 plugin scaffold 생성
│   ├── create-agent/        # Agent definition 작성
│   ├── create-skill/        # Skill definition 작성
│   ├── create-command/      # Command definition 작성
│   ├── create-hook/         # Hook 설정 생성
│   ├── validate-plugin/     # Plugin 구조 검증
│   └── publish-plugin/      # Marketplace 배포
├── scripts/
│   ├── validate-agent.sh    # Agent frontmatter 유효성 검사
│   ├── validate-hook-schema.sh  # Hook schema 검증
│   └── scaffold.sh          # Plugin template 생성
└── references/
    ├── agent-spec.md        # Agent definition 스펙
    ├── skill-spec.md        # Skill definition 스펙
    └── command-spec.md      # Command definition 스펙
```

핵심 설계:
- **7개 skill로 plugin 개발 전체 lifecycle 커버**: 생성 → 작성 → 검증 → 배포
- **Validation scripts**: Shell 스크립트로 구조적 검증. YAML frontmatter 필수 필드, 파일 구조, naming convention 등 확인
- **Reference docs + examples 번들**: Skill이 참조할 스펙 문서와 예제를 함께 배포

이 패턴은 도구가 자신을 확장하는 생태계를 만드는 bootstrapping 전략이다.

### 21.8.4 Pattern 4: Eval-Driven Development (skill-creator plugin)

```
Skill 작성
    ↓
Eval 실행 (자동화된 테스트)
    ↓
결과 분석
    ↓
Skill 개선
    ↓
(반복)
```

핵심 설계:
- **"Test first" 패턴을 LLM skill 개발에 적용**: Skill을 작성하기 전에 eval(평가 기준)을 먼저 정의
- **자동화된 eval 실행**: Skill의 output이 기대치를 충족하는지 자동 검증
- **결과 기반 iterative 개선**: Eval 결과를 보고 skill의 지시사항을 수정

이는 전통적 소프트웨어의 TDD(Test-Driven Development)를 LLM prompt engineering에 적용한 것이다. Prompt의 품질을 객관적으로 측정하고 개선할 수 있게 한다.


## 21.9 LLM 엔지니어링 인사이트

Claude Code의 plugin & skills architecture에서 도출할 수 있는 핵심 교훈들:

### 21.9.1 Markdown이 DSL이다

Agent, Skill, Command 정의가 모두 **Markdown + YAML frontmatter**로 작성된다. 이는 의도적인 설계 결정이다:

- **코드 변경 없이 LLM 행동 변경 가능**: Markdown 파일만 수정하면 agent의 역할, 사용 tool, 모델이 바뀐다
- **버전 관리 용이**: Git으로 LLM 행동의 변경 이력을 추적할 수 있다
- **비개발자 접근성**: 프로그래밍 없이도 도메인 전문가가 agent를 정의할 수 있다
- **LLM 친화적**: Markdown은 LLM training data에 풍부하여 LLM이 잘 이해하는 형식이다

### 21.9.2 Progressive Disclosure 패턴

Context window는 유한한 자원이다. 모든 skill의 전체 내용을 항상 로드하면 context가 빠르게 소진된다. Progressive Disclosure는 이 문제를 해결한다:

```
항상 로드: metadata (~100 tokens/skill)
Trigger 시: body (~500 lines/skill)
필요 시: references (무제한)
```

이 패턴은 "정보의 비용"을 계층화하여, **최소 비용으로 최대 접근성**을 달성한다. 100개의 skill을 설치해도 평소에는 ~10,000 tokens만 소비한다.

### 21.9.3 Model Tiering in Practice

실제 production plugin에서의 model 사용 패턴:

- **탐색/검색** (코드 읽기, 패턴 찾기): Sonnet — 빠르고 저렴, 충분한 이해력
- **설계/판단** (아키텍처 결정, 복잡한 추론): Opus — 느리고 비싸지만 높은 추론력
- **검증/리뷰** (패턴 매칭, 규칙 확인): Sonnet — 정해진 기준에 따른 판단

이 tiering을 적용하면 모든 작업에 opus를 사용하는 것 대비 **비용을 50-70% 절감**하면서 품질은 유지할 수 있다.

### 21.9.4 Plugin은 배포 단위

Plugin은 기술적으로는 여러 extension의 번들이지만, 본질적으로는 **관련 있는 기능의 배포 단위**이다:

- Commands + Agents + Skills + Hooks를 하나의 단위로 묶어 **함께 배포/버전관리**
- 사용자는 개별 agent나 skill이 아니라 plugin 단위로 설치/제거
- 의존성 관리: Plugin 내의 agent가 같은 plugin의 skill을 참조할 때, 항상 함께 존재함이 보장됨

이는 npm package나 Python package와 동일한 배포 철학이다.

### 21.9.5 `<example>` 블록의 위력

Agent와 Skill의 description에서 `<example>` 블록은 단순한 문서화가 아니라 **LLM의 의사결정에 직접 영향을 미치는 few-shot prompt**이다:

```markdown
description: >
  Use this agent when reviewing code changes.

  <example>
  Context: User asks to review a PR
  user: "Review my changes"
  assistant: "[launches code-reviewer agent]"
  </example>
```

Abstract한 "Use this agent when reviewing code"보다 concrete한 example이 LLM의 agent 선택 정확도를 크게 높인다. 이는 LLM의 in-context learning 특성을 직접 활용한 것이며, prompt engineering의 핵심 기법이다.


## 21.10 요약

Claude Code의 Plugin & Skills Architecture는 LLM 기반 도구의 확장 모델이 어떠해야 하는지를 보여주는 실전 사례이다.

핵심 아이디어:
1. **Plugin = 배포 단위**. Commands, Agents, Skills, Hooks, MCP를 하나로 묶는다
2. **Markdown = DSL**. 코드가 아니라 문서로 LLM 행동을 정의한다
3. **Progressive Disclosure**. Context 비용을 3-tier로 계층화한다
4. **Model Tiering**. 작업 복잡도에 따라 모델을 분리하여 비용을 최적화한다
5. **Marketplace + Policy**. 생태계의 품질과 보안을 제도적으로 관리한다

이 architecture는 "LLM의 행동을 어떻게 모듈화하고, 배포하고, 조합하는가?"라는 질문에 대한 Anthropic의 답이다.

---

## 대안 비교

| 접근 방식 | 확장 범위 | 구현 복잡도 | Context 효율 | 비개발자 접근성 | 적합한 상황 |
|----------|---------|-----------|-------------|-------------|-----------|
| **Plugin System (Claude Code 방식)** | 최대 (commands+agents+skills+hooks) | 높음 | ✅ Progressive Disclosure | ✅ Markdown DSL | 종합적 LLM 도구 플랫폼 |
| **MCP Servers** | 도구(tool) 레벨 | 중간 | 중간 (schema 로드 필요) | ❌ 서버 구현 필요 | 외부 API/서비스 연동 |
| **Code Generation (동적 코드 생성)** | 무제한 | 낮음 | ❌ 매번 생성 | ❌ | 예측 불가능한 일회성 작업 |
| **Fixed Toolset (고정 도구)** | 제한적 | 최저 | ✅ 고정 schema | N/A | MVP, 도메인 특화 에이전트 |
| **Prompt Injection 기반 확장** | 중간 | 낮음 | ❌ context 소비 큼 | ✅ | 빠른 프로토타이핑 |

**선택 기준**: 팀이 LLM 도구의 행동을 지속적으로 확장해야 한다면 Plugin system이 최적이다. 외부 서비스 연동만 필요하면 MCP server가 더 가볍다. 빠른 MVP는 fixed toolset으로 시작하고, 확장 필요가 생기면 plugin으로 마이그레이션한다.

---

## 내 프로젝트에 적용하기

### Step 1: Plugin Manifest 포맷 정의

프로젝트에 plugin 시스템을 도입할 때, 최소 manifest부터 시작한다:
```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "What this plugin does",
  "skills": "./skills",
  "agents": "./agents"
}
```
핵심은 **convention-based auto-discovery**를 채택하는 것이다. `skills/`, `agents/`, `commands/` 디렉토리에 `.md` 파일을 놓으면 자동 인식되도록 설계한다. 이렇게 하면 manifest를 최소화하면서도 확장이 용이하다.

### Step 2: Skill의 Progressive Disclosure 구현

모든 skill을 한꺼번에 context에 로드하면 token 폭발이 발생한다. 3-tier 로딩을 구현한다:
- **Tier 1 (항상 로드)**: 각 skill의 `name` + `description`만 (~100 tokens/skill). 10개 skill이 있어도 ~1,000 tokens
- **Tier 2 (trigger 시 로드)**: 사용자 요청이 skill description과 매칭되면 `SKILL.md` body 전체를 context에 주입
- **Tier 3 (on-demand)**: `references/` 디렉토리의 상세 문서는 LLM이 Read tool로 필요시 로드

이 패턴으로 100개 skill을 설치해도 평소 context 소비는 ~10,000 tokens에 불과하다.

### Step 3: Agent 정의에 Model Tiering 적용

Sub-agent를 정의할 때 작업 특성에 따라 모델을 분리한다:
- **탐색/검색 agent**: `model: sonnet` — Read, Grep, Glob만 허용. 빠르고 저렴
- **설계/판단 agent**: `model: opus` — 모든 tools 허용. 복잡한 추론이 필요한 핵심 phase에만 사용
- **검증/리뷰 agent**: `model: sonnet` — 패턴 매칭 기반, 정해진 기준에 따른 판단

Agent frontmatter의 `tools` 필드로 사용 가능 tool을 제한하면, model 비용 절감과 동시에 행동 범위를 강제할 수 있다. 모든 작업에 opus를 쓰는 것 대비 **50-70% 비용 절감**이 가능하다.
