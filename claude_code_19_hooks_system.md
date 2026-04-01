<!--
tags: agents/hooks-system, architecture/event-driven, agents/deterministic-control, llm-application/safety-guardrails, architecture/plugin-system
keywords: hooks, PreToolUse, PostToolUse, Stop, SubagentStop, SessionStart, SessionEnd, UserPromptSubmit, PreCompact, Notification, command-hook, prompt-hook, matcher, exit-codes, systemMessage, security-guardian, infinite-loop-agent, rule-engine, deterministic-control
related_files: 00_index.md, 04_tool_system.md, 09_security_model.md, 10_multi_agent.md, 15_configuration_system.md, 18_patterns_and_lessons.md
source: https://github.com/nirholas/claude-code
-->

소스: https://github.com/nirholas/claude-code


# 19. Hooks System

> LLM 워크플로우의 각 lifecycle 지점에 결정론적 검증·제어 로직을 삽입하는 이벤트 기반 확장 시스템.

## Keywords
hooks, PreToolUse, PostToolUse, Stop, SubagentStop, SessionStart, SessionEnd, UserPromptSubmit, PreCompact, Notification, command-hook, prompt-hook, matcher, exit-codes, systemMessage, security-guardian, infinite-loop-agent, rule-engine, deterministic-control, hybrid-architecture

---

## Hooks란 무엇인가

### 핵심 개념

Hooks는 **LLM 워크플로우에 결정론적(deterministic) 제어를 삽입하는 메커니즘**이다. LLM 기반 에이전트는 본질적으로 확률적(stochastic)이다. 같은 입력에 다른 출력을 낼 수 있고, 가끔 틀리며, 예상치 못한 행동을 할 수 있다. Hooks는 이 확률적 시스템의 특정 지점에 "반드시 이것을 확인하라"는 결정론적 체크포인트를 삽입한다.

### 왜 필요한가

**확률적(LLM) + 결정론적(Hooks)의 조합이 프로덕션 LLM 앱의 핵심 아키텍처 패턴이다.** 그 이유는 세 가지다:

1. **LLM은 가끔 틀린다**: 파일을 잘못된 경로에 쓰거나, 민감한 정보를 포함하거나, 존재하지 않는 API를 호출할 수 있다. 결정론적 검증이 이를 사전에 차단한다.
2. **위험한 행동을 할 수 있다**: `rm -rf /`, API key 하드코딩, 프로덕션 DB에 직접 쿼리 등. 이런 행동은 "가끔 잘 판단하는" 수준이 아니라 "절대 허용하지 않는" 수준의 제어가 필요하다.
3. **외부 시스템과의 연동이 필요하다**: CI/CD 파이프라인 트리거, 슬랙 알림 전송, 로그 기록 등은 LLM의 판단이 아니라 시스템 이벤트에 대한 결정론적 반응이다.

### 설계 철학

LLM의 창의성과 유연성은 그대로 유지하되, **안전(safety), 보안(security), 품질(quality)은 결정론적으로 보장**한다. 이것이 Hooks 시스템의 근본 철학이다. LLM에게 "위험한 코드를 쓰지 마세요"라고 프롬프트하는 것과, Hook으로 "API key 패턴이 감지되면 무조건 차단"하는 것은 본질적으로 다른 신뢰 수준이다.

---

## 9개 Hook Event Types

Claude Code의 Hooks 시스템은 에이전트 lifecycle의 9개 지점에 이벤트를 정의한다. 각 이벤트는 특정 시점에 발생하며, 그 시점에서 할 수 있는 행동이 다르다.

### 1. PreToolUse — Tool 실행 전

**Lifecycle 위치**: LLM이 tool 호출을 결정한 후, 실제 tool이 실행되기 전.

가장 빈번하게 사용되는 hook이다. `matcher`로 특정 tool만 타겟할 수 있으며, 세 가지 결정을 내릴 수 있다:
- **승인(allow)**: tool 실행을 허용
- **거부(deny/block)**: tool 실행을 차단하고, 차단 사유를 LLM에게 전달
- **수정(modify)**: tool input을 변환하여 수정된 입력으로 실행

**대표 사용 사례**: 파일 쓰기 전 보안 패턴 검사, 특정 디렉토리 접근 차단, Bash 명령어 화이트리스트 적용.

### 2. PostToolUse — Tool 실행 후

**Lifecycle 위치**: Tool이 실행을 완료하고 결과를 반환한 직후, LLM에게 결과가 전달되기 전.

