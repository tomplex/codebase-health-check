---
name: resolve-batch
description: >
  Use when you have a triaged health-check plan and need to execute the
  next batch of fixes. Also use when resuming a health check that has
  unresolved batches remaining.
---

# Resolve Batch — Health-Check Fix Execution

You are executing health-check fixes one batch at a time. Follow every phase below in strict order. Do NOT skip phases, reorder them, or begin a later phase before its prerequisites are met.

---

## Phase 0: Locate and Read State

**Goal:** Find the health-check directory and determine which batch to execute next.

1. Check if a path argument was provided (from triage handoff, the `/resolve` command, or a previous resolve-batch invocation).
2. If no path was provided, find the most recent directory under `docs/health-checks/` by sorting directory names (which are dates in `YYYY-MM-DD` format). Use the last entry in sorted order.
3. Read `plan.md` from the resolved directory. If it does not exist, stop with this error:
   > "No plan.md found in [path]. Run /triage first."
4. Read `progress.md` from the resolved directory. If it does not exist, stop with this error:
   > "No progress.md found in [path]. Run /triage first."
5. Determine the current batch: find the first row in the Batch Status table with status `next`. If no row has status `next`, find the first row with status `pending`.
6. If no unresolved batches remain (all rows are `done`), invoke `codebase-health-check:complete` with the health-check directory path and stop. Do NOT continue with any later phase.
7. Read the current batch's details from `plan.md`: the finding numbers listed under the batch heading, the batch theme (name), and the classification in parentheses (`mechanical` or `architectural`).
8. Read `report.md` from the health-check directory. For each finding number in the current batch, locate its full entry in the report — finding number, short description, severity, location, details, and suggestion. Store all of these for use in later phases.

---

## Phase 1: Pre-Batch Verification

**Goal:** Confirm the codebase is in a passing state before making any changes.

**HARD GATE: Do NOT start implementation until verification passes.**

1. Read the verification command from `plan.md` (the `**Verification command:**` line near the top of the file).
2. Run the verification command.
3. If it **fails**: STOP. Report to the user:
   > "Verification failed before starting Batch N: [batch theme]. The codebase has existing failures that must be resolved first."

   Show the failure output. Do NOT proceed to any later phase.
4. If it **passes**: continue to Phase 2 or Phase 3 based on the batch classification.

---

## Phase 2: Execute — Mechanical Batches

**This phase applies ONLY if the current batch is classified as `mechanical`. If the batch is `architectural`, skip to Phase 3.**

### Step 2a: Dispatch Subagents

Dispatch subagents for the batch using the Task tool. Use one subagent per finding unless findings are tightly related (e.g., "remove dead code from 6 related files" is one job — use judgment, but default to one-per-finding).

Each subagent prompt must include ALL of the following:

1. **The full finding text from report.md** — finding number, short description, severity, location, details, and suggestion. Paste the complete entry, not a summary.
2. **The specific fix expected** — drawn from the finding's suggestion field.
3. **Scope constraint:** "Touch only what this finding describes. Do not refactor, clean up, or improve surrounding code."
4. **Verification instruction:** "After making changes, run: `[verification command from plan.md]`"

Dispatch all subagents for the batch in a single message using multiple Task tool calls.

### Step 2b: Review Subagent Results

1. Wait for all subagents to complete.
2. Review what each subagent changed — read the actual files they modified. Do not rely on their self-reported summaries.
3. Check for conflicts between parallel subagent edits. Two subagents modifying the same file in incompatible ways is a conflict. If conflicts are found, resolve them manually.

### Step 2c: Full Verification

**HARD GATE: Run the verification command yourself. Do NOT trust subagent verification reports.**

1. Run the verification command from `plan.md`.
2. If it **fails**: diagnose the failure. Fix the issue. Re-run verification. Repeat until verification passes.
3. If it **passes**: proceed to Phase 4.

---

## Phase 3: Execute — Architectural Batches

**This phase applies ONLY if the current batch is classified as `architectural`. If the batch is `mechanical`, it was already handled in Phase 2 — skip to Phase 4.**

### Step 3a: Understand the Problem

Read the finding(s) in the current batch and the surrounding code in the codebase. Understand the current structure, the problem described, and the dependencies involved. Read widely enough to see the full picture — do not limit yourself to the exact file and line in the finding.

### Step 3b: Propose Approaches

