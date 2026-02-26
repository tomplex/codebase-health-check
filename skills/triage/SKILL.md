---
name: triage
description: >
  Use when you have a health-check report and need to decide which findings
  to fix, which to skip, and what order to tackle them. Also use when
  resuming a health check that has a report but no resolution plan.
---

# Triage — Health-Check Resolution Planning

You are triaging a codebase health-check report. Follow every phase below in strict order. Do NOT skip phases, reorder them, or begin a later phase before its prerequisites are met.

---

## Phase 0: Locate the Health-Check Directory

**Goal:** Find the health-check directory containing the report to triage.

1. Check if a path argument was provided (from the analysis skill handoff or the `/resolve` command).
2. If no path was provided, find the most recent directory under `docs/health-checks/` by sorting directory names (which are dates in `YYYY-MM-DD` format). Use the last entry in sorted order.
3. Confirm that `report.md` exists inside the resolved directory. If it does not exist, stop with this error:
   > "No report.md found in [path]. Run /health-check first."
4. If `plan.md` already exists in the directory, inform the user:
   > "Triage was already completed for this health check — plan.md exists at [path]/plan.md. Would you like to re-triage (overwrite plan.md) or proceed to resolving batches?"
   - If the user chooses to re-triage, continue to Phase 1 (plan.md will be overwritten in Phase 5).
   - If the user chooses to proceed, invoke `codebase-health-check:resolve-batch` with the health-check directory path and stop.

---

## Phase 1: Read and Classify Findings

**Goal:** Understand every finding in the report and classify it for batching.

**HARD GATE: Do NOT proceed to Phase 2 until report.md is fully read. Every finding must be seen before any classification decisions are made.**

1. Read `report.md` completely. Do not skim — read every finding, including its severity, location, details, and suggestion.

2. For each finding, classify along two axes:

   | Axis | Values | Meaning |
   |------|--------|---------|
   | Complexity | **Mechanical** | Renames, dead code removal, import cleanup, constant extraction, comment additions. No design decisions needed. Safe for subagents. |
   | | **Architectural** | Decomposition, new abstractions, pattern changes, data model migrations. Requires design thinking. Main session work. |
   | Dependency | **Foundation** | Other findings depend on this (e.g., creating a registry that other fixes rely on). Must be done first. |
   | | **Independent** | Can be done in any order. |

3. Identify won't-fix candidates. A finding is a won't-fix candidate when ANY of the following apply:
   - Fix cost clearly exceeds the benefit (e.g., migration required for a non-breaking issue)
   - The "problem" is inherent to the domain, not a real deficiency
   - Extraction would add abstraction without improving testability or clarity

4. Write a one-sentence cost-benefit rationale for each won't-fix candidate. The rationale must explain *why* the fix is not worth doing, not just restate the finding.

---

## Phase 2: Group into Batches

**Goal:** Organize actionable findings into an ordered execution plan.

Apply these ordering rules strictly:

1. **Foundation findings first** — regardless of whether they are mechanical or architectural. These unblock downstream work.
2. **Then mechanical batches, grouped by theme** — all dead code removal together, all renames together, all import cleanup together, etc. Grouping by theme keeps each batch's changes coherent and reduces merge conflicts.
3. **Then architectural findings, each as its own batch** — architectural fixes need individual design discussion before implementation. Do not combine them with mechanical work or with each other (unless they are tightly coupled aspects of the same refactoring).
4. **Batch size constraints:**
   - Mechanical batches: 3-8 findings per batch
   - Architectural batches: 1-2 findings per batch

For each batch:

- Give it a descriptive name that captures the theme (e.g., "Import cleanup", "Dead code removal", "App.tsx decomposition").
- Label its classification in parentheses after the name (e.g., "Batch 1: Element-type registry (foundation, architectural)").
- For foundation batches, note which downstream findings they unblock (e.g., "Unblocks: #8, #9, #10, #14").
- List the finding numbers and short descriptions for each finding in the batch.

---

## Phase 3: Gather Verification Command

**Goal:** Get the user's fast verification command for inclusion in the plan.