tool 실행 결과를 검증하고, 품질을 체크하거나, 부수 효과(side effect)를 트리거하는 데 사용한다.

**대표 사용 사례**: 생성된 코드의 lint 결과 확인, 테스트 커버리지 체크, 외부 시스템에 변경 이력 기록.

### 3. Stop — 세션 종료 시도 시

**Lifecycle 위치**: LLM이 작업 완료를 선언하고 세션을 종료하려 할 때.

이 hook의 가장 강력한 용도는 **완료 기준을 강제**하는 것이다. LLM이 "다 했습니다"라고 하더라도, 특정 조건을 만족하지 않으면 세션 종료를 차단하고 추가 작업을 요구할 수 있다. 이를 통해 **반복 루프(iteration loop)**를 구현할 수 있다.

**대표 사용 사례**: 모든 테스트가 통과했는지 확인, 코드 리뷰 체크리스트 완료 여부 검증, 자동 반복 에이전트 구현.

### 4. SubagentStop — Sub-agent 종료 시

**Lifecycle 위치**: 메인 에이전트가 생성한 sub-agent(Task tool로 생성)가 작업을 완료하려 할 때.

Stop hook과 동일한 메커니즘이지만, sub-agent에 대해서만 동작한다. 멀티 에이전트 아키텍처에서 각 sub-agent의 완료 기준을 개별적으로 제어할 수 있다.

### 5. SessionStart — 세션 초기화

**Lifecycle 위치**: 에이전트 세션이 시작될 때, 첫 번째 사용자 입력이 처리되기 전.

환경 설정, 컨텍스트 로딩, 상태 파일 초기화 등을 수행한다.

**대표 사용 사례**: 프로젝트별 환경 변수 설정, 이전 세션 상태 복원, 사전 조건 검증(필요한 CLI tool 설치 여부 등).

### 6. SessionEnd — 세션 정리

**Lifecycle 위치**: 세션이 정상적으로 종료될 때.

리소스 정리, 최종 보고, 상태 저장 등을 수행한다.

**대표 사용 사례**: 세션 통계 기록, 임시 파일 정리, 작업 요약 슬랙 전송.

### 7. UserPromptSubmit — 사용자 입력 처리 전

**Lifecycle 위치**: 사용자가 프롬프트를 입력한 후, LLM에게 전달되기 전.

사용자 입력을 검증하거나 변환할 수 있다. 입력 필터링, 자동 컨텍스트 추가, 입력 형식 정규화 등에 사용한다.

**대표 사용 사례**: 금지된 명령어 필터링, 자동으로 프로젝트 컨텍스트 주입, 입력 로깅.

### 8. PreCompact — Context 압축 전

**Lifecycle 위치**: 대화가 길어져서 context window를 압축(compact)하기 직전.

context 압축은 오래된 대화 내용을 요약하여 토큰을 절약하는 과정이다. 이 hook으로 **압축 시 반드시 보존해야 할 정보**를 지정할 수 있다.

**대표 사용 사례**: 핵심 아키텍처 결정사항 보존 태그 지정, 중요 변수/경로 정보 보존.

### 9. Notification — 로깅, 알림

**Lifecycle 위치**: 시스템 알림이 발생할 때.

비간섭적(non-intrusive) hook이다. 워크플로우를 차단하거나 수정하지 않고, 순수하게 관찰(observation) 목적으로 사용한다.

**대표 사용 사례**: 외부 로깅 시스템에 이벤트 전송, 모니터링 대시보드 업데이트, 슬랙/이메일 알림.

---

## Hook Contract: 통신 프로토콜

Hooks 시스템의 핵심은 **명확한 통신 프로토콜(contract)**이다. Hook과 Claude Code 사이의 데이터 교환 방식은 엄격하게 정의되어 있다.

### Hook Types

Hook은 두 가지 타입으로 나뉜다.

#### Command Hook

외부 프로세스(스크립트, 바이너리)를 실행하는 hook이다. stdin으로 JSON을 받고, stdout으로 JSON을 반환하며, exit code로 결정을 전달한다.

```json
{
  "type": "command",
  "command": "bash ${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh",
  "timeout": 60
}
```

command hook은 **어떤 언어로든** 구현할 수 있다. Python, Bash, Node.js, Go 바이너리 등 stdin/stdout/exit code를 지원하는 모든 것이 가능하다.

#### Prompt Hook

LLM 자체를 활용하여 검증하는 hook이다. 별도의 LLM 호출로 주어진 prompt를 평가한다.

