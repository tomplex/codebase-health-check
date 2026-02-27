# Health Check Resolution Pipeline — Design

## Goal

Extend the codebase-health-check plugin with a resolution pipeline that guides Claude through triaging, batching, and fixing health-check findings — encoding the patterns that proved effective across real sessions (Splash and FDY projects) and drawing on the structural techniques from the superpowers plugin.

## Architecture

Four skills in one plugin, connected by hard pipelines:

```
codebase-health-check:codebase-health-check  (existing — analysis + report)
    ↓ terminal state
codebase-health-check:triage                  (new — classify, batch, won't-fix)
    ↓ terminal state
codebase-health-check:resolve-batch           (new — execute one batch, verify, log)
    ↓ loops back to self until done, then →
codebase-health-check:complete                (new — final summary, close out)
```

A `/resolve` command provides resume entry — reads document state, invokes the right skill.

Each skill can be invoked independently on an existing health-check directory.

## Key Design Decisions

**Why separate skills, not phases in one file:** Mirrors the superpowers pattern (`brainstorming` → `writing-plans` → `executing-plans`). Each skill is focused, independently invocable, and has its own hard gates. The plugin is the install unit; skills are the workflow units.

**Why a directory, not a single file:** The analysis report is never modified. Batch results are appended as new files (Write, not Edit — simpler and can't corrupt existing content). Claude determines state via `ls` + reading `progress.md`, not parsing sections from a growing file. Batch files can hold rich detail (verification output, notes) without bloating the report.

**Why skill-decides-user-approves:** The effective sessions showed that Claude correctly identified foundation-first ordering, theme-based batching, and conservative won't-fix candidates — but only after trial and error. The skill encodes those patterns as defaults that the user approves or adjusts.

**Why subagents for mechanical, main session for architectural:** Mechanical fixes (renames, dead code, imports) are independent and safe for parallel subagent execution. Architectural fixes (decompositions, new abstractions) need design discussion before implementation — that requires the main session's conversational context.

**Why user-specified verification command:** Test suites range from seconds (Splash, 172 tests) to hours (FDY full suite). The skill asks once during triage and stores the answer. No tiered system to reason about.

---

## Document Structure

Health checks produce a directory:

```
docs/health-checks/YYYY-MM-DD/
├── report.md           # original findings (written by analysis, never modified)
├── plan.md             # triage output: batches, won't-fix, verification command
├── progress.md         # live summary table (rewritten after each batch)
├── batch-1-theme.md    # batch 1 resolution details
├── batch-2-theme.md    # batch 2 resolution details
└── ...
```

### report.md

Identical to current output. No format changes. Written once by the analysis skill, never modified afterward. Findings are numbered sequentially (#1, #2, ...) — all other documents reference findings by number.

### plan.md

Written by triage after user approval.

```markdown
# Resolution Plan

**Verification command:** `pytest tests/anthology/ -v && ruff check .`
**Date:** 2026-02-25

## Won't Fix

### #35: usePinchZoom complexity
Linear state machine, not genuine complexity. Extraction adds 8+ parameters
for no testability gain.

### #47: Eraser sentinel color
Requires data model migration for non-breaking issue. Cost exceeds benefit
until drawing tools are reworked.

## Batches

### Batch 1: Element-type registry (foundation, architectural)
Unblocks: #8, #9, #10, #14, #17, #22
- #1: Scattered element-type metadata consolidated into registry

### Batch 2: Import cleanup (mechanical)
- #24: Mixed import styles in worker
- #26: Remove from __future__ import annotations
- #29: Rename resource parameter to feature_store

### Batch 3: Dead code removal (mechanical)
- #4, #11, #16: Unused code and duplicate filtering

### Batch 4: App.tsx decomposition (architectural)
- #20: 261-line monolith → focused hooks
```

Properties:
- Verification command at top — resolve-batch reads it from here
- Won't-fix before batches (settled, not part of execution)
- Each batch has: name, classification in parentheses, finding numbers
- Foundation batches note what they unblock
- Ordering is final — resolve-batch executes top to bottom

### progress.md

Written by triage, rewritten by resolve-batch after each batch, finalized by complete.

```markdown
# Health Check Progress

| Status | Count |
|--------|-------|
| Resolved | 12 |
| Won't Fix | 2 |
| Remaining | 24 |
| **Total** | **38** |

## Batch Status
| Batch | Theme | Type | Status |
|-------|-------|------|--------|
| 1 | Element-type registry | architectural | done |
| 2 | Import cleanup | mechanical | done |
| 3 | Dead code removal | mechanical | next |
| 4 | App.tsx decomposition | architectural | pending |
```

The batch status table drives resumability: resolve-batch looks for the first row with status `next`, or the first `pending` if no `next` exists.

### batch-N-theme.md

Written by resolve-batch after each batch completes.

```markdown
# Batch 2: Import Cleanup

**Date:** 2026-02-25
**Classification:** mechanical
**Commit:** `abc1234`

## Findings Resolved
- #24: Mixed import styles → all imports now from submodules directly
- #26: Removed from __future__ import annotations from 8 files
- #29: Renamed resource → feature_store, verified across 14 workers

## Verification
$ pytest tests/anthology/ -v
146 passed in 4.2s
$ ruff check .
All checks passed

## Notes
FeatureStoreBuildContext needed quoted self-reference after __future__ removal.
```

Properties:
- Commit hash ties back to git history
- Verification output is captured as evidence, not just "tests passed"
- Notes section captures surprises and edge cases — institutional knowledge for the next session

---

## Skill Designs

### codebase-health-check:triage

**Description (for Claude search optimization):**
```
Use when you have a health-check report and need to decide which findings
to fix, which to skip, and what order to tackle them. Also use when
resuming a health check that has a report but no resolution plan.
```

**Entry:** Invoked by the analysis skill's terminal state, or independently on an existing health-check directory.

**Triage algorithm:**

Step 1 — Classify each finding along two axes:

| Axis | Values | Meaning |
|------|--------|---------|
| Complexity | **Mechanical** | Renames, dead code, imports, constants. No design decisions. Safe for subagents. |
| | **Architectural** | Decomposition, new abstractions, pattern changes. Requires design thinking. Main session. |
| Dependency | **Foundation** | Other findings depend on this. Must be done first. |
| | **Independent** | Can be done in any order. |

Step 2 — Identify won't-fix candidates. A finding is a won't-fix candidate when:
- Fix cost clearly exceeds benefit (migration for non-breaking issue)
- The "problem" is inherent to the domain, not a real deficiency
- Extraction would add abstraction without improving testability or clarity

Each candidate gets a one-sentence cost-benefit rationale.

Step 3 — Group into batches. Ordering rules:
1. Foundation findings first (regardless of mechanical/architectural)
2. Mechanical batches grouped by theme (all dead code together, all renames together)
3. Architectural findings each as their own batch (need individual planning)
4. Batch size: mechanical 3-8 findings, architectural 1-2 findings

Step 4 — Ask the user for their fast verification command (under a minute, covers the code being changed).

Step 5 — Present to user for approval:
- Proposed batch ordering with rationale for foundations first
- Won't-fix candidates with rationale for each
- Classification of each finding
- User can: approve as-is, move findings between batches, reject won't-fix candidates, reorder

**Output:** Writes `plan.md` and initial `progress.md`.

**Terminal state:** Invoke `codebase-health-check:resolve-batch`.

**Hard gates:**
- Do NOT write plan.md until user approves
- Do NOT skip won't-fix presentation — even if zero candidates, say so
- Do NOT group architectural findings into mechanical batches

---

### codebase-health-check:resolve-batch

**Description:**
```
Use when you have a triaged health-check plan and need to execute the
next batch of fixes. Also use when resuming a health check that has
unresolved batches remaining.
```

**Entry:** Invoked by triage's terminal state, or independently to resume. Reads `plan.md` and `progress.md` to determine which batch is next.

**Execution diverges by classification:**

#### Mechanical batches

1. **Pre-batch verification.** Run the verification command from plan.md. If it fails, stop — don't fix health-check findings on a broken baseline.
2. **Dispatch subagents.** One subagent per finding (or one for the whole batch if findings are tightly related). Each prompt includes: full finding text from report.md, the specific fix expected, constraint to touch only what the finding describes, requirement to run verification before reporting.
3. **Review subagent results.** Read what changed. Check for conflicts between parallel edits. Resolve conflicts if any.
4. **Full verification.** Run verification command. If failures, diagnose and fix before proceeding.
5. **Commit.** Single commit for the batch. Message references finding numbers.
6. **Update documents.** Write `batch-N-theme.md`. Rewrite `progress.md`.

#### Architectural batches

1. **Pre-batch verification.** Same gate — green baseline required.
2. **Design before implementing.** Read the finding and surrounding code. Propose 2-3 approaches for the refactoring. Present to user, get approval. Only then implement.
3. **Implement.** Main session implements the approved approach. For large decompositions, work incrementally — extract one piece, verify, extract the next.
4. **Full verification.** Same gate.
5. **Commit.** Message references finding number and describes the structural change.
6. **Update documents.** Same as mechanical.

#### After the batch (both types)

Read `progress.md`. If unresolved batches remain → re-invoke `codebase-health-check:resolve-batch`. If all batches resolved → invoke `codebase-health-check:complete`.

**Hard gates:**
- Do NOT start a batch without passing the verification command
- Do NOT skip the design step for architectural batches
- Do NOT commit without verification passing
- Do NOT proceed to next batch if current batch verification failed
- Do NOT update progress.md before the commit succeeds

**Rationalization defenses:**
| Claude thinks... | Skill responds... |
|---|---|
| "This architectural fix is simple enough to skip design" | Classification was approved in triage. Architectural gets design. |
| "Tests are slow, I'll just run lint" | Use the verification command from plan.md. The user chose it. |
| "The subagent said it verified" | Do not trust subagent success reports. Run verification yourself. |
| "I'll commit both batches together" | One commit per batch, 1:1 with resolution log. |
| "This finding is basically won't-fix, I'll skip it" | Won't-fix was decided in triage. If it's in a batch, it gets fixed. |

---

### codebase-health-check:complete

**Description:**
```
Use when all batches in a health-check resolution plan are resolved
and you need to finalize the health check.
```

**Entry:** Invoked by resolve-batch when all batches are resolved. Or independently.

**Steps:**

1. **Final progress rewrite.** Rewrite `progress.md` with final counts and completion date.
2. **Present summary to user.** Show: final counts, won't-fix items with rationale, list of commits produced, path to health-check directory.
3. **Commit.** `docs: complete health check — N resolved, M won't fix`

**Terminal state:** None. Health check is done.

**Hard gates:**
- Do NOT invoke if any batches are unresolved, unless user explicitly says to close early
- Do NOT silently drop unresolved findings — present what's still open and ask

---

## Changes to Existing Analysis Skill

Three changes. Everything else stays as-is.

**Change 1: Directory output.** Phase 4 creates `docs/health-checks/YYYY-MM-DD/report.md` instead of `docs/health-checks/YYYY-MM-DD-health-check.md`.

**Change 2: Hard pipeline in Phase 5.** Replace the loose brainstorming handoff with: "Would you like to triage and resolve these findings?" If yes → invoke `codebase-health-check:triage`. If no → end.

**Change 3: Remove superpowers:brainstorming reference.** Triage handles the "what to do" question. Resolve-batch handles per-finding design for architectural fixes inline.

---

## New Command: /resolve

```markdown
---
description: "Resume resolving findings from an existing health-check report."
---

Find the most recent health-check directory under docs/health-checks/.
Read progress.md to determine state:
- No plan.md → invoke codebase-health-check:triage
- Has plan, unresolved batches → invoke codebase-health-check:resolve-batch
- All batches resolved → invoke codebase-health-check:complete

The user's arguments are: ${ARGUMENTS}
If arguments provided, use as path to health-check directory.
```

---

## Skill Descriptions (Claude Search Optimization)

Descriptions say *when to use*, never *what the skill does*:

| Skill | Description |
|-------|-------------|
| codebase-health-check | (unchanged) Analyze a codebase for structural health issues... |
| triage | Use when you have a health-check report and need to decide which findings to fix, which to skip, and what order to tackle them. Also use when resuming a health check that has a report but no resolution plan. |
| resolve-batch | Use when you have a triaged health-check plan and need to execute the next batch of fixes. Also use when resuming a health check that has unresolved batches remaining. |
| complete | Use when all batches in a health-check resolution plan are resolved and you need to finalize the health check. |

---

## Patterns Encoded (Origin)

Each design decision traces back to observed session behavior:

| Pattern | Source | Where encoded |
|---------|--------|---------------|
| Foundation-first ordering | Splash: registry unblocked 7+ findings | Triage step 3, ordering rule 1 |
| Theme-based batching | FDY: 6 mechanical fixes in one commit | Triage step 3, ordering rule 2 |
| Ceremony ∝ complexity | Both: over-planning simple fixes wasted time | Resolve-batch: subagents for mechanical, design for architectural |
| Conservative won't-fix | Splash: 4 findings closed with rationale | Triage step 2, explicit presentation gate |
| User-specified verification | FDY: full suite takes 1hr+ | Triage step 4, stored in plan.md |
| Don't trust subagent reports | Superpowers: verification-before-completion | Resolve-batch rationalization defense |
| Append-only resolution log | Both: updating doc after each batch was valuable | Batch files as separate documents |
| Living progress tracking | Splash: inline markers enabled quick scanning | progress.md rewritten after each batch |
| Resumability | FDY: picked up days later in new session | /resolve command + document-state detection |
| One commit per batch | Both: clean git history, easy bisect | Resolve-batch hard gate |