Propose 2-3 approaches for the refactoring. For each approach, present:
- A brief description of the structural change
- Trade-offs: what improves, what gets more complex, what risks breaking
- Your recommendation and why

Present these to the user.

**HARD GATE: Do NOT implement until the user approves an approach.**

Wait for explicit approval. If the user asks questions, counter-proposes, or is ambiguous, continue the discussion until they clearly approve one approach.

### Step 3c: Implement

Implement the approved approach. For large decompositions, work incrementally:
1. Extract one piece.
2. Run the verification command.
3. If verification passes, extract the next piece.
4. If verification fails, fix before continuing.

This incremental approach prevents large broken states that are hard to diagnose.

### Step 3d: Full Verification

1. Run the verification command from `plan.md`.
2. If it **fails**: diagnose the failure. Fix the issue. Re-run verification. Repeat until verification passes.
3. If it **passes**: proceed to Phase 4.

---

## Phase 4: Commit and Document

**Goal:** Create a single commit for the batch and write the resolution log.

**HARD GATE: Do NOT commit until verification passes. Do NOT update progress.md before the commit succeeds.**

### Step 4a: Commit

Create a single commit with a descriptive message referencing finding numbers:

- **Mechanical batches:** `fix: [theme] (#N, #M, #P)`
- **Architectural batches:** `refactor: [description] (#N)`

One commit per batch. Never combine batches into a single commit.

### Step 4b: Write the Batch Resolution Log

Write `batch-N-[theme-slug].md` to the health-check directory. Use lowercase, hyphenated theme slugs (e.g., `batch-2-import-cleanup.md`).

Use this exact format:

```markdown
# Batch N: [Theme Name]

**Date:** YYYY-MM-DD
**Classification:** [mechanical | architectural]
**Commit:** `[hash]`

## Findings Resolved
- #N: [short description of what changed]
- #M: [short description of what changed]

## Verification
[paste the actual verification command output here — the real output, not "tests passed"]

## Notes
[any surprises, edge cases, or context for future sessions — or "None." if clean]
```

Use today's date. Use the actual short commit hash from Step 4a. The Findings Resolved descriptions should say what *changed*, not what the original problem was. The Verification section must contain the real terminal output from the verification command. The Notes section captures institutional knowledge — if nothing notable happened, write "None."

### Step 4c: Update progress.md

Rewrite `progress.md` with these changes:

1. **Summary counts table:** Increment the Resolved count by the number of findings in this batch. Decrement the Remaining count by the same amount. Won't Fix and Total stay the same.
2. **Batch Status table:** Set the current batch's status to `done`. If a subsequent batch exists with status `pending`, set it to `next`.

---

## Phase 5: Continue or Complete

**Goal:** Hand off to the next step in the pipeline.

1. Read the updated `progress.md`.
2. If unresolved batches remain (any row with status `next` or `pending`) -- invoke `codebase-health-check:resolve-batch` with the health-check directory path. This is the ONLY valid next step. Do NOT end the session or offer the user other options.
3. If all batches are resolved (every row has status `done`) -- invoke `codebase-health-check:complete` with the health-check directory path. This is the ONLY valid next step. Do NOT end the session or offer the user other options.

---

## Hard Gates Summary

These are non-negotiable ordering constraints. Violating any of them invalidates the batch:

| Gate | Rule |
|------|------|
| G1 | Do NOT start implementation without passing verification |
| G2 | Do NOT skip the design step for architectural batches |
| G3 | Do NOT trust subagent verification — run it yourself |
| G4 | Do NOT commit without verification passing |
| G5 | Do NOT proceed to next batch if current verification failed |
| G6 | Do NOT update progress.md before commit succeeds |
| G7 | One commit per batch — never combine batches |

---

## Rationalization Defenses

These are patterns where Claude is tempted to cut corners. The skill does not permit it:

| Claude thinks... | Skill responds... |
|---|---|
| "This architectural fix is simple enough to skip design" | Classification was approved in triage. Architectural gets design. |
| "Tests are slow, I'll just run lint" | Use the verification command from plan.md. The user chose it. |
| "The subagent said it verified" | Do not trust subagent success reports. Run verification yourself. |
| "I'll commit both batches together" | One commit per batch, 1:1 with resolution log. |
| "This finding is basically won't-fix, I'll skip it" | Won't-fix was decided in triage. If it's in a batch, it gets fixed. |
| "I'll also clean up this nearby code while I'm here" | Touch only what the finding describes. No scope creep. |