```json
{
  "type": "prompt",
  "prompt": "Evaluate if this tool use is appropriate: $TOOL_INPUT",
  "timeout": 30
}
```

command hook이 결정론적 검증(regex, 파일 존재 확인 등)에 적합하다면, prompt hook은 **맥락 이해가 필요한 검증**(코드 의도 파악, 변경의 적절성 판단 등)에 적합하다.

### Stdin Input (JSON)

Hook이 실행될 때, 표준 입력(stdin)으로 다음 형태의 JSON이 전달된다:

```json
{
  "session_id": "uuid",
  "tool_name": "Write|Edit|Bash|etc",
  "tool_input": {
    "file_path": "/path/to/file",
    "content": "...",
    "command": "..."
  },
  "transcript_path": "/path/to/transcript.jsonl"
}
```

각 필드의 의미:
- **`session_id`**: 현재 세션의 고유 식별자. 세션별 상태 관리에 사용.
- **`tool_name`**: 실행되려는 tool의 이름. PreToolUse/PostToolUse에서 제공.
- **`tool_input`**: tool에 전달될 입력 데이터. tool 종류에 따라 구조가 다르다.
- **`transcript_path`**: 현재 세션의 전체 대화 기록 파일 경로. 대화 맥락을 분석해야 할 때 사용.

### Exit Codes

Command hook의 exit code는 **결정(decision)**을 전달한다:

- **`0` (allow, 승인)**: Hook이 해당 동작을 허용한다. stdout에 추가 정보가 있으면 그것도 처리된다.
- **`2` (block, 차단)**: Hook이 해당 동작을 차단한다. stdout에 차단 사유를 포함할 수 있다.

이 설계의 핵심은 **단순함**이다. 0과 2, 단 두 개의 exit code만 사용한다. 복잡한 상태 코드 체계를 만드는 대신, 세부 정보는 JSON stdout으로 전달한다. 이것은 Unix 철학의 "단순한 것이 좋다"를 따르면서도, JSON으로 풍부한 정보를 전달할 수 있는 우아한 설계다.

### Stdout Output (JSON)

Hook의 표준 출력(stdout)으로 반환하는 JSON은 다음 구조를 갖는다:

```json
{
  "hookSpecificOutput": {
    "permissionDecision": "allow|deny|ask",
    "updatedInput": {"field": "modified_value"}
  },
  "systemMessage": "Optional message injected to Claude's context"
}
```

각 필드의 역할:
- **`hookSpecificOutput`**: hook event 종류에 따라 다른 구조. PreToolUse에서는 `permissionDecision`으로 승인/거부/사용자에게 위임을 선택하고, `updatedInput`으로 tool input을 수정할 수 있다.
- **`systemMessage`**: **LLM의 context에 직접 주입되는 메시지**. 이것이 Hooks 시스템의 가장 강력한 기능 중 하나다. Hook이 단순히 차단/허용만 하는 것이 아니라, LLM에게 "왜 이것이 차단되었는지", "대신 무엇을 해야 하는지"를 알려줄 수 있다.

### Stop Hook Special Output

Stop hook은 세션 종료를 제어하므로, 특별한 출력 구조를 갖는다:

```json
{
  "decision": "block",
  "reason": "Reinject this as next prompt",
  "systemMessage": "Status message"
}
```

- **`decision`**: "block"이면 세션 종료를 차단한다.
- **`reason`**: 차단 사유. 이 문자열이 **다음 사용자 프롬프트로 재주입**된다. 이를 통해 Stop hook이 LLM에게 "아직 끝나지 않았다, 이것을 더 해라"라는 지시를 줄 수 있다.
- **`systemMessage`**: 시스템 컨텍스트에 추가될 메시지.

---

## Configuration

### settings.json (직접 설정)

