---
description: Apply Claude Code engineering patterns to your code based on assess results
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, AskUserQuestion
argument-hint: [optional: specific gap number to start from]
---

# What Would Claude Code Do? — Claudecodefy

You are running the `:claudecodefy` command of the **what-would-cc-do** plugin. Your goal is to apply the improvements identified by the `:assess` command to the user's actual code and practices.

## Prerequisites

An `:assess` session must have been completed in this conversation with at least one gap selected for implementation. If no assess results are found in the conversation history, instruct the user to run `/what-would-cc-do:assess` first.

---

## Step 1: Locate Assess Results

**Primary**: Read `.what-would-cc-do/assess-results.json` from the project root.

1. Check if `.what-would-cc-do/assess-results.json` exists (use Glob or Read)
2. If the file exists:
   - Parse the JSON and extract all gaps where `selected: true`
   - Use the structured data directly — no need to search conversation history
3. If the file does NOT exist (fallback for backwards compatibility):
   - Find the assess output in the conversation history
   - Identify which gaps the user selected for implementation
4. List all selected improvements with their status:

```markdown
## Claudecodefy — Implementation Plan

### Selected Gaps to Apply
| # | Gap | Category | Effort | Impact | Status |
|---|-----|----------|--------|--------|--------|
| 1 | [name] | [cat] | [effort] | [impact] | pending |
| 3 | [name] | [cat] | [effort] | [impact] | pending |
```

If arguments were provided (e.g., a specific gap number), filter to only those gaps.

---

## Step 2: Generate Mini-Plans

For each selected gap, generate a concrete implementation plan:

```markdown
### Gap [N]: [Name]

**Target files**:
- `path/to/file.ts` — [what changes]
- `path/to/other.ts` — [what changes]

**Changes**:
1. [Specific change description]
2. [Specific change description]

**CC Pattern Applied**: [Brief description of the CC pattern being adopted]

**Verification**: [How to verify this change works — test command, manual check, etc.]
```

Present all mini-plans to the user and ask for confirmation before executing:

> "I'll apply these changes. Proceed with all, or would you like to modify any?"

Options via AskUserQuestion:
- **"Proceed with all"** — apply all planned changes
- **"Skip some"** — user specifies which to skip
- **"Modify a plan"** — user provides feedback on a specific gap's plan
- **"Cancel"** — abort without changes

---

## Step 3: Execute Changes

For each confirmed change:

1. **Announce**: Output `### Applying Gap [N]: [Name]`
2. **Read**: Read all target files to understand current state
3. **Implement**: Apply changes using Edit (preferred) or Write tools
   - Make minimal, surgical changes
   - Follow the CC pattern as documented in the KB
   - Preserve existing code style and conventions
4. **Verify**: Run verification step if applicable (tests, lint, build)
5. **Report**: Status update after each gap

If a change reveals unexpected issues or conflicts:
- **Stop** and report the issue
- Ask the user how to proceed before continuing
- Do NOT silently work around problems

---

## Step 4: Summary

After all changes are applied:

```markdown
## Claudecodefy — Summary

### Changes Applied
| # | Gap | Files Modified | Status |
|---|-----|---------------|--------|
| 1 | [name] | `file1.ts`, `file2.ts` | done |
| 3 | [name] | `config.ts` | done |

### Verification Results
- [What was checked and the result]

### CC Patterns Now Aligned
- [List of CC patterns the user's code now follows]

### Remaining / Manual Follow-up
- [Anything that couldn't be automated or needs user attention]
```

After presenting the summary, clean up the assess results file:
- Delete `.what-would-cc-do/assess-results.json`
- If `.what-would-cc-do/` is now empty, remove the directory
- If the file didn't exist (fallback mode was used), skip cleanup

---

## Rules

- **Follow assessed gaps exactly.** Do not add improvements beyond what was assessed and selected.
- **Minimal, surgical changes.** Change only what's needed to adopt the CC pattern.
- **Never skip user confirmation.** Always present the plan before executing.
- **Preserve user's code style.** Adopt the CC *pattern*, not CC's coding style.
- **Stop on unexpected issues.** Report and ask, don't push through.
- **No cascading changes.** If applying one gap suggests another improvement, note it but don't act on it. The user can run `:assess` again later.
