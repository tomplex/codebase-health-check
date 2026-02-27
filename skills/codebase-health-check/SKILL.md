---
name: health-check
description: >
  Analyze a codebase for structural health issues (dead code, complexity, DRY violations,
  confusing code, extensibility, inconsistencies, naming/organization) and produce a severity-ranked report.
---

# Codebase Health Check — Full Methodology

You are executing a codebase health check. Follow every phase below in strict order.
Do NOT skip phases, reorder them, or begin a later phase before its prerequisites are met.

---

## Phase 0: Determine Scope

**Goal:** Establish which part of the codebase to analyze.

1. Check if the user provided a scope argument with the `/health-check` command (e.g., `/health-check src/api`).
2. If a scope was provided, use it as the target directory or file set.
3. If NO scope was provided, ask the user:
   > What directory or area of the codebase should I analyze? (e.g., `src/`, `lib/`, or `.` for the entire repo)
4. Confirm the scope exists by checking the filesystem. If it does not exist, ask again.
5. If the scope contains more than ~5,000 source files, suggest narrowing to a specific feature directory or module. Very large scopes dilute agent attention and produce noisier results. The user can override this suggestion if they want a full-codebase scan.
6. Store the resolved scope path for use in all subsequent phases.

---

## Phase 1: Gather Codebase Context

**Goal:** Understand the codebase so agents can distinguish intentional patterns from problems.

**HARD GATE: Do NOT proceed to Phase 2 until all three questions have been answered.**

Ask the user the following questions. You may ask them one at a time or all at once, but you MUST have answers to all three before continuing. Use concise, direct phrasing:

1. **Purpose:** "What does this codebase do? (one or two sentences is fine)"
2. **Active areas:** "What areas are you actively extending or working in?"
3. **Pain points:** "Any known pain points or areas you already suspect have issues?"

Store the user's answers verbatim. These will be passed to every analysis agent as context.

---

## Phase 2: Create Task List and Dispatch Agents

**Goal:** Launch 7 parallel analysis agents, one per category.

**HARD GATE: Do NOT begin this phase until Phase 1 is fully complete (all three context answers collected).**

### Step 2a: Create the Task List

Use TaskCreate to create the following tasks. Create all of them before dispatching any agents:

1. **"Gather codebase context"** — Ask 2-3 questions about purpose, active areas, pain points (mark as complete — already done in Phase 1)
2. **"Run dead code analysis"** — Detect unused exports, unreachable code, orphan files
3. **"Run complexity analysis"** — Find deeply nested logic, long functions, high cyclomatic complexity
4. **"Run DRY violation analysis"** — Identify duplicated logic and near-duplicate patterns
5. **"Run confusing code analysis"** — Find misleading names, magic values, implicit coupling
6. **"Run extensibility analysis"** — Detect rigid patterns, hardcoded configs, missing abstractions
7. **"Run inconsistency analysis"** — Find mixed patterns, style drift, conflicting conventions
8. **"Run naming and organization analysis"** — Detect poor file structure, misleading module names, unclear boundaries
9. **"Aggregate findings and write report"** — Combine all agent results, deduplicate, write report
10. **"Present summary and offer brainstorming"** — Show executive summary, offer handoff

Set task 9 as blocked by tasks 2-8. Set task 10 as blocked by task 9.

### Step 2b: Dispatch All 7 Agents in Parallel

Read each agent prompt file from the skill directory, then dispatch all 7 agents in a SINGLE message using multiple Task tool calls. Each agent must be dispatched with `subagent_type: "general-purpose"`.

The 7 agent prompt files are located at `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/`:

- `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/agent-dead-code.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/agent-complexity.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/agent-dry.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/agent-confusing.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/agent-extensibility.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/agent-inconsistent.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/codebase-health-check/agent-naming-org.md`

**Agent dispatch template — every agent receives this prompt structure:**

```
# Codebase Health Check — [CATEGORY NAME] Analysis

## Scope
[The resolved scope path from Phase 0]

## Codebase Context
- **Purpose:** [User's answer to question 1]
- **Active areas:** [User's answer to question 2]
- **Known pain points:** [User's answer to question 3]

## Your Task
[Paste the FULL contents of the agent's prompt file here]

## Output Format
Return your findings using EXACTLY this format for each finding. Do not deviate:

Finding: [short description]
Severity: [Critical | Important | Minor]
Location: [file_path:line_number]
Details: [what is wrong and why it matters]
Suggestion: [high-level recommendation — do NOT provide implementation code]

If you find no issues in your category, return:
No findings in [CATEGORY NAME] category.

## Important Rules
- Only report genuine problems. Do not pad results.
- Do NOT suggest implementations or write fix code. Describe problems and give high-level recommendations only.
- Use absolute file paths in Location fields.
- Include line numbers wherever possible.
```

After dispatching, mark tasks 2-8 as in_progress.

---

## Phase 3: Aggregate Findings

**HARD GATE: Do NOT begin this phase until ALL 7 agents have returned their results. Check that every agent task (tasks 2-8) is complete before proceeding.**

Mark task 9 as in_progress.

### Step 3a: Collect All Findings