사용자가 `~/.claude/settings.json` 또는 프로젝트의 `.claude/settings.json`에서 직접 hooks를 설정할 수 있다:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "/path/to/script.sh",
        "matcher": "Edit|Write",
        "timeout": 10
      }
    ]
  }
}
```

각 hook event type은 **배열**로 여러 hook을 등록할 수 있다. 등록된 순서대로 실행되며, 하나라도 차단하면 해당 동작은 차단된다.

### Plugin hooks.json (플러그인을 통한 설정)

플러그인은 자체 `hooks.json` 파일로 hooks를 정의한다:

```json
{
  "description": "Security hooks",
  "hooks": {
    "PreToolUse": [...]
  }
}
```

플러그인 방식의 장점은 **배포 가능성**이다. 보안 팀이 만든 hook 세트를 팀 전체에 배포하거나, 오픈소스 hook 패키지를 설치하는 것이 가능하다.

### matcher 패턴

`matcher`는 hook이 어떤 tool에 대해 실행될지를 결정하는 필터다:

- **정규식 패턴**: `"Edit|Write|MultiEdit"` — 특정 tool 이름에만 매칭
- **미지정(없으면)**: 모든 tool에 대해 실행

matcher가 없는 hook은 **모든 tool 호출에 대해 실행**되므로, 성능과 의도를 고려하여 가능하면 matcher를 지정하는 것이 좋다.

### 환경 변수

Hook command 내에서 사용할 수 있는 환경 변수:

- **`${CLAUDE_PLUGIN_ROOT}`** — 플러그인 루트의 절대 경로. 플러그인 내부의 스크립트를 참조할 때 사용.
- **`${CLAUDE_PROJECT_DIR}`** — 현재 프로젝트 디렉토리. 프로젝트 상대 경로를 절대 경로로 변환할 때 사용.
- **`${CLAUDE_ENV_FILE}`** — 세션 환경 파일 경로. Hook에서 생성한 환경 변수를 세션 전체에 공유할 때 사용.

이 환경 변수들은 hook command 문자열 내에서 직접 사용할 수 있으며, 실행 시점에 실제 값으로 치환된다.

---

## 실전 패턴

### Pattern 1: Security Guardian (security-guidance)

**목적**: PreToolUse hook으로 파일 편집 시 보안 패턴을 검사하여, API key, 비밀번호, 민감 정보가 코드에 포함되는 것을 차단한다.

**구현 핵심**:

```python
# stdin에서 tool_input 읽기
data = json.loads(sys.stdin.read())
file_path = data["tool_input"].get("file_path", "")
content = data["tool_input"].get("content", "")

# 보안 패턴 매칭 (API keys, passwords 등)
if matches_security_pattern(content):
    sys.exit(2)  # Block
```

이 패턴의 핵심 동작:
1. stdin에서 tool input JSON을 읽는다.
2. 파일 내용(content)에 보안 패턴(API key 형태의 문자열, `password = "..."`, AWS 키 패턴 등)이 있는지 regex로 검사한다.
3. 패턴이 감지되면 `exit(2)`로 차단한다.
4. 차단 시 systemMessage로 "이 파일에 민감 정보가 포함되어 있습니다. 환경 변수로 대체하세요"와 같은 안내를 LLM에게 전달한다.

**상태 관리 패턴**: 같은 경고를 반복하지 않기 위해 세션별 상태 파일을 사용한다:
- `~/.claude/security_warnings_state_{session_id}.json`
- 이미 경고한 패턴을 추적하여 중복 경고를 방지한다.
- session_id를 파일명에 포함시켜 세션 간 격리를 보장한다.

### Pattern 2: Infinite Loop Agent (ralph-loop)

**목적**: Stop hook으로 세션 종료를 차단하고, 동일 프롬프트를 재주입하여 반복 작업 에이전트를 구현한다.

**구현 핵심**:

```bash
# YAML frontmatter에서 설정 파싱
MAX_ITER=$(parse_frontmatter "max_iterations")
CURRENT=$(parse_frontmatter "iteration")

# 완료 조건 체크 (<promise> 태그)
if grep -q "<promise>" "$TRANSCRIPT"; then
    exit 0  # Allow stop
fi

