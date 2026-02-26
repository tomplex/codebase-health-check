# Health Check Resolution Pipeline — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add triage, resolve-batch, and complete skills to the codebase-health-check plugin, plus a /resolve command, and update the existing analysis skill to output a directory and hard-pipeline into triage.

**Architecture:** Three new SKILL.md files in new skill directories under `skills/`. One new command file. Modifications to the existing SKILL.md (Phase 4 directory output + Phase 5 pipeline handoff). See `docs/plans/2026-02-26-resolution-pipeline-design.md` for full design.

**Tech Stack:** Markdown skill files for Claude Code plugin system. No code — all deliverables are structured markdown documents.

**Plugin root:** `/home/tom/.claude/plugins/local/codebase-health-check/`

---

### Task 1: Create the triage skill

**Files:**
- Create: `skills/triage/SKILL.md`

**Step 1: Create the skill directory**

```bash
mkdir -p skills/triage
```

**Step 2: Write the triage skill**

Create `skills/triage/SKILL.md` with the following content. This is the complete skill — write it exactly as specified.

Frontmatter:
```yaml
---
name: triage
description: >
  Use when you have a health-check report and need to decide which findings
  to fix, which to skip, and what order to tackle them. Also use when
  resuming a health check that has a report but no resolution plan.
---
```

Body structure (write the full skill following this outline):

1. **Opening line:** "You are triaging a codebase health-check report. Follow every phase below in strict order."

2. **Phase 0: Locate the health-check directory.**
   - Check if a path argument was provided (from the analysis skill handoff or the /resolve command).
   - If no path, find the most recent directory under `docs/health-checks/`.
   - Confirm `report.md` exists in it. If not, error.
   - If `plan.md` already exists, inform user that triage was already done and ask whether to re-triage or proceed to resolve-batch.

3. **Phase 1: Read and classify findings.**
   - HARD GATE: Do NOT proceed until report.md is fully read.
   - Read `report.md`. For each finding, classify along two axes:
     - **Complexity:** Mechanical (renames, dead code, imports, constants — no design decisions, safe for subagents) or Architectural (decomposition, new abstractions, pattern changes — requires design, main session).
     - **Dependency:** Foundation (other findings depend on this, must be done first) or Independent (any order).
   - Identify won't-fix candidates: fix cost exceeds benefit, problem is inherent to domain, or extraction adds abstraction without improving testability or clarity. Write a one-sentence cost-benefit rationale for each.

4. **Phase 2: Group into batches.**
   - Ordering rules (strict):
     1. Foundation findings first (regardless of mechanical/architectural)
     2. Mechanical batches grouped by theme (all dead code together, all renames together)
     3. Architectural findings each as their own batch
     4. Batch size: mechanical 3-8 findings, architectural 1-2 findings
   - For foundation batches, note which downstream findings they unblock.

5. **Phase 3: Gather verification command.**
   - Ask the user: "What's your fast verification command — something under a minute that covers whether the code you changed is correct? (e.g., `npm test`, `pytest tests/anthology/ -v && ruff check .`, `cargo test -p my-crate`)"

6. **Phase 4: Present for approval.**
   - Show the user:
     - Won't-fix candidates with rationale for each (even if zero, say "No won't-fix candidates identified")
     - Proposed batch ordering with classification labels and rationale for foundations first
     - The verification command they provided
   - User can: approve, move findings between batches, reject won't-fix candidates, reorder batches.
   - HARD GATE: Do NOT write plan.md until user approves.