Gather the output from all 7 agents. Parse each finding into its structured fields:
- Finding (short description)
- Severity (Critical, Important, or Minor)
- Category (which agent produced it)
- Location (file_path:line_number)
- Details
- Suggestion

### Step 3b: Deduplicate

Multiple agents may flag the same location or the same underlying issue from different angles. Deduplicate as follows:

1. **Exact location match:** If two findings reference the same file and line number, merge them. Keep the higher severity. Combine the details and note both categories.
2. **Same root cause:** If two findings describe the same underlying problem (even at different locations), group them as a single finding with multiple locations. Use the higher severity.
3. When merging, preserve all unique details and suggestions from both findings.

### Step 3c: Sort by Severity

Order all deduplicated findings:
1. **Critical** findings first (actively harmful or blocking)
2. **Important** findings second (tech debt that compounds)
3. **Minor** findings last (nice to clean up)

Within each severity level, group by category.

**Severity definitions for reference:**
- **Critical:** Actively harmful or blocking. Causes bugs, data loss, security issues, or prevents development.
- **Important:** Should fix soon. Tech debt that compounds over time, making the codebase harder to maintain or extend.
- **Minor:** Nice to clean up. Improves readability, consistency, or developer experience but is not urgent.

---

## Phase 4: Write the Report

**HARD GATE: Do NOT write the report until aggregation (Phase 3) is complete.**

### Step 4a: Create the Report Directory

Create a new directory `docs/health-checks/YYYY-MM-DD/` relative to the current working directory (the project root), where YYYY-MM-DD is today's date. If it already exists (e.g., re-running on the same day), append a suffix: `YYYY-MM-DD-2/`, `YYYY-MM-DD-3/`, etc.

### Step 4b: Write the Report File

Write the report to: `<project-root>/docs/health-checks/YYYY-MM-DD/report.md` where `<project-root>` is the current working directory.

Use today's date for `YYYY-MM-DD`.

**Report template — follow this structure exactly:**

```markdown
# Codebase Health Check — YYYY-MM-DD

**Scope:** [target path or "Full codebase"]
**Context:** [brief summary of user's answers: purpose, active areas, known pain points]

## Executive Summary

[2-3 sentences summarizing the overall health of the codebase. Mention the scope analyzed, the total number of findings, and the most significant themes.]

## Finding Counts

| Severity | Count |
|----------|-------|
| Critical | N     |
| Important| N     |
| Minor    | N     |
| **Total**| **N** |

## Findings

### Critical

#### [Finding number]. [Short description]
- **Category:** [analysis category name]
- **Location:** `[file_path:line_number]`
- **Details:** [what is wrong and why it matters]
- **Suggestion:** [high-level recommendation]

[Repeat for each Critical finding]

### Important

[Same format as Critical, continuing the finding numbering]

### Minor

[Same format as Critical, continuing the finding numbering]

## Category Summaries

### Dead Code
[2-3 sentence summary of findings in this category, or "No issues found."]

### Complexity
[2-3 sentence summary]

### DRY Violations
[2-3 sentence summary]

### Confusing Code
[2-3 sentence summary]

### Extensibility
[2-3 sentence summary]

### Inconsistent Patterns
[2-3 sentence summary]

### Naming & Organization
[2-3 sentence summary]
```

Mark task 9 as complete after writing the report.

---

## Phase 5: Present Summary and Offer Resolution

Mark task 10 as in_progress.

### Step 5a: Present the Executive Summary

Show the user the executive summary and finding counts directly in the conversation. Include the path to the full report. Format:

> **Health Check Complete**
>
> [Executive summary text from the report]
>
> | Severity | Count |
> |----------|-------|
> | Critical | N     |
> | Important| N     |
> | Minor    | N     |
>
> Full report written to: `docs/health-checks/YYYY-MM-DD/report.md`

If there are Critical findings, also list their short descriptions.

### Step 5b: Offer Resolution Pipeline

After presenting the summary, ask:

> "Would you like to triage and resolve these findings?"

- If the user says **yes**, invoke `health-check:triage` with the health-check directory path.
- If the user says **no**, mark task 10 as complete and end the health check. The report stands on its own — the user can run `/resolve` later to pick up the pipeline.

Mark task 10 as complete after the user has responded.

---

## Hard Gates Summary

These are non-negotiable ordering constraints. Violating any of them invalidates the health check:

| Gate | Rule |
|------|------|
| G1 | Do NOT dispatch agents (Phase 2) until all three context questions (Phase 1) are answered |
| G2 | Do NOT begin aggregation (Phase 3) until ALL 7 agents have returned results |
| G3 | Do NOT write the report (Phase 4) until aggregation is complete |
| G4 | Do NOT suggest implementations anywhere — findings describe problems with high-level suggestions only |

---

## Error Handling

- **Agent fails or times out:** If an agent does not return results, note the category as "Analysis incomplete — agent did not return results" in the report. Do NOT block other categories.
- **No findings at all:** If all 7 agents return no findings, write a short report confirming a clean bill of health. Still present the summary to the user.
- **Scope does not exist:** Ask the user to provide a valid path. Do not proceed until a valid scope is confirmed.
- **Report directory write fails:** Inform the user and print the report to the conversation instead.