# 차단 + 재주입
echo '{"decision":"block","reason":"Continue iteration..."}'
exit 2
```

이 패턴의 동작 흐름:
1. LLM이 작업 완료를 선언하고 Stop 이벤트가 발생한다.
2. Hook이 대화 기록(transcript)을 확인한다.
3. `<promise>` 태그(사전 정의된 완료 마커)가 없으면, 아직 완료되지 않은 것으로 판단한다.
4. `exit 2`로 종료를 차단하고, `reason` 필드의 내용이 다음 프롬프트로 재주입된다.
5. LLM은 새 프롬프트를 받아 작업을 계속한다.

**상태 관리**: Markdown state file(`.claude/ralph-loop.local.md`)에 iteration 상태를 저장한다:
- YAML frontmatter에 `iteration`, `max_iterations`, `status` 등을 기록
- 각 iteration마다 상태 파일을 업데이트
- `<promise>` 태그가 transcript에 나타나면 완료로 간주하고 `exit 0`으로 종료 허용

이 패턴은 "LLM이 스스로 반복을 결정하는 것"이 아니라 "결정론적으로 반복을 강제하는 것"이라는 점에서 Hooks 철학의 정수를 보여준다.

### Pattern 3: Rule Engine (hookify)

**목적**: 4개 event에 걸쳐 동작하는 범용 규칙 엔진. 코드를 작성하지 않고도 Markdown 파일로 hook 규칙을 정의할 수 있다.

**구현 구조**:
- `.claude/hookify.*.local.md` 파일에서 규칙을 로드한다.
- 각 규칙 파일은 YAML frontmatter로 조건을 정의한다.
- 지원하는 Operators: `regex_match`, `contains`, `equals`, `starts_with` 등
- 4개 hook event에 걸쳐 동작: **PreToolUse**, **PostToolUse**, **Stop**, **UserPromptSubmit**

**왜 이 패턴이 강력한가**: 개발자가 아닌 팀원(보안 담당자, 프로젝트 매니저 등)도 Markdown 파일을 편집하는 것만으로 에이전트의 행동 규칙을 정의할 수 있다. 코드 배포 없이 규칙을 변경할 수 있으므로, 빠른 iteration이 가능하다.

---

## LLM 엔지니어링 인사이트

### 1. 확률적 + 결정론적 하이브리드

LLM의 창의성은 유지하면서, 안전/보안/품질은 결정론적으로 보장하는 것이 프로덕션 LLM 앱의 핵심 아키텍처 패턴이다. LLM에게 "위험한 코드를 쓰지 마"라고 프롬프트하는 것은 99%의 신뢰도일 수 있지만, Hook으로 차단하는 것은 100%의 신뢰도다. 프로덕션에서는 그 1%가 사고가 된다.

### 2. 관심사 분리 (Separation of Concerns)

Hook은 **"무엇이 위험한가"만 판단**한다. "이 내용에 API key가 있으니 차단"까지만 하고, "그러면 대신 어떻게 할까"는 LLM이 결정한다. 이 분리가 중요한 이유는, Hook은 결정론적으로 빠르게 판단해야 하고, 대안 탐색은 LLM의 강점이기 때문이다. 두 역할을 섞으면 둘 다 잘 못하게 된다.

### 3. 상태 관리 패턴

세션별 상태 파일로 hook 간 상태를 공유하는 패턴이 반복적으로 나타난다. 예를 들어, Security Guardian은 이미 경고한 패턴을 추적하고, ralph-loop은 현재 iteration 번호를 추적한다. 이 패턴의 핵심은:
- **파일 기반 상태**: 프로세스 간 공유가 용이하고, 크래시 시에도 상태가 보존된다.
- **session_id 기반 격리**: 세션 간 상태가 오염되지 않는다.
- **JSON/YAML 형식**: 사람이 읽고 디버깅할 수 있다.

### 4. Exit Code 설계

0과 2만 사용하는 단순함이 핵심이다. 왜 1이 아닌 2인가? Exit code 1은 일반적으로 "에러"를 의미하며, hook 스크립트 자체의 버그로 인한 크래시와 구분이 어렵다. Exit code 2를 "의도적 차단"으로 지정함으로써, "hook이 고장났다"와 "hook이 의도적으로 차단했다"를 명확히 구분한다. 복잡한 상태 코드 체계보다 JSON stdout으로 세부 정보를 전달하는 것이 확장성과 가독성 모두에서 우월하다.

### 5. systemMessage 패턴

Hook이 LLM의 context에 직접 메시지를 주입할 수 있다는 것은 **강력한 피드백 루프**를 형성한다. 단순히 "차단됨"이 아니라, "이 파일은 보안 정책상 수정할 수 없습니다. 대신 config.yaml을 통해 설정을 변경하세요"와 같은 구체적 안내를 LLM에게 전달할 수 있다. LLM은 이 메시지를 읽고, 차단 사유를 이해하고, 적절한 대안을 찾는다. 이것은 "차단하고 끝"이 아니라 "차단하고 안내하여 더 나은 결과로 유도"하는 패턴이다.

---

## 적용하기: 나만의 Hook 설계 체크리스트

Hook을 설계할 때 다음 질문에 답해보라:

- [ ] **이 검증이 결정론적인가?** (예: regex 매칭, 파일 존재 확인, 화이트리스트 대조) → **Command hook**으로 구현. 빠르고 예측 가능하다.
- [ ] **이 검증에 맥락 이해가 필요한가?** (예: 코드 의도 파악, 변경의 적절성 판단, 자연어 분석) → **Prompt hook**으로 구현. LLM의 이해력을 활용한다.
- [ ] **차단 시 LLM에게 대안을 제시하는가?** → **systemMessage**를 활용하여 구체적인 안내를 주입한다. "차단"만 하면 LLM이 같은 실수를 반복할 수 있다.
- [ ] **상태가 세션 간 유지되어야 하는가?** → **State file 패턴**을 적용한다. session_id 기반 파일로 상태를 관리한다.
- [ ] **timeout이 적절한가?** → Command hook: 10-60초 (네트워크 호출 포함 시 길게). Prompt hook: 30초 이상 (LLM 호출 시간 고려).

---

## 대안 비교

| 접근 방식 | 구현 복잡도 | 결정론적 제어 | 확장성 | Context 피드백 | 적합한 상황 |
|----------|-----------|-------------|--------|--------------|-----------|
| **Hooks (Claude Code 방식)** | 중간 | ✅ exit code 기반 100% | 높음 (plugin 배포 가능) | ✅ systemMessage 주입 | LLM 에이전트의 safety/security guardrail |
| **Middleware (Express 스타일)** | 낮음 | ✅ | 중간 | ❌ LLM context 접근 불가 | 전통적 웹 서버 요청 파이프라인 |
| **Event Emitters (Node.js EventEmitter)** | 낮음 | ❌ 비동기, 순서 보장 약함 | 중간 | ❌ | 느슨한 결합의 알림/로깅 |
| **Plugin System (VSCode 스타일)** | 높음 | 부분적 | 높음 | 부분적 | 복잡한 UI 확장이 필요한 도구 |
| **Prompt Engineering만** | 낮음 | ❌ 확률적 (99% 신뢰) | 낮음 | N/A (프롬프트 자체가 context) | 프로토타입, 비안전 critical 영역 |

**선택 기준**: LLM 에이전트에서 "절대 허용하면 안 되는" 행동을 제어해야 한다면 Hooks가 유일한 선택이다. Middleware는 LLM context에 피드백을 줄 수 없고, Event Emitter는 차단(blocking) 제어가 어렵다. Prompt engineering만으로는 100% 신뢰도를 달성할 수 없다.

---

## 내 프로젝트에 적용하기

### Step 1: Lifecycle 이벤트 식별

프로젝트의 LLM 워크플로우를 분석하여 **결정론적 제어가 필요한 지점**을 식별한다. 일반적으로 다음 5개가 핵심이다:
- Tool 실행 전 (입력 검증) → PreToolUse에 해당
- Tool 실행 후 (결과 검증) → PostToolUse에 해당
- 세션 시작/종료 (초기화/정리) → SessionStart/SessionEnd에 해당
- 작업 완료 판단 (완료 기준 강제) → Stop에 해당

각 지점에서 "LLM에게 맡기면 안 되는 것"을 목록으로 만든다. 예: 민감 정보 노출, 위험한 명령 실행, 미완료 상태에서 종료.

### Step 2: Hook Runner 구현 (JSON stdin/stdout 프로토콜)

Hook runner는 다음 contract를 따르는 단순한 프로세스 실행기다:
1. Hook 스크립트를 spawn하고, **stdin으로 JSON을 전달**한다 (`{ tool_name, tool_input, session_id }`)
2. **exit code를 읽는다**: 0 = 승인, 2 = 차단 (1은 hook 자체 에러로 구분)
3. **stdout JSON을 파싱**한다: `systemMessage` 필드가 있으면 LLM context에 주입

이 프로토콜의 장점은 **언어 무관**이라는 것이다. Python, Bash, Go 등 어떤 언어로든 hook을 구현할 수 있다. 복잡한 SDK나 라이브러리 없이 stdin/stdout/exit code만으로 통신한다.

### Step 3: Security Guardian을 첫 번째 Hook으로 구현

가장 먼저 구현할 hook은 **보안 패턴 검사**다. PreToolUse에 등록하여:
- 파일 쓰기 시 API key, password, secret 패턴을 regex로 검사
- 매칭되면 `exit 2`로 차단하고, `systemMessage`로 "환경 변수를 사용하세요"를 LLM에게 안내
- 세션별 상태 파일(`{session_id}.json`)로 이미 경고한 패턴을 추적하여 중복 경고 방지

이 하나의 hook만으로도 LLM 에이전트의 가장 위험한 행동(민감 정보 하드코딩)을 100% 차단할 수 있다. 이후 필요에 따라 디렉토리 접근 제어, 명령어 화이트리스트 등을 점진적으로 추가한다.