Ask the user:

> "What's your fast verification command — something under a minute that covers whether the code you changed is correct? (e.g., `npm test`, `pytest tests/anthology/ -v && ruff check .`, `cargo test -p my-crate`)"

Store their answer exactly as given. This command will be written into `plan.md` and used by resolve-batch after every batch.

---

## Phase 4: Present for Approval

**Goal:** Get explicit user approval before writing any documents.

**HARD GATE: Do NOT write plan.md until the user explicitly approves the batch plan.**

Present the following to the user, clearly formatted:

1. **Won't-fix candidates** — list each candidate with its finding number, short description, and one-sentence rationale. Even if there are zero won't-fix candidates, explicitly say: "No won't-fix candidates identified." Do NOT silently skip this section.

2. **Proposed batch ordering** — show every batch with:
   - Batch number and name with classification label
   - For foundation batches: which findings they unblock
   - The findings in each batch (number and short description)
   - A brief rationale for why foundation batches are ordered first

3. **Verification command** — repeat the command the user provided in Phase 3 for their confirmation.

The user can:
- **Approve as-is** — proceed to Phase 5
- **Move findings between batches** — reassign a finding to a different batch
- **Reject won't-fix candidates** — move them back into a batch for resolution
- **Add new won't-fix candidates** — move a finding from a batch to won't-fix with rationale
- **Reorder batches** — change the execution sequence
- **Adjust batch groupings** — split or merge batches
- **Change the verification command** — provide a different command

Iterate until the user explicitly approves. Do not assume approval from ambiguous responses — ask for confirmation if unclear.

---

## Phase 5: Write Documents

**Goal:** Persist the approved plan and initialize progress tracking.

### Step 5a: Write plan.md

Write `plan.md` to the health-check directory with this exact format:

```markdown
# Resolution Plan

**Verification command:** `[user's command]`
**Date:** YYYY-MM-DD

## Won't Fix

### #N: [finding short description]
[one-sentence rationale]

### #M: [finding short description]
[one-sentence rationale]

[repeat for each won't-fix item, or "None." if no won't-fix items]

## Batches

### Batch 1: [Theme Name] ([classification])
Unblocks: #N, #M, ...  [only include this line for foundation batches]
- #N: [finding short description]
- #M: [finding short description]

### Batch 2: [Theme Name] ([classification])
- #N: [finding short description]

[repeat for all batches]
```

Use today's date for the `YYYY-MM-DD` value.

For the Won't Fix section: if there are no won't-fix items, write "None." on its own line below the `## Won't Fix` heading.

### Step 5b: Write progress.md

Write initial `progress.md` to the health-check directory:

```markdown
# Health Check Progress

| Status | Count |
|--------|-------|
| Resolved | 0 |
| Won't Fix | N |
| Remaining | M |
| **Total** | **T** |

## Batch Status
| Batch | Theme | Type | Status |
|-------|-------|------|--------|
| 1 | [theme] | [mechanical/architectural] | next |
| 2 | [theme] | [mechanical/architectural] | pending |
| 3 | [theme] | [mechanical/architectural] | pending |
[repeat for all batches]
```

Where:
- **N** = number of won't-fix findings
- **M** = number of findings across all batches (total actionable)
- **T** = N + M (total findings in report)
- The first batch gets status `next`. All subsequent batches get status `pending`.
- The Type column contains only `mechanical` or `architectural` (not the dependency axis).

---

## Terminal State

Invoke `codebase-health-check:resolve-batch` with the health-check directory path. This is the ONLY valid next step. Do NOT invoke any other skill. Do NOT end the session. Do NOT offer the user other options.

---

## Hard Gates Summary

These are non-negotiable ordering constraints. Violating any of them invalidates the triage:

| Gate | Rule |
|------|------|
| G1 | Do NOT classify findings until report.md is fully read |
| G2 | Do NOT write plan.md until user approves the batch plan |
| G3 | Do NOT skip won't-fix presentation — even if zero candidates, say so explicitly |
| G4 | Do NOT group architectural findings into mechanical batches |