7. **Phase 5: Write documents.**
   - Write `plan.md` to the health-check directory. Format per design doc:
     - Verification command at top
     - Date
     - Won't Fix section with rationale per finding
     - Batches section with ordered batches, each showing: name, classification in parentheses, finding numbers with short descriptions
     - Foundation batches include "Unblocks: #N, #M, ..."
   - Write initial `progress.md`:
     - Summary counts table (Resolved: 0, Won't Fix: N, Remaining: M, Total: T)
     - Batch Status table with first batch marked `next`, rest marked `pending`

8. **Terminal state:** Invoke `codebase-health-check:resolve-batch` with the health-check directory path.

9. **Hard gates summary table:**

| Gate | Rule |
|------|------|
| G1 | Do NOT classify findings until report.md is fully read |
| G2 | Do NOT write plan.md until user approves the batch plan |
| G3 | Do NOT skip won't-fix presentation |
| G4 | Do NOT group architectural findings into mechanical batches |

**Step 3: Verify the file was written**

```bash
cat skills/triage/SKILL.md | head -5
```
Expected: YAML frontmatter starting with `---` and `name: triage`

**Step 4: Commit**

```bash
git add skills/triage/SKILL.md
git commit -m "feat: add triage skill for health-check resolution pipeline"
```

---

### Task 2: Create the resolve-batch skill

**Files:**
- Create: `skills/resolve-batch/SKILL.md`

**Step 1: Create the skill directory**

```bash
mkdir -p skills/resolve-batch
```

**Step 2: Write the resolve-batch skill**

Create `skills/resolve-batch/SKILL.md` with the following content.

Frontmatter:
```yaml
---
name: resolve-batch
description: >
  Use when you have a triaged health-check plan and need to execute the
  next batch of fixes. Also use when resuming a health check that has
  unresolved batches remaining.
---
```

Body structure (write the full skill following this outline):

1. **Opening line:** "You are executing health-check fixes one batch at a time. Follow every phase below in strict order."

2. **Phase 0: Locate and read state.**
   - Check if a path argument was provided.
   - If no path, find the most recent directory under `docs/health-checks/`.
   - Read `plan.md` and `progress.md`.
   - Determine current batch: first row in batch status table with status `next`. If none, first `pending` row (and set it to `next`).
   - If no unresolved batches remain, invoke `codebase-health-check:complete` instead.
   - Read the batch details from plan.md (finding numbers, theme, classification).
   - Read the full finding details from report.md for each finding in this batch.

3. **Phase 1: Pre-batch verification.**
   - HARD GATE: Do NOT start implementation until verification passes.
   - Read the verification command from plan.md.
   - Run it. If it fails, STOP. Report to user: "Verification failed before starting batch N. The codebase has existing failures that must be resolved first." Do NOT proceed.

4. **Phase 2: Execute — Mechanical batches.**
   - Only applies if the current batch is classified as `mechanical`.
   - Dispatch subagents for the batch. One subagent per finding unless findings are tightly related (e.g., "remove dead code from 6 files" is one job). Each subagent prompt includes:
     - The full finding text from report.md (description, location, details, suggestion)
     - The specific fix expected
     - Constraint: "Touch only what this finding describes. Do not refactor surrounding code."
     - Requirement: "Run the following verification command before reporting back: [command from plan.md]"
   - Review subagent results. Read what each changed. Check for conflicts between parallel edits.
   - Run the verification command yourself. HARD GATE: Do NOT trust subagent verification reports. Run it yourself.
   - If verification fails, diagnose and fix before proceeding.

5. **Phase 3: Execute — Architectural batches.**
   - Only applies if the current batch is classified as `architectural`.
   - Read the finding and the surrounding code in the main session.
   - Propose 2-3 approaches for the refactoring. Present to user with trade-offs and your recommendation. Get approval before implementing.
   - HARD GATE: Do NOT implement until user approves approach.
   - Implement the approved approach. For large decompositions, work incrementally — extract one piece, verify, extract the next.
   - Run the verification command. If it fails, diagnose and fix.

6. **Phase 4: Commit and document.**
   - HARD GATE: Do NOT commit until verification passes.
   - Commit with message referencing finding numbers: `fix: [theme] (#N, #M)` for mechanical, `refactor: [description] (#N)` for architectural.
   - Write `batch-N-theme.md` to the health-check directory with:
     - Date, classification, commit hash
     - Findings Resolved: list with finding number and brief description of what changed
     - Verification: the actual command output (captured evidence, not just "passed")
     - Notes: any surprises or edge cases encountered (empty if none)
   - Rewrite `progress.md`:
     - Update summary counts (increment Resolved, decrement Remaining)
     - Set current batch status to `done`
     - Set next batch status to `next` (if one exists)

7. **Phase 5: Continue or complete.**
   - If unresolved batches remain in progress.md → re-invoke `codebase-health-check:resolve-batch`.
   - If all batches resolved → invoke `codebase-health-check:complete`.

8. **Hard gates summary:**

| Gate | Rule |
|------|------|
| G1 | Do NOT start implementation without passing verification |
| G2 | Do NOT skip design step for architectural batches |
| G3 | Do NOT trust subagent verification — run it yourself |
| G4 | Do NOT commit without verification passing |
| G5 | Do NOT proceed to next batch if current verification failed |
| G6 | Do NOT update progress.md before commit succeeds |
| G7 | One commit per batch — never combine batches |

9. **Rationalization defenses:**

| Claude thinks... | Skill responds... |
|---|---|
| "This architectural fix is simple enough to skip design" | Classification was approved in triage. Architectural gets design. |
| "Tests are slow, I'll just run lint" | Use the verification command from plan.md. The user chose it. |
| "The subagent said it verified" | Do not trust subagent success reports. Run verification yourself. |
| "I'll commit both batches together" | One commit per batch, 1:1 with resolution log. |
| "This finding is basically won't-fix, I'll skip it" | Won't-fix was decided in triage. If it's in a batch, it gets fixed. |
| "I'll also clean up this nearby code while I'm here" | Touch only what the finding describes. No scope creep. |

**Step 3: Verify the file was written**

```bash
cat skills/resolve-batch/SKILL.md | head -5
```
Expected: YAML frontmatter starting with `---` and `name: resolve-batch`

**Step 4: Commit**

```bash
git add skills/resolve-batch/SKILL.md
git commit -m "feat: add resolve-batch skill for health-check resolution pipeline"
```

---

### Task 3: Create the complete skill

**Files:**
- Create: `skills/complete/SKILL.md`

**Step 1: Create the skill directory**

```bash
mkdir -p skills/complete
```

**Step 2: Write the complete skill**

Create `skills/complete/SKILL.md` with the following content.

Frontmatter:
```yaml
---
name: complete
description: >
  Use when all batches in a health-check resolution plan are resolved
  and you need to finalize the health check.
---
```

Body structure (write the full skill following this outline):

1. **Opening line:** "You are finalizing a completed health-check resolution. Follow every step below."

2. **Phase 0: Locate and verify.**
   - Check if a path argument was provided. If not, find most recent directory under `docs/health-checks/`.
   - Read `plan.md` and `progress.md`.
   - If any batches are still unresolved: Do NOT silently complete. Present what's still open. Ask the user: "N batches remain unresolved. Close out anyway, or continue resolving?" If continue → invoke `codebase-health-check:resolve-batch`. If close out → proceed, marking remaining as deferred in the final progress.

3. **Phase 1: Final progress rewrite.**
   - Rewrite `progress.md` with:
     - Final summary counts
     - All batch statuses set to `done` (or `deferred` for skipped batches)
     - Completion date

4. **Phase 2: Present summary.**
   - Show the user in conversation:
     - Final counts (resolved / won't fix / deferred / total)
     - Won't-fix items with rationale (from plan.md)
     - List of commits produced (from batch files)
     - Path to the health-check directory

5. **Phase 3: Commit.**
   - `docs: complete health check — N resolved, M won't fix`

6. **Terminal state:** None. Health check is done.

7. **Hard gates:**

| Gate | Rule |
|------|------|
| G1 | Do NOT complete silently if batches remain — present and ask |

**Step 3: Verify the file was written**

```bash
cat skills/complete/SKILL.md | head -5
```
Expected: YAML frontmatter with `name: complete`

**Step 4: Commit**

```bash
git add skills/complete/SKILL.md
git commit -m "feat: add complete skill for health-check resolution pipeline"
```

---

### Task 4: Create the /resolve command

**Files:**
- Create: `commands/resolve.md`

**Step 1: Write the command file**

Create `commands/resolve.md` with the following content:

```markdown
---
description: "Resume resolving findings from an existing health-check report."
disable-model-invocation: true
---

Find the most recent health-check directory under docs/health-checks/.

The user's arguments are: ${ARGUMENTS}
If arguments were provided, use them as the path to the health-check directory.

Read the directory contents to determine state:
- If no plan.md exists → invoke codebase-health-check:triage with the directory path
- If plan.md exists and unresolved batches remain in progress.md → invoke codebase-health-check:resolve-batch with the directory path
- If all batches are resolved → invoke codebase-health-check:complete with the directory path
```

**Step 2: Verify**

```bash
cat commands/resolve.md | head -3
```
Expected: YAML frontmatter with description about resuming.

**Step 3: Commit**

```bash
git add commands/resolve.md
git commit -m "feat: add /resolve command for resuming health-check resolution"
```

---

### Task 5: Update the existing analysis skill

**Files:**
- Modify: `skills/codebase-health-check/SKILL.md:160-297` (Phase 4 and Phase 5)

**Step 1: Read the current file**

Read `skills/codebase-health-check/SKILL.md` fully to confirm exact content before editing.

**Step 2: Update Phase 4 — directory output**

Find the Phase 4 report path instruction (around line 170):

Old:
```markdown
Ensure the directory `docs/health-checks/` exists relative to the current working directory (the project root). Create it if it does not exist.
```

New:
```markdown
Create a new directory `docs/health-checks/YYYY-MM-DD/` relative to the current working directory (the project root), where YYYY-MM-DD is today's date. If it already exists (e.g., re-running on the same day), append a suffix: `YYYY-MM-DD-2/`, `YYYY-MM-DD-3/`, etc.
```

Find the report file path instruction (around line 170-172):

Old:
```markdown
Write the report to: `<project-root>/docs/health-checks/YYYY-MM-DD-health-check.md` where `<project-root>` is the current working directory.
```

New:
```markdown
Write the report to: `<project-root>/docs/health-checks/YYYY-MM-DD/report.md` where `<project-root>` is the current working directory.
```

**Step 3: Update Phase 5 — hard pipeline to triage**

Replace the entire Phase 5 section (from `## Phase 5:` through the end of step 5b) with:

```markdown
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

- If the user says **yes**, invoke `codebase-health-check:triage` with the health-check directory path.
- If the user says **no**, mark task 10 as complete and end the health check. The report stands on its own — the user can run `/resolve` later to pick up the pipeline.

Mark task 10 as complete after the user has responded.
```

**Step 4: Verify the edits**

Read the modified file and confirm:
- Phase 4 references `docs/health-checks/YYYY-MM-DD/report.md`
- Phase 5 references `codebase-health-check:triage` (not `superpowers:brainstorming`)
- No other phases were accidentally modified

**Step 5: Commit**

```bash
git add skills/codebase-health-check/SKILL.md
git commit -m "feat: update analysis skill to output directory and pipeline to triage"
```

---

### Task 6: Update plugin.json

**Files:**
- Modify: `.claude-plugin/plugin.json`

**Step 1: Update the description and version**

The plugin now does more than analyze — it also resolves. Update:

```json
{
  "name": "codebase-health-check",
  "description": "Analyze codebases for structural health issues and guide resolution: dead code, complexity, DRY violations, confusing implementations, extensibility concerns, inconsistent patterns, and naming/organization problems",
  "version": "0.2.0",
  "author": {
    "name": "Tom"
  },
  "license": "MIT",
  "keywords": ["health-check", "code-quality", "analysis", "refactoring", "resolution", "triage"]
}
```

Changes: description adds "and guide resolution", version bumps to 0.2.0, keywords adds "resolution" and "triage".

**Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: bump version to 0.2.0, update description for resolution pipeline"
```

---

### Task 7: Verify complete plugin structure

**Step 1: Verify directory structure**

```bash
find . -type f -not -path './.git/*' | sort
```

Expected output should include:
```
./.claude-plugin/plugin.json
./commands/health-check.md
./commands/resolve.md
./docs/plans/2026-02-26-resolution-pipeline-design.md
./docs/plans/2026-02-26-resolution-pipeline-plan.md
./skills/codebase-health-check/SKILL.md
./skills/codebase-health-check/agent-complexity.md
./skills/codebase-health-check/agent-confusing.md
./skills/codebase-health-check/agent-dead-code.md
./skills/codebase-health-check/agent-dry.md
./skills/codebase-health-check/agent-extensibility.md
./skills/codebase-health-check/agent-inconsistent.md
./skills/codebase-health-check/agent-naming-org.md
./skills/complete/SKILL.md
./skills/resolve-batch/SKILL.md
./skills/triage/SKILL.md
```

**Step 2: Verify all skill frontmatter is parseable**

```bash
head -3 skills/triage/SKILL.md
head -3 skills/resolve-batch/SKILL.md
head -3 skills/complete/SKILL.md
```

Each should start with `---` followed by `name:` on the next line.

**Step 3: Verify the analysis skill still has intact Phases 0-3**

Read `skills/codebase-health-check/SKILL.md` and confirm:
- Phase 0 (scope) is unchanged
- Phase 1 (context) is unchanged
- Phase 2 (dispatch agents) is unchanged
- Phase 3 (aggregate) is unchanged
- Phase 4 references directory output
- Phase 5 references triage pipeline

**Step 4: Commit (if any fixes were needed)**

Only commit if corrections were made in steps 1-3.

```bash
git add -A
git commit -m "fix: correct any issues found during verification"
```
