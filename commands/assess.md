---
description: Analyze your codebase/work against Claude Code's documented LLM engineering patterns
allowed-tools: Read, Write, Glob, Grep, Bash, Agent, AskUserQuestion
argument-hint: [describe what you're working on or which aspect to assess]
---

# What Would Claude Code Do? — Assess

You are running the `:assess` command of the **what-would-cc-do** plugin. Your goal is to analyze the user's codebase, code patterns, or LLM practices against Claude Code's production-grade engineering patterns documented in this plugin's knowledge base (27 markdown files).

## Core Principles

1. **CC patterns are reference, not gospel.** The user's approach may be better for their context. Always acknowledge this possibility.
2. **Trade-offs always exist.** Every recommendation includes pros AND cons. No blind adoption.
3. **Zero gaps is a valid result.** If the user's approach already aligns well with CC patterns, say so.
4. **Be specific, not generic.** Reference actual CC patterns with file numbers. Point to actual user code.
5. **Maximum 7 gaps per session.** Ordered by impact (highest first).

---

## Phase 1: Context Gathering

**If the user provided arguments**: Use them as the initial context and proceed.

**If no arguments**: Ask the user what to assess using AskUserQuestion:

Options:
- **"Assess my project's overall architecture"** — broad sweep of the current project structure
- **"Assess a specific file or module"** — focused analysis (follow up to ask which file)
- **"Assess my approach to a specific topic"** — e.g., prompt engineering, error handling, tool design, etc.

After scope is determined:

1. Read key project files: look for `CLAUDE.md`, `package.json`, main entry points, config files
2. Read the specific files/modules the user wants assessed
3. Check git status and recent changes if relevant
4. Classify the user's work into assessment categories:
   - Architecture / Modularity
   - Prompt Engineering
   - Tool / API Design
   - Error Handling / Resilience
   - Context Management
   - Security
   - Multi-Agent / Orchestration
   - Testing / Evaluation
   - State / Configuration
   - UI / UX
   - Hooks / Plugins / Extensibility
   - Performance

---

## Phase 2: KB Retrieval

Delegate to the **kb-reader** agent with:
- The identified assessment category (or categories)
- Keywords extracted from the user's code and context
- Brief context description of what the user is building

The kb-reader will return a structured summary of relevant CC patterns. If coverage is insufficient for some categories, note the gap.

---

## Phase 3: Gap Analysis

For each relevant CC pattern returned by kb-reader:

1. **Applicability check**: Does this pattern actually apply to the user's situation? Consider their scale, use case, constraints. If not applicable, move to "Not Applicable" section.
2. **Current state assessment**: What is the user currently doing in this area? Read their code to determine this.
3. **Gap identification**: What specifically differs between the user's approach and CC's approach?
4. **Classification**:
   - `already-aligned` — user follows this pattern (or equivalent)
   - `partially-aligned` — some aspects match, others don't
   - `gap` — significant divergence
   - `user-arguably-better` — user's approach has advantages CC's doesn't

---

## Phase 4: Structured Output

Present findings in this format:

```markdown
## What Would Claude Code Do? — Assessment Results

### Context
[1-2 sentences: what was assessed, which categories]

### Alignment Summary
[X of Y applicable patterns aligned | N gaps found | M not applicable]

---

### Strengths (Already Aligned with CC)
- **[Pattern]** (CC ref: claude_code_XX) — [brief note on alignment]

---

### Gap 1: [Pattern Name]
**Category**: [category]
**CC Reference**: claude_code_XX — [section name]

**What Claude Code Does**:
[Specific description of CC's approach]

**What You Currently Do**:
[Specific description from the user's code]

**The Gap**:
[Clear, specific difference]

**Pros of Adopting CC's Approach**:
- [benefit 1]
- [benefit 2]

**Cons / Trade-offs**:
- [cost 1] → **Mitigation**: [workaround if any]
- [cost 2] → **Mitigation**: [workaround if any]

**Effort**: trivial | moderate | significant
**Impact**: high | medium | low

---

### Gap 2: ...
(repeat for each gap, max 7, ordered by impact)

---

### Not Applicable
[CC patterns considered but not relevant to this user's situation, with brief reasoning]
```

---

## Phase 4.5: Persist Results

After presenting the assessment output, save it as a JSON file so that `:claudecodefy` can reliably access the results even if the conversation is compressed.

1. Create the directory `.what-would-cc-do/` in the user's project root if it doesn't exist
2. Write `.what-would-cc-do/assess-results.json` with this schema:

```json
{
  "timestamp": "<ISO-8601>",
  "context": "<1-2 sentence summary of what was assessed>",
  "strengths": [
    { "pattern": "<name>", "cc_reference": "claude_code_XX", "note": "<brief alignment note>" }
  ],
  "gaps": [
    {
      "id": 1,
      "title": "<gap title>",
      "category": "<category>",
      "cc_reference": "claude_code_XX",
      "cc_section": "<section name>",
      "cc_approach": "<what CC does>",
      "current_approach": "<what user does>",
      "the_gap": "<specific difference>",
      "pros": ["<pro 1>", "<pro 2>"],
      "cons": [{ "cost": "<cost>", "mitigation": "<mitigation>" }],
      "effort": "trivial|moderate|significant",
      "impact": "high|medium|low",
      "selected": false
    }
  ],
  "not_applicable": [
    { "pattern": "<name>", "reason": "<why not applicable>" }
  ]
}
```

3. Inform the user: "Assessment saved to `.what-would-cc-do/assess-results.json`. Consider adding `.what-would-cc-do/` to your `.gitignore`."

---

## Phase 5: Prioritization & Questions

After presenting the assessment, present all gaps as a checkbox list so the user can see and select individual items:

```markdown
### Select Gaps to Pursue

- [ ] **Gap 1**: [Title] — Effort: [effort] | Impact: [impact]
- [ ] **Gap 2**: [Title] — Effort: [effort] | Impact: [impact]
- [ ] **Gap 3**: [Title] — Effort: [effort] | Impact: [impact]
...
```

Then ask the user which to pursue using AskUserQuestion:

Options:
- **"All gaps"** — select all checkboxes
- **"High-impact only"** — select only gaps marked as high impact
- **"Specific gaps: [numbers]"** — e.g., "Specific gaps: 1, 3, 5"
- **"None — just wanted the assessment"** — informational only

For each selected gap, if the gap requires design decisions the user should make (e.g., "which retry strategy to use", "how many agents to spawn"), ask a follow-up question with concrete options. Each question should have 2-4 predefined options reflecting different valid approaches, plus the ability for custom input.

After the user makes their selection, update `.what-would-cc-do/assess-results.json`:
- Set `selected: true` for each chosen gap
- Re-write the file with the updated selections

The user's selections will be used by the `:claudecodefy` command to apply changes.

---

## Output Rules

- Always cite CC reference files by number (e.g., "claude_code_07")
- Quote specific sections from KB, not vague references
- When the user's approach is arguably better than CC's, say so explicitly
- Keep each gap description concise but specific enough to act on
- If the kb-reader returned KB gaps (topics not covered), mention them transparently
